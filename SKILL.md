# SKILL — เอกสารระบบ apmsystem

ระบบส่งงานส่งเสริมและการเกษตร ของ **บริษัทเชียงใหม่โฟรเซ่นฟูดส์ จำกัด (มหาชน)**

เว็บแอปที่ทำงานบน LINE (ผ่าน LIFF) สำหรับเจ้าหน้าที่ส่งเสริมการเกษตร, Collector และชุปเปอร์ไวเซอร์
ใช้ **Google Apps Script (GAS)** เป็น Backend และ **Google Sheets** เป็นฐานข้อมูล
หน้าเว็บเป็นไฟล์ HTML/JS/CSS ล้วน โฮสต์บน **GitHub Pages**

---

## 1. ภาพรวมระบบ

```
LINE (LIFF) ──▶ index.html (ตัวนำทาง)
                    │
                    ▼
   ┌──────────────┬────────────────┬──────────────┬──────────────┐
   ▼              ▼                ▼              ▼              ▼
promoteagriculture  suppervisor   adminapm   supervisortime  showdata / keyword
   (ส่งงาน)       (กำกับดูแล)    (จัดการข้อมูล)  (เช็คส่งงาน)   (ดูรายงาน/ถาม)
        │              │              │              │
        └──────────────┴──────────────┴──────────────┘
                          │
                          ▼
        Google Apps Script (GAS) — API /exec
                          │
                          ▼
                   Google Sheets (ฐานข้อมูล)
```

- **Frontend:** HTML + CSS + Vanilla JS (ไม่มี framework) รองรับมือถือเป็นหลัก
- **Backend:** Google Apps Script — แต่ละหน้าชี้ไปยัง Web App deployment ของตัวเอง (URL `/exec`)
- **ฐานข้อมูล:** Google Sheets (ข้อมูลอยู่ฝั่ง GAS)
- **Authentication:** LINE LIFF (`liff.init`) — ไม่ต้องล็อกอินซ้ำ
- **โฮสต์:** GitHub Pages — repo `cmfrozen1/apmsystem` สาขา `main`

---

## 2. โครงสร้างไฟล์

| ไฟล์ | หน้าที่ |
|---|---|
| `index.html` | หน้าเริ่มต้น/ตัวนำทาง — อ่านค่า `?linkurl=` แล้ว redirect ไปหน้างาน (ใช้ในเมนู LINE) |
| `promoteagriculture.html` | ฟอร์มส่งงานส่งเสริมการเกษตร (เจ้าหน้าที่ส่งเสริมกรอกข้อมูล + อัปโหลดรูป) |
| `suppervisor.html` | ฟอร์มรายงานการกำกับดูแล (ชุปเปอร์ไวเซอร์ — กรอกข้อมูล + อัปโหลดรูป) |
| `adminapm.html` | หน้าจัดการข้อมูล ชื่อส่งเสริม / Collector / ชุปเปอร์ไวเซอร์ (เพิ่ม-แก้-ลบ, ต้องใส่รหัสผ่าน) |
| `supervisortime.html` | ระบบเช็คข้อมูลการส่งงานประจำวัน — แสดง 2 ตารางใน iframe สลับด้วยแท็บล่าง |
| `showdata.html` | หน้ารายงานการส่งเสริมการเกษตร — ตารางสรุปข้อมูล + ดูรูป (lightbox) |
| `keyword.html` | หน้าเลือก Keyword ที่ต้องการถาม (เชื่อมกับบอท) |
| `promoteagriculture_test.html` | เวอร์ชันทดสอบของหน้า ส่งงาน |
| `promoteagriculture_backup.txt` | สำรองเวอร์ชันเก่าของหน้า ส่งงาน |
| `suppervisor_backup.txt` | สำรองเวอร์ชันเก่าของหน้า ชุปเปอร์ไวเซอร์ |
| `script.js` / `style.css` | สคริปต์/สไตล์ร่วม (ของระบบเก่า) |
| `README.md` | ไฟล์ README ของ repo |
| `ai.jpg`, `Manu2.jpg`, `1.jfif` | รูปประกอบ |

---

## 3. หน้าเว็บและฟังก์ชัน

### 3.1 `index.html` — หน้าเริ่มต้น (ตัวนำทาง)
- ใช้ LIFF ID `2007528760-v8N7EXG1`
- อ่าน query param `linkurl` = ปลายทางที่จะ redirect ไป
- แสดง spinner "กำลังเชื่อมต่อกับระบบ..." → redirect ไปหน้างานอัตโนมัติ
- เปิดนอก LINE ก็ยัง redirect ได้ (LIFF init fail ไม่บล็อก)

### 3.2 `promoteagriculture.html` — ส่งงานส่งเสริมการเกษตร
- LIFF ID `2007528760-v8N7EXG1`
- ฟอร์ม: ชื่อส่งเสริม, Collector, ชนิดพืช, วันที่, รายละเอียด + อัปโหลดรูป (บีบอัดรูปให้เล็กลงก่อนส่ง)
- Select ใช้ไลบรารี **Tom Select** (ค้นหา/เลือกได้)
- โหลดข้อมูล Select ด้วย **แคช localStorage 12 ชม.** + ดึงสดเบื้องหลัง, timeout 60 วิ
- API: `https://script.google.com/macros/s/AKfycbzniYO49YaS_3gv-V1WoLakPP14j6D9HScO6XCMDMzl3LdkADSI_RN1gVpaygghwdLD/exec`

### 3.3 `suppervisor.html` — รายงานการกำกับดูแล (Supervisor)
- LIFF ID `2007528760-v8N7EXG1`
- เลือกชื่อ → เลือกเขต → **ดึงข้อมูลอัตโนมัติ** (`getAutoData`) มาเติมฟอร์ม → แก้ไข/เพิ่มเติม → อัปโหลดรูป
- แคช localStorage 12 ชม. + timeout 60 วิ เหมือนหน้า ส่งงาน
- API: `https://script.google.com/macros/s/AKfycbyE8tvOVYAAR6WOiCVDEb2Fp1Yq3ebTAri9ZQYonpduMjQCFzod8xi0glBVXc6A6_qOeA/exec`

### 3.4 `adminapm.html` — จัดการข้อมูล (Admin)
- ต้องใส่รหัสผ่าน `admin` (hardcode ในโค้ด) ก่อนใช้งาน
- 3 แท็บ: **ส่งเสริม / Collector / ชุปเปอร์ไวเซอร์**
- ฟังก์ชัน: เพิ่ม / แก้ไข / ลบ ชื่อ — ส่ง POST `{action, name, row}`
- รับข้อมูลรายชื่อจาก API (`getPromoterNames`, `getCollectorNames`, `getSupervisorNames`)
- **แคช localStorage 12 ชม.** → เปิดรอบถัดไปโหลดทันที + โหลดสด 3 รายการแบบขนาน, timeout 60 วิ, retry 2 ครั้ง
- **UI ตามขนาดจอ:** จอคอม (กว้างกว่า 640px) แท็บอยู่ด้านบนเหนือตาราง / มือถือ (≤ 640px) แท็บกลายเป็นเมนูติดขอบจอด้านล่างแบบ Mobile app (ไอคอนบน-ข้อความล่าง, เผื่อ safe-area ของ iPhone)
- API: `https://script.google.com/macros/s/AKfycbwJ6CKgir6gaKJGRs5_IU8HVaqJ91FfZCbuE0ZO4CNr73H47gDrK0bV0KCMpvtrD6s/exec`

### 3.5 `supervisortime.html` — เช็คข้อมูลการส่งงานประจำวัน
- ไม่ใช้ LIFF — โหลด **2 iframe** ของ GAS:
  - ชุปเปอร์ไวเซอร์: `.../AKfycbwASVje2CA6B-YxpusKCSw0n8VVQoSYFFy3GyH8cV9asP_zHHd8WZXSOR3dKOAiOmsvWg/exec`
  - เจ้าหน้าที่ส่งเสริมฯ: `.../AKfycbyM1BUQOxd6XS_SsxKyEnQ-8MOaQUp7tkOn11qchlDvEDcw7oC0VwjxEl_DAAzQg1J11A/exec`
- แท็บด้านล่าง (Bottom Tab Bar) สลับระหว่าง 2 ตาราง พร้อม animation โหลด

### 3.6 `showdata.html` — รายงานการส่งเสริมการเกษตร
- ดึงข้อมูลทั้งหมดจาก API (`AKfycbx71XodbTvqXjFclnOgjfFEm46nDuIYO7TQoX6_tesfztpAennb4YEJbraCNsvvU3Oz/exec`)
- แสดงตาราง 13 คอลัมน์ + สถิติสรุป + ค้นหา/กรอง (debounce 250ms) + ดูรูปใหญ่ (lightbox) + ปุ่มรีเฟรช

### 3.7 `keyword.html` — เลือก Keyword ที่ต้องการถาม
- LIFF + เรียก API `AKfycbxzhmCEXRd0sqYTAkqFA-X2HvV773QoGIdndRuEcDUqi2kXrqHkdjEMCAUUx-cTKyLg1g/exec`
- ใช้สำหรับเมนูถาม-ตอบของบอท

---

## 4. Backend — Google Apps Script (GAS)

แต่ละหน้าเรียก GAS Web App ด้วย URL รูปแบบ:
```
https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec?action=<ACTION>
```

### รายการ Action ที่ระบบใช้

| Action | ใช้โดย | คำอธิบาย |
|---|---|---|
| `getPromoterNames` | ส่งงาน / ชุปเปอร์ไวเซอร์ / admin | รายชื่อเจ้าหน้าที่ส่งเสริม |
| `getCollectors` | ส่งงาน | รายชื่อ Collector |
| `getCollectorNames` | admin | รายชื่อ Collector (หน้า admin) |
| `getPlantTypes` | ส่งงาน | ชนิดพืช |
| `getDistricts` | ชุปเปอร์ไวเซอร์ | รายชื่อเขต/อำเภอ |
| `getAutoData&district=...` | ชุปเปอร์ไวเซอร์ | ดึงข้อมูลอัตโนมัติตามเขต |
| `getSupervisorNames` | admin | รายชื่อชุปเปอร์ไวเซอร์ |
| `addPromoter` / `addCollector` / `addSupervisor` | admin | เพิ่มชื่อ (POST) |
| `editPromoter` / `editCollector` / `editSupervisor` | admin | แก้ชื่อ (POST) |
| `deletePromoter` / `deleteCollector` / `deleteSupervisor` | admin | ลบชื่อ (POST) |
| `upload...` (ฝั่ง GAS) | ส่งงาน / ชุปเปอร์ไวเซอร์ | รับรูปที่อัปโหลด |

### การจัดการข้อผิดพลาดจาก GAS (ใช้ร่วมกันทุกหน้า)
- **Cold start:** GAS ครั้งแรกหลังไม่ถูกเรียกนานๆ อาจใช้เวลาถึง 40+ วิ → ตั้ง timeout 60 วิ แทน 30 วิ
- **ตอบเป็น HTML** (ยังไม่เปิดสาธารณะ / ต้องล็อกอิน Google) → ตรวจ `text[0] === '<'` แล้วแจ้งเตือนให้เปิด deployment เป็น Anyone
- **HTTP 403** = API ยังไม่เปิดสาธารณะ / **404** = deployment ถูกลบ
- **Response ว่าง / JSON ผิด** → แจ้ง error ที่เข้าใจง่าย
- **Retry แบบ backoff** สำหรับ HTTP 408/429/5xx และ timeout (เฉพาะหน้า admin)

### ข้อควรรู้เวลาแก้ GAS
- ตั้งค่า **Deploy > Manage deployments > Who has access = Anyone** ไม่งั้นหน้าเว็บจะได้ HTML หน้า login แทน JSON
- เมื่อแก้โค้ด GAS ต้อง **สร้าง deployment ใหม่** (หรือกด redeploy) ถึงจะได้ URL ใหม่ หรืออัปเดตเวอร์ชันเดิม
- แต่ละหน้าใช้ deployment ID ต่างกัน — ระวังอย่าไปแก้ URL ของหน้าอื่น

---

## 5. LINE (LIFF)

- **LIFF ID:** `2007528760-v8N7EXG1` (ใช้ร่วมใน index / ส่งงาน / ชุปเปอร์ไวเซอร์ / keyword)
- `index.html` ใช้เป็น entry point: สร้างเมนู Rich Menu ใน LINE ชี้มาที่
  `index.html?linkurl=<URL หน้าเป้าหมาย>` ระบบจะ redirect ให้อัตโนมัติ
- เปิดนอก LINE: `liff.init` จะ fail แต่ระบบยัง redirect ต่อได้ (test บนคอมได้)
- SDK: `https://static.line-scdn.net/liff/edge/versions/2.9.0/sdk.js`

---

## 6. เรื่องประสิทธิภาพ (เคยเจอมาแล้ว)

### ปัญหา: หน้า admin ค้าง "กำลังโหลดข้อมูล" นานมาก
สาเหตุ: GAS cold start (ครั้งแรก 42 วิ), โหลด 3 รายการแบบเรียงลำดับ, ไม่มีแคช, timeout 30 วิตัด request ทิ้งแล้ว retry ซ้ำ

### แนวทางที่ใช้แก้ (ทำใน adminapm แล้ว)
1. **แคช localStorage 12 ชม.** — เปิดหน้ารอบถัดไปแสดงข้อมูลทันที แล้วโหลดสดอัปเดตเบื้องหลัง
2. **โหลดหลายรายการพร้อมกัน** (parallel) แทนเรียงลำดับ
3. **แสดงผลทีละตาราง** ทันทีที่ได้ข้อมูล ไม่ต้องรอครบทั้งหมด
4. **timeout 60 วิ** + ลดจำนวน retry
5. ถ้ามีแคชแล้วโหลดสดไม่ครบ → แจ้งแบบ toast ไม่บล็อกการใช้งาน

> หน้า ส่งงาน / ชุปเปอร์ไวเซอร์ ใช้ pattern แคช 12 ชม. + timeout 60 วิ เหมือนกันแล้ว

---

## 7. การอัปเดตขึ้น GitHub Pages

```bash
git add <ไฟล์ที่แก้>
git commit -m "ข้อความคอมมิต"
git push origin main
```

- Push ขึ้น `main` แล้ว GitHub Pages จะอัปเดตภายใน 1-2 นาที
- หลังแก้ ให้ผู้ใช้ **รีเฟรชล้างแคช** (คอม: `Ctrl + F5`, มือถือ: ล้างแคชหรือรีเฟรช) เพราะเบราว์เซอร์เก็บหน้าเก่าไว้
- **บัญชี Git:** ปัจจุบันใช้บัญชี `cmfrozen1` (มีสิทธิ์ push repo นี้) — ถ้าโดน `403 Permission denied` ให้เช็คว่าล็อกอินบัญชีถูกตัวไหม:
  ```bash
  git credential-manager github logout <ชื่อบัญชีเก่า>
  git push origin main   # จะเด้งหน้าต่างให้ล็อกอินใหม่
  ```

---

## 8. ไลบรารี / เครื่องมือภายนอก

| ไลบรารี | ใช้ที่ไหน | ใช้ทำอะไร |
|---|---|---|
| LINE LIFF SDK 2.9.0 | index, ส่งงาน, ชุปเปอร์ไวเซอร์, keyword | ยืนยันตัวตน/เปิดใน LINE |
| Tom Select | ส่งงาน, ชุปเปอร์ไวเซอร์ | select แบบค้นหาได้ |
| SweetAlert2 (Swal) | ทุกหน้า | alert/toast สวยงาม |
| Google Fonts (Prompt) | ทุกหน้า | ฟอนต์ไทย |
| Google Apps Script | Backend | API + ฐานข้อมูล (Sheets) |

---

## 9. หมายเหตุ

- รหัสผ่านหน้า admin (`admin`) ฝังในโค้ด HTML — หากต้องการความปลอดภัยสูง ควรย้ายไปตรวจฝั่ง GAS
- ไฟล์ `*_backup.txt` และ `*_test.html` เป็นเวอร์ชันสำรอง/ทดสอบ ไม่ควรใช้เป็นหน้าใช้งานจริง
- ข้อมูลที่โหลดจาก API มีแคช 12 ชม. — หลังแก้ข้อมูลใน Google Sheets ผู้ใช้อาจเห็นข้อมูลเก่าสูงสุด 12 ชม. (หรือรีเฟรชล้างแคชเพื่อให้เห็นทันที)
