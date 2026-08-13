---
layout: post
title:  "Java Concurrent Collections Cheat Sheet"
date:   2026-08-13 00:00 +0800
categories: cheatsheet
excerpt: "Every java.util.concurrent collection broken down by common methods, complexity, and the concurrency design behind it — locking strategy, CAS usage, and the tradeoffs each one makes."
---
Here's each implementation broken down by common methods, complexity, and the concurrency design behind it.

## 1. Concurrent Maps

### `ConcurrentHashMap<K,V>`

**Locking design:** Pre-Java 8, this used **segment-based locking** — the map was divided into 16 segments (default), each with its own lock, so up to 16 threads could write concurrently as long as they hit different segments. Java 8+ threw out segments entirely. Now it locks at the **bin (bucket) level**: each bucket head node is the synchronization point. Reads (`get`) are fully lock-free, using `volatile` reads and `Unsafe`/`VarHandle` access — no locking at all. Writes use **CAS** to insert into an empty bin; if the bin's occupied, it synchronizes on just that bin's first node. Table resizing is done cooperatively — multiple threads can help transfer nodes to the new table.

**Bucket count & distribution:** The bucket array (`table`) always sizes to a **power of 2** — default initial capacity is 16 buckets. It grows when `size > capacity * loadFactor` (default load factor 0.75, so 16 buckets resizes at 12 entries), and each resize **doubles** the table. A key's bucket index isn't just `hashCode() % buckets` — `ConcurrentHashMap` first runs a **spread function**, `h ^ (h >>> 16)`, XOR-folding the high 16 bits of the hash into the low bits, then masks with `(n - 1) & hash` to get the index (cheap because `n` is a power of 2). The XOR-spread step matters for concurrency: without it, hash codes that only differ in high bits (a common pattern for `Integer`/enum-derived hashes) would collide into the same bin far more often, defeating the whole point of per-bin locking by forcing threads to fight over a handful of hot bins instead of spreading writes across the table. If a bin still accumulates 8+ collisions **and** the table has reached at least 64 buckets, that bin treeifies into a red-black tree; below 64 buckets it resizes the whole table instead, since treeifying a tiny table doesn't help distribution the way growing it does.

| Method | Complexity | Notes |
|---|---|---|
| `get(key)` | O(1) avg | Fully lock-free volatile read |
| `put(key, val)` | O(1) avg | CAS if bin empty; else synchronized on bin's head node |
| `putIfAbsent` | O(1) avg | Same locking as put, atomic check-then-act |
| `computeIfAbsent` | O(1) avg | Locks the bin for the whole compute — avoid slow functions here |
| `remove(key)` | O(1) avg | Synchronized on bin |
| `containsKey` | O(1) avg | Lock-free |
| `size()` | O(1) amortized | Uses a striped `LongAdder`-like counter (`CounterCell[]`), not a full scan |
| `forEach` / `reduce` (bulk) | O(n) | Can run with configurable parallelism via `ForkJoinPool.commonPool()` |

Worst case degrades to O(log n) per bin because Java 8+ treeifies bins with 8+ collisions (converts a linked list bin into a red-black tree bin).

### `ConcurrentSkipListMap<K,V>`

**Locking design:** No locks on reads at all — traversal is lock-free using `volatile` next-pointers. Insertion/deletion use **CAS** on individual node links, not a global or per-node lock. This works because a skip list's structure is a set of independent linked lists at different "levels," so a CAS-based splice at one level doesn't require blocking the rest of the structure. This is why skip lists are preferred over trees here: rebalancing a red-black tree touches many nodes and would require locking large parts of it; a skip list's insert only needs local CAS operations on the levels it touches.

<div class="callout">
  <span class="callout-title">📺 How skip lists work</span>
  If the level structure isn't clicking, <a href="https://www.youtube.com/watch?v=ol-FaNLXlR0" target="_blank" rel="noopener">this video</a> walks through skip list mechanics visually — useful background before reasoning about how CAS splices operate on top of it.
</div>

| Method | Complexity | Notes |
|---|---|---|
| `get(key)` | O(log n) | Lock-free traversal down levels |
| `put(key, val)` | O(log n) | CAS-based node splice, no locks |
| `remove(key)` | O(log n) | CAS-based unlink, uses "marker nodes" to logically delete before physical unlink |
| `firstKey` / `lastKey` | O(log n) | Or O(1) with cached fields depending on version |
| `ceilingKey` / `floorKey` / `higherKey` / `lowerKey` | O(log n) | Skip-list navigation |
| `subMap` / `headMap` / `tailMap` | O(log n) to construct | Returns a live view, not a copy |

<div class="callout callout-tradeoff">
  <span class="callout-title">⚖️ ConcurrentHashMap vs ConcurrentSkipListMap</span>
  <ul>
    <li><strong>Ordering:</strong> <code>ConcurrentHashMap</code> has none. <code>ConcurrentSkipListMap</code> keeps keys sorted, so it's the only one of the two that supports <code>firstKey</code>/<code>ceilingKey</code>/<code>subMap</code>-style range queries.</li>
    <li><strong>Lookup cost:</strong> <code>ConcurrentHashMap</code> is O(1) avg via bucket indexing. <code>ConcurrentSkipListMap</code> is O(log n) — it has to walk levels since there's no hash table underneath.</li>
    <li><strong>Write contention:</strong> both are effectively lock-free/low-lock in the common case, but skip list writes are pure CAS splices with no bin lock fallback at all, so they hold up slightly better under pathological single-bin hash collisions.</li>
    <li><strong>Memory:</strong> skip list nodes carry multiple forward-pointer levels per entry (randomized, geometric distribution) — more overhead per entry than a hash map's flat bucket array + single next-pointer.</li>
    <li><strong>Default choice:</strong> reach for <code>ConcurrentHashMap</code> unless you specifically need sorted iteration or range queries — it's faster and simpler for pure key lookups.</li>
  </ul>
</div>

## 2. Concurrent Sets

### `ConcurrentSkipListSet<E>`
Just a `ConcurrentSkipListMap<E, Boolean>` under the hood — same locking model, same complexities, values are dummy `Boolean.TRUE`.

| Method | Complexity |
|---|---|
| `add(e)` | O(log n) |
| `contains(e)` | O(log n) |
| `remove(e)` | O(log n) |
| `first()` / `last()` | O(log n) |

### `CopyOnWriteArraySet<E>`
Backed by `CopyOnWriteArrayList` (see below) with a linear `contains()` scan for uniqueness checks on `add`.

| Method | Complexity | Notes |
|---|---|---|
| `add(e)` | O(n) | Must scan for duplicate before copying array |
| `contains(e)` | O(n) | Linear scan, lock-free read |
| `remove(e)` | O(n) | Scan + copy |
| iteration | O(n) | Snapshot iterator |

### `ConcurrentHashMap.newKeySet()`
Same complexities and locking as `ConcurrentHashMap` — it's literally a `KeySetView` wrapper where values are a shared dummy `Boolean.TRUE` object.

## 3. Concurrent Lists

### `CopyOnWriteArrayList<E>`

**Locking design:** A single `ReentrantLock` guards **all mutations** (`add`, `set`, `remove`) — writers are fully serialized. On any mutation, the entire backing array is copied (`Arrays.copyOf`), the change applied to the copy, then the internal `volatile` array reference is swapped. Reads never acquire the lock — they just read the current `volatile` array reference, so a read is always working against a consistent, immutable snapshot at the moment it started.

| Method | Complexity | Notes |
|---|---|---|
| `get(index)` | O(1) | Lock-free, reads current array reference |
| `add(e)` | O(n) | Lock + full array copy |
| `set(index, e)` | O(n) | Lock + full array copy (even though it's one element) |
| `remove(index)` | O(n) | Lock + full array copy |
| `contains(e)` / `indexOf(e)` | O(n) | Lock-free, scans the snapshot array |
| iteration | O(n) | Iterator holds a reference to the array snapshot at creation — never reflects later mutations, never throws CME |

## 4. Queues and Deques

### `ConcurrentLinkedQueue<E>` (non-blocking)

**Locking design:** The Michael-Scott lock-free queue algorithm. No locks anywhere — `offer` and `poll` use **CAS loops** on the `next` pointer of the tail/head nodes. If a CAS fails (another thread got there first), the thread just retries. This is a classic lock-free (not just non-blocking, but *wait-free-ish* under low contention) data structure.

| Method | Complexity | Notes |
|---|---|---|
| `offer(e)` / `add(e)` | O(1) amortized | CAS on tail's next pointer, may retry on contention |
| `poll()` | O(1) amortized | CAS on head |
| `peek()` | O(1) | Lock-free read |
| `size()` | **O(n)** | Must walk the whole linked list — there's no maintained counter, because maintaining one would require a synchronization point that defeats the lock-free design |

### `ConcurrentLinkedDeque<E>` (non-blocking)
Same CAS-based lock-free approach as above but with doubly-linked nodes so both ends support push/pop.

| Method | Complexity |
|---|---|
| `addFirst` / `addLast` | O(1) amortized, CAS-based |
| `pollFirst` / `pollLast` | O(1) amortized, CAS-based |
| `size()` | O(n) |

### `LinkedBlockingQueue<E>` (blocking)

**Locking design:** Deliberately uses **two separate locks** — `putLock` and `takeLock` — instead of one. This is the key optimization: a producer calling `put()` and a consumer calling `take()` can proceed truly concurrently because they touch different ends of the linked structure and use different locks. An `AtomicInteger count` field coordinates between them (both locks check/update it), and two `Condition`s (`notFull`, `notEmpty`) handle blocking/waking.

| Method | Complexity | Notes |
|---|---|---|
| `put(e)` | O(1) | Acquires putLock; blocks on `notFull` condition if bounded and full |
| `take()` | O(1) | Acquires takeLock; blocks on `notEmpty` if empty |
| `offer(e)` | O(1) | Non-blocking, returns false if full |
| `poll()` | O(1) | Non-blocking, returns null if empty |
| `size()` | O(1) | Backed by the shared `AtomicInteger`, unlike ConcurrentLinkedQueue |

### `ArrayBlockingQueue<E>` (blocking)

**Locking design:** A **single `ReentrantLock`** for both put and take (unlike `LinkedBlockingQueue`'s two-lock design) — this is intentional: it's a fixed circular array, and separating locks wouldn't help much since head/tail indices are close together and contention is already low relative to array access cost. Optionally supports fairness (FIFO thread wakeup) via the lock's fair mode.

| Method | Complexity | Notes |
|---|---|---|
| `put(e)` | O(1) | Single lock, blocks on `notFull` |
| `take()` | O(1) | Single lock, blocks on `notEmpty` |
| `offer` / `poll` | O(1) | Non-blocking variants |
| `size()` | O(1) | Simple counter under the lock |

### `PriorityBlockingQueue<E>`

**Locking design:** Single lock guards a binary heap array. Notably, since Java 8, `size()`-related contention is reduced via a spare "allocation" CAS trick when growing the array (grows outside the main lock where possible), but insert/remove-min themselves are fully lock-protected because heap sift-up/sift-down must be atomic.

| Method | Complexity | Notes |
|---|---|---|
| `put(e)` / `offer(e)` | O(log n) | Lock + heap sift-up; never actually blocks (unbounded) |
| `take()` / `poll()` | O(log n) | Lock + heap sift-down; blocks on empty via condition |
| `peek()` | O(1) | Lock still acquired (heap root can move) |

### `DelayQueue<E extends Delayed>`

**Locking design:** Single lock wraps an internal `PriorityQueue` ordered by delay expiry. Adds a `leader-follower` optimization: only one waiting consumer thread ("the leader") does a *timed* wait for the head element's expiry; all others do an untimed wait, avoiding the thundering-herd problem of every consumer thread waking up on every head change.

| Method | Complexity | Notes |
|---|---|---|
| `put(e)` | O(log n) | Heap insert under lock |
| `take()` | O(log n) | Blocks until head's delay expires; leader/follower wait pattern |
| `poll()` | O(log n) | Returns null immediately if nothing expired |

### `SynchronousQueue<E>`

**Locking design:** Has **no internal storage at all** — capacity zero. Internally implemented as a **Scherer-Scott dual queue/stack algorithm**: it maintains a queue of waiting "request" nodes (either a waiting producer or a waiting consumer), and CAS is used to pair a producer's `put` directly with a consumer's `take`. Two internal modes exist: fair mode uses a wait-queue (FIFO), non-fair mode uses a wait-stack (LIFO) for better throughput under contention. No global lock — it's CAS-based node pairing, similar in spirit to lock-free queues.

| Method | Complexity | Notes |
|---|---|---|
| `put(e)` | O(1) amortized | Blocks until a matching `take()` arrives; CAS handoff |
| `take()` | O(1) amortized | Blocks until a matching `put()` arrives |
| `offer(e, timeout)` | O(1) | Times out if no match found |

### `LinkedTransferQueue<E>`

**Locking design:** Extends the same dual-queue CAS approach as `SynchronousQueue` but adds actual unbounded buffering (it's not zero-capacity) and the `transfer()` method, which optimally hands off directly to a waiting consumer without ever touching the queue structure if a consumer is already parked (this is the "fast path" — pure CAS, no lock at all in the common case).

| Method | Complexity | Notes |
|---|---|---|
| `put(e)` | O(1) | Lock-free CAS enqueue, doesn't block (unbounded) |
| `transfer(e)` | O(1) | Blocks until a consumer takes it directly — true lock-free handoff |
| `take()` | O(1) | CAS dequeue or pairs with a waiting transfer |
| `size()` | O(n) | Same limitation as ConcurrentLinkedQueue — must walk nodes |

### `LinkedBlockingDeque<E>`

**Locking design:** Unlike `LinkedBlockingQueue`, this uses a **single lock** for both ends (can't cleanly split into two locks the way a single-ended queue can, since a doubly-linked deque's head and tail operations can interact). Two `Condition`s (`notEmpty`, `notFull`) manage blocking.

| Method | Complexity | Notes |
|---|---|---|
| `putFirst` / `putLast` | O(1) | Single lock, blocks if bounded and full |
| `takeFirst` / `takeLast` | O(1) | Single lock, blocks if empty |
| `offerFirst` / `offerLast` | O(1) | Non-blocking |
| `size()` | O(1) | Maintained counter under lock |

## The underlying optimization patterns, summarized

| Pattern | Used by | Why |
|---|---|---|
| **Bin/bucket-level locking** | `ConcurrentHashMap` | Only lock what you're touching, not the whole structure |
| **Pure CAS, no locks** | `ConcurrentLinkedQueue/Deque`, `ConcurrentSkipListMap/Set`, `LinkedTransferQueue` fast path | Best throughput under contention, no thread parking overhead |
| **Two separate locks (head/tail)** | `LinkedBlockingQueue` | Producers and consumers don't contend with each other at all |
| **Single lock, whole structure** | `ArrayBlockingQueue`, `PriorityBlockingQueue`, `LinkedBlockingDeque` | Simpler correctness where the operations inherently touch shared state (array indices, heap root) |
| **Copy-on-write, lock-free reads** | `CopyOnWriteArrayList/Set` | Optimizes overwhelmingly for reads at the cost of expensive writes |
| **Leader-follower wait reduction** | `DelayQueue` | Avoid waking every waiter on every change |
| **Dual-node CAS pairing** | `SynchronousQueue`, `LinkedTransferQueue` | Zero-copy handoff between threads with no shared buffer |
