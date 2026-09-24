# Create Stored Procedure: `usp_MC0302_get_othercustomer_summary_data`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(40)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `NVARCHAR(30)` |

## Requirement (initial)

ดึงข้อมูลสรุปของลูกค้าอื่น (other customer) ตาม `@process_key` / `@process_code` — ยังไม่ได้ระบุรายละเอียด (placeholder)

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_MC0302_get_othercustomer_summary_data', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_MC0302_get_othercustomer_summary_data
GO

CREATE PROCEDURE [dbo].[usp_MC0302_get_othercustomer_summary_data]
    @process_key  NVARCHAR(40),
    @process_code NVARCHAR(5),
    @update_by    NVARCHAR(30)
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
EXEC dbo.usp_MC0302_get_othercustomer_summary_data
    @process_key  = N'TEST_KEY',
    @process_code = N'MC030',
    @update_by    = N'kosin';
```
