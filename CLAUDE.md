# DeFiLlama-Adapters (PaltaLabs fork)

## Entry summary

- Fork of `DefiLlama/DefiLlama-Adapters`. Origin is `https://github.com/defindex-io/DeFiLlama-Adapters` (`git remote -v`). It exists so PaltaLabs can write and test its own TVL adapters before sending them upstream.
- It carries ~6,100 third-party adapter entries under `projects/` (5,883 directories + 203 loose `.js` files, counted 2026-09-04). Only two are ours: `projects/defindex/index.js` and `projects/soroswap/index.js`. Ignore the rest.
- Exposes to other PaltaLabs repos: nothing importable, no package, no API. Its only output is DeFindex and Soroswap TVL appearing on defillama.com.
- Consumes from PaltaLabs: the DeFindex TVL API `https://api.defindex.io/tvl` (`projects/defindex/index.js:3`, `:7`) and 25 hardcoded Soroswap pool contract IDs on Stellar mainnet (`projects/soroswap/index.js:4-28`).
- Adapters are plain CommonJS modules exporting `{ stellar: { tvl } }`. No build step, no unit tests, no env vars needed for our two adapters.
- Run one adapter: `node test.js projects/defindex/index.js` (`README.md:29`). It hits live APIs.
- If you change the DeFindex API response shape, the adapter at `projects/defindex/index.js:10` breaks silently and reports 0.
- **Upstream already ships a different `projects/defindex/index.js`.** `DefiLlama/DefiLlama-Adapters@main` carries a Soroban on-chain version, and that is what defillama.com serves today. Our fork's API-based rewrite would *replace* it, not add to it, and the two disagree on TVL by ~8.7x. Read `docs/modules/defindex-adapter.md` before touching that adapter or opening an upstream PR.

## Cross-repo dependencies

Every PaltaLabs, DeFindex, or Soroswap service or contract our adapters call:

| This repo -> | What | Evidence |
|---|---|---|
| DeFindex API (`api.defindex.io`) | `GET /tvl` and `GET /tvl?timestamp=<unix>`, returns `{ tvl: { assetId: amount } }` | `projects/defindex/index.js:3`, `projects/defindex/index.js:7`, `projects/defindex/index.js:10` |
| Soroswap AMM pool contracts, Stellar mainnet | 25 hardcoded contract IDs, read for their USD value | `projects/soroswap/index.js:4-28`, `projects/soroswap/index.js:33` |

No PaltaLabs database, package, or repo is imported. The only other network dependencies are third party: `api.stellar.expert` (`projects/soroswap/index.js:33`) and DefiLlama's own price API `coins.llama.fi` (`test.js:523-524`).

The DeFindex API contract is owned outside this repo. Verified live against `api.defindex.io` on 2026-09-04: `GET /tvl` returns `assetId` keys that are **Stellar SAC contract IDs** (uppercase `C…`, e.g. `CCW67TSZ…MI75` = USDC) mapped to raw 7-decimal amounts encoded as **strings**, plus a sibling `timestamp` field. `?network=mainnet` is accepted but returns a byte-identical body, so the adapter omitting it is harmless.

## Scope rules for agents

- Do not document, refactor, or lint-fix third-party adapters under `projects/`. They belong to upstream.
- Never modify `package.json` or `package-lock.json` in an adapter change. CI hard-fails on either file (`.github/workflows/test.yml:51-57`); `README.md:14` states the package-lock rule.
- Adapters may not add npm packages (`README.md:47`). Use plain HTTP through `projects/helper/http.js` or `axios`.
- There is no `upstream` remote in this clone and no automated sync. Syncing with `DefiLlama/DefiLlama-Adapters` is manual.

## Commands

All verified in source. Nothing here installs or builds by default.

| Command | What it does | Source |
|---|---|---|
| `node test.js projects/defindex/index.js` | Run one adapter at the current timestamp | `README.md:29` |
| `node test.js projects/defindex/index.js 1729080692` | Run at a unix timestamp | `README.md:31` |
| `node test.js projects/defindex/index.js 2024-10-16` | Run at a `YYYY-MM-DD` date | `README.md:33` |
| `npm run lint` | `eslint -c eslint.config.js .` | `package.json:11` |
| `npm run test-interactive` | `node utils/testInteractive` | `package.json:13` |
| `npm run build` | `node scripts/buildImports.js`, writes the gitignored `scripts/tvlModules.json` | `package.json:10` |

`npm test` is not wired up: it is `echo "Error: no test specified" && exit 1` (`package.json:7`).

## Module Documentation Convention (MANDATORY)

Every in-scope module has a living doc at `docs/modules/<module>.md` (flat file, one per module). `docs/modules/README.md` is the index that routes a module's source path to its doc. These are the fast on-ramp for anyone, human or agent, touching a module.

**Progressive disclosure, do NOT load all docs at once.** When you are about to touch a module, open `docs/modules/README.md`, find the ONE doc matching the code you are changing, and read only that. Never pull the whole `docs/modules/` folder into context.

**The workflow rule:**
1. **Before modifying a module, read its `docs/modules/<module>.md` first.** It holds the file map, key methods with `file:line`, dependencies, and gotchas.
2. **After modifying a module, update its doc in the same change.** New or removed endpoints, changed behavior, new gotchas, dependency changes, all go into the doc before the work is done. Bump the "Last verified" date.
3. Doc claims must be verified against source and cite `file:line`. Never document something you have not confirmed exists. If you cannot verify it, write `_TBD, unverified_`.
4. **Adding a new module?** Create its `docs/modules/<module>.md` and add a row to `docs/modules/README.md` in the same change.

Scope exception for this repo: the convention applies only to PaltaLabs owned adapters and the shared fork tooling. Third-party adapters under `projects/` get no docs.

Docs follow a shared template: Purpose, Structure, Endpoints/Public surface, Key methods (`file:line`), Dependencies, Gotchas & invariants, Testing.
