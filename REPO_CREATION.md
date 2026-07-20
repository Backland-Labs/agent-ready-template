# Repo Creation Instructions

You are an agent setting up a new TypeScript repository for a basic web app. Execute the steps below in order, starting from an empty directory. Every step ends with a verification command. If verification fails, fix the problem before proceeding — never continue past a failing check. Do not ask the user for input unless a step explicitly says to.

## The user is nontechnical

The person you are working for does not read code and is not familiar with developer tooling. This changes how you communicate, not what you build:

- Report progress in plain language: "I set up automatic code checks that run before every save to the project's history," not "I configured a Husky pre-commit hook running the Biome gate."
- When something fails, say what it means for them and what you are doing about it. Never paste raw error output as your explanation — summarize it, keep the raw output available if they ask.
- Never ask them to choose between technical options (library A vs. B, config flags). Make the choice yourself using the defaults below and mention what you picked in one sentence.
- If you genuinely need input, ask about outcomes they can judge ("Should visitors need to log in?"), never implementation.
- Do not ask them to run commands. Everything in this document is yours to execute.

## Defaults

Use these unless the task that invoked you says otherwise:

- Runtime and package manager: **Bun**. If `bun` is not on PATH and you cannot install it, fall back to `npm` and adjust commands accordingly.
- Stack: **Vite + React + TypeScript**.
- Tooling: **Biome** (linting and formatting), **Vitest**, strict TypeScript, **Husky** pre-commit hooks.
- Hosting: GitHub with GitHub Actions CI.

Before starting, confirm prerequisites and record the versions in your working notes:

```sh
bun --version   # require >= 1.0
git --version
```

## 1. Initialize the repository

```sh
git init
```

Write `.gitignore` with exactly this content, before anything else exists to accidentally commit:

```gitignore
node_modules/
dist/
coverage/
*.local
.env
.DS_Store
```

Rules you must follow for the rest of this setup:

- Never write real secrets to any file. If the app needs configuration, create a committed `.env.example` with placeholder values.
- Never commit `node_modules/` or build output.

## 2. Scaffold the app

```sh
bun create vite@latest . --template react-ts
bun install
```

If the target directory is not empty, scaffold into a temporary directory inside your scratchpad, then move the generated files into the target. Do not delete existing files to make room — stop and report the conflict instead.

The Vite template ships ESLint; this repo uses Biome instead. Remove the template's linter so there is exactly one tool with one config:

```sh
bun remove eslint typescript-eslint eslint-plugin-react-hooks eslint-plugin-react-refresh @eslint/js globals 2>/dev/null
rm -f eslint.config.js
```

(Some of those packages may not be present depending on template version; removing what exists is sufficient.)

Verify the dev server non-interactively:

```sh
bun run dev &
sleep 3
curl -sf http://localhost:5173 >/dev/null && echo OK
kill %1
```

Require `OK`. Do not verify by describing what a browser "should" show — you must see the successful curl.

## 3. Tighten the TypeScript config

Open `tsconfig.app.json` (or `tsconfig.json` if the template has no project-references split) and ensure these compiler options are present, adding any that are missing:

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

Never weaken these later to silence an error; fix the code instead.

Verify:

```sh
bunx tsc --noEmit -p tsconfig.app.json
```

## 4. Set up Biome for linting and formatting

```sh
bun add -d @biomejs/biome
bunx biome init
```

`biome init` writes `biome.json` with a schema line matching the installed version — keep that line, then replace the rest with the config below. This ruleset is deliberately strict: it enforces complexity limits, bans `any` and non-null assertions, requires exhaustive React hook dependencies, and checks accessibility. Do not turn rules off to make existing code pass; fix the code. Requires Biome 2.x.

```jsonc
{
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": { "ignoreUnknown": true },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf"
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "semicolons": "always",
      "trailingCommas": "all",
      "arrowParentheses": "always"
    }
  },
  "linter": {
    "enabled": true,
    "domains": { "react": "recommended" },
    "rules": {
      "recommended": true,
      "complexity": {
        "noExcessiveCognitiveComplexity": {
          "level": "error",
          "options": { "maxAllowedComplexity": 12 }
        },
        "noExcessiveLinesPerFunction": {
          "level": "error",
          "options": { "maxLines": 70, "skipBlankLines": true }
        },
        "noForEach": "error",
        "noUselessFragments": "error"
      },
      "correctness": {
        "noUnusedImports": "error",
        "noUnusedVariables": "error",
        "noNestedComponentDefinitions": "error",
        "useExhaustiveDependencies": "error",
        "useHookAtTopLevel": "error",
        "useJsxKeyInIterable": "error"
      },
      "nursery": {
        "noFloatingPromises": "error",
        "noMisusedPromises": "error"
      },
      "performance": { "noNamespaceImport": "error" },
      "style": {
        "noNonNullAssertion": "error",
        "noParameterAssign": "error",
        "noUselessElse": "error",
        "useImportType": "error",
        "noExcessiveLinesPerFile": {
          "level": "warn",
          "options": { "maxLines": 500, "skipBlankLines": true }
        }
      },
      "suspicious": {
        "noExplicitAny": "error",
        "noImplicitAnyLet": "error",
        "noArrayIndexKey": "error",
        "noConsole": {
          "level": "error",
          "options": { "allow": ["warn", "error"] }
        }
      },
      "a11y": {
        "useButtonType": "error",
        "noLabelWithoutControl": "error",
        "useKeyWithClickEvents": "error",
        "useSemanticElements": "error"
      }
    }
  },
  "overrides": [
    {
      "includes": ["**/*.test.ts", "**/*.test.tsx", "**/*.spec.ts", "**/*.spec.tsx"],
      "linter": {
        "rules": {
          "style": { "noExcessiveLinesPerFile": "off" },
          "suspicious": { "noConsole": "off", "noExplicitAny": "off" }
        }
      }
    }
  ]
}
```

If a nursery rule name is rejected by the installed Biome version, check `bunx biome rage` for the version and consult the Biome docs for the rule's current group — move the rule, do not drop it.

Format the scaffolded files once, then verify:

```sh
bunx biome check --write .
bunx biome check .
```

`biome check` covers both linting and formatting — it must exit 0. The template's scaffolded code may violate the strict rules (e.g. `App.tsx` complexity); refactor it until the check passes.

## 5. Set up testing

```sh
bun add -d vitest @testing-library/react @testing-library/jest-dom jsdom
```

Write `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: false,
  },
});
```

Write one real test that renders the actual app, `src/App.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import { expect, test } from 'vitest';
import App from './App';

test('renders the app', () => {
  render(<App />);
  expect(screen.getByRole('heading')).toBeDefined();
});
```

If `App` has no heading, adapt the assertion to something the real component renders. Do not write a placeholder test like `expect(true).toBe(true)`.

Verify:

```sh
bunx vitest run
```

## 6. Wire up package scripts

Edit `package.json` so these scripts exist exactly. From this point on, run all checks through these scripts — never ad-hoc variants — so local runs, the pre-commit hook, CI, and future agents all execute the same gate:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit -p tsconfig.app.json",
    "lint": "biome check .",
    "format": "biome check --write .",
    "test": "vitest run",
    "check": "bun run typecheck && bun run lint && bun run test"
  }
}
```

Verify:

```sh
bun run check
bun run build
```

Both must exit 0.

## 7. Set up Husky pre-commit hooks

```sh
bun add -d husky
bunx husky init
```

`husky init` adds a `"prepare": "husky"` script and creates `.husky/pre-commit` with a placeholder. Overwrite the hook to run the canonical gate:

```sh
echo "bun run check" > .husky/pre-commit
```

Verify the hook actually fires by attempting a commit later in step 10 — a passing `bun run check` must appear in the commit output. If you need to confirm earlier, run the hook directly:

```sh
sh .husky/pre-commit
```

Never use `--no-verify` to bypass a failing hook; fix the failure.

## 8. Structure the source tree

Reorganize `src/` toward this shape (create the directories now even if some start empty of features):

```
src/
  app/          # App shell, routing, providers
  components/   # Shared presentational components
  features/     # One folder per feature (components + logic + tests together)
  lib/          # Pure utilities, no framework imports
  main.tsx      # Entry point
```

Enforce these rules in all code you write here:

- Place tests next to the code they test (`foo.ts` / `foo.test.ts`).
- Keep `lib/` free of React and browser globals.
- Do not create barrel files (`index.ts` re-exports).

Verify `bun run check` still passes after any moves.

## 9. Write agent instructions

Write `AGENTS.md` at the repo root. This is the contract future agents (including you) operate under:

```markdown
# AGENTS.md

## Commands

- `bun run check` — typecheck, lint, format check, and tests. Also runs as the pre-commit hook.
- `bun run dev` — dev server on http://localhost:5173
- `bun run build` — production build
- `bun run format` — fix lint and formatting with Biome

## Working with the user

The repo owner is nontechnical. Report in plain language, decide technical
questions yourself, and only ask about outcomes they can judge.

## Conventions

- Strict TypeScript; do not weaken compiler options or use `any` to silence errors.
- Biome is the only linter and formatter; do not add ESLint or Prettier. Do not
  weaken the rules in `biome.json` to make code pass.
- Tests live next to source files. New logic in `src/lib` or `src/features` needs tests.
- Never commit with `--no-verify`.

## Structure

- `src/features/<name>/` — feature code, one folder per feature
- `src/lib/` — pure utilities, no React or browser imports
```

Also write a short `README.md`: what the app is, `bun install`, `bun run dev`. Keep both files accurate to what actually exists in the repo — do not document aspirations.

## 10. Add CI

Write `.github/workflows/ci.yml`:

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

CI must run the same `check` script as the pre-commit hook and local development. Do not add CI-only check commands.

## 11. Commit and push

```sh
git add -A
git status
```

Inspect the status output before committing. Abort and fix if you see `node_modules/`, `dist/`, any `.env` file, or anything resembling a credential. Then:

```sh
git commit -m "Scaffold TypeScript web app with Vite, Biome, Vitest, Husky, and CI"
```

The pre-commit hook must run and pass during this commit — confirm its output appears. Only create and push a remote if the task asked for one:

```sh
gh repo create <name> --private --source . --push
```

## Completion criteria

Report the repo as ready only when every item below is true. State each result explicitly in your final report — do not summarize as "everything passed" without having run the commands in this session:

- [ ] `bun run check` exits 0 (typecheck, Biome lint + format, tests)
- [ ] `bun run build` exits 0
- [ ] The curl verification against the dev server printed `OK`
- [ ] The Husky pre-commit hook ran `bun run check` during the initial commit
- [ ] `.gitignore` excludes `node_modules/`, `dist/`, and `.env`
- [ ] ESLint and Prettier are fully removed; Biome is the only lint/format tool
- [ ] `AGENTS.md` and `README.md` exist and describe the repo as it actually is
- [ ] CI runs the same `check` script used locally
- [ ] The initial commit contains no secrets and no generated artifacts

If any item fails and you cannot fix it, report exactly which step failed and the full error output — do not report partial success as success.
