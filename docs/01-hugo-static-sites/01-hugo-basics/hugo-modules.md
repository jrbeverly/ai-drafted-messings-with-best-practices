# Hugo Modules

Dependency management built on Go modules for themes, shared components, and asset packages.

## Why It Matters
- Replaces git submodules with proper version pinning and dependency resolution
- Enables theme composition from multiple modules with priority-based merging
- Supports vendoring for offline/reproducible builds

## Quick Start
```bash
hugo mod init github.com/username/mysite    # Creates go.mod
```
```toml
# config.toml
[module]
  [[module.imports]]
    path = "github.com/theNewDynamic/gohugo-theme-ananke"
```
```bash
hugo mod get      # Download modules
hugo server       # Use theme
```

## Key Commands
```bash
hugo mod get -u                              # Update all modules
hugo mod get github.com/user/theme@v2.0.0   # Pin specific version
hugo mod graph                               # Show dependency tree
hugo mod tidy                                # Remove unused modules
hugo mod clean                               # Clear module cache
hugo mod vendor                              # Copy to _vendor/ for offline builds
```

## Module Mounts
Map module directories to custom locations:
```toml
[[module.imports]]
  path = "github.com/username/components"
  [[module.imports.mounts]]
    source = "layouts/partials"
    target = "layouts/partials/vendor"
  [[module.imports.mounts]]
    source = "assets/scss"
    target = "assets/scss/vendor"
```

## Merge Priority
1. Project files (highest)
2. Last imported module
3. First imported module (lowest)

## Local Development
Use `replace` directive in `go.mod` to point to a local path:
```go
replace github.com/username/hugo-theme => ../local-hugo-theme
```

## Creating a Module
```bash
hugo mod init github.com/username/my-module
git tag v1.0.0 && git push --tags           # Publish with semantic version
```
Include: `go.mod`, `README.md`, `LICENSE`, and standard Hugo directories.

## Pitfalls
- Don't use `latest` in production -- always pin versions
- Don't forget `hugo mod tidy` after removing imports
- Don't create circular dependencies between modules
- Don't include `public/` or `resources/` in published modules
- Don't break backward compatibility in patch versions
