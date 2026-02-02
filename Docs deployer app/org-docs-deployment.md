# Org-Wide Documentation Deployment

This guide explains how to set up automated daily documentation deployment for all repositories in your GitHub organization using a GitHub App + GitHub Actions.

## Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub App                               │
│  - Registered once in GitHub                                │
│  - Installed org-wide (auto-applies to new repos)           │
│  - Permissions: contents:read, actions:write                │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Scheduled GitHub Action                         │
│  - Runs daily (or on-demand)                                │
│  - Lives in the org's ".github" repo                        │
│  - Uses app token to trigger CI workflows                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Per-Repo CI Workflows                           │
│  - Each repo has its own ci.yml (from coding-workflow-      │
│    templates)                                                │
│  - Builds docs and deploys to NAS if configured             │
└─────────────────────────────────────────────────────────────┘
```

## Step 1: Create the GitHub App

### 1.1 Register the App

1. Go to **GitHub → Settings → Developer settings → GitHub Apps → New GitHub App**

   Direct link: https://github.com/organizations/MuTech-BV/settings/apps/new

2. Fill in the details:
   - **GitHub App name**: `MuTech Docs Deployer` (must be unique on GitHub)
   - **Homepage URL**: `https://github.com/MuTech-BV`
   - **Webhook**: Uncheck "Active" (we don't need webhooks)

3. Set **Permissions**:

   | Category | Permission | Access |
   |----------|------------|--------|
   | Repository | Contents | Read |
   | Repository | Actions | Write |
   | Repository | Metadata | Read (auto-selected) |

4. Under **Where can this GitHub App be installed?**:
   - Select "Only on this account"

5. Click **Create GitHub App**

### 1.2 Generate a Private Key

1. After creation, scroll to **Private keys**
2. Click **Generate a private key**
3. A `.pem` file will download - keep this safe!

### 1.3 Note the App ID

On the app's settings page, note the **App ID** (a number like `123456`).

### 1.4 Install the App

1. Go to **Install App** in the left sidebar
2. Click **Install** next to your organization
3. Choose **All repositories** (recommended) or select specific repos
4. Click **Install**

## Step 2: Configure the .github Repository

### 2.1 Clone the Repo

Your organization's `.github` repository already exists. Clone it if you haven't:

```bash
gh repo clone MuTech-BV/.github
cd .github
```

### 2.2 Add Secrets

Add the GitHub App credentials as repository secrets:

```bash
# App ID (as a variable, not secret - it's not sensitive)
gh variable set APP_ID --body "YOUR_APP_ID_HERE"

# Private key (as a secret)
gh secret set APP_PRIVATE_KEY < /path/to/your-app.private-key.pem
```

### 2.3 Create the Workflow

Create `.github/workflows/deploy-org-docs.yml`:

```yaml
name: Deploy Organization Docs

on:
  schedule:
    # Runs daily at 2:00 AM UTC
    - cron: '0 2 * * *'
  workflow_dispatch:
    inputs:
      repo:
        description: 'Specific repo to deploy (leave empty for all)'
        required: false
        type: string

permissions:
  contents: read

jobs:
  deploy-docs:
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ vars.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: MuTech-BV

      - name: Deploy docs for all repos
        if: ${{ !inputs.repo }}
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          echo "Fetching organization repositories..."

          # Get all repos in the org
          repos=$(gh repo list MuTech-BV --json name --limit 200 -q '.[].name')

          for repo in $repos; do
            echo "Checking $repo..."

            # Check if repo has mkdocs.yml
            if gh api "repos/MuTech-BV/$repo/contents/mkdocs.yml" --silent 2>/dev/null; then
              echo "  → Found mkdocs.yml, triggering docs deployment..."

              # Check if ci.yml workflow exists
              if gh api "repos/MuTech-BV/$repo/contents/.github/workflows/ci.yml" --silent 2>/dev/null; then
                # Trigger the CI workflow on main branch
                gh workflow run ci.yml --repo "MuTech-BV/$repo" --ref main || echo "  → Failed to trigger workflow"
                echo "  → Triggered CI workflow"
              else
                echo "  → No ci.yml workflow found, skipping"
              fi
            else
              echo "  → No mkdocs.yml found, skipping"
            fi

            # Small delay to avoid rate limiting
            sleep 1
          done

          echo "Done!"

      - name: Deploy docs for specific repo
        if: ${{ inputs.repo }}
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          REPO: ${{ inputs.repo }}
        run: |
          echo "Deploying docs for $REPO..."

          # Check if repo has mkdocs.yml
          if gh api "repos/MuTech-BV/$REPO/contents/mkdocs.yml" --silent 2>/dev/null; then
            gh workflow run ci.yml --repo "MuTech-BV/$REPO" --ref main
            echo "Triggered CI workflow for $REPO"
          else
            echo "Error: $REPO does not have mkdocs.yml"
            exit 1
          fi
```

### 2.4 Commit and Push

```bash
git add .github/workflows/deploy-org-docs.yml
git commit -m "Add org-wide docs deployment workflow"
git push
```

## Step 3: Verify Setup

### 3.1 Manual Test

Trigger the workflow manually:

```bash
gh workflow run deploy-org-docs.yml --repo MuTech-BV/.github
```

Or for a specific repo:

```bash
gh workflow run deploy-org-docs.yml --repo MuTech-BV/.github -f repo=python-package-template
```

### 3.2 Check Logs

```bash
gh run list --repo MuTech-BV/.github --workflow=deploy-org-docs.yml
gh run view --repo MuTech-BV/.github <RUN_ID> --log
```

## Step 4: Per-Repo Requirements

For a repo to be included in the automated deployment, it needs:

1. **`mkdocs.yml`** - Documentation configuration
2. **`.github/workflows/ci.yml`** - CI workflow with docs job (from coding-workflow-templates)
3. **Repository secrets** (for NAS deployment):
   - `NAS_HOST` - NAS hostname/IP
   - `NAS_RSYNC_PASSWORD` - rsync password
   - `TS_AUTH_KEY` - Tailscale auth key

## Troubleshooting

### "Resource not accessible by integration"

The GitHub App doesn't have the required permissions. Go to the app settings and ensure:
- Contents: Read
- Actions: Write

Then re-install the app on your organization.

### "Workflow not found"

The repo doesn't have a `.github/workflows/ci.yml` file. Ensure repos use the standard workflow from `coding-workflow-templates`.

### Rate Limiting

If you have many repos, you might hit rate limits. The workflow includes a 1-second delay between repos. Increase this if needed:

```yaml
sleep 2  # or higher
```

### Specific Repo Not Deploying

Check that the repo has:
1. `mkdocs.yml` in the root
2. `.github/workflows/ci.yml` exists
3. The GitHub App is installed on that repo

## Security Considerations

1. **Private Key**: Store securely, never commit to repos
2. **App Permissions**: Minimal permissions (contents:read, actions:write)
3. **Org-only Installation**: App can't be installed outside your org
4. **Audit Trail**: All actions are logged under the app's identity

## Maintenance

### Rotating the Private Key

1. Go to the GitHub App settings
2. Generate a new private key
3. Update the `APP_PRIVATE_KEY` secret in the .github repo
4. Delete the old key from GitHub App settings

### Updating Permissions

If you need additional permissions later:
1. Update the app's permissions in settings
2. Re-approve the new permissions in the org installation
