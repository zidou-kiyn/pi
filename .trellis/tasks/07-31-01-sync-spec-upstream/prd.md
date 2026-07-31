# Sync spec with upstream merge

## Goal

Bring `.trellis/spec/` back in sync with the code as of
`2975d1c9 Merge remote-tracking branch 'upstream/main'`, so that a later session
reading the spec gets anchors, a package list, and API descriptions it can
verify against the working tree.

## Background

The `.trellis/spec/` tree was written in `53340c77`; its baseline is the code at
that moment. `2975d1c9` then merged upstream, touching 56 files
(+3124 / -155) under `packages/`, including one brand-new package. The spec was
never updated.

All findings below come from `git diff 53340c77 HEAD -- packages/` plus a
line-by-line re-check of spec anchors against both the spec-authoring commit and
the current tree.

## Confirmed Facts

### F1 — Line anchors have drifted

32 anchors were sampled: 29 drifted, 3 are still accurate (all in
`packages/ai/src/types.ts`). Every anchor pointed at the right symbol when the
spec was written, so the drift is entirely merge-induced.

Confirmed drift (spec location → intended symbol → what that line holds today):

| Spec file | Anchor | Intended symbol | Current line |
|---|---|---|---|
| `pi-coding-agent/extensions/extension-api.md:26` | `package-manager.ts:901` | `async resolve(...)` | `await this.resolvePackageSources(...)` |
| `pi-coding-agent/extensions/extension-api.md:27` | `package-manager.ts:2309` | `addAutoDiscoveredResources(` | `};` |
| `pi-coding-agent/extensions/extension-api.md:27` | `package-manager.ts:950` | `this.addAutoDiscoveredResources(...)` | `listConfiguredPackages()` |
| `pi-coding-agent/extensions/extension-api.md:30` | `package-manager.ts:2374` | `if (projectTrusted) {` | `projectOverrides.skills,` |
| `pi-coding-agent/extensions/extension-api.md:31` | `package-manager.ts:2427` | `// User extensions from ~/.pi/agent/` | blank line |
| `pi-coding-agent/extensions/extension-api.md:13` | `loader.ts:678` | `discoverAndLoadExtensions(` | `const resolved = path.resolve(p);` |
| `pi-coding-agent/extensions/extension-api.md:43` | `loader.ts:48-72` | `VIRTUAL_MODULES` | blank line |
| `pi-coding-agent/extensions/extension-api.md:48` | `loader.ts:84-140` | `getAliases()` | `let _aliases ...` |
| `pi-coding-agent/extensions/extension-api.md:53` | `runner.ts:927` | `emitToolCall(` | `isError: currentEvent.isError,` |
| `pi-coding-agent/extensions/extension-api.md:67` | `runner.ts:938-943` | blocking branch | offset |
| `pi-coding-agent/extensions/extension-api.md:74` | `types.ts:1238` | `registerTool<...>` | `on(event: "user_bash", ...)` |
| `pi-tui/rendering/index.md:17`, `terminal-and-width.md:15,130` | `utils.ts:167` | `graphemeWidth(` | blank line (now 173) |
| `pi-tui/rendering/terminal-and-width.md:16` | `utils.ts:216` | `visibleWidth(` | `if (terminalSpacingMarkRegex...)` |
| `pi-tui/rendering/terminal-and-width.md:17` | `utils.ts:1007` | `truncateToWidth(` | `*/` (now 1030) |
| `pi-tui/rendering/terminal-and-width.md:17` | `utils.ts:786` | `wrapTextWithAnsi(` | `} else {` |
| `pi-tui/rendering/terminal-and-width.md:26` | `utils.ts:355` | `normalizeTerminalOutput(` | `let textEnd = i;` |
| `pi-tui/rendering/terminal-and-width.md:40,133` | `utils.ts:382` | `extractAnsiCode(` | Thai/Lao constants |
| `pi-coding-agent/modes/interactive-tui.md:12` | `interactive-mode.ts:345` | `export class InteractiveMode` | blank line |
| `pi-coding-agent/modes/interactive-tui.md:17,56` | `interactive-mode.ts:362,494` | `keybindings` field / `KeybindingsManager.create()` | comment / `widgetContainerBelow` |
| `pi-coding-agent/modes/interactive-tui.md:40,41,42` | `interactive-mode.ts:477,1665,2876` | `setRebindSession` / `bindExtensions` / `subscribe` | all offset |
| `pi-agent-core/harness/session-and-storage.md:105-112` | `agent-harness.ts:179,891,910,937,966,512,538` | `phase` field and branches, `flushPendingSessionWrites`, `handleAgentEvent` | all offset (`phase` now 181, `flushPendingSessionWrites` now 554) |

The spec contains ~272 `file:line` anchors overall, and ~183 `packages/...`
path references. Every path still resolves to an existing file; only line
numbers are wrong.

### F1a — Blast radius is bounded by the merge diff

Only spec docs anchored into merge-touched source files can be stale. Source
files changed by the merge: `packages/agent/src/harness/agent-harness.ts`;
`packages/ai/src/{types.ts,api/bedrock-converse-stream.ts,api/openai-completions.ts,scripts/generate-models.ts}`;
`packages/coding-agent/src/{index.ts,core/extensions/{index,loader,runner,types}.ts,core/package-manager.ts,core/pi-manifest.ts,modes/interactive/{interactive-mode.ts,components/{assistant-message,user-message,markdown-transform}.ts}}`;
`packages/tui/src/{index.ts,utils.ts,components/markdown.ts}`.

Therefore in scope for anchor repair:

- `pi-agent-core/harness/{session-and-storage,tools-and-compaction}.md`
- `pi-ai/core/{types-and-compat,model-catalog}.md`, `pi-ai/providers/adding-a-provider.md`
- `pi-coding-agent/extensions/extension-api.md`, `pi-coding-agent/core/{session-and-config,tools}.md`,
  `pi-coding-agent/modes/{interactive-tui,rpc-and-print}.md`
- `pi-tui/rendering/{index,terminal-and-width}.md`, `pi-tui/components/{component-model,markdown}` anchors

Untouched by the merge, hence out of scope: `pi-storage-sqlite-node/**`,
`pi-server/**`, `pi-evals/**`, `pi-agent-core/agent-loop/**`,
`pi-tui/components/keybindings.md`.

### F2 — New package `packages/protocol` is not covered at all

`@earendil-works/pi-protocol` 0.83.0, "Transport-neutral CBOR protocol for
remote pi sessions": 9 source files
(`cbor/{encoder,decoder,options,index}.ts`, `codec.ts`, `framing.ts`,
`schemas.ts`, `index.ts`) plus 3 test files.

Affected spec and config:

- `.trellis/config.yaml` `packages:` has no `pi-protocol`, so
  `get_context.py --mode packages` cannot see it.
- `.trellis/spec/_shared/index.md` Scope hardcodes
  `packages/{agent,ai,coding-agent,evals,server,tui}` + `storage/sqlite-node`.
- `_shared/index.md` "Package Layers" table and "Not covered by spec, on
  purpose" paragraph both omit protocol.
- `_shared/testing.md:60-66` vitest config table omits
  `packages/protocol/vitest.config.ts`.
- Root `package.json` `build` / `build:offline` chains now include `protocol`
  (between `storage/sqlite-node` and `coding-agent`);
  `_shared/checks-and-commands.md` does not describe the build chain, so it is
  unaffected.

### F3 — New APIs and behavior are undocumented

| Change | Code location | Relevant spec |
|---|---|---|
| `registerMarkdownTransformer` + `MarkdownTransformer` / `MarkdownTransformContext`, `Extension.markdownTransformer` | `core/extensions/types.ts`, `core/extensions/index.ts`, `src/index.ts`, `modes/interactive/components/markdown-transform.ts` | `pi-coding-agent/extensions/extension-api.md` |
| `PiManifest` / `readPiManifest` extracted from `package-manager.ts` into new `core/pi-manifest.ts`, with manifest validation (regression test `7187-malformed-package-manifest.test.ts`) | `core/pi-manifest.ts`, `core/package-manager.ts` | `pi-coding-agent/extensions/extension-api.md` (resource discovery) |
| `AgentHarness` task-tracking rewrite: `activeTasks` / `TrackedTaskKind` / `startOperation` / `waitForTasks` / `isShutdown` / `assertNotShutDown` replace `runPromise` / `runAbortController` | `packages/agent/src/harness/agent-harness.ts` | `pi-agent-core/harness/session-and-storage.md` |
| `graphemeWidth` gains terminal spacing-mark rules (`terminalSpacingMarkRegex`, `markCharRegex`, `nonPrintingCharRegex`) | `packages/tui/src/utils.ts` | `pi-tui/rendering/terminal-and-width.md` |
| `packages/tui/src/index.ts` newly exports `Marked` / `Token` / `Tokens` | `packages/tui/src/index.ts` | `pi-tui/rendering/index.md` or `components/` |
| Bedrock structured error metadata, openai-completions `tool_choice` | `packages/ai/src/api/{bedrock-converse-stream,openai-completions}.ts`, `src/types.ts` | `pi-ai/core/types-and-compat.md`, `pi-ai/providers/` |

### F4 — Language rule

`.trellis/spec/_shared/index.md` now carries a Language Rule: Chinese only in
replies to the user; all on-disk artifacts (code, spec, task files, journals,
commit messages) in English. This PRD was rewritten under that rule.

### F5 — `packages/protocol` has no consumers yet

`tsconfig.json:19-20` maps `@earendil-works/pi-protocol`, but no other package
imports it. It is a standalone, transport-neutral library today; the spec layer
must say so rather than implying an active integration.

## Requirements

**R1 — Repair drifted anchors.** Every `file:line` anchor in the F1a spec docs
points at the symbol the surrounding prose names, verified against the current
tree. Anchor format stays `file.ts:NNN` / `file.ts:NN-MM` (unchanged style).

**R2 — Document the new APIs from F3.** Each F3 row lands in its listed spec
doc with the same evidence density as the surrounding text (symbol name,
anchor, and the behavior a developer must not break). Stale descriptions the
merge invalidated — most importantly the `AgentHarness` `runPromise` /
`runAbortController` model — are replaced, not appended to.

**R3 — Add a `pi-protocol` spec layer.** New `.trellis/spec/pi-protocol/`
covering all 9 source files, plus registration everywhere a package must be
registered:

- `.trellis/config.yaml` `packages:` gains `pi-protocol: path: packages/protocol`
- `_shared/index.md` Scope sentence and Package Layers table include protocol
- `_shared/testing.md` vitest config table lists `packages/protocol/vitest.config.ts`

**R4 — English-only artifacts.** All files this task writes are English, per
the Language Rule in `_shared/index.md`.

## Acceptance Criteria

- [ ] **AC1** — For every anchor in the F1a spec docs, the referenced line in
      the current tree contains the symbol the prose claims. Spot-verifiable
      with `sed -n '<N>p' <file>` for any anchor picked at random.
- [ ] **AC2** — No spec text still describes removed API: grep for
      `runPromise` and `runAbortController` in `.trellis/spec/` returns nothing
      except where the text explicitly marks them as replaced.
- [ ] **AC3** — `pi-coding-agent/extensions/extension-api.md` documents
      `registerMarkdownTransformer` (with `MarkdownTransformer` /
      `MarkdownTransformContext`) and the `core/pi-manifest.ts` extraction with
      its validation behavior.
- [ ] **AC4** — `pi-agent-core/harness/session-and-storage.md` describes the
      `activeTasks` / `TrackedTaskKind` / `startOperation` / `waitForTasks` /
      `isShutdown` / `assertNotShutDown` model.
- [ ] **AC5** — `pi-tui/rendering/terminal-and-width.md` documents the
      terminal spacing-mark rules in `graphemeWidth`, and the new
      `Marked` / `Token` / `Tokens` exports are recorded in a `pi-tui` doc.
- [ ] **AC6** — `pi-ai` docs cover Bedrock structured error metadata and
      openai-completions `tool_choice`.
- [ ] **AC7** — `python3 ./.trellis/scripts/get_context.py --mode packages`
      lists `pi-protocol` with its layer, and the printed spec files exist.
- [ ] **AC8** — `.trellis/spec/pi-protocol/**` anchors resolve to real
      symbols in `packages/protocol/src/**`, same spot-check as AC1.
- [ ] **AC9** — `_shared/index.md` Scope + Package Layers and
      `_shared/testing.md` vitest table all include protocol; no "not covered
      on purpose" claim contradicts the new layer.
- [ ] **AC10** — No Chinese prose in any file this task adds or edits:
      `grep -rnP '[\x{4e00}-\x{9fff}]' .trellis/spec .trellis/tasks/07-31-01-sync-spec-upstream`
      returns only the pre-existing CJK *width sample* at
      `pi-tui/rendering/terminal-and-width.md:55` (`让`, used as a width-2
      grapheme example). Sample characters inside technical examples are data,
      not prose, and are exempt from the Language Rule.

## Out of Scope

- Spec layers whose source files the merge did not touch (see F1a):
  `pi-storage-sqlite-node/**`, `pi-server/**`, `pi-evals/**`,
  `pi-agent-core/agent-loop/**`, `pi-tui/components/keybindings.md`.
- Any product code change: this task edits `.trellis/` only.
- An automated anchor-drift verification script (considered and deferred; it
  needs an "expected symbol" annotation on every anchor, which is a separate
  mechanism design).
- Documenting the root `package.json` build chain in
  `_shared/checks-and-commands.md` — that doc never covered the build chain.

## Open Questions

None.
