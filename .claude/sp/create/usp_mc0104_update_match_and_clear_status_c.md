# Create Stored Procedure: `usp_mc0104_update_match_and_clear_status`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `VARCHAR(40)` |
| `@account` | `VARCHAR(200)` |
| `@bank_running_no` | `VARCHAR(10)` |
| `@pay_date` | `DATE` |
| `@status_cd` | `INT` |
| `@error_description` | `NVARCHAR(1000)` |
| `@match_and_clear_sap_doc_no` | `NVARCHAR(1000)` |
| `@update_by` | `VARCHAR(30)` |
| `@is_other_customer` | `BIT` |

## Requirement (initial)

Update match and clear status หลัง post SAP (script นี้ยกมาจาก proc ที่มีอยู่แล้วใน `RPA_DEV`):

- `@is_other_customer` = 0 / `NULL` → update `dbo.trn_mc_header` และ `dbo.bp_billpayment_final`
- `@is_other_customer` = 1 → update `dbo.trn_mc_othercustomer_header`
  (`@account` = `cust_cd`, `@pay_date` = `calculate_date`)

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_mc0104_update_match_and_clear_status', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_mc0104_update_match_and_clear_status
GO

-- =============================================
-- Author:		Nittaya N.
-- Create date: 29 Dec 2024
-- Description:	Update match and clear status
-- EXEC usp_mc0104_update_match_and_clear_status '20250202_201956349','8418494','T0001','2025-01-31',1,'',"Document 25000008 was posted i.The statement has been terminated","RPA"
-- =============================================
CREATE PROCEDURE [dbo].[usp_mc0104_update_match_and_clear_status]
    @process_key VARCHAR(40),
    @account VARCHAR(200),                        -- (customer_cd = account for othercustomer)
    @bank_running_no VARCHAR(10),
    @pay_date DATE,                               -- (calculate_date = pay_date for othercustomer)
    @status_cd INT,
    @error_description NVARCHAR(1000),
    @match_and_clear_sap_doc_no NVARCHAR(1000),   -- Document No. XXXXXXX was sssss
    @update_by VARCHAR(30),
    @is_other_customer BIT
AS
BEGIN

    IF ISNULL(@is_other_customer, 0) = 0
    BEGIN
        UPDATE [dbo].[trn_mc_header]
        SET [status_cd]         = @status_cd,
            [error_description] = @error_description,
            [sap_doc_no]        = @match_and_clear_sap_doc_no,
            [update_date]       = GETDATE(),
            [update_by]         = @update_by
        WHERE process_key     = @process_key
          AND pay_date        = @pay_date
          AND bank_running_no = @bank_running_no
          AND ref_1           = @account;

        UPDATE [dbo].[bp_billpayment_final]
        SET [match_clear_status_cd]  = @status_cd, -- 1:Error 2:Success 9:NOT Retail
            [match_clear_date]       = GETDATE(),
            [match_clear_by]         = @update_by,
            [match_clear_message]    = @error_description, -- add 11.2.2025 UAT
            [match_clear_sap_doc_no] = case when trim(isnull(@match_and_clear_sap_doc_no,'')) <> '' and len(trim(isnull(@match_and_clear_sap_doc_no,''))) > 30 then
                                                 substring(trim(isnull(@match_and_clear_sap_doc_no,'')) ,10,8)
                                            else
                                                 trim(isnull(@match_and_clear_sap_doc_no,''))
                                       end,
            [update_date]            = GETDATE(),
            [update_by]              = @update_by
        WHERE pay_date        = @pay_date
          AND bank_running_no = @bank_running_no
          AND ref_1           = @account;
    END
    ELSE
    BEGIN
        UPDATE trn_mc_othercustomer_header
        SET match_clear_sap_status_cd = @status_cd,
            match_clear_sap_doc_no    = @match_and_clear_sap_doc_no,
            match_clear_sap_message   = @error_description,
            match_clear_by            = @update_by,
            match_clear_date          = GETDATE()
        WHERE process_key    = @process_key
          AND calculate_date = @pay_date
          AND cust_cd        = @account;
    END
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
-- TEST_KEY has no rows: exercises the othercustomer branch without changing data
EXEC dbo.usp_mc0104_update_match_and_clear_status
    @process_key                = 'TEST_KEY',
    @account                    = '8516784',
    @bank_running_no            = '',
    @pay_date                   = '2026-10-01',
    @status_cd                  = 2,
    @error_description          = N'',
    @match_and_clear_sap_doc_no = N'',
    @update_by                  = 'kosin',
    @is_other_customer          = 1;
```
