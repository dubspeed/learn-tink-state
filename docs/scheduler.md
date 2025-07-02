# Scheduler and Batching in tink_state

## What is Batching?

Batching is a mechanism that groups multiple state changes or updates together, so that observers (bindings) are only notified once per batch, rather than after every single change. This improves performance and avoids redundant UI updates or computations.

---

## How Does tink_state Batch Updates?

### 1. Default Scheduler: Batched
By default, tink_state uses a **batched scheduler** for all observables and bindings. This means changes are collected and processed together at the end of the current execution frame (e.g., after the current event loop tick).

```haxe
// In Observable.hx
static var scheduler:Scheduler =
  #if macro
    Scheduler.direct;
  #else
    Scheduler.batched(Scheduler.batcher());
  #end
```

### 2. How Bindings are Scheduled
When you bind to an observable, the callback is not called immediately for every change. Instead, changes are collected and the callback is scheduled to run at the end of the batch.

```haxe
// In Binding.hx
public function invalidate()
  if (status == Valid) {
    status = Invalid;
    scheduler.schedule(this);
  }
```

### 3. Forcing Immediate Updates
If you want a binding to fire immediately (synchronously), you can use the `Scheduler.direct` scheduler:

```haxe
observable.bind(callback, Scheduler.direct);
```
This bypasses batching and invokes the callback as soon as the value changes.

### 4. Flushing Batches
You can manually flush all pending updates using:

```haxe
Observable.updateAll();
```
This processes all scheduled updates immediately, ensuring all bindings are up-to-date.

### 5. Atomic Updates
You can group multiple state changes into a single atomic batch using:

```haxe
Scheduler.atomically(() -> {
  // multiple state changes here
});
```
All changes inside the atomic block are batched, and observers are only notified once after the block completes.

---

## Why is Batching Important?

- **Performance:** Prevents redundant computations and UI updates.
- **Consistency:** Ensures observers see a consistent state after a group of changes.
- **Predictability:** You can reason about when and how often your bindings will fire.

---

## Example: Batched vs. Direct

Suppose you have a computed observable that depends on several states, and you update those states in a loop:

```haxe
Scheduler.atomically(() -> {
  for (item in items)
    item.done.value = true;
});
```
Without batching, each change would trigger the computed observable's binding, possibly hundreds of times. With batching, the binding is only triggered once, after all changes are done.

---

## Summary Table

| Scheduler                | Behavior                                         |
|--------------------------|--------------------------------------------------|
| `Scheduler.batched`      | Groups updates, fires bindings once per batch    |
| `Scheduler.direct`       | Fires bindings immediately (no batching)         |
| `Scheduler.atomically`   | Groups multiple changes into a single batch      |
| `Observable.updateAll()` | Manually flushes all pending updates             |

