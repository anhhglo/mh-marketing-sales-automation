# MH-03 Bot Messenger — Đánh giá thực tế 2026-05-30

## 1. Tóm tắt test

| Hạng mục | Kết quả |
|----------|---------|
| Khách test | Anh Hiếu — <EXAMPLE_PHONE> (cá nhân, 2 xe nâng 48V Lithium) |
| Kênh | Messenger (comment "giá" → DM tự động → bot tiếp quản) |
| Hoàn thành Q1–Q14 | ✅ Có (sau ~20 tin) |
| Tạo Lead Lark | ✅ Record tạo thành công |
| Tạo NKT Lark | ✅ Record tạo thành công |
| Alert Sale nhóm | ✅ Gửi đúng |
| Lỗi fallback "Xin lỗi..." | ❌ 2 lần |
| 2 tin cùng lúc | ❌ Race condition |

---

## 2. Luồng hoạt động đúng

```
Comment "giá" → Meta DM tự động
→ Khách: "Mình cần tư vấn pin xe nâng"
→ Bot: hỏi Tên + SĐT + Công ty          ✅
→ Bot: hỏi Q1 (số xe)                   ✅
→ Bot: hỏi Q2/Q3 (loại pin)             ✅
→ Khách hỏi giá → Bot từ chối khéo     ✅
→ Bot hỏi Q4–Q13 tuần tự               ✅ (có vài trùng)
→ Bot hoàn thành → ghi Lark + alert     ✅
```

---

## 3. Các lỗi và vấn đề phát hiện

### 3.1 🔴 CRITICAL — Race condition khi khách gửi 2 tin liên tiếp

**Mô tả:**
Khách gửi 2 tin nhanh liên tiếp:
```
Tin 1: "Dùng pin lithium rồi loại 48v"
Tin 2: "Chạy 8h"  (cách nhau ~1-2 giây)
```
**Kết quả:** Tin 2 bị fallback "Xin lỗi, tôi chưa hiểu rõ. Bạn có thể nói lại không?"

**Nguyên nhân kỹ thuật:**
- Mỗi tin kích hoạt 1 n8n execution riêng biệt
- Execution 1 (tin 1) đang chạy → chưa save session
- Execution 2 (tin 2) đọc session cũ → không có context "48v lithium"
- AI nhận tin "Chạy 8h" đơn độc → không đủ JSON → fallback

**Tần suất:** Xảy ra bất cứ khi nào khách gửi 2 tin trong vòng 7-10 giây.

**Mức ảnh hưởng:** Trải nghiệm xấu — khách phải nhắn lại, cảm giác bot dở.

---

### 3.2 🔴 CRITICAL — Fallback khi khách trả lời phức tạp / đặt câu hỏi ngược

**Mô tả:**
Tin: "Tư vấn cho tôi các loại pin lithium trước để tôi tham khảo xong mới quyết định"
→ Bot: "Xin lỗi, tôi chưa hiểu rõ. Bạn có thể nói lại không?"

**Nguyên nhân:**
Bot được lập trình "không báo giá, không nêu model" → AI cố gắng tạo phản hồi dài + JSON
→ max_tokens=1000 có thể bị cắt giữa chừng → JSON không hợp lệ → catch() → fallback

**Vấn đề thứ 2:** Bot không có dữ liệu sản phẩm → không thể tư vấn gì cả.

---

### 3.3 🟡 MEDIUM — Bot hỏi lại câu đã được trả lời

**Ví dụ:**
- Khách: "Mình cần mua 2 pin lithium" → Bot vẫn hỏi lại "đang dùng loại gì" thêm 1 lần
- Khách đã cho biết "2 xe nâng" nhưng bot hỏi thêm "số xe nâng là bao nhiêu"

**Nguyên nhân:** Mỗi lượt AI chỉ extract từ tin nhắn hiện tại, không lookup lại toàn bộ history thật tốt khi thông tin nằm trong câu cách 2-3 lượt.

---

### 3.4 🟡 MEDIUM — 2 reply bot liên tiếp cùng lúc

**Mô tả:**
Khách hỏi "Giá bao nhiêu" → Bot gửi 2 tin độc lập:
1. "Sale MHpower sẽ liên hệ tư vấn sau khi thu thập đủ thông tin!"
2. "Dạ giá pin sẽ tùy thuộc vào loại xe..."

**Nguyên nhân:** Khi khách gửi "Mình cần mua 2 pin lithium" và "Giá bao nhiêu" trong 2 tin riêng → 2 execution chạy → 2 reply độc lập.

---

### 3.5 🟡 MEDIUM — Dữ liệu Lark không chuẩn

**Vấn đề phát hiện sau khi xem record:**

| Field | Giá trị expected | Thực tế |
|-------|-----------------|---------|
| Tên công ty | Không có (cá nhân) | "-" hoặc trống |
| Q2 loại pin cần thay | "Lithium 48V" | Checkbox true nhưng text trống |
| Q5 ảnh hưởng CV | "Không" → False | ✅ False |
| Q14 dự kiến thay | "trong tuần này" → < 30 ngày | Cần kiểm tra mapping |
| Tóm tắt Q15 | Đầy đủ | ✅ Đúng |

**Gốc rễ:** Các field Q1-Q13 trong NKT là checkbox (true/false) — chỉ biết "có trả lời" hay "không", không lưu nội dung cụ thể. Nội dung chi tiết chỉ có ở field Q15 (tóm tắt).

---

### 3.6 🟢 LOW — Bot cứng nhắc trong cách hỏi

**Biểu hiện:**
- Luôn hỏi từng câu theo thứ tự, không đặt 2-3 câu liên quan vào 1 tin
- Khi khách trả lời dài (đã trả lời 2-3 câu), bot vẫn hỏi từng cái một
- Không có phong cách "bán hàng" — chỉ collect data máy móc

---

## 4. Đề xuất cải thiện

### 4.1 Fix race condition (2 tin cùng lúc)

**Giải pháp đơn giản nhất:** Thêm debounce lock vào session — nếu session đang xử lý (lock=true), tin thứ 2 sẽ delay hoặc merge.

```javascript
// Load Session State — thêm check lock
if (session.processing_lock) {
  // Session đang xử lý — đợi 2s rồi thử lại (hoặc skip)
  // Có thể merge: append msg_text vào pending queue
}
session.processing_lock = true;
// ... sau khi save session: session.processing_lock = false
```

**Giải pháp tốt hơn:** Dùng n8n Queue Mode hoặc add delay node 3s trước Load Session.

---

### 4.2 Tăng max_tokens và cải thiện fallback

```javascript
// Build Messenger AI Request
max_tokens: 2000  // Tăng từ 1000 → 2000 để tránh JSON bị cắt

// Parse Messenger AI Response — cải thiện catch:
} catch(e) {
  // Thay vì fallback cứng, thử extract partial reply
  const replyMatch = content.match(/"reply"\s*:\s*"([^"]+)"/);
  const replyText = replyMatch ? replyMatch[1] : 'Bạn có thể nói rõ hơn không ạ?';
  parsed = { reply: replyText, extracted: {}, is_complete: false };
}
```

---

### 4.3 Thêm Data Store sản phẩm (Admin configurable)

Tạo Lark Table hoặc n8n Datatable: **MHBot_Products**

| field | type | example |
|-------|------|---------|
| model | text | "MHP-48V-100Ah" |
| voltage | number | 48 |
| capacity_ah | number | 100 |
| runtime_hours | number | 8 |
| use_case | text | "Xe nâng 2-3 tấn" |
| price_range | text | "Liên hệ để báo giá" |
| notes | text | "Phù hợp ca 8h, sạc nhanh 3h" |

→ `Build Messenger AI Request` fetch danh sách sản phẩm → inject vào prompt:

```javascript
const products = /* fetch từ datatable */;
const productInfo = products.map(p =>
  `- ${p.model}: ${p.voltage}V ${p.capacity_ah}Ah, chạy ${p.runtime_hours}h/lần sạc`
).join('\n');

// Thêm vào prompt:
+ 'DANH SÁCH SẢN PHẨM MHPOWER:\n' + productInfo + '\n\n'
+ 'Khi khách hỏi giá/sản phẩm: giới thiệu model phù hợp, nhưng KHÔNG báo giá — sale sẽ báo giá cụ thể.\n\n'
```

---

### 4.4 Thêm Admin Guidelines (configurable từ Lark)

Tạo Lark Table: **MHBot_Config**

| key | value |
|-----|-------|
| greeting | "Chào anh/chị, MHpower tư vấn pin lithium xe nâng ạ!" |
| price_response | "Giá pin tùy theo specs xe — sale sẽ báo giá sau khi tư vấn ạ" |
| escalation_keywords | "khiếu nại, bảo hành, lỗi, hỏng" |
| escalation_response | "Anh/chị để lại SĐT, kỹ thuật MHpower sẽ liên hệ ngay ạ" |
| out_of_scope | "Câu hỏi ngoài phạm vi — chuyển ngay cho sale" |

→ Bot đọc config này khi start → inject vào system prompt.

---

### 4.5 Cải thiện prompt — bớt cứng nhắc

Thêm vào prompt của bot:

```
QUY TẮC BỔ SUNG:
- Nếu khách đã trả lời nhiều câu trong 1 tin, extract hết luôn, KHÔNG hỏi lại
- Nếu khách hỏi giá: "${price_response}" rồi tiếp tục hỏi câu tiếp
- Có thể hỏi 2 câu liên quan trong 1 tin để tự nhiên hơn
- Nếu khách là cá nhân (không có công ty): ghi company = "Cá nhân"
- Nếu tin nhắn quá ngắn và không rõ nghĩa (chỉ 1-2 từ): hỏi lại khéo léo thay vì fallback
```

---

## 5. Kế hoạch ưu tiên

### Sprint 1 — Fix bug (ngay)
- [ ] Tăng max_tokens: 1000 → 2000
- [ ] Cải thiện catch fallback (extract partial)
- [ ] Thêm rule "cá nhân" vào prompt
- [ ] Fix race condition đơn giản: thêm 2s delay trước `Build Messenger AI Request`

### Sprint 2 — Data & Config (1-2 tuần)
- [ ] Tạo Lark Table `MHBot_Products` với danh sách sản phẩm
- [ ] Tạo Lark Table `MHBot_Config` với admin guidelines
- [ ] Update `Build Messenger AI Request` để fetch và inject dữ liệu
- [ ] Test lại với khách thật

### Sprint 3 — UX cải thiện (tương lai)
- [ ] Bot gom thông tin thông minh hơn (không hỏi lại nếu đã biết)
- [ ] Thêm rich message (nút bấm, quick reply) nếu Meta cho phép
- [ ] Dashboard theo dõi conversion rate: bao nhiêu conversation → completed

---

## 6. Đánh giá tổng thể

| Tiêu chí | Điểm | Ghi chú |
|----------|------|---------|
| Luồng Q1-Q14 | 7/10 | Hoàn thành nhưng cứng, hỏi lại |
| Xử lý ngoại lệ | 4/10 | Fallback khi tin phức tạp |
| Race condition | 3/10 | 2 tin cùng lúc = lỗi |
| Lưu data Lark | 8/10 | Đúng field, đúng record, có link |
| Alert Sale | 10/10 | Hoàn hảo |
| Tổng | **6.4/10** | Production-ready ở mức cơ bản |

**Kết luận:** Bot đã hoạt động end-to-end thành công trong test thực tế. Các lỗi phát hiện không chặn luồng chính — khách vẫn hoàn thành và sale vẫn nhận được lead. Cần fix race condition và thêm product data trước khi scale traffic cao.

---

*Đánh giá lúc: 2026-05-30 | Tester: khách thật — Anh Hiếu*
*Workflow: MH-03 eZgfYcBRvfuvGl0c | n8n v2.20.9*
