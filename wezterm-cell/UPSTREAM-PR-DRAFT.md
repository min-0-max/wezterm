# Upstream PR draft — a true color cell costs a heap allocation

**Target:** `wez/wezterm`, crate `wezterm-cell` (`wezterm-cell/src/lib.rs`).
`tattoy-wezterm-cell` is a republish of that crate, so the fix belongs upstream; a
backport to the tattoy republish is only needed if upstream lands slowly.

**Status: draft. Not submitted.** Submission is a user decision.

---

## Title

cell: carry true color by value instead of spilling it to `FatAttributes`

## Problem

`CellAttributes` keeps its two colors in a `SmallColor`, which can hold exactly two things:

```rust
enum SmallColor {
    Default,
    PaletteIndex(PaletteIndex),
}
```

Anything else — that is, *any 24-bit color* — does not fit, so `set_foreground` takes the
spill path and puts it on the heap:

```rust
foreground => {
    self.foreground = SmallColor::Default;
    self.allocate_fat_attributes();          // Box::new(FatAttributes { .. })
    self.fat.as_mut().unwrap().foreground = foreground;
}
```

`FatAttributes` is documented as holding "relatively rarely used attributes". True color is
not rare: it is what every 24-bit-color TUI emits for every cell it paints. Once the pen
holds a `Box`, the cost is paid per *cell*, not per *escape sequence*, because printing
clones the pen into the cell:

```rust
// terminalstate/performer.rs, flush_print
let pen = self.pen.clone();                  // clones the Box
self.screen_mut().set_cell_grapheme(x, y, g, print_width, pen, seqno);
```

and overwriting a cell drops the cell that was there, freeing its `Box`. So repainting a
screen of true color costs roughly one malloc and one free per cell — for a color that is
16 bytes of plain data.

Under a profiler, feeding a stream whose bulk is a full-screen true color redraw (80x24,
a distinct `ESC[38;2;R;G;Bm` per cell, 200 frames), **16.4% of all samples were inside the
allocator** (`nanov2_malloc_type`, `_nanov2_free`, `_realloc`), reached from
`allocate_fat_attributes`, from the `pen.clone()` in `flush_print`, and from dropping
overwritten cells.

## Fix

Widen `SmallColor` so it can hold every case `ColorAttribute` can express, and let the
colors live in the cell:

```rust
enum SmallColor {
    Default,
    PaletteIndex(PaletteIndex),
    TrueColorWithDefaultFallback(SrgbaTuple),
    TrueColorWithPaletteFallback(SrgbaTuple, PaletteIndex),
}
```

`set_foreground` / `set_background` become total and allocation-free, and `foreground()` /
`background()` no longer have to consult the heap first. The color fields come out of
`FatAttributes`, which goes back to holding only what is genuinely rare — hyperlinks,
images, underline color.

This is an internal representation change. `SmallColor` is private and the public API is
untouched: `set_foreground` still takes `impl Into<ColorAttribute>` and `foreground()`
still returns `ColorAttribute`, losslessly, including both true color fallback forms.

The trade is size for allocations: `CellAttributes` grows by the two inline `SrgbaTuple`s.
That is a good trade even on memory, because the allocation it removes is larger than the
growth and comes with allocator overhead: a `Box<FatAttributes>` per styled cell.

## Effect

Same machine, release build. The engine is fed an 8.5 MB stream that ends in the
true color redraw described above.

| | before | after |
|---|---|---|
| feed throughput (median of 7) | 86.5 MB/s | **101.6 MB/s** (+17%) |
| live heap held by a filled 1000-row scrollback of 24-bit-color rows | 5.87 MB | **4.56 MB** (−22%) |
| resident memory, same scrollback | 9.1 MB | **7.8 MB** |

Memory falls in both measures despite the wider struct, which is the point: the per-cell
`Box` cost more than the inline color it was standing in for.

## Correctness

Colors round-trip exactly, including the distinction between
`TrueColorWithDefaultFallback` and `TrueColorWithPaletteFallback`, which the old spill path
also preserved.

- `wezterm-term`'s test suite: 49/49 pass against the patched crate.
- An acceptance suite that compares full screen state (every cell's grapheme, width,
  attributes, and fg/bg as 24-bit hex) against declared goldens, over fixtures that paint
  true color: 7/7 pass, byte-identical goldens.

## Note

`--features use_serde` does not currently build in the tattoy republish, before this change
and after it (`Hyperlink: Serialize` is unsatisfied, and `wezterm_blob_leases::lease_bytes`
is missing). Untouched here. `SrgbaTuple` already derives `Serialize`/`Deserialize` under
that feature and `wezterm-cell` already forwards `wezterm-color-types/use_serde`, so the
new variants need nothing extra once that build is fixed.
