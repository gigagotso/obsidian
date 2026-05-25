# Rust Standard Library — Notes

A small library of notes explaining how the Rust **standard library** works, what `use std::io;` actually does, and what the **prelude** is.

## Contents

1. [[01-standard-library|What is `std`?]] — overview of the standard library
2. [[02-use-keyword|The `use` keyword]] — how `use std::io;` brings items into scope
3. [[03-prelude|The Prelude]] — the items Rust imports for you automatically
4. [[04-visualization|Visualizations]] — diagrams of crates, modules, and scope

## TL;DR

```rust
use std::io;          // bring the `io` module into scope
use std::io::Write;   // bring the `Write` trait into scope

fn main() {
    let mut buf = String::new();        // String is in the prelude — no `use` needed
    io::stdin().read_line(&mut buf).unwrap();
    println!("{}", buf);                // println! is a macro from the prelude
}
```

- `std` is the standard library crate.
- `use` creates a **shortcut** to a path — it does **not** copy code.
- The **prelude** is a tiny set of items (`String`, `Vec`, `Option`, `Result`, `println!`, …) that Rust imports into every file automatically so you don't have to write `use std::string::String;` every time.
