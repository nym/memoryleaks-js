# 4. Detached DOM nodes

A "detached DOM node" is an element that's been removed from the document but is still referenced from JavaScript. The browser can't free it because your code still points at it.

## The leak

```ts
const cache: Record<string, HTMLElement> = {};

function showRow(id: string) {
  const row = document.createElement("div");
  row.textContent = id;
  document.body.appendChild(row);
  cache[id] = row; // <-- the trap
}

function clear() {
  document.body.innerHTML = ""; // detaches all rows from the DOM
  // but `cache` still references them, plus their entire subtree
}
```

Each detached `<div>` retains its parent chain, children, attached listeners, and any data attributes — sometimes megabytes per node.

## Why it happens

`innerHTML = ""` and `node.remove()` only sever the parent→child link. They don't touch JavaScript references. If anything in JS land still holds the node, it stays.

## How to detect

Chrome DevTools → **Memory** → **Heap snapshot** → filter type box for **"Detached"**.

You'll see entries like `Detached HTMLDivElement`. Click one → **Retainers** shows the JS object pinning it. The chain typically ends at a global, a closure, or a long-lived service.

A useful heuristic: if you have *any* detached nodes after a route change in an SPA, something is wrong.

## The fix

Don't cache DOM nodes longer than you need them, and clean caches when you tear down:

```ts
function clear() {
  document.body.innerHTML = "";
  for (const k of Object.keys(cache)) delete cache[k];
}
```

`WeakRef` is **not** a good fix here, despite often being suggested. While a node is in the document, the document tree pins it — `WeakRef` makes no difference. Once detached, GC timing is non-deterministic, so `deref()` may still return the node long after you'd expect. Prefer explicit lifecycle: clear the cache when you tear down. See tutorial 11 for `WeakRef`'s real (narrow) use cases.

## Takeaway

Detached DOM is a uniquely visual leak — the browser shows them to you by name. Make a habit of filtering for "Detached" after exercising your app.
