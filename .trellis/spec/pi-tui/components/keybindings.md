# Keybindings

## When This Applies

Adding, changing, or checking any keyboard shortcut inside `packages/tui` or
in a downstream package that consumes `@earendil-works/pi-tui` keybindings
(e.g. `packages/coding-agent/src/core/keybindings.ts`).

## The Local Pattern

### The add-a-binding flow in this package

A new TUI-level action is added in `packages/tui/src/keybindings.ts` in two
places that must stay in sync:

1. Add the id to the `Keybindings` interface (`keybindings.ts:7-47`), e.g.
   `"tui.editor.cursorUp": true;`.
2. Add a matching entry to `TUI_KEYBINDINGS` (`keybindings.ts:59-...`) with
   `defaultKeys: KeyId | KeyId[]` and a `description`:

```ts
"tui.editor.cursorLeft": {
	defaultKeys: ["left", "ctrl+b"],
	description: "Move cursor left",
},
```

`Keybinding = keyof Keybindings` (`keybindings.ts:49`) is what
`KeybindingsManager` methods are typed against, so both edits are required
for the id to type-check anywhere it is used.

The component then checks the binding with the shared manager, never with a
literal key string:

```ts
// packages/tui/src/components/select-list.ts:113-115
const kb = getKeybindings();
if (kb.matches(keyData, "tui.select.up")) { ... }
```

`getKeybindings()`/`setKeybindings()` (`keybindings.ts:244-253`) hold a
module-level singleton `KeybindingsManager`; the host application calls
`setKeybindings(new KeybindingsManager(KEYBINDINGS, userBindings))` once at
startup so every component in the tree resolves against the same
(possibly user-remapped) bindings.

### `KeybindingsManager` resolves defaults vs. user overrides per-id

`rebuild()` (`keybindings.ts:176`) computes, for each id, `userBindings[id]`
if present else `definition.defaultKeys` — rebinding one action's keys does
not evict another action's default that happens to reuse the same key
unless the user's config explicitly claims it too. Conflicts are only
reported for keys claimed by *multiple user-supplied* bindings
(`getConflicts()`), proven by
`packages/tui/test/keybindings.test.ts` ("does not evict cursor bindings
when another action reuses the same key" / "still reports direct user
binding conflicts without evicting defaults").

### Downstream packages extend via declaration merging, not a fork

`packages/coding-agent/src/core/keybindings.ts` adds app-level ids
(`app.interrupt`, `app.model.select`, ...) by declaration-merging into the
same `Keybindings` interface:

```ts
declare module "@earendil-works/pi-tui" {
	interface Keybindings extends AppKeybindings {}
}

export const KEYBINDINGS = {
	...TUI_KEYBINDINGS,
	"app.interrupt": { defaultKeys: "escape", description: "Cancel or abort" },
	// ...
};
```

It spreads `TUI_KEYBINDINGS` into its own `KEYBINDINGS` map rather than
duplicating pi-tui's ids by hand, and the merged, user-configurable table
is what ends up documented for end users in
`packages/coding-agent/docs/keybindings.md`.

### The existing exceptions to "no hardcoded key checks"

`AGENTS.md` forbids `matchesKey(keyData, "ctrl+x")` for user-facing actions.
Six literal `matchesKey()` calls exist outside `keys.ts`/`keybindings.ts`
today; two are documented design exceptions, four are unguarded precedent that
should not be extended.

Design exceptions:

- `packages/tui/src/tui.ts:804` — `matchesKey(data, "shift+ctrl+d")` is a
  fixed global debug trigger (`onDebug`), not a `Keybindings` entry.
- `packages/tui/src/components/editor.ts:748,752` — `matchesKey(data,
  "shift+backspace")` / `"shift+delete"` are ORed *in addition to* the
  configurable `kb.matches(data, "tui.editor.deleteCharBackward" /
  "deleteCharForward")` check, as fixed terminal-specific key variants of an
  already-configurable action, not a new unconfigurable action.

Unguarded precedent (current behavior, recorded as debt — do not copy):

- `packages/tui/src/components/editor.ts:876` — `"shift+space"`
- `packages/tui/src/components/editor.ts:1251` — `"enter"`
- `packages/coding-agent/src/modes/interactive/components/config-selector.ts:491`
  — `"ctrl+c"`

A new component action needs a `Keybindings` entry and `kb.matches()`, matching
every other call site in `select-list.ts`, `editor.ts`, `input.ts`.

## Reference Files

- `packages/tui/src/keybindings.ts` — `Keybindings`, `TUI_KEYBINDINGS`,
  `KeybindingsManager`, `getKeybindings`/`setKeybindings`
- `packages/tui/src/components/select-list.ts:113-135` — `kb.matches()` call
  sites for a simple component
- `packages/tui/src/components/editor.ts:604-870` — `kb.matches()` call
  sites for the largest component, plus the two `matchesKey()` exceptions
- `packages/coding-agent/src/core/keybindings.ts` — downstream declaration
  merging and `KEYBINDINGS` composition
- `packages/coding-agent/docs/keybindings.md` — user-facing id/default table
  and `~/.pi/agent/keybindings.json` override format
- `packages/tui/test/keybindings.test.ts` — conflict and override resolution
  behavior

## Anti-Patterns

| Anti-pattern | Why | Enforcement |
|---|---|---|
| `matchesKey(data, "ctrl+x")` for a new user-facing action | Not configurable; bypasses user remapping | `AGENTS.md` Code Quality; convention only, reviewers reject |
| Adding a `TUI_KEYBINDINGS` entry without adding the id to `Keybindings` | `Keybinding` type (`keyof Keybindings`) won't include it; `kb.matches()` call site fails to type-check | `keybindings.ts:49` |
| Forking `TUI_KEYBINDINGS` entries into a downstream package's own table instead of spreading it | Two sources of truth drift on defaults | `packages/coding-agent/src/core/keybindings.ts:65` spreads instead of forking |
