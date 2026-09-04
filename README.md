# ที่ดินเปล่าธนาคารประกาศขาย (Bank NPA Land) — บันทึกการทำงาน

## ขอบเขตที่ตกลงกับผู้ใช้
- แหล่งข้อมูล: เว็บ NPA ของธนาคารเฉพาะเจาะจง (ทำแล้ว: ออมสิน/GSB, กรุงไทย/KTB, กสิกรไทย/KBank, ไทยพาณิชย์/SCB, เกียรตินาคินภัทร/KKP — **ธกส/BAAC: ผู้ใช้ตัดสินใจข้ามไปก่อนแล้ว** ดูหัวข้อด้านล่าง)
- ประเภททรัพย์: **เฉพาะที่ดินเปล่า** (ไม่เอาที่ดินพร้อมสิ่งปลูกสร้าง/คอนโด/รถ/เครื่องจักร ฯลฯ)
- ขอบเขตพื้นที่: ทั่วประเทศ
- ความถี่ที่อยากได้: รายสัปดาห์ (แต่ยังทำแบบอัตโนมัติเต็มรูปแบบไม่ได้ — ดูข้อจำกัดด้านล่าง)

## สิ่งที่ทำไปแล้ว

### 1) ออมสิน (GSB) — 2026-09-01
- สำรวจโครงสร้างเว็บ `npa-assets.gsb.or.th` (Next.js) พบว่า:
  - หน้ารายการ `/asset/npa/all?asset_type_id=341&page_size=200` คืนข้อมูลที่ดินเปล่าทั่วประเทศ 109 แปลง ฝังเป็น JSON ใน `__NEXT_DATA__` (ที่อยู่/ราคา/ขนาด แต่ไม่มีพิกัด)
  - หน้ารายละเอียดแต่ละแปลง `/asset/npa?id={asset_group_id_npa}&asset_type_id=341&type_id=asset_group_id_npa` มีพิกัด (`info.latitude`/`info.longitude`) ฝังใน `__NEXT_DATA__`
  - ดึงพิกัดสำเร็จ 97/109 แปลง (อีก 12 แปลงธนาคารยังไม่ได้กรอกพิกัดในระบบเขาเอง)
- `asset_type_id=341` = ที่ดินเปล่า (short_code "LD"), `342` = ที่ดินพร้อมสิ่งปลูกสร้าง

### 2) กรุงไทย (KTB) — 2026-09-01
- เว็บ `npa.krungthai.com` เป็น Angular app ไม่มี SSR JSON แบบ GSB แต่มี REST API จริงที่เรียกผ่าน `fetch()`:
  - Endpoint หลัก: `POST /api/v1/product/searchAll` body `{"paging":{"currentPage":N,"rowsPerPage":100,"totalRows":0},"typeProp":["1"]}`
  - `typeProp:"1"` = ที่ดินเปล่า (ค่าอื่น เช่น `2`=ที่ดินพร้อมสิ่งปลูกสร้าง, `31`=เพื่อเกษตร, `32`=เพื่ออุตสาหกรรม, `33`=ที่ดินพัฒนา — ไม่ได้ใช้)
  - `rowsPerPage` ถูกจำกัดสูงสุดที่ 100 ต่อครั้งไม่ว่าจะขอเท่าไหร่ → ต้อง paginate (currentPage 1-3) เพื่อให้ครบ 250 แปลงทั่วประเทศ
  - **ข้อดีกว่า GSB**: response มี `lat`/`lon` มาให้ในหน้ารายการเลย ไม่ต้องเปิดทีละหน้ารายละเอียด
  - หน้ารายละเอียดจริงคือ `https://npa.krungthai.com/property-detail/{collGrpId}` (ใช้ `collGrpId` ตัวเลข ไม่ใช่ `collCode` — ต้องดึง id-mapping เพิ่มอีกชุดเพื่อสร้างลิงก์ถูกต้อง)
  - ดึงสำเร็จ 248/250 แปลง (2 แปลงตัดออกเพราะไม่มีพิกัด/พิกัดผิดปกติ — พบ 1 เคสที่ lat กับ lon เท่ากันเป๊ะ ซึ่งผิดปกติชัดเจน จึงตัดทิ้ง)

### 3) กสิกรไทย (KBank) — 2026-09-04
- เว็บ `kasikornbank.com/th/propertyforsale` เป็น SharePoint-based site (ไม่ใช่ Next.js/Angular) แต่มี backend endpoint แบบ ASP.NET WebMethod ที่เรียกผ่าน `fetch()`/XHR:
  - Endpoint หลัก: `POST https://www.kasikornbank.com/Custom/KWEB2020/NPA2023Backend13.aspx/GetProperties` body `{"filter":{"CurrentPageIndex":N,"PageSize":50,"SearchPurpose":["AllProperties"],"PropertyTypes":["01"],"Provinces":[],"Amphurs":[],"Ordering":"new"}}`
  - `PropertyTypes:["01"]` = "01 ที่ดินว่างเปล่า" (ที่ดินเปล่า — ตรงกับขอบเขตที่ต้องการพอดี)
  - `PageSize` ถูกจำกัดสูงสุดที่ 50 ต่อครั้งไม่ว่าจะขอเท่าไหร่ → ทั้งหมด 1,257 แปลงทั่วประเทศ ต้อง paginate 26 หน้า (CurrentPageIndex 1-26)
  - **ข้อดีที่สุดในสามธนาคารแรก**: response มี `Latitude`/`Longtitude` ให้ในหน้ารายการเลย และ `PropertyID` (เช่น `"016602619"`) ใช้สร้างลิงก์หน้ารายละเอียดได้ตรงๆ ไม่ต้องดึง id-mapping แยกเหมือน KTB
  - หน้ารายละเอียดจริง: `https://www.kasikornbank.com/th/propertyforsale/detail/{PropertyID}.html`
  - ดึงสำเร็จ 1,256/1,257 แปลง (1 แปลงตัดออกเพราะไม่มีพิกัด)
  - **หมายเหตุเทคนิคสำคัญ**: การส่งข้อมูลก้อนใหญ่ออกจากหน้าเว็บมาให้ Claude อ่าน ใช้วิธีฝังข้อมูลเป็น `<pre>` element ในหน้า แล้วอ่านด้วย `get_page_text` (ไม่ใช่ return ค่าจาก `javascript_tool` ตรงๆ ซึ่งถูกตัดที่ ~1000-1300 ตัวอักษร) — เมื่อเนื้อหาใหญ่เกิน token cap ของ `get_page_text` ระบบจะบันทึกลงไฟล์ในเครื่อง Claude ให้อัตโนมัติ อ่านไฟล์นั้นต่อด้วย Python ได้เลย **ข้อควรระวังที่พบภายหลัง (จากงาน SCB/KKP): ไฟล์ที่บันทึกอัตโนมัตินี้ก็ยังถูกตัดที่ประมาณ 50,000-53,000 ตัวอักษรเท่ากัน** ไม่ใช่ไม่จำกัด — ใช้ได้ดีกับข้อมูลระดับ ≤~45KB ต่อการดึงหนึ่งครั้ง ถ้าข้อมูลก้อนใหญ่กว่านั้น (เช่นไฟล์ไบนารีที่ทำ base64) ต้องแบ่งเป็นหลายก้อน

### 4) ไทยพาณิชย์ (SCB) — 2026-09-04
- เว็บ `asset.home.scb` เป็นแอปฝั่งเซิร์ฟเวอร์ (ไม่ใช่ Next.js) มี REST API แบบ GET ธรรมดา:
  - Endpoint: `GET https://asset.home.scb/api/project/cmd?type=project&page=N&limit=N&sortBy=all&command=get_project&...&asset-type=vacant_land&...`
  - Response รูปแบบ `{"s":"y","m":"Success","d":[...],"total":N}`
  - **ไม่มีการจำกัด page size ที่พบ** — ขอ `limit=500` ได้ผลลัพธ์ครบทั้งหมด 107 แปลงในการเรียกครั้งเดียว (ง่าย/เร็วที่สุดในทุกธนาคารที่ทำมา)
  - แต่ละ item มี `latitude`/`longitude` และ `slug` (ใช้สร้างลิงก์ `https://asset.home.scb/project/{slug}` ได้ตรงๆ)
  - **ข้อควรระวัง**: หน้าเว็บ (ฟิลเตอร์ค้นหา) ไม่ยอมรับการ set ค่า filter ผ่าน synthetic `.value=`/`change event` หรือ `.click()` แบบ programmatic — ต้องใช้การคลิกจริงผ่าน `computer` tool (คลิกที่ dropdown ประเภททรัพย์ → เลือก "ที่ดินเปล่า" → คลิกปุ่มค้นหา) ถึงจะ trigger การ navigate ไป URL ที่มี query param `asset-type=vacant_land` ถูกต้อง (คาดว่าเป็นเพราะ React/Vue controlled component ไม่ sync state กับการแก้ DOM ตรงๆ)
  - ดึงสำเร็จ 106/107 แปลง (1 แปลงตัดออกเพราะไม่มีพิกัด)

### 5) เกียรตินาคินภัทร (KKP) — 2026-09-04
- เว็บ `kkppropify.kkpfg.com` เป็น Next.js App Router (React Server Components) — **ไม่มี REST API แบบ URL ธรรมดา** การดึงข้อมูลจริงเป็น Next.js Server Action (POST ไปที่ URL หน้าเว็บเอง ไม่ใช่ `/api/...`) รูปแบบ response เป็น "Flight protocol" พิเศษ:
  - บรรทัดข้อมูลจริงขึ้นต้นด้วย `1:` ตามด้วย JSON `{"data":{"total":20,"totalItems":116,"items":[...]}}`
  - **ปัญหาที่พบ**: การ navigate ตรงด้วย URL query param (`?type=ที่ดินเปล่า`) ไม่ทำงาน (ได้ผลลัพธ์ 0 รายการ เพราะมี chip ตัวกรอง "โครงการ" ติดค้างอยู่ด้วย) — ต้องเปิด modal ตัวกรองจริง คลิกยกเลิก chip "โครงการ" ออกก่อน ถึงจะกรองถูกต้อง (เหลือ "ที่ดินเปล่า" อย่างเดียว, totalItems=116)
  - **การ paginate**: พบปุ่มเลขหน้า (1-6) ที่ด้านล่างรายการ, หน้าละ 20 รายการ (หน้าสุดท้าย 16) — คลิกเลขหน้าจริงทำให้เกิด request `?type=...&page=N` (ทั้ง GET prefetch และ POST server action)
  - **เทคนิคดักจับ response ที่ถูกต้อง (encoding utf-8 ปกติ ไม่ใช่ mojibake)**: `read_network_requests` คืนค่า response body ที่ผ่านการ decode ผิด (เหมือน UTF-8 bytes ถูกตีความเป็น Latin-1) และยังถูกตัดที่ ~9000 ตัวอักษรด้วย — **วิธีแก้ที่ได้ผล**: monkey-patch `window.fetch` ในหน้าเว็บ (เก็บ response.clone().text() ลง object ก่อน request ถูกส่งจริง) แล้วค่อยคลิกปุ่มเปลี่ยนหน้า (native `element.click()` ทำงานได้ปกติสำหรับปุ่ม pagination ธรรมดา ต่างจาก SCB ที่ต้องใช้ computer tool) → ข้อความที่ capture ได้จะเป็น UTF-8 ที่ถูกต้อง 100% → dump ใส่ `<pre style="position:fixed">` ใน `<main>` (ต้องใส่ position:fixed ไม่ให้กระทบ layout ของปุ่มอื่น) → อ่านด้วย `get_page_text` (เติม padding `'X'.repeat(60000)` ต่อท้ายเพื่อบังคับให้เกิน token cap แล้วบันทึกเป็นไฟล์อัตโนมัติ) → อ่านไฟล์ด้วย Python แล้วตัดเอาเฉพาะข้อความระหว่าง `###START###`/`###END###`
  - **ข้อควรระวังเรื่อง prefetch**: Next.js prefetch ลิงก์ pagination ล่วงหน้าเมื่อเลื่อนเข้า viewport (ก่อน patch fetch ทัน) ทำให้บาง request ไม่ถูกดักจับ — ถ้า `window.__kkpCaptured` ว่างหลังคลิก ให้ลองคลิกใหม่อีกครั้ง (การคลิกจริงที่ trigger client-side navigation จะ fetch ใหม่เสมอแม้เคย prefetch ไปแล้ว)
  - field `district` ของ KKP จริงๆ คือ **"จังหวัด, อำเภอ"** (เช่น `"เชียงใหม่, เมืองเชียงใหม่"` = จังหวัดเชียงใหม่ อำเภอเมืองเชียงใหม่) — ลำดับตรงข้ามกับที่คาดไว้ตอนแรก (สลับ province/district ผิดตอนแรก แล้วแก้ทีหลัง)
  - price ที่ถูกต้องคือ `specialPrice` ถ้ามี (ราคาพิเศษ) ไม่งั้นใช้ `price`
  - ดึงสำเร็จครบ 116/116 แปลง (0 สูญหาย, 0 ซ้ำ, 0 พิกัดผิดปกติ) — url สร้างจาก `https://kkppropify.kkpfg.com` + field `url` ที่ได้มาตรงๆ (เช่น `/th/products/l26002`)

### รวมข้อมูล
- ไฟล์ `bank-npa-land.json` (GeoJSON FeatureCollection) ตอนนี้รวม **1,823 แปลง** (GSB 97 + KTB 248 + KBank 1,256 + SCB 106 + KKP 116) พร้อม properties: id, bank, bank_short (GSB/KTB/KBank/SCB/KKP), asset_type, area, price, price_promo, subdistrict, district, province, url (ลิงก์ไปหน้าประกาศต้นทาง), updated
- แก้ `landscreeningmap.html`/`index.html`:
  - เพิ่ม source `bankland` (geojson, โหลดจาก `./bank-npa-land.json` แบบ lazy — โหลดเมื่อผู้ใช้ติ๊กเปิดครั้งแรก หรือเมื่อค้นหาจังหวัดในกล่องค้นหา)
  - เพิ่ม layer `l-bankland` (circle สีทอง `#D9A441`)
  - เพิ่ม checkbox "ที่ดินเปล่าธนาคารประกาศขาย" ในเมนูชั้นข้อมูล (ใช้ร่วมกันทุกธนาคาร ไม่ต้องมี checkbox แยกต่อธนาคาร)
  - คลิกจุด → popup แสดงรายละเอียด+ลิงก์ไปหน้าประกาศจริงของธนาคาร (label ลิงก์ dynamic ตาม `pr.bank_short` ไม่ hardcode ชื่อธนาคารแล้ว) — **ยืนยันแล้วว่า HTML ไม่ต้องแก้อะไรเพิ่มเมื่อเพิ่มธนาคารใหม่ (SCB, KKP) เพราะ layer เป็น bank-agnostic เต็มรูปแบบตั้งแต่รอบ KBank**
  - **หมายเหตุ 2026-09-04 (v3): พฤติกรรม "คลิกจุด → popup" นี้ถูกแทนที่แล้วในหัวข้อ #8 ด้านล่าง** ด้วยการปักหมุดวิเคราะห์เต็มรูปแบบแทน — ดูรายละเอียดที่นั่น
- ไฟล์ `bank-npa-land.json` ตอนนี้มีขนาดประมาณ 1.06MB — ยังโหลดได้ปกติผ่าน fetch แต่ถ้าจะเพิ่มธนาคารต่อไปอีกเรื่อยๆ ควรพิจารณาตัด field ที่ไม่จำเป็นออก หรือแบ่งไฟล์ต่อธนาคาร ถ้าขนาดเริ่มเป็นปัญหา (ยังไม่ถึงจุดนั้น)
- ส่งไฟล์ให้ผู้ใช้และบันทึกลงโฟลเดอร์เครื่อง `C:\Users\Admin\Desktop\landuse project\` แล้ว — **ผู้ใช้ยังต้องอัปโหลดไฟล์ขึ้น GitHub Pages (repo thunpm007/selectIUse) เอง**

### 6) ค้นหาแปลงที่ดินขายรายจังหวัด + คลิกซูมเข้าพิกัด — 2026-09-04 (v1: dropdown ในแผงข้าง, ถูกแทนที่แล้วโดย v2 ด้านล่าง)
ผู้ใช้ขอครั้งแรก: "ปรับปรุงเว็บ ให้มีการค้นหาแปลงที่มีการขาย เป็นรายจังหวัด เมื่อค้นหาแล้วมีรายการที่ขาย กดไปที่รายการแล้วซูมอิไปยังพิกัดแปลง" — ทำเป็น `<select id="bankProvinceSel">` ในกล่อง "ชั้นข้อมูล" (มุมขวาบนของแผนที่) พร้อมรายการผลลัพธ์ `#bsList`/`#bsMeta` ใต้ dropdown
- **หมายเหตุ: UI แบบนี้ถูกรื้อออกและแทนที่ทั้งหมดแล้วในหัวข้อ #7 ด้านล่าง** (element `#bankProvinceSel`/`#bsMeta`/`#bsList`/`.bs-item` ไม่มีอยู่ในโค้ดปัจจุบันแล้ว) — เก็บบันทึกนี้ไว้เป็นประวัติเท่านั้น อย่าอ้างอิงชื่อ element เหล่านี้ในการแก้โค้ดครั้งต่อไป

### 7) รีดีไซน์ช่องค้นหาเป็นแถบค้นหากลางเว็บแบบ autocomplete — 2026-09-04 (v2)
ผู้ใช้ขอปรับจาก v1: "แก้ไขทำช่อง Search หารายการแต่แยก ตรงกลางเว็บ คล้าย nav สามารถพิมพ์คำค้นหาเป็นชื่อจังหวัดหรือ ดรอปดาวน์ ให้เลือกเวลาพิมพ์" — ย้ายช่องค้นหาออกจากแผงข้าง `#layerCtl` มาเป็นแถบค้นหาอิสระลอยกลางบนของแผนที่ (nav-bar-style) พิมพ์ชื่อจังหวัดแบบ free-text พร้อม autocomplete dropdown ขึ้นขณะพิมพ์ แทนที่ `<select>` เดิมทั้งหมด:
- **HTML**: `<div class="mapctl" id="bankSearchNav">` ลอยกลางบนของแผนที่ (`top:12px;left:50%;transform:translateX(-50%)`, กว้าง 360px) ประกอบด้วย `#bankSearchInput` (text input พร้อมไอคอนแว่นขยาย), ปุ่ม `#bankSearchClear` (✕ ล้างค่า, `hidden` ตอนช่องว่าง), และ `#bankDropdown` (กล่อง dropdown ผลลัพธ์ ซ่อน/แสดงด้วย class `.show`) — ลบ `<select id="bankProvinceSel">`/`#bsMeta`/`#bsList` เดิมออกจาก `#layerCtl` แล้ว (เหลือแค่ checkbox `#tgBankLand`→`#tgAll` ตามเดิม)
- **CSS**: class `.bsn-*` ทั้งหมด (`.bsn-inputwrap`, `.bsn-icon`, `.bsn-clear`, `.bsn-dropdown`, `.bsn-empty`, `.bsn-sitem` = แถวจังหวัดในโหมดแนะนำ, `.bsn-back` = แถบ "← ชื่อจังหวัด" กลับไปโหมดแนะนำ (sticky top), `.bsn-ritem` = แถวผลลัพธ์แปลงในโหมดผลลัพธ์)
- **JS**:
  - `bankProvinceCounts` (Map: จังหวัด → จำนวนแปลง, เรียง `.localeCompare(..,'th')`) แทนที่ `<select>` เดิม — สร้างจาก `computeBankProvinceCounts()` (เดิมชื่อ `populateBankProvinces()`)
  - `bsnMode` ('suggest' | 'results') + `bsnActiveProvince` + `bsnHi` (index แถวที่ไฮไลต์ด้วยคีย์บอร์ด) เป็น state ของ dropdown
  - `renderSuggestions(query)` — filter จังหวัดด้วย substring match (`.includes(q)`), แสดงสูงสุด 30 รายการเป็น `.bsn-sitem`
  - `renderResultsFor(province)` — แสดง `.bsn-back` header + รายการ `.bsn-ritem` (จำกัด 150 แปลง เหมือน v1)
  - `bsnSelectProvince(province)` — auto-โหลดข้อมูลถ้ายังไม่โหลด, auto-ติ๊กเปิด `#tgBankLand`, เรียก `renderResultsFor`
  - event: `input`/`focus`/`keydown` (ArrowUp/ArrowDown/Enter/Escape) บน `#bankSearchInput`, click delegation บน `#bankDropdown` (`.bsn-sitem`→เลือกจังหวัด, `.bsn-ritem`→คลิกผลลัพธ์, `.bsn-back`→กลับโหมดแนะนำ), click ปุ่ม `#bankSearchClear`
  - **หมายเหตุ 2026-09-04 (v3): ตอนนั้น `.bsn-ritem` click ยังเรียก `bankPopupHTML()`/`showBankPopup()` (popup ธรรมดา) — ฟังก์ชันทั้งสองนี้ถูกลบไปแล้วในหัวข้อ #8 ด้านล่าง แทนที่ด้วย `addBankParcel()` + `bankAnnouncementHtml()` อย่าอ้างอิงชื่อ `bankPopupHTML`/`showBankPopup` อีกในการแก้โค้ดครั้งต่อไป**
- **บั๊กที่เจอและแก้แล้วระหว่างทดสอบ**: click-outside-to-close listener (`document.addEventListener('click', e=>{ if(!bsnNav.contains(e.target)) bsnClose(); })`) เดิมพังเมื่อคลิกเลือกจังหวัดจาก suggestion — เพราะ `renderResultsFor()` ทำ `bsnDrop.innerHTML=...` แทนที่ DOM ของ dropdown ทั้งหมด (รวม element ที่เพิ่งถูกคลิก) **ก่อน**ที่ click event จะ bubble ขึ้นไปถึง `document` ทำให้ `e.target` หลุดออกจาก DOM ไปแล้ว และ `bsnNav.contains(e.target)` คืนค่า `false` ผิดพลาด (นึกว่าคลิกนอกกล่อง) ทำให้ dropdown ปิดตัวเองทันทีหลังเลือก — **แก้โดยใช้ `e.composedPath()` แทน** (คำนวณ ancestor chain ตอน dispatch event ก่อนมี DOM mutation ใดๆ จึงไม่ได้รับผลกระทบจากการแทนที่ innerHTML ทีหลัง): `const path=e.composedPath?e.composedPath():[]; if(path.length?!path.includes(bsnNav):!bsnNav.contains(e.target)) bsnClose();` — **ถ้าจะแก้โค้ด dropdown ที่มีการ replace innerHTML ระหว่างอยู่ใน click handler ในอนาคต ต้องระวังปัญหานี้เสมอ ใช้ composedPath() ไม่ใช่ contains(e.target)**
- ทดสอบแล้วด้วย Playwright + stub maplibregl กับข้อมูลจริง (พิมพ์จังหวัด → suggestion กรองถูกต้อง → เลือก → auto-โหลด+ติ๊ก checkbox+แสดงผลลัพธ์ → คลิกผลลัพธ์ → zoom ไปพิกัดตรง → กลับ/ล้าง/คลิกนอกกล่อง/คีย์บอร์ดทำงานถูกต้อง ไม่มี JS error)
- ส่งไฟล์ `index.html` ให้ผู้ใช้และบันทึกลงโฟลเดอร์เครื่อง `C:\Users\Admin\Desktop\landuse project\index.html` แล้ว

### 8) แสดงเฉพาะแปลงจังหวัดที่เลือกบนแผนที่ + คลิกแปลง = ปักหมุดวิเคราะห์รัศมีเต็มรูปแบบ — 2026-09-04 (v3)
ผู้ใช้ทดลองใช้เว็บจริงบน GitHub Pages แล้วพบ 2 ปัญหา ขอให้แก้ไข (ข้อความผู้ใช้): "อยากให้แสดงแค่พิกัดจังหวัดที่เลือกเท่านั้นไม่ใช่ทั้งประเทศ และเมื่อกดดูตรงพิกัดแปลง ให้แสดงรายละเอียดในรัศมีตามที่กำหนดไว้เหมือนกับ พิกัดแปลงที่ปักเอง" — แก้ครบทั้ง 2 จุดแล้ว:

**A) แสดงเฉพาะจังหวัดที่เลือก ไม่ใช่ทั้งประเทศ**
- ก่อนหน้านี้ `loadBankLand()` เรียก `map.getSource('bankland').setData(gj)` ด้วยข้อมูลทั้งประเทศทันทีที่โหลดเสร็จ (หรือทันทีที่ติ๊ก checkbox) ทำให้จุดทั้ง 1,823 แปลงขึ้นพร้อมกันเสมอ ไม่ว่าจะค้นหาจังหวัดหรือไม่ — **แก้แล้ว**: เอาบรรทัด `setData(gj)` นั้นออกจาก `loadBankLand()` (ฟังก์ชันนี้ตอนนี้แค่โหลด `bankLandFeatures`/คำนวณ `bankProvinceCounts` เก็บไว้ในหน่วยความจำเฉยๆ ไม่แตะ source บนแผนที่)
- เพิ่มตัวแปร `bankShownProvince` (string|null) + ฟังก์ชัน `updateBankLandSource(province)` — filter `bankLandFeatures` เอาเฉพาะจังหวัดที่ระบุ แล้ว `setData(fc(filtered))` เท่านั้น (province เป็น `null`/ไม่ระบุ → setData ด้วย array ว่าง = ไม่แสดงจุดใดเลย)
- `bsnSelectProvince(province)` เรียก `updateBankLandSource(province)` ทุกครั้งที่เลือกจังหวัดจาก autocomplete (ต่อจาก auto-โหลดข้อมูล+ติ๊ก checkbox เดิม)
- `$('#tgBankLand').onchange`: ติ๊กเปิด → ถ้ามี `bankShownProvince` อยู่แล้ว (เคยเลือกจังหวัดไว้ก่อนหน้า) จะเรียก `updateBankLandSource(bankShownProvince)` คืนจุดจังหวัดนั้นกลับมาทันที; ถ้ายังไม่เคยเลือกจังหวัดเลย จะไม่แสดงจุดใดๆ (source ว่างเปล่าตามที่ตั้งไว้ตอนโหลดหน้า) พร้อม toast แนะนำ "พิมพ์ชื่อจังหวัดในช่องค้นหาด้านบนเพื่อแสดงแปลงที่ดินธนาคารบนแผนที่"
- ปุ่มล้างค่าค้นหา (`#bankSearchClear`) **ไม่**รีเซ็ต `bankShownProvince`/ข้อมูลบนแผนที่ — แค่ล้างช่องพิมพ์เพื่อค้นใหม่ จุดของจังหวัดที่เลือกไว้ก่อนหน้ายังคงค้างอยู่บนแผนที่จนกว่าจะเลือกจังหวัดใหม่

**B) คลิกแปลง = ปักหมุดวิเคราะห์เต็มรูปแบบเหมือนพิกัดที่ปักเอง**
- ระบบเดิมมี pipeline "ปักหมุด" อยู่แล้วสำหรับพิกัดที่ผู้ใช้ใส่เอง/คลิกบนแผนที่: `addParcel(lat,lon,name,src)` → push เข้า `state.parcels` → `analyze(p)` (วิเคราะห์ภูมิประเทศ: ความสูง/ความชัน/ทิศน้ำไหลจาก Horn 3×3 grid + ring samples ผ่าน Open-Meteo elevation API) → `fetchPOI(p)` (หาโรงเรียน/สถานพยาบาลในรัศมี 50 กม. ผ่าน Overpass แล้ว sidebar สรุปนับตามรัศมีที่เลือกไว้ 15/30/50 กม. — ตัวแปร `RADII`/`state.radii`/ปุ่ม `#radBtns`) → `fetchFloodHistory(p)` (เช็คน้ำท่วมซ้ำซากจาก GISTDA) → แสดงผลทั้งหมดใน panel `#detail` (`renderDetail()`) พร้อมวงรัศมีบนแผนที่ (`renderMap()` ใช้ `state.radii`)
- **เพิ่มฟังก์ชันใหม่ `addBankParcel(f)`** ที่รับ GeoJSON feature ของแปลงธนาคาร แล้วเรียก pipeline เดียวกันนี้ทั้งหมด (`addParcel(...)`) — ทำให้แปลงธนาคารได้รับการวิเคราะห์ภูมิประเทศ+รัศมีสถานที่+น้ำท่วมซ้ำซาก เหมือนพิกัดที่ปักเอง 100% (ใช้ตัวเลือกรัศมี 15/30/50 กม. เดียวกันโดยอัตโนมัติ ไม่ต้องเขียนโค้ดแยก)
  - **กันซ้ำ (dedupe)**: สร้าง `bankId = bank_short+'|'+id` แปะไว้ที่ `p.bankId` — ถ้าคลิกแปลงเดิมซ้ำ (จากแผนที่หรือ list) จะไม่สร้าง parcel ใหม่ซ้ำ แค่ select+เลื่อนไปแปลงเดิมที่มีอยู่แล้ว
  - **หมายเหตุเรื่อง async**: `addParcel()` เป็น async function แต่ส่วน push เข้า `state.parcels`+set `state.active` ทำงานแบบ synchronous ก่อนจะถึง `await analyze(p)` ตัวแรก — จึงอ่าน `state.active` ต่อได้ทันทีหลังเรียก `addParcel(...)` (ไม่ต้อง await) เพื่อแปะ `p.bankId`/`p.bankInfo` เข้ากับแปลงที่เพิ่งสร้าง
- **ลบ `bankPopupHTML()`/`showBankPopup()` (popup ลอยธรรมดา) ทิ้งไปแล้ว** แทนที่ด้วย `bankAnnouncementHtml(p)` — สร้าง HTML block "ประกาศขายจากธนาคาร" (asset_type/bank/ตำแหน่ง/ราคา/วันที่อัปเดต/ลิงก์ประกาศต้นทาง) ฝังเข้าไปใน `renderDetail()` ของ sidebar โดยตรง (แสดงก่อนบล็อกภูมิประเทศเสมอถ้า `p.bankInfo` มีค่า ไม่ว่า parcel จะอยู่ในสถานะ error/กำลังโหลด/พร้อมแล้วก็ตาม — ข้อมูลนี้มีอยู่แล้วในมือทันทีไม่ต้อง fetch เพิ่ม)
- Rail การ์ดฝั่งซ้าย (`.pcard`) เพิ่ม chip สีทอง (`#D9A441`) แสดงชื่อย่อธนาคาร (`bank_short`) สำหรับแปลงที่มาจากธนาคาร ให้แยกแยะจากแปลงที่ปักเองได้ง่าย
- Handler ที่เปลี่ยนจาก `showBankPopup(f)` → `addBankParcel(f)`: (1) `map.on('click','l-bankland',...)` คลิกจุดบนแผนที่โดยตรง — ไม่ flyTo เพิ่ม (คงมุมมองเดิมไว้ เพราะผู้ใช้เพิ่งคลิกจุดที่เห็นอยู่แล้ว) (2) click บน `.bsn-ritem` ในรายการค้นหา — เรียก `addBankParcel(f)` แล้วตามด้วย `map.flyTo({center:f.geometry.coordinates,zoom:16,duration:900})` ทันที (ซูมเข้าแปลงชัดเจนตามที่ผู้ใช้ขอไว้ตั้งแต่ v1/v2) แล้วปิด dropdown
- **ทดสอบแล้วด้วย Playwright + stub `window.maplibregl` (แก้ให้ parse `style.sources`/`style.layers` จาก constructor ตรงๆ ด้วย เพราะโค้ดจริงประกาศ source/layer ทั้งหมดไว้ใน style object ตอนสร้าง Map ไม่ได้เรียก `map.addSource()`/`map.addLayer()` แยก — เป็น gotcha ที่เจอใหม่รอบนี้ ถ้า stub ครั้งต่อไปพลาดจุดนี้จะได้ `getSource()`คืนค่า null เสมอ) + stub `window.turf` (point/distance/destination/circle/lineString ด้วยสูตร haversine/destination bearing ธรรมดา เพราะ Turf.js ก็โหลดจาก cdnjs ที่ถูกบล็อกเหมือนกัน) + mock endpoint `api.open-meteo.com` (คืนค่าความสูงสมมติ) และ `overpass` (คืนโรงเรียน/รพ. สมมติ) ผ่าน `page.route()` เพื่อให้ pipeline วิเคราะห์รันจบได้ทั้งหมดในแซนด์บ็อกซ์**: ยืนยันว่า (ก) เลือกจังหวัดเชียงใหม่แล้ว source บนแผนที่มีเฉพาะ 23 จุดของเชียงใหม่ตรงกับ `bankLandFeatures` filter เป๊ะ ไม่ปนจังหวัดอื่น (ข) คลิกผลลัพธ์ค้นหา → ปักเป็น parcel ใหม่ 1 รายการ มี `bankInfo`/`bankId` ครบ, `flyTo` ตรงพิกัดที่คลิกที่ zoom 16, dropdown ปิดอัตโนมัติ (ค) sidebar แสดงบล็อก "ประกาศขายจากธนาคาร" ควบคู่กับบล็อกภูมิประเทศ+ตารางรัศมีโรงเรียน/สถานพยาบาล (15/30/50 กม.) +รายชื่อสถานที่ใกล้สุดถูกต้องครบ (ง) คลิกแปลงเดิมซ้ำผ่าน map layer click ไม่สร้างซ้ำ (`state.parcels.length` คงที่ที่ 1, แค่ select แปลงเดิม) (จ) คลิกแปลงอื่นในจังหวัดเดียวกัน เพิ่มเป็น parcel ที่ 2 ถูกต้อง (ฉ) ติ๊กปิด-เปิด checkbox ใหม่หลังเลือกจังหวัดไว้แล้ว คืนจุดจังหวัดเดิมกลับมาถูกต้องโดยไม่ต้องค้นหาใหม่ (ช) ติ๊กเปิด checkbox ตั้งแต่แรกโดยยังไม่เคยค้นหาจังหวัดเลย ขึ้น toast แนะนำให้ค้นหาถูกต้อง ไม่มีจุดใดๆ ขึ้นมา — ไม่มี JS console error ตลอดกระบวนการทดสอบ
- ส่งไฟล์ `index.html` ให้ผู้ใช้และบันทึกลงโฟลเดอร์เครื่อง `C:\Users\Admin\Desktop\landuse project\index.html` แล้ว (sync `landscreeningmap.html` ในคลาวด์ให้ตรงกันด้วย — ไฟล์นี้ไม่ได้อยู่ในโฟลเดอร์เครื่องผู้ใช้ มีแค่ `index.html`) — ผู้ใช้ยังต้องอัปโหลด `index.html` ขึ้น GitHub Pages เองเหมือนเดิม

### 9) โมเดลอาคาร 3 มิติรอบแปลงที่เลือก + ค้นหาร้านสะดวกซื้อ/ห้างสรรพสินค้าในรัศมี 15 กม. — 2026-09-04 (v4)
ผู้ใช้ขอ (ข้อความผู้ใช้): "อยากใส่โมลเดล 3 D ของอาคารด้วยได้ไหม และเพิ่มพิกัดร้านสะดวกซื้อ ห้างสรรพสินค้าชั้นนำ ด้วย ในรัศมี 15 กม" — ถามคำถามชี้แจงขอบเขตอาคาร 3 มิติ (รอบแปลงที่เลือกในรัศมีเล็กๆ เทียบกับทั้งพื้นที่ที่มองเห็นบนแผนที่, และ massing แบบกล่องประมาณจาก OSM ไม่ใช่โมเดลสมจริง) — **ผู้ใช้เลือก "รอบแปลง 1-2 กม."** ทำครบทั้ง 2 ฟีเจอร์แล้ว:

**A) อาคาร 3 มิติ (fill-extrusion massing จาก OSM รอบแปลงที่เลือก รัศมี 1.5 กม.)**
- เพิ่ม checkbox `#tgBuildings3d` ("อาคาร 3 มิติ (รอบแปลงที่เลือก)") ในเมนูชั้นข้อมูล ต่อจาก `#tg3d` (ภูมิประเทศ 3 มิติเดิม) — เพิ่ม source `buildings3d` (geojson ว่างตอนแรก) + layer `l-buildings3d` (type `fill-extrusion`, สี `#8FA6AD`, `fill-extrusion-height` มาจาก property `height` ของแต่ละ feature ผ่าน expression `['coalesce',['get','height'],6]`, opacity 0.85, ซ่อนไว้ก่อนด้วย `visibility:'none'`)
- `BUILDINGS_RADIUS_M = 1500` — ค่าคงที่ตามที่ผู้ใช้เลือก (รัศมีรอบแปลงที่กำลังเลือกอยู่ ไม่ใช่รอบมุมมองแผนที่)
- `fetchBuildings3D(p)` — ยิง Overpass query `way["building"](around:1500,lat,lon); out geom;` ดึง footprint อาคารจริงจาก OSM รอบพิกัดแปลง แปลง `geometry` array ของแต่ละ way เป็น GeoJSON Polygon ring (auto-close ถ้ายังไม่ปิด) พร้อม property `height` (ประมาณจาก `estimateBuildingHeight(tags)`)
- `estimateBuildingHeight(tags)` — ลำดับความสำคัญ: tag `height` (เมตรตรงๆ) → tag `building:height` → tag `building:levels`×3.2 (สมมติ 3.2 ม./ชั้น) → **ค่า default 6 เมตร** ถ้าไม่มี tag ใดเลย (จำเป็นเพราะข้อมูลอาคารใน OSM ประเทศไทยส่วนใหญ่ไม่มี tag ความสูง/จำนวนชั้น — เป็น massing โดยประมาณ ไม่ใช่ความสูงจริงเสมอไป ได้แจ้งไว้ในโค้ดเป็น comment)
- Cache ผลลัพธ์ต่อพิกัด (`buildingsCache`, key = `lat.toFixed(3),lon.toFixed(3)`) + guard `buildingsShownKey` กันการยิง Overpass ซ้ำเมื่อกลับมาแสดงแปลงเดิมที่เคยโหลดแล้ว
- Hook เข้า `render()`: ถ้า `state.showBuildings3d===true` จะเรียก `fetchBuildings3D(activeParcel)` ทุกครั้งที่แปลง active เปลี่ยน (สลับดูแปลงอื่นในรายการ) — ถ้าไม่มีแปลง active จะเคลียร์ source ให้ว่าง
- **การจัดการมุมกล้อง (pitch) ให้ `#tg3d` (terrain) กับ `#tgBuildings3d` (อาคาร) ทำงานร่วมกันได้โดยไม่แย่ง/รีเซ็ตมุมกล้องของกันและกัน**: ติ๊กเปิดตัวใดตัวหนึ่งจะปรับ pitch ขึ้น (ถ้า pitch ปัจจุบันเป็น 0) แต่ติ๊ก**ปิด**ตัวใดตัวหนึ่งจะรีเซ็ต pitch กลับ 0 **ก็ต่อเมื่อ**อีกตัวก็ไม่ได้ติ๊กอยู่ด้วย (เช็ค `$('#tgBuildings3d').checked`/`$('#tg3d').checked` ก่อนสั่ง `easeTo({pitch:0})`)
- ติ๊กเปิดอาคาร 3 มิติโดยยังไม่ได้ปัก/เลือกแปลงใดเลย → toast แจ้ง "ปักหรือเลือกแปลงก่อน เพื่อโหลดอาคาร 3 มิติรอบแปลงนั้น (รัศมี 1.5 กม.)" ไม่มีอาคารขึ้น (ต้องมีแปลง active ก่อนเสมอ)
- ถ้า Overpass ไม่พบอาคารเลยในรัศมีที่กำหนด (พื้นที่ชนบท/ข้อมูล OSM ยังไม่ครอบคลุม) → toast แจ้งผู้ใช้ตรงๆ แทนที่จะเงียบ

**B) ร้านสะดวกซื้อ + ห้างสรรพสินค้า/ห้างฯ ในรัศมี 15 กม. (คงที่ แยกจากตัวเลือกรัศมี 15/30/50 เดิม)**
- ขยาย Overpass query เดิมของ `fetchPOI(p)` (เดิมหาแค่ `amenity=school/college/hospital/clinic/doctors` + `healthcare=centre` ในรัศมี 50 กม.) ให้ยิงเพิ่ม `shop=convenience`/`shop=mall`/`shop=department_store` ในรัศมี **15 กม.คงที่** (`STORE_MALL_RADIUS_KM=15`) พร้อมกันในคำสั่งเดียว (ประหยัด round-trip)
- ผลลัพธ์แยกเป็น 4 กลุ่มใน `p.poi`: `school`, `health` (เดิม), `store`, `mall` (ใหม่) — เรียงตามระยะทางใกล้สุดก่อนทุกกลุ่ม (`turf.distance`)
- **จุดออกแบบสำคัญ**: รัศมี 15 กม. ของร้านสะดวกซื้อ/ห้างฯ เป็นค่าคงที่ตายตัวตามที่ผู้ใช้ระบุ **ไม่ผูกกับปุ่มเลือกรัศมี 15/30/50 กม. เดิม** (ตัวนั้นยังคุมแค่โรงเรียน/สถานพยาบาลเหมือนเดิม) — แต่การ**แสดงหมุดร้าน/ห้างบนแผนที่**ถูก gate ด้วย `state.radii.has(15)` (ต้องติ๊กปุ่มรัศมี 15 กม. ค้างอยู่ด้วยถึงจะเห็นหมุด) เพื่อใช้ปุ่มเดิมที่มีอยู่แล้วอย่างสมเหตุสมผล ไม่ต้องเพิ่มปุ่มใหม่ซ้ำซ้อน — ส่วนตัวเลข "พบทั้งหมด N แห่ง" ในแผงรายละเอียดจะขึ้นเสมอไม่ว่าจะติ๊กปุ่มรัศมีไหนอยู่ก็ตาม
- Marker สีใหม่บนแผนที่: ร้านสะดวกซื้อ = เขียว (`--forest` `.poi-mk.st`), ห้างฯ = แดง/ส้ม (`--warn` `.poi-mk.ml`) — แยกจากโรงเรียน (สีดินเผา `--contour`) และสถานพยาบาล (สีฟ้าน้ำ `--water`) เดิม
- แผงรายละเอียด (`renderDetail`) เพิ่ม legend สีย่อ 4 หมวด + section "ร้านสะดวกซื้อใกล้สุด"/"ห้างสรรพสินค้าใกล้สุด" (แสดง 4 อันดับแรก + ตัวเลขรวม) ต่อจาก section โรงเรียน/สถานพยาบาลเดิม, ปุ่ม retry ค้นหาเปลี่ยนข้อความเป็น "ค้นหาโรงเรียน/รพ./ร้านสะดวกซื้อ/ห้างฯ"
- ตารางเปรียบเทียบแปลง (compare table) เพิ่ม 2 คอลัมน์ใหม่: `ร้านสะดวกซื้อ≤15` และ `ห้างฯ≤15` (จำนวนแห่งในรัศมี 15 กม.)
- checkbox `#tgPoi` เปลี่ยน label จาก "โรงเรียน / สถานพยาบาล" เป็น "รร./รพ./ร้านสะดวกซื้อ/ห้างฯ" ให้ตรงกับขอบเขตใหม่

**ทดสอบ**
- เขียน Playwright test แยก (`test3.js`, ลบทิ้งหลังใช้งานตามแบบแผนเดิมของโปรเจกต์นี้) ขยาย stub เดิม (`window.maplibregl`/`window.turf`) เพิ่ม `getPitch()`/`easeTo()` tracking + mock Overpass ให้แยกคำตอบ query อาคาร (คืน 3 way สมมติ มี tag ความสูงต่างสถานการณ์: มี `height`, มีแค่ `building:levels`, ไม่มี tag เลย) ออกจาก query POI (คืนโรงเรียน/รพ./ร้าน/ห้าง ที่ระยะต่างๆ รวมเคสห้างที่ระยะ ~20 กม. เพื่อทดสอบว่าตัด cutoff 15 กม. ฝั่ง client ถูกต้อง) — ยืนยันผ่านทุกเคส: ติ๊กอาคาร 3 มิติหลังเลือกแปลง → อาคารขึ้นตรงตามความสูงที่ประมาณจาก tag ที่ให้มาแต่ละเคส (รวม fallback 6 ม. ของเคสไม่มี tag) และ pitch ปรับขึ้นอัตโนมัติ, สลับดูแปลงอื่น → อาคารเปลี่ยนตามแปลงที่เลือกใหม่ (ยิง Overpass ใหม่ ไม่ใช่ cache เก่า), กลับมาดูแปลงเดิมซ้ำ → ใช้ cache ไม่ยิง Overpass ซ้ำ, ปิด `#tgBuildings3d` แต่ `#tg3d` (terrain) ยังติ๊กอยู่ → pitch ไม่ถูกรีเซ็ตกลับ 0 (คงมุมมอง terrain ไว้), ร้านสะดวกซื้อ/ห้างฯ กรองที่ระยะ >15 กม. ออกถูกต้อง (เคส ~20 กม. ไม่ปรากฏในผลลัพธ์), หมุดร้าน/ห้างบนแผนที่ขึ้นเฉพาะเมื่อติ๊กปุ่มรัศมี 15 กม. ค้างอยู่ — ไม่มี JS console error
- รัน regression test แยก (`test4_regress.js`, ลบทิ้งหลังใช้งาน) ยืนยันฟีเจอร์จากหัวข้อ #8 (กรองจังหวัด + คลิกแปลงธนาคาร=ปักหมุดวิเคราะห์เต็มรูปแบบ) ยังทำงานถูกต้องไม่มีผลกระทบจากการเปลี่ยนแปลงรอบนี้
- `node --check` ผ่านทุกครั้งก่อนส่งมอบ
- ส่งไฟล์ `index.html`/`landscreeningmap.html` (sync กันแล้ว) ให้ผู้ใช้และบันทึกลงโฟลเดอร์เครื่อง `C:\Users\Admin\Desktop\landuse project\index.html` แล้ว — ผู้ใช้ยังต้องอัปโหลด `index.html` ขึ้น GitHub Pages เองเหมือนเดิม

### 10) ตรวจสอบและแก้ 2 จุดความปลอดภัย (esc URL + SRI บน CDN scripts) — 2026-09-04 (v5)
ผู้ใช้ถาม "เว็บไซต์ของเราเรื่องความปลอดภัยเป็นไง" — ตรวจโค้ด `index.html` ทั้งไฟล์แล้วให้ภาพรวม จากนั้นผู้ใช้ตอบ "จัดไป" ให้แก้ 2 จุดที่พบ:

**ภาพรวมความปลอดภัย (ยังใช้ได้เป็น baseline สำหรับตรวจซ้ำครั้งถัดไป)**
- เป็น static site ล้วนๆ host บน GitHub Pages ผ่าน HTTPS ไม่มี backend/server ของตัวเอง ไม่มีระบบล็อกอิน/บัญชี ไม่มี cookie/session ไม่มีข้อมูลผู้ใช้ถูกเก็บที่ไหนเลย ทุกอย่างรันฝั่ง browser
- เช็คทั้งไฟล์แล้ว **ไม่มี API key/secret/token ฝังอยู่เลย** — endpoint ทั้งหมด (Open-Meteo, Overpass mirrors, GISTDA, OSM/ArcGIS/OpenTopoMap tile servers) เป็น public API ไม่ต้องใช้คีย์
- ข้อมูลจากภายนอก (ชื่อสถานที่จาก OSM, ชื่อ/ที่อยู่แปลงธนาคารจาก `bank-npa-land.json`) ถูก escape ด้วยฟังก์ชัน `esc()` (บรรทัด ~577: `replace(/[&<>"']/g,...)`) ก่อนแทรกผ่าน `innerHTML` เกือบทุกจุด — เช็คแล้วครบทุกจุดที่แทรกข้อความยกเว้นจุดเดียว (แก้แล้ว ดูด้านล่าง)

**A) แก้ XSS เล็กน้อยจาก `pr.url` ที่ไม่ผ่าน esc() ก่อนใส่ attribute**
- `bankAnnouncementHtml(p)` (บรรทัด ~1437) เดิมใส่ `<a href="${pr.url}">` ตรงๆ โดยไม่ escape — ถ้า URL ที่ดึงมาจากเว็บธนาคาร (เก็บไว้ใน `bank-npa-land.json`) มีอักขระ `"` ปนมา จะหลุดออกจาก attribute ได้ในทางทฤษฎี (ความเสี่ยงจริงต่ำมาก เพราะเป็นข้อมูลที่ Claude ดึงมาเองจากเว็บธนาคารที่รู้จัก ไม่ใช่ input จากคนแปลกหน้า แต่แก้ให้ครบเพื่อความรัดกุม)
- **แก้แล้ว**: เปลี่ยนเป็น `<a href="${esc(pr.url)}">` — ทดสอบด้วย Playwright โดยยัด URL ทดสอบที่มีโค้ดแฝงเข้าไปตรงๆ ยืนยันว่า output escape ถูกต้อง ไม่มี raw injection หลุดออกมา — ผ่าน

**B) เพิ่ม Subresource Integrity (SRI) ให้ library ภายนอกที่โหลดจาก CDN**
- เดิมโหลด `maplibre-gl` 4.7.1 และ `Turf.js` 6.5.0 จาก `cdnjs.cloudflare.com` แบบ pin เวอร์ชันแล้วแต่ไม่มี `integrity` attribute กำกับ
- **ข้อจำกัดที่พบระหว่างแก้**: แซนด์บ็อกซ์คลาวด์บล็อกการเชื่อมต่อ `cdnjs.cloudflare.com`/`cdn.jsdelivr.net` โดยตรง (`curl`/Playwright ทำไม่ได้) ทำให้คำนวณ SRI hash ของไฟล์ cdnjs เองแบบยืนยันได้ตรงๆ ไม่ได้
- **วิธีแก้ที่เลือก**: ใช้ `WebFetch` tool (เส้นทางเน็ตเวิร์กคนละเส้น เข้าถึง `data.jsdelivr.com`/`cdn.jsdelivr.net`/`cdnjs.cloudflare.com` ได้) ดึง SRI hash (sha256, base64) ที่ **jsDelivr เป็นผู้คำนวณเองจากไฟล์ที่ jsDelivr serve จริง** แล้ว**เปลี่ยน CDN จาก cdnjs → jsDelivr ทั้ง 3 ไฟล์**พร้อมใส่ hash คู่กันไปเลย (ไม่ใช้ hash จาก jsDelivr ไปแปะกับ URL cdnjs เพราะไม่รับประกันว่าไบต์จะตรงกัน)
- ไฟล์ที่เปลี่ยน (บรรทัด 12-14):
  ```html
  <link href="https://cdn.jsdelivr.net/npm/maplibre-gl@4.7.1/dist/maplibre-gl.css" rel="stylesheet" integrity="sha256-V2sIX92Uh6ZaGSFTKMHghsB85b9toJtmazgG09AI2uk=" crossorigin="anonymous">
  <script src="https://cdn.jsdelivr.net/npm/maplibre-gl@4.7.1/dist/maplibre-gl.js" integrity="sha256-vpYzxNhw4m+zfxz+XFp3GBZnEUAD6hYgeseFDY2ordE=" crossorigin="anonymous"></script>
  <script src="https://cdn.jsdelivr.net/npm/@turf/turf@6.5.0/turf.min.js" integrity="sha256-0A8+j/io+cED2tYcL9S7WBQ+FASq398J4pttsaLeCj8=" crossorigin="anonymous"></script>
  ```
  (Google Fonts URL ไม่ใส่ SRI ตามปกติ เพราะ response แปรผันตาม User-Agent ทำให้ hash ไม่คงที่)
- ทดสอบ: Playwright เฉพาะจุด (`test_sec.js`, ลบทิ้งหลังใช้งาน) ยืนยัน escape ถูกต้อง — `node --check` ผ่าน
- ส่งไฟล์ให้ผู้ใช้และบันทึกลงโฟลเดอร์เครื่อง `C:\Users\Admin\Desktop\landuse project\index.html` แล้ว
- **✅ ยืนยันแล้วว่า live-test ผ่าน (ดูหัวข้อ #11 ด้านล่าง)**: ผู้ใช้ส่งสกรีนช็อตเว็บจริงบน GitHub Pages (thunpm007.github.io/selectIUse) มาวันเดียวกัน แผนที่/ภูมิประเทศ 3 มิติ/ปักหมุด/POI ทำงานปกติทุกอย่าง แปลว่าการสลับ CDN เป็น jsDelivr+SRI ใช้งานได้จริงบนเบราว์เซอร์จริง ไม่มีปัญหา integrity mismatch ตามที่กังวลไว้

### 11) ลด exaggeration ของภูมิประเทศ 3 มิติ แก้ปัญหา "ภูเขาปลอม" เวลาซูมออกกว้าง — 2026-09-04 (v6, เวอร์ชันปัจจุบัน)
ผู้ใช้ส่งสกรีนช็อตเว็บจริงบน GitHub Pages ที่ซูมออกกว้างมาก (ค้นหา "กรุงเทพมหานคร" ในกล่องค้นหา แต่มุมมองยังกว้างระดับภาพรวมประเทศ ไม่ได้ซูมเข้าเฉพาะแปลง) พร้อมเปิด "ภูมิประเทศ 3 มิติ" (`#tg3d`) ไว้ — เห็นภูเขาแหลมๆ ขึ้นเต็มพื้นที่ทั่วทั้งภาพ **รวมถึงเหนือพื้นที่กรุงเทพฯ/ที่ราบภาคกลางซึ่งจริงๆ ราบเรียบเกือบสนิท** ผู้ใช้ถาม "มีพื้นที่ความสูงเพี้ยน สามารถแก้ได้ไหม"

**สาเหตุ**
- โค้ดเดิม (บรรทัด ~1283 เดิม): `map.setTerrain({source:'dem',exaggeration:4.5})` — ตั้ง exaggeration สูงมากตั้งแต่แรกเพราะ **ที่ดินไทยส่วนใหญ่ค่อนข้างราบ ต้องการดันความสูงมากๆ ถึงจะเห็นความต่างของภูมิประเทศชัดตอนซูมเข้าดูแปลงจริง**
- ปัญหาคือค่า 4.5 นี้สูงเกินไปสำหรับตอนซูมออกกว้าง (มุมมองระดับประเทศ/หลายจังหวัด) — ที่ระดับซูมนั้น MapLibre ใช้ไทล์ DEM (`s3.amazonaws.com/elevation-tiles-prod/terrarium`) ความละเอียดต่ำกว่าที่ zoom สูงมาก มี noise/ความคลาดเคลื่อนเล็กๆ ในข้อมูลอยู่แล้วเป็นปกติ — พอคูณด้วย exaggeration 4.5 เข้าไป noise เล็กๆ นั้นก็ถูกขยายจนกลายเป็นภูเขาแหลมๆ ปลอมๆ กระจายทั่วพื้นที่ (แม้พื้นที่จริงจะราบสนิทอย่างกรุงเทพฯ ก็ตาม) — เลเยอร์นี้เป็นแค่เอฟเฟกต์ภาพ 3 มิติบนแผนที่เท่านั้น **ไม่กระทบตัวเลขวิเคราะห์ความสูง/ความชันในแผงรายละเอียด** (ตัวเลขวิเคราะห์มาจาก Open-Meteo elevation API ผ่าน pipeline `analyze(p)` คนละส่วนกันโดยสิ้นเชิง) — เป็นบั๊กเชิงภาพเท่านั้น ไม่ใช่ข้อมูลผิด

**แก้แล้ว**
- ลด exaggeration จาก `4.5` → `1.8` ที่ `$('#tg3d').onchange` handler — ยังเห็น relief ของพื้นที่จริงที่มีความต่างของความสูงชัดอยู่ (เช่นภาคเหนือ/พื้นที่ภูเขาจริง) แต่ลด noise amplification ลงมากกว่าครึ่ง ทำให้พื้นที่ราบอย่างกรุงเทพฯ ไม่ขึ้นเป็นภูเขาปลอมอีก
- อัปเดต comment ในโค้ดอธิบาย trade-off นี้ไว้ด้วย กันลืมเหตุผลในอนาคต
- **หมายเหตุ**: นี่เป็นการลดค่าคงที่ตัวเดียว ยังไม่ได้ทำ zoom-dependent exaggeration (เช่นลดค่าอัตโนมัติเพิ่มเติมเมื่อซูมออกกว้างมากๆ) — ถ้าผู้ใช้ทดสอบแล้วยังเห็นภูเขาปลอมหลงเหลืออยู่บ้างตอนซูมออกสุดๆ ให้แจ้งกลับ จะพิจารณาลดเพิ่มหรือทำ gating ตาม zoom level
- `node --check` ผ่าน, sync `landscreeningmap.html` แล้ว, ส่งไฟล์ให้ผู้ใช้และบันทึกลงโฟลเดอร์เครื่อง `C:\Users\Admin\Desktop\landuse project\index.html` แล้ว — ผู้ใช้ยังต้องอัปโหลด `index.html` ขึ้น GitHub Pages เองเหมือนเดิม (ยังไม่ได้ live-verify ผลลัพธ์หลังลด exaggeration รอบนี้ เพราะเพิ่งแก้ ให้ผู้ใช้เช็คหลังอัปโหลดว่าภูเขาปลอมหายไปสมใจหรือไม่)

## เรื่องกฎหมาย/ใบอนุญาตของแหล่งข้อมูล — คุยกับผู้ใช้ไว้ (2026-09-04, ยังไม่ใช่งานที่ต้องทำโค้ด แค่บันทึกไว้)
ผู้ใช้ถามว่าถ้าจะเอาเว็บนี้ไปเผยแพร่เชิงพาณิชย์ได้ไหม แล้วถามต่อว่าถ้าเปิดรับบริจาคตามศรัทธา (ไม่ใช่ขายของ) จะต่างกันไหม — ตอบตามข้อเท็จจริงที่เช็คจาก ToS จริงของแต่ละแหล่งข้อมูล (ไม่ใช่คำแนะนำทางกฎหมาย):
- **MapLibre GL JS (BSD-3), Turf.js (MIT), Google Fonts (OFL)** — ใช้เชิงพาณิชย์/บริจาคได้ปกติ ไม่มีข้อจำกัด
- **Open-Meteo** — free tier ระบุชัดว่า non-commercial เท่านั้น (นิยามตาม Creative Commons: private/non-profit ที่ไม่มี subscription/โฆษณา ถือเป็น non-commercial) — **เว็บรับบริจาคล้วนๆ ไม่มี subscription/โฆษณา น่าจะเข้าข่าย non-commercial ตามนิยามของเขาเอง** ถ้าจะขายของ/มีโฆษณาจริงต้องสมัครแพ็กเกจจ่ายเงิน
- **Overpass API (เซิร์ฟเวอร์สาธารณะ overpass-api.de/kumi.systems/private.coffee)** — usage policy ระบุชัดว่า "Commercial use should use self-hosted or paid Overpass servers" และจำกัดโหลดของแอปที่ใช้งาน "regular" ไว้ที่ ~100 query/วัน, 10MB/วัน (หาร budget ปกติด้วย 100) — **ข้อจำกัดนี้ผูกกับปริมาณการใช้งาน/สเกล ไม่ใช่ว่าหาเงินหรือเปล่า** ถ้าคนใช้เว็บเยอะขึ้นจะชนขีดจำกัดนี้อยู่ดีไม่ว่าจะรับบริจาคหรือไม่ ต้องย้ายไป self-host หรือใช้ผู้ให้บริการที่จ่ายเงินในที่สุด
- **OpenTopoMap** — ขอให้ติดต่อทีมงานก่อนถ้าจะใช้กับ "โปรเจกต์ใหญ่" เช่นกันผูกกับสเกลไม่ใช่โมเดลรายได้
- **Esri World Imagery (ภาพถ่ายดาวเทียม)** — ยังไม่ได้เช็คเงื่อนไขชัดเจน (เอกสาร Esri ไม่เปิดรายละเอียดที่ดึงมาได้) โดยทั่วไปบริการฟรีแบบนี้มักตั้งใจให้ใช้แบบประเมิน/เบาๆ ไม่ใช่ backend ของแอป production จริงจัง — แนะนำเช็คกับ Esri โดยตรงถ้าจะขยายสเกล
- **GISTDA (ข้อมูลน้ำท่วมซ้ำซาก)** — open data หน่วยงานรัฐไทย โดยทั่วไปเปิดกว้างกว่า แต่ยังไม่ได้เช็คเงื่อนไขเฉพาะชุดข้อมูลนี้ให้ชัวร์
- **ข้อมูลที่ดินธนาคาร (`bank-npa-land.json`)** — **ความเสี่ยงหลักที่สุด ไม่เปลี่ยนไปตามโมเดลรายได้เลย**: ข้อมูลนี้ดึงมาจาก internal API ของเว็บ 5 ธนาคาร (GSB/KTB/KBank/SCB/KKP) โดยตรง ไม่ใช่ API สาธารณะที่เปิดให้ใช้อย่างเป็นทางการ — เว็บธนาคารมักมี ToS ห้าม scrape/นำข้อมูลไปเผยแพร่ต่อโดยไม่ได้รับอนุญาต ไม่ว่าปลายทางจะแสวงหากำไรหรือไม่ก็ตาม — **แนะนำผู้ใช้ให้ปรึกษาทนายเรื่องนี้โดยเฉพาะก่อนเผยแพร่เว็บสู่สาธารณะวงกว้าง** หรือพิจารณาขออนุญาตธนาคารตรงๆ ก่อน หรือเปลี่ยนไปแค่ลิงก์ผู้ใช้ไปหน้าประกาศต้นทางแทนการแสดงข้อมูลเองทั้งหมด
- ยังไม่ได้ตัดสินใจ/ดำเนินการอะไรเพิ่มจากเรื่องนี้ — เป็นแค่ข้อมูลให้ผู้ใช้ประกอบการตัดสินใจ ถ้าผู้ใช้จะขยายสเกล/เผยแพร่วงกว้างในอนาคตควรกลับมาคุยเรื่องนี้อีกครั้ง

## ธกส (BAAC) — สำรวจแล้ว, ผู้ใช้ตัดสินใจข้ามไปก่อน (2026-09-04)
- **สถาปัตยกรรมต่างจาก 5 ธนาคารข้างต้นโดยสิ้นเชิง**: baac.or.th ไม่มีเว็บพอร์ทัลค้นหาทรัพย์ NPA แบบมี filter/แผนที่/พิกัดเหมือนธนาคารอื่น มีแต่หน้า "ประกาศขายทอดตลาดทรัพย์สิน" (`https://www.baac.or.th` → เมนูข่าวสาร → ประกาศขายทอดตลาดทรัพย์สิน) ซึ่งเป็น**รายการไฟล์ PDF ประกาศเป็นชุดๆ** (เช่น "ครั้งที่ 6/2569") พบทั้งหมด 4 หน้า (~12 ประกาศ/หน้า ~40+ ประกาศ) แต่ละ PDF รวมทรัพย์หลายประเภท (ที่ดินเปล่า/ที่ดินพร้อมอาคาร) ปนกัน ไม่แยกเฉพาะที่ดินเปล่า
- **ไม่มีพิกัด (lat/lng) ในข้อมูลเลย** — ระบุทรัพย์ด้วยเลขโฉนดที่ดิน/ตำบล/อำเภอ/จังหวัด/สาขาธนาคารที่ดูแลเท่านั้น ถ้าจะ plot บนแผนที่ต้อง geocode เอาเองจากที่อยู่ (ได้แค่ระดับตำบล/อำเภอโดยประมาณ ไม่แม่นเท่าธนาคารอื่นที่มีพิกัดจริง)
- **ลองดึงเนื้อหา PDF แล้วติดปัญหาทางเทคนิค**:
  - ดาวน์โหลด PDF ตรงๆ ผ่าน `curl`/`device_bash` ถูกบล็อกด้วย network allowlist เดียวกับธนาคารอื่น (403)
  - เปิด PDF ในเบราว์เซอร์ (native Chrome PDF viewer ผ่าน remote-devices Browser pane) แสดงผลได้ แต่**คลิก/scroll/พิมพ์ในตัว viewer ไม่ได้เลย** — เครื่องมือ automation รายงาน error "lands in an embedded frame with no resolvable origin" ทุกครั้งที่คลิกในพื้นที่ viewer (ไม่ว่าจะปุ่ม toolbar, thumbnail, หรือตัวเอกสาร) ทำให้เปลี่ยนหน้า/ดาวน์โหลด/ซูมผ่าน UI ไม่ได้
  - `read_page`/`get_page_text` บนหน้า PDF viewer คืนค่าว่างเปล่า (viewer ไม่ expose text layer ให้ accessibility tree หรือ DOM)
  - ลองดึง PDF ด้วย `fetch()` ในหน้าเว็บ (same-origin กับ baac.or.th) **สำเร็จ** (ได้ ArrayBuffer ครบ ~968KB) แต่แปลงเป็น base64 แล้วส่งออกมาด้วยเทคนิค `<pre>`+`get_page_text` ไม่ได้ผล เพราะไฟล์บันทึกอัตโนมัติก็ถูกตัดที่ ~50-53KB เท่ากัน (ดูหมายเหตุในหัวข้อ KBank ด้านบน) — ไฟล์ PDF 1 ไฟล์มี base64 ยาวถึง ~1.29MB ต้องแบ่งส่งเป็น ~29 ก้อนต่อ 1 PDF ซึ่งไม่คุ้มเมื่อมี PDF หลายสิบไฟล์
  - ลองโหลดไลบรารี `pdf.js` จาก CDN (cdnjs.cloudflare.com) ในหน้า baac.or.th เพื่อ parse PDF ในเบราว์เซอร์เอง **ถูกบล็อกด้วย CSP ของเว็บ** (ยืนยันด้วย `securitypolicyviolation` event: `script-src-elem` บล็อก cdnjs) — ลองเปิด origin อื่น (example.com) เพื่อโหลด pdf.js ได้สำเร็จ แต่ fetch PDF ข้าม origin จาก baac.or.th ไปที่ example.com ติด CORS (baac.or.th ไม่ส่ง CORS header อนุญาต)
- **การตัดสินใจของผู้ใช้ (2026-09-04)**: ถามผู้ใช้ว่าจะ (ก) ข้าม BAAC ไปก่อน (ข) ทำแบบคุณภาพต่ำกว่า (geocode จากที่อยู่/อำเภอ) หรือ (ค) ให้ผู้ใช้ดาวน์โหลด PDF เองแล้วให้ Claude อ่าน — **ผู้ใช้เลือก "ข้ามไปก่อน"** ดังนั้น BAAC ไม่ได้รวมอยู่ใน `bank-npa-land.json` — ถ้าจะทำในอนาคต ผู้ใช้ต้องเป็นคนเปิดคำขอใหม่เอง (ไม่ต้องเสนอซ้ำเอง)

## ข้อจำกัดสำคัญที่พบ (มีผลต่อการรีเฟรชข้อมูลรายสัปดาห์)
- ห้องทำงานคลาวด์ของ Claude (รวมถึง "งานตามกำหนดเวลา" ที่รันแบบไม่มีคนเฝ้า) **ถูกบล็อกไม่ให้เชื่อมต่อโดเมนธนาคารโดยตรง** (ยืนยันแล้วด้วย curl ไปที่ GSB, KBank และ BAAC → โดน proxy ปฏิเสธ 403 ทั้งหมด; **device_bash บนเครื่องผู้ใช้ก็โดนบล็อกเหมือนกัน** เพราะใช้ egress allowlist เดียวกัน — ยืนยันแล้วกับ kasikornbank.com) — allowlist เดียวกันนี้ยังบล็อก cdnjs.cloudflare.com/cdn.jsdelivr.net จากคลาวด์แซนด์บ็อกซ์ด้วย (เจอตอนพยายามทดสอบเว็บด้วย headless browser ในคลาวด์ — ใช้วิธี stub library แทนตามที่บันทึกไว้ในหัวข้อ #6/#7/#8/#9 ด้านบน) — **แต่ `WebFetch` tool ใช้เส้นทางเน็ตเวิร์กคนละเส้น เข้าถึง cdnjs.cloudflare.com/data.jsdelivr.com/cdn.jsdelivr.net และเว็บ ToS ต่างๆ (open-meteo.com, wiki.openstreetmap.org ฯลฯ) ได้** (ใช้ประโยชน์ตรงนี้ในหัวข้อ #10 เพื่อดึง SRI hash และหัวข้อเรื่องกฎหมาย/ใบอนุญาตด้านบน โดยไม่ต้องพึ่ง curl/Playwright โดยตรง)
- ทางเดียวที่เข้าถึงข้อมูลได้ตอนนี้คือผ่าน **เบราว์เซอร์ (Chrome extension หรือ Browser pane ผ่าน remote-devices) ที่เชื่อมต่อ Claude อยู่** หรือ `WebFetch` (สำหรับ metadata/ไฟล์สาธารณะที่ไม่ต้องคลิก UI)
- ดังนั้น **การอัปเดตข้อมูลอัตโนมัติทุกสัปดาห์แบบไม่มีคนเฝ้ายังทำไม่ได้จริง** ต้องมีคนเปิดแชทกับ Claude ตอนที่เบราว์เซอร์เชื่อมต่ออยู่ (หรือรอ org admin เปิด network allowlist ให้โดเมนพวกนี้ในอนาคต)
- **ข้อจำกัดของเครื่องมือดึงข้อมูลก้อนใหญ่**: ทั้ง `javascript_tool` (คืนค่าตรง ~1000-1300 ตัวอักษร) และไฟล์ที่ `get_page_text` บันทึกอัตโนมัติเมื่อเนื้อหาเกิน token cap (~50,000-53,000 ตัวอักษร) ต่างก็มีเพดานตายตัว — ใช้ได้ดีกับ JSON/ข้อความระดับ ≤~45KB ต่อการดึงหนึ่งครั้ง (พอสำหรับ API response ทั่วไป 20-1000 แถว) แต่ไม่พอสำหรับไฟล์ไบนารีขนาดใหญ่ (เช่น PDF ที่แปลงเป็น base64) ต้องแบ่งเป็นหลายก้อนซึ่งไม่คุ้มถ้ามีไฟล์จำนวนมาก — `WebFetch` เองก็มี cap ประมาณ 200,000 ตัวอักษรต่อครั้งและผ่านโมเดลสรุป/แปลง HTML→markdown จึง**ไม่เหมาะเอาไปคำนวณ hash/checksum ของไฟล์แบบ byte-exact** (ใช้ metadata API ที่คืน hash สำเร็จรูปมาแทนดีกว่า เช่น jsDelivr data API) แต่**เหมาะมากสำหรับอ่านสรุปเนื้อหาเว็บ ToS/นโยบายต่างๆ** (ใช้ในหัวข้อเรื่องกฎหมาย/ใบอนุญาตด้านบน)
- **เทคนิคทดสอบ JS ของเว็บนี้ (ค่าเริ่มต้นที่แนะนำ, อัปเดตล่าสุด 2026-09-04 หัวข้อ #10)**: cdnjs.cloudflare.com/cdn.jsdelivr.net (ที่โหลด maplibre-gl และ turf) ถูกบล็อกโดย network egress allowlist ของแซนด์บ็อกซ์คลาวด์สำหรับ curl/Playwright โดยตรง (เหมือนโดเมนธนาคาร) — ทดสอบด้วยการ stub ทั้ง `window.maplibregl` (Map/Popup/Marker/NavigationControl/ScaleControl — **สำคัญ: constructor ของ Map ต้อง parse `opts.style.sources`/`opts.style.layers` เข้า `_sources`/`_layers` ด้วย เพราะโค้ดจริงประกาศ source/layer ทั้งหมดไว้ใน style object ตอนสร้าง ไม่ได้เรียก `addSource()`/`addLayer()` แยก** — และถ้าจะทดสอบฟีเจอร์ที่เกี่ยวกับ pitch/camera ต้อง stub `getPitch()`/`easeTo()` เพิ่มด้วย) และ `window.turf` (point/distance/destination/circle/lineString — ใช้สูตร haversine/destination bearing เขียนเองง่ายๆ พอ ไม่ต้องพึ่งไลบรารีจริง) ผ่าน Playwright `page.addInitScript()` รวมกับ `page.route()` เพื่อบล็อก/stub request ภายนอกอื่นๆ (mock `api.open-meteo.com` และ `overpass` ให้คืนค่าจำลองด้วยถ้าต้องทดสอบ pipeline วิเคราะห์แปลง/POI/อาคาร 3 มิติให้ครบ — แยก response ตามเนื้อหา query เช่นเช็คว่า postData มีคำว่า "building" หรือไม่ เพื่อคืนข้อมูลอาคารกับข้อมูล POI แยกกันถูกต้อง) แล้วรันกับไฟล์ `bank-npa-land.json` จริงผ่าน local `python3 -m http.server` (ใช้ `nohup ... & disown` แบบรันตรงไม่ subshell ถ้าจะ background server ค้างไว้ระหว่าง tool call หลายครั้ง — สังเกตว่า `pkill`/compound `pkill+sleep+nohup` ในคำสั่งเดียวมักคืน exit code 144 ในแซนด์บ็อกซ์นี้แม้จะทำงานสำเร็จจริง อย่าตกใจ ให้เช็คผลจริงด้วย `ls`/`ps aux` แทนการเชื่อ exit code เพียงอย่างเดียว) — ใช้ `node --check` กับเนื้อหา `<script>` ที่ดึงออกมาด้วย regex เพื่อเช็ค syntax ก่อนเสมอ

## วิธีรีเฟรชข้อมูลรอบถัดไป (ทำซ้ำ manual)

### ออมสิน (GSB)
1. เปิดเบราว์เซอร์เชื่อมต่อกับ Claude
2. เข้า `https://npa-assets.gsb.or.th/asset/npa/all?asset_type_id=341&page_size=200`
3. รัน `window.__NEXT_DATA__.props.pageProps.list.data.rows` เพื่อดึงรายการ (asset_group_id_npa, asset_type_id, ที่อยู่, ราคา)
4. fetch หน้ารายละเอียดทีละแปลง (หรือ batch ด้วย Promise.all) เพื่อดึง `info.latitude`/`info.longitude`

### กรุงไทย (KTB)
1. เปิดเบราว์เซอร์เชื่อมต่อกับ Claude ไปที่ `https://npa.krungthai.com`
2. เรียก `POST https://npa.krungthai.com/api/v1/product/searchAll` ด้วย body `{"paging":{"currentPage":N,"rowsPerPage":100,"totalRows":0},"typeProp":["1"]}` วนตั้งแต่ page 1 จนครบ (เช็ค `totalRows`)
3. แต่ละ item มี `lat`/`lon` ให้เลย ไม่ต้องเปิดหน้ารายละเอียดเพิ่ม (ยกเว้นต้องการ `collGrpId` เพื่อสร้างลิงก์ `property-detail/{collGrpId}` — มีอยู่ใน response เดียวกัน)
4. กรองพิกัดผิดปกติด้วย bounding box คร่าวๆ ของไทย (`5<=lat<=21`, `96<=lon<=106`) และเช็ค lat≠lon

### กสิกรไทย (KBank)
1. เปิดเบราว์เซอร์เชื่อมต่อกับ Claude ไปที่ `https://www.kasikornbank.com/th/propertyforsale`
2. เรียก `POST https://www.kasikornbank.com/Custom/KWEB2020/NPA2023Backend13.aspx/GetProperties` ด้วย body `{"filter":{"CurrentPageIndex":N,"PageSize":50,"SearchPurpose":["AllProperties"],"PropertyTypes":["01"],"Provinces":[],"Amphurs":[],"Ordering":"new"}}` วนหน้าจนครบ (เช็ค `Data.TotalRows`)
3. แต่ละ item มี `Latitude`/`Longtitude`/`PropertyID` ให้เลย ไม่ต้องดึงเพิ่ม — ลิงก์หน้ารายละเอียดคือ `https://www.kasikornbank.com/th/propertyforsale/detail/{PropertyID}.html`
4. **ดึงข้อมูลก้อนใหญ่ออกมาด้วยวิธีฝัง `<pre>` ในหน้า + อ่านด้วย `get_page_text`** (แทนการ return ค่าจาก javascript_tool ตรงๆ ซึ่งถูกตัดที่ ~1000 ตัวอักษร) — ถ้าเนื้อหาใหญ่พอ (แต่ไม่เกิน ~45KB ต่อครั้ง) ระบบจะบันทึกเป็นไฟล์อัตโนมัติให้อ่านต่อด้วย Python ได้ครบทุกแถว

### ไทยพาณิชย์ (SCB)
1. เปิดเบราว์เซอร์เชื่อมต่อกับ Claude ไปที่ `https://asset.home.scb`
2. ต้องคลิกจริงผ่าน UI (dropdown ประเภททรัพย์ → "ที่ดินเปล่า" → ปุ่มค้นหา) เพื่อให้ navigate ไป URL ที่มี `asset-type=vacant_land` — programmatic set ค่าไม่ทำงาน
3. เรียก `GET https://asset.home.scb/api/project/cmd?type=project&page=1&limit=500&sortBy=all&command=get_project&asset-type=vacant_land&...` (คัด query param อื่นจาก URL ที่ navigate ไปแล้ว) ได้ผลลัพธ์ครบในครั้งเดียว (`d` = array, `total` = จำนวนรวม)
4. แต่ละ item มี `latitude`/`longitude`/`slug` ให้เลย — ลิงก์คือ `https://asset.home.scb/project/{slug}`

### เกียรตินาคินภัทร (KKP)
1. เปิดเบราว์เซอร์เชื่อมต่อกับ Claude ไปที่ `https://kkppropify.kkpfg.com/th/products?type=ที่ดินเปล่า`
2. ถ้าผลลัพธ์ขึ้น 0 รายการ ให้เปิด modal ตัวกรอง แล้วคลิกยกเลิก chip "โครงการ" ที่ติดมาด้วย (ดูรายละเอียดด้านบน)
3. monkey-patch `window.fetch` ก่อน แล้วคลิกเลขหน้า (native `.click()` ใช้ได้) เพื่อดักจับ response ของแต่ละหน้า (1-6, หน้าละ 20 ยกเว้นหน้าสุดท้าย) — อย่าลืมคลิกหน้า 1 ด้วยแม้จะเป็นหน้าเริ่มต้น เพราะ prefetch อาจทำให้พลาด response
4. ดึง JSON จากบรรทัดที่ขึ้นต้นด้วย `1:` ในแต่ละ response (Next.js Flight format) → `data.items[]`
5. field `district` คือ "จังหวัด, อำเภอ" (สลับกับสามัญสำนึก) — แยก province/district ให้ถูกทิศทาง; ราคาใช้ `specialPrice` ก่อน ถ้าไม่มีค่อยใช้ `price`

### รวมและอัปโหลด
6. ประกอบเป็น GeoJSON ใหม่ (รวมทุกธนาคาร) ทับไฟล์ `bank-npa-land.json` เดิม แล้วส่งให้ผู้ใช้อัปโหลดทับบน GitHub Pages

## งานที่ยังไม่ได้ทำ / รอผู้ใช้ตัดสินใจ
- **BAAC/ธกส**: ผู้ใช้ตัดสินใจข้ามไปก่อนแล้ว (2026-09-04) — ไม่ต้องเสนอซ้ำ รอผู้ใช้เปิดคำขอเองถ้าเปลี่ยนใจ
- ยังไม่ได้ทำธนาคารอื่นนอกจาก GSB/KTB/KBank/SCB/KKP (เช่น กรุงเทพ, กรุงศรี, บสก./BAM, SAM) — รอผู้ใช้ระบุเพิ่มถ้าต้องการ
- ยังไม่ได้ตั้งระบบแจ้งเตือน/scheduled task ให้ผู้ใช้เปิดแชทมารีเฟรชข้อมูลทุกสัปดาห์ (ถ้าต้องการ ใช้ create_trigger แบบ weekly ที่ส่งข้อความเตือนผู้ใช้ให้เปิดเบราว์เซอร์แล้วขอ Claude รีเฟรช — ไม่ใช่ full-auto) — เคยเสนอผู้ใช้แล้ว ยังไม่ได้รับคำตอบ
- เรื่องน้ำท่วมซ้ำซากผ่าน GISTDA official API และแล้งซ้ำซาก: พักไว้ตามที่ผู้ใช้เลือกไว้ก่อนหน้านี้ (Cloudflare Worker `gistda-flood-proxy.methunpm2001.workers.dev` ยัง deploy ค้างอยู่แต่ไม่ได้ใช้งาน)
- ความสูงอาคาร 3 มิติ (หัวข้อ #9) เป็นค่าประมาณ (default 6 เมตรถ้า OSM ไม่มี tag) ไม่ใช่ความสูงจริงเสมอไป — ถ้าผู้ใช้ต้องการความแม่นยำสูงขึ้นในอนาคต ต้องพิจารณาแหล่งข้อมูลอาคารอื่น (เช่น ภาพถ่ายดาวเทียม/LiDAR) ซึ่งยังไม่ได้สำรวจ
- **การสลับ CDN เป็น jsDelivr+SRI (หัวข้อ #10)**: ยืนยันแล้วว่าใช้งานได้จริงบนเบราว์เซอร์จริงจากสกรีนช็อตที่ผู้ใช้ส่งมา (แผนที่/3D ทำงานปกติ) — ปิดประเด็นนี้แล้ว
- **exaggeration ภูมิประเทศ 3 มิติ (หัวข้อ #11)**: ลดจาก 4.5 → 1.8 แล้ว แต่ยังไม่ได้ live-verify ผลลัพธ์จริง — รอผู้ใช้เช็คหลังอัปโหลดว่า "ภูเขาปลอม" หายไปสมใจหรือยังเหลืออยู่บ้างตอนซูมออกกว้างมากๆ ถ้ายังเหลืออยู่อาจต้องลดเพิ่มหรือทำ zoom-dependent exaggeration
- **เรื่องกฎหมายข้อมูลที่ดินธนาคาร**: ยังเป็นความเสี่ยงเปิดอยู่ ผู้ใช้ยังไม่ได้ตัดสินใจ/ดำเนินการอะไรเพิ่ม — ถ้าจะขยายสเกล/เผยแพร่วงกว้างในอนาคตควรกลับมาคุยเรื่องนี้อีกครั้ง หรือปรึกษาทนายโดยเฉพาะเรื่องข้อมูลธนาคาร
