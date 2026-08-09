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

## Terminology: upgrade vs. update

"Upgrade" refers to a PostgreSQL cluster (`pg_upgrade`); "update" refers
to an extension (`ALTER EXTENSION ... UPDATE`). An extension's
version-to-version scripts are "update scripts" — never "upgrade
scripts."
