# Evals 101: Building, Testing, and Monitoring AI Systems
*[[Based on Doug Guthrie (Braintrust)](references/raw-evals-101-doug-guthrie-braintrust)]*

## The Core Challenge

Building software with Large Language Models introduces non-deterministic behavior that breaks standard software testing paradigms. Changing a prompt, swapping an underlying foundation model, or tweaking a tool schema can silently degrade output quality without triggering runtime errors. Without rigorous evaluation, teams fall into "vibes-based" development—subjectively inspecting a handful of outputs, deploying changes blindly, and reacting only when users complain.

While traditional unit tests serve to "play defense" (preventing known code regressions), evaluations allow teams to **play offense**: systematically hill-climbing on quality, validating architectural changes, and establishing the empirical trust necessary to push AI applications into production.

---

## Main Points

### 1. The Three Core Ingredients of an Evaluation

Every evaluation suite consists of three fundamental primitives:

```
┌─────────────────────────────────────────────────────────────┐
│                          EVALUATION                         │
├─────────────────┬───────────────────┬───────────────────────┤
│    1. TASK      │    2. DATASET     │       3. SCORES       │
│                 │                   │                       │
│  • Prompt       │  • Input (req)    │  • Code-based         │
│  • Tool Chain   │  • Expected (opt) │  • LLM-as-a-Judge     │
│  • Agent Graph  │  • Metadata (opt) │  • Pre-built packages │
└────────┬────────┴─────────┬─────────┴───────────┬───────────┘
         │                  │                     │
         ▼                  ▼                     ▼
   [ Run Task ]  ───►  [ Output ]  ───►  [ Evaluate Score ]
```

1. **Task:** The target computation being tested. It requires an input and produces an output. A task can range from a single prompt with mustache templating to complex, multi-turn agentic workflows invoking external tools.
2. **Dataset:** The test suite representing real-world distribution.
   - **Input (Required):** The user query, conversational history, or system context.
   - **Expected Output (Optional):** Ground truth used by deterministic comparators (e.g., Levenshtein distance, exact match).
   - **Metadata (Optional):** Key-value tags enabling granular filtering, cohort slicing, and targeted evaluation runs.
3. **Scores (Scorers):** The grading logic assessing output quality:
   - **Code-Based Scores:** Deterministic heuristics written in TypeScript or Python (e.g., regex checks, JSON schema validation, Levenshtein distance). Fast, cheap, and binary.
   - **LLM-as-a-Judge Scores:** Model-driven grading against explicit natural language rubrics, scoring outputs continuously (e.g., $0.0$ to $1.0$) or categorically (e.g., *Poor*, *Fair*, *Excellent*).
   - **Pre-Built Evaluators (`autoevals`):** Out-of-the-box code and model scorers providing immediate baselines for common metrics (relevance, hallucination, similarity).

---

### 2. The Diagnostic Matrix: Calibrating Tasks vs. Scorers

When building an evaluation pipeline, teams often struggle to diagnose discrepancies between observed outputs and automated scores. The following 2×2 diagnostic matrix isolates whether the engineering bottleneck lies in the task logic or the scoring harness:

| Output Quality | Evaluation Score | Diagnosis & Action Required |
| :--- | :--- | :--- |
| **Good Output** | **High Score** | **System Healthy:** Baseline validated; promote task to regression suite. |
| **Good Output** | **Low Score** | **Scorer Overly Strict:** Fix the evaluation logic (loosen rubrics, calibrate thresholds, or remove overly narrow assertions). |
| **Bad Output** | **High Score** | **Scorer Overly Lenient:** Fix the evaluation logic (tighten rubric criteria, add missing edge cases, or introduce code assertions). |
| **Bad Output** | **Low Score** | **Application Failing:** Scorer is accurate; optimize prompts, adjust tool schemas, or upgrade model architecture. |

#### Scorer Design Heuristics
- **Asymmetric Model Tiering:** Use high-capability models (e.g., frontier reasoning models) for scoring, even if production tasks run on smaller, cost-effective models.
- **Single-Focus Modularity:** Decompose evaluation criteria into discrete, single-attribute scorers (e.g., separate scorers for *Accuracy*, *Formatting*, and *Factual Completeness*) rather than overloading a single prompt with multiple complex criteria.
- **Context Isolation:** Provide scorers only with relevant inputs and outputs; overloading judge prompts with full conversational context induces hallucinations and scoring drift.
- **Start Small:** Do not wait to assemble a massive "golden dataset." Establish a simple baseline with a handful of test cases and basic distance metrics, then iterate.

---

### 3. Dual Mental Models: Offline vs. Online Evaluations

A complete evaluation strategy bridges pre-production iteration with live operational monitoring:

| Dimension | Offline Evals (Pre-Production) | Online Evals (Production Monitoring) |
| :--- | :--- | :--- |
| **Primary Goal** | Rapid iteration, model benchmarking, regression prevention. | Live quality monitoring, drift detection, anomaly identification. |
| **Execution Environment** | Developer playgrounds, CLI/SDK test suites, CI/CD pipelines. | Live application runtime via instrumented loggers and wrappers. |
| **Dataset Source** | Curated historical test cases and synthetic failure examples. | Live production traffic streams. |
| **Scoring Scope** | 100% of defined test cases across all iterations. | Sampled traffic (e.g., 10%–20% sampling) or targeted span triggers. |
| **Artifacts** | Versioned **Experiments** with comparative metric diffs. | Real-time traces, span metrics (latency, cost, tokens), automated alerts. |

---

### 4. Granular Span-Level Evaluation

Evaluating only the final root output of an agent or multi-step pipeline conceals root causes. In multi-turn systems, intermediate step failures cascade into catastrophic output errors:

```
[ User Query ]
      │
      ▼
┌───────────────────────┐
│ Span 1: Rephrase Query│ ───► [ Span Scorer: Semantic Faithfulness ]
└──────────┬────────────┘
           │
           ▼
┌───────────────────────┐
│ Span 2: Intent / Tools│ ───► [ Span Scorer: Tool Argument Schema ]
└──────────┬────────────┘
           │
           ▼
┌───────────────────────┐
│ Span 3: Data Retrieval│ ───► [ Span Scorer: Context Relevance ]
└──────────┬────────────┘
           │
           ▼
┌───────────────────────┐
│ Span 4: Synthesis     │ ───► [ Root Scorer: Output Quality ]
└───────────────────────┘
```

- **Early Failure Isolation:** If a query rephrasing span misunderstands user intent, downstream retrieval and tool executions are guaranteed to fail. Attaching scoped evaluators directly to intermediate spans detects failures at the point of origin.
- **Targeted Instrumentation:** Using function decorators (`@traced`) or SDK model wrappers captures structured inputs, outputs, and metadata for each operational unit.
- **Modular Sampling:** High-cost LLM judges can be applied selectively to high-risk spans (e.g., RAG relevance) rather than re-evaluating full end-to-end traces.

---

### 5. The Evaluation Flywheel & Human-in-the-Loop (HITL)

The core architectural goal of an evaluation platform is establishing a closed-loop **Flywheel Effect** that continuously recycles production reality into pre-production test suites:

```
               ┌─────────────────────────────────────────┐
               │          Production Traffic             │
               │   (Instrumented Traces & Metrics)       │
               └────────────────────┬────────────────────┘
                                    │
                                    ▼
               ┌─────────────────────────────────────────┐
               │        Filtering & Custom Views         │
               │  • Thumbs Down (User Feedback = 0)      │
               │  • High Latency / Error Codes           │
               └──────────┬───────────────────┬──────────┘
                          │                   │
                          ▼                   ▼
┌───────────────────────────────┐   ┌───────────────────────────────┐
│      Direct User Feedback     │   │       Human Review Mode       │
│  (End-user ratings/comments)  │   │  (SME scoring & rubric review)│
└──────────────┬────────────────┘   └───────────────┬───────────────┘
               │                                    │
               └──────────────────┬─────────────────┘
                                  ▼
               ┌─────────────────────────────────────────┐
               │          Dataset Promotion              │
               │ (Add failing traces to offline datasets)│
               └──────────────────┬──────────────────────┘
                                  │
                                  ▼
               ┌─────────────────────────────────────────┐
               │            Offline Evals                │
               │ (Playground iteration & CI/CD testing)  │
               └──────────────────┬──────────────────────┘
                                  │
                                  ▼
               ┌─────────────────────────────────────────┐
               │           Production Release            │
               └─────────────────────────────────────────┘
```

#### Human-in-the-Loop Mechanics
1. **User Feedback Capture:** Log explicit end-user signals (binary thumbs up/down, numeric ratings, free-text complaints) directly as metadata attributes on root spans.
2. **Custom View Triage:** Filter traces to identify high-signal failure cohorts (e.g., `user_feedback == 0`, low automated scores, or tool timeouts).
3. **Human Review Mode:** Subject Matter Experts (SMEs) inspect filtered traces in specialized review interfaces, applying standardized rubrics.
4. **Dataset Promotion:** One-click conversion of problematic production spans into permanent offline dataset test cases.
5. **Organizational Structuring:**
   - *Enterprise Scale:* Hybrid roles combining Product Management with LLM specialization (e.g., Notion's model) dedicated to reviewing traces and curating datasets.
   - *Lean Teams:* Direct engineering review paired with clear, written scoring rubrics to maintain consistency across human scorers.

---

### 6. Evals-as-Code & CI/CD Automation

Evals must integrate into the standard software engineering lifecycle rather than existing as isolated UI experiments:

- **Source Control Prompts and Scorers:** Define prompts, datasets, and scoring functions in code (Python/TypeScript). Version control them alongside application logic in Git.
- **Bidirectional Sync (`push`/`pull`):** Push code-defined assets to the centralized evaluation platform (`braintrust push`) for non-technical stakeholder visibility; pull datasets into local environments for unit testing.
- **CI/CD Regression Gates:** Execute headless evaluations inside pull request pipelines (e.g., GitHub Actions). Set automated thresholds on key metrics (e.g., blocking merges if accuracy drops below baseline).
- **Automated Prompt Optimization (AI Loops):** Leverage automated agentic loops within playgrounds where an optimizer agent modifies prompt templates, runs evaluation suites, evaluates metric diffs, and iteratively hill-climbs on prompt performance.

---

## Conclusion & Best Practices Checklist

- [ ] **Shift from Defense to Offense:** Treat evaluations not merely as regression safety nets, but as the primary engine for rapid experimentation and model upgrades.
- [ ] **Deconstruct into Three Primitives:** Ensure every eval clearly isolates its **Task**, **Dataset**, and **Scorers**.
- [ ] **Instrument Early:** Wrap LLM clients and add trace decorators to functions from day one to capture token usage, latency, and span hierarchies.
- [ ] **Use the 2×2 Diagnostic Matrix:** Calibrate scorers before rewriting prompts; confirm whether low scores reflect application failure or misaligned grading rubrics.
- [ ] **Decouple and Specialize Scorers:** Never use a single monolithic LLM prompt to grade multiple subjective dimensions. Keep scorers modular and targeted.
- [ ] **Tier Judge Models:** Use capable, high-reasoning models for offline and online scoring, even when running lightweight models in production.
- [ ] **Score Intermediate Spans:** Instrument granular trace spans (query transformation, retrieval, tool execution) to isolate cascading failures early.
- [ ] **Close the Flywheel Loop:** Implement custom views for production anomalies and user downvotes, and continuously promote failing traces into offline regression datasets.
- [ ] **Codify into CI/CD:** Version prompts and scorers in Git, and enforce evaluation passes as required checks on pull requests.
