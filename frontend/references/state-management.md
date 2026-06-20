# Frontend reference — State management with TanStack Query

The SDK's React hooks are built on TanStack Query v5. Default cache
times are tuned for Cypher's update cadence:

| Hook | `staleTime` | Why |
| --- | --- | --- |
| `useGlobalState()` | 30s | Protocol config changes rarely (admin-only) |
| `useMarket(id)` | 10s | Market state changes on bet/resolve/claim |
| `useMarkets()` | 10s | Same |
| `useUserPositions()` | 5s | Positions change on every bet |

Override per call:

```ts
const { data } = useGlobalState({ staleTime: Infinity }); // I'll invalidate manually
const { data } = useMarket(id, { refetchInterval: 1_000 }); // poll every second
```

## Manual invalidation

Mutation hooks auto-invalidate. If you go through the raw instruction
builders (`client.bets.placeYesnoIx`, etc.) you have to invalidate
manually:

```ts
import { marketKeys, positionKeys } from "@cypher-zk/sdk/react";

const qc = useQueryClient();

await sendIx(client, await client.bets.placeYesnoIx({...}));
await qc.invalidateQueries({ queryKey: marketKeys.one(marketId) });
await qc.invalidateQueries({ queryKey: positionKeys.byUser(user) });
```

## Optimistic updates

For perceived snappiness, set the bet result optimistically:

```ts
const placeBet = usePlaceBet({
  onMutate: async (variables) => {
    await qc.cancelQueries({ queryKey: marketKeys.one(variables.marketId) });
    const previous = qc.getQueryData(marketKeys.one(variables.marketId));
    qc.setQueryData(marketKeys.one(variables.marketId), (m: MarketAccount | undefined) => {
      if (!m) return m;
      return { ...m, totalBetsCount: m.totalBetsCount + 1n };
    });
    return { previous };
  },
  onError: (_err, variables, ctx) => {
    if (ctx) qc.setQueryData(marketKeys.one(variables.marketId), ctx.previous);
  },
});
```

Don't optimistically update `revealedPool*` — those are encrypted-bet
totals that come from the MPC callback. Optimistic values for them will
disagree with the real callback result.

## Prefetching

For an SSR/SSG flow, prefetch on the server and hydrate on the client:

```tsx
// app/markets/page.tsx (server)
import { QueryClient, dehydrate, HydrationBoundary } from "@tanstack/react-query";
import { CypherClient, marketKeys } from "@cypher-zk/sdk";

const qc = new QueryClient();
await qc.prefetchQuery({
  queryKey: marketKeys.all,
  queryFn: () => serverClient.markets.all(),
});

return (
  <HydrationBoundary state={dehydrate(qc)}>
    <MarketList />
  </HydrationBoundary>
);
```

## Suspense mode

The SDK hooks don't expose a suspense variant. Use `useSuspenseQuery` from
`@tanstack/react-query` directly with the SDK's query-key factories:

```ts
import { useSuspenseQuery } from "@tanstack/react-query";
import { globalStateKeys, useCypherClient } from "@cypher-zk/sdk/react";

function MyComponent() {
  const client = useCypherClient();
  const { data } = useSuspenseQuery({
    queryKey: globalStateKeys.all,
    queryFn: () => client.globalState.fetch(),
  });
  // `data` is non-nullable inside a <Suspense fallback={...}>
}
```
