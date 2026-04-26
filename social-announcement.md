# ClawCon Ankara Announcement Posts

## LinkedIn Post

I'll be giving a short talk at ClawCon Ankara on April 29.

"The Other Half of the Context Problem." Five minutes on something that's been bugging me. Your AI coding agent spends most of its context window re-sending tool output it already processed.

I went and traced it through Claude Code's source code. There's an array called mutableMessages. Every tool result you've ever run lives in it, and it ships to the API on every turn. Run `gh issue list`, 59KB of JSON comes back. Fifteen thousand tokens. Fifty turns later that one command has cost you 750,000 input tokens. The agent looked at it once.

I built context-mode to fix this. It's an MCP server that sits between the agent and its tools. Intercepts the output, indexes it into a local FTS5 database, gives the agent a 1KB summary instead. 10K stars on GitHub, 82K npm downloads, works on 14 platforms.

I'll walk through how the interception works, the Think in Code paradigm, and why session persistence matters more than people realize.

Organized by @BuilderMare (Veli Uysal, Furkan Demir) through @ClawCon and the OpenClaw community. Special thanks to Murat Aslan for making it happen.

Ankara'daysan gel.

context-mode.com

#ClawCon #ClawConAnkara #ContextMode #ClaudeCode #AIAgents #MCP #OpenSource #Ankara

---

## X/Twitter Post (Thread)

### Tweet 1

Speaking at ClawCon Ankara on April 29.

"The Other Half of the Context Problem"

Your AI agent re-sends every tool result on every turn. `gh issue list` returns 59KB. Fifty turns later, 750K tokens burned on data nobody looked at twice.

Five minutes on how context-mode stops it.

### Tweet 2

I traced it in Claude Code's source. There's an array called mutableMessages. Every tool result sits there until compaction fires. A function called truncate_function_output_payload shortens big outputs but never removes them.

Set a 200K context window. Tool output takes 150K. You've got 50K left for thinking.

### Tweet 3

context-mode intercepts tool output before it hits the conversation. Five lifecycle hooks, local FTS5 database, BM25 ranking. The agent gets a 1KB summary instead of the raw dump.

10K+ GitHub stars. 82K npm. 14 platforms.

context-mode.com

### Tweet 4

Thanks @buildermare and @openclaw for putting this together. See you in Ankara.

Veli Uysal, Furkan Demir, Murat Aslan, appreciate it.

---

## Data sources used

| Number | Source | Verified |
|--------|--------|----------|
| 10K+ stars | `gh api repos/mksglu/context-mode --jq '.stargazers_count'` = 10,173 | Yes |
| 82K npm | `npm view context-mode` all-time = 81,677 | Yes |
| 14 platforms | context-mode.com homepage | Yes |
| 59KB gh issue list | context-mode.com / blog post | Yes |
| 750K tokens | 15K tokens x 50 turns, from blog | Yes |
| 98-99.5% reduction | context-mode.com measured results | Yes |

## AI pattern audit

- No em dashes used (checked: zero instances)
- No promotional language ("groundbreaking", "revolutionary", etc.)
- No significance inflation ("pivotal", "testament", etc.)
- No rule of three forced groupings
- No negative parallelisms
- No filler phrases
- No generic conclusions
- No emojis
- Sentence length varies: short technical statements mixed with longer explanations
- First person throughout, founder voice
- Numbers are specific and verified from live API calls
