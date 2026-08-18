---
date: 2026-08-16T22:53:16.000Z
dateCreated: 2026-08-14T11:46:35.000Z
leafwiki_created_at: "2026-08-14T11:46:35.000000000Z"
leafwiki_updated_at: "2026-08-16T22:53:16.000000000Z"
---

# Architecture

One page per plugin, extracted from the code. The code is the shape, and where
they disagree the code is right.

```
incusd -> source -> debounce -> enricher -> dns -> http -> log
```

- [[architecture/ievent/source|source]] - the stream, and the walk. Not a plugin
- [[architecture/ievent/debounce|debounce]] - collapses a burst into two
- [[architecture/ievent/enricher|enricher]] - reads what an event's subject
  looks like now
- [[architecture/ievent/dns|dns]] - folds events into records and answers
  queries
- [[architecture/ievent/http|http]] - `/metrics`, `/health`, `/ready`
- [[architecture/ievent/log|log]] - prints what walked past

## The Contract

Everything below is `iutil.Plugin`, and every page here assumes it.

### Plugins Call Plugins

Each holds its successor and continues the walk itself, so the chain runs as a
call stack. A plugin can do work either side of `next`, which is what lets an
observer bracket what it observes from two positions - and what lets a plugin
hold an event and hand it on later, from its own goroutine. `debounce` releasing
the last of a burst is exactly `Next` called late; nothing else is needed for
it.

### Handle Is The Inbox Door

`Handle` runs on the _previous_ plugin's goroutine and must not block: it
enqueues and returns. The work happens on the plugin's own goroutine, and `next`
is called from there when that work is done.

So a plugin's state belongs to that one goroutine and to nothing else. A mutex
is for a plugin that genuinely cannot avoid a second reader - the handler
serving its metrics is the usual one - not the answer to `Handle` being called
from somewhere else.

Most plugins have no inbox at all. `Handle` rather than an exposed channel is
what lets them stay a function call, and what puts the decision about a full
queue on the plugin that is full, under its own name.

A plugin that acts on events checks the state first: one that is not `StateOk`
is only passing through for the observers to see.

### Wants Is Declared Before Anything Is Wired

```go
type Want struct {
    Action   string
    Enrich   Enrichment
    Debounce bool
}
```

Read once, from every plugin, in a sweep of its own. The source takes the union -
one entry per action - and hands every plugin the same finished table.

It has to be whole first because the enricher serves the entire chain from one
action. A table assembled during the wiring would be whichever part of the union
happened to exist when the enricher's own `Setup` ran.

The two fields fold opposite ways, and both toward **doing more work**:

- `Enrich` - true wins. An action anyone wants is walked, read for everything
  anyone asked of it.
- `Debounce` - false wins. One plugin needing every event stops the collapsing
  for everybody, and the zero value vetoes.

A plugin that gets either wrong pays for reads it did not need, or sees events
it could have skipped. Neither loses one. Which is why the zero value vetoes:
forgetting means debouncing quietly did not happen, and a burst arriving whole
is something you notice - the other way round an event goes missing, and that is
not.

Renames are the case to think about. Collapsing two of them keeps the last
`OldName` and loses the middle name entirely, so whatever holds records by name
never hears the first one went.

`nil` is none, and `nil` is what an observer returns: it is in the chain, so it
sees whatever walks without asking for anything of its own.

### Ordering Is The Binary's

Decided in `cmd/*/chain.go`, at compile time. A plugin does not say where it
belongs, because that only starts to pay when someone composes a chain out of
plugins they did not write.

What a deployment may do is leave a position out, and only the ones marked
optional. An `--exclude` naming a position that does not exist is an error
rather than a no-op - a typo is otherwise a plugin that silently stayed in.

### Plugins Trust Each Other

We write them, we maintain them, and our users run them. There is no third party
here, so nothing defends one plugin against another: no timeout on `Handle`, no
watchdog, no `recover` around a panic. A plugin that blocks the chain blocks it,
and a panic takes the process down.

**The test for anything proposed later:** if it only makes sense against a
plugin behaving badly, it does not get built.

That is about intent rather than accidents. The accessors on `Event` still
clone, because a caller holding a live map is a mistake worth making impossible
when it is free.

### An Event Is Derived, Never Mutated

Every field is unexported, and nothing changes one in place. `WithChainState`,
`WithDropped`, `WithFailed`, `WithInstance`, `WithNetworks`, `WithProject` and
`WithValue` all copy the struct (`next := *e`) and set one thing on the copy, so
an event held past the goroutine that produced it stays exactly what it was
then - what a plugin does to its own copy is invisible to whoever handed it in.

It carries no context and no deadline of its own; those belong to the read,
set where the source performs it. An event is safe to hold, to log, or to hand
to an observer goroutine long after the read that built it has finished.

### Enrichment Is A Set, Not A Bool Per Field

`Enrichment` is a bitset (`EnrichedInstance`, `EnrichedNetworks`,
`EnrichedProject`), not one flag and not three separate booleans, because "no
networks on this event" and "nobody asked to read them" are different answers -
a plugin acting on the first when what it actually has is the second publishes
an absence that was never real. `Enriched(want)` asks whether everything in
`want` landed; `Enrichments()` hands back the raw set for an observer with
nothing particular to ask.

### A Plugin Carries Its Own Data On The Event

`Value` and `WithValue` let a plugin stash something on an event without a side
channel of its own: `WithValue` derives a new event holding one more link in a
chain of nodes - written once, so deriving costs one node rather than copying a
map - and `Value` walks that chain back to front. The key must be an unexported
type owned by the plugin that sets it; nothing in the type system enforces that,
which is exactly why it is written down here rather than left to be discovered.
It is what lets a plugin carry something of its own forward through `next`
without `iutil` needing to know what any of it means.

### Options Are One Vocabulary, Not One Struct

There is no shared options type. Each plugin declares its own private `options`
or `Config`, with its own `Option` functional setters - `dns.Config`,
`enricher`'s `options`, `debounce`'s `options` - so that reading one plugin's
knobs is never a reason to import another's package. The enricher's own comment
on this is explicit: its `options` is its own "rather than a set shared with
every other plugin: naming one is already naming this package."

What is shared is a vocabulary of field names repeated where they apply, not a
type: `InboxSize` means the same thing and defaults to the same 1024 everywhere
it appears (`debounce`, `dns`, `enricher`), `Window` only exists on `debounce`,
`Workers`/`ReadTimeout`/`ProjectDelay`/`ReadDelay` only on `enricher`. A plugin
carries only the fields it has a use for; there is nothing to ignore, because
there is no wider struct to carry them.

That is the trade for not making `main` learn a new vocabulary per position, and
it is the same trade CoreDNS makes with its directives. The zero value of every
field means "unset", and each plugin fills its own default in: the right window
for `debounce` is not the right anything for anybody else.

Anything that is not machinery gets its own field on that plugin's own type
instead - `dns.Config` also carries the listener and the upstream resolver,
because those mean nothing to any other plugin.

### Command Goes In At The Head

A plugin telling the chain something raises a `Command`. It enters at the
**head**, not at `next`, so it reaches every position and in order against the
events that caused it. The end of a round is the case that needs it: the
enricher raises it, and everything _in front_ of it has to see it too.

The action namespace is what makes that readable. A slash prefixes whoever
raised it - `dns/ready`, `source/connected` - and the rule doing the work is
that an Incus lifecycle action cannot contain one.
