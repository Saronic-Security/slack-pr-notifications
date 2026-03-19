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

Two PATs are needed — one read-only for notifications + digest, one with write access for CODEOWNERS sync.

### 2a. `ORG_READ_PAT` (notify + digest workflows)

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
| `ORG_READ_PAT` | notify + digest | Fine-grained PAT from step 2a |
| `ORG_WRITE_PAT` | CODEOWNERS sync | Fine-grained PAT from step 2b |

**Important:** When adding each secret, set **Repository access** to **"All repositories"** (not just selected repos). If scoped too narrowly, workflows in other repos will get "secret not found" errors.

## 4. Add People to `repo-owners.yml`

The `repo-owners.yml` file in the `.github` repo is a simple GitHub → Slack user map. Workflows **auto-detect** who owns each repo (top contributor by commits) and look up their Slack ID from this file.

You only need to add people once — it works across all repos automatically.

### How to find Slack member IDs

1. Open Slack → click on the person's name/profile picture
2. Click the **three dots** (...) menu → **Copy member ID**
3. Format: `U` followed by 10 alphanumeric characters (e.g., `U04ABC12DEF`)

### Example config

```yaml
users:
  dylan-saronic:
    slack: U04ABC12DEF
    name: Dylan

  dustin-saronic:
    slack: U04XYZ78GHI
    name: Dustin

  saronic-brandyn:
    slack: U07QRS45JKL
    name: Brandyn
```

### Adding a new person

1. Edit `repo-owners.yml` in the `Saronic-Security/.github` repo
2. Add a block under `users:` with their GitHub username, Slack member ID, and display name
3. Commit to `main` — that's it, they'll be tagged automatically on any repo where they're the top contributor

### How ownership is determined

- Workflows call `GET /repos/{org}/{repo}/contributors?per_page=1` to find the top contributor
- The top contributor's GitHub username is looked up in `repo-owners.yml` for their Slack ID
- If the person isn't in the file, or the repo has no contributors, falls back to `@channel`
- CODEOWNERS sync also uses top contributor to add individual owners to branch protection

## 5. Register `slack-pr-notify.yml` as a Required Workflow

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

## 6. Test It

**Test real-time notification:**
- Open a non-draft PR in any org repo
- Confirm Slack message appears with the repo's top contributor @-mentioned
- If the contributor isn't in `repo-owners.yml`, verify it falls back to `@channel`

**Test daily digest manually:**
- Go to the `.github` repo → **Actions** → **Slack Daily PR Digest** → **Run workflow**
- Confirm digest appears with PRs tagged with their repo owner

**Test draft skipping:**
- Open a PR as draft — confirm no Slack message
- Convert it to ready for review — confirm message now appears

**Test CODEOWNERS sync:**
- Go to the `.github` repo → **Actions** → **Sync CODEOWNERS to All Org Repos** → **Run workflow**
- Verify it logs each repo with its detected owner
- Check a repo: its CODEOWNERS should include `@top-contributor @Saronic-Security/security-engineering-approvers`

## 7. PAT Rotation

Fine-grained PATs expire after 1 year max. Rotate before expiry:

1. Create a new token with the same permissions as the expiring one (see step 2a or 2b)
2. Update the org secret at `github.com/organizations/Saronic-Security/settings/secrets/actions`
3. Run **Sync CODEOWNERS** and trigger a test PR to verify the new tokens work
4. Delete the old token from **Developer settings → Fine-grained tokens**
