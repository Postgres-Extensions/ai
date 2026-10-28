# Changing settings in extension install/update scripts

Inside a `CREATE EXTENSION`/`ALTER EXTENSION ... UPDATE` script, change
settings only with `SET LOCAL`, never a plain `SET`. Also read the
setting's value first and set it back with `SET LOCAL` before the script
ends; for a setting Postgres saves, that set-back is harmless.

Postgres runs each script in its own GUC nest level. It saves a few
settings, some only conditionally (`search_path` always,
`client_min_messages` only when the caller is below `WARNING`), sets
temporary values for them, and restores the saved values when the script
ends. See `execute_extension_script()` in PostgreSQL's
`src/backend/commands/extension.c` for the full list. For
`client_min_messages` see "Avoid setting `client_min_messages`" below.

- A plain `SET` of a saved setting overrides that restore, so the script's
  value leaks to the caller.
- `SET LOCAL` of a setting Postgres did not save lasts until the end of
  the *enclosing transaction*, so anything later in that transaction
  inherits it.
- A value read inside the script is Postgres's temporary one for a saved
  setting, not the caller's. Saving it and plain-`SET`ting it back leaves
  the caller with the script's temporary `search_path`.

[trunklet's test/helpers/update_with_data.sql](https://github.com/Postgres-Extensions/trunklet/blob/master/test/helpers/update_with_data.sql)
is an example test that fails if `ALTER EXTENSION ... UPDATE` leaves any of
the caller's settings changed. It doesn't catch a `SET LOCAL` of a setting
the server didn't save.

## Avoid setting `client_min_messages`

An install script (`sql/*.sql`, `sql/*.sql.in`) shouldn't `SET`/`SET LOCAL
client_min_messages` itself just to quiet its own NOTICEs (e.g. from a
`%TYPE` conversion or an intentional no-op `GRANT`). It's redundant at
best, and worse than nothing at worst.

`CREATE EXTENSION`/`ALTER EXTENSION UPDATE` already forces
`client_min_messages` (and `log_min_messages`) up to at least `WARNING`
for the duration of the script and restores the caller's setting when it
finishes. It only *raises* the level, so a caller who set something
stricter (e.g. `ERROR`) is still respected. An unconditional
`SET LOCAL ... = WARNING` instead lowers that stricter setting, and since
Postgres didn't save it (it saves `client_min_messages` only when the
caller is below `WARNING`), the `SET LOCAL` leak above applies.

This has already been fixed and regressed more than once: fixed in
[cat_tools#26](https://github.com/Postgres-Extensions/cat_tools/pull/26),
[count_nulls#4](https://github.com/Postgres-Extensions/count_nulls/pull/4),
[extension_tools#3](https://github.com/Postgres-Extensions/extension_tools/pull/3),
and [object_reference#6](https://github.com/Postgres-Extensions/object_reference/pull/6),
then reintroduced in
[object_reference#17](https://github.com/Postgres-Extensions/object_reference/pull/17)
and [test_factory#18](https://github.com/Postgres-Extensions/test_factory/pull/18),
each time by an agent that didn't know (or forgot) the removal was
intentional. A NOTICE a script can't otherwise avoid is expected: it's
what Postgres's own suppression already handles for a real
`CREATE EXTENSION`.

The exception is a test harness like `test/build/*` that `\i`'s the raw
script instead of going through `CREATE EXTENSION`: it never gets
Postgres's suppression, so it must set `client_min_messages` itself,
immediately before the `\i`, in the harness rather than the script.
