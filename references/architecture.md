# Cypher SDK — Architecture

## Three coupled surfaces

```
┌──────────────────────────────────────────────────────────────────┐
│  Circuit (Arcium MPC)          ← Cypher team, black box for devs │
│  8 circuits, fixed-shape MPC code                                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │ queue + callback
┌────────────────────────────▼─────────────────────────────────────┐
│  Program (Anchor on-chain)     ← Cypher team, black box for devs │
│  29 ix · 5 accounts · 10 events · 45 errors                       │
└────────────────────────────┬─────────────────────────────────────┘
                             │ Anchor IDL
┌────────────────────────────▼─────────────────────────────────────┐
│  Client (@cypher-zk/sdk)       ← your surface as an app dev      │
│  Typed Program · PDA helpers · actions · React hooks              │
└──────────────────────────────────────────────────────────────────┘
```

As an app developer you only interact with the bottom layer. Errors from
the top two layers surface as `CypherError` codes — look them up in
[troubleshooting.md](troubleshooting.md#on-chain-cyphererror-codes-6000–6035).

## Account topology

```
GlobalState (PDA ["global_state"])
  ├─ admin · treasury · fee_rates · acceptedMint
  │
  └─ market_counter ++ on createMarket
            ▼
    Market (PDA ["market", id u64 LE])
      ├─ state · deadlines · revealedPool0..3 · payoutRatio · creator · category
      │
      ├─► MarketQuestion (PDA ["market_question", market]) — question text
      ├─► MarketVault (PDA ["market_vault", market]) — SPL ATA
      ├─► LPPosition (PDA ["lp-position", market, creator]) — bond + fees
      └─► EncryptedPosition (PDA ["position", market, user])
            ├─ encryptedAmount/Side · userPubkey · nonce
            └─ entryOdds · netAmount (public) · claimed

ArciumSignerAccount (PDA ["ArciumSignerAccount"]) — queue/callback CPI authority
```

## Privacy: what's encrypted vs public

| Position field | Visibility |
| --- | --- |
| `user`, `market` | Public |
| `encryptedAmount` (32 bytes) | Ciphertext (Rescue) |
| `encryptedSide` (32 bytes) | Ciphertext (Rescue) |
| `userPubkey`, `nonce` | Public (needed for decrypt) |
| `entryOdds` | Public — locked at bet time |
| **`netAmount`** | **Public** — required for on-chain vault accounting |
| `claimed` | Public |

Guarantee: **your chosen side stays secret**. Stake amount IS public.

## Resolution flow (v0.2+)

```
Resolver calls resolveMarket
       │
       ▼
  reveal_market_outcome_* (Arcium MPC)
       │
       ▼  callback writes:
         m.outcome, m.revealedPool*, m.payoutRatio
         m.state = PendingResolution (4)        ← NOT Resolved
         m.challengeDeadline = now + challenge_period

  ┌─────────────────────────────────────┐
  │  Challenge window (24h-48h)         │
  │                                     │
  │  Anyone may flagResolution          │
  │    → market.disputed = true         │
  │                                     │
  └─────────────────────────────────────┘
       │                          │
   (undisputed)               (disputed)
       │                          │
       ▼                          ▼
  finalizeResolution      adminOverrideResolution
  (anyone, after window)  (admin only, any time)
       │                          │
       ▼                          ▼
  m.state = Resolved        m.state = Resolved
  m.claimDeadline =         m.payout_ratio recomputed
  m.refundDeadline =        from plaintext pools

       │
       ▼
  claimPayout / claimRefund now open
```

## Bet flow (with progress stages)

```
User clicks "Bet"
       │
       ▼
placeBetAction
  │
  ├─ "validating"        ← market.state, amount >= minBet, side in range
  ├─ "fetching-state"    ← GlobalState · MXE pubkey · LUT (cached)
  ├─ "encrypting"        ← x25519 keypair · shared secret · nonce
  │                         RescueCipher.encrypt([netAmount, side])
  │                         → 2 × 32-byte ciphertexts
  ├─ "submitting"        ← placePrivateBetYesnoIx → sendIx → signature
  │                         (vault receives USDC, fee → treasury,
  │                          EncryptedPosition created, queue_computation)
  ├─ "awaiting-callback" ← Arcium MPC runs circuit, submits callback
  │                         (market.revealedPoolX updated, entryOdds set,
  │                          BetPlacedEvent emitted)
  ├─ "refetching"        ← client.positions.fetch(market, user)
  └─ "done"
        │
        ▼
  { signature, computation, computationOffset,
    userKeypair ← PERSIST THIS,
    position }
```

## SDK module layout

```
@cypher-zk/sdk/
├── src/
│   ├── config.ts          PROGRAM_ID, CLUSTERS, enums, constants
│   ├── pda.ts             6 PDA derivations
│   ├── wallet.ts          Wallet interface + adapters
│   ├── fees.ts            computeFees (mirrors on-chain)
│   ├── deadlines.ts       marketPhase + projectDeadlines
│   ├── errors.ts          CypherErrorCode + parseCypherError
│   ├── client.ts          CypherClient — namespace facade
│   ├── accounts/          per-account fetch + memcmp filters + layout
│   ├── arcium/            x25519 + RescueCipher + queue accounts + callbacks
│   ├── instructions/      raw TransactionInstruction builders (26 ix)
│   ├── actions/           high-level flows + progress events
│   ├── events/            parser (camelize+normalize) + subscribe/poll
│   ├── idl/               synced from cypher-main
│   └── node/              admin-only (circuit upload)
├── react/src/             @cypher-zk/sdk/react hooks
└── dist/                  ESM + .d.ts
```

## Cluster strategy

**Cluster-agnostic at runtime** — reads `GlobalState.accepted_mint` per flow. Same compiled SDK works against any deployment.

| Cluster | RPC | Mint | Arcium offset |
| --- | --- | --- | --- |
| `devnet` | `api.devnet.solana.com` | CSDC (`8AF9BABN…`) | `456` |
| `mainnet` | `api.mainnet-beta.solana.com` | USDC (`EPjFWdd5…`) | set on deploy |
| `localnet` | `localhost:8899` | CSDC test | `1116522022` |

`KNOWN_MINTS.*` are **informational only**.

## Drift protection

| Test | Catches |
| --- | --- |
| `idl.test.ts` | Instruction name additions/removals |
| `layout.test.ts` | Account field reorders → broken memcmp |
| `errors.test.ts` | New error variants outside `CypherErrorCode` |
| `events-roundtrip.test.ts` | Event field rename / type change |
| `prepublishOnly` | sync IDL → typecheck → tests → build, pre-publish |

## Where Arcium sits

```
┌─────────────────┐   queue_computation    ┌──────────────────┐
│ Cypher program  │ ────────────────────► │  Arcium MXE      │
│ (Anchor)        │ ◄──── callback ─────── │  (MPC cluster)   │
└────────┬────────┘                        └──────────────────┘
         ▲                                          │
         │ raw ix                                   │ encrypted I/O
         │                                          │
┌────────┴────────┐                        ┌────────▼─────────┐
│  CypherClient   │ ◄── shared secret ─────│ User's x25519    │
│  arcium/cipher  │                        │ keypair (client) │
└─────────────────┘                        └──────────────────┘
```

For Arcium-side details: bundled `arcium` skill.

## Performance characteristics

| Operation | Typical |
| --- | --- |
| `globalState.fetch()` | 100–300ms |
| `markets.fetch(id)` | 100–300ms |
| `markets.byCreator(pk)` | 200–800ms |
| `markets.all()` | seconds (avoid in hot paths) |
| `placeBet` (client-side prep) | <50ms |
| `placeBet` (full E2E) | 8–30s (localnet → devnet) |
| Event parse | <1ms per event |

Use Helius/QuickNode/Triton for 3–5× speedup on `getProgramAccounts`.
