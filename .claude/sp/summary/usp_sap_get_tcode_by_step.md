# สรุป Stored Procedure: `dbo.usp_sap_get_tcode_by_step`

| | |
| --- | --- |
| Database | `RPA_DEV` |
| สรุปเมื่อ | 2026-09-23 |
| create_date ใน DB | 09 Sep 2026 11:57:35 |
| modify_date ล่าสุดใน DB | 23 Sep 2026 11:37:08 |

## ภาพรวม

ดึง **ค่า config ของ SAP T-code หนึ่งขั้นตอน (step)** จากตาราง
`dbo.sap_tcode_by_process` เพื่อให้ฝั่ง RPA ใช้ควบคุมการรัน transaction บน SAP
(T-code อะไร, search variant, report layout, timeout, encoding, ไฟล์ config
ที่เกี่ยวข้อง) โดยรับ `@step_code` เข้ามาแล้ว `SELECT` แถวเดียวที่ตรงและยัง active

## Parameters

| Parameter | Type | ความหมาย |
| --- | --- | --- |
| `@step_code` | `nvarchar(10)` | รหัสขั้นตอนของ process ที่ต้องการดึงค่า config SAP T-code |

## ตารางที่เกี่ยวข้อง

- `dbo.sap_tcode_by_process` — ตารางหลักที่ใช้จริง (`WITH (NOLOCK)`), เก็บ config
  ของแต่ละ T-code ต่อ step
- `dbo.sap_print_criteria` — ถูก join ไว้เดิม (`prnt`) แต่ **comment ทิ้งทั้งหมด**
  ตั้งแต่ 10 Dec 2024 (ดูหัวข้อข้อสังเกต)
- `dbo.cfg_configuration_process` — ดึงค่า `company_code` ผ่าน subquery แบบ
  scalar เฉพาะตอนที่ `step_code = 'MC0104'` เท่านั้น (step อื่นได้ `''` เสมอ)
  โดย **ค่าที่ดึงมาจริงคือของ `process_code = 'BP01'`, `step_code = 'BP0107'`**
  (ไม่ใช่ `MC01`/`MC0104`) — คือ "ยืม" company_code ที่ config ไว้แล้วของ step
  `BP0107` มาคืนให้ `MC0104` ใช้

## ตรรกะการทำงาน

```sql
SELECT tcode.step_code, tcode.sap_tcode,
       ISNULL(tcode.sap_search_variance, '')      AS sap_search_variance,
       ISNULL(tcode.sap_report_layout, '')        AS sap_report_layout,
       ISNULL(NULLIF(tcode.sap_timeout, ''), 1)   AS sap_timeout,
       ISNULL(tcode.sap_encoding, '')             AS sap_encoding,
       ISNULL(tcode.file_info_code, '')           AS file_info_code,
       CASE WHEN tcode.step_code = 'MC0104' THEN
            ISNULL((SELECT config_value
                    FROM cfg_configuration_process
                    WHERE process_code = 'BP01'
                      AND step_code = 'BP0107'
                      AND config_code = 'COMPANY_CODE'), '')
       ELSE '' END                                AS company_code
FROM sap_tcode_by_process tcode WITH (NOLOCK)
WHERE tcode.is_active = 1
  AND tcode.step_code = @step_code
```

- คืนค่าว่าง (`''`) แทน `NULL` สำหรับคอลัมน์ optional ทุกตัว (`sap_search_variance`,
  `sap_report_layout`, `sap_encoding`, `file_info_code`)
- `sap_timeout`: ถ้าเป็นค่าว่าง (`''`) จะ fallback เป็น `1` ผ่าน
  `ISNULL(NULLIF(tcode.sap_timeout, ''), 1)`
- filter เฉพาะ `is_active = 1` และ `step_code = @step_code` — ไม่มี filter
  ด้วย `sap_tcode` (บรรทัด `AND tcode.sap_tcode = @sap_tcode` ถูก comment ไว้
  และ parameter นี้ไม่มีอยู่จริงในตัว proc)
- อ่านด้วย `WITH (NOLOCK)` — ยอมรับ dirty read เพื่อความเร็ว

## ข้อสังเกต / จุดเสี่ยง

- `company_code` scope ไว้เฉพาะ step `MC0104` เท่านั้น — เรียกด้วย `step_code`
  อื่นได้ `company_code = ''` เสมอ ไม่ error
- **แหล่งค่า `company_code` ผูกกับ `BP01`/`BP0107` แบบ hard-code** ไม่ใช่ `MC01`/
  `MC0104` — ทดสอบจริงบน `RPA_DEV` (23 Sep 2026 11:37) `EXEC ... @step_code='MC0104'`
  คืน `company_code = 7000` ซึ่งเป็นค่าของ `BP0107` ไม่ใช่ค่าเฉพาะของ `MC0104`
  (ปัจจุบันยังไม่มีแถว config ของ `MC01`/`MC0104` เองใน `cfg_configuration_process`
  เลย จึงยืมค่าของ `BP0107` มาใช้ชั่วคราว) — ควรตรวจทานกับผู้ให้ requirement ว่า
  ตั้งใจใช้ค่าเดียวกับ `BP0107` จริงหรือเป็นการอ้างอิงผิดตัว
- **โค้ดส่วน print criteria ถูก comment ทิ้งทั้งหมด** (ตั้งแต่ 10 Dec 2024, คอมเมนต์
  ระบุ "ke comment 10.12.2024 not use"): เดิม proc นี้เคย `LEFT JOIN`
  `dbo.sap_print_criteria` ผ่าน `print_code` (เงื่อนไข `prnt.is_active = 1`) แล้ว
  คืนคอลัมน์ config การพิมพ์อีกกว่า 20 คอลัมน์ (output device, printer name,
  spool request, cover page ฯลฯ) — ปัจจุบัน **ไม่ได้ใช้งานแล้ว** เหลือแต่ column
  พื้นฐาน 7 คอลัมน์
- ถ้า `step_code` ที่ส่งมาไม่ตรงกับแถวที่ `is_active = 1` เลย จะไม่มี result set
  คืนกลับ (ไม่มี fallback/`SELECT` ว่าง)
- ไม่มี `BEGIN TRY/CATCH`, ไม่มี transaction — เป็น read-only query ล้วน
- `@step_code` ยาวสุด 10 ตัวอักษร แต่คอลัมน์จริงในตาราง `step_code` เป็น
  `nvarchar(20)` — parameter แคบกว่าคอลัมน์ ถ้ามี step_code ที่ยาวเกิน 10 ตัวอักษร
  จะส่งเข้ามาไม่ได้
- **ประวัติ**: คอลัมน์ `company_code` นี้ถูกเพิ่ม/ถอด/เพิ่มกลับหลายรอบในวันเดียวกัน
  (23 Sep 2026: เพิ่ม ~09:51 → ถอดออก ~11:16 → เพิ่มกลับ ~11:37) ควรยืนยัน
  requirement ให้ชัดก่อน deploy จริงเพื่อลดความสับสน
