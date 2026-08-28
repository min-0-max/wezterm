# Soksak embedding branch

This branch starts from upstream commit `d96ba57121761cbafda4c08179fe45f8e4cf8212`
(`vtparse-0.7.0-194-gd96ba571`). It contains the headless terminal APIs and terminal-state fixes
required by the Soksak provider. Soksak component repositories pin an exact commit from this branch;
they do not depend on a workspace path.

The terminal model owns cursor shape and blink state. DECSCUSR selects block, underline, or bar and
its blink value. DECSET/DECRST 12 changes only the blink value of the current shape. DECRQM 12 reports
that engine state. Embeddings read `Terminal::cursor_pos().shape`; they do not parse the input stream
again.

The owner test is:

```sh
cargo test -p wezterm-term dec_mode_12_changes_cursor_blink_without_changing_shape
```
