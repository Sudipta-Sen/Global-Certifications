# CCDV-F Revision Note — Domain 2: Applications & Integration

> **Exam weight: ~33.1%** — by far the biggest domain (roughly **1 question in 3**). This is the Messages API: how you talk to Claude over HTTP. Know the headers, the required params, the response fields, the seven `stop_reason`s, statelessness, streaming, and the multimodal/file surface **cold**. This is where the exam is won or lost.

---

## Contents
1. [The Messages API foundation](#1-the-messages-api-foundation)
   - [1.1 Endpoint, headers & versioning](#11-endpoint-headers--versioning)
   - [1.2 Required & optional parameters](#12-required--optional-parameters)
   - [1.3 The request & response shape](#13-the-request--response-shape)
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
5. [Multimodal & file input](#5-multimodal--file-input)
   - [5.1 Vision (images)](#51-vision-images)
   - [5.2 PDFs & documents](#52-pdfs--documents)
   - [5.3 The Files API](#53-the-files-api)
6. [Advanced API surface](#6-advanced-api-surface)
   - [6.1 Prompt caching](#61-prompt-caching)
   - [6.2 Message Batches](#62-message-batches)
   - [6.3 Thinking (adaptive vs manual)](#63-thinking-adaptive-vs-manual)
   - [6.4 Structured outputs & citations](#64-structured-outputs--citations)
7. [Errors & retries](#7-errors--retries)
8. [SDKs & deployment platforms](#8-sdks--deployment-platforms)
9. [Common traps](#9-common-traps)
10. [Quick-fire revision](#10-quick-fire-revision)

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
| `service_tier` | priority/throughput tier for the request. |
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
- **`usage`** reports `input_tokens` and `output_tokens`, plus `cache_creation_input_tokens` / `cache_read_input_tokens` when caching is on (§6.1).
- Every response also carries a **`request-id`** header (e.g. `req_018Ee…`) — quote it to support when reporting a problem.

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
- An **empty `end_turn`** can occur right after tool results — avoid it by **never putting a text block immediately after `tool_result` blocks**. Retrying the empty response unchanged won't help.
- `refusal`'s `stop_details` is populated only for refusals (it's `null` for every other stop reason).
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
- **`content_block_delta`** carries the streamed content: **`text_delta`** (text), **`input_json_delta`** (partial tool-arg JSON — accumulate and parse at `content_block_stop`), **`thinking_delta`** (reasoning), with a **`signature_delta`** just before a thinking block closes.
- **`message_delta`** carries the **final `stop_reason`** and the **cumulative `usage`** — this is where you read how the turn ended.
- **Benefit:** better **perceived latency** — tokens appear as generated. **Total generation time and cost are unchanged.**
- Clients should ignore unknown event types gracefully (Anthropic may add more).

> **Trap:** streaming is **SSE**, not WebSockets; the **text** is in `content_block_delta`, the **final `stop_reason`/`usage`** is in `message_delta`.

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
- Every response carries a **`request-id`** header (and `request_id` in error bodies) — read it from `response._request_id` in the SDKs; include it in support tickets.

---

## 8. SDKs & deployment platforms
- **Seven official client SDKs** wrap the same `/v1/messages` endpoint: **Python, TypeScript, C#/.NET, Go, Java, PHP, Ruby.** They all handle **auth, streaming, automatic retries/backoff, error handling, typed responses, and OpenTelemetry tracing** — anything an SDK does, you *could* do with raw HTTP.
- They set `anthropic-version` for you, so you rarely type the header yourself.
- **Deployment platforms:** the same Messages API shape is available on **Amazon Bedrock / Claude Platform on AWS**, **Google Cloud / Vertex AI** (Anthropic-managed and Google-managed), and **Microsoft Foundry** — with provider-specific auth, and some source-type limits (e.g. Bedrock/Google-managed accept **base64 only** for images/PDFs; Foundry doesn't support the Files API on Azure-hosted deployments).

---

## 9. Common traps
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

---

## 10. Quick-fire revision
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

**One-line summary:** *`POST /v1/messages` with **`x-api-key`** + **`anthropic-version`**; **`max_tokens` required**; two roles + a separate top-level **`system`**; **stateless → resend history**; **seven `stop_reason`s** drive the loop; **streaming is SSE** (`content_block_delta` = text, `message_delta` = final stop_reason/usage); tools request via **`tool_use`** and you reply with **`tool_result`** in a user message; images/PDFs/Files add multimodal input; caching, batches, adaptive thinking, structured outputs and citations are the advanced surface — and the newest API version drops the sampling params.*