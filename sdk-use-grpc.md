# SDK: gRPC (v2)

jsonRPC (`SuiClient` from `@mysten/sui/client`) is deprecated. All new code must use gRPC (`SuiGrpcClient` from `@mysten/sui/grpc`).

When you detect jsonRPC usage in the codebase (e.g., `SuiClient`, `options: { showX }`, `getOwnedObjects`):

1. Scan the project for all deprecated patterns
2. List every occurrence with file path and line
3. For each, show the current code and the proposed gRPC replacement using the patterns below
4. Wait for approval before making any changes

## Client Initialization

```typescript
// v1 (jsonRPC)
import { SuiClient } from "@mysten/sui/client";
const client = new SuiClient({ url: "https://fullnode.mainnet.sui.io" });

// v2 (gRPC)
import { SuiGrpcClient } from "@mysten/sui/grpc";
const client = new SuiGrpcClient({ baseUrl: "https://fullnode.mainnet.sui.io", network: "mainnet" });
```

The gRPC client requires a `network` field instead of a raw URL.

## Cheatsheet (v1 → v2)

| Pattern | jsonRPC (v1) | gRPC (v2) |
|---------|-------------|-----------|
| Including outputs | `options: { showX: boolean }` | `include: { X: boolean }` |
| Reading output data | `.data.data.{...}` | `.resultType` (e.g., `.objects`) |
| Querying multiple items | `get*s` (e.g., `getCoins`, `getOwnedObjects`) | `list*s` (e.g., `listCoins`, `listOwnedObjects`) |
| Parsing types | `ResultType` | `SuiClientTypes.ResultType{...includes}` |

## Read Operations

### Owned Objects

```typescript
// v1
const objects = await client.getOwnedObjects({
  owner: address,
  options: { showContent: true },
});
const content = objects.data[0].data.content;

// v2
const objects = await client.listOwnedObjects({
  owner: address,
  include: { content: true },
});
// content is now in bcs; use "json" for the old content format (Fields)
```

### Objects (single or multiple)

```typescript
// v1
const objects = await client.multiGetObjects({
  ids: objectIds,
  options: { showContent: true, showOwner: true },
});

// v2
const objects = await client.getObjects({
  objectIds,
  include: { content: true, owner: true },
});
// include parameters are of type ObjectInclude, shared across getObject/getObjects/listOwnedObjects
```

### Dynamic Fields

```typescript
// v1
const dfs = await client.getDynamicFields({ parentId });
const dfObject = await client.getDynamicFieldObject({ parentId, name });

// v2
const dfs = await client.listDynamicFields({ parentId });
const dfObject = await client.getDynamicField({ parentId, name });
```

`getDynamicFieldObject` → `getDynamicField`.

## Write Operations

### Signing and Executing Transactions

```typescript
// v1
const result = await client.signAndExecuteTransaction({
  transaction: tx,
  signer: keypair,
  options: { showEffects: true, showObjectChanges: true },
});

// v2
const result = await client.signAndExecuteTransaction({
  transaction: tx,
  signer: keypair,
  include: { effects: true, objectChanges: true },
});
```

`options` → `include`. The rest of the API is similar but has breaking changes in response types.

### Simulating Transactions

```typescript
// v1 (two separate methods)
const dryRun = await client.dryRunTransactionBlock({ transactionBlock });
const devInspect = await client.devInspectTransactionBlock({ sender, transactionBlock });

// v2 (merged into one)
const simulation = await client.simulateTransaction({ transaction, include: { effects: true } });
```

`dryRun` and `devInspect` are merged into `simulateTransaction`.

## Known Issues and Limitations

| Issue | Description | Workaround |
|-------|-------------|------------|
| Events | Querying past events not possible via gRPC | Use graphqlRPC or fallback to deprecated jsonRPC `queryEvents` |
| Display | Display not yet supported in gRPC | Use graphqlRPC or fallback to jsonRPC `showDisplay` in options |
| Jest | Not supported (SDK uses ESM) | Migrate to vitest, or configure Jest to handle ESM |
| IDE Autocompletion | Type/argument autocompletion often broken | Explicitly import types from SDK source, or use `SuiClientTypes` |
| Object Changes | `idOperation` has ambiguous `"None"` and `"Unknown"` values | Treat `"None"` as mutated; investigate `"Unknown"` case by case |
