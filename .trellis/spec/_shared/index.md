# Shared Project Rules

> Repo-wide rules for the pi monorepo. Every package layer under
> `.trellis/spec/<package>/` links here instead of restating these rules.

## Scope

Applies to all workspaces listed in the root `package.json`:
`packages/{agent,ai,coding-agent,evals,server,tui}`,
`packages/storage/sqlite-node`, and the extension examples under
`packages/coding-agent/examples/extensions/`.

`AGENTS.md` at the repo root is the authoritative rulebook and is loaded
automatically by agents started from the repo root. This layer does not replace
it — it adds the repository evidence behind each rule: the enforcing script,
the config file, the proving test, and the real code paths. Where the two ever
disagree, `AGENTS.md` wins and this spec must be corrected.

Rules that are purely procedural (release flow, issue/PR etiquette, GitHub
labels, changelog attribution) are **not** duplicated here. Read `AGENTS.md`
and `CONTRIBUTING.md` for those.

## Pre-Development Checklist

1. Read the task's `prd.md`, then `design.md` and `implement.md` when present.
2. Read this layer's `typescript-and-style.md` before writing any `.ts`.
3. Read `testing.md` before adding or running any test. Never run the raw
   vitest suite.
4. Identify the package you are touching and read
   `.trellis/spec/<package>/<layer>/index.md` for its local conventions.
5. Search before you add: grep for an existing helper, constant, or pattern
   (`guides/code-reuse-thinking-guide.md`).
6. If the change crosses a layer boundary (event kind, JSONL record, RPC
   payload, config field), read `guides/cross-layer-thinking-guide.md` first.
7. Plan the verification: `npm run check` for code changes, plus the specific
   test file you touched.
8. Confirm the change belongs in core at all — `CONTRIBUTING.md` states pi's
   core is deliberately minimal and that features which do not belong in core
   should be extensions.

## Guidelines Index

| Guide | Covers |
|---|---|
| [TypeScript And Style](./typescript-and-style.md) | `.ts` extension imports, erasable syntax, biome formatting, `any` policy, dynamic imports |
| [Testing](./testing.md) | `./test.sh`, faux provider, coding-agent suite contract, vitest configs, tmux TUI checks |
| [Checks And Commands](./checks-and-commands.md) | The seven steps of `npm run check`, pre-commit, CI parity |
| [Dependencies And Git](./dependencies-and-git.md) | Pinned versions, lockfile guards, install-script allowlists, multi-session git rules |

## Package Layers

| Package | Layers |
|---|---|
| pi-agent-core (`packages/agent`) | `agent-loop`, `harness` |
| pi-ai (`packages/ai`) | `core`, `providers` |
| pi-coding-agent (`packages/coding-agent`) | `core`, `modes`, `extensions` |
| pi-evals (`packages/evals`) | `evals` |
| pi-server (`packages/server`) | `server` |
| pi-storage-sqlite-node (`packages/storage/sqlite-node`) | `storage` |
| pi-tui (`packages/tui`) | `rendering`, `components` |

Discover them at any time with:

```bash
python3 ./.trellis/scripts/get_context.py --mode packages
```

Not covered by spec, on purpose: `packages/storage` (container directory with
no `src`) and `packages/coding-agent/examples/extensions/*` (documentation
samples). Read the extension examples as reference material via
`.trellis/spec/pi-coding-agent/extensions/`.

## Thinking Guides

Always-on companions, read alongside this layer:
[`guides/index.md`](../guides/index.md) —
[code reuse](../guides/code-reuse-thinking-guide.md),
[cross-layer](../guides/cross-layer-thinking-guide.md), and the AI
cross-review false-positive checklist.

## Working Style

Not restated here. `AGENTS.md` "Conversational Style" and "Code Quality" own
these rules (answer before editing, state agreement explicitly, read files in
full before wide changes, ask before removing intentional code, no emojis).
This layer adds evidence only where a rule has an enforcing script, config
file, or proving test behind it — see the four guides above.
