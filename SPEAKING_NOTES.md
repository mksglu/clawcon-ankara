# Speaking Notes — The Other Half of the Context Problem

**ClawCon Ankara** · April 29, 2026
**Speaker:** B. Mert Koseoğlu

---

## Slide 1 — Title

Good evening.

Thank you Veli and Furkan from BuilderMare for putting this together. Murat Aslan, ClawCon, OpenClaw community. Ankara is a good city for this conversation.

I'm Mert. I make context-mode, an open-source MCP server. Tonight I want to talk about something that's been bothering me for a while, and I think it bothers you too. You just haven't put a name on it yet.

---

## Slide 2 — Your AI agent has no memory

So here's the thing.

Your AI coding agent doesn't remember anything. I know it feels like it does. But it doesn't. Claude Code, Cursor, Copilot, Codex, Gemini CLI. Pick your favorite. Under the hood, same story. The LLM is stateless. Zero memory between API calls.

What looks like memory is actually re-transmission. Every turn, the entire conversation, every message you sent, every tool result that came back, every file the agent read, all of it gets serialized and shipped to the API again. From scratch. That's the "memory."

Let me show you what that looks like in practice.

---

## Slide 3 — Every turn, everything gets re-sent

Here's what's really happening.

Turn one. You ask something. A tool runs. Sixty kilobytes come back. Fine. No problem.

Turn five. Now the API call includes everything from turns one through four. Plus the new stuff. Three hundred kilobytes. You're paying for those first four turns again.

Turn fifteen. Same deal. Six hundred kilobytes. Every tool result from every previous turn is still in that payload.

Turn thirty. 1.2 megabytes. One point two megabytes sent to the API on a single turn. Think about that for a second.

Turn thirty-one. Context window is full. The agent panics. It uses the model to summarize the conversation, compress it down, then it throws away the originals. Everything the agent read, every decision it made, every error it worked through, the thing it was building. Gone. Replaced by a lossy summary.

And you start explaining your codebase again from scratch.

---

## Slide 4 — One command. 750,000 tokens.

Let me put a real number on this.

You run `gh issue list`. One command. Fifty-nine kilobytes of JSON comes back from GitHub. About fifteen thousand tokens. That seems fine.

But nobody talks about what happens next. Those fifteen thousand tokens are now part of the conversation history. Next turn, they ship again. Turn after that, again. The agent isn't even looking at that data anymore. It already processed it. Gave you the answer. But the protocol requires full conversation on every API call. So it keeps shipping.

Fifty turns in. That one command has cost you 750,000 input tokens.

One command. 750,000 tokens. Because the output sat in conversation history getting re-sent over and over.

And nobody runs just one command per session. You're running twenty, thirty, sometimes fifty tool calls. Average output around thirty kilobytes each.

That's thirty megabytes of input tokens. Per session. Just tool output. Not your prompts. Not the model's replies. Just raw output from tools the agent already processed and will never look at again.

That's where your context is going. And yeah, that's where your money is going too.

---

## Slide 5 — When the context fills up, everything disappears

So the context fills up. What happens next?

The agent compacts. Summarizes the conversation. But summarization is lossy. The NoLiMa benchmark from 2025 tested twelve models. Found fifty percent accuracy drop past thirty-two thousand tokens. The model is already making worse decisions before it even compacts. Just from the noise.

After compaction it gets worse. Files the agent read, the specific lines it looked at, the patterns it spotted, all of that becomes a one-line summary. Decisions it made, the whole reasoning chain that got it to an architectural choice, flattened. Errors it found and fixed, the stack trace, the root cause, summarized into nothing. Your corrections, the thing you spent five minutes explaining about how your project works, lost.

This happens every twenty to thirty minutes in a real session. Every time, you lose fifteen minutes re-explaining everything.

For a fifty-person team at a hundred bucks per seat per month, that adds up to sixty thousand a year. Spent on an agent that keeps forgetting who you are.

---

## Slide 6 — What if the data never entered context?

*(two seconds of silence)*

What if the data never entered the context window in the first place?

Not better compression. Not smarter summarization. Not a bigger window. What if those fifty-nine kilobytes from `gh issue list` simply never became part of the conversation?

---

## Slide 7 — Intercept

context-mode intercepts it.

Technically, it's an MCP server. Sits between the agent and its tools. Every tool call goes through a routing engine. The key design choice: it hooks into the agent's lifecycle, not the tools themselves.

There are five hooks. Each one does one thing.

PreToolUse fires before a tool runs. This is where dangerous stuff gets caught. `curl`, `wget`, inline HTTP, `rm -rf`. Blocked or redirected to a sandbox. Commands that would produce large output get flagged.

PostToolUse fires after a tool returns. This is the main interception point. Your fifty-nine kilobytes of JSON goes into a local FTS5 database. Full-text search with BM25 ranking. The agent gets back a 1.1 kilobyte summary. Same information. Ninety-eight percent fewer tokens.

And this isn't truncation. The full data is still there, still searchable. The agent can query it anytime with `ctx_search`. It just doesn't sit in the conversation getting re-sent fifty times.

Now, how does one plugin work on fourteen different platforms? Three mechanisms. Claude Code and Codex have native hook support, so routing is automatic. The hooks fire on every tool call, the agent doesn't need to change anything. Cursor and Copilot have MCP support but no hooks, so context-mode runs as an MCP server exposing tools like `ctx_execute` and `ctx_search`. The agent learns to use them. Newer CLIs with neither get routing rules injected into the system prompt. Three approaches, one plugin, same outcome.

---

## Slide 8 — Sandbox (Think in Code)

Sandbox. This took the longest to get right.

We call it Think in Code. Here's the problem it solves.

Even with interception handling tool output, the agent still wants to pull data into context to "think about it." It reads forty-seven files with `Read()` to understand a codebase. Seven hundred kilobytes in context. Runs `cat access.log` to find errors. Forty-five kilobytes.

Think in Code flips that. Instead of pulling data in for the model to read, you send code to the data. The agent writes a small script. context-mode runs it in a sandbox. Only stdout comes back to context.

Real example. Access logs. Before: `cat access.log`, forty-five kilobytes enters context, stays there forever, re-sent every turn. After: `ctx_execute("javascript", ...)`, agent writes five lines to filter 500-errors and count them. 155 bytes of stdout enters context. The log file never touches the conversation.

Two hundred times smaller. And I'd argue the agent does better work this way. Writing real analysis code is just more reliable than asking a language model to eyeball a log file.

Since version 1.0.64, Think in Code is mandatory across all fourteen platform configs. Not a suggestion. Enforced by routing rules. If the agent tries to `Read()` a big file for analysis, PreToolUse redirects it to `ctx_execute_file`. The file loads into a `FILE_CONTENT` variable inside the sandbox. Agent writes code against it. Only stdout returns.

Your CPU does the work for free. Tokens cost money.

---

## Slide 9 — Index (Session persistence)

Index.

Everything the agent has touched goes into a local SQLite database. FTS5 virtual table, BM25 ranking. Tool outputs, web pages that got fetched and chunked, sandbox results. All full-text searchable. The agent runs `ctx_search`, gets ranked results. Like a local search engine for everything the agent has seen this session.

But here's the part that actually matters. Session persistence.

PostToolUse doesn't just capture tool output. It captures events. Fifteen categories, in real time. File operations, git commands, errors and how they got fixed, your corrections, environment details, project rules, decision points. All of it goes to a SessionDB, a separate SQLite store.

When context fills up and the agent compacts, normally that's it. Everything is gone. But context-mode has a PreCompact hook. Right before the wipe, it builds a structured snapshot of the session. Not a summary. Structured references to everything in the knowledge base.

Then SessionStart fires on the next turn. Restores the snapshot. Reloads indexed knowledge. The agent picks up where it was, mid-thought.

Twenty-six event categories survive each compaction and `--resume`. Your prompts, tracked files, project rules, decisions, git operations, errors and fixes, environment, session mode, tool patterns.

You stop explaining your codebase from scratch. The agent already knows. It just looks it up.

---

## Slide 10 — Measured results

Numbers.

Playwright browser snapshot: fifty-six kilobytes of raw DOM. With context-mode, 299 bytes. The relevant elements, search-ranked. 99.5% reduction.

GitHub Issues: fifty-nine kilobytes of JSON. With context-mode, 1.1 kilobytes. The issues matching the agent's actual question. 98% reduction.

Access log analysis: forty-five kilobytes of logs. With Think in Code, 155 bytes of stdout. 99.7% reduction.

Full session, fifty turns. Thirty megabytes re-sent drops to one. On top of that, context-mode enforces output compression. The agent cuts filler, pleasantries, over-explanation. 65 to 75 percent output token savings stacked on input reduction.

98 to 99.5 percent less context used. The agent does the same work, gets the same answers. But it can go longer without hitting the wall. And it makes better decisions because the window isn't stuffed with stale output from thirty turns ago.

---

## Slide 11 — Close

context-mode. Open source. Elastic License 2.0.

Over 95,000 users. 79,000 npm downloads. Fourteen platform adapters. 10,000 GitHub stars. Teams at Microsoft, Google, Meta, ByteDance, GitHub, Stripe, Datadog, NVIDIA, Supabase use it.

Fourteen adapters. Claude Code, Cursor, Codex CLI, VS Code Copilot, JetBrains Copilot, Gemini CLI, Qwen Code, Kiro, OpenCode, KiloCode, Zed, Pi, Antigravity. And a native OpenClaw adapter. Which feels right to mention here tonight.

Why open source? Because honestly, I don't see how this problem gets solved by the companies selling you tokens. They have zero incentive to help you use fewer of them. That's their business model. So it has to come from us.

context-mode.com.

Thank you Veli, Furkan, Murat, and ClawCon.
