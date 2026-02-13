# Building Real Applications

Previous: [`cancellation-and-lifetimes.md`](cancellation-and-lifetimes.md)
Core: [`../core/resource-management.md`](../core/resource-management.md), [`../core/communication-and-synchronization.md`](../core/communication-and-synchronization.md), [`../core/task-and-taskgroup-semantics.md`](../core/task-and-taskgroup-semantics.md)

This chapter builds a small but realistic asyncio application end-to-end, using
only the standard library.

We'll build a line-based TCP chat server:

- Multiple concurrent clients (each connection has a reader and a writer task)
- Structured concurrency with
  [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup)
- Streams via
  [`asyncio.start_server()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.start_server)
- Backpressure via bounded
  [`asyncio.Queue`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue)
- Deterministic shutdown

We will not cover deployment, observability, or performance tuning.

## Step 1: The Smallest Useful Server (Echo)

Start with the simplest possible server: one task per connection, read a line,
write it back.

Key APIs:

- [`asyncio.start_server()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.start_server)
- [`StreamReader.readline()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamReader.readline)
- [`StreamWriter.write()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.write)
- [`StreamWriter.drain()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.drain)
- [`StreamWriter.close()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.close)
- [`StreamWriter.wait_closed()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.StreamWriter.wait_closed)

```python
import asyncio


async def handle_echo(
    reader: asyncio.StreamReader,
    writer: asyncio.StreamWriter,
) -> None:
    try:
        while True:
            data = await reader.readline()
            if not data:
                return

            writer.write(data)
            await writer.drain()
    finally:
        writer.close()
        await writer.wait_closed()


async def main() -> None:
    server = await asyncio.start_server(handle_echo, host="127.0.0.1", port=25000)
    async with server:
        await server.serve_forever()


if __name__ == "__main__":
    asyncio.run(main())
```

This works, but it has a problem that becomes obvious as soon as you add shared
state: where do you put background tasks, and how do you shut down
deterministically?

## Step 2: Split a Connection Into Reader + Writer Tasks

Real servers usually have two flows:

- reading input from the client
- writing output to the client

If you do both in one loop, your reads can stall writes (or vice versa).
Instead, we create an outgoing queue per client, and a dedicated writer task
that drains it.

Key APIs:

- [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup)
- [`asyncio.Queue`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue)

```python
import asyncio


async def writer_loop(
    writer: asyncio.StreamWriter,
    outgoing: asyncio.Queue[str | None],
) -> None:
    try:
        while True:
            msg = await outgoing.get()
            if msg is None:
                return
            writer.write(msg.encode("utf-8"))
            await writer.drain()
    finally:
        writer.close()
        await writer.wait_closed()


async def reader_loop(
    reader: asyncio.StreamReader,
    outgoing: asyncio.Queue[str | None],
) -> None:
    while True:
        data = await reader.readline()
        if not data:
            outgoing.put_nowait(None)
            return
        outgoing.put_nowait(f"you said: {data.decode('utf-8', errors='replace')}")


async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter) -> None:
    outgoing: asyncio.Queue[str | None] = asyncio.Queue()
    async with asyncio.TaskGroup() as tg:
        tg.create_task(writer_loop(writer, outgoing))
        tg.create_task(reader_loop(reader, outgoing))
```

This is already more realistic: the connection's tasks live and die together.

## Step 3: Add Shared State and a Broadcast API

Now we implement a tiny chat server:

- Each line from a client is broadcast to all other clients
- The shared state owns the set of clients
- Each connection registers/unregisters itself in a `finally` block

One important rule: do not hold a lock while doing an `await` that can block
for arbitrary time. We'll take a snapshot of the current clients and then await
queue operations outside the lock.

Key API:

- [`asyncio.Lock`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Lock)

Sketch:

```python
import asyncio


class State:
    def __init__(self) -> None:
        self._clients: set[asyncio.Queue[str]] = set()
        self._lock = asyncio.Lock()

    async def add(self, outgoing: asyncio.Queue[str]) -> None:
        async with self._lock:
            self._clients.add(outgoing)

    async def remove(self, outgoing: asyncio.Queue[str]) -> None:
        async with self._lock:
            self._clients.discard(outgoing)

    async def broadcast(self, line: str) -> None:
        async with self._lock:
            targets = list(self._clients)
        for q in targets:
            await q.put(line)
```

## Step 4: Backpressure With a Bounded Per-Client Queue

If you broadcast to a slow client as fast as you receive messages, you have two
choices:

- buffer indefinitely (memory grows until something fails)
- apply backpressure (slow down producers when the consumer is slow)

We'll apply backpressure with a bounded outgoing queue:

```python
outgoing: asyncio.Queue[str | None] = asyncio.Queue(maxsize=32)
```

This makes `await outgoing.put(...)` a flow-control point.

## Step 5: Deterministic Shutdown

We want shutdown to be boring and repeatable:

1. Stop accepting new connections.
2. Tell existing connections to stop.
3. Close all stream writers and wait for them to close.
4. Wait for all tasks to finish.

Ctrl-C can cancel the main task under
[`asyncio.run()`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run),
so cleanup must be in `finally` blocks and must treat
[`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)
as an expected control flow signal.

For explicit signal-driven shutdown we can use:

- [`asyncio.get_running_loop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get_running_loop)
- [`loop.add_signal_handler()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.add_signal_handler)

## Full Listing

This is a complete, runnable chat server.

To try it out, run the server in one terminal. In another terminal, run the
simple client from the next section.

```python
from __future__ import annotations

import asyncio
import contextlib
import signal
from dataclasses import dataclass, field


Line = str
OutgoingItem = str | None  # None is a sentinel meaning "shut down"


@dataclass(eq=False)
class Client:
    writer: asyncio.StreamWriter
    outgoing: asyncio.Queue[OutgoingItem]
    nickname: str
    closed: asyncio.Event = field(default_factory=asyncio.Event)


class ChatState:
    def __init__(self) -> None:
        self._clients: set[Client] = set()
        self._lock = asyncio.Lock()
        self.shutting_down = asyncio.Event()

    async def add(self, client: Client) -> None:
        async with self._lock:
            self._clients.add(client)

    async def remove(self, client: Client) -> None:
        async with self._lock:
            self._clients.discard(client)

    async def broadcast(self, line: Line, *, sender: Client | None = None) -> None:
        async with self._lock:
            targets = list(self._clients)

        for client in targets:
            if sender is not None and client is sender:
                continue
            if client.closed.is_set():
                continue

            # Backpressure lives here.
            await client.outgoing.put(line)

    async def close_all(self) -> None:
        self.shutting_down.set()
        async with self._lock:
            clients = list(self._clients)

        for client in clients:
            if not client.closed.is_set():
                with contextlib.suppress(asyncio.QueueFull):
                    client.outgoing.put_nowait(None)

            client.writer.close()


async def client_writer_loop(client: Client) -> None:
    try:
        while True:
            item = await client.outgoing.get()
            if item is None:
                return
            client.writer.write(item.encode("utf-8"))
            await client.writer.drain()
    except (ConnectionError, BrokenPipeError):
        return
    finally:
        client.closed.set()
        client.writer.close()
        with contextlib.suppress(Exception):
            await client.writer.wait_closed()


async def client_reader_loop(
    state: ChatState,
    client: Client,
    reader: asyncio.StreamReader,
) -> None:
    try:
        await client.outgoing.put("Welcome! Commands: /nick NAME\n")
        await state.broadcast(f"* {client.nickname} joined *\n", sender=None)

        while True:
            data = await reader.readline()
            if not data:
                return

            line = data.decode("utf-8", errors="replace").rstrip("\r\n")
            if not line:
                continue

            if line.startswith("/nick "):
                new = line.removeprefix("/nick ").strip()
                if not new:
                    await client.outgoing.put("Usage: /nick NAME\n")
                    continue
                old = client.nickname
                client.nickname = new
                await state.broadcast(f"* {old} is now {new} *\n")
                continue

            await state.broadcast(f"{client.nickname}: {line}\n", sender=client)
    finally:
        with contextlib.suppress(asyncio.QueueFull):
            client.outgoing.put_nowait(None)


async def run_connection(
    state: ChatState,
    reader: asyncio.StreamReader,
    writer: asyncio.StreamWriter,
) -> None:
    if state.shutting_down.is_set():
        writer.close()
        await writer.wait_closed()
        return

    client = Client(
        writer=writer,
        outgoing=asyncio.Queue(maxsize=32),
        nickname="anon",
    )

    await state.add(client)
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(client_writer_loop(client))
            tg.create_task(client_reader_loop(state, client, reader))
    except asyncio.CancelledError:
        raise
    finally:
        await state.remove(client)
        if not state.shutting_down.is_set():
            await state.broadcast(f"* {client.nickname} left *\n", sender=None)
        client.writer.close()
        with contextlib.suppress(Exception):
            await client.writer.wait_closed()
        client.closed.set()


async def connection_supervisor(
    tg: asyncio.TaskGroup,
    state: ChatState,
    accepted: asyncio.Queue[tuple[asyncio.StreamReader, asyncio.StreamWriter] | None],
) -> None:
    while True:
        item = await accepted.get()
        if item is None:
            return
        reader, writer = item
        tg.create_task(run_connection(state, reader, writer))


def build_signal_stop_event() -> asyncio.Event:
    stop = asyncio.Event()
    loop = asyncio.get_running_loop()

    for sig in (signal.SIGINT, signal.SIGTERM):
        with contextlib.suppress(NotImplementedError):
            loop.add_signal_handler(sig, stop.set)

    return stop


async def main(host: str = "127.0.0.1", port: int = 25000) -> None:
    state = ChatState()
    accepted: asyncio.Queue[tuple[asyncio.StreamReader, asyncio.StreamWriter] | None]
    accepted = asyncio.Queue()

    stop = build_signal_stop_event()

    def on_connect(reader: asyncio.StreamReader, writer: asyncio.StreamWriter) -> None:
        if state.shutting_down.is_set():
            writer.close()
            return
        accepted.put_nowait((reader, writer))

    server = await asyncio.start_server(on_connect, host=host, port=port)
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(server.serve_forever())
            tg.create_task(connection_supervisor(tg, state, accepted))
            await stop.wait()
    finally:
        state.shutting_down.set()

        server.close()
        with contextlib.suppress(asyncio.QueueFull):
            accepted.put_nowait(None)

        while True:
            with contextlib.suppress(asyncio.QueueEmpty):
                item = accepted.get_nowait()
                if item is None:
                    continue
                _r, w = item
                w.close()
                continue
            break

        await state.close_all()
        await server.wait_closed()


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        pass
```

## A Minimal Test Client

This client connects, sends a few lines, then reads responses for a short
period and exits. It uses the same streams APIs as the server.

Key API: [`asyncio.open_connection()`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.open_connection)

```python
import asyncio


async def run_client(host: str = "127.0.0.1", port: int = 25000) -> None:
    reader, writer = await asyncio.open_connection(host, port)
    try:
        for line in ["/nick alice\n", "hello\n", "goodbye\n"]:
            writer.write(line.encode("utf-8"))
            await writer.drain()

        # Read whatever the server sends for a moment.
        async with asyncio.timeout(1.0):
            while True:
                data = await reader.readline()
                if not data:
                    return
                print(data.decode("utf-8", errors="replace"), end="")
    except TimeoutError:
        return
    finally:
        writer.close()
        await writer.wait_closed()


if __name__ == "__main__":
    asyncio.run(run_client())
```

## Common Failure Modes

- Task leaks: creating background tasks without a lifetime owner (prefer
  `asyncio.TaskGroup`).
- Swallowed cancellation: catching `asyncio.CancelledError` and continuing work.
- Forgetting `drain()`: writing a lot without awaiting `StreamWriter.drain()`.
- Blocking the event loop: calling `time.sleep()` (or CPU-heavy work) inside
  async code.
- Unbounded buffers: accumulating per-client output without a limit.

## Where To Go Next

- Resource lifetimes: [`../core/resource-management.md`](../core/resource-management.md)
- Communication patterns: [`../core/communication-and-synchronization.md`](../core/communication-and-synchronization.md)
- TaskGroup semantics: [`../core/task-and-taskgroup-semantics.md`](../core/task-and-taskgroup-semantics.md)

## Appendix: `asyncio.Runner` and Ctrl-C

Most programs can just use `asyncio.run()`. If you need more control over the
event loop lifetime (for example, running multiple top-level coroutines in one
process), see
[`asyncio.Runner`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.Runner).

One common pattern looks like:

```python
import asyncio


def main_sync() -> None:
    with asyncio.Runner() as runner:
        try:
            runner.run(main())
        except KeyboardInterrupt:
            pass
```
