# Shared CI workflows

`.github/workflows/claude-code-review.yml` in this repo is the single source
of truth for the Claude Code review job used across Postgres-Extensions. A
consuming repo holds only a thin caller file that invokes it via
`workflow_call`; all trigger logic, the cost gate, and the `claude-code-action`
configuration live here, in one place, so a fix lands for every repo at once
instead of needing to be copied out by hand.

## Adding the review job to a repo

Add `.github/workflows/claude-code-review.yml` with this content, replacing
`trusted_authors` with the repo's own list of trusted GitHub logins — that is
the one line every consuming repo edits; everything else is the minimum
GitHub requires to live in the caller rather than the called workflow (see
the comments in the template itself for why each piece can't move):

```yaml
name: Claude Code Review

# Thin caller. All logic lives in Postgres-Extensions/ai; read that file for
# the SECURITY rationale behind pull_request_target + the trusted-author gate.
# Everything below is the minimum GitHub requires to live in THIS repo:
#   - the trigger (a called workflow cannot declare its own)
#   - run-level concurrency (must be here; setting concurrency on a job that
#     `uses:` a reusable workflow deadlocks against the called workflow's group)
#   - the GITHUB_TOKEN ceiling (a called workflow can only narrow it, never widen)
# Do not add logic here. If this repo needs different behavior, change ai/ so
# every repo gets it.
on:
  pull_request_target:
    # `labeled` lets adding the claude-debug label start a run on its own, with
    # no push needed. Scoped in ai/'s job `if:` so only that label proceeds.
    types: [opened, synchronize, reopened, ready_for_review, labeled]

concurrency:
  # A non-debug `labeled` event gets its own per-label group so it can never
  # cancel an in-progress real review: cancellation resolves when a run is
  # admitted, before any `if:` is evaluated, so an `if:` can only no-op itself,
  # not un-cancel what it displaced. labeled+claude-debug deliberately keeps the
  # plain group -- it is meant to supersede a running review.
  # 'claude-debug' is spelled out because `inputs` is not readable here; it must
  # match ai/'s debug_label default.
  group: claude-review-${{ github.event.pull_request.number }}${{ (github.event.action == 'labeled' && github.event.label.name != 'claude-debug') && format('-{0}', github.event.label.name) || '' }}
  cancel-in-progress: true

jobs:
  claude-review:
    uses: Postgres-Extensions/ai/.github/workflows/claude-code-review.yml@v1
    permissions:
      contents: read
      pull-requests: write   # post the review comments
      checks: read           # read sibling check-runs for the cost gate
      # actions: write is the only scope that permits an Actions cache write
      # (no narrower one exists), and its implied read is what lets the cost
      # gate look up its own check-suite id. Don't "tighten" this to read.
      actions: write
    secrets: inherit
    with:
      trusted_authors: your-github-login-here
```

A repo needing genuinely different behavior (e.g. a repo-local pre-gate job
before the review runs) still uses this exact `claude-review` job as-is — it
only prepends its own job(s) that `claude-review`'s own `needs:`/`if:` then
depends on. Never edit the `uses:`/`with:` call itself; change `ai/` instead
so the fix or feature reaches every consuming repo.

## Versioning and rollback

`v1` is a moving lightweight tag on this repo. Every advance also creates an
immutable `v1.N` snapshot tag, so a rollback has a concrete target to point
at:

- **Create** (one-time): `git tag v1.0 <sha> && git tag v1 <sha> && git push origin v1.0 v1`
- **Advance**: `git tag v1.N <sha> && git tag -f v1 <sha> && git push origin v1.N && git push --force origin v1`
- **Roll back**: `git tag -f v1 <previous v1.N> && git push --force origin v1`

Advancing `v1` is a deliberate, separate act from merging to `main` — never
bundle the two. Merging a fix to `main` does not by itself affect any
`@v1`-pinned consumer; only moving the `v1` tag does. A rollback is instant
and needs no pull request in any consuming repo: every `@v1` caller picks up
the new target on its next run.

`pgxntool-test` stays pinned to `@main` permanently, as a live canary — a
change to `claude-code-review.yml` runs there for real before `v1` is ever
moved to include it.

## Adding a new `workflow_call` input

A new input requires an actual second consumer that needs a different value
from what every other repo already passes. Adding one "for flexibility," with
no concrete repo that needs it, is drift with extra steps — it recreates the
per-repo divergence this shared workflow exists to eliminate.
