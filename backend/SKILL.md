---
name: cypher-backend
description: >
  Write Node.js/Bun scripts and services that use @cypher-zk/sdk:
  keypair-based signing, creating markets programmatically, off-chain
  event indexers, automated resolver/finalizer bots, devnet smoke tests.
  Targets Bun/Node. SKIP: browser/React UI (use ../frontend/SKILL.md),
  contract or circuit changes.
metadata:
  parent_skill: cypher
  version: "0.4.1"
---

# Cypher SDK — Node.js / Scripts

This sub-skill covers every server-side and scripting use case. For
browser/React work see [`../frontend/SKILL.md`](../frontend/SKILL.md).
For the full API reference see [`../references/api-map.md`](../references/api-map.md).

## When you're in Node land

- Keypair-based market creation (creator bot, seeding script)
- Automated oracle resolver bot
- Off-chain event indexer persisting to a DB
- Devnet smoke tests / CI integration checks
- Any script that signs with a `Keypair` from disk

## Keypair signers

Wrap a `Keypair` so it satisfies the SDK's `Wallet` interface:

```ts
import { Keypair, Connection } from "@solana/web3.js";
import { readFileSync } from "node:fs";
import { homedir } from "node:os";
import { resolve } from "node:path";
import { CypherClient, keypairToWallet, CLUSTERS } from "@cypher-zk/sdk";

function loadKeypair(path = "~/.config/solana/id.json"): Keypair {
  const expanded = path.startsWith("~")
    ? resolve(homedir(), path.slice(2))
    : path;
  return Keypair.fromSecretKey(
    Uint8Array.from(JSON.parse(readFileSync(expanded, "utf8"))),
  );
}

const kp = loadKeypair(process.env.KEYPAIR_PATH);
const client = new CypherClient({
  connection: new Connection(process.env.RPC_URL ?? CLUSTERS.devnet.rpc, "confirmed"),
  wallet: keypairToWallet(kp),
  cluster: "devnet",
});
```

`keypairToWallet` sets `wallet.payer = keypair` so the provider
auto-signs — no `signers` array needed on `sendAndConfirmTransaction`.

## Creating a market from a script

```ts
import { MarketCategory } from "@cypher-zk/sdk";

const result = await client.actions.createMarket({
  creator: kp.publicKey,
  question: "Will ETH hit $10k before 2026?",
  closeTime: BigInt(Math.floor(Date.now() / 1000) + 7 * 24 * 3600),
  category: MarketCategory.Crypto,
  // bondAmount defaults to MIN_CREATOR_BOND ($20 USDC)
});

console.log(`Market #${result.marketId} → ${result.signature}`);
// Fetch the question back (it lives in a separate MarketQuestion account)
const mq = await client.marketQuestions.fetch(result.marketPda);
console.log("question:", mq?.question);
```

For MultiOutcome markets, **embed the option labels into the question string** before creating.
`getMarketOptionLabels` on the frontend reads them back from this suffix:

```ts
import { MAX_QUESTION_BYTES } from "@cypher-zk/sdk";

const options = ["Lakers", "Celtics", "Heat", "Bucks"];
const onChainQuestion = `Who wins the NBA Finals? [${options.join("|")}]`;
if (new TextEncoder().encode(onChainQuestion).length > MAX_QUESTION_BYTES) {
  throw new Error("Question too long");
}

await client.actions.createMarketMulti({
  creator: kp.publicKey,
  question: onChainQuestion,
  numOutcomes: options.length,
  closeTime: BigInt(Math.floor(Date.now() / 1000) + 7 * 24 * 3600),
  category: MarketCategory.Sports,
});
```

Without the `[…]` suffix, frontend `getMarketOptionLabels` falls back to `["Outcome 1", …]`.

## Event indexer pattern

Persist every on-chain event to a database, checkpointed so a crash
doesn't lose or duplicate rows:

```ts
// scripts/indexer.ts
import { Connection, Keypair } from "@solana/web3.js";
import { CypherClient, readonlyWallet, CLUSTERS } from "@cypher-zk/sdk";

const client = new CypherClient({
  connection: new Connection(process.env.RPC_URL ?? CLUSTERS.devnet.rpc, "confirmed"),
  wallet: readonlyWallet(Keypair.generate().publicKey),
  cluster: "devnet",
});

let lastSig: string | undefined = await loadCheckpoint();

setInterval(async () => {
  const events = await client.events.pollEvents({
    limit: 100,
    afterSignature: lastSig,
  });
  if (!events.length) return;

  for (const { event, signature, slot } of events.reverse()) {
    await db.transaction(async (tx) => {
      await persistEvent(tx, event, signature, slot);
      await tx.update(checkpoint).set({ lastSig: signature });
    });
  }
  lastSig = events[0].signature;
}, 5_000);

async function persistEvent(tx: any, event: any, signature: string, slot: number) {
  switch (event.name) {
    case "MarketCreatedEvent":
      // MarketCreatedEvent includes question text inline (emitted at creation time)
      return tx.insert(markets).values({
        marketId: event.data.marketId.toString(),
        creator: event.data.creator.toBase58(),
        question: event.data.question,       // available on the event
        closeTime: new Date(Number(event.data.closeTime) * 1000),
        signature, slot,
      });
    case "BetPlacedEvent":
      // amount/side stay encrypted — only odds, market, user are indexable
      return tx.insert(bets).values({
        market: event.data.market.toBase58(),
        user: event.data.user.toBase58(),
        entryOdds: event.data.entryOdds.toString(),
        nonce: event.data.nonce.toString(),
        signature, slot,
      });
  }
}
```

For lower latency use `client.events.subscribeAll(handle)` (WebSocket)
with reconnect logic:

```ts
let sub = client.events.subscribeAll(handle);
client.connection.onReconnect?.(() => {
  sub.unsubscribe();
  sub = client.events.subscribeAll(handle);
});
```

## Devnet smoke

Gate writes behind env vars to keep the read-only path safe in CI:

```bash
# Read-only (no keypair, no spend):
DEVNET=1 DEVNET_RPC=<url> bun test tests/devnet

# Full write flow (creates markets, optionally places bets):
DEVNET=1 DEVNET_RPC=<url> \
DEVNET_KEYPAIR=~/.config/solana/id.json \
DEVNET_PLACE_BET=1 \
  bun test tests/devnet --timeout 600000
```

The keypair must hold SOL + CSDC (min $40 for the creator bond on both
market types). See [examples/integration-test.md](examples/integration-test.md)
for a ready-to-run devnet test suite.

## Recipes

- [`examples/oracle-resolver.md`](examples/oracle-resolver.md) — automated resolver + finalizer bot
- [`examples/indexer.md`](examples/indexer.md) — production event indexer with DB checkpoints
- [`examples/integration-test.md`](examples/integration-test.md) — devnet smoke test suite

## Anti-hallucination checklist

- [ ] Scripts use `keypairToWallet(kp)`, never `wallet: kp` directly.
- [ ] `readonlyWallet` for read-only scripts (no signing needed).
- [ ] Indexer checkpoints in the SAME DB transaction as the event row.
- [ ] `MarketCreatedEvent.data.question` is available on events; current `MarketAccount` (v3) has no inline question field — legacy v1/v2 accounts have `inlineQuestion`.
- [ ] All event field accesses use camelCase + `bigint` (not snake_case / BN).
- [ ] Write scripts gate on `DEVNET_KEYPAIR` env var being set.
- [ ] Resolver bots call `finalizeResolution` after `market.challengeDeadline` elapses.
- [ ] MultiOutcome creation writes `[A|B|C]` suffix into the question string.
- [ ] Cancel scripts call `cancelEligibility(market)` before `cancelMarketAction` to surface clean errors.
