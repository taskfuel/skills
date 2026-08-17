---
name: deploy-app
description: Deploy a web app to an anonymous metered vm402 VM with taskfuel.ai integrated, so the deployed app can call any taskfuel paid service (LLMs, image gen, search, etc.) server-side. Use when deploying/hosting an app on vm402, wiring an app to the taskfuel gateway, or debugging a vm402 managed service.
license: MIT
---

# Deploy an app on vm402 with taskfuel integrated

Goal: a Node (or any) web app running on a vm402 VM, publicly reachable, that calls paid APIs
(LLM, image gen, anything in `taskfuel discover`) through the taskfuel gateway with its own API
key. Docs: https://vm402.com/llms.txt, https://vm402.com/docs, https://app.taskfuel.ai/openapi.json

## Critical facts (violating these breaks everything)

- **Processes started via `POST /vms/{id}/exec` die when the command returns.** Never launch a
  server with `&` or `nohup`. Only a *managed service* keeps running.
- **The VM sleeps ~30s after activity stops and runs NO code while asleep** — timers,
  setInterval, cron, webhooks are all frozen. It wakes in <1s on any request to its URL.
  Design apps to reconcile state lazily on incoming requests ("reconcile on wake").
- **The VM bearer `token` from creation is shown once.** Save it to a protected file immediately.
- **A taskfuel API key is delivered exactly once** by the connect flow. Write the raw poll
  response to disk BEFORE parsing it — a parsing bug otherwise loses the key forever.
- **Paid calls in automated tests are forbidden** — stub the taskfuel layer in tests.

## Step 1 — Buy the VM (the only paid vm402 call)

`POST https://vm402.com/vms` is a paid endpoint (~$0.01/hour in USDC via x402, or 10
sats/hour via L402). Pay it through taskfuel (the gateway settles it):

```bash
taskfuel call https://vm402.com/vms --method POST --body '{"hours": 24}'
```

The response contains `id`, `token` (bearer for all later calls — SHOWN ONCE), `url`,
`expires_at`. Immediately save `{"id": ..., "token": ...}` to `.secrets/vm402-credentials.json`
(chmod 600, never commit). Later: `POST /vms/{id}/topup` (also paid, no token needed) adds time;
`GET /vms/{id}/meter` is public. Expired VMs are parked (disk kept), revived by topup, deleted
after 30+ days parked.

## Step 2 — Mint the app its own taskfuel API key

Give the deployed app its OWN key (revocable independently of your CLI's key), via the
device-code flow — no auth required to start:

```bash
curl -fsSL -X POST https://app.taskfuel.ai/v1/connect/start
# → {"code": "...", "verification_url": "https://app.taskfuel.ai/connect?code=...", "poll_interval": 2}
```

Show the URL + code to the user; they approve in a browser. Then poll:

```bash
curl -fsSL "https://app.taskfuel.ai/v1/connect/poll?code=CODE"
# pending → {"status":"pending"}
# approved → {"status":"approved","key":"sk-402-..."}   ← field is `key`, delivered ONCE
# afterwards → 404 (request deleted)
```

Poll loop rule: write the raw response to a chmod-600 file first, then parse. A naive poller
that greps for the wrong field name (e.g. `api_key` instead of `key`) consumes the
single-delivery key and loses it forever — this happened; use this exact pattern (verified
working):

```bash
OUT=.secrets/taskfuel-vm-key.json
touch "$OUT" && chmod 600 "$OUT"
for i in $(seq 1 300); do
  resp=$(curl -fsSL "https://app.taskfuel.ai/v1/connect/poll?code=$CODE" 2>/dev/null)
  # save ANY non-pending response BEFORE inspecting it — the key is delivered exactly once
  if [ -n "$resp" ] && ! echo "$resp" | grep -q '"pending"'; then
    printf '%s' "$resp" > "$OUT"
    grep -q '"approved"' "$OUT" && { echo "APPROVED"; exit 0; } \
      || { echo "NON-PENDING RESPONSE SAVED (inspect $OUT)"; exit 2; }
  fi
  sleep 2
done
echo "TIMED OUT"; exit 1
```

Gotcha: if you run this in the background and later `pkill -f <scriptname>`, the pkill pattern
can match your own compound shell command and kill it (exit 144) — pkill from a command that
doesn't contain the script name.

## Step 3 — Upload the app

Tarball → upload → extract → install (uploads are raw bytes, binary-safe, no size cap):

```bash
tar czf bundle.tar.gz --exclude=node_modules --exclude=.git --exclude=.secrets .
curl -X PUT "https://vm402.com/vms/$ID/files?path=/root/bundle.tar.gz" \
  -H "Authorization: Bearer $TOKEN" --data-binary @bundle.tar.gz
curl -X POST https://vm402.com/vms/$ID/exec -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"command": "mkdir -p /app && tar xzf /root/bundle.tar.gz -C /app && cd /app && npm install"}'
```

Exec has a 120s default timeout; long installs may need chunking. The filesystem persists
across sleep and between commands.

## Step 4 — Register the managed service (with the taskfuel key as env)

```bash
curl -X PUT https://vm402.com/vms/$ID/services/web -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"cmd": "node", "args": ["server.js"], "dir": "/app",
       "env": {"PORT": "8080", "TASKFUEL_API_KEY": "sk-402-..."},
       "http_port": 8080}'
```

The response includes startup output — a crash-on-boot is visible immediately. The service
auto-starts on incoming requests and survives sleep/wake. One service holds `http_port` at a
time. Then make the URL public:

```bash
curl -X PATCH https://vm402.com/vms/$ID/url -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"visibility": "public"}'
```

App is live at `https://$ID.vm402.com`.

## Step 5 — The app calls paid services through the gateway

Everything in `taskfuel discover` is reachable server-side via one endpoint — plain fetch, no
SDK. Example (LLM via BlockRun):

```js
const r = await fetch('https://app.taskfuel.ai/v1/call', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.TASKFUEL_API_KEY}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    url: 'https://blockrun.ai/api/v1/chat/completions',
    method: 'POST',
    body: {
      model: 'zai/glm-5.2',
      messages: [{ role: 'user', content: 'Hello' }],
      max_tokens: 500,
    },
    maxAmountUsd: 0.02,          // hard cap per call — ALWAYS set it
  }),
});
const data = await r.json();     // upstream response passes through verbatim
// r.headers: x-taskfuel-cost (USD charged), x-taskfuel-balance (remaining)
```

- Binary responses (image gen) pass through too — check content-type, stream to disk.
- During development, add `dryRun: true` to read the real price without paying; in production
  skip dryRun and set a fixed `maxAmountUsd` (known price + headroom).
- Request body limit: 1 MiB. Rate limits → 429; empty balance → 402.
- Track spend with the `x-taskfuel-cost` header; enforce an app-level daily budget cap for
  anything user-triggerable.

## Operating

```bash
# status / logs / restart after redeploying code
curl https://vm402.com/vms/$ID/services -H "Authorization: Bearer $TOKEN"
curl "https://vm402.com/vms/$ID/services/web/logs?lines=100" -H "Authorization: Bearer $TOKEN"
curl -X POST https://vm402.com/vms/$ID/services/web/restart -H "Authorization: Bearer $TOKEN"
```

Redeploy = re-upload tarball (or just changed files via `PUT /files`) + extract + restart.

## Gotchas recap

- Client keeps VM awake only while requests flow (e.g. polling from open tabs); design for
  "everyone left → VM sleeps → next visitor wakes it".
- Open exec sessions/connections keep the VM awake (and the meter is wall-clock) — close them.
- `GET /vms/{id}/meter` is public; anyone can top up the VM without the token.
