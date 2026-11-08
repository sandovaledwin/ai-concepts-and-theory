# Effective Context Engineering for AI Agents
*[[Based on Anthropic's Research](references/raw-effective-context-engineering-for-ai-agents)]*

## Introduction
As language models advance, the focus is shifting from **prompt engineering** (finding the right words) to **context engineering**—curating the optimal set of information (tokens) during LLM inference to consistently achieve desired outcomes.

## Main Points

### 1. The Shift from Prompt to Context Engineering
*   **Prompt Engineering:** Focuses on writing and organizing instructions (usually static) for tasks like text generation or classification.
*   **Context Engineering:** An iterative, dynamic process of managing the *entire* context state (system instructions, tools, external data, message history) as an agent operates in a loop over time.

### 2. Context as a Finite Resource
*   **Context Rot / Attention Budget:** Like humans, LLMs lose focus when parsing massive amounts of data. Every new token depletes their "attention budget."
*   **Architectural Constraint:** Because transformers compute relationships between every token ($n^2$ relationships for $n$ tokens), longer contexts stretch the model's attention thin, leading to reduced precision in retrieval and reasoning.
*   **The Goal:** Find the smallest possible set of high-signal tokens that maximize the likelihood of the desired outcome.

### 3. The Anatomy of Effective Context
*   **System Prompts:** Must find the "Goldilocks zone" (the right altitude). Avoid brittle, hardcoded logic (too specific) and vague guidance (too broad). Use clear formatting (e.g., XML tags or Markdown headers).
*   **Tools:** Should be token-efficient, self-contained, robust to errors, and have clear input parameters. Avoid bloated toolsets to prevent agent confusion.
*   **Examples (Few-Shot Prompting):** Instead of a massive laundry list of edge cases, provide a curated set of diverse, canonical examples that illustrate expected behavior.

### 4. Context Retrieval and Agentic Search
*   **Just-in-Time Context:** Instead of loading all data upfront, agents use tools to dynamically retrieve data at runtime (e.g., querying databases, reading specific files). This mirrors human cognition.
*   **Progressive Disclosure:** Agents incrementally discover context, maintaining only what is necessary in their working memory. 
*   **Hybrid Strategy:** Sometimes, pre-loading essential data while allowing the agent to autonomously explore the rest is the most effective approach.

### 5. Managing Long-Horizon Tasks
To prevent context pollution in tasks spanning hours (e.g., coding migrations, research), three main techniques are used:
*   **Compaction:** Summarizing the conversation as it nears the context limit and reinitiating a new window with the summary. Maximizing recall first, then refining precision is key.
*   **Structured Note-Taking (Agentic Memory):** The agent writes and persists notes outside the context window (e.g., a `NOTES.md` file) to track progress and dependencies without bloating active memory.
*   **Sub-agent Architectures:** Using specialized sub-agents with clean context windows to perform deep work. They report back distilled summaries to a main coordinating agent, isolating detailed search context.

## Conclusion
Context engineering is about treating the LLM's context window as a precious resource. Even as model capabilities improve, intentionally managing what information enters the model's attention budget remains critical for building reliable, autonomous, and effective AI agents.