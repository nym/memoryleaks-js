# 2. Forgotten event listeners

Every `addEventListener` is a reference from the event target *to* your handler — and through the handler's closure, to everything the handler captures. If you never call `removeEventListener`, that chain lives as long as the target does.

## The leak

```ts
class Chart {
  constructor(private data: number[]) {
    window.addEventListener("resize", this.onResize);
  }

  onResize = () => {
    this.redraw(this.data);
  };

  redraw(data: number[]) { /* ... */ }
}

let chart: Chart | null = new Chart(bigDataset);
chart = null; // does NOT free the chart
```

`window` still holds `onResize`, which is bound to the `Chart` instance via `this`, which retains `data`. The GC cannot collect any of it.

## Why it happens

DOM nodes (and `EventEmitter`s in Node) hold strong references to their listeners. Setting your variable to `null` removes *your* reference, but not the listener's.

## How to detect

In Chrome DevTools:

1. Take a heap snapshot.
2. Trigger the create/destroy cycle several times (e.g. mount/unmount the chart 5×).
3. Take another snapshot, filter by `Chart`.
4. If the count grows linearly with cycles, listeners are pinning instances.

You can also enumerate listeners with `getEventListeners(window)` in the DevTools console.

## The fix

Always pair `addEventListener` with `removeEventListener` — and use an `AbortController` for ergonomic cleanup:

```ts
class Chart {
  private ac = new AbortController();

  constructor(private data: number[]) {
    window.addEventListener("resize", this.onResize, {
      signal: this.ac.signal,
    });
  }

  onResize = () => this.redraw(this.data);

  destroy() {
    this.ac.abort(); // removes ALL listeners registered with this signal
  }

  redraw(data: number[]) { /* ... */ }
}
```

In framework code, do the cleanup in the unmount/destroy hook: `useEffect` return-fn (React), `onBeforeUnmount` (Vue), `onDestroy` (Svelte), `ngOnDestroy` (Angular).

## Takeaway

`addEventListener` is symmetric with `removeEventListener`. Forgetting the second half is one of the most common leaks in front-end code. `AbortController` makes it hard to forget.
