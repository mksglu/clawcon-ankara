# Speaking Notes — The Other Half of the Context Problem

**ClawCon Istanbul** · May 6, 2026
**Speaker:** B. Mert Koseoğlu

---

## Slide 1 — Title

Hey everyone, good evening.

Before anything — thank you Veli, Furkan, Murat, ClawCon, OpenClaw. Really happy to be here in Istanbul tonight.

So. I'm Mert. I build context-mode, it's an open-source MCP server. And tonight I want to talk about something that's been bugging me. I think it bugs you too — you just haven't named it yet.

---

## Slide 2 — Your AI agent has no memory

OK so here's the thing.

Your AI agent? It doesn't remember anything. I know it feels like it does. But it doesn't. Claude Code, Cursor, Copilot, Codex, Gemini CLI — doesn't matter which one you use. Same story under the hood. The model is stateless. Zero memory between turns.

What looks like memory? That's just re-sending. Every single turn, the whole conversation — every message, every tool result, every file it read — gets packed up and sent to the API again. From scratch. That's the "memory."

Let me show you what that actually looks like.

---

## Slide 3 — Every turn, everything gets re-sent

So here's what's really going on.

Turn one. You ask something. A tool runs. 60 KB comes back. Fine. No big deal.

Turn five. Now the API call has everything from turns one through four, plus the new stuff. 300 KB. You're paying for those first four turns all over again.

Turn fifteen. Same thing. 600 KB. All those old tool results? Still in there.

Turn thirty. 1.2 megabytes. On a single API call.

And then turn thirty-one. Context window is full. The agent panics. It tries to summarize the whole conversation, compress it down, and then it throws away the originals. Everything it read, every decision it made, every error it fixed, the thing it was halfway through building — gone. Just a lossy summary left.

And now you're explaining your codebase again. From scratch.

---

## Slide 4 — One command. 750,000 tokens.

Let me make this real.

You run `gh issue list`. One command. 59 KB of JSON comes back. About 15,000 tokens. That's fine, right?

But here's what nobody talks about. Those 15,000 tokens? They're now part of the conversation. Next turn, they ship again. Turn after that, again. The agent isn't even looking at that data anymore. It already gave you the answer. But the protocol says: full conversation, every API call. So it keeps shipping.

Fifty turns in? That one command has cost you 750,000 tokens.

One command. 750,000 tokens.

And nobody runs just one command. You're running twenty, thirty commands in a session. Average output, about 30 KB each.

That's 30 megabytes of input tokens. Per session. Just tool output. Not your prompts, not the model's replies — just raw output from tools the agent already processed and will never look at again.

That's where your context is going. And yeah — that's where your money goes too.

---

## Slide 5 — When the context fills up, everything disappears

So the context fills up. What now?

The agent compacts. Summarizes. But summarization is lossy. The NoLiMa benchmark tested twelve models — 50% accuracy drop past 32K tokens. The model is already making worse decisions before it even compacts. Just from all the noise.

And after compaction? It gets worse. Files it read — now a one-line summary. Decisions it made — flattened. Errors it found and fixed — gone. That thing you spent five minutes explaining about how your project works — lost.

This happens every twenty to thirty minutes in a real session. Every time, you lose about fifteen minutes re-explaining everything.

For a fifty-person team at a hundred bucks per seat per month, that's 60K a year. On an agent that keeps forgetting who you are.

---

## Slide 6 — What if the data never entered context?

*(let it sit for a moment)*

What if the data never entered the context window in the first place?

Not better compression. Not smarter summarization. Not a bigger window. What if those 59 KB from `gh issue list` just... never became part of the conversation?

---

## Slide 7 — Intercept

That's what context-mode does. It intercepts.

It's an MCP server. Sits between the agent and its tools. Every tool call goes through a routing engine. The key thing — it hooks into the agent's lifecycle, not the tools themselves.

There are five hooks. Each one does one thing.

PreToolUse — fires before a tool runs. Catches dangerous stuff. `curl`, `wget`, `rm -rf`. Blocks it or redirects it to a sandbox.

PostToolUse — fires after a tool returns. This is the big one. Your 59 KB of JSON goes into a local FTS5 database. Full-text search, BM25 ranking. The agent gets back a 1.1 KB summary. Same info. 98% fewer tokens.

And this isn't truncation. The full data is still there. Still searchable. The agent just queries it with `ctx_search` when it needs it. It just doesn't sit in the conversation getting re-sent fifty times.

Now — how does one plugin work on fourteen platforms? Three ways. Claude Code and Codex have native hooks, so routing is automatic. Cursor and Copilot have MCP but no hooks, so context-mode runs as an MCP server with tools like `ctx_execute` and `ctx_search`. And newer CLIs with neither get routing rules injected into the system prompt. Three approaches, same outcome.

---

## Slide 8 — Sandbox (Think in Code)

Sandbox. This took the longest to get right.

We call it Think in Code. Here's the problem.

Even with interception, the agent still wants to pull data into context to "think about it." It reads 47 files to understand a codebase. 700 KB. Runs `cat access.log` to find errors. 45 KB. All of that just sitting in context.

Think in Code flips that. Instead of pulling data in for the model to read, you send code to the data. The agent writes a small script. context-mode runs it in a sandbox. Only the stdout comes back.

Real example. Access logs. Before: `cat access.log` — 45 KB enters context, stays there forever. After: `ctx_execute`, agent writes five lines to filter 500 errors and count them. 155 bytes of stdout. The log file never touches the conversation.

200 times smaller. And honestly? I think the agent does better work this way. Writing actual analysis code is more reliable than asking a language model to eyeball a log file.

Since version 1.0.64, this is mandatory. All fourteen platforms. Not a suggestion — enforced by routing rules. If the agent tries to `Read()` a big file for analysis, PreToolUse redirects it to the sandbox.

Your CPU does the work for free. Tokens cost money.

---

## Slide 9 — Index (Unified Persistent Memory)

Index.

Everything the agent touches goes into a local SQLite database. FTS5, BM25 ranking. Tool outputs, web pages, sandbox results. All searchable.

But here's what changed. Since v1.0.100, `ctx_search` has a timeline mode. It searches across three sources at once. Current session — tool outputs, indexed content. Prior sessions — events from the last seven days, stored in a separate SessionDB. And auto-memory — persistent preferences in markdown files. One query, three layers deep.

And when context fills up and the agent compacts — it doesn't just save data. It auto-injects behavioral directives. Your active role, key decisions, loaded skills. Not a summary the model ignores. Actual behavioral rules. 500 token cap, fires only on compaction.

26 event categories survive each compaction. Your prompts, tracked files, project rules, decisions, git operations, errors, constraints, rejected approaches — all of it carries over. Not just within a session. Across sessions.

You stop re-explaining. The agent already knows. It just looks it up.

---

## Slide 10 — Measured results

OK. Numbers.

Playwright browser snapshot. 56 KB of raw DOM. With context-mode? 299 bytes. 99.5% reduction.

GitHub Issues. 59 KB. With context-mode? 1.1 KB. 98%.

Access log analysis. 45 KB. With Think in Code? 155 bytes. 99.7%.

Full session, fifty turns. 30 megabytes re-sent drops to one. Plus output compression on top — 65 to 75% output token savings stacked on input reduction.

98 to 99.5% less context. Same work. Same answers. But the agent goes longer without hitting the wall. And it makes better decisions because the window isn't stuffed with stale output from thirty turns ago.

---

## Slide 11 — Close

context-mode. Open source. Always has been.

125,000 users. 104,000 npm downloads. 14 platforms. 12,500 GitHub stars. Teams at Microsoft, Google, Meta, ByteDance, GitHub, Stripe, Datadog, NVIDIA, Supabase use it.

14 adapters — Claude Code, Cursor, Codex, Gemini, Kiro, Cline, and more. Including a native OpenClaw adapter. Which feels right to mention here tonight.

Why open source? Because honestly — I don't see how this problem gets solved by the companies selling you tokens. They have zero reason to help you use fewer. That's their business model. So it has to come from us.

context-mode.com.

Thank you Veli, Furkan, Murat, and ClawCon. Thanks everyone.
