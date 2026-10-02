# Create Stored Procedure: `usp_mc0405_get_othercustomer_for_offset`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `VARCHAR(40)` |
| `@process_code` | `NVARCHAR(5)` |
| `@calculate_date` | `VARCHAR(8)` |
| `@customer_cd` | `VARCHAR(20)` |
| `@customer_flag` | `INT` |
| `@offset_flag` | `INT` |
| `@update_by` | `VARCHAR(20)` |

## Requirement (initial)

ยังไม่กำหนด — สร้างโครง proc ไว้ก่อน (`SELECT 1`) แล้วค่อยเพิ่ม requirement ผ่าน `/sp-update`

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_mc0405_get_othercustomer_for_offset', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_mc0405_get_othercustomer_for_offset
GO

CREATE PROCEDURE [dbo].[usp_mc0405_get_othercustomer_for_offset]
    @process_key    VARCHAR(40),
    @process_code   NVARCHAR(5),
    @calculate_date VARCHAR(8),
    @customer_cd    VARCHAR(20),
    @customer_flag  INT,
    @offset_flag    INT,
    @update_by      VARCHAR(20)
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
EXEC dbo.usp_mc0405_get_othercustomer_for_offset
    @process_key    = '1',
    @process_code   = N'MC04',
    @calculate_date = '20261001',
    @customer_cd    = '8516784',
    @customer_flag  = 1,
    @offset_flag    = 1,
    @update_by      = 'kosin';
```
