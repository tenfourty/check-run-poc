# GitGuardian Developer Setup Verification - POC

A proof-of-concept GitHub Action that verifies developers have GitGuardian configured on their development machines by checking for active personal access tokens.

## Overview

This POC implements a GitHub Actions workflow that:
1. **Scans commits for secrets** using the official GitGuardian ggshield-action
2. **Verifies developers have GitGuardian PAT tokens** by querying the GitGuardian API
3. **Blocks PRs** if either check fails (configurable)
4. **Provides clear instructions** for setting up GitGuardian if missing

## How It Works

### Two-Job Workflow

**Job 1: GitGuardian Secret Scan**
- Runs the official `GitGuardian/ggshield-action@v1`
- Scans all commits in the PR for exposed secrets
- Fails if secrets are detected

**Job 2: Developer Setup Verification**
- Runs after the secret scan (regardless of its outcome)
- Retrieves PR author's GitHub username and email
- Queries GitGuardian API to check if the developer has an active PAT token
- Reports success/failure with actionable instructions
- Can be bypassed with admin override label

### Verification Logic

1. Check if PR has `skip-gg-verification` label (admin override)
2. Get PR author information from GitHub
3. Call GitGuardian API `GET /v1/members` to find the workspace member
4. Call GitGuardian API `GET /v1/tokens` to check for active tokens
5. Report results as a GitHub check status

## Setup Instructions

### Prerequisites

- GitHub repository with Actions enabled
- GitGuardian workspace with API access
- Service account or Manager PAT with these scopes:
  - `scan` (for ggshield-action)
  - `members:read` (to list workspace members)
  - `api_tokens:read` (to view token status)

### Installation

1. **Copy the workflow file to your repository:**
   ```bash
   mkdir -p .github/workflows
   cp .github/workflows/gitguardian-check.yml .github/workflows/
   ```

2. **Add required secrets to your GitHub repository:**

   Go to: `Settings > Secrets and variables > Actions > New repository secret`

   Add these secrets:
   - `GITGUARDIAN_API_KEY`: Personal access token with `scan` scope (for secret scanning)
   - `GITGUARDIAN_SERVICE_TOKEN`: Service account token with `members:read` and `api_tokens:read` scopes

   **To create these tokens:**
   - Sign in to GitGuardian dashboard: https://dashboard.gitguardian.com
   - Navigate to **Settings > API** > **Personal Access Tokens** (or Service Accounts)
   - Create new tokens with appropriate scopes

3. **Configure branch protection (optional but recommended):**

   Go to: `Settings > Branches > Add rule` (or edit existing rule)

   - Add branch name pattern (e.g., `main`)
   - Enable "Require status checks to pass before merging"
   - Select these status checks:
     - `GitGuardian Secret Scan`
     - `Verify Developer GitGuardian Setup`

4. **Create the override label (optional):**
   ```bash
   gh label create skip-gg-verification \
     --description "Skip GitGuardian developer setup verification" \
     --color "FBCA04"
   ```

   Or via GitHub UI: `Issues > Labels > New label`

### Configuration Options

Edit `.github/workflows/gitguardian-check.yml` to customize:

**Trigger events:**
```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]  # Customize these events
  workflow_dispatch:  # Allow manual runs
```

**Override label name:**
Change `skip-gg-verification` to your preferred label name in the workflow file.

**API endpoint:**
If using GitGuardian on-premises, update the API URL:
```bash
https://api.gitguardian.com/v1/members  # Change to your instance
```

## Usage

### For Developers

When you open a PR, two checks will run:

1. **GitGuardian Secret Scan** - Scans your commits for secrets
2. **Verify Developer GitGuardian Setup** - Checks if you have GitGuardian configured

**If verification fails**, follow these steps:

1. Install ggshield:
   ```bash
   pip install ggshield
   # or
   brew install gitguardian/tap/ggshield
   ```

2. Authenticate with GitGuardian:
   ```bash
   ggshield auth login
   ```
   This will open a browser and automatically create your personal access token.

3. (Optional) Set up pre-commit hooks:
   ```bash
   # Install pre-commit
   pip install pre-commit

   # Add to .pre-commit-config.yaml
   cat <<EOF > .pre-commit-config.yaml
   repos:
     - repo: https://github.com/gitguardian/ggshield
       rev: v1.43.0
       hooks:
         - id: ggshield
           language_version: python3
           stages: [pre-commit]
   EOF

   # Install the hook
   pre-commit install
   ```

4. Push a new commit or re-run the check to verify.

### For Admins

**Override verification for a specific PR:**

Add the `skip-gg-verification` label to the PR. This is useful for:
- Emergency fixes
- External contributors not in your GitGuardian workspace
- Service accounts or bots

**View verification details:**

Click "Details" next to the check status to see:
- Whether the developer was found in the workspace
- Whether they have an active PAT token
- Setup instructions if verification failed

## Troubleshooting

### "Member not found in workspace"

**Possible causes:**
- Developer's GitHub email doesn't match their GitGuardian email
- Developer hasn't been invited to the GitGuardian workspace
- Email privacy settings on GitHub hide the real email

**Solutions:**
- Ensure developer is added to the GitGuardian workspace
- Ask developer to update their email in GitGuardian settings
- Use the override label if necessary

### "Failed to fetch workspace members"

**Possible causes:**
- `GITGUARDIAN_SERVICE_TOKEN` is missing or invalid
- Token doesn't have `members:read` scope
- API endpoint is incorrect (for on-premises installations)

**Solutions:**
- Verify the secret is set correctly in repository settings
- Regenerate the service token with correct scopes
- Check API endpoint URL in the workflow file

### Check is stuck or not running

**Possible causes:**
- Workflow file not on default branch
- GitHub Actions disabled for the repository
- Insufficient permissions

**Solutions:**
- Ensure workflow file exists in `.github/workflows/` on main/default branch
- Check `Settings > Actions > General` to ensure Actions are enabled
- Verify workflow permissions in the YAML file

## API Endpoints Used

- `GET /v1/members` - List workspace members
- `GET /v1/tokens` - List API tokens in workspace

## Limitations & Future Improvements

**Current limitations:**
- Email matching may fail if GitHub and GitGuardian emails differ
- Uses basic string parsing instead of proper JSON parsing (uses `grep` instead of `jq`)
- Doesn't verify that token is actually being used (just that it exists)

**Potential improvements:**
- Add proper JSON parsing with `jq`
- Implement more sophisticated member matching (by name, multiple emails)
- Cache API responses to reduce rate limiting
- Verify token validity by calling `/v1/health` with each developer's token
- Check for `.pre-commit-config.yaml` with ggshield configuration
- Send Slack/email notifications to developers without setup
- Track compliance metrics over time

## Architecture Decisions

### Why GitHub Action instead of GitHub App?

- **Simpler deployment**: No server infrastructure needed
- **Easier to test**: Can run locally with `act`
- **More portable**: Works across any repository by copying the workflow file
- **Better for POC**: Faster to implement and iterate

### Why check for PAT tokens instead of local configuration?

- **More verifiable**: Can't directly check developer machines from GitHub Actions
- **API-accessible**: GitGuardian API provides this data
- **Reasonable proxy**: Having a PAT token strongly suggests ggshield is set up
- **Actionable**: Can provide specific guidance on token creation

### Why two jobs instead of one?

- **Separation of concerns**: Secret scanning and developer verification are distinct
- **Independent status checks**: Each shows up separately on the PR
- **Flexible requirements**: Can require one or both checks in branch protection
- **Better UX**: Clear distinction between "you committed a secret" vs "you need to set up GitGuardian"

## Contributing

This is a proof-of-concept. Feedback and improvements are welcome!

## Resources

- [GitGuardian Documentation](https://docs.gitguardian.com)
- [ggshield GitHub Action](https://github.com/GitGuardian/ggshield-action)
- [GitGuardian API Documentation](https://api.gitguardian.com/docs)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

## License

MIT
