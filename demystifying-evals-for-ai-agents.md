# Demystifying Evals for AI Agents
*[[Based on Anthropic's Research](references/raw-demystifying-evals-for-ai-agents)]*

## Introduction
Building AI agents without evaluations leads to reactive debugging loops—catching issues only after users complain and risking regressions with every patch. Evals make behavioral changes visible before deployment, create a high-bandwidth communication channel between product and research teams, and enable fast adoption when upgrading to new frontier models.

## Core Concepts & Terminology
- **Task (Problem / Test Case):** A single test scenario with defined inputs and explicit success criteria.
- **Trial:** A single execution run of a task. Running multiple trials is essential to measure variance.
- **Grader:** Code, model, or human logic scoring specific aspects of performance via assertions/checks.
- **Transcript (Trace / Trajectory):** The complete multi-turn interaction record (API messages, tool calls, arguments, reasoning, and environment outputs).
- **Outcome:** The final external environment state at the end of a trial (e.g., database records, modified codebase, committed files)—distinct from what the agent claims in the transcript.
- **Agent Harness (Scaffold) vs. Eval Harness:**
  - *Agent Harness:* The runtime system enabling the model to act as an agent (prompt loops, tool execution, state management).
  - *Eval Harness:* The testing infrastructure running trials concurrently, isolating environments, executing graders, and aggregating metrics.
- **Evaluation Suite:** A cohesive collection of tasks measuring a capability or behavioral category.

## Grader Types
Agent evaluations combine three grader archetypes:
1. **Code-Based Graders:**
   - *Methods:* Deterministic unit tests, string matches, static analysis (linters, type checkers), state verification, transcript tool-call checks.
   - *Pros/Cons:* Fast, cheap, reproducible, objective; but brittle to valid creative variations and unsuitable for open-ended subjective tasks.
2. **Model-Based Graders (LLM-as-a-Judge):**
   - *Methods:* Rubric-based scoring, natural language assertions, pairwise comparisons.
   - *Pros/Cons:* Flexible, scalable, captures nuance in freeform outputs; but non-deterministic, costlier, and requires ongoing human calibration.
3. **Human Graders:**
   - *Methods:* SME review, crowdsourced labels, spot checks, A/B testing, inter-annotator agreement.
   - *Pros/Cons:* Gold standard for quality and real-world alignment; but slow, expensive, and difficult to scale.

## Capability vs. Regression Evals
- **Capability (Quality) Evals:** Ask *"What can this agent do?"* Designed with a low initial pass rate to test difficult capabilities and give teams a benchmark to hill-climb.
- **Regression Evals:** Ask *"Does the agent still do what it used to?"* Designed with near-100% pass rates to guard against backsliding.
- **Graduation:** As agents improve, solved capability evals "graduate" into the regression suite.

## Evaluating Agent Architectures by Domain
- **Coding Agents:** Primarily verified via deterministic execution (e.g., SWE-bench Verified, Terminal-Bench) where solutions must pass tests without breaking existing code, augmented with LLM rubrics for code style and quality.
- **Conversational Agents:** Evaluated on both task completion and interaction dynamics (empathy, conciseness, turn limits). Often requires a second LLM acting as a simulated user (e.g., $\tau$-Bench / $\tau^2$-Bench).
- **Research Agents:** Graded on synthesis quality, factual groundedness in retrieved citations, coverage of key facts, and authoritative source selection (e.g., BrowseComp).
- **Computer Use Agents:** Require sandbox environments (OS/browser) checking final artifact state (URL, database, DOM/UI elements, filesystem) via frameworks like WebArena or OSWorld.
- **Token Efficiency Trade-off:** DOM-based browser interactions are fast but token-heavy; screenshot-based interactions are slower but more efficient.

## Accounting for Non-Determinism: pass@k vs. pass^k
Because agent runs vary across trials, two distinct metrics capture different requirements:
- **`pass@k`:** Probability of getting at least **one** successful solution in $k$ attempts. As $k$ grows, `pass@k` climbs toward 100%. Useful for coding/ideation tools where a single working answer is sufficient.
- **`pass^k`:** Probability of succeeding on **all** $k$ trials ($p^k$). As $k$ grows, `pass^k` drops sharply (e.g., 75% per trial → ~42% over 3 trials). Crucial for customer-facing production agents where consistency is mandatory.

## Roadmap: Going from Zero to One
1. **Start early:** Begin with 20–50 tasks drawn from real failures rather than waiting for hundreds. Early on, large effect sizes make small samples sufficient.
2. **Use manual checks & bug logs:** Turn user complaints and QA checklists directly into eval cases.
3. **Write unambiguous tasks with reference solutions:** Ensure two domain experts would reach the same pass/fail verdict. Create reference solutions to prove tasks are solvable and graders are configured correctly.
4. **Build balanced problem sets:** Include both positive and negative cases (e.g., queries requiring search vs. queries answerable from memory) to prevent over- or undertriggering.
5. **Isolate environments:** Ensure a clean slate for each trial to eliminate cross-trial state leakage (e.g., git history, cache, resource exhaustion).
6. **Grade outcomes over paths:** Grade what the agent achieved, not the exact tool sequence taken—rigid path checks punish valid creativity. Build in partial credit.
7. **Read transcripts:** Regularly inspect traces to distinguish genuine model errors from grading bugs, ambiguous specs, or harness constraints.
8. **Monitor for saturation:** When scores hit near 100%, the eval tracks regressions but provides no improvement signal; add harder tasks.
9. **Eval-driven development:** Treat eval suites as living code owned by product teams and domain experts, supported by centralized infra teams. Define evals for planned capabilities before the agent can fulfill them.

## Holistic Understanding: The Swiss Cheese Model
No single testing method catches every bug. Reliable production agents require overlapping evaluation layers:
- **Automated Evals:** Rapid CI/CD regression testing on every change and model upgrade.
- **Production Monitoring:** Real-world metrics and runtime drift detection post-launch.
- **A/B Testing:** Measuring impact on user behavior and product KPIs once there's sufficient traffic.
- **Transcript Sampling & User Feedback:** Qualitative discovery of edge cases; triage constantly, read samples weekly.
- **Systematic Human Studies:** Calibrating LLM judges and validating subjective, high-stakes domains.
