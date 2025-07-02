# learn-tink-state
Documentation and resources about reactive programming and tink_state in Haxe

---

## Getting started

```haxe
// Install via haxelib
haxelib install tink_state

// In your .hxml
-lib tink_state

// Basic imports
import tink.state.State;
import tink.state.Observable;
```

--

## Resources & Further Learning

**tink_state Documentation and resources:**
- https://haxetink.github.io/#/
- https://github.com/dubspeed/learn-tink-state

**Related:**
- **TinkStateSharp** (C#): https://github.com/nadako/TinkStateSharp
- **Rx.NET** (C#): https://introtorx.com/
- **RxJS** (JavaScript): https://rxjs.dev/
- **MobX** (JavaScript): https://mobx.js.org/

---

## tink_state: The Basics

`tink_state` is a Haxe library for **reactive state management**.

It provides:
- **Observable**: a value you can subscribe to
- **State**: a writable observable (think: a variable you can react to)
- **ObservableArray** and **ObservableMap**: reactive collections
- **Promised** for async operations
- Advanced features like **comparators** and **guards**

---

## Your first State

```haxe
// Create a writable state
var counter = new State(0);

// Listen for changes
counter.bind(value -> trace('Counter: $value'));

// Change the value
counter.value = 1; // Outputs: Counter: 1
counter.value = 2; // Outputs: Counter: 2
```

---

## Observables in tink_state

```haxe
var a = new State(1);
var b = new State(2);

// Create a derived observable
var sum = Observable.auto(() -> a.value + b.value);

sum.bind(value -> trace('Sum is $value'));
// Changing a or b will update sum and trigger the binding
```

- `State` is like a variable you can read/write.
- `Observable.auto` creates a new observable that automatically tracks dependencies.
- derived data will automatically be updated.

---

## What Makes tink_state Different?

Most reactive libraries (like RxJS, MobX, Vue) use the idea of **operators** (`map`, `filter`, `combine`, etc.) to build new streams.

`tink_state` can do this too, but its **core innovation** is `Observable.auto`:

- **Automatic Dependency Tracking**: Just write a function that reads other observables/states. tink_state figures out what you depend on.
- **No Manual Subscription Management**: You don't have to wire up listeners for every dependency.
- **Efficient Batching**: Updates are batched by default, so even if you change many things at once, your UI only updates once.

---

## Observable.auto: How It Works

```haxe
var product = Observable.auto(() -> a.value * b.value);
product.bind(v -> trace('Product: $v'));
a.value = 6; // Triggers the binding
```

- The function you pass to `auto` is run once to compute the value.
- Any `State` or `Observable` you read inside that function is tracked as a dependency.
- If any dependency changes, the function is re-run and subscribers are notified.

---

## Example: Derived State in tink_state

```haxe
var email = new State("");
var password = new State("");
var confirmPassword = new State("");
var isValidEmail = Observable.auto(() ->
    email.value.indexOf("@") > 0 && email.value.indexOf(".") > 0
);
var isValidPassword = Observable.auto(() ->
    password.value.length >= 8
);
var passwordsMatch = Observable.auto(() ->
    password.value == confirmPassword.value
);
var canSubmit = Observable.auto(() ->
    isValidEmail.value && isValidPassword.value && passwordsMatch.value
);
```

---

## Difference to other frameworks: hot / cold

One difference of tink_state to other frameworks is how it defines "cold" and "hot":

- tink_state does NOT model cold observables where each subscriber gets its own data stream.
- Instead, **all subscribers to the same observable share the same computation and data stream**.
- "hot" and "cold" refers to **resource management**, not computation isolation

---

## Batched updates
By default, tink_state batches updates to avoid redundant notifications.

```haxe
var firstName = new State("John");
var lastName = new State("Doe");
var fullName = Observable.auto(() ->
  '${firstName.value} ${lastName.value}'
);
fullName.bind(name -> trace('Full name: $name'));
// Initial output: "Full name: John Doe"
// These updates are batched together
firstName.value = "Jane";
lastName.value = "Smith";
// Only outputs once: "Full name: Jane Smith"
```

---

## Operators like .map, .filter and .combine

```haxe
    var input = new State("  Hello World  ");
    var processed = input
      .map(s -> s.trim())                    // Remove whitespace
      .map(s -> s.toLowerCase())             // Convert to lowercase
      .map(s -> s.split(" "))                // Split into words
      .map(words -> words.join("-"));        // Join with dashes

    processed.bind(result -> trace('Processed: $result'));
    // Output: Processed: hello-world
```

- allows the use of operator chaining (similar to other frameworks)

---

## Example: Observable.auto() vs. operators

```haxe
var result = a.combine(b, (aVal, bVal) ->
  c.combine(d, (cVal, dVal) ->
    e.combine(f, (eVal, fVal) ->
      compute(aVal, bVal, cVal, dVal, eVal, fVal)
    ).value
  ).value
);

var result = Observable.auto(() -> {
  return compute(
    a.value, b.value, c.value,
    d.value, e.value, f.value
  );
});
```

Observable.auto() lets you combine complex data relationships easier than chaining operators.

---

## Automatic UI update

```haxe
canSubmit.bind(valid -> {
    submitButton.enabled = valid;
    submitButton.className = valid ? "enabled" : "disabled";
});
isValidEmail.bind(valid -> {
    emailField.className = valid ? "valid" : "invalid";
});
isValidPassword.bind(valid -> {
    passwordField.className = valid ? "valid" : "invalid";
});
passwordsMatch.bind(match -> {
    confirmField.className = match ? "valid" : "invalid";
    errorMessage.text = match ? "" : "Passwords don't match";
});
```

UI automatically updates when validation state changes

---

## Imperative vs. Declarative

```haxe
function validateForm() {
    var emailValid = validateEmail();
    var passwordValid = validatePassword();
    var passwordsMatch = checkPasswordsMatch();
    updateEmailUI(emailValid);
    updatePasswordUI(passwordValid);
    updateConfirmUI(passwordsMatch);
    updateSubmitButton(emailValid && passwordValid && passwordsMatch);
    if (emailValid && passwordValid && passwordsMatch) {
        hideErrors();
    } else {
        showErrors();
    }
}
```
```haxe
var formValid = Observable.auto(() =>
    isValidEmail.value &&
    isValidPassword.value &&
    passwordsMatch.value
);
formValid.bind(valid -> submitButton.enabled = valid);
```



