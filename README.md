# Seismic Foundry Fork DB

This repository contains Seismic's fork of foundry-fork-db

The upstream repository lives [here](https://github.com/foundry-rs/foundry-fork-db). This fork is up-to-date with it through commit `7747569`. You can see this by viewing the [main](https://github.com/SeismicSystems/seismic-foundry-fork-db/tree/main) branch on this repository

You can view all of our changes vs. upstream on this [pull request](https://github.com/SeismicSystems/seismic-foundry-fork-db/pull/6). The sole purpose of this PR is display our diff; it will never be merged in to the main branch of this repo

## Main Changes

This repository was forked to support Seismic's [modifications](https://github.com/SeismicSystems/seismic-revm) to [revm](https://github.com/bluealloy/revm) as part of [seismic-foundry](https://github.com/SeismicSystems/seismic-foundry).

### Enhanced State Tree Representation

Seismic REVM introduces a new representation for the state tree. Instead of using raw `U256` values, it employs a [`FlaggedStorage`](https://github.com/SeismicSystems/seismic-revm/blob/39b4dea21beda3d9a693023f69c2f6b8a940d29e/crates/primitives/src/state.rs#L174) struct. This struct attaches a boolean flag to each `U256` value, indicating whether it is associated with a shielded type.

For more details, please refer to Seismic REVM's [README](https://github.com/SeismicSystems/seismic-revm/blob/seismic/README.md#flagged-storage).

## Structure

Seismic's forks of the [reth](https://github.com/paradigmxyz/reth) stack all have the same branch structure:
- `main` or `master`: this branch only consists of commits from the upstream repository. However it will rarely be up-to-date with upstream. The latest commit from this branch reflects how recently Seismic has merged in upstream commits to the seismic branch
- `seismic`: the default and production branch for these repositories. This includes all Seismic-specific code essential to make our network run

#### License

<sup>
Licensed under either of <a href="LICENSE-APACHE">Apache License, Version
2.0</a> or <a href="LICENSE-MIT">MIT license</a> at your option.
</sup>

<br>

<sub>
Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in these crates by you, as defined in the Apache-2.0 license,
shall be dual licensed as above, without any additional terms or conditions.
</sub>
