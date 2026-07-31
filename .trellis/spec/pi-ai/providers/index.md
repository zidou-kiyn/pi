# Providers Guidelines

`packages/ai/src/providers/*.ts` (38 provider factories listed in `builtinProviders()`, plus
`radiusProvider` exported separately) and their
paired `providers/*.models.ts`, plus `providers/all.ts`, `providers/faux.ts`, and
`providers/images/`.

## Pre-Development Checklist

1. Read `adding-a-provider.md` in full before writing a new provider file — it corrects two
   stale steps in `.pi/skills/add-llm-provider.md`.
2. Check `packages/ai/src/providers/all.ts`'s `builtinProviders()` array to see the current
   registration list and its alphabetical order; new providers are inserted alphabetically in
   both the `import` block and the array.
3. Decide whether the new provider reuses an existing API implementation
   (`openai-completions`, `openai-responses`, `anthropic-messages`, ...) or needs a new one
   under `src/api/`; most built-in providers reuse `openai-completions` with `compat`
   overrides rather than writing a new wire protocol.
4. Check `packages/ai/src/env-api-keys.ts`'s `getApiKeyEnvVars()` for the env-var naming
   convention (`<PROVIDER>_API_KEY`) before inventing a new one.
5. Exercise the provider through `providers/faux.ts` before touching a real API key; the
   credential-gated e2e tests only run with the actual key exported (see `_shared/testing.md`).
6. Plan verification: `node ../../node_modules/vitest/dist/cli.js --run test/stream.test.ts`
   (and every other test file the provider was added to) from `packages/ai`.

## Guidelines Index

| File | Covers |
|---|---|
| [Adding A Provider](./adding-a-provider.md) | End-to-end pattern for a new provider factory: shape, auth, lazy loading, registration, model data, tests, downstream wiring |

## Shared Rules

Repo-wide rules live in [`_shared/index.md`](../../_shared/index.md); this
layer does not restate them. Thinking guides: [`guides/index.md`](../../guides/index.md).
