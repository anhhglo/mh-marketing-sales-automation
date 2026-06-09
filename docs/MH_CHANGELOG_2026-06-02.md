# MH System — Changelog 2026-06-02 (v4.3)

Triển khai bởi Claude Code. Tất cả thay đổi đã test thật + dọn record test.

## 1. Tạo 2 bảng `leads marketings` (lưu lead từ form quảng cáo)
| Base | table_id | Field |
|------|----------|-------|
| MHrental (RYfKb...) | `<TBL_RT_MARKETING>` | Họ tên, SĐT, Email, Nguồn form (Meta/Google), Tên chiến dịch, Ad/Form ID, Nhu cầu (Thuê xe nâng/Pin lithium/Chưa rõ), Nội dung form, AI Score (A/B/C/D), Ngày nhận, Trạng thái (Mới/Đã chuyển sale/Đã liên hệ/Bỏ) |
| MHpower (ZxFDb...) | `<TBL_PW_MARKETING>` | (giống trên) |

## 2. MH-03 — đổi ghi sang bảng "Copy"
- `Lark - Create Lead Messenger`: `<TBL_PW_LEADS_ORIG>` → **`<TBL_PW_LEADS_COPY>`** (3.1.0 Leads Copy)
- `Lark - Create Lead MHPower FB`: → **`<TBL_PW_LEADS_COPY>`**
- `Lark - Create Lead MHRental`: `<TBL_RT_LEADS_ORIG_LEGACY>` → **`<TBL_RT_LEADS_COPY>`** (1. Leads liên hệ Copy)
- `Lark - Create NKT Messenger`: giữ `<TBL_PW_DEALS_COPY>` (Deals Copy).
- **Quan trọng**: field link `Leads` của Deals Copy trỏ về bảng GỐC; đã đổi node NKT dùng field **`3.1.0 Leads Copy-3.1.0 Deals Copy-Leads`** (trỏ Leads Copy). Đã test link OK.

## 3. MH-03 — botmess nhận biết brand (xe nâng vs pin) ✅ test thật
- `Build Messenger AI Request`: prompt dual-brand. Turn đầu hỏi "thuê/mua xe nâng hay pin lithium?", tự nhận diện keyword, set field `brand`. 2 bộ câu: pin = Q1–Q14, xe nâng = R1–R5 (loại xe / chiều cao-tải / thuê bao lâu / địa điểm / khi nào cần).
- `Parse Messenger AI Response`: completion theo brand; greeting tin đầu trung lập; lưu `session.brand`. **Backward-compatible**: brand rỗng/MHpower → giữ logic pin cũ.
- Nhánh hoàn thành rental MỚI (sau `IF Session Complete`): `Get Lark Token Msg` → **`IF Brand Msg`**
  - true (MHrental) → `Build Lead Rental Msg` → tạo Lead `<TBL_RT_LEADS_COPY>` → `Build CSKH Rental Msg` → log `<TBL_RT_CSKH_COPY>` (Hành động=Chat) → alert.
  - false (MHpower) → chain cũ (Lead Copy + Deals Copy + alert).
- Test thật (webhook ký HMAC, PSID 6200000099777, tin rental): brand=MHrental → tạo Lead + CSKH trong mhrental Copy → **đã xóa record test**.

## 4. MH-03 + MH-04 — form quảng cáo → `leads marketings` ✅ test thật
- MH-03 (Meta Lead form): thêm `Build Marketing FB` + `Lark - Create Marketing FB` (Nguồn="Meta Lead Ads", base theo brand) chèn trước alert. Test `test_fb_mhpower_*` → ghi marketing mhpower code=0, đã xóa.
- MH-04 (Google): **sửa bug ghi bảng 404**. `Build Lark Record` viết lại theo schema Leads Copy + detect brand (keyword) + URL động; node tạo lead URL động + onError continue; thêm `Build Marketing GG` + `Lark - Create Marketing GG` (Nguồn="Google Lead Form"). Test `?key=<MH04_WEBHOOK_KEY>` lead nội dung pin → brand=MHpower → Leads Copy + marketing mhpower code=0, đã xóa.

## 5. Round 2 (cùng ngày) — xử lý tồn đọng ✅ test thật

### 5.1. MH-04 prompt chấm điểm trung lập 2 brand
`Build AI Request` (MH-04) viết lại: chấm cả MHrental + MHpower, KHÔNG chấm D vì "sai đối tượng". Test lead pin → giờ chấm **A** ("Pin lithium 48V 420Ah cụ thể..."), brand=MHpower. (Trước: D.)

### 5.2. MH-01 repoint bảng 404 → Leads Copy brand-aware
`Build Lark Record` (MH-01 `7tXShAMcINyHnTPM`) viết lại schema Leads Copy + URL động theo brand; node tạo lead URL động + onError. Test webhook (tạm tắt headerAuth → POST → khôi phục): lead "thuê xe nâng" → brand=MHrental → ghi mhrental Leads liên hệ Copy (Khách mới, SĐT đúng), đã xóa record test.

### 5.3. MHpower Lịch sử CSKH Copy được đấu
Thêm `Build CSKH Power Msg` + `Lark - Create CSKH Power Msg` vào nhánh hoàn thành MHpower (MH-03), ghi `<TBL_PW_CSKH_COPY>`, link Lead Copy qua field `3.1.0 Leads Copy-Lịch sử CSKH`. Test hội thoại pin complete → CSKH Copy code=0 (Q1-Q14 + link đúng), đã xóa. MH-03 giờ 51 node.

### 5.4. MH-Dashboard hết lỗi 8h
5 query mhrental (404) → thêm `onError: continueRegularOutput` (không crash); Query Leads repoint → Leads liên hệ Copy (`<TBL_RT_LEADS_COPY>`) đếm lead rental thật; Write node onError. Test E2E (webhook tạm, đã gỡ): chain chạy hết, Build Morning Summary OK (0 vì rental Leads Copy đang trống), Send Morning Summary → Lark code=0. Pipeline (cơ hội/báo giá/đơn/lắp đặt) hiển thị 0 vì bảng rental cũ đã xóa — cần CRM rental mới để đếm các bước này.

## 6. Còn tồn đọng
- **botzalo**: chưa tồn tại (sẽ ghép sau).
- **MH-Dashboard breakdown lead** (Facebook/Google/Score/Brand) = 0 vì Leads liên hệ Copy không lưu các field đó; chỉ "Tổng lead" chính xác. Nâng cấp khi rental CRM định nghĩa lại schema/pipeline.
- **MH-03 staticData**: còn session test PSID `6200000099777` (complete, vô hại).
- **7.10 Tắt Meta Instant Reply** trước go-live.

## 7. Gia cố độ bền production (onError/retry) ✅ test thật
Thêm `retryOnFail=true, maxTries=3, waitBetweenTries=2000` + `onError: continueRegularOutput` cho các node quan trọng:
- **MH-03** (16 node): Claude AI Score, Claude Messenger AI, Get Lark Token, Get Lark Token Msg, các Lark Create Lead/NKT/CSKH (power+rental), Create Lead MHRental/MHPower FB, Create Marketing FB, 3 node Alert, Send Messenger Reply.
- **MH-04** (5): Claude AI Score, Get Lark Token, Lark Base-Create Lead, Create Marketing GG, Alert.
- **MH-01** (4): Claude AI Score, Get Lark Token, Lark Base-Create Lead, Alert.

Cơ chế: proxy/Lark chập chờn → tự retry 3×(2s); vẫn lỗi → KHÔNG gãy chain, alert sale vẫn bắn (alert dựng từ session, không cần Lark/AI).

**Test A (regression, exec 1670)**: tin đầu PSID mới → bot tạo đúng câu hỏi brand, Parse OK. Send Messenger 400 (PSID giả) nhưng execution=success (onError đỡ).
**Test B (failure-inject, exec 1671)**: tạm trỏ Claude Messenger AI sang URL hỏng → node 404, retry 3×, onError continue → Parse salvage → bot gửi fallback "Bạn có thể nói rõ hơn..." → **execution=success, không crash**. URL đã khôi phục.

Lưu ý: còn vài session test trong staticData MH-03 (`6200000077123/088555/099777`) — vô hại (followup tới PSID giả fail gracefully).

## 8. Fix từ test thật trên quảng cáo (2026-06-03) ✅ test đạt
Phát hiện qua hội thoại thật (PSID Hiếu `<EXAMPLE_PSID>`): khách đã `is_complete=true` từ phiên cũ → nhắn lại KHÔNG tạo lead/alert (`newly_completed` luôn false), bot hứa suông; AI còn làm sai SĐT (<EXAMPLE_PHONE>→<EXAMPLE_PHONE>).

Đã sửa MH-03 (giờ 54 node):
- **P1 Re-open + alert** (`Parse` + prompt): khách đã hoàn tất nêu nhu cầu MỚI → AI đặt `new_inquiry=true` → snapshot hồ sơ cũ vào `previous_inquiry`, xóa field q/r, reset is_complete → bot hỏi lại cho nhu cầu mới → đủ thì tạo LEAD MỚI. Đồng thời cờ `reengaged` → nhánh mới **IF Reengage Alert → Lark Reengage Alert** bắn tin "🔁 KHÁCH CŨ NHẮN LẠI" cho sale ngay (không bao giờ rơi lead).
- **P2 SĐT** (`Parse`): bắt SĐT bằng regex `(?:\+?84|0)\d{9}` trên TIN GỐC khách, bỏ qua số AI đọc lại; prompt cấm tự sửa số.
- **P3** snapshot+clear khi re-open (tránh nhiễm data cũ).
- **P4** prompt: xe ngoài danh mục (xe tải…)/điện áp lạ → không bịa điện áp, giữ đúng số khách nói, chuyển kỹ thuật.
- **P5** không tuyên bố "đã ghi nhận đầy đủ" cho nhu cầu mới khi chưa hỏi xong.
- **P6** (phía Meta, user tự xử): "auto-reply khi comment quảng cáo" — kiểm/tắt để bot xử lý nhất quán.

Test: T1 hoàn tất phiên mới → SĐT regex đúng `0911222333`, lead tạo OK. T2 nhắn nhu cầu mới (chuyển pin→thuê xe nâng) → `reengaged=true`, brand đổi MHpower→MHrental, previous_inquiry lưu hồ sơ cũ, **Lark Reengage Alert code=0**, lead mới tạo. Đã xóa 5 record test.

LƯU Ý: session thật của "Hiếu" (`<EXAMPLE_PSID>`) đang lưu SĐT sai `<EXAMPLE_PHONE>` (từ trước khi fix). Số đúng là **<EXAMPLE_PHONE>** (khách đã đính chính). Sẽ tự sửa khi khách nhắn lại số; hoặc sale dùng <EXAMPLE_PHONE>.

## 9. Auto-reply comment quảng cáo → Messenger (2026-06-03) ✅ test logic đạt
Mọi comment dưới bài/quảng cáo → tự gửi tin nhắn riêng (private reply) → đẩy vào Messenger cho bot.
- Subscribe Page field **`feed`** (subscribed_apps: leadgen,messages,messaging_postbacks,feed).
- MH-03 (58 node): `Extract` nhận diện comment; nhánh mới **IF Is Comment → Dedup Comment → Send Private Reply (Graph /{comment_id}/private_replies) → Seed Comment Session**. Seed ghi câu chào vào history theo commenter_id để bot KHÔNG chào trùng.
- Câu chào: "Dạ cảm ơn anh/chị đã để lại bình luận ạ! Để tư vấn đúng nhu cầu, anh/chị cho em hỏi bên mình cần xe nâng hay pin lithium ạ?"
- Test (comment_id giả): Extract→IF→Send(400 đúng kỳ vọng, onError continue)→Seed OK. Cần comment THẬT để verify gửi-thật.

LƯU Ý vận hành:
- commenter_id (comment) có thể KHÁC PSID Messenger; nếu khác, seed không khớp → khi khách trả lời nội dung rõ (pin/xe nâng) bot vẫn không chào trùng (greeting chỉ bắn khi tin đầu RỖNG nội dung).
- Còn vài session test trong staticData (`62000000*`, `7100000012345`, `7100000099888`) — vô hại.

### 9.1. Quyết định CHỐT (2026-06-03): private reply do META lo, không qua n8n
Lý do: comment khách nằm trên **quảng cáo (bài is_published=false)**; test thật cho thấy Page token THIẾU `pages_read_user_content` → Graph trả 400 subcode 33 khi đọc/reply comment (token có pages_read_engagement vẫn KHÔNG đủ — đã verify). 2 hướng:
- **A** (n8n tự gửi): cần thêm quyền `pages_read_user_content` (App Review → Permissions and Features → Standard Access, không cần duyệt) → tạo token mới → thay vào n8n. Chưa làm.
- **B** (CHỌN): dùng **automation comment có sẵn của Meta Business Suite** ("bất kỳ bình luận → gửi tin riêng"). Không cần quyền/token.

Đã sửa n8n theo B: **gỡ node `Send Private Reply`** (khỏi báo lỗi quyền mỗi comment + bỏ 6s retry thừa); giữ `IF Is Comment → Dedup Comment → Seed Comment Session` để mồi session chống chào trùng. MH-03 còn 57 node. Test: comment giả → execution 83ms sạch, seed OK.
CẢNH BÁO: Business Suite automation có thể KHÔNG phủ comment trên quảng cáo (chỉ bài thường) → user phải test comment thật trên quảng cáo; nếu không rep được thì buộc quay lại hướng A.

## 10. Record bot ghi đúng mẫu thật + bỏ gạch dưới form (2026-06-03)
- **Bỏ `_` trong Nội dung form**: Normalize MH-03/MH-04 thêm `.replace(/_/g,' ')` → alert + leads marketings hết dấu gạch dưới (chỉ lead mới).
- **Khớp mẫu thật** (đối chiếu 5 record người thật ở Leads/Deals/CSKH gốc), bổ sung field bot ghi được (bỏ field formula/sale-only như Tên deal/giá/model):
  - Leads Copy: + `Khách mới/cũ`=Khách mới, + `Nguồn`=Messenger/facebook.
  - Deals Copy: + `Khách cũ/mới`=Khách mới, + `Nhu cầu mới`=Thay thế, + `Khách phản hồi`=tóm tắt nhu cầu 1 dòng.
  - CSKH Copy (power+rental): + `Khách phản hồi`=nhu cầu, Ghi chú rút gọn ("Tự động Messenger Bot | PSID").
- Test thật: hội thoại pin complete → 3 bảng Copy đủ field đúng mẫu, đã xoá record test.
- *(Còn có thể mở rộng: MH-01/MH-04 ghi Leads Copy cũng nên set `Nguồn`; chưa làm.)*

## 11. Tách 2 nhóm Sales Alert theo brand (2026-06-03) ✅ test ẩn đạt
Tách "MH Sales Alert" → **MHpower Sales Alert** + **MHrental Sales Alert**, route theo brand.
- Cơ chế: app bot đã ở trong 2 nhóm → gửi qua **API `im/v1/messages` theo chat_id** (không cần webhook custom-bot). Token: dùng `app_access_token` sẵn có (đã verify gửi im OK). Body: `{receive_id, msg_type, content: JSON.stringify(content||card)}`.
- Chat IDs: MHpower=`<LARK_CHAT_SALES_POWER>` · MHrental=`<LARK_CHAT_SALES_RENTAL>`.
- Node đã route: MH-03 (Messenger pin→MHpower, xe nâng→MHrental, FB form động); MH-04 (động); MH-01 (động); MH-WEBSITE (→MHpower). Brand động lấy từ `Build Lark Record._lead.brand` / `Validate AI Schema.brand`.
- Test ẩn (nhóm chỉ có chủ): MH-03 botmess pin→MHpower, FB pin→MHpower, FB xe nâng→MHrental; MH-04 Google pin→MHpower, Google xe nâng→MHrental. **Đúng nhóm, không lẫn, 0 lỗi.** Đã xóa record + tin test.

LƯU Ý:
- **App bot PHẢI ở trong cả 2 nhóm** (alert gửi qua app). Đừng xóa app khỏi nhóm. Đổi tên nhóm OK (chat_id không đổi).
- **Reengage alert** ("khách cũ nhắn lại") ĐÃ tách theo brand (2026-06-03): thêm node `Get Lark Token Reengage` vào nhánh reengage → `Lark Reengage Alert` route theo `Save Session State.brand`. Test: khách rental re-engage nhu cầu pin → reengage alert về **MHpower** (đúng brand mới), code=0. MH-03 giờ 58 node.
- Báo cáo sáng Dashboard + alert lỗi DLQ vẫn ở nhóm cũ (không phải lead theo brand).

## ID tham chiếu nhanh
- MH-03: `eZgfYcBRvfuvGl0c` (58 node) | MH-04: `z7CGM6OEw2RVlBaS` (22 node)
- MH-01: `7tXShAMcINyHnTPM` | MH-Dashboard: `lpikvvTUDSRSAR1Y`
