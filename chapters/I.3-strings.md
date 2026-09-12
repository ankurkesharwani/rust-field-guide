# Strings in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [String vs &str](#string-vs-str)
3. [Creating Strings](#creating-strings)
4. [String Operations](#string-operations)
5. [Indexing and Slicing](#indexing-and-slicing)
6. [Iterating Over Strings](#iterating-over-strings)
7. [String Methods](#string-methods)
8. [Comparing Strings](#comparing-strings)
9. [Formatting Strings](#formatting-strings)
10. [Parsing and Converting](#parsing-and-converting)
11. [Raw Strings and Escape Sequences](#raw-strings-and-escape-sequences)
12. [String Interning and Performance](#string-interning-and-performance)
13. [OsString and OsStr](#osstring-and-osstr)
14. [Quick Reference](#quick-reference)
15. [Common Patterns](#common-patterns)
16. [Common Pitfalls](#common-pitfalls)
17. [Summary](#summary)

---

## Introduction

Strings in Rust are more complex than in many other languages because Rust exposes the underlying complexity of text encoding. Key concepts:

- **UTF-8 encoding**: All Rust strings are valid UTF-8
- **Ownership**: `String` owns its data, `&str` borrows it
- **No direct indexing**: UTF-8 characters can be multiple bytes

Understanding Rust strings means understanding:
- The difference between `String` and `&str`
- UTF-8 encoding implications
- When to use each string type

---

## String vs &str

### String (Owned)

`String` is a growable, heap-allocated string:

```rust
fn main() {
    // Owned, heap-allocated
    let mut s = String::from("hello");

    // Can be modified
    s.push_str(" world");
    s.push('!');

    println!("{}", s);  // "hello world!"

    // Memory layout:
    // Stack: [ptr | len | capacity]
    // Heap:  [h][e][l][l][o][ ][w][o][r][l][d][!]
}
```

### &str (Borrowed Slice)

`&str` is a reference to a string slice:

```rust
fn main() {
    // String literal: &'static str
    let literal: &str = "hello world";

    // Slice of a String
    let owned = String::from("hello world");
    let slice: &str = &owned[0..5];

    println!("{}", literal);  // "hello world"
    println!("{}", slice);    // "hello"

    // Memory layout of &str:
    // [ptr | len]  (points to string data somewhere)
}
```

### When to Use Which

| Use `String` | Use `&str` |
|--------------|------------|
| Need ownership | Borrowing is sufficient |
| Need to modify | Read-only access |
| Returning from functions (usually) | Function parameters (usually) |
| Storing in structs (usually) | Temporary views |

```rust
// Function parameters: prefer &str
fn print_greeting(name: &str) {
    println!("Hello, {}!", name);
}

// Works with both String and &str
fn main() {
    let owned = String::from("Alice");
    let borrowed = "Bob";

    print_greeting(&owned);   // String -> &str via Deref
    print_greeting(borrowed); // &str directly
}
```

---

## Creating Strings

### From Literals

```rust
fn main() {
    // String::from
    let s1 = String::from("hello");

    // to_string() method
    let s2 = "hello".to_string();

    // to_owned() method
    let s3 = "hello".to_owned();

    // String::new() for empty string
    let s4 = String::new();

    // All are equivalent (except s4)
    assert_eq!(s1, s2);
    assert_eq!(s2, s3);
}
```

### From Characters

```rust
fn main() {
    // From a single char
    let s = String::from('a');

    // From char iterator
    let chars = vec!['h', 'e', 'l', 'l', 'o'];
    let s: String = chars.iter().collect();
    println!("{}", s);  // "hello"

    // repeat
    let s = "ab".repeat(3);
    println!("{}", s);  // "ababab"
}
```

### With Capacity

```rust
fn main() {
    // Pre-allocate space
    let mut s = String::with_capacity(100);

    // Check capacity
    println!("Capacity: {}", s.capacity());  // At least 100

    // Efficient: no reallocations if within capacity
    for i in 0..10 {
        s.push_str("hello ");
    }
}
```

### From Other Types

```rust
fn main() {
    // From numbers
    let s = 42.to_string();
    let s = format!("{}", 3.14);

    // From bytes (must be valid UTF-8)
    let bytes = vec![104, 101, 108, 108, 111];  // "hello"
    let s = String::from_utf8(bytes).unwrap();

    // Lossy from bytes (replaces invalid UTF-8)
    let bytes = vec![104, 101, 255, 108, 111];
    let s = String::from_utf8_lossy(&bytes);
    println!("{}", s);  // "he�lo"
}
```

---

## String Operations

### Appending

```rust
fn main() {
    let mut s = String::from("hello");

    // push_str: append &str
    s.push_str(" world");

    // push: append single char
    s.push('!');

    println!("{}", s);  // "hello world!"
}
```

### Concatenation

```rust
fn main() {
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");

    // Using + operator (takes ownership of s1)
    let s3 = s1 + &s2;
    // s1 is moved here, can't use s1 anymore
    println!("{}", s3);

    // Using format! (doesn't take ownership)
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");
    let s3 = format!("{}{}", s1, s2);
    // s1 and s2 still usable
    println!("{} {} {}", s1, s2, s3);
}
```

### Insertion

```rust
fn main() {
    let mut s = String::from("hello");

    // Insert char at byte position
    s.insert(0, 'H');
    println!("{}", s);  // "Hhello"

    // Insert string at byte position
    s.insert_str(1, "i ");
    println!("{}", s);  // "Hi hello"
}
```

### Removal

```rust
fn main() {
    let mut s = String::from("hello world");

    // Remove last character (returns Option<char>)
    let ch = s.pop();
    println!("{:?}", ch);  // Some('d')

    // Remove by index (panics if not char boundary)
    s.remove(0);
    println!("{}", s);  // "ello worl"

    // Truncate to length
    s.truncate(4);
    println!("{}", s);  // "ello"

    // Clear
    s.clear();
    println!("{}", s);  // ""
}
```

### Replacement

```rust
fn main() {
    let s = String::from("hello world world");

    // Replace all occurrences
    let new_s = s.replace("world", "rust");
    println!("{}", new_s);  // "hello rust rust"

    // Replace first n occurrences
    let new_s = s.replacen("world", "rust", 1);
    println!("{}", new_s);  // "hello rust world"
}
```

---

## Indexing and Slicing

### Why No Direct Indexing?

```rust
fn main() {
    let hello = "Здравствуйте";  // Russian "hello"

    // This won't compile:
    // let h = hello[0];

    // Because:
    // - UTF-8 characters can be 1-4 bytes
    // - Index 0 doesn't mean "first character"
    // - "З" is 2 bytes: [208, 151]

    println!("Bytes: {}", hello.len());      // 24 bytes
    println!("Chars: {}", hello.chars().count());  // 12 characters
}
```

### String Slicing

```rust
fn main() {
    let hello = "Здравствуйте";

    // Slice by byte range (must be valid UTF-8 boundaries!)
    let s = &hello[0..4];  // First two characters (4 bytes)
    println!("{}", s);  // "Зд"

    // This would panic (not a char boundary):
    // let s = &hello[0..1];  // PANIC!

    // Safe slicing with get()
    let maybe_slice = hello.get(0..4);
    println!("{:?}", maybe_slice);  // Some("Зд")

    let maybe_slice = hello.get(0..1);
    println!("{:?}", maybe_slice);  // None
}
```

### Accessing Characters

```rust
fn main() {
    let s = "hello";

    // Get nth character
    let third: Option<char> = s.chars().nth(2);
    println!("{:?}", third);  // Some('l')

    // First and last
    let first = s.chars().next();
    let last = s.chars().last();
    println!("{:?}, {:?}", first, last);  // Some('h'), Some('o')
}
```

---

## Iterating Over Strings

### Iterating by Character

```rust
fn main() {
    let s = "नमस्ते";  // Hindi "namaste"

    // Iterate over Unicode scalar values (chars)
    for c in s.chars() {
        println!("{}", c);
    }
    // न, म, स, ्, त, े

    // With indices
    for (i, c) in s.char_indices() {
        println!("{}: {}", i, c);
    }
}
```

### Iterating by Byte

```rust
fn main() {
    let s = "hello";

    for b in s.bytes() {
        println!("{}", b);
    }
    // 104, 101, 108, 108, 111
}
```

### Grapheme Clusters

For proper text segmentation, use the `unicode-segmentation` crate:

```rust
// Cargo.toml: unicode-segmentation = "1.10"

use unicode_segmentation::UnicodeSegmentation;

fn main() {
    let s = "नमस्ते";

    // Grapheme clusters (what users perceive as characters)
    for g in s.graphemes(true) {
        println!("{}", g);
    }
    // न, म, स्, ते

    // Compare with chars()
    println!("Chars: {}", s.chars().count());      // 6
    println!("Graphemes: {}", s.graphemes(true).count());  // 4
}
```

---

## String Methods

### Checking Content

```rust
fn main() {
    let s = "Hello, World!";

    // Contains
    println!("{}", s.contains("World"));  // true

    // Starts/ends with
    println!("{}", s.starts_with("Hello"));  // true
    println!("{}", s.ends_with("!"));        // true

    // Is empty
    println!("{}", s.is_empty());   // false
    println!("{}", "".is_empty());  // true
}
```

### Finding

```rust
fn main() {
    let s = "hello world world";

    // Find first occurrence (returns byte index)
    let pos = s.find("world");
    println!("{:?}", pos);  // Some(6)

    // Find last occurrence
    let pos = s.rfind("world");
    println!("{:?}", pos);  // Some(12)

    // Find with pattern
    let pos = s.find(char::is_whitespace);
    println!("{:?}", pos);  // Some(5)
}
```

### Splitting

```rust
fn main() {
    let s = "hello world rust";

    // Split by string
    let parts: Vec<&str> = s.split(" ").collect();
    println!("{:?}", parts);  // ["hello", "world", "rust"]

    // Split by char
    let parts: Vec<&str> = s.split(' ').collect();

    // Split by predicate
    let parts: Vec<&str> = s.split(char::is_whitespace).collect();

    // Split once
    let (first, rest) = s.split_once(" ").unwrap();
    println!("{}, {}", first, rest);  // "hello", "world rust"

    // Split at most n times
    let parts: Vec<&str> = s.splitn(2, " ").collect();
    println!("{:?}", parts);  // ["hello", "world rust"]

    // Split and keep delimiter
    let parts: Vec<&str> = s.split_inclusive(" ").collect();
    println!("{:?}", parts);  // ["hello ", "world ", "rust"]
}
```

### Trimming

```rust
fn main() {
    let s = "  hello world  ";

    // Trim whitespace
    println!("'{}'", s.trim());        // 'hello world'
    println!("'{}'", s.trim_start());  // 'hello world  '
    println!("'{}'", s.trim_end());    // '  hello world'

    // Trim specific chars
    let s = "xxxhelloxxx";
    println!("{}", s.trim_matches('x'));       // "hello"
    println!("{}", s.trim_start_matches('x')); // "helloxxx"
    println!("{}", s.trim_end_matches('x'));   // "xxxhello"

    // Trim with predicate
    let s = "123hello456";
    println!("{}", s.trim_matches(char::is_numeric));  // "hello"
}
```

### Case Conversion

```rust
fn main() {
    let s = "Hello World";

    println!("{}", s.to_lowercase());  // "hello world"
    println!("{}", s.to_uppercase());  // "HELLO WORLD"

    // For ASCII only (faster)
    println!("{}", s.to_ascii_lowercase());
    println!("{}", s.to_ascii_uppercase());

    // Check case
    let s = "hello";
    println!("{}", s.chars().all(char::is_lowercase));  // true
}
```

### Padding and Alignment

```rust
fn main() {
    let s = "hello";

    // Repeat
    println!("{}", s.repeat(3));  // "hellohellohello"

    // Using format! for padding
    println!("{:>10}", s);   // "     hello" (right align)
    println!("{:<10}", s);   // "hello     " (left align)
    println!("{:^10}", s);   // "  hello   " (center)
    println!("{:*>10}", s);  // "*****hello" (fill with *)
}
```

---

## Comparing Strings

### Equality

`==` and `!=` compare strings by **content**, not by memory address. They work on `String` and `&str` and you can mix the two freely — Rust implements `PartialEq` between them.

```rust
fn main() {
    // &str == &str
    assert!("hello" == "hello");
    assert!("hello" != "world");

    // String == String
    let a = String::from("hello");
    let b = String::from("hello");
    assert!(a == b);  // compares content, not pointers

    // String == &str (and vice versa) — no conversion needed
    let s = String::from("hello");
    assert!(s == "hello");
    assert!("hello" == s);
}
```

### Ordering

Strings implement `Ord`, so `<`, `>`, `<=`, `>=`, and `.cmp()` all work. Comparison is **lexicographic** — character by character, based on Unicode scalar values.

```rust
use std::cmp::Ordering;

fn main() {
    println!("{}", "apple" < "banana");  // true
    println!("{}", "b" > "a");           // true
    println!("{}", "abc" < "abd");       // true — differs at third char

    // cmp() returns Ordering
    match "apple".cmp("banana") {
        Ordering::Less    => println!("apple comes first"),
        Ordering::Equal   => println!("equal"),
        Ordering::Greater => println!("banana comes first"),
    }

    // Sorting a vector of strings
    let mut words = vec!["banana", "apple", "cherry"];
    words.sort();
    println!("{:?}", words);  // ["apple", "banana", "cherry"]

    // Note: uppercase letters sort before lowercase in Unicode
    println!("{}", "Z" < "a");  // true — 'Z' is 90, 'a' is 97
}
```

### Case-insensitive comparison

Use `eq_ignore_ascii_case` for ASCII strings — it avoids allocation. For full Unicode, convert both sides to lowercase first.

```rust
fn main() {
    // ASCII case-insensitive — no allocation
    assert!("Hello".eq_ignore_ascii_case("hello"));
    assert!("RUST".eq_ignore_ascii_case("rust"));

    // Unicode case-insensitive — allocates, handles all scripts
    let a = "Ñoño".to_lowercase();
    let b = "ñoño".to_lowercase();
    assert!(a == b);

    // Case-insensitive sort
    let mut words = vec!["Banana", "apple", "Cherry"];
    words.sort_by(|a, b| a.to_lowercase().cmp(&b.to_lowercase()));
    println!("{:?}", words);  // ["apple", "Banana", "Cherry"]
}
```

### Comparing after trimming whitespace

```rust
fn main() {
    let input = "  hello  ";
    let expected = "hello";

    assert!(input.trim() == expected);
}
```

### Checking start, end, and containment

```rust
fn main() {
    let s = "hello world";

    println!("{}", s.starts_with("hello"));   // true
    println!("{}", s.ends_with("world"));     // true
    println!("{}", s.contains("lo wo"));      // true
}
```

---

## Formatting Strings

### format! Macro

```rust
fn main() {
    let name = "Alice";
    let age = 30;

    // Basic formatting
    let s = format!("Name: {}, Age: {}", name, age);
    println!("{}", s);

    // Positional arguments
    let s = format!("{0} is {1} years old. {0} is great!", name, age);

    // Named arguments
    let s = format!("{name} is {age} years old", name = name, age = age);

    // Debug format
    let v = vec![1, 2, 3];
    let s = format!("{:?}", v);
    println!("{}", s);  // [1, 2, 3]

    // Pretty debug
    let s = format!("{:#?}", v);
}
```

### Number Formatting

```rust
fn main() {
    let n = 42;
    let f = 3.14159;

    // Padding and alignment
    println!("{:5}", n);     // "   42"
    println!("{:05}", n);    // "00042"
    println!("{:<5}", n);    // "42   "

    // Decimal places
    println!("{:.2}", f);    // "3.14"
    println!("{:8.2}", f);   // "    3.14"
    println!("{:08.2}", f);  // "00003.14"

    // Different bases
    println!("{:b}", n);     // "101010" (binary)
    println!("{:o}", n);     // "52" (octal)
    println!("{:x}", n);     // "2a" (hex lowercase)
    println!("{:X}", n);     // "2A" (hex uppercase)

    // Scientific notation
    println!("{:e}", 1234.5);  // "1.2345e3"
    println!("{:E}", 1234.5);  // "1.2345E3"
}
```

### Custom Display

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
    println!("{}", p);  // "(1, 2)"

    let s = format!("Point: {}", p);
}
```

---

## Parsing and Converting

### Parsing Strings to Numbers

```rust
fn main() {
    // Parse with turbofish
    let n: i32 = "42".parse().unwrap();
    let n = "42".parse::<i32>().unwrap();

    // Handle errors
    let result: Result<i32, _> = "not a number".parse();
    match result {
        Ok(n) => println!("Got: {}", n),
        Err(e) => println!("Error: {}", e),
    }

    // Parse different types
    let f: f64 = "3.14".parse().unwrap();
    let b: bool = "true".parse().unwrap();
}
```

### Converting Numbers to Strings

```rust
fn main() {
    let n = 42;

    // to_string()
    let s = n.to_string();

    // format!
    let s = format!("{}", n);

    // With formatting
    let s = format!("{:08b}", n);  // "00101010"
}
```

### Converting Between String Types

```rust
fn main() {
    // &str -> String
    let s: String = "hello".to_string();
    let s: String = String::from("hello");
    let s: String = "hello".to_owned();

    // String -> &str
    let owned = String::from("hello");
    let slice: &str = &owned;
    let slice: &str = owned.as_str();

    // &str -> &[u8]
    let bytes: &[u8] = "hello".as_bytes();

    // String -> Vec<u8>
    let bytes: Vec<u8> = String::from("hello").into_bytes();

    // Vec<u8> -> String
    let bytes = vec![104, 101, 108, 108, 111];
    let s = String::from_utf8(bytes).unwrap();
}
```

---

## Raw Strings and Escape Sequences

### Escape Sequences

```rust
fn main() {
    // Common escapes
    let newline = "hello\nworld";
    let tab = "hello\tworld";
    let quote = "say \"hello\"";
    let backslash = "path\\to\\file";

    // Unicode escape
    let heart = "\u{2764}";  // ❤
    println!("{}", heart);

    // Byte escape
    let a = "\x41";  // 'A'
    println!("{}", a);
}
```

### Raw Strings

```rust
fn main() {
    // Raw string: no escape processing
    let raw = r"C:\Users\name\file.txt";
    println!("{}", raw);  // C:\Users\name\file.txt

    // Raw string with quotes
    let raw = r#"She said "hello""#;
    println!("{}", raw);  // She said "hello"

    // Multiple # for strings containing #
    let raw = r##"This has a # and "quotes""##;
    println!("{}", raw);

    // Useful for regex
    let regex = r"\d{3}-\d{4}";
}
```

### Byte Strings

```rust
fn main() {
    // Byte string literal: &[u8; N]
    let bytes: &[u8; 5] = b"hello";
    println!("{:?}", bytes);  // [104, 101, 108, 108, 111]

    // Raw byte string
    let raw_bytes = br"hello\nworld";
    println!("{:?}", raw_bytes);  // includes literal \, n

    // Byte string with escapes
    let bytes = b"hello\nworld";  // includes actual newline byte
}
```

---

## String Interning and Performance

### String Interning with Rc/Arc

Interning means storing one copy of each distinct string and handing out cheap
references to it instead of allocating a fresh `String` every time the same text shows
up. `Rc<str>` is the reference-counted pointer for this: cloning it bumps a reference
count instead of copying the bytes, so two interned copies of `"hello"` end up pointing
at the same heap allocation. The `HashMap` below is only there to look up whether a
string has already been interned; the sharing itself comes from `Rc::clone`.

```rust
use std::rc::Rc;
use std::collections::HashMap;

struct StringInterner {
    map: HashMap<String, Rc<str>>,
}

impl StringInterner {
    fn new() -> Self {
        StringInterner { map: HashMap::new() }
    }

    fn intern(&mut self, s: &str) -> Rc<str> {
        if let Some(rc) = self.map.get(s) {
            Rc::clone(rc)
        } else {
            let rc: Rc<str> = s.into();
            self.map.insert(s.to_string(), Rc::clone(&rc));
            rc
        }
    }
}

fn main() {
    let mut interner = StringInterner::new();

    let s1 = interner.intern("hello");
    let s2 = interner.intern("hello");

    // s1 and s2 point to same memory
    assert!(Rc::ptr_eq(&s1, &s2));
}
```

### Avoiding Allocations

`Cow<str>` (clone-on-write) lets a function return either a borrowed `&str` or an owned
`String` from the same return type, deciding at runtime based on whether it actually
needs to modify the input. Callers that only read the result don't care which variant
came back; a function that always allocated a `String`, by contrast, would pay for a
copy even on the path that changed nothing.

```rust
use std::borrow::Cow;

fn process(input: &str) -> Cow<str> {
    if input.contains("bad") {
        // Need to modify: allocate
        Cow::Owned(input.replace("bad", "good"))
    } else {
        // No modification: just borrow
        Cow::Borrowed(input)
    }
}

fn main() {
    let s1 = process("hello");        // Borrowed, no allocation
    let s2 = process("hello bad");    // Owned, allocated

    println!("{}, {}", s1, s2);
}
```

### Pre-allocation

```rust
fn main() {
    // Bad: multiple reallocations
    let mut s = String::new();
    for i in 0..1000 {
        s.push_str(&i.to_string());
    }

    // Good: pre-allocate
    let mut s = String::with_capacity(4000);
    for i in 0..1000 {
        s.push_str(&i.to_string());
    }

    // Even better: use collect
    let s: String = (0..1000).map(|i| i.to_string()).collect();
}
```

The first loop starts with no heap allocation and grows `s`'s buffer by reallocating and
copying every time `push_str` would exceed the current capacity. `String::with_capacity`
allocates once up front, so none of the pushes in the second loop need to reallocate at
all as long as the final length stays under 4000 bytes.

---

## OsString and OsStr

Operating systems don't always use UTF-8 for strings. `OsString` and `OsStr` handle platform-native strings safely.

### Why OsString?

```rust
use std::ffi::{OsString, OsStr};
use std::path::Path;

fn main() {
    // File paths and environment variables may contain
    // invalid UTF-8 on some platforms:
    // - Windows: Uses UTF-16 (WTF-8 internally in Rust)
    // - Unix: Uses arbitrary bytes (not necessarily UTF-8)

    // OsString handles this safely
    let path = Path::new("/some/path");
    let os_str: &OsStr = path.as_os_str();

    println!("{:?}", os_str);
}
```

### OsString vs OsStr

| Type | Ownership | Analogous To |
|------|-----------|--------------|
| `OsString` | Owned | `String` |
| `&OsStr` | Borrowed | `&str` |

```rust
use std::ffi::{OsString, OsStr};

fn main() {
    // Creating OsString
    let owned: OsString = OsString::from("hello");

    // Borrowing as &OsStr
    let borrowed: &OsStr = owned.as_os_str();

    // From &str
    let os_str: &OsStr = OsStr::new("hello");

    // OsString is growable like String
    let mut s = OsString::new();
    s.push("hello");
    s.push(" world");
}
```

### Converting Between String Types

```rust
use std::ffi::{OsString, OsStr};

fn main() {
    // &str -> &OsStr (always works)
    let s: &str = "hello";
    let os_str: &OsStr = OsStr::new(s);

    // String -> OsString (always works)
    let string = String::from("hello");
    let os_string: OsString = OsString::from(string);

    // &OsStr -> &str (may fail - returns Option)
    let os_str = OsStr::new("hello");
    let maybe_str: Option<&str> = os_str.to_str();
    println!("{:?}", maybe_str);  // Some("hello")

    // OsString -> String (may fail - returns Result)
    let os_string = OsString::from("hello");
    let result: Result<String, OsString> = os_string.into_string();
    println!("{:?}", result);  // Ok("hello")
}
```

### Lossy Conversion

```rust
use std::ffi::OsStr;

fn main() {
    let os_str = OsStr::new("hello");

    // Always succeeds, replaces invalid UTF-8 with �
    let cow = os_str.to_string_lossy();
    println!("{}", cow);  // "hello"

    // Returns Cow<str>: borrowed if valid UTF-8, owned if conversion needed
    use std::borrow::Cow;
    match os_str.to_string_lossy() {
        Cow::Borrowed(s) => println!("Valid UTF-8: {}", s),
        Cow::Owned(s) => println!("Converted: {}", s),
    }
}
```

### Working with Paths

```rust
use std::path::{Path, PathBuf};
use std::ffi::OsStr;

fn main() {
    // Path wraps OsStr
    let path = Path::new("/home/user/file.txt");

    // Get the OsStr
    let os_str: &OsStr = path.as_os_str();

    // PathBuf wraps OsString
    let path_buf = PathBuf::from("/home/user");
    let os_string = path_buf.into_os_string();

    // File name and extension are OsStr
    let path = Path::new("/home/user/file.txt");
    let name: Option<&OsStr> = path.file_name();
    let ext: Option<&OsStr> = path.extension();

    println!("{:?}, {:?}", name, ext);  // Some("file.txt"), Some("txt")
}
```

### Environment Variables

```rust
use std::env;
use std::ffi::{OsString, OsStr};

fn main() {
    // env::var returns Result<String, VarError>
    // Fails if value is not valid UTF-8
    match env::var("HOME") {
        Ok(val) => println!("HOME: {}", val),
        Err(e) => println!("Error: {}", e),
    }

    // env::var_os returns Option<OsString>
    // Works even with non-UTF-8 values
    match env::var_os("HOME") {
        Some(val) => println!("HOME: {:?}", val),
        None => println!("HOME not set"),
    }

    // Setting environment variables
    env::set_var("MY_VAR", "value");

    // Iterating over all environment variables
    for (key, value) in env::vars_os() {
        // key and value are OsString
        println!("{:?}: {:?}", key, value);
    }
}
```

### Command Arguments

```rust
use std::env;
use std::ffi::OsString;

fn main() {
    // args() returns Strings (panics on invalid UTF-8)
    for arg in env::args() {
        println!("{}", arg);
    }

    // args_os() returns OsStrings (handles any input)
    for arg in env::args_os() {
        println!("{:?}", arg);

        // Try to convert to &str
        if let Some(s) = arg.to_str() {
            println!("  as str: {}", s);
        }
    }
}
```

### Platform-Specific Extensions

```rust
// Unix-specific: OsStr is just bytes
#[cfg(unix)]
fn unix_example() {
    use std::os::unix::ffi::{OsStrExt, OsStringExt};
    use std::ffi::{OsStr, OsString};

    // Create from raw bytes
    let bytes: &[u8] = b"hello";
    let os_str: &OsStr = OsStrExt::from_bytes(bytes);

    // Get raw bytes
    let bytes: &[u8] = os_str.as_bytes();

    // OsString from Vec<u8>
    let bytes = vec![104, 101, 108, 108, 111];
    let os_string = OsString::from_vec(bytes);

    // OsString to Vec<u8>
    let bytes: Vec<u8> = os_string.into_vec();
}

// Windows-specific: OsStr uses WTF-8 (superset of UTF-8)
#[cfg(windows)]
fn windows_example() {
    use std::os::windows::ffi::{OsStrExt, OsStringExt};
    use std::ffi::{OsStr, OsString};

    // Encode as wide characters (UTF-16)
    let os_str = OsStr::new("hello");
    let wide: Vec<u16> = os_str.encode_wide().collect();

    // Create from wide characters
    let os_string = OsString::from_wide(&wide);
}
```

### Common Patterns

```rust
use std::ffi::{OsStr, OsString};
use std::path::Path;

// Accept both &str and &OsStr with AsRef
fn process_path<P: AsRef<Path>>(path: P) {
    let path = path.as_ref();
    println!("Processing: {:?}", path);
}

// Handle potential non-UTF-8 gracefully
fn safe_filename(path: &Path) -> String {
    path.file_name()
        .unwrap_or(OsStr::new("unknown"))
        .to_string_lossy()
        .into_owned()
}

fn main() {
    // Works with &str
    process_path("/home/user");

    // Works with String
    process_path(String::from("/home/user"));

    // Works with &Path
    process_path(Path::new("/home/user"));

    // Works with PathBuf
    process_path(std::path::PathBuf::from("/home/user"));

    let name = safe_filename(Path::new("/path/to/file.txt"));
    println!("Filename: {}", name);
}
```

---

## Quick Reference

### String types

| Type | Ownership | Mutability | Memory |
|------|-----------|------------|--------|
| `String` | Owned | Mutable | Heap |
| `&str` | Borrowed | Immutable | Anywhere |
| `&mut str` | Borrowed | Mutable | Anywhere |
| `OsString` | Owned | Mutable | Analogous to `String`, platform-native |
| `&OsStr` | Borrowed | Immutable | Analogous to `&str`, platform-native |

### When to use which

| Use `String` | Use `&str` |
|--------------|------------|
| Need ownership | Borrowing is sufficient |
| Need to modify | Read-only access |
| Returning from functions (usually) | Function parameters (usually) |
| Storing in structs (usually) | Temporary views |

### Common operations

| Operation | Method |
|-----------|--------|
| Create | `String::new()`, `String::from()`, `.to_string()` |
| Append | `.push()`, `.push_str()` |
| Concatenate | `+`, `format!()` |
| Compare | `==`, `.cmp()`, `.eq_ignore_ascii_case()` |
| Find | `.find()`, `.contains()` |
| Split | `.split()`, `.lines()` |
| Trim | `.trim()`, `.trim_start()`, `.trim_end()` |
| Case | `.to_lowercase()`, `.to_uppercase()` |
| Replace | `.replace()`, `.replacen()` |

---

## Common Patterns

### Building Strings Efficiently

```rust
fn main() {
    // Using push_str
    let mut result = String::new();
    for word in ["hello", "world", "rust"] {
        if !result.is_empty() {
            result.push(' ');
        }
        result.push_str(word);
    }

    // Using join
    let words = ["hello", "world", "rust"];
    let result = words.join(" ");
    println!("{}", result);  // "hello world rust"

    // Using collect with intersperse (nightly)
    // let result: String = words.iter().intersperse(&" ").collect();
}
```

### Splitting and Rejoining

```rust
fn main() {
    let s = "hello,world,rust";

    // Split, transform, join
    let result: String = s
        .split(',')
        .map(|word| word.to_uppercase())
        .collect::<Vec<_>>()
        .join(", ");

    println!("{}", result);  // "HELLO, WORLD, RUST"
}
```

### Safe Truncation

```rust
fn truncate_to_chars(s: &str, max_chars: usize) -> &str {
    match s.char_indices().nth(max_chars) {
        Some((idx, _)) => &s[..idx],
        None => s,
    }
}

fn main() {
    let s = "Hello, 世界!";

    let truncated = truncate_to_chars(s, 8);
    println!("{}", truncated);  // "Hello, 世"
}
```

### Validating Strings

```rust
fn is_valid_username(s: &str) -> bool {
    !s.is_empty()
        && s.len() <= 20
        && s.chars().all(|c| c.is_alphanumeric() || c == '_')
        && s.chars().next().map_or(false, |c| c.is_alphabetic())
}

fn main() {
    println!("{}", is_valid_username("alice_123"));  // true
    println!("{}", is_valid_username("123alice"));   // false
    println!("{}", is_valid_username(""));           // false
}
```

### Working with Lines

```rust
fn main() {
    let text = "line 1\nline 2\nline 3";

    // Iterate lines
    for line in text.lines() {
        println!("{}", line);
    }

    // Collect lines
    let lines: Vec<&str> = text.lines().collect();

    // Join lines
    let rejoined = lines.join("\n");

    // Number of lines
    let count = text.lines().count();
    println!("Line count: {}", count);  // 3
}
```

---

## Common Pitfalls

- Strings can't be indexed by position (`hello[0]` doesn't compile) because a UTF-8
  character can take 1 to 4 bytes, so a byte index doesn't necessarily mean "the Nth
  character."
- Slicing a string at a byte index that isn't on a character boundary panics at
  runtime, for example `&hello[0..1]` on a string whose first character is 2 bytes.
  `.get(range)` returns `None` instead of panicking when the boundary is invalid.
- `.remove(index)` panics the same way if `index` isn't a char boundary.
- The `+` operator takes ownership of its left-hand `String` (`s1 + &s2` moves `s1`),
  so `s1` can't be used afterward; `format!()` doesn't take ownership of either side.
- `eq_ignore_ascii_case` is cheap because it never allocates, but it only understands
  ASCII case folding; comparing non-ASCII text case-insensitively needs
  `.to_lowercase()` on both sides first, which does allocate.
- Uppercase ASCII letters sort before lowercase ones under `Ord` (`"Z" < "a"` is `true`),
  since ordering is based on Unicode scalar values, not alphabetical position.

---

## Summary

Strings in Rust are UTF-8, so characters can take 1 to 4 bytes and there's no direct
indexing by position; `.chars()` or a byte-boundary-checked slice are the safe ways to
access characters. `String` owns and can grow its data on the heap; `&str` borrows a
view into string data anywhere. Prefer `&str` for function parameters and `String` when
a value needs to be owned or mutated. `with_capacity()` avoids repeated reallocation
when building a string incrementally, and `Cow<str>` avoids allocating at all when a
function might not need to modify its input. `OsString`/`OsStr` are the platform-native
counterparts to `String`/`&str`, needed because paths and environment variables aren't
guaranteed to be valid UTF-8 on every platform.
