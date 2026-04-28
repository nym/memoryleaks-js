# 12. A leak-hunting checklist

Print this. Tape it to your monitor.

## Symptoms

- [ ] Memory grows monotonically while the app is idle.
- [ ] Memory grows linearly with a repeated user action (open/close, navigate A→B→A).
- [ ] Process gets killed by the OOM killer or hits `--max-old-space-size`.
- [ ] `MaxListenersExceededWarning` in Node logs.
- [ ] Detached DOM count rises after route changes.

## First-pass triage (15 minutes)

1. **Reproduce in isolation.** Smallest possible flow that grows memory.
2. **Force GC** before measuring (DevTools trash-can icon, or `--expose-gc` + `global.gc()` in Node).
3. **Three heap snapshots** with the action repeated between #2 and #3.
4. **Comparison view**, sort by Delta. The growing class is your suspect.
5. **Retainer chain.** Read bottom-up to find the root that's pinning it.

## Common culprits, ranked by frequency

1. **Forgotten event listeners** — fix with `AbortController`.
2. **Timers without clearers** — track IDs, clear in teardown.
3. **Subscriptions without unsubscribe** — RxJS, observers, EventEmitters.
4. **Unbounded caches** — switch to `LRUCache` or `WeakMap`.
5. **Closures over large data** — extract primitives, narrow scope.
6. **Detached DOM** — clear node references from JS when removing.
7. **Hung promises** — add timeouts.
8. **Accidental globals** — ES modules are strict by default and don't have this problem. If you have any non-module scripts, lint with `no-implicit-globals` and prefer migrating them to modules.

## Defensive code patterns

- Every class with a constructor that subscribes/listens/timers has a `destroy()` or `[Symbol.dispose]()`.
- Use `AbortController` for any group of listeners that share a lifetime.
- Bound every cache. No exceptions.
- Add a timeout to every cross-network `await`.
- Prefer `WeakMap` when associating data with an object you don't own.

## CI / automated detection

- **Memory regression test**: run a hot loop N times in a test, assert `heapUsed` delta is below a threshold. Catches new leaks at PR time.
- **`clinic doctor`** in a perf job for server-side code.
- **Lighthouse** for client-side: the "Avoid an excessive DOM size" and memory audits flag obvious problems.

## When to give up and restart

Some leaks aren't worth fixing — or aren't yours to fix. A short-lived process with a daily restart is a legitimate engineering choice. Just write it down so the next person doesn't think it's an accident.

## Further reading

- Chrome DevTools docs → Memory panel.
- Node.js docs → "Diagnostic tooling support guide."
- V8 blog → "Concurrent marking" and "Orinoco" posts (background on the GC).
- TC39 → Explicit Resource Management proposal (`using`).

## Takeaway

You don't need to memorize every leak pattern. You need a *process*: reproduce, snapshot, compare, follow the retainer chain. Everything in this folder is a special case of that loop.
