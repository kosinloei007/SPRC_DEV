# Update Stored Procedure: `usp_bp0202_pending_billpayment_working`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_code` | `NVARCHAR(5)` |
| `@step_code` | `NVARCHAR(10)` |
| `@updated_by` | `NVARCHAR(20)` |
| `@process_key` | `VARCHAR(50)` |

## รายละเอียดการแก้ไข

Requirement ปัจจุบัน (ถอดจาก stored เดิมใน `RPA_DEV` — Author: Kannika T., 16 Dec 2024, "Pending Transaction"):

ย้ายข้อมูล Bill Payment ที่ประมวลผลจบแล้วออกจากตาราง working ไปเก็บใน history แล้วลบออกจาก working
โดยรายการที่ยัง pending (ยังทำไม่ครบขั้นตอน) ต้องคงอยู่ใน working ต่อไป

1. **ค่าตั้งต้น**
   - `@CalculateDate = GETDATE()` ใช้เป็นจุดตัด — ประมวลผลเฉพาะข้อมูลที่ `create_date < @CalculateDate`
   - รหัสธนาคาร: BBL = 1, KBANK = 2, JPMC = 3, TTB = 4
   - รูปแบบวันที่ `pay_date` ของ staging อ่านจาก `fn_get_predefine_by_key`:
     key `1005` = style ของ `CONVERT` (BBL, TTB), key `1002` = culture ของ `TRY_PARSE` (KBANK, JPMC)
2. **เตรียมข้อมูล (temp table)**
   - 2.1 `#tmpAllBillPaymentFinal` — ทุกแถวของ `bp_billpayment_final` ที่ `create_date < @CalculateDate`
   - 2.2 `#tmpBillPaymentStaging` — แถวของ `bp_billpayment_staging` ที่ `create_date < @CalculateDate` และ
     - `pay_status_cd = 3` (Cancel) ทุกกรณี หรือ
     - `pay_status_cd = 1` และ `send_to_final_billpayment_cd = 1` และ **ไม่มี** แถวใน final
       (จับคู่ด้วย `bank_cd` + `pay_date` ที่แปลงตามรูปแบบของแต่ละธนาคาร + `bank_running_no`) ที่ยังค้างอยู่
       คือ Gen ZGENFIP ยังไม่เสร็จ (`gen_zgenfip_status_cd <> 1`) หรือ Posting SAP ยังไม่เสร็จ (`sap_status_cd <> 1`)
       หรือ Match & Clear ยังไม่เสร็จ (`match_clear_status_cd` ว่าง)
   - 2.3 `#tmpZgenfipFinal` — แถวของ `bp_zgenfip_final` ที่มีคู่ใน final (`bank_cd` + `pay_date` + `bank_running_no`)
     ยกเว้นไฟล์ (`file_name`) ที่ยังรอ Posting SAP คือ final มี `pay_status_cd = 1`, `gen_zgenfip_status_cd = 1`
     และ `sap_status_cd = 2` (Wait Posting SAP / Cannot Posting SAP)
   - 2.4 `#tmpBillPaymentFinal` — แถวจาก 2.1 ที่
     - `pay_status_cd = 3` (Cancel) หรือ
     - `pay_status_cd = 1` และ ( (`sap_status_cd = 1` และ `gen_zgenfip_status_cd = 1` และ `match_clear_status_cd` ไม่ว่าง)
       หรือ `sap_status_cd = 3` (Reverse Posting SAP) )
3. **ย้ายเข้า history และลบ working ใน transaction เดียว** (ส่ง `@CalculateDate` ให้ทุกตัว; proc ลูกใช้ temp table จากข้อ 2)
   - `usp_bp0202_pending_billpayment_move_history_temp`
   - `usp_bp0202_pending_billpayment_move_history_staging`
   - `usp_bp0202_pending_billpayment_move_history_final`
   - `usp_bp0202_pending_billpayment_move_history_zgenfip`
   - insert history ของ Match & Clear **จากตารางต้นทางโดยตรง (ไม่ผ่าน temp table)** — ทุกแถวที่ `create_date < @CalculateDate` ระบุชื่อคอลัมน์ครบทุกตัว และเพิ่ม `backup_date` = `@CalculateDate`, `backup_by` = `@updated_by`
     - `trx_mc_raw_match_and_clear` → `trx_mc_raw_match_and_clear_history`
     - `trn_mc_header` → `trn_mc_header_history`
     - `trn_mc_detail` → `trn_mc_detail_history`
     - `trn_mc_othercustomer_header` → `trn_mc_othercustomer_header_history`
     - `trn_mc_othercustomer_detail` → `trn_mc_othercustomer_detail_history`
     - `trx_mc_raw_othercustomer` → `trx_mc_raw_othercustomer_history`
   - `usp_bp0202_pending_billpayment_delete_working_all`
   - ลบข้อมูลต้นทางของ Match & Clear ทั้ง 6 ตาราง ด้วยเงื่อนไขเดียวกับตอน insert history (`create_date < @CalculateDate`)
   - `COMMIT` เมื่อสำเร็จ
4. **Error handling** — `CATCH`: `ROLLBACK` ถ้ามี transaction ค้าง, drop temp table ทั้ง 4 ตัว แล้ว `RAISERROR` ข้อความเดิมออกไป

หมายเหตุ: `@updated_by` ใช้เป็นค่า `backup_by` ของ history Match & Clear; อีก 3 ตัว (`@process_code`, `@step_code`, `@process_key`) รับเข้ามาแต่ **ไม่ได้ถูกใช้** ใน body

## Script

```sql
USE [RPA_DEV]
GO

-- =============================================
-- Author:		Kannika T.
-- Create date: 16 Dec 2024
-- Description:	Pending Transaction
-- =============================================
ALTER PROCEDURE [dbo].[usp_bp0202_pending_billpayment_working]
	@process_code	NVARCHAR(5)
	,@step_code NVARCHAR(10)
	,@updated_by NVARCHAR(20)
	,@process_key VARCHAR(50)
AS
BEGIN
	IF OBJECT_ID('tempdb..#tmpBillPaymentStaging') IS NOT NULL DROP TABLE #tmpBillPaymentStaging
	IF OBJECT_ID('tempdb..#tmpBillPaymentFinal') IS NOT NULL DROP TABLE #tmpBillPaymentFinal
	IF OBJECT_ID('tempdb..#tmpAllBillPaymentFinal') IS NOT NULL DROP TABLE #tmpAllBillPaymentFinal
	IF OBJECT_ID('tempdb..#tmpZgenfipFinal') IS NOT NULL DROP TABLE #tmpZgenfipFinal

	DECLARE @ErrorMessage VARCHAR(4000)
			,@ErrorSeverity INT
			,@ErrorState INT
			,@CalculateDate DATETIME

	DECLARE  @FormatPayDateBBL INT
			 ,@FormatPayDateTTB INT
			 ,@RegionPayDateKBANK VARCHAR(20)
			 ,@RegionPayDateJPMC VARCHAR(20)
			 ,@BankCodeBBL INT
			,@BankCodeKBANK INT
			,@BankCodeJPMC INT
			,@BankCodeTTB INT
	
	SET @BankCodeBBL = 1
	SET @BankCodeKBANK = 2
	SET @BankCodeJPMC = 3
	SET @BankCodeTTB = 4

	SET @FormatPayDateBBL  = (SELECT [dbo].[fn_get_predefine_by_key](1005,@BankCodeBBL))
	SET @FormatPayDateTTB  = (SELECT [dbo].[fn_get_predefine_by_key](1005,@BankCodeTTB))
	SET @RegionPayDateKBANK = (SELECT [dbo].[fn_get_predefine_by_key](1002,@BankCodeKBANK))
	SET @RegionPayDateJPMC = (SELECT [dbo].[fn_get_predefine_by_key](1002,@BankCodeJPMC))

	SET @CalculateDate = GETDATE()

---------Start: TRY------------------------------------------------------------------------------------------
	BEGIN TRY
		--Start: 1. Prepare Data for move to backup---------------------------------------------------------
		----------Start: 1.1 All BillPayment Final---------------------------------------------------------
		SELECT * 
		INTO #tmpAllBillPaymentFinal 
		FROM bp_billpayment_final
		WHERE create_date < @CalculateDate
		----------End: 1.1 All BillPayment Final------

		----------Start: 1.2 Prepare Staging---------------
		SELECT * 
		INTO #tmpBillPaymentStaging 
		FROM bp_billpayment_staging main
		WHERE create_date < @CalculateDate
		AND (
			(main.pay_status_cd = 3) -- Move Staging with Cancel Status for all case
			OR (NOT EXISTS (SELECT 1
							FROM #tmpAllBillPaymentFinal tmp
							WHERE tmp.bank_cd = main.bank_cd
							AND tmp.pay_date = (CASE WHEN tmp.bank_cd = 1 THEN CONVERT(DATE,main.pay_date,@FormatPayDateBBL)
													WHEN tmp.bank_cd = 2 THEN  TRY_PARSE(main.pay_date AS DATE using @RegionPayDateKBANK)
													WHEN tmp.bank_cd = 3 THEN  TRY_PARSE(main.pay_date AS DATE using @RegionPayDateJPMC)
													WHEN tmp.bank_cd = 4 THEN  CONVERT(DATE,main.pay_date,@FormatPayDateTTB)
												END)
							--AND tmp.pay_date = (CASE WHEN tmp.bank_cd = 1 THEN IIF(main.pay_date ='31/03/2025', CONVERT(DATE,'01/04/2025',@FormatPayDateBBL), CONVERT(DATE,main.pay_date,@FormatPayDateBBL))
							--						WHEN tmp.bank_cd = 2 THEN  IIF(main.pay_date = N'31-มี.ค.-2568',TRY_PARSE(N'01-เม.ย.-2568' AS DATE using @RegionPayDateKBANK),  TRY_PARSE(main.pay_date AS DATE using @RegionPayDateKBANK))
							--						WHEN tmp.bank_cd = 3 THEN  IIF(main.pay_date = N'31-มี.ค.-2568',TRY_PARSE(N'01-เม.ย.-2568' AS DATE using @RegionPayDateJPMC),TRY_PARSE(main.pay_date AS DATE using @RegionPayDateJPMC))
							--						WHEN tmp.bank_cd = 4 THEN  IIF(main.pay_date ='31/03/2025', CONVERT(DATE,'01/04/2025',@FormatPayDateTTB), CONVERT(DATE,main.pay_date,@FormatPayDateTTB))
							--						ELSE NULL
							--						END)
						
							AND tmp.bank_running_no = main.bank_running_no
							AND main.pay_status_cd = 1
							AND (
								ISNULL(tmp.gen_zgenfip_status_cd,2) <>  1  -- Gen ZGENFIP not completed
								OR ISNULL(tmp.sap_status_cd,2) <> 1  -- Posting SAP is not completed
								OR ISNULL(tmp.match_clear_status_cd,'') = ''  -- Match&Clear is not completed
								)
							)
				AND main.pay_status_cd = 1 
				AND main.send_to_final_billpayment_cd = 1
			)
		)
		----------End: 1.2 Prepare Staging-----------------

		----------Start: 1.3 Prepare ZGENFIP--------
		SELECT * 
		INTO #tmpZgenfipFinal 
		FROM bp_zgenfip_final  main
		WHERE EXISTS (SELECT 1
					  FROM #tmpAllBillPaymentFinal tmp
							WHERE tmp.bank_cd = main.bank_cd
							AND tmp.pay_date = main.pay_date
							AND tmp.bank_running_no = main.bank_running_no
		) 
		AND main.[file_name] NOT IN 
		(
			SELECT DISTINCT tmp.gen_zgenfip_filename 
			FROM #tmpAllBillPaymentFinal tmp
			WHERE tmp.pay_status_cd = 1 -- Check Payment is completed
			AND ISNULL(tmp.gen_zgenfip_status_cd,2) = 1  -- Gen ZGENFIP completed
			AND ISNULL(tmp.sap_status_cd,2) = 2 --Wait Posting SAP / Cannot Posting SAP
			
		)
		----------End: 1.3 Prepare ZGENFIP---------

		----------Start: 1.4 Prepare BillPayment FInal---------------------------------------------------------
		SELECT * 
		INTO #tmpBillPaymentFinal 
		FROM #tmpAllBillPaymentFinal  tmp
		WHERE tmp.pay_status_cd = 3
		OR (tmp.pay_status_cd = 1 -- Check Payment is completed
				AND (
						(tmp.sap_status_cd = 1  -- Posting SAP is completed
						AND tmp.gen_zgenfip_status_cd = 1  -- Gen ZGENFIP completed
						AND ISNULL(tmp.match_clear_status_cd,'') <> '' -- Match & Clear is Completed or Not Retail
						)
						OR
						(tmp.sap_status_cd = 3) --Reverse Posting SAP
				)
			)
		----------End: 1.4 All BillPayment Final------

		--End: 1. Prepare Data for move to History---------------------------------------------------------


		--Start: 2. Use Transaction and Move data to History---------------------------------------------------------
		---------Start: 2.1 Begin Transaction-------------
		BEGIN TRANSACTION
		---------End: 2.1 Begin Transaction-------------
		---------Start: 2.2 Insert History-------------
		EXEC [dbo].[usp_bp0202_pending_billpayment_move_history_temp] @CalculateDate
		EXEC [dbo].[usp_bp0202_pending_billpayment_move_history_staging] @CalculateDate
		EXEC [dbo].[usp_bp0202_pending_billpayment_move_history_final] @CalculateDate
		EXEC [dbo].[usp_bp0202_pending_billpayment_move_history_zgenfip] @CalculateDate

		---------Start: 2.2.1 Insert Match & Clear History-------------
		INSERT INTO trx_mc_raw_match_and_clear_history
			(
				process_key
				,process_code
				,account
				,bank_running_no
				,pay_date
				,amount
				,sap_doc_or_no
				,ref_2
				,create_date
				,create_by
				,document_no
				,document_type
				,posting_key
				,posting_date
				,document_date
				,net_due_date
				,gl_account
				,assignment
				,invoice_ref
				,payment_block
				,thb_gross
				,backup_date
				,backup_by
			)
			SELECT
				process_key
				,process_code
				,account
				,bank_running_no
				,pay_date
				,amount
				,sap_doc_or_no
				,ref_2
				,create_date
				,create_by
				,document_no
				,document_type
				,posting_key
				,posting_date
				,document_date
				,net_due_date
				,gl_account
				,assignment
				,invoice_ref
				,payment_block
				,thb_gross
				,@CalculateDate AS backup_date
				,@updated_by AS backup_by
			FROM trx_mc_raw_match_and_clear
			WHERE create_date < @CalculateDate

		INSERT INTO trn_mc_header_history
			(
				process_key
				,pay_date
				,bank_cd
				,bank_running_no
				,ref_1
				,ref_2
				,amount
				,sap_doc_or_no
				,status_cd
				,sap_doc_no
				,error_description
				,create_date
				,create_by
				,update_date
				,update_by
				,backup_date
				,backup_by
			)
			SELECT
				process_key
				,pay_date
				,bank_cd
				,bank_running_no
				,ref_1
				,ref_2
				,amount
				,sap_doc_or_no
				,status_cd
				,sap_doc_no
				,error_description
				,create_date
				,create_by
				,update_date
				,update_by
				,@CalculateDate AS backup_date
				,@updated_by AS backup_by
			FROM trn_mc_header
			WHERE create_date < @CalculateDate

		INSERT INTO trn_mc_detail_history
			(
				process_key
				,pay_date
				,bank_running_no
				,ref_1
				,document_no
				,amount
				,net_due_date
				,thb_gross
				,accum_thb_gross
				,remaining_amt
				,resitem_flag
				,create_date
				,create_by
				,backup_date
				,backup_by
			)
			SELECT
				process_key
				,pay_date
				,bank_running_no
				,ref_1
				,document_no
				,amount
				,net_due_date
				,thb_gross
				,accum_thb_gross
				,remaining_amt
				,resitem_flag
				,create_date
				,create_by
				,@CalculateDate AS backup_date
				,@updated_by AS backup_by
			FROM trn_mc_detail
			WHERE create_date < @CalculateDate

		INSERT INTO trn_mc_othercustomer_header_history
			(
				process_key
				,process_code
				,calculate_date
				,cust_cd
				,cust_name
				,customer_flag
				,offset_flag
				,match_clear_sap_status_cd
				,match_clear_sap_doc_no
				,match_clear_sap_message
				,error_description
				,match_clear_date
				,match_clear_by
				,create_date
				,create_by
				,update_date
				,update_by
				,backup_date
				,backup_by
			)
			SELECT
				process_key
				,process_code
				,calculate_date
				,cust_cd
				,cust_name
				,customer_flag
				,offset_flag
				,match_clear_sap_status_cd
				,match_clear_sap_doc_no
				,match_clear_sap_message
				,error_description
				,match_clear_date
				,match_clear_by
				,create_date
				,create_by
				,update_date
				,update_by
				,@CalculateDate AS backup_date
				,@updated_by AS backup_by
			FROM trn_mc_othercustomer_header
			WHERE create_date < @CalculateDate

		INSERT INTO trn_mc_othercustomer_detail_history
			(
				process_key
				,process_code
				,calculate_date
				,cust_cd
				,document_no
				,reference_no
				,dc_flag
				,total_credit_amt
				,net_due_date
				,thb_gross
				,accum_thb_gross
				,remaining_amt
				,selection_flag
				,resitem_flag
				,create_date
				,create_by
				,backup_date
				,backup_by
			)
			SELECT
				process_key
				,process_code
				,calculate_date
				,cust_cd
				,document_no
				,reference_no
				,dc_flag
				,total_credit_amt
				,net_due_date
				,thb_gross
				,accum_thb_gross
				,remaining_amt
				,selection_flag
				,resitem_flag
				,create_date
				,create_by
				,@CalculateDate AS backup_date
				,@updated_by AS backup_by
			FROM trn_mc_othercustomer_detail
			WHERE create_date < @CalculateDate

		INSERT INTO trx_mc_raw_othercustomer_history
			(
				process_key
				,process_code
				,calculate_date
				,customer_cd
				,document_no
				,customer_flag
				,offset_flag
				,document_type
				,posting_key
				,posting_date
				,document_date
				,net_due_date
				,gl_account
				,reference
				,assignment
				,invoice_ref
				,payment_block
				,thb_gross
				,create_by
				,create_date
				,backup_date
				,backup_by
			)
			SELECT
				process_key
				,process_code
				,calculate_date
				,customer_cd
				,document_no
				,customer_flag
				,offset_flag
				,document_type
				,posting_key
				,posting_date
				,document_date
				,net_due_date
				,gl_account
				,reference
				,assignment
				,invoice_ref
				,payment_block
				,thb_gross
				,create_by
				,create_date
				,@CalculateDate AS backup_date
				,@updated_by AS backup_by
			FROM trx_mc_raw_othercustomer
			WHERE create_date < @CalculateDate
		---------End: 2.2.1 Insert Match & Clear History-------------
		---------End: 2.2 Insert History-------------
	
		---------Start: 2.3 Delete Data-------------
		EXEC [dbo].[usp_bp0202_pending_billpayment_delete_working_all] @CalculateDate

		---------Start: 2.3.4 Delete Match & Clear Data-------------
		DELETE main
		FROM trx_mc_raw_match_and_clear main
		WHERE main.create_date < @CalculateDate

		DELETE main
		FROM trn_mc_header main
		WHERE main.create_date < @CalculateDate

		DELETE main
		FROM trn_mc_detail main
		WHERE main.create_date < @CalculateDate

		DELETE main
		FROM trn_mc_othercustomer_header main
		WHERE main.create_date < @CalculateDate

		DELETE main
		FROM trn_mc_othercustomer_detail main
		WHERE main.create_date < @CalculateDate

		DELETE main
		FROM trx_mc_raw_othercustomer main
		WHERE main.create_date < @CalculateDate
		---------End: 2.3.4 Delete Match & Clear Data-------------
		
		---------End: 2.3 Delete Final Data-------------
		
		
		---------Start: 2.4 Commit Transaction-------------
		IF @@TRANCOUNT > 0
		BEGIN
			COMMIT TRANSACTION
		END	
		---------End: 2.4 Commit Transaction-------------[dbo].[usp_bp0202_pending_billpayment_to_history_staging]
		--End: 2. Use Transaction and Move data to History---------------------------------------------------------
	END TRY
---------End: TRY------------------------------------------------------------------------------------------
---------Start: CATCH------------------------------------------------------------------------------------------
		BEGIN CATCH
		SET @ErrorMessage = ISNULL(ERROR_MESSAGE(),'')
		SET @ErrorSeverity = ERROR_SEVERITY()
		SET @ErrorState = ERROR_STATE()
		IF @@TRANCOUNT > 0
		BEGIN
			ROLLBACK TRANSACTION
		END
		IF OBJECT_ID('tempdb..#tmpBillPaymentStaging') IS NOT NULL DROP TABLE #tmpBillPaymentStaging
		IF OBJECT_ID('tempdb..#tmpAllBillPaymentFinal') IS NOT NULL DROP TABLE #tmpAllBillPaymentFinal
		IF OBJECT_ID('tempdb..#tmpBillPaymentFinal') IS NOT NULL DROP TABLE #tmpBillPaymentFinal
		IF OBJECT_ID('tempdb..#tmpZgenfipFinal') IS NOT NULL DROP TABLE #tmpZgenfipFinal

		RAISERROR(@ErrorMessage,@ErrorSeverity,@ErrorState);
	END CATCH
---------End: CATCH------------------------------------------------------------------------------------------
END
GO
```

## ตัวอย่างการเรียกใช้

proc นี้ย้าย/ลบข้อมูลจริงในตาราง working จึง comment ตัวอย่างไว้ ไม่ให้ runner เรียกอัตโนมัติ

```sql
-- EXEC dbo.usp_bp0202_pending_billpayment_working
--     @process_code = N'...', @step_code = N'...', @updated_by = N'...', @process_key = '...';
```
