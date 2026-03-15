# SDK: Read Optimization

When reviewing or writing TypeScript SDK code that reads on-chain data, detect and flag these patterns.

---

## 1. Batch Object Reads

### Anti-pattern: Sequential `getObject` calls

Detect loops or repeated calls to `getObject` / `client.getObject` with different IDs:

```typescript
// BAD: N separate API calls
const obj1 = await client.getObject({ id: id1, include: { content: true } });
const obj2 = await client.getObject({ id: id2, include: { content: true } });
const obj3 = await client.getObject({ id: id3, include: { content: true } });

// Also BAD: getObject inside a loop
const results = [];
for (const id of objectIds) {
    const obj = await client.getObject({ id, include: { content: true } });
    results.push(obj);
}
```

### Fix: Use `getObjects` (multi-get)

```typescript
const objects = await client.getObjects({
    objectIds: [id1, id2, id3],
    include: { content: true },
});
```

One API call instead of N. The `include` parameters are the same as `getObject`.

**Detection triggers:**
- Multiple `getObject` calls in the same function/scope
- `getObject` inside `for`, `while`, `forEach`, or `.map`
- `Promise.all` wrapping multiple `getObject` calls

All of these should be refactored to a single `getObjects` call.

---

## 2. Prefer Full Object References Over `tx.object(string)`

**Suggestion:** When you see `tx.object(objectId)` with a plain string, suggest using the full object reference instead. This avoids an extra fullnode call to resolve `version` and `digest`.

```typescript
// common usage -- works, but triggers a fullnode resolve
tx.object(objectId);
```

Suggest the user provides the full reference if they have the data available:

```typescript
import { Inputs } from '@mysten/sui/transactions';

// For owned or immutable objects:
tx.object(Inputs.ObjectRef({ digest, objectId, version }));

// For shared objects:
tx.object(Inputs.SharedObjectRef({ objectId, initialSharedVersion, mutable }));
```

When suggesting, ask the user if they can provide:
- `digest` and `version` (for owned/immutable objects)
- `initialSharedVersion` and `mutable` (for shared objects)

If they don't have the data, they can query it via the SDK. This is especially worthwhile when multiple `tx.object()` calls are in the same transaction -- a single `getObjects` call resolves all of them at once, rather than the SDK making N separate resolve calls behind the scenes:

```typescript
const objects = await client.getObjects({
    objectIds: [id1, id2, id3],
    include: { owner: true },
});

const tx = new Transaction();
for (const obj of objects) {
    // owned/immutable: use ObjectRef
    tx.object(Inputs.ObjectRef({
        objectId: obj.objectId,
        version: obj.version,
        digest: obj.digest,
    }));
    // shared: use SharedObjectRef with initialSharedVersion from owner data
}
```

For a single `tx.object()`, the benefit is marginal. For multiple objects in one transaction, this saves real round trips.

---

## Detection Checklist

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Repeated `getObject` | Multiple calls or loops with `getObject` | Replace with single `getObjects` |
| `tx.object(string)` | String IDs passed when ref data is available | Suggest `Inputs.ObjectRef` or `Inputs.SharedObjectRef` |
