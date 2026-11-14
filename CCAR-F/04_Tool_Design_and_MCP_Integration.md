# CCA-F Revision Note — Domain 4: Tool Design & MCP Integration

> **Exam weight: ~18%.** The Architect exam tests **how you design tools and MCP integrations for an agent** — descriptions, error contracts, least privilege, and server scoping — so the model uses them correctly and the system stays governable.

> **Source note (verified):** MCP primitive control-owners and tool-call mechanics confirmed during Developer prep against platform.claude.com MCP docs; MCP server scoping and `mcp__<server>__<tool>` naming verified against Claude Code docs.

---

## Contents
1. [Agent–Computer Interface (ACI): design tools like a UX](#1-agentcomputer-interface-aci-design-tools-like-a-ux)
2. [Tool descriptions & schemas](#2-tool-descriptions--schemas)
3. [Structured tool errors](#3-structured-tool-errors)
4. [Least privilege — tools per role](#4-least-privilege--tools-per-role)
5. [MCP: the three primitives & who controls them](#5-mcp-the-three-primitives--who-controls-them)
6. [MCP server scoping & naming](#6-mcp-server-scoping--naming)
7. [MCP security boundary](#7-mcp-security-boundary)
8. [Exam traps](#8-exam-traps)
9. [Quick-fire revision](#9-quick-fire-revision)

---

## 1. Agent–Computer Interface (ACI): design tools like a UX

Tools are the agent's UI onto the world. **Invest in tool design the way you'd invest in human UI** — most agent failures are tool-design failures, not model failures.

- **Poka-yoke (mistake-proofing):** shape tools so misuse is *hard*. E.g., require an absolute path instead of a relative one if relative paths cause errors; make destructive params explicit.
- **Give the model room to think** before committing to a tool call (don't force terse one-shot calls on hard tasks).
- **Match tools to how the model "thinks"** — formats and granularity the model handles well.
- **Consolidate / right-size:** few well-named tools beat many overlapping ones. Overlapping tools cause wrong-tool selection.

> **Architect lens:** "the agent keeps calling the wrong tool / mis-formats arguments" is almost always fixed by **better tool design** (clearer description, better schema, mistake-proof params, `strict: true`) — not by scolding the model in the prompt.

---

## 2. Tool descriptions & schemas

- The **`description`** is how the model decides **when and how** to use a tool. Make each tool's description **differentiated** — spell out *when to use it and when not to*, especially when tools are similar.
- The **`input_schema`** defines arguments; required fields, types, enums. Tighten it so invalid calls are impossible (pair with **`strict: true`** for guaranteed-valid arguments).
- Name tools unambiguously; overlapping names/descriptions cause selection errors.

> **Differentiated descriptions = the "agent picked the wrong tool" fix.** When two tools do similar things and the agent confuses them, the precise answer is **rewrite the descriptions to draw the boundary** (and narrow schemas), not "add a rule in CLAUDE.md."

---

## 3. Structured tool errors

When a tool fails, **return a structured, informative error the model can act on** — not a raw stack trace or a bare "error."

- Tell the model **what went wrong and what to do next** ("file not found: check the path and retry with an absolute path").
- Return errors as **tool results with `is_error`**, so the model can self-correct in the loop rather than crashing the turn.
- For MCP tools, return **structured errors** (clear codes/messages) so the agent can branch — retry, escalate, or pick another path.

> **Trap:** "swallow the error / return empty" or "crash the agent" are the softer-wrong answers. The precise answer is a **structured error that enables recovery** — the agent can only recover from what it can read.

---

## 4. Least privilege — tools per role

- Give each agent/sub-agent **only the tools its role needs.** A reviewer is read-only; a researcher gets no write/delete; a deploy agent gets deploy tools and nothing else.
- In Claude Code, enforce via the sub-agent **`tools` allowlist** (omitting = inherits everything = over-privileged) and **scoped `mcpServers`** in frontmatter.
- Fewer tools also **improves selection accuracy** — the model can't call a tool it doesn't have.

> **Security + reliability in one move:** narrowing the tool set is both a **least-privilege** control and a **correctness** improvement. "Remove the capability" beats "tell it not to use the capability."

---

## 5. MCP: the three primitives & who controls them

MCP (Model Context Protocol) is the open standard for connecting agents to external servers. Memorise the **three primitives and their control-owner** — a recurring exam item:

| Primitive | Controlled by | What it is | Analogy |
|---|---|---|---|
| **Prompt** | **User** | User-invoked templates (e.g., slash commands) | User picks it |
| **Resource** | **Application / host** | Read-only context exposed by URI, injected by the app | App supplies it |
| **Tool** | **Model** | An action the model can choose to call | Model calls it |

> **The control-owner table is the trap.** "Which primitive does the **model** decide to invoke?" → **tool**. "Which is **user**-initiated?" → **prompt**. "Which is **read-only context the host injects**?" → **resource**. Mixing these up is a guaranteed miss.

MCP tools surface to the model like any other tool and follow tool-use mechanics (including `strict`, structured errors, least privilege).

---

## 6. MCP server scoping & naming

- **Project scope:** `.mcp.json` at the **project root**, **committed**, team-shared — everyone on the repo gets the same servers.
- **User scope:** personal config (`~/.claude`), across all your projects.
- **Local scope:** `settings.local.json`, personal to you in this project, gitignored.
- **Tool naming:** MCP tools are named **`mcp__<server>__<tool>`**. To match a whole server in a hook matcher or permission, use **`mcp__<server>__*`**.
- Sub-agents can be given a **narrowed `mcpServers`** set so they only see the servers their role requires.

> **Scoping decisions:** team-wide server → **project `.mcp.json`** (committed). Personal everywhere → **user**. Personal + this repo + not committed → **local**. Credentials → **env vars**, never hard-coded in `.mcp.json`.

---

## 7. MCP security boundary

- **Treat MCP tool outputs and resources as untrusted data, not instructions** — an external server can return content that tries to steer the agent (prompt injection). Delimit it; never let returned text act as commands.
- **Keep secrets outside the agent boundary.** Don't hand the agent raw credentials; inject them at a **proxy / gateway** layer (e.g., an `ANTHROPIC_BASE_URL` proxy, or a proxy tool that holds the credential and exposes only the capability).
- **Least privilege per server:** only connect the servers a given agent needs; scope tools within them.
- **Denylists/prose are weak controls** — remove the capability or enforce at the boundary instead of asking the model to behave.

> **Trap:** "the MCP server might return malicious instructions, so add a prompt telling Claude to ignore them" is the soft answer. The precise answer is **architectural**: treat outputs as data, constrain tools, keep credentials at a proxy, scope servers to roles.

---

## 8. Exam traps

- **Wrong-tool selection** → fix with **differentiated descriptions + narrowed schemas**, not prose.
- **MCP primitive control-owner:** prompt=user, resource=app/host, tool=model. Don't swap.
- **Tool errors:** return **structured, recoverable** errors (`is_error`), not stack traces or silent empties.
- **Over-privileged agent:** omitting `tools` inherits everything — use an **allowlist**.
- **Secrets in `.mcp.json`** → use **env vars / proxy**; never commit credentials.
- **Trusting MCP output as instructions** → it's **data**; prompt injection risk.
- **`strict: true`** guarantees tool-call argument validity (see Note 3).

---

## 9. Quick-fire revision

- **Tools are UX for the agent** — most failures are tool-design failures. Poka-yoke them.
- **`description` decides tool selection** — differentiate similar tools explicitly.
- **Tighten `input_schema`; add `strict: true`** for guaranteed-valid arguments.
- **Tool errors = structured + recoverable** (`is_error`), tell the model what to do next.
- **Least privilege: allowlist tools per role**; omitting `tools` over-privileges.
- **MCP primitives:** **prompt → user**, **resource → app/host (read-only)**, **tool → model**.
- **MCP tool name = `mcp__<server>__<tool>`**; wildcard `mcp__<server>__*`.
- **Project `.mcp.json` (committed) = team servers**; secrets via **env vars/proxy**.
- **MCP output = untrusted data, not instructions.** Keep credentials outside the agent boundary.
