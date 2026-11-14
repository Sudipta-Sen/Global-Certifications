# CCDV-F Revision Note — Domain 8: Tools & MCPs

> **Exam weight: ~10.6%.** Two halves that reward precision: **API-level tool use** (the three tool fields, the round trip, `tool_choice`, `is_error`, who executes) and **MCP concepts** (architecture, primitives, transports, and the Claude MCP connector). Get the vocabulary exact — most questions here are decided by one precise word.

---

## Contents
1. [Tool use fundamentals](#1-tool-use-fundamentals)
   - [1.1 What a tool is — the three fields](#11-what-a-tool-is--the-three-fields)
   - [1.2 The tool-use round trip](#12-the-tool-use-round-trip)
   - [1.3 `tool_choice` — the four modes](#13-tool_choice--the-four-modes)
   - [1.4 Parallel tool use](#14-parallel-tool-use)
2. [Where tools run & how they fail](#2-where-tools-run--how-they-fail)
   - [2.1 Client tools vs server tools](#21-client-tools-vs-server-tools)
   - [2.2 Anthropic-defined client tools](#22-anthropic-defined-client-tools)
   - [2.3 `is_error` & recovery](#23-is_error--recovery)
   - [2.4 Configuring server tools (web_search / web_fetch)](#24-configuring-server-tools-web_search--web_fetch)
   - [2.5 Strict tool use (`strict: true`)](#25-strict-tool-use-strict-true)
3. [MCP — the Model Context Protocol](#3-mcp--the-model-context-protocol)
   - [3.1 What it is & why it exists](#31-what-it-is--why-it-exists)
   - [3.2 Architecture: host / client / server](#32-architecture-host--client--server)
   - [3.3 The two layers: data (JSON-RPC) & transport](#33-the-two-layers-data-json-rpc--transport)
   - [3.4 Primitives (server & client)](#34-primitives-server--client)
4. [MCP in the Claude platform](#4-mcp-in-the-claude-platform)
   - [4.1 The Messages API MCP connector](#41-the-messages-api-mcp-connector)
   - [4.2 MCP in Claude Code & Desktop](#42-mcp-in-claude-code--desktop)
   - [4.3 Security](#43-security)
5. [Common traps](#5-common-traps)
6. [Quick-fire revision](#6-quick-fire-revision)

---

## 1. Tool use fundamentals

### 1.1 What a tool is — the three fields
A **tool** lets Claude reach out to *your* code or an external system — fetch weather, query a database, run a script. The one principle under everything in this domain: **Claude never executes the tool. It *requests* a call; your application runs it and hands back the result.** Claude is the brain deciding *what* to call; your app is the hands that actually do it.

You describe a tool with **exactly three fields**:

| Field | What it is |
|---|---|
| **`name`** | unique identifier (e.g. `get_weather`). |
| **`description`** | a clear, detailed explanation of **what it does and when to use it**. This is the **single biggest driver of reliable tool use** — Claude picks tools "based on the user's request and the tool's description." Write it like a docstring for a new teammate (ties to the ACI guidance in Domain 1). |
| **`input_schema`** | a **JSON Schema** of the parameters (types, `required`, enums). |

```json
{
  "name": "get_weather",
  "description": "Get the current weather for a location. Use whenever the user asks about weather.",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": { "type": "string", "description": "City and state, e.g. San Francisco, CA" }
    },
    "required": ["location"]
  }
}
```

> **Exam trap:** the three fields are **`name`, `description`, `input_schema`** — there is **no `output` / `returns` field** (Claude learns the result shape from the `tool_result` you send back). And a *great description* beats clever prompting for reliable calls.

### 1.2 The tool-use round trip
The flow is a loop, and the key beats are: Claude emits a **`tool_use`** block and stops with `stop_reason: "tool_use"`; you run the tool; you reply with a **`tool_result`** inside a **`user`** message:

```
1. You  →  send messages + tools (+ optional tool_choice, max_tokens)
2. Claude → assistant turn with a tool request:
     { "type":"tool_use", "id":"toolu_01A…", "name":"get_weather",
       "input":{ "location":"New York, NY" } }
     stop_reason == "tool_use"
3. Your code → runs get_weather("New York, NY")
4. You  →  send a USER message carrying the result:
     { "type":"tool_result", "tool_use_id":"toolu_01A…",
       "content":"18°C, sunny", "is_error": false }
5. Claude → final answer (stop_reason == "end_turn")
```
- The `tool_use` block carries **`id`, `name`, `input`**; `stop_reason: "tool_use"` is your signal to act.
- **`tool_use_id` must equal the `tool_use.id`** — that id, not the order, is how a result is matched to its call.
- The **`tool_result` lives in a `user` message** — never assistant, never system.
- Append **both** the assistant `tool_use` turn **and** the user `tool_result` turn before the next request — the API is **stateless** (Domain 2), so the whole history goes every time.

### 1.3 `tool_choice` — the four modes
`tool_choice` controls whether Claude *may*, *must*, or *must not* call a tool this turn:

| `tool_choice` | Behaviour |
|---|---|
| `{"type":"auto"}` | **Default when tools are present.** Claude decides each turn whether to call a tool or answer in text. |
| `{"type":"any"}` | Claude **must** use **some** tool (its pick) — never a plain text reply. |
| `{"type":"tool","name":"X"}` | Claude **must** use the **named** tool X. |
| `{"type":"none"}` | Claude **cannot** use any tool this turn. |

- Add **`disable_parallel_tool_use: true`** to force **at most one** call per turn, e.g. `{"type":"auto","disable_parallel_tool_use":true}`.
- Minor cost note: `auto`/`none` inject a smaller tool system prompt; `any`/`tool` add stricter forcing instructions (slightly more tokens).

> **Trap:** **`any`** = must use *some* tool; **`tool`** = must use *that named* tool — don't swap them. `auto` is the default **only when tools are supplied**.

### 1.4 Parallel tool use
In a single turn Claude can emit **several `tool_use` blocks at once**. You run them (often concurrently) and return **all** the `tool_result` blocks in **one** following `user` message, each matched by its `tool_use_id`:

```jsonc
// assistant (stop_reason: "tool_use") — two calls at once
"content": [
  {"type":"tool_use","id":"toolu_01","name":"get_weather","input":{"city":"Paris"}},
  {"type":"tool_use","id":"toolu_02","name":"get_time","input":{"city":"Paris"}}
]
// YOUR next user message — BOTH results together
"content": [
  {"type":"tool_result","tool_use_id":"toolu_01","content":"18°C, clear"},
  {"type":"tool_result","tool_use_id":"toolu_02","content":"14:05 CET"}
]
```
- Return **every** result before the next turn; a missing one errors.
- Turn it off with **`disable_parallel_tool_use: true`** when order or side-effects demand one-at-a-time.

---

## 2. Where tools run & how they fail

### 2.1 Client tools vs server tools
The big dividing line is **who actually executes the tool** — you, or Anthropic. It changes the whole flow.

| | **Client tools** | **Server tools** |
|---|---|---|
| Runs where? | **Your application** | **Anthropic's infrastructure** |
| Examples (verified) | your custom tools; Anthropic-schema tools like `bash`, `text_editor`, `computer` | `web_search`, `web_fetch`, `code_execution`, `tool_search` |
| Flow | `stop_reason:"tool_use"` → **you** run it → you send `tool_result` | Anthropic runs it; **you get the result directly** — no `tool_result` round trip |
| Can produce `pause_turn`? | — | **Yes** — a long server-tool loop can hit its iteration cap (default 10) and pause (Domain 2 §3.1) |

- Server tools are declared by a **versioned `type`**, e.g. `{"type":"web_search_20260209","name":"web_search"}`.
- Edge case (verified): a server tool runs on Anthropic's side **"unless Claude calls [it] in the same group of parallel tool calls as one of your client tools"** — then it's handled with your client tools.
- **MCP tools** (see §3) also reach Claude through this same tool-use mechanism — MCP is a *standard way to supply* tools, not a different calling convention.

### 2.2 Anthropic-defined client tools
A subtle, testable middle category: tools where **Anthropic publishes the schema and trains Claude on it, but your application still runs each call** and returns the `tool_result`. They are **client** tools despite being "Anthropic-defined":

- **Bash** — run shell commands.
- **Text editor** — view/edit files.
- **Computer use** — "take screenshots and control the mouse and keyboard in a desktop environment."
- **Browser use** — drive a browser.
- **Memory** — persist notes/state across turns.

> **Trap:** "Anthropic-defined" ≠ "Anthropic-run." `bash`/`text_editor`/`computer` have **Anthropic's schema** but **execute in your app** (client tools). Only `web_search`/`web_fetch`/`code_execution`/`tool_search` run on **Anthropic's infrastructure** (server tools).

### 2.3 `is_error` & recovery
When a tool call fails, don't fake a success — tell Claude. Set **`is_error: true`** on the `tool_result` (with the error message as `content`) so Claude can **recover**: retry, fix the arguments, or explain to the user.

```json
{ "type":"tool_result", "tool_use_id":"toolu_01A…",
  "content":"Error: city 'Atlantis' not found. Ask the user to clarify.",
  "is_error": true }
```
- Return **structured error info** (what failed + why) so the model can self-correct (ties to the failure-handling guidance in Domain 1).
- MCP tool results carry the **same idea** (`isError` / `is_error`), so recovery works the same across API tools and MCP tools.

### 2.4 Configuring server tools (web_search / web_fetch)
Server tools take **configuration fields on the tool entry** — a favourite *"how do you restrict / limit it?"* scenario. For **`web_search`** (verified):

```jsonc
{
  "type": "web_search_20250305",          // dated type; newer: _20260209, _20260318
  "name": "web_search",
  "max_uses": 5,                          // cap searches per turn (failed ones count; over → max_uses_exceeded)
  "allowed_domains": ["example.com"],     // ONLY these domains …
  // "blocked_domains": ["spam.com"],     // … OR never these — not both (both → 400)
  "user_location": {                      // localise results
    "type": "approximate",               // must be "approximate"
    "city": "San Francisco", "region": "California",
    "country": "US",                      // ISO 3166-1 alpha-2
    "timezone": "America/Los_Angeles"     // IANA tz
  }
}
```
- **Limit how many searches** per turn → **`max_uses`** (exceeding returns a `web_search_tool_result` error `max_uses_exceeded`).
- **Restrict to a domain allowlist** → **`allowed_domains`**; **exclude** domains → **`blocked_domains`**. Use **one or the other, never both** (both → **400**). Domains are **bare** (`example.com` or `example.com/blog`), **no scheme**.
- **Localise results** → **`user_location`** (`approximate` + any of city/region/country/timezone).
- **`web_fetch`** takes a similar **`max_uses`** cap. The dated **`type`** string advances with capability (`web_search_20250305` basic → `_20260209` dynamic filtering → `_20260318` adds `response_inclusion`).

> **Trap:** to keep a research agent on trusted sources, the answer is **`allowed_domains`** on the `web_search` tool — not a prompt instruction ("only use these sites"), which the model can ignore. Same spirit as least privilege (Domain 7).

### 2.5 Strict tool use (`strict: true`)
Set **`strict: true`** on a tool definition to **guarantee** Claude's **tool-call arguments** match your `input_schema` — it constrains token sampling to schema-valid output (**grammar-constrained sampling**). Without it, the schema is only *guidance*: Claude usually follows it but can send the **wrong type** (`"2"` or `"two"` instead of `2`), **omit a required field**, or add an extra one — breaking your function.

```python
tools=[{
  "name": "get_weather",
  "description": "Get the current weather in a location",
  "strict": True,                          # ← grammar-constrained, schema-valid inputs
  "input_schema": {
    "type": "object",
    "properties": {"location": {"type": "string"}},
    "required": ["location"],
    "additionalProperties": False          # strict schemas pin this down
  }
}]
```

- **Guarantees:** `tool_use.input` strictly follows your schema, **and** the tool `name` is always a real tool (never hallucinated).
- **≠ `output_config.format`:** `strict` constrains **tool-call inputs**; `output_config.format` constrains the **response text** (Domain 6 §2.7). Same grammar engine, different target.
- **≠ `tool_choice`:** `tool_choice` forces **whether / which** tool is called; `strict` governs **whether the arguments are valid**. (A prompt instruction is only a nudge — not a guarantee of either.)
- **Caveats (verified):** only a **subset of JSON Schema** is allowed; the **`computer` / `browser` toolsets reject `strict: true`** (→ 400); compiled schemas are **cached up to ~24 h**, so **keep PHI out of the schema** (property names, `enum`/`const`, `pattern`) — put it only in message content.
- **Supported** across current models (4.5 / 4.6 / 5.x), **stable, no beta header**.

---

## 3. MCP — the Model Context Protocol

### 3.1 What it is & why it exists
**MCP is an open standard for connecting LLM applications to external tools, data, and prompts** — often called *"a USB-C port for AI."* The point is **decoupling**: you write an integration **once as an MCP server**, and any MCP-compatible app (Claude Desktop, Claude Code, IDEs, other hosts) can use it — no bespoke glue per app.

Think of the old way as every app needing a custom cable for every device; MCP is the single universal port. It's become the standard way to give agents tools and context.

### 3.2 Architecture: host / client / server
Three roles, and the relationship between client and server is the detail that gets tested:

- **Host** — the AI application the user runs (Claude Desktop, Claude Code, an IDE). It **coordinates one or more clients**.
- **MCP client** — lives **inside the host**; maintains a **dedicated 1:1 connection** to exactly one server, and fetches context from it for the host.
- **MCP server** — a separate program that **exposes capabilities** (tools/resources/prompts). "Server" refers to the *program that serves context*, **regardless of where it runs** (local or remote).

```
┌───────────── MCP Host (e.g. Claude Code) ─────────────┐
│   MCP Client 1 ───dedicated conn───►  Server A (local, Filesystem)
│   MCP Client 2 ───dedicated conn───►  Server B (local, Database)
│   MCP Client 3 ───dedicated conn───►  Server C (remote, Sentry)
└────────────────────────────────────────────────────────┘
   one host → many clients;  each client ↔ exactly one server (1:1)
```
- One host runs **many clients → many servers**, but each **client↔server link is 1:1**.
- **Local** servers (stdio transport) typically serve a **single** client; **remote** servers (Streamable HTTP) typically serve **many** clients.
- The connection opens with a **capability/version negotiation** (the client learns which primitives the server supports before using them) — in the latest spec this is the `server/discover` step; older material calls it the `initialize` handshake. (This is the real "MCP handshake" — a protocol-level step, separate from anything in the model prompt.)

### 3.3 The two layers: data (JSON-RPC) & transport
MCP is defined in two layers:

- **Data layer** — a **JSON-RPC 2.0** protocol defining the messages: capability/version discovery, the primitives (tools/resources/prompts), and notifications. This is the inner layer, identical across transports.
- **Transport layer** — *how* the bytes move:

| Transport | Use | Notes |
|---|---|---|
| **stdio** | **Local** server as a child process | Simplest; host launches the server and talks over stdin/stdout. No network overhead. |
| **Streamable HTTP** | **Remote** server over the network | The **modern** remote transport; HTTP POST + optional SSE for streaming; supports bearer tokens / API keys / OAuth. |
| *(legacy)* **HTTP + SSE** | older remote transport | Superseded by Streamable HTTP; may appear as a distractor. |

> **Trap:** the message format is always **JSON-RPC 2.0**; the two live transports are **stdio** (local) and **Streamable HTTP** (remote). Plain **HTTP+SSE is the legacy** remote transport.

### 3.4 Primitives (server & client)
**The three *server* primitives are the single most-tested MCP fact.** The memory aid is *who controls each one*:

| Primitive | What it is | Controlled by |
|---|---|---|
| **Tools** | callable **functions/actions** (do something, may have side effects) | **Model** — Claude decides to call them |
| **Resources** | readable **data/context** (files, records, documents) | **Application** — the host decides what to attach |
| **Prompts** | reusable **prompt templates / workflows** (often slash commands) | **User** — the user picks/invokes them |

- Each primitive has methods for discovery (`*/list`), retrieval (`*/get`), and — for tools — execution (`tools/call`). Listings can change dynamically, and servers can push `list_changed` **notifications**.
- The exam answer for "the three MCP server primitives" is **Tools, Resources, Prompts** — *not* "models / weights / agents."

**Client-side primitives (the reverse direction — the client exposes these back to servers):**
- **Elicitation** (current) — a server can **ask the user for input or confirmation** mid-task (`elicitation/create`).
- **Sampling** — a server borrows the client's LLM to get a completion (stays model-independent). **Deprecated as of MCP spec `2026-07-28`** — new code should call an LLM provider directly.
- **Logging** — server→client log messages. Also **deprecated** in favour of `stderr`/OpenTelemetry.

---

## 4. MCP in the Claude platform

### 4.1 The Messages API MCP connector
You can attach **remote** MCP servers directly to a Messages API call — no client code of your own — via the **MCP connector** (a **beta** feature; header `mcp-client-2026-09-15`, which supersedes `mcp-client-2025-11-20`). You supply two matching pieces: an **`mcp_servers`** array (the connections) and an **`mcp_toolset`** entry in `tools`:

```python
response = client.beta.messages.create(
    model="claude-opus-5-5", max_tokens=1000,
    messages=[{"role": "user", "content": "What tools do you have?"}],
    mcp_servers=[{
        "type": "url",                       # only "url" is supported
        "url": "https://example.modelcontextprotocol.io/sse",  # must be https://
        "name": "example-mcp",
        "authorization_token": "YOUR_TOKEN", # OAuth bearer token (you obtain/refresh it)
    }],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "example-mcp"}],
    betas=["mcp-client-2026-09-15"],
)
```
- **Remote only:** the server must be publicly reachable over HTTP (**Streamable HTTP or SSE**). **Local stdio servers cannot be connected** through the connector.
- **Auth:** *you* run the OAuth flow and pass the bearer token as `authorization_token`; the connector doesn't do OAuth for you.
- **Only tool calls are supported** over the connector — not resources or prompts.
- Responses carry **`mcp_tool_use`** (with a `server_name`) and **`mcp_tool_result`** (with `is_error` + `content`) blocks. With the newest header a response may start with an `mcp_tool_listing` block — skip it if you read `content[0]`, and send the assistant message back unchanged.
- **Not covered by ZDR** — data exchanged with MCP servers follows standard retention.

### 4.2 MCP in Claude Code & Desktop
When a host like **Claude Code** or **Claude Desktop** connects to an MCP server, the primitives surface in the UI (ties to Domain 3):
- **MCP tools** appear as callable tools.
- **MCP prompts** surface as **slash commands**.
- **MCP resources** become context the host can attach.
Remote MCP servers are also how shareable **connectors** are built once and reused across hosts.

### 4.3 Security
A connected server can expose powerful actions and returns **untrusted content** — treat it carefully (ties to Domain 7):
- **MCP tool output is untrusted** — a prime vector for **indirect prompt injection**. Treat it as data, screen it, and don't let it silently drive actions.
- **Approve / allow-list** servers; scope their credentials **narrowly** (least privilege); **sandbox** local (`stdio`) servers.
- Reserve **human-in-the-loop** approval for sensitive or irreversible actions a server can perform.
- **Never accept a server URL or token from an untrusted source**, and remember the connector's data isn't ZDR-covered.

---

## 5. Common traps
- Claude **requests**; it does **not execute** tools — **your app** runs client tools, **Anthropic's infra** runs server tools.
- Three required tool fields: **`name`, `description`, `input_schema`** (JSON Schema). **No `output` field.**
- Great **descriptions** matter more than clever prompting.
- **`tool_result` goes in a `user` message**; `tool_use_id` must match the `tool_use.id`; all parallel results go in **one** user message.
- **"Anthropic-defined" ≠ "Anthropic-run":** `bash`/`text_editor`/`computer` are **client** tools (you execute); `web_search`/`code_execution`/etc. are **server** tools (Anthropic executes, no round trip, can `pause_turn`).
- `tool_choice`: **`any`** = some tool; **`tool`** = that named tool; **`auto`** = default when tools present; **`none`** = no tools.
- Flag a failed call with **`is_error: true`** — don't return a bogus success.
- MCP server primitives = **Tools, Resources, Prompts** (not models/weights/agents). Control: tools=**model**, resources=**app**, prompts=**user**.
- MCP architecture = **Host → (MCP) Client → (MCP) Server**, with each **client↔server 1:1**; messages are **JSON-RPC 2.0**.
- Transports: **stdio** (local) + **Streamable HTTP** (modern remote); **HTTP+SSE** is legacy. The **API MCP connector is remote-only** (no stdio).
- Client primitives: **elicitation** (current); **sampling/logging deprecated** as of spec 2026-07-28.

---

## 6. Quick-fire revision
- Q: Three required tool-definition fields? **A:** `name`, `description`, `input_schema` (no `output`).
- Q: Who executes a client tool? **A:** **Your application** — Claude only *requests*.
- Q: What `stop_reason` signals a tool request? **A:** `tool_use`.
- Q: Where does a `tool_result` go, and how is it matched? **A:** In a **`user`** message; matched by **`tool_use_id` == `tool_use.id`**.
- Q: Force a specific tool? **A:** `tool_choice:{"type":"tool","name":"…"}`.
- Q: Force *some* tool? / Default with tools present? / No tools? **A:** `{"type":"any"}` / `auto` / `{"type":"none"}`.
- Q: Allow only one tool call per turn? **A:** `disable_parallel_tool_use: true`.
- Q: Return two parallel results — how many user messages? **A:** **One**, with both `tool_result` blocks.
- Q: Signal a failed tool call? **A:** `is_error: true` in the `tool_result`.
- Q: Client vs server tools? **A:** Client run in **your app** (round-trip `tool_result`); server tools (`web_search`, `web_fetch`, `code_execution`, `tool_search`) run on **Anthropic's infra** (results returned directly, can `pause_turn`).
- Q: Are `bash`/`text_editor`/`computer` server tools? **A:** **No** — Anthropic-defined **client** tools; your app executes them.
- Q: Guarantee a tool is called with schema-valid, correctly-typed arguments? **A:** **`strict: true`** on the tool (grammar-constrained). Contrast: **`tool_choice`** forces *whether/which* tool; **`output_config.format`** constrains the *response*, not tool inputs.
- Q: What does MCP stand for / what is it? **A:** **Model Context Protocol** — an open standard ("USB-C for AI") connecting apps to tools/data/prompts.
- Q: MCP architecture pieces & the client↔server relationship? **A:** **Host → Client → Server**; one host, many clients→servers; each **client↔server is 1:1**.
- Q: What RPC format does MCP use? **A:** **JSON-RPC 2.0**.
- Q: Three MCP **server** primitives + who controls each? **A:** **Tools** (model), **Resources** (app), **Prompts** (user).
- Q: Name a **client** primitive, and one that's deprecated. **A:** **Elicitation** (current); **sampling** (and logging) **deprecated** as of spec 2026-07-28.
- Q: Map these to MCP primitives: a slash-command "draft postmortem" workflow; a host-attached read-only incident timeline; a model-decided "page the team" action. **A:** **prompt** (user-controlled, slash command), **resource** (app-controlled context by URI), **tool** (model-controlled action). Mnemonic: **prompt = user, resource = app, tool = model.**
- Q: A plugin depends on another by name (`"dependencies": ["x"]`) and a breaking **x 3.0.0** broke everyone — keep users on 2.x fixes only? **A:** Pin a **semver range**: `{"name":"x","version":"^2.0"}`; Claude Code resolves it against git tags **`<name>--v<version>`**. (Bumping your *own* version, disabling auto-update by hand, or vendoring the server in are the traps.)
- Q: Two live MCP transports + the legacy one? **A:** **stdio** (local) & **Streamable HTTP** (remote); **HTTP+SSE** is legacy.
- Q: How do you attach a remote MCP server to a Messages API call? **A:** The **MCP connector** (beta): `mcp_servers` (url + `authorization_token`) + an `mcp_toolset` in `tools`.
- Q: Can the API MCP connector use a local stdio server? **A:** **No** — remote HTTP only; and **only tool calls** are supported.
- Q: In Claude Code, how do MCP prompts appear? **A:** As **slash commands** (tools appear as callable tools).
- Q: Is MCP tool output trusted? **A:** **No** — treat it as untrusted (indirect-injection risk); screen + least privilege + human-in-the-loop for sensitive actions.
- Q: Restrict `web_search` to trusted sites / limit the number of searches? **A:** **`allowed_domains`** (or `blocked_domains` — not both) and **`max_uses`** on the tool entry; `user_location` localises results. A prompt instruction is not a real restriction.
- Q: Why use MCP at all? **A:** Build an integration **once** as a server; reuse across any MCP client — no per-app glue.

**One-line summary:** *Tools = `name` + `description` + `input_schema`; Claude **requests** (`stop_reason:"tool_use"`), **you execute** client tools and return a `tool_result` in a **user** message (match `tool_use_id`, flag failures with `is_error`) — while **server tools** run on Anthropic's infra; `tool_choice` = auto/any/tool/none. **MCP** is the open standard (JSON-RPC 2.0) with a **Host→Client→Server** (1:1) architecture exposing **Tools/Resources/Prompts** over **stdio** or **Streamable HTTP**, reachable from the API via the remote-only **MCP connector**.*