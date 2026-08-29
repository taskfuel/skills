---
name: contact-enrichment
description: Research one sales contact from their name + company or their email address, and write a Markdown brief — role, recent posts and talks, interests, company context, trigger events, and conversation-starter hooks — using free web search plus a few paid calls billed through taskfuel.ai. Use when the user wants background on a person before outbound or a sales call.
license: MIT
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
4. **Write explicit negatives.** A slot you checked and found empty gets
   *"none found (checked YYYY-MM-DD)"*. Silence is the one unusable answer: the
   next run cannot tell an empty slot from an unchecked one.

## Prerequisites

- `taskfuel` CLI connected (`taskfuel whoami`), balance ≥ ~$0.10 for a default run.
- Base `taskfuel` skill covers payment mechanics — quote with `--dry-run`, pay with
  `--max-amount`, never suppress stderr on a paid call. Read it first if you haven't.
- Use catalog URLs exactly as `taskfuel discover` prints them; Orthogonal endpoints
  live under `mpp.orthogonal.com`.

**Spending rules specific to this skill.**
- **`--dry-run` every call and pay with `--max-amount` set to the quote.** Every
  price in this file is indicative; the quote is authoritative.
- **Any call over $0.10 needs explicit OK** — that's **PDL ($0.28)** and
  **`userInsights` ($0.50)**.
- Report the live running total, and read `taskfuel balance` before and after the
  run. For authoritative billing, point the user at their taskfuel.ai dashboard.

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

Paid extras that no run takes by default — career history, phone, hiring signals,
firmographics, the LinkedIn fallback — live in [`ADD-ONS.md`](ADD-ONS.md), with the
condition that justifies each. Reach for it only when a step comes up short.

### Step 1 — Doc shell (free)
Write the file: header, input, `Status: in-progress`.

### Step 2 — Resolve identity

**One paid call, either entry point, then free web search on top — always.**
Announce the cost first. There is no paid fallback chain: if the call comes up
empty, free search is the whole recovery; if that also fails, the contact is
unresolvable (terminal state below).

```bash
# email path
taskfuel call https://mpp.orthogonal.com/apollo/api/v1/people/match \
  --method POST --body '{"email":"<email>"}' --max-amount 0.01
# name+company path
taskfuel call https://mpp.orthogonal.com/apollo/api/v1/people/match --method POST \
  --body '{"first_name":"<F>","last_name":"<L>","organization_name":"<Co>"}' \
  --max-amount 0.01
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
  **`WebFetch` that page and cite it.** A summary is confabulation risk, not evidence.
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
product-relevant repos, protocol/spec forks. No org is an **explicit negative**,
not a skipped slot.

**d) Paid fallback — Apollo `organizations/enrich` ($0.01)** is an opt-in add-on;
Step 2's `organization` object usually makes it unnecessary. See
[`ADD-ONS.md`](ADD-ONS.md).

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
proves individual depth a job title cannot. An **explicit negative** here is load-bearing:
it steers openers away from a technical hook for a non-technical contact.

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
- **⚠ Every date comes off a source page** — summaries confabulate (Step 2), and
  a wrong date in a cold email is worse than a missing one. `WebFetch` the article
  or read a dated URL slug before promoting anything to a trigger event.
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
- Nothing found? The **explicit negative** is what lets the verdict be SEND.

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
- **Write the window as a window.** "He posted about hiring in March 2025" is
  fabrication when the source supports only "sometime in 2024–2025".
- **To promote a post to a trigger event, `WebFetch` its `postUrl`** and read the
  real date.
- **No real date? Keep the item, drop the date** — "undated, LinkedIn window
  2024-08→2025-08". A hook survives undated; a trigger event does not.
- The deep tail is old — good for interests and common ground, **useless for "why
  now."**

**Twitter/X** — use the **verified** handle from Step 4.
`GET /users/by/username` ($0.005) + `GET /tweets/user` ($0.01, ≤20 tweets). A
404/empty means the handle is wrong — go back to web search, don't keep paying
against a bad handle. `GET /workflows/userInsights` ($0.50) is **explicit opt-in
only**. **Never call twit.sh write/action endpoints** (follow/like/retweet/tweet/
delete/setProfile/bookmark) — read-only research.

**Personal site/blog** — `GET https://x402.ottoai.services/web-extract?url=…`
($0.005), clean LLM-ready markdown. It renders JS and PDFs itself, so reach for
`POST /api/firecrawl/scrape` ($0.0126) only when web-extract returns an empty or
boilerplate body. **Neither may be pointed at LinkedIn** — Firecrawl refuses it
outright.

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

**Before Step 7, each slot holds a result or an explicit negative:**

| Slot | Minimum bar |
|---|---|
| **Landmines / friction** | ≥1. Search `"{name}" criticism OR rejected OR banned OR lawsuit OR controversy` and their own posts for public fights. |
| **Interests (non-work)** | ≥2 sourced. |
| **Personal technical footprint** | a verified profile. |
| **90-day company events** | a verdict-bearing item *(Step 5 gate)*. |
| **Common ground** | ≥1 candidate. |

**A blank fails the brief** — every slot holds a result or an explicit negative.

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

### Default run — $0.067, either entry point

| Step | Call | Cost |
|---|---|---|
| 2 | Apollo `people/match` + free websearch | **$0.01** |
| 3–4 | free (Apollo's `organization` object, DNS/headers, search) | $0 |
| 5 | Fiber LinkedIn posts | **$0.04** |
| 5 | twit.sh profile + timeline | **$0.015** |
| 5 | Serper news + 90-day gate | **$0.002** |
| 6–7 | free | $0 |

**Step 2 is flat with no tail** — one paid call, no escalation chain. A miss
costs $0 extra; an unresolvable contact stops the run rather than spending more.

**Opt-in add-ons** — priced, with the condition that justifies each, in
[`ADD-ONS.md`](ADD-ONS.md). In-line extras not listed there: ottoai extract
+$0.005 (Firecrawl +$0.0126 if it returns empty).

- **Steps 3–4 and Step 5's interests are free** — this halved run cost *and*
  improved quality. But free ≠ optional: Step 6.5 fails the brief on a blank.
- **Fiber is the largest line item** at $0.04. Skip it (→ ~$0.027) only when the
  contact has no LinkedIn presence.
- **The phone is the biggest cost lever** — $0.28 for one field, 4× the whole
  default run. Only buy it when the rep says they will dial.
