# Storage Guidelines

Covers `@earendil-works/pi-storage-sqlite-node`: the `node:sqlite` adapter in
`packages/storage/sqlite-node/src/index.ts` and the SQLite session backend in
`packages/storage/sqlite-node/src/sqlite/` (`migrations.ts`, `repo.ts`,
`types.ts`, `storage/*`, `migrations/001_initial.sql`).

## Pre-Development Checklist

1. Read [`../../_shared/typescript-and-style.md`](../../_shared/typescript-and-style.md)
   and [`../../_shared/testing.md`](../../_shared/testing.md) first.
2. Read [`schema-and-migrations.md`](./schema-and-migrations.md) before touching
   any SQL, the `migrations` table, or a projection table
   (`session_materialized`, `entry_materialized`, `branch_entries`,
   `session_sequences`). This is the only package in the repo with SQL, so no
   other package's precedent applies.
3. Find the tests before editing. This package ships no `test/` directory and no
   `test` script (`packages/storage/sqlite-node/package.json`); its coverage
   lives in the agent harness suite:
   `packages/agent/test/harness/sqlite-migrations.test.ts` (repo, storage,
   migrations, projections) and `packages/agent/test/harness/sqlite-node.test.ts`
   (the `node:sqlite` adapter).
4. Check the contract you are implementing. `SqliteSessionStorage`
   (`src/sqlite/storage/index.ts:116`) implements `SessionStorage` and
   `SqliteSessionRepo` (`src/sqlite/repo.ts:43`) implements `SessionRepo`, both
   declared in `packages/agent/src/harness/types.ts:498` and `:528`. The JSONL
   backend `packages/agent/src/harness/session/jsonl-storage.ts` and
   `jsonl-repo.ts` are the parity reference: same observable behavior, different
   substrate.
5. If you add or change a `SessionTreeEntry` variant in `packages/agent`, update
   all four switches in this package in the same change:
   `validateSessionTreeEntry` and `decodeEntry`
   (`src/sqlite/storage/session-entries.ts:50,140`),
   `applyEntryToMaterializedState` and `entryMaterializedValues`
   (`src/sqlite/storage/session-materialized.ts:136,339`). Three of them have a
   `never` exhaustiveness guard; `decodeEntry` does not, because `row.type`
   arrives as a database string.
6. Never change `src/sqlite/migrations/001_initial.sql` to alter a schema that
   already shipped. Add a new numbered file and register it in `loadMigrations`
   (`src/sqlite/migrations.ts:15`).
7. If you add any non-`.ts` asset under `src/`, extend
   `packages/storage/sqlite-node/scripts/prepare-dist.mjs`. `tsgo` only emits
   TypeScript output; the `.sql` files reach `dist/` solely through
   `npm run build`'s `prepare-dist.mjs copy-sqlite-migrations` step.
8. Verify with the harness config, then the repo-wide check:
   ```bash
   cd packages/agent && npm run test:harness      # vitest.harness.config.ts
   npm run check                                  # from the repo root
   ```
   Per [`../../_shared/testing.md`](../../_shared/testing.md), do not run the
   raw vitest suite.

## Guidelines Index

| File | Covers |
|---|---|
| [Schema And Migrations](./schema-and-migrations.md) | Migration workflow and packaging, table layout, the append transaction and its rollback, branch/projection materialization, adapter boundary, known debt |

## Shared Rules

- Repo-wide TypeScript, testing, check, and dependency conventions:
  [`../../_shared/index.md`](../../_shared/index.md)
- Generic thinking guides (cross-layer, code reuse):
  [`../../guides/index.md`](../../guides/index.md)

Note on package docs: `packages/agent/docs/harness.md` and
`packages/agent/docs/harness-v2.md` contain long SQLite sections, but both are
*plans* ("Durable AgentHarness plan" / "design"). The `harness_entries` table,
per-ref leaf rows, and writer-claim leases they describe do not exist in
`001_initial.sql`. Read them for intent; read the source for behavior.
