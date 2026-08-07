---
title: Ethan's Digital Garden
---

Welcome. This is the public, curated subset of my private notes — ideas worth
sharing get promoted here once they've earned it.

It's a garden, not a blog: notes grow over time, get pruned, and are allowed
to be unfinished.

## Projects

Write-ups of things I've built: what they are, why they exist, and the parts
that were interesting to solve.

- [[building-showbook|Showbook]]. A personal live-show tracker, and the setlist-prediction
  engine underneath it.
- [[building-vacation-price-tracker|Vacation Price Tracker]]. Flight and hotel price tracking for one
  specific trip, and the problem of following a single flight through time.
- [[building-mcp-review|MCP Review]]. Commit-level code review without a pull
  request, built as an MCP host whose model gathers the project context it needs
  before writing a finding.
- [[building-mcp-budget-governor|MCP Budget Governor]]. Putting a dollar ceiling on MCP tool calls: why the
  counters have to be atomic, why two language packages share one budget rather
  than merely resembling each other, and what adopting a library in two real
  projects taught me that 98% coverage didn't. Extracted from the two apps above.
- [[building-provider-router|Provider Router]]. Calling one capability through several
  interchangeable providers: why ordered failover is the easy half, why "the call
  returned 200" is not "the provider answered well", and what building the same
  contract in two languages caught within an hour. Also extracted from the two
  apps above.
- [[building-mcp-queue-doctor|MCP Queue Doctor]]. An MCP server that diagnoses
  Postgres job queues instead of listing their rows: schema knowledge kept as
  data and matched on shape, honest silence when a backend can't answer, and
  reaching production databases you can't connect to. Extracted from Showbook's
  morning health check.
