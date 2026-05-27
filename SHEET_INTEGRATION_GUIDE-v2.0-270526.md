# Google Sheet → Dashboard Integration Guide
> Nối order tracking sheet vào creative dashboard để tính Orders, Revenue, ROAS by creative

---

## Vấn đề gốc

Browser không fetch trực tiếp được Google Sheets — bị CORS block:
```
Access to fetch at 'https://docs.google.com/spreadsheets/d/…/gviz/tq?…'
from origin 'null' has been blocked by CORS policy
```

gviz endpoint không trả `Access-Control-Allow-Origin` header → browser chặn.

---

## Solution: Google Apps Script Web App

Apps Script chạy phía server Google → đọc sheet → trả JSON có CORS headers → browser fetch được.

---

## Cấu trúc sheet order tracking

| Cột | Tên header | Ghi chú |
|-----|-----------|---------|
| A | `Date` | Format `DD/MM/YYYY` — Apps Script tự convert sang `YYYY-MM-DD` |
| B | `UTM Content` | vd: `tw.122025c92` (lowercase) |
| C | `Source` | Phải là `Meta` (case-insensitive) |
| D | `Order` | Số lượng (mỗi row = 1 order) |
| E | `Product` | Tên sản phẩm kèm giá `(1399)` |
| F | `Revenue` (TWD) | Giá local currency |
| G | `Revenue` (USD) | **Cột này dùng** — dashboard lấy cột revenue cuối cùng |

> Sheet ID: `12bfNtjPYc8i4WaxW18kFVAVCN6VJIDSjG5qdNQ84Nhg` · GID: `1250375883`

---

## Apps Script Code

Tạo tại [script.google.com](https://script.google.com) → New project → paste:

```javascript
const SHEET_ID  = '12bfNtjPYc8i4WaxW18kFVAVCN6VJIDSjG5qdNQ84Nhg';
const SHEET_GID = '1250375883';

function doGet(e) {
  const from = e.parameter.from || null;
  const to   = e.parameter.to   || null;
  return ContentService
    .createTextOutput(JSON.stringify(buildSheetMap(from, to)))
    .setMimeType(ContentService.MimeType.JSON);
}

function toISODate(val) {
  if (!val) return '';
  if (val instanceof Date) return Utilities.formatDate(val, 'Asia/Taipei', 'yyyy-MM-dd');
  const s = String(val).trim();
  const m = s.match(/^(\d{1,2})\/(\d{1,2})\/(\d{4})$/);
  if (m) return `${m[3]}-${m[2].padStart(2,'0')}-${m[1].padStart(2,'0')}`;
  return s.slice(0, 10);
}

function buildSheetMap(from, to) {
  const ss    = SpreadsheetApp.openById(SHEET_ID);
  const sheet = ss.getSheets().find(s => s.getSheetId() == SHEET_GID)
             || ss.getSheets()[0];
  const data  = sheet.getDataRange().getValues();
  const hdr   = data[0].map(h => String(h).toLowerCase().trim());

  const iSrc  = hdr.findIndex(h => h === 'source');
  const iUtm  = hdr.findIndex(h => h.replace(/[\s_]+/g, '') === 'utmcontent');
  const iDate = hdr.findIndex(h => h === 'date');
  let iRev = -1;
  hdr.forEach((h, i) => { if (h === 'revenue') iRev = i; }); // lấy cột revenue CUỐI (USD)

  const map = {};
  for (let i = 1; i < data.length; i++) {
    const row    = data[i];
    const source = String(row[iSrc] || '').toLowerCase().trim();
    if (source !== 'meta') continue;

    const utm = String(row[iUtm] || '').toLowerCase().trim();
    if (!utm || utm === '-' || utm === 'n/a' || utm === '') continue;

    if (iDate >= 0 && (from || to)) {
      const rowDate = toISODate(row[iDate]);
      if (from && rowDate < from) continue;
      if (to   && rowDate > to)   continue;
    }

    const rev = parseFloat(row[iRev]) || 0;
    if (!map[utm]) map[utm] = { orders: 0, revenue: 0 };
    map[utm].orders++;
    map[utm].revenue += rev;
  }
  return map;
}
```

---

## Deploy Apps Script

1. **Deploy → New deployment**
2. Type: **Web app**
3. Execute as: **Me**
4. Who has access: **Anyone**
5. Click Deploy → copy URL dạng `.../exec`

> ⚠️ Mỗi lần sửa code phải **New deployment** (không dùng lại deployment cũ) → URL mới.

---

## Paste vào Dashboard

Field "Apps Script URL (orders)" → paste URL `/exec` → **Fetch Data**.

Dashboard tự truyền `?from=YYYY-MM-DD&to=YYYY-MM-DD` → Apps Script filter theo date range.

---

## Matching logic: UTM → Ad name

```
sheet utm_content:  "tw.122025c92"          (lowercase)
ad name từ Meta:    "TW.122025C92 - Title…" (uppercase)
```

Dashboard dùng `adName.toLowerCase().startsWith(utmKey)` để match.

---

## Gotchas đã gặp

| Triệu chứng | Nguyên nhân | Fix |
|-------------|-------------|-----|
| Orders/Revenue trống | CORS block gviz endpoint | Dùng Apps Script Web App |
| sheetMap = {} | Header `utm content` (space) thay vì `utm_content` | Normalize: `.replace(/[\s_]+/g, '')` |
| Tất cả rows bị filter | Date format sheet `DD/MM/YYYY` ≠ `YYYY-MM-DD` | Hàm `toISODate()` convert |
| Revenue sai (quá lớn) | Lấy cột Revenue TWD thay vì USD | Dùng `findLastIndex` → lấy cột revenue cuối |
| ROAS vô nghĩa (3412x) | TWD ÷ USD spend | Fix revenue → dùng cột USD |
| Orders/Revenue all-time | Browser cache code cũ | Cmd+Shift+R hard reload |

---

## Update sheet hàng tuần

Thêm row mới vào sheet → click **Fetch Data** trên dashboard là xong. Không cần redeploy Apps Script.
