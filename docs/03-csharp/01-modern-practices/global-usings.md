# Global Usings

C# 10 feature that declares common `using` statements once for the entire project, eliminating repetitive imports in every file.

## Why It Matters

- Eliminates repeated `using` blocks across hundreds of files
- Cleaner files with fewer boilerplate lines
- Change once, applies everywhere

## Key Recommendations

**Create `GlobalUsings.cs` in the project root:**
```csharp
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;
global using LibraryService.Domain.Entities;
global using LibraryService.Infrastructure.Services;
```

**Enable implicit usings** (SDK provides System, LINQ, ASP.NET Core, etc.):
```xml
<PropertyGroup>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```
Then only add project-specific namespaces in `GlobalUsings.cs`.

**Include namespaces used in 80%+ of files.** Keep rare or ambiguous namespaces as local `using` statements.

**Handle conflicts** with explicit local `using` when two global namespaces have overlapping types.

## What to Include vs Exclude

| Include globally | Keep local |
|-----------------|------------|
| System, LINQ, Tasks | Rarely used namespaces |
| Framework (ASP.NET Core) | Ambiguous type namespaces |
| Project-wide domain types | Feature-specific namespaces |

## Pitfalls to Avoid

- Including third-party libraries globally unless truly universal
- Forgetting to remove now-redundant `using` statements from individual files
- Including namespaces that cause frequent type name conflicts
