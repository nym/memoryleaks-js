# 1. Closures that capture too much

Closures are a top source of "huh, why is that still in memory?" in JavaScript. A closure keeps alive every variable that *it or a sibling closure in the same scope* references. Modern engines don't pin unreferenced locals — but they *do* pin everything captured by any sibling closure sharing the scope.

## The leak: shared scope between sibling closures

```ts
function attach() {
  const huge = new Array(1_000_000).fill("x");
  const smallId = 1;

  // Sibling A: only uses smallId, but shares scope with B.
  document.getElementById("btn")!.addEventListener("click", () => {
    console.log(smallId);
  });

  // Sibling B: captures `huge`. Because A and B share one
  // environment record, `huge` is retained for as long as
  // EITHER listener is alive.
  document.getElementById("dump")!.addEventListener("click", () => {
    console.log(huge.length);
  });
}
```

Remove the second listener (or never register it) and `huge` becomes collectable. Keep both, and even the "small" listener pins the megabyte array.

This is the canonical V8 closure-leak shape (the famous Meteor.js bug from 2013). It's a *static* decision made by the parser: if any closure in the scope captures `huge`, the variable lives in the shared context.

## Why it happens

A closure captures the *lexical environment record*, not individual variables. Sibling closures defined in the same scope share one record. The record is reachable as long as *any* of those closures is reachable, so every variable captured by *any* of them is reachable.

`eval` and `with` make this worse by forcing the parser to assume *every* variable might be referenced.

## How to detect

1. Open Chrome DevTools → **Memory** → **Heap snapshot**.
2. Take a snapshot, do the action you suspect leaks, take another.
3. Use the **Comparison** view, sort by **Delta**.
4. Look for `(closure)` entries. Click one — the **Retainers** pane shows what's holding it.

## The fix

Move the heavy work into its own scope so it's not shared with the long-lived closure:

```ts
function attach() {
  const smallId = computeSummary(); // does its own allocation + cleanup
  document.getElementById("btn")!.addEventListener("click", () => {
    console.log(smallId);
  });

  // Heavy work isolated in its own function — `huge` is a local
  // there, not in the same scope as the listener.
  registerDump();
}

function computeSummary() {
  const huge = new Array(1_000_000).fill("x");
  return huge.length;
}

function registerDump() {
  const huge = new Array(1_000_000).fill("x");
  document.getElementById("dump")!.addEventListener("click", () => {
    console.log(huge.length);
  });
}
```

For a long-lived field (a module-level cache, an instance property), nulling does help once you're done:

```ts
class Worker {
  private huge: number[] | null = new Array(1_000_000).fill(0);

  finish() {
    doWork(this.huge!);
    this.huge = null; // releases the array; the field used to pin it
  }
}
```

Note: nulling a *function-local* `let` right before the function returns is cargo-cult — the variable goes out of scope anyway. Only do it for fields and module-scope bindings that outlive the work.

## Takeaway

It's not "the closure captures everything." It's "sibling closures share their environment record, so they jointly retain everything any of them captures." Keep heavy locals in their own functions, away from long-lived listeners.
