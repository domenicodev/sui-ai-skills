# Move Conventions

Based on [Sui Move Best Practices](https://docs.sui.io/guides/developer/move-best-practices).

## Package Structure

```
sources/
    my_module.move
tests/
    my_module_tests.move
examples/
    using_my_module.move
Move.lock
Move.toml
```

- Package name in `Move.toml`: PascalCase (`name = "MyPackage"`)
- Named address: snake_case matching package name (`my_package = 0x0`)
- Always commit `Move.lock` (never gitignore it).
- Always commit `Published.toml` if exists (never gitignore it).

## Module Structure

One module per object/data structure. Variant structures get their own module.

No brackets needed in module declarations. The compiler provides default `use` statements for common modules.

```move
module conventions::wallet;

public struct Wallet has key, store {
    id: UID,
    amount: u64
}
```

### Section Order

```move
module conventions::my_module;

// === Imports ===

// === Errors ===

// === Constants ===

// === Structs ===

// === Events ===

// === Method Aliases ===

// === Public Functions ===

// === View Functions ===

// === Admin Functions ===

// === Package Functions ===

// === Private Functions ===

// === Test Functions ===
```

`init` function goes first if it exists. Sort functions by user flow. Test functions in the module should only be `[test_only]` helpers; actual tests go in `tests/`.

### Imports

Group imports by dependency. Import types and functions so you can call them directly -- avoid fully qualified `package::module::function` calls in function bodies.

```move
// good: import what you use
use sui::{
    coin::{Self, Coin},
    balance::Balance,
    table::Table
};

// then call:
let value = coin::value(&my_coin);
let bal: Balance<SUI> = coin::into_balance(my_coin);

// bad: fully qualified calls
let value = sui::coin::value(&my_coin);
```

Import `Self` when you need both the module namespace and its types (e.g., `coin::{Self, Coin}` lets you use both `Coin<T>` and `coin::value`).

## Naming

### Constants

- Regular constants: `UPPER_SNAKE_CASE` (`const MAX_NAME_LENGTH: u64 = 64;`)
- Error constants: `EPascalCase` (`const EInvalidName: u64 = 0;`)

### Structs

Abilities order: `key`, `copy`, `drop`, `store`.

Never use "potato" in struct names. The lack of abilities defines the potato pattern.

Positional fields for simple wrappers, dynamic field keys, or tuples:

```move
public struct ReceiptKey(ID) has copy, drop, store;
```

Event structs use the `Event` suffix.

### CRUD Function Names

| Function | Purpose |
|----------|---------|
| `new` | Create empty object |
| `empty` | Create empty struct |
| `create` | Create initialized object/struct |
| `add` / `remove` | Add/remove a value |
| `exists` | Check if key exists |
| `contains` | Check if collection contains value |
| `borrow` / `borrow_mut` | Immutable/mutable reference |
| `property_name` / `property_name_mut` | Field getter / mutable field getter |
| `drop` | Drop a struct |
| `destroy` | Destroy object with droppable values |
| `destroy_empty` | Destroy empty object with droppable values |
| `to_name` / `from_name` | Type conversions |

### Generics

Single letter (`T`, `U`) or descriptive names (`Data`). Prioritize readability.

## Code Patterns

### Shared Objects

Provide `new` (returns object) + `share` (shares it). This lets callers run logic before sharing.

```move
public fun new(ctx: &mut TxContext): Shop {
    Shop { id: object::new(ctx) }
}

public fun share(shop: Shop) {
    transfer::share_object(shop);
}
```

### Pure Functions

Do not call `transfer::transfer` or `transfer::public_transfer` inside core functions. Return values and let the caller decide.

```move
// correct: returns all coins
public fun add_liquidity<CoinX, CoinY, LpCoin>(
    pool: &mut Pool, coin_x: Coin<CoinX>, coin_y: Coin<CoinY>
): (Coin<LpCoin>, Coin<CoinX>, Coin<CoinY>) { ... }

// wrong: transfers inside the function
public fun impure_add_liquidity<CoinX, CoinY, LpCoin>(
    pool: &mut Pool, coin_x: Coin<CoinX>, coin_y: Coin<CoinY>, ctx: &mut TxContext
): Coin<LpCoin> { ... }
```

### Coin Argument

Pass `Coin` by value with the exact amount. Do not pass `&mut Coin` with a separate `value` parameter.

### Access Control

Use capability objects, not address arrays.

```move
// correct: capability as second parameter, keeps method associativity
// account.set(&cap, b"jose".to_string());
public fun set(account: &mut Account, _: &Admin, new_name: String) { ... }

// wrong: reversed associativity
public fun update(_: &Admin, account: &mut Account, new_name: String) { ... }
```

### Data Storage

- 1-to-1 relationship → owned objects
- Need shared functionality (e.g., cancellation by issuer) → shared objects

## Documentation

- `///` (doc comments) for public-facing explanations of functions and structs
- `//` for technical insights aimed at developers
- Use field comments to describe struct properties
- Complex functions: describe parameters and return values
- Always include a `README.md` in the package root
