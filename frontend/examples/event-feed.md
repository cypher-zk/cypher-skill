# Recipe — Live event feed with auto-reconnect

```tsx
"use client";
import { useEffect, useState } from "react";
import { useCypherClient } from "@cypher-zk/sdk/react";
import type { CypherEvent } from "@cypher-zk/sdk";

const MAX_ITEMS = 50;

export function EventFeed() {
  const client = useCypherClient();
  const [events, setEvents] = useState<CypherEvent[]>([]);

  useEffect(() => {
    const sub = client.events.subscribeAll((event) => {
      setEvents((prev) => [event, ...prev].slice(0, MAX_ITEMS));
    });
    return () => sub.unsubscribe();
  }, [client]);

  return (
    <ul>
      {events.map((e, i) => (
        <li key={i}>
          <strong>{e.name}</strong> {summarize(e)}
        </li>
      ))}
    </ul>
  );
}

function summarize(e: CypherEvent): string {
  switch (e.name) {
    case "MarketCreatedEvent":
      return `#${e.data.marketId} — ${e.data.question}`;
    case "BetPlacedEvent":
      return `market ${short(e.data.market.toBase58())} (odds ${e.data.entryOdds})`;
    case "MarketResolvedEvent":
      return `${short(e.data.market.toBase58())} → outcome ${e.data.outcome}`;
    case "PayoutClaimedEvent":
      return `${e.data.payoutAmount} → ${short(e.data.user.toBase58())}`;
    case "RefundClaimedEvent":
      return `${e.data.refundAmount} → ${short(e.data.user.toBase58())}`;
    case "MarketCancelledEvent":
      return `${short(e.data.market.toBase58())} cancelled (bond ${e.data.bondReturned})`;
    case "CreatorWithdrawnEvent":
      return `${short(e.data.creator.toBase58())} withdrew ${e.data.total}`;
    case "ResolutionFlaggedEvent":
      return `${short(e.data.market.toBase58())} flagged by ${short(e.data.flaggedBy.toBase58())}`;
    case "MarketFinalizedEvent":
      return `${short(e.data.market.toBase58())} finalized → outcome ${e.data.outcome}`;
    case "ResolutionOverriddenEvent":
      return `${short(e.data.market.toBase58())} overridden → outcome ${e.data.newOutcome}`;
  }
}

function short(s: string): string {
  return `${s.slice(0, 4)}…${s.slice(-4)}`;
}
```

## When the WebSocket drops

`Connection.onLogs` reconnects automatically on most RPC providers. For
extra resilience, key the subscription off `client` identity so a new
provider URL forces a re-subscribe:

```ts
useEffect(() => {
  const sub = client.events.subscribeAll(/* ... */);
  return () => sub.unsubscribe();
}, [client.cluster.rpc]); // re-subscribe on RPC change
```

For environments without WebSocket support (some serverless edges), use
the poll fallback:

```ts
import { useQuery } from "@tanstack/react-query";

function usePolledEvents() {
  const client = useCypherClient();
  return useQuery({
    queryKey: ["cypher", "events", "polled"],
    queryFn: () => client.events.pollEvents({ limit: 50 }),
    refetchInterval: 5_000, // every 5s
  });
}
```

## Filtering for a specific market

`subscribeAll` fires for every event on the program; filter client-side:

```ts
useEffect(() => {
  const sub = client.events.subscribeAll((evt) => {
    if (
      (evt.name === "BetPlacedEvent" ||
       evt.name === "MarketResolvedEvent" ||
       evt.name === "PayoutClaimedEvent" ||
       evt.name === "RefundClaimedEvent") &&
      evt.data.market.equals(myMarketPda)
    ) {
      handle(evt);
    }
  });
  return () => sub.unsubscribe();
}, [client, myMarketPda]);
```
