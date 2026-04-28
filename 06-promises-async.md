# 6. Promises and async iterators

Promises that never settle, and async iterators that are never closed, leak everything in their continuation.

## The leak: a promise that never resolves

```ts
const pendingResolvers: Array<(s: string) => void> = [];

function waitForSignal(): Promise<string> {
  return new Promise((resolve) => {
    pendingResolvers.push(resolve);
  });
}

async function handler(req: Request) {
  const big = await loadBigPayload(req);
  const sig = await waitForSignal(); // never resolves on this code path
  return process(big, sig);          // never runs
}
```

Each pending request leaks two things:
1. **The suspended frame**: `big`, `req`, and any other live locals are retained until the promise settles.
2. **`pendingResolvers` itself**: an unbounded array that grows by one per stuck request. Even after the request times out, its `resolve` closure stays in the array forever.

10k stuck requests = 10k retained payloads *and* 10k dead resolvers.

## Why it happens

`await` suspends the function. The engine packages up all live locals into the suspended frame and holds them until the awaited promise settles. If it never does, the frame never goes.

## How to detect

- **Heap snapshot** → look for `AsyncFunction` / `Promise` entries with high retained size.
- **Node 12+** has async stack traces on by default (`--async-stack-traces`). When you can capture a stack from a hung await, it'll show the await chain.
- Add timeouts as a defensive default — but make sure the timeout *also removes the entry from any pending list*, or you've just swapped one leak for another:

```ts
function waitForSignalWithTimeout(ms: number): Promise<string> {
  return new Promise((resolve, reject) => {
    const entry = (s: string) => {
      clearTimeout(t);
      resolve(s);
    };
    pendingResolvers.push(entry);
    const t = setTimeout(() => {
      const i = pendingResolvers.indexOf(entry);
      if (i >= 0) pendingResolvers.splice(i, 1); // <-- the missing piece
      reject(new Error("signal timeout"));
    }, ms);
  });
}
```

A naive `Promise.race(waitForSignal(), timeout)` would *not* fix the leak: the original `waitForSignal()`'s resolver stays in `pendingResolvers` forever even though nobody is awaiting it.

## The leak: an async iterator left open

```ts
async function* readLines(stream: ReadableStream<string>) {
  const reader = stream.getReader();
  while (true) {
    const { value, done } = await reader.read();
    if (done) return;
    yield value!;
  }
}

for await (const line of readLines(stream)) {
  if (line.includes("STOP")) break; // <-- early exit
}
// without `try/finally`, the reader is never released
```

## The fix

The runtime *always* calls the iterator's `return()` on `break`/`throw`/early-exit — but `return()` only runs cleanup code you wrote. Put the cleanup in `try/finally` so it actually fires:

```ts
async function* readLines(stream: ReadableStream<string>) {
  const reader = stream.getReader();
  try {
    while (true) {
      const { value, done } = await reader.read();
      if (done) return;
      yield value!;
    }
  } finally {
    reader.releaseLock();
  }
}
```

## Takeaway

Every `await` is a potential pin point. Every async iterator needs `try/finally` for cleanup. Add timeouts as a circuit-breaker, not just for correctness — they're a memory bound.
