# Asyncio: A Trio-Inspired Learning Guide

This is a general-purpose learning guide for Python's built-in `asyncio`.

The structure and teaching style are explicitly inspired by Trio's excellent
documentation:

- https://trio.readthedocs.io/en/stable/

Trio deliberately designs for structured concurrency and cancellation
ergonomics. If you can adopt Trio, it is often a great choice.

But many codebases cannot switch away from the standard library.
`asyncio` is widely deployed, and engineers deserve learning material with the
same concept-first clarity.

This guide complements the official documentation; it does not replace it.
When this guide mentions a concrete `asyncio` API, it links to the
corresponding page in the Python docs.

## Who This Is For

- You know Python, but `async`/`await` still feels slippery.
- You've written small `asyncio` snippets and now need robust, maintainable
  programs.
- You want modern, structured patterns (TaskGroup, timeouts, explicit
  lifetimes), not folklore.

## Python Version

This guide targets modern `asyncio` as documented in the current Python docs.
Many examples assume Python 3.11+ (for `asyncio.TaskGroup` and
`asyncio.timeout()`), and include notes for older versions when the model is
meaningfully different.

Official reference: https://docs.python.org/3/library/asyncio.html

## How To Read This Guide

- Start with the tutorial to build an intuition.
- Use the core concepts pages when you need precise semantics, guarantees, and
  common failure modes.

## Navigation

Tutorial (narrative learning):

- `python/asyncio/tutorial/introduction.md`
- `python/asyncio/tutorial/first-async-program.md`
- `python/asyncio/tutorial/structured-concurrency-with-asyncio.md`
- `python/asyncio/tutorial/cancellation-and-lifetimes.md`
- `python/asyncio/tutorial/building-real-applications.md`

Core concepts (semantics and mental models):

- `python/asyncio/core/mental-model.md`
- `python/asyncio/core/execution-model.md`
- `python/asyncio/core/task-and-taskgroup-semantics.md`
- `python/asyncio/core/cancellation-model.md`
- `python/asyncio/core/resource-management.md`
- `python/asyncio/core/communication-and-synchronization.md`
