# Update Stored Procedure: `usp_mc0403_insert_othercustomer_header`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## รายละเอียดการแก้ไข

- logic เดิม (group `#tbl_trx_mc_raw_fbl5n` → `#tbl_trn_mc_othercustomer_header`) ย้ายไปอยู่ที่
  `usp_mc0403_insert_othercustomer_header_temp` แล้ว
- stored นี้เปลี่ยนเป็น: เอาข้อมูลจาก `#tbl_trn_mc_othercustomer_header`
  (where `@process_key` / `@process_code`) ไป insert ลง table จริง
  `dbo.trn_mc_othercustomer_header`
- ใช้ temp table `#tbl_trn_mc_othercustomer_header` ที่ orchestrator
  `usp_mc0403_get_othercustomer_not_clear` สร้างไว้ ถ้าไม่มี (เรียกเดี่ยว ๆ) → `RAISERROR` แล้วจบ
- insert ตามคอลัมน์ของ temp table ทุกคอลัมน์ ส่วน `match_clear_date` / `match_clear_by`
  (มีเฉพาะใน table จริง) ลงเป็น `NULL`

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

    -- requires the temp table created by the orchestrator
    IF OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_header') IS NULL
    BEGIN
        RAISERROR('#tbl_trn_mc_othercustomer_header not found - call via usp_mc0403_get_othercustomer_not_clear', 16, 1);
        RETURN;
    END

    INSERT INTO dbo.trn_mc_othercustomer_header (
        process_key, process_code, calculate_date, cust_cd, cust_name,
        customer_flag, offset_flag,
        match_clear_sap_status_cd, match_clear_sap_doc_no, match_clear_sap_message,
        error_description, create_date, create_by, update_date, update_by
    )
    SELECT h.process_key,
           h.process_code,
           h.calculate_date,
           h.cust_cd,
           h.cust_name,
           h.customer_flag,
           h.offset_flag,
           h.match_clear_sap_status_cd,
           h.match_clear_sap_doc_no,
           h.match_clear_sap_message,
           h.error_description,
           h.create_date,
           h.create_by,
           h.update_date,
           h.update_by
    FROM #tbl_trn_mc_othercustomer_header h
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

EXEC dbo.usp_mc0403_get_raw_fbl5n
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

EXEC dbo.usp_mc0403_insert_othercustomer_header_temp
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

EXEC dbo.usp_mc0403_insert_othercustomer_header
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

SELECT * FROM dbo.trn_mc_othercustomer_header
WHERE process_key = 'TEST_KEY' AND process_code = N'MC04';
```
