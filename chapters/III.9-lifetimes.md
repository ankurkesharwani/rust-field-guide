# Lifetimes in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [What Are Lifetimes?](#what-are-lifetimes)
3. [Lifetime Annotations](#lifetime-annotations)
4. [Lifetime Elision Rules](#lifetime-elision-rules)
5. [Lifetimes in Structs](#lifetimes-in-structs)
6. [Lifetime Bounds](#lifetime-bounds)
7. [The 'static Lifetime](#the-static-lifetime)
8. [Multiple Lifetimes](#multiple-lifetimes)
9. [Higher-Ranked Trait Bounds](#higher-ranked-trait-bounds)
10. [Quick Reference](#quick-reference)
11. [Common Patterns](#common-patterns)
12. [Common Pitfalls](#common-pitfalls)
13. [Summary](#summary)

---

## Introduction

Lifetimes are Rust's way of ensuring that references are always valid. Every reference in Rust has a lifetime - the scope for which that reference is valid. Most of the time, lifetimes are implicit and inferred, but sometimes you need to annotate them explicitly.

Key concepts:
- Lifetimes prevent dangling references
- They are a form of generic that describes reference validity
- The borrow checker uses lifetimes to ensure memory safety
- Lifetime annotations don't change how long values live

---

## What Are Lifetimes?

### The Dangling Reference Problem

```rust
fn main() {
    let r;                  // Declare r (no value yet)

    {
        let x = 5;          // x comes into scope
        r = &x;             // r references x
    }                       // x goes out of scope, is dropped

    // println!("{}", r);   // ERROR: r would be a dangling reference
}
```

### How the Borrow Checker Sees It

```rust
fn main() {
    let r;                  // ---------+-- 'a
                            //          |
    {                       //          |
        let x = 5;          // -+-- 'b  |
        r = &x;             //  |       |
    }                       // -+       |
                            //          |
    // r lives in 'a, but   //          |
    // references data from 'b         |
    // 'b is shorter than 'a, ERROR!   |
}                           // ---------+
```

### Valid Reference

```rust
fn main() {
    let x = 5;              // ---------+-- 'a
    let r = &x;             // -+-- 'b  |
                            //  |       |
    println!("{}", r);      //  |       |
                            // -+       |
}                           // ---------+

// 'b is contained within 'a, so reference is valid
```

---

## Lifetime Annotations

### Basic Syntax

Lifetime annotations start with an apostrophe and are conventionally short:

```rust
&'a i32        // A reference with lifetime 'a
&'a mut i32    // A mutable reference with lifetime 'a
```

### Function Signatures

```rust
// Without lifetime annotations (compiler infers)
fn first_word(s: &str) -> &str {
    // ...
}

// With explicit lifetime annotations
fn first_word<'a>(s: &'a str) -> &'a str {
    &s[..s.find(' ').unwrap_or(s.len())]
}

// The annotation says: the returned reference will be valid
// for at least as long as the input reference
```

### Why Annotations Are Needed

```rust
// This won't compile without lifetime annotations:
// fn longest(x: &str, y: &str) -> &str {  // ERROR: missing lifetime specifier
//     if x.len() > y.len() { x } else { y }
// }

// Rust can't tell if return value comes from x or y
// So it doesn't know which lifetime applies

fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("long string");
    let s2 = String::from("short");

    let result = longest(&s1, &s2);
    println!("Longest: {}", result);
}
```

### What Lifetime Annotations Mean

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// This signature means:
// 1. Both x and y must live at least as long as lifetime 'a
// 2. The returned reference will be valid for lifetime 'a
// 3. 'a is the overlap (intersection) of x's and y's lifetimes
```

### Lifetime Annotations in Action

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("long string");

    {
        let s2 = String::from("short");
        let result = longest(&s1, &s2);
        println!("{}", result);  // OK: result used while both s1 and s2 valid
    }

    // Can't use result here - it might reference s2 which is dropped
}

fn main_error() {
    let s1 = String::from("long string");
    let result;

    {
        let s2 = String::from("short");
        result = longest(&s1, &s2);
    }

    // println!("{}", result);  // ERROR: s2 doesn't live long enough
}
```

---

## Lifetime Elision Rules

The compiler applies three rules to infer lifetimes when you don't specify them:

### Rule 1: Input Lifetimes

Each reference parameter gets its own lifetime:

```rust
fn foo(x: &str)                  // fn foo<'a>(x: &'a str)
fn foo(x: &str, y: &str)         // fn foo<'a, 'b>(x: &'a str, y: &'b str)
```

### Rule 2: Single Input Lifetime

If there's exactly one input lifetime, it's assigned to all output lifetimes:

```rust
fn foo(x: &str) -> &str          // fn foo<'a>(x: &'a str) -> &'a str
```

### Rule 3: &self or &mut self

If there's a `&self` or `&mut self` parameter, its lifetime is assigned to all outputs:

```rust
impl Foo {
    fn method(&self, x: &str) -> &str {
        // Returns &str with lifetime of &self
    }
}
```

### When Elision Fails

```rust
// This fails elision - ambiguous
// fn longest(x: &str, y: &str) -> &str  // ERROR

// Must specify:
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

### Elision Examples

```rust
// These pairs are equivalent:

fn first_word(s: &str) -> &str { ... }
fn first_word<'a>(s: &'a str) -> &'a str { ... }

fn foo(x: &i32) { ... }
fn foo<'a>(x: &'a i32) { ... }

impl Foo {
    fn bar(&self) -> &str { ... }
}
impl Foo {
    fn bar<'a>(&'a self) -> &'a str { ... }
}
```

---

## Lifetimes in Structs

### References in Structs

```rust
// A struct that holds a reference needs a lifetime annotation
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().unwrap();

    let excerpt = ImportantExcerpt {
        part: first_sentence,
    };

    println!("{}", excerpt.part);
}
```

### What the Lifetime Means

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

// This means:
// - ImportantExcerpt contains a reference
// - The data being referenced must outlive the struct instance
// - An ImportantExcerpt cannot outlive the reference it holds
```

### Struct Methods with Lifetimes

```rust
struct Excerpt<'a> {
    text: &'a str,
}

impl<'a> Excerpt<'a> {
    // Elision rule 3: output lifetime is 'a (from &self)
    fn level(&self) -> i32 {
        3
    }

    // Returns reference with same lifetime as self
    fn announce_and_return(&self, announcement: &str) -> &str {
        println!("Attention: {}", announcement);
        self.text
    }

    // Explicit: return has different lifetime than announcement
    fn get_text(&self) -> &'a str {
        self.text
    }
}
```

### Multiple References in Structs

```rust
struct TwoRefs<'a, 'b> {
    first: &'a str,
    second: &'b str,
}

fn main() {
    let s1 = String::from("first");
    let s2 = String::from("second");

    let two = TwoRefs {
        first: &s1,
        second: &s2,
    };

    println!("{} {}", two.first, two.second);
}
```

---

## Lifetime Bounds

### On Generic Types

```rust
// T must outlive 'a
fn foo<'a, T: 'a>(x: &'a T) -> &'a T {
    x
}

// T must be 'static (live for entire program)
fn bar<T: 'static>(x: T) {
    // x owns T, which has no references, or only 'static references
}
```

### Combining with Trait Bounds

```rust
use std::fmt::Display;

// T must implement Display AND outlive 'a
fn longest_with_announcement<'a, T>(x: &'a str, y: &'a str, ann: T) -> &'a str
where
    T: Display,
{
    println!("Announcement: {}", ann);
    if x.len() > y.len() { x } else { y }
}
```

### The 'a: 'b Syntax

```rust
// 'a must outlive 'b
fn foo<'a: 'b, 'b>(x: &'a str, y: &'b str) -> &'b str {
    x  // OK: 'a outlives 'b, so &'a str can be used where &'b str is expected
}
```

---

## The 'static Lifetime

### What is 'static?

The `'static` lifetime means the reference can live for the entire program duration:

```rust
fn main() {
    // String literals have 'static lifetime
    let s: &'static str = "I live forever!";

    // They're stored in the binary, not the heap
}
```

### 'static References

```rust
// String literals
let s: &'static str = "hello";

// Leaked heap memory
let s: &'static str = Box::leak(Box::new(String::from("hello")));

// Static variables
static GREETING: &str = "Hello, world!";
```

### 'static as a Trait Bound

```rust
// T must be either owned or have only 'static references
fn send_to_thread<T: Send + 'static>(value: T) {
    std::thread::spawn(move || {
        // Use value here
        // Since T: 'static, we know value doesn't contain
        // references that might become invalid
    });
}
```

### Common Misconception

```rust
// 'static doesn't mean "lives forever"
// It means "CAN live as long as needed up to the whole program"

fn foo() -> &'static str {
    "hello"  // OK: string literal is 'static
}

// fn bar() -> &'static String {
//     let s = String::from("hello");
//     &s  // ERROR: s is dropped at end of function
// }
```

---

## Multiple Lifetimes

### When to Use Multiple Lifetimes

```rust
// Different inputs might have different lifetimes
struct MultiRef<'a, 'b> {
    x: &'a i32,
    y: &'b i32,
}

// When output lifetime differs from inputs
fn first<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    x  // Only returns reference to x
}
```

### Lifetime Relationships

```rust
// 'a and 'b are independent
fn independent<'a, 'b>(x: &'a str, y: &'b str) {
    println!("{} {}", x, y);
}

// 'a must outlive 'b
fn outlives<'a: 'b, 'b>(x: &'a str) -> &'b str {
    x  // 'a lives at least as long as 'b
}

// Same lifetime
fn same<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

### Practical Example

```rust
struct Context<'a> {
    data: &'a str,
}

struct Parser<'a, 'b> {
    context: &'a Context<'b>,
}

impl<'a, 'b> Parser<'a, 'b> {
    fn parse(&self) -> &'b str {
        self.context.data
    }
}

fn parse_context(context: &Context) -> &str {
    let parser = Parser { context };
    parser.parse()
}
```

Two lifetimes are needed here because two different borrows are in play: `'a` is how
long the reference to the `Context` itself is valid, while `'b` is how long the string
data behind it is valid. `parse()` returns `&'b str`, tied only to the data's lifetime,
so the result can keep being used even after the `Parser` (and its `'a` borrow) is no
longer needed, as `parse_context` relies on.

---

## Higher-Ranked Trait Bounds

### The Problem

```rust
// This works for a specific lifetime
fn call_with_ref<'a>(f: fn(&'a str)) {
    let s = String::from("hello");
    f(&s);
}

// But what if we need it to work for ANY lifetime?
```

`'a` on `call_with_ref` is chosen by the caller, before the function body runs. But `s`
is created inside the body, so its lifetime doesn't exist yet at the point where the
caller would have to pick `'a`. There's no single concrete lifetime the caller could
supply that would make `f(&s)` type-check, because `s`'s lifetime is local to this
function and different on every call.

### HRTB Syntax

A higher-ranked trait bound sidesteps this by not fixing `'a` at all: `for<'a> Fn(&'a
str)` says `f` must accept a `&str` of *any* lifetime, decided fresh each time it's
called, rather than one lifetime chosen up front by the caller of `call_with_ref`. This
is what actually lets `f` be called with `&s`, a reference whose lifetime only comes into
existence inside the function body.

```rust
// for<'a> means "for any lifetime 'a"
fn call_with_ref<F>(f: F)
where
    F: for<'a> Fn(&'a str),
{
    let s = String::from("hello");
    f(&s);
}

fn main() {
    call_with_ref(|s| println!("{}", s));
}
```

### Common HRTB Patterns

```rust
// Fn traits often use HRTB implicitly
fn apply<F>(f: F)
where
    F: Fn(&str) -> bool,  // Implicitly: for<'a> Fn(&'a str) -> bool
{
    let s = String::from("hello");
    println!("{}", f(&s));
}

// Explicit HRTB
fn apply_explicit<F>(f: F)
where
    F: for<'a> Fn(&'a str) -> bool,
{
    let s = String::from("hello");
    println!("{}", f(&s));
}
```

---

## Quick Reference

| Syntax | Meaning |
|--------|---------|
| `'a` | A lifetime named `'a` |
| `&'a T` | Reference to `T` valid for `'a` |
| `&'a mut T` | Mutable reference to `T` valid for `'a` |
| `T: 'a` | `T` must outlive `'a` |
| `'a: 'b` | `'a` must outlive `'b` |
| `'static` | Lives for the entire program |
| `for<'a>` | For any lifetime `'a` (HRTB) |

Elision rules, in order: each input reference gets its own lifetime; a single input
lifetime is applied to all outputs; a `&self`/`&mut self` lifetime is applied to all
outputs.

---

## Common Patterns

### Pattern 1: Return Input Reference

```rust
fn first<'a>(slice: &'a [i32]) -> &'a i32 {
    &slice[0]
}

fn main() {
    let nums = vec![1, 2, 3];
    let first = first(&nums);
    println!("{}", first);
}
```

### Pattern 2: Return Reference from One of Multiple Inputs

```rust
fn longer<'a>(x: &'a str, y: &str) -> &'a str {
    // Only x's lifetime matters for the return value
    x
}
```

### Pattern 3: Struct That Borrows

```rust
struct Borrowed<'a> {
    value: &'a str,
}

impl<'a> Borrowed<'a> {
    fn new(value: &'a str) -> Borrowed<'a> {
        Borrowed { value }
    }

    fn value(&self) -> &'a str {
        self.value
    }
}
```

### Pattern 4: Split Borrows

The borrow checker only tracks whole values, so it can't see that the two halves of a
slice, split at `mid`, never overlap; as far as it's concerned, returning two `&mut`
slices from one `&mut` slice looks like aliasing two mutable references to the same
data. `unsafe` is what lets this function assert the halves are actually disjoint
without going through the borrow checker's more conservative view. The standard
library's own `slice::split_at_mut` does exactly this internally.

```rust
fn split_at_mut<'a>(slice: &'a mut [i32], mid: usize) -> (&'a mut [i32], &'a mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

### Pattern 5: Builder with Lifetimes

```rust
struct Config<'a> {
    name: &'a str,
    value: i32,
}

struct ConfigBuilder<'a> {
    name: Option<&'a str>,
    value: Option<i32>,
}

impl<'a> ConfigBuilder<'a> {
    fn new() -> Self {
        ConfigBuilder { name: None, value: None }
    }

    fn name(mut self, name: &'a str) -> Self {
        self.name = Some(name);
        self
    }

    fn value(mut self, value: i32) -> Self {
        self.value = Some(value);
        self
    }

    fn build(self) -> Option<Config<'a>> {
        Some(Config {
            name: self.name?,
            value: self.value?,
        })
    }
}
```

---

## Common Pitfalls

### Error: "Missing Lifetime Specifier"

```rust
// ERROR
// fn foo(x: &str, y: &str) -> &str { x }

// FIX: Add lifetime annotations
fn foo<'a>(x: &'a str, y: &str) -> &'a str { x }
```

### Error: "Lifetime May Not Live Long Enough"

```rust
// ERROR
// fn bar<'a>(x: &'a str) -> &'static str { x }

// FIX: Match lifetimes or return owned data
fn bar<'a>(x: &'a str) -> &'a str { x }
// or
fn bar(x: &str) -> String { x.to_string() }
```

### Error: "Does Not Live Long Enough"

```rust
fn main() {
    let result;
    {
        let s = String::from("hello");
        result = &s;  // ERROR: s doesn't live long enough
    }
    // println!("{}", result);
}

// FIX: Extend the scope or use owned data
fn main() {
    let s = String::from("hello");
    let result = &s;
    println!("{}", result);
}
```

### Error: "Cannot Infer an Appropriate Lifetime"

```rust
struct Foo<'a> {
    x: &'a i32,
}

// ERROR: lifetime mismatch
// impl<'a> Foo<'a> {
//     fn new(x: &i32) -> Foo {
//         Foo { x }
//     }
// }

// FIX: Be explicit about lifetimes
impl<'a> Foo<'a> {
    fn new(x: &'a i32) -> Foo<'a> {
        Foo { x }
    }
}
```

### Error: "Borrowed Value Does Not Live Long Enough"

```rust
// ERROR
// fn longest_ref() -> &str {
//     let s1 = String::from("hello");
//     let s2 = String::from("world");
//     if s1.len() > s2.len() { &s1 } else { &s2 }
// }

// FIX: Return owned data
fn longest_owned() -> String {
    let s1 = String::from("hello");
    let s2 = String::from("world");
    if s1.len() > s2.len() { s1 } else { s2 }
}
```

### Strategy: When in Doubt

1. **Start without annotations** - let compiler tell you what's needed
2. **Add annotations one at a time** - fix errors incrementally
3. **Consider owned types** - sometimes avoiding references is simpler
4. **Use `'static` sparingly** - only for truly static data
5. **Read the error message** - Rust's messages are very helpful

---

## Summary

See the Quick Reference section above for lifetime syntax and the elision rules.

### When Explicit Lifetimes Are Needed

- Multiple input references with an output reference
- Structs containing references
- Complex trait bounds
- When the compiler can't infer the relationship on its own

### Best Practices

1. Prefer owned types when lifetime complexity grows
2. Use elision when the compiler allows it
3. Name lifetimes meaningfully (`'input`, `'output` instead of `'a`, `'b`)
4. Document lifetime requirements in public APIs
5. Trust compiler errors. They guide you to the solution
