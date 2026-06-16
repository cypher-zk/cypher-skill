# Backend reference — Admin operations checklist

## One-time bootstrap (per cluster)

- [ ] Deploy `cypher-main` program to target cluster
- [ ] Create accepted SPL mint (CSDC on devnet, USDC on mainnet, test mint on localnet)
- [ ] Fund admin keypair with ≥ 2 SOL
- [ ] Run `scripts/bootstrap.ts` (see [examples/bootstrap.md](../examples/bootstrap.md))
- [ ] Verify all 8 comp def accounts exist on chain
- [ ] Verify MXE has `cluster` field populated
- [ ] Smoke: `createMarket` → assert `market.state === Active`

## After every contract rebuild

- [ ] `bun run sync:idl` in `cypher-sdk/`
- [ ] `bun test tests/unit/{layout,idl,errors}.test.ts` — all green
- [ ] If any drift test fails, update the corresponding SDK file
- [ ] `bun run typecheck`
- [ ] `bun test`
- [ ] Bump SDK version per semver (patch for non-breaking sync, minor
      for new exports, major for renames/removals)
- [ ] `npm publish` (which runs `prepublishOnly`)

## Per-release smoke

- [ ] Read-only devnet smoke: `DEVNET=1 bun test tests/devnet`
- [ ] Write devnet smoke (funded keypair):
      `DEVNET=1 DEVNET_KEYPAIR=… DEVNET_FULL_FLOW=1 bun test tests/devnet`
- [ ] React example builds + runs: `cd examples/react-vite && bun run dev`

## Updating fee rates

```ts
// Admin only — must be the address pinned in GlobalState.admin
// There's no dedicated instruction; fee rates are set at initialize
// and CAN ONLY BE CHANGED by re-initializing (which resets everything).
// If you need mutable rates, propose a contract change to add an
// `update_fee_rates` instruction.
```

## Rotating the accepted mint

The contract supports this via `update_accepted_mint`:

```ts
const ix = await client.admin.updateAcceptedMintIx({
  admin: admin.publicKey,
  newMint: NEW_MINT_PUBKEY,
  newTreasury: NEW_TREASURY_PUBKEY,
});
await sendAndConfirmTransaction(connection, new Transaction().add(ix), [admin]);
```

**Existing markets keep their old mint.** Only newly created markets use
the updated mint. Existing positions can still be claimed; vaults remain
on the old mint.

## Sweeping expired markets

After all deadlines lapse, the admin can sweep leftover vault balance:

```ts
const ix = await client.admin.adminClaimRemainingIx({
  admin: admin.publicKey,
  marketId,
  protocolTreasury: TREASURY,
});
```

Phase gating (`marketPhase(market) === "expired"`) is enforced on-chain.
Don't run this on a market still in `claimable` or `refundable` phase
— the contract will reject with `ResolutionDeadlineNotReached`.

## Disaster recovery

| Scenario | Action |
| --- | --- |
| Bootstrap step 3 (comp def init) fails partway | Re-run — it's idempotent (skips already-initialized) |
| Bootstrap step 4 (bytecode upload) fails | Re-run just that step; comp defs persist |
| Admin keypair lost | **No recovery path** — the `admin` field on `GlobalState` is immutable. Deploy a new program. |
| IDL on npm is stale | Bump patch version, re-sync, re-publish |
| Devnet wiped between deploys | Re-bootstrap from scratch — markets, positions, all gone |
| User lost their x25519 secret | They can still claim payouts/refunds blindly; they just can't *display* their position pre-claim |
