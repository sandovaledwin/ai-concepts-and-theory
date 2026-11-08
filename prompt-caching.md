# Prompt Caching
*[[Based on OpenAI's API Guide](references/raw-prompt-caching)]*

## Introduction

In modern transformer-based LLM architectures, processing long context requires calculating intermediate attention states known as **Key-Value (KV) states** for every input token. Because attention scales quadratically with context length, recalculating KV tensors on every turn of a multi-turn conversation, agent loop, or evaluation run creates massive latency overhead and compute inefficiency.

**Prompt caching** solves this by persisting computed KV tensors across requests that share an identical token prefix. Instead of reprocessing static developer instructions, system context, or prior conversation history, the inference engine reuses saved KV states directly from GPU memory. This yields three primary operational advantages:
1. **Dramatic Latency Reduction:** Cuts Time-to-First-Token (TTFT) by bypassing prefix re-computation.
2. **Substantial Cost Savings:** Reused prefix tokens receive discounts between 90% (0.1× input cost on standard models) and 95% (0.05× on GPT-6.1 Sol and Terra).
3. **Compute Efficiency:** Maximizes GPU throughput and minimizes redundant matrix operations across repeated workflows.

---

## Main Points

### 1. Architectural Foundations: KV Tensors & Prefix Matching

The prompt cache stores **key-value (KV) tensor states**, not raw token text. When a new request arrives, the model checks whether its initial sequence matches an existing cached prefix.

* **Strict Prefix Invariance:** Caching requires an exact, unbroken match starting from token 0. Any modification—including changes to hidden system instructions, developer messages, tools, or parameter configurations—invalidates the cache for all subsequent tokens.
* **Settings That Invalidate the Cached Prefix:**
  * `model`: Different model architectures and snapshots use distinct weights and KV layouts.
  * `tools`: Modifying tool names, descriptions, parameter schemas, ordering, or tool-specific instructions alters the rendered token prefix.
  * `parallel_tool_calls` & `text.verbosity`: Changes system-level formatting instructions injected before user content.
  * `text.format` (Structured Outputs): Altering output JSON schemas modifies system-level grammar constraints. (On GPT-6+, format changes can be appended dynamically without invalidating prior prefixes).
  * `reasoning.effort`: Changing top-level reasoning settings alters system-level chain-of-thought directives.
  * `context_management` (Compaction): Replaces prior history with a condensed representation, permanently resetting prefix matching from the first modified token onward (see [[effective-context-engineering|Context Compaction]]).

```text
Request 1: [System Instructions][Tool Definitions][User Message 1]  → Cached KV Prefix
Request 2: [System Instructions][Tool Definitions][User Message 1][Assistant 1][User 2]
           └──────────────── Reused from Cache ──────────────────┘ └─ New Tokens ──┘
```

---

### 2. Cache Breakpoints: Implicit vs. Explicit Caching

A **cache breakpoint** defines a boundary where the runtime can snapshot KV tensors to storage. How breakpoints are defined varies significantly between model generations:

* **Implicit Caching (Automatic):** The platform automatically places breakpoints at natural structural boundaries. On GPT-5.6+, implicit breakpoints are placed at the end of the latest eligible message:
  * User messages.
  * The final tool response in a consecutive sequence of tool outputs.
  * The final developer message in an initial consecutive block of developer messages.
* **Explicit Caching (Developer-Controlled):** Developers explicitly designate snapshot points using `prompt_cache_breakpoint: { "mode": "explicit" }` within content blocks (`input_text`) and setting `prompt_cache_options.mode: "explicit"`.
  * **Cost Avoidance on Volatile Suffixes:** In explicit-only mode, tokens appearing after the final breakpoint are processed as standard uncached input without incurring a cache-write surcharge.
  * **Multi-Tiered Caching:** A single request can write up to 4 explicit breakpoints, allowing layered caching for prefixes that change at different frequencies (e.g., global instructions vs. session history vs. single-turn inputs).
  * *Constraint:* Top-level `instructions` and `additional_tools` do not accept explicit breakpoints; reusable instructions must be placed in a `developer` message content block.

---

### 3. Model Differences & Generational Evolution

Prompt caching behavior differs substantially between earlier models and the GPT-5.6+ / GPT-6 architecture:

| Dimension | GPT-5.6 and Later (GPT-6 Family) | Earlier Models (GPT-5.5, GPT-5, GPT-4.1) |
| :--- | :--- | :--- |
| **Breakpoint Control** | Implicit and Explicit (`prompt_cache_breakpoint`) | Implicit only |
| **Breakpoint Placement** | End of latest eligible message boundary | Fixed token intervals (e.g., every 2,048 tokens) |
| **Minimum Prefix Length** | Exactly 1,024 visible tokens (excluding hidden tokens) | Variable (depends on tools, reasoning, and settings) |
| **Write Surcharge** | 1.25× (for 30m TTL) or 2.5× (for 12h TTL) | No additional write surcharge |
| **Read Pricing Discount** | 0.10× (0.05× for Sol/Terra) | Model-dependent cached input discount |
| **Retention Control** | `prompt_cache_options.ttl` (`"30m"`, `"12h"`) | `prompt_cache_retention` (`"in_memory"`, `"24h"`) |
| **Accounting & Reporting** | Exact token boundary matching | Excludes hidden tokens; rounds down to multiple of 128 |
| **Routing / Key Role** | `prompt_cache_key` used for tenant isolation / security | `prompt_cache_key` required for cache routing affinity |

---

### 4. Cache Lifetime, Retention & Prewarming

KV cache entries are ephemeral and require deliberate lifecycle management:

* **Time-to-Live (TTL) Configuration:**
  * `"30m"` (Default): Designed for high-frequency multi-turn conversations and agent loops. Billed at **1.25×** uncached input. Every cache hit refreshes the entry's active window by at least another 30 minutes.
  * `"12h"`: Designed for enterprise documentation, shared static knowledge bases (such as in [[contextual-retrieval|Contextual Retrieval]]), or infrequently called agent tools. Billed at **2.5×** uncached input in the Responses API.
* **Mixed Lifetimes within One Request:** Long-lived static developer instructions can be marked with `ttl: "12h"`, while session-specific conversation context downstream uses `ttl: "30m"`. Tokens covered by an active 12-hour write remain valid even if downstream 30-minute writes expire.
* **Prewarming the Cache (`prewarm: true`):** Interactive applications can initialize the prompt cache prior to user engagement. Sending a request with `prompt_cache_options: { "prewarm": true }` writes the KV tensors to GPU storage without generating model output tokens, eliminating cold-start latency for the user's initial interaction.

```json
{
  "model": "gpt-6.1-sol",
  "prompt_cache_options": { "mode": "explicit", "ttl": "30m" },
  "input": [
    {
      "role": "developer",
      "content": [
        {
          "type": "input_text",
          "text": "Enterprise knowledge base, policy docs, and tool schemas...",
          "prompt_cache_breakpoint": { "mode": "explicit", "ttl": "12h" }
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "Multi-turn session history...",
          "prompt_cache_breakpoint": { "mode": "explicit" }
        },
        {
          "type": "input_text",
          "text": "Current user prompt (volatile, no write charge)..."
        }
      ]
    }
  ]
}
```

---

### 5. Economics and the Minimum Cacheable Length Trap

Prompt caching involves an explicit economic trade-off: cache writes incur an upfront surcharge ($w$), while cache reads provide a steep discount ($r$).

#### The Cost Crossover Formula
When an application repeatedly sends a shared prefix of length $L$ that falls just below the minimum cacheable threshold $M$ (1,024 tokens), it may be mathematically cheaper to artificially pad the prompt with useful reference material, [[prompt-engineering-openai|few-shot examples]], or documentation to hit $M$:

$$L_{\mathrm{break\text{-}even}} = M \left( r + \frac{w - r}{N} \right)$$

Where:
* $M$ = Minimum cacheable prefix length ($1{,}024$ tokens).
* $L$ = Original prefix length ($L < M$).
* $w$ = Cache write multiplier ($1.25$ for 30m TTL).
* $r$ = Cache read multiplier ($0.10$ standard, $0.05$ Sol/Terra).
* $N$ = Total number of requests reusing this exact prefix.

#### Economic Decision Rules
* **Expanding Prefix ($L \to M$):** If the original prefix length $L > L_{\mathrm{break\text{-}even}}$, padding the prefix with high-signal context up to $1{,}024$ tokens reduces net cost across $N$ requests.
* **Numerical Example ($M = 1{,}024, w = 1.25, r = 0.10$):**
  $$L_{\mathrm{break\text{-}even}} = 102.4 + \frac{1{,}177.6}{N}$$
  * For $N = 10$ requests: Break-even is $220.2$ tokens. Any prompt longer than 221 tokens is cheaper to expand to 1,024 tokens and cache than to send uncached 10 times.
  * As $N \to \infty$: Break-even approaches $102.4$ tokens. An initial prefix of $\le 102$ tokens never benefits from expansion under these rates.

---

### 6. Cache Infrastructure: Machine Affinity, Routing & Security

Because KV tensors reside in physical GPU VRAM and local storage, request routing dictates cache performance:

* **Machine Affinity & Overflow Routing:** Requests must hit the exact machine holding the unexpired KV tensor. When throughput exceeds approximately **15 requests per minute (RPM)** for a given machine, excess traffic triggers overflow routing to alternative instances, inducing cold cache misses.
* **Role of `prompt_cache_key` Across Generations:**
  * *Legacy Models (GPT-5.5 and earlier):* Developers had to manually supply stable cache keys to hash requests to specific machines, partitioning traffic deterministically below the 15 RPM limit.
  * *Modern Models (GPT-5.6+):* OpenAI manages cache cluster routing automatically based on token prefix hashing and capacity.
* **Multi-Tenant Accounting & Cache-Hit Probing Prevention:** In GPT-5.6+, `prompt_cache_key` serves as an essential isolation boundary:
  * **Tenant Metering:** Isolates token usage and cache accounting by user, organization, or workspace.
  * **Security / Side-Channel Defense:** Prevents **cache-hit probing attacks**, where a malicious user probes candidate prompts to infer—via latency differentials or cached token counts—whether another tenant has processed sensitive data.

---

### 7. Agentic Architectures & Common Implementation Gotchas

#### Best Practices for High Hit Rates
1. **Append-Only Context Discipline:** Multi-turn agents (such as [[effective-harnesses-engineering|long-running coding agents]]) must append user messages, tool calls, and tool outputs strictly to the end of the context array. Never mutate or summarize earlier turns in-place.
2. **Mid-Conversation Configuration Updates:** Changing parameters like `reasoning.effort` or `text.format` at the top level alters the hidden system prompt and destroys cache reuse. In GPT-6+, append a `configuration_update` item to the `input` array instead:
   ```json
   { "type": "configuration_update", "reasoning": { "effort": "high" } }
   ```
3. **Static Tool Schemas & Deferred Loading:** Keep the `tools` array static and identical across all calls. Use `tool_choice: "none"` or `allowed_tools` to disable tools dynamically without altering the prompt prefix. For large tool libraries, use tool search with `defer_loading: true`.

#### Critical Gotchas
* **Gotcha 1: Static Prefix Trapped Behind Dynamic Content:** If an implicit request contains a static developer prompt followed immediately by a dynamic user query, implicit caching places the breakpoint at the end of the *dynamic* query. The next turn with a different query misses the cache entirely.
  * *Remedy:* Place an explicit breakpoint after the static developer prompt.
* **Gotcha 2: Mode Switching Misses Implicit Writes:** If Request 1 writes a cache entry in `mode: "implicit"`, Request 2 in `mode: "explicit"` will not match that boundary unless an explicit breakpoint is set at the identical token location.
* **Gotcha 3: In-Place Message Extension:** Adding text to an existing message (e.g., modifying `User Message 1` from `Content A` to `Content A + B`) pushes the previous breakpoint inside the message block, preventing prefix reuse. Always append a new message.
* **Gotcha 4: Compaction Cache Drop:** Context compaction replaces conversational history with a condensed summary (see [[effective-context-engineering|Effective Context Engineering]]). While this lowers active context length, it resets the prefix match. Balance token reduction against lost cache reuse.

---

### 8. Observability & Cost Accounting

Accurate financial tracking requires parsing the exact token breakdown in the API response:

```json
{
  "usage": {
    "input_tokens": 12000,
    "input_tokens_details": {
      "cached_tokens": 4000,
      "cache_write_tokens": 7000,
      "cache_write_tokens_details": {
        "ttl_30m_tokens": 3000,
        "ttl_12h_tokens": 4000
      }
    }
  }
}
```

#### Cost Calculation Formula
$$\text{Input Cost} = \frac{T_{\mathrm{ord}} \cdot P + T_{\mathrm{cached}} \cdot (r \cdot P) + T_{\mathrm{write,30m}} \cdot (1.25 \cdot P) + T_{\mathrm{write,12h}} \cdot (2.5 \cdot P)}{1{,}000{,}000}$$

Where:
* $T_{\mathrm{ord}} = \text{input\_tokens} - \text{cached\_tokens} - \text{cache\_write\_tokens}$
* $P$ = Base input price per million tokens.
* $r$ = Cache read multiplier ($0.10$ or $0.05$).

---

## Conclusion

Prompt caching shifts context design from simple token minimalism to **prefix lifecycle management**. Maximizing efficiency requires treating the context window as a structured hierarchy:
1. **Static anchor layer (0 to $N$ tokens):** Shared system instructions, reference knowledge, and canonical tool schemas (marked with 12h explicit breakpoints).
2. **Growing session layer:** Multi-turn message history and tool execution results maintained via append-only discipline (marked with 30m implicit/explicit breakpoints).
3. **Volatile leaf layer:** Current user queries and immediate turn parameters placed outside explicit breakpoints to prevent unnecessary write charges.

Mastering this hierarchy allows production systems to operate over massive contexts with up to 90–95% cost reductions and minimal latency overhead.
