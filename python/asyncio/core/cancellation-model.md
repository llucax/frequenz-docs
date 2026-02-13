# Cancellation Model

Tutorial: [`../tutorial/cancellation-and-lifetimes.md`](../tutorial/cancellation-and-lifetimes.md)
Core: [`resource-management.md`](resource-management.md)

This page describes modern `asyncio` cancellation semantics (Python 3.11+),
focusing on how cancellation is requested, where it is delivered, and how it
interacts with timeouts, shielding, and structured concurrency.

## What Cancellation Is

In `asyncio`, cancellation is cooperative:

1. Someone requests that a `Task` stop (typically by calling
   [`Task.cancel()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancel)).
2. The event loop delivers that request by injecting an
   [`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)
   into the task at the next checkpoint.
3. Normal exception propagation unwinds the stack; `finally` blocks and context
   manager cleanup run; if not intercepted, the task finishes in the cancelled
   state.

Cancellation is therefore an exception-driven control-flow path. Code must
assume it can be interrupted at awaits.

## `CancelledError` Inheritance (and Why It Matters)

`asyncio.CancelledError` inherits from `BaseException`, not `Exception`.

Practical consequences:

- `except Exception:` will not catch cancellation (good default).
- `except BaseException:` and bare `except:` will catch cancellation; if you
  use them, re-raise `CancelledError` unless you have a very specific boundary
  reason to convert it.

Docs:

- [`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)

## Where Cancellation Can Be Delivered

Cancellation is only delivered when a task returns control to the event loop.
In practice, that means cancellation is delivered at checkpoints such as:

- `await` points that actually suspend the task (waiting for I/O, timers, other
  tasks, locks, queues, ...)

What does not get interrupted:

- CPU-bound Python code running without an `await` (tight loops, long
  synchronous parsing/compression, large in-memory transforms)

Implication for CPU-heavy coroutines:

- periodically insert an `await` to create a checkpoint (for example
  `await asyncio.sleep(0)`), or
- offload work to a thread (`asyncio.to_thread`) or an executor

Docs:

- [`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)
- [`asyncio.to_thread()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.to_thread)

## Why Swallowing `CancelledError` Is Dangerous

If a coroutine catches `CancelledError` and continues running (or returns a
value), it breaks the assumptions other code uses to manage lifetimes.

This is especially harmful with:

- [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup):
  cancellation is how sibling tasks are asked to stop.
- timeouts: `asyncio.timeout()` and `asyncio.wait_for()` rely on cancellation
  to enforce deadlines.

Common failure modes when cancellation is swallowed:

- timeouts appear to “not work”
- `TaskGroup` hangs on exit waiting for a task that won’t stop
- shutdown takes arbitrarily long and violates deadlines

Recommended pattern:

```python
import asyncio


async def worker() -> None:
    try:
        ...
    except asyncio.CancelledError:
        # minimal cleanup
        raise
```

## Cancellation Counts and `uncancel()`

Asyncio tracks how many cancellation requests are pending for a task.

- Each call to `Task.cancel()` increments a cancellation counter.
- [`Task.cancelling()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancelling)
  returns the current count.
- [`Task.uncancel()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.uncancel)
  decrements the counter.

Why this exists:

- multiple independent sources can request cancellation (user cancel, TaskGroup
  shutdown, timeout firing)
- nested timeouts need to cancel the current task and later “consume” only the
  cancellation they created when translating it to `TimeoutError`

Guidance:

- Most application and library code should not call `uncancel()`.
- Misuse can accidentally clear someone else’s cancellation request and violate
  structured-concurrency expectations.

## Shielding (`asyncio.shield`)

[`asyncio.shield(aw)`](https://docs.python.org/3/library/asyncio-task.html#asyncio.shield)
prevents cancellation of the current task from cancelling the awaited
awaitable.

What shielding does:

- If the current task is cancelled while it is awaiting `shield(aw)`, the
  current task still gets `CancelledError`.
- But the awaited `aw` is not cancelled because of that outer cancellation.

What shielding does not do:

- It does not make the current task uncancellable.
- It does not protect `aw` from being cancelled directly by something else.

Shielding is sharp: it can easily create background work that outlives the
scope that started it. Use it deliberately.

## Timeouts

Asyncio provides timeout helpers that are implemented using cancellation.

### `asyncio.timeout()` and `asyncio.timeout_at()`

[`asyncio.timeout()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.timeout)
and
[`asyncio.timeout_at()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.timeout_at)
are async context managers that:

- schedule cancellation of the current task at the deadline
- during the timed block, the task experiences `CancelledError` like any other
  cancellation
- at the context-manager boundary, translate that timeout-driven cancellation
  into a
  [`TimeoutError`](https://docs.python.org/3/library/exceptions.html#TimeoutError)

Nesting semantics:

- If an inner timeout expires first, it cancels the task and converts its own
  cancellation into `TimeoutError` at the inner boundary.
- Cancellation counts are used so one timeout does not accidentally consume
  another cancellation request.

### `asyncio.wait_for()`

[`asyncio.wait_for(aw, timeout)`](https://docs.python.org/3/library/asyncio-task.html#asyncio.wait_for)
waits for `aw` with a deadline.

On timeout:

- it cancels the awaited operation (or its task)
- it raises `TimeoutError`

If you do not want the underlying operation to be cancelled when the timeout
expires, wrap it in `asyncio.shield()`.

Note: `wait_for()` waits until the future is actually cancelled, so the total
wait time may exceed the configured timeout.

## External vs Internal Cancellation (Timeouts, `TaskGroup`)

Timeouts and `TaskGroup` use the same underlying mechanism as user
cancellation: they call `Task.cancel()`.

This means:

- A task can have multiple active cancellation requests at once.
- The cancellation counter reflects that reality.

Behavioral rules to rely on:

- A timeout context converts cancellation into `TimeoutError` only for the
  cancellation it created.
- In a `TaskGroup`, swallowing cancellation delays group shutdown and can cause
  hangs or deadline overruns.

## Brief Comparison: Trio Cancellation Scopes

Trio models cancellation using explicit cancellation scopes (`CancelScope`).
Timeouts are cancellation scopes with deadlines, and shielding is expressed as a
scope option.

Asyncio uses task cancellation plus cancellation counts to emulate some nested
scope behavior (notably for `asyncio.timeout()`), but the scope object is not
explicit in the type system.

Trio docs:

- https://trio.readthedocs.io/en/stable/reference-core.html#cancellation-and-timeouts
