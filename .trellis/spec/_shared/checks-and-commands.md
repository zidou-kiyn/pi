# Checks And Commands

## When This Applies

After any code change, before reporting work as done, and before every commit
(the pre-commit hook runs the same gate).

## The Local Pattern

### The one command that matters

```bash
npm run check
```

Run it with full output, not piped through `tail`. Fix every error, warning,
and info before committing. It does **not** run tests.

`npm run check` is a chain of seven steps (root `package.json`); the first
failure stops the chain:

| # | Step | Script | What it fails on |
|---|---|---|---|
| 1 | `biome check --write --error-on-warnings .` | `biome.json` | Lint violations and formatting; **rewrites files in place** |
| 2 | `check:pinned-deps` | `scripts/check-pinned-deps.mjs` | Any direct external dep in any `package.json` that is not an exact version |
| 3 | `check:ts-imports` | `scripts/check-ts-relative-imports.mjs` | A relative import/export/dynamic-import specifier ending in `.js` |
| 4 | `check:shrinkwrap` | `scripts/generate-coding-agent-shrinkwrap.mjs --check` | `packages/coding-agent/npm-shrinkwrap.json` out of date, or a new lifecycle-script package not in the allowlist |
| 5 | `check:install-lock:coding-agent` | `scripts/generate-coding-agent-install-lock.mjs --check` | `packages/coding-agent/install-lock/` out of date, or a new install-script package outside `allowedInstallScriptPackages` |
| 6 | `tsgo --noEmit` | `tsconfig.json` | Type errors, and non-erasable syntax via `erasableSyntaxOnly` |
| 7 | `check:browser-smoke` | `scripts/check-browser-smoke.mjs` | `packages/ai` no longer bundles for the browser, or the agent tree-shake entry regresses |

Step 1 rewrites files, so always re-read a file after `npm run check` before
making further edits to it.

### Commands that need explicit permission

`npm run build` and `npm test` are not run unless the user asks. For tests, use
`./test.sh` or a single vitest file (see `testing.md`).

### Pre-commit

`.husky/pre-commit` runs, in order:

1. `node scripts/check-lockfile-commit.mjs` — blocks commits touching
   `package-lock.json` unless `PI_ALLOW_LOCKFILE_CHANGE=1`
2. `npm run check`
3. `npm run check:browser-smoke` again, but only when the staged set touches
   `packages/ai/*`, `package.json`, or `package-lock.json` (the hook also lists
   a `web-ui` package branch, which matches nothing in the current tree)
4. re-stages the files it had staged, because step 2 may have reformatted them

Never bypass it with `git commit --no-verify`.

### CI parity

`.github/workflows/ci.yml` runs `npm ci --ignore-scripts`, `npm run build`,
`npm run check`, `npm test` on Node 22 for every push to `main` and every PR.
A green local `npm run check` is necessary but not sufficient — CI also builds
and tests.

### Model catalog regeneration

`packages/ai/src/models.generated.ts` is never edited by hand. Change
`packages/ai/scripts/generate-models.ts`, then run `npm run generate:models`.
The resulting diff may include unrelated upstream metadata churn; committing
that alongside your change is expected.

## Reference Files

- root `package.json` — the `check*` script definitions
- `scripts/check-pinned-deps.mjs`, `scripts/check-ts-relative-imports.mjs`,
  `scripts/check-browser-smoke.mjs`, `scripts/check-lockfile-commit.mjs`
- `scripts/generate-coding-agent-shrinkwrap.mjs`,
  `scripts/generate-coding-agent-install-lock.mjs`
- `.husky/pre-commit`
- `.github/workflows/ci.yml`

## Anti-Patterns

| Anti-pattern | Consequence |
|---|---|
| `npm run check 2>&1 \| tail -20` | Hides earlier failures in the chain |
| Committing with warnings unfixed | `--error-on-warnings` means CI fails anyway |
| `git commit --no-verify` | Skips the lockfile guard and the whole check gate |
| Running `npm run build` / `npm test` unprompted | Slow, and explicitly out of scope per `AGENTS.md` |
| Editing `npm-shrinkwrap.json` or `install-lock/` by hand | Regenerate with the owning script instead |
