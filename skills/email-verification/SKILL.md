---
name: email-verification
description: Find and verify business email addresses through taskfuel.ai — look up a work address from a name and company domain, or check deliverability on addresses already in hand, paid per call from the prepaid balance. Use when the user wants to find someone's work email, or verify or clean addresses before sending.
license: MIT
---

# Email Finding & Verification

Person + domain → a deliverable address, with provenance. Or addresses in → deliverability
out. **Never drafts or sends email** — this skill establishes only that an address exists
and how much to trust it.

Pairs with the `lead-prospecting` skill, which finds *who* to contact and hands off here.

## Prerequisites

- `taskfuel` CLI connected (`taskfuel whoami`), balance ≥ ~$0.10 for a small run.
- Base `taskfuel` skill covers payment mechanics — quote with `--dry-run`, pay with
  `--max-amount`, never suppress stderr on a paid call. Read it first if you haven't.
- **Read the price from `amount_usd`, not the summary line.** `--dry-run` rounds its human
  output to cents, printing `quoted $0.01` for a $0.013 call — a 30% under-estimate on a batch.

## Rules

1. **Preview cost, then wait for OK.** Cost scales per address; always state rows × price
   before calling.
2. **Names go in unnormalized.** Never strip diacritics — see Step A.
3. **Provenance travels with the address.** Every address carries its source and status.

## Entry routes

| You have | Route | Starts at | Cost/row |
|---|---|---|---|
| Addresses to check (one, or a list/CSV) | verify-only | **Step B** | $0.008 |
| A person + a company | find-then-verify | Step A | $0.021 |
| A list of people | batch find-then-verify | Step A, per row | $0.021 |
| A handoff from `lead-prospecting` | batch find-then-verify | Step A, per row | $0.021 |

- **Addresses already in hand go straight to Step B** — verify-only is less than half the price.
- **Company name but no domain?** Resolve and confirm it first; a wrong domain wastes every
  call after it.

## Deliverable

`emails-YYYY-MM-DD.csv`, or the handoff's existing `leads-*.csv` enriched in place. One row
per address, nothing discarded.

`Person | Domain | Email | Email source | Verified? | Score | Status`

- **`Email source`** — `found` (Hunter has evidence) | `generated` (Hunter pattern-guessed) |
  `guessed` (Step A fallback) | `given` (user-supplied). Carry it through: it is how a reader
  knows whether an address is evidence-backed or inferred.
- **`Status`** — `email-found` | `email-constructed` → `verified` | `email-unconfirmed` |
  `verify-failed`

## Step A — find · `POST https://hunter.mpp.paywithlocus.com/hunter/email-finder` · $0.013/person

*Skipped on the verify-only route.* Body `{"first_name","last_name","domain"}`.

- **Pass the name exactly as written.** Hunter transliterates per language, which no fixed rule
  can do: German `Müller` → `mueller`, Finnish `Pärnänen` → `parnanen`. In testing a `ü→u` rule
  produced a **confirmed-undeliverable** address where Hunter's `ü→ue` scored 100.
- **The response is double-nested, same as Step B — read `data.data`.** A genuine miss returns
  nulls there too, so tell the two apart by structure: `data.data` present as an object means
  you parsed right and Hunter has nothing; absent means you unwrapped too few levels.
  `data.meta.params` echoes what you actually sent — check it when a result looks wrong.
- Returns `email`, `score`, `source_type`, `sources[]`, plus free `position`, `company`,
  `linkedin_url`.
- **`source_type`** — `found` means real evidence; check `sources[]`, since a
  `google.com/search` URI is a query, not a sighting. `generated` means Hunter pattern-guessed;
  still often correct (one verified at 89), so keep it and let Step B decide.
- **Treat its inline `verification` and `accept_all` as unconfirmed** — wrong in one test,
  right in another, so they carry no weight either way. Step B is the authority.
- **Try `linkedin_handle` first — a rejection is free.** `{"linkedin_handle":"alexmoreau"}`
  alone returns `first_name`, `last_name`, `email`, `score`, `domain` and `position` at the same
  $0.013, but Hunter resolves only handles it has indexed: 3 of 7 in testing, the rest erroring
  `Upstream request rejected` **at no charge**. So it costs nothing to attempt and falls back to
  `{first_name,last_name,domain}` when it misses. Handle shape does not predict which — two
  handles of the same `name-name-hash` form split one each way.
- **Truncated surname** (`Alex M.` — profile privacy): try the handle, since it returns the full
  name when it resolves. Otherwise read the name off the handle (`/in/alexmoreau` → `Alex` /
  `Moreau`). **Send a full surname or none.**
- **Same-domain shortcut:** once ≥2 people at one domain verify on the same pattern, apply it
  free to the rest of that domain and go straight to Step B.
- **Fallback ladder (free)** — only when the finder returns nothing, or Step B rejects its
  answer. **The finder runs first and guessing only fills its gaps**, within the 3-address cap.
  - Patterns: `first.last@`, `f.last@`, `flast@`, `first@`; for diacritics try **both**
    transliterations (`ü→u` *and* `ü→ue`, `ö→o/oe`, `ä→a/ae`, `š→s`). Avoid `contact@`/`info@`.
  - **Order by evidence already in this run, not by that list.** If addresses already verified
    in the batch share a pattern, try it first — a batch of early-stage startups proved
    uniformly `first@` while the default order burned 3 failed calls on one contact.
- → Email, Email source · `email-found` | `email-constructed`

## Step B — verify · `POST https://hunter.mpp.paywithlocus.com/hunter/email-verifier` · $0.008/address

One address per call — the cost that scales (1,000 ≈ $8). Body `{"email":"…"}`. The
`stableenrich.dev/api/hunter/email-verifier` route is the same upstream API at ~4× the price;
use it only if this one degrades.

- **Response is double-nested — read `data.data`:**
  ```json
  {"success": true, "data": {"data": {
     "email": "…", "status": "valid|invalid|accept_all", "score": 0, "accept_all": false,
     "smtp_check": true, "mx_records": true, "disposable": false, "webmail": false,
     "gibberish": false, "block": false, "regexp": true, "smtp_server": true,
     "result": "deprecated — read status", "sources": [] }}, "meta": {"params": {…}}}
  ```
  **Save each response to its own file** — overwriting one scratch file in a loop means a
  mis-parse costs you the calls twice.
- **Pass condition:** `status: valid` + `smtp_check: true`. The response carries
  `_deprecation_notice: "Using result is deprecated, use status instead"` — read `status`.
- **`accept_all: true` ⇒ the verdict is not proof**, and `status` returns `accept_all` rather
  than `valid` — which the pass condition above already excludes. Such a server answers
  "deliverable" to every address. Settle it with a **control** — verify a deliberately fake
  address at the same domain. A control scoring like the real address (72 vs 78, both
  `smtp_check: true`, in testing) means the domain accepts anything, so the row is
  `email-unconfirmed`. A clean control returns `status: invalid`, `score: 0`, `smtp_check: false`.
- **Above ~$3** — rows × $0.008 (verify-only) or × $0.021 (find-then-verify) — pause and offer
  to narrow the list, or run a top-N first.
- **Double-charge guard:** a call that printed "paid" but returned junk has already been
  charged, and a retry charges again — `taskfuel call` settles the 402 for you, so there is no
  idempotency key to echo. Check `taskfuel balance`, tell the user what happened, and let them
  decide whether to spend again.
- **Failure handling — at most 3 addresses per person, total** (whatever the finder returned,
  plus fallback patterns). On the 3rd failure **stop**; no 4th pattern, no silent
  substitution. Ask verbatim:
  > All addresses tried for **{Name}** at **{Company}** came back undeliverable —
  > {tried}. Would you like to suggest a specific email pattern to verify, or try a
  > different contact?
- **Verify-only has no ladder.** The address is the user's; substituting another would check
  someone else's mailbox. Report `verify-failed` and move on.
- → Verified?, Score · `verified` | `email-unconfirmed` | `verify-failed`
