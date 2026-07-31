# Writing Evals

## When This Applies

Adding or changing an eval suite (`packages/evals/src/smoke.eval.ts`,
`packages/evals/src/extensions.eval.ts`), the Pi adapter
(`packages/evals/src/pi-harness.ts`), the reporting layer
(`packages/evals/src/vitest-evals/`), or the runner
(`packages/evals/scripts/run-evals.mjs`).

## The Local Pattern

### One harness per `describeEval`, built by `createPiCodingAgentHarness`

`createPiCodingAgentHarness` (`pi-harness.ts:246`) is overloaded: pass `output`
and the harness result type becomes your `TOutput`; omit it and the result is the
final assistant string. `smoke.eval.ts` uses the string form,
`extensions.eval.ts:createExtensionAuthoringHarness` uses the `output` form to
surface `session.resourceLoader.getExtensions()`, the loaded tool names, and the
generated extension source.

Scenario-specific observations belong in `output`, not in the generic adapter.
That keeps `runPiCodingAgent` free of per-eval knowledge and keeps results
JSON-safe (`JsonValue` from `vitest-evals/harness`).

### Model selection is explicit or fully defaulted, never half

`resolveModelSelection` (`pi-harness.ts:46`) trims an explicit harness model or
falls back to `PI_PROVIDER` / `PI_MODEL`, and throws
`Select a harness model explicitly or set both PI_PROVIDER and PI_MODEL as defaults.`
when either half is missing. `packages/evals/test/pi-harness.test.ts` pins all
four cases, including the empty-string provider. Comparative harnesses that pin a
model (`model: { provider, id }`) are independent of the runner default; that is
the documented way to build model-comparison sets.

### Every run is isolated in a fresh temp tree

`runPiCodingAgent` (`pi-harness.ts`) creates `mkdtemp(join(tmpdir(),
"pi-eval-"))` and derives `workspace`, `agent`, and `sessions` under it, uses
`SettingsManager.inMemory()`, and hard-fails when the session inherited
extensions:

```ts
if (evalSession.extensionRunner.getExtensionPaths().length !== 0) {
	throw new Error("Expected an isolated eval session to start without extensions.");
}
```

Never point an eval at the repository cwd or the developer's real agent
directory; the isolation guarantee is what makes `extensions.eval.ts` able to
assert on a `hello.ts` extension the model just generated inside the temporary
workspace.

### Failures never hide cleanup failures

The run body records `outcome` as a tagged success/failure value instead of
rethrowing inline, then always snapshots the session JSONL, disposes the session,
and removes the temp root, collecting `cleanupErrors`. A failed run plus failed
cleanup throws `AggregateError`; cleanup-only failures throw the single error or
an `AggregateError`. Preserve that ordering — the snapshot must be read before
`rm(root, { recursive: true, force: true })`.

### Artifacts flow run → task → reporter

- The harness calls `setArtifact("runId", sessionManager.getSessionId())` and
  `setArtifact(PI_SESSION_SNAPSHOT_ARTIFACT, ...)` (`artifacts.ts:13`).
- `packages/evals/src/vitest-evals/setup.ts` is the eval-only `afterEach` hook
  registered by `setupFiles`; it calls `recordEvalSessionArtifact(task, run)` so
  the artifact is bound to the explicit Vitest task before reporters run.
- `EvalHarnessReporter` (`reporter.ts:87`) appends one JSONL record per test to
  `runs.jsonl` and calls `persistEvalArtifactReferences`, which writes
  attachments under `sessions/<sha256(runId)>/` or `sources/<sha256(runId)>/`
  with `mkdir` mode `0o700` and `writeFile` mode `0o600`, and rejects any
  attachment name that is not its own `basename`.
- All of that is skipped when `PI_EVAL_ARTIFACT_DIR` is unset, so do not write
  code that assumes `runs.jsonl` exists.

New artifact types are declared through the `declare module "vitest"` block in
`artifacts.ts` (`TestArtifactRegistry` keyed by a module-private `Symbol`), not
by loosening types at the call site.

### Comparative sets: `evalHarnessTable` + `describe.for`

`evalHarnessTable` (`harness-table.ts:157`) expands `{ baseline, candidate |
candidates, repetitions }` into rows ordered repetition-major, wrapping each
harness so it attaches an `EVAL_HARNESS_ITERATION_ARTIFACT` on both the success
and the error path (`getHarnessRunFromError` / `attachHarnessRunToError`).
`validateOptions` rejects an empty eval-set name, zero candidates, duplicate
harness names, and non-positive-integer repetitions.

Pairing depends on `deriveEvalGroupKey` (`harness-table.ts:110`): a non-empty
string `input.id` when present, otherwise a SHA-256 of strict canonical JSON.
`canonicalizeJson` throws on non-finite numbers, sparse arrays, circular
references, and non-plain objects — `packages/evals/test/vitest-evals/harness-table.test.ts`
pins each rejection. Keep eval inputs plain JSON so runs group correctly.

### Scoring is observational, not assertional

`summary.ts` treats a run as passing at `score >= 1`, and only counts a pair when
**both** sides have `outcome: "scored"`. Everything else becomes a
`HarnessComparisonDiagnostic` with reason `missing-observation`,
`duplicate-observation`, `harness-error`, `missing-score`, or
`unscorable-outcome` — never a silent zero. Comparative suites therefore set
`judgeThreshold: null` (`extensions.eval.ts:106` is the single `describeEval`
call, executed once per `describe.for` table row) so a low score is recorded
rather than failing the Vitest invocation. Reserve hard `expect(...)` for suite
invariants and infrastructure contracts, as `extensions.eval.ts` does for
`systemPromptHasGuidelines`.

### Runner behavior

`packages/evals/scripts/run-evals.mjs` parses `--provider` / `--model` in both `--flag value`
and `--flag=value` forms, exits non-zero when only one is supplied, creates
`.eval/<iso-timestamp>_<uuid>` with mode `0o700` (or honors an existing
`PI_EVAL_ARTIFACT_DIR`), deletes `PI_PROVIDER` / `PI_MODEL` from the child env
when no default was resolved, and forwards remaining argv to
`vitest run --config vitest.config.ts`. `.eval/` is ignored via
`packages/evals/.gitignore`.

## Reference Files

- `packages/evals/README.md` — authoring reference, comparative-eval methodology
- `packages/evals/src/pi-harness.ts` — `createPiCodingAgentHarness`,
  `resolveModelSelection`, `runPiCodingAgent`, `toTranscriptEvents`
- `packages/evals/src/smoke.eval.ts` — minimal single-harness suite
- `packages/evals/src/extensions.eval.ts` — typed `output`, custom judge,
  baseline/candidate table, source artifact recording
- `packages/evals/src/vitest-evals/artifacts.ts`, `harness-table.ts`,
  `reporter.ts`, `setup.ts`, `summary.ts`
- `packages/evals/test/pi-harness.test.ts`,
  `packages/evals/test/vitest-evals/artifacts.test.ts`,
  `packages/evals/test/vitest-evals/harness-table.test.ts`,
  `packages/evals/test/vitest-evals/summary.test.ts`
- `packages/evals/scripts/run-evals.mjs`, `packages/evals/vitest.config.ts`,
  `packages/evals/vitest.test.config.ts`

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| Deterministic logic tested in `src/**/*.eval.ts` | `vitest.test.config.ts` only includes `test/**/*.test.ts`, so it never runs under `./test.sh`, and it costs tokens when it does run | The four existing unit tests all live under `packages/evals/test/` |
| Hard-asserting a judge score in a comparative set | Turns an observation into a build failure; `expect.soft(...)` also fails the test and is not a scoring mechanism | `packages/evals/README.md`, `judgeThreshold: null` in `extensions.eval.ts` |
| Reusing a harness name inside one eval set | `validateOptions` throws `Harness names must be unique within an eval set.` | `harness-table.ts`, `harness-table.test.ts` |
| Non-plain eval input (`Date`, class instance, sparse array, circular) | `canonicalizeJson` throws, so the run never groups | `harness-table.test.ts` "rejects non-JSON and circular input" |
| Adding scenario-specific behavior to `runPiCodingAgent` | The adapter stays generic; scenarios extend through `output` | `extensions.eval.ts:createExtensionAuthoringHarness` |
| Eval that reads the repo cwd or the developer's `~/.pi` | Breaks the temp-root isolation and the zero-extensions invariant | `pi-harness.ts` `mkdtemp` + `SettingsManager.inMemory()` |
| Swallowing a cleanup error to let a run "pass" | Cleanup errors are aggregated and rethrown deliberately | `pi-harness.ts` `cleanupErrors` / `AggregateError` |
| Assuming `runs.jsonl` or attachment files exist | Both reporter paths no-op without `PI_EVAL_ARTIFACT_DIR` | `reporter.ts:appendHarnessRunReport` |
