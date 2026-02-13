# Asyncio Mental Model

Tutorial: [`../tutorial/first-async-program.md`](../tutorial/first-async-program.md), [`../tutorial/structured-concurrency-with-asyncio.md`](../tutorial/structured-concurrency-with-asyncio.md)
Next core: [`execution-model.md`](execution-model.md), [`task-and-taskgroup-semantics.md`](task-and-taskgroup-semantics.md)

This page aims to make `asyncio` feel mechanically predictable.

If you can answer these two questions for a given line of code, most asyncio
bugs become obvious:

- Who owns the lifetime of this work?
- Where are the suspension points where other work can run?

Official reference: https://docs.python.org/3/library/asyncio.html

## Core Terms (Precise but Practical)

### Event loop

The event loop is the runtime that:

- decides which coroutine/task runs next (scheduling)
- waits for external events (timers, I/O readiness) and turns them into
  resumptions
- runs callbacks and advances ready tasks until they hit a suspension point

Asyncio’s event loop is cooperative: it does not preempt your Python code.
If a task runs CPU-bound Python without reaching a suspension point, nothing
else runs.

Docs entry points:

- [`asyncio.run()`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run)
- [`asyncio.get_running_loop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get_running_loop)

### Coroutine function (`async def`)

A coroutine function is an `async def` function.

- Calling it does not run its body.
- Calling it returns a coroutine object.

Language reference: [`async def`](https://docs.python.org/3/reference/compound_stmts.html#async-def).

### Coroutine object

A coroutine object is the value produced by calling a coroutine function. It
represents “a suspended computation that can be driven forward”.

Important property: a coroutine object is inert until something drives it:

- `await coro_obj` drives it from within another coroutine, or
- scheduling it as a task asks the event loop to drive it.

Asyncio docs: [Coroutines and Tasks](https://docs.python.org/3/library/asyncio-task.html).

### Awaitable

An awaitable is any object you can use with `await`.

- Coroutine objects are awaitables.
- [`asyncio.Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) and
  [`asyncio.Future`](https://docs.python.org/3/library/asyncio-future.html#asyncio.Future)
  are awaitables.

Asyncio docs: [Awaitables](https://docs.python.org/3/library/asyncio-task.html#awaitables).

Language reference: the [`await` expression](https://docs.python.org/3/reference/expressions.html#await).

### Future

An [`asyncio.Future`](https://docs.python.org/3/library/asyncio-future.html#asyncio.Future)
is a low-level “eventual result” object:

- it will become done later, with either a result, an exception, or
  cancellation
- awaiting it suspends until it becomes done (unless it is already done)

Mental model: a Future is to the event loop what a “promise” is in other
ecosystems.

Application note: most application code should not create Futures manually.
They are typically created and completed by asyncio internals or libraries.

Also note: `asyncio.Future` is distinct from
[`concurrent.futures.Future`](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.Future).

### Task

An [`asyncio.Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task)
is a Future that wraps a coroutine object and drives it forward on the event
loop.

The key addition a task provides is independent scheduled execution:

- a coroutine object needs someone to `await` it to make progress
- a task is registered with the event loop, so it can make progress
  concurrently with the code that created it

You typically create tasks with
[`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task).

## What It Means To “Run” a Coroutine

### `asyncio.run(...)` is an event-loop lifetime manager

[`asyncio.run()`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run)
is the normal top-level entry point:

- it creates an event loop
- it schedules your main coroutine to run
- it runs the loop until that coroutine completes
- it then shuts the loop down and closes it

Practical consequence: if your program needs to do async work, there should be
exactly one place where you transition from synchronous code into `asyncio`,
and it’s usually `asyncio.run(main())`.

### A coroutine runs only while it is being driven

Inside the loop, a coroutine alternates between:

- running Python code, and
- being suspended at an `await` that does not complete immediately

Nothing else in asyncio makes your coroutine progress.

## Where Concurrency Comes From

### Cooperative scheduling

Asyncio concurrency is cooperative:

- Tasks take turns when they hit suspension points.
- Tasks do not get time-sliced preemptively.

If one task runs a long CPU-bound loop without `await`, it will starve other
tasks.

### Suspension points are “where switching can happen”

`await x` is a request: “if this awaited thing is not ready, suspend me and let
someone else run; resume me when it becomes ready.”

Common suspension sources:

- timers like [`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)
- waiting for synchronization primitives (events, locks, queues)
- waiting for tasks or futures

Important nuance: not every `await` suspends.

- If the awaited object is already done, `await` continues immediately.
- So you cannot assume that “I used `await`, therefore other tasks ran.”

If you need an explicit checkpoint (rare, but sometimes useful), `await
asyncio.sleep(0)` forces a suspension (see the docs for
[`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)).

## Why Tasks Change the Story

### `await` is sequential; tasks enable overlap

Within one coroutine:

```python
await a()
await b()
```

`b()` cannot start until `a()` finishes.

Tasks let you start work and then choose when to wait for it:

- `t = asyncio.create_task(a())` asks the loop to begin running `a()` soon
- your coroutine continues until it hits a suspension point
- later, `await t` waits for `a()`’s result (or exception)

### In-flight work has a lifetime and an owner

Once you create a task, that work exists independently of the local stack frame
that created it.

This is the source of many real-world asyncio design issues:

- If you create a task and then lose track of it, you have unstructured
  concurrency.
- Structured concurrency tools make ownership explicit.

Modern asyncio provides [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#task-groups)
(Python 3.11+) to scope task lifetimes.

## Trio (Brief Conceptual Comparison)

Trio’s core abstraction is the nursery: child tasks are spawned into a scope
that guarantees they finish (or are cancelled) before the scope exits.

- Trio nurseries:
  https://trio.readthedocs.io/en/stable/reference-core.html#nurseries-and-spawning

In modern asyncio,
[`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#task-groups)
plays a similar conceptual role.

## Minimal Examples (No Networking)

### Example 1: Sequential vs concurrent work

```python
import asyncio


async def step(name: str, delay_s: float) -> None:
    print(f"{name}: start")
    await asyncio.sleep(delay_s)
    print(f"{name}: end")


async def main() -> None:
    # Sequential: total time is ~0.3s + 0.2s.
    await step("A", 0.3)
    await step("B", 0.2)

    # Concurrent: total time is ~max(0.3s, 0.2s).
    t1 = asyncio.create_task(step("C", 0.3))
    t2 = asyncio.create_task(step("D", 0.2))
    await t1
    await t2


if __name__ == "__main__":
    asyncio.run(main())
```

### Example 2: Not every `await` suspends

This example creates a Future, marks it done immediately, and then awaits it.
That `await` completes without giving other tasks a chance to run.

```python
import asyncio


async def background() -> None:
    await asyncio.sleep(0)
    print("background ran")


async def main() -> None:
    loop = asyncio.get_running_loop()

    fut = loop.create_future()
    fut.set_result("ready")

    asyncio.create_task(background())

    value = await fut
    print("after await fut:", value)

    # This checkpoint gives the background task a chance.
    await asyncio.sleep(0)


if __name__ == "__main__":
    asyncio.run(main())
```

### Example 3: TaskGroup makes ownership explicit

```python
import asyncio


async def compute(x: int) -> int:
    await asyncio.sleep(0.1)
    return x * 2


async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(compute(10))
        t2 = tg.create_task(compute(21))

    print(t1.result(), t2.result())


if __name__ == "__main__":
    asyncio.run(main())
```

## Mental Model Checks

- If you call an `async def` and don’t `await` the result (or schedule it), the
  work did not happen.
- If you create a task, you must be able to name who owns it and where it is
  awaited.
- “Concurrent” means “can make progress while I’m awaiting something else”,
  not “runs in parallel CPU time”.
- An `await` is a potential yield point; if the awaitable is already done, you
  might not yield at all.
- If something seems stuck, look for CPU-bound work or blocking I/O without
  hitting an `await`.

## Where To Go Next

- Execution mechanics: [`execution-model.md`](execution-model.md)
- Structured concurrency and error propagation: [`task-and-taskgroup-semantics.md`](task-and-taskgroup-semantics.md)
