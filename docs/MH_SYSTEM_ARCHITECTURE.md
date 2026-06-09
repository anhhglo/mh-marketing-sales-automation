# MH Power / MH Rental — Kiến trúc hệ thống (kỹ thuật)
*Cập nhật: 2026-06-05 | n8n v2.57.1 | Server 192.168.1.122:5678 | Đối chiếu trực tiếp với workflow live*

> Tài liệu này mô tả **đúng những gì đang chạy trong n8n** (node, kết nối, bảng đích, điều kiện). Mọi ID/field lấy từ export workflow thật. Khóa/bí mật để riêng tại `MH_SYSTEM_KEYS.md`.

## ⭐ CẬP NHẬT 2026-06-09 (MỚI NHẤT — ưu tiên cao nhất)
> Chống trùng lead form-echo Messenger ⟷ leadgen nâng lên **2 chiều (bidirectional)** + housekeeping. Validate 0 lỗi.
- **MH-03 = 85 node** (từ 73): thêm 7 node dedup.
- **Chống double 2 chiều** — Data Table `mh_lead_dedup` (`5i9nFlrWv4ybFGGV`), khóa = SĐT 9 số cuối (`replace(/[^0-9]/g,'').replace(/^84/,'').replace(/^0+/,'').slice(-9)`):
  - *Forward* (leadgen về trước — thường gặp): leadgen ghi marker khóa `core` (`Build LG Dedup Key`→`DT Mark Leadgen`, sớm trước AI); Messenger khi hoàn tất `IF Session Complete`→`DT Check Dedup`→`Eval Dedup`→`IF Is Dup` (true=bỏ tạo lead, false=tạo).
  - *Reverse* (form-echo về trước — hiếm): Messenger non-dup ghi marker khóa **`core:msg`** (`Build Msg Dedup Key`→`DT Mark Messenger`); leadgen check trước khi tạo `Validate AI Schema`→`DT Check Msg Dedup`→`Eval Leadgen Dedup`→`IF Leadgen Dup` (true=bỏ, false=tạo). **Fail-open**: lỗi/empty → vẫn tạo lead leadgen (không bao giờ nuốt lead).
  - ⚠️ Khóa messenger dùng hậu tố `:msg` TÁCH khỏi leadgen `core` để `DT Mark Leadgen` (upsert theo dkey) không ghi đè marker reverse — bug đã phát hiện & sửa khi test (exec 4851).
- **Cờ session đúng:** nhánh trùng ghi `lark_created=false, dedup_skipped=true` (`Mark Dedup Skipped`→`DT Upsert Session Skipped`).
- **Marker leadgen bền hơn:** `DT Mark Leadgen` maxTries 5 + alwaysOutputData.
- **Workflow MỚI `MH-Dedup-Prune` (`60X0O5OOECjwJ8LU`, active, cron 4AM):** xóa marker `mh_lead_dedup` >48h. Tổng workflow live = **13**.
- Test thật pass: reverse skip (exec 4852), forward + cờ session (exec 4854), fail-open create (exec 4851).

## ⭐ CẬP NHẬT 2026-06-05
- **Messaging go-live:** App Facebook đã Live + `pages_messaging` thông cho **khách thật**; bot trả lời end-to-end (token PAGE admin trong 3 node MH-03; `data_access ~đầu 9/2026` → cần refresh).
- **Re-engage "khách cũ" nâng cấp** (xem 3.3.3): dedup `previous_inquiry`; luôn giữ tên/SĐT/công ty; "như cũ" → tái dùng chi tiết + xác nhận, chỉ hỏi phần mới; ép **chỉ tiếng Việt**. Test đạt (exec 2338); nhánh **tuyển dụng** test đạt (exec 2331).
- **Hạn chế (mục 11):** `staticData` chỉ persist khi execution KẾT THÚC → 2 khách nhắn chồng nhau (~20–45s) có thể **clobber session**. Kế hoạch khắc phục (chuyển sang Data Table): **`MH_SESSION_DATATABLE_PLAN.md`** (chờ duyệt).
- **Session → Data Table (HOÀN TẤT):** MH-03 lưu session ở Data Table `mh_sessions` (`PG8zfQkCDMylmrWa`) thay staticData → **hết clobber giữa các khách**. Load đọc DT-first (fallback staticData), Save dual-write + DT Upsert; cron follow-up cũng đọc/ghi DT. Thêm workflow **MH-Session-Prune** (`YOQlvkjGYmbRXtWt`, cron 3h) xóa session >30 ngày. Buffer/comment-seed vẫn staticData (rủi ro thấp). Chi tiết: `MH_SESSION_DATATABLE_PLAN.md`.
- **4-bảng Khách hàng liên kết (HOÀN TẤT):** khi khách hoàn tất, bot tạo record **Khách hàng Copy** trước rồi gắn link vào Lead+Deal+CSKH → 4 bảng Copy liên kết qua Khách hàng Copy làm hub (xem 3.3.4 + 10.1). MH-03 = **70 node**.
- **Backup rollback:** `/home/adminmh/backup/all-wf_2026-06-05_final.json` (10 workflow, sau migration + Khách hàng).

## ⭐ CẬP NHẬT 2026-06-08 (đối chiếu live — phần này MỚI hơn toàn bộ tài liệu bên dưới)
> Hệ thống đã đổi sau 06-05. Khi đọc các mục dưới, ưu tiên thông tin ở đây. Đánh giá đầy đủ: `MH_DANH_GIA_PRODUCTION_2026-06-08.md`.
- **MH-03 = 73 node** (không phải 70): thêm `AI Retry Guard` (giữa Claude Messenger AI → Parse), `Build Marketing Msg` + `Lark Create Marketing Msg` → **Messenger hoàn tất giờ CŨNG ghi `leads marketings`** (nhánh từ `Get Lark Token Msg`). *(→ nay 85 node sau fix dedup 06-09, xem mục trên.)*
- **Dashboard Ads Meta + Google ĐÃ LIVE** trong **3 workflow** (xem §8 đã viết lại): Daily `lpikvvTUDSRSAR1Y` (10 node, dựng lại), **Weekly `LhH4KE4GkWSofusL`** (mới), **Monthly `Uz5Re5kFOxUyuirL`** (mới). Tổng workflow live = **12** (không phải 10).
- **Google Ads Basic Access ĐÃ DUYỆT (06-06)** → kéo cost/click/CTR thật từ customer `1778238164`. ⚠️ REST version sunset định kỳ → khi 404 phải bump (06-08: v18→v21).
- **Daily Summary dựng lại:** KHÔNG còn ghi DB1–DB5, KHÔNG còn query 5 bảng 404; thay bằng đếm lead theo nguồn (Power/Rental) + dedup khách theo SĐT + chi phí Meta + Google.
- **Lead Rental có field `Nguồn`** (mới tạo `fldYZEYH6q`): MH-03 gắn `facebook`/`Messenger`. Chống-trùng lead: **CHỦ ĐÍCH không chặn** (khách cũ mua thêm = lead thật); báo cáo sáng chỉ hiển thị số khách duy nhất theo SĐT.

---

## 1. Tổng quan

```
NGUỒN LEAD                         XỬ LÝ (n8n + Claude AI)                ĐÍCH (Lark CRM + Sale/HR)
──────────                         ───────────────────────                ─────────────────────────
Comment QC Facebook ─(Meta auto tin riêng)─┐
Messenger (chat)  ─────────────────────────┤
FB Lead Form ──────────┐                    ├─► MH-03  ── brand routing ──► Lark Leads/Deals/CSKH Copy
Google Lead Form ──────┼─► MH-04            │      (PIN ↔ XE NÂNG ↔ Tuyển dụng)   + leads marketings
Zalo/Hotline/API ──────┴─► MH-01            │                                    + Nhóm Sale (Power/Rental)
Website form ──────────────► MH-WEBSITE ────┘                                    + Nhóm HR (tuyển dụng)
                                   │
                                   ▼
                         MH-Dashboard (8h sáng)  → Lark báo cáo + 5 bảng DB
                         MH-DLQ (errorTrigger)   → cảnh báo lỗi toàn hệ thống
```

- Hạ tầng: n8n Docker (WSL Ubuntu) + Cloudflare Tunnel (webhook public).
- Phạm vi: **thu lead đa kênh → bot AI sàng lọc Q&A → ghi CRM → báo Sale/HR**. Sau lead, sale làm thủ công (báo giá/đơn/hợp đồng CHƯA tự động hóa).
- AI: Claude Haiku qua proxy `api.nkq.vn`. Mọi node AI/Lark/FB có `retryOnFail` 3× + `onError: continueRegularOutput` (gia cố 02/06) → proxy/Lark chập chờn không gãy chain, alert sale vẫn bắn.

---

## 2. Danh sách workflow (live)

| ID | Tên | Active | Nodes | Trigger |
|----|-----|--------|-------|---------|
| `eZgfYcBRvfuvGl0c` | **MH-03 Facebook Lead Ads** (lõi) | ✅ | **85** | webhook POST `/mh-facebook-lead` + cron 10' follow-up (2 trigger) |
| `odO51CmfrBRRbnSk` | MH-03a FB Webhook Verify | ✅ | 3 | webhook GET `/mh-facebook-lead` |
| `z7CGM6OEw2RVlBaS` | MH-04 Google Ads Lead Form | ✅ | 22 | webhook POST `/mh-google-lead` |
| `7tXShAMcINyHnTPM` | MH-01 Lead Intake & AI Score | ✅ | 12 | webhook POST `/mh-lead-intake` (headerAuth) |
| `VgmoTMkmnGYSlrmi` | MH-WEBSITE | ✅ | 8 | webhook POST `/mh-website` |
| `lpikvvTUDSRSAR1Y` | MH-Dashboard Daily Summary (dựng lại + Ads) | ✅ | 10 | cron `0 8 * * *` + manual |
| `LhH4KE4GkWSofusL` | **MH-Dashboard Weekly Report** 🆕 (Meta+Google) | ✅ | 10 | schedule + manual |
| `Uz5Re5kFOxUyuirL` | **MH-Dashboard Monthly Report** 🆕 (Meta+Google) | ✅ | 10 | schedule + manual |
| `welqL0ySAYm93T58` | MH-DLQ Error Handler | ✅ | 5 | errorTrigger |
| `YOQlvkjGYmbRXtWt` | MH-Session-Prune | ✅ | 2 | cron 3h — xóa session `mh_sessions` >30 ngày |
| `60X0O5OOECjwJ8LU` | **MH-Dedup-Prune** 🆕 | ✅ | 2 | cron 4h — xóa marker `mh_lead_dedup` >48h |
| `p5NeVE8nSTA8Kd3g` | MH-BOT (Messenger cũ) | ❌ | 13 | — |
| `r74nySmGhRorFnBR` | MH-BOT-VERIFY (cũ) | ❌ | 3 | — |

> **Đã xóa hẳn:** MH-02, MH-06, MH-07, MH-09, MH-10, MH-11 (pipeline rental cũ). Backup: `/home/adminmh/all-wf.json`.

---

## 3. MH-03 — Facebook Lead Ads + Messenger Bot (85 node, 2 trigger)

Đây là lõi hệ thống. **1 webhook chung** nhận mọi sự kiện Page (leadgen / message / comment), 1 cron follow-up riêng.

### 3.1. Phân luồng sự kiện (sau verify)

```
Webhook FB Lead POST → Respond 200 OK (EVENT_RECEIVED) → Verify FB Signature (HMAC-SHA256)
   → Extract FB leadgen_id   (phân loại event_type: leadgen | message | comment)
        ├──► Dedup Leadgen ─► IF Has Lead ID ─[true: leadgen]─► (NHÁNH A: FB LEAD FORM)
        │                                     └[false]──────────► IF Is Messenger ─[message]─► (NHÁNH B: BOT CHAT)
        └──► IF Is Comment ─[comment]─► Dedup Comment ─► Seed Comment Session   (NHÁNH C: COMMENT)
```

`Extract FB leadgen_id` (code) phân tích `body.entry[0]`:
- `messaging[].message.text` → `event_type='message'`; ảnh → `[Khách gửi hình ảnh]`; postback `GET_STARTED/MENU_PRICING/MENU_PRODUCT` → map sang text tự nhiên, vẫn `event_type='message'`.
- `changes[].value.item='comment'` (verb add, không phải Page tự cmt) → `event_type='comment'`, lấy `commenter_id`, `comment_text`.
- `changes[].value.leadgen_id` → `event_type='leadgen'`.

`Verify FB Signature`: HMAC-SHA256 rawBody với App Secret `534cf2...`, so `x-hub-signature-256`. Sai → throw (vào DLQ).

### 3.2. NHÁNH A — FB Lead Form (khách điền form quảng cáo)

```
IF Has Lead ID(true) → Get Lark Token → Fetch Facebook Lead (Graph v21.0, fields=field_data,campaign...)
  → Normalize Facebook Lead ─┬─► Build AI Request → Claude AI Score → Parse AI Score → Validate AI Schema
                             └─► Build LG Dedup Key → DT Mark Leadgen (marker khóa `core`, dedup forward — ghi SỚM trước AI)
  Validate AI Schema → DT Check Msg Dedup → Eval Leadgen Dedup → IF Leadgen Dup
       ├─[trùng form-echo <6h]─► (DỪNG, KHÔNG tạo lead; fail-open: lỗi/empty vẫn tạo)
       └─[không trùng]────────► IF Brand MHrental ─[true]─► Build Lead MHRental FB ─► Lark Create Lead MHRental (mhrental tbldDVBv7XtDEsen)
                                                   └[false]► Build Lead MHPower FB  ─► Lark Create Lead MHPower FB (mhpower tbliHIVs89nmwY0G)
  (cả 2 nhánh) ─► Build Marketing FB ─► Lark Create Marketing FB (leads marketings theo brand)
                 ─► Build Sale Alert Message ─► Lark Alert Sale Team (chat brand-routed)
```

- **Brand** detect trong `Normalize Facebook Lead` từ text campaign/adset/ad: chứa `xe nang/mhrental/aerial/boom` → MHrental; còn lại → MHpower.
- **Test injection:** `leadgen_id` bắt đầu `test_fb_` → bỏ qua Graph API, sinh mock data (`test_fb_mhrental_*` / `test_fb_mhpower_*`).
- `Build Marketing FB`: ghi bảng **leads marketings** (`Nguồn form='Meta Lead Ads'`, campaign, Ad/Form ID, Nhu cầu, AI Score) — power `tblCuWfYzAIEMKHA` / rental `tblgSX3899qKP4CO`.
- Alert chọn chat theo brand: rental `oc_d7dab616...`, power `oc_d2b6deea...`.

### 3.3. NHÁNH B — Messenger Bot (chat, brand-aware Q&A)

```
IF Is Messenger(message) → Buffer Message → Wait Debounce (6s) → Load Session State
  → Build Messenger AI Request → Claude Messenger AI → Parse Messenger AI Response → Save Session State
  → Send Messenger Reply (FB Graph /me/messages)
       ├─► IF Session Complete (newly_completed=true) ─► (ghi CRM — xem 3.3.3)
       ├─► IF Reengage Alert (reengaged=true)         ─► Get Lark Token Reengage → Build/Send Reengage Alert
       └─► IF Is Recruitment (recruitment_alert=true) ─► Get Lark Token HR → Build/Send HR Alert
```

#### 3.3.1. Debounce-and-merge (gộp tin rời rạc)
- `Buffer Message`: đẩy mảnh tin vào `staticData.buffers[psid]`, đóng dấu `token` (duy nhất/execution).
- `Wait Debounce`: chờ 6s (webhook đã Respond 200 từ đầu nên không timeout Meta).
- `Load Session State`: nếu `buffer_ts[psid] !== token` → có mảnh mới hơn → `return []` (drop execution cũ). Chỉ mảnh **cuối** gộp toàn bộ `buffers[psid]` thành 1 `msg_text`, xóa buffer → gọi AI 1 lần.
- Cũng tại đây: auto-cleanup PSID test (`test_`, `99999999`); migrate session cấu trúc cũ; reset session toàn bộ reply = fallback; **reset session incomplete > 30 ngày** (B2).

#### 3.3.2. Bot AI — system prompt 3-trong-1 (`Build Messenger AI Request`)
System prompt (~ field `system` riêng) gồm:
- **Bước 1 — xác định nhu cầu:** nếu chưa rõ brand → hỏi "anh cần thuê/mua xe nâng, hay cần pin lithium?". Tự nhận diện từ khóa → điền `brand` ngay.
- **Bộ PIN (MHpower) Q1–Q14:** số xe → loại pin → pin đang dùng → bảo dưỡng → ảnh hưởng CV → TG chạy → TG sạc → giờ làm → dừng sạc → tải trọng → vấn đề chính → lý do cải thiện → người quyết định → dự kiến thay.
- **Bộ XE NÂNG (MHrental) R1–R5:** loại xe → chiều cao/tải → thuê bao lâu/mua → địa điểm → khi nào cần.
- **Khối TUYỂN DỤNG** (`recruitBlock`): khi khách hỏi việc làm → `intent='tuyendung'`, brand rỗng, không hỏi nhu cầu pin/xe; tư vấn 6 vị trí, thu `tuyen_vi_tri/name/phone/tuyen_dia_diem/tuyen_kinh_nghiem`; đưa liên hệ **Ms. Trang 0982 899 806 / vanphong04@tayha.net**.
- Quy tắc: xưng "em/anh-chị", 1–2 câu hỏi/lượt, **không báo giá**, không bịa spec, không sửa SĐT khách. Output **JSON thuần** với `reply/intent/extracted{...}/is_complete/new_inquiry/tuyen_complete`.
- Model `claude-haiku-4-5`, `max_tokens 1500`, history 8 tin gần nhất.

#### 3.3.3. `Parse Messenger AI Response` — logic chốt
- Extract `content.find(type==='text')`; nếu JSON parse fail → salvage regex `"reply":"..."`, fallback "Bạn có thể nói rõ hơn...".
- **SĐT (P2):** lấy bằng regex VN mobile từ tin gốc khách, KHÔNG tin chữ số AI sinh.
- **Brand normalize:** rental/xe nâng/thuê → MHrental; power/pin/lithium → MHpower.
- **Re-open (P1/P3):** nếu session đã `is_complete` mà khách nêu nhu cầu MỚI (`new_inquiry=true`) → snapshot Q/R cũ vào `previous_inquiry`, xóa Q/R, reset `is_complete=false`, `reengaged=true`.
- **Completion theo brand:**
  - MHrental: (name HOẶC phone) + `r1_loai_xe` + (`r3` HOẶC `r5`).
  - MHpower: `q1` + (`q2` HOẶC `q3`) + `q8` + `q11` + (name HOẶC phone) + `q13` + `q14`.
- **Tuyển dụng:** `is_recruitment=true` khi `intent='tuyendung'` hoặc có field `tuyen_*`. KHÔNG bao giờ tạo lead sale; `recruitment_alert=true` (1 lần) khi có vị trí + (tên/SĐT), set `hr_alerted`.
- `newly_completed = is_complete && !wasComplete` (trừ khi vừa reengage). Khi newly_completed → reply chốt "Dạ em đã ghi nhận đầy đủ... Sale [brand] sẽ liên hệ..." + `lark_created=true`.
- `Save Session State`: lưu `staticData.sessions[psid]`, prune giữ tối đa 200 session (xóa cũ nhất theo `updated_at`).

#### 3.3.4. Nhánh hoàn tất → ghi CRM (brand-routed, có tạo Khách hàng)
```
IF Session Complete(true) → DT Check Dedup → Eval Dedup → IF Is Dup
  ├─[trùng leadgen <6h]─► Mark Dedup Skipped → DT Upsert Session Skipped  (lark_created=false; DỪNG, KHÔNG tạo lead/marketing/CSKH)
  └─[không trùng]──────► Get Lark Token Msg → IF Brand Msg
        (song song: Build Msg Dedup Key → DT Mark Messenger [marker `core:msg` cho dedup reverse]; Build Marketing Msg → leads marketings)
  ├─[MHrental]─► Build Customer Rental ─► Lark Create Customer Rental (mhrental tblIn03z4HG6qnYm)
  │             ─► Build Lead Rental Msg ─► Lark Create Lead Rental (mhrental tbldDVBv7XtDEsen)
  │             ─► Build CSKH Rental Msg ─► Lark Create CSKH Rental (mhrental tblVxV56K2ykR0uV)
  │             ─► Build Alert Rental Msg ─► Lark Alert Rental Msg (chat rental)
  └─[MHpower]──► Build Customer Power ─► Lark Create Customer Power (mhpower tblSEqXz15IKbTJE)
                ─► Build Lead from Messenger ─► Lark Create Lead Messenger (mhpower tbliHIVs89nmwY0G)
                ─► Build NKT from Messenger ─► Lark Create NKT Messenger (mhpower tblODvWhtO7d8crO, Q1–Q14 + checkbox)
                ─► Build CSKH Power Msg ─► Lark Create CSKH Power Msg (mhpower tblegma1qJxJM6oq)
                ─► Build Alert Messenger Sale ─► Lark Alert Messenger Sale (chat power)
```
- **Khách hàng tạo ĐẦU TIÊN** (Tên công ty / Phone / Email / Note); `record_id` của nó gắn vào Lead/Deal/CSKH qua trường link → Lark tự sinh link 2 chiều. Trường link: mhpower Lead `Khách hàng Copy-3.1.0 Leads Copy-Customer`, Deal `Khách hàng Copy-3.1.0 Deals Copy-Customer`, CSKH `Khách hàng Copy-Lịch sử CSKH Copy-Khách hàng`; mhrental Lead `Khách hàng Copy-1. Leads liên hệ Copy-Customer`, CSKH `Khách hàng Copy-2. Lịch sử CSKH Copy-Khách hàng`. Các trường nghiệp vụ sau (MST/Mã CRM/công nợ/sale) để trống cho sale-kế toán.
- CSKH record link tới Lead (DuplexLink) qua field `...-Leads` / `...-2. Lịch sử CSKH`.
- NKT (Deals Copy) ghi: `15. Tổng hợp...` (toàn bộ Q1–Q14), `14. Dự kiến...` (`< 30 ngày`/`30-90 ngày`/`Chưa biết`), checkbox Q1–Q13 (chỉ ghi khi rõ yes/no), `Ngày tạo KH` (ms), `Phone`, link Lead.

#### 3.3.5. Re-open alert & HR alert (song song sau Send Reply)
- **Reengage:** `🔁 KHÁCH CŨ NHẮN LẠI — CÓ NHU CẦU MỚI` → nhóm Sale theo brand mới (kèm `previous_inquiry`). Lead mới chỉ tạo khi đủ thông tin (qua nhánh complete).
- **HR:** `🧑‍💼 ỨNG VIÊN MỚI QUA MESSENGER` (vị trí, tên, SĐT, khu vực, kinh nghiệm) → nhóm HR `oc_285c860fcf7bc2609be825179f38b2f7`.

### 3.4. NHÁNH C — Comment seeding
```
IF Is Comment → Dedup Comment (TTL 7 ngày, theo comment_id) → Seed Comment Session
```
- `Seed Comment Session`: tạo/đệm session cho `commenter_id` với 1 câu chào bot trong history. **KHÔNG gửi private reply** (việc gửi tin riêng khi comment vẫn do Meta automation lo). Mục đích: khi khách nhắn Messenger sau đó, bot không chào trùng + có context.

### 3.5. MH-05 Follow-up (cron 10' gắn trong MH-03)
```
Followup Cron (10') → Scan Silent Sessions → Send Followup Alert (FB Graph → khách)
```
- `Scan Silent Sessions`: quét `staticData.sessions`; chọn session `is_complete=false`, có history, **lượt cuối là BOT** (khách im). Tính tuổi theo `updated_at`, chọn mốc cao nhất đã vượt mà chưa nhắc: **s10m → s4h → s24h**; đánh dấu `followup_done[stage]`, push câu nhắc vào history, KHÔNG đổi `updated_at`.
- `Send Followup Alert`: **gửi thẳng cho khách** qua Messenger (`onError continue` + retry → 1 PSID lỗi không chặn batch).
- Hạn chế đã biết: "mark-before-send" — đánh dấu trước khi gửi thành công → FB lỗi tạm có thể mất 1 lần nhắc (retry giảm thiểu).

### 3.6. Session schema (staticData.global.sessions[psid])
```js
{ psid, info:{ brand, name, phone, company,
    q1_so_xe..q14_du_kien_thay,            // PIN
    r1_loai_xe..r5_khi_nao_can,            // XE NÂNG
    tuyen_vi_tri, tuyen_dia_diem, tuyen_kinh_nghiem, is_recruitment, hr_alerted,
    previous_inquiry },                     // re-open snapshot
  history:[{role:'user'|'bot',text}],       // max 20, dùng 8 gần nhất
  turn_count, created_at, updated_at, is_complete, brand,
  lark_created, followup_done:{s10m,s4h,s24h} }
```
Ngoài ra staticData còn: `processed_leadgen_ids` (dedup 24h), `processed_comment_ids` (dedup 7 ngày), `buffers`/`buffer_ts` (debounce).

---

## 4. MH-03a — FB Webhook Verify (3 node)
`FB Verify GET → IF Valid Verify Token (== '<FB_VERIFY_TOKEN>') → Respond Challenge (echo hub.challenge)`.
GET và POST dùng chung path `/mh-facebook-lead`; n8n tách theo method. Token phải khớp khai báo Meta.

---

## 5. MH-04 — Google Ads Lead Form (22 node)

```
Webhook POST /mh-google-lead → Respond OK → Verify Google Key (?key=<MH04_WEBHOOK_KEY>)
  → Normalize Google Lead → Dedup Lead (lead_id, TTL 24h) → Get Lark Token
  → Build AI Request → Claude AI Score → Parse AI Score → Validate AI Schema
  → Build Lark Record (brand-aware) → Lark Base - Create Lead
       (rental tbldDVBv7XtDEsen / power tbliHIVs89nmwY0G)
  → Build Marketing GG → Lark Create Marketing GG (leads marketings theo brand, Nguồn form='Google Lead Form')
  → Build Sale Alert Message → Lark Alert Sale Team (chat brand-routed)
```
- AI prompt chấm cả 2 brand (không loại D chỉ vì hỏi pin thay vì thuê xe). `Build Lark Record` còn tự detect brand lần nữa từ text.
- **Node orphan (404, đã ngắt input):** `IF Score A or B → Build Cơ hội → Lark Create Cơ hội (tblVd7LARtr35eGi) → Build Nhu cầu KT → Lark Create Nhu cầu KT (tbl6B9enQModpDWp) → IF Score A Only`. Các node này còn trong workflow nhưng **không có kết nối vào** → không chạy (di sản pipeline rental cũ). 2 node Lark này KHÔNG có onError nhưng vô hại vì không bao giờ thực thi.

---

## 6. MH-01 — Lead Intake & AI Score (12 node)

```
Webhook POST /mh-lead-intake (headerAuth: cred "MH Webhook Auth Token") → Get Lark Token
  → Normalize Lead Data (nhận FB Lead Ads raw HOẶC JSON tổng quát; tự detect source + brand)
  → Build AI Request → Claude AI Score → Parse AI Score → Validate AI Schema
  → Build Lark Record (brand-aware) → Lark Base - Create Lead
       (rental tbldDVBv7XtDEsen / power tbliHIVs89nmwY0G)
  → IF Score A - Urgent ─[A]─► Build Sale Alert Message → Lark Alert Sale Team (chat brand-routed)
                        └[B/C/D]─ (dừng, không alert)
```
- Kênh nhập tổng quát cho Zalo/Hotline/API. **Chỉ alert khi score A** (khác MH-03/04 alert mọi score). KHÔNG ghi leads marketings.

---

## 7. MH-WEBSITE (8 node) — ⚠️ chưa migrate

```
Webhook POST /mh-website → Prepare Scoring Prompt → Call Claude AI → Parse Score & Build Records
  → Get Lark Token (tenant_access_token) → Create Lead Record → Create Deal Record → Send Sales Alert
```
- Ghi **bảng GỐC MHpower** (KHÔNG phải Copy): Leads `tblGsiuuVf5H3QwF`, Deals `tblkevEQFeBdcQut`. Dùng **field ID** (`fldDD8vdG3`...), MHpower-only, không brand routing, không marketing.
- **Bug đã biết:** node `Send Sales Alert` đọc `$('Get Lark Token').first().json.app_access_token` nhưng token node trả `tenant_access_token` → Authorization = "Bearer undefined" → alert website nhiều khả năng **lỗi auth**. (Lead/Deal dùng `tenant_access_token` đúng nên vẫn ghi được.)
- → Nếu dùng kênh website thật: repoint sang Leads/Deals Copy + sửa token alert.

---

## 8. MH-Dashboard — Daily / Weekly / Monthly + Ads (DỰNG LẠI 2026-06-08)

> ⚠️ Mô tả CŨ (query 5 bảng 404 + ghi DB1–DB5) đã KHÔNG còn đúng. Dashboard đã được **dựng lại hoàn toàn** thành 3 workflow, tích hợp **Meta Ads + Google Ads**. KHÔNG còn ghi DB1–DB5, KHÔNG còn query bảng 404.

**3 workflow chung 1 khung (Daily 10 node + Weekly 10 + Monthly 10):**
```
(Schedule | Manual) → Get Lark Token (tenant)
  → Query Leads Power (mhpower 3.1.0 Leads Copy tbliHIVs89nmwY0G, page_size 500, automatic_fields)
  → Query Leads Rental (mhrental 1. Leads liên hệ Copy tbldDVBv7XtDEsen)
  → Get Meta Ads (Graph act_915544230489694/insights, level=campaign)
  → Get Google Ads Token (OAuth refresh_token) → Get Google Ads (googleAds:search GAQL)
  → Build Summary (code) → Send Summary (Lark bot webhook 01eb5cb4-…)
```
| Workflow | ID | Lịch | Khung thời gian Ads |
|---|---|---|---|
| Daily Summary | `lpikvvTUDSRSAR1Y` | cron `0 8 * * *` | Meta `date_preset=yesterday`, Google `DURING YESTERDAY` |
| Weekly Report 🆕 | `LhH4KE4GkWSofusL` | schedule | Meta `last_7d`, Google 7 ngày |
| Monthly Report 🆕 | `Uz5Re5kFOxUyuirL` | schedule | Meta `last_30d`, Google 30 ngày |

- **Đếm lead** (`Build Morning Summary`): lọc record tạo trong khung (theo `created_time`/field `Ngày tạo` ms), phân theo field `Nguồn` (`messenger`/`face`/`google`/khác), tách Power vs Rental. **Dedup khách theo SĐT** (84→0) → hiển thị `{lượt} / {khách} (trùng form+chat: N)` nhưng **KHÔNG xóa/chặn lead nào** (khách cũ mua thêm vẫn tính).
- **Meta Ads:** token **System User never-expire** (`EAAVsBGCZCSWcBRoOey…` — khác token messaging MH-03). Phân Power/Rental theo keyword campaign (`pin|battery|mhpower|superv|lithium` → Power, còn lại Rental). Trả spend/clicks/impressions/CTR.
- **Google Ads:** `Get Google Ads Token` đổi refresh_token → access_token; `Get Google Ads Daily` POST `googleads.googleapis.com/v21/customers/1778238164/googleAds:search` (header `developer-token`, `login-customer-id 7248299313`), GAQL `SELECT campaign.name, metrics.cost_micros, clicks, impressions, ctr ... DURING YESTERDAY`. **cost = cost_micros / 1e6.**
- Mọi node query đều `onError: continueRegularOutput` → 1 nguồn lỗi (vd Lark/Meta/Google) KHÔNG làm hỏng cả báo cáo; phần đó hiển thị "⚠️ chưa lấy được".
- **Hạn chế đã biết:** (1) ⚠️ **Google Ads REST version sunset định kỳ** (~mỗi năm) → khi chết trả HTTP 404, bị onError nuốt → báo cáo "Google Ads: không có chiến dịch"; phải dò + bump version (06-08: v18→v21). (2) **Không lưu lịch sử ads** (chưa có bảng `ads_metrics`) → không vẽ được trend/CPL theo thời gian. (3) DB1–DB5 mhrental KHÔNG còn được ghi (mồ côi — quyết định bỏ hay đấu lại).

---

## 9. MH-DLQ — Error Handler (5 node)

```
Error Trigger (errorWorkflow toàn hệ thống) → Classify Error → IF TRANSIENT
   ├─[true]─► Send Transient Alert (Lark bot webhook)
   └─[false]► Send DLQ Alert (Lark bot webhook)
```
- `Classify Error`: phân loại TRANSIENT (timeout/econn/429/503/rate limit...) vs PERMANENT; lưu log `staticData.dlq` (giữ 200 mục) + đếm retry theo execId. Alert ghi rõ workflow/node/lỗi/lần thử.

---

## 10. Bảng Lark — ánh xạ ghi (verified)

**MHpower `ZxFDb2v4Fa3lcKssShclLMzSgLh`:**
| Bảng | Table ID | Ai ghi |
|------|----------|--------|
| Khách hàng Copy | `tblSEqXz15IKbTJE` | MH-03 Messenger PIN hoàn tất (hub liên kết) |
| 3.1.0 Leads Copy | `tbliHIVs89nmwY0G` | MH-03 (Messenger PIN, FB PIN), MH-01 PIN, MH-04 PIN |
| 3.1.0 Deals Copy (NKT) | `tblODvWhtO7d8crO` | MH-03 Messenger PIN hoàn tất |
| Lịch sử CSKH Copy | `tblegma1qJxJM6oq` | MH-03 Messenger PIN hoàn tất |
| leads marketings | `tblCuWfYzAIEMKHA` | MH-03 FB PIN, MH-04 Google PIN |
| 3.1.0 Leads / Deals (gốc) | `tblGsiuuVf5H3QwF` / `tblkevEQFeBdcQut` | MH-WEBSITE, MH-BOT(cũ) |

**MHrental `RYfKboTX0arHkRspUwMlMEUkgMd`:**
| Bảng | Table ID | Ai ghi |
|------|----------|--------|
| Khách hàng Copy | `tblIn03z4HG6qnYm` | MH-03 Messenger XE NÂNG hoàn tất (hub liên kết) |
| 1. Leads liên hệ Copy | `tbldDVBv7XtDEsen` | MH-03 (Messenger XE NÂNG, FB XE NÂNG), MH-01 rental, MH-04 rental; Dashboard đọc |
| 2. Lịch sử CSKH Copy | `tblVxV56K2ykR0uV` | MH-03 Messenger XE NÂNG hoàn tất |
| leads marketings | `tblgSX3899qKP4CO` | MH-03 FB XE NÂNG, MH-04 Google rental |
| DB1–DB5 | `tbllaxpZn1tL6ZCg` / `tblIPiWOEMgzrPfM` / `tblkAbgyd63auOgC` / `tblEW4PAju29iOKs` / `tblVyUieVKjHpICj` | MH-Dashboard ghi |
| Hợp đồng Copy | `tblFZq2kgyubr5SE` | (chưa đấu) |

**Nguyên tắc ghi Lark:** dùng tên cột tiếng Việt đầy đủ dấu; phone field (type 13) chỉ gán khi có giá trị; DuplexLink = mảng `[recordId]`; cùng 1 Lark App ghi cả 2 base.

### 10.1. Quan hệ 4 bảng Copy (Khách hàng Copy = hub)
Khi Messenger hoàn tất, MH-03 tạo Khách hàng Copy trước → gắn `record_id` vào trường link của Lead/Deal/CSKH (Lark tự sinh link 2 chiều):
```
mhpower:  Khách hàng Copy (tblSEqXz15IKbTJE) ─┬─ 3.1.0 Leads Copy (tbliHIVs89nmwY0G)
                                              ├─ 3.1.0 Deals Copy (tblODvWhtO7d8crO)
                                              └─ Lịch sử CSKH Copy (tblegma1qJxJM6oq)
mhrental: Khách hàng Copy (tblIn03z4HG6qnYm) ─┬─ 1. Leads liên hệ Copy (tbldDVBv7XtDEsen)
                                              └─ 2. Lịch sử CSKH Copy (tblVxV56K2ykR0uV)
```
Khách hàng Copy có schema y hệt bảng Khách hàng thật (mhpower `tblZVtsai7CSh3vF`, mhrental `tblPJspWzjxWVpam`). Bot chỉ điền Tên công ty/Phone/Email/Note; trường nghiệp vụ (MST/Mã CRM/công nợ/sale) để trống. Hợp đồng Copy (rental) KHÔNG link Khách hàng (bước cuối pipeline). **Nhánh FB Lead Form hiện chỉ tạo Lead (chưa tạo Khách hàng) — chỉ Messenger hoàn tất mới tạo đủ 4 bảng.**

---

## 11. Điểm rủi ro kỹ thuật (đang quản lý)

1. **FB token data access ~26/08/2026** (token phái sinh user) → đổi System User token. *Sống còn #1.*
2. **Secrets hardcode trong node** (Claude key, Lark secret, FB token/secret) — rotate = sửa nhiều node.
3. **Phụ thuộc tên cột Lark tiếng Việt** — nhân viên đổi/xóa cột "Copy" → fail ngầm (cron monitor chưa xây).
4. **Session trong staticData n8n** — mất nếu re-import workflow; giới hạn 200 session. **+ Clobber giữa các khách:** staticData persist khi execution KẾT THÚC → 2 khách nhắn trong ~20–45s (cửa sổ execution) có thể ghi đè session của nhau (last-write-wins). Debounce 6s chỉ che cùng 1 PSID. Khắc phục bền: chuyển sang Data Table — `MH_SESSION_DATATABLE_PLAN.md`. OK cho quy mô SME hiện tại; đừng re-import ẩu.
5. **MH-WEBSITE** chưa migrate (bảng gốc + alert lỗi auth).
6. **Meta Instant Reply/AI Inbox** nếu bật → giành tin nhắn, bot không nhận.

---

## 12. Bài học vận hành
- Verify token MH-03a phải khớp Meta; đổi 1 bên đổi cả 2.
- Xóa/sửa Lark CHỈ theo `record_id` đã xác nhận — KHÔNG filter wildcard tên (đã từng xóa nhầm 4 lead thật, khôi phục từ thùng rác Lark).
- Test tạo record phải ghi lại `record_id` ngay để xóa đúng đích; PSID test dùng tiền tố `6200000*`/`test_` (Load Session tự dọn).
- Đổi nội dung bot: sửa prompt trong `Build Messenger AI Request`; KHÔNG đổi JSON output schema (sẽ vỡ Parse).

---

*Tài liệu liên quan:* `MH_SYSTEM_KEYS.md` (khóa/ID) · `BAO_CAO_DU_AN_MH.md` (quản lý) · `MH_PROJECT_HANDOVER.md` (bàn giao) · `MH_USER_GUIDE.md` (vận hành theo vai trò).
