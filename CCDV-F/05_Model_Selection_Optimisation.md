# CCDV-F Revision Note — Domain 5: Model Selection & Optimisation

> **Exam weight: ~16.8%** (2nd heaviest). Lots of *"pick the right model / cheapest-correct optimisation"* scenario questions. The through-line: **right-size by evals, then pull cost/latency levers.**

---

## Contents
1. [Selecting the right model](#1-selecting-the-right-model)
   - [1.1 The model tiers (map scenario → tier)](#11-the-model-tiers-map-scenario--tier)
   - [1.2 The capability ↔ latency ↔ cost triangle](#12-the-capability--latency--cost-triangle)
   - [1.3 Evaluation-driven selection (right-sizing)](#13-evaluation-driven-selection-right-sizing)
2. [Optimisation levers](#2-optimisation-levers)
   - [2.1 Prompt caching (big cost/latency lever)](#21-prompt-caching-big-costlatency-lever)
   - [2.2 Message Batches API (bulk cost lever)](#22-message-batches-api-bulk-cost-lever)
   - [2.3 Model routing](#23-model-routing)
   - [2.4 Reasoning levers: `effort` and extended thinking](#24-reasoning-levers-effort-and-extended-thinking)
   - [2.5 Other optimisation levers](#25-other-optimisation-levers)
3. [Putting it together](#3-putting-it-together)
   - [3.1 Latency vs cost (they're different)](#31-latency-vs-cost-theyre-different)
   - [3.2 Lever cheat-sheet & decision flow](#32-lever-cheat-sheet--decision-flow)
4. [Common traps](#4-common-traps)
5. [Quick-fire revision](#5-quick-fire-revision)

---

## 1. Selecting the right model
*First decide **which model**, driven by evals — then (§2) make it cheaper and faster.*

### 1.1 The model tiers (map scenario → tier)
The exam tests the **tier concept**, not exact version strings (those change — verify current IDs/prices on the pricing page). Three families:

| Tier | Character | Use when |
|---|---|---|
| **Haiku** | **Fastest, cheapest, smallest** | High volume, latency-sensitive, well-scoped simple tasks: **classification, extraction, routing, moderation, simple Q&A** |
| **Sonnet** | **Balanced** capability/cost/speed | The **everyday** choice: coding, RAG, agents, general tasks |
| **Opus** | **Top-tier** reasoning/agentic, high cost/latency | **Hardest reasoning**, complex multi-step agents, research, high-stakes accuracy *(see Fable note below — it now sits above Opus)* |

- Within a family, versions advance (e.g. Haiku 5.5, Sonnet 5.5, Opus 5.5) — the **relative positioning stays**: Haiku < Sonnet < Opus on capability, and the reverse on speed/cost.
- **Haiku is also the standard "cheap helper"** — use it for harmlessness screens (Domain 7), routing decisions, and sub-tasks inside a bigger pipeline.

> **Current lineup (verified — the docs' own "choosing a model" matrix):** the family triangle still holds, but today's guidance maps *starting points* like this:
> | When you need… | Start with… |
> |---|---|
> | **Highest available capability** (hours-long agents, deep research) | **Fable 5.1** |
> | **Complex agentic coding / enterprise, vision-heavy, computer use** | **Opus 5.5** |
> | **Everyday coding, agents, enterprise** (speed *and* capability) | **Sonnet 5.5** |
> | **Lowest latency & price** (real-time, high-volume, subagents) | **Haiku 5.5** |
>
> The docs note **"most workloads start with Claude Opus 5.5."** There's a **Fable** tier *above* Opus now — so "Opus = most capable" is no longer strictly true; frame it as *"Opus = top-tier reasoning/agentic; Fable = the capability ceiling."*

> **Two official starting strategies (verified):**
> - **Efficiency-first:** *start on Haiku 5.5, test thoroughly, upgrade only if a specific capability gap appears.* Best for prototyping, tight latency, cost-sensitive, high-volume-simple work — "often sufficient for many common applications."
> - **Capability-first:** *start on Opus 5.5* when intelligence is paramount (complex reasoning, scientific/math, high-autonomy agents, accuracy > cost).
> Either way, **you then right-size with evals** (§1.3).

> **Exam trap:** don't reflexively default everything to the **top** tier — that's needless cost. Pick a starting point by the matrix above, then **move by evals**: down if a cheaper tier still passes, up only if quality demands. (And before switching models at all, try tuning **effort** — §2.4.)

### 1.2 The capability ↔ latency ↔ cost triangle
Every selection balances three things — **you can't max all three**:
- Smartest answer → **Opus** (or **Fable** for the very top; pay in **$** and **time**).
- Speed / scale / cheap → **Haiku**.
- Sensible middle → **Sonnet**.
- **Routing** (§2.3) escapes the trade-off *per request*: easy inputs → Haiku, hard inputs → Opus.

### 1.3 Evaluation-driven selection (right-sizing)
The method the exam wants (ties into Domain 4):
1. **Start** at the matrix starting point (§1.1) — efficiency-first (Haiku) or capability-first (Opus).
2. **Build an eval set** from your **actual prompts and data**, with a scoring method. The docs are blunt: *"Having a good evaluation set is the most important step in the process."*
3. **Measure** on **accuracy, response quality, edge-case handling** — then weigh the **performance ↔ cost** trade-off.
4. **Move down** a tier if the cheaper one still passes; **move up** only if quality demands it.

> **The #1 cost optimisation = the smallest model that passes your evals.** Guesswork ("the biggest model is safest") wastes money; evals make the choice **defensible**.

> **Before you switch models, try `effort` (verified):** *"Tuning effort is often a better lever than switching models."* On Opus 5.5 / Haiku 5.5 the default effort is **`medium`** — nudge it up for a hard case before jumping a tier, down to save cost (see §2.4).

---

## 2. Optimisation levers
*You've picked a model (§1). These levers cut its cost and latency without downgrading quality.*

### 2.1 Prompt caching (big cost/latency lever)
**The idea in one line:** pay to *memorise* a big, unchanging chunk of the prompt once, then **re-read it almost for free** on every later call that starts with the same chunk — so you stop paying full price to reprocess the same system prompt / tool defs / documents every turn. It cuts **cost *and* latency** on the cached part.

**What gets cached — a prefix, in a fixed order.** Caching references the whole prompt in the order **`tools` → `system` → `messages`**, *up to and including* the block you mark. So the cache is always a **prefix** from the very start of the request; you can't cache something in the middle while leaving earlier content uncached.

**Two ways to set it (verified):**
- **Automatic** — one top-level field; the API puts the breakpoint on the last cacheable block and advances it as the conversation grows:
  ```python
  resp = client.messages.create(
      model="claude-opus-5-5",
      max_tokens=1024,
      cache_control={"type": "ephemeral"},        # ← top-level = automatic
      system="<long static instructions…>",
      messages=[{"role": "user", "content": "…"}],
  )
  ```
- **Explicit** — put `cache_control` on the specific block that ends your static prefix (needed for fine control / multiple breakpoints):
  ```python
  system=[
      {"type": "text", "text": "<long static instructions…>"},
      {"type": "text", "text": "<entire 50-page policy doc…>",
       "cache_control": {"type": "ephemeral"}},    # ← breakpoint here
  ]
  ```
- **Up to 4 breakpoints** total (automatic caching uses one of the 4 slots).

**Order matters:** static content **first** (tools → system → long docs), dynamic/user content **last**. The cache matches a prefix, so a single changed byte early in the prompt misses the whole cache; anything after the last breakpoint is billed as normal input.

**TTL:** **default 5 minutes**; a **1-hour** option via `{"type":"ephemeral","ttl":"1h"}` (costs more to write). **Every hit refreshes the TTL for free**, and the lifetime is measured from the **start** of the request that reads/writes it — so a steady stream of calls keeps a cache alive indefinitely.

**Economics (verified multipliers vs base input price):**

| Operation | Multiplier |
|---|---|
| **Cache write, 5-min TTL** | **1.25×** base input |
| **Cache write, 1-hour TTL** | **2×** base input |
| **Cache read (hit)** — most models | **0.1×** base input |
| **Cache read (hit)** — Opus 5.5 / Sonnet 5.5 | **0.05×** |
| **Cache read (hit)** — Fable 5.1 / Mythos 5.1 | **0.025×** |

→ Writing costs a **bit more** than normal input *once*; **reading is ~10–40× cheaper**. Net win only when the prefix is **reused enough** to amortise the write (roughly: worth it from the **2nd–3rd reuse** within the TTL).

**Minimum cacheable length (verified, per-model — tiny prefixes just won't cache, *no error*):**

| Min tokens | Models |
|---|---|
| **512** | Fable/Mythos 5.1, Opus 5.5, Opus 5, Sonnet 5.5, Fable/Mythos 5, Haiku 5.5 |
| **1,024** | Opus 4.8, Sonnet 5, Sonnet 4.6 |
| **2,048** | Opus 4.7, Mythos Preview |
| **4,096** | Opus 4.6, Opus 4.5, Haiku 4.5 |

**Usage fields (how to *prove* caching worked):**
- **`cache_creation_input_tokens`** — tokens **written** to the cache (a miss / first write, billed at the 1.25×/2× write rate).
- **`cache_read_input_tokens`** — tokens **served from** the cache (a hit — the cheap 0.1×/0.05×/0.025× rate).
- **`input_tokens`** — tokens **after** the last breakpoint (neither read nor written).
- `total_input = cache_read_input_tokens + cache_creation_input_tokens + input_tokens`.
- **Both cache fields `0` ⇒ nothing cached** (prefix too short, or no prefix match).

```jsonc
// usage on a cache HIT: 100k-token prefix reused + 50 new tokens
"usage": {
  "cache_read_input_tokens": 100000,   // ← served cheap from cache
  "cache_creation_input_tokens": 0,
  "input_tokens": 50                    // ← only the new, dynamic part
}                                        // total_input = 100,050
```

**What can / can't be cached (verified):** cacheable = **tool definitions, system blocks, text/image/document blocks, and tool-use & tool-result blocks** (user *and* assistant turns). **Thinking blocks can't be cached directly** with `cache_control` (they ride along inside a previous assistant turn), and you can't cache sub-blocks like citations or empty text blocks.

**Where the breakpoint sits (prefix model):**
```
┌──────────────────────────────────────────────┐
│ tools        (static)                          │ ┐
├──────────────────────────────────────────────┤ │  cached
│ system       (static)                          │ ├─ prefix
├──────────────────────────────────────────────┤ │  (0.1× / 0.05× / 0.025× on hit)
│ long documents / few-shot   (static)           │ ┘
│   …………………………………………… cache_control ✔ breakpoint │ ◄── end of cached prefix
├──────────────────────────────────────────────┤
│ the user's dynamic question   (changes)        │ ◄── normal input_tokens
└──────────────────────────────────────────────┘
   change anything ABOVE the breakpoint → whole cache misses
```

> **Trap:** a **cache hit** shows up as **`cache_read_input_tokens`**, *not* `input_tokens`. Dynamic content must come **after** the cached block, or the prefix won't match and you'll pay full price.

### 2.2 Message Batches API (bulk cost lever)
**The idea:** hand the API a big pile of requests, walk away, and collect the results later — in exchange for a **50% discount**. You trade *immediacy* for *price*.

- **Endpoint:** create via `client.messages.batches.create(requests=[…])` (`/v1/messages/batches`); poll with `GET /v1/messages/batches/{id}` (idempotent), delete with `DELETE`.
- **Cost (verified):** **all usage at 50% of standard API prices** — a flat half-price on both input and output.
- **Timing (verified — don't misread this):** fully **asynchronous**; **most batches finish in under 1 hour**. The **24 hours is a hard cap, not a target**: you can pull results once *all* requests complete **or** after 24h, whichever comes first, and **any batch that hasn't completed in 24h expires** (expired/errored/cancelled requests aren't billed).
- **Limits (verified):** a batch is capped at **100,000 requests or 256 MB, whichever it hits first**. Each `custom_id` matches `^[a-zA-Z0-9_-]{1,64}$`; each request needs `max_tokens ≥ 1`.
- **Results availability:** downloadable for **29 days** after creation; batches are **scoped to a Workspace**.
- **Use for:** latency-tolerant bulk jobs — **large-scale evals, content moderation at volume, data analysis/summaries, bulk content generation**.
- **Not for:** anything interactive/real-time.
- **Stacks with caching** (discounts combine) — but cache hits inside a batch are **best-effort** (observed hit rates ~**30–98%**); keep every request's `cache_control` block **identical** and consider the **1-hour TTL** so the shared prefix survives the batch.

> **Trap:** "results within 24 hours" is the **expiry deadline**, not a speed promise — most finish far sooner, but a batch is **async** and must never sit on an interactive path.

### 2.3 Model routing
- A **cheap model triages first**, then **escalates only hard cases** to a bigger model.
- Pattern: **Haiku classifies difficulty / handles the easy 80%**, **Opus/Sonnet handle the hard 20%**.
- Gives **Haiku economics on easy inputs** and **Opus quality on hard ones** — beats sending everything to one tier.
- The router itself is a cheap, well-scoped classification call (a great Haiku job).

### 2.4 Reasoning levers: `effort` and extended thinking
Two dials buy *more reasoning* without changing tiers — reach for these **before** upgrading the model.

**`effort` (verified — often the better first lever):**
- Controls **how much reasoning the model spends** per request. On **Opus 5.5 / Haiku 5.5** the default is **`medium`**.
- *"Tuning effort is often a better lever than switching models"* — nudge **up** for a hard case (may rescue a cheaper tier), **down** to cut cost/latency on easy traffic.
- Cheaper to try than a tier jump: same model, same prices, just more/less internal work.

**Extended thinking (`thinking` + `budget_tokens`):**
- Buys **more reasoning** for hard tasks — can let a **mid tier succeed** where it otherwise wouldn't, sometimes avoiding a jump to Opus.
- **Costs more tokens + latency** (thinking tokens are billed as output and count toward the window — Note 06 §4.1), so use it **where reasoning quality matters**, not everywhere.
- Trade-off lives on the same triangle: thinking spends tokens/time to buy capability.

> **Order of escalation for a too-hard task:** raise **`effort`** → add/raise **extended thinking** → only then **upgrade the tier**. Each step costs more, so climb only as far as the evals require.

### 2.5 Other optimisation levers
- **`max_tokens` tuning** — cap output length; shorter outputs = **lower cost & latency** (output tokens usually cost more than input).
- **Prompt efficiency** — trim redundant context; shorter inputs cost less (but keep what's needed for quality).
- **Streaming** — improves **perceived** latency (tokens appear immediately) but **does not reduce cost or total time**.
- **Choose the smallest model that passes evals** — the headline lever (see §1.3).

---

## 3. Putting it together
*Pick the model (§1), stack the levers (§2) — these two subsections help you choose which lever for which goal.*

### 3.1 Latency vs cost (they're different)
Some scenarios ask about **speed**, not money — don't conflate:
- **Lower latency:** smaller tier (**Haiku**), **shorter outputs** (`max_tokens`), **prompt caching** (less prefix to process), **streaming** (perceived), avoid unnecessary extended thinking.
- **Lower cost:** smaller tier, **caching** (reused prefix), **Batches** (async bulk), shorter prompts/outputs, routing.
- **Streaming** helps *perceived latency only* — it's **not** a cost lever and doesn't speed total generation.

### 3.2 Lever cheat-sheet & decision flow
| Goal | Best lever |
|---|---|
| Cheaper bulk, offline | **Batches API** (50% off, async; most <1h, 24h cap) |
| Cheaper repeated big prefix | **Prompt caching** (0.1×/0.05×/0.025× reads) |
| Faster *feel* in chat | **Streaming** (perceived only) |
| Lower cost per task | **Smaller tier** / shorter prompts & outputs |
| Hard + easy mixed inputs | **Routing** (Haiku ↔ Opus) |
| A bit more reasoning, same tier | **`effort` up** → then **extended thinking** |
| Hard reasoning on a mid tier | **Extended thinking** (`budget_tokens`) |
| Pick the right model at all | **Evals** → smallest tier that passes |

**Decision flow:** *Is it interactive?* → yes: pick tier by evals, cache the static prefix, stream for feel, route if inputs vary. → no (bulk/offline): use the **Batches API** (+ caching) on the smallest tier that passes evals.

---

## 4. Common traps
- **Caching ≠ batching.** Caching = reuse a **static prefix** across calls (real-time OK). Batching = **async bulk discount** (not real-time).
- **Batches "24h" is the expiry cap, not a speed promise** — most finish <1h, but it's still async (never on an interactive path).
- **Streaming does not reduce cost** — only perceived latency.
- Don't reflexively default to the **top** tier = wasted money; pick a start point from the matrix (often **Opus 5.5**, or **Haiku** efficiency-first) and **move by evals**.
- **Try `effort` before switching models** — same model/price, more or less reasoning.
- **"Opus = most capable" is dated** — **Fable 5.1** now sits above it as the capability ceiling.
- Cache matches a **prefix** → dynamic content must come **after** the cached block.
- A **cache hit** = **`cache_read_input_tokens`** (not `input_tokens`); both 0 ⇒ nothing cached.
- **Cache write costs more** (1.25× 5-min / 2× 1-hour); it only pays off if the prefix is **reused enough**.
- Minimum cacheable length applies — a **short prefix just won't cache** (no error).
- **Right-sizing is driven by evals**, not vibes.

---

## 5. Quick-fire revision
- Q: Everyday-default balanced tier? **A:** **Sonnet 5.5** (speed + capability for general work).
- Q: Cheapest/fastest for simple high-volume? **A:** **Haiku** (also the efficiency-first starting point).
- Q: Top reasoning/agentic tier — and the capability *ceiling* above it? **A:** **Opus 5.5** for complex agentic/coding; **Fable 5.1** is the highest-capability tier.
- Q: #1 cost optimisation? **A:** **Smallest model that passes your evals.**
- Q: The two official starting strategies? **A:** **Efficiency-first** (start Haiku, upgrade only on a capability gap) and **capability-first** (start Opus).
- Q: Cheaper lever to try *before* switching models? **A:** **Tune `effort`** (default `medium` on Opus 5.5 / Haiku 5.5) — then extended thinking.
- Q: Flag to cache a block? **A:** `cache_control: {"type":"ephemeral"}` (add `"ttl":"1h"` for 1 hour).
- Q: Default cache TTL? **A:** **5 minutes** (refreshed free on hit); **1-hour** option available.
- Q: Cache write vs read cost? **A:** Write **1.25×** (5-min) / **2×** (1-hour); read **0.1×** base — **0.05×** on Opus 5.5/Sonnet 5.5, **0.025×** on Fable/Mythos 5.1.
- Q: Field proving a cache hit? **A:** **`cache_read_input_tokens`**.
- Q: Where must dynamic content go relative to a cached prefix? **A:** **After** it (cache matches a prefix: `tools`→`system`→`messages`).
- Q: Batch API discount, timing & size cap? **A:** **50% off**, **async** (most **<1h**, **24h** = expiry cap), **100k requests / 256 MB** max; results kept **29 days**.
- Q: Does streaming cut cost? **A:** **No** — only perceived latency.
- Q: How to get Opus quality on hard inputs but Haiku cost on easy ones? **A:** **Routing** (triage cheap, escalate hard).
- Q: Lever to boost reasoning without jumping to Opus? **A:** **Extended thinking** (`budget_tokens`).
- Q: Two levers that cut **latency** specifically? **A:** Smaller tier + shorter `max_tokens` (also caching; streaming for *perceived*).
- Q: Can you maximise capability, latency **and** cost all at once? **A:** **No** — it's a trade-off triangle; pick two, or use **routing** to escape it *per request*.
- Q: The eval-driven right-sizing loop? **A:** **Start** at the matrix point → **build an eval set** (the *most important step*) → **measure** accuracy/quality/edge-cases vs cost → **move down** if the cheaper tier still passes, **up** only if quality demands.
- Q: How many cache breakpoints can you set? **A:** **Up to 4** (`cache_control` marks the end of each cacheable prefix).
- Q: Minimum cacheable length? **A:** Per-model: **512** (current flagships), **1,024**, **2,048**, up to **4,096** (older); a shorter prefix is processed **without caching, no error**.
- Q: Can a **thinking block** be cached directly? **A:** **No** — not via `cache_control`; it only rides along inside a cached previous assistant turn.
- Q: What does it mean if **both** `cache_creation_input_tokens` **and** `cache_read_input_tokens` are 0? **A:** **Nothing was cached** — prefix too short or no prefix match.
- Q: Formula for total input tokens? **A:** `total_input = cache_read + cache_creation + input_tokens`.
- Q: Caching vs batching — the difference? **A:** **Caching** = reuse a static **prefix** across calls (real-time OK); **batching** = **async bulk** discount (not real-time). They **stack**.
- Q: Why not reflexively default to the top tier? **A:** **Needless cost** — pick a matrix start point (often **Opus 5.5**, or **Haiku** efficiency-first), then **move by evals**.
- Q: What's the cost of extended thinking? **A:** **More tokens + latency** — use it only where reasoning quality pays (it can let a **mid tier** succeed instead of jumping to Opus).
- Q: Is the **router** itself an expensive call? **A:** **No** — it's a cheap, well-scoped **classification** call (a great **Haiku** job).
- Q: Does a cache **hit** refresh the TTL? **A:** **Yes** — each hit refreshes the 5-min (or 1-hr) TTL **for free**.

**One-line summary:** *Right-size the tier (Haiku → Sonnet → Opus → Fable ceiling) **by evals**; try **`effort`** before switching models; **cache** static prefixes (write 1.25×/2×, read 0.1×/0.05×/0.025×, hit = `cache_read_input_tokens`), **batch** bulk async (50% off, most <1h / 24h cap), **stream** for feel (not cost), **route** easy vs hard, and spend **extended thinking** only where reasoning pays.*