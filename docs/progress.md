# Progress

Progress in tink_state provides observable progress tracking for asynchronous operations. It wraps `tink.core.Progress` and adds reactive capabilities through Observable integration.

## Important API Distinction

⚠️ **Warning**: `tink.state.Progress` and `tink.core.Progress` have similar but different APIs:

| Feature | tink.state.Progress | tink.core.Progress |
|---------|-------------------|-------------------|
| Observable integration | ✅ `observe()` method | ❌ No direct observable |
| Reactive binding | ✅ `bind()` method | ❌ Uses `handle()` instead |
| Progress tracking | ✅ Wraps core functionality | ✅ Core implementation |
| Future integration | ✅ Inherited from core | ✅ Native support |

Always check your imports to avoid confusion!

## Basic Usage

### Creating Progress

```haxe
import tink.state.Progress;

// Create a progress trigger for manual control
var trigger = Progress.trigger();
var progress = trigger.asProgress();

// Bind to progress changes
progress.bind(status -> {
  switch status {
    case InProgress(value):
      trace('Progress: ${value.value}/${value.total}');
    case Finished(result):
      trace('Completed: $result');
  }
});

// Report progress
trigger.progress(0.5, Some(1.0)); // 50% complete
trigger.finish("Done!"); // Complete with result
```

### Progress from Custom Function

```haxe
var progress = Progress.make((reportProgress, finish) -> {
  // Simulate async operation
  var current = 0.0;
  var timer = haxe.Timer.delay(() -> {
    current += 0.25;
    reportProgress(current, Some(1.0));
    if (current >= 1.0) {
      finish("Operation Complete");
    }
  }, 100);

  // Return cleanup function
  return () -> timer.stop();
});
```

## Progress Values and Normalization

Progress values consist of a current value and optional total:

```haxe
var trigger = Progress.trigger();
var progress = trigger.asProgress();

progress.bind(status -> {
  switch status {
    case InProgress(value):
      // Access raw values
      trace('Current: ${value.value}');
      trace('Total: ${value.total}');

      // Normalize to 0-1 range
      switch value.normalize() {
        case Some(normalized):
          trace('Percentage: ${normalized.toPercentageString(1)}');
        case None:
          trace('No total specified');
      }
    case Finished(_):
  }
});

// Progress without total
trigger.progress(50, None);

// Progress with total (normalizable)
trigger.progress(25, Some(100)); // 25%
trigger.progress(75, Some(100)); // 75%
```

## Integration with Promises and Futures

### Promise Integration

```haxe
// Progress from Promise<Progress<T>> becomes Progress<Outcome<T, Error>>
var promiseProgress: Promise<Progress<String>> = someAsyncOperation();
var progress: Progress<Outcome<String, Error>> = promiseProgress;

progress.bind(status -> {
  switch status {
    case Finished(Success(result)): trace('Success: $result');
    case Finished(Failure(error)): trace('Error: $error');
    case InProgress(value): trace('Loading: ${value.value}');
  }
});
```

### Future Integration

```haxe
// Future<Progress<T>> automatically becomes Progress<T>
var futureProgress: Future<Progress<String>> = getFutureProgress();
var progress: Progress<String> = futureProgress;

progress.bind(status -> {
  switch status {
    case InProgress(value): trace('Progress: ${value.value}');
    case Finished(result): trace('Result: $result');
  }
});
```

## Observable Integration

Progress works seamlessly with other tink_state observables:

```haxe
var trigger = Progress.trigger();
var progress = trigger.asProgress();
var isEnabled = new State(true);

// Computed observable combining progress and state
var status = Observable.auto(() -> {
  if (!isEnabled.value) return "disabled";

  return switch progress.observe().value {
    case InProgress(value): 'loading: ${Math.round(value.value * 100)}%';
    case Finished(_): "complete";
  }
});

status.bind(s -> trace('Status: $s'));

// Updates trigger reactive recalculation
trigger.progress(0.3, Some(1.0)); // "loading: 30%"
isEnabled.set(false); // "disabled"
```

## Advanced Features

### Progress Mapping

Transform progress results while maintaining progress tracking:

```haxe
var trigger = Progress.trigger();
var progress = trigger.asProgress();
var uppercaseProgress = progress.map(result -> result.toUpperCase());

uppercaseProgress.bind(status -> {
  switch status {
    case InProgress(value): trace('Progress: ${value.value}');
    case Finished(result): trace('Result: $result'); // Will be uppercase
  }
});

trigger.progress(0.5, Some(1.0));
trigger.finish("hello"); // Result will be "HELLO"
```

### Multiple Subscribers

Progress supports multiple subscribers efficiently:

```haxe
var trigger = Progress.trigger();
var progress = trigger.asProgress();

// Multiple independent subscribers
var link1 = progress.bind(status -> trace('Subscriber 1: $status'));
var link2 = progress.bind(status -> trace('Subscriber 2: $status'));
var link3 = progress.bind(status -> trace('Subscriber 3: $status'));

// All subscribers receive updates
trigger.progress(0.5, Some(1.0));
trigger.finish("Done");

// Clean up
link1.cancel();
link2.cancel();
link3.cancel();
```

### Late Subscribers

Subscribers that bind after progress has started immediately receive the current status:

```haxe
var trigger = Progress.trigger();
var progress = trigger.asProgress();

// Update progress before subscribing
trigger.progress(0.7, Some(1.0));

// Late subscriber immediately gets current status
progress.bind(status -> {
  // This will immediately fire with InProgress(0.7)
  trace('Current status: $status');
});
```

### Cancellation and Cleanup

Progress operations can be cancelled and cleaned up:

```haxe
var progress = Progress.make((reportProgress, finish) -> {
  var cancelled = false;
  var timer = haxe.Timer.delay(() -> {
    if (!cancelled) {
      reportProgress(0.5, Some(1.0));
      // Continue operation...
    }
  }, 100);

  // Return cleanup function
  return () -> {
    cancelled = true;
    timer.stop();
  };
});

var link = progress.bind(status -> trace('Status: $status'));

// Cancel the subscription (triggers cleanup)
link.cancel();
```

## Common Patterns

### Loading States

```haxe
var dataLoader = Progress.make((reportProgress, finish) -> {
  // Simulate data loading with progress
  var loaded = 0;
  var total = 100;

  var timer = haxe.Timer.delay(() -> {
    loaded += 10;
    reportProgress(loaded, Some(total));

    if (loaded >= total) {
      finish(["data1", "data2", "data3"]);
    }
  }, 50);

  return () -> timer.stop();
});

dataLoader.bind(status -> {
  switch status {
    case InProgress(value):
      updateLoadingBar(value.normalize());
    case Finished(data):
      displayData(data);
  }
});
```

### Progress Composition

```haxe
// Combine multiple progress operations
var operation1 = createOperation1();
var operation2 = createOperation2();

var combinedProgress = Observable.auto(() -> {
  var status1 = operation1.observe().value;
  var status2 = operation2.observe().value;

  return switch [status1, status2] {
    case [Finished(_), Finished(_)]: "All complete";
    case [InProgress(v1), InProgress(v2)]:
      'Progress: ${(v1.value + v2.value) / 2}';
    case _: "In progress...";
  }
});
```

## Error Handling

```haxe
// Handle errors in Promise-based progress
var errorProneProgress: Progress<Outcome<String, Error>> = somePromiseProgress;

errorProneProgress.bind(status -> {
  switch status {
    case Finished(Success(result)):
      trace('Success: $result');
    case Finished(Failure(error)):
      trace('Error: ${error.message}');
      showErrorDialog(error);
    case InProgress(value):
      updateProgressBar(value);
  }
});
```
