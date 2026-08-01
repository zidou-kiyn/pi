# Implement: personal Pi preset enhancements

The parent owns sequencing and integration review. Do not start it as an implementation task; start and archive each child independently.

## Ordered child execution

- [ ] 1. Review and approve all parent/child planning artifacts.
- [ ] 2. Start `08-01-integrate-grill-me`.
- [ ] 3. Complete its local-first experiment, preset implementation, checks, product-repository commit, and archive.
- [ ] 4. Start `08-01-interactive-models-config` only after step 3 is complete.
- [ ] 5. Complete its implementation, checks, product-repository commit, and archive.

## Final integration review

- [ ] 6. Create a clean temporary `HOME`, `XDG_STATE_HOME`, and `PI_CODING_AGENT_DIR`.
- [ ] 7. Install the updated preset from its public git source.
- [ ] 8. Verify `/preset-sync`, `/preset-skills-sync`, `/preset-models-add`, and `/vibrant-footer` each register exactly once.
- [ ] 9. Run `/preset-skills-sync`; verify exactly one `grill-me` and one `grilling` command after reload.
- [ ] 10. Run one provider-family wizard path; verify `pi --list-models` exposes its full fixed bundle without a provider request.
- [ ] 11. Re-run the existing `/preset-sync` idempotence smoke test.
- [ ] 12. Run the preset repository's targeted Node tests and `scripts/scan-secrets.sh` over working tree and history.
- [ ] 13. Run `npm run check` from the Pi monorepo root as the required repository-wide quality gate.
- [ ] 14. Review README install/update/reload/recovery instructions against the tested commands.
- [ ] 15. Record integration results, update any durable Trellis spec that changed, and archive the parent task.

## Rollback points

- If the skills child fails, revert only its `pi-preset` commit; the models child has not started.
- If the models child fails, revert only its commit; `/preset-skills-sync` remains independently usable.
- If the final public history scan finds a secret, do not push further history. Recreate the public repository history and rotate any exposed credential.
