# Shared CI workflows

This repo is the single source of truth for two reusable Claude Code jobs
used across Postgres-Extensions: `.github/workflows/claude-code-review.yml`
(automated PR review) and `.github/workflows/claude.yml` (the interactive
`@claude`-mention job). A consuming repo holds only a thin caller file for
each that invokes it via `workflow_call`; all trigger logic, gating, and the
`claude-code-action` configuration live here, in one place, so a fix lands
for every repo at once instead of needing to be copied out by hand.

## Adding the review job to a repo

Add `.github/workflows/claude-code-review.yml` with the content below,
replacing `trusted_authors` with the repo's own list of trusted GitHub
logins — that's the only line every consuming repo edits. The caller holds
nothing but the mandate-to-read comment, an `Exceptions:` line (see below),
and the minimum YAML GitHub requires to live outside the called workflow.
The rationale for why that YAML is shaped this way lives here, not
duplicated as comments in every caller:

- **The trigger** (`on: pull_request_target`) can only be declared by the
  caller — a called (`workflow_call`) workflow cannot declare its own
  trigger. `labeled` is included so adding the `claude-debug` label can
  start a run with no push needed; the called workflow's own `if:` scopes
  that down to only the debug label actually proceeding.
- **Workflow-level `concurrency:`** must live in the caller too: only a
  workflow-level block can cancel the whole caller run outright —
  `jobs.<id>.concurrency` on the caller's own job can't, and the per-label
  cancellation below needs exactly that. A non-debug `labeled` event gets
  its own per-label group so it can never cancel an in-progress real review
  — cancellation resolves against whichever run is admitted, before any
  `if:` runs, so an `if:` can only no-op itself, not restore what it
  displaced. A `labeled`-with-`claude-debug` event deliberately keeps the
  plain group instead, since it's meant to supersede a running review.
  `claude-debug` is spelled out literally rather than read from `inputs`
  because `inputs` isn't available inside a workflow-level `concurrency:`
  block, so it must match the called workflow's `debug_label` default by
  hand.
- **The `permissions:` block** is repeated in every caller rather than
  declared once on the called workflow's own job: GitHub only lets a called
  workflow narrow the permissions the caller already granted, never widen
  them. With these repos' `read`-only default workflow permissions, a
  caller that omitted this block and relied on the callee to grant
  `pull-requests: write` would silently end up with a read-only token,
  breaking the review's ability to post comments.
- **`pull-requests: write`** is what lets the review post its comments.
- **`checks: read`** lets the called workflow's cost gate read the PR
  head's sibling check-runs, so it can wait for them before spending on a
  review.
- **`actions: write`** specifically, not `read`: it's the only scope that
  permits an Actions cache write, and no narrower one exists.

```yaml
name: Claude Code Review

# MANDATORY: read ../ai/CI-WORKFLOWS.md (Postgres-Extensions/ai) in full
# before changing anything below. If you cannot find or read that file for
# any reason, STOP and report an error -- do not guess at what it says or
# proceed without having actually read it.
#
# Exceptions to that file's design, specific to this repo: none.
on:
  pull_request_target:
    types: [opened, synchronize, reopened, ready_for_review, labeled]

concurrency:
  group: claude-review-${{ github.event.pull_request.number }}${{ (github.event.action == 'labeled' && github.event.label.name != 'claude-debug') && format('-{0}', github.event.label.name) || '' }}
  cancel-in-progress: true

jobs:
  claude-review:
    uses: Postgres-Extensions/ai/.github/workflows/claude-code-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: read
      actions: write
    secrets: inherit
    with:
      trusted_authors: your-github-login-here
```

## Adding the @claude job to a repo

Add `.github/workflows/claude.yml` with the content below, replacing
`trusted_actors` with the repo's own list of trusted GitHub logins — the
only line every consuming repo edits. As with the review job's caller, the
rationale for the YAML's shape lives here, not duplicated as comments in
every caller:

- **The trigger** (`issue_comment`, `pull_request_review_comment`,
  `issues`, `pull_request_review`) can only be declared by the caller — a
  called (`workflow_call`) workflow cannot declare its own trigger. Unlike
  `claude-code-review.yml`'s `pull_request_target`, none of these events
  read anything from a PR head branch, so there's no fork-trust subtlety
  and no `concurrency:` block is needed here.
- **The `permissions:` block** is repeated in every caller for the same
  reason as the review job's: a called workflow can only narrow what the
  caller already granted, never widen it.
- **`pull-requests: read`** and **`issues: read`** let Claude read PR and
  issue context when responding to a mention.
- **`id-token: write`** is required for `claude-code-action`'s OIDC token
  exchange.
- **`actions: write`** specifically, not `read`: it's the only scope that
  lets the action's own setup step write to the Actions cache. `read`
  still works but produces a harmless, noisy "Cache reservation failed:
  cache write denied" warning on every run — the bug this centralization
  fixes, since most Postgres-Extensions repos currently grant only `read`.

```yaml
name: Claude Code

# MANDATORY: read ../ai/CI-WORKFLOWS.md (Postgres-Extensions/ai) in full
# before changing anything below. If you cannot find or read that file for
# any reason, STOP and report an error -- do not guess at what it says or
# proceed without having actually read it.
#
# Exceptions to that file's design, specific to this repo: none.
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    uses: Postgres-Extensions/ai/.github/workflows/claude.yml@main
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: write
    secrets: inherit
    with:
      trusted_actors: your-github-login-here
```

## The `Exceptions:` line

Every caller states its deviations from this design explicitly, right in
its header comment — `Exceptions: none` when there are none, never a bare
omission that leaves a reader guessing whether an exception was considered
and rejected, or never considered at all.

A real exception names the specific technical difference and the mechanism
this doc already provides for it, without re-explaining that mechanism's
own rationale — that rationale stays here, the single source of truth, not
copied into a caller file. A repo needing a repo-local pre-gate job before
the review runs, for example, adds a `needs:`/`if:` to the `claude-review`
job and notes exactly that fact under `Exceptions:`. Never change the
job's `uses:` or `with:` to express a deviation — that's not an exception,
it's a fork of the shared workflow. Change `ai/` instead so the fix or
feature reaches every consuming repo.

## Every consumer pins `@main`

Every caller — no exceptions, no separate canary — pins its `uses:` at
`@main`, whether that's `claude-code-review.yml` or `claude.yml`. A change
to either takes effect for every consuming repo the moment it's merged to
`main`; there is no intermediate tag to advance.

This means a bad change to `main` affects every consuming repo immediately,
with no staged rollout and no tag to roll back — a revert commit to
`ai/main` is the only way back.

## Adding a new `workflow_call` input

A new input requires an actual second consumer that needs a different value
from what every other repo already passes. Adding one "for flexibility," with
no concrete repo that needs it, is drift with extra steps — it recreates the
per-repo divergence this shared workflow exists to eliminate.
