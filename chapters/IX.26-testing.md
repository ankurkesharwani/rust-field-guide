# Testing

## Table of Contents

1. [Introduction](#introduction)
2. [Writing Tests with #[test]](#writing-tests-with-test)
3. [assert! and assert_eq!](#assert-and-assert_eq)
4. [Conditional Compilation with #[cfg(test)]](#conditional-compilation-with-cfgtest)
5. [Integration Tests](#integration-tests)
6. [Doc-Tests](#doc-tests)
7. [Quick Reference](#quick-reference)
8. [Common Patterns](#common-patterns)
9. [Common Pitfalls](#common-pitfalls)
10. [Summary](#summary)

---

## Introduction

Rust builds its test runner into `cargo` rather than treating it as a separate tool.
Marking a function `#[test]` and running `cargo test` is enough to get a working test
suite, with no test framework to add as a dependency. This chapter covers `#[test]`
functions, the `assert!` family of macros, why test code lives behind
`#[cfg(test)]`, the difference between unit tests (in `src/`) and integration tests (in
`tests/`), and doc-tests, which run the code fences inside your documentation comments
as tests.

---

## Writing Tests with #[test]

A function marked `#[test]` becomes a test that `cargo test` will run. The function
takes no arguments and its return type is either `()` or a `Result`; `cargo test`
considers the test passed if the function returns without panicking (and, for a
`Result`-returning test, if it returns `Ok`).

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 2), 4);
    }
}
```

`cargo test` finds every `#[test]` function in the crate by scanning for the attribute
at compile time, not by naming convention, so a test function can be called anything.
By default it runs them concurrently, one per thread, which is why two tests should
never read and write the same piece of shared mutable state (a global counter, a
temp file with a fixed name, an environment variable) without their own synchronization:
run in parallel, they'll race. `cargo test -- --test-threads=1` runs them one at a time
if you need to confirm that a failure is a real bug and not a race between tests.

You can run a subset of tests by passing a substring that matches the test's fully
qualified name: `cargo test adds` runs every test whose name contains `adds`. A test
marked `#[ignore]` is skipped by default and only runs with `cargo test -- --ignored`,
which is the usual way to keep a slow or environment-dependent test in the suite
without slowing down every `cargo test` invocation.

```rust
#[test]
#[ignore]
fn slow_integration_style_test() {
    // Run explicitly with `cargo test -- --ignored`.
    assert_eq!(2 + 2, 4);
}
```

A test can also report failure by returning a `Result`. `cargo test` treats `Ok(())` as
a pass and any `Err` as a failure, printing the error's `Debug` output. This lets you use
`?` inside a test instead of `.unwrap()`, which matters when the thing under test
already returns `Result`:

```rust
pub fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn division_result_style() -> Result<(), String> {
        let quotient = divide(10, 2)?;
        assert_eq!(quotient, 5);
        Ok(())
    }
}
```

A `Result`-returning test cannot also be `#[should_panic]`; use one style or the other
depending on whether the failure you're testing for is a panic or an `Err`.

---

## assert! and assert_eq!

`assert!(condition)` panics, failing the test, if `condition` is `false`. The panic
message it produces is just the source location, so on its own it doesn't tell you what
was being compared. `assert_eq!(left, right)` and `assert_ne!(left, right)` exist for
the common case of comparing two values: on failure they panic with a message that
prints both sides, which is why they're preferred over `assert!(a == b)` whenever
there's a concrete left and right value to show.

```rust
fn main() {
    assert_eq!(2 + 2, 5, "math is broken");
}
```

Running that produces:

```text
thread 'main' panicked at src/main.rs:2:5:
assertion `left == right` failed: math is broken
  left: 4
 right: 5
```

Both `assert!` and `assert_eq!`/`assert_ne!` take an optional trailing format string
(and arguments) as a custom failure message, shown above as `"math is broken"`. Both
macros require `left` and `right` to implement `PartialEq`, and `assert_eq!`/`assert_ne!`
additionally require `Debug`, since that's what prints the two values on failure.

`assert!`, `assert_eq!`, and `assert_ne!` all run in release builds too (they're not
behind `#[cfg(test)]` or `debug_assertions`), unlike `debug_assert!` and
`debug_assert_eq!`, which compile to nothing when `debug-assertions` is off (the
default for `cargo build --release`). Reach for the `debug_` variants for checks that
are only worth the runtime cost during development, and the plain variants for
invariants you want checked in production too.

---

## Conditional Compilation with #[cfg(test)]

`#[cfg(test)]` is conditional compilation, the same mechanism behind
`#[cfg(target_os = "linux")]` or a Cargo feature flag: the compiler only includes the
annotated item when the crate is built for testing. `mod tests` in the examples above
is only compiled in when you run `cargo test`; a normal `cargo build` or `cargo run`
never sees it, so test helpers, test-only imports, and test data don't bloat the
binary you ship or leak test-only dependencies into it.

```rust
pub struct Config {
    pub max_size: usize,
}

#[cfg(test)]
mod tests {
    use super::*;

    // Only exists in test builds; `cargo build` never compiles this.
    fn test_config() -> Config {
        Config { max_size: 100 }
    }

    #[test]
    fn config_has_expected_size() {
        assert_eq!(test_config().max_size, 100);
    }
}
```

Because `mod tests` is a regular child module in the same crate, `use super::*;` gives
it access to everything the parent module can see, private items included. That's the
key difference from integration tests, covered next: a unit test in `src/` can exercise
private functions directly, while code outside the crate (including a test in `tests/`)
can only reach what's `pub`.

---

## Integration Tests

Each `.rs` file directly under a `tests/` directory at the crate root is compiled as
its own separate crate that depends on yours, the same way an external user's code
would. That means an integration test can only call your crate's public API. It's the
right place to test the crate the way a caller actually uses it, and the wrong place
to reach for a private helper function.

```rust
// tests/integration_test.rs
use my_crate::{add, divide};

#[test]
fn add_from_outside_the_crate() {
    assert_eq!(add(3, 4), 7);
}

#[test]
fn divide_from_outside_the_crate() {
    assert_eq!(divide(9, 3), Ok(3));
}
```

Each file in `tests/` gets its own binary and its own `mod` namespace, so `cargo test`
compiles and runs them as independent crates rather than one combined test binary; you
can run just one integration test file with `cargo test --test integration_test`. A
file under `tests/` that isn't meant to be a test suite on its own (shared setup code, a
fixture builder) should go in a subdirectory such as `tests/common/mod.rs` rather than
directly under `tests/`, since anything directly under `tests/` is compiled as a
separate test crate and would otherwise show up as an (empty) test run of its own.

Only library crates (or a binary crate that also exposes a `lib.rs`) can be integration
tested this way, because `tests/` files depend on the crate as a library; a pure binary
crate with only `src/main.rs` has no public API for an integration test to import.

---

## Doc-Tests

A `///` doc comment on an item can include a fenced code block, and `cargo test`
compiles and runs each one as its own test, on the theory that an example that no
longer compiles is worse than no example at all.

```rust
/// Adds one to the given number.
///
/// ```
/// let x = my_crate::add_one(5);
/// assert_eq!(x, 6);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

`cargo test` (or `cargo test --doc` to run only doc-tests) extracts the fenced block,
wraps it in a `fn main() { ... }`, compiles it as its own tiny program against your
crate as a dependency, and runs it; a panic or a non-zero exit fails the test just like
any other. Lines starting with `# ` (a `#` followed by a space) inside the fence are
compiled and run but hidden from the rendered documentation, which is how you set up
values or imports an example needs without cluttering what the reader sees:

```rust
/// ```
/// # let starting_value = 41;
/// let x = my_crate::add_one(starting_value);
/// assert_eq!(x, 42);
/// ```
```

Three fence annotations change how the block runs: ` ```ignore ` compiles nothing at
all (for pseudocode or a snippet with an intentional error, which is why it's the right
choice for the "math is broken" example earlier in this chapter), ` ```no_run `
compiles and runs `main`'s setup but skips actually invoking it, for an example that
would open a real socket or write a real file, and ` ```should_panic ` compiles and
runs it but expects the code to panic, the doc-test equivalent of `#[should_panic]`.
Leaving off `rust` after the triple backtick (plain ` ``` `) is treated the same as
` ```rust `; only an explicit other language tag (` ```text `, ` ```toml `) is compiled as
a doc-test.

---

## Quick Reference

### Attributes

| Attribute | Effect |
|---|---|
| `#[test]` | Marks a function as a test `cargo test` should run |
| `#[cfg(test)]` | Compiles the item only for test builds |
| `#[should_panic]` | Test passes only if the function panics |
| `#[should_panic(expected = "...")]` | Also requires the panic message to contain the substring |
| `#[ignore]` | Skips the test unless run with `cargo test -- --ignored` |

### Doc-test fence annotations

| Fence | Behavior |
|---|---|
| ` ```` ` / ` ```rust ` | Compiled and run as a test |
| ` ```ignore ` | Not compiled at all |
| ` ```no_run ` | Compiled, not executed |
| ` ```should_panic ` | Compiled and run, must panic to pass |
| ` ```text `, ` ```toml `, ... | Rendered as a code block, never compiled |

### Common commands

```text
cargo test                       # unit + integration + doc tests
cargo test <substring>           # only tests whose name contains <substring>
cargo test -- --ignored          # only #[ignore]d tests
cargo test -- --test-threads=1   # run tests sequentially
cargo test -- --nocapture        # show println! output even for passing tests
cargo test --doc                 # only doc-tests
cargo test --test integration_test  # only tests/integration_test.rs
```

---

## Common Patterns

- Return `Result<(), E>` from a test to use `?` on fallible setup instead of `.unwrap()`, keeping the failure's real error message in the test output. See Writing Tests with #[test] above.
- Prefer `assert_eq!`/`assert_ne!` over `assert!` whenever there's a concrete left and right value, since the failure message shows both sides instead of just `false`. See assert! and assert_eq! above.
- Put fixtures and helpers shared by several integration test files in `tests/common/mod.rs` rather than directly under `tests/`, so they aren't compiled as a test crate of their own. See Integration Tests above.
- Use a doc-test's hidden `# ` lines for setup so the rendered example stays focused on the API being demonstrated. See Doc-Tests above.

---

## Common Pitfalls

- Tests run concurrently by default, so two tests that touch the same global state (a fixed temp file path, an environment variable, a `static` counter) can flake without any bug in the code under test. See Writing Tests with #[test] above.
- A unit test in `src/` can see private items through `super::*`; an integration test in `tests/` compiles as a separate crate and can only reach `pub` items, so a test that needs a private helper belongs in `src/`, not `tests/`. See Conditional Compilation with #[cfg(test)] and Integration Tests above.
- `debug_assert!`/`debug_assert_eq!` compile to nothing when debug assertions are off (the `cargo build --release` default), so a check that must hold in production needs `assert!`/`assert_eq!` instead. See assert! and assert_eq! above.
- `#[should_panic]` and a `Result`-returning test signature don't combine; a test testing for an `Err` should return `Result` and check the error value, not panic. See Writing Tests with #[test] above.

---

## Summary

`#[test]` functions, run by `cargo test`, are Rust's built-in unit of testing; `assert!`
and `assert_eq!`/`assert_ne!` are how they report failure, with the latter printing both
compared values. `#[cfg(test)]` keeps test-only code out of non-test builds. Unit tests
in `src/` can see private items; integration tests in `tests/` compile as separate
crates against your public API only. Doc-tests compile and run the code fences in your
`///` comments, keeping documentation examples honest. The attributes, fence
annotations, and commands above are collected in the Quick Reference section.
