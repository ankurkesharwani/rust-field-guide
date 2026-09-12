# Unsafe Rust

## Table of Contents

1. [Introduction](#introduction)
2. [unsafe fn and Raw Pointers](#unsafe-fn-and-raw-pointers)
3. [The Five Superpowers](#the-five-superpowers)
4. [When You Actually Need It](#when-you-actually-need-it)
5. [A Safe Wrapper Example](#a-safe-wrapper-example)
6. [Quick Reference](#quick-reference)
7. [Common Patterns](#common-patterns)
8. [Common Pitfalls](#common-pitfalls)
9. [Summary](#summary)

---

## Introduction

This is a light-touch chapter by design. Ordinary Rust (what this whole reference has
been about until now) is called "safe Rust," and the compiler enforces its rules for
you: every reference is valid, every borrow follows the rules from [Chapter 8:
Ownership and the Borrow Checker](III.8-ownership-and-borrow-checker.md), and it's not
possible to read uninitialized memory or create a data race. `unsafe` opts a specific,
marked piece of code out of some of those compiler checks. It doesn't turn off the
borrow checker or type checking; it unlocks five additional operations (the "five
superpowers" below) that the compiler can't otherwise verify are safe.

The honest default is that you'll rarely write `unsafe` in application code. It exists
for a handful of real situations: calling into C through FFI, a data structure or
algorithm where a safety invariant is genuinely too complex for the borrow checker to
see (even though you can prove it by hand), or a hot path where the checked version's
overhead actually matters and has been measured. This chapter shows what the syntax
looks like, what the five superpowers are, and one short example of the pattern real
unsafe code follows: a safe function wrapping a small unsafe implementation.

---

## unsafe fn and Raw Pointers

Marking a function `unsafe fn` is a promise to its callers: "this function has
preconditions the compiler can't check, and it's your job to uphold them." Calling an
`unsafe fn` requires an `unsafe` block, which is Rust's way of making you explicitly
acknowledge that promise at the call site:

```rust
unsafe fn dangerous(ptr: *const i32) -> i32 {
    unsafe { *ptr }
}

fn main() {
    let n = 5;
    let ptr = &n as *const i32;
    println!("{}", unsafe { dangerous(ptr) });
}
```

Marking `dangerous` as `unsafe fn` doesn't make its body unsafe by magic. It's still
just a function; the `unsafe` block inside it is what actually allows the dereference
of a raw pointer on the next line. And marking the function `unsafe` doesn't make the
rest of `main` unsafe either. Calling `dangerous` from `main` still needs its own
`unsafe { ... }` wrapper, because unsafety in Rust is scoped to exactly the block that
opts in, not inherited by whatever function happens to contain it. This is deliberate:
it keeps the "trust me" parts of a codebase small and searchable (`grep -rn unsafe`
finds all of them), instead of an `unsafe fn` silently making its entire caller
suspect.

Raw pointers (`*const T` and `*mut T`) are the other half of this section. Unlike
references, they're allowed to be null, to dangle (point at memory that's been freed),
and to alias (a `*const T` and a `*mut T` can point at the same memory at the same
time, which the borrow checker would never allow for `&T` and `&mut T`). You can create
a raw pointer in safe code; only dereferencing one requires `unsafe`:

```rust
fn main() {
    let mut num = 5;

    let r1 = &raw const num;
    let r2 = &raw mut num;

    unsafe {
        println!("r1 is: {}", *r1);
        *r2 = 10;
        println!("r2 is: {}", *r2);
    }
}
```

Creating `r1` and `r2` is safe, even though they alias the same variable, because a raw
pointer that's never dereferenced can't cause a memory safety problem. The unsafety is
entirely in the `unsafe` block where they're read and written.

---

## The Five Superpowers

`unsafe` unlocks exactly five operations, and nothing else. Everything else about the
language (move semantics, generics, trait resolution, pattern matching) works exactly
the same inside an `unsafe` block as outside one.

1. Dereference a raw pointer (shown above).
2. Call an `unsafe fn` or an `unsafe` method (shown above).
3. Access or modify a mutable `static` variable.
4. Implement an `unsafe trait`.
5. Access a field of a `union`.

```rust
// 3. Mutable statics
static mut COUNTER: i32 = 0;

fn bump_counter() -> i32 {
    unsafe {
        COUNTER += 1;
        COUNTER
    }
}

// 4. Unsafe traits
unsafe trait Marker {}

struct Widget;
unsafe impl Marker for Widget {}
```

A mutable `static` is unsafe to touch because it's shared, mutable global state: two
threads reading and writing `COUNTER` at once is a data race, and nothing about the
type `static mut COUNTER: i32` tells the compiler that isn't happening. An `unsafe
trait` works the other direction: the trait's author is declaring that implementing it
carries an obligation the compiler can't check (`Send` and `Sync`, both from the
standard library, are the canonical examples: implementing either by hand is a promise
that the type really is safe to move or share across threads), and `unsafe impl` is the
implementer acknowledging that obligation.

---

## When You Actually Need It

In practice, almost none of this comes up in typical application code, because the
situations that call for `unsafe` are narrow:

- **FFI.** Calling a C function, or being called by one, crosses a boundary the Rust
  compiler has no visibility into, so the call itself has to be marked `unsafe`.
- **A hot path with measured overhead.** Bounds checks, reference-counting, and similar
  safe-Rust bookkeeping cost real cycles. Removing them with `unsafe` is worth doing
  only after profiling shows the cost matters, not as a default optimization.
- **Building a safe abstraction the borrow checker can't see through.** Some data
  structures (a doubly linked list, a lock-free queue, `Vec` itself) have an invariant
  that's true by construction but not expressible in terms the borrow checker
  understands. The fix is `unsafe` code inside the data structure, wrapped in a safe
  public API, which is exactly the pattern in the next section.

If none of these apply, the code you're writing almost certainly doesn't need
`unsafe`. Reaching for it to work around a borrow checker error is nearly always a sign
that the code should be restructured, not that the check is wrong.

---

## A Safe Wrapper Example

The standard library's own `[T]::split_at_mut` needs `unsafe` internally: it returns
two mutable slices into the same underlying array, which the borrow checker can't
verify are non-overlapping just from the function signature (as far as it can see,
you're producing two `&mut` borrows of the same memory, which safe Rust never
allows). The implementation proves non-overlap by hand instead, using raw pointers,
and exposes a completely safe function:

```rust
fn split_at_mut_demo(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
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

fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];
    let (a, b) = split_at_mut_demo(&mut v, 3);
    a[0] = 100;
    b[0] = 200;
    println!("{:?} {:?}", a, b); // [100, 2, 3] [200, 5, 6]
}
```

The `assert!(mid <= len)` before the `unsafe` block is doing real safety work, not just
input validation: it's the runtime check that makes the two `from_raw_parts_mut` calls
below it valid. `from_raw_parts_mut` itself has a documented contract (the pointer must
be non-null and valid for the given length, and the two ranges must not overlap), and
nothing in the function signature enforces that contract; it's on the caller, which
here means it's on the eleven lines of this function to get right, once, so that every
caller of `split_at_mut_demo` gets to stay in safe Rust.

That's the shape unsafe code takes in practice: small, contained, justified by an
invariant the author checked by hand, and wrapped behind a safe function so the
unsafety never leaks into the code that calls it.

---

## Quick Reference

| Superpower | Requires `unsafe` for |
|---|---|
| Raw pointers | Dereferencing (`*ptr`); creating one is safe |
| `unsafe fn` | Calling it (defining it doesn't need a block, but its body often will) |
| Mutable `static` | Reading or writing it |
| `unsafe trait` | Providing an `impl` for it |
| `union` | Accessing a field |

```rust
// Raw pointer types
let p1: *const T = /* ... */;
let p2: *mut T = /* ... */;

// Creating raw pointers to a place (safe)
let r1 = &raw const value;
let r2 = &raw mut value;

// Dereferencing (needs unsafe)
unsafe { *r1 }
```

---

## Common Patterns

- Wrap `unsafe` code behind a safe function whose signature and runtime checks (like `assert!`) uphold whatever invariant the `unsafe` block relies on, so callers never need `unsafe` themselves. See A Safe Wrapper Example above.
- Keep an `unsafe` block as small as possible, ideally just the one or two operations that actually need it, so the invariant it depends on is easy to state and check by reading that block alone.
- Reach for FFI, a profiled hot path, or a safety invariant the borrow checker can't see as the only real reasons to write `unsafe`. See When You Actually Need It above.

---

## Common Pitfalls

- `unsafe fn` doesn't make the function's body automatically checked or its caller automatically unsafe elsewhere; it only lets that one function's body use the five superpowers, and callers still need their own `unsafe` block. See unsafe fn and Raw Pointers above.
- A raw pointer can be null or dangling with no compiler warning; creating one is safe precisely because nothing has been proven about it yet, and all the risk is deferred to the dereference. See unsafe fn and Raw Pointers above.
- Reading or writing a mutable `static` from more than one thread is a data race with no compiler protection; `unsafe` lets you do it, it doesn't make it correct. See The Five Superpowers above.
- Treating `unsafe` as a way to silence a borrow checker error you don't understand, rather than a last resort for one of the narrow cases in When You Actually Need It, tends to convert a compile-time error into a runtime crash or, worse, silent memory corruption that only shows up later.

---

## Summary

`unsafe` unlocks exactly five operations the compiler otherwise forbids: dereferencing
a raw pointer, calling an `unsafe fn`, touching a mutable `static`, implementing an
`unsafe trait`, and reading a `union` field. Everything else about Rust still applies
inside an `unsafe` block. The idiomatic use of it is narrow (FFI, measured
performance work, or a safety invariant the borrow checker can't see) and it almost
always takes the shape shown in A Safe Wrapper Example: a small, justified `unsafe`
block encapsulated behind a safe public function, not scattered through ordinary
business logic.
