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

## On-chain `CypherError` codes (6000–6035)

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

Use `parseCypherError(err)` to extract the code; full list in `CypherErrorCode` enum.

## Localnet integration

### Tests hang on first `placeBet`

Arcium localnet not running or MXE has no cluster.
```bash
cd cypher-main && arcium localnet up
# Wait for "MXE active"
```

### `Cannot fetch GlobalState account`

`initialize` hasn't been called. Run `scripts/bootstrap.ts` first.

### Comp defs "already in use"

Normal between test runs — idempotent. Catch the error.

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

`getProgramAccounts` is slow on public RPCs. Use Helius/QuickNode/Triton, filter aggressively (`{ state: MarketState.Active }`), or move to an off-chain indexer.

### `pollEvents` slow

Fetches each tx individually. For high-throughput indexing, use `getSignaturesForAddress` + batched `getTransactions` directly.

### `placeBet` takes >30s on devnet

Typical end-to-end: localnet 5–8s, devnet 15–30s. Bump `timeoutMs` if hitting 120s default.

## When you can't diagnose

Capture: (1) tx signature from `submitting` stage, (2) computation offset from progress event, (3) `client.cluster.name` + `client.connection.rpcEndpoint`, (4) `cat node_modules/@cypher-zk/sdk/package.json | grep version`. File at [github.com/cypher-zk/cypher-sdk/issues](https://github.com/cypher-zk/cypher-sdk/issues).
