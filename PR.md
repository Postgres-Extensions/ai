# PR and commit conventions

Conventions for opening, describing, and pushing PRs across
Postgres-Extensions repos. A few entries note a narrow, repo-specific
exception (release-only CI-trigger PRs, `gh stack` quirks) rather than
stating the general rule as if it had none.

**Be concise, most important first.** State a change once, in the fewest
words that convey it, ordered by decreasing importance. This isn't just
style: the top of a description should be usable as-is for a squash-merge
commit message.

## A PR is not a branch

A **branch** is a git ref — a line of commits, living in exactly one repo.
A **pull request** is a different thing: think of it as its URL. That URL
is always `https://github.com/<repo-owner>/<repo>/pull/<n>`, and
`<repo-owner>/<repo>` is the PR's one and only home — regardless of which
repo the branch it's requesting a merge from happens to live in.

**For every repo in this org, that URL must be under `Postgres-Extensions/`
— never under `jnasbyupgrade/` or any other fork owner.** A PR at
`https://github.com/jnasbyupgrade/<repo>/pull/<n>` is wrong, full stop,
even if its branch, title, and content are otherwise perfect — it never
reached anyone upstream. A branch living on a fork is fine, often the
default (see "Fork vs. direct-to-upstream" below); a *PR* on a fork is
not.

This sounds obvious stated plainly, but conflating "where the branch
lives" with "what repo the PR itself belongs to" is a repeat source of
mistakes — it has produced real PRs at a fork-owner URL, invisible from
upstream's own PR list, sitting forgotten for weeks.

## Merge authority

- Never merge a PR — the repo owner does, personally, always.
- Never push directly to a default branch (`master`/`main`); everything
  goes through a PR.
- Never apply or remove a maintainer-gated label, even with permission to —
  flag it and let a human apply it.
- Never delete a branch without explicit user approval (`git push origin
  --delete`, `git branch -d`/`-D`) — ask first.
- Exception seen in practice: a release workflow's CI-trigger-only empty PR
  exists solely to run CI and is closed once CI passes, never merged.

## Fork vs. direct-to-upstream

**Branch location and PR location are two different questions — don't
conflate them (see "A PR is not a branch" above).**

- Branch: defaults to living on your fork. Push feature-branch commits
  there, not to upstream — pushing to upstream when the branch is supposed
  to be on the fork creates a stray branch and doesn't update the PR.
- PR: its URL is always `github.com/Postgres-Extensions/<repo>/pull/<n>`,
  even when the branch lives on your fork. Create it with `gh pr create
  --repo <org>/<repo> --base <default-branch> --head
  <fork-owner>:<branch>` — don't rely on `gh pr create`'s default target,
  which (run from a fork checkout without `--repo`) puts the PR at a
  fork-owner URL instead. That's the actual bug seen in practice: two PRs
  ended up at `github.com/jnasbyupgrade/object_reference/pull/1` and
  `.../pull/3` (an old pgxntool-version bump, a since-superseded
  test-foundation PR) — wrong URL, invisible from upstream's own PR list,
  forgotten for weeks.
- Find a PR at a fork-owner URL? Close it (it never reached upstream
  reviewers) and redo it targeting upstream.
- Periodically check `gh pr list --repo <fork>` for anything left behind
  by this mistake.
- Exception: a `gh stack`-tracked series needs the branch itself upstream
  too (same-repo) — stack tooling assumes same-repo branches and can
  silently rewrite a PR's base against a fork otherwise.
- `upstream` is otherwise only for the default branch, release tags, and
  branches created directly there (e.g. a stack).
- A PR whose branch lives on a fork exercises fork-based CI behavior
  (trust gates, checkout guards) that a same-repo PR wouldn't — a side
  benefit of the default, not a reason to prefer the fork over upstream
  when a stack requires otherwise.
- CI for a PR whose branch lives on a fork runs under the base repo's
  Actions — monitor there, not the fork's.

## PR titles

- No `"Phase N:"` or similar prefixes.
- No cross-repo references (e.g. a linked companion-repo PR) — those go in
  the body. The title stands alone.
- Just enough to recognize the PR when scanning history, not a summary of
  every changed line.
- `CI: ` prefix (capital, colon, space) only when the diff is confined
  entirely to files that don't affect the actual code:
  - Touching SQL/source, a `.control` file, or anything shipped/behavioral
    disqualifies it, full stop. Unsure → don't apply the prefix.
  - `test/` counts as NOT CI-only by default.
  - CI-*motivated* but touching a real code/test file (a `bin/` script the
    workflow calls, a `Makefile` hook, a submodule bump) is NOT CI-only.
  - Non-code files outside `.github/workflows/` qualify too — `.gitignore`,
    `CLAUDE.md`, pure docs.
  - Verify against the actual file list (`gh pr view <n> --json files`)
    before applying — don't guess from the title or memory.

## PR descriptions

- The opening must work standalone as the commit message (see above): no
  leading header/title line — the PR title is already the subject — and
  no marker delimiting "the commit message part" from "the rest." Just
  let the opening carry its own weight, with any extra context following
  after it.
- Lead the opening with the substantive change and why. Keep incidental
  changes (minor doc tweaks, dependency/action version bumps, small
  cleanups) OUT of it — put them lower or omit them; the diff carries
  those details for anyone who wants them.
- Describe the diff as it stands right now — pull the current file
  list/diff before writing. PRs get rebased and cascaded through; a
  description rots faster than expected.
- State what the PR does and why, not the process that led to it — this
  covers both trial-and-error narration ("first I tried X, then found Y")
  and discovery narration ("found this while doing Y" / "noticed this
  while adding Z") equally; both describe how you got here, not what the
  change is. If the discovery context is genuinely relevant — e.g. it's
  the reason a companion PR exists — state the resulting fact or
  relationship directly ("X and Y both need this fix") rather than
  narrating the sequence of events that led you to it.
- State the actual, verified reason for a change. If the description
  asserts a fact, verify it directly — don't state it from memory.
- Strip historical narrative once it's not load-bearing — no rebase notes,
  no "recreated from #26 because...", no "supersedes #15/#21/#23" story.
- No local-only file paths (`~/foo.md`, `/root/...`) — nothing outside that
  session can resolve them. Describe the change itself, or point at a code
  comment anyone can check.
- Do not hard-wrap paragraphs at a fixed column — write each paragraph as
  a single long line, with a blank line between paragraphs.
- Length past the opening is fine — backstory and detail are often worth
  keeping. If there's enough of it to justify the length, give it real
  structure (headers, bullet lists, separate sections), not one
  undifferentiated block of prose — but structure isn't a substitute for
  cutting length that doesn't earn its place.
- No "Test plan" section by default — rely on CI. Add one only when
  something needs verification CI can't cover.
- A forward-looking coordination note to another PR (e.g. "whichever
  merges second needs to grep for X") is fine to keep. Anything else: does
  removing it make the PR harder to review right now? If not, cut it.
- If merging doesn't fully finish the job — some step must happen live and
  can't be expressed as a diff — put a bolded line at the top of the
  description calling that out.

## Scope: one PR, one concern

- Split unrelated changes even if they touch a file a PR is already
  changing — a functional change from doc/comment cleanup, one fix from
  another. When in doubt, ask rather than defaulting to "it's already in
  the diff."
- Grey area: a trivial, minor fix *can* ride along in an otherwise-related
  PR if it makes sense. Test: would anyone need to read about this specific
  change in git history? If no, bundling is fine — every commit to `main`
  has a cost, and not every fix needs its own PR. Either way, check first
  whether an open PR or issue already covers it.
- If a newly found fix touches the same file/area an already-open PR is
  actively changing, fold it in rather than opening a new PR for the same
  file.
- If several small, separate open PRs address the same narrow concern,
  consolidate into one and close the others as superseded (comment
  pointing at the survivor).
- After combining two PRs' work, update the surviving PR's
  title/description to describe the combined scope, and say in the closing
  comment on the superseded PR that it was folded in, with the commit SHA.
- Ask what order split PRs should merge in if it's not obvious.

## Before pushing: verify for real

- Run the actual test/lint suite, not a dry run (a dry-run flag can be
  misleading) — don't trust a prior session/agent's claim that it passed.
- Never hand-edit generated/expected-output files — regenerate via the
  project's own tooling. If that tooling refuses over an already-reviewed
  diff, look for a documented bypass — never as a way to skip looking at
  the diff.
- Check the PR's actual head-repo owner before pushing a fix — pushing
  only to `upstream` when the head is a fork produces a false "still not
  fixed" result.
- After a squash-merge upstream of your branch, verify ancestry directly
  (`git merge-base --is-ancestor <branch> <new-base>`) — cached
  mergeable/merge-state fields lag.
- `--force-with-lease`, never a bare `--force`, after a rebase.

## Rebasing a PR stack

- Prefer `git rebase --onto <new-base> <old-base-tip>` over a plain
  rebase — a plain rebase can replay huge amounts of superseded history.
- If the base's history was rewritten (not just advanced), diff your old
  known-good tip against the new tip first. Empty diff → reset to the new
  tip and cherry-pick your unique commits, rather than fighting a rebase.
- Rerun the real test suite and lint after, before pushing.
- Under a tight runner budget, don't cascade a rebase across a whole stack
  for a cosmetic fix — batch it into the next substantive push, and say so.

## Force-push / history rewriting

- Never force-push a PR branch without asking first and getting a real
  reply — not even your own draft PR, not even when a task seems to
  require it. A task that implies a rewrite is a signal to ask, not a
  substitute for the user saying yes.
- Need to correct a pushed commit? Add a new commit or ask how to
  restructure — don't default to amend+force-push.
- Any exception must be pre-authorized by name for a specific, narrowly
  gated task — never inferred.
- Never delegate force-push, default-branch pushes, or merges to a
  subagent, even with "check in first" guardrails — these aren't cleanly
  reversible and subagents don't reliably hold the line when a shortcut is
  tempting. Have it report back; the main session does the irreversible
  step.

## Closing issues

- One `Fixes #N` / `Closes #N` / `Resolves #N` per line — never
  comma-combine. Auto-close only reads the first number and silently drops
  the rest.
- Check for duplicates before closing — closing only one of a duplicate
  pair leaves the other open.
- If a project maintains a per-release "issues fixed" line, update it
  per-commit, not just at release time.

## Commit messages

- Specific about outcomes, not vague ("Fix race condition where `X` fails
  due to `Y`", not "Fix various timing issues").
- Don't narrate the journey — end state and why it matters, not how you
  got there. Don't over-explain the obvious.
- Backticks around code identifiers/commands.
- Commit via heredoc, never `-i` flags.
- `Co-Authored-By: Claude <noreply@anthropic.com>` is fine; no "Generated
  with Claude Code" line.
- Change spans two coupled repos (e.g. a library and its test repo): commit
  the primary repo first, capture its hash, then commit the companion repo
  referencing it. Prefer a PR URL over a bare hash once it exists — a URL
  survives a rebase/amend; a hash goes stale silently.

## Multi-session / state-drift hygiene

- Before editing anything already live (a PR's title/description/base/
  branch, a file, a label), check its current actual state first — never
  rely on what an earlier turn, session, or agent assumed. State drifts:
  concurrent merges, mid-session convention changes, a fresh agent with no
  memory of earlier decisions.
- When briefing a subagent for PR work, state current conventions
  explicitly in its prompt (naming rules, branch structure, decisions
  already made) — don't assume it'll discover or preserve them. Verify its
  output against actual current state after, not just its own summary.
- Asked to act on a PR this session didn't open and isn't already working
  on? Confirm with the user first — concurrent sessions make it easy to
  grab the wrong one.
- Before branching off a default branch, fetch and confirm local isn't
  behind remote (compare SHAs) — a stale base risks redoing fixed work.

## Isolation

Use a worktree or fresh clone for edits, not the shared checkout you
started in — unless told explicitly to work there.

## Follow-ups and reminders

- A required follow-up human action (apply a gated label, merge, run
  something manually) goes as a short reminder at the end of the response,
  not buried mid-message.
- A question mixed in with other tasks: answer it inline where relevant,
  and repeat it briefly at the end so it isn't lost.
