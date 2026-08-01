# Design: upstream grill-me integration

## Delivery boundary

The upstream skill files are not committed to `pi-preset`. The preset ships only a command that installs or refreshes the selected upstream content through the official Vercel `skills` CLI.

The implementation has two stages:

1. install and behaviorally validate `grill-me` + `grilling` in the real local Pi;
2. only after that succeeds, implement and validate `/preset-skills-sync` in `/home/heixiaohu/pi-preset`.

## Upstream unit

Both directories are required:

```text
skills/productivity/grill-me/
skills/productivity/grilling/
```

`grill-me` remains an explicit wrapper and `grilling` remains the model-invoked interview primitive. The preset never combines or rewrites them.

## Command architecture

```text
extensions/preset-skills-sync.ts
  -> read-only preflight and rendered plan
  -> ctx.ui.confirm (TUI/RPC)
  -> snapshot current skill/lock state
  -> fixed no-shell skills CLI invocation
  -> validate exactly two intended results
  -> rollback on any failure
  -> compare folder hashes for installed/updated/current report
  -> ctx.reload() when content changed
```

Suggested pure modules:

```text
src/skills-sync-plan.ts    paths, runner discovery, duplicate checks, lock inspection
src/skills-sync-apply.ts   snapshot, process execution, validation, rollback
```

The extension entry only adapts Pi mode/UI/reload APIs. Pure modules use Node APIs and accept injected process runners where tests need deterministic failures.

## Fixed CLI contract

Every real synchronization uses the same explicit add command:

```bash
npx --yes skills@latest add mattpocock/skills \
  --skill grill-me --skill grilling \
  --agent pi --global --copy --yes
```

The implementation does not call `skills update`: the current update path drops `--agent pi` and `--copy`, can retarget detected agents, and can convert a Pi-only copy into a canonical/symlink layout.

If `npx` is unavailable, use the no-shell equivalent:

```bash
npm exec --yes --package=skills@latest -- skills add ...
```

All arguments are fixed argv entries. The child environment sets `DISABLE_TELEMETRY=1`, `DO_NOT_TRACK=1`, and `npm_config_ignore_scripts=true`. No metadata option is passed.

## Paths and duplicate handling

The command inspects:

- `~/.pi/agent/skills/grill-me`
- `~/.pi/agent/skills/grilling`
- `~/.agents/skills/grill-me`
- `~/.agents/skills/grilling`
- `$XDG_STATE_HOME/skills/.skill-lock.json` or `~/.agents/.skill-lock.json`

A symlink and its canonical target are one resource when `realpath` matches. Two independent same-name copies are a blocker: the command reports both paths and requires the user to resolve ownership instead of deleting intentional third-party content.

## Snapshot and rollback

Before invoking the CLI, copy the four possible skill entries and the complete lock file into a private temporary directory, preserving symlinks and modes. Record which paths were absent.

After invocation, validate all of the following before accepting the result:

- Pi target files exist for both fixed names;
- frontmatter names are `grill-me` and `grilling`;
- global lock entries use source `mattpocock/skills` and the expected upstream paths;
- no separate same-name copy exists in the other global root;
- no unrelated Matt skill was installed into the Pi target by this run.

On process failure or validation failure, remove only the newly produced versions of the five tracked paths and restore the snapshot. npm caches and temporary download directories are outside the user-visible transaction and need no rollback.

Content-folder hashes before and after determine the report:

- absent -> present: installed;
- different: updated;
- equal: already current.

## Mode behavior

- TUI: render plan, confirm, run, report, reload on content change.
- RPC: use the extension UI confirm protocol, then the same apply path; no custom TUI component is required.
- Print/JSON: report that interactive consent is required and perform no network or write. JSON stdout remains valid JSONL; diagnostics go to stderr/no-op UI as appropriate.

`ctx.reload()` is terminal for the command handler. Notify success before calling it and return immediately afterward.

## Local behavioral validation

After the direct local install, run `/skill:grill-me` against an underspecified plan. The smoke test must observe at least three separate dependent questions, recommendations with each question, environment lookup for facts, and no implementation before shared-understanding confirmation. A direct `/skill:grilling` invocation verifies the primitive separately.

## Documentation and license

README additions identify Matt Pocock, the upstream repository, and MIT license; explain why both skills are required; document `/preset-skills-sync`, direct skill commands, reload, runner/network recovery, state paths, and Pi's warning that skills execute as agent instructions.

Because the preset does not redistribute the upstream files, it links and attributes rather than embedding the full license text. If that distribution boundary changes later, the full upstream MIT notice becomes required.
