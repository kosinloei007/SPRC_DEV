---
description: Run / create the usp_kosintest stored procedure against RPA_DEV
argument-hint: "[run | create]   (default: run)"
allowed-tools: Read, Glob, Bash, PowerShell
---

## Task

Apply the `usp_kosintest` stored-procedure spec to the `RPA_DEV` database, following
the execution procedure defined in `.claude/commands/run-sp.md` (extract the
`## Script` fence plus the `## ตัวอย่างการเรียกใช้` fence, write to a scratchpad
`.sql`, run with `sqlcmd -b -i`, stop on the first error).

## Target

- Procedure : `dbo.usp_kosintest`
- Create spec : `.claude/sp/create/usp_kosintest_c.md`
- Update spec : `.claude/sp/update/usp_kosintest_u.md`

## Which file to run

- `$ARGUMENTS` = `create` → run the **create spec** (`_c.md`).
- `$ARGUMENTS` = `run` or empty → run the **update spec** (`_u.md`).

## After running

Show the current definition:

```
sqlcmd -S "DESKTOP-785IB33\MSSQLSERVER2017" -U amulet_dev -P "P@ssw0rd" -d RPA_DEV -y 0 -Q "SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.usp_kosintest'))"
```
