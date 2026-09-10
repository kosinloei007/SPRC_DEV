# Create Stored Procedure: `usp_kosintest`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| _(none)_ | |

## Requirement (initial)

Throwaway test procedure. Returns a single constant column (`SELECT 1;`).

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_kosintest', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_kosintest
GO

CREATE PROCEDURE [dbo].[usp_kosintest]
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
EXEC dbo.usp_kosintest;
```
