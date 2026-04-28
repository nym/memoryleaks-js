# 10. Detection: allocation timelines and `--inspect`

Snapshots tell you *what* is leaking. Allocation profiling tells you *where it was allocated*.

## Allocation sampling (low overhead)

DevTools → Memory → **Allocation sampling**. Statistical, low overhead — safe to run against production-like load (much cheaper than the timeline mode below; exact cost depends on V8 version and allocation rate). Gives you:

- A flame graph of allocation sites.
- Bytes allocated per call stack.

It does **not** tell you what's still alive — only what was allocated. Combine with a snapshot afterwards to learn which of those sites produced retained objects.

## Allocation timeline (full instrumentation)

DevTools → Memory → **Allocation instrumentation on timeline**. Heavy — records every allocation with its stack. Use it on a small reproducer, not a live system.

The view: time on the X axis, allocations as vertical bars. After GC, freed allocations turn grey; surviving ones stay blue. Click a blue bar → see the exact stack that allocated the still-alive object.

This is the most direct way to map "retained object" → "line of code that created it."

## Node: connecting Chrome DevTools

```bash
node --inspect ./dist/server.js
# or to break before the first line:
node --inspect-brk ./dist/server.js
```

Open `chrome://inspect` in Chrome → **Inspect** under "Remote Target." The Memory tab works exactly as it does for a page.

For headless leak hunting:

```bash
node --heapsnapshot-near-heap-limit=3 --max-old-space-size=512 ./dist/server.js
```

This writes up to 3 `.heapsnapshot` files as the heap approaches the limit (one per approach, capped at the count you give). If the heap retreats and then approaches again, you get another snapshot. Open them in DevTools after the crash.

## On-demand snapshots from inside the process

```ts
import { writeHeapSnapshot } from "node:v8";

process.on("SIGUSR2", () => {
  const file = writeHeapSnapshot();
  console.log("heap snapshot written to", file);
});
```

```bash
kill -SIGUSR2 $(pgrep -f node)
```

Take three of these over an hour, diff in DevTools. Same technique as in-browser, just from a long-running server.

## `clinic.js` — the easy button

```bash
npx clinic doctor -- node ./dist/server.js
npx clinic heapprofile -- node ./dist/server.js
```

`clinic doctor` flags whether you have a "memory issue" without you needing to read flame graphs. `clinic heapprofile` records V8's sampling heap profiler. (Clinic also offers `flame` and `bubbleprof`; the older standalone `heapprofiler` package was folded in.) Good first stop for production-shaped leaks.

## Takeaway

Snapshots = *what* is alive. Allocation profiles = *where it came from*. You usually need both: snapshot to find the leaky class, allocation profile to find the line that creates it.
