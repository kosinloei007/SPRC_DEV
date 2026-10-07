# Update Stored Procedure: `usp_mc0104_update_match_and_clear_status`

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

## รายละเอียดการแก้ไข

requirement ปัจจุบัน (ถอดจาก proc ที่มีอยู่ใน `RPA_DEV` — ยังไม่มีการแก้ไข logic):

- หน้าที่: update สถานะ match and clear หลัง post SAP โดยแยก 2 กรณีตาม `@is_other_customer`
- ความหมาย parameter:
  - `@account` = `ref_1` (กรณีปกติ) / `cust_cd` (กรณี other customer)
  - `@pay_date` = `pay_date` (กรณีปกติ) / `calculate_date` (กรณี other customer)
  - `@status_cd`: `1` = Error, `2` = Success, `9` = NOT Retail
  - `@error_description` = ข้อความ error / message จาก SAP
  - `@match_and_clear_sap_doc_no` = ข้อความเลขที่เอกสารจาก SAP
    เช่น `Document 25000008 was posted i.The statement has been terminated`
- กรณีปกติ (`@is_other_customer` = `0` หรือ `NULL`) → update 2 table:
  1. `dbo.trn_mc_header`
     where `process_key` / `pay_date` / `bank_running_no` / `ref_1` (= `@account`)
     - `status_cd` = `@status_cd`
     - `error_description` = `@error_description`
     - `sap_doc_no` = `@match_and_clear_sap_doc_no` (เก็บข้อความเต็ม ไม่ตัด)
     - `update_date` = `GETDATE()`, `update_by` = `@update_by`
  2. `dbo.bp_billpayment_final`
     where `pay_date` / `bank_running_no` / `ref_1` (= `@account`) — **ไม่ได้ใช้ `process_key`**
     - `match_clear_status_cd` = `@status_cd`
     - `match_clear_date` = `GETDATE()`, `match_clear_by` = `@update_by`
     - `match_clear_message` = `@error_description` (เพิ่มตอน UAT 11.2.2025)
     - `match_clear_sap_doc_no` = ตัดเลขเอกสารจาก `@match_and_clear_sap_doc_no` (trim ก่อน):
       ถ้าไม่ว่างและยาวเกิน 30 ตัวอักษร → `SUBSTRING(..., 10, 8)` (ได้ `25000008` จากตัวอย่างข้างบน)
       นอกนั้น → ใช้ค่าที่ trim แล้วทั้งก้อน (`NULL` → `''`)
     - `update_date` = `GETDATE()`, `update_by` = `@update_by`
- กรณี other customer (`@is_other_customer` = `1`) → update `dbo.trn_mc_othercustomer_header`
  where `process_key` / `calculate_date` (= `@pay_date`) / `cust_cd` (= `@account`)
  - `match_clear_sap_status_cd` = `@status_cd`
  - `match_clear_sap_doc_no` = `@match_and_clear_sap_doc_no` (เก็บข้อความเต็ม ไม่ตัดเลขเอกสาร)
  - `match_clear_sap_message` = `@error_description`
  - `match_clear_by` = `@update_by`, `match_clear_date` = `GETDATE()`
  - ไม่ได้ใช้ `@bank_running_no`; ไม่ได้ update `error_description` / `update_date` / `update_by`
- ไม่มี `SET NOCOUNT ON`, ไม่มี transaction / `TRY...CATCH`, ไม่คืน result set

## Script

```sql
USE [RPA_DEV]
GO

-- =============================================
-- Author:		Nittaya N.
-- Create date: 29 Dec 2024
-- Description:	Update match and clear status
-- EXEC usp_mc0104_update_match_and_clear_status '20250202_201956349','8418494','T0001','2025-01-31',1,'',"Document 25000008 was posted i.The statement has been terminated","RPA"
-- =============================================
ALTER PROCEDURE [dbo].[usp_mc0104_update_match_and_clear_status]
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
