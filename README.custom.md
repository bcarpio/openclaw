# Custom Docker Setup

Workarounds for running OpenClaw in Docker on WSL2. Addresses known issues with the upstream `docker-setup.sh`:

- DNS resolution fails inside containers ([#14593](https://github.com/openclaw/openclaw/issues/14593))
- Skill installs fail — brew not available, Go too old via apt, npm permissions ([#3987](https://github.com/openclaw/openclaw/issues/3987))
- Gateway crash-loops during onboard due to `allowedOrigins` chicken-and-egg

## What's changed

### `docker-compose.yml`

Added explicit DNS servers to `openclaw-gateway` so the CLI container (which shares its network namespace) can resolve external hosts:

```yaml
dns:
  - 8.8.8.8
  - 1.1.1.1
```

### `Dockerfile.local`

Extends `openclaw:local` with:

- **Homebrew (Linuxbrew)** so brew-only skills install without manual workarounds
- **Go 1.24** from the official tarball (Debian apt ships 1.19 which is too old for most Go-based skills)
- **User-writable npm global prefix** so `npm install -g` works as the non-root `node` user

## Usage

```bash
# 1. Build the base image first
docker build -t openclaw:local -f Dockerfile .

# 2. Build the custom layer on top, tagged as openclaw:local
# docker-setup.sh only skips pulling when the image is named "openclaw:local"
docker build -t openclaw:local -f Dockerfile.local .

# 3. Run setup
export OPENCLAW_DOCKER_APT_PACKAGES=gh
export OPENCLAW_HOME_VOLUME=openclaw-home
./docker-setup.sh
```

## Post-setup configuration

After `docker-setup.sh` completes, replace `~/.openclaw/openclaw.json` with a working config. The example below includes LiteLLM proxy configuration (for AWS Bedrock), `allowedOrigins` for the Control UI, and Telegram.

Replace tokens, API keys, and bot tokens with your own values:

```json
{
  "wizard": {
    "lastRunAt": "2026-03-14T20:18:08.251Z",
    "lastRunVersion": "2026.3.14",
    "lastRunCommand": "onboard",
    "lastRunMode": "local"
  },
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
        "primary": "litellm/claude-sonnet"
      },
      "models": {
        "litellm/claude-sonnet": {},
        "litellm/claude-haiku": {}
      },
      "workspace": "/home/node/.openclaw/workspace"
    }
  },
  "tools": {
    "profile": "coding",
    "web": {
      "search": {
        "enabled": true,
        "provider": "gemini",
        "gemini": {
          "apiKey": "<YOUR_GEMINI_API_KEY>"
        }
      }
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "session": {
    "dmScope": "per-channel-peer"
  },
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "boot-md": { "enabled": true },
        "command-logger": { "enabled": true },
        "session-memory": { "enabled": true }
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",
      "botToken": "<YOUR_TELEGRAM_BOT_TOKEN>",
      "groupPolicy": "allowlist",
      "streaming": "partial"
    }
  },
  "gateway": {
    "controlUi": {
      "allowedOrigins": ["http://localhost:18789", "http://127.0.0.1:18789"]
    },
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "<YOUR_GATEWAY_TOKEN>"
    },
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    },
    "nodes": {
      "denyCommands": [
        "camera.snap",
        "camera.clip",
        "screen.record",
        "contacts.add",
        "calendar.add",
        "reminders.add",
        "sms.send"
      ]
    }
  },
  "plugins": {
    "entries": {
      "telegram": { "enabled": true }
    }
  }
}
```

Key things to replace:
- `<YOUR_GEMINI_API_KEY>` — Gemini API key for web search
- `<YOUR_TELEGRAM_BOT_TOKEN>` — Telegram bot token from @BotFather
- `<YOUR_GATEWAY_TOKEN>` — printed by `docker-setup.sh` during setup (also in `.env`)

The `gateway.controlUi.allowedOrigins` must be set manually — `docker-setup.sh` fails to set it due to the gateway crash-loop during onboard.

## Control UI pairing

After updating `openclaw.json`, restart the gateway and open the dashboard:

```bash
docker compose restart openclaw-gateway
docker compose run --rm openclaw-cli dashboard --no-open
```

Copy the URL it prints and open it in your browser. You'll see "pairing required". To approve:

```bash
# List pending device requests
docker compose exec openclaw-gateway node dist/index.js devices list

# Approve your browser using the requestId from the list
docker compose exec openclaw-gateway node dist/index.js devices approve <requestId>
```

Refresh the browser and you're in.

## Post-setup skill install

The gateway crash-loops during onboard (before `allowedOrigins` is set), so skill installs may fail during the wizard. Skip them and install after setup completes:

```bash
docker compose run --rm openclaw-cli onboard-skills
```

Or install individual tools directly:

```bash
# npm-based skills
docker compose run --rm --entrypoint sh openclaw-cli -c "npm install -g @xdevplatform/xurl clawhub mcporter"

# Go-based skills
docker compose run --rm --entrypoint sh openclaw-cli -c "go install github.com/Hyaxia/blogwatcher/cmd/blogwatcher@latest"
```

## Notes

- `OPENCLAW_HOME_VOLUME` persists `/home/node` so installed binaries survive container restarts.
- Homebrew is included in the image so all skill install methods (brew, npm, go) work out of the box. Adds ~500MB to the image.

## Upstream issues

- [#14593](https://github.com/openclaw/openclaw/issues/14593) — Skill install fails in Docker: brew not installed
- [#3987](https://github.com/openclaw/openclaw/issues/3987) — Feature: Include brew install in Dockerfile
- [#12622](https://github.com/openclaw/openclaw/issues/12622) — Docker unable to install skills
- [#4130](https://github.com/openclaw/openclaw/issues/4130) — Skill install errors during docker setup
