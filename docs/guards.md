# Guards in tink_state

The `guard` parameter in the State constructor provides a mechanism for **value sanitization and transformation** in reactive state management. It allows you to ensure data integrity and consistency by applying transformations to values both when they are set and when they are first accessed.

## What is a Guard?

A guard is an optional function with the signature `(raw:T)->T` that serves as a **value sanitizer and transformer** for state values. It acts as a gatekeeper that can:

- Validate and sanitize input values
- Transform values to ensure consistency
- Enforce business rules and constraints
- Normalize data formats

## How Guards Work

### Constructor Signature

```haxe
public function new(
    value:T,
    ?comparator:Comparator<T>,
    ?guard:(raw:T)->T,
    ?onStatusChange:(isWatched:Bool)->Void
)
```

### Implementation Decision

When you provide a `guard` function to the State constructor, tink_state creates a `GuardedState` instead of a `SimpleState`:

```haxe
this = switch guard {
    case null: new SimpleState(value, comparator, onStatusChange);
    case f: new GuardedState(value, guard, comparator, onStatusChange);
}
```

## Bidirectional Application

Guards are applied in **two scenarios**:

### 1. On Write (Setting Values)
When `state.set(newValue)` is called, the guard function transforms the value before storage:

```haxe
override function set(value:T):T {
    if (!guardApplied)
        applyGuard();
    return super.set(guard(value));
}
```

### 2. On Read (Lazy Application)
When `state.getValue()` is called for the first time, the guard is applied to the initial value:

```haxe
override function getValue():T
    return
        if (!guardApplied) applyGuard();
        else value;

@:extern inline function applyGuard():T {
    this.guardApplied = true;
    return value = guard(value);
}
```

## Basic Examples

### Value Clamping
Ensure numeric values stay within specified bounds:

```haxe
import tink.state.State;

// Clamp values between 0 and 100
var percentage = new State(50, null, value -> Math.max(0, Math.min(100, value)));

percentage.set(150); // Stored as 100
percentage.set(-10); // Stored as 0
trace(percentage.value); // 0
```

### String Normalization
Ensure strings are always trimmed and lowercase:

```haxe
var username = new State("  Alice  ", null, value -> value.trim().toLowerCase());

username.set("  BOB  "); // Stored as "bob"
trace(username.value); // "alice" (initial value was also normalized)
```

### Type Validation with Fallback
Reject invalid values by returning a fallback:

```haxe
var positiveNumber = new State(1, null, value -> value > 0 ? value : 1);

positiveNumber.set(-5); // Stored as 1 (fallback)
positiveNumber.set(10); // Stored as 10
trace(positiveNumber.value); // 1
```

## Advanced Use Cases

### Data Transformation
Round floating point numbers to specific precision:

```haxe
var currency = new State(3.14159, null, value -> Math.round(value * 100) / 100);

currency.set(2.999); // Stored as 3.00
currency.set(1.234567); // Stored as 1.23
```

### Validation with State Preservation
Maintain the last valid value when invalid input is provided:

```haxe
class ValidatedState<T> {
    var lastValid:T;
    public final state:State<T>;

    public function new(initial:T, validator:T->Bool) {
        lastValid = initial;
        state = new State(initial, null, value -> {
            if (validator(value)) {
                lastValid = value;
                return value;
            }
            return lastValid;
        });
    }
}

// Usage: Email validation
var emailState = new ValidatedState("user@example.com", email -> email.indexOf("@") > 0);
emailState.state.set("invalid-email"); // Keeps "user@example.com"
emailState.state.set("new@email.com"); // Updates to "new@email.com"
```

### Complex Object Sanitization
Transform and validate complex data structures:

```haxe
typedef UserProfile = {
    name: String,
    age: Int,
    email: String
}

var profileGuard = (profile:UserProfile) -> {
    return {
        name: profile.name.trim(),
        age: Math.max(0, Math.min(120, profile.age)),
        email: profile.email.toLowerCase().trim()
    };
};

var userProfile = new State({
    name: "  John Doe  ",
    age: -5,
    email: "  JOHN@EXAMPLE.COM  "
}, null, profileGuard);

// Initial value is sanitized on first access
trace(userProfile.value.name); // "John Doe"
trace(userProfile.value.age);  // 0
trace(userProfile.value.email); // "john@example.com"
```

### Immutable Updates
Ensure objects are always copied to maintain immutability:

```haxe
typedef Config = {
    theme: String,
    language: String,
    features: Array<String>
}

var configGuard = (config:Config) -> {
    return {
        theme: config.theme,
        language: config.language,
        features: config.features.copy() // Always create a new array
    };
};

var appConfig = new State(initialConfig, null, configGuard);
```

## Performance Considerations

### Lazy Application
- The guard is applied **lazily** on the first read
- The `guardApplied` flag prevents multiple applications of the guard to the same value
- This optimizes performance when states are created but not immediately accessed

### Efficient Guard Functions
Write efficient guard functions to avoid performance bottlenecks:

```haxe
// Good: Simple, fast operations
var trimGuard = (s:String) -> s.trim();

// Be careful: Expensive operations in guards
var expensiveGuard = (data:Array<String>) -> {
    // This runs on every set() call
    return data.map(s -> s.toLowerCase()).filter(s -> s.length > 0);
};
```

## Integration with Other Features

### Guards with Comparators
Guards work seamlessly with custom comparators:

```haxe
var normalizedString = new State(
    "  Hello  ",
    (a, b) -> a.toLowerCase() == b.toLowerCase(), // Case-insensitive comparison
    s -> s.trim() // Trim whitespace
);
```

### Guards with Status Change Callbacks
Monitor when guarded states become active:

```haxe
var monitoredState = new State(
    initialValue,
    null,
    value -> sanitize(value),
    isWatched -> trace('State is ${isWatched ? "active" : "inactive"}')
);
```

## Best Practices

### 1. Keep Guards Pure
Guards should be pure functions without side effects:

```haxe
// Good: Pure function
var pureGuard = (value:Int) -> Math.max(0, value);

// Avoid: Side effects
var impureGuard = (value:Int) -> {
    trace('Setting value: $value'); // Side effect
    return Math.max(0, value);
};
```

### 2. Handle Edge Cases
Always consider null, undefined, or invalid inputs:

```haxe
var safeGuard = (value:Null<String>) -> {
    if (value == null) return "";
    return value.trim();
};
```

### 3. Document Guard Behavior
Clearly document what transformations your guards perform:

```haxe
/**
    Creates a state that ensures usernames are:
    - Trimmed of whitespace
    - Converted to lowercase
    - At least 3 characters long (padded with 'x' if shorter)
**/
function createUsernameState(initial:String) {
    return new State(initial, null, username -> {
        var normalized = username.trim().toLowerCase();
        while (normalized.length < 3) normalized += 'x';
        return normalized;
    });
}
```

### 4. Consider Validation vs Transformation
Decide whether to reject invalid values or transform them:

```haxe
// Transformation approach: Always return valid value
var transformGuard = (age:Int) -> Math.max(0, Math.min(120, age));

// Validation approach: Throw on invalid input
var validationGuard = (age:Int) -> {
    if (age < 0 || age > 120) throw 'Invalid age: $age';
    return age;
};
```

## Common Patterns

### Range Validation
```haxe
function createRangeState<T:Float>(initial:T, min:T, max:T) {
    return new State(initial, null, value -> Math.max(min, Math.min(max, value)));
}

var volume = createRangeState(50.0, 0.0, 100.0);
var temperature = createRangeState(20.0, -273.15, 1000.0);
```

### Format Normalization
```haxe
function createPhoneState(initial:String) {
    return new State(initial, null, phone -> {
        // Remove all non-digits
        var digits = ~/[^\d]/g.replace(phone, "");
        // Format as (XXX) XXX-XXXX if 10 digits
        return if (digits.length == 10) {
            '(${digits.substr(0,3)}) ${digits.substr(3,3)}-${digits.substr(6,4)}';
        } else digits;
    });
}
```

### Enum Validation
```haxe
enum Theme {
    Light;
    Dark;
    Auto;
}

function createThemeState(initial:Theme) {
    return new State(initial, null, theme -> {
        // Ensure theme is always valid
        return switch theme {
            case Light | Dark | Auto: theme;
            case _: Light; // Default fallback
        };
    });
}
```
