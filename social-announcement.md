# ClawCon Istanbul — LinkedIn Announcement

I'll be giving a short talk at ClawCon Istanbul on Wednesday, May 6.

"The Other Half of the Context Problem." Five minutes on what's actually bleeding your context window — and your wallet.

Your AI coding agent has no memory. The model is stateless. Every turn, the entire conversation ships to the API again — every tool result, every file read, every error you debugged together. Sixty turns in, you're paying to re-send thirty megabytes of stale output the agent already processed. Then the window fills up, the agent compacts, and your codebase, decisions, half-finished refactor — gone. You start explaining your project from scratch.

In 80 hours of pair programming with Opus: $487 in API costs saved, 22 hours of re-explaining recovered, 268 sessions resumed without "what were we doing?". 6.2 MB of raw tool output became 124 KB in context. 98% reduction.

I'll walk through three things in five minutes:

→ Intercept — how five lifecycle hooks pull tool output out of conversation before it ever lands. 59 KB of GitHub JSON becomes a 1.1 KB summary. The full data stays searchable, just not re-sent fifty times.

→ Think in Code — stop pulling data into context for the model to "read." Send code to the data instead. 47 files for codebase analysis: 700 KB → 3.6 KB. Your CPU does the work for free.

→ Session Persistence — why losing memory at compaction is the real cost. 26 event categories survive across compactions, --resume, and new sessions. The agent stops re-learning your codebase.

12.5K GitHub stars. 104K npm downloads. 14 platform adapters — including a native OpenClaw one, which feels right to mention here.

Open source. Elastic License 2.0. No telemetry. No account.

Organized by @clawcon and the @openclaw community. Hosted by @0xtotaylor, @msg, and @buildermare at Tech Istanbul, Şişli.

Istanbul'daysan gel.

context-mode.com

#ClawCon #ClawConIstanbul #ContextMode #ClaudeCode #AIAgents #MCP #OpenSource #Istanbul

---

## Data sources used (verified live, uncached)

| Number | Source | Value |
|--------|--------|-------|
| 12.5K stars | `api.github.com/repos/mksglu/context-mode` | 12,472 |
| 862 forks | `api.github.com/repos/mksglu/context-mode` | 862 |
| 104K npm | `cdn.jsdelivr.net/gh/mksglu/context-mode@main/stats.json` | 103.9k+ |
| 125K users | `cdn.jsdelivr.net/gh/mksglu/context-mode@main/stats.json` | 125.4k+ |
| 14 platforms | context-mode README | 14 |
| $487 / 22h / 268 sessions | personal 80h Opus session report | — |
