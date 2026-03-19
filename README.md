# Slack PR Notifications

GitHub Actions workflows that keep your team informed about pull requests via Slack.

## What's Included

### Real-Time Notification (`slack-pr-notify.yml`)

Posts to Slack immediately when a PR is opened or marked ready for review. Skips drafts automatically.

- Triggers on `opened` (non-draft) and `ready_for_review` events
- Uses `@channel` to alert all reviewers
- Validates webhook URL points to `slack.com` (defense-in-depth)
- Escapes mrkdwn special characters to prevent injection via crafted PR titles

### Daily Digest (`slack-pr-digest.yml`)

Runs daily at 9 AM CDT and posts a summary of all open PRs across the org, bucketed by age:

- :red_circle: **3+ days** — stale, needs attention
- :yellow_circle: **1-2 days** — aging
- :green_circle: **< 1 day** — fresh

Handles GitHub Search API pagination and the 1000-result cap gracefully.

## Setup

See [docs/slack-notifications-setup.md](docs/slack-notifications-setup.md) for the full setup guide covering:

1. Creating a Slack Incoming Webhook
2. Creating GitHub Fine-Grained PATs
3. Adding org-level secrets (`SLACK_WEBHOOK_URL`, `ORG_READ_PAT`)
4. Registering as a required workflow (optional, for org-wide coverage)
5. Testing
6. PAT rotation schedule

## Required Secrets

| Secret | Used By | Purpose |
|--------|---------|---------|
| `SLACK_WEBHOOK_URL` | Both workflows | Slack incoming webhook URL |
| `ORG_READ_PAT` | Digest only | GitHub fine-grained PAT with read access to PRs/issues |

## Usage

### As an org-wide `.github` repo workflow

Copy the workflows into your org's `.github` repo under `.github/workflows/`. Register `slack-pr-notify.yml` as a required workflow to trigger on PRs across all repos.

### Per-repo

Copy the workflows into any repo's `.github/workflows/` directory and add the required secrets.

## Security

- Webhook URL validated against `slack.com` domain
- All PR metadata escaped before inclusion in Slack messages
- Pagination URLs validated against `api.github.com` (SSRF defense)
- Secrets passed via environment variables, never interpolated into code
- Python `urllib` used instead of `curl` to avoid exposing secrets in process table

## License

MIT
