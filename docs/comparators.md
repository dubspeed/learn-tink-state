# Comparators in tink_state

The `comparator` parameter in tink_state allows you to customize **how value changes are detected**. It determines when observers (bindings) should be notified of changes.

## What is a Comparator?

A `Comparator<T>` is a function that takes two values of type `T` and returns `true` if they should be considered equal, `false` if they're different:

```haxe
typedef Comparator<T> = (T, T) -> Bool
```

## How Comparators Work

When a value changes in tink_state, the system uses the comparator to decide whether to notify observers:

1. **Old value** vs **New value** comparison
2. If `comparator.eq(oldValue, newValue)` returns `true` → **No notification** (values considered equal)
3. If `comparator.eq(oldValue, newValue)` returns `false` → **Notify observers** (values considered different)

## Default Behavior

When no comparator is provided (`null`), tink_state uses **reference equality** (`==`):

```haxe
import tink.state.*;

// Default comparator behavior
var state = new State(42);
state.bind(value -> trace('Value: ${value}'));
state.value = 42; // No notification - same value
state.value = 43; // Notification fired - different value
```

## Usage Locations

Comparators can be specified at multiple levels, creating a **hierarchy**:

### Observable/State Level
```haxe
import tink.state.*;

// Custom comparator for the entire state
var state = new State(initialValue, customComparator);
```

### Binding Level
```haxe
import tink.state.*;

// Custom comparator for a specific binding
observable.bind(callback, customComparator);
```

### Hierarchy Resolution

Comparators work in a **two-stage process**:

1. **State Comparator (Gate)**: When `state.set(newValue)` is called, the state's comparator determines whether to update the value and fire invalidation events
2. **Binding Comparator (Filter)**: If the state fires invalidation, each binding uses its comparator to decide whether to invoke its callback

```haxe
var state = new State(0, stateComparator);
state.bind(callback, bindingComparator);

// When state.set(newValue) is called:
// 1. stateComparator.eq(oldValue, newValue) determines if state updates
// 2. If state updates, bindingComparator.eq(oldValue, newValue) determines if callback fires
```

#### State Comparator as Gate

The state comparator acts as a **gate** - if it considers values equal, *no* invalidation occurs, *no* value is set and *no* bindings are notified:

```haxe
// State with "always equal" comparator
var state = new State(0, (_, _) -> true);
state.bind(value -> trace('Never called'), (_, _) -> false); // Binding comparator is irrelevant

state.value = 42; // State comparator says equal, so no invalidation occurs
state.value = 100; // Still no invalidation - binding comparator never gets a chance to run
```

#### Binding Comparator as Filter

If the state comparator allows invalidation, binding comparators act as **filters** for individual callbacks:

```haxe
// State with "always different" comparator
var state = new State(0, (_, _) -> false);

// Binding 1: always fire
state.bind(value -> trace('Always: $value'), (_, _) -> false);

// Binding 2: never fire after initial
state.bind(value -> trace('Once: $value'), (_, _) -> true);

state.value = 42;
// State comparator allows invalidation
// state.value will change from 0 -> 42
// Binding 1 fires: "Always: 42"
// Binding 2 doesn't fire: binding comparator says equal

state.value = 42; // Same value
// State comparator still allows invalidation (always different)
// Binding 1 fires again: "Always: 42"
// Binding 2 still doesn't fire
```

#### When Binding Comparators Are Used

Binding comparators are always used as **filters** in the second stage. When the observable has no comparator, default `==` comparison is used as the **gate**:

```haxe
// Observable with NO comparator
var state = new State(0); // No comparator specified - uses == as gate

// This binding will use == as gate, customComparator as filter
state.bind(callback, customComparator);

// This binding will use == as both gate and filter
state.bind(callback);
```

## Basic Examples

### Always Fire (Never Equal)
Force notifications even when values are the same:

```haxe
import tink.state.*;

var state = new State(0, (_, _) -> false); // State comparator: never equal
state.bind(value -> trace('Value: ${value}'));

state.value = 0; // Still fires! State comparator says values are never equal
state.value = 0; // Fires again!
// Output: Value: 0, Value: 0, Value: 0
```

### Never Fire (Always Equal)
Suppress all notifications:

```haxe
import tink.state.*;

var state = new State(0, (_, _) -> true); // State comparator: always equal
state.bind(value -> trace('Value: ${value}'));

state.value = 42; // No notification - state comparator says values are always equal
state.value = 100; // Still no notification
// Output: Value: 0 (only initial binding)
```

## Practical Examples

### Deep Object Comparison
Compare objects by their properties instead of reference:

```haxe
import tink.state.*;

typedef Person = { name: String, age: Int };

var personComparator = (a: Person, b: Person) ->
    a.name == b.name && a.age == b.age;

var state = new State({ name: "Alice", age: 30 }, personComparator);
state.bind(person -> trace('Person: ${person.name}, ${person.age}'));

state.value = { name: "Alice", age: 30 }; // No notification - same content
state.value = { name: "Alice", age: 31 }; // Notification - different age
// Output: Person: Alice, 30, Person: Alice, 31
```

### Compare by Derived Values
Use the built-in (private) `byDerived` helper:

```haxe
import tink.state.*;

typedef User = { id: Int, name: String, lastLogin: Date };

// Only notify when the ID changes, ignore other properties
@:privateAccess
var comparator = Comparator.byDerived((user: User) -> user.id);
var state = new State(user, comparator);

state.bind(user -> trace('User ID: ${user.id}'));
state.value = { id: 1, name: "Bob", lastLogin: Date.now() }; // No notification - same ID
state.value = { id: 2, name: "Alice", lastLogin: Date.now() }; // Notification - different ID
```

### Performance Optimization
Avoid expensive re-renders when only irrelevant properties change:

```haxe
import tink.state.*;

typedef Model = { title: String, isVisible: Bool, internalData: String };

// Only update UI when visible properties change
var uiComparator = (a: Model, b: Model) ->
    a.title == b.title && a.isVisible == b.isVisible;

var model = new State(initialModel, uiComparator);
model.bind(updateUI); // Only fires when title or isVisible changes

model.value = ({ title: "Same", isVisible: true, internalData: "changed" }; // No UI update
model.value = { title: "Different", isVisible: true, internalData: "changed" }; // UI updates
```

### Threshold-Based Comparisons
Only notify when values differ significantly:

```haxe
import tink.state.*;

var threshold = 0.1;
var floatComparator = (a: Float, b: Float) ->
    Math.abs(a - b) < threshold;

var sensor = new State(0.0, floatComparator);
sensor.bind(value -> trace('Sensor: ${value}'));

sensor.value = 0.05; // No notification - within threshold
sensor.value = 0.15; // Notification - exceeds threshold
// Output: Sensor: 0.0, Sensor: 0.15
```

## Composable Comparators

Comparators can be combined using logical operators:

### AND Operator (`&&`)
Both comparators must consider values equal:

```haxe
import tink.state.*;

typedef Person = { name: String, age: Int };

final nameComparator:Comparator<Person> = (a, b) -> a.name == b.name;
final ageComparator:Comparator<Person> = (a, b) -> a.age == b.age;

var bothComparator = nameComparator && ageComparator;
// Only equal if BOTH name AND age are the same

var state = new State(person, bothComparator);
state.bind(p -> trace('Person: ${p.name}, ${p.age}'));

state.value = { name: "Alice", age: 30 }; // No notification - both same
state.value = { name: "Alice", age: 31 }; // Notification - age different
state.value = { name: "Bob", age: 31 }; // Notification - name different
```

### OR Operator (`||`)
Either comparator can consider values equal:

```haxe
import tink.state.*;

typedef Person = { name: String, age: Int };

final nameComparator:Comparator<Person> = (a, b) -> a.name == b.name;
final ageComparator:Comparator<Person> = (a, b) -> a.age == b.age;

var eitherComparator = nameComparator || ageComparator;
// Equal if EITHER name OR age is the same

var state = new State(person, eitherComparator);
state.bind(p -> trace('Person: ${p.name}, ${p.age}'));

state.value = { name: "Alice", age: 31 }; // No notification - name same
state.value = { name: "Bob", age: 30 }; // No notification - age same
state.value = { name: "Bob", age: 32 }; // Notification - both different
```

## Real-World Use Cases

### UI Performance Optimization
Avoid unnecessary DOM updates:

```haxe
import tink.state.*;

typedef ViewState = {
    title: String,
    isVisible: Bool,
    data: Array<String>,
    metadata: Dynamic
};

// Only re-render when user-visible properties change
var renderComparator = (a: ViewState, b: ViewState) ->
    a.title == b.title &&
    a.isVisible == b.isVisible &&
    a.data.length == b.data.length;

var viewState = new State(initialState, renderComparator);
viewState.bind(state -> expensiveRender(state));

// Metadata changes don't trigger expensive re-renders
viewState.value = { ...current, metadata: newMetadata }; // No render
viewState.value = { ...current, title: "New Title" }; // Renders
```

### Array/Collection Changes
For collections that return the same reference but with different contents:

```haxe
import tink.state.*;

// Always notify for collection changes since we reuse the same reference
var entityQueries = Observable.auto(() -> {
    // ... computation returns same map reference with different contents
    return cache;
}, (_, _) -> false); // Always fire since we reuse the same object

entityQueries.bind(cache -> updateUI(cache));
```

### Debouncing Similar Values
Implement value-based debouncing:

```haxe
import tink.state.*;

var tolerance = 5.0;
var debounceComparator = (a: Float, b: Float) ->
    Math.abs(a - b) <= tolerance;

var mousePosition = new State(0.0, debounceComparator);
mousePosition.bind(pos -> expensiveCalculation(pos));

// Small movements are ignored
mousePosition.value = 102.0; // No notification - within tolerance of 100.0
mousePosition.value = 107.0; // Notification - exceeds tolerance
```

### Complex Data Structure Comparisons
Handle nested objects and arrays:

```haxe
import tink.state.*;

typedef Config = {
    settings: { theme: String, language: String },
    features: Array<String>,
    version: String
};

var configComparator = (a: Config, b: Config) -> {
    // Compare settings
    var settingsEqual = a.settings.theme == b.settings.theme &&
                       a.settings.language == b.settings.language;

    // Compare features array
    var featuresEqual = a.features.length == b.features.length &&
                       a.features.toString() == b.features.toString();

    return settingsEqual && featuresEqual && a.version == b.version;
};

var config = new State(initialConfig, configComparator);
config.bind(cfg -> applyConfiguration(cfg));
```
