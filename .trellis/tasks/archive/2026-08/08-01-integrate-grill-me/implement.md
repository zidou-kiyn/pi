# Implement: upstream grill-me integration

Workspace: `/home/heixiaohu/pi-preset` for product code. The first phase intentionally changes only the real local Pi skill directories.

## Phase 1 — Local-first experiment

- [x] 1.1 Record the current real `~/.pi/agent/skills/`, `~/.agents/skills/`, and global skill-lock state without printing unrelated skill contents.
- [x] 1.2 Run the fixed official CLI install for `grill-me` and `grilling` with `--agent pi --global --copy --yes` and telemetry disabled.
- [x] 1.3 Verify both installed `SKILL.md` files, lock v3 source/path/hash fields, and absence of duplicate same-name resources.
- [x] 1.4 Reload/restart Pi and verify `/skill:grill-me` plus `/skill:grilling` register exactly once.
- [x] 1.5 Run the three-turn behavioral TUI smoke test. Stop before preset implementation if delegation or interview behavior fails.

## Phase 2 — Pure sync planning and transaction logic

- [x] 2.1 Add path/lock/duplicate-state types and a read-only plan renderer.
- [x] 2.2 Add portable `npx`/`npm exec` runner discovery and fixed argv builders; never construct a shell command string.
- [x] 2.3 Add snapshot/restore for both global roots and the complete lock file, preserving absence, symlinks, and modes.
- [x] 2.4 Add the CLI runner with timeout, bounded sanitized output, telemetry opt-out, and ignored npm lifecycle scripts.
- [x] 2.5 Add post-install validation for the two target files, frontmatter names, lock source/path/hash, unrelated-skill exclusion, and duplicate real paths.
- [x] 2.6 Compare deterministic directory-content hashes before/after and report installed, updated, or already current.

## Phase 3 — Extension and docs

- [x] 3.1 Add `extensions/preset-skills-sync.ts` with TUI/RPC confirmation, print/JSON no-write behavior, error/recovery reporting, and terminal `ctx.reload()` after changed content.
- [x] 3.2 Add README attribution, security warning, direct invocation, update/repair, path, reload, and troubleshooting sections.
- [x] 3.3 Keep `/preset-sync`, manifest package entries, fonts, and web-search behavior untouched.

## Phase 4 — Tests and sandbox verification

- [x] 4.1 Add Node tests for install/current/repair plans, runner fallback, duplicate detection, argv exactness, and lock parsing.
- [x] 4.2 Add injected-runner tests for decline/no call, successful install, no-change refresh, process failure rollback, validation failure rollback, and unrelated lock-entry preservation.
- [x] 4.3 Run the tests with `node --test` from `/home/heixiaohu/pi-preset`.
- [x] 4.4 Run a clean temporary `HOME`/`XDG_STATE_HOME` real CLI sandbox and assert exactly the two selected Pi skills and lock entries.
- [x] 4.5 Query Pi RPC `get_commands` and assert exactly one command for each intended skill path.
- [x] 4.6 Re-run the TUI behavioral smoke through the preset command and verify reload exposes the same two skills.
- [x] 4.7 Run `scripts/scan-secrets.sh` and the Pi monorepo `npm run check`.

`npm run check` was hydrated and run in full. It reached `tsgo --noEmit` and failed only on 15 pre-existing stale ZAI model references (`glm-4.5-air` and `glm-5.1`) outside this task. The user explicitly approved proceeding with the scoped passing checks on 2026-08-01.

## Validation commands

```bash
cd /home/heixiaohu/pi-preset
node --test test/skills-sync-plan.test.ts test/skills-sync-apply.test.ts
./scripts/scan-secrets.sh

cd /home/heixiaohu/桌面/pi
npm run check
```

The real CLI sandbox and tmux commands are recorded in the sibling research files under `08-01-interactive-models-config/research/`.

## Rollback points

- Phase 1: restore the recorded real local skill/lock snapshot if the direct install fails validation.
- Phase 2/3: revert only the new skills-sync files and README section; existing preset commands are untouched.
- Never remove a separate same-name skill copy automatically. Report the collision and ask the user to resolve ownership.
