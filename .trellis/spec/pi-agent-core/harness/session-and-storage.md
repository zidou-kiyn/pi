# Session And Storage

## When This Applies

Any change to `packages/agent/src/harness/session/`, to the `SessionTreeEntry`
union or `SessionStorage` contract in `packages/agent/src/harness/types.ts`, to
message projection in `packages/agent/src/harness/messages.ts`, to the
persistence paths of `packages/agent/src/harness/agent-harness.ts`, or to any
other `SessionStorage` implementation such as
`packages/storage/sqlite-node/src/sqlite/storage/index.ts`.

## The Local Pattern

### The log is append-only; the leaf is data, not a cursor

A session is an ordered list of `SessionTreeEntry` (`types.ts:453`) plus one
active leaf. `leafIdAfterEntry` (`session/jsonl-storage.ts:134`) defines the
rule: appending any entry moves the leaf to that entry, except a `leaf` entry,
which moves it to `targetId`. `setLeafId()` therefore *appends* a durable
`LeafEntry` (`jsonl-storage.ts:254`, `memory-storage.ts:75`) instead of mutating
an in-memory pointer, so reopening storage reconstructs the same leaf —
asserted by `test/harness/storage.test.ts` ("loads existing entries and
reconstructs leaf"). A new backend that keeps the leaf only in memory is broken
even if every other method passes.

Entry ids are the last 8 characters of a `uuidv7()` with a 100-attempt collision
retry (`jsonl-storage.ts:43`). The source comment states why: the uuidv7 prefix
is timestamp-derived and nearly constant between calls, so the entropy must come
from the tail.

### Three interchangeable backends

`SessionStorage` (`types.ts:498`) is implemented by `InMemorySessionStorage`,
`JsonlSessionStorage`, and `SqliteSessionStorage`
(`packages/storage/sqlite-node/src/sqlite/storage/index.ts:116`). Semantics are
proven by a parity suite: `runSessionSuite()` in
`packages/agent/test/harness/session.test.ts` runs the same `Session` assertions
against in-memory and JSONL storage, with an optional `inspect()` callback that
checks the JSONL file layout. New `Session` behavior goes into that suite, not
into a backend-specific test.

Known debt: `memory-storage.ts` and `jsonl-storage.ts` currently carry verbatim
copies of `updateLabelCache`, `buildLabelsById`, `generateEntryId`,
`leafIdAfterEntry`, `getSessionStats`, and `getPathToRootOrCompaction`. There is
no shared helper module for them. When you change one of these semantics, change
all copies in the same edit.

### `Session` is a thin typed writer

`Session` (`session/session.ts:150`) holds a `SessionStorage` and a
`SessionContextBuildOptions`. Every writer follows one shape: build the entry
with `await storage.createEntryId()`, `await storage.getLeafId()`, and
`new Date().toISOString()`, close it with `satisfies <EntryType>`, then hand it
to `appendTypedEntry`. A new entry type needs a matching `append*` here; nothing
else in the package constructs entries directly.

`moveTo()` (`session.ts:338`) is the only tree-navigation writer. It validates
the target, calls `setLeafId()`, and optionally appends a `branch_summary` whose
`parentId` is the new leaf.

### Context is built in a fixed pipeline

1. `Session.getBranch()` → `storage.getPathToRootOrCompaction(leafId)` walks
   `parentId` upward and stops at a `compaction` with a `retainedTail`, or at
   its `firstKeptEntryId`.
2. `buildContextEntries()` (`session.ts:92`) applies
   `defaultContextEntryTransform` (`session.ts:59`) first, then any stacked
   `entryTransforms`.
3. `sessionEntryToContextMessages()` (`session.ts:103`) projects entries to
   `AgentMessage[]`. `custom` entries produce nothing unless an
   `entryProjectors[customType]` is registered; `custom_message`, `compaction`,
   and `branch_summary` map through the factories in `messages.ts`.
4. `convertToLlm()` (`messages.ts`) collapses the pi-specific roles
   (`bashExecution`, `custom`, `branchSummary`, `compactionSummary`) into
   provider `user` messages at the provider boundary only.

`deriveSessionContextState` (`session.ts:39`) reads thinking level, model, and
active tool names from the *full* branch, not from the transformed entries, so
config survives compaction.

### Errors: `Result` in, `SessionError` out

Storage and repos consume filesystem `Result`s and convert them at exactly one
place, `getFileSystemResultOrThrow()` (`session/repo-utils.ts:24`), which maps
`not_found` to `SessionError("not_found", ...)` and everything else to
`"storage"`. Do not unwrap a `FileError` anywhere else in
`src/harness/session/`.

### Repos own naming, listing, and forking

`JsonlSessionRepo` (`session/jsonl-repo.ts:38`) stores sessions under
`<sessionsRoot>/<encodeCwd(cwd)>/<timestamp>_<id>.jsonl`; `encodeCwd`
(`jsonl-repo.ts:34`) turns `/tmp/my-project` into `--tmp-my-project--`, checked
by `test/harness/repo.test.ts`. `list()` swallows only
`SessionError` with code `invalid_session` and rethrows everything else, so one
corrupt file cannot hide a real I/O failure.

Fork selection is shared by both repos through `getEntriesToFork()`
(`repo-utils.ts:32`): default `position: "before"` requires the target to be a
user message and forks from its parent; `position: "at"` forks including the
target.

### Writes during an active operation are deferred, not dropped

`AgentHarness` persists immediately while `phase === "idle"` (the `phase` field
is declared at `agent-harness.ts:181`; the branches that test it are at `:951`,
`:973`, `:1006`, `:1040`) and otherwise pushes a `PendingSessionWrite`
(`types.ts:555` — session entry shapes minus `id`/`parentId`/`timestamp`).
`flushPendingSessionWrites()` (`agent-harness.ts:554`) drains the queue in FIFO
order, shifting each entry only after it is persisted, so a mid-flush failure
loses nothing. Flushes happen in `handleAgentEvent`
(`agent-harness.ts:580`) at `turn_end` and `agent_end`, in `prepareNextTurn`,
and in the `finally` of `executeTurn`. Ordering is asserted by
`test/harness/agent-harness.test.ts` ("orders pending listener session writes
after agent-emitted messages"): `["user", "assistant", "custom"]`.

## Reference Files

- `packages/agent/src/harness/types.ts` — entry union, `SessionStorage`,
  `SessionRepo`, `PendingSessionWrite`, `SessionError`
- `packages/agent/src/harness/session/session.ts` — `Session`, context pipeline
- `packages/agent/src/harness/session/jsonl-storage.ts` — v3 header validation,
  leaf reconstruction, stats
- `packages/agent/src/harness/session/memory-storage.ts` — in-memory parity
- `packages/agent/src/harness/session/jsonl-repo.ts`,
  `memory-repo.ts`, `repo-utils.ts`
- `packages/agent/src/harness/messages.ts` — summary/custom message factories,
  `convertToLlm`
- `packages/storage/sqlite-node/src/sqlite/storage/index.ts` — third backend
- `packages/agent/test/harness/session.test.ts` — `runSessionSuite` parity suite
- `packages/agent/test/harness/storage.test.ts` — backend contract tests
- `packages/agent/test/harness/repo.test.ts` — layout, listing, forking
- `packages/agent/test/harness/session-test-utils.ts` — shared message and
  temp-dir helpers

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| Tracking the leaf as in-memory state without appending a `leaf` entry | Reopened storage restores the wrong leaf | `jsonl-storage.ts:254`; `storage.test.ts` leaf reconstruction test |
| Rewriting or deleting an existing JSONL line | The format is append-only; `JsonlSessionStorage` only ever calls `appendFile` | `jsonl-storage.ts` `appendEntry` / `setLeafId` |
| Adding an entry type without touching all storages, the compaction cut-point switch, and `convertToLlm` | Silent context loss for that entry kind | `compaction.ts:328` `findValidCutPoints` enumerates every entry type explicitly |
| Adding a `Session` test to one backend only | Backend drift; the parity suite exists to prevent it | `session.test.ts` `runSessionSuite` |
| Unwrapping a filesystem `Result` outside `getFileSystemResultOrThrow` | Loses the `not_found` → `SessionError` mapping | `repo-utils.ts:24` |
| Writing to the session directly while the harness is busy | Splits an assistant tool-call message from its tool results | `agent-harness.ts:554`; ordering test in `agent-harness.test.ts` |
| Importing `node:fs` into `src/harness/session/` | Breaks the browser bundle check | storages take a `Pick<FileSystem, ...>`; enforced by `scripts/check-browser-smoke.mjs` |
