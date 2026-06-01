# Dashboard Changelog

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
