# Resource Management in asyncio

Tutorial: [`../tutorial/building-real-applications.md`](../tutorial/building-real-applications.md)
Core: [`cancellation-model.md`](cancellation-model.md)

Async code does not get a free pass on lifetime management. It still has to
close files, sockets, subprocesses, and background tasks.

What changes with `asyncio` is that you now have concurrency, cancellation, and
partial progress to contend with.

## Why Resource Management Is Harder With Concurrency

In synchronous code, "cleanup" often means "run this when the function
returns".

In concurrent async code, you frequently have:

- multiple in-flight operations that share a resource
- work that outlives the scope that started it (background tasks)
- cancellation arriving at any `await`, including during cleanup
- structured shutdown requirements: stop accepting new work, wind down active
  work, then release resources

The result: you must be explicit about ownership and boundaries.

## The Two Primitives: `try/finally` and `async with`

Use `try/finally` for the most direct "do X, then always do Y" lifetime.
Use `async with` (async context managers) when a resource has a natural
acquire/release protocol.

Language docs:

- `try` / `finally`: https://docs.python.org/3/reference/compound_stmts.html#the-try-statement
- async context managers (`async with`): https://docs.python.org/3/reference/compound_stmts.html#the-async-with-statement

### `try/finally` around the exact ownership scope

```python
import asyncio


async def use_lock_safely(lock: asyncio.Lock) -> None:
    await lock.acquire()
    try:
        # access shared state
        ...
    finally:
        lock.release()
```

Guidelines:

- Acquire as late as possible; release as early as possible.
- Keep the `try` body small so the ownership boundary is obvious.

### Async context managers (`async with`)

Prefer `async with lock:` over manual acquire/release when possible:

```python
import asyncio


async def use_lock(lock: asyncio.Lock) -> None:
    async with lock:
        ...
```

Docs: [`asyncio.Lock`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Lock)

If a resource does not come as an async context manager, wrap it.

Standard-library helpers:

- `contextlib.asynccontextmanager`: https://docs.python.org/3/library/contextlib.html#contextlib.asynccontextmanager
- `contextlib.AsyncExitStack`: https://docs.python.org/3/library/contextlib.html#contextlib.AsyncExitStack

Example wrapper for a stream connection:

```python
from contextlib import asynccontextmanager
import asyncio


@asynccontextmanager
async def open_stream(host: str, port: int):
    reader, writer = await asyncio.open_connection(host, port)
    try:
        yield reader, writer
    finally:
        writer.close()
        await writer.wait_closed()
```

Docs: [`asyncio.open_connection()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.open_connection)

## Cleanup Under Cancellation

In asyncio, cancellation is delivered by raising
[`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)
into the task at an `await`. This affects cleanup in two ways:

1. `finally` blocks run.
2. Any `await` inside cleanup can itself be cancelled.

The default guidance is:

- Make cleanup idempotent and best-effort.
- Keep cleanup fast.
- Do not swallow `CancelledError` (let it keep propagating after cleanup).

If you have cleanup that must `await` (closing a stream, stopping a child
task), it’s fine to do it in `finally` — just assume it might be interrupted.

If you truly need “cleanup must complete even under cancellation”, you are
implementing a cancellation boundary. That is an advanced topic: it often
requires careful use of timeouts, shielding, and cancellation counts.
See [`cancellation-model.md`](cancellation-model.md).

## Working With Streams and Servers

### Streams: `close()` + `wait_closed()`

For stream writers, `close()` starts the close; `wait_closed()` lets you await
the completion of the close handshake and transport teardown.

```python
import asyncio


async def talk(host: str, port: int) -> bytes:
    reader, writer = await asyncio.open_connection(host, port)
    try:
        writer.write(b"hello")
        await writer.drain()
        return await reader.read(1024)
    finally:
        writer.close()
        await writer.wait_closed()
```

Docs:

- Streams overview: https://docs.python.org/3/library/asyncio-stream.html
- [`StreamWriter.close()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.close)
- [`StreamWriter.wait_closed()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.wait_closed)

### Servers: `server.close()` + `await server.wait_closed()`

Servers returned by `asyncio.start_server()` must be closed explicitly during
shutdown.

```python
import asyncio


async def run_server() -> None:
    server = await asyncio.start_server(lambda r, w: None, "127.0.0.1", 0)
    try:
        await server.serve_forever()
    finally:
        server.close()
        await server.wait_closed()
```

Docs:

- [`asyncio.start_server()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.start_server)
- [`Server.close()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.Server.close)
- [`Server.wait_closed()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.Server.wait_closed)

## Async Generators: Finalization and `aclose()`

Async generators often own resources inside a `try/finally`. Their `finally`
blocks run when the generator is finalized.

If you stop consuming early (break/return), explicit closure is the safest way
to ensure timely cleanup.

Language / library docs:

- Async generators: https://docs.python.org/3/reference/expressions.html#asynchronous-generator-iterators
- `agen.aclose()`: https://docs.python.org/3/reference/expressions.html#agen.aclose
- `contextlib.aclosing`: https://docs.python.org/3/library/contextlib.html#contextlib.aclosing

```python
from contextlib import aclosing


async def lines(reader):
    try:
        while True:
            line = await reader.readline()
            if not line:
                return
            yield line
    finally:
        ...


async def consume_some(reader) -> None:
    async with aclosing(lines(reader)) as agen:
        async for line in agen:
            if line == b"STOP\n":
                break
```

### What `asyncio.run()` does at shutdown

`asyncio.run()` manages loop lifetime for you. In addition to running your
main awaitable to completion, it performs shutdown steps such as:

- finalizing async generators
- shutting down the default executor
- closing the event loop

Docs:

- [`asyncio.run()`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run)
- [`loop.shutdown_asyncgens()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.shutdown_asyncgens)

Even with `asyncio.run()`, explicit ownership boundaries are still valuable:
they make cleanup happen when you intend, not “eventually at shutdown”.

## Patterns

### Own your children (structured concurrency)

If you start background tasks, something must:

- keep a reference so they are not lost
- cancel them when the owner is done
- await them so exceptions are observed

[`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#task-groups)
makes this explicit: child tasks live within the `async with` scope, and the
group coordinates cancellation and error propagation.

### Own your resources (context managers as boundaries)

Prefer “resources are created inside a context manager” over “resources are
created somewhere and maybe closed later”.

When acquisition is conditional or you need to manage multiple resources,
`AsyncExitStack` often keeps the code honest.
