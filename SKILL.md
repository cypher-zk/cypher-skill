---
name: cypher
description: >
  Build apps on the Cypher privacy-preserving prediction market on Solana
  using @cypher-zk/sdk. Use when wiring a frontend (React or vanilla) to
  read markets/positions, place private encrypted bets, resolve, claim,
  or subscribe to events; when writing Node/admin scripts (initialize,
  init comp defs, IDL sync, indexers); when handling Arcium MPC callback
  flows; when persisting the per-bet x25519 user secret; when computing
  fees/deadlines/phases; or when debugging Anchor + Arcium boundary bugs
  (encryption param order, snake_case event fields, circular barrel
  imports, claim-phase mismatches). SKIP: generic Solana prediction-
  market designs unrelated to Cypher; Token-2022 confidential transfers;
  raw Arcium circuit authoring (use the bundled `arcium` skill instead);
  smart-contract changes to cypher-main.
license: MIT
metadata:
  author: cypher-zk
  version: "0.2.0"
  sdk_package: "@cypher-zk/sdk"
  sdk_repo: "https://github.com/cypher-zk/cypher-sdk"
  contract_program_id: "cyphPe923pnPGVXJL3a3P7t2W9mJsagBcg1oeauoh2B"
---

# Cypher SDK

Privacy-preserving prediction market on Solana. User bets stay encrypted
on chain via [Arcium MPC](https://docs.arcium.com); the SDK
(`@cypher-zk/sdk`) wraps the encrypt → submit → await callback → refetch
choreography behind one async `client.actions.placeBet({...})` call with
per-stage progress events.

> **Targets `@cypher-zk/sdk@0.1.x`** which pins `@anchor-lang/core` 1.0.2,
> `@solana/web3.js` 1.95.x, and `@arcium-hq/client` 0.10.x. Always verify
> the installed version with `cat node_modules/@cypher-zk/sdk/package.json`
> before reasoning about API shape.

## Three coupled surfaces

Most bugs live in the gap between two of these. Read whichever surface
matches the failure.

| Surface | Lives in | Failure mode you see |
| --- | --- | --- |
| **Circuit** (Arcis MPC logic) | `cypher-main/encrypted-ixs/src/lib.rs` | MPC computation aborts / returns garbage / never finalizes |
| **Program** (Anchor on-chain) | `cypher-main/programs/cypher_main/src/lib.rs` | Tx rejected with a `CypherError` |
| **Client** (this SDK) | `@cypher-zk/sdk/src/` | Type error, snake_case key mismatch, wrong PDA, undefined data field |

You only need to touch the client surface from inside this skill. Circuit
or program changes go in `cypher-main` — see the bundled `arcium` skill
for that.

## When to use this skill vs. the `arcium` skill

| Task | Skill |
| --- | --- |
| Add a `usePlaceBet` hook to a Next.js app | **this skill** |
| Show a loader that updates per Arcium stage | **this skill** |
| Decode `BetPlacedEvent` from a tx receipt | **this skill** |
| Subscribe to live market resolution events | **this skill** |
| Write a Node script that initializes all 8 comp defs | **this skill** |
| Add a new Arcis circuit / change `place_private_bet_yesno` | **`arcium` skill** |
| Change a PDA seed in `programs/cypher_main/` | **`arcium` skill** |
| Debug an `ArgBuilder` ordering bug in cypher-main tests | **`arcium` skill** |

## Intent router

Pick the row that matches your task. Each row points at a single file you
should read before writing code.

| Intent | Read this |
| --- | --- |
| First-time setup, install, connect to devnet | [frontend/SKILL.md → "Set up a client"](frontend/SKILL.md#1-set-up-a-client) |
| Place a private bet from a browser | [frontend/SKILL.md → "Place a private bet"](frontend/SKILL.md#3-place-a-private-bet-with-live-progress) |
| Show per-stage loader during a bet | [frontend/SKILL.md → "Progress events"](frontend/SKILL.md#progress-events) |
| Persist the user's x25519 secret | [frontend/SKILL.md → "Persisting user secrets"](frontend/SKILL.md#persisting-user-secrets) |
| Create / cancel / resolve / claim flows in React | [frontend/SKILL.md → "Hooks reference"](frontend/SKILL.md#hooks-reference) |
| Subscribe to live events (WebSocket or poll) | [frontend/SKILL.md → "Live events"](frontend/SKILL.md#live-events) |
| **v0.2+: dispute window UI (flag / finalize / admin override)** | [frontend/examples/dispute-window.md](frontend/examples/dispute-window.md) |
| **v0.2+: resolver bot that finalizes after the window** | [backend/examples/oracle-resolver.md](backend/examples/oracle-resolver.md) |
| Initialize the protocol (`initialize` + 8 × initCompDef) | [backend/SKILL.md → "Bootstrap a deployment"](backend/SKILL.md#bootstrap-a-deployment) |
| Run a Node script that uses a Keypair | [backend/SKILL.md → "Keypair signers"](backend/SKILL.md#keypair-signers) |
| Build an off-chain indexer | [backend/SKILL.md → "Event indexer pattern"](backend/SKILL.md#event-indexer-pattern) |
| Sync the IDL after a contract rebuild | [backend/SKILL.md → "Syncing the IDL"](backend/SKILL.md#syncing-the-idl) |
| Look up an SDK export / function signature | [references/api-map.md](references/api-map.md) |
| Diagnose an error / failure mode | [references/troubleshooting.md](references/troubleshooting.md) |
| Understand account topology + bet flow | [references/architecture.md](references/architecture.md) |

## NEVER (anti-hallucination)

The agent has a measurable habit of inventing these — **do not**:

- **NEVER** call `client.actions.placeBet` without persisting
  `result.userKeypair.privateKey`. Without that key the user cannot
  decrypt their position later to claim a payout. There is no on-chain
  recovery path.
- **NEVER** read event field names as snake_case. The SDK's `parseLogs`
  / `subscribeAll` / `on*` helpers all return **camelCase `bigint`**
  fields (e.g. `data.payoutAmount`, `data.entryOdds`) — even though
  Anchor's raw `BorshCoder.decode` yields `payout_amount` / BN. If a
  field is `undefined` at runtime, you typo'd the camelCase name or
  bypassed `parseLogs`.
- **NEVER** import types from the `index.ts` barrel inside
  `react/src/` — it causes Rollup chunk circularity. Import from the
  specific module (e.g. `../../src/accounts/market.ts`, not
  `../../src/accounts/index.ts`).
- **NEVER** instantiate a new `RescueCipher` per call without a fresh
  16-byte nonce. Reusing a nonce with the same key produces garbled
  output. Use `freshNonce()` or `encryptBetInput`'s default behavior.
- **NEVER** call `claim*Action` without checking `marketPhase(market)`
  first. The action does this internally, but UI logic that gates
  buttons must mirror it — see the phase table in
  [references/api-map.md](references/api-map.md#marketphase-values).
- **NEVER** hard-code `KNOWN_MINTS.devnetCSDC` as `acceptedMint` in a
  bet flow. Always read `acceptedMint` from
  `client.globalState.fetch().acceptedMint` — the SDK is
  cluster-agnostic at runtime and the constant is informational only.
- **NEVER** put the `node`-only subpath (`@cypher-zk/sdk/node`) in
  browser bundles. It depends on `node:fs` for circuit-bytecode upload
  and is admin-only.
- **NEVER** invent instruction names. v0.2+: the user-facing set is 29 —
  3 admin, 8 init comp def, 4 market lifecycle, 2 bet, 2 resolve,
  4 claim, **3 dispute (`flag_resolution`, `finalize_resolution`,
  `admin_override_resolution`)**, plus 3 program-internal callbacks.
  Source of truth: `src/idl/cypher.json`.
- **NEVER** (v0.2+) call `claimPayout` while `market.state ===
  PendingResolution` (state value 4). Reveal callbacks now flip the
  market to `PendingResolution`, not `Resolved`. Claim doesn't open
  until someone calls `finalizeResolution` (anyone, after the challenge
  window) or `adminOverrideResolution` (admin, on disputed). The SDK
  action throws a clear pre-flight error — surface it in your UI.
- **NEVER** (v0.2+) omit `challengePeriod` when calling
  `createMarketIx` / `createMarketMultiIx` directly. The high-level
  `client.actions.createMarket(...)` defaults to
  `MIN_CHALLENGE_PERIOD_SECS` (24h), but the raw builders REQUIRE it.
  Allowed range: 24h–48h. Out-of-range throws client-side.

## Core mental model

```
Frontend                  @cypher-zk/sdk                Solana / Arcium
   │
   │ user clicks "Bet"
   ├──────────────────────►│
   │                        │ ┌──────────────────────────────────────┐
   │                        │ │ 1. validate                          │
   │                        │ │ 2. fetch state (GlobalState, MXE)    │
   │                        │ │ 3. encrypt (x25519 + RescueCipher)   │
   │                        │ │ 4. send tx (queue_computation)       │ ──► Solana
   │                        │ │ 5. await Arcium callback             │ ──► Arcium MPC
   │                        │ │ 6. refetch position                  │
   │                        │ └──────────────────────────────────────┘
   │                        │
   │  result + userKeypair  │
   │◄───────────────────────┤
   │
   ▼ persist privateKey, render entryOdds
```

Each numbered step fires an `onProgress({ stage, message, signature?,
computationOffset? })` event so the UI can render fine-grained loading
state. See [frontend/SKILL.md → Progress events](frontend/SKILL.md#progress-events).

## Verification checklist (before claiming "done")

- [ ] `bun run typecheck` clean (no `any` smuggled through `as never`).
- [ ] `bun test tests/unit` green.
- [ ] Every `placeBet` callsite stores `userKeypair.privateKey`.
- [ ] Every event-field access uses camelCase, not snake_case.
- [ ] No `react/src/` file imports from `../../src/*/index.ts` barrels.
- [ ] No browser bundle imports from `@cypher-zk/sdk/node`.
- [ ] If touching action signatures: `tests/unit/actions-behavior.test.ts`
      still passes.
- [ ] If touching event decode: `tests/unit/events-roundtrip.test.ts`
      still passes.

## Where to find canonical truth

| Question | Authoritative source |
| --- | --- |
| What instructions exist? | `src/idl/cypher.json` (always rerun `bun run sync:idl` first) |
| What account fields exist + their byte offsets? | `src/idl/cypher.json` + `src/accounts/layout.ts` |
| What error codes can the program throw? | `src/errors.ts` (CypherErrorCode enum) |
| What action helpers are available? | `src/actions/index.ts` |
| What React hooks ship? | `react/src/index.ts` |
| What event shapes get emitted? | `src/events/parser.ts` (typed interfaces match runtime after camelize+normalize) |

If a docstring contradicts the IDL, **the IDL wins** — file an issue.

## Resources

- **SDK source**: `@cypher-zk/sdk` npm — installable in any TS project
- **GitHub**: [cypher-zk/cypher-sdk](https://github.com/cypher-zk/cypher-sdk)
- **Sister skill**: bundled `arcium` skill for circuit/program changes
- **Architecture doc**: [references/architecture.md](references/architecture.md)
- **Recipes**: [frontend/examples/](frontend/examples/) and [backend/examples/](backend/examples/)
- **Troubleshooting**: [references/troubleshooting.md](references/troubleshooting.md)
- **Full SDK API map**: [references/api-map.md](references/api-map.md)
