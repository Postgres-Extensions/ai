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
`client_min_messages` see "Avoid setting `client_min_messages` inside an
extension install script" in `CODE_STYLE.md`.

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
