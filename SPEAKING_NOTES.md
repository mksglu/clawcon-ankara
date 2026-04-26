# Speaking Notes — The Other Half of the Context Problem

**ClawCon Ankara** · April 29, 2026
**Speaker:** B. Mert Koseoğlu

---

## Slide 1 — Title

> Good evening everyone.

Quick thank you to Veli and Furkan from BuilderMare, to Murat Aslan, and to ClawCon and the OpenClaw community for making this happen in Ankara.

*(pause)*

I'm Mert. I build open-source tools for AI coding agents. Tonight I want to show you a problem that every single person in this room is living with. You just might not have a name for it yet.

---

## Slide 2 — Your AI agent has no memory

Let's start with a fact that might surprise you.

*(pause)*

Your AI coding agent has no memory. None. Claude Code, Cursor, Copilot, Codex, Gemini CLI — under the hood, they all work exactly the same way. The model is stateless. It doesn't remember anything from one turn to the next.

So how does it seem like it remembers? Let me show you.

---

## Slide 3 — Every turn, everything gets re-sent

Here's what actually happens.

*(pause)*

Turn one — you ask something, a tool runs, sixty kilobytes come back. That's fine.

Turn five — all the output from turns one through four gets re-sent along with the new stuff. Three hundred kilobytes now.

Turn fifteen — everything ships again. Six hundred kilobytes.

Turn thirty — you're at 1.2 megabytes and the context window is almost full.

*(pause)*

Turn thirty-one? The agent compacts. It summarizes the conversation. Throws away the originals. And just like that — it forgets everything it read, every decision it made, the task it was in the middle of.

You start over.

---

## Slide 4 — One command. 750,000 tokens.

Let me make this concrete.

*(pause)*

You run `gh issue list`. One command. Fifty-nine kilobytes of JSON come back. About fifteen thousand tokens. Seems reasonable, right?

*(pause)*

But next turn, those fifteen thousand tokens ship again. Turn after that — again. Fifty turns in, that single command has cost you **seven hundred fifty thousand** input tokens.

*(pause)*

And you're not running one command per session. You're running twenty. Average output: thirty kilobytes each.

That's thirty megabytes of input tokens. On tool output alone.

That's where your context goes.

---

## Slide 5 — When the context fills up, everything disappears

And when it fills up, everything disappears.

*(pause)*

Files it read — gone. Decisions it made — gone. Errors it encountered and fixed — gone. Your instructions and corrections — gone. The task it was in the middle of building — gone.

*(pause)*

Fifty percent accuracy drop after thirty-two thousand tokens. Fifteen minutes lost every time it resets, re-explaining your codebase from scratch. Sixty thousand dollars a year for a fifty-seat team — burned on an agent that forgets everything every twenty minutes.

---

## Slide 6 — What if the data never entered context?

*(let it sit for a moment)*

What if the data never entered the context window in the first place?

---

## Slide 7 — Intercept

context-mode intercepts.

*(pause)*

Your agent calls `gh issue list`. Fifty-nine kilobytes of JSON come back. Normally, all of that dumps straight into your context window.

But context-mode is sitting between the agent and its tools. It uses five lifecycle hooks — PreToolUse, PostToolUse, SessionStart, PreCompact, UserPromptSubmit — to intercept tool output before it reaches the conversation.

The full fifty-nine kilobytes goes to a local FTS5 database. Only a one-point-one kilobyte summary enters context.

The agent still gets the answer. Your context window barely notices.

---

## Slide 8 — Sandbox (Think in Code)

Sandbox.

*(pause)*

This is the paradigm shift. We call it **Think in Code**.

Instead of pulling forty-five kilobytes of access logs into your context window for the agent to read — you send code to the data. The agent writes a small script, context-mode executes it in a sandbox, and only the stdout — one hundred fifty-five bytes — enters context.

*(pause)*

Two hundred times reduction. The agent still gets the answer. It wrote the analysis. But the raw data never touched the context window.

This is mandatory across all fourteen platforms context-mode supports.

---

## Slide 9 — Index (Session Persistence)

Index.

*(pause)*

Everything the agent touches goes into a local FTS5 database with BM25 ranking. Tool outputs, web pages, sandbox results — all searchable.

But here's what makes it special: **session persistence**.

*(pause)*

Twenty-six event categories — files modified, decisions made, errors encountered, user corrections, git state, environment details — all of it carries over through compactions and even through session resume.

The agent compacts? It doesn't forget. It searches the knowledge base and picks up right where it left off.

You never have to explain your codebase twice.

---

## Slide 10 — Measured Results

Let's look at the numbers.

*(pause)*

Playwright snapshot: fifty-six kilobytes down to two hundred ninety-nine bytes. Ninety-nine point five percent reduction.

GitHub issues: fifty-nine kilobytes down to one point one.

Access logs: forty-five kilobytes down to one hundred fifty-five bytes.

*(pause)*

Per session — thirty megabytes re-sent becomes one megabyte. **Ninety-eight to ninety-nine point five percent** context reduction.

Same work. Same answers. The agent doesn't lose any capability. It just stops wasting your context window on data it already processed.

---

## Slide 11 — Close

context-mode. Open source. Always has been.

*(pause)*

Ninety-five thousand users. Seventy-nine thousand npm downloads. Fourteen platform adapters — including a native OpenClaw adapter, which feels right to mention here tonight. Ten thousand GitHub stars.

*(pause)*

It's open source because I believe the answer to the problem I just showed you can only come from the community — not from vendors.

*(pause)*

context-mode.com.

Thank you again to Veli, Furkan, Murat, and ClawCon.
