---
title: MCP Queue Doctor
date: 2026-08-07
tags:
  - mcp
  - infrastructure
  - open-source
  - observability
description: An MCP server that diagnoses Postgres job queues instead of listing their rows — schema knowledge as data, honest silence over fake zeros, and reaching databases you can't connect to.
---

# MCP Queue Doctor

An [MCP](https://modelcontextprotocol.io) server that **diagnoses** Postgres job
queues — [pg-boss](https://github.com/timgit/pg-boss) and
[graphile-worker](https://github.com/graphile/worker) — rather than listing
their rows: retry storms, stuck workers, missed schedules, expiry overruns.
Every finding carries the evidence it was drawn from and the safest recovery
step, so the agent reading it can check the reasoning instead of trusting it.

- Source: [github.com/ethanasm/mcp-queue-doctor](https://github.com/ethanasm/mcp-queue-doctor)
- [npm](https://www.npmjs.com/package/mcp-queue-doctor) · listed in the
  [MCP Registry](https://registry.modelcontextprotocol.io/v0.1/servers?search=io.github.ethanasm/mcp-queue-doctor)
  as `io.github.ethanasm/mcp-queue-doctor`

The product is the difference between these two answers:

> ❌ "3 jobs in `enrichment/corpus-fill` are in state 'failed'."

> ✅ "`enrichment/corpus-fill` failed 140 times over 3 minutes, 91% sharing one
> error that looks like an upstream rate limit. This is one fault reproduced
> many times, not many separate faults — the fix belongs at the source, and
> bulk-retrying will reproduce it."

Existing queue MCPs expose task and worker operations. The first answer is
what raw introspection gives you; an agent armed only with it will happily
bulk-retry a rate-limit storm and make it worse.

## Where it came from

[[building-showbook|Showbook]] runs about thirty cron queues on pg-boss, and
every morning a health-check email tells me whether the night went fine. A
year of running that in production left behind something more valuable than
the email: a catalog of queue pathologies, each with a threshold tuned by a
real false positive or a real missed failure. The retry-storm rule exists
because a daily API quota once tipped over and 875 jobs failed in one night —
the count suggested 875 problems; the shape showed one. The expiry-overrun
rule exists because a corpus sweep quietly stopped fitting inside its
30-minute expiry once an upstream started throttling.

This is the third time one of my apps has grown an organ worth extracting —
after [[building-mcp-budget-governor|MCP Budget Governor]] (spend ceilings)
and [[building-provider-router|Provider Router]] (provider failover), and
alongside [[building-mcp-review|MCP Review]] on the agent-tooling side. The
pattern holds: the extraction is never just packaging. Generalising the
heuristics for queues that aren't mine forced every assumption about "normal"
into an explicit, overridable threshold — and the defaults being tuned to one
queue stopped being an implementation detail and became configuration the
moment anyone else could run it.

## Schema knowledge is data, matched by shape

pg-boss publishes no contract mapping releases to schema versions, and its
tables genuinely change — columns renamed, whole tables added and removed. The
adapter therefore keeps everything it knows about each schema generation in
one file, as data: which relations must exist, which must *not*, which columns
identify the generation. A live database is matched on its **observed shape**,
never on the version integer it happens to store.

Booting real pg-boss versions instead of trusting my recollection paid for
itself twice before launch. Once by correcting two column names I was sure
about. Once by catching a real bug: the archive table was removed in v11, not
v10 as I'd encoded — so the dialect calling itself "v10plus" was rejecting v10
databases outright. And when the tool meets a schema version it has never
been verified against, it says so — "matched on shape, not explicitly
verified" — instead of guessing silently.

## Silence over zeros

The second backend, graphile-worker, models work so differently that it
forced an honest design. It deletes a job on success, has no per-job expiry,
no worker heartbeats, and keeps its cron expressions in a config file rather
than the database. Four of the seven diagnosis rules have nothing to read.

The tempting implementation returns zero findings for those rules. But "0
missed schedules" *reads like a measurement* when the truth is "this backend
cannot tell you." So every backend declares its capabilities, rules gate on
them, and the ones that can't be answered stay silent — with the server
explaining each omission by name. Even inside a rule that runs on both
backends the honesty shows: on pg-boss a stuck-job finding can say the worker
stopped heartbeating three hours ago; on graphile it can only say the job has
been locked for five, because there is no heartbeat to read. Same rule,
narrower answer, and the narrowing is stated.

## Reaching a database you cannot connect to

The queues most worth diagnosing are the hardest to reach. My production
Postgres binds to loopback on a box with no port forwarding; the only ingress
is an HTTP tunnel. Every conventional fix is a door — publish the database,
add a TCP tunnel, join a VPN — and each grants far more than a diagnostic
needs.

So the server can run its SQL through a **read-only HTTP endpoint** instead of
a Postgres connection: the app in front of the database already exposes an
authenticated, engine-enforced read-only SQL route, and the diagnostic walks
through the door that already exists. The counterintuitive part is that this
is *stronger* than a direct connection, not weaker. The safety properties move
server-side, where they stop depending on my client being correct: the
endpoint owns the read-only transaction, the statement timeout, the row cap,
the rate limit, an audit log of every query — and can run as a database role
with narrower grants than the app itself.

## What dogfooding cannot prove

The transport work delivered the sharpest lesson of the project, in two
halves.

The first live run against the real endpoint failed instantly: the adapter
binds an array for a `= any($1)` filter, and the endpoint's parameter
validator only admitted scalars. Both sides' unit tests were green
throughout, and could not have caught it — a fake `fetch` answers whatever
it's told, so it proves a request is *well-formed*, never that it's
*acceptable*. Only running real software against real software found it.

Then a code review found the inverse case: the widened validator now admitted
boolean arrays, which the Postgres driver silently mis-binds — `[[true]]`
comes back as a single `false`, no error. My live run had passed *because*
this tool only ever sends arrays of strings. Dogfooding validates the paths
your caller takes; it says nothing about the rest of the surface you just
promised to support. The rule I keep from this pair: a unit test proves I
built what I meant, a live run proves I meant the right thing — and neither
one covers the contract you published beyond your own usage of it.

The first run against a real production queue closed the loop: thirty-two
queues, zero critical, zero warnings, and eleven informational notes — every
one of them true, flagging queues that are deliberately registered for
on-demand use and meant to sit idle. Correct findings that read as noise are
their own bug report: that run is why the thresholds became operator
configuration, and why an idle-queue claim is now bounded by the queue's own
retention window instead of overclaiming "has never run" from an empty table
that deletion had emptied.
