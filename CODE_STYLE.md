# Code and documentation conventions

Conventions for comments, references, and terminology in code and docs
across Postgres-Extensions repos.

## Comments

- Use block comment format (`/* ... */`) for multi-line comments in SQL
  files, never `--` line comments for a multi-line explanation:
  ```sql
  /*
   * First line of comment.
   * Second line of comment.
   */
  ```
- Be concise — say the point in as few words as it needs, not as many as
  the topic could support. Humans have limited context too, not just AI.
- Explain something at the *first* place it's mentioned, not somewhere
  later that assumes context the reader hasn't reached yet. One real
  exception: you're already mid-explanation of A and need to mention B
  (which also needs its own explanation) — stopping to fully explain B
  right there often confuses the explanation of A more than deferring B to
  its own natural spot would. Use judgment.
- When pointing to another spot in the same file, say `above` or `below`
  so the reader doesn't have to search both directions.
- Attribution is fine when adapting a pattern from elsewhere ("modeled on
  X's Y script"). Justification that leans on context a reader of *this*
  repo doesn't have ("unlike \<other repo\>, which does Z for reasons
  specific to its own history") is not — every comment should stand on its
  own logic, as if arrived at independently.
- Only add a comment at a bug-fix site when the bug was caused by
  something non-obvious (a missed corner case, a subtle interaction, an
  incorrect assumption). The comment should explain what was missed, not
  label itself as a bug fix. Skip comments for obvious fixes. Never repeat
  the same comment verbatim in adjacent code — write it once and reference
  it ("same as above").

## Prefer `%TYPE` over a hardcoded type

When a function parameter, variable, or column exists to hold a copy of
another table's column value, declare it as `table.column%TYPE` instead of
hardcoding the type. This ties the declaration to the column's actual
type, so a future column type change doesn't silently create a mismatch
that a hardcoded type would miss.

Don't "clean up" an existing `%TYPE` reference by replacing it with the
literal type it currently resolves to — that's removing the exact
protection it exists to provide, not simplifying dead weight (see
[Postgres-Extensions/test_factory#18](https://github.com/Postgres-Extensions/test_factory/pull/18),
where this was done and then reverted).

## Don't set `client_min_messages` inside an extension install script

`CREATE EXTENSION`/`ALTER EXTENSION UPDATE` already forces
`client_min_messages` (and `log_min_messages`) up to at least `WARNING`
for the duration of the install/update script, restoring the caller's
original setting the moment the script finishes — confirmed directly
against `execute_extension_script()` in PostgreSQL's
`backend/commands/extension.c`, and verified empirically on PG12 and
PG17. It only *raises* the level, so a caller who set something stricter
(e.g. `ERROR`) is still respected.

Because Postgres already does this, an install script (`sql/*.sql`,
`sql/*.sql.in`) must never `SET`/`SET LOCAL client_min_messages` itself to
quiet its own NOTICEs (e.g. from a `%TYPE` conversion or an intentional
no-op `GRANT`) — it's redundant at best, and at worst *worse* than doing
nothing: an unconditional `SET LOCAL ... = WARNING` unconditionally lowers
a caller's stricter setting for the duration of the script, rather than
only raising to it like Postgres's own mechanism does.

This has already been fixed and regressed more than once — fixed in
[cat_tools#26](https://github.com/Postgres-Extensions/cat_tools/pull/26),
[count_nulls#4](https://github.com/Postgres-Extensions/count_nulls/pull/4),
[extension_tools#3](https://github.com/Postgres-Extensions/extension_tools/pull/3),
and [object_reference#6](https://github.com/Postgres-Extensions/object_reference/pull/6),
then reintroduced later in
[object_reference#17](https://github.com/Postgres-Extensions/object_reference/pull/17)
and [test_factory#18](https://github.com/Postgres-Extensions/test_factory/pull/18) —
each time by an agent that didn't know (or forgot) the removal was
intentional. If a script has a NOTICE it can't otherwise avoid, that's
expected: it's exactly what Postgres's own suppression already handles
for a real `CREATE EXTENSION`.

The one place that *does* need its own suppression is a test harness like
`test/build/*` that `\i`'s the raw script directly instead of going
through `CREATE EXTENSION` — it never gets Postgres's script-scoped
suppression, so it must set `client_min_messages` itself, immediately
before the `\i`, in the harness rather than the script.

## Closing non-indentable blocks

When closing a code block that cannot be indented to show its nesting
(SQL `\endif`, `DO $$...$$`, shell heredocs, column-0 `fi`/`esac`) AND
that block contains nested blocks, label the closer with a comment naming
which block it closes — e.g. shell `esac  # basename dispatch`, or a named
dollar-quote `DO $DO$ ... $DO$` for a DO block.

Where the language rejects a trailing comment on the closer (psql
`\endif` warns "extra argument ignored"), put the label on the line
immediately above instead, with a matching label above the opening
statement too — psql's `\if` parses the rest of its line as a boolean
expression (a trailing comment breaks parsing outright), and
`\else`/`\endif` parse it as an "extra argument": ignored, but still
printed as a warning into actual output.

```sql
-- existing-mode install
\if :is_existing
  ...
-- existing-mode install
\else
  ...
-- existing-mode install
\endif
```

Short blocks — still visible on screen alongside their `\else`/`\endif` —
don't need this.

## Nested dollar quoting

A dollar-quoted string nested inside another must use a tag distinct
from the enclosing one. Reusing the outer tag closes the outer string
right there, and the rest of what was meant to be its body gets
parsed as bare SQL instead.

This includes text inside comments in the nested body — the lexer is
just scanning for the closing tag and doesn't skip comments to find
it.

Just pick a distinct tag (e.g. `$body$` nested inside `$$`); it isn't
worth a comment explaining why the tag differs.

## References to PRs and issues in committed files

A reference to a GitHub PR or issue inside a **committed file** (code
comments, CI config comments, `CLAUDE.md`, docs) must be a full URL, e.g.
`https://github.com/Postgres-Extensions/<repo>/issues/28` — never a bare
`#28` (meaningless once the file is read outside GitHub) — **if it's
something a reader might still need to act on or look up**: an open TODO,
a workaround to revisit once another issue lands, a known limitation. For
purely historical context (explaining why past code looks the way it
does, where the issue is already resolved and there's nothing left to do),
a bare `#28` is fine — less noise, less likely to need chasing down. When
in doubt, use the full URL. This rule is about committed files only —
referencing by number is always fine in GitHub-native text (PR/issue
titles, descriptions, review comments).
