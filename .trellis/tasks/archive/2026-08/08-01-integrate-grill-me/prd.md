# PRD: Integrate upstream grill-me skill

## Goal

Make Matt Pocock's upstream `grill-me` workflow available in the user's local Pi, prove that it conducts a deeper multi-turn requirements interview than the current Trellis brainstorm flow, and then integrate a maintainable upstream-refresh path into `zidou-kiyn/pi-preset`.

## Background and Confirmed Facts

- Upstream repository: `https://github.com/mattpocock/skills` (MIT).
- `skills/productivity/grill-me/SKILL.md` is a thin, explicitly invoked wrapper with `disable-model-invocation: true` whose body is `Run a /grilling session.`
- `skills/productivity/grilling/SKILL.md` owns the actual behavior:
  - interview relentlessly until shared understanding;
  - walk the decision tree branch by branch and resolve dependencies in order;
  - ask one question at a time and wait for feedback;
  - include a recommended answer with every question;
  - inspect the environment for facts instead of asking the user;
  - do not act until the user confirms shared understanding.
- The two skills are separate folders and the Agent Skills format has no transitive dependency field. Installing only `grill-me` does not guarantee that `grilling` is present.
- Pi supports these skills natively and exposes them as `/skill:grill-me` and `/skill:grilling`.
- The Vercel `skills` CLI supports `--agent pi --global`, installs into `~/.pi/agent/skills/`, records upstream paths/hashes in `~/.agents/.skill-lock.json`, and can later update selected global skills.
- The preset itself follows its default Git branch through `pi update --extensions`, but copying upstream skill files into the preset would require a separate mechanism to keep that copy current.

## Requirements

- **R1 Complete upstream unit:** install and manage both `grill-me` and `grilling`; never leave the wrapper without its primitive.
- **R2 Upstream fidelity:** use the upstream files without local behavioral edits. Preserve attribution and license information in preset documentation.
- **R3 Local-first experiment:** install both skills into the local Pi, restart/reload as required, and verify that both skill commands are discovered before changing the preset repository.
- **R4 Behavioral smoke test:** invoke `/skill:grill-me` with an intentionally underspecified plan and verify at least three dependent, one-at-a-time questions with recommendations before shared understanding is confirmed. It must not begin implementation during the interview.
- **R5 Preset reproducibility:** a fresh user following the preset documentation can obtain the same two skills without manually copying files.
- **R6 Upstream refresh:** provide an explicit one-command refresh backed by the official `skills` CLI, detect/report failures clearly, and avoid silently pinning an obsolete snapshot.
- **R7 No duplicate loading:** the integration must not load both preset-vendored and globally installed copies of the same skill name.
- **R8 Existing preset safety:** the integration must not add credentials, private endpoints, or unrelated Matt Pocock skills to the public preset.
- **R9 Dedicated entry point:** register `/preset-skills-sync` for both first installation and later update checks; do not add network/npm work to `/preset-sync`.
- **R10 Deterministic Pi-only layout:** invoke the official CLI with fixed `--skill grill-me --skill grilling --agent pi --global --copy --yes` arguments for both install and refresh so later updates cannot silently target other detected agents or switch to a canonical symlink layout.
- **R11 Failure rollback:** snapshot the two target skill entries and the global skill lock before invoking the CLI; on any non-zero or invalid post-install result, restore the complete previous state.
- **R12 Mode and reload behavior:** TUI and RPC require explicit confirmation before network or writes; print/JSON modes perform no install. After changed skill content is validated, reload Pi resources so the commands are available without a restart.
- **R13 Privacy:** disable optional `skills` CLI telemetry for preset-managed invocations and never pass arbitrary metadata.

## Acceptance Criteria

- **AC1** `pi` discovers exactly one `grill-me` and one `grilling` skill in the local experiment.
- **AC2** `/skill:grill-me` delegates successfully to the installed `grilling` behavior and completes the R4 multi-turn smoke test.
- **AC3** In a clean Pi sandbox, the documented preset flow installs or makes available both required skills and no unrelated Matt Pocock skills.
- **AC4** Running the documented upstream refresh after an upstream folder hash change updates both skills; a no-change run reports current state without creating duplicates.
- **AC5** Failure to reach GitHub/npm or failure of the external installer leaves the previous working skill copies intact and reports a recovery command.
- **AC6** The preset README identifies the upstream repository, explains why both skills are needed, and documents invocation and update commands.

## Out of Scope

- Modifying Trellis brainstorm to imitate `grill-me`.
- Installing `grill-with-docs`, `wayfinder`, or the rest of Matt Pocock's collection.
- Maintaining a behavioral fork of either upstream skill.
- Automatic execution of `grill-me` for every task; upstream keeps it user-invoked.

## Key Decision

- **D1 Explicit update:** do not load the full upstream repository as a Pi git package and do not vendor an automatically synchronized copy. Use the `skills` CLI to install/update exactly `grill-me` and `grilling`, preserving its global lock and upstream folder hashes.
- **D2 Dedicated command:** `/preset-skills-sync` owns all Matt Pocock skill install/update work so failures cannot partially block the preset's existing local reconciliation.
- **D3 Refresh implementation:** always run the explicit filtered `skills@latest add` command rather than `skills update`. The current update implementation drops `--agent pi` and `--copy`; content hashes before and after the add determine whether the result was installed, updated, or already current.
