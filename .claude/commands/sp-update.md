---
description: Apply an update spec (_u.md) for a stored procedure to RPA_DEV, optionally editing it for a new requirement first
argument-hint: <proc_name> [requirement description...]
allowed-tools: Read, Glob, Edit, Write, Bash, PowerShell
---

## Task

Update an existing stored procedure by running its `_u.md` update spec against
the `RPA_DEV` database — optionally revising the spec first for a new requirement.

## Arguments

`$ARGUMENTS` = `<proc_name> [requirement description...]`

1. **`proc_name`** (required, first token) — the procedure name. Accept any of:
   `usp_cfg_get_file_info_20260909`, `usp_cfg_get_file_info_20260909_u`,
   `usp_cfg_get_file_info_20260909_u.md`, or a full/relative path. Normalise to
   the file `.claude/sp/update/<proc_name>_u.md`.
2. **requirement description** (optional, everything after the first token) —
   a change to apply to the spec before running it.

## Steps

1. **Resolve the update spec.**
   - Strip a trailing `.md`, then a trailing `_u`, from the first token to get
     `<proc_name>`.
   - Target file = `.claude/sp/update/<proc_name>_u.md`.
   - If it does not exist → stop and report. Suggest `/sp-new <proc_name>` if the
     procedure has never been scaffolded.

2. **If a requirement description was given:**
   - `Read` the spec.
   - Edit the `## Script` block so it is the **complete latest** `ALTER PROCEDURE`
     that satisfies the new requirement (full body — never a fragment).
   - Update `## รายละเอียดการแก้ไข` with a short note of what changed.
   - Update `## ตัวอย่างการเรียกใช้` if the parameters changed.
   - Show the user the edited `## Script` block and pause for confirmation before
     running it against the database.

   **If no requirement description was given:** assume the user already
   hand-edited `_u.md`. Proceed straight to step 3.

3. **Run the spec** following the execution procedure in
   `.claude/commands/run-sp.md`:
   - extract the `## Script` fence + the `## ตัวอย่างการเรียกใช้` fence,
   - write them to a scratchpad `.sql` (Script, then `GO`, then the example),
   - execute with
     `sqlcmd -S "DESKTOP-785IB33\MSSQLSERVER2017" -U amulet_dev -P "P@ssw0rd" -d RPA_DEV -b -l 5 -i "<temp>.sql"`
     (Bash tool; fall back to PowerShell if arg parsing misbehaves),
   - stop and report if it errors.

4. **Verify** — show the current definition:

   ```
   sqlcmd -S "DESKTOP-785IB33\MSSQLSERVER2017" -U amulet_dev -P "P@ssw0rd" -d RPA_DEV -y 0 -Q "SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.<proc_name>'))"
   ```

## Rules

- `_u.md` `## Script` must always hold the full `ALTER PROCEDURE`, not stacked
  migrations — re-running it must be idempotent.
- Never touch `_c.md` from this command.
- Do not commit anything.

## Examples

```
/sp-update usp_cfg_get_file_info_20260909
```
รัน `_u.md` ตามที่แก้ไว้แล้ว

```
/sp-update usp_cfg_get_file_info_20260909 เพิ่มเงื่อนไข filter เฉพาะ company_code = @company_code และเพิ่ม parameter @company_code NVARCHAR(5)
```
Claude แก้ `## Script` ให้ตาม requirement แล้วรัน
