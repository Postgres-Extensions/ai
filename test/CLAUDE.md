# Extension testing: schema/search_path isolation

## The one thing to get right

**When testing a PostgreSQL extension, its own schema(s) must be absent
from `search_path` for at least part of the test run.** This is not
optional, and it is not only for schema-flexible extensions — it applies
to *every* extension in this org, including ones permanently pinned to a
fixed schema via `schema=` in their `.control` file.

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
  alone) before running assertions.
- Don't assume the ambient/default search_path already excludes the
  extension's schema — check what it actually resolves to
  (`current_schemas(false)` or similar) rather than assuming `public` or
  any other specific value. Something upstream of your test (a
  dependency's own install step, a prior test in the same session) can
  leave the extension's schema on `search_path` without you doing it
  deliberately.

## Double-check at the *end* of the test run too

A single check at the start of the suite only proves the schema was
absent from `search_path` at that moment — it says nothing about whether
some test in between left it there (a test that runs `SET search_path`
and never resets it, a helper that creates the extension's schema and
leaves it on the path, etc.). **Add a final assertion, after the rest of
the suite has run, that re-checks the extension's schema is still absent
from `search_path`.** This is cheap insurance against exactly the kind of
mid-suite state leak that a start-of-suite-only check will never catch.
