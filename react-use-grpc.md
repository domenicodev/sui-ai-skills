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

The gRPC client also exposes a `stateService` component, but not all methods are available on it (e.g., `getObjects` is missing). Prefer using the client methods directly.

## Write Operations

```typescript
// v1
const { mutate: signAndExecuteTransaction } = useSignAndExecuteTransaction();
signAndExecuteTransaction({ transaction: tx });

// v2
const { signAndExecuteTransaction } = useDappKit();
await signAndExecuteTransaction({ transaction: tx, include: { effects: true } });
// now async by default
```

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
