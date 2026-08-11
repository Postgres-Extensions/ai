# RELEASE.md

The shared release process for Postgres-Extensions repos that distribute a
PGXN extension and build on `pgxntool` (`make tag`/`make dist`,
`META.in.json`, `<extension>.control`). Written to be followed standalone.

**Does not apply to:** non-PGXN tooling repos (`linter`, this `ai` repo
itself — neither has a version/release concept at all), or to
`pgxntool-test`, whose "release" is a different, paired mechanism tied to
`pgxntool`'s own release (see `pgxntool-test/.claude/skills/release/SKILL.md`
if you land there expecting this process — it doesn't apply).

**Only keep a local `RELEASE.md` in a consuming repo if there's a
legitimate, repo-specific need for one** — a real gotcha discovered the hard
way, a genuine historical exception, a deviation from the steps below that
actually affects how that repo is released. A repo with nothing like that
doesn't need a `RELEASE.md` at all; don't add one just for symmetry with
other repos, and don't copy these steps into a local file "for
completeness" — copies drift (see "Notes / gotchas" at the bottom — several
were found independently, in the same words, in more than one repo's local
copy before this doc existed). If a local file does exist for a genuine
reason, keep it to just that reason and link here for everything else.

## Overview

Releases use pgxntool's built-in `make tag`/`make dist` to produce a `.zip`,
which is then uploaded to PGXN Manager **by hand** — there is currently no
CI automation for publishing anywhere in the org. See "Future: CI
automation" below.

## Ongoing development (every PR, between releases)

Keep the next release ready to cut at any time, so cutting one is just
renaming things (step 4 below), not writing a changelog or an update path
from scratch under time pressure:

- If a PR makes a user-facing change (bug fix, behavior change, new/changed
  function — CI config, contributor-only docs, and other internal-only
  changes don't count), add an entry to the repo's changelog file
  (`HISTORY.asc` or `HISTORY.md`, depending on the repo) at the repo root:
  - If the top section is already headed `STABLE`/`stable`, add your entry
    to it.
  - If the top section is a real version number (nothing has changed since
    the last release yet), insert a new `STABLE`/`stable` section above it
    with your entry.
- If a PR changes an extension's SQL, also maintain the update script from
  the last released version to `stable`:
  `sql/<ext>--<last-released-version>--stable.sql[.in]`. Create it if it
  doesn't exist yet (first SQL-touching PR since the last release). Every
  subsequent SQL-touching PR adds whatever `ALTER ...`/`CREATE OR REPLACE
  ...` statements are needed to bring an install on the last released
  version up to your changes.

This is the flip side of the `stable` pseudo-version described in
`CLAUDE.md`'s "Version-specific SQL files" — that section covers why it's
gitignored; this is the workflow that keeps it current.

## 1. Safety check: verify committed version files haven't drifted

Before anything else, confirm every committed versioned install script
still matches what that version actually shipped:

- [ ] For each committed versioned install script, find its last-touching
      commit: `git log -1 --format='%H %ad' -- sql/<ext>--<version>.sql[.in]`.
- [ ] Confirm that commit is no later than when that version was actually
      tagged/released. Once a repo is on tag-based release history, compare
      directly against the tag (`git log -1 --format='%H %ad' <version>`).
      If any releases predate tagging (tracked some other way, or not
      tracked at all), fall back to the extension's PGXN.org listing —
      remembering that it lists the *distribution* version, which can lag
      or differ from the *extension* version if this repo distinguishes the
      two (see step 3).
- [ ] A version file touched by a commit **later** than its own release is
      a red flag — it likely means `default_version` was left pointing at
      that real (non-`stable`) version after release, and a later source
      edit on master silently regenerated (corrupted) it. Investigate
      before proceeding.
- [ ] **Known exception, not necessarily a corruption:** a version file
      whose last-touching commit is much later than its real release can
      also mean it was legitimately backfilled or reformatted after the
      fact (e.g. a newer pgxntool version started requiring something that
      wasn't tracked before). A late touch-date alone isn't suspicious —
      only worry about a file whose *content* looks like it might differ
      from what actually shipped.

## 2. Pre-release checks

- [ ] Open issues/PRs intended for this release reviewed, merged, or
      explicitly deferred.
- [ ] CI green on all supported PostgreSQL versions.
- [ ] Locally: `make verify-results` passes. Depending on the pgxntool
      version this repo vendors, plain `make test`/`make installcheck` may
      **not** be a reliable gate on its own — older pgxntool marks
      `installcheck` `.IGNORE`, so it can report success even when every
      `pg_regress` test failed. `verify-results` (which inspects
      `test/regression.diffs` directly) is the gate that's actually
      trustworthy regardless of vendored version; when in doubt, also
      eyeball the raw `pg_regress` output rather than trusting a green
      checkmark alone.
- [ ] **No CI dependency-override toggle is currently set.** If this repo's
      CI has a toggle to build a dependency from a git ref instead of its
      published PGXN version (typically added because the published
      version doesn't yet satisfy what this repo actually needs), it must
      be unset before you proceed. Check the toggle's *live value in CI
      config* (e.g. `ci.yml`'s `env:`), not just whether the Makefile
      *supports* the override — a normally-empty, always-present opt-in
      variable existing in the Makefile doesn't mean anything is currently
      pinned. Releasing while it's set produces a real, publishable zip
      that declares a dependency floor nothing on PGXN can actually
      satisfy — anyone who installs it gets a build failure, not the
      tested code. If it's set, stop: wait for the real dependency version
      to land on PGXN and the override to be reverted before cutting this
      release.

## 3. Decide the version and what to track

- [ ] Pick the new version (semantic versioning, unprefixed — e.g. `1.0.0`,
      never `v1.0.0`).
- [ ] Some repos track two version numbers that can differ: the
      **distribution version** (`META.in.json`'s top-level `version`, what
      PGXN.org lists a release under) and the **extension version**
      (`<ext>.control`'s `default_version`, what `CREATE EXTENSION`
      installs and `pg_extension.extversion` reports). If this repo makes
      that distinction, decide here whether the extension version needs to
      move at all, or only the distribution version does (e.g. a
      packaging/CI-only fix with no SQL changes).
- [ ] Decide whether to commit the generated versioned install script for
      this release. Default to committing it — for a small extension the
      storage cost is negligible, and it's the only thing that makes "can I
      install any prior version and `ALTER EXTENSION UPDATE` from it"
      testable. Only skip it for a repo that's deliberately chosen not to
      track these (check this repo's own notes) or for a truly trivial
      change where that coverage isn't worth even the small cost. The
      **update script** (`sql/<ext>--<prev>--<version>.sql[.in]`) is always
      committed regardless — without it there is no testable update path at
      all.

## 4. Update version + changelog

> **⚠️ CRITICAL — you are temporarily leaving the `stable` pseudo-version.**
> Master's `default_version` normally sits at the literal string `stable`,
> so ordinary source edits regenerate the current install script and never
> touch a frozen, already-shipped version's file. Stamping a real version
> here points that same generation rule at the real version's file instead.
> The moment this release is merged you **must** flip `default_version`
> back to `stable` (step 7). If you forget, the next source edit on master
> will regenerate — and corrupt — the just-released version's install file.

- [ ] If the extension version is moving, bump `default_version` in
      `<ext>.control` by hand. If this repo distinguishes distribution vs.
      extension version and only the distribution version is moving, leave
      `default_version` alone.
- [ ] Bump the version in `META.in.json` — the top-level `version` (always,
      the distribution version) and the extension's entry under `provides`
      (only if it's moving). Do **not** touch `meta-spec.version` — that's
      the PGXN metadata spec version, always `1.0.0` regardless of this
      distribution's version. `META.json`, `control.mk`, and `meta.mk`
      (which feeds `PGXNVERSION`) regenerate via `make` — never hand-edit
      `META.json` directly.
- [ ] Advance `release_status` in `META.in.json` as appropriate (`unstable`
      → `testing` → `stable`).
- [ ] If the extension version moved: finish the update script from the
      previous version to the new one; confirm `ALTER EXTENSION ... UPDATE`
      actually reaches the new version from the previous one, on multiple
      PostgreSQL majors.
- [ ] Rename the changelog's top `STABLE`/`stable` heading to the new
      (distribution) version number, matching its dashes/underline length
      if the format uses one.
- [ ] `git mv` the `--stable` update script to
      `sql/<ext>--<last-released-version>--<new-version>.sql[.in]`.
- [ ] Commit the version bump + changelog + renamed update script together
      in one commit.

## 5. Verify

- [ ] `make verify-results` green (runs the suite, then gates on the
      results).
- [ ] From a clean checkout (or `git archive` of the tag once cut):
      `make && make install` regenerates and installs cleanly, and creating
      the extension reports the expected version — confirms a PGXN
      consumer can build from tracked sources alone. This mirrors what
      `make dist` actually ships, since it archives the tag: committed
      files only.

## 6. Tag and distribute

- [ ] Working tree must be clean — `make tag` aborts with "Untracked
      changes!" on a dirty tree.
- [ ] Make sure `origin` in your checkout points at the canonical upstream
      repo (`Postgres-Extensions/<repo>`), **not** a personal fork —
      `make tag`/`make dist` push to `origin` by default, and a tag pushed
      to a fork does nothing for PGXN. If you work from a fork, pass
      `PGXN_REMOTE=<remote-name>` to every target below instead (added in
      pgxntool 2.2.0 after exactly this mistake shipped a tag to a fork on
      a real release — check `git remote -v` rather than assuming the
      canonical remote is named `upstream`).
- [ ] `make tag` — creates a git tag named exactly the (distribution)
      version, **unprefixed** (e.g. `1.0.0`, never `v1.0.0`), taken from
      `PGXNVERSION`, and pushes it to `origin` (or `$(PGXN_REMOTE)`).
      Idempotent if the tag already points at HEAD; errors if it exists on
      a different commit. To move an existing tag: `make forcetag`
      (= `make rmtag` + `make tag`). Don't do this once the version has
      been public for a while — moving a published tag out from under
      people is disruptive. `make rmtag` deletes the tag locally and on the
      remote.
- [ ] `make dist` — depends on `tag` (and builds HTML docs), then `git
      archive`s the tag into `../<dist-name>-<version>.zip` in the parent
      directory. Because it archives the tag, only committed files are
      included — if `.gitattributes` exists it must be committed, or `dist`
      aborts (`git archive` only honors `export-ignore` for committed
      files). `make forcedist` = `forcetag` + `dist`.

## 7. Upload to PGXN (manual)

- [ ] Upload the resulting zip at <https://manager.pgxn.org/> (log in with
      an account that has release rights to this distribution).
- [ ] Verify the new version shows up at `https://pgxn.org/dist/<name>/`
      (indexing can take a few minutes).

## 8. Return master to `stable` (CRITICAL — do not skip if the extension version moved)

- [ ] As soon as the release is merged, flip `default_version` back to
      `stable` in `<ext>.control`, open a fresh top `STABLE`/`stable`
      section in the changelog, and re-seed a
      `sql/<ext>--<this-release>--stable.sql[.in]` update script
      (content-identical to the source at this point — it exists purely so
      the update path to `stable` is always available) for the next cycle.
      Leaving master stamped at the real version means the next source edit
      regenerates and corrupts the just-released install file.
- [ ] Keep this PR's description small — "Reset version back to `stable`
      after release." is enough; it's a mechanical, low-risk step that
      doesn't need the rationale a real content change would.

## Future: CI automation

Right now this is entirely manual across the org. The `pgtap` extension (a
sibling project, not part of Postgres-Extensions) has a
`.github/workflows/release.yml` that auto-publishes to PGXN on tag push
using the `pgxn/pgxn-tools` Docker image, and auto-creates a GitHub release
from the changelog. Adopting the same pattern here would remove step 7
above, but requires storing PGXN Manager credentials as a secret in every
repo that adopts it — not yet done anywhere in the org.

## Notes / gotchas worth knowing before you start

- `make tag`/`make dist` create a *real* git tag, despite older
  `pgxntool/README.asc` wording describing the result as a "branch" — check
  the vendored version's actual behavior if a repo's docs still say
  "branch."
- CI passing is **not** proof the test suite passed on every vendored
  pgxntool version — see step 2. This has bitten real PRs (every
  `pg_regress` test failing while CI reported success).
- A CI dependency-override toggle (build a dependency from a git ref
  instead of its published PGXN version) must be unset before cutting a
  release — see step 2 above.
