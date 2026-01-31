# OpenClaw on Zerops

Zerops recipe for [OpenClaw](https://github.com/openclaw/openclaw) — an AI coding gateway that unifies access to multiple LLM providers (OpenAI, Anthropic, Google, etc.) behind a single API.

## Why the wrapper

OpenClaw gateway is designed to run locally. Three things break when you put it behind a cloud load balancer:

1. **Proxy trust** — Gateway validates caller IPs against an exact `trustedProxies` list (no CIDR). The Zerops L7 balancer IP is unknown and can change. The wrapper proxies from `127.0.0.1`, so `trustedProxies: ["127.0.0.1"]` always works.

2. **Onboarding** — Gateway is configured via interactive CLI (`openclaw onboard`). The wrapper serves a `/setup` web UI that calls `openclaw onboard --non-interactive` with the right flags.

3. **Auth token** — Gateway requires a Bearer token on every request. The wrapper injects it automatically into all proxied HTTP and WebSocket requests.

## How it works

```
Internet → Zerops L7 balancer → Express wrapper (:8080) → OpenClaw gateway (:18789)
                                       │
                                  /setup    → web setup wizard (Basic auth)
                                  /openclaw → Control UI (token auto-injected)
                                  /*        → proxy to gateway
```

**Lifecycle:**
1. First deploy — wrapper starts, gateway is off, all requests redirect to `/setup`
2. User opens `/setup` (protected by `SETUP_PASSWORD` via HTTP Basic auth)
3. User picks LLM provider + pastes API key → wrapper runs `openclaw onboard --non-interactive`
4. Wrapper sets `trustedProxies`, auth config, optional chat channels (Telegram/Discord/Slack)
5. Gateway starts on loopback `:18789`, wrapper proxies everything to it

## Quick start

1. Import into a Zerops project:
   ```bash
   zcli project service-import import.yaml --projectId <PROJECT_ID>
   ```
2. Open `https://<subdomain>.zerops.app/setup`, authenticate with `SETUP_PASSWORD` from env vars
3. Pick provider, paste API key, click **Run setup**

## Services

| Service   | Type             | Purpose                            |
|-----------|------------------|------------------------------------|
| `openclaw`| `nodejs@22`      | Express wrapper + gateway on :8080 |
| `storage` | `shared-storage` | Persistent state at `/mnt/storage` |

## Env vars

Auto-generated secrets (`import.yaml`):
- `SETUP_PASSWORD` — password for `/setup` wizard
- `OPENCLAW_GATEWAY_TOKEN` — Bearer token for gateway API

Runtime (`zerops.yml`):
- `NODE_ENV=production`
- `NODE_OPTIONS="--max-old-space-size=1536 --no-deprecation"`
- `OPENCLAW_STATE_DIR=/mnt/storage/.openclaw` — gateway config and state
- `OPENCLAW_WORKSPACE_DIR=/mnt/storage/workspace`

## Files

- `import.yaml` — Zerops service definitions (creates `openclaw` + `storage`)
- `zerops.yml` — build, deploy, run config (npm install, health checks, env vars)
- `src/server.js` — Express wrapper: proxy, setup API, gateway lifecycle, graceful shutdown
- `src/public/setup.html` — setup wizard frontend (Alpine.js)
- `src/public/styles.css` — dark/light theme styles

## Updating

Trigger a rebuild in Zerops GUI or `zcli push` — pulls latest `openclaw` from npm.
