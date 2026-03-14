# Backup & Cost Controls

## Docker volumes

This setup uses three named Docker volumes that hold all persistent state:

| Volume | Contents | What you lose if deleted |
|--------|----------|------------------------|
| `openclaw-home` | `/home/node` — gog config/tokens, npm globals, go binaries, shell history | Tool auth, installed skill binaries |
| `openclaw_linuxbrew` | `/home/linuxbrew` — Homebrew and brew-installed skills | All brew-installed tools (gog, etc.) |
| `pgdata` | LiteLLM Postgres — usage logs, spend tracking, rate limit state | Cost history and budget tracking |

Host bind mounts (not volumes — these are on your filesystem):
- `~/.openclaw/` — OpenClaw config, agent sessions, skill settings
- `~/.openclaw/workspace/` — agent workspace files
- `~/.aws/` — AWS credentials (read-only mount)
- `~/.claude/` — Claude Code credentials

## Backup

### Quick backup (volumes)

```bash
# List volumes
docker volume ls | grep -E "openclaw|pgdata|linuxbrew"

# Backup a volume to a tar file
docker run --rm -v openclaw-home:/data -v $(pwd):/backup alpine \
  tar czf /backup/openclaw-home-backup.tar.gz -C /data .

docker run --rm -v openclaw_linuxbrew:/data -v $(pwd):/backup alpine \
  tar czf /backup/linuxbrew-backup.tar.gz -C /data .

docker run --rm -v openclaw_pgdata:/data -v $(pwd):/backup alpine \
  tar czf /backup/pgdata-backup.tar.gz -C /data .
```

### Quick backup (host config)

```bash
cp -r ~/.openclaw ~/.openclaw-backup-$(date +%Y%m%d)
```

### Restore a volume

```bash
docker volume create openclaw-home
docker run --rm -v openclaw-home:/data -v $(pwd):/backup alpine \
  tar xzf /backup/openclaw-home-backup.tar.gz -C /data
```

## Cost controls via LiteLLM

LiteLLM provides a web UI for spend tracking and budget management at:

```
http://localhost:4000/ui
```

Authenticate with your `LITELLM_MASTER_KEY`.

### What you can set

- **Per-key spend limits** — daily or monthly caps per API key. If OpenClaw hits the limit, LiteLLM returns 429 and the agent stops.
- **Per-model rate limits** — requests/minute and tokens/minute per model. Prevents runaway loops.
- **Per-user budgets** — if you add more users or agents later, each gets their own budget.

### Recommended setup

1. Create a dedicated API key for OpenClaw in the LiteLLM UI
2. Set a daily spend cap (e.g. $5/day) on that key
3. Set a lower rate limit on Sonnet than Haiku (Sonnet is ~30x more expensive)
4. Monitor the usage dashboard weekly

### Cost comparison

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|----------------------|
| Claude Haiku 4.5 | $0.80 | $4.00 |
| Claude Sonnet 4.6 | $3.00 | $15.00 |

By defaulting to Haiku for Telegram and routine chat, you keep daily costs minimal. Sonnet is available in the OpenClaw web UI when you need it for complex tasks — just switch models in the chat interface.

### Context growth and compaction costs

As a chat session grows, OpenClaw sends more tokens per request — the full conversation history is included with each call. This means input costs grow with every message.

OpenClaw handles this with **compaction** (see upstream docs: `docs/concepts/compaction.md`). When the session approaches the model's context window, it automatically summarizes older messages and keeps only recent turns verbatim. Your config uses `compaction.mode: "safeguard"` which structures summaries with decisions, TODOs, constraints, and identifiers.

**Cost implications:**
- The compaction summarization itself is an LLM call that costs tokens
- Before compaction triggers, every message includes the growing history as input
- Long Telegram conversations can quietly accumulate input costs before compaction kicks in
- The compaction call counts against your virtual key budget

**Tips:**
- Start new sessions (`/new`) periodically instead of running one endless conversation
- Consider setting `compaction.model` to `litellm/claude-haiku` so compaction summaries use the cheap model even if you switched to Sonnet mid-conversation
- Monitor per-request token counts in the LiteLLM logs to spot cost spikes

### Budget exceeded behavior

When your virtual key's budget is exceeded, LiteLLM returns an error like:

```
Budget has been exceeded! Current cost: 0.21, Max budget: 0.0
```

**Important:** The OpenClaw chat interface does not show this error to the user — it just silently fails to respond. Check the LiteLLM logs at `http://localhost:4000/ui` (Logs page) if the bot stops responding.

### Why this matters

AI agents can generate unbounded API calls — especially with tool loops, retries, and multi-step reasoning. Without spend limits, a single runaway session could burn through hundreds of dollars. LiteLLM is your circuit breaker.
