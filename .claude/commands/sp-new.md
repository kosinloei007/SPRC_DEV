---
description: Scaffold a new stored-procedure spec (_c.md + _u.md) and its own /sp:<topic> slash command
argument-hint: <proc_name> [topic-slug]
allowed-tools: Read, Glob, Write
---

## Task

Create a brand-new stored procedure spec set and a dedicated slash command for it.

## Arguments

`$ARGUMENTS` = `<proc_name> [topic-slug]`

1. **`proc_name`** (required) — full procedure name. Must match `^usp_[a-z0-9_]+$`.
   If missing or invalid, stop and explain the expected format.
   Then **drop a trailing `_YYYYMMDD` date segment** (a literal `_` followed by
   exactly 8 digits) if present. The trimmed name is what gets used as the
   procedure name and in every spec file name below.
   Example: `usp_cfg_get_file_info_20290909` → `usp_cfg_get_file_info`.
2. **`topic-slug`** (optional) — the name of the generated slash command.
   Default: take the trimmed `proc_name`, drop the leading `usp_`, then
   replace `_` with `-`.
   Example: `usp_cfg_get_file_info` → `cfg-get-file-info`.

## Steps

In every step below, `<proc_name>` means the **trimmed** name from Argument 1
(with any trailing `_YYYYMMDD` removed).

1. **Refuse if it already exists.** If `.claude/sp/create/<proc_name>_c.md` or
   `.claude/commands/sp/<topic-slug>.md` exists, stop and report — never overwrite.

2. **Gather the initial spec.** From the invoking message if given, otherwise ask
   the user for:
   - parameter list (name + SQL type), and
   - a one/two-line requirement describing what the proc must do.

3. **Create `.claude/sp/create/<proc_name>_c.md`** — use the `_c.md` template
   below. Fill the real parameters into the table and the `CREATE PROCEDURE`
   signature. Put a minimal but plausible body that satisfies the initial
   requirement (fall back to `SELECT 1;` only if the requirement is unclear).

4. **Create `.claude/sp/update/<proc_name>_u.md`** — use the `_u.md` template
   below. Its `## Script` block must be the same body as `_c.md` but with
   `ALTER PROCEDURE` instead of the `IF OBJECT_ID ... DROP` + `CREATE`. It must
   be runnable as-is; the user will hand-edit it later for new requirements.

5. **Create `.claude/commands/sp/<topic-slug>.md`** — use the per-topic command
   template below, with `<proc_name>` and the two spec paths baked in.

6. **Report**: list the 3 files created, state the new command name
   `/sp:<topic-slug>`, and give a one-line usage reminder:
   - `/sp:<topic-slug>` → apply the latest update (`_u.md`)
   - `/sp:<topic-slug> create` → create the proc from scratch (`_c.md`)

   Then ask whether to run `_c.md` now (do NOT run it automatically).

---

## Template — `.claude/sp/create/<proc_name>_c.md`

````markdown
# Create Stored Procedure: `<proc_name>`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@param` | `TYPE` |

## Requirement (initial)

<one or two lines describing what the proc must do>

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.<proc_name>', 'P') IS NOT NULL
    DROP PROCEDURE dbo.<proc_name>
GO

CREATE PROCEDURE [dbo].[<proc_name>]
    @param TYPE
AS
BEGIN
    SET NOCOUNT ON;

    -- initial implementation
    SELECT 1;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.<proc_name> @param = ...;
```
````

## Template — `.claude/sp/update/<proc_name>_u.md`

````markdown
# Update Stored Procedure: `<proc_name>`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@param` | `TYPE` |

## รายละเอียดการแก้ไข

<ผู้ใช้แก้ส่วนนี้ทุกครั้งที่มี requirement ใหม่ — อธิบายสิ่งที่เปลี่ยน>

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[<proc_name>]
    @param TYPE
AS
BEGIN
    SET NOCOUNT ON;

    -- current full implementation (edit this to match the latest requirement)
    SELECT 1;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.<proc_name> @param = ...;
```
````

## Template — `.claude/commands/sp/<topic-slug>.md`

````markdown
---
description: Run / create the <proc_name> stored procedure against RPA_DEV
argument-hint: "[run | create]   (default: run)"
allowed-tools: Read, Glob, Bash, PowerShell
---

## Task

Apply the `<proc_name>` stored-procedure spec to the `RPA_DEV` database, following
the execution procedure defined in `.claude/commands/run-sp.md` (extract the
`## Script` fence plus the `## ตัวอย่างการเรียกใช้` fence, write to a scratchpad
`.sql`, run with `sqlcmd -b -i`, stop on the first error).

## Target

- Procedure : `dbo.<proc_name>`
- Create spec : `.claude/sp/create/<proc_name>_c.md`
- Update spec : `.claude/sp/update/<proc_name>_u.md`

## Which file to run

- `$ARGUMENTS` = `create` → run the **create spec** (`_c.md`).
- `$ARGUMENTS` = `run` or empty → run the **update spec** (`_u.md`).

## After running

Show the current definition:

```
sqlcmd -S "DESKTOP-785IB33\MSSQLSERVER2017" -U amulet_dev -P "P@ssw0rd" -d RPA_DEV -y 0 -Q "SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.<proc_name>'))"
```
````
