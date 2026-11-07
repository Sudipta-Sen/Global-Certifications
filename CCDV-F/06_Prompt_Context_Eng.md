# CCDV-F Revision Note — Domain 6: Prompt & Context Engineering

> **Exam weight: ~11.0%**. Best-practices and *"which technique fits here?"* questions. Know the **techniques, when each applies, and context-window discipline**. (Verified against Anthropic's current *Prompting best practices*, *Long-context tips*, *Structured outputs* and *Context windows* docs — guidance has shifted on newer models, flagged inline.)

---

## Contents
1. [Core concepts](#1-core-concepts)
   - [1.1 Prompt engineering vs context engineering](#11-prompt-engineering-vs-context-engineering)
   - [1.2 The technique toolkit (simplest lever first)](#12-the-technique-toolkit-simplest-lever-first)
2. [Writing the prompt — core techniques](#2-writing-the-prompt--core-techniques)
   - [2.1 Be clear, direct & specific](#21-be-clear-direct--specific)
   - [2.2 Multishot (few-shot) examples](#22-multishot-few-shot-examples)
   - [2.3 XML tags (structure)](#23-xml-tags-structure)
   - [2.4 System prompt & roles](#24-system-prompt--roles)
   - [2.5 Let Claude think (CoT & extended thinking)](#25-let-claude-think-cot--extended-thinking)
   - [2.6 Prefilling the response (and its limits)](#26-prefilling-the-response-and-its-limits)
   - [2.7 Structured outputs](#27-structured-outputs)
3. [Structuring the work](#3-structuring-the-work)
   - [3.1 Prompt chaining](#31-prompt-chaining)
4. [Context engineering & the window](#4-context-engineering--the-window)
   - [4.1 The context window (what counts)](#41-the-context-window-what-counts)
   - [4.2 Managing long context](#42-managing-long-context)
   - [4.3 Prompt anatomy (how to order a prompt)](#43-prompt-anatomy-how-to-order-a-prompt)
5. [Common traps](#5-common-traps)
6. [Quick-fire revision](#6-quick-fire-revision)

---

## 1. Core concepts

### 1.1 Prompt engineering vs context engineering
- **Prompt engineering** = wording a single request well — instructions, examples, structure, role.
- **Context engineering** = deliberately curating **what information is in the window and how it's organised** — a **superset** of prompt wording that also covers documents, history, tool definitions, and retrieval. For agents, it's the bigger lever.
- The exam mixes both: *"rewrite this prompt"* (engineering one message) and *"the agent's context is bloated — what do you do?"* (engineering the whole window, §4).

**Example — same task, two different jobs.** *Task: "answer a customer's question about their contract."*

| | **Prompt engineering** (wording one request) | **Context engineering** (curating the window) |
|---|---|---|
| Focus | How you *ask* | What Claude can *see* |
| Move | Rewrite `"summarize this contract"` → `"You are a contracts analyst. In 5 bullets cover parties, term, payment, termination, liability. <contract>…</contract>"` | Of 200 stored clauses, **retrieve only the 3 relevant ones** (RAG), drop 20 stale chat turns, put the contract **at the top** and the question **at the end**, cache the static system prompt |
| Fixes | A vague or mis-formatted answer | A **bloated / irrelevant** window that's slow, costly, and dilutes the signal |

> Same question — but prompt engineering sharpens the *instruction*, while context engineering decides *what information is even in front of the model*.

> **Before you prompt-engineer at all:** the docs say prompt engineering assumes you already have **clear success criteria, a way to test empirically, and a first-draft prompt** (Domain 4). And *"not every failing eval is best solved by prompt engineering"* — sometimes a **different model** fixes latency/cost faster (Domain 5).

### 1.2 The technique toolkit (simplest lever first)
There's **no official "ranked ladder"** in the current docs (that was the Claude 3-era framing) — they point to one living *Prompting best practices* reference. But the sound engineering habit still holds: **reach for the lightest technique that hits your eval bar before adding scaffolding.** Rough progression:

1. **Be clear & direct** (§2.1) — state task, audience, format, constraints.
2. **Add examples** (§2.2) — show input → output.
3. **Structure with XML tags** (§2.3) — separate instructions from data.
4. **Give a role** via the system prompt (§2.4).
5. **Let Claude think** (§2.5) — thinking / CoT for hard reasoning.
6. **Enforce output shape** (§2.7) — structured outputs.
7. **Chain** complex tasks into steps (§3.1).
8. **Engineer the context window** (§4) — retrieve, prune, order.

**Example — climbing only as far as you need.** *Goal: classify support tickets as `billing` / `technical` / `other`.*
1. **Clear & direct:** *"Classify the ticket into exactly one of: billing, technical, other. Reply with only the label."* → passes? **stop here.**
2. Still mislabels edge cases? **Add 3–5 examples** (§2.2) covering the tricky ones.
3. Still fuzzy on borderline tickets? **Let it think** (§2.5) before answering.
4. Need a guaranteed machine-readable label? **Structured output** (§2.7) — `{"label": "billing"}`.

> You rarely need step 4 if step 1 already clears your eval bar. **Measure at each rung** (Domain 4) and stop climbing once it passes — every extra technique adds tokens/latency.

> Don't jump to complex scaffolding when a **clearer instruction** would do — the #1 cause of bad output is an ambiguous prompt, not a weak model.

---

## 2. Writing the prompt — core techniques

### 2.1 Be clear, direct & specific
> **The mental model (docs):** *"Think of Claude as a brilliant but new employee who lacks context on your norms and workflows. The more precisely you explain what you want, the better the result."*

- **Key test (docs):** *"Show your prompt to a colleague with minimal context and ask them to follow it. If they'd be confused, Claude will be too."*
- **Be specific about output format and constraints.** "Use 2-space indentation" beats "format nicely."
- **Explain the *why*.** Giving the motivation lets Claude **generalise**. Not *"NEVER use ellipses"* but *"Your response will be read aloud by a text-to-speech engine, so never use ellipses — it can't pronounce them."*
- **Sequential steps** as **numbered lists / bullets** when order or completeness matters.
- **Tell Claude what *to* do, not what *not* to do.** Instead of *"Do not use markdown,"* try *"Write in smoothly flowing prose paragraphs."*
- **Use action verbs.** *"Change this function to improve performance"* gets edits; *"Can you suggest changes?"* gets only suggestions.
- **Ask for extra effort explicitly** if you want it — don't rely on inference (*"Go beyond the basics; include as many relevant features as possible"*).

### 2.2 Multishot (few-shot) examples
> Docs: *"Examples are one of the most reliable ways to steer Claude's output format, tone, and structure."*

- **Include 3–5 examples** for best results — usually **more effective than describing** the format in prose.
- Make them **relevant** (mirror your real use case), **diverse** (cover edge cases, so Claude doesn't latch onto an unintended pattern), and **structured** — wrap each in `<example>` tags, the set in `<examples>`, so Claude tells them apart from instructions.
- You can ask Claude to **critique your examples** for relevance/diversity or generate more.

> **Trap:** few-shot is about **demonstrating format/behaviour** — it is **not** a token-saving trick (it *adds* input tokens).

### 2.3 XML tags (structure)
> Docs: *"XML tags help Claude parse complex prompts unambiguously… Wrapping each type of content in its own tag (`<instructions>`, `<context>`, `<input>`) reduces misinterpretation."*

- Cleanly **separate instructions from data** — also an **injection defence** (Domain 7): wrap untrusted content and label it as data.
- **Best practices:** use **consistent, descriptive** tag names; **nest** when there's a hierarchy (`<documents>` → `<document index="n">` → `<document_content>` + `<source>`).
- Make outputs **parseable** by asking for the answer in a named tag (e.g. `<answer>…</answer>`).

> **Trap:** the recommended structuring device is **XML tags** — not ALL-CAPS, not Markdown-only. There are **no reserved/canonical tag names** — just be consistent.

### 2.4 System prompt & roles
- Set via the **top-level `system` parameter** — separate from the `messages` array (not a `system`-role message; Domain 2).

```python
message = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=1024,
    system="You are a helpful coding assistant specializing in Python.",
    messages=[{"role": "user", "content": "How do I sort a list of dicts by key?"}],
)
```

- **Role prompting:** docs say *"Setting a role in the system prompt focuses Claude's behavior and tone for your use case. Even a single sentence makes a difference."* (Current docs frame it as **behaviour + tone** — don't overclaim an accuracy boost.)
- Good home for: persona, high-level task, **output rules**, the **"treat tool/user data as content, not instructions"** policy, and guardrails (Domain 7).

### 2.5 Let Claude think (CoT & extended thinking)
This is the area where guidance **changed most** — know the current picture:

- **Extended thinking is now the primary mechanism.** The **`thinking`** parameter (with an **effort / `budget_tokens`** control) gives dedicated reasoning space; it's **on by default or always-on for several newer models**, and off by default on some (e.g. Opus 4.6–4.8, Sonnet 4.6) when you omit the parameter.
- **Prefer general instructions over hand-written plans:** *"A prompt such as 'think thoroughly' often produces better reasoning than a hand-written step-by-step plan."*
- **Manual chain-of-thought is now a *fallback* for when thinking is off** — ask Claude to reason before answering and put the result in `<answer>` tags to extract it.
- **⚠️ Don't force `<thinking>` tags on newer models:** for the newest models a prompt asking them to write visible reasoning in `<thinking>` tags **"may be declined"** — keep reasoning in the model's own thinking blocks instead. (With Opus 4.5 + thinking disabled, the docs even say avoid the word *"think"* — use *"consider," "evaluate," "reason through."*)
- **Trade-off:** thinking **adds latency and tokens** — docs: use it *"only when it will meaningfully improve answer quality — typically multistep reasoning. When in doubt, respond directly."*

> **Exam takeaway:** CoT/thinking buys accuracy on **complex, multi-step** problems at the cost of **tokens + latency**; the modern default is the **`thinking` parameter**, with manual *"think step by step → `<answer>`"* as the fallback when thinking is off.

### 2.6 Prefilling the response (and its limits)
**What it is:** start the assistant turn with partial content (make the last message `role:"assistant"`) to steer the output — classic uses: prefill `{` to force JSON, prefill *"Sure—"* to skip hedging, prefill a tag to force structure, or hold a character.

**Example — the classic prefill patterns (older models):**
```python
# 1 · Force JSON — prefill the opening brace so Claude continues the object
messages=[
    {"role": "user", "content": "Give me the name and age as JSON."},
    {"role": "assistant", "content": "{"},          # ← prefill; Claude continues: "name": ...
]

# 2 · Skip the preamble / hedging
messages=[
    {"role": "user", "content": "Write a haiku about autumn."},
    {"role": "assistant", "content": "Here is a haiku:"},  # ← or just a prefix to jump straight in
]

# 3 · Hold a character (role consistency)
messages=[
    {"role": "user", "content": "What's in the cave?"},
    {"role": "assistant", "content": "[Dungeon Master] "},  # ← forces the persona tag every turn
]
```

**Example — what happens on a current model (Claude 4.6+):**
```python
# Same prefill, new model → rejected
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=512,
    messages=[
        {"role": "user", "content": "Give me the name and age as JSON."},
        {"role": "assistant", "content": "{"},      # ← prefilled LAST turn
    ],
)
# ✗ 400 invalid_request_error — prefilled last assistant turn not supported on 4.6+
```

**Example — the modern replacement (force JSON without prefill):**
```python
# Use structured outputs instead of prefilling "{"
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=512,
    messages=[{"role": "user", "content": "Give me the name and age as JSON."}],
    output_config={"format": {
        "type": "json_schema",
        "schema": {
            "type": "object",
            "properties": {"name": {"type": "string"}, "age": {"type": "integer"}},
            "required": ["name", "age"],
            "additionalProperties": False,
        },
    }},
)
# ✓ guaranteed-valid JSON, no prefill needed
```
*(An assistant message that isn't the **last** turn — e.g. mid-conversation history — is still fine on 4.6+; only a **trailing** prefill is rejected.)*

> **⚠️ Deprecated on newer models (verified):** *"Starting with Claude 4.6 models and Claude Mythos Preview, prefilled responses … on the last assistant turn are no longer supported. Requests with prefilled assistant messages to these models return a 400 error."* Earlier models still support it; adding assistant messages **elsewhere** in the conversation is unaffected.
- Rationale (docs): model instruction-following advanced enough that **most prefill use cases no longer need it**.
- **Modern replacements:** use **structured outputs** (§2.7) to force JSON, and **clear instructions** to skip preamble / hold format.
- Also note: prefilling is **incompatible with structured outputs** (§2.7).

### 2.7 Structured outputs
The reliable way to get valid, parseable JSON — via **constrained decoding**, not polite requests.

- **JSON outputs:** pass a schema in **`output_config.format`** with `type: "json_schema"` → Claude's **response** is guaranteed-valid JSON matching the schema. *(The old `output_format` param is **deprecated**; it moved to `output_config.format` and no longer needs a beta header.)*
- **Strict tool use:** set **`strict: true`** on a tool → the tool **name + inputs** are schema-validated. Use JSON outputs and strict tools **independently or together**.

**API `messages.create` bodies (verified) — define the schema first, then reference it:**
```python
# ── Step 1 · define the JSON Schema once ──
contact_schema = {
    "type": "object",
    "properties": {
        "name":   {"type": "string"},
        "email":  {"type": "string"},
        "urgent": {"type": "boolean"},
    },
    "required": ["name", "email", "urgent"],
    "additionalProperties": False,          # required for structured outputs / strict
}

# ── Route A · JSON outputs (constrains Claude's final response) ──
resp = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Extract the contact from this email: ..."}],
    output_config={"format": {"type": "json_schema", "schema": contact_schema}},  # ← reference it
)
# read: json.loads(next(b.text for b in resp.content if b.type == "text"))

# ── Route B · Strict tool use (validates the tool's name + inputs) ──
resp = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Log this contact."}],
    tools=[{
        "name": "record_contact",
        "strict": True,                     # ← schema-validate this tool's input
        "input_schema": contact_schema,     # ← same schema, reused
    }],
    # tool_choice defaults to {"type":"auto"}
)
# read: the tool_use block's .input is guaranteed to match contact_schema
```
> Defining `contact_schema` once and referencing it keeps the call bodies clean and lets **both routes share one schema**.

> **Can I use both in one call?** Yes — and this is the common real-world setup: Claude needs to *gather* something with a tool **and** hand back a clean structured answer. Put **both** `output_config={"format": …}` and `tools=[… strict:true …]` in the same request. They don't conflict because each governs a **different part of the turn**:
> - `output_config.format` → constrains **only Claude's final written text** into your JSON schema.
> - the strict tool → constrains **the tool-call input** (name + arguments) into the tool's own schema.
>
> So Claude can call your tool (valid inputs), read the result, then write a final answer that is schema-valid JSON — in one call.
>
> **Worked picture:** *"Look up this employee's manager, then return `{name, manager, dept}` as JSON."* → Claude calls `lookup_employee` (input shaped by the tool's strict schema) → reads the result → writes the final answer, forced to match `output_config.format`.
>
> ⚠️ **The one exception is `web_search`.** It always attaches citations to the output, and citations are incompatible with `output_config.format` → **400**. Custom tools (DB query, employee fetch) combine fine; `web_search` does not. (See §"Combining" above for the two workarounds.)
**Example — a strict tool definition (schema-validated name + inputs):**
```python
tools = [{
    "name": "extract_contact",
    "description": "Extract the contact's details from the email.",
    "strict": True,                        # schema-validate the tool input
    "input_schema": {
        "type": "object",
        "properties": {
            "name":   {"type": "string"},
            "email":  {"type": "string"},
            "urgent": {"type": "boolean"},
        },
        "required": ["name", "email", "urgent"],
        "additionalProperties": False,     # required for strict tool use
    },
}]
```
*(A tool has **three** fields — `name`, `description`, `input_schema` — there is **no `output`/`returns` field**; Domain 8.)*

**What the model sends back when it wants a tool — and how to detect it:**
When Claude decides to call the tool, it does **not** answer in text. Two signals tell you so, both on the response object:
- **`response.stop_reason == "tool_use"`** — the turn ended *because* Claude wants a tool, not because it finished talking.
- a **`tool_use` block** inside `response.content` — carrying the tool `name`, a unique `id`, and the `input` (your arguments, schema-valid if the tool is `strict`).

```python
resp = client.messages.create(..., tools=tools)   # the extract_contact tool above

# 1) Did Claude ask for a tool at all?
if resp.stop_reason == "tool_use":
    # 2) Pull the tool_use block out of the content array
    #    (content can also hold text / thinking blocks — filter by type)
    tool_call = next(b for b in resp.content if b.type == "tool_use")

    print(tool_call.name)     # "extract_contact"
    print(tool_call.id)       # "toolu_01A…"  ← echo this back in the tool_result
    print(tool_call.input)    # {"name": "...", "email": "...", "urgent": true}  ← your data

    # For a structured-output tool, tool_call.input IS the answer — just read it.
    # For a real action tool, you'd run it and return a tool_result with this id.
```

Shape of the raw JSON response (abridged):
```json
{
  "stop_reason": "tool_use",
  "content": [
    { "type": "text", "text": "Let me pull those details out." },
    { "type": "tool_use",
      "id": "toolu_01A09q90qw",
      "name": "extract_contact",
      "input": { "name": "Dana Lee", "email": "dana@acme.io", "urgent": true } }
  ]
}
```
> **Read the fields, don't eyeball the text:** branch on `stop_reason == "tool_use"`, then find the block whose `type == "tool_use"`. `content` is a **list of blocks** and may also contain `text` or `thinking` — never assume `content[0]` is the tool call.

**`tool_choice` — make Claude actually call the tool (verified):**

| Mode | Meaning | Default? |
|---|---|---|
| `{"type":"auto"}` | Claude decides whether to call a tool or answer directly | ✅ default **when `tools` are provided** |
| `{"type":"any"}` | **Must** use one of the tools (Claude picks which) | — |
| `{"type":"tool","name":"…"}` | **Force a specific** named tool — the direct way to coerce structured output via a tool | — |
| `{"type":"none"}` | Cannot use any tool | ✅ default **when no `tools`** |

- To **force** structured output through a tool, set `tool_choice` to `any` or a specific `tool`.
- Add **`disable_parallel_tool_use: true`** (e.g. `{"type":"auto","disable_parallel_tool_use":true}`) for **at most one** tool call per turn.
- **⚠️ Model caveats (verified):** on **Opus 5.5 / Sonnet 5.5 / Fable 5.1 / Mythos 5.1**, `any` and `tool` return a **400** — the docs recommend **`auto` + strict tool use, or structured outputs** instead. `any`/`tool` are also unsupported with **manual extended thinking**, and forcing a tool **prefills the assistant turn** (no text before the `tool_use`, and thinking is skipped). Changing `tool_choice` also **invalidates cached blocks** (Domain 5).
- *(The full tool round trip — `stop_reason:"tool_use"` → you execute → `tool_result` — plus parallel calls and `is_error`, are in **Note 08**.)*

**Combining structured output with *other* tools (verified):**
You can gather info with tools **and** return a structured answer — but the mechanism matters:
- **Grammar scope:** `output_config.format` constrains **only Claude's direct text output**, *not* tool calls/results/thinking — docs: *"allowing Claude to think freely while still producing structured output in the final response."* So Claude runs the normal loop (calls your tools across turns) and **only the final answer is schema-shaped**. JSON outputs **work alongside tools**, incl. `strict` ones; non-strict tools can coexist.
- **✅ Custom data tools (e.g. `employee_info_fetch`):** combine directly — keep `tool_choice: auto`, define the tool, set `output_config.format`. Claude fetches, then emits valid JSON.
- **⚠️ `web_search` is the exception:** it **always emits citations** (not configurable), and structured outputs is **incompatible with citations** → **`output_config.format` + web_search = 400**. Two fixes: **(a)** drop `output_config.format` and use a **`strict` `submit_answer` tool** as the output channel (read the data from its validated `input`) — citations only clash with `output_config.format`, not strict tools; or **(b)** split into two calls: search first (citations fine), then a second call with `output_config.format` over the gathered text.
- **Don't *force* the output tool** (`tool_choice:{type:"tool"/"any"}`) if you still need to gather — forcing calls it **immediately**, emits nothing else, and **skips thinking**. Use `auto` so the loop gathers first. (And `any`/`tool` now **400** on Opus 5.5 / Sonnet 5.5 / Fable 5.1 / Mythos 5.1.)

**Two routes to structured output (pick by whether you need `web_search`):**

| | **Via `output_config.format`** (JSON outputs) | **Via a tool** (`tool_choice`, read `tool_use.input`) |
|---|---|---|
| What it is | Constrained decoding on Claude's **final text** | A tool whose `input_schema` = your output; read from its `input` |
| Shape guaranteed valid? | **Yes** (grammar-constrained) | Only with **`strict: true`** |
| Guaranteed to be *produced*? | **Yes** — it's the response format | **No** with `auto` — must force / loop (below) |
| Works with `web_search`? | **No → 400** (web_search forces citations; `.format` ✗ citations) | **Yes** — ordinary tool use doesn't clash with citations |
| Multi-turn tool gathering | Yes — only the **final answer** is shaped | Yes — it's **one tool among many** |
| Best when | A single structured answer, **no** citation tools | You need **`web_search`/other tools + a structured finish** |

The table's **"Guaranteed to be *produced*?"** row flagged the weak spot of the **via-a-tool** route: unlike `output_config.format` (where the schema *is* the response), a tool only shapes the answer *if Claude decides to call it*. That's the gap this next part closes.

**Guaranteeing the output *tool* actually fires (the `auto` gap):** `tool_choice: auto` does **not** guarantee Claude calls your output tool — it may just answer in text. Earn the guarantee with:
- **A control loop** (works on any model): don't treat the turn as done until `record_answer` is called; else send *"now call `record_answer`."*
- **Force on a *final* turn:** `tool_choice:{"type":"tool","name":"record_answer"}` **guarantees** the call — but you can't gather in that same call (no text, thinking skipped), and it **400s on Opus 5.5 / Sonnet 5.5 / Fable 5.1 / Mythos 5.1**.
- **Two-phase (robust default):** Phase 1 `auto` + `web_search`/custom tools to gather → Phase 2 a fresh call that **forces** the output tool (or uses `.format`).
- **System-prompt steer** (*"finish by calling `record_answer` only"*) — raises the hit rate, **not** a hard guarantee.
- **⚠️ Two different guarantees:** `strict: true` guarantees the **shape if the tool is called** — never that it **is** called.

**Which method to reach for (most reliable first):** if you want JSON out of Claude, there are three ways to ask — and they are *not* equally dependable. Prefer them in this order:

1. **Schema-enforced** — `output_config.format` **or** a tool with `strict: true`. The API *mechanically* enforces the shape, so the output can't come back malformed. **Use this.**
2. ~~**Prefill the opening `{`**~~ — the old trick of seeding the assistant turn with `{` to nudge JSON. **No longer works** — prefilling the last assistant turn **400s on Claude 4.6+** (see §2.6). Listed only so you recognize it as outdated.
3. **Ask in the prompt** — describe the schema in words and say "return *only* that JSON." Works, but nothing enforces it, so Claude can still add prose or drift. **Fallback only.**

Whichever you use, **always validate the output and handle errors** — enforcement covers shape, not every edge case (next bullet).

**Edge cases that bite even with enforcement (verified):**
- **Refusal:** if Claude refuses (`stop_reason: "refusal"`), the response **won't match your schema** — it's a refusal, not data. Check `stop_reason` before parsing.
- **Truncation:** hitting `max_tokens` mid-answer cuts the JSON off incomplete → **retry with a higher `max_tokens`**.
- **Enum casing:** an enum value's **capitalization isn't guaranteed** (you might get `"High"` not `"HIGH"`) → **compare case-insensitively**.
- **Can't combine with prefilling or citations:** pairing `output_config.format` with either → **400**.

> **Trap (exam takeaway):** the reliable way to get structure is **schema enforcement** — `output_config.format` **or** a strict tool schema. *Not* a politely-worded prose request, and **not** prefill (dead on current models).

> **Cross-ref:** **Claude Code** does structured output as a literal **tool call** — `--json-schema` injects a synthetic **`StructuredOutput`** tool and the agent *calls it* (read the data from the result's `.structured_output`). Mechanics in **Note 03 §6 (CI/CD `-p` mode)**.

---

## 3. Structuring the work

### 3.1 Prompt chaining
- **Break a complex task into linked prompts**, each handling one step (one step's output → the next step's input).
- **Current-model nuance (docs):** *"With adaptive thinking and subagent orchestration, Claude handles most multistep reasoning internally. Explicit prompt chaining … is still useful when you need to inspect intermediate outputs or enforce a specific pipeline structure."*
- **Why chain:** each step is a **separate API call** → you can **log, evaluate, or branch** at any point (debuggability + validation gates). Overlaps Domain 1's prompt-chaining pattern.
- **Canonical pattern — self-correction:** *generate a draft → review it against criteria → refine based on the review* (an evaluator–optimiser flavour).

---

## 4. Context engineering & the window

### 4.1 The context window (what counts)
- **Sizes (verified):** a **200K-token** window is the baseline (older models, e.g. Sonnet 4.5 deprecated); **current flagship models default to a 1M-token window** (Opus 4.6–5.5, Sonnet 4.6/5/5.5, Haiku 5.5, Fable/Mythos 5/5.1) — **no beta header needed**, billed at standard pricing (**exception:** on **Haiku 5.5**, prompts over **100K tokens cost more**) — and a single request can generate up to **128K output tokens**.
- **Everything counts** toward the window: the **system prompt**, **every message** (incl. tool results, images, documents), your **tool definitions**, and the **output Claude generates** (incl. extended-thinking tokens).
- **Prompt caching doesn't change what counts** — only what you pay; cached tokens still occupy the window (Domain 5).
- **Overflow:** input alone over the limit → **400 `invalid_request_error`** ("prompt is too long"); on 4.5+, input + `max_tokens` over the limit can stop with **`stop_reason: "model_context_window_exceeded"`** (Domain 2).

### 4.2 Managing long context
**Curate, don't dump** — a bloated window is slower, pricier, and *less* accurate (relevant signal gets diluted).

- **Retrieve only what's relevant (RAG)** instead of stuffing everything in.
- **Summarise / prune / compact** older, less-relevant turns while keeping salient facts & goals. Two API features do this for you (both verified):
  - **Compaction** — *server-side summarising*; the docs call it the **primary strategy for long-running conversations**. (A **client-side** variant also exists in the TS/Ruby SDKs via `tool_runner`.)
  - **Context editing** (**beta** — header `context-management-2025-06-27`) — automatically **clears stale tool results** (`clear_tool_uses_20250919`) and **old thinking blocks** (`clear_thinking_20251015`) once the window crosses a threshold, replacing them with placeholders while your client keeps the full history.
- **Long-document layout (verified):** for data-rich inputs (**20K+ tokens**), put the **documents near the TOP**, above your query/instructions/examples; keep the **query at the END** — *"Queries at the end can improve response quality by up to 30% in tests,"* especially with complex multidocument inputs.
- **Structure docs with XML tags:** wrap each in `<document>` with `<document_content>` and `<source>` subtags.
- **Ground responses in quotes:** ask Claude to **quote the relevant passages first**, then answer — *"helps Claude focus on the relevant content and ignore the rest."*
- **Memory (for agents):** persist durable facts/state **outside** the window and reload only what's needed — don't rely on an ever-growing transcript.

### 4.3 Prompt anatomy (how to order a prompt)

**What "prompt anatomy" means.** A prompt isn't one blob of text — it's assembled from several distinct parts: the system prompt, tool definitions, any documents you paste in, examples, your instructions, and finally the actual question. *"Anatomy"* just means **which parts a prompt is made of, and in what order you arrange them** — the way the anatomy of a body is how its parts are laid out.

**Why the order matters (so, why study it).** The order isn't cosmetic — it changes two things at once, and they happen to point the same way:

1. **Answer quality.** Claude attends to different positions differently. Putting long documents near the **top** and the actual question at the very **end** measurably improves answers — the docs say **up to 30%** on complex, multi-document inputs (§4.2). Bury the question in the middle, or put documents *after* it, and quality drops.
2. **Cost and speed (prompt caching).** Caching works on a **prefix** — everything from the start up to the first byte that changes can be cached and reused cheaply. So the *static* stuff (system, tools, big documents) belongs **first**, and the *changing* stuff (the user's specific question) belongs **last**. Put something dynamic early and you **break the cache** for everything after it (Domain 5).

**The useful coincidence.** *"Big/static first, question last"* wins on **both** axes at once. That's the whole payoff — one layout gives you better answers **and** cheaper, faster calls, so you don't have to trade one for the other.

**Why it's on the exam.** This is where Domain 6 (prompt/context engineering) meets Domain 5 (caching/cost). A likely question is *"where does the user's question go in a long-context prompt?"* or *"what ordering maximizes both cache hits and quality?"* — and this layout is the answer.

A well-ordered long-context prompt therefore looks like:

```
┌───────────────────────────────────────────────┐
│ SYSTEM  (role + rules + untrusted-data policy)  │ ── static ─┐
├───────────────────────────────────────────────┤            │  cache
│ TOOL DEFINITIONS                                │ ── static ─┤  the
├───────────────────────────────────────────────┤            │  prefix
│ LONG DOCUMENTS   (20K+ tokens, at the TOP)      │ ── static ─┘
│   <document><source/><document_content/>…       │
├───────────────────────────────────────────────┤
│ EXAMPLES   (<examples><example>…)               │
├───────────────────────────────────────────────┤
│ INSTRUCTIONS   (numbered, what TO do)           │
├───────────────────────────────────────────────┤
│ THE QUERY / QUESTION   (at the END)             │ ◄── dynamic
└───────────────────────────────────────────────┘
   then: quote-relevant-passages-first → answer
```

> The docs put **long docs first, the query last**, and prompt-caching wants **static content first, dynamic last** — the two line up, so this ordering helps **both quality and cost**.

---

## 5. Common traps
- **Ambiguity** is the #1 failure — be specific; apply the **"confused colleague"** test.
- **XML tags** are the recommended structuring device (not ALL-CAPS, not Markdown-only); no reserved tag names.
- **Few-shot = demonstrate format**, not save tokens (it *adds* tokens).
- **CoT/thinking costs** tokens + latency — not for trivial tasks; on newer models **use the `thinking` parameter** and **don't force `<thinking>` tags** (they may be declined).
- **Prefill is unavailable on Claude 4.6+ last assistant turn** (400) — use **structured outputs** / clear instructions instead.
- Most reliable **structured output** = **`output_config.format` / tool schema** (constrained decoding), **not** prose and **not** prefill; it's **incompatible with prefilling + citations**.
- **System prompt is a parameter**, not a `system`-role message; a role sets **behaviour + tone**.
- **Bigger context ≠ better** — irrelevant context *hurts* accuracy and cost; retrieve/prune. Everything (system, tools, history, thinking) counts toward the window.
- **Long documents first, question last** (up to 30% quality lift); key instructions near the query.
- "Context engineering" ≠ just prompt wording — it's **managing the whole window**.

---

## 6. Quick-fire revision
- Q: Best way to delimit instructions vs data? **A:** **XML tags** (consistent, descriptive, nestable).
- Q: How many few-shot examples, and why? **A:** **3–5**, to **demonstrate** format/tone/structure (not to save tokens).
- Q: Mental model for a clear prompt? **A:** A **brilliant but new employee** — spell out norms, format, and the **why**.
- Q: One cheap trick to improve a vague instruction? **A:** **Explain the motivation** (Claude generalises) + the **"confused colleague" test**.
- Q: Modern mechanism for step-by-step reasoning? **A:** The **`thinking` parameter** (extended thinking); manual *"think → `<answer>`"* is the **fallback when thinking is off**.
- Q: Should you ask a newest model to reason in `<thinking>` tags? **A:** **No** — it may be **declined**; keep reasoning in its thinking blocks.
- Q: Force a JSON reply reliably — how (current)? **A:** **Structured outputs** (`output_config.format`, `type: json_schema`) — **not** prefill.
- Q: Is prefill still supported? **A:** **Not on Claude 4.6+** (last assistant turn → **400**); earlier models only. Use structured outputs / instructions.
- Q: JSON outputs vs strict tool use? **A:** JSON outputs constrain **Claude's response**; `strict: true` validates **tool names + inputs**. Combine if needed.
- Q: Can you use tools (data fetch) *and* return structured output? **A:** **Yes** — `output_config.format` shapes only the **final direct output**; Claude still calls tools in the loop (`tool_choice: auto`). Custom tools combine directly.
- Q: Why does `web_search` + structured outputs fail, and the fix? **A:** `web_search` **always emits citations**, and `output_config.format` is **incompatible with citations** → **400**. Fix: use a **`strict` `submit_answer` tool** as the output, or **split into two calls** (search, then structure).
- Q: Two routes to structured output, and which works with `web_search`? **A:** **`output_config.format`** (constrains final text; **✗ web_search** → citation 400) vs **via a tool** (`tool_choice` + read `tool_use.input`; **✓ web_search**).
- Q: With `tool_choice: auto` + many tools, is the output tool guaranteed to fire? **A:** **No.** Guarantee it via a **control loop**, **forcing on a final turn** (can't on 5.5-era → 400), or a **two-phase** gather-then-force. `strict:true` only guarantees the **shape if called**.
- Q: Where does the system prompt live, and what does a role do? **A:** The top-level **`system`** parameter; a role **focuses behaviour + tone**.
- Q: Where to place a long document vs the question? **A:** **Document near the top (20K+ tokens), question at the end** (up to **30%** quality lift).
- Q: One grounding trick for long docs? **A:** Ask Claude to **quote relevant passages first**, then answer.
- Q: What counts toward the context window? **A:** **Everything** — system, all messages (tool results/images/docs), tool defs, and generated output incl. thinking.
- Q: Current flagship context size? **A:** **1M tokens by default** (no beta header) on current models; **200K** baseline on older ones; up to **128K** output.
- Q: Tactic when nearing the limit? **A:** **Retrieve only relevant (RAG)**, **summarise/prune/compact** stale turns, persist **memory** outside the window.
- Q: Does more context always help? **A:** **No** — irrelevant context dilutes accuracy and raises cost/latency.
- Q: "Prompt engineering" vs "context engineering"? **A:** Wording one message vs **curating the whole window** (a superset).

**One-line summary:** *Be clear and specific (the "new employee" test), show **3–5 examples**, delimit with **XML tags**, set a **role** in the `system` param, use the **`thinking` parameter** for hard reasoning (don't force `<thinking>` tags, and **prefill is gone on 4.6+**), force structure via **`output_config.format`/tool schemas**, chain only to inspect/branch — and **engineer the context window** (retrieve & prune, docs first / question last, everything counts).*