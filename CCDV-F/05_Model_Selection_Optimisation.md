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
   - [2.4 Reasoning levers: `effort` and thinking modes](#24-reasoning-levers-effort-and-thinking-modes)
   - [2.5 Other optimisation levers](#25-other-optimisation-levers)
   - [2.6 Service tier / Priority Tier (reliability lever)](#26-service-tier--priority-tier-reliability--overload-lever)
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
When you choose a model you're really juggling **three things you want at once**:
- **Capability** — how smart/accurate the answer is.
- **Latency** — how *fast* it replies (low latency = quick).
- **Cost** — how cheap it is per token.

The catch is the classic **"fast, good, cheap — pick two"** rule. You **can't max all three**, because they're linked: a bigger, smarter model has more to compute, so it's **slower and pricier**; a small model is **fast and cheap** but gives up some capability. Picking a model is really deciding **which corner you lean toward** for this particular task:

```
                 CAPABILITY (smartest)
                    ▲  Opus / Fable
                   ╱ ╲      · pay in $$ + time
                  ╱   ╲
                 ╱     ╲
                ╱ Sonnet╲        ◄── the balanced middle
               ╱  (mid)  ╲
              ╱           ╲
  LATENCY ◄──┴─────────────┴──► COST
  (fastest)     Haiku            (cheapest)
             fast + cheap, less capable
```

- Need the **smartest** answer? → **Opus** (or **Fable** for the very top) — and accept **more $ and more waiting**.
- Need **speed / scale / low cost**? → **Haiku** — and accept a **lower capability ceiling**.
- Want a **sensible balance** of all three? → **Sonnet** sits in the middle.

**The one escape hatch: routing (§2.3).** The trade-off above is per *model choice* — but you don't have to make one choice for *every* request. **Routing** sends each input to a different model: easy inputs → **Haiku** (fast + cheap), hard inputs → **Opus** (smart). That way you get Haiku economics on the easy 80% and Opus quality on the hard 20%, instead of overpaying for easy work or under-serving hard work.

> **Exam cue:** if a question stresses **one** corner — "needs to be *real-time*", "*cheapest* at huge volume", "*highest accuracy* regardless of cost" — it's pointing you at a corner (Haiku for speed/cost, Opus/Fable for capability). If it mixes easy and hard inputs, the answer is usually **routing**, not a single tier.

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
**Plain idea:** normally the model re-reads your *whole* prompt from scratch on every call, and you pay for all of it each time. Caching lets you say *"this big chunk never changes — remember it."* You pay a small premium to **write** it to the cache once, and every later call that starts with the same chunk **reads** it back for a fraction of the price — and faster, because there's less to reprocess. Classic targets: a long **system prompt**, your **tool definitions**, or a big **document** you reference every turn. It cuts **cost *and* latency** on the cached part.

**It only caches a *prefix*.** Caching looks at your request in the fixed order **`tools` → `system` → `messages`** and caches everything **from the start up to the block you mark** — so the cached part is always the *opening slice* of the prompt. You can't cache a bit in the middle while leaving the start uncached. The practical rule: **put the unchanging stuff first, the per-request stuff last.**

**What decides *how much* gets cached — the breakpoint (nothing else).** A **cache breakpoint** is just a marker that says *"the cached part ends HERE."* The cache always begins at the very top of the request; the only choice anyone makes is **where it stops**. Everything from the top up to **and including** the block carrying the marker is cached; everything after it is processed fresh. Think of your prompt as a document and the breakpoint as a **bookmark**: everything up to the bookmark is saved, everything past it is re-read each call. The API does **not** scan for what changed — the portion is decided purely by the breakpoint's **position**. The two "ways" below are just *who places the bookmark*: the **API** (automatic) or **you** (explicit).

**Two ways to switch it on (verified):**
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
  So **automatic = the API picks the stopping point** (the last cacheable block, nudged forward each turn — you cache as much as it safely can); **explicit = you pick it** by attaching `cache_control` to the exact block where your static content ends (and you can set several).
- **Up to 4 breakpoints** total (automatic caching uses one of the 4 slots). Each `cache_control` writes one entry (a hash of the prefix ending there); on a later call the system checks at most **20 positions** per breakpoint for a match. If you mix TTLs, the **1-hour entry must come before any 5-minute** one.

> **Why you'd use more than one breakpoint (the "4 breakpoints" budget).** You get a **budget of 4** bookmarks per request. Each one caches *"everything from the start up to here,"* so multiple breakpoints are **nested prefixes**, and on a later call the API reuses the **longest prefix that still matches**. This protects you when parts of the prompt change at different speeds. Example with two breakpoints — ① end of `tools` → caches **[tools]**; ② end of `system` → caches **[tools + system]**. Edit the system prompt but not the tools, and on the next call entry ② **misses** (system changed) while entry ① **still hits** (tools unchanged) — so you *keep* the tools cached instead of reprocessing them. With only **one** breakpoint (at end of `system`), that same edit misses the lone entry and you repay for the tools too. Think **save points in a game**: more valid saves = reload from the latest one that still works, rather than starting over. "Using 2 of your 4" just means you placed 2 of the 4 allowed bookmarks, leaving 2 spare (e.g. for stable points in a long, growing conversation).

> **What does `"type": "ephemeral"` mean?** *Ephemeral* = **short-lived / temporary** — it tells the API this cache isn't permanent and should **expire** after its TTL (5 min default, or 1 h), after which the next call rewrites it. It describes the **lifetime**, not which portion. **It's the only cache type that exists right now** — so you always write `{"type": "ephemeral"}`. Don't confuse *type* with *duration*: the 5-min-vs-1-hour choice is the separate **`ttl`** field *inside* the object — `{"type":"ephemeral","ttl":"1h"}` — not a second type.

**Worked example — one breakpoint caching `tools` + `system` together.** A single `cache_control` on the **last `system` block** captures the whole static prefix *before* it — and because the order is `tools` → `system` → `messages`, that includes **all the tool definitions** plus every system block. Only the user's question (in `messages`) stays outside the cache:
```python
resp = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=1024,
    tools=[                                            # ┐
        {"name": "get_order_status",                   # │ tools — cached
         "description": "Look up an order by its ID.",  # │ (they come first
         "input_schema": {"type": "object",            # │  in the prefix order)
             "properties": {"order_id": {"type": "string"}},
             "required": ["order_id"]}},
    ],                                                 # ┘
    system=[
        {"type": "text", "text": "<long static support instructions…>"},   # cached
        {"type": "text", "text": "<entire 50-page returns policy…>",
         "cache_control": {"type": "ephemeral"}},      # ◄── breakpoint = end of cached prefix
    ],
    messages=[{"role": "user", "content": "Where is order A-5521?"}],  # NOT cached (after breakpoint)
)
```
*First call (cold — writes the cache):* `usage` shows the tools + both system blocks under **`cache_creation_input_tokens`** (billed at the 1.25× write rate), the question under **`input_tokens`**, and `cache_read_input_tokens: 0`.
```jsonc
"usage": { "cache_creation_input_tokens": 9800,  // tools + system written once
           "cache_read_input_tokens": 0,
           "input_tokens": 12 }                    // "Where is order A-5521?"
```
*Every later call within the TTL (warm — reads the cache):* the same prefix now comes back as **`cache_read_input_tokens`** at the cheap 0.05× rate, and only the new question is fresh input:
```jsonc
"usage": { "cache_creation_input_tokens": 0,
           "cache_read_input_tokens": 9800,        // tools + system served cheap
           "input_tokens": 14 }                     // the next user question
```
> Want the tools cached under their **own** entry (e.g. tools are stable but you tweak the system prompt)? Add a **second** breakpoint on the **last tool** as well — that's 2 of your 4 breakpoints, one ending the `tools` prefix and one ending the `system` prefix.

**Why order matters:** the cache matches a **prefix**, so a single changed byte **early** in the prompt misses the *whole* cache — that's why static content goes **first** (tools → system → long docs) and dynamic/user content goes **last**. Anything after the last breakpoint is billed as normal input.

**Lifetime (TTL):** **default 5 minutes**; a **1-hour** option via `{"type":"ephemeral","ttl":"1h"}` (costs more to write). **Every hit refreshes the clock for free**, timed from the **start** of the request — so a steady stream of calls keeps a cache alive indefinitely. Bonus: **cache hits don't count against your rate limit.**

**Economics (verified multipliers vs base input price):**

| Operation | Multiplier |
|---|---|
| **Cache write, 5-min TTL** | **1.25×** base input |
| **Cache write, 1-hour TTL** | **2×** base input |
| **Cache read (hit)** — most models | **0.1×** base input |
| **Cache read (hit)** — Opus 5.5 / Sonnet 5.5 | **0.05×** |
| **Cache read (hit)** — Fable 5.1 / Mythos 5.1 | **0.025×** |

→ Writing costs a **bit more** than normal input *once*; **reading is ~10–40× cheaper**. So you only come out ahead if the prefix is **reused enough** to pay back the write — roughly from the **2nd–3rd reuse** within the TTL.

**Minimum size to cache (verified, per-model).** A prefix shorter than this simply **won't cache — and you get no error** (it's just billed as normal input):

| Min tokens | Models |
|---|---|
| **512** | Fable/Mythos 5.1, Opus 5.5, Opus 5, Sonnet 5.5, Fable/Mythos 5, Haiku 5.5 |
| **1,024** | Opus 4.8, Sonnet 5, Sonnet 4.6 |
| **2,048** | Opus 4.7, Mythos Preview |
| **4,096** | Opus 4.6, Opus 4.5, Haiku 4.5 |

**How to *prove* it worked (usage fields).** The response's `usage` tells you exactly what the cache did:
- **`cache_creation_input_tokens`** — tokens **written** to the cache (a first write / miss, billed at the 1.25×/2× write rate). Breaks down into `ephemeral_5m_input_tokens` + `ephemeral_1h_input_tokens`.
- **`cache_read_input_tokens`** — tokens **served from** the cache (a hit — the cheap 0.1×/0.05×/0.025× rate).
- **`input_tokens`** — tokens **after** the last breakpoint (neither read nor written).
- `total_input = cache_read_input_tokens + cache_creation_input_tokens + input_tokens`.
  - *Why `cache_creation` is in the total:* writing the cache **doesn't make those tokens free** — on the cold call the model still **reads and processes** the whole prefix *that turn* (it just also stores a copy). So they're genuine input, billed at the write rate and put in their own bucket. The three fields are mutually-exclusive buckets sorted by **how each token was billed** (read-rate / write-rate / normal), so summing all three = every input token the request consumed. The **same** prefix tokens sit in `cache_creation` on the first call (written) and in `cache_read` on every later call (reused).

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
**Plain idea:** if you have a *huge pile* of requests and don't need the answers right this second, hand them all to the API at once, walk away, and collect the results later — in exchange for **50% off** everything. You're trading **speed for price**.

- **How:** create with `client.messages.batches.create(requests=[…])`; each request carries its own params and a **`custom_id`** (must match `^[a-zA-Z0-9_-]{1,64}$`, unique in the batch). You then **poll** for status and download results when it's done.
- **Cost (verified):** **all usage at 50% of standard prices** — flat half-price on both input and output.
- **Timing (don't misread this):** fully **asynchronous**; **most batches finish in under 1 hour**. The **24 hours is a hard expiry cap, not a target** — you get results once *all* requests finish **or** at 24h, whichever comes first, and **anything not done by 24h expires** (expired/errored/cancelled requests aren't billed).
- **Size cap (verified):** **100,000 requests or 256 MB**, whichever it hits first. Each request needs `max_tokens ≥ 1`.
- **Results kept 29 days** after creation; batches are **scoped to a Workspace**.
- **Not supported inside a batch (verified):** streaming (`stream:true`), **Fast mode** (`speed`), server-side **`fallbacks`**, and `max_tokens:0`. Also **not ZDR- or HIPAA-eligible**.
- **Use for:** latency-tolerant bulk jobs — **large-scale evals, content moderation at volume, data analysis/summaries, bulk content generation**. **Not for** anything interactive/real-time.
- **Stacks with caching** (the two discounts combine) — but cache hits inside a batch are **best-effort** (observed ~**30–98%**): keep every request's `cache_control` block **identical** and consider the **1-hour TTL** so the shared prefix survives the whole batch.

> **Trap:** "results within 24 hours" is the **expiry deadline**, not a speed promise — most finish far sooner, but a batch is **async** and must never sit on an interactive path.

### 2.3 Model routing
**Plain idea:** instead of sending *every* request to one model, put a **cheap model at the door** to sort them, and only **escalate the hard ones** to a bigger model.
- Typical split: **Haiku classifies difficulty / handles the easy ~80%**; **Sonnet/Opus take the hard ~20%**.
- You get **Haiku's price on easy inputs** and **Opus's quality on hard ones** — better than paying one flat tier for everything.
- The router itself is a small, well-scoped **classification call** — a perfect Haiku job, so it adds little cost. (This is the per-request escape from the §1.2 triangle.)

### 2.4 Reasoning levers: `effort` and thinking modes
On current models **thinking is always on**, so the question isn't *"should it think?"* but *"how much?"* — and that buys more reasoning **without changing tiers**. Reach for these **before** upgrading the model.

**`effort` — the main reasoning dial (verified):**
- Five levels: **`low` < `medium` < `high` < `xhigh` < `max`**. Set it with **`output_config: {"effort": "..."}`**.
- **Defaults:** **`medium`** on **Opus 5.5** and **Haiku 5.5**; **`high`** on **Sonnet 4.6** (and Opus 5). The docs call it *"the main control for how much Claude thinks… the first setting to adjust when trading off intelligence, latency, and cost."*
- Nudge **up** for a hard case (may rescue a cheaper tier); drop it **down** to cut cost + latency on easy traffic — *"lowering effort reduces thinking… more reliably than prompt instructions do."* Cheaper to try than a tier jump: same model, same prices, just more/less internal work.
- **Trap — names aren't comparable across models:** Opus 5.5 at `medium` *matches or beats* Opus 5 at `high`. So set it per-model against **your own evals**; don't carry a setting over. Reserve **`xhigh`/`max`** for cases where you've *measured* a quality gain.
- **Cache trap:** changing the top-level `effort` **between requests invalidates the prompt cache**; to change it mid-conversation use the **per-message effort change (beta)**, which keeps the cache.

**Thinking modes — adaptive (current) vs manual (deprecated):**
- **Adaptive thinking (current, recommended):** `thinking: {"type": "adaptive"}` with **no `budget_tokens`** — Claude **decides per request whether to think and how much**, and at low effort it may **skip thinking entirely on easy inputs**. Depth is set by **`output_config.effort`** (above). It interleaves between tool calls automatically (no beta header).
- **Manual extended thinking is DEPRECATED (verified):** `thinking: {"type":"enabled","budget_tokens": N}` — **Claude 4.7 and later reject it with a 400 error.** It lingers only for workloads needing *predictable latency / precise cost control* on older models (4.6 and earlier), where it set a thinking-token budget (**min 1,024**, must be **< `max_tokens`**).
- **Billing & window (either mode):** thinking tokens are billed as **output** and count toward `max_tokens` and the context window — track them via `usage.output_tokens_details.thinking_tokens` (Note 06 §4.1). For very deep thinking (budgets/effort in the **>32k** range) use **batch processing** to avoid timeouts.
- **Tool-use loops — pass thinking back *verbatim* (verified):** to continue a tool-use turn with thinking on, echo the assistant turn back **unmodified — the full `content` array, including `thinking` *and* `redacted_thinking` blocks.** Modifying or dropping them → **400 `invalid_request_error`** ("thinking blocks cannot be modified"), and resending the same body fails again (it's a deterministic 400 — fix the body, don't retry). A filter that keeps only `thinking`/`text`/`tool_use` silently drops **`redacted_thinking`** (present only when part of the reasoning is safety-redacted → a *rare, intermittent* failure). **Preserve exactly; don't strip.**

> **Order of escalation for a too-hard task:** raise **`effort`** → (older models only) raise the manual **thinking budget** → only then **upgrade the tier**. Each step costs more, so climb only as far as the evals require.

### 2.5 Other optimisation levers
- **`max_tokens` tuning** — cap how long the output can get; shorter outputs = **lower cost & latency** (output tokens usually cost more than input). It's also the **hard ceiling** on a turn — thinking tokens count against it.
- **Prompt efficiency** — trim redundant context; shorter inputs cost less (but keep what quality needs).
- **Streaming** — makes the reply *feel* faster (tokens appear as they're generated) but **does not reduce cost or total generation time** — it improves **perceived** latency only.
- **Choose the smallest model that passes evals** — the headline lever (see §1.3).
- **Long outputs → stream or batch** so they don't hit the SDK's **10-min** non-streaming timeout (Note 02 §7). Not a cost lever — a *reliability* one.

### 2.6 Service tier / Priority Tier (reliability & overload lever)
When the problem is **"we get `529 overloaded` / latency spikes at peak,"** the lever isn't a smaller model — it's **capacity tier** (verified):
- **`service_tier` request param:** **`"auto"`** (default — use Priority Tier capacity if available, else fall back to standard) or **`"standard_only"`** (never touch Priority capacity). The response reports what it got in **`usage.service_tier`** (e.g. `"priority"`).
- **Priority Tier** = committed input/output tokens-per-minute capacity that the API **prioritises over other requests**, minimising "server overloaded" errors even at peak (targets ~99.5% uptime). It's a **reliability/throughput** guarantee, **not** a latency SLA. Over-capacity requests **fall back to standard**.
- **Buying it:** new Priority Tier **commitments are no longer sold** (existing contracts run to term; "contact sales" for guaranteed capacity) — so on the exam treat it as the **concept**: the way to harden against overload is **committed/priority capacity + `service_tier`**, plus graceful **429/529 backoff** (Note 02 §7), not a model swap.

> **Exam cue:** "peak-time `529`s / need reliable throughput" → **Priority Tier / `service_tier`** (+ backoff), *not* "use Haiku." Model choice is about capability/cost/latency; **tier is about capacity/reliability.**

---

## 3. Putting it together
*Pick the model (§1), stack the levers (§2) — these two subsections help you choose which lever for which goal.*

### 3.1 Latency vs cost (they're different)
Some scenarios ask about **speed**, not money — don't conflate:
- **Lower latency:** smaller tier (**Haiku**), **shorter outputs** (`max_tokens`), **prompt caching** (less prefix to process), **streaming** (perceived), and **lower `effort`** to cut unnecessary thinking.
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
| A bit more reasoning, same tier | **raise `effort`** (adaptive thinking does the rest) |
| Hard reasoning on a mid tier | **raise `effort`** / `thinking:{type:"adaptive"}` (manual `budget_tokens` is deprecated) |
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
- Q: Cheaper lever to try *before* switching models? **A:** **Tune `effort`** (levels `low`→`max`; default `medium` on Opus 5.5 / Haiku 5.5, `high` on Sonnet 4.6) via `output_config`.
- Q: Flag to cache a block? **A:** `cache_control: {"type":"ephemeral"}` (add `"ttl":"1h"` for 1 hour).
- Q: Default cache TTL? **A:** **5 minutes** (refreshed free on hit); **1-hour** option available.
- Q: Cache write vs read cost? **A:** Write **1.25×** (5-min) / **2×** (1-hour); read **0.1×** base — **0.05×** on Opus 5.5/Sonnet 5.5, **0.025×** on Fable/Mythos 5.1.
- Q: Field proving a cache hit? **A:** **`cache_read_input_tokens`**.
- Q: Where must dynamic content go relative to a cached prefix? **A:** **After** it (cache matches a prefix: `tools`→`system`→`messages`).
- Q: Batch API discount, timing & size cap? **A:** **50% off**, **async** (most **<1h**, **24h** = expiry cap), **100k requests / 256 MB** max; results kept **29 days**.
- Q: Does streaming cut cost? **A:** **No** — only perceived latency.
- Q: How to get Opus quality on hard inputs but Haiku cost on easy ones? **A:** **Routing** (triage cheap, escalate hard).
- Q: Lever to boost reasoning without jumping to Opus? **A:** **Raise `effort`** (with adaptive thinking); manual `budget_tokens` extended thinking is **deprecated** (4.7+ return 400).
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
- Q: What's the cost of more thinking (higher `effort`)? **A:** **More tokens + latency** (thinking tokens billed as output) — use it only where reasoning quality pays; it can let a **mid tier** succeed instead of jumping to Opus.
- Q: Adaptive vs manual thinking? **A:** **Adaptive** (`thinking:{type:"adaptive"}`, depth via `effort`) — Claude decides whether/how much to think, skipping it on easy inputs; **manual** (`type:"enabled"`+`budget_tokens`) is **deprecated**, rejected by 4.7+ with a 400.
- Q: Is the **router** itself an expensive call? **A:** **No** — it's a cheap, well-scoped **classification** call (a great **Haiku** job).
- Q: Does a cache **hit** refresh the TTL? **A:** **Yes** — each hit refreshes the 5-min (or 1-hr) TTL **for free**.
- Q: Peak-time `529 overloaded` errors / need reliable throughput — which lever? **A:** **Capacity tier**, not a smaller model: **Priority Tier** (committed capacity, prioritised → ~99.5% uptime) selected via **`service_tier: "auto"`** (`standard_only` to opt out), plus **429/529 backoff**. Tier = capacity/reliability; model = capability/cost/latency.

**One-line summary:** *Right-size the tier (Haiku → Sonnet → Opus → Fable ceiling) **by evals**; tune **`effort`** (adaptive thinking; manual `budget_tokens` is deprecated) before switching models; **cache** static prefixes (write 1.25×/2×, read 0.1×/0.05×/0.025×, hit = `cache_read_input_tokens`), **batch** bulk async (50% off, most <1h / 24h cap), **stream** for feel (not cost), and **route** easy vs hard inputs.*