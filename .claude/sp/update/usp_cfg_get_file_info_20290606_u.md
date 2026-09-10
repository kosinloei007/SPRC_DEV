# Update Stored Procedure: `usp_cfg_get_file_info_20290606`

## Parameters

| ชื่อ | ชนิด |
| --- | --- |
| `@process_code` | `NVARCHAR(5)` |
| `@process_key` | `NVARCHAR(100)` |

## รายละเอียดการแก้ไข

<ผู้ใช้แก้ส่วนนี้ทุกครั้งที่มี requirement ใหม่ — อธิบายสิ่งที่เปลี่ยน>

ปัจจุบัน: `SELECT` ข้อมูลทุก field จากตาราง `dbo.cfg_file_information`
ตามเงื่อนไข `is_active = 0` และ
`ISNULL(NULLIF(cfg.process_code, ''), '') = @process_code`.

