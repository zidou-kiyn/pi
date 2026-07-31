# Implementation Plan

Only `.trellis/**` is modified. No file under `packages/`, `scripts/`,
`.github/`, or the root config files is touched.

## Step 0 — Safety net

```bash
cp -r .trellis/spec /tmp/trellis-spec-backup-$(date +%s)
```

`.trellis/` is untracked, so git cannot restore it. Keep the backup path in the
journal until the task is archived. This is the rollback point for every step
below.

## Step 1 — Prune the tree (main session)

```bash
rm -rf .trellis/spec/pi-extension-with-deps \
       .trellis/spec/pi-extension-custom-provider-anthropic \
       .trellis/spec/pi-extension-custom-provider-gitlab-duo \
       .trellis/spec/pi-extension-sandbox \
       .trellis/spec/pi-extension-gondolin \
       .trellis/spec/storage
rm -rf .trellis/spec/*/backend .trellis/spec/*/frontend
```

Then edit `.trellis/config.yaml`: drop the six removed packages from
`packages:`, set `default_package: pi-agent-core`.

Validate: `python3 ./.trellis/scripts/get_context.py --mode packages`
→ 7 packages, all showing `Spec: not configured`.

## Step 2 — Write `_shared` (main session, sets the house style)

Write in this order; each file follows the topic-file contract in `design.md` §4.

1. `_shared/typescript-and-style.md` — sources: `AGENTS.md` "Code Quality",
   `biome.json`, `tsconfig.base.json`, `tsconfig.json`,
   `scripts/check-ts-relative-imports.mjs`, real examples of the erasable-syntax
   constraint in `packages/*/src`.
2. `_shared/testing.md` — sources: `test.sh`, `vitest.base.ts`,
   `packages/*/vitest*.config.ts`, `packages/coding-agent/test/suite/harness.ts`,
   `packages/coding-agent/test/suite/README.md`,
   `packages/coding-agent/test/suite/regressions/`,
   `packages/agent/vitest.harness.config.ts`.
3. `_shared/checks-and-commands.md` — sources: root `package.json` scripts,
   `scripts/check-*.mjs`, `.husky/`, `.github/workflows/ci.yml`, `pr-gate.yml`.
4. `_shared/dependencies-and-git.md` — sources: `AGENTS.md` dependency/git
   sections, `scripts/check-pinned-deps.mjs`,
   `scripts/generate-coding-agent-shrinkwrap.mjs`,
   `scripts/generate-coding-agent-install-lock.mjs`, `.npmrc`.
5. `_shared/index.md` — scope, Pre-Development Checklist, Guidelines Index,
   pointers to `AGENTS.md` / `CONTRIBUTING.md` / `guides/index.md`.
6. Append a short "Project Rules" pointer section to
   `.trellis/spec/guides/index.md` linking `_shared/index.md`. Do not rewrite
   the existing guide content.

Validate: `grep -RIn "To be filled" .trellis/spec/_shared` is empty; every path
cited in `_shared` exists.

## Step 3 — Package layers via sub-agents

Dispatch `trellis-implement` sub-agents. Every dispatch prompt starts with
`Active task: .trellis/tasks/00-bootstrap-guidelines`, names the exact spec
directory the agent owns, forbids writing outside it, and requires it to read
`_shared/` first as the style reference.

Batch A (largest surfaces, parallel, disjoint dirs):
- `pi-coding-agent/{core,modes,extensions}`
- `pi-ai/{core,providers}`
- `pi-tui/{rendering,components}`

Batch B (parallel):
- `pi-agent-core/{agent-loop,harness}`
- `pi-server/server` + `pi-evals/evals`
- `pi-storage-sqlite-node/storage`

Per-agent acceptance: files match the `design.md` §4 contract, every referenced
path exists, no placeholder text, `index.md` lists exactly the files present.

## Step 4 — Consistency pass (main session)

- Read every produced file end to end.
- Check heading structure, tone, and length (60-150 lines per topic file).
- Remove rules that merely restate `_shared` without adding evidence.
- Verify no spec cites a path that does not exist.

## Step 5 — Verification

```bash
grep -RIn "To be filled\|TODO: fill\|placeholder\|(To be filled by the team)" .trellis/spec
python3 ./.trellis/scripts/get_context.py --mode packages
python3 ./.trellis/scripts/task.py validate 00-bootstrap-guidelines
git status --short          # nothing under packages/ may appear
```

Ad-hoc link checker: write to `/tmp/spec-link-check.mjs`, run it, delete it
(per `AGENTS.md` ad-hoc script rule).

Then dispatch `trellis-check` for an independent review against `prd.md`
acceptance criteria.

## Step 6 — Close out

- Tick the package checklist in `prd.md`.
- Record the outcome in `.trellis/workspace/zidou-kiyn/journal-1.md`.
- Ask the developer whether `.trellis/` should be committed before running any
  git command (this repo is a clone of the upstream `pi-mono`; `.trellis/`,
  `.agents/`, `.pi/` are all currently untracked).
- `python3 ./.trellis/scripts/task.py finish` then
  `python3 ./.trellis/scripts/task.py archive 00-bootstrap-guidelines`.

## Risky Files

| Path | Risk | Guard |
|---|---|---|
| `.trellis/config.yaml` | Breaking package discovery | Re-run `get_context.py --mode packages` right after editing |
| `.trellis/spec/guides/index.md` | Overwriting pre-filled guide content | Append-only edit |
| `.trellis/spec/**` (bulk `rm -rf`) | Untracked, unrecoverable | Step 0 backup |
| `packages/**` | Out of scope | `git status --short` at Step 5 |

## Not Run In This Task

`npm run check`, `npm run build`, `./test.sh` — this task changes only markdown
and YAML under `.trellis/`, and `AGENTS.md` restricts `npm run check` to code
changes.
