# Journal - zidou-kiyn (Part 1)

> AI development session journal
> Started: 2026-07-30

---

## 2026-07-31 — 00-bootstrap-guidelines completed

Replaced the 136-file placeholder `.trellis/spec/` tree with 37 evidence-backed
files: 3 `guides`, 5 `_shared`, 29 package-layer docs across 7 product packages.
`config.yaml` pruned to the 7 packages, `default_package: pi-agent-core` fixed.

Execution: `_shared` written in the main session as the style reference, then
package layers via `trellis-implement` sub-agents (Batch A: pi-ai,
pi-coding-agent, pi-tui; Batch B: pi-agent-core, pi-evals, pi-server,
pi-storage-sqlite-node). Consistency pass added missing H1s and unified link
style. `trellis-check` returned 0 CRITICAL / 11 WARNING / 16 INFO; every
WARNING was independently verified against source and fixed.

### Incident: spec tree partially rolled back mid-session

Around 13:46-13:47 the four Batch B package directories (10 files) were deleted
and several Batch A edits reverted. Cause: a second pi process (PID 1014105,
started 13:46) operating in the same cwd. `.trellis/` was untracked, so git
could not restore it; the Step 0 backup predated Batch B and was useless.
Recovery was possible only because every lost file had been read in full
earlier in the session and was reconstructible verbatim from context.

Lessons:
- Snapshot after each batch, not once at Step 0. A single pre-work backup does
  not cover work produced during the task.
- Untracked `.trellis/` has no recovery path under concurrent sessions. The
  developer approved committing `.trellis/spec/**` and `.trellis/config.yaml`
  for exactly this reason.

### Deviations from plan

- K6 (clean `guides/`) was added mid-execution; the planning assumption that
  the guides were repo-agnostic was wrong.
- design.md §8's 150-line target is now a soft target; 4 files exceed it
  because the overflow is verified evidence (Anti-Patterns tables, Known Debt).
  Exceptions recorded in design.md §8.
