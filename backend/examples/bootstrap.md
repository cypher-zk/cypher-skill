# Recipe — Bootstrap a fresh deployment

Run **once per cluster** after deploying `cypher-main`. Initializes the
protocol, registers all 8 Arcium circuits, and uploads their bytecode.

```ts
#!/usr/bin/env bun
// scripts/bootstrap.ts
import {
  Connection,
  Keypair,
  Transaction,
  sendAndConfirmTransaction,
} from "@solana/web3.js";
import { readFileSync } from "node:fs";
import { homedir } from "node:os";
import { resolve } from "node:path";
import {
  CypherClient,
  keypairToWallet,
  INIT_COMP_DEF_INSTRUCTIONS,
  type InitCompDefMethodName,
} from "@cypher-zk/sdk";
import { uploadCypherCircuit } from "@cypher-zk/sdk/node";

/* -------------------------------------------------------- *
 *                  configurable bits                        *
 * -------------------------------------------------------- */
const RPC_URL = process.env.RPC_URL ?? "https://api.devnet.solana.com";
const CLUSTER = (process.env.CLUSTER ?? "devnet") as "devnet" | "mainnet" | "localnet";
const KEYPAIR_PATH = process.env.KEYPAIR_PATH ?? "~/.config/solana/id.json";
const ACCEPTED_MINT_PUBKEY = new (await import("@solana/web3.js")).PublicKey(
  process.env.ACCEPTED_MINT!,
);
const PROTOCOL_FEE_BPS = 50;   // 0.5 %
const LP_FEE_BPS = 150;        // 1.5 %
const ARCIS_DIR = process.env.ARCIS_DIR ?? "../cypher-main/build";

/* -------------------------------------------------------- */

function loadKeypair(p: string): Keypair {
  const expanded = p.startsWith("~") ? resolve(homedir(), p.slice(2)) : p;
  const raw = JSON.parse(readFileSync(expanded, "utf8")) as number[];
  return Keypair.fromSecretKey(Uint8Array.from(raw));
}

const admin = loadKeypair(KEYPAIR_PATH);
const connection = new Connection(RPC_URL, "confirmed");
const client = new CypherClient({
  connection,
  wallet: keypairToWallet(admin),
  cluster: CLUSTER,
});

console.log(`Admin:   ${admin.publicKey.toBase58()}`);
console.log(`Cluster: ${CLUSTER} (${RPC_URL})`);
console.log(`Mint:    ${ACCEPTED_MINT_PUBKEY.toBase58()}`);

/* -------------------------------------------------------- *
 *  1. initialize                                            *
 * -------------------------------------------------------- */
console.log("\n[1/4] initialize");
const gsExisting = await client.globalState.fetch().catch(() => null);
if (gsExisting) {
  console.log(`  = already initialized (marketCounter=${gsExisting.marketCounter})`);
} else {
  const ix = await client.admin.initializeIx({
    protocolFeeRateBps: PROTOCOL_FEE_BPS,
    lpFeeRateBps: LP_FEE_BPS,
    admin: admin.publicKey,
    protocolTreasury: admin.publicKey,
    acceptedMint: ACCEPTED_MINT_PUBKEY,
  });
  const sig = await sendAndConfirmTransaction(
    connection,
    new Transaction().add(ix),
    [admin],
  );
  console.log(`  ✓ initialized — ${sig}`);
}

/* -------------------------------------------------------- *
 *  2. discover MXE address lookup table                     *
 * -------------------------------------------------------- */
console.log("\n[2/4] discover MXE LUT");
const { fetchMxeLookupTable } = await import("@cypher-zk/sdk");
const { lut } = await fetchMxeLookupTable(client);
console.log(`  LUT: ${lut.toBase58()}`);

/* -------------------------------------------------------- *
 *  3. init all 8 comp defs (idempotent)                     *
 * -------------------------------------------------------- */
console.log("\n[3/4] init_*_comp_def (8 circuits)");
for (const methodName of Object.keys(INIT_COMP_DEF_INSTRUCTIONS) as InitCompDefMethodName[]) {
  const circuit = INIT_COMP_DEF_INSTRUCTIONS[methodName];
  const ix = await client.compDefs.initIx(methodName, {
    payer: admin.publicKey,
    addressLookupTable: lut,
  });
  try {
    const sig = await sendAndConfirmTransaction(
      connection,
      new Transaction().add(ix),
      [admin],
    );
    console.log(`  ✓ ${methodName.padEnd(36)} → ${sig.slice(0, 8)}…`);
  } catch (err) {
    const msg = String(err);
    if (msg.includes("already in use") || msg.includes("custom program error: 0x0")) {
      console.log(`  = ${methodName.padEnd(36)} (already initialized)`);
    } else {
      throw new Error(`init ${methodName} failed: ${msg}`);
    }
  }
}

/* -------------------------------------------------------- *
 *  4. upload circuit bytecode                               *
 * -------------------------------------------------------- */
console.log("\n[4/4] upload circuit bytecode");
for (const [methodName, circuitName] of Object.entries(INIT_COMP_DEF_INSTRUCTIONS)) {
  const arcisPath = resolve(ARCIS_DIR, `${circuitName}.arcis`);
  try {
    const result = await uploadCypherCircuit({
      client,
      circuitName,
      arcisPath,
      payer: admin,
    });
    console.log(`  ✓ ${circuitName.padEnd(36)} → ${result?.cid ?? "uploaded"}`);
  } catch (err) {
    console.error(`  ✗ ${circuitName} failed: ${String(err).slice(0, 120)}`);
  }
}

console.log("\nBootstrap complete.");
```

## Pre-flight checklist

Before running this:

- [ ] `cypher-main` is built (`anchor build && arcium build`)
- [ ] `cypher-main/build/<circuit>.arcis` files exist (8 of them)
- [ ] Program is deployed on the target cluster
- [ ] Admin keypair has at least 2 SOL (rent for the 8 comp def accounts + tx fees)
- [ ] An SPL token mint exists for the accepted mint (CSDC on devnet,
      USDC on mainnet, a test mint on localnet)
- [ ] `ACCEPTED_MINT` env var is set to that mint's pubkey

## Verification after bootstrap

```bash
bun -e '
import { CypherClient, readonlyWallet, ALL_CIRCUITS, compDefOffsetU32 } from "@cypher-zk/sdk";
import { getCompDefAccAddress } from "@arcium-hq/client";
import { Connection, Keypair } from "@solana/web3.js";

const client = new CypherClient({
  connection: new Connection(process.env.RPC_URL),
  wallet: readonlyWallet(Keypair.generate().publicKey),
  cluster: "devnet",
});
const gs = await client.globalState.fetch();
console.log("protocolFeeRate:", gs.protocolFeeRate);
console.log("lpFeeRate:", gs.lpFeeRate);
console.log("acceptedMint:", gs.acceptedMint.toBase58());

for (const circuit of ALL_CIRCUITS) {
  const pda = getCompDefAccAddress(client.programId, compDefOffsetU32(circuit));
  const info = await client.connection.getAccountInfo(pda);
  console.log(circuit, info ? "✓" : "✗");
}
'
```

All 8 should print `✓`. Any `✗` means the upload step needs retrying
(comp defs persist between runs; just rerun step 4).
