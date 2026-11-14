# CCDV-F Revision Note — Domain 3: Claude Code

> **Exam weight: ~3.1%** (light). Few questions — aim for quick, confident recall. Don't over-invest.

---

## Contents
1. [What Claude Code is](#1-what-claude-code-is)
2. [`CLAUDE.md` — project memory](#2-claudemd--project-memory-most-tested-concept)
   - [Memory hierarchy (load order)](#memory-hierarchy-load-order-broadest--most-specific-most-specific-is-read-last)
   - [Keep `CLAUDE.md` modular with `@path` imports](#keep-claudemd-modular-with-path-imports)
3. [Path-specific rules (`.claude/rules/`)](#3-path-specific-rules-clauderules)
4. [Hooks](#4-hooks)
5. [Slash commands](#5-slash-commands)
6. [Claude Code configuration](#6-claude-code-configuration)
   - [Settings, permissions & servers](#settings-permissions--servers)
   - [Plan mode vs direct execution](#plan-mode-vs-direct-execution)
   - [When to use plan mode](#when-to-use-plan-mode)
   - [Plan content — where plans go & plan-mode settings](#plan-content--where-plans-go--plan-mode-settings)
   - [CI/CD integration — non-interactive (`-p`) mode](#cicd-integration--non-interactive--p-mode)
7. [Skills & how they load](#7-skills--how-they-load)
8. [`CLAUDE.md` vs `SKILL.md` vs `/command`](#8-claudemd-vs-skillmd-vs-command)
   - [Worked example — a repo with 10 pre-merge checks](#worked-example--a-repo-with-10-pre-merge-checks)
9. [Subagents & context fork — context fork with respect to skills](#9-subagents--context-fork--context-fork-with-respect-to-skills)
10. [Common traps](#10-common-traps)
11. [Quick-fire revision](#11-quick-fire-revision)

---

## 1. What Claude Code is
Claude Code is Anthropic's **agentic command-line tool** for software engineering: it reads your codebase, edits files, runs commands/tests, and uses tools — driven by natural language in the terminal (and via IDE integrations). It is itself an agent built on the Claude models.

### Plain-language breakdown
- **Agentic CLI for coding** — a *CLI* is a tool you run by typing in your terminal (not a point-and-click app). *Agentic* means it **takes actions** to finish a job, not just answers questions. So it's a terminal coding assistant with "hands."
- **Reads your repo** — a *repo* is your project's code folder; it scans your files to understand the project first.
- **Edits files** — it actually changes your code (fix a bug, add a feature) and writes the changes to disk.
- **Runs commands/tests** — it executes things in your terminal, e.g. running your test suite to verify its own work, installing packages, starting the app.
- **Uses tools** — it has extra capabilities (search, file ops, etc.) to get tasks done.
- **Connects MCP servers** — via **MCP** (Model Context Protocol — your Domain 8 topic) it plugs into *external* systems (a database, GitHub, a company tool) for extra abilities.
- **Driven by natural language, terminal + IDE** — you tell it what you want in plain English ("fix the login bug"); it also works inside editors like VS Code, not only the bare terminal.

**Analogy:** a normal chatbot tells you *how* to fix your car over the phone; Claude Code is handed the keys and toolbox — it opens the hood (reads repo), replaces the part (edits files), starts the engine to check (runs tests), and grabs a special tool from the garage if needed (MCP).

---

## 2. `CLAUDE.md` — project memory (most-tested concept)
- A special Markdown file that Claude Code **loads automatically** as standing context for the project.
- Put in it: project conventions, architecture notes, key commands (build/test/lint), coding style, "do/don't" rules, and anything you'd otherwise repeat every session.
- Think of it as persistent, reusable instructions so you don't re-explain the repo each time.

### Memory hierarchy (load order: broadest → most specific; most specific is read LAST)
```
1. Managed policy  (org/IT)         → org-wide rules; highest authority, loads first
2. User            ~/.claude/CLAUDE.md→ your personal prefs across all projects
3. Project root    ./CLAUDE.md        → team-shared, committed to the repo
4. Project local   ./CLAUDE.local.md  → your personal, this-project-only, .gitignored
5. Subfolder       <dir>/CLAUDE.md     → loaded on demand when working in that subtree
```
- **`./CLAUDE.md` (project root)** — team-shared. Committed to source control so everyone on the project gets it. Holds project architecture, coding standards, common workflows. (Can also live at `./.claude/CLAUDE.md`.)
- **`./CLAUDE.local.md` (project local)** — personal, this project only. You add it to `.gitignore` so it's not shared. Holds your own sandbox URLs, preferred test data, personal tweaks for this one repo. It loads alongside `CLAUDE.md` and is treated the same way — just private to you.
- **`<dir>/CLAUDE.md` (subfolder)** — area-specific. Written for one part of the tree (e.g. `services/api/CLAUDE.md`) and committed like the root file, or gitignored if it's just for you. It doesn't load at launch; Claude pulls it in on demand, only once it reads, writes, or edits a file in that subdirectory — read last, so it's the most specific.

**How conflicts resolve (nuance):** files are **concatenated into context, not hard-overridden** — Claude treats them as context, not enforced config. More-specific/later-loaded text (local after root; subfolder last) **tends** to win, but when two conflict Claude *may follow either*, so keep them consistent. **Managed policy** is the real top authority; to truly *enforce* something, use a **PreToolUse hook**, not `CLAUDE.md`.

### Keep `CLAUDE.md` modular with `@path` imports
- Inside a `CLAUDE.md`, write **`@path/to/file.md`** to pull another file's contents into context — split one long file into topic files (e.g. `@docs/testing.md`, `@docs/api-design.md`) instead of one giant file.
- **Relative paths resolve relative to the importing file** (not the working directory); absolute paths and home paths (`@~/.claude/my-instructions.md`) also work.
- Imports can nest **recursively, up to 4 hops** deep.
- **Caveat:** imports help *organise* but **don't reduce context cost** — imported files load together with the file that imports them. To actually trim context, use path-scoped `.claude/rules/` (loaded only when Claude touches matching files) — see §3.
- Import parsing **skips code** — wrap a path in backticks (`` `@README` ``) to mention it literally without importing.
- A project file importing a file **outside** the working directory (e.g. a home-dir path) triggers a **one-time external-import approval**. This is the recommended way to **share personal instructions across git worktrees** (since a gitignored `CLAUDE.local.md` lives in only one worktree). You can also pull in `@AGENTS.md` to reuse another tool's instructions.

---

## 3. Path-specific rules (`.claude/rules/`)
For larger projects, break instructions into **topic files** under **`.claude/rules/`** instead of one big `CLAUDE.md`. Each file covers one topic and is discovered **recursively**, so you can group them into subdirectories:
```
your-project/
├── .claude/
│   ├── CLAUDE.md           # Main project instructions
│   └── rules/
│       ├── code-style.md   # Code style guidelines
│       ├── testing.md      # Testing conventions
│       └── security.md     # Security requirements
```

### Two load behaviours — decided by the `paths` frontmatter
- **No `paths` field → always on.** Loaded at **launch, every session**, with the **same priority as `.claude/CLAUDE.md`**. Applies to all files. (Just a modular way to split a long `CLAUDE.md`.)
- **With a `paths` field → path-scoped / conditional.** The rule loads **only when Claude works with a matching file** — i.e. uses the **Read / Write / Edit** tool on it, **or** views it with a Bash command that **"counts as a read."** **This is the real context saver:** the rule stays out of context until it's relevant.
  - **Commands that count as a read** (each on **one file**, **no pipes/redirects**): `cat`, `nl`, `bat`, `batcat`, `head`, `tail`, `sed -n 'X,Yp'`, **`grep`**, `egrep`, `fgrep`, `rg`. So a matching **`grep`/`rg` search triggers the rule too** — but a **search that prints no matches does *not* count** (nothing was actually read), and anything outside this list doesn't count.

*Example* — a rule scoped to the API layer, `.claude/rules/api.md`:
````markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules
- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments
````
This text enters context **only** when Claude touches a file like `src/api/users.ts`.

### The `paths` field
- **`paths` is the *only* field Claude Code reads from a rule** — any other frontmatter is ignored silently. It accepts a **YAML list or a comma-separated string**. (If the YAML doesn't parse, the rule loads as if it had no `paths`.)
- ⚠️ **Scope is by file *path* only — not by file *content*.** `paths` matches **filename/path globs**. There is **no** way to trigger a rule based on a **pattern *inside* the file** (e.g. "load this rule whenever a file contains `TODO` or imports `express`"). Content-pattern scoping is an **open feature request, not a shipped feature** — treat "scope a rule by file content" as a wrong answer.
  - **There is no `grep:` frontmatter field.** Writing something like `grep: console\.log\(` in a rule does **nothing useful** — `grep:` isn't a field, so it's **ignored**, and (with no valid `paths`) the rule then loads **unconditionally, every session, for all files** — the opposite of scoping it. You *cannot* say "apply only to files that contain `console.log(`."
  - **Want to react to file *content* (e.g. block/flag `console.log(`)?** Use a **`PreToolUse` hook** (§4) — a hook can actually `grep` the content and allow/block — not a rule. *Rules = path globs; content matching = hooks.*
- **Glob patterns:**

  | Pattern | Matches |
  |---|---|
  | `**/*.ts` | All TypeScript files in any directory |
  | `src/**/*` | All files under `src/` |
  | `*.md` | Markdown files in the project root |
  | `src/components/*.tsx` | React components in one directory |

- **Multiple patterns + brace expansion** in one rule:
  ````markdown
  ---
  paths:
    - "src/**/*.{ts,tsx}"
    - "lib/**/*.ts"
    - "tests/**/*.test.ts"
  ---
  ````
- **Base directory** each pattern is relative to: for **project rules**, the directory that contains the `.claude` folder; for **user-level rules**, the session's primary working directory.

### User-level rules (`~/.claude/rules/`)
- Live in `~/.claude/rules/` and apply to **every project on your machine** — personal preferences/workflows.
- Loaded **before** project rules (so a project rule appears **later** in context). **Neither overrides the other** — if a user rule and a project rule conflict, Claude may follow either, so keep them consistent (same conflict rule as `CLAUDE.md`).

### Merge order — when several rules (and `CLAUDE.md` files) apply at once
**Golden principle: everything is *concatenated* into context, not hard-overridden.** More-specific / later-loaded text **tends** to win, but when two pieces genuinely conflict Claude **may follow either**, so keep them consistent. (Only **managed policy** is a real top authority; to *enforce* something, use a **hook** / managed settings, not memory files.)

**Load order — broadest first, most specific read LAST (last tends to win):**
```
1. Managed policy CLAUDE.md        (org/IT)      ← highest authority, first
2. User scope      ~/.claude/CLAUDE.md
                 + ~/.claude/rules/*.md          ← user rules load here (BEFORE project)
3. Project scope   ./CLAUDE.md  (or ./.claude/CLAUDE.md)
                 + .claude/rules/*.md  WITHOUT paths  ← same priority as project CLAUDE.md
4. Local           ./CLAUDE.local.md              (appended after CLAUDE.md at that level)
5. Path-scoped rules + subfolder CLAUDE.md        ← ON DEMAND when a matching file is touched → latest = most specific
```

**Key ordering facts (exam-relevant):**
- A **rule *without* `paths`** loads at launch with the **same priority as `.claude/CLAUDE.md`** (project level).
- **User-level rules load *before* project rules** → a project rule appears **later** in context. **Neither overrides the other** — on conflict Claude may follow either; keep them consistent.
- In the directory tree, files are ordered **root → down to the working dir** (so `foo/CLAUDE.md` before `foo/bar/CLAUDE.md`), and within a directory **`CLAUDE.local.md` is appended *after* `CLAUDE.md`** (your personal notes read last at that level).
- **Path-scoped rules and subfolder `CLAUDE.md`s load on demand** when their file is touched, so they land **latest** of all — the most specific context for the file in hand.
- The docs **don't guarantee an order *among* same-scope rules** (e.g. two project `.claude/rules/*.md`), so **don't rely on one rule overriding another** — keep rules non-conflicting. Audit with `/doctor prompt-audit`; see what loaded with `/context`.

### Rules vs CLAUDE.md vs skills (when to use which)
- **Rule without `paths`** ≈ a modular slice of `CLAUDE.md` — always loaded.
- **Path-scoped rule** → loads **only for matching files**; use it to move "only matters for part of the codebase" instructions out of the always-on `CLAUDE.md` and cut context cost.
- **Skill** → loads **only when invoked or judged relevant**; use it for task-specific, multi-step *procedures* (not file-type guidance).

---

## 4. Hooks
- **Hooks** = your own shell commands that run **automatically on lifecycle events**.
- Common events: **`PreToolUse`** (before a tool runs — can block/allow), **`PostToolUse`** (after), plus notification/stop-type events.
- Uses: auto-format or lint after edits, run tests, log actions, enforce guardrails (e.g. block edits to protected files).
- Mental model: deterministic automation around Claude's non-deterministic actions.

**Config shape** (in `settings.json`) — three levels: **event → matcher group → handler(s)**:
```
"hooks": { "<EventName>": [ { "matcher": "<tool filter>", "hooks": [ { "type": "command", "command": "<shell cmd>" } ] } ] }
```
- **`matcher`** filters which tool fires it: `"*"`/`""`/omitted = all; exact names or `|`-lists (`Edit|Write`); anything else is a **regex** (`mcp__memory__.*`). Non-tool events (e.g. `Stop`) ignore it.

*Example A — auto-format after every edit* (`PostToolUse`):
````json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [ { "type": "command", "command": "npm run format" } ] }
    ]
  }
}
````

*Example B — block `rm -rf` before it runs* (`PreToolUse`). Config + script:
````json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash",
        "hooks": [ { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh" } ] }
    ]
  }
}
````
````bash
#!/bin/bash
# .claude/hooks/block-rm.sh — receives the tool call as JSON on stdin
COMMAND=$(jq -r '.tool_input.command')
if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{ hookSpecificOutput: {
    hookEventName: "PreToolUse",
    permissionDecision: "deny",
    permissionDecisionReason: "Destructive command blocked by hook" } }'
else
  exit 0   # no decision → normal permission flow
fi
````

**How a hook decides (two ways — pick one):**
- **Exit code:** `0` = proceed (any JSON it printed still applies) · **`2` = block** (on events that can block; stderr is sent to Claude as the reason) · any other code = non-blocking error, action proceeds.
- **JSON on stdout (exit 0):** for **`PreToolUse`** use `hookSpecificOutput.permissionDecision` = `allow` / `deny` / `ask` / `defer`. For `UserPromptSubmit` / `PostToolUse` / `Stop`, use top-level **`decision: "block"`** (omit it to allow).
- ⚠️ Exit 2 **always blocks**, even if the JSON says `allow`.
- Handler `type` can also be `http`, `mcp_tool`, `prompt`, or `agent` — but **`command`** is the one to know.

---

## 5. Slash commands
- **What:** `/`-prefixed commands typed in the Claude Code prompt — either to control the session or to run a reusable prompt.
- **Built-in examples:** `/help`, `/clear` (reset context), `/compact` (summarise & shrink context), `/init` (scaffold a `CLAUDE.md`), `/memory` (view/edit memory files), `/context` (see what loaded), `/config`, `/review`, `/model`, `/agents`, `/mcp`, `/permissions`, `/doctor`, `/status`, `/usage`.
- **Custom commands** — create a Markdown file; the **filename (no extension) = the command name**:
  - *Project (team-shared):* `.claude/commands/<name>.md` → `/<name>`, committed to git.
  - *Personal (all your projects):* `~/.claude/commands/<name>.md`.
  - *Note:* custom commands are now **unified with skills** — `.claude/skills/<name>/SKILL.md` also creates `/<name>`; on a name clash the **skill wins**. Existing `commands/` files keep working.
- **Namespacing:** a subdirectory becomes a prefix — `.claude/commands/frontend/component.md` → `/frontend:component`.
- **Arguments:** `$ARGUMENTS` = everything passed; positional args are **0-based** — `$0` (first), `$1`, `$2` … (same as `$ARGUMENTS[0]`, `$ARGUMENTS[1]` …); or name them via the `arguments` frontmatter field. Quote multi-word values: `/cmd "hello world" second` makes `$0` = `hello world`.

  *Example* — `.claude/commands/open-pr.md` (invoked as `/open-pr 142 feature/login`):
  ````markdown
  ---
  description: Open a PR for an issue
  argument-hint: [issue] [branch]
  arguments: [issue, branch]
  ---

  Create a pull request for issue #$issue from branch "$branch".
  ````
  Named: `$issue` = `142`, `$branch` = `feature/login`. The positional equivalents are `$0` = `142` and `$1` = `feature/login`, and `$ARGUMENTS` = the whole string `142 feature/login`.
- **Frontmatter fields:** `description`, `argument-hint` (autocomplete hint), `allowed-tools` (tools usable without a permission prompt), `model` (override for this command).

  *Example* — command frontmatter with an **allowlist** of tools:
  ````markdown
  ---
  description: Commit staged changes
  argument-hint: [message]
  allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
  disallowed-tools: Bash(git push *)
  model: claude-haiku-4-5
  ---
  ````
  `allowed-tools` lets those tools run **without a permission prompt**; `disallowed-tools` **blocks** tools for that command/skill. For broader, standing control across the whole session, use the **permissions layer** in `settings.json` (or the `--allowedTools` / `--disallowedTools` CLI flags for one run):
  ````json
  {
    "permissions": {
      "allow": ["Bash(git diff:*)", "Read(./src/**)"],
      "deny":  ["Bash(git push:*)", "Read(./.env)", "WebFetch"],
      "ask":   ["Bash(rm:*)"]
    }
  }
  ````
  Here `allow` = run without asking, `deny` = never allowed (blocked), `ask` = always prompt first.
- **Dynamic content:** `` !`<bash>` `` runs a shell command and injects its output into the prompt; `@path` pulls in a file's contents.

  *Example* — `.claude/commands/commit.md` (invoked as `/commit fix`):
  ````markdown
  ---
  description: Draft a commit message from staged changes
  argument-hint: [scope]
  allowed-tools: Bash(git diff *)
  ---

  ## Staged changes
  !`git diff --cached`

  Write a concise commit message for the "$1" scope based on the changes above,
  following the conventions in @.github/COMMIT_CONVENTION.md
  ````
  Here `` !`git diff --cached` `` injects the live diff, `@.github/COMMIT_CONVENTION.md` pulls in that file's contents, and `$1` becomes `fix`.
- **MCP prompts** exposed by a connected server also surface as slash commands (ties into Domain 8).

---

## 6. Claude Code configuration
How you set up and steer a session: the settings files and permissions, the plan-vs-execute behaviour, and where plans are written.

### Settings, permissions & servers
- **`settings.json`** — configuration (permissions, allowed tools, environment). Can be project- or user-scoped.
- **Permissions model** — tools/actions can require approval; you can allow-list safe commands (`allow` / `deny` / `ask` — see §5).
- **MCP servers** — Claude Code can connect to MCP servers to gain extra tools/data (ties into Domain 8).
- **Subagents** — Claude Code can spawn focused subagents for delegated tasks (see §8).

**Where it lives (scope):** `~/.claude/settings.json` (user, all projects) · `.claude/settings.json` (project, committed/shared) · `.claude/settings.local.json` (project, personal — gitignored) · managed/enterprise (highest authority).

*Example* — a project `.claude/settings.json` (strict JSON — **no comments allowed** in the real file):
````json
{
  "model": "claude-sonnet-5",
  "permissions": {
    "allow": ["Bash(npm run test:*)", "Read(./src/**)"],
    "ask":   ["Bash(git push:*)"],
    "deny":  ["Read(./.env)", "Read(./secrets/**)", "WebFetch"],
    "defaultMode": "plan"
  },
  "env": {
    "NODE_ENV": "development"
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npm run format" }
        ]
      }
    ]
  },
  "enableAllProjectMcpServers": false,
  "cleanupPeriodDays": 30,
  "attribution": { "commit": false },
  "statusLine": { "type": "command", "command": "echo \"$(git branch --show-current)\"" }
}
````
What each key does:
- **`model`** — the model every new session starts with (`--model` / `ANTHROPIC_MODEL` override it per-session).
- **`permissions`** — `allow` (run without a prompt), `ask` (always prompt), `deny` (block outright — including reads of secret files), `defaultMode` (which permission mode new sessions start in, e.g. `plan`).
- **`env`** — environment variables set for every session and its subprocesses.
- **`hooks`** — shell commands run on lifecycle events; each entry has a `matcher` (which tool) + a list of `{type:"command", command:"…"}` (see §4).
- **`enableAllProjectMcpServers`** — approve every server in project `.mcp.json` files without a prompt.
- **`cleanupPeriodDays`** — how long transcripts are kept before deletion.
- **`attribution`** — controls the commit trailer (`attribution.commit`) / PR line (`attribution.pr`); replaces the deprecated `includeCoAuthoredBy`.
- **`statusLine`** — a command that renders the status line under the prompt.

> **Precedence (highest → lowest):** managed/enterprise → CLI flags → `.claude/settings.local.json` → `.claude/settings.json` (project) → `~/.claude/settings.json` (user).

> **⚠️ That order is for *scalar* settings — permissions work differently (high-value trap).** Permission **rules don't override; they *merge* (union) across every scope.** Then **rule *type* decides, not the file**: evaluation is **`deny` → `ask` → `allow`**, with **`deny` checked first and winning from *any* scope.** So an **`allow` in `settings.local.json` canNOT cancel a `deny` in the committed `.claude/settings.json`** — both survive the merge and `deny` wins → the command is **blocked, no prompt** (it does *not* "resolve to ask"). This is deliberate: a committed `deny` is a guardrail that an individual's local `allow` must never be able to defeat. File precedence only settles a single-valued key (e.g. `model`); it never lets `allow` beat `deny`.

### Plan mode vs direct execution
Claude Code always runs in a **permission mode** — the setting that decides what it can do **without asking you first**. The exam-relevant split: **plan mode = research & propose, make *no* changes**; the other modes = **direct execution** (Claude edits/runs, with or without prompts).

#### Plan mode — "research and propose changes without making them"
- Claude **reads files, runs shell commands to explore, and writes a plan** — but **does *not* edit your source.** Edits stay blocked until **you approve the plan**.
- Purpose: explore a codebase / design the approach **before** anything changes — safe, reversible thinking step.
- **Enter it** three ways: press **`Shift+Tab`** to cycle modes, launch `claude --permission-mode plan`, or prefix a single prompt with `/plan`. The status bar shows `⏸ plan mode on`.
- **Shell during planning:** if **auto mode** is available, a classifier approves/blocks exploratory commands; otherwise commands outside the built-in read-only set prompt for approval.
- **Approve / exit:** when the plan is ready Claude asks how to proceed — **Yes, and use auto mode** · **Yes, manually approve edits** · **No, keep planning**. Approving **exits plan mode and switches the session to the chosen execution mode**, then Claude starts editing. `Shift+Tab` again leaves plan mode *without* approving; `Ctrl+G` opens the plan to edit it first.

#### Direct-execution modes (Claude acts, doesn't just plan)
| Mode | Runs **without** asking | Best for |
|---|---|---|
| **`default`** (Manual) | **Reads only** — prompts for every edit/command | Reviewing each action; sensitive work |
| **`acceptEdits`** | Reads **+ file edits** + common FS commands (`mkdir`, `touch`, `mv`, `cp`…) | Iterating on code you're watching |
| **`plan`** | Reads + classifier-approved exploration; **no edits** | Exploring before changing |
| **`auto`** | **Everything**, with background safety checks by a classifier | Long tasks, less prompt fatigue |
| **`dontAsk`** | Reads in working dirs + pre-approved tools; anything else **denied** | Locked-down CI / scripts |
| **`bypassPermissions`** | **Everything**, no checks | **Isolated containers/VMs only** ⚠️ |

- **`Shift+Tab` cycle:** from `auto` the first press → `default`, then it runs `default → acceptEdits → plan`.
- **Plan vs `default`:** both let Claude *read*, but **`default` can edit/run (prompting for anything beyond reads)** while **plan mode refuses to touch your source until you approve a plan**. That's the core "plan vs direct execution" distinction.
- ⚠️ **`bypassPermissions`** skips *all* checks — docs limit it to isolated environments (containers/VMs) without internet access.

#### When to use plan mode
| Situation | Mode |
|---|---|
| Multi-file migration touching many parts of a codebase at once | **Plan mode** |
| Directory-wide operations (bulk rename, restructuring) | **Plan mode** |
| Any operation with unknown or hard-to-reverse blast radius | **Plan mode** |
| A single-file typo fix or small, easily reversible change | **Direct execution** |
| Non-interactive CI/CD runs (no human present to review a plan) | **Direct execution** with explicit criteria in the prompt (the headless **`-p`** flag) |

> **⚠️ Exam trap:** Plan mode **assumes a human is actually present** to review before execution. A **CI/CD pipeline running non-interactively** (`-p` / headless) **can't meaningfully use plan mode's human-approval gate** — for that context, correctness has to come from **precise upfront criteria in the prompt and code-level gates** (tests/hooks), **not** a plan-review step nobody is there to perform.

### Plan content — where plans go & plan-mode settings
When Claude plans, it writes a **plan document** (the proposed steps/changes) that you review on the plan-accept screen. A few settings control that content and behaviour:
- **`plansDirectory`** — chooses **where plan mode writes its plan files** (the saved plan content). Point it at a folder to keep plans (e.g. version them, or keep them out of the repo).
- **`showClearContextOnPlanAccept`** — adds a **"clear context"** option on the **plan-accept screen**, so you can start the build with a clean context right after approving.
- **`useAutoModeDuringPlan`** — lets the **auto-mode classifier review shell commands during planning**; set **`false`** to be **prompted** for commands outside the read-only set instead.
- **`permissions.defaultMode`** — sets the **permission mode new sessions start in** (e.g. always start in `plan` or `acceptEdits` rather than `default`).
- **Editing the plan:** press **`Ctrl+G`** to open the proposed plan in your editor before approving; approving then switches the session into the execution mode you pick.

### CI/CD integration — non-interactive (`-p`) mode
Run Claude Code **headlessly** (no TUI, no prompts) by adding **`-p` / `--print`** with the prompt. This is how it drops into CI/CD, scripts, and git hooks.

**Core flags for automation:**
- **`-p "…"`** — non-interactive: run the prompt, print the result, exit. **Exit code 0 = success, non-zero = failure** → use it directly as a **CI gate**.
- **`--bare`** — recommended for CI: skips auto-discovery of host hooks/skills/commands/plugins/MCP/`CLAUDE.md` so you get the **same result on every machine**. (Needs `ANTHROPIC_API_KEY` — bare mode doesn't use subscription login.)
- **`--allowedTools "Bash,Read,Edit"`** — pre-approve tools so they run **without a prompt** (permission-rule syntax, e.g. `Bash(git diff *)`).
- **`--permission-mode auto|acceptEdits|dontAsk`** — session baseline; **`dontAsk`** (deny anything that would prompt) suits locked-down CI.
- **`--permission-prompts none`** — unattended runs: anything that would prompt is **denied** (nobody's there to approve) and Claude is told not to retry.
- **`--output-format json`** (or `stream-json`) — structured result for scripts; parse with `jq` (`.result`, `.session_id`, `total_cost_usd`).
- **Pipe stdin:** `cat build-error.txt | claude -p "explain the root cause"` (stdin capped at 10 MB).

**Output formats (`--output-format`):**

| Format | What you get | Use it for |
|---|---|---|
| **`text`** *(default)* | Plain text — just the final response | Quick scripts, piping to a file |
| **`json`** | One JSON object: **`result`** (the text) + metadata — `session_id`, `usage`, **`total_cost_usd`** (+ per-model cost breakdown). With `--json-schema`, the typed data is in **`structured_output`** | Scripts that parse the answer / track cost / enforce a schema |
| **`stream-json`** | **Newline-delimited JSON**, one event per line, in real time; the **last line is a `result` message** with final text + metadata | Streaming UIs; following subagent messages |

- `text` is the default — omit `--output-format` to get it.
- **`json`** → parse with `jq`: `… --output-format json | jq -r '.result'`.
- **`stream-json`** needs **`--verbose`** (and `--include-partial-messages` for token-by-token deltas):
  ````bash
  claude -p "Explain recursion" --output-format stream-json --verbose --include-partial-messages
  ````
- **Schema-constrained output:** `--output-format json --json-schema '{…}'` → read `.structured_output` (check the field exists first — it's absent if Claude didn't produce valid output).

**How schema output is produced — it's "tool-call-shaped" (verified):** when you pass `--json-schema` (SDK: `outputFormat`), Claude Code **injects a synthetic tool called `StructuredOutput`** and the agent **returns its answer by *calling that tool*.** Each attempt in the stream is:
- a **`tool_use`** block named **`StructuredOutput`** whose **`input` is your schema object**, then
- a **`tool_result`** reading *"Structured output provided successfully."*

On a schema mismatch the `tool_result` has **`is_error: true`** and the agent **calls `StructuredOutput` again** — so one query can hold several calls (bounded by a retry limit; exhausting it → `error_max_structured_output_retries`). **Read the validated data from the `structured_output` field on the *result* message, not from the `tool_use` block**, and skip/ignore `StructuredOutput` if you log tool calls. *(Contrast: the **Claude API**'s own structured outputs — `output_config.format` (JSON outputs) and `strict: true` (strict tool use) — are detailed in **Note 06 §2.7**; they constrain a **single-turn** response via constrained decoding, unlike this agent-loop `StructuredOutput` tool call.)*

*Example* — review a PR diff in a script and print just the text result:
````bash
gh pr diff "$1" | claude --bare -p \
  --append-system-prompt "You are a security engineer. Review for vulnerabilities." \
  --allowedTools "Read" \
  --output-format json | jq -r '.result'
````

*Example* — a **GitHub Actions** step that **fails the build** when issues are found:
````yaml
- name: Claude typo/lint check
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
  run: |
    RESULT=$(git diff origin/main | claude --bare -p \
      "You are a typo linter. For each typo in this diff print filename:line then the issue. If there are none, output exactly: PASS" \
      --permission-mode dontAsk \
      --output-format json | jq -r '.result')
    echo "$RESULT"
    [ "$RESULT" = "PASS" ] || exit 1   # non-zero exit → CI fails
````

> **Ties back to the plan-mode trap:** a non-interactive `-p` run has **no human to approve a plan**, so you **don't use plan mode** here — correctness comes from **explicit criteria in the prompt**, **`--allowedTools` / permission-mode gates**, and the **exit-code check**, not a human-approval step.

---

## 7. Skills & how they load
**What a skill is.** A directory containing a `SKILL.md` that extends what Claude can do. Claude uses it automatically when relevant, or you invoke it with `/skill-name`. `SKILL.md` = YAML frontmatter (`name`, `description`, …) + a markdown **body** of instructions. Optional **supporting files** sit in the same folder; keep `SKILL.md` under ~500 lines.

Two layout conventions (both valid — it's a convention, not a fixed filename):

*Small skill — flat* (Claude Code docs style):
```
my-skill/
├── SKILL.md        (required — overview + instructions)
├── reference.md    (single reference doc, loaded only when needed)
└── scripts/
    └── helper.py   (executed, NOT loaded into context)
```

*Larger skill — foldered* (Agent Skills spec style):
```
my-skill/
├── SKILL.md        (required — metadata + instructions)
├── references/     (docs loaded on demand — one file per topic, e.g. aws.md, gcp.md)
├── scripts/        (executable code — run, NOT loaded)
└── assets/         (templates, fonts, icons used in the output)
```

**Where skills live** (name-conflict precedence: **enterprise > personal > project**):
- *Personal:* `~/.claude/skills/<name>/SKILL.md` — all your projects on this machine.
- *Project:* `.claude/skills/<name>/SKILL.md` — this repo; commit to share.
- *Plugin:* `<plugin>/skills/<name>/SKILL.md` — invoked as `/plugin-name:skill-name`.
- *Managed/enterprise:* deployed org-wide via managed settings.
- *claude.ai account skills:* synced to `~/.claude/skills/synced/` in terminal sessions.

**How skills load — progressive disclosure (3 tiers):** this is the key concept.
1. **Always in context:** every skill's **name + description** (so Claude knows what's available). The listing text is truncated (~1,536 chars) — put the key use case first. `disable-model-invocation: true` removes the description from context entirely.
2. **On invocation:** the **body** of `SKILL.md` is added to the conversation as one message and **persists** (Claude doesn't re-read the file) — so write it as standing instructions.
3. **On demand:** **supporting files** (`reference.md`, scripts) are read/executed only when `SKILL.md` points to them and the task needs them.

Net effect: descriptions are cheap and always available; the heavy content loads only when the skill is actually used — the same context-economy idea as path-scoped rules.

**Supporting files (`reference.md`, `examples.md`, `scripts/`):**
- **Only `SKILL.md` is required** — supporting files are **optional**. Their job is to keep `SKILL.md` lean (**under ~500 lines**; move bulky detail out).
- *What goes where:* reference docs = detailed material (full API docs, schemas, option/enum tables, edge cases) — a single **`reference.md`** file for a small skill, or a **`references/` folder** (plural, one file per topic: `aws.md`, `gcp.md`, …) for a larger one; `scripts/` = code that is **executed, not loaded**; `assets/` = templates/fonts/icons used in the output. (`reference.md` vs `references/` is just scale — both are valid conventions.)
- **How a reference file is referenced:** point to it from `SKILL.md` with a **plain relative Markdown link**, saying what it contains and when to use it — Claude then reads it **on demand** (no `@import` needed):
  ````markdown
  ## Additional resources
  - For complete API details, see [reference.md](reference.md)
  - For usage examples, see [examples.md](examples.md)
  ````
- **How a script is referenced:** by path using the **`${CLAUDE_SKILL_DIR}`** variable (the folder holding `SKILL.md`), so it resolves regardless of the working directory; put the same path in `allowed-tools` so it runs without a prompt:
  ````markdown
  ---
  name: render-chart
  description: Render a chart from a CSV file
  allowed-tools: Bash(${CLAUDE_SKILL_DIR}/scripts/render.sh *)
  ---
  Run `${CLAUDE_SKILL_DIR}/scripts/render.sh <csv-file>` to render the chart.
  ````
  (Other substitutions: `${CLAUDE_PROJECT_DIR}`, `${CLAUDE_PLUGIN_ROOT}`, `$ARGUMENTS`/`$0`…)

**How skills are invoked:**
- *Model-invoked:* Claude matches your request against the `description` and loads the skill automatically.
- *User-invoked:* type `/skill-name` at the **start** of your message.
- Frontmatter gates: `disable-model-invocation: true` (you-only, e.g. `/deploy`), `user-invocable: false` (Claude-only, for background knowledge).

**Loading & management commands:**
- **`/skills`** — list available skills (synced claude.ai skills appear under a `claude.ai sync` group).
- **`/reload-skills`** — needed only when you create a **new top-level skills directory** that didn't exist at session start; edits to existing `SKILL.md` files are picked up **live**, no reload needed.
- **`/reload-plugins`** — for plugin changes beyond `SKILL.md` (e.g. `hooks/`, `.mcp.json`).
- ⚠️ **There is no `/load-skills` command** — skills load automatically by file discovery; `/reload-skills` is the manual refresh for a brand-new directory.
- The **`Skill` tool** is how Claude invokes a skill; permission rules gate it — `Skill` (block all), `Skill(commit)` (allow one), `Skill(deploy *)` (deny with args).

---

## 8. `CLAUDE.md` vs `SKILL.md` vs `/command`
These three overlap but play different roles. One-line model:
- **`CLAUDE.md`** = *always-on memory* (standing context, auto-loaded every session).
- **`SKILL.md`** = *on-demand capability* (loaded only when needed — by Claude or by you).
- **`/command`** = *a user-triggered shortcut* — in current Claude Code basically **a user-invoked skill** (commands and skills have merged).

| | `CLAUDE.md` | `SKILL.md` | `/command` |
|---|---|---|---|
| **What it is** | Persistent project/user instructions ("memory") | A packaged capability: instructions + optional scripts/files | A reusable prompt fired by name |
| **When it loads** | **Always** — injected into every session at launch | **Progressive:** name+description always in context; **body on invocation**; files on demand | On demand, when you type it |
| **Who triggers it** | Nobody — it's just there | **Claude** (auto, via `description`) *or* **you** (`/skill-name`) | **You** (type `/name`) |
| **Lives in** | `./CLAUDE.md`, `~/.claude/CLAUDE.md`, subfolder `CLAUDE.md` | `.claude/skills/<name>/SKILL.md` (or personal/plugin) | `.claude/commands/<name>.md` (or `~/.claude/commands/`) |
| **Best for** | Conventions, architecture, "always do X" rules | Task-specific workflows/knowledge not needed every session | A quick repeatable action (`/commit`, `/review`) |
| **Context cost** | Paid **every** session (keep it lean, ~<200 lines) | Cheap until used (only description always loaded) | Cheap until used |

**Key distinctions:**
- **`CLAUDE.md` vs `SKILL.md` = context economy.** CLAUDE.md loads into *every* conversation, so it costs tokens every time → keep it short. A skill costs only its description until invoked, then the body loads. So *"always true, keep it short" → CLAUDE.md; "only sometimes relevant, can be detailed" → skill*.
- **`SKILL.md` vs `/command` = they've merged.** `.claude/commands/deploy.md` and `.claude/skills/deploy/SKILL.md` both create `/deploy` (skill wins on a clash). The real difference is *invocation*: a plain command is something **you** type; a skill can **also** be model-invoked via its `description`, and adds extras (supporting files, `context: fork`, invocation controls). Docs recommend writing new ones as skills.
- **Summary:** CLAUDE.md is **passive always-on context**; skills/commands are **active, invokable capabilities** — a command is a skill you trigger by name, while a full skill can also trigger itself.

### Worked example — a repo with 10 pre-merge checks
Suppose every PR must pass 10 checks before merge (lint, unit, integration, e2e, security scan, migration safety, …). The three pieces divide the work cleanly:

- **`CLAUDE.md` — the always-on overview + rule.** It holds the *high-level policy*: *"Before merging any PR, all 10 pre-merge checks must pass. Never merge if one fails."* plus a one-line list of the checks. Loaded into **every** session, so Claude always knows the checks exist and the rule — but it stays short and does **not** contain each test's full procedure.
- **One `SKILL.md` per check — the detailed how-to, on demand.** e.g. `.claude/skills/integration-tests/SKILL.md` has the exact commands, DB/env setup, how to read failures, with a `reference.md` for edge cases. Only *that* skill's body loads when *that* check is being run, so the other 9 procedures never bloat context. (Claude can auto-invoke it when its `description` matches what you're doing.)
- **Slash command — the manual trigger (this is where `/command` comes in).** Because each skill is invokable by name, `/integration-tests` is simply **you** running that one check on demand without waiting for Claude to decide. You'd typically also add an **orchestrator** skill `/pre-merge` that runs all 10 in order and reports pass/fail.

**Roles line up as:** `CLAUDE.md` = *what the checks are + the rule* (always on) · `SKILL.md` = *how to perform each check* (loaded on demand) · `/command` = *the button you press* to run one (`/integration-tests`) or all (`/pre-merge`). The skill and its `/command` are the **same capability** — the slash command is just the human-initiated way to invoke the skill by name.

---

## 9. Subagents & context fork — context fork with respect to skills
Sits at the intersection of Domain 1 (subagent context isolation) and Domain 3.

**Core idea.** By default a **subagent starts fresh** — a clean, isolated context window. It *doesn't see your conversation history, the skills you've already invoked, or the files Claude has already read.* It does its task and returns a summary. That isolation is the point: it keeps the main thread uncluttered and lets the subagent focus.

### ⚠️ Two different "forks" — don't conflate them
The word "fork" is overloaded in Claude Code. One inherits the conversation; the other does **not**.

**(A) Forking the *conversation*** — a subagent that **inherits the entire conversation so far**: same system prompt, tools, model, and message history. *This is the one that has the prior conversation.* Its own tool calls stay out of your thread; only the final result returns. Triggered by **fork mode** (`CLAUDE_CODE_FORK_SUBAGENT`, on by default), **`/subtask`** (or `/fork`), or the Agent tool's `subagent_type: "fork"`. Reuses the parent's prompt cache, so it's cheaper than a fresh subagent. Drops the input isolation subagents normally give; can't spawn further forks.

**(B) A skill with `context: fork`** — **does NOT give the subagent your conversation.** Claude Code starts a **fresh, isolated subagent** (of the `agent:` type) with the **skill content as its prompt**; it **does not see your conversation history**, so the skill's instructions must stand on their own. Runs in the background by default (`background: false` to wait), then summarises and returns results to the main thread. Purpose: keep a heavy/multi-step skill's execution detail out of the main conversation — *not* to hand it prior context.

**Key reason to choose it:** because it ignores the chat, the skill runs **consistently every time, independent of the previous conversation**. Use `context: fork` when you want a skill's output to be the **same each run** — unaffected by whatever was discussed earlier — rather than coloured by the current conversation's state.

> **Official docs, verbatim:** *"Despite the name, a skill with `context: fork` doesn't run in a fork of the current conversation, which would hand the subagent everything you've discussed so far. When the task depends on that history, fork the conversation instead of using `context: fork`."*

**Quick contrast:**

| | Fresh subagent (default) | Skill `context: fork` | Conversation fork (`/subtask`, fork mode) |
|---|---|---|---|
| Sees prior conversation? | ❌ No | ❌ **No** | ✅ **Yes** (full history) |
| Prompt it runs on | Its definition / delegation msg | **The skill content** | The main conversation's prompt |
| Tools | Filtered | The `agent:` type's tools | Main conversation's exact tools |
| Main use | Self-contained delegated task | Isolate a heavy skill's execution | Side task that needs prior context |

*Example* — a skill with `context: fork` (note: instructions are **self-contained**, because it can't see the chat):
````markdown
---
name: deep-research
description: Research a topic thoroughly
context: fork        # fresh, isolated subagent — does NOT inherit the conversation
agent: Explore       # which agent type to run as
---

Research $ARGUMENTS thoroughly:
1. Find relevant files with Glob and Grep
2. Read and analyse the code
3. Summarise findings with specific file references
````
Because the subagent starts fresh, everything it needs must be in the skill (and in `$ARGUMENTS`) — you can't rely on "the logs we discussed." If the task **does** need the chat history, use a **conversation fork** (`/subtask …`) instead.

> **Note:** `context: fork` is a **skill** frontmatter field, *not* a subagent-definition field (subagent definitions use `name`, `description`, `tools`, `model`, `isolation`).

> **Exam angle:** know the *concept* — fresh/isolated vs. inherited context — and above all that **`context: fork` is isolated (no chat history)**, while a **conversation fork inherits it**. The shared word is the trap.

---

## 10. Common traps
- `CLAUDE.md` is **auto-loaded context**, not a build file or a secrets store. **Never put API keys in it.**
- Hooks run **your** commands deterministically — they're not model behaviour.
- `PreToolUse` can **prevent** an action; `PostToolUse` reacts **after**.
- **`Stop` hook to force-continue (verified):** a `Stop` hook runs when the agent tries to *finish*. Return **`decision: "block"` + `reason`** to prevent the stop and tell Claude why to keep going (e.g. "pytest still red"). Guard the loop with **`stop_hook_active`** (plus Claude Code's cap on consecutive continuations) so it can't run forever. This *enforces* "don't finish until green" — a system prompt or a higher `max_turns` does **not** (the agent is *choosing* to stop, not running out of turns). A `PostToolUse` hook only *informs* after an edit; it never stops the turn ending.
- **Hook matcher for MCP tools (verified):** MCP tools are named **`mcp__<server>__<tool>`** (double underscores), and hook matchers match as an **exact string** — so `mcp__warehouse` matches *no* tool and the hook never fires. Use the wildcard **`mcp__<server>__*`** (or the full tool name). MCP tools do **not** run through the built-in `Bash` tool, and only **`PreToolUse`** can block *before* execution (`PostToolUse` is too late).
- **Agent SDK isolation (verified):** by default the SDK loads filesystem config (stray `CLAUDE.md`, rules) and **auto-memory**. To make behaviour come *only* from app code: pass **`settingSources: []`** (load no filesystem settings) and supply instructions via the **`systemPrompt`** option; and set **`CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`** in the `env` option to stop cross-session memory. `mcpServers: {}` is a **wrong-knob** trap — MCP isn't the source of the CLAUDE.md rules or the memories; a prose "ignore other files" `CLAUDE.md` is advisory.

---

## 11. Quick-fire revision
- Q: What is `CLAUDE.md` for? **A:** Persistent, auto-loaded project instructions/context.
- Q: Where do you put a personal, non-shared instruction? **A:** `CLAUDE.local.md` / user-level `~/.claude/CLAUDE.md`.
- Q: Project-level vs subfolder `CLAUDE.md`? **A:** Project-level (repo root) is team-shared for the whole project; a subfolder `CLAUDE.md` loads on demand only when working in that subtree and supplements/overrides the root there.
- Q: Hook that runs before a tool call and can block it? **A:** `PreToolUse`.
- Q: How does a hook block an action? **A:** Exit **`2`** (blocking error), or exit 0 and print JSON — `PreToolUse`: `permissionDecision:"deny"`; others: `decision:"block"`. (Exit 2 always wins over a JSON `allow`.)
- Q: Three config levels of a hook + what `matcher` does? **A:** **event → matcher group → handler** (`{type:"command", command:…}`); `matcher` filters which tool fires it (`"*"`/exact/`|`-list/regex).
- Q: Does an `allow` in `settings.local.json` override a `deny` in the committed project `settings.json`? **A:** **No.** Permission lists **merge across scopes**, and **`deny` beats `allow` from any scope** (order `deny`→`ask`→`allow`) → **blocked, no prompt**. File precedence only decides scalar keys (e.g. `model`), not allow-vs-deny.
- Q: How does Claude Code gain extra external tools? **A:** By connecting MCP servers.
- Q: How do you keep a `CLAUDE.md` modular? **A:** Use `@path` imports to pull in topic files (relative-to-file paths; up to 4 hops; organises but doesn't cut context cost).
- Q: How do you load an instruction **only when Claude touches certain files**? **A:** A path-scoped rule in `.claude/rules/` with a `paths:` glob (e.g. `src/api/**/*.ts`) — it loads on Read/Write/Edit of a matching file (or a `cat`/`head`/**`grep`**/`rg` that counts as a read), not at launch.
- Q: Does a `grep` on a matching file trigger a path-scoped rule? **A:** **Yes** — `grep`/`rg`/`cat`/`head` etc. (one file, no pipes) count as a read. **Exception:** a search with **no matches** doesn't count.
- Q: Can a rule be scoped by what's *inside* a file (its content)? **A:** **No** — `paths` matches **path globs only**; there's no content/regex scoping. (It's an open feature request, not shipped.)
- Q: Several rules/`CLAUDE.md`s apply at once — how are they combined? **A:** **Concatenated, not overridden**; later/more-specific tends to win, true conflicts are arbitrary → keep consistent. Order: managed → **user (incl. `~/.claude/rules/`)** → **project (incl. unconditional `.claude/rules/`)** → local → **path-scoped rules + subfolder `CLAUDE.md` on demand (latest)**.
- Q: Do user rules override project rules (or vice-versa)? **A:** **Neither overrides** — user loads first, project appears later; on conflict Claude may follow either. (No guaranteed order among same-scope rules.)
- Q: What's the one frontmatter field a rule reads, and what happens without it? **A:** `paths` (the only field read). **No `paths`** → the rule loads every session at launch, same priority as `.claude/CLAUDE.md`.
- Q: `@path` import vs path-scoped rule for cutting context? **A:** Imports only *organise* (they load with the file that imports them); a **path-scoped rule** actually trims context because it loads only for matching files.
- Q: Where do custom slash commands live and how are they named? **A:** Markdown files in `.claude/commands/` (project) or `~/.claude/commands/` (personal); the filename without extension is the command (`deploy.md` → `/deploy`). Now also creatable as skills.
- Q: `CLAUDE.md` vs `SKILL.md` vs `/command` in one line? **A:** CLAUDE.md = always-on memory (every session); SKILL.md = on-demand capability (model- or user-invoked); `/command` = a user-triggered shortcut, now basically a user-invoked skill.
- Q: Why keep instructions in a skill rather than `CLAUDE.md`? **A:** CLAUDE.md costs tokens every session; a skill only loads when invoked — use it for detailed, sometimes-relevant content.
- Q: Plan mode vs direct execution in one line? **A:** Plan mode = Claude **researches & proposes but doesn't edit your source** (until you approve the plan); direct-execution modes (`default`/`acceptEdits`/`auto`…) let it **actually edit/run**.
- Q: How do you enter plan mode? **A:** `Shift+Tab` to cycle modes, `claude --permission-mode plan`, or prefix a prompt with `/plan`.
- Q: What can Claude do *in* plan mode? **A:** Read files + run exploratory shell commands + write a plan — but **not edit source**.
- Q: Which mode runs *everything* without asking, and where is it safe? **A:** `bypassPermissions` — **only** in isolated containers/VMs (no internet). (`auto` also runs everything but with classifier safety checks.)
- Q: In `default` (Manual) mode, what runs without a prompt? **A:** **Reads only** — every edit/command prompts.
- Q: When do you pick plan mode vs direct execution? **A:** **Plan mode** for big/risky/wide blast radius (multi-file migrations, bulk renames, hard-to-reverse ops); **direct execution** for small reversible fixes — and for **non-interactive CI/CD** (no human to approve), where you rely on explicit prompt criteria + code gates, not plan mode.
- Q: Why is plan mode a poor fit for a non-interactive CI/CD pipeline? **A:** It **assumes a human is present to review/approve** before execution — a headless (`-p`) run has nobody to approve, so correctness must come from upfront criteria + tests/hooks instead.
- Q: Which flag runs Claude Code non-interactively for CI/CD, and how does CI know it passed? **A:** **`-p` / `--print`** (optionally `--bare` for reproducibility); CI branches on the **exit code** (0 = success, non-zero = failure).
- Q: How do you keep an unattended `-p` run from hanging on permission prompts? **A:** `--permission-prompts none` (deny anything that would prompt) and/or `--permission-mode dontAsk`; pre-approve needed tools with `--allowedTools`.
- Q: Three `-p` output formats? **A:** `text` (default), `json` (one object: `result` + metadata like `total_cost_usd`), `stream-json` (newline-delimited events, needs `--verbose`). Schema-constrain with `--json-schema` → read `.structured_output`.
- Q: How does `--json-schema` produce its output (what shape)? **A:** As a **tool call** — Claude Code injects a synthetic **`StructuredOutput`** tool and the agent calls it (`tool_use` input = your object; `tool_result` = "provided successfully"; `is_error:true` → retried). Read the data from the **result message's `structured_output`**, not the `tool_use` block.
- Q: Which setting controls **where plan files are written**? **A:** `plansDirectory`. (`useAutoModeDuringPlan` = let the classifier vet commands while planning; `permissions.defaultMode` = which mode new sessions start in.)
- Q: How do skills load (progressive disclosure)? **A:** Names + descriptions always in context; the `SKILL.md` body loads on invocation; supporting files load on demand.
- Q: Is `reference.md` required, and how is it referenced? **A:** Optional (only `SKILL.md` is required); reference it from `SKILL.md` with a plain relative Markdown link (e.g. `[reference.md](reference.md)`) so Claude reads it on demand. Scripts use a `${CLAUDE_SKILL_DIR}/…` path instead.
- Q: `reference.md` file or `references/` folder? **A:** Both are valid conventions — a single `reference.md` for a small skill; a `references/` folder (one file per topic) for larger ones, alongside `scripts/` and `assets/`.
- Q: Command to refresh skills, and does `/load-skills` exist? **A:** `/skills` lists them, `/reload-skills` refreshes a brand-new skills directory (existing edits load live). There is **no `/load-skills`**.
- Q: Does a skill with `context: fork` see the prior conversation? **A:** **No** — it runs in a fresh, isolated subagent with the skill content as its prompt; instructions must be self-contained.
- Q: Which "fork" *does* inherit the conversation? **A:** A **conversation fork** (`/subtask`, fork mode / `CLAUDE_CODE_FORK_SUBAGENT`, or Agent `subagent_type: "fork"`) — same history, prompt, tools, model.
- Q: Task needs the chat history but should run in a subagent — which do you use? **A:** Fork the conversation (`/subtask`), **not** `context: fork`.

**One-line summary:** *`CLAUDE.md` = auto-loaded project memory; hooks = deterministic shell automation on tool events; configure tools/permissions in `settings.json`.*