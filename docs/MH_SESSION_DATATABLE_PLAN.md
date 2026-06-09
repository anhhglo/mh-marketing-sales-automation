# KẾ HOẠCH: Chuyển lưu session staticData → n8n Data Table
*Tạo: 2026-06-05 | Workflow: MH-03 (`eZgfYcBRvfuvGl0c`) | Trạng thái: CHỜ DUYỆT (chưa thực hiện)*

> Mục đích: khắc phục **clobber session giữa các khách** do n8n chỉ ghi `staticData` xuống DB khi execution KẾT THÚC, và các execution chạy chồng nhau ghi đè toàn cục (last-write-wins).

---

## 1. Vấn đề (đã quan sát thực tế 2026-06-05)
- `staticData.sessions` là **1 object toàn cục**, được n8n persist **khi execution kết thúc** (không phải lúc node `Save Session State` chạy).
- Tin hoàn tất chạy ~24–46s (do nhánh ghi Lark + alert + retry Send). Trong cửa sổ đó, nếu **2 khách khác nhau nhắn cách nhau ~20–45s**, execution chồng nhau → khi kết thúc, bản ghi sau **đè** bản trước → **mất session**.
- Debounce 6s chỉ chống trùng cho **cùng 1 PSID**, KHÔNG chống xung đột **giữa các PSID khác nhau**.
- Bằng chứng: test bắn 3 tin gần nhau → 2/3 session setup bị mất; bắn tuần tự cách 65s (> thời gian execution) → session lưu đúng, re-engage chạy hoàn hảo.

**Tần suất:** thấp khi tin thưa; tăng theo lưu lượng chat. Tải cao → rủi ro mất session/lead thật.

---

## 2. Giải pháp
Lưu mỗi session thành **1 dòng riêng theo PSID** trong **n8n Data Table** (upsert nguyên tử từng dòng) → khác PSID = khác dòng = **không clobber**.

### Ràng buộc kỹ thuật
n8n **Code node KHÔNG truy cập Data Table trực tiếp** — phải dùng **node Data Table** (Get/Upsert/Delete rows) đặt trước/sau Code node. MH-03 sẽ thêm ~6–10 node. Code đổi: đọc từ input (node Data Table Get phía trước), xuất ra node Data Table Upsert phía sau.

---

## 3. Thiết kế bảng

### `mh_sessions` (1 dòng/khách)
| Cột | Kiểu | Dùng |
|---|---|---|
| `psid` | string | khoá tra cứu |
| `session_json` | string | toàn bộ session (info / history / turn_count / is_complete / brand / previous_inquiry / followup_done ...) |
| `is_complete` | boolean | lọc cron follow-up |
| `last_is_bot` | boolean | lọc follow-up (lượt cuối là bot) |
| `updated_at` | number | tuổi follow-up + prune |
| `created_at` | number | TTL 30 ngày |

### `mh_buffers` (debounce, 1 dòng/khách)
`psid` (string) · `fragments_json` (string) · `token` (string) · `ts` (number)

> **Dedup** (`processed_leadgen_ids`, `processed_comment_ids`): TẠM GIỮ ở staticData — clobber ở đây chỉ rủi ro hiếm khi trùng 1 lead/comment, tác hại thấp. Migrate sau nếu cần.

---

## 4. Rewire luồng Messenger (MH-03)
```
IF Is Messenger
 → DT Get Buffer(psid) → Buffer Message(code: append fragment + token mới) → DT Upsert Buffer
 → Wait 6s
 → DT Get Buffer(psid) [đọc token mới nhất] → DT Get Session(psid)
 → Load Session(code: token != buffer → drop; gộp fragments; parse session_json; cleanup/migrate/TTL 30 ngày)
 → Build Messenger AI Request → Claude Messenger AI → Parse Messenger AI Response
 → Save Session(code: dựng session + 4 cột index) → DT Upsert Session → DT Delete Buffer
 → Send Messenger Reply → (nhánh complete / reengage / HR giữ NGUYÊN)
```
**Follow-up cron:**
`Followup Cron → DT Get Rows(is_complete=false AND last_is_bot=true AND updated_at < now-10m) → Scan Silent Sessions(code chọn mốc 10m/4h/24h + câu nhắc) → DT Upsert từng dòng + Send Followup Alert`
(nhờ cột index, không phải tải/parse toàn bộ).
**Comment seed:**
`Dedup Comment → DT Get Session(commenter_id) → Seed Comment Session(code) → DT Upsert Session`

---

## 5. Giai đoạn (mỗi bước test + revert được)
| GĐ | Việc | Rủi ro |
|---|---|---|
| **0. Chuẩn bị** | Tạo 2 Data Table rỗng (`mh_sessions`, `mh_buffers`). Không đụng production. | Không |
| **1. Dựng & test trên CLONE** | Clone MH-03, dựng luồng session-DataTable, test HMAC — **đặc biệt test concurrency**: 2 PSID khác nhau ĐỒNG THỜI → cả 2 đều lưu (không mất). | Thấp (clone) |
| **2. Cut over sessions** | Migrate 1 lần session đang sống staticData → bảng; chuyển Load/Save bản thật sang Data Table; tắt code staticData session. Buffer tạm giữ staticData. | TB — làm lúc vắng khách, có backup |
| **3. Buffer + cron + comment** | Chuyển buffers, Scan Silent Sessions, Seed Comment sang Data Table. | Thấp–TB |
| **4. Prune + dedup (tuỳ)** | Job dọn dòng `is_complete && >30 ngày` hoặc giữ N; (tuỳ) chuyển dedup. | Thấp |
| **5. Giám sát** | Theo dõi 1–2 ngày, gỡ tàn dư staticData. | Thấp |

**Khuyến nghị:** làm **GĐ0 + GĐ2 (sessions) trước** — gỡ ~90% rủi ro clobber, ít xáo trộn nhất; buffer/dedup theo sau.

---

## 6. Rủi ro & lưu ý
- **Độ trễ:** +vài trăm ms/tin (Get+Upsert) — chấp nhận được.
- **Read-modify-write cùng 1 PSID** (append buffer, mark followup) vẫn có thể đua nhưng **rất hiếm** (cùng 1 người) + debounce che phần lớn; cần tuyệt đối → dùng **fragment dòng-riêng** (append-only).
- **Giới hạn/giữ dòng Data Table** → thêm job prune (GĐ4).
- **Rollback:** giữ backup `all-wf_<ngày>.json` + clone; cutover lúc vắng khách; bật lại path staticData nếu lỗi.

## 7. Ước lượng
~Nửa ngày làm + test, chia nhỏ nên an toàn & quay lui được.

---

## 8. Phương án nhẹ hơn (nếu chưa muốn migrate)
Giữ staticData nhưng **rút ngắn execution** (đẩy phần ghi Lark sang sub-workflow async qua Execute Workflow → execution chính kết thúc nhanh → cửa sổ clobber nhỏ lại). Giảm rủi ro nhưng **không triệt để** như Data Table.

---

## 9. Trạng thái thực hiện
- [x] **GĐ0** — tạo 2 Data Table: `mh_sessions` (ID `<N8N_DT_SESSIONS>`), `mh_buffers` (ID `<N8N_DT_BUFFERS>`). ✅ 2026-06-05
- [x] **GĐ1** — PoC test concurrency (`MH-DT-CONCURRENCY-TEST` `y4KxxfO9q2aM1pR2`, đã tắt): **3 PSID gửi đồng thời (chồng execution qua Wait 8s) → CẢ 3 dòng đều lưu** (không clobber, khác hẳn staticData mất 2/3). **Upsert theo psid update đúng dòng cũ, không tạo trùng** (gửi lại AAA111 → cập nhật id 1, không thêm dòng). ✅ 2026-06-05 → **kết luận: cơ chế Data Table chống clobber + idempotent theo psid, đúng yêu cầu.**
- [x] **GĐ2** — cut over sessions (LIVE 2026-06-05): thêm node **DT Get Session** (trước Load) + **DT Upsert Session** (nhánh song song sau Save); sửa `Load Session State` đọc **DT-first → fallback staticData → tạo mới**. Save GIỮ NGUYÊN (dual-write staticData để cron/comment GĐ3 chưa-migrate vẫn chạy). MH-03 = 64 node, validate 0 lỗi. **Test đạt:** 2 PSID đồng thời → cả 2 lưu DT (không clobber); tin tiếp theo nạp đúng session từ DT (lịch sử nối tiếp, update đúng dòng). Session đang sống tự migrate sang DT khi khách nhắn lần tới (lazy, qua fallback). ✅
- [x] **GĐ3 (cron follow-up)** — LIVE 2026-06-05: thêm **DT Get Incomplete** (sau Followup Cron) + **DT Upsert Followup** (sau Scan); `Scan Silent Sessions` đọc session từ **DT rows** (không staticData). **Test đạt:** chèn session im >10', cron tick chọn đúng s10m, đánh dấu `followup_done.s10m`, giữ nguyên `updated_at`, ghi lại DT. MH-03 = 66 node, validate 0 lỗi. ✅
  - *Còn lại của GĐ3 (không bắt buộc cho go-live):* buffer + comment-seed vẫn dùng staticData (rủi ro clobber rất thấp: buffer chỉ cùng-PSID + debounce che; comment chỉ chạy khi có webhook comment). `Save Session State` vẫn dual-write staticData (vô hại, làm an toàn cho 2 phần trên). Migrate nốt sau nếu cần.
- [x] **GĐ4 (prune)** — workflow **MH-Session-Prune** (`YOQlvkjGYmbRXtWt`, ACTIVE): cron 3h sáng xóa dòng `updated_at < now-30 ngày`. ✅
- [x] **GĐ5** — PoC `MH-DT-CONCURRENCY-TEST` đã xóa; dữ liệu test đã dọn; backup final `/home/adminmh/backup/all-wf_2026-06-05_final.json` (10 wf). Giám sát: theo dõi `mh_sessions` có dòng/khách thật + DLQ alert.

### Cấu hình node Data Table đã verify (dùng cho GĐ2)
- `dataTableId`: `{ "__rl": true, "mode": "id", "value": "<tableId>", "cachedResultName": "mh_sessions" }` (thêm `cachedResultName` để UI hiển thị đúng).
- **Get/Upsert filter:** `matchType: "allConditions"`, `filters: { conditions: [{ keyName: "psid", condition: "eq", keyValue: "={{ $json.psid }}" }] }`.
- **Upsert columns (resourceMapper):** `{ mappingMode: "defineBelow", matchingColumns: ["psid"], value: {...map...}, schema: [{id,displayName,type,display:true,required:false,canBeUsedToMatch,removed:false}...] }`.
- **Get row khi không có match** → 0 item (flow dừng) → đặt `alwaysOutputData: true` ở node Get để code sau xử lý "session mới".

*(Cập nhật checkbox khi triển khai.)*
