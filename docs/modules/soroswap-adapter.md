# Soroswap Adapter Module

> **Living document.** Read this before modifying the module. Update it in the same change whenever the module's behavior, endpoints, files, or dependencies change.

**Source:** `projects/soroswap/` · **Last verified:** 2026-09-04

## Purpose

Reports Soroswap AMM TVL on Stellar to DefiLlama. It walks a hardcoded list of 25 Soroswap pool contract IDs, asks stellar.expert for each contract's USD value, and sums them (`projects/soroswap/index.js:31-36`). Blast radius is the Soroswap protocol page on defillama.com. Because the pool list is a literal in the file, a new Soroswap pool contributes nothing to reported TVL until someone edits this file.

## Structure

| File | Purpose |
|---|---|
| `projects/soroswap/index.js` | The whole adapter: the `pools` list, the `tvl` function, and the module export. Single file, 40 lines. |

## Public surface

CommonJS module export (`projects/soroswap/index.js:38-40`):

| Export key | Value | Meaning |
|---|---|---|
| `stellar` | `{ tvl }` | Chain key, so TVL is attributed to the `stellar` chain (`projects/helper/chains.json:408`). |

That is the entire export. There is no `methodology`, no `start`, and no `timetravel`, so DefiLlama treats this adapter as current-value only.

## Key methods

- **`tvl(api)`** (`projects/soroswap/index.js:31`) - sequential `for` loop over `pools`. For each pool it GETs `https://api.stellar.expert/explorer/public/contract/<contractId>/value` and calls `api.addUSDValue(data.total / 1e7)` (`projects/soroswap/index.js:33-34`).
  - The `/1e7` is Stellar's 7 decimal fixed point scaling. The same divisor appears in the shared Stellar helper when reading a contract value from the same endpoint (`projects/helper/chain/stellar.js:17-21`).
  - `api.addUSDValue` adds a raw USD amount rather than a token balance, so this adapter bypasses the DefiLlama pricing pipeline entirely. The harness names this method explicitly as the sanctioned way to add USD (`test.js:377`); it is provided by `@defillama/sdk` (`package.json:28`), not by this repo.

- **`pools`** (`projects/soroswap/index.js:3-29`) - 25 Stellar mainnet contract IDs, each with a trailing comment naming the pair (for example `XLM-USDC` at `projects/soroswap/index.js:9`). This is the only place Soroswap deployment addresses live in this repo.

## Dependencies

- **`axios`** directly (`projects/soroswap/index.js:1`), declared at `package.json:32`. Note this adapter does not use the repo's shared `get` helper (`projects/helper/http.js:27`) the way the DeFindex adapter does.
- **stellar.expert public API**, `https://api.stellar.expert/explorer/public/contract/<id>/value` (`projects/soroswap/index.js:33`). Third-party, unauthenticated, no key in this repo.
- **Soroswap pool contracts on Stellar mainnet**, 25 addresses hardcoded at `projects/soroswap/index.js:4-28`.
- **DefiLlama `ChainApi`** from `@defillama/sdk`, instantiated by the harness (`test.js:54`).

## Gotchas & invariants

- **The pool list is manual and stale by construction.** Nothing discovers pools from a Soroswap factory contract. Every new pool needs a PR editing `projects/soroswap/index.js:3-29`. Treat reported TVL as a lower bound.
- **No `misrepresentedTokens` flag.** DefiLlama uses that flag to mark TVL that is a USD number rather than priced token balances. The other Stellar adapter in this tree that uses `api.addUSDValue` does set it (`projects/stellarx/index.js:10-12`); this one does not (`projects/soroswap/index.js:38-40`). The local harness does not enforce it, so this is a listing-accuracy gap rather than a test failure. Confirm with upstream before adding it, since it changes how the protocol is displayed.
- **No error handling.** Unlike the DeFindex adapter, a single failing stellar.expert request throws and aborts the whole run. That is arguably the better default (a loud failure beats a silent zero) but it means one bad pool ID breaks all 25.
- **Sequential requests.** The loop is `await` inside `for` (`projects/soroswap/index.js:32-35`), so 25 round trips run back to back. Fine at this size; if the pool list grows a lot, batch rather than parallelise blindly, stellar.expert rate limits.
- **This adapter is upstream code, not fork-local.** Its two commits (`ec3a77acf` "Soroswap TVL Adapter (#10735)", `22546b257` "Fix: Soroswap (#14060)") came in through upstream DefiLlama PRs and are present in `origin/main` only because the fork tracks upstream. Changes here belong in a PR to `DefiLlama/DefiLlama-Adapters`, not in a fork-only commit. See [fork-workflow.md](fork-workflow.md).

## Testing

No unit test. Verify by running the shared harness (`README.md:28-34`):

```bash
node test.js projects/soroswap/index.js
```

Passing a timestamp is pointless here: the adapter has no `timetravel` export and stellar.expert `/value` returns current state only. The run needs network access. Lint with `npm run lint` (`package.json:11`).
