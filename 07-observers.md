# 7. Observers and subscriptions

Anything with a `subscribe()` method probably needs an `unsubscribe()`. The pattern repeats across `IntersectionObserver`, `MutationObserver`, `ResizeObserver`, RxJS, Redux, and most pub/sub libraries.

## The leak

```ts
import { fromEvent, Subscription } from "rxjs";

class SearchBox {
  constructor(input: HTMLInputElement) {
    fromEvent(input, "input").subscribe((e) => {
      this.search((e.target as HTMLInputElement).value);
    });
  }
  search(q: string) { /* ... */ }
}
```

The subscription is never disposed. The DOM listener `fromEvent` registers retains the subscriber, which retains the `SearchBox`. When the SPA tears down the input, JS-side references via `fromEvent`'s subscriber chain may still pin the SearchBox unless something explicitly unsubscribes.

## Why it happens

A subscription is a two-way binding. The producer holds the consumer (to call its callback). The consumer holds the producer (to receive values). Neither side can be GC'd until the link is broken.

## How to detect

- Heap snapshots: filter for `Subscription`, `Subscriber`, or your observer class names. A growing count is a smoking gun.
- RxJS exposes no public API for inspecting subscriber counts — its internal field name (`_finalizers` in v7+, `_teardowns` historically) is private and changes between versions. Don't rely on it; use heap snapshots.
- Browser observers (`IntersectionObserver`, etc.) have no count API — diagnose with retainer chains.

## The fix

Always retain the subscription handle and dispose it in the teardown path:

```ts
class SearchBox {
  private sub: Subscription;

  constructor(input: HTMLInputElement) {
    this.sub = fromEvent(input, "input").subscribe(/* ... */);
  }

  destroy() {
    this.sub.unsubscribe();
  }
}
```

For RxJS in components, prefer `takeUntil(this.destroy$)` or the `takeUntilDestroyed()` operator (Angular 16+). For browser observers:

```ts
const obs = new IntersectionObserver(callback);
obs.observe(node);
// later:
obs.disconnect(); // releases all observed nodes
```

## Takeaway

For every `subscribe`/`observe`/`addListener` you write, ask: *what calls the inverse, and where?* If you can't answer, you have a leak.
