# Common Derivable Traits — Quick Reference

## Table of Contents

1. [Introduction](#introduction)
2. [Debug](#debug)
3. [Clone and Copy](#clone-and-copy)
4. [PartialEq and Eq](#partialeq-and-eq)
5. [PartialOrd and Ord](#partialord-and-ord)
6. [Hash](#hash)
7. [Default](#default)
8. [Combining Derives](#combining-derives)
9. [Quick Reference](#quick-reference)
10. [Common Patterns](#common-patterns)
11. [Common Pitfalls](#common-pitfalls)
12. [Summary](#summary)

---

## Introduction

Rust's standard library ships a set of traits whose implementations can be generated
automatically with `#[derive(...)]` instead of writing them by hand. This chapter is a
cheat sheet for the traits you'll reach for most: what each one gives you, and a minimal
example of deriving it. `Display` is not in this list: unlike `Debug`, it has no derive
macro in the standard library and needs a manual `impl` (see
[Chapter 13: Traits & Trait Implementations](IV.13-traits-and-trait-implementations.md)).

---

## Debug

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);   // Point { x: 1, y: 2 }
    println!("{:#?}", p);  // Pretty printed
}
```

---

## Clone and Copy

```rust
#[derive(Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(Clone)]  // Can't derive Copy because String is not Copy
struct Person {
    name: String,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1;  // Copy
    println!("{:?}", p1);  // Still valid

    let person1 = Person { name: String::from("Alice") };
    let person2 = person1.clone();  // Explicit clone
}
```

---

## PartialEq and Eq

```rust
#[derive(PartialEq, Eq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 1, y: 2 };
    let p3 = Point { x: 3, y: 4 };

    println!("{}", p1 == p2);  // true
    println!("{}", p1 == p3);  // false
}
```

---

## PartialOrd and Ord

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Priority {
    level: u32,
}

fn main() {
    let low = Priority { level: 1 };
    let high = Priority { level: 10 };

    println!("{}", low < high);  // true

    let mut priorities = vec![high, low];
    priorities.sort();
}
```

---

## Hash

```rust
use std::collections::HashMap;

#[derive(Hash, PartialEq, Eq)]
struct Key {
    id: u32,
    name: String,
}

fn main() {
    let mut map = HashMap::new();
    map.insert(
        Key { id: 1, name: String::from("one") },
        "first",
    );
}
```

---

## Default

```rust
#[derive(Default, Debug)]
struct Config {
    debug: bool,
    max_size: usize,
    name: String,
}

fn main() {
    let config = Config::default();
    println!("{:?}", config);
    // Config { debug: false, max_size: 0, name: "" }

    // Partial override
    let config = Config {
        debug: true,
        ..Default::default()
    };
}
```

---

## Combining Derives

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]
struct Entity {
    id: u64,
    name: String,
    active: bool,
}
```

---

## Quick Reference

| Derive | Purpose | Requires |
|--------|---------|----------|
| `Debug` | Enable `{:?}` formatting | All fields implement `Debug` |
| `Clone` | Enable `.clone()` | All fields implement `Clone` |
| `Copy` | Enable implicit copying | All fields implement `Copy`; also requires `Clone` |
| `PartialEq` | Enable `==` and `!=` | All fields implement `PartialEq` |
| `Eq` | Total equality | Requires `PartialEq`; no `f32`/`f64` fields |
| `PartialOrd` | Enable `<`, `>`, `<=`, `>=` | All fields implement `PartialOrd` |
| `Ord` | Total ordering | Requires `PartialOrd` + `Eq`; no `f32`/`f64` fields |
| `Hash` | Enable use as a `HashMap`/`HashSet` key | All fields implement `Hash` |
| `Default` | Enable `Default::default()` | All fields implement `Default` |

---

## Common Patterns

- Derive `Debug` on almost everything, including internal-only types: `{:?}` output is
  what `println!`, `assert_eq!` failure messages, and `dbg!` all rely on. See Debug
  above.
- Derive the whole equality/ordering/hashing group together (`PartialEq, Eq, Hash`) on
  any type used as a `HashMap`/`HashSet` key, rather than picking just one; the compiler
  will point out which is missing wherever you try to use the type. See PartialEq and Eq
  and Hash above.
- Reach for `..Default::default()` when constructing a struct where most fields should
  take their default value and only a few need to be set explicitly. See Default above.
- Stack derives on one line (`#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]`)
  rather than writing separate `#[derive(...)]` attributes; there's no difference in
  behavior, only in how many attributes you scan. See Combining Derives above.

---

## Common Pitfalls

- `Copy` requires `Clone` and requires every field to be `Copy`; a struct holding a
  `String`, `Vec`, or any other heap-allocated field can't derive `Copy`, only `Clone`.
  See Clone and Copy above.
- `Eq` and `Ord` can't be derived on a type containing `f32`/`f64` fields, because
  floating-point values don't have a total order (`NaN` compares unequal to everything,
  including itself). `PartialEq`/`PartialOrd` still work on floats. See PartialEq and Eq
  and PartialOrd and Ord above.
- Deriving `PartialOrd`/`Ord` compares fields in declaration order, top to bottom: a
  `Priority` with `level` declared before `label` sorts by `level` first, and only
  compares `label` to break ties. Reordering the fields changes the sort order. See
  PartialOrd and Ord above.
- Adding a field to a struct that derives `Hash` (or `PartialEq`/`Eq`) changes its hash
  and equality behavior for every existing user of that type; a previously-inserted
  `HashMap` key won't compare equal after the type gains a field with a new value. See
  Hash above.

---

## Summary

Derive `Debug`, `Clone`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, and `Default`
instead of hand-writing them whenever every field of a struct or enum already implements
the trait being derived: the compiler generates a field-by-field implementation for you.
`Copy` and `Eq`/`Ord` add extra requirements (see the Quick Reference table above), and
`Display` has to be implemented by hand since it isn't derivable in the standard library.
