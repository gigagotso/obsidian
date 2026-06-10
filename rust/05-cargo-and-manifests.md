# Cargo and the `Cargo.toml` manifest

Cargo is Rust's package manager + build tool + test runner + docs generator + benchmark runner, all in one. You almost never call `rustc` directly — you call `cargo`, and it calls `rustc` for you with the right flags.

A Rust project is called a **crate**. A crate is one `Cargo.toml` file plus a `src/` directory.

## The manifest: `Cargo.toml`

```toml
[package]
name         = "nginx-monitor"
version      = "0.1.0"
edition      = "2021"
rust-version = "1.70"
license      = "MIT"

[[bin]]
name = "nginx-monitor"
path = "src/main.rs"

[dependencies]
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
regex       = "1"
anyhow      = "1"
clap        = { version = "4", features = ["derive"] }

[dev-dependencies]
tempfile = "3"
```

What each section does:

| Section | Purpose |
|---|---|
| `[package]` | Crate identity: name, version, what Rust edition (2015/2018/2021/2024) it targets. |
| `[[bin]]` | "I produce an executable." Multiple `[[bin]]` sections = multiple binaries from one crate. Default is `src/main.rs` if omitted. |
| `[lib]` | "I produce a library." Default is `src/lib.rs`. A crate can have both `[lib]` and `[[bin]]`. |
| `[dependencies]` | Runtime crates pulled in. Versions are SemVer-ish: `"1"` means `>=1.0.0, <2.0.0`. |
| `[dev-dependencies]` | Only compiled for `cargo test` and `cargo bench`. `tempfile` is the canonical example. |

## Features

A crate can have optional capabilities behind named flags:

```toml
serde = { version = "1", features = ["derive"] }
```

Without the `derive` feature, you'd get `serde::Serialize` and `Deserialize` as traits but not the `#[derive(Serialize)]` proc-macro. Most crates ship powerful capabilities behind features so consumers don't pay the compile cost unless asked.

## The lockfile: `Cargo.lock`

`Cargo.toml` says "I want regex `1.*`". `Cargo.lock` records "I resolved that to `regex = 1.10.5` exactly, here are the SHA-256 hashes of every transitive dep."

Convention:

- **Binary crates** (programs you ship) → **commit** `Cargo.lock`. Builds are reproducible across machines.
- **Library crates** (published to crates.io for others to use) → **ignore** `Cargo.lock`. Let downstream binaries pin their own versions.

For the nginx-monitor workspace, the lock is committed because the workspace contains a binary.

## Quick command reference

```sh
cargo new my_project              # create new crate (default: binary)
cargo new --lib my_lib            # create new library crate

cargo build                       # debug build (fast compile, slow code)
cargo build --release             # release build (slow compile, fast code)

cargo check                       # parse + type-check, skip codegen (fast feedback)
cargo run                         # build + run
cargo run -- --some-arg           # pass args to the binary

cargo test                        # run all tests
cargo test some_name              # run only tests with "some_name" in the path

cargo doc --open                  # generate + open docs for your crate + deps

cargo add serde                   # add a dep to Cargo.toml without editing by hand
cargo update                      # bump deps within their SemVer range
```

See [[18-cargo-cheatsheet]] for the full set with options.

## Why the editions

Every few years, Rust ships an "edition" (2015, 2018, 2021, 2024) that lets the language make small backward-incompatible improvements. Your crate picks one in `Cargo.toml`; the compiler then applies that edition's rules to your code. Crates on different editions can call each other freely — it's strictly a source-level distinction. Use the latest you can (`2021` is current, `2024` upcoming).

## See also

- [[06-workspaces|Workspaces — many crates, one repo]]
- [[07-lib-vs-bin-crates|Library crates vs binary crates]]
- [[15-crates-we-used|The crates we actually used]]
- [[18-cargo-cheatsheet|Cargo commands cheatsheet]]
