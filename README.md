# Slack PR Notifications

GitHub Actions workflows that keep your team informed about pull requests via Slack, with per-repo owner tagging for final approval accountability.

## What's Included

### Real-Time Notification (`slack-pr-notify.yml`)

Posts to Slack immediately when a PR is opened or marked ready for review. Tags the repo owner as the final approver.

- Triggers on `opened` (non-draft) and `ready_for_review` events
- Reads `repo-owners.yml` from the org `.github` repo to identify the repo owner
- Tags the owner directly via Slack `<@member_id>` with "you have final approval" message
- Falls back to `@channel` for repos without a configured owner
- Shows PR stats (additions, deletions, files changed)
- Validates webhook URL points to `slack.com` (defense-in-depth)
- Escapes mrkdwn special characters to prevent injection via crafted PR titles

### Daily Digest (`slack-pr-digest.yml`)

Runs daily at 9 AM CDT and posts a summary of all open PRs across the org, grouped by repo owner and bucketed by age:

- PRs grouped under their repo owner with "Final approver" header
- Unowned repos listed separately under "No owner assigned"
- Age bucketing within each owner section:
  - :red_circle: **3+ days** — stale, needs attention
  - :yellow_circle: **1-2 days** — aging
  - :green_circle: **< 1 day** — fresh
- Summary footer with total PRs, repos, and owners tagged
- Handles GitHub Search API pagination and the 1000-result cap gracefully

### Repo Ownership Config (`repo-owners.yml`)

A YAML config file in the org `.github` repo that maps repositories to their primary owners:

```yaml
owners:
  security-platform:
    github: dustin-saronic    # GitHub username
    slack: U04ABC12DEF        # Slack member ID for @-mentions
    name: Dustin              # Display name in messages
```

This config also drives the CODEOWNERS sync — owners are automatically added to their repo's CODEOWNERS file so GitHub branch protection requires their approval.

## Setup

See [docs/slack-notifications-setup.md](docs/slack-notifications-setup.md) for the full setup guide covering:

1. Creating a Slack Incoming Webhook
2. Creating GitHub Fine-Grained PATs
3. Adding org-level secrets (`SLACK_WEBHOOK_URL`, `ORG_READ_PAT`)
4. Configuring repo owners in `repo-owners.yml`
5. Registering as a required workflow (optional, for org-wide coverage)
6. Testing
7. PAT rotation schedule

## Required Secrets

| Secret | Used By | Purpose |
|--------|---------|---------|
| `SLACK_WEBHOOK_URL` | Both workflows | Slack incoming webhook URL |
| `ORG_READ_PAT` | Digest only | GitHub fine-grained PAT with read access to PRs/issues |

## Usage

### As an org-wide `.github` repo workflow

Copy the workflows into your org's `.github` repo under `.github/workflows/`. Register `slack-pr-notify.yml` as a required workflow to trigger on PRs across all repos. Place `repo-owners.yml` in the `.github` repo root.

### Per-repo

Copy the workflows into any repo's `.github/workflows/` directory and add the required secrets. The workflows will still read `repo-owners.yml` from the org `.github` repo.

## Security

- Webhook URL validated against `slack.com` domain
- All PR metadata escaped before inclusion in Slack messages
- Pagination URLs validated against `api.github.com` (SSRF defense)
- Secrets passed via environment variables, never interpolated into code
- Python `urllib` used instead of `curl` to avoid exposing secrets in process table
- YAML config parsed without external dependencies (regex-based, controlled format)

## License

MIT
