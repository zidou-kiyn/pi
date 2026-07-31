# Messages And Codec

## When This Applies

Any change to `packages/protocol/src/schemas.ts` (message shapes, protocol
version, phase vocabulary) or `packages/protocol/src/codec.ts` (validation,
encode/decode entry points).

## The Local Pattern

### `PROTOCOL_VERSION` is a wire contract, not a package version

`PROTOCOL_VERSION = 2 as const` (`schemas.ts:3`) is independent of the npm
version. `ServerHelloSchema` pins `version: Type.Literal(PROTOCOL_VERSION)`
(`schemas.ts:376`), and `isSupportedProtocolVersion` (`codec.ts:170`) is the
only accepted check — it requires an exact integer match, with no range or
"greater than" tolerance. `ClientHelloSchema` (`schemas.ts:346`) accepts any
`Type.Integer({ minimum: 0 })` so the server can reject a mismatch with a
`hello_error` instead of failing to parse the frame; its comment spells out
that the version is "intentionally an integer, not a coercible string".

Bumping the constant breaks every peer at the handshake. Do it deliberately
and say so in the commit message.

### Every object schema is closed

`StrictObject` (`schemas.ts:7`) wraps `Type.Object(properties, {
additionalProperties: false })` and is used for *every* object in the file.
An unknown field is a validation failure, not ignored data, which is what makes
forward-compatible field additions a version-level decision rather than a
silent one. Building a message schema with a bare `Type.Object` is a bug even
when it type-checks.

Shared leaf schemas keep their constraints in one place: `IdSchema`
(`schemas.ts:5`) is `Type.String({ minLength: 1 })`, `TimestampSchema`
(`schemas.ts:6`) is `Type.Integer({ minimum: 0 })`. Free-form payloads use
`JsonValueSchema` (`schemas.ts:24`), a `Type.Cyclic` recursive union wrapped in
`Type.Unsafe<JsonValue>` so the emitted TypeScript type stays the hand-written
`JsonValue` (`schemas.ts:10`) rather than an inferred recursive mess.

### `SessionPhaseSchema` is a deliberate duplicate of `AgentHarnessPhase`

`schemas.ts:38` carries the comment "Matches AgentHarnessPhase so adapters do
not need a second phase vocabulary", and lists `idle`, `turn`, `compaction`,
`branch_summary`, `retry`. This package intentionally does *not* import
`pi-agent-core` to get it — a transport-neutral protocol must not depend on the
runtime it describes. The cost of that choice is a manual invariant: changing
the phase union in either place without the other splits the vocabulary, and
nothing in CI catches it.

### The message tree is two closed unions

| Direction | Union | Members |
|---|---|---|
| Client → server | `ClientMessageSchema` (`schemas.ts:359`) | `hello`, `request` (`RequestEnvelopeSchema`, an `id` plus one `Command`) |
| Server → client | `ServerMessageSchema` (`schemas.ts:402`) | `hello`, `hello_error`, `response`, `event` |

`CommandSchema` (`schemas.ts:275`) is the discriminated union of client
commands (`create`, `attach`, `prompt`, `steer`, `abort`, `set_model`,
`set_thinking`, `list`, `detach`), and `CommandResultSchema` (`schemas.ts:326`)
mirrors it one-for-one on the response side. `ResultForCommand`
(`schemas.ts:339`) is the type-level mapping between them. `ResponseEnvelopeSchema`
(`schemas.ts:384`) is a union of an `ok: true` shape carrying `result` and an
`ok: false` shape carrying `error`, never one object with both optional — a
response cannot be simultaneously successful and failed.

Adding a command means touching all four: `CommandSchema`, its result schema,
`CommandResultSchema`, and `ResultForCommand`.

### Validation is two stages, and the first one is not typebox

`parseClientMessage` / `parseServerMessage` (`codec.ts:41`, `:48`) run
`isProtocolValue(value)` (`codec.ts:25`) *before* typebox's `Check`. That guard
walks the value and rejects anything that is not a primitive, a plain array, or
an object whose prototype is exactly `Object.prototype`, and it tracks
`ancestors` to reject cycles. Reason: typebox validates *shape*, so a class
instance or a prototype-polluted object with the right fields would pass while
carrying behavior the peer never sent. `undefined` is allowed only in an
optional property position (`optionalProperty`), never as an array element.

Both stages throw the same `ProtocolValidationError` (`codec.ts:18`) with a
deliberately generic message — no offending value is echoed back.

### Encode validates before it serializes; decode validates after

`encodeProtocolMessage` (`codec.ts:60`) parses first, then encodes CBOR and
frames it, then runs `assertCompleteFrame` on its own output. An invalid
message therefore never reaches the wire, and a `ProtocolValidationError` from
the parse stage is rethrown unchanged rather than being wrapped as an encoding
failure.

`ClientMessageDecoder` / `ServerMessageDecoder` (`codec.ts:129`, `:146`) are
thin wrappers over `ValidatedMessageDecoder` (`codec.ts:88`), which chains
`FrameDecoder` → `decodeCbor` → `parse`. Both directions pass `maxFrameLength`
down as the CBOR `maxByteLength`, so one limit bounds framing and payload
together.

Failure is terminal here too: any throw sets `failed = true` (`codec.ts:111`, `:122`)
and every later `push()`/`end()` throws immediately. A peer that sends one
invalid frame gets no second chance on that connection.

`boundedErrorMessage` (`codec.ts:55`) caps any wrapped error text at 500
characters (497 plus an ellipsis) so a hostile payload cannot inflate a log
line through the error path.

## Reference Files

- `packages/protocol/src/schemas.ts` — `PROTOCOL_VERSION`, `StrictObject`,
  command/result unions, client and server envelopes
- `packages/protocol/src/codec.ts` — `isProtocolValue`, parse/encode/decode,
  `ProtocolValidationError`, decoder poisoning
- `packages/protocol/test/protocol.test.ts` — round-trip and rejection cases
- `packages/agent/src/harness/types.ts` — `AgentHarnessPhase`, the union
  `SessionPhaseSchema` must stay identical to

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| `Type.Object(...)` without `additionalProperties: false` | Unknown fields silently pass, turning a schema change into an undetectable compatibility drift | `schemas.ts:7` — every schema goes through `StrictObject` |
| Accepting `version >= PROTOCOL_VERSION` in a handshake | The only supported check is exact equality; a newer peer is not a compatible peer | `codec.ts:170` |
| Changing `SessionPhaseSchema` without `AgentHarnessPhase` | Two phase vocabularies with no CI guard between them | `schemas.ts:38` comment |
| Dropping `isProtocolValue` because typebox already validates | typebox checks shape, not prototype or cycles; a class instance would pass | `codec.ts:25` |
| Including the offending value or a raw provider error in a `ProtocolValidationError` | Leaks peer-controlled data into logs; messages are generic and length-bounded by design | `codec.ts:18`, `:55` |
| Continuing to use a message decoder after it threw | Poisoning is intentional — the connection is no longer trustworthy | `codec.ts:111` |
| Adding a command schema without its result schema and `ResultForCommand` arm | The request compiles but has no typed response | `schemas.ts:275`, `:326`, `:339` |
