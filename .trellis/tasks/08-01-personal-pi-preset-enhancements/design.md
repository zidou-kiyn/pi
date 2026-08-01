# Design: personal Pi preset enhancements

## Delivery boundary

The product changes live in the standalone public repository at `/home/heixiaohu/pi-preset`. The Pi monorepo receives only Trellis planning/spec records; no Pi core API change is required.

The parent task has no direct product implementation. It coordinates two child deliverables in this order:

1. `integrate-grill-me`
2. `interactive-models-config`

The order is explicit because the user requested local `grill-me` validation before any further preset enhancement.

## Child architecture

### Upstream skills child

- Ship a dedicated `/preset-skills-sync` extension command.
- Keep `grill-me` and `grilling` outside the preset repository.
- Use the official `skills` CLI with fixed filters to install or refresh exactly those two global Pi skills.
- Validate locally first, then add the command to the public preset.
- Reload Pi resources after a changed install.

### Models wizard child

- Ship a dedicated TUI-only `/preset-models-add` extension command.
- Keep provider templates credential-free in the public repository.
- Collect family, one or more model selections, provider identifier, base URL, and a masked API key.
- Preview a redacted provider-only diff and use a second confirmation for replacement collisions.
- Atomically upsert only the target provider in `models.json`.

## Shared invariants

- Existing `/preset-sync` behavior remains isolated and unchanged.
- No secret, private endpoint, or user-specific provider identifier enters the public repository or its history.
- External writes are plan/confirm/apply flows with explicit failure reporting.
- Every mutable user file or directory has a rollback artifact before replacement.
- No real provider request is part of automated validation.

## Integration data flow

```text
pi-preset install
  ├─ /preset-sync          -> existing local package/config/font reconciliation
  ├─ /preset-skills-sync   -> official skills CLI -> two global Pi skills -> ctx.reload()
  └─ /preset-models-add    -> masked TUI wizard -> models.json -> open /model
```

The three commands do not call one another. A network/npm failure in the skills command cannot block local preset reconciliation, and a credential-entry cancellation cannot affect packages, fonts, or web-search settings.

## Compatibility and rollback

- Pi package updates continue through the existing unpinned preset git source.
- Matt Pocock skill updates are intentionally separate and user-triggered.
- Skill rollback restores the previous two skill entries and global skill lock.
- Models rollback uses the existing `.preset-bak` convention and a sibling atomic rename.
- README instructions distinguish `/reload` after skill changes, `/model` after `models.json` changes, and the existing full restart required after `packages[]` changes.

## Commit and rollout shape

Each child lands as its own atomic commit in `pi-preset` after its own tests pass. The second child builds on the published first child. The parent closes only after a clean-install integration smoke test and a final secret-history scan.
