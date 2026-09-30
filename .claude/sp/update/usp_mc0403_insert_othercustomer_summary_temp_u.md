# Update Stored Procedure: `usp_mc0403_insert_othercustomer_summary_temp`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## รายละเอียดการแก้ไข

- ย้าย logic มาจาก `usp_mc0403_insert_othercustomer_summary` ทั้งหมด orchestrator
  `usp_mc0403_get_othercustomer_not_clear` (STEP 3) เรียก stored นี้แทน stored เดิมแล้ว
- ใช้ temp table `#tbl_trx_mc_raw_fbl5n` / `#tbl_trn_mc_othercustomer_summary`
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
  - `count_debit_all_type` = `COUNT(DISTINCT document_type)` ที่ `document_type` อยู่ใน
    `cfg_predefine_det` `predefine_cd = 2002` (ปัจจุบัน DB, DR, RV)
  - `count_credit_all_type` = `COUNT(DISTINCT document_type)` ที่ `document_type` อยู่ใน
    `cfg_predefine_det` `predefine_cd = 2003` (ปัจจุบัน DA, DG, DZ)
  - `total_credit_amt` = `SUM(amount_in_doc_currency)` ต่อ `process_key, process_code, account`
    เฉพาะ `document_type` ใน `cfg_predefine_det` `predefine_cd = 2003`
    และ `arrears_after_net_due_date` >= 0 (ไม่มีรายการ → `0`)
  - `total_debit_amt` = `SUM(amount_in_doc_currency)` ต่อ `process_key, process_code, account`
    เฉพาะ `document_type` ใน `cfg_predefine_det` `predefine_cd = 2002`
    และ `arrears_after_net_due_date` < 0 (ไม่มีรายการ → `0`)
  - `count_due_date` = `COUNT` รายการที่ `arrears_after_net_due_date` >= 0
    ต่อ `process_key, process_code, account`
  - `count_notyetdue_date` = `COUNT` รายการที่ `arrears_after_net_due_date` < 0
    ต่อ `process_key, process_code, account` (ไม่ทับกับ `count_due_date` ที่ >= 0)
  - `count_debit_offset_type` = ค่าเดียวกับ `count_debit_all_type`
  - `count_credit_offset_type` = จำนวน `document_type` (ไม่ซ้ำ) ใน `predefine_cd = 2003`
    ต่อ `process_key, process_code, account` ยกเว้นเจอ **DG ตัวเดียว → 0**
    - DG → 0, DA → 1, DZ → 1
    - DA+DZ → 2, DA+DG → 2, DZ+DG → 2
    - DA+DZ+DG → 3
  - `notyetdue_minimum_amt` = `MIN(amount_in_doc_currency)` ของรายการที่
    `arrears_after_net_due_date` < 0 ต่อ `process_key, process_code, account`
    — ถ้า account นั้นไม่มีรายการ < 0 เลย ลง `0` (คอลัมน์เป็น `NOT NULL`)
  - `customer_flag` = `1` ถ้าเข้าเงื่อนไขใดเงื่อนไขหนึ่ง นอกนั้น `0`
    (คำนวณด้วย `UPDATE` หลัง INSERT เพราะต้องใช้ค่าที่ aggregate แล้ว)
    1. `total_credit_amt < 0 AND total_debit_amt > 0 AND count_notyetdue_date = 0`
    2. `total_credit_amt < 0 AND total_debit_amt > 0 AND count_notyetdue_date <> 0
       AND (notyetdue_minimum_amt + total_credit_amt) >= 0`
  - `offset_flag` = `1` ถ้าเข้าเงื่อนไขใดเงื่อนไขหนึ่ง นอกนั้น `0`
    (คำนวณด้วย `UPDATE` หลังจากคำนวณ `customer_flag` แล้ว)
    1. `customer_flag = 1`
    2. `total_credit_amt < 0 AND total_debit_amt > 0 AND count_notyetdue_date > 0`
  - `total_minimum_amt` = `0` (ค่าเริ่มต้น รอ requirement)
  - `create_date` = `GETDATE()`, `create_by` = `@update_by`

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0403_insert_othercustomer_summary_temp]
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
           COUNT(DISTINCT CASE WHEN r.is_debit_type  = 1 THEN r.document_type END) AS count_debit_all_type,
           COUNT(DISTINCT CASE WHEN r.is_credit_type = 1 THEN r.document_type END) AS count_credit_all_type,
           -- distinct credit types (2003); DG alone does not count -> 0
           CASE WHEN COUNT(DISTINCT CASE WHEN r.is_credit_type = 1 THEN r.document_type END) = 1
                 AND MAX(CASE WHEN r.is_credit_type = 1 AND r.document_type = 'DG' THEN 1 ELSE 0 END) = 1
                THEN 0
                ELSE COUNT(DISTINCT CASE WHEN r.is_credit_type = 1 THEN r.document_type END)
           END                                                      AS count_credit_offset_type,
           COUNT(DISTINCT CASE WHEN r.is_debit_type  = 1 THEN r.document_type END) AS count_debit_offset_type,
           COUNT(CASE WHEN r.arrears >= 0 THEN 1 END)               AS count_due_date,
           COUNT(CASE WHEN r.arrears < 0 THEN 1 END)                AS count_notyetdue_date,
           -- min amount among not-yet-due rows; 0 when the account has none (column is NOT NULL)
           ISNULL(CAST(MIN(CASE WHEN r.arrears < 0 THEN r.amt END) AS DECIMAL(18, 2)), 0) AS notyetdue_minimum_amt,
           0                                                        AS total_minimum_amt,
           -- credit document types (2003) that are due (arrears >= 0)
           CAST(SUM(CASE WHEN r.is_credit_type = 1 AND r.arrears >= 0 THEN r.amt ELSE 0 END) AS DECIMAL(18, 2)) AS total_credit_amt,
           -- debit document types (2002) that are not yet due (arrears < 0)
           CAST(SUM(CASE WHEN r.is_debit_type = 1 AND r.arrears < 0 THEN r.amt ELSE 0 END) AS DECIMAL(18, 2)) AS total_debit_amt,
           0                                                        AS customer_flag,
           0                                                        AS offset_flag,
           GETDATE()                                                AS create_date,
           @update_by                                               AS create_by
    FROM (
        SELECT process_key,
               process_code,
               account,
               cust_name,
               document_type,
               -- debit document types: cfg_predefine_det predefine_cd = 2002
               CASE WHEN document_type IN (SELECT predefine_key FROM dbo.cfg_predefine_det WHERE predefine_cd = 2002)
                    THEN 1 ELSE 0 END                     AS is_debit_type,
               -- credit document types: cfg_predefine_det predefine_cd = 2003
               CASE WHEN document_type IN (SELECT predefine_key FROM dbo.cfg_predefine_det WHERE predefine_cd = 2003)
                    THEN 1 ELSE 0 END                     AS is_credit_type,
               TRY_CAST(amount_in_doc_currency AS FLOAT)   AS amt,
               TRY_CAST(arrears_after_net_due_date AS INT) AS arrears
        FROM #tbl_trx_mc_raw_fbl5n
        WHERE process_key  = @process_key
          AND process_code = @process_code
    ) r
    GROUP BY r.process_key,
             r.process_code,
             r.account;

    -- customer_flag: needs the aggregated totals above
    UPDATE s
    SET customer_flag =
            CASE
                WHEN s.total_credit_amt < 0
                 AND s.total_debit_amt > 0
                 AND s.count_notyetdue_date = 0
                    THEN 1
                WHEN s.total_credit_amt < 0
                 AND s.total_debit_amt > 0
                 AND s.count_notyetdue_date <> 0
                 AND (s.notyetdue_minimum_amt + s.total_credit_amt) >= 0
                    THEN 1
                ELSE 0
            END
    FROM #tbl_trn_mc_othercustomer_summary s
    WHERE s.process_key  = @process_key
      AND s.process_code = @process_code;

    -- offset_flag: needs customer_flag set above
    UPDATE s
    SET offset_flag =
            CASE
                WHEN s.customer_flag = 1
                    THEN 1
                WHEN s.total_credit_amt < 0
                 AND s.total_debit_amt > 0
                 AND s.count_notyetdue_date > 0
                    THEN 1
                ELSE 0
            END
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

SELECT * FROM #tbl_trn_mc_othercustomer_summary;
```
