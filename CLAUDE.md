# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

```bash
# Start the local web server (opens browser automatically at http://localhost:8765)
python server.py

# Run the scraper manually to refresh price data
python scraper.py
```

Windows batch shortcuts:
- `starten.bat` — starts the server
- `update_und_öffnen.bat` — runs the scraper then starts the server
- `auto_update.bat` — scraper only

## Deployment

- **Local:** `python server.py` → http://localhost:8765
- **GitHub Pages (static):** https://florianumgelter-eng1.github.io/op14-tracker-web/ — push to `web` remote via `git push web main`
- **GitHub Actions** auto-commits updated `prices_all.json` daily at 22:59 UTC (`.github/workflows/scraper.yml`)

## Architecture

Single-page app, no build step, no framework. One HTML file + one Python backend.

### Backend (`server.py`)

Minimal `HTTPServer` with these endpoints:

| Endpoint | Description |
|---|---|
| `GET /api/prices` | Returns `prices_all.json` |
| `GET /api/update` | Runs scraper **synchronously** (legacy, blocks server) |
| `GET /api/scrape/start` | Starts scraper in background thread, returns immediately |
| `GET /api/scrape/status` | Returns `{ running, progress, current, done, error, data? }` |
| `GET /api/scrape/cancel` | Cancels running scraper |

The async scraper state is held in a module-level `_scrape_state` dict protected by `_scrape_lock`.

### Scraper (`scraper.py`)

- Fetches booster box price history + individual card prices from **PriceCharting.com** for OP-01 through OP-15
- Merges new data into `prices_all.json` without overwriting historical points (`merge_history`)
- Data shape per set: `{ name, url, history: [[timestamp_ms, price_cents], ...], snapshots, cards, recent_sales, available, box_img }`
- Card shape: `{ name, number, price, grade9, psa10, url, img }`

### Frontend (`index.html`)

Single self-contained HTML file — all CSS and JS inline. Design: Apple-inspired light theme (`#F2F2F7` background, `#007AFF` accent, frosted glass header/sidebar via `backdrop-filter`).

**Key JS functions:**

| Function | Purpose |
|---|---|
| `refreshData()` | Starts async scraper (local) or loads JSON (GitHub Pages); polls `/api/scrape/status` every 2s |
| `buildSidebar(data)` | Renders set list; calls `buildSparklines()` after |
| `buildSparklines(data)` | Draws 30-day Chart.js sparklines on sidebar canvases |
| `renderDetail(id)` | Renders main panel for a set (stats, chart, cards tab) |
| `renderCardsTab(sd)` | Builds card table with hover preview, grade picker, inventory buttons |
| `showCardPreview(el, src, name, price)` | Floating hover preview near cursor |
| `showGradePicker(event, card, setId)` | Grade selection popup (Ungraded / PSA 9 / PSA 10) |
| `addToInventory(card, setId, grade)` | Adds card to localStorage inventory with `addedAt` timestamp |
| `renderInventoryList()` | Renders inventory drawer: stats, doughnut chart, card list with P&L |
| `setBuyPrice(k, val)` | Saves purchase price per inventory entry, triggers P&L recalc |
| `exportInventory(format)` | Downloads inventory as CSV or JSON |

**Inventory localStorage schema** (`op_inventory`):
```
key: "{setId}__{cardNumber}__{grade}"   // grade ∈ ungraded | psa9 | psa10
value: { name, number, price, grade, qty, setId, opNum, addedAt, buyPrice? }
```
`price` is set to the grade-specific price at time of adding. `buyPrice` is user-entered purchase price per unit for P&L calculation.

**Charts:** Chart.js 4.4 + date-fns adapter. Main chart: time-series line. Sidebar: static sparklines (no interaction). Inventory: doughnut.

### Data flow

1. `refreshData()` → starts async scrape or loads `prices_all.json` → `applyData()`
2. `applyData()` → `buildSidebar()` + `buildSparklines()` → `renderDetail(activeSet)`
3. Selecting a set → `renderDetail(id)` → `buildChart(pts)`
4. Inventory is entirely client-side (localStorage); scraper/server never touches it
