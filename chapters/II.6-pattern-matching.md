# Pattern Matching in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [The match Expression](#the-match-expression)
3. [Pattern Types](#pattern-types)
4. [Destructuring](#destructuring)
5. [Match Guards](#match-guards)
6. [Binding with @](#binding-with-)
7. [Matching References](#matching-references)
8. [Or Patterns](#or-patterns)
9. [Exhaustiveness and the Wildcard](#exhaustiveness-and-the-wildcard)
10. [Advanced Patterns](#advanced-patterns)
11. [Quick Reference](#quick-reference)
12. [Common Patterns](#common-patterns)
13. [Common Pitfalls](#common-pitfalls)
14. [Summary](#summary)

---

## Introduction

Pattern matching is one of Rust's most powerful features. It allows you to compare a value against a series of patterns and execute code based on which pattern matches. Pattern matching in Rust is:

- **Exhaustive**: The compiler ensures all possible cases are handled
- **Expressive**: Supports complex destructuring and binding
- **Safe**: Prevents common bugs like missing cases

---

## The match Expression

### Basic Syntax

```rust
fn main() {
    let number = 3;

    match number {
        1 => println!("One!"),
        2 => println!("Two!"),
        3 => println!("Three!"),
        _ => println!("Something else"),
    }
}
```

### match as an Expression

`match` returns a value, making it an expression:

```rust
fn main() {
    let number = 3;

    let word = match number {
        1 => "one",
        2 => "two",
        3 => "three",
        _ => "other",
    };

    println!("Number is: {}", word);
}
```

### Multiple Statements in Arms

Use curly braces for multiple statements:

```rust
fn main() {
    let number = 3;

    let result = match number {
        1 => {
            println!("Found one!");
            "one"
        }
        2 => {
            println!("Found two!");
            "two"
        }
        _ => {
            println!("Found something else!");
            "other"
        }
    };

    println!("Result: {}", result);
}
```

### Matching Enums

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn move_player(direction: Direction) {
    match direction {
        Direction::North => println!("Moving north"),
        Direction::South => println!("Moving south"),
        Direction::East => println!("Moving east"),
        Direction::West => println!("Moving west"),
    }
}

fn main() {
    move_player(Direction::North);
}
```

### Matching Option

```rust
fn main() {
    let some_number = Some(5);
    let no_number: Option<i32> = None;

    match some_number {
        Some(n) => println!("Got a number: {}", n),
        None => println!("No number"),
    }

    // Using the matched value
    let doubled = match some_number {
        Some(n) => Some(n * 2),
        None => None,
    };
}
```

### Matching Result

```rust
use std::fs::File;

fn main() {
    let file_result = File::open("hello.txt");

    let file = match file_result {
        Ok(f) => f,
        Err(error) => {
            panic!("Problem opening the file: {:?}", error);
        }
    };
}
```

---

## Pattern Types

### Literal Patterns

Match against literal values:

```rust
fn main() {
    let x = 1;

    match x {
        1 => println!("one"),
        2 => println!("two"),
        3 => println!("three"),
        _ => println!("anything"),
    }

    // String literals
    let greeting = "hello";
    match greeting {
        "hello" => println!("Hi!"),
        "goodbye" => println!("Bye!"),
        _ => println!("Unknown greeting"),
    }

    // Character literals
    let c = 'a';
    match c {
        'a'..='z' => println!("lowercase letter"),
        'A'..='Z' => println!("uppercase letter"),
        _ => println!("something else"),
    }
}
```

### Variable Patterns

Bind matched values to variables:

```rust
fn main() {
    let x = Some(5);

    match x {
        Some(value) => println!("Got: {}", value),  // value is bound
        None => println!("Nothing"),
    }

    // The variable shadows outer scope
    let y = 10;
    match x {
        Some(y) => println!("Matched y = {}", y),  // y shadows outer y
        None => println!("No match"),
    }
    println!("Outer y = {}", y);  // Still 10
}
```

### Range Patterns

Match a range of values:

```rust
fn main() {
    let x = 5;

    match x {
        1..=5 => println!("one through five"),   // Inclusive range
        6..=10 => println!("six through ten"),
        _ => println!("something else"),
    }

    // Character ranges
    let c = 'c';
    match c {
        'a'..='j' => println!("early ASCII letter"),
        'k'..='z' => println!("late ASCII letter"),
        _ => println!("something else"),
    }
}
```

### Wildcard Pattern

The `_` pattern matches anything:

```rust
fn main() {
    let x = 5;

    match x {
        1 => println!("one"),
        _ => println!("anything else"),  // Catches all other cases
    }

    // Ignore parts of a value
    let point = (3, 5);
    match point {
        (0, _) => println!("On the y-axis"),
        (_, 0) => println!("On the x-axis"),
        (x, _) => println!("x is {}", x),
    }
}
```

### Rest Pattern (..)

Ignore remaining parts:

```rust
fn main() {
    let numbers = (1, 2, 3, 4, 5);

    match numbers {
        (first, .., last) => {
            println!("First: {}, Last: {}", first, last);
        }
    }

    // With structs
    struct Point3D { x: i32, y: i32, z: i32 }
    let point = Point3D { x: 1, y: 2, z: 3 };

    match point {
        Point3D { x, .. } => println!("x is {}", x),
    }
}
```

---

## Destructuring

### Destructuring Structs

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    // Full destructuring
    match p {
        Point { x, y } => println!("x: {}, y: {}", x, y),
    }

    // Partial destructuring with specific values
    match p {
        Point { x: 0, y } => println!("On y-axis at {}", y),
        Point { x, y: 0 } => println!("On x-axis at {}", x),
        Point { x, y } => println!("At ({}, {})", x, y),
    }

    // Renaming fields
    match p {
        Point { x: a, y: b } => println!("a: {}, b: {}", a, b),
    }
}
```

### Destructuring Enums

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let msg = Message::ChangeColor(0, 160, 255);

    match msg {
        Message::Quit => println!("Quit"),
        Message::Move { x, y } => {
            println!("Move to ({}, {})", x, y);
        }
        Message::Write(text) => println!("Text: {}", text),
        Message::ChangeColor(r, g, b) => {
            println!("Change color to ({}, {}, {})", r, g, b);
        }
    }
}
```

### Destructuring Tuples

```rust
fn main() {
    let tuple = (1, "hello", 3.14);

    match tuple {
        (a, b, c) => println!("{}, {}, {}", a, b, c),
    }

    // Nested tuples
    let nested = ((1, 2), (3, 4));
    match nested {
        ((a, b), (c, d)) => println!("{} {} {} {}", a, b, c, d),
    }

    // Mixed with ignoring
    let point = (3, -5);
    match point {
        (x, y) if x == y => println!("On diagonal"),
        (x, _) if x == 0 => println!("On y-axis"),
        (_, y) if y == 0 => println!("On x-axis"),
        (x, y) => println!("At ({}, {})", x, y),
    }
}
```

### Destructuring Arrays and Slices

```rust
fn main() {
    let arr = [1, 2, 3];

    match arr {
        [1, _, _] => println!("Starts with 1"),
        [_, 2, _] => println!("Middle is 2"),
        _ => println!("Something else"),
    }

    // Slices
    let slice: &[i32] = &[1, 2, 3, 4, 5];

    match slice {
        [] => println!("Empty"),
        [single] => println!("Single: {}", single),
        [first, second] => println!("Two: {}, {}", first, second),
        [first, .., last] => println!("First: {}, Last: {}", first, last),
    }

    // Binding slice parts
    match slice {
        [first, rest @ ..] => {
            println!("First: {}, Rest: {:?}", first, rest);
        }
        [] => println!("Empty"),
    }
}
```

### Nested Destructuring

```rust
enum Color {
    Rgb(i32, i32, i32),
    Hsv(i32, i32, i32),
}

enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(Color),
}

fn main() {
    let msg = Message::ChangeColor(Color::Hsv(0, 160, 255));

    match msg {
        Message::ChangeColor(Color::Rgb(r, g, b)) => {
            println!("RGB: {}, {}, {}", r, g, b);
        }
        Message::ChangeColor(Color::Hsv(h, s, v)) => {
            println!("HSV: {}, {}, {}", h, s, v);
        }
        _ => (),
    }
}
```

---

## Match Guards

Match guards add extra conditions to patterns. A guard only runs after its pattern
already matches, and if the guard's condition is false, `match` moves on to the next
arm rather than failing, which is why an arm with a guard doesn't count toward
exhaustiveness on its own: `Some(x) if x < 5` doesn't cover every `Some(x)`, so a
`match` relying on it still needs a later arm (or wildcard) to handle the case where
the guard doesn't hold.

```rust
fn main() {
    let num = Some(4);

    match num {
        Some(x) if x < 5 => println!("less than five: {}", x),
        Some(x) => println!("{}", x),
        None => (),
    }

    // Multiple conditions
    let x = 4;
    let y = false;

    match x {
        4 | 5 | 6 if y => println!("yes"),
        _ => println!("no"),
    }

    // Using external variables
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Ok(age) = age {
        match age {
            n if n < 18 => println!("Minor"),
            n if n < 65 => println!("Adult"),
            _ => println!("Senior"),
        }
    }
}
```

### Complex Guards

```rust
fn main() {
    let pair = (2, -2);

    match pair {
        (x, y) if x == y => println!("Equal"),
        (x, y) if x + y == 0 => println!("Sum is zero"),
        (x, _) if x % 2 == 0 => println!("First is even"),
        _ => println!("No match"),
    }

    // With Option
    let robot_name: Option<String> = Some(String::from("Bors"));

    match robot_name {
        Some(ref name) if name.starts_with("B") => {
            println!("Name starts with B: {}", name);
        }
        Some(name) => println!("Name: {}", name),
        None => println!("No name"),
    }
}
```

---

## Binding with @

The `@` operator lets you bind a value while also testing it:

```rust
fn main() {
    let x = 5;

    match x {
        // Bind the value to `n` while testing the range
        n @ 1..=5 => println!("Got a small number: {}", n),
        n @ 6..=10 => println!("Got a medium number: {}", n),
        n => println!("Got a large number: {}", n),
    }
}
```

### @ with Enums

```rust
enum Message {
    Hello { id: i32 },
}

fn main() {
    let msg = Message::Hello { id: 5 };

    match msg {
        Message::Hello {
            id: id_variable @ 3..=7,
        } => println!("Found an id in range: {}", id_variable),
        Message::Hello { id: 10..=12 } => {
            println!("Found an id in another range")
        }
        Message::Hello { id } => println!("Found some other id: {}", id),
    }
}
```

### @ with Structs and Patterns

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 5, y: 10 };

    match p {
        Point {
            x: x_val @ 0..=10,
            y: y_val @ 0..=20,
        } => {
            println!("In bounds: ({}, {})", x_val, y_val);
        }
        Point { x, y } => {
            println!("Out of bounds: ({}, {})", x, y);
        }
    }

    // Binding entire struct variants
    let opt = Some(42);
    match opt {
        some @ Some(_) => println!("Got {:?}", some),
        None => println!("Nothing"),
    }
}
```

---

## Matching References

### Matching with References

A pattern has to line up with what it's matching: matching `reference: &i32` with `&val`
peels off the reference so `val` binds the `i32` itself, while matching `*reference`
already dereferences before the pattern runs, so a plain `val` binds the same `i32`
either way. `ref` runs the opposite direction: it takes a pattern matching a plain value
and makes the binding a reference into it instead of moving or copying it out, and `ref
mut` does the same for a mutable reference.

```rust
fn main() {
    let reference = &4;

    match reference {
        &val => println!("Got a value: {}", val),
    }

    // Dereferencing
    match *reference {
        val => println!("Got a value: {}", val),
    }

    // ref keyword creates a reference
    let value = 5;
    match value {
        ref r => println!("Got a reference: {}", r),
    }

    // ref mut for mutable reference
    let mut mut_value = 5;
    match mut_value {
        ref mut m => {
            *m += 10;
            println!("Modified: {}", m);
        }
    }
}
```

In modern Rust, match ergonomics usually make `ref` unnecessary: matching a reference
with a plain variable pattern (`Some(x)` against `&Some(String::from("hi"))`, for
instance) automatically binds `x` as a reference instead of trying to move out of the
borrow. `ref` still matters when you're matching an owned value directly, as above, and
want a borrow instead of a move.

### Matching Option with References

```rust
fn main() {
    let s = Some(String::from("hello"));

    // This would move s
    // match s {
    //     Some(inner) => println!("{}", inner),
    //     None => (),
    // }
    // println!("{:?}", s); // Error: s was moved

    // Use ref to borrow instead
    match s {
        Some(ref inner) => println!("{}", inner),
        None => (),
    }
    println!("{:?}", s); // OK: s wasn't moved

    // Or match on a reference
    match &s {
        Some(inner) => println!("{}", inner),
        None => (),
    }
    println!("{:?}", s); // OK
}
```

### ref and ref mut

```rust
fn main() {
    let mut tuple = (5, String::from("hello"));

    match tuple {
        (ref x, ref y) => {
            println!("x: {}, y: {}", x, y);
        }
    }

    match tuple {
        (ref mut x, ref mut y) => {
            *x += 1;
            y.push_str(" world");
        }
    }

    println!("{:?}", tuple); // (6, "hello world")
}
```

---

## Or Patterns

Use `|` to match multiple patterns:

```rust
fn main() {
    let x = 1;

    match x {
        1 | 2 => println!("one or two"),
        3 | 4 => println!("three or four"),
        _ => println!("anything"),
    }

    // With ranges
    let c = 'c';
    match c {
        'a'..='z' | 'A'..='Z' => println!("letter"),
        '0'..='9' => println!("digit"),
        _ => println!("other"),
    }
}
```

### Or Patterns with Enums

```rust
enum Animal {
    Dog,
    Cat,
    Bird,
    Fish,
}

fn main() {
    let animal = Animal::Cat;

    match animal {
        Animal::Dog | Animal::Cat => println!("Mammal"),
        Animal::Bird => println!("Bird"),
        Animal::Fish => println!("Fish"),
    }
}
```

### Or Patterns with Bindings

```rust
enum Command {
    Move { x: i32, y: i32 },
    Jump { height: i32 },
    Stop,
}

fn main() {
    let cmd = Command::Move { x: 5, y: 10 };

    // All variants in or-pattern must bind same names
    match cmd {
        Command::Move { x, y } | Command::Jump { height: x, .. } if x > 0 => {
            // Note: this doesn't compile as written
            // because Jump binds `height` not `y`
        }
        _ => (),
    }

    // Correct version with same bindings
    let value = Some(5);
    match value {
        Some(x) | None if false => { // x bound in Some, not None - ERROR
            // This won't compile
        }
        _ => (),
    }
}
```

---

## Exhaustiveness and the Wildcard

### Exhaustive Matching

Rust requires all possibilities to be covered:

```rust
enum Color {
    Red,
    Green,
    Blue,
}

fn describe_color(color: Color) -> &'static str {
    match color {
        Color::Red => "red",
        Color::Green => "green",
        Color::Blue => "blue",
        // All variants covered, no wildcard needed
    }
}

// This would not compile:
// fn incomplete(color: Color) -> &'static str {
//     match color {
//         Color::Red => "red",
//         // ERROR: missing Green and Blue
//     }
// }
```

### Using Wildcards

```rust
fn main() {
    let x = 5;

    // _ matches anything
    match x {
        1 => println!("one"),
        _ => println!("something else"),
    }

    // _ doesn't bind
    let opt = Some(5);
    match opt {
        Some(_) => println!("Something"),
        None => println!("Nothing"),
    }
}
```

### Ignoring Values with _

```rust
fn main() {
    // Ignore function parameter
    fn foo(_: i32, y: i32) {
        println!("y is {}", y);
    }
    foo(3, 4);

    // Ignore parts of value
    let numbers = (1, 2, 3, 4, 5);
    match numbers {
        (first, _, third, _, fifth) => {
            println!("first: {}, third: {}, fifth: {}", first, third, fifth);
        }
    }

    // Ignore unused variable warning
    let _unused = 42;  // No warning
}
```

### Default Arms with _

```rust
fn main() {
    let dice_roll = 9;

    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        _ => reroll(),  // Default case
    }

    // Using _ to do nothing
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        _ => (),  // Do nothing for other values
    }
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
fn reroll() {}
```

---

## Advanced Patterns

### Matching Multiple Conditions

```rust
fn main() {
    let pair = (0, -2);

    match pair {
        (0, y) => println!("First is 0, y = {}", y),
        (x, 0) => println!("x = {}, second is 0", x),
        _ => println!("It doesn't matter what they are"),
    }

    // Combining techniques
    let triple = (0, -2, 3);
    match triple {
        (0, y, z) if y < 0 && z > 0 => {
            println!("First is 0, y is negative, z is positive");
        }
        (0 | 1, ..) => println!("First is 0 or 1"),
        _ => println!("Something else"),
    }
}
```

### Matching Box and Smart Pointers

```rust
fn main() {
    let boxed = Box::new(5);

    match boxed {
        box_val => println!("Boxed value: {}", box_val),
    }

    // Dereferencing
    match *boxed {
        5 => println!("Five!"),
        _ => println!("Not five"),
    }
}
```

Stable Rust has no pattern that reaches inside a `Box` directly; `box_val` above just
binds the whole `Box<i32>`, moving it out of `boxed`. To match against the value a `Box`
points to, dereference it first, as `*boxed` does in the second `match`, which matches
on the `i32` underneath.

### Matching with Lifetimes

```rust
fn match_lifetime<'a>(s: &'a Option<String>) -> Option<&'a str> {
    match s {
        Some(ref inner) => Some(inner.as_str()),
        None => None,
    }
}

fn main() {
    let s = Some(String::from("hello"));
    if let Some(slice) = match_lifetime(&s) {
        println!("Got: {}", slice);
    }
}
```

The returned `&str` is a view into the `String` stored inside `s`, so it can't outlive
`s`. The shared lifetime `'a` on both the parameter and the return type tells the
compiler that constraint directly, rather than leaving it to infer a relationship
between two independent borrows. See [Chapter 9: Lifetimes](III.9-lifetimes.md) for why
this annotation is required here.

### const in Patterns

```rust
const THRESHOLD: i32 = 10;

fn main() {
    let value = 15;

    match value {
        0 => println!("Zero"),
        THRESHOLD => println!("At threshold"),
        n if n < THRESHOLD => println!("Below threshold: {}", n),
        n => println!("Above threshold: {}", n),
    }
}
```

Whether a name in a pattern tests a value or binds one depends on what the name refers
to, not on its position: `THRESHOLD` is a `const`, so the arm checks `value == THRESHOLD`,
while `n` isn't in scope anywhere else, so it's treated as a fresh binding that matches
unconditionally. This is also why constants are conventionally `SCREAMING_SNAKE_CASE`:
naming one in lowercase would silently make it look like a binding pattern instead of an
equality check, and the compiler warns about exactly that.

### Matching Generic Types

```rust
fn process<T: std::fmt::Debug>(opt: Option<T>) {
    match opt {
        Some(value) => println!("Got: {:?}", value),
        None => println!("Nothing"),
    }
}

fn main() {
    process(Some(42));
    process(Some("hello"));
    process::<i32>(None);
}
```

---

## Quick Reference

| Pattern | Description |
|---------|-------------|
| `_` | Matches anything, ignores value |
| `x` | Binds matched value to `x` |
| `1 \| 2` | Matches 1 or 2 |
| `1..=5` | Matches range 1 to 5 inclusive |
| `(a, b)` | Destructures tuple |
| `Point { x, y }` | Destructures struct |
| `Some(x)` | Destructures enum variant |
| `ref x` | Binds by reference |
| `x @ 1..=5` | Binds `x` while testing range |
| `..` | Ignores rest of value |

---

## Common Patterns

### Replacing if-else Chains

```rust
fn classify_number(n: i32) -> &'static str {
    match n {
        0 => "zero",
        1..=9 => "single digit",
        10..=99 => "double digit",
        100..=999 => "triple digit",
        _ => "many digits",
    }
}
```

### State Machines

```rust
enum State {
    Idle,
    Running { speed: u32 },
    Paused,
    Stopped,
}

fn transition(state: State, event: &str) -> State {
    match (state, event) {
        (State::Idle, "start") => State::Running { speed: 10 },
        (State::Running { .. }, "pause") => State::Paused,
        (State::Running { .. }, "stop") => State::Stopped,
        (State::Paused, "resume") => State::Running { speed: 10 },
        (State::Paused, "stop") => State::Stopped,
        (state, _) => state,  // No transition
    }
}
```

### Error Handling Patterns

```rust
use std::io::{self, Read};
use std::fs::File;

fn read_file(path: &str) -> Result<String, io::Error> {
    let mut file = match File::open(path) {
        Ok(f) => f,
        Err(e) => return Err(e),
    };

    let mut contents = String::new();
    match file.read_to_string(&mut contents) {
        Ok(_) => Ok(contents),
        Err(e) => Err(e),
    }
}
```

---

## Common Pitfalls

- A variable pattern inside a match arm introduces a new binding that shadows any
  outer variable of the same name for the rest of that arm; it doesn't refer back to
  the outer one (see the `Some(y)` example under "Variable Patterns").
- Matching a non-`Copy` value like `Option<String>` by value moves it out, so the
  original binding can't be used afterward. Match on a reference (`match &s`) or bind
  with `ref` inside the pattern (`Some(ref inner)`) to borrow instead of moving.
- Every alternative in an or-pattern (`A | B`) must introduce the same set of bindings
  with the same types; a pattern that binds a name in one alternative but not another
  won't compile.
- `match` must be exhaustive: leaving out a variant of an enum (with no wildcard arm)
  is a compile error, not a runtime one.

---

## Summary

Pattern matching is exhaustive (the compiler checks that every case is handled),
supports destructuring structs, enums, tuples, and slices in the pattern itself, and
can add conditions with match guards or capture a value while testing it with `@`.
Or-patterns (`|`) match several alternatives in one arm. Because it's checked at
compile time, matching against structured data is generally preferred over chains of
if-else statements.
