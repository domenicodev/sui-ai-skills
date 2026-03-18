---
name: sui
description: >-
  Develop on the Sui blockchain using Move smart contracts and/or the TypeScript SDK.
  Use when working with Move.toml files, Sui Move modules, @mysten/sui dependencies,
  or any Sui-related development task.
---

# Sui Development

## Project Detection

Before starting any task, detect the project type by scanning the workspace:

### Move Project

Look for a `Move.toml` file. The directory containing it is the Move project root.

### TypeScript SDK Project (backend / service)

Look for a top-level `package.json` (not inside `node_modules`) that lists `@mysten/sui` as a dependency or devDependency, **without** dAppKit indicators.

### React dAppKit Project (frontend)

A TypeScript project is a dAppKit project if any of these are true:
- `package.json` lists `@mysten/dapp-kit`, `@mysten/dapp-kit-react`, or `@mysten/dapp-kit-core` as a dependency
- The project contains `main.tsx` and `vite.config`(with js or ts-like extensions) files

A dAppKit project will always also have `@mysten/sui`. Apply both SDK and dAppKit guides.

### Move + TypeScript Project (e2e / integration)

A workspace is a combined Move + TypeScript project when **both** of these are true:
- A `Move.toml` file exists (Move project root)
- A `package.json` lists `@mysten/sui` as a dependency or devDependency

**And none** of the dAppKit indicators are present (no `@mysten/dapp-kit`, `@mysten/dapp-kit-react`, `@mysten/dapp-kit-core` in `package.json`, no `dApp-kit.ts` files).

This is the typical setup for projects that deploy Move contracts and test or interact with them via TypeScript (e2e tests, scripts, CLI tools). Detect each part independently and apply **both** the Move and the combined-project guides for the task at hand.

## Reference Guides

### For a Move project

| Guide | When to read |
|-------|-------------|
| [move-conventions.md](move-conventions.md) | Writing or reviewing Move code |
| [move-audit-types.md](move-audit-types.md) | Reviewing structs, abilities, phantom types for vulnerabilities |
| [move-audit-logic.md](move-audit-logic.md) | Reviewing access control, data handling, and protocol limits |
| [move-tests.md](move-tests.md) | Writing or running Move tests |

### For a TypeScript SDK project

| Guide | When to read |
|-------|-------------|
| [sdk-use-grpc.md](sdk-use-grpc.md) | Writing SDK code or migrating from deprecated jsonRPC to gRPC |
| [sdk-reads.md](sdk-reads.md) | Reviewing SDK read patterns (batching, object references) |
| [sdk-transactions.md](sdk-transactions.md) | Reviewing SDK transaction patterns (gas budget, PTB merging) |

### For a Move + TypeScript project

All Move guides and SDK guides apply. In addition:

| Guide | When to read |
|-------|-------------|
| [move-conventions.md](move-conventions.md) | Writing or reviewing Move code |
| [move-audit-types.md](move-audit-types.md) | Reviewing structs, abilities, phantom types for vulnerabilities |
| [move-audit-logic.md](move-audit-logic.md) | Reviewing access control, data handling, and protocol limits |
| [move-tests.md](move-tests.md) | Writing or running Move unit tests |
| [e2e-tests.md](e2e-tests.md) | Writing or running TypeScript e2e tests against Move contracts |
| [sdk-use-grpc.md](sdk-use-grpc.md) | Writing SDK code or migrating from deprecated jsonRPC to gRPC |
| [sdk-reads.md](sdk-reads.md) | Reviewing SDK read patterns (batching, object references) |
| [sdk-transactions.md](sdk-transactions.md) | Reviewing SDK transaction patterns (gas budget, PTB merging) |

### For a React dAppKit project

| Guide | When to read |
|-------|-------------|
| [react-use-grpc.md](react-use-grpc.md) | Writing dAppKit code or migrating from deprecated `@mysten/dapp-kit` to v2 |

Read only the guides relevant to the current task. Do not read all of them upfront.

## Workflow

Before writing or modifying any file, switch to **Plan mode**. For each change, list:
- The file path
- The existing code (or "new file" if creating)
- The proposed code
- Why the change is needed

Wait for explicit approval before proceeding. Do not write code until the plan is approved.

### Move-only project

1. **Audit and analyze** — review Move code using the Move guides listed in the Reference Guides section above.
2. **Move tests** — write or update Move unit tests following `move-tests.md`. Target good coverage.
3. **No e2e tests** — do not create TypeScript e2e tests if the project has no TypeScript code.

### TypeScript-only project

1. **SDK code** — follow the SDK guides listed in the Reference Guides section above.
2. **No Move tests** — do not write Move unit tests if the project has no Move code.

### Move + TypeScript project

1. **Audit and analyze** — review Move code using the Move guides listed in the Reference Guides section above.
2. **Move tests** — write or update Move unit tests following `move-tests.md`. Target good coverage. Complete this step before moving on.
3. **E2E tests** — once Move tests are ready, create or update TypeScript e2e tests following `e2e-tests.md`.

**Test sync rule**: Move unit tests and TypeScript e2e tests must stay in sync:
- When a Move test is **added**, create a matching e2e test.
- When a Move test is **changed**, update the corresponding e2e test to match.
- If a corresponding e2e test does not exist yet, create it.
