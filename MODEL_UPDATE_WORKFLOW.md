# Lemonade Model Update Workflow

Use this workflow whenever a new model is added to Lemonade and needs a LiteLLM alias.

## Goal

When a model is pulled or added to Lemonade:

1. Update `litellm/config.yaml` in this repo.
2. Push the repo changes.
3. Pull the repo changes on the real Lemonade server.
4. Restart LiteLLM on the real server.
5. Verify the new alias appears in `/litellm/v1/models`.

## Important Context

This repo may be only a local clone.

Updating files here does not update the running Lemonade server until the real host pulls the changes and restarts LiteLLM.

## Step 1: Confirm the new Lemonade model ID

On the real Lemonade host, list the upstream models:

```bash
curl http://127.0.0.1:13305/v1/models
```

Identify the exact model ID that Lemonade exposes. Example:

```text
user.gemma-4-E4B-it-OBLITERATED-Q8_0
```

## Step 2: Add a LiteLLM alias in this repo

Edit `litellm/config.yaml` and add a new entry under `model_list`.

Example:

```yaml
  - model_name: gemma4-E4B
    litellm_params:
      model: openai/user.gemma-4-E4B-it-OBLITERATED-Q8_0
      api_base: os.environ/LEMONADE_API_BASE
      api_key: os.environ/LEMONADE_API_KEY
```

Guidelines:

- `model_name` is the clean alias clients will use through LiteLLM.
- `model` must match the exact upstream Lemonade model ID.
- Most Lemonade-backed models should continue using `LEMONADE_API_BASE` and `LEMONADE_API_KEY`.
- Only use a separate `api_base` when the model is served by a different local service, such as the `phi4-npu` FLM server.

## Step 3: Commit and push from this repo clone

From this repo:

```bash
git status
git add litellm/config.yaml
git commit -m "Add LiteLLM alias for <model-name>"
git push
```

If you also updated docs or client config, include those files in the same commit.

## Step 4: Pull on the real Lemonade server

On the real Lemonade host:

```bash
cd /path/to/lemonade
git pull
```

## Step 5: Restart LiteLLM on the real server

On the real Lemonade host:

```bash
docker compose up -d --force-recreate litellm
```

If Compose configuration changed more broadly, restart the full stack instead:

```bash
docker compose up -d
```

## Step 6: Verify the alias is live

Check the LiteLLM model list:

```bash
curl -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  https://lemonade.tailc224aa.ts.net/litellm/v1/models
```

Confirm the new alias appears in the response.

## Optional: Add the model to Continue

If you want the new alias available in Continue, also update:

`~/.continue/config.yaml`

Example:

```yaml
  - name: "Gemma 4 E4B"
    provider: "openai"
    model: "gemma4-E4B"
    apiBase: "https://lemonade.tailc224aa.ts.net/litellm/v1"
    apiKey: "${LITELLM_MASTER_KEY}"
    useResponsesApi: false
    roles:
      - "chat"
```

## Quick Checklist

- New model confirmed in Lemonade `/v1/models`
- Alias added to `litellm/config.yaml`
- Changes committed and pushed
- Real Lemonade host pulled latest repo changes
- LiteLLM restarted on the real host
- Alias confirmed in `/litellm/v1/models`