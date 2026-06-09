# Hướng dẫn vận hành — Hệ thống MH Marketing & Sales Automation
**Cập nhật:** 2026-06-05 | **Dành cho:** Quản lý · Marketing · Sale · HR · Kỹ thuật

> ⚠️ **Phạm vi thực tế (đọc trước):** Hệ thống hiện tự động hóa **thu lead → bot AI sàng lọc → ghi CRM → báo Sale/HR**. Các bước **báo giá / đơn hàng / lắp đặt / bảo hành KHÔNG còn tự động** (pipeline rental cũ đã gỡ) — sau khi lead vào Lark, **Sale làm thủ công** từ đó. Bản hướng dẫn này mô tả đúng những gì hệ thống đang làm.

---

## Mục lục
1. Tổng quan
2. Vai trò: Khách hàng
3. Vai trò: Marketing
4. Vai trò: Sale
5. Vai trò: HR (tuyển dụng)
6. Vai trò: Quản lý
7. Luồng end-to-end
8. Lark Base — bảng dùng hàng ngày
9. Hệ thống tự làm gì thay bạn
10. Xử lý sự cố thường gặp

---

## 1. Tổng quan

### Hệ thống làm gì?
Tự động **nhận khách tiềm năng từ nhiều kênh**, dùng **bot AI** hỏi nhu cầu, **ghi hồ sơ vào Lark** và **báo đội Sale/HR**. Con người làm phần tư vấn/chốt từ sau khi nhận thông báo.

### Hai thương hiệu + tuyển dụng
| Mảng | Sản phẩm/việc | Kênh chính |
|---|---|---|
| **MHpower** | Pin lithium SuperV, ắc quy cho xe nâng điện | Facebook Ads, Messenger |
| **MHrental** | Cho thuê & bán xe nâng (hàng/người/scissor/boom) | Google Ads, Messenger, telesales |
| **Tuyển dụng** | 6 vị trí (kỹ thuật, kinh doanh, kế toán) | Messenger (bot tự nhận) |

### Sơ đồ tổng quan (đúng thực tế)
```
KHÁCH HÀNG                       HỆ THỐNG TỰ ĐỘNG                   TEAM NỘI BỘ
────────────                     ──────────────────                ──────────────
Comment QC Facebook   ─────►  Meta gửi tin riêng → bot tiếp quản   
Nhắn Messenger        ─────►  MH-03 bot hỏi nhu cầu (pin/xe/việc)  ► Sale (Power/Rental) hoặc HR nhận alert
Điền form FB Lead Ads ─────►  MH-03 → chấm điểm → ghi CRM           ► (kèm hồ sơ nhu cầu đầy đủ)
Điền form Google      ─────►  MH-04 → chấm điểm → ghi CRM
Zalo/Hotline/API      ─────►  MH-01 → chấm điểm → ghi CRM
Website               ─────►  MH-WEBSITE (⚠️ xem mục 10)
                                     │
                                     ├─ Khách im → bot tự nhắc 10 phút / 4 giờ / 24 giờ
                                     ├─ Khách cũ nhắn nhu cầu mới → tạo lead mới + báo "🔁 khách cũ"
                                     └─ 8h sáng → Dashboard báo cáo tổng lead lên Lark
                                     ↓
                        [Sale gọi khách, tư vấn, báo giá, chốt — THỦ CÔNG]
```

---

## 2. Vai trò: Khách hàng

Khách **không cần biết gì về hệ thống**. Hành trình tự nhiên:

### Kịch bản A — Messenger / Comment
1. Khách comment quảng cáo hoặc nhắn Page → bot trả lời, hỏi: *"Anh cần thuê/mua xe nâng, hay cần pin lithium ạ?"*
2. Bot hỏi tiếp thông tin theo nhu cầu (pin: số xe, loại pin, giờ làm...; xe nâng: loại xe, chiều cao, thời gian thuê...).
3. Khi đủ thông tin → Sale nhận hồ sơ chi tiết, gọi lại. Khách **được trả lời 24/7**, không phải chờ.

### Kịch bản B — Form quảng cáo (Facebook / Google)
1. Khách điền form trên FB/Google → dữ liệu tự vào Lark, AI chấm điểm.
2. Sale nhận thông báo, gọi lại (lead A "rất nóng" được ưu tiên).

### Kịch bản C — Hỏi việc làm
1. Khách nhắn hỏi tuyển dụng → bot tư vấn vị trí phù hợp (6 vị trí), xin tên/SĐT/khu vực.
2. Bot đưa liên hệ **Ms. Trang <RECRUIT_CONTACT_PHONE> / <RECRUIT_CONTACT_EMAIL>** + báo bộ phận HR.

---

## 3. Vai trò: Marketing

### Công việc
Marketing chạy chiến dịch; hệ thống **tự nhận lead** từ các kênh. Không cần nhập tay lead quảng cáo.

### Thiết lập (1 lần — đã xong)
| Việc | Trạng thái |
|---|---|
| Facebook Ads/Lead Form → MH-03 | ✅ |
| Google Lead Form → MH-04 | ✅ |
| Bảng `leads marketings` (2 base) nhận lead quảng cáo | ✅ |

### Theo dõi lead quảng cáo
Mở Lark Base → bảng **leads marketings** (mỗi base 1 bảng):
- MHpower: `<TBL_PW_MARKETING>` · MHrental: `<TBL_RT_MARKETING>`
- Mỗi dòng: Họ tên, SĐT, Email, **Nguồn form** (Meta/Google), Tên chiến dịch, Ad/Form ID, Nhu cầu (pin/xe nâng), Nội dung form, **AI Score** (A/B/C/D), Ngày nhận, Trạng thái.
→ Lọc theo Nguồn form / Score để biết kênh nào ra lead chất lượng.

### Báo cáo sáng
8h mỗi ngày, hệ thống gửi **Báo cáo buổi sáng** vào nhóm Lark: tổng lead, tách Facebook/Google, số lead Hot(A)/Tiềm năng(B). *(Lưu ý: phần breakdown chi tiết theo brand/pipeline hiện còn hạn chế — xem mục 10; số "tổng lead" là chính xác.)*

### ✅ Đọc số liệu chi phí quảng cáo (CẬP NHẬT 06-08 — ĐÃ CHẠY)
Phần đọc **chi phí/click/CTR** từ **Meta + Google Ads** nay **ĐÃ CHẠY** (Google Ads Basic Access duyệt 06-06). Mỗi sáng hệ thống tự đưa chi phí Meta (tách Power/Rental) + chi phí Google vào **Báo cáo buổi sáng**; thêm **Báo cáo Tuần** và **Báo cáo Tháng**. Marketing không cần nhập tay số liệu quảng cáo. *(Hiện số liệu chỉ ở dạng báo cáo text, chưa lưu thành bảng để vẽ biểu đồ xu hướng.)*

---

## 4. Vai trò: Sale

### Nhận thông báo (theo brand)
Mở Lark → nhóm của bạn:
- **MHpower Sales Alert** — mọi lead PIN.
- **MHrental Sales Alert** — mọi lead XE NÂNG.

Các loại thông báo:
- `🔥 LEAD FACEBOOK / GOOGLE - GỌI NGAY!` → lead từ form quảng cáo (có tên, SĐT, brand, mức gấp, AI nhận xét).
- `🤖 LEAD MỚI QUA MESSENGER BOT` → bot đã hỏi đủ Q1–Q14 (pin) hoặc R1–R5 (xe nâng), kèm toàn bộ câu trả lời.
- `🏗️ LEAD XE NÂNG (MHrental) QUA MESSENGER BOT` → lead thuê xe.
- `🔁 KHÁCH CŨ NHẮN LẠI — CÓ NHU CẦU MỚI` → khách từng chốt hồ sơ, nay có nhu cầu khác.

### Quy trình xử lý
**Bước 1 — Gọi ngay.** Lead hot (A) giảm giá trị theo giờ → gọi trong 5–10 phút.

**Bước 2 — Mở Lark xem chi tiết.**
- Lead PIN: Base MHpower → **3.1.0 Leads Copy** (thông tin liên hệ) + **3.1.0 Deals Copy** (toàn bộ nhu cầu Q1–Q14) + **Lịch sử CSKH Copy** (nhật ký trao đổi).
- Lead XE NÂNG: Base MHrental → **1. Leads liên hệ Copy** + **2. Lịch sử CSKH Copy**.
→ Mở Deals/CSKH Copy là thấy ngay khách đã trao đổi gì, **không cần hỏi lại từ đầu**.

**Bước 3 — Tư vấn & báo giá (thủ công).** Bot KHÔNG báo giá. Sale tư vấn cấu hình, gửi báo giá, thương lượng.

**Bước 4 — Cập nhật.** Ghi tiến độ vào bảng Lark (trạng thái khách mới/cũ, ghi chú). Việc tạo cơ hội/báo giá/đơn hàng làm thủ công theo quy trình hiện hành của công ty (chưa có workflow tự động).

### Follow-up tự động (bot làm thay phần nhắc khách)
Khách nhắn rồi im (chưa đủ thông tin) → **bot tự nhắn nhắc khách** qua Messenger ở mốc **10 phút / 4 giờ / 24 giờ**. Sale không phải canh — chỉ cần xử lý khi khách phản hồi và hồ sơ "chín" được báo lên nhóm.

---

## 5. Vai trò: HR (tuyển dụng)

### Nhận ứng viên
Khi khách nhắn Messenger hỏi việc làm, bot tự tư vấn vị trí và thu thập thông tin, rồi gửi vào **nhóm HR** (`<LARK_CHAT_HR>`):
```
🧑‍💼 ỨNG VIÊN MỚI QUA MESSENGER
📌 Vị trí ứng tuyển: ...
👤 Họ tên · 📞 SĐT · 📍 Khu vực · 🧰 Kinh nghiệm
(Bot đã gửi liên hệ Ms. Trang <RECRUIT_CONTACT_PHONE> cho ứng viên.)
```

### 6 vị trí bot đang tư vấn
| Vị trí | Nơi làm | Mức |
|---|---|---|
| NV Kỹ thuật (MH POWER) | Hà Nội, Long Biên | 10–15tr |
| Thực tập sinh Kỹ thuật (MH POWER) | Hà Nội, Long Biên | từ 6tr |
| NV Kỹ thuật (MH RENTAL) | HN / Bình Dương / Long An | 10–20tr |
| NV Kinh doanh (MH RENTAL) | HN / Bình Dương / Long An | cứng 9–21tr, thu nhập 15–50tr |
| Kế toán Công nợ/Nội bộ (MH RENTAL) | Hà Nội, Nam Từ Liêm | 10–15tr |
| Kế toán Bán hàng (MH RENTAL) | Long An, Bến Lức | 10–15tr |

### Việc của HR
Nhận alert → liên hệ ứng viên, hướng dẫn nộp CV (email `<RECRUIT_CONTACT_EMAIL>`), phỏng vấn. Nếu thay đổi vị trí/mức lương → báo để cập nhật nội dung bot (sửa prompt trong MH-03).

---

## 6. Vai trò: Quản lý

### Theo dõi hàng ngày
- **Nhóm Sale (Power/Rental):** xem lead về có được gọi không.
- **Báo cáo buổi sáng (8h):** tổng lead hôm trước, tách Facebook/Google, Hot(A)/Tiềm năng(B).
- **Nhóm cảnh báo lỗi (DLQ):** nếu có `🔴 [DLQ] ...` → hệ thống gặp lỗi ghi Lark/FB/AI → báo kỹ thuật.

### Báo cáo Dashboard (CẬP NHẬT 06-08)
Hệ thống gửi **3 báo cáo** vào nhóm Lark (không còn ghi 5 bảng DB1–DB5 như trước):
- **Báo cáo sáng (8h hằng ngày):** leads hôm qua theo nguồn (Messenger/Facebook/Google, tách Power/Rental) + số khách duy nhất theo SĐT + **chi phí Meta** (Power/Rental) + **chi phí Google**.
- **Báo cáo Tuần** + **Báo cáo Tháng:** tổng leads + chi phí Meta/Google theo tuần/tháng.
> **Lưu ý thực tế:** số leads + chi phí quảng cáo là đáng tin. Pipeline chi tiết (cơ hội/báo giá/đơn/lắp đặt) chưa có vì pipeline rental cũ đã gỡ — sẽ đầy đủ khi dựng CRM rental mới. Nếu báo cáo Google hiện "không có chiến dịch" → thường do Google Ads đổi version API (báo kỹ thuật bump version).

### Việc quản lý cần quyết
- Đổi **FB token sang System User token** trước ~26/08/2026 (việc sống còn — nếu không bot có thể ngừng nhận tin mà không báo).
- Giữ **tắt Meta Instant Reply/AI Inbox** để bot n8n nhận được tin nhắn.
- Bổ sung nhân sự vào đúng nhóm Sale/HR.
- Quyết định có dựng lại pipeline rental (báo giá → đơn → hợp đồng) + bot Zalo + kênh website hay không.

---

## 7. Luồng end-to-end (ví dụ thực tế)

### Ví dụ: khách thuê boom lift qua Messenger
```
09:15  Khách comment quảng cáo "thuê xe nâng người 30m"
       → Meta gửi tin riêng → khách trả lời trong Messenger
       → Bot: "Anh cần thuê/mua xe nâng, hay cần pin lithium ạ?"
09:16  Khách: "Thuê xe nâng người 30m, 3 ngày, ở Hà Nội"
       → Bot hỏi nốt R-questions, xin tên + SĐT
09:18  Đủ thông tin → Hệ thống tạo:
         • 1. Leads liên hệ Copy (MHrental): tên, SĐT
         • 2. Lịch sử CSKH Copy: tóm tắt "thuê nâng người 30m, HN, 3 ngày"
         • Alert nhóm MHrental Sales: đầy đủ R1–R5
09:20  Sale gọi: "Anh ơi, em gọi từ MHrental về yêu cầu boom lift 30m..."
       → tư vấn, báo giá (thủ công), chốt
(nếu khách im) → bot tự nhắc lại sau 10 phút / 4 giờ / 24 giờ
```

### Ví dụ: lead pin qua form Google
```
Khách điền form Google "cần pin lithium 48V cho 3 xe nâng"
 → MH-04 chấm điểm A → ghi 3.1.0 Leads Copy (MHpower) + leads marketings
 → Alert nhóm MHpower Sales: "🔥 LEAD GOOGLE - GỌI NGAY!"
 → Sale gọi, tư vấn pin SuperV, báo giá thủ công
```

---

## 8. Lark Base — bảng dùng hàng ngày

| Bảng | Base | Marketing | Sale | Quản lý | HR |
|---|---|---|---|---|---|
| Khách hàng Copy `<TBL_PW_CUSTOMER_COPY>` | MHpower | — | ⭐ Hồ sơ KH (nối Lead/Deal/CSKH) | Xem | — |
| Khách hàng Copy `<TBL_RT_CUSTOMER_COPY>` | MHrental | — | ⭐ Hồ sơ KH (nối Lead/CSKH) | Xem | — |
| 3.1.0 Leads Copy `<TBL_PW_LEADS_COPY>` | MHpower | Xem | Gọi, cập nhật | Xem | — |
| 3.1.0 Deals Copy `<TBL_PW_DEALS_COPY>` | MHpower | — | ⭐ Nhu cầu Q1–Q14 | Xem | — |
| Lịch sử CSKH Copy `<TBL_PW_CSKH_COPY>` | MHpower | — | Nhật ký chăm sóc | — | — |
| 1. Leads liên hệ Copy `<TBL_RT_LEADS_COPY>` | MHrental | Xem | Gọi, cập nhật | Xem | — |
| 2. Lịch sử CSKH Copy `<TBL_RT_CSKH_COPY>` | MHrental | — | Nhật ký chăm sóc | — | — |
| leads marketings (2 base) | cả 2 | ⭐ Lead QC | Xem | Review kênh | — |
| DB1–DB5 (báo cáo) | MHrental | DB1 | — | ⭐ KPI | — |

> Trạng thái "Khách mới/cũ" và các field bot ghi đều dùng **tên cột tiếng Việt** — **đừng đổi/xóa tên cột "Copy"**, bot sẽ ghi lỗi ngầm. Báo kỹ thuật trước khi đổi cấu trúc bảng.

---

## 9. Hệ thống tự làm gì thay bạn

| Trước đây | Bây giờ |
|---|---|
| Nhập lead thủ công từ Facebook/Google | ✅ Tự động |
| Chatbot trả lời Messenger 24/7 | ✅ Bot AI 2 brand, hỏi đúng nhu cầu |
| Phân loại pin / xe nâng | ✅ Bot tự nhận diện, ghi đúng base |
| Tạo hồ sơ Khách hàng + nối Lead/Deal/CSKH | ✅ Tự tạo khi khách hoàn tất, không bỏ trống trường liên kết |
| Mất hồ sơ khi nhiều khách nhắn cùng lúc | ✅ Đã hết (session lưu Data Table riêng từng khách) |
| Xử lý khách hỏi tuyển dụng | ✅ Bot tư vấn vị trí + báo HR |
| Nhắc khách im | ✅ Bot tự nhắn 10 phút / 4 giờ / 24 giờ |
| Khách cũ quay lại nhu cầu mới | ✅ Tạo lead mới + báo "🔁 khách cũ" |
| Chấm điểm/phân loại lead | ✅ AI chấm A/B/C/D + nhận xét |
| Ghi nhật ký chăm sóc | ✅ Tự tạo dòng Lịch sử CSKH |
| Báo cáo tổng lead cuối ngày | ✅ Dashboard 8h sáng |

> **Hệ thống KHÔNG tự làm (sale làm thủ công):** soạn/duyệt báo giá, tạo đơn hàng, lịch lắp đặt, theo dõi bảo hành. *(Các workflow tự động hóa các bước này đã gỡ; sẽ dựng lại khi có CRM rental mới.)*

---

## 10. Xử lý sự cố thường gặp

### Bot không trả lời khách trên Messenger
- Kiểm tra **Meta Instant Reply / AI Inbox** đã TẮT chưa (nếu bật, Meta giành tin, bot không nhận).
- Kiểm n8n đang chạy: `http://<N8N_LOCAL_IP>:5678` → MH-03 active.
- Token Facebook còn hạn không (xem `MH_SYSTEM_KEYS.md` mục 8).

### Sale không nhận được thông báo
- App bot còn trong nhóm Lark không? (kick App ra → ngừng nhận).
- Lead có vào Lark không (mở bảng Leads Copy kiểm tra)?
- Có `🔴 [DLQ]` trong nhóm cảnh báo lỗi không?

### Khách điền form Google nhưng không thấy lead
- URL webhook phải kèm `?key=<MH04_WEBHOOK_KEY>`.
- Kiểm MH-04 → Executions trong n8n có lỗi không.

### Lead website không vào / báo cáo website thiếu
- ⚠️ **MH-WEBSITE chưa được migrate**: hiện ghi vào bảng *gốc* MHpower (không phải Copy), chỉ MHpower, và thông báo Sale của kênh website nhiều khả năng lỗi (đọc nhầm loại token). Nếu cần dùng kênh website, báo kỹ thuật repoint sang bảng Copy + sửa token alert.

### Dashboard chỉ thấy tổng lead, các số khác = 0
- Đúng hiện trạng: pipeline rental cũ đã gỡ, bảng Leads Copy không lưu field breakdown → chỉ tổng lead chính xác. Cần CRM rental mới để đếm cơ hội/báo giá/đơn/lắp đặt.

### Báo cáo sáng gửi nhầm/0 lead
- Có thể chạy lại thủ công: n8n → MH-Dashboard → Manual Trigger.

---

*Tài liệu kỹ thuật: `MH_SYSTEM_ARCHITECTURE.md` · Keys & ID: `MH_SYSTEM_KEYS.md` · Bàn giao: `MH_PROJECT_HANDOVER.md` · Báo cáo dự án: `BAO_CAO_DU_AN_MH.md`*
