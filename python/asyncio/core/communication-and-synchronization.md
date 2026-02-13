# Communication and Synchronization

Tutorial: [`../tutorial/building-real-applications.md`](../tutorial/building-real-applications.md)
Core: [`execution-model.md`](execution-model.md)

Asyncio programs are rarely about one coroutine: they are about many tasks that
communicate, coordinate, and apply backpressure to each other.

This page stays strictly within the standard library and focuses on patterns
and pitfalls when using:

- Queues: [`asyncio.Queue`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue),
  [`asyncio.PriorityQueue`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.PriorityQueue),
  [`asyncio.LifoQueue`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.LifoQueue)
- Sync primitives: [`asyncio.Lock`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Lock),
  [`asyncio.Event`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Event),
  [`asyncio.Condition`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Condition),
  [`asyncio.Semaphore`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Semaphore),
  [`asyncio.BoundedSemaphore`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.BoundedSemaphore),
  [`asyncio.Barrier`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Barrier)

## Backpressure Is a First-Class Problem

Backpressure is what stops your system from buffering forever when a consumer
is slower than a producer.

In `asyncio`, you do not get backpressure “for free” just because code is
asynchronous. If you do not choose where waiting happens, you tend to get:

- fast producers that keep running
- memory growth (queues, lists, write buffers) until the process falls over

Good designs make backpressure explicit by putting flow control at the boundary
where work enters a subsystem:

- a bounded queue (`await q.put(...)` becomes the flow-control point)
- a stream write (`await StreamWriter.drain()` becomes the flow-control point)
- a concurrency limit (`async with Semaphore(...)` becomes the flow-control
  point)

Two APIs are worth treating as “where backpressure lives”:

- `StreamWriter.drain()`:
  https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.drain
- `Queue(maxsize=...)`:
  https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue

Tie-in to the tutorial chat server: it uses both.

## Queues

Reference: https://docs.python.org/3/library/asyncio-queue.html

### Choosing a Queue Type

- `Queue`: FIFO. Use this by default.
- `LifoQueue`: newest-first.
- `PriorityQueue`: lowest priority first.

`PriorityQueue` pitfall: if you push tuples like `(priority, item)` and two
priorities are equal, Python compares the second tuple element to break the
tie. If `item` objects are not orderable, you can get `TypeError`.

Common fix: include a monotonic tie-breaker:

```python
import itertools


counter = itertools.count()
await pq.put((priority, next(counter), item))
```

Docs: [`asyncio.PriorityQueue`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.PriorityQueue)

### `maxsize`: Make Buffers Explicit

If `maxsize <= 0`, the queue is unbounded and `put()` never blocks.
If `maxsize > 0`, `await put()` blocks when the queue reaches capacity.

That blocking is not just a performance detail; it is the mechanism that
prevents unbounded buffering.

Practical advice:

- Prefer a bounded queue at subsystem boundaries (ingress queues, per-client
  outgoing buffers).
- Treat `maxsize` as part of your correctness story: it defines a worst-case
  buffer size.

### `join()` / `task_done()`

`join()` is for “wait until everything that was enqueued has been processed”.
It is unrelated to shutting down consumers.

The invariant:

- every `get()` that returns a work item must have exactly one `task_done()`

Pitfalls:

- Forgetting `task_done()` causes `join()` to block forever.
- Calling `task_done()` too many times raises `ValueError`.

Docs:

- [`Queue.join()`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue.join)
- [`Queue.task_done()`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue.task_done)

### Shutdown (Python 3.13+): `Queue.shutdown()`

Python 3.13 added explicit queue shutdown:

- `q.shutdown(immediate=False)` prevents further `put()` and lets consumers
  drain already-queued items.
- `get()` / `put()` raise `asyncio.QueueShutDown` once shut down.

Docs:

- [`Queue.shutdown()`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue.shutdown)
- [`asyncio.QueueShutDown`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.QueueShutDown)

This API is useful, but treat `shutdown(immediate=True)` as a sharp tool. It
can break the usual `join()` invariant.

### Earlier Versions: Alternatives to `Queue.shutdown()`

If you need a “no more items will arrive” signal on Python < 3.13, you
generally choose one of these patterns.

1) Sentinel objects (common for worker pools)

```python
import asyncio


SENTINEL: object = object()


async def worker(q: asyncio.Queue[object]) -> None:
    while True:
        item = await q.get()
        try:
            if item is SENTINEL:
                return
            await process(item)
        finally:
            q.task_done()


async def stop_workers(q: asyncio.Queue[object], n: int) -> None:
    for _ in range(n):
        await q.put(SENTINEL)
```

Pitfalls: you must enqueue one sentinel per consumer; with a bounded queue,
shutdown can block while trying to enqueue sentinels.

2) Cancel the consumer tasks

- Create consumer tasks in a scope that owns their lifetime (prefer
  `asyncio.TaskGroup`).
- On shutdown, cancel them and let `CancelledError` unwind.

Docs:

- [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup)
- [`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)

## Synchronization Primitives

These primitives coordinate access to shared state. Like queues, they are
designed for tasks within one event loop and are not thread-safe.

Reference: https://docs.python.org/3/library/asyncio-sync.html

### `Lock`: Mutual Exclusion (Fair Acquisition)

`asyncio.Lock` is a mutex.

- Acquiring a lock is documented as fair: the first task that starts waiting
  gets it next.

Docs:

- [`asyncio.Lock.acquire()`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Lock.acquire)

Common pitfall: holding a lock across an `await` that can block for a long
time.

Safe pattern: take a snapshot under the lock, then await outside it.

```python
import asyncio


class State:
    def __init__(self) -> None:
        self._lock = asyncio.Lock()
        self._clients: set[asyncio.Queue[str]] = set()

    async def broadcast(self, msg: str) -> None:
        async with self._lock:
            targets = list(self._clients)

        for q in targets:
            await q.put(msg)
```

### `Event`: Broadcast Notification

`asyncio.Event` is a broadcast signal: many tasks can wait on it, and setting
it wakes them.

It is level-triggered:

- If the flag is already set, `wait()` returns immediately.

Docs: [`asyncio.Event`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Event)

### `Condition`: Wait for State Changes

`asyncio.Condition` is for “wait until some predicate about shared state
becomes true”. It combines a lock with notifications.

Two rules:

1. Always check the predicate under the condition’s lock.
2. Be prepared for spurious wakeups; prefer `wait_for(predicate)`.

Docs: [`asyncio.Condition`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Condition)

### `Semaphore` / `BoundedSemaphore`: Limit Concurrency

Semaphores are the standard tool for limiting concurrent access to some
resource.

Docs:

- [`asyncio.Semaphore`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Semaphore)
- [`asyncio.BoundedSemaphore`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.BoundedSemaphore)

### `Barrier`: Phase Synchronization

`asyncio.Barrier(parties)` blocks tasks until a fixed number have arrived.

Docs:

- [`asyncio.Barrier`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Barrier)
- [`asyncio.BrokenBarrierError`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.BrokenBarrierError)

## Patterns

### Producer/Consumer with a Bounded Queue

This is the default “pipe” shape in asyncio.

```python
import asyncio


async def producer(q: asyncio.Queue[int]) -> None:
    for i in range(1000):
        await q.put(i)  # backpressure point


async def consumer(q: asyncio.Queue[int]) -> None:
    while True:
        item = await q.get()
        try:
            await process(item)
        finally:
            q.task_done()
```

If you need deterministic shutdown of the consumers, use one of the shutdown
patterns described earlier (sentinels, cancellation, or queue shutdown on
Python 3.13+).

### Broadcast notification vs broadcast data

“Broadcast” can mean two different things:

1. Broadcast a signal (“something changed”): use `Event` or `Condition`.
2. Broadcast data (“every subscriber must receive this message”): a common
   pattern is “one queue per subscriber”, protected by a lock.

That is exactly what the tutorial chat server does.

## Tie-In: The Tutorial Chat Server

The chat server in [`../tutorial/building-real-applications.md`](../tutorial/building-real-applications.md)
uses two flow-control points:

1. Per-client bounded queue

- each client has a `Queue(maxsize=...)` for outgoing messages
- broadcast awaits `put()` to slow down when a client is slow

2. `StreamWriter.drain()`

- the writer task writes then awaits `drain()`
- `drain()` yields when the transport write buffer is full

Docs:

- [`StreamWriter.write()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.write)
- [`StreamWriter.drain()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.drain)

## Pitfall Checklist

- Unbounded buffers: `Queue()` without `maxsize`, `StreamWriter.write()` without
  `drain()`.
- Waiting while holding a lock: avoid long `await`s inside `async with lock:`.
- Condition misuse: always re-check predicates; prefer `Condition.wait_for()`.
- `join()` hangs: missing `task_done()`, or mismatched work/sentinel accounting.
