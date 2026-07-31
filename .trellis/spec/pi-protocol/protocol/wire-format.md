# Wire Format

## When This Applies

Any change to `packages/protocol/src/cbor/` (`encoder.ts`, `decoder.ts`,
`options.ts`) or `packages/protocol/src/framing.ts` — that is, anything that
changes the bytes on the wire or the limits applied to them.

## The Local Pattern

### The CBOR subset is intentionally narrow, and every exclusion is a rejection

`encodeCbor` (`cbor/encoder.ts:211`) emits "the protocol's strict,
definite-length RFC 8949 subset" and `decodeCbor` (`cbor/decoder.ts:161`)
accepts exactly that subset. This is not an incomplete implementation to be
finished later; each gap is a hardening decision with an explicit throw:

| Construct | Behavior | Where |
|---|---|---|
| Tags (major type 6) | `CborError("CBOR tags are not supported")` | `cbor/decoder.ts:80` |
| Indefinite-length strings/arrays/maps | rejected via `readLength` | `cbor/decoder.ts:113`, `:139` |
| Break marker (`0xff`) | rejected | `cbor/decoder.ts:106` |
| Half/single-precision floats, other simple values | rejected; only `false`/`true`/`null`/float64 decode | `cbor/decoder.ts:88-109` |
| Non-string map keys | `CborError("CBOR map keys must be strings")` | `cbor/decoder.ts:67`, `cbor/encoder.ts:183` |
| Duplicate map keys | rejected | `cbor/decoder.ts:68` |
| Trailing bytes after one item | `CborError("CBOR payload contains trailing data")` | `cbor/decoder.ts:22` |
| `NaN` / `Infinity` | rejected on both sides | `cbor/encoder.ts:138`, `cbor/decoder.ts:99` |
| Integers outside the safe range | rejected on both sides | `cbor/encoder.ts:140`, `cbor/decoder.ts:39`, `:101` |
| Cycles | `CborError("CBOR values must not contain cycles")` | `cbor/encoder.ts:161`, `:180` |
| Array holes, `undefined` array elements | rejected | `cbor/encoder.ts:170` |
| Any non-plain-object value (class instance, `Map`, `Date`) | `CborError("Unsupported CBOR value type")` | `cbor/encoder.ts:207` |
| Lone surrogates in text | rejected by a round-trip check | `cbor/encoder.ts:114` |

Two asymmetries are deliberate:

- **`undefined` object properties are dropped, not rejected**
  (`cbor/encoder.ts:189`), so an optional field left unset encodes as absent
  rather than failing. The same value inside an *array* is an error, because
  a hole would silently change the array's shape.
- **Decoded maps are built with `Object.defineProperty`**
  (`cbor/decoder.ts:70`) rather than assignment, so a `__proto__` key becomes an
  own data property instead of mutating the prototype chain. Do not "simplify"
  that to `result[key] = ...`.

Adding a CBOR construct means adding it to both files plus
`test/cbor/cbor.test.ts`; a decoder that accepts what the encoder cannot
produce is a protocol asymmetry that peers will disagree about.

### Limits live in one place and are resolved, never read raw

`cbor/options.ts` owns every bound: `DEFAULT_MAX_CBOR_BYTE_LENGTH` (16 MiB),
`DEFAULT_MAX_CBOR_CONTAINER_LENGTH` (1,000,000 entries),
`DEFAULT_MAX_CBOR_DEPTH` (64), and the hard ceiling `MAX_CONFIGURED_DEPTH`
(512). Callers pass `CborOptions`; both the encoder and the decoder start by
calling `resolveOptions` (`cbor/options.ts:42`), which validates each value
through `resolveLimit` and throws `RangeError` for a non-integer, negative, or
over-ceiling limit. The header comment states the intent: "Safe defaults for
untrusted protocol payloads."

The depth limit is enforced on both sides (`cbor/encoder.ts:126`,
`cbor/decoder.ts:27`), so a deeply nested object cannot be encoded locally and
then blow the stack remotely.

### Framing is a 4-byte big-endian length prefix, nothing more

`encodeFrame` (`framing.ts:28`) writes `payload.byteLength` as unsigned 32-bit
big-endian followed by the payload. There is no type byte, checksum, or
version field on the frame — versioning lives in the message body
(see [`messages-and-codec.md`](./messages-and-codec.md)).

`assertCompleteFrame` (`framing.ts:42`) is the one-shot validator: it requires
a full header, a length within `maxFrameLength`
(`DEFAULT_MAX_FRAME_LENGTH` = 16 MiB, `framing.ts:6`), and *exactly*
`4 + length` bytes — a frame with trailing bytes is invalid, not tolerated.

### `FrameDecoder` is a state machine that fails permanently

`FrameDecoder` (`framing.ts:58`) accumulates arbitrary chunk boundaries:
`push()` fills a 4-byte header across calls, then copies the payload into
`PAYLOAD_BLOCK_SIZE` (64 KiB) blocks and concatenates once complete — a
64 KiB-block strategy so a declared-but-never-delivered 16 MiB frame does not
force one giant allocation up front. Zero-length frames are emitted directly
without entering the payload path.

Three rules govern its lifecycle:

- The length limit is checked **before** any payload is buffered
  (`framing.ts:92`), so an oversized declaration is rejected without allocating
  for it.
- `fail()` (`framing.ts:155`) moves the decoder to a terminal `"failed"` state,
  clears all buffers, and throws. Every later `push()` or `end()` throws
  `FrameError` again. A decoder is never recovered after a protocol error;
  the caller must drop the connection.
- `end()` (`framing.ts:146`) fails on a partial header or an incomplete
  payload — "Truncated frame at end of stream". A clean end after a complete
  frame moves it to `"ended"`, which is also terminal.

`test/framing.test.ts` covers split headers, split payloads, zero-length
frames, oversize rejection, and the terminal states. A new framing behavior
without a case there is unverified.

## Reference Files

- `packages/protocol/src/cbor/options.ts` — limits, `resolveOptions`,
  `CborError`, shared `TextEncoder`/`TextDecoder` (`fatal: true`)
- `packages/protocol/src/cbor/encoder.ts` — `CborWriter`, `encodeCbor`
- `packages/protocol/src/cbor/decoder.ts` — `CborReader`, `decodeCbor`
- `packages/protocol/src/framing.ts` — `encodeFrame`, `assertCompleteFrame`,
  `FrameDecoder`, `FrameError`
- `packages/protocol/test/cbor/cbor.test.ts`,
  `packages/protocol/test/framing.test.ts`

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| Accepting a CBOR construct in the decoder without emitting it from the encoder | Peers disagree about what is legal; the subset stops being a contract | `cbor/encoder.ts:211` / `cbor/decoder.ts:161` are two halves of one grammar |
| Assigning decoded map entries with `result[key] = value` | Reintroduces prototype pollution through a `__proto__` key | `cbor/decoder.ts:70` uses `Object.defineProperty` |
| Reading a limit from `CborOptions` directly instead of `resolveOptions` | Skips the integer/range validation and lets a caller pass `Infinity` | `cbor/options.ts:42` |
| Raising a default limit to make a large payload work | The defaults bound untrusted input; the caller should pass an explicit option for its own trusted case | `cbor/options.ts` "Safe defaults for untrusted protocol payloads" |
| Reusing a `FrameDecoder` after it threw | Failure is terminal by design; the buffers are already cleared | `framing.ts:155` |
| Buffering a frame before checking its declared length | Lets a 4-byte header trigger an arbitrary allocation | `framing.ts:92` |
