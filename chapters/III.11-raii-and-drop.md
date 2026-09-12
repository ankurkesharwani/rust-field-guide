# RAII in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [How RAII Works in Rust](#how-raii-works-in-rust)
3. [The Drop Trait](#the-drop-trait)
4. [Drop Order](#drop-order)
5. [Explicit Drop](#explicit-drop)
6. [RAII in the Standard Library](#raii-in-the-standard-library)
7. [Implementing RAII for Your Own Types](#implementing-raii-for-your-own-types)
8. [Quick Reference](#quick-reference)
9. [Common Patterns](#common-patterns)
10. [Common Pitfalls](#common-pitfalls)
11. [Summary](#summary)

---

## Introduction

If you've written C, you've had bugs like this:

```c
FILE *f = fopen("data.txt", "r");
if (something_failed()) {
    return -1;  // Oops — file is never closed
}
fclose(f);
```

You open a resource, forget to close it on one of the early-return paths, and now you have a leak.

**RAII** (Resource Acquisition Is Initialization) is a pattern that ties a resource's lifetime to a variable's lifetime. When the variable is created, it acquires the resource. When the variable goes out of scope, it automatically releases the resource — no matter how the scope is exited (normal return, early return, or panic).

Rust builds RAII directly into the language through the `Drop` trait and its ownership system. You cannot forget to clean up, because the compiler does it for you.

---

## How RAII Works in Rust

The core idea is simple: when a value goes out of scope, Rust calls its cleanup code automatically.

```rust
use std::fs::File;
use std::io::Write;

fn write_greeting(path: &str) -> std::io::Result<()> {
    let mut file = File::create(path)?;  // resource acquired

    file.write_all(b"Hello!")?;

    Ok(())
}   // file goes out of scope — closed automatically, even if write_all returned Err
```

Compare this to languages without RAII:

| Language | How you close resources |
|----------|------------------------|
| C | Manual `fclose()` / `free()` — easy to forget |
| Java / Go | `finally` block or `defer` — explicit, but separate from resource creation |
| Python | `with` statement — explicit, opt-in |
| Rust | Automatic via `Drop` — no opt-in required, impossible to skip |

---

## The Drop Trait

`Drop` is the trait that makes RAII work. It has one method: `drop`, which Rust calls automatically when a value goes out of scope.

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("Releasing resource: {}", self.name);
    }
}

fn main() {
    let r = Resource { name: String::from("database connection") };
    println!("Resource acquired");
    // ... use r ...
}   // r goes out of scope here — Rust calls drop(&mut r) automatically
    // Output: "Releasing resource: database connection"
```

You almost never call `drop` yourself — Rust calls it for you. In fact, calling `r.drop()` directly is a compile error:

```rust
// r.drop(); // ERROR: explicit use of destructor method
drop(r);     // Use the free function `drop` instead (see Explicit Drop section)
```

### Drop runs on every exit path

This is the key guarantee: `drop` is called regardless of how the scope is exited.

```rust
fn might_fail(fail: bool) -> Result<(), &'static str> {
    let r = Resource { name: String::from("conn") };

    if fail {
        return Err("something went wrong");  // drop is called here too
    }

    println!("Success");
    Ok(())
}   // drop is called here on the success path
```

Both paths — `Ok` and `Err` — will print `"Releasing resource: conn"`. The resource is never leaked.

---

## Drop Order

When multiple values go out of scope at the same time, Rust drops them in **reverse order of creation** (last-in, first-out — like a stack).

```rust
struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("Dropping: {}", self.0);
    }
}

fn main() {
    let a = Noisy("first");
    let b = Noisy("second");
    let c = Noisy("third");
    println!("All created");
}
// Output:
// All created
// Dropping: third
// Dropping: second
// Dropping: first
```

This matters when resources depend on each other. For example, if a logger writes to a file, the logger should be dropped before the file — and because the file is created first, it will be dropped last, which is exactly the right order.

```rust
fn main() {
    let file = File::create("log.txt").unwrap();   // created first, dropped last
    let mut writer = BufWriter::new(file);          // created second, dropped first
    // BufWriter flushes and is dropped before File is closed — correct order
}
```

### Drop order within structs

Struct fields are dropped in **declaration order** (top to bottom).

```rust
struct App {
    config: Config,     // dropped first
    database: DbConn,   // dropped second
    logger: Logger,     // dropped third
}
```

---

## Explicit Drop

Sometimes you need to release a resource before the end of the scope — for example, releasing a lock early, or closing a file before renaming it.

Use the `drop()` free function, which takes ownership of the value and immediately runs its `Drop` implementation:

```rust
use std::fs::{self, File};
use std::io::Write;

fn atomic_write(path: &str, content: &[u8]) -> std::io::Result<()> {
    let tmp = format!("{}.tmp", path);
    let mut file = File::create(&tmp)?;
    file.write_all(content)?;
    file.flush()?;

    drop(file);              // close the file explicitly before renaming
    fs::rename(&tmp, path)?; // rename requires the file to be closed on Windows

    Ok(())
}
```

Another common use: releasing a lock guard early so other threads can proceed.

```rust
use std::sync::Mutex;

fn update(lock: &Mutex<Vec<i32>>, value: i32) {
    let mut data = lock.lock().unwrap();  // lock acquired
    data.push(value);
    drop(data);              // lock released here, not at end of function

    do_expensive_work();     // runs without holding the lock
}

fn do_expensive_work() { /* ... */ }
```

---

## RAII in the Standard Library

Rust's standard library is full of RAII types. You use them every day without thinking
about it; see the Quick Reference section below for the full list.

### MutexGuard example

```rust
use std::sync::Mutex;

fn main() {
    let counter = Mutex::new(0);

    {
        let mut guard = counter.lock().unwrap(); // lock acquired
        *guard += 1;
    } // guard dropped — lock released automatically

    // Other threads can now acquire the lock
    println!("{}", counter.lock().unwrap());
}
```

---

## Implementing RAII for Your Own Types

Any time your type manages a resource (a connection, a temp file, a handle, memory), implement `Drop` to release it.

### Example: a temporary directory

```rust
use std::fs;
use std::path::{Path, PathBuf};

struct TempDir {
    path: PathBuf,
}

impl TempDir {
    fn new(path: &str) -> std::io::Result<Self> {
        let p = PathBuf::from(path);
        fs::create_dir_all(&p)?;
        Ok(TempDir { path: p })
    }

    fn path(&self) -> &Path {
        &self.path
    }
}

impl Drop for TempDir {
    fn drop(&mut self) {
        if let Err(e) = fs::remove_dir_all(&self.path) {
            eprintln!("Failed to clean up temp dir: {}", e);
        }
    }
}

fn main() -> std::io::Result<()> {
    let tmp = TempDir::new("/tmp/my-work-dir")?;

    // Do work inside the temp directory...
    fs::write(tmp.path().join("output.txt"), b"results")?;

    Ok(())
}   // tmp dropped here — /tmp/my-work-dir is deleted automatically
```

### Example: a database transaction

```rust
struct Transaction<'a> {
    db: &'a mut Database,
    committed: bool,
}

impl<'a> Transaction<'a> {
    fn new(db: &'a mut Database) -> Self {
        db.begin();
        Transaction { db, committed: false }
    }

    fn commit(mut self) {
        self.db.commit();
        self.committed = true;
    }
}

impl<'a> Drop for Transaction<'a> {
    fn drop(&mut self) {
        if !self.committed {
            self.db.rollback(); // automatically roll back if not committed
        }
    }
}

fn transfer(db: &mut Database, from: u32, to: u32, amount: u64) -> Result<(), DbError> {
    let tx = Transaction::new(db);

    tx.db.debit(from, amount)?;   // if this fails, drop rolls back
    tx.db.credit(to, amount)?;    // if this fails, drop rolls back

    tx.commit();  // only commit if both succeed
    Ok(())
}

struct Database;
struct DbError;
impl Database {
    fn begin(&mut self) {}
    fn commit(&mut self) {}
    fn rollback(&mut self) {}
    fn debit(&mut self, _: u32, _: u64) -> Result<(), DbError> { Ok(()) }
    fn credit(&mut self, _: u32, _: u64) -> Result<(), DbError> { Ok(()) }
}
```

---

## Quick Reference

| Guarantee | How it works |
|-----------|-------------|
| No resource leaks | `Drop` always runs when a value goes out of scope |
| No use-after-free | Ownership system prevents using values after drop |
| No double-free | One owner means drop runs exactly once |
| Correct cleanup order | Reverse declaration order, enforced by the compiler |

| Type | Resource acquired | Released on drop |
|------|-------------------|------------------|
| `File` | OS file handle | File is closed |
| `MutexGuard<T>` | Mutex lock | Lock is released |
| `RwLockReadGuard<T>` | Read lock | Lock is released |
| `RwLockWriteGuard<T>` | Write lock | Lock is released |
| `BufWriter<W>` | Write buffer | Buffer is flushed |
| `TcpStream` | Network connection | Connection is closed |
| `Box<T>` | Heap allocation | Memory is freed |
| `Vec<T>` | Heap buffer | Memory is freed |
| `String` | Heap buffer | Memory is freed |
| `Rc<T>` / `Arc<T>` | Reference count | Memory freed when count reaches 0 |

---

## Common Patterns

### Pattern: scope guard (run code on exit)

Sometimes you want to run cleanup code at scope exit without building a full struct. A minimal scope guard works well for this:

```rust
struct OnDrop<F: FnOnce()>(Option<F>);

impl<F: FnOnce()> OnDrop<F> {
    fn new(f: F) -> Self {
        OnDrop(Some(f))
    }

    fn disarm(mut self) {
        self.0 = None; // cancel the cleanup
    }
}

impl<F: FnOnce()> Drop for OnDrop<F> {
    fn drop(&mut self) {
        if let Some(f) = self.0.take() {
            f();
        }
    }
}

fn main() {
    println!("Starting");

    let _guard = OnDrop::new(|| println!("Cleanup ran"));

    println!("Working...");
    // guard is dropped here — prints "Cleanup ran"
}
```

> In real projects, prefer the [`scopeguard`](https://crates.io/crates/scopeguard) crate, which provides a polished version of this pattern.

### Pattern: release a lock before awaiting in async code

In async Rust, holding a `MutexGuard` across an `.await` point is a common mistake — it holds the lock while waiting, blocking other tasks. Drop it explicitly first:

```rust
use std::sync::Mutex;

async fn process(shared: &Mutex<Vec<i32>>) {
    let snapshot = {
        let guard = shared.lock().unwrap();
        guard.clone()   // copy the data out
    };  // guard dropped here — lock released before await

    some_async_work(&snapshot).await;
}

async fn some_async_work(_data: &[i32]) {}
```

### Pattern: close a file before renaming (Windows-safe)

Windows does not allow renaming a file that is currently open. Close it explicitly with `drop` before the rename:

```rust
use std::fs::{self, File};
use std::io::Write;

fn safe_write(final_path: &str, content: &str) -> std::io::Result<()> {
    let tmp_path = format!("{}.tmp", final_path);

    let mut file = File::create(&tmp_path)?;
    file.write_all(content.as_bytes())?;
    file.flush()?;
    drop(file);                          // closed here

    fs::rename(&tmp_path, final_path)?;  // safe to rename now
    Ok(())
}
```

### Pattern: nested resources in the right order

Create resources in the order that ensures they are dropped correctly (reverse creation = correct teardown):

```rust
fn run() -> std::io::Result<()> {
    let file = File::create("output.txt")?;    // dropped last
    let mut buf = BufWriter::new(file);         // dropped first (flushes into file)

    for i in 0..100 {
        writeln!(buf, "line {}", i)?;
    }

    Ok(())
    // buf is dropped first: flushes buffered data to file
    // file is dropped second: OS closes the file handle
    // Correct order guaranteed by declaration order
}

use std::fs::File;
use std::io::{BufWriter, Write};
```

### Pattern: connection with automatic cleanup

```rust
struct DbConnection {
    id: u32,
}

impl DbConnection {
    fn connect(id: u32) -> Self {
        println!("Opening connection {}", id);
        DbConnection { id }
    }

    fn query(&self, sql: &str) -> Vec<String> {
        println!("[conn {}] Running: {}", self.id, sql);
        vec![]
    }
}

impl Drop for DbConnection {
    fn drop(&mut self) {
        println!("Closing connection {}", self.id);
    }
}

fn main() {
    let conn = DbConnection::connect(42);
    let _results = conn.query("SELECT * FROM topics");
}
// Output:
// Opening connection 42
// [conn 42] Running: SELECT * FROM topics
// Closing connection 42
```

---

## Common Pitfalls

### Pitfall 1: BufWriter silently drops buffered data on error

`BufWriter` flushes on drop, but if the flush fails, the error is silently ignored. Always flush explicitly when correctness matters:

```rust
use std::fs::File;
use std::io::{BufWriter, Write};

fn write_data(path: &str) -> std::io::Result<()> {
    let file = File::create(path)?;
    let mut writer = BufWriter::new(file);

    writeln!(writer, "important data")?;

    writer.flush()?;  // explicit flush — catches errors that drop() would swallow
    Ok(())
}
```

### Pitfall 2: forgetting that drop order matters for locks

If two `MutexGuard`s exist in the same scope, be careful about the order they are dropped. Dropping in the wrong order can cause a deadlock if other threads are involved.

```rust
use std::sync::Mutex;

fn careful(a: &Mutex<i32>, b: &Mutex<i32>) {
    let guard_a = a.lock().unwrap();
    let guard_b = b.lock().unwrap();

    // guard_b dropped first, then guard_a (reverse declaration order)
    // Make sure all callers agree on lock acquisition order to avoid deadlock
}
```

### Pitfall 3: implementing Drop prevents moving out of fields

Once you implement `Drop` for a struct, Rust won't let you move individual fields out of it (because `drop` still needs to run on the whole struct):

```rust
struct Wrapper {
    data: String,
}

impl Drop for Wrapper {
    fn drop(&mut self) { /* cleanup */ }
}

fn consume(w: Wrapper) -> String {
    // w.data  // ERROR: cannot move out of `w` because it implements `Drop`

    // Solution: use Option to take ownership
    // Or restructure so the wrapped value is returned before drop runs
    w.data.clone() // workaround: clone instead
}
```

The clean solution is to wrap the field in `Option<T>` and use `.take()` in drop:

```rust
struct Wrapper {
    data: Option<String>,
}

impl Drop for Wrapper {
    fn drop(&mut self) {
        if let Some(data) = self.data.take() {
            println!("Dropping: {}", data);
        }
    }
}
```

---

## Summary

See the Quick Reference section above for what RAII guarantees and which standard
library types rely on it.

### When to implement Drop

Implement `Drop` when your type:
- Holds a raw OS resource (file descriptor, socket, handle)
- Manages heap memory not tracked by Rust (e.g. via FFI)
- Needs to run side-effectful cleanup (flush a buffer, commit/rollback a transaction, log a shutdown event)

Do **not** implement `Drop` just because you have heap data — `Vec`, `String`, `Box`, etc. already handle their own cleanup, and Rust will call their `drop` automatically.

### Key rules

1. `drop` is called automatically — you never need to call it manually
2. Use `drop(value)` to force early cleanup within a scope
3. Drop order is reverse of creation order
4. `Drop` and `Copy` are mutually exclusive: a type that manages cleanup cannot be trivially copied, because a bitwise `Copy` would produce two values that both think they own the same resource, so both would run `drop` on it
