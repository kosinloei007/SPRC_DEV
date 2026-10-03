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
- ใช้ temp table `#tbl_trn_mc_othercustomer_detail` ที่ orchestrator
  (`usp_mc0403_get_othercustomer_not_clear`) สร้างไว้ เป็นที่พักข้อมูลและคำนวณ offset
- หลังคำนวณเสร็จ → insert ข้อมูลจาก `#tbl_trn_mc_othercustomer_detail` ลง table จริง
  `dbo.trn_mc_othercustomer_detail` (16 field; ไม่เอา `seq` เพราะ table จริงไม่มี)
  - ก่อน insert → `DELETE` ข้อมูลเดิมใน table จริงของ key เดียวกัน
    (`process_key` / `process_code` / `calculate_date` / `cust_cd`) ที่ `customer_flag` /
    `offset_flag` ตรงกับ parameter (เช็คผ่าน `EXISTS` กับ `trx_mc_raw_othercustomer`
    ด้วย key + `document_no`) — รันซ้ำได้ ไม่ชน PK
- ถ้าไม่มี temp table (เรียกเดี่ยว ๆ) → `RAISERROR` แล้วจบ
- select ข้อมูลจาก `dbo.trx_mc_raw_othercustomer` ไป insert ลง `#tbl_trn_mc_othercustomer_detail`
  โดยมีเงื่อนไขเป็น `process_key` / `process_code` / `calculate_date` / `cust_cd` (= `@customer_cd`)
  / `customer_flag` (= `@customer_flag`) / `offset_flag` (= `@offset_flag`)
- เอาเฉพาะแถวที่ `net_due_date <= calculate_date` (`net_due_date` ใน raw เป็นข้อความ
  `DD.MM.YYYY` → `TRY_CONVERT(DATE, ..., 104)`; แปลงไม่ได้ / `NULL` → ไม่เอา)
- `ORDER BY net_due_date, document_no` (เรียง `net_due_date` แบบวันที่ ไม่ใช่ข้อความ)
  เฉพาะ result set ที่คืนกลับ — ตอน select จาก `trx_mc_raw_othercustomer` ไป insert
  **ยังไม่ order by**
- `@calculate_date` รับเป็น `YYYYMMDD` แล้วแปลงเป็น `date` (style 112)
- ก่อน insert → `DELETE` ข้อมูลใน `#tbl_trn_mc_othercustomer_detail` **ทั้งหมดแบบไม่มีเงื่อนไข**
  (temp table จะเหลือเฉพาะข้อมูลของการเรียกครั้งล่าสุด)
- mapping:
  - `reference_no` = `r.reference` (ต่อจาก `document_no`); ถ้า `NULL` → `''` (คอลัมน์เป็น NOT NULL)
  - `thb_gross` ใน raw เป็นข้อความรูปแบบ SAP (`40,091.92-`) → ตัด `,` แล้วย้าย `-` ท้ายมาไว้ข้างหน้า → `DECIMAL(18,2)`
  - `dc_flag` = `'C'` ถ้า `thb_gross` ติดลบ ไม่งั้น `'D'`
  - `net_due_date` ลงตามเดิม
  - `total_credit_amt` = `SUM(thb_gross)` ของแถวที่ `dc_flag = 'C'` group by key
    (`process_key` / `process_code` / `calculate_date` / `cust_cd`) ใส่ทุกแถวของ key นั้น
    (ใช้ `SUM() OVER (PARTITION BY ...)`; ค่าเป็นลบตาม `thb_gross`, ไม่มี credit → `0`)
  - `resitem_flag` = `0` (fix)
  - `accum_thb_gross` / `remaining_amt` / `selection_flag` — คำนวณหลัง insert
    (ตาม `.claude/img/2026-10-03_21-16-46.jpg`):
    - แถว `dc_flag = 'C'` → `accum_thb_gross = NULL`, `remaining_amt = NULL`, `selection_flag = 2`
      (รูปเขียนว่าให้ลง `0` แต่แก้เป็น `NULL` ตาม requirement ล่าสุด)
    - แถว `dc_flag = 'D'` ไล่ตาม `seq` (loop ทีละแถว):
      - `accum_thb_gross` = `remaining_amt` ของแถว D ก่อนหน้า − `thb_gross`
        (แถว D แรก: `total_credit_amt` แปลงเป็นบวก − `thb_gross`)
      - `accum_thb_gross >= 0` → `remaining_amt = accum_thb_gross`, `selection_flag = 1`
      - `accum_thb_gross < 0` → `remaining_amt` คงค่าของแถวก่อนหน้า, `selection_flag = 0`
        (แถว D แรกติดลบ → `remaining_amt` = `total_credit_amt` แปลงเป็นบวก)
  - `create_date` = `GETDATE()`, `create_by` = `@update_by`
  - `seq` = `ROW_NUMBER() OVER (ORDER BY dc_flag, net_due_date, document_no)`
    (`C` มาก่อน `D`; `net_due_date` เรียงแบบวันที่) — คอลัมน์นี้มีเฉพาะใน temp table- เพิ่ม `SET XACT_ABORT ON` + `BEGIN TRY / BEGIN CATCH`
- `DELETE` + `INSERT` อยู่ใน transaction เดียวกัน (`BEGIN TRANSACTION` / `COMMIT TRANSACTION`)
- ถ้าเกิด error (รวมถึง `@calculate_date` แปลงเป็น date ไม่ได้) → `ROLLBACK TRANSACTION`
  (ถ้ามี transaction เปิดอยู่) แล้ว `RAISERROR` ส่ง error message / severity / state เดิมกลับไปให้ผู้เรียก
- หลัง `COMMIT` → คืน result set จาก `#tbl_trn_mc_othercustomer_detail` ตามเงื่อนไข parameter
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

    -- offset running calculation (debit rows, by seq)
    DECLARE @seq       INT,
            @thb_gross DECIMAL(18, 2),
            @accum     DECIMAL(18, 2),
            @remaining DECIMAL(18, 2);

    -- requires the temp table created by the orchestrator
    IF OBJECT_ID('tempdb..#tbl_trn_mc_othercustomer_detail') IS NULL
    BEGIN
        RAISERROR('#tbl_trn_mc_othercustomer_detail not found - call via usp_mc0403_get_othercustomer_not_clear', 16, 1);
        RETURN;
    END

    BEGIN TRY
        SET @calc_date = CONVERT(DATE, @calculate_date, 112);

        BEGIN TRANSACTION;

        -- clear the whole temp table before inserting (no condition)
        DELETE FROM #tbl_trn_mc_othercustomer_detail;

        -- insert into the temp table (persisted to dbo.trn_mc_othercustomer_detail below)
        INSERT INTO #tbl_trn_mc_othercustomer_detail (
            process_key, process_code, calculate_date, cust_cd, document_no,
            reference_no, dc_flag, total_credit_amt, net_due_date, thb_gross,
            accum_thb_gross, remaining_amt, selection_flag, resitem_flag,
            create_date, create_by, seq
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
               NULL,           -- accum_thb_gross: calculated below
               NULL,           -- remaining_amt:   calculated below
               NULL,           -- selection_flag:  calculated below
               0,              -- resitem_flag
               GETDATE(),
               @update_by,
               -- seq: running number by dc_flag, net_due_date, document_no
               ROW_NUMBER() OVER (ORDER BY CASE WHEN g.thb_gross < 0 THEN 'C' ELSE 'D' END,
                                           TRY_CONVERT(DATE, r.net_due_date, 104),
                                           r.document_no)
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
          AND r.customer_flag  = @customer_flag
          AND r.offset_flag    = @offset_flag
          -- already due only: net_due_date (text DD.MM.YYYY) <= calculate_date
          AND TRY_CONVERT(DATE, r.net_due_date, 104) <= @calc_date;

        -- credit rows: nothing to offset against
        UPDATE #tbl_trn_mc_othercustomer_detail
        SET accum_thb_gross = NULL,
            remaining_amt   = NULL,
            selection_flag  = 2
        WHERE dc_flag = 'C';

        -- debit rows, by seq: use up the credit total row by row
        --   accum_thb_gross = previous remaining_amt - thb_gross
        --                     (first debit row: total_credit_amt as positive - thb_gross)
        --   accum >= 0 -> remaining_amt = accum,              selection_flag = 1
        --   accum <  0 -> remaining_amt = previous remaining, selection_flag = 0
        SELECT @remaining = ABS(ISNULL(MAX(total_credit_amt), 0))
        FROM #tbl_trn_mc_othercustomer_detail;

        SELECT @seq = MIN(seq)
        FROM #tbl_trn_mc_othercustomer_detail
        WHERE dc_flag = 'D';

        WHILE @seq IS NOT NULL
        BEGIN
            SELECT @thb_gross = ISNULL(thb_gross, 0)
            FROM #tbl_trn_mc_othercustomer_detail
            WHERE seq = @seq;

            SET @accum = @remaining - @thb_gross;

            IF @accum >= 0
                SET @remaining = @accum;

            UPDATE #tbl_trn_mc_othercustomer_detail
            SET accum_thb_gross = @accum,
                remaining_amt   = @remaining,
                selection_flag  = CASE WHEN @accum >= 0 THEN 1 ELSE 0 END
            WHERE seq = @seq;

            SELECT @seq = MIN(seq)
            FROM #tbl_trn_mc_othercustomer_detail
            WHERE dc_flag = 'D'
              AND seq > @seq;
        END

        -- persist: #tbl_trn_mc_othercustomer_detail -> dbo.trn_mc_othercustomer_detail
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
                  AND r.offset_flag    = @offset_flag
          );

        -- seq is temp-only: not persisted
        INSERT INTO dbo.trn_mc_othercustomer_detail (
            process_key, process_code, calculate_date, cust_cd, document_no,
            reference_no, dc_flag, total_credit_amt, net_due_date, thb_gross,
            accum_thb_gross, remaining_amt, selection_flag, resitem_flag,
            create_date, create_by
        )
        SELECT t.process_key, t.process_code, t.calculate_date, t.cust_cd, t.document_no,
               t.reference_no, t.dc_flag, t.total_credit_amt, t.net_due_date, t.thb_gross,
               t.accum_thb_gross, t.remaining_amt, t.selection_flag, t.resitem_flag,
               t.create_date, t.create_by
        FROM #tbl_trn_mc_othercustomer_detail t;

        COMMIT TRANSACTION;

        -- return the result set after the commit
        SELECT d.resitem_flag,
               d.calculate_date,
               d.cust_cd AS customer_cd,
               d.document_no,
               d.selection_flag
        FROM #tbl_trn_mc_othercustomer_detail d
        WHERE d.process_key    = @process_key
          AND d.process_code   = @process_code
          AND d.calculate_date = @calc_date
          AND d.cust_cd        = @customer_cd
        ORDER BY TRY_CONVERT(DATE, d.net_due_date, 104),
                 d.document_no;
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
-- simulate the orchestrator's temp table
SELECT TOP (0) *,
       CAST(NULL AS INT) AS seq
INTO #tbl_trn_mc_othercustomer_detail
FROM dbo.trn_mc_othercustomer_detail;

EXEC dbo.usp_mc0405_get_othercustomer_for_offset
    @process_key    = '1',
    @process_code   = N'MC04',
    @calculate_date = '20261001',
    @customer_cd    = '8516784',
    @customer_flag  = 1,
    @offset_flag    = 1,
    @update_by      = 'kosin';

SELECT * FROM #tbl_trn_mc_othercustomer_detail
WHERE process_key = '1' AND process_code = N'MC04' AND cust_cd = '8516784'
ORDER BY seq;

SELECT * FROM dbo.trn_mc_othercustomer_detail
WHERE process_key = '1' AND process_code = N'MC04' AND cust_cd = '8516784';
```
