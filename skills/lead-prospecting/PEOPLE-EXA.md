# Step 2 fallback — Exa people search

`POST https://stableenrich.dev/api/exa/search` · $0.01/company

Substitutes for Step 2 of [`SKILL.md`](SKILL.md) when FullEnrich is unavailable, or the run has
≲15 companies. Exa cannot batch — one call per company, so 25 companies cost $0.25 against one
batched $0.15. Yields the same columns as Step 2 · `candidate` | `no-contact-found`.

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
