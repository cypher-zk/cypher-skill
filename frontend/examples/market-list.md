# Recipe — Market list with phase badges + filters

```tsx
"use client";
import { useState, useEffect } from "react";
import { useMarkets, useCypherClient } from "@cypher-zk/sdk/react";
import {
  MarketState,
  MarketCategory,
  marketPhase,
  fetchMarketQuestions,
  parseEmbeddedOptions,
  type MarketPhase,
  type MarketAccount,
} from "@cypher-zk/sdk";
import { PublicKey } from "@solana/web3.js";

const CATEGORY_LABELS: Record<number, string> = {
  [MarketCategory.Crypto]: "Crypto",
  [MarketCategory.Politics]: "Politics",
  [MarketCategory.Sports]: "Sports",
  [MarketCategory.Tech]: "Tech",
  [MarketCategory.Economy]: "Economy",
  [MarketCategory.Culture]: "Culture",
  [MarketCategory.Beyond]: "Beyond",
};

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
        {markets.map(({ publicKey, account }) => (
          <MarketCard
            key={publicKey.toBase58()}
            pda={publicKey}
            market={account}
            question={questions.get(publicKey.toBase58()) ?? ""}
          />
        ))}
      </ul>
    </>
  );
}

function MarketCard({
  pda,
  market,
  question,
}: {
  pda: PublicKey;
  market: MarketAccount;
  question: string;
}) {
  const phase = marketPhase(market);
  const badge = PHASE_BADGE[phase];
  return (
    <li>
      <a href={`/markets/${market.marketId}`}>
        <span
          className="badge"
          style={{ background: badge.color, color: "#fff" }}
        >
          {badge.label}
        </span>
        <span className="category">{CATEGORY_LABELS[market.category]}</span>
        <h3>{question ? parseEmbeddedOptions(question).displayQuestion : <em>Loading…</em>}</h3>
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

- `MarketAccount` does not include `question`. Call `fetchMarketQuestions(client, markets)`
  to batch-fetch all questions in one `getMultipleAccountsInfo` RPC call, keyed by
  market PDA base58. Missing entries default to `""`.
- `useMarkets({ state })` uses a server-side memcmp filter — efficient even with many markets.
- `useMarkets({ creator })` filters by creator pubkey.
- Pass `{ creator, state }` together → SDK picks `creator` (more selective). Combine
  manually if you need both.
- For "my markets across all states", fetch `useMarkets({ creator: wallet.publicKey })`
  then filter client-side.
