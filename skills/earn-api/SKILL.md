---
name: earn-api
description: Quicknode Earn is a non-custodial USDC yield optimizer that rebalances USDC across curated Morpho ERC-4626 vaults on 7 EVM chains (Ethereum, Optimism, Unichain, Polygon, Monad, Base, Arbitrum) with CCTP V2 bridging. Use when programmatically interacting with the Quicknode Earn public API (earn-api.quicknode.dev, /v1) - walking a user through the strategy-creation wizard; creating, funding, editing, pausing, or closing strategies; generating deposit/withdraw/emergency-claim calldata; reading vault APYs and rankings, OpenCover cover capacity, the strategy leaderboard, and platform stats; wallet balances, approvals, prefs, ToS; strategy performance, history, bridges, or intents; or building SIWE-signed write requests.
---

# Quicknode Earn Public API (/v1)

HTTP API that drives the full Earn strategy lifecycle: browse vaults, create a strategy, fetch ready-to-sign approval and deposit/withdraw calldata, broadcast it yourself, and poll to completion. **The API never broadcasts transactions and never holds keys. It plans and encodes; the caller signs with the strategy owner's wallet and submits.**

Spec v1.15.0. 31 operations across 27 paths, all listed in the served `GET /v1/openapi.json`.

## Base URL and apikey

| Environment                                        | Base URL                                                    |
| -------------------------------------------------- | ----------------------------------------------------------- |
| Production (canonical)                             | `https://earn-api.quicknode.dev/functions/v1/api`           |

Full URL = base + `/v1/<endpoint>`, e.g. `https://earn-api.quicknode.dev/functions/v1/api/v1/config`.

Every request needs an `apikey` header: the Supabase publishable key, **public by design, NOT auth**:

- `sb_publishable_3xcdKa_uMRhK71Izd6BLdg_2vskXZ_h`

The spec declares it required; always send it. Enforcement is NOT guaranteed though: with `verify_jwt=false` the gateway currently serves requests without the header, so treat it as a convention you honor, not a gate you can rely on. If the platform ever does reject a missing key, that rejection happens upstream and does NOT use the `/v1` error envelope. Real auth is per-action SIWE on writes. There is deliberately no internal-service bypass header.

```bash
curl -s "https://earn-api.quicknode.dev/functions/v1/api/v1/config" \
  -H "apikey: sb_publishable_3xcdKa_uMRhK71Izd6BLdg_2vskXZ_h"
```

## RPC endpoints for broadcast

The API never broadcasts transactions. The caller must send signed transactions to a real RPC node. Use one public endpoint per chain below, or your own node.

| Chain ID | Name     | Public RPC                               |
| -------- | -------- | ---------------------------------------- |
| 1        | Ethereum | `https://ethereum-rpc.publicnode.com`    |
| 10       | Optimism | `https://mainnet.optimism.io`            |
| 130      | Unichain | `https://mainnet.unichain.org`           |
| 137      | Polygon  | `https://polygon-bor-rpc.publicnode.com` |
| 143      | Monad    | `https://rpc.monad.xyz`                  |
| 8453     | Base     | `https://mainnet.base.org`               |
| 42161    | Arbitrum | `https://arb1.arbitrum.io/rpc`           |

**Nonce warning**: when you send more than one transaction in a row from one wallet (for example, many vault approvals), do not let each call re-fetch the nonce from the RPC. Public RPC nodes can lag behind their own confirmed blocks. Fetch the nonce once, then increase it by one for each transaction you send. If a send fails with "replacement transaction underpriced," this is the cause.

## Conventions

- **Error envelope**: every error is `{ "error": { "code": "<stable machine code>", "message": "<human text>" } }`. Branch on `code`, never parse `message`. Success bodies are plain resources with no wrapper (some keep legacy wrapper keys like `{ strategy }`, `{ strategies, closedStrategies }`, `{ events }`, `{ transfers }`).
- **Units, read models** (vaults, strategies, history, performance, prefs): USDC values are decimal JSON numbers (`1234.56` = 1,234.56 USDC). APYs are percentage numbers, not fractions (`5.42` = 5.42%); realized APY is unbounded and may be negative. 2dp on browse/detail/series windows, 3dp on rankings and calldata plan APYs, 4dp on `intervalApy`.
- **Units, transaction templates** (`transactions[]`, `approvalsNeeded[]`, balances, allowances): base-unit decimal STRINGS. USDC is a 6-decimal integer string (`"1000000"` = 1 USDC), native gas is wei, shares/allowances are raw uint256. The 78-digit maxUint256 withdraw sentinel stays a string end to end; never round-trip these through a JS number.
- **One in-row exception**: `CctpTransfer.amount_usdc` is a base-unit string while its sibling `fee_usdc` is a decimal USDC number.
- **Addresses**: served lowercased, except approvals `token`/`vaultAddress`/`spender` (checksummed) and `CctpTransfer.user` (bytes32-padded lowercase). Vault keys are `"<chainId>:0x<lowercase address>"`.
- **Pagination**: none anywhere. Fixed caps with an additive `truncated` flag instead: performance 2000 rows (keeps NEWEST when capped), history 100 events / 50 transfers (`truncated` only in `format=entries`), bridges hard 50 with no flag, APY series 4000-row internal cap.
- **CORS**: `Access-Control-Allow-Origin: *`; allowed methods GET/POST/PUT/PATCH/DELETE/OPTIONS; bodies must be `Content-Type: application/json`.
- **HTTP client note**: the edge network in front of this API blocks some HTTP clients by their connection fingerprint, independent of any API error. Python's bare `urllib` gets a 403 with body `error code: 1010` even on a well-formed request. `curl` and Python's `requests` library both work; use one of those.
- **Stability**: `v1` is additive-only; new optional fields may appear. Never add a new key to a signed-payload `required` set (it would break third-party payload hashes).
- **Supported chains** (from `GET /v1/config`, presence gated on the chain's RPC secret; read it at runtime rather than hardcoding): 1 Ethereum, 10 Optimism, 130 Unichain, 137 Polygon, 143 Monad, 8453 Base, 42161 Arbitrum.

## Rate limits

Postgres fixed-window limiter. Falls back to a bounded in-memory window on limiter infra errors, fails CLOSED with 429 + `Retry-After: <seconds>` header (also in `Access-Control-Expose-Headers`) when over budget. Body: `{ "error": { "code": "rate_limited", "message": "Rate limit exceeded. Retry later." } }`.

Router-level per-IP buckets (IP = `cf-connecting-ip`, else the first `x-forwarded-for` entry, else `x-real-ip`; IPv6 is keyed on its /64):

| Bucket              | Limit  | Routes                                                                                                                                      |
| ------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `reads`             | 240/min| `/v1`, `/v1/openapi.json`, stats, `/v1/stats/fees`, `/v1/stats/strategies`, prices, config, history, bridges, performance, intents, prefs GET                       |
| `strategies_list`   | 240/min| `GET /v1/strategies`                                                                                                                        |
| `strategies_detail` | 240/min| `GET /v1/strategies/{id}`                                                                                                                   |
| `vaults`            | 120/min| every `/v1/vaults*` read (browse, `cover-capacity`, rankings, detail, apy) AND `GET /v1/strategies/leaderboard`                             |
| `balances`          | 120/min| balances                                                                                                                                    |
| `approvals`         | 120/min| approvals                                                                                                                                   |
| `auth_probe`        | 120/min| every SIWE mutation and calldata POST/PATCH/DELETE, before body parse or SIWE work |

In-handler buckets, applied AFTER SIWE recovery (or per strategy for calldata):

| Bucket      | Limit  | Key                           | Where                                                    |
| ----------- | ------ | ----------------------------- | -------------------------------------------------------- |
| `create`    | 40/min | `wallet:<recovered>\|ip:<ip>` | `POST /v1/strategies`                                    |
| `mutations` | 80/min | `wallet:<recovered>\|ip:<ip>` | strategy PATCH/DELETE, prefs PATCH, tos, opencover-terms |
| `calldata`  | 40/min | `strategy:<id>\|ip:<ip>`      | calldata deposit + withdraw + claim (one shared budget)  |

A SIWE write or calldata call therefore consumes from two buckets: `auth_probe` then its per-wallet/per-strategy bucket.

## SIWE-signed writes

Six operations require a per-action Sign-In-With-Ethereum proof **in the request body** (no auth headers). The recovered signer is the authoritative owner; any `wallet` field in the body is ignored. All reads and all three calldata endpoints are SIWE-free.

| Route                                     | Action (statement is `Authorize <action>`) | `- path:` resource in the proof                     |
| ----------------------------------------- | ------------------------------------------ | --------------------------------------------------- |
| `POST /v1/strategies`                     | `strategy.create`                          | `/strategies`                                       |
| `PATCH /v1/strategies/{id}`               | `strategy.update`                          | `/strategies/{id}` (id lowercased)                  |
| `DELETE /v1/strategies/{id}`              | `strategy.delete`                          | `/strategies/{id}` (id lowercased)                  |
| `PATCH /v1/wallets/{addr}/prefs`          | `prefs.update`                             | `/wallets/{addr}/prefs` (addr lowercased)           |
| `POST /v1/wallets/{addr}/tos`             | `tos_agreement`                            | `/wallets/{addr}/tos` (addr lowercased)             |
| `POST /v1/wallets/{addr}/opencover-terms` | `opencover_terms`                          | `/wallets/{addr}/opencover-terms` (addr lowercased) |

The resource is the `/v1`-relative path: no `/v1` prefix, no query, lowercased. The three wallet routes additionally require recovered wallet == path `{addr}` (else 403 `owner_mismatch`).

### Body shape

```json
{
  "siwe": { "message": "<the exact signed message>", "signature": "0x..." },
  "...payload fields...": "..."
}
```

Signed payload = the body minus `siwe`. An empty payload signs `{}`.

### Message template (exactly 14 lines, joined with `\n`)

```
<domain> wants you to sign in with your Ethereum account:
<address>

Authorize <action>

URI: <uri>
Version: 1
Chain ID: <chainId>
Nonce: <nonce>
Issued At: <ISO-8601>
Expiration Time: <ISO-8601>
Resources:
- sha256:<payloadHash>
- path:<resource>
```

Rules as enforced server-side:

- `<domain>` allowlist: `earn.quicknode.com`, `earn.quicknode.dev`, `earn-api.quicknode.dev`, `earn-frontend.vercel.app`, `localhost:3000`, `127.0.0.1:3000`. An agent should use `earn-api.quicknode.dev`. Wrong domain: 401 `siwe_domain`.
- The two blank lines shown in the template (one after the address, one after the statement) are mandatory. Statement must be exactly `Authorize <action>` (401 `siwe_action` otherwise).
- `URI:` and `Version:` prefixes are required but their VALUES are not validated; send `Version: 1` and any https origin.
- `Chain ID:` must parse to a positive integer (401 `siwe_chain` on non-finite or <= 0 values); a positive but UNSUPPORTED chain id fails later as a 500 `internal_error`, so always use a supported chain. Verification runs on that chain: for an EOA any supported chain works (plain ecrecover); for ERC-1271/Safe wallets it must be the chain where the contract wallet is deployed.
- `Nonce:` any string is accepted (even empty parses; use the client convention of 32 hex chars from 16 random bytes). **Single-use per wallet**: reuse returns 401 `siwe_replay` ("Signature already used; sign a fresh request"). The burn happens after all validation, immediately before the first DB write, so a fixable 400 never consumes a Safe user's collected signature. On create the burn happens after the idempotency read, so keyed byte-identical replays still return the stored 201.
- `Issued At:` at most 1h in the future; no maximum age. `Expiration Time:` strictly in the future, and `expiration - issuedAt` must be > 0 and <= 25h (401 `siwe_stale` on any temporal failure). Typical: 5-minute window for EOAs, up to 24h for Safe signature collection.
- `- sha256:<payloadHash>`: lowercase hex only (uppercase is rejected). Mismatch: 401 `siwe_payload`.
- `- path:<resource>`: required; missing or mismatched is 401 `siwe_payload`. Legacy 13-line proofs (no path line) are unconditionally rejected. Extra trailing `- ` Resources entries are tolerated.

### Payload canonicalization (mirror byte-for-byte or every write fails `siwe_payload`)

```
canonicalize(null | primitive) = JSON.stringify(value)
canonicalize(array)            = "[" + elements.map(canonicalize).join(",") + "]"
canonicalize(object)           = "{" + sortedKeys(dropping undefined values)
                                   .map(k => JSON.stringify(k) + ":" + canonicalize(v[k]))
                                   .join(",") + "}"
```

Keys sorted at every depth, `undefined` dropped. Empty payload canonicalizes to `"{}"` (its sha256 is `44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a`). Hash = lowercase SHA-256 hex of the UTF-8 canonical string. Neither wire-body key order nor numeric spelling matters on the wire (the server re-parses the JSON and canonicalizes it itself); the trap is YOUR canonicalizer: it must stringify values exactly as JavaScript `JSON.stringify` does (`1.0` must become `"1"`), or your hash will not match the server's.

### Signature

EIP-191 personal-message signature over the full message (`signMessage` in any wallet lib). Recovery order: EOA ecrecover, then ERC-1271 `isValidSignature` for deployed contract wallets, then ERC-6492 unwrap for counterfactual wallets; a direct EIP-1271 fallback accepts `"0x"` as the signature for Safe pre-signed (SignMessageLib) flows. Mismatch: 401 `siwe_recover`. RPC failure during verification: 503 `siwe_rpc` (retryable, not a bad signature).

### Error order on a write

router `auth_probe` limit (429) -> JSON parse (400 `invalid_request`) -> siwe block shape (400 `siwe_missing`/`siwe_shape`) -> message parse (400 `siwe_parse`) -> domain (401) -> statement (401) -> timestamps (401 `siwe_stale`) -> payload hash (401 `siwe_payload`) -> path binding (401 `siwe_payload`) -> recovery (401 `siwe_recover` / 503 `siwe_rpc`) -> owner check (403 `owner_mismatch`, wallet lane only, BEFORE the bucket, so a mismatch never consumes it) -> per-wallet bucket (429) -> payload validation (400) -> nonce burn (401 `siwe_replay`) -> write.

### Worked example (delete, empty payload)

```
earn-api.quicknode.dev wants you to sign in with your Ethereum account:
0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B

Authorize strategy.delete

URI: https://earn-api.quicknode.dev
Version: 1
Chain ID: 8453
Nonce: 8f14e45fceea167a5a36dedd4bea2543
Issued At: 2026-07-28T12:00:00.000Z
Expiration Time: 2026-07-28T12:05:00.000Z
Resources:
- sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a
- path:/strategies/2b9d8f2e-1c34-4f0a-9a51-7e2d54c1b111
```

```bash
curl -s -X DELETE \
  "https://earn-api.quicknode.dev/functions/v1/api/v1/strategies/2b9d8f2e-1c34-4f0a-9a51-7e2d54c1b111" \
  -H "apikey: $APIKEY" -H "content-type: application/json" \
  -d '{"siwe":{"message":"<message above, \n-joined>","signature":"0x<sig>"}}'
```

## The wizard flow: guided create-and-fund

When a user asks to create or set up an Earn strategy, run this flow: interview, preview, confirm, act, deliver the URL. It is the agent-driven equivalent of the product's New Strategy wizard.

**Capability check first.** Driving this end to end requires (a) signing EIP-191 messages as the user's wallet (the SIWE proofs) and (b) signing + broadcasting transactions on the target chains. If the agent controls the key it does both itself; otherwise it prepares each message/transaction and hands it to the user's wallet to sign, waiting for the result before continuing.

### Step 1: interview

Ground the questions first with three reads: `GET /v1/config` (which chains are live, the Earn proxy), `GET /v1/wallets/{addr}/balances` (where the USDC and gas are), and `GET /v1/wallets/{addr}/prefs` (`has_agreement` pre-answers the ToS step, `opencover_terms_accepted` tells you whether the covered path still needs the terms signature). Then collect, one field at a time, with defaults offered:

| Ask the user                                  | Field                                     | Default / rule                                                                                                                                                                                                                                                                                                                                                                                                            |
| --------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Which chains should it run on?                | `chain_ids` (set `chain_id` to the first) | from config's live list; picking more than one unlocks the cross-network question                                                                                                                                                                                                                                                                                                                                         |
| How much USDC?                                | `capital_usdc`                            | must be <= the source-chain `usdcBalance`; create balance-checks the primary chain only                                                                                                                                                                                                                                                                                                                                   |
| Name?                                         | `name`                                    | 1-50 chars, `^[a-zA-Z0-9 _\-.]{1,50}$`; suggest a pattern like `Base-Arb USDC 5k`                                                                                                                                                                                                                                                                                                                                         |
| Max simultaneous vault positions?             | `max_positions`                           | REQUIRED, integer >= 1 (2-5 is typical)                                                                                                                                                                                                                                                                                                                                                                                   |
| Minimum vault TVL to consider?                | `min_tvl_usd`                             | suggest 50000; too-high values silently starve eligibility and surface later as `no_eligible_vaults`. This also sets how many approval transactions step 3 needs, one per eligible vault: on Base, a 50000 floor gave 27 eligible vaults, a 1000000 floor gave 18. Raise it for fewer approvals                                                                                                                           |
| Minimum withdrawable-liquidity floor?         | `min_liquidity_usd`                       | 0 = off (server default); a non-zero floor also drives liquidity-forced exits                                                                                                                                                                                                                                                                                                                                             |
| Entry liquidity multiplier?                   | `min_liquidity_entry_multiplier`          | server default 2 (a non-held vault needs multiplier x floor to be entered)                                                                                                                                                                                                                                                                                                                                                |
| Same-network APY-gap threshold (pct points)?  | `delta_pct`                               | hard floor 3, or 5 when the chain set includes Ethereum (1); omitted = the floor itself                                                                                                                                                                                                                                                                                                                                   |
| Cross-network threshold? (multi-chain only)   | `cross_chain_delta_pct`                   | 0 = inherit `delta_pct`; the EFFECTIVE bar (value when > 0, else `delta_pct`) must be >= 5, or 7 with Ethereum. NOTE: for a multi-chain set, omitting BOTH threshold fields is invalid when `delta_pct`'s floor is below the cross floor (inherit would fail the check); the compliant default is an explicit `cross_chain_delta_pct` at the cross floor (5, or 7 with Ethereum) while `delta_pct` stays at its own floor |
| Consecutive confirmations before a rebalance? | `delta_confirmations`                     | 12 (60 min) when omitted                                                                                                                                                                                                                                                                                                                                                                                                  |
| APY smoothing window (minutes)?               | `apy_smoothing_minutes`                   | 360; snaps to 5/10/30/60/120/240/360/720/1440                                                                                                                                                                                                                                                                                                                                                                             |
| Any vaults to exclude?                        | `hidden_vault_keys`                       | optional; show candidates from `GET /v1/vaults`; the eligible pool must keep >= `max_positions + 1` vaults or create 409s                                                                                                                                                                                                                                                                                                 |
| OpenCover coverage?                           | `covered`                                 | open to every wallet while `GET /v1/config` shows `coveredStrategiesEnabled: true`; needs the opencover-terms ack; pins the invest set to Base (`chain_ids` → "8453"); the funding chain (`chain_id`) is free and bridges via CCTP. Before you offer it, call `GET /v1/vaults/cover-capacity` and keep `capital_usdc` <= `remainingUsd` (the pool is shared and finite; `null` = unknown, not zero)                          |

Validate the threshold floors during the interview (rules above) so the user never hits a 400 after signing.

### Step 2: preview and confirm

`GET /v1/vaults/rankings` with THE SAME parameters the strategy will store (`capital`, `maxPositions`, `chains`, `minTvl`, `minLiquidity`, `minLiquidityMultiplier`, `apySmoothingMinutes`, `hiddenVaultKeys`, `wallet`). Show the user the selected vaults, `estimatedApy`, and any `excludedChains` reasons. Empty `vaults`? Loosen `min_tvl_usd`/`min_liquidity_usd` or add chains and re-preview. Get an explicit go-ahead before signing anything.

### Step 3: act

1. **ToS**: if the grounding prefs read showed `has_agreement: false`, sign + `POST .../tos` (SIWE `tos_agreement`), binding `"terms_url": "https://earn.quicknode.com/terms"` in the signed payload (the URL the product signs). Covered strategies: `POST .../opencover-terms` too.
2. **Pre-approve the FULL ELIGIBLE POOL now, BEFORE creating the strategy** — not just the Step 2 preview's selected rows. Approvals only need the wallet address, not a strategy id, so there is no reason to wait for create; doing this first also means the 1-hour `pending_setup` GC clock (step 3 below) only starts once every approval is already confirmed, instead of ticking during a long or retried approval batch. `GET /v1/vaults/rankings` caps its response at `maxPositions` rows, so it under-represents the pool; enumerate the pool instead with `GET /v1/vaults?chains=<chain_ids>&minTvl=<min_tvl_usd>&minLiquidity=<min_liquidity_usd>`, then locally drop any vault with `apy` null/<=0, any key in `hidden_vault_keys`, and (for a non-zero `min_liquidity_usd`) any vault whose `availableLiquidityUsd` is below `min_liquidity_usd * min_liquidity_entry_multiplier`. This mirrors what the product's own wizard approves (the full eligible set, not the top-N it happens to enter first). Then call `GET /v1/wallets/{addr}/approvals` with `chains`, `requiredUsdc` (base units, = `capital_usdc`), and `vaults` set to every one of those keys. Sign + broadcast the `usdc.tx` (exact capital, to the proxy) and every `vaults[].tx` (share tokens, maxUint256, to the proxy) from the owner wallet, one at a time with an explicit, incrementing nonce (see RPC endpoints above) — do not rely on the RPC to track a fast sequence of nonces for you. **This step is not optional even though it looks redundant with step 5 below**: deposit-mode vault selection (and later, rebalance-time entry selection) runs an on-chain check of each candidate's real share-token allowance to the proxy and silently DROPS anything unapproved before the plan is even built. A wallet with zero prior approvals gets `400 no_eligible_vaults` ("No vaults available with APY data") on its very first `calldata/deposit` call, not a helpful `approvalsNeeded` list — that field only ever reports the USDC leg and any vault selection ALREADY survives to the plan stage. Approving only the top-N selected vaults (instead of the whole eligible pool) reproduces this same failure the first time Auto-Pilot later wants to rotate into an eligible vault outside that initial set — approve the pool once, up front, so every future rebalance target is already covered.
3. **Create**: `POST /v1/strategies` (SIWE `strategy.create`) with the interview payload plus an `idempotencyKey`. Save `strategy.id`. The 1-hour funding clock starts now.
4. **Deposit calldata**: `POST /v1/strategies/{id}/calldata/deposit` with `{}`.
5. **Approvals (recovery only)**: if `approvalsNeeded` is still non-empty here (selection drifted since the Step 2 preview, or a partial step-2 broadcast), sign + broadcast every remaining `approvalsNeeded[].tx` on its `chainId`, wait for inclusion, then re-call deposit calldata (same `intentId`, now with a real `gasHint`).
6. **Deposit**: sign + broadcast `transactions[0]` FROM THE OWNER WALLET, `data` verbatim, gas limit = `gasHint`.
7. **Confirm**: poll `GET .../intents/{intentId}` until `status: "fulfilled"` (branch on `status`, never `fulfilled_at`), then `GET /v1/strategies/{id}` until `status: "active"`; multi-chain, also wait for `pending_bridges_count` to reach 0 and the `/bridges` legs to hit `confirmed`. A 5-10 second cadence sits comfortably inside the read buckets (240/min); the detail read is cached ~12s server-side anyway.

### Step 4: deliver the URL

Report what was created (vaults entered, estimated APY) and ALWAYS end with the strategy's live dashboard URL:

```
https://earn.quicknode.com/strategy/<strategy.id>
```

(`<strategy.id>` is the uuid from the create response; the page is the product's strategy dashboard.) Remind the user that rebalancing is autonomous from here; no further signatures are needed until they edit or exit.

**Failure branches**: `no_eligible_vaults` at deposit time, on the FIRST call = you skipped step 3 (pre-approve the preview's candidate vaults' share tokens to the proxy, not just USDC), then re-call deposit; on a LATER call = candidate set moved and needs new share-token approvals, or genuinely loosen filters via PATCH; insufficient-balance 400 at create = fund the wallet or lower `capital_usdc`; 403 `covered_disabled` = covered strategies are switched off, offer standard; 403 `covered_terms_required` = the wallet has not signed the OpenCover terms ack; 503 `cover_pool_exhausted` at deposit calldata = the shared cover pool is smaller than `capital_usdc`, so read `GET /v1/vaults/cover-capacity`, lower `capital_usdc` via PATCH to at most `remainingUsd`, then re-call (pre-check this before create); funding stalled past the hour = the row was GC'd, re-create (the idempotency key will NOT resurrect it; use a fresh key).

## Strategy lifecycle, end to end

The platform plans allocations and rebalances autonomously once funded; your wallet signs only approvals, the deposit, and per-chain withdrawals.

**1. Discover.** `GET /v1/config` (chains + Earn proxy address), `GET /v1/wallets/{addr}/balances` (where the USDC and gas are), `GET /v1/vaults` (browse) and `GET /v1/vaults/rankings` (preview: the SAME pipeline that plans real deposits). You never hand-pick vaults; you steer selection through strategy config (`chain_ids`, `max_positions`, `min_tvl_usd`, `min_liquidity_usd`, `hidden_vault_keys`).

**2. Prerequisites.** `GET /v1/wallets/{addr}/prefs` for `has_agreement`; if false, `POST /v1/wallets/{addr}/tos` (SIWE). Note: ToS is recorded consent, not server-enforced by create. Covered strategies additionally need `POST /v1/wallets/{addr}/opencover-terms` (SIWE); there is no wallet allowlist. Approvals can be pre-read via `GET /v1/wallets/{addr}/approvals`, but the practical shortcut is step 4's `approvalsNeeded`.

**3. Create.** `POST /v1/strategies` (SIWE `strategy.create`). Result: `status "pending_setup"`, `active true`, `first_deposit_at null`. Invisible to Auto-Pilot until funded. **CRITICAL: a never-funded `pending_setup` strategy is garbage-collected about 1 hour after `updated_at`.** Every successful deposit-calldata build bumps `updated_at` (a lease). Fund within an hour of the last calldata call or the row vanishes (reads then 404). Multisigs must regenerate calldata at least hourly while collecting signatures.

**4. Fund.** `POST /v1/strategies/{id}/calldata/deposit` (public, empty body). Strategy must be `pending_setup` (top-ups are not supported). If `approvalsNeeded` is non-empty (`gasHint` null, `gasHintReason "approvals_required"`): sign and broadcast each approval `tx`, then re-call deposit (same `intentId` returns, now with a real `gasHint`). Then sign and broadcast `transactions[0]` FROM THE STRATEGY OWNER WALLET (`selfBatchDeposit` pulls USDC from `msg.sender`), submitting `data` byte-for-byte (attribution matches `keccak256(calldata)` against the server-written intent). Confirm: poll `GET /v1/strategies/{id}/intents/{intentId}` for `status "fulfilled"` and `GET /v1/strategies/{id}` until `status "active"` and `positions[]` fills. Same-chain positions appear on deposit confirmation; cross-chain legs ride CCTP (watch `pending_bridges_count` and `/bridges`), normal latency minutes. Stuck deposit leg (>= 30 min, relayer never landed it): `POST .../calldata/claim` with the `transferId` from `/bridges`; 409 `attestation_pending` + `Retry-After: 60` while Circle finalizes; minted USDC goes to the owner's wallet, bypassing the vault.

**5. Monitor.** `GET /v1/strategies?wallet=` (list + rollups + `cycle_state`), `GET /v1/strategies/{id}` (detail + the rest of the autopilot telemetry: `pending_rebalance`, per-position `current_apy`), `/history` (events, or `?format=entries` accordion view), `/bridges` (CCTP legs), `/performance` (yield time series). `cycle_state "action_needed"` means the autopilot is blocked (e.g. no eligible replacement vault); fix by PATCHing filters. Rebalances need no user signatures.

**6. Edit.** `PATCH /v1/strategies/{id}` (SIWE `strategy.update`): thresholds, floors, name, capital, `status` (`active`/`paused` pause toggle only), hide-list ops. Config changes, pause included, take effect on the executor's next tick (minutes; the executor loads only `status='active'` rows). Threshold floors are validated only when a threshold field is in the payload (pre-floor strategies stay grandfathered until touched).

**7. Exit.** Always ask `POST /v1/strategies/{id}/calldata/withdraw` FIRST; use `DELETE` only when withdraw returns `fallback: "delete"` (or the strategy was never funded). Withdraw returns one tx per chain, source chain first: **sign and submit in array order** (the finalizer keys off it). Withdrawal bridge legs mint USDC directly to your wallet, no claim step; note the emergency claim endpoint rejects withdrawal legs, so there is NO API escape hatch for a stalled withdrawal leg (that is a platform-side relayer incident, not something the caller can unstick). Cross-chain closes pass through `status "closing"` before `"closed"`; poll detail. There is no partial-amount withdraw, only whole positions per chain (`chainIds` subset). Paused positions are skipped by the plan: those ERC-4626 shares must be redeemed directly against the vault, outside this API. Never DELETE while real on-chain positions exist (it only mutates the DB; shares would orphan).

**Status machine**: `pending_setup -> active` (first deposit) -> [`active <-> paused` via PATCH] -> `active|paused -> closing` (cross-chain close in flight) -> `closed`; or `active|paused -> closed` directly (same-chain close / DELETE-close). A FUNDED strategy still sitting in `pending_setup` (deposit confirmed on-chain but the status flip not yet processed) DELETE-closes rather than hard-deletes, so `pending_setup -> closed` also exists. `pending_setup -> deleted` (DELETE pre-deposit, or the 1-hour GC). `closed` is terminal (`active` false); backward transitions are illegal. `active` is a coarse boolean (true for every non-closed status, including paused); `status` is the lifecycle.

---

## Worked examples

### 1. Bash + curl: rank vaults, create a strategy, fetch deposit calldata

```bash
API="https://earn-api.quicknode.dev/functions/v1/api"
APIKEY="sb_publishable_3xcdKa_uMRhK71Izd6BLdg_2vskXZ_h"
```

**Step 1 — rank vaults for a 1,000 USDC deposit on Base, up to 3 positions:**

```bash
curl -s "$API/v1/vaults/rankings?capital=1000&maxPositions=3&chains=8453&minTvl=50000" \
  -H "apikey: $APIKEY"
```

Returns `{ "mode": "ranked", "vaults": [...], "excludedChains": [...], "eligibleCount": <n>, "estimatedApy": <n|null> }`.

**Step 2 — build the signed payload for `POST /v1/strategies`.**

The SIWE proof authorizes action `strategy.create` against resource `/strategies`. The payload is the request body minus the `siwe` block. Per the canonicalization rule, object keys are sorted alphabetically at every depth before hashing (the JSON you actually POST can be in any key order — only the hash you compute locally has to follow this rule):

```bash
PAYLOAD='{"capital_usdc":1000,"chain_ids":[8453],"delta_pct":3,"max_positions":3,"min_tvl_usd":50000,"name":"Base USDC 1k"}'
```

curl can't hash on its own, so shell out to Python or OpenSSL for the sha256 (lowercase hex):

```bash
# Option A: python3
python3 -c "import hashlib,sys; print(hashlib.sha256(sys.argv[1].encode()).hexdigest())" "$PAYLOAD"

# Option B: openssl
printf '%s' "$PAYLOAD" | openssl dgst -sha256 -r | cut -d' ' -f1
```

Both print:

```
8cf69fed5eb021a1532f328d68e81aaf12dbcb3e3f8b93fd26daef8ec920d9ae
```

**Step 3 — build the SIWE message** (exact 14-line template, domain `earn-api.quicknode.dev`, resource `/strategies`):

```
earn-api.quicknode.dev wants you to sign in with your Ethereum account:
0xYourWalletAddressHere

Authorize strategy.create

URI: https://earn-api.quicknode.dev
Version: 1
Chain ID: 8453
Nonce: 7edd4dbda6a24ae30c539cbc36c5d142
Issued At: 2026-07-28T12:00:00.000Z
Expiration Time: 2026-07-28T12:05:00.000Z
Resources:
- sha256:8cf69fed5eb021a1532f328d68e81aaf12dbcb3e3f8b93fd26daef8ec920d9ae
- path:/strategies
```

Sign it with the owner wallet's key using EIP-191 personal-message signing (`personal_sign` / `signMessage` — curl alone can't sign; see the Python script below for a full signer). Then POST, with the `siwe` block plus the same payload fields used above:

```bash
curl -s -X POST "$API/v1/strategies" \
  -H "apikey: $APIKEY" -H "content-type: application/json" \
  -d '{
    "siwe": {
      "message": "earn-api.quicknode.dev wants you to sign in with your Ethereum account:\n0xYourWalletAddressHere\n\nAuthorize strategy.create\n\nURI: https://earn-api.quicknode.dev\nVersion: 1\nChain ID: 8453\nNonce: 7edd4dbda6a24ae30c539cbc36c5d142\nIssued At: 2026-07-28T12:00:00.000Z\nExpiration Time: 2026-07-28T12:05:00.000Z\nResources:\n- sha256:8cf69fed5eb021a1532f328d68e81aaf12dbcb3e3f8b93fd26daef8ec920d9ae\n- path:/strategies",
      "signature": "0x<signature-from-personal_sign>"
    },
    "capital_usdc": 1000,
    "chain_ids": [8453],
    "delta_pct": 3,
    "max_positions": 3,
    "min_tvl_usd": 50000,
    "name": "Base USDC 1k"
  }'
```

201 response: `{ "strategy": { "id": "<uuid>", "status": "pending_setup", ... } }`. Save `strategy.id`.

**Step 4 — fetch deposit calldata** (public, no SIWE, empty body):

```bash
STRATEGY_ID="<id from the 201 response>"

curl -s -X POST "$API/v1/strategies/$STRATEGY_ID/calldata/deposit" \
  -H "apikey: $APIKEY" -H "content-type: application/json" \
  -d '{}'
```

Returns `{ "kind": "deposit", "transactions": [<one selfBatchDeposit tx>], "approvalsNeeded": [...], "plan": {...}, "expiresAt": "..." }`. If `approvalsNeeded` is non-empty, sign and broadcast each listed `tx` first (USDC + share-token approvals to the Earn proxy), then re-call this same endpoint before signing `transactions[0]`.

### 2. Python: end-to-end scripted strategy creation

Requires `pip install eth-account requests`. This script signs with a real private key and calls the write endpoints, but **it never signs or broadcasts any on-chain transaction** — it only prints the transactions the caller still has to sign and send.

```python
#!/usr/bin/env python3
"""
Create an Earn strategy end to end via the Quicknode Earn public API,
then fetch the deposit calldata.

This script SIGNS SIWE MESSAGES (EIP-191 personal-message signatures) with
the loaded private key, and it CALLS write endpoints on the caller's
behalf. It does NOT sign or broadcast any on-chain transaction — the
approval and deposit transactions it prints at the end must still be
signed and sent by the strategy owner's wallet, through whatever
on-chain signing path that wallet normally uses.

WARNING: WALLET_PRIVATE_KEY must be a developer or test wallet key that
you (the human operator) are deliberately handing to this script. Never
point this at a key an agent obtained, generated, or discovered on its
own, and never load it from anywhere an agent could have written to.
"""

import hashlib
import json
import os
import secrets
import sys
import uuid
from datetime import datetime, timedelta, timezone

import requests
from eth_account import Account
from eth_account.messages import encode_defunct

API_BASE = "https://earn-api.quicknode.dev/functions/v1/api"
APIKEY = "sb_publishable_3xcdKa_uMRhK71Izd6BLdg_2vskXZ_h"
DOMAIN = "earn-api.quicknode.dev"
URI = f"https://{DOMAIN}"
CHAIN_ID = 8453  # Base

HEADERS = {"apikey": APIKEY, "content-type": "application/json"}


# --- payload canonicalization (mirrors the API's rule byte-for-byte) --------

def _js_number(n):
    # JSON.stringify spells whole-valued floats without a trailing ".0"
    # (e.g. 1.0 -> "1"). All numeric fields used below are plain ints, so
    # this only matters if you extend the payload with float fields.
    if isinstance(n, int):
        return str(n)
    if float(n).is_integer():
        return str(int(n))
    return repr(float(n))


def canonicalize(value):
    if value is None:
        return "null"
    if isinstance(value, bool):
        return "true" if value else "false"
    if isinstance(value, (int, float)):
        return _js_number(value)
    if isinstance(value, str):
        return json.dumps(value)
    if isinstance(value, list):
        return "[" + ",".join(canonicalize(v) for v in value) + "]"
    if isinstance(value, dict):
        # sort keys at every depth; there is no Python "undefined" to drop,
        # so simply never put optional/absent fields into the dict
        items = sorted(value.items())
        return "{" + ",".join(
            json.dumps(k) + ":" + canonicalize(v) for k, v in items
        ) + "}"
    raise TypeError(f"cannot canonicalize value of type {type(value)!r}")


def payload_hash(payload: dict) -> str:
    canonical = canonicalize(payload)
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()


# --- SIWE message ------------------------------------------------------------

def build_siwe_message(address, action, resource, payload, window_seconds=300):
    now = datetime.now(timezone.utc)
    issued_at = now.strftime("%Y-%m-%dT%H:%M:%S.") + f"{now.microsecond // 1000:03d}Z"
    expiry = now + timedelta(seconds=window_seconds)
    expiration = expiry.strftime("%Y-%m-%dT%H:%M:%S.") + f"{expiry.microsecond // 1000:03d}Z"
    nonce = secrets.token_hex(16)  # 32 hex chars, single-use per wallet
    h = payload_hash(payload)

    message = (
        f"{DOMAIN} wants you to sign in with your Ethereum account:\n"
        f"{address}\n"
        f"\n"
        f"Authorize {action}\n"
        f"\n"
        f"URI: {URI}\n"
        f"Version: 1\n"
        f"Chain ID: {CHAIN_ID}\n"
        f"Nonce: {nonce}\n"
        f"Issued At: {issued_at}\n"
        f"Expiration Time: {expiration}\n"
        f"Resources:\n"
        f"- sha256:{h}\n"
        f"- path:{resource}"
    )
    return message


def sign_siwe(account, message):
    signable = encode_defunct(text=message)
    signed = Account.sign_message(signable, private_key=account.key)
    signature = signed.signature.hex()
    if not signature.startswith("0x"):
        signature = "0x" + signature
    return signature


# --- API helper ---------------------------------------------------------------

def api_call(method, path, body):
    resp = requests.request(method, f"{API_BASE}{path}", headers=HEADERS, json=body)
    data = resp.json()
    if isinstance(data, dict) and "error" in data:
        err = data["error"]
        raise RuntimeError(f"{method} {path} -> {err['code']}: {err['message']}")
    return data


def main():
    # Load the wallet. See the module docstring: this MUST be a developer
    # or test wallet key you are explicitly trusting this script with.
    private_key = os.environ["WALLET_PRIVATE_KEY"]
    account = Account.from_key(private_key)
    address = account.address
    print(f"Signing as {address}")

    # 1. The signed payload for POST /v1/strategies (body minus "siwe").
    payload = {
        "name": "Base USDC 1k",
        "capital_usdc": 1000,
        "max_positions": 3,
        "chain_ids": [8453],
        "delta_pct": 3,
        "min_tvl_usd": 50000,
        "idempotencyKey": str(uuid.uuid4()),
    }

    message = build_siwe_message(
        address=address,
        action="strategy.create",
        resource="/strategies",
        payload=payload,
    )
    signature = sign_siwe(account, message)

    # 2. Create the strategy.
    create_body = {"siwe": {"message": message, "signature": signature}, **payload}
    created = api_call("POST", "/v1/strategies", create_body)
    strategy = created["strategy"]
    strategy_id = strategy["id"]
    print(f"Created strategy {strategy_id} (status: {strategy['status']})")

    # 3. Fetch deposit calldata. Public endpoint, no SIWE required.
    calldata = api_call("POST", f"/v1/strategies/{strategy_id}/calldata/deposit", {})

    # 4. Print what still needs a real signature and broadcast. This
    #    script stops here — it does not sign or send anything on-chain.
    print("\n=== ACTION REQUIRED ===")
    print(f"Sign and broadcast the following from {address}."
          " This script has NOT done so.\n")

    approvals = calldata.get("approvalsNeeded") or []
    if approvals:
        print("Approvals (sign/broadcast these FIRST, then re-call")
        print("POST /v1/strategies/{id}/calldata/deposit before the deposit tx):")
        for a in approvals:
            print(f"  chainId={a['chainId']} token={a['token']} "
                  f"spender={a['spender']} requiredAllowance={a['requiredAllowance']}")
            print(f"    tx.to={a['tx']['to']}")
            print(f"    tx.data={a['tx']['data']}")
            print(f"    tx.value={a['tx']['value']}")
    else:
        print("No pending approvals.")

    print("\nDeposit transaction:")
    for tx in calldata["transactions"]:
        print(f"  chainId={tx['chainId']} kind={tx['kind']} "
              f"intentId={tx['intentId']}")
        print(f"    to={tx['to']}")
        print(f"    data={tx['data']}")
        print(f"    value={tx['value']}")
        print(f"    gasHint={tx['gasHint']}  (use this as the gas limit)")

    print(f"\nexpiresAt={calldata['expiresAt']}  (regenerate calldata if you miss this window)")
    print(f"Dashboard: https://earn.quicknode.com/strategy/{strategy_id}")


if __name__ == "__main__":
    main()
```

### 3. No-install alternative: Foundry's `cast`

If a Python environment with `pip install` is not available, any tool that does EIP-191 personal-message signing and raw transaction broadcast will work. Foundry's `cast` (https://getfoundry.sh) needs no extra packages:

```bash
# Sign a SIWE message (EIP-191 personal_sign)
cast wallet sign --private-key "$PRIVATE_KEY" "$(cat message.txt)"

# Broadcast a transaction with exact calldata, unmodified
cast send "$TO" "$DATA" --private-key "$PRIVATE_KEY" --rpc-url "$RPC"
```

`cast send` accepts a raw `0x`-prefixed calldata string in place of a function signature, and sends it byte-for-byte. This matters for the deposit transaction, where the API matches your transaction to its stored intent by hashing `data` exactly as sent.

To send many approvals faster, skip the wait between each one: add `--async` to print the transaction hash and return right away, and pass `--nonce <n>` yourself, increasing it by one each time (see the nonce warning under RPC endpoints above). Then check all the receipts once, at the end, instead of one at a time.

# Endpoint reference

## Discovery and platform

### GET /v1

API index derived at runtime from the served spec: `{ name, description?, version, openapi (absolute URL of the spec), endpoints: [{ method, path, summary? }] }`. Bucket `reads`.

### GET /v1/openapi.json

The served OpenAPI 3.1 contract for every endpoint in this file. Bucket `reads`.

### GET /v1/stats

Platform aggregates: `{ total_usdc (decimal dollars, best-effort live on-chain sum with DB fallback), active_strategies (integer, includes paused), total_rebalances (integer), updated_at }`. Never 500s on RPC failure (degrades to DB aggregate). ~2.5s in-isolate cache. Bucket `reads`.

### GET /v1/stats/fees

Platform-wide fees paid by users vs gas paid by the protocol, as a daily or hourly series: `{ from, to, granularity, totals: { fees_usdc, gas_usdc }, series: [{ bucket, fees_usdc, gas_usdc, cum_fees_usdc, cum_gas_usdc }] }`. Query `days` (integer, default 30, clamped to [1, 90]; non-numeric falls back to 30, never a 400) and `granularity` (`day` default, or `hour`; anything else is 400 `invalid_request`; `hour` clamps `days` to 7). For a calendar window use `from` and `to` instead of `days`: a date (`2026-09-01`) or an ISO-8601 datetime, UTC; a plain-date `to` is inclusive, so `from=2026-09-01&to=2026-09-30` is all of September; `to` defaults to now and is clamped to now; `to` without `from`, a bad date, a future `from`, `from` after `to`, or a range over 366 days (7 at `hour`) is 400 `invalid_request`; `days` is ignored when `from` is given. Add `wallet=0x...` (either case; malformed is 400) to limit the sums to that wallet's strategies: the body then echoes the lowercased `wallet`, and a wallet with no activity returns a zero-filled series, not a 404. Buckets are UTC. The window starts `days` back, aligned down to a bucket boundary, and ends at the current partial bucket, so 30 days at `day` is 31 buckets. Every bucket is present, zero-filled. `cum_*` start at 0 at the window start (not at inception) and `totals` equals the last row's `cum_*`. USDC values are decimal numbers rounded to 4 places. Fees and gas are the sums recorded on contract events for all strategies, including closed and deleted ones. Gas a user's wallet pays to sign its own deposit or withdraw is not recorded and is not included. No per-strategy breakdown. 500 `internal_error` on a DB failure (never an empty 200). 60s in-isolate cache, `Cache-Control: public, max-age=60`. Bucket `reads`.

### GET /v1/stats/strategies

How many strategies were funded and by how many wallets, per UTC day. `{ from, to, totals: { strategies_funded, wallets, strategies_per_wallet }, all_time: { strategies_funded, wallets, strategies_per_wallet }, series: [{ bucket, strategies_funded, wallets }] }`. Query: `days` (default 30, clamped to [1, 366]) or `from` / `to` (same rules as `GET /v1/stats/fees`). `totals` covers the window; `all_time` covers everything. `strategies_per_wallet` = `strategies_funded / wallets`, 2 places, null when `wallets` is 0.

Gotchas: a strategy counts on the day of its FIRST DEPOSIT (`first_deposit_at`). A strategy that was never funded is deleted after an hour, so it is not counted anywhere; closed strategies still count. `wallets` is distinct within its own scope, so the daily `wallets` do not add up to `totals.wallets`. The series starts at the UTC day holding the window start and ends at the current partial day; every day is present, zeros included. 60s cache. Bucket `reads`.

### Platform stats: which call answers what?

Public, SIWE-free reads. Pick by question:

| Question | Call |
| -------- | ---- |
| How much AUM, how many active strategies, how many rebalances overall? | `GET /v1/stats` |
| How many strategies were created (funded) last week, or in a month? | `GET /v1/stats/strategies?days=7` or `?from=2026-09-01&to=2026-09-30`: read `totals.strategies_funded` |
| How many strategies does the average wallet make? | `GET /v1/stats/strategies`: read `all_time.strategies_per_wallet` (or `totals.strategies_per_wallet` for a window) |
| How much in fees did the platform earn in a month or the last N days? | `GET /v1/stats/fees?from=2026-09-01&to=2026-09-30` or `?days=14`: read `totals.fees_usdc` |
| How much in fees has this wallet paid over the past 2 weeks? | `GET /v1/stats/fees?days=14&wallet=0x...`: read `totals.fees_usdc` |
| What does the protocol pay in gas? | `GET /v1/stats/fees`: read `totals.gas_usdc` |
| Which strategies perform best, per-strategy APY across all wallets? | `GET /v1/strategies/leaderboard` |
| One wallet's strategies, positions or history? | `GET /v1/strategies?wallet=0x...`, then `/history` per strategy |

"Fees we earned" and "fees users paid" are the same figure: `fees_usdc` is the Earn fee charged on rebalances and partial withdrawals. All dates are UTC; when the user says "last week" or "September", state the exact dates you queried. Report numbers as returned; do not infer a revenue or profit figure beyond `fees_usdc`.

### GET /v1/prices

Native gas token USD prices, hourly cron: `{ ETH, POL, MON (numbers, USD), updated_at, chains: { "<chainId>": price } }`. `chains` keys are STRINGS. 503 `price_data_unavailable` when any token row is missing, older than 2 hours, or non-positive (loud-fail, never stale data). Bucket `reads`.

### GET /v1/config

Deployment discovery, 5-min cache (`Cache-Control: public, max-age=300`): `{ earnContract (the Earn proxy address, same on every chain), chains: [{ chainId, name, minStrategyUsdc (number|null, decimal dollars, UI GUIDANCE ONLY, not enforced by create) }], banner (object|null: { text, buttonCta, buttonIcon|null, buttonUrl }), coveredStrategiesEnabled (boolean: the operator kill switch for covered strategies; when false do NOT offer `covered`, the create route 403s `covered_disabled`; this read is cached 5 min, the create-time check is live) }`. A chain appears only when its RPC secret is configured; treat this as the authoritative live chain list. 500 only when the contract address env is missing/malformed; banner failures fail soft to null. Bucket `reads`.

## Vaults

### GET /v1/vaults

Unranked browse of every approved ACTIVE vault from the latest 5-minute snapshot. Query:

- `chains` (CSV of chain ids; non-numeric entries silently dropped, all-non-numeric is 400)
- `window` (minutes 1-1440, default 5; snapped UP to a precomputed column: 5/10/30/60/120/240/360/720/1440)
- `minTvl`, `minLiquidity` (decimal dollars, default 0; garbage silently falls back to 0; rows with null values are filtered out when a positive floor is set)
- `includeCovered` (only literal `true` opts covered OpenCover wrapper rows in; anything else excludes them)

200 `{ mode: "browse", window, vaults: [...] }`. Each vault: `chainId`, `vaultAddress` (lowercase), `name|null`, `apy` (number|null, percent 2dp at the snapped window; null = insufficient history, NOT zero), `windows { m5, m10, m30, h1, h2, h4, h6, h12, h24 }` (each number|null), `tvlUsd`/`availableLiquidityUsd`/`maxDepositUsd` (whole-dollar integers|null; `maxDepositUsd` null = uncapped), `atCapacity` (true iff max deposit is exactly 0), `latestAt`, `covered` (present-and-true ONLY on covered wrappers; key absent otherwise, treat absence as uncovered), `coverCapacity` (covered rows only, and only when `includeCovered=true`: `{ remainingUsd: number|null, checkedAt: date-time|null }`. It is ONE pool shared by all covered wrappers, so every covered row shows the same figure: never sum it across rows. It is a CACHED display figure refreshed by a sweep about every 5 minutes, NOT the deposit gate. `remainingUsd: null` = unknown, never zero. To size a covered deposit, use `GET /v1/vaults/cover-capacity` instead).

Errors: 400 `ranked_moved` if you send legacy ranked params (`capital`/`maxPositions`) here; 400 `invalid_request`; 500 on DB error (never a masking empty 200). Gotchas: the response `window` echoes the RAW requested value, not the snapped one; paused vaults (`active=false`) are excluded (also hides unlaunched vaults); APY is share-price-only (no reward emissions); 10s in-isolate cache. Bucket `vaults` 120/min.

### GET /v1/vaults/rankings

Capital-aware allocation preview: the same `selectVaults` pipeline that plans real deposits. Query: `capital` (REQUIRED, > 0, decimal dollars), `maxPositions` (REQUIRED, integer >= 1), `chains`, `minTvl`, `minLiquidity`, `minLiquidityMultiplier` (query default 2; the real deposit path uses the strategy's STORED `min_liquidity_entry_multiplier`, which create also defaults to 2, so pass your strategy's actual value for a faithful preview; a literal 1 fallback applies only to legacy rows whose column is NULL), `apySmoothingMinutes` (default 360, snapped to a precomputed column), `hiddenVaultKeys` (CSV of `chainId:0xaddress`, strict format, 400 on any malformed entry), `wallet` (merges that wallet's saved global hide list; 503 `hide_lookup_failed` if the read fails, fail-closed), `covered` (`"true"`/`"false"`).

200 `{ mode: "ranked", vaults: [{ chainId, vaultAddress, name (empty string if unknown, not null), apy (percent, 3dp), tvlUsd, availableLiquidityUsd|null, maxDepositUsd|null, atCapacity }], excludedChains: [{ chainId, reason }], eligibleCount (before the maxPositions cut), estimatedApy (mean of selected positive APYs, |null) }`.

Rankings takes only the multiplier form of the entry level: for a strategy that sets `min_liquidity_entry_usd`, pass `minLiquidityMultiplier = min_liquidity_entry_usd / min_liquidity_usd` for a faithful preview.

Gotcha: the spec says omitting `covered` gives an unscoped ranking, but the handler currently FORCES the uncovered partition when the param is absent (pre-launch gate); send `covered=true` explicitly to rank covered wrappers. At most `maxPositions` rows. 10s cache. Bucket `vaults`.

### GET /v1/vaults/cover-capacity

Remaining OpenCover underwriting capacity, read from OpenCover's capacity endpoint. No parameters. Capacity is ONE pool shared by all covered wrappers, so the response is a single figure, not a per-vault list.

200 `{ remainingUsd: number|null, asOf: date-time }`. `remainingUsd` is decimal dollars (`997077.665305`). **`null` means unknown, NOT zero**: the feed is not configured, no covered wrapper is listed, or no wrapper answered. Do not read it as "no capacity left". But the deposit gate fails closed on the same missing reading, so a `null` predicts a covered deposit will fail now with 503 `no_covered_capacity`. On `null`: retry after ~10s (the cache window); if it is still `null`, warn the user and offer a standard strategy or a later retry.

Use this, not the cached `coverCapacity` on vault rows, to size a covered deposit. The deposit gate (`calldata/deposit`) re-reads the same OpenCover source and passes only when `remaining - deposit >= 0` (a deposit of exactly the remaining pool passes). Pre-check before create: keep `capital_usdc` <= `remainingUsd`. Otherwise deposit calldata returns 503 `cover_pool_exhausted`.

Freshness: `asOf` is the SERVER time the reading was taken, not an OpenCover timestamp. The handler keeps a 10-second in-isolate cache (the spec says "request time" and documents no cache), so `asOf` can lag your request by up to ~10s, and a `null` reading is cached too. The unconfigured and no-wrapper paths skip the cache and return `asOf` = now. The handler stamps `asOf` itself and never reads it from a stored value, so an `asOf` older than ~1 minute means a stale copy was served: re-call, and tell the developer.

Errors: 429 `rate_limited` + `Retry-After`; 500 `internal_error` ("Coverage capacity is temporarily unavailable") when the vault lookup fails. A failed OpenCover call is NOT an error: it returns 200 with `remainingUsd: null`. Bucket `vaults` 120/min.

### GET /v1/vaults/{chainId}/{address}

Latest snapshot for ONE vault, same shape as a browse row, plus optional live on-chain liquidity. Path: `chainId` positive integer, `address` 0x + 40 hex (either case). Query: `window` (as browse), `include=liveLiquidity` (the only supported include).

200 = BrowseVault fields + conditionally: `coverCapacity { remainingUsd|null, checkedAt|null }` and `underlyingVault { address, name|null }` (both on covered wrappers only; `coverCapacity` is the same cached sweep figure as on browse rows, always present here with no `includeCovered` needed), and with `include=liveLiquidity` either `liveLiquidity { morphoVersion (v1|v2|v2_with_v1_adapter), withdrawableUsdc (decimal dollars, what redeem() can pull NOW), forceDeallocatableUsdc }` or `liveLiquidity: null` + `liveLiquidityError: "rpc_error"` (still 200). 404 `not_found` when not an approved vault.

Gotchas: NO `active` filter (paused vaults stay inspectable by address, unlike browse); for covered wrappers the live read is retargeted at the UNDERLYING vault while `morphoVersion` stays the wrapper's; `withdrawableUsdc` is decimal dollars vs the snapshot's whole-dollar integer. No cache. Bucket `vaults`.

### GET /v1/vaults/{chainId}/{address}/apy

Raw 5-minute APY snapshot series over `[from, to]` plus `intervalApy` (annualized share-price return over the range). Query: `from` (ISO, default `to` - 24h, clamped to the 7-day retention; clamping sets `clamped: true`), `to` (ISO, default now), `resolution` (integer minutes 5-1440; omit for the full 5-min series). 400 when `from >= to` or the range is entirely outside retention.

200 `{ chainId, vaultAddress, from (EFFECTIVE, post-clamp), to, clamped, retentionDays: 7, resolutionMinutes|null, intervalApy (percent 4dp |null; computed from the FULL undownsampled range so resolution never changes it; share-price-only), snapshots: [{ snapshotAt, apy5m..apy24h (number|null 2dp), assetsPerShare (number|null; OPAQUE 100x-scaled share price, meaningful only as a ratio between snapshots), tvlUsd|null, availableLiquidityUsd|null }] }`.

Gotchas: NO 404 for unknown vaults (200 with empty `snapshots`); downsampling keeps the LAST real snapshot per bucket (whole rows, never averages). Bucket `vaults`.

## Strategies: reads

### GET /v1/strategies?wallet=0x...

`wallet` query is REQUIRED (400 before the rate limiter if missing; a malformed non-empty value is NOT rejected, it just matches nothing). 200 `{ strategies: [...], closedStrategies: [...] }`.

Strategy object, stored config: `id`, `wallet_address`, `name`, `capital_usdc` (decimal dollars), `delta_pct` (percentage points, same-network rebalance APY-gap bar), `cross_chain_delta_pct` (0 = inherit `delta_pct`), `delta_confirmations`, `max_positions`, `min_tvl_usd`, `min_liquidity_usd`, `min_tvl_exit_usd` (held-vault TVL exit floor, decimal dollars; 0 = TVL exit off), `min_liquidity_entry_usd` (liquidity a vault needs before the strategy ENTERS it; never below `min_liquidity_usd`), `min_liquidity_entry_multiplier` and `min_tvl_exit_ratio` (the two deprecated aliases of those dollar fields, always consistent with them: entry = `min_liquidity_usd` x multiplier, exit = `min_tvl_usd` x ratio), `chain_id` (legacy primary), `chain_ids` (CSV STRING|null, authoritative), `apy_smoothing_minutes`, `hidden_vault_keys` (string[]), `status` (`pending_setup|active|paused|closing|closed`), `active` (boolean), `created_at`, `updated_at`, `first_deposit_at|null`, `deactivated_at|null`, `final_value_usdc|null`, `final_realized_apy|null`, `covered`, `coverage_activated_at|null` (a FUTURE instant: `first_deposit_at` + 24h; coverage in force only once now >= it), `cycle_state` (`idle|pending|error|action_needed`|null, plus `rebalancing` on legacy rows only; the autopilot's cycle phase; `null` only until the first cycle: a resting strategy reads `idle`).

Computed rollups: `total_value_usdc` (live on-chain `convertToAssets` sum), `pending_bridge_value_usdc`, `pending_bridges_count`, `net_value_usdc` (total + in-flight bridges), `realized_apy` (annualized net %, null under 30 min of history, may be negative), `rebalance_count`, `total_fees_usdc`, `live_apy` (value-weighted position APY at the fixed 5-min window |null), `strategy_apy` (same at the strategy's own smoothing window; null when smoothing = 5), `positions: [{ id, strategy_id|null, vault_name, vault_address, protocol|null, chain_id, shares_raw (uint256 string|null), usdc_value (live, decimal dollars), entry_apy (0 when unrecorded), initial_usd_value|null, paused }]`.

`closedStrategies` rows differ: bridge fields pinned 0, `net_value_usdc` = `total_value_usdc` (from `final_value_usdc`), `realized_apy` prefers stored `final_realized_apy`, NO `live_apy`/`strategy_apy` keys, and positions omit `chain_id`/`initial_usd_value`/`paused`. 8s per-wallet in-isolate cache. Bucket `strategies_list`.

### GET /v1/strategies/{id}

Everything the list returns plus the REMAINING autopilot telemetry. No ownership check (id is the capability). 200 `{ strategy: {...} }` adding:

- `last_cycle_at|null`, `last_error|null`, `swap_signals` (opaque per-pair confirmation counters|null), `unmatched_exits|null`, `last_skip|null`
- `has_agreement` (owner signed ToS)
- `positions[]` with `current_apy` (number|null: the autopilot's own APY for the position at the strategy's smoothing window; authoritative for held vaults absent from browse, i.e. paused or covered-partition vaults; detail-only, no `strategy_id` field here)
- `closed_positions[]` (OMITTED entirely when none, not an empty array)
- `vault_names` (map `chainId:0xaddress` -> name; built from active, uncovered `approved_vaults` only; held positions carry their own names)
- `pending_rebalance` (object|null): in-flight same-chain rebalance: `{ chain_id, from_vault|null, to_vault|null, submitted_tx_hash|null, phase ("submitted"=pre-broadcast | "confirming"=awaiting confirmation), created_at, expires_at }`
- `cover` (covered strategies ONLY, key omitted otherwise): `{ premium_rate_bps (105 = 1.05%/yr), premiums_paid_usdc|null (null preserved, never coerced to 0), projected_1w_usdc, projected_1mo_usdc, projected_1y_usdc (computed from live value at read time, not byte-stable between polls), coverage_activated_at|null }`
- `total_value_usdc` by status: `closing` = live + closed-position total; `closed` = `final_value_usdc` verbatim; else live total.

Errors: 404 `not_found` (also for non-UUID ids: this handler does not pre-validate the UUID, unlike its siblings). 12s per-id cache. Bucket `strategies_detail`.

### GET /v1/strategies/leaderboard

Public board of every eligible strategy across ALL wallets, ranked by realized APY. No SIWE, no query parameters (any query string is ignored; there is no filter, sort or page control, so filter and rank client-side). The one cross-wallet strategy read; the list read needs a `wallet`.

What is measured:

- Realized APY is over TOTAL contributed capital (initial deposit plus top-ups), with mid-life withdrawals credited back. A top-up cannot pose as yield. Annualized linearly from the deposit clock start (when the deposit fully landed, bridges included) to the valuation instant.
- `live` rows (status active, paused or closing) are valued from their latest yield snapshot (~hourly) at that snapshot's own timestamp. `closed` rows use their settled `final_value_usdc` at the close time.
- Included: any strategy that ever took a deposit, however short-lived. Excluded: `pending_setup`, deleted rows, and closed rows with no recorded final value. A live row shows up only after its first yield snapshot (within ~1 hour of funding); closed rows appear at once.
- Sorted by `realized_apy_pct` descending, nulls LAST, ties by earliest `first_deposit_at`. Capped at 500 rows. There is no rank field: number your own filtered view.

200:

```
{ updated_at (date-time: when the board was computed),
  rows: [{
    strategy_id (uuid), name (string, "" if unset),
    wallet ("0x1234…abcd": truncated server-side, with a real "…" character),
    first_deposit_at (timestamp), ended_at (timestamp for closed, null for live),
    measured_at (date-time: the valuation instant),
    status (live | closed),
    value_usdc (number, decimal dollars, 2dp, at measured_at),
    realized_apy_pct (number 4dp | null; NET of Earn fees; unbounded, may be negative),
    gross_apy_pct (number 4dp | null; same with fees added back),
    fees_usdc (number, 2dp), rebalance_count (integer),
    covered (boolean), clone_count (integer), cloned_from (uuid | null),
    config: { delta_pct, cross_chain_delta_pct (0 = inherit delta_pct), delta_confirmations,
              max_positions, min_tvl_usd, min_liquidity_usd, min_tvl_exit_usd,
              min_liquidity_entry_usd, min_tvl_exit_ratio, min_liquidity_entry_multiplier,
              apy_smoothing_minutes, chain_id (legacy primary), chain_ids (integer[]) }
  }] }
```

`realized_apy_pct` and `gross_apy_pct` are null when the row has under 30 minutes of measured time, or while its deposit bridge is still in flight. The row still ships. All other fields are non-null numbers, strings or booleans. `config` values are numbers. `min_tvl_exit_ratio` and `min_liquidity_entry_multiplier` are deprecated aliases of the two dollar fields (see the strategy object). Withheld from the board: `capital_usdc`, owner id, the full wallet, `hidden_vault_keys`.

Errors: 429 `rate_limited` + `Retry-After`; 500 `internal_error` ("Leaderboard data is temporarily unavailable") when the data read fails. The spec also lists 400, but the handler takes no parameters and never returns it. Bucket `vaults` 120/min (the router uses the `vaults` bucket here, not `strategies_list`).

Gotchas:

- Values lag. A live figure is up to ~1h old (snapshot cadence) plus up to ~5 min more (the board is recomputed at most every 5 minutes; `updated_at` is the compute time and each edge isolate caches its own copy). A paused strategy's `measured_at` stops moving, so derive age from `first_deposit_at` to `measured_at`, not to now.
- `wallet` is truncated for display only. `strategy_id` is not secret: `GET /v1/strategies/{id}` returns the full strategy with no ownership check.
- Timestamps mix `+00:00` and `Z` suffixes (`first_deposit_at`, `ended_at` vs `measured_at`): parse them, never compare as strings.
- Spec discrepancy: the spec describes `status: live` as "active/paused" only. The handler also marks `closing` strategies `live`.
- `clone_count` counts distinct wallets that hold a live funded clone made with settings unchanged (the owner's own wallet and closed or undeposited clones do not count). `cloned_from` always names the root original.
- Do not confuse `realized_apy_pct` with a forecast or with a vault's `apy`. It is a past, per-strategy, after-fee result.

### GET /v1/strategies/{id}/history

Event log, newest-first, cap 100. `format` query: `events` (default) or `entries`; anything else 400. UUID-validated (400).

`format=events`: `{ events: [{ id, strategy_id, wallet_address|null, event_type (legacy label), kind (deposit|crosschain_deposit|rebalance|crosschain_rebalance|withdrawal|crosschain_withdrawal), from_vault|null, to_vault|null, from_vault_name|null, to_vault_name|null, from_apy|null, to_apy|null (percent), usdc_amount|null, fee_usdc|null, gas_cost_usdc|null (decimal dollars), tx_hash|null, burn_tx_hash|null, timestamp, chain_id, log_index|null, forced (true = liquidity/hide-driven exit from a higher-APY vault) }] }`. No truncation flag: exactly 100 events means older history silently dropped.

`format=entries` (the UI accordion aggregation): `{ entries: [{ id (byte-stable), category (strategy_enter|strategy_exit|rebalance|crosschain_rebalance|force_rebalance|force_crosschain_rebalance), timestamp, amountUsdc, feeUsdc|null, pending?, phases: [{ subHeader?, timestamp|null, chainId|null, fromVault?, toVault? (vault obj or array: { name, apy|null, amountUsdc? }), txHash|null, pendingLabel? }] }], truncated }`. `truncated` true when internal caps (100 events / 50 transfers) were hit.

Gotcha: unknown strategy (valid UUID) returns 200 with empty events, never 404. Bucket `reads`.

### GET /v1/strategies/{id}/bridges

CCTP transfer rows, newest-first, hard cap 50, no flag. 200 `{ transfers: [{ id, strategy_id|null, source_chain_id, dest_chain_id, amount_usdc (STRING, 6dp base units), burn_tx_hash, relay_tx_hash|null, status (burn_submitted|attestation_pending|relay_submitted|confirmed|user_claimed), user (bytes32-padded|null), hooks_completed, source_vault|null, dest_vault|null, dest_vaults|null, transfer_type (deposit|withdrawal|null legacy), fee_usdc (NUMBER, decimal dollars), batch_id|null (groups sibling legs of one logical deposit/close; null on legacy and automated rebalance legs), created_at, updated_at }] }`.

This is where you find `transferId` for the emergency claim. Unknown strategy: 200 empty. Bucket `reads`.

### GET /v1/strategies/{id}/performance

Yield time series from ~55-minute snapshots, oldest-first. Query `hours` (default 168, clamped [1, 2160]; garbage falls back to default, never 400). 200 `{ hours (clamped), truncated (cap 2000, keeps NEWEST rows), snapshots: [{ id, total_value_usdc|null, period_yield|null (gross yield since previous row), cumulative_yield|null (since INCEPTION, not window-scoped), cumulative_fees|null (since inception), period_fee_usdc (derived diff, first row always 0), net_yield (period_yield - period_fee_usdc), period_cover_fee_usdc (COVERED strategies only, absent otherwise; never subtracted from net_yield, the premium is already in the wrapper share price), weighted_apy|null (percent), snapshot_at }] }`.

The only strategy read with a real existence check: 404 `not_found` on unknown id. Note 90-day windows exceed the 2000-row cap (~2356 rows), so `truncated` will be true. Treat null numerics as 0 when aggregating. Bucket `reads`.

### GET /v1/strategies/{id}/intents/{intentId}

Poll one intent (attribution row written by the calldata endpoints or the system). Both ids UUID-validated (400). 200:

```
{ id, strategy_id, chain_id,
  kind (user: deposit | close | emergency_claim; system: rebalance | withdraw_bridge | relay_deposit),
  created_at, expires_at,
  status (DERIVED: pending | fulfilled | failed | expired),
  fulfilled_at|null, fulfilled_tx_hash|null, submitted_tx_hash|null, failure_reason|null }
```

`status` rules: `failed` wins over fulfilled (a failure sentinel sets `fulfilled_at`, so never branch on `fulfilled_at` alone); `fulfilled_tx_hash` is ALWAYS null on failure (the internal sentinel never leaks); `failure_reason` is URL-redacted and capped at 200 chars. `expires_at` is the 30-day attribution TTL, NOT the calldata response's ~1h advisory `expiresAt`; different clocks. 404 `intent_not_found` covers both nonexistent and other-strategy intents (no existence oracle). Bucket `reads`.

## Strategies: writes (SIWE)

### POST /v1/strategies

Creates a strategy in `pending_setup` owned by the recovered wallet. DB-only; funding is a separate step. SIWE `strategy.create`, resource `/strategies`. Buckets: `auth_probe` then `create` 40/min per wallet+IP.

Signed payload fields:

| Field                            | Type         | Required          | Notes                                                                                                                                                                                                                                                                                                                                                                                        |
| -------------------------------- | ------------ | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`                           | string       | yes               | trimmed; `^[a-zA-Z0-9 _\-.]{1,50}$` else 400                                                                                                                                                                                                                                                                                                                                                 |
| `capital_usdc`                   | number       | yes (spec)        | decimal dollars. WARNING: not handler-validated; omission/garbage surfaces as 500, not 400. Balance-checked on the primary chain only, FAIL-OPEN on RPC error; insufficient: 400 "Insufficient USDC balance..."                                                                                                                                                                              |
| `max_positions`                  | integer      | yes (spec)        | WARNING: also not handler-validated; omission is a 500                                                                                                                                                                                                                                                                                                                                       |
| `delta_pct`                      | number       | no                | same-network APY-gap bar, percentage points. Hard floors: >= 3, or >= 5 when the chain set includes Ethereum (1). OMITTED defaults to the floor itself; a PRESENT below-floor value is 400                                                                                                                                                                                                   |
| `cross_chain_delta_pct`          | number >= 0  | no                | **0 (and omitted) = "inherit delta_pct" sentinel, never a literal bar.** Multi-network: the EFFECTIVE cross bar (value when > 0, else `delta_pct`) must be >= 5 (>= 7 with Ethereum) else 400. Single-network skips the check                                                                                                                                                                |
| `delta_confirmations`            | integer      | no                | consecutive confirmation cycles before a rebalance; defaults to 12 (60 min) when omitted                                                                                                                                                                                                                                                                                                     |
| `min_tvl_usd`                    | number       | no                | vault TVL floor, USD                                                                                                                                                                                                                                                                                                                                                                         |
| `min_liquidity_usd`              | number       | no, default 0     | liquidity floor; also drives liquidity-forced exits                                                                                                                                                                                                                                                                                                                                          |
| `min_liquidity_entry_multiplier` | number       | no, default 2     | DEPRECATED alias of `min_liquidity_entry_usd`: entry gate = multiplier x `min_liquidity_usd`. Not with `min_liquidity_entry_usd` (400)                                                                                                                                                                                                                                                     |
| `min_liquidity_entry_usd`        | number >= 0  | no                | entry gate in dollars (preferred, v1.10.0): a non-held vault needs this much liquidity to be entered. Below `min_liquidity_usd` it is raised to it; `min_liquidity_usd` 0 stores 0. Omitted = 2 x `min_liquidity_usd`                                                                                                                                                                       |
| `min_tvl_exit_usd`               | number >= 0  | no                | held-vault TVL exit floor in dollars (preferred): a held vault is force-exited only when its live TVL drops below it. 0 = TVL exit off. Must not exceed `min_tvl_usd` (400). Not with `min_tvl_exit_ratio` (400). Omitted = 0.75 x `min_tvl_usd`                                                                                                                                       |
| `min_tvl_exit_ratio`             | number 0-1   | no                | DEPRECATED alias of `min_tvl_exit_usd`: exit floor = ratio x `min_tvl_usd`. Outside 0-1 is 400. Omitted = 0.75                                                                                                                                                                                                                                                                              |
| `apy_smoothing_minutes`          | integer      | no, default 360   | ranking window, snaps to 5/10/30/60/120/240/360/720/1440                                                                                                                                                                                                                                                                                                                                     |
| `chain_id`                       | integer      | no                | primary chain. Invalid values NEVER 400: silent fallback to the first enabled chain (Base 8453 in practice today, but this can vary by deployment)                                                                                                                                                                                                                                           |
| `chain_ids`                      | integer[]    | no                | ARRAY in, **CSV string out**. Unsupported entries silently dropped; empty result collapses to the primary chain                                                                                                                                                                                                                                                                              |
| `covered`                        | boolean      | no, default false | set-once, IMMUTABLE. `true` pins the invest set to Base (`chain_ids` → "8453"; client `chain_ids` ignored) while the funding chain (`chain_id`) stays free (a non-Base value bridges the deposit to Base via CCTP), and requires the kill switch on (403 `covered_disabled` otherwise) plus a stored opencover-terms signature (403 `covered_terms_required`); lookup errors fail closed 503 |
| `hidden_vault_keys`              | string[]     | no                | `chainId:0xaddress` entries, strict format (400). Non-empty triggers the hide-floor check: merged with the wallet's global hides, the eligible pool must keep >= `max_positions + 1` vaults else 409 `hide_floor_violated`                                                                                                                                                                   |
| `idempotencyKey`                 | string 1-128 | no                | inside the signed payload (a replay must carry the identical signature). Wallet-scoped. A hit returns the ORIGINAL stored 201 verbatim, Stripe-style, no TTL. Never reuse a key for a logically different request                                                                                                                                                                            |

201 `{ strategy: {...} }` (the allow-listed row; see the list-read shape for fields; `status "pending_setup"`, an omitted `delta_pct` reads back as the floor, `chain_ids` comes back as a CSV string).

Errors: 400 `invalid_request` / `siwe_*` 400s; 401 `siwe_*`; 403 `covered_disabled` / `covered_terms_required`; 409 `hide_floor_violated`; 429; 500 `internal_error` (including the unvalidated-required-field cases); 503 `covered_access_lookup_failed` / `hide_lookup_failed` / `siwe_rpc`.

### PATCH /v1/strategies/{id}

Partial config update. SIWE `strategy.update`, resource `/strategies/{id}` (lowercased). Buckets: `auth_probe` then `mutations` 80/min per wallet+IP. Ownership by scoping the UPDATE to id + recovered wallet.

Accepted fields (any subset; absent = untouched; **explicit `null` writes NULL and can 500 on non-nullable columns, do not send nulls casually**): `name`, `capital_usdc` (NO balance check on update), `delta_pct`, `cross_chain_delta_pct` (send **0**, not null, to reset a split cross bar back to inherit), `delta_confirmations`, `max_positions`, `min_tvl_usd`, `min_liquidity_usd`, `min_tvl_exit_usd`, `min_liquidity_entry_usd`, `min_liquidity_entry_multiplier` and `min_tvl_exit_ratio` (the two aliases are deprecated; sending a dollar field AND its alias is 400. Since v1.10.0, changing `min_tvl_usd` or `min_liquidity_usd` ALONE does not move the dollar levels: they stay, and the aliases are re-derived. A stored exit floor above a lowered `min_tvl_usd` clamps down to it, and an entry level below a raised `min_liquidity_usd` rises to it. An explicit `min_tvl_exit_usd` above the effective post-patch `min_tvl_usd` is 400), `apy_smoothing_minutes`, `status` (**enum `active`/`paused` ONLY**; pause/resume; a closed row can never be revived; 400 otherwise), `hidden_vault_keys` (full-array REPLACE), `hide_vault_key` / `unhide_vault_key` (single-op server-side merge; exactly one per request, never combined with the array replace, 400).

Threshold floors are validated ONLY when a threshold field is in the payload, against the effective post-PATCH pair (payload value when present, stored otherwise). A `delta_pct`-only PATCH on an inherit-mode multi-network strategy moves BOTH bars and is validated against the cross floor too. Hide ops run the same 409 `hide_floor_violated` check as create (`unhide` skips it). NOT updatable, silently ignored: `chain_id`, `chain_ids`, `covered`.

200 `{ strategy: {...} }`. Errors: 400; 401 `siwe_*`; 409 `hide_floor_violated`; 429; **500 `internal_error` for a missing/unowned id (deliberate legacy coupling, NOT 404)**, though if the payload contains a threshold field the pre-write row read hits first and returns 503 `strategy_read_failed` instead; 503 `strategy_read_failed` / `hide_lookup_failed` / `siwe_rpc`.

### DELETE /v1/strategies/{id}

Delete-or-close. SIWE `strategy.delete`, resource `/strategies/{id}` (lowercased). Buckets: `auth_probe` then `mutations`. Missing or unowned id: **404 `not_found`** (unlike PATCH). Despite the spec marking the body optional, the handler requires a JSON body (at least the `siwe` block); an absent body is 400.

Optional close payload (part of the signed payload): legacy `{ txHash, finalValueUsdc }`, or `perChain: { "<chainId>": { txHash, finalValueUsdc, feeUsdc } }` (keys are chain-id STRINGS; when present it must cover every chain that still has active positions, else 400 "Active positions remain on chain(s) X"). Withdrawal amounts already recorded from on-chain close events always take precedence over a legacy `finalValueUsdc`: only its excess above the recorded event total is counted (floored at zero), standing in for the legs those events do not cover.

Three branches, check `action` in the response to know which ran:

1. **Already closed**: immediate 200 `{ success: true, action: "closed" }`, nothing touched, nonce NOT burned (safe replay).
2. **Pre-deposit** (`first_deposit_at` null AND `status "pending_setup"`): hard delete of the row, positions, snapshots (unrecoverable). 200 `{ success: true, action: "deleted" }`.
3. **Funded close** (everything else): computes final value per chain (body value when > 0, else DB position sum, plus already-recorded withdrawal proceeds), claims the close race-safely (a lost race still returns 200 `closed`), deactivates positions, appends the final yield snapshot. 200 `{ success: true, action: "closed" }`.

Treat a 200 `closed` as terminal; only use this endpoint on funded strategies when the withdraw plan returned `fallback: "delete"`.

## Calldata (public, no SIWE)

Shared contract for all three: the on-chain signature is the authorization boundary. The handler writes the `pending_intent` row server-side in the same call, so `pending_intent.calldata_hash == keccak256(data)` by construction: **never POST an intent yourself, and submit `data` VERBATIM** (re-encoding breaks attribution). Regenerating an unchanged plan within the intent's 30-day TTL returns the SAME `intentId`; a changed plan or expired/fulfilled intent mints a new one. `expiresAt` (~1h) is ADVISORY plan freshness, a different clock from the intent TTL. All three share the `calldata` bucket, 40/min per strategy+IP, behind the router's `auth_probe`. Responses share a discriminated union on top-level `kind` (`deposit`/`withdraw`/`claim`).

Transaction object: `{ chainId, to (the Earn proxy), data (hex, submit verbatim), value "0", intentId, kind (deposit|close|emergency_claim), description, gasHint (string|null: server estimate x 1.10; USE IT as the gas limit, wallet re-estimates can OOG), gasHintReason? ("approvals_required" | "estimation_failed") }`.

ApprovalNeeded object: `{ chainId, token, spender, currentAllowance, requiredAllowance (base-unit strings; maxUint256 78-digit string for share tokens), tx { to, data, value } }` (ready-to-sign approve).

### POST /v1/strategies/{id}/calldata/deposit

Builds the full allocation plan for a `pending_setup` strategy and returns EXACTLY ONE `selfBatchDeposit` transaction on the strategy's source chain (local vault deposits + per-remote-chain CCTP burn legs). Body: none/`{}` (non-object JSON is 400).

200 `{ kind: "deposit", transactions: [tx], approvalsNeeded: [...], plan: { vaults (local only), amounts, burns (per remote chain: destDomain, mintRecipient, destinationCaller, amount, maxFee, minFinalityThreshold), burnVaultBreakdown, totalUsdc, apys, vaultNames, chainIds }, expiresAt }`.

Errors: 400 `invalid_state` when not `pending_setup` (exact message "Strategy status is '<status>', expected 'pending_setup'"; no top-ups); 400 `no_eligible_vaults`; 400 `gas_estimation_failed` (post-approval sim revert; the just-written intent is auto-marked failed); 404; 429; 500 `burn_route_unavailable` (a planned remote chain lacks a CCTP route; hard error, never a partial tx); 503 `hide_lookup_failed`; 503 `cover_pool_exhausted` / `no_covered_capacity` (covered strategy blocked by the OpenCover capacity gate — checked HERE and nowhere else in the lifecycle; coverage capacity is ONE shared pool across all covered wrappers, and the rule is `remaining - deposit >= 0` (a deposit of exactly the remaining pool passes); `cover_pool_exhausted` = the pool cannot cover the deposit amount, retrying will not help — the message carries the live remaining figure ("only $X ... remains"), reduce `capital_usdc` via PATCH to at most that figure or wait for capacity to free up; avoid it by checking `GET /v1/vaults/cover-capacity` BEFORE you create the strategy; `no_covered_capacity` = the capacity feed could not be read, transient, retry shortly; NEITHER is in the public spec's code list).

Gotchas: a successful build bumps `strategies.updated_at`, restarting the 1-hour never-deposited GC clock (regenerate hourly while collecting multisig signatures); source-chain USDC allowance is always evaluated for the FULL capital even when fully bridged; vault selection merges strategy + wallet-global hide lists and judges entry liquidity at the strategy's STORED `min_liquidity_entry_multiplier` (create default 2; the literal 1 is only a fallback for legacy rows with a NULL column); covered strategies select ONLY covered wrappers.

### POST /v1/strategies/{id}/calldata/withdraw

Close/withdraw plan across active positions: one `selfBatchWithdraw` transaction PER CHAIN, source chain first then ascending chain id. **Sign and submit in array order.** Remote legs CCTP-burn redeemed USDC back to the source chain. No status gate. Body optional: `{}`/empty = all chains; `{ "chainIds": [8453] }` = subset (whole positions only; there is no partial-amount withdraw).

200 `{ kind: "withdraw", transactions: [...], approvalsNeeded (share-token approvals only; OMITTED on the empty fallback response), plan: { chains: { "<chainId>": { vaults, shares, feeAmounts (all "0"), totalFeeUsdc (0), burns } } }, fallback?: "delete", expiresAt }`. `fallback: "delete"` appears ONLY when `transactions` is empty and no subset was requested: nothing is withdrawable on-chain, close via DELETE instead.

Errors: 400 `invalid_request` (bad `chainIds` type); 400 `no_positions_for_chains` (subset matched nothing; deliberately NOT the delete fallback, protecting other chains' positions); 404; 429; 500.

Gotchas: the final burn leg per remote chain uses the maxUint256 78-digit string amount ("burn all redeemed USDC"), keep it a string; paused positions are excluded entirely (redeem those shares directly against the vault); per-chain `gasHint` degrades to `estimation_failed` instead of erroring (a close is never blocked); all per-chain intents share one `metadata.batchId`, surfaced later as `cctp_transfers.batch_id`.

### POST /v1/strategies/{id}/calldata/claim

Emergency escape hatch for a STALLED DEPOSIT bridge leg the relayer never landed. Fetches Circle's finalized (message, attestation) pair and returns one `emergencyClaimBridge` transaction on the transfer's DESTINATION chain. Minted USDC goes directly to the burn-time beneficiary's wallet, bypassing the vault deposit. Body REQUIRED: `{ "transferId": "<uuid from GET .../bridges>" }`.

200 `{ kind: "claim", transactions: [one tx, chainId = dest chain, kind "emergency_claim"], approvalsNeeded: [] (always), expiresAt (nominal; a finalized attestation never goes stale) }`.

Errors: 400 `invalid_state`, one message per gate: transfer already terminal; not deposit-shaped ("Withdrawal legs mint to your wallet without the relayer"); younger than 30 minutes; beneficiary mismatch. 404 `transfer_not_found` (also for other strategies' transfer ids). **409 `attestation_pending` + `Retry-After: 60`: poll on the status code** (Circle not finalized yet). 500 `unsupported_chain_pair` (CCTP config gap); 500 on intent-write failure (hard, unlike the legacy browser flow). 429.

Gotcha: `gasHint: null` + `estimation_failed` on claim is often GOOD news: the message nonce was already consumed, i.e. the relayer landed the deposit after all; check on-chain before assuming failure. Anyone can broadcast the tx; funds always land at the burn-time beneficiary.

## Wallets

### GET /v1/wallets/{addr}/balances

Per-chain USDC + native gas balances. Path `addr` must be 0x + 40 hex (400). Query `chains` CSV, STRICT: any non-numeric or unsupported entry is a 400 (unlike the vaults lane's silent drop). 200 `{ address (lowercased), chains: [{ chainId, nativeSymbol, nativeBalanceWei (wei string|null), nativeUsdPrice (NUMBER, decimal dollars|null, from the DB price snapshot), usdc (token address|null), usdcBalance (6dp base-unit string|null), error ("rpc_error"|null) }] }`. Per-chain isolation: one chain's RPC failure never fails the response. Bucket `balances` 120/min.

### GET /v1/wallets/{addr}/approvals

Current USDC + per-vault share-token allowances toward the Earn proxy (the single spender), grouped per chain, with a ready-to-sign `approve` template on every entry that is not yet approved. Query:

- `chains` CSV (strict, as balances). The legacy `chainId=` param is REJECTED with a 400 pointing at `chains=`.
- `requiredUsdc` (base-unit integer STRING, default "0"): evaluated against EACH requested chain's USDC allowance. With the default 0 the USDC entry is always `approved: true` with no `tx`; pass the real deposit amount to get a usable template.
- `vaults` CSV of `chainId:0xaddress` keys (strict format; a key targeting a chain outside the requested set is a 400). Omitted = every approved vault per chain, EXCLUDING covered wrappers (request those explicitly).

200 `{ address (lowercased), chains: [{ chainId, spender (proxy, checksummed|null), usdc (ApprovalEntry|null), vaults: ApprovalEntry[], error ("rpc_error"|null) }] }`. ApprovalEntry: `{ token (checksummed), vaultAddress? (share-token entries only, absent on USDC), currentAllowance, requiredAllowance (maxUint256 string for share tokens, `requiredUsdc` for USDC), approved (share token: allowance > 0; USDC: allowance >= required), tx? ({ to, data, value "0" }, present only when not approved) }`.

To approve: for each chain with `error: null`, sign+broadcast `usdc.tx` (if present) and every unapproved `vaults[].tx` on that `chainId`, then re-poll. The USDC approve encodes the EXACT `requiredUsdc` amount (the spec's example showing maxUint256 for USDC is wrong; only share tokens use maxUint256). 500 on `approved_vaults` DB failure or missing proxy config (loud, never masked as per-chain rpc_error). Bucket `approvals` 120/min.

### GET /v1/wallets/{addr}/prefs

Public read; unknown wallets return defaults, never 404. 200 `{ address (lowercased), has_agreement (ToS recorded), hidden_vault_keys (global hide list), hide_approval (banner dismissed), covered_access (always true: covered strategies are open to every wallet), opencover_terms_accepted (SOFT signal) }`. The legacy `approved` field no longer exists on the wire (stale spec prose mentions it). Bucket `reads`.

### PATCH /v1/wallets/{addr}/prefs

SIWE `prefs.update`, owner-checked (403 `owner_mismatch`). Payload: at least one of `hide_vault_key` (add a `chainId:0xaddress` to the GLOBAL hide list), `unhide_vault_key` (remove; may be combined with hide, unlike the strategy lane; same key in both = removed), `hide_approval` (boolean; `null` is a 400).

200 `{ ok: true, hidden_vault_keys (full merged list), hide_approval, warnings? }`. `warnings` (present only when non-empty, only computed for `hide_vault_key` requests) is ADVISORY: the write is always applied, unlike the strategy lane's 409. Items: `{ code: "hide_floor_violated", strategy_id, strategy_name, message }`, one per `pending_setup`/`active`/`paused` strategy whose eligible pool would drop below `max_positions + 1` (strategies in `closing` are skipped too). Hiding a held vault is allowed and can force-exit positions (advertised flow). 503 `prefs_read_failed` = nothing written, retry. Buckets `auth_probe` + `mutations`.

### POST /v1/wallets/{addr}/tos

SIWE `tos_agreement`, owner-checked. Records platform ToS acceptance (timestamp only; no signature persisted). Payload: optional `terms_url` (bound into the signature; echoed back, absent from the response when not sent). 200 `{ ok: true, address, has_agreement: true, terms_url? }`. NOTE: not enforced server-side by any other endpoint; it is recorded consent surfaced as `has_agreement`. Buckets `auth_probe` + `mutations`.

### POST /v1/wallets/{addr}/opencover-terms

SIWE `opencover_terms`, owner-checked. Unlike the platform ToS, the FULL signed message + raw signature are persisted as proof (third-party terms), one row per wallet, last-write-wins. Payload: optional `terms_url`. 200 `{ ok: true, address, opencover_terms_accepted: true, terms_url? }`. This ack HARD-GATES covered creates (`covered_terms_required` without it). Buckets `auth_probe` + `mutations`.

## Error code catalog

| Code                                                                                                              | Status | Meaning                                                                                                                                               |
| ----------------------------------------------------------------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `invalid_request`                                                                                                 | 400    | malformed input; message names the field                                                                                                              |
| `siwe_missing` / `siwe_shape` / `siwe_parse`                                                                      | 400    | siwe block absent / malformed / unparseable message                                                                                                   |
| `siwe_domain` / `siwe_action` / `siwe_stale` / `siwe_payload` / `siwe_chain` / `siwe_recover` / `siwe_replay`     | 401    | domain not allowlisted / wrong statement / temporal failure / payload-hash or path-binding mismatch / bad chain id / signature mismatch / nonce reuse |
| `owner_mismatch`                                                                                                  | 403    | wallet-lane signer != path address                                                                                                                    |
| `covered_disabled` / `covered_terms_required`                                                                     | 403    | covered create gate (kill switch off / OpenCover terms ack missing)                                                                                   |
| `not_found` / `intent_not_found` / `transfer_not_found`                                                           | 404    | resource lookups (intent/transfer codes also cover "belongs to another strategy")                                                                     |
| `method_not_allowed`                                                                                              | 405    | path matched, method did not                                                                                                                          |
| `invalid_state`                                                                                                   | 400    | wrong lifecycle state (deposit on non-pending_setup; claim eligibility gates)                                                                         |
| `no_eligible_vaults` / `no_positions_for_chains`                                                                  | 400    | empty selection / withdraw subset matched nothing                                                                                                     |
| `gas_estimation_failed`                                                                                           | 400    | deposit-only post-approval sim revert                                                                                                                 |
| `ranked_moved`                                                                                                    | 400    | ranked params sent to `/v1/vaults`                                                                                                                    |
| `hide_floor_violated`                                                                                             | 409    | hide would shrink the eligible pool below `max_positions + 1`                                                                                         |
| `attestation_pending`                                                                                             | 409    | claim: Circle not finalized; poll with `Retry-After`                                                                                                  |
| `rate_limited`                                                                                                    | 429    | over budget; `Retry-After` header                                                                                                                     |
| `burn_route_unavailable` / `unsupported_chain_pair`                                                               | 500    | CCTP config gaps                                                                                                                                      |
| `internal_error`                                                                                                  | 500    | generic; ALSO the deliberate "missing/unowned strategy" result on PATCH                                                                               |
| `price_data_unavailable`                                                                                          | 503    | prices stale/missing                                                                                                                                  |
| `hide_lookup_failed` / `strategy_read_failed` / `prefs_read_failed` / `covered_access_lookup_failed` / `siwe_rpc` | 503    | transient read/RPC failures; retry                                                                                                                    |
| `cover_pool_exhausted`                                                                                            | 503    | shared OpenCover pool cannot cover the deposit; message carries the live remaining figure (pre-check with `GET /v1/vaults/cover-capacity`)                                    |
| `no_covered_capacity`                                                                                             | 503    | OpenCover capacity feed unreadable; transient, retry                                                                                                  |

## Top traps (read these before writing code)

1. **The 1-hour pending_setup GC.** Create, then fund within an hour of your last deposit-calldata call, or the strategy row is deleted. Regenerate calldata hourly during multisig collection.
2. **Submit calldata verbatim, from the owner wallet.** Attribution is `keccak256(data)`; re-encoding or a different sender breaks intent matching (and `selfBatchDeposit` pulls USDC from `msg.sender`).
3. **Base-unit strings stay strings.** The 78-digit maxUint256 withdraw sentinel corrupts silently through a JS number.
4. **`cross_chain_delta_pct: 0` means inherit, not zero.** And omitted `delta_pct` on create stores the floor (3 or 5), so read-back differs from what you sent.
5. **Spec-required create fields are not handler-validated.** Omitting `capital_usdc` or `max_positions` is a 500, not a 400.
6. **PATCH on a missing/unowned strategy is a 500** (or a 503 if you sent a threshold field). DELETE is the one that 404s.
7. **Two different `expires` clocks**: calldata `expiresAt` (~1h advisory plan freshness) vs intent `expires_at` (30-day attribution TTL).
8. **Never branch on `fulfilled_at`**; branch on the derived intent `status` (failed intents also set `fulfilled_at`).
9. **Withdraw-then-DELETE order.** DELETE only when withdraw says `fallback: "delete"`; DELETE never touches the chain.
10. **Sign withdraw transactions in array order** (source chain first); paused positions are skipped and must be redeemed directly against the vault.
11. **Rankings preview can diverge from execution.** The preview uses its own query params while the deposit path uses the strategy's STORED settings (pass your strategy's actual `min_liquidity_entry_multiplier` etc. for a faithful preview), and rankings forces the uncovered partition when `covered` is omitted.
12. **SIWE nonces are single-use**; sign a fresh message per request. Idempotent create replays are the exception (same body + same key returns the stored 201).
13. **Always send the `apikey` header even though enforcement is not guaranteed today**; if the platform ever rejects a missing key, that rejection happens upstream of the API and does not use the JSON error envelope.
14. **A fresh wallet's FIRST `calldata/deposit` call needs vault share-token approvals granted BEFORE you make it, not after — for the WHOLE eligible pool, not just the top-N `maxPositions` vaults.** Deposit-mode vault selection (and later, rebalance-time entry selection) checks each candidate's real on-chain share-token allowance to the proxy and drops anything unapproved before building a plan — a never-approved wallet gets `400 no_eligible_vaults` on the very first call, with no `approvalsNeeded` hint (that field only ever reports the USDC leg). `GET /v1/vaults/rankings` only returns `maxPositions` rows, so approving just those reproduces the same failure the first time a rebalance wants to enter a different eligible vault later. Enumerate the full pool via `GET /v1/vaults?chains=...&minTvl=...&minLiquidity=...` and pre-approve every row in it (see the wizard's Step 3).

## Help improve this skill

If you are the agent using this file and you hit an ambiguity, a missing detail, or something that turned out to be wrong while following these instructions, do not just work around it silently. Tell the developer directing you exactly what was unclear or incorrect, and suggest the specific wording that would have made it unambiguous, so the next agent to use this file does not hit the same wall.