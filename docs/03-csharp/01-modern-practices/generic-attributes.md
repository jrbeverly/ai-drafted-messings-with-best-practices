# Generic Attributes

C# 11 feature enabling type parameters on attribute classes, replacing `typeof()` with compile-time type safety.

## Why It Matters

- Compile-time type checking instead of runtime `typeof()` validation
- Better IntelliSense, refactoring safety, and cleaner syntax
- Natural fit for validator, factory, and converter attribute patterns

## Key Recommendations

**Replace `typeof()` attribute parameters with generic type arguments:**
```csharp
// Before: runtime type, no compile-time check
[Validator(typeof(EmailValidator))]
public string Email { get; set; }

// After: compile-time type safety
[Validator<EmailValidator>]
public string Email { get; set; }
```

**Define generic attributes with constraints:**
```csharp
[AttributeUsage(AttributeTargets.Property)]
public class ValidateWithAttribute<T> : Attribute where T : IValidator, new() { }
```

**Use for validator, factory, serializer, and DI attributes:**
```csharp
[ValidateWith<EmailValidator>]
public string Email { get; set; }

[Factory<UserFactory>]
public class User { }
```

**Access via reflection** using `GetCustomAttributes(typeof(ValidateWithAttribute<>), false)` and inspecting generic arguments.

## Pitfalls to Avoid

- Overusing for simple cases where `typeof()` is already clear
- Complex generic hierarchies on attributes (keep constraints simple)
- Forgetting `AttributeUsage` to restrict valid targets
