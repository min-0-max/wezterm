# Upstream PR draft — wide grapheme at the right margin ignores DECAWM

**Target:** `wez/wezterm`, crate `wezterm-term` (`term/src/terminalstate/performer.rs`).
`tattoy-wezterm-term` is a republish of that crate, so the fix belongs upstream; a
backport to the tattoy republish is only needed if upstream lands slowly.

**Status: draft. Not submitted.** Submission is a user decision.

---

## Title

term: wrap a double-width grapheme that does not fit before the right margin

## Problem

`flush_print` decides where to put a grapheme by asking whether the cursor has passed the
right margin (`wrap_next`, set by the *previous* print), but never whether the grapheme
about to be printed actually *fits* in the columns that remain:

```rust
let x = self.cursor.x;
let width = self.left_and_right_margins.end;
let wrappable = x + print_width >= width;
...
self.screen_mut().set_cell_grapheme(x, y, g, print_width, pen, seqno);
```

With 79 of 80 columns filled and a double-width grapheme next, `wrap_next` is false (the
79th narrow cell fit exactly), so no wrap happens and the grapheme is written at column 79
with `print_width == 2` — a cell that needs columns 79 *and* 80 in a grid whose last column
is 79.

A width-2 character occupies two adjacent cells (UAX #11) and cannot be placed in one. With
DECAWM enabled — the default — a character that does not fit before the right margin wraps
to the next line, and the single column it could not use is left blank. Alacritty, ghostty,
and vt100 all do this; two of them keep a dedicated cell state for that reserved column
(`LEADING_WIDE_CHAR_SPACER`, `WIDE_SPACER_HEAD`), which is what the reserved column is for.

## Effect

The line ends up claiming 81 columns of content in an 80-column grid. Downstream that shows
up as a screen model that cannot be rendered faithfully, and — because the grapheme did not
move to a line of its own — a scrollback one row shorter than every other terminal produces
for the same byte stream.

## Reproduction

```rust
let mut term = Terminal::new(/* 80x24 */);
term.advance_bytes(b"aaaaaaaa...a");  // 79 narrow cells
term.advance_bytes("가".as_bytes());   // width 2, one column left
```

Expected: the row holds 79 `a` and one blank column; `가` begins the next line.
Actual: `가` is written into column 79 with width 2.

## Fix

Before printing, wrap when the grapheme is wider than the columns that remain:

```rust
// A double-width grapheme occupies two adjacent cells and cannot be split
// by the right margin, so "doesn't fit" means fewer columns remain than the
// grapheme is wide, not just that the cursor is past the margin. When
// DECAWM is enabled such a grapheme moves to the next line, and the single
// column it could not use is left blank.
if self.dec_auto_wrap
    && print_width > 1
    && self.cursor.x + print_width > self.left_and_right_margins.end
{
    let y = self.cursor.y;
    let screen = self.screen_mut();
    let phys_y = screen.phys_row(y);
    screen.line_mut(phys_y).set_last_cell_was_wrapped(true, seqno);
    self.new_line(true);
}
```

The condition only fires for a grapheme that cannot fit: a narrow grapheme in the last
column still satisfies `x + 1 == width` and takes the existing deferred-wrap path
unchanged, and a wide grapheme with two columns left (`x + 2 == width`) is likewise
untouched. The line is marked wrapped so reflow keeps the logical line together, matching
the existing `wrap_next` path.

Behaviour with DECAWM disabled is left as it is; this change is about what DECAWM requires.

## Testing

The crate's own suite passes unchanged (49 tests). A terminal-restore conformance suite that
grades an engine's screen against a declared golden — one that three other engines pass —
goes from 6/7 to 7/7 with this patch, with no change to the suite.
