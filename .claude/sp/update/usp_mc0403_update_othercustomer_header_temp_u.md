# Update Stored Procedure: `usp_mc0403_update_othercustomer_header_temp`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## รายละเอียดการแก้ไข

- stored นี้ใช้ temp table `#tbl_trn_mc_othercustomer_header` / `#tbl_trn_mc_othercustomer_summary`
  ที่ orchestrator `usp_mc0403_get_othercustomer_not_clear` สร้างไว้
- ถ้าไม่มี temp table ใดตัวหนึ่ง (เรียกเดี่ยว ๆ) → `RAISERROR` แล้วจบ
- update `#tbl_trn_mc_othercustomer_header` (where `@process_key` / `@process_code`)
  join `#tbl_trn_mc_othercustomer_summary` ด้วย `process_key, process_code, cust_cd`
  - `customer_flag` / `offset_flag` = ค่าจาก summary
  - `update_date` = `GETDATE()`, `update_by` = `@update_by`

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0403_update_othercustomer_header_temp]
    @process_key  NVARCHAR(50),
    @process_code NVARCHAR(5),
    @update_by    VARCHAR(30)
AS
BEGIN
    SET NOCOUNT ON;

    -- requires temp tables created by the orchestrator
    IF OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_header') IS NULL
       OR OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_summary') IS NULL
    BEGIN
        RAISERROR('#tbl_trn_mc_othercustomer_header / #tbl_trn_mc_othercustomer_summary not found - call via usp_mc0403_get_othercustomer_not_clear', 16, 1);
        RETURN;
    END

    UPDATE h
    SET customer_flag = s.customer_flag,
        offset_flag   = s.offset_flag,
        update_date   = GETDATE(),
        update_by     = @update_by
    FROM #tbl_trn_mc_othercustomer_header h
    INNER JOIN #tbl_trn_mc_othercustomer_summary s
        ON  s.process_key  = h.process_key
        AND s.process_code = h.process_code
        AND s.cust_cd      = h.cust_cd
    WHERE h.process_key  = @process_key
      AND h.process_code = @process_code;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
-- simulate the orchestrator's temp tables
SELECT TOP (0)
       process_key, process_code, cleared_openitems_flag, account, cust_name,
       reference, assignment, document_type, billing_document, document_number,
       entry_date, posting_date, document_date, arrears_after_net_due_date, net_due_date,
       special_gl_indicator, amount_in_doc_currency, local_currency,
       amount_in_doc_currency2, local_currency2, payment_block, username,
       clearing_document, clearing_date
INTO #tbl_trx_mc_raw_fbl5n
FROM dbo.trx_mc_raw_fbl5n;

SELECT TOP (0) *
INTO #tbl_trn_mc_othercustomer_header
FROM dbo.trn_mc_othercustomer_header;

SELECT TOP (0) *
INTO #tbl_trn_mc_othercustomer_summary
FROM dbo.trn_mc_othercustomer_summary;

EXEC dbo.usp_mc0403_get_raw_fbl5n
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

EXEC dbo.usp_mc0403_insert_othercustomer_header_temp
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

EXEC dbo.usp_mc0403_insert_othercustomer_summary_temp
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

EXEC dbo.usp_mc0403_update_othercustomer_header_temp
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

SELECT * FROM #tbl_trn_mc_othercustomer_header;
```
