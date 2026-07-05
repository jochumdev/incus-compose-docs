---
date: 2026-07-05T01:28:11.187Z
dateCreated: 2026-07-05T01:03:26.331Z
description: null
editor: markdown
published: true
tags: null
title: Progress
leafwiki_id: pazuq_fvg
leafwiki_title: Progress
leafwiki_created_at: "2026-07-05T03:54:00.685195744Z"
leafwiki_updated_at: "2026-07-05T04:14:14.26629555Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# Progress

Live progress reporting for long-running Incus operations (image pulls,
instance lifecycle). The client emits progress events; a renderer turns them
into terminal output.

## Data Flow

```
Incus operation
  -> operation hook registers op.AddHandler   (client)
  -> reportProgress reads Metadata "*_progress"
  -> Progress{Percent, Text}
  -> progressHandler callback                  (set by the consumer)
  -> renderer paints a line                    (cmd/incus-compose)
```

Two distinct signals feed a line, and they arrive on different paths:

- **Progress events** report mid-operation state via the handler callback.
- **Completion** is _not_ a progress event. It arrives as an after-hook, so the
  renderer also registers `AddHookAfter` to mark a line done. This is what lets
  quick actions (start, stop, delete) that never emit progress still report.

## Client Side

### SetProgressHandler

Register a callback for live operation progress; pass `nil` to disable. Operations
run in parallel, so the handler may be called concurrently and must be safe for
concurrent use.

```go
gc.SetProgressHandler(func(action client.Action, r client.Resource, _ client.Options, p client.Progress) {
    // render p for r
})
```

The handler is wired in through the operation hooks: when an action runs an Incus
operation, the hook calls `op.AddHandler`, which on each update reads the operation
Metadata for a key ending in `_progress`, parses a `NN%` out of it, and invokes the
handler.

Not every wait is an Incus operation. In-client waits that block without an
operation (e.g. an instance blocking on a dependency's health) call
`emitProgress` to push a synthetic `Progress{Percent: -1, Text: ...}` straight to
the handler, so the line shows a spinner with status instead of stalling silently.
It lands on the acting resource's current line (same action key), so the wait and
the operation that follows share one line.

### Progress

```go
type Progress struct {
    Percent int    // 0-100, or -1 when the operation reports no percentage
    Text    string // raw status text from Incus, empty when none
}
```

Two sources, two shapes:

- **Native images** report a real percentage (`"rootfs: 42% (3.10MB/s)"`): `Percent`
  is set and `Text` carries the transfer speed.
- **OCI image pulls** emit only status text (`"Retrieving OCI image from registry"`):
  `Percent` is -1 because the registry download runs as an opaque skopeo subprocess
  with no byte or percentage feedback. Render these with a spinner, not a bar.

## Renderer (Reference Consumer)

`cmd/incus-compose/progress.go` is the canonical consumer. `startProgress` attaches
it and returns a finish func:

```go
finish := startProgress(globalClient, client, os.Stderr)
defer finish(success)
```

Attaching does three things: registers the renderer as the progress handler,
registers an `AddHookAfter` to mark lines done, and (in animate mode) reroutes log
output above the live block. The finish func clears the handler, flushes the final
frame, and restores log routing.

### Two Modes

Selected once, from whether stderr is a real terminal:

- **Animate** (terminal) - repaints the whole block in place each frame: a spinner
  ticker (120ms), progress bars, color, and cursor-up/clear sequences.
- **Plain** (piped output) - emits one line per distinct status change, no cursor
  control. Piped output and `NO_COLOR` both degrade to this cleanly.

Color is gated on `noColor`; cursor movement is gated on `animate` - so the two
concerns degrade independently.

### Line Identity and Ordering

Lines are keyed by `action + "/" + IncusName()`, so a resource that goes through
several actions (restart = stop then start) gets one line per action. Batches run in
priority order, so images report done before instances.

### Log Interleaving

While a live block is on screen, slog output would be overwritten by the next
repaint. To avoid that, log records are routed through a `swapWriter` to a
`bypassWriter` that prints whole lines _above_ the block (erase block, write,
repaint below). Partial lines are buffered until their newline arrives so a torn
write cannot split the block. Plain mode has no in-place block, so it skips this.

### Concurrency

Operations run in parallel, so `handle`, `markDone`, and the bypass writer all guard
shared state with a single mutex. The spinner ticker takes the same lock before each
repaint.

### Constraints

- **ASCII only** - the spinner is `- \ | /`; braille frames would violate the
  no-non-ASCII rule and misrender in narrow terminals.
- **Truncate to width** - every line is cut to the terminal width. A wrapped line
  spans two rows and breaks the cursor-up repositioning, leaving stale copies of the
  block behind.
- **Clean final frame** - on success every line is marked done so the last frame
  reads cleanly; on failure the last observed state is left in place.

## See Also

- [Hooks](/docs/architecture/client/hooks) - the after-hook that drives completion
- [Image](/docs/architecture/client/image) - image pull progress (native vs OCI)
