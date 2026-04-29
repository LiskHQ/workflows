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
- **Trigger**: On-demand only — `/review` comment (optionally `/review opus` or `/review haiku`) or `needs-review` label. Skips bot actors. No auto-run on push (avoids token burn from PRs with many pushes).
- **Does**: Structured code review with inline comments, severity levels, focuses on bugs/security/performance
- **Example**: See `examples/workflows/claude-pr-review-caller.yml`

### 2. PR Summary
- **File**: `claude-pr-summary.yml`
- **Trigger**: Runs on PR `opened`, `reopened`, or `ready_for_review`. Skips doc-only PRs (`**/*.md`, `docs/**`).
- **Does**: Generates concise summary, updates PR description
- **Example**: See `examples/workflows/claude-pr-summary-caller.yml`

### 3. Interactive Claude
- **File**: `claude.yml`
- **Trigger**: Mention `@claude` in issues/PRs
- **Does**: Flexible assistant - explain code, investigate bugs, write docs, answer questions
- **Example**: See `examples/workflows/claude-interactive-caller.yml`

### 4. Auto-assign Reviewer
- **File**: `auto-assign-reviewer.yml`
- **Trigger**: PR `opened` or `ready_for_review` (skips drafts, bots, and authors not in any squad)
- **Does**: Picks a reviewer using `review-map.yml` (squad × domain matrix with sibling cascade) and requests review via the GitHub API. Soft-fails — never blocks merges.
- **Config**: [`review-map.yml`](./review-map.yml) at repo root
- **Docs**: [`docs/auto-reviewer.md`](./docs/auto-reviewer.md)
- **Example**: See `examples/workflows/auto-assign-reviewer-caller.yml`

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
