# Recipe — Place a private bet (end-to-end React)

Complete drop-in component for a Next.js or Vite app. Shows:
- Wallet-adapter integration
- Phase gating
- Live per-stage progress UI
- User-secret persistence
- Error mapping via `parseCypherError`

```tsx
"use client";
import { useState } from "react";
import { usePlaceBet, useMarket, useCypherClient } from "@cypher-zk/sdk/react";
import {
  marketPhase,
  parseCypherError,
  type ActionProgressEvent,
} from "@cypher-zk/sdk";
import { useWallet } from "@solana/wallet-adapter-react";
import { saveSecret } from "./persist-secret";

interface PlaceBetCardProps {
  marketId: bigint;
  question?: string; // from fetchMarketQuestions — MarketAccount has no question field
}

export function PlaceBetCard({ marketId, question }: PlaceBetCardProps) {
  const wallet = useWallet();
  const client = useCypherClient();
  const { data: market } = useMarket(marketId);
  const [side, setSide] = useState<0 | 1>(1);
  const [amount, setAmount] = useState("5");
  const [progress, setProgress] = useState<ActionProgressEvent | null>(null);

  const placeBet = usePlaceBet({
    onSuccess: ({ position, userKeypair, signature }) => {
      if (position) saveSecret(position.market, userKeypair.privateKey);
      // Optimistically clear the form
      setAmount("5");
      console.log("Bet placed:", signature);
    },
  });

  if (!market) return <p>Loading market…</p>;

  const phase = marketPhase(market);
  if (phase !== "betting") {
    return <p>Betting closed. Market is now <strong>{phase}</strong>.</p>;
  }

  const amountMicroUsdc = BigInt(Math.round(parseFloat(amount || "0") * 1_000_000));
  const validAmount = amountMicroUsdc >= market.minBet;
  const clusterParam = client.cluster.name === "mainnet" ? "" : `?cluster=${client.cluster.name}`;

  return (
    <div className="card">
      {question && <h2>{question}</h2>}

      <fieldset>
        <label>
          <input
            type="radio"
            checked={side === 1}
            onChange={() => setSide(1)}
            disabled={placeBet.isPending}
          />
          YES
        </label>
        <label>
          <input
            type="radio"
            checked={side === 0}
            onChange={() => setSide(0)}
            disabled={placeBet.isPending}
          />
          NO
        </label>
      </fieldset>

      <label>
        Amount (USDC)
        <input
          type="number"
          step="0.01"
          min={Number(market.minBet) / 1_000_000}
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          disabled={placeBet.isPending}
        />
      </label>

      {!validAmount && (
        <p className="hint">
          Minimum bet is ${Number(market.minBet) / 1_000_000}.
        </p>
      )}

      <button
        disabled={!wallet.publicKey || !validAmount || placeBet.isPending}
        onClick={() =>
          placeBet.mutate({
            payer: wallet.publicKey!,
            user: wallet.publicKey!,
            marketId,
            side,
            amountUsdc: amountMicroUsdc,
            onProgress: setProgress,
          })
        }
      >
        {placeBet.isPending && progress
          ? labelFor(progress)
          : `Bet $${amount} on ${side === 1 ? "YES" : "NO"}`}
      </button>

      {placeBet.error && (
        <ErrorBanner error={placeBet.error} />
      )}

      {placeBet.isSuccess && (
        <p className="success">
          Bet placed!{" "}
          <a
            href={`https://solscan.io/tx/${placeBet.data.signature}${clusterParam}`}
            target="_blank"
            rel="noreferrer"
          >
            View on Solscan ↗
          </a>
        </p>
      )}
    </div>
  );
}

function labelFor(e: ActionProgressEvent): string {
  switch (e.stage) {
    case "validating":        return "Validating…";
    case "fetching-state":    return "Loading state…";
    case "encrypting":        return "Encrypting your bet…";
    case "submitting":        return "Submitting…";
    case "awaiting-callback": return "Awaiting MPC nodes (~10s)…";
    case "refetching":        return "Updating position…";
    case "done":              return "Done!";
  }
}

function ErrorBanner({ error }: { error: Error }) {
  const parsed = parseCypherError(error);
  return (
    <p className="error">
      {parsed ? `${parsed.name}: ${parsed.msg}` : error.message}
    </p>
  );
}
```

## What the user sees

The button label streams through the stages:

```
[ Bet $5 on YES ]        ◄ initial
        ↓ click
[ Validating… ]          ◄ ~10ms
[ Loading state… ]       ◄ ~200ms (cached on subsequent bets)
[ Encrypting your bet… ] ◄ ~50ms
[ Submitting… ]          ◄ ~1-2s (slot confirmation)
[ Awaiting MPC nodes (~10s)… ]
[ Updating position… ]
[ Done! ]
[ Bet placed! View on Solscan ↗ ]
```

## What can fail

| Error | Cause | UX |
| --- | --- | --- |
| `MarketNotActive` | Market closed between render and click | Refetch market, hide bet form |
| `BetTooSmall` | Amount under `market.minBet` | Validated client-side here; should not reach chain |
| `WrongMint` | User's ATA is the wrong mint | Show "Please use USDC/CSDC" |
| Computation timeout | Arcium cluster slow | Surface "Network busy, try again" |
| User rejected signature | Wallet rejection | Silent — `error.message` includes "User rejected" |

`parseCypherError` extracts the `CypherErrorCode` from Anchor's error
wrapper so you can branch on `parsed?.name` instead of substring
matching.
