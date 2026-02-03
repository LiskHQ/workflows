# Claude Code Workflows

Reusable GitHub Actions workflows for Claude Code reviews. Each repo defines its own context via CLAUDE.md.

## Setup

1. Add secret `CLAUDE_CODE_OAUTH_TOKEN` to your repo (get from https://claude.ai/settings/oauth-tokens)
2. Copy workflow files from `examples/workflows/` to your `.github/workflows/`
3. Update `uses: lisk/workflows/...` to your org name
4. (Optional) Create CLAUDE.md in repo root with review guidelines

## Workflows

### 1. PR Review
- **File**: `claude-pr-review.yml`
- **Trigger**: Comment `/review` or add `needs-review` label
- **Does**: Structured code review with inline comments, severity levels, focuses on bugs/security/performance
- **Example**: See `examples/workflows/claude-pr-review-caller.yml`

### 2. PR Summary
- **File**: `claude-pr-summary.yml`
- **Trigger**: Auto-runs when PR opens or updates
- **Does**: Generates concise summary, updates PR description
- **Example**: See `examples/workflows/claude-pr-summary-caller.yml`

### 3. Interactive Claude
- **File**: `claude.yml`
- **Trigger**: Mention `@claude` in issues/PRs
- **Does**: Flexible assistant - explain code, investigate bugs, write docs, answer questions
- **Example**: See `examples/workflows/claude-interactive-caller.yml`

## CLAUDE.md (Optional)

Create in repo root to define review priorities:

```markdown
# Repository Context

## Technology Stack
- TypeScript + Express.js + PostgreSQL

## Critical
- SQL injection prevention (use Prisma, no raw queries)
- JWT auth on all endpoints
- Never log passwords/tokens

## Testing
- Integration tests for all endpoints
- 80%+ coverage for new code
```

## Troubleshooting

**Workflow not running?**
- Check secret is set
- Verify trigger (comment `/review`, mention `@claude`, etc)
- Check workflow file is in `.github/workflows/`

**"workflow_call not found"?**
- Verify `uses:` path matches your org
- Check branch exists (@main)

## How It Works

Claude automatically reads CLAUDE.md when it exists. If not, it infers context from package.json, README, etc. Same workflow files work across all repos.
