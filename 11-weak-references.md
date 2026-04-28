# 11. WeakMap, WeakSet, WeakRef, FinalizationRegistry

The "weak" family lets you reference an object *without preventing its garbage collection*. Used correctly, they fix whole categories of leaks. Used incorrectly, they introduce bugs that are worse than leaks.

## `WeakMap` — the safe one

```ts
const metadata = new WeakMap<Request, RequestMeta>();

function annotate(req: Request, meta: RequestMeta) {
  metadata.set(req, meta);
}
```

When `req` is no longer reachable from anywhere else, the entry is removed automatically. Perfect for "extra data attached to an object I don't own."

Limits:
- Keys must be objects, or (since ES2023) non-registered Symbols. Not strings or numbers.
- Not iterable. You can't list entries; you can only ask "is this key here?"

## `WeakSet` — same idea, no values

```ts
const seen = new WeakSet<HTMLElement>();
seen.add(el);
seen.has(el); // true while `el` is reachable from elsewhere
```

Useful for marking objects without preventing collection. Don't rely on `has(el)` becoming false at any specific time — GC timing is unobservable, so you can't use entry-removal as a signal.

## `WeakRef` — use sparingly

```ts
const ref = new WeakRef(bigObject);
// ...later
const obj = ref.deref();
if (obj) use(obj); // may or may not still exist
```

The MDN warning is worth quoting: avoid `WeakRef` if you can. The reasons:

1. **Non-determinism**: when GC runs is unspecified. The same code may behave differently on different engines or even runs.
2. **KeepDuringJob semantics**: once `deref()` returns the target in a job, the target is kept alive at least until the end of that job. So observing the value subtly extends its lifetime — a bad fit for any logic that branches on "is it still alive?"

Legitimate uses: opportunistic caches where missing the cache is fine, and observer registries where you want auto-cleanup.

## `FinalizationRegistry` — even more sparingly

Lets you register a callback to run when an object is GC'd:

```ts
const registry = new FinalizationRegistry((heldValue: string) => {
  console.log("cleaning up", heldValue);
});

registry.register(myObject, "myObject's id");
```

Spec explicitly says callbacks may **never run**. So:
- Don't use it for **required** cleanup (closing a file handle, releasing a lock).
- Use it as a *backstop* for telemetry or best-effort cleanup, paired with explicit `dispose()`.

ECMAScript's newer **`using` / explicit resource management** (`Symbol.dispose`) is the correct tool for required cleanup. Requires TypeScript 5.2+ and either a runtime with native support (Node 22+, recent V8) or downleveling via TS:

```ts
{
  using conn = openConnection();
  conn.send(data);
} // conn[Symbol.dispose]() runs here, deterministically
```

## When to reach for which

| Need                                              | Use                                  |
|---------------------------------------------------|--------------------------------------|
| Attach metadata to an object you don't own        | `WeakMap`                            |
| Mark objects without retaining them               | `WeakSet`                            |
| Cache where missing is OK                         | `WeakRef` (or LRU)                   |
| Required cleanup at scope end                     | `using` + `Symbol.dispose`           |
| Best-effort cleanup as a safety net               | `FinalizationRegistry`               |

## Takeaway

`WeakMap` is the unsung hero. `WeakRef` and `FinalizationRegistry` are sharp tools — pick `using`/`Symbol.dispose` first when you can.
