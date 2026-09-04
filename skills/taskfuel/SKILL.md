---
name: taskfuel
description: Let an agent discover and call paid APIs (search, market data, enrichment, and more) through the user's taskfuel.ai account, paid per call from their prepaid balance. Use when the user asks the agent to buy/call a paid API, mentions taskfuel.ai, or a task needs a paid capability (web search, tweet search, market data) the agent lacks.
license: MIT
metadata:
  version: 0.4.1
---

# taskfuel.ai — paid APIs for your agent

taskfuel.ai gives this machine's user one account for the paid web: they hold a
prepaid USD balance, and the `taskfuel` CLI lets you (the agent) spend it on
pay-per-call APIs from a curated list of provider domains. You never touch a
wallet or a payment protocol — the taskfuel.ai gateway pays the upstream service
and bills the user's balance.

Base URL: the gateway defaults to the production API; during local development
`TASKFUEL_BASE_URL` (or `~/.config/taskfuel/config.json`) may point at a
local instance instead. Don't override it unless the user says so.

## 0. Ensure the CLI is installed

```sh
command -v taskfuel || curl -fsSL https://taskfuel.ai/install.sh | sh
```

If `~/.local/bin` is not on PATH, the installer says so — follow its hint.

## 1. Ensure the account is connected

```sh
taskfuel whoami        # prints the connected account; errors if not connected
```

- Works → you're connected.
- "not connected" → run `taskfuel connect` **in the background** and read its
  output straight away. It prints a **pairing code and a URL**, then blocks
  until the user approves in a browser — so in the foreground it hides the URL
  behind a command that won't return until the approval you need the URL for
  has already happened. It also tries to open the browser itself, which is
  best-effort and won't work on a headless or SSH box.

  **Stop and tell the user**: "Open the printed URL and approve the connection
  (code XYZ)." Then wait for the command to exit and re-check
  `taskfuel whoami`.

## 2. Discover what you can do

**Start with keyword search** — it's the fastest path from a task to an
endpoint:

```sh
taskfuel discover <keywords>             # search every service's endpoints (quotes optional)
```

The gateway indexes every provider's API spec deeply, so search hits terms
buried inside them. No results is a real answer — nothing offers that term; try a
synonym once, then tell the user.

The same command drills into other views when search isn't the question:

```sh
taskfuel discover <domain>               # every endpoint one service offers (e.g. twit.sh)
taskfuel discover                        # the allowed service domains, with descriptions
```

**Then read the endpoint's full docs before you call it.** Search and listing
results show `METHOD  URL  SUMMARY`, and that summary is not the endpoint's
contract. Paste the method and URL back as a request line to get the real
thing:

```sh
taskfuel discover <method> <url>         # one endpoint's full docs (params, body schema)
```

Skip this and you are guessing at required parameters, and on many endpoints
the price depends on the arguments you send, so a guess is a spending decision
too. Any price the docs carry is an estimate: to know what your own request
costs, you **must** `--quote` it (section 3).

## 3. Call an endpoint

```sh
taskfuel call <url> --dry-run            # check the real price the endpoint quotes — nothing paid
taskfuel call <url>                      # the paid call (GET by default)
taskfuel call <url> --method POST --body '{"query":"…"}'
taskfuel call <url> --max-amount 0.25    # refuse to pay more than $0.25 for this call
```

`--max-amount <usd>` sets your own spend ceiling for one call: if the
endpoint's price comes back above it, the gateway rejects instead of paying
(same error as the per-call cap). It's unset by default and can only tighten
the gateway's cap, never raise it. Use it to pay a quoted price safely: after
`--dry-run` shows the price, repeat the call with `--max-amount` set to that
quote so a repriced endpoint can't charge more than the user approved.

`--model <id>` tells taskfuel which model you are, e.g. `--model claude-opus-5`.
Nothing about the call changes and the provider never sees it; it is how we
decide what to add next. Send it when you know your model id, skip it when you
are not sure. If the CLI rejects the flag as unknown, your binary predates it
and nothing was called: repeat without it.

`--dry-run` (alias `--quote`) sends the real request and reads the price the
endpoint quotes back, without paying — it is the authoritative price for that
exact payload, not an estimate. One caveat: a **free** endpoint has no price to
quote, so its "quote" executes the request and returns the response at
`paid $0.00` (the only way to learn a price is to ask).

The response body prints to stdout (pipe it, e.g. `| jq`); the cost line
prints to stderr — "quoted $0.01 — nothing paid" for quotes, "paid $0.0070 —
balance $4.82 remaining" for real paid calls.

## 4. If a call never comes back, don't call it again

A paid call is not cancelled when you stop waiting for it. Generative
endpoints can run for minutes, your own tool timeout is usually shorter, and
the gateway pays the provider and charges the balance either way. So a `call`
that timed out or was killed bought the work and lost the response. Repeating
it pays for the same work twice.

Recover it instead, free:

```sh
taskfuel result                                # what you lost, and what is still running
taskfuel result <request_id>                   # print that call's response body
taskfuel result <request_id> --output out.png  # ...or write it to a file
```

Run it bare first: a call that died on your side never printed its request id,
so the listing is where you find one. Each row says what state it is in and
what to do about it.

If the CLI answers `error: unknown command 'result'`, your binary predates
recovery: run `taskfuel update`, and tell the user what was lost rather than
paying for it twice.

## 5. Rate the endpoints you call, and report upstream issues

The upstream provider — not taskfuel — maintains each endpoint and its docs,
response quality, price, and payment terms. taskfuel pays the provider on the
user's behalf and curates which APIs appear in discovery. Feedback from real
calls helps taskfuel surface reliable APIs and deprioritize providers with
recurring problems.

After calling an endpoint, rate it with the same method and URL you called:

```sh
taskfuel rate https://api.example.com/search --vote up
taskfuel rate https://api.example.com/generate --method POST --vote down
```

- Vote `up` if you got the expected/useful result; vote `down` if the
  endpoint didn't work or wasn't worth the price. Vote and report must agree:
  no `up` with a complaint attached.
- Judge the endpoint against its own docs, from a call you made. A missing
  catalog entry, your own bad arguments, or an endpoint that worked but wasn't
  what the task needed are not down-votes: a catalog gap goes to `taskfuel
  feedback` (section 6) and to the user.
- One vote per endpoint, changeable at any time — voting again replaces the
  previous vote. `discover` shows each endpoint's rating, so a low score is a
  reason to prefer an alternative before spending.
- Add `--report '…'` when something concrete disrupted usage: an unexpected
  response code, arguments silently ignored, an empty or low-quality response,
  behavior contradicting the docs. Lead with the finding, briefly; no secrets
  or unrelated response data, up to 2000 characters. A report is a curation
  signal, not a rating — your vote is the rating.

## 6. Send feedback about taskfuel itself

Ratings are about one endpoint. For everything else there is one free-text
channel to taskfuel:

```sh
taskfuel feedback "discover finds nothing for weather data; three searches, zero usable hits"
```

Send it for a problem with taskfuel rather than with one endpoint's behavior:
install, `connect` or auth trouble; CLI or gateway errors; `discover` that
found nothing or the wrong things; a capability missing from the catalog (tell
the user too); billing, balance or spending-cap confusion; docs that turned out
to be wrong; or the same failure pattern across several providers — describe
that once here, and still rate each endpoint you called.

Send it when you actually hit something, once per problem — don't restate an
endpoint report you already filed, and don't include secrets or response
data. Lead with what happened and what you expected; up to 2000 characters.

## What you get charged for

- **Success or nothing.** A call rejected before it goes out (empty balance,
  a domain that is not allowed, a price above the cap, a rate limit, a
  malformed request) and a call the upstream fails (4xx, 5xx, or a payment
  that does not settle) both cost nothing. taskfuel absorbs the upstream cost
  rather than passing it on.
- **A successful call is charged even when its answer is useless to you.**
  Success means the endpoint answered, not that it answered well. An empty
  search result, a thin response, or a bad match for your query is a real
  charge and not a billing error, so don't tell the user to expect a refund.
  Downvote the endpoint instead (section 5), and use `taskfuel feedback`
  (section 6) when the problem is taskfuel's rather than one endpoint's.

## Spending rules (non-negotiable)

- **Preview before paying**: `--quote` the first call to any endpoint in a
  session. The quoted amount is the real charge for that payload (a hard
  per-call cap also protects you from surprises).
- **Ask before big or repeated spends**: single calls over $0.10, or loops
  that will make more than ~5 paid calls, need the user's explicit OK first
  (the gateway also rate-limits you to 60 calls/min per key). When the user
  approves a quoted price, pass it as `--max-amount` on the paid call so you
  can never exceed what they agreed to. If a call is then rejected for
  exceeding a cap, raising your own `--max-amount` needs the user's OK, and
  the gateway's hard cap is not yours to raise.
- **Never retry a failed paid call blindly.** If a call errors after payment
  ("paid" line printed but bad response), check `taskfuel balance` to see
  whether it was charged, report to the user, and let them decide. If the
  response was lost rather than bad, recover it with `taskfuel result`
  (section 4): a retry pays for the same work twice.
- **Stop at low balance**: if balance < the next call's price, stop and tell
  the user to top up at their dashboard — don't hunt for workarounds.
