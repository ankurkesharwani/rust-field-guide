# File Handling in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Automatic File Closing (RAII)](#automatic-file-closing-raii)
3. [Opening and Creating Files](#opening-and-creating-files)
4. [Reading Files](#reading-files)
5. [Writing Files](#writing-files)
6. [Appending to Files](#appending-to-files)
7. [Working with Paths](#working-with-paths)
8. [Directory Operations](#directory-operations)
9. [File Metadata](#file-metadata)
10. [Error Handling with Files](#error-handling-with-files)
11. [Quick Reference](#quick-reference)
12. [Common Patterns](#common-patterns)
13. [Common Pitfalls](#common-pitfalls)
14. [Summary](#summary)

---

## Introduction

Rust's file I/O lives primarily in `std::fs` and `std::io`. The API is explicit and error-driven — every operation that can fail returns a `Result`, which forces you to handle errors at the call site rather than discovering them later.

The key types you'll use most:

| Type | Purpose |
|---|---|
| `std::fs::File` | A handle to an open file |
| `std::fs::OpenOptions` | Builder for configuring how a file is opened |
| `std::path::Path` / `PathBuf` | Borrowed / owned file-system paths |
| `std::io::BufReader` | Buffered reading wrapper |
| `std::io::BufWriter` | Buffered writing wrapper |

---

## Automatic File Closing (RAII)

Unlike C, you do **not** need to manually close files in Rust. Files are closed automatically when they go out of scope, via the **RAII** (Resource Acquisition Is Initialization) pattern — the same mechanism that manages memory.

When a `File` value is dropped (goes out of scope), Rust calls its `Drop` implementation, which closes the underlying OS file handle. This is guaranteed whether the function returns normally or with an error.

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_file(path: &str) -> io::Result<String> {
    let mut file = File::open(path)?;   // file opened
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}   // <-- file goes out of scope here, Drop is called, file is closed automatically
```

You can also force an early close by calling `drop(file)` explicitly — useful when you need to release a file lock or close before a rename:

```rust
let file = File::create("output.txt")?;
// ... write ...
drop(file);                         // closed here, not at end of scope
fs::rename("output.txt", "final.txt")?;
```

> **Key point:** Rust makes it *impossible* to forget to close a file. The compiler enforces cleanup through ownership — there is no `fclose()` to call.

---

## Opening and Creating Files

### Open an existing file for reading

```rust
use std::fs::File;

fn main() -> std::io::Result<()> {
    let file = File::open("hello.txt")?;
    // `file` is dropped and closed at end of scope
    Ok(())
}
```

### Create a new file (truncates if it already exists)

```rust
use std::fs::File;

fn main() -> std::io::Result<()> {
    let file = File::create("output.txt")?;
    Ok(())
}
```

### Full control with `OpenOptions`

`OpenOptions` is a builder that lets you mix any combination of read, write, append, create, and truncate:

```rust
use std::fs::OpenOptions;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)       // create if missing
        .truncate(false)    // do NOT wipe existing content
        .open("data.txt")?;

    Ok(())
}
```

Common flag combinations:

| Goal | Flags |
|---|---|
| Read only | `.read(true)` |
| Overwrite / create | `.write(true).create(true).truncate(true)` |
| Append / create | `.append(true).create(true)` |
| Read + write, must exist | `.read(true).write(true)` |
| Read + write, create | `.read(true).write(true).create(true)` |

---

## Reading Files

### Read the entire file into a `String`

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    let contents = fs::read_to_string("hello.txt")?;
    println!("{}", contents);
    Ok(())
}
```

### Read the entire file into raw bytes

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    let bytes: Vec<u8> = fs::read("image.png")?;
    println!("Read {} bytes", bytes.len());
    Ok(())
}
```

### Read line by line with `BufReader`

For large files, reading line by line avoids loading everything into memory at once.
`BufReader` also matters for performance here even on small files: a bare `File` has no
buffer, so reading one line at a time without it would mean one `read` syscall per
line. `BufReader` reads a larger chunk from the OS in one syscall and serves subsequent
`.lines()` calls out of that in-memory buffer, refilling only when it runs out.

```rust
use std::fs::File;
use std::io::{self, BufRead};

fn main() -> io::Result<()> {
    let file = File::open("log.txt")?;
    let reader = io::BufReader::new(file);

    for line in reader.lines() {
        let line = line?;    // each line() call returns Result<String>
        println!("{}", line);
    }
    Ok(())
}
```

### Read into a pre-allocated buffer

Unlike `read_to_string`/`read`, the `Read::read` method doesn't fill the whole buffer or
read the whole file; it returns as soon as at least one byte is available, and the
return value tells you how many bytes actually landed in `buffer`. Reading a large file
this way means calling `read` in a loop until it returns `0` (end of file), checking the
count each time rather than assuming the buffer came back full.

```rust
use std::fs::File;
use std::io::Read;

fn main() -> std::io::Result<()> {
    let mut file = File::open("data.bin")?;
    let mut buffer = [0u8; 1024];

    let bytes_read = file.read(&mut buffer)?;
    println!("Read {} bytes", bytes_read);
    Ok(())
}
```

---

## Writing Files

### Write a string in one call

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    fs::write("greeting.txt", "Hello, Rust!\n")?;
    Ok(())
}
```

### Write bytes in one call

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    let data: &[u8] = &[0x48, 0x65, 0x6c, 0x6c, 0x6f];
    fs::write("bytes.bin", data)?;
    Ok(())
}
```

### Incremental writes with `BufWriter`

`BufWriter` batches small writes into larger system calls, which is much faster when writing many small pieces:

```rust
use std::fs::File;
use std::io::{self, BufWriter, Write};

fn main() -> io::Result<()> {
    let file = File::create("output.txt")?;
    let mut writer = BufWriter::new(file);

    for i in 0..1000 {
        writeln!(writer, "Line {}", i)?;
    }
    // BufWriter flushes automatically on drop, but explicit flush
    // lets you catch errors:
    writer.flush()?;
    Ok(())
}
```

> **Note:** Always call `.flush()` explicitly when you need to guarantee data is written to disk before the writer goes out of scope, because a silent drop discards buffered data on error.

---

## Appending to Files

```rust
use std::fs::OpenOptions;
use std::io::Write;

fn main() -> std::io::Result<()> {
    let mut file = OpenOptions::new()
        .append(true)
        .create(true)
        .open("log.txt")?;

    writeln!(file, "New log entry")?;
    Ok(())
}
```

---

## Working with Paths

`Path` is a borrowed path slice (like `str`), and `PathBuf` is the owned, heap-allocated version (like `String`).

```rust
use std::path::{Path, PathBuf};

fn main() {
    // Build a path from components — handles OS separators automatically
    let mut path = PathBuf::from("/home/user");
    path.push("projects");
    path.push("main.rs");

    println!("{}", path.display()); // /home/user/projects/main.rs

    // Inspect parts
    let p = Path::new("/home/user/projects/main.rs");
    println!("file name : {:?}", p.file_name());   // Some("main.rs")
    println!("stem      : {:?}", p.file_stem());   // Some("main")
    println!("extension : {:?}", p.extension());   // Some("rs")
    println!("parent    : {:?}", p.parent());      // Some("/home/user/projects")
}
```

### Checking existence

```rust
use std::path::Path;

fn main() {
    let p = Path::new("config.toml");

    if p.exists() {
        if p.is_file() {
            println!("It's a file");
        } else if p.is_dir() {
            println!("It's a directory");
        }
    } else {
        println!("Path does not exist");
    }
}
```

---

## Directory Operations

### Create a directory

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    fs::create_dir("new_dir")?;                 // fails if parent missing
    fs::create_dir_all("a/b/c/deep_dir")?;     // creates all missing parents
    Ok(())
}
```

### List directory contents

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    for entry in fs::read_dir(".")? {
        let entry = entry?;
        let path = entry.path();
        let file_type = entry.file_type()?;
        println!("{:?}  (dir: {})", path, file_type.is_dir());
    }
    Ok(())
}
```

### Remove files and directories

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    fs::remove_file("old.txt")?;           // single file
    fs::remove_dir("empty_dir")?;          // directory must be empty
    fs::remove_dir_all("tree")?;           // recursive delete
    Ok(())
}
```

### Copy and rename

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    fs::copy("source.txt", "dest.txt")?;       // copies content + permissions
    fs::rename("old_name.txt", "new_name.txt")?; // atomic on the same filesystem
    Ok(())
}
```

---

## File Metadata

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    let meta = fs::metadata("hello.txt")?;

    println!("size       : {} bytes", meta.len());
    println!("is file    : {}", meta.is_file());
    println!("is dir     : {}", meta.is_dir());
    println!("read-only  : {}", meta.permissions().readonly());

    if let Ok(modified) = meta.modified() {
        println!("modified   : {:?}", modified);
    }
    Ok(())
}
```

---

## Error Handling with Files

File errors are `std::io::Error`. Inspect the `ErrorKind` to act on specific cases:

```rust
use std::fs::File;
use std::io::{self, ErrorKind};

fn open_or_create(path: &str) -> io::Result<File> {
    match File::open(path) {
        Ok(file) => Ok(file),
        Err(e) if e.kind() == ErrorKind::NotFound => {
            println!("'{}' not found, creating it.", path);
            File::create(path)
        }
        Err(e) => Err(e), // permission denied, etc. — propagate
    }
}

fn main() -> io::Result<()> {
    let _file = open_or_create("config.txt")?;
    Ok(())
}
```

Common `ErrorKind` variants:

| Variant | Meaning |
|---|---|
| `NotFound` | Path does not exist |
| `PermissionDenied` | Insufficient permissions |
| `AlreadyExists` | File already exists (e.g. exclusive create) |
| `WouldBlock` | Non-blocking I/O would have blocked |
| `UnexpectedEof` | Read ended before buffer was filled |

---

## Quick Reference

### Key types

| Type | Purpose |
|---|---|
| `std::fs::File` | A handle to an open file |
| `std::fs::OpenOptions` | Builder for configuring how a file is opened |
| `std::path::Path` / `PathBuf` | Borrowed / owned file-system paths |
| `std::io::BufReader` | Buffered reading wrapper |
| `std::io::BufWriter` | Buffered writing wrapper |

### OpenOptions flag combinations

| Goal | Flags |
|---|---|
| Read only | `.read(true)` |
| Overwrite / create | `.write(true).create(true).truncate(true)` |
| Append / create | `.append(true).create(true)` |
| Read + write, must exist | `.read(true).write(true)` |
| Read + write, create | `.read(true).write(true).create(true)` |

### Common ErrorKind variants

| Variant | Meaning |
|---|---|
| `NotFound` | Path does not exist |
| `PermissionDenied` | Insufficient permissions |
| `AlreadyExists` | File already exists (e.g. exclusive create) |
| `WouldBlock` | Non-blocking I/O would have blocked |
| `UnexpectedEof` | Read ended before buffer was filled |

---

## Common Patterns

### Pattern: process a file line by line, skip blank lines

```rust
use std::fs::File;
use std::io::{self, BufRead};

fn main() -> io::Result<()> {
    let file = File::open("data.txt")?;
    let reader = io::BufReader::new(file);

    for line in reader.lines() {
        let line = line?;
        let trimmed = line.trim();
        if trimmed.is_empty() { continue; }
        // process trimmed
        println!("{}", trimmed);
    }
    Ok(())
}
```

### Pattern: read a structured CSV-like file

```rust
use std::fs::File;
use std::io::{self, BufRead};

fn main() -> io::Result<()> {
    let file = File::open("scores.csv")?;
    let reader = io::BufReader::new(file);

    for (i, line) in reader.lines().enumerate() {
        let line = line?;
        if i == 0 { continue; }  // skip header

        let fields: Vec<&str> = line.splitn(3, ',').collect();
        if let [name, score, grade] = fields.as_slice() {
            println!("name={} score={} grade={}", name, score, grade);
        }
    }
    Ok(())
}
```

### Pattern: write a temp file then atomically replace the target

Writing to a temporary file and then renaming is the safest way to update a file — readers never see a half-written state:

```rust
use std::fs;
use std::io::Write;

fn atomic_write(path: &str, content: &str) -> std::io::Result<()> {
    let tmp = format!("{}.tmp", path);
    let mut file = fs::File::create(&tmp)?;
    file.write_all(content.as_bytes())?;
    file.flush()?;
    drop(file);                         // close before rename
    fs::rename(&tmp, path)?;            // atomic on same filesystem
    Ok(())
}

fn main() -> std::io::Result<()> {
    atomic_write("config.toml", "[settings]\nkey = \"value\"\n")
}
```

### Pattern: recursively walk a directory tree

The standard library only does one level at a time. For deep traversal, implement it yourself or use the `walkdir` crate:

```rust
// With std only (manual recursion)
use std::fs;
use std::path::Path;
use std::io;

fn walk(dir: &Path) -> io::Result<()> {
    for entry in fs::read_dir(dir)? {
        let entry = entry?;
        let path = entry.path();
        if path.is_dir() {
            walk(&path)?;
        } else {
            println!("{}", path.display());
        }
    }
    Ok(())
}

fn main() -> io::Result<()> {
    walk(Path::new("."))
}
```

```toml
# With walkdir crate (Cargo.toml)
[dependencies]
walkdir = "2"
```

```rust
// With walkdir crate
use walkdir::WalkDir;

fn main() {
    for entry in WalkDir::new(".").into_iter().filter_map(|e| e.ok()) {
        println!("{}", entry.path().display());
    }
}
```

### Pattern: read a config file, fall back to defaults

```rust
use std::fs;
use std::io::ErrorKind;

#[derive(Debug)]
struct Config {
    host: String,
    port: u16,
}

impl Default for Config {
    fn default() -> Self {
        Config { host: "localhost".into(), port: 8080 }
    }
}

fn load_config(path: &str) -> Config {
    match fs::read_to_string(path) {
        Ok(contents) => parse_config(&contents).unwrap_or_default(),
        Err(e) if e.kind() == ErrorKind::NotFound => {
            eprintln!("Config not found, using defaults");
            Config::default()
        }
        Err(e) => {
            eprintln!("Could not read config: {e}");
            Config::default()
        }
    }
}

fn parse_config(_contents: &str) -> Option<Config> {
    // parse logic here
    None
}

fn main() {
    let config = load_config("app.toml");
    println!("{:?}", config);
}
```

### Pattern: collect all files with a given extension

```rust
use std::fs;
use std::path::PathBuf;

fn files_with_ext(dir: &str, ext: &str) -> std::io::Result<Vec<PathBuf>> {
    let mut result = Vec::new();
    for entry in fs::read_dir(dir)? {
        let path = entry?.path();
        if path.is_file() && path.extension().map_or(false, |e| e == ext) {
            result.push(path);
        }
    }
    Ok(result)
}

fn main() -> std::io::Result<()> {
    let rs_files = files_with_ext("src", "rs")?;
    for f in rs_files {
        println!("{}", f.display());
    }
    Ok(())
}
```

---

## Common Pitfalls

### BufWriter can silently drop buffered data

`BufWriter` flushes on drop, but a drop that happens during unwinding, or one where the
flush itself fails, discards the buffered data without telling you. Call `.flush()`
explicitly when you need to be sure the data reached disk.

### fs::rename is only atomic on the same filesystem

`fs::rename` is atomic when the source and destination are on the same filesystem, which
is what makes the write-to-temp-then-rename pattern above safe. Renaming across
filesystems (or across mount points) falls back to a copy-then-delete, which loses that
atomicity guarantee.

### read_dir only lists one directory level

`fs::read_dir` reads a single directory's entries; it does not descend into
subdirectories. Recursing into a full directory tree means writing that recursion
yourself, as shown above, or reaching for a crate like `walkdir`.

---

## Summary

Rust's file I/O lives in `std::fs` and `std::io`, with `File` handles closed
automatically through RAII rather than a manual close call. `OpenOptions` covers any
combination of read, write, append, create, and truncate; `BufReader`/`BufWriter` batch
reads and writes for efficiency; `Path`/`PathBuf` handle file-system paths; and file
errors surface as `std::io::Error`, inspectable through `ErrorKind`. The Common Patterns
above cover the recurring shapes: line-by-line processing, atomic writes via a temp file
and rename, recursive directory walks, and config loading with a fallback default.
