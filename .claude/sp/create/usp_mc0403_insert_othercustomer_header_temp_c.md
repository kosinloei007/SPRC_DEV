# Create Stored Procedure: `usp_mc0403_insert_othercustomer_header_temp`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## Requirement (initial)

ย้าย logic จาก `usp_mc0403_insert_othercustomer_header` มาไว้ที่ stored นี้ ซึ่งจะ group
`#tbl_trx_mc_raw_fbl5n` by `process_key, process_code, account` แล้ว insert ลง
`#tbl_trn_mc_othercustomer_header` โดย orchestrator `usp_mc0403_get_othercustomer_not_clear`
จะเรียก stored นี้แทน stored เดิม

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_mc0403_insert_othercustomer_header_temp', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_mc0403_insert_othercustomer_header_temp
GO

CREATE PROCEDURE [dbo].[usp_mc0403_insert_othercustomer_header_temp]
    @process_key  NVARCHAR(50),
    @process_code NVARCHAR(5),
    @update_by    VARCHAR(30)
AS
BEGIN
    SET NOCOUNT ON;

    -- requires temp tables created by the orchestrator
    IF OBJECT_ID('tempdb..#tbl_trx_mc_raw_fbl5n') IS NULL
       OR OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_header') IS NULL
    BEGIN
        RAISERROR('#tbl_trx_mc_raw_fbl5n / #tbl_trn_mc_othercustomer_header not found - call via usp_mc0403_get_othercustomer_not_clear', 16, 1);
        RETURN;
    END

    INSERT INTO #tbl_trn_mc_othercustomer_header (
        process_key, process_code, calculate_date, cust_cd, cust_name,
        customer_flag, offset_flag,
        match_clear_sap_status_cd, match_clear_sap_doc_no, match_clear_sap_message,
        error_description, create_date, create_by, update_date, update_by
    )
    SELECT r.process_key,
           r.process_code,
           CAST(GETDATE() AS DATE)     AS calculate_date,
           r.account                   AS cust_cd,
           LEFT(MAX(r.cust_name), 200) AS cust_name,
           0                           AS customer_flag,
           0                           AS offset_flag,
           NULL                        AS match_clear_sap_status_cd,
           NULL                        AS match_clear_sap_doc_no,
           NULL                        AS match_clear_sap_message,
           NULL                        AS error_description,
           GETDATE()                   AS create_date,
           @update_by                  AS create_by,
           NULL                        AS update_date,
           NULL                        AS update_by
    FROM #tbl_trx_mc_raw_fbl5n r
    WHERE r.process_key  = @process_key
      AND r.process_code = @process_code
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

SELECT * FROM #tbl_trn_mc_othercustomer_header;
```
