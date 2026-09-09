# Update Stored Procedure: `usp_cfg_get_file_info_20290909`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_code` | `NVARCHAR(5)` |
| `@process_key` | `NVARCHAR(100)` |

## รายละเอียดการแก้ไข

เปลี่ยน body ให้ `SELECT` ข้อมูลทุก field จากตาราง `dbo.cfg_file_information`
ตามเงื่อนไข:

- `is_active = 1`
- `ISNULL(NULLIF(cfg.process_code, ''), '') = @process_code`

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_cfg_get_file_info_20290909]
    @process_code   NVARCHAR(5),
    @process_key    NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT cfg.*
    FROM dbo.cfg_file_information cfg WITH (NOLOCK)
    WHERE cfg.is_active = 1
    AND ISNULL(NULLIF(cfg.process_code, ''), '') = @process_code;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_cfg_get_file_info_20290909
     @process_code = 'MC01',
     @process_key  = '20290909-0001';
```
