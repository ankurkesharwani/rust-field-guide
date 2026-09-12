# Handling User Input in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Reading from Standard Input](#reading-from-standard-input)
3. [Parsing Input](#parsing-input)
4. [Validating Input](#validating-input)
5. [Command-Line Arguments](#command-line-arguments)
6. [Building CLIs with Clap](#building-clis-with-clap)
7. [Custom Value Parsers](#custom-value-parsers)
8. [Environment Variables](#environment-variables)
9. [Quick Reference](#quick-reference)
10. [Common Patterns](#common-patterns)
11. [Common Pitfalls](#common-pitfalls)
12. [Summary](#summary)

---

## Introduction

Rust provides several ways to read user input: standard input (`stdin`) for interactive programs, command-line arguments for CLI tools, and environment variables for configuration. All of these follow Rust's core principle of making errors explicit — every operation that can fail returns a `Result`.

The key APIs:

| Source | Primary Types |
|---|---|
| Standard input | `std::io::stdin`, `std::io::BufRead` |
| CLI arguments | `std::env::args`, or the `clap` crate |
| Environment variables | `std::env::var` |

---

## Reading from Standard Input

The simplest way to read a line from the user is with `stdin().read_line()`.

```rust
use std::io;

fn main() {
    let mut input = String::new();
    println!("Enter your name:");
    io::stdin().read_line(&mut input).expect("Failed to read line");

    // read_line includes the trailing newline — trim it
    let name = input.trim();
    println!("Hello, {name}!");
}
```

**Important:** `read_line` *appends* to the string rather than replacing it. Always start with an empty `String` or call `input.clear()` before reusing.

### Reading multiple lines

```rust
use std::io::{self, BufRead};

fn main() {
    let stdin = io::stdin();
    for line in stdin.lock().lines() {
        let line = line.expect("Failed to read line");
        println!("Got: {line}");
    }
}
```

`stdin().lock()` returns a `StdinLock` which implements `BufRead`, giving access to the `.lines()` iterator. This is more efficient than calling `read_line` in a loop.

---

## Parsing Input

Raw input arrives as a `&str`. Use `.parse()` to convert it to any type that implements `FromStr`.

```rust
use std::io;

fn main() {
    let mut input = String::new();
    println!("Enter a number:");
    io::stdin().read_line(&mut input).unwrap();

    let n: i32 = input.trim().parse().expect("Please enter a valid integer");
    println!("Double: {}", n * 2);
}
```

Handle parse errors gracefully with `match` instead of `expect`:

```rust
let n: Result<i32, _> = input.trim().parse();
match n {
    Ok(value) => println!("Got: {value}"),
    Err(_) => println!("That wasn't a number."),
}
```

---

## Validating Input

### Using a validation function

Extract validation into a function that returns `Result<T, String>`. This pattern is reusable and composable.

```rust
fn parse_positive(s: &str) -> Result<u32, String> {
    let n: u32 = s.trim().parse().map_err(|_| format!("'{s}' is not a positive integer"))?;
    if n == 0 {
        return Err("Value must be greater than zero".to_string());
    }
    Ok(n)
}
```

### Looping until valid input

```rust
use std::io;

fn read_positive() -> u32 {
    loop {
        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();
        match parse_positive(&input) {
            Ok(n) => return n,
            Err(e) => println!("Invalid input: {e}. Try again:"),
        }
    }
}
```

---

## Command-Line Arguments

The standard library provides `std::env::args()`, an iterator over the program's arguments as `String`s.

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    // args[0] is always the program name
    if args.len() < 2 {
        eprintln!("Usage: {} <name>", args[0]);
        std::process::exit(1);
    }

    println!("Hello, {}!", args[1]);
}
```

`std::env::args()` panics if any argument contains invalid UTF-8. Use `std::env::args_os()` for an `OsString` iterator if you need to handle arbitrary bytes.

---

## Building CLIs with Clap

For anything beyond trivial argument parsing, [`clap`](https://docs.rs/clap) is the standard choice. Its derive API lets you define the entire CLI as a struct.

Add to `Cargo.toml`:

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

### Defining a CLI

`#[derive(Parser)]` generates an `impl` that reads `std::env::args()`, matches each
argument against the struct's fields, and either fills in `Cli` or prints a
usage/error message and exits, so `Cli::parse()` below is doing the same job as the
manual `env::args()` loop in the previous section, just generated for you from the
struct's shape. Each field name becomes a `--flag-name` (or a positional argument, for
fields without an `#[arg]` attribute changing that), each field's type determines how
the raw string is parsed and validated, and each doc comment (`///`) becomes that
argument's `--help` text. `#[derive(Subcommand)]` does the same for an enum, turning
each variant into a subcommand name.

```rust
use clap::{Parser, Subcommand, Args, ValueEnum};

#[derive(Parser)]
#[command(name = "myapp")]
#[command(about = "A sample CLI application")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Greet a user
    Greet(GreetArgs),
    /// Show version info
    Version,
}

#[derive(Args)]
struct GreetArgs {
    /// Name to greet
    pub name: String,

    /// Language to use
    #[arg(long, value_enum, default_value_t = Lang::English)]
    pub lang: Lang,
}

#[derive(ValueEnum, Clone)]
enum Lang {
    English,
    Spanish,
}

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Commands::Greet(args) => match args.lang {
            Lang::English => println!("Hello, {}!", args.name),
            Lang::Spanish => println!("Hola, {}!", args.name),
        },
        Commands::Version => println!("v1.0.0"),
    }
}
```

Clap automatically generates `--help` output, validates required arguments, and surfaces errors with descriptive messages.

### Optional arguments and flags

```rust
#[derive(Args)]
struct SearchArgs {
    /// Search query
    pub query: String,

    /// Limit results
    #[arg(long)]
    pub limit: Option<usize>,

    /// Case-sensitive search
    #[arg(long)]
    pub case_sensitive: bool,
}
```

- `Option<T>` fields are optional; absent means `None`.
- `bool` fields become flags (present = `true`, absent = `false`).

---

## Custom Value Parsers

Clap lets you attach a custom parser to any argument with `value_parser`. This is the right place to enforce domain-level constraints like format or allowed character sets.

```rust
fn parse_track_name(s: &str) -> Result<String, String> {
    let valid = !s.is_empty()
        && s.split('-')
            .all(|part| !part.is_empty() && part.chars().all(|c| c.is_ascii_alphanumeric()));
    if valid {
        Ok(s.to_string())
    } else {
        Err(format!(
            "'{s}' is not a valid track name — expected alphanumeric segments separated by hyphens (e.g. main, my-track)"
        ))
    }
}

#[derive(Args)]
struct CreateArgs {
    /// Name of the new track
    #[arg(value_parser = parse_track_name)]
    pub name: String,
}
```

When the user provides an invalid value, Clap prints the error message returned from the parser and exits — no extra error handling needed in your command logic.

### Parsing comma-separated lists

```rust
#[derive(Args)]
struct FilterArgs {
    /// Tags to filter by, comma-separated (e.g. --tag DFS,BFS)
    #[arg(long, num_args = 1.., value_delimiter = ',')]
    pub tag: Vec<String>,
}
```

`num_args = 1..` means at least one value is required; `value_delimiter = ','` splits a single string on commas.

---

## Environment Variables

Use `std::env::var` to read environment variables. It returns `Result<String, VarError>`.

```rust
use std::env;

fn main() {
    let api_key = env::var("API_KEY").unwrap_or_else(|_| "default-key".to_string());
    println!("Using key: {api_key}");
}
```

`VarError` has two variants:

| Variant | Meaning |
|---|---|
| `VarError::NotPresent` | Variable is not set |
| `VarError::NotUnicode` | Variable is set but contains invalid UTF-8 |

For structured configuration from environment variables, consider the [`envy`](https://docs.rs/envy) crate, which deserializes environment variables directly into a struct via `serde`.

---

## Quick Reference

### Key APIs

| Source | Primary types |
|---|---|
| Standard input | `std::io::stdin`, `std::io::BufRead` |
| CLI arguments | `std::env::args`, or the `clap` crate |
| Environment variables | `std::env::var` |

### VarError variants

| Variant | Meaning |
|---|---|
| `VarError::NotPresent` | Variable is not set |
| `VarError::NotUnicode` | Variable is set but contains invalid UTF-8 |

---

## Common Patterns

### Prompt and read in one helper

```rust
use std::io::{self, Write};

fn prompt(message: &str) -> String {
    print!("{message} ");
    io::stdout().flush().unwrap();   // ensure prompt appears before blocking on read
    let mut input = String::new();
    io::stdin().read_line(&mut input).unwrap();
    input.trim().to_string()
}
```

`stdout` is line-buffered by default, so `flush()` is necessary when printing without a trailing newline.

### Reading a yes/no confirmation

```rust
fn confirm(prompt: &str) -> bool {
    loop {
        let answer = self::prompt(&format!("{prompt} [y/n]:"));
        match answer.to_lowercase().as_str() {
            "y" | "yes" => return true,
            "n" | "no" => return false,
            _ => println!("Please answer y or n."),
        }
    }
}
```

### Combining CLI flags with environment variable fallbacks

A common pattern for tools: flags take precedence over environment variables, which fall back to a default.

```rust
fn resolve_token(flag_value: Option<String>) -> String {
    flag_value
        .or_else(|| std::env::var("MY_TOKEN").ok())
        .unwrap_or_else(|| "anonymous".to_string())
}
```

---

## Common Pitfalls

### read_line appends instead of replacing

`read_line` appends to the string you pass it rather than overwriting it. Reusing a
buffer across loop iterations without calling `.clear()` first accumulates every
previous line's input.

### env::args() panics on invalid UTF-8

`std::env::args()` panics if any command-line argument contains invalid UTF-8. Use
`std::env::args_os()`, which yields `OsString`, when you need to handle arbitrary bytes
without risking a panic.

### Forgetting to flush stdout before reading input

`stdout` is line-buffered by default, so a `print!` without a trailing newline may not
appear on screen before the program blocks on `stdin().read_line()`. Flush explicitly
after prompting, as the "Prompt and read in one helper" pattern above does.

---

## Summary

Rust reads user input through three channels: standard input via `std::io::stdin`,
command-line arguments via `std::env::args` (or `clap` for anything beyond the
trivial), and environment variables via `std::env::var`. All three surface failure
explicitly, either as a `Result` to handle or, in clap's case, as a generated `--help`
and validation error message. The Common Patterns above cover prompting for input,
confirming yes/no answers, and layering CLI flags over environment-variable fallbacks.
