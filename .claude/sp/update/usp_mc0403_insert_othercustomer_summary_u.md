# Update Stored Procedure: `usp_mc0403_insert_othercustomer_summary`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## รายละเอียดการแก้ไข

- stored นี้ถูก call จาก orchestrator `usp_mc0403_get_othercustomer_not_clear` (STEP 3)
  และใช้ temp table `#tbl_trx_mc_raw_fbl5n` / `#tbl_trn_mc_othercustomer_summary`
  ที่ orchestrator สร้างไว้
- ถ้าไม่มี temp table ใดตัวหนึ่ง (เรียกเดี่ยว ๆ) → `RAISERROR` แล้วจบ
- select จาก `#tbl_trx_mc_raw_fbl5n` (where `@process_key` / `@process_code`)
  group by `process_key, process_code, account` แล้ว insert ลง
  `#tbl_trn_mc_othercustomer_summary`
  - `cust_cd` = `account`
  - `cust_name` = `MAX(cust_name)` จาก `#tbl_trx_mc_raw_fbl5n` ตัดเหลือ 200 ตัวอักษร
    (`trn_mc_othercustomer_summary.cust_name` เป็น `NVARCHAR(200)` แล้ว)
  - `calculate_date` = วันที่ปัจจุบัน
  - ยอดเงินใช้ `amount_in_doc_currency` แปลงผ่าน `TRY_CAST(... AS FLOAT)`
    (ข้อมูลบางแถวเป็นรูปแบบ `-1.70923e+006`) แล้วเป็น `DECIMAL(18,2)`
  - `count_debit_all_type` / `total_debit_amt` = จำนวน / ผลรวมของรายการที่ยอด > 0
  - `count_credit_all_type` / `total_credit_amt` = จำนวน / ผลรวมของรายการที่ยอด < 0
  - `count_due_date` = จำนวนรายการที่ `arrears_after_net_due_date` > 0 (เลย due แล้ว)
  - `count_notyetdue_date` = จำนวนรายการที่ `arrears_after_net_due_date` <= 0
  - `count_credit_offset_type`, `count_debit_offset_type`, `notyetdue_minimum_amt`,
    `total_minimum_amt`, `customer_flag`, `offset_flag` = `0` (ค่าเริ่มต้น รอ requirement)
  - `create_date` = `GETDATE()`, `create_by` = `@update_by`

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

    -- requires temp tables created by the orchestrator
    IF OBJECT_ID('tempdb..#tbl_trx_mc_raw_fbl5n') IS NULL
       OR OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_summary') IS NULL
    BEGIN
        RAISERROR('#tbl_trx_mc_raw_fbl5n / #tbl_trn_mc_othercustomer_summary not found - call via usp_mc0403_get_othercustomer_not_clear', 16, 1);
        RETURN;
    END

    INSERT INTO #tbl_trn_mc_othercustomer_summary (
        process_key, process_code, calculate_date, cust_cd, cust_name,
        count_debit_all_type, count_credit_all_type,
        count_credit_offset_type, count_debit_offset_type,
        count_due_date, count_notyetdue_date,
        notyetdue_minimum_amt, total_minimum_amt,
        total_credit_amt, total_debit_amt,
        customer_flag, offset_flag, create_date, create_by
    )
    SELECT r.process_key,
           r.process_code,
           CAST(GETDATE() AS DATE)                                  AS calculate_date,
           r.account                                                AS cust_cd,
           LEFT(MAX(r.cust_name), 200)                              AS cust_name,
           SUM(CASE WHEN r.amt > 0 THEN 1 ELSE 0 END)               AS count_debit_all_type,
           SUM(CASE WHEN r.amt < 0 THEN 1 ELSE 0 END)               AS count_credit_all_type,
           0                                                        AS count_credit_offset_type,
           0                                                        AS count_debit_offset_type,
           SUM(CASE WHEN r.arrears > 0 THEN 1 ELSE 0 END)           AS count_due_date,
           SUM(CASE WHEN r.arrears <= 0 THEN 1 ELSE 0 END)          AS count_notyetdue_date,
           0                                                        AS notyetdue_minimum_amt,
           0                                                        AS total_minimum_amt,
           CAST(SUM(CASE WHEN r.amt < 0 THEN r.amt ELSE 0 END) AS DECIMAL(18, 2)) AS total_credit_amt,
           CAST(SUM(CASE WHEN r.amt > 0 THEN r.amt ELSE 0 END) AS DECIMAL(18, 2)) AS total_debit_amt,
           0                                                        AS customer_flag,
           0                                                        AS offset_flag,
           GETDATE()                                                AS create_date,
           @update_by                                               AS create_by
    FROM (
        SELECT process_key,
               process_code,
               account,
               cust_name,
               TRY_CAST(amount_in_doc_currency AS FLOAT)   AS amt,
               TRY_CAST(arrears_after_net_due_date AS INT) AS arrears
        FROM #tbl_trx_mc_raw_fbl5n
        WHERE process_key  = @process_key
          AND process_code = @process_code
    ) r
    GROUP BY r.process_key,
             r.process_code,
             r.account;
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

EXEC dbo.usp_mc0403_insert_othercustomer_summary
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';

SELECT * FROM #tbl_trn_mc_othercustomer_summary;
```
