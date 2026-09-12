# Basic Syntax in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Hello World](#hello-world)
3. [Variables and Mutability](#variables-and-mutability)
4. [Functions](#functions)
5. [Comments](#comments)
6. [Control Flow](#control-flow)
7. [Operators](#operators)
8. [Blocks and Expressions](#blocks-and-expressions)
9. [Quick Reference](#quick-reference)
10. [Common Patterns](#common-patterns)
11. [Common Pitfalls](#common-pitfalls)

Scalar and compound types, integer overflow, casting, and constants vs. statics moved to
[Chapter 2: Data Types](I.2-data-types.md).

---

## Introduction

Rust is a systems programming language focused on safety, speed, and concurrency. This document covers the fundamental syntax elements that form the building blocks of every Rust program.

---

## Hello World

Every Rust program starts with a `main` function, which is the entry point of the program.

```rust
fn main() {
    println!("Hello, World!");
}
```

### Key Points

- `fn` declares a function
- `main` is the special entry point function
- `println!` is a macro (note the `!`) that prints to stdout
- Statements end with semicolons `;`
- Code blocks are enclosed in curly braces `{}`

### Compiling and Running

```bash
# Compile
rustc main.rs

# Run
./main
```

Or using Cargo (Rust's package manager):

```bash
# Create a new project
cargo new my_project

# Build
cargo build

# Run
cargo run
```

---

## Variables and Mutability

### Immutable Variables (Default)

By default, variables in Rust are immutable:

```rust
fn main() {
    let x = 5;
    println!("The value of x is: {}", x);

    // This would cause a compile error:
    // x = 6; // ERROR: cannot assign twice to immutable variable
}
```

### Mutable Variables

Use the `mut` keyword to make a variable mutable:

```rust
fn main() {
    let mut x = 5;
    println!("The value of x is: {}", x);

    x = 6; // This is allowed
    println!("The value of x is: {}", x);
}
```

### Variable Shadowing

You can declare a new variable with the same name, which "shadows" the previous one:

```rust
fn main() {
    let x = 5;
    let x = x + 1;  // Shadows the previous x

    {
        let x = x * 2;  // Shadows in inner scope
        println!("The value of x in inner scope is: {}", x); // 12
    }

    println!("The value of x is: {}", x); // 6
}
```

### Shadowing vs Mutability

Shadowing allows changing the type:

```rust
fn main() {
    let spaces = "   ";        // &str
    let spaces = spaces.len(); // usize - different type!

    // With mut, this wouldn't work:
    // let mut spaces = "   ";
    // spaces = spaces.len(); // ERROR: mismatched types
}
```

### Type Annotations

You can explicitly specify types:

```rust
fn main() {
    let x: i32 = 5;
    let y: f64 = 3.14;
    let z: bool = true;
    let s: &str = "hello";
    let c: char = 'a';
}
```

---

## Functions

### Basic Function Syntax

```rust
fn main() {
    greet();
    let result = add(5, 3);
    println!("5 + 3 = {}", result);
}

fn greet() {
    println!("Hello!");
}

fn add(x: i32, y: i32) -> i32 {
    x + y  // No semicolon = return expression
}
```

### Function Parameters

Parameters must have explicit type annotations:

```rust
fn print_labeled_measurement(value: i32, unit: &str) {
    println!("The measurement is: {} {}", value, unit);
}

fn main() {
    print_labeled_measurement(5, "meters");
}
```

### Return Values

Functions return the last expression or use `return` keyword:

```rust
// Implicit return (preferred for simple cases)
fn square(x: i32) -> i32 {
    x * x
}

// Explicit return
fn absolute(x: i32) -> i32 {
    if x < 0 {
        return -x;
    }
    x
}

// Multiple return points
fn divide(dividend: i32, divisor: i32) -> Option<i32> {
    if divisor == 0 {
        return None;
    }
    Some(dividend / divisor)
}

// Returning unit type (no meaningful return)
fn print_number(x: i32) {
    println!("{}", x);
    // implicitly returns ()
}

// Returning tuples
fn swap(a: i32, b: i32) -> (i32, i32) {
    (b, a)
}
```

### Function Pointers

A plain `fn` item is itself a value with a type, written `fn(i32) -> i32` here. Any
function whose signature matches can be passed where that type is expected, which is
what lets `do_twice` call whatever function it was handed rather than one hardcoded
function:

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn main() {
    let result = do_twice(add_one, 5);
    println!("{}", result); // 12
}
```

`fn(i32) -> i32` is a function pointer type: a plain pointer to compiled code, distinct
from a closure. Closures and the higher-order methods that accept them are covered in
[Chapter 17](VI.17-closures-and-higher-order-methods.md).

---

## Comments

### Line Comments

```rust
fn main() {
    // This is a line comment
    let x = 5; // This is an inline comment

    // Comments can span
    // multiple lines
    // like this
}
```

### Block Comments

```rust
fn main() {
    /* This is a block comment
       that spans multiple lines */

    let x = /* inline block comment */ 5;
}
```

### Documentation Comments

```rust
/// This is a documentation comment for the following item.
/// It supports **Markdown** formatting.
///
/// # Examples
///
/// ```
/// let result = my_crate::add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

//! This is a module-level documentation comment.
//! It documents the enclosing module or crate.
```

`///` attaches to the item that follows it and becomes that item's entry in the
generated docs (`cargo doc`). The fenced code block inside it is not just an example for
readers: `cargo test` compiles and runs it as a doctest, so a doc comment's examples stay
correct as the code changes. `//!` documents the item it's written inside rather than the
item after it, which is why it's used at the top of a module or crate file, where there
is nothing "after" to attach to.

---

## Control Flow

### if Expressions

```rust
fn main() {
    let number = 6;

    // Basic if
    if number < 5 {
        println!("less than 5");
    } else {
        println!("greater than or equal to 5");
    }

    // if-else if-else
    if number % 4 == 0 {
        println!("divisible by 4");
    } else if number % 3 == 0 {
        println!("divisible by 3");
    } else if number % 2 == 0 {
        println!("divisible by 2");
    } else {
        println!("not divisible by 4, 3, or 2");
    }

    // if as an expression
    let condition = true;
    let value = if condition { 5 } else { 6 };
    println!("value is: {}", value);
}
```

### loop (Infinite Loop)

```rust
fn main() {
    let mut counter = 0;

    // Basic loop
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;  // Return value from loop
        }
    };
    println!("Result: {}", result); // 20

    // Labeled loops for nested breaking
    'outer: loop {
        println!("Entered outer loop");
        'inner: loop {
            println!("Entered inner loop");
            break 'outer;  // Breaks out of outer loop
        }
    }
    println!("Exited outer loop");
}
```

A bare `break` only exits the loop it's written in, so from inside `'inner` it would
just return to `'outer` and loop again. Labeling a loop (`'outer:`, `'inner:`) gives
`break` and `continue` a target further out, which is the only way to exit a nested loop
directly.

### while Loop

```rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{}!", number);
        number -= 1;
    }

    println!("LIFTOFF!!!");
}
```

### for Loop

```rust
fn main() {
    // Iterating over a collection
    let a = [10, 20, 30, 40, 50];
    for element in a {
        println!("value: {}", element);
    }

    // Using a range
    for number in 1..4 {        // 1, 2, 3
        println!("{}", number);
    }

    for number in 1..=4 {       // 1, 2, 3, 4 (inclusive)
        println!("{}", number);
    }

    // Reverse range
    for number in (1..4).rev() {
        println!("{}", number); // 3, 2, 1
    }

    // Enumerate
    for (index, value) in a.iter().enumerate() {
        println!("Index {}: {}", index, value);
    }
}
```

---

## Operators

### Arithmetic Operators

```rust
fn main() {
    let a = 10;
    let b = 3;

    println!("a + b = {}", a + b);  // Addition: 13
    println!("a - b = {}", a - b);  // Subtraction: 7
    println!("a * b = {}", a * b);  // Multiplication: 30
    println!("a / b = {}", a / b);  // Integer division: 3
    println!("a % b = {}", a % b);  // Remainder: 1

    // Floating-point division
    let x = 10.0;
    let y = 3.0;
    println!("x / y = {}", x / y);  // 3.3333...
}
```

### Comparison Operators

```rust
fn main() {
    let a = 5;
    let b = 10;

    println!("a == b: {}", a == b); // Equal: false
    println!("a != b: {}", a != b); // Not equal: true
    println!("a < b: {}", a < b);   // Less than: true
    println!("a > b: {}", a > b);   // Greater than: false
    println!("a <= b: {}", a <= b); // Less than or equal: true
    println!("a >= b: {}", a >= b); // Greater than or equal: false
}
```

### Logical Operators

```rust
fn main() {
    let t = true;
    let f = false;

    println!("t && f: {}", t && f); // Logical AND: false
    println!("t || f: {}", t || f); // Logical OR: true
    println!("!t: {}", !t);         // Logical NOT: false

    // Short-circuit evaluation
    let result = false && expensive_function(); // expensive_function() not called
    let result = true || expensive_function();  // expensive_function() not called
}

fn expensive_function() -> bool {
    println!("This won't print if short-circuited");
    true
}
```

### Bitwise Operators

```rust
fn main() {
    let a: u8 = 0b1010;  // 10
    let b: u8 = 0b1100;  // 12

    println!("a & b = {:08b}", a & b);   // AND: 00001000 (8)
    println!("a | b = {:08b}", a | b);   // OR:  00001110 (14)
    println!("a ^ b = {:08b}", a ^ b);   // XOR: 00000110 (6)
    println!("!a = {:08b}", !a);          // NOT: 11110101 (245)
    println!("a << 2 = {:08b}", a << 2); // Left shift: 00101000 (40)
    println!("a >> 2 = {:08b}", a >> 2); // Right shift: 00000010 (2)
}
```

### Compound Assignment Operators

```rust
fn main() {
    let mut x = 10;

    x += 5;  // x = x + 5
    x -= 3;  // x = x - 3
    x *= 2;  // x = x * 2
    x /= 4;  // x = x / 4
    x %= 3;  // x = x % 3

    let mut bits: u8 = 0b1010;
    bits &= 0b1100;  // AND assignment
    bits |= 0b0001;  // OR assignment
    bits ^= 0b1111;  // XOR assignment
    bits <<= 1;      // Left shift assignment
    bits >>= 1;      // Right shift assignment
}
```

---

## Blocks and Expressions

### Everything is an Expression

In Rust, most things are expressions that return values:

```rust
fn main() {
    // Block expression
    let y = {
        let x = 3;
        x + 1  // No semicolon = this is the return value
    };
    println!("y = {}", y); // 4

    // if is an expression
    let condition = true;
    let number = if condition { 5 } else { 6 };

    // match is an expression
    let x = 1;
    let message = match x {
        1 => "one",
        2 => "two",
        _ => "other",
    };
}
```

### Statements vs Expressions

```rust
fn main() {
    // Statement: performs action, doesn't return value
    let x = 5;  // let statement

    // Expression: evaluates to a value
    let y = {
        let a = 1;
        let b = 2;
        a + b       // Expression (no semicolon)
    };

    // Adding semicolon turns expression into statement
    let z = {
        let a = 1;
        let b = 2;
        a + b;      // Now a statement, block returns ()
    };
    // z is now ()
}
```

### The Unit Type

The unit type `()` represents "no meaningful value":

```rust
fn main() {
    // Functions that don't return anything return ()
    let result: () = print_something();

    // Empty blocks return ()
    let empty = {};

    // Statements return ()
    let x = (let y = 5);  // ERROR: let is a statement
}

fn print_something() {
    println!("Hello");
    // Implicitly returns ()
}
```

### Early Returns and Diverging Functions

```rust
// Early return
fn check_positive(x: i32) -> i32 {
    if x < 0 {
        return 0;  // Early return
    }
    x  // Normal return
}

// Diverging function (never returns)
fn infinite() -> ! {
    loop {
        // This function never returns
    }
}

// panic! is also diverging
fn will_panic() -> ! {
    panic!("This function panics!");
}
```

`!` is the never type: it has no values, and it marks a function that can't return
normally, whether by looping forever or by panicking. That is different from returning
`()`, which does return, just with no useful value. Because a function that diverges
never produces a value at all, the compiler lets its call site stand in for any type,
which is why `panic!()` or a `loop {}` can appear in a branch that's otherwise expected
to return, say, `i32`.

---

## Quick Reference

### Operators

| Category | Operators |
|---|---|
| Arithmetic | `+` `-` `*` `/` `%` |
| Comparison | `==` `!=` `<` `>` `<=` `>=` |
| Logical | `&&` `\|\|` `!` |
| Bitwise | `&` `\|` `^` `!` `<<` `>>` |
| Compound assignment | `+=` `-=` `*=` `/=` `%=` `&=` `\|=` `^=` `<<=` `>>=` |

### Control flow forms

| Form | Notes |
|---|---|
| `if` / `else if` / `else` | Also usable as an expression: `let value = if condition { 5 } else { 6 };` |
| `loop` | Infinite loop; `break value` returns `value` from the loop |
| `while` | Loops while a condition holds |
| `for` | Iterates over a collection or a range (`1..4`, `1..=4`) |

---

## Common Patterns

- Shadowing lets you reuse a variable name while changing its type, which `mut` cannot
  do: `let spaces = "   "; let spaces = spaces.len();` goes from `&str` to `usize`.
- `if`, `loop`, and `match` are expressions, so a value can be produced directly from
  them instead of assigned inside each branch, as in
  `let result = loop { ... break counter * 2; };`.
- A function's last expression (no trailing semicolon) is its return value, and `return`
  is only needed for an early exit, as shown in `check_positive`.

---

## Common Pitfalls

- Assigning to a variable that wasn't declared `mut` is a compile error
  (`x = 6;` after `let x = 5;`).
- Adding a semicolon after the last expression in a block turns it into a statement, so
  the block evaluates to `()` instead of the value: compare `a + b` and `a + b;` in the
  "Statements vs Expressions" example.
- `let` is a statement, not an expression, so it can't be used where a value is expected
  (`let x = (let y = 5);` is an error).

---

## Summary

This document covered the fundamental syntax elements of Rust:

- Variables: immutable by default, use `mut` for mutability
- Functions: declared with `fn`, parameters need type annotations, return values use `->`
- Control flow: `if`, `loop`, `while`, `for` - all are expressions
- Operators: arithmetic, comparison, logical, and bitwise
- Expressions: most constructs in Rust are expressions that return values

Scalar and compound types, plus constants and statics, are covered in
[Chapter 2: Data Types](I.2-data-types.md).
