# Custom Docker Setup (WSL2)

Fixes and workarounds for running OpenClaw in Docker on WSL2. The upstream `docker-setup.sh` has several issues that prevent a working install out of the box.

## Known upstream issues addressed

- DNS resolution fails inside containers ([#14593](https://github.com/openclaw/openclaw/issues/14593))
- Skill installs fail — no brew, old Go via apt, npm EACCES ([#3987](https://github.com/openclaw/openclaw/issues/3987), [#12622](https://github.com/openclaw/openclaw/issues/12622), [#4130](https://github.com/openclaw/openclaw/issues/4130))
- Gateway crash-loops during onboard before `allowedOrigins` is set
- `host.docker.internal` doesn't resolve on WSL2
- Extensions (Telegram, etc.) fail with "Cannot find module" — Docker image ships `dist/` but extensions import from `src/`
- Skill binaries and tool configs lost on container recreate

## What's changed

### `docker-compose.yml`

- **DNS** — explicit `8.8.8.8` / `1.1.1.1` so containers can resolve external hosts
- **`host.docker.internal`** — `extra_hosts` mapping so the gateway can reach host services (LiteLLM, etc.)
- **`GOG_KEYRING_PASSWORD`** — env var for gog's file-based keyring in non-interactive contexts
- **`~/.claude` mount** — so the `coding-agent` skill can use your Claude Code credentials
- **`openclaw-home` volume** — persists `/home/node` (gog config, npm globals, go binaries, etc.)
- **`linuxbrew` volume** — persists `/home/linuxbrew` so brew-installed skills survive recreates

### `Dockerfile.local`

Extends `openclaw:local` with:

- **Source tree (`src/`)** — extensions import from `../../../src/` via jiti but the Docker image only ships `dist/`. Fixes Telegram and other extension loading failures
- **Claude Code CLI** (`@anthropic-ai/claude-code`) for the `coding-agent` skill
- **Homebrew (Linuxbrew)** so brew-only skills install without manual workarounds (~500MB)
- **Go 1.24** from the official tarball (Debian apt ships 1.19 which is too old)
- **User-writable npm global prefix** so `npm install -g` works as the non-root `node` user

## Setup

### 1. Run upstream setup

Skip skill installs during the wizard — they'll fail due to the gateway crash-loop.

```bash
export OPENCLAW_DOCKER_APT_PACKAGES=gh
export OPENCLAW_HOME_VOLUME=openclaw-home
./docker-setup.sh
```

### 2. Build custom image layer

```bash
docker build -t openclaw:local -f Dockerfile.local .
```

### 3. Recreate gateway with custom image

Always use `--force-recreate`, not `restart`, when compose file changes:

```bash
docker compose up -d --force-recreate openclaw-gateway
```

### 4. Set compaction model to Haiku

OpenClaw summarizes old conversation history when sessions get long. By default it uses the primary model. Force it to use Haiku so compaction doesn't burn Sonnet tokens:

```bash
docker compose run --rm openclaw-cli config set agents.defaults.compaction.model "litellm/claude-haiku"
```

## Configure `openclaw.json`

After setup, edit `~/.openclaw/openclaw.json`. The example below routes through LiteLLM (AWS Bedrock), uses Haiku as the default model, and includes Telegram.

Replace placeholder values with your own:

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
        "primary": "litellm/claude-haiku"
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

Placeholders to replace:
- `<YOUR_GEMINI_API_KEY>` — Gemini API key for web search
- `<YOUR_TELEGRAM_BOT_TOKEN>` — from @BotFather
- `<YOUR_GATEWAY_TOKEN>` — printed by `docker-setup.sh` (also in `.env`)

The `gateway.controlUi.allowedOrigins` must be set manually — `docker-setup.sh` fails to set it due to the gateway crash-loop during onboard.

## Control UI pairing

```bash
# Get the dashboard URL
docker compose run --rm openclaw-cli dashboard --no-open

# Open the URL in your browser — you'll see "pairing required"

# List pending device requests
docker compose exec openclaw-gateway node dist/index.js devices list

# Approve your browser
docker compose exec openclaw-gateway node dist/index.js devices approve <requestId>
```

## Telegram pairing

When you first message your bot, it will show a pairing code. Approve it:

```bash
docker compose run --rm openclaw-cli pairing approve telegram <PAIRING_CODE>
```

## Install skills

Skip skills during onboard. Install them after setup via the web UI at `http://127.0.0.1:18789` (Skills page), or from the CLI:

```bash
# Install via the UI (recommended — just click install)

# Or manually via CLI:
docker compose exec openclaw-gateway brew install steipete/tap/gogcli
docker compose run --rm --entrypoint sh openclaw-cli -c "npm install -g @xdevplatform/xurl clawhub mcporter"
docker compose run --rm --entrypoint sh openclaw-cli -c "go install github.com/Hyaxia/blogwatcher/cmd/blogwatcher@latest"
```

After installing skills, restart the gateway so the binary cache refreshes:

```bash
docker compose restart openclaw-gateway
```

## gog (Google Workspace CLI) setup

gog requires OAuth credentials from Google Cloud Console.

### 1. Google Cloud setup

1. Create a project at [console.cloud.google.com](https://console.cloud.google.com/)
2. Go to **APIs & Services > Library** and enable: Gmail API, Google Calendar API, Google Drive API, People API, Google Sheets API, Google Docs API
3. Go to **APIs & Services > OAuth consent screen** — choose **Internal** if you have Google Workspace, otherwise External
4. Add read-only scopes only (to prevent the agent from writing/sending/deleting):
   - `https://www.googleapis.com/auth/gmail.readonly`
   - `https://www.googleapis.com/auth/calendar.readonly`
   - `https://www.googleapis.com/auth/drive.readonly`
   - `https://www.googleapis.com/auth/contacts.readonly`
   - `https://www.googleapis.com/auth/spreadsheets.readonly`
   - `https://www.googleapis.com/auth/documents.readonly`
5. Go to **APIs & Services > Credentials** (or Clients > Create Client) — create a **Desktop app** OAuth client
6. Download the JSON file

### 2. Configure gog in the container

```bash
# Copy credentials into the container
docker cp ~/path/to/client_secret.json openclaw-openclaw-gateway-1:/tmp/client_secret.json

# Set file-based keyring (avoids passphrase prompts)
docker compose exec -e GOG_KEYRING_PASSWORD=openclaw openclaw-gateway gog auth keyring file

# Store credentials
docker compose exec -e GOG_KEYRING_PASSWORD=openclaw openclaw-gateway gog auth credentials set /tmp/client_secret.json

# Auth with --manual flag (callback port isn't exposed from container)
docker compose exec -e GOG_KEYRING_PASSWORD=openclaw openclaw-gateway gog auth add you@domain.com --services gmail,calendar,drive,contacts,docs,sheets --manual
```

The `--manual` flag prints a URL. Open it in your browser, authorize, then copy the redirect URL from your browser's address bar (it won't load — that's expected) and paste it back into the terminal.

### 3. Verify

```bash
docker compose exec -e GOG_KEYRING_PASSWORD=openclaw openclaw-gateway gog auth status
```

gog config persists in the `openclaw-home` volume at `/home/node/.config/gogcli/`.

## Important: `restart` vs `recreate`

- `docker compose restart` — restarts the container with the **same** config. Does NOT pick up changes to `docker-compose.yml`.
- `docker compose up -d --force-recreate` — recreates the container with **new** config. Use this after editing `docker-compose.yml`.

## Upstream issues

- [#14593](https://github.com/openclaw/openclaw/issues/14593) — Skill install fails in Docker: brew not installed
- [#3987](https://github.com/openclaw/openclaw/issues/3987) — Feature: Include brew install in Dockerfile
- [#12622](https://github.com/openclaw/openclaw/issues/12622) — Docker unable to install skills
- [#4130](https://github.com/openclaw/openclaw/issues/4130) — Skill install errors during docker setup
