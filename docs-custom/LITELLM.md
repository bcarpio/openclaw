# LiteLLM Proxy Setup (AWS Bedrock)

LiteLLM sits between OpenClaw and AWS Bedrock, giving you a single OpenAI-compatible endpoint for all your models. This keeps AWS credentials out of OpenClaw and lets you add spend limits, logging, and model routing in one place.

## Architecture

```
OpenClaw (Docker) --> LiteLLM (Docker) --> AWS Bedrock (us-west-2)
  :18789               :4000               Claude Sonnet 4.6
                                           Claude Haiku 4.5
```

OpenClaw calls `http://host.docker.internal:4000` using the OpenAI completions API. LiteLLM translates that to Bedrock's API using your AWS credentials.

## Prerequisites

- AWS account with Bedrock model access enabled for Claude Sonnet 4.6 and Claude Haiku 4.5 in `us-west-2`
- AWS CLI configured with a profile (default: `admin`) that has Bedrock permissions
- Docker and Docker Compose

## Files

### `docker-compose.yml`

```yaml
services:
  litellm:
    image: docker.litellm.ai/berriai/litellm:main-stable
    ports:
      - "4000:4000"
    volumes:
      - ./litellm_config.yaml:/app/config.yaml
      - ~/.aws:/root/.aws:ro
    environment:
      - AWS_PROFILE=admin
      - LITELLM_MASTER_KEY
      - DATABASE_URL=postgresql://litellm:litellm@db:5432/litellm
    command: --config /app/config.yaml
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: litellm
      POSTGRES_PASSWORD: litellm
      POSTGRES_DB: litellm
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Key points:
- `~/.aws:/root/.aws:ro` — mounts your AWS credentials read-only into the container
- `AWS_PROFILE=admin` — uses the `admin` AWS CLI profile. Change to match yours.
- `LITELLM_MASTER_KEY` — set this in your environment before starting. This is the API key OpenClaw uses to authenticate with LiteLLM.
- Postgres stores usage tracking, spend logs, and rate limit state.

### `litellm_config.yaml`

```yaml
model_list:
  - model_name: claude-sonnet
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-6
      aws_region_name: us-west-2

  - model_name: claude-haiku
    litellm_params:
      model: bedrock/global.anthropic.claude-haiku-4-5-20251001-v1:0
      aws_region_name: us-west-2

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

The `model_name` values here (`claude-sonnet`, `claude-haiku`) are what you reference in OpenClaw's config as `litellm/claude-sonnet` and `litellm/claude-haiku`.

## Start

```bash
export LITELLM_MASTER_KEY="your-secret-key-here"
docker compose up -d
```

## Verify

```bash
# Health check
curl http://localhost:4000/health

# Test a model call
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "claude-haiku", "messages": [{"role": "user", "content": "hello"}]}'
```

## OpenClaw integration

In `~/.openclaw/openclaw.json`, the LiteLLM connection looks like:

```json
{
  "auth": {
    "profiles": {
      "litellm:default": {
        "provider": "litellm",
        "mode": "api_key"
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "litellm": {
        "baseUrl": "http://host.docker.internal:4000",
        "api": "openai-completions",
        "models": [
          {
            "id": "claude-sonnet",
            "name": "Claude Sonnet 4.6",
            "api": "openai-completions",
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          },
          {
            "id": "claude-haiku",
            "name": "Claude Haiku 4.5",
            "api": "openai-completions",
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "litellm/claude-haiku"
      },
      "models": {
        "litellm/claude-sonnet": {},
        "litellm/claude-haiku": {}
      }
    }
  }
}
```

Notes:
- `baseUrl` uses `host.docker.internal` because OpenClaw runs in Docker and LiteLLM runs on the host (or in a separate Docker network). The `extra_hosts` mapping in OpenClaw's `docker-compose.yml` makes this resolve on WSL2.
- `cost` is set to `0` because billing is handled by AWS Bedrock, not per-token through LiteLLM.
- Default model is `claude-haiku` to minimize token costs. Switch to `claude-sonnet` for complex tasks.

## Spend limits (optional)

LiteLLM supports budget and rate limiting via its admin UI at `http://localhost:4000/ui` (authenticate with your `LITELLM_MASTER_KEY`). You can set:

- Per-key daily/monthly spend caps
- Per-model rate limits (requests/min, tokens/min)
- Per-user budgets

This is your safety net against runaway agent costs — even if OpenClaw goes wild, LiteLLM will cut it off at your limit.

## Why LiteLLM instead of direct Bedrock?

- **Single endpoint** — OpenClaw only needs one URL, not per-model Bedrock configs
- **AWS credentials stay out of OpenClaw** — the container never sees your AWS keys
- **Spend controls** — budget caps and rate limits at the proxy layer
- **Model swapping** — change the backing model in LiteLLM config without touching OpenClaw
- **Logging** — Postgres stores every request for cost tracking and debugging
