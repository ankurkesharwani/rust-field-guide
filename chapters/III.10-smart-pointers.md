# Smart Pointers

## Table of Contents

1. [Introduction](#introduction)
2. [Box](#box)
3. [Rc and Arc](#rc-and-arc)
4. [RefCell and Cell](#refcell-and-cell)
5. [Weak](#weak)
6. [Interior Mutability](#interior-mutability)
7. [Quick Reference](#quick-reference)
8. [Common Patterns](#common-patterns)
9. [Common Pitfalls](#common-pitfalls)
10. [Summary](#summary)

---

## Introduction

[Chapter 8](III.8-ownership-and-borrow-checker.md) established the default rule: every
value has exactly one owner, and the borrow checker enforces either one mutable
reference or any number of shared references, all checked at compile time. That rule
covers most code, but it is also strict enough to rule out some perfectly safe patterns:
a value whose size isn't known until runtime, a value that genuinely needs more than one
owner, or a value that needs to be mutated through a shared reference.

Smart pointers are types that own heap-allocated data and add some behavior on top of a
plain reference, and each of the ones in this chapter exists to relax one specific part
of the ownership rule:

- `Box<T>` gives a value a single owner, like normal, but moves it to the heap so its
  size doesn't have to be known at compile time.
- `Rc<T>` and `Arc<T>` allow more than one owner, by counting owners at runtime instead
  of enforcing a single one at compile time.
- `RefCell<T>` and `Cell<T>` allow mutation through a shared reference, by checking the
  "one writer XOR many readers" rule at runtime instead of compile time.
- `Weak<T>` is a non-owning reference used alongside `Rc`/`Arc` to avoid the reference
  cycles that shared ownership can otherwise create.

Each of these trades a compile-time guarantee for a runtime check or a runtime cost.
That trade is worth understanding precisely, not just the API, so this chapter spends as
much time on why each type behaves the way it does as on how to call it.

---

## Box

`Box<T>` is the simplest smart pointer: it allocates `T` on the heap and stores a
pointer to it on the stack. Moving a `Box` moves the pointer, not the data it points to,
and when the `Box` is dropped, the heap allocation is freed. There is no reference
counting and no runtime check involved; ownership works exactly like it does for any
other value, just with the data relocated to the heap.

### Recursive Types

A struct or enum's size has to be known at compile time, because Rust needs to know how
much stack space to reserve for it. A directly recursive type breaks that: if `List`
contained a `List`, the compiler would need to compute an infinite size.

```rust
// ERROR: recursive type `List` has infinite size
enum List {
    Cons(i32, List),
    Nil,
}
```

`Box<T>` fixes this because a `Box` is just a pointer, and a pointer has a fixed size
regardless of what it points to:

```rust
#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
    println!("{:?}", list); // Cons(1, Cons(2, Cons(3, Nil)))
}
```

Each `Cons` variant now stores an `i32` plus one pointer-sized `Box`, so `List` has a
fixed size no matter how many elements are chained together. This is the classic
justification for `Box` in the Rust book, and it generalizes to any type that needs to
refer to itself: trees, linked lists, and similar structures all reach for `Box` (or
`Rc`, covered next) for the same reason.

### Trait Objects

The other common reason to reach for `Box` is storing a value whose concrete type isn't
known until runtime. `Box<dyn Trait>` stores a pointer to some type that implements
`Trait`, without the caller needing to know which concrete type it is:

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

fn make_shape() -> Box<dyn Shape> {
    Box::new(Circle { radius: 2.0 })
}

fn main() {
    let shape = make_shape();
    println!("area = {}", shape.area()); // area = 12.566370614359172
}
```

This is the same problem as the recursive type, restated: `dyn Shape` has no fixed size
because different implementors of `Shape` can be different sizes, so it can only be
handled through a pointer. [Chapter 13](IV.13-traits-and-trait-implementations.md#returning-traits)
covers this from the trait side, including why `impl Trait` in return position can't
express "one of several possible types" the way `Box<dyn Trait>` can.

### Zero-Cost Indirection

Beyond those two uses, `Box::new(value)` is also just a way to force a value onto the
heap, for example to keep a large value from bloating the stack frame it lives in, or to
move it into a data structure that needs to store it uniformly with other heap-allocated
values. Deref coercion (see [Chapter 13](IV.13-traits-and-trait-implementations.md#common-standard-library-traits))
means a `Box<T>` can be used almost everywhere a `T` could, so this indirection is
usually invisible at the call site.

---

## Rc and Arc

`Box<T>` still enforces single ownership; it just moves the data to the heap. `Rc<T>`
("reference counted") drops the single-ownership requirement itself: several `Rc`
handles can point at the same heap allocation, and the allocation is freed only once the
last handle is dropped.

### How Rc::clone Works

Calling `.clone()` on most types deep-copies the data. Calling `Rc::clone()` does not;
it increments an integer counter stored alongside the data and returns a new pointer to
the same allocation. Dropping an `Rc` decrements that counter, and the underlying value
is deallocated only when the counter reaches zero. This is why `Rc::clone(&a)` is
idiomatically preferred over `a.clone()` for `Rc` values: it makes clear at the call site
that this is a cheap pointer-and-counter bump, not a deep copy.

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(5);
    println!("count after creating a = {}", Rc::strong_count(&a)); // 1

    let b = Rc::clone(&a);
    println!("count after creating b = {}", Rc::strong_count(&a)); // 2

    {
        let c = Rc::clone(&a);
        println!("count after creating c = {}", Rc::strong_count(&a)); // 3
        drop(c);
    }

    println!("count after c goes out of scope = {}", Rc::strong_count(&a)); // 2
    println!("b = {}", b);
}
```

`Rc<T>` only hands out shared references (`&T`); it does not implement `DerefMut`,
because it can't know whether any other `Rc` handle is currently reading the value. That
restriction is also why `Rc<T>` alone can't be mutated, and why it's so often paired with
`RefCell<T>` (see [Interior Mutability](#interior-mutability) below) when shared,
mutable state is actually needed.

### Arc: Rc for Multiple Threads

`Rc<T>`'s counter is a plain, non-atomic integer. Incrementing it from two threads at
once is a data race, so the compiler refuses to let `Rc<T>` cross a thread boundary at
all (it doesn't implement `Send`). `Arc<T>` ("atomically reference counted") is the same
idea implemented with an atomic counter instead, which makes it safe to share across
threads:

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    let mut handles = vec![];

    for i in 0..3 {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            println!("thread {i} sees {data:?}");
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

Atomic operations are more expensive than plain integer increments, so `Arc` costs more
than `Rc` even in single-threaded code. Use `Rc` when a value never needs to leave the
current thread, and `Arc` when it does; see [Chapter 21](VIII.21-multi-threading.md) for
the threading side of this, including how `Arc` is typically combined with `Mutex` to
share *mutable* state across threads.

---

## RefCell and Cell

The borrow checker's "one writer or many readers" rule is normally enforced at compile
time, which means the compiler sometimes rejects code that is actually safe, just not
provably so from where it's standing. `RefCell<T>` moves that same rule to runtime: it
compiles code the borrow checker would otherwise reject, and instead panics if the rule
is actually violated while the program runs.

### RefCell: Runtime-Checked Borrowing

`RefCell<T>` hands out `Ref<T>` and `RefMut<T>` guards through `.borrow()` and
`.borrow_mut()`, and keeps its own internal count of how many of each are currently
alive. Requesting a borrow that would violate the rule doesn't fail to compile; it
panics immediately, at the `.borrow()` or `.borrow_mut()` call:

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    let b1 = cell.borrow_mut();
    let b2 = cell.borrow_mut(); // panics: already mutably borrowed
    println!("{b1} {b2}");
}
```

Running this panics with `already borrowed: BorrowMutError` (or the equivalent for two
overlapping mutable borrows), at runtime, with no compiler warning beforehand. This is
the real cost of `RefCell`: the borrow-checker safety net is still there, but it has
moved from "your code won't compile" to "your code will crash if this path is ever hit,"
so a `RefCell` bug can ship and only surface under a specific sequence of calls.

The most common legitimate use is mutating a field through a shared `&self`:

```rust
use std::cell::RefCell;

struct Counter {
    count: RefCell<i32>,
}

impl Counter {
    fn increment(&self) {
        *self.count.borrow_mut() += 1;
    }
}

fn main() {
    let counter = Counter { count: RefCell::new(0) };
    counter.increment();
    counter.increment();
    println!("count = {}", counter.count.borrow()); // count = 2
}
```

Note that `increment` takes `&self`, not `&mut self`; without `RefCell`, mutating
`count` here would require `&mut self`, which would in turn require every caller to hold
a unique reference to the whole `Counter`. `RefCell` narrows that requirement down to
just the one field.

### Cell: Copy Types Without Borrow Guards

`Cell<T>` is a narrower version of the same idea, restricted to `T: Copy`. Because the
value is copied in and out rather than borrowed, `Cell` never hands out a reference to
its contents, so there's no borrow state to track and no possibility of a borrow panic:

```rust
use std::cell::Cell;

struct Point {
    x: Cell<i32>,
    y: Cell<i32>,
}

fn main() {
    let p = Point { x: Cell::new(0), y: Cell::new(0) };
    p.x.set(p.x.get() + 1);
    p.y.set(p.y.get() + 2);
    println!("({}, {})", p.x.get(), p.y.get()); // (1, 2)
}
```

`.get()` copies the value out and `.set()` overwrites it; there's no `.borrow()` step in
between. Prefer `Cell` over `RefCell` whenever the contained type is small and `Copy`,
since it removes the possibility of a borrow panic entirely.

---

## Weak

Sharing ownership with `Rc` introduces a new failure mode that single ownership can't
have: a reference cycle. If two `Rc`-owned values point back at each other, each one's
strong count is kept above zero by the other, so neither is ever dropped, even after the
rest of the program has lost interest in them. This is a real memory leak, one the
borrow checker cannot catch, because it's a runtime property of the reference graph, not
a static one.

`Weak<T>` is the fix: it points at an `Rc`-managed allocation without incrementing its
*strong* count, only a separate *weak* count. A `Weak` reference does not keep the value
alive by itself. To use the value, you call `.upgrade()`, which returns `Option<Rc<T>>`:
`Some` if the value is still alive (strong count above zero), `None` if it's already
been dropped. That `Option` is the whole point: it forces you to handle the case where
the thing you're pointing at is gone.

The canonical use case is a parent/child tree, where a child needs to point back at its
parent. Children should keep their parent alive (a strong `Rc`), but the parent should
not be kept alive by its children pointing at it (a `Weak`), otherwise a parent and its
child hold each other's strong count above zero forever:

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade().is_some()); // false

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!(
        "leaf's parent value = {:?}",
        leaf.parent.borrow().upgrade().map(|n| n.value)
    ); // Some(5)

    println!(
        "branch strong = {}, weak = {}",
        Rc::strong_count(&branch),
        Rc::weak_count(&branch)
    ); // branch strong = 1, weak = 1

    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf)
    ); // leaf strong = 2, weak = 0
}
```

`branch`'s strong count is 1 (only the local `branch` variable owns it; `leaf`'s pointer
back to it is weak and doesn't count), while its weak count is 1 (`leaf`'s
`Weak<Node>`). `leaf`'s strong count is 2 (the local `leaf` variable, plus `branch`'s
`children` vector), so dropping `branch` first would drop its strong claim on `leaf`,
but `leaf` would survive via the local variable; dropping `leaf` first is fine too,
since `branch` never held a strong reference to it in the first place. Either order
terminates cleanly, which is exactly what the strong/weak split is for.

---

## Interior Mutability

"Interior mutability" is the name for the pattern `RefCell`, `Cell`, and (in
concurrent code) `Mutex`/`RwLock` all implement: mutating data through a type that is
itself only immutably (`&`) accessible. Ordinarily, `&T` in Rust means "this data won't
change while you're looking at it," and the compiler relies on that guarantee to make
other optimizations and reason about aliasing safely. A type with interior mutability
keeps that outer promise, callers still only ever see `&T`, while moving the actual
mutation, and the bookkeeping needed to keep it sound, inside the type itself.

Under the hood, these types use `UnsafeCell<T>`, the one construct in `std` that's
allowed to hand out a raw mutable pointer from a shared reference. `RefCell` wraps that
with the runtime borrow tracking shown above; `Cell` wraps it with copy-in/copy-out
instead of borrow guards. Reaching for `UnsafeCell` directly is a `unsafe`-Rust concern
covered in [Chapter 28](X.28-unsafe-rust.md); the point here is that `RefCell` and
`Cell` exist precisely so ordinary, safe code never has to.

The most common combination in single-threaded code is `Rc<RefCell<T>>`: `Rc` provides
the multiple owners, and `RefCell` provides the ability to mutate through each of them:

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct SharedData {
    value: i32,
}

fn main() {
    let shared = Rc::new(RefCell::new(SharedData { value: 1 }));

    let handle_a = Rc::clone(&shared);
    let handle_b = Rc::clone(&shared);

    handle_a.borrow_mut().value += 10;
    handle_b.borrow_mut().value += 100;

    println!("final = {:?}", shared.borrow()); // final = SharedData { value: 111 }
}
```

`handle_a` and `handle_b` are two independent `Rc` pointers to the same allocation, each
free to borrow it mutably in turn. Nothing here is `unsafe` and nothing bypasses Rust's
aliasing rule: the rule has just moved from a compile-time check on `&`/`&mut` to a
runtime check inside `RefCell`.

---

## Quick Reference

| Type | Ownership | Mutability | Thread-safe | Cost |
|---|---|---|---|---|
| `Box<T>` | Single | Via `&mut` (normal rules) | If `T: Send` | Heap allocation only |
| `Rc<T>` | Shared (refcounted) | Immutable by default | No | Non-atomic counter ops |
| `Arc<T>` | Shared (refcounted) | Immutable by default | Yes | Atomic counter ops |
| `RefCell<T>` | Single | Interior, runtime-checked | No | Borrow-state check per call |
| `Cell<T>` | Single | Interior, copy in/out | No | Copy per `get`/`set` |
| `Weak<T>` | Non-owning | N/A (borrows via `Rc`/`Arc`) | Matches `Rc`/`Arc` | `.upgrade()` check |

```rust
// Creating and using each:
let b: Box<i32>              = Box::new(5);
let rc: Rc<i32>               = Rc::new(5);
let arc: Arc<i32>             = Arc::new(5);
let cell: RefCell<i32>        = RefCell::new(5);
let c: Cell<i32>              = Cell::new(5);
let weak: Weak<i32>           = Rc::downgrade(&rc);

*b;                            // deref
let shared = Rc::clone(&rc);   // bump strong count
let shared = Arc::clone(&arc); // bump strong count (atomic)
*cell.borrow_mut() += 1;       // runtime-checked mutable borrow
c.set(c.get() + 1);            // copy out, copy in
weak.upgrade();                // Option<Rc<i32>>
```

---

## Common Patterns

- `Box<T>` for a recursive type or to erase a concrete type behind `dyn Trait`. See [Recursive Types](#recursive-types) and [Trait Objects](#trait-objects) above.
- `Rc<T>` for shared, read-mostly ownership within a single thread; `Arc<T>` for the same thing across threads. See [Rc and Arc](#rc-and-arc) above.
- `Rc<RefCell<T>>` (or `Arc<Mutex<T>>` across threads) when shared ownership and mutation are both needed. See [Interior Mutability](#interior-mutability) above.
- `Weak<T>` for a back-reference in a parent/child structure, so the child doesn't keep the parent alive forever. See [Weak](#weak) above.
- `Cell<T>` instead of `RefCell<T>` whenever the contained value is a small `Copy` type, to avoid any risk of a borrow panic. See [Cell: Copy Types Without Borrow Guards](#cell-copy-types-without-borrow-guards) above.

---

## Common Pitfalls

- Calling `.borrow_mut()` while another borrow of the same `RefCell` is still alive panics at runtime instead of failing to compile; the bug can hide until a specific call sequence triggers it. See [RefCell: Runtime-Checked Borrowing](#refcell-runtime-checked-borrowing) above.
- Two `Rc`-owned values that reference each other keep each other's strong count above zero forever, a genuine memory leak the borrow checker can't detect. Break the cycle with `Weak` on at least one side. See [Weak](#weak) above.
- Sending an `Rc<T>` across a thread boundary is a compile error, not a runtime one; the fix is `Arc<T>`, not `unsafe`. See [Arc: Rc for Multiple Threads](#arc-rc-for-multiple-threads) above.
- `Rc<T>` and `Arc<T>` only give out `&T`; without an inner `RefCell`/`Mutex`, the data they point to can never be mutated at all. See [How Rc::clone Works](#how-rcclone-works) above.

---

## Summary

`Box<T>`, `Rc<T>`/`Arc<T>`, and `RefCell<T>`/`Cell<T>` each relax one part of the default
ownership model from [Chapter 8](III.8-ownership-and-borrow-checker.md): heap allocation
for values of unknown or dynamic size, shared ownership counted at runtime instead of
enforced at compile time, and mutation through a shared reference checked at runtime
instead of compile time. `Weak<T>` complements `Rc`/`Arc` by providing a non-owning
reference, which is what makes it possible to build cyclic structures like parent/child
trees without leaking memory. The type table and constructor forms are collected in
[Quick Reference](#quick-reference) above.
