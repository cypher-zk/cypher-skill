# Recipe — Market list with phase badges + filters

```tsx
"use client";
import { useState } from "react";
import { useMarkets } from "@cypher-zk/sdk/react";
import {
  MarketState,
  MarketCategory,
  marketPhase,
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
  betting:          { color: "#22c55e", label: "Open" },
  awaitingResolve:  { color: "#fbbf24", label: "Awaiting Resolve" },
  claimable:        { color: "#3b82f6", label: "Claim Open" },
  refundable:       { color: "#f97316", label: "Refundable" },
  expired:          { color: "#6b7280", label: "Expired" },
  cancelled:        { color: "#ef4444", label: "Cancelled" },
};

export function MarketList() {
  const [stateFilter, setStateFilter] = useState<number | undefined>(
    MarketState.Active,
  );
  const { data: markets, isLoading } = useMarkets({ state: stateFilter });

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
          <MarketCard key={publicKey.toBase58()} pda={publicKey} market={account} />
        ))}
      </ul>
    </>
  );
}

function MarketCard({ pda, market }: { pda: PublicKey; market: MarketAccount }) {
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
        <h3>{market.question}</h3>
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

- `useMarkets({ state })` uses a memcmp filter at byte offset `8 + 469`
  (the discriminator plus the offset of `state` in the `Market` struct).
  Filter is server-side via `getProgramAccounts` — efficient even with
  many markets.
- `useMarkets({ creator })` filters by `creator` Pubkey (offset 8 + 212).
- Pass `{ creator, state }` together → SDK picks `creator` (the more
  selective filter). Combine manually if you need both.
- For "my markets across all states", filter client-side after fetching
  `useMarkets({ creator: wallet.publicKey })`.
