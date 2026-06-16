---
name: cypher-backend
description: >
  Build the Node/admin/scripts surface for a Cypher deployment with
  @cypher-zk/sdk: keypair-based signers, protocol bootstrap
  (`initialize` + 8 × initCompDef), IDL sync after contract rebuilds,
  off-chain event indexers, integration tests against Arcium localnet,
  devnet smoke scripts, the optional `@cypher-zk/sdk/node` subpath for
  circuit-bytecode upload. Targets Bun/Node. SKIP: browser/React UI
  (use ../frontend/SKILL.md), contract changes to cypher-main.
metadata:
  parent_skill: cypher
  version: "0.1.1"
---

# Cypher SDK — Backend

This sub-skill handles every Node/Bun/admin/script task. For browser
and React work see [`../frontend/SKILL.md`](../frontend/SKILL.md). For
SDK API reference see [`../references/api-map.md`](../references/api-map.md).

## When you're in backend land

- Writing a deploy script (`initialize` + comp-def init)
- Running an off-chain event indexer that persists to a DB
- Building an oracle/resolver bot
- Wiring CI integration tests against Arcium localnet
- Anything that touches `Keypair.fromSecretKey` or `@solana/web3.js`'s
  `Connection.requestAirdrop`
- Anything that uses `@cypher-zk/sdk/node` (circuit bytecode upload)

## Keypair signers

Wrap a `Keypair` so it satisfies the SDK's `Wallet` interface:

```ts
import { Keypair } from "@solana/web3.js";
import { readFileSync } from "node:fs";
import { homedir } from "node:os";
import { resolve } from "node:path";
import { keypairToWallet } from "@cypher-zk/sdk";

function loadKeypair(path = "~/.config/solana/id.json"): Keypair {
  const expanded = path.startsWith("~")
    ? resolve(homedir(), path.slice(2))
    : path;
  const raw = JSON.parse(readFileSync(expanded, "utf8")) as number[];
  return Keypair.fromSecretKey(Uint8Array.from(raw));
}

const admin = loadKeypair();
const wallet = keypairToWallet(admin);
```

`keypairToWallet` sets `wallet.payer = keypair`, which the provider uses
to auto-sign (no `signers` array needed for the keypair itself).

## Bootstrap a deployment

A fresh deployment of `cypher-main` needs five things, in order:

1. **`initialize`** — sets fee rates, treasury, accepted mint.
2. **8 × `init_*_comp_def`** — registers the 8 Arcium circuits on chain.
3. **Upload circuit bytecode** to each comp def — uses `@cypher-zk/sdk/node`.
4. **Verify** that the MXE has a cluster assigned (Arcium-side
   handshake).
5. **Smoke test** a `createMarket` to confirm the wiring works.

```ts
// scripts/bootstrap.ts — run with `bun run scripts/bootstrap.ts`
import { Connection, Keypair, sendAndConfirmTransaction, Transaction } from "@solana/web3.js";
import {
  CypherClient,
  keypairToWallet,
  INIT_COMP_DEF_INSTRUCTIONS,
  type InitCompDefMethodName,
} from "@cypher-zk/sdk";
import { uploadCypherCircuit } from "@cypher-zk/sdk/node";

const admin = Keypair.fromSecretKey(/* … */);
const connection = new Connection(process.env.RPC_URL!, "confirmed");
const client = new CypherClient({
  connection,
  wallet: keypairToWallet(admin),
  cluster: "devnet",
});

// 1. initialize
const initIx = await client.admin.initializeIx({
  protocolFeeRateBps: 50,
  lpFeeRateBps: 150,
  admin: admin.publicKey,
  protocolTreasury: admin.publicKey,
  acceptedMint: ACCEPTED_MINT_PUBKEY,
});
await sendAndConfirmTransaction(connection, new Transaction().add(initIx), [admin]);
console.log("✓ initialize");

// 2. Init all 8 comp defs (idempotent — re-running is safe per SDK)
const lut = (await client.compDefs.buildAllInitIx({
  payer: admin.publicKey,
  addressLookupTable: LUT_ADDRESS, // see below
}))[0].ix.keys.find(k => k.pubkey)!.pubkey;

for (const methodName of Object.keys(INIT_COMP_DEF_INSTRUCTIONS) as InitCompDefMethodName[]) {
  const ix = await client.compDefs.initIx(methodName, {
    payer: admin.publicKey,
    addressLookupTable: lut,
  });
  try {
    await sendAndConfirmTransaction(connection, new Transaction().add(ix), [admin]);
    console.log(`✓ ${methodName}`);
  } catch (err) {
    // Already initialized — Anchor errors with "already in use"
    if (String(err).includes("already in use")) console.log(`= ${methodName} (existing)`);
    else throw err;
  }
}

// 3. Upload circuit bytecode for each (requires `cypher-main/build/*.arcis` on disk)
for (const [methodName, circuitName] of Object.entries(INIT_COMP_DEF_INSTRUCTIONS)) {
  await uploadCypherCircuit({
    client,
    circuitName,
    arcisPath: `../cypher-main/build/${circuitName}.arcis`,
    payer: admin,
  });
  console.log(`✓ uploaded ${circuitName}.arcis`);
}

// 4. Verify MXE has a cluster
const arciumProgram = (await import("@arcium-hq/client")).getArciumProgram(client.provider);
const mxe = await arciumProgram.account.mxeAccount.fetch(
  (await import("@arcium-hq/client")).getMXEAccAddress(client.programId),
);
if (!mxe.cluster) throw new Error("MXE has no cluster assigned — run `arcium activate`");
console.log("✓ MXE cluster:", mxe.cluster);
```

See [`examples/bootstrap.md`](examples/bootstrap.md) for the full,
production-ready version including idempotency, LUT creation, and
error recovery.

## Syncing the IDL

After **any** rebuild of `cypher-main` (whether the public surface
changed or not), re-sync the IDL into the SDK:

```bash
# Standalone (uses default ../cypher-main path)
cd cypher-sdk
bun run sync:idl

# Custom source path
bun run sync:idl --from /abs/path/to/cypher-main

# Via env var
CYPHER_MAIN_PATH=/abs/path bun run sync:idl
```

After sync, run the drift tests — they catch field reorders that would
silently break memcmp filters:

```bash
bun test tests/unit/layout.test.ts
bun test tests/unit/idl.test.ts
bun test tests/unit/errors.test.ts
```

If any of those three fail, the SDK's `accounts/layout.ts` /
`accounts/filter.ts` / `errors.ts` need a manual update.

## Event indexer pattern

Persist every event to a database, with checkpointing so a crash
doesn't lose or duplicate events:

```ts
// scripts/indexer.ts
import { Connection } from "@solana/web3.js";
import { CypherClient, readonlyWallet, parseLogs } from "@cypher-zk/sdk";
import { Keypair } from "@solana/web3.js";

const client = new CypherClient({
  connection: new Connection(process.env.RPC_URL!, "confirmed"),
  wallet: readonlyWallet(Keypair.generate().publicKey),
  cluster: "devnet",
});

// Checkpoint: last processed signature.
let lastSig: string | undefined = await loadCheckpoint();

// Poll loop — every 5s grab everything new since the checkpoint.
setInterval(async () => {
  const events = await client.events.pollEvents({
    limit: 100,
    afterSignature: lastSig,
  });
  if (events.length === 0) return;
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
      return tx.insert(markets).values({
        marketId: event.data.marketId.toString(),
        creator: event.data.creator.toBase58(),
        question: event.data.question,
        closeTime: new Date(Number(event.data.closeTime) * 1000),
        signature, slot,
      });
    case "BetPlacedEvent":
      // Bet amount/side stay encrypted on-chain; only odds + market + user
      // are indexable.
      return tx.insert(bets).values({
        market: event.data.market.toBase58(),
        user: event.data.user.toBase58(),
        entryOdds: event.data.entryOdds.toString(),
        nonce: event.data.nonce.toString(),
        signature, slot,
      });
    // … one case per event …
  }
}
```

For a WebSocket-driven indexer (lower latency, but reconnect-prone),
use `subscribeAll` with reconnect logic:

```ts
let sub = client.events.subscribeAll(handle);
client.connection.onReconnect?.(() => {
  sub.unsubscribe();
  sub = client.events.subscribeAll(handle);
});
```

## Integration tests against Arcium localnet

```ts
// tests/integration/lifecycle.test.ts
import { describe, test, expect, beforeAll } from "bun:test";
import { CypherClient, keypairToWallet } from "@cypher-zk/sdk";
import { Connection, Keypair, LAMPORTS_PER_SOL } from "@solana/web3.js";

const SHOULD_RUN = process.env.INTEGRATION === "1";

describe.skipIf(!SHOULD_RUN)("integration: full lifecycle", () => {
  let client: CypherClient;
  let admin: Keypair;

  beforeAll(async () => {
    admin = Keypair.generate();
    const connection = new Connection("http://localhost:8899", "confirmed");
    client = new CypherClient({
      connection,
      wallet: keypairToWallet(admin),
      cluster: "localnet",
    });
    // Airdrop in chunks (some validators rate-limit big drops)
    for (let i = 0; i < 5; i++) {
      const sig = await connection.requestAirdrop(admin.publicKey, 2 * LAMPORTS_PER_SOL);
      await connection.confirmTransaction(sig, "confirmed");
    }
  });

  test("createMarket → placeBet → resolve → claim", async () => {
    const create = await client.actions.createMarket({
      creator: admin.publicKey,
      question: "Test market",
      closeTime: BigInt(Math.floor(Date.now() / 1000) + 3600),
      category: 0,
    });
    expect(create.market).not.toBeNull();
    // … rest of the lifecycle
  });
});
```

Run with:

```bash
# In one terminal:
cd ../cypher-main && arcium localnet up

# In another:
INTEGRATION=1 bun test tests/integration
```

See [`examples/integration-test.md`](examples/integration-test.md) for
the full sequential lifecycle with all 10 stages.

## Devnet smoke

Useful for CI/release verification. Read-only smoke needs nothing; write
smoke needs a funded keypair:

```bash
# Read-only (always safe):
DEVNET=1 bun test tests/devnet

# Read + write (spends real SOL/CSDC):
DEVNET=1 \
DEVNET_KEYPAIR=~/.config/solana/devnet.json \
DEVNET_FULL_FLOW=1 \
  bun test tests/devnet
```

The read-only path verifies:
- Program exists and is executable on devnet
- `GlobalState` (if initialized) pins the expected mint
- A random `marketId` returns `null` cleanly
- `pollEvents` returns a `[]` or array without crashing

The write path (gated by `DEVNET_KEYPAIR`) verifies a real
`createMarket` succeeds end-to-end.

## The `@cypher-zk/sdk/node` subpath

Only one helper lives here: `uploadCypherCircuit`. It reads a `.arcis`
bytecode file from disk and uploads it to IPFS + registers the result
on chain. It depends on `node:fs`, so it's gated behind a separate
import path to keep browser bundles clean:

```ts
import { uploadCypherCircuit } from "@cypher-zk/sdk/node";
//                              ^^^^^^^^^^^^^^^^^^^^^^^
//                              Never import this in browser code.
```

If a build error says "Cannot resolve 'node:fs'" in a browser bundle,
you've leaked this subpath in.

## Recipes

- [`examples/bootstrap.md`](examples/bootstrap.md) — full protocol bootstrap script
- [`examples/sync-idl.md`](examples/sync-idl.md) — IDL sync + drift verification
- [`examples/indexer.md`](examples/indexer.md) — production event indexer with checkpoints
- [`examples/oracle-resolver.md`](examples/oracle-resolver.md) — automated market-resolution bot
- [`examples/integration-test.md`](examples/integration-test.md) — full localnet lifecycle suite

## Anti-hallucination checklist (backend-specific)

- [ ] Scripts use `keypairToWallet(kp)`, never `wallet: kp` directly.
- [ ] All `init_*_comp_def` calls are idempotent (catch "already in use").
- [ ] Indexer persists checkpoints in the SAME transaction as the event row.
- [ ] No browser bundle imports from `@cypher-zk/sdk/node`.
- [ ] Devnet smoke scripts gate writes behind `DEVNET_KEYPAIR`.
- [ ] Integration tests are gated by `INTEGRATION=1` (skipped by default).
- [ ] IDL sync runs in `prepublishOnly` before any package publish.
