# Recipe — Full localnet integration test

End-to-end suite against Arcium localnet. Run with:

```bash
# Terminal 1
cd ../cypher-main && arcium localnet up

# Terminal 2 — once the validator is ready:
INTEGRATION=1 bun test tests/integration
```

```ts
// tests/integration/lifecycle.test.ts
import { describe, test, expect, beforeAll } from "bun:test";
import {
  Connection,
  Keypair,
  PublicKey,
  LAMPORTS_PER_SOL,
  Transaction,
  sendAndConfirmTransaction,
} from "@solana/web3.js";
import {
  CypherClient,
  keypairToWallet,
  MarketState,
  MarketType,
  CLUSTERS,
  INIT_COMP_DEF_INSTRUCTIONS,
  type InitCompDefMethodName,
} from "@cypher-zk/sdk";

const SHOULD_RUN = process.env.INTEGRATION === "1";

interface State {
  admin: Keypair;
  resolver: Keypair;
  bettor: Keypair;
  client: CypherClient;
  acceptedMint?: PublicKey;
  marketId?: bigint;
}
const s: State = {
  admin: Keypair.generate(),
  resolver: Keypair.generate(),
  bettor: Keypair.generate(),
  client: null as never,
};

describe.skipIf(!SHOULD_RUN)("integration: lifecycle", () => {
  beforeAll(async () => {
    const connection = new Connection("http://localhost:8899", "confirmed");
    s.client = new CypherClient({
      connection,
      wallet: keypairToWallet(s.admin),
      cluster: "localnet",
    });
    // Airdrop SOL (chunked to avoid rate limits).
    for (const kp of [s.admin, s.resolver, s.bettor]) {
      for (let i = 0; i < 3; i++) {
        const sig = await connection.requestAirdrop(kp.publicKey, 2 * LAMPORTS_PER_SOL);
        await connection.confirmTransaction(sig, "confirmed");
      }
    }
  });

  test("01 — cluster preset matches localnet", () => {
    expect(s.client.cluster.name).toBe("localnet");
    expect(s.client.cluster.arciumClusterOffset)
      .toBe(CLUSTERS.localnet.arciumClusterOffset);
  });

  test("02 — program is deployed", async () => {
    const info = await s.client.connection.getAccountInfo(s.client.programId);
    expect(info).not.toBeNull();
    expect(info!.executable).toBe(true);
  });

  test("03 — initialize (idempotent)", async () => {
    const existing = await s.client.globalState.fetch().catch(() => null);
    if (existing) {
      s.acceptedMint = existing.acceptedMint;
      return;
    }
    // Use a test mint created by `arcium localnet up`. If you don't have
    // one, create with spl-token CLI before running this suite.
    const TEST_MINT = new PublicKey(process.env.TEST_MINT!);
    s.acceptedMint = TEST_MINT;
    const ix = await s.client.admin.initializeIx({
      protocolFeeRateBps: 50,
      lpFeeRateBps: 150,
      admin: s.admin.publicKey,
      protocolTreasury: s.admin.publicKey,
      acceptedMint: TEST_MINT,
    });
    await sendAndConfirmTransaction(
      s.client.connection,
      new Transaction().add(ix),
      [s.admin],
    );
  });

  test("04 — init all 8 comp defs (idempotent)", async () => {
    const { lut } = await (await import("@cypher-zk/sdk")).fetchMxeLookupTable(s.client);
    for (const methodName of Object.keys(INIT_COMP_DEF_INSTRUCTIONS) as InitCompDefMethodName[]) {
      const ix = await s.client.compDefs.initIx(methodName, {
        payer: s.admin.publicKey,
        addressLookupTable: lut,
      });
      try {
        await sendAndConfirmTransaction(
          s.client.connection,
          new Transaction().add(ix),
          [s.admin],
        );
      } catch (err) {
        if (!String(err).includes("already in use")) throw err;
      }
    }
  });

  test("05 — createMarket (YesNo)", async () => {
    const result = await s.client.actions.createMarket({
      creator: s.admin.publicKey,
      acceptedMint: s.acceptedMint!,
      question: "Will integration tests pass?",
      closeTime: BigInt(Math.floor(Date.now() / 1000) + 600),
      category: 0,
    });
    s.marketId = result.marketId;
    expect(result.market).not.toBeNull();
    expect(result.market!.state).toBe(MarketState.Active);
    expect(result.market!.marketType).toBe(MarketType.YesNo);
  });

  test("06 — placeBet (YesNo)", async () => {
    // Bettor needs the accepted mint in their ATA — set that up in
    // beforeAll if running real funds. Sketched here:
    const result = await s.client.actions.placeBet({
      payer: s.bettor.publicKey,
      user: s.bettor.publicKey,
      marketId: s.marketId!,
      side: 1,
      amountUsdc: 5_000_000n,
    });
    expect(result.position).not.toBeNull();
    expect(result.computation.status).toBe("finalized");
  });

  test("07 — resolveMarket", async () => {
    // In a real test, fast-forward the validator clock past close_time
    // first (e.g. solana-test-validator --warp-slot).
    const result = await s.client.actions.resolveMarket({
      payer: s.admin.publicKey,
      resolver: s.resolver.publicKey,
      marketId: s.marketId!,
      outcomeValue: 1,
    });
    expect(result.market?.state).toBe(MarketState.Resolved);
  });

  test("08 — claimPayout", async () => {
    const result = await s.client.actions.claimPayout({
      payer: s.bettor.publicKey,
      user: s.bettor.publicKey,
      marketId: s.marketId!,
    });
    expect(result.position?.claimed).toBe(true);
  });

  test("09 — createMarketMulti", async () => {
    const result = await s.client.actions.createMarketMulti({
      creator: s.admin.publicKey,
      acceptedMint: s.acceptedMint!,
      question: "Which team wins?",
      closeTime: BigInt(Math.floor(Date.now() / 1000) + 600),
      category: 2,
      numOutcomes: 3,
    });
    expect(result.market?.marketType).toBe(MarketType.MultiOutcome);
    expect(result.market?.numOutcomes).toBe(3);
  });

  test("10 — pollEvents returns recent activity", async () => {
    const events = await s.client.events.pollEvents({ limit: 20 });
    expect(events.length).toBeGreaterThan(0);
    // Should have at least the createMarket + bet + resolve + claim from above
    const names = new Set(events.map((e) => e.event.name));
    expect(names.has("MarketCreatedEvent")).toBe(true);
    expect(names.has("BetPlacedEvent")).toBe(true);
    expect(names.has("MarketResolvedEvent")).toBe(true);
  });
});
```

## Pre-requisites for running this suite

1. **Arcium localnet up** — `cd ../cypher-main && arcium localnet up`
2. **Program deployed** to the local validator
3. **Test mint created** and `TEST_MINT` env var set to its pubkey
4. **Bettor ATA funded** with the test mint (`spl-token create-account
   <mint>` then `mint <mint> 100 <bettor>`)
5. **Comp defs initialized** — the test does this lazily, but bytecode
   upload (step 4 of [bootstrap](./bootstrap.md)) must be done
   separately (the integration test doesn't shell out to `arcium upload`)

## CI run

The full lifecycle takes ~30s on a clean localnet. In CI:

```yaml
- name: Start Arcium localnet
  run: |
    cd cypher-main
    arcium localnet up --detach
    sleep 15

- name: Deploy + bootstrap
  run: |
    cd cypher-main
    anchor deploy
    cd ../cypher-sdk
    RPC_URL=http://localhost:8899 \
    KEYPAIR_PATH=~/.config/solana/id.json \
    bun run scripts/bootstrap.ts

- name: Integration tests
  run: |
    cd cypher-sdk
    INTEGRATION=1 TEST_MINT=$(cat .test-mint) bun test tests/integration
```
