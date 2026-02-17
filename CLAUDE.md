# Seismic Foundry Fork DB

Fork of [foundry-fork-db](https://github.com/foundry-rs/foundry-fork-db) that adds **FlaggedStorage** support for Seismic's confidential storage layer. Used as a dependency in [seismic-foundry](https://github.com/SeismicSystems/seismic-foundry). Upstream is tracked through the `main` branch.

This document focuses on Seismic-specific changes. For upstream documentation, see the `main` branch or the [upstream repo](https://github.com/foundry-rs/foundry-fork-db). For cross-repo Seismic context, see the [workspace CLAUDE.md](../CLAUDE.md).

## What This Does

Foundry's fork-db provides smart caching and deduplication for forking providers — when Foundry forks a live chain, this library handles fetching, caching, and deduplicating RPC requests for account state, storage, blocks, and transactions.

Seismic's fork replaces raw `U256` storage values with `FlaggedStorage` (from [seismic-revm](https://github.com/SeismicSystems/seismic-revm)), a struct that attaches a boolean flag to each storage value indicating whether it belongs to a shielded type. Storage is fetched via the custom `eth_getFlaggedStorageAt` RPC method. A `PrivateStorage` error variant prevents unauthorized reads of shielded slots.

## Key Seismic Modifications

All changes center on replacing `U256` with `FlaggedStorage` in the storage pipeline:

- **`backend.rs`** — `StorageFuture` returns `FlaggedStorage`; `request_account_storage()` calls `eth_getFlaggedStorageAt` RPC instead of `eth_getStorageAt`; `StorageSender` sends `FlaggedStorage`; `DatabaseRef::storage_ref()` returns `FlaggedStorage`
- **`cache.rs`** — `StorageInfo = HashMap<U256, FlaggedStorage>`; serialization/deserialization handles the flagged type; `MemDb` storage maps use `FlaggedStorage`
- **`error.rs`** — `PrivateStorage(Address, U256)` variant for unauthorized shielded slot access
- **`Cargo.toml`** — Metadata updated (authors, homepage, repository); `[patch.crates-io]` redirects crates to Seismic forks; `seismic-prelude` added as dependency
- **`test-data/storage.json`** — Storage values changed from raw hex strings to `FlaggedStorage` objects (`{ "value": "0x...", "is_private": false }`)

### Dependency Patches

The `[patch.crates-io]` section is critical — it redirects these crates to Seismic forks:

| Patch group        | Crates                                             | Purpose                                           |
| ------------------ | -------------------------------------------------- | ------------------------------------------------- |
| seismic-alloy-core | `alloy-primitives`, `alloy-dyn-abi`, `alloy-sol-*` | Adds `FlaggedStorage` to alloy primitives         |
| seismic-revm       | `revm`, `seismic-revm`                             | Modified EVM with shielded storage opcodes        |
| seismic-trie       | `alloy-trie`                                       | Trie support for flagged storage                  |
| seismic-alloy      | `seismic-prelude`                                  | Network types (`AnyNetwork`, `AnyRpcBlock`, etc.) |
| enclave            | `seismic-enclave`                                  | Enclave support                                   |

## Build

Rust library crate using Cargo. MSRV is **1.88**.

**Important:** `Cargo.lock` is gitignored (library convention), so dependency resolution happens fresh each build and can pull in breaking versions. If the build fails due to dependency resolution, check for version incompatibilities and pin the offending crate as needed.

```bash
cargo build
```

## Test

```bash
cargo test

# With all features
cargo test --all-features
```

All tests are in-process and require no external services (no RPC, no Anvil). Test data lives in `test-data/storage.json`.

### Lint & Format

```bash
# Clippy (may have known warnings from FlaggedStorage conversions)
cargo clippy --all --all-targets --all-features

# Format check (requires nightly)
cargo +nightly fmt --all --check
```

## Project Layout

```
src/
  lib.rs           Library entry point — exports BackendHandler, SharedBackend, BlockchainDb, DatabaseError
  backend.rs       Core logic — BackendHandler, SharedBackend, provider request futures
  cache.rs         Cache layer — BlockchainDb, JsonBlockCacheDB, MemDb, serialization
  error.rs         Error types — DatabaseError enum with PrivateStorage variant
test-data/
  storage.json     Serialized cache state for unit tests
scripts/
  changelog.sh     Git-cliff changelog generation
  check_no_std.sh  no_std compatibility check
```

## Code Style

See `rustfmt.toml`. Key rules:

- Max **100 chars** per line
- `imports_granularity = "Crate"` (group imports by crate)
- `use_small_heuristics = "Max"` (prefer single-line formatting)
- Format requires **nightly** rustfmt
- Clippy MSRV set in `clippy.toml` to match `rust-version`
- Lint policy: `unused_must_use` = deny, `rust_2018_idioms` = deny

## CI

Only `.github/workflows/seismic.yml` runs on the `seismic` branch. Other workflow files (e.g. `ci.yml`) are inherited from upstream and do not trigger on our branch.

`seismic.yml` runs on push/PR to `seismic`:

| Job        | What it does                          | Timeout |
| ---------- | ------------------------------------- | ------- |
| `rustfmt`  | `cargo +nightly fmt --all --check`    | 5 min   |
| `build`    | `cargo build`                         | 30 min  |
| `warnings` | `RUSTFLAGS="-D warnings" cargo check` | 30 min  |
| `test`     | `cargo test`                          | 30 min  |

## Branches

- `seismic` — main branch (PR target)
- `main` — upstream-only mirror (reflects last upstream merge point)

## Troubleshooting

| Problem                                                | Fix                                                                                                                                    |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| Build fails after fresh dependency resolution          | `Cargo.lock` is gitignored, so versions resolve fresh. Check error for version conflicts and pin the offending crate with `cargo update <crate> --precise <version>`. |
| Build slow on first run                                | ~200 deps including seismic-revm. Subsequent builds use incremental compilation.                                                       |
| `failed to get account for ...` errors at runtime      | If using a non-archive node, the forking provider can't access historical state. Use an archive RPC endpoint.                          |
