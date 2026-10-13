---
layout: post
title:  "Engineering a Durable Workflow System: Credit Card Application"
date:   2026-10-13 16:00 +0800
categories: system-design
excerpt: "How a credit card application workflow survives double-clicks, crashes, duplicate events, a vendor without idempotency keys and lost webhooks, using nothing but Postgres."
---
A credit card application looks like a form and one API call. Behind it is a workflow that has to survive double-clicks, crashed instances, duplicate events, a billed vendor and webhooks that may never arrive. Each section below opens with the one-line answer, then the follow-ups. Source code is at [mettyoung/credit-card-system](https://github.com/mettyoung/credit-card-system), and the whole flow is animated in the [decision loop diagram](https://creditcardapplication.mettyoung.com/architecture/decision-loop.html).

The loop is draft → upload ID → submit → identity check (Onfido) → webhook → decision. Paths below are relative to `src/main/java/com/mettyoung/creditcardapplication/`.

1. [Starting the workflow exactly once](#idempotency)
2. [The dual write problem](#dual-write)
3. [Duplicates are a side effect of the outbox](#duplicates)
4. [Crashes in the middle of a step](#leases)
5. [Calling a paid vendor without idempotency support](#vendor)
6. [Dedupe on the outbox and the inbox](#inbox)
7. [The waiting period deadline](#deadline)
8. [Audit trail](#audit)
9. [Running it on a minimal footprint](#footprint)

<h2 id="idempotency">1. Starting the workflow exactly once</h2>

**Let the database refuse the duplicate; never check-then-act.**

A double-click, a retried request after a lost `201`, or two open tabs must not start two workflows. Reading "does a draft exist?" and then inserting is a race. Instead:

- The server assigns the id before the insert: a **UUIDv7**, so rows sort by creation time. The client never invents it.
- A partial unique index, `ux_application_one_live` (`db/migration/V10__one_live_application.sql`), allows one live application per user and product. The second insert fails; the service translates the violation *by constraint name* into `409 DraftAlreadyExists` carrying the existing `applicationId`. The client carries on with that id.
- `createDraft` is deliberately **not** `@Transactional`: the aborted transaction could not run the follow-up lookup of the existing id.
- Saves carry the `version` the client read, checked against JPA `@Version`. A stale copy gets `409` with the current version. The update runs in its own `TransactionTemplate` with `saveAndFlush`, so a lost race is translated after the rollback instead of surfacing as `UnexpectedRollbackException` at commit.

See `application/internal/ApplicationService.java` and ADR 0006, *The database settles concurrency*.

<h2 id="dual-write">2. The dual write problem</h2>

**Write the event as a row in the same transaction as the state change: a transactional outbox.**

Submit must change state *and* tell the rest of the system. With a database plus a broker, there is always a window where one write exists without the other, whichever order you pick.

On `POST /v1/applications/{id}/submit`, one transaction writes:

- the status transition,
- the `WORKFLOW_STARTED` audit row,
- an `outbox` row (`db/migration/V5__outbox.sql`).

If it rolls back, none exist; if it commits, all do. Nothing has been sent to anyone yet. The response is **`202 Accepted`**, because the checks have not run. `shared/outbox/OutboxWriter.java` is `@Transactional(propagation = MANDATORY)`, so an outbox row cannot be written outside the transaction it describes. Event payloads carry ids only, so no personal data sits in the log.

<h2 id="duplicates">3. Duplicates are a side effect of the outbox</h2>

**A poller can only deliver at-least-once; a dedupe table on the consumer turns that into exactly-once processing.**

`shared/outbox/OutboxRelay.java` polls committed rows every second and dispatches them in process. If it crashes after the consumer committed but before marking the row published, the event goes out again. That is unavoidable.

The consumer therefore claims each event by inserting into `processed_event`:

```sql
CREATE TABLE processed_event (
    consumer     text        NOT NULL,
    event_id     uuid        NOT NULL,
    processed_at timestamptz NOT NULL,
    PRIMARY KEY (consumer, event_id)
);
```

The primary key, not application logic, refuses the second attempt. The marker, the workflow step and "mark published" commit in one transaction, so delivery is at-least-once but the effect happens once.

The workflow itself is advanced by one component, `application/internal/ApplicationProcess.java`, the only thing allowed to transition an application. The next step comes from `Evaluator.evaluate`, a pure function with no I/O and no clock. If two events race on one application, `@Version` makes one lose, and the relay replays that event from the start instead of resolving a half-applied decision.

<h2 id="leases">4. Crashes in the middle of a step</h2>

**Claim work with a lease column in the database; an expired lease is treated exactly like a crash.**

A row lock ends when the claiming transaction commits, but the work outlives that transaction. So both the relay and the vendor worker claim in a short transaction of their own:

```sql
SELECT ... FROM outbox
 WHERE published_at IS NULL
   AND (claimed_until IS NULL OR claimed_until < :now)
 ORDER BY id
 FOR UPDATE SKIP LOCKED
```

then set `claimed_by` and `claimed_until` (one-minute lease, `OutboxRelay.LEASE`, `VendorWorker.LEASE`) and commit. `SKIP LOCKED` only covers the instant between the select and the update: a second instance polling at the same moment skips the row instead of blocking or claiming it twice. The lease covers everything after.

If the instance dies, nobody has to notice. The lease runs out, the row matches the claim query again, and another instance picks it up. No coordinator, no heartbeat. See `shared/outbox/OutboxRepository.java` and `vendor/internal/VendorCheckRepository.java`.

<h2 id="vendor">5. Calling a paid vendor without idempotency support</h2>

**Look before you create: list the vendor's existing checks (free) before creating one (billed), and adopt what you find.**

The ideal fix is an idempotency key on the vendor's side, so a retried request returns the original result. Onfido ignores `Idempotency-Key`, and only `POST /checks` is billed. A lost reply followed by a naive retry pays twice.

`vendor/internal/OnfidoIdvAdapter.java` and `VendorWorker.java` handle it in steps:

- **The table is the queue.** The orchestrator inserts a `vendor_check` row (`V6__vendor_check.sql`) as the command, keyed `application:IDV:document`.
- **Claim, call, record: three transactions, never one.** The claim commits before Onfido is called. Holding a transaction open across a third-party call turns a slow vendor into a connection-pool outage.
- **Register the applicant first.** Onfido's applicant id is committed in its own small transaction before the billed call, so a later attempt knows which applicant to ask about.
- **List before create.** Every attempt lists the applicant's checks first. If one exists, it is adopted and our row is updated to point at it; billed checks stay at one.
- **Backoff.** Retryable failures are rescheduled at `2^attempts` seconds plus jitter, so instances do not retry in step, up to `app.onfido.max-attempts=5`. The lease is cleared, so the row becomes ordinary due work at `next_attempt_at`.

<h2 id="inbox">6. Dedupe on the outbox and the inbox</h2>

**A webhook is a notification, never a result: verify it, store it once, answer 200, and fetch the real answer yourself.**

The outbox dedupes what we send; the inbox dedupes what we receive. `vendor/internal/IdvWebhookController.java`:

- Verifies the HMAC-SHA256 over the **raw bytes** (`SignatureVerifier.java`). Re-serialising would reorder keys and break a genuine signature.
- Inserts a `vendor_inbox` row and returns `200` fast, for duplicates too. `ux_vendor_inbox_event (vendor, event_id)` makes a redelivery insert nothing, even when two copies arrive at once. Onfido sends no event id, so the key is the resource plus the action.

`vendor/internal/InboxWorker.java` then claims unprocessed rows, calls Onfido's status endpoint outside any transaction, and completes the check with a conditional update:

```sql
UPDATE vendor_check ... WHERE id = :id AND status = AWAITING_CALLBACK
```

Completion, audit row and the `VendorCheckCompleted` outbox row commit together. Zero rows updated means someone else already completed it, so nothing is written. That conditional update is what ties inbox and outbox together: however many webhooks, retries or pollers reach the same check, it produces one outbox event, and `processed_event` makes that event advance the workflow once.

The worker marks inbox rows processed when it claims them. If it dies between the claim and the completion, that row is not retried, and the check is still `AWAITING_CALLBACK`. That gap is closed by the next section.

<h2 id="deadline">7. The waiting period deadline</h2>

**A workflow cannot wait forever: give every wait a deadline and a reconciler that polls until it passes.**

When the check is created, `deadline_at` is set 30 minutes out (`app.onfido.result-deadline=30m`). `vendor/internal/Reconciler.java` polls checks still `AWAITING_CALLBACK` (every 30 seconds by default) and asks Onfido directly. It races the inbox worker on purpose; both use the same conditional update, so only one wins.

This one job covers every failure in section 6: a lost webhook, a forged one that was rejected, or an inbox worker that died mid-step. Each becomes a delay, not a stuck application.

When the deadline passes, or retries run out, the requirement settles as **`UNAVAILABLE`**, never as a decline. `application/internal/DecisionRules.java` is a fixed table:

- all clean → `APPROVED`;
- a `FRAUD` identity result or an `UNAVAILABLE` requirement → `REFERRED` for manual review (`EVIDENCE_UNAVAILABLE`);
- only a reviewer can decline, with a reason.

An outage on our side must not decline an applicant. `DeadlineSweep` then counts referrals older than 48 hours every 15 minutes for alerting, and never decides one.

<h2 id="audit">8. Audit trail</h2>

**In finance, a log that disagrees with the data is worse than no log, so the audit write joins the change's transaction and the table refuses edits.**

- `audit/internal/AuditLog.java` is `@Transactional(propagation = MANDATORY)`. Leaving the annotation off is not enough: Spring Data's `save` is itself transactional, so a caller without a transaction would silently commit an entry for a change that later rolls back. `MANDATORY` makes that an exception at the first call.
- Append-only is a database trigger (`V2__audit_event.sql`) that raises `audit_event is append-only` on `UPDATE` or `DELETE`, rather than a permission someone can forget to revoke.
- `seq` is per application, backed by a unique index, so a concurrent writer fails instead of producing a duplicate.

Every transition, webhook receipt, vendor completion and review decision lands here in the same commit as the change.

<h2 id="footprint">9. Running it on a minimal footprint</h2>

**Postgres is the only infrastructure.** Queue, outbox, inbox, leases, dedupe and audit are all tables. There is no broker, no scheduler service and no cache.

The [live demo](https://creditcardapplication.mettyoung.com/) runs in one Cloudflare Container under `supervisord`: Postgres, MinIO for uploads, a WireMock Onfido mock, a bootstrap step running Flyway, and the Spring Boot app (`cf-container/`). The container's disk is ephemeral, so the data is wiped when it sleeps after 30 idle minutes or on each deploy. That is fine for a demo.

## Takeaways

- Let unique indexes and `@Version` settle races; don't check first.
- Outbox for sending, inbox for receiving, a dedupe key on each side.
- Leases in a column survive crashes without a coordinator.
- If the vendor has no idempotency key, look before you create.
- A webhook is a hint; a reconciler with a deadline is the guarantee.
- An unanswered check goes to a person, not to a decline.
