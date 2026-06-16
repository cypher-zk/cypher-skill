# Recipe — Sync IDL + verify drift

## Manual sync

```bash
cd cypher-sdk

# Default: sync from ../cypher-main
bun run sync:idl

# Custom source
bun run sync:idl --from /abs/path/to/cypher-main

# Via env var
CYPHER_MAIN_PATH=/abs/path bun run sync:idl
```

`scripts/sync-idl.ts` copies two files into `src/idl/`:
- `cypher.json` — the runtime IDL (snake_case names)
- `cypher.ts` — the typed view (camelCase, used by Anchor's `Program<Cypher>`)

It fails loudly if either source file is missing, with a hint to run
`anchor build` upstream first.

## Verify drift after sync

The SDK has three drift-detection tests. Run all three after every sync:

```bash
bun test tests/unit/layout.test.ts
bun test tests/unit/idl.test.ts
bun test tests/unit/errors.test.ts
```

| Test | Catches |
| --- | --- |
| `layout.test.ts` | Account field reorders that would break memcmp filters |
| `idl.test.ts` | Instruction name additions/removals/renames |
| `errors.test.ts` | New error variants not yet in `CypherErrorCode` |

If any of these fail, the affected SDK file needs a manual update:

| Failing test | Update this file |
| --- | --- |
| `layout.test.ts` | `src/accounts/layout.ts` byte-offset constants |
| `idl.test.ts` | `src/index.ts` exports + relevant instruction builder |
| `errors.test.ts` | `src/errors.ts` `CypherErrorCode` enum |

## CI gate

Add this to your CI workflow before any publish step:

```yaml
- name: Sync IDL
  run: bun run sync:idl --from ./cypher-main

- name: Fail if IDL drifted
  run: |
    git diff --exit-code src/idl/ || (
      echo "::error::IDL drifted from cypher-main — regenerate locally and commit"
      exit 1
    )

- name: Drift tests
  run: |
    bun test tests/unit/layout.test.ts
    bun test tests/unit/idl.test.ts
    bun test tests/unit/errors.test.ts
```

## What you cannot fix by syncing

Some upstream changes need code work beyond regenerating offsets:

| Upstream change | SDK action |
| --- | --- |
| New instruction added | Add a builder in `src/instructions/`, wire into `client.ts` namespace |
| New event added | Add typed interface in `src/events/parser.ts`, add to `CypherEvent` union |
| New error added | Add entry to `CypherErrorCode` enum in `src/errors.ts` |
| PDA seed change | Update `src/pda.ts` (drift test will catch with a failing seed comparison) |
| Account field renamed (snake_case) | Update `src/accounts/<acct>.ts` decoder + the typed `<Acct>Account` interface |
| Account field reordered | Just update `src/accounts/layout.ts` constants (drift test tells you new offsets) |

## What happens if you skip sync

The IDL on npm gets stale. Symptoms:

- `unknown program method 'fooBarBaz'` errors at runtime from instruction builders
- `expect(IDL.instructions.length).toBeGreaterThan(20)` fails in `idl.test.ts`
- Memcmp filters return zero accounts because field offsets shifted
- New `CypherErrorCode` values fall through `parseCypherError` and return `null`

The `prepublishOnly` script (`bun run sync:idl && bun run typecheck &&
bun run test:unit && bun run build`) catches all of this before
publish — never bypass it with `npm publish --no-scripts`.
