# MH System — Bàn giao dự án (Handover)
*Chốt: 2026-06-05 | n8n v2.57.1 — 192.168.1.122:5678 | Kèm: MH_SYSTEM_ARCHITECTURE.md, MH_SYSTEM_KEYS.md, BAO_CAO_DU_AN_MH.md, MH_USER_GUIDE.md, MH_SESSION_DATATABLE_PLAN.md*

> **CẬP NHẬT 2026-06-05 (chạy chính thức):** (1) **FB messaging GO-LIVE** — App Live, bot gửi được cho khách thật (token PAGE admin trong n8n, refresh trước ~đầu 9/2026). (2) **Re-engage "khách cũ" + tuyển dụng** nâng cấp & test đạt. (3) **Session → n8n Data Table `mh_sessions`** (hết clobber) + workflow **MH-Session-Prune** (cron 3h xóa >30 ngày). (4) **4 bảng Khách hàng liên kết** — Messenger hoàn tất → tạo Khách hàng Copy + nối Lead/Deal/CSKH (MH-03 = 70 node). (5) **Meta Ads** đã verify lấy được số liệu — dashboard chờ token Google làm chung (`MH_ADS_DASHBOARD_PLAN.md`). (6) Backup: `/home/adminmh/backup/all-wf_2026-06-05_final.json` (10 wf).

> **⭐ CẬP NHẬT 2026-06-08 (mới hơn cả block trên):** (1) **Dashboard Ads Meta + Google ĐÃ LIVE** — 3 workflow: Daily (dựng lại), **Weekly** (`LhH4KE4GkWSofusL`), **Monthly** (`Uz5Re5kFOxUyuirL`). Google Ads Basic Access đã duyệt (06-06). (2) **MH-03 = 73 node** (thêm AI Retry Guard + ghi marketing từ Messenger). (3) Tổng **12 workflow** (không phải 10). (4) Daily Summary KHÔNG còn ghi DB1–DB5/query bảng 404. (5) Đánh giá production + checklist đóng gói: **`MH_DANH_GIA_PRODUCTION_2026-06-08.md`**. ⚠️ Backup 06-05 + tài liệu bên dưới đã LẠC HẬU — đọc kèm file đánh giá.

---

## 1. Hệ thống làm gì
Thu lead đa kênh → **bot AI 2 thương hiệu** (MHpower pin lithium / MHrental thuê-mua xe nâng) + **phân luồng tuyển dụng** → ghi CRM Lark → báo Sale/HR. Sau khi lead vào Lark, **sale làm thủ công** (báo giá/đơn/hợp đồng chưa tự động hóa).

```
Comment QC ─(Meta keyword → tin riêng)─┐
Messenger ─────────────────────────────┤
FB Lead Form ──┐                        ├─→ MH-03 bot ─→ Lark (Leads/Deals/CSKH Copy) + Alert Sale
Google Lead Form ─┼─→ MH-04             │            + leads marketings + Alert HR (tuyển dụng)
Zalo/Hotline/API ─┴─→ MH-01             │
Website ───────────→ MH-WEBSITE ────────┘
```

## 2. Workflows (n8n) — 8 ACTIVE + 2 tắt
| ID | Tên | Webhook/Trigger | Ghi chú |
|---|---|---|---|
| `eZgfYcBRvfuvGl0c` | **MH-03 FB Lead + Messenger bot (85 node)** | POST `/mh-facebook-lead` + cron 10' | Lõi: bot 2 brand + tuyển dụng, comment-seed, follow-up, re-open, **tạo Khách hàng**, **session Data Table**, **ghi marketing từ Messenger**, **chống trùng lead 2 chiều** |
| `60X0O5OOECjwJ8LU` | **MH-Dedup-Prune** 🆕 (2) | cron 4h | Xóa marker `mh_lead_dedup` >48h |
| `odO51CmfrBRRbnSk` | MH-03a FB Webhook Verify | GET `/mh-facebook-lead` | Verify token `<FB_VERIFY_TOKEN>` |
| `z7CGM6OEw2RVlBaS` | MH-04 Google Lead Form (22) | POST `/mh-google-lead?key=<MH04_WEBHOOK_KEY>` | → Leads Copy + marketing |
| `7tXShAMcINyHnTPM` | MH-01 Lead Intake (12) | POST `/mh-lead-intake` (headerAuth) | → Leads Copy brand-aware, alert score A |
| `VgmoTMkmnGYSlrmi` | MH-WEBSITE (8) | POST `/mh-website` | ⚠️ chưa migrate (bảng gốc) |
| `lpikvvTUDSRSAR1Y` | MH-Dashboard Daily (10, dựng lại + Ads) | cron 8h | Báo cáo sáng: leads + Meta + Google Ads |
| `LhH4KE4GkWSofusL` | **MH-Dashboard Weekly** 🆕 (10) | schedule | Báo cáo tuần Meta+Google |
| `Uz5Re5kFOxUyuirL` | **MH-Dashboard Monthly** 🆕 (10) | schedule | Báo cáo tháng Meta+Google |
| `welqL0ySAYm93T58` | MH-DLQ Error Handler (5) | errorTrigger | Bắt lỗi toàn hệ thống → alert |
| `YOQlvkjGYmbRXtWt` | MH-Session-Prune (2) | cron 3h | Xóa session `mh_sessions` >30 ngày |
| `p5NeVE8nSTA8Kd3g` / `r74nySmGhRorFnBR` | MH-BOT / MH-BOT-VERIFY (cũ) | — | **TẮT** (đã thay bằng MH-03/03a) |

**Đã XÓA:** MH-02/06/07/09/10/11 (pipeline rental cũ). Backup 15 workflow: `/home/adminmh/all-wf.json`.
Webhook public: `https://webhook.aituonglai.id.vn/webhook/<path>` (Cloudflare Tunnel).

## 3. Bảng Lark đích (đã đấu)
**MHpower `ZxFDb2v4Fa3lcKssShclLMzSgLh`:**
- **Khách hàng Copy `tblSEqXz15IKbTJE`** (hub) · 3.1.0 Leads Copy `tbliHIVs89nmwY0G` · Deals Copy `tblODvWhtO7d8crO` · Lịch sử CSKH Copy `tblegma1qJxJM6oq` · **leads marketings `tblCuWfYzAIEMKHA`**
- *(MH-WEBSITE/MH-BOT cũ còn ghi bảng gốc Leads `tblGsiuuVf5H3QwF`, Deals `tblkevEQFeBdcQut`)*

**MHrental `RYfKboTX0arHkRspUwMlMEUkgMd`:**
- **Khách hàng Copy `tblIn03z4HG6qnYm`** (hub) · 1. Leads liên hệ Copy `tbldDVBv7XtDEsen` · 2. Lịch sử CSKH Copy `tblVxV56K2ykR0uV` · **leads marketings `tblgSX3899qKP4CO`** · Hợp đồng Copy `tblFZq2kgyubr5SE` (chưa đấu)

> **4 bảng Copy liên kết qua Khách hàng Copy (hub):** Messenger hoàn tất → bot tạo Khách hàng Copy trước → nối Lead/Deal/CSKH bằng DuplexLink. Khách hàng Copy có schema = bảng Khách hàng thật (mhpower `tblZVtsai7CSh3vF`, mhrental `tblPJspWzjxWVpam`). FB Lead Form hiện chỉ tạo Lead. Session bot lưu ở **n8n Data Table `mh_sessions`** (không còn staticData).
- DB1–DB5 báo cáo: `tbllaxpZn1tL6ZCg / tblIPiWOEMgzrPfM / tblkAbgyd63auOgC / tblEW4PAju29iOKs / tblVyUieVKjHpICj`

Lark App: `<LARK_APP_ID>` (ghi cả 2 base). Nhóm Sale: Power `oc_d2b6deea979c138f3c833e6e3cc66419`, Rental `oc_d7dab616d1e8059353744413e231c006`, **HR `oc_285c860fcf7bc2609be825179f38b2f7`**. Báo cáo/lỗi: bot webhook `01eb5cb4-...`.

## 4. ✅ Đã verify chạy thật
- Messenger bot **nhận biết brand** (pin Q1–Q14 / xe nâng R1–R5) → tạo Lead + Deals/CSKH Copy + alert đúng nhóm.
- **Comment QC → mầm session → khách nhắn Messenger → bot tiếp quản** (không chào trùng).
- **Khách cũ nhắn nhu cầu mới** → re-open + alert "🔁 khách cũ" + tạo lead mới.
- 🆕 **Tuyển dụng:** khách hỏi việc làm → bot tư vấn 6 vị trí + đưa liên hệ Ms. Trang → bắn HR alert.
- **SĐT** bắt bằng regex trên tin gốc. Form Meta + Google → Leads Copy + leads marketings.
- MH-01 intake, MH-Dashboard, follow-up tự động — đều chạy. Gia cố retry/onError trên 25 node.

## 5. 🔴 Việc CÒN LẠI của bạn (Meta-side)
1. **TẮT Meta tự trả lời TIN NHẮN Messenger** (AI Inbox / Instant Reply / automation message-keyword). Chỉ giữ automation **comment → tin riêng**. Nếu không, Meta giành tin, bot n8n không nhận.
2. **FB Token**: token PAGE hiện hợp lệ nhưng `data_access_expires_at ≈ 26/08/2026` (token phái sinh user) + thiếu `pages_read_user_content`. → Tạo **System User token** (data access vĩnh viễn) có `pages_read_user_content, pages_messaging, pages_read_engagement, pages_manage_engagement, pages_manage_metadata, leads_retrieval` → thay vào MH-03 (3 node: Fetch Facebook Lead, Send Messenger Reply, Send Followup Alert). Khi đó có thể chuyển private-reply comment về n8n.

## 6. 🟡 Phạm vi chưa làm
- **botzalo**: chưa xây.
- **MH-WEBSITE**: chưa migrate sang Copy (ghi Leads/Deals gốc MHpower; node Send Sales Alert đọc nhầm `app_access_token` trong khi token là `tenant_access_token` → alert website nhiều khả năng lỗi). Sửa nếu dùng kênh web.
- **Pipeline rental** (cơ hội/báo giá/đơn/lắp đặt/bảo hành + Hợp đồng Copy): bảng cũ đã xóa, chưa dựng lại → Dashboard chỉ đếm tổng lead rental, pipeline = 0.
- **Cron monitor tên cột Lark**: chưa xây (rủi ro fail ngầm khi đổi/xóa cột Copy).
- **Đọc số liệu QC** (Google Ads API): chờ duyệt Basic Access.

## 7. Vận hành nhanh
- **Giám sát**: theo dõi DLQ alert (Lark) + execution lỗi trong n8n. Tuần đầu kiểm 5 hội thoại thật.
- **Test bot không cần Meta**: gửi webhook ký HMAC (PSID `6200000*`, xóa record theo record_id sau test).
- **Đổi câu/nội dung bot**: sửa node `Build Messenger AI Request` (prompt) trong MH-03; KHÔNG đổi JSON output schema. Khối tuyển dụng + 6 vị trí cũng nằm trong prompt này.
- **Comment**: node `Seed Comment Session` chỉ mầm session; private-reply hiện do Meta automation (bật `Send Private Reply` khi có quyền `pages_read_user_content`).

## 8. Lưu ý an toàn dữ liệu
Xóa/ghi đè Lark CHỈ dùng `record_id` đã xác nhận — KHÔNG filter wildcard theo tên (đã từng xóa nhầm 4 lead thật, khôi phục từ thùng rác Lark). Session bot lưu trong staticData n8n — **đừng re-import workflow ẩu** (mất session); giới hạn 200 session.

## 9. Lịch sử thay đổi
Chi tiết: `MH_CHANGELOG_2026-06-02.md`. Mốc chính: tạo bảng marketing → đổi sang bảng Copy → brand routing 2 phía (FB form + Messenger) → MH-01/04/Dashboard repoint Copy → MHpower/rental CSKH Copy → gia cố retry/onError → fix khách-cũ + SĐT + ngoài-phạm-vi → comment-handler → **(04/06) thêm nhánh tuyển dụng + HR alert**.
