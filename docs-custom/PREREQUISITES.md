# Prerequisites

## System requirements

- **WSL2** on Windows 10/11
- **Docker Desktop** with WSL2 backend enabled
- **Git** configured with SSH or HTTPS auth

## AWS credentials

LiteLLM mounts `~/.aws` read-only into its container. Set up your credentials carefully:

### Create a dedicated AWS user for Bedrock

Do not use your admin credentials as the default profile. Create a dedicated IAM user with only the permissions needed for Bedrock inference:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*"
    }
  ]
}
```

### Configure AWS CLI profiles

```ini
# ~/.aws/credentials

[default]
aws_access_key_id = <BEDROCK_ONLY_USER_KEY>
aws_secret_access_key = <BEDROCK_ONLY_USER_SECRET>

[admin]
aws_access_key_id = <YOUR_ADMIN_KEY>
aws_secret_access_key = <YOUR_ADMIN_SECRET>
```

The `default` profile should be the restricted Bedrock user. Your admin profile should be explicitly named so nothing uses it accidentally.

In `docker-compose.yml`, LiteLLM is configured with `AWS_PROFILE=admin` — change this to `default` (or whatever you named your Bedrock-only profile) so the container never has admin credentials in scope:

```yaml
environment:
  - AWS_PROFILE=default
```

### Why this matters

Since `~/.aws` is mounted into the LiteLLM container, any process inside that container could read your credentials. If the default profile has `AdministratorAccess`, a compromised container could access your entire AWS account. Restricting the default profile to Bedrock-only limits blast radius.

## Google Cloud (for gog skill)

Only needed if you want the gog (Google Workspace CLI) skill:

- Google Cloud project with APIs enabled (Gmail, Calendar, Drive, People, Sheets, Docs)
- OAuth consent screen configured (Internal if you have Google Workspace)
- Desktop app OAuth credentials (client_secret.json)

See the gog setup section in [README.custom.md](../README.custom.md#gog-google-workspace-cli-setup).

## Telegram (optional)

- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- Your Telegram user ID (the bot will show it on first message)

## GitHub (for gh-issues / coding-agent skills)

- A dedicated GitHub user with a fine-grained PAT (see [GITHUB-SAFETY.md](GITHUB-SAFETY.md))
- Branch protection rules on shared repos

## Keeping your fork in sync with upstream

Add the upstream remote:

```bash
git remote add upstream https://github.com/openclaw/openclaw.git
```

To pull upstream changes:

```bash
git fetch upstream
git rebase upstream/main
```

Conflicts will likely occur in `docker-compose.yml` since we've modified it. Resolve by keeping our additions (DNS, extra_hosts, volumes, env vars) and accepting their changes elsewhere.

After rebasing, rebuild and recreate:

```bash
docker build -t openclaw:local -f Dockerfile .
docker build -t openclaw:local -f Dockerfile.local .
docker compose up -d --force-recreate openclaw-gateway
```

Check if any of our workarounds are no longer needed — upstream may fix the DNS, extension import, or skill install issues over time. Remove fixed workarounds to reduce maintenance burden.
