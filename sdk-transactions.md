# SDK: Transaction Optimization

When reviewing or writing TypeScript SDK code that builds and executes transactions, detect and flag these patterns.

---

## 1. Always Set Gas Budget

### Anti-pattern: No explicit gas budget

```typescript
// BAD: relies on SDK estimation, which may under/overestimate
const tx = new Transaction();
tx.moveCall({ ... });
await client.signAndExecuteTransaction({ transaction: tx, signer: keypair });
```

### Fix: Set gas budget explicitly

```typescript
const tx = new Transaction();
tx.setGasBudget(50_000_000); // in MIST (0.05 SUI)
tx.moveCall({ ... });
await client.signAndExecuteTransaction({ transaction: tx, signer: keypair });
```

Set the budget based on the expected complexity of the transaction. Common ranges:
- Simple transfers: `10_000_000` (0.01 SUI)
- Single Move call: `50_000_000` (0.05 SUI)
- Complex PTB with multiple calls: `100_000_000` (0.1 SUI)

If unsure, simulate first with `client.simulateTransaction` and use the gas cost from the result as the baseline, with a small buffer.

---

## 2. Merge Sequential Transactions into One PTB

### Anti-pattern: Multiple transactions executed sequentially

Detect patterns where multiple `Transaction` objects are created and executed one after another in the same flow:

```typescript
// BAD: two separate transactions when one PTB would suffice
async function setupAndDeposit(pool: string, coin: string) {
    const tx1 = new Transaction();
    tx1.moveCall({ target: `${pkg}::pool::initialize`, arguments: [tx1.object(pool)] });
    await client.signAndExecuteTransaction({ transaction: tx1, signer: keypair });

    const tx2 = new Transaction();
    tx2.moveCall({ target: `${pkg}::pool::deposit`, arguments: [tx2.object(pool), tx2.object(coin)] });
    await client.signAndExecuteTransaction({ transaction: tx2, signer: keypair });
}
```

### Fix: Combine into a single Programmable Transaction Block

```typescript
async function setupAndDeposit(pool: string, coin: string) {
    const tx = new Transaction();
    tx.moveCall({ target: `${pkg}::pool::initialize`, arguments: [tx.object(pool)] });
    tx.moveCall({ target: `${pkg}::pool::deposit`, arguments: [tx.object(pool), tx.object(coin)] });
    await client.signAndExecuteTransaction({ transaction: tx, signer: keypair });
}
```

One transaction = one signature, one gas payment, atomic execution. If the first call fails, the second doesn't execute with stale state.

### Cross-file detection

Look for any flow where a caller awaits multiple functions that each internally create and execute their own `Transaction`. The function names will vary -- what matters is the pattern: separate functions, each with their own `new Transaction()` + `signAndExecuteTransaction`, called sequentially from the same caller.

Example:

```typescript
// file: initPool.ts
export async function initPool(pool: string) {
    const tx = new Transaction();
    tx.moveCall({ target: `${pkg}::pool::initialize`, arguments: [tx.object(pool)] });
    await client.signAndExecuteTransaction({ transaction: tx, signer: keypair });
}

// file: deposit.ts
export async function deposit(pool: string, coin: string) {
    const tx = new Transaction();
    tx.moveCall({ target: `${pkg}::pool::deposit`, arguments: [tx.object(pool), tx.object(coin)] });
    await client.signAndExecuteTransaction({ transaction: tx, signer: keypair });
}

// file: setup.ts
await initPool(poolId);
await deposit(poolId, coinId); // <-- two transactions in one flow
```

**Propose refactoring** to builder functions that add commands to a shared transaction:

```typescript
// file: initPool.ts
export function buildInitPool(tx: Transaction, pool: string) {
    tx.moveCall({ target: `${pkg}::pool::initialize`, arguments: [tx.object(pool)] });
}

// file: deposit.ts
export function buildDeposit(tx: Transaction, pool: string, coin: string) {
    tx.moveCall({ target: `${pkg}::pool::deposit`, arguments: [tx.object(pool), tx.object(coin)] });
}

// file: setup.ts
const tx = new Transaction();
buildInitPool(tx, poolId);
buildDeposit(tx, poolId, coinId);
await client.signAndExecuteTransaction({ transaction: tx, signer: keypair });
```

Each builder adds its commands to the transaction without executing. The caller controls when to sign and execute.

**When NOT to merge:**
- The transactions are in completely separate user flows with no shared caller
- The second transaction depends on on-chain state changes from the first that must be committed (e.g., object creation that needs to be indexed before the next step)
- Different signers are required for each transaction

---

## Detection Checklist

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Missing gas budget | `Transaction` without `setGasBudget` | Add explicit `setGasBudget` |
| Sequential transactions | Multiple `signAndExecuteTransaction` in one flow | Merge into single PTB with builder functions |
