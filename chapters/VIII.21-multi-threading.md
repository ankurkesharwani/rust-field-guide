# Multi-threading in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Spawning Threads](#spawning-threads)
3. [Moving Data into Threads](#moving-data-into-threads)
4. [Joining Threads](#joining-threads)
5. [Shared State with Arc and Mutex](#shared-state-with-arc-and-mutex)
6. [RwLock: Multiple Readers or One Writer](#rwlock-multiple-readers-or-one-writer)
7. [Message Passing with Channels](#message-passing-with-channels)
8. [Thread-Local Storage](#thread-local-storage)
9. [The Send and Sync Traits](#the-send-and-sync-traits)
10. [Data Parallelism with Rayon](#data-parallelism-with-rayon)
11. [Quick Reference](#quick-reference)
12. [Common Patterns](#common-patterns)
13. [Common Pitfalls](#common-pitfalls)
14. [Summary](#summary)

---

## Introduction

Rust's threading model is built on two guarantees that most other languages do not provide at compile time:

- **No data races** — the borrow checker prevents two threads from simultaneously accessing the same data when at least one is writing.
- **No use-after-free across threads** — ownership ensures data cannot outlive the thread that owns it.

A data race is specifically two or more threads accessing the same memory at the same
time with no synchronization, where at least one access is a write; the result depends
on timing that varies from run to run, which is what makes these bugs so hard to
reproduce in languages that only catch them at runtime, if at all. Rust's ordinary
borrowing rules (one mutable reference, or many immutable ones, never both) already rule
this out within a single thread; the `Send` and `Sync` traits (covered later in this
chapter) extend the same rule across thread boundaries, so the compiler rejects a data
race before the program runs instead of leaving it to show up under load in production.

This means many threading bugs that would only appear at runtime in C++, Go, or Java are rejected by the Rust compiler before your program even runs.

Rust threads map directly to OS threads (1:1 model). Each thread has its own stack and runs in parallel. For cooperative, lightweight concurrency, see the async Rust document instead.

---

## Spawning Threads

Use `std::thread::spawn` to start a new OS thread. It takes a closure and runs it on a new thread.

```rust
use std::thread;

fn main() {
    thread::spawn(|| {
        println!("Hello from a new thread!");
    });

    println!("Hello from the main thread!");
    // Warning: the program may exit before the spawned thread prints anything
}
```

The return value of `spawn` is a `JoinHandle<T>`, where `T` is the return type of the closure. If you don't join it, the thread is detached — it may or may not finish before `main` exits.

---

## Moving Data into Threads

Closures passed to `spawn` must be `'static` — they cannot borrow local variables, because the thread may outlive the scope where the variable lives. Use the `move` keyword to transfer ownership into the closure.

```rust
use std::thread;

fn main() {
    let message = String::from("hello from thread");

    thread::spawn(move || {
        // `message` is moved into this closure
        println!("{}", message);
    });

    // println!("{}", message); // ERROR: message was moved
}
```

If you need to share data between multiple threads, use `Arc` (see next section). Moving gives each thread exclusive ownership; `Arc` lets multiple threads share ownership.

```rust
use std::thread;
use std::sync::Arc;

fn main() {
    let data = Arc::new(vec![1, 2, 3, 4, 5]);

    let data_clone = Arc::clone(&data);
    thread::spawn(move || {
        println!("Thread sees: {:?}", data_clone);
    });

    println!("Main sees: {:?}", data);
}
```

---

## Joining Threads

Call `.join()` on a `JoinHandle` to wait for a thread to finish. `join()` returns `Result<T, Box<dyn Any>>` — `Ok(T)` if the thread completed, `Err` if it panicked.

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        let sum: u64 = (1..=1_000_000).sum();
        sum
    });

    let result = handle.join().expect("Thread panicked");
    println!("Sum: {}", result);
}
```

### Joining multiple threads

```rust
use std::thread;

fn main() {
    let handles: Vec<_> = (0..5)
        .map(|i| {
            thread::spawn(move || {
                println!("Thread {} working", i);
                i * i
            })
        })
        .collect();

    let results: Vec<u64> = handles
        .into_iter()
        .map(|h| h.join().unwrap())
        .collect();

    println!("Results: {:?}", results);
}
```

---

## Shared State with Arc and Mutex

To share mutable data between threads, combine `Arc` (shared ownership across threads) with `Mutex` (mutual exclusion).

- `Arc<T>` — atomically reference-counted pointer; allows multiple owners across threads
- `Mutex<T>` — wraps a value so only one thread can access it at a time

Neither works alone here: `Mutex<T>` makes it safe for one thread at a time to mutate
the value, but `thread::spawn`'s closures each need to own their own handle to that same
`Mutex`, and a plain `Mutex<T>` has exactly one owner, so moving it into one closure
would leave none for the rest. `Arc<T>` solves the ownership problem by letting many
handles share one heap allocation (cloning an `Arc` bumps a reference count instead of
copying the data), so `Arc<Mutex<T>>` gives every thread its own owned handle to the
same underlying value, with `Mutex` still guarding access to it once they're there.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0u32));

    let handles: Vec<_> = (0..10)
        .map(|_| {
            let counter = Arc::clone(&counter);
            thread::spawn(move || {
                let mut guard = counter.lock().unwrap(); // blocks until lock is acquired
                *guard += 1;
            })  // guard dropped here — lock released
        })
        .collect();

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Counter: {}", counter.lock().unwrap()); // 10
}
```

### How Mutex works

`lock()` returns a `MutexGuard<T>`, which implements `Deref` (so you can use it like a `&mut T`) and `Drop` (so the lock is released when the guard goes out of scope — RAII in action).

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(String::from("hello"));

    {
        let mut s = m.lock().unwrap();
        s.push_str(", world");
    }  // lock released here

    println!("{}", m.lock().unwrap());
}
```

### Poisoned mutexes

If a thread panics while holding a lock, the `Mutex` is marked as "poisoned". Subsequent `lock()` calls return `Err`. Handle this explicitly if your program needs to recover:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(0));
    let data_clone = Arc::clone(&data);

    let _ = thread::spawn(move || {
        let _guard = data_clone.lock().unwrap();
        panic!("thread panicked while holding lock");
    }).join();

    match data.lock() {
        Ok(guard) => println!("Value: {}", *guard),
        Err(poisoned) => {
            let guard = poisoned.into_inner(); // recover the guard anyway
            println!("Recovered poisoned value: {}", *guard);
        }
    }
}
```

---

## RwLock: Multiple Readers or One Writer

`RwLock<T>` allows many concurrent readers **or** one exclusive writer — useful when reads are far more frequent than writes.

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let config = Arc::new(RwLock::new(String::from("initial config")));

    // Spawn multiple reader threads
    let readers: Vec<_> = (0..3)
        .map(|i| {
            let config = Arc::clone(&config);
            thread::spawn(move || {
                let value = config.read().unwrap(); // shared read lock
                println!("Reader {}: {}", i, *value);
            })
        })
        .collect();

    // Spawn a writer thread
    let config_clone = Arc::clone(&config);
    let writer = thread::spawn(move || {
        let mut value = config_clone.write().unwrap(); // exclusive write lock
        *value = String::from("updated config");
        println!("Writer updated config");
    });

    for r in readers { r.join().unwrap(); }
    writer.join().unwrap();
}
```

### RwLock vs Mutex

| | `Mutex<T>` | `RwLock<T>` |
|---|---|---|
| Concurrent reads | No | Yes |
| Exclusive write | Yes | Yes |
| Best for | Write-heavy or mixed | Read-heavy |
| Overhead | Lower | Higher |

---

## Message Passing with Channels

Rust's standard library provides **multi-producer, single-consumer (mpsc)** channels. Multiple threads can send values into a channel; one thread receives them.

> "Do not communicate by sharing memory; share memory by communicating." — Go proverb, equally applicable here.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let messages = vec!["one", "two", "three"];
        for msg in messages {
            tx.send(msg).unwrap();
        }
        // tx is dropped here — channel is closed, rx will stop iterating
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```

### Multiple producers

Clone the sender to give multiple threads access:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    let handles: Vec<_> = (0..5)
        .map(|i| {
            let tx = tx.clone();
            thread::spawn(move || {
                tx.send(format!("message from thread {}", i)).unwrap();
            })
        })
        .collect();

    drop(tx); // drop original so rx ends when all clones are dropped

    for msg in rx {
        println!("{}", msg);
    }

    for h in handles { h.join().unwrap(); }
}
```

### Synchronous (bounded) channels

`mpsc::sync_channel(n)` creates a channel with capacity `n`. Senders block when the buffer is full, providing backpressure:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::sync_channel(2); // buffer holds 2 messages

    thread::spawn(move || {
        for i in 0..5 {
            println!("Sending {}", i);
            tx.send(i).unwrap(); // blocks if buffer is full
        }
    });

    thread::sleep(std::time::Duration::from_millis(10));
    for val in rx {
        println!("Received {}", val);
    }
}
```

---

## Thread-Local Storage

`thread_local!` declares a variable that each thread owns independently. Useful for per-thread caches or ID counters without any synchronization overhead.

```rust
use std::cell::RefCell;
use std::thread;

thread_local! {
    static BUFFER: RefCell<Vec<String>> = RefCell::new(Vec::new());
}

fn log(msg: &str) {
    BUFFER.with(|buf| {
        buf.borrow_mut().push(msg.to_string());
    });
}

fn flush() -> Vec<String> {
    BUFFER.with(|buf| buf.borrow_mut().drain(..).collect())
}

fn main() {
    let handles: Vec<_> = (0..3)
        .map(|i| {
            thread::spawn(move || {
                log(&format!("thread {} event A", i));
                log(&format!("thread {} event B", i));
                flush()
            })
        })
        .collect();

    for h in handles {
        println!("{:?}", h.join().unwrap()); // each thread has its own buffer
    }
}
```

---

## The Send and Sync Traits

These are marker traits the compiler uses to enforce thread safety. You rarely implement them yourself, but understanding them explains why certain types can or cannot cross thread boundaries.

| Trait | Meaning | Implemented by |
|-------|---------|----------------|
| `Send` | Safe to transfer ownership to another thread | Most types; not `Rc<T>`, raw pointers |
| `Sync` | Safe to share a reference between threads | Most types; not `Cell<T>`, `RefCell<T>` |

```rust
use std::rc::Rc;
use std::sync::Arc;

fn needs_send<T: Send>(_: T) {}

fn main() {
    let arc = Arc::new(42);
    needs_send(arc); // OK: Arc<i32> is Send

    let rc = Rc::new(42);
    // needs_send(rc); // ERROR: Rc<i32> is not Send
}
```

The compiler automatically derives `Send` and `Sync` for your structs when all fields implement them. If you use a non-`Send` type inside a struct that crosses threads, you get a compile error — not a runtime race condition.

---

## Data Parallelism with Rayon

The [`rayon`](https://crates.io/crates/rayon) crate makes it trivial to parallelize iterator pipelines. Replace `.iter()` with `.par_iter()` and rayon splits the work across a thread pool automatically.

```toml
# Cargo.toml
[dependencies]
rayon = "1"
```

```rust
use rayon::prelude::*;

fn main() {
    let numbers: Vec<u64> = (1..=1_000_000).collect();

    // Sequential
    let sum_seq: u64 = numbers.iter().sum();

    // Parallel — same result, uses all CPU cores
    let sum_par: u64 = numbers.par_iter().sum();

    assert_eq!(sum_seq, sum_par);
    println!("Sum: {}", sum_par);
}
```

### Parallel map and filter

```rust
use rayon::prelude::*;

fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    (2..=(n as f64).sqrt() as u64).all(|i| n % i != 0)
}

fn main() {
    let primes: Vec<u64> = (2u64..100_000)
        .collect::<Vec<_>>()
        .par_iter()
        .copied()
        .filter(|&n| is_prime(n))
        .collect();

    println!("Found {} primes", primes.len());
}
```

### When to use Rayon vs manual threads

| | Manual threads | Rayon |
|---|---|---|
| Use case | Long-running tasks, I/O, messaging | CPU-bound data transformations |
| Setup | Explicit | Drop-in iterator replacement |
| Thread count | Manual | Automatic (matches CPU cores) |
| Work distribution | Manual | Automatic (work-stealing) |

---

## Quick Reference

### Choosing the right primitive

| Problem | Solution |
|---------|----------|
| Run work concurrently | `thread::spawn` |
| Share immutable data across threads | `Arc<T>` |
| Share mutable data, exclusive access | `Arc<Mutex<T>>` |
| Share mutable data, many readers | `Arc<RwLock<T>>` |
| Send values between threads | `mpsc::channel` |
| Parallelize iterator pipelines | `rayon` |
| Per-thread state | `thread_local!` |
| Initialize once, read forever | `OnceLock` |

### RwLock vs Mutex

| | `Mutex<T>` | `RwLock<T>` |
|---|---|---|
| Concurrent reads | No | Yes |
| Exclusive write | Yes | Yes |
| Best for | Write-heavy or mixed | Read-heavy |
| Overhead | Lower | Higher |

### Send and Sync

| Trait | Meaning | Implemented by |
|-------|---------|----------------|
| `Send` | Safe to transfer ownership to another thread | Most types; not `Rc<T>`, raw pointers |
| `Sync` | Safe to share a reference between threads | Most types; not `Cell<T>`, `RefCell<T>` |

### Manual threads vs Rayon

| | Manual threads | Rayon |
|---|---|---|
| Use case | Long-running tasks, I/O, messaging | CPU-bound data transformations |
| Setup | Explicit | Drop-in iterator replacement |
| Thread count | Manual | Automatic (matches CPU cores) |
| Work distribution | Manual | Automatic (work-stealing) |

---

## Common Patterns

### Pattern: worker pool with a channel

Distribute work to a fixed number of threads and collect results:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (work_tx, work_rx) = mpsc::channel::<u64>();
    let (result_tx, result_rx) = mpsc::channel::<u64>();

    let work_rx = std::sync::Arc::new(std::sync::Mutex::new(work_rx));

    // Spawn worker threads
    let workers: Vec<_> = (0..4)
        .map(|_| {
            let work_rx = std::sync::Arc::clone(&work_rx);
            let result_tx = result_tx.clone();
            thread::spawn(move || {
                loop {
                    let job = {
                        let rx = work_rx.lock().unwrap();
                        rx.recv()
                    };
                    match job {
                        Ok(n) => result_tx.send(n * n).unwrap(),
                        Err(_) => break, // channel closed
                    }
                }
            })
        })
        .collect();

    // Send work
    for i in 1u64..=20 {
        work_tx.send(i).unwrap();
    }
    drop(work_tx);  // signal workers to stop
    drop(result_tx);

    // Collect results
    let mut results: Vec<u64> = result_rx.iter().collect();
    results.sort();
    println!("{:?}", results);

    for w in workers { w.join().unwrap(); }
}
```

### Pattern: parallel read, single writer

Use `Arc<RwLock<T>>` when many threads need to read a shared value but occasional writes happen:

```rust
use std::sync::{Arc, RwLock};
use std::thread;

struct Cache {
    data: Arc<RwLock<std::collections::HashMap<String, String>>>,
}

impl Cache {
    fn new() -> Self {
        Cache { data: Arc::new(RwLock::new(std::collections::HashMap::new())) }
    }

    fn get(&self, key: &str) -> Option<String> {
        self.data.read().unwrap().get(key).cloned()
    }

    fn set(&self, key: String, value: String) {
        self.data.write().unwrap().insert(key, value);
    }
}
```

### Pattern: one-time initialization with OnceLock

Use `std::sync::OnceLock` for values that are initialized once and then read-only:

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<String> = OnceLock::new();

fn get_config() -> &'static str {
    CONFIG.get_or_init(|| {
        // Expensive initialization runs only once
        std::env::var("APP_CONFIG").unwrap_or_else(|_| "default".to_string())
    })
}

fn main() {
    let handles: Vec<_> = (0..5)
        .map(|_| std::thread::spawn(|| println!("{}", get_config())))
        .collect();

    for h in handles { h.join().unwrap(); }
}
```

---

## Common Pitfalls

### Pitfall 1: deadlock from acquiring locks in different orders

If two threads each hold one lock and try to acquire the other, they deadlock forever.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let a = Arc::new(Mutex::new(0));
    let b = Arc::new(Mutex::new(0));

    let a1 = Arc::clone(&a);
    let b1 = Arc::clone(&b);

    // Thread 1 locks a then b
    let t1 = thread::spawn(move || {
        let _ga = a1.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(10));
        let _gb = b1.lock().unwrap(); // waits for thread 2 to release b — deadlock
    });

    // Thread 2 locks b then a
    let t2 = thread::spawn(move || {
        let _gb = b.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(10));
        let _ga = a.lock().unwrap(); // waits for thread 1 to release a — deadlock
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```

**Fix:** Always acquire locks in the same global order across all threads.

### Pitfall 2: holding a MutexGuard longer than needed

The lock is held for the entire lifetime of the guard. Keep critical sections short:

```rust
use std::sync::Mutex;

fn main() {
    let data = Mutex::new(vec![1, 2, 3]);

    // Bad: lock held during expensive work
    {
        let guard = data.lock().unwrap();
        let _result = expensive_computation(&guard); // other threads blocked here
    }

    // Good: copy out what you need, release the lock, then compute
    let snapshot = data.lock().unwrap().clone();
    let _result = expensive_computation(&snapshot);  // lock not held
}

fn expensive_computation(_: &[i32]) -> u64 { 0 }
```

### Pitfall 3: forgetting to drop the original channel sender

`rx` only ends iteration (returns `None`) when **all** senders are dropped. If you clone `tx` and forget to drop the original, `rx` never ends.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<i32>();
    let tx2 = tx.clone();

    thread::spawn(move || {
        tx2.send(1).unwrap();
        // tx2 dropped here
    });

    drop(tx); // Without this, rx.iter() loops forever

    for val in rx {
        println!("{}", val);
    }
}
```

---

## Summary

See the Quick Reference above for which primitive to reach for.

### Key rules

1. Data shared across threads must be `Send` and/or `Sync` — the compiler enforces this
2. `Arc` provides shared ownership; `Mutex` provides safe mutation — they are almost always used together
3. Keep lock scopes short to reduce contention
4. Always acquire locks in a consistent global order to avoid deadlocks
5. Drop the original `mpsc` sender when using clones, so receivers know when to stop
