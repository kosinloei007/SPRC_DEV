# Create Stored Procedure: `usp_mc0403_get_othercustomer_not_clear`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## Requirement (initial)

ดึงข้อมูล other customer ที่ยังไม่ clear ตาม `@process_key` / `@process_code` (รายละเอียด logic จะระบุภายหลังใน `_u.md`)

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_mc0403_get_othercustomer_not_clear', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_mc0403_get_othercustomer_not_clear
GO

CREATE PROCEDURE [dbo].[usp_mc0403_get_othercustomer_not_clear]
    @process_key  NVARCHAR(50),
    @process_code NVARCHAR(5),
    @update_by    VARCHAR(30)
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
EXEC dbo.usp_mc0403_get_othercustomer_not_clear
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';
```
