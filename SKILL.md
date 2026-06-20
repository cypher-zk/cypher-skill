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
  version: "0.4.0"
  sdk_package: "@cypher-zk/sdk"
  sdk_repo: "https://github.com/cypher-zk/cypher-sdk"
  contract_program_id: "cyphPe923pnPGVXJL3a3P7t2W9mJsagBcg1oeauoh2B"
---

# Cypher SDK

Privacy-preserving prediction market on Solana. User bets stay encrypted
on chain via Arcium MPC; the SDK (`@cypher-zk/sdk`) wraps the encrypt →
submit → await callback → refetch choreography behind one async
`client.actions.placeBet({...})` call with per-stage progress events.

> **Targets `@cypher-zk/sdk@0.7.6`**. Always verify: `cat node_modules/@cypher-zk/sdk/package.json | grep version`

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

## NEVER (anti-hallucination)

- **NEVER** call `client.actions.placeBet` without persisting
  `result.userKeypair.privateKey`. Without that key the user cannot
  decrypt their position later. There is no on-chain recovery path.
- **NEVER** read `market.question` — `MarketAccount` has no question
  field. Questions live in a separate `MarketQuestion` account. Use
  `fetchMarketQuestions(client, markets)` for batch fetching or
  `client.marketQuestions.fetch(pda)` for a single market.
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

- [ ] Every `placeBet` callsite stores `userKeypair.privateKey`.
- [ ] Every event-field access uses camelCase, not snake_case.
- [ ] Questions are fetched via `fetchMarketQuestions`, not `market.question`.
- [ ] No browser bundle imports from `@cypher-zk/sdk/node`.
- [ ] No `react/src/` file imports from `../../src/*/index.ts` barrels.
- [ ] Claim/resolve/flag buttons are gated on `marketPhase(market)`.

## API reference

- **[references/api-map.md](references/api-map.md)** — every export, organized by intent
- **[references/architecture.md](references/architecture.md)** — account topology + bet flow
- **[references/troubleshooting.md](references/troubleshooting.md)** — error codes + fixes
- **GitHub**: [cypher-zk/cypher-sdk](https://github.com/cypher-zk/cypher-sdk)
