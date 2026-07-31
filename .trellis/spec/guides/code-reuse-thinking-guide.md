# Code Reuse Thinking Guide

> **Purpose**: Stop and think before creating new code - does it already exist?

---

## The Problem

**Duplicated code is the #1 source of inconsistency bugs.**

When you copy-paste or rewrite existing logic:
- Bug fixes don't propagate
- Behavior diverges over time
- Codebase becomes harder to understand

---

## Before Writing New Code

### Step 1: Search First

```bash
# Search for similar function names
grep -r "functionName" .

# Search for similar logic
grep -r "keyword" .
```

### Step 2: Ask These Questions

| Question | If Yes... |
|----------|-----------|
| Does a similar function exist? | Use or extend it |
| Is this pattern used elsewhere? | Follow the existing pattern |
| Could this be a shared utility? | Create it in the right place |
| Am I copying code from another file? | **STOP** - extract to shared |

---

## Common Duplication Patterns

### Pattern 1: Copy-Paste Functions

**Bad**: Copying a validation function to another file

**Good**: Extract to shared utilities, import where needed

### Pattern 2: Similar Modules

**Bad**: Creating a module that is 80% similar to an existing one — a new
provider file that re-implements request building instead of reusing
`packages/ai/src/api/` shared helpers, or a TUI widget that re-implements
wrapping instead of reusing `packages/tui/src/components/`.

**Good**: Extend the existing module with options, or lift the shared part into
the layer that already owns it.

### Pattern 3: Repeated Constants

**Bad**: Defining the same constant in multiple files

**Good**: Single source of truth, import everywhere

### Pattern 4: Repeated Payload Field Extraction

**Bad**: Multiple consumers cast the same JSON/event fields locally:

```typescript
const description = (ev as { description?: string }).description;
const context = (ev as { context?: ContextEntry[] }).context;
```

This is duplicated contract logic even when the code is only two lines. Each
consumer now has its own definition of what a valid payload means.

**Good**: Put the decoder, type guard, or projection next to the data owner:

```typescript
if (isThreadEvent(ev)) {
  renderThreadEvent(ev);
}
```

**Rule**: If the same untyped payload field is read in 2+ places, create a
shared type guard / normalizer / projection before adding a third reader.

---

## When to Abstract

**Abstract when**:
- Same code appears 3+ times
- Logic is complex enough to have bugs
- Multiple people might need this

**Don't abstract when**:
- Only used once
- Trivial one-liner
- Abstraction would be more complex than duplication

---

## After Batch Modifications

When you've made similar changes to multiple files:

1. **Review**: Did you catch all instances?
2. **Search**: Run grep to find any missed
3. **Consider**: Should this be abstracted?

### Reducers Should Use Exhaustive Structure

When state is derived from action-like values (`action`, `kind`, `status`,
`phase`), prefer a reducer with one `switch` over scattered `if/else` updates.

```typescript
// BAD - action-specific state transitions are hard to audit
if (action === "opened") { ... }
else if (action === "comment") { ... }
else if (action === "status") { ... }

// GOOD - one reducer owns the transition table
switch (event.action) {
  case "opened":
    ...
    return;
  case "comment":
    ...
    return;
}
```

This matters when the event log is the source of truth. A reducer is the
documented replay model; display code and commands should not duplicate pieces
of that replay model.

---

## Checklist Before Commit

- [ ] Searched for existing similar code
- [ ] No copy-pasted logic that should be shared
- [ ] No repeated untyped payload field extraction outside a shared decoder
- [ ] Constants defined in one place
- [ ] Similar patterns follow same structure
- [ ] Reducer/action transitions live in one reducer or command dispatcher

---

## Gotcha: Union Widened Without An Exhaustive Guard

**Problem**: Adding a member to a discriminated union (event `type`, message
`role`, stop reason) silently falls into an existing `default` branch. Nothing
fails to compile, and the new case is quietly dropped at runtime.

**Prevention**: end the `switch` with a `never` assignment so the compiler
flags every unhandled member. The repo already does this:

```typescript
default: {
	const _exhaustiveCheck: never = proxyEvent;
	console.warn(`Unhandled proxy event type: ${(proxyEvent as any).type}`);
	return undefined;
}
```

Reference implementations:
- `packages/agent/src/proxy.ts:361`
- `packages/ai/src/api/google-shared.ts:352`
- `packages/ai/src/api/openai-responses-shared.ts:752`

When you widen a union, grep for every `switch` over that discriminant — a
branch without a `never` guard will not tell you it is now incomplete.

---

## Gotcha: Asymmetric Mechanisms Producing Same Output

**Problem**: When two different mechanisms must produce the same file set (e.g., recursive directory copy for init vs. manual `files.set()` for update), structural changes (renaming, moving, adding subdirectories) only propagate through the automatic mechanism. The manual one silently drifts.

**Symptom**: Init works perfectly, but update creates files at wrong paths or misses files entirely.

**Prevention**:
- **Best**: Eliminate the asymmetry — have the manual path call the automatic one (e.g., `collectTemplateFiles()` calls `getAllScripts()` instead of maintaining its own list)
- **If asymmetry is unavoidable**: Add a regression test that compares outputs from both mechanisms
- When migrating directory structures, search for ALL code paths that reference the old structure

**Real example in this repo**: `packages/coding-agent/npm-shrinkwrap.json` and
`packages/coding-agent/install-lock/` are both derived from the root
`package-lock.json`, each by its own generator
(`scripts/generate-coding-agent-shrinkwrap.mjs`,
`scripts/generate-coding-agent-install-lock.mjs`). The asymmetry is contained
because each generator also runs in `--check` mode inside `npm run check`, so a
hand-edited or stale artifact fails the build instead of drifting silently.
When you introduce a second mechanism that must produce the same output, ship
the `--check` counterpart with it.

---

## Single Registration Point

When a subsystem has a list that everything else reads from, add to that list —
do not start a parallel one.

| Subsystem | Single registration point |
|---|---|
| LLM providers | `packages/ai/src/providers/all.ts` — import the provider, then add it to the array returned by `builtinProviders()` |
| Model metadata | `packages/ai/scripts/generate-models.ts` → regenerates `packages/ai/src/models.generated.ts` |
| Agent tools | `packages/agent/src/harness/tools/index.ts` |

Missing the registration is the classic failure: the new file compiles, its
tests may even pass in isolation, but nothing reaches it at runtime.
`.pi/skills/add-llm-provider.md` exists precisely because the provider case has
several such touch points.

**Verification habit**: after adding a file to one of these subsystems, grep
the registration point for the new symbol before considering the work done.
