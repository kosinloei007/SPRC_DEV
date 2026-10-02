# Update Stored Procedure: `usp_mc0403_get_othercustomer_summary`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `VARCHAR(40)` |
| `@process_code` | `VARCHAR(5)` |
| `@step_code` | `VARCHAR(10)` |
| `@update_by` | `VARCHAR(20)` |

## รายละเอียดการแก้ไข

- select ข้อมูลจาก `dbo.trn_mc_othercustomer_header`
  where `process_key = @process_key` และ `process_code = @process_code`
- column แรกเป็นค่า parameter `@step_code AS step_code`
- column ที่ 2 เป็น `cust_cd AS account`
- column ที่ 3 เป็น `clearing_date` = `calculate_date` แปลงเป็น `NVARCHAR(10)` style 104 (`DD.MM.YYYY`)
- column ที่ 4 เป็น `period` = `MONTH(calculate_date)` (INT 1–12)
- column ที่ 5 เป็น `company_code` = `config_value` จาก `dbo.cfg_configuration_process`
  where `process_code = 'BP01'` / `step_code = 'BP0107'` / `config_code = 'COMPANY_CODE'`
  (อ่านใส่ตัวแปรครั้งเดียวก่อน select; ไม่เจอ config → `NULL`)
- column ที่ 6–8 เป็นค่า fix: `currency = 'THB'`, `standard_ois = 'YES'`, `additional_selection = 1`
- column ที่ 9 เป็น `calculate_date` แปลงเป็น `NVARCHAR(8)` style 112 (`YYYYMMDD`)
- column ที่ 10 เป็น `cust_cd AS customer_cd`
- column ที่ 11 เป็น `customer_flag` ตาม `@step_code`: `'MC0404'` → `1`, `'MC0405'` → `0`, อื่น ๆ → `NULL`
- column ที่ 12 เป็น `offset_flag` fix `1`
- `@update_by` ยังไม่ได้ใช้

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0403_get_othercustomer_summary]
    @process_key  VARCHAR(40),
    @process_code VARCHAR(5),
    @step_code    VARCHAR(10),
    @update_by    VARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @company_code NVARCHAR(200);

    SELECT @company_code = c.config_value
    FROM dbo.cfg_configuration_process c
    WHERE c.process_code = N'BP01'
      AND c.step_code    = N'BP0107'
      AND c.config_code  = N'COMPANY_CODE';

    SELECT @step_code AS step_code,
           h.cust_cd AS account,
           CONVERT(NVARCHAR(10), h.calculate_date, 104) AS clearing_date,    -- DD.MM.YYYY
           MONTH(h.calculate_date) AS period,
           @company_code AS company_code,
           'THB' AS currency,
           'YES' AS standard_ois,
           1     AS additional_selection,
           CONVERT(NVARCHAR(8), h.calculate_date, 112) AS calculate_date,   -- YYYYMMDD
           h.cust_cd AS customer_cd,
           CASE @step_code
                WHEN 'MC0404' THEN 1
                WHEN 'MC0405' THEN 0
           END AS customer_flag,
           1 AS offset_flag
    FROM dbo.trn_mc_othercustomer_header h
    WHERE h.process_key  = @process_key
      AND h.process_code = @process_code;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_mc0403_get_othercustomer_summary
    @process_key  = '1',
    @process_code = 'MC04',
    @step_code    = 'MC0404',
    @update_by    = 'kosin';
```
