# If-Let and Let-Else Patterns in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [The if let Pattern](#the-if-let-pattern)
3. [if let with else](#if-let-with-else)
4. [if let else if Chains](#if-let-else-if-chains)
5. [while let](#while-let)
6. [let-else Pattern](#let-else-pattern)
7. [Combining Patterns](#combining-patterns)
8. [Quick Reference](#quick-reference)
9. [Common Patterns](#common-patterns)
10. [Best Practices](#best-practices)
11. [Common Pitfalls](#common-pitfalls)
12. [Summary](#summary)

---

## Introduction

The `if let` construct provides a concise way to handle values that match one pattern while ignoring others. It's syntactic sugar for a `match` that only cares about one case. Rust also provides `while let` for loops and `let-else` for early returns.

These patterns are particularly useful when:
- You only care about one variant of an enum
- A full `match` would be verbose with `_ => ()` arms
- You want to destructure and use a value conditionally

---

## The if let Pattern

### Basic Syntax

```rust
fn main() {
    let some_value = Some(3);

    // Using match (verbose)
    match some_value {
        Some(x) => println!("Got: {}", x),
        None => (), // We don't care about None
    }

    // Using if let (concise)
    if let Some(x) = some_value {
        println!("Got: {}", x);
    }
}
```

### How it Works

The `if let` takes a pattern and an expression separated by `=`. If the pattern matches, the code block executes with any bindings from the pattern available.

```rust
fn main() {
    let config_max = Some(3u8);

    // Pattern = Expression
    if let Some(max) = config_max {
        println!("The maximum is configured to be {}", max);
    }
    // If pattern doesn't match, nothing happens
}
```

### With Option

```rust
fn main() {
    let name: Option<String> = Some(String::from("Alice"));

    // Check and use the value
    if let Some(n) = name {
        println!("Hello, {}!", n);
    }

    // With None
    let empty: Option<String> = None;
    if let Some(n) = empty {
        println!("This won't print: {}", n);
    }
}
```

### With Result

```rust
use std::fs::File;

fn main() {
    let file = File::open("hello.txt");

    if let Ok(f) = file {
        println!("File opened successfully: {:?}", f);
    }

    // Handle error case
    let file = File::open("nonexistent.txt");
    if let Err(e) = file {
        println!("Failed to open file: {}", e);
    }
}
```

### With Custom Enums

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(String), // State name for quarters
}

fn main() {
    let coin = Coin::Quarter(String::from("Alaska"));

    // Only interested in quarters with state info
    if let Coin::Quarter(state) = coin {
        println!("State quarter from {}!", state);
    }
}
```

### Destructuring in if let

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point = Some(Point { x: 10, y: 20 });

    // Destructure the struct inside Option
    if let Some(Point { x, y }) = point {
        println!("Point is at ({}, {})", x, y);
    }

    // With nested enums
    enum Message {
        Move { x: i32, y: i32 },
        Write(String),
        Quit,
    }

    let msg = Message::Move { x: 5, y: 10 };

    if let Message::Move { x, y } = msg {
        println!("Moving to ({}, {})", x, y);
    }
}
```

---

## if let with else

When you need to handle the non-matching case:

```rust
fn main() {
    let some_value: Option<i32> = None;

    // if let with else
    if let Some(x) = some_value {
        println!("Got: {}", x);
    } else {
        println!("Got nothing");
    }

    // Equivalent match
    match some_value {
        Some(x) => println!("Got: {}", x),
        _ => println!("Got nothing"),
    }
}
```

### Practical Example

```rust
enum UserStatus {
    Active { name: String, role: String },
    Inactive,
    Banned { reason: String },
}

fn greet_user(status: UserStatus) {
    if let UserStatus::Active { name, role } = status {
        println!("Welcome, {} ({})!", name, role);
    } else {
        println!("Access denied");
    }
}

fn main() {
    let user = UserStatus::Active {
        name: String::from("Alice"),
        role: String::from("Admin"),
    };
    greet_user(user);

    let banned = UserStatus::Banned {
        reason: String::from("Spam"),
    };
    greet_user(banned);
}
```

---

## if let else if Chains

You can chain multiple `if let` statements:

```rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("Using your favorite color, {}, as the background", color);
    } else if is_tuesday {
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    } else {
        println!("Using blue as the background color");
    }
}
```

### Mixed Conditions

```rust
enum Temperature {
    Celsius(f64),
    Fahrenheit(f64),
}

fn describe_temp(temp: Option<Temperature>) {
    if let Some(Temperature::Celsius(c)) = temp {
        if c > 30.0 {
            println!("{}°C - It's hot!", c);
        } else if c < 10.0 {
            println!("{}°C - It's cold!", c);
        } else {
            println!("{}°C - It's pleasant", c);
        }
    } else if let Some(Temperature::Fahrenheit(f)) = temp {
        println!("{}°F", f);
    } else {
        println!("No temperature reading");
    }
}

fn main() {
    describe_temp(Some(Temperature::Celsius(35.0)));
    describe_temp(Some(Temperature::Fahrenheit(98.6)));
    describe_temp(None);
}
```

---

## while let

The `while let` loop continues as long as a pattern matches:

### Basic Usage

```rust
fn main() {
    let mut stack = Vec::new();
    stack.push(1);
    stack.push(2);
    stack.push(3);

    // Pop until empty
    while let Some(top) = stack.pop() {
        println!("{}", top);
    }
    // Prints: 3, 2, 1
}
```

### With Iterators

```rust
fn main() {
    let mut iter = (0..5).into_iter();

    while let Some(n) = iter.next() {
        println!("{}", n);
    }

    // More idiomatic: use for loop
    for n in 0..5 {
        println!("{}", n);
    }
}
```

### Processing Optional Values

```rust
fn main() {
    let values = vec![Some(1), Some(2), None, Some(4)];
    let mut iter = values.into_iter();

    // This stops at None
    while let Some(Some(value)) = iter.next() {
        println!("{}", value);
    }
    // Prints: 1, 2 (stops at None)
}
```

`while let` ends the loop the first time the pattern fails to match, it doesn't skip a
non-matching item and keep going. `iter.next()` still returns `Some(None)` for the third
element, but that doesn't match `Some(Some(value))`, so the loop stops there even though
a `Some(4)` remains after it.

### Nested while let

```rust
fn main() {
    let mut outer = vec![vec![1, 2], vec![3, 4]].into_iter();

    while let Some(mut inner) = outer.next() {
        while let Some(value) = inner.pop() {
            println!("{}", value);
        }
    }
}
```

### State Machine with while let

```rust
enum State {
    Running(i32),
    Paused(i32),
    Stopped,
}

fn main() {
    let mut state = State::Running(10);

    while let State::Running(count) | State::Paused(count) = state {
        println!("Count: {}", count);
        state = if count > 0 {
            State::Running(count - 1)
        } else {
            State::Stopped
        };
    }
}
```

`State` doesn't implement `Copy`, so each check of `while let ... = state` moves `state`
into the pattern; the loop only compiles because the body unconditionally assigns a new
value back to `state` before the next check. Leaving any path where `state` isn't
reassigned would be a use of a moved value on the following iteration.

---

## let-else Pattern

The `let-else` pattern (Rust 1.65+) provides a way to bind a pattern or diverge (return, break, continue, or panic):

### Basic Syntax

```rust
fn get_count_item(s: &str) -> (u64, &str) {
    let mut iter = s.splitn(2, ' ');

    // If pattern doesn't match, execute the else block which must diverge
    let Some(count_str) = iter.next() else {
        panic!("No count found");
    };

    let Some(item) = iter.next() else {
        panic!("No item found");
    };

    let Ok(count) = count_str.parse::<u64>() else {
        panic!("Invalid count");
    };

    (count, item)
}

fn main() {
    let (count, item) = get_count_item("5 apples");
    println!("{} {}", count, item);
}
```

### Early Return Pattern

```rust
fn process_user(id: Option<u32>) -> Result<String, &'static str> {
    // Early return if None
    let Some(user_id) = id else {
        return Err("No user ID provided");
    };

    // Continue with user_id bound
    Ok(format!("Processing user {}", user_id))
}

fn main() {
    println!("{:?}", process_user(Some(42)));
    println!("{:?}", process_user(None));
}
```

### With Destructuring

```rust
struct Config {
    host: Option<String>,
    port: Option<u16>,
}

fn connect(config: Config) -> Result<(), String> {
    let Some(host) = config.host else {
        return Err("Host not configured".to_string());
    };

    let Some(port) = config.port else {
        return Err("Port not configured".to_string());
    };

    println!("Connecting to {}:{}", host, port);
    Ok(())
}
```

### let-else vs if-let

`if let`'s binding only exists inside its block, so using it past the check means either
putting the rest of the function inside that block or returning early from within it, as
`process_option` does here. `let-else` binds into the surrounding scope instead: the
pattern's bindings are usable after the `let` statement, and only the failure path is
confined to the `else` block, which must diverge (`return`, `break`, `continue`, or
`panic!`) since there's no other way to give `x` a value.

```rust
fn process_option(opt: Option<i32>) -> i32 {
    // Using if let: the "no match" case falls through, so the match case
    // has to return early to skip it.
    if let Some(x) = opt {
        return x * 2;
    }
    0
}

fn process_option_let_else(opt: Option<i32>) -> i32 {
    // Using let-else: x is bound in the rest of the function, not just
    // inside a block.
    let Some(x) = opt else {
        return 0;
    };
    x * 2
}
```

### Complex Patterns in let-else

```rust
enum Message {
    Text { from: String, content: String },
    Image { from: String, data: Vec<u8> },
    System(String),
}

fn process_text_message(msg: Message) -> Option<String> {
    let Message::Text { from, content } = msg else {
        return None;  // Not a text message
    };

    Some(format!("{}: {}", from, content))
}

fn main() {
    let text = Message::Text {
        from: "Alice".to_string(),
        content: "Hello!".to_string(),
    };

    let image = Message::Image {
        from: "Bob".to_string(),
        data: vec![],
    };

    println!("{:?}", process_text_message(text));   // Some("Alice: Hello!")
    println!("{:?}", process_text_message(image));  // None
}
```

---

## Combining Patterns

### if let with && and ||

Let chains join a pattern match and a boolean condition, or several pattern matches,
into one `if` with `&&`, short-circuiting the same way ordinary `&&` does: the second
`let Some(y) = b` only runs if the first pattern already matched, and `y` is only in
scope once both have. Without let chains this would need nested `if let`s. This syntax
needs edition 2024.

```rust
fn main() {
    let x = Some(5);
    let y = true;

    // Combine if let with boolean conditions
    if let Some(n) = x && y {
        println!("Got {} and y is true", n);
    }

    // Multiple let conditions
    let a = Some(1);
    let b = Some(2);

    if let Some(x) = a && let Some(y) = b {
        println!("x = {}, y = {}", x, y);
    }
}
```

### Chained Conditions

```rust
fn main() {
    let values: Vec<Option<i32>> = vec![Some(1), None, Some(3)];

    for (index, value) in values.iter().enumerate() {
        if let Some(n) = value && *n > 2 {
            println!("Index {} has value {} > 2", index, n);
        }
    }
}
```

### Pattern Guards in if let

Note: `if let` doesn't directly support pattern guards like `match`, but you can nest conditions:

```rust
fn main() {
    let pair = Some((3, 5));

    // Can't do: if let Some((x, y)) if x < y = pair
    // Instead, use nested if:
    if let Some((x, y)) = pair {
        if x < y {
            println!("{} < {}", x, y);
        }
    }

    // Or use match with guard:
    match pair {
        Some((x, y)) if x < y => println!("{} < {}", x, y),
        _ => (),
    }
}
```

---

## Quick Reference

| Pattern | Use case | Example |
|---------|----------|---------|
| `if let` | Match single pattern | `if let Some(x) = opt { ... }` |
| `if let else` | Match or alternative | `if let Ok(v) = res { ... } else { ... }` |
| `while let` | Loop while matching | `while let Some(x) = iter.next() { ... }` |
| `let-else` | Match or diverge | `let Some(x) = opt else { return; };` |

---

## Common Patterns

### Optional Configuration

```rust
struct Config {
    debug_mode: Option<bool>,
    log_level: Option<String>,
    max_connections: Option<u32>,
}

fn apply_config(config: &Config) {
    if let Some(debug) = config.debug_mode {
        println!("Debug mode: {}", debug);
    }

    if let Some(ref level) = config.log_level {
        println!("Log level: {}", level);
    }

    if let Some(max) = config.max_connections {
        if max > 100 {
            println!("Warning: high connection limit ({})", max);
        }
    }
}
```

### Event Handling

```rust
enum Event {
    KeyPress { key: char, modifiers: u8 },
    MouseClick { x: i32, y: i32, button: u8 },
    Resize { width: u32, height: u32 },
    Quit,
}

fn handle_event(event: Event) {
    // Only handle keyboard events
    if let Event::KeyPress { key, modifiers } = event {
        println!("Key '{}' pressed with modifiers: {:08b}", key, modifiers);
    }

    // Or handle specific mouse events
    // if let Event::MouseClick { x, y, button: 0 } = event {
    //     println!("Left click at ({}, {})", x, y);
    // }
}
```

### Error Recovery

```rust
use std::fs;
use std::io;

fn read_config(path: &str) -> String {
    let result = fs::read_to_string(path);

    if let Ok(contents) = result {
        return contents;
    }

    // Fallback to default config
    String::from("default=true")
}

fn try_multiple_paths(paths: &[&str]) -> Option<String> {
    for path in paths {
        if let Ok(contents) = fs::read_to_string(path) {
            return Some(contents);
        }
    }
    None
}
```

### Parsing and Validation

```rust
fn parse_coordinate(input: &str) -> Option<(i32, i32)> {
    let parts: Vec<&str> = input.split(',').collect();

    if let [x_str, y_str] = parts.as_slice() {
        if let (Ok(x), Ok(y)) = (x_str.trim().parse(), y_str.trim().parse()) {
            return Some((x, y));
        }
    }
    None
}

fn main() {
    if let Some((x, y)) = parse_coordinate("10, 20") {
        println!("Coordinate: ({}, {})", x, y);
    }

    if let Some((x, y)) = parse_coordinate("invalid") {
        println!("This won't print: ({}, {})", x, y);
    } else {
        println!("Failed to parse coordinate");
    }
}
```

### Working with Iterators

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Find first even number
    if let Some(first_even) = numbers.iter().find(|&&x| x % 2 == 0) {
        println!("First even: {}", first_even);
    }

    // Get element at index
    if let Some(third) = numbers.get(2) {
        println!("Third element: {}", third);
    }

    // Check last element
    if let Some(&last) = numbers.last() {
        println!("Last element: {}", last);
    }
}
```

---

## Best Practices

### When to Use if let vs match

**Use `if let` when:**
- You only care about one pattern
- The non-matching case does nothing or has simple handling
- Code would have `_ => ()` in a match

```rust
// Good use of if let
fn log_error(result: Result<(), String>) {
    if let Err(e) = result {
        eprintln!("Error: {}", e);
    }
}
```

**Use `match` when:**
- You need to handle multiple variants
- All cases need explicit handling
- You want exhaustiveness checking

```rust
// Better as match
fn describe_option(opt: Option<i32>) -> &'static str {
    match opt {
        Some(n) if n > 0 => "positive",
        Some(n) if n < 0 => "negative",
        Some(0) => "zero",
        None => "nothing",
    }
}
```

### When to Use let-else

**Use `let-else` when:**
- You need to extract a value or return early
- The binding should be available after the check
- You're implementing a guard clause pattern

```rust
// Good use of let-else
fn process_data(data: Option<Vec<u8>>) -> Result<String, &'static str> {
    let Some(bytes) = data else {
        return Err("No data provided");
    };

    let Ok(text) = String::from_utf8(bytes) else {
        return Err("Invalid UTF-8");
    };

    Ok(text.to_uppercase())
}
```

---

## Common Pitfalls

### Deeply nested if let

Deeply nested `if let` can be hard to read:

```rust
// Avoid this
fn nested_bad(a: Option<Option<Option<i32>>>) {
    if let Some(b) = a {
        if let Some(c) = b {
            if let Some(d) = c {
                println!("{}", d);
            }
        }
    }
}

// Better: use methods or flatten
fn nested_good(a: Option<Option<Option<i32>>>) {
    if let Some(d) = a.flatten().flatten() {
        println!("{}", d);
    }

    // Or with and_then
    if let Some(d) = a.and_then(|b| b).and_then(|c| c) {
        println!("{}", d);
    }
}
```

### Moving a value into if let

Being careless with ownership in `if let` moves the matched value out:

```rust
fn main() {
    let name = Some(String::from("Alice"));

    // This moves name
    // if let Some(n) = name {
    //     println!("{}", n);
    // }
    // println!("{:?}", name); // ERROR: name was moved

    // Use ref to borrow
    if let Some(ref n) = name {
        println!("{}", n);
    }
    println!("{:?}", name); // OK

    // Or match on reference
    if let Some(n) = &name {
        println!("{}", n);
    }
    println!("{:?}", name); // OK
}
```

---

## Summary

`if let` is syntactic sugar for a `match` with one arm plus a `_ => ()`, so it reduces
boilerplate when only one pattern matters. `while let` loops for as long as a pattern
keeps matching, and `let-else` (Rust 1.65+) binds a pattern or diverges in one step,
keeping the binding in scope afterward instead of only inside a block. Prefer `match`
over `if let` when multiple cases need explicit handling or exhaustiveness checking
matters, and watch ownership when matching by value: use `ref` or match on a reference
when the original binding still needs to be used afterward.
