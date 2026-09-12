# Data Types in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Scalar Types](#scalar-types)
3. [Integer Overflow and Checked Arithmetic](#integer-overflow-and-checked-arithmetic)
4. [Casting with `as`](#casting-with-as)
5. [Compound Types](#compound-types)
6. [Constants and Statics](#constants-and-statics)
7. [Quick Reference](#quick-reference)
8. [Common Patterns](#common-patterns)
9. [Common Pitfalls](#common-pitfalls)
10. [Summary](#summary)

---

## Introduction

Rust is statically typed: every value's type is known at compile time, either from an
annotation or from how the value is used. This chapter covers the scalar and compound
types built into the language, how Rust handles arithmetic that overflows those types,
and the difference between `const` and `static`.

---

## Scalar Types

Scalar types represent a single value.

### Integer Types

| Length  | Signed  | Unsigned |
|---------|---------|----------|
| 8-bit   | `i8`    | `u8`     |
| 16-bit  | `i16`   | `u16`    |
| 32-bit  | `i32`   | `u32`    |
| 64-bit  | `i64`   | `u64`    |
| 128-bit | `i128`  | `u128`   |
| arch    | `isize` | `usize`  |

```rust
fn main() {
    let a: i32 = -42;
    let b: u32 = 42;
    let c: isize = -100;  // Architecture-dependent size
    let d: usize = 100;   // Architecture-dependent size

    // Integer literals
    let decimal = 98_222;      // Decimal (underscore for readability)
    let hex = 0xff;            // Hexadecimal
    let octal = 0o77;          // Octal
    let binary = 0b1111_0000;  // Binary
    let byte = b'A';           // Byte (u8 only)
}
```

### Floating-Point Types

```rust
fn main() {
    let x = 2.0;      // f64 (default)
    let y: f32 = 3.0; // f32

    // Floating-point operations
    let sum = 5.0 + 10.0;
    let difference = 95.5 - 4.3;
    let product = 4.0 * 30.0;
    let quotient = 56.7 / 32.2;
}
```

### Boolean Type

```rust
fn main() {
    let t = true;
    let f: bool = false;

    // Boolean operations
    let and = true && false;  // false
    let or = true || false;   // true
    let not = !true;          // false
}
```

### Character Type

Characters in Rust are Unicode scalar values (4 bytes):

```rust
fn main() {
    let c = 'z';
    let z: char = 'ℤ';
    let heart_eyed_cat = '😻';
    let chinese = '中';

    println!("Size of char: {} bytes", std::mem::size_of::<char>()); // 4
}
```

---

## Integer Overflow and Checked Arithmetic

Every integer type has a fixed range: a `u8` holds `0..=255`, an `i32` holds
`-2147483648..=2147483647`, and so on. An operation that pushes the result outside that
range overflows, and Rust's behavior depends on how the binary was built.

```rust
fn main() {
    let x: u8 = std::hint::black_box(255);
    let y = x + 1;
    println!("{}", y);
}
```

In a debug build (`cargo run`, `cargo build`) this panics at runtime with `attempt to add
with overflow`. In a release build (`cargo run --release`, `cargo build --release`) the
same code does not panic; it silently wraps around using two's-complement arithmetic and
`y` becomes `0`. Debug builds add the overflow check because it is cheap and catches
bugs early; release builds skip it for speed, on the assumption that the check already
passed during testing. (`black_box` above just stops the compiler from computing `255 +
1` at compile time and rejecting the literal outright, so the addition genuinely happens
at runtime, the way it would if `x` came from user input.)

Relying on that debug/release difference is fragile, so the standard library gives every
integer type four families of methods that make the overflow behavior explicit and
consistent in both build profiles:

```rust
fn main() {
    let x: u8 = 255;

    let checked: Option<u8> = x.checked_add(1);        // None: overflow detected
    let wrapped: u8 = x.wrapping_add(1);                // 0: wraps around
    let saturated: u8 = x.saturating_add(1);            // 255: clamps to the max
    let (value, overflowed): (u8, bool) = x.overflowing_add(1); // (0, true)

    println!("{:?} {} {} {} {}", checked, wrapped, saturated, value, overflowed);
    // None 0 255 0 true
}
```

- `checked_*` returns `Option<T>`: `Some(result)` on success, `None` on overflow. Use it
  when overflow means something went wrong and the caller needs to decide what to do.
- `wrapping_*` always returns `T`, wrapping around on overflow. This makes the
  release-mode behavior explicit and portable, instead of relying on which profile
  happens to be active.
- `saturating_*` always returns `T`, clamped to the type's minimum or maximum instead of
  wrapping. Useful when a value like a progress percentage or a volume level should stop
  at a boundary rather than wrap back around.
- `overflowing_*` returns `(T, bool)`: the wrapped result and whether it overflowed. This
  is what the other three are built on internally, and is useful when you need both the
  value and the flag.

Every arithmetic operator has all four variants: `checked_add`/`checked_sub`/
`checked_mul`/`checked_div`, and likewise for `wrapping_`, `saturating_`, and
`overflowing_`.

---

## Casting with `as`

`as` converts a value from one primitive type to another. Unlike the `checked_*`/
`wrapping_*`/`saturating_*` methods above, `as` never fails and never panics; it always
produces a value of the target type, using a fixed rule for what to do when the source
value doesn't fit.

```rust
fn main() {
    let big: i32 = 300;
    let a: u8 = big as u8;          // 44: truncates to the low 8 bits (300 % 256)

    let mid: i32 = 200;
    let b: i8 = mid as i8;          // -56: same bits as u8, reinterpreted as signed

    let c: u32 = (-1i32) as u32;    // 4294967295: two's-complement bit pattern reused

    let d: u8 = 3.99_f64 as u8;     // 3: float-to-int truncates toward zero
    let e: u8 = 300.0_f64 as u8;    // 255: out-of-range float saturates to the max
    let f: i8 = (-300.0_f64) as i8; // -128: out-of-range float saturates to the min

    println!("{} {} {} {} {} {}", a, b, c, d, e, f);
}
```

Two different rules are at work here:

- **Integer-to-integer casts truncate.** Rust keeps the low-order bits of the source
  value and reinterprets them as the target type. Casting between a signed and unsigned
  type of the same width (like `i32` to `u32`) doesn't change any bits at all; it only
  changes how those bits are interpreted, which is why `-1i32 as u32` becomes
  `u32::MAX` rather than an error.
- **Float-to-integer casts saturate.** If the float is outside the target integer's
  range (including `NaN`, which saturates to `0`), the cast clamps to the target type's
  minimum or maximum instead of wrapping. This has been the behavior since Rust 1.45;
  earlier versions produced unspecified values for out-of-range float casts, which was
  considered a footgun.

Because `as` silently discards information instead of reporting a problem, it's best
suited to conversions you know are safe (like `u8 as u32`, which always fits) or where
lossy truncation is exactly what you want. When a conversion should be validated or can
fail, use `TryFrom`/`TryInto` instead; see
[Chapter 14: Type Conversions](IV.14-type-conversions.md) for the trait-based
alternatives to `as`.

---

## Compound Types

Compound types group multiple values into one type.

### Tuples

Fixed-length, heterogeneous collection:

```rust
fn main() {
    // Creating a tuple
    let tup: (i32, f64, u8) = (500, 6.4, 1);

    // Destructuring
    let (x, y, z) = tup;
    println!("y is: {}", y);

    // Accessing by index
    let five_hundred = tup.0;
    let six_point_four = tup.1;
    let one = tup.2;

    // Unit tuple (empty tuple)
    let unit: () = ();
}
```

### Arrays

Fixed-length, homogeneous collection (stored on stack):

```rust
fn main() {
    // Array declaration
    let a = [1, 2, 3, 4, 5];
    let b: [i32; 5] = [1, 2, 3, 4, 5];

    // Initialize with same value
    let c = [3; 5]; // [3, 3, 3, 3, 3]

    // Accessing elements
    let first = a[0];
    let second = a[1];

    // Array length
    let len = a.len();

    // Iterating
    for element in a.iter() {
        println!("{}", element);
    }

    // Slices
    let slice = &a[1..3]; // [2, 3]
}
```

---

## Constants and Statics

### Constants

Constants are always immutable and must have their type annotated:

```rust
const MAX_POINTS: u32 = 100_000;
const PI: f64 = 3.14159265358979323846;
const GREETING: &str = "Hello, World!";

fn main() {
    println!("Max points: {}", MAX_POINTS);
    println!("Pi: {}", PI);
    println!("{}", GREETING);
}
```

### Key Properties of Constants

- Must be annotated with a type
- Can be declared in any scope, including global
- Can only be set to a constant expression (computed at compile time)
- Valid for the entire time a program runs
- Naming convention: SCREAMING_SNAKE_CASE

### Static Variables

Static variables have a fixed address in memory:

```rust
static LANGUAGE: &str = "Rust";
static mut COUNTER: u32 = 0;  // Mutable static (unsafe to access)

fn main() {
    println!("Language: {}", LANGUAGE);

    // Accessing mutable statics is unsafe
    unsafe {
        COUNTER += 1;
        println!("Counter: {}", COUNTER);
    }
}
```

---

## Quick Reference

### Scalar types

| Type | Category | Notes |
|---|---|---|
| `i8`...`i128`, `isize` | Signed integer | `isize` matches pointer width |
| `u8`...`u128`, `usize` | Unsigned integer | `usize` matches pointer width |
| `f32`, `f64` | Floating point | `f64` is the default |
| `bool` | Boolean | `true` / `false` |
| `char` | Character | 4-byte Unicode scalar value |

### `const` vs. `static`

| Feature | `const` | `static` |
|---------|---------|----------|
| Memory | Inlined at use site | Single memory location |
| Mutability | Never mutable | Can be mutable (unsafe) |
| Lifetime | None (inlined) | `'static` |
| Address | No fixed address | Fixed address |

---

## Common Patterns

- Default to `i32` for integers and `f64` for floats unless a specific reason (a field
  that must match a file format's width, or values that need to index a collection)
  calls for a different size. See Integer Types and Floating-Point Types above.
- Use `usize` for anything that indexes or sizes a collection (`.len()`, array/slice
  indices) since that's what the standard library uses for those positions. See Integer
  Types above.
- Reach for `checked_*` when overflow signals a bug the caller must handle, and
  `saturating_*` when clamping to a boundary (a percentage, a volume level) is the
  actually-correct behavior rather than an error. See Integer Overflow and Checked
  Arithmetic above.
- Pull a repeated literal into a `const` once it appears more than once or once its
  meaning isn't obvious from the value alone. See Constants and Statics above.

---

## Common Pitfalls

- A debug build panics on integer overflow, but a release build wraps silently. Code
  that only ever ran in `cargo run` can still overflow unnoticed once shipped as a
  release binary. See Integer Overflow and Checked Arithmetic above.
- `as` never fails: casting `300` to `u8` doesn't error, it truncates to `44`. If the
  value should be validated instead of silently altered, use `TryFrom`/`TryInto`. See
  Casting with `as` above.
- Accessing a `static mut` requires an `unsafe` block because nothing stops two threads
  from reading and writing it at the same time; reach for `AtomicU32`/`Mutex` instead of
  a mutable static in real code. See Static Variables above.
- `f32`/`f64` don't implement `Eq` and can't be used as `HashMap`/`HashSet` keys or
  compared with `==` reliably (`NaN != NaN`), because floating-point equality isn't a
  total order. See [Chapter 15: Common Derivable Traits](IV.15-common-derivable-traits-quick-reference.md#partialeq-and-eq)
  for what `Eq` requires.

---

## Summary

This chapter covered Rust's scalar types (integers, floats, booleans, characters) and
compound types (tuples, arrays), along with `const` and `static` for compile-time and
fixed-address values. It also covered what happens when integer arithmetic overflows
(a panic in debug builds, a silent wrap in release builds) and the `checked`/`wrapping`/
`saturating`/`overflowing` methods for handling that explicitly, plus the truncating and
saturating rules `as` uses to convert between primitive types. See
[Chapter 14: Type Conversions](IV.14-type-conversions.md) for the related trait-based
conversions (`From`/`Into`/`TryFrom`), which validate or fail instead of silently
truncating.
