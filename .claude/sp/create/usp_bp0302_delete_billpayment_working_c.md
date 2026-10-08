# Create Stored Procedure: `usp_bp0302_delete_billpayment_working`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_code` | `NVARCHAR(5)` |
| `@step_code` | `NVARCHAR(10)` |
| `@updated_by` | `NVARCHAR(20)` |
| `@process_key` | `VARCHAR(50)` |
## Requirement (initial)

Requirement ปัจจุบัน (ถอดจาก stored เดิมใน `RPA_DEV` — Author: Kannika T., 16 Dec 2024, "Delete Transaction"):

ลบข้อมูลเก่าที่เกินระยะเวลาเก็บ ออกจากตาราง history ของ Bill Payment และตารางของ Match & Clear

1. **ค่าตั้งต้น**
   - `@CalculateDate = GETDATE()`
   - `@Day` = `fn_get_predefine_by_key(1006, 'DAY_DELETE_DATA_BP')`, `@DayMC` = `fn_get_predefine_by_key(1006, 'DAY_DELETE_DATA_MC')`
   - จุดตัด = `CAST(DATEADD(day, <จำนวนวัน>, @CalculateDate) AS DATE)` — ลบแถวที่ `create_date <=` จุดตัด
2. **ลบข้อมูล Bill Payment** (ใช้ `@Day`): `bp_billpayment_staging_history`, `bp_billpayment_final_history`,
   `bp_zgenfip_final_history`, `trx_bp_raw_upload_file_result`, `trx_bp_raw_document_file_result`
3. **ลบข้อมูล Match & Clear** (ใช้ `@DayMC`): `trx_mc_raw_match_and_clear`, `trx_raw_mas_customer_sales`,
   `trn_mc_detail`, `trn_mc_header`
   - `trn_mc_othercustomer_header`, `trn_mc_othercustomer_detail`, `trx_mc_raw_othercustomer` — เขียน delete ไว้แล้วแต่ **comment ออกก่อน** (ยังไม่ลบ)
   - ตาราง history ของ Match & Clear (ใช้ `@DayMC` เงื่อนไขเดียวกัน): `trx_mc_raw_match_and_clear_history`, `trn_mc_header_history`,
     `trn_mc_detail_history`, `trn_mc_othercustomer_header_history`, `trn_mc_othercustomer_detail_history`,
     `trx_mc_raw_othercustomer_history`
4. ทั้งหมดอยู่ใน transaction เดียว — `COMMIT` เมื่อสำเร็จ; `CATCH`: `ROLLBACK` แล้ว `RAISERROR` ข้อความเดิมออกไป

หมายเหตุ: parameter ทั้ง 4 ตัว (`@process_code`, `@step_code`, `@updated_by`, `@process_key`) รับเข้ามาแต่ **ไม่ได้ถูกใช้** ใน body
## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_bp0302_delete_billpayment_working', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_bp0302_delete_billpayment_working
GO

-- =============================================
-- Author:		Kannika T.
-- Create date: 16 Dec 2024
-- Description:	Delete Transaction
-- =============================================
CREATE PROCEDURE [dbo].[usp_bp0302_delete_billpayment_working]
	@process_code	NVARCHAR(5)
	,@step_code NVARCHAR(10)
	,@updated_by NVARCHAR(20)
	,@process_key VARCHAR(50)
AS
BEGIN
	DECLARE @ErrorMessage VARCHAR(4000)
			,@ErrorSeverity INT
			,@ErrorState INT
			,@Day INT
			,@DayMC INT
			,@CalculateDate DATETIME
	BEGIN TRY
		SET @CalculateDate = GETDATE()
		SET @Day = (SELECT [dbo].[fn_get_predefine_by_key](1006,'DAY_DELETE_DATA_BP'))
		SET @DayMC =(SELECT [dbo].[fn_get_predefine_by_key](1006,'DAY_DELETE_DATA_MC')) 
		BEGIN TRANSACTION
		DELETE
		FROM bp_billpayment_staging_history WHERE create_date <= CAST(DATEADD(day,@Day,@CalculateDate) AS DATE)

		DELETE
		FROM bp_billpayment_final_history WHERE create_date <= CAST(DATEADD(day,@Day,@CalculateDate) AS DATE)

		DELETE
		FROM bp_zgenfip_final_history WHERE create_date <= CAST(DATEADD(day,@Day,@CalculateDate) AS DATE)
		
		DELETE
		FROM trx_bp_raw_upload_file_result WHERE create_date <= CAST(DATEADD(day,@Day,@CalculateDate) AS DATE)
		
		DELETE
		FROM trx_bp_raw_document_file_result WHERE create_date <= CAST(DATEADD(day,@Day,@CalculateDate) AS DATE)

	

        --#Start - Delete Match and Clear Data
		DELETE
		FROM [trx_mc_raw_match_and_clear] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		DELETE
		FROM [trx_raw_mas_customer_sales] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)
		
		DELETE
		FROM [trn_mc_detail] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		DELETE
		FROM [trn_mc_header] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		--DELETE
		--FROM [trn_mc_othercustomer_header] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		--DELETE
		--FROM [trn_mc_othercustomer_detail] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		--DELETE
		--FROM [trx_mc_raw_othercustomer] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		DELETE
		FROM [trx_mc_raw_match_and_clear_history] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		DELETE
		FROM [trn_mc_header_history] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		DELETE
		FROM [trn_mc_detail_history] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		DELETE
		FROM [trn_mc_othercustomer_header_history] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		DELETE
		FROM [trn_mc_othercustomer_detail_history] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)

		DELETE
		FROM [trx_mc_raw_othercustomer_history] WHERE create_date <= CAST(DATEADD(day,@DayMC,@CalculateDate) AS DATE)
		 --#End - Delete Match and Clear Data

		IF @@TRANCOUNT > 0
		BEGIN
			COMMIT TRANSACTION
		END	

	END TRY
	BEGIN CATCH
		SET @ErrorMessage = ISNULL(ERROR_MESSAGE(),'')
		SET @ErrorSeverity = ERROR_SEVERITY()
		SET @ErrorState = ERROR_STATE()
		IF @@TRANCOUNT > 0
		BEGIN
			ROLLBACK TRANSACTION
		END
		RAISERROR(@ErrorMessage,@ErrorSeverity,@ErrorState);
	END CATCH
END
GO
```

## ตัวอย่างการเรียกใช้

proc นี้ลบข้อมูลจริง จึง comment ตัวอย่างไว้ ไม่ให้ runner เรียกอัตโนมัติ

```sql
-- EXEC dbo.usp_bp0302_delete_billpayment_working
--     @process_code = N'...', @step_code = N'...', @updated_by = N'...', @process_key = '...';
```
