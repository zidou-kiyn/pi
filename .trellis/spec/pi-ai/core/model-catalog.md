# Model Catalog

## When This Applies

- Adding or regenerating a provider's model list, or changing model metadata (cost, context
  window, thinking levels).
- Touching `packages/ai/src/models.generated.ts`, any `packages/ai/src/providers/*.models.ts`,
  or `packages/ai/src/providers/data/*.json`.
- Debugging why `packages/ai` fails to build, test, or bundle on a fresh checkout.

## The Local Pattern

### Generated file chain

`packages/ai/scripts/generate-models.ts` fetches/parses provider sources and writes two things
per provider: `packages/ai/src/providers/data/<provider>.json` (raw model data, git-ignored —
`.gitignore:11`) and `packages/ai/src/providers/<provider>.models.ts` (a thin generated
wrapper). Each wrapper imports its own JSON and flattens it through `flattenModelCatalog()`
(`packages/ai/src/model-catalog.ts:22-27`):

```ts
import values from "./data/anthropic.json" with { type: "json" };
import { flattenModelCatalog, type ModelCatalog } from "../model-catalog.ts";
export const ANTHROPIC_MODELS: ModelCatalog<typeof values, "anthropic"> =
	flattenModelCatalog("anthropic", values);
```

(`packages/ai/src/providers/anthropic.models.ts`)

`packages/ai/src/models.generated.ts` aggregates every `<PROVIDER>_MODELS` export into the
`MODELS` map read by `getBuiltinModel()` (`packages/ai/src/providers/all.ts:59`) and
`getBuiltinModels()` (`all.ts:77`).
Both file kinds carry the header `// Do not edit manually - run 'npm run generate-models' to
update`; `biome.json:35-36` excludes `**/models.generated.ts` and `**/*.models.ts` from
lint/format, so a hand edit gets no formatting signal to catch it either.

### Regeneration commands

Root: `npm run generate:models` runs `packages/ai`'s `generate-models` script (`--strict`) then
`generate-image-models` (`package.json:24`). Inside `packages/ai`, `generate-models` is
`node scripts/generate-models.ts --strict`. `npm run hydrate:model-data` (root) /
`hydrate-model-data` (package) runs the same generator with `--data-only`, refreshing
`providers/data/*.json` without touching the generated `.ts` wrappers.

### Fresh checkouts have no `providers/data/`

`providers/data/` is git-ignored (`.gitignore:11`). Root `npm run build` regenerates it for
every package (the `build` script builds `ai` before `agent`/`coding-agent`/`server`,
`package.json`), which is why CI's `Build` step runs before `Check` and `Test`
(`.github/workflows/ci.yml`). `scripts/check-browser-smoke.mjs`'s `generatedCatalogDataPlugin`
(`scripts/check-browser-smoke.mjs:11-24`) resolves any missing `./data/*.json` import to `{}`
specifically so the smoke bundle still builds on a checkout that has not run the generator yet
— the comment on the plugin states this explicitly.

### Image models are a parallel, separately-generated catalog

`packages/ai/scripts/generate-image-models.ts` is a second generator with its own `--strict`
flag, invoked by `generate-image-models` (chained after `generate-models` in the root
`npm run generate:models`). It produces `packages/ai/src/image-models.generated.ts` (609
lines, same "do not edit manually" header) directly — unlike the chat catalog, there is no
per-provider `providers/<name>.images.ts` split, because only one image provider (OpenRouter)
exists today. `image-models.ts` reads `IMAGE_MODELS` from that generated file into a
`Map<provider, Map<modelId, ImagesModel<ImagesApi>>>` for `getImageModel()`/`getImageModels()`/
`getImageProviders()`.

### Validation

`packages/ai/scripts/model-data.ts` defines `MODEL_DATA_SCHEMA_VERSION`, a manifest file
(`.manifest.json`) with a `structureHash`, and `readModelDataProviderIds()`, which parses
`models.generated.ts` with `MODEL_DATA_IMPORT_PATTERN` (`model-data.ts:17-18`) to recover the
provider list straight from the aggregator file — the generator round-trips through its own
generated output as a consistency check. `packages/ai/scripts/check-model-data.ts` (root:
`npm run check:model-data`) runs this validation standalone, outside `npm run check`.
`packages/ai/test/model-data-validation.test.ts` builds a synthetic package root under a temp
dir and exercises `validateModelDataDirectory()` against it.

## Reference Files

- `packages/ai/src/model-catalog.ts` — `flattenModelCatalog()`, `ModelCatalog<TGroups, TProvider>`
- `packages/ai/src/models.generated.ts`, `packages/ai/src/providers/anthropic.models.ts` —
  generated shape (read-only)
- `packages/ai/scripts/generate-models.ts`, `packages/ai/scripts/generate-image-models.ts`,
  `packages/ai/scripts/model-data.ts`, `packages/ai/scripts/check-model-data.ts`
- `scripts/check-browser-smoke.mjs` — `generatedCatalogDataPlugin`
- `packages/ai/test/model-data-validation.test.ts`, `packages/ai/test/model-catalog-types.test.ts`

## Anti-Patterns

| Anti-pattern | Why | Enforcement |
|---|---|---|
| Hand-editing `models.generated.ts` or any `providers/*.models.ts` | Overwritten by the next `npm run generate-models` run | `AGENTS.md`; biome excludes both from lint |
| Committing `providers/data/*.json` | Git-ignored raw data, regenerated per checkout | `.gitignore:11` |
| A provider `.ts` file importing `models.generated.ts` or `providers/all.ts` directly | Defeats per-provider tree-shaking; the browser-smoke agent-treeshake check forbids these as inputs to a single-provider bundle | `scripts/check-browser-smoke.mjs` (see `pi-ai/providers/adding-a-provider.md`) |
