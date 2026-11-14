# CCA-F Revision Note — Domain 3: Prompt Engineering & Structured Output

> **Exam weight: ~20%.** On the Architect exam this is about **designing prompts and output contracts for a system** — reliable, parseable, machine-consumable outputs that downstream code can trust — not hand-tuning a one-off prompt.

> **Source note (verified):** Structured-output mechanics (`output_config.format` vs tool-call `strict: true`), prefill behaviour on 4.6+ models, and extended-thinking rules are verified against platform.claude.com structured-outputs / strict-tool-use docs and confirmed during Developer prep.

---

## Contents
1. [Prompt engineering for systems](#1-prompt-engineering-for-systems)
2. [The structured-output decision](#2-the-structured-output-decision)
   - [2.1 output_config.format (response shape)](#21-output_configformat-response-shape)
   - [2.2 strict: true (tool-call inputs)](#22-strict-true-tool-call-inputs)
   - [2.3 Which knob for which job](#23-which-knob-for-which-job)
3. [Prefill — and why it breaks on new models](#3-prefill--and-why-it-breaks-on-new-models)
4. [Extended thinking + structured output](#4-extended-thinking--structured-output)
5. [Context engineering (prompt side)](#5-context-engineering-prompt-side)
6. [Exam traps](#6-exam-traps)
7. [Quick-fire revision](#7-quick-fire-revision)

---

## 1. Prompt engineering for systems

The reliable-prompt toolkit, framed as design choices:

- **Be explicit and specific.** Ambiguity is the main cause of unreliable output. State the task, constraints, and the exact output format.
- **Use XML tags** to delimit sections (`<context>`, `<instructions>`, `<example>`). Claude is tuned for them; they make prompts parseable and reduce bleed between sections.
- **Few-shot / examples** — show 2–5 examples of the exact input→output you want. The single highest-leverage lever for format consistency. Cover edge cases in examples, not just the happy path.
- **Positive instructions** ("respond only with JSON") beat negative ones ("don't add prose") — tell the model what to do.
- **Role / system prompt** sets stable behaviour; put durable instructions in the **system prompt**, per-request data in the user turn.
- **Order matters:** long context first, instructions/question **last** (recency helps instruction-following for long inputs). Pair with prompt caching so the stable prefix is cached.
- **Let it think** for reasoning tasks (step-by-step / extended thinking) — but keep reasoning **out of** the machine-parsed field.

> **Architect lens:** a prompt in a pipeline is a **contract**. The question is usually "why is the downstream parser breaking?" and the answer is a **stronger format guarantee** (structured outputs) plus **examples**, not "add more words asking nicely."

---

## 2. The structured-output decision

Two different mechanisms. Confusing them is a classic multi-select trap.

### 2.1 output_config.format (response shape)

- **`output_config.format`** constrains the **model's response** to a schema (e.g., JSON matching a shape you define).
- Use it when **downstream code consumes the model's answer** and must parse it reliably — the response is guaranteed to fit the schema.
- This is about the **output you read back**.

### 2.2 strict: true (tool-call inputs)

- **`strict: true`** on a **tool definition** makes the model's **tool-call input arguments** grammar-constrained to the tool's `input_schema` — the arguments are **guaranteed valid** against the schema.
- Use it when a **tool/function must receive well-formed arguments** every time (no missing required fields, no wrong types).
- This is about the **inputs the model sends to a tool**.

### 2.3 Which knob for which job

| Requirement | Mechanism |
|---|---|
| "The model's **reply** must be valid JSON my app parses" | **`output_config.format`** |
| "The **tool call arguments** must always match the function schema" | **`strict: true`** on the tool |
| "Loosely nudge toward JSON" | Prompt + examples (no guarantee) |

> **Trap:** a scenario about a **tool receiving malformed arguments** is **`strict: true`**, *not* `output_config.format`. A scenario about **the final answer being unparseable** is `output_config.format`. They are not interchangeable — one governs tool *inputs*, the other governs the *response*.

---

## 3. Prefill — and why it breaks on new models

- **Prefill** = putting words in Claude's mouth by starting the **assistant** turn, historically used to force JSON (prefill `{`) or skip preamble.
- **On Claude 4.6+ models, prefilling the assistant message is no longer supported and returns a 400.** Design doesn't rely on prefill anymore.

> **Architect answer:** to force structured output on current models, use **`output_config.format`** (or `strict: true` for tool inputs) — **not** an assistant prefill. If a scenario shows a 400 after "we prefill `{` to force JSON," the fix is "switch to structured outputs," not "retry" or "escape the prefill."

---

## 4. Extended thinking + structured output

- When extended thinking is on in a tool-use turn, you must **pass the `thinking` AND `redacted_thinking` blocks back verbatim** in the conversation history. **Modifying or dropping them → 400.**
- Keep the **machine-parsed field separate from reasoning**: let the model reason in thinking blocks, and emit the clean structured answer in the response — don't make your parser wade through chain-of-thought.

> **Trap:** "strip the thinking blocks to save tokens before sending the next turn" → **400**. Thinking blocks are part of the required context on 4.x thinking turns; preserve them verbatim.

---

## 5. Context engineering (prompt side)

(Deep reliability/window mechanics are in Note 5; here's the prompt-authoring half.)

- **Put the right things in context, in the right order:** stable system instructions, then reference material, then the specific task last.
- **Few-shot examples are context** — they cost tokens but buy reliability; cache them.
- **Don't over-stuff.** Irrelevant context degrades quality ("context rot") and costs money. Retrieve/seles only what the task needs.
- **Delimit** retrieved/untrusted content clearly (XML tags) so instructions in the data can't be mistaken for your instructions (ties into security: **data is not instructions**).

> **Architect lens:** context engineering is **curation under a budget** — the skill is choosing the *minimum* high-signal context that makes the output reliable, not maximising what you cram in.

---

## 6. Exam traps

- **`output_config.format` vs `strict: true`:** response shape vs tool-call inputs. Don't swap them.
- **Prefill to force JSON on 4.6+** → **400**; use structured outputs instead.
- **Stripping thinking/redacted_thinking blocks** on a thinking turn → **400**.
- **"Add more instructions telling it to output JSON"** when a hard guarantee is needed → weak; use structured outputs.
- **Negative-only instructions** ("don't include X") → prefer positive framing.
- **Instructions buried before a huge context block** → move the task to the **end**.
- **Over-stuffing context** "to be safe" → degrades quality; curate.

---

## 7. Quick-fire revision

- **`output_config.format`** = guarantee the **response** fits a schema (downstream parsing).
- **`strict: true`** = guarantee **tool-call arguments** fit the tool's `input_schema`.
- **Prefill is dead on 4.6+** (400) — force format with structured outputs, not prefill.
- **Thinking + redacted_thinking blocks: pass back verbatim** or 400.
- **XML tags** delimit sections and separate **data from instructions**.
- **Few-shot examples** = the top lever for format reliability; cache the stable prefix.
- **Task/question goes last; long context first.**
- **Positive instructions > negative.** Say what to do.
- **Context engineering = minimum high-signal context under a budget**, not maximum stuffing.
