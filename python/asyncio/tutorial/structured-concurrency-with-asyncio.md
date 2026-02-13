# Structured Concurrency With asyncio

Previous: [`first-async-program.md`](first-async-program.md)
Next: [`cancellation-and-lifetimes.md`](cancellation-and-lifetimes.md)
Core: [`../core/task-and-taskgroup-semantics.md`](../core/task-and-taskgroup-semantics.md), [`../core/cancellation-model.md`](../core/cancellation-model.md)

## The Problem: Unstructured Background Tasks

The easiest way to start concurrency in `asyncio` is
[`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task).
The easiest way to create bugs is also `asyncio.create_task()`.

If you create tasks and then lose track of them, you get unstructured
concurrency:

- Leaks: tasks keep running after the operation that started them has
  returned.
- Hidden exceptions: failures show up as "Task exception was never retrieved"
  (or get logged later, far away).
- Nondeterministic shutdown: the event loop stops while tasks are mid-flight,
  or shutdown becomes ad-hoc cancellation code.

This pattern looks harmless:

```python
import asyncio


async def do_work() -> None:
    await asyncio.sleep(0.2)
    raise RuntimeError("boom")


async def handler() -> str:
    asyncio.create_task(do_work())
    return "ok"  # the task is now someone else's problem
```

The caller of `handler()` has no idea there is still work running, no way to
wait for it, and no clear place to handle its errors.

## Trio In One Paragraph (Conceptual Comparison)

Trio bakes in structured concurrency: you spawn child tasks inside a nursery,
and the nursery scope guarantees they finish (or are cancelled) before the
scope exits. That makes lifetimes explicit and failure propagation predictable.
See Trio's docs on nurseries:
https://trio.readthedocs.io/en/stable/reference-core.html#nurseries-and-spawning

## The asyncio Solution: `asyncio.TaskGroup`

In modern `asyncio` (Python 3.11+),
[`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#task-groups)
is the structured concurrency primitive.

### 1) Lifetime scoping with `async with`

`TaskGroup` is an async context manager:

- Tasks created inside the `async with` block are owned by the block.
- When the block exits, the task group waits for all child tasks to finish.

### 2) Exception propagation + cancellation of siblings

If any child task fails with an exception:

- The task group cancels the remaining child tasks (siblings).
- It waits for them to acknowledge cancellation and exit.
- Then it re-raises the failure to the caller.

This is the key difference from "fire-and-forget": failure is not hidden, and
cleanup is coordinated.

### 3) Exception aggregation (`ExceptionGroup`) + `except*`

Multiple child tasks can fail around the same time (or fail during
cancellation). In that case, `TaskGroup` raises an
[`ExceptionGroup`](https://docs.python.org/3/library/exceptions.html#ExceptionGroup)
(PEP 654: https://peps.python.org/pep-0654/).

You handle these with `except*` (Python 3.11+), which matches and splits
exception groups by type.

Reference for `try`/`except*`:
https://docs.python.org/3/reference/compound_stmts.html#the-try-statement

Example:

```python
import asyncio


class FetchError(RuntimeError):
    pass


async def fail(name: str, started: asyncio.Event, go: asyncio.Event) -> None:
    started.set()
    await go.wait()
    raise FetchError(f"fetch {name} failed")


async def main() -> None:
    started1 = asyncio.Event()
    started2 = asyncio.Event()
    go = asyncio.Event()

    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(fail("users", started1, go))
            tg.create_task(fail("repos", started2, go))
            await started1.wait()
            await started2.wait()
            go.set()  # let both tasks fail "at once"
    except* FetchError as eg:
        for exc in eg.exceptions:
            print("fetch failed:", exc)
```

(`asyncio.Event` docs:
https://docs.python.org/3/library/asyncio-sync.html#asyncio.Event)

## Contrast: `asyncio.gather()` Is Not Structured

[`asyncio.gather()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)
is useful, but its failure semantics surprise people:

- If one awaitable fails, `gather()` raises that exception.
- The other awaitables are not cancelled by default; they keep running.

If you want "this whole operation is a unit; either it all succeeds or it all
gets cancelled together", prefer `TaskGroup`.

## Example 1: Parallel Fetch (Dummy I/O)

Problem: do multiple independent I/O operations concurrently, but make sure the
operation is all-or-nothing and doesn't leak work.

Solution: create tasks in a `TaskGroup`, then collect results after the scope.

```python
import asyncio


async def fake_fetch(name: str, delay_s: float) -> str:
    await asyncio.sleep(delay_s)
    return f"{name} (took {delay_s:.2f}s)"


async def main() -> None:
    requests = [("users", 0.30), ("repos", 0.10), ("settings", 0.20)]

    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fake_fetch(name, delay)) for name, delay in requests]

    results = [t.result() for t in tasks]
    print(results)


if __name__ == "__main__":
    asyncio.run(main())
```

If any fetch raises, the others are cancelled and you get an exception
(possibly an `ExceptionGroup`) at the `async with` boundary.

## Example 2: Fan-Out Workers, One Fails

Problem: you have a small pool of worker tasks consuming jobs. If any worker
hits a fatal error, you want the whole operation to stop, and you want a clear
place where the failure is raised.

Solution: keep the workers inside a `TaskGroup`. A failure cancels siblings and
propagates (possibly as an `ExceptionGroup`).

(`asyncio.Queue` docs: https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue)

```python
import asyncio


async def producer(q: asyncio.Queue[int | None], n_workers: int) -> None:
    for i in range(10):
        await q.put(i)
    for _ in range(n_workers):
        await q.put(None)  # sentinel: shutdown


async def worker(name: str, q: asyncio.Queue[int | None]) -> None:
    while True:
        item = await q.get()
        try:
            if item is None:
                return

            await asyncio.sleep(0.1)

            if item == 5 and name == "w2":
                raise RuntimeError("corrupt input 5")
        finally:
            q.task_done()


async def main() -> None:
    q: asyncio.Queue[int | None] = asyncio.Queue()
    n_workers = 3

    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(producer(q, n_workers))
            for name in ["w1", "w2", "w3"]:
                tg.create_task(worker(name, q))
    except* RuntimeError as eg:
        for exc in eg.exceptions:
            print("worker failure:", exc)


if __name__ == "__main__":
    asyncio.run(main())
```

What happens when `w2` raises:

- The task group cancels the other workers (and the producer task, if it's
  still running).
- Workers blocked on `q.get()` are cancelled instead of hanging forever.
- The `async with` exits by raising an `ExceptionGroup` containing the
  `RuntimeError` (and any other failures that occurred concurrently).

## Guidance For Python < 3.11

If you can't use `asyncio.TaskGroup`, you can approximate the shape of
structured concurrency, but you must be explicit about cancellation.

Sketch:

```python
import asyncio


async def run_scoped(*coros):
    tasks = [asyncio.create_task(c) for c in coros]
    try:
        return await asyncio.gather(*tasks)
    except Exception:
        for t in tasks:
            t.cancel()
        await asyncio.gather(*tasks, return_exceptions=True)
        raise
```

(`asyncio.CancelledError` docs:
https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)

This is easy to get subtly wrong (especially once you add timeouts,
shielding/cleanup, and nested operations), which is why `TaskGroup` is such a
big deal.
