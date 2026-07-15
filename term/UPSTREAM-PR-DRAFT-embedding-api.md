# Upstream PR draft — two reads that force a copy on embedders

**Target:** `wez/wezterm`, crate `wezterm-term` (`term/src/terminal.rs`, `term/src/screen.rs`).
`tattoy-wezterm-term` is a republish of that crate, so these belong upstream.

**Status: draft. Not submitted.** Submission is a user decision. Smaller and less
interesting than the `wezterm-cell` true color fix; it is two additive methods, no
behavior change.

---

## Title

term: let an embedder observe actions, read a line, and skip the writer thread

## Problem 1 — watching the action stream costs a second parse, or a full buffer

An embedder that needs to see what the parser produced — to record it, log it, or track
state the terminal model does not expose through a getter (most DEC private modes have no
public accessor) — has two options today, and both are wasteful:

- run a second parser over the same bytes, parsing everything twice; or
- collect the actions into a `Vec` and hand that to `perform_actions`, which buffers the
  entire stream before any of it is applied.

The second is the tempting one, and it is the one that bites: on a multi-megabyte burst
the `Vec<Action>` grows to millions of entries, and the reallocations show up in a profile
as `realloc` → `vm_copy` → `mach_msg2_trap`, the kernel copying multi-megabyte buffers.
`advance_bytes` already does the right thing — it streams, applying each action as the
parser emits it — but it gives the caller no way to look.

### Fix

```rust
pub fn advance_bytes_observed<B: AsRef<[u8]>, F: FnMut(&Action)>(&mut self, bytes: B, observer: F)
```

`advance_bytes` becomes `advance_bytes_observed(bytes, |_| {})`. The observer sees exactly
the actions this terminal applies, in order, as they are applied — one parse, no buffer.

Dropping a `Vec<Action>` in favor of this tap was worth **+8%** feed throughput
(69.7 → 75.5 MB/s, same machine, release, median of 7) on an 8.5 MB stream.

## Problem 2 — reading a line costs a clone

`Screen` exposes `line_mut(idx) -> &mut Line`, but the only public way to *read* a line is
`lines_in_phys_range`, which clones:

```rust
pub fn lines_in_phys_range(&self, phys_range: Range<PhysRowIndex>) -> Vec<Line> {
    self.lines.iter().skip(..).take(..).cloned().collect()
}
```

A reader walking the scrollback to serialize it pays an allocation per row, for lines it
never mutates. (`all_lines` exists but is `cfg(test)`.)

### Fix

```rust
/// Get a reference to a line, relative to start of scrollback.
#[inline]
pub fn line(&self, idx: PhysRowIndex) -> &Line {
    &self.lines[idx]
}
```

The counterpart to `line_mut`, for readers that only need to look.

## Problem 3 — the writer thread is mandatory, even when the writer cannot block

`TerminalState::new` always wraps the writer:

```rust
let writer = BufWriter::new(ThreadedWriter::new(writer));
```

The reason is sound and is documented on `ThreadedWriter`: the writer is the pty, writing
to it can block when the child is not reading, and blocking the thread that drains the
child's output would deadlock a large paste.

But it is unconditional. An embedding whose writer *cannot* block — a headless terminal
that captures answerbacks into memory rather than sending them anywhere — pays for that
safety without needing it: a thread per terminal (a mux of N panes spawns N of them), an
allocation and a channel send per write, and, because delivery is now asynchronous, no way
to read back what a given chunk of input answered without first synchronizing with that
thread. An embedder that wants deterministic answerback capture ends up building a barrier
(write a unique marker through the same channel, block until the writer sees it) — a
workaround for a thread it never wanted.

### Fix

A `TerminalConfiguration` method, defaulting to today's behavior:

```rust
fn threaded_writer(&self) -> bool { true }
```

`TerminalState::new` consults it and uses the writer directly when it is false. Interactive
terminals are unaffected; they never implement it.

The cost of the thread is not visible in a benchmark that feeds one big buffer, because it
is paid *per feed call*, not per byte. Feeding the same 8.5 MB stream in the chunk sizes a
pty actually delivers shows it — and shows it disappear:

| chunk | with writer thread (+ barrier) | direct writer |
|---|---|---|
| one 8.5 MB buffer | 101.4 MB/s | 98.3 MB/s |
| 256 KB | 101.8 MB/s | 98.4 MB/s |
| 64 KB | 99.8 MB/s | 98.5 MB/s |
| 16 KB | 95.3 MB/s | 97.9 MB/s |
| 4 KB | **84.3 MB/s** | **98.5 MB/s** |

Throughput goes flat: the per-chunk cost is gone, and with it the barrier the embedder
needed in order to read answerbacks at all.

## Correctness

None of the three changes behavior by default. `advance_bytes` keeps its exact semantics
(the observer is a no-op closure), `line` is a new read path, and `threaded_writer`
defaults to true, so an embedder that does not implement it gets the writer thread exactly
as before. `wezterm-term`'s test suite: 49/49 pass.
