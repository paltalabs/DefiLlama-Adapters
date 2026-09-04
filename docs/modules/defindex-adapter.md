# DeFindex Adapter Module

> **Living document.** Read this before modifying the module. Update it in the same change whenever the module's behavior, endpoints, files, or dependencies change.

**Source:** `projects/defindex/` · **Last verified:** 2026-09-04

## Purpose

Reports DeFindex vault TVL on Stellar to DefiLlama. It is a thin client: it does not read chain state itself, it asks the DeFindex API for an already aggregated asset balance map and hands it to the DefiLlama pricing pipeline (`projects/defindex/index.js:6-16`). Blast radius is the DeFindex protocol page on defillama.com. If the shape of `GET /tvl` changes or the endpoint goes down, DefiLlama silently reports zero TVL for DeFindex (see Gotchas).

**This adapter is not the one DefiLlama runs today.** Upstream ships its own `projects/defindex/index.js` that reads the vaults on-chain; ours is an unmerged replacement for it. Read the first gotcha before doing anything with this file.

## Structure

| File | Purpose |
|---|---|
| `projects/defindex/index.js` | The whole adapter: constants, the `tvl` function, and the module export. Single file, 23 lines. |

## Public surface

The adapter is a CommonJS module. Its export object is the contract DefiLlama consumes (`projects/defindex/index.js:18-23`):

| Export key | Value | Meaning |
|---|---|---|
| `timetravel` | `true` | The adapter can be called at an arbitrary past timestamp. |
| `start` | `1748518054` (`DAY_ONE`, 2025-05-29 11:27:34 UTC) | First timestamp DefiLlama should backfill from. |
| `methodology` | string | Shown on the DefiLlama protocol page. |
| `stellar` | `{ tvl }` | Chain key. Every balance the adapter adds is attributed to the `stellar` chain. |

All four keys are on the whitelist the local test harness enforces: `methodology` (`projects/helper/whitelistedExportKeys.json:4`), `timetravel` (`:7`), `start` (`:9`), `tvl` (`:2`). `stellar` is a known chain (`projects/helper/chains.json:408`).

## Key methods

- **`tvl(api)`** (`projects/defindex/index.js:6`) - builds the request URL, fetches it, and adds every returned asset balance to the DefiLlama `ChainApi` accumulator.
  - URL selection (`projects/defindex/index.js:7`): if `api.timestamp` is set it appends `?timestamp=<unix>`, otherwise it hits the bare `/tvl`. `api.timestamp` is always populated by the harness (`test.js:54` constructs the `ChainApi` with a `timestamp`), so in practice the timestamped branch is the one that runs locally.
  - Accumulation (`projects/defindex/index.js:10-12`): iterates `Object.entries(data.tvl)` and calls `api.add(assetId, amount)` once per asset. The adapter therefore delegates decimals, pricing, and USD conversion entirely to DefiLlama's `computeTVL` (`test.js:392`), which resolves each key against `https://coins.llama.fi/prices/...` (`test.js:520-524`).
  - Error handling (`projects/defindex/index.js:13-15`): the whole fetch plus loop is wrapped in `try/catch` that only `console.error`s. This exists because the shared `get()` helper throws on any non-2xx (`projects/helper/http.js:36-39`) and an unhandled throw would fail the whole DefiLlama run. The cost is silent zeros; see Gotchas.

## Dependencies

- **`get` from `projects/helper/http.js`** (`projects/defindex/index.js:1`, defined at `projects/helper/http.js:27`). Plain axios GET, no retry, throws `Failed to get <endpoint>` on any error.
- **DeFindex API, `https://api.defindex.io`** (`projects/defindex/index.js:3`). Endpoints used: `GET /tvl` and `GET /tvl?timestamp=<unix seconds>` (`projects/defindex/index.js:7`). Expected response shape: an object with a `tvl` property whose entries are `assetId -> amount` (`projects/defindex/index.js:10`).
- **DefiLlama `ChainApi`** from `@defillama/sdk` (`package.json:28`), instantiated by the harness at `test.js:54`. `api.add` and `api.timestamp` come from there.
- No env vars. No RPC. No contract addresses are hardcoded in this adapter.

## Gotchas & invariants

- **Failures are invisible.** The `catch` at `projects/defindex/index.js:13` swallows every error and returns normally, so `api` stays empty and the run reports TVL 0 instead of erroring. A rate limit, a DNS blip, or a 500 from `api.defindex.io` looks exactly like "DeFindex has no TVL". When debugging a chart gap, check the console for `Error fetching DeFindex TVL:` before suspecting the protocol.
- **No retry.** `get()` does not retry (`projects/helper/http.js:27-40`). A retry wrapper exists on the unmerged branch `origin/feat/defindex-timetravel` (commit `90a75ce42`, `getWithRetry`, 4 attempts) but is **not** on `main`. Do not re-derive it, rebase that branch. Note its commit message advertises "1s/2s/4s/8s" backoff but the loop sleeps `1000 * 2**attempt` only for attempts 0-2, so the real backoff is **1s/2s/4s** and the 8s wait never happens.
- **`network=mainnet` is not sent.** The commit that introduced the adapter (`509be44ad`) documents the call as `GET /tvl?timestamp=<unix>&network=mainnet`, but the code only sends `timestamp` (`projects/defindex/index.js:7`). Resolved 2026-09-04 by querying the live API: `GET /tvl` and `GET /tvl?network=mainnet` return byte-identical bodies, so the API defaults to mainnet and the missing parameter is harmless. Do not add it back to silence the discrepancy with the commit message.
- **Asset key format is set by the API, not by this repo.** `api.add(assetId, ...)` passes the key straight through, and DefiLlama only prices keys it recognises. Verified 2026-09-04 against the live API: the keys are **Stellar SAC contract IDs** (uppercase `C…`, 56 chars), *not* the `CODE-ISSUER-N` form the sibling Stellar helper uses (`projects/helper/chain/stellar.js:18`). `coins.llama.fi` prices them at 7 decimals and resolves them correctly (XLM, USDC, EURC, CETES all returned `confidence: 0.99`). Stellar is in the case-sensitive chain list (`projects/helper/tokenMapping.js:26`), so keys reach the price API with their original casing — but the price API itself answered identically for the uppercase and lowercased forms when tested, so casing is **not** currently a failure mode. Amounts arrive as JSON **strings**, which `api.add` accepts.
- **Upstream already has a DeFindex adapter, and ours contradicts it.** `DefiLlama/DefiLlama-Adapters@main` ships a `projects/defindex/index.js` that discovers vaults from `api.defindex.io/vault/discover?network=mainnet` and reads each one on-chain with `callSoroban(vault, 'fetch_total_managed_funds')`. That is the module DefiLlama runs (`api.llama.fi/protocol/defindex` reports `module: defindex/index.js`), and on 2026-09-04 it reported **$20.05M** across 14 mainnet vaults. Feeding the same day's `GET /tvl` response through DefiLlama's own prices yields **~$174.3M** — an 8.7x jump, essentially all of it one USDC entry of 173,998,863 units. Merging our version as-is would put a step change of that size on the public chart. Why the API's `/tvl` disagrees with on-chain managed funds is owned by `defindex-api`, not by this repo, and is _TBD, unverified_ here. **Reconcile the two numbers before opening an upstream PR.**
- **`start` is a hard floor.** `DAY_ONE = 1748518054` (`projects/defindex/index.js:4`) is DeFindex mainnet day one. Lowering it makes DefiLlama request timestamps the API cannot reconstruct.
- **Do not add npm packages** to support this adapter. Upstream rejects project-specific dependencies (`README.md:47`) and CI fails the PR if `package.json` is touched (`.github/workflows/test.yml:52-57`).

## Testing

There is no unit test. Verification is running the adapter through the shared harness (`README.md:28-34`):

```bash
node test.js projects/defindex/index.js            # current TVL
node test.js projects/defindex/index.js 2025-06-01 # historical, exercises timetravel
```

The harness validates export keys and chain names before running (`test.js:154`, `test.js:278-307`) and throws if the resulting TVL is not a number (`test.js:66-73`). Both invocations hit the live DeFindex API and the live DefiLlama pricing API, so they need network access. Lint with `npm run lint` (`package.json:11`).
