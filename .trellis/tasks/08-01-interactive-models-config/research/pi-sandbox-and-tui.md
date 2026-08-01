# Pi discovery, reload, sandbox, and TUI verification

## Pi discovery contract

Pi documents global skill locations as `~/.pi/agent/skills/` and `~/.agents/skills/`, and project locations as trusted `.pi/skills/` and ancestor `.agents/skills/` (`packages/coding-agent/docs/skills.md:20-41`). It scans names/descriptions at startup and exposes explicit commands as `/skill:name` (`packages/coding-agent/docs/skills.md:64-90`). Missing descriptions prevent loading; same-name resources from different locations warn and keep the first (`packages/coding-agent/docs/skills.md:176-188`).

The source confirms that user resources are collected from both `~/.pi/agent/skills/` and `~/.agents/skills/` (`packages/coding-agent/src/core/package-manager.ts:2396-2426`). Resolved paths are canonicalized and duplicate real paths are removed (`packages/coding-agent/src/core/package-manager.ts:2502-2527`). The lower-level skill loader applies the same real-path deduplication, but reports separate-file name collisions (`packages/coding-agent/src/core/skills.ts:394-426`).

The relevant verification must run from a clean working directory with no project `.pi/skills` or `.agents/skills`, and must inspect both global roots. A global install alone does not prove that a project-local duplicate is absent.

## Reproducible clean sandbox

The following is a validation command, not a request to manually copy skill files. It makes the external CLI write only to a temporary home and state directory:

```bash
SANDBOX="$(mktemp -d)"
mkdir -p "$SANDBOX/home" "$SANDBOX/state" "$SANDBOX/work"
export HOME="$SANDBOX/home"
export XDG_STATE_HOME="$SANDBOX/state"
export PI_CODING_AGENT_DIR="$HOME/.pi/agent"
cd "$SANDBOX/work"

npx --yes skills@latest add mattpocock/skills \
  --skill grill-me --skill grilling \
  --agent pi --global --copy --yes
```

The CLI uses `homedir()` for global paths, while Pi's config resolver honors `PI_CODING_AGENT_DIR` (`packages/coding-agent/src/config.ts:510-521`). Set all variables before starting `npx`; the official CLI captures its home value at module load (`/tmp/vercel-skills-research/src/agents.ts:1-10`). `XDG_STATE_HOME` deliberately makes the lock location explicit for the test.

### Exact file and lock assertions

```bash
set -eu

[ -f "$HOME/.pi/agent/skills/grill-me/SKILL.md" ]
[ -f "$HOME/.pi/agent/skills/grilling/SKILL.md" ]

actual="$(find "$HOME/.pi/agent/skills" -mindepth 1 -maxdepth 1 \
  -type d -exec basename {} \; | LC_ALL=C sort)"
expected=$'grill-me\ngrilling'
test "$actual" = "$expected"

LOCK="$XDG_STATE_HOME/skills/.skill-lock.json"
jq -e '
  (.version >= 3) and
  ([.skills | keys[]] | sort == ["grill-me", "grilling"]) and
  (.skills["grill-me"].source == "mattpocock/skills") and
  (.skills["grilling"].source == "mattpocock/skills") and
  (.skills["grill-me"].skillFolderHash | length > 0) and
  (.skills["grilling"].skillFolderHash | length > 0)
' "$LOCK"

sha256sum "$HOME/.pi/agent/skills/grill-me/SKILL.md" \
          "$HOME/.pi/agent/skills/grilling/SKILL.md"
```

The expected hashes for the researched upstream snapshot are recorded in `upstream-skills.md`. An implementation test can instead compare to a separate checkout using `git show`; it must not commit a local copy to `pi-preset`.

### RPC discovery assertion

Pi's RPC `get_commands` response includes skill names, source, location, and absolute path (`packages/coding-agent/docs/rpc.md:791-830`). Run it from the clean sandbox:

```bash
printf '%s\n' '{"type":"get_commands"}' |
  env HOME="$HOME" PI_CODING_AGENT_DIR="$PI_CODING_AGENT_DIR" \
      XDG_STATE_HOME="$XDG_STATE_HOME" \
  pi --mode rpc --no-session |
  jq -s '[ .[]
    | select(.type == "response" and .command == "get_commands")
    | .data.commands[]
    | select(.source == "skill")
    | .name ] | sort'
```

Expected output in the clean sandbox:

```json
["skill:grill-me", "skill:grilling"]
```

A second query should inspect paths and reject any skill command whose path is outside the two intended directories:

```bash
printf '%s\n' '{"type":"get_commands"}' |
  env HOME="$HOME" PI_CODING_AGENT_DIR="$PI_CODING_AGENT_DIR" \
      XDG_STATE_HOME="$XDG_STATE_HOME" \
  pi --mode rpc --no-session |
  jq -s '[ .[]
    | select(.type == "response" and .command == "get_commands")
    | .data.commands[]
    | select(.source == "skill" and
             (.name == "skill:grill-me" or .name == "skill:grilling"))
    | {name, path} ]'
```

## Reload behavior

After an external CLI changes `SKILL.md` files, the running Pi process has stale resource state. `/reload` reruns the resource reload flow; `ctx.reload()` is the extension API equivalent (`packages/coding-agent/docs/extensions.md:1275-1299`). The session reload path reloads settings, providers, and the resource loader (`packages/coding-agent/src/core/agent-session.ts:2602-2613`). Skill commands are rebuilt from the reloaded skill list (`packages/coding-agent/src/modes/interactive/interactive-mode.ts:637-649`).

For a skill-only refresh, `/reload` should be sufficient; a full Pi restart is an acceptable fallback for a manual verification. A command handler calling `await ctx.reload()` must treat reload as terminal and return, because code after it uses stale extension state (`packages/coding-agent/docs/extensions.md:1289-1299`). Do not reuse an old context after reload.

This differs from `/preset-sync`: that command changes `settings.json` `packages[]`, and its README warns that the current session keeps a stale package snapshot and must restart before `/config` or `pi install` (`/home/heixiaohu/pi-preset/extensions/preset-sync.ts:83-96`, `/home/heixiaohu/pi-preset/README.md:100-106`). The skill command writes the external skill roots and lock, not Pi's package list, so its post-success instruction can be `/reload` rather than a package-install restart.

## TUI smoke test

The upstream contract is behavioral and model-dependent. Use a real interactive TUI only for a smoke test with a configured provider; do not put an API key in a command, prompt, transcript, or test fixture. Restrict tools to read-only discovery if the test should prove that interview-time environment inspection cannot edit files:

```bash
SESSION="pi-grill-smoke-$$"
tmux new-session -d -s "$SESSION" -x 120 -y 40

tmux send-keys -t "$SESSION" \
  "HOME=$HOME PI_CODING_AGENT_DIR=$PI_CODING_AGENT_DIR XDG_STATE_HOME=$XDG_STATE_HOME pi --no-session --tools read,grep,find" Enter
sleep 3
tmux capture-pane -t "$SESSION" -p

tmux send-keys -t "$SESSION" \
  "/skill:grill-me I want a command that synchronizes third-party skills, but the scope and failure behavior are still undecided." Enter
sleep 15
tmux capture-pane -t "$SESSION" -p

tmux kill-session -t "$SESSION"
```

Acceptance observations:

1. the wrapper delegates to `grilling`;
2. at least three questions are asked in separate turns;
3. each question includes a recommended answer;
4. later questions depend on earlier answers;
5. environment facts are inspected rather than repeatedly requested; and
6. no implementation begins before the user confirms shared understanding.

The transcript should be reviewed manually or against a deterministic faux-provider harness. Do not assert exact wording because the upstream skill intentionally leaves the questions to the model. A second direct `/skill:grilling` invocation verifies that the primitive itself is present, not merely the wrapper.

## Mode matrix for `/preset-skills-sync`

Pi extension contexts distinguish TUI/RPC from print/JSON (`packages/coding-agent/docs/extensions.md:940-946`, `2889-2898`):

| Mode | `ctx.hasUI` | Research-required behavior |
|---|---:|---|
| TUI | true | Render plan, ask explicit confirmation, invoke CLI only after approval, report result, offer `/reload`. |
| RPC | true | Use dialog requests over the RPC UI protocol; do not use TUI-only custom components. `custom()` is unavailable (`packages/coding-agent/docs/rpc.md:1155-1164`). |
| JSON | false | Report a plan/error and perform no network or write because there is no consent channel. |
| Print (`-p`) | false | Same no-consent/no-write behavior; direct the user to interactive TUI/RPC mode. |

The existing `/preset-sync` already follows the safe non-UI pattern: it prints a dry-run plan and returns when `ctx.hasUI` is false (`/home/heixiaohu/pi-preset/extensions/preset-sync.ts:60-71`). The new command should not silently treat a non-TUI invocation as approval.

## Offline and sandbox test matrix

The implementation should isolate pure planning from the external command runner so these tests can run without network:

- **Runner absence:** fake `PATH` with no `npx`/`npm`; assert an actionable error and unchanged skill files/lock.
- **Offline:** inject a runner that returns a network error, or explicitly make the command honor an offline test flag before spawning; assert no partial install and no Pi preset file changes. `pi --offline` by itself does not constrain a child `npx` process.
- **Decline:** TUI/RPC confirmation false; assert runner not called and all target mtimes unchanged.
- **First install:** clean `HOME`, `XDG_STATE_HOME`, and `PI_CODING_AGENT_DIR`; assert exactly two directories and two lock entries.
- **No-change update:** valid v3 lock and unchanged hashes; assert no child `add` invocation and no duplicate files.
- **Changed update:** altered upstream folder hash; assert exactly the expected update/repair argv and then `/reload` exposes the same two names.
- **CLI partial failure:** fake a non-zero runner after one skill succeeds; assert the wrapper's chosen rollback/staging invariant, because the official CLI itself leaves successful results in place.
- **Duplicate path:** place a symlink and target for one skill; accept one Pi command only when `realpath` is identical. Place two independent copies and require a collision diagnostic/failure rather than silently accepting the first.
- **TUI smoke:** use tmux and capture only non-secret prompts/results; confirm the three-question grilling behavior separately from installer unit tests.
