# Recipe — Next.js 14 App Router setup

Cypher SDK works in Next.js with one important constraint: **all SDK
calls must happen on the client**. The SDK uses `@solana/web3.js` which
isn't SSR-safe (browser Buffer assumptions). Tag every component that
touches the SDK with `"use client"`.

## Layout

```
app/
├── layout.tsx                  # Root — wraps in Providers
├── providers.tsx               # "use client" — CypherProvider + QueryClient
├── cypher-client.ts            # Module-level CypherClient cache
├── persist-secret.ts           # x25519 secret persistence
├── markets/
│   ├── page.tsx                # Server component — passes data down
│   └── [id]/
│       ├── page.tsx
│       └── bet-card.tsx        # "use client"
└── api/
    └── markets/route.ts        # Optional — server-rendered market list
```

## `app/providers.tsx`

```tsx
"use client";

import { CypherProvider } from "@cypher-zk/sdk/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import {
  ConnectionProvider,
  WalletProvider,
  useWallet,
} from "@solana/wallet-adapter-react";
import { WalletModalProvider } from "@solana/wallet-adapter-react-ui";
import { PhantomWalletAdapter } from "@solana/wallet-adapter-wallets";
import { useMemo, useState } from "react";
import { Connection } from "@solana/web3.js";
import { CypherClient } from "@cypher-zk/sdk";

const RPC = process.env.NEXT_PUBLIC_RPC_URL ?? "https://api.devnet.solana.com";
const wallets = [new PhantomWalletAdapter()];

function CypherShell({ children }: { children: React.ReactNode }) {
  const wallet = useWallet();
  const client = useMemo(() => {
    if (!wallet.publicKey) return null;
    return new CypherClient({
      connection: new Connection(RPC, "confirmed"),
      wallet: wallet as never,
      cluster: "devnet",
    });
  // wallet.signTransaction is a new function ref every render on most adapters
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [wallet.publicKey]);

  if (!client) return <>{children}</>;
  return <CypherProvider client={client}>{children}</CypherProvider>;
}

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: { queries: { staleTime: 10_000 } },
  }));

  return (
    <ConnectionProvider endpoint={RPC}>
      <WalletProvider wallets={wallets} autoConnect>
        <WalletModalProvider>
          <QueryClientProvider client={queryClient}>
            <CypherShell>{children}</CypherShell>
          </QueryClientProvider>
        </WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
}
```

## `app/layout.tsx`

```tsx
import { Providers } from "./providers";
import "@solana/wallet-adapter-react-ui/styles.css";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

## `next.config.js`

```js
// next.config.js
/** @type {import('next').NextConfig} */
module.exports = {
  webpack: (config) => {
    // @solana/web3.js needs Node polyfills in the browser
    config.resolve.fallback = {
      ...config.resolve.fallback,
      crypto: false,
      fs: false,
      stream: false,
    };
    return config;
  },
};
```

If you hit `Buffer is not defined` errors, add the polyfill in
`providers.tsx`:

```tsx
"use client";
import { Buffer } from "buffer";
if (typeof window !== "undefined") (window as never).Buffer ??= Buffer;
```

## Server-side data fetching

You can prefetch read-only data on the server using a "read-only client":

```tsx
// app/markets/page.tsx (server component)
import { Connection, Keypair } from "@solana/web3.js";
import { CypherClient, readonlyWallet } from "@cypher-zk/sdk";

export const dynamic = "force-dynamic";

export default async function MarketsPage() {
  const client = new CypherClient({
    connection: new Connection(process.env.RPC_URL!, "confirmed"),
    wallet: readonlyWallet(Keypair.generate().publicKey),
    cluster: "devnet",
  });
  const markets = await client.markets.byState(0); // Active
  return <MarketGrid initialMarkets={markets} />;
}
```

`readonlyWallet` throws on `signTransaction`/`signAllTransactions`, so
the server never accidentally signs. Pass the prefetched data into a
"use client" component that takes over with `useMarkets()` for live
updates.

## Don't put the SDK in `app/api/*.ts`

API routes run server-side without Buffer/crypto polyfills. If you need
server-side bet placement (you almost never do — bets need user
signatures), do it in a separate Node service, not in a Next.js API
route.
