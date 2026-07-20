# Repo Creation Guide

This document walks an agent through setting up a TypeScript repository for a basic web app, from empty directory to a verified working baseline. Follow the steps in order. Each step ends with a verification command — do not move on until it passes.

## Assumptions and defaults

- Runtime and package manager: **Bun** (fast installs, built-in TypeScript execution). If Bun is unavailable, substitute `npm` and `tsx`; the structure is the same.
- Stack: **Vite + React + TypeScript** for the frontend. This is a sensible default for a basic web app; swap in another framework only if the task demands it.
- Tooling: **ESLint** (flat config), **Prettier**, **Vitest**, strict TypeScript.
- Git hosting: GitHub, with CI via GitHub Actions.

Check prerequisites before starting:

```sh
bun --version   # need >= 1.0
git --version
```

## 1. Initialize the repository

```sh
git init
```

Create `.gitignore`:

```gitignore
node_modules/
dist/
coverage/
*.local
.env
.DS_Store
```

Never commit `.env` files or secrets. If the app needs configuration, create a committed `.env.example` with placeholder values instead.

## 2. Scaffold the app

```sh
bun create vite@latest . --template react-ts
bun install
```

If the directory is not empty, scaffold into a temp directory and move the files in. Verify:

```sh
bun run dev
```

The dev server should start on `http://localhost:5173`. Stop it with Ctrl+C once confirmed. (If running unattended, use `curl -sf http://localhost:5173 >/dev/null` against a backgrounded server instead of checking visually.)

## 3. Tighten the TypeScript config

Vite's template ships a reasonable `tsconfig.json`, but confirm these compiler options are set in `tsconfig.app.json` (or `tsconfig.json` if there is no project-references split):

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "verbatimModuleSyntax": true
  }
}
```

Verify:

```sh
bunx tsc --noEmit -p tsconfig.app.json
```

## 4. Set up formatting and linting

Install:

```sh
bun add -d prettier eslint typescript-eslint eslint-plugin-react-hooks
```

Create `.prettierrc.json`:

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100
}
```

Create `.prettierignore`:

```
dist/
coverage/
bun.lock
```

The Vite template ships an `eslint.config.js` using the flat config format; keep it. If starting from scratch, a minimal flat config:

```js
// eslint.config.js
import tseslint from 'typescript-eslint';

export default tseslint.config(
  { ignores: ['dist/', 'coverage/'] },
  tseslint.configs.recommended,
);
```

Verify:

```sh
bunx prettier --check .
bunx eslint .
```

Fix any issues with `bunx prettier --write .` before committing.

## 5. Set up testing

```sh
bun add -d vitest @testing-library/react @testing-library/jest-dom jsdom
```

Create `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: false,
  },
});
```

Write one real test to prove the harness works, e.g. `src/App.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import { expect, test } from 'vitest';
import App from './App';

test('renders the app', () => {
  render(<App />);
  expect(screen.getByRole('heading')).toBeDefined();
});
```

Verify:

```sh
bunx vitest run
```

## 6. Wire up package scripts

Ensure `package.json` has a single canonical entry point for every check. Agents and CI should always go through these, never ad-hoc commands:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit -p tsconfig.app.json",
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "test": "vitest run",
    "check": "bun run typecheck && bun run lint && bun run format:check && bun run test"
  }
}
```

Verify the whole gate:

```sh
bun run check
bun run build
```

Both must pass with zero errors.

## 7. Structure the source tree

Organize `src/` by capability, not by file type. A basic starting shape:

```
src/
  app/          # App shell, routing, providers
  components/   # Shared presentational components
  features/     # One folder per feature (components + logic + tests together)
  lib/          # Pure utilities, no framework imports
  main.tsx      # Entry point
```

Rules to establish now, while the repo is small:

- Keep tests next to the code they test (`foo.ts` / `foo.test.ts`).
- Keep `lib/` free of React and browser globals so it stays trivially testable.
- Avoid barrel files (`index.ts` re-exports) until there is a demonstrated need.

## 8. Add agent instructions

Create `AGENTS.md` at the repo root so future agents know how to work here:

```markdown
# AGENTS.md

## Commands

- `bun run check` — typecheck, lint, format check, and tests. Run before every commit.
- `bun run dev` — dev server on http://localhost:5173
- `bun run build` — production build

## Conventions

- Strict TypeScript; do not weaken compiler options or use `any` to silence errors.
- Tests live next to source files. New logic in `src/lib` or `src/features` needs tests.
- Format with Prettier via `bun run format`; do not hand-format.

## Structure

- `src/features/<name>/` — feature code, one folder per feature
- `src/lib/` — pure utilities, no React or browser imports
```

Also create a short `README.md` covering what the app is, how to install (`bun install`), and how to run it (`bun run dev`).

## 9. Add CI

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun run check
      - run: bun run build
```

CI runs the exact same `check` script as local development — one gate, defined once.

## 10. First commit

```sh
git add -A
git status        # review: no secrets, no node_modules, no dist
git commit -m "Scaffold TypeScript web app with Vite, ESLint, Prettier, Vitest, and CI"
```

If pushing to GitHub:

```sh
gh repo create <name> --private --source . --push
```

## Final verification checklist

Before declaring the repo ready, confirm all of the following:

- [ ] `bun run check` passes (typecheck, lint, format, tests)
- [ ] `bun run build` succeeds
- [ ] `bun run dev` serves the app
- [ ] `.gitignore` excludes `node_modules/`, `dist/`, and `.env`
- [ ] `AGENTS.md` and `README.md` exist and match reality
- [ ] CI workflow runs the same `check` script used locally
- [ ] The initial commit contains no secrets or generated artifacts
