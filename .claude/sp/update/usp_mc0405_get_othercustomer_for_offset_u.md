# Update Stored Procedure: `usp_mc0405_get_othercustomer_for_offset`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_key` | `VARCHAR(40)` |
| `@process_code` | `NVARCHAR(5)` |
| `@calculate_date` | `VARCHAR(8)` |
| `@customer_cd` | `VARCHAR(20)` |
| `@customer_flag` | `INT` |
| `@offset_flag` | `INT` |
| `@update_by` | `VARCHAR(20)` |

## รายละเอียดการแก้ไข

- copy concept มาจาก `usp_mc0404_get_othercustomer_for_offset` (จะแก้บางสูตรภายหลัง)
- select ข้อมูลจาก `dbo.trx_mc_raw_othercustomer` ไป insert ลง `dbo.trn_mc_othercustomer_detail`
  โดยมีเงื่อนไขเป็น `process_key` / `process_code` / `calculate_date` / `cust_cd` (= `@customer_cd`)
  / `customer_flag` (= `@customer_flag`)
- `@calculate_date` รับเป็น `YYYYMMDD` แล้วแปลงเป็น `date` (style 112)
- ก่อน insert → `DELETE` ข้อมูลเดิมใน `dbo.trn_mc_othercustomer_detail` ของ key เดียวกัน (รันซ้ำได้ ไม่ชน PK)
  และ `customer_flag = @customer_flag` — detail ไม่มีคอลัมน์ `customer_flag` จึงเช็คผ่าน
  `EXISTS` กับ `trx_mc_raw_othercustomer` (join ด้วย key + `document_no`)
- mapping:
  - `reference_no` = `r.reference` (ต่อจาก `document_no`); ถ้า `NULL` → `''` (คอลัมน์เป็น NOT NULL)
  - `thb_gross` ใน raw เป็นข้อความรูปแบบ SAP (`40,091.92-`) → ตัด `,` แล้วย้าย `-` ท้ายมาไว้ข้างหน้า → `DECIMAL(18,2)`
  - `dc_flag` = `'C'` ถ้า `thb_gross` ติดลบ ไม่งั้น `'D'`
  - `net_due_date` ลงตามเดิม
  - `total_credit_amt` = `SUM(thb_gross)` ของแถวที่ `dc_flag = 'C'` group by key
    (`process_key` / `process_code` / `calculate_date` / `cust_cd`) ใส่ทุกแถวของ key นั้น
    (ใช้ `SUM() OVER (PARTITION BY ...)`; ค่าเป็นลบตาม `thb_gross`, ไม่มี credit → `0`)
  - `selection_flag` = `1` ถ้า `r.customer_flag = 1` และ `r.net_due_date <= @calculate_date` ไม่งั้น `0`
    (`net_due_date` ใน raw เป็นข้อความ `DD.MM.YYYY` → `TRY_CONVERT(DATE, ..., 104)`; แปลงไม่ได้ → `0`)
  - `resitem_flag` = `0` (fix)
  - `accum_thb_gross` / `remaining_amt` → `NULL` (ไว้คำนวณ offset ทีหลัง)
  - `create_date` = `GETDATE()`, `create_by` = `@update_by`
- `@offset_flag` ยังไม่ได้ใช้
- เพิ่ม `SET XACT_ABORT ON` + `BEGIN TRY / BEGIN CATCH`
- `DELETE` + `INSERT` อยู่ใน transaction เดียวกัน (`BEGIN TRANSACTION` / `COMMIT TRANSACTION`)
- ถ้าเกิด error (รวมถึง `@calculate_date` แปลงเป็น date ไม่ได้) → `ROLLBACK TRANSACTION`
  (ถ้ามี transaction เปิดอยู่) แล้ว `RAISERROR` ส่ง error message / severity / state เดิมกลับไปให้ผู้เรียก
- หลัง `COMMIT` → คืน result set จาก `dbo.trn_mc_othercustomer_detail` ตามเงื่อนไข parameter
  (`process_key` / `process_code` / `calculate_date` / `cust_cd`):
  `resitem_flag`, `calculate_date`, `cust_cd AS customer_cd`, `document_no`, `selection_flag`

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_mc0405_get_othercustomer_for_offset]
    @process_key    VARCHAR(40),
    @process_code   NVARCHAR(5),
    @calculate_date VARCHAR(8),
    @customer_cd    VARCHAR(20),
    @customer_flag  INT,
    @offset_flag    INT,
    @update_by      VARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    DECLARE @ErrorMessage  NVARCHAR(4000),
            @ErrorSeverity INT,
            @ErrorState    INT;

    DECLARE @calc_date DATE;

    BEGIN TRY
        SET @calc_date = CONVERT(DATE, @calculate_date, 112);

        BEGIN TRANSACTION;

        -- re-run safe: clear this key's rows before inserting
        -- detail has no customer_flag: match it via the raw rows
        DELETE d
        FROM dbo.trn_mc_othercustomer_detail d
        WHERE d.process_key    = @process_key
          AND d.process_code   = @process_code
          AND d.calculate_date = @calc_date
          AND d.cust_cd        = @customer_cd
          AND EXISTS (
                SELECT 1
                FROM dbo.trx_mc_raw_othercustomer r
                WHERE r.process_key    = d.process_key
                  AND r.process_code   = d.process_code
                  AND r.calculate_date = d.calculate_date
                  AND r.cust_cd        = d.cust_cd
                  AND r.document_no    = d.document_no
                  AND r.customer_flag  = @customer_flag
          );

        INSERT INTO dbo.trn_mc_othercustomer_detail (
            process_key, process_code, calculate_date, cust_cd, document_no,
            reference_no, dc_flag, total_credit_amt, net_due_date, thb_gross,
            accum_thb_gross, remaining_amt, selection_flag, resitem_flag,
            create_date, create_by
        )
        SELECT r.process_key,
               r.process_code,
               r.calculate_date,
               r.cust_cd,
               r.document_no,
               ISNULL(r.reference, ''),   -- reference_no is NOT NULL
               CASE WHEN g.thb_gross < 0 THEN 'C' ELSE 'D' END,
               -- sum of credit (dc_flag = 'C') thb_gross per key
               SUM(CASE WHEN g.thb_gross < 0 THEN g.thb_gross ELSE 0 END)
                   OVER (PARTITION BY r.process_key, r.process_code,
                                      r.calculate_date, r.cust_cd),
               r.net_due_date,
               g.thb_gross,
               NULL,
               NULL,
               -- selection_flag: customer-flagged and already due
               CASE WHEN r.customer_flag = 1
                         AND TRY_CONVERT(DATE, r.net_due_date, 104) <= @calc_date
                    THEN 1 ELSE 0 END,
               0,              -- resitem_flag
               GETDATE(),
               @update_by
        FROM dbo.trx_mc_raw_othercustomer r
        -- SAP amount text '40,091.92-' -> -40091.92
        CROSS APPLY (
            SELECT CAST(
                       CASE WHEN RIGHT(LTRIM(RTRIM(r.thb_gross)), 1) = '-'
                            THEN '-' + LEFT(REPLACE(LTRIM(RTRIM(r.thb_gross)), ',', ''),
                                            LEN(REPLACE(LTRIM(RTRIM(r.thb_gross)), ',', '')) - 1)
                            ELSE REPLACE(LTRIM(RTRIM(r.thb_gross)), ',', '')
                       END AS DECIMAL(18, 2)) AS thb_gross
        ) g
        WHERE r.process_key    = @process_key
          AND r.process_code   = @process_code
          AND r.calculate_date = @calc_date
          AND r.cust_cd        = @customer_cd
          AND r.customer_flag  = @customer_flag;

        COMMIT TRANSACTION;

        -- return the result set after the commit
        SELECT d.resitem_flag,
               d.calculate_date,
               d.cust_cd AS customer_cd,
               d.document_no,
               d.selection_flag
        FROM dbo.trn_mc_othercustomer_detail d
        WHERE d.process_key    = @process_key
          AND d.process_code   = @process_code
          AND d.calculate_date = @calc_date
          AND d.cust_cd        = @customer_cd;
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
EXEC dbo.usp_mc0405_get_othercustomer_for_offset
    @process_key    = '1',
    @process_code   = N'MC04',
    @calculate_date = '20261001',
    @customer_cd    = '8517448',
    @customer_flag  = 1,
    @offset_flag    = 0,
    @update_by      = 'kosin';

SELECT * FROM dbo.trn_mc_othercustomer_detail
WHERE process_key = '1' AND process_code = N'MC04' AND cust_cd = '8517448';
```
