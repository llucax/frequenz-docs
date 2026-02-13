# Introduction to `asyncio`

This tutorial is a general-purpose guide to Python's standard-library async I/O
framework, [`asyncio`](https://docs.python.org/3/library/asyncio.html),
targeting modern Python (3.11+).

Teaching-style note: the structure here is mainly inspired by the Trio
tutorial/docs (especially how it introduces concepts before APIs):
https://trio.readthedocs.io/en/stable/

## What You Should Already Know

Before starting, you should be comfortable with:

- Writing and calling Python functions
- Exceptions and tracebacks
- Basic I/O concepts (network requests, file I/O) at a high level

You do *not* need prior experience with threads or event loops, but you should
be ready for a different execution model than "one statement at a time".

## What `asyncio` Is (and Isn't)

`asyncio` is a library for running many I/O-bound operations efficiently
within a single OS thread by switching between tasks while they wait on I/O.

The big idea is cooperative scheduling:

- Your code runs until it reaches an `await` point that is able to pause.
- At that pause, the event loop can run other tasks.
- Nothing preempts you automatically: if you never `await`, you never yield
  control.

This leads to core limitations you need to keep in mind:

- You must yield: if an `async def` function does not hit an `await` that can
  block, other tasks don't get a chance to run.
- CPU-bound work blocks the loop: a long-running calculation inside `async def`
  still runs on the CPU and can freeze all other async work.
- Async is not parallelism: concurrency (interleaving) is different from
  parallel execution on multiple cores.

## The "Async Sandwich"

A practical way to remember how `asyncio` code is structured is the
"async sandwich":

1. Synchronous entry: a normal Python script starts in normal (sync) code.
2. Async middle: you enter an async "world" to run coroutines and tasks.
3. Async primitives: inside that world you use `asyncio` building blocks like
   sleeping, tasks, timeouts, networking, etc.

In `asyncio`, the usual entry point is
[`asyncio.run()`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run):

- It creates and runs an event loop for you.
- It runs one top-level coroutine (or other awaitable) until completion.
- It finalizes async resources and closes the loop.

Once you're inside your async code (the middle of the sandwich), you write
`async def` functions and use `await` to cooperate with the scheduler.

## Hello, World (Minimal)

This is the smallest "shape" of an `asyncio` program:

```python
import asyncio


async def main() -> None:
    print("Hello from asyncio!")


if __name__ == "__main__":
    asyncio.run(main())
```

Key point: the only place you need `asyncio.run()` is at the top level of your
program (or at a well-defined boundary). Most of your code should just be
`async def` functions that can be called/awaited by other async functions.

## "Two Things at Once" (Interleaving With `sleep`)

To see cooperative scheduling, we'll run two tasks that each wait a different
amount of time. Waiting is represented with
[`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep),
which always yields control back to the event loop.

Python 3.11 introduced structured concurrency via
[`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup),
which is the recommended way to start and manage related tasks:

```python
import asyncio


async def say_after(delay: float, message: str) -> None:
    await asyncio.sleep(delay)
    print(message)


async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(say_after(1.0, "first"))
        tg.create_task(say_after(0.2, "second"))


if __name__ == "__main__":
    asyncio.run(main())
```

What to notice:

- Both tasks start quickly.
- The one that sleeps for `0.2` seconds prints first.
- Nothing is "multithreaded" here; the tasks take turns only at `await` points.

Version note: `asyncio.TaskGroup` is new in Python 3.11. In older versions,
you can create tasks with
[`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)
and wait for them (commonly with
[`asyncio.gather()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)),
but the semantics are not identical.

## Where To Go Next

- Next chapter: [`first-async-program.md`](first-async-program.md)
- Core background reading:
  - [`../core/mental-model.md`](../core/mental-model.md)
  - [`../core/execution-model.md`](../core/execution-model.md)
