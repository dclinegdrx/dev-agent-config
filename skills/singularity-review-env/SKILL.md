---
name: singularity-review-env
description: Use when starting a Singularity PR review or before running Singularity tests to prepare Node, Corepack, pnpm, and dependencies from repo version metadata.
---

# Singularity Review Environment

Use this skill before running tests, lint, typecheck, or build commands while reviewing a Singularity PR. It prepares the local shell using the versions declared by the repository instead of hardcoding assumptions.

## Goal

Ensure the review environment is in a good state before validation commands run:

- Node matches `.nvmrc` / `package.json#engines.node`.
- pnpm matches `package.json#packageManager` / `package.json#engines.pnpm`.
- Corepack is enabled for the active Node version.
- Dependencies are installed before package scripts try to resolve local binaries like `jest`, `eslint`, or `tsc`.

## When To Run

Run this at the start of a Singularity PR review when the user expects local verification, or immediately after seeing errors like:

- `pnpm: command not found`
- `jest: command not found`
- `spawn ENOENT`
- `node_modules missing`
- `Unsupported engine: wanted {"node":"~24.14.0"}`

## Version Sources

Use repo metadata as source of truth:

- Node version: `.nvmrc`
- Required Node range: `package.json` `engines.node`
- pnpm version: `package.json` `packageManager`
- Required pnpm version: `package.json` `engines.pnpm`

For this repo, do not guess or recommend a global pnpm version without checking `package.json` first.

## Setup Workflow

Run commands from the repo root.

First inspect the declared versions:

```bash
node -p "({ nvmrc: require('fs').readFileSync('.nvmrc', 'utf8').trim(), nodeEngine: require('./package.json').engines?.node, packageManager: require('./package.json').packageManager, pnpmEngine: require('./package.json').engines?.pnpm })"
```

Then prepare the shell. This works in non-interactive bash sessions where `nvm` may not already be loaded:

```bash
export NVM_DIR="$HOME/.nvm"; [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"; nvm use; corepack enable; corepack prepare "$(node -p "require('./package.json').packageManager.split('+')[0]")" --activate; hash -r; node --version; pnpm --version
```

If `nvm use` reports that the required Node version is not installed, install it and rerun the setup command:

```bash
export NVM_DIR="$HOME/.nvm"; [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"; nvm install; nvm use
```

If `pnpm` is still missing after `corepack enable`, activate the exact repo version explicitly:

```bash
corepack prepare "$(node -p "require('./package.json').packageManager.split('+')[0]")" --activate
```

## Dependency Install

If `node_modules` is missing, or package scripts cannot find local binaries, install dependencies from the repo root:

```bash
pnpm install --frozen-lockfile
```

If the install fails because the lockfile is intentionally out of date on the branch, report that clearly and ask before running a non-frozen install. Do not modify `pnpm-lock.yaml` during a review unless the user explicitly asks.

## Validation After Setup

Before rerunning the requested validation command, confirm the toolchain:

```bash
node --version
pnpm --version
pnpm exec jest --version
```

Then rerun the focused command. For example:

```bash
pnpm --filter @goodrx/pages.accounts.conditions-subscription test -- MultiSelectQuestion.spec.tsx
```

## Troubleshooting Notes

- `pnpm: command not found` usually means Corepack was not enabled for the active Node version.
- With `nvm`, global tools are per Node version. Switching from Node 20 to Node 24 can make previously available globals disappear.
- `jest: command not found` usually means dependencies are not installed, because package scripts resolve `jest` from local `node_modules/.bin`.
- If `nvm` is not found in a non-interactive shell, source `$HOME/.nvm/nvm.sh` first as shown above.
- If CI passes but local tests cannot find binaries, treat it as local environment setup, not a PR failure.

## Response Guidance

When this skill fixes an environment issue, summarize:

- The Node version selected.
- The pnpm version activated.
- Whether dependencies were installed.
- The exact validation command rerun.
- Any remaining blocker, such as registry auth or lockfile mismatch.
