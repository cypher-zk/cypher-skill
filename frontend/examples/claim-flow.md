# Recipe — Claim flow (payout + refund dual path)

```tsx
"use client";
import { useState } from "react";
import {
  useClaimPayout,
  useClaimRefund,
  useMarket,
  useCypherClient,
} from "@cypher-zk/sdk/react";
import {
  marketPhase,
  type ActionProgressEvent,
  type EncryptedPositionAccount,
} from "@cypher-zk/sdk";
import { useWallet } from "@solana/wallet-adapter-react";
import { useQuery } from "@tanstack/react-query";

export function ClaimCard({ marketId }: { marketId: bigint }) {
  const wallet = useWallet();
  const client = useCypherClient();
  const { data: market } = useMarket(marketId);

  // Load this user's position via PDA derivation:
  const { data: position } = useQuery({
    queryKey: ["cypher", "position", String(marketId), wallet.publicKey?.toBase58()],
    queryFn: () => {
      if (!market || !wallet.publicKey) return null;
      // Use the market PDA from useMarket's keys — derived already.
      return client.positions.fetch(getMarketPda(market.marketId), wallet.publicKey);
    },
    enabled: !!market && !!wallet.publicKey,
  });

  const [stage, setStage] = useState<ActionProgressEvent | null>(null);
  const payout = useClaimPayout({});
  const refund = useClaimRefund({});

  if (!market || !position) return null;
  if (position.claimed) return <p>Already claimed.</p>;

  const phase = marketPhase(market);
  const variant: "payout" | "refund" | null =
    phase === "claimable" ? "payout" : phase === "refundable" ? "refund" : null;

  if (!variant) return <p>Nothing to claim right now (phase: {phase}).</p>;

  const mutation = variant === "payout" ? payout : refund;
  return (
    <button
      disabled={mutation.isPending}
      onClick={() =>
        mutation.mutate({
          payer: wallet.publicKey!,
          user: wallet.publicKey!,
          marketId,
          onProgress: setStage,
        })
      }
    >
      {mutation.isPending && stage
        ? labelFor(stage)
        : variant === "payout"
          ? "Claim payout"
          : "Claim refund"}
    </button>
  );
}

function labelFor(e: ActionProgressEvent): string {
  switch (e.stage) {
    case "validating":        return "Checking eligibility…";
    case "fetching-state":    return "Loading…";
    case "submitting":        return "Submitting…";
    case "awaiting-callback": return "Awaiting MPC nodes…";
    case "refetching":        return "Confirming…";
    case "done":              return "Done!";
    default:                  return "Working…";
  }
}

function getMarketPda(marketId: bigint) {
  // Tiny helper — usually you'd import `marketPda` from "@cypher-zk/sdk"
  // and call `marketPda(marketId)[0]`.
  const { marketPda } = require("@cypher-zk/sdk");
  return marketPda(marketId)[0];
}
```

## What about losing bets?

For a losing position on a resolved market, **the SDK still lets you
call `claimPayout`** — the on-chain circuit will compute payout = 0 and
mark `position.claimed = true` without transferring anything. The user
pays the MPC computation fee but recovers no stake.

Pattern: gate the button behind a client-side decrypt-and-check so users
on the losing side aren't charged for a guaranteed-zero payout.

```ts
import { decryptBetInput, createCipher, bigIntToLeBytes, fetchMxePublicKey } from "@cypher-zk/sdk";
import { loadSecret } from "./persist-secret";

async function isLikelyWinner(
  client: CypherClient,
  position: EncryptedPositionAccount,
  marketOutcome: number,
): Promise<boolean | null> {
  const secret = loadSecret(position.market);
  if (!secret) return null; // can't tell without the key
  const mxe = await fetchMxePublicKey(client);
  if (!mxe) return null;
  const cipher = createCipher(secret, mxe);
  const { side } = decryptBetInput(
    position,
    cipher,
    bigIntToLeBytes(position.nonce, 16),
  );
  return side === marketOutcome;
}
```

If `isLikelyWinner` returns `false`, hide the Claim Payout button and
show "Your position did not win this market." If it returns `null` (no
saved key), show both buttons and let the user decide.
