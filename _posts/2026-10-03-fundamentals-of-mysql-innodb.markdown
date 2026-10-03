---
layout: post
title:  "Fundamentals of MySQL InnoDB"
date:   2026-10-03 00:00 +0800
categories: databases
excerpt: "What InnoDB's B+tree leaves actually store for primary and secondary indexes, how its index-organised tables differ from PostgreSQL's heap, and how the undo and redo logs relate to MVCC and durability."
---
Each section opens with the one-line answer, then the follow-ups.

1. [What a leaf is](#leaf)
2. [What a secondary index leaf stores](#secondary)
3. [What a primary index leaf stores](#primary)
4. [Are the leaves the data pages? InnoDB versus PostgreSQL](#heap)
5. [Does MVCC use the redo and undo logs?](#mvcc)
6. [Why use a redo log?](#redo)

## 1. What a leaf is
{: #leaf}

<div class="callout">An InnoDB index is a B+tree. The leaves are its bottom level, where the search ends and the data sits.</div>

A tree has three kinds of pages, each a 16 KB block on disk:

- **Root:** the single page at the top, where every lookup starts.
- **Internal pages:** the middle levels. They hold only keys and pointers to child pages, like signposts.
- **Leaf pages:** the bottom level. They hold the data and are linked to each other in key order, so a range scan can walk sideways.

```text
                 [ 100 | 200 ]                 root (signposts only)
               /       |       \
      [..50..]     [..150..]     [..250..]     internal (signposts only)
       /    \        /    \        /    \
    [leaf]<->[leaf]<->[leaf]<->[leaf]<->[leaf]  leaves (the data)
```

Analogy: the secondary index is the index at the back of a textbook, giving you a page number. The primary index is the book itself.

## 2. What a secondary index leaf stores
{: #secondary}

<div class="callout">The indexed column values plus the primary key of the row. Not the full row, and not a physical pointer.</div>

- **Second lookup:** if the query needs columns that are not in the index, InnoDB takes the primary key from the leaf and searches the clustered index to fetch the row.
- **Covering index:** if every column the query needs is already in the secondary index, the second lookup is skipped and EXPLAIN shows "Using index".
- **Primary key size matters:** the primary key is copied into every secondary index, so a wide key bloats all of them. Prefer a compact, increasing key.

### Why the primary key and not a physical address?

Rows move when pages split or merge. Storing the logical key means secondary indexes do not need rewriting when that happens.

## 3. What a primary index leaf stores
{: #primary}

<div class="callout">The full row: the primary key, every other column whether indexed or not, and two hidden MVCC fields.</div>

- **Transaction ID:** the last transaction that modified the row.
- **Roll pointer:** a pointer into the undo log, where the previous version of the row lives.

```text
users(id, email, name, balance)

Primary index leaf:      id=7 | trx_id | roll_ptr | email=a@x.com | name=Ana | balance=500
Secondary (email) leaf:  email=a@x.com | id=7
```

`SELECT name FROM users WHERE email = 'a@x.com'` finds `id=7` in the email index, then goes to the primary index leaf to read `name`, because `name` only exists there.

### Follow-ups

- **No primary key defined:** InnoDB uses the first unique index whose columns are all NOT NULL; otherwise it generates a hidden 6-byte row ID and clusters on that.
- **Random keys hurt:** rows are physically ordered by primary key, so random values such as UUID v4 land mid-page and cause page splits and fragmentation. An increasing key appends to the last page.
- **Large values:** long TEXT, BLOB or VARCHAR values may be moved to overflow pages, with the leaf keeping a pointer and sometimes a short prefix.

## 4. Are the leaves the data pages?
{: #heap}

<div class="callout">Yes. In InnoDB the leaf pages of the primary index are the data pages. There is no separate heap.</div>

The idea that indexes and data pages are separate describes a different storage design:

- **Heap-organised (PostgreSQL, MyISAM):** rows live in a heap in no particular order. Every index, including the primary key index, is a separate structure whose leaves point to the row's physical location.
- **Index-organised (InnoDB):** the table is stored as the primary key B+tree. Internal pages are pure index; leaf pages hold the full rows. Secondary index leaves hold the primary key.

```text
PostgreSQL:  PK index leaf    -> (page 12, slot 3) -> heap page: full row
             email index leaf -> (page 12, slot 3) -> heap page: full row

InnoDB:      PK index leaf    =  full row
             email index leaf -> id=7 -> PK index leaf: full row
```

| Aspect | InnoDB | PostgreSQL |
|---|---|---|
| Primary key lookup | One tree, then done | Index, then heap |
| Secondary lookup | Two trees | One tree, then jump to the heap page |
| Row moves or is updated | Secondary indexes unaffected; they store the primary key | New tuple version at a new location; indexes need new entries unless it is a HOT update |
| Primary key choice | Sets physical order and is copied into every secondary index; keep it small and increasing | Matters much less |

**One line to remember:** Postgres indexes point at the heap; InnoDB has no heap, and the primary index is the table.

## 5. Does MVCC use the redo and undo logs?
{: #mvcc}

<div class="callout">MVCC uses the undo log, not the redo log.</div>

- **Undo log, used by MVCC:** before a row is changed, InnoDB saves the old version in the undo log and the row's roll pointer points to it. A reader whose snapshot should not see the new version follows that pointer back. The same records are used to roll back a transaction.
- **Redo log, not part of MVCC:** it records physical page changes so a committed transaction can be replayed after a crash. Readers never consult it.

In ACID terms: undo gives atomicity (rollback) and, through MVCC, isolation; redo gives durability.

### Follow-ups

- **They interact:** undo records live in pages, so changes to the undo log are themselves written to the redo log. After a crash, InnoDB replays redo first, then uses undo to roll back transactions that had not committed.
- **Undo cleanup:** the purge thread can only remove old versions once no active read view needs them, so a long-running transaction makes the undo history grow.
- **Contrast with Postgres:** no undo log. Old row versions stay in the table and vacuum clears them; its WAL plays the role of the redo log.

## 6. Why use a redo log?
{: #redo}

<div class="callout">It turns random page writes into sequential log appends, so commits are both fast and crash-safe.</div>

### The problem

InnoDB changes pages in memory (the buffer pool). Flushing every changed page at commit would be slow:

- **Random I/O:** one transaction can touch pages scattered across the data files.
- **Wasteful writes:** a whole 16 KB page would be written to persist a change of a few bytes.

### What the redo log does instead

It is a write-ahead log. Before a commit is acknowledged, a small record describing the change is appended to the redo log and synced to disk. The data pages stay dirty in memory and are flushed later in the background.

- **Durability at low cost:** a commit needs only a small sequential append.
- **Crash recovery:** on restart, InnoDB replays the log from the last checkpoint.
- **Group commit:** many transactions' records can be synced in a single fsync.

### Follow-ups

- **Checkpoint:** the redo log is fixed-size and circular. Space is reused only once the pages it covers have been flushed, so if writes outrun flushing, InnoDB stalls to flush.
- **Durability setting:** `innodb_flush_log_at_trx_commit=1` syncs on every commit, which is what you want for payments. 0 or 2 can lose roughly the last second of commits in exchange for speed.
- **Why not just the binlog:** the binlog is a logical, server-level log for replication and point-in-time recovery. It cannot tell InnoDB which pages were half-written at crash time.
