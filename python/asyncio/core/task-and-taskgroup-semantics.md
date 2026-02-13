# Task and TaskGroup Semantics

Tutorial: [`../tutorial/structured-concurrency-with-asyncio.md`](../tutorial/structured-concurrency-with-asyncio.md)
Core: [`cancellation-model.md`](cancellation-model.md)

This page is a precise, asyncio-only reference for how `asyncio` schedules
work (`Task`), how cancellation interacts with awaiting, and how structured
concurrency works in modern `asyncio` (`TaskGroup`, `ExceptionGroup`, and
`except*`).

All API links below point to the official Python documentation.

## Tasks, Coroutines, and What Runs Where

- A coroutine object is what you get when you call an `async def` function. It
  does not run until it is awaited (directly or indirectly).
- An [`asyncio.Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task)
  is a wrapper that schedules a coroutine to run concurrently on the event
  loop.

Creating a task schedules the coroutine to start running soon and returns a
handle you can await, cancel, and inspect.

API reference:

- [`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)

## Task Lifecycle and State

From an application perspective, tasks move through three observable states:

1. Not done yet (running or waiting on an `await`).
2. Done successfully (has a result).
3. Done with an error (raised an exception) or done via cancellation.

Key inspection APIs:

- [`Task.done()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.done)
- [`Task.result()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.result)
- [`Task.exception()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.exception)
- [`Task.cancelled()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancelled)

Important semantics:

- `task.done()` becomes true for all terminal outcomes (success, error,
  cancellation).
- `task.cancelled()` is only true if the task finished by propagating
  [`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)
  out of the task.
- `task.result()` returns the value if successful; it re-raises the task’s
  exception if failed; it raises `CancelledError` if cancelled.
- `task.exception()` returns the exception instance if failed; it raises
  `CancelledError` if the task was cancelled.

## `create_task()` Semantics and Why You Keep References

[`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)
starts concurrency, but it does not create structure by itself.

Two practical reasons to keep a strong reference to tasks you spawn:

- Lifetime/cleanup control: without a reference you cannot reliably await it,
  cancel it, or coordinate shutdown.
- Failure visibility: if a task raises and nobody ever retrieves the exception
  (via `await`, `task.result()`, or `task.exception()`), asyncio logs "Task
  exception was never retrieved".

There is also a more literal reason: the event loop only keeps weak references
to tasks. A task that isn’t referenced elsewhere may get garbage collected at
any time, even before it’s done (the official docs call this out explicitly).

Reference:

- [`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)
- [`asyncio.all_tasks()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.all_tasks)

If you truly want "fire-and-forget" background tasks, keep them in a
collection and drop the reference when done:

```python
import asyncio


background_tasks: set[asyncio.Task[object]] = set()


async def start_background() -> None:
    task = asyncio.create_task(asyncio.sleep(1))
    background_tasks.add(task)
    task.add_done_callback(background_tasks.discard)
```

Docs:

- [`Task.add_done_callback()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.add_done_callback)
- [`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)

## Awaiting a Task vs Awaiting a Coroutine

Awaiting a coroutine runs it within the current task.
Awaiting a task waits for a concurrently scheduled task.

```python
import asyncio


async def child(label: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return label


async def sequential() -> list[str]:
    a = await child("a", 0.2)
    b = await child("b", 0.2)
    return [a, b]


async def concurrent() -> list[str]:
    ta = asyncio.create_task(child("a", 0.2))
    tb = asyncio.create_task(child("b", 0.2))
    return [await ta, await tb]
```

Cancellation interaction to remember: if the current task is cancelled while it
is awaiting another task/future, the cancellation request is propagated into
the awaited future (unless you use
[`asyncio.shield()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.shield)).

## Task Cancellation Basics

Cancellation in asyncio is cooperative and is delivered as
`asyncio.CancelledError` at an `await` point.

Key APIs:

- [`Task.cancel()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancel)
- [`Task.cancelled()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancelled)
- [`Task.cancelling()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancelling)
- [`Task.uncancel()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.uncancel)

### `cancel()` vs `cancelled()`

- `task.cancel()` requests cancellation. It does not guarantee the task stops
  immediately (or at all) if the task catches `CancelledError` and continues.
- `task.cancelled()` is only true once the task has finished in the cancelled
  state.

### Multiple cancellation requests: `cancelling()` and `uncancel()`

Modern asyncio tasks can have a cancellation request count.
`task.cancelling()` tells you how many requests are pending.

`task.uncancel()` decrements that count.

Most application code should not call `uncancel()`. It is primarily used by
asyncio internals and helpers (notably timeouts) to create well-defined
cancellation boundaries.

## TaskGroup Semantics (Structured Concurrency)

[`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup)
is the structured-concurrency primitive in Python 3.11+.

It is an async context manager:

```python
import asyncio


async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(asyncio.sleep(1))
        tg.create_task(asyncio.sleep(2))
    # Exiting the block waits for both tasks.
```

API reference:

- [`TaskGroup.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup.create_task)

### Lifetime rules

- Tasks created by `tg.create_task(...)` are children of the TaskGroup.
- The `async with` scope does not exit successfully until all children are
  finished.
- A TaskGroup is not reusable: once the context manager is exiting/exited, you
  cannot create new tasks in it.

### Creating tasks only while active

If you call `TaskGroup.create_task()` when the group is not active, it raises
an exception (typically `RuntimeError`).

In modern asyncio, the coroutine object you pass is closed if it cannot be
scheduled because the group is inactive. This avoids “coroutine was never
awaited” warnings, but it also means the following is a bug:

```python
import asyncio


async def bad(tg: asyncio.TaskGroup) -> None:
    coro = asyncio.sleep(1)
    tg.create_task(coro)  # if tg is inactive, this raises and coro gets closed
```

Prefer creating the coroutine object at the point you spawn it:

```python
tg.create_task(asyncio.sleep(1))
```

### Exception propagation and sibling cancellation

If any child task raises an exception (other than a normal cancellation), the
TaskGroup:

1. Requests cancellation of the remaining child tasks (siblings).
2. Waits for them to finish.
3. Re-raises at the `async with` boundary.

If multiple child tasks raise, the TaskGroup raises an
[`ExceptionGroup`](https://docs.python.org/3/library/exceptions.html#ExceptionGroup).

Handling aggregated errors uses `except*` (PEP 654):

- `ExceptionGroup` docs:
  https://docs.python.org/3/library/exceptions.html#ExceptionGroup
- `try` / `except*` syntax docs:
  https://docs.python.org/3/reference/compound_stmts.html#the-try-statement

Example:

```python
import asyncio


class FetchError(RuntimeError):
    pass


async def fail(msg: str) -> None:
    raise FetchError(msg)


async def main() -> None:
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(fail("users"))
            tg.create_task(fail("repos"))
    except* FetchError as eg:
        for exc in eg.exceptions:
            print("fetch failed:", exc)
```

### Cancellation interactions

There are two common cancellation paths around a TaskGroup:

1. A child fails: the group cancels siblings as part of error handling.
2. The parent scope is cancelled: the group cancels all children as part of
   unwinding.

In either case, child code must treat `CancelledError` as a control-flow
signal and generally re-raise it after cleanup. Swallowing cancellation can
cause a TaskGroup to hang while waiting for a task that refuses to stop.

See also:

- [`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)
- [`asyncio.timeout()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.timeout)

## Contrast: TaskGroup vs gather/wait/as_completed

These APIs all run multiple awaitables, but their semantics differ.

### `asyncio.gather()`

- Docs: [`asyncio.gather()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)
- Returns results in the same order as inputs.
- If one awaitable raises, `gather()` raises that exception.
- Crucially: other awaitables are not cancelled just because one failed (by
  default).

### `asyncio.wait()`

- Docs: [`asyncio.wait()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.wait)
- Low-level: splits a set of tasks/futures into `done` and `pending`.
- Does not propagate exceptions automatically.

### `asyncio.as_completed()`

- Docs: [`asyncio.as_completed()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.as_completed)
- Lets you consume results as inputs finish.
- You are responsible for cancellation, error handling, and deciding when to
  stop.

## When to Use What

| Need | Prefer | Why |
| --- | --- | --- |
| A scoped unit of concurrent work that must not leak past a block | [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup) | Structured lifetime; sibling cancellation; aggregates errors |
| A long-lived background task with an explicit owner | [`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) | You want a handle you store and manage explicitly |
| Run N awaitables and collect results in input order; you will handle failures carefully | [`asyncio.gather()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather) | Simple API; but not structured on failure |
| Implement custom waiting policies (first done, partial completion) | [`asyncio.wait()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.wait) | Low-level control |
| Consume results as they finish | [`asyncio.as_completed()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.as_completed) | Completion-order iteration |

If you are not sure, default to `TaskGroup` for bounded concurrency inside an
operation.
