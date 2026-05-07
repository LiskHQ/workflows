# Auto-Reviewer

Cross-domain PR reviewer assignment for LiskHQ product repos. When an engineer
opens a PR in `lisk-backend`, `lisk-web`, `lisk-infra`, `lisk-mobile`, or
`lisk-contracts`, this workflow picks a reviewer based on the engineer's squad
and the repo's domain.

## Why

CODEOWNERS routes by file path; team review distributes within a chosen team.
Neither can express "given the author's squad, find the right domain
specialist." This workflow does that with one shared map.

## Files

- [`review-map.yml`](../review-map.yml) — squad / specialist / fallback
  configuration. Single source of truth.
- [`.github/workflows/auto-assign-reviewer.yml`](../.github/workflows/auto-assign-reviewer.yml)
  — reusable workflow with the cascade logic.
- [`.github/workflows/validate-review-map.yml`](../.github/workflows/validate-review-map.yml)
  — CI lint that runs on PRs touching the map.
- [`examples/workflows/auto-assign-reviewer-caller.yml`](../examples/workflows/auto-assign-reviewer-caller.yml)
  — caller stub each product repo copies, varying only `domain:`.

## Cascade

**Bot / external authors short-circuit before the cascade.** If the PR author
is in `bot_authors`, the workflow requests the repo's `bot_pr_owners[<repo>]`
(if mapped) and exits. Bots have no squad and don't need round-robin
distribution; a single per-repo owner is enough. If no owner is mapped for
the repo, the workflow exits 0 silently.

Given a human PR author `A`, repo domain `D`, author squad `S`:

1. **Own-squad specialists.** If `S.specialists[D]` minus `A` is non-empty,
   pick `eligible[PR# % len]`.
2. **Own-squad equivalent domain.** For each `E` in `domain_equivalence[D]`,
   if `S.specialists[E]` minus `A` is non-empty, pick `eligible[PR# % len]`.
   Keeps the review inside the squad when web/mobile are interchangeable.
3. **Own-squad team round-robin.** If prior steps produced nothing, pick from
   `S.members` minus `A`. In-team review is the priority — only after every
   teammate is exhausted do we cross squads. Useful when the author is the
   sole specialist for the domain (e.g., matjazv on a backend or contracts
   PR in `org`) so the PR still gets reviewed by their own team.
4. **Sibling cascade.** Walk `sibling_fallback_order`. For each sibling squad
   `T` (`T ≠ S`), if `T.specialists[D]` minus `A` is non-empty, pick
   `eligible[PR# % len]`. Equivalence is **not** applied cross-squad.
5. **Soft-fail.** Post a PR comment asking for manual assignment. Workflow
   succeeds (never blocks merge).

`PR# % N` gives stateless deterministic round-robin. Author exclusion happens
before modulo, so distribution stays balanced.

### Domain equivalence

```yaml
domain_equivalence:
  web: [mobile]
  mobile: [web]
```

Web and mobile are treated as interchangeable skills *within* a squad — when
an org engineer opens a mobile PR, org's web specialists handle it (instead
of cascading to money). Equivalence is intra-squad only; cross-squad
cascade always uses the actual domain.

## Squad / domain matrix

| Squad     | backend                | web                                 | infra      | mobile                  | contracts |
|-----------|------------------------|-------------------------------------|------------|-------------------------|-----------|
| platform  | sameersubudhi, ishantiw| —                                   | Nazgolze   | —                       | —         |
| money     | vardan10               | —                                   | —          | oskarleonard, Balanced02, eniolam1000752, 5heri | — |
| org       | matjazv                | mmarinovic, mvuco00, ikem-legend, mislavtomic | —          | —                       | matjazv   |
| global    | —                      | —                                   | —          | —                       | ricott1   |

`—` = no own-squad specialist; falls back to any other team member, then
cascades to siblings if the team has no one else available.

`sibling_fallback_order: [money, org, platform, global]`.

## Updating the map

1. Open a PR editing `review-map.yml`.
2. The `validate-review-map.yml` CI checks structure and that every
   GitHub login is an `LiskHQ` org member.
3. Squad TL approves their own squad's section.
4. Merge.

## Kill switches

- **All assignment off:** set `enabled: false` in `review-map.yml`.
- **One repo off:** remove it from `enabled_repos`.
- **Bypass squad cascade for an author:** add login to `bot_authors`. They get
  routed to `bot_pr_owners[<repo>]` if mapped, otherwise skipped.
- **Stop bot routing in a repo:** remove the repo from `bot_pr_owners` (bot
  PRs in that repo will then be skipped).

All take effect on the next PR open with no other action.

## Bot / external author routing

`bot_pr_owners` maps repo name → single GitHub login. Designed for PRs where
squad-based routing doesn't apply: dependabot bumps, renovate bumps, claude
PR commits, copilot review bot, plus listed external contributors.

Current mapping:

| Repo            | Owner       |
|-----------------|-------------|
| `lisk-backend`  | `ishantiw`  |
| `lisk-web`      | `mmarinovic`|
| `lisk-mobile`   | `5heri`     |
| `lisk-infra`    | `Nazgolze`  |
| `lisk-contracts`| `matjazv`   |

The owner is requested verbatim — no round-robin, no domain check, no
fallback. If they're unavailable, the comment trail surfaces who the API
rejected so a human can re-assign.

## What does and doesn't fail the workflow

The workflow is best-effort. It always exits 0 (so it never blocks merges)
even when:

- The author isn't in any squad.
- No eligible reviewer is found (posts a comment instead).
- The reviewer-request API rejects the call (e.g., user isn't a
  collaborator on the target repo).

The workflow exits non-zero only when the `domain:` input is invalid (bad
caller config) — that's a configuration error worth surfacing.

## Triggers

`pull_request_target` on `[opened, ready_for_review]`. We don't fire on
`synchronize` (every push) — that's noise; once a reviewer is requested they
stay requested. Drafts are skipped via `if: github.event.pull_request.draft ==
false` in the caller.

`pull_request_target` is used so PRs from forks work — the workflow only
calls the GitHub API and never checks out PR code, so the security
considerations of `pull_request_target` don't apply here.

## Adding a new repo / domain

1. Add the repo to `enabled_repos` in `review-map.yml`.
2. If it introduces a new domain (not one of the existing 5), update:
   - The `domain` input enum in `auto-assign-reviewer.yml`.
   - The allowed-domains list in `validate-review-map.yml`.
   - Each squad's `specialists` map (add the new domain key where
     applicable).
3. Copy `examples/workflows/auto-assign-reviewer-caller.yml` into the new
   repo's `.github/workflows/`, set `domain:` accordingly.

## Adding a new squad

1. Add the squad block under `squads:` in `review-map.yml`.
2. Add the squad name to `sibling_fallback_order` (position determines
   priority).
3. List members and (optionally) per-domain specialists.
