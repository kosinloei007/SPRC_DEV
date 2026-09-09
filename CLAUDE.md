# SPRCSystem

## Database

This project connects to a SQL Server database. For connection details and
instructions, see [.claude/docs/database_connect.md](./.claude/docs/database_connect.md).

## Stored procedure workflow

Stored procedures are managed as versioned spec files under `.claude/sp/`:

| Path | Holds |
| --- | --- |
| `.claude/sp/create/<proc>_c.md` | initial `CREATE PROCEDURE` script + first requirement |
| `.claude/sp/update/<proc>_u.md` | the **current full** `ALTER PROCEDURE` script (hand-edited) |
| `.claude/sp/summary/<proc>_sum.txt` | human-readable summary of an existing proc |

Each spec `.md` has a `## Script` fenced `sql` block (the DDL) and a
`## ตัวอย่างการเรียกใช้` block (an `EXEC` smoke test).

`_u.md` always contains the **complete latest** `ALTER PROCEDURE` body — not
stacked migrations. When a new requirement arrives, edit `_u.md` so its
`## Script` block reflects the whole proc, then run it.

### Slash commands

| Command | Purpose |
| --- | --- |
| `/sp-new <proc_name> [topic-slug]` | scaffold `_c.md` + `_u.md` + a `/sp:<topic>` command for a new proc |
| `/sp-update <proc_name> [requirement...]` | run a proc's `_u.md` against `RPA_DEV`; if a requirement is given, edit `_u.md` first |
| `/sp:<topic>` | apply the latest update (`_u.md`) for that proc to `RPA_DEV` |
| `/sp:<topic> create` | create the proc from scratch (`_c.md`) |
| `/run-sp <spec.md> ...` | generic runner — execute any spec file(s) in order, stop on first error |

All runners execute against `RPA_DEV` via `sqlcmd`, reading credentials from
`.claude/docs/database_connect.md`.
