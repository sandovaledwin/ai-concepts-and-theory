# How to Write a System Prompt That Actually Works
*[[Based on Pristren's Guide](references/raw-system-prompt-guide-with-examples)]*

## Introduction

A **system prompt** is a persistent instruction block that shapes how a language model behaves across an entire interaction. It transforms a generic model into a specialized, calibrated tool by establishing role boundaries, output formats, and behavioral constraints before user interaction begins.

A well-crafted system prompt reduces per-request engineering, prevents drift, and acts as the first line of defense against hallucinations and sycophancy.

## What a System Prompt Controls

1. **Role and Persona:** Activates relevant domain knowledge and communication style (e.g., "senior backend engineer" vs. generic "helpful assistant").
2. **Behavioral Constraints & Guardrails:** Establishes refusal criteria, topics to avoid, escalation paths to humans, and how to handle unknown facts.
3. **Output Format:** Guarantees structural consistency (Markdown, JSON, schema-specific lists) without relying on per-turn prompts.
4. **Tone and Register:** Enforces conciseness, technical depth, formality, and point of view (e.g., first-person singular).
5. **Knowledge Boundary & Grounding:** Defines authoritative truth (e.g., provided context documents) vs. general training weights.

## Key Patterns Across Production Use Cases

The article outlines six complete prompt patterns, each demonstrating a critical prompting mechanism:

### 1. Customer Support Agent
* **Key Mechanism:** *Escalation triggers & anti-confabulation.*
* **Rule:** Explicitly state what to escalate (billing disputes, data loss, security) and forbid guessing feature capabilities.

### 2. Code Review Assistant
* **Key Mechanism:** *Severity separation & actionable feedback.*
* **Rule:** Enforce a strict structure (`Critical Issues` vs. `Suggestions` vs. `What Works Well`) with line numbers and required code snippets for critical bugs. Require direct phrasing rather than tentative phrasing.

### 3. Research Assistant
* **Key Mechanism:** *Strict grounding & standardized uncertainty markers.*
* **Rule:** Disallow outside knowledge unless requested. Mandate citation tags (`[Document: filename, section X]`) and standardized certainty markers (`"The document states..."`, `"This suggests..."`, `"It is unclear whether..."`).

### 4. Writing Editor
* **Key Mechanism:** *Negative constraints ("What NOT to change").*
* **Rule:** Define what not to touch (voice, intentional stylistic choices, bracketed text) to avoid flattening the author's writing style. Include a required change summary section.

### 5. Data Analyst
* **Key Mechanism:** *Correlation vs. causation guardrails.*
* **Rule:** Disclaim causation explicitly and proactively flag anomalies/data quality issues instead of assuming clean data.

### 6. Project Manager
* **Key Mechanism:** *Atomic hierarchy & structured risk quantification.*
* **Rule:** Structure work into Phases → Modules → Atomic Tasks. Mandate tabular risk tracking (`Risk | Probability | Impact | Mitigation`) and require non-goals in specifications.

## Length and Sizing Guidelines

* **Short (3–5 sentences):** Best for single classification tasks, simple tone calibrations, or basic output formatting.
* **Long (1–2 pages):** Necessary for multi-scenario customer support, rigorous code review checklists, or complex downstream API pipelines.
* **The "Explain Every Line" Test:** If you cannot explain the concrete failure mode prevented by a line in your system prompt, remove it. Move rare edge cases to per-message prompts.

## Provider Differences (OpenAI vs. Anthropic vs. Google)

| Provider / Model | Delivery Mechanism | Behavior & Nuances |
|---|---|---|
| **OpenAI (GPT-4o)** | `messages` array with role: `"system"` (or `developer`) | Exceptionally strict at following formatting constraints and structured outputs. |
| **Anthropic (Claude)** | Top-level `system` parameter | Excels at complex multi-part prompts, long-context fidelity, and expressing calibrated uncertainty when requirements conflict. |
| **Google (Gemini)** | `system_instruction` parameter | Can be less rigid with complex negative constraints; requires thorough validation for strict enforcement. |

## Best Practices Checklist

1. Define a clear role and persona.
2. Specify behavioral constraints and escalation rules.
3. Enforce the output format explicitly.
4. Set tone and register.
5. Establish knowledge boundaries to reduce hallucination.
6. Keep it concise — remove redundant or conflicting instructions.
7. Test with edge cases.
8. Show specific examples of desired vs. undesired behavior.

