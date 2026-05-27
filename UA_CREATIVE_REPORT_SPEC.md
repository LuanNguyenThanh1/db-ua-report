# UA Creative Weekly Performance Report — Project Spec

## Overview

A webapp for tracking and reporting weekly UA creative performance across 3 markets: **TW, KR, HK**.

- **Backend**: Google Apps Script
- **Frontend**: HTML boards served via Apps Script `doGet()`
- **Database**: Google Sheets (data extracted via DigiXport from Meta Ads Manager)
- **Update model**: Frontend pulls fresh data on every page load (no polling needed — auto-updates when new rows appended to sheet)

---

## Database: Google Sheets

### Structure
- 3 tabs, one per market: `HK`, `TW`, `KR`
- Data source: DigiXport → Meta Ads Manager → appends rows weekly
- Granularity: **1 row = 1 ad (ad-level data)**

### Key Columns (exact names from DigiXport export)
| Column | Description |
|--------|-------------|
| `DATE` | Date of the data row |
| `Ad Name` | Ad identifier, e.g. `HK.042026C09 - Nimo_sb_note_2` |
| `Amount Spent` | Budget spent (USD) |
| `entrytestresults` | Number of Entry Test Takers (ETR) — primary conversion event |

### Derived Metrics (computed by backend)
- **CPA** = `Amount Spent` / `entrytestresults`
- **Total ETR** = sum of `entrytestresults` per ad
- **Total Spend** = sum of `Amount Spent` per ad
- **Start Date** = earliest `DATE` for that ad
- **Last Active Date** = most recent `DATE` with `Amount Spent > 0`
- **Weeks Inactive** = weeks since Start Date with zero spend

---

## Creative Classification Logic

Each ad is classified into one of 4 tiers based on **lifetime aggregated data**:

| Tier | Label | Condition |
|------|-------|-----------|
| 🏆 Best | `BEST` | Highest spend AND most ETR in the market (top performer) |
| ✅ Good | `GOOD` | Total spend $2–$5 AND has at least 1 ETR |
| ⚪ Neutral | `NEUTRAL` | Total spend < $2, OR zero spend but < 3 inactive weeks |
| ❌ Bad | `BAD` | Zero spend for **3+ consecutive weeks** from ad start date, OR has spend but CPA > $5 |

> **Week definition**: Monday 00:00 → next Monday 00:00 (Mon–Mon)
> **"3 consecutive weeks inactive"**: counted from the ad's `Start Date` (first DATE in sheet), not from last active date.

### Classification Priority (if multiple conditions match)
`BAD` > `BEST` > `GOOD` > `NEUTRAL`

---

## Frontend: HTML Boards

7 boards total, rendered as tabs or sections in a single HTML page served by Apps Script.

### Board 1 — Overview (All Markets)

**Summary cards per market (TW / KR / HK):**
- Total creatives setup this week
- Total spend
- Total ETR
- Average CPA
- Count by tier: BEST / GOOD / NEUTRAL / BAD

**Creative type breakdown** (parsed from Ad Name):
- `_sb_` → Static Banner
- `_ugc_` → UGC Video
- `_hm_` → Homemade Video
- *(naming convention TBD — confirm with user)*

---

### Boards 2–4 — Top 5 Per Market (TW / KR / HK)

One board per market. Shows top 5 creatives ranked by **ETR descending** (ties broken by Spend).

Columns:
| Ad Name | Start Date | Last Report Date | Total Spend | Total ETR | CPA | Tier |

Visual indicator: color-coded row by tier (gold / green / grey / red).

---

### Boards 5–7 — Full Creative List Per Market (TW / KR / HK)

One board per market. Full list of all ads, with **date range filter** (timeframe selector).

**Timeframe filter options:**
- This week (current Mon–Mon)
- Last week
- Last 2 weeks
- Last 4 weeks
- Custom range (date picker)

Columns:
| Ad Name | Creative Type | Start Date | Last Report Date | Total Spend | Total ETR | CPA | Tier |

Sortable by any column. Default sort: Spend descending.

---

## Apps Script Architecture

### Files
```
Code.gs          ← doGet(), main router
DataService.gs   ← reads & aggregates Google Sheet data
Classifier.gs    ← applies BEST/GOOD/NEUTRAL/BAD logic
DateUtils.gs     ← week boundary calculations
index.html       ← main HTML template
```

### Key Functions

```javascript
// Code.gs
doGet(e)  // serves HTML page, passes market + date params

// DataService.gs
getAllAdsData(market, startDate, endDate)  // returns aggregated ad rows
getSummaryByMarket()                       // returns overview stats

// Classifier.gs
classifyAd(adData)  // returns 'BEST' | 'GOOD' | 'NEUTRAL' | 'BAD'

// DateUtils.gs
getCurrentWeekRange()   // returns {start: Date, end: Date} Mon–Mon
getWeekRanges(n)        // returns last n week ranges
```

### Data Flow
1. User opens webapp URL
2. `doGet()` calls `getSummaryByMarket()` + `getAllAdsData()` for each market
3. Data passed as JSON to HTML template via `scriptlets` or `google.script.run`
4. HTML renders boards with data

---

## Build Order (Step by Step)

1. **Step 1** — Confirm Google Sheet access + column names match spec
2. **Step 2** — Build `DataService.gs`: read sheet, aggregate per-ad metrics
3. **Step 3** — Build `Classifier.gs`: apply tier logic
4. **Step 4** — Build `DateUtils.gs`: week boundary logic
5. **Step 5** — Build `doGet()` + JSON data endpoint
6. **Step 6** — Build Board 1 (Overview HTML)
7. **Step 7** — Build Boards 2–4 (Top 5 per market)
8. **Step 8** — Build Boards 5–7 (Full list + filter)
9. **Step 9** — Polish: colors, mobile layout, loading state

---

## Open Questions (Confirm Before Building)

- [ ] Creative type identifier in Ad Name — what substring marks UGC vs Static vs Homemade? (e.g. `_ugc_`, `_sb_`, `_hm_`?)
- [ ] Google Sheet ID / share access for Apps Script
- [ ] Is `BEST` one per market, or top N% of spend+ETR?
