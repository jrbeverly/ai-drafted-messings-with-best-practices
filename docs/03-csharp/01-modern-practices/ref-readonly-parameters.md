# Ref Readonly Parameters

C# 12 parameter modifier for passing large value types by reference without allowing modification and without defensive copies.

## Why It Matters

- Avoids copying large structs (>16 bytes) on every method call
- Clearer semantics than `in` at both call site and definition
- Combined with `readonly struct`, eliminates all hidden defensive copies

## Key Recommendations

**Use `ref readonly` for large structs in performance-sensitive APIs:**
```csharp
public static double Determinant(ref readonly Matrix4x4 matrix) => /* no 128-byte copy */;
double det = Determinant(in transform);  // 'in' or 'ref' at call site
```

**Declare structs as `readonly`** to eliminate defensive copies entirely:
```csharp
public readonly struct Vector3 { public readonly float X, Y, Z; }
// ref readonly + readonly struct = zero hidden copies guaranteed
```

**Mark individual methods `readonly`** on non-readonly structs to avoid copies per call:
```csharp
public struct Vector3
{
    public float X, Y, Z;
    public readonly float Length() => MathF.Sqrt(X * X + Y * Y + Z * Z);
}
```

## Parameter Passing Modes Comparison

| Modifier | Copies? | Mutable? | Call Site |
|----------|---------|----------|-----------|
| (none) | Yes | Local copy | `Method(val)` |
| `ref` | No | Original | `Method(ref val)` |
| `out` | No | Must assign | `Method(out val)` |
| `in` | No* | No | `Method(in val)` or `Method(val)` |
| `ref readonly` | No* | No | `Method(ref val)` or `Method(in val)` |

*Defensive copy if struct is not `readonly`.

**Rule of thumb:** structs <= 16 bytes pass by value; larger structs use `ref readonly`.

## Pitfalls to Avoid

- Using `ref readonly` for small types (`int`, `double`, `Guid`) where copying is cheaper than indirection
- Forgetting `readonly` on immutable structs (causes hidden defensive copies with `in` and `ref readonly`)
- Using `ref` when you do not intend to modify the value (use `ref readonly`)
- Mixing `in` and `ref readonly` inconsistently in the same API
