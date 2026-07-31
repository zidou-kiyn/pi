# Schema And Migrations

## When This Applies

Any change to `packages/storage/sqlite-node/src/sqlite/migrations/001_initial.sql`,
`migrations.ts`, `repo.ts`, or `storage/*`; any new table, column, or index; any
change to how an entry is encoded, decoded, or projected into
`session_materialized` / `entry_materialized` / `branch_entries`.

## The Local Pattern

### Migrations are file-based, id-recorded, and one transaction each

`applyMigrations` (`src/sqlite/migrations.ts:34`) creates the `migrations`
table (`id TEXT PRIMARY KEY, applied_at TEXT NOT NULL`), loads the applied ids,
and runs each unapplied migration inside `db.transaction`, inserting its id in
the same transaction. The id is the filename, not a hash of the SQL — so
editing `001_initial.sql` has no effect on a database that already recorded it
(`src/sqlite/migrations.ts:41`). A schema change is a new `002_<name>.sql` plus
a new entry in `loadMigrations` (`src/sqlite/migrations.ts:15`), which is a
hardcoded array, not a directory scan.

`applyMigrations` runs from `SqliteSessionRepo.openDatabase`
(`src/sqlite/repo.ts:74`) on **every** repo operation, after
`configureSqliteDatabase` sets `PRAGMA journal_mode=WAL`,
`PRAGMA synchronous=FULL`, `PRAGMA busy_timeout=5000` (`src/sqlite/repo.ts:30`).
It is on the hot path and must stay cheap and idempotent.

### `.sql` files reach `dist/` only through `prepare-dist.mjs`

`loadMigrationSql` (`src/sqlite/migrations.ts:11`) resolves the SQL relative to
its own module URL (`new URL(relativePath, import.meta.url)`), which in `dist/`
is `dist/sqlite/migrations/` — a directory `tsgo` never populates. The package
`build` script chains `node ./scripts/prepare-dist.mjs copy-sqlite-migrations`
(`packages/storage/sqlite-node/package.json`), and `files` publishes only
`dist`. A new non-`.ts` asset without the same treatment breaks the published
package at runtime while the source tree keeps working.

### Table layout and why `WITHOUT ROWID`

`001_initial.sql` defines six tables (the seventh, `migrations`, is created in
`src/sqlite/migrations.ts:27`, not in SQL). `sessions`, `session_sequences`,
`branch_entries`, `session_materialized`, and `entry_materialized` are
`WITHOUT ROWID` (their primary key is the access path);
`packages/agent/test/harness/sqlite-migrations.test.ts:95-104` asserts exactly
that set. `session_entries` is a normal rowid table with
`PRIMARY KEY (session_id, id)` plus a unique index on `(session_id, entry_seq)`.
Sequence allocation is explicit, not `AUTOINCREMENT`: `getNextSequence` reads
`session_sequences.next_seq` and `advanceSequence` writes `nextSeq + 1`
(`src/sqlite/storage/session-sequences.ts:4,14`), both inside the append
transaction.

### One append = one transaction, with in-memory rollback

`SqliteSessionStorage.appendEntry` (`src/sqlite/storage/index.ts:291`)
snapshots `materializedState`, `byId`, `currentLeafId`, and `activeBranchId`,
then does inside a single `db.transaction`: detect whether the parent already
had a child, allocate the sequence, insert into `session_entries`, advance the
sequence, rewrite `session_materialized`, insert any `entry_materialized` rows,
update `sessions.active_leaf_id`, and extend or rebuild `branch_entries`. On
any failure it restores all four snapshots and rethrows a
`SessionError("storage", ...)` unless the error already is a `SessionError`.
`packages/agent/test/harness/sqlite-migrations.test.ts:323-356` proves the
rollback by making the `UPDATE sessions SET active_leaf_id` statement throw.

Because `NodeSqliteDatabase.transaction` (`src/index.ts:63`) issues raw
`BEGIN` / `COMMIT` / `ROLLBACK` with no savepoints, transactions do not nest.
Helpers called from inside `appendEntry` (`materializeBranch`,
`appendToActiveBranch`, `getNextSequence`) take the `db` and issue statements
directly; none of them opens its own transaction.

### `branch_entries` is a cache, rebuilt only on branch change

`materializeBranch` (`src/sqlite/storage/index.ts:146`) mints a new `branch_id`
(`uuidv7`) and writes the whole path for the leaf; `appendToActiveBranch`
(`:167`) adds only the new tip for a linear append. A rebuild happens on a
`leaf` entry (branch switch) or when the parent already had a child (fork).
`packages/agent/test/harness/sqlite-migrations.test.ts:170-197` pins the
resulting row counts (three branch ids, the root row duplicated three times).
Reads go through `getMaterializedBranchPathOrCompaction`
(`src/sqlite/storage/branch-entries.ts:11`), which drops `leaf` entries because
they are navigation markers, not context.

### Projections are write-through, and deliberately narrow

`applyEntryToMaterializedState` (`src/sqlite/storage/session-materialized.ts:136`)
folds each entry into the in-memory summary; `serializeSummary` (`:210`) writes
it to `session_materialized.payload` as JSON. `entryMaterializedValues` (`:339`)
currently emits a row **only** for `label`; every other entry type returns `[]`,
and `packages/agent/test/harness/sqlite-migrations.test.ts:435-436` asserts that
no `thinking` or `model` rows exist. Reopen state is rebuilt by
`materializedStateFromRows` (`:281`) from those two tables, never by replaying
`session_entries`.

### Decode is permissive on bulk reads, strict on branch reads

`decodeEntryRows` (`src/sqlite/storage/index.ts:28`) and `findEntries` (`:366`)
swallow malformed rows with the comment "Keep JSONL-like permissive resume
behavior", matching `packages/agent/src/harness/session/jsonl-storage.ts`.
`getMaterializedBranchPathOrCompaction` instead throws `invalidSession`
(`src/sqlite/storage/branch-entries.ts:45,53`). Keep that split: a corrupt row
must not silently shorten a model-context path.

### The adapter is the only `node:sqlite` dependency

`src/index.ts` is the entire Node binding: `isNamedParameters` (`:5`) routes
object-first calls to `node:sqlite`'s named-parameter form (proven by
`packages/agent/test/harness/sqlite-node.test.ts`), and results are narrowed
with `Number(result.changes)` because `node:sqlite` returns bigints. Everything
under `src/sqlite/` talks only to the `SqliteDatabase` / `SqliteStatement`
interfaces in `src/sqlite/types.ts`, which is why the harness tests can inject
`CountingDatabase` and `ThrowingStatement`
(`packages/agent/test/harness/sqlite-migrations.test.ts:23-64`). Do not import
`node:sqlite` outside `src/index.ts`.

### Connections are per-operation

`list` and `delete` open a database and close it in `finally`
(`src/sqlite/repo.ts:120,144`). `create`, `open`, and `fork` hand the open
database to `SqliteSessionStorage`, which owns it until `cleanup()`
(`src/sqlite/storage/index.ts:446`); they close it themselves only on the error
path. New repo methods must follow one of those two shapes.

## Reference Files

Paths are under `packages/storage/sqlite-node/` unless noted.

- `src/index.ts`, `src/sqlite/types.ts` — adapter, capability interfaces
- `src/sqlite/migrations.ts`, `src/sqlite/migrations/001_initial.sql`,
  `scripts/prepare-dist.mjs` — migration mechanism and packaging
- `src/sqlite/repo.ts` — repo methods, pragmas, connection lifecycle
- `src/sqlite/storage/` — `index.ts` (engine), `branch-entries.ts`,
  `session-entries.ts`, `session-materialized.ts`, `session-sequences.ts`,
  `sessions.ts`, `shared.ts`
- `packages/agent/test/harness/sqlite-migrations.test.ts` and
  `sqlite-node.test.ts` — the only tests for this package
- `packages/agent/src/harness/types.ts` (contracts) and
  `packages/agent/src/harness/session/jsonl-storage.ts` (parity reference)

## Anti-Patterns

| Anti-pattern | Why | Evidence / enforcement |
|---|---|---|
| Editing `001_initial.sql` to change an existing table | Skipped on every database that recorded the id; only fresh databases pick it up | `src/sqlite/migrations.ts:41` |
| Adding a `.sql` (or any asset) without updating `prepare-dist.mjs` | Published `dist/` lacks it; `loadMigrationSql` throws at runtime | `packages/storage/sqlite-node/package.json` build script |
| Calling `db.transaction` inside another `db.transaction` | Raw `BEGIN` with no savepoint; SQLite rejects the nested begin | `src/index.ts:63` |
| Mutating `this.byId` / `currentLeafId` / `activeBranchId` outside the snapshot-and-restore block | Leaves in-memory state ahead of a rolled-back transaction | `sqlite-migrations.test.ts:323-356` |
| Importing `node:sqlite` under `src/sqlite/` | Breaks the injectable-factory boundary the harness tests rely on | `src/sqlite/types.ts`, `CountingDatabase` in the harness test |
| Adding a `SessionTreeEntry` type without updating all four switches | `decodeEntry` has no `never` guard, so the gap only shows at runtime | `session-entries.ts:214`, `session-materialized.ts:203,363` |
| Rebuilding the whole branch on a linear append | Turns O(1) appends into O(path) writes | `src/sqlite/storage/index.ts:167` comment |

## Known Debt

Record these as debt; do not treat them as the intended design.

- `SqliteMigration.order` (`src/sqlite/migrations.ts:7`) is set but never read;
  ordering comes from array position in `loadMigrations`.
- `materializedStateFromRows` (`src/sqlite/storage/session-materialized.ts:281`)
  always returns `modelThinkingConfigs: []`, so the configs accumulated by
  `addModelThinkingConfig` are lost on reopen. No consumer reads them today.
- `session-materialized.ts:316-317` is an empty `if (row.type !== "label") {}`
  block left from the label-only projection.
- The 7-column `SELECT ... FROM session_entries` string is duplicated six times
  across `storage/index.ts` and `storage/branch-entries.ts`.
- `tsconfig.json:15` maps `@earendil-works/pi-agent-sqlite-node`, but the
  package is named `@earendil-works/pi-storage-sqlite-node`. The alias resolves
  nothing, which is why the harness tests import
  `../../../storage/sqlite-node/src/index.ts` by relative path.
- `SessionStorage` (`packages/agent/src/harness/types.ts:498`) has no
  `cleanup()`, so `cleanupSessionStorage` (`src/sqlite/repo.ts:36`) probes for
  it structurally before forking.
