# CCDV-F Revision Note — Domain 1: Agents & Workflows

> **Exam weight: ~14.7%** (top-3 heaviest). Applied/scenario questions: *"which pattern fits?"* and *"what's the fix?"* — not just definitions. Grounded in Anthropic's **"Building Effective Agents."**

> **Source note (verified):** the five patterns, the workflow/agent split, the augmented-LLM block, and the ACI/poka-yoke guidance are all confirmed against *Building Effective Agents* (originally Dec 2024). Anthropic flags that the **tooling** in that piece has since moved on — current build guidance lives in the **Managed Agents** docs — but the **conceptual patterns below are unchanged**, which is what the exam tests.

---

## Contents
1. [Foundations](#1-foundations)
   - [1.1 The core mental model](#11-the-core-mental-model)
   - [1.2 The augmented LLM (the building block)](#12-the-augmented-llm-the-building-block)
   - [1.3 Workflow vs Agent (memorise)](#13-workflow-vs-agent-memorise)
2. [The five workflow patterns](#2-the-five-workflow-patterns)
   - [2.1 Prompt chaining](#21-prompt-chaining)
   - [2.2 Routing](#22-routing)
   - [2.3 Parallelisation (sectioning / voting)](#23-parallelisation-sectioning--voting)
   - [2.4 Orchestrator–workers](#24-orchestratorworkers)
   - [2.5 Evaluator–optimiser](#25-evaluatoroptimiser)
3. [Agents](#3-agents)
   - [3.1 Agents & the agentic loop](#31-agents--the-agentic-loop)
   - [3.2 Multi-agent / orchestrator design](#32-multi-agent--orchestrator-design)
4. [Building robust agentic systems](#4-building-robust-agentic-systems)
   - [4.1 Tool & environment design (ACI)](#41-tool--environment-design-aci)
   - [4.2 Guardrails, budgets & robustness](#42-guardrails-budgets--robustness)
5. [Pitfalls](#5-pitfalls)
   - [5.1 Anti-patterns](#51-anti-patterns)
   - [5.2 Common traps](#52-common-traps)
6. [Quick-fire revision](#6-quick-fire-revision)

---

## 1. Foundations
*What this domain is really testing, the one block everything is built from, and the distinction the exam hammers.*

### 1.1 The core mental model
This domain tests whether you can compose LLM calls into **reliable systems**: when a single prompt suffices, when a fixed **workflow** fits, and when you truly need an autonomous **agent**.

> **Golden rule (Anthropic): use the simplest thing that works.** Start with a single prompt → add retrieval/examples/tools → use a **workflow** → reach for a full **agent** only when the task genuinely needs **dynamic decision-making**. Each step up adds capability **and** cost, latency, and failure surface.

Anthropic's vocabulary (know the distinction):
- **Workflows** — systems where LLMs and tools are orchestrated through **predefined code paths**.
- **Agents** — systems where the **LLM dynamically directs its own process and tool use**, keeping control over how it accomplishes the task.
- Both are **"agentic systems"**; the split is **who controls the flow** — your code (workflow) or the model (agent).

### 1.2 The augmented LLM (the building block)
Everything is built from one block: an **LLM augmented with**:
- **Retrieval** — fetch relevant external knowledge (RAG) into context.
- **Tools** — call out to code/APIs/systems (function calling).
- **Memory** — carry state/history across steps.

The model can **generate its own search queries, pick tools, and decide what to keep**. Get this block right (clear tool docs, good retrieval) before composing anything larger.

### 1.3 Workflow vs Agent (memorise)
| | **Workflow** | **Agent** |
|---|---|---|
| Control flow | **Predefined** code paths | Model **dynamically decides** its own steps |
| Predictability | High, deterministic structure | Lower — path depends on the task |
| Tool use | Tools in a **fixed** sequence | Chooses tools **and order at runtime** |
| Best for | Well-understood, **decomposable** tasks | **Open-ended** tasks where steps can't be known in advance |
| Cost/latency | Easier to **bound** | Can **balloon** if unbounded |
| Debugging | Easier (fixed path) | Harder (emergent path) |

> **Exam trap:** "agent" ≠ "anything with tools." A **fixed tool sequence is a workflow.** Agents are usually **more expensive**, not cheaper — don't pick one just because tools are involved.

---

## 2. The five workflow patterns
Anthropic names five composable workflow patterns. Know each + **when to use** + the tell-tale scenario.

### 2.1 Prompt chaining
- **Decompose a task into fixed steps**; each LLM call's output feeds the next.
- Add a programmatic **"gate" / check** between steps to catch errors early.
- **Use when:** the task **cleanly splits** into sequential subtasks (e.g. outline → draft → polish; generate → translate).
- Trades latency for **higher accuracy per step**.

### 2.2 Routing
- **Classify the input**, then send it down a **specialised path** (often a cheaper/better-suited **model per path**).
- **Use when:** inputs fall into **distinct categories** better handled separately (support tiers: simple → Haiku, complex → Opus).
- Separates concerns → each path's prompt stays focused.

### 2.3 Parallelisation (sectioning / voting)
- **Sectioning:** split work into **independent chunks** run **concurrently**, then combine (e.g. one subtask per document; guardrail check in parallel with the main answer).
- **Voting:** run the **same task multiple times** for diverse outputs / **consensus** (e.g. multiple reviewers, take majority).
- **Use when:** subtasks are independent (sectioning) or you want **confidence/coverage** (voting). Improves speed and/or quality.

### 2.4 Orchestrator–workers
- A **central LLM dynamically breaks down** the task, **delegates** to worker subagents, and **synthesises** their results.
- **Use when:** you **can't predict the subtasks in advance** (e.g. code changes spanning an unknown set of files; multi-source research).
- Difference from parallel sectioning: the subtasks are **decided at runtime by the orchestrator**, not fixed.

### 2.5 Evaluator–optimiser
- One LLM **generates**, another **evaluates/critiques**, loop until the evaluator is satisfied.
- **Use when:** you have **clear eval criteria** and iteration measurably helps (e.g. literary translation; complex search refined over rounds).

| Pattern | One-line tell | Canonical doc example |
|---|---|---|
| Prompt chaining | Fixed **sequential** steps, output→input | Generate marketing copy → translate it |
| Routing | **Classify** then specialise | Support queries: general / refund / technical → different paths |
| Parallelisation | **Concurrent** subtasks (sectioning) or **repeats** (voting) | Guardrail screen alongside the answer (sectioning); several prompts review code for vulns (voting) |
| Orchestrator–workers | Subtasks **decided at runtime**, then synthesised | Code change spanning an unknown set of files |
| Evaluator–optimiser | **Generate → critique → refine** loop | Literary translation refined over critiques |

**Choosing a pattern (decision flow):**
```
Can you write the steps as fixed code paths?
│
├─ NO  → steps only knowable at runtime? ───────────► AGENT (§3.1)
│                                                      or ORCHESTRATOR–WORKERS (runtime subtasks)
└─ YES → it's a WORKFLOW. Which shape?
         │
         ├─ one task, sequential stages ───────────► PROMPT CHAINING
         ├─ distinct input categories ─────────────► ROUTING
         ├─ independent sub-parts, run together ───► PARALLELISATION · sectioning
         ├─ want consensus / coverage on one task ─► PARALLELISATION · voting
         └─ clear eval criteria + iteration helps ─► EVALUATOR–OPTIMISER
```
> **The fork that trips people:** *orchestrator–workers* vs *parallelisation·sectioning* both fan out — but sectioning's subtasks are **fixed in your code**, while the orchestrator **decides the subtasks at runtime**. Runtime decomposition ⇒ orchestrator.

---

## 3. Agents
*When the steps can't be hardcoded: the loop itself, then coordinating several agents.*

### 3.1 Agents & the agentic loop
An **agent** plans and acts in a loop, using tools and **environment feedback**, until it decides the goal is met (or a stop condition hits). Use agents for **open-ended problems where you can't hardcode the steps** and you can trust the model's decision-making with good tools + guardrails.

**The agentic loop (know it cold):**
```
1. Send request to the model (with tools defined)
2. Model responds → check stop_reason
3. If stop_reason == "tool_use":
      - execute the requested tool(s) in YOUR code
      - append the assistant tool_use turn + a user tool_result turn
      - go to step 1 (loop)
4. If stop_reason == "end_turn":
      - done — return the final answer
   (also stop on a max-step / budget guard)
```
- **The model never runs tools itself** — it *requests*; your app executes and feeds results back (`tool_result`).
- **Loop until `end_turn`** *or* a guard trips.
- Carry the **full history forward** each iteration — the API is **stateless** (Domain 2).
- Agents work on **environment feedback** (tool results, errors) — the doc's phrasing: agents are *"typically just LLMs using tools based on environmental feedback in a loop,"* gaining **"ground truth"** (tool results, code execution) at each step.

**The loop in code (verified pattern):**
```python
messages = [{"role": "user", "content": user_query}]
for _ in range(MAX_STEPS):                      # ← budget guard (never `while True`)
    resp = client.messages.create(model="claude-opus-5-5", max_tokens=1024,
                                  messages=messages, tools=tools)
    if resp.stop_reason == "tool_use":
        results = execute_tools(resp.content)   # YOUR code runs the tool(s)
        messages.append({"role": "assistant", "content": resp.content})   # the tool_use turn
        messages.append({"role": "user", "content": results})             # tool_result blocks
        continue                                # loop again with the new ground truth
    break                                       # end_turn / refusal / max_tokens → done
```
- **One `tool_result` per `tool_use` block**, and **don't add text after** the `tool_result` blocks (it can end the turn early). Keep the **same `tools` array** every iteration.

**`stop_reason` — the 7 values (verified; the loop's switchboard):**

| Value | Meaning | Loop action |
|---|---|---|
| **`end_turn`** | Finished naturally (most common) | Return the answer |
| **`tool_use`** | Claude is calling a tool you must run | Execute → append `tool_result` → loop |
| **`max_tokens`** | Hit your `max_tokens` | If it's a truncated `tool_use`, retry higher; else continue/flag |
| **`stop_sequence`** | Emitted one of your `stop_sequences` | Read which one fired |
| **`pause_turn`** | A **server-tool** loop hit its iteration cap | Append the content back unchanged and resend |
| **`refusal`** | Claude declined | Read `stop_details`; retry on a fallback model |
| **`model_context_window_exceeded`** | Output filled the window | Treat as truncated |

> **Tip (verified):** for most tool loops the docs recommend the SDK **tool runner**, which handles execution, result formatting, and history for you — hand-rolling the loop is for when you need custom control.

### 3.2 Multi-agent / orchestrator design
- **Coordinator (orchestrator)** responsibilities: **decompose** the goal, **delegate** subtasks, **aggregate/synthesise** outputs, decide **when the goal is met**.
- **Subagent context isolation** — each subagent gets its **own focused context** (an *isolated event stream with its own conversation history*). Benefits:
  - keeps each subagent **on-task** (no cross-contamination of instructions/results),
  - prevents one subtask's **noise** polluting another,
  - enables **parallel** execution.
- **Trade-off:** more agents = **more tokens + coordination complexity**. Use only when the task genuinely decomposes. (Relates to Claude Code subagents/`context: fork` in Domain 3.)

> **What "isolation" actually means (verified — Managed Agents orchestration):** it's **context** that's isolated, **not** a security boundary. Agents still **share the same sandbox, filesystem, and vault credentials** (vault creds are session-scoped). So:
> - **MCP servers are *agent-scoped*** — each agent definition declares its **own** servers/tools, so you *limit* an agent by giving it only the tools it needs (an **inline** agent has no definition, so it inherits the session agent's servers/tools).
> - **Permission policies** apply to the tools a run's agents call; **network controls** (`allowed_hosts` in a `limited` environment) gate MCP hosts.
> - **Delegation is one level deep** — a subagent can't spin up its own subagents.
> - To restrict a session to only the agents you list, disable inline agents (`inline_agents: {"type":"disabled"}`).
> **Exam framing:** context isolation = *focus & parallelism*, **not** containment. Real "blast-radius" control comes from **agent-scoped tools + permission policies + network limits** (ties to Domain 7 least privilege).

---

## 4. Building robust agentic systems
*The two investments that decide whether an agent is reliable in production: the tools it's given, and the guardrails around the loop.*

### 4.1 Tool & environment design (ACI)
Anthropic stresses: *"put as much effort into the ACI as into the human-computer interface."* Tool definitions deserve **prompt-engineering effort "just as much"** as the overall prompt — for agents it's the biggest driver of reliable behaviour.
- **Write tool descriptions like a docstring for a junior developer (verified):** what it does, when to use it, each param, **example usage, edge cases**, and the **boundary vs similar tools** (so the model doesn't confuse two) (Domain 8).
- Choose **formats the model saw often in training**; give it **room to reason before writing**; avoid formatting overhead like **line-counting or string escaping**.
- **"Poka-yoke"** the tools — change the arguments so it's **hard to make a mistake**. Verified example: on **SWE-bench**, requiring **absolute filepaths** (not relative) made the model use the tool reliably.
- Give enough **tokens/room to think**, and return **useful errors** (`is_error: true` on the `tool_result`) so the agent can self-correct.
- **Test/iterate:** run **many example inputs** through the tool and fix the mistakes you observe.
- **Transparency:** show the agent's **planning steps** explicitly — it's one of the doc's core principles alongside simplicity and a well-crafted ACI.

### 4.2 Guardrails, budgets & robustness
Robust long-running/agentic systems need:
- **Termination guards** — honour `stop_reason` **and** cap **max iterations/steps**, tokens, and time.
- **Validate intermediate results** (schema/sanity checks) before continuing.
- **Retries with exponential backoff** for transient failures (Domain 7).
- **Checkpoint state** so a failed step can **resume**, not restart the whole pipeline.
- **Graceful degradation** — one tool failure shouldn't kill the run; return `is_error` so the model recovers.
- **Human-in-the-loop** for sensitive/irreversible actions (Domain 7).
- **Least privilege** on tools to bound blast radius.

---

## 5. Pitfalls
*The exam loves "what's wrong / what's the fix" scenarios — these two subsections are the answer bank.*

### 5.1 Anti-patterns
- **No termination condition** → infinite tool-calling loop. **Fix:** honour `stop_reason` **+** add a **max-step guard**.
- **Ignoring `stop_reason`** and blindly re-prompting.
- **Over-engineering** — using an agent where a **single prompt or workflow** would do.
- **Not validating tool outputs** before feeding them back (also an injection risk — Domain 7).
- **Unbounded cost/latency** — no token/step/time budget.
- **Over-isolating or over-splitting** into agents that then can't share needed context.

### 5.2 Common traps
- "Agent" ≠ "anything with tools." A **fixed tool sequence is a workflow.**
- Cheaper ≠ agent. Agents are usually **more** expensive — prefer the **simplest** pattern that works.
- The model does **not** execute tools or hold session state — **your app does both**.
- **Infinite loops are a design bug** (no guard), not a model defect.
- Orchestrator–workers (**runtime** subtasks) ≠ parallel sectioning (**fixed** subtasks).
- Know the **two parallelisation flavours**: sectioning (split) vs voting (repeat/consensus).
- **Subagent context isolation ≠ containment** — agents share sandbox/filesystem/creds; limit them with agent-scoped tools + permission policies, not by assuming a boundary.
- **`tool_use` isn't the only non-final stop** — `pause_turn`, `max_tokens`, `refusal` also interrupt; a robust loop switches on all 7.

---

## 6. Quick-fire revision
- Q: Workflow vs agent in one line? **A:** Workflow = **predefined** code paths; agent = model **directs its own** steps/tools.
- Q: Anthropic's golden rule? **A:** Use the **simplest thing that works**; add complexity only when needed.
- Q: The basic building block? **A:** The **augmented LLM** (retrieval + tools + memory).
- Q: Name the five workflow patterns. **A:** Prompt chaining, routing, parallelisation, orchestrator–workers, evaluator–optimiser.
- Q: Pattern where output of step N feeds step N+1? **A:** **Prompt chaining**.
- Q: Pattern that classifies then specialises? **A:** **Routing**.
- Q: Two parallelisation flavours? **A:** **Sectioning** (split independent subtasks) and **voting** (repeat for consensus).
- Q: Orchestrator–workers vs sectioning? **A:** Orchestrator **decides subtasks at runtime**; sectioning uses **fixed** subtasks.
- Q: Pattern = generate → critique → refine? **A:** **Evaluator–optimiser**.
- Q: What ends an agent loop? **A:** `stop_reason: "end_turn"` (or an iteration/budget guard).
- Q: Who executes the tools in the loop? **A:** **Your application** (model only requests).
- Q: First fix for a runaway agent? **A:** Add a **max-step guard** + honour `stop_reason`.
- Q: Benefit of subagent context isolation? **A:** Focus, no cross-contamination, parallelism.
- Q: Does context isolation contain what a subagent can touch? **A:** **No** — agents **share sandbox/filesystem/vault creds**; real limits come from **agent-scoped MCP tools + permission policies + network controls**.
- Q: How deep can delegation go? **A:** **One level** — a subagent can't create its own subagents.
- Q: What should you invest in besides the prompt, for agents? **A:** The **tool/environment design (ACI)** — "as much effort as the human-computer interface."
- Q: How should a tool description read? **A:** Like a **docstring for a junior dev** — purpose, params, **example usage, edge cases, boundary vs similar tools**.
- Q: The poka-yoke SWE-bench example? **A:** Requiring **absolute filepaths** (not relative) made tool use reliable.
- Q: Canonical example of prompt chaining? **A:** **Generate copy → translate it** (sequential stages).
- Q: Canonical example of voting (parallelisation)? **A:** **Several prompts review the same code for vulnerabilities** → consensus.
- Q: Canonical orchestrator–workers example? **A:** A **code change spanning an unknown set of files**.
- Q: `stop_reason` when the model wants a tool vs when it's done? **A:** **`tool_use`** (run it, return `tool_result`) vs **`end_turn`** (finished).
- Q: Name the 7 `stop_reason` values. **A:** `end_turn`, `tool_use`, `max_tokens`, `stop_sequence`, `pause_turn`, `refusal`, `model_context_window_exceeded`.
- Q: One rule when returning tool results? **A:** **One `tool_result` per `tool_use`**, and **no text after** the results (ends the turn early).
- Q: Doc's one-line definition of an agent? **A:** *"LLMs using tools based on environmental feedback in a loop."*

**One-line summary:** *Prefer the simplest pattern (prompt → workflow → agent); compose from the augmented LLM (retrieval + tools + memory); know the five workflow patterns (chaining, routing, parallelisation·sectioning/voting, orchestrator–workers, evaluator–optimiser); build the agent loop around the 7 `stop_reason` values with step/budget guards; isolate subagent contexts for focus (not containment — that's agent-scoped tools + policies); and engineer the ACI (docstring-grade tool docs, poka-yoke, `is_error`) so the model can act and recover.*