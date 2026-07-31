# Design: `.trellis/spec/` Reshape For The pi Monorepo

## 1. Decision Summary

| # | Decision | Rationale / evidence |
|---|---|---|
| D1 | Delete the generated `backend` / `frontend` layer split and replace it with layers named after real code boundaries | No web frontend, no React, no ORM in this repo; `_scan_spec_layers` (`.trellis/scripts/common/packages_context.py:30`) derives layers from directory names, so any layout is valid |
| D2 | Cover only the 7 product packages | The 5 `pi-extension-*` entries are samples under `packages/coding-agent/examples/extensions/`, `storage` has no `src` |
| D3 | Add a repo-level `_shared` layer | Global rules (TS style, tests, checks, deps/git) apply to every package; 13 copies would drift |
| D4 | `AGENTS.md` stays the authority; `_shared` adds evidence, never contradicts | `CONTRIBUTING.md` tells contributors to run agents from repo root so `AGENTS.md` is picked up automatically |
| D5 | Link `_shared/index.md` from `guides/index.md` | `get_context_packages_text` only advertises `guides/` (`packages_context.py:206-211`); `trellis-before-dev` step 6 always reads `guides/index.md`, so this is the only always-on discovery path |
| D6 | Prune `config.yaml` `packages:` to the 7 covered packages | Keeps `get_context.py --mode packages` consistent with the spec tree |
| D7 | Fix `default_package` to `pi-agent-core` | `get_packages_info` compares `pkg_name == default_pkg` (`packages_context.py:113`); the generated value `@earendil-works/pi-agent-core` never matches any key, so no package is ever flagged default |

## 2. Target Spec Tree

```
.trellis/spec/
  guides/                      # existing content kept; index.md gains a link to _shared
    index.md                   # + "Project Rules" pointer section
    code-reuse-thinking-guide.md
    cross-layer-thinking-guide.md
  _shared/
    index.md                   # entry point + pre-development checklist
    typescript-and-style.md    # no any, no inline imports, erasable syntax, biome
    testing.md                 # ./test.sh, vitest configs, suite/harness, regressions
    checks-and-commands.md     # what each npm run check sub-check enforces
    dependencies-and-git.md    # pinned deps, lockfiles, shrinkwrap, commit rules
  pi-agent-core/
    agent-loop/     index.md, streaming-and-tools.md
    harness/        index.md, session-and-storage.md, tools-and-compaction.md
  pi-ai/
    core/           index.md, model-catalog.md, types-and-compat.md
    providers/      index.md, adding-a-provider.md
  pi-coding-agent/
    core/           index.md, session-and-config.md, tools.md
    modes/          index.md, interactive-tui.md, rpc-and-print.md
    extensions/     index.md, extension-api.md
  pi-evals/
    evals/          index.md, writing-evals.md
  pi-server/
    server/         index.md, ipc-and-supervisor.md
  pi-storage-sqlite-node/
    storage/        index.md, schema-and-migrations.md
  pi-tui/
    rendering/      index.md, terminal-and-width.md
    components/     index.md, component-model.md, keybindings.md
```

~34 files replace the current 136 placeholder files. The implementer may split,
merge, or rename a topic file when the source evidence demands it, provided the
owning `index.md` is updated in the same change.

Removed entirely: `.trellis/spec/pi-extension-with-deps/`,
`pi-extension-custom-provider-anthropic/`, `pi-extension-custom-provider-gitlab-duo/`,
`pi-extension-sandbox/`, `pi-extension-gondolin/`, `storage/`.

## 3. Layer Decomposition Evidence

| Layer | Source scope | Why it is its own layer |
|---|---|---|
| `pi-agent-core/agent-loop` | `packages/agent/src/{agent-loop,agent,stream-fn,proxy,types}.ts` | Provider-agnostic loop primitives; docs `packages/agent/docs/models.md`, `observability.md` |
| `pi-agent-core/harness` | `packages/agent/src/harness/**` (session, tools, compaction, prompt-templates, skills) | Separate public surface documented in `packages/agent/docs/{harness,harness-v2,agent-harness,durable-harness,hooks}.md` |
| `pi-ai/core` | `src/{types,models,models-store,model-catalog}.ts`, `src/api/`, `src/auth/`, `src/compat/`, `*.generated.ts` | Catalog generation rules are strict (`AGENTS.md`: never edit `models.generated.ts`, edit `scripts/generate-models.ts`) |
| `pi-ai/providers` | `src/providers/*.ts` (~36 providers, each with a `.models.ts` sibling, registered in `providers/all.ts`) | Highly repetitive add-a-provider pattern; existing checklist at `.pi/skills/add-llm-provider.md` |
| `pi-coding-agent/core` | `src/core/**` (+ `config.ts`, `migrations.ts`, `utils/`) | Session, config, tools, compaction, export — largest surface (~55.6k LOC package) |
| `pi-coding-agent/modes` | `src/modes/interactive/**`, `src/modes/rpc/**`, `src/cli/`, `src/main.ts` | TUI/RPC/print entry points with their own testing rules (`test/suite/harness.ts`, tmux workflow in `AGENTS.md`) |
| `pi-coding-agent/extensions` | `src/extensions/**`, `src/core/extensions/**`, `examples/extensions/**`, `docs/extensions.md` | `CONTRIBUTING.md` mandates extension-first design; examples are the reference material |
| `pi-evals/evals` | `packages/evals/src/**` (`vitest-evals/`, `*.eval.ts`, `pi-harness.ts`) | Single small surface |
| `pi-server/server` | `packages/server/src/**` (`ipc/`, `supervisor.ts`, `serve.ts`, `handler.ts`) | Single small surface |
| `pi-storage-sqlite-node/storage` | `src/sqlite/**` incl. `migrations/001_initial.sql`, `repo.ts`, `storage/*` | Only place in the repo with SQL and migrations |
| `pi-tui/rendering` | `tui.ts`, `TuiMainScreen.ts`, `TuiAltScreen.ts`, `terminal*.ts`, `utils.ts`, `native/` | Differential rendering + width/CJK/emoji correctness; regression tests in `test/` prove the rules |
| `pi-tui/components` | `src/components/*`, `editor-component.ts`, `autocomplete.ts`, `keys.ts`, `keybindings.ts`, `stdin-buffer.ts` | Component/keybinding conventions; `AGENTS.md` forbids hardcoded key checks |

## 4. Document Contract

Every `index.md` (layer entry point) has exactly these sections:

1. `# <Layer> Guidelines` + one-line scope statement with the source paths it governs.
2. `## Pre-Development Checklist` — ordered, actionable items (`trellis-before-dev`
   step 4 reads this section by name).
3. `## Guidelines Index` — table of the topic files in the same directory.
4. `## Shared Rules` — pointer to `.trellis/spec/_shared/index.md` and
   `.trellis/spec/guides/index.md`.

Every topic file has:

1. `## When This Applies`
2. `## The Local Pattern` — with real `path:symbol` references
3. `## Reference Files` — real paths only (source + test)
4. `## Anti-Patterns` — what the repo actually avoids, with the enforcing check
   or reviewer rule where one exists

Hard content rules:

- Every claim is traceable to a file in this repo. No generic framework advice.
- Prefer path + symbol references over long code blocks; snippets stay under
  ~15 lines and are copied verbatim from the source.
- Documented behavior is current behavior. Known debt is described as debt, not
  as an aspiration.
- English only.

## 5. Shared-Rules Strategy (anti-duplication)

`_shared` restates a rule only when it can add repository evidence that
`AGENTS.md` does not carry — the enforcing script, the config file, the test
that proves it, or the real file path. Example: `AGENTS.md` says deps stay
pinned; `_shared/dependencies-and-git.md` names `scripts/check-pinned-deps.mjs`
as the enforcing check and `npm run check:pinned-deps` as the local command.

Process-only rules (release flow, issue/PR etiquette, GitHub labels) are not
copied; `_shared/index.md` points to `AGENTS.md` and `CONTRIBUTING.md` for them.

Package layers never restate a `_shared` rule; they link to it.

## 6. Config Changes

`.trellis/config.yaml`:

- `packages:` keeps `pi-agent-core`, `pi-ai`, `pi-coding-agent`, `pi-evals`,
  `pi-server`, `pi-storage-sqlite-node`, `pi-tui`; drops `storage` and the five
  `pi-extension-*` entries.
- `default_package: pi-agent-core` (was `@earendil-works/pi-agent-core`, which
  matches no key).

No product source, build config, or CI file is touched.

## 7. Verification

```bash
grep -RIn "To be filled\|TODO: fill\|placeholder" .trellis/spec   # must be empty
python3 ./.trellis/scripts/get_context.py --mode packages          # 7 packages, real layers
python3 ./.trellis/scripts/task.py validate 00-bootstrap-guidelines
```

Link check: every path referenced by a spec file must exist. Run
`node /tmp/spec-link-check.mjs` (ad-hoc script written during implementation,
removed afterwards) or an equivalent inline check that extracts
`packages/...`, `scripts/...`, `.trellis/...` tokens from the spec markdown and
tests each with `test -e`.

## 8. Risks

| Risk | Mitigation |
|---|---|
| Spec drifts from `AGENTS.md` over time | `_shared` links instead of copying wherever no evidence is added |
| Sub-agents writing different houses styles across packages | `_shared` is written first and acts as the style reference; a final consistency pass reviews all files |
| `trellis update` may re-create template layer dirs | Verified only as a possibility, not observed; re-run the verification commands after any `trellis update` |
| Over-long specs dilute signal | Each topic file targets 60-150 lines (soft target) |

Accepted length exceptions (developer-approved during execution): the 150-line
target is soft. Four topic files exceed it because the overflow is verified
evidence — the `Anti-Patterns` table required by §4 plus, where applicable, a
`Known Debt` section required by R6. Compressing them would delete facts, not
prose.

| File | Lines | Reason |
|---|---|---|
| `pi-storage-sqlite-node/storage/schema-and-migrations.md` | 170 | Anti-Patterns table + 6-item Known Debt list |
| `pi-agent-core/agent-loop/streaming-and-tools.md` | 163 | 9-row Anti-Patterns table with per-row evidence |
| `pi-server/server/ipc-and-supervisor.md` | 159 | 8-row Anti-Patterns table + Known Debt paragraph |
| `pi-agent-core/harness/tools-and-compaction.md` | 153 | 8-row Anti-Patterns table |
