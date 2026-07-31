# Adding A Provider

## When This Applies

Adding a new LLM provider to `packages/ai`, or any change that touches more than one existing
provider's registration/auth/model-data shape at once.

## The Local Pattern

### Provider factory shape

Every provider file exports one `<name>Provider(): Provider<TApi>` factory built from
`createProvider()` (`packages/ai/src/models.ts`). `packages/ai/src/providers/anthropic.ts` is
representative: id/name/baseUrl, an `auth` object with `apiKey` and optionally `oauth`, a
static `models` array sourced from the provider's own `<NAME>_MODELS` catalog, and an `api`
implementation — a single `ProviderStreams`, or a map keyed by `model.api` for mixed-API
providers. `packages/ai/src/providers/github-copilot.ts` is the mixed-API example: it
dispatches `anthropic-messages` / `openai-completions` / `openai-responses` per model and also
sets `filterModels` to narrow the catalog by the OAuth credential's `availableModelIds`.

### Auth

Standard key-or-stored-credential resolution uses `envApiKeyAuth(name, envVars)`
(`packages/ai/src/auth/helpers.ts`) — stored credential wins, then the first set env var, with
a generated `login` prompt. Providers with non-standard resolution write their own
`ApiKeyAuth.resolve()`: `anthropicApiKeyAuth()` in `providers/anthropic.ts` gives
`ANTHROPIC_AUTH_TOKEN` bearer-header precedence over `ANTHROPIC_OAUTH_TOKEN`/
`ANTHROPIC_API_KEY`, and `getEnvApiKey()`'s `google-vertex`/`amazon-bedrock` branches in
`packages/ai/src/env-api-keys.ts` check ambient ADC/AWS credentials instead of a single env
var. OAuth providers wrap their flow in `lazyOAuth({ name, load })` (`auth/helpers.ts`) so the
OAuth implementation module loads only on the first `login`/`refresh`/`toAuth` call, not at
provider-file import time.

### Lazy-load the API implementation, not the provider file

Provider factories are statically imported by `providers/all.ts`, but the SDK-heavy API
implementation stays behind a lazy wrapper: `anthropicMessagesApi()` in
`packages/ai/src/api/anthropic-messages.lazy.ts` is one line,
`lazyApi(() => import("./anthropic-messages.ts"))` (`api/lazy.ts`). `lazyApi()` returns a
`ProviderStreams` whose `stream`/`streamSimple` trigger the dynamic `import()` on first call;
load failures terminate the returned stream with an `error` event instead of throwing
(`api/lazy.ts`'s `lazyStream()`). A new API implementation under `src/api/<id>.ts` needs a
matching `src/api/<id>.lazy.ts` wrapper; provider factories import the `.lazy.ts` module,
never the implementation module directly.

### Registration point is `providers/all.ts`, not `register-builtins.ts`

`packages/ai/src/providers/all.ts` statically imports every provider factory and lists them,
alphabetically, in both the `import` block (`all.ts:5-45`) and the `builtinProviders():
Provider[]` array (`all.ts:87-128`). `builtinModels()` iterates that array and calls
`models.setProvider()` for each. There is no `providers/register-builtins.ts` for chat
providers — that filename only exists for image providers
(`packages/ai/src/providers/images/register-builtins.ts`, which self-registers via
`registerImagesApiProvider()` at module load, called once at the bottom of the file). A new
chat provider is added directly to `all.ts`'s import block and array; it does not need a
self-registering side-effect import.

### Model data is generated, not hand-written

The provider's model list comes from a generated `providers/<name>.models.ts` (see
`pi-ai/core/model-catalog.md`), produced by extending `packages/ai/scripts/generate-models.ts`
to fetch/parse the provider's models and map them to `Model<TApi>`. Do not write a
`.models.ts` file by hand.

### Bundle hygiene: a single-provider import must not pull in the aggregate

`scripts/agent-treeshake-smoke-entry.ts` imports only `@earendil-works/pi-ai/providers/
anthropic`; `scripts/check-browser-smoke.mjs` asserts that bundle's dependency graph excludes
`packages/ai/src/compat.ts`, `packages/ai/src/models.generated.ts`, and
`packages/ai/src/providers/all.ts`. A new provider file must import only its own
`<name>.models.ts` (which in turn imports only its own `./data/<name>.json`) and never
`models.generated.ts` or `all.ts`, or `npm run check:browser-smoke` fails.

### Tests

`stream.test.ts` is the required minimum — add the provider with at least one representative
model, even when it reuses an existing API implementation. Extend the matrix in
`tokens.test.ts`, `abort.test.ts`, `empty.test.ts`, `context-overflow.test.ts`,
`unicode-surrogate.test.ts`, `tool-call-without-result.test.ts`, `image-tool-result.test.ts`,
`total-tokens.test.ts`, and `cross-provider-handoff.test.ts` where applicable. Every e2e case
that hits a real endpoint is gated with `describe.skipIf`, e.g.
`describe.skipIf(!process.env.OPENAI_API_KEY)` (`test/abort.test.ts:130`); providers with
ambient credentials use a helper predicate instead of a single env var, e.g.
`describe.skipIf(!hasBedrockCredentials())` (`test/abort.test.ts:312`, predicate from
`test/bedrock-utils.ts`). Use `providers/faux.ts` (`fauxProvider()`, `fauxAssistantMessage()`)
for deterministic, non-network tests of the surrounding `Models`/`createProvider()` machinery,
as the `fauxProvider` describe block in `test/providers.test.ts` does.

### Downstream wiring outside this package

Coding-agent needs its own default model id in `defaultModelPerProvider`
(`packages/coding-agent/src/core/model-resolver.ts:14`) and env-var documentation in
`packages/coding-agent/src/cli/args.ts`. That wiring belongs to the `pi-coding-agent/core`
spec layer, not this one.

Documentation also needs updating in the same change: `packages/ai/README.md`'s "Supported
Providers" list and "Environment Variables" table, and `packages/ai/CHANGELOG.md` under
`## [Unreleased]` — both are prose, not generated, so they do not follow the model-data
regeneration path above.

## Reference Files

- `packages/ai/src/providers/anthropic.ts` — single-API provider with a custom
  `ApiKeyAuth.resolve()` and `lazyOAuth`
- `packages/ai/src/providers/github-copilot.ts` — mixed-API provider (`api` as a
  per-model-api map) plus `filterModels`
- `packages/ai/src/providers/all.ts` — the registration point (`builtinProviders()`)
- `packages/ai/src/providers/images/register-builtins.ts` — the one place a
  `register-builtins.ts` file actually exists (images only)
- `packages/ai/src/api/lazy.ts`, `packages/ai/src/api/anthropic-messages.lazy.ts` — the
  lazy-load wrapper pattern
- `packages/ai/src/auth/helpers.ts` — `envApiKeyAuth()`, `lazyOAuth()`
- `packages/ai/src/env-api-keys.ts` — `getApiKeyEnvVars()` naming convention
- `packages/ai/src/providers/faux.ts` — `fauxProvider()`, `fauxAssistantMessage()`
- `packages/ai/test/providers.test.ts`, `packages/ai/test/stream.test.ts`,
  `packages/ai/test/abort.test.ts`
- `.pi/skills/add-llm-provider.md` — existing checklist; steps 1, 2, 4, 5, 7 remain accurate.
  Step 3's `register-builtins.ts` reference and step 6's `provider-display-names.ts`
  reference are stale — neither file exists in this repo today.

## Anti-Patterns

| Anti-pattern | Why | Enforcement |
|---|---|---|
| Statically importing an `src/api/<id>.ts` implementation from a provider factory | Pulls the SDK into every bundle that imports the provider, defeating lazy loading | `scripts/check-browser-smoke.mjs` agent-treeshake check |
| A provider file importing `models.generated.ts` or `providers/all.ts` | Same bundle-hygiene failure, plus a circular-import risk (`all.ts` imports every provider) | `scripts/check-browser-smoke.mjs` |
| Hand-writing `providers/<name>.models.ts` | Regenerated by `generate-models.ts`; biome excludes `*.models.ts` from lint/format | `AGENTS.md`; `scripts/check-model-data.ts` |
| Following `.pi/skills/add-llm-provider.md` step 3 literally (`register-builtins.ts`) for a chat provider | That file does not exist for chat providers; the actual registration point is `providers/all.ts` | Verified in this repo — only `providers/images/register-builtins.ts` exists |
