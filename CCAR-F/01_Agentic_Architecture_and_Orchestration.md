# CCA-F Revision Note — Domain 1: Agentic Architecture & Orchestration

> **Exam weight: ~27% — the single heaviest domain.** This is where the Architect exam lives. Questions are *design* questions: "which architecture fits this business case?", "who should own this responsibility?", "should this be a workflow, an agent, or a sub-agent?", "enforce it or ask for it?" — never "write the code."

> **Source note (verified):** Conceptual patterns are grounded in Anthropic's *Building Effective Agents*. Sub-agent mechanics (context isolation, tool inheritance, delegation, model resolution) are verified against the official Claude Code **Subagents** documentation (code.claude.com/docs/en/sub-agents). Where the Developer exam tested *what* a pattern is, the Architect exam tests *when to reach for it and what it costs*.

---

## Contents
1. [The architect's mental model](#1-the-architects-mental-model)
   - [1.1 The least-power ladder](#11-the-least-power-ladder)
   - [1.2 Workflow vs Agent — who controls the flow](#12-workflow-vs-agent--who-controls-the-flow)
2. [The five orchestration patterns (design lens)](#2-the-five-orchestration-patterns-design-lens)
3. [The agentic loop](#3-the-agentic-loop)
   - [3.1 How the loop terminates](#31-how-the-loop-terminates)
   - [3.2 pause_turn and long-running loops](#32-pause_turn-and-long-running-loops)
4. [Multi-agent architecture (sub-agents)](#4-multi-agent-architecture-sub-agents)
   - [4.1 The coordinator–subagent pattern](#41-the-coordinatorsubagent-pattern)
   - [4.2 Context isolation — the rule that trips people](#42-context-isolation--the-rule-that-trips-people)
   - [4.3 Tool access & least privilege per role](#43-tool-access--least-privilege-per-role)
   - [4.4 When to use a sub-agent (and when not to)](#44-when-to-use-a-sub-agent-and-when-not-to)
5. [Enforce vs ask — the architect's signature decision](#5-enforce-vs-ask--the-architects-signature-decision)
6. [Escalation, confidence & human-in-the-loop](#6-escalation-confidence--human-in-the-loop)
7. [Anti-patterns & exam traps](#7-anti-patterns--exam-traps)
8. [Quick-fire revision](#8-quick-fire-revision)

---

## 1. The architect's mental model

### 1.1 The least-power ladder

There are rungs of capability, each buying power at the cost of **latency, money, and new failure modes**:

- **Single prompt** → one call does it.
- **Augmented prompt** → add retrieval, examples, or a tool.
- **Workflow** → *you* wire the steps in code (predefined paths).
- **Agent** → the *model* wires its own steps at runtime (dynamic control).
- **Multi-agent** → a coordinator delegates to sub-agents with isolated context.

> **Golden rule (unchanged from Developer, but now the whole exam):** pick the **least powerful option that still does the job.** The correct Architect answer is almost always the *simpler* architecture that meets the requirement — not the most capable one. An agent where a workflow would do is an over-engineering wrong answer; a multi-agent system where a single agent would do is worse.

**Exam framing:** when a scenario describes a task with **fixed, known steps**, the answer is a **workflow**. When it describes a task needing **dynamic decisions, open-ended tool use, or unknown step count**, the answer is an **agent**. "The number of steps can't be known ahead of time" is the phrase that signals *agent*.

### 1.2 Workflow vs Agent — who controls the flow

| | **Workflow** | **Agent** |
|---|---|---|
| Who controls the flow | Your **code** | The **model** |
| Steps | Predefined paths | Decided at runtime |
| Predictability | High | Lower |
| Cost / latency | Lower, bounded | Higher, variable |
| Use when | Steps are known & stable | Task needs dynamic decision-making |

Both are "**agentic systems**." The dividing line is **control of the flow**, nothing else. Memorise this — scenarios bait you with complexity ("it uses tools!" "it's multi-step!") that does *not* make something an agent. Tools + fixed steps = still a workflow.

---

## 2. The five orchestration patterns (design lens)

The Architect exam reuses the *Building Effective Agents* patterns, but asks **which one fits a described system**:

| Pattern | Shape | Pick it when the scenario says… |
|---|---|---|
| **Prompt chaining** | Output of step N → input of step N+1 | Task decomposes into **fixed sequential subtasks**; each step can be checked before the next ("gate"). |
| **Routing** | Classify input → send to a specialised handler | Inputs fall into **distinct categories** best handled separately (e.g., refund vs technical vs billing). |
| **Parallelisation** | Run subtasks concurrently, then aggregate | Independent subtasks (**sectioning**) or multiple attempts for **voting/consensus**; latency matters. |
| **Orchestrator–workers** | A lead model **dynamically** breaks work into subtasks and delegates | Subtasks **can't be predicted in advance**; the lead decides the breakdown at runtime. |
| **Evaluator–optimiser** | One model generates, another critiques, loop | There are **clear eval criteria** and iterative refinement measurably helps. |

> **Routing vs Orchestrator–workers (classic trap):** Routing picks **one** predefined path from a known set (static classification). Orchestrator–workers **decides the subtasks dynamically** and may spawn many. If the categories are known up front → routing. If the breakdown is discovered at runtime → orchestrator–workers.

> **Parallelisation vs Orchestrator–workers:** Parallelisation splits work you **already know how to split**, runs it concurrently, aggregates. Orchestrator–workers is dynamic. "Fan out to the same analysis over many files" = parallelisation; "figure out what analyses are even needed" = orchestrator.

---

## 3. The agentic loop

An agent runs: **call model → model requests tool(s) → execute tool(s) → feed results back → repeat → model stops.** The loop is the engine under every agent and multi-agent system.

### 3.1 How the loop terminates

The loop ends when the model returns a **`stop_reason` that is not `tool_use`** — typically **`end_turn`** (the model is done and has no more tool calls). As long as the model keeps returning `tool_use`, the harness keeps looping.

> **Design rule:** a well-built agent loop has a **termination condition** — it ends on `end_turn`, and it also needs **guards** against never terminating: a **max-iteration cap**, a **budget/turn limit**, and an **escalation path** when it stalls. "The agent could loop forever" is a reliability defect the exam expects you to fix with a bound, not with a bigger timeout.

Other terminal stop reasons to recognise: `max_tokens` (ran out of output budget — not an error, resumable), `stop_sequence`, `refusal`, and on 4.5+ models `model_context_window_exceeded` (input + max_tokens exceeds the window — returned as a **stop_reason on a 200**, *not* a 400).

### 3.2 pause_turn and long-running loops

With **server-side tools** (e.g., server-run tool loops), a long turn can hit the server's iteration cap and return **`stop_reason: "pause_turn"`**. This is **not an error and not a failure**. The correct handling: **send the response back unchanged, with the same tools**, to let the loop continue. Don't retry, don't modify the content — just continue.

---

## 4. Multi-agent architecture (sub-agents)

This is the heart of Domain 1. Sub-agents are how an architect **isolates context and enforces least privilege** within an agentic system.

### 4.1 The coordinator–subagent pattern

A **coordinator (lead) agent** holds the overall goal and **delegates** bounded tasks to **sub-agents**, each of which runs in its **own context window** and returns a **summary**. The coordinator composes the final result from those summaries.

Why an architect reaches for it:
- **Context hygiene:** a noisy side-task (reading 50 files, running a test suite) fills a context window with output you'll never reference again. Isolate it in a sub-agent so only the *summary* returns to the main thread.
- **Least privilege:** give each sub-agent only the tools its role needs (a reviewer gets read-only; a researcher gets no write/edit).
- **Specialisation:** each sub-agent gets its own system prompt tuned to its job.

In Claude Code, a sub-agent is a **Markdown file with YAML frontmatter** in `.claude/agents/` (project) or `~/.claude/agents/` (user). Frontmatter body = the sub-agent's **system prompt**.

```markdown
---
name: code-reviewer
description: Reviews code for quality, security and best practices
tools: Read, Glob, Grep        # allowlist → read-only, no MCP tools
model: sonnet                  # or opus / haiku / fable / inherit / full ID
---
You are a code reviewer. Analyse the diff and give specific, actionable feedback.
```

Required fields: **`name`** and **`description`** only. `description` is what the coordinator uses to **decide when to delegate** — write it as "when to use this."

### 4.2 Context isolation — the rule that trips people

> **The one fact most people get wrong:** a (non-fork) sub-agent **does NOT see the main conversation.** Per the official docs: *"It doesn't see your conversation history, the skills you've already invoked, or the files Claude has already read."* It starts from its **own** system prompt + a **delegation message** the coordinator writes + CLAUDE.md files + a git snapshot.

Consequences an architect must design around:

- **In:** the sub-agent only knows what the coordinator **writes into the delegation prompt** (plus files it reads itself). **A rule or constraint that must reach the sub-agent has to be restated in that prompt** — it does *not* inherit the main thread's context automatically.
- **Out:** the sub-agent returns **only its final summary**, not its full transcript. This is the feature — verbose work stays out of the main context.
- **Fresh each time:** every invocation is a **new instance** unless the coordinator **resumes by agent ID** (which restores full prior history).
- **Exception — fork:** a *fork* sub-agent **inherits the parent conversation** instead of starting blank. Use a fork when the side-task genuinely needs the full context; use a normal sub-agent when it shouldn't.

> **Exam trap — "explicit context passing":** if a scenario says a sub-agent "didn't follow the project's rule" or "didn't know about X," the fix is **pass it explicitly in the delegation prompt** (or put it in CLAUDE.md), **not** "assume it inherits the main context." Softer-wrong = "it should have known." Precise-right = "restate it in the delegation message."

### 4.3 Tool access & least privilege per role

Verified tool-resolution rules (official docs):

- **`tools` omitted** → sub-agent **inherits** all tools available to sub-agents (built-in + MCP), minus a fixed always-removed set (e.g., `AskUserQuestion`, `EnterPlanMode`, `ExitPlanMode`).
- **`tools` as an allowlist** → **only** those tools, nothing else. `tools: Read, Grep, Glob, Bash` → **cannot Write/Edit and gets no MCP tools.**
- **`disallowedTools`** (denylist) → applied **first**, then `tools` resolves against what remains. A tool in both is removed.
- If **no** listed tool resolves, the sub-agent **usually fails to launch**.

> **Architect takeaway:** to enforce least privilege, **use an allowlist (`tools:`)** — don't omit the field and hope. Omitting = inherits everything = over-privileged. "Give the agent only the tools its role requires" is the recurring *precise* answer.

Model resolution order (highest first): per-invocation `model` param → frontmatter `model` (`inherit` = main model) → `CLAUDE_CODE_SUBAGENT_MODEL` → main conversation's model.

### 4.4 When to use a sub-agent (and when not to)

| Use a sub-agent when… | Keep it in the main agent when… |
|---|---|
| A side-task would **flood context** with output you won't reuse | The task needs **frequent back-and-forth** with the main thread |
| You need to **enforce tool restrictions** on a bounded job | The work is tightly coupled to ongoing conversation state |
| The work is **parallelisable** / independent | Latency of spinning up isolated context isn't worth it |
| You want a **specialised system prompt** for the job | The task is trivial |

> **Over-engineering trap:** multi-agent adds coordination overhead, latency, and more failure surface. If a **single agent with the right tools** meets the requirement, that's the correct answer — not a committee of sub-agents.

---

## 5. Enforce vs ask — the architect's signature decision

This is the Domain 1 version of your "softer vs precise" reflex, and it appears constantly.

| Mechanism | Nature | Guarantees behaviour? |
|---|---|---|
| **Prompt / CLAUDE.md / rules** | Guidance the model *reads* | ❌ No — model *may* follow |
| **Hooks** | Deterministic code on lifecycle events | ✅ Yes — runs regardless of model |
| **Permissions (allow/deny)** | Access control enforced by the harness | ✅ Yes |
| **Tool allowlist** | Capability removed entirely | ✅ Yes — can't do what it can't call |

> **The rule:** if a requirement is **"must never" / "always" / "guaranteed" / "for compliance,"** the answer is **enforcement** — a **hook**, a **permission deny**, or **removing the capability**. If it's a *preference* or *style*, guidance (prompt/CLAUDE.md/rule) is fine. Official docs are explicit: *"rules are guidance Claude reads, not configuration Claude Code enforces. For guaranteed behaviour use hooks or permissions."*

Canonical mappings:
- "Block commits of secrets" → **PreToolUse hook** that inspects and blocks (not "tell it not to in CLAUDE.md").
- "This agent must never write files" → **omit Write/Edit from its tool allowlist** (not "instruct it not to").
- "Never run destructive commands without review" → **permission `ask`/`deny`** (not a prose warning).
- "Always format after editing" → **PostToolUse hook** running the formatter.

---

## 6. Escalation, confidence & human-in-the-loop

Reliable agentic design means knowing **when the agent should stop and hand off**:

- **Escalation triggers:** low confidence, repeated tool failures, hitting an iteration/budget cap, an action outside policy, or ambiguity in the request → **escalate to a human or a higher-tier model**, don't let the loop grind or guess.
- **Confidence calibration:** an agent should **signal uncertainty** rather than fabricate. Design for "ask / escalate when unsure," and treat **self-reported confidence as weak** — prefer **verification** (a validation step, a check against ground truth) over trusting the model's own "I'm confident."
- **Human-in-the-loop checkpoints:** insert approval gates before **irreversible or high-impact** actions (sending money, deleting data, emailing customers). This is the *precise* answer whenever a scenario involves real-world side effects.

> **Trap:** "the agent should double-check its own work by asking itself if it's sure" is the **softer** answer. The **precise** answer is an **independent verification step** (evaluator, validator, external check) — self-report doesn't catch the errors the model is confident about.

---

## 7. Anti-patterns & exam traps

- **Reaching for an agent when a workflow fits.** Fixed steps → workflow. Over-powered = wrong.
- **Multi-agent for a single-agent job.** Coordination overhead without benefit.
- **Assuming sub-agents inherit context.** They don't — pass it in the delegation prompt.
- **Omitting `tools:` and calling it "secure."** Omitting inherits everything; least privilege needs an **allowlist**.
- **Fixing a potential infinite loop with a bigger timeout.** Bound it with a **max-iteration/turn cap** + escalation.
- **Treating `pause_turn` as an error to retry.** It's a continuation signal — resend unchanged.
- **Using a prompt/CLAUDE.md to guarantee a hard rule.** Guarantees need **hooks/permissions**.
- **Trusting model self-confidence for reliability.** Use **independent verification**.

---

## 8. Quick-fire revision

- **Control of the flow:** code = workflow, model = agent. Both are "agentic systems."
- **Least power that works.** Simpler architecture is usually the right answer.
- **"Steps unknown ahead of time"** → agent / orchestrator–workers.
- **Known categories** → routing. **Known split** → parallelisation. **Dynamic split** → orchestrator–workers.
- **Loop ends on `end_turn`** (any non-`tool_use` stop_reason). Guard with iteration cap + escalation.
- **`pause_turn`** → resend unchanged, same tools. Not an error.
- **Sub-agent = own context window**, doesn't see main conversation, returns **only a summary**.
- **Pass rules/constraints into the delegation prompt** — sub-agents don't inherit them.
- **`tools` omitted = inherits all; allowlist = only those (no MCP).** Least privilege → allowlist.
- **`disallowedTools` applied first**, then `tools`.
- **Fork** inherits parent context; normal sub-agent starts fresh.
- **Guarantee a behaviour → hook / permission / remove capability.** Prompt & rules only *guide*.
- **Irreversible action → human-in-the-loop gate.** Uncertainty → **escalate / verify**, don't self-report.
