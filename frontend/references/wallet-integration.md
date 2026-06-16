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

For dashboards or server-rendered pages that only read state:

```ts
import { readonlyWallet } from "@cypher-zk/sdk";

const client = new CypherClient({
  connection,
  wallet: readonlyWallet(somePubkey),
  cluster: "devnet",
});

await client.markets.all(); // ✅ reads work
await client.actions.placeBet(...); // ❌ throws "readonlyWallet cannot sign"
```

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
