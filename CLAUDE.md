# CLAUDE.md

This file provides guidance to Claude Code (and other AI agents) working
in Postgres-Extensions repos.

## First: make sure a local checkout exists

If you reached this file via
https://github.com/Postgres-Extensions/ai/blob/main/CLAUDE.md rather than
a local `../ai/CLAUDE.md`, clone this repo now so future references
resolve locally instead of over the network:

```bash
git clone https://github.com/Postgres-Extensions/ai.git ../ai
```

(run from the current repo's root, so `ai/` lands as a sibling: `../ai/`)

## Keep ../ai/ up to date

`../ai/` may be a checkout shared with other concurrent agents/sessions —
it can move even on a session where you never checked yourself. Worse, a
typical checkout's `origin` is a personal fork that itself lags the real
upstream, so `git -C ../ai pull --ff-only` can report "already up to
date" while `Postgres-Extensions/ai` has moved on — a no-op pull is not
proof you're actually current. At the start of every session, use
`../ai/bin/check-upstream-sync` instead: it resolves whichever remote
actually points at `Postgres-Extensions/ai` (by URL, not by name, so it
works whether that's `origin` or `upstream`) and compares against that
directly:

```bash
../ai/bin/check-upstream-sync <sha-you-last-read-at>
```

It prints the new commits and exits non-zero when there are any it
hasn't seen; it prints nothing and exits 0 otherwise.

Note the commit `../ai/` is at the first time you read its docs each
session, and pass that noted commit to the script on every later check.
If anything shows up, re-read the affected file(s) before continuing to
rely on your in-context memory of them.

### Re-checking during a long session: use a scheduler, not memory

Written instructions don't survive a long session on their own — nothing
forces action on a wall-clock cadence mid-session without something to
enforce it. If your tool has a scheduler that actually re-enqueues a
prompt at a future time (Claude Code's `CronCreate`), use it:

- **Sync job**: recurring, durable, roughly hourly — use the current
  minute (whatever it is when you create the job) as the cron minute
  rather than picking one; nudge it by a minute if it happens to land
  exactly on `:00`/`:30` (see `CronCreate`'s own note on thundering-herd
  minutes). Prompt prefixed `[ai-sync]`, instructing: run
  `../ai/bin/check-upstream-sync` against the last noted SHA, re-read
  anything it reports.
- **Renewal job**: recurring, durable, every few days — comfortably
  under `CronCreate`'s 7-day auto-expiry (e.g. every 3 days leaves margin
  even across a month-boundary cron quirk). Prompt prefixed
  `[ai-sync-renew]`, instructing: list jobs, delete the current sync and
  renewal jobs (matched by their prefixes), then recreate both fresh.
  This is what keeps the pair alive indefinitely — skip it and both jobs
  silently expire after 7 days, and the checking stops with no warning.

This mechanism is Claude Code-specific. An agent whose tool has no
equivalent scheduler has no way to actually enforce the cadence, and
should say so rather than silently falling back to an easily-forgotten
"remember to check" habit.

## Related docs in this repo

- [PR.md](./PR.md) — PR, commit, and git-process conventions (merge
  authority, fork workflow, force-push policy, etc.)
- [CODE_STYLE.md](./CODE_STYLE.md) — comment conventions and PR/issue
  reference format
- [test/CLAUDE.md](./test/CLAUDE.md) — extension testing conventions
  (schema/search_path isolation)
- [RELEASE.md](./RELEASE.md) — shared release process for repos that
  distribute a PGXN extension via pgxntool (versioning, tagging, PGXN
  upload)

## GitHub comments: identify as an agent

Any comment or reply an agent posts on GitHub — a PR comment, an issue
comment, an inline review reply, a commit comment — must open with a
short self-identification (e.g. "Claude here") so a reader can tell at a
glance it's agentic, not from a human. This does not apply to PR titles,
descriptions, or commit messages, which are already attributed
structurally (PR author, `Co-Authored-By` line) rather than through the
text itself.

## GitHub CI: monitor after every push

After every `git push` that updates a branch — whether it opens/updates a
PR or pushes to a branch with none yet — monitor the resulting CI run to
completion in a background task, and fix failures immediately rather than
leaving them for the user to notice. Don't consider the push complete
until its CI run is green, or the failure is understood and explicitly
accepted by the user.

- Use `gh pr checks <pr> --watch` when the branch has an open PR;
  otherwise `gh run list --branch <branch> --limit 1` to find the run,
  then `gh run watch <run-id>`.
- Prefer `--commit <SHA>` over `--branch` when available — `--branch` has
  a race: two pushes landing close together (e.g. two sessions pushing in
  parallel) can make it pick up the wrong run.
- If a repo's CI has a docs-only/changed-files gate, check that jobs
  skipped for the expected reason rather than assuming a quiet run means
  CI didn't trigger.
- Where a repo has a `claude-code-review` job, a green conclusion only
  means the review *ran*, not that its findings were addressed. Fetch and
  read its actual PR comment (from the comment URL, or `gh pr view --json
  comments`) once the job completes.
  - For every finding, as soon as you see it: fix it, or ask the user for
    direction. Never leave a finding unaddressed and unacknowledged just
    because the check itself is green.
  - This applies to any code review, not just the automated job: once a
    finding (bot or human reviewer) is fixed, reply to it — an inline
    reply on the specific comment if it's an inline thread, otherwise a PR
    comment referencing it — stating what changed. A fix visible only in
    the diff looks identical to an ignored finding from the reviewer's
    side.
  - If you believe a finding doesn't need fixing, that's not your call to
    make unilaterally — state your reasoning and get the user's explicit
    confirmation before treating it as resolved. Silently deciding not to
    fix something and moving on is exactly the "unacknowledged" failure
    mode this section exists to prevent.
  - If there's any question in your mind about whether a finding even
    makes sense — you don't follow what it's pointing at, or its claim
    doesn't seem to match what the code actually does — ask the user
    rather than guessing at an interpretation and acting on that guess.
  - Don't blindly implement findings either — treat each one as a claim
    to verify against the actual code before acting on it. The review
    re-reads the diff each run without the code's full history or design
    intent in mind, and can be wrong or miss context.
  - If a finding — or one remotely similar to a finding from an earlier
    push on the same PR — shows up again, stop immediately and ask the
    user before doing anything else. A recurrence means you and the
    review bot are not in alignment (either the earlier fix didn't
    actually address it, or you and the reviewer disagree about it), and
    continuing to guess burns tokens without resolving the actual
    disagreement.

## Before writing to memory, check whether it belongs in ../ai/ instead

Before saving anything to a memory system (auto-memory, a session note,
any other persistent-across-sessions store), check whether it's actually
an org-wide convention that belongs in `../ai/` instead — memory is
scoped to one agent/repo/session, so a convention saved only there is
invisible to everyone else and gets re-derived inconsistently. Ask the
user when genuinely unclear which it is, rather than deciding
unilaterally.

## Testability

Ask, before writing code, whether it could be tested in isolation (as a
unit) instead of only through a full end-to-end path — isolated tests are
faster (no environment/service/build setup) and easier to make exhaustive.
Retrofitting testability later is more expensive than designing for it up
front.

- Prefer accepting configuration as explicit function/script arguments
  over implicit environment variables or global state, when otherwise
  equivalent — an argument can be varied directly in a test loop; an env
  var has to be set/unset around each invocation and can leak between
  cases.
- When a test needs to prove something was (or wasn't) called — not just
  that a failure from it was tolerated, which is a materially weaker claim
  — reach for a stub/mock standing in for the real script/command rather
  than only checking visible side effects, or faking out an entire
  directory tree. Prefer a dedicated "which script/path to invoke"
  variable that a test can override over a broader, more expensive fake.
- Don't re-test behavior that's already established and owned elsewhere —
  an underlying dependency's own documented mechanics (e.g. a build
  framework's own file-deletion rules), or a different layer of this
  project's own code. Test that *your own code* correctly wires into,
  configures, or triggers that other layer, not that the other layer then
  does the right thing with it — that's its own well-established
  responsibility.

## Executable bit safety

`sed -i` (and similar in-place file-rewriting tools) can silently drop a
file's executable bit — the in-place edit creates a new file and renames
it over the original, using default permissions rather than preserving
the original mode. This has caused real regressions (a script losing its
`+x` bit cascaded into a wide swath of unrelated-looking test failures
before the actual cause was found).

After using `sed -i` or any other tool that rewrites a file in place,
check whether the executable bit survived. For a tracked file, `git diff
--summary` right after the edit is the fastest check — a `100755 =>
100644` line means it needs `chmod +x` restored before doing anything
else with it.

## Scripts

- Use `#!/usr/bin/env <interpreter>` (e.g. `#!/usr/bin/env bash`,
  `#!/usr/bin/env python3`) for any executable script, never a hardcoded
  interpreter path (`#!/bin/bash`, `#!/usr/bin/python3`) — this applies to
  every interpreted language a script might be written in, not just
  shell. A hardcoded path fails on systems where that interpreter lives
  elsewhere (some BSDs, NixOS, Homebrew on macOS); `env <interpreter>`
  finds it on `PATH`.
- In shell scripts specifically: never use `echo ""` to print a blank
  line; just use `echo` with no arguments.

## Test failures are never acceptable

A test failure — whether from a smoke test, a verification run, or a full
suite — must be reported to the user immediately, never rationalized as
"pre-existing," "expected on this branch," or "unrelated" without
verifying that directly. If failures exist, work with the user to fix
them or plan commit/merge order to avoid them.

## Writing documentation

When documenting current behavior, avoid narrating the past ("previously
X, but now Y") unless the change itself is the point being made — readers
generally need to know what *is* true now, not what used to be.

## If this repo embeds pgxntool

Most repos in this org vendor `pgxntool` (via `git subtree`) as their
build system. Its own docs (`pgxntool/README.asc`, `pgxntool/CLAUDE.md`)
are **not** auto-loaded by an agent working in the consuming repo — for
any non-trivial build or test work, read them first rather than
re-deriving or re-documenting pgxntool's own behavior locally. (`DATA`,
`MODULES`, `DOCS`, and `installcheck` are PGXS variables/targets that
pgxntool only wraps/seeds, not its own — a common point of confusion.)

Bias toward putting a new test dimension (a new PostgreSQL major, an
UPDATE-vs-CREATE-EXTENSION leg, etc.) in `make` rather than `ci.yml`: it
can then be run locally, and it never spins up an extra container/job.
This is a real tradeoff, not an absolute rule — needing real isolation to
mean anything (e.g. a dedicated cluster for one extension's install path)
is one good reason to put a dimension in CI instead, not the only one.

### Version-specific SQL files

pgxntool's own `README.asc` ("Version-Specific SQL Files") already covers
tracking version-specific install/update scripts by default, why, when
it's OK to skip one, and why you must never hand-edit one that's no
longer current — read that rather than re-deriving it here.

One case its docs don't yet cover: a pseudo-version a repo uses to mean
"current, between releases" (e.g. a `stable` default_version) is the
*opposite* of the skipped-version case they document — it's permanently
current, not transiently left behind, so it would be regenerated and
re-diffed on every single source edit if tracked, for zero
test-coverage value. Gitignore it instead of the one-time `rm` their docs
recommend for a skipped version. This belongs in pgxntool's own docs
long-term; this is the canonical statement of it until it lands there.

Name the gitignore entry exactly (`sql/<ext>--stable.sql`), not a glob like
`sql/*--stable.sql` — that `*` also matches across the second `--` in the
update-*to*-stable script's own name (`sql/<ext>--<last-released>--stable.sql`,
see `RELEASE.md`'s "Ongoing development" section), which must stay
committed. A glob there silently sweeps up the update script too (caught
directly: it stopped showing as untracked right after being created).

### PostgreSQL version support policy

Never support a fresh install on a PostgreSQL version where the
extension's update path is known to be broken — a version that can't be
updated *to* isn't truly supported. If a PostgreSQL major changes what an
update script is allowed to do (e.g. a statement that can't run inside an
extension update script before some version), that's a real reason to
drop support for installing on the affected older majors, not just
document around it.

### Terminology: upgrade vs. update

"Upgrade" refers to a PostgreSQL cluster (`pg_upgrade`); "update" refers
to an extension (`ALTER EXTENSION ... UPDATE`). An extension's
version-to-version scripts are "update scripts" — never "upgrade
scripts."

### `pg_upgrade` safety vs. the common case

Any lock, guard, repair mechanism, or other logic that assumes or detects
"did a `pg_upgrade` just happen" must be correct under the assumption that
a client can connect and start issuing calls the instant the upgraded
cluster starts accepting connections — nothing about `pg_upgrade` prevents
that, so it's the only assumption that's actually safe to design for.

That correctness requirement doesn't mean concurrent access during the
upgrade window is the case to design *around*. In practice `pg_upgrade` is
run during a maintenance window with the application already stopped, so
there normally is no concurrent access at all — the immediate-access case
is the safety net, not the expected situation. This changes how a
trade-off should be weighed: a mechanism that only pays an extra-work cost
in the rare event a client connects immediately is a reasonable design; a
mechanism that pays that same cost on every upgrade because it was built
as if concurrent access were the normal case is not.

## Session state in create/update scripts must be reverted explicitly

A `CREATE EXTENSION`/`ALTER EXTENSION ... UPDATE` script can't assume
it's alone in its transaction, so it can't rely on `SET LOCAL` to undo a
session-state change when the script ends. Confirmed by direct testing:
`SET LOCAL` reverts at the end of the *enclosing transaction*, not the
script — if something else runs afterward in that same transaction
(another extension installed in the same transaction, a migration tool
batching several DDL statements), it inherits the change.

If a script needs to change session state (`search_path`,
`statement_timeout`, etc.), save the prior value and set it back
explicitly before the script ends — don't lean on `SET LOCAL`'s revert
to do that. (`client_min_messages` is the one exception: Postgres itself
already manages it for the duration of an install/update script — see
"Don't set `client_min_messages` inside an extension install script" in
`CODE_STYLE.md`.)

## RAISE: follow Postgres's own error-message style guide

Errors raised to a user should follow the PostgreSQL project's own
[error style guide](https://www.postgresql.org/docs/current/error-style-guide.html)
— this is a Postgres extension, and its errors should read like one of
Postgres's own:

- Primary message: a short, factual phrase — no trailing period,
  lowercase unless starting with a proper noun/quoted identifier, no
  embedded newlines. Elaboration goes in `DETAIL`, an actionable next
  step in `HINT`.
- Quote substituted object names/values (`%I`/`%L`) so user data is
  distinguishable from the message's own wording.
- "cannot" = never possible (a fixed limitation); "could not" = this
  attempt failed. Don't use "cannot" for a transient/environmental
  failure.
- Match the `RAISE` level to actual severity — only `EXCEPTION` aborts
  the (sub)transaction.
- Assign a real `ERRCODE` when one of Postgres's existing codes fits,
  rather than leaving every raised error at the generic default.
