# Context Rot Is a Harness Problem
*[[Based on Decoding AI](references/raw-context-rot-is-a-harness-problem)]*

## The Core Challenge
When an autonomous coding agent starts making compounding errors, hallucinating, or failing at simple reasoning tasks, practitioners often blame the underlying foundation model. However, model swapping (e.g., switching from OpenAI to Anthropic) rarely solves the issue. The root cause is almost always **context rot**—an architectural failure of the agent harness that allows the context window to become polluted with stale files, massive raw tool outputs, redundant syntax retries, and conversational noise.

Every token loaded into a context window incurs three distinct penalties: increased financial cost, higher latency, and degraded model attention. Managing context is not a prompt-writing problem; it is a **harness engineering problem**. A well-architected agent harness enforces strict token economics through four core mechanisms: session-to-session memory persistence, progressive skill disclosure, LSP-powered deterministic feedback loops, and intelligent context trimming (micro-compaction and compaction with tail preservation).

## Main Points

### 1. Dual-Tier Memory Architecture
To prevent the agent from suffering from amnesia across sessions without overwhelming the initial context window, the harness separates static project guidance from dynamic preference extraction:

*   **Manual Domain Knowledge (`agent.md`):** Static project conventions, architecture guidelines, and operational constraints authored by developers. Always injected into the context window across every session.
*   **Dynamic Preference Extraction (`memory.md`):** Automatically maintained cross-session memory (similar to Claude Code's memory pattern).
    *   *Trigger Mechanism:* Executed automatically on session termination (`/quit` or `/clear`) before discarding conversation state.
    *   *Append-Only Log Structure:* Captures a single-sentence summary paired with a date timestamp per conversation (`YYYY-MM-DD: <one-sentence extraction>`). Constraining extractions to a single sentence forces the summarizing model to preserve only high-signal operational preferences and discard noise.
    *   *Rolling Compression Threshold:* Hard-capped at **200 lines** (or ~25 KB). When the log exceeds this threshold, a specialized LLM condenses the entries into a higher-level summary.

### 2. Three-Tier Progressive Disclosure for Skills
Eagerly loading instructions, tool schemas, scripts, and documentation into the initial context window quickly exhausts the model's attention budget. The harness enforces **progressive disclosure** across three distinct tiers:

| Tier | Component Loaded | Context Impact | Trigger Condition |
| :--- | :--- | :--- | :--- |
| **Tier 1: Catalog** | Skill name (ID) + trigger description | Minimal (~1% context cap) | Always present in system prompt |
| **Tier 2: Activation** | `skill.md` prompt body + asset path manifests | Moderate (task-specific instructions) | Explicit agent invocation via `skill(name)` |
| **Tier 3: Execution** | Referenced scripts, assets, and documentation | As-needed (transient payloads) | Agent autonomously invokes file-read tools |

*   **Catalog Assembly:** The harness builds a lightweight dictionary mapping skill IDs to concise activation descriptions. Capping catalog metadata (e.g., at 1% of the total context window) ensures that scaling the skill library does not crowd out working memory.
*   **Two-Way Awareness:** The agent reads only the metadata until the task demands execution. Upon calling the skill, the harness returns the skill prompt along with explicit file paths to relevant assets (e.g., execution scripts or reference schemas), enabling the agent to selectively fetch deep context via standard read tools.

### 3. Language Server Protocol (LSP) as the Cheapest Feedback Loop
Forcing an LLM to read entire codebases to trace references or parse raw compiler outputs to debug syntax errors wastes massive amounts of context. Offloading code intelligence to an embedded Language Server Protocol (LSP) server provides deterministic, low-token feedback loops in two operational modes:

*   **Active Mode (`lsp` Tool):** The agent explicitly queries the server for precise code navigation operations:
    *   `definition`: Jumps directly to symbol origins without exploratory file reading.
    *   `references`: Returns exact file coordinates (file, line, column) and match counts where symbols are used.
    *   `hover`: Retrieves typed signatures and docstrings on demand.
*   **Passive Diagnostics on Write/Edit:** The harness automatically intercepts `write` and `edit` tool calls.
    *   Immediately after a file is saved, the harness queries the LSP for compiler diagnostics and syntax errors.
    *   If diagnostics return clean (`None`), execution proceeds seamlessly.
    *   If errors exist, the harness formats a concise error summary and injects it directly into the next turn of the agent loop.
    *   *Result:* The agent receives immediate, deterministic correction feedback without invoking slow, expensive build/runtime bash commands or hallucinating syntax issues.

### 4. Dual-Threshold Context Trimming: Compaction vs. Micro-Compaction
Context degradation follows a non-linear curve: as working memory approaches saturation, retrieval accuracy drops sharply. A resilient harness avoids naive context clearing (which erases all working memory) by implementing a two-phase reduction pipeline:

```
Context Window Usage:
[0% ------------------------ 60% ------------------ 80% ------------------ 100%]
                              |                      |
                     Micro-Compaction         Full Compaction
                   (Tool Output Masking)    (Summarization + Tail)
```

*   **Micro-Compaction (Triggered at ~60% Capacity):**
    *   *Mechanism:* Identifies historical tool outputs and retry error messages that have already served their purpose and replaces their bodies with minimal placeholders (e.g., `[output pruned]`).
    *   *Rationale:* Tool execution outputs (e.g., grep results, raw terminal logs, LSP dumps) constitute the vast majority of context bloat. Once processed, their exact token content is rarely needed.
    *   *Tail Exemption:* Preserves the raw outputs of the most recent turns intact so the agent maintains immediate operational awareness.
*   **Full Compaction with Tail Preservation (Triggered at ~80% Capacity):**
    *   *Reserve Buffer:* Enforces a fixed buffer (e.g., 20% remaining capacity) to ensure the agent never hits hard provider token limits during multi-turn calls.
    *   *Summary + Tail Split:* Splits conversation history into an older segment and an active recent "tail" (the last $k$ turns).
    *   *State Reconstruction:* An LLM summarizes the older turns into a cohesive status message. The harness rebuilds the context by prepending the system prompt, appending the structured summary, and replaying the uncompressed recent tail.

## Architectural Best Practices

*   **Treat Context as an Attention Budget:** Prioritize token compactness over conversational completeness. Every token must justify its presence.
*   **Offload Deterministic Work to Non-LLM Systems:** Use LSP servers for syntax checking, symbol lookups, and dependency graphs instead of relying on the LLM to inspect multiple raw source files.
*   **Prune Tool Outputs First:** Never compress reasoning steps or goals before pruning redundant, historical tool payloads.
*   **Preserve the Recent Tail:** When summarizing or compacting, always leave the latest conversational steps uncompressed to prevent disjointed agent behavior.
