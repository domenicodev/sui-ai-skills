# E2E Tests (TypeScript)

E2E tests exercise Move contracts through the TypeScript SDK on a live network. They mirror the Move unit tests 1-to-1 — same functionalities, same flow — while also covering the integration boundary: PTB composition, event parsing, client-side serialization, and cross-package interactions after publishing.

## Test Structure

Each test file has **one top-level `describe`** block named after the Move module or domain. Inside it, use only `it(...)` blocks — no nested `describe` blocks. Group related tests by proximity, not by wrapping them in sub-describes.

```typescript
// GOOD: one describe, flat it blocks
describe('pool', () => {
  it('creates a pool with zero liquidity', async () => { ... });
  it('creates multiple pools in sequence', async () => { ... });
  it('rejects swap on empty pool', async () => { ... });
  it('emits a PoolCreated event', async () => { ... });
});

// BAD: multiple describes or nested describes
describe('pool creation', () => {
  it('creates a pool', async () => { ... });
});
describe('pool swap', () => {
  it('rejects swap on empty pool', async () => { ... });
});
```

## File Organization

E2E tests live in a `tests/` directory at the TypeScript project root, alongside `package.json`. Name test files after the Move module or domain they cover — matching the Move test file names.

```
project/
├── move/
│   ├── sources/
│   │   └── pool.move
│   ├── tests/
│   │   └── pool_tests.move
│   ├── Move.toml
│   └── Move.lock
├── src/
│   └── helpers/
│       ├── getSigner.ts
│       └── suiClient.ts
├── tests/
│   ├── pool.test.ts
│   └── pool.utils.ts
├── .env
├── package.json
└── tsconfig.json
```

Each Move test file (`pool_tests.move`) should have a corresponding TypeScript test file (`pool.test.ts`). A `.utils.ts` file is only needed when the test file duplicates many behaviors (shared transaction builders, assertion helpers, object lookups). If the tests are straightforward, keep everything in the test file itself.

## TypeScript Project Root Detection

In a combined Move + TypeScript workspace, the TypeScript code often lives in a subdirectory (e.g. `ts/`, `sdk/`, `app/`) rather than the repository root. Before running any command (`install`, `test`, etc.), detect the TypeScript project root:

1. Find `package.json` files that list `@mysten/sui` as a dependency.
2. If the matching `package.json` is **not** at the workspace root, the directory containing it is the TypeScript project root.
3. Confirm the `.env` file is in (or expected in) that directory — `dotenv.config()` loads `.env` from the **current working directory**, not from the test file's location.

**All TypeScript commands** (`install`, `test`, `vitest run`, etc.) **must be run from the directory that contains `.env`**, not the workspace root. Always `cd` there before running anything. For example, if `.env` and `package.json` are in `ts/`:

```bash
cd ts && <pm> test
```

Do **not** add workarounds like importing `loadEnv` from `vite`, setting custom `envDir` in `vitest.config.ts`, or using `dotenv.config({ path: ... })` with a custom path. These are never the correct fix. The only correct fix is to `cd` to the directory that contains `.env`.

## Package Manager Detection

Before running any install or script command, detect the project's package manager by checking for lock files in the project root. If multiple lock files exist, use the highest-priority match:

| Priority | Lock file | Package manager |
|----------|-----------|-----------------|
| 1 (highest) | `bun.lock` or `bun.lockb` | `bun` |
| 2 | `yarn.lock` | `yarn` |
| 3 | `package-lock.json` | `npm` |

If no lock file is found, default to `npm`.

Use the detected package manager for **all** install and run commands throughout the project (e.g. `bun add`, `yarn add`, `npm install`; `bun test`, `yarn test`, `npm test`).

## Dependencies

```bash
<pm> install -D vitest @mysten/sui dotenv
```

> Replace `<pm> install` with the correct command for the detected package manager: `npm install`, `yarn add`, or `bun add`.

`zod` is optional but recommended for env validation:

```bash
<pm> install -D zod
```

Add test scripts to `package.json`:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

## Environment Configuration

Tests connect to a real network using credentials from `.env`. **Never hardcode network URLs, secret keys, or object IDs.**

```
NETWORK=testnet
USER_SECRET_KEY=suiprivkey1...
PACKAGE_ID=0x...
```

Additional environment variables (object IDs, admin caps, etc.) can be added as needed per project:

```
SOME_OBJECT_ID=0x...
ADMIN_CAP_ID=0x...
```

### Loading and asserting env vars

**Always use `dotenv`.** Call `dotenv.config()` in `beforeAll` before accessing any env var, regardless of test runner. After loading, **assert that all required vars are present** — never assume they exist.

Prefer `zod` + `TestUtils` when there is much to test (many operations, shared state, repeated transaction patterns). For simple test files with a handful of self-contained checks, manual assertions without a utils class are fine.

- **Simple** (few tests, no shared state): assert env vars directly in `beforeAll`.
- **Complex** (many tests, shared setup): instantiate `TestUtils` in `beforeAll` — the constructor validates env vars with `zod`.

**Manual assertion** (simple tests):

```typescript
import dotenv from 'dotenv';

beforeAll(() => {
  dotenv.config();
  if (!process.env.NETWORK) throw new Error('NETWORK is not set in .env');
  if (!process.env.USER_SECRET_KEY) throw new Error('USER_SECRET_KEY is not set in .env');
  if (!process.env.PACKAGE_ID) throw new Error('PACKAGE_ID is not set in .env');
});
```

**With `zod`** (preferred for complex tests):

```typescript
import dotenv from 'dotenv';
import { z } from 'zod';

const envSchema = z.object({
  NETWORK: z.enum(['testnet', 'mainnet', 'devnet', 'localnet']),
  USER_SECRET_KEY: z.string().startsWith('suiprivkey'),
  PACKAGE_ID: z.string().startsWith('0x'),
});

beforeAll(() => {
  dotenv.config();
  envSchema.parse(process.env);
});
```

### Shared helpers

These helpers read from `process.env` directly. Ensure `dotenv.config()` has been called in `beforeAll` before these helpers are invoked.

`src/helpers/suiClient.ts`:

```typescript
import dotenv from 'dotenv';
import { SuiClient, getJsonRpcFullnodeUrl } from '@mysten/sui/client';

dotenv.config();

const network = process.env.NETWORK as 'testnet' | 'mainnet' | 'devnet' | 'localnet';
export const suiClient = new SuiClient({ url: getJsonRpcFullnodeUrl(network) });
```

**If the project already has a file that exports an initialized Sui client** (`SuiClient`, `SuiGrpcClient`, etc.) **and reads `process.env` at module scope** — regardless of file name (`suiClient.ts`, `client.ts`, `sui.ts`, etc.) — it **must** call `dotenv.config()` at the top of that file before accessing any env var. Otherwise the client will be initialized with `undefined` values because the module executes before any `beforeAll` in the test files.

`src/helpers/getSigner.ts`:

```typescript
import { decodeSuiPrivateKey } from '@mysten/sui/cryptography';
import { Ed25519Keypair } from '@mysten/sui/keypairs/ed25519';

export function getSigner(): Ed25519Keypair {
  const { secretKey } = decodeSuiPrivateKey(process.env.USER_SECRET_KEY!);
  return Ed25519Keypair.fromSecretKey(secretKey);
}
```

## Keep Tests Simple When Possible

Not every test needs a utils class. If you are testing a single read or a self-contained operation, keep everything in the test file.

```typescript
import { describe, it, expect, beforeAll } from 'vitest';
import dotenv from 'dotenv';
import { suiClient } from '../src/helpers/suiClient';

beforeAll(() => {
  dotenv.config();
  if (!process.env.POOL_ID) throw new Error('POOL_ID is not set in .env');
});

describe('pool reads', () => {
  it('can fetch the pool object', async () => {
    const obj = await suiClient.getObject({
      objectId: process.env.POOL_ID!,
      include: { json: true },
    });
    expect(obj.object).toBeDefined();
  });
});
```

## TestUtils Pattern

When a test file covers a module with many related operations, create a `TestUtils` class that mirrors the `#[test_only]` helpers in Move. The constructor asserts env vars and initializes the signer.

`tests/pool.utils.ts`:

```typescript
import { Transaction, type TransactionResult } from '@mysten/sui/transactions';
import { suiClient } from '../src/helpers/suiClient';
import { getSigner } from '../src/helpers/getSigner';

export class TestUtils {
  private packageId: string;
  private signer;

  constructor() {
    if (!process.env.PACKAGE_ID) throw new Error('PACKAGE_ID is not set in .env');
    this.packageId = process.env.PACKAGE_ID;
    this.signer = getSigner();
  }

  prepareTransaction(tx: Transaction, target: string, args: TransactionResult[]) {
    tx.moveCall({
      target: `${this.packageId}::${target}`,
      arguments: [...args],
    });
  }

  async sendTransaction(tx: Transaction) {
    const result = await suiClient.signAndExecuteTransaction({
      signer: this.signer,
      transaction: tx,
      include: { objectTypes: true, effects: true },
    });
    if (!result.Transaction) throw new Error('Transaction failed');
    await suiClient.waitForTransaction({ digest: result.Transaction.digest });
    return result;
  }

  findCreatedObject(result: any, typeSuffix: string) {
    const tx = result.Transaction!;
    const changedObjects = tx.effects?.changedObjects ?? [];
    const entry = changedObjects.find(
      (obj: any) =>
        obj.idOperation === 'Created' &&
        tx.objectTypes?.[obj.objectId]?.includes(typeSuffix),
    );
    return entry ? { objectId: entry.objectId, objectType: tx.objectTypes?.[entry.objectId] } : undefined;
  }
}
```

`tests/pool.test.ts`:

```typescript
import { describe, it, expect, beforeAll } from 'vitest';
import dotenv from 'dotenv';
import { Transaction } from '@mysten/sui/transactions';
import { TestUtils } from './pool.utils';

let testUtils: TestUtils;

beforeAll(() => {
  dotenv.config();
  testUtils = new TestUtils();
});

describe('pool', () => {
  it('creates a pool', async () => {
    const tx = new Transaction();
    testUtils.prepareTransaction(tx, 'pool::create', []);

    const result = await testUtils.sendTransaction(tx);
    const created = testUtils.findCreatedObject(result, '::pool::Pool');
    expect(created).toBeDefined();
  });
});
```

## Mirroring Move Tests

Each Move test function should have a corresponding TypeScript `it(...)` block that tests the same thing through the SDK. The TypeScript test exercises the real network path while the Move test exercises the VM path.

**Move unit test:**

```move
#[test]
fun test_create_pool() {
    let mut ctx = tx_context::dummy();
    let pool = pool::new(&mut ctx);
    assert!(pool::liquidity(&pool) == 0);
    std::unit_test::destroy(pool);
}
```

**Corresponding TypeScript e2e test:**

```typescript
it('creates a pool with zero liquidity', async () => {
  const tx = new Transaction();
  testUtils.prepareTransaction(tx, 'pool::new', []);

  const result = await testUtils.sendTransaction(tx);
  const created = testUtils.findCreatedObject(result, '::pool::Pool');
  expect(created).toBeDefined();

  const obj = await suiClient.getObject({
    objectId: created!.objectId,
    include: { json: true },
  });
  expect(obj.object.json.liquidity).toBe('0');
});
```

Both test the same logic. The Move test uses `tx_context::dummy()` + `destroy`. The TypeScript test uses the SDK + real transactions.

### Handling transferred or destroyed objects

In Move unit tests, objects are often cleaned up with `transfer::public_transfer(obj, @sender)` or `std::unit_test::destroy(obj)`. On a real network, every object produced by a PTB that doesn't have a `drop` ability must have a destination — objects left dangling cause the transaction to fail. When the Move function **returns** an object (instead of transferring it internally), the TypeScript PTB must transfer it explicitly:

```typescript
const tx = new Transaction();
const pool = tx.moveCall({
  target: `${process.env.PACKAGE_ID}::pool::new`,
});
tx.transferObjects([pool], getSigner().toSuiAddress());

const result = await testUtils.sendTransaction(tx);
```

Use `getSigner().toSuiAddress()` to obtain the current wallet address as the transfer recipient. This applies to any `moveCall` whose Move function returns one or more objects — wrap them with `tx.transferObjects` so the transaction succeeds.

**Keep them in sync**: whenever a Move test is added or changed, update the corresponding e2e test to match. If a corresponding e2e test does not exist yet, create it.

## Helper Functions

Extract repeated setup or assertion patterns into utility functions, just as Move uses `#[test_only]` helpers.

```typescript
export async function getObjectFields(objectId: string): Promise<Record<string, any>> {
  const obj = await suiClient.getObject({
    objectId,
    include: { json: true },
  });
  return obj.object.json as Record<string, any>;
}

export function findCreatedObjectByType(result: any, typeSuffix: string) {
  const tx = result.Transaction!;
  const changedObjects = tx.effects?.changedObjects ?? [];
  const entry = changedObjects.find(
    (obj: any) =>
      obj.idOperation === 'Created' &&
      tx.objectTypes?.[obj.objectId]?.includes(typeSuffix),
  );
  return entry ? { objectId: entry.objectId, objectType: tx.objectTypes?.[entry.objectId] } : undefined;
}
```

## Test Expected Failures

To verify that a Move abort is raised, catch the error and match the abort code. Always match the specific error name to avoid false passes — mirroring Move's `#[expected_failure(abort_code = ...)]`.

```typescript
it('rejects swap on empty pool', async () => {
  const tx = new Transaction();
  tx.moveCall({
    target: `${process.env.PACKAGE_ID}::pool::swap`,
    arguments: [/* ... */],
  });

  await expect(
    testUtils.sendTransaction(tx),
  ).rejects.toThrow(/MoveAbort.*EInsufficientLiquidity/);
});
```

## Asserting on Events

```typescript
it('emits a PoolCreated event', async () => {
  const tx = new Transaction();
  testUtils.prepareTransaction(tx, 'pool::create', []);

  const result = await testUtils.sendTransaction(tx);

  const event = result.Transaction!.events?.find((e: any) => e.type.includes('::pool::PoolCreated'));
  expect(event).toBeDefined();
  expect(event!.parsedJson).toMatchObject({ creator: getSigner().toSuiAddress() });
});
```

## Useful Testing Utilities

| Utility | Purpose |
|---------|---------|
| `suiClient.getObject` | Fetch object state to assert on fields |
| `suiClient.getObjects` | Batch-fetch multiple objects |
| `suiClient.signAndExecuteTransaction` | Sign and send a transaction (use `include: { objectTypes: true, effects: true }` to inspect created objects) |
| `tx.moveCall` | Build a Move call in a PTB |
| `tx.pure.string` / `tx.pure.address` / `tx.pure.u64` | Serialize pure arguments |
| `bcs` (from `@mysten/sui/bcs`) | Client-side BCS encoding for derivation and serialization |
| `Ed25519Keypair` | Generate random keypairs for multi-sender tests |

## Running Tests

**CRITICAL: Before running any test command, `cd` into the directory that contains the `.env` file.** `dotenv.config()` loads `.env` from the current working directory. If you run tests from the wrong directory, env vars will be undefined and tests will fail. Do **not** add workarounds like `loadEnv`, `envDir`, `dotenv.config({ path: ... })` with a custom path, or any other env-loading hack. The only correct fix is to `cd` to where `.env` lives.

```bash
cd <directory-containing-.env> && <pm> test
```

Use the detected package manager (see **Package Manager Detection** above) for all commands.

Run all tests:

```bash
cd <directory-containing-.env> && <pm> test
```

Run a specific test file:

```bash
cd <directory-containing-.env> && <pm> vitest run tests/pool.test.ts
```

> Replace `<pm> vitest` with the correct runner: `npx vitest`, `yarn vitest`, or `bunx vitest`.

Watch mode during development:

```bash
cd <directory-containing-.env> && <pm> run test:watch
```

**Always run the tests after writing or updating them.** The task is not complete until all tests pass. If a test fails, investigate the failure and fix it before moving on.
