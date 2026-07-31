# Extension API surface used by this task

Condensed from `packages/coding-agent/docs/extensions.md` (120 KB, exceeds the
32 KB context-injection limit, so only the sections this task depends on are
extracted here). Line anchors point back into that file.

## `pi.registerCommand(name, options)` — `docs/extensions.md:1493`

```typescript
pi.registerCommand("stats", {
  description: "Show session statistics",
  handler: async (args, ctx) => {
    ctx.ui.notify(`${count} entries`, "info");
  },
});
```

- Duplicate command names are **not** an error: pi keeps every registration and
  assigns numeric invocation suffixes in load order (`/review:1`, `/review:2`).
  This is why a duplicated footer extension is silently loaded twice (R7) — the
  loader never dedupes by name.
- `getArgumentCompletions(prefix)` is optional; `/preset-sync` takes no
  arguments and does not need it.

## `ctx.ui` dialogs — `docs/extensions.md:2485`

```typescript
const ok = await ctx.ui.confirm("Delete?", "This cannot be undone");
ctx.ui.notify("Done!", "info"); // "info" | "warning" | "error"
```

Type signatures (`src/core/extensions/types.ts:135-142`):

```typescript
confirm(title: string, message: string, opts?: ExtensionUIDialogOptions): Promise<boolean>;
notify(message: string, type?: "info" | "warning" | "error"): void;
```

**Cancel semantics (`docs/extensions.md:2518`)**: `confirm()` returns `false` on
Escape, on "No", and on timeout. A single `if (!ok) return;` therefore covers
every decline path — the guarantee AC5 rests on.

`opts` supports `{ timeout, signal }`. `/preset-sync` uses neither: the sync
must not auto-cancel on a slow reader, and an auto-cancel is indistinguishable
from a deliberate decline without an `AbortController`.

## `ctx.hasUI` / `ctx.mode` — `docs/extensions.md:940-946`

- `ctx.hasUI` is `true` in TUI and RPC modes, `false` in print (`-p`) and JSON
  modes. It gates `select` / `confirm` / `input` / `editor` and the
  fire-and-forget `notify` / `setStatus` / `setFooter`.
- `ctx.mode` is `"tui" | "rpc" | "json" | "print"`; only `"tui"` may use
  `custom()` and direct TUI rendering.

**Consequence for `/preset-sync`**: when `ctx.hasUI` is `false` there is no way
to obtain consent, so the command must print the plan and exit without writing
rather than assume approval.

## `ctx.ui.setFooter` — footer registration

The shipped footer already uses this API (`vibrant-footer/index.ts:563,567`):

```typescript
ctx.ui.setFooter(undefined);                       // restore the default footer
ctx.ui.setFooter((tui, theme, footerData) => { … }); // install a custom footer
```

`setFooter` is **last-writer-wins**, not additive — which is precisely why a
double-loaded footer is hard to notice visually and must be prevented at the
filesystem level by `footer.demote` (R7/AC10) rather than by the render layer.
`footerData` exposes `onBranchChange`, `getGitBranch`, and
`getExtensionStatuses` (`index.ts:569,586,604`).
