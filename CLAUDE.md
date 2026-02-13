# Seismic Foundry Fork DB

Fork of [foundry-fork-db](https://github.com/foundry-rs/foundry-fork-db) that adds **FlaggedStorage** support for Seismic's confidential storage layer. Used as a dependency in [seismic-foundry](https://github.com/SeismicSystems/seismic-foundry). Upstream is tracked through the `main` branch.

## What This Does

Foundry's fork-db provides smart caching and deduplication for forking providers — when Foundry forks a live chain, this library handles fetching, caching, and deduplicating RPC requests for account state, storage, blocks, and transactions.

Seismic's fork replaces raw `U256` storage values with `FlaggedStorage` (from [seismic-revm](https://github.com/SeismicSystems/seismic-revm)), a struct that attaches a boolean flag to each storage value indicating whether it belongs to a shielded type. Storage is fetched via the custom `eth_getFlaggedStorageAt` RPC method. A `PrivateStorage` error variant prevents unauthorized reads of shielded slots.

## Build

Rust library crate using Cargo. MSRV is **1.88**. `Cargo.lock` is gitignored (library convention).

**Important:** Because `Cargo.lock` is not committed, dependency resolution happens fresh each build. A known incompatibility between `alloy-tx-macros` (resolves to latest 1.x) and `alloy-consensus` (resolves to 1.0.x) can cause build failures. After generating the lock file, pin the macro crate:

```bash
cargo update alloy-tx-macros --precise 1.0.41
```

### macOS

```bash
# Prerequisites: Rust 1.88+ (rustup install 1.88)
# No system dependencies beyond Rust toolchain

# Build
cargo build

# If you hit the alloy-tx-macros / TransactionEnvelope error:
cargo update alloy-tx-macros --precise 1.0.41
cargo build
```

### Linux (Ubuntu)

```bash
# Prerequisites
sudo apt-get update
sudo apt-get install -y build-essential pkg-config libssl-dev
# Rust 1.88+ (rustup install 1.88)

# Build (requires git CLI for patched deps)
CARGO_NET_GIT_FETCH_WITH_CLI=true cargo build
```

### Verify

```bash
cargo build
# Should end with: Finished `dev` profile ... target(s) in ...
```

## Test

```bash
# All 11 unit tests
cargo test

# With all features
cargo test --all-features
```

All tests are in-process and require no external services (no RPC, no Anvil). Test data lives in `test-data/storage.json`.

### Lint & Format

```bash
# Clippy (2 known warnings: useless FlaggedStorage conversions in backend.rs)
cargo clippy --all --all-targets --all-features

# Format check (requires nightly)
cargo +nightly fmt --all --check
```

## Project Layout

```
src/
  lib.rs           Library entry point — exports BackendHandler, SharedBackend, BlockchainDb, DatabaseError
  backend.rs       Core logic (1431 lines) — BackendHandler, SharedBackend, provider request futures
  cache.rs         Cache layer (681 lines) — BlockchainDb, JsonBlockCacheDB, MemDb, serialization
  error.rs         Error types (81 lines) — DatabaseError enum with PrivateStorage variant
test-data/
  storage.json     Serialized cache state for unit tests
scripts/
  changelog.sh     Git-cliff changelog generation
  check_no_std.sh  no_std compatibility check
```

## Key Seismic Modifications

All changes center on replacing `U256` with `FlaggedStorage` in the storage pipeline:

- **`backend.rs`** — `StorageFuture` returns `FlaggedStorage`; `request_account_storage()` calls `eth_getFlaggedStorageAt` RPC; `StorageSender` sends `FlaggedStorage`
- **`cache.rs`** — `StorageInfo = HashMap<U256, FlaggedStorage>`; serialization/deserialization handles the flagged type; `MemDb` storage maps use `FlaggedStorage`
- **`error.rs`** — `PrivateStorage(Address, U256)` variant for unauthorized shielded slot access
- **`Cargo.toml` `[patch.crates-io]`** — Redirects `revm`, `alloy-primitives`, `alloy-trie`, and other crates to Seismic forks via git revisions

### Dependency Patches

The `[patch.crates-io]` section is critical — it redirects these crates to Seismic forks:

| Patch group        | Crates                                             | Purpose                                           |
| ------------------ | -------------------------------------------------- | ------------------------------------------------- |
| seismic-alloy-core | `alloy-primitives`, `alloy-dyn-abi`, `alloy-sol-*` | Adds `FlaggedStorage` to alloy primitives         |
| seismic-revm       | `revm`, `seismic-revm`                             | Modified EVM with shielded storage opcodes        |
| seismic-trie       | `alloy-trie`                                       | Trie support for flagged storage                  |
| seismic-alloy      | `seismic-prelude`                                  | Network types (`AnyNetwork`, `AnyRpcBlock`, etc.) |
| enclave            | `seismic-enclave`                                  | Enclave support                                   |

## Code Style

See `rustfmt.toml`. Key rules:

- Max **100 chars** per line
- `imports_granularity = "Crate"` (group imports by crate)
- `use_small_heuristics = "Max"` (prefer single-line formatting)
- Format requires **nightly** rustfmt
- Clippy MSRV set in `clippy.toml` to match `rust-version`
- Lint policy: `unused_must_use` = deny, `rust_2018_idioms` = deny

## CI

GitHub Actions (`.github/workflows/seismic.yml`) — runs on push/PR to `seismic`:

| Job        | What it does                          | Timeout |
| ---------- | ------------------------------------- | ------- |
| `rustfmt`  | `cargo +nightly fmt --all --check`    | 5 min   |
| `build`    | `cargo build`                         | 30 min  |
| `warnings` | `RUSTFLAGS="-D warnings" cargo check` | 30 min  |
| `test`     | `cargo test`                          | 30 min  |

CI sets `CARGO_NET_GIT_FETCH_WITH_CLI=true` for fetching patched git dependencies.

There is also an upstream-tracking `ci.yml` for the `main` branch with a full matrix (stable/nightly/MSRV × feature combinations), nextest, clippy, docs, and cargo-deny.

## Branches

- `seismic` — main branch (PR target)
- `main` — upstream-only mirror (reflects last upstream merge point)

## Troubleshooting

| Problem                                                                  | Fix                                                                                                                            |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| `method 'tx_type' is not a member of trait 'crate::TransactionEnvelope'` | `alloy-tx-macros` resolved too high. Run: `cargo update alloy-tx-macros --precise 1.0.41`                                      |
| `Cargo.lock` not found / fresh resolution pulls broken versions          | Expected — `Cargo.lock` is gitignored. Generate it with `cargo generate-lockfile`, then apply the `alloy-tx-macros` pin above. |
| `CARGO_NET_GIT_FETCH_WITH_CLI` errors on Linux CI                        | Set `CARGO_NET_GIT_FETCH_WITH_CLI=true` — required for git-based `[patch]` deps.                                               |
| Clippy warns about useless `FlaggedStorage` conversion                   | Known warnings in `backend.rs:584` and `backend.rs:1057`. Non-blocking.                                                        |
| Build slow on first run                                                  | ~200 deps including seismic-revm. Subsequent builds use incremental compilation.                                               |
| `failed to get account for ...` errors at runtime                        | If using a non-archive node, the forking provider can't access historical state. Use an archive RPC endpoint.                  |
