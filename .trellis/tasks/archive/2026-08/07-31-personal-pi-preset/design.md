# Design: personal-pi-preset

## Deliverable boundary

The artifact is a **standalone public GitHub repository**, not part of the pi monorepo. Local development path: `~/pi-preset/` — outside the pi checkout so the monorepo stays clean, and free of non-ASCII path segments so the shell scripts in this package stay portable.

**No product code changes land in the pi monorepo** — only the planning artifacts under `.trellis/tasks/07-31-personal-pi-preset/`.

Hosting the package inside this fork was evaluated and rejected on evidence:
- `parseSource` / `parseGitUrl` (`package-manager.ts:1419`) carry only host / path / ref — **git sources have no subdirectory support**, so a `preset/` subdirectory of the monorepo can never be a `git:` install target. The clone root is the package root.
- That leaves an orphan branch (`git:github.com/zidou-kiyn/pi@preset`), which works — `:1845` reconciles a branch ref via `fetch origin <ref>` + reset to `FETCH_HEAD`, so it does advance — but `:1822` clones with **no `--depth`**, so every machine pulls the whole monorepo (62 MB `.git`) for a ~50 KB package. The fork's root `package.json` is `pi-monorepo` with `workspaces` and no `pi` key, so `main` itself can never be the package root without a change that conflicts with every upstream merge.
- A local-path install keeps the files next to the planning artifacts but drops out of `pi update` entirely (local sources are never iterated) and hardcodes a non-ASCII absolute path into `settings.json`.

## Repository layout

```
pi-preset/
├── package.json              # pi manifest + peerDependencies
├── README.md                 # bootstrap, git ref semantics, terminal font note
├── LICENSE
├── .gitignore
├── extensions/
│   ├── vibrant-footer.ts     # moved from ~/.pi/agent/extensions/vibrant-footer/index.ts
│   └── preset-sync.ts        # the /preset-sync command
└── src/
    ├── manifest.ts           # single source of truth (13 packages + 2 config keys)
    ├── plan.ts               # computes the diff plan (pure, unit-testable)
    ├── apply.ts              # executes the plan (the only writer)
    ├── json-merge.ts         # deep merge + atomic write
    ├── font.ts               # font detect / download / install
    └── paths.ts              # resolves the pi agent dir and platform font dirs
```

`package.json`:

```json
{
  "name": "pi-preset",
  "keywords": ["pi-package"],
  "type": "module",
  "pi": { "extensions": ["./extensions"] },
  "peerDependencies": {
    "@earendil-works/pi-coding-agent": "*",
    "@earendil-works/pi-tui": "*",
    "@earendil-works/pi-ai": "*"
  }
}
```

The convention directory `extensions/` is auto-discovered; the `pi` key is declared explicitly so that adding `src/` later cannot be mis-scanned. Core packages use peer + `"*"` per `docs/packages.md` "Dependencies". There are no third-party runtime dependencies (zip extraction shells out to the system `unzip`/`tar`, see below).

## Desired state declaration (`src/manifest.ts`)

Single source of truth; both `plan.ts` and the README derive from it:

```ts
export const REQUIRED_PACKAGES = [
  "npm:pi-wtf", "npm:pi-workspace-history", "npm:@ff-labs/pi-fff",
  "npm:pi-tool-display", "npm:@narumitw/pi-chrome-devtools",
  "npm:pi-playwright", "npm:pi-web-access",
  "npm:@lll9p/pi-better-compaction", "npm:pi-web-search",
  "git:github.com/code-yeongyu/pi-apply-patch",
  "npm:@juicesharp/rpiv-todo", "npm:@juicesharp/rpiv-ask-user-question",
  "npm:@amaster.ai/pi-image-gen",
] as const;

export const WEB_SEARCH_PATCH = {
  webSearch: { enabled: false },
  ssrf: { trustEnvProxy: true },
} as const;

export const FONT = {
  family: "Maple Mono NF CN",
  repo: "subframe7536/maple-font",
  assetPattern: /^MapleMono-NF-CN-unhinted\.zip$/,
} as const;
```

No `pi-startup-redraw-fix` (R6). No preference keys (D4).

## /preset-sync data flow

```
plan()  ── read-only ──▶  SyncPlan { steps: Step[] }
   │
   ├─ readSettings()      ~/.pi/agent/settings.json
   ├─ readWebSearch()     ~/.pi/web-search.json
   ├─ detectFont()        fc-list / directory scan
   └─ detectLocalFooter() ~/.pi/agent/extensions/vibrant-footer/
   │
   ▼
render(plan) ──▶ ctx.ui.confirm("Apply preset?", body)
   │                              │
   │                     declined ─┴─▶ return with zero writes (AC5)
   ▼
apply(plan) ── the only writer ──▶ run steps in order; stop on failure and report what completed
```

`plan()` is pure read plus pure computation, so it is unit-testable outside pi. `apply()` is the single side-effecting point. When the plan is empty it calls `ctx.ui.notify("preset: already in sync")` and skips the confirmation entirely (AC4).

### Step kinds

| Step | Target | Idempotence check |
|---|---|---|
| `settings.packages.add` | `settings.json` | Skip when the source is already in `packages[]`, using the identity rule from `docs/packages.md`: npm compares package name, git compares repository URL without ref |
| `webSearch.patch` | `~/.pi/web-search.json` | Skip when the target key already holds the desired value |
| `footer.demote` | local `vibrant-footer/` → `extensions-disabled/` | Skip when the local directory is absent |
| `font.install` | platform font directory | Skip when the font family is already present. **Never planned on win32 or an unknown platform**: a step that cannot write would keep the plan permanently non-empty there, so no run could ever report "already in sync" (AC4) and every run would prompt to "apply" nothing. The manual instructions ride along as a note instead, and the empty-plan path prints notes for exactly that reason |

## Write strategy

**Deep merge plus atomic write** (`src/json-merge.ts`):
- Read the original file → `JSON.parse` → recursively merge the patch (only leaf keys explicitly present in the patch are overwritten) → `JSON.stringify(_, null, 2)` → write `<file>.tmp` → `renameSync`.
- Missing file → start from `{}`.
- `JSON.parse` failure → **abort that step with an error**, never overwrite (this is what prevents swallowing a `web-search.json` full of API keys).
- Before writing, `copyFileSync` the target to `<file>.preset-bak` so a rollback is trivial.

`packages[]` is an array: deduplicate by identity, then **append**. Never reorder, never remove packages the user added themselves (R3). `apply()` re-reads the file and **re-derives identities** immediately before writing rather than trusting the plan's string list — pi's own package manager writes this same file, and an entry it added in the interim may carry a version suffix (`npm:pi-tool-display@1.2.3`) that a plain string comparison would miss, producing a duplicate.

Path resolution is **two resolvers, not one**. `settings.json` follows pi's `getAgentDir()` (`config.ts:515-521`): `PI_CODING_AGENT_DIR` → `~/.pi/agent`, with no XDG branch. `web-search.json` follows pi-web-access's `getWebSearchConfigDir()` (`utils.ts:5-13`): `PI_CODING_AGENT_DIR` → `$XDG_CONFIG_HOME/pi` → `~/.pi`. Using the latter for `settings.json` would write it where pi never reads it. Both collapse onto `PI_CODING_AGENT_DIR`, which is what makes a single-directory sandbox work.

## Font installation (`src/font.ts`)

```
detect()
  linux : spawn("fc-list", [":", "family"]) → contains "Maple Mono NF CN"?
          fc-list missing → fall back to scanning ~/.local/share/fonts and
          /usr/share/fonts for *Maple*NF*CN* filenames
  darwin: scan ~/Library/Fonts and /Library/Fonts for *Maple*NF*CN*
  win32 : return unsupported immediately

install()  // only runs when detect() is false
  1. GET https://api.github.com/repos/subframe7536/maple-font/releases/latest
     → match assets against assetPattern (no hardcoded version)
  2. Download into os.tmpdir()
  3. Extract: `unzip -o` on linux/darwin, falling back to `tar -xf` (bsdtar reads zip)
  4. Copy *.ttf into the target directory
     linux : ~/.local/share/fonts/maple-nf-cn/   → then `fc-cache -f`
     darwin: ~/Library/Fonts/                    → no refresh needed
  5. Clean up temporary files
  win32: no writes; print the release page link and manual steps
```

Network and extraction both have timeouts. Any failure fails only the `font.install` step; steps that already completed stay applied and are reported truthfully.

## Footer migration (R7)

`resource-loader.ts` performs no name-based dedupe for extensions, so `~/.pi/agent/extensions/*/index.ts` and the packaged `extensions/vibrant-footer.ts` would both load and register the footer twice.

The `footer.demote` step runs `renameSync(~/.pi/agent/extensions/vibrant-footer, ~/.pi/agent/extensions-disabled/vibrant-footer.local.bak)`. If the destination exists, append a timestamp suffix. It only moves, never deletes.

## Footer code migration

`vibrant-footer/index.ts` moves into `extensions/vibrant-footer.ts` verbatim, with three adjustments only:
1. The `Runtime: ~/.pi/agent/extensions/vibrant-footer/index.ts` line in the header comment becomes the in-package path.
2. Keep the `/vibrant-footer` toggle command.
3. Read it through once to confirm there is no machine-local absolute path and no personal identifier (`heixiaohu`, `sub2api`, private IPs).

No rewrite, no refactor — it already works and already satisfies R1.

## Secret protection (R4 / AC6)

- `scripts/scan-secrets.sh` at the repository root greps every tracked file for `sk-[A-Za-z0-9]{20,}`, `heixiaohu`, `anyrouter`, `sub2api`, `127\.0\.0\.1:8317`, and `"apiKey"`.
- It scans the working tree and `git log -p --all` in one run, and **excludes itself from both** — the script necessarily contains every pattern it greps for, so without the history-side pathspec exclusion its own diff is a guaranteed false positive.
- Run it once after `git init` and before the first push (AC6).
- If a secret was ever committed, recreate the repository rather than rebasing — public history cannot be trusted.

## Local cleanup (R6 / R9)

Not part of the preset repository; a one-time local operation:
1. `pi remove npm:pi-startup-redraw-fix`
2. `rm ~/.pi/agent/AGENTS.md`
3. `/preset-sync` triggers `footer.demote`

## Post-implementation review

An adversarial review after Phase 4/5 found six defects, all fixed in `5d9f3b7`:

| Finding | Failing input | Fix |
|---|---|---|
| Atomic write dropped the file mode | `chmod 600 web-search.json` → `0644` after sync, because the tmp file was created under the umask | `statSync` the target, `chmodSync` the tmp file before `renameSync` |
| Symlinked config replaced, not written through | `web-search.json -> dotfiles/web-search.json` became a regular file; the dotfiles copy silently stopped tracking | `lstatSync`/`realpathSync` resolve the target first; a dangling link still counts as absent |
| Font step could never converge | `fc-list` exits 0 without reporting the family while the TTFs are on disk (fontconfig < 2.13.94 ignores `XDG_DATA_HOME`; or `fc-cache` failed) → plan never empty, 159 MB re-downloaded every run | A clean `fc-list` without a match is inconclusive, not proof of absence: fall through to the disk scan |
| Settings written even with nothing to append | pi installed the packages between plan and apply → rewrote a credential-bearing file, bumped mtime, clobbered a good `.preset-bak`, reported "added 0" | Early return before the write |
| Bare package name misidentified | `packages: ["pi-wtf"]` satisfied `npm:pi-wtf`, but pi's `isLocalPath` (`utils/paths.ts:41-55`) treats an unprefixed string as a **local path**, so pi resolved a nonexistent directory | Identity mirrors `isLocalPath`; only `npm:`/`git:`/`github:`/`http(s):`/`ssh:` are non-local |
| Git identity spelling drift | `git:GitHub.com/user/repo/` appended a duplicate pi would dedupe at load | Lowercase the host, strip a trailing slash |

Accepted and documented rather than fixed:

- **`packages[]` writes are invisible to the running session.** `persistScopedSettings` (`settings-manager.ts:578-606`) re-merges only *modified* fields, and for an array field takes the whole-value branch — so `/config` or `pi install` in the same session persists the startup-era array over the new entries. Extensions get no access to `SettingsManager` (absent from `ExtensionAPI` and `ExtensionCommandContext`), so this cannot be fixed from an extension; `/preset-sync` and the README now state the restart requirement explicitly. Codified in `.trellis/spec/pi-coding-agent/core/session-and-config.md`.
- **The asset *filename* is still a literal** (`MapleMono-NF-CN-unhinted.zip`). R5 forbids a hardcoded version, not a hardcoded name. If upstream renames the asset, `resolveLatestAsset` throws every run with the release URL — fail-safe, and a rename is a one-line manifest change.
- **`.preset-bak` files hold credentials** and are never cleaned up. They inherit the source file's mode via `copyFileSync`, so a `0600` original yields a `0600` backup.
- **`git log -p` shows no diff for merge commits** and never sees unreachable objects. The 5.1 sweep over every blob in `git rev-list --all --objects` covers the second gap for reachable history; the repository has no merges.

## Trade-offs recorded

- **No `pi-image-gen` config skeleton**: any `apiKey` placeholder invites committing a real key into a public repository. Instead `/preset-sync` prints one line at the end saying pi-image-gen needs its own provider credentials.
- **No `adm-zip`/`yauzl` dependency**: `unzip`/`bsdtar` are available by default on the supported platforms, and this removes a supply-chain layer. The fallback order is `unzip` → `bsdtar` → `tar`, not "tar reads zip": **GNU tar cannot read a zip archive** (verified — it reports the file does not look like a tar archive). Plain `tar` is last only because on macOS it *is* bsdtar.
- **No automatic `pi install`**: `/preset-sync` only writes `packages[]` and lets pi install on its next start, avoiding concurrent writes into the same directory as pi's own package manager.
- **The preset's own `packages[]` entry omits `@ref`**: git sources are only reconciled to the configured ref, so following the default branch is what makes "effective on push" true. The README states this explicitly.
