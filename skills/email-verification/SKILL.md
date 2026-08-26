---
name: email-verification
description: Find and verify business email addresses through taskfuel.ai — give it a person and a company domain and it looks up their real address; give it addresses you already have and it checks deliverability, paid per call from the prepaid balance. Use when the user wants to find someone's work email, verify an address is real, or clean a list before sending.
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

## Rules

1. **Preview cost, then wait for OK.** Cost scales per address; always state rows × price
   before calling.
2. **Names go in unnormalized.** Never strip diacritics — see Step A.
3. **Provenance travels with the address.** Never emit one without its source and status.

## Entry routes

| You have | Route | Starts at | Cost/row |
|---|---|---|---|
| Addresses to check (one, or a list/CSV) | verify-only | **Step B** | $0.008 |
| A person + a company | find-then-verify | Step A | $0.018 |
| A list of people | batch find-then-verify | Step A, per row | $0.018 |
| A handoff from `lead-prospecting` | batch find-then-verify | Step A, per row | $0.018 |

- **Never run Step A on an address the user already has** — verify-only is less than half
  the price.
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

## Step A — find · `POST https://hunter.mpp.paywithlocus.com/hunter/email-finder` · $0.01/person

*Skipped on the verify-only route.* Body `{"first_name","last_name","domain"}`.

- **Pass the name exactly as written; never normalize diacritics.** Hunter transliterates per
  language, which no fixed rule can do: German `Müller` → `mueller`, Finnish `Pärnänen` →
  `parnanen`. In testing a `ü→u` rule produced a **confirmed-undeliverable** address where
  Hunter's `ü→ue` scored 100.
- Returns `email`, `score`, `source_type`, `sources[]`, plus free `position`, `company`,
  `linkedin_url`.
- **`source_type`** — `found` means real evidence; check `sources[]`, since a
  `google.com/search` URI is a query, not a sighting. `generated` means Hunter pattern-guessed;
  still often correct (one verified at 89), so never discard it, just rely on Step B.
- **Ignore its inline `verification` and `accept_all`** — both were wrong in testing. Step B
  is the authority.
- **Truncated surname** (`Alex M.` — profile privacy): recover it from the LinkedIn handle,
  usually firstname+lastname (`/in/alexmoreau` → `Alex` / `Moreau`). If the handle is opaque
  (`alex-m-1e4f`), say so and ask for a different contact. **Never invent a surname or send a
  bare initial.** The documented `linkedin_handle` parameter returns `Upstream request
  rejected` — do not use it.
- **Same-domain shortcut:** once ≥2 people at one domain verify on the same pattern, apply it
  free to the rest of that domain and go straight to Step B.
- **Fallback ladder (free)** — only when the finder returns nothing, or Step B rejects its
  answer. **Guessing is never a substitute for the finder**, and never runs past the 3-address
  cap below.
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
     "email": "…", "status": "valid|invalid", "result": "deliverable|undeliverable",
     "score": 0, "accept_all": false, "smtp_check": true, "mx_records": true,
     "disposable": false, "webmail": false, "gibberish": false, "sources": [] }}}
  ```
  Unwrapping one level yields `null` everywhere and reads as an empty result. **Save each
  response to its own file** — overwriting one scratch file in a loop means a mis-parse costs
  you the calls twice.
- **Pass condition:** `status: valid` + `result: deliverable` + `smtp_check: true`.
- **`accept_all: true` ⇒ the verdict is not proof.** Such a server answers "deliverable" to
  every address. Confirm by verifying a **deliberately fake address at the same domain**: if
  the junk address also returns `valid`, mark the row `email-unconfirmed`, never `verified`.
- **Preview cost every run** — rows × $0.008 (verify-only) or × $0.018 (find-then-verify).
  Above ~$3, pause and offer to narrow or do a top-N first.
- **Double-charge guard:** on any retry, echo the `x-locus-request-id` UUID from the initial
  402 — it is an idempotency key, and the endpoint answers **202** if that request is already
  in flight. Never fire a fresh un-keyed retry for an address already submitted.
- **Failure handling — at most 3 addresses per person, total** (whatever the finder returned,
  plus fallback patterns). On the 3rd failure **stop**; no 4th pattern, no silent
  substitution. Ask verbatim:
  > All addresses tried for **{Name}** at **{Company}** came back undeliverable —
  > {tried}. Would you like to suggest a specific email pattern to verify, or try a
  > different contact?
- **Verify-only has no ladder.** The address is the user's; substituting another would check
  someone else's mailbox. Report `verify-failed` and move on.
- → Verified?, Score · `verified` | `email-unconfirmed` | `verify-failed`
