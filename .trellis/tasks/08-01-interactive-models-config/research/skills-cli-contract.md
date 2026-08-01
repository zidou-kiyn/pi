# Official skills CLI contract and update risks

## Source and runtime requirements

The Vercel CLI snapshot researched is `1164afa5f0e21ebd01e6fc11249759353f494ad1` (`skills` package `1.5.21`). Its package metadata declares MIT and Node `>=22.20.0` (`/tmp/vercel-skills-research/package.json:116-145`). The local Node version observed during research was `v24.16.0`.

The upstream Matt README documents `npx skills@latest add mattpocock/skills` for Codex and other agents (`/tmp/mattpocock-skills-research/README.md:48-70`). The official CLI's supported flags and examples are in `/tmp/vercel-skills-research/README.md:50-90`.

## First install and repair path

The explicit, non-interactive command is:

```bash
npx --yes skills@latest add mattpocock/skills \
  --skill grill-me --skill grilling \
  --agent pi --global --copy --yes
```

Meaning of each argument:

- `skills@latest`: use the official package rather than a vendored copy;
- `add mattpocock/skills`: fetch the upstream GitHub source;
- two repeated `--skill` flags: select exactly the wrapper and primitive;
- `--agent pi`: do not auto-select any other agent;
- `--global`: use the user-level installation;
- `--copy`: make the initial Pi target explicit and avoid an implicit multi-agent layout; and
- `--yes`: bypass CLI prompts, including the optional `find-skills` follow-up.

The CLI's add option type includes all of these switches (`src/add.ts:540-556`); the parser accepts repeated `--agent` and `--skill` values as separate argv tokens (`src/add.ts:2156-2202`). Use separate tokens, not a shell-built command string.

The same command is the repair path when either skill directory or its lock entry is missing. It is also the only direct path that deterministically reasserts `--agent pi --copy`.

## Subsequent update path

The official global update command is:

```bash
npx --yes skills@latest update grill-me grilling --global --yes
```

The CLI documents positional skill names and `-g/--global`, `-y/--yes` at `/tmp/vercel-skills-research/README.md:152-177`; the parser implements only those update scope flags in `src/update.ts:37-65`. `--agent pi` and `--copy` are not update options.

A unified preset command can distinguish the two states by a read-only preflight:

1. **Install/repair:** either target `SKILL.md` is absent, either lock entry is absent, the lock is older than v3, the source is not `mattpocock/skills`, or a required `skillPath`/folder hash is missing. Dispatch the explicit `add` command above.
2. **Refresh:** both targets and both valid lock entries exist. Dispatch the official `update grill-me grilling --global --yes` command.
3. **No-change:** let `update` perform its hash checks; it prints that global skills are up to date and does not spawn an `add` child when no hash changed (`src/update.ts:628-651`).

The preflight must not treat “files exist” alone as an installed state: files without lock entries cannot be selected by the official update flow (`src/update.ts:465-510`). Conversely, a lock entry without its target file must be repaired with `add`, not sent to `update`.

This is a state-machine description, not a product choice. The important compatibility constraint is that the official CLI has different semantics for first install and update.

## Global locations and lock file

The CLI defines Pi as a non-universal agent with global target `~/.pi/agent/skills/` (`src/agents.ts:549-556`). Pi independently discovers `~/.pi/agent/skills/` and `~/.agents/skills/` (`packages/coding-agent/docs/skills.md:24-40`).

The CLI's global lock is:

- `$XDG_STATE_HOME/skills/.skill-lock.json` when `XDG_STATE_HOME` is set; otherwise
- `~/.agents/.skill-lock.json`.

This is implemented by `src/skill-lock.ts:64-75`. The current lock schema is version 3 (`src/skill-lock.ts:8-10`), with per-skill `source`, `sourceType`, `sourceUrl`, optional `ref`, `skillPath`, `skillFolderHash`, and installation/update timestamps (`src/skill-lock.ts:12-40`). Adding or updating a skill preserves the original `installedAt` and refreshes `updatedAt` (`src/skill-lock.ts:206-225`).

Sandbox commands must set `HOME`, `XDG_STATE_HOME`, and `PI_CODING_AGENT_DIR` before starting the CLI process. The CLI captures `homedir()` during module initialization (`src/agents.ts:1-10`), so changing these variables after `npx` has started is too late.

## Hash and update behavior

For GitHub sources, `skillFolderHash` is the GitHub tree SHA for the entire skill directory, so any file change in that directory changes the hash (`src/skill-lock.ts:26-31`). `updateGlobalSkills` fetches the repository tree and compares the current folder hash with the lock value (`src/update.ts:541-577`). For non-Git sources it clones and computes the local folder hash (`src/update.ts:582-618`).

When an update is found, the CLI invokes its own executable with argv equivalent to:

```text
<node> <cli-entry> add <source-from-lock> [-full-depth-if-needed] -g -y
```

This call is made with `shell: false` and passes the source/ref as a discrete argv value (`src/update.ts:654-700`). The update command reports failures and sets a non-zero exit code if any requested update fails (`src/update.ts:933-982`).

## Important layout caveat in `update`

The child `add` call above does **not** pass `--agent pi` or `--copy` (`src/update.ts:677-690`). The add path with `-y` detects installed agents and appends universal agents (`src/add.ts:669-728`, `src/add.ts:1039-1084`, `src/add.ts:1461-1469`). The installer treats universal agents as canonical `.agents/skills` and Pi as an agent-specific `~/.pi/agent/skills` target (`src/agents.ts:766-831`, `src/installer.ts:113-148` and `src/installer.ts:291-371`).

Consequences to test and document:

- an initial explicit Pi-only `--copy` install can be converted by a later official `update` into a canonical `~/.agents/skills` copy plus a Pi symlink;
- if other supported agents are detected, the child add may target them too; and
- the resulting Pi resource loader can still deduplicate a symlinked copy, but separate physical copies with the same name can collide.

The implementation must either accept and test this official-update layout behavior or use the repair/add path for deterministic Pi-only placement. The source does not provide an update flag that combines hash-aware refresh with an explicit Pi-only copy target.

## Preview and confirmation limitations

The official CLI has no dry-run or preview flag for `skills update`. It performs the remote hash check, prints “found update” or “up to date”, and immediately invokes the child `add` with `-g -y` (`src/update.ts:647-700`). The add command can show an installation summary and ask for confirmation only when `--yes` is absent (`src/add.ts:804-857`); the update path deliberately supplies `-y`.

Therefore a `/preset-skills-sync` command that promises preview-before-write cannot use the official update prompt as its only confirmation. It needs a local read-only plan and an explicit Pi confirmation before invoking the CLI. A hash-level preview can report skill names, source, old lock hash, candidate new hash, target paths, and whether the operation is install or update. Obtaining the candidate hash requires a read-only network check; the official CLI does not expose that check as a dry-run API, so duplicating the check or showing only a less precise “the CLI will check” plan are separate implementation alternatives.

No skill body or other unneeded source content should be printed in the plan. The installed skill itself is third-party executable instruction and must be reviewed.

## Missing runner, network failure, and partial failure

### Missing `npx`

Preflight the runner before presenting an actionable plan:

```bash
command -v npx
node --version
```

If `npx` is absent but `npm` is available, the equivalent package runner is:

```bash
npm exec --yes --package=skills@latest -- skills add mattpocock/skills \
  --skill grill-me --skill grilling \
  --agent pi --global --copy --yes
```

If neither runner exists, stop before invoking any network or installer operation and report that Node/npm (with Node `>=22.20.0`) is required. Do not attempt a shell download or silently fall back to manual copying. The command runner should pass argv without a shell; the official update itself demonstrates this security property (`src/update.ts:678-690`).

### Network and Git failures

The CLI first tries its blob path for eligible GitHub sources and falls back to a clone if blob resolution fails (`src/add.ts:1178-1219`). The blob path refuses partial blob installation when any selected skill download fails (`src/blob.ts:541-552`). Clone failures clean the temporary directory and report timeout/auth recovery guidance (`src/git.ts:235-298`); the timeout is configurable with `SKILLS_CLONE_TIMEOUT_MS` (`src/git.ts:9-16`, `src/git.ts:252-261`).

The preset command should distinguish “runner unavailable”, “remote check failed”, “installer exited non-zero”, and “Pi reload still required”. It should not print raw environment variables, auth material, or unbounded CLI output. A recovery report should repeat the safe explicit `add` or `update` command, never a command containing a secret.

`pi --offline` only controls Pi startup network operations. It does not automatically prevent a newly spawned external `npx skills` process from using the network; offline tests must inject a failing runner or have the command explicitly honor the offline policy before spawning it.

### Partial installer failure

The official installer is not transactional across the selected skill/agent matrix. It installs each result in sequence (`src/add.ts:1740-1777`), records successful entries in the lock only (`src/add.ts:1844-1887`), reports failed entries (`src/add.ts:2026-2038`), and can leave successful updates in place when another selected target fails. The installer cleans/recreates a target directory before copying (`src/installer.ts:155-170`, `src/installer.ts:336-360`). It does not restore the previous copy on failure.

This is a direct risk against an acceptance criterion requiring the previous working pair to remain intact after any installer failure. A wrapper that requires that invariant must add an external transaction boundary, for example:

- stage the entire CLI run under isolated temporary `HOME`/state directories and promote the complete result only after both skills succeed; or
- snapshot both skill directories and the lock, run the CLI, and restore all snapshots on non-zero exit, with careful handling of symlinks, modes, and the two separate target roots.

Those are implementation alternatives to evaluate; invoking the official CLI directly is not sufficient to prove rollback safety.

## Duplicate skill handling

The CLI's explicit filters avoid selecting the rest of Matt's repository. Do not use `--all` because it expands to all skills, all agents, and `-y` (`src/add.ts:1066-1071`). The lock is keyed by skill name, so a repeated same-name entry is not a safe way to represent two sources.

At the Pi layer, the loader canonicalizes real paths and silently skips the same file reached through a symlink, but reports a name collision for separate files with the same skill name (`packages/coding-agent/src/core/skills.ts:394-426`). A duplicate test must therefore check both:

```bash
find "$HOME/.pi/agent/skills" "$HOME/.agents/skills" \
  -type f -path '*/SKILL.md' -print | sort
```

and Pi's RPC command list, including each command's `path`. A symlink and its canonical target are acceptable only when they resolve to the same real file; two independent copies are not.

## Additional network and telemetry behavior

The official CLI has two network surfaces beyond the GitHub/npm package fetch:

- `src/telemetry.ts:1-22`, `86-89`, and `142-189` sends anonymous install/update telemetry to `https://add-skill.vercel.sh/t` unless `DISABLE_TELEMETRY` or `DO_NOT_TRACK` is set; the README documents those opt-outs at `/tmp/vercel-skills-research/README.md:499-516`.
- `src/telemetry.ts:94-135` starts a security-audit request to `https://add-skill.vercel.sh/audit` for selected skills. The audit request is best-effort and never blocks installation, but it is still an outbound request. Do not supply the CLI's `--metadata` option for this integration, because it accepts caller-provided JSON that would become telemetry data (`src/add.ts:540-550`, `2156-2197`).

Offline and privacy tests should set `DISABLE_TELEMETRY=1` and `DO_NOT_TRACK=1`, inject/fake the audit/network layer where possible, and assert that no secret or arbitrary environment data enters argv or telemetry metadata. Whether the production preset command should set these variables is a separate product/operational decision; the source behavior must be documented rather than assumed away.

## Why this must not be a Pi package install

Pi's package documentation warns that packages have full system access and that extensions/skills can cause arbitrary actions (`packages/coding-agent/docs/packages.md:18-20`). A package manifest can expose a `skills/` directory, and Pi recursively discovers those resources (`packages/coding-agent/docs/packages.md:116-165`). Installing `git:github.com/mattpocock/skills` as a Pi package would therefore load the repository's other skills and violate the exact-two-skill requirement. The official CLI with explicit `--skill` filters is the narrower boundary.
