# ObservableDate in tink_state

`ObservableDate` is a specialized observable that tracks **time-based events and conditions**. It provides reactive monitoring of whether dates have passed and enables age-based logic in your applications.

## What is ObservableDate?

`ObservableDate` is an observable that monitors a specific date and notifies observers when that date is reached. It can be used to:

- Implement timeouts and deadlines
- Track the age of data or events
- Schedule reactive updates based on time
- Create time-sensitive state machines
- Monitor session expiration

```haxe
import tink.state.ObservableDate;
using DateTools;

// Create an observable for a future date
var deadline = new ObservableDate(Date.now().delta(5000)); // 5 seconds from now

// Monitor when the deadline passes
deadline.observe().bind(passed -> {
    if (passed) trace("Deadline reached!");
});
```

## Core Concepts

### Date Tracking
ObservableDate tracks a specific `Date` and provides a reactive `passed` property:

```haxe
var timeout = new ObservableDate(Date.now().delta(1000)); // 1 second timeout

trace(timeout.date); // The date being tracked
trace(timeout.passed); // Boolean: has the date passed?
```

### Age-Based Logic
Use `isOlderThan()` and `becomesOlderThan()` for age-based conditions:

```haxe
var createdAt = new ObservableDate(); // Defaults to now

// Check current age
if (createdAt.isOlderThan(5000)) { // 5 seconds old
    trace("Data is stale");
}

// React to age changes
var ageObservable = createdAt.becomesOlderThan(10000); // 10 seconds
ageObservable.bind(isOld -> {
    if (isOld) trace("Data expired");
});
```

## Basic Usage

### Simple Timeout
Create a timeout that fires after a specific duration:

```haxe
import tink.state.ObservableDate;
using DateTools;

var timeout = new ObservableDate(Date.now().delta(3000)); // 3 second timeout

timeout.observe().bind(expired -> {
    if (expired) {
        trace("Operation timed out!");
        // Handle timeout logic
    }
});
```

### Current Time Tracking
Create an ObservableDate for the current moment:

```haxe
var startTime = new ObservableDate(); // Defaults to Date.now()

// Later, check how much time has passed
if (startTime.isOlderThan(2000)) {
    trace("More than 2 seconds have elapsed");
}
```

### Past Date Handling
ObservableDate handles past dates immediately:

```haxe
var pastDate = new ObservableDate(Date.now().delta(-1000)); // 1 second ago

trace(pastDate.passed); // true - immediately
trace(pastDate.isOlderThan(500)); // true - it's older than 500ms
```

## Advanced Use Cases

### Session Management
Track session validity with reactive updates:

```haxe
class SessionManager {
    final sessionStart:ObservableDate;
    final maxSessionTime:State<Int>;
    public final isValid:Observable<Bool>;

    public function new(maxTime:Int) {
        sessionStart = new ObservableDate();
        maxSessionTime = new State(maxTime);

        // Reactive session validity
        isValid = Observable.auto(() -> {
            return !sessionStart.isOlderThan(maxSessionTime.value);
        });
    }

    public function extendSession(additionalTime:Int) {
        maxSessionTime.set(maxSessionTime.value + additionalTime);
    }
}

var session = new SessionManager(30000); // 30 second sessions
session.isValid.bind(valid -> {
    if (!valid) trace("Session expired!");
});
```

### Data Freshness Tracking
Monitor data age and trigger refreshes:

```haxe
class DataCache<T> {
    final data:State<T>;
    final lastUpdated:ObservableDate;
    final maxAge:Int;

    public final isStale:Observable<Bool>;

    public function new(initialData:T, maxAge:Int) {
        this.data = new State(initialData);
        this.lastUpdated = new ObservableDate();
        this.maxAge = maxAge;

        isStale = Observable.auto(() -> lastUpdated.isOlderThan(maxAge));
    }

    public function update(newData:T) {
        data.set(newData);
        lastUpdated = new ObservableDate(); // Reset timestamp
    }

    public function get():T {
        return data.value;
    }
}

var cache = new DataCache("initial", 5000); // 5 second max age
cache.isStale.bind(stale -> {
    if (stale) {
        trace("Cache is stale, refreshing...");
        // Trigger data refresh
    }
});
```

### Multiple Time Thresholds
Handle different behaviors at various time intervals:

```haxe
var itemCreated = new ObservableDate();

// Define multiple age-based states
var itemState = Observable.auto(() -> {
    if (itemCreated.isOlderThan(300000)) return "expired";    // 5 minutes
    if (itemCreated.isOlderThan(60000)) return "stale";       // 1 minute
    if (itemCreated.isOlderThan(10000)) return "aging";       // 10 seconds
    return "fresh";
});

itemState.bind(state -> {
    switch state {
        case "fresh": trace("Item is fresh");
        case "aging": trace("Item is getting old");
        case "stale": trace("Item is stale");
        case "expired": trace("Item has expired");
    }
});
```

### Scheduled Events
Create multiple scheduled events with different timing:

```haxe
class EventScheduler {
    final events:Array<{name:String, date:ObservableDate}> = [];

    public function schedule(name:String, delay:Int) {
        var eventDate = new ObservableDate(Date.now().delta(delay));
        events.push({name: name, date: eventDate});

        eventDate.observe().bind(passed -> {
            if (passed) {
                trace('Event "$name" triggered');
                executeEvent(name);
            }
        });
    }

    function executeEvent(name:String) {
        // Handle event execution
        trace('Executing event: $name');
    }
}

var scheduler = new EventScheduler();
scheduler.schedule("reminder", 5000);    // 5 seconds
scheduler.schedule("timeout", 10000);    // 10 seconds
scheduler.schedule("cleanup", 15000);    // 15 seconds
```

### Timeout with Reset Capability
Implement resettable timeouts for user activity:

```haxe
class ActivityTimeout {
    var currentTimeout:ObservableDate;
    var timeoutLink:CallbackLink;
    final duration:Int;
    final onTimeout:Void->Void;

    public function new(duration:Int, onTimeout:Void->Void) {
        this.duration = duration;
        this.onTimeout = onTimeout;
        reset();
    }

    public function reset() {
        // Cancel existing timeout
        if (timeoutLink != null) timeoutLink.cancel();

        // Create new timeout
        currentTimeout = new ObservableDate(Date.now().delta(duration));
        timeoutLink = currentTimeout.observe().bind(expired -> {
            if (expired) onTimeout();
        });
    }

    public function cancel() {
        if (timeoutLink != null) timeoutLink.cancel();
    }
}

var activityTimeout = new ActivityTimeout(30000, () -> {
    trace("User inactive for 30 seconds");
    // Handle inactivity
});

// Reset timeout on user activity
document.addEventListener("mousemove", _ -> activityTimeout.reset());
document.addEventListener("keypress", _ -> activityTimeout.reset());
```

## Integration with Other Observables

### Loading States with Timeout
Combine data loading with timeout handling:

```haxe
class DataLoader<T> {
    final loadingState:State<Bool>;
    final data:State<Null<T>>;
    final timeout:ObservableDate;

    public final status:Observable<String>;

    public function new(timeoutMs:Int) {
        loadingState = new State(false);
        data = new State(null);
        timeout = new ObservableDate(Date.now().delta(timeoutMs));

        status = Observable.auto(() -> {
            if (timeout.passed) return "timeout";
            if (data.value != null) return "loaded";
            if (loadingState.value) return "loading";
            return "idle";
        });
    }

    public function load():Promise<T> {
        loadingState.set(true);
        timeout = new ObservableDate(Date.now().delta(5000)); // Reset timeout

        return fetchData().next(result -> {
            data.set(result);
            loadingState.set(false);
            return result;
        });
    }

    function fetchData():Promise<T> {
        // Simulate async data fetching
        return Future.delay(2000, cast "loaded data");
    }
}

var loader = new DataLoader(10000); // 10 second timeout
loader.status.bind(status -> {
    switch status {
        case "loading": trace("Loading data...");
        case "loaded": trace("Data loaded successfully");
        case "timeout": trace("Loading timed out");
        case "idle": trace("Ready to load");
    }
});
```

### Reactive Time-Based Validation
Create validation that changes based on time:

```haxe
class TimeSensitiveValidator {
    final submissionTime:ObservableDate;
    final maxEditWindow:Int;

    public final canEdit:Observable<Bool>;
    public final timeRemaining:Observable<Int>;

    public function new(maxEditWindow:Int) {
        this.submissionTime = new ObservableDate();
        this.maxEditWindow = maxEditWindow;

        canEdit = Observable.auto(() -> !submissionTime.isOlderThan(maxEditWindow));

        timeRemaining = Observable.auto(() -> {
            var elapsed = Date.now().getTime() - submissionTime.date.getTime();
            return Std.int(Math.max(0, maxEditWindow - elapsed));
        });
    }

    public function resetEditWindow() {
        submissionTime = new ObservableDate();
    }
}

var validator = new TimeSensitiveValidator(60000); // 1 minute edit window

validator.canEdit.bind(canEdit -> {
    if (!canEdit) trace("Edit window has expired");
});

validator.timeRemaining.bind(remaining -> {
    if (remaining > 0) trace('${remaining}ms remaining to edit');
});
```

## Performance Considerations

### Efficient Timer Management
ObservableDate uses `haxe.Timer` internally and handles timer cleanup automatically:

```haxe
// ObservableDate manages timers efficiently
var dates = [for (i in 0...100) new ObservableDate(Date.now().delta(i * 100))];

// Timers are automatically cleaned up when dates pass
// No manual cleanup required
```

### Memory Management
Cancel bindings when no longer needed to prevent memory leaks:

```haxe
var timeout = new ObservableDate(Date.now().delta(5000));
var link = timeout.observe().bind(passed -> {
    if (passed) trace("Timeout!");
});

// Cancel when done
link.cancel();
```

## Best Practices

### 1. Use Appropriate Time Units
Be explicit about time units in your code:

```haxe
// Good: Clear time units
var FIVE_MINUTES = 5 * 60 * 1000;
var timeout = new ObservableDate(Date.now().delta(FIVE_MINUTES));

// Avoid: Unclear time values
var timeout = new ObservableDate(Date.now().delta(300000));
```

### 2. Handle Past Dates Gracefully
Always consider that dates might already be in the past:

```haxe
function createTimeout(delayMs:Int) {
    var timeout = new ObservableDate(Date.now().delta(delayMs));

    if (timeout.passed) {
        // Handle immediate expiration
        trace("Timeout already expired");
        return;
    }

    timeout.observe().bind(expired -> {
        if (expired) handleTimeout();
    });
}
```

### 3. Combine with State for Complex Logic
Use ObservableDate with State for sophisticated time-based behavior:

```haxe
var isPaused = new State(false);
var startTime = new ObservableDate();

var effectiveAge = Observable.auto(() -> {
    if (isPaused.value) return 0; // Age doesn't increase when paused
    return Date.now().getTime() - startTime.date.getTime();
});
```

### 4. Document Time-Based Behavior
Clearly document timing expectations:

```haxe
/**
    Creates a session with automatic expiration.
    @param maxIdleTime Maximum idle time in milliseconds before expiration
    @param warningTime Time in milliseconds before expiration to show warning
**/
function createSession(maxIdleTime:Int, warningTime:Int) {
    // Implementation
}
```

## Common Patterns

### Debounced Actions
Implement debouncing using ObservableDate:

```haxe
class Debouncer {
    var lastAction:ObservableDate;
    var pendingLink:CallbackLink;
    final delay:Int;

    public function new(delay:Int) {
        this.delay = delay;
    }

    public function debounce(action:Void->Void) {
        // Cancel previous action
        if (pendingLink != null) pendingLink.cancel();

        lastAction = new ObservableDate(Date.now().delta(delay));
        pendingLink = lastAction.observe().bind(expired -> {
            if (expired) action();
        });
    }
}

var debouncer = new Debouncer(500); // 500ms debounce

// Usage: debounce search requests
searchInput.bind(query -> {
    debouncer.debounce(() -> performSearch(query));
});
```

### Retry Logic with Backoff
Implement retry logic with exponential backoff:

```haxe
class RetryManager {
    var attempt:Int = 0;
    final maxAttempts:Int;
    final baseDelay:Int;

    public function new(maxAttempts:Int, baseDelay:Int) {
        this.maxAttempts = maxAttempts;
        this.baseDelay = baseDelay;
    }

    public function retry(operation:Void->Promise<Dynamic>):Promise<Dynamic> {
        return operation().recover(error -> {
            if (attempt >= maxAttempts) {
                return Promise.reject(error);
            }

            attempt++;
            var delay = baseDelay * Math.pow(2, attempt - 1); // Exponential backoff
            var retryTime = new ObservableDate(Date.now().delta(Std.int(delay)));

            return retryTime.observe().nextTime().next(_ -> retry(operation));
        });
    }
}
```

