# Extensions Guidelines

Governs `packages/coding-agent/src/extensions/**` (built-in bundled
extensions) and `packages/coding-agent/src/core/extensions/**` (the loader,
runner, and `ExtensionAPI`/`ToolDefinition` types that both built-in and
third-party extensions use).
`packages/coding-agent/examples/extensions/**` is reference material for this
layer — worked examples of the same API — not itself a spec target
(`.trellis/spec/_shared/index.md` "Package Layers").

## Pre-Development Checklist

1. Read `.trellis/spec/_shared/index.md` first.
2. Read `packages/coding-agent/docs/extensions.md` end to end before adding a
   new event, `ExtensionAPI` method, or `ToolDefinition` field — it is the
   user-facing contract for everything in `core/extensions/types.ts`.
3. Confirm the feature belongs in core at all. `CONTRIBUTING.md` states pi's
   core is deliberately minimal; a feature that only a subset of users need
   should usually be an extension, not a `core/` addition
   (`_shared/index.md` Pre-Development Checklist item 8).
4. Find an existing example under `packages/coding-agent/examples/extensions/`
   closest to the behavior you are adding or documenting; the README table
   there is the index of "what pattern already exists" — read it before
   inventing a new one.
5. If you add a new `ExtensionEvent` variant, update `core/extensions/types.ts`,
   the corresponding `emit*` method in `core/extensions/runner.ts`, the
   `AgentSession` call site that fires it, and `docs/extensions.md` in the
   same change.
6. Plan verification: `npm run check`, plus the closest extension-loading
   regression test under `packages/coding-agent/test/suite/regressions/`
   (see `extension-api.md` Reference Files).

## Guidelines Index

| File | Covers |
|---|---|
| [Extension API](./extension-api.md) | Loader discovery order, event dispatch (`block`/short-circuit), `ToolDefinition`/`defineTool`, bundled virtual modules |

## Shared Rules

Repo-wide rules (TypeScript/style, testing, `npm run check`, deps/git):
[`.trellis/spec/_shared/index.md`](../../_shared/index.md).

Generic thinking aids (cross-layer, code reuse):
[`.trellis/spec/guides/index.md`](../../guides/index.md).
