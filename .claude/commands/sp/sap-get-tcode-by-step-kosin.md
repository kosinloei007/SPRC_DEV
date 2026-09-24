---
description: Run / create the usp_sap_get_tcode_by_step_kosin stored procedure against RPA_DEV
argument-hint: "[run | create]   (default: run)"
allowed-tools: Read, Glob, Bash, PowerShell
---

## Task

Apply the `usp_sap_get_tcode_by_step_kosin` stored-procedure spec to the `RPA_DEV` database, following
the execution procedure defined in `.claude/commands/run-sp.md` (extract the
`## Script` fence plus the `## ตัวอย่างการเรียกใช้` fence, write to a scratchpad
`.sql`, run with `sqlcmd -b -i`, stop on the first error).

## Target

- Procedure : `dbo.usp_sap_get_tcode_by_step_kosin`
- Create spec : `.claude/sp/create/usp_sap_get_tcode_by_step_kosin_c.md`
- Update spec : `.claude/sp/update/usp_sap_get_tcode_by_step_kosin_u.md`

## Which file to run

- `$ARGUMENTS` = `create` → run the **create spec** (`_c.md`).
- `$ARGUMENTS` = `run` or empty → run the **update spec** (`_u.md`).

## After running

Show the current definition (fill `<server>`, `<login>`, `<password>` from `.claude/docs/database_connect.md`):

```
sqlcmd -S "<server>" -U <login> -P "<password>" -d RPA_DEV -y 0 -Q "SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.usp_sap_get_tcode_by_step_kosin'))"
```
