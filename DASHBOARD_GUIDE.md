# Meta Ads Creative Dashboard — Build Guide
> Stack: Claude Code + Meta MCP + Meta Graph API · Output: static HTML file

---

## Prerequisites

- Claude Code với Meta MCP đã kết nối
- Meta access token (User token hoặc System User token)
- Ad account IDs (`act_XXXXXXXXX`) của các accounts cần track

---

## Step 1 — Clarify requirements trước khi build

Prompt Claude Code:

```
Tôi muốn build 1 static HTML dashboard để visualize creative performance
từ Meta Ads. Hỏi tôi các câu hỏi cần thiết trước khi build.
```

**Những thông tin cần xác nhận:**
| Item | Cần biết |
|------|----------|
| Output format | HTML tĩnh / Python app / React |
| Account IDs | `act_XXXXXXXXX` cho từng market |
| Breakdown level | Ad / Adset / Campaign |
| Metrics | Spend, Results, CPM, CPC, CTR, Hook Rate, Hold Rate |
| Results event | Tên custom pixel event (vd: `EntryTestResult`) |
| Hook Rate formula | 3s views / impressions |
| Hold Rate formula | p50 views / impressions |
| Date range | Custom picker / preset |
| Filter | Theo account, search theo tên ad |

---

## Step 2 — Kiểm tra conversion event qua MCP

Trước khi build, dùng MCP verify data structure:

```
Dùng ads_get_ad_entities với fields [id, name, spend, results, actions]
cho account act_XXXXXXX, date_preset last_7d, limit 5.
Tôi cần biết results field trả về format gì.
```

**Kết quả quan trọng cần ghi nhớ:**

- `results` field từ raw API = `[{"indicator":"conversions:offsite_conversion.fb_pixel_custom.EventName"}]` — chỉ có label, **không có count**
- Count nằm trong `actions` array, key = `offsite_conversion.fb_pixel_custom`
- Custom conversion object = `offsite_conversion.custom.{ID}`
- Video 3s views = `action_type: "video_view"` trong `actions` array
- Hold Rate (p50) = field `video_p50_watched_actions`
- Field `video_3_sec_watched_actions` **không valid** ở API v21.0

---

## Step 3 — Build dashboard

```
Build 1 file HTML tĩnh, gọi Meta Graph API trực tiếp từ browser.
Accounts: [paste account list]
Metrics: Spend, Results, CPR, CPM, CPC, CTR, Hook Rate, Hold Rate
- Hook Rate = video_view action / impressions
- Hold Rate = video_p50_watched_actions / impressions  
- Results: label từ results[0].indicator, count từ actions offsite_conversion.fb_pixel_custom
- Attribution window: 7d_click + 1d_view
- Token lưu sessionStorage
- Filter theo account tabs + search box
- Sort by column header
- Export CSV
```

---

## Step 4 — API fields đúng

Khi Claude build API call, đảm bảo fields này:

```
ad_id, ad_name, spend, impressions, clicks, cpc, cpm, ctr,
results, actions, video_p50_watched_actions
```

URL params cần có:
```
&action_attribution_windows=["7d_click","1d_view"]
```

---

## Step 5 — Debug nếu Results = 0

Nếu cột Results trống, add debug log:

```
Add console.log in fetchAccount:
JSON.stringify(d.results) và JSON.stringify(d.actions)
để xem raw format API trả về.
```

**Các lỗi thường gặp:**

| Triệu chứng | Nguyên nhân | Fix |
|-------------|-------------|-----|
| Results = 0 | Parse `results` sai format | label từ `results[0].indicator`, count từ `actions` |
| Results sai event | Dùng highest-count heuristic | Không dùng priority/count — dùng đúng field `offsite_conversion.fb_pixel_custom` |
| `video_3_sec_watched_actions` invalid | Field deprecated v21.0 | Dùng `action_type: "video_view"` trong `actions` array |
| CPR trống | Results = 0 → divide by zero | Fix results trước, CPR = spend / results tự tính |

---

## Step 6 — Lưu version & mở lại

**Init git và commit v1.0:**
```bash
cd "path/to/folder"
git init && git add dashboard.html && git commit -m "feat: dashboard v1.0"
```

**Mở lại dashboard:**
```bash
cd "path/to/folder"
python3 -m http.server 7788
# → http://localhost:7788/dashboard.html
```

**Rollback về version cũ:**
```bash
git log --oneline                          # xem list versions
git checkout <commit-hash> dashboard.html  # rollback file
```

---

## Quick reference — Meta API gotchas

| Field | Raw API behavior |
|-------|-----------------|
| `results` | Array with indicator only — no count value |
| `actions` | All action types including conversions |
| `offsite_conversion.fb_pixel_custom` | All custom pixel events grouped (no event name) |
| `offsite_conversion.custom.{ID}` | Meta Custom Conversion object |
| `video_view` action_type | = 3-second video views |
| `cost_per_result` | May return array or null — unreliable for count |
| `ctr` | Returned as percentage string (e.g. `"1.82"`) not decimal |

---

## Rate limit — thực tế

- 1 fetch session (4 accounts ~150 ads) ≈ 8 API calls
- Ngưỡng Meta: ~200 calls/giờ per token
- Dùng bình thường không bao giờ chạm ngưỡng
- Bị limit → error `#17` → đợi 5–10 phút
- **Recommend:** dùng System User token (không expire, rate limit cao hơn)
