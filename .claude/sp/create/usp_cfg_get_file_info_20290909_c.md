# Create Stored Procedure: `usp_cfg_get_file_info_20290909`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_code` | `NVARCHAR(5)` |
| `@process_key` | `NVARCHAR(100)` |

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_cfg_get_file_info_20290909', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_cfg_get_file_info_20290909
GO

CREATE PROCEDURE [dbo].[usp_cfg_get_file_info_20290909]
    @process_code   NVARCHAR(5),
    @process_key    NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 1;
END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_cfg_get_file_info_20290909
     @process_code = 'MC01',
     @process_key  = '20290909-0001';
```
