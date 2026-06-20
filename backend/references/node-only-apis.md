# The `@cypher-zk/sdk/node` subpath

The SDK ships three importable subpaths:

```ts
import { ... } from "@cypher-zk/sdk";          // browser-safe core + Node
import { ... } from "@cypher-zk/sdk/react";    // React hooks
import { ... } from "@cypher-zk/sdk/node";     // Node-only (circuit upload — Cypher team only)
```

## What you need as an app developer

Everything you need as an app developer is in the **core** subpath:

- `CypherClient`, all action helpers, all fetchers — work in both browser and Node
- `keypairToWallet`, `readonlyWallet` — Node-safe
- `parseLogs`, `pollEvents`, `subscribeAll` — Node-safe
- Encryption helpers (`createUserKeypair`, `encryptBetInput`, `decryptBetInput`) — Node-safe

## What's in `/node` (and why you don't need it)

The `/node` subpath contains `uploadCypherCircuit`, which uploads Arcium
circuit bytecode to IPFS. This is a Cypher team operation performed once
per deployment — app developers never call it.

It depends on `node:fs` and is split out so browser bundlers don't choke:

- **Vite**: `Module "node:fs" has been externalized for browser compatibility`
- **Webpack**: `Module not found: Can't resolve 'node:fs'`
- **esbuild**: `Could not resolve "node:fs"`

If you see one of these errors, you have a `@cypher-zk/sdk/node` import
in a browser-bound file. Remove it — you shouldn't need it.
