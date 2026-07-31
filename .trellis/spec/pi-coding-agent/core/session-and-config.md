# Session And Config

## When This Applies

Changing `AgentSession` lifecycle/events, the services-vs-session split,
settings resolution (global vs. project), the session JSONL file format, or
one-time startup migrations.

## The Local Pattern

### Services are cwd-scoped infrastructure, separate from the session

`createAgentSessionServices` (`agent-session-services.ts:129`) builds
`AgentSessionServices` (`modelRuntime`, `settingsManager`, `resourceLoader`)
bound to one `cwd`. The doc comment is explicit: "This is infrastructure
only. The AgentSession itself is created separately so session options can be
resolved against these services first." `createAgentSessionFromServices`
(`agent-session-services.ts`) then builds the actual session against already
resolved services. Services are recreated whenever the effective session
`cwd` changes (e.g. `/cd`, extension `newSession`), the session object is not.

### `AgentSession` emits two parallel event shapes from the same call sites

`AgentSession._emit` (`agent-session.ts:548`) delivers `AgentSessionEvent` to
listeners added via `session.subscribe`. Most state transitions in the same
method also call `this._extensionRunner.emit({...})` with a differently
shaped `ExtensionEvent` (`agent_start`, `agent_end`, `tool_call`, ...). These
are two independent type hierarchies (`AgentSessionEvent` in
`agent-session.ts`, `ExtensionEvent` in `core/extensions/types.ts`) fired in
pairs — adding a new lifecycle point usually means adding to both, not one.

### Settings: global + project, deep-merged, project gated on trust

`SettingsManager` (`settings-manager.ts:274`) holds two independent `Settings`
objects: global from `<agentDir>/settings.json`, project from
`<cwd>/.pi/settings.json` (`FileSettingsStorage`, `settings-manager.ts:189-196`).
`deepMergeSettings` merges nested objects key-by-key; primitives and arrays
from the override (project) win outright. Project settings are never loaded
for an untrusted project:

```ts
private static loadFromStorage(storage: SettingsStorage, scope: SettingsScope, projectTrusted = true): Settings {
    if (scope === "project" && !projectTrusted) {
        return {};
    }
```

(`settings-manager.ts:350-351`). All settings mutation goes through
`SettingsManager` methods, which take a `settings.json` file lock via
`proper-lockfile` and track `modifiedFields`/`modifiedProjectFields` for
partial re-merge after external file changes.

That re-merge protects **only unmodified fields, and within a modified field
only its untouched nested keys**. `persistScopedSettings`
(`settings-manager.ts:578-606`) re-reads the file inside the lock, then for
each entry of `modifiedFields` either merges per nested key (when
`modifiedNestedFields` has that field and the value is a non-null object) or
falls through to `mergedSettings[field] = value` — the whole in-memory value,
written over whatever the file now holds. Array fields always take that second
branch. So an external write to `packages[]` survives only until the session
itself modifies `packages` (`setPackages()`, `settings-manager.ts:973-976`,
reached from `togglePackageResource`, `config-selector.ts:583`), at which point
the session's startup-era array is persisted and the external entries vanish
silently. Holding the same lock does not help; the staleness is in the
snapshot, not the write. An out-of-process writer must therefore require a
restart before the user touches `/config` or `pi install` in that session.

### Session files are append-only JSONL, versioned and migrated in place

`session-manager.ts` defines `SessionHeader` plus a `SessionEntry` union
(`SessionMessageEntry`, `ModelChangeEntry`, `CompactionEntry`, `LabelEntry`,
`CustomEntry`, ...). `CURRENT_SESSION_VERSION = 3` (`session-manager.ts:30`);
`migrateV1ToV2` / `migrateV2ToV3` upgrade older files in place through
`migrateSessionEntries`, run every time a session file is loaded
(`loadEntriesFromFile`, `session-manager.ts:514`).

### Startup migrations are one-shot and distinct from session-file migration

`migrations.ts` `runMigrations(cwd)` runs once per process start:
`migrateAuthToAuthJson` (legacy `oauth.json` / `settings.json.apiKeys` ->
`auth.json`), `migrateSessionsFromAgentRoot` (fixes a v0.30.0 bug that wrote
sessions to `~/.pi/agent/*.jsonl` instead of the per-cwd session directory,
see `earendil-works/pi-mono#320`), `migrateToolsToBin`,
`migrateKeybindingsConfigFile`, and `migrateExtensionSystem` (renames
`commands/` to `prompts/`, warns about deprecated `hooks/`/`tools/`
directories). These never touch the JSONL entry format.

### Compaction is I/O-free, driven by `AgentSession` events

`core/compaction/compaction.ts` has no file or network I/O of its own; it
takes `SessionEntry[]` / `AgentMessage[]` plus a `StreamFn` and returns a
`CompactionResult` (summary, `firstKeptEntryId`, usage). `AgentSession` emits
`compaction_start` / `compaction_end` around the call
(`agent-session.ts:1787`, `2072-2182`) and `SessionManager` is what appends
the resulting `CompactionEntry` and reloads session state.

## Reference Files

- `packages/coding-agent/src/core/agent-session.ts`
- `packages/coding-agent/src/core/agent-session-services.ts`
- `packages/coding-agent/src/core/agent-session-runtime.ts`
- `packages/coding-agent/src/core/session-manager.ts`
- `packages/coding-agent/src/core/settings-manager.ts`
- `packages/coding-agent/src/migrations.ts`
- `packages/coding-agent/src/core/compaction/compaction.ts`
- `packages/coding-agent/test/suite/agent-session-runtime.test.ts`
- `packages/coding-agent/test/suite/regressions/3616-settings-inmemory-reload.test.ts`
- `packages/coding-agent/test/suite/regressions/2753-reload-stale-resource-settings.test.ts`

## Anti-Patterns

- Constructing `AgentSession` without going through
  `createAgentSessionServices` / `createAgentSessionFromServices`: services
  are shared with the resource loader and model runtime, and bypassing them
  desyncs cwd-scoped state.
- Writing `settings.json` directly instead of through `SettingsManager`:
  loses the file lock, the `modifiedFields` merge tracking, and
  `migrateSettings()` upgrade handling. Extensions have no access to
  `SettingsManager` (it is absent from `ExtensionAPI` and
  `ExtensionCommandContext` in `core/extensions/types.ts`), so an extension
  that must edit settings can only write the file and then tell the user to
  restart — see the array-field caveat in "Settings layering" above.
- Adding a new `SessionEntry` variant without bumping
  `CURRENT_SESSION_VERSION` and adding a `migrateVxToVy` step —
  `migrateToCurrentVersion` must be able to read every past on-disk shape.
- Adding a new `AgentSession` lifecycle point that only calls `this._emit`
  or only `this._extensionRunner.emit`, when the surrounding code shows both
  are expected for that transition.
