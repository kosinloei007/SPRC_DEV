---
description: Run the SQL in one or more SP spec .md files against RPA_DEV, in order
argument-hint: "[spec.md ...]  (default: the usp_cfg_get_file_info_20290909 pair)"
allowed-tools: Read, Glob, Bash, PowerShell
---

## Task

Execute the SQL contained in SP spec `.md` files against the `RPA_DEV` database,
in the order given, stopping at the first failure.

## Files to run

`$ARGUMENTS`

If `$ARGUMENTS` is empty, run this default sequence:

1. `.claude/sp/create/usp_cfg_get_file_info_20290909_c.md`
2. `.claude/sp/update/usp_cfg_get_file_info_20290909_u.md`

## How to resolve each file argument

For every token in `$ARGUMENTS`, in order:

- If it is an absolute path or exists exactly as given → use it.
- Else resolve it against `.claude/sp/` (e.g. `create/foo_c.md` →
  `.claude/sp/create/foo_c.md`).
- Else, if it is a bare filename, search `.claude/sp/create/` then
  `.claude/sp/update/`.
- If it still cannot be found → stop and report which argument failed; run nothing.

## Connection

Read connection details from `database_connect.md` (fall back to
`.claude/docs/database_connect.md`). Do not hard-code the password. Expected:

- Server: `DESKTOP-785IB33\MSSQLSERVER2017`
- Login: `amulet_dev`
- Database: `RPA_DEV`

## Steps — for each resolved file, in order

1. `Read` the file.
2. Extract the ```` ```sql ```` fenced block under the `## Script` heading.
3. Extract the ```` ```sql ```` fenced block under the `## ตัวอย่างการเรียกใช้`
   heading (the `EXEC` smoke test). If that heading is missing, run only the
   `## Script` block.
4. Write to a scratchpad temp file `run-sp-<basename>.sql` in this order:
   the `## Script` block, then a line `GO`, then the example `EXEC` block.
5. Execute (batch-abort on error):

   ```
   sqlcmd -S "DESKTOP-785IB33\MSSQLSERVER2017" -U amulet_dev -P "P@ssw0rd" -d RPA_DEV -b -l 5 -i "<temp>.sql"
   ```

   Run it with the Bash tool. If argument parsing misbehaves, retry the exact
   same flags with the PowerShell tool.
6. Print: the file name, the sqlcmd output, and `OK` or `FAILED`.

## Rules

- Stop on the first failure — do NOT run later files if an earlier one errors.
- End with a short summary: which files ran, result of each, which were skipped.
- This command only executes SQL and reports. Do not commit anything.

## Verify after running

```
sqlcmd -S "DESKTOP-785IB33\MSSQLSERVER2017" -U amulet_dev -P "P@ssw0rd" -d RPA_DEV -y 0 -Q "SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.usp_cfg_get_file_info_20290909'))"
```
