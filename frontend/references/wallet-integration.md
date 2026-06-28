# Frontend reference — Wallet integration

`@cypher-zk/sdk` accepts any object satisfying the `Wallet` interface:

```ts
interface Wallet {
  publicKey: PublicKey;
  signTransaction<T extends Transaction | VersionedTransaction>(tx: T): Promise<T>;
  signAllTransactions<T extends Transaction | VersionedTransaction>(txs: T[]): Promise<T[]>;
  payer?: Keypair; // Node-only; never set in browser wallets
}
```

Every `@solana/wallet-adapter` wallet (Phantom, Solflare, Backpack,
Glow, Coinbase, Ledger via adapter, …) satisfies this interface. Pass
the `useWallet()` hook's result directly:

```tsx
const wallet = useWallet();
const client = new CypherClient({ connection, wallet: wallet as never, cluster: "devnet" });
```

The `as never` cast is safe — the adapter shape matches the SDK
interface but TS doesn't auto-narrow the optional `publicKey` field on
wallet-adapter's union type.

## Read-only mode (no signing)

For dashboards or server-rendered pages that only read state, and as a
fallback while the user's wallet adapter is hydrating:

```ts
import { readonlyWallet } from "@cypher-zk/sdk";

const client = new CypherClient({
  connection,
  wallet: readonlyWallet(somePubkey),
  cluster: "devnet",
});

await client.markets.all();           // ✅ reads work
await client.actions.placeBet(...);   // ❌ throws ReadonlyWalletError
```

### Detecting the read-only case in mutations (0.8.9+)

A `readonlyWallet` sign attempt throws a typed `ReadonlyWalletError`
(`code === "READONLY_WALLET"`, `name === "ReadonlyWalletError"`) — much
more useful than the old generic `Error`. Catch it in mutation `onError`
to show a "connect wallet" prompt instead of leaking the SDK detail:

```ts
import { ReadonlyWalletError } from "@cypher-zk/sdk";
import { usePlaceBet } from "@cypher-zk/sdk/react";

const placeBet = usePlaceBet({
  onError: (err) => {
    if (err instanceof ReadonlyWalletError) {
      toast.error("Connect your wallet to place a bet.");
      return;
    }
    toast.error(err.message);
  },
});
```

For programmatic detection across SDK-bundle boundaries (RSC, multiple
SDK copies), match on the stable `code`:

```ts
if ((err as { code?: string })?.code === "READONLY_WALLET") { /* … */ }
```

This error fires whenever a write hook executes against a `CypherClient`
whose wallet is still on the readonly fallback. The standard fix is to
gate the calling button on your adapter's "ready" state (e.g. with
Privy: `usePrivy().ready && authenticated && useWallets().wallets[0]`),
not to suppress the error.

On older SDKs (`<0.8.9`) you'll see a plain
`Error("readonlyWallet cannot sign transactions")`. Match on the
substring as a fallback when you can't pin the SDK version.

## Multi-wallet scenarios

If your app has separate "payer" and "user" identities (e.g. a relayer
pays gas, the user owns the position), every action helper takes both:

```ts
placeBet({
  payer: relayer.publicKey, // signs tx, pays SOL fees
  user: wallet.publicKey,   // signs the bet ix as the position owner
  ...
});
```

Both must sign — pass them as `signers` to the underlying `sendIx`:

```ts
import { sendIx, placePrivateBetYesnoIx } from "@cypher-zk/sdk";
// Use the raw builder instead of the action when you need extra signers:
const ix = await client.bets.placeYesnoIx({...});
await sendIx(client, ix, { signers: [userKeypair] }); // relayer signs as payer via provider
```

## Mobile wallets

For mobile wallets (Phantom Mobile, Solflare Mobile, etc.) that use
deep-linking or the Mobile Wallet Adapter, just wrap them with the
[`@solana-mobile/wallet-adapter-mobile`](https://github.com/solana-mobile/mobile-wallet-adapter)
adapter — they implement the same `Wallet` interface.

WebSocket subscriptions can be flaky on mobile; prefer
`client.events.pollEvents` for mobile.
