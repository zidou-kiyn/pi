# TypeScript And Style

## When This Applies

Every `.ts` file under `packages/*/src`, `packages/*/test`,
`packages/storage/*/src`, `packages/storage/*/test`, and
`packages/coding-agent/examples`. That is the exact `include` list in the root
`tsconfig.json`, mirrored by `biome.json` `files.includes`.

## The Local Pattern

### Relative imports must carry the `.ts` extension

The repo compiles with `allowImportingTsExtensions` +
`rewriteRelativeImportExtensions` (`tsconfig.base.json`). Source files import
each other by their real `.ts` path:

```ts
import { runAgentLoop, runAgentLoopContinue } from "./agent-loop.ts";
import { resolvePath } from "../utils/paths.ts";
```

A relative specifier ending in `.js` is a build failure, not a style
preference: `scripts/check-ts-relative-imports.mjs` walks every `.ts` file in
the repo and fails on `import`, `export ... from`, dynamic `import()`, and
`import("...")` type nodes whose specifier matches `^\.\.?/` and ends in `.js`.
Cross-package imports use the workspace names mapped in `tsconfig.json`
`paths` (`@earendil-works/pi-ai`, `@earendil-works/pi-tui`, ...), never deep
relative paths across package boundaries.

### Erasable syntax only

`tsconfig.base.json` sets `erasableSyntaxOnly: true`, so `tsgo --noEmit` (part
of `npm run check`) rejects parameter properties, `enum`, `namespace`/`module`,
`import =`, and `export =`. Classes declare explicit fields and assign them in
the constructor.

### Type imports are separated

Type-only imports use `import type { ... }`, kept as their own statement even
when the same module also provides values — see the two import blocks at the
top of `packages/agent/src/agent.ts`.

### Formatting is not negotiable

`biome.json` `formatter`: tabs, `indentWidth: 3`, `lineWidth: 120`. Do not
hand-format; `npm run check` runs `biome check --write`, so a mismatched file
is silently rewritten and shows up as unexpected diff noise.

### Generated files are off-limits to hand edits

`biome.json` excludes `**/models.generated.ts`, `**/*.models.ts`, and
`**/test-sessions.ts`. `packages/ai/src/models.generated.ts` is regenerated
from `packages/ai/scripts/generate-models.ts` via `npm run generate:models`;
editing the generated file directly is always wrong.

## Reference Files

- `biome.json` — lint rules, formatter settings, covered file globs
- `tsconfig.base.json` — `strict`, `erasableSyntaxOnly`, `target: ES2022`
- `tsconfig.json` — workspace `paths` aliases, `include` / `exclude`
- `scripts/check-ts-relative-imports.mjs` — the `.js` specifier check
- `packages/agent/src/agent.ts` — import layout, `import type` separation
- `packages/coding-agent/src/core/agent-session-services.ts` — relative
  `.ts` imports inside a package

## Anti-Patterns

| Anti-pattern | Why | Enforcement |
|---|---|---|
| `from "./foo.js"` in a `.ts` file | Breaks the TS-extension import model | `npm run check:ts-imports` (hard fail) |
| `enum`, `namespace`, parameter properties | Needs JS emit; repo runs Node strip-only | `tsgo --noEmit` (hard fail) |
| Editing `packages/ai/src/models.generated.ts` | Overwritten by the generator | Convention; reviewers reject |
| An ad-hoc `await import(...)` for an ordinary module | Hides the dependency graph; the sanctioned lazy path is a `*.lazy.ts` wrapper | Convention only — not machine-checked |
| Loosening a type to silence an error from an outdated dep | Masks a real upgrade | Convention; upgrade the dep instead |

### `any` is a convention rule, not a lint rule

`biome.json` sets `suspicious.noExplicitAny: "off"`. The repo currently has
~112 `any` occurrences across 28 source files, mostly in generic parameter
positions such as `satisfies Model<any>` (`packages/agent/src/agent.ts`).
`AGENTS.md` still forbids new `any` unless unavoidable — that rule is enforced
by reviewers, not tooling, so introducing one needs an explicit justification
in the PR or task notes.

### Dynamic `import()` is confined to two sanctioned shapes

`AGENTS.md` bans inline imports; in practice the repo allows dynamic `import()`
only where deferring the module load is the point.

**1. `*.lazy.ts` wrappers in `packages/ai` (the documented pattern).** Eleven
files under `packages/ai/src/api/*.lazy.ts` wrap an SDK-heavy implementation as
`lazyApi(() => import("./<id>.ts"))`, so a provider factory can be statically
imported without pulling its SDK into the bundle. `lazyOAuth` in
`packages/ai/src/auth/helpers.ts` does the same for OAuth flows. This shape is
required for new provider APIs — see
`.trellis/spec/pi-ai/providers/adding-a-provider.md`.

**2. Node-only capability probes behind an indirection.**
`packages/ai/src/auth/context.ts:12` and `packages/ai/src/env-api-keys.ts:8`
both define a one-line `(specifier) => import(specifier)` so browser bundles
never statically reference `node:*`.

Outside those two shapes, bare `await import(...)` exists at exactly three
places, all runtime entry points or optional native deps:

- `packages/ai/src/api/openrouter-images.lazy.ts:5` — lazy image API
- `packages/coding-agent/src/bun/cli.ts:14-15` — Bun entry shim
- `packages/coding-agent/src/utils/photon.ts:128` — optional native dependency

A new ad-hoc dynamic import needs the same class of justification. Type-position
`typeof import("node:fs")` is a separate thing and is used freely for Node type
references in browser-safe modules (`env-api-keys.ts:2-4`).
