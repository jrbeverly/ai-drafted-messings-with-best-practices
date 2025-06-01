# Collections Performance

Collection type selection guide and performance characteristics for C#.

## Why It Matters

- Wrong collection type causes O(n) lookups instead of O(1)
- Pre-sizing avoids repeated resize allocations
- Frozen/immutable collections optimize read-heavy and thread-safe scenarios

## Collection Selection Guide

| Need | Use | Avoid |
|------|-----|-------|
| Key-value lookup | `Dictionary<K,V>` | `List<T>` + linear search |
| Membership test | `HashSet<T>` | `List<T>.Contains` (O(n)) |
| Static lookup table | `FrozenDictionary` / `FrozenSet` (.NET 8) | Dictionary rebuilt per request |
| Thread-safe map | `ConcurrentDictionary` | Dictionary + manual lock |
| Producer/consumer | `Channel<T>` | `ConcurrentQueue` |
| Read-only, rarely mutated | `ImmutableArray<T>` | `ImmutableList<T>` (slower) |
| Sorted unique | `SortedSet<T>` | Sort after insert |

## Key Recommendations

**Pre-size when count is known:**
```csharp
var users = new List<User>(dtos.Count);
var lookup = new Dictionary<string, User>(expectedCount);
```

**FrozenDictionary for config/lookup tables built at startup:**
```csharp
FrozenDictionary<string, int> codes = dict.ToFrozenDictionary();
// ~35% faster reads than Dictionary, higher creation cost
```

**ArrayPool for temporary buffers in hot paths:**
```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(size);
try { /* use buf */ }
finally { ArrayPool<byte>.Shared.Return(buf, clearArray: true); }
```

**ReadOnlySpan for zero-allocation slicing:**
```csharp
ReadOnlySpan<int> slice = data.AsSpan(2..5); // no copy
```

## Pitfalls to Avoid

- `List<T>.Contains` for membership checks (use `HashSet<T>`)
- Creating `FrozenDictionary` in hot loops (creation is expensive)
- Using `ImmutableList` when `ImmutableArray` suffices
- Forgetting to return rented arrays to `ArrayPool`
