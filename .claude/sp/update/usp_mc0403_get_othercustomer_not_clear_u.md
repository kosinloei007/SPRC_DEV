# Update Stored Procedure: `usp_mc0403_get_othercustomer_not_clear`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `NVARCHAR(50)` |
| `@process_code` | `NVARCHAR(5)` |
| `@update_by` | `VARCHAR(30)` |

## รายละเอียดการแก้ไข

- เพิ่ม `SET XACT_ABORT ON` + `BEGIN TRY / BEGIN CATCH`
- เปิด transaction เองใน proc (`BEGIN TRANSACTION` / `COMMIT TRANSACTION`) เพื่อให้
  การแก้ไขข้อมูลทั้งหมดเป็น atomic
- ถ้าเกิด error → `ROLLBACK TRANSACTION` (ถ้ามี transaction เปิดอยู่) แล้ว
  `RAISERROR` ส่ง error message / severity / state เดิมกลับไปให้ผู้เรียก
- ปรับ stored นี้เป็น **[orchestrator]** ของ flow MC0403
- สร้าง temp table `#tbl_trx_mc_raw_fbl5n` (24 field ตาม `trx_mc_raw_fbl5n`
  ไม่รวม created_/updated_)
- สร้าง temp table `#tbl_trn_mc_othercustomer_header` (schema เดียวกับ
  `dbo.trn_mc_othercustomer_header`) และ `#tbl_trn_mc_othercustomer_summary`
  (schema เดียวกับ `dbo.trn_mc_othercustomer_summary`)
- STEP 1: call `usp_mc0403_get_raw_fbl5n` — ตัว stored นั้นเป็นคน insert ลง
  `#tbl_trx_mc_raw_fbl5n` เอง (ไม่ใช้ `INSERT INTO ... EXEC` ใน orchestrator แล้ว)
- STEP 2: call `usp_mc0403_insert_othercustomer_header` — group by
  `process_key, process_code, account` จาก `#tbl_trx_mc_raw_fbl5n` แล้ว insert ลง
  `#tbl_trn_mc_othercustomer_header`
- STEP 3: call `usp_mc0403_insert_othercustomer_summary` (ภายใน transaction
  เดียวกัน) โดย summary ใช้ข้อมูลจาก `#tbl_trx_mc_raw_fbl5n`

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0403_get_othercustomer_not_clear]
    @process_key  NVARCHAR(50),
    @process_code NVARCHAR(5),
    @update_by    VARCHAR(30)
AS
-- [orchestrator]
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    DECLARE @ErrorMessage  NVARCHAR(4000),
            @ErrorSeverity INT,
            @ErrorState    INT;

    -- temp table: same schema as the result of usp_mc0403_get_raw_fbl5n
    IF OBJECT_ID('tempdb..#tbl_trx_mc_raw_fbl5n') IS NOT NULL
        DROP TABLE #tbl_trx_mc_raw_fbl5n;

    CREATE TABLE #tbl_trx_mc_raw_fbl5n (
        process_key                NVARCHAR(50)   NOT NULL,
        process_code               NVARCHAR(5)    NOT NULL,
        cleared_openitems_flag     INT            NOT NULL,
        account                    NVARCHAR(50)   NOT NULL,
        cust_name                  NVARCHAR(1000) NOT NULL,
        reference                  NVARCHAR(200)  NULL,
        assignment                 NVARCHAR(100)  NULL,
        document_type              NVARCHAR(5)    NULL,
        billing_document           NVARCHAR(50)   NOT NULL,
        document_number            NVARCHAR(50)   NOT NULL,
        entry_date                 NVARCHAR(20)   NULL,
        posting_date               NVARCHAR(20)   NULL,
        document_date              NVARCHAR(20)   NULL,
        arrears_after_net_due_date NVARCHAR(20)   NULL,
        net_due_date               NVARCHAR(20)   NULL,
        special_gl_indicator       NVARCHAR(20)   NULL,
        amount_in_doc_currency     NVARCHAR(20)   NULL,
        local_currency             NVARCHAR(20)   NULL,
        amount_in_doc_currency2    NVARCHAR(20)   NULL,
        local_currency2            NVARCHAR(20)   NULL,
        payment_block              NVARCHAR(20)   NULL,
        username                   NVARCHAR(20)   NULL,
        clearing_document          NVARCHAR(50)   NULL,
        clearing_date              NVARCHAR(20)   NULL
    );

    -- temp table: same schema as dbo.trn_mc_othercustomer_header
    IF OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_header') IS NOT NULL
        DROP TABLE #tbl_trn_mc_othercustomer_header;

    CREATE TABLE #tbl_trn_mc_othercustomer_header (
        process_key               VARCHAR(40)   NOT NULL,
        process_code              NVARCHAR(5)   NOT NULL,
        calculate_date            DATE          NOT NULL,
        cust_cd                   VARCHAR(50)   NOT NULL,
        cust_name                 NVARCHAR(200) NOT NULL,
        customer_flag             INT           NOT NULL,
        offset_flag               INT           NOT NULL,
        match_clear_sap_status_cd INT           NULL,
        match_clear_sap_doc_no    VARCHAR(30)   NULL,
        match_clear_sap_message   NVARCHAR(MAX) NULL,
        error_description         NVARCHAR(MAX) NULL,
        create_date               DATETIME      NOT NULL,
        create_by                 VARCHAR(30)   NOT NULL,
        update_date               DATETIME      NULL,
        update_by                 VARCHAR(30)   NULL
    );

    -- temp table: same schema as dbo.trn_mc_othercustomer_summary
    IF OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_summary') IS NOT NULL
        DROP TABLE #tbl_trn_mc_othercustomer_summary;

    CREATE TABLE #tbl_trn_mc_othercustomer_summary (
        process_key              VARCHAR(40)    NOT NULL,
        process_code             NVARCHAR(5)    NOT NULL,
        calculate_date           DATE           NOT NULL,
        cust_cd                  VARCHAR(50)    NOT NULL,
        cust_name                NVARCHAR(200)  NOT NULL,
        count_debit_all_type     INT            NOT NULL,
        count_credit_all_type    INT            NOT NULL,
        count_credit_offset_type INT            NOT NULL,
        count_debit_offset_type  INT            NOT NULL,
        count_due_date           INT            NOT NULL,
        count_notyetdue_date     INT            NOT NULL,
        notyetdue_minimum_amt    DECIMAL(18, 2) NOT NULL,
        total_minimum_amt        DECIMAL(18, 2) NOT NULL,
        total_credit_amt         DECIMAL(18, 2) NOT NULL,
        total_debit_amt          DECIMAL(18, 2) NOT NULL,
        customer_flag            INT            NOT NULL,
        offset_flag              INT            NOT NULL,
        create_date              DATETIME       NOT NULL,
        create_by                VARCHAR(30)    NOT NULL
    );

    BEGIN TRY
        BEGIN TRANSACTION;

        -- STEP 1: load raw FBL5N (inserts into #tbl_trx_mc_raw_fbl5n)
        EXEC dbo.usp_mc0403_get_raw_fbl5n
            @process_key  = @process_key,
            @process_code = @process_code,
            @update_by    = @update_by;

        -- STEP 2: header (#tbl_trx_mc_raw_fbl5n -> #tbl_trn_mc_othercustomer_header)
        EXEC dbo.usp_mc0403_insert_othercustomer_header
            @process_key  = @process_key,
            @process_code = @process_code,
            @update_by    = @update_by;

        -- STEP 3: summary (reads #tbl_trx_mc_raw_fbl5n)
        EXEC dbo.usp_mc0403_insert_othercustomer_summary
            @process_key  = @process_key,
            @process_code = @process_code,
            @update_by    = @update_by;

        -- TODO: next steps

        COMMIT TRANSACTION;

        -- return the result set after the commit
        SELECT 1;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        SELECT @ErrorMessage  = ERROR_MESSAGE(),
               @ErrorSeverity = ERROR_SEVERITY(),
               @ErrorState    = ERROR_STATE();

        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_mc0403_get_othercustomer_not_clear
    @process_key  = N'TEST_KEY',
    @process_code = N'MC04',
    @update_by    = 'kosin';
```
