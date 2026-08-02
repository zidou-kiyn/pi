# PRD: Add pi-patty-bg-tasks to the personal preset

## Goal

Ship `pi-patty-bg-tasks` (Claude Code style background tasks: auto-background at 120s, Ctrl+B, `jobs`/`monitor`/`agent_bg`, `/bg-list`) as part of `zidou-kiyn/pi-preset`, and resolve its `bash` tool-override conflict with `pi-tool-display` so both extensions work as intended on every platform.

## Confirmed Facts (verified in repo / installed environment)

- Preset lives at `~/pi-preset` (public repo, installed as `git:github.com/zidou-kiyn/pi-preset`). Desired state is centralized in `~/pi-preset/src/manifest.ts`; `REQUIRED_PACKAGES` currently has 13 entries and does not include `pi-patty-bg-tasks` (`src/manifest.ts:24-38`).
- `/preset-sync` is diff-first, confirm-before-write, idempotent: `src/plan.ts` is read-only and computes steps, `src/apply.ts` is the single side-effecting point.
- Existing step kinds: `settings.packages.add`, `webSearch.patch`, `footer.demote`, `font.install` (`src/plan.ts:139-176`). There is **no** step kind that patches an extension's own `config.json` today.
- `WEB_SEARCH_PATCH` shows the established pattern for a leaf-level deep merge that never overwrites the surrounding file (`src/manifest.ts:47-50`, `planWebSearch` in `src/plan.ts`).
- The tool-override conflict is real on **all** platforms, not only Windows, and it is **fatal, not a precedence question** (corrected during implementation after sandbox reproduction on pi 0.83.0). `detectExtensionConflicts` records a duplicate tool name as a load **error** (`packages/coding-agent/src/core/resource-loader.ts:1058-1078`), and `main.ts:844-848` exits 1 when any diagnostic is an error. Observed in a clean sandbox with only these two packages installed:
  ```
  Error: Failed to load extension ".../pi-patty-bg-tasks/index.ts": Tool "bash" conflicts with .../pi-tool-display/index.ts
  Hint: Start without extensions using "pi -ne".
  ```
  pi does not start at all. The runtime first-wins merge (`extensions/runner.ts:450-461`) never gets a chance to apply, and reordering `packages[]` cannot fix it — only the opt-out can.
- `pi-tool-display` exposes the official opt-out `registerToolOverrides.bash: false`, read from `$PI_CODING_AGENT_DIR/extensions/pi-tool-display/config.json` (default `~/.pi/agent/extensions/pi-tool-display/config.json`) — confirmed in the installed package README (`README.md:134-135`) and `src/config-store.ts:24`. The local file currently has `"bash": true`.
- The preset's `getAgentDir()` already resolves that same directory (`src/paths.ts:34-38`, `getUserExtensionsDir`), so no new path resolver is needed. `keybindings.json` sits in the same `getAgentDir()`, so it needs no new resolver either.
- The write primitive already satisfies the safety requirements: `writeJsonObjectAtomic` creates missing parent directories, preserves an existing file's mode, writes a `.preset-bak` backup, resolves symlinks instead of replacing them, and renames atomically (`src/json-merge.ts:213-240`). `readJsonObject` treats a missing file as `{}` and refuses to overwrite non-object or invalid JSON (`src/json-merge.ts:56-80`). `deepMerge` + `flattenLeaves` + `getPath` already give per-leaf diffing (`src/json-merge.ts:90-128`), exactly what `planWebSearch` uses.
- `pi-patty-bg-tasks` requires Pi v0.37+, has no external dependencies and no tmux; installable as `npm:pi-patty-bg-tasks`.
- Ctrl+B startup warning, root cause confirmed: the extension registers `ctrl+b` unconditionally with no config knob (`package/src/shortcuts.ts:28`, npm 1.1.6), and pi's default `tui.editor.cursorLeft` is `["left", "ctrl+b"]` (`packages/tui/src/keybindings.ts:63`). `tui.editor.cursorLeft` is **not** in `RESERVED_KEYBINDINGS_FOR_EXTENSION_CONFLICTS` (`packages/coding-agent/src/core/extensions/runner.ts:71-90`), so `restrictOverride === false`: the shortcut is **not** skipped, the extension wins, and pi emits the diagnostic `Extension shortcut conflict: 'ctrl+b' is built-in shortcut for tui.editor.cursorLeft and <path>. Using <path>.` on every startup (`runner.ts:519-524`). Functionality is unaffected; only the warning is noise.
- User keybindings **replace** an action's whole key list rather than merging (`packages/tui/src/keybindings.ts:196-200`), so `{"tui.editor.cursorLeft": ["left"]}` in `$PI_CODING_AGENT_DIR/keybindings.json` (default `~/.pi/agent/keybindings.json`, `packages/coding-agent/src/core/keybindings.ts:349`) removes `ctrl+b` from the builtins and silences the warning. That file does not exist on this machine today.
- Only `ctrl+b` collides. The extension's other shortcuts (`ctrl+shift+b`, `ctrl+shift+j`, `shift+down`, `ctrl+shift+x`) match no built-in keybinding.

## Key Decisions

- **D4 (user-requested scope addition):** also ship `npm:pi-markdown-preview` (LaTeX/math rendering, `/preview*`, PDF export). Conflict-checked against the other two (npm 0.11.1 source + sandbox with all three installed on pi 0.83.0): its only tool is `preview_export` (`index.ts:4383`), its only commands are `/preview*`, and it registers **no** global shortcuts, so there is no tool-name collision and no keybinding warning. Its runtime prerequisites (pandoc, Chromium browser, optional LaTeX engine) are system tools the preset documents but does not install.

- **D1 (user-confirmed):** the preset **actively writes** the `pi-tool-display` opt-out (option A), as a manifest-driven step with the same diff-first / confirm / idempotent contract as `webSearch.patch` — not a warn-only note. Accepted consequence: if the user later flips `registerToolOverrides.bash` back to `true` by hand, every `/preset-sync` will offer to flip it back to `false`.
- **D2 (user-confirmed):** the preset also **actively writes** `keybindings.json` with `"tui.editor.cursorLeft": ["left"]` to silence the recurring Ctrl+B conflict warning. Accepted consequences: the emacs-style `ctrl+b` = cursor-left binding is given up (the `left` key remains), the preset now owns a third config file, and a hand-restored `ctrl+b` will be offered for removal on every sync.
- **D3:** since three steps now share one shape (path + deep-merge patch + per-leaf diff), `webSearch.patch` should generalize into one manifest-driven JSON patch step rather than growing two near-duplicate step kinds. This keeps `renderPlan`, `apply`, and idempotence logic single-sourced.

## Requirements

- **R1** Add `npm:pi-patty-bg-tasks` and `npm:pi-markdown-preview` to `REQUIRED_PACKAGES` so `/preset-sync` installs them and `pi update --extensions` keeps them updated.
- **R2** Ensure `pi-patty-bg-tasks` owns the `bash` tool by making the preset turn off `pi-tool-display`'s `registerToolOverrides.bash`, using the same diff-first / confirm-before-write / idempotent contract as the other steps.
- **R3** The write must be a leaf-level deep merge into the existing `pi-tool-display` `config.json`: every other key (`readOutputMode`, `diff*`, `bashOutputMode`, ...) is preserved, and a missing file or missing directory is handled without failing the whole sync.
- **R4** No credential, private host, or personal preference enters the preset (existing repo invariant).
- **R5** README documents the new extension, the conflict, and what the preset changes on the user's behalf, including how to revert (`registerToolOverrides.bash: true` + remove the package).
- **R6** Existing `/preset-sync` behavior (packages, web-search, footer demote, font) is unchanged; a fully synced machine still reports "nothing to do".
- **R7** The preset sets `tui.editor.cursorLeft` to `["left"]` in `keybindings.json` through the same patch contract, preserving every other action the user has customized there, and handles the file not existing yet.
- **R8** README also documents the Ctrl+B warning, why the preset removes the built-in `ctrl+b` binding, and how to restore it (`"tui.editor.cursorLeft": ["left", "ctrl+b"]`, accepting the startup warning).

## Acceptance Criteria

- **AC1** In a sandbox (`PI_CODING_AGENT_DIR=<tmp>`), a first `/preset-sync` run shows both `+ settings.json packages[]: add ... npm:pi-patty-bg-tasks` and the `pi-tool-display` `registerToolOverrides.bash: true -> false` diff before any write; declining leaves both files byte-identical.
- **AC2** Accepting writes both changes; a second run reports no remaining steps for them (idempotent).
- **AC3** A pre-existing `pi-tool-display/config.json` keeps every unrelated key and its file permissions after the patch.
- **AC4** In real Pi after sync, `bash` is served by `pi-patty-bg-tasks` (Ctrl+B hint appears / `run_in_background` parameter accepted) while `pi-tool-display` still renders `read`, `grep`, `find`, `ls`, `edit`, `write`.
- **AC7** With `pi-tool-display`, `pi-patty-bg-tasks`, and `pi-markdown-preview` all installed, pi starts with zero extension issues and registers `/preview`, `/preview-browser`, `/preview-pdf`, `/preview-clear-cache` alongside `/bg*` and `/tool-display`.
- **AC5** Secret scan of the preset tree and history stays clean.
- **AC6** After sync, starting pi produces no `Extension shortcut conflict: 'ctrl+b'` warning, Ctrl+B backgrounds the running command, and a pre-existing `keybindings.json` with unrelated custom actions keeps them intact.

## Out of Scope

- Reordering existing `packages[]` entries as an alternative conflict fix.
- Managing config for any other extension.
- Any change to `pi-tool-display` or `pi-patty-bg-tasks` upstream.
- Any change to pi itself (e.g. reclassifying `tui.editor.cursorLeft` or downgrading the diagnostic in `pi-mono`).
- Managing any other `keybindings.json` action.
