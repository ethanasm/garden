---
title: Building Showbook
date: 2026-08-06
description: A personal tracker for live shows, and the setlist-prediction engine underneath it.
tags:
  - projects
  - typescript
  - machine-learning
---

# Building Showbook

**[Showbook](https://github.com/ethanasm/showbook)** keeps track of live
entertainment: concerts, theatre, comedy, festivals. It remembers the shows I've
been to, watches for ones I'd want to know about, and for upcoming concerts it
tries to tell me what the artist is going to play.

It's self-hosted on a machine at home. There's a web app and a native mobile
app, both feature-complete, both in daily use by an audience of roughly one.

## How it came about

I'd been logging shows in an older app for years. It worked, but it felt stuck
somewhere around 2014, and logging was all it did. Everything else about going
to shows lived in apps that had no idea the log existed: Ticketmaster for
buying, Songkick for hearing that an artist I like just announced a tour, ATG
for theatre.

Four apps, none of them talking. Ticketmaster knew what I'd bought through
Ticketmaster and nothing about the rest. Songkick knew which artists I followed
but not that I already had tickets. The log app knew where I'd been and nothing
about what was coming.

So Showbook started as a fairly boring question: what if that was one app. Parse
the ticket emails so history fills itself in, follow venues and artists so
announcements come to me, and keep past and future in the same place.

Scope did what side-project scope does. If the app knows I'm seeing an artist
next month, it can also tell me they just added a second night. And if it knows
that artist's recent setlists, which setlist.fm publishes, it can make a real
guess at what I'm about to hear. That last one turned into the most interesting
problem in the project, so most of this note is about it.

**Stack**, for skimmers: TypeScript throughout. Next.js 15 on web, Expo on
mobile, tRPC between them, Drizzle and Postgres for data, pg-boss for background
jobs, Groq for LLM calls. Nx monorepo with pnpm. Data comes from Ticketmaster,
setlist.fm, Spotify, Google Places and Wikidata.

## Predicting setlists

The obvious approach is a frequency count. Look at the artist's last twenty
setlists, rank songs by how often they show up, print the top twenty. This works
embarrassingly well for some artists and is worse than useless for others, which
is really the whole problem.

Coldplay play more or less the same show every night, so counting works. Phish
don't. They'll go years between playing a given song, so a frequency count ranks
their most-played songs highest at exactly the moment those songs are *least*
likely to turn up tonight. Cirque du Soleil run an identical scripted show every
night. A DJ set has no song-level structure to predict at all.

Before predicting anything, then, the system has to work out what kind of artist
it's looking at.

### Step one: classify the artist

A small classifier reads the artist's recent setlists and computes three numbers:

* **Mean pairwise Jaccard similarity.** Take every pair of setlists and measure
  the overlap. High means "same show every night."
* **Unique song ratio.** Distinct songs across the corpus divided by total song
  slots. High means a deep rotating repertoire.
* **Mean setlist length.** Short sets are the giveaway for DJ sets and free jazz.

Those three sort artists into four archetypes:

| Style | Signals | Example |
|:------|:--------|:--------|
| `theatrical` | Jaccard ≥ 0.95, unique ratio < 0.1 | Cirque, a residency show |
| `stable` | Jaccard ≥ 0.75, unique ratio < 0.3 | Coldplay, Tate McRae |
| `rotating` | Jaccard ≤ 0.45, unique ratio > 0.5 | Phish, Pearl Jam |
| `improvised` | Mean length < 6 | DJ sets |

Each style gets its own prediction algorithm. Under five setlists the classifier
gives up and returns `unknown`, falling back to a small hand-curated seed table,
because three data points will confidently tell you whatever you want to hear.

Classification is also sticky. One festival set can make a stable artist look
rotating for a night, so the stored style only changes after the classifier
disagrees with it three nightly runs in a row. I can also override it by hand,
and the override always wins.

### Step two: predict, for stable artists

For stable artists the model is roughly Bayesian, and most of the interesting
work is in the weighting.

Every setlist gets sorted into a tier based on two things: how far it is from
the target date, and whether it belongs to the same tour leg. The tiers carry
very different weights.

```
Tier A  same tour, within 30 days      weight 1.00
Tier B  same tour, within 180 days     weight 0.55
Tier C  same tour, older               weight 0.20
Tier D  different tour, within a year  weight 0.10
Tier E  everything else                weight 0.04
```

A song's probability is its weighted appearance count over the weighted corpus
total, smoothed with a Beta(2, 2)-style prior scaled to the corpus size, so a
song seen once in a three-setlist corpus doesn't come back at 33%.

Then a handful of corrections, all of which exist because the plain version got
something wrong:

* **The active-tour anchor.** A song that shows up in 80% or more of Tier-A
  setlists, on a leg that started within 60 days, gets floored at 0.85. Tour
  openers are near-certainties and the smoothing was underselling them.
* **The answer-key exclusion.** Any setlist on the target date itself is dropped
  from the corpus. Obvious once you see it, less obvious when you're
  back-testing and your accuracy looks suspiciously good.
* **The far-future pivot.** For a show four months out, no setlist falls within
  30 days of the target, so everything collapses to Tier E and confidence rounds
  to zero. The fix anchors tier-bucketing to the artist's most recent activity
  instead, while still measuring recency decay against the real target date. A
  show next week scores higher than one in August, which is what you want.
* **Multi-night anti-repeat.** On night three of a three-night run, songs already
  played that run get their probability cut by 95%.
* **Album drops.** When an artist releases an album they haven't toured yet,
  synthetic corpus rows seed the new tracks at a capped weight so they show up as
  plausible without drowning out real evidence.

Songs then land in four buckets: `core` at 0.65 and above, `likely` at 0.35,
`wildcards` at 0.1, `rotation` below that. Each song also gets a role (opener,
closer, encore) based on where it tends to sit, so the output reads like a
setlist instead of a ranked list.

The confidence number on top blends three things: how many recent setlists we
have, capped at six; how consistent they are with each other; and how recent the
latest one is. When it comes out low the UI explains why in plain English.
"No active tour right now" is a much better answer than an unexplained 34%.

### Step three: predict, for rotating artists

For rotating artists frequency is the wrong signal, so the model flips it around
and asks how *overdue* a song is.

For each song: time since it was last played, divided by that song's historical
mean gap. A score of 1.0 means it's due right on schedule. 3.0 means it's three
times overdue. Anything above 1.5 goes in a **Due** list, and anything above 3.0
with at least five historical plays becomes a **bust-out candidate**, the rare
one nobody's heard in years that's statistically ripe.

The output is shaped differently too. Instead of one ranked setlist it's
per-slot candidate pools (who opens, who closes the encore) with an entropy
score on each. For a Phish show, "here are the eight plausible openers" is a
more honest answer than a single guess.

### Step four: check whether any of it works

A prediction you can't score is just a vibe. So a nightly back-test replays
historical shows, generates the prediction it would have made at the time, and
scores it against what actually got played:

* **Brier score**, squared error on each song's probability, which punishes
  confident wrongness specifically.
* **Precision@10 and recall@15**, for whether the top predictions actually
  turned up.
* **A calibration curve.** Of everything predicted at around 70%, did about 70%
  play? This is the one that catches a model ranking things correctly while
  lying about how sure it is.

Then the part I like most: a release gate. The thresholds are written down
(stable-style Brier at or below 0.15, rotating recall@15 at or above 0.55, no
calibration bin off by more than 0.20), and if the latest back-test breaches one,
the matching display variant is forced off in the UI. The app refuses to show a
prediction it can't currently justify.

Every prediction served is also snapshotted, so "what did we actually show on
that date" stays answerable later, rather than only "what would we say today."

## Other bits worth mentioning

**Getting data in without typing it.** There are four ingestion paths and they
all land in the same enrichment pipeline. A Gmail scanner finds ticket
confirmations, using cheap heuristics first and only calling the LLM on what
survives, with a PDF-attachment fallback for the emails that hide everything in
the attachment. A vision model reads a festival poster or a photo of a theatre
playbill into a structured lineup or cast list. Followed artists import from
Spotify and Apple Music. Past orders import from Eventbrite. The LLM is only ever
a parser: it produces structured fields that go through exactly the same matching
code as the manual form.

**Entity resolution across five catalogs.** The same artist is a Ticketmaster
attraction, a MusicBrainz ID on setlist.fm, a Spotify artist, and sometimes a
Wikidata entity. Theatre performers are usually in none of the first three. A
nightly sweep resolves the identifiers in dependency order and handles the races
explicitly, because two jobs resolving the same performer at once is not a
hypothetical.

**The music layer.** Connect Spotify and a predicted setlist becomes a "hype"
playlist for a show you're about to see, while setlists you actually heard become
a "heard" playlist. A nightly job reads recently-played and buckets it per show,
which is how the app can tell you that a song you now love is one you first heard
live.

**The app tells me when it's broken.** A morning cron runs nine health checks
covering failed jobs, missed schedules, error volume, queue depth, data
freshness, external APIs and CI, then emails a summary. Unglamorous, and the
reason I found out about a dead vision model from an email instead of eventually
noticing that festival posters had quietly stopped returning any artists.

## Honest notes

A few things I'd tell you if you asked in an interview.

**The observability rewrite was avoidable.** Axiom caps a dataset at 256 columns
and every distinct log field becomes a column. Ad-hoc structured-log keys ate the
budget twice, and both times the fix just doubled the ceiling instead of removing
it. The real fix, folding everything outside a small allowlist into one map
field, took an afternoon and should have been the first design.

**The parity tax is real.** Every user-facing feature exists twice, once in
Next.js and once in React Native. It's the right call for something I actually
use on my phone, and it roughly doubles the cost of every feature.

**The prediction model is tuned, not learned.** Every threshold in it was set by
hand from a spec and adjusted against back-tests. That's defensible at this data
volume, and it's also the obvious place a real model would beat it.

Two pieces have since been pulled out into standalone open-source libraries.
[mcp-budget-governor](https://github.com/ethanasm/mcp-budget-governor), which
puts cost ceilings on LLM and tool calls, started here before being generalised.
The other, [provider-router](https://github.com/ethanasm/provider-router), came
out of the sibling project: [[building-vacation-price-tracker]].
