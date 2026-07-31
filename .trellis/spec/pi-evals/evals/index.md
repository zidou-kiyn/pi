# Evals Guidelines

Covers the model-backed eval harness in `packages/evals/src/`: the Pi adapter
(`pi-harness.ts`), the eval suites (`smoke.eval.ts`, `extensions.eval.ts`), the
`vitest-evals` integration layer (`packages/evals/src/vitest-evals/`), the runner
`packages/evals/scripts/run-evals.mjs`, and the two vitest configs
(`packages/evals/vitest.config.ts`, `packages/evals/vitest.test.config.ts`).

The package is `private: true` and has no build step. It exists to measure Pi's
end-to-end behavior against real providers, so every change here has a token cost.

## Pre-Development Checklist

1. Read `../../_shared/testing.md` first — it defines the repo's non-negotiable
   rule that unit tests never touch a real provider. This package is the one place
   where a *separate* suite intentionally does, and the two suites must not mix.
2. Identify which suite you are editing:
   - `packages/evals/src/` `*.eval.ts` — model-backed, run only by
     `npm run eval` through `packages/evals/vitest.config.ts`
     (`include: ["src/**/*.eval.ts"]`, `fileParallelism: false`, 120s timeout);
   - `packages/evals/test/` `*.test.ts` — deterministic, run by
     `npm test --workspaces` (and therefore by `./test.sh`) through
     `packages/evals/vitest.test.config.ts` (`include: ["test/**/*.test.ts"]`).
   A deterministic test placed under `src/` never runs in CI and burns tokens
   when it does run.
3. Read `packages/evals/README.md`. It is the authoring reference for harness
   options, comparative sets, judges, and telemetry interpretation, and it is
   kept current with `src/pi-harness.ts`.
4. Start from an existing suite: `packages/evals/src/smoke.eval.ts` for a single
   harness, `packages/evals/src/extensions.eval.ts` for a baseline/candidate
   comparison with a custom judge and a typed `output`.
5. Before running anything model-backed, decide the provider and model
   explicitly. `npm run eval` (root) forwards to
   `packages/evals/scripts/run-evals.mjs`, which requires `--provider` and
   `--model` together, or `PI_PROVIDER` and `PI_MODEL` together. Every run writes
   prompts, responses, source code, and tool output into `.eval/`.
6. Verify deterministic changes with, from `packages/evals`:
   `node ../../node_modules/vitest/dist/cli.js --run --config vitest.test.config.ts test/<file>.test.ts`.
   Then `npm run check` from the repo root for any `.ts` edit.

## Guidelines Index

| File | Covers |
|---|---|
| [Writing Evals](./writing-evals.md) | Harness construction and model resolution, temp-directory isolation, cleanup and artifact contracts, comparative harness tables, judge and scoring conventions, runner behavior |

## Shared Rules

- Repo-wide TypeScript, testing, and check conventions:
  [`../../_shared/index.md`](../../_shared/index.md)
- Generic thinking guides (cross-layer, code reuse):
  [`../../guides/index.md`](../../guides/index.md)
