---
name: taskfuel
version: 0.2.2
description: Let an agent discover and call paid HTTP-402 APIs (search, market data, enrichment, and more) through the user's taskfuel.ai account — no crypto, paid from their prepaid balance. Use when the user asks the agent to buy/call a paid API, mentions taskfuel.ai, or a task needs a paid capability (web search, tweet search, market data) the agent lacks.
---

# taskfuel.ai — paid APIs for your agent

taskfuel.ai gives this machine's user one account for the paid web: they hold a
prepaid USD balance, and the `taskfuel` CLI lets you (the agent) spend it on
pay-per-call APIs from a curated list of provider domains. You never touch a
wallet or a payment protocol — the taskfuel.ai gateway pays the upstream service
and bills the user's balance.

Base URL: the gateway defaults to the production API; during local development
`FOURZEROTWO_BASE_URL` (or `~/.config/taskfuel/config.json`) may point at a
local instance instead. Don't override it unless the user says so.

## 0. Ensure the CLI is installed

```sh
command -v taskfuel || curl -fsSL ${FOURZEROTWO_BASE_URL:-https://app.taskfuel.ai}/install.sh | sh
```

If `~/.local/bin` is not on PATH, the installer says so — follow its hint.

## 1. Ensure the account is connected

```sh
taskfuel whoami        # prints the connected account; errors if not connected
```

- Works → you're connected.
- "not connected" → run `taskfuel connect`. It prints a **pairing code and a
  URL**, and tries to open the user's browser (best-effort — it may not on a
  headless/SSH box). **Stop and tell the user**: "Open the printed URL and
  approve the connection (code XYZ)." Wait for the command to finish, then
  re-check `taskfuel whoami`.

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
taskfuel discover <method> <url>         # one endpoint's full docs (params, body schema)
taskfuel discover <domain>               # every endpoint one service offers (e.g. twit.sh)
taskfuel discover                        # the allowed service domains, with descriptions
```

Search and listing results show `METHOD  URL  SUMMARY`. To learn an
endpoint's exact parameters and request-body schema before calling, paste the
method and URL back as a request line: `taskfuel discover POST https://…`.

## 3. Call an endpoint

```sh
taskfuel call <url> --dry-run            # check the real price from the 402 challenge — nothing paid
taskfuel call <url>                      # the paid call (GET by default)
taskfuel call <url> --method POST --body '{"query":"…"}'
```

`--dry-run` (alias `--quote`) sends the real request and reads the price off
the endpoint's HTTP-402 challenge without paying — it is the authoritative
price for that exact payload, not an estimate. One caveat: a **free** endpoint
never issues a 402, so its "quote" executes the request and returns the
response at `paid $0.00` (the only way to learn a price is to ask).

The response body prints to stdout (pipe it, e.g. `| jq`); the cost line
prints to stderr — "quoted $0.01 — nothing paid" for quotes, "paid $0.0070 —
balance $4.82 remaining" for real calls.

## Spending rules (non-negotiable)

- **Preview before paying**: `--quote` the first call to any endpoint in a
  session. The quoted amount is the real charge for that payload (a hard
  per-call cap also protects you from surprises).
- **Ask before big or repeated spends**: single calls over $0.10, or loops
  that will make more than ~5 paid calls, need the user's explicit OK first.
- **Never retry a failed paid call blindly.** If a call errors after payment
  ("paid" line printed but bad response), check `taskfuel balance` to see
  whether it was charged, report to the user, and let them decide.
- **Stop at low balance**: if balance < the next call's price, stop and tell
  the user to top up at their dashboard — don't hunt for workarounds.

## Errors you may see

The CLI prints the gateway's message as plaintext (`error: …`) — recognize
the phrasing:

- "Your balance is empty. Top up …" — credits exhausted; tell the user to
  top up at https://app.taskfuel.ai, then stop.
- "taskfuel.ai only calls upstreams on its allowed domains …" — that URL's
  domain isn't callable; `taskfuel discover` lists what is, and keyword
  search finds alternatives.
- "Upstream price $X exceeds the per-call cap $Y." — the endpoint costs more
  than the hard cap allows; don't retry, report the price to the user.
- "Rate limit exceeded — try again in a minute." — wait, don't hammer.
