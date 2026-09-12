# Traits and Trait Implementations in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Defining Traits](#defining-traits)
3. [Implementing Traits](#implementing-traits)
4. [Default Implementations](#default-implementations)
5. [Traits as Parameters](#traits-as-parameters)
6. [Returning Traits](#returning-traits)
7. [Trait Bounds](#trait-bounds)
8. [Associated Types](#associated-types)
9. [Supertraits](#supertraits)
10. [Operator Overloading](#operator-overloading)
11. [Common Standard Library Traits](#common-standard-library-traits)
12. [Advanced Trait Patterns](#advanced-trait-patterns)
13. [Quick Reference](#quick-reference)
14. [Common Patterns](#common-patterns)
15. [Common Pitfalls](#common-pitfalls)
16. [Summary](#summary)

---

## Introduction

Traits define shared behavior across types. They're similar to interfaces in other languages but more powerful. Traits enable:

- **Polymorphism**: Different types implementing the same interface
- **Code reuse**: Default implementations shared across types
- **Generics constraints**: Specifying what capabilities a type must have
- **Operator overloading**: Custom behavior for operators

---

## Defining Traits

### Basic Trait Definition

```rust
trait Summary {
    fn summarize(&self) -> String;
}
```

### Trait with Multiple Methods

```rust
trait Animal {
    fn name(&self) -> &str;
    fn speak(&self) -> String;
    fn number_of_legs(&self) -> u32;
}
```

### Traits with Associated Constants

```rust
trait Greet {
    const GREETING: &'static str;

    fn greet(&self) -> String {
        format!("{}, I am here!", Self::GREETING)
    }
}

struct Friendly;

impl Greet for Friendly {
    const GREETING: &'static str = "Hello";
}

fn main() {
    let f = Friendly;
    println!("{}", f.greet());  // "Hello, I am here!"
}
```

`GREETING` has no value in the trait itself, only a type, so every implementor must
supply one; `Self::GREETING` inside the default method resolves to whichever value the
concrete implementing type provided. This is the same shape as a required method, just
for a constant instead of a function.

---

## Implementing Traits

### Basic Implementation

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct NewsArticle {
    headline: String,
    author: String,
    content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {}", self.headline, self.author)
    }
}

struct Tweet {
    username: String,
    content: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.username, self.content)
    }
}

fn main() {
    let article = NewsArticle {
        headline: String::from("Breaking News!"),
        author: String::from("Jane Doe"),
        content: String::from("..."),
    };

    let tweet = Tweet {
        username: String::from("rust_lang"),
        content: String::from("Rust is awesome!"),
    };

    println!("{}", article.summarize());
    println!("{}", tweet.summarize());
}
```

### Implementing External Traits

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{}", p);  // (1, 2)
}
```

`fmt::Display` is a trait defined in the standard library, and `Point` is a type defined
here; implementing an external trait for a local type is always allowed. `println!("{}",
...)` works for any type that implements `Display` because the macro calls the trait's
`fmt` method under the hood, which is why writing this `impl` is what makes `{}`
formatting available for `Point` at all.

### Orphan Rule

You can only implement a trait for a type if either the trait or the type is local to
your crate. Without this rule, two unrelated crates could each implement the same
external trait for the same external type in different, incompatible ways; if your
program depended on both crates, the compiler would have no way to know which
implementation to use. Requiring at least one side (trait or type) to be local guarantees
only one crate could ever have written the `impl` in the first place:

```rust
// In your crate:

// OK: Your trait on external type
trait MyTrait {
    fn do_thing(&self);
}

impl MyTrait for String {
    fn do_thing(&self) {
        println!("{}", self);
    }
}

// OK: External trait on your type
struct MyType;

impl std::fmt::Display for MyType {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "MyType")
    }
}

// NOT OK: External trait on external type
// impl std::fmt::Display for String { }  // ERROR: orphan rule
```

---

## Default Implementations

### Providing Defaults

```rust
trait Summary {
    fn summarize_author(&self) -> String;

    // Default implementation using another method
    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}

struct Tweet {
    username: String,
    content: String,
}

impl Summary for Tweet {
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
    // Uses default summarize()
}

fn main() {
    let tweet = Tweet {
        username: String::from("rust_lang"),
        content: String::from("Hello!"),
    };

    println!("{}", tweet.summarize());
    // "(Read more from @rust_lang...)"
}
```

### Overriding Defaults

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}

struct Article {
    title: String,
}

impl Summary for Article {
    // Override the default
    fn summarize(&self) -> String {
        format!("Article: {}", self.title)
    }
}

struct PlainText;

impl Summary for PlainText {
    // Uses default - empty impl block
}

fn main() {
    let article = Article { title: String::from("News") };
    let plain = PlainText;

    println!("{}", article.summarize());  // "Article: News"
    println!("{}", plain.summarize());    // "(Read more...)"
}
```

---

## Traits as Parameters

### impl Trait Syntax

```rust
trait Summary {
    fn summarize(&self) -> String;
}

// Accept any type that implements Summary
fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

// Multiple parameters
fn notify_twice(item1: &impl Summary, item2: &impl Summary) {
    println!("1: {}", item1.summarize());
    println!("2: {}", item2.summarize());
}
```

`impl Summary` here isn't a trait object: the compiler still knows the concrete type at
each call site and generates a specialized version of `notify` for it (monomorphization),
the same as it would for a generic function. The `impl Trait` spelling is just sugar for
the generic form below; there's no dynamic dispatch and no heap allocation involved.

### Trait Bound Syntax

```rust
// Equivalent to impl Trait
fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}

// Force same type
fn notify_same<T: Summary>(item1: &T, item2: &T) {
    println!("1: {}", item1.summarize());
    println!("2: {}", item2.summarize());
}

trait Summary {
    fn summarize(&self) -> String;
}
```

### Multiple Trait Bounds

```rust
use std::fmt::{Display, Debug};

// With + syntax
fn notify(item: &(impl Summary + Display)) {
    println!("{}: {}", item, item.summarize());
}

// With trait bound syntax
fn notify_generic<T: Summary + Display>(item: &T) {
    println!("{}: {}", item, item.summarize());
}

trait Summary {
    fn summarize(&self) -> String;
}
```

### Where Clauses

```rust
use std::fmt::{Display, Debug};

// Complex bounds with where clause
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // Implementation
    0
}

// Compare to inline bounds (harder to read):
// fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32
```

---

## Returning Traits

### impl Trait in Return Position

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Tweet {
    content: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        self.content.clone()
    }
}

fn create_summary() -> impl Summary {
    Tweet {
        content: String::from("Hello!"),
    }
}

fn main() {
    let summary = create_summary();
    println!("{}", summary.summarize());
}
```

`impl Summary` in return position hides the concrete return type from the caller (they
only know it implements `Summary`) while the compiler still tracks the real type
internally. It's static dispatch, just like `impl Trait` in parameter position, which is
exactly why the next section's limitation exists.

### Limitation: Single Concrete Type

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Tweet { content: String }
struct Article { title: String }

impl Summary for Tweet {
    fn summarize(&self) -> String { self.content.clone() }
}

impl Summary for Article {
    fn summarize(&self) -> String { self.title.clone() }
}

// ERROR: Can't return different types
// fn create_summary(is_tweet: bool) -> impl Summary {
//     if is_tweet {
//         Tweet { content: String::from("hi") }
//     } else {
//         Article { title: String::from("News") }
//     }
// }

// Solution: Use trait objects (dynamic dispatch)
fn create_summary(is_tweet: bool) -> Box<dyn Summary> {
    if is_tweet {
        Box::new(Tweet { content: String::from("hi") })
    } else {
        Box::new(Article { title: String::from("News") })
    }
}
```

`impl Trait` compiles to one concrete return type per function, so a function that
returns a different type down different branches can't use it. `dyn Summary` names the
trait itself as a type, standing in for "some type that implements `Summary`, decided at
runtime." Because different implementors of `Summary` can have different sizes, `dyn
Summary` has no size known at compile time and can't be returned or stored directly; it
has to sit behind a pointer, here `Box`, which gives it a fixed-size home on the heap.
Calls to `summarize()` on the `Box<dyn Summary>` are resolved at runtime through a vtable
(dynamic dispatch) rather than being generated per concrete type at compile time
(static dispatch), which is what lets one function return either `Tweet` or `Article`
behind the same return type.

---

## Trait Bounds

### Conditional Method Implementation

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Pair { x, y }
    }
}

// Only implement for types that are Display + PartialOrd
impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

### Blanket Implementations

```rust
use std::fmt::Display;

trait Printable {
    fn print(&self);
}

// Implement for ALL types that implement Display
impl<T: Display> Printable for T {
    fn print(&self) {
        println!("{}", self);
    }
}

fn main() {
    5.print();
    "hello".print();
    3.14.print();
}
```

`impl<T: Display> Printable for T` implements `Printable` once, for every type `T` that
already implements `Display`, rather than writing a separate `impl Printable for i32`,
`impl Printable for &str`, and so on. Any type that implements `Display` gets `print()`
for free, including types defined after this blanket impl or in other crates, as long as
they satisfy the bound. The orphan rule still applies to the blanket impl itself: you can
only write it if `Printable` is a local trait.

### Blanket Implementations in std

```rust
// From the standard library:
// impl<T: Display> ToString for T {
//     fn to_string(&self) -> String {
//         // ...
//     }
// }

fn main() {
    // Every Display type gets to_string() for free
    let s: String = 5.to_string();
    let s: String = "hello".to_string();
}
```

---

## Associated Types

### Basic Associated Types

```rust
trait Iterator {
    type Item;  // Associated type

    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;  // Specify the associated type

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}
```

`Item` isn't a parameter the caller chooses, like `T` in a generic trait would be; it's a
type the implementor pins down once, and every other item in the trait (here, `next`'s
return type) refers back to it as `Self::Item`. Code that's generic over `I: Iterator`
can write `I::Item` to name whatever type that particular iterator produces, without
needing a second generic parameter for it.

### Associated Types vs Generics

```rust
// With generics: can implement multiple times
trait ConvertTo<T> {
    fn convert(&self) -> T;
}

struct MyNum(i32);

impl ConvertTo<i32> for MyNum {
    fn convert(&self) -> i32 { self.0 }
}

impl ConvertTo<f64> for MyNum {
    fn convert(&self) -> f64 { self.0 as f64 }
}

// With associated type: can only implement once
trait ToNumber {
    type Number;
    fn to_number(&self) -> Self::Number;
}

impl ToNumber for MyNum {
    type Number = i32;
    fn to_number(&self) -> Self::Number { self.0 }
}

// Can't implement again with different type
```

### Default Associated Types

`Rhs = Self` is a default type parameter: if an `impl` doesn't say otherwise, `Rhs` is
the same type as whatever implements `Add`, which is why `impl Add for Point` below can
add two `Point`s without naming `Rhs` at all. The second `impl` overrides the default so
`Point` can also be added to a plain tuple.

```rust
trait Add<Rhs = Self> {  // Default type parameter
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}

struct Point {
    x: i32,
    y: i32,
}

// Using default Rhs = Self
impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

// Custom Rhs
impl Add<(i32, i32)> for Point {
    type Output = Point;

    fn add(self, (dx, dy): (i32, i32)) -> Point {
        Point {
            x: self.x + dx,
            y: self.y + dy,
        }
    }
}
```

---

## Supertraits

### Requiring Other Traits

```rust
use std::fmt::Display;

// OutlinePrint requires Display
// A supertrait bound: any type implementing OutlinePrint must also implement Display.
// That guarantee is what lets the default method below call `self.to_string()`, a
// method that only exists because Display provides a blanket ToString impl.
trait OutlinePrint: Display {
    fn outline_print(&self) {
        let output = self.to_string();  // Can use Display methods
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("* {} *", output);
        println!("{}", "*".repeat(len + 4));
    }
}

struct Point {
    x: i32,
    y: i32,
}

impl Display for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

impl OutlinePrint for Point {}

fn main() {
    let p = Point { x: 1, y: 2 };
    p.outline_print();
}
```

### Multiple Supertraits

```rust
use std::fmt::{Display, Debug};

trait Printable: Display + Debug {
    fn print_all(&self) {
        println!("Display: {}", self);
        println!("Debug: {:?}", self);
    }
}
```

---

## Operator Overloading

Operators like `+` aren't special-cased by the compiler for user types; `p1 + p2` is
sugar for `p1.add(p2)`, where `add` comes from the `std::ops::Add` trait. Implementing
the trait for your own type is what makes the operator syntax available for it.

### Add Trait

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let p3 = p1 + p2;
    println!("{:?}", p3);  // Point { x: 4, y: 6 }
}
```

### Other Operator Traits

```rust
use std::ops::{Add, Sub, Mul, Div, Neg};

#[derive(Debug, Clone, Copy, PartialEq)]
struct Complex {
    real: f64,
    imag: f64,
}

impl Add for Complex {
    type Output = Self;
    fn add(self, other: Self) -> Self {
        Complex {
            real: self.real + other.real,
            imag: self.imag + other.imag,
        }
    }
}

impl Sub for Complex {
    type Output = Self;
    fn sub(self, other: Self) -> Self {
        Complex {
            real: self.real - other.real,
            imag: self.imag - other.imag,
        }
    }
}

impl Mul for Complex {
    type Output = Self;
    fn mul(self, other: Self) -> Self {
        Complex {
            real: self.real * other.real - self.imag * other.imag,
            imag: self.real * other.imag + self.imag * other.real,
        }
    }
}

impl Neg for Complex {
    type Output = Self;
    fn neg(self) -> Self {
        Complex {
            real: -self.real,
            imag: -self.imag,
        }
    }
}
```

### Index Traits

`matrix[(0, 1)]` desugars to `*matrix.index((0, 1))`, and the assignment form to
`*matrix.index_mut((1, 0)) = 10`. `Index::index` returns a reference rather than an
owned value because indexing normally borrows from the collection instead of moving out
of it; `IndexMut` additionally requires the collection to be borrowed mutably, which is
what allows the assignment on the last line below.

```rust
use std::ops::{Index, IndexMut};

struct Matrix {
    data: Vec<Vec<i32>>,
}

impl Index<(usize, usize)> for Matrix {
    type Output = i32;

    fn index(&self, (row, col): (usize, usize)) -> &Self::Output {
        &self.data[row][col]
    }
}

impl IndexMut<(usize, usize)> for Matrix {
    fn index_mut(&mut self, (row, col): (usize, usize)) -> &mut Self::Output {
        &mut self.data[row][col]
    }
}

fn main() {
    let mut matrix = Matrix {
        data: vec![vec![1, 2], vec![3, 4]],
    };

    println!("{}", matrix[(0, 1)]);  // 2
    matrix[(1, 0)] = 10;
    println!("{}", matrix[(1, 0)]);  // 10
}
```

---

## Common Standard Library Traits

### Display and Debug

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

// For user-facing output
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

// For debugging
impl fmt::Debug for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{}", p);   // Display: (1, 2)
    println!("{:?}", p); // Debug: Point { x: 1, y: 2 }
}
```

`Display` and `Debug` serve different audiences and that's why Rust keeps them as two
traits instead of one: `Display` is for output a user of the program should see, so
there's no blanket way to derive it (the compiler can't know what a user-facing format
should look like), while `Debug` is for the developer inspecting a value, so
`#[derive(Debug)]` can generate a reasonable default by just listing the fields.

### Clone and Copy

```rust
// Clone: explicit duplication
#[derive(Clone)]
struct Expensive {
    data: Vec<i32>,
}

// Copy: implicit bitwise copy (requires Clone)
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let e1 = Expensive { data: vec![1, 2, 3] };
    let e2 = e1.clone();  // Explicit clone

    let p1 = Point { x: 1, y: 2 };
    let p2 = p1;  // Implicit copy
    println!("{:?}", p1);  // Still valid
}
```

`Expensive` holds a `Vec`, which owns a heap allocation, so duplicating it has a real
cost; Rust makes that cost visible by requiring the explicit `.clone()` call rather than
letting `let e2 = e1;` silently duplicate it (that line would instead move `e1`, making
it unusable afterward). `Point` holds only `i32` fields, so a bitwise copy is cheap and
safe, and `Copy` (which requires `Clone`, since anything bitwise-copyable can trivially
be explicitly cloned too) makes `let p2 = p1;` copy instead of move, which is why `p1`
is still valid on the last line. A type can only derive `Copy` if all of its fields are
`Copy`.

### PartialEq, Eq, PartialOrd, Ord

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Person {
    age: u32,
    name: String,
}

fn main() {
    let p1 = Person { age: 25, name: String::from("Alice") };
    let p2 = Person { age: 30, name: String::from("Bob") };

    println!("{}", p1 == p2);  // false
    println!("{}", p1 < p2);   // true (compares by age first)

    let mut people = vec![p2, p1];
    people.sort();  // Requires Ord
}
```

Deriving these traits on a struct compares (or orders) fields in declaration order, so
`Person`'s derived `Ord` compares `age` first and only falls back to `name` if two people
have the same age, matching the field order above. `PartialEq`/`PartialOrd` allow
comparisons that can fail to produce an answer (as `f64`'s `NaN` does, which is neither
equal to nor less than anything, including itself); `Eq`/`Ord` are the stricter versions
that promise a total, always-defined comparison, which is why floating-point types
implement only the "partial" pair and why a `HashMap` key must implement `Eq` alongside
`Hash`.

### Default

```rust
#[derive(Default, Debug)]
struct Config {
    debug: bool,
    verbose: bool,
    max_size: usize,
}

// Custom implementation
struct Counter {
    count: i32,
}

impl Default for Counter {
    fn default() -> Self {
        Counter { count: 1 }  // Start at 1, not 0
    }
}

fn main() {
    let config = Config::default();
    println!("{:?}", config);

    let counter = Counter::default();
    println!("{}", counter.count);  // 1
}
```

### From and Into

```rust
struct Millimeters(u32);
struct Meters(u32);

impl From<Meters> for Millimeters {
    fn from(m: Meters) -> Self {
        Millimeters(m.0 * 1000)
    }
}

fn main() {
    let m = Meters(2);

    // Using From
    let mm = Millimeters::from(m);

    // Using Into (automatic from From)
    let m2 = Meters(3);
    let mm2: Millimeters = m2.into();

    println!("{}mm", mm.0);   // 2000mm
    println!("{}mm", mm2.0);  // 3000mm
}
```

Only `From` is implemented here; `Into` comes for free from a blanket implementation in
the standard library (`impl<T, U: From<T>> Into<U> for T`), so implementing `From<Meters>
for Millimeters` also makes `meters.into()` work anywhere the target type can be
inferred. That's why the convention is to implement `From`, not `Into`, directly.

### TryFrom and TryInto

```rust
use std::convert::TryFrom;

struct EvenNumber(i32);

impl TryFrom<i32> for EvenNumber {
    type Error = &'static str;

    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value % 2 == 0 {
            Ok(EvenNumber(value))
        } else {
            Err("Value is not even")
        }
    }
}

fn main() {
    let even = EvenNumber::try_from(4);
    let odd = EvenNumber::try_from(5);

    println!("{:?}", even.map(|e| e.0));  // Ok(4)
    println!("{:?}", odd);                 // Err("Value is not even")
}
```

### AsRef and AsMut

```rust
fn print_length<T: AsRef<str>>(s: T) {
    println!("Length: {}", s.as_ref().len());
}

fn main() {
    print_length("hello");           // &str
    print_length(String::from("hi")); // String
}
```

`AsRef<str>` lets `print_length` accept anything that can produce a cheap `&str` view of
itself, `&str` and `String` alike, without committing to one concrete parameter type or
forcing a conversion that allocates. It's the trait bound of choice for "give me
something string-like" APIs, the same role `AsRef<Path>` plays for file paths.

### Deref and DerefMut

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(value: T) -> MyBox<T> {
        MyBox(value)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}

fn main() {
    let x = MyBox::new(5);
    assert_eq!(5, *x);  // Deref coercion

    let s = MyBox::new(String::from("hello"));
    hello(&s);  // Deref coercion: &MyBox<String> -> &String -> &str
}

fn hello(s: &str) {
    println!("Hello, {}!", s);
}
```

`*x` works because `Deref::deref` gives the compiler a `&T` to dereference; this is the
same mechanism the standard library's own smart pointers (`Box`, `Rc`, `String`) rely on.
Deref coercion goes further: when a function expects `&str` and you pass `&MyBox<String>`,
the compiler inserts repeated `deref()` calls automatically, `&MyBox<String> ->
&String -> &str`, which is why `hello(&s)` compiles without an explicit conversion.

---

## Advanced Trait Patterns

### Trait Objects (Dynamic Dispatch)

```rust
trait Draw {
    fn draw(&self);
}

struct Circle { radius: f64 }
struct Square { side: f64 }

impl Draw for Circle {
    fn draw(&self) { println!("Drawing circle with radius {}", self.radius); }
}

impl Draw for Square {
    fn draw(&self) { println!("Drawing square with side {}", self.side); }
}

fn main() {
    // Trait object: Box<dyn Draw>
    let shapes: Vec<Box<dyn Draw>> = vec![
        Box::new(Circle { radius: 1.0 }),
        Box::new(Square { side: 2.0 }),
    ];

    for shape in shapes {
        shape.draw();
    }
}
```

Each `Box<dyn Draw>` is a fat pointer: one pointer to the value's data on the heap, and
one to a vtable of function pointers for that concrete type's implementation of `Draw`.
`shape.draw()` looks up `draw` in the vtable at runtime instead of the compiler picking
one fixed function at compile time, which is exactly what lets the same `Vec` hold both
`Circle`s and `Square`s.

### Object Safety

```rust
// Object-safe trait (can be used as trait object)
trait Draw {
    fn draw(&self);
}

// NOT object-safe (has Self in return type)
trait Clone2 {
    fn clone2(&self) -> Self;  // Can't use as dyn Clone2
}

// NOT object-safe (has generic method)
trait Serialize {
    fn serialize<T>(&self, target: T);  // Can't use as dyn Serialize
}
```

A vtable entry has to be one fixed function pointer, valid no matter which concrete type
is behind the `dyn Trait`. A method returning `Self` breaks that: `dyn Clone2`'s vtable
would need a return size that depends on the caller's concrete type, which isn't known
once the type has been erased to `dyn Clone2`. A generic method breaks it the same way
`impl Trait` in return position does: `serialize::<T>` needs a separate compiled version
per `T`, but a vtable can only hold one entry per method name, not one per possible `T`.

### Fully Qualified Syntax

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) { println!("Flying as pilot"); }
}

impl Wizard for Human {
    fn fly(&self) { println!("Flying as wizard"); }
}

impl Human {
    fn fly(&self) { println!("Flying as human"); }
}

fn main() {
    let person = Human;

    person.fly();           // Calls Human::fly
    Pilot::fly(&person);    // Calls Pilot::fly
    Wizard::fly(&person);   // Calls Wizard::fly

    // Fully qualified syntax for associated functions
    // <Type as Trait>::function(receiver);
    <Human as Pilot>::fly(&person);
}
```

### Marker Traits

A marker trait carries no methods; it exists purely so the compiler (and other trait
bounds) can ask "does this type have property X?" `Send`, `Sync`, `Sized`, and `Unpin`
are also auto traits: the compiler implements them automatically for a type based on its
fields, rather than requiring an explicit `impl`. A struct is `Send` (safe to move to
another thread) and `Sync` (safe to share via `&T` across threads) only if every field is;
a raw pointer isn't `Send` or `Sync`, so a struct containing one, like `NotSend` below,
isn't either, without you writing anything:

```rust
// Send: safe to transfer between threads
// Sync: safe to share between threads
// Sized: has known size at compile time
// Unpin: can be moved after being pinned

// Opt-out of automatic trait
struct NotSend {
    data: *const (),
}

// impl !Send for NotSend {}  // Unstable, raw pointers are !Send by default
```

### Extension Traits

```rust
// Add methods to existing types
trait StringExt {
    fn shout(&self) -> String;
}

impl StringExt for str {
    fn shout(&self) -> String {
        format!("{}!", self.to_uppercase())
    }
}

fn main() {
    let s = "hello";
    println!("{}", s.shout());  // "HELLO!"
}
```

---

## Quick Reference

### Trait definition syntax

```rust
trait MyTrait {
    // Required method
    fn required(&self);

    // Provided method (default implementation)
    fn provided(&self) {
        println!("Default");
    }

    // Associated type
    type Output;

    // Associated constant
    const VALUE: i32;
}
```

### Common standard library traits

| Trait | Purpose |
|-------|---------|
| `Debug` | Debug formatting `{:?}` |
| `Display` | User-facing formatting `{}` |
| `Clone` | Explicit duplication |
| `Copy` | Implicit bitwise copy |
| `PartialEq` / `Eq` | Equality comparison |
| `PartialOrd` / `Ord` | Ordering comparison |
| `Default` | Default values |
| `From` / `Into` | Type conversion |
| `AsRef` / `AsMut` | Cheap reference conversion |
| `Deref` / `DerefMut` | Smart pointer behavior |
| `Iterator` | Iteration protocol |
| `Drop` | Destructor |

### Trait bound syntax

```rust
// impl Trait (simple)
fn foo(x: impl MyTrait) {}

// Generics (explicit)
fn foo<T: MyTrait>(x: T) {}

// Multiple bounds
fn foo<T: MyTrait + Clone>(x: T) {}

// Where clause (complex)
fn foo<T, U>(x: T, y: U)
where
    T: MyTrait + Clone,
    U: Debug + Default,
{}
```

---

## Common Patterns

- Trait objects (`Box<dyn Trait>`, or `Vec<Box<dyn Trait>>` for a mixed collection) let you hold several concrete types behind one shared trait. See Trait Objects (Dynamic Dispatch) above.
- Blanket implementations (`impl<T: Display> Printable for T`) give every type that already implements one trait a new capability for free. See Blanket Implementations above.
- Extension traits add methods to a type you don't own, as long as the trait itself is defined in your crate. See Extension Traits above.
- Fully qualified syntax (`<Type as Trait>::method(&value)`) disambiguates when a type and more than one trait define a method with the same name. See Fully Qualified Syntax above.

---

## Common Pitfalls

- The orphan rule blocks implementing an external trait for an external type: either the trait or the type has to be local to your crate. See Orphan Rule above.
- Returning `impl Trait` only works when every code path returns the same concrete type; returning different types from different branches needs a trait object (`Box<dyn Trait>`) instead. See Returning Traits > Limitation: Single Concrete Type above.
- Not every trait can become a trait object: a method that returns `Self` or takes a generic parameter makes a trait non-object-safe. See Object Safety above.

---

## Summary

The trait definition syntax, the standard library trait table, and the trait bound
syntax forms are collected in the Quick Reference section above.

### Best Practices

1. Prefer composition over inheritance: use multiple traits.
2. Keep traits focused on a single responsibility.
3. Provide default implementations when sensible.
4. Use associated types for "has one" relationships.
5. Use generics for "can be any" relationships.
6. Document trait contracts clearly.
