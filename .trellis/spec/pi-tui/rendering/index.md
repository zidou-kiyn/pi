# Rendering Guidelines

Covers the terminal renderers and their support modules: `packages/tui/src/tui.ts`,
`TuiMainScreen.ts`, `TuiAltScreen.ts`, `terminal.ts`, `terminal-colors.ts`,
`terminal-image.ts`, `utils.ts`, `native-modifiers.ts`, `native/`.

## Pre-Development Checklist

1. Read `../../_shared/typescript-and-style.md` and `../../_shared/testing.md` first.
2. Identify whether the change affects line width accounting (`utils.ts`
   `visibleWidth` / `truncateToWidth` / `wrapTextWithAnsi`), differential
   rendering (`TuiMainScreen.doRender`, `TuiAltScreen`), or terminal I/O
   (`terminal.ts`, `native-modifiers.ts`).
3. If it touches width accounting, find the regression test that already
   covers the character class you are changing (CJK, emoji, regional
   indicators, tabs, ANSI) before editing — see `terminal-and-width.md`.
   `graphemeWidth` in `packages/tui/src/utils.ts:173` is the single source of
   truth; do not duplicate width logic elsewhere.
4. Every `Component.render(width)` implementation must return lines whose
   `visibleWidth(line) <= width`; `TuiMainScreen`/`TuiAltScreen` throw
   `Rendered line exceeds terminal width` otherwise. Check any new render path
   against this before wiring it in.
5. Run the specific test files you touch with `node --test` or vitest by
   explicit filename — `packages/tui/vitest.config.ts` only auto-includes
   `test/wrap-ansi.test.ts` (see `../../_shared/testing.md` config table).
   Most files under `packages/tui/test/*.test.ts` use `node:test`, not
   vitest; run them with
   `node --experimental-strip-types --test packages/tui/test/<file>.test.ts`.
6. Verify with `npm run check` from the repo root after any `.ts` edit.

## Guidelines Index

| File | Covers |
|---|---|
| [Terminal And Width](./terminal-and-width.md) | Grapheme width table, ANSI/OSC/APC parsing, differential rendering strategies, image-line handling, native modifier detection |

## Shared Rules

- Repo-wide TypeScript, testing, and check conventions:
  [`../../_shared/index.md`](../../_shared/index.md)
- Generic thinking guides (cross-layer, code reuse):
  [`../../guides/index.md`](../../guides/index.md)
