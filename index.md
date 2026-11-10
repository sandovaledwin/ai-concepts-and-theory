
# AI Theory & Concepts

Notes on how AI agents are built and run: prompts, system prompts, context management, harnesses, and the theory behind them.

## Notes

- [[effective-context-engineering|Effective Context Engineering for AI Agents]]
  — Prompt engineering → context engineering: context as a finite attention budget, the anatomy of good context (system prompt, tools, few-shot examples), just-in-time retrieval & progressive disclosure, and the three long-horizon techniques (compaction, structured note-taking, sub-agents).

- [[effective-harnesses-engineering|Effective Harnesses for Long-Running Agents]]
  — Bridging sessions across context windows: an Initializer Agent (`init.sh`, `claude-progress.txt`, first git commit, JSON feature list) plus a Coding Agent that ships one feature at a time behind a fixed startup routine and end-to-end testing.

- [[prompt-engineering-openai|Prompt Engineering (OpenAI)]]
  — Practical prompting for the OpenAI API: model selection (reasoning vs. GPT models), the `developer`/`user` authority hierarchy, prompts-as-code with pinned snapshots and evals, prompt anatomy (Identity → Instructions → Examples → Context) with Markdown + XML, prompt caching, few-shot learning, RAG and context windows, plus playbooks for coding, front-end, and agentic tasks.

- [[system-prompt-guide-with-examples|How to Write a System Prompt That Actually Works]]
  — Anatomy of a production system prompt (role, constraints, format, tone, grounding), length heuristics (3–5 sentences vs. 1–2 pages), differences across OpenAI, Anthropic, and Google, plus six domain prompt patterns (customer support, code review, research, editing, data analysis, project management).

- [[demystifying-evals-for-ai-agents|Demystifying Evals for AI Agents]]
  — Why evals matter (escaping reactive loops, compound value), core anatomy (tasks, trials, graders, transcripts, outcomes, harnesses), grader archetypes (code-based, model-based, human), evaluating agent domains (coding, conversational, research, computer use), statistical interpretation (pass@k vs. pass^k), the zero-to-one roadmap, and holistic evaluation via the Swiss cheese model.

- [[contextual-retrieval|Contextual Retrieval]]
  — Overcoming context loss in traditional RAG: situational chunk contextualization via lightweight LLMs, hybrid Contextual Embeddings + Contextual BM25, prompt caching for cost-effective ingestion ($1.02/M tokens), and cross-encoder reranking to reduce retrieval failures by up to 67%.

- [[prompt-caching|Prompt Caching]]
  — How prompt caching reuses KV cache states across requests: prefix matching mechanics, explicit vs. implicit breakpoints, write/read pricing and the minimum length trap ($L_{\mathrm{break\text{-}even}}$ formula), cache routing and multi-tenant security, and architectural patterns for long-horizon agents.

- [[context-rot-is-a-harness-problem|Context Rot Is a Harness Problem]]
  — Why agent performance degradation stems from unmanaged context rather than model limitations: an architectural framework featuring dual-tier memory (`agent.md`/`memory.md`), three-tier progressive disclosure for skills, active/passive LSP integration, and dual-threshold context compaction.

- [[turn-ai-agent-bugs-into-regression-tests|Turn AI Agent Bugs into Regression Tests]]
  — Converting production failures into deterministic regression tests via Hamel Hussein's Error Analysis Framework and Kitaru replays: session sampling, cohort clustering, deterministic code-based evaluators, tool caching policies (`fail` vs. `passthrough`), and safe model swapping.

- [[evals-101|Evals 101: Building, Testing, and Monitoring AI Systems]]
  — A practical guide to evaluating LLM applications: the three core primitives (tasks, datasets, scores), the 2×2 diagnostic matrix for calibrating tasks vs. scorers, offline experimentation vs. online span-level monitoring, and establishing the evaluation flywheel with human-in-the-loop triage.

## Raw sources

- [[references/raw-effective-context-engineering-for-ai-agents|Raw: Effective Context Engineering]]
- [[references/raw-effective-harnesses-for-long-running-agents|Raw: Effective Harnesses]]
- [[references/raw-demystifying-evals-for-ai-agents|Raw: Demystifying Evals for AI Agents]]
- [[references/raw-prompt-engineering-openai|Raw: Prompt Engineering (OpenAI)]]
- [[references/raw-system-prompt-guide-with-examples|Raw: System Prompt Guide with Examples]]
- [[references/raw-contextual-retrieval|Raw: Contextual Retrieval]]
- [[references/raw-prompt-caching|Raw: Prompt Caching]]
- [[references/raw-context-rot-is-a-harness-problem|Raw: Context Rot Is a Harness Problem]]
- [[references/raw-turn-ai-agent-bugs-into-regression-tests|Raw: Turn AI Agent Bugs Into Regression Tests]]
- [[references/raw-evals-101-doug-guthrie-braintrust|Raw: Evals 101 (Braintrust)]]
