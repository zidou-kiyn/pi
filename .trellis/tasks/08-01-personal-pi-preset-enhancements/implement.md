# Implement: personal Pi preset enhancements

The parent owns sequencing and integration review. Do not start it as an implementation task; start and archive each child independently.

## Ordered child execution

- [x] 1. Review and approve all parent/child planning artifacts.
- [x] 2. Start `08-01-integrate-grill-me`.
- [x] 3. Complete its local-first experiment, preset implementation, checks, product-repository commit, and archive.
- [x] 4. Start `08-01-interactive-models-config` only after step 3 is complete.
- [x] 5. Complete its implementation, checks, product-repository commit, and archive.

## Final integration review

- [x] 6. Create a clean temporary `HOME`, `XDG_STATE_HOME`, and `PI_CODING_AGENT_DIR`.
- [x] 7. Install the updated preset from its public git source.
- [x] 8. Verify `/preset-sync`, `/preset-skills-sync`, `/preset-models-add`, and `/vibrant-footer` each register exactly once.
- [x] 9. Run `/preset-skills-sync`; verify exactly one `grill-me` and one `grilling` command after reload.
- [x] 10. Run one provider-family wizard path; verify `pi --list-models` exposes its full fixed bundle without a provider request.
- [x] 11. Re-run the existing `/preset-sync` idempotence smoke test.
- [x] 12. Run the preset repository's targeted Node tests and `scripts/scan-secrets.sh` over working tree and history.
- [x] 13. Run `npm run check` from the Pi monorepo root as the required repository-wide quality gate.
- [x] 14. Review README install/update/reload/recovery instructions against the tested commands.
- [x] 15. Record integration results, update any durable Trellis spec that changed, and archive the parent task.

The final public-source integration result is recorded in `integration-review.md`. The root check remains nonzero only for the same 15 stale ZAI model references explicitly accepted by the user; all task-specific checks passed.

## Rollback points

- If the skills child fails, revert only its `pi-preset` commit; the models child has not started.
- If the models child fails, revert only its commit; `/preset-skills-sync` remains independently usable.
- If the final public history scan finds a secret, do not push further history. Recreate the public repository history and rotate any exposed credential.
