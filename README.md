# TypeScript Memory Leaks — Bite-Size Tutorials

A short, hands-on tour of how memory leaks happen in TypeScript/JavaScript, how to spot them, and how to fix them. Each tutorial is self-contained: read it in 5 minutes, run the example, then move on.

## How memory works (30-second refresher)

JavaScript engines (V8, JSC, SpiderMonkey) use **tracing garbage collection**. An object is kept alive as long as it is *reachable* from a GC root (globals, the active call stack, live closures, the DOM). A "leak" is almost never the GC failing — it's your code unintentionally keeping a reference alive.

The mental model that matters:

> **A leak is a reference you forgot you were holding.**

The tutorials below are a catalog of common ways that happens.

## Tutorials

1. [Closures that capture too much](01-closures.md)
2. [Forgotten event listeners](02-event-listeners.md)
3. [Timers and intervals](03-timers.md)
4. [Detached DOM nodes](04-detached-dom.md)
5. [Unbounded caches and Maps](05-caches.md)
6. [Promises and async iterators](06-promises-async.md)
7. [Observers and subscriptions](07-observers.md)
8. [Node.js: streams, EventEmitters, requires](08-node-specifics.md)
9. [Detection: Chrome DevTools heap snapshots](09-heap-snapshots.md)
10. [Detection: allocation timelines and `--inspect`](10-allocation-profiling.md)
11. [Tooling: WeakRef, FinalizationRegistry, WeakMap](11-weak-references.md)
12. [A leak-hunting checklist](12-checklist.md)

## Running the examples

Snippets are illustrative — copy the relevant block into a `.ts` file and run it with `tsx` or `ts-node`. Browser examples (DOM, DevTools) need a page; paste them into a `<script type="module">` and open DevTools to follow along.

The point isn't to ship this code — it's to *see the leak* in a profiler, then *see the fix*.
