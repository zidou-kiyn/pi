# PRD: Add interactive models configuration

## Goal

Add a deterministic interactive preset command that lets a user select Anthropic, OpenAI, or DeepSeek, choose one or more predefined models from that family, and provide a provider identifier, base URL, and API key. The command then previews and safely upserts the selected provider configuration into `~/.pi/agent/models.json`.

## User Outcome

A user chooses model names but does not need to understand Pi's `models.json` schema, API mode names, compatibility flags, thinking-level maps, token limits, or pricing metadata.

## Background and Confirmed Facts

- Pi reads custom providers from `~/.pi/agent/models.json`; the file is hot-reloaded whenever `/model` is opened.
- A non-built-in provider with custom models needs `baseUrl`, an API at provider or model level, and a `models` array. Authentication may be stored in provider `apiKey`.
- The local file already supplies the requested model metadata. Only its reusable family templates may be copied; local provider names, endpoints, and credentials are not preset data.
- Existing preset JSON utilities already abort on invalid JSON, preserve symlink targets and file modes, create `.preset-bak`, and write via sibling tmp-file rename.
- The simple extension `ctx.ui.input()` is ordinary text input and has no secret/masking option. A safe API-key prompt therefore needs a TUI-specific masked component or another interaction that does not echo the key.

## Supported Templates

### Anthropic

- Provider API: `anthropic-messages`
- Provider compatibility:
  - `supportsEagerToolInputStreaming: false`
  - `supportsLongCacheRetention: true`
  - `forceAdaptiveThinking: true`
  - `supportsStrictTools: true`
- Models, copied from the credential-free portions of the local config:
  - `claude-fable-5`
  - `claude-opus-5`
  - `claude-sonnet-5`

### OpenAI

- Provider API: `openai-responses`
- Provider compatibility:
  - `supportsDeveloperRole: true`
  - `supportsStrictMode: true`
- Models:
  - `gpt-5.6-sol`
  - `gpt-5.6-terra`
  - `gpt-5.6-luna`

### DeepSeek

- Provider API: `openai-responses`
- Provider compatibility:
  - `supportsDeveloperRole: false`
  - `supportsLongCacheRetention: false`
  - `supportsStrictMode: true`
- Models:
  - `deepseek-v4-flash`

The final design must preserve the exact model names, reasoning flags, input modalities, context windows, output limits, costs, cost tiers, and thinking-level maps from the current local templates.

## Requirements

- **R1 Interactive flow:** select a family, select one or more models in that family, input provider identifier, input base URL, securely input API key, preview a redacted diff, then explicitly confirm.
- **R2 Minimal user knowledge:** the user chooses from model display names but supplies no raw model ID, API mode, compatibility field, cost, token, or thinking configuration.
- **R3 Fixed mapping:** Anthropic always uses `anthropic-messages`; OpenAI and DeepSeek always use `openai-responses` for this MVP.
- **R4 Per-family model selection:** selecting a family opens an initially empty multi-select list of its predefined models. At least one model is required, and only selected models are written in the fixed catalog order.
- **R5 Credential handling:** never display the API key in previews, notifications, logs, command arguments, test fixtures, or error messages. A newly created file must use mode `0600`; an existing owner-only mode is preserved, while broader modes block the operation.
- **R6 Non-destructive upsert:** modify only `providers[providerId]`. Preserve every sibling provider and top-level field. If the provider identifier already exists with different content, show that collision and require a second explicit replacement confirmation.
- **R7 Safe write:** invalid/non-object JSON aborts without writing; an existing-file write creates a permission-preserving backup and uses atomic rename; declining either confirmation writes nothing.
- **R8 Validation:** reject an empty model selection, empty or reserved provider identifiers, identifiers outside the approved character/length pattern, malformed/unsupported base URLs, and empty API keys before preview.
- **R9 Hot reload:** after success, instruct the user to open `/model`; no Pi restart should be required.
- **R10 Mode behavior:** the credential-entry path must not fall back to an unmasked prompt. Unsupported non-TUI modes must report that the wizard requires interactive TUI mode and perform no write.
- **R11 Public preset safety:** templates contain no real provider URL or credential, and the secret scanner is refined so schema/property names remain legal while credential-like values remain blocked.
- **R12 Existing provider replacement:** if `providers[providerId]` differs, show a redacted diff and require a second explicit confirmation before replacing that provider object as a unit. Never field-merge stale models or compatibility settings into the selected template.
- **R13 Command boundary:** register the feature as the dedicated `/preset-models-add` command; do not add credential entry or `models.json` writes to `/preset-sync`.
- **R14 Pi-compatible parsing:** accept Pi's supported `//` comments and trailing commas on read. Preserve all top-level and sibling-provider data semantically; a successful write may normalize formatting and remove comments, with the original bytes retained in `.preset-bak`.
- **R15 Permission gate:** create a missing file as `0600`; preserve an existing mode only when it grants no group/other permissions. If an existing file is broader than `0600`, abort before preview/write with an actionable `chmod 600` instruction.
- **R16 Write-time conflict check:** re-read immediately before applying. If the selected provider changed after preview, abort and require a new run; preserve unrelated concurrent sibling changes.
- **R17 Delivery order:** implementation begins only after the `integrate-grill-me` child has completed its local and preset validation.

## Acceptance Criteria

- **AC1** Anthropic sandbox runs produce `api: "anthropic-messages"`, the fixed Anthropic compatibility object, and exactly the selected non-empty subset of `claude-fable-5`, `claude-opus-5`, and `claude-sonnet-5` with current local metadata.
- **AC2** OpenAI sandbox runs produce `api: "openai-responses"`, the fixed OpenAI compatibility object, and exactly the selected non-empty subset of `gpt-5.6-sol`, `gpt-5.6-terra`, and `gpt-5.6-luna` with current local metadata.
- **AC3** DeepSeek sandbox run produces `api: "openai-responses"`, the fixed DeepSeek compatibility object, and exactly the selected `deepseek-v4-flash` template.
- **AC4** A seeded file containing unrelated top-level keys and sibling providers retains them semantically unchanged after an add.
- **AC5** Cancel at family selection, any input, redacted preview confirmation, or collision replacement confirmation leaves the target and its mtime unchanged.
- **AC6** Invalid JSON, an invalid base URL, or an empty API key produces an actionable error and no backup/tmp/output mutation.
- **AC7** The API key is absent from captured TUI output, notifications, test output, and preset git history; the resulting file contains it with mode `0600` (or the original stricter mode).
- **AC8** Re-running with the exact same provider object reports already configured and performs no write.
- **AC9** After writing, `/model` or `pi --list-models` exposes every selected model and no unselected family model under the provider identifier.
- **AC10** Existing `/preset-sync` behavior remains unchanged, and shared-writer regression tests cover both old and new callers.
- **AC11** A valid commented/trailing-comma `models.json` is accepted; after writing, its unrelated data remains semantically equal and its original bytes are available in `.preset-bak`.
- **AC12** A file with group/other permission bits, a dangling symlink, or a target-provider TOCTOU conflict produces no provider write.
- **AC13** Every family checklist initially has zero selected models; attempting to continue without selecting one is blocked without advancing or writing.

## Out of Scope

- Arbitrary/custom model entry or editing predefined model metadata.
- Google, OpenRouter, Azure, Bedrock, or other provider families.
- `/login`, `auth.json`, OAuth, environment-variable generation, or secret-manager integration.
- Remote endpoint health checks or paid inference requests.
- Changing the active/default provider or model automatically.

## Key Decision

- **D1 Collision behavior:** support reconfiguration of an existing provider through redacted diff + second confirmation + backup. All sibling providers remain untouched.
- **D2 Secret storage:** store the supplied literal key in the selected provider object, matching the requested three-input flow and the existing local configuration. Environment-variable and secret-manager flows remain out of scope.
- **D3 Interaction mode:** API-key entry is available only in interactive TUI mode because Pi's ordinary input and RPC input protocols are not masked.
- **D4 Selection semantics:** model membership is selected per run; provider API and compatibility remain family-level fixed values, while each selected model uses its complete immutable template.
- **D5 Initial selection:** every family checklist starts with no models selected so adding any model is always an explicit user action.
