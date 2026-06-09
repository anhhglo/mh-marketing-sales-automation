# BÁO CÁO DỰ ÁN — HỆ THỐNG TỰ ĐỘNG HÓA MARKETING & SALES MH
*Cập nhật: 2026-06-05 | Nền tảng: n8n v2.57.1 + Lark Bitable + Claude AI | Trạng thái: ĐANG VẬN HÀNH (tầng thu lead + bot 2 thương hiệu + tuyển dụng)*

> **CẬP NHẬT 2026-06-05 (chạy chính thức):**
> - Facebook Messenger đã **go-live** — App Live + `pages_messaging` thông cho khách thật, bot trả lời end-to-end qua n8n.
> - Bot **"khách cũ nhắn lại"** (giữ tên/SĐT, "như cũ" tái dùng + xác nhận) + **tuyển dụng** đã nâng cấp & kiểm thử đạt.
> - **Lưu session sang n8n Data Table** → hết lỗi mất session khi nhiều khách nhắn đồng thời; thêm cron tự dọn session cũ.
> - **4 bảng Khách hàng liên kết:** khi khách hoàn tất, hệ thống **tự tạo Khách hàng** + nối Lead/Deal/CSKH (không bỏ trống trường liên kết) → mở 1 khách thấy đủ hồ sơ.
> - Backup toàn bộ workflow: `/home/adminmh/backup/all-wf_2026-06-05_final.json`. Số liệu **Meta Ads** đã verify lấy được (dashboard chờ token Google làm chung — `MH_ADS_DASHBOARD_PLAN.md`).

> **⭐ CẬP NHẬT 2026-06-08 (mới hơn block trên):**
> - **Dashboard Quảng cáo ĐÃ XONG & CHẠY:** Google Ads Basic Access đã được duyệt (06-06) → hệ thống tự kéo **chi phí/click/CTR của cả Meta + Google** vào báo cáo (sáng hằng ngày, + báo cáo Tuần và Tháng mới). Marketing không cần nhập tay số liệu quảng cáo nữa.
> - **Báo cáo sáng nâng cấp:** đếm lead theo nguồn (Messenger/Facebook/Google) tách Power/Rental + đếm khách duy nhất theo SĐT (khách mua thêm vẫn giữ) + chi phí Meta + chi phí Google.
> - Tổng **13 workflow** (thêm 2 báo cáo Tuần/Tháng + `MH-Dedup-Prune`). Bot lõi MH-03 nâng lên **85 node** (06-09: thêm chống trùng lead 2 chiều; 06-08: ghi marketing từ Messenger).
> - **Lưu ý quản lý:** các con số/bảng workflow ở phần dưới (06-05) đã lạc hậu — đọc kèm **`MH_DANH_GIA_PRODUCTION_2026-06-08.md`** (đánh giá tổng quan + checklist trước khi đóng gói).

---

## MỤC LỤC
1. Tổng quan & mục tiêu
2. Sơ đồ luồng hoạt động
3. Các kênh thu lead
4. Bot AI tư vấn 2 thương hiệu + tuyển dụng
5. Cấu trúc CRM (Lark Bitable)
6. Thông báo Sale & HR
7. Chăm sóc khách hàng & khách cũ quay lại
8. Tính năng nâng cao
9. Danh sách workflow & hạ tầng
10. Kết quả kiểm thử
11. Vận hành & giám sát
12. Việc còn lại / hạn chế / lộ trình
13. Việc admin cần quyết / hỗ trợ

---

## 1. TỔNG QUAN & MỤC TIÊU

**Mục tiêu:** Tự động thu thập khách tiềm năng (lead) từ nhiều kênh, dùng AI sàng lọc/hỏi nhu cầu, ghi vào CRM (Lark) và báo đội Sale — không bỏ sót lead, sale có sẵn hồ sơ trước khi gọi.

**Hai thương hiệu:**
- **MHpower** — Pin Lithium SuperV & Ắc quy công nghiệp cho xe nâng.
- **MHrental** — Cho thuê & bán xe nâng (nâng hàng, nâng người, scissor, boom).

**Phạm vi đã làm:** Thu lead → bot AI hỏi nhu cầu (PIN ↔ XE NÂNG) → ghi CRM → báo Sale + chăm sóc tự động. **Mới (04/06):** bot còn nhận diện **khách hỏi tuyển dụng** → tư vấn vị trí + báo nhóm HR.

**Sau bước này:** Sale tư vấn/báo giá/chốt đơn **thủ công** (chưa tự động hóa báo giá/đơn hàng/hợp đồng — pipeline rental cũ đã gỡ).

---

## 2. SƠ ĐỒ LUỒNG HOẠT ĐỘNG

```
NGUỒN LEAD                       XỬ LÝ (n8n + AI)                       KẾT QUẢ
─────────                        ──────────────                          ───────
Comment QC ──(Meta auto tin riêng)─┐
Messenger ─────────────────────────┤   ┌──────────────────────┐   ┌─ Leads Copy (hồ sơ KH)
Form QC Facebook ──────────────────┤   │ Nhận diện brand       │──►├─ Deals Copy (nhu cầu Q1–Q14)
Form QC Google ────────────────────┼──►│ + tuyển dụng          │   ├─ Lịch sử CSKH Copy
Website ───────────────────────────┤   │ Bot AI hỏi nhu cầu    │   ├─ leads marketings
Zalo/Hotline/API ──────────────────┘   │ Chấm điểm AI (A/B/C/D)│   ├─ Nhóm Sale (Power / Rental)
                                        │ Ghi CRM + báo Sale/HR │   └─ Nhóm HR (tuyển dụng)
                                        └──────────────────────┘
```
Chạy trên **server nội bộ** (n8n Docker, WSL Ubuntu) + **Cloudflare Tunnel** nhận webhook công khai từ Meta/Google.

---

## 3. CÁC KÊNH THU LEAD

| Kênh | Cách vào | Workflow | Brand |
|------|----------|----------|-------|
| **Bình luận QC (Facebook)** | Khách comment → Meta tự gửi tin riêng → khách trả lời Messenger → bot tiếp quản | MH-03 | Bot tự hỏi |
| **Tin nhắn Messenger** | Khách nhắn trực tiếp Page | MH-03 | Bot tự nhận diện |
| **Form QC Facebook (Lead Ads)** | Điền form trên FB | MH-03 | Tự nhận từ campaign |
| **Form QC Google (Lead Form)** | Điền form Google | MH-04 | Tự nhận từ nội dung |
| **Website** | Form trên web | MH-WEBSITE | MHpower (⚠️ xem mục 12) |
| **Zalo / Hotline / API** | Webhook API có auth | MH-01 | Tự nhận / mặc định |
| **Zalo OA riêng** | (CHƯA tích hợp) | — | — |

> **Kênh comment:** Việc gửi tin nhắn riêng khi khách comment do **Meta tự xử lý** (automation từ khóa). Khi nhận webhook comment, MH-03 chỉ **mầm sẵn session** (để bot không chào trùng + có ngữ cảnh); khi khách trả lời, **bot n8n tiếp quản** (hỏi nhu cầu → tạo lead → báo sale).

> **Về số liệu quảng cáo (CẬP NHẬT 06-08):** Thu LEAD từ form Google/Facebook đã CHẠY. **Đọc số liệu hiệu suất QC** (chi phí/CTR qua Meta + Google Ads API) **NAY ĐÃ CHẠY** — Google Ads Basic Access đã duyệt 06-06; dashboard tự kéo chi phí/click/CTR mỗi sáng (+ báo cáo Tuần/Tháng). ⚠️ Google Ads REST version sunset định kỳ → khi 404 cần dò + bump version (kỹ thuật).

---

## 4. BOT AI TƯ VẤN 2 THƯƠNG HIỆU + TUYỂN DỤNG

Bot dùng **Claude AI (Haiku)** qua proxy nội bộ, xưng "em", tư vấn tự nhiên, **không bao giờ báo giá** (chuyển sale). Một bot duy nhất xử lý 3 luồng:

### 4.1. Nhận diện nhu cầu (turn đầu)
Nếu chưa rõ, bot hỏi: *"Anh đang cần thuê/mua xe nâng, hay cần pin lithium cho xe ạ?"* — rồi đi đúng bộ câu hỏi.

### 4.2. Bộ câu hỏi MHpower (PIN) — Q1→Q14
Tên → SĐT → Công ty → Q1 Số xe → Q2 Loại pin cần thay → Q3 Pin đang dùng → Q4 Người bảo dưỡng → Q5 Ảnh hưởng CV → Q6 TG chạy → Q7 TG sạc → Q8 Giờ làm/ngày → Q9 Dừng sạc giữa ca → Q10 Tải trọng → Q11 Vấn đề chính → Q12 Lý do cải thiện → Q13 Người quyết định → Q14 Dự kiến thay.

### 4.3. Bộ câu hỏi MHrental (XE NÂNG) — R1→R5
Tên → SĐT → Công ty → R1 Loại xe (nâng hàng/người/scissor/boom) → R2 Chiều cao/tải trọng → R3 Thuê bao lâu hay mua → R4 Địa điểm → R5 Khi nào cần.

### 4.4. 🆕 Tuyển dụng (HR)
Khi khách hỏi việc làm/ứng tuyển, bot **chuyển chế độ tuyển dụng** (không hỏi pin/xe nâng), tư vấn **6 vị trí** đang tuyển và thu: vị trí, tên, SĐT, khu vực, kinh nghiệm. Bot đưa liên hệ **Ms. Trang 0982 899 806 / vanphong04@tayha.net**. Khi đủ thông tin → **báo nhóm HR** (không tạo lead sale).

6 vị trí: NV Kỹ thuật (Power) · TTS Kỹ thuật (Power) · NV Kỹ thuật (Rental) · NV Kinh doanh (Rental) · Kế toán Công nợ/Nội bộ (Rental) · Kế toán Bán hàng (Rental).

### 4.5. Điều kiện "đủ hồ sơ" (tạo Deal + báo Sale)
- **MHpower:** (tên HOẶC SĐT) + Q1 + (Q2 hoặc Q3) + Q8 + Q11 + Q13 + Q14.
- **MHrental:** (tên HOẶC SĐT) + R1 + (R3 hoặc R5).
- Khách trả lời càng đầy đủ → hồ sơ Deal càng chi tiết.

### 4.6. Nguyên tắc bot (tuyệt đối không)
Không báo giá/số tiền · không bịa thông số kỹ thuật · không cam kết bảo hành/giao hàng cụ thể · không gửi STK · không chê đối thủ · không ép mua · không tự sửa SĐT của khách.

---

## 5. CẤU TRÚC CRM (LARK BITABLE)

Hai base, app Lark `<LARK_APP_ID>` ghi được cả hai.

### 5.1. Base MHpower — `ZxFDb2v4Fa3lcKssShclLMzSgLh`
| Bảng | Table ID | Bot ghi |
|------|----------|---------|
| **Khách hàng Copy** | `tblSEqXz15IKbTJE` | Tên công ty, Phone, Email, Note — **hub nối Lead/Deal/CSKH** |
| 3.1.0 Leads Copy | `tbliHIVs89nmwY0G` | Người liên hệ, SĐT, Email, Khách mới/cũ, Nguồn (Messenger/facebook/google), Ghi chú, link Khách hàng |
| 3.1.0 Deals Copy | `tblODvWhtO7d8crO` | Toàn bộ Q1–Q14, dự kiến thay, ngày tạo, link Lead, link Khách hàng |
| Lịch sử CSKH Copy | `tblegma1qJxJM6oq` | Ngày, Hành động=Chat, Khách phản hồi, Ghi chú, link Lead, link Khách hàng |
| leads marketings | `tblCuWfYzAIEMKHA` | Họ tên, SĐT, Email, Nguồn form, Chiến dịch, Ad/Form ID, Nhu cầu, AI Score, Ngày, Trạng thái |

### 5.2. Base MHrental — `RYfKboTX0arHkRspUwMlMEUkgMd`
| Bảng | Table ID | Bot ghi |
|------|----------|---------|
| **Khách hàng Copy** | `tblIn03z4HG6qnYm` | Tên công ty, Phone, Email, Note — **hub nối Lead/CSKH** |
| 1. Leads liên hệ Copy | `tbldDVBv7XtDEsen` | Người liên hệ, SĐT, Email, Khách mới/cũ, link Khách hàng |
| 2. Lịch sử CSKH Copy | `tblVxV56K2ykR0uV` | Ngày, Hành động=Chat, Khách phản hồi (nhu cầu thuê xe), link Lead, link Khách hàng |
| leads marketings | `tblgSX3899qKP4CO` | (giống marketing mhpower) |
| Hợp đồng Copy | `tblFZq2kgyubr5SE` | *(chưa đấu — bước cuối pipeline, sale tạo sau)* |
| DB1–DB5 (báo cáo) | `tbllaxpZn1tL6ZCg`... | MH-Dashboard ghi |

> **Nguyên tắc dữ liệu:** Bot chỉ điền field nó biết chắc; field giai đoạn sau (giá, model, số lượng chốt, sale phụ trách, mã hợp đồng) để trống cho Sale điền.

### 5.3. Phân luồng ghi theo brand
- Khách hỏi **PIN** → base MHpower. Khách hỏi **XE NÂNG** → base MHrental.
- Khách Messenger **hoàn tất**: tạo **Khách hàng** trước, rồi **Lead + Deals(pin)/CSKH** nối vào Khách hàng đó + báo Sale (1 lần). Với rental: Khách hàng + Lead + CSKH + báo Sale. → Mở 1 Khách hàng thấy đủ Lead/Deal/CSKH liên quan, không bỏ trống trường liên kết.

---

## 6. THÔNG BÁO SALE & HR (3 NHÓM)

| Nhóm Lark | chat_id | Nhận |
|-----------|---------|------|
| **MHpower Sales Alert** | `oc_d2b6deea979c138f3c833e6e3cc66419` | Mọi lead PIN |
| **MHrental Sales Alert** | `oc_d7dab616d1e8059353744413e231c006` | Mọi lead XE NÂNG |
| **HR / Tuyển dụng** 🆕 | `oc_285c860fcf7bc2609be825179f38b2f7` | Ứng viên qua Messenger bot |
| MH Sales Alert (bot webhook) | hook `01eb5cb4-...` | Báo cáo sáng + cảnh báo lỗi hệ thống |

**Cơ chế:** gửi qua API tin nhắn của App Lark theo chat_id. ⚠️ **App bot phải LUÔN ở trong 3 nhóm** — kick App ra thì nhóm đó ngừng nhận. Đổi tên nhóm thoải mái.

**Nội dung báo Sale:** tên, SĐT, công ty, nhu cầu tóm tắt, toàn bộ câu hỏi đã trả lời, mức gấp, nhận xét AI. **Báo HR:** vị trí ứng tuyển, tên, SĐT, khu vực, kinh nghiệm.

---

## 7. CHĂM SÓC KHÁCH HÀNG & KHÁCH CŨ QUAY LẠI

### 7.1. Nhật ký chăm sóc (Lịch sử CSKH)
Mỗi lead "chín" → tự tạo 1 dòng **Lịch sử CSKH Copy** (cả 2 base): Ngày · Hành động=Chat · Khách phản hồi (tóm tắt nhu cầu) · Ghi chú · liên kết Lead. Sale mở là thấy ngay khách đã trao đổi gì.

### 7.2. Tự động nhắc khách im (follow-up)
Khách nhắn rồi im (chưa đủ thông tin, lượt cuối là bot) → bot tự nhắn nhắc qua Messenger: **10 phút → 4 giờ → 24 giờ**. Chạy bằng cron 10 phút gắn trong MH-03. Chỉ áp dụng lead **chưa hoàn tất**.

### 7.3. Khách cũ quay lại (re-open)
Khách **đã hoàn tất hồ sơ** nhắn lại với **nhu cầu MỚI/KHÁC**: bot lưu hồ sơ cũ vào ghi chú → hỏi lại cho nhu cầu mới → khi đủ tạo **lead MỚI** + bắn **"🔁 KHÁCH CŨ NHẮN LẠI"** vào đúng nhóm Sale theo nhu cầu mới. *(Đã kiểm thử: khách thuê xe trước, nay hỏi pin → alert đúng nhóm MHpower, không lẫn.)*

---

## 8. TÍNH NĂNG NÂNG CAO

| Tính năng | Mô tả |
|-----------|-------|
| **Nhận diện brand tự động** | Bot hỏi/đoán pin hay xe nâng turn đầu, đi đúng bộ câu hỏi + ghi đúng base. |
| 🆕 **Phân luồng tuyển dụng** | Khách hỏi việc làm → bot tư vấn 6 vị trí + báo nhóm HR (không lẫn lead sale). |
| **Khách cũ nhắn lại (re-open)** | Lưu hồ sơ cũ, hỏi lại nhu cầu mới, tạo lead mới + alert "🔁 khách cũ". |
| **Chăm sóc tự động (follow-up)** | Khách im → bot tự nhắc 10 phút / 4 giờ / 24 giờ. |
| **Gộp tin rời rạc (debounce)** | Khách nhắn nhiều mảnh → bot chờ 6 giây gộp, trả lời 1 lần. |
| **Bắt SĐT chính xác** | Lấy SĐT bằng nhận dạng dãy số trong tin gốc, không để AI sửa nhầm. |
| **Chống chào trùng** | Khách từ comment vào Messenger không bị chào lại 2 lần. |
| **Chấm điểm AI** | Mỗi lead chấm A/B/C/D + nhận xét, ghi CRM. |
| **Độ bền cao** | Mọi node AI/Lark/FB tự retry 3 lần; lỗi không làm sập luồng, alert vẫn bắn. |
| **Xử lý lỗi tập trung (DLQ)** | Mọi lỗi gom về 1 workflow → cảnh báo Lark. |

---

## 9. DANH SÁCH WORKFLOW & HẠ TẦNG

### 9.1. Workflows (n8n — đang chạy)
| ID | Tên | Webhook/Trigger | Vai trò |
|----|-----|------------------|---------|
| `eZgfYcBRvfuvGl0c` | **MH-03** FB Lead + Messenger bot (**85 node**) | POST `/mh-facebook-lead` + cron 10' | Lõi: bot 2 brand + tuyển dụng, comment, follow-up, re-open, ghi marketing, **chống trùng lead 2 chiều** |
| `60X0O5OOECjwJ8LU` | **MH-Dedup-Prune** (2 node) | cron 4h | Xóa marker `mh_lead_dedup` >48h |
| `odO51CmfrBRRbnSk` | MH-03a Verify webhook FB | GET `/mh-facebook-lead` | Xác thực với Meta (token `<FB_VERIFY_TOKEN>`) |
| `z7CGM6OEw2RVlBaS` | MH-04 Google Lead Form (22 node) | POST `/mh-google-lead?key=<MH04_WEBHOOK_KEY>` | Lead Google → CRM + marketing |
| `7tXShAMcINyHnTPM` | MH-01 Lead Intake | POST `/mh-lead-intake` (headerAuth) | Lead Zalo/hotline/API |
| `VgmoTMkmnGYSlrmi` | MH-WEBSITE | POST `/mh-website` | Lead website (⚠️ chưa migrate) |
| `lpikvvTUDSRSAR1Y` | MH-Dashboard Daily (dựng lại + Ads) | cron 8h sáng | Báo cáo sáng: leads + Meta + Google → Lark |
| `LhH4KE4GkWSofusL` | **MH-Dashboard Weekly** 🆕 | schedule | Báo cáo tuần Meta+Google |
| `Uz5Re5kFOxUyuirL` | **MH-Dashboard Monthly** 🆕 | schedule | Báo cáo tháng Meta+Google |
| `welqL0ySAYm93T58` | MH-DLQ Error Handler | errorTrigger | Bắt lỗi toàn hệ thống |

Webhook công khai: `https://webhook.aituonglai.id.vn/webhook/<path>` (Cloudflare Tunnel).
**Đã gỡ:** MH-02/06/07/09/10/11 (pipeline rental cũ).

### 9.2. Hạ tầng
- **n8n v2.57.1** Docker (Docker Desktop) trong **WSL Ubuntu-22.04** trên Windows `192.168.1.122:5678`.
- **Cloudflare Tunnel** (cloudflared, systemd) → webhook ra Internet qua domain `aituonglai.id.vn`.
- **Tự khởi động khi máy bật:** n8n (`restart=unless-stopped` + Docker autostart) + Cloudflared (systemd) + task "WSL2 Auto Start". → Sau khi **đăng nhập Windows**, toàn bộ tự lên.

### 9.3. Tích hợp ngoài
- **Claude AI:** proxy `api.nkq.vn/v1/messages`, model `claude-haiku-4-5`.
- **Lark/Feishu:** App `<LARK_APP_ID>`, domain `open.larksuite.com`.
- **Facebook:** Page "MH POWER" (`105324452323085`), App `1526140942371175`, subscribe `leadgen, messages, messaging_postbacks, feed`.

> Khóa/bí mật chi tiết tại `MH_SYSTEM_KEYS.md` — KHÔNG đưa vào báo cáo chia sẻ.

---

## 10. KẾT QUẢ KIỂM THỬ

Test bằng webhook ký HMAC hợp lệ (mô phỏng khách thật), đã xác minh và dọn dữ liệu test.

| Hạng mục | Kết quả |
|----------|---------|
| Bot Messenger nhận diện brand (pin/xe nâng) | ✅ |
| Khách thật từ comment QC → Messenger → bot tạo lead xe nâng + báo sale (message_id thật) | ✅ |
| Khách trả lời FULL Q1–Q14 → Deal đủ thông tin | ✅ |
| Form Meta + Google → Leads Copy + leads marketings | ✅ |
| Khách cũ nhắn nhu cầu mới → re-open + alert đúng brand | ✅ |
| 🆕 Khách hỏi ứng tuyển → bot tư vấn vị trí + (đường HR alert đã dựng) | ✅ logic chạy; verify gửi-thật HR alert nên theo dõi lần đầu |
| SĐT bắt đúng, không bị AI sửa | ✅ |
| Gia cố: proxy AI sập → bot vẫn trả lời fallback, không crash | ✅ |
| Tách 2 nhóm Sale theo brand (kể cả reengage) | ✅ đúng nhóm, không lẫn |
| Dashboard chạy không lỗi (đếm tổng lead rental) | ✅ |

---

## 11. VẬN HÀNH & GIÁM SÁT

- **Hằng ngày:** theo dõi cảnh báo DLQ (lỗi ghi Lark/FB/AI) trong nhóm Lark; kiểm execution n8n.
- **Tuần đầu:** xem ~5 hội thoại thật, kiểm chất lượng trích xuất + record CRM đủ field.
- **Đổi nội dung bot:** sửa prompt node `Build Messenger AI Request` (MH-03) — KHÔNG đổi cấu trúc JSON output.
- **Thêm nhân viên Sale/HR:** thêm vào đúng nhóm MHpower/MHrental Sales Alert / HR → tự nhận thông báo.
- **Test nội bộ không cần khách thật:** gửi webhook ký HMAC (PSID test `6200000*`, nhớ xóa record theo record_id).

---

## 12. VIỆC CÒN LẠI / HẠN CHẾ / LỘ TRÌNH

### 🔴 Phía Meta (admin xử lý)
- **Đổi FB token → System User token** trước ~26/08/2026 (data access hết hạn → bot có thể sập ngầm). Việc sống còn #1.
- **Tắt Meta Instant Reply / AI Inbox** (giữ automation comment→tin riêng), tránh khách nhận 2 reply.

### 🟡 Kỹ thuật còn dang dở
- **MH-WEBSITE chưa migrate:** vẫn ghi bảng *gốc* MHpower (không phải Copy), MHpower-only, node alert nhiều khả năng lỗi auth token. Cần repoint + sửa nếu dùng kênh website thật.
- **Cron monitor tên cột Lark** (cảnh báo khi nhân viên đổi/xóa cột "Copy") — chưa xây.
- **Dashboard breakdown** (FB/Google/Score/pipeline) = 0 vì Leads Copy rental không lưu các field đó + bảng pipeline cũ đã xóa; chỉ "tổng lead" chính xác.

### ⚪ Theo lộ trình
- **Bot Zalo OA** — chưa tích hợp.
- **Pipeline rental** (cơ hội → báo giá → đơn → lắp đặt → bảo hành + Hợp đồng Copy) — chưa dựng lại; sau lead, sale làm thủ công.
- ~~**Đọc số liệu quảng cáo** (Google/Meta Ads API)~~ → ✅ **ĐÃ XONG 06-08** (Meta + Google live trong Daily/Weekly/Monthly). Còn lại: bảng `ads_metrics` lưu lịch sử để vẽ trend/CPL (chưa làm).

---

## 13. VIỆC ADMIN CẦN QUYẾT / HỖ TRỢ

1. **Cấp/tạo System User token Facebook** (đủ quyền, vĩnh viễn) — sống còn để hệ thống chạy bền.
2. **Xác nhận giữ tắt** mọi tự-trả-lời-tin-nhắn của Meta (chỉ giữ automation comment → tin riêng).
3. **Thêm nhân viên** vào 3 nhóm MHpower/MHrental Sales Alert + HR.
4. **Quyết định lộ trình:** có dựng lại pipeline rental (báo giá/đơn/hợp đồng) + bot Zalo không, khi nào; có dùng kênh website (cần migrate MH-WEBSITE) không.

---

## TÀI LIỆU LIÊN QUAN
- `MH_SYSTEM_ARCHITECTURE.md` — kiến trúc kỹ thuật từng workflow.
- `MH_SYSTEM_KEYS.md` — khóa/bí mật + ID (lưu riêng).
- `MH_PROJECT_HANDOVER.md` — bản bàn giao tra cứu nhanh.
- `MH_USER_GUIDE.md` — hướng dẫn vận hành theo vai trò.

---

*Kết luận: Hệ thống thu lead đa kênh + bot AI 2 thương hiệu + phân luồng tuyển dụng + ghi CRM + báo Sale/HR theo nhóm đã hoàn thiện, kiểm thử đạt, đang chạy ổn định. Việc còn lại chủ yếu nằm ở token Facebook (admin xử lý) và các phần mở rộng theo lộ trình.*
