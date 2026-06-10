# Cargo command cheatsheet

The commands you'll reach for daily, grouped by what you're trying to do.

## Project setup

```sh
cargo new my_thing            # new binary crate (creates dir)
cargo new --lib my_lib        # new library crate
cargo init                    # initialise an existing directory as a crate
cargo init --bin              # … as a binary crate
cargo init --lib              # … as a library crate
```

For workspaces: create the dir + `Cargo.toml` with `[workspace]` manually, then `cargo init` each member.

## Iteration loop

```sh
cargo check                   # parse + type-check only. ~5–10× faster than build.
cargo check -p some-crate     # one workspace member
cargo check --workspace       # everything

cargo build                   # debug build — fast compile, slow code, big binary
cargo build --release         # optimised. Slow compile, fast code, smaller binary.

cargo run                     # build + run (debug profile)
cargo run --release           # build + run (release profile)
cargo run -- --some-flag      # pass args to your binary
cargo run -p nginx-monitor -- monitor --config ./config.json    # workspace member + binary args

cargo clean                   # nuke target/. Forces a clean rebuild.
cargo clean -p some-crate     # just one member's build artifacts
```

The fast inner loop while writing code: `cargo check -p <crate>` — gets you compile errors in seconds. Run tests less often, builds even less.

## Tests

```sh
cargo test                    # everything
cargo test -p monitor-core    # one crate
cargo test counter            # only tests matching "counter" in their path
cargo test -- --nocapture     # show println! output (default: suppressed on pass)
cargo test -- --test-threads=1 # run serially (useful if tests share fs/env)
cargo test --release          # tests under release profile (find perf regressions)
```

## Dependencies

```sh
cargo add serde --features derive          # add a dep, edit Cargo.toml for you
cargo add serde_json
cargo remove some_dep                       # remove from Cargo.toml
cargo update                                # bump within SemVer range, update Cargo.lock
cargo update -p regex                       # update one dep
cargo tree                                  # tree of all transitive deps
cargo tree -d                               # show only duplicate deps (different versions)
```

## Cross-compilation (for our case)

```sh
rustup target add x86_64-unknown-linux-musl                  # one-time
cargo build --release --target x86_64-unknown-linux-musl -p nginx-monitor
```

See [[14-cross-compilation-musl]] for the `.cargo/config.toml` glue needed on macOS.

## Documentation

```sh
cargo doc                     # build docs for your crate + deps
cargo doc --open              # … and open in browser
cargo doc --no-deps           # skip deps' docs (faster, smaller output)
```

Docs come from your `///` doc comments. They render as HTML next to the actual API.

## Profile / debug

```sh
cargo build --timings         # generates target/cargo-timings/cargo-timing.html
                              # — shows which crates took longest to compile
cargo +nightly rustc -- -Z unstable-options --emit=mir   # advanced: see MIR
```

## Toolchain (`rustup`)

`cargo` is the package manager; `rustup` manages the toolchain itself.

```sh
rustup update                    # latest stable Rust
rustup install nightly           # also get the nightly channel
rustup default stable            # which channel cargo uses by default
rustup target list               # all available cross-compile targets
rustup target add aarch64-unknown-linux-musl    # add a target (we did this for x86_64)
rustup component add clippy      # add Clippy (lints)
rustup component add rustfmt     # add rustfmt (auto-formatter)
rustup component add rust-analyzer   # the official LSP server
```

## Formatting + linting

```sh
cargo fmt                     # auto-format everything (uses rustfmt)
cargo fmt -- --check          # CI mode: fail if anything would change
cargo clippy                  # lint your code (catches a huge class of mistakes)
cargo clippy --workspace --all-targets -- -D warnings   # CI mode: warnings as errors
```

Run `cargo clippy` regularly — it's one of the most valuable tools in the Rust ecosystem. Catches stylistic issues, common bugs, perf footguns.

## Quick "what's in this crate?" intel

```sh
cargo metadata --format-version 1 | jq .packages[].name     # all deps + your crates
cargo pkgid -p nginx-monitor                                 # the exact spec of a crate
cat Cargo.lock | grep '^name = "regex"' -A 1                 # version of regex actually pinned
```

## Common errors you'll see and what they mean

| Error | Meaning |
|---|---|
| `cannot borrow X as mutable, as it is behind a &` | You have a shared (`&T`) reference; you need `&mut T` |
| `borrowed value does not live long enough` | A reference outlives the value it points to. Rework lifetimes. |
| `the trait bound `T: SomeTrait` is not satisfied` | Type `T` doesn't implement the trait you need. Either implement it or constrain `T` differently. |
| `unused import / unused variable` | Warnings — clean up to keep the build clean |
| `cannot find type `X` in this scope` | Missing `use ...` or `mod ...;` |
| `expected `&T`, found `T`` | Add `&` where you call the function, or change the signature |

## The "I'm stuck" sequence

When you can't figure out why something isn't compiling:

1. `cargo check` — read the actual error, not the secondary "see also" hints.
2. If it's about a trait → check if you need `use SomeTrait;` (traits must be in scope to call their methods — see [[02-use-keyword]]).
3. If it's about lifetimes → simplify. Often a `.to_string()` or `.clone()` here is the right call while you're learning; optimise references later.
4. `cargo clippy` — sometimes Clippy explains the same error more clearly.
5. `rustc --explain E0502` — the long form of any error code.

## See also

- [[05-cargo-and-manifests]]
- [[06-workspaces]]
- [[14-cross-compilation-musl]]
- [[15-crates-we-used]]
