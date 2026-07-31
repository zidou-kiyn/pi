# Protocol Guidelines

Covers `@earendil-works/pi-protocol`: the transport-neutral CBOR protocol for
remote pi sessions in `packages/protocol/src/` — `cbor/` (`encoder.ts`,
`decoder.ts`, `options.ts`, `index.ts`), `framing.ts`, `schemas.ts`,
`codec.ts`, and the barrel `index.ts`.

The package is deliberately transport-free: it produces and consumes
`Uint8Array`, and never opens a socket, pipe, or HTTP connection. Anything
that binds these bytes to a transport belongs in the caller, not here.

## No consumers yet

Nothing imports this package. `tsconfig.json:19-20` maps
`@earendil-works/pi-protocol` and `@earendil-works/pi-protocol/*` to the source
tree, and the root `package.json` `build` / `build:offline` chains build it
between `storage/sqlite-node` and `coding-agent`, but a repo-wide grep for the
specifier outside `packages/protocol` returns only those two `tsconfig`
lines. Do not go looking for a server or CLI integration to mirror — there is
none. The corollary: this package's tests are the *only* thing pinning its
behavior, so a change without a test change is unverified.

## Pre-Development Checklist

1. Read [`../../_shared/typescript-and-style.md`](../../_shared/typescript-and-style.md)
   and [`../../_shared/testing.md`](../../_shared/testing.md) first.
2. Decide which side of the split you are on and read that file:
   [`wire-format.md`](./wire-format.md) for bytes (CBOR subset, limits, length
   prefix, streaming decoder), [`messages-and-codec.md`](./messages-and-codec.md)
   for the message contract (schemas, validation, envelopes).
3. Treat every limit as a security boundary, not a tuning knob. `options.ts`
   and `framing.ts` exist to bound attacker-controlled input; a change that
   raises or removes a default needs an explicit justification.
4. If you touch `PROTOCOL_VERSION` (`src/schemas.ts:3`), you are making a wire
   compatibility change. `ServerHelloSchema` pins `version` to that literal, so
   old and new peers stop handshaking. Say so in the PR.
5. If you touch `SessionPhaseSchema` (`src/schemas.ts:38`), update
   `AgentHarnessPhase` in `packages/agent` in the same change — the source
   comment states the two vocabularies are deliberately identical.
6. This package depends only on `typebox` (`package.json` `dependencies`). It
   has no `@earendil-works/*` dependency and must keep it that way; importing
   `pi-agent-core` here would make a transport-neutral protocol depend on the
   runtime it describes.
7. Run the tests by explicit file from the package root, per
   [`../../_shared/testing.md`](../../_shared/testing.md):
   ```bash
   cd packages/protocol
   node ../../node_modules/vitest/dist/cli.js --run test/protocol.test.ts
   node ../../node_modules/vitest/dist/cli.js --run test/framing.test.ts
   node ../../node_modules/vitest/dist/cli.js --run test/cbor/cbor.test.ts
   npm run check                                  # from the repo root
   ```

## Guidelines Index

| File | Covers |
|---|---|
| [Wire Format](./wire-format.md) | The strict CBOR subset and why each construct is rejected, the resource limits, the 4-byte length prefix, `FrameDecoder` streaming state and its terminal failure |
| [Messages And Codec](./messages-and-codec.md) | `PROTOCOL_VERSION`, strict typebox schemas, client/server envelopes, the two-stage validation in `codec.ts`, decoder poisoning, bounded error text |

## Shared Rules

- Repo-wide TypeScript, testing, check, and dependency conventions:
  [`../../_shared/index.md`](../../_shared/index.md)
- Generic thinking guides (cross-layer, code reuse):
  [`../../guides/index.md`](../../guides/index.md)

`packages/protocol/README.md` documents the same surface for external
consumers. It is prose maintained by hand; when behavior changes, update it in
the same commit as the code.
