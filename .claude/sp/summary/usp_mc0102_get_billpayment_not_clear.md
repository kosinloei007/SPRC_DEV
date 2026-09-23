# สรุป Stored Procedure: `dbo.usp_mc0102_get_billpayment_not_clear`

| | |
| --- | --- |
| Database | `RPA_DEV` 
| สรุปเมื่อ | 2026-09-23 |
| ผู้เขียน | Nittaya N. |
| สร้าง | 29 Dec 2024 |
| create_date / modify_date ใน DB | 09 Sep 2026 11:57:35 (เท่ากัน — ยังไม่เคยแก้หลัง deploy ครั้งนี้) |

## ภาพรวม

ดึงรายการ **bill payment ที่ยัง match & clear ไม่สำเร็จ (ของลูกค้า retail
เท่านั้น)** จาก `bp_billpayment_final` มาเตรียมส่งให้ RPA ไปเคลียร์ต่อบน SAP —
พร้อมกับ "ปิดเคส" ลูกค้าที่ไม่ใช่ retail ทิ้งไปเลย (ไม่ต้อง manual/RPA clear)

## Parameters

| Parameter | Type | ความหมาย |
| --- | --- | --- |
| `@process_key` | `varchar(40)` | คีย์ของรอบประมวลผล ใช้บันทึกลง `trn_mc_header` และดึงผลกลับ |
| `@process_code` | `varchar(5)` | **รับเข้ามาแต่ไม่ได้ใช้งานในตัว query เลย** |
| `@update_by` | `nvarchar(20)` | ผู้ทำรายการ ใช้ตอน update `bp_billpayment_final` และ insert `trn_mc_header` |

ตัวอย่างการเรียกจากคอมเมนต์ในสคริปต์: `EXEC usp_mc0102_get_billpayment_not_clear 'Ke1','MC01','KRPA'`

## ตารางที่เกี่ยวข้อง

- `dbo.bp_billpayment_final` — แหล่งข้อมูลหลัก, ถูกทั้ง `UPDATE` (step 2) และ
  `SELECT` เข้า (step 3)
- `dbo.mas_customer_sales` — master แยกลูกค้า retail/non-retail (`is_retail`)
- `dbo.trn_mc_header` — ตารางปลายทาง: `DELETE` ของเดิมทิ้งก่อน แล้ว `INSERT` ใหม่
  ทุกครั้งที่เรียกด้วย `process_key` เดิม (ทำให้ re-run ได้)

## ขั้นตอนการทำงาน

**STEP 1 — เคลียร์ของเก่า**
`DELETE FROM trn_mc_header WHERE process_key = @process_key` (ถ้ามี) — กันข้อมูลซ้ำเวลา re-run

**STEP 2 — "ปิดเคส" ลูกค้าที่ไม่ใช่ retail**
```sql
UPDATE bp_billpayment_final a
LEFT JOIN mas_customer_sales b ON a.ref_1 = b.customer
SET match_clear_status_cd  = CASE WHEN b.is_retail = 1 THEN NULL ELSE 9 END,
    match_clear_date       = CASE WHEN b.is_retail = 1 THEN NULL ELSE GETDATE() END,
    match_clear_by         = CASE WHEN b.is_retail = 1 THEN NULL ELSE @update_by END,
    match_clear_sap_doc_no = '',
    update_date            = CASE WHEN b.is_retail = 1 THEN NULL ELSE GETDATE() END,
    update_by              = CASE WHEN b.is_retail = 1 THEN NULL ELSE @update_by END
WHERE ISNULL(sap_doc_or_no,0) NOT IN (0, '')
  AND sap_status_cd IS NOT NULL
  AND ISNULL(match_clear_status_cd,'') = ''
```
- `LEFT JOIN` ทำให้ครอบคลุมทั้งลูกค้าที่ไม่ใช่ retail (`is_retail = 0`) **และ**
  ลูกค้าที่ไม่พบใน `mas_customer_sales` เลย (`b.is_retail IS NULL` → เข้า `ELSE`
  เหมือนกัน) → ถูกตั้ง `match_clear_status_cd = 9` (ปิดเคส ไม่ต้อง clear)
- เงื่อนไข `CASE WHEN is_retail = 1 THEN NULL` หมายถึง "ถ้าเป็น retail ไม่ต้อง
  แตะแถวนั้น" (`SET` เป็น `NULL` แต่แถวนั้นจะไม่ถูกดึงมาทำต่อใน STEP 3 อยู่แล้ว
  เพราะมี filter `match_clear_status_cd` ว่างซ้ำอีกชั้น — ผลลัพธ์สุทธิคือลูกค้า
  retail ไม่ถูก UPDATE จริง เพราะเงื่อนไข `WHERE` ตรวจแค่ก่อน UPDATE ครั้งเดียว)
- `sap_doc_or_no` ต้องมีค่า (ไม่ใช่ `0`/`''`) และ `sap_status_cd` ต้องไม่ใช่
  `NULL` (แปลว่าผ่านขั้นตอนส่ง SAP มาแล้ว) และยังไม่เคยตั้ง `match_clear_status_cd`

**STEP 3 — ดึงเฉพาะ retail มา insert ลง `trn_mc_header`**
```sql
INSERT INTO trn_mc_header (process_key, pay_date, bank_cd, bank_running_no,
                            ref_1, ref_2, amount, sap_doc_or_no,
                            create_date, create_by)
SELECT @process_key, pay_date, bank_cd, bank_running_no, ref_1, ref_2,
       amount, sap_doc_or_no, GETDATE(), @update_by
FROM bp_billpayment_final a
INNER JOIN mas_customer_sales b ON a.ref_1 = b.customer AND is_retail = 1
WHERE ISNULL(sap_doc_or_no,0) NOT IN (0, '')
  AND sap_status_cd IS NOT NULL
  AND ISNULL(match_clear_status_cd,'') = ''
ORDER BY ref_1
```
- `INNER JOIN ... is_retail = 1` → เฉพาะลูกค้า retail เท่านั้นที่ถูกดึงเข้าคิว
  ให้ไป match & clear ต่อ (ลูกค้าที่ถูกปิดเคสใน STEP 2 จะไม่ผ่าน filter
  `match_clear_status_cd` ว่างอีกต่อไป)

**STEP 4 — คืนผลลัพธ์**
```sql
SELECT ref_1, bank_cd, bank_running_no,
       CONVERT(nvarchar(8), pay_date, 112) AS payment_date
FROM trn_mc_header
WHERE process_key = @process_key
ORDER BY ref_1
```

**TRY/CATCH**: ถ้า error → `ROLLBACK` (ถ้ามี transaction เปิดอยู่) แล้ว
`RAISERROR` re-throw กลับผู้เรียกด้วย severity/state เดิม

## ข้อสังเกต / จุดเสี่ยง

- **`@process_code` ประกาศไว้แต่ไม่ได้ใช้งานจริงในตัว proc เลย**
- **ไม่มี `BEGIN TRANSACTION` ในตัว proc เอง** — โค้ดเช็ค `IF @@TRANCOUNT > 0
  COMMIT/ROLLBACK` ซึ่งจะทำงานจริงก็ต่อเมื่อ "ผู้เรียก" เปิด transaction ครอบมา
  ก่อนแล้วเท่านั้น ถ้าเรียกแบบ `EXEC` เดี่ยว ๆ (ตามตัวอย่างในคอมเมนต์)
  `@@TRANCOUNT` จะเป็น 0 เสมอ → ทั้ง `DELETE` (step 1), `UPDATE` (step 2), และ
  `INSERT` (step 3) จะ **ไม่อยู่ใน transaction เดียวกัน** ถ้า error เกิดระหว่างทาง
  ข้อมูลบางส่วนอาจถูกเปลี่ยนไปแล้วโดยไม่ rollback
- **STEP 2 อัปเดตลูกค้า non-retail แบบ "ปิดเคสถาวร"** (`match_clear_status_cd = 9`)
  — ทำทุกครั้งที่เรียก proc นี้ ถ้าเงื่อนไขตรง ไม่ใช่แค่ query ประกอบ แต่เป็น
  side-effect ถาวรบน `bp_billpayment_final`
- `ISNULL(a.sap_doc_or_no,0) <> '0'` เทียบ `int` (`0`) กับ `varchar` ผ่าน implicit
  convert แล้วเทียบกับ string `'0'` — ทำงานได้เพราะ SQL Server auto-convert
  แต่เป็น pattern ที่สับสน (mix ชนิดข้อมูลใน `ISNULL`)
- เงื่อนไข `WHERE a.pay_date = CONVERT(DATE,GETDATE())` ถูก comment ไว้ทั้ง 2 จุด
  (มีคอมเมนต์ "TODO -3 fortest") — ปัจจุบันดึงข้อมูล**ทุกวันที่**ที่ยังไม่ clear
  ไม่ได้จำกัดเฉพาะวันนี้
- ไม่มี index hint / `NOLOCK` เหมือน proc อื่นในชุดนี้ — อ่าน/เขียนตรงแบบ
  default isolation level
