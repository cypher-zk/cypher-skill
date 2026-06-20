# Recipe — Claim flow (payout + refund dual path)

```tsx
"use client";
import { useState } from "react";
import {
  useClaimPayout,
  useClaimRefund,
  useMarket,
  useUserPositions,
} from "@cypher-zk/sdk/react";
import {
  marketPhase,
  marketPda,
  type ActionProgressEvent,
  type EncryptedPositionAccount,
} from "@cypher-zk/sdk";
import { useWallet } from "@solana/wallet-adapter-react";

export function ClaimCard({ marketId }: { marketId: bigint }) {
  const wallet = useWallet();
  const { data: market } = useMarket(marketId);

  // useUserPositions uses positionKeys.byUser internally — auto-invalidated after claim.
  const { data: allPositions } = useUserPositions(wallet.publicKey ?? undefined);
  const targetPda = marketPda(marketId)[0];
  const position = allPositions
    ?.find(({ account }) => account.market.equals(targetPda))
    ?.account ?? null;

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
