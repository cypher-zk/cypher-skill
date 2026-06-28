# Recipe — Devnet integration test

End-to-end test against real devnet. No localnet, no bootstrapping — just
a funded keypair with SOL + CSDC (min $40 CSDC for two creator bonds).

```bash
DEVNET=1 \
DEVNET_RPC=<your-rpc-url> \
DEVNET_KEYPAIR=~/.config/solana/id.json \
DEVNET_PLACE_BET=1 \
  bun test tests/devnet/e2e.test.ts --timeout 600000
```

```ts
// tests/devnet/e2e.test.ts
import { describe, test, expect, beforeAll } from "bun:test";
import { Connection, Keypair, PublicKey } from "@solana/web3.js";
import { readFileSync, existsSync } from "node:fs";
import { resolve } from "node:path";
import { homedir } from "node:os";
import {
  CypherClient,
  CLUSTERS,
  MarketState,
  MarketType,
  MarketCategory,
  keypairToWallet,
  readonlyWallet,
  type ActionProgressEvent,
} from "@cypher-zk/sdk";

const DEVNET    = process.env.DEVNET === "1";
const PLACE_BET = process.env.DEVNET_PLACE_BET === "1";

function loadKeypairOrNull(): Keypair | null {
  const path = process.env.DEVNET_KEYPAIR;
  if (!path) return null;
  const expanded = path.startsWith("~") ? resolve(homedir(), path.slice(2)) : path;
  if (!existsSync(expanded)) return null;
  return Keypair.fromSecretKey(Uint8Array.from(JSON.parse(readFileSync(expanded, "utf8"))));
}

const WRITE_KP = DEVNET ? loadKeypairOrNull() : null;

function makeClient(kp?: Keypair): CypherClient {
  const rpc = process.env.DEVNET_RPC ?? CLUSTERS.devnet.rpc;
  return new CypherClient({
    connection: new Connection(rpc, "confirmed"),
    wallet: kp ? keypairToWallet(kp) : readonlyWallet(Keypair.generate().publicKey),
    cluster: "devnet",
  });
}

function closeTime(secondsFromNow: number): bigint {
  return BigInt(Math.floor(Date.now() / 1000) + secondsFromNow);
}

// ── YesNo market ──────────────────────────────────────────────────────────────

describe.skipIf(!DEVNET || !WRITE_KP)("devnet e2e: create YesNo market", () => {
  let client: CypherClient;
  let createdMarketId: bigint;
  let createdMarketPda: PublicKey;
  const QUESTION = "Will BTC hit $200k before end of 2025?";

  beforeAll(() => { client = makeClient(WRITE_KP!); });

  test("createMarket returns valid signature + PDA", async () => {
    const result = await client.actions.createMarket({
      creator: WRITE_KP!.publicKey,
      question: QUESTION,
      closeTime: closeTime(3600),
      category: MarketCategory.Crypto,
    });
    expect(result.signature.length).toBeGreaterThan(0);
    expect(result.marketId).toBeGreaterThanOrEqual(0n);
    createdMarketId = result.marketId;
    createdMarketPda = result.marketPda;
  }, 30_000);

  test("fetch by ID returns correct fields", async () => {
    const m = await client.markets.fetch(createdMarketId);
    expect(m!.marketType).toBe(MarketType.YesNo);
    expect(m!.state).toBe(MarketState.Active);
    expect(m!.category).toBe(MarketCategory.Crypto);
    expect(m!.creator.equals(WRITE_KP!.publicKey)).toBe(true);
    expect(Number(m!.totalBetsCount)).toBe(0);
  }, 15_000);

  test("marketQuestions.fetch returns the question string", async () => {
    const mq = await client.marketQuestions.fetch(createdMarketPda);
    expect(mq!.question).toBe(QUESTION);
  }, 15_000);

  test("marketQuestions.fetchMany batch includes the new market", async () => {
    // 0.8.8+: use the client namespace (routes through the chunked +
    // retried RPC layer). Equivalent to the loose `fetchMarketQuestions`
    // helper, just exposed on the client surface.
    const entry = { publicKey: createdMarketPda };
    const map = await client.marketQuestions.fetchMany([entry]);
    expect(map.get(createdMarketPda.toBase58())).toBe(QUESTION);
  }, 15_000);

  test.skipIf(!PLACE_BET)("placeBet on YES and verify position", async () => {
    const result = await client.actions.placeBet({
      payer: WRITE_KP!.publicKey,
      user:  WRITE_KP!.publicKey,
      marketId: createdMarketId,
      side: 1,
      amountUsdc: 5_000_000n,
      onProgress: (e: ActionProgressEvent) =>
        console.log(`[bet] ${e.stage}`, e.message ?? ""),
    });
    expect(result.computation.status).toBe("finalized");
    expect(result.position!.market.equals(createdMarketPda)).toBe(true);
    expect(result.position!.netAmount).toBeGreaterThan(0n);

    const pos = await client.positions.fetch(createdMarketPda, WRITE_KP!.publicKey);
    expect(pos!.encryptedAmount.length).toBe(32);

    const m = await client.markets.fetch(createdMarketId);
    expect(Number(m!.totalBetsCount)).toBe(1);
  }, 300_000);
});

// ── MultiOutcome market ───────────────────────────────────────────────────────

describe.skipIf(!DEVNET || !WRITE_KP)("devnet e2e: create MultiOutcome market", () => {
  let client: CypherClient;
  let createdMarketId: bigint;
  let createdMarketPda: PublicKey;
  const QUESTION = "Which team will win the 2025 FIFA Club World Cup?";

  beforeAll(() => { client = makeClient(WRITE_KP!); });

  test("createMarketMulti returns valid signature + PDA", async () => {
    const result = await client.actions.createMarketMulti({
      creator: WRITE_KP!.publicKey,
      question: QUESTION,
      closeTime: closeTime(3600),
      category: MarketCategory.Sports,
      numOutcomes: 4,
    });
    expect(result.marketId).toBeGreaterThanOrEqual(0n);
    createdMarketId = result.marketId;
    createdMarketPda = result.marketPda;
  }, 30_000);

  test("fetch returns MultiOutcome type with numOutcomes=4", async () => {
    const m = await client.markets.fetch(createdMarketId);
    expect(m!.marketType).toBe(MarketType.MultiOutcome);
    expect(m!.numOutcomes).toBe(4);
    expect(m!.state).toBe(MarketState.Active);
  }, 15_000);

  test("marketQuestions.fetch returns the question string", async () => {
    const mq = await client.marketQuestions.fetch(createdMarketPda);
    expect(mq!.question).toBe(QUESTION);
  }, 15_000);

  test.skipIf(!PLACE_BET)("placeBet on outcome 0 and verify position", async () => {
    const result = await client.actions.placeBet({
      payer: WRITE_KP!.publicKey,
      user:  WRITE_KP!.publicKey,
      marketId: createdMarketId,
      side: 0,
      amountUsdc: 5_000_000n,
      onProgress: (e: ActionProgressEvent) =>
        console.log(`[bet] ${e.stage}`, e.message ?? ""),
    });
    expect(result.computation.status).toBe("finalized");
    expect(result.userKeypair.privateKey.length).toBe(32);

    const positions = await client.positions.forMarket(createdMarketPda);
    expect(positions.length).toBe(1);
  }, 300_000);
});
```

## Pre-requisites

- Funded keypair: `solana balance ~/.config/solana/id.json --url devnet`
- CSDC balance: at least $40 (2× creator bond). Airdrop from the Cypher faucet if needed.
- A private RPC is strongly recommended — public devnet RPCs throttle `getProgramAccounts`.
