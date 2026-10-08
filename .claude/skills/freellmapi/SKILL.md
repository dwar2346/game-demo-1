---
name: freellmapi
description: Use when setting up, running, or calling FreeLLMAPI — the self-hosted gateway (github.com/tashfeenahmed/freellmapi) that pools free LLM provider tiers behind one OpenAI- and Anthropic-compatible endpoint. Covers install (Docker or Node), first-run account and unified key, adding provider keys, calling /v1/chat/completions and /v1/messages, auto:* routing, reading X-Routed-Via / fallback headers, and pointing Claude Code or OpenAI SDKs at it.
---

# FreeLLMAPI

A single-user gateway that aggregates ~40 free LLM providers (Groq, Gemini, Cerebras,
OpenRouter, Mistral, NVIDIA NIM, GitHub Models, …) behind one `/v1` endpoint. A router picks
a model per request, fails over when a provider is rate-limited, and tracks per-key quotas.

- Default port: **3001** — dashboard at `http://localhost:3001`, API at `http://localhost:3001/v1`
- Auth for the API: the **unified key** (`freellmapi-…`), not the individual provider keys
- Interactive API docs: `GET /v1/docs`; OpenAPI spec: `GET /v1/openapi.json`

## 1. Run it

Pick one. Check first whether something already answers on 3001:
`curl -s localhost:3001/api/auth/status`.

**Docker (recommended for real use)**
```bash
curl -fsSL https://freellmapi.co/install.sh | bash   # creates ~/freellmapi, .env, starts container
```
Read the script before piping it to bash if the user hasn't approved that. Or use the repo's
`docker-compose.yml` (`docker compose up -d`). Docker publishes on 127.0.0.1 only unless
`HOST_BIND=0.0.0.0` is set.

**From source (Node 20.18–24, npm ≥ 10)**
```bash
git clone --depth 1 https://github.com/tashfeenahmed/freellmapi.git && cd freellmapi
npm ci
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$(node -e 'console.log(require("crypto").randomBytes(32).toString("hex"))')" > .env
npm run dev                         # server :3001 + Vite dashboard :5173 (hot reload)
# or production-style, one port:
npm run build && NODE_ENV=production node server/dist/index.js
```
If `npm ci` hangs: the upstream `package-lock.json` resolves some tarballs from
`registry.npmmirror.com`, which restricted networks block. Rewrite them (integrity hashes still apply):
`sed -i 's#https://registry.npmmirror.com/#https://registry.npmjs.org/#g' package-lock.json`.

Run long-lived servers in the background and poll `/api/auth/status` until it answers.
`ENCRYPTION_KEY` encrypts stored provider keys — losing it makes them undecryptable.

## 2. First-run setup and the unified key (headless)

The first account can be created without a setup code only from loopback.

```bash
B=http://localhost:3001
# create the admin account (only works while no user exists) → returns a session token
TOKEN=$(curl -s -X POST $B/api/auth/setup -H 'Content-Type: application/json' \
  -d '{"email":"admin@example.com","password":"a-long-password"}' | node -pe 'JSON.parse(require("fs").readFileSync(0)).token')
# later sessions: POST $B/api/auth/login with the same body

# unified API key used by every client
KEY=$(curl -s $B/api/settings/api-key -H "Authorization: Bearer $TOKEN" | node -pe 'JSON.parse(require("fs").readFileSync(0)).apiKey')
```
Remote first-run requires the one-time `setupCode` printed in the server log.
Alternative: declarative config via `FREEAPI_CONFIG_JSON` / `FREEAPI_CONFIG_PATH`, e.g.
`{"admin":{"email":"…","password":"…"},"keys":[{"platform":"groq","key":"gsk_…"}]}` — applied on every boot.

## 3. Add provider keys

Nothing routes until at least one provider is usable. Dashboard → **Keys**, or:
```bash
curl -s -X POST $B/api/keys -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"platform":"groq","key":"gsk_...","label":"main"}'
```
Platform ids: `google`, `groq`, `cerebras`, `openrouter`, `mistral`, `nvidia`, `github`, `cohere`,
`cloudflare` (`account_id:token`), `huggingface`, `zhipu`, `ollama`, `llm7`, `pollinations`, …
(full list: `docs/en/providers/01-supported-platforms.md`). **Keyless**: `kilo`, `ovh`, `aihorde` —
these work with no signup but are slow / heavily rate-limited (and Kilo logs prompts).
`GET /api/keys/providers` lists what this install supports.

Never ask the user to paste provider keys into chat if avoidable; prefer they enter them in the
dashboard. Never commit `.env` or keys.

## 4. Call it

**OpenAI shape**
```bash
curl $B/v1/chat/completions -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"model":"auto","messages":[{"role":"user","content":"hi"}]}'
```
```python
from openai import OpenAI
c = OpenAI(base_url="http://localhost:3001/v1", api_key=KEY)
r = c.chat.completions.with_raw_response.create(model="auto", messages=[{"role":"user","content":"hi"}])
print(r.parse().choices[0].message.content, r.headers.get("x-routed-via"))
```

**Model strings**
- `auto` — follows the dashboard fallback chain
- `auto:fast` | `auto:smart` | `auto:reliable` | `auto:balanced` | `auto:cheap` — rank all enabled models by that axis
- a concrete id (e.g. `gemini-2.5-flash`), a named chain, or `fusion` (multi-model synthesis)
- `GET /v1/models?execution_status=ready` — only models callable right now (`needsKey` / `exhausted` are the other states)

Also supported: streaming (`"stream": true`), tool calling, vision, `/v1/embeddings`,
`/v1/responses`, native Gemini (`/v1beta`), Ollama emulation (`/api/*`), image/TTS media.

**Anthropic shape / Claude Code**
```bash
curl $B/v1/messages -H "x-api-key: $KEY" -H 'anthropic-version: 2023-06-01' -H 'Content-Type: application/json' \
  -d '{"model":"claude-sonnet-4-5","max_tokens":256,"messages":[{"role":"user","content":"hi"}]}'

export ANTHROPIC_BASE_URL=http://localhost:3001   # origin, no /v1
export ANTHROPIC_AUTH_TOKEN=$KEY                    # NOT ANTHROPIC_API_KEY — Claude Code refuses that combo
claude
```
Claude model families map to free models under Keys → Anthropic (`PUT /api/settings/anthropic-map`).

## 5. Debugging a request

| Signal | Meaning |
| --- | --- |
| `X-Routed-Via: <platform>/<model>` | who actually answered |
| `X-Fallback-Attempts`, `X-Fallback-Trail` | hops that failed and why (`rate_limited`, `timeout`, …) |
| `X-Fallback-Detail` | per-hop timing; enable with `FALLBACK_DETAIL_HEADER=1` |
| 401 | wrong/missing unified key (provider keys don't work here) |
| 503 / "no models available" | no usable provider key, or all are on cooldown — check `/v1/models?execution_status=ready` |
| `X-Fallback-Trail: … =timeout` on every hop | the host can't reach the provider (sandbox/firewall egress policy) — test with `curl https://api.groq.com/openai/v1/models` |
| 413 `request_too_large` | raise `REQUEST_BODY_LIMIT_MB` |

Per-request headers: `X-FreeLLM-Cache: on|off`, `X-FreeLLM-Compress: off|lossless|standard|aggressive`,
`X-FreeLLM-Task-Type: code|chat|auto`. Common env knobs live in `.env.example`
(`PROXY_RATE_LIMIT_RPM`, `PROVIDER_TIMEOUT_<PLATFORM>`, `RESPONSE_CACHE`, `MODEL_ROUTING_OVERRIDES`).
More: `docs/en/troubleshooting/01-common-issues.md`.

## Caveats to tell the user

- Free tiers change constantly; some providers log prompts or train on them — don't send secrets.
- It's single-user: anyone with the unified key can spend all pooled quotas. Keep it on localhost unless
  behind auth/TLS (`TRUST_PROXY`, `FREEAPI_BLOCK_PRIVATE_PROVIDER_URLS=true` on a VPS).
- Respect each provider's terms of service; one key per account, no multi-account farming.
