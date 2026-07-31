# Interactive TUI

## When This Applies

Working in `packages/coding-agent/src/modes/interactive/**`
(`interactive-mode.ts`, `components/`, `theme/`) or `src/cli/startup-ui.ts`.

## The Local Pattern

### One `InteractiveMode` class composes `pi-tui` `Container`s

`InteractiveMode` (`interactive-mode.ts:345`) owns the entire screen tree as
`Container` fields (`chatContainer`, `statusContainer`, `editorContainer`,
`headerContainer`, `widgetContainerAbove`/`Below`, ...) built from
`@earendil-works/pi-tui` primitives, plus one `KeybindingsManager` instance
shared with custom editors, selectors, and extension UI
(`interactive-mode.ts:362`, `494`). There is no per-feature controller class;
new interactive features are new private state plus new methods on this one
class, or a new `Component` under `components/`.

### `createInteractiveTui` picks the screen implementation, not the caller

```ts
export function createInteractiveTui(options: InteractiveTuiOptions): TUI {
	const terminal = options.terminal ?? new ProcessTerminal();
	if (options.alt) {
		return new TuiAltScreen(terminal, options.showHardwareCursor, options.logDirectory, { openUrl: openBrowser });
	}
	return new TuiMainScreen(terminal, options.showHardwareCursor, options.logDirectory);
}
```

(`interactive-mode.ts:337-343`.) `TuiAltScreen` (alternate screen buffer,
`--alt`) versus `TuiMainScreen` (default inline TUI) is selected by a boolean
option, not autodetected from the terminal.

### Session rebind mirrors print/RPC mode, without a shared base class

`InteractiveMode` installs `runtimeHost.setRebindSession(...)`
(`interactive-mode.ts:477`), which calls `session.bindExtensions(...)`
(`interactive-mode.ts:1665`) and then `session.subscribe(...)`
(`interactive-mode.ts:2876`) to re-attach the TUI whenever the current
`AgentSession` changes (`/new`, `/fork`, session switch). `print-mode.ts` and
`rpc-mode.ts` implement the identical rebind → `bindExtensions` →
`subscribe` sequence against the same `AgentSessionRuntime`
(see `../../pi-coding-agent/modes/rpc-and-print.md`) — the shape is shared by
convention across the three modes, not by inheritance.

### Keybindings are resolved by namespaced id, never by literal key match

`core/keybindings.ts` defines `AppKeybindings` (ids like `app.model.select`,
`app.tools.expand`) and a `KEYBINDINGS` default map merged with
`TUI_KEYBINDINGS` from `@earendil-works/pi-tui`. `interactive-mode.ts` checks
a pressed key against a registered action with
`matchesKey(data, shortcutStr as KeyId)` (`interactive-mode.ts:1863`), and
`KeybindingsManager.create()` (`interactive-mode.ts:494`) loads user
overrides from `~/.pi/agent/keybindings.json`.
`packages/coding-agent/docs/keybindings.md` documents every id and its
default; `migrateKeybindingsConfig` (invoked from `src/migrations.ts`)
upgrades pre-namespaced ids such as `cursorUp`.

### Tool rendering goes through `ToolExecutionComponent`, not ad-hoc strings

`components/tool-execution.ts` `ToolExecutionComponent` (`tool-execution.ts:13`)
reads `renderShell` off the tool's `ToolDefinition` — falling back to a
built-in definition, then `"default"` — and builds a `ToolRenderContext`
(`getRenderContext`, `tool-execution.ts:115`) so a tool like `edit` can own
its full render lifecycle (`renderCall`/`renderResult`) inside the chat
container. `test/edit-tool-no-full-redraw.test.ts` renders through this
component with a `FakeTerminal` to assert updates stay incremental instead of
forcing a full-screen clear.

## Reference Files

- `packages/coding-agent/src/modes/interactive/interactive-mode.ts`
- `packages/coding-agent/src/modes/interactive/components/tool-execution.ts`
- `packages/coding-agent/src/core/keybindings.ts`
- `packages/coding-agent/docs/keybindings.md`
- `packages/coding-agent/test/edit-tool-no-full-redraw.test.ts`
- `AGENTS.md` ("Never hardcode key checks...")

## Anti-Patterns

- `matchesKey(keyData, "ctrl+x")` with a literal string instead of a
  namespaced id registered in `KEYBINDINGS`/`AppKeybindings` — explicitly
  forbidden by `AGENTS.md`.
- Adding session-lifecycle handling inside `InteractiveMode` that does not
  mirror the `rebindSession` → `bindExtensions` → `subscribe` triad used by
  `print-mode.ts`/`rpc-mode.ts` — produces state that resyncs correctly in
  two modes but not the third.
- Rendering a tool's output as a plain string when its `ToolDefinition`
  declares `renderShell: "self"` — bypasses the incremental update path
  `ToolExecutionComponent` provides and other coding tools rely on.
