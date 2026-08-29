---
name: lead-prospecting
description: Build a CRM-ready B2B lead list through taskfuel.ai — discover companies matching an ideal-customer profile, then find the right decision-maker at each, paid per call from the prepaid balance. Use when the user wants sales leads, a target account list, or contacts at companies they name.
license: MIT
---

# Lead Prospecting

ICP → companies → the right person at each, as a CSV of **leads** (ICP-matched contacts,
not qualified prospects). Email lookup and deliverability are the companion
`email-verification` skill, handed off after Step 3. **Never drafts or sends outreach.**

## Prerequisites

- `taskfuel` CLI connected (`taskfuel whoami`), balance ≥ ~$0.35 for a full discover run.
- Base `taskfuel` skill covers payment mechanics — quote with `--dry-run`, pay with
  `--max-amount`, never suppress stderr on a paid call. Read it first if you haven't.

## Rules

1. **Prompt, never guess.** Ask for each step's parameters with examples so answering takes
   one word. Guessed ICPs and job functions return wrong results or none.
2. **Announce cost, then wait for OK.** Never chain straight into a paid call. Confirm before
   any step adds >~50 rows.
3. **Seller-agnostic — no built-in ICP.** Fit criteria come only from the user's own words;
   never assume what is being sold.

## Intake

- **Entry point:** *discover* (Step 1), or *bring-your-own domains* — skip Step 1, seed the
  table from the pasted domains, start at Step 2.
- **In discover mode, pick the route from the shape of the ICP — never ask the user to name
  a tool:**
  - **Categorical / numeric** (an industry, a country, a headcount band) → **route A**.
  - **Thematic / semantic** ("customer-support automation", "cross-border payroll") → **route B**.
    FullEnrich's 490-value industry enum has no agent/AI entry — its documented stand-ins
    (`Software Development` for SaaS, `Research Services` for AI labs) are far wider than the theme.
  - Both kinds given → route B to find them, then filter on the numbers client-side.
- **Fit criteria (optional):** *"What are you selling, and what makes a company a good fit?"*
  Free text. Blank ⇒ skip Step 1b and present unranked.

## Deliverable

`leads-YYYY-MM-DD.csv`, written at the first data step and enriched in place. **Nothing
discarded** — one row per person, plus a placeholder row per contactless company. `Selected?`
marks the primary contact; the rest stay as fallbacks. All rows go to the CSV; show only a
top slice in chat.

`Company | Domain | Fit | Fit reason | Person | Title | Seniority | LinkedIn URL | Selected? | Status`

`email-verification` appends `Email | Email source | Verified? | Score` on handoff, and keeps
writing the existing `Status` column rather than adding one — leave the four new columns
absent until then rather than writing them empty.

`Status`: `company-found` → `duplicate` | `no-contact-found` | `candidate` → `selected`.
The companion skill continues this same enum from `selected`, with `email-found` |
`email-constructed` → `verified` | `email-unconfirmed` | `verify-failed`.

## 1 — discover companies · *skipped in BYO-domains mode*

Intake picked the route. Parameters, response shape and the gotchas that cost real money live
beside this file — **read the one route you picked before calling it**:

- **Route A** · categorical/numeric ICP · $0.15/call, free when zero results →
  [`ROUTE-A.md`](ROUTE-A.md)
- **Route B** · thematic ICP · $0.01 flat → [`ROUTE-B.md`](ROUTE-B.md)

→ Company, Domain, HQ, headcount · `company-found`

## 1a — dedupe · free

- Same non-empty domain → keep the fullest, mark copies `duplicate` (park, don't delete).
- Cross-domain near-dupes (same name/HQ, different TLD) → **ask which to keep**; never
  auto-merge, since TLDs can be distinct entities.
- Empty domain → resolve by web search before Step 2.
- **After route B:** drop any platform host `excludeDomains` did not catch — app stores,
  review sites, link-in-bio pages. Unfiltered, these were 16 of 31 rows in testing (9 of them
  `*.notion.site`), so add each new offender to `excludeDomains` for the next call.

## 1b — score fit · free · *only if fit criteria given*

- The scoring text is already in the Step 1 response — **no extra call.** After route A use
  `description` + `specialties`; after route B use `results[].entities[].properties.description`
  plus the result `text`.
- To target the criteria directly, re-run route B with `contents.highlights.query` set to the
  fit criteria — it takes its **own** query and returns a per-company relevance snippet, still
  $0.01.
- Set **Fit** (`High`/`Med`/`Low`) and a one-line **Fit reason** citing the criteria.
- **Annotate, don't gate** — low-fit rows stay in the table; it is a heuristic over marketing copy.
- → Fit, Fit reason

## 2 — people-search · `POST https://stableenrich.dev/api/fullenrich/people-search` · $0.15/call, free when no matches

**Ask:** job function and/or seniority, and which domains.

- **Batch every domain into one `current_company_domains`** — same $0.15 as a single company.
  One-at-a-time multiplies cost by company count for no extra data.
- `current_position_job_functions` takes **canonical Function enums** (`Software` =
  engineering, `Sales`, `Product`, `Design`, `Marketing`, `Human Resources`, `Operations`,
  `Finance`, `Legal`, `Executive & Leadership`…). Full list:
  `taskfuel discover POST https://stableenrich.dev/api/fullenrich/people-search`.
- `current_position_titles` is **exact-match only** — "CTO" matches that literal string and
  returns far fewer rows. Prefer seniority or function.
- `current_position_seniority_level`: `Owner`, `Founder`, `C-level`, `Partner`, `VP`, `Head`,
  `Director`, `Manager`, `Senior`.
- **AND across categories, OR within one — use ≤2 categories.** Three or more often returns
  zero. Prefer domains plus one of seniority/function.
- Check `metadata.total` — a free early warning of pool size. Page with `offset`/`search_after`
  at another $0.15 each.
- Response is a **lean roster**: `people[].employment.current.{title,seniority}`, LinkedIn
  under `people[].social_profiles.professional_network`, company detail in a top-level
  `companies` map keyed by `company_id`, plus `headline`, `location` and `educations`. Add
  `include_employment_history` or `verbose` only if full career history is needed.
- **Expect most domains to return nobody** — 19 domains yielded 9 people at 6 companies. A thin
  roster is normal for small companies, not a failed call: mark the rest `no-contact-found`.
- **When FullEnrich is unavailable, or the run has ≲15 companies**, Exa substitutes at
  $0.01/company → [`PEOPLE-EXA.md`](PEOPLE-EXA.md).
- → Person, Title, Seniority, LinkedIn URL, country · `candidate` | `no-contact-found`

## 3 — select · free

- **Ask run mode first:** *single* (one picked company, cheapest) or *fan-out* (one contact
  per company; handoff cost scales at ~$0.021 each). All rows stay in the table either way —
  mode only sets how many advance.
- Then pick manually, or auto-select **most senior**: Owner/Founder/C-level > VP > Head >
  Director > Manager. In fan-out, ask the rule **once**, apply it to all, and show the planned
  table for a **single approval**.
- → Selected?=yes · `selected`

## Handoff to `email-verification` · ~$0.021 per contact

Finding and verifying the address is a separate skill. Invoke it with the selected rows; it
enriches this same CSV in place. Per contact:

```json
{"first_name": "Alex", "last_name": "Moreau", "domain": "example.com"}
```
returning `{"email", "email_source", "verified", "score", "status"}`.

- **Pass names exactly as they appear — never normalize diacritics on the way out.**
  Transliteration is language-dependent and belongs to the finder; a fixed `ü→u` rule produced
  a confirmed-undeliverable address in testing.
- **Hand off full surnames.** Recover a truncated one from the LinkedIn handle, or select a
  different contact — a bare initial cannot be looked up.
- State the cost before handing off: selected rows × ~$0.021.
