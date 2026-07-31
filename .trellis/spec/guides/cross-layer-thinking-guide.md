# Cross-Layer Thinking Guide

> **Purpose**: Think through data flow across layers before implementing.

---

## The Problem

**Most bugs happen at layer boundaries**, not within layers.

Common cross-layer bugs:

- API returns format A, frontend expects format B
- Database stores X, service transforms to Y, but loses data
- Multiple layers implement the same logic differently

---

## Before Implementing Cross-Layer Features

### Step 1: Map the Data Flow

Draw out how data moves:

```
Source → Transform → Store → Retrieve → Transform → Display
```

For each arrow, ask:

- What format is the data in?
- What could go wrong?
- Who is responsible for validation?

### Step 2: Identify Boundaries

| Boundary                   | Common Issues                                  |
| -------------------------- | ---------------------------------------------- |
| Provider ↔ agent loop      | Stream event shape, stop reasons, usage fields |
| Agent loop ↔ session store | Persisted JSONL entry shape and versioning     |
| Session store ↔ mode       | Replay order, missing entry kinds              |
| Core ↔ TUI / RPC           | Serialization, event kinds a mode does not handle |
| Extension API ↔ core       | Public surface drift, silently ignored hooks   |

### Step 3: Define Contracts

For each boundary:

- What is the exact input format?
- What is the exact output format?
- What errors can occur?

---

## Common Cross-Layer Mistakes

### Mistake 1: Implicit Format Assumptions

**Bad**: Assuming date format without checking

**Good**: Explicit format conversion at boundaries

### Mistake 2: Scattered Validation

**Bad**: Validating the same thing in multiple layers

**Good**: Validate once at the entry point

### Mistake 3: Leaky Abstractions

**Bad**: Component knows about database schema

**Good**: Each layer only knows its neighbors

### Mistake 4: Every Consumer Parses The Same Payload

**Bad**: A command reads JSONL events and casts fields inline:

```typescript
const thread = (ev as { thread?: string }).thread;
const labels = (ev as { labels?: string[] }).labels;
```

This looks local, but it means every consumer owns a private version of the
event contract. The next field change will update one command and miss another.

**Good**: Decode once at the event boundary, then export typed projections:

```typescript
if (!isThreadEvent(ev)) return false;
return ev.thread === filter.thread;
```

**Rule**: For append-only logs, JSON streams, RPC payloads, or config files,
create one owner for:

- event / payload type definitions
- type guards and normalization from `unknown`
- metadata projections used by UI commands
- reducers that replay state from the source of truth

Rendering code may format fields, but it must not redefine the payload contract.

---

## Checklist for Cross-Layer Features

Before implementation:

- [ ] Mapped the complete data flow
- [ ] Identified all layer boundaries
- [ ] Defined format at each boundary
- [ ] Decided where validation happens

After implementation:

- [ ] Tested with edge cases (null, empty, invalid)
- [ ] Verified error handling at each boundary
- [ ] Checked data survives round-trip
- [ ] Checked that consumers import shared decoders / projections instead of
      casting payload fields locally
- [ ] Checked that derived state points back to the source event identifier
      (`seq`, `id`, `version`) instead of inventing a second cursor

---

## Boundaries That Actually Exist In This Repo

| Boundary | Owner of the contract | Failure mode when bypassed |
|---|---|---|
| Session JSONL (`version: 3` header) | `packages/agent/src/harness/session/jsonl-storage.ts`, types in `packages/agent/src/harness/types.ts` | A writer emits a field no reader understands; old sessions stop replaying |
| RPC / IPC payloads | `packages/server/src/ipc/protocol.ts` (discriminated `type` unions), `packages/coding-agent/src/modes/rpc/` | Client and supervisor drift on the same message name |
| Model catalog | `packages/ai/scripts/generate-models.ts` → `packages/ai/src/models.generated.ts` → `models-store.ts` → CLI selectors | Hand-edited generated data disappears on the next regeneration |
| Agent events | `packages/agent/src/types.ts` (`AgentEvent`), consumed by `packages/coding-agent/src/modes/` | A new event type silently drops out of one mode's `switch` |
| Tool results | `packages/agent/src/harness/tools/` → renderers in `packages/coding-agent/src/core/tools/` | Renderer re-parses raw tool output instead of the typed result |

### Checklist: After Adding An Event Kind, JSONL Field, Or RPC Message

- [ ] Add the variant to the owning type module, not to the consumer
- [ ] Add or extend the type guard / decoder next to that type
- [ ] Check every `switch` over the discriminant; add a `never` exhaustive guard
      where one is missing (see `packages/agent/src/proxy.ts:361`)
- [ ] Keep id / sequence assignment in the writer only
- [ ] Confirm persisted-format changes stay readable for existing session files,
      or bump the version and handle both
- [ ] Add a regression under `packages/coding-agent/test/suite/regressions/`
      when the change fixes a reported issue

---

## Remote Probe Checklist

When code probes a remote resource to decide behavior — for example
`withRemoteCatalog` in `packages/coding-agent/src/core/remote-catalog-provider.ts`
deciding whether a remote model catalog is available:

Before implementing:

- [ ] The probe runs on **all** code paths that consume its result, not just the
      interactive one
- [ ] "Not found" and "transient failure" are distinguished; a timeout must not
      be reported as absence
- [ ] Transient errors abort or retry; they never silently switch mode
- [ ] Caches and prefetched data are reset when the source changes

After implementing:

- [ ] Trace every path from probe result to the decision branch — no fallthrough
- [ ] Read the full response before parsing; never parse a fixed-size prefix as
      complete JSON
- [ ] Shortcut paths (explicit flag skipping a picker) get the same error
      handling quality as the probed path

---

## When to Create Flow Documentation

Create detailed flow docs when:

- Feature spans 3+ layers
- Multiple teams are involved
- Data format is complex
- Feature has caused bugs before

---

## Event Log / Projection Boundary

Append-only logs are cross-layer contracts. A single event travels through:

```
user input → agent loop → session JSONL → reader → projection → mode renderer
```

In this repo that chain is `packages/agent/src/agent-loop.ts` →
`packages/agent/src/harness/session/jsonl-storage.ts` (`version: 3` header) →
`jsonl-repo.ts` → `packages/coding-agent/src/modes/`. The persisted format is
documented in `packages/coding-agent/docs/session-format.md`.

### Checklist: After Adding A New Event Kind Or Field

- [ ] Add the event kind to the central event taxonomy
- [ ] Add a typed event variant or type guard at the event layer
- [ ] Add normalization helpers for array/object fields that come from
      user input or JSON
- [ ] Keep `seq` / `id` assignment in the event writer only
- [ ] Make filters and reducers consume the typed event guard, not local casts
- [ ] Make display code consume reducer output or typed events, not raw JSON
- [ ] Add at least one regression that proves history replay and live filtering
      use the same filter model

**Why the version header matters**: `SessionHeader` in
`packages/agent/src/harness/session/jsonl-storage.ts` pins `version: 3`. Any
change to the persisted entry shape must either stay readable for files written
by earlier versions or bump that number and handle both paths on read. Sessions
on disk outlive the code that wrote them.
