---
description: Run / create the usp_bp0302_delete_billpayment_working stored procedure against RPA_DEV
argument-hint: "[run | create]   (default: run)"
allowed-tools: Read, Glob, Bash, PowerShell
---

## Task

Apply the `usp_bp0302_delete_billpayment_working` stored-procedure spec to the `RPA_DEV` database, following
the execution procedure defined in `.claude/commands/run-sp.md` (extract the
`## Script` fence plus the `## ตัวอย่างการเรียกใช้` fence, write to a scratchpad
`.sql`, run with `sqlcmd -b -i`, stop on the first error).

## Target

- Procedure : `dbo.usp_bp0302_delete_billpayment_working`
- Create spec : `.claude/sp/create/usp_bp0302_delete_billpayment_working_c.md`
- Update spec : `.claude/sp/update/usp_bp0302_delete_billpayment_working_u.md`

## Which file to run

- `$ARGUMENTS` = `create` → run the **create spec** (`_c.md`).
- `$ARGUMENTS` = `run` or empty → run the **update spec** (`_u.md`).

## After running

Show the current definition (fill `<server>`, `<login>`, `<password>` from `.claude/docs/database_connect.md`):

```
sqlcmd -S "<server>" -U <login> -P "<password>" -d RPA_DEV -y 0 -Q "SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.usp_bp0302_delete_billpayment_working'))"
```
