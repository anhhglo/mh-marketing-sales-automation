# MH Bot Messenger — Kế hoạch nâng cấp toàn diện
*Tạo ngày: 2026-05-30 | Dựa trên 7 file MD chatbot cũ + đánh giá thực tế*

---

## 1. Quyết định: Datastore hay Vectorstore?

### Kết luận: **Datastore (n8n Datatable) + Enhanced System Prompt**

| Tiêu chí | Datastore | Vectorstore (RAG) |
|---|---|---|
| Setup | Đơn giản, dùng ngay | Cần embedding pipeline |
| Latency/tin | +50ms | +500-2000ms |
| Chi phí | Thấp | Cần Jina/Voyage mỗi query |
| Phù hợp dữ liệu | Bảng giá, config, sản phẩm (structured) | FAQ lớn, unstructured (100+ items) |
| Cập nhật admin | Dễ (Lark Table) | Phức tạp (re-embed) |
| Khi nào cần vectorstore | Bot mở rộng thành knowledge base lớn 100+ FAQ | — |

**Lý do chọn Datastore:**
- Bot hiện tại chủ yếu collect Q1-Q14 + tư vấn cơ bản
- Kiến thức sản phẩm/giá đủ nhỏ để nhét vào system prompt (~1000 tokens)
- Config/admin guidelines → Lark Table `MHBot_Config`
- Catalog sản phẩm → Lark Table `MHBot_Products`
- Vectorstore chỉ hợp lý khi bot expand thành full knowledge base (Sprint 4+)

---

## 2. Phân tích vấn đề từ file MD cũ vs bot hiện tại

### 2.1 Những gì bot cũ có mà bot Messenger hiện tại THIẾU

| Năng lực | Bot cũ (Zalo) | Bot Messenger hiện tại | Sau nâng cấp |
|---|---|---|---|
| Phong cách giao tiếp tự nhiên | ✅ Đầy đủ | ❌ Cứng nhắc "CSKH" | ✅ Thêm vào system prompt |
| Kỹ năng xử lý phản đối giá | ✅ 8 mẫu | ❌ Không có | ✅ Key patterns trong prompt |
| Kiến thức sản phẩm SuperV | ✅ Model, specs, giá | ❌ Không có | ✅ Giá tham khảo + dòng SP |
| Tư duy TCO (tổng chi phí) | ✅ Đầy đủ | ❌ Không có | ✅ Thêm vào prompt |
| Đọc tâm lý khách (5 loại) | ✅ Có | ❌ Không có | ✅ Trong system prompt |
| Chốt bước tiếp theo | ✅ 6 loại | ❌ Máy móc | ✅ Cải thiện |
| Giá tham khảo | ✅ Bảng đầy đủ | ❌ Hoàn toàn không biết | ✅ Giá range trong prompt |
| FAQ 10 câu | ✅ Có | ❌ Fallback | ✅ Thêm key FAQ |
| After-sales cadence | ✅ 7d/30d/90d/6m | ❌ Không có | 📋 Sprint 3 |

### 2.2 Quy tắc KHÔNG ĐƯỢC làm (bắt buộc giữ)
Từ file 02 và 04:
1. KHÔNG bịa giá, tồn kho, thời gian giao hàng, thông số kỹ thuật
2. KHÔNG hướng dẫn tháo pin, sửa BMS, bypass bảo vệ, thay cell, đấu dây
3. KHÔNG cam kết bảo hành/chiết khấu khi chưa có sale xác nhận
4. KHÔNG gửi thông tin chuyển khoản
5. KHÔNG chê đối thủ trực tiếp
6. KHÔNG ép mua ("chốt đi", "không mua là sai")
7. KHÔNG hỏi quá 1-3 câu/lượt
8. KHÔNG báo giá chính thức khi chưa biết đủ cấu hình

---

## 3. Kiến trúc mới đề xuất

```
Facebook Messenger
    │
    ▼
Webhook → Verify Sig → Extract → IF Messenger
                                     │
                               Load Session State
                                     │
                         ┌─── [Sprint 2] Fetch MHBot_Config + MHBot_Products ───┐
                         │                                                        │
                         ▼                                                        │
                   Build AI Request                                               │
                   ┌─────────────────────────────────────────────────────────┐   │
                   │ SYSTEM PROMPT (Enhanced — Sprint 1):                    │   │
                   │ - Phong cách giao tiếp (file 03)                        │   │
                   │ - Kỹ năng bán hàng (file 04)                           │◄──┘
                   │ - Knowledge base sản phẩm (file 05)                    │
                   │ - Giá tham khảo (file 06)                              │
                   │ - Key FAQ + scripts (file 07)                          │
                   │ + [Sprint 2] Dynamic: MHBot_Config + MHBot_Products    │
                   └─────────────────────────────────────────────────────────┘
                         │
                   Claude Haiku AI
                         │
                   Parse AI Response (fixed scope bug)
                         │
                   Save Session → Send Reply → IF Complete
                                                    │
                                              Lark Lead + NKT + Alert ✅
```

---

## 4. Roadmap chi tiết

### Sprint 1 — Enhanced Prompt (THỰC HIỆN NGAY, hôm nay)

**Mục tiêu:** Bot tư vấn như sale giỏi, không chỉ collect data máy móc.

**Thay đổi:**
1. ✅ Tách `system` prompt riêng (không gộp vào `user` message)
2. ✅ Thêm phong cách giao tiếp tự nhiên (xưng hô, nhịp điệu)
3. ✅ Thêm xử lý phản đối giá/so sánh
4. ✅ Thêm kiến thức sản phẩm SuperV (dòng SP, BMS A/B/C, kho)
5. ✅ Thêm giá range tham khảo (không phải giá chính thức)
6. ✅ Fix bug: `content` variable scope trong catch block
7. ✅ Thêm tình huống đặc biệt: xe hỏng gấp, khách kỹ tính, so giá

**Files thay đổi:** `Build Messenger AI Request`, `Parse Messenger AI Response`

---

### Sprint 2 — Dynamic Data (1-2 tuần)

**Mục tiêu:** Admin có thể cập nhật dữ liệu sản phẩm/config mà không cần sửa code.

**Tạo Lark Tables:**

**MHBot_Products** (tblXXX trong cùng Base <LARK_BASE_MHPOWER>):
| Cột | Kiểu | Ví dụ |
|---|---|---|
| model | text | SVF48-314 |
| product_group | text | xe_nang_hang |
| voltage | text | 51.2V |
| capacity_ah | number | 314 |
| price_2y_million | number | 60.0 |
| price_5y_million | number | 74.4 |
| use_case | text | Xe nâng 2-3 tấn, 48V |
| notes | text | Phù hợp ca 8h, có đối trọng |
| active | boolean | true |

**MHBot_Config** (tblXXX):
| key | value |
|---|---|
| greeting | "Dạ em chào anh, em hỗ trợ tư vấn pin lithium SuperV..." |
| price_response | "Dạ giá tùy cấu hình, sale sẽ báo chính xác sau khi tư vấn" |
| escalation_keywords | "khiếu nại,bảo hành,lỗi,hỏng,cháy" |
| max_history | "10" |
| model | "claude-haiku-4-5-20251001" |

**Thay đổi workflow:**
- Thêm node `Fetch Bot Config` sau `Load Session State`
- Thêm node `Fetch Products` (parallel)
- `Build AI Request` inject dữ liệu từ tables vào prompt

---

### Sprint 3 — After-sales Cadence (2-4 tuần)

**Mục tiêu:** Tự động chăm sóc sau bán và follow-up lead im lặng.

**Workflow mới (MH-04):**

```
Lark Trigger (cron mỗi giờ) →
  Đọc Leads chưa có sale gọi →
  Tính time delta →
  IF 4h im lặng → gửi tin nhắn follow-up (mẫu 04h)
  IF 24h im lặng → gửi tin nhắn follow-up (mẫu 24h)
  IF 72h → hỏi lại nhẹ (mẫu 72h)
  IF 7 ngày → lưu cold, hỏi lần cuối (mẫu 7d)

Lark Trigger (trigger khi Bàn giao = Done) →
  IF 7 ngày → kiểm tra pin tuần đầu (mẫu 7d)
  IF 30 ngày → kiểm tra 1 tháng (mẫu 30d)
  IF 90 ngày → check định kỳ (mẫu 90d)
  IF 6 tháng → upsell xe còn lại (mẫu 6m)
```

**Mẫu tin nhắn follow-up** (từ file 07):
- 4h: "Dạ em gửi lại để anh tiện xem. Chỉ cần ảnh tem bình hoặc V/Ah là em kiểm tra nhanh."
- 24h: "Dạ anh còn nhu cầu thay pin lithium không? Em gửi 2 phương án: tối ưu chi phí và phương án bền hơn."
- 72h: "Dạ em hỏi lại, hiện tại mình tạm dừng hay vẫn cần báo giá ạ?"
- 7d: "Dạ nếu chưa triển khai, em lưu thông tin. Khi cần anh nhắn lại em sẽ hỗ trợ nhanh."
- 7d after install: "Dạ bộ pin mới lắp tuần đầu ổn không ạ?"
- 30d: "Dạ em kiểm tra 1 tháng sử dụng. Pin vận hành ổn không?"
- 90d: "Dạ 3 tháng vừa rồi pin có phát sinh lỗi nào không?"
- 6m: "Dạ bộ pin 6 tháng rồi. Đội xe còn xe bình chì em có thể tư vấn phương án thay dần."

---

### Sprint 4 — Vectorstore (tương lai, khi cần)

**Điều kiện:** Chỉ làm khi:
- FAQ > 50 câu thường xuyên thay đổi
- Bot expand thành full consultative AI (không chỉ Q1-Q14)
- Có nhu cầu semantic search trong knowledge base lớn

**Stack:** Jina AI embedding (đã có key) + n8n vector store node + Qdrant/Pinecone

---

## 5. Metrics theo dõi hiệu quả

| Metric | Trước nâng cấp | Mục tiêu |
|---|---|---|
| Completion rate Q1-Q14 | ~70% | >85% |
| Fallback rate | ~15% | <5% |
| Re-ask same question | Thường xuyên | <10% |
| Conversion: chat → lead Lark | ~70% | >90% |
| Avg turns to complete | 20+ | <15 |
| Khách hỏi giá → tiếp tục | Thấp | >60% |

---

## 6. Tóm tắt thay đổi Sprint 1 (thực hiện hôm nay)

### Node `Build Messenger AI Request`
- **Trước:** Prompt gộp vào `user` message, không có sales skills, không biết giá, fallback ngắn
- **Sau:** Tách `system` + `user`, có phong cách tự nhiên, biết xử lý phản đối, có giá range tham khảo

### Node `Parse Messenger AI Response`
- **Trước:** Bug `content` variable out of scope trong catch block
- **Sau:** Move `content` ra ngoài try/catch, fix scope

### Không thay đổi
- Lark Lead/NKT creation flow (đã hoạt động tốt)
- Alert sale workflow
- Session management
- Dedup logic
- HMAC signature verification

---

*Kế hoạch này sẽ được cập nhật sau mỗi sprint. Xem thêm MH_BOT_EVALUATION_2026-05-30.md cho bug list chi tiết.*
