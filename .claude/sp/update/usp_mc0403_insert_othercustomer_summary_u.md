# Update Stored Procedure: `usp_mc0403_insert_othercustomer_summary`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## รายละเอียดการแก้ไข

- logic เดิม (คำนวณ summary จาก `#tbl_trx_mc_raw_fbl5n` → `#tbl_trn_mc_othercustomer_summary`
  รวม `customer_flag` / `offset_flag`) ย้ายไปอยู่ที่ `usp_mc0403_insert_othercustomer_summary_temp` แล้ว
- stored นี้เปลี่ยนเป็น: เอาข้อมูลจาก `#tbl_trn_mc_othercustomer_summary`
  (where `@process_key` / `@process_code`) ไป insert ลง table จริง
  `dbo.trn_mc_othercustomer_summary`
- ใช้ temp table `#tbl_trn_mc_othercustomer_summary` ที่ orchestrator
  `usp_mc0403_get_othercustomer_not_clear` สร้างไว้ ถ้าไม่มี (เรียกเดี่ยว ๆ) → `RAISERROR` แล้วจบ
- ก่อน insert → `DELETE` ข้อมูลเดิมใน `dbo.trn_mc_othercustomer_summary` ของ
  `@process_key` / `@process_code` (รันซ้ำได้ ไม่ชน PK)
- insert ครบทุกคอลัมน์ (schema temp table ตรงกับ table จริง)

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0403_insert_othercustomer_summary]
    @process_key  NVARCHAR(50),
    @process_code NVARCHAR(5),
    @update_by    VARCHAR(30)
AS
BEGIN
    SET NOCOUNT ON;

    -- requires the temp table created by the orchestrator
    IF OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_summary') IS NULL
    BEGIN
        RAISERROR('#tbl_trn_mc_othercustomer_summary not found - call via usp_mc0403_get_othercustomer_not_clear', 16, 1);
        RETURN;
    END

    -- re-run safe: clear this key's rows before inserting
    DELETE FROM dbo.trn_mc_othercustomer_summary
    WHERE process_key  = @process_key
      AND process_code = @process_code;

    INSERT INTO dbo.trn_mc_othercustomer_summary (
        process_key, process_code, calculate_date, cust_cd, cust_name,
        count_debit_all_type, count_credit_all_type,
        count_credit_offset_type, count_debit_offset_type,
        count_due_date, count_notyetdue_date,
        notyetdue_minimum_amt, total_minimum_amt,
        total_credit_amt, total_debit_amt,
        customer_flag, offset_flag, create_date, create_by
    )
    SELECT s.process_key,
           s.process_code,
           s.calculate_date,
           s.cust_cd,
           s.cust_name,
           s.count_debit_all_type,
           s.count_credit_all_type,
           s.count_credit_offset_type,
           s.count_debit_offset_type,
           s.count_due_date,
           s.count_notyetdue_date,
           s.notyetdue_minimum_amt,
           s.total_minimum_amt,
           s.total_credit_amt,
           s.total_debit_amt,
           s.customer_flag,
           s.offset_flag,
           s.create_date,
           s.create_by
    FROM #tbl_trn_mc_othercustomer_summary s
    WHERE s.process_key  = @process_key
      AND s.process_code = @process_code;
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
INTO #tbl_trn_mc_othercustomer_summary
FROM dbo.trn_mc_othercustomer_summary;

EXEC dbo.usp_mc0403_get_raw_fbl5n
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

EXEC dbo.usp_mc0403_insert_othercustomer_summary_temp
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

EXEC dbo.usp_mc0403_insert_othercustomer_summary
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

SELECT * FROM dbo.trn_mc_othercustomer_summary
WHERE process_key = 'TEST_KEY' AND process_code = N'MC04';
```
