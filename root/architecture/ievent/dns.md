---
date: 2026-08-17T03:18:22.000Z
dateCreated: 2026-08-14T11:46:35.000Z
leafwiki_created_at: "2026-08-14T11:46:35.000000000Z"
leafwiki_updated_at: "2026-08-17T03:18:22.000000000Z"
---

# dns

The DNS half of the chain: it folds events into records and answers queries from
them. What it serves for an operator is [[coredns|the coredns page]]; this is
how it is built.

## A Plugin Owns What It Serves

It is not a plugin that hands records to a server somewhere else. It holds the
engine, starts the listener, and puts the forwarder behind itself, so `main`
composes a chain and knows nothing about DNS - which is what lets a second
binary be a copy of `main` with a different list.

Its listener outlives its fold loop. Answering from the last snapshot while the
chain drains beats refusing, and the two cannot hurt each other: one goroutine
writes an `atomic.Pointer` and every query goroutine reads it. That is the only
crossing in the package, and it is why there is no lock in it.

It serves UDP and TCP together, always: a truncated UDP reply is only useful to
a client that can retry over TCP. A panic from the query chain behind the
engine is recovered and answered SERVFAIL - the one place in this package that
recovers at all, unlike the rest of the ievent chain, because `Behind` is
arbitrary CoreDNS plugins this package did not write and cannot vouch for. The
adapter answering these is hand-shaped after CoreDNS's own
`dnsserver.Server.ServeDNS`, less what a single-chain binary has no use for -
server blocks, views, proxy protocol - and has to be diffed by hand against
upstream on every CoreDNS bump.

## Two Halves, One Atomic Pointer

```
events -> fold -> build -> Snapshot -> atomic store
                                          |
query -----------------------------> atomic load
```

**Work happens when things change, not when queries arrive.** A whole-fleet
snapshot is built when something moves, records rendered and filtered ahead of
time, and answering is then three map lookups - no filtering, no intersection,
no I/O. A query never touches Incus.

## The Engine Derives Nothing

`ecs_view`, under here, holds the records and answers. It knows nothing about
Incus, and `just boundary` enforces that it cannot even see where a record came
from. What keeps the query path free of derivation is that it _cannot_ derive.

A view is a set of network keys. The answer to a query depends on the querier
only through that set, so every querier on the same set gets identical answers
and they share one view.

## Placing The Querier

A querier is identified two ways: an EDNS0 client-subnet option, if a resolver
relayed one, otherwise the query's own source address - the option wins,
since a resolver forwarding on a client's behalf puts that client's address
there for every query it relays.

That address is checked against an instance's own claim first, and only then
against which known subnets it falls inside. Belonging to no host but sitting
inside one or more known networks makes it an anonymous member of all of
them, which is what lets something elsewhere on a bridge resolve records for
the other hosts on that bridge without being one itself.

An address that cannot be placed on any network at all is refused outright,
never answered from everything: there is no default view, because guessing
would leak one network's records to a querier on another, and a real client is
always on some network this server knows about. The zone apex is the one
answer that does not vary by querier - its scope is left at 0 so a resolver
caching an echoed subnet reply may share it fleet-wide - everything else is
scoped to the querier's whole address, since a different address may be a
different view.

## Building Is Fleet-Wide, Never Per Project

A set of networks can span projects: two instances on one shared bridge sit in
different projects and different zones while being on the same wire. Building
per project could only ever fill a view with the names of whichever project
built it, so those two would go mutually blind - and both halves of that read as
NXDOMAIN, which is indistinguishable from correct isolation.

## Labels Layer, Project Under Instance

A project's own `coredns.*` labels are the defaults its instances start from;
an instance's own labels override them field by field. `transfer` is the one
exception: it opts a _zone_ into transfer, so it is read off the project alone
and dropped if an instance tries to set it. `aliases` is the reverse
exception - it never inherits, since a project-wide alias would claim one name
for every instance in it.

The zone itself is `<project>.<suffix>` unless `user.label.coredns.zone` names
one instead. The name a scaled service answers to is
`user.label.coredns.service`, or `incus-compose.service` when compose set it -
compose wins, so a fleet built from a compose file is named by the file that
owns it and not by a label somebody typed once by hand.

## Forward Names

Held by project **and** name, because two projects may each have a `web` and
they are different hosts in different zones.

- `<instance>.<zone>` always
- `<service>.<zone>` when labelled - a scaled service's replicas append to one
  slice, which is all "a service name resolves to every replica" means
- aliases, as CNAMEs

An address claimed by more than one instance resolves to `AmbiguousView` rather
than to either of them. Two projects on overlapping subnets really can hand out
the same one, and deciding by map order would answer a querier with the other's
view, non-deterministically.

## Aliases And CNAMEs

`user.label.coredns.aliases` becomes a CNAME onto `<instance>.<zone>` rather
than a second copy of the addresses, so a host that moves changes them in one
place.

Rendered per network key from the target's own rendered records, which is what
makes an alias reach exactly as far as the name it points at and never wider.
The chase is done at build time - the CNAME goes into the A and AAAA sets as
well as its own, canonical name first - so answering through one stays three map
lookups.

One CNAME value serves every network the name is reachable on, since it is the
one record that does not vary by network. `Gather` drops the repeat by identity
when a querier shares more than one of them.

An alias is dropped, never resolved by picking, when it collides: a name a host
or service already answers to, a name two instances both claim, or a zone apex.
Each would put a CNAME beside other records or two CNAMEs at one name.

An absolute alias landing outside every zone gets one invented for its parent,
marked `Fallthrough`: that zone claims the aliased names and hands everything
else to the forwarder, so one label on one instance cannot blackhole a domain
the fleet still resolves. A name it _does_ hold is answered under the ordinary
rule, invisible included, so falling through cannot be used to tell a hidden
name from an absent one.

## Reverse Records

Every address is filed backwards under the name `dns.ReverseAddr` gives it,
keyed by the same network the forward record is keyed by. That keying is the
whole of it: a PTR then reaches a view through the same `Gather`, so an address
on a network the querier is not on reads as NXDOMAIN exactly as its forward name
does.

A reverse zone is the `/24` or `/64` an address lands in, taken from **the
address rather than from its network's prefix**. A subnet Incus manages but
nothing sits on is therefore never claimed. Where a bridge is the usual `/24` or
`/64` the two derivations agree; where it is a `/22`, deriving from the prefix
would make this server authoritative for the enclosing `/16`.

The network's prefixes still decide _whether_ there is a reverse at all: an
address outside every prefix of the network it sits on is served forward and not
backward. One instance bridged onto a public `/24` would otherwise have this
server answering NXDOMAIN for the rest of it. It is the one place the two
directions deliberately disagree.

## NS Records

`Zone.NS` is the apex NS set, read straight off `user.label.coredns.ns` and
resolved against the zone the way an alias is - a trailing dot absolute,
anything else relative. Empty falls back to a placeholder name nothing answers
for, harmless until the zone is transferred or delegated.

It is not derived from the fleet. A secondary that would receive it is outside
the fleet, or it would not need a transfer, so there is nothing in `held` to
read it off - the operator's own list, typed once at the delegation's parent,
is the only source there is.

The wire answer and the AXFR carry the same set. `Answer` and `transfer` share
`nsRecords`/`soa`, so there is exactly one place the NS set could disagree with
itself, and it does not exist: two secondaries handed different sets under one
serial could never tell that they had been, the same reasoning Zone Transfer
below already applies to the records themselves.

Two projects resolving to one zone name union their `ns` labels into one set,
the same way their instances union into one zone. A reverse zone unions every
contributing project's `ns`, since it belongs to no project of its own.

A zone `aliasZones` invents for one absolute alias never gets one: it is
`Fallthrough`, so `Answer` never reaches the apex branch for it, and handing a
name server for a domain this server holds one name in would make it
authoritative for the rest of that domain too.

## Zone Serials

A zone's serial moves when that zone's records move and at no other time. It is
derived from a hash taken **from the instances rather than the rendered
records**, because it has to notice a change in reachability and not only in
addresses: an instance moving between networks keeps its address while changing
who can see it, and that is a change to the zone. The NS set goes into the same
digest - relabeling `ns` moves no address, and a serial that does not move for
it is one no secondary ever re-transfers on.

The plugin stamps it because it is the only thing that knows what it published
last - the engine is a read-only view and holds nothing to compare against.

## Zone Transfer

AXFR and IXFR both answer from the current snapshot, not a journal - there is
no delta to cut, so an IXFR asking for anything but the serial already served
gets the whole zone anyway. UDP is refused outright; a transfer is a stream of
messages and UDP carries one.

A zone goes over whole or not at all, which is the one answer in this package
not filtered by who is asking: `Gather` still runs per name, but its result is
the union across every network that name is reachable on rather than one
querier's view, because the serial is per zone and not per view - two
secondaries handed different records under one serial could never tell that
they had been.

Two gates stand before the snapshot is even read: the peer's address against
an operator-configured allow-list (no prefixes configured allows nobody, so
transfer is opt-in at both the zone's `transfer` label and the listener's
config), and - once secrets are configured - the request's TSIG status, which
becomes the only assertion of who the peer is once it exists. A zone the
request names that isn't being served, or is served without `Transfer` set,
is refused with no case of its own: handing it over would make the secondary
authoritative for a domain holding a single alias.

Records are sorted by name before they're written, so two transfers of an
unchanged zone are the same byte stream. Everything is rendered and cut into
envelopes - capped under the 64 KiB wire limit, leaving room for the framing
and a TSIG record - before the peer's `Transfer.Out` reads any of it, off a
channel that is already closed: nothing is left running if the peer hangs up
mid-stream, because there was never a producer goroutine to hang.

## The Cold Store

With `ColdDir` set, the distilled instances and every zone's serial are written
to disk. Records can be read back from Incus in a second; a serial cannot be
recovered from anywhere, and a secondary reading one going backwards
re-transfers on every restart of this process.

Encoding happens on the fold goroutine and writing on its own, so a slow disk
cannot stall the chain. What crosses between them is a finished `[]byte`, which
is why the writer _cannot_ reach into live state rather than merely not doing
so. One slot, and a queued encoding is discarded rather than waited on: a newer
one says everything it said.

The on-disk form is explicit types with explicit tags, because the file outlives
the process that wrote it - a field renamed in Go must not silently change what
a running deployment can read back. A file of any other version is ignored
rather than migrated, since this is a cache and starting without it costs one
whole-fleet read.

## It Reconciles Its Own Store

`held` is this plugin's store, with the cold store behind it, and **a store is
reconciled by whoever owns it**. That is why `dns` keeps the connection at
`Setup` even though the enricher exists: the enricher's model is a second store,
nothing on the chain can keep the two from drifting, and a record dropped on
another plugin's word is dropped for a reason this one cannot check.

On `enricher/sweep-end` it asks a goroutine of its own about every project it
holds records in, and drops what is no longer served. No timer and no flag: the
round already comes round, and its end is the moment a listing is worth taking.

Per project it reads the project first, then lists its instance names:

| the project        |                                                       |
| ------------------ | ----------------------------------------------------- |
| gone (404)         | every record in it goes; no listing is made           |
| there, marker gone | the same - **for us an unserved project is a delete** |
| there and served   | its names are listed and diffed                       |
| could not be read  | nothing goes                                          |

The marker is the one question here Incus cannot answer, so `main` hands this
plugin the same predicate it hands the enricher. Reading the project also tells a
deleted project from an empty one, which its instance listing cannot: Incus
answers `/1.0/instances?project=gone` with `[]` and a 200, not a 404.

**Absence is decided against what was asked about.** The keys held in a project
are recorded when its listing goes out; what is pruned is that record minus the
listing. A record folded in while the request was in flight is not in the
record, so it survives - and it had to: nothing would put it back, because the
next round reads an unchanged instance and emits nothing.

A listing that failed drops nothing. An empty answer and an unreachable daemon
arrive the same way, and one of them means every record in the project.

This is also the only thing that reaches a name the cold store restored and
Incus lost while the process was down. Nothing announced that delete, so no
event carries it, and no round's diff contains it - the enricher's model started
empty.

## Warm Is This Plugin's To Set

`dns` is the last consumer, so it raises `Command{ChainState: ChainWarm}` when a
round ends. Set at the enricher it would be true while that round's own events
were still in flight; set here it says the whole chain has caught up. It is also
a claim that there is something worth acting on rather than that a listing
finished, and only the last position can make it.

The flag is set only when the command was taken. A chain left cold publishes no
live change ever again, and the next round says the same thing.

## Readiness Is An Event

`dns` raises `dns/ready` through `Command` when what it published becomes worth
answering from. It does not hold a reference to
[[architecture/ievent/http|http]], and http holds none to it: the event goes in
at the head and is folded in order against the events that caused it.

Edges rather than a level, so whoever folds them starts not-ready - the truth
before anything has been read.

## Known Gaps

- **A restored instance that still exists but is stopped keeps its cold-store
  addresses.** Incus still lists it, so the reconcile leaves it, and a read that
  finds no address returns before `held` is rewritten.
- **Upstream metrics label off `metrics.WithServer`**, which reads a context key
  `core/dnsserver` sets - and nothing here runs it. So `coredns_cache_*` and
  friends carry `server=""`. Ours does not: `ecs_view` takes the name as a field
  at construction.
- **Any upstream plugin links `core/dnsserver`**, because each carries a
  `setup.go` in its own package and Go compiles a package whole. It is a toll
  paid once rather than per plugin - `forward` is 408 packages and `cache` 415,
  nearly the same set - costing 8.7 MiB of binary and nothing at runtime. A
  `-light` build is `main` with `queryChain` returning nil.
