# Design: interactive models configuration

## Delivery boundary and dependency

Product changes live in `/home/heixiaohu/pi-preset`. No Pi core change is required. Implementation starts only after the `integrate-grill-me` child is completed and archived.

The feature is a dedicated `/preset-models-add` command. It does not participate in `/preset-sync` and does not register providers dynamically; it writes Pi's native `models.json` source of truth and tells the user to open `/model` for hot reload.

## Module boundary

```text
extensions/preset-models-add.ts   mode guard and interaction orchestration
src/model-templates.ts            three credential-free fixed family templates
src/models-config.ts              parse, validate, plan, redact, semantic diff
src/models-config-apply.ts        write-time re-read, conflict check, provider upsert
src/models-wizard-ui.ts           masked input and scrollable redacted diff confirmation
src/json-merge.ts                 extended atomic writer options
src/paths.ts                      getModelsPath()
```

Pure config modules use only Node APIs so their tests run without installing Pi peer packages. The extension/UI adapter owns imports from `@earendil-works/pi-coding-agent` and `@earendil-works/pi-tui`.

## Interaction flow

```text
mode guard (TUI only)
  -> select Anthropic/OpenAI/DeepSeek
  -> multi-select one or more family models
  -> input provider identifier
  -> input base URL
  -> read/validate models.json and permission/symlink state
  -> masked API-key input
  -> build candidate and semantic plan
  -> exact match? report no-op
  -> scrollable redacted diff confirmation
  -> existing different provider? second short replacement confirmation
  -> queued apply with write-time re-read
  -> notify: open /model, no restart
```

Any cancellation returns before the writer is reachable. Invalid file/permission state is checked before collecting the API key.

## Template contract

All templates omit provider identifier, base URL, and API key.

### Anthropic

```text
api: anthropic-messages
compat:
  supportsEagerToolInputStreaming: false
  supportsLongCacheRetention: true
  forceAdaptiveThinking: true
  supportsStrictTools: true
```

| Model | Input | Context | Max output | Input / output / cache read / cache write |
|---|---|---:|---:|---|
| `claude-fable-5` | text,image | 1,000,000 | 128,000 | 10 / 50 / 1 / 12.5 |
| `claude-opus-5` | text,image | 1,000,000 | 128,000 | 5 / 25 / 0.5 / 6.25 |
| `claude-sonnet-5` | text,image | 1,000,000 | 128,000 | 2 / 10 / 0.2 / 2.5 |

All three are reasoning models with `off`/`minimal` disabled and `low` through `max` mapped to same-name values.

### OpenAI

```text
api: openai-responses
compat:
  supportsDeveloperRole: true
  supportsStrictMode: true
```

| Model | Input | Context | Max output | Input / output / cache read / cache write |
|---|---|---:|---:|---|
| `gpt-5.6-sol` | text,image | 372,000 | 128,000 | 5 / 30 / 0.5 / 6.25 |
| `gpt-5.6-terra` | text,image | 272,000 | 128,000 | 2 / 12 / 0.2 / 2.5 |
| `gpt-5.6-luna` | text,image | 272,000 | 128,000 | 0.2 / 1.2 / 0.02 / 0.25 |

All three are reasoning models. Their thinking map uses `off: "none"`, disables `minimal`, and maps `low` through `max` to same-name values.

Long-context tiers above 272,000 input tokens are:

- Sol: 10 / 45 / 1 / 12.5
- Terra: 4 / 18 / 0.4 / 5
- Luna: 0.4 / 1.8 / 0.04 / 0.5

### DeepSeek

```text
api: openai-responses
compat:
  supportsDeveloperRole: false
  supportsLongCacheRetention: false
  supportsStrictMode: true
```

`deepseek-v4-flash` is reasoning-enabled, text-only, context 1,000,000, max output 384,000, cost 0.14 / 0.28 / 0.0028 / 0. Its thinking map is `off: "none"`, `low: "low"`, `high: "high"`, `max: "max"`, with the other levels unsupported.

## Input validation

- Provider identifier: `^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$`; reject `__proto__`, `prototype`, and `constructor`.
- Base URL: `new URL()` must yield `http:` or `https:`, a hostname, no username/password, no query, and no fragment. Store the normalized URL string.
- API key: reject empty or whitespace-only input without including it in the error.
- Model selection: require at least one ID from the selected family's fixed catalog; preserve catalog order regardless of toggle order.

These constraints keep provider/model references unambiguous and prevent credentials from being embedded in the previewed URL.

## Model multi-select

`models-wizard-ui.ts` adds a checklist component before credential inputs. It renders one row per family model plus a final `Continue` row.

- `tui.select.up/down` moves the cursor.
- `tui.select.confirm` toggles a model row or submits the `Continue` row.
- `tui.select.cancel` cancels the wizard.
- All model rows start unchecked.
- The `Continue` row is disabled while the selection is empty.
- The returned IDs are normalized to template catalog order.

Using a final action row avoids introducing a hardcoded Space-key action or a new global keybinding.

## Masked input

Pi's ordinary input displays the real value. `models-wizard-ui.ts` wraps a TUI `Input` as an editing engine but never calls its `render()` method.

The component:

- implements `Component` and `Focusable`;
- delegates non-confirm/cancel input to `Input.handleInput()`;
- renders an empty state or one fixed mask such as `••••••••`, independent of secret length;
- places `CURSOR_MARKER` after the fixed mask;
- clears the inner `Input` before resolving/cancelling;
- never stores the secret in rendered/cached strings.

RPC, JSON, and print modes never fall back to ordinary input. RPC receives an error notification; JSON/print diagnostics go to stderr so JSON stdout remains valid.

## Parsing and planning

`readModelsDocument()` mirrors Pi's `//` comment and trailing-comma stripping before `JSON.parse`.

- Missing or empty file starts as `{ providers: {} }`.
- Root must be a plain object.
- Missing `providers` is treated as an empty object.
- Present non-object `providers` aborts.
- Existing group/other permission bits or a dangling symlink abort before secret entry.

The candidate provider is:

```text
{ baseUrl, apiKey, api, compat, models: selectedTemplates }
```

Planning captures the complete existing target provider as the write-time baseline. Semantic deep equality yields an already-configured no-op with no backup, tmp file, or mtime change.

## Redaction and diff

Redaction recursively replaces:

- every `apiKey` value;
- every value under `headers`;
- credential-like keys such as `authorization`, `token`, `secret`, and `password`.

Existing and supplied secrets use distinct fixed markers (`<redacted-existing>` and `<redacted-supplied>`) so a key-only change remains visible. No marker includes secret length, hash, prefix, or suffix.

The provider-only diff is deterministic and dependency-free: serialize the redacted old/new objects and render removed lines with `-` and added lines with `+`. A custom component scrolls the long preview with injected `tui.select.*` keybindings and truncates each rendered line to terminal width.

## Apply and write transaction

The extension wraps apply in `withFileMutationQueue(modelsPath, ...)` so in-process extension/tool mutations serialize on the canonical file path.

Inside the queue:

1. re-read with the same parser and permission checks;
2. compare the latest target provider with the preview baseline;
3. abort on a target conflict;
4. preserve the latest root and sibling providers;
5. replace only `providers[providerId]` as one object;
6. write through an enhanced atomic writer.

Writer options for this command are:

```text
newFileMode: 0600
rejectDanglingSymlink: true
```

For a valid symlink, backup/tmp files are created beside the real target and the symlink inode remains. Existing owner-only modes are preserved. A broader existing mode is a blocker rather than silently storing a credential in a world/group-readable file.

The writer creates `.preset-bak` from the original bytes, writes a unique sibling tmp file, applies the selected mode, and renames it atomically. Successful JSONC writes normalize to strict formatted JSON; the backup retains comments and original formatting.

## Secret scanner change

The current scanner rejects the property name `"apiKey"`, which prevents legitimate schema/builder code. Remove that property-name-only pattern, retain high-confidence credential prefixes/private-host patterns, and add a value-shaped pattern for long literal values assigned to credential-like keys while allowing redacted markers, environment references, and command references.

## Rollback and recovery

- Cancel/validation/permission/conflict paths perform no write.
- A successful replacement has `<real-target>.preset-bak` with the original bytes and mode.
- A write failure cleans tmp files and leaves either the old target or the complete new target, never a truncated file.
- Recovery instructions identify the backup and recommend rerunning the wizard after resolving malformed JSON, permissions, dangling links, or target conflicts.
