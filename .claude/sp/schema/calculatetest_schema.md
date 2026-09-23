# Table Schema: `calculatetest`

## ที่มา

สร้างจากภาพตัวอย่างรายงาน `.claude/img/calculate.png` — รายงานผลการคำนวณ
match & clear รายลูกค้า

## Columns

| ชื่อ Column | ชนิด | คำอธิบาย | ตัวอย่างข้อมูล |
| --- | --- | --- | --- |
| `id` | `INT IDENTITY(1,1) PRIMARY KEY` | Surrogate key | `1` |
| `calculate_date` | `DATE` | วันที่คำนวณ | `2026-09-11` |
| `customer_code` | `VARCHAR(20)` | รหัสลูกค้า | `8434535` |
| `customer_name` | `VARCHAR(200)` | ชื่อลูกค้า | `PHUKET PHOONPHO...` |
| `match_clear_code` | `VARCHAR(30)` | รหัสอ้างอิง match & clear | `MC202609110001` |
| `match_clear_status` | `VARCHAR(20)` | สถานะ match & clear | `Offset`, `Can't Offset` |
| `match_clear_date` | `DATETIME` | วันเวลาที่ทำ match & clear | `2026-09-11 13:15:20` |
| `match_clear_by` | `VARCHAR(30)` | ผู้/ระบบที่ทำรายการ | `SPRCRPABOT01` |

## Script

```sql
USE [RPA_DEV]
GO

IF OBJECT_ID('dbo.calculatetest', 'U') IS NOT NULL
    DROP TABLE dbo.calculatetest
GO

CREATE TABLE dbo.calculatetest
(
    id                   INT IDENTITY(1,1) PRIMARY KEY,
    calculate_date       DATE          NOT NULL,
    customer_code        VARCHAR(20)   NOT NULL,
    customer_name        VARCHAR(200)  NULL,
    match_clear_code     VARCHAR(30)   NULL,
    match_clear_status   VARCHAR(20)   NULL,
    match_clear_date     DATETIME      NULL,
    match_clear_by       VARCHAR(30)   NULL
)
GO
```

## ตัวอย่างข้อมูล (จากภาพ)

```sql
INSERT INTO dbo.calculatetest
    (calculate_date, customer_code, customer_name, match_clear_code, match_clear_status, match_clear_date, match_clear_by)
VALUES
    ('2026-09-11', '8434535', 'PHUKET PHOONPHO',      'MC202609110001', 'Offset',       '2026-09-11 13:15:20', 'SPRCRPABOT01'),
    ('2026-09-11', '8453007', 'WINNER STAR PETR',     'MC202609110002', 'Offset',       '2026-09-11 13:15:35', 'SPRCRPABOT01'),
    ('2026-09-11', '8237602', 'P.S. OIL SERVICE CO',  'MC202609110003', 'Can''t Offset', '2026-09-11 13:16:02', 'SPRCRPABOT01');
```
