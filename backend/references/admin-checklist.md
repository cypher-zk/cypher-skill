# Integrator pre-launch checklist

Checklist for shipping a Cypher-powered app to production.

## Before going live

- [ ] RPC endpoint is a private/paid node (Helius, QuickNode, Triton) — public RPCs throttle `getProgramAccounts`
- [ ] `acceptedMint` is read from `client.globalState.fetch().acceptedMint`, never hard-coded
- [ ] Every `placeBet` callsite persists `userKeypair.privateKey` keyed by `position.market`
- [ ] All market-action buttons are gated on `marketPhase(market)` — not just on `market.state`
- [ ] Question text is fetched via `fetchMarketQuestions(client, markets)`, never via `market.question`
- [ ] `PendingResolution` (state=4) is surfaced in the UI — don't show Claim button until state is `Resolved`
- [ ] `claimPayout` and `claimRefund` deadlines are shown to users so positions don't expire unclaimed

## Wallet integration

- [ ] `CypherClient` constructed once (module-level or `useMemo`) — not per render
- [ ] `CypherProvider` is mounted inside `QueryClientProvider`, not above it
- [ ] `readonlyWallet` used for unauthenticated views (market list, etc.) — no wallet required to browse
- [ ] Wallet disconnect clears any cached client instance

## Error handling

- [ ] `parseCypherError(err)` used at top-level catch to surface readable error names to users
- [ ] `placeBet` timeout bumped for slow networks: `{ timeoutMs: 300_000 }`
- [ ] `claimPayout` / `claimRefund` catch `AlreadyClaimed` (race on double-click)

## Node.js scripts

- [ ] Write scripts (create market, resolve) gate on an env var (`KEYPAIR_PATH`, `DRY_RUN`, etc.)
- [ ] Resolver bot handles `AlreadyResolved` gracefully (two instances can race)
- [ ] Resolver bot calls `finalizeResolution` after `market.challengeDeadline` elapses

## devnet smoke before mainnet

```bash
DEVNET=1 DEVNET_RPC=<url> bun test tests/devnet
```
