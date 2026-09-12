# Iterators in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [The Iterator Trait](#the-iterator-trait)
3. [Creating Iterators](#creating-iterators)
4. [Iterator Adaptors](#iterator-adaptors)
5. [Consuming Iterators](#consuming-iterators)
6. [Implementing Custom Iterators](#implementing-custom-iterators)
7. [Iterator Performance](#iterator-performance)
8. [Double-Ended Iterators](#double-ended-iterators)
9. [Exact Size Iterators](#exact-size-iterators)
10. [Common Patterns](#common-patterns)
11. [Quick Reference](#quick-reference)
12. [Common Pitfalls](#common-pitfalls)
13. [Summary](#summary)

---

## Introduction

Iterators are a core abstraction in Rust that provide a way to process sequences of elements. They are:

- **Lazy**: No computation until consumed
- **Zero-cost**: Compiled to efficient machine code
- **Composable**: Chain multiple operations
- **Safe**: Type-checked and memory-safe

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Create iterator (lazy - no work done yet)
    let iter = numbers.iter();

    // Consume iterator (work happens here)
    let sum: i32 = iter.sum();
    println!("Sum: {}", sum); // 15
}
```

---

## The Iterator Trait

### Definition

```rust
pub trait Iterator {
    type Item;  // Associated type: the type of elements yielded

    fn next(&mut self) -> Option<Self::Item>;

    // Many provided methods...
}
```

`next` is the only method a type has to implement; every adaptor and consumer covered in
this chapter (`map`, `filter`, `collect`, `sum`, and dozens more) is a provided method
with a default implementation defined purely in terms of repeated calls to `next`. That's
why implementing `Iterator` for a custom type, shown later in this chapter, only ever
requires writing one method to unlock the entire method set.

### How next() Works

```rust
fn main() {
    let v = vec![1, 2, 3];
    let mut iter = v.iter();

    println!("{:?}", iter.next()); // Some(&1)
    println!("{:?}", iter.next()); // Some(&2)
    println!("{:?}", iter.next()); // Some(&3)
    println!("{:?}", iter.next()); // None
    println!("{:?}", iter.next()); // None (stays None forever)
}
```

### Manual Iteration

```rust
fn main() {
    let v = vec![1, 2, 3];
    let mut iter = v.iter();

    while let Some(value) = iter.next() {
        println!("{}", value);
    }

    // Equivalent to for loop
    for value in v.iter() {
        println!("{}", value);
    }
}
```

---

## Creating Iterators

### From Collections

```rust
fn main() {
    let vec = vec![1, 2, 3];
    let arr = [1, 2, 3];
    let slice = &arr[..];

    // iter() - immutable references (&T)
    for x in vec.iter() {
        println!("{}", x);  // x is &i32
    }

    // iter_mut() - mutable references (&mut T)
    let mut vec = vec![1, 2, 3];
    for x in vec.iter_mut() {
        *x *= 2;  // x is &mut i32
    }

    // into_iter() - owned values (T)
    let vec = vec![1, 2, 3];
    for x in vec.into_iter() {
        println!("{}", x);  // x is i32, vec is consumed
    }
}
```

### IntoIterator Trait

```rust
fn main() {
    let vec = vec![1, 2, 3];

    // These are equivalent:
    for x in &vec { }      // calls vec.iter()
    for x in &mut vec { }  // calls vec.iter_mut()
    for x in vec { }       // calls vec.into_iter()
}
```

A `for` loop is sugar for calling `.into_iter()` on whatever follows `in` and then
repeatedly calling `.next()` on the result. `Vec<T>` provides three different
`IntoIterator` implementations, one for `Vec<T>` itself, one for `&Vec<T>`, and one for
`&mut Vec<T>`, each producing a different item type (`T`, `&T`, `&mut T`), which is why
the form you write after `in` (the vec by value, or a `&`/`&mut` reference to it) decides
whether the loop borrows or consumes it.

### Range Iterators

```rust
fn main() {
    // Exclusive range
    for i in 0..5 {
        println!("{}", i);  // 0, 1, 2, 3, 4
    }

    // Inclusive range
    for i in 0..=5 {
        println!("{}", i);  // 0, 1, 2, 3, 4, 5
    }

    // Characters
    for c in 'a'..='z' {
        print!("{}", c);  // abcdefghijklmnopqrstuvwxyz
    }
}
```

### Generator Functions

```rust
use std::iter;

fn main() {
    // repeat: infinite iterator of same value
    let ones: Vec<i32> = iter::repeat(1).take(5).collect();
    println!("{:?}", ones);  // [1, 1, 1, 1, 1]

    // repeat_with: infinite iterator from closure
    let mut count = 0;
    let counts: Vec<i32> = iter::repeat_with(|| {
        count += 1;
        count
    }).take(5).collect();
    println!("{:?}", counts);  // [1, 2, 3, 4, 5]

    // once: single element
    let single: Vec<i32> = iter::once(42).collect();
    println!("{:?}", single);  // [42]

    // empty: no elements
    let empty: Vec<i32> = iter::empty().collect();
    println!("{:?}", empty);  // []

    // from_fn: generate from closure
    let mut state = 0;
    let generated: Vec<i32> = iter::from_fn(|| {
        state += 1;
        if state <= 3 { Some(state) } else { None }
    }).collect();
    println!("{:?}", generated);  // [1, 2, 3]

    // successors: each element from previous
    let powers: Vec<i32> = iter::successors(Some(1), |&n| {
        if n < 100 { Some(n * 2) } else { None }
    }).collect();
    println!("{:?}", powers);  // [1, 2, 4, 8, 16, 32, 64]
}
```

### String Iterators

```rust
fn main() {
    let s = "hello world";

    // chars(): Unicode scalar values
    for c in s.chars() {
        println!("{}", c);
    }

    // bytes(): raw bytes
    for b in s.bytes() {
        println!("{}", b);
    }

    // char_indices(): (index, char)
    for (i, c) in s.char_indices() {
        println!("{}: {}", i, c);
    }

    // lines(): split by newlines
    let text = "line 1\nline 2\nline 3";
    for line in text.lines() {
        println!("{}", line);
    }

    // split(): by pattern
    for word in s.split(' ') {
        println!("{}", word);
    }
}
```

### HashMap and HashSet Iterators

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    let mut map = HashMap::new();
    map.insert("a", 1);
    map.insert("b", 2);

    // Iterate over key-value pairs
    for (key, value) in &map {
        println!("{}: {}", key, value);
    }

    // Iterate over keys only
    for key in map.keys() {
        println!("{}", key);
    }

    // Iterate over values only
    for value in map.values() {
        println!("{}", value);
    }

    // Mutable values
    for value in map.values_mut() {
        *value *= 2;
    }

    // HashSet
    let set: HashSet<i32> = [1, 2, 3].into();
    for value in &set {
        println!("{}", value);
    }
}
```

---

## Iterator Adaptors

Iterator adaptors create new iterators from existing ones. They are lazy.

### map

Transform each element:

```rust
fn main() {
    let numbers = vec![1, 2, 3];

    let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
    println!("{:?}", doubled);  // [2, 4, 6]

    // Type transformation
    let strings: Vec<String> = numbers.iter().map(|x| x.to_string()).collect();
    println!("{:?}", strings);  // ["1", "2", "3"]
}
```

### filter

Keep elements matching predicate:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6];

    let evens: Vec<&i32> = numbers.iter().filter(|x| *x % 2 == 0).collect();
    println!("{:?}", evens);  // [2, 4, 6]
}
```

### filter_map

Filter and transform in one step:

```rust
fn main() {
    let strings = vec!["1", "two", "3", "four", "5"];

    // Only keeps successfully parsed numbers
    let numbers: Vec<i32> = strings
        .iter()
        .filter_map(|s| s.parse::<i32>().ok())
        .collect();
    println!("{:?}", numbers);  // [1, 3, 5]
}
```

### flat_map

Map and flatten:

```rust
fn main() {
    let words = vec!["hello", "world"];

    let chars: Vec<char> = words.iter().flat_map(|s| s.chars()).collect();
    println!("{:?}", chars);  // ['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd']
}
```

### flatten

Flatten nested iterators:

```rust
fn main() {
    let nested = vec![vec![1, 2], vec![3, 4], vec![5]];

    let flat: Vec<i32> = nested.into_iter().flatten().collect();
    println!("{:?}", flat);  // [1, 2, 3, 4, 5]

    // Flatten Options
    let options = vec![Some(1), None, Some(3)];
    let values: Vec<i32> = options.into_iter().flatten().collect();
    println!("{:?}", values);  // [1, 3]
}
```

### enumerate

Add indices:

```rust
fn main() {
    let letters = vec!['a', 'b', 'c'];

    for (index, letter) in letters.iter().enumerate() {
        println!("{}: {}", index, letter);
    }
}
```

### zip

Combine two iterators:

```rust
fn main() {
    let names = vec!["Alice", "Bob", "Charlie"];
    let ages = vec![30, 25, 35];

    let people: Vec<_> = names.iter().zip(ages.iter()).collect();
    println!("{:?}", people);  // [("Alice", 30), ("Bob", 25), ("Charlie", 35)]

    // Different lengths: stops at shorter
    let short = vec![1, 2];
    let long = vec!['a', 'b', 'c', 'd'];
    let zipped: Vec<_> = short.iter().zip(long.iter()).collect();
    println!("{:?}", zipped);  // [(1, 'a'), (2, 'b')]
}
```

### chain

Concatenate iterators:

```rust
fn main() {
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];

    let combined: Vec<&i32> = a.iter().chain(b.iter()).collect();
    println!("{:?}", combined);  // [1, 2, 3, 4, 5, 6]
}
```

### take and skip

Limit iteration:

```rust
fn main() {
    let numbers: Vec<i32> = (1..=10).collect();

    let first_three: Vec<_> = numbers.iter().take(3).collect();
    println!("{:?}", first_three);  // [1, 2, 3]

    let after_three: Vec<_> = numbers.iter().skip(3).collect();
    println!("{:?}", after_three);  // [4, 5, 6, 7, 8, 9, 10]
}
```

### take_while and skip_while

Limit by predicate:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 10, 4, 5];

    let small: Vec<_> = numbers.iter().take_while(|&&x| x < 5).collect();
    println!("{:?}", small);  // [1, 2, 3]

    let after_big: Vec<_> = numbers.iter().skip_while(|&&x| x < 5).collect();
    println!("{:?}", after_big);  // [10, 4, 5]
}
```

### peekable

Look ahead without consuming:

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    let mut iter = numbers.iter().peekable();

    // Peek at next without consuming
    println!("Peek: {:?}", iter.peek());  // Some(&1)
    println!("Next: {:?}", iter.next());  // Some(&1)
    println!("Next: {:?}", iter.next());  // Some(&2)

    // peek_mut: modify peeked value
    let mut numbers = vec![1, 2, 3];
    let mut iter = numbers.iter_mut().peekable();
    if let Some(x) = iter.peek_mut() {
        **x = 100;
    }
}
```

### fuse

Ensure None forever after first None:

```rust
fn main() {
    // Most iterators already fuse, but some custom ones might not
    struct Flaky(bool);

    impl Iterator for Flaky {
        type Item = i32;
        fn next(&mut self) -> Option<i32> {
            if self.0 {
                self.0 = false;
                None
            } else {
                self.0 = true;
                Some(1)
            }
        }
    }

    let mut iter = Flaky(false);
    println!("{:?}", iter.next());  // Some(1)
    println!("{:?}", iter.next());  // None
    println!("{:?}", iter.next());  // Some(1) (not fused!)

    let mut fused = Flaky(false).fuse();
    println!("{:?}", fused.next());  // Some(1)
    println!("{:?}", fused.next());  // None
    println!("{:?}", fused.next());  // None (stays None)
}
```

### cycle

Repeat infinitely:

```rust
fn main() {
    let pattern = vec![1, 2, 3];

    let repeated: Vec<_> = pattern.iter().cycle().take(8).collect();
    println!("{:?}", repeated);  // [1, 2, 3, 1, 2, 3, 1, 2]
}
```

### inspect

Debug without consuming:

```rust
fn main() {
    let result: Vec<i32> = (1..=5)
        .inspect(|x| println!("Before filter: {}", x))
        .filter(|x| x % 2 == 0)
        .inspect(|x| println!("After filter: {}", x))
        .collect();
}
```

### scan

Stateful transformation:

```rust
fn main() {
    // Running sum
    let numbers = vec![1, 2, 3, 4, 5];
    let running_sum: Vec<i32> = numbers
        .iter()
        .scan(0, |state, &x| {
            *state += x;
            Some(*state)
        })
        .collect();
    println!("{:?}", running_sum);  // [1, 3, 6, 10, 15]
}
```

### step_by

Skip elements:

```rust
fn main() {
    let every_other: Vec<i32> = (0..10).step_by(2).collect();
    println!("{:?}", every_other);  // [0, 2, 4, 6, 8]
}
```

---

## Consuming Iterators

Consumers drive iteration and produce a final value.

### collect

Convert to collection:

```rust
fn main() {
    let iter = 1..=5;

    // To Vec
    let vec: Vec<i32> = iter.clone().collect();

    // To HashSet
    use std::collections::HashSet;
    let set: HashSet<i32> = iter.clone().collect();

    // To HashMap (from tuples)
    use std::collections::HashMap;
    let pairs = vec![("a", 1), ("b", 2)];
    let map: HashMap<&str, i32> = pairs.into_iter().collect();

    // To String
    let chars = vec!['h', 'e', 'l', 'l', 'o'];
    let string: String = chars.into_iter().collect();
}
```

### collect with Result

```rust
fn main() {
    // All Ok -> Ok(Vec)
    let strings = vec!["1", "2", "3"];
    let numbers: Result<Vec<i32>, _> = strings.iter().map(|s| s.parse()).collect();
    println!("{:?}", numbers);  // Ok([1, 2, 3])

    // Any Err -> first Err
    let strings = vec!["1", "x", "3"];
    let numbers: Result<Vec<i32>, _> = strings.iter().map(|s| s.parse()).collect();
    println!("{:?}", numbers);  // Err(ParseIntError)
}
```

### fold

Accumulate values:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Sum
    let sum = numbers.iter().fold(0, |acc, x| acc + x);
    println!("Sum: {}", sum);  // 15

    // Product
    let product = numbers.iter().fold(1, |acc, x| acc * x);
    println!("Product: {}", product);  // 120

    // Build string
    let joined = numbers.iter().fold(String::new(), |acc, x| {
        if acc.is_empty() {
            x.to_string()
        } else {
            format!("{}, {}", acc, x)
        }
    });
    println!("Joined: {}", joined);  // "1, 2, 3, 4, 5"
}
```

### reduce

Like fold but uses first element as initial:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let max = numbers.iter().reduce(|a, b| if a > b { a } else { b });
    println!("{:?}", max);  // Some(&5)

    let empty: Vec<i32> = vec![];
    let result = empty.iter().reduce(|a, b| if a > b { a } else { b });
    println!("{:?}", result);  // None
}
```

### sum and product

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let sum: i32 = numbers.iter().sum();
    println!("Sum: {}", sum);  // 15

    let product: i32 = numbers.iter().product();
    println!("Product: {}", product);  // 120
}
```

### count

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let total = numbers.iter().count();
    println!("Count: {}", total);  // 5

    let even_count = numbers.iter().filter(|x| *x % 2 == 0).count();
    println!("Even count: {}", even_count);  // 2
}
```

### min and max

```rust
fn main() {
    let numbers = vec![3, 1, 4, 1, 5, 9];

    let min = numbers.iter().min();
    let max = numbers.iter().max();
    println!("Min: {:?}, Max: {:?}", min, max);  // Some(&1), Some(&9)

    // By key
    let words = vec!["apple", "pie", "strawberry"];
    let longest = words.iter().max_by_key(|s| s.len());
    println!("Longest: {:?}", longest);  // Some(&"strawberry")

    // Custom comparison
    let max = numbers.iter().max_by(|a, b| b.cmp(a));  // Actually min
    println!("{:?}", max);  // Some(&1)
}
```

### find

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let first_even = numbers.iter().find(|&&x| x % 2 == 0);
    println!("{:?}", first_even);  // Some(&2)

    let first_big = numbers.iter().find(|&&x| x > 10);
    println!("{:?}", first_big);  // None

    // find_map: find and transform
    let strings = vec!["a", "1", "b", "2"];
    let first_number = strings.iter().find_map(|s| s.parse::<i32>().ok());
    println!("{:?}", first_number);  // Some(1)
}
```

### position and rposition

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 4, 3, 2, 1];

    let pos = numbers.iter().position(|&x| x == 3);
    println!("First 3 at: {:?}", pos);  // Some(2)

    let rpos = numbers.iter().rposition(|&x| x == 3);
    println!("Last 3 at: {:?}", rpos);  // Some(6)
}
```

### any and all

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let has_even = numbers.iter().any(|&x| x % 2 == 0);
    println!("Has even: {}", has_even);  // true

    let all_positive = numbers.iter().all(|&x| x > 0);
    println!("All positive: {}", all_positive);  // true

    let all_even = numbers.iter().all(|&x| x % 2 == 0);
    println!("All even: {}", all_even);  // false
}
```

### partition

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let (evens, odds): (Vec<i32>, Vec<i32>) = numbers
        .into_iter()
        .partition(|&x| x % 2 == 0);

    println!("Evens: {:?}", evens);  // [2, 4]
    println!("Odds: {:?}", odds);    // [1, 3, 5]
}
```

### for_each

```rust
fn main() {
    let numbers = vec![1, 2, 3];

    numbers.iter().for_each(|x| println!("{}", x));

    // Equivalent to
    for x in numbers.iter() {
        println!("{}", x);
    }
}
```

### nth

Get element at index:

```rust
fn main() {
    let mut iter = (0..10);

    println!("{:?}", iter.nth(2));  // Some(2)
    println!("{:?}", iter.nth(2));  // Some(5) - continues from where it left off
    println!("{:?}", iter.nth(10)); // None
}
```

### last

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let last = numbers.iter().last();
    println!("{:?}", last);  // Some(&5)
}
```

---

## Implementing Custom Iterators

### Basic Implementation

```rust
struct Counter {
    count: u32,
    max: u32,
}

impl Counter {
    fn new(max: u32) -> Counter {
        Counter { count: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {
    let counter = Counter::new(5);

    for n in counter {
        println!("{}", n);  // 1, 2, 3, 4, 5
    }

    // Use iterator methods
    let sum: u32 = Counter::new(5).sum();
    println!("Sum: {}", sum);  // 15
}
```

### Iterator Over Struct Fields

```rust
struct Point {
    x: i32,
    y: i32,
    z: i32,
}

struct PointIter<'a> {
    point: &'a Point,
    index: usize,
}

impl<'a> Iterator for PointIter<'a> {
    type Item = i32;

    fn next(&mut self) -> Option<Self::Item> {
        let result = match self.index {
            0 => Some(self.point.x),
            1 => Some(self.point.y),
            2 => Some(self.point.z),
            _ => None,
        };
        self.index += 1;
        result
    }
}

impl Point {
    fn iter(&self) -> PointIter {
        PointIter { point: self, index: 0 }
    }
}

fn main() {
    let point = Point { x: 1, y: 2, z: 3 };

    for coord in point.iter() {
        println!("{}", coord);  // 1, 2, 3
    }
}
```

### Fibonacci Iterator

```rust
struct Fibonacci {
    curr: u64,
    next: u64,
}

impl Fibonacci {
    fn new() -> Fibonacci {
        Fibonacci { curr: 0, next: 1 }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        let current = self.curr;
        self.curr = self.next;
        self.next = current + self.next;
        Some(current)
    }
}

fn main() {
    let fibs: Vec<u64> = Fibonacci::new().take(10).collect();
    println!("{:?}", fibs);  // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
}
```

### IntoIterator Implementation

```rust
struct MyCollection {
    items: Vec<i32>,
}

impl IntoIterator for MyCollection {
    type Item = i32;
    type IntoIter = std::vec::IntoIter<i32>;

    fn into_iter(self) -> Self::IntoIter {
        self.items.into_iter()
    }
}

impl<'a> IntoIterator for &'a MyCollection {
    type Item = &'a i32;
    type IntoIter = std::slice::Iter<'a, i32>;

    fn into_iter(self) -> Self::IntoIter {
        self.items.iter()
    }
}

fn main() {
    let collection = MyCollection { items: vec![1, 2, 3] };

    // Works in for loop
    for item in &collection {
        println!("{}", item);
    }

    for item in collection {  // Consumes collection
        println!("{}", item);
    }
}
```

---

## Iterator Performance

### Zero-Cost Abstraction

Iterators compile to the same code as hand-written loops:

```rust
fn main() {
    let numbers: Vec<i32> = (0..1000).collect();

    // Iterator style
    let sum1: i32 = numbers.iter().map(|x| x * 2).sum();

    // Loop style (compiles to same code)
    let mut sum2 = 0;
    for x in &numbers {
        sum2 += x * 2;
    }

    assert_eq!(sum1, sum2);
}
```

### Lazy Evaluation Benefits

```rust
fn main() {
    let numbers: Vec<i32> = (0..1000000).collect();

    // Only processes first 3 that match
    let first_three: Vec<i32> = numbers
        .iter()
        .filter(|x| *x % 2 == 0)
        .map(|x| x * 2)
        .take(3)
        .cloned()
        .collect();

    println!("{:?}", first_three);  // [0, 4, 8]
}
```

### Avoiding Allocation

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Creates intermediate Vec (avoid)
    let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
    let filtered: Vec<&i32> = doubled.iter().filter(|x| **x > 5).collect();

    // No intermediate allocation (preferred)
    let result: Vec<i32> = numbers
        .iter()
        .map(|x| x * 2)
        .filter(|x| *x > 5)
        .collect();
}
```

---

## Double-Ended Iterators

An iterator over a `Vec` or slice knows both ends of its remaining elements, so it can
hand them out from the front with `next()` or from the back with `next_back()`, in any
order, on the same iterator. `rev()` is built on this: it's just an adaptor that swaps
which end `next()` pulls from.

### DoubleEndedIterator Trait

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let mut iter = numbers.iter();

    println!("{:?}", iter.next());      // Some(&1)
    println!("{:?}", iter.next_back()); // Some(&5)
    println!("{:?}", iter.next());      // Some(&2)
    println!("{:?}", iter.next_back()); // Some(&4)
    println!("{:?}", iter.next());      // Some(&3)
    println!("{:?}", iter.next());      // None
}
```

### rev()

Reverse iteration:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    for n in numbers.iter().rev() {
        println!("{}", n);  // 5, 4, 3, 2, 1
    }

    // Collect reversed
    let reversed: Vec<_> = numbers.iter().rev().collect();
    println!("{:?}", reversed);  // [&5, &4, &3, &2, &1]
}
```

### Implementing DoubleEndedIterator

```rust
struct CountUpDown {
    front: i32,
    back: i32,
}

impl CountUpDown {
    fn new(start: i32, end: i32) -> CountUpDown {
        CountUpDown { front: start, back: end }
    }
}

impl Iterator for CountUpDown {
    type Item = i32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.front <= self.back {
            let result = self.front;
            self.front += 1;
            Some(result)
        } else {
            None
        }
    }
}

impl DoubleEndedIterator for CountUpDown {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.front <= self.back {
            let result = self.back;
            self.back -= 1;
            Some(result)
        } else {
            None
        }
    }
}

fn main() {
    let iter = CountUpDown::new(1, 5);

    // Forward
    let forward: Vec<_> = CountUpDown::new(1, 5).collect();
    println!("{:?}", forward);  // [1, 2, 3, 4, 5]

    // Backward
    let backward: Vec<_> = CountUpDown::new(1, 5).rev().collect();
    println!("{:?}", backward);  // [5, 4, 3, 2, 1]
}
```

---

## Exact Size Iterators

Most iterators can only guess at how many elements remain (`filter`, for instance, won't
know until it has checked every element), but some, like a `Vec`'s, know the exact count
up front. `ExactSizeIterator` exposes that as `.len()` and lets callers like `.collect()`
pre-allocate the target collection to the right capacity instead of growing it
incrementally as elements arrive.

### ExactSizeIterator Trait

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let iter = numbers.iter();

    // Known size
    println!("Length: {}", iter.len());  // 5

    // Size hint always accurate
    println!("Size hint: {:?}", iter.size_hint());  // (5, Some(5))
}
```

### size_hint()

For iterators without exact size:

```rust
fn main() {
    // Filter doesn't know final size
    let iter = (0..100).filter(|x| x % 2 == 0);

    let (min, max) = iter.size_hint();
    println!("Min: {}, Max: {:?}", min, max);  // 0, Some(100)
}
```

---

## Common Patterns

### Windowing

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Sliding windows
    for window in numbers.windows(3) {
        println!("{:?}", window);
    }
    // [1, 2, 3], [2, 3, 4], [3, 4, 5]

    // Chunks
    for chunk in numbers.chunks(2) {
        println!("{:?}", chunk);
    }
    // [1, 2], [3, 4], [5]

    // Exact chunks
    for chunk in numbers.chunks_exact(2) {
        println!("{:?}", chunk);
    }
    // [1, 2], [3, 4]
}
```

### Interleaving

```rust
fn main() {
    let a = vec![1, 3, 5];
    let b = vec![2, 4, 6];

    // Interleave manually
    let interleaved: Vec<i32> = a.iter()
        .zip(b.iter())
        .flat_map(|(x, y)| vec![*x, *y])
        .collect();

    println!("{:?}", interleaved);  // [1, 2, 3, 4, 5, 6]
}
```

### Grouping with fold

```rust
use std::collections::HashMap;

fn main() {
    let words = vec!["apple", "apricot", "banana", "blueberry", "cherry"];

    let grouped: HashMap<char, Vec<&str>> = words.iter().fold(
        HashMap::new(),
        |mut acc, word| {
            let first_char = word.chars().next().unwrap();
            acc.entry(first_char).or_insert_with(Vec::new).push(word);
            acc
        },
    );

    println!("{:?}", grouped);
    // {'a': ["apple", "apricot"], 'b': ["banana", "blueberry"], 'c': ["cherry"]}
}
```

### try_fold for Early Exit

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Sum until we exceed a limit
    let result = numbers.iter().try_fold(0, |acc, &x| {
        let sum = acc + x;
        if sum > 6 {
            Err(sum)  // Stop early
        } else {
            Ok(sum)
        }
    });

    println!("{:?}", result);  // Err(10)
}
```

### Creating Infinite Sequences

```rust
fn main() {
    // Natural numbers
    let naturals = 1..;

    // Primes (simple sieve)
    fn is_prime(n: u64) -> bool {
        if n < 2 { return false; }
        (2..=(n as f64).sqrt() as u64).all(|i| n % i != 0)
    }

    let primes: Vec<u64> = (2..).filter(|&n| is_prime(n)).take(10).collect();
    println!("{:?}", primes);  // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
}
```

---

## Quick Reference

### Creating iterators

| Method | Yields | Ownership |
|--------|--------|-----------|
| `.iter()` | `&T` | Borrows |
| `.iter_mut()` | `&mut T` | Borrows mutably |
| `.into_iter()` | `T` | Takes ownership |

### Key adaptors

| Adaptor | Purpose |
|---------|---------|
| `map` | Transform elements |
| `filter` | Keep matching elements |
| `flat_map` | Map and flatten |
| `enumerate` | Add indices |
| `zip` | Combine iterators |
| `take`/`skip` | Limit elements |
| `chain` | Concatenate |
| `peekable` | Look ahead |

### Key consumers

| Consumer | Returns |
|----------|---------|
| `collect` | Collection |
| `fold`/`reduce` | Single value |
| `sum`/`product` | Numeric result |
| `count` | Length |
| `min`/`max` | Extremes |
| `find` | First match |
| `any`/`all` | Boolean |
| `partition` | Two collections |

---

## Common Pitfalls

- `zip` stops at the shorter of the two iterators instead of erroring or padding the
  longer one. See Iterator Adaptors > zip above.
- A custom iterator isn't guaranteed to keep returning `None` once it has returned it
  once; only iterators wrapped in `.fuse()` (or already covered by it) make that
  guarantee. See the fuse section above.
- `nth()` doesn't reset the iterator: calling it again continues from where the previous
  call left off rather than counting from the start. See the nth section above.
- Chaining adaptors like `.map()` then `.filter()` into separate `.collect()` calls
  builds an intermediate collection at each step; chaining them before a single
  `.collect()` avoids that. See Iterator Performance > Avoiding Allocation above.

---

## Summary

The method tables for creating iterators, adaptors, and consumers are collected in the
Quick Reference section above.

### Best Practices

1. Use iterators over manual loops for clarity.
2. Chain adaptors instead of building intermediate collections.
3. Take advantage of laziness with `take()` for early termination.
4. Implement `Iterator` for custom sequences.
5. Use `fold` for aggregations that don't fit `sum`, `product`, or `collect`.
