# Recipe — Dispute window UI (v0.2+)

After a market's reveal callback fires, it sits in `PendingResolution`
(state=4) for `challenge_period` seconds (24h–48h, set at create time).
During this window:

- **Anyone** can `flagResolution` to dispute the outcome.
- **Anyone** can `finalizeResolution` once the window elapses
  undisputed.
- **Only the admin** can `adminOverrideResolution` on a disputed
  market.

The market's `marketPhase` resolves to one of: `pendingResolution`,
`awaitingFinalize`, or `disputed`.

```tsx
"use client";
import { useEffect, useState } from "react";
import {
  useMarket,
  useFlagResolution,
  useFinalizeResolution,
  useAdminOverrideResolution,
} from "@cypher-zk/sdk/react";
import {
  marketPhase,
  MarketType,
  type ActionProgressEvent,
} from "@cypher-zk/sdk";
import { useWallet } from "@solana/wallet-adapter-react";

export function DisputeWindowCard({
  marketId,
  isAdmin,
}: {
  marketId: bigint;
  isAdmin: boolean;
}) {
  const wallet = useWallet();
  const { data: market } = useMarket(marketId, { refetchInterval: 5_000 });
  const [now, setNow] = useState(() => BigInt(Math.floor(Date.now() / 1000)));
  const [stage, setStage] = useState<ActionProgressEvent | null>(null);

  // Tick once a second so the countdown stays live.
  useEffect(() => {
    const id = setInterval(
      () => setNow(BigInt(Math.floor(Date.now() / 1000))),
      1_000,
    );
    return () => clearInterval(id);
  }, []);

  const flag = useFlagResolution({});
  const finalize = useFinalizeResolution({});
  const override = useAdminOverrideResolution({});

  if (!market) return null;
  const phase = marketPhase(market, now);
  if (phase !== "pendingResolution" && phase !== "awaitingFinalize" && phase !== "disputed") {
    return null; // not in the challenge window
  }

  const secondsLeft = market.challengeDeadline > now ? market.challengeDeadline - now : 0n;

  return (
    <div className="card">
      <h2>Challenge Window</h2>
      <p>
        Outcome <strong>{market.outcome}</strong> — payout ratio{" "}
        <code>{market.payoutRatio.toString()}</code>
      </p>
      <p>{describePhase(phase, secondsLeft)}</p>

      {phase === "pendingResolution" && wallet.publicKey && (
        <button
          disabled={flag.isPending}
          onClick={() =>
            flag.mutate({
              flagger: wallet.publicKey!,
              marketId,
              onProgress: setStage,
            })
          }
        >
          {flag.isPending && stage ? labelFor(stage) : "Flag this resolution"}
        </button>
      )}

      {phase === "awaitingFinalize" && wallet.publicKey && (
        <button
          disabled={finalize.isPending}
          onClick={() =>
            finalize.mutate({
              caller: wallet.publicKey!,
              marketId,
              onProgress: setStage,
            })
          }
        >
          {finalize.isPending && stage ? labelFor(stage) : "Finalize resolution"}
        </button>
      )}

      {phase === "disputed" && isAdmin && (
        <AdminOverridePanel
          marketId={marketId}
          marketType={market.marketType}
          numOutcomes={market.numOutcomes}
          currentOutcome={market.outcome}
          onSubmit={(outcomeValue) =>
            override.mutate({
              admin: wallet.publicKey!,
              marketId,
              outcomeValue,
              onProgress: setStage,
            })
          }
          isPending={override.isPending}
          stage={stage}
        />
      )}
      {phase === "disputed" && !isAdmin && (
        <p className="hint">Awaiting admin review.</p>
      )}

      {(flag.error ?? finalize.error ?? override.error) && (
        <p className="error">
          {(flag.error ?? finalize.error ?? override.error)!.message}
        </p>
      )}
    </div>
  );
}

function describePhase(
  phase: "pendingResolution" | "awaitingFinalize" | "disputed",
  secondsLeft: bigint,
): string {
  if (phase === "pendingResolution") {
    return `Challenge window closes in ${formatDuration(secondsLeft)}. Anyone can flag this resolution as wrong.`;
  }
  if (phase === "awaitingFinalize") {
    return "Challenge window elapsed undisputed. Anyone can finalize.";
  }
  return "This resolution has been disputed. Awaiting admin override.";
}

function formatDuration(seconds: bigint): string {
  const s = Number(seconds);
  const h = Math.floor(s / 3600);
  const m = Math.floor((s % 3600) / 60);
  return `${h}h ${m}m`;
}

function labelFor(e: ActionProgressEvent): string {
  switch (e.stage) {
    case "validating": return "Checking…";
    case "submitting": return "Submitting…";
    case "refetching": return "Refreshing…";
    case "done":       return "Done!";
    default:           return "Working…";
  }
}

function AdminOverridePanel({
  marketId,
  marketType,
  numOutcomes,
  currentOutcome,
  onSubmit,
  isPending,
  stage,
}: {
  marketId: bigint;
  marketType: number;
  numOutcomes: number;
  currentOutcome: number;
  onSubmit: (outcomeValue: number) => void;
  isPending: boolean;
  stage: ActionProgressEvent | null;
}) {
  const [outcome, setOutcome] = useState(currentOutcome);
  const max = marketType === MarketType.YesNo ? 1 : numOutcomes - 1;

  return (
    <div>
      <label>
        Corrected outcome:{" "}
        <input
          type="number"
          min={0}
          max={max}
          value={outcome}
          onChange={(e) => setOutcome(Number(e.target.value))}
          disabled={isPending}
        />
      </label>
      <button disabled={isPending} onClick={() => onSubmit(outcome)}>
        {isPending && stage ? labelFor(stage) : "Submit admin override"}
      </button>
    </div>
  );
}
```

## What the user sees

1. **`pendingResolution`** — countdown timer ("Challenge window closes
   in 23h 47m"), Flag button enabled, no Finalize button.
2. **`awaitingFinalize`** — "Window elapsed undisputed", Finalize
   button enabled. Anyone clicks to finish.
3. **`disputed`** — non-admin users see "Awaiting admin review".
   Admin sees an outcome-picker + Override button.

## On-chain consequences

- **Flag** → `market.disputed = true`. Blocks `finalizeResolution`
  until admin overrides. Does NOT cancel the original outcome.
- **Finalize** → `market.state = Resolved`. `claim_deadline` and
  `refund_deadline` get set. `claimPayout` opens.
- **Admin override** → recomputes `payout_ratio` from the on-chain
  plaintext pools, flips state to `Resolved`. Emits
  `ResolutionOverriddenEvent` with old + new outcome.

## Subscribing to challenge-window events for live UI

```ts
useEffect(() => {
  const subs = [
    client.events.onResolutionFlagged((data) => {
      if (data.market.equals(marketPda)) refetchMarket();
    }),
    client.events.onMarketFinalized((data) => {
      if (data.market.equals(marketPda)) refetchMarket();
    }),
    client.events.onResolutionOverridden((data) => {
      if (data.market.equals(marketPda)) refetchMarket();
    }),
  ];
  return () => subs.forEach((s) => s.unsubscribe());
}, [client, marketPda]);
```

## Anti-hallucination

- **NEVER** show "Claim payout" while phase is `pendingResolution`,
  `awaitingFinalize`, or `disputed`. The SDK's `claimPayoutAction`
  rejects pre-flight with a hint, but UI should mirror.
- **NEVER** show `Flag` after the window closed — the contract throws
  `ChallengePeriodElapsed` (6039).
- **NEVER** show `Finalize` before the window elapses — throws
  `ChallengePeriodNotElapsed` (6038).
- **NEVER** show `Finalize` if `market.disputed === true` — throws
  `MarketDisputed` (6040). Admin must override.
