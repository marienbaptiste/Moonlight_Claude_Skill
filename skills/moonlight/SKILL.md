---
name: moonlight
description: "Ideation-to-evaluation pipeline. Takes raw ideas, runs intake interviews + market research + multi-agent evaluation, and produces ranked, sourced executive summaries in the user's Notion workspace."
---

# Moonlight Skill

Moonlight is a structured ideation-to-evaluation pipeline. It takes raw ideas, interviews the user, researches the market, runs multi-agent evaluation, and produces ranked, sourced executive summaries in Notion.

The skill is **portable across users** — no hardcoded workspace IDs. On first run, it discovers or creates the workspace structure inside the user's own Notion. All IDs are cached in a per-user config so subsequent runs are zero-setup.

---

## Prerequisites

The user must have the **Notion connector** authorized in Claude (so `notion-fetch`, `notion-search`, `notion-create-pages`, `notion-create-database`, `notion-update-page`, `notion-update-data-source` are available). If these tools aren't available, stop and tell the user how to enable Notion in their Claude client, then retry.

---

## Phase 0: Resolve Workspace (run on every invocation, exit early when possible)

Before doing anything the user asked for, ensure you know the workspace IDs. The config lives at `~/.claude/moonlight/config.json` (POSIX) / `%USERPROFILE%\.claude\moonlight\config.json` (Windows).

### Step 0.1 — Try config first

Read the config file. If it exists and has all four IDs (`root_page_id`, `resources_page_id`, `changelog_page_id`, `ideas_db_id`) and `schema_version`, you're done — proceed to the user's actual request.

Config shape:
```json
{
  "schema_version": 1,
  "root_page_id": "...",
  "resources_page_id": "...",
  "changelog_page_id": "...",
  "ideas_db_id": "..."
}
```

### Step 0.2 — Search the workspace

If config is missing or incomplete, run `notion-search` for `"Moonlighter's Den"` (page title). If a page is found:

1. Fetch it with `notion-fetch` to enumerate children
2. Identify the Resources page (icon 🧰 or title "Resources"), Changelog page (icon 📋 or title "Changelog"), and Ideas Database (inline database)
3. Extract their IDs (and the data source ID for the database — it's the `collection://` URL inside the `<database>` tag)
4. Write the config file
5. Proceed to user's request

### Step 0.3 — Onboard (create the structure)

If nothing is found in the workspace, ask the user:

> "I don't see a Moonlight workspace in your Notion yet. Where should I create it?
> Paste a Notion page URL to use as the parent, or reply `root` to create at the workspace root.
> I will only create new pages — I will not modify any existing content."

Once they reply, create the structure programmatically:

1. **Root page** — title `Moonlighter's Den`, icon `🌙`, body from `assets/root_page.md`. Title must be plain text — set the emoji via the `icon` field, never embedded in the title.
2. **Resources child page** — title `Resources`, icon `🧰`, body from `assets/resources_template.md`.
3. **Changelog child page** — title `Changelog`, icon `📋`, empty body (will be appended to over time).
4. **Ideas Database** — created inline on the root page. **Use the SQL DDL schema in `assets/ideas_db_schema.sql` verbatim.** The Notion `create-database` tool takes a `schema` parameter that expects a `CREATE TABLE` statement (not JSON). Read the `.sql` file and pass its full content as the `schema` parameter, with `title: "Ideas Database"`, icon `📊`, and the parent set to the root page id.

The schema includes 19 columns: 16 inputs + 3 FORMULA columns (RICE Score, Opportunity Score (/5), Verdict). **Do not omit any columns.** Do not translate the schema into JSON or rewrite it — pass the SQL DDL string as-is to preserve the formula expressions exactly.

If the call fails, paste the exact error message to the user and stop. Do not silently drop columns or "fall back" to a partial schema. The 3 formulas are core to the skill — without them, scores don't compute.

The `ideas_db_schema.json` file in `assets/` is for documentation only — human-readable reference for the property structure. The `.sql` file is the executable schema.

After creation:
- Save all four IDs and `schema_version: 1` to `~/.claude/moonlight/config.json`
- Confirm to the user with a link to the new root page
- Append a Changelog entry: `## [YYYY-MM-DD] Workspace initialized` with type `Setup`
- Proceed to the user's original request

### Step 0.4 — Schema migrations (future)

If `config.schema_version` is less than the current schema version (in `assets/ideas_db_schema.json`), run any defined migration steps before continuing. (No migrations defined yet — this is a forward-compatibility hook.)

---

## Workflow: Handling /moonlight Requests

After Phase 0 completes, detect the request type and act:

| Request type | Action |
|---|---|
| New idea submission | Intake Interview → **Load User Profile from Resources** → Research → Multi-Agent Eval → Write Summary → Update DB |
| Evaluate/re-rank existing ideas | Fetch DB → **Load User Profile from Resources** → Re-run scoring with current profile → Update rankings |
| Market research on a topic | **Load User Profile from Resources** → Run research phase only → Report findings (call out user-specific implications) |
| Update resources | Fetch Resources page → guide user through updates → confirm before save |
| Review pipeline | Fetch DB → summarize current state, flag stale ideas |

**Resources/User Profile is loaded BEFORE every evaluation, every re-rank, every research run.** Skipping this step produces generic scores that ignore the user's actual constraints. See Phase 1.5 for the load procedure.

**Always fetch before editing.** Notion pages change between sessions — never rely on cached state. When you need any of the four pages/DB, fetch by ID from config.

---

## Phase 1: Intake Interview

When the user submits a new idea, run them through questions until you have enough to evaluate. Ask questions in batches of 2–3, not one at a time. Stop when all critical fields are covered.

### Required fields to capture

**Core**
- One-line pitch (what is it?)
- Problem statement (what pain does it solve?)
- Target user / customer segment
- How the user discovered this pain point (personal experience, observation, request?)

**Market**
- Known competitors or alternatives
- Why existing solutions fail or fall short
- Willingness to pay (is this a "hair on fire" problem?)

**Execution**
- Revenue model (SaaS, one-time, marketplace, ads, etc.)
- MVP scope — what's the absolute minimum to validate?
- Tech stack preferences or constraints
- Time the user can dedicate (hours/week)

**Ambition**
- Target scale (lifestyle biz, venture-scale, side income?)
- Any unfair advantage (domain expertise, existing audience, proprietary data?)

### Adaptive questioning
- If the user gives a detailed pitch, skip questions they've already answered
- If the idea is vague, ask more probing questions to sharpen it
- Always confirm the final captured summary before proceeding

### Resource check
Fetch the Resources page (id from config). If it's still the unfilled stub (placeholders like `[number]` / `[e.g., ...]`), tell the user: *"Your Resources page is empty. Evaluation will use generic assumptions — scores will be less tailored. Want to fill it in now (5 min) or proceed?"* Wait for their answer. Don't block; their choice.

---

## Phase 1.5: Load User Profile (mandatory before evaluation)

Before Phase 3 runs, extract a structured **User Profile** from the Resources page content. This is the one source of truth that every agent in Phase 3 will reference. Hold it in conversation context for the rest of the run.

```
USER PROFILE
- Technical skills: <list, or "unknown" if blank>
- Domain expertise: <list, or "unknown">
- Hours/week available: <number, or "unknown">
- Duration commitment: <e.g., 3 months, or "unknown">
- Monthly budget cap: <$, or "unknown">
- One-time investment cap: <$, or "unknown">
- Preferred tech stack: <list, or "unknown">
- Existing audience: <description, or "none">
- Existing assets: <domains, codebases, datasets, network — or "none">
```

If a field is "unknown", every agent that depends on it must flag the gap explicitly in its output (e.g., "Effort estimate assumes typical mid-level full-stack capability — actual effort depends on user's skill profile, which is unset"). Don't fabricate values.

---

## Phase 2: Market Research

After intake, conduct live research to validate the pain point. Every claim in the final summary must be grounded.

### Research checklist

1. **Reddit search** — Search relevant subreddits. Look for:
   - Frequency of complaints (how many posts?)
   - Emotional intensity (frustrated? desperate? mildly annoyed?)
   - Existing workarounds people describe
   - Willingness to pay signals ("I'd pay for...", "shut up and take my money")

2. **Competitor analysis** — Search for existing solutions:
   - Direct competitors (same problem, same approach)
   - Indirect competitors (same problem, different approach)
   - Pricing tiers, reviews, complaints
   - Gaps in their offerings

3. **Market sizing** — Estimate TAM, SAM, SOM:
   - **TAM** — total addressable market (industry/category size)
   - **SAM** — serviceable addressable market (segment you can realistically reach)
   - **SOM** — serviceable obtainable market in users/month (what's realistic in year 1)
   - Use industry reports, competitor public numbers, adjacent market data when niche is too small for direct data

4. **Technical feasibility signals** — Search for:
   - Open-source tools/APIs that could accelerate development
   - Technical barriers others have encountered
   - Cost of key infrastructure (APIs, hosting, data)

### Research standards
- Use `web_search` and `web_fetch` for every claim
- Record the source URL for every datapoint
- Note the date of each source
- If a claim can't be sourced, mark it as `[UNVERIFIED]` and flag it

---

## Phase 3: Multi-Agent Evaluation

Run the idea through 4 expert perspectives. Each agent produces a structured assessment.

**Every agent must explicitly reference the User Profile from Phase 1.5.** If a relevant profile field is "unknown", call it out in the output. Don't silently assume defaults.

### Agent 1: Business Analyst 📊
Evaluate:
- **TAM / SAM / SOM**: market sizing in dollars and users/month, sourced
- **Impact** (1–3): Minimal, Significant, Transformative
- **Revenue potential** (1–5): 1=<$1k/mo ceiling, 3=$5–20k/mo, 5=$100k+/mo
- **Time to first $10k**: months — **factor in user's `Hours/week available` and `Existing audience` from User Profile** (an existing audience compresses time-to-revenue dramatically)
- **Market timing**: is this the right moment? (growing trend, regulatory change, tech enabler?)
- **Competitive window** (1–5): how long before incumbents copy or new entrants saturate

Output: 3–5 bullet assessment with scores, sources, and any User Profile gaps that affected the estimate.

### Agent 2: CTO 🏗️
Evaluate:
- **Confidence** (0.5 / 0.8 / 1.0): technical feasibility — **drop to 0.5 if user's `Preferred tech stack` and `Technical skills` don't cover what this idea needs**
- **Effort (weeks)**: person-weeks for MVP — **factor in user's `Hours/week available` (a 30hr/wk user delivers ~2× faster than a 10hr/wk user) AND `Technical skills` (skill gaps add weeks for learning curve or contractor hire)**
- **Skill fit/gap**: explicitly compare what this idea needs vs. user's `Technical skills`. Name the gap if any. If `Monthly budget cap` is too low to hire for the gap, flag it.
- **Tech risk**: any hard technical unknowns?
- **Time to market**: calendar weeks to a usable MVP, given user's hours/week
- **Scalability**: will the architecture hold if it works?

Output: 3–5 bullet assessment with effort estimates, skill gap callouts, and any User Profile gaps.

### Agent 3: CPO 📋
Evaluate:
- **MVP definition**: smallest thing that validates the hypothesis — **scope to fit user's `Duration commitment` and `Hours/week available`**
- **User journey**: key touchpoints from discovery to retention
- **Rollout plan**: Phase 1 (validate) → Phase 2 (grow) → Phase 3 (monetize) — **timeline must respect user's Duration commitment**
- **Key metrics**: what to measure at each phase
- **Timeline**: week-by-week for Phase 1, month-by-month for Phases 2–3, calibrated to user's hours/week

Output: phased plan, 1 page max. Flag if scope can't realistically fit the user's available time.

### Agent 4: Marketing Expert 📣
Evaluate:
- **Positioning**: one-line positioning statement
- **Channel strategy**: top 3 channels ranked by expected ROI **given user's `Monthly budget cap` and `Existing audience`** (e.g., if user has 5k newsletter, that's free distribution; if budget cap is $0, paid ads drop in priority)
- **Launch plan**: pre-launch, launch week, post-launch — **must fit budget cap**
- **Content strategy**: what content validates demand before building?
- **Campaign effort**: hours/week and budget needed — **compare to user's `Hours/week available` and `Monthly budget cap`; flag any overrun**
- **Mock assets**: describe (don't generate) 2–3 key marketing assets — landing page headline, ad copy angle, social post hook

Output: channel-ranked plan with effort estimates, budget fit explicitly noted.

---

## Phase 4: Scoring & Ranking

The Notion database has three formula properties that compute scores from the inputs you write:

- **RICE Score** — formula in Notion, derived from `SOM (users/mo)`, `Impact`, `Confidence (/1.0)`, `Effort (weeks)`
- **Opportunity Score (/5)** — composite weighted score combining RICE, Moat, Unfair Advantage, Revenue Potential, Time to $10k
- **Verdict** — six tiers based on Opportunity Score thresholds: ≥4 `🔥 Strong Go`, ≥3.5 `🟢 High Conviction`, ≥3 `✅ Side Bet`, ≥2.5 `🔶 Conditional Go`, ≥2 `🟡 Weak Maybe`, else `🚫 Hard No`

You write the **inputs**; Notion computes the scores. Inputs to capture per idea:

| Input | Type | Scale / Notes |
|---|---|---|
| TAM | Number | Total addressable market ($) |
| SAM | Number | Serviceable addressable market ($) |
| SOM (users/mo) | Number | Serviceable obtainable market — users/month, year 1 realistic |
| Impact | Select | `Minimal (1)`, `Significant (2)`, `Transformative (3)` |
| Confidence (/1.0) | Number | 0.5 (speculative), 0.8 (reasonable), 1.0 (strong data) |
| Effort (weeks) | Number | Person-weeks for MVP |
| Moat (/5) | Number | Defensibility: 1=none, 5=deep (network effects, data, proprietary tech) |
| Unfair Advantage (/5) | Number | User's specific edge: 1=none, 5=massive |
| Revenue Potential (/5) | Number | 1=<$1k/mo ceiling, 5=$100k+/mo |
| Time to $10k (months) | Number | Estimated calendar months to $10k cumulative revenue |
| Competitive Window (/5) | Number | 1=incumbents will crush, 5=long defensible window |

If the formula columns are unset (e.g., they failed to create during onboarding), compute the scores yourself in the executive summary text and tell the user to add the formulas.

---

## Phase 5: Write to Notion

### Per-idea page

Each idea is a row in the Ideas Database. Set all input properties listed above. The formula properties auto-compute. Also set:
- **Idea** (title) — short noun phrase (the one-line pitch becomes the page body, not the title)
- **Status** — start with `💡 New`, move to `🔍 Researching` during research, `✅ Evaluated` after summary written
- **Origin** — `AI` or `Human` based on who proposed it
- **Category** — pick the closest fit
- **Date Added** — today

### Executive summary (page body)

Each idea's row has a child page body — write the executive summary there.

```markdown
**One-line pitch:** [pitch]
**Submitted:** [date]
**Status:** [status]

---

## The Problem
[2–3 sentences. What's broken, for whom, and why it matters. Every claim sourced.]

## Market Validation
[Key findings from Reddit/search research. Quotes from real users (anonymized). Competitor gaps. All sourced.]

## Solution & MVP
[What to build first. Why this scope. What it deliberately excludes.]

## Market Sizing
| | Value | Source |
|---|---|---|
| TAM | $X | [url] |
| SAM | $Y | [url] |
| SOM | Z users/mo | [reasoning] |

## Scoring Inputs
| Input | Value | Rationale |
|---|---|---|
| Impact | [1-3] | [reasoning] |
| Confidence | [0.5-1.0] | [reasoning] |
| Effort (weeks) | [n] | [reasoning] |
| Moat | [1-5] | [reasoning] |
| Unfair Advantage | [1-5] | [reasoning] |
| Revenue Potential | [1-5] | [reasoning] |
| Time to $10k (months) | [n] | [reasoning] |
| Competitive Window | [1-5] | [reasoning] |

## Expert Assessments

### Business Analyst
[3–5 bullets]

### CTO
[3–5 bullets]

### CPO — Rollout Plan
[Phase 1/2/3 summary, key milestones]

### Marketing
[Channel strategy, launch plan, effort estimate]

## Risks & Mitigations
[Top 3 risks, each with a mitigation]

## Verdict
[2–3 sentences. Go/no-go recommendation with reasoning. Should align with the Verdict formula output.]

---
*Sources: [numbered list of all URLs referenced]*
```

### Length & tone rules (non-negotiable)
- Executive summary: 1–2 pages max (~500–1000 words). No exceptions.
- No filler: ban "It's worth noting", "Leveraging", "Seamlessly", "In today's landscape", "This ensures", "comprehensive solution"
- Every factual claim needs a source. No source = `[UNVERIFIED]` tag.
- Direct, opinionated. If an idea is weak, say so. Don't hedge everything.
- Tables over prose for structured data.
- No nested bullets deeper than 2 levels.

---

## Changelog Management

After every meaningful change, fetch then prepend to the Changelog page (id from config):

```markdown
## [YYYY-MM-DD] <Change Title>
**Type:** [Setup | New Idea | Evaluation | Re-rank | Resource Update | Cleanup]
**Pages affected:** <list>

<Description>
```

Newest entries first. Fetch before writing.

---

## Important Rules

1. **Always run Phase 0 first.** Never assume IDs — read config or discover/create.
2. **Always fetch before editing** — Notion pages change between sessions.
3. **Always log to Changelog** — every change, no matter how small.
4. **Never fabricate market data** — if you can't find it, say so.
5. **Source everything** — URL + date for every factual claim.
6. **Respect the 1–2 page limit** — ruthlessly cut fluff.
7. **Ask before killing ideas** — never archive without user confirmation.
8. **Resources page is context** — always check it before evaluating.
9. **Be opinionated** — the user wants honest signal, not diplomatic hedging.
10. **Create missing structure** — if the DB or pages don't exist, run Phase 0 onboarding.
11. **Rank the database** — after every new evaluation, re-sort by Opportunity Score.
12. **Icons go on the page, not in the title** — set emoji via the `icon` field only. Titles must be plain text (e.g., `icon: "📊"`, `title: "Ideas Database"` — never `title: "📊 Ideas Database"`). Applies to all page creation and updates.
13. **Never modify the user's existing Notion content during onboarding** — only create new pages under the parent they specified.
