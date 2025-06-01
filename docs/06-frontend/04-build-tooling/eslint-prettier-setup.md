# ESLint and Prettier Setup

`keywords: eslint, prettier, flat-config, eslint9, vue-eslint, typescript-eslint, auto-fix, formatting, linting, ci-cd, editor-integration`

> **Principle:** Linting and formatting are automated, non-negotiable quality gates. Configure them once, run them everywhere (editor, pre-commit, CI), and never debate style in code review again.

---

## ESLint 9 Flat Config with Vue and TypeScript

ESLint 9 introduced the flat config format (`eslint.config.js`). This replaces `.eslintrc.*` files with a single JavaScript/TypeScript configuration using arrays of config objects.

### Basic Setup

Install dependencies:

```bash
pnpm add -D eslint @eslint/js typescript-eslint eslint-plugin-vue
```

Create `eslint.config.ts`:

```ts
// eslint.config.ts
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import pluginVue from 'eslint-plugin-vue';

export default tseslint.config(
  // Global ignores
  {
    ignores: [
      'dist/**',
      'coverage/**',
      'node_modules/**',
      'src/auto-imports.d.ts',
      'src/components.d.ts',
    ],
  },

  // Base JS rules
  js.configs.recommended,

  // TypeScript rules
  ...tseslint.configs.recommended,

  // Vue rules
  ...pluginVue.configs['flat/recommended'],

  // Vue files use TypeScript parser
  {
    files: ['**/*.vue'],
    languageOptions: {
      parserOptions: {
        parser: tseslint.parser,
      },
    },
  },

  // Project-specific overrides
  {
    rules: {
      'no-console': 'warn',
      'no-debugger': 'warn',
    },
  },
);
```

### With Type-Aware Rules

For stricter checking that uses the TypeScript compiler:

```ts
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import pluginVue from 'eslint-plugin-vue';

export default tseslint.config(
  {
    ignores: ['dist/**', 'coverage/**', 'node_modules/**'],
  },

  js.configs.recommended,

  // Type-aware rules (slower but more thorough)
  ...tseslint.configs.recommendedTypeChecked,

  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },

  ...pluginVue.configs['flat/recommended'],

  {
    files: ['**/*.vue'],
    languageOptions: {
      parserOptions: {
        parser: tseslint.parser,
      },
    },
  },
);
```

Type-aware rules catch more issues (like `no-floating-promises`, `no-misused-promises`) but run slower because they invoke the TypeScript compiler. Use them when correctness matters more than speed.

---

## Prettier Integration

Prettier handles formatting. ESLint handles logic errors and code quality. Never let them fight.

### Install

```bash
pnpm add -D prettier eslint-config-prettier
```

### Configure Prettier

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf",
  "vueIndentScriptAndStyle": false
}
```

### Disable ESLint Formatting Rules

Add `eslint-config-prettier` as the **last** config to disable all ESLint rules that conflict with Prettier:

```ts
// eslint.config.ts
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import pluginVue from 'eslint-plugin-vue';
import prettier from 'eslint-config-prettier';

export default tseslint.config(
  { ignores: ['dist/**', 'coverage/**'] },
  js.configs.recommended,
  ...tseslint.configs.recommended,
  ...pluginVue.configs['flat/recommended'],
  {
    files: ['**/*.vue'],
    languageOptions: {
      parserOptions: { parser: tseslint.parser },
    },
  },

  // Must be last — disables conflicting format rules
  prettier,
);
```

### Prettier Ignore File

```bash
# .prettierignore
dist
coverage
node_modules
pnpm-lock.yaml
*.min.js
src/auto-imports.d.ts
src/components.d.ts
```

---

## Vue-Specific Rules

### vue/recommended Rules

The `flat/recommended` config includes sensible defaults. Common overrides:

```ts
{
  rules: {
    // Component naming
    'vue/component-name-in-template-casing': ['error', 'PascalCase'],
    'vue/component-definition-name-casing': ['error', 'PascalCase'],

    // Template style
    'vue/html-self-closing': ['error', {
      html: { void: 'always', normal: 'always', component: 'always' },
      svg: 'always',
      math: 'always',
    }],
    'vue/max-attributes-per-line': ['warn', {
      singleline: 3,
      multiline: 1,
    }],

    // Script setup preferences
    'vue/define-macros-order': ['error', {
      order: ['defineProps', 'defineEmits', 'defineOptions', 'defineSlots'],
    }],
    'vue/block-order': ['error', {
      order: ['script', 'template', 'style'],
    }],

    // Prevent common mistakes
    'vue/no-unused-vars': 'error',
    'vue/no-mutating-props': 'error',
    'vue/no-v-html': 'warn',
    'vue/require-v-for-key': 'error',
  },
}
```

---

## TypeScript-ESLint Rules

### Recommended Overrides

```ts
{
  rules: {
    // Allow unused vars when prefixed with underscore
    '@typescript-eslint/no-unused-vars': ['error', {
      argsIgnorePattern: '^_',
      varsIgnorePattern: '^_',
    }],

    // Enforce consistent type imports
    '@typescript-eslint/consistent-type-imports': ['error', {
      prefer: 'type-imports',
      fixStyle: 'inline-type-imports',
    }],

    // Allow explicit any in specific cases (warn instead of error)
    '@typescript-eslint/no-explicit-any': 'warn',

    // Enforce return types on exported functions only
    '@typescript-eslint/explicit-function-return-type': 'off',
    '@typescript-eslint/explicit-module-boundary-types': 'off',
  },
}
```

---

## Auto-Fix on Save

### VS Code Settings

Create `.vscode/settings.json` in the project root:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },

  "[vue]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[json]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },

  "eslint.validate": [
    "javascript",
    "typescript",
    "vue"
  ]
}
```

### Recommended VS Code Extensions

```json
// .vscode/extensions.json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "Vue.volar"
  ]
}
```

The workflow: Prettier formats on save, then ESLint fixes remaining issues. Both run automatically without developer intervention.

---

## Running in CI/CD

### Package.json Scripts

```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

### CI Pipeline

```yaml
# In your CI configuration
steps:
  - name: Lint
    run: pnpm lint

  - name: Format check
    run: pnpm format:check
```

Use `--check` in CI (not `--write`). CI should report violations, not fix them. Developers fix locally.

### Failing Fast

Run linting early in your CI pipeline. It is faster than tests and catches issues immediately:

```yaml
steps:
  - pnpm lint          # Seconds — catches code quality issues
  - pnpm format:check  # Seconds — catches formatting issues
  - pnpm typecheck     # Seconds — catches type errors
  - pnpm test          # Minutes — catches logic errors
  - pnpm build         # Minutes — catches build errors
```

---

## Custom Rules for Project Conventions

### No Restricted Imports

Enforce import conventions across the project:

```ts
{
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        {
          group: ['../*'],
          message: 'Prefer path aliases (@/) over relative parent imports.',
        },
      ],
      paths: [
        {
          name: 'lodash',
          message: 'Use lodash-es for tree-shaking support.',
        },
        {
          name: 'moment',
          message: 'Use date-fns instead of moment.js.',
        },
      ],
    }],
  },
}
```

### File Naming Conventions

Use `eslint-plugin-check-file` for consistent file names:

```bash
pnpm add -D eslint-plugin-check-file
```

```ts
import checkFile from 'eslint-plugin-check-file';

// In your config array:
{
  plugins: { 'check-file': checkFile },
  rules: {
    'check-file/filename-naming-convention': ['error', {
      '**/*.vue': 'PASCAL_CASE',
      '**/*.ts': 'CAMEL_CASE',
      '**/*.spec.ts': 'CAMEL_CASE',
    }],
    'check-file/folder-naming-convention': ['error', {
      'src/**/': 'KEBAB_CASE',
    }],
  },
}
```

---

## Ignoring Generated Files

Generated files (auto-imports, component declarations, API types) should be excluded from linting and formatting:

```ts
// eslint.config.ts — global ignores
{
  ignores: [
    'dist/**',
    'coverage/**',
    'node_modules/**',

    // Generated files
    'src/auto-imports.d.ts',
    'src/components.d.ts',
    'src/api/generated/**',

    // Build artifacts
    '*.min.js',
    '*.d.ts',
  ],
}
```

Keep `.prettierignore` in sync with ESLint ignores.

---

## Complete Configuration Example

A full working setup combining everything:

```ts
// eslint.config.ts
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import pluginVue from 'eslint-plugin-vue';
import prettier from 'eslint-config-prettier';

export default tseslint.config(
  // Ignores
  {
    ignores: [
      'dist/**',
      'coverage/**',
      'node_modules/**',
      'src/auto-imports.d.ts',
      'src/components.d.ts',
    ],
  },

  // Base configs
  js.configs.recommended,
  ...tseslint.configs.recommended,
  ...pluginVue.configs['flat/recommended'],

  // Vue file parser
  {
    files: ['**/*.vue'],
    languageOptions: {
      parserOptions: { parser: tseslint.parser },
    },
  },

  // Project rules
  {
    rules: {
      'no-console': 'warn',
      'no-debugger': 'warn',
      '@typescript-eslint/no-unused-vars': ['error', {
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      '@typescript-eslint/consistent-type-imports': ['error', {
        prefer: 'type-imports',
        fixStyle: 'inline-type-imports',
      }],
      'vue/component-name-in-template-casing': ['error', 'PascalCase'],
      'vue/block-order': ['error', {
        order: ['script', 'template', 'style'],
      }],
    },
  },

  // Prettier must be last
  prettier,
);
```

---

## Best Practices

### DO

- Use ESLint for code quality, Prettier for formatting — never mix responsibilities
- Place `eslint-config-prettier` last in the config array to disable conflicting rules
- Run `lint` and `format:check` in CI as fast-fail gates
- Commit `.vscode/settings.json` and `.vscode/extensions.json` for consistent editor behavior
- Use `--fix` locally, `--check` in CI
- Start with `recommended` configs and override only what you need
- Prefix unused variables with underscore (`_unused`) instead of disabling the rule

### DON'T

- Don't use `eslint-plugin-prettier` — it runs Prettier inside ESLint and is slower than running them separately
- Don't configure formatting rules in ESLint (indent, quotes, semi) — let Prettier handle all formatting
- Don't disable rules globally to fix a single file — use inline `eslint-disable` with a comment explaining why
- Don't skip CI linting because "it works locally" — environments differ
- Don't use the legacy `.eslintrc.*` format — ESLint 9 flat config is the current standard
- Don't auto-fix in CI — report violations and let developers fix them intentionally

---

## Guidelines

### Essential

- ESLint 9 flat config with `typescript-eslint` and `eslint-plugin-vue`
- Prettier configured with `eslint-config-prettier` to prevent conflicts
- CI pipeline runs `lint` and `format:check` before tests
- Editor configured for auto-fix on save

### Recommended

- Type-aware rules (`recommendedTypeChecked`) for stricter checking
- Custom rules for project import conventions (`no-restricted-imports`)
- File naming convention enforcement via `eslint-plugin-check-file`
- VS Code workspace settings and extension recommendations committed to repo

### Advanced

- Custom ESLint plugin for project-specific rules
- Lint-staged integration for pre-commit checking
- Per-directory rule overrides for different code areas (tests, scripts, source)
- Performance profiling with `TIMING=1 eslint .`

---

## Benefits

- Consistent code style across all contributors and AI-generated code
- Automated formatting eliminates style debates in code review
- Catch bugs early with static analysis before tests run
- Fast CI feedback with sub-second lint and format checks
- Type-safe ESLint configuration with IDE autocompletion

---

## Related

- [vite-configuration.md](vite-configuration.md) — Build tool configuration that ESLint and Prettier integrate with
- [cicd-validation-over-hooks.md](cicd-validation-over-hooks.md) — CI/CD strategy for running linting as a quality gate
