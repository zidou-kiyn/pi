# Testing

## When This Applies

Any change that adds or modifies a test, and any change to `packages/*/src`
that needs a regression guard.

## The Local Pattern

### Never run the raw vitest suite

Many tests are live-provider e2e tests gated on credentials:

```ts
describe.skipIf(!process.env.OPENAI_API_KEY)("openai responses cache affinity e2e", () => { ... });
describe.skipIf(!process.env.GEMINI_API_KEY)("Google Provider Abort", () => { ... });
```

(`packages/ai/test/openai-responses-cache-affinity-e2e.test.ts:5`,
`packages/ai/test/abort.test.ts:101`.) If the corresponding env var is present
in your shell, those tests wake up and spend real tokens.

Two safe entry points:

```bash
./test.sh                                                   # whole repo, sanitized env
node ../../node_modules/vitest/dist/cli.js --run test/x.test.ts   # single file, from package root
```

`test.sh` builds a throwaway `HOME`, `TMPDIR`, and npm config under
`$TMPDIR/pi-test.XXXXXX`, starts from `env -i`, and forwards only an explicit
allowlist (`PI_NO_LOCAL_LLM=1`, `GIT_CONFIG_GLOBAL=/dev/null`,
`AWS_EC2_METADATA_DISABLED=true`, ...). That is why it is the only sanctioned
full-suite command.

### Use the faux provider instead of real APIs

`registerFauxProvider` (`packages/ai/src/compat.ts:160`) is the registration
entry point; the message builders `fauxAssistantMessage`, `fauxText`,
`fauxThinking`, `fauxToolCall` and the lower-level `fauxProvider` /
`createFauxCore` live in `packages/ai/src/providers/faux.ts`. Tests import them
from `@earendil-works/pi-ai/compat`. Register in the test, unregister in
`afterEach` — see `packages/agent/test/e2e.test.ts:18-37` for the
registration-tracking pattern.

### coding-agent suite tests have their own contract

`packages/coding-agent/test/suite/README.md` is binding:

- use `test/suite/harness.ts`
- use the faux provider; no real provider APIs, keys, network, or paid tokens
- keep tests CI-safe and deterministic
- do not extend the legacy `test/test-harness.ts` unless a capability is missing
- broad lifecycle tests go directly in `test/suite/`
- issue regressions go in `test/suite/regressions/` named
  `<issue-number>-<short-slug>.test.ts`, e.g.
  `2023-queued-slash-command-followup.test.ts`

### Config layout

| Config | Purpose |
|---|---|
| `vitest.base.ts` | Root aliases mapping `@earendil-works/pi-*` to workspace source, so tests run against `src`, not `dist` |
| `packages/coding-agent/vitest.config.ts` | Merges the base config, forces `PI_OFFLINE=1`; opt back in with `allowNetwork()` from `test/test-network-env.ts` |
| `packages/agent/vitest.harness.config.ts` | Harness-only run (`test/harness/**/*.test.ts`) with v8 coverage over `src/harness/**/*.ts`, `src/agent.ts`, `src/agent-loop.ts` |
| `packages/tui/vitest.config.ts` | `include` is limited to `test/wrap-ansi.test.ts`; every other TUI test must be named explicitly on the command line |

### Interactive TUI verification uses tmux

For behavior that only appears in a real terminal, `AGENTS.md` prescribes:

```bash
tmux new-session -d -s pi-test -x 80 -y 24
tmux send-keys -t pi-test "./pi-test.sh" Enter
sleep 3 && tmux capture-pane -t pi-test -p
tmux kill-session -t pi-test
```

## Reference Files

- `test.sh` — sanitized full-suite runner
- `vitest.base.ts`, `packages/*/vitest*.config.ts`
- `packages/coding-agent/test/suite/README.md` — suite rules
- `packages/coding-agent/test/suite/harness.ts` — the harness to use
- `packages/coding-agent/test/suite/regressions/` — naming precedent
- `packages/agent/test/e2e.test.ts` — faux provider registration/cleanup
- `packages/ai/test/abort.test.ts` — credential gating with `describe.skipIf`

## Anti-Patterns

| Anti-pattern | Consequence |
|---|---|
| `npx vitest` / `npm test` at repo root with provider keys exported | Fires paid e2e tests |
| New test that calls a real provider | Non-deterministic CI, burns tokens |
| Regression test placed outside `test/suite/regressions/` or misnamed | Breaks the issue-to-test trail |
| Test asserting behavior that passes with the feature deleted | Tautological test; `guides/index.md` lists this as a review trigger |
| Extending `test/test-harness.ts` for new suite work | Legacy path; `test/suite/harness.ts` is the target |

Rule: if you create or modify a test file, run it and iterate until it passes
before reporting the work done.
