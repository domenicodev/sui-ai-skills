# dAppKit: gRPC (v2)

`@mysten/dapp-kit` is deprecated. All new React dApps must use `@mysten/dapp-kit-react` (v2).

When you detect deprecated dAppKit usage (e.g., `@mysten/dapp-kit`, `useSuiClientQuery`, `useSignAndExecuteTransaction`, `SuiClientProvider`):

1. Scan the project for all deprecated patterns
2. List every occurrence with file path and line
3. For each, show the current code and the proposed v2 replacement using the patterns below
4. Wait for approval before making any changes

## Package Changes

The package is now split:
- `@mysten/dapp-kit-react` -- React bindings
- `@mysten/dapp-kit-core` -- framework-agnostic (vanilla JS, Vue, Next.js, etc.)

Key changes:
- `@tanstack/react-query` is no longer a hard dependency
- Radix UI removed, Tailwind by default
- Queries use the underlying gRPC client directly instead of `useSuiClientQuery`

## Cheatsheet (v1 → v2)

| Pattern | `@mysten/dapp-kit` (v1) | `@mysten/dapp-kit-react` (v2) |
|---------|------------------------|------------------------------|
| Reading data | `useSuiClientQuery("METHOD", { ...args })` | `const client = useCurrentClient(); client.METHOD(...)` |
| Writing data | `const { mutate: signAndExecuteTransaction } = useSignAndExecuteTransaction()` | `const { signAndExecuteTransaction } = useDappKit()` |
| Handling queries | `const { data, isLoading } = useSuiClientQuery(...)` (tanstack-managed) | `client.METHOD(...)` (handle loading/error manually or with native tanstack `useQuery`) |
| Reading object fields | `obj.data.content.fields.myField` | `obj.object.json.myField` (requires `include: { json: true }`) |
| Network config | `<SuiClientProvider networks={networkConfig} defaultNetwork="NETWORK">` in `main.tsx` | `createDappKit({ networks: [...], defaultNetwork: "NETWORK" })` in `dApp-kit.tsx` |

## Read Operations

Use the underlying client directly, same API as the SDK:

```typescript
// v1
const { data } = useSuiClientQuery("getOwnedObjects", { owner: address });

// v2
const client = useCurrentClient();
const objects = await client.listOwnedObjects({ owner: address, include: { content: true } });
```

### Reading Object Fields

`obj.data.content.fields` no longer exists. Use `obj.object.json` instead, which flattens the fields directly onto the object. Requires `include: { json: true }`.

```typescript
// v1
const { data } = useSuiClientQuery("getObject", { id, options: { showContent: true } });
const value = data.data.content.fields.myField;

// v2
const client = useCurrentClient();
const obj = await client.getObject({ objectId: id, include: { json: true } });
const value = obj.object.json.myField;
```

The gRPC client also exposes a `stateService` component, but not all methods are available on it (e.g., `getObjects` is missing). Prefer using the client methods directly.

## Write Operations

```typescript
// v1
const { mutate: signAndExecuteTransaction } = useSignAndExecuteTransaction();
signAndExecuteTransaction({ transaction: tx });

// v2
const client = useCurrentClient();
const { signAndExecuteTransaction } = useDappKit();
const result = await signAndExecuteTransaction({ transaction: tx, include: { effects: true } });
if (!result.Transaction) throw new Error('Transaction failed');
await client.waitForTransaction({ digest: result.Transaction.digest });
```

**Always check `result.Transaction` and call `waitForTransaction` after `signAndExecuteTransaction`.** `result.Transaction` may be undefined if the transaction failed. If it exists, the digest lives at `result.Transaction.digest`. Without `waitForTransaction`, any subsequent read can return stale data or a "not found" error.

### Checking Created Objects After a Transaction

To verify which objects were created by a transaction, include **both** `objectTypes` and `effects`:

```typescript
const client = useCurrentClient();
const { signAndExecuteTransaction } = useDappKit();
const result = await signAndExecuteTransaction({
  transaction: tx,
  include: { objectTypes: true, effects: true },
});
if (!result.Transaction) throw new Error('Transaction failed');
await client.waitForTransaction({ digest: result.Transaction.digest });
```

- `effects` — contains `changedObjects`, a map of every object touched by the transaction (created, mutated, deleted).
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

Without `objectTypes: true`, you cannot match objects by their Move type. Without `effects: true`, `changedObjects` is not available.

## Simulating Transactions

```typescript
// v1
const { data } = useSuiClientQuery("dryRunTransactionBlock", { transactionBlock });

// v2
const client = useCurrentClient();
const result = await client.simulateTransaction({ transaction, include: { effects: true } });
```

## Known Issues

| Issue | Description | Workaround |
|-------|-------------|------------|
| Display | Not yet supported in gRPC | Use graphqlRPC or fallback to jsonRPC `showDisplay` |
| Events | Querying past events not possible via gRPC | Use graphqlRPC or fallback to deprecated jsonRPC `queryEvents` |
