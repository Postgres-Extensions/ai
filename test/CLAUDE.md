# Extension testing: schema/search_path isolation

**When testing a PostgreSQL extension, its own schema(s) must be absent
from `search_path` for the *entire* test run — not "most of it," not
"at the start," the whole thing.** A window where the schema is
reachable is a window where an unqualified reference can resolve by
accident and go undetected; letting it back onto `search_path` at any
point defeats the purpose for whatever ran during that window. This is
not optional, and it is not only for schema-flexible extensions — it
applies to *every* extension in this org, including ones permanently
pinned to a fixed schema via `schema=` in their `.control` file.

Why this matters: if an extension's schema is reachable via
`search_path`, any unqualified reference inside the extension's own SQL
to one of its own objects (a function calling another function by bare
name, a view referencing a table with no schema prefix, etc.) resolves
successfully by accident. The test suite passes, but the extension is
carrying a latent bug — the same reference breaks the moment it's called
from a session whose `search_path` doesn't happen to include that
schema. Testing with the schema excluded from `search_path` is the only
way to catch this: it forces every one of the extension's own internal
references to prove it doesn't need help from an ambient search path.

**This holds even for a schema-pinned extension.** "Pinned" only means
the install location can't be *changed* — it says nothing about whether
the extension's own code correctly qualifies its own references. A
fixed-schema extension can carry exactly the same unqualified-reference
bug; it's just less visible, because nobody's ever going to test it in a
different location to expose it. Excluding its one fixed schema from
`search_path` during testing is still the only way to catch it. Do not
skip this check just because an extension can't be relocated.

## How to exclude the schema from search_path

The goal is the opposite of "make sure the schema is reachable" — prove
the extension's own code works when its schema is specifically
**absent** from `search_path`:

- Set `search_path` for the test session to something that does not
  contain the extension's schema (e.g. a scratch schema, or `pg_catalog`
  alone) before running assertions. If your repo's build system is
  pgxntool, its `test/install` feature (`README.asc`) — files that run
  once, committed, before the rest of the suite in the same `pg_regress`
  invocation — is the natural place to put this: set it there and it
  stays set for every test file that follows. There's more info about
  testing in pgxntool's own `CLAUDE.md`.
- Don't assume the ambient/default search_path already excludes the
  extension's schema — check what it actually resolves to
  (`current_schemas(false)` or similar) rather than assuming `public` or
  any other specific value. Something upstream of your test (a
  dependency's own install step, a prior test in the same session) can
  leave the extension's schema on `search_path` without you doing it
  deliberately.

## Verify it held for the whole run

If you explicitly set `search_path` yourself before running assertions
(as above), checking it again right after proves nothing — you already
know what you just set it to. **The check that actually matters is at
the *end*, after the rest of the suite has run: re-assert the
extension's schema is still absent from `search_path`.** That's what
catches the failure mode this section exists to guard against — not "did
my own setup work," but "did some test in between run `SET search_path`
(or otherwise put the schema back) and never reset it," silently
reintroducing the exact accidental-resolution risk this whole exercise
exists to rule out.

Ideally you'd check continuously throughout the run, not just once at
the end — but for a file-based test suite (e.g. `pg_regress`, where each
test is its own `.sql` file), injecting a check into every single file
isn't practical. The end-of-run check is the practical compromise, not
the ideal.

A cheap complement that gets closer to continuous coverage without the
per-file overhead: it's worth adding an automated test — checked in and
run as part of the suite, not a one-off manual grep — that asserts
`search_path` is mentioned in exactly two places in the suite's own
files: the file that sets it at the start, and the file that re-checks
it at the end. A third match means some other test is touching
`search_path`, which needs tracking down regardless of whether the
end-of-run check's specific assertion happens to notice its effect.
