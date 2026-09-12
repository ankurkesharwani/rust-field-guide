# Type Conversions

## Table of Contents

1. [Introduction](#introduction)
2. [From and Into](#from-and-into)
3. [TryFrom and TryInto](#tryfrom-and-tryinto)
4. [Parsing with .parse()](#parsing-with-parse)
5. [Quick Reference](#quick-reference)
6. [Common Patterns](#common-patterns)
7. [Common Pitfalls](#common-pitfalls)
8. [Summary](#summary)

---

## Introduction

[Chapter 2: Data Types](I.2-data-types.md#casting-with-as) covers `as`, which converts
between Rust's built-in numeric and primitive types. `as` is a keyword: the compiler
knows how to reinterpret bits between `i32` and `f64`, or truncate a `u64` down to a
`u8`, and it does that conversion unconditionally, even when the result is wrong (a
`300i32 as u8` silently becomes `44`).

This chapter covers a different kind of conversion: trait-based conversions between
your own types, or between library types, expressed through `From`, `Into`, `TryFrom`,
`TryInto`, and `.parse()`. These aren't compiler built-ins. They're ordinary traits
implemented in the standard library and in your own code, which means the compiler
enforces nothing beyond "does an implementation exist" and the conversion logic itself
is exactly the code you or a library author wrote. [Chapter 13: Traits and Trait
Implementations](IV.13-traits-and-trait-implementations.md#common-standard-library-traits)
already introduced `From`/`Into` and `TryFrom`/`TryInto` as two entries in the standard
trait table; this chapter goes further into fallibility, error types, chaining, and the
`?` operator.

---

## From and Into

`From<T>` converts a value of type `T` into `Self`, and it's infallible: implementing
`From` is a promise that the conversion always succeeds. `Into<U>` is the mirror image,
converting `Self` into `U`. You almost never implement `Into` directly, because the
standard library provides a blanket implementation:

```rust
// From the standard library (conceptually):
// impl<T, U> Into<U> for T where U: From<T> {
//     fn into(self) -> U {
//         U::from(self)
//     }
// }
```

Implementing `From<T> for U` gives you `Into<U> for T` for free. Which direction you
call depends on which type the context already knows:

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

fn print_temp(f: Fahrenheit) {
    println!("{}F", f.0);
}

fn main() {
    // Explicit: `From` names the target type
    let f = Fahrenheit::from(Celsius(100.0));

    // Implicit: `.into()` needs the target type to be inferable
    print_temp(Celsius(0.0).into());
    let f2: Fahrenheit = Celsius(37.0).into();
}
```

`Fahrenheit::from(...)` reads better when the surrounding code doesn't already make the
target type obvious. `.into()` reads better when it does, such as a function parameter
with a known type or a `let` binding with an explicit annotation. `.into()` can't be
called on its own; the compiler needs some other clue in the expression to know which
`From` implementation to run, because in principle several types could implement
`From<Celsius>`.

Because `Into` is generic, you can also accept "anything convertible to `Fahrenheit`"
as a bound instead of a concrete type:

```rust
fn take_fahrenheit<T: Into<Fahrenheit>>(t: T) -> f64 {
    let f: Fahrenheit = t.into();
    f.0
}
```

This is the same shape you'll see in library APIs that accept `impl Into<String>` or
`impl Into<PathBuf>`: the function commits to one internal representation but lets
callers hand in anything with a conversion path to it, instead of forcing every caller
to write the conversion out by hand first.

---

## TryFrom and TryInto

Some conversions can fail. Converting an arbitrary `i32` into a `u8` might overflow;
parsing a string into a struct might find the string malformed; converting a raw
integer into an enum might land on a value with no matching variant. `From` isn't the
right tool for these, because `From` has no way to report failure; it can only return
`Self`. `TryFrom` exists for exactly this case: it returns a `Result` and has an
associated `Error` type, so the caller is forced to decide what happens when the
conversion doesn't work, instead of the conversion silently producing a garbage value
or panicking.

```rust
use std::convert::TryFrom;
use std::fmt;

#[derive(Debug)]
struct ParsePercentError(i32);

impl fmt::Display for ParsePercentError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} is not a valid percent (0..=100)", self.0)
    }
}

struct Percent(u8);

impl TryFrom<i32> for Percent {
    type Error = ParsePercentError;

    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if (0..=100).contains(&value) {
            Ok(Percent(value as u8))
        } else {
            Err(ParsePercentError(value))
        }
    }
}

fn main() {
    let ok = Percent::try_from(50);
    let err = Percent::try_from(150);

    println!("{:?}", ok.map(|p| p.0));   // Ok(50)
    println!("{:?}", err.map(|p| p.0));  // Err(ParsePercentError(150))
}
```

`TryInto` mirrors `Into` the same way `TryFrom` mirrors `From`, and the standard
library gives it to you for free once you implement `TryFrom`. Because the result is a
plain `Result`, a fallible conversion composes with `?` inside a function that already
returns a compatible `Result`:

```rust
fn load_percent(raw: i32) -> Result<Percent, ParsePercentError> {
    let p = Percent::try_from(raw)?;
    Ok(p)
}
```

There's also a blanket implementation that connects `TryFrom` back to `From`:

```rust
// From the standard library (conceptually):
// impl<T, U> TryFrom<U> for T where U: Into<T> {
//     type Error = Infallible;
//     fn try_from(value: U) -> Result<Self, Self::Error> {
//         Ok(T::from(value))
//     }
// }
```

Every infallible `From` conversion also gets a `TryFrom` whose error type is
[`std::convert::Infallible`](https://doc.rust-lang.org/std/convert/enum.Infallible.html),
an enum with no variants that can never actually be constructed. This matters in
generic code: a function written against `T: TryFrom<U>` works uniformly whether the
underlying conversion is fallible or not, and code that pattern-matches on the `Err`
case of an `Infallible` result is unreachable by construction, not just by convention.

---

## Parsing with .parse()

`.parse()` converts a string slice into some other type `T`, and it works for any `T`
that implements [`FromStr`](https://doc.rust-lang.org/std/str/trait.FromStr.html). It's
a `TryFrom`-shaped conversion (fallible, with an associated `Err` type) that predates
`TryFrom` in the standard library and has stuck around because it reads naturally at
the call site:

```rust
fn main() {
    let n: i32 = "42".parse().unwrap();
    let n2 = "42".parse::<i32>().unwrap();
    println!("{n} {n2}");

    let bad: Result<i32, _> = "abc".parse();
    println!("{}", bad.is_err()); // true
}
```

`.parse()` is generic over its return type, so the compiler has to know what you're
parsing into before it can pick a `FromStr` implementation. The two calls above show
the two ways to supply that: an explicit type annotation on the binding (`let n: i32`),
or the turbofish `::<i32>` on the call itself. Without either, `"42".parse()` alone is
ambiguous and won't compile, because `bool`, `f64`, `i32`, and every other `FromStr`
implementor are all equally valid candidates as far as the parser call is concerned.

Implementing `FromStr` for your own type makes it parseable the same way:

```rust
use std::str::FromStr;

struct Point {
    x: i32,
    y: i32,
}

impl FromStr for Point {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let (x_str, y_str) = s
            .split_once(',')
            .ok_or_else(|| format!("expected \"x,y\", got {s:?}"))?;

        let x = x_str.trim().parse().map_err(|_| format!("bad x: {x_str:?}"))?;
        let y = y_str.trim().parse().map_err(|_| format!("bad y: {y_str:?}"))?;

        Ok(Point { x, y })
    }
}

fn main() -> Result<(), String> {
    let p: Point = "3, 4".parse()?;
    println!("{}, {}", p.x, p.y);
    Ok(())
}
```

---

## Quick Reference

| Trait | Direction | Fallible? | Typical use |
|---|---|---|---|
| `From<T>` | `T` to `Self` | No | Define once, get `Into` for free |
| `Into<U>` | `Self` to `U` | No | Blanket impl from `From`; call site sugar |
| `TryFrom<T>` | `T` to `Self` | Yes (`Result`) | Conversions that can be rejected |
| `TryInto<U>` | `Self` to `U` | Yes (`Result`) | Blanket impl from `TryFrom` |
| `FromStr` | `&str` to `Self` | Yes (`Result`) | Backs `.parse::<Self>()` |

```rust
// Implement one direction, get the other for free:
impl From<A> for B { /* ... */ }       // gives Into<B> for A
impl TryFrom<A> for B { /* ... */ }    // gives TryInto<B> for A

// Every From also gives a TryFrom with Error = Infallible
```

---

## Common Patterns

- Accept `impl Into<T>` as a function parameter type to let callers pass anything with a conversion path to `T`, instead of forcing them to convert by hand first. See From and Into above.
- Use `?` on a `TryFrom`/`TryInto`/`.parse()` call inside a function that returns a compatible `Result`, rather than matching on `Ok`/`Err` by hand. See TryFrom and TryInto and Parsing with .parse() above.
- Implement `FromStr` on a small value type (a coordinate, an ID, a config value) so it can be parsed with the same `.parse()` call site used for primitives. See Parsing with .parse() above.
- Reach for `TryFrom` with a small custom error enum instead of a `String` error once a fallible conversion needs to distinguish more than one failure reason; see [Chapter 20: Error Handling Strategy](VII.20-error-handling-strategy.md) for designing that enum.

---

## Common Pitfalls

- `.parse()` alone doesn't compile without something else in the expression pinning down the target type; add a `let` annotation or a turbofish. See Parsing with .parse() above.
- `From`/`Into` can't report failure. A `From` implementation that panics or silently clamps out-of-range input on invalid data should usually be `TryFrom` instead, so the caller has to handle the failure case. See TryFrom and TryInto above.
- Implementing both `From<A> for B` and `TryFrom<A> for B` by hand for the same pair of types conflicts with the standard library's blanket `TryFrom` (which already covers that case via `From`); implement whichever one matches the conversion's fallibility and let the blanket impl provide the other trait.
- An `Infallible` error type means the `Err` arm is unreachable, not that it can be skipped: code that's generic over `T: TryFrom<U>` still has to handle the `Result`, even when a specific instantiation can never actually produce an `Err`.

---

## Summary

`From`/`Into` handle conversions that always succeed; `TryFrom`/`TryInto` and
`.parse()`/`FromStr` handle conversions that might not, returning a `Result` instead of
panicking or producing a wrong-but-valid value. Implementing one direction of `From` or
`TryFrom` gives you the other (`Into` or `TryInto`) for free through blanket
implementations in the standard library, and every infallible `From` conversion is also
a `TryFrom` conversion whose error type, `Infallible`, can never actually be
constructed. Together with the `as`-casting from [Chapter 2](I.2-data-types.md#casting-with-as),
this covers the full range of conversions built into ordinary Rust code.
