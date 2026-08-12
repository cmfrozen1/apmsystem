# SPEC — ข้อกำหนดระบบ apmsystem

ระบบส่งงานส่งเสริมและการเกษตร ของ **บริษัทเชียงใหม่โฟรเซ่นฟูดส์ จำกัด (มหาชน)**

เวอร์ชันสเปก: 1.0 (อัปเดตล่าสุด: สิงหาคม 2026)

---

## 1. บทนำ

### 1.1 วัตถุประสงค์
พัฒนาระบบบันทึกการส่งงานของเจ้าหน้าที่ส่งเสริมการเกษตร (Promoter), Collector และชุปเปอร์ไวเซอร์ (Supervisor)
เพื่อแทนที่การส่งงานด้วยเอกสารกระดาษ ให้สามารถบันทึกงานภาคสนามผ่านมือถือได้ทันที พร้อมข้อมูลพิกัด (GPS) และรูปภาพประกอบ

### 1.2 กลุ่มผู้ใช้งาน
| บทบาท | หน้าที่ | หน้าหลัก |
|---|---|---|
| เจ้าหน้าที่ส่งเสริม (Promoter) | ส่งงานรายวัน — บันทึกการปฏิบัติงานภาคสนาม | `promoteagriculture.html` |
| ชุปเปอร์ไวเซอร์ (Supervisor) | กำกับดูแล — บันทึกรายงานการกำกับดูแล, ตรวจสอบข้อมูลส่งงาน | `suppervisor.html`, `supervisortime.html` |
| Admin | จัดการรายชื่อ ส่งเสริม / Collector / ชุปเปอร์ไวเซอร์ | `adminapm.html` |
| ผู้บริหาร/หน่วยงาน | ดูรายงานสรุปการส่งงาน | `showdata.html` |

### 1.3 สภาพแวดล้อม
- **ผู้ใช้:** เปิดผ่าน LINE (LIFF) บนมือถือเป็นหลัก, เปิดบนเบราว์เซอร์คอมพิวเตอร์ได้
- **Frontend:** HTML5 + CSS3 + Vanilla JavaScript (ไม่ใช้ framework)
- **Backend:** Google Apps Script (GAS) Web App + Google Sheets เป็นฐานข้อมูล
- **โฮสต์:** GitHub Pages (repo `cmfrozen1/apmsystem`, สาขา `main`)
- **การยืนยันตัวตน:** LINE LIFF (`liff.init`) — ระบบเชื่อว่าผู้ใช้ที่เปิดจาก LINE ผ่านการยืนยันแล้ว

---

## 2. สถาปัตยกรรม

```
┌──────────┐    LIFF (LINE)     ┌─────────────────┐
│  LINE    │ ────────────────▶ │  index.html     │  ตัวนำทาง: อ่าน ?linkurl= → redirect
└──────────┘                   └────────┬────────┘
                                        ▼
               ┌──────────┬─────────────┼─────────────┬───────────┐
               ▼          ▼             ▼             ▼           ▼
         ส่งงาน      ชุปเปอร์ฯ      admin       เช็คส่งงาน    รายงาน/ถาม
   promoteagriculture suppervisor  adminapm  supervisortime  showdata/keyword
               │          │             │            │
               └──────────┴──────┬──────┴────────────┘
                                 ▼
                   Google Apps Script (REST /exec)
                                 ▼
                          Google Sheets
```

### 2.1 การไหลของข้อมูล (Data Flow)
1. ผู้ใช้กดเมนูใน LINE → เปิด `index.html?linkurl=<หน้าเป้าหมาย>` → ระบบ init LIFF แล้ว redirect
2. หน้าเป้าหมายเรียก GAS ด้วย HTTP GET (`?action=...`) เพื่อโหลดข้อมูลรายการเลือก (select) หรือ POST เพื่อบันทึกข้อมูล
3. GAS อ่าน/เขียน Google Sheets แล้วคืนค่า JSON
4. ฝั่ง Frontend แสดงผล หรือแจ้งผลสำเร็จ/ข้อผิดพลาด

### 2.2 ข้อกำหนดด้านเทคนิคหลัก
- **GAS response** เป็น JSON เสมอ (GAS คืน header `text/html` — Frontend ต้อง parse เนื้อหาเอง)
- **Timeout:** 60 วินาที (GAS มี cold start อาจใช้เวลา 40+ วิ เมื่อไม่ถูกเรียกนาน)
- **แคชข้อมูลเลือก (dropdown):** localStorage อายุ 12 ชม. — โหลดจากแคชทันที แล้วดึงสดอัปเดตเบื้องหลัง
- **การอัปโหลดรูป:** บีบอัดฝั่ง client ก่อนส่ง (Canvas) — ด้านยาวสูงสุด 900px, คุณภาพ JPEG 0.6, ไฟล์ ≤ 150KB ส่งเดิม
- **GPS:** ตรวจสอบตำแหน่ง (Geolocation) ก่อนเปิดฟอร์ม — ฟอร์มจะซ่อนอยู่จนกว่า GPS พร้อม

---

## 3. ข้อกำหนดฟังก์ชันรายหน้า (Functional Requirements)

### 3.1 `index.html` — หน้าเริ่มต้น (Router)
**FR-01** ระบบต้อง init LIFF ด้วย LIFF ID `2007528760-v8N7EXG1`
**FR-02** ระบบต้องอ่าน query param `linkurl` และ redirect ไปยัง URL นั้นอัตโนมัติ (หน่วง 800ms)
**FR-03** ถ้าไม่มี `linkurl` → แสดงข้อความ "ไม่พบลิงก์ปลายทาง" พร้อมปุ่มกลับ
**FR-04** ถ้าเปิดนอก LINE (LIFF init fail) → ยังต้อง redirect ต่อไปได้
**FR-05** ต้องปิดหน้าต่าง LIFF เองอัตโนมัติหลัง 50 วิ (กันค้างใน LINE)

### 3.2 `promoteagriculture.html` — ฟอร์มส่งงานส่งเสริมการเกษตร
**FR-10** ตรวจสอบ GPS ก่อนแสดงฟอร์ม (มีปุ่ม "ลองอีกครั้ง" ถ้าไม่สำเร็จ)
**FR-11** วันที่ปฏิบัติงาน = วันที่ปัจจุบัน อัตโนมัติ แก้ไขไม่ได้ (readonly)
**FR-12** ช่องเลือก: ชื่อส่งเสริม, Collector, ชนิดพืช — โหลดจาก GAS, ใช้ Tom Select (ค้นหาได้), แคช 12 ชม.
**FR-13** ช่องกรอก: อำเภอ*, ผู้ติดต่อ*, ชนิดพืช*, "อื่นๆ ถ้ามี", รายละเอียดการปฏิบัติงาน*
**FR-14** ฟิลด์ * = จำเป็น (required)
**FR-15** แผนที่แสดงพิกัดตำแหน่งปัจจุบัน (อัตโนมัติจาก GPS)
**FR-16** อัปโหลดรูป 1 รูป (จำเป็น) — บีบอัดอัตโนมัติก่อนส่ง, แสดงตัวอย่างรูปก่อนบันทึก
**FR-17** กด "บันทึกการส่งงาน" → ส่งข้อมูล + รูป (base64) ไปยัง GAS → แจ้งผลสำเร็จ/ผิดพลาด

**ฟิลด์ข้อมูลที่ส่ง:**
| ฟิลด์ | ชื่อ field | ประเภท | จำเป็น |
|---|---|---|---|
| วันที่ปฏิบัติงาน | `date` | date (อัตโนมัติ) | ✅ |
| ชื่อส่งเสริม | `firstName` | select | ✅ |
| Collector | `collector` | select | ✅ |
| อำเภอ | `district` | text | ✅ |
| ผู้ติดต่อ | `contact` | text | ✅ |
| ชนิดพืช | `plantType` | select | ✅ |
| พืชอื่นๆ | `other` | text | – |
| รายละเอียด | `details` | textarea | ✅ |
| พิกัด GPS | `coords` (lat/lng) | text | ✅ |
| รูปภาพ | `imageFile` | base64 | ✅ |

### 3.3 `suppervisor.html` — ฟอร์มรายงานการกำกับดูแล
**FR-20** ตรวจสอบ GPS ก่อนแสดงฟอร์ม (เหมือน 3.2)
**FR-21** วันที่ปฏิบัติงาน = ปัจจุบัน อัตโนมัติ แก้ไขไม่ได้
**FR-22** เลือกชื่อชุปเปอร์ไวเซอร์ + เขต → ระบบดึงข้อมูลพื้นที่ (หมู่บ้าน, ตำบล) **อัตโนมัติ** (`getAutoData`) มาเติมให้ (readonly)
**FR-23** กรอกรายละเอียดปฏิบัติงาน*, พิกัดอัตโนมัติ, อัปโหลดรูป (จำเป็น, บีบอัดอัตโนมัติ)
**FR-24** กด "บันทึกการกำกับดูแล" → ส่งข้อมูลไป GAS

**ฟิลด์ข้อมูลที่ส่ง:** `date`, `firstName`, `district`, `village`, `subDistrict`, `details`, `coords`, `imageFile`

### 3.4 `adminapm.html` — หน้าจัดการข้อมูล (Admin)
**FR-30** ต้องกรอกรหัสผ่านก่อนเข้าถึง (`admin` — ฝังในโค้ดฝั่ง client)
**FR-31** แสดง 3 แท็บ: **ส่งเสริม / Collector / ชุปเปอร์ไวเซอร์** — แต่ละแท็บมีตารางรายชื่อ + ช่องเพิ่มชื่อ
**FR-32** ปุ่มเพิ่ม → POST `{action: add<Type>, name}` → แจ้งผล
**FR-33** ปุ่มแก้ไข (ต่อแถว) → dialog ใส่ชื่อใหม่ → POST `{action: edit<Type>, row, name}`
**FR-34** ปุ่มลบ (ต่อแถว) → ยืนยันก่อนลบ → POST `{action: delete<Type>, row}`
**FR-35** โหลดรายชื่อจาก GAS แบบขนาน 3 ชุด + แคช localStorage 12 ชม. + timeout 60 วิ + retry 2 ครั้ง
**FR-36** โชว์ "กำลังโหลดข้อมูล..." ระหว่างโหลดครั้งแรก (ไม่มีตัวเลขต่อท้าย)
**FR-37** แสดงตารางทีละแท็บทันทีที่ข้อมูลพร้อม (ไม่ต้องรอครบ 3)
**FR-38** UI ตามขนาดจอ:
- จอใหญ่ (> 640px): แท็บอยู่**ด้านบน** เหนือตาราง
- มือถือ (≤ 640px): แท็บเป็น**เมนูลอยติดขอบจอด้านล่าง**แบบ Mobile app (ไอคอนบน-ข้อความล่าง, มี badge จำนวนข้อมูล, เผื่อ safe-area iPhone)

**Actions:**
| Action | Method | Body | ผลลัพธ์ |
|---|---|---|---|
| `getPromoterNames` / `getCollectorNames` / `getSupervisorNames` | GET | – | array รายชื่อ |
| `addPromoter` / `addCollector` / `addSupervisor` | POST | `{action, name}` | `{status:'success'}` |
| `editPromoter` / `editCollector` / `editSupervisor` | POST | `{action, row, name}` | `{status:'success'}` |
| `deletePromoter` / `deleteCollector` / `deleteSupervisor` | POST | `{action, row}` | `{status:'success'}` |

### 3.5 `supervisortime.html` — ระบบเช็คข้อมูลการส่งงานประจำวัน
**FR-40** โหลดตาราง 2 ชุดใน iframe: ชุปเปอร์ไวเซอร์ และ เจ้าหน้าที่ส่งเสริมฯ (GAS Web App โดยตรง)
**FR-41** แถบแท็บด้านล่าง (Bottom Tab Bar) สลับระหว่าง 2 ตาราง
**FR-42** แสดง animation "กำลังโหลดข้อมูล" ขณะ iframe โหลด

### 3.6 `showdata.html` — หน้ารายงานการส่งเสริมการเกษตร
**FR-50** ดึงข้อมูลทั้งหมดจาก GAS (`getData`) แล้วแสดงในตาราง 13 คอลัมน์
**FR-51** แสดงสถิติสรุป (จำนวนงาน, ยอดรวม ฯลฯ) ด้านบนตาราง
**FR-52** ค้นหา/กรองข้อมูล (debounce 250ms)
**FR-53** คลิกที่รูป → เปิด lightbox ดูรูปขนาดใหญ่
**FR-54** ปุ่มรีเฟรช + แสดงเวลาอัปเดตล่าสุด ("อัปเดตล่าสุด: ...")

### 3.7 `keyword.html` — หน้าเลือก Keyword
**FR-60** init LIFF แล้วโหลดรายการ Keyword จาก GAS
**FR-61** ผู้ใช้เลือก Keyword → ส่งต่อไปยังบอท/ระบบถาม-ตอบ

---

## 4. ข้อกำหนด API (Backend — GAS)

### 4.1 URL รูปแบบทั่วไป
```
GET  https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec?action=<ACTION>
POST https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec   (body: JSON {action, ...})
```

### 4.2 ตาราง Deployment ID ต่อหน้า
| หน้า | Deployment ID (ส่วนท้าย URL) |
|---|---|
| ส่งงาน (`promoteagriculture`) | `AKfycbzniYO49YaS_3gv-V1WoLakPP14j6D9HScO6XCMDMzl3LdkADSI_RN1gVpaygghwdLD` |
| ชุปเปอร์ไวเซอร์ (`suppervisor`) | `AKfycbyE8tvOVYAAR6WOiCVDEb2Fp1Yq3ebTAri9ZQYonpduMjQCFzod8xi0glBVXc6A6_qOeA` |
| จัดการข้อมูล (`adminapm`) | `AKfycbwJ6CKgir6gaKJGRs5_IU8HVaqJ91FfZCbuE0ZO4CNr73H47gDrK0bV0KCMpvtrD6s` |
| เช็คส่งงาน ชุปเปอร์ฯ (`supervisortime`) | `AKfycbwASVje2CA6B-YxpusKCSw0n8VVQoSYFFy3GyH8cV9asP_zHHd8WZXSOR3dKOAiOmsvWg` |
| เช็คส่งงาน ส่งเสริมฯ (`supervisortime`) | `AKfycbyM1BUQOxd6XS_SsxKyEnQ-8MOaQUp7tkOn11qchlDvEDcw7oC0VwjxEl_DAAzQg1J11A` |
| รายงาน (`showdata`) | `AKfycbx71XodbTvqXjFclnOgjfFEm46nDuIYO7TQoX6_tesfztpAennb4YEJbraCNsvvU3Oz` |
| Keyword (`keyword`) | `AKfycbxzhmCEXRd0sqYTAkqFA-X2HvV773QoGIdndRuEcDUqi2kXrqHkdjEMCAUUx-cTKyLg1g` |

### 4.3 ตาราง Action รวม
| Action | ใช้โดย | ประเภท | คำอธิบาย |
|---|---|---|---|
| `getPromoterNames` | ส่งงาน / ชุปเปอร์ฯ / admin | GET | รายชื่อเจ้าหน้าที่ส่งเสริม |
| `getCollectors` | ส่งงาน | GET | รายชื่อ Collector |
| `getCollectorNames` | admin | GET | รายชื่อ Collector (admin) |
| `getPlantTypes` | ส่งงาน | GET | ชนิดพืช |
| `getDistricts` | ชุปเปอร์ฯ | GET | รายชื่อเขต |
| `getAutoData` | ชุปเปอร์ฯ | GET | ข้อมูลอัตโนมัติตามเขต (`?district=`) |
| `getSupervisorNames` | admin | GET | รายชื่อชุปเปอร์ไวเซอร์ |
| `add<Type>` | admin | POST | เพิ่มชื่อ (`{action, name}`) |
| `edit<Type>` | admin | POST | แก้ชื่อ (`{action, row, name}`) |
| `delete<Type>` | admin | POST | ลบชื่อ (`{action, row}`) |
| `getData` | รายงาน | GET | ดึงข้อมูลส่งงานทั้งหมด |
| `save...` / upload | ส่งงาน / ชุปเปอร์ฯ | POST | บันทึกงาน + รูป |

> `<Type>` = `Promoter` | `Collector` | `Supervisor`

### 4.4 ข้อกำหนดการตอบกลับ (Response Contract)
- สำเร็จ: JSON array (รายชื่อ) หรือ `{"status":"success", ...}`
- ผิดพลาด: `{"status":"error", "message":"..."}` หรือ `{"error":"..."}`
- ต้องไม่ตอบหน้า HTML (ถ้าเป็น HTML = deployment ยังไม่เปิดสาธารณะ/ต้องล็อกอิน)

### 4.5 ข้อกำหนดการจัดการข้อผิดพลาด (Error Handling)
| สถานการณ์ | การตรวจจับ | การตอบสนอง |
|---|---|---|
| Cold start ช้า | เกิน 8 วิ ยังไม่ตอบ (ไม่มีแคช) | toast "กำลังโหลดข้อมูล... API กำลังตอบช้า" |
| หมดเวลา | abort หลัง 60 วิ | แจ้ง "API ตอบช้าเกินไป" / retry (admin) |
| ตอบเป็น HTML | `text[0] === '<'` | แจ้งให้เปิด deployment เป็น Anyone |
| HTTP 403 | status 403 | แจ้ง "API ยังไม่เปิดสาธารณะ" |
| HTTP 404 | status 404 | แจ้ง "ไม่พบ API — ตรวจ URL/deployment" |
| HTTP 408/429/5xx | status ในชุด | retry แบบ backoff (1s, 2s) — เฉพาะ admin |
| ตอบว่าง / JSON ผิด | parse fail | แจ้ง error ที่อ่านเข้าใจ |

---

## 5. ข้อกำหนดที่ไม่ใช่ฟังก์ชัน (Non-Functional Requirements)

### 5.1 ประสิทธิภาพ
- **NFR-01** เปิดหน้ารอบที่ 2 ขึ้นไป (มีแคช) ต้องแสดงข้อมูล dropdown ทันที ไม่ต้องรอ API
- **NFR-02** โหลดข้อมูลหลายชุด (admin) ต้องทำแบบขนาน (parallel)
- **NFR-03** ขนาดรูปที่อัปโหลดต้องถูกบีบอัด (≤ 900px / คุณภาพ 0.6) เพื่อให้บันทึกเร็ว
- **NFR-04** timeout รวมของ API = 60 วิ

### 5.2 ความเข้ากันได้
- **NFR-10** ต้องใช้งานได้บน LINE In-App Browser (iOS/Android) และเบราว์เซอร์ทั่วไป
- **NFR-11** ต้องรองรับ safe-area ของ iPhone (เมนูติดขอบจอ)
- **NFR-12** UI ต้องเป็น Responsive (มือถือหลัก, คอมรอง)

### 5.3 ความปลอดภัย
- **NFR-20** หน้า admin ต้องมีรหัสผ่าน (ปัจจุบัน: ฝั่ง client — หมายเหตุ: ควรย้ายไปตรวจฝั่ง GAS ในอนาคต)
- **NFR-21** ไม่ควรเก็บข้อมูลลับ/รหัสผ่านในไฟล์ที่ push ขึ้น GitHub

### 5.4 ความพร้อมใช้งาน
- **NFR-30** เมื่อ GAS cold start (40+ วิ) ผู้ใช้ต้องไม่เห็นหน้า "ค้าง" — ต้องมี feedback (spinner/toast)
- **NFR-31** ถ้าโหลดสดไม่สำเร็จแต่มีแคช → ใช้ข้อมูลแคชต่อไป ไม่บล็อกการใช้งาน

---

## 6. ข้อกำหนดข้อมูล (Data Requirements)

### 6.1 ข้อมูลหลัก (Master Data)
- **ส่งเสริม (Promoter):** รายชื่อเจ้าหน้าที่ส่งเสริม
- **Collector:** รายชื่อ Collector
- **ชุปเปอร์ไวเซอร์ (Supervisor):** รายชื่อ + เขตที่รับผิดชอบ
- **ชนิดพืช (PlantType):** ประเภทพืชที่ส่งเสริม
- **เขต (District):** เขต/อำเภอสำหรับการกำกับดูแล

### 6.2 ข้อมูลธุรกรรม (Transaction Data)
- **การส่งงาน:** วันที่, ชื่อส่งเสริม, Collector, อำเภอ, ผู้ติดต่อ, ชนิดพืช, อื่นๆ, รายละเอียด, พิกัด GPS, รูปภาพ
- **การกำกับดูแล:** วันที่, ชื่อชุปเปอร์ไวเซอร์, เขต, หมู่บ้าน, ตำบล (อัตโนมัติ), รายละเอียด, พิกัด, รูปภาพ

---

## 7. ข้อกำหนดการทดสอบ (Acceptance Criteria)

| # | เงื่อนไข | ผลลัพธ์ที่คาดหวัง |
|---|---|---|
| AC-01 | เปิดจาก LINE → กดเมนูส่งงาน | redirect ไปฟอร์มส่งงาน, ตรวจ GPS, ฟอร์มเปิด |
| AC-02 | เปิดฟอร์มครั้งแรก (ไม่มีแคช) | แสดง "กำลังโหลดข้อมูล..." จนกว่าข้อมูล dropdown มา |
| AC-03 | เปิดฟอร์มครั้งที่ 2 (มีแคช) | dropdown แสดงทันทีจากแคช แล้วค่อยอัปเดตสด |
| AC-04 | กรอกครบ + เลือกรูป | บันทึกสำเร็จ แจ้ง success |
| AC-05 | ไม่กรอกฟิลด์จำเป็น | บล็อกการส่ง, ชี้ฟิลด์ที่ต้องกรอก |
| AC-06 | เปิด admin บนคอม | แท็บอยู่ด้านบน เหนือตาราง |
| AC-07 | เปิด admin บนมือถือ | แท็บเป็นเมนูลอยติดขอบจอด้านล่าง |
| AC-08 | เพิ่ม/แก้/ลบชื่อใน admin | ข้อมูลอัปเดตและแสดงผลใหม่ถูกต้อง |
| AC-09 | API ตอบช้า/หมดเวลา | ผู้ใช้เห็น toast แจ้ง ไม่ค้าง |
| AC-10 | API ตอบ HTML (ยังไม่เปิดสาธารณะ) | แจ้งข้อความแนะนำวิธีเปิดใช้งาน |

---

## 8. ขอบเขตที่ไม่รองรับ (Out of Scope) / หมายเหตุ

- รหัสผ่าน admin ฝังในโค้ด HTML — **ควรย้ายไปตรวจฝั่ง GAS** ในเฟสถัดไป
- ยังไม่มีระบบ login/สิทธิ์รายบุคคล (ใช้ LIFF ระบุตัวตนเท่านั้น)
- ข้อมูลมีแคช 12 ชม. — หลังแก้ข้อมูลใน Sheets ผู้ใช้อาจเห็นข้อมูลเก่าสูงสุด 12 ชม.
- ไม่มีระบบสำรองข้อมูลอัตโนมัติ (ใช้ Google Sheets ตามค่าเริ่มต้นของ Google)
