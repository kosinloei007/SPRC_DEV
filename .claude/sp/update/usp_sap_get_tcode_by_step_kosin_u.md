# Update Stored Procedure: `usp_sap_get_tcode_by_step_kosin`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@step_code` | `NVARCHAR(10)` |

## รายละเอียดการแก้ไข

ก็อป logic จาก `usp_sap_get_tcode_by_step` (ดู
`.claude/sp/summary/usp_sap_get_tcode_by_step.md`) มาใช้กับ proc ทดสอบตัวนี้
แทนที่ scratch body (`SELECT 1`) เดิม:

- ดึงค่า config ของ SAP T-code หนึ่ง step จาก `dbo.sap_tcode_by_process`
  (T-code, search variant, report layout, timeout, encoding, file_info_code)
  ตาม `@step_code` เฉพาะแถวที่ `is_active = 1`
- เพิ่มคอลัมน์ `company_code` — คืนค่าเฉพาะตอน `step_code = 'MC0104'` เท่านั้น
  (step อื่นได้ `''` เสมอ) โดยดึงจาก `dbo.cfg_configuration_process` ของ
  `process_code = 'BP01'`, `step_code = 'BP0107'`, `config_code = 'COMPANY_CODE'`
  (ยืมค่า company_code ที่ config ไว้แล้วของ step `BP0107` มาใช้กับ `MC0104`)

## Script

```sql
USE [RPA_DEV]
GO

ALTER PROCEDURE [dbo].[usp_sap_get_tcode_by_step_kosin]
    @step_code NVARCHAR(10)
AS
BEGIN

    SELECT tcode.step_code
         , tcode.sap_tcode
         , ISNULL(tcode.sap_search_variance, '')    AS sap_search_variance
         , ISNULL(tcode.sap_report_layout, '')      AS sap_report_layout
         , ISNULL(NULLIF(tcode.sap_timeout, ''), 1) AS sap_timeout
         , ISNULL(tcode.sap_encoding, '')           AS sap_encoding
         , ISNULL(tcode.file_info_code, '')         AS file_info_code
         , CASE WHEN tcode.step_code = 'MC0104' THEN
                ISNULL((SELECT config_value
                        FROM cfg_configuration_process
                        WHERE process_code = 'BP01'
                          AND step_code = 'BP0107'
                          AND config_code = 'COMPANY_CODE'), '')
           ELSE ''
           END AS company_code
    FROM sap_tcode_by_process tcode WITH (NOLOCK)
    WHERE tcode.is_active = 1
      AND tcode.step_code = @step_code

END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_sap_get_tcode_by_step_kosin @step_code = 'MC0104';
EXEC dbo.usp_sap_get_tcode_by_step_kosin @step_code = 'BP0107';
```
