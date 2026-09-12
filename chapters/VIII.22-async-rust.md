# Async Rust

## Table of Contents

1. [Introduction](#introduction)
2. [async and await Basics](#async-and-await-basics)
3. [Futures](#futures)
4. [Choosing a Runtime](#choosing-a-runtime)
5. [Spawning Tasks](#spawning-tasks)
6. [Async Channels](#async-channels)
7. [Selecting Between Futures](#selecting-between-futures)
8. [Timeouts and Cancellation](#timeouts-and-cancellation)
9. [Async Streams](#async-streams)
10. [Sharing State in Async Code](#sharing-state-in-async-code)
11. [Quick Reference](#quick-reference)
12. [Common Patterns](#common-patterns)
13. [Common Pitfalls](#common-pitfalls)
14. [Async vs Threads](#async-vs-threads)
15. [Summary](#summary)

---

## Introduction

Async Rust is designed for programs that spend most of their time **waiting** — waiting for network responses, file I/O, timers, or database queries. Instead of blocking an OS thread during each wait, async code suspends the current task and lets other tasks run, all within a small number of threads.

### The core idea

In synchronous code, a blocked thread does nothing:

```
Thread 1: [do work]---[wait for network]---[do work]
Thread 2: [do work]---[wait for disk]-----[do work]
```

In async code, a single thread can interleave many tasks during their waits:

```
Thread 1: [task A work][task B work][task C work][task A work][task B work]...
           task A waits ^           ^ task A ready
```

This makes async ideal for I/O-bound workloads (web servers, database clients, CLI tools fetching remote data) where you want to handle many concurrent operations without spawning thousands of OS threads.

### Async is not always better

| | Async | OS Threads |
|---|---|---|
| Best for | Many concurrent I/O operations | CPU-heavy parallel computation |
| Overhead | Very low per task | ~8 MB stack per thread |
| Blocking code | Must avoid — stalls the runtime | Fine — each thread is independent |
| Complexity | Higher | Lower |

---

## async and await Basics

Mark a function with `async` to make it return a `Future`. Use `.await` inside an async function to pause execution until the future completes.

```rust
async fn fetch_length(url: &str) -> usize {
    // .await suspends this task until the HTTP response arrives
    let body = reqwest::get(url).await.unwrap().text().await.unwrap();
    body.len()
}
```

A plain async function call does nothing by itself: it returns a `Future`. You must `.await` it (or hand it to a runtime) to drive it forward. This is the opposite of `thread::spawn`, which starts running the closure on an OS thread immediately, with or without anyone waiting on the result; an `async fn` is inert until something polls it.

```rust
async fn say_hello() {
    println!("Hello!");
}

async fn main_task() {
    say_hello(); // Does nothing — the future is created but not polled
    say_hello().await; // This actually runs it
}
```

### Running sequential async work

```rust
use tokio; // see Choosing a Runtime

#[tokio::main]
async fn main() {
    let a = step_one().await;
    let b = step_two(a).await;
    println!("Result: {}", b);
}

async fn step_one() -> u32 { 1 }
async fn step_two(n: u32) -> u32 { n + 1 }
```

### Running concurrent async work

Use `tokio::join!` to run multiple futures concurrently on the same task (not in parallel on separate threads — just interleaved):

```rust
use tokio;

#[tokio::main]
async fn main() {
    let (a, b, c) = tokio::join!(fetch("url1"), fetch("url2"), fetch("url3"));
    println!("{} {} {}", a, b, c);
}

async fn fetch(_url: &str) -> String { String::from("response") }
```

`join!` runs all futures concurrently — if one waits on I/O, the others make progress. All must complete before `join!` returns.

---

## Futures

A `Future` represents a value that will be available at some point. It is a trait:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

In practice, you almost never implement `Future` manually — the compiler generates an implementation for every `async fn` and `async` block. But understanding the model helps when debugging:

- A future does nothing until it is **polled**.
- When polled, it either returns `Poll::Ready(value)` (done) or `Poll::Pending` (not done yet, will be woken up later).
- The runtime calls `poll` on your future, which calls `poll` on inner futures, and so on.

```rust
// async fn is syntactic sugar for a function returning impl Future
async fn compute() -> u32 {
    42
}

// Roughly equivalent to:
fn compute_manual() -> impl std::future::Future<Output = u32> {
    async { 42 }
}
```

---

## Choosing a Runtime

Rust's `async`/`await` syntax is built into the language, but the **runtime** that drives futures is not — you choose it as a dependency. The most common choice is **Tokio**.

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

The `#[tokio::main]` attribute macro converts your `async fn main()` into a regular `fn main()` that starts the Tokio runtime:

```rust
#[tokio::main]
async fn main() {
    println!("Running on Tokio");
}

// Expands roughly to:
fn main() {
    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async {
            println!("Running on Tokio");
        });
}
```

### Runtime flavors

```toml
# Multi-threaded (default) — uses a thread pool, best for most applications
tokio = { version = "1", features = ["full"] }

# Single-threaded — useful for environments without multiple threads
```

```rust
// Multi-threaded runtime
#[tokio::main]
async fn main() { /* ... */ }

// Single-threaded runtime
#[tokio::main(flavor = "current_thread")]
async fn main() { /* ... */ }
```

---

## Spawning Tasks

`tokio::spawn` creates an independent async task that runs concurrently with the current task. Unlike `join!`, spawned tasks can run on different threads in the thread pool.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        println!("Running in a separate task");
        42u32
    });

    println!("Main task continues immediately");

    let result = handle.await.unwrap(); // wait for the spawned task
    println!("Task returned: {}", result);
}
```

### Spawning multiple tasks and waiting for all

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..5 {
        set.spawn(async move {
            tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
            i * i
        });
    }

    while let Some(result) = set.join_next().await {
        println!("Task result: {}", result.unwrap());
    }
}
```

### spawn vs join!

| | `tokio::spawn` | `tokio::join!` |
|---|---|---|
| Tasks | Run independently | Run on current task |
| Cancellation | Handle can be aborted | Cancelled when future is dropped |
| Errors | Panics are caught in JoinHandle | Panics propagate immediately |
| Use when | Fire-and-forget, parallelism | Wait for related sub-operations |

---

## Async Channels

For communication between tasks, Tokio provides async-aware channel types that can `.await` on send and receive operations.

### mpsc: multiple producers, single consumer

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32); // buffer of 32 messages

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
        }
        // tx dropped — channel closed
    });

    while let Some(value) = rx.recv().await {
        println!("Received: {}", value);
    }
}
```

### oneshot: send one value from one task to another

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        let result = do_work().await;
        tx.send(result).unwrap();
    });

    let value = rx.await.unwrap();
    println!("Got: {}", value);
}

async fn do_work() -> u32 { 42 }
```

### broadcast: one sender, many receivers

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, _) = broadcast::channel(16);

    let mut rx1 = tx.subscribe();
    let mut rx2 = tx.subscribe();

    tokio::spawn(async move {
        tx.send("event").unwrap();
    });

    println!("rx1: {}", rx1.recv().await.unwrap());
    println!("rx2: {}", rx2.recv().await.unwrap());
}
```

### watch: always holds the latest value

Useful for configuration updates or state notifications — receivers always see the most recent value, older values are discarded.

```rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = watch::channel("initial");

    tokio::spawn(async move {
        tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
        tx.send("updated").unwrap();
    });

    rx.changed().await.unwrap(); // waits until a new value is sent
    println!("Config is now: {}", *rx.borrow());
}
```

---

## Selecting Between Futures

`tokio::select!` polls multiple futures simultaneously and proceeds with whichever completes first. The others are cancelled.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        result = fetch_data() => println!("Fetch finished first: {}", result),
        _ = sleep(Duration::from_secs(1)) => println!("Timed out"),
    }
}

async fn fetch_data() -> String {
    sleep(Duration::from_millis(500)).await;
    String::from("data")
}
```

### select! in a loop

A common pattern: process messages until a shutdown signal. The `deadline` future is
reused across every loop iteration inside `select!`, and `select!` only takes futures by
value or by mutable reference, not by repeated ownership, so it needs a stable address
to poll each time; `tokio::pin!` pins it to the stack so `&mut deadline` can be handed
to `select!` on every pass instead of being consumed after the first one.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(8);

    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(format!("msg {}", i)).await.unwrap();
            sleep(Duration::from_millis(100)).await;
        }
    });

    let deadline = sleep(Duration::from_millis(350));
    tokio::pin!(deadline);

    loop {
        tokio::select! {
            Some(msg) = rx.recv() => println!("Got: {}", msg),
            _ = &mut deadline => {
                println!("Deadline reached, stopping");
                break;
            }
        }
    }
}
```

---

## Timeouts and Cancellation

Wrap any future with `tokio::time::timeout` to fail it if it doesn't complete within a duration:

```rust
use tokio::time::{timeout, Duration};

#[tokio::main]
async fn main() {
    match timeout(Duration::from_secs(1), slow_operation()).await {
        Ok(result) => println!("Finished: {}", result),
        Err(_) => println!("Timed out"),
    }
}

async fn slow_operation() -> u32 {
    tokio::time::sleep(tokio::time::Duration::from_secs(5)).await;
    42
}
```

### Cancellation via task abort

Spawned tasks can be cancelled externally via `JoinHandle::abort()`:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        loop {
            tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
            println!("tick");
        }
    });

    tokio::time::sleep(tokio::time::Duration::from_millis(350)).await;
    handle.abort();

    match handle.await {
        Err(e) if e.is_cancelled() => println!("Task was cancelled"),
        _ => {}
    }
}
```

> **Important:** When a future is cancelled (dropped), it stops at the next `.await` point. Any work done after that point is lost. Design cancellation-sensitive code to be safe at every `.await`.

---

## Async Streams

A `Stream` is like an async iterator — it produces values over time. Tokio provides stream utilities via the `tokio-stream` crate.

```toml
[dependencies]
tokio-stream = "0.1"
```

```rust
use tokio_stream::{self as stream, StreamExt};

#[tokio::main]
async fn main() {
    let mut s = stream::iter(vec![1, 2, 3, 4, 5]);

    while let Some(value) = s.next().await {
        println!("{}", value);
    }
}
```

### Stream from a channel receiver

```rust
use tokio::sync::mpsc;
use tokio_stream::wrappers::ReceiverStream;
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel(8);

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
        }
    });

    let mut stream = ReceiverStream::new(rx);
    while let Some(value) = stream.next().await {
        println!("{}", value);
    }
}
```

### Applying stream combinators

```rust
use tokio_stream::{self as stream, StreamExt};

#[tokio::main]
async fn main() {
    let results: Vec<u32> = stream::iter(0..10)
        .filter(|x| x % 2 == 0)
        .map(|x| x * x)
        .collect()
        .await;

    println!("{:?}", results); // [0, 4, 16, 36, 64]
}
```

---

## Sharing State in Async Code

### Use tokio::sync::Mutex, not std::sync::Mutex

`std::sync::Mutex` blocks the thread when contended. In async code this stalls the runtime — all tasks on that thread are frozen. Use `tokio::sync::Mutex` instead, which suspends only the current task.

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(vec![]));

    let handles: Vec<_> = (0..5)
        .map(|i| {
            let data = Arc::clone(&data);
            tokio::spawn(async move {
                let mut guard = data.lock().await; // suspends task, not thread
                guard.push(i);
            })
        })
        .collect();

    for h in handles { h.await.unwrap(); }
    println!("{:?}", data.lock().await);
}
```

### When std::sync::Mutex is actually fine

If you lock, do a small operation, and unlock without ever awaiting while holding the lock, `std::sync::Mutex` is fine (and faster):

```rust
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0u32));

    let c = Arc::clone(&counter);
    tokio::spawn(async move {
        // lock, increment, unlock — no .await while holding the lock
        *c.lock().unwrap() += 1;

        // Now it's safe to await
        tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
    }).await.unwrap();

    println!("{}", counter.lock().unwrap());
}
```

---

## Quick Reference

### Core concepts

| Concept | What it is |
|---------|-----------|
| `async fn` | Returns a `Future`; does nothing until polled |
| `.await` | Suspends current task until a future is ready |
| Runtime | Drives futures to completion (Tokio, async-std) |
| Task | Lightweight concurrent unit of work (`tokio::spawn`) |
| `select!` | Runs multiple futures, proceeds with first to finish |
| `join!` | Runs multiple futures concurrently, waits for all |
| Stream | Async iterator, produces values over time |

### spawn vs join!

| | `tokio::spawn` | `tokio::join!` |
|---|---|---|
| Tasks | Run independently | Run on current task |
| Cancellation | Handle can be aborted | Cancelled when future is dropped |
| Errors | Panics are caught in JoinHandle | Panics propagate immediately |
| Use when | Fire-and-forget, parallelism | Wait for related sub-operations |

### Channel types at a glance

| Channel | Producers | Consumers | Use case |
|---------|-----------|-----------|----------|
| `mpsc` | Many | One | Task pipelines, work queues |
| `oneshot` | One | One | Request/response, task result |
| `broadcast` | One | Many | Event fan-out, pub/sub |
| `watch` | One | Many | Latest-value notification (config, state) |

---

## Common Patterns

### Pattern: concurrent fetch with error handling

```rust
use tokio;

#[tokio::main]
async fn main() {
    let urls = vec!["url1", "url2", "url3"];

    let results = futures::future::join_all(
        urls.iter().map(|url| fetch(url))
    ).await;

    for (url, result) in urls.iter().zip(results) {
        match result {
            Ok(body) => println!("{}: {} bytes", url, body.len()),
            Err(e) => eprintln!("{}: error — {}", url, e),
        }
    }
}

async fn fetch(_url: &str) -> Result<String, String> {
    Ok(String::from("body"))
}
```

### Pattern: retry with backoff

`f` needs to be called more than once, so it's bound by `FnMut`, and each call must
produce a fresh future to await, so `f`'s return type `Fut` is bounded separately as
`Future<Output = Result<T, E>>` rather than writing `f: impl Fn() -> Result<T, E>`
directly; an `async` closure only returns a value once its future is awaited, not when
it's called, so the function has to name and await that intermediate future itself.

```rust
use tokio::time::{sleep, Duration};

async fn retry<T, E, F, Fut>(mut f: F, max_attempts: u32) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut delay = Duration::from_millis(100);

    for attempt in 1..=max_attempts {
        match f().await {
            Ok(val) => return Ok(val),
            Err(e) if attempt == max_attempts => return Err(e),
            Err(_) => {
                sleep(delay).await;
                delay *= 2; // exponential backoff
            }
        }
    }
    unreachable!()
}

#[tokio::main]
async fn main() {
    let result = retry(|| async { flaky_call().await }, 3).await;
    println!("{:?}", result);
}

async fn flaky_call() -> Result<String, &'static str> {
    Err("not ready")
}
```

### Pattern: background task with graceful shutdown

```rust
use tokio::sync::oneshot;
use tokio::time::{sleep, Duration};

async fn background_worker(mut shutdown: oneshot::Receiver<()>) {
    loop {
        tokio::select! {
            _ = sleep(Duration::from_secs(1)) => {
                println!("Worker tick");
            }
            _ = &mut shutdown => {
                println!("Worker shutting down");
                break;
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let (shutdown_tx, shutdown_rx) = oneshot::channel();

    let worker = tokio::spawn(background_worker(shutdown_rx));

    sleep(Duration::from_secs(3)).await;

    shutdown_tx.send(()).unwrap(); // signal graceful shutdown
    worker.await.unwrap();
    println!("Done");
}
```

---

## Common Pitfalls

### Pitfall 1: blocking inside async code

Never call blocking functions (file I/O, `std::thread::sleep`, CPU-heavy loops) directly inside an async function. They block the thread and prevent all other tasks from running.

```rust
#[tokio::main]
async fn main() {
    // BAD: blocks the entire async runtime thread
    std::thread::sleep(std::time::Duration::from_secs(1));

    // GOOD: suspends only this task
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
}
```

For unavoidable blocking work (synchronous file I/O, CPU computation), offload to a dedicated thread:

```rust
#[tokio::main]
async fn main() {
    // Runs the blocking closure on a thread pool, returns a future
    let result = tokio::task::spawn_blocking(|| {
        expensive_cpu_work()
    }).await.unwrap();

    println!("{}", result);
}

fn expensive_cpu_work() -> u64 {
    (1u64..=10_000_000).sum()
}
```

### Pitfall 2: holding an async Mutex across await in the wrong way

Always release lock guards before awaiting long operations — holding the lock keeps other tasks waiting:

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(0u32));

    // BAD: lock held while awaiting, blocks all other tasks that need this lock
    {
        let mut guard = data.lock().await;
        long_io_operation().await;  // all other tasks waiting on this mutex are blocked
        *guard += 1;
    }

    // GOOD: get what you need, release the lock, then await
    {
        let value = *data.lock().await; // lock acquired and released quickly
        long_io_operation().await;      // lock not held here
        *data.lock().await += value;   // re-acquire only when needed
    }
}

async fn long_io_operation() {
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
}
```

### Pitfall 3: forgetting to await a future

Calling an async function without `.await` silently does nothing. Rust will warn you, but it is easy to miss:

```rust
async fn send_email() { /* ... */ }

async fn notify() {
    send_email(); // WARNING: unused future — email is never sent
    send_email().await; // Correct
}
```

### Pitfall 4: using std::sync::Mutex across an await point

Rust will reject this, but the error can be confusing:

```rust
use std::sync::Mutex;

async fn bad_example(m: &Mutex<u32>) {
    let guard = m.lock().unwrap();
    tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
    // ERROR: MutexGuard is not Send — cannot be held across an await in a
    // multi-threaded runtime because the task may resume on a different thread
    println!("{}", *guard);
}
```

Fix: drop the guard before awaiting, or switch to `tokio::sync::Mutex`.

### Pitfall 5: spawning tasks that reference local variables

`tokio::spawn` requires `'static` bounds — spawned tasks cannot borrow from the current stack frame:

```rust
async fn process(data: &[u32]) {
    // tokio::spawn(async {
    //     println!("{:?}", data); // ERROR: `data` does not live long enough
    // });

    // Fix: clone or use Arc
    let data = data.to_vec();
    tokio::spawn(async move {
        println!("{:?}", data);
    });
}
```

---

## Async vs Threads

Use this as a guide for choosing between async and OS threads:

| Question | Lean toward async | Lean toward threads |
|----------|-------------------|---------------------|
| Workload | I/O-bound (network, file, timers) | CPU-bound (computation, encoding) |
| Concurrency count | Thousands of concurrent operations | Dozens of parallel workers |
| Blocking code in use? | No (or offloaded via spawn_blocking) | Yes — threads tolerate it |
| Latency requirements | Very low per-task overhead | Predictable, independent |
| Ecosystem | Needs async ecosystem (reqwest, sqlx, axum) | Needs sync ecosystem |

In practice, many applications use both: an async runtime for I/O, with `spawn_blocking` or `rayon` for CPU work.

---

## Summary

See the Quick Reference above for the core concepts and channel types at a glance.

### Key rules

1. Never block inside async code — use `tokio::time::sleep`, not `std::thread::sleep`
2. Use `spawn_blocking` for CPU-heavy or unavoidably synchronous work
3. Prefer `tokio::sync::Mutex` when holding a lock across `.await`; use `std::sync::Mutex` for short, non-awaiting critical sections
4. Always `.await` your futures — an unawaited future does nothing
5. Spawned tasks must be `'static` — clone or use `Arc` instead of borrowing
