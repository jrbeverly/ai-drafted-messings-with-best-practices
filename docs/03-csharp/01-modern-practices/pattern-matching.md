# Pattern Matching

Concise, declarative syntax for type checking, value comparison, deconstruction, and conditional logic (C# 8-11).

## Why It Matters

- Replaces verbose `if`/`is`/`as`/`switch` chains with readable expressions
- Compiler enforces exhaustiveness in switch expressions
- Generates optimized IL with no runtime penalty

## Key Recommendations

**Type patterns** -- check and cast in one step:
```csharp
if (obj is string str) Console.WriteLine(str.Length);
```

**Null checks** -- prefer `is not null` over `!= null`:
```csharp
if (value is not null) { /* safe */ }
```

**Switch expressions** -- return values directly:
```csharp
var discount = type switch
{
    CustomerType.VIP => 0.20m,
    CustomerType.Premium => 0.10m,
    _ => 0m
};
```

**Property patterns** -- match on object properties:
```csharp
var shipping = order switch
{
    { Total: > 100 } => "Free",
    { Total: > 50 }  => "$5",
    _                 => "$10"
};
```

**Relational and logical patterns:**
```csharp
bool valid = score is >= 0 and <= 100;
bool weekend = day is DayOfWeek.Saturday or DayOfWeek.Sunday;
```

**List patterns** (C# 11):
```csharp
bool startsWithOne = numbers is [1, ..];
var desc = arr switch { [] => "empty", [var x] => $"one: {x}", [var f, .., var l] => $"{f}..{l}" };
```

**Positional patterns** with records and tuples:
```csharp
var quadrant = point switch { (> 0, > 0) => "I", (< 0, > 0) => "II", _ => "other" };
```

**Tuple patterns for state machines:**
```csharp
var next = (current, action) switch
{
    (State.Pending, "confirm") => State.Confirmed,
    (_, "cancel")              => State.Cancelled,
    _ => throw new InvalidOperationException()
};
```

## Pitfalls to Avoid

- Missing the `_` discard arm (non-exhaustive switch expressions throw at runtime)
- Overly nested patterns that reduce readability
- Forgetting that `when` guards are not checked for exhaustiveness
