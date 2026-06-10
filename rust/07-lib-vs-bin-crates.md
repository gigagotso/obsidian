# Library crates vs binary crates

A Rust crate is one of two things — or, occasionally, both:

| | **Library crate** | **Binary crate** |
|---|---|---|
| Entry point | `src/lib.rs` | `src/main.rs` (or `src/bin/<name>.rs`) |
| Has `fn main()` | No | Required |
| Produces | `.rlib` (link target) | An executable file |
| Can be a dep of | Other crates | Nothing else |
| Has tests | Yes (`cargo test`) | Yes |
| Example in our repo | `monitor-core` | `nginx-monitor` |

## How Cargo decides which

By **convention**:

- `src/lib.rs` exists → library crate
- `src/main.rs` exists → binary crate
- Both exist → both
- You can override with `[lib]` and `[[bin]]` sections in `Cargo.toml`

```toml
# Library — Cargo.toml has no [[bin]] section
[package]
name = "monitor-core"

# Binary — Cargo.toml has a [[bin]]
[package]
name = "nginx-monitor"
[[bin]]
name = "nginx-monitor"
path = "src/main.rs"
```

## Multiple binaries from one crate

```
my-tool/
├── Cargo.toml
└── src/
    ├── lib.rs         ← shared logic
    └── bin/
        ├── server.rs  ← `cargo run --bin server`
        └── cli.rs     ← `cargo run --bin cli`
```

Both binaries can `use my_tool::*` to reach into the lib. This is how a tool like `cargo` itself is built: one crate, many subcommands as binaries.

## The "lib + bin in same crate" pattern

Common when you want testable internals plus a thin CLI:

```
my-tool/
├── Cargo.toml
└── src/
    ├── lib.rs        ← all the real logic, well-tested
    └── main.rs       ← parses args, calls into lib
```

`main.rs` does:

```rust
use my_tool::*;

fn main() {
    // parse CLI, call lib functions, format output
}
```

Why split? Integration tests in `tests/*.rs` can `use my_tool::*` because it's a library — but a binary-only crate can't be imported. Splitting unlocks testing.

We chose the **multi-crate** approach (separate `monitor-core` lib + `nginx-monitor` bin) for the same reason but at workspace scale: future binaries (`pbx-monitor`, …) reuse the lib too. See [[06-workspaces]].

## Three rules of thumb for what goes where

1. **Could this be reused by another binary?** → library
2. **Is it source-specific** (a parser for one log format, a CLI dispatch, an enforcement mechanism)? → binary
3. **Undecided?** → start in the binary. Promoting to the lib later is easy; demoting is awkward.

## Building & running

```sh
# Whole workspace
cargo build --release

# Just the library (no exe produced)
cargo build -p monitor-core

# Just the binary (also builds its lib deps)
cargo build --release -p nginx-monitor

# Run the binary directly without scp'ing it
cargo run -p nginx-monitor -- monitor --config ./config.json --state-dir ./state
```

The `--` separates Cargo's args from the binary's args. Everything after `--` gets passed to your program.

## What "depending on a library" looks like in code

In `nginx-monitor/Cargo.toml`:

```toml
[dependencies]
monitor-core = { path = "../monitor-core" }
```

In `nginx-monitor/src/main.rs`:

```rust
use monitor_core::reader::read_new_bytes;
use monitor_core::counter::Counter;
use monitor_core::blocker::{Block, BlockState, Blocker};
```

The crate name in code uses `_` even if the package name uses `-` (`monitor-core` → `monitor_core`). Rust replaces dashes with underscores in import paths.

## See also

- [[05-cargo-and-manifests|The Cargo.toml that declares which kind]]
- [[06-workspaces|Workspaces, where libs and bins live together]]
- [[08-modules-and-visibility|How items inside a crate become visible to others]]
