# Soksak embedding branch

This branch starts from upstream commit `d96ba57121761cbafda4c08179fe45f8e4cf8212`
(`vtparse-0.7.0-194-gd96ba571`). It contains the headless terminal APIs and terminal-state fixes
required by the Soksak provider. Soksak component repositories pin an exact commit from this branch;
they do not depend on a workspace path.

The terminal model owns cursor shape and blink state. DECSCUSR selects block, underline, or bar and
its blink value. DECSET/DECRST 12 changes only the blink value of the current shape. DECRQM 12 reports
that engine state. Embeddings read `Terminal::cursor_pos().shape`; they do not parse the input stream
again.

The terminal model also owns explicit OSC 4/10/11/12 color override presence. `ColorOverrides`
reports the raw indexed, foreground, background and cursor values separately from the effective
`ColorPalette`. Setting a color equal to the configured base remains an explicit override. OSC
reset and RIS clear the corresponding presence. Embeddings consume this state instead of
comparing the effective palette with configured colors or parsing terminal input again.

The owner test is:

```sh
cargo test -p wezterm-term dec_mode_12_changes_cursor_blink_without_changing_shape
cargo test -p wezterm-term osc_color_overrides_preserve_explicit_presence_and_reset
```
