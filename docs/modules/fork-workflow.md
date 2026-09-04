# Fork Workflow Module

> **Living document.** Read this before modifying the module. Update it in the same change whenever the module's behavior, endpoints, files, or dependencies change.

**Source:** repo root (`test.js`, `package.json`, `README.md`, `.github/workflows/`) · **Last verified:** 2026-09-04

## Purpose

This is a fork of `DefiLlama/DefiLlama-Adapters` kept so PaltaLabs can develop and test the DeFindex and Soroswap TVL adapters before sending them upstream. This doc covers the parts of the repo shared by every adapter: the local runner, the export contract it enforces, CI, and how the fork sits relative to upstream. It does not cover the ~6,100 third-party adapter entries under `projects/` (5,883 directories + 203 loose `.js` files, counted 2026-09-04).

## Structure

| File | Purpose |
|---|---|
| `test.js` | The local runner and validator. `node test.js <adapter file> [timestamp]`. |
| `package.json` | Scripts and the dependency allowlist. Adapters may not add packages. |
| `README.md` | Upstream contributor guide. Source of truth for the run command and the adapter rules. |
| `projects/helper/` | Shared helpers (`http.js`, `chain/stellar.js`, `chains.json`, `whitelistedExportKeys.json`). |
| `.github/workflows/test.yml` | PR CI: runs changed adapters through `test.js` (`:59-71`), then ESLint (`:73-75`). |
| `.github/workflows/build-modules.yml` | On push to `main`, builds the tvlModules artifact. |
| `.github/workflows/alert.yml` | On push to `main`, pings a DefiLlama refresh endpoint. |

## Public surface: the adapter export contract

An adapter is a CommonJS module whose chain-named keys hold the TVL functions. `test.js` enforces the rules:

- Chain keys must exist in `projects/helper/chains.json` (checked at `test.js:302`); `stellar` is at `projects/helper/chains.json:408`.
- Chain keys must match `/^[a-z0-9_]+$/` (`test.js:294`).
- Every other export key must be on the whitelist in `projects/helper/whitelistedExportKeys.json` (checked at `test.js:307`). The whitelist is `tvl`, `staking`, `methodology`, `pool2`, `misrepresentedTokens`, `timetravel`, `borrowed`, `start`, `doublecounted`, `hallmarks`, `isHeavyProtocol`, `deadFrom`, `ownTokens`, `vesting` (`projects/helper/whitelistedExportKeys.json:2-15`).
- `tvl`, `staking`, `pool2`, `borrowed`, `treasury`, `offers`, `vesting` must live inside a chain key, not at the root (`test.js:291`, enforced at `test.js:336`).
- The module directory name must start with a lowercase letter unless `LLAMA_RUN_LOCAL` is set (`test.js:129-134`).

## Key methods

- **`getTvl(...)`** (`test.js:43`) - constructs `new sdk.ChainApi({ chain, block, timestamp, storedKey })` (`test.js:54`), calls the adapter's `tvl` function with it, then prices whatever the adapter accumulated. It throws if the result is not a number (`test.js:66-73`), which is what turns a broken adapter into a failing CI run.
- **`computeTVL(balances, timestamp)`** (`test.js:392`) - the pricing pipeline. `fixBalances` (`test.js:373`) rejects a literal `usd` balance key and tells you to use `api.addUSDValue()` instead (`test.js:377`), then keys are normalised (`test.js:400`) and priced.
- **`buildPricesGetQueries(readKeys, timestamp)`** (`test.js:520`) - builds the price lookup URLs against `https://coins.llama.fi/prices/current/` or `.../prices/historical/<timestamp>/` (`test.js:523-524`). Historical pricing kicks in only when the requested timestamp is more than 30 minutes old (`test.js:524`).
- **`normalizeAddress(address, chain, extractChain)`** (`projects/helper/tokenMapping.js:153`) - lowercases token keys **except** for case-sensitive chains, and `stellar` is on that list (`projects/helper/tokenMapping.js:26`). This is why Stellar asset identifiers must be passed through with exact casing.

## Dependencies

- `@defillama/sdk` at `latest` (`package.json:28`) supplies `ChainApi`, `api.add`, `api.addUSDValue`, and the cache layer. It is not pinned, so behaviour can shift under you without a repo change.
- `axios` (`package.json:32`) and `dotenv` (`package.json:41`).
- CI runs Node 20 with pnpm 10 (`.github/workflows/test.yml:19-29`).
- Runtime network dependencies for our two adapters: `api.defindex.io`, `api.stellar.expert`, and `coins.llama.fi`. See the per-adapter docs.

## Fork and upstream

- The only configured remote is `origin` = `https://github.com/defindex-io/DeFiLlama-Adapters` (verified with `git remote -v`). **There is no `upstream` remote configured in this clone.** To sync you must add `DefiLlama/DefiLlama-Adapters` as a remote yourself, or use the GitHub fork sync button.
- `main` is upstream history with exactly two fork-only commits on top: `509be44ad` (adds `projects/defindex/index.js` with timetravel) and `6cba39893` (adds the try/catch and the URL constant). Everything below `78de9e881` is upstream.
- **The fork is far behind upstream.** Its base commit `78de9e881` is dated 2026-02-03; upstream has moved on since. Any claim about "what upstream has" must be checked against `github.com/DefiLlama/DefiLlama-Adapters`, not against this tree.
- **Upstream now ships its own `projects/defindex/index.js`, and it is not ours.** Verified 2026-09-04: upstream's version uses `getConfig` + `callSoroban(vault, 'fetch_total_managed_funds')` against vaults from `api.defindex.io/vault/discover?network=mainnet`, with no `timetravel` and no `start`. It is the module DefiLlama actually runs. The lineage is visible on `origin/feat/defindex-adapter`: its early commits (`f033fc356`, `a980faea4`, `8ff80dc26`, 2026-02-04) built that discover-based adapter, which upstream took and evolved; commit `db1e578e0` then rewrote that branch to the `/tvl` API approach `main` now carries. So our `main` is a **replacement** for a live upstream adapter, not a new one. See [defindex-adapter.md](defindex-adapter.md) for the TVL discrepancy this creates.
- Two feature branches exist on `origin`:
  - `origin/feat/defindex-adapter` - contains `projects/defindex/index.js` on a different base than `main`.
  - `origin/feat/defindex-timetravel` - one commit ahead of `main` (`90a75ce42`), adding a `getWithRetry` wrapper with exponential backoff around the DeFindex API call. Not merged.
- `.github/workflows/alert.yml:13` still curls `https://born-to-llama.herokuapp.com/refresh` on push to `main`. That is upstream's deployment hook and is meaningless for the fork; expect it to fail or no-op here.
- **No automated upstream sync exists in this repo.** No workflow, script, or config performs it. Syncing is manual.

## Gotchas & invariants

- **Never touch `package.json` or `package-lock.json` in an adapter PR.** CI hard-fails and comments on the PR (`.github/workflows/test.yml:51-57`), and `README.md:14` repeats the package-lock rule. Separately, `README.md:47` forbids adding npm packages at all, so an adapter cannot pull in a project-specific Soroswap or Stellar SDK; use plain HTTP.
- **CI only runs changed adapters.** `.github/workflows/test.yml:38-42` derives the file list from the PR diff, so an adapter you did not touch is never exercised. A green PR is not evidence that our other adapter still works.
- **Running an adapter always hits the network.** There is no offline or fixture mode. `node test.js` calls the protocol's API and the DefiLlama price API for real.
- **Do not edit third-party adapters** under `projects/` to make something pass. They are upstream's.
- **`npm run build`** is `node scripts/buildImports.js` (`package.json:10`) and writes `scripts/tvlModules.json`, which is gitignored (`.gitignore`, last line). It is not needed to run a single adapter.

## Testing

Commands, all verified in source:

```bash
node test.js projects/defindex/index.js               # run one adapter, current timestamp   (README.md:29)
node test.js projects/defindex/index.js 1729080692    # run at a unix timestamp              (README.md:31)
node test.js projects/defindex/index.js 2024-10-16    # or a YYYY-MM-DD date                 (README.md:33)
npm run lint                                          # eslint -c eslint.config.js .         (package.json:11)
npm run test-interactive                              # node utils/testInteractive           (package.json:13)
```

`npm test` is **not** wired up: it is `echo "Error: no test specified" && exit 1` (`package.json:7`). There is no unit test suite in this repo; `test.js` is a live integration runner, not a test framework.
