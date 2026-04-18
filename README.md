# local-llm

Containerized Caddy + LiteLLM in front of an existing native Lemonade host service.

## Implemented topology

- **Tailscale Serve/Funnel** keeps terminating HTTPS on `:443`
- **Caddy** runs in Docker with **host networking** on `127.0.0.1:8000`
- **LiteLLM** runs in Docker with **host networking** on `127.0.0.1:4000`
- **Native Lemonade** stays on the host and is proxied through Caddy

Current host assumptions detected during implementation:

- Tailscale Funnel is already configured for `https://lemonade.tailc224aa.ts.net`
- Tailscale currently proxies `443 -> http://localhost:8000`
- Native Lemonade is live on `127.0.0.1:13305`
- Native Lemonade already exposes an OpenAI-compatible API on `/v1`

## Routing

- `/` -> native Lemonade
- `/v1/*` -> native Lemonade
- `/litellm/*` -> LiteLLM

That keeps the current Lemonade UI/API intact while exposing LiteLLM as a separate OpenAI-compatible endpoint at:

```text
https://lemonade.tailc224aa.ts.net/litellm/v1
```

## Files

- `docker-compose.yml` - Caddy + LiteLLM stack
- `caddy/Caddyfile` - path-based reverse proxy
- `litellm/config.yaml` - model aliases and auth
- `.env.example` - deployment variables

## Setup

1. Install Docker Engine and the Docker Compose plugin on the host.
2. Copy the environment file and fill in the real local settings:

   ```bash
   cp .env.example .env
   ```

   If Lemonade does not require auth on `/v1`, keep:

   ```bash
   LEMONADE_API_KEY=none
   ```

3. Start the stack:

   ```bash
   docker compose up -d
   ```

4. Confirm Tailscale still points at Caddy on `localhost:8000`:

   ```bash
   tailscale serve status
   ```

If you need to recreate the current Funnel mapping, the backend should remain:

```bash
tailscale serve funnel 443 http://localhost:8000
```

## Rollout

1. Leave the native Lemonade service running on `127.0.0.1:13305`.
2. Bring up LiteLLM and Caddy with Docker Compose.
3. This stack uses **host networking** so the containers can reach host-only services bound to `127.0.0.1`.
4. Test the LiteLLM path first:

   ```bash
   curl -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
     https://lemonade.tailc224aa.ts.net/litellm/v1/models
   ```

5. After `/litellm/v1` works, point Continue, autonomous agents, and mobile clients at that base URL.

## Client configuration

### Continue (VS Code)

Use:

- **Base URL:** `https://lemonade.tailc224aa.ts.net/litellm/v1`
- **API key:** the value of `LITELLM_MASTER_KEY`

### Mobile apps like Liquid Apollo

Use the same OpenAI-compatible base URL:

- **Base URL:** `https://lemonade.tailc224aa.ts.net/litellm/v1`
- **API key:** the value of `LITELLM_MASTER_KEY`

### Available LiteLLM aliases

- `coding`
- `gemma4-general`
- `qwen-chatbot-fast`
- `qwen-chatbot-think`
- `qwen-deep`

These aliases route back to the native Lemonade OpenAI-compatible API, so your local Lemonade models are available through LiteLLM with cleaner names.

- `gemma4-general` - Gemma 4 31B general-purpose profile
- `qwen-chatbot-fast` - Qwen 35B general-purpose profile for lower latency
- `qwen-chatbot-think` - Qwen 35B chatbot profile backed by the HauhauCS aggressive Q6 model
- `qwen-deep` - Qwen 122B general-purpose profile for heavier reasoning

If you want different upstream models or alias names, edit `litellm/config.yaml`.

## Operations

- **Logs:** `docker compose logs -f caddy litellm`
- **Updates:** `docker compose pull && docker compose up -d`
- **Secrets:** keep real credentials only in `.env`
- **Backups:** preserve `.env`, `litellm/config.yaml`, and the Docker volumes `caddy-data` / `caddy-config`
