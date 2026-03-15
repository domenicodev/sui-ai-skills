# Move Audit: Type System

Bugs from misusing Move's ability system, phantom types, and UID. When reviewing or writing Move structs and generics, check for these patterns.

---

## 1. Droppable Hot Potato

**Bug:** Adding `drop` to a struct meant to enforce an action (flash loan receipt, promise, ticket).

```move
// BUG: drop lets the caller ignore the receipt entirely
struct FlashLoanReceipt has drop {
    pool_id: ID,
    amount: u64,
}
```

**Why:** The caller can borrow funds, discard the receipt (it gets silently dropped at end of scope), and never repay. The whole enforcement model is gone.

**Fix:** Hot potatoes must have zero abilities. No `key`, no `store`, no `copy`, no `drop`.

```move
struct FlashLoanReceipt {
    pool_id: ID,
    amount: u64,
}
```

---

## 2. copy + drop on Asset Structs

**Bug:** A struct representing a token or asset has `copy` and/or `drop`.

```move
// BUG: can duplicate and destroy tokens freely
struct TokenCoin has copy, drop, store {
    amount: u64,
}
```

**Why:** `copy` allows infinite duplication. `drop` allows silent destruction. Together they break any scarcity guarantee.

**Fix:** Assets must only have `key` + `store`. Use separate snapshot structs for computation.

```move
struct TokenCoin has key, store {
    id: UID,
    balance: Balance,
}

// computation-only, not an asset
struct BalanceSnapshot has copy, drop {
    amount: u64,
}
```

**Rule:** If it represents value, never `copy`, never `drop`.

---

## 3. Missing Phantom Type on Receipts

**Bug:** A receipt or proof struct doesn't encode the generic type it was created with.

```move
// BUG: receipt doesn't track which CoinType was used
struct PaymentReceipt {
    amount: u64,
}
```

**Why:** An attacker creates a worthless custom coin, calls `purchase<ScamCoin>`, gets a valid receipt, then claims a real item. The receipt looks identical regardless of coin type.

**Fix:** Use `phantom` type parameters to bind the receipt to the coin type.

```move
struct PaymentReceipt<phantom CoinType> {
    amount: u64,
}

public fun purchase<CoinType>(
    payment: Coin<CoinType>
): PaymentReceipt<CoinType> { ... }

public fun claim_item<CoinType>(
    receipt: PaymentReceipt<CoinType>,
    item: Item
) { ... }
```

Now `PaymentReceipt<SUI>` and `PaymentReceipt<ScamCoin>` are different types.

---

## 4. UID with drop Ability

**Bug:** Declaring an object struct with `drop`.

```move
// BUG: won't compile -- UID doesn't have drop
struct GameItem has key, drop, store {
    id: UID,
    power: u64,
}
```

**Why:** `UID` intentionally lacks `drop` to maintain object graph integrity. Objects cannot be silently discarded.

**Fix:** Remove `drop`. Provide an explicit deletion function.

```move
struct GameItem has key, store {
    id: UID,
    power: u64,
}

public fun delete(item: GameItem) {
    let GameItem { id, power: _ } = item;
    object::delete(id);
}
```

For temporary computation structs (no `id: UID`), `drop` is fine.

---

## 5. Hot Potato Wrapped in Option

**Bug:** Wrapping a non-droppable type in `Option` and handling it conditionally.

```move
// BUG: compiler error -- maybe_receipt must be consumed on all paths
public fun risky(maybe_receipt: Option<FlashLoanReceipt>) {
    if (option::is_some(&maybe_receipt)) {
        let receipt = option::extract(&mut maybe_receipt);
        process(receipt);
    }
    // maybe_receipt is still alive here
}
```

**Why:** Even an empty `Option` holding a non-droppable type must be explicitly destroyed. The compiler doesn't track that the Option is None after extraction.

**Fix:** Call `option::destroy_none` on all paths. Better yet, avoid wrapping hot potatoes in Option -- design deterministic APIs instead.

```move
public fun safe(maybe_receipt: Option<FlashLoanReceipt>) {
    if (option::is_some(&maybe_receipt)) {
        let receipt = option::extract(&mut maybe_receipt);
        process(receipt);
    };
    option::destroy_none(maybe_receipt);
}
```

---

## 6. Generic Phantom Role Escalation

**Bug:** A role-gated function accepts a generic `RoleCap<R>` instead of a specific role.

```move
public struct RoleCap<phantom R> has key, store { id: UID }

struct UserRole has drop {}
struct AdminRole has drop {}

// BUG: accepts ANY role, not just ModRole
public fun checkout_admin<R>(
    _role: &RoleCap<R>,
    ctx: &mut TxContext
): RoleCap<AdminRole> {
    RoleCap<AdminRole> { id: object::new(ctx) }
}
```

**Why:** The generic `R` is a wildcard -- a user holding `RoleCap<UserRole>` can call this function and receive an `AdminCap`. The type system accepts any `RoleCap` regardless of the phantom type.

**Fix:** Constrain the function to the specific role it requires.

```move
public fun checkout_admin(
    _role: &RoleCap<ModRole>,
    ctx: &mut TxContext
): RoleCap<AdminRole> {
    RoleCap<AdminRole> { id: object::new(ctx) }
}
```

---

*References:
[The Ability Mistakes That Will Drain Your Sui Move Protocol](https://medium.com/@pushkarm029/the-ability-mistakes-that-will-drain-your-sui-move-protocol-1a2c317e373f)
[Sui Move Security Workshop](https://medium.com/@monethic/sui-move-security-workshop-writeup-material-480c5e7d1da3)*
