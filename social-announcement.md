# ClawCon Ankara — LinkedIn Announcement

I'll be giving a short talk at ClawCon Ankara on April 29.

"The Other Half of the Context Problem." Five minutes on something that's been bugging me. Your AI coding agent spends most of its context window re-sending tool output it already processed. Every turn, the full conversation ships to the API again. The context fills up, the agent compacts, and you start over. Re-explaining your codebase to a model that just lost its memory.

I built context-mode to fix this. It's an MCP server that sits between the agent and its tools. Intercepts the output, indexes it into a local FTS5 database, gives the agent a 1KB summary instead. 10K stars on GitHub, 82K npm downloads, works on 14 platforms.

I'll walk through how the interception works, the Think in Code paradigm, and why session persistence matters more than people realize.

Organized by @BuilderMare (Veli Uysal, Furkan Demir) through @ClawCon and the OpenClaw community. Special thanks to Murat Aslan for making it happen.

Ankara'daysan gel.

context-mode.com

#ClawCon #ClawConAnkara #ContextMode #ClaudeCode #AIAgents #MCP #OpenSource #Ankara

---

## Data sources used

| Number | Source | Verified |
|--------|--------|----------|
| 10K+ stars | `gh api repos/mksglu/context-mode --jq '.stargazers_count'` = 10,173 | Yes |
| 82K npm | `npm view context-mode` all-time = 81,677 | Yes |
| 14 platforms | context-mode.com homepage | Yes |
