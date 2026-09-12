# Higher-Order Methods in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Closures](#closures)
3. [Iterator Methods](#iterator-methods)
4. [Slice and Vec Methods](#slice-and-vec-methods)
5. [String Methods](#string-methods)
6. [Function Pointers vs Closures](#function-pointers-vs-closures)
7. [Quick Reference](#quick-reference)
8. [Common Patterns](#common-patterns)
9. [Common Pitfalls](#common-pitfalls)
10. [Performance Considerations](#performance-considerations)

The higher-order methods defined on `Option` and `Result` specifically now live in
[Chapter 19: Option & Result Types](VII.19-option-and-result-types.md), alongside
everything else about those two types.

---

## Introduction

Higher-order methods are methods that take functions (or closures) as arguments or return functions. They're fundamental to Rust's expressive, functional programming style and are used extensively throughout the standard library.

Benefits of higher-order methods:
- **Expressive**: Describe what you want, not how to do it
- **Composable**: Chain operations together fluently
- **Safe**: Type-checked by the compiler
- **Efficient**: Often optimized to be as fast as hand-written loops

---

## Closures

Closures are anonymous functions that can capture their environment.

### Basic Closure Syntax

```rust
fn main() {
    // Basic closure
    let add_one = |x| x + 1;
    println!("{}", add_one(5)); // 6

    // With type annotations
    let add_one: fn(i32) -> i32 = |x: i32| -> i32 { x + 1 };

    // Multi-line closure
    let complex = |x, y| {
        let sum = x + y;
        let product = x * y;
        sum + product
    };
    println!("{}", complex(2, 3)); // 11

    // No parameters
    let say_hello = || println!("Hello!");
    say_hello();
}
```

### Capturing Environment

```rust
fn main() {
    let x = 10;

    // Borrow immutably
    let add_x = |y| x + y;
    println!("{}", add_x(5)); // 15
    println!("{}", x); // Still accessible

    // Borrow mutably
    let mut counter = 0;
    let mut increment = || {
        counter += 1;
    };
    increment();
    increment();
    println!("{}", counter); // 2

    // Take ownership with move
    let name = String::from("Alice");
    let greet = move || {
        println!("Hello, {}!", name);
    };
    greet();
    // name is no longer accessible here
}
```

By default a closure captures each variable the least restrictively it can get away
with: by shared reference if it only reads the variable, by mutable reference if it
mutates it, and only by value (moving it in) if the body requires ownership, for example
by passing the variable somewhere that consumes it. `move` overrides that inference and
forces every captured variable to be taken by value, even ones that would otherwise be
borrowed. That's what makes `name` inaccessible after `greet` is defined: the closure now
owns it. `move` closures are what you reach for when the closure has to outlive the
current scope, such as one handed off to a new thread.

### Closure Traits

Closures implement one or more of these traits, chosen by what the closure body does
with its captures rather than by any annotation you write. The three traits form a
hierarchy: every `Fn` closure is also `FnMut`, and every `FnMut` closure is also
`FnOnce`, since being callable repeatedly without mutation implies being callable
repeatedly with mutation allowed, which implies being callable at all. A closure that
consumes a captured variable (as `consume` does below, by dropping `s`) can only ever be
`FnOnce`, because calling it a second time would try to consume something already gone.

```rust
// FnOnce: can be called once, may consume captured variables
// FnMut: can be called multiple times, may mutate captured variables
// Fn: can be called multiple times, only borrows captured variables

fn call_once<F>(f: F)
where
    F: FnOnce(),
{
    f();
}

fn call_mut<F>(mut f: F)
where
    F: FnMut(),
{
    f();
    f();
}

fn call_many<F>(f: F)
where
    F: Fn(),
{
    f();
    f();
    f();
}

fn main() {
    let s = String::from("hello");

    // FnOnce: consumes s
    let consume = || {
        drop(s);
    };
    call_once(consume);

    // FnMut: mutates counter
    let mut counter = 0;
    let mut mutate = || {
        counter += 1;
    };
    call_mut(&mut mutate);

    // Fn: only reads
    let x = 10;
    let read = || println!("{}", x);
    call_many(read);
}
```

---

## Iterator Methods

Iterator methods are the most common higher-order methods in Rust.

### Transforming: map

Transform each element:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Square each number
    let squared: Vec<i32> = numbers.iter().map(|x| x * x).collect();
    println!("{:?}", squared); // [1, 4, 9, 16, 25]

    // Convert types
    let strings: Vec<String> = numbers.iter().map(|x| x.to_string()).collect();
    println!("{:?}", strings); // ["1", "2", "3", "4", "5"]

    // Chain transformations
    let result: Vec<i32> = numbers
        .iter()
        .map(|x| x * 2)
        .map(|x| x + 1)
        .collect();
    println!("{:?}", result); // [3, 5, 7, 9, 11]
}
```

### Filtering: filter

Keep elements matching a predicate:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // Keep even numbers
    let evens: Vec<&i32> = numbers.iter().filter(|x| *x % 2 == 0).collect();
    println!("{:?}", evens); // [2, 4, 6, 8, 10]

    // Keep numbers greater than 5
    let big: Vec<&i32> = numbers.iter().filter(|&x| *x > 5).collect();
    println!("{:?}", big); // [6, 7, 8, 9, 10]

    // Combine filter and map
    let result: Vec<i32> = numbers
        .iter()
        .filter(|x| *x % 2 == 0)
        .map(|x| x * x)
        .collect();
    println!("{:?}", result); // [4, 16, 36, 64, 100]
}
```

### filter_map

Filter and map in one step:

```rust
fn main() {
    let strings = vec!["1", "two", "3", "four", "5"];

    // Parse only valid numbers
    let numbers: Vec<i32> = strings
        .iter()
        .filter_map(|s| s.parse::<i32>().ok())
        .collect();
    println!("{:?}", numbers); // [1, 3, 5]

    // Equivalent to
    let numbers: Vec<i32> = strings
        .iter()
        .map(|s| s.parse::<i32>().ok())
        .filter(|opt| opt.is_some())
        .map(|opt| opt.unwrap())
        .collect();
}
```

### flat_map

Map and flatten in one step:

```rust
fn main() {
    let words = vec!["hello", "world"];

    // Get all characters
    let chars: Vec<char> = words.iter().flat_map(|s| s.chars()).collect();
    println!("{:?}", chars); // ['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd']

    // Flatten nested vectors
    let nested = vec![vec![1, 2], vec![3, 4], vec![5]];
    let flat: Vec<&i32> = nested.iter().flat_map(|v| v.iter()).collect();
    println!("{:?}", flat); // [1, 2, 3, 4, 5]

    // Generate multiple outputs per input
    let numbers = vec![1, 2, 3];
    let repeated: Vec<i32> = numbers
        .iter()
        .flat_map(|&x| vec![x, x * 10])
        .collect();
    println!("{:?}", repeated); // [1, 10, 2, 20, 3, 30]
}
```

### Reducing: fold and reduce

Combine all elements into one:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Sum with fold
    let sum = numbers.iter().fold(0, |acc, x| acc + x);
    println!("Sum: {}", sum); // 15

    // Product with fold
    let product = numbers.iter().fold(1, |acc, x| acc * x);
    println!("Product: {}", product); // 120

    // Build a string
    let concat = numbers.iter().fold(String::new(), |mut acc, x| {
        acc.push_str(&x.to_string());
        acc
    });
    println!("Concat: {}", concat); // "12345"

    // reduce doesn't need initial value
    let max = numbers.iter().reduce(|a, b| if a > b { a } else { b });
    println!("Max: {:?}", max); // Some(5)
}
```

### Finding: find and position

Locate elements:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Find first even
    let first_even = numbers.iter().find(|x| *x % 2 == 0);
    println!("{:?}", first_even); // Some(2)

    // Find position
    let pos = numbers.iter().position(|x| *x == 3);
    println!("{:?}", pos); // Some(2)

    // Find last
    let last_even = numbers.iter().rfind(|x| *x % 2 == 0);
    println!("{:?}", last_even); // Some(4)

    // Find with transformation
    let first_big_square = numbers
        .iter()
        .find_map(|x| {
            let sq = x * x;
            if sq > 10 { Some(sq) } else { None }
        });
    println!("{:?}", first_big_square); // Some(16)
}
```

### Testing: any and all

Check conditions across elements:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Check if any element matches
    let has_even = numbers.iter().any(|x| x % 2 == 0);
    println!("Has even: {}", has_even); // true

    // Check if all elements match
    let all_positive = numbers.iter().all(|x| *x > 0);
    println!("All positive: {}", all_positive); // true

    let all_even = numbers.iter().all(|x| x % 2 == 0);
    println!("All even: {}", all_even); // false
}
```

### Counting and Aggregating

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Count
    let count = numbers.iter().count();
    println!("Count: {}", count); // 5

    // Count matching
    let even_count = numbers.iter().filter(|x| *x % 2 == 0).count();
    println!("Even count: {}", even_count); // 2

    // Sum and product
    let sum: i32 = numbers.iter().sum();
    let product: i32 = numbers.iter().product();
    println!("Sum: {}, Product: {}", sum, product); // 15, 120

    // Min and max
    let min = numbers.iter().min();
    let max = numbers.iter().max();
    println!("Min: {:?}, Max: {:?}", min, max); // Some(1), Some(5)

    // With custom comparison
    let words = vec!["apple", "pie", "strawberry"];
    let longest = words.iter().max_by_key(|s| s.len());
    println!("Longest: {:?}", longest); // Some("strawberry")
}
```

### for_each

Execute side effects:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Print each (more functional style than for loop)
    numbers.iter().for_each(|x| println!("{}", x));

    // With enumeration
    numbers.iter().enumerate().for_each(|(i, x)| {
        println!("Index {}: {}", i, x);
    });

    // Side effects with mutable state
    let mut sum = 0;
    numbers.iter().for_each(|x| sum += x);
    println!("Sum: {}", sum); // 15
}
```

### take and skip

Limit iteration:

```rust
fn main() {
    let numbers: Vec<i32> = (1..=100).collect();

    // Take first n
    let first_five: Vec<&i32> = numbers.iter().take(5).collect();
    println!("{:?}", first_five); // [1, 2, 3, 4, 5]

    // Skip first n
    let after_five: Vec<&i32> = numbers.iter().skip(95).collect();
    println!("{:?}", after_five); // [96, 97, 98, 99, 100]

    // Take while condition holds
    let small: Vec<&i32> = numbers.iter().take_while(|x| **x < 5).collect();
    println!("{:?}", small); // [1, 2, 3, 4]

    // Skip while condition holds
    let big: Vec<&i32> = numbers.iter().skip_while(|x| **x < 95).collect();
    println!("{:?}", big); // [95, 96, 97, 98, 99, 100]
}
```

### zip

Combine iterators:

```rust
fn main() {
    let names = vec!["Alice", "Bob", "Charlie"];
    let ages = vec![30, 25, 35];

    // Zip together
    let people: Vec<_> = names.iter().zip(ages.iter()).collect();
    println!("{:?}", people); // [("Alice", 30), ("Bob", 25), ("Charlie", 35)]

    // Use zipped values
    for (name, age) in names.iter().zip(ages.iter()) {
        println!("{} is {} years old", name, age);
    }

    // Compute dot product
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    let dot: i32 = a.iter().zip(b.iter()).map(|(x, y)| x * y).sum();
    println!("Dot product: {}", dot); // 32
}
```

### enumerate

Add indices:

```rust
fn main() {
    let chars = vec!['a', 'b', 'c'];

    // Get index with each element
    for (index, char) in chars.iter().enumerate() {
        println!("Index {}: {}", index, char);
    }

    // Collect into indexed pairs
    let indexed: Vec<(usize, &char)> = chars.iter().enumerate().collect();
    println!("{:?}", indexed); // [(0, 'a'), (1, 'b'), (2, 'c')]

    // Find index of element
    let pos = chars.iter().enumerate()
        .find(|(_, c)| **c == 'b')
        .map(|(i, _)| i);
    println!("{:?}", pos); // Some(1)
}
```

### chain and cycle

Extend iterators:

```rust
fn main() {
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];

    // Chain iterators
    let combined: Vec<&i32> = a.iter().chain(b.iter()).collect();
    println!("{:?}", combined); // [1, 2, 3, 4, 5, 6]

    // Cycle infinitely
    let repeated: Vec<&i32> = a.iter().cycle().take(10).collect();
    println!("{:?}", repeated); // [1, 2, 3, 1, 2, 3, 1, 2, 3, 1]
}
```

### partition

Split by predicate:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // Split into two collections
    let (evens, odds): (Vec<i32>, Vec<i32>) = numbers
        .into_iter()
        .partition(|x| x % 2 == 0);

    println!("Evens: {:?}", evens); // [2, 4, 6, 8, 10]
    println!("Odds: {:?}", odds);   // [1, 3, 5, 7, 9]
}
```

### inspect

Debug without consuming:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let result: Vec<i32> = numbers
        .iter()
        .inspect(|x| println!("Before filter: {}", x))
        .filter(|x| *x % 2 == 0)
        .inspect(|x| println!("After filter: {}", x))
        .map(|x| x * 2)
        .inspect(|x| println!("After map: {}", x))
        .collect();

    println!("Result: {:?}", result);
}
```

---

## Slice and Vec Methods

Higher-order methods on slices and vectors:

### sort_by and sort_by_key

```rust
fn main() {
    let mut numbers = vec![3, 1, 4, 1, 5, 9, 2, 6];

    // Sort with custom comparator
    numbers.sort_by(|a, b| b.cmp(a)); // Descending
    println!("{:?}", numbers); // [9, 6, 5, 4, 3, 2, 1, 1]

    // Sort by key
    let mut words = vec!["apple", "pie", "strawberry", "a"];
    words.sort_by_key(|s| s.len());
    println!("{:?}", words); // ["a", "pie", "apple", "strawberry"]

    // Stable sort preserves order of equal elements
    numbers.sort_by(|a, b| a.cmp(b));
}
```

### retain

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // Keep only even numbers (modifies in place)
    numbers.retain(|x| x % 2 == 0);
    println!("{:?}", numbers); // [2, 4, 6, 8, 10]

    // With mutable access to elements
    let mut pairs = vec![(1, "a"), (2, "b"), (3, "c")];
    pairs.retain(|(n, _)| *n > 1);
    println!("{:?}", pairs); // [(2, "b"), (3, "c")]
}
```

### dedup_by

```rust
fn main() {
    let mut data = vec![1, 1, 2, 2, 2, 3, 3];

    // Remove consecutive duplicates
    data.dedup();
    println!("{:?}", data); // [1, 2, 3]

    // With custom equality
    let mut words = vec!["Hello", "HELLO", "world", "WORLD"];
    words.dedup_by(|a, b| a.eq_ignore_ascii_case(b));
    println!("{:?}", words); // ["Hello", "world"]
}
```

### binary_search_by

```rust
fn main() {
    let sorted = vec![1, 2, 4, 5, 6, 8, 9];

    // Custom binary search
    let result = sorted.binary_search_by(|x| x.cmp(&5));
    println!("{:?}", result); // Ok(3)

    // By key
    let pairs = vec![(1, "a"), (3, "c"), (5, "e")];
    let result = pairs.binary_search_by_key(&3, |&(k, _)| k);
    println!("{:?}", result); // Ok(1)
}
```

---

## String Methods

Higher-order methods on strings:

### Splitting and Matching

```rust
fn main() {
    let text = "hello,world,rust";

    // Split with closure
    let parts: Vec<&str> = text.split(|c| c == ',').collect();
    println!("{:?}", parts); // ["hello", "world", "rust"]

    // Split with predicate
    let text = "hello123world456rust";
    let parts: Vec<&str> = text.split(char::is_numeric).collect();
    println!("{:?}", parts); // ["hello", "", "", "world", "", "", "rust"]

    // Trim with predicate
    let trimmed = "###hello###".trim_matches(|c| c == '#');
    println!("{}", trimmed); // "hello"

    // Find with predicate
    let pos = "hello".find(|c: char| c == 'l');
    println!("{:?}", pos); // Some(2)
}
```

### Mapping Characters

```rust
fn main() {
    let text = "Hello World";

    // Map characters
    let upper: String = text.chars().map(|c| c.to_uppercase().next().unwrap()).collect();
    println!("{}", upper); // "HELLO WORLD"

    // Filter characters
    let letters: String = text.chars().filter(|c| c.is_alphabetic()).collect();
    println!("{}", letters); // "HelloWorld"

    // Replace with closure (using chars)
    let replaced: String = text
        .chars()
        .map(|c| if c == ' ' { '_' } else { c })
        .collect();
    println!("{}", replaced); // "Hello_World"
}
```

---

## Function Pointers vs Closures

### Function Pointers

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn apply_twice(f: fn(i32) -> i32, x: i32) -> i32 {
    f(f(x))
}

fn main() {
    // Use function pointer
    let result = apply_twice(add_one, 5);
    println!("{}", result); // 7

    // Function pointers with iterators
    let numbers = vec![1, 2, 3];
    let incremented: Vec<i32> = numbers.iter().map(|&x| add_one(x)).collect();

    // Can also pass function directly (coerced to fn pointer)
    let parsed: Vec<i32> = vec!["1", "2", "3"]
        .iter()
        .map(|s| s.parse().unwrap())
        .collect();
}
```

### Closures vs Function Pointers

```rust
fn apply<F>(f: F, x: i32) -> i32
where
    F: Fn(i32) -> i32,
{
    f(x)
}

fn main() {
    // Closure that captures nothing can be coerced to fn pointer
    let closure: fn(i32) -> i32 = |x| x + 1;

    // Closure that captures cannot be coerced
    let y = 10;
    // let closure: fn(i32) -> i32 = |x| x + y; // ERROR

    // Use trait bound instead
    let result = apply(|x| x + y, 5);
    println!("{}", result); // 15
}
```

---

## Quick Reference

| Category | Common Methods |
|----------|----------------|
| Transform | `map`, `flat_map`, `filter_map` |
| Filter | `filter`, `take`, `skip`, `take_while`, `skip_while` |
| Reduce | `fold`, `reduce`, `sum`, `product` |
| Search | `find`, `position`, `any`, `all` |
| Combine | `zip`, `chain`, `enumerate` |
| Collect | `collect`, `partition` |
| Side Effects | `for_each`, `inspect` |

---

## Common Patterns

### Pipeline Pattern

```rust
fn main() {
    let data = vec!["  hello  ", "WORLD", " rust "];

    let result: Vec<String> = data
        .iter()
        .map(|s| s.trim())           // Remove whitespace
        .map(|s| s.to_lowercase())   // Lowercase
        .filter(|s| s.len() > 4)     // Keep long words
        .collect();

    println!("{:?}", result); // ["hello", "world"]
}
```

### Error Collection

```rust
fn main() {
    let strings = vec!["1", "2", "three", "4"];

    // Collect all errors
    let (successes, failures): (Vec<_>, Vec<_>) = strings
        .iter()
        .map(|s| s.parse::<i32>())
        .partition(Result::is_ok);

    let successes: Vec<i32> = successes.into_iter().map(|r| r.unwrap()).collect();
    let failures: Vec<_> = failures.into_iter().map(|r| r.unwrap_err()).collect();

    println!("Successes: {:?}", successes);
    println!("Failures: {:?}", failures);
}
```

### Building Complex Structures

```rust
use std::collections::HashMap;

fn main() {
    let pairs = vec![("a", 1), ("b", 2), ("c", 3)];

    // Build HashMap with fold
    let map: HashMap<&str, i32> = pairs
        .into_iter()
        .fold(HashMap::new(), |mut acc, (k, v)| {
            acc.insert(k, v);
            acc
        });

    // Or use collect
    let map: HashMap<&str, i32> = vec![("a", 1), ("b", 2)].into_iter().collect();

    println!("{:?}", map);
}
```

### Grouping

```rust
use std::collections::HashMap;

fn main() {
    let words = vec!["apple", "ant", "banana", "bear", "cat"];

    // Group by first letter
    let grouped: HashMap<char, Vec<&str>> = words
        .iter()
        .fold(HashMap::new(), |mut acc, word| {
            let first = word.chars().next().unwrap();
            acc.entry(first).or_insert_with(Vec::new).push(word);
            acc
        });

    println!("{:?}", grouped);
    // {'a': ["apple", "ant"], 'b': ["banana", "bear"], 'c': ["cat"]}
}
```

### Windowing

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Sliding windows
    let windows: Vec<_> = numbers.windows(3).collect();
    println!("{:?}", windows); // [[1, 2, 3], [2, 3, 4], [3, 4, 5]]

    // Moving average
    let averages: Vec<f64> = numbers
        .windows(3)
        .map(|w| w.iter().sum::<i32>() as f64 / w.len() as f64)
        .collect();
    println!("{:?}", averages); // [2.0, 3.0, 4.0]

    // Chunks
    let chunks: Vec<_> = numbers.chunks(2).collect();
    println!("{:?}", chunks); // [[1, 2], [3, 4], [5]]
}
```

---

## Common Pitfalls

- A closure that captures its environment cannot be coerced to a plain `fn` pointer; only
  a non-capturing closure can. See Closures vs Function Pointers above.
- An `FnOnce` closure consumes what it captures, so it can only be called once; passing
  it where an `Fn` or `FnMut` is required won't compile. See Closure Traits above.

---

## Performance Considerations

### Lazy Evaluation

Iterators are lazy - they don't do work until consumed:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // No work done yet
    let iter = numbers
        .iter()
        .map(|x| {
            println!("Mapping {}", x);
            x * 2
        })
        .filter(|x| {
            println!("Filtering {}", x);
            *x > 5
        });

    println!("Created iterator, no work done yet");

    // Work happens here
    let result: Vec<_> = iter.collect();
    println!("{:?}", result);
}
```

### Short-Circuiting

Some methods stop early:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // find stops at first match
    let found = numbers.iter().find(|&&x| {
        println!("Checking {}", x);
        x == 3
    });
    // Only prints "Checking 1", "Checking 2", "Checking 3"

    // any stops at first true
    let has_even = numbers.iter().any(|x| {
        println!("Testing {}", x);
        x % 2 == 0
    });
    // Only prints "Testing 1", "Testing 2"
}
```

### Collect vs For Loop

```rust
fn main() {
    let numbers: Vec<i32> = (1..=1000000).collect();

    // Both are efficient - iterator is zero-cost abstraction
    let sum1: i32 = numbers.iter().sum();

    let mut sum2 = 0;
    for n in &numbers {
        sum2 += n;
    }

    assert_eq!(sum1, sum2);
}
```

---

## Summary

See the Quick Reference section above for the method table by category.

Key principles:
1. Prefer a declarative style: use iterator methods over manual loops.
2. Chain operations to build pipelines of transformations.
3. Remember that iterators are lazy and don't compute until consumed.
4. They compile down to a zero-cost abstraction, so this style has no runtime penalty.
5. The compiler verifies every transformation in the chain for type safety.
