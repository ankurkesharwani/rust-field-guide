# Option & Result Types in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Result Definition](#result-definition)
3. [Creating Results](#creating-results)
4. [Handling Results](#handling-results)
5. [The ? Operator](#the--operator)
6. [Result Methods](#result-methods)
7. [Transforming Results](#transforming-results)
8. [Combining Results](#combining-results)
9. [Option Methods](#option-methods)
10. [Custom Error Types](#custom-error-types)
11. [Quick Reference](#quick-reference)
12. [Common Patterns](#common-patterns)
13. [Result vs Option](#result-vs-option)
14. [Common Pitfalls](#common-pitfalls)
15. [Summary](#summary)

---

## Introduction

`Result` is Rust's primary mechanism for handling recoverable errors, and `Option`
handles the related case of a value that may or may not be present. Unlike exceptions in
other languages, both types force you to acknowledge and handle the absent or failing
case at compile time, leading to more robust code. This chapter is the canonical home
for both types, including the custom error types built on top of `Result`.

Key characteristics:
- Explicit: absence and errors are part of the function signature
- Composable: both types can be chained and transformed with the same combinator style
- Zero-cost: no runtime overhead compared to null checks or return codes
- Type-safe: the compiler ensures every case is handled

---

## Result Definition

`Result` is an enum defined in the standard library:

```rust
enum Result<T, E> {
    Ok(T),    // Contains the success value of type T
    Err(E),   // Contains the error value of type E
}
```

### Type Parameters

- `T`: The type of the value returned on success
- `E`: The type of the error returned on failure

Both are generic, so each fallible operation picks its own error type instead of
sharing one exception hierarchy. `File::open` fails with `io::Error`, `"x".parse()`
fails with a `ParseIntError` or similar, and the two are not interchangeable unless you
convert between them. The function signature tells you exactly what can go wrong.

```rust
use std::fs::File;
use std::io::Error;

fn main() {
    // Result<File, Error> - success returns File, error returns io::Error
    let file_result: Result<File, Error> = File::open("hello.txt");

    // Result<i32, &str> - success returns i32, error returns &str
    let parse_result: Result<i32, &str> = Ok(42);
}
```

---

## Creating Results

### Using Ok and Err

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Cannot divide by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    let result1 = divide(10.0, 2.0);  // Ok(5.0)
    let result2 = divide(10.0, 0.0);  // Err("Cannot divide by zero")

    println!("{:?}", result1);
    println!("{:?}", result2);
}
```

### Functions That Return Result

```rust
use std::num::ParseIntError;

fn parse_number(s: &str) -> Result<i32, ParseIntError> {
    s.parse::<i32>()  // parse() already returns Result
}

fn validate_age(age: i32) -> Result<i32, &'static str> {
    if age < 0 {
        Err("Age cannot be negative")
    } else if age > 150 {
        Err("Age is unrealistically high")
    } else {
        Ok(age)
    }
}

fn main() {
    println!("{:?}", parse_number("42"));      // Ok(42)
    println!("{:?}", parse_number("not"));     // Err(ParseIntError)
    println!("{:?}", validate_age(25));        // Ok(25)
    println!("{:?}", validate_age(-5));        // Err("Age cannot be negative")
}
```

### Standard Library Functions Returning Result

I/O, parsing, and environment access all reach outside the program, where failure is
routine (a missing file, malformed input, an unset variable), so the standard library
returns `Result` from these functions instead of panicking. Because the convention is
consistent, the same `?` operator and combinator methods work regardless of which
corner of the standard library you're calling into.

```rust
use std::fs::{self, File};
use std::io::{self, Read, Write};
use std::net::TcpStream;

fn demonstrate_results() {
    // File operations
    let file: Result<File, io::Error> = File::open("test.txt");
    let contents: Result<String, io::Error> = fs::read_to_string("test.txt");

    // Parsing
    let number: Result<i32, std::num::ParseIntError> = "42".parse();
    let float: Result<f64, std::num::ParseFloatError> = "3.14".parse();

    // Network operations
    let stream: Result<TcpStream, io::Error> = TcpStream::connect("127.0.0.1:8080");

    // Environment
    let var: Result<String, std::env::VarError> = std::env::var("HOME");
}
```

---

## Handling Results

### Using match

The most explicit way to handle Results:

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_file(path: &str) -> Result<String, io::Error> {
    let file = File::open(path);

    let mut file = match file {
        Ok(f) => f,
        Err(e) => return Err(e),
    };

    let mut contents = String::new();

    match file.read_to_string(&mut contents) {
        Ok(_) => Ok(contents),
        Err(e) => Err(e),
    }
}

fn main() {
    match read_file("hello.txt") {
        Ok(contents) => println!("File contents:\n{}", contents),
        Err(error) => println!("Error reading file: {}", error),
    }
}
```

### Using if let

When you only care about one case:

```rust
use std::fs::File;

fn main() {
    // Only handle success
    if let Ok(file) = File::open("hello.txt") {
        println!("File opened: {:?}", file);
    }

    // Only handle error
    if let Err(e) = File::open("nonexistent.txt") {
        eprintln!("Failed to open file: {}", e);
    }
}
```

### Using unwrap and expect

For quick prototyping or when failure should panic:

```rust
fn main() {
    // unwrap: panics with generic message on Err
    let contents = std::fs::read_to_string("config.txt").unwrap();

    // expect: panics with custom message on Err
    let file = std::fs::File::open("config.txt")
        .expect("config.txt should exist in the project root");

    // unwrap_err: panics on Ok, returns error on Err
    let error = "not a number".parse::<i32>().unwrap_err();
    println!("Got error: {}", error);
}
```

**Warning**: Avoid `unwrap()` and `expect()` in production code for recoverable errors. Use them for:
- Prototyping
- Tests
- When you can prove the Result is always Ok

### Using unwrap_or and unwrap_or_else

Provide default values:

```rust
fn main() {
    // unwrap_or: provide default value
    let count = "invalid".parse::<i32>().unwrap_or(0);
    println!("Count: {}", count);  // 0

    // unwrap_or_else: compute default lazily
    let count = "invalid".parse::<i32>().unwrap_or_else(|e| {
        eprintln!("Parse error: {}, using default", e);
        0
    });

    // unwrap_or_default: use Default trait
    let count: i32 = "invalid".parse().unwrap_or_default();  // 0
}
```

---

## The ? Operator

The `?` operator provides concise error propagation:

```rust
use std::fs::File;
use std::io::{self, Read};

// Without ? operator
fn read_file_verbose(path: &str) -> Result<String, io::Error> {
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

// With ? operator
fn read_file_concise(path: &str) -> Result<String, io::Error> {
    let mut file = File::open(path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}

// Even more concise
fn read_file_chain(path: &str) -> Result<String, io::Error> {
    let mut contents = String::new();
    File::open(path)?.read_to_string(&mut contents)?;
    Ok(contents)
}

fn main() {
    match read_file_concise("test.txt") {
        Ok(contents) => println!("{}", contents),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

### How ? Works

The `?` operator:
1. On `Ok(value)`: Unwraps and returns `value`
2. On `Err(e)`: Returns early with `Err(e)` from the function

```rust
// This:
let value = some_result?;

// Is roughly equivalent to:
let value = match some_result {
    Ok(v) => v,
    Err(e) => return Err(e.into()),  // Note: calls .into() for error conversion
};
```

### ? with Error Conversion

The `?` operator automatically converts errors using the `From` trait:

```rust
use std::fs::File;
use std::io::{self, Read};
use std::num::ParseIntError;

#[derive(Debug)]
enum MyError {
    Io(io::Error),
    Parse(ParseIntError),
}

impl From<io::Error> for MyError {
    fn from(error: io::Error) -> Self {
        MyError::Io(error)
    }
}

impl From<ParseIntError> for MyError {
    fn from(error: ParseIntError) -> Self {
        MyError::Parse(error)
    }
}

fn read_and_parse(path: &str) -> Result<i32, MyError> {
    let mut contents = String::new();
    File::open(path)?.read_to_string(&mut contents)?;  // io::Error -> MyError
    let number = contents.trim().parse()?;              // ParseIntError -> MyError
    Ok(number)
}
```

### ? in main()

You can use `?` in main by returning Result:

```rust
use std::fs::File;
use std::io::{self, Read};

fn main() -> Result<(), io::Error> {
    let mut file = File::open("hello.txt")?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    println!("{}", contents);
    Ok(())
}
```

---

## Result Methods

### Checking State

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("error");

    // is_ok() and is_err()
    println!("ok.is_ok(): {}", ok.is_ok());     // true
    println!("ok.is_err(): {}", ok.is_err());   // false
    println!("err.is_ok(): {}", err.is_ok());   // false
    println!("err.is_err(): {}", err.is_err()); // true

    // is_ok_and() and is_err_and() (Rust 1.70+)
    let ok: Result<i32, &str> = Ok(42);
    println!("{}", ok.is_ok_and(|x| x > 40));   // true
    println!("{}", ok.is_ok_and(|x| x > 50));   // false
}
```

### Getting Values

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("error");

    // ok() - converts to Option<T>
    assert_eq!(ok.ok(), Some(42));
    assert_eq!(err.ok(), None);

    // err() - converts to Option<E>
    assert_eq!(ok.err(), None);
    assert_eq!(err.err(), Some("error"));

    // as_ref() - Result<&T, &E>
    let ok_ref: Result<&i32, &&str> = ok.as_ref();

    // as_mut() - Result<&mut T, &mut E>
    let mut ok = Ok(42);
    if let Ok(ref mut value) = ok.as_mut() {
        *value += 1;
    }
    assert_eq!(ok, Ok(43));
}
```

### Extracting Values Safely

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("error");

    // unwrap_or
    assert_eq!(ok.unwrap_or(0), 42);
    assert_eq!(err.unwrap_or(0), 0);

    // unwrap_or_else
    assert_eq!(ok.unwrap_or_else(|_| 0), 42);
    assert_eq!(err.unwrap_or_else(|e| e.len() as i32), 5);

    // unwrap_or_default
    let err: Result<i32, &str> = Err("error");
    assert_eq!(err.unwrap_or_default(), 0);

    // expect (panics with message on Err)
    let ok: Result<i32, &str> = Ok(42);
    assert_eq!(ok.expect("should have a value"), 42);
}
```

---

## Transforming Results

### map - Transform Success Value

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(5);
    let err: Result<i32, &str> = Err("error");

    // Transform the Ok value
    let doubled = ok.map(|x| x * 2);
    assert_eq!(doubled, Ok(10));

    // Err passes through unchanged
    let doubled = err.map(|x| x * 2);
    assert_eq!(doubled, Err("error"));

    // Change the type
    let stringified = ok.map(|x| x.to_string());
    assert_eq!(stringified, Ok(String::from("5")));
}
```

### map_err - Transform Error Value

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(5);
    let err: Result<i32, &str> = Err("error");

    // Transform the Err value
    let err_len = err.map_err(|e| e.len());
    assert_eq!(err_len, Err(5));

    // Ok passes through unchanged
    let ok_mapped = ok.map_err(|e| e.len());
    assert_eq!(ok_mapped, Ok(5));

    // Convert error types
    let err: Result<i32, &str> = Err("not found");
    let err_string: Result<i32, String> = err.map_err(|e| e.to_uppercase());
    assert_eq!(err_string, Err(String::from("NOT FOUND")));
}
```

### map_or and map_or_else

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(5);
    let err: Result<i32, &str> = Err("error");

    // map_or: provide default if Err
    assert_eq!(ok.map_or(0, |x| x * 2), 10);
    assert_eq!(err.map_or(0, |x| x * 2), 0);

    // map_or_else: compute default lazily
    assert_eq!(
        ok.map_or_else(|e| e.len() as i32, |x| x * 2),
        10
    );
    assert_eq!(
        err.map_or_else(|e| e.len() as i32, |x| x * 2),
        5
    );
}
```

### and_then - Chain Fallible Operations

```rust
fn square(x: i32) -> Result<i32, &'static str> {
    if x > 1000 {
        Err("number too large")
    } else {
        Ok(x * x)
    }
}

fn main() {
    let ok: Result<i32, &str> = Ok(5);
    let err: Result<i32, &str> = Err("error");

    // Chain operations that might fail
    let result = ok.and_then(square);
    assert_eq!(result, Ok(25));

    // Err short-circuits
    let result = err.and_then(square);
    assert_eq!(result, Err("error"));

    // Chain multiple operations
    let result = Ok(10)
        .and_then(square)   // Ok(100)
        .and_then(square);  // Ok(10000)
    assert_eq!(result, Ok(10000));

    // Error in chain
    let result = Ok(100)
        .and_then(square)   // Ok(10000)
        .and_then(square);  // Err("number too large")
    assert_eq!(result, Err("number too large"));
}
```

### or_else - Provide Alternative on Error

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(5);
    let err: Result<i32, &str> = Err("error");

    // Try alternative on error
    let result = err.or_else(|_| Ok(0));
    assert_eq!(result, Ok(0));

    // Ok passes through
    let result = ok.or_else(|_| Ok(0));
    assert_eq!(result, Ok(5));

    // Chain alternatives
    fn try_parse(s: &str) -> Result<i32, &'static str> {
        s.parse().map_err(|_| "parse error")
    }

    let result = try_parse("abc")
        .or_else(|_| try_parse("123"))
        .or_else(|_| try_parse("456"));
    assert_eq!(result, Ok(123));
}
```

### inspect and inspect_err

Execute a side effect without consuming:

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(5);

    // inspect: peek at Ok value (Rust 1.76+)
    let _ = ok.inspect(|x| println!("Got value: {}", x));

    // inspect_err: peek at Err value
    let err: Result<i32, &str> = Err("error");
    let _ = err.inspect_err(|e| eprintln!("Error: {}", e));
}
```

---

## Combining Results

### and and or

```rust
fn main() {
    let ok1: Result<i32, &str> = Ok(1);
    let ok2: Result<i32, &str> = Ok(2);
    let err: Result<i32, &str> = Err("error");

    // and: return second if first is Ok, else return first's Err
    assert_eq!(ok1.and(ok2), Ok(2));
    assert_eq!(err.and(ok1), Err("error"));

    // or: return first if Ok, else return second
    assert_eq!(ok1.or(ok2), Ok(1));
    assert_eq!(err.or(ok1), Ok(1));
}
```

### Collecting Results

Convert `Vec<Result<T, E>>` to `Result<Vec<T>, E>`:

```rust
fn main() {
    // All Ok -> Ok(Vec)
    let results: Vec<Result<i32, &str>> = vec![Ok(1), Ok(2), Ok(3)];
    let collected: Result<Vec<i32>, &str> = results.into_iter().collect();
    assert_eq!(collected, Ok(vec![1, 2, 3]));

    // Any Err -> first Err
    let results: Vec<Result<i32, &str>> = vec![Ok(1), Err("error"), Ok(3)];
    let collected: Result<Vec<i32>, &str> = results.into_iter().collect();
    assert_eq!(collected, Err("error"));
}
```

### Using partition

Separate successes and failures:

```rust
fn main() {
    let results = vec![Ok(1), Err("a"), Ok(2), Err("b"), Ok(3)];

    let (successes, failures): (Vec<_>, Vec<_>) = results
        .into_iter()
        .partition(|r| r.is_ok());

    let successes: Vec<i32> = successes.into_iter().map(|r| r.unwrap()).collect();
    let failures: Vec<&str> = failures.into_iter().map(|r| r.unwrap_err()).collect();

    println!("Successes: {:?}", successes);  // [1, 2, 3]
    println!("Failures: {:?}", failures);    // ["a", "b"]
}
```

### Transposing Option and Result

An `Option<Result<T, E>>` shows up when a value is optional but, once present, parsing
or validating it can fail (an optional config field that must be a valid number if
it's set at all). `transpose` swaps it to `Result<Option<T>, E>` so you can use `?` to
propagate the error and keep the `Option` wrapping the success value, rather than
matching through both layers by hand.

```rust
fn main() {
    // Option<Result<T, E>> -> Result<Option<T>, E>
    let opt_result: Option<Result<i32, &str>> = Some(Ok(5));
    let result_opt: Result<Option<i32>, &str> = opt_result.transpose();
    assert_eq!(result_opt, Ok(Some(5)));

    // Result<Option<T>, E> -> Option<Result<T, E>>
    let result_opt: Result<Option<i32>, &str> = Ok(Some(5));
    let opt_result: Option<Result<i32, &str>> = result_opt.transpose();
    assert_eq!(opt_result, Some(Ok(5)));
}
```

---

## Option Methods

Higher-order methods on Option:

### map and map_or

```rust
fn main() {
    let some_number = Some(5);
    let none: Option<i32> = None;

    // map: transform if Some
    let doubled = some_number.map(|x| x * 2);
    println!("{:?}", doubled); // Some(10)

    let doubled = none.map(|x| x * 2);
    println!("{:?}", doubled); // None

    // map_or: provide default
    let value = some_number.map_or(0, |x| x * 2);
    println!("{}", value); // 10

    let value = none.map_or(0, |x| x * 2);
    println!("{}", value); // 0

    // map_or_else: compute default lazily
    let value = none.map_or_else(|| expensive_default(), |x| x * 2);
}

fn expensive_default() -> i32 {
    println!("Computing default...");
    42
}
```

### and_then (flatMap)

`map` applies a plain function to the contained value, so if that function itself
returns an `Option`, `map` would leave you with `Option<Option<T>>`. `and_then` expects
its closure to return an `Option` and merges the two layers into one, which is what you
want when chaining several steps that can each fail to produce a value.

```rust
fn main() {
    fn square_if_positive(x: i32) -> Option<i32> {
        if x > 0 { Some(x * x) } else { None }
    }

    let some = Some(5);
    let result = some.and_then(square_if_positive);
    println!("{:?}", result); // Some(25)

    let some = Some(-5);
    let result = some.and_then(square_if_positive);
    println!("{:?}", result); // None

    // Chain multiple
    let result = Some(5)
        .and_then(|x| Some(x * 2))
        .and_then(|x| Some(x + 1));
    println!("{:?}", result); // Some(11)
}
```

### filter

```rust
fn main() {
    let some = Some(4);

    let result = some.filter(|x| x % 2 == 0);
    println!("{:?}", result); // Some(4)

    let result = some.filter(|x| *x > 10);
    println!("{:?}", result); // None
}
```

### or_else and or

```rust
fn main() {
    let none: Option<i32> = None;
    let some = Some(5);

    // or: provide alternative Option
    let result = none.or(Some(10));
    println!("{:?}", result); // Some(10)

    let result = some.or(Some(10));
    println!("{:?}", result); // Some(5)

    // or_else: compute alternative lazily
    let result = none.or_else(|| Some(compute_default()));
    println!("{:?}", result); // Some(42)
}

fn compute_default() -> i32 {
    42
}
```

### get_or_insert and get_or_insert_with

```rust
fn main() {
    let mut opt = None;

    // Insert if None, return mutable reference
    let value = opt.get_or_insert(10);
    println!("{}", value); // 10

    *value += 1;
    println!("{:?}", opt); // Some(11)

    // Lazy insertion
    let mut opt: Option<String> = None;
    let value = opt.get_or_insert_with(|| String::from("default"));
    println!("{}", value); // "default"
}
```

---

## Custom Error Types

### Simple String Errors

A `String` error is the easiest way to fail with a message, but the caller only gets
text: they can print it, but they can't match on what actually went wrong without
parsing the message back apart. It's a reasonable starting point for a small function,
and a poor fit once callers need to react differently to different failures.

```rust
fn validate_username(name: &str) -> Result<&str, String> {
    if name.is_empty() {
        Err(String::from("Username cannot be empty"))
    } else if name.len() < 3 {
        Err(format!("Username '{}' is too short (min 3 chars)", name))
    } else {
        Ok(name)
    }
}
```

### Enum Error Types

An enum error type fixes that: each failure becomes its own variant, so callers can
`match` on exactly which case occurred and pull out whatever structured data came with
it (like `min`/`actual` below), instead of re-parsing a string.

```rust
#[derive(Debug)]
enum ValidationError {
    Empty,
    TooShort { min: usize, actual: usize },
    TooLong { max: usize, actual: usize },
    InvalidCharacter(char),
}

fn validate_username(name: &str) -> Result<&str, ValidationError> {
    if name.is_empty() {
        return Err(ValidationError::Empty);
    }

    if name.len() < 3 {
        return Err(ValidationError::TooShort {
            min: 3,
            actual: name.len(),
        });
    }

    if name.len() > 20 {
        return Err(ValidationError::TooLong {
            max: 20,
            actual: name.len(),
        });
    }

    for c in name.chars() {
        if !c.is_alphanumeric() && c != '_' {
            return Err(ValidationError::InvalidCharacter(c));
        }
    }

    Ok(name)
}

fn main() {
    match validate_username("ab") {
        Ok(name) => println!("Valid: {}", name),
        Err(ValidationError::Empty) => println!("Username is empty"),
        Err(ValidationError::TooShort { min, actual }) => {
            println!("Too short: need {} chars, got {}", min, actual)
        }
        Err(ValidationError::TooLong { max, actual }) => {
            println!("Too long: max {} chars, got {}", max, actual)
        }
        Err(ValidationError::InvalidCharacter(c)) => {
            println!("Invalid character: '{}'", c)
        }
    }
}
```

### Implementing std::error::Error

`std::error::Error` is what lets an error type interoperate with the rest of the
ecosystem: code that only knows about `Box<dyn Error>`, logging that calls
`error.to_string()`, or a caller that walks `error.source()` (see the next section) can
all accept your type without knowing anything about it. The trait requires `Debug` and
`Display` as supertraits, which is why both are implemented here even though `Error`
itself adds no new methods you have to write.

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct MyError {
    message: String,
}

impl MyError {
    fn new(msg: &str) -> MyError {
        MyError {
            message: msg.to_string(),
        }
    }
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.message)
    }
}

impl Error for MyError {}

fn do_something() -> Result<(), MyError> {
    Err(MyError::new("Something went wrong"))
}

fn main() {
    if let Err(e) = do_something() {
        println!("Error: {}", e);  // Uses Display
        println!("Debug: {:?}", e); // Uses Debug
    }
}
```

### Wrapping Other Errors

`source()` links an error to the lower-level error that caused it, so something
printing the error can walk the chain and show the whole story instead of just the
outermost message. The `From` implementations below plug into that same `?` conversion
covered earlier: once `From<io::Error>` and `From<ParseIntError>` exist for
`ConfigError`, `?` calls them automatically wherever those errors surface.

```rust
use std::error::Error;
use std::fmt;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    Io(io::Error),
    Parse(ParseIntError),
    Missing(String),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Io(e) => write!(f, "IO error: {}", e),
            ConfigError::Parse(e) => write!(f, "Parse error: {}", e),
            ConfigError::Missing(key) => write!(f, "Missing config key: {}", key),
        }
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::Io(e) => Some(e),
            ConfigError::Parse(e) => Some(e),
            ConfigError::Missing(_) => None,
        }
    }
}

impl From<io::Error> for ConfigError {
    fn from(error: io::Error) -> Self {
        ConfigError::Io(error)
    }
}

impl From<ParseIntError> for ConfigError {
    fn from(error: ParseIntError) -> Self {
        ConfigError::Parse(error)
    }
}

// Now ? works with automatic conversion
fn load_config(path: &str) -> Result<i32, ConfigError> {
    let contents = std::fs::read_to_string(path)?;  // io::Error -> ConfigError
    let number: i32 = contents.trim().parse()?;      // ParseIntError -> ConfigError
    Ok(number)
}
```

---

## Quick Reference

### Result methods

| Method | Purpose |
|--------|---------|
| `is_ok()`, `is_err()` | Check state |
| `unwrap()`, `expect()` | Extract or panic |
| `unwrap_or()`, `unwrap_or_else()` | Extract with default |
| `map()`, `map_err()` | Transform values |
| `and_then()`, `or_else()` | Chain operations |
| `and()`, `or()` | Combine with another `Result`, eagerly |
| `?` operator | Propagate errors |
| `ok()`, `err()` | Convert to `Option` |

### Option methods

| Method | Purpose |
|--------|---------|
| `map()`, `map_or()`, `map_or_else()` | Transform the contained value |
| `and_then()` | Chain an operation that itself returns `Option` |
| `filter()` | Keep the value only if a predicate holds |
| `or()`, `or_else()` | Provide a fallback `Option` |
| `get_or_insert()`, `get_or_insert_with()` | Insert a value if `None`, then return a mutable reference |
| `ok_or()`, `ok_or_else()` | Convert to `Result` |

---

## Common Patterns

### The try Pattern (Before ?)

Historical context - the `?` operator replaced this pattern:

```rust
// Old try! macro (deprecated)
// let file = try!(File::open("file.txt"));

// Modern equivalent
// let file = File::open("file.txt")?;
```

### Early Return Pattern

```rust
fn process_data(input: &str) -> Result<i32, &'static str> {
    if input.is_empty() {
        return Err("input is empty");
    }

    let trimmed = input.trim();
    if trimmed.is_empty() {
        return Err("input is whitespace only");
    }

    let number: i32 = trimmed.parse().map_err(|_| "invalid number")?;

    if number < 0 {
        return Err("number must be non-negative");
    }

    Ok(number * 2)
}
```

### Builder Pattern with Results

```rust
#[derive(Debug)]
struct Config {
    host: String,
    port: u16,
    timeout: u32,
}

struct ConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    timeout: Option<u32>,
}

impl ConfigBuilder {
    fn new() -> Self {
        ConfigBuilder {
            host: None,
            port: None,
            timeout: None,
        }
    }

    fn host(mut self, host: &str) -> Self {
        self.host = Some(host.to_string());
        self
    }

    fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }

    fn timeout(mut self, timeout: u32) -> Self {
        self.timeout = Some(timeout);
        self
    }

    fn build(self) -> Result<Config, &'static str> {
        Ok(Config {
            host: self.host.ok_or("host is required")?,
            port: self.port.ok_or("port is required")?,
            timeout: self.timeout.unwrap_or(30),
        })
    }
}

fn main() {
    let config = ConfigBuilder::new()
        .host("localhost")
        .port(8080)
        .build();

    println!("{:?}", config);

    let bad_config = ConfigBuilder::new()
        .timeout(60)
        .build();

    println!("{:?}", bad_config);  // Err("host is required")
}
```

### Fallback Chain

```rust
fn get_config_value(key: &str) -> Result<String, &'static str> {
    // Try environment variable first
    std::env::var(key)
        .map_err(|_| "not in env")
        // Then try config file
        .or_else(|_| read_from_config(key))
        // Then try default
        .or_else(|_| get_default(key))
}

fn read_from_config(_key: &str) -> Result<String, &'static str> {
    Err("not in config")
}

fn get_default(key: &str) -> Result<String, &'static str> {
    match key {
        "timeout" => Ok("30".to_string()),
        "retries" => Ok("3".to_string()),
        _ => Err("no default available"),
    }
}
```

---

## Result vs Option

### When to Use Each

| Use `Option<T>` | Use `Result<T, E>` |
|-----------------|-------------------|
| Value may or may not exist | Operation can succeed or fail |
| Absence is not an error | Failure needs explanation |
| Finding something | Doing something |
| No error information needed | Error details matter |

### Converting Between Them

```rust
fn main() {
    // Option -> Result
    let opt: Option<i32> = Some(5);
    let result: Result<i32, &str> = opt.ok_or("missing value");

    let opt: Option<i32> = None;
    let result: Result<i32, String> = opt.ok_or_else(|| "computed error".to_string());

    // Result -> Option
    let result: Result<i32, &str> = Ok(5);
    let opt: Option<i32> = result.ok();

    let result: Result<i32, &str> = Err("error");
    let opt: Option<&str> = result.err();
}
```

### Practical Example

```rust
// Option: Looking up a value
fn find_user(id: u32) -> Option<User> {
    // User might not exist - that's not an error
    users.get(&id).cloned()
}

// Result: Creating a user
fn create_user(name: &str) -> Result<User, ValidationError> {
    // Creation can fail for various reasons
    validate_name(name)?;
    let user = User::new(name);
    save_to_database(&user)?;
    Ok(user)
}
```

---

## Common Pitfalls

- Reaching for `unwrap()` or `expect()` outside prototypes, tests, or a case you can
  prove is always `Ok`/`Some` turns a recoverable failure into a panic. See Using unwrap
  and expect above.
- `and()`/`or()` take an already-built `Result`/`Option` as their argument, so that value
  is computed eagerly even when it turns out not to be needed; `and_then()`/`or_else()`
  take a closure instead, so the alternative is only computed when it's actually used.

---

## Summary

The Result and Option method tables, and the comparison of when to use each type, are
collected in the Quick Reference and Result vs Option sections above.

Key principles:
1. Explicit handling: a `Result` or `Option` must be used or explicitly ignored.
2. Propagation with `?` makes bubbling an error up the call stack easy.
3. Both types share a rich set of combinators for transforming their contained value.
4. The compiler ensures every case is handled, at zero runtime cost compared to manual
   error codes or null checks.
