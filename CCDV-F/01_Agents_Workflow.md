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
   - [4.2 Guardrails, failure handling & robustness](#42-guardrails-failure-handling--robustness)
   - [4.3 Hooks — guardrails made deterministic](#43-hooks--guardrails-made-deterministic)
5. [Pitfalls](#5-pitfalls)
   - [5.1 Anti-patterns](#51-anti-patterns)
   - [5.2 Common traps](#52-common-traps)
6. [Quick-fire revision](#6-quick-fire-revision)

---

## 1. Foundations
*The simplest-first ladder, the one block every pattern is built from, and the workflow-vs-agent line.*

### 1.1 The core mental model
There are three ways to get work done with an LLM, in rising order of power **and** risk:
- a **single prompt** — one call does it;
- a **workflow** — *you* wire the steps in code;
- an **agent** — the *model* wires its own steps at runtime.

The skill this domain rewards is picking the **least powerful option that still does the job** — because each rung up buys capability at the cost of latency, money, and new ways to fail.

> **Golden rule (Anthropic): use the simplest thing that works.** Start with a single prompt → add retrieval/examples/tools → use a **workflow** → reach for a full **agent** only when the task genuinely needs **dynamic decision-making**. Each step up adds capability **and** cost, latency, and failure surface.

Anthropic's vocabulary (know the distinction):
- **Workflows** — systems where LLMs and tools are orchestrated through **predefined code paths**.
- **Agents** — systems where the **LLM dynamically directs its own process and tool use**, keeping control over how it accomplishes the task.
- Both are **"agentic systems"**; the split is **who controls the flow** — your code (workflow) or the model (agent).

### 1.2 The augmented LLM (the building block)
A **plain LLM is just a text predictor** with three hard limits: it only knows its training data, it can't *do* anything in the world, and it forgets everything once a call ends. **Augmenting** means bolting on three capabilities that remove exactly those limits — and this upgraded block is what *every* pattern and agent below is built from.

Picture a brilliant person in a bare room with no phone, internet, or notepad — they can only answer from memory. The three augmentations hand them:

| Augmentation | Like giving them… | Removes the limit… | In practice |
|---|---|---|---|
| **Retrieval** | a search engine | stale / limited knowledge | fetch relevant external or private knowledge into context (RAG) |
| **Tools** | a phone with buttons | can't act or get live data | call out to code / APIs / systems (function calling) |
| **Memory** | a notepad | forgets between calls | carry state / history across steps |

The key twist: **the model drives these itself** — it writes its own search queries, picks which tool to call, and decides what to keep. 

> **Why "the building block":** get this *one* block solid first — clear tool descriptions, good retrieval — because workflows (§2) and agents (§3) are just this block **composed or repeated**. A shaky block makes every pattern above it shaky.

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

> **How Anthropic actually frames "task decomposition."** You may see practice questions call this *"task decomposition strategies"* — heads-up: that umbrella phrase is **not official Anthropic terminology** (it appears only on third-party exam-prep sites). Anthropic never bundles it under one name; it describes *breaking a complex task into subtasks* concretely, inside the patterns. Three of the five decompose a task, and the real thing a question is probing is **two axes — fixed vs. dynamic subtasks, and sequential vs. parallel:**
>
> | Pattern | Who defines the subtasks | Fixed or dynamic? | Shape |
> |---|---|---|---|
> | **Prompt chaining** (§2.1) | **you**, up front in code | **fixed** | **sequential** — each step feeds the next |
> | **Parallelization → sectioning** (§2.3) | **you**, up front in code | **fixed** | **parallel** — independent subtasks at once |
> | **Orchestrator–workers** (§2.4) | the **orchestrator LLM**, at runtime | **dynamic** (can't predict them) | fan out → gather → synthesise |
>
> The other two **don't decompose** at all — **routing** (§2.2) classifies an input and sends it down one specialised path, and **evaluator–optimiser** (§2.5) is a generate-then-critique loop — so if a "decomposition" question lists either, that's the distractor. (On the prompting side, manual decomposition has one official name: **"chain complex prompts,"** with *self-correction* — draft → review → refine — as its most common shape.)

### 2.1 Prompt chaining
A task is broken into a **fixed line of steps**, and each step's output becomes the next step's input — like an assembly line. *You* decide the stations up front in code; the LLM just works one station at a time. Because each call does one small, focused job, it's more accurate than asking for everything in a single shot.

- **Decompose a task into fixed steps**; each LLM call's output feeds the next.
- Add a programmatic **"gate" / check** between steps to catch errors early.
- **Use when:** the task **cleanly splits** into sequential subtasks (e.g. outline → draft → polish; generate → translate).
- Trades latency for **higher accuracy per step**.

### 2.2 Routing
A first step **classifies the input** and decides which of several specialised paths should handle it — like a receptionist directing you to the right department. Each path then runs its *own* focused prompt (and often its own model), so no single prompt has to be a jack-of-all-trades.

- **Classify the input**, then send it down a **specialised path** (often a cheaper/better-suited **model per path**).
- **Use when:** inputs fall into **distinct categories** better handled separately (support tiers: simple → Haiku, complex → Opus).
- Separates concerns → each path's prompt stays focused.

### 2.3 Parallelisation (sectioning / voting)
Instead of one call doing everything in sequence, you run **several LLM calls at the same time** and combine their outputs in code — faster, and often better. It comes in two flavours: split one task into **independent pieces** (sectioning), or run the **same task several times** and compare (voting).

- **Sectioning:** split work into **independent chunks** run **concurrently**, then combine (e.g. one subtask per document; guardrail check in parallel with the main answer).
- **Voting:** run the **same task multiple times** for diverse outputs / **consensus** (e.g. multiple reviewers, take majority).
- **Use when:** subtasks are independent (sectioning) or you want **confidence/coverage** (voting). Improves speed and/or quality.

> **"Fan-out / fan-in" (not official wording).** You'll hear this generic distributed-systems term for the shape parallelisation has — heads-up: **Anthropic never uses "fan-out" or "fan-in"** (I checked *Building Effective Agents* and the multi-agent research post). It's a handy mental model, not an exam term. It names the two halves of running work in parallel:
> - **Fan-out** = one coordinator kicks off **many pieces at once**. In Anthropic's own research system, *"the lead agent spins up 3–5 subagents in parallel rather than serially."*
> - **Fan-in** = **collect all the results and combine them**. In that same system, *"the LeadResearcher synthesises these results"* into the final answer.
>
> Where it maps on the five patterns: fan-out is **sectioning** (pieces **fixed** up front) or **orchestrator–workers** (§2.4, pieces decided **at runtime**); fan-in is the **synthesis/aggregation** step — or, in the **voting** flavour, a **consensus/majority** step instead of a merge. The most concrete place to see it is the **API itself** (§3.1): **fan-out** = Claude emitting **several `tool_use` blocks in one response** (parallel tool use); **fan-in** = you returning **all the `tool_result`s in one `user` message**, linked by `tool_use_id`. So whenever a question says "fan-out/fan-in," translate it to **parallel-dispatch → synthesise**.

### 2.4 Orchestrator–workers
A lead **"orchestrator" LLM looks at the task, decides *on the fly* what subtasks are needed**, hands each to a worker LLM, then stitches the results back together. The giveaway vs sectioning: the subtasks aren't fixed in your code — the orchestrator **invents them per input**.

- A **central LLM dynamically breaks down** the task, **delegates** to worker subagents, and **synthesises** their results.
- **Use when:** you **can't predict the subtasks in advance** (e.g. code changes spanning an unknown set of files; multi-source research).
- Difference from parallel sectioning: the subtasks are **decided at runtime by the orchestrator**, not fixed.

**How the workers actually run.** A **worker is normally a single augmented-LLM call** (§1.2), *not* a nested workflow by default. The run goes:
1. **Orchestrator call** → outputs a **list of subtasks** (usually as structured output / tool calls so your code can parse it).
2. **Your code dispatches a worker call per subtask** — each gets *that one subtask* as its prompt, its own focused context, and only the tools it needs. Workers usually run **in parallel**.
3. **Your code collects** every worker's output.
4. **Synthesis call** → the outputs go back to an LLM to **combine** into the final answer.

So just like the agentic loop, **your app runs the calls and moves data between them** — the model never dispatches workers itself. The **orchestrator + its workers together are the one workflow** (not "a workflow calling workflows"); a worker *could* be made more complex — even its own chain or agent — but the pattern doesn't require it.

```python
# 1) Orchestrator decides the subtasks (structured output → parseable list)
plan = client.messages.create(model="claude-opus-5-5", max_tokens=1024,
                              messages=[{"role": "user", "content": task}],
                              tools=[plan_tool], tool_choice={"type": "tool", "name": "plan"})
subtasks = extract_subtasks(plan)          # e.g. ["edit auth.py", "update tests", "fix docs"]

# 2) + 3) YOUR code fans out one worker call per subtask, in parallel, and collects results
def run_worker(st):
    r = client.messages.create(model="claude-sonnet-5-5", max_tokens=2048,
                               messages=[{"role": "user", "content": worker_prompt(st)}])
    return r.content
worker_outputs = [run_worker(st) for st in subtasks]   # run concurrently in practice

# 4) Synthesis: hand the gathered outputs back to an LLM to combine
final = client.messages.create(model="claude-opus-5-5", max_tokens=2048,
                               messages=[{"role": "user",
                                          "content": synthesise_prompt(task, worker_outputs)}])
```
> **Worker ≠ nested workflow by default** — it's one focused LLM call. Nesting is allowed (composition), not required.

### 2.5 Evaluator–optimiser
Two LLMs run a **writer-and-editor loop**: one generates an answer, the other grades it against criteria and gives feedback, and the first revises — repeating until the evaluator is satisfied. It works when you can *say* what "good" looks like and iteration measurably improves the result.

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
*The loop itself, then coordinating several agents.*

### 3.1 Agents & the agentic loop
An **agent** plans and acts in a loop, using tools and **environment feedback**, until it decides the goal is met (or a stop condition hits). Use agents for **open-ended problems where you can't hardcode the steps** and you can trust the model's decision-making with good tools + guardrails.

> **Orchestrator–workers (workflow) vs agent — the subtlest line in this domain.** Both decompose at runtime, so the orchestrator *looks* agentic. The split is **what's fixed vs. what the model controls** — and note that *Building Effective Agents* classes orchestrator–workers as a **workflow**, not an agent.
> | | **Orchestrator–workers** (workflow) | **Agent** |
> |---|---|---|
> | Overall structure | **Fixed scaffold your code defines**: decompose → delegate → synthesise | **No fixed scaffold** — the model drives a loop |
> | What's dynamic | Only **what the subtasks are** | **Everything** — next action, which tool, order, when to stop |
> | Who owns the flow | **Your code** (orchestrator just fills the slots) | **The model** |
> | Shape of a run | Usually **one pass**: fan out → gather → done | **Iterates** on feedback until done / a guard trips |
> | Bounding | Easy to predict & bound | Emergent — *must* add step/budget guards |
>
> **One-liner:** *orchestrator–workers = the model fills slots in a flow **your code** owns; agent = the model **owns the flow**.* The orchestrator decides subtask *contents*, never the control flow itself.

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

**What the user turn looks like — single vs. parallel tool calls.** Claude may request **one tool, or several in the *same* response** (parallel tool use). Either way you reply with **exactly one `user` message**, carrying **one `tool_result` per `tool_use`**, each linked by **`tool_use_id`** (it's the id that matches them, not the order).

*Case A — single tool call:*
```jsonc
// assistant response  (stop_reason: "tool_use")
"content": [
  {"type": "text", "text": "Let me check the weather."},
  {"type": "tool_use", "id": "toolu_01", "name": "get_weather", "input": {"city": "Paris"}}
]
// YOUR next user turn — one tool_result
"content": [
  {"type": "tool_result", "tool_use_id": "toolu_01", "content": "18°C, clear"}
]
```

*Case B — two tool calls in one response (parallel):*
```jsonc
// assistant response  (stop_reason: "tool_use") — TWO tool_use blocks
"content": [
  {"type": "tool_use", "id": "toolu_01", "name": "get_weather", "input": {"city": "Paris"}},
  {"type": "tool_use", "id": "toolu_02", "name": "get_time",    "input": {"city": "Paris"}}
]
// YOUR next user turn — BOTH results in ONE user message
"content": [
  {"type": "tool_result", "tool_use_id": "toolu_01", "content": "18°C, clear"},
  {"type": "tool_result", "tool_use_id": "toolu_02", "content": "14:05 CET"}
]
```

*Code that handles both (loop over the blocks):*
```python
tool_results = []
for block in resp.content:
    if block.type == "tool_use":
        out = run_tool(block.name, block.input)      # your dispatch
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": block.id,                 # ← link by id, not position
            "content": out,
            # "is_error": True,                      # ← set on a failed tool call
        })
messages.append({"role": "assistant", "content": resp.content})   # the tool_use turn, verbatim
messages.append({"role": "user", "content": tool_results})        # ONE user msg, N results
```
> **Traps:** all results for a parallel batch go in **one** user message (not one message each); a **missing** `tool_result` for any `tool_use_id` errors; mark failures with **`"is_error": true`**; to force **at most one** call per turn, set **`disable_parallel_tool_use: true`**.

**`stop_reason` — the 7 values (verified; the loop's switchboard):**

| Value | Meaning (who/what caused it) | Normal or problem? | Loop action |
|---|---|---|---|
| **`end_turn`** | Claude finished naturally (most common) | Normal | Return the answer |
| **`tool_use`** | Claude is calling a tool you must run | Normal | Execute → append `tool_result` → loop |
| **`max_tokens`** | Hit **your** `max_tokens` cap | Problem (truncated) | If it's a truncated `tool_use`, retry higher; else continue / flag |
| **`stop_sequence`** | **You** set a stop string; Claude emitted it | Normal (your design) | Read which one fired (the `stop_sequence` field) |
| **`pause_turn`** | A **server-tool** loop hit its iteration cap mid-run | Normal | Append the content back unchanged and resend |
| **`refusal`** | Claude **declined** (safety/policy) | Problem | Read `stop_details`; retry on a fallback model |
| **`model_context_window_exceeded`** | Output **filled the model's window** | Problem (truncated) | Treat the output as truncated |

**The less-obvious four, in plain terms:**
- **`stop_sequence`** — *you* pass a list of `stop_sequences` (strings that mean "stop the moment you produce this"). When Claude emits one, generation halts and the response tells you **which** string fired. It's **your own cut-off working as intended**, not an error. (Never set any? You'll never see this.)
  - *Real-life example:* you prompt Claude to write **one** Q&A pair and pass `stop_sequences=["\nQ:"]`. The instant Claude starts a *second* question with `Q:`, generation stops — so you get exactly one pair, no runaway list. `stop_reason: "stop_sequence"`, `stop_sequence: "\nQ:"`.
  ```python
  resp = client.messages.create(
      model="claude-sonnet-5-5", max_tokens=1024,
      stop_sequences=["\nQ:"],                 # ← stop before a 2nd question
      messages=[{"role": "user", "content": "Write a Q&A about photosynthesis."}])
  # resp.stop_reason == "stop_sequence";  resp.stop_sequence == "\nQ:"
  ```
- **`pause_turn`** — only with **server-side tools** (ones Anthropic runs, e.g. web search) that loop. If that loop hits an internal iteration limit **mid-task**, the API pauses and hands back the work so far. **Send the assistant's content back unchanged** to continue — think *"⏸ to be continued,"* not a failure. (Client tools *you* run give `tool_use`, never this.)
- **`refusal`** — Claude **declined** on safety/policy grounds. Arrives as a normal **HTTP 200** (not an error code), but the content won't be your expected answer. Read `stop_details` and **retry on a fallback model** (Haiku 5.5 has no server-side fallback — wire that retry yourself).
- **`model_context_window_exceeded`** — while generating, **input + output filled the model's whole window**, so it stopped. **Treat the output as truncated.** Contrast `max_tokens` (*your* output cap) vs this (the model's **hard window limit**, on 4.5+ models; older models raise an error instead).

> **Tip (verified):** for most tool loops the docs recommend the SDK **tool runner**, which handles execution, result formatting, and history for you — hand-rolling the loop is for when you need custom control.

### 3.2 Multi-agent / orchestrator design
Sometimes one agent isn't the right shape for the job. When a goal splits into chunks that can be worked on independently — research three competitors, review five files, draft six sections — you put **one agent in charge (the orchestrator)** and let it hand pieces out to **helper agents (subagents)** that each work in their own private context, then it stitches the results back together. Think of it like a lead engineer breaking a project into tickets, handing each to a teammate who doesn't need to see the others' code, and then assembling the final build from what comes back.

The win is **focus and parallelism**: every subagent sees only its own slice, so instructions and results don't bleed across tasks, and the slices can run at the same time. The cost is **tokens and coordination** — every extra agent is more calls to pay for and more moving parts to get wrong — so reach for this only when the task genuinely decomposes, not by default. And one thing it is **not**: giving a subagent its own context does **not** sandbox it (see the callout below) — isolation is about keeping each agent on-task, not about containment.

> **"Wait — isn't this just the orchestrator–workers *workflow* from §2.4?"** They look like twins, because both "break a goal into pieces, delegate them, and synthesise the results." The giveaway that tells them apart is **not the shape** — it's **what a "worker" is and who owns the control flow:**
> - In the **§2.4 workflow**, each "worker" is a **plain LLM call** your code fires off and collects. The orchestrator LLM only decides *what* the subtasks are; **your code owns the flow** (fan out → gather → done, usually one pass). That fixed, code-owned flow is exactly why §2.4 is a *workflow*.
> - In a **multi-agent system (here)**, a worker can be a **full agent** — running *its own* loop, with its own tools and private context, iterating many steps before it reports back — and the orchestrator can itself be an **agent** deciding on the fly when and whom to delegate to. That autonomy is why this lives under **Agents**.
>
> | | **Orchestrator–workers** *(§2.4 — a workflow)* | **Multi-agent system** *(§3.2 — agents)* |
> |---|---|---|
> | A "worker" is… | a **single LLM call** filling a slot | a **full agent** (own loop, tools, context) |
> | Who owns the control flow | **your code** (fixed scaffold) | the **orchestrator/model** (dynamic) |
> | Shape of a run | usually **one fan-out pass** | agents **iterate**; delegation is **one level deep** |
> | What you worry about | decomposition & synthesis quality | **isolation vs. sharing, tool scoping, blast radius** |
>
> **One-liner:** orchestrator–workers is the **bounded, code-owned special case**; a multi-agent system is the **general, model-owned case** where the pieces are real agents — more power, less predictability.

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

**Spinning up subagents programmatically (Python Agent SDK, verified).** You don't build the orchestrator loop by hand — you **declare** the subagents in the `agents` parameter and let Claude invoke them through the built-in **`Agent` tool**. Each one is an `AgentDefinition` whose **`description`** tells Claude *when* to use it, **`prompt`** is its system prompt, and **`tools`** restricts what it may touch (omit → inherits all subagent tools; this is your least-privilege knob). `model` is optional (`"opus"`/`"sonnet"`/`"haiku"`/`"inherit"`/a full ID).

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def main():
    async for message in query(
        prompt="Review the auth module for security issues, then run the tests",
        options=ClaudeAgentOptions(
            # "Agent" MUST be allowed — it's the tool Claude uses to spawn a subagent
            allowed_tools=["Read", "Grep", "Glob", "Agent"],
            agents={
                "code-reviewer": AgentDefinition(
                    description="Security/quality review specialist. Use for code review.",
                    prompt="You are a code reviewer. Flag security and performance issues.",
                    tools=["Read", "Grep", "Glob"],   # read-only → can't modify files
                    model="sonnet",                     # optional per-agent override
                ),
                "test-runner": AgentDefinition(
                    description="Runs and analyses test suites.",
                    prompt="You run tests and summarise failures.",
                    tools=["Bash", "Read", "Grep"],     # Bash → can execute tests
                ),
            },
            max_budget_usd=5.0,                         # hard spend cap for the whole run
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

- **Who decides to spawn?** Claude does — **automatic delegation** matches the task to each subagent's `description`. To **force** one, name it in the prompt (*"Use the `code-reviewer` agent to…"*). That's the SDK version of the §2.4-vs-§3.2 line: here the **model** owns the delegation, not your code.
- **Dynamic definitions:** an `AgentDefinition` is just data, so a **factory function** can build one per request (e.g. `model="opus"` for a strict review, `"sonnet"` otherwise) — subtasks decided at runtime, exactly the orchestrator pattern.
- **Detect a spawn** in the stream: a `tool_use` block whose `name` is `"Agent"` (older SDKs: `"Task"` — match both); messages from inside a subagent carry `parent_tool_use_id`.
- **Bound the tree (the guardrail that matters):** Claude chooses *how many* to spawn and subagents can spawn their own, so one prompt can fan into a tree. Cap it with **`max_budget_usd`** and the env vars **`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`** (default 3; set 1 to stop nesting) and **`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`** (default 20). **Opus 5 delegates more eagerly**, so these matter most there.
- **Exam traps:** forget **`"Agent"`** in `allowed_tools` → Claude can't spawn anything; a spawned subagent **runs in the background by default** (`status: "async_launched"`) unless Claude needs its report or you set `background`/ask it to wait; the parent sees **only the subagent's final message**, never its intermediate tool calls — so put every path/error/decision the subagent needs **in the prompt string**, since it starts with a fresh context.

**Gathering the results — the SDK does most of it for you.** This is the big difference from the hand-rolled orchestrator in §2.4, where *your* code collected each worker's output and fired a separate synthesis call. With `query()`, **Claude is the orchestrator**: each subagent's final report flows back to the parent automatically, the parent combines them, and the run ends with a single **`ResultMessage`** whose **`.result`** is the finished, user-facing answer. So in the common case you don't "gather" anything — you just consume the stream and pick out that last message:

```python
async def run(prompt, options):
    final_answer, cost = None, None
    async for message in query(prompt=prompt, options=options):
        # The run's LAST message is a ResultMessage; .result is the
        # parent's synthesised, ready-to-show answer.
        if hasattr(message, "result"):
            final_answer = message.result
            cost = getattr(message, "total_cost_usd", None)
    return final_answer, cost      # ← final_answer is what you show the user
```

**When you want each subagent's raw output (not the parent's paraphrase).** The parent may *summarise* a subagent's report inside its final answer. If you need the outputs verbatim — to display them separately, post-process them, or assemble your own document — capture them from the stream yourself. A subagent's final report returns as the **`Agent` tool's result** (a `ToolResultBlock`), and you match it to the spawn by `tool_use_id`:

```python
from claude_agent_sdk import query, ToolUseBlock, ToolResultBlock

async def run_and_collect(prompt, options):
    spawned, reports, final_answer = {}, {}, None     # id→name, name→report
    async for message in query(prompt=prompt, options=options):
        for block in getattr(message, "content", None) or []:
            # 1. Record each spawn: the Agent tool call names the subagent
            if isinstance(block, ToolUseBlock) and block.name in ("Agent", "Task"):
                spawned[block.id] = block.input.get("subagent_type", "unknown")
            # 2. The matching result block carries that subagent's final report
            if isinstance(block, ToolResultBlock) and block.tool_use_id in spawned:
                reports[spawned[block.tool_use_id]] = block.content   # str or block list
        if hasattr(message, "result"):
            final_answer = message.result
    return reports, final_answer
```

> **Tip:** if you take the *parent's* answer (snippet 1) but need a subagent quoted exactly, tell the parent so in the main `query()` prompt/`systemPrompt` (e.g. "include each reviewer's findings verbatim") — otherwise it may compress them.

**Preparing the final output yourself.** Once you hold the raw `reports`, you assemble the deliverable in code instead of trusting the model's merge — predictable structure, every section present, no paraphrasing:

```python
def build_report(reports: dict) -> str:
    out = ["# Code review summary\n"]
    for name, text in reports.items():
        body = text if isinstance(text, str) else \
               "".join(b.get("text", "") for b in text)   # flatten block list
        out.append(f"## {name}\n{body}\n")
    return "\n".join(out)

reports, parent_answer = await run_and_collect(prompt, options)
final = build_report(reports) if reports else parent_answer   # your output, or fall back
```

So: **trust the parent's synthesis** (snippet 1) for the simple case, or **collect-and-assemble yourself** (snippets 2–3) when you need control over the exact final output — the same "gather → synthesise" shape as §2.4, just with the gathering happening in the SDK stream rather than your own dispatch loop.

---

## 4. Building robust agentic systems
*Two investments decide whether an agent is reliable: the tools it's given, and the guardrails around the loop.*

### 4.1 Tool & environment design (ACI)
Anthropic stresses: *"put as much effort into the ACI as into the human-computer interface."* Tool definitions deserve **prompt-engineering effort "just as much"** as the overall prompt — for agents it's the biggest driver of reliable behaviour.
- **Write tool descriptions like a docstring for a junior developer (verified):** what it does, when to use it, each param, **example usage, edge cases**, and the **boundary vs similar tools** (so the model doesn't confuse two) (Domain 8).
- Choose **formats the model saw often in training**; give it **room to reason before writing**; avoid formatting overhead like **line-counting or string escaping**.
- **"Poka-yoke"** the tools — change the arguments so it's **hard to make a mistake**. Verified example: on **SWE-bench**, requiring **absolute filepaths** (not relative) made the model use the tool reliably.
- Give enough **tokens/room to think**, and return **useful errors** (`is_error: true` on the `tool_result`) so the agent can self-correct.
- **Test/iterate:** run **many example inputs** through the tool and fix the mistakes you observe.
- **Transparency:** show the agent's **planning steps** explicitly — it's one of the doc's core principles alongside simplicity and a well-crafted ACI.

### 4.2 Guardrails, failure handling & robustness
**Why agents fail more than workflows.** An agent gains *"ground truth"* from the environment at each step (tool results, errors, test output) and adapts — that's its strength. But the same autonomy means **"higher costs, and the potential for compounding errors"**: a small wrong turn early can snowball over many turns. Anthropic's prescription is blunt — *"extensive testing in sandboxed environments, along with the appropriate guardrails"* — plus **programmatic checks** on intermediate steps "to ensure that the process is still on track," and the ability to **"pause for human feedback at checkpoints or when encountering blockers."**

**The key move: a failure is not an exception to crash on — it's feedback to feed back.** The loop stays alive; you hand the agent the error (or a human) and let it recover. Here's each failure mode and the handling:

| Failure mode | What it looks like | How you handle it |
|---|---|---|
| **Transient tool error** (timeout, 5xx, rate limit) | the tool itself throws | **retry with exponential backoff**; return `is_error: true` on the `tool_result` so the agent can adapt if retries fail |
| **Bad tool input** the model chose | wrong args, not-found, validation error | return a **useful error message** (`is_error: true`) → the model **self-corrects** next turn; then **poka-yoke** the tool so it's hard to get wrong again |
| **Invalid / off-schema output** | malformed JSON, missing field | **validate before using**; re-prompt, or enforce shape with **strict tools / `output_config`** (Domain 6) |
| **No progress / infinite loop** | same action repeated, never reaches `end_turn` | **budget guard** — cap max steps/tokens/time (`for _ in range(MAX_STEPS)`, never `while True`); detect stalls |
| **Refusal** (`stop_reason: "refusal"`) | HTTP **200**, but the content isn't your answer | read `stop_details`; **retry on a fallback model** (Haiku 5.5 has no server fallback — wire it yourself) |
| **Context window exceeded** (`model_context_window_exceeded`, or 400 "prompt too long") | run stops mid-generation | treat the output as **truncated**; **compact/trim** history (Domain 6) before resuming |
| **Sensitive / irreversible action** | agent about to delete, pay, email, deploy | **human-in-the-loop checkpoint** *before* it runs; **least privilege** on tools to bound blast radius (Domain 7) |
| **Compounding drift** | small errors accumulate over many turns | **programmatic checkpoints** that verify "still on track"; **checkpoint state** so a failed step can **resume**, not restart the whole run |
| **Multi-agent cascade** | one subagent errors or returns junk | **graceful degradation** — one failure shouldn't kill the run; isolate it, synthesise around the gap or re-delegate |

**Robustness checklist (the short version to memorise):**
- **Termination guards** — honour `stop_reason` **and** cap **max iterations/steps**, tokens, and time.
- **Validate intermediate results** (schema/sanity checks) before continuing.
- **Retries with backoff** for transient failures; **`is_error`** for ones the model should see.
- **Checkpoint & resume**, don't restart.
- **Graceful degradation** over hard crashes.
- **Human-in-the-loop** for sensitive/irreversible actions; **least privilege** to bound blast radius.
- **Test extensively in a sandbox** before trusting autonomy.

> **Exam framing:** the answer to "an agent's tool call failed — what do you do?" is almost never *"abort"*. It's **return the error to the model (`is_error: true`) and let it recover**, with a **step/budget guard** so recovery can't loop forever, and a **human checkpoint** reserved for anything sensitive or irreversible. Crash-and-abort is the distractor.

### 4.3 Hooks — guardrails made deterministic
The guardrails in §4.2 are only as good as their enforcement. If you *ask* the model to respect a rule, it usually will — but "usually" isn't a guarantee. **Hooks** close that gap: a hook is a **deterministic handler** (a shell command, or an HTTP / MCP-tool call) that the framework fires **automatically at a fixed point in the agent's lifecycle**, in *your* code, every time — so the rule is enforced regardless of what the model decides. Think of hooks as **the deterministic layer wrapped around a non-deterministic agent**.

**Scope note:** hooks are a **Claude Code / Agent SDK** feature — **not** part of the raw Messages API loop. In a hand-rolled loop you *are* the hook (your code gates each tool call yourself); hooks are the framework giving you that gate declaratively, configured in settings files or plugins with **matchers** (by tool name, or by agent type).

**The lifecycle events.** An agent's run isn't one atomic action — it moves through a predictable sequence of moments: a session starts, a prompt comes in, Claude decides to call a tool, the tool runs, a subagent spins up and later finishes, the context gets compacted, Claude stops, the session ends. A hook is just you saying *"run my code at **this** moment."* So the table below is really the **menu of moments you can attach to** — pick the event for the point you want to observe or control. The column that matters most is the last one: only some moments let your hook **block** what happens next (those are your enforcement points); the rest are **observe-only** (logging, notifications, context injection). As a rule of thumb, the **`Pre*`** events and the **stop** events can block; the **`Post*` / `*End` / `Notification`** events cannot, because the thing already happened.

| Event | Fires when | Can it block? |
|---|---|---|
| `SessionStart` | a session begins or resumes | No (context only) |
| `UserPromptSubmit` | a prompt is submitted, before Claude processes it | Yes (exit 2) |
| **`PreToolUse`** | **before a tool call executes** | **Yes — the key gate** |
| `PostToolUse` | after a tool call succeeds | No (it already ran) |
| `Stop` | Claude finishes responding | Yes (exit 2 keeps it going) |
| **`SubagentStart`** | **a subagent is spawned** | — |
| **`SubagentStop`** | **a subagent finishes** | Yes (exit 2 prevents it stopping) |
| `PreCompact` | before context compaction | Yes (exit 2 blocks it) |
| `SessionEnd` / `Notification` | session ends / a notification is sent | No |

**The agent-specific parts (why hooks matter here):**
- **`SubagentStart` / `SubagentStop`** fire around each subagent. A `Stop` hook declared in a **subagent's own frontmatter is converted to `SubagentStop`**; its input carries **`last_assistant_message`** (use that, not the transcript, which can lag a turn).
- **Tool hooks fire *inside* subagents too** — when a subagent calls a tool, the same `PreToolUse` / `PostToolUse` hooks run, and the input carries **`agent_id` and `agent_type`**, so you can scope a guardrail to one kind of subagent. (Hooks from settings, managed policy, and plugins are inherited into subagents; a subagent's *own* hooks are removed when it finishes.)

**How `PreToolUse` gives deterministic control** (the "block the agent's action in code" mechanic):
- **Exit code 2** → **blocks** the tool call; the hook's `stderr` is handed back to Claude as the reason.
- **Exit 0 with no output** → "no decision," so the call continues through the **normal permission flow** — *silence is not approval*.
- **JSON decision** via `hookSpecificOutput.permissionDecision` = `allow` / `deny` / `ask` / `defer` (+ a reason), and it can even **rewrite the arguments** with `updatedInput`.
- **Precedence:** **exit 2 beats everything** — even a JSON `"allow"` can't override it.
- **Fail-safe:** a timed-out hook does **not** block by default → set **`onFailure: "block"`** so a crashing/missing guard **fails closed**, not open.

```bash
#!/bin/bash
# PreToolUse guard: deny destructive shell commands deterministically
input=$(cat)
command=$(jq -r '.tool_input.command' <<<"$input")
if [[ "$command" == *"rm -rf"* ]]; then
  jq -n '{hookSpecificOutput: {hookEventName: "PreToolUse",
          permissionDecision: "deny",
          permissionDecisionReason: "Destructive command blocked"}}'
  exit 0
fi
exit 0   # no decision → normal permission flow applies
```

> **Exam framing:** hooks = **guardrails made deterministic**. If a question asks how to *guarantee* an agent never runs a forbidden tool (or to force human approval before a risky one), the answer is a **`PreToolUse` hook** that denies/asks — **not** a better prompt. Remember: **exit 2 blocks and overrides a JSON `allow`**, silence ≠ approval, and set **`onFailure: "block"`** for a guard that must fail closed. The agent-scoped events are **`SubagentStart` / `SubagentStop`**, plus tool hooks that see **`agent_id` / `agent_type`** inside subagents.

---

## 5. Pitfalls
*The mistakes to avoid, and the one-liners that catch them.*

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