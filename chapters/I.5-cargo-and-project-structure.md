# Cargo & Project Structure

## Table of Contents

1. [Introduction](#introduction)
2. [Cargo.toml Anatomy](#cargotoml-anatomy)
3. [Dependencies](#dependencies)
4. [Features](#features)
5. [Profiles](#profiles)
6. [Workspaces](#workspaces)
7. [Cargo Commands](#cargo-commands)
8. [Quick Reference](#quick-reference)
9. [Common Patterns](#common-patterns)
10. [Common Pitfalls](#common-pitfalls)
11. [Summary](#summary)

---

## Introduction

Cargo is Rust's build tool and package manager in one binary. It resolves dependencies,
compiles your crate and everything it depends on, runs tests, and can publish a crate to
a registry. `Cargo.toml` describes the package; `Cargo.lock` pins the exact dependency
versions that were resolved so a build is reproducible across machines. This chapter
covers the shape of `Cargo.toml`, how dependencies and features work, build profiles,
multi-crate workspaces, and the day-to-day `cargo` subcommands.

---

## Cargo.toml Anatomy

### The `[package]` table

```toml
[package]
name = "terminal-codelab"
version = "0.1.0"
edition = "2024"
```

`name` is the crate's identity on crates.io and in other crates' dependency lists.
`version` follows semantic versioning (`major.minor.patch`) and is what dependency
version requirements match against, covered below. `edition` selects a language edition
("2015", "2018", "2021", "2024"). Editions let Rust make small breaking changes to
syntax and defaults (like which keywords are reserved, or how closures capture fields)
without breaking every crate already on crates.io: each crate opts into an edition, and
the compiler mixes editions freely across a dependency graph, so upgrading one crate's
edition never forces its dependents to upgrade theirs.

### Targets: `[[bin]]`, `[lib]`, `[[example]]`

A crate builds one or more compilation targets. Cargo infers most of them from the
`src/` layout, but they can also be declared explicitly:

```toml
[[bin]]
name = "tcodelab"
path = "src/main.rs"

[lib]
name = "terminal_codelab"
path = "src/lib.rs"

[[example]]
name = "demo"
path = "examples/demo.rs"
```

Without any of this, Cargo uses convention: `src/main.rs` becomes a binary named after
the package, `src/lib.rs` becomes the library target, files under `examples/` become
example binaries, and files under `tests/` become integration test binaries (see
[Chapter 26: Testing](IX.26-testing.md#integration-tests)). The explicit `[[bin]]` table
above is only needed when the binary name should differ from the package name, which is
why this project has one: the package is `terminal-codelab` but the compiled binary is
`tcodelab`.

---

## Dependencies

### Version requirements

```toml
[dependencies]
thiserror = "2"
serde = { version = "1", features = ["derive"] }
tabled = "0.17"
```

A bare version string like `"2"` is shorthand for a caret requirement, `^2`, meaning
"any version Cargo considers compatible with 2, starting from the first `2.x.y` it can
find." Caret requirements follow semver's compatibility rule: for a version `0.0.z`,
only that exact patch matches; for `0.y.z` with `y > 0`, any `0.y.*` matches; for
`x.y.z` with `x > 0`, any `x.*` matches. In practice this means `"1"` accepts `1.9.4`
but not `2.0.0`, and `"0.9"` accepts `0.9.3` but not `0.10.0`, because a `0.x` bump in
the minor version is allowed to break things. This is why `serde_yaml = "0.9"` in this
project's `Cargo.toml` pins tighter than `thiserror = "2"` does: pre-1.0 crates treat
their minor version the way post-1.0 crates treat their major version.

The `{ version = "...", features = [...] }` table form is needed as soon as a dependency
needs more than a bare version, most commonly to turn on optional functionality (see
Features below).

### Path and git dependencies

```toml
[dependencies]
my_local_crate = { path = "../my_local_crate" }
some_crate = { git = "https://github.com/example/some_crate", branch = "main" }
```

A `path` dependency points at another crate on disk instead of a registry, which is how
workspace members and in-progress local crates get pulled in. A `git` dependency builds
straight from a repository, pinned by `branch`, `tag`, or `rev` (a specific commit).
Neither of these publishes to crates.io as a dependency spec: a crate that depends on
another crate via `path` or `git` cannot itself be published to crates.io unless that
dependency also has a registry `version` for the published build to fall back on.

### dev-dependencies and build-dependencies

```toml
[dev-dependencies]
tempfile = "3"

[build-dependencies]
cc = "1"
```

`[dependencies]` ships with the compiled crate and is required by anyone who depends on
it. `[dev-dependencies]` is only pulled in for `cargo test`, `cargo bench`, and examples
built from this crate, never by downstream users, which is the right place for test
helpers like `tempfile` or `assert_matches`. `[build-dependencies]` is a third, separate
graph used only by a `build.rs` script if the crate has one; it is compiled for the
machine running the build, not the target platform, which matters for cross-compilation.

---

## Features

### Declaring and gating features

```toml
[features]
default = ["json"]
json = ["dep:serde_json"]
yaml = ["dep:serde_yaml"]

[dependencies]
serde_json = { version = "1", optional = true }
serde_yaml = { version = "0.9", optional = true }
```

A feature is a named flag that a downstream crate (or you, from the command line) can
turn on or off. Marking a dependency `optional = true` removes it from the default
build and, since Rust 2021's `dep:` syntax, also removes the implicit feature of the
same name that older Cargo versions used to create automatically, so `dep:serde_json`
here is what actually pulls the crate in when the `json` feature is enabled. Inside your
own code, a feature gates compilation with a `cfg` attribute:

```rust
#[cfg(feature = "json")]
fn load_json(path: &str) -> serde_json::Value {
    // ...
    todo!()
}
```

Code behind `#[cfg(feature = "json")]` is not merely skipped at runtime, it is not
compiled at all when that feature is off, so it cannot bloat a binary that never enables
it and it cannot reference a dependency that isn't there. `cfg(feature = "...")` is the
same mechanism `cfg(test)` uses (see
[Chapter 26: Testing](IX.26-testing.md#conditional-compilation-with-cfgtest)); a feature
is just a `cfg` flag that lives in `Cargo.toml` instead of being built into the
compiler.

### Enabling features from the command line

```bash
cargo build --features json
cargo build --features "json yaml"
cargo build --no-default-features --features yaml
cargo build --all-features
```

`default` in the `[features]` table is itself just a feature, one that's on unless the
caller passes `--no-default-features`. Features are additive across a whole dependency
graph: if two crates in the same build both depend on the same third crate but request
different features of it, that crate is compiled once with the union of everything
anyone asked for. This is why a feature should only ever add capability, never change
behavior that other code might already depend on; a feature is not the right tool for
"pick one of two incompatible implementations."

---

## Profiles

### dev vs release

```bash
cargo build          # dev profile: target/debug/
cargo build --release  # release profile: target/release/
```

Cargo builds under a named profile, and the two built-in ones default to opposite
tradeoffs. `dev` optimizes for compile speed and debuggability: `opt-level = 0` (little
to no optimization), full debug info, overflow checks on. `release` optimizes for
runtime speed: `opt-level = 3`, overflow checks off, debug info stripped by default.
This is why an unoptimized `dev` binary can be many times slower than the same code
under `--release`, and also why integer overflow panics in `dev` builds but silently
wraps in `release` unless `overflow-checks` is turned back on explicitly (see
[Chapter 2: Data Types](I.2-data-types.md#integer-overflow-and-checked-arithmetic)).

### Customizing a profile

```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

Each of these trades compile time for runtime characteristics. `opt-level` (0 to 3, or
`"s"`/`"z"` for size) controls how aggressively LLVM optimizes; higher is slower to
compile and usually faster to run. `lto` (link-time optimization) lets the optimizer see
across crate boundaries instead of optimizing each compilation unit in isolation, which
can inline and dead-code-eliminate more but makes linking much slower. `codegen-units`
controls how many pieces LLVM splits a crate into for parallel compilation; the default
(256 in `dev`, 16 in `release`) trades some runtime performance for parallel build
speed, and setting it to `1` gives the optimizer the whole crate to work with at the
cost of a fully serial compile. `panic = "abort"` removes the unwinding machinery a
panic normally uses to run destructors up the stack, producing a smaller, slightly
faster binary that can no longer be caught with `catch_unwind`. `strip = true` removes
debug symbols from the final binary, shrinking it. None of this is free: the same
settings that shrink and speed up a release binary can turn a two-second `cargo build`
into a two-minute one, which is why they belong under `[profile.release]` rather than
in the defaults used for everyday `cargo build` / `cargo check` iteration.

### Custom profiles

```toml
[profile.profiling]
inherits = "release"
debug = true
```

`inherits` builds a new named profile on top of an existing one, overriding only the
fields that differ. This one keeps `release`'s optimizations but adds back debug info,
useful for running a profiler against an optimized binary. It's invoked with
`cargo build --profile profiling`, and its output goes to `target/profiling/`, not
`target/release/`.

---

## Workspaces

### Declaring a workspace

```toml
# top-level Cargo.toml, no [package] table
[workspace]
members = ["core", "cli", "macros"]
resolver = "3"
```

A workspace groups several crates that share one `Cargo.lock` and one `target/`
directory. Sharing the lock file means every member crate resolves a common dependency
(say, `serde`) to the same version, instead of each crate potentially locking a
different one; sharing `target/` means a dependency compiled for one member is reused
by another member that needs the same dependency at the same version, instead of
rebuilding it once per crate. `resolver = "3"` (the default for `edition = "2024"`
workspaces) opts into Cargo's newer feature-unification rules, which avoid unifying
features across build targets, dev-dependencies, and normal dependencies that would
otherwise leak unwanted features into a release build.

### Member and workspace-level settings

```toml
# top-level Cargo.toml
[workspace]
members = ["core", "cli"]

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }

# core/Cargo.toml
[package]
name = "core"
version = "0.1.0"
edition = "2024"

[dependencies]
serde.workspace = true
```

`[workspace.dependencies]` declares a dependency's version once at the workspace root;
member crates opt in with `dependency.workspace = true` instead of repeating the version
string, so bumping `serde` for the whole workspace is a one-line change at the top
instead of an edit in every member's `Cargo.toml`. A member crate can still add its own
`features` on top of a workspace dependency, or depend on a sibling member directly with
a `path`:

```toml
[dependencies]
core = { path = "../core" }
```

---

## Cargo Commands

### Building and running

```bash
cargo build            # compile, don't run
cargo build --release  # compile with the release profile
cargo run              # compile (if needed) and run the default binary
cargo run --bin tcodelab -- arg1 arg2   # run a specific binary, passing args after --
cargo check            # type-check without generating machine code
```

`cargo check` runs the compiler's parsing, type-checking, and borrow-checking passes but
stops before codegen, the slowest phase of a build. For a tight edit-check loop that's
often five to ten times faster than a full `cargo build`, which is why it's the command
most editors run on save; run `cargo build` or `cargo run` only when you actually need a
binary.

### Testing and linting

```bash
cargo test                    # run all tests
cargo test some_test_name     # run only tests whose name contains this substring
cargo test -- --nocapture     # let println! output through instead of capturing it
cargo clippy                  # lint for common mistakes and non-idiomatic code
cargo fmt                     # reformat the crate per rustfmt's style rules
```

`cargo test` and `cargo clippy` both compile the crate first, under a profile suited to
that command, so the first run after a change pays a compile cost the same way
`cargo build` does. Flags after a lone `--` are passed through to the test binary itself
rather than interpreted by Cargo, which is why `--nocapture` goes after it.

### Managing dependencies

```bash
cargo add serde --features derive
cargo remove serde_yaml
cargo update              # re-resolve dependencies within existing version requirements
cargo update -p serde     # re-resolve just one dependency
```

`cargo add` edits `Cargo.toml` for you and re-resolves `Cargo.lock`; it is equivalent to
hand-editing the `[dependencies]` table and then running a build, but it also looks up
the latest version matching the requirement you gave. `cargo update` never changes a
version *requirement* in `Cargo.toml`, it only changes which concrete version
`Cargo.lock` has pinned within what the requirements already allow, which is why it
can't cross a `^1` to `^2` boundary on its own.

### Documentation

```bash
cargo doc --open   # build rustdoc HTML and open it in a browser
```

`cargo doc` renders the same `///` doc comments and `# Examples` blocks covered in
[Chapter 26: Testing](IX.26-testing.md#doc-tests) into browsable HTML, including
cross-linked types from every dependency, which is what generates the pages published
on docs.rs for any crate published to crates.io.

---

## Quick Reference

### Common `cargo` subcommands

| Command | Purpose |
|---|---|
| `cargo new`/`cargo init` | Scaffold a new package |
| `cargo build` | Compile (add `--release` for the release profile) |
| `cargo run` | Compile and run the default binary |
| `cargo check` | Type-check without codegen (fast) |
| `cargo test` | Run unit, integration, and doc tests |
| `cargo clippy` | Lint |
| `cargo fmt` | Format |
| `cargo add`/`cargo remove` | Edit `[dependencies]` and re-resolve |
| `cargo update` | Re-resolve `Cargo.lock` within existing requirements |
| `cargo doc --open` | Build and view rustdoc |
| `cargo metadata` | Print the resolved dependency graph as JSON |

### Dependency section quick reference

| Table | Compiled for | Available to downstream crates |
|---|---|---|
| `[dependencies]` | Every build | Yes |
| `[dev-dependencies]` | `cargo test`/`bench`/examples only | No |
| `[build-dependencies]` | The host, for `build.rs` | No |

### `const` vs. `static`

See [Chapter 2: Data Types](I.2-data-types.md#const-vs-static) for that comparison;
it is unrelated to Cargo but is a common mix-up at the same "project setup" stage of
learning Rust.

---

## Common Patterns

- Bare version strings like `"1.2.3"` are caret requirements; write `"=1.2.3"` explicitly when a dependency needs to be pinned to one exact version. See Version Requirements above.
- Use `[workspace.dependencies]` plus `dependency.workspace = true` to keep one version of a shared dependency across every crate in a workspace instead of repeating (and risking drift in) the version string per member. See Member and Workspace-Level Settings above.
- Reach for `inherits` to define a custom profile (a profiling build, a smaller release build) instead of duplicating an entire `[profile.release]` block. See Custom Profiles above.
- Use `cargo check` while iterating and save `cargo build`/`cargo run` for when a binary is actually needed; the type-checking passes are identical, only codegen is skipped. See Building and Running above.

---

## Common Pitfalls

- `cargo update` cannot cross a major (or, pre-1.0, minor) version boundary on its own; bumping `serde = "1"` to `serde = "2"` requires editing the requirement in `Cargo.toml` first. See Managing Dependencies above.
- A feature can only add capability. Two crates in the same dependency graph that enable different features of a shared dependency get the union of both, so a feature that changes rather than adds behavior will silently affect code that never asked for it. See Enabling Features from the Command Line above.
- `path` and `git` dependencies build fine locally but block publishing to crates.io unless a registry `version` is also given as a fallback; a crate meant for crates.io should prefer workspace `path` dependencies only between members that are published together. See Path and Git Dependencies above.
- Debug assertions and overflow checks are only on by default in the `dev` profile. Code that "worked" under `cargo run` can panic or wrap silently once shipped under `cargo run --release`. See dev vs. release above.

---

## Summary

`Cargo.toml` declares a package's identity, targets, dependencies, and features;
`Cargo.lock` pins the versions actually resolved so builds stay reproducible.
Dependencies are matched by semver-aware version requirements and separated into
`[dependencies]`, `[dev-dependencies]`, and `[build-dependencies]` depending on when
they're needed. Features are additive `cfg` flags resolved as a union across the whole
build. Profiles (`dev`, `release`, or custom ones built with `inherits`) trade compile
time against runtime speed, binary size, and debuggability. Workspaces let several
crates share one lock file and build directory, with `[workspace.dependencies]` keeping
shared versions in one place. The command table and dependency-table comparison above
cover the day-to-day `cargo` invocations.
