
# AI Theory & Concepts

Notes on how AI agents are built and run: prompts, system prompts, context management, harnesses, and the theory behind them.

## Notes

- [[effective-context-engineering|Effective Context Engineering for AI Agents]]
  — Prompt engineering → context engineering: context as a finite attention budget, the anatomy of good context (system prompt, tools, few-shot examples), just-in-time retrieval & progressive disclosure, and the three long-horizon techniques (compaction, structured note-taking, sub-agents).

- [[effective-harnesses-engineering|Effective Harnesses for Long-Running Agents]]
  — Bridging sessions across context windows: an Initializer Agent (`init.sh`, `claude-progress.txt`, first git commit, JSON feature list) plus a Coding Agent that ships one feature at a time behind a fixed startup routine and end-to-end testing.

## Raw sources

- [[references/raw-effective-context-engineering-for-ai-agents|Raw: Effective Context Engineering]]
- [[references/raw-effective-harnesses-for-long-running-agents|Raw: Effective Harnesses]]
