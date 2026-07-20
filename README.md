# agent-ready-template

A single prompt, `REPO_CREATION.md`, that walks a coding agent through setting up a TypeScript web app repository from scratch: Bun, Vite + React, strict TypeScript, Biome for linting and formatting, Vitest, and Husky pre-commit hooks.

The prompt is written for an agent working on behalf of a nontechnical user — it makes the technical decisions itself, verifies every step with a command, and reports back in plain language.

## Usage

Point an agent at an empty directory and give it the contents of `REPO_CREATION.md` as its task.
