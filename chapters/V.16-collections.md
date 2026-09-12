# Collections

## Table of Contents

1. [Introduction](#introduction)
2. [Vec](#vec)
3. [HashMap](#hashmap)
4. [HashSet](#hashset)
5. [VecDeque](#vecdeque)
6. [BTreeMap](#btreemap)
7. [Choosing a Collection](#choosing-a-collection)
8. [Quick Reference](#quick-reference)
9. [Common Patterns](#common-patterns)
10. [Common Pitfalls](#common-pitfalls)
11. [Summary](#summary)

---

## Introduction

The standard library's collections all store a variable number of values, but they differ
in how those values are laid out in memory and what that layout makes cheap or expensive.
This chapter covers `Vec`, `HashMap`, `HashSet`, `VecDeque`, and `BTreeMap`: what each one
is built on, what operations it's fast at, and when to reach for it instead of the others.

---

## Vec

### Creating and Growing a Vec

```rust
let mut v: Vec<i32> = Vec::new();
v.push(1);
v.push(2);
v.push(3);

let v2 = vec![1, 2, 3]; // shorthand for the same thing
```

A `Vec<T>` is a growable array: a pointer, a length, and a capacity, all stored on the
stack, pointing at a single heap allocation that holds the elements contiguously. Because
the elements sit next to each other in memory, indexing is a direct offset calculation
and iterating is cache-friendly, which is why `Vec` is the default sequential collection
in Rust unless something more specific is needed.

### Indexing and Safe Access with `get`

```rust
let v = vec![10, 20, 30];

let third = v[2]; // panics if the index is out of bounds

match v.get(5) {
    Some(x) => println!("{x}"),
    None => println!("no element at index 5"),
}
```

`v[i]` and `v.get(i)` do the same bounds check internally, but they disagree on what to do
when the check fails. Indexing panics, which is the right choice when an out-of-bounds
index means a bug in your own code. `get` returns `Option<&T>` instead, which is the right
choice when the index comes from outside (user input, a parsed file, another system) and
being out of range is an expected outcome you need to handle rather than crash on.

### Iterating

```rust
let v = vec![1, 2, 3];
for x in &v {
    println!("{x}"); // x: &i32, v is still usable after the loop
}

let mut v2 = vec![1, 2, 3];
for x in &mut v2 {
    *x += 10; // x: &mut i32, mutates in place
}

let v3 = vec![1, 2, 3];
for x in v3 {
    println!("{x}"); // x: i32, v3 is consumed
}
```

The three forms of `for x in ...` correspond to the three ways a `Vec` can be iterated,
and Rust picks between them based on whether you loop over `&v`, `&mut v`, or `v` itself.
Looping over `&v` borrows each element, `&mut v` borrows each element mutably so you can
update it in place, and looping over `v` by value moves the vector in and hands you owned
elements, after which `v` can no longer be used. This is the same borrow-checker logic
that governs any value, applied through the `Iterator` machinery instead of spelled out
explicitly.

### Capacity and Reallocation

```rust
let mut v = Vec::with_capacity(10);
println!("len {} cap {}", v.len(), v.capacity()); // len 0 cap 10

let mut v2 = Vec::new();
for i in 0..5 {
    v2.push(i);
    println!("len={} cap={}", v2.len(), v2.capacity());
}
```

`len` is how many elements are actually stored; `capacity` is how many the current
allocation has room for before it needs to grow. Pushing past capacity doesn't reallocate
one slot at a time: the `Vec` allocates a new, larger buffer (typically by doubling),
copies the existing elements over, and frees the old buffer. The exact growth numbers are
an implementation detail and not something to depend on, but the amortized effect is that
`push` is cheap on average even though any individual call might trigger a reallocation.
`Vec::with_capacity` sidesteps repeated reallocation up front when you already know
roughly how many elements you'll end up storing.

### Retain and Dedup

```rust
let mut v = vec![1, 2, 3, 4, 5, 6];
v.retain(|&x| x % 2 == 0);
println!("{v:?}"); // [2, 4, 6]

let mut v2 = vec![1, 1, 2, 2, 3, 1];
v2.dedup();
println!("{v2:?}"); // [1, 2, 3, 1]
```

`retain` keeps only the elements for which the closure returns `true`, shifting the
survivors down in place rather than allocating a new vector. `dedup` removes
*consecutive* duplicate elements, which is why the trailing `1` in `v2` survives: it isn't
adjacent to the earlier `1`s. Sort the vector first if you want to remove every duplicate
regardless of position.

---

## HashMap

### Creating and Inserting

```rust
use std::collections::HashMap;

let mut scores: HashMap<String, i32> = HashMap::new();
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
```

A `HashMap<K, V>` stores key-value pairs and uses a hash of the key to jump almost
directly to where that pair lives, so insertion and lookup are both O(1) on average
rather than the O(n) scan a `Vec` of pairs would need.

### Ownership: a HashMap Takes Its Keys and Values

```rust
let mut map = HashMap::new();
let key = String::from("favorite color");
let value = String::from("blue");
map.insert(key, value);
// key and value have been moved into map; using them here would not compile
```

For types that don't implement `Copy`, like `String`, `insert` takes ownership of both
the key and the value rather than borrowing them. The map needs to own its contents for
as long as they live inside it, so this follows the same move rules as any other value
being handed off, just less visible because it happens inside a method call.

### Looking Up with `get`

```rust
let team_name = String::from("Blue");
match scores.get(&team_name) {
    Some(score) => println!("{score}"),
    None => println!("no such team"),
}
```

`get` takes a reference to the key and returns `Option<&V>`, so a missing key is an
ordinary `None` rather than a panic, unlike indexing a `Vec` out of bounds.

### The `entry` API

```rust
let text = "hello world wonderful world";
let mut word_count: HashMap<&str, i32> = HashMap::new();

for word in text.split_whitespace() {
    let count = word_count.entry(word).or_insert(0);
    *count += 1;
}
```

Counting words the naive way (check if the key exists, insert `0` if not, then increment)
looks up the key twice: once for `contains_key`, again for the increment. `entry` looks
the key up exactly once and returns a handle to that slot whether or not the key was
already present. `or_insert(0)` fills the slot with `0` if it was empty, and either way
returns a `&mut i32` pointing at the value, which is what `*count += 1` mutates directly.
This is the idiomatic way to do "update if present, insert a default otherwise" in one
pass.

### Iterating a HashMap

```rust
for (key, value) in &scores {
    println!("{key}: {value}");
}
```

Iteration order over a `HashMap` is not the insertion order and is not guaranteed to be
consistent even between two runs of the same program: it falls out of how the keys happen
to hash, and Rust deliberately randomizes the hasher's seed per map to avoid code
depending on any particular order. Reach for `BTreeMap` below when iteration order
matters.

---

## HashSet

### A HashMap with No Values

```rust
use std::collections::HashSet;

let mut books: HashSet<String> = HashSet::new();
books.insert(String::from("A Game of Thrones"));
let inserted_again = books.insert(String::from("The Hobbit"));
println!("inserted again: {inserted_again}");

println!("{}", books.contains("The Hobbit"));
```

`HashSet<T>` is implemented as a `HashMap<T, ()>`: it gets the same O(1) average
membership check as `HashMap`'s key lookup, just without storing a value alongside each
key. `insert` returns `true` when the value was newly added and `false` when it was
already present, which doubles as a one-line "was this a duplicate?" check.

### Set Algebra

```rust
let a: HashSet<i32> = [1, 2, 3, 4].into_iter().collect();
let b: HashSet<i32> = [3, 4, 5, 6].into_iter().collect();

let intersection: Vec<&i32> = a.intersection(&b).collect(); // in both: 3, 4
let union: Vec<&i32> = a.union(&b).collect();               // in either: 1, 2, 3, 4, 5, 6
let difference: Vec<&i32> = a.difference(&b).collect();     // in a but not b: 1, 2
```

`intersection`, `union`, and `difference` all return iterators rather than new sets, so
you decide how to collect the result: into a `Vec` for a fixed snapshot, or straight into
another `HashSet` if you need to keep doing set operations on it. Each one walks the
smaller of the two sets internally and probes the other, so they're proportional to the
size of the sets rather than needing a nested loop.

---

## VecDeque

### A Ring Buffer, Not a Linked List

```rust
use std::collections::VecDeque;

let mut queue: VecDeque<i32> = VecDeque::new();
queue.push_back(1);
queue.push_back(2);
queue.push_front(0);
println!("{queue:?}"); // [0, 1, 2]
```

`VecDeque<T>` ("double-ended queue") is a growable ring buffer over a single contiguous
allocation, not a linked list: it keeps a start offset and wraps around the end of the
buffer, which is what lets it push and pop from either end in O(1) without shifting every
other element, while still keeping the cache-friendly contiguous storage a `Vec` has.

### Pushing and Popping from Both Ends

```rust
println!("{:?}", queue.pop_front()); // Some(0)
println!("{queue:?}");               // [1, 2]
```

`push_back`/`pop_back` behave like a `Vec`'s `push`/`pop`; `push_front`/`pop_front` are
the operations a plain `Vec` doesn't offer efficiently. That's the whole reason
`VecDeque` exists: use it for FIFO queues, sliding windows, or anything that needs to grow
or shrink from the front as often as the back.

### VecDeque vs. Vec for Front Insertion

```rust
let mut v = vec![2, 3, 4];
v.insert(0, 1); // O(n): every existing element shifts right by one

let mut dq: VecDeque<i32> = VecDeque::from([2, 3, 4]);
dq.push_front(1); // O(1) amortized: no shifting
```

Both end up holding `[1, 2, 3, 4]`, but `Vec::insert(0, ...)` has to move every existing
element over by one slot because a `Vec` guarantees its contents are contiguous starting
at offset zero. `VecDeque` has no such requirement, so `push_front` just moves the start
offset backward (wrapping around the buffer if needed) with no shifting at all. If a
collection is going to see frequent front insertions or removals, that difference is the
reason to pick `VecDeque` over `Vec` from the start.

---

## BTreeMap

### Ordered Iteration

```rust
use std::collections::BTreeMap;

let mut map: BTreeMap<String, i32> = BTreeMap::new();
map.insert(String::from("banana"), 3);
map.insert(String::from("apple"), 5);
map.insert(String::from("cherry"), 1);

for (key, value) in &map {
    println!("{key}: {value}"); // apple, banana, cherry: sorted by key
}
```

`BTreeMap<K, V>` keeps its entries sorted by key at all times, backed by a B-tree rather
than a hash table. Lookup and insertion are O(log n) instead of `HashMap`'s average O(1),
which is the price for getting a deterministic, sorted iteration order for free. Reach for
it when you need that ordering (printing a sorted report, walking keys in range) rather
than raw lookup speed.

### Range Queries

```rust
let mut map: BTreeMap<i32, &str> = BTreeMap::new();
map.insert(1, "one");
map.insert(5, "five");
map.insert(10, "ten");
map.insert(15, "fifteen");

for (key, value) in map.range(3..12) {
    println!("{key}: {value}"); // 5: five, 10: ten
}
```

`range` walks only the keys inside the given bound, taking advantage of the tree's sorted
structure to skip everything outside it rather than filtering a full scan. `HashMap` has
no equivalent, because there's no ordering relationship between its keys to walk.

---

## Choosing a Collection

| If you need to... | Reach for |
|---|---|
| Store items in insertion order, indexed by position | `Vec` |
| Look up values by key, don't care about order | `HashMap` |
| Track membership, don't care about order | `HashSet` |
| Push/pop from both ends (a queue, a sliding window) | `VecDeque` |
| Look up by key *and* keep keys sorted, or query ranges | `BTreeMap` |

Start with `Vec` for sequences and `HashMap` for lookups; they cover most cases. Move to
`VecDeque` or `BTreeMap` only once you have a concrete need (front insertion, sorted
iteration, range queries) that `Vec`/`HashMap` can't do efficiently.

---

## Quick Reference

| Collection | Backing structure | Lookup | Insert | Ordered? |
|---|---|---|---|---|
| `Vec<T>` | Contiguous array | O(n) by value, O(1) by index | O(1) amortized at the end, O(n) elsewhere | Insertion order |
| `HashMap<K, V>` | Hash table | O(1) average | O(1) average | No |
| `HashSet<T>` | Hash table (`HashMap<T, ()>`) | O(1) average | O(1) average | No |
| `VecDeque<T>` | Ring buffer over a contiguous array | O(n) by value, O(1) by index | O(1) amortized at either end | Insertion order |
| `BTreeMap<K, V>` | B-tree | O(log n) | O(log n) | Sorted by key |

```rust
// Construction shorthand
let v: Vec<i32> = vec![1, 2, 3];
let m: HashMap<&str, i32> = HashMap::from([("a", 1), ("b", 2)]);
let s: HashSet<i32> = HashSet::from([1, 2, 3]);
let d: VecDeque<i32> = VecDeque::from([1, 2, 3]);
let b: BTreeMap<&str, i32> = BTreeMap::from([("a", 1), ("b", 2)]);
```

---

## Common Patterns

- Use `entry(key).or_insert(default)` (or `or_insert_with`, `or_default`) to update-or-initialize a `HashMap`/`BTreeMap` value in a single lookup. See The `entry` API above.
- Use `iter().collect::<HashSet<_>>()` to deduplicate a sequence, or `HashSet::intersection`/`union`/`difference` to compare two sequences by membership. See Set Algebra above.
- Use `VecDeque` for a queue, and `Vec` plus `.pop()` (which removes from the end) for a stack; both are O(1). See A Ring Buffer, Not a Linked List above.
- Use `BTreeMap::range` to pull a contiguous slice of sorted keys without scanning the whole map. See Range Queries above.

---

## Common Pitfalls

- Indexing a `Vec` with `v[i]` panics on an out-of-bounds index; use `v.get(i)` when the index isn't guaranteed to be valid. See Indexing and Safe Access with `get` above.
- Iterating a `HashMap` or `HashSet` gives no guarantee about order, and that order can differ between runs of the same program. Use `BTreeMap`/`BTreeSet`, or sort the collected entries, when order matters. See Iterating a HashMap above.
- Inserting at the front of a `Vec` (`v.insert(0, x)`) is O(n) because every element has to shift; use `VecDeque::push_front` if front insertion happens often. See VecDeque vs. Vec for Front Insertion above.
- `dedup` only removes *consecutive* duplicates; sort first if duplicates can be spread throughout the collection. See Retain and Dedup above.

---

## Summary

`Vec` is the default sequential collection: contiguous, cache-friendly, cheap to push at
the end. `HashMap`/`HashSet` trade ordering for O(1) average lookup. `VecDeque` adds cheap
front operations on top of `Vec`'s contiguous layout. `BTreeMap` trades `HashMap`'s raw
lookup speed for sorted, rangeable iteration. The Quick Reference table above summarizes
the complexity and ordering trade-offs across all five.
