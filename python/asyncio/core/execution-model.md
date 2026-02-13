# Execution Model

Tutorial: [`../tutorial/introduction.md`](../tutorial/introduction.md), [`../tutorial/first-async-program.md`](../tutorial/first-async-program.md)
Core: [`mental-model.md`](mental-model.md), [`cancellation-model.md`](cancellation-model.md)

This page describes what the asyncio event loop does and what it means for how
your code runs: scheduling, fairness, and where cancellation/timeouts take
effect.

If you keep one sentence in mind, make it this:

- asyncio is cooperatively scheduled: a task runs until it reaches a suspension
  point that actually suspends.

Official reference entry points:

- Event loop APIs: https://docs.python.org/3/library/asyncio-eventloop.html
- Task scheduling and cancellation: https://docs.python.org/3/library/asyncio-task.html

## What The Event Loop Is Responsible For

An event loop iteration (“one turn of the loop”) has three conceptual jobs:

1. Run ready callbacks and resume ready tasks.
2. Poll for I/O readiness (network, subprocess pipes, etc.), and enqueue
   callbacks/tasks that can now make progress.
3. Maintain a timer schedule (deadlines/timeouts) and enqueue anything whose
   timer has expired.

This corresponds to three families of mechanisms exposed on the loop:

- Ready callbacks: [`loop.call_soon()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.call_soon)
- Timers: [`loop.call_later()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.call_later), [`loop.call_at()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.call_at)
- The loop's monotonic clock: [`loop.time()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.time)

## A Useful Pseudocode Model

This is not literal CPython code, but it’s a good approximation of observable
behavior:

```text
while running:
    ready.extend(timers.pop_all_due(now=loop.time()))

    timeout = 0 if ready else time_until_next_timer_or_none()
    ready.extend(poll_io(timeout))

    while ready:
        cb = ready.popleft()
        cb()
```

Two consequences fall straight out of this:

- If user code runs for a long time without suspending, the entire loop is
  blocked.
- Timeouts and cancellation are not “out-of-band interrupts”: they are
  processed by the loop and delivered when the task reaches a checkpoint.

## Cooperative Scheduling: Tasks, Futures, And “Ready”

Asyncio concurrency is built around tasks:

- A [`Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) runs a
  coroutine inside the event loop.
- A task runs until it awaits something that is not done yet, at which point
  the task suspends.

“Ready” means “scheduled in the loop’s ready queue and allowed to run now”. A
task becomes ready when whatever it was waiting for completes.

## Fairness (What Is And Isn’t Guaranteed)

Asyncio provides cooperative fairness, not preemptive fairness:

- The loop runs one callback/task at a time in its thread.
- There is no time slicing: if a callback/task doesn’t suspend, nothing else
  runs.

Within that constraint, ready callbacks are generally processed in registration
order:

- [`loop.call_soon()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.call_soon)
  specifies callbacks are called in the order they are registered.

But you should still treat fairness as “best effort”. A task can starve others
by not suspending.

## Yield Points / Checkpoints

The event loop can only switch between tasks when the current task suspends.
That means yield points matter for:

- responsiveness
- cancellation delivery

### Why `await` Isn’t Always A Suspension

Whether `await x` suspends depends on `x`:

- If `x` is pending, your task suspends and the loop can run other work.
- If `x` is already done, `await` can continue immediately.

You cannot use “some `await` happens here” as an implicit fairness or
cancellation checkpoint.

### Why `asyncio.sleep(0)` Is Explicit

[`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)
is documented to always suspend the current task.

Setting the delay to `0` is an optimized “yield to the loop” path:

```python
await asyncio.sleep(0)
```

Use this sparingly and comment why. If you need it frequently, it’s often a
sign you should refactor CPU-heavy work or blocking calls out of the event-loop
thread.

## Where Cancellation And Timeouts Hook In

Cancellation and timeouts are implemented via scheduling and cancellation.

### Task cancellation

Calling [`Task.cancel()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancel)
requests cancellation.

Operationally:

- The task is marked as cancelled.
- At the next opportunity (typically the next time the task is resumed from an
  `await`), asyncio injects an
  [`asyncio.CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError)
  into the coroutine.

See also: [`cancellation-model.md`](cancellation-model.md).

### Timeouts

- [`asyncio.timeout()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.timeout)
  schedules a deadline and cancels the current task if it expires; at the
  context-manager boundary, it converts the internal cancellation into
  `TimeoutError`.
- [`asyncio.wait_for()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.wait_for)
  cancels the awaited operation and raises `TimeoutError` on expiry.

Two implications:

- A timeout cannot “interrupt” CPU-bound Python code; it takes effect once the
  task hits a suspension point.
- If code swallows `CancelledError`, timeout semantics can break.

## Getting The Running Loop (And Why `asyncio.run()` Is Preferred)

Most application code should not manage event loops directly.

### Preferred: `asyncio.run()`

[`asyncio.run()`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run)
creates a loop, runs your top-level awaitable to completion, finalizes
asynchronous generators, shuts down the default executor, and closes the loop.

It also optionally enables debug mode (`debug=True`).

### Inside async code: `get_running_loop()`

Inside a coroutine or callback, use
[`asyncio.get_running_loop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get_running_loop).

### `get_event_loop()` behavior has been changing

[`asyncio.get_event_loop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get_event_loop)
has complex behavior and is easy to misuse. In coroutines/callbacks, prefer
`get_running_loop()`. At the top level, prefer `asyncio.run()`.

Version note (brief, but important):

- The asyncio policy system is deprecated and scheduled for removal; see the
  note in the docs for `get_event_loop()` and `asyncio.run()`.
- As of Python 3.14, `get_event_loop()` raises `RuntimeError` if there is no
  current event loop.

If you are writing a library that genuinely needs to create/manage loops,
prefer explicit APIs like
[`asyncio.new_event_loop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.new_event_loop)
and
[`asyncio.set_event_loop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.set_event_loop),
or the high-level
[`asyncio.Runner`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.Runner).

## Debug Mode And “The Loop Is Blocked”

“The loop is blocked” usually means: the loop is alive, but it can’t get back
to its scheduling cycle because some callback/task is running for too long
without suspending.

Enable debug mode using any of:

- [`PYTHONASYNCIODEBUG=1`](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONASYNCIODEBUG)
- `asyncio.run(..., debug=True)`
- [`loop.set_debug(True)`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.set_debug)

Official docs: https://docs.python.org/3/library/asyncio-dev.html#debug-mode

Debug mode can help you find:

- slow callbacks (threshold controlled by
  [`loop.slow_callback_duration`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.slow_callback_duration))
- tasks that don't yield (symptoms: late timers, delayed cancellation)

When you see blocked-loop behavior, the root cause is almost always one of:

- CPU-bound work running directly in the event loop thread
- a synchronous/blocking call inside async code
- a coroutine that logically "waits" but doesn’t hit a real suspension point
  (e.g., it keeps awaiting already-completed awaitables)
