# Turn AI Agent Bugs into Regression Tests
*[[Based on Decoding AI](references/raw-turn-ai-agent-bugs-into-regression-tests)]*

## The Core Challenge
Modifying an autonomous agent—whether tweaking a single line in a system prompt, swapping underlying foundation models, or altering tool schemas—frequently introduces subtle, cascading regressions. Traditional software development relies on unit tests, and while standard LLM observability tools capture production traces, reproducing multi-turn agent bugs remains exceptionally difficult. When an agent is rerun from scratch, non-deterministic model behavior and shifting external tool environments prevent developers from reconstructing the exact context in which the failure occurred.

To systematically eliminate agent regressions, teams need an organic workflow that converts real-world production failures into permanent test suites. By combining the **Error Analysis Framework** (popularized by Hamel Hussein) with **Deterministic Replays**, developers can cache tool outputs to recreate exact conversational contexts, isolate failure modes into cohorts, implement lightweight deterministic code evaluators, and verify fixes before promoting tests into a permanent regression harness.

## Main Points

### 1. The Organic Regression Loop (Error Analysis Framework)
Relying on synthetic benchmark tasks often fails to reflect production distributions. Instead, regression suites should grow organically from observed real-world failures through a structured, human-in-the-loop lifecycle:

```
[ Production Traces (Real-World Sessions) ]
                    │
                    ▼
[ Sample Investigation Batch (20–30 Sessions) ]
                    │
                    ▼
[ Human Labeling: Acceptable / Problematic / Uncertain ]
                    │ (Stop at First Failure Point)
                    ▼
   [ Cluster Failures into Cohorts ]
                    │ (Rank by Frequency & Severity)
                    ▼
[ Implement Scoped Code Evaluator ]
                    │ (Confirm 100% Failure on Baseline)
                    ▼
[ Fork Code & Run Replay Experiment ]
                    │ (Verify All Evaluators Pass)
                    ▼
[ Graduate Session + Evaluator to Regression Suite ]
```

1. **Capture Real-World Traces:** Ingest complete, multi-turn agent sessions directly from production into the eval harness (e.g., Kitaru, or exported from observability platforms like Langfuse, Arize Phoenix, or Braintrust).
2. **Sample an Investigation:** Rather than attempting to analyze thousands of logs, sample a representative batch (e.g., 20–30 sessions per iteration).
3. **Human Review & Labeling:** Inspect sessions manually and classify each as *Acceptable* (`1`), *Problematic* (`0`), or *Uncertain* (`-1`), accompanied by a concise critique.
4. **Stop at the First Failure:** When labeling a problematic session, halt analysis at the initial point of failure. Later turns in an agent trajectory inevitably cascade from the first error; debugging secondary effects adds noise and wastes effort.
5. **Cluster into Cohorts:** Group failing sessions into cohorts based on shared root causes (e.g., model provider disconnections, rate limits, invalid tool schemas, hallucinated file paths).
6. **Prioritize by Frequency and Severity:** Rank cohorts so engineering effort targets the highest-impact bottlenecks first.
7. **Author Scoped Evaluators:** Write targeted evaluators designed exclusively to detect the specific error characteristic of that cohort.
8. **Verify Fixes in Experiments:** Run the evaluator across the failing cohort baseline to ensure reliable detection. Fork the codebase, implement the fix (in prompts, tools, or model routing), rerun the cohort via replay experiments, and confirm that all sessions pass.
9. **Graduate into the Regression Suite:** Add the verified evaluator and representative session traces to the permanent test harness to guard against future regressions.

### 2. Code-Based Evaluators vs. LLM Judges
Because error cohorts isolate specific failure modes down to root causes, evaluators rarely need open-ended natural language reasoning:

| Dimension | Code-Based Evaluators | LLM / Decision Model Judges |
| :--- | :--- | :--- |
| **Execution Cost** | Near-zero (CPU compute) | High (per-token API fees) |
| **Execution Speed** | Sub-millisecond | Seconds to tens of seconds (high latency) |
| **Determinism** | 100% reproducible | Non-deterministic; varies with sampling |
| **Drift & Maintenance** | Zero drift over time | High drift risk across model versions/prompts |
| **Implementation Complexity** | Substrings, regex, status codes, schema checks | Complex prompt engineering, few-shot calibration |

*Heuristic:* Prefer **code-based evaluators** whenever possible. Scoping evaluators to targeted error clusters allows simple string markers (e.g., matching connection error payloads or HTTP 429 status codes) to replace expensive LLM judges. Reserve model-based judges strictly for open-ended, subjective outputs where deterministic parsing is infeasible.

### 3. Eval Harness Architecture
Deploying this methodology in production requires a four-part system architecture that decouples agent execution from evaluation orchestration:

*   **Agent Under Test (AUT):** The target agent (e.g., a Pydantic AI agent) connected to the harness via a lightweight instrumentation wrapper or custom offline trace importers.
*   **Eval Control Plane (e.g., Kitaru):** Centralizes all session logs, human review investigations, failure cohorts, experiment runs, and regression suites.
*   **Sandboxed Workers:** Isolated runner environments executing the agent across parameterized experiments in parallel, preventing cross-test state pollution or filesystem collisions.
*   **Coding Agent Integration (Skills + MCP):** Equipping an autonomous coding assistant (e.g., Claude Code, Cursor, Codex) with domain skills (error analysis heuristics, cohort partitioning rules) and an MCP server connected to the control plane allows the coding agent to autonomously cluster traces, write targeted evaluators, and execute verification experiments.

### 4. Deterministic Replays: Tool Interception and Policies
The core obstacle in reproducing agent bugs is **context drift**: external APIs, databases, and filesystem states change between runs, causing agents to diverge along different trajectories. **Replays** solve this by caching all tool outputs during the original session and injecting the cached data during replay runs:

```
Production Run:   Agent ──(Tool Call: args)──▶ Live Tool Execution ──▶ External State
                                                      │
                                             [ Cached Tool Output ]
                                                      │
Replay Run:       Agent ──(Tool Call: args)──▶ Replay Harness ───────▶ [ Return Cached Data ]
```

Harnesses manage replays using granular **tool policies** defined across two dimensions:

#### Replay Type
*   `history` (Cache): Replays the exact cached tool output recorded during the production session when matching tool arguments are supplied.
*   `static`: Returns static mock outputs defined in test configurations.
*   `passthrough`: Bypasses the replay cache and executes the live tool against real systems.
*   `llm`: Uses a secondary LLM to synthetically generate plausible tool outputs on the fly.

#### On-Miss Behavior (Uncached Tool Calls)
Defines what happens when an agent calls a tool with arguments not found in the replay cache:
*   `fail` (Strict / Recommended): Immediately fails the replay run if a tool call deviates from the cache. Prevents the agent from wandering off-track and eliminates unintended external side effects. Ideal for precision bug reproduction.
*   `passthrough` (Permissive): Executes the live tool when a cache miss occurs. Essential for sessions that crashed on turn zero before executing tools, but introduces the risk of live side effects (external API consumption, database writes, or destructive mutations).

### 5. Safe Model Swapping and Differential Testing
Once an organic regression suite is established, it enables high-velocity iteration across agent infrastructure:
*   **Differential Trace Inspection:** Compare the baseline failing session and the patched replay session side-by-side to verify the exact causal effect of code changes.
*   **Safe Model Upgrades:** When switching foundation models (e.g., upgrading between model generations or routing to smaller open-weights models), run the full cohort regression suite across sandboxed workers with model overrides. A clean pass across all evaluators confirms model compatibility without regression risk.
*   **Prompt & Configuration Experiments:** Safely adjust system prompts, temperature settings, and tool schemas against cached environment context, isolating model reasoning from external runtime volatility.

## Operational Boundaries (What Replays Do Not Solve)

Replays are designed specifically for reproducing known failures and securing regression baselines; they are not a substitute for open-ended testing:
*   **Not for Synthetic Benchmarks:** Benchmarking requires evaluating model exploration on novel tasks from scratch; cached tool outputs constrain exploration.
*   **Not for Performance & Latency Tuning:** Replays short-circuit network I/O with memory lookups, masking real-world latency, network jitter, and external API rate limits.
*   **Not for Unseen Failure Discovery:** Replays only validate behavior against previously encountered execution paths. Discovering novel edge cases still requires exploratory testing, adversarial red-teaming, and production observability.

## Best Practices Checklist

- [ ] **Ingest Real Traces:** Build regression cases exclusively from production sessions; avoid synthetic mock data.
- [ ] **Sample Manageable Batches:** Review 20–30 sessions per investigation cycle to maintain human labeling rigor.
- [ ] **Isolate the First Error:** Always stop trace evaluation at the earliest failure point before context degrades.
- [ ] **Group into Error Cohorts:** Cluster failures by root cause and address the top 20% most frequent or severe issues.
- [ ] **Default to Code Evaluators:** Use deterministic string, regex, or status code checks instead of expensive, drifting LLM judges.
- [ ] **Enforce Strict Replay Policies:** Set on-miss behavior to `fail` during bug reproduction to avoid side effects and trajectory drift.
- [ ] **Integrate with CI/CD:** Run full cohort replay suites on every prompt tweak, tool refactor, and model upgrade.
