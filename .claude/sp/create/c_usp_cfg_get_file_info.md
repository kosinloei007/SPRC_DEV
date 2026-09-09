# Create Stored Procedure: `usp_cfg_get_file_info_20290909`

## วัตถุประสงค์

สร้าง stored procedure เวอร์ชันใหม่ (ลงวันที่ `20290909`) ของ
`dbo.usp_cfg_get_file_info` เพื่อดึงข้อมูลการตั้งค่าไฟล์ (working directory,
ชื่อไฟล์, ชื่อ sheet) ของ process หนึ่ง ๆ พร้อมแทนค่า token วันที่ /
process key ลงใน parameter ของ path และชื่อไฟล์

## Parameters

| ชื่อ | ชนิด | คำอธิบาย |
| --- | --- | --- |
| `@process_code` | `NVARCHAR(5)` | รหัส process (อ้างอิง `cfg_file_information.process_code`) |
| `@process_key` | `NVARCHAR(100)` | คีย์ของรอบประมวลผล ใช้แทน token `[ProcessKey]` |

## Token ที่รองรับ (ใน `*_parameter`)

| Token | แทนด้วย |
| --- | --- |
| `[PreviousDay]` | `GETDATE() - 1 วัน` จัดรูปตาม format ใน `{...}` |
| `[CurrentDate]` | วันปัจจุบัน จัดรูปตาม format ใน `{...}` |
| `[FirstDayCurrentMonth]` | วันแรกของเดือนปัจจุบัน จัดรูปตาม format ใน `{...}` |
| `[ProcessKey]` | ค่า `@process_key` |

รูปแบบ token: `[TokenName]{format}` เช่น `[CurrentDate]{yyyyMMdd}`

## แหล่งข้อมูล

- `dbo.cfg_file_information` (WHERE `is_active = 1` และ `process_code` ตรง หรือว่าง)

## Result set

คอลัมน์ที่คืน: `file_info_code`, `file_info_name`, `process_code`,
`company_code`, `is_working_directory`, `working_directory_path`,
`work_file_name`, `work_sheet_name`

## หมายเหตุ

- Logic เหมือน `usp_cfg_get_file_info` เดิมทุกประการ เปลี่ยนเฉพาะชื่อ object
- ใช้ dynamic SQL (`EXEC`) ต่อ `FORMATMESSAGE(...)` เพื่อประกอบ path/ชื่อไฟล์
  จาก `working_dir_path` / `file_name` / `sheet_name` (เป็น format string)
- Clean up temp table ทุกตัวก่อนจบ (`#tmp_file_info`, `#tmp_work_dir`,
  `#tmp_file_name`, `#tmp_sheet_name`, `#tmp_cfg_info`)

---

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.usp_cfg_get_file_info_20290909', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_cfg_get_file_info_20290909
GO

CREATE PROCEDURE [dbo].[usp_cfg_get_file_info_20290909]
    @process_code           NVARCHAR(5),
    @process_key            NVARCHAR(100)
AS
BEGIN

--0. Declare prefix & suffix
DECLARE @date_prefix NVARCHAR(10)       = '['
        ,  @date_suffix NVARCHAR(10)    = ']'
        ,  @format_prefix NVARCHAR(10)  = '{'
        ,  @format_suffix NVARCHAR(10)  = '}'

DECLARE @PreviousDay            NVARCHAR(100) = 'PreviousDay'
        , @ProcessKey           NVARCHAR(100) = 'ProcessKey'
        , @CurrentDate          NVARCHAR(100) = 'CurrentDate'
        , @FirstDayCurrentMonth NVARCHAR(100) = 'FirstDayCurrentMonth'

--1. Declare Date
DECLARE @current_date          DATETIME2 = GETDATE()
DECLARE @previous_date         DATETIME2 = DATEADD(D, -1, @current_date)
DECLARE @firstday_current_month DATETIME2 = DATEADD(month, DATEDIFF(month, 0, @current_date), 0)
DECLARE @sql_string            NVARCHAR(MAX) = ''

SET @ProcessKey = ISNULL(@ProcessKey, '')

--1. Filter master to temp table
SELECT cfg.file_info_code
    , cfg.file_info_name
    , cfg.process_code
    , cfg.is_working_directory
    , cfg.working_dir_path
    , cfg.working_dir_parameter
    , cfg.[file_name]              AS work_file_name
    , cfg.file_name_parameter      AS work_file_name_parameter
    , cfg.[sheet_name]             AS work_sheet_name
    , cfg.[sheet_name_parameter]   AS work_sheet_name_parameter
    , cfg.company_code
INTO #tmp_file_info
FROM cfg_file_information cfg WITH(NOLOCK)
WHERE ISNULL(NULLIF(cfg.process_code, ''), @process_code) = @process_code
AND cfg.is_active = 1

--CROSS JOIN for working_dir
SELECT file_info_code
     , process_code
     , company_code
     , STRING_AGG(CASE LEFT(ISNULL(working_dir_parameter, ''), PATINDEX('%]%', working_dir_parameter))
                        WHEN @date_prefix + @PreviousDay + @date_suffix          THEN FORMAT(@previous_date, working_dir_format)
                        WHEN @date_prefix + @CurrentDate + @date_suffix          THEN FORMAT(@current_date, working_dir_format)
                        WHEN @date_prefix + @FirstDayCurrentMonth + @date_suffix THEN FORMAT(@firstday_current_month, working_dir_format)
                        WHEN @date_prefix + @ProcessKey + @date_suffix           THEN @process_key
                        ELSE working_dir_parameter
                        END, ''',''') working_dir_parameter
INTO #tmp_work_dir
FROM (
        SELECT cfg.file_info_code
            , cfg.company_code
            , cfg.process_code
            , work_dir_param.[value] AS working_dir_parameter
            , CASE WHEN LEN(work_dir_param.[value]) > 0 AND CHARINDEX('{', work_dir_param.[value]) > 0 AND CHARINDEX('}', work_dir_param.[value]) > 0
                        THEN SUBSTRING(work_dir_param.[value], PATINDEX('%{%', work_dir_param.[value]) + 1, (PATINDEX('%}%', work_dir_param.[value]) - PATINDEX('%{%', work_dir_param.[value])) - 1)
                    ELSE '' END AS working_dir_format
        FROM #tmp_file_info cfg WITH(NOLOCK)
        CROSS APPLY STRING_SPLIT(ISNULL(working_dir_parameter, ''), ',') work_dir_param
    ) TAB
GROUP BY file_info_code
     , process_code
     , company_code

--CROSS JOIN for file name
SELECT file_info_code
     , process_code
     , company_code
     , STRING_AGG(CASE LEFT(ISNULL(file_name_parameter, ''), PATINDEX('%]%', file_name_parameter))
                        WHEN @date_prefix + @PreviousDay + @date_suffix          THEN FORMAT(@previous_date, file_name_format)
                        WHEN @date_prefix + @CurrentDate + @date_suffix          THEN FORMAT(@current_date, file_name_format)
                        WHEN @date_prefix + @FirstDayCurrentMonth + @date_suffix THEN FORMAT(@firstday_current_month, file_name_format)
                        WHEN @date_prefix + @ProcessKey + @date_suffix           THEN @process_key
                        ELSE file_name_parameter
                        END, ''',''') file_name_parameter
INTO #tmp_file_name
FROM (
        SELECT cfg.file_info_code
            , cfg.company_code
            , cfg.process_code
            , file_name_param.[value] AS file_name_parameter
            , CASE WHEN LEN(file_name_param.[value]) > 0 AND CHARINDEX('{', file_name_param.[value]) > 0 AND CHARINDEX('}', file_name_param.[value]) > 0
                        THEN SUBSTRING(file_name_param.[value], PATINDEX('%{%', file_name_param.[value]) + 1, (PATINDEX('%}%', file_name_param.[value]) - PATINDEX('%{%', file_name_param.[value])) - 1)
                    ELSE '' END AS file_name_format
        FROM #tmp_file_info cfg WITH(NOLOCK)
        CROSS APPLY STRING_SPLIT(ISNULL(work_file_name_parameter, ''), ',') file_name_param
    ) TAB
GROUP BY file_info_code
     , process_code
     , company_code

--CROSS JOIN for sheet name
SELECT file_info_code
     , process_code
     , company_code
     , STRING_AGG(CASE LEFT(ISNULL(sheet_name_parameter, ''), PATINDEX('%]%', sheet_name_parameter))
                        WHEN @date_prefix + @PreviousDay + @date_suffix          THEN FORMAT(@previous_date, sheet_name_format)
                        WHEN @date_prefix + @CurrentDate + @date_suffix          THEN FORMAT(@current_date, sheet_name_format)
                        WHEN @date_prefix + @FirstDayCurrentMonth + @date_suffix THEN FORMAT(@firstday_current_month, sheet_name_format)
                        WHEN @date_prefix + @ProcessKey + @date_suffix           THEN @process_key
                        ELSE sheet_name_parameter
                        END , ''',''') sheet_name_parameter
INTO #tmp_sheet_name
FROM (
        SELECT cfg.file_info_code
            , cfg.company_code
            , cfg.process_code
            , sheet_name_param.[value] AS sheet_name_parameter
            , CASE WHEN LEN(sheet_name_param.[value]) > 0 AND CHARINDEX('{', sheet_name_param.[value]) > 0 AND CHARINDEX('}', sheet_name_param.[value]) > 0
                        THEN SUBSTRING(sheet_name_param.[value], PATINDEX('%{%', sheet_name_param.[value]) + 1, (PATINDEX('%}%', sheet_name_param.[value]) - PATINDEX('%{%', sheet_name_param.[value])) - 1)
                    ELSE '' END AS sheet_name_format
        FROM #tmp_file_info cfg WITH(NOLOCK)
        CROSS APPLY STRING_SPLIT(ISNULL(work_sheet_name_parameter, ''), ',') sheet_name_param
    ) TAB
GROUP BY file_info_code
     , process_code
     , company_code

SELECT cfg.file_info_code
     , cfg.file_info_name
     , cfg.process_code
     , cfg.company_code
     , cfg.is_working_directory
     , cfg.working_dir_path
     , ISNULL(wk_dir.working_dir_parameter, '')    working_dir_parameter
     , ISNULL(cfg.work_file_name, '')              work_file_name
     , ISNULL(fl_name.file_name_parameter, '')     file_name_parameter
     , ISNULL(cfg.work_sheet_name, '')             work_sheet_name
     , ISNULL(sh_name.sheet_name_parameter, '')    sheet_name_parameter
     , ROW_NUMBER() OVER(ORDER BY cfg.file_info_code, cfg.company_code ASC) rowno
INTO #tmp_cfg_info
FROM #tmp_file_info cfg WITH(NOLOCK)
LEFT JOIN #tmp_work_dir wk_dir WITH(NOLOCK)
ON  cfg.file_info_code = wk_dir.file_info_code
AND cfg.process_code   = wk_dir.process_code
AND cfg.company_code   = wk_dir.company_code
LEFT JOIN #tmp_file_name fl_name WITH(NOLOCK)
ON  cfg.file_info_code = fl_name.file_info_code
AND cfg.process_code   = fl_name.process_code
AND cfg.company_code   = fl_name.company_code
LEFT JOIN #tmp_sheet_name sh_name WITH(NOLOCK)
ON  cfg.file_info_code = sh_name.file_info_code
AND cfg.process_code   = sh_name.process_code
AND cfg.company_code   = sh_name.company_code

DECLARE @cfg_file_infor TABLE
(
    file_info_code            NVARCHAR(100)
    , file_info_name          NVARCHAR(1000)
    , process_code            NVARCHAR(100)
    , company_code            NVARCHAR(100)
    , is_working_directory    BIT
    , working_directory_path  NVARCHAR(1000)
    , work_file_name          NVARCHAR(1000)
    , work_sheet_name         NVARCHAR(1000)
)

SET NOCOUNT ON;

DECLARE @cur_rowno INT

DECLARE cfg_file CURSOR FOR
SELECT rowno
FROM #tmp_cfg_info
ORDER BY rowno;

OPEN cfg_file

FETCH NEXT FROM cfg_file
INTO @cur_rowno

WHILE @@FETCH_STATUS = 0
BEGIN

    SET @sql_string = ''

    SET @sql_string = (SELECT 'SELECT ''' + file_info_code + ''''
                                + ',''' + file_info_name + ''''
                                + ',''' + process_code + ''''
                                + ',''' + company_code + ''''
                                + ',' + CAST(ISNULL(is_working_directory, 0) AS NVARCHAR(2)) + ''
                                + CASE WHEN ISNULL(NULLIF(working_dir_path, ''), '') = '' THEN ', ''' + working_dir_parameter + ''''
                                       ELSE ', FORMATMESSAGE(''' + working_dir_path + ''',''' + working_dir_parameter + ''')' END
                                + CASE WHEN ISNULL(NULLIF(work_file_name, ''), '') = '' THEN ', ''' + file_name_parameter + ''''
                                       ELSE ', FORMATMESSAGE(''' + work_file_name + ''',''' + file_name_parameter + ''')' END
                                + CASE WHEN ISNULL(NULLIF(work_sheet_name, ''), '') = '' THEN ', ''' + sheet_name_parameter + ''''
                                       ELSE ', FORMATMESSAGE(''' + work_sheet_name + ''',''' + sheet_name_parameter + ''')' END
                    FROM #tmp_cfg_info
                    WHERE rowno = @cur_rowno)

    INSERT INTO @cfg_file_infor
    EXEC(@sql_string);

    FETCH NEXT FROM cfg_file
    INTO @cur_rowno
END
CLOSE cfg_file;
DEALLOCATE cfg_file;

--Return Result
SELECT * FROM @cfg_file_infor

DROP TABLE IF EXISTS #tmp_file_info
DROP TABLE IF EXISTS #tmp_work_dir
DROP TABLE IF EXISTS #tmp_file_name
DROP TABLE IF EXISTS #tmp_sheet_name
DROP TABLE IF EXISTS #tmp_cfg_info

END
GO
```

## ตัวอย่างการเรียกใช้

```sql
EXEC dbo.usp_cfg_get_file_info_20290909
     @process_code = 'MC01',
     @process_key  = '20290909-0001';
```
