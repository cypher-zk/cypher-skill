# Cypher SDK — Troubleshooting

Symptom → diagnosis → fix.

## Encryption / events

### `data.payoutAmount` (or any event field) is `undefined`

**Cause**: reading snake_case (`data.payout_amount`) but the SDK emits camelCase.

**Fix**: use camelCase. The SDK normalizes Anchor's raw snake_case BN output on decode via `parseLogs`. If you bypassed `parseLogs` and called Anchor's `EventParser.parseLogs` directly, you'll see snake_case BN — switch to `parseLogs` from `@cypher-zk/sdk`.

### Numeric event field is a BN, not a `bigint`

Same cause. The SDK's `parseLogs` converts BN → bigint via duck-typing. If you bypassed it, you'll see BN.

### Decrypted amount is garbage after `decryptBetInput`

Causes (most likely first):
1. **Wrong nonce bytes** — `position.nonce` is u128; convert with `bigIntToLeBytes(position.nonce, 16)`.
2. **Wrong x25519 private key** — load the SAME key used to encrypt (keyed by `position.market`).
3. **MXE pubkey rotation** — refresh with `client.globalState.invalidate()`; historical positions need the original pubkey.

### `placeBet` callback never fires (computation timeout)

Default 120s. Causes: Arcium cluster slow, tx didn't land, network issues.

**Fix**: bump timeout — `client.actions.placeBet({ ..., timeoutMs: 300_000 })`.

## React / TypeScript

### `useCypherClient must be used within a <CypherProvider>`

Component renders **above** `<CypherProvider>`. Move it inside or move the provider up.

### React Query cache stale after a raw instruction call

Only mutation hooks auto-invalidate. After raw `client.bets.placeYesnoIx` + `sendIx`:
```ts
import { marketKeys, positionKeys } from "@cypher-zk/sdk/react";
qc.invalidateQueries({ queryKey: marketKeys.one(marketId) });
qc.invalidateQueries({ queryKey: positionKeys.byUser(user) });
```

### `An update to TestComponent inside a test was not wrapped in act(...)`

Bun's `bun:test` + happy-dom doesn't set `IS_REACT_ACT_ENVIRONMENT` automatically:
```ts
beforeAll(() => {
  (globalThis as { IS_REACT_ACT_ENVIRONMENT?: boolean }).IS_REACT_ACT_ENVIRONMENT = true;
});
```

### React 19 dev bundle throws `ReactSharedInternals.S` is undefined

react + react-dom version mismatch. Pin both to the same major:
```bash
bun add -d react@18 react-dom@18
```

## Build / bundling

### Vite: `Module "node:fs" has been externalized for browser compatibility`

Imported from `@cypher-zk/sdk/node` in browser code. Stop importing from `/node` — use the core subpath.

### Next.js: `Buffer is not defined`

```ts
"use client";
import { Buffer } from "buffer";
if (typeof window !== "undefined") (window as never).Buffer ??= Buffer;
```

Plus `next.config.js`:
```js
webpack: (config) => {
  config.resolve.fallback = { ...config.resolve.fallback, crypto: false, fs: false, stream: false };
  return config;
},
```

### Rollup: `circular dependency between chunks`

`react/src/*` imports from a barrel that re-exports the same module. Import from the specific file:
```ts
// ❌ import type { MarketAccount } from "../../src/accounts/index.ts";
// ✅ import type { MarketAccount } from "../../src/accounts/market.ts";
```

## On-chain `CypherError` codes (6000–6044)

| Code | Name | Cause | Frontend fix |
| --- | --- | --- | --- |
| 6000 | `MarketNotActive` | Market closed/resolved/cancelled | Refetch + re-render with `marketPhase` |
| 6003 | `AlreadyResolved` | Race on resolve | Refetch market (now resolved) |
| 6005 | `UnauthorizedResolver` | Caller ≠ market.resolver | Only the pinned resolver can resolve |
| 6007 | `AlreadyClaimed` | Race on claim | Refetch position |
| 6009 | `BetTooSmall` | Amount < market.minBet | Validate client-side first |
| 6023 | `ResolutionDeadlinePassed` | Resolver too late | Offer `claimRefund` instead |
| 6026 | `ClaimPeriodExpired` | Past claim deadline | Position forfeited |
| 6027 | `RefundPeriodExpired` | Past refund deadline | Position forfeited |
| 6029 | `ComputationVerificationFailed` | Arcium callback failed | Not recoverable — report tx sig |
| 6033 | `WrongMint` | User's ATA is wrong mint | Read mint from `GlobalState.acceptedMint` |
| 6036 | `InvalidChallengePeriod` | `challengePeriod` outside 24h–48h | Use `MIN_CHALLENGE_PERIOD_SECS` ≤ value ≤ `MAX_CHALLENGE_PERIOD_SECS` |
| 6037 | `BondTooSmall` | `bondAmount < MIN_CREATOR_BOND` ($20 USDC) | Pass `bondAmount >= 20_000_000n`; no upper bound |
| 6038 | `NotPendingResolution` | flag / finalize / override called on wrong state | Only call while `market.state === PendingResolution` (4) |
| 6039 | `ChallengePeriodNotElapsed` | finalize called too early | Wait until `now > market.challengeDeadline` |
| 6040 | `ChallengePeriodElapsed` | flag called after the window closed | Window closed — no further flags accepted |
| 6041 | `MarketDisputed` | finalize called on a flagged market | Admin must call `adminOverrideResolution` first |
| 6042 | `MarketNotDisputed` | admin override called on a non-disputed market | Only valid when `market.disputed === true` |
| 6043 | `MarketNotDisputed` | Same as above (historical IDL artifact) | Treat same as 6042 |
| 6044 | `UnauthorizedUser` | Caller ≠ position user | Ensure wallet matches the position owner |

Use `parseCypherError(err)` to extract the code; full list in `CypherErrorCode` enum.

## Devnet smoke

### `payer has 0 lamports`

```bash
solana airdrop 1 ~/.config/solana/devnet.json --url devnet
```

### Read-only passes but write skips

Expected — write gated by `DEVNET_KEYPAIR=…`.

### `Could not find ATA for mint`

```bash
spl-token create-account <ACCEPTED_MINT> --url devnet
spl-token mint <ACCEPTED_MINT> 100 --url devnet
```

## Performance

### `useMarkets()` slow

`getProgramAccounts` is slow on public RPCs. Use Helius/QuickNode/Triton, filter aggressively (`{ state: MarketState.Active }`), or pass `{ ids }` to fetch only the page you're rendering (0.8.8+). For very large deployments, move to an off-chain indexer.

### `getMultipleAccountsInfo` rejects with "Too many account keys" or "input too long"

**Cause**: SDK < 0.8.8 calls `connection.getMultipleAccountsInfo` directly with the full pubkey list. Solana caps that at 100 keys per request, so once a deployment has more than ~100 markets, `fetchMarketQuestions` blows past it.

**Fix**: upgrade to `@cypher-zk/sdk@^0.8.8` — the SDK now chunks bulk reads at 100, runs with bounded concurrency, and retries transient errors. The high-level entry points are `client.markets.byIds(ids)`, `client.marketQuestions.fetchMany(markets)`, and the React `useMarketQuestions(markets)` hook. Tune via `CypherClientOptions.rpcOptions = { concurrency, retries, … }`.

### Mutation toast says "readonlyWallet cannot sign transactions" or "Wallet is read-only"

**Cause**: a write hook (`usePlaceBet`, `useClaimPayout`, …) fired against a `CypherClient` whose wallet adapter hadn't finished hydrating. The provider was still on its `readonlyWallet` fallback.

**Fix**: gate the calling button on the wallet adapter's "ready" state. With Privy that means `usePrivy().ready && authenticated && useWallets().wallets[0]`. In `onError`, detect the case with the typed error (0.8.9+) and show a connect-wallet prompt:
```ts
import { ReadonlyWalletError } from "@cypher-zk/sdk";

usePlaceBet({
  onError: (err) => {
    if (err instanceof ReadonlyWalletError) {
      toast.error("Connect your wallet to place a bet.");
      return;
    }
    toast.error(err.message);
  },
});
```

On older SDKs you'll see the generic `Error("readonlyWallet cannot sign transactions")` — match on the substring as a fallback.

### `pollEvents` slow

Fetches each tx individually. For high-throughput indexing, use `getSignaturesForAddress` + batched `getTransactions` directly.

### `placeBet` takes >30s on devnet

Typical end-to-end: localnet 5–8s, devnet 15–30s. Bump `timeoutMs` if hitting 120s default.

## Multi-outcome & question display

### Market title shows `[Alice|Bob|Carol]` suffix in the UI

**Cause**: raw question string rendered without stripping the embedded option labels.

**Fix**:
```ts
import { parseEmbeddedOptions } from "@cypher-zk/sdk";
const { displayQuestion } = parseEmbeddedOptions(rawQuestion);
// render displayQuestion, not rawQuestion
```

### MultiOutcome market shows 100+ outcomes or buttons labelled "Outcome 1 / Outcome 2 / …"

Two separate causes:

**100+ outcomes** → decoding a v1/v2 legacy account with a raw Anchor coder. The SDK
handles v1/v2 transparently via an explicit byte-walker. Stop decoding raw bytes manually.

**"Outcome 1..N"** → market was created without the `[A|B|C]` suffix, so `getMarketOptionLabels`
falls back to generic names. This is expected for older markets. For new markets, embed labels on creation:
```ts
const onChainQuestion = `${question} [${options.join("|")}]`;
```

### Legacy market (v1/v2) shows blank question

**Cause**: `fetchMarketQuestions` returns no entry for v1/v2 markets — they have no `MarketQuestion` PDA.

**Fix**: fall back to `MarketAccount.inlineQuestion`:
```ts
const rawQuestion =
  questionMap?.get(pda) ||
  account.inlineQuestion ||   // populated for v1/v2; "" for v3
  `Market #${account.marketId}`;
```

### `cancelMarket` throws "N bet(s) placed — cannot cancel" or "state is X (only Active markets can be cancelled)"

`cancelMarketAction` runs `cancelEligibility` before sending the tx,
giving a clean error instead of a raw Anchor `ConstraintSeeds` failure.
Check eligibility in the UI before even showing the button:
```ts
import { cancelEligibility } from "@cypher-zk/sdk";
const { ok, reason } = cancelEligibility(market);
// ok === false → show reason to the user, disable the cancel button
```

---

## Positions tab is empty / `useUserPositions` returns `[]` despite on-chain accounts existing

**Symptom**: `getProgramAccounts` confirms positions exist on-chain
(visible in the network tab) but `useUserPositions(user)` / `fetchUserPositions`
returns an empty array.

**Root cause (two-part bug, both fixed)**:

1. **Padding wasn't in the IDL** (pre-program upgrade): on-chain
   `ENCRYPTED_POSITION_SPACE` reserved 5 trailing padding bytes but the
   Rust struct didn't declare them, so the IDL was 5 bytes short. Anchor's
   coder threw `"offset is out of range. Received 200"` on every 216-byte
   account. Same shape for `LPPosition` (6-byte tail).
2. **Anchor's `.all()` is all-or-nothing** (pre-SDK 0.8.5): devnet has
   two live `EncryptedPosition` layouts — current (216 bytes, with
   `bet_index`) and legacy (208 bytes, pre-`bet_index`). Even after the
   IDL was fixed, `program.account.encryptedPosition.all(...)` would
   throw on the first legacy account it encountered and abort the entire
   query — turning "wallet has 1 current + 15 legacy positions" into
   "wallet has 0 positions" silently.

**Fix landed in SDK `0.8.5` + program upgrade**:

- Rust structs now declare `pub _padding: [u8; 5]`
  (and `[u8; 6]` for `LPPosition`) as the final field; the IDL describes
  the full account layout.
- `fetchPosition` / `fetchUserPositions` / `fetchPositionsForMarket` now
  go through raw `getProgramAccounts` + per-account size-aware decode:
  Anchor's `coder.accounts.decode("encryptedPosition", buf)` for 216-byte
  accounts, hand-walked borsh for 208-byte legacy accounts (`betIndex`
  defaults to `0n`), and orphan sizes are skipped. Both layouts coexist
  in one query.

**If you see this symptom on `>=0.8.5`**:

1. Confirm the wallet actually owns positions: `getProgramAccounts` with
   the `EncryptedPosition` discriminator + a `memcmp` on offset 8 for the
   user pubkey. Account sizes should be 208 or 216 — anything else is the
   next layout bump and needs a SDK update.
2. Stale bundled IDL — re-sync with `bun sync:idl` from the SDK root.
3. React Query cached the empty result before the wallet connected —
   invalidate `positionKeys.byUser(user)` after wallet connection.

---

## v0.2: claimPayout rejected on a market that's clearly resolved

**Symptom**: `market.state === 4` (PendingResolution) and `claimPayout`
throws `NotResolved` or the SDK pre-flight error "call
finalizeResolution first".

**Cause**: v0.2 added a challenge window. Reveal callbacks now leave
the market in `PendingResolution`, not `Resolved`. The market needs
either `finalizeResolution` (anyone, after the challenge window
elapses undisputed) or `adminOverrideResolution` (admin, on a
disputed market) before claims open.

**Fix**: gate the Claim button on `marketPhase(market) === "claimable"`.
Show a "Finalize" button when phase is `"awaitingFinalize"` and a
"Pending challenge window" label when phase is `"pendingResolution"`.

## When you can't diagnose

Capture: (1) tx signature from `submitting` stage, (2) computation offset from progress event, (3) `client.cluster.name` + `client.connection.rpcEndpoint`, (4) `cat node_modules/@cypher-zk/sdk/package.json | grep version`. File at [github.com/cypher-zk/cypher-sdk/issues](https://github.com/cypher-zk/cypher-sdk/issues).
