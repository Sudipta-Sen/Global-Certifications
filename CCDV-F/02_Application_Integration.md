# CCDV-F Revision Note — Domain 2: Applications & Integration

> **Exam weight: ~33.1%** — by far the biggest domain (roughly **1 question in 3**). This is the Messages API: how you talk to Claude over HTTP. Know the headers, the required params, the response fields, the seven `stop_reason`s, statelessness, streaming, and the multimodal/file surface **cold** — and **where each piece runs** (browser vs your backend vs Anthropic, §9), a favourite scenario angle. This is where the exam is won or lost.

---

## Contents
1. [The Messages API foundation](#1-the-messages-api-foundation)
   - [1.1 Endpoint, headers & versioning](#11-endpoint-headers--versioning)
   - [1.2 Required & optional parameters](#12-required--optional-parameters)
   - [1.3 The request & response shape](#13-the-request--response-shape)
   - [1.4 The context window & overflow](#14-the-context-window--overflow)
2. [Conversation structure](#2-conversation-structure)
   - [2.1 Roles & the system prompt](#21-roles--the-system-prompt)
   - [2.2 Content blocks](#22-content-blocks)
   - [2.3 Statelessness & multi-turn](#23-statelessness--multi-turn)
   - [2.4 Prefill (and why it's going away)](#24-prefill-and-why-its-going-away)
3. [Controlling & reading the turn](#3-controlling--reading-the-turn)
   - [3.1 `stop_reason` — the seven values](#31-stop_reason--the-seven-values)
   - [3.2 Streaming (SSE)](#32-streaming-sse)
   - [3.3 Token counting](#33-token-counting)
4. [Tool use — the turn flow](#4-tool-use--the-turn-flow)
   - [4.1 The tool-use loop](#41-the-tool-use-loop)
   - [4.2 Single vs parallel tool calls](#42-single-vs-parallel-tool-calls)
   - [4.3 The `tools` array and `tool_choice` shapes](#43-the-tools-array-and-tool_choice-shapes)
5. [Multimodal & file input](#5-multimodal--file-input)
   - [5.1 Vision (images)](#51-vision-images)
   - [5.2 PDFs & documents](#52-pdfs--documents)
   - [5.3 The Files API](#53-the-files-api)
6. [Advanced API surface](#6-advanced-api-surface)
   - [6.1 Prompt caching](#61-prompt-caching)
   - [6.2 Message Batches](#62-message-batches)
   - [6.3 Thinking (adaptive vs manual)](#63-thinking-adaptive-vs-manual)
   - [6.4 Structured outputs & citations](#64-structured-outputs--citations)
   - [6.5 Embeddings (Anthropic doesn't offer them)](#65-embeddings-anthropic-doesnt-offer-them)
   - [6.6 The MCP connector (`mcp_servers`)](#66-the-mcp-connector-mcp_servers)
   - [6.7 `context_management` (compaction and context editing)](#67-context_management-compaction-and-context-editing)
7. [Errors & retries](#7-errors--retries)
8. [SDKs & deployment platforms](#8-sdks--deployment-platforms)
9. [Where it runs: browser vs your backend vs Anthropic](#9-where-it-runs-browser-vs-your-backend-vs-anthropic)
   - [9.1 The standard architecture & the API-key rule](#91-the-standard-architecture-and-the-api-key-rule)
   - [9.2 Client tools vs server tools: where execution happens](#92-client-tools-vs-server-tools-where-execution-happens)
   - [9.3 Where each piece of state & work lives](#93-where-each-piece-of-state-and-work-lives)
   - [9.4 Productionizing the integration — wrap the SDK in an integration service](#94-productionizing-the-integration--wrap-the-sdk-in-an-integration-service)
10. [Common traps](#10-common-traps)
11. [Quick-fire revision](#11-quick-fire-revision)

---

## 1. The Messages API foundation

### 1.1 Endpoint, headers & versioning
Everything you do with Claude over HTTP goes through **one endpoint**: `POST /v1/messages`. You send a list of messages, Claude replies with the next message. Three headers are mandatory:

| Header | Value | What it does |
|---|---|---|
| **`x-api-key`** | your API key | Authentication. **Not** `Authorization: Bearer …` — this is the single most-tested header gotcha. |
| **`anthropic-version`** | a dated string, e.g. `2023-06-01` | Pins the API's *behaviour* to a version. **This is not the model version.** |
| **`content-type`** | `application/json` | Standard JSON body. |

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-5-5","max_tokens":1024,
       "messages":[{"role":"user","content":"Hello"}]}'
```

**About `anthropic-version` (verified).** The header is required on every raw HTTP call (the SDKs set it for you). Valid values are `2023-01-01` (the initial release), **`2023-06-01`** (the long-standing stable value you'll see everywhere), and **`2026-10-21`** (the newest). For any given version Anthropic promises not to break existing input/output parameters — so pinning a version keeps your integration stable even as the API evolves.

> **Version callout (verified — and a likely exam curveball).** The newest version, **`2026-10-21`, makes breaking changes** you should recognise: it **removes `temperature`, `top_p`, and `top_k`** entirely (sending any of them returns a **400** on *every* model), rejects a `document` block placed in an `assistant` message (400), and rejects a `text` block carrying a non-empty `citations` array in a `user` message (400). On the classic `2023-06-01` those sampling params still work. So "is `temperature` a valid param?" now has a *version-dependent* answer — the Foundations exam most likely assumes `2023-06-01`, but know that the latest version drops it.

### 1.2 Required & optional parameters
**Three body params are required — memorise them:**

| Required | Meaning |
|---|---|
| **`model`** | e.g. `claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`. |
| **`messages`** | the conversation array (see §2). |
| **`max_tokens`** | the **cap on output tokens**. Yes — **`max_tokens` is mandatory**, not optional. |

**Common optional params:**

| Optional | Use |
|---|---|
| `system` | top-level system prompt (string or array of text blocks) — see §2.1. |
| `stop_sequences` | strings that halt generation (→ `stop_reason: "stop_sequence"`). |
| `stream` | `true` for SSE streaming (§3.2). |
| `tools` / `tool_choice` | tool definitions and how to pick (`auto` / `any` / `tool` / `none`, + `disable_parallel_tool_use`) — §4. |
| `thinking` | reasoning mode: `adaptive`, `disabled`, `between_tools`, or deprecated `enabled` — §6.3. |
| `output_config` | `{effort, format}` — reasoning depth (`low`→`max`) and structured-output JSON schema (§6.4). |
| `metadata` | `{user_id}` — an opaque id ≤ **512 chars**. **Never** put a name or email here. |
| `service_tier` | throughput tier: **`auto`** (default — use Priority Tier capacity when available, else standard) or **`standard_only`**. Priority Tier = dedicated capacity. |
| `mcp_servers` | remote MCP servers to connect, paired with an `mcp_toolset` in `tools` (beta header) — §6.6. |
| `context_management` | automatic pruning of old context / tool results (beta header) — §6.7. |
| `container` | reuse a code-execution container by id, and attach skills — Domain 8. |
| `diagnostics` | `{previous_message_id}` — helps diagnose cache misses; doesn't make the API stateful (§2.3). |
| `betas` | opt into beta features (body-param equivalent of the `anthropic-beta` header). |
| `temperature` / `top_p` / `top_k` | sampling controls (`temperature` 0.0–1.0, default 1.0) — **present on `2023-06-01`, removed on `2026-10-21`** (§1.1). |

### 1.3 The request & response shape
A request is just `model` + `max_tokens` + `messages` (+ optional `system`):
```json
{
  "model": "claude-sonnet-5-5",
  "max_tokens": 1024,
  "system": "You are a helpful assistant.",
  "messages": [ { "role": "user", "content": "Hello" } ]
}
```

The response is a **Message object** — learn these fields, they come up constantly:
```json
{
  "id": "msg_01…",
  "type": "message",
  "role": "assistant",
  "content": [ { "type": "text", "text": "Hi! How can I help?" } ],
  "model": "claude-sonnet-5-5",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": { "input_tokens": 10, "output_tokens": 25 }
}
```
- **`content` is *always* an array of blocks** — even a plain text reply is `[{"type":"text", …}]`. Never assume it's a bare string.
- **`stop_reason`** tells you *why* generation stopped (§3.1); **`stop_sequence`** holds the matched string when that's the reason, else `null`.
- **`usage`** reports `input_tokens` and `output_tokens`, plus `cache_creation_input_tokens` / `cache_read_input_tokens` when caching is on (§6.1). It also carries a `cache_creation` breakdown by TTL and a `server_tool_use` count (e.g. `web_search_requests`) when those apply.
- Every response also carries a **`request-id`** header (e.g. `req_018Ee…`) — quote it to support when reporting a problem.

### 1.4 The context window & overflow
Everything in a request shares **one budget** — the **context window** — and the output draws on it too. Know the numbers and what happens when you blow past them.

- **Window size:** **1M tokens** on the newest models (it's the **default — no beta header**), **200K** on older ones. The **output cap is 128K tokens** (`max_tokens` can't exceed it).
- **What counts:** *everything* — the `system` prompt, the whole `messages` array (including tool results, images, documents), the `tools` definitions, **and** the generated output (including thinking tokens). History is resent every turn (§2.3), so the window **grows each turn**.
- **Caching doesn't shrink it:** cached tokens still *occupy* the window — `input_tokens` + `cache_read_input_tokens` + `cache_creation_input_tokens` all count. Caching changes what you *pay*, not whether tokens count (§6.1).

**Three different "too big" outcomes — don't confuse them:**

| Situation | Result |
|---|---|
| Output hits **your** `max_tokens` cap | **`stop_reason: "max_tokens"`** — output truncated; raise the cap or continue (§3.1). |
| **Input alone** exceeds the window | HTTP **400** `invalid_request_error` — *"prompt is too long"* (every model). |
| Input fits, but **input + generated output** overruns the window mid-generation | **`stop_reason: "model_context_window_exceeded"`** — output truncated (Claude 4.5+; older models need the `model-context-window-exceeded-2025-08-26` beta header, else they error). |

*(Trimming or summarising history to stay under the window — compaction, context editing, tool-result clearing — is context engineering, Domain 6.)*

---

## 2. Conversation structure

### 2.1 Roles & the system prompt
Think of a conversation as a transcript with **two speakers**: **`user`** and **`assistant`**. That's it — there is **no `system` role inside `messages`**. The system prompt rides as a **separate top-level `system` parameter**.

```json
{
  "system": "You are a terse assistant.",
  "messages": [
    { "role": "user", "content": "Hi" },
    { "role": "assistant", "content": "Hello." },
    { "role": "user", "content": "Explain TCP." }
  ]
}
```
- Messages should **alternate** user/assistant, and the **first** message is `user`. Two same-role messages in a row are **merged**.
- Putting instructions in a fake `{"role":"system", …}` message is the classic wrong answer — system is a *parameter*, not a message.

### 2.2 Content blocks
A message's **`content`** is either a plain **string** or an **array of typed blocks**. The block types you must recognise:

| Block | Who emits it | What it is |
|---|---|---|
| **`text`** | both | plain text. |
| **`image`** | user | an image for vision (§5.1). |
| **`document`** | user | a PDF/text file for analysis (§5.2). |
| **`tool_use`** | **assistant** | a request to call a tool (§4). |
| **`tool_result`** | **user** | the result you return for a `tool_use`. |
| **`thinking`** | assistant | extended/adaptive reasoning (§6.3). |

(You'll also meet `server_tool_use`, `mcp_tool_use`, `search_result`, and `container_upload` in the tool/file domains.) The mental rule: **the assistant *requests* with `tool_use`; you *reply* with `tool_result` in a `user` message.**

### 2.3 Statelessness & multi-turn
**The single most important architectural fact in this domain: the Messages API is stateless.** There is **no server-side session and no `conversation_id`**. The server remembers *nothing* between calls.

So to continue a conversation, **you resend the entire prior history every time** — all previous user and assistant turns — and append the new user message. *You* (the developer) own storing that history.

```
Turn 2 request = [user#1, assistant#1, user#2]      ← resend #1 and #1's reply
Turn 3 request = [user#1, assistant#1, user#2, assistant#2, user#3]
```
- When a turn used a tool, you must **preserve both** the assistant **`tool_use`** turn *and* the user **`tool_result`** turn, in order — drop either and the next call breaks.
- Longer history = **more input tokens every single turn** → this is exactly why **prompt caching** (§6.1) and history pruning matter for cost (Domains 5/6).
- *New:* a `diagnostics: {previous_message_id}` field lets you ask the API to help diagnose **prompt-cache misses** across turns — but it does **not** make the API stateful; you still resend history.

> **Exam trap:** "How do you maintain conversation context?" → **resend the full message history**; there is **no `conversation_id`** and no hidden session.

### 2.4 Prefill (and why it's going away)
**Prefilling** = you write the **start** of the assistant's reply yourself, as a final `assistant` message, and Claude continues from there. Historically it was used to force a format (open with `{` to force JSON) or skip a preamble.

```json
"messages": [
  { "role": "user", "content": "List 3 colors as a JSON array." },
  { "role": "assistant", "content": "[" }        // ← prefill: Claude continues the array
]
```

> **Verified — prefill is being retired.** *"Starting with Claude 4.6 models and Claude Mythos Preview, prefilled responses … on the last assistant turn are no longer supported. Requests with prefilled assistant messages to these models return a 400 error."* The reasoning: newer models follow instructions well enough that you rarely need it. **Earlier models still support prefill**, and adding assistant messages **elsewhere** in the conversation (not the final turn) is unaffected. Modern replacement: just *ask* for the format, or use **structured outputs** (§6.4).

---

## 3. Controlling & reading the turn

### 3.1 `stop_reason` — the seven values
Every successful response carries a `stop_reason` telling you **why Claude stopped**. Your loop branches on it, so know all seven and what to *do*:

| Value | Meaning (verified) | What you do |
|---|---|---|
| **`end_turn`** | Claude finished naturally (most common). | Use the response. |
| **`max_tokens`** | Hit **your** `max_tokens` cap — output is **truncated**. | Raise `max_tokens`, or continue; if it cut off mid-`tool_use`, retry higher. |
| **`stop_sequence`** | Claude emitted one of **your** `stop_sequences`. | Read `stop_sequence` to see which fired. |
| **`tool_use`** | Claude is calling a tool. | Run it, return a `tool_result` (§4). |
| **`pause_turn`** | A **server-tool loop** hit its iteration limit (default **10**). | Append `response.content` back as an assistant turn and resend (pass the `container` id too, if any). |
| **`refusal`** | Claude **declined** (safety/policy). | Comes back as **HTTP 200**, not an error; read `stop_details`, retry on a fallback model. |
| **`model_context_window_exceeded`** | Output **filled the model's context window**. | Treat the output as **truncated**. |

**Worth knowing (verified nuances):**
- **The "empty `end_turn`" gotcha (and how to avoid it).** When you hand tool results back, you send them in a **`user` message as `tool_result` blocks**. If you *also* tack a **text block on *after*** those `tool_result` blocks (e.g. a chatty `{"type":"text","text":"Here's the result"}`), you accidentally teach Claude a pattern — *"after every tool call the user says something next"* — so Claude just **ends its turn and waits for you**, returning an **empty reply with `stop_reason: "end_turn"`** instead of the answer. **Fix:** put the `tool_result` blocks **alone** in that user message (no trailing text). Resending the empty response unchanged won't help — Claude already thinks it's done; as a **last resort**, append a **new** user message like `"Please continue"` and resend.
- **`stop_details` — only filled in for refusals.** The response has an extra top-level field **`stop_details`** (next to `stop_reason`). It is **`null` almost always**; it's populated **only when `stop_reason` is `refusal`**, where it identifies **which policy category** triggered the refusal. So only ever read `stop_details` on a refusal — for every other stop reason there's nothing in it.
- `model_context_window_exceeded` is typed in the SDK's **beta namespace** (`client.beta.messages`); on models **older than Sonnet 4.5** you must add the `model-context-window-exceeded-2025-08-26` beta header to get it (older models raise an error instead).

> **Exam trap:** `max_tokens` (**your** output cap) ≠ `model_context_window_exceeded` (the model's **whole** window). `pause_turn` is **not** the end — you must continue it. `refusal` is a **stop_reason on a 200**, not an HTTP error.

### 3.2 Streaming (SSE)
Set **`stream: true`** and the response arrives incrementally as **Server-Sent Events (SSE)** — **not** WebSockets. Think of it as the model narrating as it writes, so the user sees tokens immediately.

```
message_start                    ← the Message shell (empty content, stop_reason: null)
  content_block_start            ← a block begins (has an `index`)
    content_block_delta          ← incremental pieces (text_delta / input_json_delta / thinking_delta)
    content_block_delta
  content_block_stop
message_delta                    ← top-level changes: FINAL stop_reason + CUMULATIVE usage
message_stop
(ping events may appear anytime; error events are possible, e.g. overloaded_error)
```

**How to read that skeleton (common confusion).** It is **not** a flat list of "seven events, once each." It is two layers:
- **The outer wrapper fires exactly once per response:** `message_start` at the very top → (all the blocks) → `message_delta` → `message_stop` at the very end.
- **The middle group — `content_block_start` → `content_block_delta`(s) → `content_block_stop` — repeats once *per content block*.** The indentation above is showing *one* such group.

So two counts vary, which is why no two traces look the same:
1. **The number of `content_block_delta` events is "one or more," not two.** The skeleton drew two only to mean *"these repeat."* `"Hi there!"` arrives in 2 pieces; a long paragraph might be 200; a one-word reply, 1.
2. **The start→delta→stop group repeats once per block.** Plain text answer = **1** block (the group appears once). *Think, then answer* = **2** blocks. *Think, answer, call a tool* = **3** blocks — that's why the full trace further below stacks three `content_block_start`…`content_block_stop` sections.

Nothing is missing in the examples: `message_start`, `content_block_start`, its `content_block_delta`s, `content_block_stop`, `message_delta`, `message_stop` are **always** there. A single-block response just shows the middle group **once**; a three-block response shows the *same* group **three times**. (Pattern fixed; counts vary.)

*The same skeleton with real data — one text block ("Hi there!") streamed in two pieces.* Each labelled line of the skeleton becomes one SSE message: an `event:` line plus a `data:` line. *(On the wire each `data:` is a single compact line; pretty-printed here for reading.)*
```
event: message_start                    ← the shell: content empty, stop_reason not decided
data: {
  "type": "message_start",
  "message": {
    "id": "msg_01…",
    "role": "assistant",
    "model": "claude-opus-5-5",
    "content": [],
    "stop_reason": null,
    "usage": { "input_tokens": 12, "output_tokens": 1 }
  }
}

event: content_block_start              ← block index 0 begins (empty text)
data: {
  "type": "content_block_start",
  "index": 0,
  "content_block": { "type": "text", "text": "" }
}

event: content_block_delta              ← first piece
data: {
  "type": "content_block_delta",
  "index": 0,
  "delta": { "type": "text_delta", "text": "Hi" }
}

event: content_block_delta              ← second piece
data: {
  "type": "content_block_delta",
  "index": 0,
  "delta": { "type": "text_delta", "text": " there!" }
}

event: content_block_stop               ← block 0 done → text = "Hi" + " there!" = "Hi there!"
data: {
  "type": "content_block_stop",
  "index": 0
}

event: message_delta                    ← FINAL stop_reason + CUMULATIVE usage
data: {
  "type": "message_delta",
  "delta": { "stop_reason": "end_turn", "stop_sequence": null },
  "usage": { "output_tokens": 5 }
}

event: message_stop                     ← terminator (no payload)
data: {
  "type": "message_stop"
}
```
- **`content_block_delta` — the incremental content of *one* block**, matched to its block by `index`. The **`delta.type`** tells you what the piece is and where it goes:
  - **`text_delta`** → `delta.text`: a chunk of visible text. Append it to that block's text — **this is what you stream to the user.**
  - **`input_json_delta`** → `delta.partial_json`: a fragment of a `tool_use` block's input arguments, as a **raw JSON *string* piece**. Concatenate the pieces; they're not valid JSON alone, so you only `json.loads` the whole thing at `content_block_stop`.
  - **`thinking_delta`** → `delta.thinking`: a chunk of extended/adaptive reasoning. Accumulate it like text, but **don't show it as the answer** (buffer, don't forward live).
  - **`signature_delta`** → `delta.signature`: a one-off cryptographic signature sent **just before a thinking block's `content_block_stop`**; it verifies the thinking block's integrity. Keep it attached to the thinking block (if the turn continues into tool use you must return the thinking block *and* its signature unmodified).

  *What each delta looks like (pretty-printed; real wire is one line each):*
  ```
  # text_delta — visible text → forward LIVE to the user
  data: {
    "type": "content_block_delta",
    "index": 0,
    "delta": { "type": "text_delta", "text": "The weather in San" }
  }

  # input_json_delta — ONE fragment of a tool's arguments → buffer, don't parse yet.
  # partial_json is itself a STRING holding a piece of JSON text, so the quotes inside
  # it are backslash-escaped (\"). That escaping is why it looks noisy — it's unavoidable.
  data: {
    "type": "content_block_delta",
    "index": 1,
    "delta": { "type": "input_json_delta", "partial_json": "{\"city\": \"San Fr" }
  }
  data: {
    "type": "content_block_delta",
    "index": 1,
    "delta": { "type": "input_json_delta", "partial_json": "ancisco\"}" }
  }
  #   → concatenate the two partial_json strings (unescaped) → {"city": "San Francisco"}
  #   → json.loads that ONCE, at this block's content_block_stop

  # thinking_delta, then a one-off signature_delta just before the block closes
  data: {
    "type": "content_block_delta",
    "index": 0,
    "delta": { "type": "thinking_delta", "thinking": "Let me check the forecast…" }
  }
  data: {
    "type": "content_block_delta",
    "index": 0,
    "delta": { "type": "signature_delta", "signature": "EqoBCkgIAR…" }
  }
  ```
- **`message_delta` — the turn's final top-level info**, emitted **once near the end**, after every content block has closed. This is where you read **how the turn ended**:
  ```
  # final stop_reason / stop_sequence + CUMULATIVE usage for the whole response
  data: {
    "type": "message_delta",
    "delta": { "stop_reason": "end_turn", "stop_sequence": null },
    "usage": { "output_tokens": 15 }
  }
  ```
  - **`delta.stop_reason`** (+ `delta.stop_sequence`) — the final stop reason (`end_turn`, `tool_use`, …) and the matched stop string if any. **Branch your loop here** (§3.1).
  - **`usage`** — **cumulative** token totals for the whole response (not a per-event increment); server-tool turns add fields like `server_tool_use`. The **`message_stop`** event right after it is just the terminator (no payload).

  *Putting it together — one complete turn, every event.* A single response where Claude **thinks**, says a line of **text**, then calls a **tool** (`stop_reason: "tool_use"`). Real SSE sends an `event:` line and a `data:` line per message:
  *(Pretty-printed; on the wire each `data:` is one line.)*
  ```
  event: message_start
  data: {
    "type": "message_start",
    "message": {
      "id": "msg_01…",
      "role": "assistant",
      "model": "claude-opus-5-5",
      "content": [],
      "stop_reason": null,
      "usage": { "input_tokens": 472, "output_tokens": 1 }
    }
  }

  # ── block index 0 : a THINKING block ──
  event: content_block_start
  data: {
    "type": "content_block_start",
    "index": 0,
    "content_block": { "type": "thinking", "thinking": "", "signature": "" }
  }
  event: content_block_delta
  data: {
    "type": "content_block_delta",
    "index": 0,
    "delta": { "type": "thinking_delta", "thinking": "User wants SF weather — call the tool." }
  }
  event: content_block_delta              ← signature arrives just before the stop
  data: {
    "type": "content_block_delta",
    "index": 0,
    "delta": { "type": "signature_delta", "signature": "EqoBCkgIAR…" }
  }
  event: content_block_stop
  data: { "type": "content_block_stop", "index": 0 }

  # ── block index 1 : a TEXT block ──
  event: content_block_start
  data: {
    "type": "content_block_start",
    "index": 1,
    "content_block": { "type": "text", "text": "" }
  }
  event: content_block_delta              ← forward LIVE
  data: {
    "type": "content_block_delta",
    "index": 1,
    "delta": { "type": "text_delta", "text": "Let me look that up." }
  }
  event: content_block_stop
  data: { "type": "content_block_stop", "index": 1 }

  # ── block index 2 : a TOOL_USE block ──
  event: content_block_start              ← input:{} is a PLACEHOLDER, not the real args
  data: {
    "type": "content_block_start",
    "index": 2,
    "content_block": { "type": "tool_use", "id": "toolu_01…", "name": "get_weather", "input": {} }
  }
  event: content_block_delta
  data: {
    "type": "content_block_delta",
    "index": 2,
    "delta": { "type": "input_json_delta", "partial_json": "{\"city\": \"San Fr" }
  }
  event: content_block_delta
  data: {
    "type": "content_block_delta",
    "index": 2,
    "delta": { "type": "input_json_delta", "partial_json": "ancisco\"}" }
  }
  #   → glue the partial_json pieces → {"city": "San Francisco"}; json.loads at the stop below
  event: content_block_stop
  data: { "type": "content_block_stop", "index": 2 }

  # ── the turn wraps up ──
  event: message_delta                    ← final stop_reason → branch; usage is CUMULATIVE
  data: {
    "type": "message_delta",
    "delta": { "stop_reason": "tool_use", "stop_sequence": null },
    "usage": { "output_tokens": 57 }
  }
  event: message_stop
  data: { "type": "message_stop" }
  (a ping event may land between any two of these — ignore it)
  ```

  *The frame never changes — only two things do.* Every block, whatever its type, is the **same envelope**: `content_block_start` → one or more `content_block_delta` → `content_block_stop`, all sharing one `index`; and the whole turn is always wrapped by `message_start` … `message_delta` → `message_stop`. Across block types **only two things differ:**
  1. the **`content_block_start`'s `content_block.type`** — `text` / `thinking` / `tool_use` (/ `server_tool_use`, `redacted_thinking`, …), which tells you which kind of block this slot holds;
  2. the **`delta.type`** inside its deltas and the **field that carries the payload** — `text_delta`→`text`, `thinking_delta`→`thinking` (plus a one-off `signature_delta`→`signature` right before its stop), `input_json_delta`→`partial_json`.

  Everything else — the event names, the `index` bookkeeping, the top-level `message_*` wrapper — is **identical regardless of block type.** So to handle a new block type you only learn its `content_block.type` and its `delta.type`; the loop around it stays the same.
- **Benefit:** better **perceived latency** — tokens appear as generated. **Total generation time and cost are unchanged.**
- Clients should ignore unknown event types gracefully (Anthropic may add more).

**Consuming the stream — the decisions the exam asks about.** The events tell your loop *when* each thing is ready:

| Question | Answer |
|---|---|
| When is a **text block** finished / safe to use? | At that block's **`content_block_stop`** (matched by `index`). The block's text = all its `text_delta` `text` values joined in order. |
| When can I **forward text to the user**? | **Immediately** — relay each **`text_delta`** as it arrives (that's the point of streaming). **Don't** forward `input_json_delta` (partial JSON) or `thinking_delta` live. |
| When is a **`tool_use` block** complete? | After its **`content_block_stop`**. The `content_block_start` gives the `id` + `name` but an **empty `input: {}`** — a placeholder, *not* the arguments. The real arguments arrive as **`input_json_delta.partial_json`** string fragments you **accumulate**, then **`json.loads` once at `content_block_stop`** (fragments aren't valid JSON on their own; the final `input` is always an object). |
| When do I actually **run the tool**? | Only after the turn ends: the tool block's **`content_block_stop`** *and* the final **`stop_reason: "tool_use"`** in **`message_delta`**. Then send `tool_result`s back (§4). |
| When is the **whole message** done? | At **`message_stop`**. The preceding **`message_delta`** holds the final **`stop_reason`**, **`stop_sequence`**, and **cumulative `usage`** (totals, not per-event increments). |
| Several blocks at once? | Every event carries an **`index`** = its slot in the final `content` array. Key your accumulator by `index` (a text block and a tool call can interleave). |

**Mental model:** `*_delta` = *still arriving* (buffer it) · `content_block_stop` = *this block is final* (parse / act on it) · `message_delta` → `message_stop` = *the turn is final* (read `stop_reason`, branch). **Text** is the one thing you relay live; **tool inputs and thinking** you buffer until their block closes.

*Condensed accumulator (raw events):*
```python
blocks, stop_reason = {}, None
for e in stream:                                    # stream=True
    if e.type == "content_block_start":
        cb = e.content_block
        blocks[e.index] = {"type": cb.type, "text": "", "json": "",
                           "id": getattr(cb, "id", None), "name": getattr(cb, "name", None)}
    elif e.type == "content_block_delta":
        b, d = blocks[e.index], e.delta
        if d.type == "text_delta":         b["text"] += d.text; print(d.text, end="")  # forward LIVE
        elif d.type == "input_json_delta": b["json"] += d.partial_json                  # buffer, don't parse
    elif e.type == "content_block_stop":
        b = blocks[e.index]
        if b["type"] == "tool_use":        b["input"] = json.loads(b["json"] or "{}")   # parse ONCE, here
    elif e.type == "message_delta":        stop_reason = e.delta.stop_reason            # final stop_reason
    elif e.type == "message_stop":         break
# if stop_reason == "tool_use": execute the collected tool_use blocks, then return tool_result(s)
```

**Prefer the SDK helpers.** `with client.messages.stream(...) as stream:` gives **`stream.text_stream`** (text only, to relay) and **`stream.get_final_message()`** (the fully reassembled `Message`, identical to a non-streamed call) — so you rarely hand-accumulate. The Python/TS helpers also expose parsed tool `input` incrementally.

**Mid-stream failures.** A stream can emit an **`error`** event (e.g. `overloaded_error` = the streamed form of a 529) — treat it as a failure; mid-stream errors **don't** follow the normal auto-retry path (§7). To resume: on **Claude 4.5 and earlier**, resend the partial text as the *start of an assistant message* and continue; on **4.6+**, put the partial text in a *user* message asking Claude to continue. **`tool_use` and `thinking` blocks can't be partially recovered** — resume from the last *complete* text block. **`ping`** events can appear anytime — ignore them.

*What the two "anytime" events look like:*
```
# ping — keep-alive, carries nothing → ignore it
event: ping
data: { "type": "ping" }

# error — the stream failed partway (here a 529 in streamed form)
event: error
data: {
  "type": "error",
  "error": { "type": "overloaded_error", "message": "Overloaded" }
}
```

*Resuming after an error.* Say the stream died having delivered only the text `"The capital of France is "`. You don't restart from scratch — you feed that partial text back and ask Claude to keep going. The **shape depends on the model**:
```python
# You captured this much before the error event arrived:
partial = "The capital of France is "

# ── Claude 4.5 and earlier: PREFILL — partial text is the START of an assistant turn ──
messages = [
    {"role": "user",      "content": "What is the capital of France?"},
    {"role": "assistant", "content": partial},          # Claude continues FROM here
]
# → it streams "Paris." as the continuation of the same sentence.

# ── Claude 4.6+: CONTINUE — partial goes in a USER message asking it to carry on ──
messages = [
    {"role": "user",      "content": "What is the capital of France?"},
    {"role": "assistant", "content": partial},
    {"role": "user",      "content": "Continue exactly from where you left off."},
]
```
Key point: only **complete text** is safe to replay this way. If the stream cut off inside a **`tool_use`** block (half-built `partial_json`) or a **`thinking`** block, you **can't** splice the fragment — discard back to the **last block that reached its `content_block_stop`** and resume from there.

> **Trap:** streaming is **SSE**, not WebSockets; the **text** is in `content_block_delta`, the **final `stop_reason`/`usage`** is in `message_delta`. A mid-stream **`error`** event is a failure you handle yourself — it is **not** covered by the SDK's automatic retry/backoff (that only applies to a request that fails *before* the stream opens).

**Streaming in production — why it freezes or arrives all at once (debugging the transport path).** This is the single most common *deployment* streaming question. Claude streaming is **ordinary SSE over one long-lived HTTP response** — there is no special transport. So once it's deployed, the tokens have to survive **every hop between Claude and the browser**: the SDK's long-lived HTTP response → your server relay → reverse proxy / ingress (nginx) → load balancer → CDN → the browser. Any one of those hops can break the *drip* without breaking the *request*. Two signature symptoms:

| Symptom | Cause | Fix (the transport, not the model) |
|---|---|---|
| **Arrives all at once** (batched at the end) | A hop is **buffering** the response body and flushing only on close — proxies/CDNs/compression buffer by default. | Turn off response buffering on that hop: nginx `proxy_buffering off;` or emit header **`X-Accel-Buffering: no`**; **flush after every event** in your relay; disable gzip/compression buffering on the stream. |
| **Freezes after 30–60 s**, then dies | An **idle / read timeout** on a hop (proxy, LB) is killing the long-lived connection; or backpressure from a slow client. | **Raise the read/idle timeouts** on proxy + LB; rely on periodic **`ping`**/keep-alive events to keep it "active"; handle **client disconnect** to cancel the upstream request (don't leak it). |

- **"Works locally, breaks in prod" is the tell.** Locally there's no proxy / LB / CDN in the path, so the buffering and timeout hops *only exist in production*. Combined with logs that show the **request starts fine and there are no prompt/model errors**, that points **away** from the agent code (tool loop, subagents, memory) and **toward the transport path**.
- **The right move is to trace the whole path end-to-end:** SDK long-lived HTTP response → are you **flushing** each chunk in your server relay? → is the **proxy/ingress buffering**? → are **LB/idle timeouts** too short? → **backpressure** on a slow browser? → do you **cancel upstream** when the browser disconnects?

> **Exam traps on this scenario (all three are wrong fixes):**
> - **Don't switch the browser-facing hop to WebSockets.** Streaming is *already* SSE over HTTP; changing the protocol on one hop while leaving the **buffering/timeout hops unchanged** doesn't touch the cause (and your backend still reads SSE from Claude). The fix is **transport configuration**, not a new protocol.
> - **Don't lower `max_tokens` or switch to a "faster" model tier.** A freeze/batch means bytes are being **withheld in transit**, not that the model is "reasoning too long." Model changes won't move a proxy's buffer.
> - **Don't just wrap a generic retry around the SDK call.** A stream that *succeeds but is buffered* isn't an error to retry; and the SDK's transport isn't the thing to "abstract away" here — the bug is downstream of it.

> **"But a stream sends data constantly — how can it time out?"** It isn't constant. A stream has **silent gaps**: before the first token (**TTFT** — a long prompt, retrieval, or extended thinking) and **mid-stream** during a server-tool run (`pause_turn`) or a generation stall. An **idle / read timeout** fires during one of those gaps — which is exactly what the periodic **`ping`** keep-alive events exist to prevent. Separately, a hop may enforce an **absolute / max-duration cap** that closes the connection at a **fixed wall-clock time even while bytes are still flowing** (common on API gateways and serverless limits). Quick read: *"dies at the same elapsed time every run"* → **absolute cap**; *"freezes after a quiet stretch"* → **idle timeout**. And raising **your own** client/SDK timeout won't help — the timer killing the stream lives on an **intermediary hop you don't control**, so you raise it **there** (or switch to streaming/Batches so bytes keep moving).

### 3.3 Token counting
**`POST /v1/messages/count_tokens`** returns the **input token count** for a payload **without generating a response** — and it's **free** (subject only to its own rate limit).

```python
resp = client.messages.count_tokens(
    model="claude-opus-5-5",
    system="You are a scientist",
    messages=[{"role": "user", "content": "Hello, Claude"}],
)
print(resp.json())   # { "input_tokens": 14 }
```
- Accepts the **same inputs** as a real call — `messages` (incl. prefill), `system`, `tools`, images, PDFs — so you can **budget** and check you're under the window before spending.
- It's an **estimate** (actual usage can differ slightly); you're **not billed** for system-added tokens.
- Not supported in the count: server tools (web search, code exec) and the **MCP connector** error out; **images/PDFs must be base64** (no url/file sources here).
- **Tokenizer note:** Claude **4.7 and later** (and Mythos Preview) use a newer tokenizer that produces **~30% more tokens** for the same text — recount against the model you'll actually use.

---

## 4. Tool use — the turn flow

### 4.1 The tool-use loop
Tools let Claude call *your* code. The flow is a loop, and the key idea is that **the model never runs the tool — it *requests*, your app executes, you feed the result back.**

```
1. You send:    messages + tools
2. Assistant:   {type:"tool_use", id:"toolu_123", name:"get_weather", input:{…}}
                stop_reason == "tool_use"
3. You run the tool, then send a NEW user message:
                {role:"user", content:[
                  {type:"tool_result", tool_use_id:"toolu_123",
                   content:"18°C, clear", is_error:false}
                ]}
4. Assistant:   uses the result → final answer (end_turn)
```
- **`tool_use_id` must match** the `id` from the `tool_use` block — results link to calls by **id, not order**.
- Mark a failed tool call with **`is_error: true`** on its `tool_result` so Claude can recover.
- Keep the **same `tools` array** every iteration, and carry the full history (the API is stateless, §2.3).

### 4.2 Single vs parallel tool calls
Claude may request **one tool, or several in the same response** (parallel tool use). Either way you reply with **exactly one `user` message** carrying **one `tool_result` per `tool_use`**, each linked by `tool_use_id`:

```jsonc
// assistant response (stop_reason: "tool_use") — TWO tool_use blocks
"content": [
  {"type":"tool_use","id":"toolu_01","name":"get_weather","input":{"city":"Paris"}},
  {"type":"tool_use","id":"toolu_02","name":"get_time","input":{"city":"Paris"}}
]
// YOUR next user turn — BOTH results in ONE user message
"content": [
  {"type":"tool_result","tool_use_id":"toolu_01","content":"18°C, clear"},
  {"type":"tool_result","tool_use_id":"toolu_02","content":"14:05 CET"}
]
```
- To force **at most one** call per turn, set **`disable_parallel_tool_use: true`** (inside `tool_choice`).
- `tool_choice` options: **`auto`** (model decides), **`any`** (must call some tool), **`tool`** (must call a named one), **`none`**.
- Full tool mechanics (schemas, server tools, ACI design) live in **Domain 8** — this is the Messages-API view.

### 4.3 The `tools` array and `tool_choice` shapes
The exact request-body shapes, for the "which fields go in a tool definition?" questions. *(Deep tool/ACI design: Domain 8.)*

**A custom (client) tool** — one object per tool in `tools`:

```jsonc
{
  "name": "get_weather",                       // required — regex ^[a-zA-Z0-9_-]{1,128}$
  "description": "Get current weather …",      // required — what/when/how, aim for 3–4+ sentences
  "input_schema": {                            // required — a JSON Schema object
    "type": "object",
    "properties": { "city": {"type":"string"} },
    "required": ["city"]
  },
  "cache_control": {"type":"ephemeral","ttl":"5m"}, // optional — cache breakpoint here (ttl "5m"|"1h", §6.1)
  "strict": true,                              // optional — constrained decoding on names/inputs
  "defer_loading": true,                       // optional — omit from initial prompt, load via tool search (can't also carry cache_control)
  "input_examples": [{"city":"San Francisco"}],// optional — example inputs (not on server tools)
  "allowed_callers": ["direct"]                // optional — "direct" (model, default) and/or "code_execution_20260120"
}
```

| Field | Required | Notes |
|---|---|---|
| `name` | ✅ | `^[a-zA-Z0-9_-]{1,128}$`. |
| `description` | ✅ | Plain text — what it does, when to use it, how it behaves. |
| `input_schema` | ✅ | JSON Schema (`type` / `properties` / `required`). |
| `cache_control` | — | `{"type":"ephemeral"}` (+ optional `ttl`) — a prompt-cache breakpoint. |
| `strict` | — | Guarantee schema-valid tool names/inputs. |
| `defer_loading` | — | Load on demand via tool search; **can't** combine with `cache_control`. |
| `input_examples` | — | Example inputs; user-defined & Anthropic-schema client tools only. |
| `allowed_callers` | — | Who may call it: `"direct"` (default) and/or `"code_execution_20260120"`. |

**An Anthropic-defined tool** (server *or* client) is instead a **dated `type` string with no `input_schema`** — Anthropic owns the schema:

```jsonc
"tools": [
  {"type":"web_search_20260318","name":"web_search"},               // SERVER tool — Anthropic runs it
  {"type":"bash_20250124","name":"bash"},                           // CLIENT tool — YOU run it (§9.2)
  {"type":"text_editor_20250728","name":"str_replace_based_edit_tool"}
]
```
(Exact date strings, and which need a beta header — e.g. `computer-use-2025-11-24` — are Domain 8.)

**`tool_choice`** — how Claude picks (the `tools` array is the same either way):

```jsonc
{"type":"auto"}                         // model decides (default when tools are present)
{"type":"any"}                          // must call SOME tool
{"type":"tool","name":"get_weather"}    // must call THIS tool
{"type":"none"}                         // no tools (default when tools absent)
// any of the above may add:  "disable_parallel_tool_use": true   → at most one call per turn
```

> **⚠️ Current-model caveat (verified):** on the newest models (**Opus 5.5, Sonnet 5.5, Fable 5.2/5.1, Mythos 5.2/5.1**), **`any` and `tool` now return a 400**. To force tool output, use **`strict` tool use under `auto`**, or **structured outputs** (§6.4); `auto` and `none` still work. Also: `any`/`tool` prefill the assistant turn (no natural-language text before the `tool_use`), and **changing `tool_choice` invalidates cached blocks**.

---

## 5. Multimodal & file input

### 5.1 Vision (images)
Send images as **`image` content blocks** in a `user` message, from one of **three source types**: **base64**, a **URL**, or a **`file_id`** from the Files API (§5.3). Put **images before text** — Claude works best that way.

```python
{"type": "image", "source": {"type": "base64",
  "media_type": "image/png", "data": img_b64}}
# or {"type":"url","url":"https://…"} , or {"type":"file","file_id":"file_…"}
```
- **Formats:** JPEG, PNG, GIF, WebP. Animations aren't supported — only the **first frame** of a GIF is read.
- **Limits:** up to **100 images/request** on 200k-context models (**600** on others); max **8000×8000 px**; max **10 MB** base64 (first-party API; 5 MB on Bedrock). If a request has **>20 images**, a stricter per-image dimension limit applies (keep each ≤ **3000 px**, or ≤ 20 image+document blocks).
- **Token cost:** images are billed as input tokens, processed in **28×28-px patches** → `⌈width/28⌉ × ⌈height/28⌉` visual tokens. Oversized images are **downscaled** first. Resolution tiers: **Claude 4.7+** high-res (long edge ≤ 2,576 px, ≤ 4,784 tokens); **others** standard (≤ 1,568 px, ≤ 1,568 tokens). (A 1000×1000 image ≈ **1,296** tokens.)

### 5.2 PDFs & documents
Send PDFs as **`document` content blocks**, again via **url / base64 / `file_id`**. Claude reads **both the text and the visuals** of each page:

> The system **converts each page into an image** *and* **extracts the page's text**, then analyses both — so Claude can read charts, tables, and diagrams, not just the text layer.

```python
{"type": "document", "source": {"type": "file", "file_id": file_id}}
# or {"type":"base64","media_type":"application/pdf","data":pdf_b64}, or {"type":"url","url":"…"}
```
- **Limits:** request ≤ **128 MB**; ≤ **600 pages** (**100** when the request's context window is under 1M tokens); **standard PDFs only** (no passwords/encryption). Dense pages can fill the window before the page limit.
- **Token cost:** text ≈ **1,500–3,000 tokens/page** plus the per-page **image** tokens (same vision maths as §5.1). No extra "PDF fee."
- **Plain text** (`.txt`, `.csv`, `.md`) works as a `document` with MIME `text/plain`. **Binary formats (`.xlsx`, `.docx`) are *not* supported** — convert to text or PDF first.
- For large or reused docs: upload via the **Files API** and reference by `file_id`, and/or add **`cache_control`** to the document block (§6.1).

### 5.3 The Files API
The Files API is the **"upload once, reference many times"** pattern: upload a file, get a **`file_id`**, and point to it from `document`/`image` blocks in any later request — so you never re-send the bytes. **It's out of beta — no beta header needed.**

> **What `file_id` saves — and what it doesn't.** It cuts **upload / transfer** (bandwidth, payload size, re-encoding) — **not tokens.** The document is still read and **tokenised on every request**, so your **input-token cost is identical** whether you send base64 or a `file_id`. To cut the *token/cost* of a **reused** document, that's **prompt caching** (§6.1) — cache reads bill at a fraction of input. Mnemonic: **`file_id` = transfer efficiency; prompt caching = token efficiency** (they stack).

| Operation | Endpoint |
|---|---|
| Upload | `POST /v1/files` |
| List | `GET /v1/files` |
| Metadata | `GET /v1/files/{id}` |
| Download | `GET /v1/files/{id}/content` |
| Delete | `DELETE /v1/files/{id}` |

```python
up = client.files.upload(file=("doc.pdf", open("doc.pdf","rb"), "application/pdf"))
client.messages.create(model="claude-opus-5-5", max_tokens=1024, messages=[{
  "role":"user","content":[
    {"type":"text","text":"Summarise this."},
    {"type":"document","source":{"type":"file","file_id": up.id}}]}])
```
- **Types:** PDF & plain text → `document`; images → `image`; datasets for code execution → `container_upload`.
- **Limits:** **500 MB/file**, **1 TB/organization**. Set **`expires_in_seconds`** at upload (**1 hour – 90 days**, fixed once set); otherwise files persist until deleted. Deleted files are unrecoverable.
- **You can't download files *you* uploaded** (only outputs created by skills/code-execution are downloadable) — trying returns 400.
- **Security (verified, important):** files are **not scoped to a user, conversation, or session** — any API key in the workspace can read any file in it. **Never accept a `file_id` from an end user or untrusted source**; track the user→file mapping in your app, and use a **separate workspace per tenant** for multi-tenant apps.

---

## 6. Advanced API surface
*(Deep cost/latency treatment is in Domain 5 and context engineering in Domain 6 — here's the API surface you invoke.)*

### 6.1 Prompt caching
Mark a stable **prefix** (tools / system / long docs) with **`cache_control: {"type":"ephemeral"}`** so later calls **re-read it cheaply** instead of reprocessing it. Cache **writes** show as `cache_creation_input_tokens`; **hits** as `cache_read_input_tokens` (much cheaper). Up to **4 breakpoints**; TTL **5 min** default or **1 h**. Great for a big system prompt or document reused across turns. *(Full economics, multipliers, and min-lengths: Domain 5 §2.1.)*

### 6.2 Message Batches
**`POST /v1/messages/batches`** runs a big pile of requests **asynchronously** for **50% off** both input and output. **Most finish < 1 hour; 24 hours is the hard *expiry* cap, not a target.** A batch holds up to **100,000 requests or 256 MB**; results are kept **29 days**; batches are **Workspace-scoped**. **Not supported inside a batch:** streaming, Fast mode (`speed`), server-side `fallbacks`, `max_tokens:0`. Use for latency-tolerant bulk work (evals, moderation, bulk generation) — **never** on an interactive path. *(Domain 5 §2.2.)*

### 6.3 Thinking (adaptive vs manual)
Give Claude room to reason before answering. There are a few `thinking` modes — and the manual one is now deprecated:
- **`adaptive`** (current) — `thinking: {"type":"adaptive"}`; Claude **decides per request whether and how much to think**, with depth set by **`output_config.effort`** (`low`→`max`). At low effort it may skip thinking on easy inputs.
- **`disabled`** / **`between_tools`** — off, or interleaved between tool calls.
- **`enabled` + `budget_tokens`** is **DEPRECATED** — **Claude 4.7 and later reject it with a 400**; it survives only on older models for predictable-latency/precise-cost cases.
- Either way, **thinking tokens are billed as output** and count toward `max_tokens` and the window (Domain 6). *(Reasoning levers: Domain 5 §2.4.)*

### 6.4 Structured outputs & citations
- **Structured outputs** — pass a JSON schema in **`output_config.format`** to constrain Claude's final text to **schema-valid JSON** (verified structured output). *(Domain 6 covers this vs. strict tool use.)*
- **Citations** — set **`citations.enabled: true`** on your `document` blocks (**all-or-none** within a request). Claude then returns **text blocks with a `citations` list**, each citation giving **`cited_text`** plus a location: **character indices** (plain text), **page numbers** (PDF), or **block indices** (custom content). `cited_text` is free (doesn't count as output). **Only text citations** exist (no image citations), and **citations cannot be combined with structured outputs** (`output_config.format`) — that returns a **400**.

> **Exam trap (parse reliability).** Scenario: *"Claude's JSON intermittently fails to parse — extra prose, a trailing sentence, a missing field."* The right fix is **structured outputs (`output_config.format`)** or **`strict` tool use** so the output is **guaranteed schema-valid**, plus **validate and handle parse errors** in your wrapper (§9.4). **Wrong** answers: scraping the text with regex, "just ask it more firmly" in the prompt, or lowering temperature — none of those *guarantee* valid JSON.

### 6.5 Embeddings (Anthropic doesn't offer them)
A small but classic gotcha: **Anthropic does *not* provide its own embedding model or an embeddings endpoint.** The Messages API generates text, calls tools, and reasons — it does **not** turn text into vectors. For semantic search / RAG retrieval you reach for a **third-party embeddings provider** (Anthropic's docs point to **Voyage AI by MongoDB** as one option, while telling you to evaluate vendors for your use case).

> **Exam trap:** "Which Anthropic endpoint produces embeddings?" → **none**. Any answer claiming a `/v1/embeddings` route or an Anthropic embedding model is wrong — embeddings come from an outside provider.

### 6.6 The MCP connector (`mcp_servers`)
Call **remote MCP servers** straight from the Messages API, with no MCP client code of your own. *(Full MCP model: Domain 8.)* Needs a beta header: **`mcp-client-2025-11-20`** (or its superset **`mcp-client-2026-09-15`**).

```jsonc
"mcp_servers": [
  {
    "type": "url",                             // required — only "url" is supported
    "url": "https://mcp.example.com/sse",      // required — must start with https://
    "name": "example-mcp",                     // required — unique; referenced by one mcp_toolset
    "authorization_token": "YOUR_OAUTH_TOKEN"  // optional — if the server needs OAuth
  }
]
```
Which of that server's tools are exposed is set by a matching **`mcp_toolset`** entry in the **`tools`** array:

```jsonc
"tools": [
  {
    "type": "mcp_toolset",
    "mcp_server_name": "example-mcp",
    "default_config": {"enabled": false},
    "configs": { "tool1": {"enabled": true}, "tool2": {"enabled": true, "defer_loading": false} }
  }
]
```
- **Remote / HTTP only** — Streamable HTTP or SSE; **local stdio servers can't be connected** this way.
- Each tool option takes `enabled` (default `true`) and `defer_loading` (default `false`).
- *(Legacy `mcp-client-2025-04-04` put the allowlist in a `tool_configuration: {enabled, allowed_tools}` on the server entry — deprecated; use `mcp_toolset` now.)*

### 6.7 `context_management` (compaction and context editing)
Ask the API to **prune old context automatically** so long / agentic conversations don't blow the window (§1.4). *(Context-engineering strategy: Domain 6.)* Needs beta header **`context-management-2025-06-27`**.

```jsonc
"context_management": {
  "edits": [
    { "type": "clear_thinking_20251015",             // if present, must be FIRST in edits
      "keep": {"type":"thinking_turns","value":2}    // or "all"
    },
    { "type": "clear_tool_uses_20250919",             // tool-result clearing
      "trigger":        {"type":"input_tokens","value":30000},  // when to start (default 100000)
      "keep":           {"type":"tool_uses","value":3},          // recent pairs to keep (default 3)
      "clear_at_least": {"type":"input_tokens","value":5000},    // else the edit is skipped
      "exclude_tools":  ["web_search"],                          // never clear these tools' results
      "clear_tool_inputs": false                                 // also drop the call args? (default false)
    }
  ]
}
```

| Edit type | Clears | Key fields (defaults) |
|---|---|---|
| **`clear_tool_uses_20250919`** | old tool-use/result pairs | `trigger` (100k input tokens), `keep` (3 tool uses), `clear_at_least`, `exclude_tools`, `clear_tool_inputs` (`false`) |
| **`clear_thinking_20251015`** | old thinking blocks | `keep` (`thinking_turns` N>0, or `"all"`); **must come first** if both edits are used |

---

## 7. Errors & retries
Failures come back as HTTP **4xx/5xx** with a JSON body `{ "type":"error", "error": {"type","message"}, "request_id" }`:

| Code | Type | Action |
|---|---|---|
| `400` | `invalid_request_error` | Fix the request (also returned when you hit a configured spend limit, except Claude Code workspaces → 429). |
| `401` | `authentication_error` | Bad/missing key — check `x-api-key`. |
| `402` | `billing_error` | Billing/payment issue. |
| `403` | `permission_error` | Key lacks access to the resource. |
| `404` | `not_found_error` | Wrong path or resource id. |
| `409` | `conflict_error` | Conflicts with current state (e.g. concurrent change) → resolve, retry. |
| `413` | `request_too_large` | Shrink payload. Size caps: **Messages & Token Counting 128 MB, Batch 256 MB, Files 500 MB**. |
| `429` | `rate_limit_error` | Rate/spend limit → **back off**, respect `retry-after`. |
| `500` | `api_error` | Internal error → retry with backoff (quote the `request_id` if it persists). |
| `504` | `timeout_error` | Timed out processing → retry. |
| `529` | `overloaded_error` | Service busy → retry with backoff. |

- Build **retries with exponential backoff + jitter**; the **SDKs do this automatically** (twice by default, honouring `retry-after`) — configurable via `max_retries`.
- **Two exceptions:** a 429 from a **spend cap** has no `retry-after` and keeps failing until access resumes; **mid-stream** errors (after a 200) don't follow the standard retry path — handle the stream's `error` events.
- **Mnemonic:** **4xx = your request** — fix it, don't retry (400/401/403/404/413); **5xx = Anthropic's side** — retry with backoff (500/529, and 504 `timeout_error`). A **429** is the hinge: retry **only** when it carries `retry-after` (a true rate limit), not when it's a spend cap.
- Every response carries a **`request-id`** header (and `request_id` in error bodies) — read it from `response._request_id` in the SDKs; include it in support tickets.

### Rate limits — the headers & token budgets
Beyond the 429 code, the API returns **rate-limit headers on *every* response** so you can self-throttle. Limits are enforced per model class as a continuously-replenishing **token bucket** — so a short burst can 429 even when you're under your per-minute number (60 RPM may be enforced as ~1 request/second).

Three limit types:
- **RPM** — requests per minute.
- **ITPM** — input tokens per minute. **Counts `input_tokens` + `cache_creation_input_tokens`; `cache_read_input_tokens` does *not* count** — another reason caching helps: cache **reads** don't eat your input-rate budget. `max_tokens` does **not** affect ITPM.
- **OTPM** — output tokens per minute, counted as tokens are generated.

**Headers to read** (each comes as `-limit` / `-remaining` / `-reset`): `anthropic-ratelimit-requests-*`, `anthropic-ratelimit-tokens-*`, `anthropic-ratelimit-input-tokens-*`, `anthropic-ratelimit-output-tokens-*`, plus **`retry-after`** (seconds) on a rate-limited 429. (`remaining` token counts are rounded to the nearest thousand.) Usage **tiers** — Evaluation → Start → Build → Scale → Custom — raise these automatically as your account's history grows.

> **Reinforces the 429 split above:** a **spend-cap** 429 carries **no `retry-after`** and won't recover on retry (`error.details.error_code == "enforced_spend_limit_reached"`); a true **rate-limit** 429 does — back off and honour `retry-after`.

> **Reading these headers with the official SDK (verified):** the typed `Message` object carries **no HTTP headers**. Use **`client.messages.with_raw_response.create(...)`**, read **`response.headers`**, then **`.parse()`** (which returns the same `Message`) — this **keeps the SDK's typed errors and automatic retries**. Traps: dropping to a plain HTTP request (loses typed errors/retries), reading **`message.usage`** (that's *this request's* tokens, not remaining capacity), or the **Rate Limits API** (returns *configured* limits, not the live remaining-this-minute value — which is only in the response header).

### Long-running requests — stream or batch to avoid timeouts
A whole scenario family: "a long generation **intermittently times out / fails in production**." The fix is almost always **streaming or batching**, not a bigger retry count.
- The official SDKs set a **10-minute default timeout** (600 s; 5 s to connect) and auto-retry timeouts twice.
- **Anthropic's explicit guidance:** for anything that may run long (large `max_tokens`, slow generations), use the **streaming** Messages API **or** the **Message Batches** API — *"especially those over 10 minutes."*
- **Why:** some networks **drop idle connections**, so a long **non-streaming** request can fail or time out **with no response** even though Claude was working. Streaming keeps bytes flowing so the connection stays active; **Batches** let you **poll** instead of holding one connection open.
- The Python/TS SDK **refuses** a non-streaming call it predicts will exceed 10 minutes (`ValueError: "Streaming is required for operations that may take longer than 10 minutes"`). The guard assumes ~**128k tokens/hour** (~35 tok/s), so a rule of thumb is: keep **non-streaming `max_tokens` under ~20k**; above that, stream (or deliberately raise the timeout, e.g. `timeout=1200`).
- Want one result but the safety of streaming? Open a stream and call **`stream.get_final_message()`** — same reassembled `Message`, over a kept-alive connection.
- Related errors: **504 `timeout_error`** (and sometimes a **500** when the server's time limit hits *before* the response starts) → the fix is **streaming/batch**, not blind retries. For raw-HTTP integrations, set a **TCP socket keep-alive**.

> **Exam trap:** "big outputs occasionally fail with timeouts" → **switch to streaming (or Batches)**, don't just raise `max_retries` or the timeout and hope. Non-streaming is only safe for **short** outputs.

### Safe retries & idempotency (your job — there's no idempotency key)
The Messages API has **no idempotency-key header.** Retries — the SDK's automatic 2×, *plus* any you add, *plus* a client that re-sends after a dropped connection — mean a request that **actually succeeded server-side can run again** (a second generation, **billed again**), and a **side-effecting tool handler can be invoked more than once**.
- **Idempotency is on you:** make side-effecting tool handlers **idempotent** — dedupe on an id *you* pass in (order id, request id) so a replay is a **no-op**. Guard anything that charges a card, sends mail, or writes a row.
- **Only retry what's safe:** a 429/500/529/timeout **before** any side effect is safe to retry; a retry **after** a tool already charged a customer is **not**, unless the handler dedupes.
- Keep each response's **`request-id`** to trace suspected duplicates.

> **Exam trap:** "we retry on 5xx and a customer sometimes gets **charged/emailed twice**" → fix is **idempotent handlers / dedupe keys**, *not* ripping out retries (and there's no Anthropic idempotency header to lean on).

---

## 8. SDKs & deployment platforms
- **Seven official client SDKs** wrap the same `/v1/messages` endpoint: **Python, TypeScript, C#/.NET, Go, Java, PHP, Ruby.** They all handle **auth, streaming, automatic retries/backoff, error handling, typed responses, and OpenTelemetry tracing** — anything an SDK does, you *could* do with raw HTTP.
- They set `anthropic-version` for you, so you rarely type the header yourself.
- **Deployment platforms:** the same Messages API shape is available on **Amazon Bedrock / Claude Platform on AWS**, **Google Cloud / Vertex AI** (Anthropic-managed and Google-managed), and **Microsoft Foundry** — with provider-specific auth, and some source-type limits (e.g. Bedrock/Google-managed accept **base64 only** for images/PDFs; Foundry doesn't support the Files API on Azure-hosted deployments). **Feature parity differs by platform — a matching request *shape* ≠ every feature.** On **Amazon Bedrock**, **prompt caching works**, but the **Message Batches endpoint and the Files API are NOT supported** — so a nightly batch job or a `file_id` document flow must be redesigned when porting there.
- **Discover models at runtime:** `GET /v1/models` (and `GET /v1/models/{id}`) lists available models with each one's `max_input_tokens`, `max_tokens`, and `capabilities` — useful for picking an ID and its limits programmatically instead of hard-coding them.

---

## 9. Where it runs: browser vs your backend vs Anthropic

A whole cluster of exam questions aren't about *what* a feature does but about *where the code for it lives* — on the end user's **browser** (client-side), on **your own server** (backend), or on **Anthropic's infrastructure**. Get the three "places" straight and these scenarios become easy.

**Three places, and who trusts whom:**
- **The browser / client UI** — the end user's device. **Untrusted:** anything shipped here is visible to the user and anyone who opens dev-tools.
- **Your backend** — a server *you* run. **Trusted:** this is where your **API key**, your **conversation history**, and your **tool code** belong.
- **Anthropic's infrastructure** — the Messages API and the things it hosts (the model, server tools, the Files store, the prompt cache, batch jobs).

### 9.1 The standard architecture & the API-key rule

The default, production-safe shape is **three tiers**, with your backend in the middle holding the secret:

```
┌───────────┐     ┌────────────────────────┐     ┌─────────────────────┐
│  Browser  │────▶│   YOUR backend          │────▶│  Anthropic API       │
│ (client   │◀────│   • holds x-api-key      │◀────│  api.anthropic.com   │
│  UI)      │     │   • stores history       │     │  • the model         │
│           │     │   • runs YOUR tools      │     │  • server tools      │
└───────────┘     └────────────────────────┘     └─────────────────────┘
  untrusted            trusted (secret)                 Anthropic
```

**The rule you'll be tested on: your Anthropic API key lives on the backend, never in browser or mobile client code.** A key shipped to the client can be read by anyone who opens dev-tools and used to spend your money. So the browser talks to *your* server, and *your* server (which reads the key from an environment variable / secret manager) talks to Anthropic. The browser never sees the key.

> **The CORS nuance (so you recognise the trick):** the Messages API *does* support cross-origin calls — you opt in with the header **`anthropic-dangerous-direct-browser-access: true`** — so it is *technically* possible to call Claude straight from a browser. The word **`dangerous`** is in the header name on purpose: doing so exposes your key. The only *legitimate* direct-browser cases are **(a) internal tools** used only by trusted users, and **(b) "bring-your-own-key" apps** where each user supplies *their own* key (so there's no shared secret to leak). For a normal public product, **proxy through your backend** — that's the expected exam answer.

### 9.2 Client tools vs server tools: where execution happens

Tool use has a terminology trap that is *entirely* about "where it runs." The split is **who executes the tool**, and the response tells you which:

| | **Client tools** | **Server tools** |
|---|---|---|
| **Runs on** | **your application** (your backend) | **Anthropic's infrastructure** |
| **You write a handler?** | **Yes** — your code runs it, returns a `tool_result` | **No** — you just see the results |
| **How the turn goes** | Claude replies `stop_reason: "tool_use"`; you execute, send `tool_result` back (§4.1) | Claude runs it mid-turn; may emit `pause_turn` (§3.1) |
| **Examples** | your own functions, **plus the Anthropic-schema tools** `bash`, `text_editor`, `computer`, `memory`, `browser` | `web_search`, `web_fetch`, `code_execution`, `tool_search`, and the **MCP connector** |

> **⚠️ The big trap — "Anthropic-defined ≠ Anthropic-run."** `bash`, `text_editor`, `computer`, `memory`, and `browser` have schemas *defined by Anthropic*, but they are **client tools: your application still executes them.** Anthropic defining the *shape* of the tool does **not** mean Anthropic runs it. Only `web_search` / `web_fetch` / `code_execution` / `tool_search` (and the MCP connector) actually **run on Anthropic's servers** with no handler code from you. *(Deep tool & MCP mechanics: Domain 8.)*

### 9.3 Where each piece of state & work lives

Zooming out from tools, here's the full "who holds what" map — the answer key for any "where does X live / where do I implement X" question:

| Thing | Where it lives / runs | Why / note |
|---|---|---|
| **Your API key** | **Your backend** (env var / secret manager) | Never in the browser or in client bundles (§9.1). |
| **Conversation history** | **Your backend** (your DB / store) | API is **stateless** — no `conversation_id`; you resend it each turn (§2.3). |
| **The model + inference** | **Anthropic** | You never run the model yourself. |
| **Client-tool execution** | **Your backend** | `bash` / `text_editor` / your own functions — *you* run them (§9.2). |
| **Server-tool execution** | **Anthropic** | `web_search` / `code_execution` / etc. — no handler code (§9.2). |
| **Uploaded files (Files API)** | **Anthropic** (workspace-scoped) | Referenced by `file_id`; **not** user-scoped (§5.3). |
| **Prompt cache** | **Anthropic** | You just mark the prefix with `cache_control` (§6.1). |
| **Batch jobs** | **Anthropic** (async) | You submit and poll for results (§6.2). |
| **Streaming (SSE)** | Anthropic **→ your backend →** browser | Your server relays tokens to the client; the key still stays server-side (§3.2). |
| **The Messages API itself** | Anthropic — **or** Bedrock / Vertex / Foundry | Same request shape, provider-specific auth & hosting (§8). |

> **Exam trap:** if a scenario says the API key "must be available in the app to call Claude" and offers a browser/client-side option, that's the **wrong** answer — the call goes through **your backend**. And "where does `bash` / `computer` run?" → **your application**, not Anthropic (only the `web_search`-class *server* tools run on Anthropic's infrastructure).

### 9.4 Productionizing the integration — wrap the SDK in an integration service

The other big architecture question isn't *where* the call runs but *how you structure the code around it* when Claude goes into a real production backend (customer-support drafts, etc.) that must be **reliable, observable, and safe to change**. The expected answer is almost always: **treat the Claude integration as a first-class production software component** — wrap the SDK behind **one integration service / module with a stable boundary**, and put it through **normal SDLC controls**. Don't scatter raw `client.messages.create(...)` calls across every request handler.

**What the wrapper centralizes (one place to own, one place to change):**
- **Model / prompt / schema / tool config** — the model id, system prompt, output schema, tool definitions, versioned in **one** place so a model or prompt change is a single reviewed edit, not a hunt across handlers.
- **Resilience** — **retries with backoff** (honor `retry-after`, §7/§7-rate-limits), timeouts, fallback model, circuit-breaking.
- **Error handling & parsing** — one place that turns API errors, `stop_reason`s (incl. `refusal`, `max_tokens`), and **response-parsing failures** into typed results your app can act on.
- **Observability** — structured **logging + metrics + traces** for failures, **latency**, **token usage** (from `usage`), and **parse errors** — exactly the "operator visibility" the scenario asks for.

**Why a stable boundary matters:** it's an **anti-corruption layer** — your app depends on *your* interface, not on the SDK's surface, so you can absorb **future model/prompt/API changes** (new model, new `anthropic-version`, new params) by editing the wrapper, without touching callers. It also makes behavior changes **detectable and reversible**.

**How changes ship (normal controls, not cowboy edits):** code review → **automated integration tests** → **staging smoke tests** → monitoring → **rollback plan** (and feature-flag / canary the rollout). This is what makes it "meet normal reliability standards."

> **Exam traps (the wrong answers this question pairs it with):**
> - **"Call the SDK directly from every request handler and let the SDK's abstraction handle retries/logging/config/future API changes."** Wrong — the SDK is a *transport client*, **not** your reliability, observability, or config layer. Direct calls scatter config and give no central place for retries, metrics, or safe model swaps.
> - **"Ship it, then add tests/monitoring only if users report problems."** Wrong — reliability and observability are **requirements up front** for a production service, not a reaction to incidents.
> - **"Let on-call engineers edit prompts/schemas/model choices live in prod with no review/staging."** Wrong — prompts and schemas are **behavior-defining code**; editing them unreviewed in prod is an uncontrolled change (no test, no rollback, no audit). Version them and ship them through the pipeline.

---

## 10. Common traps
- Auth header is **`x-api-key`**, **not** `Authorization: Bearer`.
- **`anthropic-version` ≠ model version** — it pins API behaviour (`2023-06-01` classic; `2026-10-21` newest, which **removes `temperature`/`top_p`/`top_k`**).
- **`max_tokens` is required**, not optional.
- There is **no `system` role** — the system prompt is a **top-level parameter**; valid message roles are only **`user`/`assistant`**.
- The API is **stateless** → **resend full history**; there is **no `conversation_id`**.
- **`content` is always an array of blocks** in responses, even for plain text.
- **`tool_result` goes in a `user` message** (not assistant/system); `tool_use_id` must match, and all parallel results go in **one** user message.
- Streaming is **SSE**, not WebSockets; text in `content_block_delta`, final `stop_reason`/`usage` in `message_delta`.
- `max_tokens` (your cap) ≠ `model_context_window_exceeded` (the whole window); `pause_turn` means **continue**; `refusal` is a **stop_reason on a 200**.
- **Prefill on the last turn is rejected (400) on Claude 4.6+/Mythos Preview** — use structured outputs or just ask for the format.
- **Manual `thinking: enabled` + `budget_tokens` is deprecated** (4.7+ → 400); use **adaptive thinking + `effort`**.
- **Citations can't be combined with structured outputs** (400); **binary files (`.xlsx`/`.docx`) aren't valid `document` blocks**.
- **Files aren't user-scoped** — never trust an externally supplied `file_id`.
- **Your API key belongs on the backend, never in browser/client code.** Direct browser calls are possible via `anthropic-dangerous-direct-browser-access: true` but expose the key (OK only for internal tools or BYO-key apps); normal products **proxy through your backend** (§9.1).
- **Client tools run in *your* app; server tools run on Anthropic.** The Anthropic-schema tools `bash`/`text_editor`/`computer`/`memory`/`browser` are **client** tools — *you* execute them. Only `web_search`/`web_fetch`/`code_execution`/`tool_search` + the MCP connector run on Anthropic. "Anthropic-defined ≠ Anthropic-run" (§9.2).
- **Conversation history lives on your backend**, not Anthropic — the API is stateless (no `conversation_id`) (§2.3, §9.3).
- **Anthropic has no embeddings model or endpoint** — use a third-party provider (docs cite **Voyage AI**); a `/v1/embeddings` answer is wrong (§6.5).
- **Three different "too big" outcomes:** `max_tokens` hit → `stop_reason:"max_tokens"` (truncated); input + output overruns the window → `model_context_window_exceeded`; input *alone* too big → **400 "prompt is too long"** (§1.4).
- **ITPM counts `input_tokens` + `cache_creation` but *not* `cache_read`** — cached reads don't count against the input-rate limit (§7).
- **Context window is 1M** on current models (no beta header; **200K** older), **128K** max output (§1.4).
- **Long/large outputs → stream or batch**, don't push `max_tokens` high on a non-streaming call. SDK default timeout is **10 min**; non-streaming is only safe for **short** outputs (~<20k tokens) (§7).
- **No idempotency-key header** — retries can double-generate (double-bill) or double-fire a side-effecting tool; make your **handlers idempotent** (§7).
- **Unreliable JSON → structured outputs / `strict` tool use + validation**, not regex or prompt-nagging (§6.4).

---

## 11. Quick-fire revision
- Q: Endpoint & method? **A:** `POST /v1/messages`.
- Q: Three required body params? **A:** `model`, `messages`, `max_tokens`.
- Q: Auth header? **A:** `x-api-key` (not Bearer).
- Q: What does `anthropic-version` set, and name a value? **A:** API **behaviour/version** (not the model); `2023-06-01` (classic) or `2026-10-21` (newest).
- Q: What does `2026-10-21` remove? **A:** **`temperature`, `top_p`, `top_k`** — sending them → 400 on every model.
- Q: Where's the system prompt? **A:** Top-level **`system`** parameter (no `system` role in `messages`).
- Q: Valid message roles? **A:** `user`, `assistant`.
- Q: Keep multi-turn context? **A:** **Resend full history** — the API is **stateless**, no `conversation_id`.
- Q: Is `content` ever a bare string in a response? **A:** No — **always an array of blocks**.
- Q: All seven `stop_reason`s? **A:** `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `refusal`, `model_context_window_exceeded`.
- Q: `stop_reason` for a tool request? **A:** `tool_use`.
- Q: What is `pause_turn` and the fix? **A:** A server-tool loop hit its limit (default **10**) → append content back and resend.
- Q: Is `refusal` an HTTP error? **A:** No — it's a `stop_reason` on an **HTTP 200**; read `stop_details`.
- Q: Which message carries `tool_result`? **A:** A **`user`** message; `tool_use_id` must match.
- Q: Two tools called at once — how many user messages back? **A:** **One**, with both `tool_result` blocks.
- Q: Streaming transport? **A:** **SSE** (not WebSockets).
- Q: Which stream event has the text vs the final stop_reason? **A:** `content_block_delta` (`text_delta`) vs `message_delta`.
- Q: In a stream, when is a text block safe to use, and the whole message? **A:** The block at its **`content_block_stop`** (text = its `text_delta`s joined by `index`); the whole message at **`message_stop`**.
- Q: Where do a streamed tool call's arguments come from, and when do you parse? **A:** Accumulate **`input_json_delta.partial_json`** fragments, then **`json.loads` once at `content_block_stop`**. The `content_block_start` `input:{}` is an empty placeholder, not the args.
- Q: When do you execute a streamed tool? **A:** After the tool block's **`content_block_stop`** *and* the final **`stop_reason:"tool_use"`** in `message_delta`.
- Q: What may you forward to the user live, and what must you buffer? **A:** Forward **`text_delta`** live; **buffer** `input_json_delta` (partial JSON) and `thinking_delta` (not final until `signature_delta`).
- Q: How do you keep multiple interleaved blocks straight? **A:** Key your accumulator by the **`index`** on every event (its slot in the final `content` array).
- Q: A stream errors mid-way — can you resume a half-finished tool call? **A:** **No** — `tool_use`/`thinking` can't be partially recovered; resume from the last complete **text** block (4.5− as an assistant prefill, 4.6+ as a user "continue").
- Q: Stream works locally but in prod **freezes after ~30–60 s** or **arrives all at once**, request starts fine, no prompt errors — best next step? **A:** **Debug the transport path**, not the agent code. SSE is a long-lived HTTP response; a prod-only freeze/batch is **proxy buffering** (→ all at once; fix with `proxy_buffering off;` / `X-Accel-Buffering: no` + flush each event) or **idle/read timeouts** on proxy/LB (→ freeze; raise timeouts, keep-alive, cancel on disconnect).
- Q: Is switching the browser hop to **WebSockets** the fix for a buffered/frozen stream? **A:** **No** — streaming is already SSE over HTTP; the cause is buffering/timeout hops. Fix the **transport config**, not the protocol. (Also wrong: lowering `max_tokens`, switching model tier, or a generic SDK retry.)
- Q: Best way to put Claude into a production backend that must be reliable, changeable, and observable? **A:** **Wrap the SDK in one integration service** with a stable boundary — centralize model/prompt/schema config, retries, logging, error/parse handling — and ship changes via **code review + integration tests + staging smoke tests + monitoring + rollback**. (Wrong: raw SDK calls in every handler; ship-first-add-tests-later; live unreviewed prompt/schema edits in prod.)
- Q: Why wrap the SDK instead of calling it directly everywhere? **A:** It's an **anti-corruption layer** — callers depend on *your* interface, so future model/prompt/`anthropic-version`/param changes are absorbed in **one** place, and behavior changes stay **detectable and reversible**. The SDK is a transport client, not your config/retry/observability layer.
- Q: A long generation intermittently **times out** in production — best fix? **A:** **Use streaming (or the Batches API)**, not a bigger retry count. Networks drop idle connections on long non-streaming calls; streaming keeps bytes flowing, Batches let you **poll**. SDK default timeout is **10 min**; keep non-streaming `max_tokens` small (~<20k).
- Q: Does the Messages API have an **idempotency key**? **A:** **No.** Retries can re-run a request that already succeeded (double-bill) or **double-fire a side-effecting tool** — so make **your** handlers idempotent (dedupe on an id you pass) and only retry what's safe.
- Q: Claude's **JSON won't parse reliably** — best fix? **A:** **Structured outputs (`output_config.format`)** or **`strict` tool use** for schema-valid output, plus validate/handle parse errors in your wrapper. Not regex-scraping or prompt-nagging.
- Q: Count tokens without generating? **A:** `POST /v1/messages/count_tokens` — **free**, returns `input_tokens` (an estimate).
- Q: Image source types? **A:** base64, URL, `file_id`; formats JPEG/PNG/GIF/WebP; images **before** text.
- Q: How are image tokens computed? **A:** 28×28-px patches → `⌈w/28⌉×⌈h/28⌉` visual tokens (downscaled first).
- Q: How does Claude read a PDF? **A:** Each page → **image + extracted text**; ≤ 600 pages (100 under 1M ctx), ≤ 128 MB.
- Q: What does the Files API give you? **A:** Upload once, reference by **`file_id`**; 500 MB/file, 1 TB/org; out of beta.
- Q: Files API security gotcha? **A:** Files are **workspace-wide, not user-scoped** — never trust an external `file_id`.
- Q: Batch discount/timing/limits? **A:** **50% off**, async (most **<1h**, **24h** = expiry cap), **100k/256 MB**, results 29 days.
- Q: Which fields appear in `usage` with caching? **A:** `cache_creation_input_tokens`, `cache_read_input_tokens`.
- Q: Current thinking mode vs deprecated? **A:** **adaptive** (depth via `effort`); **`enabled`+`budget_tokens`** is deprecated (4.7+ → 400).
- Q: Enable citations — how, and one restriction? **A:** `citations.enabled:true` on **all** documents; **can't** combine with structured outputs (400).
- Q: Does prefill still work? **A:** Not on the **last turn** for Claude **4.6+/Mythos Preview** (400); earlier models still allow it.
- Q: 429 vs 529? **A:** Rate-limited vs overloaded → both **retry with backoff**.
- Q: How many official SDKs, and name three? **A:** **Seven** — Python, TypeScript, C#, Go, Java, PHP, Ruby.
- Q: Where else does the Messages API run? **A:** Amazon Bedrock, Google Vertex AI, Microsoft Foundry (same shape, provider auth).
- Q: Where does your Anthropic API key belong? **A:** On **your backend** (env var / secret manager) — **never** in browser or mobile client code. Direct browser calls need the `anthropic-dangerous-direct-browser-access` header and leak the key (fine only for internal or BYO-key apps).
- Q: Where do `bash` / `computer` / `text_editor` tools execute? **A:** In **your application** — they're **client tools** (Anthropic defines the *schema*, you run them). Only `web_search`/`web_fetch`/`code_execution`/`tool_search` + the MCP connector are **server tools** that run on Anthropic.
- Q: Where does conversation history live? **A:** In **your backend** — the API is stateless (no `conversation_id`), so you store it and resend it each turn.
- Q: Browser → ? → Anthropic. What's in the middle, and why? **A:** **Your backend** — it holds the API key, stores history, runs your client tools, and relays streaming to the browser; the key never reaches the client.
- Q: Does Anthropic offer an embeddings model? **A:** **No** — use a third-party provider (docs cite **Voyage AI**); there is no Anthropic `/v1/embeddings`.
- Q: Context window & max output sizes? **A:** **1M** tokens on current models (default, no beta header; **200K** on older), **128K** max output.
- Q: Input alone exceeds the window — what happens? **A:** HTTP **400** `invalid_request_error`, *"prompt is too long"* (not a `stop_reason`).
- Q: Which token type does **not** count toward ITPM? **A:** **`cache_read_input_tokens`** — cache reads are exempt (ITPM = `input_tokens` + `cache_creation_input_tokens`).
- Q: Name the rate-limit response headers. **A:** `anthropic-ratelimit-{requests,tokens,input-tokens,output-tokens}-{limit,remaining,reset}` + `retry-after`.
- Q: `service_tier` values? **A:** `auto` (default) or `standard_only`; Priority Tier = dedicated capacity.
- Q: Minimum fields for a custom tool? **A:** `name` (`^[a-zA-Z0-9_-]{1,128}$`), `description`, `input_schema` (JSON Schema). Optional: `cache_control`, `strict`, `defer_loading`, `input_examples`, `allowed_callers`.
- Q: How does an Anthropic-defined tool appear in `tools`? **A:** A **dated `type` string** (e.g. `web_search_20260318`, `bash_20250124`) + `name`, with **no `input_schema`**.
- Q: On the newest models, which `tool_choice` values return 400? **A:** **`any` and `tool`** — use `auto` + `strict`, or structured outputs; `auto`/`none` still work.
- Q: Can the MCP connector reach a local stdio server? **A:** **No** — `mcp_servers` is **remote HTTPS only** (Streamable HTTP / SSE); needs the `mcp-client-2025-11-20` (or `2026-09-15`) beta header.
- Q: What does `context_management` do, and name an edit type? **A:** Auto-prunes old context to protect the window — e.g. **`clear_tool_uses_20250919`** / **`clear_thinking_20251015`**; beta header `context-management-2025-06-27`.

**One-line summary:** *`POST /v1/messages` with **`x-api-key`** + **`anthropic-version`**; **`max_tokens` required**; two roles + a separate top-level **`system`**; **stateless → resend history**; **seven `stop_reason`s** drive the loop; **streaming is SSE** (`content_block_delta` = text, `message_delta` = final stop_reason/usage); tools request via **`tool_use`** and you reply with **`tool_result`** in a user message; images/PDFs/Files add multimodal input; caching, batches, adaptive thinking, structured outputs and citations are the advanced surface — and the newest API version drops the sampling params.*