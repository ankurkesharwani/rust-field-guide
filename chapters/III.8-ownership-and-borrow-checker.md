# Ownership and the Borrow Checker in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Ownership Rules](#ownership-rules)
3. [Move Semantics](#move-semantics)
4. [Copy Types](#copy-types)
5. [References and Borrowing](#references-and-borrowing)
6. [The Borrow Checker](#the-borrow-checker)
7. [Mutable Borrowing](#mutable-borrowing)
8. [Slice Types](#slice-types)
9. [Ownership in Structs](#ownership-in-structs)
10. [Interior Mutability](#interior-mutability)
11. [Quick Reference](#quick-reference)
12. [Common Patterns](#common-patterns)
13. [Common Pitfalls](#common-pitfalls)
14. [Summary](#summary)

---

## Introduction

Rust's ownership system is its most unique feature, providing memory safety without a garbage collector. It enforces strict rules at compile time to prevent:

- **Use after free**: Accessing memory that has been deallocated
- **Double free**: Freeing memory twice
- **Dangling pointers**: References to invalid memory
- **Data races**: Concurrent access causing undefined behavior

Understanding ownership is essential for writing Rust code effectively.

---

## Ownership Rules

Rust has three fundamental ownership rules:

```rust
// Rule 1: Each value in Rust has a variable that's called its owner.
// Rule 2: There can only be one owner at a time.
// Rule 3: When the owner goes out of scope, the value will be dropped.

fn main() {
    {
        let s = String::from("hello"); // s is the owner of the String

        // s is valid here
        println!("{}", s);

    } // s goes out of scope here, String is dropped, memory is freed

    // s is no longer valid here
    // println!("{}", s); // ERROR: s is not in scope
}
```

### Scope and Drop

```rust
fn main() {
    let outer = String::from("outer");

    {
        let inner = String::from("inner");
        println!("{} {}", outer, inner); // Both valid
    } // inner is dropped here

    println!("{}", outer); // outer still valid
    // println!("{}", inner); // ERROR: inner is not in scope

} // outer is dropped here
```

### The Drop Trait

```rust
struct CustomResource {
    name: String,
}

impl Drop for CustomResource {
    fn drop(&mut self) {
        println!("Dropping resource: {}", self.name);
    }
}

fn main() {
    let r1 = CustomResource { name: String::from("First") };
    let r2 = CustomResource { name: String::from("Second") };

    println!("Resources created");
} // Outputs: "Dropping resource: Second" then "Dropping resource: First"
  // (dropped in reverse order of creation)
```

---

## Move Semantics

### What is a Move?

When you assign a value to another variable or pass it to a function, ownership is transferred (moved):

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1; // s1's ownership moved to s2

    // println!("{}", s1); // ERROR: s1 has been moved
    println!("{}", s2); // OK: s2 is the new owner
}
```

### Why Move Instead of Copy?

```rust
// String in memory:
// s1 -> [ptr|len|capacity] -> "hello" on heap
//
// After s2 = s1:
// s1 -> [invalid]
// s2 -> [ptr|len|capacity] -> "hello" on heap
//
// If both were valid, "hello" would be freed twice (double free bug)
```

### Move in Function Calls

```rust
fn take_ownership(s: String) {
    println!("{}", s);
} // s is dropped here

fn main() {
    let s = String::from("hello");
    take_ownership(s); // s is moved into the function

    // println!("{}", s); // ERROR: s has been moved
}
```

### Returning Ownership

```rust
fn give_ownership() -> String {
    let s = String::from("hello");
    s // Ownership is moved to the caller
}

fn take_and_give_back(s: String) -> String {
    s // Ownership is moved back to the caller
}

fn main() {
    let s1 = give_ownership(); // s1 owns the String

    let s2 = String::from("world");
    let s3 = take_and_give_back(s2); // s2 moved in, s3 owns result

    // s2 is invalid, s3 is valid
    println!("{}, {}", s1, s3);
}
```

### Move in Pattern Matching

```rust
fn main() {
    let opt = Some(String::from("hello"));

    match opt {
        Some(s) => println!("{}", s), // s takes ownership
        None => println!("None"),
    }

    // println!("{:?}", opt); // ERROR: opt was moved

    // Use ref to borrow instead
    let opt = Some(String::from("hello"));
    match opt {
        Some(ref s) => println!("{}", s), // s borrows
        None => println!("None"),
    }
    println!("{:?}", opt); // OK: opt wasn't moved
}
```

---

## Copy Types

### The Copy Trait

Types that implement `Copy` are duplicated instead of moved:

```rust
fn main() {
    // Copy types - simple values stored entirely on the stack
    let x = 5;
    let y = x; // Copy, not move

    println!("x = {}, y = {}", x, y); // Both valid!

    // These types implement Copy:
    let a: i32 = 1;        // All integers
    let b: f64 = 3.14;     // All floats
    let c: bool = true;    // Booleans
    let d: char = 'a';     // Characters
    let e: (i32, i32) = (1, 2); // Tuples of Copy types
    let f: [i32; 3] = [1, 2, 3]; // Arrays of Copy types
}
```

### What Cannot Be Copy?

```rust
fn main() {
    // These types do NOT implement Copy:

    // String - owns heap data
    let s1 = String::from("hello");
    let s2 = s1; // Move, not copy

    // Vec - owns heap data
    let v1 = vec![1, 2, 3];
    let v2 = v1; // Move, not copy

    // Any type that implements Drop cannot be Copy
    // (because drop implies cleanup of owned resources)
}
```

### Implementing Copy

```rust
// Derive Copy for simple structs
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1; // Copy, not move

    println!("p1: ({}, {})", p1.x, p1.y); // Both valid
    println!("p2: ({}, {})", p2.x, p2.y);
}

// Cannot derive Copy if any field doesn't implement Copy
// #[derive(Copy, Clone)]
// struct BadPoint {
//     x: i32,
//     name: String, // ERROR: String doesn't implement Copy
// }
```

### Clone vs Copy

```rust
fn main() {
    // Copy: implicit, bitwise copy, cheap
    let x = 5;
    let y = x; // Implicit copy

    // Clone: explicit, can be expensive
    let s1 = String::from("hello");
    let s2 = s1.clone(); // Explicit clone

    println!("{}, {}", s1, s2); // Both valid

    // Copy implies Clone, but Clone doesn't imply Copy
}
```

---

## References and Borrowing

### Immutable References

References allow you to refer to a value without taking ownership:

```rust
fn main() {
    let s = String::from("hello");

    let len = calculate_length(&s); // & creates a reference

    println!("'{}' has length {}", s, len); // s still valid!
}

fn calculate_length(s: &String) -> usize {
    s.len()
} // s goes out of scope, but doesn't drop the String (it doesn't own it)
```

### How References Work

```rust
// Memory layout:
// s  -> [ptr|len|capacity] -> "hello" on heap
//        ^
//        |
// ref -> [ptr to s's ptr]
//
// The reference points to s, not directly to the heap data
```

A reference is a pointer to the variable's storage, not a second handle to the heap
allocation. `s` still owns the heap data and is the only thing that ever frees it, which
is why a function that only borrows a `String` never needs to worry about running its
destructor.

### Multiple Immutable References

```rust
fn main() {
    let s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    let r3 = &s;

    // Multiple immutable references are allowed
    println!("{}, {}, {}", r1, r2, r3);
}
```

### Dereferencing

```rust
fn main() {
    let x = 5;
    let r = &x;

    // Use * to dereference
    println!("r = {}", r);   // Auto-deref for Display
    println!("*r = {}", *r); // Explicit deref

    // Comparison
    assert_eq!(5, *r);
    assert_eq!(&5, r);

    // References to references
    let rr = &r;
    println!("{}", **rr); // Double deref
}
```

---

## The Borrow Checker

### The Fundamental Rule

At any given time, you can have either:
- **One mutable reference**, OR
- **Any number of immutable references**

But not both simultaneously.

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;     // OK: first immutable borrow
    let r2 = &s;     // OK: second immutable borrow

    // let r3 = &mut s; // ERROR: cannot borrow as mutable while immutable borrows exist

    println!("{}, {}", r1, r2);

    // After this point, r1 and r2 are no longer used
    // So we can create a mutable reference

    let r3 = &mut s; // OK: no active immutable borrows
    println!("{}", r3);
}
```

### Non-Lexical Lifetimes (NLL)

The borrow checker is smart about when references are actually used:

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{}, {}", r1, r2);
    // r1 and r2 are no longer used after this point

    let r3 = &mut s; // OK: r1 and r2 are "dead"
    println!("{}", r3);
}
```

### Preventing Dangling References

```rust
fn main() {
    // let reference_to_nothing = dangle(); // ERROR
}

// fn dangle() -> &String {
//     let s = String::from("hello");
//     &s // ERROR: returning reference to local variable
// } // s is dropped here, reference would be invalid

fn no_dangle() -> String {
    let s = String::from("hello");
    s // Ownership is moved out, no dangling reference
}
```

### Aliasing and Mutation

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    // This would be dangerous without the borrow checker:
    // let first = &v[0];
    // v.push(4); // Could reallocate, invalidating first
    // println!("{}", first); // Would be dangling!

    // Borrow checker prevents this:
    let first = &v[0];
    // v.push(4); // ERROR: cannot borrow v as mutable
    println!("{}", first);

    // After first is no longer used:
    v.push(4); // OK
}
```

---

## Mutable Borrowing

### Basic Mutable References

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);

    println!("{}", s); // "hello, world"
}

fn change(s: &mut String) {
    s.push_str(", world");
}
```

### Only One Mutable Reference

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    // let r2 = &mut s; // ERROR: cannot borrow `s` as mutable more than once

    println!("{}", r1);
}
```

### Mutable Reference Scope

```rust
fn main() {
    let mut s = String::from("hello");

    {
        let r1 = &mut s;
        r1.push_str(" world");
    } // r1 goes out of scope

    let r2 = &mut s; // OK: r1 is out of scope
    r2.push('!');

    println!("{}", s);
}
```

### Cannot Mix Mutable and Immutable

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;     // Immutable borrow
    let r2 = &s;     // Another immutable borrow
    // let r3 = &mut s; // ERROR: cannot borrow as mutable

    println!("{} and {}", r1, r2);
    // r1 and r2 no longer used

    let r3 = &mut s; // OK now
    println!("{}", r3);
}
```

---

## Slice Types

### String Slices

```rust
fn main() {
    let s = String::from("hello world");

    let hello = &s[0..5];   // "hello"
    let world = &s[6..11];  // "world"

    // Shorthand
    let hello = &s[..5];    // From start
    let world = &s[6..];    // To end
    let whole = &s[..];     // Whole string

    println!("{} {}", hello, world);
}
```

### Slice Type

```rust
fn main() {
    let s = String::from("hello world");

    let word = first_word(&s);
    println!("First word: {}", word);

    // String literals are slices
    let literal: &str = "hello";
}

fn first_word(s: &str) -> &str {  // Takes &str, works with String and &str
    let bytes = s.as_bytes();

    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[..i];
        }
    }

    &s[..]
}
```

### Slices Enforce Borrowing Rules

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    // s.clear(); // ERROR: cannot borrow s as mutable while word exists

    println!("{}", word);

    s.clear(); // OK: word is no longer used
}

fn first_word(s: &str) -> &str {
    &s[..5]
}
```

### Array and Vec Slices

```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];
    let slice: &[i32] = &arr[1..4];
    println!("{:?}", slice); // [2, 3, 4]

    let vec = vec![1, 2, 3, 4, 5];
    let slice: &[i32] = &vec[1..4];
    println!("{:?}", slice); // [2, 3, 4]

    // Mutable slice
    let mut vec = vec![1, 2, 3, 4, 5];
    let slice: &mut [i32] = &mut vec[1..4];
    slice[0] = 20;
    println!("{:?}", vec); // [1, 20, 3, 4, 5]
}
```

---

## Ownership in Structs

### Owned Data in Structs

```rust
struct User {
    username: String,  // User owns this String
    email: String,     // User owns this String
    active: bool,
}

fn main() {
    let user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        active: true,
    };

    // user owns username and email
    println!("{}", user.username);
} // user dropped, username and email are freed
```

### References in Structs (Require Lifetimes)

```rust
// References in structs need lifetime annotations
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

    println!("{}", user.username);
}
```

### Methods and Self

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // &self - borrows self immutably
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // &mut self - borrows self mutably
    fn double(&mut self) {
        self.width *= 2;
        self.height *= 2;
    }

    // self - takes ownership of self
    fn into_square(self) -> Rectangle {
        let side = std::cmp::min(self.width, self.height);
        Rectangle { width: side, height: side }
    }
}

fn main() {
    let mut rect = Rectangle { width: 10, height: 20 };

    println!("Area: {}", rect.area()); // Borrows immutably

    rect.double(); // Borrows mutably
    println!("Doubled: {}x{}", rect.width, rect.height);

    let square = rect.into_square(); // Takes ownership
    // rect is now invalid
    println!("Square: {}x{}", square.width, square.height);
}
```

---

## Interior Mutability

### Cell and RefCell

Sometimes you need to mutate data even when you only have an immutable reference:

```rust
use std::cell::{Cell, RefCell};

fn main() {
    // Cell: for Copy types
    let cell = Cell::new(5);
    cell.set(10); // No &mut needed
    println!("{}", cell.get());

    // RefCell: for any type, runtime borrow checking
    let refcell = RefCell::new(String::from("hello"));

    // Borrow mutably at runtime
    refcell.borrow_mut().push_str(" world");

    println!("{}", refcell.borrow());
}
```

### RefCell Panics on Violation

`RefCell` still enforces the one-mutable-or-many-immutable rule, it just does it at
runtime instead of compile time: it keeps a borrow counter, `borrow()` and
`borrow_mut()` increment it, and dropping the returned guard decrements it. If
`borrow_mut()` is called while the count shows any other borrow outstanding, it panics
instead of returning a reference. This trades a compile error for a runtime one, which
is what makes `RefCell` usable in cases the compiler can't verify statically, such as
mutating shared state reached through an `Rc`.

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    let borrow1 = cell.borrow();
    let borrow2 = cell.borrow(); // OK: multiple immutable borrows

    // let mut_borrow = cell.borrow_mut(); // PANIC at runtime!
    // Cannot have mutable borrow while immutable borrows exist

    drop(borrow1);
    drop(borrow2);

    let mut_borrow = cell.borrow_mut(); // OK now
}
```

### Rc with RefCell

`Rc<T>` only ever hands out shared (`&T`) access to what it wraps, since it has no way
to know whether another `Rc` pointing at the same data is being read from elsewhere.
Wrapping the shared value in a `RefCell` is what gets mutation back: `Rc<RefCell<T>>`
gives every clone shared ownership of the same `RefCell`, and each one can call
`borrow_mut()` on it independently, with `RefCell`'s runtime check standing in for the
compile-time check that shared ownership rules out.

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    // Rc<RefCell<T>> allows shared ownership with interior mutability
    let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

    let a = Rc::clone(&shared);
    let b = Rc::clone(&shared);

    // Both can modify the inner Vec
    a.borrow_mut().push(4);
    b.borrow_mut().push(5);

    println!("{:?}", shared.borrow()); // [1, 2, 3, 4, 5]
}
```

---

## Quick Reference

| Type | Ownership | Thread safety |
|------|-----------|---------------|
| `T` | Owned | Depends on `T` |
| `&T` | Borrowed immutably | If `T: Sync` |
| `&mut T` | Borrowed mutably | Exclusive |
| `Box<T>` | Owned, heap-allocated | Depends on `T` |
| `Rc<T>` | Shared ownership | Single-threaded |
| `Arc<T>` | Shared ownership | Multi-threaded |
| `Cell<T>` | Interior mutability | Single-threaded, `Copy` |
| `RefCell<T>` | Interior mutability | Single-threaded |

The borrow checker allows either any number of immutable references (`&T`) or exactly
one mutable reference (`&mut T`) at a time, never both, and a reference must never
outlive the value it points to.

---

## Common Patterns

### Pattern 1: Take and Return

```rust
fn process(mut data: Vec<i32>) -> Vec<i32> {
    data.push(100);
    data
}

fn main() {
    let v = vec![1, 2, 3];
    let v = process(v); // v moves in, result moves out
    println!("{:?}", v);
}
```

### Pattern 2: Borrow for Read-Only

```rust
fn analyze(data: &[i32]) -> (i32, i32) {
    let sum: i32 = data.iter().sum();
    let count = data.len() as i32;
    (sum, count)
}

fn main() {
    let v = vec![1, 2, 3, 4, 5];
    let (sum, count) = analyze(&v);
    println!("Sum: {}, Count: {}", sum, count);
    println!("Original: {:?}", v); // v still valid
}
```

### Pattern 3: Mutable Borrow for Modification

```rust
fn modify(data: &mut Vec<i32>) {
    for x in data.iter_mut() {
        *x *= 2;
    }
}

fn main() {
    let mut v = vec![1, 2, 3];
    modify(&mut v);
    println!("{:?}", v); // [2, 4, 6]
}
```

### Pattern 4: Clone When Needed

```rust
fn main() {
    let original = String::from("hello");

    // Clone when you need both the original and a modified version
    let mut copy = original.clone();
    copy.push_str(" world");

    println!("Original: {}", original);
    println!("Modified: {}", copy);
}
```

---

## Common Pitfalls

### Pitfall 1: Moving Out of a Borrowed Context

```rust
struct Container {
    value: String,
}

impl Container {
    // ERROR: cannot move out of borrowed content
    // fn take_value(&self) -> String {
    //     self.value // ERROR: cannot move
    // }

    // Solution 1: Clone
    fn clone_value(&self) -> String {
        self.value.clone()
    }

    // Solution 2: Return reference
    fn get_value(&self) -> &String {
        &self.value
    }

    // Solution 3: Take ownership of self
    fn into_value(self) -> String {
        self.value
    }
}
```

### Pitfall 2: Borrowing in a Loop

```rust
fn main() {
    let mut vec = vec![1, 2, 3, 4, 5];

    // ERROR: cannot borrow as mutable more than once
    // for item in &mut vec {
    //     vec.push(*item); // ERROR
    // }

    // Solution: collect indices first, then modify
    let len = vec.len();
    for i in 0..len {
        let item = vec[i];
        vec.push(item);
    }
}
```

`for item in &mut vec` holds a mutable borrow of `vec` for the whole loop, so calling
`vec.push` inside it would be a second mutable borrow, and it would be unsound even if
the borrow checker allowed it: pushing can reallocate the backing buffer, which would
invalidate the iterator's pointer into the old buffer. Reading `len` once before the
loop and indexing by `i` avoids holding any borrow across the `push` calls.

### Pitfall 3: Returning References to Local Variables

```rust
// ERROR: returning reference to local
// fn create_string() -> &String {
//     let s = String::from("hello");
//     &s // ERROR: s is dropped
// }

// Solution: return owned value
fn create_string() -> String {
    String::from("hello")
}

// Or take a reference parameter
fn append_world(s: &mut String) {
    s.push_str(" world");
}
```

### Pitfall 4: Self-Referential Structs

```rust
// This doesn't work:
// struct SelfRef {
//     data: String,
//     reference: &str, // Can't reference data
// }

// Solution 1: Use indices instead of references
struct WithIndex {
    data: String,
    start: usize,
    end: usize,
}

impl WithIndex {
    fn get_slice(&self) -> &str {
        &self.data[self.start..self.end]
    }
}

// Solution 2: Use Pin and unsafe (advanced)
// Solution 3: Use separate allocations
```

A struct can't hold a reference into its own field because Rust is free to move a struct
in memory (returning it, putting it in a `Vec`, and so on), and moving it would leave the
reference pointing at the old, now-invalid location. `WithIndex` sidesteps this by storing
offsets instead of a pointer, so it recomputes the slice through `self.data` each time
rather than keeping a stale pointer around.

### Pitfall 5: Holding References Across Await Points

```rust
// In async code, be careful with references across .await

// This pattern can be problematic:
// async fn problematic(data: &mut Data) {
//     let reference = &data.field;
//     some_async_operation().await; // reference held across await
//     use_reference(reference);
// }

// Solution: restructure to not hold references
async fn better(data: &mut Data) {
    let value = data.field.clone();
    some_async_operation().await;
    data.field = process(value);
}

struct Data { field: String }
async fn some_async_operation() {}
fn process(_: String) -> String { String::new() }
```

---

## Summary

### Ownership Rules

1. Each value has exactly one owner
2. When owner goes out of scope, value is dropped
3. Ownership can be transferred (moved) or borrowed

### Borrowing Rules

1. Any number of immutable references (`&T`)
2. OR exactly one mutable reference (`&mut T`)
3. References must always be valid

See the Quick Reference section above for how ownership and thread safety compare across
`T`, `&T`, `&mut T`, `Box`, `Rc`, `Arc`, `Cell`, and `RefCell`.

### Best Practices

1. Default to owned types in structs
2. Use references for function parameters when possible
3. Clone judiciously when ownership becomes complex
4. Use `Rc`/`Arc` for shared ownership
5. Use `RefCell`/`Mutex` for interior mutability when needed
6. Trust the borrow checker. It's usually right
