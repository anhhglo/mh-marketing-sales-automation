# MH System — ĐÁNH GIÁ TỔNG QUAN & CHUYÊN SÂU TRƯỚC KHI ĐÓNG GÓI PRODUCTION
*Lập: 2026-06-08 | Đối chiếu TRỰC TIẾP với n8n live (192.168.1.122:5678, v2.57.1) + Lark API + filesystem*
*Người đánh giá: Claude Code — quét toàn bộ codebase, 12 workflow live, bảng Lark, thư mục dự án*

> Tài liệu này là **bản soi chiếu cuối cùng**: so sánh những gì TÀI LIỆU đang mô tả với những gì HỆ THỐNG THẬT đang chạy hôm nay, chỉ ra độ lệch, rủi ro, lỗ hổng đóng gói, và checklist trước khi bàn giao/đóng gói production. Đọc kèm: `MH_SYSTEM_ARCHITECTURE.md`, `MH_SYSTEM_KEYS.md`, `BAO_CAO_DU_AN_MH.md`, `MH_PROJECT_HANDOVER.md`, `MH_USER_GUIDE.md`.

> ⭐ **CẬP NHẬT 2026-06-09 (sau khi lập bản này):** Chống trùng lead nâng lên **2 chiều** (bidirectional, fail-open) → **MH-03 nay 85 node** (mọi chỗ ghi "73 node" bên dưới là số 06-08, đã cũ); thêm workflow **`MH-Dedup-Prune`** cron 4AM → **tổng 13 workflow**; codebase `mh-system/workflows/all-wf.json` đã re-export (sanitize + bỏ staticData). Chi tiết: `MH_CHANGELOG_2026-06-09.md`.

---

## 0. TÓM TẮT ĐIỀU HÀNH

**Trạng thái: ĐANG CHẠY ỔN ĐỊNH, sẵn sàng production ở mức SME.** Hệ thống thu lead đa kênh + bot AI 2 thương hiệu + tuyển dụng + CRM Lark + dashboard quảng cáo đã hoạt động end-to-end và **không có execution lỗi nào kể từ 2026-06-05** (kiểm tra tới 2026-06-08 07:50).

| Hạng mục | Điểm | Ghi chú |
|---|---|---|
| Kiến trúc & độ bền (retry/onError/DLQ) | 9/10 | Phòng thủ tốt, fail không gãy chain |
| Sức khỏe runtime (error rate) | 9/10 | 0 lỗi 3 ngày gần nhất; cron 10' đều đặn |
| Tính năng (bot/CRM/ads dashboard) | 9/10 | Nhiều hơn cả tài liệu mô tả |
| **Độ chính xác tài liệu** | **6/10** | ⚠️ Tài liệu LẠC HẬU ~3 ngày so với live (xem §1) |
| **Bảo mật / sẵn sàng đóng gói** | **5/10** | ⚠️ Secrets hardcode khắp nơi; script đóng gói có lỗ hổng (xem §5) |
| Khả năng bàn giao / vận hành | 8/10 | Bộ tài liệu vai-trò đầy đủ, cần đồng bộ lại |
| **TỔNG THỂ** | **7.7/10** | **Production-ready về kỹ thuật; cần đồng bộ tài liệu + vá đóng gói trước khi bàn giao** |

**3 việc PHẢI làm trước khi đóng gói (chi tiết §8–9):**
1. 🔴 **Đồng bộ tài liệu với live** — tài liệu đang mô tả hệ thống của 06-05; live đã khác (Dashboard Ads, MH-03 73 node, 2 workflow mới).
2. 🔴 **Vá `_build_package.py`** — sanitizer bỏ sót Lark Secret + Claude key + Google Ads creds; và đang đóng gói `all-wf.json` bản CŨ (2026-05-31).
3. 🔴 **Export workflow MỚI** — backup hiện tại (06-05) đã cũ so với live 06-08.

---

## 1. ⚠️ ĐỘ LỆCH TÀI LIỆU ↔ HỆ THỐNG LIVE (quan trọng nhất)

Toàn bộ tài liệu MH_* được cập nhật lần cuối **2026-06-05**. Nhưng n8n cho thấy có thay đổi lớn **2026-06-06 → 2026-06-08** chưa được ghi vào bất kỳ tài liệu nào. Đây là điều phải biết trước khi đóng gói:

| # | Tài liệu đang nói | Thực tế LIVE (2026-06-08) | Bằng chứng |
|---|---|---|---|
| 1 | **Dashboard Ads "HOÃN — chờ token Google"** (`MH_ADS_DASHBOARD_PLAN.md`) | **ĐÃ DỰNG & CHẠY LIVE.** Daily Summary kéo Meta Ads + Google Ads thật mỗi sáng | Daily Summary có node `Get Meta Ads Daily`, `Get Google Ads Token`, `Get Google Ads Daily` — chạy OAuth refresh token Google |
| 2 | Chỉ có **1 workflow Dashboard** (Daily 8h) | **CÓ 3:** Daily (`lpikvvT...`) + **Weekly** (`LhH4KE4GkWSofusL`, tạo 06-06) + **Monthly** (`Uz5Re5kFOxUyuirL`, tạo 06-06) — đều active, đều có Meta+Google Ads | `n8n_list_workflows` trả 12 workflow (tài liệu ghi 10) |
| 3 | **MH-03 = 70 node** | **MH-03 = 73 node.** Thêm `AI Retry Guard` (chặn giữa Claude Messenger AI → Parse) + `Build Marketing Msg` + `Lark Create Marketing Msg` (Messenger hoàn tất giờ CŨNG ghi `leads marketings`) | `n8n_get_workflow` structure: nodeCount=73, cập nhật 2026-06-08T04:32 |
| 4 | **Daily Summary = 12 node**, ghi 5 bảng DB1–DB5, query 5 bảng 404 | **Daily Summary = 10 node, ĐÃ DỰNG LẠI.** KHÔNG còn ghi DB1–DB5, KHÔNG còn query bảng 404; thay bằng: Query Leads Power + Rental + Meta Ads + Google Ads → báo cáo gộp | `n8n_get_workflow` full: code `Build Morning Summary` mới hoàn toàn |
| 5 | **Google Ads "đang chờ Basic Access"** (`MH_SYSTEM_KEYS.md §7`) | **ĐÃ ĐƯỢC DUYỆT.** Node Google Ads gọi `customers/1778238164/googleAds:search` với dev token + refresh token thật, đang chạy hằng ngày | Memory `project_google_ads_basic_access` (verify 06-06) + node live trong 3 dashboard |
| 6 | Báo cáo sáng "breakdown brand = 0, chỉ tổng lead chính xác" | Báo cáo sáng giờ **đếm lead theo nguồn** (Messenger/FB/Google) tách Power/Rental, **dedup khách theo SĐT**, + chi phí Meta (Power/Rental) + chi phí Google | code `Build Morning Summary` mới |

**Hệ quả:** Nếu đóng gói NGAY bằng tài liệu + backup hiện có, người nhận sẽ đọc một hệ thống KHÁC với hệ thống đang chạy. **Phải đồng bộ tài liệu §1 này vào các file MH_* trước khi đóng gói** (đặc biệt `MH_SYSTEM_ARCHITECTURE.md` mục 8, `MH_ADS_DASHBOARD_PLAN.md`, và bảng workflow ở mọi file).

> File backup `backup/*_BEFORE_2026-06-08.js` (ParseMessengerAI, LoadSessionState, BuildMorningSummary) xác nhận: 3 file code này đã bị sửa ngày 06-08.

---

## 2. BẢN ĐỒ HỆ THỐNG THỰC TẾ (live 2026-06-08)

### 2.1. Workflows — 12 cái (11 active + 1 active phụ trợ; 2 cũ tắt)

| ID | Tên | Active | Node (live) | Trigger | So với tài liệu |
|----|-----|:--:|:--:|---|---|
| `eZgfYcBRvfuvGl0c` | **MH-03 FB Lead + Messenger bot** (lõi) | 🟢 | **73** | POST `/mh-facebook-lead` + cron 10' | tài liệu ghi 70 → **+3** |
| `odO51CmfrBRRbnSk` | MH-03a FB Webhook Verify | 🟢 | 3 | GET `/mh-facebook-lead` | khớp |
| `z7CGM6OEw2RVlBaS` | MH-04 Google Lead Form | 🟢 | 22 | POST `/mh-google-lead?key=<MH04_WEBHOOK_KEY>` | khớp |
| `7tXShAMcINyHnTPM` | MH-01 Lead Intake & AI Score | 🟢 | 12 | POST `/mh-lead-intake` (headerAuth) | khớp |
| `VgmoTMkmnGYSlrmi` | MH-WEBSITE | 🟢 | 8 | POST `/mh-website` | khớp — ⚠️ vẫn chưa migrate |
| `lpikvvTUDSRSAR1Y` | **MH-Dashboard Daily Summary** | 🟢 | **10** | cron `0 8 * * *` + manual | **DỰNG LẠI** (tài liệu ghi 12, ghi DB1–5) |
| `LhH4KE4GkWSofusL` | **MH-Dashboard Weekly Report** 🆕 | 🟢 | 10 | schedule + manual | **KHÔNG có trong tài liệu** |
| `Uz5Re5kFOxUyuirL` | **MH-Dashboard Monthly Report** 🆕 | 🟢 | 10 | schedule + manual | **KHÔNG có trong tài liệu** |
| `welqL0ySAYm93T58` | MH-DLQ Error Handler | 🟢 | 5 | errorTrigger toàn hệ thống | khớp |
| `YOQlvkjGYmbRXtWt` | MH-Session-Prune | 🟢 | 2 | cron 3h (xóa session >30 ngày) | khớp |
| `p5NeVE8nSTA8Kd3g` | MH-BOT (Messenger cũ) | 🔴 | 13 | — | tắt |
| `r74nySmGhRorFnBR` | MH-BOT-VERIFY (cũ) | 🔴 | 3 | — | tắt |

### 2.2. Dashboard Ads — đã LIVE (Meta + Google), điểm sáng mới

Cả 3 dashboard (Daily/Weekly/Monthly) chung 1 khung: `Get Lark Token → Query Leads Power → Query Leads Rental → Get Meta Ads → Get Google Ads Token → Get Google Ads → Build Summary → Send (Lark bot webhook 01eb5cb4-…)`.

- **Meta Ads:** `GET graph.facebook.com/v21.0/act_915544230489694/insights` (`level=campaign`, `date_preset=yesterday|last_7d|last_30d`), token **System User "never-expire"** (`EAAVsBGCZCSWcBRoOey…` — khác token messaging) → **bền, không hết hạn cứng**.
- **Google Ads:** OAuth refresh token → `googleads.googleapis.com/v21/customers/1778238164/googleAds:search`, `login-customer-id=7248299313`, dev token `bwCKeaL…`. GAQL lấy cost_micros/clicks/impressions/ctr theo campaign.
- **Báo cáo sáng** giờ gộp: lead theo nguồn (tách Power/Rental) + dedup khách theo SĐT + chi phí Meta (phân Power/Rental theo keyword campaign) + chi phí Google. Mọi node `onError: continueRegularOutput` → 1 nguồn lỗi không làm hỏng cả báo cáo.

> ⚠️ Khác với `MH_ADS_DASHBOARD_PLAN.md` đề xuất: **KHÔNG tạo bảng Lark `ads_metrics`** — số liệu ads chỉ gửi dạng text vào nhóm Lark, không lưu lịch sử để vẽ trend. Nếu cần phân tích theo thời gian, đây là việc mở rộng đáng làm.

### 2.3. Bảng Lark (đối chiếu live)

**Base MHpower `ZxFDb2v4…` — 39 bảng thật.** Bot ghi vào các bảng **"Copy" (staging)**, KHÔNG đụng CRM vận hành thật:
- Bảng Copy bot ghi: `3.1.0 Leads Copy` (rev 113), `Khách hàng Copy` (rev 33), `3.1.0 Deals Copy` (rev 64), `Lịch sử CSKH Copy` (rev 35), `leads marketings` (rev 21) → revision THẤP = ít dữ liệu, đúng vai trò staging.
- Bảng vận hành THẬT (bot KHÔNG ghi): `3.1.0 Leads` (rev 1806), `3.1.0 Deals` (rev 12198), `Khách hàng` (rev 2358), kho/sản xuất/đơn hàng/đánh-seri (rev tới 11524), công việc nhân viên, doanh số… → **đây là base vận hành sống của công ty**.
- **Đánh giá:** Tách Copy/thật là **thiết kế an toàn tốt** — bot không bao giờ làm bẩn dữ liệu thật. Khi go-live đủ tin cậy, có thể quyết định trỏ thẳng bảng thật (cân nhắc kỹ).
- Tàn dư cần dọn: `Bản sao Khách hàng` (`tblWHI9OR…`), `MHBot Leads` (`tblgKevSnOQNrpIh`, rev 1) — không workflow nào dùng.

**Base MHrental `RYfKboTX…`:** Leads liên hệ Copy `tbldDVBv7XtDEsen`, CSKH Copy `tblVxV56K2ykR0uV`, Khách hàng Copy `tblIn03z4HG6qnYm`, leads marketings `tblgSX3899qKP4CO`, Hợp đồng Copy (chưa đấu), DB1–DB5 (KHÔNG còn được Daily Summary ghi sau khi dựng lại — xem §6).

### 2.4. n8n Data Table (session bot — không phải Lark)
- `mh_sessions` (`PG8zfQkCDMylmrWa`): 1 dòng/PSID, chống clobber. ĐÃ LIVE.
- `mh_buffers` (`G4Tmb5YbzC6TY3e4`): dự phòng, hiện chưa dùng (buffer vẫn staticData).

### 2.5. Thư mục & tài sản dự án
| Thư mục/file | Nội dung | Vai trò khi đóng gói |
|---|---|---|
| `MH_*.md` (11 file) + `botmrketlark.md` | Bộ tài liệu chính | **Cần đồng bộ §1 trước khi gói** |
| `all-wf.json` (gốc, 178KB) | Export workflow **2026-05-31** — CŨ | ⚠️ `_build_package.py` đang gói file này → STALE |
| `backup/all-wf_2026-06-05_final.json` (381KB) | Export 06-05 (10 wf) | Mới hơn nhưng vẫn thiếu thay đổi 06-08 |
| `backup/*_BEFORE_2026-06-08.js` | Snapshot code trước sửa 06-08 | Bằng chứng thay đổi; có thể dùng rollback |
| `_build_package.py` / `_scan_secrets.py` | Công cụ đóng gói + quét secret | **Có lỗ hổng — xem §5** |
| `filemdchatbot-old/` (7 file MD) | Knowledge base bot Zalo cũ: system prompt, quy định, phong cách, **skill bán hàng (42KB)**, **kiến thức SuperV**, **bảng giá**, FAQ/cadence | Nguồn nội dung nâng cấp bot (Sprint 2+) — chưa nhúng hết vào bot Messenger |
| `chatbotlark/` (4 file) | Hệ thống RIÊNG — trợ lý Lark nội bộ "Minh 🦞" | Xem §7 |
| `jd my company/` (6 PDF) | JD 6 vị trí tuyển dụng | Khớp 6 vị trí prompt HR của bot |
| `migration/` | Bộ kit di chuyển stack sang host mới (`mh-stack-*.tar.zst` 531MB, setup.sh, README) | Hữu ích cho deploy production sang máy khác |
| `*.sh`, `*.py` (refresh_and_subscribe, subscribe_page, get_pages…) | Script vận hành Meta/tunnel | Tiện ích, chứa secret → cần sanitize khi gói |

---

## 3. ĐÁNH GIÁ KIẾN TRÚC & CHẤT LƯỢNG (điểm mạnh)

1. **Độ bền production tốt.** Mọi node AI/Lark/FB có `retryOnFail` 3× + `onError: continueRegularOutput`. Proxy AI/Lark chập chờn → tự retry → vẫn lỗi thì KHÔNG gãy chain, alert sale vẫn bắn (alert dựng từ session, không phụ thuộc Lark/AI). Có thêm `AI Retry Guard` (mới 06-08) gia cố nhánh Messenger.
2. **Xử lý lỗi tập trung (DLQ).** `errorWorkflow` gán toàn hệ thống → mọi lỗi gom về MH-DLQ → phân loại TRANSIENT/PERMANENT → cảnh báo Lark. Vận hành có thể giám sát 1 nơi.
3. **Chống mất session (Data Table).** Đã chuyển session sang `mh_sessions` (1 dòng/PSID) → hết clobber khi nhiều khách nhắn đồng thời — đã PoC chứng minh (3 PSID đồng thời đều lưu). Đây là fix đúng cho vấn đề gốc của staticData.
4. **Thiết kế an toàn dữ liệu.** Bot ghi bảng "Copy" staging, không đụng CRM thật; quy tắc xóa/sửa Lark CHỈ theo `record_id` (sau sự cố xóa nhầm 4 lead). Phone field type 13 chỉ gán khi có giá trị. DuplexLink dùng mảng phẳng.
5. **Bot 3-trong-1 mạch lạc.** 1 webhook chung phân luồng leadgen/message/comment; bot tự nhận brand (PIN Q1–Q14 / XE NÂNG R1–R5) + tuyển dụng; debounce gộp tin; re-open khách cũ; follow-up 10'/4h/24h; bắt SĐT bằng regex trên tin gốc (không tin số AI sinh).
6. **4 bảng liên kết qua Khách hàng Copy làm hub** — mở 1 khách thấy đủ Lead/Deal/CSKH, không bỏ trống trường liên kết.
7. **Token ads bền.** Dashboard dùng System User token never-expire cho Meta → không chết token hằng ngày.

---

## 4. SỨC KHỎE PRODUCTION (kiểm tra execution live)

- **0 execution lỗi** trong toàn hệ thống kể từ **2026-06-05 01:56** (lỗi gần nhất) đến 2026-06-08 07:50. Các lỗi cũ đều ở giai đoạn test/tinh chỉnh (05-27 → 06-05).
- **MH-03 follow-up cron** chạy đều mỗi 10 phút, mỗi lần ~30–80ms, **status success** liên tục (exec 4635→4649). Webhook xử lý tin thật (exec 4637) success.
- n8n health: **v2.57.1, up-to-date**, response ~221ms.
- Tự khởi động khi bật máy: n8n (Docker `unless-stopped`) + Cloudflared (systemd) + WSL auto-start.

**Kết luận runtime:** ổn định, không có dấu hiệu hồi quy sau thay đổi 06-08.

---

## 5. 🔐 BẢO MẬT — QUAN TRỌNG NHẤT KHI ĐÓNG GÓI

### 5.1. Secrets đang hardcode ở đâu
Toàn bộ khóa nằm **plaintext** trong: (a) node n8n (URL/header/body), (b) `MH_SYSTEM_KEYS.md`, (c) nhiều `*.sh`/`*.py` gốc. Gồm: FB App Secret, FB Page token + System User token, Lark App Secret, Claude proxy key, Cloudflare API token + tunnel secret, **Google Ads dev token + client_secret + refresh_token** (mới, trong node dashboard).

→ Rủi ro: rotate 1 khóa = sửa nhiều nơi; lộ 1 file = lộ toàn bộ. Khuyến nghị dài hạn: chuyển sang **n8n Credentials store** thay vì hardcode trong node.

### 5.2. ⚠️ Lỗ hổng trong `_build_package.py` (script đóng gói "secret-free")
Script chỉ redact theo LITERALS + REGEX cố định. **Bỏ sót các secret KHÔNG khớp pattern** → sẽ LỌT vào package "sạch":
- **Lark App Secret** `tu7mll…` (32 ký tự alphanumeric, KHÔNG phải hex, không có `=`) → không regex nào bắt → **LỌT**.
- **Claude proxy key** `AGOP-5681-…` → không pattern → **LỌT**.
- **Google Ads dev token** `bwCKeaL-…`, **client_secret** `GOCSPX-…`, **refresh_token** `1//04…` → không pattern → **LỌT**.
- Lark App ID bị thay `<LARK_APP_ID>` nhưng App **Secret** thì không.

→ **PHẢI bổ sung các literal/regex này vào `LITERALS`/`REGEXES` của `_build_package.py`** (và vào LEAK scan ở cuối) trước khi tin tưởng output. Chạy lại `_scan_secrets.py` trên thư mục `~/mh-system` sau khi build để xác nhận 0 leak.

### 5.3. ⚠️ `_build_package.py` đóng gói `all-wf.json` BẢN CŨ
Script đọc `~/all-wf.json` = export **2026-05-31** → thiếu toàn bộ: migration Copy, Khách hàng hub, Session Data Table, Ads Dashboard, Weekly/Monthly, các fix 06-08. Package sẽ chứa workflow KHÁC với live.
→ **Export lại 12 workflow hiện tại** (n8n → Download all, hoặc API) ghi đè `all-wf.json` trước khi build; đồng thời thêm các DOCS mới vào danh sách `DOCS` (đang thiếu file đánh giá này + bất kỳ doc mới nào).

### 5.4. Lưu ý khi chia sẻ
- **KHÔNG đưa `MH_SYSTEM_KEYS.md` vào package công khai** (script đã chủ động loại — giữ nguyên).
- Tài liệu đánh giá này (`MH_DANH_GIA_PRODUCTION_2026-06-08.md`) **cố tình không in giá trị secret**, chỉ nêu loại/vị trí → an toàn để đưa vào package nội bộ; nhưng vẫn nên rà trước khi public.

---

## 6. RỦI RO & NỢ KỸ THUẬT (xếp hạng)

| Mức | Vấn đề | Tác động | Hành động |
|---|---|---|---|
| 🔴 P0 | **FB Page token messaging** `data_access ~đầu 9/2026` (token phái sinh user) | Sau mốc, bot có thể **im ngầm không báo** | Đổi sang **System User page token** (đã có token nguồn never-expire — chính token dashboard đang dùng cho ads); hoặc đặt lịch refresh. Sống còn #1 |
| 🔴 P0 | **Tài liệu lạc hậu vs live** (§1) | Bàn giao sai hiện trạng | Đồng bộ tài liệu trước khi gói |
| 🔴 P0 | **Lỗ hổng đóng gói** (§5.2/5.3) | Lộ secret + gói workflow cũ | Vá script + export mới |
| 🟠 P1 | **Secrets hardcode** trong node + file | Rotate khó, bề mặt lộ rộng | Dồn về n8n Credentials; rà file `.sh/.py` gốc |
| 🟠 P1 | **Phụ thuộc tên cột Lark tiếng Việt** | Nhân viên đổi/xóa cột "Copy" → fail ngầm | Chưa có cron monitor — nên xây |
| 🟠 P1 | **Google Ads refresh token / OAuth** | Token thu hồi → dashboard Google = 0 (đã onError, không crash) | Theo dõi; có cảnh báo khi rỗng |
| 🟡 P2 | **MH-WEBSITE chưa migrate** | Ghi bảng gốc MHpower, alert đọc nhầm `app_access_token` (token là `tenant_access_token`) → alert web lỗi auth | Repoint Copy + sửa token nếu dùng kênh web |
| 🟡 P2 | **DB1–DB5 mồ côi** | Daily Summary dựng lại KHÔNG còn ghi 5 bảng DB này; bảng vẫn tồn tại nhưng đứng yên | Quyết định: bỏ hẳn hay đấu lại |
| 🟡 P2 | **Node orphan 404 trong MH-04** (Cơ hội/Nhu cầu KT) | Không chạy (đã ngắt input) nhưng còn rác | Dọn cho sạch workflow |
| 🟡 P2 | **Ads metrics không lưu lịch sử** | Không vẽ được trend chi phí/CPL | Tạo bảng `ads_metrics` nếu cần phân tích |
| ⚪ P3 | **botzalo chưa xây**, **pipeline rental chưa dựng lại**, **buffer/comment-seed vẫn staticData** | Theo lộ trình | Mở rộng khi cần |

---

## 7. THÀNH PHẦN LIÊN QUAN (không thuộc lõi thu-lead nhưng cùng dự án)

### 7.1. `chatbotlark/` — Trợ lý Lark nội bộ "Minh 🦞" (hệ thống RIÊNG)
Một bot **khác** với bot thu lead: trợ lý Lark cho **4 admin nội bộ**, chạy trên **Hermes + OpenViking** (websocket persistent), RAG bằng **Jina AI `jina-embeddings-v3` dim=1024** (≈185 vectors), recall ~430ms–1s, đọc schema 2 base (Power 32 + Rental 22 = 54 bảng), cron sync 07:00, có fallback Opus-4-7. Báo cáo tại `chatbotlark/BAO-CAO-HE-THONG.md` (cập nhật 2026-05-25 — cũng cần xem có còn đúng không). Liên quan memory `project_openviking_embedding`.
> Nếu "đóng gói dự án" bao gồm cả trợ lý này, cần đánh giá riêng (config OpenViking, state.db 40MB chat history, backup 4 lớp). Nếu chỉ gói hệ thống thu-lead MH-0x thì để riêng.

### 7.2. `migration/` — kit di chuyển host
`mh-stack-*.tar.zst` (531MB) + `setup.sh` + `migrate-restore.sh` + README → dựng lại toàn stack trên máy mới. Hữu ích nếu production = chuyển sang server riêng (khỏi phụ thuộc WSL trên máy cá nhân).

### 7.3. `filemdchatbot-old/` — knowledge base chưa khai thác hết
7 file MD chất lượng (skill bán hàng 42KB, kiến thức SuperV, bảng giá, FAQ/cadence). Bot Messenger hiện **chưa nhúng** phần lớn nội dung này (mới ở Sprint 1 theo `MH_BOTMESSENGER_UPGRADE_PLAN.md`). Đây là tài sản để nâng chất bot (Sprint 2: `MHBot_Products`/`MHBot_Config`).

---

## 8. ✅ CHECKLIST ĐÓNG GÓI PRODUCTION

> **ĐÃ LÀM trong phiên 2026-06-08:** ✅ Đồng bộ 6 tài liệu MH_*.md với live (block "CẬP NHẬT 2026-06-08" + bảng workflow + trạng thái Ads/Google). ✅ Vá `_build_package.py` (redact thêm Lark Secret/Claude key/Google creds + auto-discover docs + tự nhặt export mới nhất + cảnh báo stale) — chạy thử **0 leak**. ✅ Export tươi 12 workflow → `backup/all-wf_2026-06-08_final.json` (mốc rollback). **CÒN LẠI:** mục C (Meta-side, admin) + mục D (dọn dẹp tùy chọn).

**A. Đồng bộ tài liệu (P0)** — ✅ XONG
- [ ] Cập nhật `MH_SYSTEM_ARCHITECTURE.md` §8 (Dashboard) theo bản dựng lại + thêm Weekly/Monthly + Ads Meta/Google.
- [ ] Cập nhật bảng workflow (10→12, MH-03 70→73) ở MỌI file: ARCHITECTURE, HANDOVER, BAO_CAO, USER_GUIDE, SYSTEM_KEYS.
- [ ] Đổi `MH_ADS_DASHBOARD_PLAN.md` từ "HOÃN" → "ĐÃ TRIỂN KHAI" (mô tả thực tế 3 dashboard).
- [ ] `MH_SYSTEM_KEYS.md §7`: Google Ads "chờ duyệt" → "ĐÃ DUYỆT + đang chạy"; ghi rõ login-customer-id 7248299313, customer 1778238164, dev token, vị trí OAuth.
- [ ] Ghi rõ MH-03 dùng staticData cho buffer/comment, Data Table cho session.

**B. Vá đóng gói & secrets (P0)**
- [ ] Export 12 workflow live → ghi đè `all-wf.json` (hoặc trỏ script sang export mới).
- [ ] Thêm vào `_build_package.py`: redact Lark App Secret, Claude key, Google dev token/client_secret/refresh_token; bổ sung chúng vào LEAK scan.
- [ ] Thêm doc mới (gồm file này) vào danh sách `DOCS`.
- [ ] Build → chạy `_scan_secrets.py` trên `~/mh-system` → xác nhận **0 leak**.
- [ ] Xác nhận `MH_SYSTEM_KEYS.md` KHÔNG nằm trong package public.

**C. Việc Meta-side (P0/P1 — admin)**
- [ ] Đổi FB messaging token sang System User page token (trước ~đầu 9/2026).
- [ ] Xác nhận đã **tắt Meta Instant Reply / AI Inbox** (giữ automation comment→tin riêng).
- [ ] App bot Lark vẫn ở trong 3 nhóm Sale/HR.

**D. Dọn dẹp (P2)**
- [ ] Quyết định DB1–DB5 (bỏ hay đấu lại); dọn node orphan 404 MH-04; dọn bảng `MHBot Leads`/`Bản sao Khách hàng`.
- [ ] Migrate MH-WEBSITE hoặc tắt nếu không dùng.

**E. Backup chốt**
- [ ] Tạo `backup/all-wf_2026-06-08_final.json` (12 wf) làm mốc rollback production.

---

## 9. KHUYẾN NGHỊ ƯU TIÊN

**P0 (làm trước khi đóng gói):** Đồng bộ tài liệu §1 → Export workflow mới → Vá `_build_package.py` (secret + all-wf) → Đổi/đặt lịch FB messaging token.

**P1 (tuần đầu production):** Dồn secrets về n8n Credentials → Cron monitor tên cột Lark → Cảnh báo khi Meta/Google ads trả rỗng → Theo dõi 5 hội thoại thật + báo cáo sáng.

**P2 (lộ trình):** Bảng `ads_metrics` lưu lịch sử → Nhúng knowledge base `filemdchatbot-old` vào bot (Sprint 2) → Migrate/tắt MH-WEBSITE → Dựng lại pipeline rental nếu cần → botzalo.

---

## 10. KẾT LUẬN

Về **kỹ thuật**, hệ thống **đã production-ready ở quy mô SME**: chạy ổn định, 0 lỗi 3 ngày gần nhất, phòng thủ lỗi tốt, tính năng vượt cả tài liệu (Dashboard Ads Meta+Google đã live, Weekly/Monthly mới). Đây là một hệ tự động hóa marketing-sales hoàn chỉnh và đáng tin.

Về **đóng gói/bàn giao**, còn 2 khoảng hở phải bịt: (1) **tài liệu trễ ~3 ngày so với live** — người nhận sẽ hiểu sai hiện trạng; (2) **script đóng gói có lỗ hổng** — sẽ lọt secret và gói workflow cũ. Cả hai đều sửa nhanh (vài giờ). Cộng với việc **đổi FB messaging token** (rủi ro sống còn theo thời gian), đó là 3 việc P0 cần xong trước khi coi là "đóng gói production".

> Sau khi hoàn tất checklist §8, hệ thống đủ điều kiện bàn giao production an toàn.

---
*Tài liệu này lập tự động bằng đối chiếu trực tiếp n8n live + Lark API + filesystem ngày 2026-06-08. Mọi node/ID/trạng thái lấy từ hệ thống thật, không suy đoán.*
