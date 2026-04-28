# 8. Node.js: streams, EventEmitters, requires

Server-side TypeScript leaks have a different texture from browser leaks. The process runs for weeks; even a tiny leak per request adds up.

## EventEmitter leaks

```ts
import { EventEmitter } from "node:events";
import type { IncomingMessage } from "node:http";
const bus = new EventEmitter();

export function handle(req: IncomingMessage) {
  bus.on("shutdown", () => cleanup(req)); // added per request, never removed
}
```

You'll see this warning:

```
(node:1234) MaxListenersExceededWarning: Possible EventEmitter memory leak
detected. 11 shutdown listeners added.
```

That warning at 10 listeners is a great early-warning signal. **Don't** silence it with `bus.setMaxListeners(0)` — that's hiding the leak. Fix the listener lifecycle.

```ts
export function handle(req: IncomingMessage) {
  const onShutdown = () => cleanup(req);
  bus.once("shutdown", onShutdown);
  req.on("close", () => bus.off("shutdown", onShutdown));
}
```

## Stream leaks

A stream that's never consumed, errored, or destroyed pins its buffers. The classic shape: a manual `source.pipe(transform).pipe(dest)` where the middle stage errors but the source isn't destroyed, leaving its buffers and any pending reads alive.

Use `pipeline` from `node:stream/promises` — it destroys every stage on error or completion:

```ts
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";

await pipeline(
  createReadStream("in"),
  transform,
  createWriteStream("out"),
); // every stage destroyed on any error
```

## `require()` cache and dynamic imports

`require()` caches modules forever in the process. Hot-reloaders that delete `require.cache` entries to "reload" leak the old module's exports if anything outside still references them. Use a real reload mechanism (`tsx watch`, `nodemon`) which restarts the process.

## Async-local storage leaks

`AsyncLocalStorage` is a strong reference rooted at the async context. Putting a giant object in `als.run(big, ...)` keeps `big` alive for the entire async operation tree. Store IDs, not payloads.

## How to detect in Node

```bash
node --inspect --expose-gc ./dist/server.js
```

Then in Chrome `chrome://inspect`:
- **Heap snapshots**: same UI as the browser.
- **Allocation sampling**: low overhead, OK for production-like loads.

For automated leak detection in tests, launch Node with `--expose-gc` and call `global.gc()` directly:

```bash
node --expose-gc ./node_modules/.bin/vitest run
```

```ts
declare const gc: () => void; // available with --expose-gc

test("doWork doesn't leak", async () => {
  gc(); const before = process.memoryUsage().heapUsed;
  for (let i = 0; i < 1000; i++) await doWork();
  gc(); const after = process.memoryUsage().heapUsed;
  expect(after - before).toBeLessThan(1_000_000);
});
```

(There's a fragile runtime trick using `v8.setFlagsFromString("--expose-gc")` + `vm.runInNewContext("gc")` to enable GC mid-process. It works but is non-portable across Node versions — prefer the launch flag.)

## Takeaway

In Node, the listener-warning is your friend, `pipeline` is your other friend, and process restart is the only reliable way to "unload" code.
