# Implement: interactive models configuration

Workspace: `/home/heixiaohu/pi-preset`. Do not start this child until `08-01-integrate-grill-me` is completed and archived.

## Phase 1 — Templates and pure config logic

- [x] 1.1 Add `src/model-templates.ts` with exact current credential-free Anthropic, OpenAI, and DeepSeek templates, including `gpt-5.6-luna` and the revised OpenAI parameters/tiers.
- [x] 1.2 Add provider-id, URL, and API-key validators with secret-free errors.
- [x] 1.3 Add Pi-compatible comment/trailing-comma parsing and models-root normalization.
- [x] 1.4 Add selected-model normalization plus semantic plan states: add, replace, already configured, and blocked.
- [x] 1.5 Add recursive role-aware redaction and deterministic provider-only diff rendering.

## Phase 2 — Safe apply and shared writer

- [x] 2.1 Add `getModelsPath()` using the existing `getAgentDir()` resolver.
- [x] 2.2 Extend `writeJsonObjectAtomic` with optional new-file mode, dangling-symlink rejection, unique sibling tmp names, and explicit backup/tmp chmod while preserving existing callers' semantics.
- [x] 2.3 Add permission preflight: owner-only modes pass; group/other bits block.
- [x] 2.4 Add write-time re-read and target-provider baseline comparison.
- [x] 2.5 Upsert only the target provider while preserving latest sibling/top-level data.
- [x] 2.6 Wrap command apply with `withFileMutationQueue()`.

## Phase 3 — TUI workflow

- [x] 3.1 Add the family-model checklist with all rows initially unchecked, toggle rows, a disabled-until-nonempty `Continue` row, and catalog-order output.
- [x] 3.2 Add the masked secret component; never call ordinary input render for the key.
- [x] 3.3 Add a width-safe scrollable diff confirmation component using injected configurable keybindings.
- [x] 3.4 Add `extensions/preset-models-add.ts` with TUI-only mode guard and the ordered family/models/id/URL/preflight/key/preview/collision/apply flow.
- [x] 3.5 Report already-configured without writing; after success instruct the user to open `/model` and state that no restart is required.

## Phase 4 — Secret scan and documentation

- [x] 4.1 Refine `scripts/scan-secrets.sh` so credential property names are legal but high-confidence and value-shaped credential literals remain blocked in working tree and history.
- [x] 4.2 Extend README with command usage, supported bundles, fixed API modes, permission behavior, redacted replacement flow, backup path, hot reload, and out-of-scope providers.
- [x] 4.3 Include no real endpoint, provider identifier, or credential in code/docs/tests.

## Phase 5 — Automated tests

- [x] 5.1 Template snapshots for all three families, including Luna, revised OpenAI costs/tiers/context/thinking maps, modalities, and limits.
- [x] 5.2 Validation tests for identifiers, reserved keys, URLs, and empty keys.
- [x] 5.3 Selection/plan/redaction tests for initially empty state, empty continuation blocking, individual models, multi-model catalog ordering, add, replacement, key-only changes, exact no-op, and nested secret fields.
- [x] 5.4 Apply tests for sibling/top-level preservation, replacement-as-unit, commented input, invalid JSON, missing/non-object providers, target conflict, and no-op mtime.
- [x] 5.5 Writer tests for new `0600`, existing `0600`/`0400`, broader-mode rejection, valid symlink preservation, dangling symlink rejection, backup bytes/mode, and tmp cleanup.
- [x] 5.6 Workflow tests with injected UI for cancellation at every stage and second-confirm rejection.
- [x] 5.7 Run all created Node test files and iterate until they pass.

## Phase 6 — Pi sandbox/TUI verification

- [x] 6.1 Run RPC/print/JSON invocations and prove no unmasked prompt or write occurs.
- [x] 6.2 Run TUI sandboxes covering single-model and multi-model selections in each family using a runtime-generated key pasted through a tmux buffer.
- [x] 6.3 Assert the key is absent from tmux capture, notifications, test output, and `PI_TUI_WRITE_LOG`.
- [x] 6.4 Verify resulting file content, mode, backup, cancellation mtimes, and collision behavior.
- [x] 6.5 Run `PI_OFFLINE=1 pi --list-models` and verify every configured model ID; do not send a model prompt.
- [x] 6.6 Run the existing Pi hot-reload regression and existing preset sync smoke checks.
- [x] 6.7 Run `scripts/scan-secrets.sh` and Pi monorepo `npm run check`.

The preset tests required the declared Pi peer packages to be installed locally; validation used `npm install --ignore-scripts --no-package-lock` and left package metadata unchanged. `npm run check` was hydrated and run in full. It still fails only on the same 15 pre-existing stale ZAI model references (`glm-4.5-air` and `glm-5.1`) accepted by the user for this task.

## Validation commands

```bash
cd /home/heixiaohu/pi-preset
node --test \
  test/model-templates.test.ts \
  test/models-config.test.ts \
  test/models-config-apply.test.ts \
  test/models-wizard-workflow.test.ts \
  test/json-merge.test.ts
./scripts/scan-secrets.sh

cd /home/heixiaohu/桌面/pi/packages/coding-agent
node ../../node_modules/vitest/dist/cli.js \
  --run test/suite/regressions/6999-models-json-hot-reload.test.ts

cd /home/heixiaohu/桌面/pi
npm run check
```

## High-risk files and rollback points

| File/path | Risk | Rollback |
|---|---|---|
| `~/.pi/agent/models.json` | Holds literal credentials | `.preset-bak`, owner-only mode gate, atomic rename |
| `src/json-merge.ts` | Shared by existing `/preset-sync` | options are opt-in; run existing sync regression before commit |
| `scripts/scan-secrets.sh` | False negatives could publish credentials | test allowed schema names and blocked literal values; run full history scan |
| masked/diff UI | Secret could reach terminal output | fixed-length-independent mask plus tmux and raw ANSI log assertions |

## Pre-start gate

- [x] The grill-me child is completed.
- [x] The latest PRD/design/implement summary has explicit user approval.
- [x] `implement.jsonl` and `check.jsonl` contain real spec/research entries.
- [x] `/home/heixiaohu/pi-preset` has no unrelated uncommitted changes.
