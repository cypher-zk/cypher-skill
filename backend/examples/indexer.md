# Recipe — Production event indexer

Polls for new transactions, parses events, persists with checkpointing.
Designed to survive crashes without losing or duplicating events.

```ts
#!/usr/bin/env bun
// scripts/indexer.ts
import { Connection, Keypair } from "@solana/web3.js";
import {
  CypherClient,
  readonlyWallet,
  type CypherEvent,
} from "@cypher-zk/sdk";

const POLL_INTERVAL_MS = 5_000;
const BATCH_LIMIT = 100;

const client = new CypherClient({
  connection: new Connection(process.env.RPC_URL!, "confirmed"),
  wallet: readonlyWallet(Keypair.generate().publicKey),
  cluster: "devnet",
});

// --- checkpoint store (replace with your DB) -----------------------------
async function loadCheckpoint(): Promise<string | null> {
  return Bun.file("./.indexer-checkpoint").exists()
    ? (await Bun.file("./.indexer-checkpoint").text()).trim() || null
    : null;
}
async function saveCheckpoint(sig: string): Promise<void> {
  await Bun.write("./.indexer-checkpoint", sig);
}

// --- persistence (replace with your DB) ----------------------------------
async function persistEvent(
  event: CypherEvent,
  signature: string,
  slot: number,
): Promise<void> {
  // Single switch keeps you typesafe per event variant.
  switch (event.name) {
    case "MarketCreatedEvent":
      console.log(`[create] #${event.data.marketId} "${event.data.question}"`);
      // await db.insert(markets).values({...});
      break;
    case "BetPlacedEvent":
      console.log(
        `[bet]    market=${event.data.market.toBase58().slice(0, 6)} odds=${event.data.entryOdds}`,
      );
      // bet amount + side are encrypted; index only public fields.
      // await db.insert(bets).values({...});
      break;
    case "MarketResolvedEvent":
      console.log(
        `[resolve] market=${event.data.market.toBase58().slice(0, 6)} outcome=${event.data.outcome} ratio=${event.data.payoutRatio}`,
      );
      // await db.update(markets).set({ resolved: true, ... });
      break;
    case "PayoutClaimedEvent":
      console.log(`[payout] ${event.data.payoutAmount} → ${event.data.user.toBase58().slice(0, 6)}`);
      break;
    case "RefundClaimedEvent":
      console.log(`[refund] ${event.data.refundAmount} → ${event.data.user.toBase58().slice(0, 6)}`);
      break;
    case "MarketCancelledEvent":
      console.log(`[cancel] market=${event.data.market.toBase58().slice(0, 6)}`);
      break;
    case "CreatorWithdrawnEvent":
      console.log(`[withdraw] ${event.data.creator.toBase58().slice(0, 6)} took ${event.data.total}`);
      break;
  }
}

// --- main loop -----------------------------------------------------------
let lastSig: string | null = await loadCheckpoint();
console.log(`Starting indexer. Last checkpoint: ${lastSig ?? "(none)"}`);

let busy = false;
async function tick() {
  if (busy) return;
  busy = true;
  try {
    const events = await client.events.pollEvents({
      limit: BATCH_LIMIT,
      afterSignature: lastSig ?? undefined,
    });
    if (events.length === 0) return;

    // pollEvents returns newest-first; reverse for chronological persist.
    for (const { event, signature, slot } of events.reverse()) {
      try {
        await persistEvent(event, signature, slot);
        await saveCheckpoint(signature);
        lastSig = signature;
      } catch (err) {
        // Don't advance checkpoint on persist failure — we'll retry the
        // same batch on the next tick.
        console.error(`persist failed at ${signature}:`, err);
        return;
      }
    }
  } catch (err) {
    console.error("poll failed:", err);
  } finally {
    busy = false;
  }
}

setInterval(tick, POLL_INTERVAL_MS);
tick();

// Graceful shutdown
process.on("SIGINT", () => {
  console.log("\nShutting down. Last checkpoint:", lastSig);
  process.exit(0);
});
```

## Crash safety

- **Checkpoint advances only after persist succeeds.** If the DB write
  fails, the next tick re-processes from the same signature.
- **Batch is atomic in the loop.** A crash mid-batch means re-processing
  some signatures — design `persistEvent` to be **idempotent** (use a
  unique constraint on `signature`).
- **`busy` flag** prevents overlapping ticks if a single batch takes
  longer than `POLL_INTERVAL_MS`.

## Idempotency pattern

```sql
CREATE UNIQUE INDEX bets_signature_market_user
  ON bets (signature, market, user);

-- On insert, ignore duplicates:
INSERT INTO bets (...) VALUES (...) ON CONFLICT DO NOTHING;
```

## WebSocket alternative

For lower latency (~hundreds of ms vs poll's ~5s), use `subscribeAll`:

```ts
let sub = client.events.subscribeAll(async (event) => {
  // We get the event but not the signature/slot — fetch from logs context
  // using a different approach (Connection.onLogs gives you the signature
  // directly; the SDK wraps it but doesn't surface signature for now).
});
```

Caveat: `Connection.onLogs` provides the signature in its raw callback
but the SDK's `subscribeAll` strips it. For WS-based indexing with
signature attribution, drop down to `Connection.onLogs` directly and
call `parseLogs(logInfo.logs)` per batch.

## Backfilling history

For an empty checkpoint on a long-running program, batch in chunks of
1000 signatures at a time walking backwards:

```ts
let before: string | undefined = undefined;
let total = 0;
while (true) {
  const sigs = await client.connection.getSignaturesForAddress(
    client.programId,
    { limit: 1000, before },
    "confirmed",
  );
  if (sigs.length === 0) break;
  for (const s of sigs) {
    const tx = await client.connection.getTransaction(s.signature, {
      commitment: "confirmed",
      maxSupportedTransactionVersion: 0,
    });
    if (!tx?.meta?.logMessages) continue;
    const events = (await import("@cypher-zk/sdk")).parseLogs(tx.meta.logMessages);
    for (const event of events) await persistEvent(event, s.signature, s.slot);
    total++;
  }
  before = sigs[sigs.length - 1].signature;
  console.log(`Backfilled ${total} signatures (now at ${before.slice(0, 8)}…)`);
}
```

Set the checkpoint to the *newest* signature seen so the live indexer
picks up correctly.
