# Implementation Plan — Sync spec with upstream merge

Ordered so each step ends at a commit boundary from `design.md` D5.

## Step 0 — Baseline

- [ ] `git status` shows only `.trellis/` changes; nothing under `packages/`.
- [ ] Confirm HEAD is still `2975d1c9` (the sync baseline). If upstream moved,
      re-run `git diff 53340c77 HEAD -- packages/ --stat` and update PRD F1a
      before proceeding.

## Step 1 — Commit the already-written shared rules

Files: `.trellis/spec/_shared/index.md`,
`.trellis/spec/_shared/dependencies-and-git.md` (both currently uncommitted).

- [ ] Re-read both diffs; confirm English-only and no unrelated edits.
- [ ] Commit: `docs(spec): add language and atomic-commit rules`.

## Step 2 — Anchor repair (R1 → AC1)

Work one doc at a time; do not batch across files. For each anchor: recover the
intended symbol from the prose → `grep -n` it → rewrite the number →
`sed -n '<N>p'` to confirm.

- [ ] `pi-agent-core/harness/session-and-storage.md` (25 anchors; PRD F1 lists
      7 known-bad in `agent-harness.ts`)
- [ ] `pi-agent-core/harness/tools-and-compaction.md` (34 anchors; only the
      `agent-harness.ts` ones can be stale)
- [ ] `pi-coding-agent/extensions/extension-api.md` (14 anchors; 11 known-bad)
- [ ] `pi-coding-agent/modes/interactive-tui.md` (10 anchors; 6 known-bad)
- [ ] `pi-coding-agent/core/session-and-config.md`, `core/tools.md`,
      `modes/rpc-and-print.md` (only anchors into merge-touched files)
- [ ] `pi-tui/rendering/terminal-and-width.md` (20 anchors; 6 known-bad)
- [ ] `pi-tui/rendering/index.md` (1 anchor: `utils.ts:167` → `173`)
- [ ] `pi-tui/components/component-model.md` (anchors into
      `components/markdown.ts` / `src/index.ts` only)
- [ ] `pi-ai/core/types-and-compat.md`, `core/model-catalog.md`,
      `providers/adding-a-provider.md` (3 `types.ts` anchors verified accurate;
      re-check the `generate-models.ts` and `api/*` ones)
- [ ] Anchors whose symbol no longer exists: do not guess — record them and
      handle in Step 3 as stale prose.
- [ ] Commit: `docs(spec): repair anchors drifted by the upstream merge`.

## Step 3 — New API documentation (R2 → AC2–AC6)

Placement is fixed by `design.md` D2.

- [ ] `pi-agent-core/harness/session-and-storage.md`: replace the
      `runPromise` / `runAbortController` lifecycle prose with the
      `activeTasks` / `TrackedTaskKind` / `startOperation` / `waitForTasks` /
      `isShutdown` / `assertNotShutDown` model. Read
      `packages/agent/src/harness/agent-harness.ts` and
      `packages/agent/test/harness/agent-harness.test.ts` (237 new lines) for
      the guaranteed behavior.
- [ ] `pi-coding-agent/extensions/extension-api.md`: add the
      `registerMarkdownTransformer` subsection (`MarkdownTransformer`,
      `MarkdownTransformContext`, `Extension.markdownTransformer`), and
      cross-link `MarkdownOptions.transform` in `pi-tui`.
- [ ] `pi-coding-agent/extensions/extension-api.md`: update resource discovery
      for `core/pi-manifest.ts` (`PiManifest`, `readPiManifest`) and the
      malformed-manifest behavior from
      `test/suite/regressions/7187-malformed-package-manifest.test.ts`.
- [ ] `pi-tui/rendering/terminal-and-width.md`: document the spacing-mark rules
      in `graphemeWidth` (`terminalSpacingMarkRegex`, `markCharRegex`,
      `nonPrintingCharRegex`) and name the covering test
      (`packages/tui/test/truncate-to-width.test.ts`).
- [ ] `pi-tui/components/component-model.md`: new "Markdown component"
      subsection — `MarkdownOptions.transform` semantics (runs before parsing,
      receives `contentWidth`), the `Marked` / `Token` / `Tokens` re-exports
      from `packages/tui/src/index.ts`, and the back-link to
      `registerMarkdownTransformer`.
- [ ] `pi-ai/core/types-and-compat.md` + `pi-ai/providers/adding-a-provider.md`:
      Bedrock structured error metadata
      (`packages/ai/test/bedrock-error-metadata.test.ts` is the contract) and
      openai-completions `tool_choice`
      (`packages/ai/test/openai-completions-tool-choice.test.ts`).
- [ ] Commit: `docs(spec): document APIs added by the upstream merge`.

## Step 4 — `pi-protocol` layer (R3 → AC8)

Create `.trellis/spec/pi-protocol/protocol/` with the three files from D3.
Mirror the tone and density of
`.trellis/spec/pi-storage-sqlite-node/storage/index.md`.

- [ ] `index.md` — scope over all 9 source files, pre-development checklist,
      guidelines index, shared-rules links, and the F5 "no consumers yet"
      statement.
- [ ] `wire-format.md` — CBOR encoder/decoder/options, `encodeFrame`,
      `assertCompleteFrame`, `FrameDecoder`, `DEFAULT_MAX_FRAME_LENGTH`,
      `FrameError`, `PAYLOAD_BLOCK_SIZE`, the `MAX_UINT32` bound. Tests:
      `test/cbor/cbor.test.ts`, `test/framing.test.ts`.
- [ ] `messages-and-codec.md` — `PROTOCOL_VERSION = 2`, `StrictObject`
      (`additionalProperties: false`), `SessionPhaseSchema` ↔
      `AgentHarnessPhase` coupling, `JsonValue` cyclic schema,
      `isProtocolValue` prototype/cycle guard running before typebox `Check`,
      `parseClientMessage` / `parseServerMessage`, `ProtocolValidationError`,
      the 500-char bounded error message. Test: `test/protocol.test.ts`.
- [ ] Commit: `docs(spec): add pi-protocol layer`.

## Step 5 — Registration (R3 → AC7, AC9)

- [ ] `.trellis/config.yaml`: add `pi-protocol: path: packages/protocol`,
      keeping the existing alphabetical-ish ordering and the exclusion comment
      intact.
- [ ] `_shared/index.md`: Scope workspace list + Package Layers table row
      (`pi-protocol (packages/protocol) | protocol`).
- [ ] `_shared/testing.md`: config table row for
      `packages/protocol/vitest.config.ts` (globals on, node env, dot reporter,
      no `include` narrowing — unlike `packages/tui`).
- [ ] Commit: `chore(trellis): register pi-protocol in config and shared spec`.

## Validation

```bash
# AC7 — package + layer visible to tooling
python3 ./.trellis/scripts/get_context.py --mode packages

# AC1 / AC8 — spot-check anchors (repeat per file:line pair)
sed -n '181p' packages/agent/src/harness/agent-harness.ts
sed -n '173p' packages/tui/src/utils.ts

# AC2 — no removed API left in the spec
grep -rn "runPromise\|runAbortController" .trellis/spec/

# AC10 — English-only artifacts
grep -rlP '[\x{4e00}-\x{9fff}]' .trellis/spec .trellis/tasks/07-31-01-sync-spec-upstream

# every referenced path still exists
grep -rhoE 'packages/[A-Za-z0-9_./-]+\.(ts|json|md)' .trellis/spec \
  | sort -u | while read -r p; do [ -e "$p" ] || echo "MISSING: $p"; done
```

`npm run check` is not required: no `.ts` file is touched.

## Risky Files / Rollback

| File | Risk | Rollback |
|---|---|---|
| `.trellis/config.yaml` | A malformed `packages:` block breaks every session's context load | Commit 5 alone; `git revert` restores it without touching spec prose |
| `_shared/index.md` | Edited in commits 1 and 5 for different reasons | Keep the two edits in disjoint sections (Language Rule vs Scope/Package Layers) so reverts do not conflict |
| `pi-agent-core/harness/session-and-storage.md` | Largest prose rewrite; risk of describing behavior the tests do not guarantee | Anchor every claim to `agent-harness.test.ts`; drop any claim without test or code evidence |

## Follow-ups (not this task)

- Anchor-drift verification script (PRD Out of Scope).
- Re-run this sync procedure after the next upstream merge; D5's commit slicing
  is the reusable shape.
