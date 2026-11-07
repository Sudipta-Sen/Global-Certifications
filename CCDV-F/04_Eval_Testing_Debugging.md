# CCDV-F Revision Note — Domain 4: Eval, Testing & Debugging

> **Exam weight: ~2.6%** (lightest). Only a question or two — but the concepts underpin **Model Selection (Domain 5)**, so they're worth knowing cleanly. Learn the core principles and the tool-call debugging workflow.

---

## Contents
1. [Core concepts](#1-core-concepts)
   - [1.1 Core vocabulary](#11-core-vocabulary)
   - [1.2 Testing vs evaluation (the distinction)](#12-testing-vs-evaluation-the-distinction)
   - [1.3 Why evals matter](#13-why-evals-matter)
2. [Evaluation: designing & grading](#2-evaluation-designing--grading)
   - [2.1 Designing a good eval](#21-designing-a-good-eval)
   - [2.2 The three grading methods](#22-the-three-grading-methods)
   - [2.3 LLM-as-judge done right](#23-llm-as-judge-done-right)
   - [2.4 Success criteria, metrics & the pass bar](#24-success-criteria-metrics--the-pass-bar)
3. [Testing & iteration](#3-testing--iteration)
   - [3.1 Test-driven iteration loop](#31-test-driven-iteration-loop)
   - [3.2 Observability & regression testing](#32-observability--regression-testing)
4. [Debugging](#4-debugging)
   - [4.1 Error categories (triage first)](#41-error-categories-triage-first)
   - [4.2 Debugging tool-call failures (applied favourite)](#42-debugging-tool-call-failures-applied-favourite)
5. [Common traps](#5-common-traps)
6. [Quick-fire revision](#6-quick-fire-revision)

---

## 1. Core concepts

### 1.1 Core vocabulary
- **Evaluation (eval)** — a systematic way to **measure** how well a model/prompt performs against **success criteria**, over a **dataset** of cases. Produces a **score/metric**, not just pass/fail.
- **Test case** — one input (+ often a **golden answer / ground truth**) run through the system to produce an output to grade.
- **Success criteria** — what "good" means, made **Specific, Measurable, Achievable, Relevant** (see §2.4).
- **Grader / grading method** — how each output is scored: **code-based**, **human**, or **LLM-based** (see §2.2).
- **Pass bar / target metric** — the threshold the aggregate score must clear (e.g. "≥90% correct").
- **Held-out set** — cases kept aside so you don't **overfit** the prompt to the examples you keep reading.
- **Regression test** — re-running the eval suite on every change to catch quality drops (see §3.2).

### 1.2 Testing vs evaluation (the distinction)
Both ask *"does it work?"* — but they answer different questions, and the exam domain bundles them as **"Eval, Testing & Debugging."**

| | **Testing** | **Evaluation (evals)** |
|---|---|---|
| Question | *"Does the code behave correctly?"* | *"How **well** does the model perform?"* |
| Answer shape | **Binary** pass/fail per case | **Graded / statistical** — a score or % across a dataset |
| Target | **Deterministic** parts (tool functions, parsers, routing, schema validity) | **Non-deterministic** model output (quality, tone, correctness) |
| Repeatability | Same input → same result every time | Same input can vary → judge **aggregate** behaviour, not one run |
| Typical tools | Unit / integration tests, assertions, CI | Eval harness + graders (code / human / LLM-judge) |
| Fails when | A function returns the wrong value | Quality drops below the **pass bar** |

- **Deterministic logic → test it** (exact assertions). **Model behaviour → evaluate it** (graded, over many cases) because output is **non-deterministic**, so a single "eyeball it once" run proves nothing.
- **Debugging** is the third leg: once a test/eval **fails**, debugging is **diagnosing *why*** (classify the error — §4.1 — then drill in, e.g. the tool-call workflow in §4.2) — testing/evals *detect*, debugging *explains*.
- In practice they **layer**: deterministic unit tests on your tool/pipeline code **plus** evals on the model's output, and the whole eval suite doubles as a **regression test** (§3.2).

> **Exam framing:** don't treat "testing" and "evaluation" as synonyms. **Testing = deterministic, binary, per-case**; **evaluation = graded, statistical, handles non-determinism.** Anthropic's docs group both under *"Test and evaluate"* and centre on **evaluations** for model quality.

### 1.3 Why evals matter
LLM outputs are **non-deterministic**, so "eyeball it once" doesn't work. **Evals** are the systematic way to:
- **Measure quality** against criteria,
- **Catch regressions** when you change a prompt/model/temperature,
- **Justify** a model/version choice (the backbone of right-sizing in Domain 5).

> Treat prompt development like **test-driven development**: write the eval, then iterate the prompt/model against it.

---

## 2. Evaluation: designing & grading

### 2.1 Designing a good eval
- **Representative, diverse test cases** — cover the **real input distribution**, including **edge cases** and known-hard examples. One cherry-picked easy case gives **false confidence**.
- **Decide grading criteria *before* looking at outputs** (avoids bias).
- **Volume over polish** — a **lot of auto-graded** cases beat a **handful of hand-graded** ones. Why: more cases cover more of your real inputs (better signal), and — the key part — **auto-graded suites re-run for free on every prompt/model change**, while hand-graded ones mean re-paying humans each time, so they quietly stop getting used. Trade a little per-case precision for many cases you can **actually replay**.
- **Task-specific** — grade what actually matters (format validity, factual correctness, tone, safety), not a generic "good/bad."
- Keep a **held-out set** so you don't overfit the prompt to the cases you keep reading.

### 2.2 The three grading methods
| Method | How | Best for | Trade-off |
|---|---|---|---|
| **Code-based / exact-match** | Deterministic checks: exact match (`output == golden`), **string match** (key phrase in output), regex, **JSON-schema validity**, cosine similarity, ROUGE-L, numeric tolerance | Clear right/wrong answers | *"Fastest and most reliable, extremely scalable"* — but **lacks nuance** for complex judgment |
| **Human grading** | People score against a rubric | Subtle quality, final sign-off | *"Most flexible and high quality, but slow and expensive"* — **avoid if possible** |
| **LLM-based (LLM-as-judge)** | A model scores outputs against a **rubric** | Open-ended quality at scale | *"Fast and flexible, scalable, suitable for complex judgment"* — **test reliability first, then scale** |

- **Prefer code-based grading** whenever the answer is checkable — the docs call it *fastest and most reliable*.
- Use **LLM-based** grading for open-ended outputs; use **humans** for the hardest/most sensitive calls (avoid when you can — slow + costly).
- *(The docs name the three methods exactly **code-based**, **human**, **LLM-based** — not "model-based." Each criterion maps to a method, e.g. tone → LLM Likert 1–5, PHI privacy → LLM yes/no, sentiment → exact match.)*

### 2.3 LLM-as-judge done right
- Give the judge a **detailed, clear rubric** (what counts as pass/fail, with examples) — the docs' #1 tip. One success criterion may need **several rubrics**.
- **Be empirical/specific:** make it output *only* `correct`/`incorrect`, `yes`/`no`, or a **1–5 scale** — *"Purely qualitative evaluations are hard to assess quickly and at scale."* (Pull the value out of a tag, e.g. `<result>…</result>`.)
- **Encourage reasoning:** run the grader **with thinking on** so it reasons before scoring — the docs say this *"increases evaluation performance, particularly for tasks requiring complex judgment."*
- **Use a *different* model as the judge** than the one that produced the output (docs: *"best practice to evaluate with a different model"*).
- **Validate the judge** against a sample of human grades before trusting it at scale (*"test to ensure reliability first, then scale"*).
- A **cheap model (Haiku)** often suffices as the judge — ties into cost (Domain 5).

### 2.4 Success criteria, metrics & the pass bar
- Start from **success criteria** that are **Specific, Measurable, Achievable, Relevant** (the docs' framing). Bad: *"classify sentiments well."* Good: *"F1 ≥ 0.85 on a held-out set of 10,000 diverse posts."*
- Define a **target metric and pass bar up front** (e.g. "≥95% schema-valid", "≥90% judged correct"); ground the bar in **benchmarks / prior experiments / feasibility**, not vibes.
- **Most use cases need multidimensional evaluation** — several criteria at once (e.g. F1 ≥ 0.85 **and** 99.5% non-toxic **and** 95% latency < 200 ms).
- Common criteria (docs): **task fidelity, consistency, relevance/coherence, tone/style, privacy preservation, context utilisation, latency, price.**
- Track **quality + cost + latency** together — the eval is also how you compare **tiers** (Domain 5).

---

## 3. Testing & iteration

### 3.1 Test-driven iteration loop
```
Write eval cases  →  run current prompt/model  →  measure against criteria
      ↑                                                     |
      └──────────  refine prompt / swap model  ←────────────┘
```
> **Golden rule: change ONE variable at a time** (prompt wording, examples, model tier, temperature) so you can **attribute** the effect. Changing prompt *and* model together tells you nothing about which helped.

### 3.2 Observability & regression testing
- **Log everything** needed to reproduce: full request (system, messages, tools), response, `stop_reason`, `usage`, and timings.
- Use **`metadata`** / request IDs for traceability (also a Domain 7 safety practice).
- **Re-run the eval suite** on every prompt/model/version change → a **regression test** for your LLM app.
- **Monitor in production** (quality signals, refusals, errors, latency, cost) and feed failures back into the eval set.
- Watch for **regressions**: a prompt tweak that helps one case can hurt others — the full suite catches it.

---

## 4. Debugging

### 4.1 Error categories (triage first)
Before fixing anything, **classify the failure** — each category has a *different* fix, and retrying the wrong one wastes time (or money).

| Category | What it is | Typical signals | Fix | Retry? |
|---|---|---|---|---|
| **Integration error** | Your code/wiring is wrong — bad params, auth, endpoint, malformed `tool_result`, schema mismatch | HTTP **400/401/403/404/413** (`invalid_request_error`, `authentication_error`, `permission_error`, `not_found_error`, `request_too_large`) | **Fix the request** (params, key, payload, schema) — see **Note 02** error table | ❌ No — a retry just repeats it |
| **Infrastructure error** | Transient platform/network problem on Anthropic's side or in transit | HTTP **429** `rate_limit_error`, **500** `api_error`, **529** `overloaded_error`, **504** `timeout_error`, connection errors | **Retry with exponential backoff + jitter**; honour `retry-after`. SDKs **auto-retry** transient 5xx/429 **twice by default** | ✅ Yes (bounded) |
| **Model-output error** | The request *succeeded* but the **content** is wrong — hallucination, wrong format, refusal, incomplete/truncated `tool_use`, low quality | HTTP **200** with bad output; `stop_reason: "refusal"`; incomplete `tool_use` from `max_tokens`; schema-invalid JSON | **Fix with prompting / evals / schema / higher `max_tokens`** — not retries | ⚠️ Only after **changing** the prompt/params |

**1 · Integration error — _"your letter got bounced at the door."_**
Claude never even sees the request — your code addressed the envelope wrong. Maybe you forgot `max_tokens`, sent the wrong types for a tool's `input_schema`, used a bad key (`401`), a key with no access (`403`), the wrong model ID (`404`), or an oversized payload (`413`).

The tell: it's a **`4xx`**, and it fails **the exact same way every single time**. So don't retry — you'll just bounce again. Fix the request, then resend. (Full code→type table in **Note 02**.)

**2 · Infrastructure error — _"it's not you, the line's just busy."_**
Your request was perfect; the plumbing hiccuped. Anthropic's servers were briefly swamped (`529`), threw an internal wobble (`500`), timed out (`504`), the connection dropped, or you knocked too fast and hit a rate limit (`429`).

The tell: it usually **fixes itself**. So **wait and retry** — exponential backoff + jitter, honouring `retry-after`. The SDKs even do it for you (twice by default). **One gotcha:** if that `429` is a **spend cap**, backing off won't save you — there's no `retry-after` and it keeps failing **until the bill resets**. That's a wallet problem, not a blip (Domain 7).

**3 · Model-output error — _"Claude answered… it just answered badly."_**
Green light, `200 OK` — and the content is still wrong. This is the one **evals exist for**. It shows up as a **hallucination** (confident nonsense), the **wrong format** (not your schema), a **refusal** (`stop_reason: "refusal"` — and it's a normal **200**, so error-rate dashboards sail right past it; Domain 7), a **cut-off answer** (hit `max_tokens`, leaving a half-finished `tool_use`), or just a **meh** reply that missed the point.

The tell: hitting "retry" changes nothing — same prompt, same bad answer. The fix is upstream: sharpen the prompt, add examples, tighten the schema, raise `max_tokens`, or move up a tier (Domains 5–6) — then **prove it worked with an eval** (§2), not a vibe check on one run.

> **Triage order:** read the **HTTP status first** (integration vs infrastructure); if it's **200**, inspect the **content / `stop_reason`** (model-output). **Retrying only helps infrastructure errors.**

> **Exam framing:** *integration → fix the request · infrastructure → back off & retry · model-output → fix the prompt/schema.* The classic trap is **retrying a 4xx** or a **model-output error** — neither improves on retry.

*Doc-verified: the HTTP codes/types and that SDKs auto-retry transient connection/rate-limit/5xx errors with exponential backoff (twice by default, honouring `retry-after`); the spend-cap-429 caveat; refusal = HTTP 200. The three-way category naming (integration / infrastructure / model-output) is a debugging framing that organises those doc facts, not a single doc-named taxonomy.*

### 4.2 Debugging tool-call failures (applied favourite)
Most tool failures are **integration errors** (schema/arguments) or **model-output errors** (a truncated `tool_use`) from §4.1 — not model "stupidity." Work through it systematically:
1. **Inspect the generated tool `input`** against the tool's **`input_schema`** — wrong types, missing required fields, invalid JSON?
2. **Improve the tool `description` and schema** — ambiguous descriptions cause wrong/garbled calls. **Clear descriptions are the single biggest lever** on tool-use quality (Domain 8).
3. **Return `is_error: true`** tool results with a helpful message so the model can **self-correct** instead of trusting bad output.
4. **Check `stop_reason`** — e.g. `max_tokens` leaving an **incomplete `tool_use` block** (the docs' term; the partial JSON won't parse) → **retry with a higher `max_tokens`**; or a `tool_use` you forgot to handle; or `pause_turn` not continued (Domain 2).
5. **Use structured output** to make behaviour **observable and assertable**.
6. **Log the full message history** (including the `tool_use`/`tool_result` turns) to **reproduce** the failure deterministically.

> **Trap:** don't blame the model first — **check the schema, arguments, and `stop_reason`** before anything else.

---

## 5. Common traps
- Grading **"by vibes"** after shipping is **not** an eval.
- Testing only **one easy example** gives false confidence.
- Changing **prompt *and* model at once** → you can't tell what helped.
- **Blaming the model** before checking the schema/arguments/`stop_reason`.
- Forgetting that hitting `max_tokens` leaves an **incomplete `tool_use` block** (partial, unparseable JSON) — detect via `stop_reason` and **retry with a higher `max_tokens`**.
- Using an LLM judge **without a rubric** or without validating it against humans.
- Treating **testing and evaluation as the same thing** — one is deterministic/binary, the other graded/statistical (§1.2).

---

## 6. Quick-fire revision
- Q: Testing vs evaluation? **A:** **Testing** = *does the code behave correctly?* — **deterministic, binary pass/fail** per case. **Evaluation** = *how well does the model perform?* — **graded/statistical** over a dataset, handling **non-determinism**.
- Q: Where does debugging fit? **A:** Testing/evals **detect** failure; **debugging diagnoses *why*** (e.g. input-vs-schema, `stop_reason`).
- Q: Three error categories & their fixes? **A:** **Integration** (4xx — fix the request), **infrastructure** (429/5xx — back off + retry), **model-output** (200 but bad content — fix prompt/schema). Retrying only helps **infrastructure**.
- Q: Which error type should you NOT retry? **A:** **Integration (4xx)** and **model-output** errors — a retry repeats the same failure; only **infrastructure** (429/5xx/timeouts) is retry-worthy.
- Q: What do you test vs what do you evaluate? **A:** **Test** deterministic logic (tool functions, parsers, schema validity); **evaluate** non-deterministic **model output** (quality, tone, correctness).
- Q: Why evals (not eyeballing)? **A:** Outputs are **non-deterministic** — measure quality, catch regressions, justify model choice.
- Q: Three ways to grade an eval? **A:** **Code/exact-match**, **human**, **LLM-based (LLM-as-judge)**.
- Q: Which grading is fastest/most reliable? **A:** **Code/exact-match** (use it whenever the answer is checkable).
- Q: Which is highest-quality but least scalable? **A:** **Human** grading (avoid if possible — slow + costly).
- Q: Key to a good LLM-as-judge? **A:** A **detailed rubric** + a **specific** output (correct/incorrect, 1–5); use a **different** model, and **validate** against human grades.
- Q: Rule when iterating on prompts? **A:** **Change one variable at a time**; measure against fixed criteria.
- Q: First step when a tool call keeps failing? **A:** Compare the generated **`input`** to the **`input_schema`**.
- Q: Biggest lever on tool-use quality? **A:** A **clear tool `description`** (+ schema).
- Q: How does the model recover from a failed tool call? **A:** Return **`is_error: true`** so it can retry/adjust.
- Q: A `tool_use` block is incomplete/unparseable — likely cause & fix? **A:** **`max_tokens` cut it off** (`stop_reason: "max_tokens"`) → **retry with a higher `max_tokens`**.
- Q: What makes an LLM app's eval a regression test? **A:** **Re-running the suite** on every prompt/model/version change.

**One-line summary:** *Evals = **representative cases + predefined grading** (code > human > LLM-judge by scalability); iterate **one variable at a time**; debug tool calls by checking **input-vs-schema and `stop_reason`** first and returning **`is_error`** for recovery.*