# Update Stored Procedure: `usp_kosintest`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_code` | `NVARCHAR(5)` |
| `@updateby` | `NVARCHAR(10)` |
| `@updatedate` | `date` |

## รายละเอียดการแก้ไข

<ผู้ใช้แก้ส่วนนี้ทุกครั้งที่มี requirement ใหม่ — อธิบายสิ่งที่เปลี่ยน>
ให้ SELECT 'kosin' as test2 ออกมา

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_kosintest]
    @process_code   NVARCHAR(5),
    @updateby       NVARCHAR(10),
    @updatedate     DATE
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 'kosin' AS test2;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_kosintest
     @process_code = 'MC01',
     @updateby     = 'kosin',
     @updatedate   = '2026-09-10';
```
