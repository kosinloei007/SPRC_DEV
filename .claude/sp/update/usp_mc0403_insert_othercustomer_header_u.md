# Update Stored Procedure: `usp_mc0403_insert_othercustomer_header`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## รายละเอียดการแก้ไข

<ผู้ใช้แก้ส่วนนี้ทุกครั้งที่มี requirement ใหม่ — อธิบายสิ่งที่เปลี่ยน>

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0403_insert_othercustomer_header]
    @process_key  NVARCHAR(50),
    @process_code NVARCHAR(5),
    @update_by    VARCHAR(30)
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
EXEC dbo.usp_mc0403_insert_othercustomer_header
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';
```
