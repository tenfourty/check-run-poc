# GitGuardian Check Run POC - Project Context

## What We're Building

A proof-of-concept GitHub Action that verifies developers have GitGuardian (ggshield) properly set up on their development machines before allowing PRs to be merged. The goal is to enforce that all developers have the security tooling configured locally, not just relying on CI-level scanning.

## The Problem

While ggshield can run in CI to catch secrets, it's better to catch them earlier in the development workflow. Pre-commit hooks on developer machines are ideal, but they're hard to enforce at an organizational level since they can be bypassed or simply not installed.

## Our Solution

A two-part GitHub Actions workflow that:
1. Scans commits for secrets using the existing ggshield-action
2. Verifies the PR author has an active GitGuardian personal access token (as a proxy for having ggshield installed and configured)

If either check fails, the PR is blocked until resolved.

## Current Status

**✅ Initial Implementation Complete**
- Two-job workflow created
- GitGuardian API integration for token verification
- Admin override capability via labels
- Comprehensive documentation

**🚧 Remaining Work**
- ~~Test with real GitGuardian workspace~~ ✅ Tested successfully
- Investigate correct API endpoint/permissions for listing tokens (currently getting 404)
- Improve member matching logic (email/username mapping)
- Add proper JSON parsing with `jq`
- Consider adding token validity checks via `/v1/health`
- Evaluate performance and rate limiting

**📝 Testing Notes**
- Successfully verified workspace membership via `/v1/members` endpoint
- `/v1/tokens` endpoint returns 404 - needs investigation on correct path/permissions
- Current implementation falls back to workspace membership verification (acceptable for POC)

**💡 Future Considerations**
- Check for `.pre-commit-config.yaml` with ggshield configured
- Cache API responses to reduce rate limiting
- Track compliance metrics over time
- Support for on-premises GitGuardian instances

## Development Notes

- We use conventional commits (feat, fix, docs, etc.) for commit messages
- This is a POC, so we're prioritizing speed and clarity over production-readiness
- The current implementation uses basic shell scripting; production would benefit from proper tooling
