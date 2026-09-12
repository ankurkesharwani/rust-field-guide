# Rust Field Guide

A chapter-by-chapter Rust reference, covering the language from basic syntax through
ownership, traits, collections, error handling, concurrency, async, macros, and unsafe
code.

## How it's organized

The reference is split into 28 chapters across ten parts (Foundations, Control Flow,
Ownership & Memory, Type System, Collections, Functional Programming, Error Handling,
Concurrency & Async, I/O & Ecosystem, and Advanced), kept in [chapters/](chapters/).
Every chapter follows the same shape:

- an introduction that says what the chapter covers and why it's split out the way it is
- the core content, broken into subsections with worked, compiled examples
- a Quick Reference section for fast lookup
- Common Patterns and Common Pitfalls
- a short Summary

Code examples come with prose explaining the concept they demonstrate: why a borrow
checker error happens the way it does, why `Rc::clone` is cheap, why `RefCell` panics at
runtime instead of failing to compile.

## Where to start

[chapters/00-index.md](chapters/00-index.md) has the full table of contents with links
to every chapter. If you're new to Rust, start at Part I and read in order; each part
builds on the ones before it. If you already know the language, the chapters also work
as standalone lookup material, cross-linked to each other where one concept depends on
another.

## Prerequisites

No prior Rust knowledge is assumed. A working Rust toolchain (`rustc`/`cargo`) is useful
if you want to run the examples yourself rather than read them.
