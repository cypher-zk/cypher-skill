# Cypher SDK — Architecture

## Three coupled surfaces

```
┌──────────────────────────────────────────────────────────────────┐
│  Circuit (Arcis MPC)                                              │
│  cypher-main/encrypted-ixs/src/lib.rs                             │
│  8 circuits, pure fixed-shape MPC code                            │
└────────────────────────────┬─────────────────────────────────────┘
                             │ queue + callback
┌────────────────────────────▼─────────────────────────────────────┐
│  Program (Anchor)                                                 │
│  26 ix · 4 accounts · 7 events · 36 errors                        │
└────────────────────────────┬─────────────────────────────────────┘
                             │ Anchor IDL
┌────────────────────────────▼─────────────────────────────────────┐
│  Client (@cypher-zk/sdk)                                          │
│  Typed Program · PDA helpers · actions · React hooks              │
└──────────────────────────────────────────────────────────────────┘
```

Most bugs live in the gap between two surfaces. This skill targets the bottom layer.

## Account topology

```
GlobalState (PDA ["global_state"])
  ├─ admin · treasury · fee_rates · acceptedMint
  │
  └─ market_counter ++ on createMarket
            ▼
    Market (PDA ["market", id u64 LE])
      ├─ question · state · deadlines · revealedPool0..3 · payoutRatio
      │
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
