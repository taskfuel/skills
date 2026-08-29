# Route A — FullEnrich company-search

`POST https://stableenrich.dev/api/fullenrich/company-search` · $0.15/call, free when zero results

Step 1 of [`SKILL.md`](SKILL.md) when the ICP is categorical or numeric. Yields Company, Domain,
HQ, headcount · `company-found`, then continues at Step 1a.

**Ask:** industries, HQ locations, headcount band
(e.g. `Software Development` / `Zurich` / `1-50`).

- Needs ≥1 real filter (`names`, `domains`, `industries`, `headquarters_locations`,
  `keywords`, `specialties`); `offset` alone → 400. Arrays take plain strings or
  `[{"value":"…","exact_match":true}]`.
- `industries` must be an **exact enum value** — free text like "fintech" → 400.
- `names` matches the **legal entity, not the brand** (Anysphere ≠ Cursor).
- **Returns ≤10 companies per call; there is no `limit`.** Page with `offset` (≤10,000) or
  `search_after` — **each page is another $0.15.** Budget by pages, not rows. A zero-result
  query is free, so an over-narrow first attempt costs nothing.
- **`metadata.total` is the whole pool** (148 for Stuttgart × Software Development × 10–200).
  Read it off page 1 to decide how many further pages are worth buying.
- Headcount filters **server-side** (`headcount_min`/`headcount_max`), so every row is in-band.
- Spot-check `locations.headquarters` for loose HQ matches. When `headcount` (number) and
  `headcount_range` (string) disagree, **trust the number**.
- `industry` is an object — read `industry.main_industry`. `specialties` and `description`
  come back on every row, which is what Step 1b scores.
