# PR and commit conventions

Synthesized from `PR.md` scratch notes left behind in five separate
Postgres-Extensions repos (`extension_tools`, `linter`, `object_reference`,
`pgxntool`, `pgxntool-test`, `test_factory`) — each one a running log of
corrections an agent actually received in-session. Collected here as one
cross-repo reference. Where a rule was stated identically in multiple repos
it's merged into one entry; where a repo had a narrow local exception
(release-only CI-trigger PRs, `gh stack` quirks) that's called out as such
rather than generalized away.

## Merge authority

- **Never merge a PR, for any reason** — not even when CI is green and the
  PR looks done. The repo owner merges every PR personally, always.
- **Never push directly to a default branch** (`master`/`main`), even for a
  one-line fix. Everything goes through a PR.
- Never apply or remove a maintainer-gated label (e.g. one that overrides a
  required check) even if the authenticated account technically has
  permission to. If a PR looks like it needs one, say so and let a human
  apply it.
- Narrow, project-specific exception seen in practice: a release workflow's
  CI-trigger-only empty commit/PR that exists solely to run CI and must
  never be merged by anyone — it gets closed once CI passes, not merged.
  Exceptions like this are a property of a specific documented workflow, not
  something to infer generally.

## Fork vs. direct-to-upstream

- **Default: open new PRs through your fork** — push the branch to your
  fork, PR fork → upstream's default branch. This is the same path any real
  external contributor has to use, so it actually exercises whether CI
  (trust gates, checkout logic, unsafe-fork-checkout guards, etc.) behaves
  correctly for fork-headed PRs, not just the easier same-repo case.
- **Exception: a `gh stack`-tracked stacked-PR series.** Stack tooling's
  rebase/tracking mechanics (and `gh stack link` specifically) assume
  same-repo branches and can silently rewrite a PR's base with no warning
  when a fork is involved. Use direct-to-upstream branches for a stack, and
  say why in the PR body so it doesn't read as a random inconsistency.
- **Never open a PR with both base and head on your fork** (fork-to-fork).
  If something needs to stack on other work, push the base branch upstream
  and stack there.
- Push feature-branch commits to your fork (`origin`), never to `upstream`
  for a fork-headed PR — pushing to `upstream` creates a stray branch and
  does *not* update the PR.
- `upstream` is only for: the default branch, release tags, and the rare
  branch that was itself created directly on upstream.
- CI for a fork-headed PR runs under the **base repo's** Actions, not the
  fork's — monitor/poll there.
- Don't let fork-based and direct-to-upstream modes bleed into each other
  out of habit. Pick one per PR for a real reason, not by default or
  convenience in the moment.

## PR titles

- No `"Phase N:"` or similar process-phase prefixes.
- Never put a cross-repo reference in the title (e.g. a linked PR in a
  companion repo) — that belongs in the body. The title should describe the
  change itself and stand alone.
- Contain only the essential info needed to recognize the PR later when
  scanning history — not a summary of every changed line.
- A `CI: ` prefix (capital, colon, space) applies only when the diff is
  confined **entirely** to files that don't affect the actual code — not
  just "lives under `.github/workflows/`". Concretely:
  - If anything touches SQL/source, a `.control` file, or anything else
    affecting what gets installed or how it behaves at runtime, it is NOT
    CI-only, full stop. When unsure, don't apply the prefix.
  - `test/` is treated as NOT CI-only by default, even though test files
    aren't shipped code themselves — default to excluding it rather than
    judging case by case.
  - A PR that's CI-*motivated* but also touches a real code/test file (a
    `bin/` script a workflow calls, a `Makefile` hook that affects what
    ships, a submodule bump) is NOT CI-only under this reading.
  - Other genuinely non-code files also qualify even outside
    `.github/workflows/` — `.gitignore`, a `CLAUDE.md` change, pure
    documentation.
  - Verify with the actual file list (e.g. `gh pr view <n> --json files`)
    before applying the prefix — don't guess from the title or memory of
    what the PR was "supposed to" contain.

## PR descriptions

- **Describe the diff as it stands right now.** Pull the actual current
  file list/diff before writing or editing a description — don't reuse what
  you last knew about the PR's contents. PRs get rebased, squash-merged
  into, and cascaded through; the description rots faster than expected.
- **State what the PR does, not how you got there.** No "first I tried X,
  then found Y" narrative, no log of false starts or intermediate states
  that were later fixed. Default to a short, flat list of the real changes.
- **State the actual, verified reason for a change, not a plausible-sounding
  one.** If a description asserts a fact ("X is still true today", "Y
  doesn't work"), verify it directly — empirically, if that's what it
  takes — before writing it down, rather than stating it from memory or
  plausibility.
- **Strip historical narrative once it's no longer load-bearing** — no
  "rebase notes," no "recreated from #26 because...", no blow-by-blow of
  what conflicted with what, no "this supersedes #15/#21/#23" story. Once
  that context isn't needed to understand or review the current diff, cut
  it; don't archive it in the PR body.
- **Be concise.** Say what changed and why, once. Don't restate the same
  point in a summary paragraph, a numbered list, and a test-plan checklist.
- **Order lists by decreasing importance** — impact × likelihood someone
  reading history later cares, most impactful first.
- **Be specific about outcomes, not vague summaries.** "Fix race condition
  where `X` fails due to `Y`" beats "Fix various timing issues."
- **No local-only file paths** (`~/foo.md`, `/root/...`, anything inside a
  particular session's working directory). Nothing outside that session can
  resolve them. If a local doc's guidance shaped the change, describe the
  change itself, or point at a code comment that explains it in terms
  anyone can check.
- **No "Test plan" section by default** — rely on CI. Only add one when
  something needs verification CI genuinely can't cover (a manual step, a
  live trigger, a documented known limitation).
- A description referencing another PR purely for forward-looking
  coordination (e.g. "whichever of these merges second needs to grep for
  X") is fine to keep — that's actually useful to the next reader. The test
  for any other aside: does removing it make the PR harder to review right
  now? If not, cut it.
- **If merging doesn't fully finish the job** — some step has to happen
  live and can't be expressed as a diff (e.g. configuring branch protection
  via the API) — put a bolded line at the very top of the description
  calling that out, so it isn't missed once the PR is green and merged.

## Scope: one PR, one concern

- Don't cram unrelated changes into one PR just because they were found or
  fixed in the same pass — split a functional change from doc/comment-only
  cleanup, and split independent fixes from each other, even if they touch
  a file a PR is already changing. When in doubt whether something belongs,
  ask rather than defaulting to "it's already in the diff, may as well
  leave it."
- Exception: if a newly found fix touches the same file/area an
  *already-open* PR is actively changing, fold it into that PR rather than
  spinning up a separate one for the same file.
- The reverse also holds: if several small, currently-separate open PRs all
  address the same narrow concern, consolidate into one and close the
  others as superseded (with a comment on the closed ones pointing at the
  survivor) rather than leaving them to collide at merge time.
- After combining two PRs' work into one, update the surviving PR's
  title/description to describe the *combined* scope — don't leave it
  describing only the original, narrower scope. State plainly in the
  closing comment on the superseded PR that it was folded in, with the
  commit SHA.
- Ask what order split PRs should merge in if it's not obvious; don't
  assume.

## Before pushing: verify for real

- Run the actual test/lint suite, not a dry run — a dry-run flag can be
  actively misleading (a recipe containing a nested make invocation, for
  example, executes for real even under `-n`) — and don't trust a previous
  session/agent's claim that something passed.
- Never hand-write or hand-edit generated/expected-output files (test
  fixtures, snapshots) — regenerate them via the project's own tooling. If
  that tooling refuses to run because of a real-but-expected, already
  reviewed difference, look for a documented bypass for that specific
  safeguard — but only after independently confirming the diff is the
  intended change, never as a way to skip looking at it.
- Check the actual head-repository owner on the PR before pushing a fix —
  don't assume fork vs. same-repo based on earlier state. Pushing a fix only
  to `upstream` when the PR's real head lives on a fork produces a
  confusing false "still not fixed" result that looks like a platform bug
  but isn't.
- After a squash-merge anywhere upstream of a branch you're working on,
  verify ancestry directly (`git merge-base --is-ancestor <branch>
  <new-base>`) before trusting a hosting platform's cached
  mergeable/merge-state fields — those lag and can show stale values.
- Use `--force-with-lease`, never a bare `--force`, after any rebase.

## Rebasing a PR stack

- When an earlier PR in a stack merges, prefer `git rebase --onto
  <new-base> <old-base-tip>` over a plain `git rebase <new-base>` — a plain
  rebase can try to replay huge amounts of already-superseded history
  (e.g. squash-merged subtree commits) and produce a wall of spurious
  conflicts.
- If the base branch's history was itself rewritten (not just advanced)
  since your last fetch, diff your old known-good tip against the new tip
  first. If it's empty, the content didn't change, only commit hashes did —
  the safe move is resetting to the new tip and cherry-picking just your
  branch's own unique commits on top, rather than fighting a rebase against
  history that no longer exists in the same shape.
- Always rebuild and rerun the real test suite (and lint) after any of
  this, before pushing — a clean rebase doesn't guarantee the result still
  works.
- When runner budget/capacity is tight, don't cascade a rebase across a
  whole PR stack just for a cosmetic fix — batch it into the next
  substantive push instead, and say so explicitly rather than silently
  deferring.

## Force-push / history rewriting

- **Never force-push a PR branch for any reason without literally asking
  first and getting a real reply** — not even your own unreviewed/draft PR,
  not even when a task seems to structurally require rewriting history. A
  task that *implies* a rewrite is a signal to stop and ask, not a
  substitute for the user explicitly saying yes.
- If a pushed commit needs correcting, or a branch needs a new base, add a
  new commit or ask how to restructure — don't default to amend+force-push.
- Any standing exception to this must be pre-authorized by name for a
  specific, narrowly-gated task (e.g. one documented workflow step that
  amends a single tip-of-default-branch commit under specific, verified
  conditions) — never inferred from context.
- Never delegate force-push, default-branch pushes, or merges to a
  subagent, even with explicit "stop and check in first" guardrails in the
  prompt — subagents don't reliably honor those guardrails when a shortcut
  is tempting, and these actions aren't cleanly reversible. Keep the
  capability out of the subagent's hands entirely; have it report back and
  let the main session perform the irreversible step itself.

## Closing issues

- One `Fixes #N` / `Closes #N` / `Resolves #N` per line. Never
  comma-combine (`Fixes #7, #14, #19`) — the hosting platform's auto-close
  only reads the first number in that pattern and silently drops the rest.
- Check for duplicate issue reports before closing one — closing only one
  of a duplicate pair leaves the other open unnecessarily.
- If a project maintains a per-release "issues fixed" changelog line,
  update it per-commit as issues are closed, not just compiled at release
  time — "does this change need its own changelog bullet" and "was a
  numbered issue closed" are two separate decisions; the latter always goes
  on the list when true.

## Commit messages

- Order bullets by decreasing importance, same as PR descriptions.
- Be specific about outcomes, not vague ("Fix race condition where `X`
  fails due to `Y`", not "Fix various timing issues").
- Don't narrate the development journey — describe the end state and why
  it matters, not how you got there.
- Don't over-explain the obvious.
- Wrap code identifiers/commands in backticks.
- Commit via heredoc (`git commit -m "$(cat <<'EOF' ... EOF)"`), never `-i`
  flags.
- `Co-Authored-By: Claude <noreply@anthropic.com>` trailer is fine; don't
  add a "Generated with Claude Code" line anywhere in the body.
- When a change spans two coupled repos (e.g. a library and its companion
  test repo), commit the primary repo first, capture its hash, then commit
  the companion repo referencing it — never the other order, since the
  reference needs a real hash/PR URL to point at. Prefer linking a PR URL
  over a bare commit hash once the PR exists — a URL survives a rebase or
  amend of the referenced commit; a bare hash goes stale silently.

## Multi-session / state-drift hygiene

This theme recurred across every repo's notes, independently:

- **Before editing anything already live** (a PR's title, description,
  base, or branch; a file; a label) — look at its current actual state
  first. Never rely on what an earlier turn, an earlier session, or a
  different agent assumed it to be. State drifts: other merges happen
  concurrently, conventions get established mid-session, and a fresh agent
  has no memory of decisions made earlier in the same thread — it will
  re-derive (and can get wrong) whatever pattern it infers from what it can
  currently see.
- **When briefing a subagent for PR-related work**, state the current
  relevant conventions explicitly in its prompt (naming rules, active
  branch structure, decisions already made) rather than assuming it will
  discover or preserve them. After it reports back, verify its output
  against the actual current state rather than trusting its own summary.
- If asked to act on an existing PR that the current session didn't open
  and isn't already working on, confirm with the user first before
  touching it — multiple sessions can run concurrently across related
  repos and it's easy to grab the wrong one.
- Before branching off a default branch, fetch and confirm the local copy
  isn't behind the remote (compare SHAs directly) — branching from a stale
  base risks redoing work already fixed upstream.

## Isolation

Use a worktree or a fresh clone for any actual edit, not the shared
checkout you started in — only work directly in a shared checkout when
told to explicitly.

## Follow-ups and reminders

- When a task requires a follow-up human action (apply a maintainer-gated
  label, merge a PR, run something manually), state that as a short
  reminder **at the end** of the response, not buried mid-message.
- When a prompt mixes a question with other tasks or instructions, answer
  the question inline where relevant *and* repeat it briefly at the very
  end of the response, so it isn't lost in a longer, action-heavy reply.

## Code comments

- Attribution is fine when adapting a pattern from elsewhere ("modeled on
  X's Y script").
- Justification that leans on context a reader of *this* repo doesn't
  have ("unlike \<other repo\>, which does Z for reasons specific to its own
  history") is not — every comment should stand on its own logic, as if
  arrived at independently.
- Only add a comment at a bug-fix site when the bug was caused by something
  non-obvious (a missed corner case, a subtle interaction, an incorrect
  assumption). The comment should explain what was missed, not label
  itself as a bug fix. Skip comments for obvious fixes.
