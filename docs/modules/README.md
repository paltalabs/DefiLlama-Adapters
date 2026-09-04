# Module Documentation Index

Living docs, one per in-scope module. **Read the relevant doc before modifying a module; update it in the same change.** See the "Module Documentation Convention" section in `CLAUDE.md` for the workflow.

<!--
Router index. Keep it small: one row per module, no prose bodies.
Scope note: this repo is a fork of DefiLlama/DefiLlama-Adapters and carries 7000+
third-party adapters under projects/. Those are NOT documented here and must not be.
Only PaltaLabs owned adapters (DeFindex, Soroswap) and the fork workflow get docs.
-->

| Doc | Module | One-liner |
|---|---|---|
| [defindex-adapter.md](defindex-adapter.md) | `projects/defindex/` | DeFindex vault TVL on Stellar, read from the DeFindex API with historical timetravel support. |
| [soroswap-adapter.md](soroswap-adapter.md) | `projects/soroswap/` | Soroswap AMM TVL on Stellar, summed from a hardcoded list of pool contracts via stellar.expert. |
| [fork-workflow.md](fork-workflow.md) | repo root (`test.js`, `package.json`, `.github/workflows/`) | How this fork relates to upstream DefiLlama, and how to run and lint a single adapter locally. |

## Out of scope

Everything else under `projects/` is upstream third-party adapter code. Do not document it, do not scan it wholesale, and do not edit it unless upstream asked for a fix. Other Stellar adapters in the tree (`projects/stellar-dex/`, `projects/stellarx/`, `projects/StellarisFinance/`) are third-party and are not PaltaLabs work.
