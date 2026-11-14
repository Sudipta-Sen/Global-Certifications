# CCA-F Revision Note — Domain 2: Claude Code Configuration & Workflows

> **Exam weight: ~20%.** Huge jump from the Developer exam (where Claude Code was ~3%). The Architect exam treats Claude Code as a **team platform** you configure, govern, and run in CI — not a personal CLI. Questions are about **the `.claude` hierarchy, precedence, enforcement, and non-interactive runs**.

> **Source note (verified):** Grounded in official Claude Code docs — the **`.claude` directory** (code.claude.com/docs/en/claude-directory), **Memory/CLAUDE.md**, **settings precedence**, **Subagents**, and **hooks**. The array-merge vs scalar-most-specific rule and the managed-settings-wins rule are quoted from those pages.

---

## Contents
1. [The `.claude` directory map](#1-the-claude-directory-map)
2. [CLAUDE.md & the memory hierarchy](#2-claudemd--the-memory-hierarchy)
3. [settings.json & the precedence ladder](#3-settingsjson--the-precedence-ladder)
   - [3.1 Array merge vs scalar most-specific](#31-array-merge-vs-scalar-most-specific)
   - [3.2 Permission precedence (deny > ask > allow)](#32-permission-precedence-deny--ask--allow)
4. [Rules, skills, commands, output-styles](#4-rules-skills-commands-output-styles)
5. [Hooks — deterministic enforcement](#5-hooks--deterministic-enforcement)
6. [Plan mode & permission modes](#6-plan-mode--permission-modes)
7. [MCP server scoping (project vs user vs local)](#7-mcp-server-scoping-project-vs-user-vs-local)
8. [Non-interactive / CI runs](#8-non-interactive--ci-runs)
9. [Quick-fire revision](#9-quick-fire-revision)

---

## 1. The `.claude` directory map

Claude Code reads config from the **project's `.claude/`** (committed, team-shared) and **`~/.claude/`** (personal, every project). Verified contents:

| Path | Purpose | Scope |
|---|---|---|
| `CLAUDE.md` | Instructions loaded every session | Project + global |
| `settings.json` | Permissions, hooks, env vars, model defaults | Project + global |
| `settings.local.json` | Personal project overrides (gitignored) | Project only |
| `rules/*.md` | Topic-scoped instructions, optionally path-gated | Project + global |
| `skills/<name>/SKILL.md` | Prompts invoked with `/name` or auto-invoked | Project + global |
| `commands/*.md` | Single-file slash-command prompts (skills preferred for new work) | Project + global |
| `agents/*.md` | Sub-agents with own prompt + tools | Project + global |
| `output-styles/*.md` | Custom instruction sets that change how Claude responds | Project + global |
| `workflows/*.js` | Dynamic workflow scripts coordinating sub-agents | Project + global |
| `.mcp.json` (project **root**) | Team-shared MCP servers | Project only |

> **Hooks are NOT a separate file** — they live in `settings.json` under the `hooks` key. (Common distractor: "put a hook in `.claude/hooks/`." The *scripts* can live anywhere; the *hook config* is in `settings.json`.)

> **Commit what the team shares.** `.claude/` (CLAUDE.md, settings.json, rules, agents, commands, `.mcp.json`) is meant to be **committed** so the whole team inherits the same governance. `settings.local.json` and `CLAUDE.local.md` are **personal/gitignored**.

---

## 2. CLAUDE.md & the memory hierarchy

CLAUDE.md is **persistent instruction memory** loaded at session start. Key architect facts:

- Lives at **project root** or **`.claude/CLAUDE.md`**; `~/.claude/CLAUDE.md` loads in **every** project.
- **Both global and project CLAUDE.md are loaded into context together — they are NOT merged key-by-key.** On conflict, **project-level instructions win**.
- `CLAUDE.local.md` = private, manually created, gitignored project memory, loaded alongside.
- Claude Code can also read an existing **`AGENTS.md`**.
- **Split large CLAUDE.md into `rules/`** once it approaches ~200 lines — keeps context lean.

> **CLAUDE.md vs settings.json — don't confuse them.** CLAUDE.md is **guidance the model reads** (both files in context, project wins on conflict). settings.json is **configuration the harness enforces** (array-merge / scalar-most-specific precedence). "It's advisory" = CLAUDE.md/rules. "It's enforced" = settings.json / hooks / permissions.

---

## 3. settings.json & the precedence ladder

Verified order, **highest precedence first**:

1. **Managed settings (enterprise)** — *"take precedence over everything,"* apart from documented exceptions.
2. **CLI flags** (`--permission-mode`, `--settings`) — override `settings.json` for that session.
3. **`settings.local.json`** — highest of the user-editable files; CLI flags & managed still win.
4. **Project `settings.json`**.
5. **Global `~/.claude/settings.json`** — lowest.

> **Enterprise/managed always wins.** When a scenario says "the organisation must guarantee X across all developers," the answer is **managed (enterprise) settings** — a developer's local settings cannot override them.

### 3.1 Array merge vs scalar most-specific

> **The rule to memorise (quoted):** *"Array settings like `permissions.allow` combine across all scopes; scalar settings like `model` use the most specific value."*

- **Arrays** (permission lists) → **union/merge** across every scope.
- **Scalars** (e.g., `model`) → the **most specific scope's single value** wins.

### 3.2 Permission precedence (deny > ask > allow)

Within the merged permission set, evaluation is **deny > ask > allow**: a **deny from *any* scope wins**, regardless of an allow elsewhere. So allowlists merge across scopes, but a single deny anywhere blocks.

> **Architect enforcement pattern:** to hard-stop a capability org-wide, put a **`deny`** in **managed settings** — it merges in and beats any local `allow`.

---

## 4. Rules, skills, commands, output-styles

- **`rules/*.md`** — topic-split instructions. A rule **without** `paths:` frontmatter loads at session start like CLAUDE.md; a rule **with** `paths:` loads **only when Claude reads a matching file** (conditional, path-gated). Subdirectories auto-discovered (`rules/frontend/react.md`). **Rules are guidance, not enforcement** — for guaranteed behaviour use hooks/permissions.
- **Skills** (`skills/<name>/SKILL.md`) — reusable prompt packages, `/name`-invoked or auto-invoked by description. Preferred over commands for new work.
- **Commands** (`commands/*.md`) — single-file slash prompts; legacy relative to skills.
- **Output-styles** — change *how* Claude responds (tone/format), as swappable instruction sets.

> **Path-gated rules are the "context efficiency" answer:** when a scenario wants framework-specific guidance to apply **only** when touching those files (and not bloat every session's context), the answer is a **`rules/*.md` with `paths:`**, not stuffing it all into CLAUDE.md.

---

## 5. Hooks — deterministic enforcement

Hooks run **your code deterministically** on lifecycle events — the enforcement backbone.

| Hook event | Fires… | Can it block? |
|---|---|---|
| **PreToolUse** | **before** a tool runs | ✅ Yes — block the action before it happens |
| **PostToolUse** | **after** a tool runs | Reacts/validates after (can surface errors) |
| **Stop** | when Claude tries to finish | ✅ Can **force continue** |
| UserPromptSubmit, SessionStart, etc. | various | — |

- **PreToolUse** is where you **prevent** (block secret commits, block destructive commands). **PostToolUse** is where you **react** (auto-format, run tests, log).
- **Stop hook force-continue:** return **`decision: "block"`** + a **`reason`**, and check **`stop_hook_active`** to avoid an infinite loop.
- **Hook matchers are exact-string** against tool names. For MCP tools (named `mcp__<server>__<tool>`), match a whole server with the wildcard **`mcp__<server>__*`**.

> **"Guidance vs guarantee" again:** CLAUDE.md/rules *ask*; **hooks enforce**. Any "must/never/for compliance" requirement → hook (or permission), never prose.

---

## 6. Plan mode & permission modes

**Permission modes** (set via `--permission-mode`, or per-sub-agent `permissionMode`):

- **`default`** — asks before sensitive actions.
- **`plan`** — **read-only planning**: Claude investigates and proposes a plan **without making changes**. Use when you want analysis/design before any edits.
- **`acceptEdits`** — auto-accepts file edits.
- **`dontAsk`** / auto — proceeds without prompts.
- **`bypassPermissions`** — no permission checks (dangerous; CI/sandbox only).

> **Sub-agent interaction (verified):** a sub-agent's `permissionMode` applies **only when the main session is in `default` or `dontAsk`**. If the main session runs in `bypassPermissions`/`acceptEdits`/auto, the **sub-agent runs in that mode too** — you can't tighten a sub-agent below a loosened parent via `permissionMode` alone.

> **Plan mode = the "investigate before acting" answer.** When a scenario wants Claude to analyse a large change and present an approach for human approval **before touching anything**, that's **plan mode**, not "tell it to be careful."

---

## 7. MCP server scoping (project vs user vs local)

MCP servers can be configured at different scopes:

- **Project scope** — `.mcp.json` at the **project root**, **committed**, **team-shared**. Everyone on the repo gets the same servers.
- **User scope** — `~/.claude` config, **personal**, applies across all your projects.
- **Local/`settings.local.json`** — personal to you **in this project**, gitignored.

> **Architect scoping decisions:**
> - "The whole team needs this MCP server" → **project `.mcp.json`** (committed).
> - "Only I use this, everywhere" → **user scope**.
> - "Only I use this, only here, and it shouldn't be committed" → **local**.
> - **Secrets/credentials for an MCP server** never go in the committed `.mcp.json` — reference **env vars**, keep the secret out of version control.

Sub-agents can also be given a **scoped `mcpServers`** set in frontmatter, so a sub-agent only sees the servers its role needs (least privilege at the MCP layer).

---

## 8. Non-interactive / CI runs

The Architect exam expects you to run Claude Code **head-less** in automation:

- **Non-interactive / print mode** (`-p` / `--print`) runs a single prompt and exits — the building block for CI pipelines, pre-commit checks, and scripted workflows.
- In CI you commit `.claude/` config so the agent inherits the same **rules, hooks, permissions, and sub-agents** the team uses.
- For an **isolated/sandboxed** automation agent (Agent SDK), lock it down: **`settingSources: []`** (don't auto-load filesystem settings), an explicit **`systemPrompt`**, and **`CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`** in `env` so it doesn't silently pick up CLAUDE.md. This gives a **reproducible, governed** agent rather than one shaped by whatever happens to be on disk.
- Enforcement that must hold in CI (secret-blocking, forbidden commands) belongs in **hooks/permissions**, because there's **no human to approve prompts** in a headless run.

> **"No human in the loop" is the CI tell.** In non-interactive runs, `ask` prompts have nobody to answer them — so governance must be **deterministic** (hooks, deny permissions, allowlisted tools), and risky modes like `bypassPermissions` only belong in a **sandboxed** environment.

---

## 9. Quick-fire revision

- **`.claude/` project dir = committed, team-shared governance.** `settings.local.json` / `CLAUDE.local.md` = personal, gitignored.
- **Hooks live in `settings.json`** (not a `hooks/` config file).
- **CLAUDE.md:** global + project **both loaded**, **not** key-merged; **project wins** on conflict.
- **Settings precedence:** managed(enterprise) > CLI flags > settings.local.json > project settings.json > global.
- **Arrays merge across scopes; scalars take the most specific.** `permissions.allow` combines; `model` picks most specific.
- **Permissions: deny > ask > allow** — a deny from any scope wins.
- **Org-wide guarantee → managed settings + deny.**
- **Rules:** `paths:`-gated = conditional, context-efficient; no `paths:` = always on. **Rules guide, hooks/permissions enforce.**
- **PreToolUse blocks before; PostToolUse reacts after.** Stop hook: `decision:"block"` + `reason` + `stop_hook_active`.
- **MCP matcher wildcard:** `mcp__<server>__*`.
- **Plan mode = read-only investigate-before-acting.**
- **Project `.mcp.json` = team MCP servers (committed), secrets via env vars, not in the file.**
- **CI = headless (`-p`)**, no human → enforce with hooks/permissions; isolate SDK agents with `settingSources: []` + explicit `systemPrompt` + `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`.
