# Macros

## Table of Contents

1. [Introduction](#introduction)
2. [macro_rules! Basics](#macro_rules-basics)
3. [A Small Logging/Assertion-Style Macro](#a-small-loggingassertion-style-macro)
4. [Macro vs. Function](#macro-vs-function)
5. [Quick Reference](#quick-reference)
6. [Common Patterns](#common-patterns)
7. [Common Pitfalls](#common-pitfalls)
8. [Summary](#summary)

---

## Introduction

This is a light-touch chapter by design. It covers `macro_rules!`: what it's for, a
couple of realistic examples, and when to reach for a macro instead of a plain
function. It doesn't cover token-tree internals or how to write your own proc macro.
For how the derive macros this project already depends on (`clap`, `serde`,
`thiserror`) work from the caller's side, see [Chapter 25: Third-Party Library
Reference](IX.25-third-party-library-reference.md); those are a different kind of
macro (proc macros, run by a separate crate at compile time) than the
`macro_rules!` macros covered here.

---

## macro_rules! Basics

A `macro_rules!` macro matches the tokens it's called with against one or more
patterns, and expands to the code in whichever pattern matched. This is different from
a textual find-and-replace: the matcher works on Rust's token trees (identifiers,
literals, operators, and bracketed groups), not on raw text, and each pattern position
is tagged with a fragment specifier that says what kind of token tree is allowed there.
`expr` matches an expression, `ident` an identifier, `ty` a type, and `tt` a single
token tree (used as an escape hatch when nothing more specific fits).

```rust
macro_rules! square {
    ($x:expr) => {
        $x * $x
    };
}

fn main() {
    println!("{}", square!(5)); // 25
}
```

`$x:expr` binds whatever expression is passed as `$x`, and the macro body substitutes
it into `$x * $x`. Because the matcher requires a complete expression, `square!(5)`
expands to `(5) * (5)` in effect (the compiler is careful to preserve the argument as
one unit, so `square!(2 + 3)` doesn't accidentally expand to `2 + 3 * 2 + 3`).

A macro can generate more than an expression. This one generates a whole function
body, useful for cutting down on repetitive getter methods:

```rust
macro_rules! make_getter {
    ($field:ident, $ty:ty) => {
        fn $field(&self) -> $ty {
            self.$field
        }
    };
}

struct Point {
    x: i32,
    y: i32,
}

impl Point {
    make_getter!(x, i32);
    make_getter!(y, i32);
}

fn main() {
    let p = Point { x: 3, y: 4 };
    println!("{} {}", p.x(), p.y()); // 3 4
}
```

`$field:ident` only accepts a bare identifier, and `$ty:ty` only accepts a type, so
`make_getter!(x, i32)` couldn't accidentally be called with an expression in the wrong
position; the matcher rejects it before expansion.

Macros can also repeat over a variable number of arguments using `$(...)` with a
separator and a repetition operator (`*` for zero or more, `+` for one or more):

```rust
macro_rules! max_of {
    ($x:expr) => {
        $x
    };
    ($x:expr, $($rest:expr),+) => {{
        let x = $x;
        let rest_max = max_of!($($rest),+);
        if x > rest_max { x } else { rest_max }
    }};
}

fn main() {
    println!("{}", max_of!(3, 7, 2, 9, 4)); // 9
}
```

`macro_rules!` tries each pattern in order, top to bottom, and uses the first one that
matches. Here that means the single-argument pattern is the recursion's base case:
`max_of!(4)` matches the first arm directly, while `max_of!(9, 4)` matches the second
arm and recurses into `max_of!(4)`. This is the same shape the standard library's own
`vec![]` macro uses internally to accept a comma-separated list of any length.

---

## A Small Logging/Assertion-Style Macro

Beyond `expr`, `ident`, and `ty`, the `tt` (token tree) specifier matches a single
token or a single bracketed group, and `$($arg:tt)*` is the standard way to accept
"whatever `format!` would accept" and forward it unchanged:

```rust
macro_rules! log_debug {
    ($($arg:tt)*) => {
        println!("[DEBUG {}:{}] {}", file!(), line!(), format!($($arg)*))
    };
}

fn main() {
    log_debug!("starting up, pid = {}", 42);
    // [DEBUG src/main.rs:9] starting up, pid = 42
}
```

`file!()` and `line!()` are their own tiny built-in macros that expand to the current
source location at compile time, which is why the printed location always matches
where `log_debug!` was actually called, not where it's defined. Wrapping `format!` this
way is exactly how logging crates like `log` and `tracing` implement `debug!`, `info!`,
and friends: the macro adds a consistent prefix and lets the caller's arguments pass
straight through to `format!`'s own argument syntax.

The same `$($arg:tt)*` forwarding pattern works for a small assertion-style macro that
returns an error instead of panicking:

```rust
macro_rules! ensure {
    ($cond:expr, $($arg:tt)*) => {
        if !$cond {
            return Err(format!($($arg)*));
        }
    };
}

fn withdraw(balance: i32, amount: i32) -> Result<i32, String> {
    ensure!(amount > 0, "amount must be positive, got {amount}");
    ensure!(amount <= balance, "insufficient funds: have {balance}, need {amount}");
    Ok(balance - amount)
}

fn main() {
    println!("{:?}", withdraw(100, 50));  // Ok(50)
    println!("{:?}", withdraw(100, 500)); // Err("insufficient funds: have 100, need 500")
}
```

`ensure!` expands to a bare `if` with a `return` inside it, using the caller's own
`return`, not the macro's. That only works because `macro_rules!` expansion is
textual-after-matching in this sense: the expanded code runs in the caller's function,
so `return Err(...)` exits `withdraw`, not some hidden helper function. This is the
same idea behind the standard library's real `assert!` and the `?` operator: control
flow that a plain function couldn't express (a function called by `withdraw` can't
make `withdraw` return early) is exactly what a macro can, because the macro's output
is spliced directly into the call site.

---

## Macro vs. Function

Reach for a function first. A function is easier to read, easier to step through in a
debugger, and gets full type-checking on its own signature independent of any call
site. A macro is worth it when a function genuinely can't do the job:

- The number or kind of arguments varies (`println!`, `vec!`, `max_of!` above), where a
  function would need a fixed arity or a slice/`Vec` argument instead.
- The macro needs to generate code, not just a value, such as several functions, a
  struct's worth of boilerplate, or repeated `impl` blocks (`make_getter!` above).
- The macro needs to affect control flow in the caller, such as returning early from
  the caller's function (`ensure!` above) or the standard library's `?`, `assert!`, and
  `dbg!`.
- The macro needs compile-time information a function can't see, such as the call
  site's source location (`file!()`/`line!()`) or the argument as an unevaluated
  expression rather than its runtime value (useful for `assert!`, which prints the
  failing expression's source text in its panic message).

Outside of those cases, a macro adds a layer of indirection between the code you write
and the code that compiles, at zero real benefit over a function.

---

## Quick Reference

| Fragment specifier | Matches |
|---|---|
| `expr` | An expression |
| `ident` | An identifier |
| `ty` | A type |
| `tt` | A single token tree (fallback for anything else) |
| `pat` | A pattern |
| `block` | A `{ ... }` block |
| `path` | A path like `std::collections::HashMap` |

```rust
macro_rules! name {
    (pattern1) => { expansion1 };
    (pattern2) => { expansion2 };
}

// Repetition: zero or more, comma-separated
$($x:expr),*

// Repetition: one or more, comma-separated
$($x:expr),+
```

---

## Common Patterns

- Use `$($arg:tt)*` to forward arbitrary `format!`-style arguments through a wrapper macro, as `println!`-based logging macros do. See A Small Logging/Assertion-Style Macro above.
- Use recursion with a single-argument base case to process a repeated `$(...)` list one item at a time. See macro_rules! Basics above (the `max_of!` example).
- Use a macro instead of a function when the macro needs to `return` out of the caller, not out of itself. See A Small Logging/Assertion-Style Macro above (the `ensure!` example).

---

## Common Pitfalls

- A macro is not a text substitution: each `$x:expr` capture is matched and substituted as one indivisible unit, which avoids accidental operator-precedence bugs but also means you can't build a new identifier by pasting two `ident` fragments together without a separate helper macro or crate.
- Multiple statements in a macro expansion need to be wrapped in their own `{ }` block (see the double braces in the `max_of!` recursive arm above), or the macro can only be used in positions that accept a single expression or statement.
- Reaching for a macro to avoid writing out a generic function is usually the wrong tradeoff: a generic function is type-checked once, at its own definition, while a macro is only checked after expansion at every call site, so type errors show up later and at the wrong location. See Macro vs. Function above.

---

## Summary

`macro_rules!` matches on token trees using fragment specifiers like `expr`, `ident`,
and `tt`, and expands to code spliced directly into the call site, which is what lets
macros generate repeated code, accept a variable number of arguments, and affect the
caller's own control flow. A plain function is the right default; a macro earns its
place only when one of those three abilities is actually needed. Proc macros (the kind
behind `#[derive(...)]`) are a different mechanism entirely and are covered from the
caller's side in [Chapter 25](IX.25-third-party-library-reference.md).
