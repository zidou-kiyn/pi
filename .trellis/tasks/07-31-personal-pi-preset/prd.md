# PRD: personal pi-agent preset package (personal-pi-preset)

## Goal

Package zidou-kiyn's pi environment as a public GitHub repository shaped as a pi package, so a new machine can reproduce the footer status bar, the extension set, the non-default extension config, and the Nerd Font with two commands — without leaking any credential, and without losing each extension's native `pi update --extensions` capability.

## Background

### Current local state

| Item | Location | Disposition |
|---|---|---|
| Footer extension | `~/.pi/agent/extensions/vibrant-footer/index.ts` (655 lines) | **Ship**; demote the local copy |
| Old footer backup | `~/.pi/agent/extensions-disabled/pi-native-footer.bak/` (226 lines) | Do not ship |
| Extension list | `~/.pi/agent/settings.json` → `packages[]` (14 entries) | **Ship** (drop startup-redraw-fix → 13) |
| pi-tool-display config | `~/.pi/agent/extensions/pi-tool-display/config.json` | **Do not ship**: byte-equivalent to upstream defaults |
| web-search config | `~/.pi/web-search.json` | **Ship 2 keys**, merge-write only |
| AGENTS.md | `~/.pi/agent/AGENTS.md` | **Do not ship**, and delete locally (user instruction) |
| officecli skill | `~/.pi/agent/skills/officecli/` (417 lines) | **Do not ship** (user instruction) |
| Font | `~/.local/share/fonts/m/MapleMono_NF_CN_*.ttf` | **Install on demand**, never bundled |
| Provider config | `models.json` / `anyrouter.json` / `settings.json` `defaultProvider` etc. | **Do not ship** (all carry credentials or point at a private gateway) |

### Final extension list (13)

```
npm:pi-wtf                              npm:pi-web-access
npm:pi-workspace-history                npm:@lll9p/pi-better-compaction
npm:@ff-labs/pi-fff                     npm:pi-web-search
npm:pi-tool-display                     git:github.com/code-yeongyu/pi-apply-patch
npm:@narumitw/pi-chrome-devtools        npm:@juicesharp/rpiv-todo
npm:pi-playwright                       npm:@juicesharp/rpiv-ask-user-question
npm:@amaster.ai/pi-image-gen
```

### pi mechanics (all verified in code)

**Package model** (`packages/coding-agent/docs/packages.md`)
- A pi package only recognizes four resource kinds: `extensions` / `skills` / `prompts` / `themes`. `settings.json`, per-extension config files, and fonts are outside that model, so extension code must write them itself.
- Core packages must be declared as `peerDependencies: "*"`; `@earendil-works/pi-{ai,agent-core,coding-agent,tui}` must never be bundled.
- A git source is cloned to `~/.pi/agent/git/<host>/<path>`, and `npm install` runs automatically when `package.json` exists.

**Update semantics** (`packages/coding-agent/src/core/package-manager.ts`)
- `update()` (`:1032`) only iterates sources explicitly listed in `settings.json` `packages[]`. → A `bundledDependencies` single-package design would freeze the inner extensions permanently, so it is rejected.
- Only unpinned npm sources participate in updates (`updateConfiguredSources`, `:1064`); `updateNpmBatch` (`:1139`) installs them as `@latest`.
- Git sources are only reconciled to the configured ref; they never advance to a newer tag on their own.
- Installs always use `--legacy-peer-deps` / `--omit=peer` (`:1758-1778`), so **peerDependency ranges are never enforced**. → The user's concern about running `pi update --extensions` before `pi update` (or the reverse) does not hold at install time: the two commands share no state, and version drift between machines cannot cause an install failure. The only residual risk is pi core API drift breaking an older extension at runtime, which is independent of ordering.

**Both config consumers fall back per key, so a partial config is valid and cannot be frozen by a stale snapshot**
- `normalizeToolDisplayConfig` (`pi-tool-display/src/config-store.ts:203-241`) falls back per field to `DEFAULT_TOOL_DISPLAY_CONFIG` (`src/types.ts:68-95`). A field-by-field comparison of the local file yields `DIFF vs default: {}` — the file is just a full snapshot written by the config modal, not a customization.
- pi-web-access uses optional reads: `config.webSearch?.enabled === false` (`index.ts:226,797,1482,2046`) and `ssrf.trustEnvProxy` (`extract.ts:107,257,702`). Unset means default (search tools registered / env proxy untrusted).
- `web-search.json` (path resolution at `pi-web-access/utils.ts:11-13`) is also where every provider API key lives, so **whole-file overwrite is forbidden**.

**Extensions have no name-based dedupe**
- `resource-loader.ts` emits collision diagnostics only for prompts (`:969`) and themes (`:995`); extensions are collected by path and appended directly.
- `~/.pi/agent/extensions/*/index.ts` is a convention auto-discovery path (`docs/extensions.md:117-118`). → A local `vibrant-footer/` and the packaged footer would both load, so the local copy must be moved away.

**pi-startup-redraw-fix is now a no-op**
- It rewrites `\x1b[3J\x1b[2J\x1b[H` into `\x1b[H\x1b[2J\x1b[3J` (`src/terminal-clear-patch.ts:3-4`).
- pi 0.83.0 emits `\x1b[2J\x1b[H\x1b[3J` at `packages/tui/src/TuiMainScreen.ts:181` (introduced by commit `c13ffe18`, alternate-screen renderer), which does not match its `BROKEN_FULL_CLEAR_SEQUENCE`.
- → The patch never fires; safe to remove.

**Font facts**
- The installed font is **Maple Mono NF CN v7.0 unhinted**: `fc-query` reports `fontversion 458752` (= 7.0) and family `Maple Mono NF CN`; the TTF table set is `[GSUB, OS/2, cmap, gasp, glyf, head, hhea, hmtx, loca, maxp, meta, name, post]` with no `fpgm`/`prep`/`cvt ` → unhinted.
- 16 TTFs total **319 MB**; the official zip is roughly 150 MB. → It fits neither an npm tarball nor a git repository, so it must be fetched from a GitHub Release on demand.
- Upstream latest is v7.9; release assets are named like `MapleMono-NF-CN-unhinted.zip`. **Track latest, do not pin v7.0** (user decision). The family name `Maple Mono NF CN` is stable across versions, so detection keys on the family rather than filenames (the local v7.0 files use underscores, `MapleMono_NF_CN_*.ttf`, which newer releases may not).

## Key Decisions

- **D1 Architecture = bootstrap package + independent sources (hybrid).** The preset ships only the footer extension, the `/preset-sync` logic, and font installation; the 13 extensions stay as independent entries in `packages[]` and keep native update behavior. Cost: extension versions may drift between machines (proven harmless at install time).
- **D2 Distribution = public GitHub repository, `pi install git:github.com/zidou-kiyn/pi-preset`.** No prerequisites, no publish workflow, effective on push. Cost: the repository is publicly visible.
- **D3 Config application = explicit `/preset-sync` command that renders a diff and then asks `ctx.ui.confirm` before writing**, idempotent. Cost: a new machine must trigger it once by hand.
- **D4 No personal preferences shipped**: no `theme`, no `defaultThinkingLevel`, no `AGENTS.md`.
- **D5 Font platform scope = automatic on Linux and macOS, manual instructions on Windows**; the version tracks the **latest release** unhinted NF-CN asset with no hardcoded version number.
- **D6 Excluded from the package**: officecli skill, pi-tool-display config, pi-startup-redraw-fix.

## Requirements

- **R1 Single-install reproducibility**: after `pi install git:github.com/zidou-kiyn/pi-preset` plus `/preset-sync`, a new machine has the footer, the 13 extensions, the 2 web-search keys, and the font in place.
- **R2 Extensions stay updatable**: the 13 extensions exist as independent `packages[]` entries, and `pi update --extensions` applies to every one of them.
- **R3 Minimal config surface with non-destructive writes**: write only keys that genuinely deviate from upstream defaults (currently just `webSearch.enabled=false` and `ssrf.trustEnvProxy=true`); writes to `web-search.json` and `settings.json` are deep merges that must never delete or overwrite existing fields, above all the provider API keys stored in the same file.
- **R4 Zero sensitive data in the artifact**: no API keys, no private gateway host or port, no sessions, logs, credentials, or personal prompts.
- **R5 Idempotent font install**: detect by font family `Maple Mono NF CN` and skip when present (no version comparison, no network request); when missing, resolve and download the unhinted NF-CN asset from the GitHub **latest release** and install it into the platform font directory (Linux additionally runs `fc-cache -f`). The asset name must not hardcode a version. Windows prints the download link and manual steps.
- **R6 Retire pi-startup-redraw-fix**: excluded from the package, and removed from local `packages[]` plus uninstalled.
- **R7 Prevent double-loading the footer**: when `/preset-sync` finds both a local `~/.pi/agent/extensions/vibrant-footer/` and the packaged footer, it moves the local copy into `extensions-disabled/`.
- **R8 Diff preview and idempotence**: `/preset-sync` lists every change (added / modified / already matching / untouched) before writing; declining writes nothing, and repeated runs have no side effects.
- **R9 Local cleanup**: delete `~/.pi/agent/AGENTS.md`.

## Acceptance Criteria

- **AC1** (R1/R2) In a clean `PI_CODING_AGENT_DIR` sandbox, installing the preset (local path before the first push, `git:github.com/zidou-kiyn/pi-preset` after it) then running `/preset-sync` and confirming leaves `settings.json` `packages[]` with exactly 14 entries (preset + 13 extensions), all listed by `pi list`.
- **AC2** (R2) In the same sandbox `pi update --extensions` succeeds and attempts an update for each of the 13 extensions, not only for the preset entry.
- **AC3** (R3) Seed a `web-search.json` containing `openaiApiKey`, `braveApiKey`, and `webSearch.enabled=true`; after `/preset-sync` both keys retain their original values, `webSearch.enabled` becomes `false`, `ssrf.trustEnvProxy` becomes `true`, and no other field is added or removed.
- **AC4** (R8) Running `/preset-sync` twice in a row: the second run reports no changes, and neither file content nor mtime changes.
- **AC5** (R8) After declining the confirmation dialog, `settings.json`, `web-search.json`, and the font directory are byte-identical to their pre-run state.
- **AC6** (R4) A secret scan across the whole repository (`sk-`, `heixiaohu.de`, `anyrouter`, `127.0.0.1:8317`, `apiKey`) returns zero hits, and `git log -p` over full history returns zero hits as well.
- **AC7** (R5) On a machine that already has the font, `/preset-sync` reports `font ok` and issues no network request (local v7.0 is not reinstalled just because upstream moved to v7.9); with the font simulated as missing, it downloads and installs from the latest release, after which `fc-list : family | grep "Maple Mono NF CN"` matches. No hardcoded `v7.` version string appears in the code.
- **AC8** (R5) Under `process.platform === "win32"` no font directory write is attempted, and the output contains the download URL plus manual steps.
- **AC9** (R6) The shipped `packages[]` manifest excludes `pi-startup-redraw-fix`; locally `pi remove npm:pi-startup-redraw-fix` has run and `pi list` no longer shows it.
- **AC10** (R7) With both the preset package and a local `~/.pi/agent/extensions/vibrant-footer/` present, `/preset-sync` moves the local copy into `extensions-disabled/`, and after restarting pi the footer renders exactly once.
- **AC11** (R9) `~/.pi/agent/AGENTS.md` does not exist.

## Out of Scope

- officecli skill, pi-tool-display config.json, pi-native-footer.bak
- Any provider credential, private gateway address, `models.json` / `anyrouter.json` / `auth.json` / `trust.json` / `sessions` / `state` / `statusline-transcripts` / permission-system logs / `REALTIME-SYSTEM-PROMPT.md` / `pi-codex-conversion.json`
- Personal preferences such as `theme` / `defaultProvider` / `defaultModel` / `defaultThinkingLevel`
- Automatic font installation on Windows
- Terminal emulator font selection itself (the user must change it manually)
- pi-image-gen provider credentials (`/preset-sync` only prints a reminder to configure them)

## Risks

- **A public repository exposes personal taste**: accepted (D2). Mitigation: secret scanning before push (AC6).
- **Git source refs do not advance automatically**: the preset entry in `packages[]` omits `@ref` so it follows the default branch; README documents this semantic.
- **Upstream extension API drift**: pi never validates peer ranges, so failures only surface at runtime. Mitigation: keep the footer's `@earendil-works/*` surface minimal and read defensively.
