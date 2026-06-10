# Inline unit tests (`#[cfg(test)] mod tests`)

Rust tests live **next to the code they test**. The convention: a `tests` submodule at the bottom of each file, marked `#[cfg(test)]` so it's only compiled when running `cargo test`.

## The shape

```rust
// in some_file.rs

pub fn double(x: u32) -> u32 {
    x * 2
}

#[cfg(test)]
mod tests {
    use super::*;                  // bring parent's items into scope

    #[test]
    fn doubles_zero() {
        assert_eq!(double(0), 0);
    }

    #[test]
    fn doubles_normal() {
        assert_eq!(double(21), 42);
    }
}
```

What each attribute does:

- `#[cfg(test)]` — "compile this only when building for tests." Zero overhead in the release binary.
- `mod tests { … }` — a child module containing the tests. Conventional name; you can call it anything.
- `use super::*;` — pulls in everything from the enclosing module (the code being tested).
- `#[test]` — marks a function as a test case. The function must take no args and return either `()` or `Result<(), E>`.

## Running tests

```sh
cargo test                       # everything in workspace
cargo test -p monitor-core       # one crate
cargo test counter               # only tests whose path contains "counter"
cargo test -- --nocapture        # print stdout (default is suppressed for pass)
cargo test -- --test-threads=1   # serial (useful for tests that share fs/env)
```

## Real examples from our codebase

### Pure logic — easy

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use serde_json::json;

    #[test]
    fn class_threshold_catches_below_aggregate() {
        let mut c = Counter::new();
        for _ in 0..28 { c.observe(ip("1.2.3.4"), "any"); c.observe(ip("1.2.3.4"), "4xx"); }
        for _ in 0..2  { c.observe(ip("1.2.3.4"), "any"); c.observe(ip("1.2.3.4"), "2xx"); }

        let off = c.find_offenders(&thresholds(json!({ "any": 50, "4xx": 15 })));
        assert_eq!(off.len(), 1);
        assert_eq!(off[0].trips, vec![("4xx".to_string(), 15)]);
    }
}
```

This test runs in microseconds — no I/O, no setup. Most of `monitor-core::counter` is tested this way: in-memory inputs, asserted in-memory outputs.

### Tests with filesystem — use `tempfile`

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::io::Write;

    fn setup(initial: &str) -> (tempfile::TempDir, PathBuf, PathBuf) {
        let dir = tempfile::tempdir().unwrap();
        let log = dir.path().join("test.log");
        let state = dir.path().join("state");
        fs::create_dir_all(&state).unwrap();
        fs::write(&log, initial).unwrap();
        (dir, log, state)
    }

    #[test]
    fn rotation_resets_offset() {
        let (_d, log, state) = setup("old content\n");
        let _ = read_new_bytes(&log, &state).unwrap();      // seed at EOF

        // simulate logrotate
        let rotated = log.with_extension("log.1");
        fs::rename(&log, &rotated).unwrap();
        fs::write(&log, b"post-rotate\n").unwrap();

        let bytes = read_new_bytes(&log, &state).unwrap();
        assert_eq!(&bytes[..], b"post-rotate\n");
    }
}
```

`tempfile::tempdir()` creates a unique directory under `/tmp`. Returning the `TempDir` from `setup` keeps it alive — when the test ends, the directory is automatically deleted.

Add `tempfile` to `[dev-dependencies]` so it's only pulled in for tests, not the release build:

```toml
[dev-dependencies]
tempfile = "3"
```

## Assertions

| Macro | Use |
|---|---|
| `assert!(x)` | true / panic |
| `assert_eq!(a, b)` | equal / panic with both values shown |
| `assert_ne!(a, b)` | not equal |
| `panic!("msg")` | force fail with message |
| Custom predicate | `assert!(s.contains("foo"))` |

`assert_eq!` prints both sides on failure, which is hugely useful:

```
---- counter::tests::class_threshold_catches stdout ----
thread 'tests::class_threshold_catches' panicked at 'assertion `left == right` failed
  left: 1
 right: 0', src/counter.rs:108
```

You instantly see what the actual value was.

## Tests that return `Result`

If your test's `?` operator would be useful, return a `Result`:

```rust
#[test]
fn loads_minimal_valid() -> anyhow::Result<()> {
    let (_d, p) = write_config(r#"{ "log_dir": "/tmp", "logs": [] }"#)?;
    let cfg = load(&p)?;
    assert_eq!(cfg.log_dir.as_os_str(), "/tmp");
    Ok(())
}
```

The test fails if any `?` returns `Err`.

## Integration tests (separate from unit tests)

Files in a top-level `tests/` directory get compiled as **separate binaries**, each one importing your crate as if it were an external dependency. Useful for end-to-end behaviour:

```
my-crate/
├── src/
│   └── lib.rs
└── tests/
    ├── integration_smoke.rs
    └── another_scenario.rs
```

```rust
// tests/integration_smoke.rs
use my_crate::*;

#[test]
fn full_round_trip() {
    let c = Counter::new();
    // …
}
```

Integration tests only see the **public** API of your crate (no `pub(crate)` items). They prove your public surface actually works.

We don't have any integration tests in nginx-monitor yet — every module is unit-tested in place and the binary is mostly orchestration. If we add one, it'd live at `crates/nginx-monitor/tests/`.

## Doctests (bonus)

Code blocks in doc comments are run as tests:

```rust
/// Doubles a number.
///
/// ```
/// assert_eq!(my_crate::double(2), 4);
/// ```
pub fn double(x: u32) -> u32 { x * 2 }
```

`cargo test` runs these too. They keep documentation honest — the example can't drift from real behaviour.

## See also

- [[09-error-handling-result-anyhow|Tests that return Result use ? cleanly]]
- [[17-case-study-nginx-monitor|Each module in our codebase has its own #[cfg(test)] block]]
