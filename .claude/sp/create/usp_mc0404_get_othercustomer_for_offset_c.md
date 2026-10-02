# Create Stored Procedure: `usp_mc0404_get_othercustomer_for_offset`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `VARCHAR(40)` |
| `@calculate_date` | `VARCHAR(8)` |
| `@customer_cd` | `VARCHAR(20)` |
| `@customer_flag` | `INT` |
| `@offset_flag` | `INT` |
| `@update_by` | `VARCHAR(20)` |

## Requirement (initial)

ดึงข้อมูล other-customer header (`dbo.trn_mc_othercustomer_header`) ที่ตรงกับ `@process_key` /
`@calculate_date` / `@customer_cd` / `@customer_flag` / `@offset_flag` เพื่อนำไปทำ offset

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_mc0404_get_othercustomer_for_offset', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_mc0404_get_othercustomer_for_offset
GO

CREATE PROCEDURE [dbo].[usp_mc0404_get_othercustomer_for_offset]
    @process_key    VARCHAR(40),
    @calculate_date VARCHAR(8),
    @customer_cd    VARCHAR(20),
    @customer_flag  INT,
    @offset_flag    INT,
    @update_by      VARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;

    -- other-customer header rows for offset
    SELECT h.*
    FROM dbo.trn_mc_othercustomer_header h
    WHERE h.process_key    = @process_key
      AND h.calculate_date = @calculate_date
      AND h.cust_cd        = @customer_cd
      AND h.customer_flag  = @customer_flag
      AND h.offset_flag    = @offset_flag;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_mc0404_get_othercustomer_for_offset
    @process_key    = 'TEST_KEY',
    @calculate_date = '20261001',
    @customer_cd    = 'C0001',
    @customer_flag  = 1,
    @offset_flag    = 1,
    @update_by      = 'kosin';
```
