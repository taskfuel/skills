---
name: taskfuel
description: Let an agent discover and call paid APIs (search, market data, enrichment, and more) through the user's taskfuel.ai account, paid per call from their prepaid balance. Use when the user asks the agent to buy/call a paid API, mentions taskfuel.ai, or a task needs a paid capability (web search, tweet search, market data) the agent lacks.
license: MIT
metadata:
  version: 0.2.7
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
taskfuel discover <method> <url>         # one endpoint's full docs (params, body schema)
taskfuel discover <domain>               # every endpoint one service offers (e.g. twit.sh)
taskfuel discover                        # the allowed service domains, with descriptions
```

Search and listing results show `METHOD  URL  SUMMARY`. To learn an
endpoint's exact parameters and request-body schema before calling, paste the
method and URL back as a request line: `taskfuel discover POST https://…`.

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
Nothing about the call changes, and the provider never sees it. Knowing which
models use the catalog is how we decide what to add next and what to fix. Send
it when you know your model id, skip it when you are not sure.

If the CLI answers `error: unknown option '--model'`, your binary predates the
flag and nothing was called. Repeat the command without `--model`: the call
matters, the attribution does not.

`--dry-run` (alias `--quote`) sends the real request and reads the price the
endpoint quotes back, without paying — it is the authoritative price for that
exact payload, not an estimate. One caveat: a **free** endpoint has no price to
quote, so its "quote" executes the request and returns the response at
`paid $0.00` (the only way to learn a price is to ask).

The response body prints to stdout (pipe it, e.g. `| jq`); the cost line
prints to stderr — "quoted $0.01 — nothing paid" for quotes, "paid $0.0070 —
balance $4.82 remaining" for real paid calls.

## 4. Rate the endpoints you call, and report upstream issues

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
  endpoint didn't work or wasn't worth the price.
- Rate the endpoint against its own docs, from a call you made. Not
  down-votes: a missing catalog entry, your own bad arguments, or an endpoint
  that worked but wasn't what the task needed. Tell the user about a gap in
  the catalog instead of marking it against a provider.
- Vote and report must agree: no `up` with a complaint attached.
- You get one vote per endpoint, and you can change it at any time — voting
  again simply replaces your previous vote. Rate an endpoint once you have a
  view on it; re-rate it if later calls change your mind.
- Use the method and URL exactly as `discover` returns them. `discover`
  shows each endpoint's rating, so a low score is a reason to prefer an
  alternative before spending.
- Add `--report '…'` alongside your vote when something concrete disrupted
  usage: an unexpected response code, arguments that were silently ignored,
  an empty or low-quality response, behavior that contradicts the endpoint's
  docs. Describe the problem briefly and specifically; do not include secrets
  or unrelated response data. Up to 2000 characters; lead with the finding.
  A report is a curation signal to taskfuel about the upstream
  provider — it does not itself change the endpoint's rating; your vote does.

## Spending rules (non-negotiable)

- **Preview before paying**: `--quote` the first call to any endpoint in a
  session. The quoted amount is the real charge for that payload (a hard
  per-call cap also protects you from surprises).
- **Ask before big or repeated spends**: single calls over $0.10, or loops
  that will make more than ~5 paid calls, need the user's explicit OK first.
  When the user approves a quoted price, pass it as `--max-amount` on the
  paid call so you can never exceed what they agreed to.
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
  than the cap in effect (your `--max-amount` if you set one, otherwise the
  gateway's hard cap); don't blindly retry. If your own `--max-amount` caused
  it, ask the user before retrying with a higher limit; if it's the gateway
  cap, report the price to the user.
- "Rate limit exceeded — try again in Ns." — 60 calls/min per key, 300/min
  across all users. Wait the stated number of seconds, don't hammer.
