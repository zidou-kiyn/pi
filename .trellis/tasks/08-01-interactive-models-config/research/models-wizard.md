# Models wizard research

## Pi contracts

- Custom providers are read from `~/.pi/agent/models.json`, or `<PI_CODING_AGENT_DIR>/models.json` when that environment variable is set (`packages/coding-agent/src/config.ts:494-531`, `packages/coding-agent/src/core/model-runtime.ts:135-144`).
- Opening `/model` calls `modelRuntime.refresh()` and re-reads the file, so a successful write needs no restart (`packages/coding-agent/src/modes/interactive/components/model-selector.ts:131-170`, `packages/coding-agent/src/core/model-runtime.ts:518-532`, `packages/coding-agent/test/suite/regressions/6999-models-json-hot-reload.test.ts:46-67`).
- Pi accepts `//` comments and trailing commas before JSON parsing through `stripJsonComments()` (`packages/coding-agent/src/utils/json.ts:1-6`, `packages/coding-agent/src/core/model-config.ts:241-279`).
- The file root and `providers` value are objects; provider/model/cost/thinking schemas are defined in `packages/coding-agent/src/core/model-config.ts:55-205`.

## Secret input and modes

- `ctx.ui.input()` uses the ordinary `Input` component and renders its real value (`packages/coding-agent/src/modes/interactive/components/extension-input.ts:63-80`, `packages/tui/src/components/input.ts:378-385`).
- RPC free-form input also has no secret flag, and `custom()` is unavailable in RPC (`packages/coding-agent/docs/rpc.md`, Extension UI Protocol).
- The safe path is a TUI-only `ctx.ui.custom()` component that delegates editing to `Input.handleInput()` but never calls `Input.render()`. It renders a fixed mask independent of secret length and emits `CURSOR_MARKER` as a `Focusable` component (`packages/coding-agent/docs/tui.md`, Focusable Interface and Custom Components).
- The command must branch on `ctx.mode === "tui"`, not `ctx.hasUI`, because RPC has UI dialogs but no masked custom component (`packages/coding-agent/docs/extensions.md`, Mode Behavior).

## Safe write requirements

- The current preset writer already resolves valid symlinks, backs up existing files, preserves existing mode, and uses sibling tmp-file rename (`/home/heixiaohu/pi-preset/src/json-merge.ts`).
- It needs options for a new-file mode, dangling-symlink rejection, and unique sibling tmp names. The models command must pass `newFileMode: 0o600`.
- Existing files with group/other permission bits must be rejected before collecting/writing the credential. Existing owner-only modes are preserved.
- The apply phase must re-read the file after confirmation, verify the selected provider still equals the preview baseline, preserve concurrent sibling changes, and then replace only `providers[providerId]`.

## Preview and redaction

- Redaction must happen before serialization. All `apiKey` values, header values, and credential-like fields (`authorization`, `token`, `secret`, `password`) are replaced with role-specific fixed markers.
- Existing and supplied API keys need distinct markers so a key-only replacement still produces a visible diff.
- A custom scrollable diff component should use injected keybindings (`tui.select.up/down/pageUp/pageDown/confirm/cancel`) and width-safe rendering.

## Test strategy

- Pure template, validation, planning, redaction, and apply tests can use Node's built-in test runner from the standalone preset repository without provider calls.
- TUI masking requires a tmux smoke test with a runtime-generated secret pasted through a tmux buffer; both `PI_TUI_WRITE_LOG` and captured output must exclude the secret.
- `pi --list-models` and the existing hot-reload regression verify discovery without sending inference requests.

## Refreshed local templates

The credential-free metadata was re-extracted from the current local `models.json` after the user's update.

- OpenAI now includes `gpt-5.6-luna`.
- `gpt-5.6-sol` context is 372,000 and its thinking map uses `off: "none"`.
- `gpt-5.6-terra` now carries a 272,000-token pricing tier and uses `off: "none"`.
- `gpt-5.6-luna` uses context 272,000, max output 128,000, base cost 0.2 / 1.2 / 0.02 / 0.25, and a long-context tier of 0.4 / 1.8 / 0.04 / 0.5.
- Anthropic and DeepSeek template metadata remain as previously recorded.

The wizard must select model membership independently from immutable model metadata. Every family checklist starts empty, at least one model is required, and selected templates are emitted in catalog order.
