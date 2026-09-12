# Custom Types in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Structs](#structs)
3. [Tuple Structs](#tuple-structs)
4. [Unit Structs](#unit-structs)
5. [Enums](#enums)
6. [Type Aliases](#type-aliases)
7. [Newtype Pattern](#newtype-pattern)
8. [Constants in Types](#constants-in-types)
9. [Generic Types](#generic-types)
10. [Methods and Associated Functions](#methods-and-associated-functions)
11. [Quick Reference](#quick-reference)
12. [Common Patterns](#common-patterns)
13. [Common Pitfalls](#common-pitfalls)
14. [Summary](#summary)

Visibility and privacy moved to
[Chapter 4: Modules, Crates & Visibility](I.4-modules-crates-visibility.md); the derive
macro reference moved to
[Chapter 15: Common Derivable Traits](IV.15-common-derivable-traits-quick-reference.md).

---

## Introduction

Rust provides several ways to define custom types:

- **Structs**: Group related data together
- **Enums**: Define a type with multiple variants
- **Type aliases**: Give existing types new names
- **Newtype pattern**: Create distinct types from existing ones

Custom types help you:
- Model your domain accurately
- Enforce invariants at compile time
- Make code self-documenting
- Enable the type system to catch bugs

---

## Structs

### Basic Struct Definition

```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    // Creating an instance
    let user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        sign_in_count: 1,
        active: true,
    };

    // Accessing fields
    println!("User: {}", user.username);
}
```

### Mutable Struct Instances

```rust
fn main() {
    let mut user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        sign_in_count: 1,
        active: true,
    };

    // Modify fields (entire struct must be mutable)
    user.email = String::from("newalice@example.com");
    user.sign_in_count += 1;
}

struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
```

Rust has no per-field `mut`: mutability belongs to the binding, not the field. Marking
`user` as `mut` makes every field assignable through it; there's no way to make only
`email` mutable while leaving `sign_in_count` fixed.

### Field Init Shorthand

```rust
fn build_user(username: String, email: String) -> User {
    User {
        username,  // Same as username: username
        email,     // Same as email: email
        sign_in_count: 1,
        active: true,
    }
}

struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
```

### Struct Update Syntax

```rust
fn main() {
    let user1 = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        sign_in_count: 1,
        active: true,
    };

    // Create user2 from user1, overriding some fields
    let user2 = User {
        email: String::from("bob@example.com"),
        username: String::from("bob"),
        ..user1  // Get remaining fields from user1
    };

    // Note: user1.sign_in_count and user1.active were copied (they're Copy)
    // If we had used user1.username or user1.email, user1 would be partially moved
}

struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
```

`..user1` fills in every field you didn't name explicitly, taken from `user1`. It follows
the same rules as any other field access: a `Copy` field like `sign_in_count` is copied
out, but a non-`Copy` field like `username` would be moved out, leaving `user1` partially
moved (see Common Pitfalls below for what that means for `user1`'s later use).

### Struct with References (Requires Lifetimes)

```rust
struct UserRef<'a> {
    username: &'a str,
    email: &'a str,
}

fn main() {
    let name = String::from("alice");
    let email = String::from("alice@example.com");

    let user = UserRef {
        username: &name,
        email: &email,
    };

    println!("{}: {}", user.username, user.email);
}
```

`UserRef` stores references instead of owned `String`s, and a struct holding a reference
must say how long that reference is guaranteed to stay valid. The `'a` here ties the
lifetime of the fields to the lifetime of whatever `name` and `email` point at, so the
compiler can reject a `UserRef` that would outlive the data it borrows. See
[Chapter 9: Lifetimes](III.9-lifetimes.md) for the full mechanics.

---

## Tuple Structs

Tuple structs have a name but unnamed fields:

```rust
// Define tuple structs
struct Color(u8, u8, u8);
struct Point(f64, f64, f64);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0.0, 0.0, 0.0);

    // Access fields by index
    println!("R: {}, G: {}, B: {}", black.0, black.1, black.2);
    println!("x: {}, y: {}, z: {}", origin.0, origin.1, origin.2);

    // Destructuring
    let Color(r, g, b) = black;
    println!("Red: {}", r);
}
```

### When to Use Tuple Structs

```rust
// Good for simple wrappers
struct Meters(f64);
struct Seconds(f64);

fn calculate_speed(distance: Meters, time: Seconds) -> f64 {
    distance.0 / time.0
}

fn main() {
    let dist = Meters(100.0);
    let time = Seconds(9.58);

    // This provides type safety:
    // calculate_speed(time, dist) would be a compile error!

    let speed = calculate_speed(dist, time);
    println!("Speed: {} m/s", speed);
}
```

---

## Unit Structs

Unit structs have no fields:

```rust
struct AlwaysTrue;
struct Marker;

fn main() {
    let _t = AlwaysTrue;
    let _m = Marker;
}
```

### Use Cases for Unit Structs

```rust
// 1. Marker types for type-level programming
struct Kilometers;
struct Miles;

struct Distance<Unit> {
    value: f64,
    _marker: std::marker::PhantomData<Unit>,
}

// `Distance<Kilometers>` and `Distance<Miles>` are different types even though
// `Unit` never appears in a real field. PhantomData<Unit> tells the compiler to
// treat the type parameter as used (so it's not rejected as unused), without
// actually storing a `Unit` value at runtime, since a unit struct has no data
// to store in the first place.

// 2. Implementing traits without data
struct EmptyIterator;

impl Iterator for EmptyIterator {
    type Item = ();
    fn next(&mut self) -> Option<Self::Item> {
        None
    }
}

// 3. Error types
struct NotFoundError;

impl std::fmt::Display for NotFoundError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Resource not found")
    }
}
```

---

## Enums

### Basic Enum Definition

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn main() {
    let dir = Direction::North;

    match dir {
        Direction::North => println!("Going north!"),
        Direction::South => println!("Going south!"),
        Direction::East => println!("Going east!"),
        Direction::West => println!("Going west!"),
    }
}
```

### Enums with Data

```rust
enum Message {
    Quit,                       // No data
    Move { x: i32, y: i32 },   // Named fields (like struct)
    Write(String),              // Single value (like tuple struct)
    ChangeColor(u8, u8, u8),   // Multiple values
}

fn main() {
    let messages = vec![
        Message::Quit,
        Message::Move { x: 10, y: 20 },
        Message::Write(String::from("hello")),
        Message::ChangeColor(255, 128, 0),
    ];

    for msg in messages {
        process_message(msg);
    }
}

fn process_message(msg: Message) {
    match msg {
        Message::Quit => println!("Quitting"),
        Message::Move { x, y } => println!("Moving to ({}, {})", x, y),
        Message::Write(text) => println!("Writing: {}", text),
        Message::ChangeColor(r, g, b) => println!("Color: ({}, {}, {})", r, g, b),
    }
}
```

### Option Enum

```rust
// Defined in standard library:
// enum Option<T> {
//     Some(T),
//     None,
// }

fn main() {
    let some_number: Option<i32> = Some(5);
    let no_number: Option<i32> = None;

    // Must handle both cases
    match some_number {
        Some(n) => println!("Got: {}", n),
        None => println!("Got nothing"),
    }

    // Common methods
    let value = some_number.unwrap_or(0);
    let mapped = some_number.map(|n| n * 2);
    let is_some = some_number.is_some();
}
```

### Result Enum

```rust
// Defined in standard library:
// enum Result<T, E> {
//     Ok(T),
//     Err(E),
// }

fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    match divide(10.0, 2.0) {
        Ok(result) => println!("Result: {}", result),
        Err(e) => println!("Error: {}", e),
    }
}
```

### Enum with Methods

```rust
enum Status {
    Active,
    Inactive,
    Pending { since: String },
}

impl Status {
    fn is_active(&self) -> bool {
        matches!(self, Status::Active)
    }

    fn describe(&self) -> String {
        match self {
            Status::Active => String::from("Currently active"),
            Status::Inactive => String::from("Not active"),
            Status::Pending { since } => format!("Pending since {}", since),
        }
    }
}

fn main() {
    let status = Status::Pending {
        since: String::from("2024-01-01"),
    };

    println!("{}", status.describe());
    println!("Is active: {}", status.is_active());
}
```

### C-like Enums

```rust
#[derive(Debug)]
enum HttpStatus {
    Ok = 200,
    NotFound = 404,
    InternalServerError = 500,
}

fn main() {
    let status = HttpStatus::NotFound;

    println!("Status code: {}", status as i32);  // 404

    // Can use repr for specific integer types
    #[repr(u8)]
    enum Small {
        A = 1,
        B = 2,
        C = 3,
    }
}
```

Without an explicit `= value`, Rust still assigns each variant a discriminant (starting
at 0 and counting up), but that discriminant is an implementation detail you can't rely
on unless you set it yourself, as `HttpStatus` does here. The cast `status as i32` only
works because the enum has no variants carrying data; `#[repr(u8)]` (or `u16`, `i32`, and
so on) additionally fixes the discriminant's storage type, which matters when the enum
needs a guaranteed layout, for example when passing it across an FFI boundary.

---

## Type Aliases

### Basic Type Aliases

```rust
// Create an alias for a type
type Kilometers = i32;
type UserId = u64;

fn main() {
    let distance: Kilometers = 100;
    let user_id: UserId = 12345;

    // Note: these are the SAME type, just different names
    let d: i32 = distance;  // No conversion needed
}
```

### Simplifying Complex Types

```rust
use std::collections::HashMap;

// Long type
// HashMap<String, Vec<(i32, String)>>

// Create an alias
type UserActions = HashMap<String, Vec<(i32, String)>>;

fn process(actions: &UserActions) {
    for (user, action_list) in actions {
        println!("{}: {} actions", user, action_list.len());
    }
}
```

### Type Aliases with Generics

```rust
type Result<T> = std::result::Result<T, std::io::Error>;

// Now you can write:
fn read_file() -> Result<String> {
    std::fs::read_to_string("file.txt")
}

// Instead of:
// fn read_file() -> std::result::Result<String, std::io::Error>
```

This alias deliberately shadows the `Result` the prelude brings into scope. Inside a
module that only ever fails with `std::io::Error`, redefining `Result<T>` to already
bake in the error type means every function signature in the module can write `Result<T>`
and get `E` for free instead of repeating it everywhere. It's a common pattern in crates
and modules with one dominant error type; error-handling crates like `thiserror`
generalize this further (see
[Chapter 25: Third-Party Library Reference](IX.25-third-party-library-reference.md)).

---

## Newtype Pattern

### Creating Distinct Types

```rust
// Newtype wraps an existing type to create a distinct type
struct Meters(f64);
struct Feet(f64);

impl Meters {
    fn to_feet(&self) -> Feet {
        Feet(self.0 * 3.28084)
    }
}

impl Feet {
    fn to_meters(&self) -> Meters {
        Meters(self.0 / 3.28084)
    }
}

fn main() {
    let height = Meters(1.8);
    let height_feet = height.to_feet();

    println!("{} meters = {} feet", height.0, height_feet.0);

    // These are different types - compiler prevents mixing them
    // let bad: Meters = height_feet;  // ERROR
}
```

### Implementing Traits on External Types

```rust
// Can't implement external traits on external types directly
// But newtype allows it!

use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("a"), String::from("b")]);
    println!("{}", w);  // [a, b]
}
```

### Newtype for Type Safety

```rust
struct UserId(u64);
struct PostId(u64);

fn get_user_posts(user_id: UserId) -> Vec<PostId> {
    // Implementation
    vec![PostId(1), PostId(2)]
}

fn main() {
    let user = UserId(42);
    let post = PostId(1);

    let posts = get_user_posts(user);
    // get_user_posts(post);  // ERROR: expected UserId, got PostId
}
```

### Deref for Transparent Newtypes

```rust
use std::ops::Deref;

struct Email(String);

impl Deref for Email {
    type Target = String;

    fn deref(&self) -> &String {
        &self.0
    }
}

fn main() {
    let email = Email(String::from("user@example.com"));

    // Can use String methods directly
    println!("Length: {}", email.len());
    println!("Contains @: {}", email.contains("@"));
}
```

`email.len()` compiles because `Email` doesn't define `len()` itself: when method lookup
fails on `Email`, the compiler follows `Deref` to `String` and retries there. This is the
same deref coercion that lets `&String` work where `&str` is expected. It's convenient,
but implementing `Deref` on a newtype that's meant to enforce an invariant (like `Email`
validating that it contains an `@`) also exposes every `String` method, including ones
that could produce a value the type was meant to disallow. Reach for `Deref` when the
newtype is genuinely a smart pointer to its inner value; expose specific accessor methods
instead when the newtype exists to restrict what you can do with the inner value.

---

## Constants in Types

### Associated Constants

```rust
struct Circle {
    radius: f64,
}

impl Circle {
    // Associated constant
    const PI: f64 = 3.14159265358979;

    fn area(&self) -> f64 {
        Self::PI * self.radius * self.radius
    }

    fn circumference(&self) -> f64 {
        2.0 * Self::PI * self.radius
    }
}

fn main() {
    let c = Circle { radius: 5.0 };
    println!("Area: {}", c.area());
    println!("Circumference: {}", c.circumference());
    println!("PI: {}", Circle::PI);
}
```

### Constants in Enums

```rust
enum ErrorCode {
    NotFound,
    PermissionDenied,
    Internal,
}

impl ErrorCode {
    const NOT_FOUND_CODE: u32 = 404;
    const PERMISSION_DENIED_CODE: u32 = 403;
    const INTERNAL_CODE: u32 = 500;

    fn code(&self) -> u32 {
        match self {
            ErrorCode::NotFound => Self::NOT_FOUND_CODE,
            ErrorCode::PermissionDenied => Self::PERMISSION_DENIED_CODE,
            ErrorCode::Internal => Self::INTERNAL_CODE,
        }
    }
}
```

---

## Generic Types

### Generic Structs

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let int_point = Point { x: 5, y: 10 };
    let float_point = Point { x: 1.5, y: 2.5 };

    // T is inferred from usage
}
```

A single type parameter `T` used for both fields means `x` and `y` must be the *same*
type: `Point { x: 5, y: 1.5 }` won't compile, because that would need `x: i32` and
`y: f64` at once. Each concrete usage (`Point<i32>`, `Point<f64>`, ...) is compiled to
its own monomorphized version of the struct, so there's no runtime cost to the
genericity, but it does mean the two fields can't independently vary in type. `Pair<T, U>`
below shows the fix when they need to.

### Multiple Type Parameters

```rust
struct Pair<T, U> {
    first: T,
    second: U,
}

fn main() {
    let pair = Pair {
        first: 1,
        second: "hello",
    };

    println!("{}, {}", pair.first, pair.second);
}
```

### Generic Enums

```rust
enum MyResult<T, E> {
    Ok(T),
    Err(E),
}

enum MyOption<T> {
    Some(T),
    None,
}
```

### Generic Methods

```rust
struct Container<T> {
    value: T,
}

impl<T> Container<T> {
    fn new(value: T) -> Self {
        Container { value }
    }

    fn get(&self) -> &T {
        &self.value
    }

    fn into_inner(self) -> T {
        self.value
    }
}

// Methods only for specific types
impl Container<String> {
    fn len(&self) -> usize {
        self.value.len()
    }
}
```

The first `impl<T> Container<T>` block applies to every instantiation of `Container`.
The second, `impl Container<String>`, adds `len()` only when `T` is concretely `String`;
a `Container<i32>` has no `len()` method at all. This lets a generic type grow
type-specific methods without needing a separate trait, as long as you're the one
defining both the type and the extra `impl` block.

### Generic Constraints

```rust
use std::fmt::Display;

struct Showable<T: Display> {
    value: T,
}

impl<T: Display> Showable<T> {
    fn show(&self) {
        println!("{}", self.value);
    }
}

// With where clause for complex bounds
struct Complex<T, U>
where
    T: Display + Clone,
    U: Debug,
{
    t: T,
    u: U,
}

use std::fmt::Debug;
```

---

## Methods and Associated Functions

### Defining Methods

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // Method: takes &self
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // Method: takes &mut self
    fn double(&mut self) {
        self.width *= 2;
        self.height *= 2;
    }

    // Method: takes self (consumes)
    fn into_square(self) -> Rectangle {
        let side = std::cmp::min(self.width, self.height);
        Rectangle { width: side, height: side }
    }
}
```

The receiver's form controls what the method can do and what happens to the caller's
value afterward. `&self` borrows immutably, so the caller keeps using the value
afterward but the method can only read it. `&mut self` borrows mutably, requiring the
caller's binding to be `mut`, and lets the method modify fields in place. Plain `self`
takes ownership, moving the value into the method; `into_square` can only be called on a
`Rectangle` the caller is willing to give up, and the original binding is no longer valid
afterward, exactly like moving a value into any other function.

### Associated Functions

```rust
impl Rectangle {
    // Associated function (no self) - like a static method
    fn new(width: u32, height: u32) -> Rectangle {
        Rectangle { width, height }
    }

    fn square(size: u32) -> Rectangle {
        Rectangle { width: size, height: size }
    }
}

fn main() {
    // Call with ::
    let rect = Rectangle::new(10, 20);
    let square = Rectangle::square(5);
}
```

### Multiple impl Blocks

```rust
struct MyStruct {
    value: i32,
}

impl MyStruct {
    fn new(value: i32) -> Self {
        MyStruct { value }
    }
}

impl MyStruct {
    fn get(&self) -> i32 {
        self.value
    }
}

// Useful for organizing code or conditional compilation
#[cfg(test)]
impl MyStruct {
    fn test_helper(&self) -> bool {
        self.value > 0
    }
}
```

### Builder Pattern

```rust
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
}

struct ServerBuilder {
    host: String,
    port: u16,
    max_connections: u32,
}

impl ServerBuilder {
    fn new() -> Self {
        ServerBuilder {
            host: String::from("localhost"),
            port: 8080,
            max_connections: 100,
        }
    }

    fn host(mut self, host: &str) -> Self {
        self.host = host.to_string();
        self
    }

    fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn max_connections(mut self, max: u32) -> Self {
        self.max_connections = max;
        self
    }

    fn build(self) -> Server {
        Server {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
        }
    }
}

fn main() {
    let server = ServerBuilder::new()
        .host("0.0.0.0")
        .port(3000)
        .max_connections(1000)
        .build();

    println!("{:?}", server);
}
```

Each builder method takes `mut self` by value, mutates the field, and returns `self`, so
calls can be chained: `.host(...)` consumes the builder produced by `new()` and hands
back an owned, updated builder for `.port(...)` to consume in turn. Taking `self` rather
than `&mut self` is what makes the chain a single expression instead of needing a
separate `let mut builder = ...;` statement before each call.

---

## Quick Reference

| Type | Use case |
|------|----------|
| Struct | Group related data with named fields |
| Tuple Struct | Lightweight wrapper with positional fields |
| Unit Struct | Marker types, trait implementations |
| Enum | Type with multiple variants |
| Type Alias | Simplify complex types |
| Newtype | Create distinct types from existing ones |

---

## Common Patterns

This chapter's Methods and Associated Functions section already walks through the
builder pattern in full (a `ServerBuilder` that sets fields with chained methods and
finishes with `.build()`). Reach for it whenever a type has several optional fields and
you want to avoid a constructor with a long parameter list.

The newtype pattern, covered above, is also a pattern in its own right: wrap an existing
type (`struct UserId(u64)`) to get a distinct type the compiler won't let you mix up with
another wrapper around the same underlying type (`PostId(u64)`), or to implement an
external trait on an external type.

---

## Common Pitfalls

### Struct update syntax moves non-Copy fields

`..user1` in struct update syntax copies `Copy` fields but moves everything else. If any
of the fields you didn't override are not `Copy` (a `String`, for instance), `user1` is
partially moved and can no longer be used as a whole:

```rust
let user2 = User {
    email: String::from("bob@example.com"),
    username: String::from("bob"),
    ..user1  // sign_in_count and active are copied; username/email would move if reused
};
```

### A struct can't derive Copy if any field isn't Copy

```rust
// #[derive(Copy, Clone)]
// struct BadPoint {
//     x: i32,
//     name: String, // ERROR: String doesn't implement Copy
// }
```

---

## Summary

Struct, tuple struct, unit struct, enum, type alias, and newtype cover most custom-type
needs; see the Quick Reference above for when to reach for each.

### Best Practices

1. Use structs for grouping related data
2. Use enums for types with distinct variants
3. Derive common traits when appropriate. See
   [Chapter 15: Common Derivable Traits](IV.15-common-derivable-traits-quick-reference.md)
4. Use newtype for type safety
5. Keep fields private by default. See
   [Chapter 4: Modules, Crates & Visibility](I.4-modules-crates-visibility.md)
6. Provide constructors via associated functions
7. Use the builder pattern for complex initialization
