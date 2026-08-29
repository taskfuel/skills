# Route B — Exa semantic discovery

`POST https://stableenrich.dev/api/exa/search` · $0.01 flat, `numResults` ≤ 100

Step 1 of [`SKILL.md`](SKILL.md) when the ICP is thematic. Yields Company, Domain, HQ, headcount
(+ founded year) · `company-found`, then continues at Step 1a.

```json
{"query":"<the ICP as the target's own self-description>","category":"company","numResults":40,
 "excludeDomains":["linkedin.com","notion.site","notion.so","github.com","linktr.ee"],
 "contents":{"text":{"maxCharacters":700,"verbosity":"compact"}}}
```

- **Phrase the query as the target's self-description** — the words they would put on their own
  homepage ("AI agent platform for customer support"). An ICP written as an activity retrieves the
  agencies that sell that activity as a service, not the companies that do it. A survivor list of
  2–5 person shops is the tell — re-query from the product at $0.01 before paying for Step 2.
- **Fit criteria sharpen the query, not just the score.** They usually name the customer where the
  ICP names the activity, so fold them into the query text before spending, rather than saving
  them for Step 1b.
- **`category:"company"` is mandatory** — without it you get pages *about* companies
  (directories, review sites, app stores), not company sites. It is unvalidated free text, so a
  typo returns those pages instead of a 400 — check `results[].entities[]` is populated.
- **`contents` is free** and is what populates `entities`. Always send it.
- **`entities[]` hangs off each result, not off the response** — read
  `results[].entities[]` of `type:"company"` → `properties`: `name`, `description`,
  `foundedYear`, `workforce.total`, `headquarters.{city,country}`. Domain = the result `url` host.
  `financials.fundingTotal` is documented but came back empty on every row; treat it as absent.
- **Over-fetch, then cut at the tail.** Nothing filters server-side, so ask ~4× the target
  (~4/15 rows land in a given headcount band) and filter client-side; `numResults` caps at 100
  and delivers fewer than asked (40 → 31, or 19 with `excludeDomains`). A narrow theme runs out
  of real matches partway down — genuine companies to ~rank 20, off-topic pages after — so read
  down the ranking, cut where matching stops, and widen the theme rather than paging deeper.
- **Sanity-check every headcount against the description** — a multinational's domain returned
  `workforce.total: 22`, a mis-attached subsidiary record.
- Results carry **no `score` field** — rank them yourself.
- **Expect gaps** — roughly half the rows lack HQ or description. Backfill from the result
  `text`, which repeats industry, headcount and founded year as prose.
