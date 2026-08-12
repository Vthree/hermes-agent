# Railway: Hermes as SuperGrok OpenAI-compatible proxy

> For AI agents / ops taking over **Vthree/hermes-agent** fork deployment.  
> Upstream: NousResearch/hermes-agent. This fork adds Railway proxy packaging only.

## Goal

Run **only** `hermes proxy` on Railway so sibling bots (telegram-grok-bot, discord-grok-bot) can call SuperGrok OAuth models via private networking, without paying xAI API for those requests when the subscription path works.

**Not in scope on Railway:** Telegram/Discord **gateway** with user bot tokens (that stays on the owner’s Windows Hermes Agent).

## Files in this fork

| Path | Purpose |
|------|---------|
| `railway.toml` | `Dockerfile.railway`, startCommand proxy on `$PORT` |
| `Dockerfile.railway` | Same as upstream Dockerfile **without** Docker `VOLUME` (Railway rejects VOLUME) |
| `docs/RAILWAY_PROXY.md` | This document |

## railway.toml (canonical)

```toml
[build]
builder = "DOCKERFILE"
dockerfilePath = "Dockerfile.railway"

[deploy]
startCommand = "sh -c 'exec hermes proxy start --provider xai --host 0.0.0.0 --port ${PORT:-8645}'"
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 10
```

- **No** `healthcheckPath` unless you attach a public domain (internal-only service).
- Bind `0.0.0.0` so Railway private network can reach the process.
- Prefer `$PORT` from Railway; bots use fixed internal port if you also set service var `PORT=8645`.

## Required Railway resources

1. Service name: `hermes` (DNS: `hermes.railway.internal`).
2. Volume mount: **`/opt/data`** (= `HERMES_HOME`) — persists `auth.json`.
3. Same Railway **project** as the bots (private networking).
4. Env optional: `HERMES_HOME=/opt/data`, `PORT=8645`.

## Auth bootstrap (SuperGrok OAuth)

Proxy exits/restarts if not logged in:

```text
Not logged into xAI Grok OAuth. Run `hermes auth add xai-oauth` first.
```

**Recommended bootstrap (used in production):**

1. Temporarily set startCommand to `sleep infinity`, deploy, wait Online.
2. Upload local Windows Hermes auth file to volume root as `/auth.json`  
   (source: `%LOCALAPPDATA%\hermes\auth.json` on owner PC).
3. Restore proxy startCommand and redeploy.
4. Verify: `curl http://127.0.0.1:8645/health` → `authenticated: true`.

Alternative: interactive `hermes auth add xai-oauth` via Railway SSH/web terminal (device code in browser).

Token refresh is handled by Hermes (~6h access token); volume must stay mounted.

## Endpoints

| Path | Notes |
|------|-------|
| `GET /health` | `{status, upstream, authenticated}` |
| `GET /v1/models` | Any bearer |
| `POST /v1/chat/completions` | OpenAI-compatible |
| `POST /v1/responses` | Used by bots for web_search |

Proxy accepts **any** `Authorization: Bearer …` and attaches real OAuth credentials.  
**Do not** expose publicly without an extra auth layer.

## Bot configuration

```env
HERMES_ENABLED=on
HERMES_BASE_URL=http://hermes.railway.internal:8645/v1
```

Bots keep `XAI_API_KEY` for automatic failover.

## Deploy notes / pitfalls

| Issue | Detail |
|-------|--------|
| Docker VOLUME error | Use `Dockerfile.railway`, not stock `Dockerfile` |
| `railway add -r` Unauthorized | GitHub App / CLI agent-scoped token; create empty service or use dashboard |
| Large `railway up` timeout | Prefer `serviceInstanceDeployV2` GraphQL redeploy from linked repo |
| CLI `AI_AGENT=hermes-agent` | Railway may issue read-limited OAuth; use human TTY login for writes |
| Dual gateway | Never put TG/Discord tokens on this Railway hermes service |

## Verify

```bash
railway ssh -s hermes -- curl -s http://127.0.0.1:8645/health
railway ssh -s worker -- curl -s http://hermes.railway.internal:8645/health
```

## Related releases

- telegram-grok-bot **v1.6.0** — client failover
- discord-grok-bot **v1.1.0** — client failover
