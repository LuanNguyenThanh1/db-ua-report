# Dashboard Changelog

---

## v1.5.3 (build từ v1.5.2, latest)
- **Bỏ cột Status** + filter status dropdown + toàn bộ code manual-status (picker, localStorage, STATUS_META)
- **Font x2** — gấp đôi toàn bộ 50 giá trị font-size cho dễ đọc trên desktop
- Version tag footer → v1.5.3
- **Bugfix Results sai (account TW-TOEIC 03 và mọi account):** `parseResultsField` cũ đoán count từ `actions` array bằng action_type hardcode → sai khi ad có nhiều custom event hoặc objective non-pixel (messaging). Vd Begin-checkout hiện 90 thay vì 31, messaging hiện 2 thay vì 22.
  - Fix: lấy count trực tiếp từ `results[].values` (= đúng cột Results của Ads Manager, đúng objective + attribution). Fallback actions array chỉ khi thiếu values.
  - Fix double-count: `results[].values` có nhiều entry (7d_click, 1d_view, **default**). Entry `default` = đúng số Ads Manager (combined dedupe) → lấy entry `default`, không sum (tránh C152=168).
  - Fix over-count khi objective = 0: nếu `results` có indicator nhưng KHÔNG có `values` (vd custom conversion 0 lần) → count=0, không đoán mù từ actions array (tránh C61 hiện 2 thay vì 0).
  - **Verified với data thật TW-TOEIC 03:** C152=84, C62=31, C6=22, C61=0 — khớp Ads Manager.
- **Tile mới "UA Creative Report"** (dưới Weekly Spend Report):
  - Auto group creative theo conversion type (EntryTestResult, Custom Conv., Messaging, Completed Registration...), show top 3 mỗi type rank theo results count
  - Layout: card hàng ngang scroll-x, collapsible (lưu trạng thái localStorage), header có total/type
  - Bám theo filter account + month + search hiện tại
  - Mỗi dòng: rank, ad name, account dot, CPR, results count
  - ⚠ Hạn chế: mọi custom conversion gộp chung label "Custom Conv." (label không tách được theo custom ID)
- **Filter mới "Result type"** cạnh month filter — lọc creative theo conversion type (populate động từ data), áp cả table + CSV export
- **Fix label:** `labelFromIndicator` strip prefix `actions:`/`conversions:` + map thêm `mobile_app_install`→App Install, `fb_pixel_complete_registration`→Completed Registration, các fb_pixel_* khác; fallback prettify thay vì hiện raw string
  - Thêm nhãn `messaging_conversation_started_7d` → "Messaging"

---

## v1.5.2 (build từ v1.5.1 — status thủ công)
**Thay đổi lớn:** Bỏ HẾT auto-classification (floor, CPR benchmark, top-performer, delta). Status giờ user tự gán tay.
- Xóa: `getCreativeStatus`, `computeAccountFloors`, `isTopPerformer`, `getBenchmarkCPR`, `CPR_BENCHMARKS`, các threshold const
- **Thêm cột "Status"** ngay cạnh cột Ad — dropdown chọn tay 1 trong 5: 🏆 Performing / 📉 Fatigue / 🆕 New / ○ Neutral / → Stable (hoặc bỏ trống)
- Dropdown viền màu theo status đã chọn
- Lưu **localStorage** theo `adId` (`manual_status`) → giữ nguyên khi re-fetch / đổi date range / reload
- Filter status (toolbar) giờ lọc theo status gán tay, không cần Compare Period

---

## v1.5.1 (build từ v1.5 base — status logic v3)
**Lý do:** v1.7/v1.8 status logic thấy chưa hợp lý → rollback về v1.5, định nghĩa lại cùng team.

**5 status — priority (match đầu thắng): New → Fatigue → Performing → Neutral → Stable**
- 🆕 **New**: có Compare Period nhưng creative không có data kỳ trước + spend>0
- 📉 **Fatigue**: spend delta < -10% **VÀ** không còn top performer
  - top performer = spend ≥ floor **VÀ** CPR ≤ benchmark
  - fix: creative tụt spend nhẹ nhưng vẫn top (CPR tốt) KHÔNG bị gắn Fatigue → rớt xuống Stable
- 🏆 **Performing**: spend ≥ floor **VÀ** CPR ≤ benchmark **VÀ** delta > +10%
  - fix: creative tiêu ít ($2 +100%) KHÔNG còn lọt Performing — phải vượt floor
- ○ **Neutral**: spend < $5 (đang test, kết quả chưa đáng kể) — chạy không cần Compare
- → **Stable**: fallback — có kỳ trước + spend ≥ $5 + không match trên

**Floor (động, per account):** `10% × tổng spend account đó (kỳ hiện tại)`
- Tính per-account kể cả ở "All accounts" (mỗi creative xét theo market của chính nó)

**Benchmark CPR ("average") hardcoded theo objective:**
- Begin-checkout / Custom Conv. → $4.0 | EntryTestResult → $1.0
- Completed Registration → $7.0 | messaging_conversation_started_7d → $4.5

**Khác:** Status filter luôn enabled (Neutral chạy không cần Compare). Bỏ disabled/dim cũ.

---

## v1.8
**Features — Status logic v2 (5 statuses, từ v1.6 base)**
- Bỏ **New** + **Stable**. Còn 5 status. Priority (match đầu thắng):
  **Fatigue → Performing → Bad → Rising Star → Neutral**
  - 🏆 **Performing**: spend ≥ floor + CPR < benchmark + delta > +10%
  - 📉 **Fatigue**: delta < -10% VÀ không còn là top performer hiện tại
  - 🚀 **Rising Star**: spend ≥ $5 VÀ (creative mới — không có kỳ trước HOẶC delta > +20%)
  - ⚠ **Bad**: spend ≥ $5 + CPR > benchmark × 1.2 (đắt hơn 20%)
  - ○ **Neutral**: spend < $5 + results ≤ 2 (đang test, chưa đáng kể)
- **Floor động** = 10% tổng spend của account/market đó (per account, kể cả ở All view — mỗi creative xét theo market của chính nó). Fix vấn đề creative tiêu $2 +100% bị gắn Performing.
- **Top performer hiện tại** = spend ≥ floor VÀ CPR ≤ benchmark. Fatigue chỉ gắn khi creative đã RỚT khỏi top (spend -10% + CPR vẫn ngon thì KHÔNG Fatigue).
- CPR benchmarks theo objective: Begin-checkout/Custom Conv. $4 · EntryTestResult $1 · Completed Registration $7 · messaging_conversation_started_7d $4.5
- Bad + Neutral chạy không cần Compare Period; Performing/Fatigue/Rising(+20%) cần Compare

---

## v1.7
**Features — Status Classification Rewrite**
- 6 statuses, định nghĩa chốt cùng team. Priority (match đầu tiên thắng):
  **Fatigue → Performing → Bad → Rising Star → Stable → Neutral**
  - 📉 **Fatigue**: kỳ TRƯỚC là performing (prev spend>$20 + prev CPR≤benchmark) VÀ kỳ này spend giảm <-10%
  - 🏆 **Performing**: top-5 spend/account + spend≥$10 + CPR<benchmark + delta>+10%
  - ⚠ **Bad**: CPR > benchmark×1.2 (đắt hơn 20%) — override Rising/Stable/Neutral
  - 🚀 **Rising Star**: spend≥$5 VÀ (creative mới — không có kỳ trước HOẶC delta>+20%)
  - **→ Stable**: spend>$10 + delta ±10%
  - **○ Neutral**: spend $5–$10 + CPR≤benchmark
- CPR benchmarks ("average") hardcoded theo objective:
  - Begin-checkout / Custom Conv. → $4.0
  - EntryTestResult → $1.0
  - Completed Registration → $7.0
  - messaging_conversation_started_7d → $4.5
- Top-5 tính per account (không phải global) — All accounts = 20 slots
- Fatigue check "was-performing" dựa trên prev period data (spend>$20 + CPR≤benchmark, không cần top-5)
- Aggregates pre-computed mỗi rerender, trước filter
- Bad + Neutral + Rising (new creative) hoạt động không cần Compare Period; Fatigue/Performing/Stable cần Compare

---

## v1.6
**UI**
- Tăng font size toàn dashboard cho desktop readability (+2–4px theo tier)
  - Body: 13 → 15px | Table: 12 → 14px | Ad name: 12 → 14px
  - KPI value: 20 → 24px | Header h1: 15 → 18px
  - Labels/muted: 10 → 11px | Status bar: 11 → 12px
  - Account tabs: 12 → 13px | Toolbar selects: 12 → 14px
  - Delta badges: 10–11 → 11–12px

---

## v1.5
**Features**
- Creative preview popover: click 👁 icon trong ad row → fetch creative từ Meta API (1 call / 1 ad, on-demand)
- Cache result trong memory (`thumbnailMap`) — click lần 2 instant, 0 API call
- Popover hiện: thumbnail/image, creative name, body copy, "⚙ Ads Manager" deeplink, "📸 IG Post" hoặc "📘 FB Post" nếu có permalink
- Button state: mờ khi loading, xanh khi đã cached
- Đóng: click ✕, click ngoài, hoặc click lại 👁 cùng row
- `thumbnailMap` reset khi Fetch Data mới (tránh stale cache)

**Removed (cleanup từ v1.4)**
- Xóa toàn bộ batch `fetchThumbnails()` — 400 ads × fetch = rate limit risk
- Xóa hover-triggered preview, mousemove listener
- Xóa skeleton shimmer CSS

**Checkpoint**
- v1.2: clean base (performance filter, no thumbnail code)
- v1.4: batch thumbnail approach (deprecated, kept as reference)

---

## v1.4
**Features**
- Creative thumbnail preview: sau khi Fetch Data, tự động batch-fetch thumbnail/image của từng ad từ Meta API (`creative{thumbnail_url,image_url,name,title,body}`)
- Thumbnails load non-blocking — table render trước, ảnh pop in theo từng batch 50 ads
- Mini thumbnail 40×40 hiện inline trong Ad Name cell, skeleton shimmer khi đang load
- Hover vào thumbnail → floating preview card 240px: ảnh lớn, ad name, creative body copy, tags (type, result event, performance status)
- Preview card follow chuột, tự adjust vị trí không bị out-of-viewport
- Creative name/title hiện nhỏ dưới ad name trong table row

---

## v1.3
**Features**
- Creative performance filter: classify từng ad là 🟢 Performing / 🔴 Fatigue / → Stable / ★ New dựa trên spend delta vs prev period (threshold ±10%)
- Status badge hiện inline trong Ad Name cell (chỉ show khi Compare Period đang active)
- Dropdown filter "All status / Performing / Fatigue / Stable / New" — disabled + dimmed khi chưa fetch compare, tự enable sau khi compare xong
- Clear Compare tự reset status filter về "All"

---

## v1.2
**UI & UX**
- Xóa box Results và CPR khỏi KPI summary row (số liệu không reliable ở cấp aggregate)
- Compare period: thêm loading overlay với spinner + progress text theo từng account
- Compare period: thêm 2s delay giữa mỗi account call → tránh Meta rate limit error
- Error message token hết hạn: đổi sang `Meta access token hết hạn — gọi Tessa cầu cứu 🆘`

---

## v1.1
**Features**
- Compare prev. period: fetch cùng kì timeframe đang chọn, hiện delta `↑/↓ %` inline trong table và KPI cards
- Month filter: dropdown filter creative theo tháng (tự extract từ tên ad `TW.MMYYYY...`)
- Onboarding tour: welcome modal + 5-step tooltip tour cho user mới, lưu localStorage

**UI**
- Redesign sang Notion-inspired light theme (default)
- Dark mode toggle (🌙 / ☀) — lưu preference localStorage
- Inter font
- KPI cards: rounded card với shadow thay vì border cứng
- Account tabs: pill style, màu tinted theo market
- Upload tile: đổi từ image → HTML file embed (iframe srcdoc) cho Weekly Spend Report

---

## v1.0
**Core features**
- Fetch Meta Ads data: 4 accounts (TW-TOEIC 03, TW-IELTS 2, KR-TOEIC, HK-IELTS)
- Metrics: Spend, Results, CPR, CPM, CPC, CTR, Hook Rate (3s/impr), Hold Rate (p50/impr)
- Attribution window: 7d_click + 1d_view
- Google Sheet integration qua Apps Script: Orders, Revenue, ROAS by creative (filter theo date range)
- Filter: account tabs, search ad name
- Sort: click column header
- Export CSV
- Dark UI (GitHub-inspired)
- Token lưu sessionStorage, Apps Script URL lưu sessionStorage
- Weekly Spend Report: image upload tile (localStorage)
- Git version control + deploy GitHub Pages
