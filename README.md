# Booking Market - Google Apps Script

ระบบจองล็อคตลาดแบบ Google Apps Script ใช้ Google Sheets เป็นฐานข้อมูล และ Google Drive เก็บสลิปโอนเงิน

## ไฟล์

- `Code.gs` - API ฝั่ง server, จัดการ Sheet, Drive, locking, admin
- `index.html` - หน้าลูกค้าสำหรับจองล็อค
- `admin.html` - หน้าแอดมินสำหรับจัดการล็อคและรายการจอง
- `appsscript.json` - manifest ของ Apps Script

## วิธีติดตั้ง

1. สร้าง Google Sheet ใหม่ 1 ไฟล์
2. สร้าง Google Drive Folder สำหรับเก็บสลิป 1 โฟลเดอร์
3. เปิด Apps Script แล้วสร้างโปรเจกต์ใหม่
4. เพิ่มไฟล์ตามชื่อในโปรเจกต์ Apps Script:
   - `Code.gs`
   - `index.html`
   - `admin.html`
   - `appsscript.json`
5. แก้ค่าใน `Code.gs`

```js
const CONFIG = {
  SPREADSHEET_ID: 'ใส่ ID ของ Google Sheet',
  SLIP_FOLDER_ID: 'ใส่ ID ของ Drive Folder',
  ADMIN_PIN: 'เปลี่ยนรหัสแอดมิน',
  TEST_API_KEY: 'เปลี่ยนเป็นรหัสยาว ๆ สำหรับ load test',
  HOLD_MINUTES: 15,
  VIEWER_TTL_SECONDS: 45,
  MARKET_NAME: 'ชื่อตลาด',
};
```

6. กด Run ฟังก์ชัน `setup` หนึ่งครั้ง และอนุญาตสิทธิ์
7. Deploy เป็น Web app
   - Execute as: Me
   - Who has access: Anyone
8. เปิด URL ที่ได้
   - หน้าลูกค้า: `WEB_APP_URL`
   - หน้าแอดมิน: `WEB_APP_URL?page=admin`

## สถานะล็อค

- `available` - ว่าง
- `holding` - มีคนกำลังจอง ระบบกันไว้ชั่วคราว
- `pending_review` - ส่งจองแล้ว รอแอดมินตรวจสลิป
- `paid` - จองสำเร็จ
- `blocked` - ปิดไม่ให้จอง

## ความสามารถหน้า Admin

- เพิ่มล็อคแบบกำหนดแถวและช่วงเลข เช่น แถว A เลข 1-10
- ระบบเช็กล็อคซ้ำก่อนเพิ่ม เช่น `A-01` ถ้ามีอยู่แล้วจะไม่เพิ่มทับ
- ลบล็อคทีละช่อง
- ลบล็อคทั้งแถว
- ลบล็อคทั้งหมดออกจากผัง
- รีเซ็ตทุกอย่าง โดยลบล็อค รายการจอง viewer ค้าง และย้ายไฟล์สลิปใน Drive folder ไปถังขยะ
- เปลี่ยนสถานะล็อคจาก dropdown พร้อมเห็นสถานะปัจจุบันของแต่ละล็อค

## หมายเหตุ

ระบบนี้เป็น realtime แบบ polling ทุก 4 วินาที ไม่ใช่ WebSocket แต่เหมาะกับงานจองตลาดขนาดเล็กถึงกลาง และทำให้ดูแลผ่าน Google Sheets/Drive ได้ง่าย

หากคนใช้งานพร้อมกันเยอะมาก ควรขยับไปใช้ Firebase, Supabase หรือ backend จริงในรอบถัดไป

## Load test จำลอง 200 คน

ไฟล์ `load-test.js` ใช้จำลองผู้ใช้หลายคนยิงเข้า Web App จริง โดย flow คือโหลดสถานะ เลือกล็อค กดกันล็อค รอเหมือนกรอกฟอร์ม แล้วส่งจองบางส่วน

ก่อนทดสอบ:

1. แก้ `TEST_API_KEY` ใน `Code.gs` เป็นรหัสยาว ๆ ห้ามใช้ค่าเดิม
2. Deploy Web App ใหม่
3. เข้า admin แล้วสร้างล็อคให้พอ เช่น 20 แถว x 20 ล็อค = 400 ล็อค
4. ใช้ Google Sheet/Drive ชุดทดสอบ ไม่ควรยิงใส่ข้อมูลจริง

รันจากเครื่อง:

```bash
WEB_APP_URL="https://script.google.com/macros/s/xxxx/exec" \
TEST_API_KEY="รหัสเดียวกับใน Code.gs" \
USERS=200 \
BATCH_SIZE=25 \
node load-test.js
```

แนะนำให้ไล่เทสเป็นขั้น:

```bash
USERS=25 BATCH_SIZE=5 node load-test.js
USERS=50 BATCH_SIZE=10 node load-test.js
USERS=100 BATCH_SIZE=20 node load-test.js
USERS=200 BATCH_SIZE=25 node load-test.js
```

ถ้าเจอ error ประมาณ `หมดเวลาการล็อก` จำนวนมาก แปลว่า Google Apps Script/Sheet รับ concurrent write ไม่ไหวแล้ว โดยเฉพาะ flow จองที่ต้องกันข้อมูลซ้ำด้วย `LockService`

ปรับค่าได้:

- `USERS=200` จำนวนผู้ใช้จำลอง
- `BATCH_SIZE=25` จำนวนที่ยิงพร้อมกันต่อชุด
- `THINK_MS=1200` เวลารอจำลองตอนกรอกฟอร์ม
- `SUBMIT_RATIO=0.75` สัดส่วนคนที่ส่งจองจริง หลังจากกด hold แล้ว

หลังทดสอบควรเข้า admin แล้วล้างสถานะล็อคทั้งหมด หรือใช้ Google Sheet ทดสอบแยกต่างหาก
