# CI/CD Validation Over Git Hooks

`keywords: CI/CD, git-hooks, validation, linting, pipeline, Gitea-Actions, branch-protection, husky, lint-staged`

## Principle

**Use CI/CD pipelines as the authoritative enforcement layer for code quality checks.** Git hooks are a convenience supplement, not a replacement for server-side validation. Quality gates that matter must run in a controlled, non-bypassable environment.

---

## Why CI/CD Validation Is Better Than Git Hooks

Git hooks run on developer machines. CI/CD pipelines run on controlled servers. This distinction has fundamental implications for reliability.

### The Core Problem with Git Hooks as Gatekeepers

Git hooks are **advisory, not authoritative**. Any developer can bypass them:

```bash
# Bypass all hooks with a single flag
git commit --no-verify -m "skip everything"

# Or just delete the hook files
rm .husky/pre-commit
```

When hooks are the only quality gate, a single `--no-verify` lets broken code into the main branch. CI/CD pipelines enforced through branch protection cannot be bypassed by contributors.

---

## Problems with Git Hooks

### 1. Easily Bypassed

```bash
# Every developer knows this shortcut
git commit --no-verify -m "quick fix"
git push --no-verify

# Hooks are not enforced - they are voluntary
```

### 2. Environment-Dependent

Hooks run in the developer's local environment, which varies:

```bash
# Developer A: Node 20, everything works
# Developer B: Node 18, different ESLint behavior
# Developer C: Windows, path separator issues in scripts
# CI Server: Controlled Node 20 environment, consistent every time
```

### 3. Slow Feedback on Large Codebases

Pre-commit hooks that run full lint/typecheck/test suites slow down every commit:

```bash
# Typical pre-commit hook - runs on every commit
npm run lint        # 15 seconds
npm run typecheck   # 20 seconds
npm run test        # 45 seconds
# Total: 80 seconds per commit - developers start using --no-verify
```

### 4. Installation Friction

Hooks require setup and can fail silently:

```bash
# New developer clones repo, forgets to run:
npm install  # husky install may or may not run

# Hooks silently not installed - no validation at all
```

### 5. Inconsistent Across Tools

Different Git clients handle hooks differently:

- Command-line Git: runs hooks
- Some GUI clients: may skip hooks
- IDE Git integrations: behavior varies
- Git worktrees: hooks may not propagate

---

## CI Pipeline for Frontend Validation

Move all quality checks into a CI pipeline that runs on every push and pull request.

### Gitea Actions Workflow

```yaml
# .gitea/workflows/frontend-validation.yml
name: Frontend Validation

on:
  push:
    branches: [main]
    paths:
      - 'app/**'
      - 'package.json'
      - 'tsconfig.json'
  pull_request:
    branches: [main]
    paths:
      - 'app/**'
      - 'package.json'
      - 'tsconfig.json'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run typecheck

      - name: Unit tests
        run: npm run test:unit -- --reporter=verbose

      - name: Build
        run: npm run build
```

### Parallel Jobs for Faster Feedback

Split checks into parallel jobs to reduce total pipeline time:

```yaml
# .gitea/workflows/frontend-validation.yml
name: Frontend Validation

on:
  pull_request:
    branches: [main]
    paths:
      - 'app/**'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit

  build:
    runs-on: ubuntu-latest
    needs: [lint, typecheck, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
```

---

## Pre-Merge Checks

Ensure all validation passes before code enters the main branch.

### Branch Protection Rules

Configure branch protection to require CI status checks:

```yaml
# Repository settings (Gitea/GitHub)
# Branch protection for 'main':

# Required status checks:
#   - lint (must pass)
#   - typecheck (must pass)
#   - test (must pass)
#   - build (must pass)

# Additional protections:
#   - Require pull request before merging
#   - Require approvals: 1 (or 0 for solo projects with CI checks)
#   - Dismiss stale reviews on new commits
#   - Require branches to be up-to-date before merging
```

### Status Check Configuration

Map CI job names to required status checks:

```yaml
# Each job name becomes a status check
jobs:
  lint:        # Required check: "lint"
  typecheck:   # Required check: "typecheck"
  test:        # Required check: "test"
  build:       # Required check: "build"
```

With branch protection enabled, merging is blocked until all required checks pass. There is no `--no-verify` equivalent for CI pipelines.

---

## When Git Hooks ARE Appropriate

Git hooks serve well for fast, formatting-only, developer-experience tasks that do not serve as quality gates.

### Commit Message Format

Enforcing commit message conventions is a good use of hooks because the check is instant and the CI pipeline cannot retroactively fix a commit message:

```bash
#!/bin/sh
# .husky/commit-msg

# Enforce conventional commit format
commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore|build|ci)(\(.+\))?: .{1,72}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
  echo "ERROR: Commit message must follow Conventional Commits format."
  echo "  Format: type(scope): description"
  echo "  Example: feat(auth): add login form validation"
  echo ""
  echo "  Types: feat, fix, docs, style, refactor, test, chore, build, ci"
  exit 1
fi
```

### Auto-Formatting on Save (Pre-Commit)

Auto-formatting staged files is fast and improves developer experience without serving as a gate:

```bash
#!/bin/sh
# .husky/pre-commit

# Only format staged files - fast operation
npx lint-staged
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx,vue}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{css,scss}": [
      "prettier --write"
    ],
    "*.{json,md}": [
      "prettier --write"
    ]
  }
}
```

---

## Husky and lint-staged as Supplement, Not Replacement

Use husky and lint-staged for developer convenience. Never rely on them as the sole quality enforcement.

### Proper Setup

```bash
# Install
npm install -D husky lint-staged

# Initialize husky
npx husky init
```

```json
// package.json
{
  "scripts": {
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{ts,vue}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

```bash
#!/bin/sh
# .husky/pre-commit

# Fast formatting only - NOT a quality gate
npx lint-staged
```

### What Hooks Should Do vs. What CI Should Do

| Check | Git Hook | CI Pipeline |
|---|---|---|
| Auto-format staged files | Yes | No (already formatted) |
| Commit message format | Yes | Optional (verify) |
| Full ESLint check | No (slow) | Yes (required) |
| TypeScript type check | No (slow) | Yes (required) |
| Unit tests | No (slow) | Yes (required) |
| Build verification | No (slow) | Yes (required) |
| Bundle size check | No | Yes (required) |
| Security audit | No | Yes (required) |

### The Mental Model

```
Git Hooks = "Help developers catch issues early" (advisory)
CI Pipeline = "Prevent broken code from merging" (authoritative)

Hooks fail gracefully (--no-verify exists by design).
CI fails authoritatively (branch protection blocks merge).
```

---

## Best Practices

### DO

- Run all lint, typecheck, and test validation in CI pipelines
- Configure branch protection to require CI status checks before merge
- Use git hooks only for fast, formatting-only operations
- Use lint-staged to scope pre-commit hooks to changed files only
- Run CI jobs in parallel to minimize total pipeline time
- Pin Node version in CI to match project requirements
- Cache npm dependencies in CI for faster runs

### DON'T

- Rely on git hooks as the sole quality enforcement mechanism
- Run full test suites in pre-commit hooks (developers will bypass them)
- Skip CI validation because "hooks already check it"
- Assume all developers have hooks installed and working
- Block commits with slow hook operations (over 5 seconds)
- Use hooks for checks that vary by environment (integration tests, E2E tests)

---

## Guidelines

### Essential

- CI pipeline runs lint, typecheck, and tests on every pull request
- Branch protection requires CI status checks to pass before merge
- No quality-critical check relies solely on git hooks

### Recommended

- Parallel CI jobs for lint, typecheck, and test to reduce feedback time
- lint-staged with husky for auto-formatting staged files on commit
- Conventional commit message enforcement via commit-msg hook
- npm dependency caching in CI for faster pipeline execution

### Advanced

- Bundle size budget checks in CI (fail if budget exceeded)
- Automated security audit (npm audit) in CI pipeline
- Visual regression testing in CI for UI components
- Performance budget validation in CI

---

## Benefits

- **Non-bypassable quality gates** - branch protection enforces CI checks
- **Consistent environment** - CI runs same Node version, same OS every time
- **Fast developer workflow** - hooks only do formatting, CI handles heavy checks
- **Reliable enforcement** - works regardless of developer's local setup
- **Parallel execution** - CI jobs run simultaneously, reducing total wait time

---

## Related

- [eslint-prettier-setup.md](../eslint-prettier-setup.md) - ESLint and Prettier configuration
- [vite-configuration.md](../vite-configuration.md) - Vite build configuration for CI builds
