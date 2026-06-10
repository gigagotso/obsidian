# Cross-compiling a static Linux binary from macOS

The deploy story for nginx-monitor — and the right shape for any Rust ops binary:

1. Build on your dev machine (macOS).
2. Target: `x86_64-unknown-linux-musl` — fully static, no glibc dependency.
3. Result: one binary, scp to the server, done.

No Rust, no runtime, no shared libs needed on the server.

## Why musl, not glibc

Linux has two C libraries:

| | glibc | musl |
|---|---|---|
| What it is | The default system libc on most Linux distros | An alternative, minimalist libc |
| Binary depends on | The server's `/lib/libc.so.6` of the right version | Nothing — statically linked into the binary |
| Cross-version compatibility | Build host's glibc must be ≤ server's glibc | Doesn't matter — no shared lib at runtime |
| Binary size | Smaller | A bit larger (~3–5 MB for ours) |
| Suitable for | Same-distro deploys, container builds | Single-binary "scp anywhere" deploys |

For an ops tool you scp around, **musl** is the right answer. The binary just works on any Linux x86_64 system without "missing GLIBC_2.38" errors.

## One-time setup

```sh
# Install the Rust target.
rustup target add x86_64-unknown-linux-musl

# Cross-compiler (only needed on macOS — Linux build hosts have it).
brew install FiloSottile/musl-cross/musl-cross
```

That installs `x86_64-linux-musl-gcc` and friends under `/opt/homebrew/bin/`.

## Tell Cargo which linker to use

The default linker on macOS is Apple's `ld`, which doesn't understand GNU linker flags. Rust emits GNU flags for the musl target. We need to point Cargo at the musl-cross linker explicitly.

`.cargo/config.toml` (at the workspace root):

```toml
[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
ar     = "x86_64-linux-musl-ar"
```

This applies only when building for the musl target — your native macOS builds are unaffected.

Why this matters: without it you get errors like:

```
ld: unknown options: --as-needed -Bstatic --eh-frame-hdr …
```

Apple's linker rejecting GNU's option flags.

## The build command

```sh
cargo build --release --target x86_64-unknown-linux-musl -p nginx-monitor
```

What each flag does:

- `--release` — full optimisations + LTO. Slow first build (~3 min), fast incremental (~5 s).
- `--target x86_64-unknown-linux-musl` — produce a Linux binary, not a macOS one.
- `-p nginx-monitor` — only this binary (and its deps from `monitor-core`).

Output: `target/x86_64-unknown-linux-musl/release/nginx-monitor` — ~4 MB, single file.

Verify:

```sh
file target/x86_64-unknown-linux-musl/release/nginx-monitor
# ELF 64-bit LSB pie executable, x86-64, static-pie linked, stripped
```

`static-pie` = statically linked, position-independent. Drops anywhere.

## Profile-level optimisations

In the workspace `Cargo.toml`:

```toml
[profile.release]
lto           = "thin"      # link-time optimisation across crate boundaries
codegen-units = 1           # single codegen unit = better inlining at cost of compile time
strip         = "symbols"   # strip debug symbols from the binary
panic         = "abort"     # smaller binary; panics terminate instead of unwinding
```

These all trade compile time for smaller / faster binaries. Worth it for a deploy artifact.

## Deploy

```sh
scp target/x86_64-unknown-linux-musl/release/nginx-monitor \
    root@35.156.76.29:/usr/local/bin/
ssh root@35.156.76.29 chmod +x /usr/local/bin/nginx-monitor
```

No companion files needed. Config lives separately (passed via `--config /path/to/config.json`).

## Alternative: build inside Docker

If you don't want to install the cross-compiler locally, use Rust's official image:

```sh
docker run --rm \
    -v "$PWD":/work \
    -w /work \
    rust:1-alpine \
    cargo build --release --target x86_64-unknown-linux-musl -p nginx-monitor
```

Alpine's image has musl built-in. ~30–60 s per build after the first one (image cached locally).

Slower iteration than native cargo, but no host installs needed. Useful when:

- You're on a machine without homebrew (Windows, fresh Linux dev VM, …)
- You want CI to build the binary reproducibly
- You want zero host-toolchain footprint

## Alternative: the `cross` tool

```sh
cargo install cross
cross build --release --target x86_64-unknown-linux-musl -p nginx-monitor
```

`cross` is a community tool that wraps Docker so you don't have to write the docker incantation. Same Docker requirement.

## What ends up in the binary

A `--release --target musl` build of `nginx-monitor` contains:

- Your code, optimised
- All workspace deps (`monitor-core`, every transitive)
- musl libc — statically linked
- The Rust standard library — statically linked

Anything dynamic (HTTPS via `ureq`, which uses `rustls`) is also bundled. No "missing libssl" errors.

## Mental model

```mermaid
graph TB
    subgraph mac["macOS build host"]
        src[Source code]
        toolchain[rustup + cargo<br/>+ musl-cross]
        src -->|cargo build --target musl| toolchain
    end

    toolchain -->|produces| binary["nginx-monitor<br/>~4 MB ELF<br/>static-pie linked"]

    subgraph server["Any Linux x86_64 server"]
        binary -->|scp| placed[/usr/local/bin/nginx-monitor]
        config[/etc/nginx-monitor/config.json]
        state[/var/lib/nginx-monitor/]
        placed -.uses.-> config
        placed -.writes.-> state
    end

    classDef art fill:#d4f4dd,stroke:#2a8c3c
    classDef cfg fill:#dde7ff,stroke:#3858b3
    class binary,placed art
    class config,state cfg
```

## See also

- [[05-cargo-and-manifests|Cargo flags used here]]
- [[06-workspaces|The -p flag selects a workspace member]]
- [[15-crates-we-used|once_cell, ureq, regex etc. all statically linked]]
- [[17-case-study-nginx-monitor|How the binary lands on the production server]]
