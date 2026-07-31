# Core Guidelines

Provider-agnostic runtime: `packages/ai/src/{types.ts,models.ts,models-store.ts,model-catalog.ts,
models.generated.ts,image-models.ts,images.ts,images-models.ts,images-api-registry.ts,compat.ts,
legacy-api-aliases.ts,env-api-keys.ts}`, `packages/ai/src/api/**`, `packages/ai/src/auth/**`,
`packages/ai/src/utils/**`, and `packages/ai/scripts/generate-models.ts`.

## Pre-Development Checklist

1. Read `_shared/typescript-and-style.md` first — this layer has the repo's densest
   generated-file rules (`models.generated.ts`, every `*.models.ts`).
2. Decide whether the change is provider-agnostic (this layer) or provider-specific
   (`.trellis/spec/pi-ai/providers/`); `types.ts`, `models.ts`, and `model-catalog.ts` are shared
   by every provider, so a change here ripples across all ~36.
3. If touching `Api`, `Model<TApi>`, `ApiOptionsMap`, or a `*Compat` interface, read
   `types-and-compat.md` first — these types drive typed dispatch (`hasApi()`,
   `ApiStreamOptions<TApi>`) across every provider file.
4. If touching model data, generation, or the JSON catalog, read `model-catalog.md` —
   `models.generated.ts` and every `providers/*.models.ts` are generated, never hand-edited.
5. Before adding a new compat flag, `grep -rn "Compat\b" packages/ai/src/types.ts`; most
   auto-detection lives in the `openai-completions`/`openai-responses` API implementations,
   not in this layer.
6. Plan verification: `node ../../node_modules/vitest/dist/cli.js --run test/<file>.test.ts`
   from `packages/ai`, plus `npm run check` (the `check:browser-smoke` step bundles this
   layer's public entrypoints and fails on accidental Node-only imports).

## Guidelines Index

| File | Covers |
|---|---|
| [Model Catalog](./model-catalog.md) | Generated catalog pipeline: `models.generated.ts`, `providers/*.models.ts`, `providers/data/*.json`, `generate-models.ts`, hydration, validation |
| [Types And Compat](./types-and-compat.md) | `Api`/`Model<TApi>`/`ApiOptionsMap` shape, per-API `*Compat` interfaces, the chat/images type split |

## Shared Rules

Repo-wide rules (TypeScript style, testing, `npm run check`, deps/git) live in
[`_shared/index.md`](../../_shared/index.md); this layer does not restate them.
Thinking guides: [`guides/index.md`](../../guides/index.md).
