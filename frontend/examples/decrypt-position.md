# Recipe — Decrypt and render a user's own bet

The on-chain `EncryptedPosition` account hides `amount` and `side`. The
**user themselves** can decrypt it client-side using the x25519 private
key they saved when placing the bet.

```tsx
"use client";
import { useQuery } from "@tanstack/react-query";
import {
  createCipher,
  decryptBetInput,
  bigIntToLeBytes,
  fetchMxePublicKey,
  type EncryptedPositionAccount,
} from "@cypher-zk/sdk";
import { useCypherClient } from "@cypher-zk/sdk/react";
import { loadSecret } from "./persist-secret";

export function MyPositionRow({ position }: { position: EncryptedPositionAccount }) {
  const client = useCypherClient();
  const { data: decrypted } = useQuery({
    queryKey: [
      "cypher", "decrypt",
      position.market.toBase58(),
      position.betIndex.toString(),
    ],
    queryFn: async () => {
      const secret = loadSecret(position.market, position.betIndex);
      if (!secret) return null;
      const mxe = await fetchMxePublicKey(client);
      if (!mxe) return null;
      const cipher = createCipher(secret, mxe);
      const nonceBytes = bigIntToLeBytes(position.nonce, 16);
      return decryptBetInput(position, cipher, nonceBytes);
    },
    staleTime: Infinity,
  });

  return (
    <div>
      <div>Market: {position.market.toBase58().slice(0, 6)}…</div>
      {decrypted ? (
        <div>
          You bet <strong>${Number(decrypted.amount) / 1_000_000}</strong> on{" "}
          <strong>{decrypted.side === 1 ? "YES" : "NO"}</strong>
        </div>
      ) : (
        <div className="muted">🔒 Saved key not available — cannot decrypt</div>
      )}
      <div>Entry odds: {position.entryOdds.toString()}</div>
      <div>Net amount (public, on-chain): {position.netAmount.toString()}</div>
      <div>Claimed: {position.claimed ? "Yes" : "No"}</div>
    </div>
  );
}
```

## What's actually on-chain vs encrypted

| Field | Visibility |
| --- | --- |
| `user` | Public (everyone sees you placed a bet on this market) |
| `market` | Public |
| `encryptedAmount` (32 bytes) | Ciphertext — only the user can decrypt |
| `encryptedSide` (32 bytes) | Ciphertext — only the user can decrypt |
| `userPubkey` (32 bytes) | Public x25519 pubkey (the one the user generated) |
| `nonce` (u128) | Public (needed to decrypt; reused only by the user) |
| `entryOdds` (u64) | Public — locked-in odds at bet time |
| `netAmount` (u64) | **Public** — the after-fees stake. Anyone can see *how much* you bet, just not *which side* |
| `claimed` (bool) | Public |

The privacy guarantee Cypher offers: **your side stays secret**. Bet
amounts are visible on-chain (you can verify them by reading the
`Market.revealedPoolX` totals against your `position.netAmount`).

## If the user loses their saved key

There is **no on-chain recovery path**. The user can still:
- See that they have a position on the market (public `EncryptedPosition` account).
- Call `claimPayout` blindly — the on-chain circuit will compute payout from
  whatever side they originally chose, regardless of whether the user
  remembers it.
- Call `claimRefund` if the market never resolved (refunds always succeed
  for non-resolved markets — they refund the full `netAmount`).

What they **cannot** do without the key:
- Display "you bet X on Y" in the UI.
- Pre-check whether they're a winner before paying for the MPC payout
  computation.
