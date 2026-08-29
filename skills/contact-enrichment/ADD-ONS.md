# Opt-in paid add-ons

Extras to the default $0.067 run of [`SKILL.md`](SKILL.md). **None of these run by
default.** Offer each by name and price and wait; PDL and `userInsights` clear the
$0.10 threshold, so they need explicit OK rather than a nod.

| Add-on | Price | Buy it when |
|---|---|---|
| Exa career + education | $0.01 | the brief needs dated career history or education *and* Apollo's `employment_history` came back thin |
| Apollo `organizations/enrich` | $0.01 | Step 2 was skipped or returned no `organization` object |
| PredictLeads hiring | $0.04 | hiring signals are wanted and the careers page cannot be scraped |
| Aviato `social/person/posts` | $0.08 | Fiber returned nothing |
| **PDL phone** | **$0.28** | the rep says they will dial |
| **twit.sh `userInsights`** | **$0.50** | explicitly requested |

Worst case, all of them: ~$0.92. Without PDL and `userInsights`: ~$0.13.

## Career + education — Step 2

`POST https://stableenrich.dev/api/exa/search` · $0.01

Only when the brief needs **dated career history or education** (→ common ground)
and Apollo's `employment_history` came back thin. Offer by name; skip by default.

Query must be **bare**: `"{name} {company}"`, `numResults: 1`, no `category`, no
qualifiers — that's what triggers the entity matcher.
- **Entity hit:** `results[0].entities[0].properties` → name, location,
  `workHistory[]`, `educationHistory[]`.
- **Search hit only:** no `entities`, but `title` is usually
  `"{Name} – {Title} at {Company}"` and `url` is their LinkedIn. Still usable.
- **Miss:** harmless — free search covers it.

## Phone — Step 2

`POST https://stableenrich.dev/api/pdl/people-enrich` · $0.28

Apollo does not return phones. PDL is the stack's only phone source. **Needs an
email** — from intake, or one Apollo returned. Offer it **by name and price**,
only when the rep says they need to dial; it exceeds the $0.10 threshold, so it
needs explicit OK. Never buy it to fill a blank field.

**Two failure modes, check both:**
1. **Explicit no-match:** HTTP 200 with `{"status":404,"data":null}` — check the
   *body*, not the status code.
2. **⚠ Confident null:** `status:200`, high `likelihood`, every useful field
   `null`. **`likelihood` = match confidence, NOT completeness.** Verify
   `job_title` / `job_company_name` / `linkedin_url` are populated; if not, treat
   as failed resolution regardless of score.

## Company firmographics — Step 3

`GET https://mpp.orthogonal.com/apollo/api/v1/organizations/enrich` · $0.01

**Usually unnecessary — Step 2 already returned the `organization` object.** Buy
only if Step 2 was skipped or returned no org, *and* (a) found no funding entry
and no company page. Announce first; sanity-check against Step 2.

## LinkedIn posts fallback — Step 5

`GET https://mpp.orthogonal.com/aviato/social/person/posts` · $0.08

The fallback when Fiber returns nothing — same feed, half the depth, twice the
price, no permalinks.

## Hiring signals — Step 5

`GET https://mpp.orthogonal.com/predictleads/v3/companies/{domain}/job_openings` · $0.04

Yields trigger events. Free alternative: scrape the careers page.
