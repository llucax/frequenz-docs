# Cancellation and Lifetimes

Previous: [`structured-concurrency-with-asyncio.md`](structured-concurrency-with-asyncio.md)
Next: [`building-real-applications.md`](building-real-applications.md)
Core: [`../core/cancellation-model.md`](../core/cancellation-model.md), [`../core/resource-management.md`](../core/resource-management.md)

Cancellation is the mechanism that makes lifetimes real in `asyncio`: it is how
a parent scope tells its children (and itself) "stop what you're doing and wind
down".

In `asyncio`, cancellation is cooperative: it does not kill code mid-bytecode.
It is a request delivered as an exception that your code sees at well-defined
points.

## What Cancellation Is

Cancellation in `asyncio` is a request delivered as
[`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError).

- Someone requests cancellation, typically by calling
  [`asyncio.Task.cancel()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancel).
- The cancelled task observes the request when it reaches an `await` that gives
  control back to the event loop.
- At that point the task gets `CancelledError`, which unwinds the stack like
  any other exception.

This is why cancellation is predictable: it is delivered at `await` points.

If you write a long CPU-bound loop with no `await`, cancellation cannot be
delivered until you yield. A common pattern is to add a periodic yield with
[`asyncio.sleep(0)`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)
to make the loop responsive.

Modern nuance: `asyncio.CancelledError` inherits from `BaseException`, not from
`Exception` (changed in Python 3.8). This reduces the risk that broad
`except Exception:` handlers swallow cancellation.

## Handling `CancelledError` Correctly

The default rule is simple:

- Use `try/finally` for cleanup.
- If you catch `asyncio.CancelledError`, you typically re-raise it.

The reason is semantic: cancellation is how timeouts and TaskGroup shutdown
work. If you swallow it, higher-level code can no longer enforce lifetimes.

Pattern:

```python
import asyncio


async def run_until_cancelled() -> None:
    resource = acquire_resource()  # synchronous setup
    try:
        while True:
            await do_one_step(resource)  # cancellation can be delivered here
            await asyncio.sleep(0.5)
    except asyncio.CancelledError:
        # Optional: record logs/metrics.
        # Important: do not swallow cancellation.
        raise
    finally:
        resource.close()
```

## Timeouts Are Structured Cancellation

`asyncio` offers two common timeout tools:

- [`asyncio.timeout()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.timeout):
  an async context manager that applies a deadline to a block.
- [`asyncio.wait_for()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.wait_for):
  wraps an awaitable with a deadline.

### `asyncio.timeout()`

With `asyncio.timeout()`, the event loop enforces a lifetime for a block:

```python
import asyncio


async def do_with_deadline() -> None:
    async with asyncio.timeout(2.0):
        await something_that_might_hang()
```

Semantics to internalize:

- When the deadline expires, `asyncio` cancels the current task.
- The cancellation is delivered as `asyncio.CancelledError` at an `await`.
- On exiting the `asyncio.timeout()` block, that internal cancellation is
  converted into
  [`TimeoutError`](https://docs.python.org/3/library/exceptions.html#TimeoutError).

Important pitfall:

- If code inside the timed block catches `asyncio.CancelledError` and keeps
  going, it can run past the deadline. A timeout is enforced via cancellation;
  swallowing cancellation breaks timeout semantics.

Also note the distinction between timeout cancellation and external
cancellation:

- `asyncio.timeout()` only converts the cancellation it generated.
- If an outer scope cancels the task (for example, a parent
  [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup)
  is exiting), you still get `asyncio.CancelledError`.

### `asyncio.wait_for()`

`asyncio.wait_for()` is often useful at API boundaries where you cannot easily
rewrite code into a timeout scope.

- On timeout, it cancels the awaited operation and raises `TimeoutError`.
- If the awaited operation suppresses cancellation and returns a value,
  `asyncio.wait_for()` may return that value instead of raising `TimeoutError`.
  This is another reason cancellation must not be swallowed.

## Shielding (And Why It Is Sharp)

[`asyncio.shield()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.shield)
protects an awaitable from being cancelled by the current task's cancellation.

What it does:

- Outer task cancellation does not cancel the inner awaitable.

What it does not do:

- It does not make the outer task immune. The outer task still receives
  `asyncio.CancelledError`.

Why this is sharp:

- If you shield something and then allow the outer task to unwind, the shielded
  operation can keep running in the background.

## TaskGroup Interplay: Cancellation Is Control Flow

In a
[`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup),
errors and shutdown are implemented by cancelling sibling tasks.

Two practical consequences:

- If a task in a TaskGroup catches `asyncio.CancelledError` and does not
  re-raise it, the TaskGroup can get stuck waiting for it.
- Timeouts around a TaskGroup rely on cancellation to stop work. Swallowing
  cancellation can defeat the timeout and turn a deadline into an unbounded
  wait.

Rule of thumb: treat `asyncio.CancelledError` like a "must propagate" control
signal, unless you are implementing a very intentional boundary.

## Advanced: Cancellation Counts and `Task.uncancel()`

Modern `asyncio` tasks can accumulate multiple cancellation requests.
APIs in this space include:

- [`asyncio.Task.cancelling()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancelling)
- [`asyncio.Task.uncancel()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.uncancel)

Most application code should not use these directly. They exist primarily for
utilities that must implement careful cancellation boundaries.

If you reach for them, write tests around timeout behavior and TaskGroup
shutdown.

## Example 1: A Cancellable Loop With Correct Cleanup

This example shows three basics:

- cancellation is observed at `await` points
- `finally` always runs
- `CancelledError` is re-raised if caught

```python
import asyncio


class Resource:
    def close(self) -> None:
        pass


def acquire_resource() -> Resource:
    return Resource()


async def do_one_step(resource: Resource) -> None:
    await asyncio.sleep(0.1)


async def worker() -> None:
    resource = acquire_resource()
    try:
        while True:
            await do_one_step(resource)
            await asyncio.sleep(0)  # keep loop cancellation-responsive
    except asyncio.CancelledError:
        raise
    finally:
        resource.close()
```

## Example 2: A Timeout-Wrapped Operation With Correct Exceptions

This example cleanly distinguishes:

- `TimeoutError`: the operation exceeded its deadline
- `asyncio.CancelledError`: the caller cancelled us

```python
import asyncio


async def slow_operation() -> str:
    await asyncio.sleep(10)
    return "ok"


async def main() -> str:
    try:
        async with asyncio.timeout(2.0):
            return await slow_operation()
    except TimeoutError:
        return "timed out"
    except asyncio.CancelledError:
        raise
```

## Conceptual Comparison: Trio Cancellation Scopes

If you've used Trio, `asyncio.timeout()` is the closest standard-library analog
to a Trio cancellation scope (deadline-bound lifetime), though the APIs and
guarantees differ.

Trio reference: https://trio.readthedocs.io/en/stable/reference-core.html#trio.CancelScope
