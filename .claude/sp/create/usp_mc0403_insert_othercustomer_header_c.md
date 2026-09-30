# Create Stored Procedure: `usp_mc0403_insert_othercustomer_header`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## Requirement (initial)

insert ข้อมูล header ของ other customer ตาม `@process_key` / `@process_code` (รายละเอียด logic จะระบุภายหลังใน `_u.md`)

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_mc0403_insert_othercustomer_header', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_mc0403_insert_othercustomer_header
GO

CREATE PROCEDURE [dbo].[usp_mc0403_insert_othercustomer_header]
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
EXEC dbo.usp_mc0403_insert_othercustomer_header
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';
```
