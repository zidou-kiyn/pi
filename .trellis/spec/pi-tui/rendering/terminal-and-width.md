# Terminal And Width

## When This Applies

Any change touching cell-width math, ANSI/OSC/APC parsing, differential
rendering strategy, or image-line detection: `packages/tui/src/utils.ts`,
`packages/tui/src/tui.ts` (`compositeTuiLine`), `TuiMainScreen.ts`,
`TuiAltScreen.ts`, `terminal-image.ts`, `terminal-colors.ts`, `terminal.ts`,
`native-modifiers.ts`.

## The Local Pattern

### One width function, no duplication

`graphemeWidth()` (`packages/tui/src/utils.ts:167`) is the only place cell
width is computed per grapheme. `visibleWidth()` (`utils.ts:216`),
`truncateToWidth()` (`utils.ts:1007`), `wrapTextWithAnsi()` (`utils.ts:786`),
`sliceWithWidth()`, and `extractSegments()` all route through it via
`Intl.Segmenter` grapheme iteration. Do not hand-roll a second width
calculation; add a case to `graphemeWidth()` instead.

Special cases baked into `graphemeWidth()`:

- Tabs are fixed width 3 (`utils.ts:168`), matching
  `normalizeTerminalOutput()` which expands `\t` to three spaces
  (`utils.ts:355`) so tab stops in the real terminal cannot desync layout.
- Regional indicator singletons (`U+1F1E6`-`U+1F1FF`) are forced to width 2
  even when isolated, because terminals render a lone flag half as
  full-width during streaming — proven by
  `test/regression-regional-indicator-width.test.ts`.
- `couldBeEmoji()` (`utils.ts:27`) is a cheap codepoint-range pre-filter run
  before the expensive `rgiEmojiRegex` (`\p{RGI_Emoji}`) test, so ordinary
  ASCII/CJK text never pays for emoji detection.
- Trailing halfwidth/fullwidth forms and Thai/Lao AM vowels that segment
  with a base character add their own width on top of the base
  (`utils.ts:198-208`).

### ANSI/OSC/APC parsing has one entry point

`extractAnsiCode()` (`utils.ts:382`) recognizes CSI (`ESC [ ... m/G/K/H/J`),
OSC (`ESC ] ...` terminated by BEL or `ESC \`), and APC (`ESC _ ...`, used
for `CURSOR_MARKER` in `tui.ts`) sequences. Every width/truncate/wrap/slice
helper calls this before treating a character as visible text. New escape
sequence handling must extend this function, not add a second scanner.
`AnsiCodeTracker` (`utils.ts` class, used by `wrapTextWithAnsi`) replays SGR
codes across wrapped lines so styles do not bleed past a line break;
`getLineEndReset()` closes only underline and OSC 8 hyperlinks at line end
to avoid clobbering background colors set by callers like `Box`/`Text`.

### Overlay compositing excludes partial wide graphemes

`compositeTuiLine()` (`tui.ts:253`) short-circuits to the raw line when
`isImageLine(baseLine)` is true (never composites text over an image escape
sequence), then calls `extractSegments()` with `strict=true` so a wide
grapheme (e.g. `让`, width 2) straddling the overlay boundary is dropped from
`before` rather than rendered partially —
`test/regression-overlay-cjk-boundary.test.ts` asserts `before` stops at 4
visible columns, not inside the CJK character at column 5.

### Differential rendering strategies

`TuiMainScreen.doRender()` (`TuiMainScreen.ts:146`) picks one of three
strategies in order, each logged via `PI_DEBUG_REDRAW=1`
(`TuiMainScreen.ts:219`):

1. First render (`previousLines.length === 0`): write everything,
   `fullRender(false)` — no scrollback clear.
2. Width change, height change (except Termux), or shrink-below-max with no
   overlay and `getClearOnShrink()` true: `fullRender(true)` — clears
   scrollback via `\x1b[2J\x1b[H\x1b[3J`, tracked by `test/tui-shrink.test.ts`.
3. Otherwise: move the cursor to the first changed line and rewrite only
   changed lines (`TuiMainScreen.ts:373` onward).

Every write is wrapped in synchronized-output markers
`\x1b[?2026h` / `\x1b[?2026l` (`TuiMainScreen.ts:178,200`) so terminals that
support CSI 2026 apply the frame atomically. `TuiAltScreen` owns a
terminal-height viewport instead of scrollback: it slices `contentLines` to
`[scrollTop, scrollTop + height)` and, for Kitty images scrolled partially
out of view, crops the placement with `cropKittyImageLine()`
(`TuiAltScreen.ts:371-380`) rather than redrawing. iTerm2 has no crop/delete
operation for inline images, so `TuiAltScreen` renders image components as
plain text stand-ins under iTerm2 (see `packages/tui/README.md:548`
"Alternate-screen image compatibility"); `TuiMainScreen` still draws iTerm2
inline images normally.

### Image lines skip width checks by design

`isImageLine()` (`terminal-image.ts:149`) uses `includes()`, not
`startsWith()`, to find `KITTY_PREFIX` (`\x1b_G`) or `ITERM2_PREFIX`
(`\x1b]1337;File=`) anywhere in the line. `startsWith()` was the original
implementation and caused a crash: when no image protocol is detected,
`getImageEscapePrefix()` returned `null`, so lines with image data after
leading text were not recognized as images and went through the normal
`visibleWidth()` check against a 300KB+ base64 payload, throwing
`Rendered line exceeds terminal width`. Regression:
`test/bug-regression-isimageline-startswith-bug.test.ts`.

### Native modifier detection degrades to false

`isNativeModifierPressed()` (`native-modifiers.ts:51`) loads a prebuilt
native addon (`native/darwin/prebuilds/darwin-<arch>/darwin-modifiers.node`)
only on `darwin` with `x64`/`arm64`, used to distinguish a real Shift+Enter
from Apple Terminal's ambiguous `\r` sequence
(`terminal.ts:APPLE_TERMINAL_SHIFT_ENTER_SEQUENCE`). Any load or call
failure is swallowed and the function returns `false` — this is a silent
capability gap on other platforms/archs, not a bug to "fix" by throwing.

## Reference Files

- `packages/tui/src/utils.ts` — `graphemeWidth`, `visibleWidth`,
  `extractAnsiCode`, `truncateToWidth`, `wrapTextWithAnsi`, `extractSegments`
- `packages/tui/src/tui.ts` — `compositeTuiLine`, `TuiBase.doRender` overlay
  compositing call site
- `packages/tui/src/TuiMainScreen.ts` — `doRender()` strategy selection
- `packages/tui/src/TuiAltScreen.ts` — viewport slicing, Kitty crop, iTerm2
  fallback
- `packages/tui/src/terminal-image.ts` — `isImageLine`, `getCapabilities`
- `packages/tui/test/regression-regional-indicator-width.test.ts`
- `packages/tui/test/regression-overlay-cjk-boundary.test.ts`
- `packages/tui/test/bug-regression-isimageline-startswith-bug.test.ts`
- `packages/tui/test/tui-overlay-style-leak.test.ts`
- `packages/tui/test/tab-width.test.ts`
- `packages/tui/test/tui-shrink.test.ts`
- `packages/tui/test/truncate-to-width.test.ts`, `wrap-ansi.test.ts`

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| Computing width with `.length` or a third-party string-width call elsewhere | Diverges from `graphemeWidth()` special cases (tabs, regional indicators, CJK trailing forms) | `utils.ts:167-210` |
| Using `line.startsWith(prefix)` to detect image escape sequences | Exact bug this package already fixed | `bug-regression-isimageline-startswith-bug.test.ts` |
| Compositing overlay text without `strict` boundary handling | Splits a wide grapheme and renders a corrupted half-character | `regression-overlay-cjk-boundary.test.ts` |
| Adding a second ANSI/OSC scanner instead of extending `extractAnsiCode` | Two parsers drift; only one is used by the tracker that prevents style bleed | `utils.ts:382`, `tui-overlay-style-leak.test.ts` |
