# Create Stored Procedure: `usp_sap_get_tcode_by_step_kosin`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@param` | `NVARCHAR(10)` |

## Requirement (initial)

Scratch/test proc — ยังไม่มี requirement เฉพาะ จะใส่ logic จริงทีหลังผ่าน `/sp-update`

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_sap_get_tcode_by_step_kosin', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_sap_get_tcode_by_step_kosin
GO

CREATE PROCEDURE [dbo].[usp_sap_get_tcode_by_step_kosin]
    @param NVARCHAR(10)
AS
BEGIN
    SET NOCOUNT ON;

    -- initial implementation
    SELECT 1;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_sap_get_tcode_by_step_kosin @param = 'test';
```
