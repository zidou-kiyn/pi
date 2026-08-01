# Upstream grill-me and grilling contract

## Research snapshots

- Matt Pocock repository: `https://github.com/mattpocock/skills`
- Snapshot researched: `2ab958093e83e0ec752e6c1c5932da465bf23e0c`
- Vercel CLI snapshot used for the installer analysis: `1164afa5f0e21ebd01e6fc11249759353f494ad1`
- The exact files below were read from the Matt repository snapshot, not copied into the preset.

## Files and behavior

`skills/productivity/grill-me/SKILL.md:1-7` is only:

```markdown
---
name: grill-me
description: A relentless interview to sharpen a plan or design.
disable-model-invocation: true
---

Run a `/grilling` session.
```

`skills/productivity/grilling/SKILL.md:1-12` owns the behavior. It requires:

- relentless interviewing until shared understanding;
- resolving the decision tree and dependencies one decision at a time;
- exactly one question at a time, waiting for the answer;
- a recommended answer with each question;
- checking discoverable environment facts instead of asking for them; and
- no action until the user confirms shared understanding.

The two directories are independent. There is no transitive dependency field in the Agent Skills format, so installing only `grill-me` does not make `grilling` available. The upstream README makes the same separation visible in its role list at `README.md:219-228`.

Each directory also contains `agents/openai.yaml`, but the Pi skill resource is the directory's `SKILL.md`; those metadata files are not additional Pi skills.

## Pi invocation semantics

Pi's skill frontmatter documentation says `disable-model-invocation: true` hides a skill from the model prompt while preserving explicit `/skill:name` invocation (`packages/coding-agent/docs/skills.md:137-150`). The loader implements this distinction in `packages/coding-agent/src/core/skills.ts:327-336`.

Therefore the expected user-facing pair is:

```text
/skill:grill-me
/skill:grilling
```

`grill-me` should remain the explicit wrapper; `grilling` should remain the reusable primitive. Do not rewrite either body or merge them into one local skill.

## Fidelity checks

The files at the researched snapshot have these SHA-256 values:

```text
6189dfceb7304a6e5558f75d87e68fa3bc7fcf7ba120e44f21f8a61fe01eba54  skills/productivity/grill-me/SKILL.md
44331dda57f461db4fec3f2efb6ddabe7aaaa0a57ae0f88a883bc61aed8a0587  skills/productivity/grilling/SKILL.md
```

A local validation can compare the installed files with a separately cloned upstream snapshot without introducing a copy into `pi-preset`:

```bash
sha256sum "$HOME/.pi/agent/skills/grill-me/SKILL.md" \
         "$HOME/.pi/agent/skills/grilling/SKILL.md"
```

For a clean experiment, compare against `git show 2ab958093e83e0ec752e6c1c5932da465bf23e0c:<path>` or the hashes above. The public preset should document the upstream source and install it through the official CLI; it should not vendor or behaviorally fork these files.

## Excluding the rest of Matt's collection

The upstream README warns that installing two distribution mechanisms can produce duplicate copies (`README.md:25-28`). For this integration, use only the official `skills` CLI path and explicit skill filters:

```bash
npx --yes skills@latest add mattpocock/skills \
  --skill grill-me --skill grilling \
  --agent pi --global --copy --yes
```

Do not use `--all`, omit `--agent`, install the Claude plugin, or install the whole Matt repository as a Pi package. The exact selected names are parsed by `src/add.ts:615-667` and the explicit agent is honored by `src/add.ts:673-687` in the Vercel CLI snapshot. `--yes` also suppresses the CLI's optional post-install `find-skills` prompt (`src/add.ts:2071-2129`), which is necessary to avoid an unrelated skill.

A clean-sandbox assertion should require that the global Pi directory contains exactly `grill-me` and `grilling`, and that Pi's `get_commands` response contains exactly `skill:grill-me` and `skill:grilling` from the intended paths. Check both `~/.pi/agent/skills/` and `~/.agents/skills/` because Pi discovers both locations.

## License and attribution

The upstream repository is MIT, copyright Matt Pocock 2026 (`LICENSE:1-20`). The notice requires the copyright and permission notice in copies or substantial portions. The preset does not redistribute the upstream files if it only invokes the official CLI at install time, but its README must still identify the repository, author, MIT license, and that the files remain upstream-owned. If any upstream file is later committed or bundled, include the full MIT notice rather than relying only on a link.

The upstream behavior itself runs as agent instructions. Pi's security warning says skills can instruct the model to perform any action and may include executable code (`packages/coding-agent/docs/skills.md:20-23`); the README should tell users to review the two upstream files before use.

## Local baseline observed during research

At the time of research, `/home/heixiaohu/.pi/agent/skills/` contained only the unrelated `officecli` skill, and `/home/heixiaohu/.agents/.skill-lock.json` was absent. No local `grill-me` or `grilling` copy was installed by this research session; installation commands are documented for the implementation/verification phase.
