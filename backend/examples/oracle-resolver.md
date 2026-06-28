# Recipe — Automated resolver bot

A long-running bot that watches markets approaching their close time,
fetches the real-world outcome from an oracle, and calls
`resolveMarket` on behalf of the configured resolver keypair.

```ts
#!/usr/bin/env bun
// scripts/resolver-bot.ts
import { Connection, Keypair } from "@solana/web3.js";
import { readFileSync } from "node:fs";
import {
  CypherClient,
  keypairToWallet,
  MarketState,
  marketPhase,
  MarketType,
  parseCypherError,
} from "@cypher-zk/sdk";

const RESOLVER_PATH = process.env.RESOLVER_KEYPAIR!;
const RPC_URL = process.env.RPC_URL!;
const CHECK_INTERVAL_MS = 60_000; // every minute

const resolver = Keypair.fromSecretKey(
  Uint8Array.from(JSON.parse(readFileSync(RESOLVER_PATH, "utf8"))),
);
const client = new CypherClient({
  connection: new Connection(RPC_URL, "confirmed"),
  wallet: keypairToWallet(resolver),
  cluster: "devnet",
});

/**
 * Replace with your oracle adapter — Pyth, Switchboard, an API call,
 * a custom feed, whatever. Returns the winning outcome index, or null
 * if the answer isn't available yet (the bot will retry next tick).
 *
 * Note: MarketAccount has no question field. Pass the question string
 * from a separate `client.marketQuestions.fetchMany` call (see tick() below).
 */
async function fetchOutcome(market: {
  marketId: bigint;
  question: string; // fetched separately from MarketQuestion account
  marketType: number;
  numOutcomes: number;
}): Promise<number | null> {
  // Stub: parse the question, hit an API, return.
  // E.g. "Will BTC > $100k by 2025-12-31?" → query Coingecko
  return null;
}

async function tick() {
  const allMarkets = await client.markets.byState(MarketState.Active);
  const resolverMarkets = allMarkets.filter(
    ({ account }) =>
      account.resolver.equals(resolver.publicKey) &&
      marketPhase(account) === "awaitingResolve",
  );
  if (!resolverMarkets.length) return;

  // Batch-fetch question text. 0.8.8+ chunks at Solana's 100-key cap,
  // retries on transient errors, and runs with bounded concurrency.
  const questions = await client.marketQuestions.fetchMany(resolverMarkets);

  for (const { publicKey, account } of resolverMarkets) {
    const question = questions.get(publicKey.toBase58()) ?? "";
    const outcome = await fetchOutcome({ ...account, question });
    if (outcome == null) {
      console.log(`[skip] #${account.marketId} — outcome not yet available`);
      continue;
    }

    if (outcome >= (account.marketType === MarketType.YesNo ? 2 : account.numOutcomes)) {
      console.error(`[skip] #${account.marketId} — outcome ${outcome} out of range`);
      continue;
    }

    try {
      console.log(`[resolve] #${account.marketId} → outcome=${outcome}`);
      const result = await client.actions.resolveMarket({
        payer: resolver.publicKey,
        resolver: resolver.publicKey,
        marketId: account.marketId,
        outcomeValue: outcome,
        onProgress: ({ stage }) => process.stdout.write(`  ${stage}\r`),
      });
      console.log(`\n  ✓ resolved — ${result.signature}`);
    } catch (err) {
      const parsed = parseCypherError(err);
      console.error(
        `  ✗ resolve failed: ${parsed ? `${parsed.name}: ${parsed.msg}` : String(err)}`,
      );
    }
  }
}

setInterval(tick, CHECK_INTERVAL_MS);
tick();
```

## Why a separate resolver keypair

The market's `resolver` is fixed at create time. A creator who wants to
delegate resolution can pin the bot's pubkey as the resolver — the bot
then has exclusive permission to resolve that market.

If a creator wants to retain manual control, pin the creator's own
pubkey as resolver and skip this bot.

## Race-safety

- `resolveMarket` is rejected by the program if `market.state` is
  already `Resolved`. So even if two bot instances run concurrently,
  the second loses cleanly with `AlreadyResolved`.
- The bot uses `marketPhase(account) === "awaitingResolve"` as the
  client-side gate; the contract enforces it again on-chain.

## v0.2+: finalize after the challenge window

After `client.actions.resolveMarket(...)` returns, the market is in
`PendingResolution` — NOT `Resolved`. The position can't be claimed
yet. Either schedule a `finalizeResolution` call ~5 minutes after
`market.challengeDeadline` (anyone can call it), or leave it to
bettors / a separate finalizer bot.

```ts
import { marketPhase } from "@cypher-zk/sdk";

setInterval(async () => {
  const markets = await client.markets.byState(/* PendingResolution */ 4);
  for (const { account } of markets) {
    if (marketPhase(account) !== "awaitingFinalize") continue;
    try {
      await client.actions.finalizeResolution({
        caller: resolver.publicKey,
        marketId: account.marketId,
      });
    } catch {
      // disputed → MarketDisputed (6040); skip — admin must override
    }
  }
}, 5 * 60_000);
```

## Operational notes

- **SOL for fees**: keep the resolver wallet topped up — each resolve
  costs SOL for the queue tx + Arcium computation fees.
- **Backoff**: if `fetchOutcome` returns `null`, the bot just waits for
  the next tick. Don't busy-loop on a slow oracle.
- **Logging**: log every resolve attempt with the signature — easy to
  trace in Solscan if a dispute arises.
- **Resolution deadline**: if the bot misses
  `market.resolutionDeadline`, the market becomes unresolvable and
  bettors can `claimRefund`. The bot should alert on markets within ~1
  hour of their `resolutionDeadline`.
