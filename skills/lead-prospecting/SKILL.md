---
name: lead-prospecting
description: Build a CRM-ready B2B lead list through taskfuel.ai — discover companies matching an ideal-customer profile by industry/size/geography or by a plain-language theme like "builds AI agents", then find the right decision-maker at each, paid per call from the prepaid balance. Use when the user wants sales leads or prospects, a target account list, or contacts at specific companies.
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
  - **Thematic / semantic** ("builds AI agents", "does cross-border payroll") → **route B**.
    FullEnrich's industry enum has no AI entry; such an ICP is inexpressible there at any price.
  - Both kinds given → route B to find them, then filter on the numbers client-side.
- **Fit criteria (optional):** *"What are you selling, and what makes a company a good fit?"*
  Free text. Blank ⇒ skip Step 1b and present unranked.

## Deliverable

`leads-YYYY-MM-DD.csv`, written at the first data step and enriched in place. **Nothing
discarded** — one row per person, plus a placeholder row per contactless company. `Selected?`
marks the primary contact; the rest stay as fallbacks. All rows go to the CSV; show only a
top slice in chat.

`Company | Domain | Fit | Fit reason | Person | Title | Seniority | LinkedIn URL | Selected? | Status`

`email-verification` appends `Email | Email source | Verified? | Score` on handoff — leave
those columns absent until then rather than writing them empty.

`Status`: `company-found` → `duplicate` | `no-contact-found` | `candidate` → `selected`
(the companion skill continues the enum from there).

## 1 · route A — company-search · `POST https://stableenrich.dev/api/fullenrich/company-search` · $0.15/call, free when zero results

*Skipped in BYO-domains mode.* **Ask:** industries, HQ locations, headcount band
(e.g. `Software Development` / `Zurich` / `1-50`).

- Needs ≥1 real filter (`names`, `domains`, `industries`, `headquarters_locations`,
  `keywords`, `specialties`); `offset` alone → 400. Arrays take plain strings or
  `[{"value":"…","exact_match":true}]`.
- `industries` must be an **exact enum value** — free text like "fintech" → 400.
- `names` matches the **legal entity, not the brand** (Anysphere ≠ Cursor).
- **Returns ≤10 companies per call; there is no `limit`.** Page with `offset` (≤10,000) or
  `search_after` — **each page is another $0.15.** Budget by pages, not rows. A zero-result
  query is free, so an over-narrow first attempt costs nothing.
- Headcount filters **server-side** (`headcount_min`/`headcount_max`), so every row is in-band.
- HQ matching is loose — cross-check `locations.headquarters`. When `headcount` (number) and
  `headcount_range` (string) disagree, **trust the number**.
- → Company, Domain, HQ, headcount · `company-found`

## 1 · route B — semantic discovery · `POST https://stableenrich.dev/api/exa/search` · $0.01 flat, any `numResults`

*Skipped in BYO-domains mode.*

```json
{"query":"<the ICP in plain words>","category":"company","numResults":40,
 "contents":{"text":{"maxCharacters":700,"verbosity":"compact"}}}
```

- **`category:"company"` is mandatory** — without it you get pages *about* companies
  (directories, review sites, app stores), not company sites.
- **`contents` is free** and is what populates `entities[]`. Always send it.
- Read each `entities[]` entry of `type:"company"` → `properties`: `name`, `description`,
  `foundedYear`, `workforce.total`, `headquarters.{city,country}`, `financials.fundingTotal`.
  Domain = the result `url` host.
- **No server-side filtering — filter client-side.** Only ~4/15 results landed inside a
  50–200 headcount band in testing, so **over-fetch ~4× the target**; at $0.01 flat that is
  still far below route A.
- **Never trust a lone headcount figure** — a multinational's domain returned
  `workforce.total: 22`, a mis-attached subsidiary record. Sanity-check outliers against the
  description.
- `score` is `0.000` on every row — **do not sort by it.**
- Observed fill rates: headcount 10/10, description 10/10, HQ 8/10.
- → Company, Domain, HQ, headcount (+ founded year, funding) · `company-found`

## 1a — dedupe · free

- Same non-empty domain → keep the fullest, mark copies `duplicate` (park, don't delete).
- Cross-domain near-dupes (same name/HQ, different TLD) → **ask which to keep**; never
  auto-merge, since TLDs can be distinct entities.
- Empty domain → resolve by web search before Step 2.
- **After route B:** drop rows whose host is a platform rather than the company —
  `linkedin.com`, `github.com`, app stores, review sites. Two of twenty in testing.

## 1b — score fit · free · *only if fit criteria given*

- The scoring text is already in the Step 1 response — **no extra call.** After route A use
  `description` + `specialties`; after route B use `entities[].properties.description` plus
  the result `text`.
- To target the criteria directly, re-run route B with `contents.highlights.query` set to the
  fit criteria — it takes its **own** query and returns a per-company relevance snippet, still
  $0.01.
- Set **Fit** (`High`/`Med`/`Low`) and a one-line **Fit reason** citing the criteria.
- **Annotate, don't gate** — never drop low-fit rows; it is a heuristic over marketing copy.
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
  `companies` map keyed by `company_id`. Add `include_employment_history` or `verbose` only if
  full career history is needed.
- → Person, Title, Seniority, LinkedIn URL, country · `candidate` | `no-contact-found`

## 2-fallback — Exa people search · `POST https://stableenrich.dev/api/exa/search` · $0.01/company

**Only when FullEnrich is unavailable, or the run has ≲15 companies.** Exa cannot batch — it
is one call per company, so 25 companies cost $0.25 against one batched $0.15.

```json
{"query":"<seniority/function> at <Company>","category":"people","numResults":5,
 "contents":{"text":{"maxCharacters":300}}}
```

- **Name one company per query.** A query naming three returned 10/15 on-target and drifted
  to lookalike companies.
- Read `entities[]` of `type:"person"` → `firstName`, `lastName` (**already split — feed
  straight to the handoff**), `location`, `workHistory[]`. The **current** role is the entry
  whose `dates.to` is `null`.
- **Match `company.name` exactly, not by substring** — a query for one company returned a
  person at a similarly-named other company, and substring checks passed two more.
- **Surnames may be truncated** (`Alex M.` — profile privacy). Recover the full name from the
  handle in the result `url` (`/in/alexmoreau` → `Alex Moreau`) before it reaches the table.
- **No seniority enum** — infer Step 3's ranking from the title. Seniority wording steers
  results but does not filter; the tail is still ICs.
- **No `metadata.total`** — preview cost from the company count instead.
- Observed precision on single-company queries: 10/10 and 8/10.
- → same columns as Step 2 · `candidate` | `no-contact-found`

## 3 — select · free

- **Ask run mode first:** *single* (one picked company, cheapest) or *fan-out* (one contact
  per company; handoff cost scales at ~$0.018 each). All rows stay in the table either way —
  mode only sets how many advance.
- Then pick manually, or auto-select **most senior**: Owner/Founder/C-level > VP > Head >
  Director > Manager. In fan-out, ask the rule **once**, apply it to all, and show the planned
  table for a **single approval**.
- → Selected?=yes · `selected`

## Handoff to `email-verification` · ~$0.018 per contact

Finding and verifying the address is a separate skill. Invoke it with the selected rows; it
enriches this same CSV in place. Per contact:

```json
{"first_name": "Alex", "last_name": "Moreau", "domain": "example.com"}
```
returning `{"email", "email_source", "verified", "score", "status"}`.

- **Pass names exactly as they appear — never normalize diacritics on the way out.**
  Transliteration is language-dependent and belongs to the finder; a fixed `ü→u` rule produced
  a confirmed-undeliverable address in testing.
- **Never hand off a truncated surname.** Recover it from the LinkedIn handle, or select a
  different contact — a bare initial cannot be looked up.
- State the cost before handing off: selected rows × ~$0.018.
