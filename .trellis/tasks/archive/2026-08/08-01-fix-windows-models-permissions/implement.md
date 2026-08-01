# Implement: Windows models permission compatibility

Workspace: `/home/heixiaohu/pi-preset`.

## Implementation

- [x] Add a platform-aware permission decision in `src/models-config.ts` that bypasses only the POSIX group/other bit check on `win32`.
- [x] Keep target resolution, dangling-symlink rejection, reads, and ordinary filesystem errors unchanged.
- [x] Update README permission wording for POSIX modes versus Windows ACLs.

## Regression tests

- [x] Add simulated-Windows first-add coverage.
- [x] Add simulated-Windows second identical add/no-op coverage, including unchanged bytes, mtime, and backup absence.
- [x] Add simulated-Windows replacement coverage, including provider-as-unit replacement, sibling preservation, and original backup bytes.
- [x] Retain explicit simulated-Linux and simulated-macOS broad-mode rejection before credential entry.
- [x] Make exact numeric mode assertions platform-correct without weakening POSIX assertions.

## Validation

```bash
cd /home/heixiaohu/pi-preset
npm install --ignore-scripts --no-package-lock
node --test \
  test/model-templates.test.ts \
  test/models-config.test.ts \
  test/models-config-apply.test.ts \
  test/models-wizard-workflow.test.ts \
  test/json-merge.test.ts \
  test/skills-sync-plan.test.ts \
  test/skills-sync-apply.test.ts
./scripts/scan-secrets.sh

cd /home/heixiaohu/桌面/pi/packages/coding-agent
node ../../node_modules/vitest/dist/cli.js --run test/suite/regressions/6999-models-json-hot-reload.test.ts

cd /home/heixiaohu/桌面/pi
npm run hydrate:model-data
npm run check
```

The previously accepted stale ZAI model-ID failures remain unrelated unless their error set changes.

## Validation result

- Product regression suite: 89 passed, 0 failed.
- Secret scan: working tree and full history clean.
- Pi hot-reload regression: 1 passed, 0 failed.
- Model-data hydration: passed without tracked generated changes.
- Root `npm run check`: only the accepted 15 stale ZAI references remained (9 `glm-4.5-air`, 6 `glm-5.1`); no new failures.
- Final `trellis-check`: no findings.

## Spec update review

No `.trellis/spec/` update is needed. The change fixes platform-specific behavior in the external `/home/heixiaohu/pi-preset` package without changing a Pi core extension API or repository-wide convention. The reusable contract is captured in this task's design, the public README, and platform-specific regression tests; adding it to the Pi core extension spec would incorrectly broaden that spec's scope.
