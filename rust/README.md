# Rust — Notes

A personal library of Rust notes, grounded in real code from the
[`livecaller-ops`](file:///Users/giga/Developer/livecaller/livecaller-ops/)
nginx-monitor project. Each note has concrete examples from what we built,
not abstract textbook explanations.

## Foundations (the original notes)

1. [[01-standard-library|What is `std`?]] — the standard library
2. [[02-use-keyword|The `use` keyword]] — bringing items into scope
3. [[03-prelude|The Prelude]] — what Rust auto-imports for every file
4. [[04-visualization|Visualizations]] — diagrams of crates, modules, scope

## Project, packaging, and modules

5. [[05-cargo-and-manifests|Cargo and `Cargo.toml`]] — the manifest, lockfile, features
6. [[06-workspaces|Workspaces]] — many crates, one repo
7. [[07-lib-vs-bin-crates|Library vs binary crates]] — what each is for
8. [[08-modules-and-visibility|Modules and visibility]] — `pub`, `pub(crate)`, file vs. inline

## Patterns

9.  [[09-error-handling-result-anyhow|Error handling]] — `Result`, `?`, `anyhow`, `Context`
10. [[10-traits-the-blocker-pattern|Traits as seams]] — the `Blocker` pattern
11. [[11-newtype-pattern|The newtype pattern]] — wrapping types to enforce invariants
12. [[12-lazy-statics-once-cell|Lazy statics]] — compile-once regexes with `once_cell::Lazy`
13. [[13-inline-tests|Inline unit tests]] — `#[cfg(test)] mod tests`

## Build and deploy

14. [[14-cross-compilation-musl|Cross-compilation]] — building a static Linux binary on macOS
15. [[15-crates-we-used|Crates we used]] — inventory of every external dep
16. [[16-cargo-cheatsheet|Cargo command cheatsheet]] — daily commands by use case

## Case study

17. [[17-case-study-nginx-monitor|nginx-monitor architecture]] — tying it all together with real code

## What this project taught (in one paragraph)

Build a Rust ops binary by structuring the repo as a Cargo **workspace**
containing a shared **library crate** (`monitor-core`) and one or more
**binary crates** (`nginx-monitor`, future `pbx-monitor`, …). The library
exports common building blocks (offset-tracked log reads, per-IP counter,
Slack formatter, block-state TSV, a `Blocker` **trait**); each binary wires
those together with its own log parser and its own `impl Blocker` for the
specific enforcement mechanism. Errors propagate via `anyhow::Result<T>`
and `?`; tests live inline via `#[cfg(test)] mod tests`; the deploy
artifact is a single ~4 MB **static-musl ELF** produced by
`cargo build --release --target x86_64-unknown-linux-musl`, scp'd to the
server with zero runtime dependencies.

## TL;DR commands

```sh
# One-time setup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add x86_64-unknown-linux-musl
brew install FiloSottile/musl-cross/musl-cross    # macOS only

# Daily iteration
cargo check -p nginx-monitor                 # ~5–10s, fastest feedback
cargo test --workspace                       # run unit tests
cargo build --release --target x86_64-unknown-linux-musl -p nginx-monitor
                                              # ~1 min first time, <10s after

# Deploy
scp target/x86_64-unknown-linux-musl/release/nginx-monitor root@host:/usr/local/bin/
```
