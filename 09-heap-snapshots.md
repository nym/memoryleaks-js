# 9. Detection: Chrome DevTools heap snapshots

The single most useful skill for finding leaks. Worth practicing on a known-leaky example before you need it for real.

## The 3-snapshot technique

This is the gold standard:

1. **Snapshot 1**: load the app to a steady state (logged in, on the page you're testing). Don't include startup allocations in your diff.
2. **Action**: perform the suspicious cycle 5–10 times (open/close a modal, navigate route A→B→A, etc.).
3. **Snapshot 2**: after the cycles.
4. Repeat the action 5–10 more times.
5. **Snapshot 3**.

Then in DevTools, switch the dropdown to **Comparison** and compare Snapshot 3 to Snapshot 1.

What you're looking for:
- Objects whose **count grows linearly with cycles** — `#New` minus `#Deleted` should ideally be 0 for any class created during the cycle.
- The **Retained Size** column tells you who's holding the most.

## Reading retainer chains

Click a leaked object → bottom pane shows **Retainers**. Read it bottom-up: each entry is "X is retained by Y because Y has property Z pointing at X."

Common shapes:
- `(closure)` chain ending at a `Window` or global → forgotten listener or timer.
- `Map` → `(internal)` → your object → cache without eviction.
- `Detached HTMLElement` → JS variable → DOM ref outliving its node.

## Filters that earn their keep

In the snapshot view, the **Class filter** box accepts substrings:

- `Detached` — finds all detached DOM.
- Your component class name — counts instances.
- `(closure)` — all closures, sortable by retained size.
- `(string)` — useful when you're leaking serialized payloads.

## "Allocation instrumentation on timeline"

Memory tab → second profile type → record while you exercise the suspicious flow. The timeline shows blue (allocated, still alive) vs grey (allocated and freed) bars over time. Persistent blue bars between idle moments = retained allocations.

## Pitfalls

- **Don't trust a single snapshot.** Always compare. Memory naturally fluctuates with caches and JIT data.
- **Heap snapshots already trigger a GC** for you — you don't need to click the trash-can first. (You *do* need it before reading values from the **Performance** panel's memory graph, which doesn't auto-GC.)
- **Compiled/inlined functions** can show up as `(anonymous)` or be attributed to a `system / Context` rather than the function you wrote. Follow the retainer chain rather than trusting the label.

## Takeaway

Three snapshots, comparison view, retainer chain. That sequence catches 80% of leaks. The rest are exercises in patience and `WeakRef` (tutorial 11).
