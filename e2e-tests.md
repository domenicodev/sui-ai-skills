# E2E Tests (TypeScript)

E2E tests exercise Move contracts through the TypeScript SDK on a live network. They mirror the Move unit tests 1-to-1 — same functionalities, same flow — while also covering the integration boundary: PTB composition, event parsing, client-side serialization, and cross-package interactions after publishing.

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

## Dependencies

```bash
npm install -D vitest @mysten/sui dotenv
```

`dotenv` is **required**. `zod` is optional but recommended for env validation:

```bash
npm install -D zod
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

Every test file must call `dotenv.config()` inside `beforeAll`. After loading, **assert that all required env vars are present** — never assume they exist.

Prefer `zod` + `TestUtils` when there is much to test (many operations, shared state, repeated transaction patterns). For simple test files with a handful of self-contained checks, manual assertions without a utils class are fine.

- **Simple** (few tests, no shared state): call `dotenv.config()` and assert env vars directly in `beforeAll`.
- **Complex** (many tests, shared setup): call `dotenv.config()` in `beforeAll`, then instantiate `TestUtils` — the constructor validates env vars with `zod`.

**Manual assertion** (simple tests):

```typescript
beforeAll(() => {
  dotenv.config();
  if (!process.env.NETWORK) throw new Error('NETWORK is not set in .env');
  if (!process.env.USER_SECRET_KEY) throw new Error('USER_SECRET_KEY is not set in .env');
  if (!process.env.PACKAGE_ID) throw new Error('PACKAGE_ID is not set in .env');
});
```

**With `zod`** (preferred for complex tests):

```typescript
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

These helpers read from `process.env` directly, so they only work after `dotenv.config()` has been called.

`src/helpers/suiClient.ts`:

```typescript
import { SuiClient, getJsonRpcFullnodeUrl } from '@mysten/sui/client';

const network = process.env.NETWORK as 'testnet' | 'mainnet' | 'devnet' | 'localnet';
export const suiClient = new SuiClient({ url: getJsonRpcFullnodeUrl(network) });
```

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
import dotenv from 'dotenv';
import { describe, it, expect, beforeAll } from 'vitest';
import { suiClient } from '../src/helpers/suiClient';

beforeAll(() => {
  dotenv.config();
  if (!process.env.POOL_ID) throw new Error('POOL_ID is not set in .env');
});

describe('pool reads', () => {
  it('can fetch the pool object', async () => {
    const obj = await suiClient.getObject({
      id: process.env.POOL_ID!,
      options: { showContent: true },
    });
    expect(obj.data).toBeDefined();
  });
});
```

## TestUtils Pattern

When a test file covers a module with many related operations, create a `TestUtils` class that mirrors the `#[test_only]` helpers in Move. The constructor asserts env vars and initializes the signer — do not read env vars at module scope (top-level `const`), since `dotenv.config()` has not run yet at import time.

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
    return suiClient.signAndExecuteTransaction({
      signer: this.signer,
      transaction: tx,
      options: { showObjectChanges: true, showEvents: true },
    });
  }

  findCreatedObject(result: any, typeSuffix: string) {
    return result.objectChanges?.find(
      (c: any) => c.type === 'created' && c.objectType.includes(typeSuffix),
    );
  }
}
```

`tests/pool.test.ts`:

```typescript
import dotenv from 'dotenv';
import { describe, it, expect, beforeAll } from 'vitest';
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
    id: created!.objectId,
    options: { showContent: true },
  });
  const fields = obj.data!.content!.fields as Record<string, any>;
  expect(fields.liquidity).toBe('0');
});
```

Both test the same logic. The Move test uses `tx_context::dummy()` + `destroy`. The TypeScript test uses the SDK + real transactions.

**Keep them in sync**: whenever a Move test is added or changed, update the corresponding e2e test to match. If a corresponding e2e test does not exist yet, create it.

## Helper Functions

Extract repeated setup or assertion patterns into utility functions, just as Move uses `#[test_only]` helpers.

```typescript
export async function getObjectFields(objectId: string): Promise<Record<string, any>> {
  const obj = await suiClient.getObject({
    id: objectId,
    options: { showContent: true },
  });
  return obj.data!.content!.fields as Record<string, any>;
}

export function findCreatedObjectByType(result: any, typeSuffix: string) {
  return result.objectChanges?.find(
    (c: any) => c.type === 'created' && c.objectType.includes(typeSuffix),
  );
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

  const event = result.events?.find((e: any) => e.type.includes('::pool::PoolCreated'));
  expect(event).toBeDefined();
  expect(event!.parsedJson).toMatchObject({ creator: getSigner().toSuiAddress() });
});
```

## Useful Testing Utilities

| Utility | Purpose |
|---------|---------|
| `suiClient.getObject` | Fetch object state to assert on fields |
| `suiClient.getObjects` | Batch-fetch multiple objects |
| `suiClient.queryEvents` | Query events by type for post-tx assertions |
| `suiClient.signAndExecuteTransaction` | Sign and send a transaction |
| `tx.moveCall` | Build a Move call in a PTB |
| `tx.pure.string` / `tx.pure.address` / `tx.pure.u64` | Serialize pure arguments |
| `bcs` (from `@mysten/sui/bcs`) | Client-side BCS encoding for derivation and serialization |
| `Ed25519Keypair` | Generate random keypairs for multi-sender tests |

## Running Tests

Run all tests:

```bash
npm test
```

Run a specific test file:

```bash
npx vitest run tests/pool.test.ts
```

Watch mode during development:

```bash
npm run test:watch
```
