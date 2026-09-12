# Modules, Crates & Visibility in Rust

## Table of Contents

1. [Introduction](#introduction)
2. [Modules with `mod`](#modules-with-mod)
3. [Bringing Paths into Scope with `use`](#bringing-paths-into-scope-with-use)
4. [Re-exports](#re-exports)
5. [Crate vs. Module Tree](#crate-vs-module-tree)
6. [Visibility and Privacy](#visibility-and-privacy)
7. [Quick Reference](#quick-reference)
8. [Common Patterns](#common-patterns)
9. [Common Pitfalls](#common-pitfalls)
10. [Summary](#summary)

---

## Introduction

Rust organizes code into a tree of modules within a crate, and controls what other code
can see with a small set of visibility keywords. This chapter covers `mod` and `use`,
re-exporting items, how the crate/module tree fits together, and the privacy rules that
were previously part of the Custom Types chapter. Module organization is a distinct
topic from struct/enum design, so it lives here instead.

---

## Modules with `mod`

`mod` declares a module: a named container for items (functions, structs, enums,
traits, other modules) that groups related code and gives it its own path.

### Inline Modules

The simplest form puts the module's body directly in braces:

```rust
mod shapes {
    pub fn describe() -> &'static str {
        "a shape"
    }

    pub mod circle {
        pub fn area(radius: f64) -> f64 {
            std::f64::consts::PI * radius * radius
        }
    }
}

fn main() {
    println!("{}", shapes::describe());
    println!("{}", shapes::circle::area(2.0));
}
```

`shapes` and the nested `circle` module each get their own path (`shapes::describe`,
`shapes::circle::area`), and every item inside a module is private to that module by
default; only the `pub` items are reachable from outside. See Visibility and Privacy
below for the full set of rules.

### File-per-Module

Once a module's body grows, it's more common to move it into its own file and declare
it with a `mod` statement that has no body:

```rust
// in src/main.rs (or lib.rs)
mod shapes;  // looks for src/shapes.rs, or src/shapes/mod.rs
```

```rust
// in src/shapes.rs
pub fn describe() -> &'static str {
    "a shape"
}

pub mod circle;  // looks for src/shapes/circle.rs
```

```rust
// in src/shapes/circle.rs
pub fn area(radius: f64) -> f64 {
    std::f64::consts::PI * radius * radius
}
```

`mod shapes;` tells the compiler "the body of this module lives in another file," and it
looks in two places: `shapes.rs` next to the file that declared it, or `shapes/mod.rs`.
Both forms are still supported, but the `foo.rs` + `foo/` sibling-directory layout (used
above, where `circle.rs` sits inside a `shapes/` directory next to `shapes.rs`) is the
modern convention; the older `foo/mod.rs` style is mostly seen in pre-2018-edition code.
Either way, the module tree in your code (`shapes::circle`) mirrors the directory
structure on disk, which is what lets you jump from a `use` path straight to a file.

### Nested Module Trees

Modules nest arbitrarily deep, whether inline or file-per-module, and the path to an
item is just the chain of module names joined with `::`:

```rust
mod app {
    pub mod ui {
        pub mod widgets {
            pub fn render_button() -> &'static str {
                "[ OK ]"
            }
        }
    }
}

fn main() {
    println!("{}", app::ui::widgets::render_button());
}
```

`app::ui::widgets::render_button` is the full path from the crate root. A file-based
version of this tree would put `widgets.rs` inside `src/app/ui/`, mirroring the nesting
on disk.

---

## Bringing Paths into Scope with `use`

Writing out a full path like `shapes::circle::area(2.0)` every time gets tedious. `use`
brings a path into scope so you can refer to it by its last segment instead.

```rust
mod shapes {
    pub mod circle {
        pub fn area(radius: f64) -> f64 {
            std::f64::consts::PI * radius * radius
        }
    }
    pub mod square {
        pub fn area(side: f64) -> f64 {
            side * side
        }
    }
}

use shapes::circle;
use shapes::square::area as square_area; // `as` renaming

fn main() {
    println!("{}", circle::area(2.0));   // brought `circle` into scope, not `area` itself
    println!("{}", square_area(3.0));    // renamed on import to avoid clashing with circle::area
}
```

`use shapes::circle` imports the module `circle`, not its contents, so you still write
`circle::area`. Importing a function directly (`use shapes::circle::area;`) would let
you call it as `area(2.0)`, but then a second `use` for `shapes::square::area` would
collide with it. `as` renaming sidesteps that: `square::area` becomes `square_area`
locally, without changing its real name anywhere else.

### Nested Grouping

Multiple `use` paths that share a prefix can be combined with curly braces:

```rust
use std::collections::{HashMap, HashSet};
use std::fmt::{self, Display};
```

`self` inside the braces (as in `fmt::{self, Display}`) imports `fmt` itself alongside
`fmt::Display`, so both `fmt::Result` and a bare `Display` are usable.

### Glob Imports

`use shapes::circle::*;` imports every public item from `circle` at once. This is
convenient for test modules (`use super::*;`, covered in
[Chapter 26: Testing](IX.26-testing.md)) and for preludes, but in ordinary code it makes
it harder to tell where a name came from, so most style guides reserve it for those
specific cases rather than everyday imports.

---

## Re-exports

`pub use` re-exports an item: it brings a path into scope the same way `use` does, but
also makes that name part of the current module's own public API, as if the item had
been defined there directly.

```rust
mod outer {
    pub mod inner {
        pub fn helper() -> i32 {
            42
        }
    }

    // Re-export: callers can reach `helper` via `outer::helper`, not just `outer::inner::helper`
    pub use inner::helper;
}

fn main() {
    println!("{}", outer::inner::helper()); // still works: the original path
    println!("{}", outer::helper());        // also works: the re-exported path
}
```

This is the standard way library authors flatten a deep internal module tree into a
simpler public one: internal code stays organized into whatever nested modules make
sense to maintain, while `pub use` at the crate root re-exports the pieces users are
meant to reach directly (`my_crate::Client` instead of
`my_crate::transport::http::client::Client`). Renaming works the same way it does for
plain `use`: `pub use inner::helper as run;` re-exports it under a different name.

---

## Crate vs. Module Tree

A crate is the unit `cargo` compiles: one crate produces one library or one binary.
Every module you declare with `mod` lives inside some crate's module tree, rooted at
that crate's entry file.

- A **binary crate** has a `main` function and compiles to an executable. Its module
  tree is rooted at `src/main.rs`.
- A **library crate** has no `main` function and is meant to be used by other crates
  (or by a binary crate in the same package). Its module tree is rooted at `src/lib.rs`.

A single Cargo package can contain both: a `src/lib.rs` holding the real logic as a
library crate, and one or more files under `src/bin/` (or a `src/main.rs`) as thin
binary crates that call into it. This project (`terminal-codelab`) is set up the simpler
way, as a single binary crate: its `Cargo.toml` has one `[[bin]]` target pointing at
`src/main.rs`, and every module (`cli`, `commands`, `config`, `display`, `utils`) is
declared with `mod` from that one root, forming a single tree.

`mod` never crosses a crate boundary; it only declares modules within the current
crate's tree. To use another crate's public items, you depend on it in `Cargo.toml` (see
[Chapter 5: Cargo & Project Structure](I.5-cargo-and-project-structure.md)) and refer to
it by its crate name, the same way `std::collections::HashMap` refers to the `std`
crate's `collections` module.

---

## Visibility and Privacy

### Default Privacy

Everything declared inside a module, whether a struct, a field, or a method, is private
to that module and its descendants unless explicitly marked `pub`. A struct can be
`pub` while individual fields and methods stay private, which is how a type exposes a
constructor and a read-only accessor while keeping its internal representation free to
change:

```rust
mod my_module {
    // Private by default
    struct PrivateStruct {
        field: i32,
    }

    // Public struct with private field
    pub struct PublicStruct {
        pub public_field: i32,
        private_field: i32,  // Still private!
    }

    impl PublicStruct {
        pub fn new(value: i32) -> Self {
            PublicStruct {
                public_field: value,
                private_field: value * 2,
            }
        }

        // Private method
        fn private_helper(&self) -> i32 {
            self.private_field
        }

        // Public method
        pub fn get_private(&self) -> i32 {
            self.private_helper()
        }
    }
}

fn main() {
    let s = my_module::PublicStruct::new(5);
    println!("{}", s.public_field);
    // println!("{}", s.private_field);  // ERROR: private field
}
```

### pub(crate) and Other Restrictions

Plain `pub` exposes an item to any code that can reach its path, including other
crates. The restricted forms narrow that: `pub(crate)` stops at the current crate's
boundary, `pub(super)` stops at the immediate parent module, and `pub(in path)` stops at
whatever module path you name. Each is useful when an item needs to be shared across
module boundaries internally without becoming part of the crate's public API:

```rust
mod outer {
    pub(crate) struct CrateVisible {
        pub(crate) field: i32,
    }

    pub mod inner {
        pub(super) struct SuperVisible {
            pub(super) field: i32,
        }

        pub(in crate::outer) struct PathVisible {
            field: i32,
        }
    }
}
```

### Enum Visibility

A `pub` enum's variants are always reachable wherever the enum itself is reachable;
there's no way to mark one variant more restricted than the enum. That's different from
a `pub struct`, where each field's visibility is set independently:

```rust
mod my_mod {
    // If enum is pub, all variants are pub
    pub enum PublicEnum {
        Variant1,
        Variant2(i32),
        Variant3 { field: String },
    }

    // Struct fields in variants follow normal rules
    // (fields are pub if not explicitly marked otherwise)
}
```

---

## Quick Reference

| Visibility | Meaning |
|---|---|
| (none, default) | Visible only within the current module and its descendants |
| `pub` | Visible to any code that can reach the item's path |
| `pub(crate)` | Visible anywhere within the current crate |
| `pub(super)` | Visible to the parent module |
| `pub(in path)` | Visible within the given module path |

A `pub` enum's variants are always `pub`; there's no way to make one variant more
restricted than the enum itself.

---

## Common Patterns

- Start with inline modules while a file is small, and split a module out to its own
  file once its body grows large enough to make scrolling past it annoying. See Modules
  with `mod` above.
- Re-export the pieces callers actually need at a shallow path (`pub use` at the crate
  root) while keeping the implementation organized into whatever deeper module tree
  makes sense internally. See Re-exports above.
- Group `use` paths that share a prefix with `{...}`, and reach for `as` renaming when
  two imports would otherwise collide on the same last segment. See Bringing Paths into
  Scope with `use` above.
- Default new items to private and widen visibility only as far as a real caller needs:
  `pub(crate)` for cross-module use within the same crate, `pub` only for the actual
  public API. See Visibility and Privacy above.

---

## Common Pitfalls

- `use shapes::circle;` imports the module, not its contents; calling `area(2.0)`
  directly (instead of `circle::area(2.0)`) is a compile error unless you specifically
  imported the function itself. See Bringing Paths into Scope with `use` above.
- A struct being `pub` doesn't make its fields `pub`; each field's visibility is
  declared separately, so a public struct can still hide its internals behind private
  fields and public accessor methods. See Visibility and Privacy above.
- Glob imports (`use shapes::circle::*;`) make it hard to tell where a name came from
  once a module has more than a couple of exports; they're best kept to test modules
  and preludes rather than everyday code. See Bringing Paths into Scope with `use`
  above.
- `mod foo;` looks for `foo.rs` or `foo/mod.rs` relative to the declaring file, not
  relative to the crate root; a module declared inside `src/app/mod.rs` looks for its
  children under `src/app/`, not `src/`. See Modules with `mod` above.

---

## Summary

This chapter covered how Rust organizes code: `mod` declares a module, either inline or
backed by its own file; `use` brings a path into scope, with `as` for renaming and `{}`
for grouping; `pub use` re-exports an item as part of the current module's own API; and
a crate (binary or library) is the root that every module tree hangs off of. On top of
that structure sits Rust's default-private visibility model: items are private to their
module and its descendants unless marked `pub`, `pub(crate)`, `pub(super)`, or
`pub(in path)`, and a public enum's variants inherit its visibility.
