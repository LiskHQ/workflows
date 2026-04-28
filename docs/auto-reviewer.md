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

Given PR author `A`, repo domain `D`, author squad `S`:

1. **Own-squad specialists.** If `S.specialists[D]` minus `A` is non-empty,
   pick `eligible[PR# % len]`.
2. **Squad-wide fallback.** If `D ∈ S.fallback_to_squad_members_for` and step 1
   produced nothing, pick from `S.members` minus `A`.
3. **Sibling cascade.** Walk `sibling_fallback_order`. For each sibling squad
   `T` (`T ≠ S`), if `T.specialists[D]` minus `A` is non-empty, pick
   `eligible[PR# % len]`.
4. **Soft-fail.** Post a PR comment asking for manual assignment. Workflow
   succeeds (never blocks merge).

`PR# % N` gives stateless deterministic round-robin. Author exclusion happens
before modulo, so distribution stays balanced.

## Squad / domain matrix

| Squad     | backend                | web                                 | infra      | mobile                  | contracts |
|-----------|------------------------|-------------------------------------|------------|-------------------------|-----------|
| platform  | sameersubudhi, ishantiw| —                                   | Nazgolze   | —                       | —         |
| money     | vardan10               | —                                   | —          | oskarleonard, Balanced02, eniolam1000752, 5heri | — |
| org       | matjazv                | mmarinovic, mvuco00, ikem-legend, mislavtomic | —          | —                       | matjazv   |
| global    | —                      | —                                   | —          | —                       | ricott1   |

`—` = no own-squad specialist; cascades to siblings.

`platform.fallback_to_squad_members_for: [infra]` — when Nazgolze opens an
infra PR, fall back to any other platform member instead of jumping to
siblings.

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
- **Skip a specific author:** add login to `bot_authors`.

All take effect on the next PR open with no other action.

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
