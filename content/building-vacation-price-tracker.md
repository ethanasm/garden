---
title: Vacation Price Tracker
date: 2026-08-06
description: The Google Flights price graph I wanted, tracking specific options rather than the market minimum.
tags:
  - projects
  - python
  - distributed-systems
---

# Vacation Price Tracker

A price tracker for one specific trip: these airports, these dates, this hotel
city. It tells you when the total drops below a number you set, and you can also
just talk to it, since there's a chat interface that creates and manages trips
through tool calls. Web app, native mobile app, and a backend that wakes up daily
to go and check.

- Source: [github.com/ethanasm/vacation-price-tracker](https://github.com/ethanasm/vacation-price-tracker)

## How it came about

Google Flights already tracks prices. You pick a route and dates, it draws you a
graph, it emails you when things move. I used it constantly.

The catch is what that graph is measuring. It's a route-level number: the
cheapest anyone can fly SFO to Boston on those dates. That line moves for
reasons that have nothing to do with the trip I'm planning. Today's cheapest is a
6am connection through Denver, last week's was a nonstop at a civilised hour, and
watching the two of them swap places tells you about the market rather than about
your trip.

What I wanted was to pick the flight I'd actually take and watch *that* price.
Same for the hotel, and then the combined total, because that's the number the
decision actually turns on.

So this version stores the whole search. Every check writes a snapshot of a few
hundred offers with their prices, not just the day's minimum. Once you have that,
you can follow any individual option backwards through time and ask questions
Google's graph can't answer. Basically the Google Flights I wish they'd built.

It was also a good excuse to build something real on Temporal, which I'd been
wanting a reason to use.

**Stack**: FastAPI on Python 3.12, with SQLModel and Postgres owning the data.
Temporal for the scheduled price-check workflows, Redis for idempotency and rate
limiting, Next.js 16 on the web, Expo for iOS and Android, Groq for the
assistant. Self-hosted prod stack behind a Cloudflare Tunnel, web front end on
Vercel.

## Following one flight through time

This sounds trivial and isn't.

To draw "cheapest option each day" you take the minimum of each snapshot and
you're done. But that line has the same problem as the Google graph. It's not
tracking a price, it's tracking a shifting definition of which flight counts.

To track a specific flight instead, you need to know that the flight in today's
results and the flight in yesterday's results are the same flight. They don't
come with a stable identifier. Provider offer IDs regenerate on every response,
sometimes with the price encoded in them, and they're useless across time.

### Building an identity that survives

So flights get a synthetic key built from the things that genuinely don't change
about a scheduled flight: carrier, flight number, the airports, the date. One
component per segment, concatenated across the whole itinerary.

```
UA-100|SFO-DEN|2026-09-14+UA-455|DEN-BOS|2026-09-14
```

A connection through a different hub produces a different key, which is right.
It's a different trip. Hotels get a normalised-name key, since they don't have
flight numbers and the name is the only thing that reliably persists.

That handles matching. Then reality complicates it three times.

**There's more than one snapshot a day.** The daily cron runs, and then you hit
refresh twice because you're impatient. If you only match your selected flight
against the day's cheapest-total snapshot, you can miss the moment where *your*
flight was quoted lower in one of the others. So the selected offer gets matched
across every snapshot that day, cheapest match winning, independently of which
snapshot won on total.

**Your flight disappears.** Some days the search just doesn't return it, because
it sold out in that fare class, or the provider paginated it away, or the
provider was having a bad afternoon. Plotting zero is wrong. Leaving a hole in
the line is also wrong. It carries the last known price forward instead, so the
line stays continuous and honest about being flat.

**Snapshots can come back empty.** A provider can return a technically
successful response where nothing is priced. Sum that naively and you get a $0
total, which as a chart point looks like the deal of the century. Those snapshots
get detected and excluded, so they neither plot a fake point nor displace a
genuinely priced snapshot as the day's best.

The result is what I wanted in the first place. Pick a flight and a hotel and the
chart shows three lines: the market minimum, your specific selection, and the
combined total, one point per day.

## Three providers, one contract

Flights come from one of three sources: the Skiplagged MCP server, the Kiwi.com
MCP server, or a Google Flights scraper. They're switchable at runtime from a
settings toggle with no redeploy. That flexibility exists for an unglamorous
reason, which is that the original provider started returning sustained 429s
mid-project and the app needed somewhere else to go.

The subtle problem with swappable providers is that they're compatible on their
outputs but not their inputs, and the gaps are silent. Pass a cabin-class
constraint to a provider that doesn't support it and you don't get an error. You
get a price. Just a price for a different question than the one you asked.

So capabilities live in a declared table rather than being re-derived at each
call site, and any constraint the active provider can't honour gets logged as a
dropped-constraint warning instead of vanishing.

There's a lesson buried in that table that cost me real data. I filled in one
provider's capabilities by reading *our client code*. Our client didn't send a
cabin parameter, so I wrote down `cabin: false`. The provider had supported it
all along, under a different parameter name. Every tracked trip was being priced
in economy no matter what the user picked, and nothing anywhere errored. The
note in the codebase now says that a capability describes the provider, not your
adapter, and to go read the provider's own schema. I made that same mistake three
times before writing it down.

The routing layer has since been extracted into a standalone library,
[provider-router](https://github.com/ethanasm/provider-router), which does
ordered failover across interchangeable providers with normalised output.

## Durable workflows, and failing halfway

Each daily price check is a Temporal workflow: load the trip, fetch flights and
hotels in parallel, filter by the user's preferences, save a snapshot, evaluate
the notification thresholds. Temporal handles retries, timeouts, and the "server
rebooted mid-run" case, which is the actual reason to reach for it. The home
server does reboot.

The design decision I like most here is what happens on partial failure. The two
fetches run concurrently with exceptions collected rather than thrown, so a
hotel-search outage doesn't throw away a perfectly good flight result. The
snapshot gets written with whatever succeeded, plus a marker recording what
didn't, and a later retry is allowed to supersede a degraded snapshot while
leaving a successful one alone.

The general principle: partial data beats no data, as long as you record which
part is missing. A gap in a price history costs you a lot more than a labelled
partial reading.

## The assistant is a client, not a feature

The chat interface talks to Groq with a set of tools for creating a trip, listing
trips, adjusting a threshold, pausing tracking, searching flights. The part that
matters architecturally is that those tools call the same service layer the REST
API does. The assistant is another client of the application rather than a
parallel implementation of it, so there's no drift between what you can do by
talking and what you can do by tapping.

Trip creation requires an idempotency key held in Redis, which matters more than
usual with an LLM in the loop. A retried tool call shouldn't leave you with two
identical trips.

## Keeping it from getting expensive

LLM tokens and third-party API calls both cost money, and the failure mode of an
agentic loop is a great many calls very quickly. So there are always-on ceilings:
per-user daily quotas plus a global daily spend circuit breaker, backed by atomic
Redis counters that reset at UTC midnight.

That's now its own open-source library, [[mcp-budget-governor|MCP Budget Governor]], and this app is
where it came from. Extracting it turned up a production bug that had been
invisible while it sat inside the app. A user at their chat quota was draining the overall API quota on
rejected retries, because rejections were being metered as usage. Pulling a
pattern out of the app that grew it, and having to state its rules precisely
enough for strangers, turns out to be a good way to find out what it was really
doing.

## Honest notes

**Post-fetch filtering that didn't need to be.** Airline preferences get filtered
in memory after fetching, on the assumption that providers couldn't do it
server-side. Two of them can. It works correctly, it just fetches and discards
results the API would happily have excluded. Same root cause as the cabin bug.

**Room-type filtering genuinely can't move.** Hotel search takes city, dates and
occupancy and nothing else, so "king bed, ocean view" really does have to happen
after the fetch. Worth telling apart the constraint you're stuck with from the
one you assumed.

**95% coverage on the Python side is a strong gate and I'd keep it.** It's the
reason I can swap a data provider at runtime without much anxiety.

The sibling project, and the one with the harder algorithm in it:
[[building-showbook|Showbook]].
