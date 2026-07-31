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

(`types.ts:777-786`). Compat flags override auto-detection: most `OpenAICompletionsCompat`
fields default to "auto-detected from baseUrl" per their doc comments (e.g.
`supportsDeveloperRole`), and the generated catalog sets fields explicitly where
auto-detection is not enough — `packages/ai/test/providers.test.ts` asserts
`getBuiltinModel("openai", "gpt-5.4").compat` includes `supportsOpenAIGrammarTools: true`
while `getBuiltinModel("openai", "gpt-4o").compat?.supportsOpenAIGrammarTools` is `undefined`.
A custom `createProvider()` model sets `compat` directly, since it has no baseUrl-based
auto-detection entry in the generated catalog.

### Chat and image types are deliberately separate

`ImagesModel<TApi>` (`types.ts:788-793`) is `Omit<Model<Api>, "api" | "provider" | "reasoning"
| "contextWindow" | "maxTokens" | "compat">` plus its own `api`/`provider`/`output`.
`ImagesApi`/`ImagesProviderId` do not overlap with `Api`/`ProviderId`. `ImagesOptions`
(`types.ts:255-301`) also duplicates most of `StreamOptions`' fields one by one rather than
extending it — there is no shared base interface between chat and image request options today.

## Reference Files

- `packages/ai/src/types.ts` — `Api`, `KnownProvider`, `Model<TApi>`, `ApiOptionsMap`,
  `*Compat` interfaces, `ImagesModel`
- `packages/ai/src/models.ts` — `hasApi()`, `ApiStreamOptions<T>` usage inside
  `Provider.stream()`
- `packages/ai/test/providers.test.ts` — asserts generated `compat` values on built-in models
- `packages/ai/test/model-catalog-types.test.ts` — type-level catalog assertions

## Anti-Patterns

| Anti-pattern | Why | Enforcement |
|---|---|---|
| Casting `Model<Api>` to a concrete `Model<"...">` | Bypasses the `hasApi()` guard; can desync from the actual `model.api` value at runtime | Convention; reviewers reject |
| Adding a `KnownApi` literal without an `ApiOptionsMap` entry | Silently falls back to the untyped option shape for that API everywhere `ApiStreamOptions` is used | Convention only — not machine-checked |
| Extending `ImagesOptions` from `StreamOptions` | Not the current pattern; the two option shapes are hand-duplicated on purpose as separate surfaces | Convention; reviewers reject |
