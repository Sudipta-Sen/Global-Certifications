# CCA-F Revision Note — Domain 5: Context Management & Reliability

> **Exam weight: ~15%.** This is a **new standalone domain** on the Architect exam and — per your Developer scorecard (Context Engineering 50%, Cost/Token 50%, Debugging 33%) — your **weakest cluster**. Spend disproportionate prep time here. It's about making agentic systems **stay within the window, stay affordable, and stay correct**.

> **Source note (verified):** Window/stop-reason behaviour, prompt caching, cumulative streaming usage, retry/error semantics, and tokenizer changes confirmed during Developer prep against platform.claude.com docs.

---

## Contents
1. [The context window as a budget](#1-the-context-window-as-a-budget)
2. [Managing a growing conversation](#2-managing-a-growing-conversation)
3. [Cost & token management at architecture scale](#3-cost--token-management-at-architecture-scale)
   - [3.1 Prompt caching](#31-prompt-caching)
   - [3.2 Token counting & the newer tokenizer](#32-token-counting--the-newer-tokenizer)
4. [Reliability patterns](#4-reliability-patterns)
   - [4.1 Validation–retry loops](#41-validationretry-loops)
   - [4.2 Confidence calibration & provenance](#42-confidence-calibration--provenance)
   - [4.3 Escalation triggers](#43-escalation-triggers)
5. [Error handling for reliability](#5-error-handling-for-reliability)
6. [Exam traps](#6-exam-traps)
7. [Quick-fire revision](#7-quick-fire-revision)

---

## 1. The context window as a budget

The window holds **everything**: system prompt, CLAUDE.md, tool definitions, conversation history, tool results, retrieved docs, thinking blocks. It's finite, and **the API is stateless** — each turn resends the **entire** history, so context **grows every turn** and so does cost and latency.

Key behaviours:
- **On 4.5+ models, input + `max_tokens` exceeding the window returns `stop_reason: "model_context_window_exceeded"`** — a **200 with a stop reason**, not a pre-generation 400. Fix by shrinking input or `max_tokens`.
- **"Context rot":** as the window fills with low-signal content, output quality degrades even before you hit the limit. More context is not better.

> **Architect lens:** context is a **budget to curate**, not a bag to fill. The recurring correct answer is **reduce / isolate / summarise** context — not "use a model with a bigger window" (that defers the problem and raises cost).

---

## 2. Managing a growing conversation

Strategies, and when each is the right answer:

| Strategy | What it does | Use when |
|---|---|---|
| **Compaction / summarisation** | Replace old turns with a summary | Long sessions; old detail no longer needed verbatim |
| **Sub-agent isolation** | Run noisy side-work in a sub-agent, return only a summary | A task would flood the main window (see Note 1) |
| **Retrieval (RAG)** | Fetch only the relevant slice on demand | Knowledge base too big to inline |
| **Selective context** | Include only high-signal material, delimited | Always — curate every turn |
| **Externalise state** | Store state in files/DB, not the transcript | Durable or large state |

> **Sub-agents are a context-management tool, not just an orchestration tool.** "The main agent's context fills up with file dumps / test output" → delegate that work to a **sub-agent** so only the summary returns. This is both a Domain-1 and a Domain-5 answer.

> **Trap:** "the conversation is getting too long, so raise the token limit / switch to a bigger-window model." The precise answer is usually **compaction, retrieval, or sub-agent isolation** — manage the context, don't just enlarge the container.

---

## 3. Cost & token management at architecture scale

Cost is driven by **tokens in + tokens out, every turn**, times volume. Architect-level levers:

- **Right-size the model per task:** cheap/fast model (Haiku) for classification/routing/extraction; mid (Sonnet) for most work; top (Opus) only where the task needs it. Use **routing** so expensive models handle only the hard slice.
- **Prompt caching** for repeated stable prefixes (below).
- **Batch** non-urgent work via the Message Batches API for a discount (where supported — **not on Bedrock**).
- **Trim context** (compaction/retrieval) — fewer input tokens per turn compounds across a long session.
- **Cap output** with sensible `max_tokens`; stream long outputs.

### 3.1 Prompt caching

- Caches a **stable prefix** (system prompt, tool defs, long context) so repeated calls **don't re-pay full input cost** for it — **token/cost efficiency**.
- Put **stable content first**, variable content last, so the cached prefix is reused.
- **Works on Bedrock**; **Message Batches and the Files API do NOT** (platform parity differs — know this).

> **Caching vs file_id — don't conflate:** **prompt caching** reduces **token cost** for repeated context. **`file_id`** (Files API) is **transfer/upload efficiency** — reuse an uploaded file without re-sending bytes — it does **not** reduce the token count the file contributes. Different problems.

### 3.2 Token counting & the newer tokenizer

- **Count before you send** with the `count_tokens` endpoint — **pass the target model**, because tokenization differs by model.
- **4.7+ / 5.x use a newer tokenizer** that can produce **~30% more tokens** for the same text — don't estimate from an old model's counts; **recount** against the actual model. Budgeting on stale counts under-provisions the window.

> **Trap:** "estimate tokens with a rule of thumb (~4 chars/token)" when precision matters → **count with `count_tokens` for the specific model**. Count, don't estimate — especially across tokenizer changes.

---

## 4. Reliability patterns

### 4.1 Validation–retry loops

The core reliability construct: **generate → validate → on failure, retry with the error fed back → bounded retries → escalate.**

- Validate output against a **schema, rules, or an independent check** before trusting it.
- On failure, **feed the specific error back** so the model can self-correct (not a blind retry of the same prompt).
- **Bound the retries** (max attempts) and **escalate** when exhausted — don't loop forever.
- This is the **evaluator–optimiser** pattern applied to reliability.

> **Trap:** "retry the same request on a bad output" (blind retry) vs **"validate, feed the error back, retry, then escalate."** The precise answer includes the **validation + bounded + escalation** structure, not just "try again."

### 4.2 Confidence calibration & provenance

- **Don't trust model self-reported confidence** as a reliability control — models are often confidently wrong. Prefer **independent verification** (a validator, a second check, ground-truth comparison).
- **Provenance / grounding:** have the agent **cite its source** (which document/tool result an answer came from) so claims are checkable and hallucinations are catchable. Ground answers in retrieved evidence rather than parametric memory for factual tasks.
- Design the system to **surface uncertainty** (ask/escalate) instead of fabricating.

> **Self-check vs independent check:** "ask the model if it's sure" is weak. "Verify against the source / a validator" is strong. Verification beats introspection.

### 4.3 Escalation triggers

Escalate to a human or higher-tier model when:
- Confidence is low or the request is ambiguous.
- Repeated tool/validation failures.
- An iteration/budget cap is hit.
- The action is **irreversible or high-impact** (money, data deletion, external comms) → **human-in-the-loop gate**.
- The task falls outside policy/scope.

---

## 5. Error handling for reliability

Architect-level error posture (the 4xx/5xx reflex, applied to system design):

- **4xx = client error → fix, don't retry** (400 malformed, 401 auth, 403 perms, 404 not found, 413 too large).
- **5xx = server error → retry with exponential backoff + jitter** (500, 529 overloaded, 504 gateway timeout).
- **429 is the hinge:** retry **only** when there's a **`retry-after`** header; a spend-cap 429 (`enforced_spend_limit_reached`, **no** retry-after) is a **hard stop — fix the account, don't retry**.
- **Streaming reliability:** `message_delta` carries top-level `stop_reason` + **cumulative** `usage` (don't sum events); `ping` keep-alives hold the connection; distinguish **idle/read timeout** from **absolute/max-duration timeout**. For long jobs, **stream** so you don't hold one long request past a gateway limit.
- **Idempotency & retries:** design retried operations to be safe to repeat (don't double-charge, double-send).

> **Trap:** blanket "retry with backoff on any error." Retrying a **4xx** (including a spend-cap 429) just reproduces it. Sort **4xx vs 5xx** first, then check the **429 hinge**, then watch for a **`stop_reason` masquerading as an error** (`model_context_window_exceeded`, `pause_turn`).

---

## 6. Exam traps

- **"Bigger window / bigger model"** to fix a full context → usually wrong; **compact / retrieve / isolate**.
- **Blind retry** on bad output → use **validate → feed error back → bounded → escalate**.
- **Trusting self-reported confidence** → use **independent verification + provenance**.
- **Estimating tokens** when precision matters → **`count_tokens` with the model**; 4.7+/5.x ≈ +30%.
- **Caching vs file_id** conflation → caching = token cost; file_id = transfer.
- **Prompt caching on Bedrock works; Batches/Files don't.**
- **Retrying a 4xx / spend-cap 429** → fix, don't retry.
- **No escalation path** on irreversible actions → add a **human-in-the-loop gate**.

---

## 7. Quick-fire revision

- **Stateless API → full history resent each turn → context & cost grow.** Curate the budget.
- **4.5+: input + max_tokens over window = `model_context_window_exceeded` stop_reason (200), not 400.**
- **Manage context: compaction, RAG, sub-agent isolation, externalise state** — not "bigger window."
- **Right-size models + route**; cheap model for easy slices, top model only where needed.
- **Prompt caching = token cost savings** (stable prefix first). **file_id = transfer efficiency**, not token reduction.
- **Count tokens with `count_tokens`, pass the model**; 4.7+/5.x newer tokenizer ≈ +30%.
- **Reliability = validate → feed error back → bounded retry → escalate.**
- **Verify independently + cite provenance**; don't trust self-confidence.
- **Escalate on low confidence, repeated failure, caps, and irreversible actions (human gate).**
- **4xx fix / 5xx retry / 429 hinge (retry-after) / spend-cap 429 = hard stop.**
- **Streaming: `message_delta` usage is cumulative; idle vs absolute timeout; stream long jobs.**
