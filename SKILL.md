---
name: cypher
description: >
  Build apps on the Cypher privacy-preserving prediction market on Solana
  using @cypher-zk/sdk. Use when wiring a frontend (React or vanilla) to
  read markets/positions, place private encrypted bets, resolve, claim,
  or subscribe to events; when writing Node.js scripts (indexers, oracle
  resolver bots, keypair-based market creation); when handling Arcium MPC
  callback flows; when persisting the per-bet x25519 user secret; when
  computing fees/deadlines/phases; or when debugging SDK errors
  (encryption param order, snake_case event fields, circular barrel
  imports, claim-phase mismatches). SKIP: generic Solana prediction-
  market designs unrelated to Cypher; Token-2022 confidential transfers;
  raw Arcium circuit authoring; smart-contract changes.
license: MIT
metadata:
  author: cypher-zk
  version: "0.6.0"
  sdk_package: "@cypher-zk/sdk"
  sdk_repo: "https://github.com/cypher-zk/cypher-sdk"
  contract_program_id: "cyphPe923pnPGVXJL3a3P7t2W9mJsagBcg1oeauoh2B"
---

# Cypher SDK

Privacy-preserving prediction market on Solana. User bets stay encrypted
on chain via Arcium MPC; the SDK (`@cypher-zk/sdk`) wraps the encrypt →
submit → await callback → refetch choreography behind one async
`client.actions.placeBet({...})` call with per-stage progress events.

> **Targets `@cypher-zk/sdk@0.8.9`**. Always verify the actual version the user has installed in their `package.json` or `node_modules`. The 0.8.x line introduced multi-bet support (`bet_index`) — pre-0.8 codebases follow different secret-persistence and claim signatures. **0.8.5** fixes the position-decode bug where `useUserPositions` silently returned `[]` whenever the wallet had any pre-`bet_index` (208-byte) accounts alongside current (216-byte) ones — pin `>=0.8.5` if you see an empty Positions tab despite on-chain accounts existing. **0.8.6** (breaking for raw-ix callers only): `placePrivateBet*Ix` dropped the `betIndex` arg in favor of a new on-chain `user_state` PDA that auto-increments. The SDK adds the `user_state` account and reads `next_bet_index` automatically; raw-ix callers must include it. `placeBetAction` / `usePlaceBet` callers are unaffected. **0.8.8** added the RPC batching layer: `batchedGetMultipleAccountsInfo`, `CypherClientOptions.rpcOptions`, `client.markets.byIds(ids)`, `client.marketQuestions.fetchMany(markets)`, React `useMarkets({ ids })` filter and `useMarketQuestions(markets)` hook. Pin `>=0.8.8` if the user reports the "input too long" / "Too many account keys" error from `getMultipleAccountsInfo` on `fetchMarketQuestions`. **0.8.9** turned the readonly-wallet sign error into a typed `ReadonlyWalletError` (`code === "READONLY_WALLET"`) so frontends can detect the "wallet still connecting" case in mutation `onError` handlers.

## Your surface

As an app developer you only interact with the **Client** layer:

| Layer | Owned by | What you see |
| --- | --- | --- |
| Circuit (Arcium MPC) | Cypher team | Computation aborts / garbage / never finalizes |
| Program (Anchor on-chain) | Cypher team | `CypherError` codes on rejected txs |
| **Client (`@cypher-zk/sdk`)** | **You** | Type errors, wrong PDA, undefined fields, stale cache |

The circuit and program are black boxes — the SDK handles all MPC
orchestration. If a `CypherError` code appears, look it up in
[references/troubleshooting.md](references/troubleshooting.md).

## Intent router

| Intent | Read this |
| --- | --- |
| Install + connect to devnet | [frontend/SKILL.md → "Set up a client"](frontend/SKILL.md#1-set-up-a-client) |
| Place a private bet from a browser | [frontend/SKILL.md → "Place a private bet"](frontend/SKILL.md#3-place-a-private-bet-with-live-progress) |
| Show per-stage loader during a bet | [frontend/SKILL.md → "Progress events"](frontend/SKILL.md#progress-events) |
| Persist the user's x25519 secret | [frontend/SKILL.md → "Persisting user secrets"](frontend/SKILL.md#persisting-user-secrets) |
| Create / cancel / resolve / claim in React | [frontend/SKILL.md → "Hooks reference"](frontend/SKILL.md#hooks-reference) |
| Subscribe to live events (WebSocket or poll) | [frontend/SKILL.md → "Live events"](frontend/SKILL.md#live-events) |
| Dispute window UI (flag / finalize) | [frontend/examples/dispute-window.md](frontend/examples/dispute-window.md) |
| Display market list with question text | [frontend/examples/market-list.md](frontend/examples/market-list.md) |
| Automated resolver bot (Node.js) | [backend/examples/oracle-resolver.md](backend/examples/oracle-resolver.md) |
| Off-chain event indexer (Node.js) | [backend/SKILL.md → "Event indexer pattern"](backend/SKILL.md#event-indexer-pattern) |
| Keypair-based scripts (create market, etc.) | [backend/SKILL.md → "Keypair signers"](backend/SKILL.md#keypair-signers) |
| Devnet integration test | [backend/examples/integration-test.md](backend/examples/integration-test.md) |
| SDK export / function signature | [references/api-map.md](references/api-map.md) |
| Error diagnosis | [references/troubleshooting.md](references/troubleshooting.md) |
| Account topology + bet flow | [references/architecture.md](references/architecture.md) |

## AI / Agent Directives

When acting as an AI assistant using this skill to help a user, follow these strict rules:

1. **IMMEDIATE STARTUP CHECK (Verify Version)**: The very first thing you must do when loaded in a session is to verify the user's installed `@cypher-zk/sdk` version using `view_file` or `grep_search` on their `package.json` (or `node_modules`). The user might be on an older version of the SDK. If their version is older than or differs from the target version (0.8.5), you MUST notify them immediately before writing any code and account for potential breaking changes. Particular landmines: pre-0.8.0 positions were keyed by `(market, user)` only (no `bet_index`), `saveSecret(market, secret)` took two args, and `claimPayout`/`claimRefund` did not accept a `betIndex`. Pre-0.8.5 `useUserPositions`/`fetchUserPositions` returns an empty array on devnet wallets that mix legacy 208-byte and current 216-byte `EncryptedPosition` accounts — direct users to upgrade if they report empty Positions while `getProgramAccounts` shows on-chain entries.
2. **Strict Web3.js Version**: The Cypher SDK and Anchor fundamentally require `@solana/web3.js` version 1 (e.g., `^1.95.4`). **NEVER** install or suggest `@solana/web3.js` version 2 (`@solana/web3.js@latest`) as it is a complete rewrite and breaks everything.
3. **Consult Source of Truth**: When in doubt about function signatures, component props, enum values, or data structures, use `view_file` on the actual `node_modules/@cypher-zk/sdk/**/*.d.ts` files. Do not guess or hallucinate the API structure based on generic Solana/Anchor knowledge.
4. **Trace Implementation**: If a bug occurs, trace the actual SDK source code or view the comprehensive `api-map.md` rather than assuming Anchor IDL behaviors. The Cypher SDK wraps Anchor interactions heavily (especially around Arcium MPC).

## NEVER (anti-hallucination)

- **NEVER** call `client.actions.placeBet` without persisting
  `result.userKeypair.privateKey`. Without that key the user cannot
  decrypt their position later. There is no on-chain recovery path.
- **NEVER** read `market.question` — `MarketAccount` has no question
  field. Questions live in a separate `MarketQuestion` account. Use
  `client.marketQuestions.fetchMany(markets)` (0.8.8+, batched + retried)
  or `client.marketQuestions.fetch(pda)` for a single market. In React,
  use `useMarketQuestions(markets)`.
- **NEVER** read event field names as snake_case. The SDK's `parseLogs`
  / `subscribeAll` / `on*` helpers return **camelCase `bigint`** fields
  (`data.payoutAmount`, `data.entryOdds`). If a field is `undefined`,
  you used snake_case or bypassed `parseLogs`.
- **NEVER** call `claim*Action` without `marketPhase(market)` gating in
  the UI. The action pre-flights it, but buttons must mirror it — see
  the phase table in [references/api-map.md](references/api-map.md#marketphase-values).
- **NEVER** call `claimPayout` while `market.state === 4`
  (`PendingResolution`). Claims open only after `finalizeResolution`
  (anyone, after the challenge window) or `adminOverrideResolution`
  (admin, on disputed markets).
- **NEVER** hard-code `KNOWN_MINTS.devnetCSDC` as `acceptedMint`.
  Always read from `client.globalState.fetch().acceptedMint` — the SDK
  is cluster-agnostic at runtime.
- **NEVER** import `@cypher-zk/sdk/node` in browser code. That subpath
  pulls in `node:fs` and will break any browser bundler.
- **NEVER** import types from `react/src/` barrels (`index.ts`) — it
  causes Rollup chunk circularity. Import from the specific module.
- **NEVER** render a raw question string directly. MultiOutcome markets
  embed `[Option A|Option B|…]` in the on-chain question. Always strip it:
  `parseEmbeddedOptions(rawQuestion).displayQuestion`. Pass the raw string
  (unsstripped) to `getMarketOptionLabels`.
- **NEVER** hardcode `["YES","NO"]` buttons for a MultiOutcome market.
  Use `getMarketOptionLabels(market, rawQuestion)` — it returns `["NO","YES"]`
  for YesNo markets and the real option names for MultiOutcome.
- **NEVER** hardcode category, state, or market-type display strings.
  Use `marketCategoryName(c)`, `marketStateName(s)`, `marketTypeName(t)` from
  the SDK so they stay in sync with the on-chain enum.
- **NEVER** call `cancelMarketAction` without first checking
  `cancelEligibility(market)`. The action runs this check internally,
  but surfacing the reason to the user before sending a tx saves a
  round-trip.
- **NEVER** assume one position per user per market. Since 0.8.0 the
  `EncryptedPosition` PDA is seeded by `(market, user, bet_index)` — a
  wallet can hold many positions on the same market. Key persisted
  secrets by `(market, betIndex)` and pass `betIndex` to
  `claimPayout` / `claimRefund` / `usePosition` / `loadSecret`.

## Core mental model

```
Your app               @cypher-zk/sdk              Solana / Arcium
   │
   │ placeBet(...)
   ├─────────────────►│
   │                   │ 1. validate inputs
   │                   │ 2. fetch GlobalState + MXE pubkey
   │                   │ 3. encrypt (x25519 + RescueCipher)
   │                   │ 4. send tx → Solana ──────────────────► Solana
   │                   │ 5. await Arcium callback ─────────────► Arcium MPC
   │                   │ 6. refetch position
   │                   │
   │ { signature,      │
   │   userKeypair, ◄──┤  ← PERSIST privateKey
   │   position }      │
```

Each step fires `onProgress({ stage, message, signature?, computationOffset? })`.
See [frontend/SKILL.md → Progress events](frontend/SKILL.md#progress-events).

## Verification checklist

- [ ] Every `placeBet` callsite stores `userKeypair.privateKey` keyed by `(position.market, betIndex)`.
- [ ] Every event-field access uses camelCase, not snake_case.
- [ ] Question text rendered via `parseEmbeddedOptions(rawQuestion).displayQuestion` — never raw.
- [ ] Option labels rendered via `getMarketOptionLabels(market, rawQuestion)` — never hardcoded.
- [ ] Question fetching uses the full fallback: `questionMap.get(pda) || account.inlineQuestion || "Market #N"`.
- [ ] Category/state/type names use `marketCategoryName` / `marketStateName` / `marketTypeName`.
- [ ] No browser bundle imports from `@cypher-zk/sdk/node`.
- [ ] No `react/src/` file imports from `../../src/*/index.ts` barrels.
- [ ] Claim/resolve/flag buttons are gated on `marketPhase(market)`.
- [ ] MultiOutcome creation embeds labels into the question before `createMarketMulti`.

## API reference

- **[references/api-map.md](references/api-map.md)** — every export, organized by intent
- **[references/architecture.md](references/architecture.md)** — account topology + bet flow
- **[references/troubleshooting.md](references/troubleshooting.md)** — error codes + fixes
- **GitHub**: [cypher-zk/cypher-sdk](https://github.com/cypher-zk/cypher-sdk)
