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

## Related docs in this repo

- [PR.md](./PR.md) — PR, commit, and git-process conventions (merge
  authority, fork workflow, force-push policy, etc.)
- [CODE_STYLE.md](./CODE_STYLE.md) — comment conventions, PR/issue
  reference format, terminology

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

## Shell scripts

- Use `#!/usr/bin/env bash`, never `#!/bin/bash` — the latter hardcodes
  the path and fails on systems where bash lives elsewhere (some BSDs,
  NixOS, Homebrew on macOS); `env bash` finds it on `PATH`.
- Never use `echo ""` to print a blank line; just use `echo` with no
  arguments.

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

For extensions built on pgxntool's versioned-SQL generation:

- Version-specific install scripts (`sql/<ext>--<version>.sql[.in]`) are
  tracked by default — they enable update testing (install an old
  version, `ALTER EXTENSION UPDATE`, verify) and let CI catch a
  PostgreSQL major version that unpredictably breaks installing an
  *older* extension version. See
  https://github.com/Postgres-Extensions/pgxntool/issues/51.
- Update scripts (`sql/<ext>--<v1>--<v2>.sql[.in]`) are essential to the
  update path and have no substitute — always track them.
- Version-specific files must never be edited manually — edit the base
  source and regenerate.
- A version that changes little and ships no nontrivial update-path
  machinery may omit its generated install script (regenerable from
  source at build time, little test-coverage value) — but track it
  whenever the version carries meaningful changes or a nontrivial update
  script.
- A pseudo-version a repo uses to mean "current, between releases" (e.g.
  a `stable` default_version) is the opposite case from a
  normally-tracked-but-occasionally-skipped version: it would be
  regenerated and re-diffed on every source edit if tracked, for zero
  test-coverage value. Gitignore it instead.

### PostgreSQL version support policy

Never support a fresh install on a PostgreSQL version where the
extension's update path is known to be broken — a version that can't be
updated *to* isn't truly supported. If a PostgreSQL major changes what an
update script is allowed to do (e.g. a statement that can't run inside an
extension update script before some version), that's a real reason to
drop support for installing on the affected older majors, not just
document around it.
