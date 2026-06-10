# The newtype pattern (and why types matter)

Rust's type system lets you wrap a value in a struct just to give it a stronger meaning. This is the **newtype pattern**. It costs nothing at runtime — the wrapper compiles to the same machine code as the inner value — but enforces invariants and prevents bugs.

## The problem newtype solves

Bad:

```rust
fn block_ip(ip: String, ttl_minutes: u32) { … }

block_ip(ttl_minutes.to_string(), ip);   // compiles. argument order is silently swapped.
```

Better:

```rust
pub struct Ip(pub Ipv4Addr);
pub struct TtlMinutes(pub u32);

fn block_ip(ip: Ip, ttl: TtlMinutes) { … }

block_ip(TtlMinutes(60), Ip(addr));      // compile error — wrong types
```

The compiler now refuses to mix them up. In our project we use `std::net::Ipv4Addr` directly (it's stronger than `String`), and `u32` for ttl — but the principle is the same.

## A real newtype in `monitor-core`: `Whitelist`

```rust
pub struct Whitelist {
    set: HashSet<Ipv4Addr>,       // private field
}

impl Whitelist {
    pub fn from_raw(raw: &serde_json::Value) -> Result<Self> {
        let mut set = HashSet::new();
        let obj = match raw {
            serde_json::Value::Object(m) => m,
            serde_json::Value::Null      => return Ok(Self { set }),
            other => bail!("whitelist must be an object, got {}", type_name(other)),
        };
        for (k, _v) in obj {
            if k.starts_with('_') { continue; }                  // skip _comment keys
            let ip: Ipv4Addr = k.parse()
                .map_err(|_| anyhow!("whitelist non-IPv4 key: {k}"))?;
            set.insert(ip);
        }
        Ok(Self { set })
    }

    pub fn contains(&self, ip: &Ipv4Addr) -> bool {
        self.set.contains(ip)
    }
}
```

The whole point: the *only* way to build a `Whitelist` is `from_raw`, which:

- Filters out keys starting with `_` (comments).
- Parses each remaining key as IPv4 — fails on anything else.

A future change that wants to construct a `Whitelist` differently would have to add a new constructor method, which forces the author to think about whether the invariant holds. They can't just `let wl: Whitelist = HashSet::from([...]);` because `set` is private.

That's `pub struct { private field }` doing real work: **encapsulation**.

## `Thresholds` and `Offender` are the same shape

```rust
pub struct Thresholds(pub BTreeMap<String, u32>);

impl Thresholds {
    pub fn get(&self, key: &str) -> Option<u32> { … }
    pub fn any(&self) -> u32 { … }
    pub fn validate(&self, ctx: &str) -> Result<()> { … }
}
```

`pub BTreeMap<…>` field — fully exposed in this case, but the type alias gives it a *name* (`Thresholds`) and a place to attach methods. `Thresholds::validate(...)` is more discoverable than a free function `validate_thresholds(...)`.

## Why `BTreeMap` not `HashMap` here

A type choice that's meaningful, not stylistic:

| | `HashMap<K, V>` | `BTreeMap<K, V>` |
|---|---|---|
| Lookup speed | O(1) average | O(log n) |
| Iteration order | **Unspecified** (random each run) | **Sorted by key**, deterministic |
| Memory | A bit more | A bit less |

When we build the "triggered by: …" line:

```
triggered by:  any >= 50, 4xx >= 15, 404 >= 8
```

We want the order **stable across runs** so operators can recognise patterns at a glance. `BTreeMap` gives that for free; `HashMap` would shuffle each cron tick. Same data, different ergonomics.

## `#[derive(Default)]` and constructors

```rust
#[derive(Default)]
pub struct BlockState {
    pub blocks: Vec<Block>,
}
```

`Default` gives you `BlockState::default()`, which builds the "zero value" — empty `Vec`. Useful when:

- Constructing for the first time (`let mut state = BlockState::default();`)
- Hand-built tests
- The `Result` of `load()` when the state file doesn't exist

Most simple structs derive `Default`. Anything with a complex invariant (like `Whitelist`) shouldn't — that's what custom constructors are for.

## Enums for closed sets of variants

```rust
pub enum LogKind {
    Access,
    Error,
}
```

An enum lists the *exact* set of possible values. The compiler knows:

```rust
match log.kind {
    LogKind::Access => check_access(log),
    LogKind::Error  => check_error(log),
    // no need for a default; compiler verifies exhaustiveness
}
```

If you add `LogKind::Json` later, every `match` on `LogKind` becomes a compile error until you handle the new case. That's the compiler telling you exactly where to update code. No "I forgot one branch" bug.

Compare to `String`-based "types":

```rust
// bad
match log.kind.as_str() {
    "access" => check_access(log),
    "error"  => check_error(log),
    _        => panic!("unknown kind: {}", log.kind),    // runtime crash on typo
}
```

Enums are how Rust expresses "here are all the variants, please handle each."

## When to reach for newtype

| | Newtype it |
|---|---|
| Two values of the same primitive type with different meanings (ip vs ttl, seconds vs minutes) | Yes — prevents argument-swap bugs |
| A collection that must be constructed via validation | Yes — wrap with private field |
| A value with a domain-specific method set | Yes — gives it a home for `impl` |
| A short-lived helper variable | No — overkill |
| A direct re-export of a stdlib type | No — type alias (`type MyMap = HashMap<…>`) is enough |

## See also

- [[10-traits-the-blocker-pattern|Traits — another type-system tool for the same kind of safety]]
- [[09-error-handling-result-anyhow|Validation errors are returned via Result]]
- [[17-case-study-nginx-monitor|Where Whitelist, Thresholds, LogKind appear]]
