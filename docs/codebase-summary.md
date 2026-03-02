# Tóm Tắt Codebase

## Cấu Trúc Tệp

```
petrolimex-calculator/
├── index.html              # HTML structure (90 LOC)
├── sw.js                   # Service Worker (46 LOC)
├── manifest.json           # PWA manifest (14 LOC)
├── README.md
├── assets/
│   ├── icon-180.png        # Apple touch icon
│   ├── icon-192.png        # Android home screen icon
│   └── icon-512.png        # Splash screen icon
├── js/                     # ES Modules (native, no build)
│   ├── constants.js        # App-wide constants (6 LOC)
│   ├── utils.js            # Pure functions + showToast (47 LOC)
│   ├── storage.js          # localStorage operations (44 LOC)
│   ├── state.js            # App state + persistence (38 LOC)
│   ├── render.js           # DOM rendering (103 LOC)
│   ├── history.js          # History modal UI (110 LOC)
│   ├── print.js            # Print + CSV export (80 LOC)
│   ├── handlers.js         # Event handlers + row management (115 LOC)
│   └── main.js             # Entry point + window exports (37 LOC)
├── css/                    # Modular CSS
│   ├── variables.css       # CSS custom properties (45 LOC)
│   ├── base.css            # Reset, body, header, summary bar (87 LOC)
│   ├── form.css            # Date picker, formula hint (65 LOC)
│   ├── components.css      # Cards, inputs, buttons, auto-save (152 LOC)
│   ├── history.css         # History modal (145 LOC)
│   ├── toast.css           # Toast + print-view hidden (22 LOC)
│   ├── responsive.css      # Media queries (60 LOC)
│   └── print.css           # @media print (46 LOC)
└── docs/                   # Tài liệu
```

**Tổng LOC JS**: ~580 (9 modules) | **Tổng LOC CSS**: ~622 (8 files)
**Phụ Thuộc**: 0 | **Framework**: Vanilla JS | **Build**: Không cần

## Module Dependencies (js/)

```
constants.js ← storage.js, state.js
utils.js     ← storage.js, render.js, history.js, print.js, handlers.js
storage.js   ← state.js, history.js, handlers.js
state.js     ← render.js, history.js, handlers.js, print.js, main.js
render.js    ← history.js, handlers.js, main.js
history.js   ← handlers.js, main.js
print.js     ← main.js
handlers.js  ← main.js
```

Không có circular dependency.

## Mô Tả Từng Module

### js/constants.js
App-wide constants:
- `STORAGE_KEY`, `TARE_DB_KEY`, `HISTORY_KEY` — localStorage keys
- `DEFAULT_ROWS` = 40, `MAX_HISTORY` = 50

### js/utils.js
Pure utility functions:
- `parseNum(v)` — parse số, hỗ trợ dấu phẩy/chấm
- `calcGas(c)` — gas = total - tare (round 2 decimals)
- `escapeHTML(str)` — XSS prevention
- `formatDateVN(dateStr)` — "2026-03-02" → "2/3/2026"
- `formatTimeVN(isoStr)` — ISO → "HH:mm"
- `showToast(message)` — Toast notification 2.5s
- `showConfirm(message)` — Promise-based custom confirm modal (thay thế `window.confirm()`)

### js/storage.js
Tất cả localStorage operations:
- `safeSave(key, value)` — save với quota fallback (xóa history cũ)
- `getHistory()`, `saveHistory(history)` — đọc/ghi history array
- `saveTareWeight(seri, tare)`, `getTareWeight(seri)` — tare DB

### js/state.js
App state và persistence:
- `state` — `{ date, cylinders: [{ seri, total, tare }] }` (exported const, mutate properties)
- `createEmptyRows(count)` — tạo N dòng trống
- `loadData()` — restore từ localStorage
- `saveData()`, `autoSave()` — lưu state (debounce 800ms)

### js/render.js
Tất cả DOM rendering:
- `checkDuplicate(seri, index)` — check seri trùng
- `updateDuplicateWarnings()` — update warning tất cả cards
- `createCard(index)`, `buildCardHTML(...)` — tạo card element
- `renderAllCards()` — render lại toàn bộ danh sách
- `updateCardResult(index)` — update kết quả gas 1 card
- `updateSummary()` — update sticky summary bar

### js/history.js
History modal UI:
- `archiveCurrentForm()` — lưu phiếu hiện tại vào history
- `showHistory()` — mở modal, render danh sách
- `toggleHistoryDetail(index)` — expand/collapse chi tiết
- `closeHistory()` — đóng modal
- `loadHistory(index)` — async, tải phiếu cũ vào state (confirm bằng `showConfirm`)
- `deleteHistory(index)` — async, xóa phiếu khỏi history (confirm bằng `showConfirm`)

### js/print.js
In và xuất dữ liệu:
- `generatePrintView()` — render HTML bảng 2 cột cho in
- `printForm()` — gọi window.print()
- `exportCSV()` — xuất CSV UTF-8 BOM

### js/handlers.js
Event handlers và quản lý bình:
- `onFieldInput(index, field, value)` — xử lý input thay đổi
- `onEnterKey(event, index, field)` — Enter key navigation
- `autoFillTare(index, seri)` — tự điền tare từ DB
- `addRow()`, `removeRow(index)`, `clearAll()` — quản lý bình
- `initDatePicker()` — setup 3 dropdown ngày/tháng/năm
- `initIOSKeyboardDismiss()` — touch listener dismiss keyboard

### js/main.js
Entry point:
- Import và expose tất cả public functions lên `window.*` cho inline HTML handlers
- `registerServiceWorker()`
- `initOfflineIndicator()` — lắng nghe sự kiện `online`/`offline`, hiển thị offline bar
- `DOMContentLoaded` init sequence

## Data Flow

```
User Input (seri/total/tare)
       ↓
onFieldInput() [handlers.js]
       ↓
updateCardResult() + updateSummary() [render.js]
       ↓
autoSave() debounce 800ms [state.js]
       ↓
safeSave() [storage.js]
       ↓
localStorage.setItem()
```

## localStorage Schema

```js
// phieu-can-gas-data — state hiện tại
{ date: "2026-03-02", cylinders: [{ seri, total, tare }] }

// phieu-can-gas-tare-db — tare memory theo seri
{ "ABC123": "10.5", "DEF456": "12" }

// phieu-can-gas-history — lịch sử phiếu (max 50)
[{ id, date, cylinders, totalGas, filledCount, savedAt }]
```

## CSS Modules

| File | Nội dung |
|------|---------|
| `variables.css` | :root CSS custom properties, design tokens |
| `base.css` | Reset, body, header, summary bar, offline bar |
| `form.css` | Date picker, formula hint |
| `components.css` | Cylinder cards, inputs, buttons, auto-save, gas-negative warning |
| `confirm.css` | Custom confirm modal (thay thế `window.confirm()`) |
| `history.css` | History modal, detail expand |
| `toast.css` | Toast notification |
| `responsive.css` | @media 768px, 1024px, 1440px |
| `print.css` | @media print |

## Browser Support

- iOS 12+ (Safari — hỗ trợ ES modules)
- Android Chrome 61+
- Desktop Chrome/Firefox/Edge hiện đại

---

**Phiên Bản**: 1.2 | **Cập Nhật**: 02/03/2026
