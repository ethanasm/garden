---
title: Provider Router
date: 2026-08-06
tags:
  - infrastructure
  - open-source
  - resilience
description: One capability, several interchangeable providers, ordered failover. Extracted from a flight tracker, and what building it in two languages caught.
---

# Provider Router

A small library for calling **one capability through several interchangeable
providers**: try them in a preferred order, fail over on rate limits and
transient errors, and normalize every provider's output to one contract.

- Source: [github.com/ethanasm/provider-router](https://github.com/ethanasm/provider-router)
- [PyPI](https://pypi.org/project/provider-router/) · [npm](https://www.npmjs.com/package/provider-router) (same name on both registries)
- Zero runtime dependencies in either language

## Where it came from

[[building-vacation-price-tracker|Vacation Price Tracker]] gets flight prices
from three sources: the Skiplagged MCP, the Kiwi.com MCP, and a Google Flights
scraper. In July 2026 Skiplagged's flight backend started returning sustained
`429`s. The fix at the time was a database setting and a human: notice the
refreshes failing, flip `flight_provider` to `kiwi`, move on.

That worked, and it was obviously the wrong shape. The knowledge that "if this
one is throttled, ask that one" was in my head and in a runbook, not in the
code.

The first instinct was to write the fallback inline. The second, more useful
instinct was noticing I had already written it somewhere else.
[[building-showbook|Showbook]] geocodes venues with Google Places, and falls
back to OpenStreetMap's Nominatim when Google can't answer. Same shape.
Different domain, different language, hand-rolled twice.

## The part that isn't a list

Ordered failover sounds trivial, and the ordering part is: it's an array. The
part that isn't trivial is that **every provider says "I'm rate limited" in a
different language**:

| Provider | How it says *rate limited* |
|:---|:---|
| One API | HTTP 429 with a `Retry-After` header |
| Another | HTTP 200, and the payload says `"rate limit exceeded"` |
| A scraper | an unparseable block page |
| A geocoder | HTTP 200, and the field you needed is just missing |

A router can't parse those; it doesn't know the domain. Your adapter can.
Translating each provider's dialect into one shared vocabulary turned out to be
the actual content of the library. Everything else follows from having it.

So the library ships **no adapters**, on purpose. An adapter encodes *your*
providers and *your* failure modes; a catalog of them would mean a dependency
per provider and a package that ages at the speed of other people's APIs. What
ships is the contract adapters implement, the router that drives them, and a
conformance test that holds them honest.

## Two ideas worth stealing even if you never use the library

**"The call returned 200" is not "the provider answered well."** A geocoder can
return a perfectly valid 200 with no latitude or longitude. A search can union
several upstream queries, lose one, and return a well-formed result that
silently omits an entire category. Both are successes by every transport
measure and useless to the caller. So results are graded `OK` or `DEGRADED`,
and a degraded answer is kept but not settled for. The router holds it while it
keeps looking for a good one, returns it only if nothing better appears, and
always labels it.

**Declining is not failing.** If a provider can't honour a constraint in the
request, the useful response is to decline rather than answer a *different
question*. A flight search that quietly drops your cabin-class filter hasn't
failed. It has changed what the price is a price *for*, and nothing downstream
can tell. That distinction has its own outcome type.

There's a third, learned the expensive way: **a spend ceiling aborts the whole
route rather than failing over.** Trying the next provider after a daily budget
trips spends more against the ceiling that just tripped. Failure that amplifies
itself is the one kind worth special-casing.

## Building it twice, on purpose

I needed it in Python (the flight tracker) and TypeScript (Showbook), so there
are two implementations. Two implementations of one contract drift. Not
maliciously, just because someone fixes an edge case in whichever language
they're in that day.

The thing that stops it is a file of **shared test vectors**: twenty routing
scenarios as data, provider behaviour in and expected decisions out, read by
*both* test suites. A behavioural change lands in both ports or CI goes red in
one.

It earned its place on the first run. The TypeScript router guarded against an
adapter whose error-classifier returns nothing; the Python one raised an
`AttributeError` from inside the router, the kind of bug that reads as a
*router* fault and sends you to the wrong file entirely. I'd written both
within an hour of each other and had no idea they disagreed.

## How it's used

**Vacation Price Tracker** routes flight searches through it. The
`flight_provider` setting stopped being "the provider" and became "the provider
we prefer": a Skiplagged 429 now moves to Kiwi automatically. Only the
overnight tracking sweep opts in, though. That runs unattended, so a failed
refresh is a missing point in a history nobody is watching. The interactive
chat search doesn't, because a user is right there and can simply ask again.
Different failure economics, different default.

That change needed a prerequisite I didn't expect. Snapshots record which
provider priced them, but the notification logic compared each new price to the
previous one *without looking at the provider*. Three sources see different
inventory, so switching between them changes the number for reasons that have
nothing to do with the market. An "alert me on any drop" rule would have
emailed about a fare that never moved, and automatic failover would have made
that happen without anyone deciding to. The comparison is now suppressed
across a provider change.

**Showbook** routes venue geocoding through it. What's left of that file is the
part that's actually about geocoding; the ordering, the failover, and the
one-call-per-second politeness Nominatim's usage policy requires all come from
the library. Two behaviours that had been implicit got names: Nominatim results
are `DEGRADED` (coordinates, but no Place ID and no photo, both of which
callers want), and the rate limit is per-provider pacing rather than a
module-level timestamp every caller shared whether or not it was about to hit
Nominatim.

Two behaviours are new rather than relocated. A repeatedly failing Google now
stops being called at all instead of being asked again every time. And a `429`
carrying `Retry-After` opens the circuit for exactly that long on the *first*
refusal. It's a public endpoint with a usage policy, and making it say no
three times before listening was never defensible.

## What I'd tell myself at the start

The interesting part of this library is not the failover. It's the vocabulary:
`OK`, `DEGRADED`, `RATE_LIMITED`, `TRANSIENT`, `TERMINAL`, `UNSUPPORTED`,
`BUDGET`. Once providers are forced to describe themselves in it, a lot of
previously invisible failure becomes something you can act on.

I found that out backwards: I set out to write a fallback list and discovered
the list was the easy half.

Related: [[building-mcp-budget-governor|MCP Budget Governor]] came out of the
same repos and the same instinct — notice the thing you've written twice, and
give it a name.
