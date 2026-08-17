---
name: brand-check
description: Vet brand/product name candidates with real paid APIs through taskfuel.ai — WHOIS domain-availability sweep, dev-platform username checks, X-handle lookup, web collision search, and US trademark search, paid per-call from the prepaid balance. Use when the user wants to check, compare, or brainstorm names/domains ("is X available?", "help me name this", "/brand-check").
license: MIT
---

# Brand Check

Vet candidate brand/product names end-to-end using pay-per-call APIs routed
through taskfuel.ai. Sweep domain availability, check handle coverage, search the
web for name collisions, and hit the US trademark register — then write the
customer a clearance report. Every paid step goes through the `taskfuel` CLI
(quote first, then pay).

## Prerequisites

- `taskfuel` CLI connected (`taskfuel whoami`), balance ≥ ~$0.20 for a full
  ~10-candidate run.
- Base [`taskfuel`](../../taskfuel/SKILL.md) skill covers all payment mechanics
  (install, connect, quote, spending rules) — read it first if you haven't.

## Workflow

### 1. Gather candidates
The user's ideas plus your own brainstorm (8–12 base names). taskfuel has no paid
name-gen endpoint — brainstorm in-context.

### 2. Sweep domain availability
Announce the budget, then one WHOIS call per `name × TLD` across the TLDs the user
cares about (default `.com .io .dev .ai`):

```sh
taskfuel call "https://2s.io/api/domain/whois?domain=acme.dev"   # $0.001
```

**A 404 here means the domain is AVAILABLE, not that the call failed.** The CLI
prints `error: gateway returned HTTP 404` in red, which reads like a broken
endpoint — it isn't. No WHOIS/RDAP record exists because nobody has registered
the name. An unregistered sweep therefore looks like a wall of red errors, and
that is the *good* outcome. Read it as a result, not an outage.

`data.items` non-empty ⇒ **taken** (read `.registrar.name` and `.expiresAt` — flag
domains expiring < ~120 days out as backorder candidates); `items: []` or
HTTP 404 ⇒ **available**.

Before concluding an endpoint is down, **run one control with a value that must
exist** — `google.com`, `web.dev`, `crates.io`, `anthropic.ai`. If the control
returns data, the endpoint is healthy and your 404 is a real negative. RDAP is
verified working for `.com .io .dev .ai .net .app` (2026-07-30). Don't use
`ottoai/whois-lookup` — it returns `result: null` false-negatives for registered
domains; `2s.io/api/domain/whois` is authoritative.
Summarize the sweep as an availability matrix. Keep it ≤ ~15 name×TLD combos
(one call each — no bulk endpoint) or ask before a larger run.

### 3. Shortlist 2–3 front-runners
With the user's priorities in mind, then in parallel per front-runner check handle
coverage — a 404 means the handle is free:

```sh
taskfuel call "https://2s.io/api/github/user?username=acme"        # GitHub
taskfuel call "https://2s.io/api/news/hn-user?username=acme"       # Hacker News
taskfuel call "https://x402.twit.sh/users/by/username?username=acme" # X/Twitter, $0.005
```

### 4. Deep-dive the top pick
In parallel: full domain profile, a company check, and a web collision search:

```sh
taskfuel call "https://2s.io/api/domain/intel?domain=acme.ai"                  # DNS+WHOIS+rep
taskfuel call https://brave.mpp.paywithlocus.com/brave/web-search \
  --method POST --body '{"q":"acme","count":20}'                               # collisions, $0.035
taskfuel call "https://stableenrich.dev/api/companyenrich/org-enrich" --method POST --body '{"name":"Acme"}'
```

Two things to read in the Brave response beyond the hits themselves:
`data.query.altered` — if the engine "corrected" your name to a real word, the
name has essentially no web presence, which is a strong availability signal *and*
a warning that people will mistype it — and a case-insensitive grep for the term
across the raw response. If the only occurrence is the echoed query, the name is
genuinely unused.

The web search surfaces existing products/companies already trading under the
name — the highest-value finding the user can't see elsewhere.

### 5. Trademark
One full-text search on the core wordmark (add `intlClass` for the user's category
if known); if it surfaces a live in-category mark, pull detail on that serial:

```sh
taskfuel call "https://2s.io/api/law/trademark-search?query=acme"           # US register
taskfuel call "https://2s.io/api/law/trademark-status?serial=NNNNNNNN"      # prosecution detail
```

### 6. Report — save it, don't just say it
Write the clearance report to `~/Documents/brand-checks/<YYYY-MM-DD>-<winner>.md`
and `xdg-open` it, then summarize the verdict in chat. Structure: YAML frontmatter
(date, brief, candidates, winner, verdict, checks run, cost, degraded/skipped
checks) followed by the customer-facing doc — one ✅/⚠️/❌ verdict section per name
with the evidence in plain language, a ranked recommendation, then a **"Do this
now"** list with urgency (register X today, claim handles, backorder expiring
domains, lawyer only if a trademark hit lands in-category). No API mechanics in
the body — costs live in the frontmatter and a one-line footer. Remind the
customer availability is a live snapshot, not a reservation, and that the
trademark search covers the **US** register only.

Run independent paid calls in parallel (single Bash block, background `&` +
`wait`) — separate invoices, safe to parallelize. Save raw responses to the job
tmp dir for the report.

## Costs (observed 2026-07, quote to confirm)

| Step | Price |
|---|---|
| Domain availability (`2s.io/domain/whois`) | $0.001/domain |
| Domain profile (`2s.io/domain/intel`) | ~$0.005 |
| GitHub / HN username (`2s.io`) | ~$0.001 each |
| X/Twitter handle (`x402.twit.sh`) | $0.005 |
| Web collision search (`ottoai/web-search`) | $0.009 |
| US trademark search (`2s.io/law/trademark-search`) | ~$0.0024 |
| Company enrich (`stableenrich/org-enrich`) | quote |

A full ~10-candidate run lands around $0.10–0.20.
