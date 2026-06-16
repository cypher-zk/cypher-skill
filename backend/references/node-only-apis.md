# Backend reference — `@cypher-zk/sdk/node` subpath

The SDK has three subpaths:

```ts
import { ... } from "@cypher-zk/sdk";          // browser-safe core
import { ... } from "@cypher-zk/sdk/react";    // React hooks
import { ... } from "@cypher-zk/sdk/node";     // Node-only admin helpers
```

The `node` subpath holds anything that depends on `node:fs`,
`node:crypto`, or other Node-exclusive APIs. **Never import from `node`
in browser code** — it will break the build.

## What lives in `node`

| Export | Purpose |
| --- | --- |
| `uploadCypherCircuit({...})` | Read `.arcis` bytecode from disk and upload to IPFS + register on chain |
| (more as needed) | (currently this is the only public node-only helper) |

Everything else — `CypherClient`, instruction builders, action helpers,
events, fees, deadlines, errors, encryption — lives in the core
subpath and works in both browser and Node.

## Why the split

Browser bundlers (Vite, webpack, esbuild) refuse to bundle `node:fs`,
`node:crypto`, etc. Without the split, importing `CypherClient` in a
browser would pull in `uploadCypherCircuit` → `node:fs` → bundle error.

The split also makes intent obvious in code review: a `from
"@cypher-zk/sdk/node"` import means "this code never runs in a browser",
no further inspection needed.

## When you'll use it

Only during admin/bootstrap scripts that upload circuit bytecode. Step 4
of [bootstrap.md](../examples/bootstrap.md) is the canonical use case.

```ts
import { uploadCypherCircuit } from "@cypher-zk/sdk/node";
import { Keypair } from "@solana/web3.js";

await uploadCypherCircuit({
  client,
  circuitName: "place_private_bet_yesno",
  arcisPath: "/abs/path/to/cypher-main/build/place_private_bet_yesno.arcis",
  payer: adminKeypair,
});
```

The helper wraps `@arcium-hq/client`'s `uploadCircuit`, which chunks the
bytecode into 814-byte segments (Arcium's per-tx upload limit) and
submits them sequentially. Expect ~30-60 seconds per circuit.

## Bundler hygiene

If you accidentally import from `@cypher-zk/sdk/node` in a browser-bound
file, bundlers report this as:

- **Vite**: `Module "node:fs" has been externalized for browser
  compatibility`
- **Webpack**: `Module not found: Can't resolve 'node:fs'`
- **esbuild**: `Could not resolve "node:fs"`

The fix is always: stop importing from `/node` in that file.

## What is NOT in `node`

These are Node-friendly but live in core because they don't require any
Node-only APIs:

- `keypairToWallet(Keypair)` — uses `@solana/web3.js` `partialSign`,
  browser-safe
- All read-only fetchers (`client.markets.*`, `client.events.pollEvents`)
- Event parsing (`parseLogs`, `parseLogsFor`)
- Encryption helpers (`createUserKeypair`, `encryptBetInput`)

Use them from Node scripts directly via the core import.
