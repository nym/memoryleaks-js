# 3. Timers and intervals

`setInterval` and `setTimeout` keep their callback alive until they fire (or forever, in the case of intervals). The callback's closure keeps everything it references alive too.

## The leak

```ts
class Poller {
  private buffer: any[] = [];

  constructor() {
    setInterval(() => {
      this.buffer.push(fetchSomething());
    }, 1000);
  }
}

let p: Poller | null = new Poller();
p = null; // interval still runs; Poller and its growing buffer live forever
```

Two leaks in one: the interval keeps `Poller` alive, and `buffer` grows without bound.

## Why it happens

`setInterval` returns an ID, but the runtime holds an internal reference to your callback until you call `clearInterval`. No `clearInterval` = no GC.

## How to detect

- **Symptom**: memory grows steadily over time even when the app is "idle".
- **Node**: run with `node --inspect` and watch the heap in `chrome://inspect`. A sawtooth that climbs is normal; a monotonic climb is a leak.
- **Browser**: Performance panel → record → look for retained size that grows with wall-clock time, not with user activity.

## The fix

Track the timer ID and clear it on teardown:

```ts
class Poller {
  private buffer: any[] = [];
  private timerId: ReturnType<typeof setInterval>;

  constructor() {
    this.timerId = setInterval(() => {
      this.buffer.push(fetchSomething());
      if (this.buffer.length > 100) this.buffer.shift(); // bound it
    }, 1000);
  }

  destroy() {
    clearInterval(this.timerId);
  }
}
```

Two extras worth knowing:

- **Node.js**: `timer.unref()` lets the process exit even if the timer is pending. It does *not* affect GC — the callback and its closure are still retained by the active timer. `unref()` is about process lifetime, not memory.
- **`setTimeout` recursion**: `setTimeout(fn, ...)` where `fn` reschedules itself has the same leak shape as `setInterval`. Keep a flag and check it.

## Takeaway

Every timer needs a clearer. If a class starts a timer in its constructor, it needs a `destroy()` method that clears it.
