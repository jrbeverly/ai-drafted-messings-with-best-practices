# npm vs pnpm vs Yarn

`keywords: npm, pnpm, yarn, package-manager, workspace, monorepo, lock-file, caching, ci-cd, npmrc, corepack, node-modules`

> **Principle:** Pick one package manager for your project, configure it correctly, and enforce it everywhere. The best package manager is the one your team uses consistently.

---

## Comparison Overview

| Feature | npm | pnpm | Yarn (Berry/4+) |
|---------|-----|------|-----------------|
| **Speed** | Moderate | Fast | Fast |
| **Disk usage** | High (duplicates) | Low (content-addressable store) | Moderate (PnP) or Low (nodeLinker: pnpm) |
| **Strictness** | Loose (phantom deps) | Strict by default | Strict with PnP |
| **Workspace support** | Basic | Excellent | Excellent |
| **Lock file** | `package-lock.json` | `pnpm-lock.yaml` | `yarn.lock` |
| **Ships with Node** | Yes | Via Corepack | Via Corepack |
| **Learning curve** | Low | Low | Moderate (PnP) |
| **Ecosystem compat** | Universal | Very high | High (some PnP issues) |

---

## npm

npm ships with Node.js. It works everywhere with no extra setup.

### Strengths

- Zero installation — comes with Node.js
- Universal ecosystem support
- Familiar to every JavaScript developer
- `npx` for running one-off commands

### Weaknesses

- Flat `node_modules` allows phantom dependencies (you can import packages you did not declare)
- Slower than pnpm for large projects
- Duplicates packages across projects on disk
- Workspace support is functional but less polished than alternatives

### Configuration

```ini
# .npmrc
engine-strict=true
save-exact=false
audit=true
fund=false
```

### Workspace Setup

```json
// package.json (root)
{
  "workspaces": [
    "packages/*",
    "apps/*"
  ]
}
```

```bash
# Run script in a specific workspace
npm run build -w packages/shared

# Install dependency in a specific workspace
npm install zod -w packages/shared

# Run script in all workspaces
npm run test --workspaces
```

---

## pnpm

pnpm uses a content-addressable store and strict `node_modules` structure. It is the recommended choice for projects that value correctness and efficiency.

### Strengths

- **Strict by default** — cannot import undeclared dependencies (no phantom deps)
- **Fast** — parallel operations, hardlinks from global store
- **Disk efficient** — packages stored once globally, hardlinked into projects
- **Excellent workspaces** — built-in filtering, parallel execution, dependency graph awareness
- **Content-addressable store** — deduplicates across all projects on your machine

### Weaknesses

- Not bundled with Node.js (requires Corepack or separate install)
- Strict `node_modules` can expose issues in poorly written packages
- Less common in documentation examples (most examples show npm)

### Installation

```bash
# Option 1: Corepack (recommended, built into Node.js)
corepack enable
corepack prepare pnpm@9.15.0 --activate

# Option 2: Standalone
npm install -g pnpm
```

### Configuration

```ini
# .npmrc
strict-peer-dependencies=false
auto-install-peers=true
shamefully-hoist=false
```

| Setting | Purpose |
|---------|---------|
| `strict-peer-dependencies=false` | Warn (not error) on peer dep mismatches |
| `auto-install-peers=true` | Automatically install peer dependencies |
| `shamefully-hoist=false` | Keep strict node_modules layout (default, do not change) |

### Workspace Setup

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'
```

```bash
# Run script in a specific package
pnpm --filter @myorg/shared run build

# Install dependency in a specific package
pnpm --filter @myorg/shared add zod

# Run script in all packages
pnpm -r run test

# Run script in all packages that changed since main
pnpm --filter '...[origin/main]' run test

# Run build in dependency order
pnpm -r --sort run build
```

### Workspace Protocol

Reference workspace packages using the `workspace:` protocol:

```json
// apps/portal/package.json
{
  "dependencies": {
    "@myorg/shared": "workspace:*",
    "@myorg/ui-components": "workspace:^1.0.0"
  }
}
```

- `workspace:*` — always resolve to the local workspace version
- `workspace:^1.0.0` — resolve locally, but when publishing, replace with the actual version

---

## Yarn (Berry / 4+)

Yarn Berry (versions 2+, currently v4) introduced Plug'n'Play (PnP) as an alternative to `node_modules`.

### Strengths

- **Plug'n'Play** — no `node_modules`, instant installs after first time
- **Constraints** — enforce dependency rules across workspaces
- **Offline capable** — zero-installs when `.yarn/cache` is committed
- **Strong workspace support** — parallel execution, topological ordering

### Weaknesses

- PnP mode has compatibility issues with some packages
- Requires editor SDK setup for IDE support
- Larger learning curve than npm or pnpm
- Less common in the ecosystem

### Configuration

```yaml
# .yarnrc.yml
nodeLinker: node-modules    # Use traditional node_modules (most compatible)
# nodeLinker: pnpm          # Use pnpm-style node_modules (strict)
# nodeLinker: pnp           # Use Plug'n'Play (strictest, fastest)

enableGlobalCache: true
```

### Workspace Setup

```json
// package.json (root)
{
  "workspaces": [
    "packages/*",
    "apps/*"
  ]
}
```

```bash
# Run script in a specific workspace
yarn workspace @myorg/shared run build

# Install dependency in a workspace
yarn workspace @myorg/shared add zod

# Run in all workspaces
yarn workspaces foreach -A run test

# Run in topological order (dependencies first)
yarn workspaces foreach -At run build
```

---

## Lock File Formats

Each package manager has its own lock file. They are not interchangeable.

| Manager | Lock File | Format | Merge Friendliness |
|---------|-----------|--------|-------------------|
| npm | `package-lock.json` | JSON | Poor (large diffs) |
| pnpm | `pnpm-lock.yaml` | YAML | Good (smaller diffs) |
| yarn | `yarn.lock` | Custom | Good (designed for merging) |

**Rules:**
- Commit exactly one lock file
- Add others to `.gitignore`
- Never manually edit lock files
- Resolve merge conflicts by deleting the lock file and running install

```bash
# .gitignore (when using pnpm)
package-lock.json
yarn.lock
.yarn/
```

---

## CI/CD Caching Strategies

### npm

```yaml
- name: Cache npm
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
    restore-keys: npm-

- run: npm ci
```

Use `npm ci` (not `npm install`) in CI. It is faster and ensures reproducible builds from the lock file.

### pnpm

```yaml
- name: Setup pnpm
  uses: pnpm/action-setup@v4

- name: Get pnpm store
  id: pnpm-cache
  run: echo "dir=$(pnpm store path)" >> $GITHUB_OUTPUT

- name: Cache pnpm
  uses: actions/cache@v4
  with:
    path: ${{ steps.pnpm-cache.outputs.dir }}
    key: pnpm-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: pnpm-

- run: pnpm install --frozen-lockfile
```

Use `--frozen-lockfile` in CI to fail if the lock file is out of date.

### Yarn

```yaml
- name: Cache yarn
  uses: actions/cache@v4
  with:
    path: .yarn/cache
    key: yarn-${{ hashFiles('yarn.lock') }}
    restore-keys: yarn-

- run: yarn install --immutable
```

Use `--immutable` in CI to fail if the lock file would change.

### Cache Key Strategy

The pattern `key: {manager}-{hash of lock file}` ensures:
- Exact cache hit when dependencies have not changed
- Full reinstall when dependencies change
- `restore-keys` prefix provides fallback to the most recent cache

---

## When to Use Which

### Use npm When

- Starting a simple project with no special requirements
- Working with a team unfamiliar with other managers
- Using tools that assume npm (some CI platforms, deployment services)
- Minimal configuration and setup is the priority

### Use pnpm When

- Correctness matters — strict dependency resolution catches phantom deps
- Disk space matters — shared content-addressable store
- Speed matters — faster installs, especially in monorepos
- Working in a monorepo — superior workspace filtering and dependency graph tools
- This is the **recommended default** for new projects

### Use Yarn When

- Already using Yarn in an existing project
- Need Plug'n'Play for zero-install workflows
- Need Yarn constraints for workspace dependency governance
- Working with a team that has Yarn expertise

---

## Migration Between Package Managers

### npm to pnpm

```bash
# Remove npm artifacts
rm -rf node_modules package-lock.json

# Install pnpm
corepack enable
corepack prepare pnpm@9.15.0 --activate

# Generate pnpm lock file
pnpm import     # Imports from package-lock.json if it still exists
# OR
pnpm install    # Generates fresh lock file

# Update .gitignore
echo "package-lock.json" >> .gitignore

# Add packageManager field
npm pkg set packageManager="pnpm@9.15.0"
```

### pnpm to npm

```bash
rm -rf node_modules pnpm-lock.yaml
npm install
echo "pnpm-lock.yaml" >> .gitignore
```

### Key Migration Checklist

1. Remove old `node_modules` and lock file
2. Install with the new manager
3. Update `.gitignore` to ignore old lock files
4. Update `packageManager` field in `package.json`
5. Update CI/CD scripts (install commands, caching)
6. Update developer documentation
7. Update any scripts that reference the old manager directly
8. Test the full build pipeline (dev, build, test, lint)

---

## .npmrc Configuration

The `.npmrc` file configures behavior for npm and pnpm (both read it). Commit it to your repository.

### Common Settings

```ini
# .npmrc

# Enforce Node.js version from engines field
engine-strict=true

# Suppress funding messages
fund=false

# Suppress audit on every install (run manually)
audit=false

# pnpm: auto-install peer dependencies
auto-install-peers=true

# pnpm: warn (not error) on peer dep mismatches
strict-peer-dependencies=false

# Registry (if using private registry)
# registry=https://npm.pkg.github.com
# @myorg:registry=https://npm.pkg.github.com
```

### Per-Project vs Global

- **Commit** `.npmrc` to the repo root for project-level settings
- **Do not commit** authentication tokens — use environment variables in CI
- **Global** `.npmrc` (`~/.npmrc`) for personal settings like registry auth

```ini
# CI environment variable for private registry auth
# Set this in CI secrets, not in .npmrc
# //npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

---

## Best Practices

### DO

- Pick one package manager and enforce it across the team
- Use `packageManager` field in `package.json` with Corepack
- Use frozen/immutable lock file installs in CI (`--frozen-lockfile`, `--immutable`, `ci`)
- Cache the package manager store in CI for faster builds
- Commit `.npmrc` with project-level configuration
- Audit dependencies periodically (`pnpm audit`, `npm audit`)

### DON'T

- Don't mix package managers in the same project
- Don't commit multiple lock files
- Don't use `install` in CI — use `ci` (npm) or `--frozen-lockfile` (pnpm)
- Don't commit authentication tokens in `.npmrc`
- Don't set `shamefully-hoist=true` in pnpm unless absolutely necessary (it defeats strictness)
- Don't ignore the lock file in version control

---

## Guidelines

### Essential

- One package manager per project, enforced via `packageManager` field
- Lock file committed to version control
- CI uses frozen/immutable installs
- `.npmrc` committed with project settings (no secrets)

### Recommended

- Use pnpm for new projects (strict, fast, disk efficient)
- Enable Corepack for automatic package manager version management
- Cache package store in CI pipelines
- Run `pnpm audit` or `npm audit` in CI

### Advanced

- Workspace filtering for monorepo partial builds
- Private registry configuration for internal packages
- pnpm `overrides` for transitive dependency fixes
- Yarn constraints for workspace dependency governance

---

## Benefits

- Reproducible installs across all environments with lock files
- Strict dependency resolution catches undeclared imports early
- Content-addressable storage saves disk space and install time
- Workspace support enables monorepo development with clear dependency graphs
- Corepack ensures consistent package manager versions without manual setup

---

## Related

- [package-json-structure.md](package-json-structure.md) — How to structure the manifest that the package manager reads
- [typescript-monorepo-packages.md](typescript-monorepo-packages.md) — Monorepo patterns using workspace features
