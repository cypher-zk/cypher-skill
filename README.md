# @cypher-zk/skill

> Agent skill for building apps on the Cypher privacy-preserving prediction market with [`@cypher-zk/sdk`](https://www.npmjs.com/package/@cypher-zk/sdk).

Drop-in instructions for Claude / Cursor / any agent that speaks the
[Anthropic Agent Skill format](https://www.anthropic.com/news/agent-skills) so
it can wire frontends (React or vanilla), write Node admin scripts, place
encrypted bets via Arcium MPC, decode events, and avoid the silent-failure
mistakes that crop up around encryption-param ordering and stage-aware UI.

## Install

```bash
# Once your editor/agent supports it
npx skills add cypher-zk/cypher-skill

# Or the alternative installer
npx add-skill cypher-zk/cypher-skill
```

The skill registers as `cypher` and exposes two sub-skills:

- [`frontend/`](./frontend/SKILL.md) — browser, React, wallet adapter, hooks, progress UI
- [`backend/`](./backend/SKILL.md) — Node, admin scripts, IDL sync, indexers, integration tests

Plus shared references:

- [`references/api-map.md`](./references/api-map.md) — every SDK export with signature + intent
- [`references/troubleshooting.md`](./references/troubleshooting.md) — symptoms → diagnoses
- [`references/architecture.md`](./references/architecture.md) — three-surface model + account topology + bet trace

## Why this skill exists

Working with `@cypher-zk/sdk` involves several non-obvious gotchas that
agents tend to hallucinate around:

1. **The user's x25519 keypair from `placeBet` MUST be persisted** — without
   it the user can never decrypt their own position to claim a payout. The
   skill is loud about this in every relevant section.
2. **Event payloads are camelCase `bigint`, not snake_case BN** — Anchor's
   `BorshCoder` returns snake_case BN, but `@cypher-zk/sdk`'s `parseLogs`
   normalizes both. Agents that try to read `data.entry_odds` get
   `undefined`. The skill enforces camelCase access.
3. **React mutation flows have multiple async stages** — encrypt, submit,
   await MPC callback (~10s). Without progress events the UI looks frozen.
   The skill teaches the `onProgress` callback pattern.
4. **`marketPhase` gates which claim is available** — `claimPayout` works
   in `"claimable"` phase, `claimRefund` in `"refundable"`. UI buttons
   must mirror this or burn user gas.
5. **The SDK is cluster-agnostic at runtime** — the accepted SPL mint
   comes from `GlobalState.accepted_mint`, not from a constant.
   Hard-coding `KNOWN_MINTS.devnetCSDC` is a silent bug on mainnet.

## Versioning

This skill targets `@cypher-zk/sdk@^0.1.0`. The skill's `metadata.version`
in `SKILL.md` tracks SDK compatibility. Upgrade the skill alongside any
SDK major-version bump.

## Layout

```
cypher-skill/
├── SKILL.md             # main skill — frontmatter, intent router, anti-hallucination
├── README.md            # this file
├── package.json         # for npx install
├── frontend/
│   ├── SKILL.md         # browser + React sub-skill
│   ├── examples/        # complete code recipes (place-bet, resolve, claim, …)
│   └── references/      # frontend-specific reference material
├── backend/
│   ├── SKILL.md         # Node + admin sub-skill
│   ├── examples/        # scripts (init-comp-defs, indexer, devnet-smoke, …)
│   └── references/      # backend-specific reference material
├── references/
│   ├── api-map.md       # complete SDK surface
│   ├── troubleshooting.md
│   └── architecture.md
└── evals/
    └── evals.json       # discrimination prompts to test agent understanding
```

## Contributing

Found a hallucination the skill missed? Open a PR adding the failure mode
to `references/troubleshooting.md` and a probe to `evals/evals.json`.

## License

MIT. See [LICENSE](./LICENSE) once published.
