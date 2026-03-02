# Kiến Trúc Hệ Thống

## Tổng Quan Kiến Trúc

```
┌─────────────────────────────────────────────┐
│     Presentation Layer (HTML/CSS)           │
│  index.html + css/* (8 modules)             │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│     Business Logic Layer (JS Modules)       │
│  js/main.js → handlers, history, print      │
│           → render → state → storage        │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│     Data Layer (localStorage)               │
│  State Persistence, Tare DB, History        │
└─────────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│     Offline Layer (Service Worker)          │
│  Cache-first, all js/* + css/* cached       │
└─────────────────────────────────────────────┘
```

## Module Architecture (js/)

```
main.js (entry point)
   ├── handlers.js     — events, row management, date picker
   │   ├── state.js    — app state, loadData, autoSave
   │   │   ├── storage.js  — safeSave, getHistory, tare DB
   │   │   │   ├── constants.js
   │   │   │   └── utils.js
   │   │   └── constants.js
   │   ├── render.js   — DOM rendering, duplicate check
   │   │   ├── state.js
   │   │   └── utils.js
   │   ├── history.js  — history modal UI, archiveCurrentForm
   │   │   ├── state.js, storage.js, utils.js, render.js
   │   └── utils.js, storage.js
   ├── history.js
   ├── print.js        — generatePrintView, exportCSV
   │   ├── state.js
   │   └── utils.js
   └── render.js
```

**Không có circular dependency.**

## State Container

```javascript
// js/state.js — exported const, mutate properties
const state = {
  date: "2026-03-02",              // YYYY-MM-DD
  cylinders: [
    { seri: "ABC123", total: "53.3", tare: "10.5" },
    { seri: "", total: "", tare: "" },
    // ... up to DEFAULT_ROWS (40) hoặc nhiều hơn nếu addRow()
  ]
}
```

State được export như `const` object — các module khác mutate **properties** (không reassign biến), giúp ES module live binding hoạt động đúng.

## Event Flow Chi Tiết

### Initialization
```
DOMContentLoaded [main.js]
   ├── loadData()           — restore state từ localStorage
   ├── initDatePicker()     — populate 3 dropdown, restore date
   ├── renderAllCards()     — render tất cả cylinder cards
   ├── updateSummary()      — update sticky bar
   ├── initIOSKeyboardDismiss()
   └── registerServiceWorker()
```

### Input → Save
```
User types → oninput="onFieldInput(index, field, value)" [HTML]
   ↓
window.onFieldInput() → onFieldInput() [handlers.js]
   ├── state.cylinders[index][field] = value
   ├── updateCardResult(index)    [render.js]
   ├── updateSummary()            [render.js]
   ├── autoSave()                 [state.js] — debounce 800ms
   ├── (if tare) saveTareWeight() [storage.js]
   └── (if seri) autoFillTare() + updateDuplicateWarnings()
```

### Print/Export
```
onclick="printForm()" → window.printForm() [main.js]
   → printForm() [print.js]
       ├── generatePrintView() — render HTML 2 cột vào #printView
       └── window.print()      — browser print dialog

onclick="exportCSV()" → exportCSV() [print.js]
   ├── Build CSV string với BOM (\uFEFF)
   ├── Blob → Object URL
   └── <a>.click() → download
```

### History Modal
```
showHistory() [history.js]
   ├── getHistory() from localStorage [storage.js]
   ├── Render danh sách HTML vào #historyList
   └── modal.classList.add('show')

loadHistory(index) [history.js]
   ├── archiveCurrentForm() — lưu phiếu hiện tại nếu có data
   ├── state.date = item.date
   ├── state.cylinders = deep copy of item.cylinders
   ├── Update date picker UI
   ├── renderAllCards() + updateSummary()
   └── saveData() + closeHistory()
```

## Data Layer - localStorage

**3 keys:**

```javascript
// 1. State hiện tại
localStorage['phieu-can-gas-data'] = {
  date: "2026-03-02",
  cylinders: [{ seri, total, tare }]
}

// 2. Tare weight memory theo seri
localStorage['phieu-can-gas-tare-db'] = {
  "ABC123": "10.5",
  "DEF456": "12"
}

// 3. Lịch sử phiếu (max 50, newest first)
localStorage['phieu-can-gas-history'] = [
  { id, date, cylinders, totalGas, filledCount, savedAt }
]
```

**Quota Handling (safeSave):**
```
Try localStorage.setItem(key, value)
  ├── OK → done
  └── QuotaExceeded
        ├── history.pop() — xóa phiếu cũ nhất
        ├── Save history mới
        ├── Toast "Bộ nhớ đầy"
        └── Retry setItem
```

## CSS Architecture (css/)

| File | Concern | LOC |
|------|---------|-----|
| `variables.css` | Design tokens (:root) | 45 |
| `base.css` | Reset, body, header, summary | 87 |
| `form.css` | Date picker, formula hint | 65 |
| `components.css` | Cards, inputs, buttons | 152 |
| `history.css` | Modal, expandable detail | 145 |
| `toast.css` | Toast notification | 22 |
| `responsive.css` | Breakpoints 768/1024/1440px | 60 |
| `print.css` | @media print | 46 |

Load order trong `index.html`: variables → base → form → components → history → toast → responsive → print

## Service Worker (sw.js)

**Strategy: Cache-First**

```
Request → Cache hit? → Return cached
                ↓ miss
          Fetch network → Return
```

**Cache version:** `phieu-can-gas-v3`

**Cached assets:** index.html, manifest.json, assets/*, js/* (9 files), css/* (8 files)

Khi deploy code mới → bump `CACHE_NAME` version → service worker activate xóa cache cũ.

## Inline Handler Pattern

Vì HTML dùng `onclick="addRow()"` trong string templates, các functions cần expose lên `window`:

```javascript
// js/main.js
window.addRow = addRow;
window.onFieldInput = onFieldInput;
// ... tất cả public functions
```

Đây là trade-off để giữ HTML template đơn giản mà không cần event delegation phức tạp.

## Scalability Considerations

### Hiện tại
- Max 50 history entries (configurable `MAX_HISTORY`)
- Default 40 bình (configurable `DEFAULT_ROWS`)
- ~5MB localStorage limit

### Nếu cần scale
1. **Nhiều user, nhiều thiết bị** → Backend API + JWT auth + sync
2. **Nhiều data hơn** → IndexedDB thay localStorage
3. **Feature phức tạp hơn** → Consider framework (Vue/React)
4. **Audit trail** → Server-side logging

## Technology Decisions

| Quyết định | Lý do |
|-----------|-------|
| Vanilla JS, no framework | Không overhead, đơn giản, offline-first |
| Native ES Modules | Tách module không cần build step |
| localStorage | Browser built-in, không cần server |
| Cache-first SW | Offline ngay lập tức, predictable |
| CSS custom properties | Theming dễ, không cần preprocessor |
| No build step | Deploy trực tiếp, không cần Node.js |

---

**Phiên Bản**: 1.1 | **Cập Nhật**: 02/03/2026
