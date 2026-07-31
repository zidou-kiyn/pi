# Implement: personal-pi-preset

Workspace: `~/pi-preset/` (new, outside the pi monorepo). No product code changes inside the pi monorepo.

## Checklist

### Phase 1 — Repository skeleton
- [x] 1.1 `mkdir ~/pi-preset && git init`; write `.gitignore` (`node_modules/`, `*.tmp`, `*.preset-bak`)
- [x] 1.2 Write `package.json`: `name: pi-preset`, `keywords: ["pi-package"]`, `type: module`, `pi.extensions: ["./extensions"]`, three core packages as `peerDependencies: "*"`
- [x] 1.3 Write `LICENSE` (MIT)
- [x] 1.4 Write `scripts/scan-secrets.sh` (patterns per design.md "Secret protection"); `chmod +x`

### Phase 2 — Declaration source and pure logic
- [x] 2.1 `src/manifest.ts`: the 13 packages, `WEB_SEARCH_PATCH`, `FONT` constants (**no pi-startup-redraw-fix, no preference keys, no hardcoded font version**)
- [x] 2.2 `src/paths.ts`: **two resolvers, not one** — the step as originally written was wrong. `pi-web-access/utils.ts:5-13` (`PI_CODING_AGENT_DIR` → `XDG_CONFIG_HOME/pi` → `~/.pi`) governs *only* `web-search.json`. pi's own `getAgentDir()` (`packages/coding-agent/src/config.ts:515-521`) is `PI_CODING_AGENT_DIR` → `~/.pi/agent` with **no XDG branch**; applying the web-search precedence to `settings.json` would write it where pi never reads it. Both are implemented separately. Plus platform font directories
- [x] 2.3 `src/json-merge.ts`: deep merge, `.preset-bak` backup, tmp + rename atomic write, abort on parse failure
- [x] 2.4 `src/plan.ts`: pure read plus pure computation producing a `SyncPlan`; deduplicate `packages[]` by npm package name / git URL without ref
- [x] 2.5 `src/font.ts`: `detect()` / `install()`, latest-release resolution, assetPattern matching, platform branches
- [x] 2.6 `src/apply.ts`: run steps in order, stop on failure, report what already completed

### Phase 3 — Extensions
- [x] 3.1 Move the footer: `~/.pi/agent/extensions/vibrant-footer/index.ts` → `extensions/vibrant-footer.ts`; update the header comment path; read through to confirm no `heixiaohu` / `sub2api` / private IP / absolute path
- [x] 3.2 `extensions/preset-sync.ts`: `pi.registerCommand("preset-sync", ...)` — plan, notify and return when empty, otherwise render the diff, `ctx.ui.confirm`, apply, then summarize and print the pi-image-gen credential reminder
- [x] 3.3 Write `README.md`: the two bootstrap commands, the 13-extension table, git ref semantics (omitting `@ref` is what follows the default branch), the note that the terminal must be set to `Maple Mono NF CN` manually, and Windows manual font steps

### Phase 4 — Sandbox verification
- [x] 4.1 `pi install /home/heixiaohu/pi-preset` into `PI_CODING_AGENT_DIR=$(mktemp -d)`, then plan+apply → `packages[]` = 14 (preset + 13), `pi list` shows all → **AC1**
- [x] 4.2 `pi update --extensions` in the same sandbox: exit 0, and all 13 sources resolved on disk (12 under `npm/node_modules`, `pi-apply-patch` cloned under `git/`) — not just the preset entry → **AC2**
- [x] 4.3 Seeded `openaiApiKey`/`braveApiKey`/`webSearch.enabled=true`/`webSearch.provider=brave`: both keys and the sibling `provider` survived, `enabled`→false, `ssrf.trustEnvProxy`→true, final field set exactly `{openaiApiKey, braveApiKey, webSearch, ssrf}` → **AC3**
- [x] 4.4 Second run: `steps: (none)`, md5 **and** mtime identical → **AC4**
- [x] 4.5 Plan-only run (the decline path, since `apply()` is the only writer): md5 unchanged and no `.preset-bak` created → **AC5**
- [x] 4.6 Isolated `XDG_DATA_HOME` + `fc-list` off `PATH` → `detect()` false; full install pulled `MapleMono-NF-CN-unhinted.zip` (v7.9, 159 MB), wrote 16 TTFs, `fc-scan` reports family `Maple Mono NF CN`, `detect()` now true, temp dir cleaned → **AC7**
- [x] 4.7 `platform=win32`: **zero** `font.install` steps planned (a no-write step would make the plan permanently non-empty there and break AC4), instructions with the release URL emitted as a note → **AC8**
- [x] 4.8 Local `extensions/vibrant-footer/` alongside the package: RPC `get_commands` showed `vibrant-footer:1` + `vibrant-footer:2`, after `footer.demote` a single `vibrant-footer`, local copy intact (655 lines) in `extensions-disabled/` → **AC10**

### Phase 5 — Publish and local cleanup
- [x] 5.1 `./scripts/scan-secrets.sh` clean on both working tree and history, plus an independent sweep of **every blob in `git rev-list --all --objects`**: zero hits → **AC6**
- [x] 5.2 `gh repo create pi-preset --public --push` → https://github.com/zidou-kiyn/pi-preset
- [x] 5.3 `pi install git:github.com/zidou-kiyn/pi-preset` on the real machine
- [x] 5.4 Ran the plan against the real environment and applied `footer.demote` → local copy now at `~/.pi/agent/extensions-disabled/vibrant-footer.local.bak` → **AC10**
- [x] 5.5 `pi remove npm:pi-startup-redraw-fix`; `pi list` no longer shows it; `packages[]` is exactly 14 → **AC9**
- [x] 5.6 `rm ~/.pi/agent/AGENTS.md` (content was the single line `Always respond in 简体中文 (Simplified Chinese).`, recorded here for restoration) → **AC11**
- [x] 5.7 Real environment via RPC `get_commands`: 20 extension commands from 10 packages, `vibrant-footer` and `preset-sync` each registered exactly once, zero duplicate-suffixed commands

## Validation commands

```bash
# Sandbox
export PI_CODING_AGENT_DIR=$(mktemp -d)
pi install ~/pi-preset
pi list

# Secret scan
cd ~/pi-preset && ./scripts/scan-secrets.sh
git log -p | grep -nE 'sk-[A-Za-z0-9]{20,}|heixiaohu|anyrouter|sub2api|127\.0\.0\.1:8317'

# Font
fc-list : family | grep "Maple Mono NF CN"

# Idempotence
md5sum ~/.pi/web-search.json ~/.pi/agent/settings.json   # once before sync, once after
```

## High-risk files and rollback points

| File | Risk | Rollback |
|---|---|---|
| `~/.pi/web-search.json` | Holds every provider API key; a whole-file overwrite loses all of them | `.preset-bak` before writing; abort on parse failure |
| `~/.pi/agent/settings.json` | Holds the `pi-image-gen` credential and provider defaults | Same as above; `packages[]` is append-only |
| `~/.pi/agent/extensions/vibrant-footer/` | The only copy of the footer source | Only `rename` into `extensions-disabled/`, never delete; and it is packaged first |
| `~/.pi/agent/AGENTS.md` | Deletion is irreversible | Content is a single line, already recorded in prd.md |
| Public GitHub repository | Once pushed, a leaked secret is public | 5.1 runs before 5.2; if leaked, delete and recreate the repository and rotate the key |

## Pre-start checks

- [x] `~/pi-preset` does not exist or is empty
- [x] GitHub target is `github.com/zidou-kiyn/pi-preset` (confirmed via `gh api user`); the repository does not exist yet and is created at 5.2
- [x] `pi --version` is 0.83.0 and `node -v` is v24.16.0
- [x] `unzip` and `bsdtar` are both available (GNU `tar` is **not** a valid zip fallback — verified, it rejects the archive)
