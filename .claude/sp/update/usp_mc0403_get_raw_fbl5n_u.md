# Update Stored Procedure: `usp_mc0403_get_raw_fbl5n`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## รายละเอียดการแก้ไข

- ดึงข้อมูลจาก `dbo.trx_mc_raw_fbl5n` โดย where `process_key = @process_key`
  และ `process_code = @process_code`
- คืนค่าทุก field ยกเว้น `created_date`, `created_by`, `updated_date`, `updated_by`
- เปลี่ยนจากคืน result set เป็น `INSERT INTO #tbl_trx_mc_raw_fbl5n` แทน — temp table
  ถูกสร้างโดย orchestrator `usp_mc0403_get_othercustomer_not_clear`
- ถ้าไม่มี `#tbl_trx_mc_raw_fbl5n` (เรียกเดี่ยว ๆ) → `RAISERROR` แล้วจบ

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0403_get_raw_fbl5n]
    @process_key  NVARCHAR(50),
    @process_code NVARCHAR(5),
    @update_by    VARCHAR(30)
AS
BEGIN
    SET NOCOUNT ON;

    -- requires #tbl_trx_mc_raw_fbl5n created by the orchestrator
    IF OBJECT_ID('tempdb..#tbl_trx_mc_raw_fbl5n') IS NULL
    BEGIN
        RAISERROR('#tbl_trx_mc_raw_fbl5n not found - call via usp_mc0403_get_othercustomer_not_clear', 16, 1);
        RETURN;
    END

    INSERT INTO #tbl_trx_mc_raw_fbl5n (
        process_key, process_code, cleared_openitems_flag, account, cust_name,
        reference, assignment, document_type, billing_document, document_number,
        entry_date, posting_date, document_date, arrears_after_net_due_date, net_due_date,
        special_gl_indicator, amount_in_doc_currency, local_currency,
        amount_in_doc_currency2, local_currency2, payment_block, username,
        clearing_document, clearing_date
    )
    SELECT process_key,
           process_code,
           cleared_openitems_flag,
           account,
           cust_name,
           reference,
           assignment,
           document_type,
           billing_document,
           document_number,
           entry_date,
           posting_date,
           document_date,
           arrears_after_net_due_date,
           net_due_date,
           special_gl_indicator,
           amount_in_doc_currency,
           local_currency,
           amount_in_doc_currency2,
           local_currency2,
           payment_block,
           username,
           clearing_document,
           clearing_date
    FROM dbo.trx_mc_raw_fbl5n
    WHERE process_key  = @process_key
      AND process_code = @process_code;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
-- simulate the orchestrator's temp table
SELECT TOP (0)
       process_key, process_code, cleared_openitems_flag, account, cust_name,
       reference, assignment, document_type, billing_document, document_number,
       entry_date, posting_date, document_date, arrears_after_net_due_date, net_due_date,
       special_gl_indicator, amount_in_doc_currency, local_currency,
       amount_in_doc_currency2, local_currency2, payment_block, username,
       clearing_document, clearing_date
INTO #tbl_trx_mc_raw_fbl5n
FROM dbo.trx_mc_raw_fbl5n;

EXEC dbo.usp_mc0403_get_raw_fbl5n
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

SELECT * FROM #tbl_trx_mc_raw_fbl5n;
```
