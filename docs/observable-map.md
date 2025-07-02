# ObservableMap in tink_state

**ObservableMap** is a reactive map structure in tink_state that provides reactive key-value storage with automatic change notification. It's essentially a wrapper around Haxe's standard `Map<K, V>` that adds reactive capabilities.

## Core Architecture

1. **Abstract Wrapper**: `ObservableMap<K, V>` is an abstract that wraps `MapImpl<K, V>`
2. **Implements IMap**: It implements Haxe's `IMap<K, V>` interface, so it works like a standard map
3. **Reactive Foundation**: Built on tink_state's reactive system using `Invalidator` and `ObservableObject`

## Key Features

### 1. Standard Map Operations

```haxe
import tink.state.ObservableMap;

var map = new ObservableMap(["x" => 10, "y" => 20]);

// Standard map operations work
map.set("z", 30);
var value = map.get("x");
var exists = map.exists("y");
map.remove("x");
map.clear();
```

### 2. Reactive Change Detection

- **Automatic Invalidation**: Any modification (`set`, `remove`, `clear`) invalidates the map
- **Dependency Tracking**: When accessed within `Observable.auto()`, the map is tracked as a dependency
- **Change Notification**: Observers are notified when the map changes

### 3. Entry-Level Observables

```haxe
// Observe a specific key
var valueObservable = map.entry("key");
valueObservable.bind(value -> trace('Key changed: $value'));

// Changes to that specific key trigger the binding
map.set("key", "new value"); // Triggers the binding
```

### 4. View Abstraction

```haxe
// Access the view for advanced operations
var view = map.view;

// Copy the map
var copy = view.copy();

// Convert to plain map
var plainMap = view.toMap();
```

## Internal Implementation

### MapImpl Class

- **Extends Invalidator**: Provides the reactive foundation
- **Valid Flag**: Tracks whether the map is currently valid
- **Entries Storage**: Holds the actual `Map<K, V>` data

### Key Methods

- **`update()`**: Performs mutations and invalidates if needed
- **`calc()`**: Performs reads and tracks dependencies
- **`fire()`**: Notifies all observers of changes

### Reactive Behavior

```haxe
@:extern inline function update<T>(fn:Void->T) {
  var ret = fn();
  if (valid) {
    valid = false;
    fire(); // Notify observers
  }
  return ret;
}

@:extern inline function calc<T>(f:Void->T) {
  valid = true;
  AutoObservable.track(this); // Track as dependency
  return f();
}
```

## Usage Patterns

### 1. Basic Reactive Map

```haxe
var userMap = new ObservableMap<Int, User>([]);

// React to any changes
userMap.bind(users -> updateUI(users));

// Add/remove users
userMap.set(1, new User("Alice"));
userMap.set(2, new User("Bob"));
userMap.remove(1);
```

### 2. Entry-Level Observation

```haxe
var config = new ObservableMap<String, String>([]);

// Observe specific configuration keys
config.entry("theme").bind(theme -> applyTheme(theme));
config.entry("language").bind(lang -> changeLanguage(lang));

// Only relevant observers are notified
config.set("theme", "dark"); // Only theme observer fires
config.set("language", "en"); // Only language observer fires
```

### 3. Derived Computations

```haxe
var scores = new ObservableMap<String, Int>([]);

// Compute derived values
var totalScore = Observable.auto(() -> {
  var total = 0;
  for (score in scores) total += score;
  return total;
});

totalScore.bind(total -> trace('Total: $total'));
```

### 4. Nested Reactive Structures

```haxe
// Complex nested structure
var baseMap = new ObservableMap<Int, Entity>([]);

// Each entity has its own ObservableMap
class Entity {
  public final id:Int;
  public final subMap:ObservableMap<String, Entity> = new ObservableMap([]);

  public function new(id) {
    this.id = id;
  }
}

// Changes propagate through the hierarchy
var entity0 = new Entity(0);
var entity1 = new Entity(1);

baseMap.set(0, entity0);
entity0.subMap.set("foo", entity1);
```

### 5. Advanced Map Operations

```haxe
import tink.state.ObservableMap;

// Create an observable map and observe all keys
var map = new ObservableMap(["x" => 10, "y" => 20]);

// Observe all values
map.view.copy().iterator().foreach(v -> trace('Value: $v'));

// Reactively count entries
var count = Observable.auto(() -> {
  var c = 0;
  for (_ in map.view.copy()) c++;
  return c;
});
count.bind(n -> trace('Map size: $n'));

map.set("z", 30); // Triggers the count binding
```

### Entry Observation

```haxe
var map = new ObservableMap([5 => 0, 6 => 0]);
var log = [];

function report(k:Int) return (v:Int) -> log.push('$k:$v');

var watch = [
  map.entry(5).bind(report(5), direct),
  map.entry(6).bind(report(6), direct),
];

map.set(5, 1);
map.set(5, 2);
map.set(5, 3);
map.set(4, 3);

// Log shows: "5:0,6:0,5:1,5:2,5:3"
// Only the observed keys trigger bindings
```

### Iterator Reactivity

```haxe
var map = new ObservableMap<String, String>(new Map());
map.set('key', 'value');

var counts = [];

var watch = Observable.auto(() -> {
  var counter = 0;
  for (i in [map.keys(), map.iterator()])
    for (x in i) counter++;
  return counter;
}).bind(counts.push);

map.set('key2', 'value'); // Triggers recomputation
map.set('key3', 'value'); // Triggers recomputation

// counts will be [2, 4, 6] as entries are added
```

## Best Practices

### 1. Use Entry Observation for Granular Updates

```haxe
// Good: Observe specific keys
config.entry("theme").bind(updateTheme);
config.entry("language").bind(updateLanguage);

// Avoid: Observe entire map for specific key changes
config.bind(entireConfig -> {
  if (entireConfig.exists("theme")) updateTheme(entireConfig.get("theme"));
});
```

### 2. Leverage View Abstraction

```haxe
// Use view for advanced operations
var filtered = map.view.copy().filter((k, v) -> v > 10);
var sorted = map.view.copy().sort((a, b) -> a.key - b.key);
```

### 3. Combine with Other Observables

```haxe
var userMap = new ObservableMap<Int, User>([]);
var selectedUserId = new State<Null<Int>>(null);

// Derived observable combining map and state
var selectedUser = Observable.auto(() -> {
  var id = selectedUserId.value;
  return id != null ? userMap.get(id) : null;
});
```

### 4. Handle Nested Structures

```haxe
// For complex nested maps, consider the hierarchy
var entityMap = new ObservableMap<Int, Entity>([]);

// Observe both the map and nested maps
entityMap.bind(entities -> updateEntityList(entities));
for (entity in entityMap) {
  entity.subMap.bind(subEntities -> updateSubEntities(entity.id, subEntities));
}
```
