# 5. Unbounded caches and Maps

A cache without an eviction policy is a memory leak with a friendly name.

## The leak

```ts
const userCache = new Map<string, User>();

async function getUser(id: string) {
  if (userCache.has(id)) return userCache.get(id)!;
  const user = await fetchUser(id);
  userCache.set(id, user);
  return user;
}
```

Looks fine in tests with 10 users. In production with 10 million users and a long-running Node process, this is a slow OOM.

## Why it happens

`Map`, `Set`, `Object`, and arrays are **strong references**. Anything you put in stays until you explicitly remove it.

The classic mistake is treating "cache" as synonymous with "speed-up." A cache is a *bounded* store with an *eviction policy*. Without bounds + eviction, it's a memory tomb.

## How to detect

- Heap snapshot → filter for `Map` (the JS `Map` constructor, not `system / Map` — those are V8 hidden-class shapes and will dominate the list if you don't filter). Sort by retained size.
- In Node: `process.memoryUsage().heapUsed` over time. A monotonic climb that correlates with unique-input cardinality is a cache leak.

## The fix

Use a bounded cache with an eviction policy. The standard answers:

```ts
// lru-cache v7+ (the v6 API used `maxAge` instead of `ttl`)
import { LRUCache } from "lru-cache";

const userCache = new LRUCache<string, User>({
  max: 10_000,                     // count-based bound
  ttl: 1000 * 60 * 5,              // time-based expiry
  updateAgeOnGet: false,
});
```

If the keys are objects you don't own (e.g. a request object), use `WeakMap`:

```ts
const perRequestData = new WeakMap<Request, RequestMeta>();
```

`WeakMap` keys don't prevent GC of the key, so when the request finishes and is dropped, its entry vanishes automatically.

## Takeaway

"Cache" without a bound is just "leak" with a marketing budget. Pick `LRUCache` for value caches; pick `WeakMap` when the key has its own lifetime.
