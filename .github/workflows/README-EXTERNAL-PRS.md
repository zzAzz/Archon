# Claude Review for External PRs (Forked Repositories)

## Overview

This document explains the two-stage workflow system for running Claude Code reviews on pull requests from forked repositories. This approach solves the GitHub Actions security limitation where forked PRs cannot access repository secrets.

## The Problem

- **Direct PRs**: Contributors with write access push branches directly to the main repository. Workflows have full access to secrets. ✅
- **Forked PRs**: External contributors fork the repo and submit PRs. GitHub blocks secret access for security. ❌

## The Solution: Two-Stage Workflow

### Stage 1: PR Information Collection (`claude-review-ext-stage1.yml`)

- **Trigger**: Runs on `pull_request` events or when someone comments `@claude-review-ext`
- **Permissions**: Read-only, no secrets
- **Purpose**: Safely collect PR information from untrusted code
- **Output**: Uploads PR details as an artifact

### Stage 2: Secure Review (`claude-review-ext.yml`)

- **Trigger**: Runs after Stage 1 completes via `workflow_run`
- **Permissions**: Has access to repository secrets
- **Purpose**: Downloads PR info and runs Claude review in secure context
- **Security**: Never checks out untrusted PR code directly

## How to Use

### For External Contributors

1. Fork the repository
2. Create your feature branch
3. Submit a pull request
4. Wait for automatic review or ask a maintainer to comment `@claude-review-ext`

### For Maintainers

To trigger a review on an external PR:
```
@claude-review-ext
```

Authorized users:
- @Wirasm
- @coleam00
- @sean-eskerium

## Security Model

### Stage 1 (Untrusted Environment)
```yaml
permissions:
  contents: read      # Read-only
  pull-requests: read # Read-only
# No secrets available
```

### Stage 2 (Trusted Environment)
```yaml
permissions:
  contents: read      # Read base repo
  pull-requests: write # Post comments
  issues: write       # Post comments
  id-token: write     # OIDC auth
# Secrets ARE available
```

### Key Security Features

1. **No Direct Code Execution**: Stage 2 never runs PR code directly
2. **Artifact-Based Communication**: PR information passed via artifacts, not code execution
3. **Authorization Check**: Only authorized users can trigger reviews via comments
4. **Base Branch Checkout**: Stage 2 checks out main branch, not PR branch
5. **Diff Analysis**: Reviews changes via git diff, not by running PR code

## Workflow Diagram

```
┌──────────────────┐
│ External Fork PR │
└────────┬─────────┘
         │
         v
┌────────────────────────┐
│ Stage 1: Collect Info  │
│ (No Secrets)           │
│ - Checkout PR code     │
│ - Gather PR details    │
│ - Upload as artifact   │
└────────┬───────────────┘
         │
         v
┌────────────────────────┐
│ Stage 2: Claude Review │
│ (Has Secrets)          │
│ - Download artifact    │
│ - Checkout base branch │
│ - Analyze diff         │
│ - Post review          │
└────────────────────────┘
```

## Comparison with Standard Workflows

| Feature | Standard (`@claude-review`) | External (`@claude-review-ext`) |
|---------|----------------------------|--------------------------------|
| Trigger | `@claude-review` | `@claude-review-ext` |
| Works on direct branches | ✅ | ✅ |
| Works on forked PRs | ❌ | ✅ |
| Workflow files | 1 | 2 |
| Security model | Direct execution | Two-stage isolation |
| Setup complexity | Simple | Moderate |

## Troubleshooting

### Review Not Triggering

1. Check if you used the correct trigger: `@claude-review-ext` (not `@claude-review`)
2. Verify you're an authorized user
3. Check if Stage 1 workflow completed successfully

### Stage 1 Fails

- Usually indicates a problem with the PR itself
- Check workflow logs for details

### Stage 2 Fails

- Often indicates an issue with Claude API or token
- Check if `CLAUDE_CODE_OAUTH_TOKEN` secret is configured
- Verify artifact was created in Stage 1

## Limitations

1. **Slight Delay**: Two-stage process takes longer than direct review
2. **Artifact Size**: Large PRs might exceed artifact limits
3. **Manual Trigger**: Automatic reviews on PR open require maintainer action for forks

## Future Improvements

- GitHub App integration for better fork support
- Automatic security scanning before review
- Caching for faster reviews on PR updates

## References

- [GitHub Actions Security Best Practices](https://docs.github.com/en/actions/security-guides)
- [Preventing PWN Requests](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/)
- [Claude Code GitHub Actions](https://docs.anthropic.com/en/docs/claude-code/github-actions)