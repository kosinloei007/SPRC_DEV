# Update Stored Procedure: `usp_mc0302_get_othercustomer_summary_data`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(40)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `NVARCHAR(30)` |

## รายละเอียดการแก้ไข

ดึงข้อมูลสรุปจาก `dbo.trn_mc_othercustomer_report` โดยกรองด้วย
`process_key = @process_key` และ `process_code = @process_code`

คอลัมน์ที่คืนค่า: `calculate_date`, `cust_cd`, `cust_name`,
`offset_total_credit_amt`, `offset_total_debit_amt`, `offset_diff_amt`,
`notoffset_total_credit_amt`, `notoffset_total_debit_amt`, `notoffset_diff_amt`,
`match_clear_sap_doc_no`, `match_clear_status_cd`, `match_clear_date`, `match_clear_by`

`match_clear_status_cd` แปลงค่าก่อนคืน: `1` → `'Y'`, `0` → `'N'`, `NULL` → `'N'`
(ค่าอื่นที่ไม่ใช่ `1` → `'N'`)

เรียงผลลัพธ์ตาม `cust_cd`

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0302_get_othercustomer_summary_data]
    @process_key  NVARCHAR(40),
    @process_code NVARCHAR(5),
    @update_by    NVARCHAR(30)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        [calculate_date],
        [cust_cd],
        [cust_name],
        [offset_total_credit_amt],
        [offset_total_debit_amt],
        [offset_diff_amt],
        [notoffset_total_credit_amt],
        [notoffset_total_debit_amt],
        [notoffset_diff_amt],
        [match_clear_sap_doc_no],
        CASE WHEN [match_clear_status_cd] = 1 THEN 'Y' ELSE 'N' END AS [match_clear_status_cd],
        [match_clear_date],
        [match_clear_by]
    FROM dbo.trn_mc_othercustomer_report
    WHERE process_key  = @process_key
      AND process_code = @process_code
    ORDER BY [cust_cd];
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_mc0302_get_othercustomer_summary_data
    @process_key  = N'TEST_KEY',
    @process_code = N'MC030',
    @update_by    = N'kosin';
```
