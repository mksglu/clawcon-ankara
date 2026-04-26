# Speaking Notes — The Other Half of the Context Problem

**ClawCon Ankara** · April 29, 2026
**Speaker:** B. Mert Koseoğlu

---

## Slide 1 — Title

> Good evening everyone.

Quick thank you to Veli and Furkan from BuilderMare, to Murat Aslan, and to ClawCon and the OpenClaw community for making this happen in Ankara.

*(pause)*

I'm Mert. I build context-mode — an open-source MCP server for AI coding agents. Tonight I want to talk about a problem that costs real money and real time, and that every single person in this room is living with right now. You just might not have a name for it yet.

---

## Slide 2 — Your AI agent has no memory

Let's start with a fact that might surprise you.

*(pause)*

Your AI coding agent has no memory. None. Claude Code, Cursor, Copilot, Codex, Gemini CLI — under the hood, they all work exactly the same way. The LLM is stateless. There is no persistent state between API calls. It doesn't remember anything from one turn to the next.

So how does it seem like it remembers? It doesn't. Every single turn, the full conversation — every message, every tool result, every file it read — gets serialized and re-sent to the API from scratch. The illusion of memory is just re-transmission.

Let me show you what that actually looks like.

---

## Slide 3 — Every turn, everything gets re-sent

Here's what actually happens under the hood.

*(pause)*

Turn one — you ask something, a tool runs, sixty kilobytes come back. That's fine. Fits in context easily.

Turn five — the API call now includes all the output from turns one through four, plus the new stuff. Three hundred kilobytes of input tokens. You're paying for turns one through four again.

Turn fifteen — everything ships again. Six hundred kilobytes. Every tool result from every previous turn is in that payload.

Turn thirty — you're at 1.2 megabytes. The context window is almost full. Think about that — 1.2 megabytes of input tokens sent to the API on a single turn.

*(pause)*

Turn thirty-one? The context window hits its limit. The agent compacts. It uses the model itself to summarize the conversation into a shorter form, then throws away the originals. And just like that — every file it read, every decision it made, every error it encountered and fixed, the task it was in the middle of — gone. Replaced by a lossy summary.

You start over.

---

## Slide 4 — One command. 750,000 tokens.

Let me make this concrete with a real example.

*(pause)*

You run `gh issue list`. One command. GitHub returns fifty-nine kilobytes of JSON. That's about fifteen thousand tokens. Seems reasonable, right?

*(pause)*

But here's the thing nobody talks about. Next turn, those fifteen thousand tokens ship again as part of the conversation history. Turn after that — again. Turn after that — again. The agent isn't using that data anymore. It already processed it. But the protocol requires the full conversation on every API call.

Fifty turns in, that single command has cost you **seven hundred fifty thousand input tokens**. One command. Seven hundred fifty thousand tokens. Just because the output sat in the conversation history and got re-sent on every subsequent turn.

*(pause)*

And you're not running one command per session. You're running twenty, thirty, sometimes fifty tool calls. Average output per call: about thirty kilobytes each.

That's thirty megabytes of input tokens per session. On tool output alone. Not your prompts. Not the model's responses. Just the raw output from tools that the agent already processed and will never look at again.

That's where your context goes. That's where your money goes.

---

## Slide 5 — When the context fills up, everything disappears

And when it fills up, everything disappears.

*(pause)*

The agent compacts. It uses the model to summarize the conversation into a compressed form. But summarization is lossy. The NoLiMa benchmark from 2025 tested twelve major models and found a **fifty percent accuracy drop** once you cross thirty-two thousand tokens. The model literally makes worse decisions as context fills up — before it even compacts.

And after compaction? Files it read — gone. The specific lines it looked at, the patterns it noticed — replaced by a one-line summary. Decisions it made — gone. The reasoning chain that led to an architectural choice — flattened. Errors it encountered and fixed — gone. The exact stack trace, the root cause analysis — summarized away. Your corrections — gone. That thing you spent five minutes explaining about your project's conventions — lost.

*(pause)*

Fifteen minutes lost every time this happens, re-explaining your codebase from scratch. And it happens every twenty to thirty minutes in a real coding session.

For a fifty-seat engineering team at a hundred dollars per seat per month — that's sixty thousand dollars a year burned on an agent that forgets everything and makes you repeat yourself.

---

## Slide 6 — What if the data never entered context?

*(let it sit for two seconds)*

What if the data never entered the context window in the first place?

*(pause)*

Not compression. Not better summarization. Not a bigger context window. What if the fifty-nine kilobytes from `gh issue list` simply never became part of the conversation history at all?

---

## Slide 7 — Intercept

context-mode intercepts.

*(pause)*

It's an MCP server — Model Context Protocol — that sits between your agent and its tools. Every tool call passes through a routing engine. And this is the key architectural decision: it hooks into the agent's own lifecycle, not the tools themselves.

Five hooks. Each one does something specific:

**PreToolUse** fires before a tool executes. This is where dangerous commands get blocked — `curl`, `wget`, inline HTTP, `rm -rf`. They get redirected to a sandbox or denied entirely. Large output commands get flagged for interception.

**PostToolUse** fires after a tool returns. This is the main interception point. The raw output — your fifty-nine kilobytes of JSON — goes into a local FTS5 database. Full-text search with BM25 ranking. The agent gets back a one-point-one kilobyte summary instead. Same answer. Ninety-eight percent fewer tokens.

*(pause)*

The critical thing: this isn't filtering or truncating. The full data is preserved and searchable. The agent can query it anytime with `ctx_search`. It just doesn't sit in the conversation history getting re-sent fifty times.

On platforms with native hook support — Claude Code, Codex CLI — the routing is automatic. The hooks fire on every tool call without the agent having to do anything different. On platforms without hooks — Cursor, Copilot, Gemini CLI — context-mode injects routing rules into the system prompt. The agent learns to call `ctx_execute` and `ctx_search` instead of raw tool calls. Same result, different mechanism. That's how one plugin works across fourteen platforms.

---

## Slide 8 — Sandbox (Think in Code)

Sandbox.

*(pause)*

This is the paradigm shift, and it's the part that took the longest to get right. We call it **Think in Code**.

Here's the problem it solves. Even with interception, the agent still has a bad habit: it wants to pull data into context to analyze it. It calls `Read()` on forty-seven files to understand a codebase. That's seven hundred kilobytes in context. It runs `cat access.log` to find error patterns. Forty-five kilobytes.

Think in Code flips this. Instead of pulling data into context for the model to "read" — you send code to the data. The agent writes a JavaScript or Python script. context-mode executes it in a sandboxed subprocess. Only the stdout — the agent's own printed conclusions — enters context.

*(pause)*

Concrete example: analyzing access logs. Before — `cat access.log`, forty-five kilobytes enters context, stays there forever, gets re-sent on every turn. After — `ctx_execute("javascript", ...)`, the agent writes five lines of code to filter five-hundred errors and count them. One hundred fifty-five bytes of stdout enters context. The forty-five kilobyte log file never touches the conversation.

Two hundred times reduction. And the agent arguably does a better job — because it's writing actual analysis code instead of trying to pattern-match in a text window.

*(pause)*

As of version 1.0.64, Think in Code is mandatory across all fourteen platform configs. It's not a suggestion in the docs. It's enforced by the routing rules. If the agent tries to `Read()` a large file for analysis, PreToolUse redirects it to `ctx_execute_file` instead. The raw file loads into a `FILE_CONTENT` variable inside the sandbox. The agent writes code against it. Only stdout comes back.

Your CPU does the processing for free. Tokens cost money. This is the right trade-off.

---

## Slide 9 — Index (Session Persistence)

Index.

*(pause)*

Everything the agent touches goes into a local SQLite database with an FTS5 virtual table and BM25 ranking. Tool outputs, web pages fetched and chunked, sandbox execution results — all full-text searchable. The agent calls `ctx_search` with a query, gets ranked results back. Like a local search engine for everything the agent has ever seen in this session.

But here's what actually makes context-mode different from a caching layer. **Session persistence.**

*(pause)*

PostToolUse doesn't just capture tool output. It captures events. Fifteen categories of events, as they happen in real-time: file operations, git commands, errors and their fixes, user corrections, environment details, project rules, decision points. All of it goes into a SessionDB — a separate SQLite store.

When the context window fills up and the agent compacts — normally that's game over. Everything summarized and lost. But context-mode has a **PreCompact** hook. Right before the wipe happens, it builds a structured snapshot of the entire session state. Not a lossy summary — structured references to everything in the knowledge base.

Then **SessionStart** fires on the next turn. It restores that snapshot. Reloads the indexed knowledge. The agent picks up mid-thought.

*(pause)*

Twenty-six event categories carry over through each compaction and through `--resume`. Your prompts, files tracked, project rules, decisions, git operations, errors and fixes, environment state, session mode, tool usage patterns.

The agent compacts? It doesn't forget. It searches the knowledge base and picks up exactly where it left off. You never have to explain your codebase twice. You never re-describe your conventions. You never re-establish the context of what you're building.

---

## Slide 10 — Measured Results

Let's look at the actual numbers.

*(pause)*

Playwright browser snapshot: fifty-six kilobytes raw DOM. With context-mode: two hundred ninety-nine bytes. A search summary of the relevant elements. Ninety-nine point five percent reduction.

GitHub Issues query: fifty-nine kilobytes of JSON. With context-mode: one-point-one kilobytes. The issues matching the agent's actual question. Ninety-eight percent reduction.

Access log analysis: forty-five kilobytes of raw logs. With context-mode and Think in Code: one hundred fifty-five bytes of stdout. Ninety-nine point seven percent reduction.

*(pause)*

Aggregate across a full fifty-turn session — thirty megabytes re-sent becomes one megabyte. And it's not just input tokens. context-mode also enforces output compression — the agent drops filler words, pleasantries, verbose explanations. Sixty-five to seventy-five percent output token savings on top of the input reduction.

**Ninety-eight to ninety-nine point five percent** total context reduction. Same work. Same answers. The agent doesn't lose any capability. It actually gains capability — because it can work longer before hitting the context wall, and it makes better decisions because the context window isn't polluted with stale tool output.

---

## Slide 11 — Close

context-mode. Open source. Elastic License 2.0. Always has been.

*(pause)*

Ninety-five thousand users. Seventy-nine thousand npm downloads. Fourteen platform adapters. Ten thousand GitHub stars. Used across teams at Microsoft, Google, Meta, ByteDance, GitHub, Stripe, Datadog, NVIDIA, Supabase, and more.

Fourteen adapters — Claude Code, Cursor, Codex CLI, VS Code Copilot, JetBrains Copilot, Gemini CLI, Qwen Code, Kiro, OpenCode, KiloCode, Zed, Pi, Antigravity — and a native **OpenClaw adapter**, which feels especially right to mention here at ClawCon tonight.

*(pause)*

The adapter architecture is the reason one plugin works everywhere. Platforms with lifecycle hooks — Claude Code, Codex — get automatic interception at the hook level. Platforms with MCP support but no hooks — Cursor, Copilot — get an MCP server that exposes `ctx_execute`, `ctx_search`, `ctx_batch_execute` as tools the agent can call. Platforms with neither — some newer CLIs — get instruction-based routing injected into the system prompt. Three mechanisms, one plugin, same result.

*(pause)*

It's open source because I believe the answer to the problem I just showed you can only come from the community — not from vendors. The vendors have no incentive to reduce token usage. That's their revenue. The solution has to come from the ecosystem.

context-mode.com.

Thank you again to Veli, Furkan, Murat, and ClawCon.
