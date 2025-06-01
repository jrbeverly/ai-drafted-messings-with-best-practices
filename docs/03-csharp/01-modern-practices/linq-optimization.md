# LINQ Optimization

Performance considerations for LINQ: deferred execution, allocation awareness, and when to use alternatives on hot paths.

## Why It Matters

- Deferred execution can cause multiple enumerations (re-executing the entire chain)
- Every LINQ operator allocates an iterator; closures allocate additional objects
- `IQueryable` pushes work to the database; `IEnumerable` evaluates in memory

## Key Recommendations

**Materialize before enumerating multiple times:**
```csharp
// BAD: enumerates twice
if (query.Any()) return query.First();

// GOOD: single enumeration
var result = query.FirstOrDefault();
```

**Keep IQueryable as long as possible** before materializing:
```csharp
// BAD: fetches all rows, filters in memory
var users = dbContext.Users.ToList().Where(u => u.Age > 18);

// GOOD: filters in database
var users = await dbContext.Users.Where(u => u.Age > 18).ToListAsync();
```

**Filter before sorting** to reduce downstream work:
```csharp
items.Where(x => x.IsActive).OrderBy(x => x.Name).Take(10).ToList();
```

**Choose the right materialization:**

| Method | Use When |
|--------|----------|
| `ToList()` | Need to add/remove later |
| `ToArray()` | Fixed size, cache-friendly |
| `ToHashSet()` | O(1) membership checks |
| `ToDictionary()` | O(1) keyed access |
| `ToFrozenSet()` | Read-heavy lookup (.NET 8) |

**Replace LINQ with loops/Span only in profiled hot paths** -- LINQ is fine for 99% of code.

**Use `ToLookup` instead of repeated `GroupBy`** when accessing groups multiple times.

## Pitfalls to Avoid

- Enumerating `IEnumerable` multiple times without materializing (CA1851 warning)
- Calling `ToList()` on `IQueryable` before applying filters
- Premature LINQ optimization in code that runs once per request
- Chaining `OrderBy` before `Where` (sorts all items, then filters)
