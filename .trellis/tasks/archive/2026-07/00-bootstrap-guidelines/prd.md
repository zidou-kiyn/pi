# Bootstrap Guidelines: Populate `.trellis/spec/` From The Real Codebase

> Phase 1 planning complete. Implementation starts only after `task.py start`.

## Goal

Replace the template-only `.trellis/spec/` tree produced by `trellis init` with
project-specific coding guidance derived from this repository, so that future
`trellis-implement` / `trellis-check` sub-agents write code matching the pi
monorepo's actual conventions instead of generic framework advice.

Spec documents are written in **English**. Planning discussion with the
developer happens in **Simplified Chinese**.

## Background

### Repository shape (measured)

| Package (config.yaml name) | Path | `src` TS files | LOC | Nature |
|---|---|---|---|---|
| pi-agent-core | `packages/agent` | 35 | ~10.0k | Agent loop + harness library |
| pi-ai | `packages/ai` | 169 | ~21.3k | Multi-provider LLM SDK + model catalog |
| pi-coding-agent | `packages/coding-agent` | 178 | ~55.6k | CLI app, TUI/RPC modes, tools, extensions |
| pi-evals | `packages/evals` | 8 | ~1.3k | vitest-evals harness |
| pi-server | `packages/server` | 13 | ~2.0k | HTTP/IPC supervisor for RPC sessions |
| pi-storage-sqlite-node | `packages/storage/sqlite-node` | 12 | ~1.6k | SQLite storage backend + SQL migrations |
| pi-tui | `packages/tui` | 30 | ~12.8k | Terminal UI toolkit + native addon |
| storage | `packages/storage` | 0 | 0 | Container directory, no `src` |
| pi-extension-* (5 entries) | `packages/coding-agent/examples/extensions/*` | 1-2 each | tiny | Documentation samples |

The five `pi-extension-*` entries are npm workspaces only because root
`package.json` lists `packages/coding-agent/examples/extensions/*`.

### Current spec tree

- 136 markdown files, ~7.3k lines; every non-`guides` file is pure template
  (`(To be filled by the team)`), with 13 identical copies per template file.
- Template layers are `backend` / `frontend` with web-oriented topics
  (`component-guidelines.md`, `hook-guidelines.md`, `state-management.md`,
  `database-guidelines.md`, ...). The repo has no web frontend, no React, and
  no database outside `pi-storage-sqlite-node`.
- `.trellis/spec/guides/` is already populated and repo-agnostic.

### Constraints discovered in the toolchain

- Spec layers are directory scans: `_scan_spec_layers`
  (`.trellis/scripts/common/packages_context.py:30`). Nothing hardcodes
  `backend` / `frontend`, so any layout is valid and is reflected immediately by
  `get_context.py --mode packages`.
- Only `.trellis/spec/guides/` is auto-advertised by
  `get_context_packages_text` (`packages_context.py:206-211`), and
  `trellis-before-dev` step 6 always reads `guides/index.md`. A repo-level
  `_shared` layer therefore needs a pointer from `guides/index.md` to be
  discoverable.
- `default_package: @earendil-works/pi-agent-core` in `.trellis/config.yaml`
  matches no `packages:` key, so `get_packages_info` never flags a default
  (`packages_context.py:113`).
- `.agents/skills/trellis-spec-bootstrap/SKILL.md` explicitly authorizes
  deleting, renaming, and splitting template files.

### Convention sources to import from

`AGENTS.md` (root, authoritative global rules), `CONTRIBUTING.md`, `biome.json`,
`tsconfig.base.json`, `tsconfig.json`, `vitest.base.ts`, `test.sh`,
`scripts/check-*.mjs`, `.husky/`, `.github/workflows/{ci,pr-gate}.yml`,
`packages/agent/docs/*.md` (7 files), `packages/coding-agent/docs/*.md`
(31 files), `.pi/skills/add-llm-provider.md`, per-package `README.md`.

## Key Decisions

- **K1** Reshape the spec tree to real code boundaries; drop the
  `backend`/`frontend` split. (Developer-approved.)
- **K2** Cover the 7 product packages only; delete spec directories for the 5
  sample extensions and the empty `storage` container, and prune the matching
  `config.yaml` entries. (Developer-approved.)
- **K3** Add a repo-level `_shared` layer for cross-package rules, linked from
  `guides/index.md`.
- **K4** `AGENTS.md` stays the single source of truth. `_shared` restates a rule
  only when it adds repository evidence (enforcing script, config file, proving
  test, real path); process-only rules are linked, not copied.
- **K5** Fix `default_package` to `pi-agent-core`.
- **K6** Clean `.trellis/spec/guides/` (scope change approved mid-execution). The
  planning assumption that these guides were repo-agnostic was wrong: they carry
  ~200 lines of upstream Trellis product content that does not exist in pi
  (`src/templates/trellis/index.ts`, `packages/cli/src/templates/trellis/scripts/`,
  `docs-site/beta/**`, `.trellis/workflow.md` parser rules) plus three sections
  duplicated verbatim in `cross-layer-thinking-guide.md` (lines 126/223,
  141/238, 197/267). Because `trellis-before-dev` step 6 and
  `get_context_packages_text` make `guides/` always-on, that content reaches
  every sub-agent. Action: delete non-applicable and duplicated sections, keep
  the generic thinking frameworks, and re-anchor portable checklists to real pi
  constructs.

Full layer map, document contract, and rationale: `design.md`.
Execution order and validation commands: `implement.md`.

## Requirements

- **R1** Every rule in `.trellis/spec/` is traceable to a real file, symbol, or
  repeated pattern in this repository. No generic framework advice.
- **R2** No template placeholder text remains anywhere under `.trellis/spec/`.
- **R3** Each layer directory has an `index.md` containing a
  `Pre-Development Checklist` section and a Guidelines Index listing exactly the
  files present in that directory.
- **R4** Global rules are not forked into per-package copies; package layers
  link to `_shared` instead of restating it.
- **R5** `.trellis/config.yaml` and `get_context.py --mode packages` stay
  coherent with the final spec tree (7 packages, real layer names, working
  `default_package`).
- **R6** Documented behavior is current behavior. Known tech debt is recorded as
  debt, not as an aspiration.
- **R7** Only `.trellis/**` is modified. No product source, test, build, or CI
  file changes.
- **R8** Spec documents are written in English.

## Acceptance Criteria

- [x] AC1 `grep -RIn "To be filled\|TODO: fill\|placeholder" .trellis/spec`
      returns no matches. (R2)
- [x] AC2 `python3 ./.trellis/scripts/get_context.py --mode packages` lists
      exactly the 7 product packages, each with its designed layers, and marks
      `pi-agent-core` as default. (R5, K5)
- [x] AC3 Every filesystem path cited in a spec file exists (link check at
      `implement.md` Step 5). (R1) Three accepted exceptions, each explicit in
      the citing text: `packages/observability` (documented as a non-existent
      proposal), `packages/ai/src/providers/data` (git-ignored, generated), and
      `test/x.test.ts` (a command-template filename).
- [x] AC3b `.trellis/spec/guides/` contains no reference to paths outside this
      repository and no section duplicated verbatim. (K6)
- [x] AC4 Every layer `index.md` has a `Pre-Development Checklist` and a
      Guidelines Index matching the directory contents. (R3)
- [x] AC5 Each of the 7 packages has at least one spec file citing source or
      test files from that package. (R1) Distinct in-package citations: agent
      35, ai 33, coding-agent 57, evals 17, server 15, sqlite-node 6, tui 27.
- [x] AC6 `git status --short` shows no modification under `packages/`,
      `scripts/`, `.github/`, or root config files. (R7)
- [x] AC7 `python3 ./.trellis/scripts/task.py validate 00-bootstrap-guidelines`
      passes. (R5)
- [x] AC8 An independent `trellis-check` pass reports no unresolved
      CRITICAL/WARNING finding against these criteria. Zero CRITICAL; all 11
      WARNINGs were verified against source and fixed, as were the material
      INFO items.

## Package Checklist

- [x] `_shared` (typescript-and-style, testing, checks-and-commands,
      dependencies-and-git, index)
- [x] `guides/index.md` pointer section
- [x] `guides/` pruned per K6 (upstream Trellis sections removed, duplicates
      removed, portable checklists re-anchored to pi constructs)
- [x] pi-agent-core (`agent-loop`, `harness`)
- [x] pi-ai (`core`, `providers`)
- [x] pi-coding-agent (`core`, `modes`, `extensions`)
- [x] pi-evals (`evals`)
- [x] pi-server (`server`)
- [x] pi-storage-sqlite-node (`storage`)
- [x] pi-tui (`rendering`, `components`)
- [x] `config.yaml` pruned, `default_package` fixed
- [x] Deleted: 5 `pi-extension-*` spec dirs + `storage` spec dir

Final tree: 37 markdown files (3 `guides`, 5 `_shared`, 29 package-layer),
replacing the original 136 placeholder files.

## Out Of Scope

- Changing product source code, tests, build config, or CI.
- Rewriting `AGENTS.md` / `CONTRIBUTING.md` content.
- Rewriting the generic thinking frameworks in `.trellis/spec/guides/` from
  scratch. Per K6 the guides are pruned (delete upstream-only and duplicated
  sections, re-anchor portable checklists), not rewritten.
- Introducing new lint rules or CI checks.
- Spec coverage for the 5 sample extension packages and the empty `storage`
  container.

## Completion Procedure

```bash
python3 ./.trellis/scripts/task.py finish
python3 ./.trellis/scripts/task.py archive 00-bootstrap-guidelines
```

`.trellis/` is currently untracked in git. Whether to commit it is decided with
the developer at `implement.md` Step 6, before any git command runs.
