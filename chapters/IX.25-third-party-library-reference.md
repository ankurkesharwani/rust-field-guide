# Project Libraries Reference

## Table of Contents

1. [Introduction](#introduction)
2. [clap — CLI Argument Parsing](#clap--cli-argument-parsing)
   - [How the derive macros work](#how-the-derive-macros-work)
   - [Parser, Subcommand, Args](#parser-subcommand-args)
   - [The `#[arg(...)]` attribute](#the-arg-attribute)
   - [ValueEnum](#valueenum)
   - [Custom value parsers](#custom-value-parsers)
3. [serde + serde_yaml — Serialization](#serde--serde_yaml--serialization)
   - [How `#[derive(Serialize, Deserialize)]` works](#how-deriveserialize-deserialize-works)
   - [Field attributes](#field-attributes)
   - [Reading and writing YAML](#reading-and-writing-yaml)
4. [thiserror — Error Types](#thiserror--error-types)
   - [What `#[derive(Error)]` generates](#what-deriveerror-generates)
   - [The `#[error("...")]` attribute](#the-error-attribute)
   - [The `#[from]` attribute](#the-from-attribute)
   - [The `#[source]` attribute](#the-source-attribute)
   - [The `#[transparent]` attribute](#the-transparent-attribute)
   - [Full plain Rust expansion](#full-plain-rust-expansion)
5. [tabled — Terminal Tables](#tabled--terminal-tables)
   - [What `#[derive(Tabled)]` generates](#what-derivetabled-generates)
   - [Column attributes](#column-attributes)
   - [Table styles](#table-styles)
   - [Modifying specific cells](#modifying-specific-cells)
   - [Built-in colors and bold](#built-in-colors-and-bold)
   - [Targeting rows and segments](#targeting-rows-and-segments)
6. [owo-colors — Terminal Colors](#owo-colors--terminal-colors)
   - [Inline coloring](#inline-coloring)
   - [Reusable styles](#reusable-styles)
   - [How ANSI codes work](#how-ansi-codes-work)
7. [Quick Reference](#quick-reference)
8. [Common Patterns](#common-patterns)
   - [Bridging owo-colors and tabled](#bridging-owo-colors-and-tabled)
9. [Common Pitfalls](#common-pitfalls)
10. [Summary](#summary)

---

## Introduction

This chapter covers every third-party crate used in this project. For each one it
explains what the derive macros and attributes actually generate, shows the plain Rust
equivalent, and documents the patterns this project uses them with.

---

## clap — CLI Argument Parsing

`clap` turns Rust structs and enums into a fully featured CLI — including `--help`, `--version`, argument validation, subcommands, and error messages — entirely from derive macros.

### How the derive macros work

`clap` uses four derive macros:

| Macro | Applied to | Purpose |
|---|---|---|
| `#[derive(Parser)]` | Top-level struct | The entry point. Calls `Cli::parse()` to read `std::env::args()` |
| `#[derive(Subcommand)]` | Enum | Each variant becomes a subcommand |
| `#[derive(Args)]` | Struct | A group of arguments attached to a command |
| `#[derive(ValueEnum)]` | Enum | Each variant becomes a valid string value for an `--arg` |

Each macro generates an impl of the corresponding `clap` trait. At runtime, `Cli::parse()` reads `argv`, matches it against the generated structure, and either populates your structs or exits with a help/error message.

### Parser, Subcommand, Args

From `cli.rs`:

```rust
#[derive(Parser)]
#[command(name = "tcodelab")]
#[command(about = "An offline coding lab in your terminal", long_about = None)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,
}
```

`#[derive(Parser)]` on `Cli` makes `Cli::parse()` available. The `#[command(...)]` attributes set the binary name and the `--help` description.

`#[command(subcommand)]` on a field tells clap that this field should be parsed as a subcommand — it must be a type that derives `Subcommand`.

```rust
#[derive(Subcommand)]
pub enum Commands {
    /// List topics
    Topics(TopicArgs),
    /// Manage problems
    Problem {
        #[command(subcommand)]
        subcommand: ProblemCommands,
    },
}
```

Each enum variant becomes a CLI subcommand. The variant name is converted to `kebab-case` automatically (`Topics` → `topics`). Doc comments (`///`) become the help text shown in `--help`.

A variant can hold:
- A tuple struct of `Args` — the args are attached directly to that subcommand
- Named fields with `#[command(subcommand)]` — for nested subcommands
- No fields — a bare subcommand with no arguments

```rust
#[derive(Args)]
pub struct TopicArgs {
    #[arg(long)]
    pub filter: Option<String>,

    #[arg(long, value_enum, default_value_t = TopicSortBy::Name)]
    pub sort_by: TopicSortBy,
}
```

`#[derive(Args)]` on a struct makes it embeddable inside a `Subcommand` variant or composable with `#[command(flatten)]`.

### The `#[arg(...)]` attribute

`#[arg(...)]` configures how a single field is parsed. Common options:

| Option | Effect |
|---|---|
| `long` | Makes it a `--flag` (derived from field name) |
| `short` | Makes it a `-f` flag |
| `long = "custom"` | Overrides the flag name |
| `default_value_t = expr` | Default value (uses `Display`) |
| `default_value = "str"` | Default as a string literal |
| `value_enum` | Parses via a `ValueEnum` type |
| `value_parser = fn` | Custom parsing function |
| `num_args = 1..` | Accept multiple values |
| `value_delimiter = ','` | Split on delimiter |

Fields typed `Option<T>` are automatically optional. Fields typed `bool` with `#[arg(long)]` become flags that default to `false`.

Example from the project:

```rust
/// Filter by one or more tags, comma-separated (e.g. --tag DFS,"2D-Array",BFS)
#[arg(long, num_args = 1.., value_delimiter = ',')]
pub tag: Vec<String>,
```

This accepts `--tag DFS BFS` (space-separated) or `--tag DFS,BFS` (comma-separated) and collects them into a `Vec<String>`.

### ValueEnum

`#[derive(ValueEnum)]` turns an enum into a set of accepted string values for an argument. Variant names are lowercased and hyphenated automatically.

```rust
#[derive(ValueEnum, Clone)]
pub enum Difficulty {
    Easy,    // accepted as "easy"
    Medium,  // accepted as "medium"
    Hard,    // accepted as "hard"
}
```

Usage in an `Args` struct:

```rust
#[arg(long, value_enum)]
pub difficulty: Option<Difficulty>,
```

clap validates the input and shows the accepted values in `--help`. Passing an unknown string exits with an error message.

`#[default]` marks the default variant when used with `default_value_t`:

```rust
#[derive(ValueEnum, Clone, Default)]
pub enum SortOrder {
    #[default]
    Asc,
    Desc,
}
```

### Custom value parsers

`value_parser = fn` points to a function with signature `fn(&str) -> Result<T, String>`. clap calls it to parse and validate the raw string before populating the field.

```rust
fn parse_track_name(s: &str) -> Result<String, String> {
    let valid = !s.is_empty()
        && s.split('-').all(|part| {
            !part.is_empty() && part.chars().all(|c| c.is_ascii_alphanumeric())
        });
    if valid {
        Ok(s.to_string())
    } else {
        Err(format!("'{}' is not a valid track name ...", s))
    }
}

#[arg(value_parser = parse_track_name)]
pub name: String,
```

If the function returns `Err`, clap prints the error message and exits. The `Ok` value is what gets stored in the struct.

---

## serde + serde_yaml — Serialization

`serde` is a framework for serializing and deserializing Rust data structures. It defines traits (`Serialize`, `Deserialize`) and a data model. Format-specific crates like `serde_yaml` implement those traits for a specific format.

### How `#[derive(Serialize, Deserialize)]` works

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct WorkspaceConfig {
    pub default_track: String,
}
```

`#[derive(Serialize)]` generates:
```rust
impl serde::Serialize for WorkspaceConfig {
    fn serialize<S: serde::Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error> {
        let mut map = serializer.serialize_struct("WorkspaceConfig", 1)?;
        map.serialize_field("default_track", &self.default_track)?;
        map.end()
    }
}
```

`#[derive(Deserialize)]` generates the inverse — it reads fields from whatever the deserializer provides and constructs the struct.

The key insight: `Serialize`/`Deserialize` are format-agnostic. The same derived impl works with JSON, YAML, TOML, MessagePack, or any other serde-compatible format. The format crate (e.g. `serde_yaml`) provides the `Serializer`/`Deserializer`.

### Field attributes

| Attribute | Effect |
|---|---|
| `#[serde(rename = "key")]` | Use a different key name in the serialized output |
| `#[serde(skip)]` | Exclude field entirely |
| `#[serde(skip_serializing_if = "Option::is_none")]` | Omit field if `None` |
| `#[serde(default)]` | Use `Default::default()` if key is missing during deserialization |
| `#[serde(default = "fn_name")]` | Call a function for the default |
| `#[serde(alias = "old_name")]` | Accept multiple key names during deserialization |
| `#[serde(flatten)]` | Inline a nested struct's fields into the parent |

Example:
```rust
#[derive(Deserialize)]
struct Config {
    #[serde(rename = "defaultTrack")]
    default_track: String,

    #[serde(default)]
    verbose: bool,
}
```

This deserializes from:
```yaml
defaultTrack: main
```

### Reading and writing YAML

`serde_yaml` provides two main functions:

```rust
// Deserialize: YAML string → Rust value
let config: WorkspaceConfig = serde_yaml::from_str(&yaml_string)?;

// Serialize: Rust value → YAML string
let yaml_string = serde_yaml::to_string(&config)?;
```

Both return `Result<_, serde_yaml::Error>`. The error implements `std::error::Error` and can be converted to a `ConfigError` via `#[from]` (as seen in `config.rs`).

In `config.rs`:
```rust
let serialised_config = serde_yaml::to_string(&root_config)?;   // write
let workspace: WorkspaceConfig = serde_yaml::from_str(&contents)?; // read
```

The `?` operator works here because `ConfigError` has `#[from] serde_yaml::Error`.

---

## thiserror — Error Types

To use a type as an error (e.g. with `?`, `Box<dyn Error>`, `impl Error`), it must implement:
- `std::fmt::Display` — how the error is printed
- `std::error::Error` — the marker trait

Writing these by hand for every error enum is tedious. `thiserror` generates them from attributes.

### What `#[derive(Error)]` generates

`#[derive(Error)]` generates the `impl std::error::Error` block. The trait has no required methods, so the generated impl is usually just:

```rust
impl std::error::Error for MyError {}
```

But if `#[source]` or `#[from]` fields are present, it also generates the `.source()` method (see below).

### The `#[error("...")]` attribute

Controls the `Display` output per variant. The string is a format template.

```rust
#[derive(Debug, Error)]
pub enum IoError {
    #[error("Path '{path}' does not exist")]
    NotFound { path: PathBuf },

    #[error("IO error: {0}")]
    Other(#[from] std::io::Error),
}
```

- `{path}` — interpolates the named field `path`
- `{0}` — interpolates the first tuple field
- Bare strings — used for unit variants

Plain Rust equivalent:
```rust
impl std::fmt::Display for IoError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            IoError::NotFound { path } => write!(f, "Path '{}' does not exist", path.display()),
            IoError::Other(e)          => write!(f, "IO error: {}", e),
        }
    }
}
```

### The `#[from]` attribute

Generates a `From<OtherError> for MyError` impl. This is what makes `?` work across different error types — the `?` operator calls `From::from` under the hood.

```rust
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("{0}")]
    EnvError(#[from] EnvError),

    #[error("{0}")]
    IoError(#[from] IoError),

    #[error("Failed to serialize/deserialize config")]
    SerializationError(#[from] serde_yaml::Error),
}
```

Now in any function returning `Result<_, ConfigError>`, you can `?` on `EnvError`, `IoError`, or `serde_yaml::Error` and they'll be automatically converted.

Plain Rust equivalent:
```rust
impl From<EnvError> for ConfigError {
    fn from(e: EnvError) -> Self { ConfigError::EnvError(e) }
}
impl From<IoError> for ConfigError {
    fn from(e: IoError) -> Self { ConfigError::IoError(e) }
}
impl From<serde_yaml::Error> for ConfigError {
    fn from(e: serde_yaml::Error) -> Self { ConfigError::SerializationError(e) }
}
```

`#[from]` also implicitly marks the field as `#[source]`.

### The `#[source]` attribute

Controls the `.source()` method on the `Error` trait. This enables error chains — callers can walk the chain to find the root cause.

```rust
#[derive(Debug, Error)]
pub enum IoError {
    #[error("Failed to create directory '{path}': {source}")]
    DirectoryCreateFailed {
        path: PathBuf,
        #[source]
        source: std::io::Error,
    },
}
```

Plain Rust equivalent:
```rust
impl std::error::Error for IoError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            IoError::DirectoryCreateFailed { source, .. } => Some(source),
            _ => None,
        }
    }
}
```

Without `#[source]`/`#[from]`, the generated `.source()` always returns `None`.

### The `#[transparent]` attribute

Delegates both `Display` and `source()` to the inner error. Used when wrapping without adding any message.

```rust
#[derive(Debug, Error)]
enum AppError {
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

Plain Rust equivalent:
```rust
impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self { AppError::Other(e) => e.fmt(f) }
    }
}
impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self { AppError::Other(e) => e.source() }
    }
}
```

### Full plain Rust expansion

From `errors.rs`, the `IoError` with `thiserror`:
```rust
#[derive(Debug, Error)]
pub enum IoError {
    #[error("Path '{path}' does not exist")]
    NotFound { path: PathBuf },

    #[error("Failed to create directory '{path}': {source}")]
    DirectoryCreateFailed { path: PathBuf, #[source] source: std::io::Error },

    #[error("IO error: {0}")]
    Other(#[from] std::io::Error),
}
```

Full plain Rust equivalent — everything `thiserror` generates:
```rust
impl std::fmt::Display for IoError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            IoError::NotFound { path } =>
                write!(f, "Path '{}' does not exist", path.display()),
            IoError::DirectoryCreateFailed { path, source } =>
                write!(f, "Failed to create directory '{}': {}", path.display(), source),
            IoError::Other(e) =>
                write!(f, "IO error: {}", e),
        }
    }
}

impl std::error::Error for IoError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            IoError::DirectoryCreateFailed { source, .. } => Some(source),
            IoError::Other(e) => Some(e),  // #[from] implies #[source]
            _ => None,
        }
    }
}

impl From<std::io::Error> for IoError {
    fn from(e: std::io::Error) -> Self { IoError::Other(e) }
}
```

`thiserror` compresses ~30 lines of mechanical boilerplate into ~8 lines of annotated definition.

---

## tabled — Terminal Tables

`tabled` renders a collection of structs as a formatted Unicode table.

### What `#[derive(Tabled)]` generates

```rust
#[derive(Tabled)]
struct TopicRow {
    #[tabled(rename = "Code")]
    code: String,
    #[tabled(rename = "Name")]
    name: String,
    #[tabled(rename = "Problems")]
    problems: usize,
}
```

`#[derive(Tabled)]` generates an impl of the `Tabled` trait, which has two methods:
- `fields()` — returns the row's cell values as `Vec<String>`
- `headers()` — returns the column header names as `Vec<String>`

Plain Rust equivalent:
```rust
impl tabled::Tabled for TopicRow {
    fn fields(&self) -> Vec<std::borrow::Cow<'_, str>> {
        vec![
            self.code.to_string().into(),
            self.name.to_string().into(),
            self.problems.to_string().into(),
        ]
    }

    fn headers() -> Vec<std::borrow::Cow<'static, str>> {
        vec!["Code".into(), "Name".into(), "Problems".into()]
    }
}
```

`Table::new(&rows)` calls `headers()` once for the header row and `fields()` on each item for data rows, then lays them out into a grid.

### Column attributes

| Attribute | Effect |
|---|---|
| `#[tabled(rename = "Label")]` | Override the column header |
| `#[tabled(skip)]` | Exclude this field from the table |
| `#[tabled(display_with = "fn_name")]` | Use a custom function to format the value |
| `#[tabled(order = N)]` | Set the column's display position |

### Table styles

`Style` controls the border characters. Apply with `.with(Style::...)`:

```rust
Table::new(&rows).with(Style::sharp())
```

| Style | Description |
|---|---|
| `Style::sharp()` | Unicode box-drawing with sharp corners (used in this project) |
| `Style::modern()` | Unicode with rounded corners |
| `Style::ascii()` | Pure ASCII `+`, `-`, `\|` |
| `Style::blank()` | Aligned columns, no borders |
| `Style::psql()` | PostgreSQL `\df`-style |
| `Style::markdown()` | GitHub Markdown table |

`Style::sharp()` output:
```
┌───────┬─────────────────────┬──────────┐
│ Code  │ Name                │ Problems │
├───────┼─────────────────────┼──────────┤
│ dp    │ Dynamic Programming │ 12       │
│ graph │ Graphs              │ 8        │
└───────┴─────────────────────┴──────────┘
```

### Modifying specific cells

`Modify::new(<selector>).with(<setting>)` applies a setting to a subset of cells. Settings include `Color`, `Alignment`, `Width`, `Padding`, and more.

From `topic.rs`:
```rust
Table::new(&self.0)
    .with(Style::sharp())
    .with(Modify::new(Rows::first()).with(owo_to_tabled_color(&bold)))
    .with(Modify::new(Segment::new(1.., 0..1)).with(owo_to_tabled_color(&bold_yellow)))
    .to_string()
```

- First `.with(Modify...)` — makes the header row bold
- Second `.with(Modify...)` — makes the first column of all data rows bold yellow

### Built-in colors and bold

`tabled`'s `Color` type has built-in constants for common styles — no external crate needed:

```rust
use tabled::settings::Color;

Color::BOLD         // bold text
Color::UNDERLINE    // underlined text
Color::FG_YELLOW    // yellow foreground
Color::FG_CYAN      // cyan foreground
Color::BG_BLUE      // blue background
// Full list: FG_BLACK/RED/GREEN/YELLOW/BLUE/MAGENTA/CYAN/WHITE (and BRIGHT_ variants)
//            BG_BLACK/RED/GREEN/YELLOW/BLUE/MAGENTA/CYAN/WHITE (and BRIGHT_ variants)
```

Combine multiple styles with `|`:

```rust
Table::new(&rows)
    .with(Style::sharp())
    .with(Modify::new(Rows::first()).with(Color::BOLD | Color::FG_YELLOW))
    .with(Modify::new(Segment::new(1.., 0..1)).with(Color::FG_CYAN))
    .to_string()
```

For 24-bit RGB color:

```rust
Color::rgb_fg(255, 165, 0)   // orange foreground
Color::rgb_bg(30, 30, 30)    // dark grey background
```

The `|` operator merges the ANSI prefix and suffix strings of both colors, so `Color::BOLD | Color::FG_YELLOW` applies bold and yellow in a single cell pass.

### Targeting rows and segments

| Selector | Targets |
|---|---|
| `Rows::first()` | Header row (row 0) |
| `Rows::last()` | Last data row |
| `Rows::new(1..)` | All data rows (skips header) |
| `Columns::first()` | First column |
| `Columns::new(0..2)` | Columns 0 and 1 |
| `Segment::new(rows, cols)` | Intersection of a row range and column range |
| `Cell::new(r, c)` | A single cell at row `r`, column `c` |

`Segment::new(1.., 0..1)` means: rows 1 and beyond (all data rows), column 0 only. Both arguments are standard Rust range expressions.

---

## owo-colors — Terminal Colors

`owo-colors` wraps values in ANSI escape codes to produce colored terminal output.

### Inline coloring

```rust
use owo_colors::OwoColorize;

println!("{}", "hello".red());
println!("{}", "world".bold().green());
println!("{}", 42.yellow());
```

`OwoColorize` is a blanket trait implemented for every `T: Display`. Calling `.red()` wraps `self` in a newtype that adds ANSI codes when formatted. The original value is unchanged.

Common methods: `.red()`, `.green()`, `.blue()`, `.yellow()`, `.cyan()`, `.magenta()`, `.white()`, `.black()`, `.bold()`, `.italic()`, `.underline()`, `.dimmed()`, `.on_red()` (background), `.on_blue()`, etc.

Methods can be chained: `.bold().yellow()` applies both styles.

### Reusable styles

`Style` (aliased in the project as `OwoStyle`) lets you define a set of styles once and apply them to multiple values:

```rust
use owo_colors::Style as OwoStyle;

let bold = OwoStyle::new().bold();
let bold_yellow = OwoStyle::new().bold().yellow();

// Apply with .style(value) — returns a Display wrapper
println!("{}", bold.style("Section Title"));
println!("{}", bold_yellow.style("Warning"));
```

This is how `topic.rs` defines its table colors — once at the top of `render()`, then passed to `owo_to_tabled_color`.

### How ANSI codes work

`owo-colors` generates standard ANSI escape sequences. Every colored string is:

```
\x1b[1m  Hello  \x1b[0m
  ^                ^
"start bold"     "reset all"
```

`\x1b` (ESC) is the escape character. The terminal intercepts these sequences and changes rendering — they consume no visible space.

`Style` exposes two lower-level methods that write these sequences directly to a `Formatter`:
- `fmt_prefix(f)` — writes the opening sequence (e.g. `\x1b[1;33m` for bold yellow)
- `fmt_suffix(f)` — writes the reset sequence (`\x1b[0m`)

These are used in the `owo_to_tabled_color` bridge (see next section).

---

## Quick Reference

| Crate | Purpose | Key derive/attribute |
|---|---|---|
| `clap` | CLI argument parsing | `#[derive(Parser)]`, `#[derive(Subcommand)]`, `#[derive(Args)]`, `#[derive(ValueEnum)]`, `#[arg(...)]` |
| `serde` + `serde_yaml` | Serialization | `#[derive(Serialize, Deserialize)]`, `#[serde(...)]`, `serde_yaml::from_str`/`to_string` |
| `thiserror` | Error types | `#[derive(Error)]`, `#[error("...")]`, `#[from]`, `#[source]`, `#[error(transparent)]` |
| `tabled` | Terminal tables | `#[derive(Tabled)]`, `#[tabled(...)]`, `Style::...()`, `Modify::new(...)` |
| `owo-colors` | Terminal colors | `OwoColorize` (`.red()`, `.bold()`, ...), `Style::new()` |

---

## Common Patterns

### Bridging owo-colors and tabled

`tabled`'s `Color::new(prefix, suffix)` takes raw ANSI strings. `owo-colors`'s `Style` can produce those strings but only through a `Formatter` — there is no `.to_string()` method on the sequences directly.

The bridge in `topic.rs` solves this with local newtype wrappers that implement `Display` using `fmt_prefix`/`fmt_suffix`, allowing `.to_string()` to extract the raw strings:

```rust
fn owo_to_tabled_color(style: &OwoStyle) -> Color {
    struct Prefix<'a>(&'a OwoStyle);
    impl fmt::Display for Prefix<'_> {
        fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
            self.0.fmt_prefix(f)   // writes e.g. "\x1b[1;33m"
        }
    }

    struct Suffix<'a>(&'a OwoStyle);
    impl fmt::Display for Suffix<'_> {
        fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
            self.0.fmt_suffix(f)   // writes "\x1b[0m"
        }
    }

    // .to_string() triggers Display, which triggers fmt_prefix / fmt_suffix
    Color::new(Prefix(style).to_string(), Suffix(style).to_string())
}
```

The two local structs exist purely to give `.to_string()` something to call `fmt` on. This is a common Rust idiom when you need to extract a `String` from something that only writes to a `Formatter`. The structs are private to the function — they have no other purpose.

---

## Common Pitfalls

- A `clap` field typed `Vec<T>` without `num_args`/`value_delimiter` only accepts one
  value per `--flag` occurrence (`--tag a --tag b`); it won't split a single
  comma-separated argument unless `value_delimiter` is set. See The `#[arg(...)]`
  attribute above.
- `#[serde(default)]` only fills in a missing key; it does nothing for a key that's
  present but has the wrong type or shape, which still fails deserialization with an
  error. See Field attributes above.
- Adding `#[from]` to a new error variant only helps call sites that already return that
  error type via `?`; it doesn't retroactively change unrelated `match` arms, and two
  variants can't both have `#[from]` pointing at the same source type. See The `#[from]`
  attribute above.
- `Modify::new(...)` settings apply in the order they're chained; a later `.with(...)`
  targeting the same cells overrides an earlier one instead of merging with it, so order
  matters when combining color, alignment, and width settings. See Modifying specific
  cells above.
- `owo-colors` methods like `.red()` emit ANSI escape codes unconditionally, even when
  the output is piped to a file or another program rather than a terminal. Detecting
  whether color is actually appropriate (a real terminal, `NO_COLOR` unset) is a
  separate opt-in step (the crate's `if_supports_color` helper), not something `.red()`
  does on its own.

---

## Summary

This chapter walked through the derive-heavy crates this project depends on: `clap` for
argument parsing, `serde`/`serde_yaml` for reading and writing config as YAML,
`thiserror` for defining error types with `Display`, `From`, and `.source()` generated
from attributes, and `tabled`/`owo-colors` for rendering colored terminal tables. Each
section paired the derive-generated behavior with its plain Rust equivalent so the
"magic" stays inspectable.
