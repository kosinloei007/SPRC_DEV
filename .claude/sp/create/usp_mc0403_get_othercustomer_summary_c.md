# Create Stored Procedure: `usp_mc0403_get_othercustomer_summary`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `VARCHAR(40)` |
| `@process_code` | `VARCHAR(5)` |
| `@step_code` | `VARCHAR(10)` |
| `@update_by` | `VARCHAR(20)` |

## Requirement (initial)

ยังไม่กำหนด — สร้างโครง proc ไว้ก่อน (`SELECT 1`) แล้วค่อยเพิ่ม requirement ใน `_u.md`

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_mc0403_get_othercustomer_summary', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_mc0403_get_othercustomer_summary
GO

CREATE PROCEDURE [dbo].[usp_mc0403_get_othercustomer_summary]
    @process_key  VARCHAR(40),
    @process_code VARCHAR(5),
    @step_code    VARCHAR(10),
    @update_by    VARCHAR(20)
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
EXEC dbo.usp_mc0403_get_othercustomer_summary
    @process_key  = '1',
    @process_code = 'MC04',
    @step_code    = 'STEP01',
    @update_by    = 'kosin';
```
