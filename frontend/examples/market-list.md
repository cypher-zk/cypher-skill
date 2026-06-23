# Recipe — Market list with phase badges + filters

```tsx
"use client";
import { useState, useEffect } from "react";
import { useMarkets, useCypherClient } from "@cypher-zk/sdk/react";
import {
  MarketState,
  marketPhase,
  marketCategoryName,
  fetchMarketQuestions,
  parseEmbeddedOptions,
  type MarketPhase,
  type MarketAccount,
} from "@cypher-zk/sdk";
import { PublicKey } from "@solana/web3.js";

const PHASE_BADGE: Record<MarketPhase, { color: string; label: string }> = {
  betting:             { color: "#22c55e", label: "Open" },
  awaitingResolve:     { color: "#fbbf24", label: "Awaiting Resolve" },
  pendingResolution:   { color: "#a78bfa", label: "Pending" },
  awaitingFinalize:    { color: "#60a5fa", label: "Finalizable" },
  disputed:            { color: "#f87171", label: "Disputed" },
  claimable:           { color: "#3b82f6", label: "Claim Open" },
  refundable:          { color: "#f97316", label: "Refundable" },
  expired:             { color: "#6b7280", label: "Expired" },
  cancelled:           { color: "#ef4444", label: "Cancelled" },
};

export function MarketList() {
  const client = useCypherClient();
  const [stateFilter, setStateFilter] = useState<number | undefined>(
    MarketState.Active,
  );
  const { data: markets, isLoading } = useMarkets({ state: stateFilter });
  const [questions, setQuestions] = useState<Map<string, string>>(new Map());

  useEffect(() => {
    if (!markets?.length) return;
    fetchMarketQuestions(client, markets).then(setQuestions);
  }, [client, markets]);

  if (isLoading) return <p>Loading markets…</p>;
  if (!markets) return null;

  return (
    <>
      <select
        value={stateFilter ?? ""}
        onChange={(e) =>
          setStateFilter(e.target.value === "" ? undefined : Number(e.target.value))
        }
      >
        <option value="">All states</option>
        <option value={MarketState.Active}>Active</option>
        <option value={MarketState.Resolved}>Resolved</option>
        <option value={MarketState.Closed}>Cancelled</option>
      </select>

      <ul>
        {markets.map(({ publicKey, account }) => {
          // Full fallback chain: v3 PDA question → v1/v2 inlineQuestion → placeholder
          const rawQuestion =
            questions.get(publicKey.toBase58()) ||
            account.inlineQuestion ||
            `Market #${account.marketId}`;
          return (
            <MarketCard
              key={publicKey.toBase58()}
              pda={publicKey}
              market={account}
              rawQuestion={rawQuestion}
            />
          );
        })}
      </ul>
    </>
  );
}

function MarketCard({
  pda,
  market,
  rawQuestion,
}: {
  pda: PublicKey;
  market: MarketAccount;
  rawQuestion: string;
}) {
  const phase = marketPhase(market);
  const badge = PHASE_BADGE[phase];
  const { displayQuestion } = parseEmbeddedOptions(rawQuestion);
  return (
    <li>
      <a href={`/markets/${market.marketId}`}>
        <span
          className="badge"
          style={{ background: badge.color, color: "#fff" }}
        >
          {badge.label}
        </span>
        <span className="category">{marketCategoryName(market.category)}</span>
        <h3>{displayQuestion}</h3>
        <small>
          {phase === "betting"
            ? `Closes ${new Date(Number(market.closeTime) * 1000).toLocaleString()}`
            : `Closed ${new Date(Number(market.closeTime) * 1000).toLocaleString()}`}
        </small>
      </a>
    </li>
  );
}
```

## Notes

- **Question text**: current (v3) markets store questions in a separate `MarketQuestion` PDA.
  Call `fetchMarketQuestions(client, markets)` to batch-fetch them in one RPC call (keyed by PDA base58).
  Legacy v1/v2 markets have no `MarketQuestion` PDA — their question is in `account.inlineQuestion`.
  Always use the full fallback chain:
  ```ts
  const rawQ = questions.get(pda) || account.inlineQuestion || `Market #${id}`;
  const { displayQuestion } = parseEmbeddedOptions(rawQ);
  ```
- **Option labels**: use `getMarketOptionLabels(account, rawQ)` to get the betting buttons for any market type.
  Pass the raw question (with `[…]` suffix), not the stripped `displayQuestion`.
- `useMarkets({ state })` uses a server-side memcmp filter — efficient even with many markets.
- `useMarkets({ creator })` filters by creator pubkey.
- Pass `{ creator, state }` together → SDK picks `creator` (more selective). Combine
  manually if you need both.
- For "my markets across all states", fetch `useMarkets({ creator: wallet.publicKey })`
  then filter client-side.
