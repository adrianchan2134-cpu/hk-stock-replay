# HK Stock Trading Replay Tool

**Give this file + the GitHub repo to any AI (ChatGPT, Claude, WorkBuddy, etc.) and ask it to duplicate.**

---

## What It Is

A mobile-first web app for blind replay trading of Hong Kong stocks. You see past K-line data day by day without knowing the future. Buy/sell at close prices with simulation tracking (¥100,000 initial, 0.1% fee per trade, full position only).

- **1,854 HK stocks** with up to 10 years of daily OHLCV data
- **Live**: https://adrianchan2134-cpu.github.io/hk-stock-replay/
- **Repo**: https://github.com/adrianchan2134-cpu/hk-stock-replay

---

## Architecture

```
build_online.py          # Python build script → generates deploy/index.html
deploy/index.html        # Single-page app (HTML+CSS+JS inline)
deploy/XXXX.json         # 1,854 stock data files (one per code)
deploy/stocks_index.json # Stock metadata index
```

`build_online.py` reads `stocks_index.json` and pre-embeds `0700.json` as default data, then injects the full stock list into a single self-contained HTML file.

Stock data files are plain JSON arrays:
```json
[{"date":"2016-07-13","open":162.5,"high":164.3,"low":161.2,"close":163.8,"volume":8456123,"pct":1.25,"vma10":7456234,"vma20":7123456}, ...]
```

---

## How To Replicate (Step by Step)

### Step 1: Get Stock Data

Use **Yahoo Finance** via `yfinance` (Python). For each HK stock code (4 digits):

```python
import yfinance as yf
import json

ticker = yf.Ticker("0700.HK")
df = ticker.history(period="10y", interval="1d", auto_adjust=True)

data = []
prev = None
for idx, row in df.iterrows():
    c = float(row["Close"])
    pct = round((c - prev) / prev * 100, 2) if prev else 0
    prev = c
    data.append({
        "date": idx.strftime("%Y-%m-%d"),
        "open": float(row["Open"]), "high": float(row["High"]),
        "low": float(row["Low"]), "close": c,
        "volume": int(row["Volume"])
    })

# Compute 10-day and 20-day volume moving averages
for i in range(len(data)):
    if i >= 9:  data[i]["vma10"] = round(sum(data[j]["volume"] for j in range(i-9, i+1)) / 10)
    if i >= 19: data[i]["vma20"] = round(sum(data[j]["volume"] for j in range(i-19, i+1)) / 20)

with open("0700.json", "w") as f:
    json.dump(data, f)
```

### Step 2: Build the HTML

The build script (`build_online.py`) does:
1. Build a `STOCK_LIST` JS array from `stocks_index.json`
2. Pre-embed one stock's data as default
3. Inject everything into a single HTML template

Key parts of the JS:
- **Candlestick chart** drawn on `<canvas>` with GPU-accelerated rendering
- **Volume bars** + VMA10/VMA20 moving average volume lines
- **Lowest price marker** — red dot at visible range's low
- **Cursor highlight** — subtle blue column behind current candle (no vertical line)
- **Blind replay mode** — `maxCursor` variable locks future data until you press +1d/+5d etc.
- **Portfolio simulation** — ¥100,000 initial, full buy/sell with 0.1% per-side fee
- **On-demand stock loading** — fetches `XXXX.json` from same directory on code change
- **localStorage cache** — last 10 stocks cached for offline reuse
- **Pinch-to-zoom** — mobile gesture for candle range (no UI buttons)
- **Stock search** — fuzzy match on code or name, dropdown selector
- **Random stock button** — 🎲 picks a random stock

### Step 3: Deploy

**GitHub Pages** (free, permanent):
```bash
# Push everything in deploy/ to a GitHub repo
cd deploy
git init
git add -A
git commit -m "HK stock replay"
git remote add origin https://github.com/YOUR_USER/hk-stock-replay.git
git push -u origin master
# Then enable Pages in repo Settings → Pages → branch: master, folder: / (root)
```

---

## Key Design Decisions

| Decision | Why |
|----------|-----|
| No MA on price chart | User preference — clean candles only |
| Green = up, red = down | Western color convention |
| VMA10 (blue) / VMA20 (red dashed) | Volume moving averages, independently scaled from volume bars |
| No vertical cursor line | Replaced by subtle column highlight — doesn't cover candles |
| Pinch-to-zoom, no zoom buttons | Saves mobile screen space |
| `safe-area-inset-top` CSS | iOS notch/status-bar safe zone |
| Single HTML file | No server needed, one file + JSON data |

---

## Files Your AI Needs

All under `deploy/`:
- `index.html` — the app (single file, ~440KB)
- `stocks_index.json` — stock metadata
- `XXXX.json` — one per stock (1,854 files, ~400KB each, ~675MB total)

Or point the AI to the GitHub repo for the full source + build scripts.

---

## Common Fixes Already Applied

- **Wrapper format bug**: Stock JSON files are plain arrays, not `{"data": [...]}` objects
- **VMA invisible**: `computeIndicators(DATA)` must run on default embedded data at init
- **VMA squished by volume outliers**: VMAs use independent `maxVMA` scale, not `maxVol`
- **Volume bars too small**: Minimum 0.8px height + 0.5px width
- **NaN on new stock load**: Defensive null/undefined checks in draw(), updateChartInfo, all loops
- **Mobile scroll peeking**: `maxCursor` variable blocks slider/swipe past unlocked position
