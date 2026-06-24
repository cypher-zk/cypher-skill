# Cypher SDK — Complete API map

Every export from `@cypher-zk/sdk`, organized by intent. Skim left
column to find the function, click through to the file for full
JSDoc.

> **Authority**: this map mirrors `src/index.ts`. If anything here
> disagrees with the IDE's autocomplete, the IDE wins (it reads the
> shipped `.d.ts`).

## Client

| Export | Kind | Lives in |
| --- | --- | --- |
| `CypherClient` | class | `src/client.ts` |
| `CypherClientOptions` | type | `src/client.ts` |

### `new CypherClient({ connection, wallet, cluster?, commitment?, programId?, confirmOptions? })`

```ts
const client = new CypherClient({
  connection: new Connection("https://api.devnet.solana.com"),
  wallet: keypairToWallet(kp),
  cluster: "devnet", // or "mainnet" / "localnet" / a full ClusterConfig
});
```

Construction is synchronous, no RPC fires. Holds:
- `connection`, `wallet`, `provider`, `program`, `programId`, `cluster`
- Namespaces: `globalState`, `markets`, `positions`, `lpPositions`,
  `events`, `actions`, `admin`, `compDefs`, `marketIx`, `bets`,
  `resolveIx`, `claimIx`

## Cluster + constants

| Export | Kind | Value / shape |
| --- | --- | --- |
| `PROGRAM_ID` | const | `cyphPe923pnPGVXJL3a3P7t2W9mJsagBcg1oeauoh2B` |
| `KNOWN_MINTS` | const | `{ devnetCSDC, mainnetUSDC }` |
| `CLUSTERS` | const | `{ devnet, mainnet, localnet }` — see below |
| `ClusterName` | type | `"devnet" \| "mainnet" \| "localnet"` |
| `ClusterConfig` | type | `{ name, rpc, arciumClusterOffset, expectedMint }` |
| `ACCOUNT_DISCRIMINATOR_SIZE` | const | `8` |
| `MIN_BET_USDC` | const | `1_000_000n` ($1 USDC) |
| `MIN_CREATOR_BOND` | const | `20_000_000n` ($20 USDC — minimum, no upper bound) (v0.2+) |
| `DEFAULT_RESOLUTION_WINDOW_SECS` | const | `7 * 24 * 3600` |
| `DEFAULT_CLAIM_PERIOD_SECS` | const | `14 * 24 * 3600` |
| `DEFAULT_REFUND_PERIOD_SECS` | const | `14 * 24 * 3600` |
| `MAX_PROTOCOL_FEE_BPS` | const | `100` (1%) |
| `MAX_LP_FEE_BPS` | const | `500` (5%) |
| `BPS_DENOMINATOR` | const | `10_000n` |
| `ODDS_SCALE` | const | `1_000_000_000n` (1e9) |
| `MAX_QUESTION_BYTES` | const | `200` |
| `MIN_OUTCOMES_MULTI` | const | `2` |
| `MAX_OUTCOMES_MULTI` | const | `4` |
| `MIN_CHALLENGE_PERIOD_SECS` | const | `24 * 3600` (v0.2+) |
| `MAX_CHALLENGE_PERIOD_SECS` | const | `48 * 3600` (v0.2+) |
| `MARKET_SIZE_V3` | const | `393` — current layout (question in separate PDA) |
| `MARKET_SIZE_V2_INLINE_CHALLENGE` | const | `594` — legacy layout with inline question + challenge fields |
| `MARKET_SIZE_V1_INLINE_NOCHALL` | const | `577` — oldest layout with inline question, no challenge fields |
| `MarketState` | enum-like | `{ Active: 0, Closed: 1, Resolved: 2, Unresolved: 3, PendingResolution: 4 }` (v0.2+ adds 4) |
| `MarketType` | enum-like | `{ YesNo: 0, MultiOutcome: 1 }` |
| `MarketCategory` | enum-like | `{ Crypto: 0, Politics: 1, Sports: 2, Tech: 3, Economy: 4, Culture: 5, Beyond: 6 }` |

## PDA derivations

All return `[PublicKey, bumpSeed]`. All accept an optional
`programId` last argument for redeploys with different keypairs.

| Export | Seeds |
| --- | --- |
| `globalStatePda(programId?)` | `["global_state"]` |
| `marketPda(marketId, programId?)` | `["market", market_id u64 LE]` |
| `marketVaultPda(market, programId?)` | `["market_vault", market]` |
| `marketQuestionPda(market, programId?)` | `["market_question", market]` — separate question account (v0.2+) |
| `positionPda(market, user, programId?)` | `["position", market, user]` |
| `lpPositionPda(market, creator, programId?)` | `["lp-position", market, creator]` |
| `arciumSignerPda(programId?)` | `["ArciumSignerAccount"]` |

## Wallet

| Export | Kind |
| --- | --- |
| `Wallet` | type |
| `keypairToWallet(keypair)` | `(Keypair) => Wallet` |
| `readonlyWallet(publicKey)` | `(PublicKey) => Wallet` (throws on sign) |

## Fees

| Export | Kind | Use |
| --- | --- | --- |
| `computeFees(amount, { protocolFeeRateBps, lpFeeRateBps })` | function | Preview fees before a bet |
| `isBetAmountValid(amount, rates, minBet?)` | function | Validate min + non-zero net |
| `FeeRates` | type | `{ protocolFeeRateBps, lpFeeRateBps }` |
| `FeeBreakdown` | type | `{ amount, protocolFee, lpFee, netAmount }` |

```ts
const { protocolFee, lpFee, netAmount } = computeFees(5_000_000n, {
  protocolFeeRateBps: 50, lpFeeRateBps: 150,
});
// 5_000_000 * 50/10000 = 25_000  (protocol)
// 5_000_000 * 150/10000 = 75_000 (lp)
// netAmount = 4_900_000
```

## Deadlines

| Export | Kind |
| --- | --- |
| `marketPhase(market, nowSec?)` | `(MarketDeadlineInputs, bigint?) => MarketPhase` |
| `projectDeadlines(closeTime)` | `(bigint) => { closeTime, resolutionDeadline, projectedClaimDeadline, projectedRefundDeadline }` |
| `isValidCloseTime(closeTime, nowSec?)` | `(bigint, bigint?) => boolean` (≥ now + 60s) |
| `MarketPhase` | type | literal union |
| `MarketDeadlineInputs` | type | minimal Market subset |

### `MarketPhase` values

| Value | Meaning | What's clickable |
| --- | --- | --- |
| `"betting"` | Active + before `closeTime` | `placeBet` |
| `"awaitingResolve"` | Active + before `resolutionDeadline` | `resolveMarket` (resolver only) |
| `"pendingResolution"` (v0.2+) | PendingResolution + inside challenge window, not disputed | `flagResolution` (anyone) |
| `"awaitingFinalize"` (v0.2+) | PendingResolution + window elapsed, not disputed | `finalizeResolution` (anyone) |
| `"disputed"` (v0.2+) | PendingResolution + flagged | `adminOverrideResolution` (admin only) |
| `"claimable"` | Resolved + before `claimDeadline` | `claimPayout` |
| `"refundable"` | Active + past `resolutionDeadline`, before `refundDeadline` | `claimRefund` |
| `"expired"` | All deadlines elapsed | `adminClaimRemaining` (admin only) |
| `"cancelled"` | State = Closed (creator cancelled before any bets) | — |

## Question & label helpers

Helpers for safely extracting question text and option labels. Handle both YesNo and
MultiOutcome markets, and both current (v3) and legacy (v1/v2) account layouts.

| Export | Signature | Returns |
| --- | --- | --- |
| `parseEmbeddedOptions(question)` | `(string) => { displayQuestion, optionLabels }` | Strips `[A\|B\|C]` suffix |
| `getMarketOptionLabels(market, rawQuestion)` | `(MarketAccount, string) => string[]` | Canonical option labels |
| `formatOutcome(market, rawQuestion)` | `(MarketAccount, string) => string \| null` | Resolved outcome name or null |

### `parseEmbeddedOptions(question: string)`

MultiOutcome markets embed option labels directly in the on-chain question:
`"Who wins? [Alice|Bob|Carol]"`. Always strip this before rendering.

```ts
const { displayQuestion, optionLabels } = parseEmbeddedOptions(rawQuestion);
// displayQuestion: "Who wins?"           ← render this
// optionLabels: ["Alice", "Bob", "Carol"] ← or call getMarketOptionLabels
```

### `getMarketOptionLabels(market, rawQuestion)`

Returns the canonical bet-button labels for any market type.
**Pass the raw question** (with the `[…]` suffix intact), not the stripped `displayQuestion`.

```ts
const labels = getMarketOptionLabels(market, rawQuestion);
// YesNo:        ["NO", "YES"]
// MultiOutcome: ["Alice", "Bob", "Carol"]   — from embedded labels
//               ["Outcome 1", …, "Outcome N"] — fallback when no suffix
```

### `formatOutcome(market, rawQuestion)`

Returns a human-readable outcome string once the market is resolved, or `null` when unresolved.

---

## Display name helpers

Use these instead of hardcoding enum number → string mappings.

| Export | Kind | Notes |
| --- | --- | --- |
| `MARKET_STATE_NAMES` | `Readonly<Record<number, string>>` | Frozen map |
| `MARKET_TYPE_NAMES` | `Readonly<Record<number, string>>` | Frozen map |
| `MARKET_CATEGORY_NAMES` | `Readonly<Record<number, string>>` | Frozen map |
| `marketStateName(state)` | `(number) => string` | Falls back to `"Unknown(N)"` |
| `marketTypeName(t)` | `(number) => string` | Falls back to `"Unknown(N)"` |
| `marketCategoryName(c)` | `(number) => string` | Falls back to `"Unknown(N)"` |
| `cancelEligibility(market)` | `(Pick<MarketAccount, "state"\|"totalBetsCount">) => { ok, reason }` | Pre-flight before cancel |
| `marketFormatVersion(accountSize)` | `(number) => "v1" \| "v2" \| "v3" \| null` | From raw byte length |
| `isLegacyMarketAccount(accountSize)` | `(number) => boolean` | `true` for v1 (577) or v2 (594) |

### `cancelEligibility(market)`

Client-side pre-flight that mirrors the on-chain `cancel_market` requirements.
Returns `{ ok: true, reason: null }` when cancellable; otherwise `{ ok: false, reason: "..." }`.

```ts
const { ok, reason } = cancelEligibility(market);
if (!ok) throw new Error(reason!);
// "1 bet(s) placed — cannot cancel"
// "state is Resolved (only Active markets can be cancelled)"
```

---

## Errors

| Export | Kind |
| --- | --- |
| `CypherErrorCode` | const enum of 45 codes (6000–6044) |
| `CypherErrorName` | union of 45 string names |
| `CypherErrorCodeValue` | union of the numeric codes |
| `CYPHER_ERROR_MESSAGES` | `Record<CypherErrorName, string>` |
| `cypherErrorName(code)` | `(number) => CypherErrorName \| undefined` |
| `parseCypherError(err)` | `(unknown) => ParsedCypherError \| null` |
| `ParsedCypherError` | `{ code, name, msg }` |

`parseCypherError` handles every error shape we've seen in the wild:
direct `AnchorError`, `SendTransactionError.logs` containing
`custom program error: 0x…`, and plain `Error.message` strings with
`Error Number: NNNN`.

## Account fetchers

All client namespaces forward to these. Use the namespaces in app code;
import these directly only when constructing tests.

| Export | Returns |
| --- | --- |
| `fetchGlobalState(client)` | `Promise<GlobalStateAccount>` |
| `fetchMarket(client, idOrPda)` | `Promise<MarketAccount \| null>` |
| `fetchAllMarkets(client)` | `Promise<{publicKey, account}[]>` |
| `fetchMarketsByCreator(client, creator)` | `Promise<{publicKey, account}[]>` |
| `fetchMarketsByState(client, state)` | `Promise<{publicKey, account}[]>` |
| `fetchMarketQuestion(client, market)` | `Promise<MarketQuestionAccount \| null>` — single question (v0.2+) |
| `fetchMarketQuestions(client, markets)` | `Promise<Map<string, string>>` — batch, keyed by market PDA base58 (v0.2+) |
| `fetchPosition(client, market, user, betIndex?)` | `Promise<EncryptedPositionAccount \| null>` — `betIndex` defaults to `0n` |
| `fetchUserPositions(client, user)` | `Promise<{publicKey, account}[]>` — all bets across all markets |
| `fetchPositionsForMarket(client, market)` | `Promise<{publicKey, account}[]>` |
| `nextBetIndex(client, market, user)` | `Promise<bigint>` — first free `betIndex` slot for this `(market, user)` pair |
| `fetchLpPosition(client, market, creator)` | `Promise<LpPositionAccount \| null>` |
| `fetchLpPositionsByProvider(client, provider)` | `Promise<{publicKey, account}[]>` |

### Account interfaces

| Type | Key fields |
| --- | --- |
| `GlobalStateAccount` | `marketCounter`, `protocolFeeRate`, `lpFeeRate`, `acceptedMint`, `admin`, `protocolTreasury` |
| `MarketAccount` | `marketId`, `marketType`, `numOutcomes`, `state`, `closeTime`, `minBet`, `revealedPool0..3`, `payoutRatio`, `category`, `creator`, `resolver`, `creatorBond`, `bondWithdrawn`, `outcome`, `disputed`, `totalBetsCount`, `inlineQuestion`, deadlines |
| `MarketQuestionAccount` | `question` (UTF-8 string), `questionLen`, `bump` |
| `EncryptedPositionAccount` | `user`, `market`, `encryptedAmount`, `encryptedSide`, `userPubkey`, `nonce`, `entryOdds`, `netAmount`, `betIndex`, `claimed`, `computationQueued`, `bump` |
| `LpPositionAccount` | `lpProvider`, `market`, `liquidityProvided`, `feesClaimed`, `feesClaimedAmount` |

All numerics are `bigint`. All byte arrays are `Uint8Array(32)`.

**`MarketAccount.inlineQuestion`**: non-empty for v1/v2 legacy accounts (577 or 594 bytes); always `""` for
current v3 accounts (393 bytes). Current markets store questions in a separate `MarketQuestion` PDA —
use `fetchMarketQuestions(client, markets)` to batch-fetch those. For the canonical question fallback:
```ts
const rawQuestion = questionMap?.get(pda) || account.inlineQuestion || `Market #${account.marketId}`;
```

## Memcmp filter helpers

| Export | Returns |
| --- | --- |
| `marketFilters.byCreator(pk)` | `GetProgramAccountsFilter` |
| `marketFilters.byResolver(pk)` | `GetProgramAccountsFilter` |
| `marketFilters.byState(n)` | single-byte filter |
| `marketFilters.byCategory(n)` | single-byte filter |
| `marketFilters.byMarketType(n)` | single-byte filter |
| `positionFilters.byUser(pk)` | `GetProgramAccountsFilter` |
| `positionFilters.byMarket(pk)` | `GetProgramAccountsFilter` |
| `lpPositionFilters.byProvider(pk)` | `GetProgramAccountsFilter` |
| `lpPositionFilters.byMarket(pk)` | `GetProgramAccountsFilter` |
| `discriminatorFilter(accountName)` | `GetProgramAccountsFilter` |

Byte-offset constants exported for advanced filtering:
`MARKET_OFFSETS`, `ENCRYPTED_POSITION_OFFSETS`, `LP_POSITION_OFFSETS`,
`GLOBAL_STATE_OFFSETS`.

## Arcium glue

| Export | Kind |
| --- | --- |
| `createUserKeypair()` | `() => UserCryptoKeypair` |
| `deriveSharedSecret(priv, mxePub)` | `(Uint8Array, Uint8Array) => Uint8Array` |
| `freshNonce()` | `() => Uint8Array(16)` |
| `leBytesToBigInt(bytes)` | `(Uint8Array) => bigint` |
| `createCipher(priv, mxePub)` | `=> RescueCipher` |
| `fetchMxePublicKey(client)` | `=> Promise<Uint8Array \| null>` (cached) |
| `CIRCUITS` | const map of 8 circuit names |
| `ALL_CIRCUITS` | array of 8 `CircuitName`s |
| `compDefOffsetBytes(circuit)` | `=> Uint8Array(4)` |
| `compDefOffsetU32(circuit)` | `=> number` |
| `buildArciumQueueAccounts(client, {circuit, computationOffset, addressLookupTable})` | `=> ArciumQueueAccounts` |
| `fetchMxeLookupTable(client)` | `=> Promise<{lut, slot}>` (cached) |
| `encryptBetInput({amount, side}, cipher, nonce?)` | `=> EncryptedBetInput` |
| `encryptRefundInput({amount}, cipher, nonce?)` | `=> EncryptedRefundInput` |
| `decryptBetInput(enc, cipher, nonceBytes)` | `=> BetInput` |
| `bigIntToLeBytes(value, byteLength)` | `=> Uint8Array` |
| `awaitComputation(client, computationOffset, opts?)` | `=> Promise<ComputationResult>` |
| `randomComputationOffset()` | `=> bigint` (non-zero u64) |

## Instruction builders (raw)

Each returns `Promise<TransactionInstruction>`. Use the action helpers
unless you need to compose/simulate/bundle.

### Admin
- `initializeIx(client, { protocolFeeRateBps, lpFeeRateBps, admin, protocolTreasury, acceptedMint })`
- `updateAcceptedMintIx(client, { admin, newMint, newTreasury })`
- `adminClaimRemainingIx(client, { admin, marketId, protocolTreasury })`

### Market lifecycle
- `createMarketIx(client, { creator, acceptedMint, question, closeTime, category, expectedMarketId, challengePeriod, bondAmount })`
- `createMarketMultiIx(client, { ...above, numOutcomes })` — `bondAmount >= MIN_CREATOR_BOND` ($20 USDC, no upper bound)
- `cancelMarketIx(client, { creator, marketId, acceptedMint })`
- `withdrawCreatorFundsIx(client, { creator, marketId, acceptedMint })`

### Bet
- `placePrivateBetYesnoIx(client, params)` — see PlacePrivateBetParams below
- `placePrivateBetMultiIx(client, params)`

### Resolve
- `resolveMarketYesnoIx(client, { payer, resolver, marketId, computationOffset, outcomeValue, addressLookupTable })`
- `resolveMarketMultiIx(client, ...)`

### Claim
- `claimPayoutYesnoIx(client, { payer, user, marketId, acceptedMint, computationOffset, addressLookupTable })`
- `claimPayoutMultiIx(client, ...)`
- `claimRefundYesnoIx(client, ...)`
- `claimRefundMultiIx(client, ...)`

### Dispute / challenge window (v0.2+)
- `flagResolutionIx(client, { flagger, marketId })` — anyone can flag during the window
- `finalizeResolutionIx(client, { caller, marketId })` — anyone can finalize after window elapses undisputed
- `adminOverrideResolutionIx(client, { admin, marketId, outcomeValue })` — admin re-resolves a disputed market

### `PlacePrivateBetParams` (shared between yesno/multi)
- `payer, user: PublicKey`
- `marketId: bigint | number`
- `acceptedMint, protocolTreasury: PublicKey`
- `computationOffset: bigint | number`
- `betAmountUsdc: bigint | number`
- `encryptedAmount, encryptedSide, userX25519Pubkey: Uint8Array(32)`
- `nonce: bigint` (u128)
- `addressLookupTable: PublicKey`

## High-level actions

Each returns a Promise + accepts optional `onProgress` callback. Use
these for end-to-end flows.

| Export | Returns |
| --- | --- |
| `createMarketAction(client, inputs)` | `CreateMarketResult` |
| `createMarketMultiAction(client, inputs)` | `CreateMarketResult` |
| `cancelMarketAction(client, inputs)` | `{signature, market}` |
| `withdrawCreatorFundsAction(client, inputs)` | `{signature}` |
| `placeBetAction(client, inputs)` | `PlaceBetResult` (includes `userKeypair`) |
| `resolveMarketAction(client, inputs)` | `ResolveMarketResult` |
| `claimPayoutAction(client, inputs)` | `ClaimResult` |
| `claimRefundAction(client, inputs)` | `ClaimResult` |
| `flagResolutionAction(client, inputs)` (v0.2+) | `ResolutionActionResult` |
| `finalizeResolutionAction(client, inputs)` (v0.2+) | `ResolutionActionResult` |
| `adminOverrideResolutionAction(client, inputs)` (v0.2+) | `ResolutionActionResult` |
| `sendIx(client, ix, opts?)` | `Promise<signature>` |
| `sendIxAndAwaitArcium(client, ix, {computationOffset, ...})` | `Promise<{signature, computation}>` |

### Progress events

```ts
interface ActionProgressEvent {
  stage: ActionStage;
  message?: string;
  signature?: string;        // from "submitting" onward
  computationOffset?: bigint; // from "submitting" onward
}

type ActionStage =
  | "validating"
  | "fetching-state"
  | "encrypting"          // placeBet only
  | "submitting"
  | "awaiting-callback"   // Arcium MPC actions only
  | "refetching"
  | "done";
```

## Events

| Export | Kind |
| --- | --- |
| `parseLogs(logs)` | `(string[]) => CypherEvent[]` |
| `parseLogsFor<N>(logs, name)` | `(string[], N) => DataFor<N>[]` (narrowly typed) |
| `ALL_EVENT_NAMES` | `ReadonlySet<CypherEventName>` |
| `subscribeAll(connection, callback, opts?)` | `=> EventSubscription` |
| `subscribeEvent<N>(connection, name, callback, opts?)` | `=> EventSubscription` |
| `onMarketCreated`, `onBetPlaced`, … (7 helpers) | typed per-event subscribers |
| `pollEvents(connection, opts?)` | `=> Promise<PolledEvent[]>` |

### Event types (all camelCase fields, `bigint` numerics)

| Event | Key fields |
| --- | --- |
| `MarketCreatedEvent` | `marketId, marketType, category, creator, question, closeTime` |
| `BetPlacedEvent` | `market, user, encryptedAmount, encryptedSide, nonce, entryOdds` |
| `MarketResolvedEvent` | `market, outcome, revealedPool0..3, payoutRatio` |
| `MarketCancelledEvent` | `market, creator, bondReturned` |
| `CreatorWithdrawnEvent` | `market, creator, bond, lpFees, total` |
| `PayoutClaimedEvent` | `market, user, payoutAmount` |
| `RefundClaimedEvent` | `market, user, refundAmount` |
| `ResolutionFlaggedEvent` (v0.2+) | `market, flaggedBy` |
| `MarketFinalizedEvent` (v0.2+) | `market, outcome, payoutRatio` |
| `ResolutionOverriddenEvent` (v0.2+) | `market, oldOutcome, newOutcome, newPayoutRatio, admin` |

## Client namespaces (shortcuts)

Equivalent to importing the functions directly, but pre-bound to the
client instance:

| Namespace | Methods |
| --- | --- |
| `client.globalState` | `fetch({refresh?})`, `invalidate()` |
| `client.markets` | `fetch(id)`, `fetchByPda(pda)`, `all()`, `byCreator(pk)`, `byState(n)` |
| `client.marketQuestions` | `fetch(marketPda)` → `MarketQuestionAccount \| null` |
| `client.positions` | `fetch(market, user)`, `byUser(user)`, `forMarket(market)` |
| `client.lpPositions` | `fetch(market, creator)`, `byProvider(pk)` |
| `client.events` | `subscribeAll`, `subscribe`, `onMarketCreated`, …, `pollEvents`, `parseLogs` |
| `client.actions` | `createMarket`, `createMarketMulti`, `cancelMarket`, `withdrawCreatorFunds`, `placeBet`, `resolveMarket`, `claimPayout`, `claimRefund`, `flagResolution`, `finalizeResolution`, `adminOverrideResolution` (last 3 = v0.2+) |
| `client.admin` | `initializeIx`, `updateAcceptedMintIx`, `adminClaimRemainingIx` — **Cypher team only** |
| `client.compDefs` | `initIx(method, params)`, `buildAllInitIx(params)` — **Cypher team only** |
| `client.marketIx` | `createIx`, `createMultiIx`, `cancelIx`, `withdrawCreatorFundsIx` |
| `client.bets` | `placeYesnoIx`, `placeMultiIx` |
| `client.resolveIx` | `yesnoIx`, `multiIx` |
| `client.claimIx` | `payoutYesnoIx`, `payoutMultiIx`, `refundYesnoIx`, `refundMultiIx` |
| `client.resolutionIx` (v0.2+) | `flagIx`, `finalizeIx`, `adminOverrideIx` |

## React subpath (`@cypher-zk/sdk/react`)

| Export | Kind |
| --- | --- |
| `CypherProvider` | component — wraps a `CypherClient` in context |
| `useCypherClient()` | hook — read the client (throws outside provider) |
| `useGlobalState(opts?)` | query hook |
| `useMarket(id, opts?)` | query hook |
| `useMarkets({ creator?, state? }, opts?)` | query hook |
| `useUserPositions(user, opts?)` | query hook — all positions for a user across all markets |
| `usePosition(market, user, betIndex?, opts?)` | query hook — single position for a `(market, user, betIndex)` tuple (`betIndex` defaults to `0n`); returns `null` when no bet exists |
| `usePlaceBet(opts?)` | mutation hook |
| `useCreateMarket(opts?)` | mutation hook |
| `useResolveMarket(opts?)` | mutation hook |
| `useClaimPayout(opts?)` | mutation hook |
| `useClaimRefund(opts?)` | mutation hook |
| `useCancelMarket(opts?)` | mutation hook |
| `useFlagResolution(opts?)` (v0.2+) | mutation hook |
| `useFinalizeResolution(opts?)` (v0.2+) | mutation hook |
| `useAdminOverrideResolution(opts?)` (v0.2+) | mutation hook |
| `useMarketEvents(opts?)` | subscription hook (returns `CypherEvent[]`) |
| `globalStateKeys`, `marketKeys`, `positionKeys` | query-key factories — `positionKeys.byUser(pk)`, `positionKeys.forMarket(pk)`, `positionKeys.forPair(market, user, betIndex?)` |
| `UseMarketsFilter` | type |

## Node subpath (`@cypher-zk/sdk/node`)

This subpath exists but is **not needed by app developers**. It is currently an empty placeholder
for future Node-only admin helpers. Never import it in browser code — it will break
any browser bundler. See
[backend/references/node-only-apis.md](../backend/references/node-only-apis.md).
