# MH System — Changelog 2026-06-09

> Chống trùng lead **2 chiều (bidirectional)** + housekeeping cho MH-03. Tất cả đã live, validate 0 lỗi, test thật pass.

## Bối cảnh
Bản chống-double 06-08 (`mh_lead_dedup`, gate `IF Is Dup`) chỉ chặn **1 chiều**: leadgen về trước → form-echo Messenger sau bị bỏ. Còn 4 điểm hở/housekeeping:
1. **Race ngược chiều** — form-echo về trước, leadgen sau → vẫn lọt 1 cặp trùng.
2. `DT Mark Leadgen` fire-and-forget — fail thì không có marker.
3. `mh_lead_dedup` không prune → phình.
4. `session.lark_created=true` set trước gate → khi bị chặn vẫn ghi `true` (hiểu nhầm).

## Thay đổi (MH-03 `eZgfYcBRvfuvGl0c`: 73 → **85 node**)

### #1 — Chống double 2 chiều (full bidirectional, fail-open)
- **Messenger non-dup ghi marker** (`Build Msg Dedup Key` → `DT Mark Messenger`): khóa **`core:msg`**, chỉ khi `is_fb_form`.
- **Leadgen check trước khi tạo** (`Validate AI Schema` → `DT Check Msg Dedup` → `Eval Leadgen Dedup` → `IF Leadgen Dup`): nếu thấy marker `core:msg` < 6h → bỏ tạo lead leadgen; ngược lại tạo bình thường.
- **Fail-open:** mọi lỗi/empty → `_is_dup=false` → vẫn tạo lead leadgen. Worst case = trùng như trước, KHÔNG bao giờ nuốt lead.
- **Tradeoff:** ca hiếm form-echo về trước → giữ lead Messenger đã tạo, bỏ bản leadgen (mất AI score bản đó).

### ⚠️ Bug phát hiện khi test (exec 4851) → đã sửa
Leadgen và messenger marker ban đầu dùng **chung khóa `dkey=core`** → `DT Mark Leadgen` (upsert theo dkey) **ghi đè** marker messenger thành `source=leadgen` trước khi leadgen check → reverse vô hiệu. **Fix:** tách khóa messenger sang **`core:msg`** (leadgen giữ `core`). 2 row riêng, hết va chạm. Forward gate (`DT Check Dedup` đọc `core`) KHÔNG đổi.

### #2 — `DT Mark Leadgen` bền hơn
`maxTries` 3→5, `waitBetweenTries` 1000→1500, `alwaysOutputData=true`.

### #3 — Workflow mới `MH-Dedup-Prune` (`60X0O5OOECjwJ8LU`, active)
Cron 4AM, xóa marker `mh_lead_dedup` có `created_at` > 48h (TTL dedup thực tế 6h). Mirror `MH-Session-Prune`. → Tổng workflow live = **13**.

### #4 — Cờ session đúng
Nhánh trùng (`IF Is Dup` true, trước là ngõ cụt) → `Mark Dedup Skipped` → `DT Upsert Session Skipped`: ghi `lark_created=false, dedup_skipped=true`. Hết hiểu nhầm khi đọc session.

## Kiểm thử (webhook ký HMAC thật, soi execution)
| Ca | Kết quả |
|---|---|
| Reverse: marker `core:msg` tồn tại → bắn leadgen | exec **4852** `_is_dup=true` → `IF Leadgen Dup` route skip, **không tạo lead** ✓ |
| Forward + #4: marker `core` tồn tại → form-echo trùng | exec **4854** `is_dup=true`, session `lark_created=false, dedup_skipped=true`, **không tạo lead** ✓ |
| Fail-open: không có marker `:msg` → leadgen | exec **4851** `_is_dup=false` → tạo lead bình thường ✓ |
| #1A (ghi marker non-dup) | verify config-parity (`DT Mark Messenger` ≡ `DT Mark Leadgen`); không live-fire để tránh tạo record Lark không xóa được qua MCP |

Đã dọn toàn bộ marker/session/record Lark test. Test harness: `MH-03_HMAC` (ký SHA256 rawBody bằng FB App Secret, POST `/mh-facebook-lead`).

## Không ảnh hưởng
Khách Messenger thuần (1 webhook), chat thường, comment, tuyển dụng, leadgen thuần → không đổi. Toàn bộ node mới `onError: continueRegularOutput`.

## Codebase
`mh-system/workflows/all-wf.json` đã re-export (13 workflow, sanitize secret, **bỏ staticData** để không lộ PII khách). `MH_SYSTEM_ARCHITECTURE.md` cập nhật node count + sơ đồ dedup 2 chiều + bảng workflow.
