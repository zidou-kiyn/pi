# Types And Compat

## When This Applies

- Adding a new `Api` value, a new `KnownProvider`, or a new field on `Model<TApi>`.
- Adding or changing a provider-compatibility flag (`OpenAICompletionsCompat`,
  `OpenAIResponsesCompat`, `AnthropicMessagesCompat`, `BedrockCompat`).
- Writing code that narrows a dynamically-looked-up `Model<Api>` to a concrete API.

## The Local Pattern

### Open unions, not closed enums

`Api` is `KnownApi | (string & {})` (`packages/ai/src/types.ts:28`) — a closed literal union
(`KnownApi`) unioned with an open `string`. This lets `Model<TApi>` type-check exhaustively for
built-in APIs while still accepting a custom API id from a user-supplied `createProvider()`
call without a cast. `KnownProvider`/`ProviderId` (`types.ts:34-73`) and `KnownImagesApi`/
`ImagesApi` (`types.ts:30-32`) follow the same shape.

### `ApiOptionsMap` is the single source of per-API option typing

```ts
export interface ApiOptionsMap {
	"anthropic-messages": AnthropicOptions;
	"openai-completions": OpenAICompletionsOptions;
	// ...
}
export type ApiStreamOptions<TApi extends Api> = TApi extends keyof ApiOptionsMap
	? ApiOptionsMap[TApi]
	: StreamOptions & Record<string, unknown>;
```

(`types.ts:207-226`). Every entry is a **type-only** import from the matching `src/api/<id>.ts`
module (`types.ts:1-12`); the doc comment on `ApiOptionsMap` states this is erased at emit and
therefore tree-shake safe — importing the type does not pull the implementation module into a
bundle. Adding a new API means adding both the `KnownApi` literal and an `ApiOptionsMap` entry
together, or `Provider.stream()` silently falls back to the untyped `StreamOptions &
Record<string, unknown>` shape for that API.

### `hasApi()` narrows a runtime `Model<Api>`

`models.getModel()` returns `Model<Api>` (the open union) because a `Models` collection mixes
providers. `hasApi(model, "anthropic-messages")` (`packages/ai/src/models.ts:635`) is a type
guard that narrows to `Model<"anthropic-messages">`, unlocking `ApiStreamOptions<"anthropic-
messages">` on the next `stream()`/`complete()` call. This is the sanctioned way to get typed
options from a dynamic lookup instead of casting.

### `compat` is a conditional type keyed off `TApi`

```ts
compat?: TApi extends "openai-completions"
	? OpenAICompletionsCompat
	: TApi extends "openai-responses" | "azure-openai-responses" | "openai-codex-responses"
		? OpenAIResponsesCompat
		: TApi extends "anthropic-messages"
			? AnthropicMessagesCompat
			: TApi extends "bedrock-converse-stream"
				? BedrockCompat
				: never;
```

(`types.ts:779-788`). Compat flags override auto-detection: most `OpenAICompletionsCompat`
fields default to "auto-detected from baseUrl" per their doc comments (e.g.
`supportsDeveloperRole`), and the generated catalog sets fields explicitly where
auto-detection is not enough — `packages/ai/test/providers.test.ts` asserts
`getBuiltinModel("openai", "gpt-5.4").compat` includes `supportsOpenAIGrammarTools: true`
while `getBuiltinModel("openai", "gpt-4o").compat?.supportsOpenAIGrammarTools` is `undefined`.
A custom `createProvider()` model sets `compat` directly, since it has no baseUrl-based
auto-detection entry in the generated catalog.

### A compat flag exists so a provider quirk never becomes a code branch

`supportsFinishReason` (`types.ts:529`) is the current example of the shape a
new flag must take. Default `true` in `detectCompat`
(`api/openai-completions.ts:1451`), merged with the model's override in
`getCompat` (`:1501`), and read at exactly two places in the stream tail
(`:574`, `:580`): when it is `false` and the stream ended without a
`finish_reason`, `stopReason` is inferred as `toolUse` if any content block is
a tool call and `stop` otherwise, instead of throwing
`"Stream ended without finish_reason"`. `openai-completions-tool-choice.test.ts`
asserts both directions — "errors when a stream ends after only null
finish_reason chunks" for the default, "accepts streams without finish_reason
when compat disables it" for the override. A quirk handled by sniffing the
baseUrl inside the stream loop instead of a named compat field is not
overridable by a custom provider and is rejected on that basis.

### Provider failures carry structured diagnostics, and `errorMessage` stays byte-identical

`AssistantMessage.diagnostics` (`types.ts:407`) is an append-only list written
only through `appendAssistantMessageDiagnostic`
(`utils/diagnostics.ts:40`). Bedrock is the reference implementation:
`appendBedrockFailureDiagnostic` (`api/bedrock-converse-stream.ts:403`) emits a
`bedrock_response_failure` entry whose `details` may contain `status`,
`errorCode`, and `requestId`.

Three rules are encoded there and apply to any provider that adds diagnostics:

- **Never touch `errorMessage`.** `isRetryableAssistantError` matches against
  that string, so structured metadata is added *alongside* it, never folded
  into it.
- **Omit unknown fields; never guess.** A modeled mid-stream exception arrives
  as a bare object with no HTTP metadata, so `responseRequestId` is captured
  outside the `try` (`api/bedrock-converse-stream.ts:225`) purely so the catch
  can still correlate the request. If nothing is known, no diagnostic is
  emitted at all.
- **Drop over-long values instead of truncating.**
  `normalizeDiagnosticValue` (`:381`) rejects anything over 200 characters — a
  truncated request id is not a request id.

`extractBedrockErrorCode` (`:393`) reads `error.name` and requires an
`Exception` suffix, because the SDK puts the modeled code there for both
service exceptions and unmodeled stream errors, while transport failures use
names like `TimeoutError`. `packages/ai/test/bedrock-error-metadata.test.ts` is
the contract.

### Chat and image types are deliberately separate

`ImagesModel<TApi>` (`types.ts:790-795`) is `Omit<Model<Api>, "api" | "provider" | "reasoning"
| "contextWindow" | "maxTokens" | "compat">` plus its own `api`/`provider`/`output`.
`ImagesApi`/`ImagesProviderId` do not overlap with `Api`/`ProviderId`. `ImagesOptions`
(`types.ts:255-301`) also duplicates most of `StreamOptions`' fields one by one rather than
extending it — there is no shared base interface between chat and image request options today.

## Reference Files

- `packages/ai/src/types.ts` — `Api`, `KnownProvider`, `Model<TApi>`, `ApiOptionsMap`,
  `*Compat` interfaces, `ImagesModel`
- `packages/ai/src/models.ts` — `hasApi()`, `ApiStreamOptions<T>` usage inside
  `Provider.stream()`
- `packages/ai/src/utils/diagnostics.ts` — `AssistantMessageDiagnostic`,
  `appendAssistantMessageDiagnostic`
- `packages/ai/test/providers.test.ts` — asserts generated `compat` values on built-in models
- `packages/ai/test/bedrock-error-metadata.test.ts` — structured failure metadata contract
- `packages/ai/test/openai-completions-tool-choice.test.ts` — `supportsFinishReason` both directions
- `packages/ai/test/model-catalog-types.test.ts` — type-level catalog assertions

## Anti-Patterns

| Anti-pattern | Why | Enforcement |
|---|---|---|
| Casting `Model<Api>` to a concrete `Model<"...">` | Bypasses the `hasApi()` guard; can desync from the actual `model.api` value at runtime | Convention; reviewers reject |
| Adding a `KnownApi` literal without an `ApiOptionsMap` entry | Silently falls back to the untyped option shape for that API everywhere `ApiStreamOptions` is used | Convention only — not machine-checked |
| Extending `ImagesOptions` from `StreamOptions` | Not the current pattern; the two option shapes are hand-duplicated on purpose as separate surfaces | Convention; reviewers reject |
