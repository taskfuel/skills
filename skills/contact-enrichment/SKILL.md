---
name: contact-enrichment
description: Given a sales contact's name + company or email address, build a Markdown contact brief covering role, responsibilities, recent posts and talks, interests, company context, and conversation-starter hooks for personalizing outbound email or prepping a pre-call brief. Free web search covers company context, profile discovery, interests and news; paid calls (Apollo identity, Fiber LinkedIn posts, twit.sh, Serper news) are billed per call through the user's taskfuel.ai balance and used only where free search is proven weaker. Use when the user wants to research a contact before outbound or a sales call, or wants background on a person from their name and company, or their email.
license: MIT
metadata:
  version: 1.0.0
---

# Contact Enrichment

Build a research brief on one sales contact: who they are, what they've said
recently, what's happening at their company, and how to open a conversation.
Free web search does most of the work; a handful of paid calls fill the gaps
where free search is measurably weaker.

## Three rules governing every step

1. **Always prompt, never guess.** Ask for each step's parameters; offer concrete
   examples so answering is one word.
2. **Announce, then wait.** Before any paid step, state what it does and what it
   costs, then wait for explicit OK. Report the live running total, never a fixed
   promise up front.
3. **Company-agnostic.** Assume nothing about the user's own company or product.
   The "why us" angle comes only from their intake answer — never a built-in ICP.

## Paying — taskfuel.ai

Every paid endpoint is billed per call against the user's prepaid balance. No
wallet, no chain, no 402 headers.

```bash
command -v taskfuel || curl -fsSL https://taskfuel.ai/install.sh | sh
taskfuel whoami        # connected account + balance
```
Not connected? Run `taskfuel connect` **in the background** and read its output
immediately — it prints a pairing code and URL, then blocks until the user
approves. Hand them the URL, wait, re-check `whoami`.

**Quote, then pay:**
```bash
taskfuel call <url> --method POST --body '<json>' --dry-run --model claude-opus-5
taskfuel call <url> --method POST --body '<json>' --max-amount 0.04 --model claude-opus-5
```
`--dry-run` sends the real request and reads back the price **without paying** —
authoritative for that exact payload, and free. Then pass the quoted figure as
`--max-amount` so a repriced endpoint can't exceed what was approved. Body →
stdout; cost line → stderr.

**Spending rules.**
- Quote the first call to any endpoint in a run.
- **Any call over $0.10 needs explicit OK** — that's **PDL ($0.28)** and
  **`userInsights` ($0.50)**.
- Never blindly retry a call that printed "paid" but returned junk. Check
  `taskfuel balance`, tell the user, let them decide.
- Balance below the next call's price → stop, send them to
  https://app.taskfuel.ai.

**Reporting cost.** Give the quoted total, and read `taskfuel balance` before and
after the run. The CLI has no transaction log — for authoritative billing, point
the user at their taskfuel.ai dashboard.

**Discovery is free.** `taskfuel discover <keywords>` searches every provider's
spec; `<domain>` lists one service; `<METHOD> <url>` returns full docs plus
current price. Use it before assuming an endpoint moved or a price changed.

**Rate what you call.** `taskfuel rate <url> [--method POST] --vote up|down`; add
`--report '…'` for concrete breakage. Judge the endpoint against its own docs,
from a call you made — never down-vote a catalog gap or your own bad arguments.

**Host note.** Use catalog URLs exactly as `taskfuel discover` prints them —
Orthogonal is `mpp.orthogonal.com`. *(v1 banned that host; the hazard was paying
it directly over x402, which taskfuel doesn't do.)*

## Intake

**Two entry points, both $0.01 — pick by what the rep has, not by price.**

- **Email** — slightly preferable when available: unique, so it skips
  confirm-the-match, and it carries the company domain free.
- **Name + company** — the common case (referral, conference, LinkedIn browse).
  **Requires a confirm-the-match step** before Step 3.

Free web search runs on top of both, always — not as a fallback.

**Reject name alone.** Nothing here disambiguates a bare name. Anything else the
rep knows is a tiebreaker, never assumed.

**Also ask (optional):** *"What are you selling, and what's your angle with this
person?"* Drives "why us" and sharpens openers. Blank ⇒ omit that section.

## Deliverable

`briefs/<first-last-company>-YYYY-MM-DD.md`. **If one exists, ask:** new dated
file or overwrite. Create it right after intake — **before any paid call** — with
`Status: in-progress`, so a run survives interruption. If identity resolution
fails, slug whatever the input was so failed runs stay traceable.

**Section order** (most are Step 7 synthesis over data already bought, $0 extra):

header (name/title/company/confidence + buying-role + **outreach verdict**) ·
**handle with care** *(placed here on SOFTEN/HOLD; otherwise in its usual slot
below — never omitted silently)* · trigger events / "why now" ·
conversation-starter hooks · suggested openers · relevance angle / "why us"
*(only if an angle was given)* · possible common ground · role & responsibilities
· company context · online presence · personal technical footprint · recent
posts/talks/activity · interests · shared connections & internal relationship
history · free-search completeness check · call logistics · sources & cost.

## Pipeline

### Step 1 — Doc shell (free)
Write the file: header, input, `Status: in-progress`.

### Step 2 — Resolve identity

**One paid call, either entry point, then free web search on top — always.**
Announce the cost first. There is no paid fallback chain: if the call comes up
empty, free search is the whole recovery; if that also fails, the contact is
unresolvable (terminal state below).

**⚠ `--dry-run` before calling.** Never trust a price written in these docs.

```bash
# email path
taskfuel call https://mpp.orthogonal.com/apollo/api/v1/people/match \
  --method POST --body '{"email":"<email>"}' --max-amount 0.01 --model claude-opus-5
# name+company path
taskfuel call https://mpp.orthogonal.com/apollo/api/v1/people/match --method POST \
  --body '{"first_name":"<F>","last_name":"<L>","organization_name":"<Co>"}' \
  --max-amount 0.01 --model claude-opus-5
```

Returns in one shot: `name` · `title` · `linkedin_url` · `city`/`state`/`country`
· `email` · **`seniority` + `departments`** (buying-role as typed fields, no
inference) · **`employment_history`** · and a full **`organization` object**
(domain, industry, headcount, founded year, company LinkedIn, keywords) which
**covers most of Step 3a at $0 extra**.

**⚠ Check `employment_history` before anything else.** A `current: false` on the
company the rep named means **the contact has moved** — that outranks every other
field, and the rep's premise (and angle) may be aimed at the wrong company. Depth
varies by contact: sometimes one untyped entry, sometimes five with start/end
dates. **Read it before buying Exa or PDL for career dates.**

**⚠ Firmographics are estimates and have been wrong** — a `founded_year` five
years off, a headcount of 1 for a two-person team. Cite source + date; prefer a
primary page.

**Confirm the match with the user before Step 3** on the name+company path.

#### Phone escalation — PDL `POST /api/pdl/people-enrich` ($0.28), opt-in only
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

#### Optional add-on — Exa `POST /api/exa/search` ($0.01), career + education
Only when the brief needs **dated career history or education** (→ common ground)
and Apollo's `employment_history` came back thin. Offer by name; skip by default.

Query must be **bare**: `"{name} {company}"`, `numResults: 1`, no `category`, no
qualifiers — that's what triggers the entity matcher.
- **Entity hit:** `results[0].entities[0].properties` → name, location,
  `workHistory[]`, `educationHistory[]`.
- **Search hit only:** no `entities`, but `title` is usually
  `"{Name} – {Title} at {Company}"` and `url` is their LinkedIn. Still usable.
- **Miss:** harmless — free search covers it.

#### Websearch on top — free, mandatory, both paths
Not a fallback. It runs **even when the paid call succeeds**, because that's what
catches paid-source errors — a mis-geocoded location, a dead X handle, an
unverified LinkedIn URL, a `founded_year` five years off.

Search `"{name}" {company}` / `"{name}" {domain}`; on the email path also the
address, its local part and its domain (a company's own about/team page often
names the person outright).

**Accept a websearch-derived identity only if corroborated ≥2× independently,
at least one source primary** — their LinkedIn, the company's own site, their
GitHub, a conference speaker page — ***not* RocketReach-style aggregators, which
scrape the same upstream and corroborate nothing** — and with **no same-name
ambiguity**.

**⚠ Websearch confabulates rather than returning null.** Paid sources return null
when they don't know; a search summarizer invents a plausible answer. Observed: a
summary dating an article "March 2026" that its own page dates 2025-11-26; and a
past job implied as current.

#### Conflict precedence
- **Typed fields come from the paid source:** `seniority`/`departments`, the
  `organization` object, `employment_history`, `workHistory`/`educationHistory`
  (Exa), phone (PDL). Websearch gives prose, not typed data.
- **Except firmographics** — those lose to a company's own page or a funding
  database.
- **On conflict, websearch wins only when corroborated against a primary page** —
  and you must **`WebFetch` that page and cite it.** Never resolve a conflict
  from a search summary; that's what fabricates dates.
- **Location: no source is authoritative.** Cross-check company HQ (and the phone
  country code if the escalation was bought).
- Record an unresolved conflict rather than silently picking a winner.

#### Terminal state — paid call empty *and* websearch empty
The contact is **unresolvable**. Stop and ask: different input, manual research,
or abandon. There is no further endpoint to try, by design. **Never build a brief
on a guessed identity.**

**Writes:** name, title, company + domain, career history, match confidence,
which source resolved it. From the same response at $0 extra: role start date →
trigger events · location → timezone · `seniority`/`departments` → buying-role ·
`organization` → Step 3a · employer + education → common ground.

### Step 3 — Company context (free; the paid call is a rare fallback)

**a) Profile — free search.** `"{company}" employees revenue founded`,
`"{company}" funding investors`, `"{company}" founders`. Funding databases
(Crunchbase / PitchBook / Tracxn) and the company's own site cover most fields.
**Aggregator headcount/revenue are dated estimates** — cite source + date. Free
search returns exact figures where paid returns coarse buckets, and **buckets
hide source conflicts.**

**b) Tech stack — free, from live infrastructure** (beats every paid source):
```bash
curl -sI https://<domain>   # server:/cf-ray: → CDN · x-powered-by: → runtime
dig +short MX  <domain>     # aspmx.l.google.com → Google Workspace
dig +short TXT <domain>     # SPF includes: zendesk, sendgrid, protonmail, …
```
Cross-check the runtime against the contact's own skills — a match is a hook.

**c) Company GitHub — free search.** `"{company}" GitHub` (distinct from the
*contact's*, Step 4). Prefer over PredictLeads' paid `github_repositories`
($0.04), which returned a false negative. Pull primary languages,
product-relevant repos, protocol/spec forks. Note "no public org found" rather
than skipping silently.

**d) Paid fallback — Apollo `GET /apollo/api/v1/organizations/enrich`, $0.01.**
**Usually unnecessary — Step 2 already returned the `organization` object.** Buy
only if Step 2 was skipped or returned no org, *and* (a) found no funding entry
and no company page. Announce first; sanity-check against Step 2.

**Writes:** industry, headcount, revenue, funding, description, tech stack, repo
findings — each with source + date. Funding/launch → trigger events; headcount +
seniority → buying-role.

### Step 4 — Social/profile surface (free)

Paid discovery is removed — it returned nothing free search missed and produced
two false negatives.

Search free, skipping anything already known: **LinkedIn URL · GitHub profile ·
personal site/blog · talk/conference/podcast mentions.**

**Verify every identifier — supplied or found — before use.** Any can be silently
wrong: a dead X handle, an unverified LinkedIn URL, a denied GitHub account that
exists. Run a free search (`"{name}" {company} {network}`); accept only when the
result corroborates the known name **and** company. **Mark "confirmed" only once
that search has happened** — never on a paid source's say-so. **Verify before any
paid call that consumes the identifier** (twit.sh, Fiber).

**Personal technical footprint — free, and the strongest hook for a technical
contact.** Distinct from the company org. Search `"{name}" {company} GitHub` and
`"{name}" blog OR Medium OR author`. A protocol-level fork or a how-to they wrote
proves individual depth a job title cannot. **Write "no personal footprint found"
rather than skipping** — that finding correctly steers openers away from a
technical hook for a non-technical contact.

**Email — out of scope to *discover*, in scope to *record*.**
- **Never spend to find one.** No pattern construction, no `email-finder`, no
  PDL. Finding *and verifying* an address needs role-mailbox rejection (`mgt@`,
  `info@`) plus a verify→retry loop — a different job from this skill, and one a
  dedicated email-finding skill should own.
- **If a call already made hands one over, keep it** — Apollo's Step 2 response
  carries `email` free. Record it, labelled **unverified** (Apollo doesn't prove
  deliverability).
- Otherwise write `Email: not resolved`.
- **⚠ A `⚡name@domain` is a lightning address, not an inbox.**

**Writes:** verified LinkedIn · personal site/blog · verified GitHub · verified X
handle · talks; explicit "not found" notes for empty results.

### Step 5 — Recent activity: posts, talks, interests, news
Announce each sub-source's cost separately; any can be skipped.

**Interests / writings / talks — free search. Never buy Happenstance** (free
search returns its entire advertised output, sourced). Search author pages
(Medium, company blog, trade press) and interviews — **interviews are the richest
hobby source.** Sourced is the normal path; tag "(inferred from public activity)"
only when nothing personal surfaces.

**News — `POST https://mpp.orthogonal.com/serper/news` ($0.002), a default step.**
Returns items with `date` + `source`. At ~3% of a run it is rounding error, and
what it catches — a breach, layoffs, a lawsuit — is the most expensive thing to
miss in outbound. Free search runs alongside.
- **⚠ Use the Orthogonal host**, not `stableenrich.dev/api/serper/news` ($0.04) —
  same Google News, 20× the price. (`search1api.com/news` is $0.003 but returns
  **no dates**, which is the whole point — don't swap.)
- **⚠ Hard rule: dates come from the source page, never a search summary.** The
  summarizer *will* invent them. A wrong date in a cold email is worse than a
  missing one — `WebFetch` the article or read a dated URL slug before promoting
  anything to a trigger event.
- Read news both ways: positive/neutral → trigger events + hooks; negative →
  handle with care.

**⚠ The 90-day recency gate — run before Step 7, every time.** Ask explicitly:
*"has anything happened to this company in the last 90 days?"* Search
`"{company}" news` and `"{company}" incident OR outage OR breach OR layoffs OR
lawsuit`, plus the company's blog/status page. **This is a distinct check, not a
by-product of the news call** — a topical query can rank an evergreen feature
above a three-week-old incident.
- Anything material inside 90 days **sets the outreach verdict** (Step 7) and
  goes in handle with care, directly under the header.
- Nothing found? Write **"no material events in the last 90 days (checked
  YYYY-MM-DD)"** — an explicit negative is what lets a verdict be SEND.

**LinkedIn posts — Fiber ($0.04), the strongest personalization signal:**
```bash
taskfuel call https://mpp.orthogonal.com/fiber/v1/linkedin-live-fetch/profile-posts \
  --method POST --body '{"identifier":"<verified slug>"}' --max-amount 0.04
```
Per item: `caption` (**their own words**), `resharedPost` (the amplified post, a
separate field), `postUrl` (permalink), `author`, `postedAt`, `engagement`.
**Because reshares are a separate field, attribution is unambiguous** — quote
`caption` as theirs; cite `resharedPost.author` as *"{Contact} amplified
{Actor}'s post: …"*. Up to 50/page; **continuations bill $0** via
`parentRequestId` with unchanged filters. Authored ratio varies a lot by contact
(20/38 and 42/50 observed).

**⚠ The date-bucket trap — neither Fiber nor Aviato returns a post date.** Both
return a `noEarlierThan`/`noLaterThan` **window**: ~30 days for the last year,
**365 days** beyond.
- **Never write a bucket midpoint as a date.** "He posted about hiring in March
  2025" is fabrication when the source supports only "sometime in 2024–2025".
- **To promote a post to a trigger event, `WebFetch` its `postUrl`** and read the
  real date.
- **No real date? Keep the item, drop the date** — "undated, LinkedIn window
  2024-08→2025-08". A hook survives undated; a trigger event does not.
- The deep tail is old — good for interests and common ground, **useless for "why
  now."**

*Aviato `social/person/posts` ($0.08) is the fallback if Fiber returns nothing —
same feed, half the depth, twice the price, no permalinks.*

**Twitter/X** — use the **verified** handle from Step 4.
`GET /users/by/username` ($0.005) + `GET /tweets/user` ($0.01, ≤20 tweets). A
404/empty means the handle is wrong — go back to web search, don't keep paying
against a bad handle. `GET /workflows/userInsights` ($0.50) is **explicit opt-in
only**. **Never call twit.sh write/action endpoints** (follow/like/retweet/tweet/
delete/setProfile/bookmark) — read-only research.

**Personal site/blog** — `GET https://x402.ottoai.services/web-extract?url=…`
($0.005), clean LLM-ready markdown. Fall back to `POST /api/firecrawl/scrape`
($0.01) when the page needs real JS rendering. **Neither may be pointed at
LinkedIn** — Firecrawl refuses it outright.

**Hiring signals** — `GET mpp.orthogonal.com/predictleads/v3/companies/{domain}/job_openings`
($0.04) → trigger events. Free alternative: scrape the careers page.

**Writes:** recent posts/talks/activity (source + date tagged), hiring signals →
trigger events, interests (sourced; tagged inferred only when synthesized).

### Step 6 — Shared connections & internal relationship history (free)
Neither is queryable in this stack — two honest manual-check notes, deliberate
choices, not placeholders:
- **Shared connections** — tell the rep to check LinkedIn "mutual connections"
  from their own account.
- **Internal relationship history** — no CRM in the stack; tell them to check
  theirs / ask the team before cold-emailing someone a colleague knows.

### Step 6.5 — Free-search completeness check (free, mandatory)

Paid structured data crowds out patient free search — measured, on the same
contact, as a *thinner* landmine section in a cheaper run. This step is the guard.

**Before Step 7, each slot must hold a result *or* an explicit "none found
(checked YYYY-MM-DD)":**

| Slot | Minimum bar |
|---|---|
| **Landmines / friction** | ≥1, or explicit none. Search `"{name}" criticism OR rejected OR banned OR lawsuit OR controversy` and their own posts for public fights. |
| **Interests (non-work)** | ≥2 sourced, or explicit none. |
| **Personal technical footprint** | verified profile, or explicit none. |
| **90-day company events** | verdict-bearing item, or explicit none *(Step 5 gate)*. |
| **Common ground** | ≥1 candidate, or explicit none. |

**A blank fails the brief.** "None found" is a useful answer; a silent omission
isn't — the next run can't tell "checked, nothing there" from "never looked."

### Step 7 — Synthesize + finalize (free)
Pure reasoning over data already bought. Each item carries a citation or an
explicit "(inferred)" tag.

#### The outreach verdict — decide FIRST; the openers inherit it
Handle-with-care must be able to **overrule** the openers, not sit beside them.
Set one in the header:

- **SEND** — nothing material in 90 days; openers stand.
- **SOFTEN** — outreach is fine, but every opener must acknowledge the event or
  deliberately route around it. Say which.
- **HOLD** — recommend not sending yet, **and say what to wait for**. Still write
  the openers, marked *for later*.

**The verdict is a recommendation to a human, never a refusal to do the work.**
Produce the full brief regardless; the rep may overrule you.

**A verdict can also be about the premise** — if `employment_history` shows the
contact left the company the rep named, say so up front. The brief is still
worth writing; the angle needs re-aiming.

#### Sections to synthesize
- **Trigger events / "why now"** — new role / funding / launch / hiring / recent
  posts, most-recent first. **Every date read off a source page.**
- **Conversation-starter hooks** — 3–5 bullets, source + date tagged.
- **Suggested openers** — 2–3 draft lines grounded in tagged hooks; tie one to
  the "why us" angle if given. Drafts, not send-as-is. **Each must be consistent
  with the verdict** — on SOFTEN/HOLD, an opener that ignores the landmine is a
  defect.
- **Relevance angle / "why us"** — **omit entirely if no angle was given.** Never
  invent a pitch.
- **Possible common ground.**
- **Handle with care** — the 90-day gate *and* durable landmines (public
  friction, stated allergies, topics to avoid). Under the header on SOFTEN/HOLD;
  write "none surfaced (checked YYYY-MM-DD)" rather than omitting it.
- **Buying-role framing** — decision-maker / champion / influencer, taken
  straight from `seniority`/`departments` rather than inferred → header.
- **Call logistics** — timezone always; phone only if the PDL escalation was
  bought, else `Phone: not resolved`.
- **Sources & cost** — every call, its cost, run total, balance delta. Set
  `Status: complete`.

**End the run:** print the **full brief inline**, then a last line with the saved
path (`Saved to: briefs/…`). The rep acts on it now; the file is the durable copy.

## Cost

**Indicative — `--dry-run` before every call.**

### Default run — $0.067, either entry point

| Step | Call | Cost |
|---|---|---|
| 2 | Apollo `people/match` + free websearch | **$0.01** |
| 3–4 | free (Apollo's `organization` object, DNS/headers, search) | $0 |
| 5 | Fiber LinkedIn posts | **$0.04** |
| 5 | twit.sh profile + timeline | **$0.015** |
| 5 | Serper news + 90-day gate | **$0.002** |
| 6–7 | free | $0 |

**These are quoted prices** — `--dry-run` before every call, and report the
figure the account actually shows.

**Step 2 is flat with no tail** — one paid call, no escalation chain. A miss
costs $0 extra; an unresolvable contact stops the run rather than spending more.

**Opt-in add-ons**, each announced by name and price: Exa career/education
+$0.01 · ottoai extract +$0.005 (Firecrawl +$0.01 for JS) · PredictLeads hiring
+$0.04 · Apollo `organizations/enrich` +$0.01 · **PDL phone +$0.28** ·
`userInsights` +$0.50. The last two need explicit OK.

- **Steps 3–4 and Step 5's interests are free** — this halved run cost *and*
  improved quality. **Never buy Happenstance.** But free ≠ optional: Step 6.5
  fails the brief on a silent blank.
- **Fiber is the largest line item** at $0.04. Skip it (→ ~$0.027) only when the
  contact has no LinkedIn presence.
- **The phone is the biggest cost lever** — $0.28 for one field, 4× the whole
  default run. Only buy it when the rep says they will dial.
- **Worst case** (every add-on): ~$0.92. Without PDL and `userInsights`: ~$0.13.

## Notes on the paid endpoints

Each choice below was benchmarked head-to-head before adoption:

- **Apollo `people/match` ($0.01)** replaced both a $0.28 email-identity endpoint
  and a $0.01 name-identity one. It takes either input, and on a test contact it
  resolved someone the alternative missed at the same price.
- **Fiber `profile-posts` ($0.04)** replaced a $0.08 LinkedIn source. Same
  underlying feed, ~4x the depth, real permalinks, and reshares in a separate
  field so attribution can't be fumbled.
- **Serper news on the Orthogonal host ($0.002)** is the same Google News as a
  $0.04 endpoint elsewhere in the catalog.
- **Free beats paid** for company profile, tech stack (live DNS/headers), GitHub,
  interests and profile discovery — measured, not assumed. Step 6.5 exists
  because paid structured data otherwise crowds free search out.

Re-quote everything with `--dry-run` before relying on a price here.
