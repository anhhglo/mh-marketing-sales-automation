# DASHBOARD ADS (Meta + Google) — ✅ ĐÃ TRIỂN KHAI & ĐANG CHẠY
*Tạo: 2026-06-05 | Cập nhật: 2026-06-08 | Trạng thái: **LIVE** — Meta + Google Ads đã ghép chung dashboard, chạy hằng ngày*

> **CẬP NHẬT 2026-06-08 — ĐÃ LÀM XONG (khác hẳn trạng thái "HOÃN" ban đầu):** Google Ads Basic Access **đã được duyệt (2026-06-06)** → dashboard Meta + Google đã được **dựng và chạy LIVE** trong 3 workflow n8n:
> - **MH-Dashboard: Daily Summary** (`lpikvvTUDSRSAR1Y`) — báo cáo sáng 8h: leads theo nguồn (Power/Rental) + chi phí Meta + chi phí Google (hôm qua).
> - **MH-Dashboard: Weekly Report** (`LhH4KE4GkWSofusL`) — báo cáo tuần (Meta last_7d + Google).
> - **MH-Dashboard: Monthly Report** (`Uz5Re5kFOxUyuirL`) — báo cáo tháng (Meta last_30d + Google).
>
> **Cách lấy số (đang chạy):** Meta = Graph `<META_AD_ACCOUNT_MAIN>/insights` (token System User never-expire); Google = OAuth refresh token → `googleads.googleapis.com/v21/customers/<GOOGLE_ADS_CUSTOMER_ID>/googleAds:search` (login-customer-id <GOOGLE_ADS_LOGIN_CUSTOMER_ID>, dev token <GOOGLE_ADS_DEV_TOKEN>…). Mọi node `onError: continueRegularOutput`.
>
> **CÒN LẠI (chưa làm, theo §3 dưới):** (1) Chưa tạo bảng Lark `ads_metrics` — số liệu chỉ gửi text vào nhóm Lark, KHÔNG lưu lịch sử để vẽ trend/CPL theo thời gian. (2) ⚠️ Google Ads REST version **sunset định kỳ** (~mỗi năm) → khi chết trả HTTP 404, bị onError nuốt → báo cáo hiện "Google Ads: không có chiến dịch"; phải dò + bump version (2026-06-08 đã bump v18→v21). (3) Token Meta dashboard hiện đã dùng System User never-expire (tốt).
>
> *Phần kế hoạch gốc bên dưới giữ lại để tham khảo lịch sử.*

---

## (LỊCH SỬ) Quyết định ban đầu 2026-06-05: HOÃN
> Quyết định của chủ dự án (2026-06-05): **chưa làm ngay**. Meta Ads đã verify lấy được số liệu, nhưng để ghép chung 1 dashboard với Google → **chờ token/Basic Access Google Ads duyệt rồi làm cả hai cùng lúc.** *(→ Đã duyệt 06-06, đã triển khai 06-08 — xem block trên.)*

---

## 1. Meta Ads — ĐÃ VERIFY LẤY ĐƯỢC (2026-06-05)
Gọi thẳng Facebook Graph API `v21.0` bằng token có `ads_read` + `ads_management` (token suy từ admin Kiên MH Power). **Không cần chờ duyệt gì** — chạy được ngay.

**Tài khoản quảng cáo truy cập được:**
| account_id | Tên | Currency | Đã chi (tổng) |
|---|---|---|---|
| `<META_AD_ACCOUNT_MAIN>` | CÔNG TY TNHH MH POWER | VND | (chính) |
| `<META_AD_ACCOUNT_ALT>` | Kiên MH Power | VND | (phụ) |

**Số liệu mẫu đã kéo (account `<META_AD_ACCOUNT_MAIN>`, last_30d):**
spend=6.363.037đ · impressions=133.472 · clicks=3.927 · CTR=2.94% · CPC=1.620đ · reach=65.521.

**Campaign-level đã chạy OK** (lấy được campaign_name, spend, impressions, clicks, ctr...). VD top: "KH tiềm năng nhà máy SuperV", "Quảng cáo pin mới", "Tuyển TTS Pin" (CTR 6.95%), "Tuyển dụng kỹ thuật pin" (CTR 8.67%)...

**Endpoint dùng:**
- Tài khoản: `GET /me/adaccounts?fields=account_id,name,account_status,currency,amount_spent`
- Insights account: `GET /act_<id>/insights?fields=spend,impressions,clicks,ctr,cpc,cpm,reach,frequency&date_preset=last_30d`
- Insights campaign: thêm `level=campaign&fields=campaign_name,...&limit=N`
- Có thể `level=adset` / `level=ad`; breakdown theo ngày: `time_increment=1`; theo vùng/tuổi/giới/placement: `breakdowns=...`.

**Lấy được cho dashboard:** chi phí, hiển thị, click, CTR, CPC, CPM, reach, frequency, conversions, cost-per-lead, ROAS — theo account/campaign/adset/ad, theo thời gian + breakdown.

---

## 2. Google Ads — ĐANG CHỜ (chưa lấy được)
- Dev token `<GOOGLE_ADS_DEV_TOKEN>` ở **chế độ Test** → chỉ đọc test account, không đọc account thật `<GOOGLE_ADS_CUSTOMER_ID>`.
- Đang chờ duyệt **Basic Access** (Google Ads API Center). Chi tiết: `MH_SYSTEM_KEYS.md` §7 + memory `project_google_ads_basic_access`.
- **Khi được duyệt** → cấu hình `developer_token`, `login_customer_id=<GOOGLE_ADS_LOGIN_CUSTOMER_ID>`, `customer_id=<GOOGLE_ADS_CUSTOMER_ID>` + OAuth refresh token → kéo cost/CTR/keyword.

---

## 3. KẾ HOẠCH KHI LÀM (cả 2 cùng lúc)
1. **Token bền cho Meta `ads_read`:** dùng System User token (data access vĩnh viễn) thay token admin ngắn hạn — để dashboard tự chạy hằng ngày không chết token. (Gắn với việc chuyển FB token sang System User.)
2. **Bảng Lark `ads_metrics`** (mới) — schema đề xuất:
   - Ngày (date) · Nguồn (select: Meta | Google) · Tài khoản · Tên chiến dịch · Cấp (account/campaign/adset/ad)
   - Chi phí (number) · Hiển thị · Click · CTR (%) · CPC · CPM · Reach
   - Lead (number, ghép từ `leads marketings`) · CPL (chi phí/lead) · Ghi chú
3. **Workflow n8n `MH-Ads-Dashboard`:** cron mỗi sáng → (a) Meta Graph insights, (b) Google Ads API → normalize chung schema → ghi `ads_metrics` → ghép `leads marketings` theo `Tên chiến dịch` để tính **CPL thật theo campaign** → gửi tóm tắt vào nhóm Lark.
4. **Dashboard view trong Lark:** chi phí theo campaign/brand, CTR, CPL, xu hướng theo ngày, so sánh Meta vs Google.

---

## 4. Việc cần làm khi mở lại (checklist)
- [ ] Google Ads Basic Access đã duyệt? (điều kiện kích hoạt)
- [ ] Chuẩn bị token Meta `ads_read` dài hạn (System User)
- [ ] Tạo bảng Lark `ads_metrics`
- [ ] Dựng workflow `MH-Ads-Dashboard` (Meta + Google)
- [ ] Ghép CPL với `leads marketings`
- [ ] Test + activate + thêm vào báo cáo sáng

*(Liên quan: `MH_SYSTEM_KEYS.md` §7, `MH_SESSION_DATATABLE_PLAN.md`, `botmrketlark.md`.)*
