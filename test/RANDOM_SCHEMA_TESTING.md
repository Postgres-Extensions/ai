# Random-schema testing for non-fixed-schema extensions

This doc covers two things beyond the universal search_path-isolation
rule (`CLAUDE.md`, this same directory), both specific to extensions that
are *not* pinned to a fixed schema via `schema=` in their `.control`
file.

## Catching hardcoded schema references

An extension with no `schema=` line is *supposed* to be installable
anywhere the caller chooses — but the only way to prove its own SQL
doesn't secretly hardcode a specific schema name (`CREATE SCHEMA foo`,
an unqualified reference that happens to only work in one particular
schema, etc.) is to actually install it somewhere unpredictable and
confirm it still works.

**Generate a randomly-named schema for the install, rather than using a
fixed pair of test schema names.** A fixed, always-the-same test schema
name can coincidentally match (or simply never expose) a hardcoded
reference — the extension's own code and the test schema name drift
into agreement over time precisely because the same name is used every
run. A freshly randomized name each run gives a hardcoded reference
nowhere to hide.

- Include a name that requires SQL identifier quoting (mixed case, or a
  space) in at least one run — an unquoted reference silently folds to
  lowercase, which would mask exactly the kind of missing-quote bug this
  is meant to catch.
- Target the schema with `CREATE EXTENSION ... SCHEMA <name>`, never by
  mutating `search_path` first — mutating it first would let the install
  succeed via a coincidentally-arranged path, masking exactly the kind of
  bug this testing exists to catch.

## Relocatable extensions: verify relocation actually works

A `relocatable = true` extension has one more wrinkle beyond ordinary
schema-flexibility: you also need to prove `ALTER EXTENSION ... SET
SCHEMA` actually works on it, not just that it can be *installed*
anywhere.

The thorough version of this is its own CI dimension: install into a
first schema, run the full suite, relocate the extension, run the full
suite again. That's probably overkill for most repos.

A cheaper alternative that still catches the same class of bug: create
the extension in one randomly-named schema, then `ALTER EXTENSION ...
SET SCHEMA` it into a *second*, also randomly-named schema, and run the
suite once against that final state. Randomizing both names (not just
using two fixed ones) rules out a hardcoded reference that happens to
survive because it matches one of two predictable names.

Before assuming this pattern is worth building at all, check what the
extension's own install script actually does: an extension marked
`relocatable = true` that doesn't create any schema-dependent objects of
its own (e.g. it only calls into another, separately-fixed extension's
API rather than creating its own tables/functions) may not need
relocation testing in the first place.
