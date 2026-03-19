# Slack PR Notifications — Setup Guide

One-time setup required before the workflows are active.

## 1. Create the Slack Incoming Webhook

1. Go to your Slack workspace → **Apps** → search **Incoming Webhooks** → **Add to Slack**
2. Choose the existing security team channel
3. Copy the Webhook URL (format: `https://hooks.slack.com/services/...`)

**Validate it works before saving:**
```bash
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"text": "test — slack-pr-notify setup check"}' \
  https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```
Expected response: `ok`. If you see anything else, the URL is wrong.

## 2. Create GitHub Fine-Grained PATs

Two PATs are needed — one read-only for the digest, one with write access for CODEOWNERS sync.

### 2a. `ORG_READ_PAT` (digest workflow)

1. Go to GitHub → **Settings** → **Developer settings** → **Fine-grained tokens** → **Generate new token**
2. Set:
   - **Token name:** `saronic-security-slack-digest`
   - **Resource owner:** `Saronic-Security`
   - **Expiration:** Maximum (1 year)
   - **Repository access:** All repositories
   - **Permissions:**
     - Pull requests → **Read-only**
     - Issues → **Read-only** *(required for `/search/issues` endpoint)*
3. Copy the token

### 2b. `ORG_WRITE_PAT` (CODEOWNERS sync workflow)

1. Same flow → **Generate new token**
2. Set:
   - **Token name:** `saronic-security-codeowners-sync`
   - **Resource owner:** `Saronic-Security`
   - **Expiration:** Maximum (1 year)
   - **Repository access:** All repositories
   - **Permissions:**
     - Contents → **Read and write** *(pushes CODEOWNERS to all repos)*
3. Copy the token

## 3. Add Org-Level Secrets

Go to `https://github.com/organizations/Saronic-Security/settings/secrets/actions` and add:

| Secret Name | Used by | Value |
|-------------|---------|-------|
| `SLACK_WEBHOOK_URL` | notify + digest | Slack webhook URL from step 1 |
| `ORG_READ_PAT` | digest | Fine-grained PAT from step 2a |
| `ORG_WRITE_PAT` | CODEOWNERS sync | Fine-grained PAT from step 2b |

**Important:** When adding each secret, set **Repository access** to **"All repositories"** (not just selected repos). If scoped too narrowly, workflows in other repos will get "secret not found" errors.

## 4. Register `slack-pr-notify.yml` as a Required Workflow

> **Note:** GitHub has two ways to require workflows depending on your plan:
> - **Classic (GHEC/older):** org Settings → Actions → Required workflows
> - **Rulesets (newer GHEC):** org Settings → Rules → Rulesets → New ruleset → add "Require workflows to pass" condition
>
> Use whichever is available in your org. The steps below are for classic Required Workflows.

1. Go to `https://github.com/organizations/Saronic-Security/settings/actions`
2. Scroll to **Required workflows** → click **Add workflow**
3. In the **Workflow** field, enter the full path: `Saronic-Security/.github/.github/workflows/slack-pr-notify.yml@main`
   > This looks like a double `.github` — that's correct. The repo is named `.github` and the file lives at `.github/workflows/` within it.
4. Set **Apply to:** All repositories
5. Save

After this, every PR opened in any Saronic-Security repo triggers the notification.
`slack-pr-digest.yml` runs automatically on its cron schedule — no additional setup needed.

## 5. Test It

**Test real-time notification:**
- Open a non-draft PR in any org repo
- Confirm Slack message appears in the team channel within ~30 seconds

**Test daily digest manually:**
- Go to the `.github` repo → **Actions** → **Slack Daily PR Digest** → **Run workflow**
- Confirm digest appears in Slack (or "No open PRs — skipping digest" logged if none exist)

**Test draft skipping:**
- Open a PR as draft — confirm no Slack message
- Convert it to ready for review — confirm message now appears

**Test CODEOWNERS sync:**
- Go to the `.github` repo → **Actions** → **Sync CODEOWNERS to All Org Repos** → **Run workflow**
- Verify it logs `Created: X repos` and `Unchanged: Y repos`

## 6. PAT Rotation

Fine-grained PATs expire after 1 year max. Rotate before expiry:

1. Create a new token with the same permissions as the expiring one (see step 2a or 2b)
2. Update the org secret at `github.com/organizations/Saronic-Security/settings/secrets/actions`
3. Run **Sync CODEOWNERS** and trigger a test PR to verify the new tokens work
4. Delete the old token from **Developer settings → Fine-grained tokens**
