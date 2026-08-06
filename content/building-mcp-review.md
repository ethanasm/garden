---
title: MCP Review
date: 2026-08-05
description: A commit-level code-review CLI, and what I learned from making an LLM gather its own project context through MCP.
tags:
  - projects
  - mcp
  - typescript
  - developer-tools
---

# MCP Review

A code-review CLI for the workflow where there is no pull request. Point it at
a commit, a range, or the staged index and it gives the change a second pass
before or after it lands.

The useful part is not that an LLM can read a diff. It is that the reviewer can
inspect the repository around the diff — call sites, tests, lint configuration,
and the patterns the project already uses — before deciding what is actually a
problem.

- Source: [github.com/ethanasm/mcp-review](https://github.com/ethanasm/mcp-review)

## Why review commits instead of pull requests

Most automated review products attach themselves to a pull request. That makes
sense for a team, but it leaves an awkward gap for a solo project where the
normal path is a small commit straight to `main`. Opening a branch and a pull
request just to ask for a second set of eyes adds more ceremony than the review
is worth, so it gets skipped.

MCP Review starts from Git instead of from a hosting platform. The default is
the last commit, but it also understands a single commit, an explicit revision
range, the last N commits, everything since a date, and staged changes. It works
against local history, needs no GitHub token, and can run before the commit
exists.

That also makes it useful as a plain Unix-ish tool. Human-readable output is for
the terminal; JSON output and meaningful exit codes are for scripts and CI.
Watch mode can review each new commit as it appears. A content hash caches the
result, so asking the same question about the same diff does not buy the same
answer twice.

## A diff is evidence, not context

The first version did what most small review tools do: put the diff in a prompt
and ask for findings. It produced the familiar genre of technically reasonable,
mostly generic advice. The model could see that a function had changed, but not
that every neighboring module used a different error pattern, or that the type
was exported into three callers, or that a matching test file already showed
the intended convention.

Stuffing the whole repository into every prompt is the opposite failure. It is
expensive, slow, and makes the model search a pile of irrelevant text for the
few relationships that matter.

The design I settled on is to give the model the diff plus a small set of
read-only tools and let it ask for the context it needs. The system prompt is
deliberately specific: findings should cite files and lines, and a convention
claim should point to where the project already follows that convention. "This
is usually a best practice" is not enough.

## MCP on the other side

Most of my MCP projects are servers. This one is a **host**.

The CLI starts four MCP servers as child processes over stdio:

- **Git diff** supplies diffs, statistics, commit messages, and blame.
- **File context** reads files and line ranges and can return a file together
  with its exports and importers.
- **Conventions** inspects lint configuration, searches for comparable patterns,
  and reads project instructions.
- **Related files** traces imports, exports, type references, and likely tests.

At startup the host discovers each server's tools with `tools/list`, registers
which process owns which tool, and routes the model's `tools/call` requests back
to it. The conversation manager does not need to know how a tool is implemented,
and a server does not need to know anything about the review loop. The longer
version is in the repo's
[host architecture document](https://github.com/ethanasm/mcp-review/blob/main/MCP_HOST_ARCHITECTURE.md).

Separate processes are more machinery than four imported functions. That is
intentional here, partly because the project is a working exercise in the host
half of MCP, but the boundary also stays honest: the tools expose capabilities,
the host discovers and coordinates them, and the model decides which ones to
use. A fifth server can join without teaching the other four about it.

## Keeping the reviewer bounded

An agent that can browse a repository can also browse forever. The guardrails
are mostly boring, which is exactly what I want from guardrails:

- Tools are read-only. Reviewing code does not grant permission to change it.
- Changed files can be preloaded, but each file and the total diff have size
  limits; oversized diffs are truncated at file boundaries rather than halfway
  through an arbitrary hunk.
- Tool calls run in parallel and the model gets at most two rounds. The prompt
  pushes it to batch related questions rather than discover one file at a time.
- Ignore patterns, file-count limits, and explicit focus areas bound the review
  further.
- Tool results are cached within a session. Asking twice for the same file or
  import graph does not repeat the filesystem or process round trip.
- The final answer has a schema: critical findings, suggestions, positive notes,
  and a confidence level. Fenced JSON is parsed; an unparsable response degrades
  to a low-confidence result instead of crashing the renderer.

There is also a provider boundary underneath the conversation. Anthropic is the
default, while an OpenAI-compatible implementation supports OpenRouter,
DeepSeek, Kimi, and other compatible endpoints. Model aliases are configuration,
not branches through the review code.

## The performance work changed my mind

The early host was slow for reasons that looked obvious: four child processes,
several tools, several model turns. So I made the mechanical path cheap. Servers
start in parallel and execute their compiled entry points directly instead of
paying an `npx` startup on every review. Tool calls within a round are parallel.
The diff is computed once. Predictable file reads can be preloaded. A composite
file-context tool replaces three small calls. Timers report API time, tool time,
and overhead separately. The full list is recorded in the
[performance ADR](https://github.com/ethanasm/mcp-review/blob/main/docs/adr/001-performance-tuning.md).

Then I measured a more ambitious design: a cheap model gathers context, hands a
flattened evidence packet to a better model, and the second model writes the
review. Architecturally it was cleaner, and it is documented in the
[gather/analyze ADR](https://github.com/ethanasm/mcp-review/blob/main/docs/adr/002-multi-model-gather-analyze.md).
It was also slower.

Across five commits and three uncached runs each, the single-model version
averaged about **16.6 seconds of measured review time**. The first two-model
version averaged **21.6 seconds**. After changing the gather prompt so the model
batched 6–15 tools in its first round and usually stopped on its own, the result
rose to about **24.0 seconds**, 45% slower than the single-model baseline. One
run produced a 50,000-token handoff and took 162 seconds.

The timers made the reason unambiguous: tools were generally under 300ms in
total. API calls consumed effectively all measured wall time. Optimizing file
reads or process IPC further would have polished the part that was already
close to free. The expensive step was serializing all gathered context into a
new prompt and asking a second model to understand it from scratch.

So the two-phase branch did not ship. Separation of concerns is not automatically
a user benefit, and an architecture diagram is not a benchmark. The experiment
did leave behind better batching instructions, instrumentation, and a useful
rule for this kind of agent: measure model latency and context growth before
optimizing the tools around them.

## Honest notes

This is still a young tool. It is source-install software rather than a polished
registry release, and the package is still internally named
`mcp-git-reviewer` while the command and repository are `mcp-review`. The README
documents `npm link`, which is fine for my machines and not a distribution
story.

The benchmark also found a real prompt-construction bug: when focus areas are
configured, that path bypasses the preloaded-file section. The review still
works because the model can fetch the files through tools, but the code and the
performance claim disagree. Finding that was another reminder that an
optimization needs a test that proves the optimized path actually runs, not
only a benchmark around the whole program.

And context-aware does not mean correct. The model can fetch an importer and
still misunderstand it; a high-confidence structured finding is not proof. The
tool is a second pass, not an approval gate, and its best output is a precise
claim with enough local evidence that I can verify it quickly.

## Related projects

The other public write-up about MCP infrastructure is
[[building-mcp-budget-governor|MCP Budget Governor]], a cross-language spending
guard extracted from [[building-vacation-price-tracker|Vacation Price Tracker]].
That app's provider failover later became
[provider-router](https://github.com/ethanasm/provider-router).

[[building-showbook|Showbook]] produced a different kind of agent tool: its
production queue-health heuristics became
[mcp-queue-doctor](https://github.com/ethanasm/mcp-queue-doctor), an MCP server
that explains retry storms, stuck jobs, and missed schedules instead of merely
listing queue rows.
