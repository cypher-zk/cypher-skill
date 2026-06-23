---
name: cypher-frontend
description: >
  Build the browser-facing surface of a Cypher app with @cypher-zk/sdk:
  set up CypherClient, wire React hooks via CypherProvider, place
  encrypted bets with per-stage progress UI, persist x25519 user
  secrets, decode events, gate UI on marketPhase, integrate any
  @solana/wallet-adapter wallet. Targets Next.js / Vite / any React 18+
  setup. SKIP: Node-only admin scripts (use ../backend/SKILL.md),
  contract changes, raw circuit work.
metadata:
  parent_skill: cypher
  version: "0.4.1"
---

# Cypher SDK — Frontend

This sub-skill handles every browser/React task. For Node admin scripts
see [`../backend/SKILL.md`](../backend/SKILL.md). For SDK API reference
see [`../references/api-map.md`](../references/api-map.md).

## Mental model

Three layers, top to bottom:

```
┌────────────────────────────────────────┐
│  Your component                        │  Reads state with hooks,
│  ├─ useGlobalState()                   │  fires mutations on user
│  ├─ useMarket(id) / useMarkets()       │  action, renders progress
│  └─ usePlaceBet({ onProgress }).mutate │  from the onProgress event.
└────────────────┬───────────────────────┘
                 │
┌────────────────▼───────────────────────┐
│  @cypher-zk/sdk/react                  │  Thin wrappers around
│  ├─ CypherProvider (context)           │  client.actions.* and
│  ├─ use* (queries via TanStack Query)  │  client.markets.* with
│  └─ mutation hooks (auto-invalidate)   │  query-key invalidation.
└────────────────┬───────────────────────┘
                 │
┌────────────────▼───────────────────────┐
│  CypherClient (core)                   │  Handles encrypt → send →
│  └─ client.actions.placeBet({...})     │  await Arcium callback →
│                                         │  refetch in one async call.
└────────────────────────────────────────┘
```

The frontend never needs to know about PDA derivation, Arcium queue
accounts, RescueCipher, or x25519 — `client.actions.*` hides all of it.

## Quick start

### 1. Set up a client

```bash
bun add @cypher-zk/sdk @solana/web3.js@^1.95.4 @tanstack/react-query react
```
> **CRITICAL**: The SDK relies on Anchor and `@solana/web3.js` v1. Do **not** install `@solana/web3.js` v2.x, as it is a complete rewrite and fundamentally incompatible.

```tsx
// app/cypher-client.ts (Next.js) or src/cypher-client.ts (Vite)
import { Connection } from "@solana/web3.js";
import { CypherClient } from "@cypher-zk/sdk";

let cached: CypherClient | null = null;

export function getCypherClient(wallet: import("@cypher-zk/sdk").Wallet) {
  if (cached) return cached;
  const connection = new Connection(
    process.env.NEXT_PUBLIC_RPC_URL ?? "https://api.devnet.solana.com",
    "confirmed",
  );
  cached = new CypherClient({ connection, wallet, cluster: "devnet" });
  return cached;
}
```

> The client is cheap to construct (no RPC fires), but caching it lets
> TanStack Query share the `GlobalState` cache across components.

### 2. Mount the provider

```tsx
// app/providers.tsx
"use client";
import { CypherProvider } from "@cypher-zk/sdk/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useMemo } from "react";
import { useWallet } from "@solana/wallet-adapter-react"; // any wallet adapter
import { getCypherClient } from "./cypher-client";

const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 10_000 } },
});

export function Providers({ children }: { children: React.ReactNode }) {
  const wallet = useWallet();
  // Wallet adapters expose signTransaction/signAllTransactions/publicKey
  // — they satisfy the SDK's Wallet interface as-is.
  const client = useMemo(
    () => (wallet.publicKey ? getCypherClient(wallet as never) : null),
    [wallet.publicKey],
  );
  if (!client) return <p>Connect a wallet first.</p>;
  return (
    <QueryClientProvider client={queryClient}>
      <CypherProvider client={client}>{children}</CypherProvider>
    </QueryClientProvider>
  );
}
```

### 3. Place a private bet with live progress

This is the headline flow. Get this one right and the rest follow.

```tsx
"use client";
import { useState } from "react";
import { usePlaceBet } from "@cypher-zk/sdk/react";
import { useWallet } from "@solana/wallet-adapter-react";
import type { ActionProgressEvent } from "@cypher-zk/sdk";
import { saveSecret } from "./persist-secret"; // see § Persisting user secrets

export function BetButton({ marketId }: { marketId: bigint }) {
  const wallet = useWallet();
  const [stage, setStage] = useState<ActionProgressEvent | null>(null);

  const placeBet = usePlaceBet({
    onSuccess: ({ position, userKeypair }) => {
      // CRITICAL: persist the user's x25519 secret. Without it the user
      // can never decrypt this position to claim a payout later.
      if (position) saveSecret(position.market, userKeypair.privateKey);
    },
  });

  return (
    <>
      <button
        disabled={placeBet.isPending || !wallet.publicKey}
        onClick={() =>
          placeBet.mutate({
            payer: wallet.publicKey!,
            user: wallet.publicKey!,
            marketId,
            side: 1,                // 1 = YES, 0 = NO
            amountUsdc: 5_000_000n, // $5 (6 decimals)
            onProgress: setStage,
          })
        }
      >
        {placeBet.isPending && stage ? labelFor(stage) : "Bet $5 on YES"}
      </button>
      {placeBet.error && (
        <p className="error">{placeBet.error.message}</p>
      )}
    </>
  );
}

function labelFor(e: ActionProgressEvent): string {
  switch (e.stage) {
    case "validating":         return "Checking…";
    case "fetching-state":     return "Loading market…";
    case "encrypting":         return "Encrypting your bet…";
    case "submitting":         return "Submitting transaction…";
    case "awaiting-callback":  return "Awaiting MPC nodes…";
    case "refetching":         return "Updating position…";
    case "done":               return "Done!";
  }
}
```

## Progress events

Every multi-step action takes an optional `onProgress: (e: ActionProgressEvent) => void`.
The event shape:

```ts
interface ActionProgressEvent {
  stage:
    | "validating"        // client-side input checks
    | "fetching-state"    // GlobalState, Market, MXE pubkey, LUT
    | "encrypting"        // RescueCipher + x25519 (placeBet only)
    | "submitting"        // sending the queue tx
    | "awaiting-callback" // polling Arcium computation account (~10s)
    | "refetching"        // re-reading affected accounts
    | "done";
  message?: string;             // human-friendly description
  signature?: string;           // available from "submitting" onward
  computationOffset?: bigint;   // available from "submitting" onward
}
```

Stages per action:

| Action | Stages |
| --- | --- |
| `placeBet` | validating → fetching-state → encrypting → submitting → awaiting-callback → refetching → done |
| `resolveMarket` | validating → fetching-state → submitting → awaiting-callback → refetching → done |
| `claimPayout` / `claimRefund` | validating → fetching-state → submitting → awaiting-callback → refetching → done |
| `createMarket` / `createMarketMulti` | validating → fetching-state → submitting → refetching → done |
| `cancelMarket` (0.7.8+) | validating → fetching-state → submitting → refetching → done |
| `withdrawCreatorFunds` | (no progress callback — single-step) |

The callback is invoked synchronously inside the action. A throwing
callback is swallowed (the SDK never lets a logging callback crash an
on-chain flow), so you can wire telemetry without try/catch.

## Persisting user secrets

`placeBet` returns `userKeypair: { privateKey: Uint8Array (32 bytes),
publicKey: Uint8Array (32 bytes) }`. The **`privateKey` is the only
thing that lets the user later decrypt this specific position** — for
viewing their stake amount, for claiming payouts, etc.

Recommended pattern: encrypt the secret under the wallet's signature, key
it by the position's market PDA.

```ts
import { PublicKey } from "@solana/web3.js";

export async function saveSecret(market: PublicKey, secret: Uint8Array) {
  // Simplest pattern: localStorage keyed by market PDA + wallet pubkey.
  // Production: encrypt with a key derived from a wallet signature.
  const key = `cypher:pos:${market.toBase58()}`;
  const hex = Array.from(secret, (b) => b.toString(16).padStart(2, "0")).join("");
  localStorage.setItem(key, hex);
}

export function loadSecret(market: PublicKey): Uint8Array | null {
  const hex = localStorage.getItem(`cypher:pos:${market.toBase58()}`);
  if (!hex) return null;
  const arr = new Uint8Array(hex.length / 2);
  for (let i = 0; i < arr.length; i++) {
    arr[i] = parseInt(hex.slice(i * 2, i * 2 + 2), 16);
  }
  return arr;
}
```

To **display** the user's own decrypted position:

```ts
import { createCipher, decryptBetInput, bigIntToLeBytes, fetchMxePublicKey } from "@cypher-zk/sdk";

async function decryptMyPosition(client: CypherClient, position: EncryptedPositionAccount) {
  const secret = loadSecret(position.market);
  if (!secret) return null; // user lost the key — position is opaque
  const mxe = await fetchMxePublicKey(client);
  if (!mxe) return null;
  const cipher = createCipher(secret, mxe);
  const nonceBytes = bigIntToLeBytes(position.nonce, 16);
  return decryptBetInput(position, cipher, nonceBytes);
  // { amount: bigint, side: 0 | 1 | 2 | 3 }
}
```

## Hooks reference

All hooks live under `@cypher-zk/sdk/react`. Each query hook returns the
TanStack Query `UseQueryResult` shape; each mutation returns
`UseMutationResult`. Mutation hooks auto-invalidate the relevant query
keys on success.

`useMarkets()` returns `MarketAccount` objects which have **no `question`
field**. After the query resolves, batch-fetch questions with
`fetchMarketQuestions(client, markets)` from `@cypher-zk/sdk`. For a
single market's question use `client.marketQuestions.fetch(marketPda)`.

| Hook | Returns | Auto-invalidates on success |
| --- | --- | --- |
| `useCypherClient()` | `CypherClient` | — |
| `useGlobalState()` | `UseQueryResult<GlobalStateAccount>` | — |
| `useMarket(id)` | `UseQueryResult<MarketAccount \| null>` | — |
| `useMarkets({ creator?, state? })` | `UseQueryResult<{publicKey, account}[]>` — no question field | — |
| `useUserPositions(user)` | `UseQueryResult<{publicKey, account}[]>` — all markets | — |
| `usePosition(market, user)` | `UseQueryResult<EncryptedPositionAccount \| null>` — single pair | — |
| `usePlaceBet()` | `UseMutationResult<PlaceBetResult, Error, PlaceBetInputs>` | `marketKeys.one(marketId)` + `positionKeys.byUser(user)` |
| `useCreateMarket()` | `UseMutationResult<CreateMarketResult, ...>` | `marketKeys.all` |
| `useResolveMarket()` | `UseMutationResult<ResolveMarketResult, ...>` | `marketKeys.one(marketId)` |
| `useClaimPayout()` | `UseMutationResult<ClaimResult, ...>` | market + positions |
| `useClaimRefund()` | `UseMutationResult<ClaimResult, ...>` | market + positions |
| `useCancelMarket()` | `UseMutationResult<{signature, market}, ...>` | market + `marketKeys.all` |
| `useFlagResolution()` (v0.2+) | `UseMutationResult<ResolutionActionResult, ...>` | `marketKeys.one(marketId)` |
| `useFinalizeResolution()` (v0.2+) | `UseMutationResult<ResolutionActionResult, ...>` | `marketKeys.one(marketId)` |
| `useAdminOverrideResolution()` (v0.2+) | `UseMutationResult<ResolutionActionResult, ...>` | `marketKeys.one(marketId)` |
| `useMarketEvents(opts?)` | `CypherEvent[]` (live) | — |

Query-key factories you can hand to `queryClient.invalidateQueries()`
manually: `globalStateKeys.all`, `marketKeys.all`, `marketKeys.one(id)`,
`marketKeys.byCreator(pk)`, `marketKeys.byState(n)`,
`positionKeys.byUser(pk)`, `positionKeys.forMarket(pk)`,
`positionKeys.forPair(market, user)`.

### One position per user per market

The on-chain `EncryptedPosition` PDA is seeded by `["position", market, user_wallet]`.
A wallet can only hold **one position per market** — a second `place_private_bet_*` call
will fail with `AccountAlreadyInUse`. Always check before allowing a user to submit:

```ts
import { usePosition } from "@cypher-zk/sdk/react";

// marketPda: PublicKey, userPubkey: PublicKey | null
const { data: position, isLoading } = usePosition(marketPda, userPubkey, {
  refetchInterval: 5_000,
});
const hasBet = position != null; // null = no position, object = already bet
```

Use `isLoading` to disable your submit button while the check is in flight (e.g. on
page load). Do **not** rely only on local state — it resets on page refresh.

## Live events

```ts
// Real-time WebSocket subscription (component-scoped):
import { useMarketEvents } from "@cypher-zk/sdk/react";

function Feed() {
  const events = useMarketEvents(); // CypherEvent[] (newest last)
  return (
    <ul>
      {events.slice(-20).map((evt, i) => (
        <li key={i}>{evt.name}</li>
      ))}
    </ul>
  );
}
```

```ts
// Lower-level typed subscriptions (outside React):
const sub = client.events.onBetPlaced((data) => {
  // data: BetPlacedEvent — camelCase bigint fields
  console.log("market:", data.market.toBase58(), "odds:", data.entryOdds);
});
// later: sub.unsubscribe();

// Generic, narrowed by name:
const sub2 = client.events.subscribe("MarketResolvedEvent", (data) => {
  // data: MarketResolvedEvent
  console.log("outcome:", data.outcome, "payoutRatio:", data.payoutRatio);
});
```

For environments without WebSocket support (mobile wrappers, edge
functions), poll instead:

```ts
const recent = await client.events.pollEvents({ limit: 20 });
for (const { event, signature, slot } of recent) {
  console.log(event.name, "at slot", slot);
}
```

## Phase-gating UI

Action helpers refuse pre-flight if the market isn't in the right phase,
but the UI should mirror this so the relevant button only appears when
it's actually clickable:

```ts
import { marketPhase } from "@cypher-zk/sdk";

const phase = marketPhase(market);
// "betting" | "awaitingResolve" | "claimable" | "refundable" | "expired" | "cancelled"

const canBet           = phase === "betting";
const canResolve       = phase === "awaitingResolve";
const canFlag          = phase === "pendingResolution"; // v0.2+: anyone, during window
const canFinalize      = phase === "awaitingFinalize";  // v0.2+: anyone, window elapsed undisputed
const canAdminOverride = phase === "disputed";          // v0.2+: admin only
const canClaim         = phase === "claimable";         // only after a finalized market
const canRefund        = phase === "refundable";        // unresolved past resolution_deadline
```

## Multi-outcome markets & Option Labels

MultiOutcome markets (where `market.marketType === 1`) embed their option labels inside the question text using a bracketed suffix: `Which team will win? [Lakers|Celtics|Heat]`.

The SDK provides three helpers to abstract this away so you don't need to write regexes, and they gracefully handle both YesNo and MultiOutcome markets:

```ts
import { parseEmbeddedOptions, getMarketOptionLabels, formatOutcome } from "@cypher-zk/sdk";

// 1. Clean up the question text for display
const { displayQuestion, optionLabels } = parseEmbeddedOptions(questionText);
// Returns:
// displayQuestion: "Which team will win?"
// optionLabels: ["Lakers", "Celtics", "Heat"] (or empty if it's a YesNo market)

// 2. Get the actual labels for the betting buttons/UI
const labels = getMarketOptionLabels(market, questionText);
// If YesNo: returns ["No", "Yes"]
// If Multi: returns ["Lakers", "Celtics", "Heat"]

// 3. Format the final resolved outcome label
const result = formatOutcome(market, questionText);
// If Resolved YesNo: returns "Yes" or "No"
// If Resolved Multi: returns "Lakers", etc.
// If Unresolved: returns null
```

Always use `parseEmbeddedOptions(question).displayQuestion` instead of rendering the raw question text directly, otherwise users will see the ugly `[A|B|C]` suffix on MultiOutcome markets.

### Legacy markets and `inlineQuestion`

v1/v2 accounts (created before the current on-chain layout) have no `MarketQuestion` PDA —
`fetchMarketQuestions` returns no entry for them. Their question text lives in
`MarketAccount.inlineQuestion`. Use the full fallback chain every time you need question text:

```ts
const rawQuestion =
  questionMap?.get(pda.toBase58()) ||
  account.inlineQuestion ||
  `Market #${account.marketId}`;

const { displayQuestion } = parseEmbeddedOptions(rawQuestion);
const optionLabels = getMarketOptionLabels(account, rawQuestion); // pass raw, not stripped
```

### Creating a MultiOutcome market — embed labels in the question

When creating a MultiOutcome market, write the option labels into the question string so that
`getMarketOptionLabels` can extract them after the fact:

```ts
import { MAX_QUESTION_BYTES } from "@cypher-zk/sdk";

const onChainQuestion = `${question} [${options.join("|")}]`;
if (new TextEncoder().encode(onChainQuestion).length > MAX_QUESTION_BYTES) {
  throw new Error("Question too long after adding option labels");
}
await client.actions.createMarketMulti({
  creator: wallet.publicKey!,
  question: onChainQuestion,
  numOutcomes: options.length,
  // ...
});
```

Without this suffix, `getMarketOptionLabels` falls back to `["Outcome 1", "Outcome 2", …]`.

## Recipes

Full standalone code samples — copy, adapt, ship:

- [`examples/place-bet.md`](examples/place-bet.md) — end-to-end private bet UI with progress
- [`examples/market-list.md`](examples/market-list.md) — paginated market list with phase badges
- [`examples/claim-flow.md`](examples/claim-flow.md) — payout + refund dual-path UI
- [`examples/event-feed.md`](examples/event-feed.md) — live event feed with auto-reconnect
- [`examples/decrypt-position.md`](examples/decrypt-position.md) — decrypt and render user's own bet
- [`examples/next-app-router.md`](examples/next-app-router.md) — Next.js 14 App Router setup

## Anti-hallucination checklist (frontend-specific)

- [ ] Every `placeBet.onSuccess` persists `userKeypair.privateKey`.
- [ ] Every event field access uses **camelCase** + treats numbers as `bigint`.
- [ ] No component imports from `@cypher-zk/sdk/node`.
- [ ] Provider is mounted INSIDE `QueryClientProvider`, not above.
- [ ] `CypherClient` is constructed once (memo or module-level), not per render.
- [ ] Bet/resolve/claim buttons are gated on `marketPhase` matching.
- [ ] `acceptedMint` is never hard-coded — always read from `GlobalState`.
- [ ] Question text is always stripped before rendering: `parseEmbeddedOptions(raw).displayQuestion`.
- [ ] Option labels always come from `getMarketOptionLabels(market, rawQuestion)` — never hardcode `["YES","NO"]` for MultiOutcome markets.
- [ ] Question fallback chain used: `questionMap.get(pda) || account.inlineQuestion || "Market #N"`.
- [ ] MultiOutcome creation embeds labels: `\`${question} [${options.join("|")}]\`` before calling `createMarketMulti`.

## When something breaks

- Hook says "useCypherClient must be used within a `<CypherProvider>`":
  the component renders **above** `<CypherProvider>` in the tree. Move
  it inside.
- `placeBet` resolves with `position === null`: the chain accepted the
  bet but the post-callback refetch hasn't seen it yet. Add a `waitFor`
  or refetch on the next `useMarket` interval.
- `data.payout_amount` is `undefined`: you're reading snake_case but the
  SDK emits camelCase. Use `data.payoutAmount`.
- Stale market data after a tx: mutation hooks auto-invalidate, but if
  you went through a raw instruction builder (`client.bets.placeYesnoIx`)
  you must call `queryClient.invalidateQueries(...)` yourself.

For deeper diagnosis: [`../references/troubleshooting.md`](../references/troubleshooting.md).
