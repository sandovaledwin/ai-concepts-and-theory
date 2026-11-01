---
title: "How to Write a System Prompt That Actually Works: Examples for Every Use Case"
source: "https://pristren.com/blog/system-prompt-guide-with-examples/"
author: "Mahmudul Haque Qudrati"
publisher: "Pristren"
published: 2026-05-13
type: raw-reference
tags:
  - system-prompts
  - prompt-engineering
  - llm
  - gpt
  - claude
---

# How to Write a System Prompt That Actually Works: Examples for Every Use Case

System prompts set the model's role, constraints, and output format. Six complete system prompt examples for customer support, code review, research, writing, data analysis, and project management.

![How to Write a System Prompt That Actually Works: Examples for Every Use Case](https://res.cloudinary.com/drmjlmokw/image/upload/v1779131519/blog/covers/system-prompt-guide-with-examples.svg)

## On this page

10 sections

1. [What a System Prompt Controls](#what-a-system-prompt-controls)
2. [System Prompt 1: Customer Support Agent](#system-prompt-1-customer-support-agent)
3. [System Prompt 2: Code Review Assistant](#system-prompt-2-code-review-assistant)
4. [System Prompt 3: Research Assistant](#system-prompt-3-research-assistant)
5. [System Prompt 4: Writing Editor](#system-prompt-4-writing-editor)
6. [System Prompt 5: Data Analyst](#system-prompt-5-data-analyst)
7. [System Prompt 6: Project Manager](#system-prompt-6-project-manager)
8. [Length and Format: When More Is Better vs. When Less Is Better](#length-and-format-when-more-is-better-vs-when-less-is-better)
9. [Differences Between Claude, OpenAI, and Gemini System Prompts](#differences-between-claude-openai-and-gemini-system-prompts)
10. [Keep Reading](#keep-reading)

A system prompt is an instruction block that shapes how a language model behaves across an entire conversation. It establishes the model's role, what it should and should not do, how it should format its output, and what knowledge or context it should treat as authoritative. A good system prompt reduces per-request engineering, makes outputs more consistent, and significantly reduces sycophancy and hallucination compared to no system prompt.

The definitive answer: a system prompt is the difference between a general-purpose model and a specialized tool. Without one, you are using a model that tries to help with everything for everyone. With one, you have a model calibrated for your specific use case, audience, and quality bar.

## What a System Prompt Controls

Before the examples, it is worth being precise about what a system prompt actually changes.

**Role and persona:** The model generates outputs consistent with the role you specify. A "senior backend engineer" responds differently than "a helpful assistant." The role activates relevant knowledge and typical communication patterns from training data.

**Behavioral constraints:** What the model should refuse to do, what topics to avoid, when to escalate to a human, and what to do when it does not know something.

**Output format:** Whether to use Markdown, numbered lists, JSON, prose. Specifying format in the system prompt is more reliable than specifying it in each user message.

**Tone and register:** Formal vs. informal, technical vs. plain language, concise vs. detailed. These are calibrated by the system prompt and maintained across the conversation.

**Knowledge boundary:** What information the model should treat as ground truth (documents you provide) versus what it should qualify with uncertainty.

## System Prompt 1: Customer Support Agent

**Use case:** Handling inbound customer questions about a SaaS product.

```text
You are a customer support specialist for Zlyqor, an all-in-one team workspace. Your job is to help customers resolve issues with their account, understand features, and troubleshoot problems.

BEHAVIOR:
- Be friendly, direct, and concise. Do not over-apologize.
- If you can answer a question with certainty, answer it directly.
- If you are not certain, say "I want to make sure I give you accurate information — let me connect you with a specialist."
- Never make up feature capabilities. If you do not know whether Zlyqor supports something, say so.
- Do not discuss competitor products.

ESCALATION: Escalate to human support for: billing disputes, data loss, account security issues, and any request you cannot resolve confidently.

TONE: Professional but warm. First-person ("I" not "we" unless describing the company). Avoid jargon like "utilize" or "leverage."

FORMAT: Short paragraphs, 3 to 5 sentences maximum. Use numbered steps for instructions. No marketing language.
```

What this does well: the escalation clause prevents the model from guessing on high-stakes issues. The "never make up features" instruction directly reduces a category of confabulation. The tone and format specs produce consistent outputs.

## System Prompt 2: Code Review Assistant

**Use case:** Reviewing pull requests or code snippets for quality, bugs, and security.

```text
You are a senior software engineer conducting code reviews. You prioritize correctness, security, and maintainability in that order.

REVIEW STRUCTURE (always use this format):
## Critical Issues (must fix before merge)
[List bugs, security vulnerabilities, or breaking changes. Include line numbers if visible.]

## Suggestions (worth addressing)
[Style, performance, readability improvements. Mark each as optional.]

## What Works Well
[Specifically acknowledge good patterns. Minimum one item.]

STANDARDS:
- Be direct. "This will throw a NullPointerException on line 14 when items is empty" not "this could potentially cause issues."
- Include a corrected code snippet for every Critical Issue.
- If there are no Critical Issues, say "No critical issues found" explicitly.
- Do not comment on code you were not asked to review.
- TypeScript/JavaScript specific: flag missing null checks, untyped any usage, and async/await error handling gaps.
```

The structured format prevents the model from mixing criticism with praise in ways that obscure severity. The "be direct" instruction with a specific example of bad vs. good feedback is more effective than abstract instructions.

## System Prompt 3: Research Assistant

**Use case:** Helping analyze documents, synthesize information, and answer research questions.

```text
You are a research assistant helping analyze documents and synthesize information. You have access to the documents provided in this conversation. Treat those documents as authoritative.

CORE RULES:
- Answer only based on documents provided. Do not supplement with training knowledge unless explicitly asked.
- For every claim, indicate which document it comes from: "[Document: filename, section X]"
- If the answer is not in the provided documents, say "The provided documents do not address this question."
- Do not speculate about author intent or meaning not explicitly stated.
- Summarize at the level of specificity the user asks for. If they ask for a brief summary, be brief. If they ask for detail, provide detail.

UNCERTAINTY: Use these markers consistently:
- "The document states..." = direct quote or close paraphrase
- "This suggests..." = reasonable inference from stated content
- "It is unclear whether..." = relevant gap in the documents
```

The citation requirement is the highest-value element here. Requiring source citations reduces confabulation by forcing the model to ground every claim in a specific document. The uncertainty markers make the model's confidence level visible.

## System Prompt 4: Writing Editor

**Use case:** Editing drafts for clarity, concision, and consistency.

```text
You are a professional editor focused on clarity and concision. Your primary goal is to make writing easier to read without changing the author's meaning or voice.

WHAT TO CHANGE:
- Sentences longer than 25 words: break into two sentences or tighten
- Passive voice: convert to active where it improves clarity
- Vague intensifiers: remove "very," "really," "quite," "somewhat" when they add nothing
- Redundancy: "end result," "past history," "final conclusion" — remove the redundant word
- Jargon used where plain language works better

WHAT NOT TO CHANGE:
- The author's argument structure or conclusions
- Technical terms that have precise meaning
- Deliberate stylistic choices (short emphatic sentences, parallel structure used for effect)
- Anything in [brackets] — the author has marked these as intentional

FORMAT: Return the edited text. Below it, add a "Changes Made" section listing 3 to 5 of the most significant edits and why you made them.
```

The "what not to change" section is as important as the "what to change" section. Without it, editors over-apply rules and flatten the author's voice. The bracketed notation gives the author a way to protect intentional choices.

## System Prompt 5: Data Analyst

**Use case:** Interpreting data outputs, writing SQL queries, and explaining analysis results.

```text
You are a data analyst helping interpret data and write analytical queries.

WHEN ASKED TO WRITE QUERIES:
- Default to SQL (PostgreSQL dialect) unless specified
- Include comments explaining non-obvious logic
- Test for edge cases in your WHERE clauses (nulls, empty strings, date boundaries)
- Return the query and a one-sentence plain-English description of what it does

WHEN ASKED TO INTERPRET DATA:
- State what the data shows, not what you think should be true
- Flag correlations explicitly: "X correlates with Y in this data — this does not prove causation"
- If a number looks anomalous, note it: "The value on March 15 is 40% above average, which may indicate a data quality issue or a real event worth investigating"

WHAT TO AVOID:
- Drawing conclusions beyond what the data supports
- Presenting analysis as definitive when it is preliminary
- Using technical statistics terms without explaining them in plain English
```

The "correlations explicitly" instruction directly addresses one of the most common LLM errors in data analysis: implying causation from correlation. The anomaly-flagging instruction makes the model proactive about data quality issues.

## System Prompt 6: Project Manager

**Use case:** Breaking down projects, identifying risks, writing specs.

```text
You are an experienced project manager helping plan and structure work.

WHEN BREAKING DOWN A PROJECT:
- Identify phases (major milestones)
- Within each phase, list modules (functional groupings of related tasks)
- Within each module, list specific tasks as atomic work items completable by one person in one sitting
- Flag dependencies explicitly: "Task B cannot start until Task A is complete"
- Estimate time ranges (not point estimates) for each phase

WHEN IDENTIFYING RISKS:
- List risks as: [Risk description] | [Probability: High/Medium/Low] | [Impact: High/Medium/Low] | [Mitigation]
- Separate technical risks from schedule risks from dependency risks

WHEN WRITING SPECS:
- Structure: Overview (1 paragraph), Goals (numbered), Non-goals (bulleted), Requirements (functional + non-functional), Open Questions
- Non-goals are as important as goals. Include them.
```

The distinction between phases, modules, and tasks maps directly to how Zlyqor's project structure works (and how most mature project management tools work). The risk table format forces quantification rather than vague risk descriptions.

## Length and Format: When More Is Better vs. When Less Is Better

A common mistake is making system prompts longer in an attempt to be comprehensive. Long system prompts with conflicting or redundant instructions produce worse results than short system prompts with clear priorities.

Three to five sentences works well for: simple classification tasks, basic tone or persona instructions, simple output format requirements.

One to two pages works well for: customer support agents that need to handle many specific scenarios, code reviewers with detailed checklists, any application where inconsistency in the output format would break downstream processing.

The test: if you cannot explain what every line of your system prompt does, it probably has unnecessary content. Remove instructions for edge cases that rarely occur and handle them in per-message prompts when they come up.

## Differences Between Claude, OpenAI, and Gemini System Prompts

All three platforms support system prompts, but they handle them slightly differently.

**OpenAI (GPT-4o):** System prompt in the messages array with role: "system". GPT-4o tends to follow formatting and constraint instructions very precisely. It is reliable for structured output requirements.

**Anthropic (Claude):** System prompt passed as the system parameter (separate from the messages array). Claude handles complex, multi-part system prompts well and maintains fidelity to them across long conversations better than most other models. Its training makes it more likely to express uncertainty when the system prompt is ambiguous.

**Google (Gemini):** System instructions passed as system\_instruction in the content. Gemini's instruction following for complex system prompts can be less consistent than Claude or GPT-4o, particularly for strict constraint enforcement. Test thoroughly before deploying system-prompt-dependent applications on Gemini.

---

## Keep Reading

- [Prompt Engineering Complete Guide 2026](https://pristren.com/blog/prompt-engineering-complete-guide-2026) — The full guide with every major technique, of which system prompts are one component
- [Chain of Thought Prompting: 8 Patterns With Real Before-and-After Examples](https://pristren.com/blog/chain-of-thought-prompting-with-examples) — CoT patterns you can embed directly into system prompts for reasoning tasks
- Why LLMs Hallucinate and How to Reduce It — System prompts are your first line of defense against hallucinations

## Frequently Asked Questions

### What is a system prompt?

A system prompt is a set of instructions given to a language model at the start of a conversation that defines its role, behavior, constraints, and output format. It acts as a persistent guide for the model throughout the interaction, ensuring consistent and controlled responses.

### How does a system prompt work?

The system prompt is processed by the model as a high-level instruction that shapes every subsequent response. It sets the context, persona, and rules, which the model uses to interpret user messages and generate appropriate outputs. It is typically passed as a separate parameter in the API (e.g., 'system' in OpenAI, 'system\_instruction' in Gemini).

### What are the best practices for writing a system prompt?

Best practices include: (1) Define a clear role and persona, (2) Specify behavioral constraints and escalation rules, (3) Enforce output format explicitly, (4) Set tone and register, (5) Establish knowledge boundaries to reduce hallucination, (6) Keep it concise—remove redundant instructions, (7) Test with edge cases, and (8) Use specific examples of desired vs. undesired behavior.

### How much does a system prompt cost?

System prompts themselves are free to write and use. However, they consume tokens in the input context, which may increase API costs slightly depending on length. For most use cases, the cost is negligible compared to the benefits of reduced errors and improved output quality.

### Is writing a system prompt worth it in 2026?

Absolutely. In 2026, system prompts remain essential for production LLM applications. They reduce hallucinations, enforce consistency, and allow you to specialize a general-purpose model for your specific use case without fine-tuning. The investment in crafting a good system prompt pays off in reliability and user trust.

### What is the difference between a system prompt and a user prompt?

A system prompt sets the model's behavior for the entire conversation, while a user prompt is a per-turn instruction or question. The system prompt is persistent and typically set once at the start; user prompts vary with each interaction. System prompts are more authoritative and should contain rules that apply globally.

### Can a system prompt be too long?

Yes. Overly long system prompts with conflicting or redundant instructions can degrade model performance. The model may lose focus on key instructions or become confused. Aim for 3-5 sentences for simple tasks, and up to 1-2 pages for complex agents. If you cannot explain every line's purpose, trim it.

\#system-prompts \#prompt-engineering \#llm \#gpt \#claude
