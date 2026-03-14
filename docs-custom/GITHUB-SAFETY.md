# GitHub Second-User + Branch Protection

When OpenClaw's `coding-agent` or `gh-issues` skill has write access to your repos, you want guardrails so an AI agent can't push directly to main, delete branches, or force-push over your work.

The pattern: create a dedicated GitHub user with limited permissions, then use branch protection rules to force all changes through PRs with human approval.

## 1. Create a dedicated GitHub user

Create a new GitHub account separate from your main account. Use a random suffix to avoid name squatting concerns:

```bash
# Generate a random suffix
openssl rand -hex 12
```

Example username: `openclaw-3d879df289d4522931c147a4`

This user is the only identity OpenClaw ever authenticates as. Your main account never touches OpenClaw.

## 2. Create a Personal Access Token (PAT)

On the dedicated user, create a fine-grained PAT with minimal permissions:

**Read-only access:**
- Dependabot alerts
- Administration
- Attestations API
- Deployments
- Issues
- Merge queues
- Metadata
- Pull requests
- Security events

**Read and Write access:**
- Code (needed to push branches and create PRs)

That's it. No access to settings, secrets, actions, webhooks, or anything else. The agent can read issues and push code to branches, but can't modify repo settings or access secrets.

## 3. Share repos with the dedicated user

From your main account or org, invite the dedicated user as a collaborator on the repos you want OpenClaw to access. Only share what's needed — don't give org-wide access.

## 4. Set up branch protection rules

On each shared repo, create a ruleset for the default branch (e.g. `main`) with these settings:

### Restrictions

- **Restrict creations** — Only allow users with bypass permission to create matching refs
- **Restrict updates** — Only allow users with bypass permission to update matching refs
- **Restrict deletions** — Only allow users with bypass permissions to delete matching refs

### Merge requirements

- **Require linear history** — Prevent merge commits from being pushed to matching refs
- **Require a pull request before merging** — All commits must go through a PR
- **Required approvals: 1** — A human (you, from your main account) must approve
- **Require conversation resolution before merging** — All review comments must be resolved

### What this means in practice

The dedicated user (OpenClaw) can:
- Read issues, PRs, code
- Create branches
- Push commits to non-protected branches
- Open pull requests

The dedicated user **cannot**:
- Push directly to main
- Merge PRs without your approval
- Delete branches matching the protection rules
- Force-push to protected branches
- Bypass any of the above

Every code change the agent makes requires your explicit review and approval before it lands on main.

## 5. Configure the PAT in OpenClaw

Store the PAT in OpenClaw's skill config. In `~/.openclaw/openclaw.json`:

```json
{
  "skills": {
    "entries": {
      "gh-issues": {
        "apiKey": "<DEDICATED_USER_PAT>"
      },
      "github": {
        "enabled": true
      }
    }
  }
}
```

Or set it via the Skills page in the OpenClaw web UI.

## Why this matters

Without these guardrails, an AI agent with a PAT that has write access to your repos could:
- Push directly to main (breaking builds, deploying bad code)
- Delete branches (losing in-progress work)
- Force-push (rewriting history, destroying commits)
- Merge its own PRs (no human review)

The second-user pattern ensures the agent operates in a sandbox within GitHub itself. Even if the agent hallucinates or goes off-script, the worst it can do is create a bad PR that you decline.
