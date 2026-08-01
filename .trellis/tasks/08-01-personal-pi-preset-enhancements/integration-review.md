# Final integration review

Date: 2026-08-01

Product revision: public `zidou-kiyn/pi-preset` `main` at `7722cae4ea8b33fb1f96a1c03ed0865b31825562`.

## Results

- Installed `git:github.com/zidou-kiyn/pi-preset` into a clean temporary `HOME`, `XDG_STATE_HOME`, and `PI_CODING_AGENT_DIR`. The installed clone came from GitHub and matched `7722cae`.
- RPC discovery found exactly one registration each for `/preset-sync`, `/preset-skills-sync`, `/preset-models-add`, and `/vibrant-footer`, both before and after resource reload.
- `/preset-skills-sync` installed both upstream skills through the confirmed RPC flow and reloaded Pi. Discovery then found exactly one `/skill:grill-me` and one `/skill:grilling`. The Pi skill root contained only those two directories, the duplicate-check root was absent, and the v3 lock contained exactly the two `mattpocock/skills` source entries with the expected upstream paths and non-empty hashes.
- Ran `/preset-models-add` in tmux with OpenAI and explicitly selected all three models from the initially empty checklist. A runtime-generated credential was pasted through a tmux buffer. The preview showed `<redacted-supplied>`, the resulting `models.json` had mode `0600`, and `PI_OFFLINE=1 pi --list-models` discovered exactly `gpt-5.6-sol`, `gpt-5.6-terra`, and `gpt-5.6-luna` under the selected provider without sending a provider request.
- The runtime credential was absent from all tmux captures, the raw `PI_TUI_WRITE_LOG`, success notifications, RPC logs, install/model-list logs, and other sandbox logs. It appeared only in the expected runtime seed and `models.json`.
- `/preset-sync` first added the 13 declared packages and the two web-search leaves while preserving unrelated nested data. A second run requested no confirmation and left `settings.json`, `web-search.json`, and `models.json` byte-for-byte and mtime-for-mtime unchanged.
- Pi-preset Node tests: 85 passed, 0 failed.
- Pi models hot-reload regression: 1 passed, 0 failed.
- `scripts/scan-secrets.sh`: working tree clean across 28 files; full reachable history clean.
- README review passed: the tested install, sync, TUI-only wizard, reload/restart, `/model`, backup, credential, and recovery instructions match observed behavior.
- `npm run hydrate:model-data` passed and produced no tracked changes.
- No new durable Trellis contract was discovered, so no spec update was needed.

## Repository-wide check

`npm run check` completed Biome, pinned-dependency, TypeScript-import, shrinkwrap, and coding-agent install-lock checks, then exited in `tsgo --noEmit` on the same 15 previously accepted stale ZAI model references: nine `glm-4.5-air` references and six `glm-5.1` references. No new error category or task-related failure appeared. Browser smoke was not reached because type-checking exited first.

## Conclusion

No integration defect was found and no product code or documentation was changed. The only blocker is the already accepted, unchanged stale ZAI model-ID type-check failure in the Pi monorepo.
