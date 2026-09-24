---
description: Run / create the usp_MC0302_get_othercustomer_summary_data stored procedure against RPA_DEV
argument-hint: "[run | create]   (default: run)"
allowed-tools: Read, Glob, Bash, PowerShell
---

## Task

Apply the `usp_MC0302_get_othercustomer_summary_data` stored-procedure spec to the `RPA_DEV` database, following
the execution procedure defined in `.claude/commands/run-sp.md` (extract the
`## Script` fence plus the `## ตัวอย่างการเรียกใช้` fence, write to a scratchpad
`.sql`, run with `sqlcmd -b -i`, stop on the first error).

## Target

- Procedure : `dbo.usp_MC0302_get_othercustomer_summary_data`
- Create spec : `.claude/sp/create/usp_MC0302_get_othercustomer_summary_data_c.md`
- Update spec : `.claude/sp/update/usp_MC0302_get_othercustomer_summary_data_u.md`

## Which file to run

- `$ARGUMENTS` = `create` → run the **create spec** (`_c.md`).
- `$ARGUMENTS` = `run` or empty → run the **update spec** (`_u.md`).

## After running

Show the current definition (fill `<server>`, `<login>`, `<password>` from `.claude/docs/database_connect.md`):

```
sqlcmd -S "<server>" -U <login> -P "<password>" -d RPA_DEV -y 0 -Q "SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.usp_MC0302_get_othercustomer_summary_data'))"
```
