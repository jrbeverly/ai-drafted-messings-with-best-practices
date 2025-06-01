# File-Scoped Namespaces

C# 10 feature that removes one indentation level by declaring a single namespace per file with a semicolon.

## Why It Matters

- Saves one indentation level in every file
- Reduces horizontal noise, more vertical space for code
- Modern C# convention (C# 10+)

## Key Recommendations

**Use semicolon syntax instead of braces:**
```csharp
// Before: extra indentation
namespace MyApp.Services
{
    public class UserService { }
}

// After: file-scoped
namespace MyApp.Services;

public class UserService { }
```

**One namespace per file** (enforced by the compiler -- cannot declare a second namespace).

**Enforce via EditorConfig:**
```ini
[*.cs]
csharp_style_namespace_declarations = file_scoped:warning
```

**Use for all new files.** Migrate existing files incrementally or with IDE refactoring tools.

## Pitfalls to Avoid

- Mixing file-scoped and block-scoped in the same file (compiler error)
- Multiple namespaces per file (not allowed with file-scoped syntax; split into separate files)
