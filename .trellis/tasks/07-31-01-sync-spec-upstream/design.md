# Design — Sync spec with upstream merge

Scope of this document: how the spec edits are structured. No product code is
touched; the only artifacts are `.trellis/spec/**` and `.trellis/config.yaml`.

## D1 — Anchor repair method (R1)

Anchor style stays `path/file.ts:NNN` and `path/file.ts:NN-MM`. Rejected
alternatives and why:

| Alternative | Rejected because |
|---|---|
| Drop line numbers, keep symbol names only | Loses the "jump straight to evidence" property that makes this spec tree usable; the drift cost is one repair per upstream merge. |
| Replace anchors with a generated block | Requires a generator + annotation format; explicitly deferred (see PRD Out of Scope). |

Repair procedure per anchor, applied doc by doc:

1. Read the prose around the anchor to recover the *intended symbol*, not just
   the old number.
2. Locate it in the current tree with `grep -n '<symbol>' <file>`.
3. Rewrite the number. For a range anchor, re-derive both ends from the current
   block boundaries rather than shifting by a constant delta — the merge
   inserted code inside several of these blocks.
4. Verify with `sed -n '<N>p' <file>` before moving to the next anchor.

A constant-offset patch is not acceptable: `packages/tui/src/utils.ts` gained
lines at three separate places (spacing-mark regexes, `visibleWidth` branch,
`truncateToWidth` docblock), so the delta differs per anchor within one file.

## D2 — Where each new API is documented (R2)

Placement follows the existing layer ownership; no new doc files outside
`pi-protocol`.

| Change | Target doc | Shape of the edit |
|---|---|---|
| `AgentHarness` task tracking (`activeTasks`, `TrackedTaskKind`, `startOperation`, `waitForTasks`, `isShutdown`, `assertNotShutDown`) | `pi-agent-core/harness/session-and-storage.md` | Rewrite the lifecycle paragraph in place. The old `runPromise` / `runAbortController` description is deleted, not annotated. |
| `registerMarkdownTransformer`, `MarkdownTransformer`, `MarkdownTransformContext`, `Extension.markdownTransformer` | `pi-coding-agent/extensions/extension-api.md` | New subsection in the extension-surface section, next to the other `register*` hooks. |
| `PiManifest` / `readPiManifest` moved to `core/pi-manifest.ts` + manifest validation | `pi-coding-agent/extensions/extension-api.md` | Update the resource-discovery section: new file path, plus the malformed-manifest behavior proven by `test/suite/regressions/7187-malformed-package-manifest.test.ts`. |
| `graphemeWidth` terminal spacing-mark rules (`terminalSpacingMarkRegex`, `markCharRegex`, `nonPrintingCharRegex`) | `pi-tui/rendering/terminal-and-width.md` | Extend the width-table section; name the regression test that pins the behavior. |
| `MarkdownOptions.transform` + `Marked` / `Token` / `Tokens` re-exports | `pi-tui/components/component-model.md` | New short "Markdown component" subsection. `pi-tui` currently has zero markdown coverage, and `transform` is the TUI-side half of `registerMarkdownTransformer`. |
| Bedrock structured error metadata; openai-completions `tool_choice` | `pi-ai/core/types-and-compat.md` (type surface) and `pi-ai/providers/adding-a-provider.md` (provider-side obligation) | Add to the existing per-capability lists. |

**Cross-layer link.** `registerMarkdownTransformer` (coding-agent) reaches the
renderer through `MarkdownOptions.transform` (tui). Both docs must name the
other end, otherwise the next session changing one side will not find the
other. This is the only cross-package contract introduced by the merge.

## D3 — `pi-protocol` layer structure (R3)

One layer, `protocol`, matching the package's single concern and the
`pi-storage-sqlite-node` precedent (1 layer, small doc set) rather than the
`pi-tui` 2-layer split. Source is 1198 lines across 9 files.

```
.trellis/spec/pi-protocol/protocol/
├── index.md              # package scope, pre-dev checklist, guidelines index, shared-rules links
├── wire-format.md        # cbor/{encoder,decoder,options}.ts + framing.ts
└── messages-and-codec.md # schemas.ts + codec.ts
```

Split rationale: `wire-format.md` owns bytes on the wire (CBOR encoding rules,
`DEFAULT_MAX_FRAME_LENGTH`, the 4-byte big-endian length prefix, `FrameDecoder`
streaming state, `FrameError`). `messages-and-codec.md` owns the message
contract (`PROTOCOL_VERSION = 2`, typebox `StrictObject` with
`additionalProperties: false`, `ClientMessage` / `ServerMessage`,
`parseClientMessage` / `parseServerMessage`, `isProtocolValue` prototype
guard, `ProtocolValidationError`, bounded error messages). A reader changing a
message shape never needs the CBOR doc, and vice versa.

Facts the layer must carry, because they are non-obvious and easy to break:

- `SessionPhaseSchema` deliberately mirrors `AgentHarnessPhase`
  (`packages/protocol/src/schemas.ts` comment says so) — changing one without
  the other splits the phase vocabulary.
- `PROTOCOL_VERSION` is a wire contract; bumping it is a compatibility event.
- `isProtocolValue` rejects non-plain-object prototypes and cycles *before*
  typebox `Check` runs; validation is not typebox alone.
- The package has no consumers yet (`tsconfig.json:19-20` maps the alias, no
  `src` import exists). The layer states this so nobody hunts for integrations.
- Test entry: `packages/protocol/vitest.config.ts` exists and `npm test` in the
  package runs `vitest --run`, but `_shared/testing.md`'s "never run the raw
  vitest suite" rule still governs; run by explicit file from the package root.

## D4 — Registration surface (R3)

A package is "registered" in exactly four places; missing one is what made this
task necessary:

1. `.trellis/config.yaml` → `packages.pi-protocol.path: packages/protocol`
   (drives `get_context.py --mode packages` and the session overview).
2. `_shared/index.md` → Scope sentence workspace list.
3. `_shared/index.md` → Package Layers table row.
4. `_shared/testing.md` → vitest config table row for
   `packages/protocol/vitest.config.ts`.

The "Not covered by spec, on purpose" paragraph in `_shared/index.md` keeps its
two existing entries and must not be extended — protocol is now covered.

## D5 — Commit slicing

Per `_shared/dependencies-and-git.md` "Atomic commits, sliced for review", this
task lands as five commits in dependency order, each one reviewable alone:

| # | Commit | Contents |
|---|---|---|
| 1 | `docs(spec): add language and atomic-commit rules` | `_shared/index.md` Language Rule, `_shared/dependencies-and-git.md` atomic commits (already written, uncommitted) |
| 2 | `docs(spec): repair anchors drifted by the upstream merge` | R1 only — pure line-number corrections, no prose change |
| 3 | `docs(spec): document APIs added by the upstream merge` | R2 across agent/coding-agent/tui/ai docs |
| 4 | `docs(spec): add pi-protocol layer` | New `.trellis/spec/pi-protocol/**` |
| 5 | `chore(trellis): register pi-protocol in config and shared spec` | `config.yaml` + `_shared/{index,testing}.md` registration rows |

Commits 2 and 3 stay separate on purpose: 2 is mechanically verifiable
(`sed -n` per anchor), 3 needs a semantic read. Mixing them would force the
reviewer to re-verify every number while reading new prose.

## D6 — Risks

| Risk | Mitigation |
|---|---|
| Repairing an anchor to the wrong symbol when the merge moved *and* renamed it | Always re-derive from the prose's intended symbol; if the symbol no longer exists, that is an R2 case (prose is stale), not an R1 case. |
| New `pi-protocol` docs drifting immediately on the next upstream merge | Keep anchors on stable exported symbols; prefer symbol-bearing lines (`export function ...`) over interior lines. |
| Spec claiming a protocol integration that does not exist | F5 is stated explicitly in `index.md`. |
