# First Async Program

Previous: [`introduction.md`](introduction.md)
Next: [`structured-concurrency-with-asyncio.md`](structured-concurrency-with-asyncio.md)
Core: [`../core/mental-model.md`](../core/mental-model.md), [`../core/execution-model.md`](../core/execution-model.md)

This chapter builds the smallest useful intuition for `asyncio`:

- An `async def` function does not run when you call it.
- `await` is the moment your coroutine can pause and let other work run.
- Tasks are how you ask the event loop to run multiple coroutines concurrently.

## `async def` and Coroutine Objects

An `async def` function is an async function. When you call it, you do not
execute its body. Instead, you get a coroutine object (a value that represents
the work to do later).

```python
import asyncio


async def hello() -> None:
    print("hello")
    await asyncio.sleep(0.1)
    print("world")


def main_sync() -> None:
    c = hello()
    print(c)


if __name__ == "__main__":
    main_sync()
```

If you run that file, you'll see something like `<coroutine object hello at
0x...>`, and you will not see `hello`/`world`, because nothing ever drove the
coroutine forward.

To actually run a top-level coroutine, use
[`asyncio.run()`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run):

```python
import asyncio


async def hello() -> None:
    print("hello")
    await asyncio.sleep(0.1)
    print("world")


if __name__ == "__main__":
    asyncio.run(hello())
```

This is the first important separation to internalize:

- Calling `hello()` creates a coroutine object.
- Something else (the event loop) must run it.

## `await` Basics

Inside an `async def`, you can use `await`:

- to get the result of another coroutine
- and to (potentially) suspend, letting the event loop run something else

```python
import asyncio


async def add_one_later(x: int) -> int:
    await asyncio.sleep(0.1)
    return x + 1


async def main() -> None:
    y = await add_one_later(41)
    print(y)


if __name__ == "__main__":
    asyncio.run(main())
```

Docs: [`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep).

## Pitfall: Forgetting `await` (the RuntimeWarning)

The most common early bug is creating a coroutine object and then forgetting to
`await` it or schedule it as a task.

```python
import asyncio


async def work() -> None:
    await asyncio.sleep(0.1)
    print("done")


async def main() -> None:
    work()  # BUG: coroutine created, but never awaited


if __name__ == "__main__":
    asyncio.run(main())
```

Often you will get a warning like:

- `RuntimeWarning: coroutine 'work' was never awaited`

What it means:

- You created a coroutine object (by calling `work()`), but it was never driven
  forward.
- Your program probably silently skipped that async work.

Fixes:

- If you meant to run it now and wait for it: `await work()`.
- If you meant to start it concurrently: use a task (next section).

## Tasks: Starting Work Concurrently

So far, this is sequential:

```python
await a()
await b()
```

`b()` does not even start until `a()` finishes.

To ask the event loop to run a coroutine concurrently with your current one,
wrap it in a task using
[`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task):

```python
import asyncio


async def worker(name: str) -> None:
    for i in range(3):
        print(f"{name}: step {i}")
        await asyncio.sleep(0)  # deliberate yield point


async def main() -> None:
    t1 = asyncio.create_task(worker("A"))
    t2 = asyncio.create_task(worker("B"))

    # Important: keep a reference, and make sure tasks are awaited.
    await t1
    await t2


if __name__ == "__main__":
    asyncio.run(main())
```

When you run this, you'll typically see interleaving like:

- `A: step 0`
- `B: step 0`
- `A: step 1`
- `B: step 1`

Two key points:

- `create_task()` schedules the coroutine to run soon on the event loop.
- The task only makes progress when it gets CPU time, which happens at
  suspension points.

If you create a task and never await it, you are back in "fire-and-forget"
territory: errors can get logged later, lifetimes become unclear, and shutdown
becomes nondeterministic.

In modern asyncio, you'll usually prefer structured concurrency via
[`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup),
which the next chapter focuses on.

## Where Task Switching Can Happen (Yield Points / Checkpoints)

Asyncio is cooperative:

- The event loop does not preempt your Python code.
- Other tasks only get a chance to run when your task hits an `await` that
  actually suspends.

`await asyncio.sleep(0)` is a deliberate yield:

- It asks the event loop to resume you on the next loop iteration.
- It gives other ready tasks a chance to run.
- It creates a useful checkpoint for cancellation delivery.

Be careful with this mental model: not every `await` necessarily suspends. If
the thing you await is already complete (for example, awaiting a finished task
or a future that is already done), then `await` may continue immediately.

This is why you should not rely on "some await" for fairness or responsiveness.
If you need a clear checkpoint (rare, but sometimes useful), write it
explicitly and comment why.

Docs: [`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep).

## Code Review Checklist (Early asyncio)

- Every `async def` call is either awaited or wrapped in `asyncio.create_task()`.
- Tasks are not lost: task objects are kept, and their completion is awaited.
- Concurrency is intentional: you can explain why `create_task()` is needed.
- Yield points are explicit when they matter (e.g. `await asyncio.sleep(0)`),
  and you do not assume that "any await" will necessarily suspend.
- No blocking calls in async code (for example, `time.sleep()`); blocking
  prevents other tasks from running.
