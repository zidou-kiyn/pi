# pi-preset module boundary, failure isolation, and documentation requirements

## Existing package boundary

`/home/heixiaohu/pi-preset/package.json:1-15` declares `./extensions` as the Pi extension directory and has no runtime dependency on Matt's repository. A new command placed in `extensions/` will be discovered as part of the existing package; the upstream skill files should remain outside the package and be installed by the official CLI.

The current `/preset-sync` is intentionally local and diff-first (`extensions/preset-sync.ts:1-11`). It computes a read-only plan, stops without UI consent (`extensions/preset-sync.ts:45-72`), and applies local package/config/font changes only after confirmation (`extensions/preset-sync.ts:74-101`). Its apply module explicitly documents sequential, non-rollback behavior for independent local steps (`src/apply.ts:1-7`).

The dedicated `/preset-skills-sync` boundary should therefore remain separate from:

- `extensions/preset-sync.ts` and its command handler;
- `src/plan.ts`/`src/apply.ts`, whose write contract is for the existing local preset;
- `src/manifest.ts`, which is the public, credential-free desired-state list for packages/config (`src/manifest.ts:1-15`, `38-49`); and
- the package's `settings.json` reconciliation path.

A useful separation is:

1. a read-only skill-sync preflight module: detect runner, inspect target files and lock, resolve first-install/update state, and render a redacted metadata plan;
2. an external CLI runner module: receive an argv array, use `pi.exec` or an equivalent no-shell process call, capture exit code, and sanitize output; and
3. a dedicated extension command: gate on `ctx.hasUI`, ask confirmation, call the runner only after approval, report failure/recovery, and offer `/reload` after success.

The exact file names are implementation detail. The invariant is that no network/npm call is reachable from `/preset-sync`, and no external skill write occurs during pure planning or a declined confirmation.

## Network and package behavior

The current README says `/preset-sync` does not automatically run `pi install` and that package-list changes require a restart (`/home/heixiaohu/pi-preset/README.md:100-106`). Keep that behavior unchanged. The new command may invoke the external `skills` CLI only from `/preset-skills-sync`; it should not add `mattpocock/skills` to Pi's `settings.json packages[]`, because loading the whole git package would discover unrelated resources and defeat the explicit two-skill boundary.

Do not make the normal `pi update --extensions` path refresh Matt's skills. The delegated PRD fixes an explicit user-triggered refresh backed by the official CLI (`.trellis/tasks/08-01-integrate-grill-me/prd.md:23-33`, `51-54`).

## Transaction and reporting boundary

The official `skills` CLI can leave a partially updated set after a per-skill/per-agent failure; see `skills-cli-contract.md` and `/tmp/vercel-skills-research/src/add.ts:1740-1777`, `1844-1887`, `2026-2054`. The extension must not claim atomicity unless it adds staging or snapshot/restore around the external process.

For any chosen rollback design, test all of these as one unit:

- `~/.pi/agent/skills/grill-me/`;
- `~/.pi/agent/skills/grilling/`;
- any canonical `~/.agents/skills/` path created by the official CLI; and
- the global `.skill-lock.json` path, including `$XDG_STATE_HOME`.

Symlinks, file modes, timestamps, and lock contents must be restored consistently. A plain “exit code non-zero” check is insufficient because the official installer can have already removed and recreated one target.

All child-process arguments must be passed as an argv array, never interpolated into `sh -c`. The official update implementation uses `shell: false` specifically to prevent a lock-controlled ref from becoming shell syntax (`/tmp/vercel-skills-research/src/update.ts:678-690`; regression test `/tmp/vercel-skills-research/tests/update.test.ts:537-585`).

Reports should include only safe metadata: operation type, the two fixed skill names, source repository, target roots, hash state, exit status, and a copy-pasteable recovery command. Do not echo environment variables, shell-expanded command strings containing arbitrary lock data, or full third-party output without bounds.

## No-duplicate invariant

Pi can discover both global roots, canonicalize symlinks, silently deduplicate one real file, and warn on separate same-name files (`packages/coding-agent/src/core/skills.ts:394-426`). Tests should verify the resolved paths rather than only directory names:

```bash
find "$HOME/.pi/agent/skills" "$HOME/.agents/skills" \
  -type f -path '*/SKILL.md' -print0 |
  xargs -0 -n1 realpath | sort
```

The clean successful state must have exactly one resolved file for each of `grill-me` and `grilling`. The RPC command response must also contain exactly one command for each name, with `source: "skill"` and a path under an intended global root. A project-local or unrelated Matt copy must fail the sandbox test even if Pi happens to select one copy as the winner.

## README content required by the upstream contract

The public README should add a dedicated section that contains all of the following:

1. **Attribution:** link `https://github.com/mattpocock/skills`, identify Matt Pocock, and state MIT licensing. Explain that files are obtained from upstream at install/update time and are not behaviorally rewritten or vendored.
2. **Why two skills:** `grill-me` is an explicitly invoked wrapper and `grilling` is the separate behavior primitive; both are required.
3. **Exact install path:** document the official command with two `--skill` filters, `--agent pi`, `--global`, and `--yes`; explicitly warn against `--all`, the Claude plugin, and whole-repository package installation.
4. **Dedicated command:** document `/preset-skills-sync` as the only preset entry point for first install and refresh, and state that `/preset-sync` does not perform network/npm work.
5. **Direct use:** document `/skill:grill-me` and `/skill:grilling`.
6. **Update and recovery:** document the official update command and the explicit repair/add command, including the Node/npm prerequisite and what happens when `npx` or network access is unavailable.
7. **State locations:** identify `~/.pi/agent/skills/` and the default/`XDG_STATE_HOME` lock location `~/.agents/.skill-lock.json` or `$XDG_STATE_HOME/skills/.skill-lock.json`.
8. **Reload:** state that successful skill-only changes require `/reload` in an existing session; a restart is a fallback. Keep the separate restart warning for changes to `settings.json packages[]`.
9. **Security:** quote or summarize Pi's warning that skills run as model instructions with agent permissions, and tell users to review upstream content.
10. **No secrets:** confirm that the preset repository contains no credentials or private endpoints; the skill integration adds no provider configuration.

The current README already establishes public MIT licensing (`README.md:122-124`), no credentials/secret scanning (`README.md:100-104`, `112-114`), and package bootstrap (`README.md:7-19`). Extend those sections rather than creating contradictory restart or ownership language.

Because the preset itself is MIT (`package.json:6`), retain that license. The Matt repository's MIT notice is at `/tmp/mattpocock-skills-research/LICENSE:1-20`. The Vercel `skills` CLI package is also MIT (`/tmp/vercel-skills-research/package.json:116-145`); identify it as the runtime installer, not as content owned by the preset. If upstream files are ever redistributed in the repository, include their full copyright and permission notice.

## Verification commands for the README examples

The README's copy-pasteable commands should be validated in a clean temporary home, not against the developer's real global skill directory:

```bash
SANDBOX="$(mktemp -d)"
mkdir -p "$SANDBOX/home" "$SANDBOX/state" "$SANDBOX/work"
(
  export HOME="$SANDBOX/home"
  export XDG_STATE_HOME="$SANDBOX/state"
  export PI_CODING_AGENT_DIR="$HOME/.pi/agent"
  cd "$SANDBOX/work"
  npx --yes skills@latest add mattpocock/skills \
    --skill grill-me --skill grilling \
    --agent pi --global --copy --yes
  test -f "$HOME/.pi/agent/skills/grill-me/SKILL.md"
  test -f "$HOME/.pi/agent/skills/grilling/SKILL.md"
  npx --yes skills@latest update grill-me grilling --global --yes
)
```

After the command exits, run the exact file/lock/RPC checks in `pi-sandbox-and-tui.md`. For TUI behavior, use the tmux smoke test there with read-only tools and a configured provider; do not put credentials in the README command, test fixture, captured output, or commit history.

## Extension process execution API

Pi exposes `pi.exec(command, args, options)` with captured stdout/stderr/exit code (`packages/coding-agent/docs/extensions.md:1636-1643`). The implementation delegates to `execCommand` and uses `spawn(command, args, { shell: false, ... })` (`packages/coding-agent/src/core/extensions/loader.ts:343-346`, `packages/coding-agent/src/core/exec.ts:34-45`). This is the appropriate process boundary for the dedicated command: pass `npx`/`npm` and every flag as separate arguments, set a timeout/abort signal, and never interpolate the source or lock ref into a shell command.
