# Slack PR Notifications

GitHub Actions workflows that keep your team informed about pull requests via Slack, with automatic repo owner tagging.

## How It Works

1. Workflows auto-detect who owns each repo (top contributor by commits)
2. Look up that person's Slack ID from `repo-owners.yml` (a simple GitHub → Slack user map in the `.github` repo)
3. Tag them directly in Slack notifications

No per-repo config needed — just add people to the user map once.

## What's Included

### Real-Time Notification (`slack-pr-notify.yml`)

Posts to Slack when a PR is opened or marked ready for review:

- Tags the repo owner (top contributor) directly via `<@member_id>`
- Shows PR title, body summary, diff stats, and timestamp
- Falls back to `@channel` if the owner isn't in the user map
- Skips drafts automatically

### Daily Digest (`slack-pr-digest.yml`)

Runs daily at 9 AM CDT with a summary of all open PRs:

- Headline: total PRs, how many stale/aging/fresh
- Stale PRs (3+ days) listed first with repo owner tagged
- Each PR shows age, author, and who needs to approve

### User Map (`repo-owners.yml`)

Lives in the org `.github` repo. Simple format:

```yaml
users:
  dustin-saronic:
    slack: U04ABC12DEF
    name: Dustin
  dylan-saronic:
    slack: U04XYZ78GHI
    name: Dylan
```

## Required Secrets

Set these at `https://github.com/organizations/Saronic-Security/settings/secrets/actions`:

| Secret | Used By | Purpose |
|--------|---------|---------|
| `SLACK_WEBHOOK_URL` | Both workflows | Slack incoming webhook URL |
| `ORG_READ_PAT` | Both workflows | GitHub PAT with read access to PRs/issues |

## Setup

See [docs/slack-notifications-setup.md](docs/slack-notifications-setup.md) for the full guide.

## Security

- Webhook URL validated against `slack.com` domain
- All PR metadata escaped before inclusion in Slack messages
- Pagination URLs validated against `api.github.com`
- Secrets passed via environment variables, never interpolated into code

## License

MIT
