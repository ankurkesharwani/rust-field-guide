# Error Handling in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Unrecoverable Errors: panic!](#unrecoverable-errors-panic)
3. [Recoverable Errors: Result](#recoverable-errors-result)
4. [Error Propagation](#error-propagation)
5. [Error Chain](#error-chain)
6. [Error Handling Crates](#error-handling-crates)
7. [Quick Reference](#quick-reference)
8. [Common Patterns](#common-patterns)
   - [Application Entry Point Patterns](#application-entry-point-patterns)
9. [When to Panic vs Return Error](#when-to-panic-vs-return-error)
10. [Testing Error Conditions](#testing-error-conditions)
11. [Common Pitfalls](#common-pitfalls)
12. [Summary](#summary)

Custom error types, the `std::error::Error` trait, and the mechanics of the `?` operator
now live in [Chapter 19: Option & Result Types](VII.19-option-and-result-types.md). This
chapter focuses on the surrounding strategy instead: panic vs. Result, `thiserror` vs.
`anyhow`, and higher-level patterns for structuring error handling across an application.

---

## Introduction

Rust handles errors differently than most languages. Instead of exceptions, Rust uses two mechanisms:

1. **Unrecoverable errors**: `panic!` macro - for bugs and programming errors
2. **Recoverable errors**: `Result<T, E>` type - for expected failures

This approach provides:
- **Compile-time safety**: Errors must be handled explicitly
- **No hidden control flow**: Errors are values, not exceptions
- **Zero-cost abstraction**: No runtime overhead for error handling
- **Self-documenting code**: Function signatures show what can fail

---

## Unrecoverable Errors: panic!

### Basic Usage

```rust
fn main() {
    // Explicit panic
    panic!("crash and burn");
}
```

### Panic with Formatting

```rust
fn main() {
    let file_name = "config.txt";
    let line_number = 42;

    panic!(
        "Failed to process {} at line {}",
        file_name,
        line_number
    );
}
```

### When Panics Occur

```rust
fn main() {
    // Array out of bounds
    let v = vec![1, 2, 3];
    // v[99]; // panic: index out of bounds

    // Integer overflow in debug mode
    // let x: u8 = 255 + 1; // panic in debug, wraps in release

    // Unwrap on None
    let opt: Option<i32> = None;
    // opt.unwrap(); // panic: called `Option::unwrap()` on a `None` value

    // Unwrap on Err
    let result: Result<i32, &str> = Err("error");
    // result.unwrap(); // panic: called `Result::unwrap()` on an `Err` value
}
```

### Panic Behavior

By default, panic:
1. Prints an error message
2. Unwinds the stack (cleaning up resources)
3. Exits the program

### Controlling Panic Behavior

The default `unwind` strategy walks back up the call stack running destructors, which
lets a caller catch the panic with `std::panic::catch_unwind` and lets long-running
programs (like a server that panics inside one request) keep going. `abort` skips all
of that and ends the process immediately, trading recoverability for a smaller binary
and slightly faster panics, which is a reasonable trade for a CLI or embedded target
that would exit anyway.

In `Cargo.toml`:

```toml
[profile.release]
panic = 'abort'  # Don't unwind, just abort (smaller binary)
```

### Getting Backtraces

```bash
RUST_BACKTRACE=1 cargo run
RUST_BACKTRACE=full cargo run  # Full backtrace with all frames
```

### unreachable! and todo!

These three panic under different circumstances, and choosing the right one documents
your intent to both the compiler and the next reader: `unreachable!` marks a branch you
have reasoned can never execute (useful after a `match` the compiler can't otherwise
prove is exhaustive), `todo!` marks a function you haven't written yet, and
`unimplemented!` marks a case you've deliberately decided not to support. All three
compile fine and only panic if actually reached, so a `todo!` placeholder won't stop the
rest of the program from building and running.

```rust
fn process(value: i32) -> i32 {
    match value {
        0 => 0,
        1 => 1,
        _ if value > 0 => value * 2,
        _ => unreachable!("Negative numbers should be filtered before this"),
    }
}

fn future_feature() -> String {
    todo!("Implement this feature later")
}

fn unimplemented_variant(option: u8) -> &'static str {
    match option {
        1 => "one",
        2 => "two",
        _ => unimplemented!("Handle other options"),
    }
}
```

### assert! Macros

```rust
fn main() {
    let x = 5;

    // Basic assertion
    assert!(x > 0);

    // With custom message
    assert!(x > 0, "x must be positive, got {}", x);

    // Equality assertions
    assert_eq!(x, 5);
    assert_ne!(x, 0);

    // Debug assertions (only in debug builds)
    debug_assert!(x > 0);
    debug_assert_eq!(x, 5);
}
```

---

## Recoverable Errors: Result

### Basic Result Usage

```rust
use std::fs::File;
use std::io::Read;

fn main() {
    let file_result = File::open("hello.txt");

    let mut file = match file_result {
        Ok(file) => file,
        Err(error) => {
            eprintln!("Error opening file: {}", error);
            return;
        }
    };

    let mut contents = String::new();
    match file.read_to_string(&mut contents) {
        Ok(bytes) => println!("Read {} bytes: {}", bytes, contents),
        Err(error) => eprintln!("Error reading file: {}", error),
    }
}
```

### Handling Different Error Types

A plain `match` on `Result` only tells you that `File::open` failed, not why. `io::Error`
carries an `ErrorKind` (an enum covering cases like "not found" or "permission denied")
behind its `.kind()` method, so you can match on the specific failure and, for example,
create the file only when it's genuinely missing rather than for every kind of error.

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let file = File::open("hello.txt");

    let file = match file {
        Ok(f) => f,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => {
                match File::create("hello.txt") {
                    Ok(fc) => fc,
                    Err(e) => panic!("Couldn't create file: {:?}", e),
                }
            }
            ErrorKind::PermissionDenied => {
                panic!("Permission denied!");
            }
            other_error => {
                panic!("Couldn't open file: {:?}", other_error);
            }
        },
    };
}
```

### Shortcuts: unwrap and expect

```rust
use std::fs::File;

fn main() {
    // unwrap: panic with generic message on Err
    let file = File::open("hello.txt").unwrap();

    // expect: panic with custom message on Err
    let file = File::open("hello.txt")
        .expect("hello.txt should be included in this project");

    // unwrap_or: provide default on Err
    let file = File::open("hello.txt")
        .unwrap_or(File::create("hello.txt").unwrap());

    // unwrap_or_else: compute default lazily
    let file = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == std::io::ErrorKind::NotFound {
            File::create("hello.txt").unwrap()
        } else {
            panic!("Couldn't open file: {:?}", error);
        }
    });
}
```

### Result Methods for Error Handling

```rust
fn main() {
    let result: Result<i32, &str> = Ok(5);

    // Check state
    if result.is_ok() {
        println!("Success!");
    }

    // Map success value
    let doubled = result.map(|x| x * 2); // Ok(10)

    // Map error value
    let err: Result<i32, &str> = Err("error");
    let upper = err.map_err(|e| e.to_uppercase()); // Err("ERROR")

    // Provide default
    let value = err.unwrap_or(0); // 0
    let value = err.unwrap_or_default(); // 0 (uses Default trait)

    // Convert to Option
    let opt = result.ok(); // Some(5)
    let opt = err.err();   // Some("error")
}
```

---

## Error Propagation

The `?` operator's core mechanics (unwrapping `Ok`, returning `Err` early, and automatic
error conversion via `From`) are covered in
[Chapter 19: The ? Operator](VII.19-option-and-result-types.md#the--operator). The two
extensions below aren't.

### ? with Option

```rust
fn last_char_of_first_line(text: &str) -> Option<char> {
    text.lines().next()?.chars().last()
}

fn main() {
    let result = last_char_of_first_line("Hello\nWorld");
    println!("{:?}", result); // Some('o')

    let result = last_char_of_first_line("");
    println!("{:?}", result); // None
}
```

### Chaining with ?

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader};

fn count_lines(path: &str) -> Result<usize, io::Error> {
    let file = File::open(path)?;
    let reader = BufReader::new(file);
    let count = reader.lines().count();
    Ok(count)
}

// More complex chaining
fn process_config() -> Result<Config, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string("config.json")?;
    let config: Config = serde_json::from_str(&content)?;
    validate_config(&config)?;
    Ok(config)
}

struct Config {
    // fields...
}

fn validate_config(_config: &Config) -> Result<(), &'static str> {
    Ok(())
}
```

---

Custom error types (simple string errors, enum error types, errors carrying data) and
the `std::error::Error` trait (implementing it, wrapping other errors with `From`, and
the resulting error chains) are covered in
[Chapter 19: Custom Error Types](VII.19-option-and-result-types.md#custom-error-types).
The rest of this chapter builds on that: when to reach for `thiserror` vs. `anyhow`, and
higher-level patterns for structuring error handling.

## Error Chain

A wrapped error (like `ConfigError::Io` from the previous chapter) hides the lower-level
error inside it, but `source()` still exposes that inner error. Walking `source()`
repeatedly, as `print_error_chain` does below, prints the full causal chain instead of
just the outermost, most-generic message.

```rust
use std::error::Error;

fn print_error_chain(error: &dyn Error) {
    eprintln!("Error: {}", error);

    let mut current = error.source();
    while let Some(cause) = current {
        eprintln!("Caused by: {}", cause);
        current = cause.source();
    }
}

fn main() {
    if let Err(e) = read_config() {
        print_error_chain(&e);
    }
}

fn read_config() -> Result<(), Box<dyn Error>> {
    // ... operation that might fail
    Ok(())
}
```

---

## Error Handling Crates

### thiserror (For Libraries)

`#[derive(Error)]` generates the `Display` and `Error` implementations that writing them
by hand (as in the previous chapter) requires: `#[error("...")]` becomes the `Display`
message, with `{path}`/`{0}` interpolating fields; `#[source]` marks which field
`source()` returns; and `#[from]` additionally generates a `From` impl, so `?` can
convert into that variant automatically. `#[error(transparent)]` forwards both `Display`
and `source()` straight to the wrapped error, for a variant that's just a pass-through.
See [Chapter 25](IX.25-third-party-library-reference.md#thiserror--error-types) for the
full plain-Rust expansion of what the derive macro generates.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum DataError {
    #[error("Failed to read data from {path}")]
    ReadError {
        path: String,
        #[source]
        source: std::io::Error,
    },

    #[error("Invalid data format: {0}")]
    FormatError(String),

    #[error("Data validation failed")]
    ValidationError {
        #[from]
        source: ValidationError,
    },

    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

#[derive(Error, Debug)]
#[error("Validation failed: {message}")]
struct ValidationError {
    message: String,
}
```

### anyhow (For Applications)

`anyhow::Result<T>` is shorthand for `Result<T, anyhow::Error>`, a single error type that
can hold any error implementing `std::error::Error`, which is why application code that
doesn't need to match on specific error variants can use it everywhere instead of
defining an enum per module. `.context(...)` attaches a message to an error as it's
propagated, so the top-level handler sees "failed to read file: foo.txt" layered over
the original I/O error rather than just the latter on its own. `bail!` returns an `Err`
immediately with a formatted message, and `ensure!` is `if !cond { bail!(...) }` in one
call, mirroring `assert!` but returning an error instead of panicking.

```rust
use anyhow::{Context, Result, bail, ensure};

fn process_file(path: &str) -> Result<String> {
    let content = std::fs::read_to_string(path)
        .context(format!("Failed to read file: {}", path))?;

    ensure!(!content.is_empty(), "File is empty");

    if content.starts_with("ERROR") {
        bail!("File contains error marker");
    }

    Ok(content)
}

fn main() -> Result<()> {
    let content = process_file("data.txt")?;
    println!("{}", content);
    Ok(())
}
```

### Comparison

| Feature | thiserror | anyhow |
|---------|-----------|--------|
| Use Case | Libraries | Applications |
| Error Types | Custom enums | Box<dyn Error> |
| Adds Derive | Yes | No |
| Context | Manual | Built-in |
| Performance | Zero-cost | Small overhead |

---

## Quick Reference

### Error handling types

| Type | Use case | Example |
|------|----------|---------|
| `panic!` | Unrecoverable bugs | `panic!("invariant violated")` |
| `Result<T, E>` | Recoverable errors | `File::open(path)` |
| `Option<T>` | Absence (not error) | `map.get(key)` |

### Error handling methods

| Method | Description |
|--------|-------------|
| `?` | Propagate error early |
| `unwrap()` | Panic on error |
| `expect()` | Panic with message |
| `unwrap_or()` | Provide default |
| `unwrap_or_else()` | Compute default |
| `map()` | Transform success |
| `map_err()` | Transform error |
| `and_then()` | Chain operations |
| `or_else()` | Handle error |

---

## Common Patterns

### Application Entry Point Patterns

When multiple functions return `Result`, there are several strategies for handling them uniformly at the top level (e.g., in `main`).

#### Pattern 1: `run()` + unified `AppError` (most idiomatic for CLIs)

Define a top-level `AppError` with `thiserror`, delegate all logic to `run()`, and handle the error once in `main`:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error(transparent)]
    Track(#[from] TrackError),
    #[error(transparent)]
    Config(#[from] ConfigError),
}

fn main() {
    if let Err(e) = run() {
        eprintln!("Error: {}", e);
        std::process::exit(1);
    }
}

fn run() -> Result<(), AppError> {
    let config = config::load()?;
    let cli = Cli::parse();

    match cli.command {
        Commands::Track { subcommand } => match subcommand {
            TrackCommands::List => commands::track::list(&config)?,
            // other arms...
        },
    }

    Ok(())
}
```

This is the pattern used by `cargo` and most production Rust CLIs. Error handling is centralized, `?` propagates cleanly, and each command function focuses solely on its logic.

#### Pattern 2: `main` returns `Result` directly

Quick and low-ceremony. Rust prints `Error: <msg>` automatically on `Err`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = config::load()?;
    // ...
    commands::track::list(&config)?;
    Ok(())
}
```

Downside: less control over exit codes and error message formatting.

#### Pattern 3: `anyhow` for ergonomic application error handling

Same as Pattern 2 but with richer context via `.context()`:

```rust
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let config = config::load().context("Failed to load config")?;
    commands::track::list(&config).context("Failed to list tracks")?;
    Ok(())
}
```

Best for applications where typed error matching isn't needed. Pair with `thiserror` in library code.

#### Pattern 4: per-call `unwrap_or_else`

Handle each `Result` individually at the call site:

```rust
commands::track::list(&config).unwrap_or_else(|e| {
    eprintln!("Error: {}", e);
    std::process::exit(1);
});
```

Avoid this when many commands return `Result` — it becomes repetitive and scatters error handling logic.

---

The early-return pattern for validating and unwrapping several fallible steps in a row
is covered in
[Chapter 19: Common Patterns](VII.19-option-and-result-types.md#common-patterns).

### Try Block Pattern (Unstable)

```rust
// Requires #![feature(try_blocks)]
// fn example() -> Result<(), Error> {
//     let result: Result<i32, Error> = try {
//         let a = operation1()?;
//         let b = operation2(a)?;
//         a + b
//     };
//     // Handle result...
// }

// Stable alternative: closure
fn example() -> Result<(), Box<dyn std::error::Error>> {
    let result: Result<i32, Box<dyn std::error::Error>> = (|| {
        let a: i32 = "5".parse()?;
        let b: i32 = "10".parse()?;
        Ok(a + b)
    })();

    match result {
        Ok(sum) => println!("Sum: {}", sum),
        Err(e) => eprintln!("Error: {}", e),
    }
    Ok(())
}
```

The same fallback-chain idea built on `Result` instead of `Option` is covered in
[Chapter 19: Common Patterns](VII.19-option-and-result-types.md#common-patterns).

### Collecting All Errors

`?` and the early-return pattern both stop at the first failure. That's right for a
sequence of dependent steps, but wrong for something like validating a form: the user
wants every problem reported at once, not one at a time across repeated submissions.
`filter_map` here keeps only the `Err` side of each attempt (via `.err()`), building up
every failure instead of returning on the first one.

```rust
fn validate_all(items: &[String]) -> Result<(), Vec<String>> {
    let errors: Vec<String> = items
        .iter()
        .filter_map(|item| validate_item(item).err())
        .collect();

    if errors.is_empty() {
        Ok(())
    } else {
        Err(errors)
    }
}

fn validate_item(item: &str) -> Result<(), String> {
    if item.is_empty() {
        Err("Item is empty".to_string())
    } else {
        Ok(())
    }
}
```

### Retry Pattern

The operation is taken as `FnMut` rather than `Fn` because it needs to be called more
than once, and a closure that mutates something it captures (a counter, a connection
handle) only implements `FnMut`. A plain `Fn` bound would reject that closure even
though calling it repeatedly is exactly the point here.

```rust
use std::time::Duration;
use std::thread;

fn retry<T, E, F>(mut operation: F, max_attempts: u32, delay: Duration) -> Result<T, E>
where
    F: FnMut() -> Result<T, E>,
{
    let mut attempts = 0;

    loop {
        attempts += 1;
        match operation() {
            Ok(value) => return Ok(value),
            Err(e) if attempts >= max_attempts => return Err(e),
            Err(_) => {
                thread::sleep(delay);
            }
        }
    }
}

fn flaky_operation() -> Result<String, &'static str> {
    static mut CALLS: u32 = 0;
    unsafe {
        CALLS += 1;
        if CALLS < 3 {
            Err("Failed")
        } else {
            Ok("Success".to_string())
        }
    }
}

fn main() {
    let result = retry(flaky_operation, 5, Duration::from_millis(100));
    println!("{:?}", result);
}
```

### Context Pattern

This is roughly what `anyhow`'s `.context()` does under the hood, built from scratch
with plain `std::error::Error`: `ResultExt` is an extension trait that adds a `context`
method to every `Result`, and `ContextError<E>` wraps the original error together with a
message while keeping it reachable through `source()`, so nothing about the original
error is lost, only a description of what the caller was trying to do is added on top.

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
struct ContextError<E> {
    context: String,
    source: E,
}

impl<E: Error> fmt::Display for ContextError<E> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}: {}", self.context, self.source)
    }
}

impl<E: Error + 'static> Error for ContextError<E> {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        Some(&self.source)
    }
}

trait ResultExt<T, E> {
    fn context(self, ctx: &str) -> Result<T, ContextError<E>>;
}

impl<T, E> ResultExt<T, E> for Result<T, E> {
    fn context(self, ctx: &str) -> Result<T, ContextError<E>> {
        self.map_err(|e| ContextError {
            context: ctx.to_string(),
            source: e,
        })
    }
}

fn read_config() -> Result<String, std::io::Error> {
    std::fs::read_to_string("config.txt")
}

fn main() {
    let result = read_config().context("Failed to load configuration");
    if let Err(e) = result {
        eprintln!("{}", e);
    }
}
```

---

## When to Panic vs Return Error

### Use panic! for:

1. **Bugs in your code**
   ```rust
   fn get_element(slice: &[i32], index: usize) -> i32 {
       // If index is out of bounds, it's a bug
       slice[index]
   }
   ```

2. **Invariant violations**
   ```rust
   fn process(data: &[u8]) {
       assert!(!data.is_empty(), "Data must not be empty");
       // ... process data
   }
   ```

3. **Unrecoverable situations**
   ```rust
   fn initialize() {
       let config = Config::load()
           .expect("Cannot proceed without configuration");
   }
   ```

4. **Examples and prototypes**
   ```rust
   fn main() {
       let file = File::open("file.txt").unwrap();
   }
   ```

### Use Result for:

1. **Expected failures**
   ```rust
   fn read_file(path: &str) -> Result<String, io::Error> {
       std::fs::read_to_string(path)
   }
   ```

2. **User input validation**
   ```rust
   fn parse_age(input: &str) -> Result<u32, ParseError> {
       let age: u32 = input.parse()?;
       if age > 150 {
           return Err(ParseError::InvalidAge);
       }
       Ok(age)
   }
   ```

3. **External system interactions**
   ```rust
   fn fetch_data(url: &str) -> Result<Data, NetworkError> {
       // Network can fail
   }
   ```

4. **Library functions**
   ```rust
   // Let the caller decide how to handle errors
   pub fn connect(addr: &str) -> Result<Connection, ConnectError> {
       // ...
   }
   ```

### Decision Guide

| Question | panic! | Result |
|----------|--------|--------|
| Is it a bug? | ✓ | |
| Can caller recover? | | ✓ |
| Is it expected to fail? | | ✓ |
| Is it in a library? | | ✓ |
| Is it prototype/example? | ✓ | |
| Is it during initialization? | Maybe | Maybe |

---

## Testing Error Conditions

### Testing panic!

```rust
#[cfg(test)]
mod tests {
    #[test]
    #[should_panic]
    fn test_panic() {
        panic!("This test should panic");
    }

    #[test]
    #[should_panic(expected = "divide by zero")]
    fn test_panic_message() {
        divide(10, 0);
    }

    fn divide(a: i32, b: i32) -> i32 {
        if b == 0 {
            panic!("Cannot divide by zero");
        }
        a / b
    }
}
```

### Testing Results

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_ok_result() {
        let result = parse_positive("42");
        assert!(result.is_ok());
        assert_eq!(result.unwrap(), 42);
    }

    #[test]
    fn test_err_result() {
        let result = parse_positive("-5");
        assert!(result.is_err());
    }

    #[test]
    fn test_specific_error() {
        let result = parse_positive("-5");
        match result {
            Err(ParseError::Negative) => (), // Expected
            _ => panic!("Expected Negative error"),
        }
    }

    fn parse_positive(s: &str) -> Result<i32, ParseError> {
        let n: i32 = s.parse().map_err(|_| ParseError::InvalidFormat)?;
        if n < 0 {
            Err(ParseError::Negative)
        } else {
            Ok(n)
        }
    }

    #[derive(Debug, PartialEq)]
    enum ParseError {
        InvalidFormat,
        Negative,
    }
}
```

### Testing with ? in Tests

```rust
#[cfg(test)]
mod tests {
    use std::error::Error;

    #[test]
    fn test_with_result() -> Result<(), Box<dyn Error>> {
        let value: i32 = "42".parse()?;
        assert_eq!(value, 42);
        Ok(())
    }
}
```

---

## Common Pitfalls

### Letting `main` return `Result` hide exit codes and formatting

Pattern 2 (`main` returning `Result<(), Box<dyn Error>>` directly) is quick, but Rust's
default `Error: <msg>` printing gives you less control over exit codes and error message
formatting than the `run()` + `AppError` pattern does.

### Scattering error handling with per-call `unwrap_or_else`

Handling each `Result` individually at the call site (Pattern 4) doesn't scale: once
several commands return `Result`, the same `unwrap_or_else` block repeats at every call
site and error handling logic ends up scattered across the codebase instead of
centralized in one place.

---

## Summary

This chapter covers Rust's two error handling mechanisms, `panic!` for unrecoverable
bugs and `Result<T, E>` for recoverable failures, along with `thiserror` and `anyhow` for
building and propagating error types, and application-level patterns (see Quick
Reference above) for wiring `Result` up to a program's entry point. The custom error
types and `?` operator mechanics that these patterns build on live in
[Chapter 19: Option & Result Types](VII.19-option-and-result-types.md).

1. Return `Result` from functions that can fail
2. Use `?` for propagation within functions
3. Create domain-specific error types for libraries
4. Use `anyhow` for applications, `thiserror` for libraries
5. Provide context when propagating errors
6. Only panic for bugs and unrecoverable situations
7. Document error conditions in function documentation
