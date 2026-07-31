# Components Guidelines

Covers the built-in `Component` implementations and the input/editing
infrastructure they share: `packages/tui/src/components/*.ts`,
`editor-component.ts`, `autocomplete.ts`, `fuzzy.ts`, `keys.ts`,
`keybindings.ts`, `kill-ring.ts`, `undo-stack.ts`, `stdin-buffer.ts`,
`word-navigation.ts`.

## Pre-Development Checklist

1. Read `../../_shared/typescript-and-style.md` and `../../_shared/testing.md`
   first, then `../rendering/terminal-and-width.md` if the component renders
   text (every component must respect `visibleWidth(line) <= width`).
2. Search `packages/tui/src/keybindings.ts` `TUI_KEYBINDINGS` before wiring
   any keyboard shortcut. Never call `matchesKey(data, "literal")` for a
   user-facing action — see `keybindings.md` for the required flow and the
   two narrow exceptions that already exist in this codebase.
3. If the component edits text, check whether `KillRing`
   (`packages/tui/src/kill-ring.ts`) or `UndoStack`
   (`packages/tui/src/undo-stack.ts`) already covers the behavior — both
   `Editor` and `Input` share these instead of each reimplementing kill/yank
   or undo.
4. If the component needs a cursor and IME support, implement `Focusable`
   (`focused: boolean` + `CURSOR_MARKER` in output) per
   `component-model.md`; container components with an embedded `Input`/
   `Editor` must propagate `focused` to the child.
5. Add or update the matching `*.test.ts` in `packages/tui/test/`. Most
   existing component tests use plain `node:test` (`import { describe, it }
   from "node:test"`), not vitest — match the existing file's test runner.
6. Run the file directly, e.g.
   `node --experimental-strip-types --test packages/tui/test/select-list.test.ts`,
   since `packages/tui/vitest.config.ts` does not auto-include it.

## Guidelines Index

| File | Covers |
|---|---|
| [Component Model](./component-model.md) | `Component`/`Focusable`/`Container` contracts, render caching, composition (`Loader extends Text`), autocomplete provider pattern |
| [Keybindings](./keybindings.md) | `TUI_KEYBINDINGS` registry, `KeybindingsManager.matches()`, the add-a-binding flow, downstream extension via declaration merging |

## Shared Rules

- Repo-wide TypeScript, testing, and check conventions:
  [`../../_shared/index.md`](../../_shared/index.md)
- Generic thinking guides (cross-layer, code reuse):
  [`../../guides/index.md`](../../guides/index.md)
