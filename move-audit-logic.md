# Move Audit: Access Control, Data & Limits

Bugs from incorrect access control, unsafe data handling, and protocol limit violations. When reviewing or writing Move functions and storage, check for these patterns.

---

## 1. Shared AdminCap

**Bug:** Sharing an admin capability with `transfer::public_share_object`.

```move
// BUG: anyone can use this AdminCap
fun init(ctx: &mut TxContext) {
    let cap = AdminCap { id: object::new(ctx) };
    transfer::public_share_object(cap); // <-- everyone is admin
}
```

**Why:** Shared objects are globally accessible. Any transaction can pass the AdminCap as an argument. All admin-gated functions become publicly callable.

**Fix:** Transfer the capability to the deployer.

```move
fun init(ctx: &mut TxContext) {
    let cap = AdminCap { id: object::new(ctx) };
    transfer::transfer(cap, ctx.sender());
}
```

---

## 2. Unchecked Publisher for Access Control

**Bug:** Accepting a `Publisher` reference as an access control gate without verifying it belongs to your package or module.

```move
// BUG: any Publisher from any package passes this check
public fun admin_mint(pub: &Publisher, ctx: &mut TxContext): MyNFT {
    MyNFT { id: object::new(ctx) }
}
```

**Why:** `Publisher` is a standard Sui object anyone can create for their own package via `sui::package::claim`. It proves authorship of *some* package, not *your* package. Without verification, an attacker uses their own Publisher to call your admin function.

**Fix:** Always verify with `package::from_module<T>` or `package::from_package<T>`.

```move
public fun admin_mint(pub: &Publisher, ctx: &mut TxContext): MyNFT {
    assert!(package::from_module<MyNFT>(pub), ENotAuthorized);
    MyNFT { id: object::new(ctx) }
}
```

`from_module<T>` checks that the Publisher was claimed from the module defining `T`. `from_package<T>` checks at the package level. Use `from_module` for tighter scoping.

---

## 3. public(package) entry Confusion

**Bug:** Assuming `public(package) entry` restricts who can call a function.

```move
// BUG: anyone can call this as a transaction entry point
public(package) entry fun emergency_withdraw(vault: &mut Vault, ctx: &mut TxContext) {
    let balance = balance::value(&vault.balance);
    let coin = coin::take(&mut vault.balance, balance, ctx);
    transfer::public_transfer(coin, tx_context::sender(ctx));
}
```

**Why:** The `entry` modifier makes the function directly callable as a transaction entry point by anyone, regardless of `public(package)` visibility. `public(package)` only restricts which *modules* can call it in Move code -- it does not restrict transaction-level access.

**Fix:** Never combine `entry` with `public(package)` for privileged operations. Use a capability object to gate access.

```move
public entry fun emergency_withdraw(vault: &mut Vault, _: &AdminCap, ctx: &mut TxContext) {
    let balance = balance::value(&vault.balance);
    let coin = coin::take(&mut vault.balance, balance, ctx);
    transfer::public_transfer(coin, ctx.sender());
}
```

---

## 4. Untrusted Caller Address Parameter

**Bug:** Accepting an `address` parameter for authorization instead of using `tx_context::sender`.

```move
// BUG: caller parameter is attacker-controlled
public entry fun withdraw_all(
    vault: &mut Vault,
    caller: address, // <-- attacker can pass vault.admin here
    ctx: &mut TxContext
) {
    assert!(caller == vault.admin, ENotAdmin);
    // withdraws to tx_context::sender, not to caller
    let coin = coin::take(&mut vault.balance, balance::value(&vault.balance), ctx);
    transfer::public_transfer(coin, ctx.sender());
}
```

**Why:** The attacker passes the admin's address as `caller`, the assert passes, but `ctx.sender()` is the attacker. The function performs privileged operations on behalf of someone who isn't the actual sender.

**Fix:** Always validate against the actual transaction sender.

```move
public entry fun withdraw_all(vault: &mut Vault, ctx: &mut TxContext) {
    assert!(ctx.sender() == vault.admin, ENotAdmin);
    let coin = coin::take(&mut vault.balance, balance::value(&vault.balance), ctx);
    transfer::public_transfer(coin, ctx.sender());
}
```

---

## 5. Unbounded Vector Growth

**Bug:** A public function pushes into a vector without checking its length against a maximum.

```move
// BUG: anyone can call this until the object exceeds 256KB and becomes unusable
public fun add_member(group: &mut Group, member: address) {
    vector::push_back(&mut group.members, member);
}
```

**Why:** Sui objects have a 256KB (262,144 bytes) hard limit. If a vector grows unchecked, the object can hit this limit, causing all future transactions that touch it to abort. For shared objects this is a permanent denial-of-service.

**Fix:** Always enforce a max length before pushing. Choose the limit based on the element type, leaving headroom (~200KB of vector data) for the object's other fields.

```move
const MAX_MEMBERS: u64 = 6_250; // address = 32 bytes

public fun add_member(group: &mut Group, member: address) {
    assert!(vector::length(&group.members) < MAX_MEMBERS, ETooManyMembers);
    vector::push_back(&mut group.members, member);
}
```

**Safe max lengths per type** (targeting ~200KB of vector data):

| Element type | Size (bytes) | Safe max items |
|-------------|-------------|----------------|
| `bool` / `u8` | 1 | 200,000 |
| `u16` | 2 | 100,000 |
| `u32` | 4 | 50,000 |
| `u64` | 8 | 25,000 |
| `u128` | 16 | 12,500 |
| `u256` / `address` | 32 | 6,250 |
| Custom structs | varies | 1,000 (conservative default) |

For any third-party-writable vector (shared objects, public push functions), this check is mandatory. For owned objects with controlled access, it's still recommended.

Consider using `Table`, `Bag`, `ObjectTable`, or `LinkedTable` for collections that grow beyond ~1,000 items -- they store entries as dynamic fields and are not subject to the 256KB object limit.

---

## 6. Table Duplicate Key Abort

**Bug:** Using `table::add` without checking if the key already exists.

```move
// BUG: aborts on second deposit -- user permanently locked out
public entry fun deposit(bank: &mut Bank, coin: Coin<SUI>, ctx: &mut TxContext) {
    let sender = ctx.sender();
    let amount = coin::value(&coin);
    table::add(&mut bank.balances, sender, amount); // aborts if key exists
    coin::put(&mut bank.reserves, coin);
}
```

**Why:** `table::add` aborts if the key is already present. After a user's first deposit, all subsequent deposits fail. This is a permanent denial-of-service for that user.

**Fix:** Check existence first, then add or update.

```move
public entry fun deposit(bank: &mut Bank, coin: Coin<SUI>, ctx: &mut TxContext) {
    let sender = ctx.sender();
    let amount = coin::value(&coin);
    if (table::contains(&bank.balances, sender)) {
        let existing = table::borrow_mut(&mut bank.balances, sender);
        *existing = *existing + amount;
    } else {
        table::add(&mut bank.balances, sender, amount);
    };
    coin::put(&mut bank.reserves, coin);
}
```

Applies to any key-value collection (`Table`, `ObjectTable`, `Bag`, etc.) -- always check before `add`.

---

## 7. Unbounded Iteration on User-Controlled Data

**Bug:** Iterating over an entire collection whose size is controlled by users.

```move
// BUG: gas exhaustion if scores grows large
public fun process_all_scores(board: &mut Leaderboard) {
    let len = vector::length(&board.scores);
    let mut i = 0;
    while (i < len) {
        let score = vector::borrow(&board.scores, i);
        // expensive processing per entry
        i = i + 1;
    };
}
```

**Why:** An attacker fills the collection with thousands of entries. The function becomes uncallable because it exceeds gas limits. Unlike #5 (object size limit), this can happen well before 256KB if processing per entry is expensive.

**Fix:** Process in bounded batches with a start/end index, or use pagination.

```move
public fun process_scores(board: &mut Leaderboard, start: u64, count: u64) {
    let len = vector::length(&board.scores);
    let end = math::min(start + count, len);
    let mut i = start;
    while (i < end) {
        let score = vector::borrow(&board.scores, i);
        // processing
        i = i + 1;
    };
}
```

Also ensure there is a mechanism to remove stale entries so collections don't grow indefinitely.

---

## 8. Clock Timestamp Unit Mismatch

**Bug:** Mixing seconds and milliseconds when using `clock::timestamp_ms`.

```move
const LOCK_TIME_SECONDS: u64 = 10 * 24 * 60 * 60; // 10 days in seconds

public entry fun stake(vault: &mut Vault, clock: &Clock, ctx: &mut TxContext) {
    let position = StakePosition {
        id: object::new(ctx),
        timestamp: clock::timestamp_ms(clock), // stores MILLISECONDS
        amount: 100,
    };
    // ...
}

public entry fun unstake(position: &StakePosition, clock: &Clock) {
    let now_seconds = clock::timestamp_ms(clock) / 1000;
    // BUG: comparing seconds against milliseconds
    assert!(now_seconds >= position.timestamp + LOCK_TIME_SECONDS, ETooEarly);
}
```

**Why:** `position.timestamp` is in milliseconds, `now_seconds` and `LOCK_TIME_SECONDS` are in seconds. The comparison is off by a factor of 1,000. A 10-day lock unlocks almost instantly.

**Fix:** Pick one unit and use it consistently. Since `clock::timestamp_ms` returns milliseconds, work in milliseconds throughout.

```move
const LOCK_TIME_MS: u64 = 10 * 24 * 60 * 60 * 1000; // 10 days in ms

public entry fun unstake(position: &StakePosition, clock: &Clock) {
    let now = clock::timestamp_ms(clock);
    assert!(now >= position.timestamp + LOCK_TIME_MS, ETooEarly);
}
```

**Rule:** Sui's clock always returns milliseconds. Name your fields and constants with `_ms` suffix to make the unit explicit.

---

## 9. Protocol Limits

Sui enforces hard limits at the protocol level. Exceeding any of these causes the transaction to be rejected or aborted. Design around them proactively.

| Limit | Value |
|-------|-------|
| Transaction size | 128 KB |
| Object size | 256 KB |
| Single pure argument | 16 KB |
| Objects created per transaction | 2,048 |
| Dynamic fields created per transaction | 1,000 |
| Dynamic fields accessed per transaction | 1,000 |
| Events emitted per transaction | 1,024 |

### Practical implications

**Pure argument cap (16KB):** A vector of `address` (32 bytes each) passed as a transaction argument can hold ~500 entries max. To go beyond, split into multiple arguments and join with `vector::append()` in Move or in the Transaction Block.

**Object creation cap (2,048):** Batch minting or bulk operations that create objects must stay under this. Dynamic fields count toward this limit too (both key and value are objects), so the effective cap for dynamic field creation is ~1,000 per transaction.

**Dynamic field access cap (1,000):** Iterations over large `Table`/`Bag` collections in a single transaction can hit this. Design operations to touch a bounded number of fields per call.

**Event cap (1,024):** Loops that emit an event per iteration (e.g., per-user notifications) must be bounded or batched.

Source: [Building Against Limits](https://move-book.com/guides/building-against-limits/)

---

## 10. Unaudited Dependencies

**Context:** The $223M Cetus exploit (May 2025) passed three professional audits. The bug was in `integer-mate`, a widely-used math library -- not in Cetus's own code.

**Lesson:** Move's type system prevents reentrancy and asset mishandling, but not logic bugs (overflows, off-by-one, incorrect math) in dependencies. Audit imported libraries with the same rigor as your own code.

---

*References:
[The Ability Mistakes That Will Drain Your Sui Move Protocol](https://medium.com/@pushkarm029/the-ability-mistakes-that-will-drain-your-sui-move-protocol-1a2c317e373f)
[Building Against Limits](https://move-book.com/guides/building-against-limits/)
[Sui Move Security Workshop](https://medium.com/@monethic/sui-move-security-workshop-writeup-material-480c5e7d1da3)*
