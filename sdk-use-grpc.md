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
| Reading object fields | `obj.data.content.fields.myField` | `obj.object.json.myField` (requires `include: { json: true }`) |
| Getting an object | `getObject({ id })` | `getObject({ objectId })` |
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

### Reading Object Fields

`obj.data.content.fields` no longer exists. Use `obj.object.json` instead, which flattens the fields directly onto the object. Requires `include: { json: true }`.

```typescript
// v1
const obj = await client.getObject({ id, options: { showContent: true } });
const value = obj.data.content.fields.myField;

// v2
const obj = await client.getObject({ objectId: id, include: { json: true } });
const value = obj.object.json.myField;
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
  include: { effects: true, objectTypes: true },
});
if (!result.Transaction) throw new Error('Transaction failed');
await client.waitForTransaction({ digest: result.Transaction.digest });
```

`options` → `include`. The rest of the API is similar but has breaking changes in response types.

**Always check `result.Transaction` and call `waitForTransaction` after `signAndExecuteTransaction`.** `result.Transaction` may be undefined if the transaction failed. If it exists, the digest lives at `result.Transaction.digest`. Without `waitForTransaction`, any subsequent read (e.g. `getObject`) can return stale data or a "not found" error.

### Checking Created Objects After a Transaction

To verify which objects were created (or mutated) by a transaction, include **both** `objectTypes` and `effects`:

```typescript
const result = await client.signAndExecuteTransaction({
  transaction: tx,
  signer: keypair,
  include: { objectTypes: true, effects: true },
});
if (!result.Transaction) throw new Error('Transaction failed');
await client.waitForTransaction({ digest: result.Transaction.digest });
```

- `effects` — contains `changedObjects`, a map of every object touched by the transaction (created, mutated, deleted) with their owner and digest.
- `objectTypes` — attaches the full type string (e.g. `0x…::pool::Pool`) to each changed object entry.

All response data lives under `result.Transaction`. For example: `result.Transaction.effects`, `result.Transaction.objectTypes`, `result.Transaction.events`.

- `objectTypes` maps `[objectId] → full type string` (e.g. `"0x…::pool::Pool"`)
- `effects.changedObjects` maps `[objectId] → { ...metadata, idOperation: string }`

`idOperation` values: `"Created"` (new object), `"None"` (mutated), `"Deleted"` (destroyed). Use these to filter by operation type.

To find a specific created object by type:

```typescript
const tx = result.Transaction!;
const changedObjects = tx.effects?.changedObjects ?? [];
const created = changedObjects.find(
  (obj) =>
    obj.idOperation === 'Created' &&
    tx.objectTypes?.[obj.objectId]?.includes('::pool::Pool'),
);
```

Without `objectTypes: true`, the type information is not included and you cannot match objects by their Move type. Without `effects: true`, `changedObjects` is not available.

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
| Changed Objects | `effects.changedObjects` entries have `idOperation` with ambiguous `"None"` and `"Unknown"` values | Treat `"None"` as mutated; investigate `"Unknown"` case by case |
