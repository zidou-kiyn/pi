# Component Model

## When This Applies

Writing or modifying a `Component` in `packages/tui/src/components/*.ts`, or
any code that composes/wraps existing components (e.g. containers with
embedded inputs, custom editor implementations via `EditorComponent`).

## The Local Pattern

### The contract is small and render-only

`Component` (`packages/tui/src/tui.ts:23-46`) requires only `render(width):
string[]` and `invalidate(): void`; `handleInput?(data)` and
`wantsKeyRelease?` are optional. There is no lifecycle hook beyond these —
no mount/unmount, no virtual DOM, no React-style props/state. A component is
a plain class that turns its internal state into an array of strings each
frame `render()` is called; every returned line's `visibleWidth()` must be
`<= width` or the TUI throws (`../rendering/terminal-and-width.md`).

### `Container` is the only composition primitive

`Container` (`tui.ts:211`) holds `children: Component[]` and concatenates
their `render()` output. `Box` (`components/box.ts`) wraps `Container`-like
child rendering with padding and a background function; it is not a
`Container` subclass, it reimplements `addChild`/`removeChild` and adds its
own render cache. There is no other grouping abstraction — new composite
components either extend `Container` or hold an internal array and forward
calls manually, matching the existing files.

### Subclassing an existing component is a valid pattern

`Loader` (`components/loader.ts:17`) `extends Text`, calling `super("", 1,
0)` in its constructor and driving `Text`'s cache-invalidating `setText()`
from a `setInterval` tick to animate a spinner. `CancellableLoader` in turn
extends `Loader` and adds `AbortSignal`/`onAbort` wiring for Escape. Prefer
this over duplicating a component's render logic when the new component is
a strict superset of behavior.

### Render caching is opt-in and manual

Components with expensive or frequently-unchanged output cache their last
`render()` result and invalidate on state or width change:

```ts
// packages/tui/src/components/text.ts
if (this.cachedLines && this.cachedText === this.text && this.cachedWidth === width) {
	return this.cachedLines;
}
```

`Text`, `Box`, `Image` (`components/image.ts:33-34`, `cachedLines`/
`cachedWidth`), and `Markdown` all follow this shape: private `cached*`
fields compared at the top of `render()`, cleared in both `invalidate()` and
any setter that changes rendered content. `TruncatedText` and `SelectList`
have no cache — their `invalidate()` is a documented no-op
(`"// No cached state to invalidate currently"`). Add a cache only when a
component has observable per-render cost; do not add empty cache
scaffolding to new components that do not need it.

### `Markdown` renders source text and exposes one transform seam

`Markdown` (`components/markdown.ts:112`) takes the raw Markdown string plus a
`MarkdownOptions` bag (`components/markdown.ts:98`). The only hook that
rewrites content is `MarkdownOptions.transform`
(`components/markdown.ts:104`):

```ts
transform?: (markdown: string, availableWidth: number) => string;
```

It runs at the top of `render()` — after `contentWidth` is computed as
`width - paddingX * 2`, before the tab normalization and the lexer
(`components/markdown.ts:161`) — so a transformer sees the exact column budget
its output will be wrapped into, and everything downstream (theme, wrapping,
fence handling) still applies to the transformed text.

The render cache keys on the *source* text and `width`
(`components/markdown.ts:155`), never on the transform result. A transformer
must therefore be a pure function of `(markdown, availableWidth)`; one that
depends on wall-clock time or external mutable state will show a stale frame
until the source or the width changes. `test/markdown.test.ts` "caches
transformed Markdown by source and available width" pins this: two renders at
width 80 invoke the transformer once, and only the width change to 60 invokes
it again.

`packages/coding-agent` drives this seam from its extension API — see
`registerMarkdownTransformer` in
[`../../pi-coding-agent/extensions/extension-api.md`](../../pi-coding-agent/extensions/extension-api.md).

For callers that need the token stream rather than rendered lines,
`packages/tui/src/index.ts:3` re-exports `Marked` and the `Token` / `Tokens`
types from `marked`. Import them from `@earendil-works/pi-tui` instead of
adding a direct `marked` dependency, so every consumer parses with the same
version this package renders with.

### `Focusable` is required for any component with a cursor

A component that shows a text cursor and needs correct IME candidate-window
placement implements `Focusable` (`tui.ts:63-66`: `focused: boolean`, set by
the TUI) and emits `CURSOR_MARKER` (`tui.ts:79`, a zero-width APC sequence)
immediately before the cursor position in its rendered line. `Input`
(`components/input.ts:26` declares `focused`) and `Editor` both implement this
directly. A
container that embeds one of them (e.g. a search dialog) must implement
`Focusable` itself and forward `focused` to the child in its setter —
otherwise the hardware cursor is positioned on the container, not the
nested input. See `packages/coding-agent/docs/tui.md` "Container Components
with Embedded Inputs" for the canonical example.

## Scenario: Masked input in an extension TUI

### 1. Scope / Trigger

Use this pattern when an extension must collect a credential or other secret.
`ctx.ui.input()` renders the ordinary `Input` value, and RPC does not support
custom TUI components, so secret entry must be explicitly TUI-only.

### 2. Signatures

```ts
class MaskedInput implements Component, Focusable {
	focused: boolean;
	handleInput(data: string): void;
	render(width: number): string[];
	invalidate(): void;
}

ctx.ui.custom<string | undefined>((tui, theme, keybindings, done) =>
	new MaskedInput(/* ... */),
);
```

### 3. Contracts

- Reuse a private `Input` as the editing engine by calling
  `Input.handleInput()` and `Input.getValue()`.
- Never call the inner `Input.render()` and never include its value in a
  rendered or cached string.
- Render either an empty state or a fixed mask independent of secret length.
- Implement `Focusable`, forward `focused` to the inner input, and emit
  `CURSOR_MARKER` at the visible cursor position.
- Resolve the real value only on confirmation. Replace the inner `Input` on
  settle/dispose so its text, undo history, and kill ring become unreachable.
- Reject print, JSON, and RPC modes before prompting; never fall back to
  ordinary input.

### 4. Validation & Error Matrix

| Condition | Expected handling |
|---|---|
| Mode is not `tui` | Report that interactive TUI mode is required; perform no prompt or write |
| User cancels | Clear the inner input and resolve `undefined` |
| User confirms an empty value | Return it to the caller for secret-free validation |
| Input arrives after settlement | Ignore it |
| Terminal width is zero or narrow | Return lines whose `visibleWidth()` is at most the supplied width |

### 5. Good / Base / Bad Cases

- Good: secrets of different lengths produce identical non-empty rendered
  masks and never appear in terminal captures.
- Base: an empty input renders only the prompt and cursor.
- Bad: using `ctx.ui.input()`, calling `Input.render()`, or rendering one mask
  glyph per secret character.

### 6. Tests Required

- Different-length secrets render the same fixed mask.
- Rendered lines, notifications, captured TUI output, and raw TUI logs do not
  contain the runtime-generated secret.
- Confirm returns the value once; cancel returns `undefined`; later input is
  ignored.
- Width tests include `0`, `1`, and normal terminal widths.
- Non-TUI mode tests prove the component and write path are never reached.

### 7. Wrong vs Correct

Wrong:

```ts
const secret = await ctx.ui.input("API key"); // ordinary input renders it
```

Correct:

```ts
if (ctx.mode !== "tui") return;
const secret = await ctx.ui.custom<string | undefined>((tui, theme, keybindings, done) =>
	new MaskedInput(/* fixed mask; delegates editing only */, done),
);
```

### Editing components share infrastructure, not just conventions

`Editor` (`components/editor.ts`) and `Input` (`components/input.ts`) both
construct their own `KillRing` (`kill-ring.ts`) for Emacs-style kill/yank
and `UndoStack<S>` (`undo-stack.ts`, clone-on-push via `structuredClone`)
for undo. Word-boundary logic is centralized in `findWordBackward`/
`findWordForward` (`word-navigation.ts`), used by both components' Ctrl/Alt
word-movement handlers. New editing components should reuse these three
modules rather than reimplementing kill rings, undo stacks, or word
boundary detection.

### `EditorComponent` is the extension seam for custom editors

`editor-component.ts` defines `EditorComponent extends Component` as an
interface (not the concrete `Editor` class) so that
`packages/coding-agent/src/modes/interactive/components/extension-editor.ts`
and the extension runner (`packages/coding-agent/src/core/extensions/
runner.ts`) can swap in a different editor implementation (vim mode, custom
keybindings) while the rest of the app depends only on the interface. Most
methods beyond `getText`/`setText`/`handleInput` are optional, letting a
minimal replacement implement only what it needs.

### Autocomplete providers are pluggable, not hardcoded into `Editor`

`Editor.setAutocompleteProvider()` takes an `AutocompleteProvider`
(`autocomplete.ts`); the built-in `CombinedAutocompleteProvider` composes
slash-command and file-path completion, with file matching delegated to
`fuzzyFilter()` (`fuzzy.ts`). File-path candidates come from `readdirSync`/
`spawn("fd", ...)` in `autocomplete.ts`, not from `Editor` itself — a new
completion source is a new `AutocompleteProvider`, not a change to `Editor`.

## Reference Files

- `packages/tui/src/tui.ts` — `Component`, `Focusable`, `Container`,
  `CURSOR_MARKER`
- `packages/tui/src/components/text.ts`, `box.ts`, `image.ts` — cache pattern
- `packages/tui/src/components/loader.ts`, `cancellable-loader.ts` —
  subclassing pattern
- `packages/tui/src/components/input.ts`, `editor.ts` — `Focusable`,
  `KillRing`/`UndoStack` usage
- `packages/tui/src/kill-ring.ts`, `undo-stack.ts`, `word-navigation.ts`
- `packages/tui/src/components/markdown.ts` — `MarkdownOptions.transform`,
  render cache keys
- `packages/tui/src/editor-component.ts`, `autocomplete.ts`, `fuzzy.ts`
- `packages/coding-agent/src/modes/interactive/components/extension-editor.ts`
- `packages/tui/test/select-list.test.ts`, `input.test.ts`, `editor.test.ts`,
  `autocomplete.test.ts`, `fuzzy.test.ts`, `word-navigation.test.ts`,
  `markdown.test.ts`

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| A `MarkdownOptions.transform` that is not a pure function of `(markdown, availableWidth)` | The render cache keys on source + width, so the stale result is served until one of them changes | `components/markdown.ts:155`; `test/markdown.test.ts` transform caching test |
| Adding a direct `marked` dependency in a downstream package | Two `marked` versions parse and render differently; `pi-tui` already re-exports `Marked`/`Token`/`Tokens` | `packages/tui/src/index.ts:3` |
| Adding lifecycle hooks (mount/unmount) or React/DOM vocabulary to a component | Not how this renderer works; `render(width)` is the entire contract | `tui.ts:23-46` |
| Reimplementing kill/yank or undo inside a new editing component | `KillRing`/`UndoStack` already exist and are shared by `Editor` and `Input` | `kill-ring.ts`, `undo-stack.ts` |
| Emitting `CURSOR_MARKER` from a container without propagating `focused` to the embedded input | IME candidate window is positioned on the wrong component | `packages/coding-agent/docs/tui.md` "Container Components with Embedded Inputs" |
| Adding a render cache with fields that are never compared or cleared | Dead code; `TruncatedText`/`SelectList` intentionally have no cache instead | `components/truncated-text.ts`, `components/select-list.ts` |
