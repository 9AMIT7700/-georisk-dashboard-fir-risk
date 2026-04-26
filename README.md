# GeoRisk Intelligence Dashboard
### Global Macro Trading Signal System

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        UI LAYER                             │
│              Streamlit + Plotly (Dark Sovereign Theme)      │
│   Gauge │ Decomposition │ Market Tiles │ News │ Alerts      │
└───────────────────┬─────────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────┐
│                  PROCESSING LAYER                           │
│                                                             │
│  Risk Engine          Market Mapper         Alert System    │
│  ─────────────        ────────────          ────────────    │
│  compute_risk_score() get_asset_impact()    generate_alerts │
│  score NLP keywords   7 asset classes       threshold logic │
│  5-factor composite   directional signals   regime signals  │
└───────────────────┬─────────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────┐
│                    DATA LAYER                               │
│                                                             │
│  yfinance (5-min cache)    feedparser (10-min cache)        │
│  ────────────────────      ─────────────────────────        │
│  Oil / WTI  → CL=F         BBC World News                   │
│  Gold       → GC=F         Reuters World                    │
│  US 10Y     → ^TNX         Al Jazeera                       │
│  VIX        → ^VIX         NYT World                        │
│  DXY        → DX-Y.NYB                                      │
│  Bitcoin    → BTC-USD                                        │
│  S&P 500    → ^GSPC                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Risk Score Methodology

**Total Score: 0–100**

| Component            | Max Pts | Trigger                                  |
|---------------------|---------|------------------------------------------|
| VIX Fear Index      | 30      | VIX > 10 adds points linearly to 55      |
| Oil Price Shock     | 15      | 5-day WTI return > 0% (supply disruption)|
| Gold Safe-Haven     | 15      | 5-day Gold return > 0% (crisis demand)   |
| USD Flight-to-Safety| 10      | 5-day DXY return > 0% (capital flight)   |
| News Sentiment NLP  | 30      | Keyword density across 4 RSS feeds       |

**Regime Thresholds:**
- `0–29`   → LOW RISK / RISK-ON (Green)
- `30–49`  → MODERATE RISK (Gold)
- `50–69`  → ELEVATED RISK (Amber)
- `70–100` → EXTREME RISK-OFF (Red)

---

## Market Impact Logic

```
Risk Score → Asset Class Direction

HIGH (≥70):  Equities ↓  Gold ↑  Oil ↑(geopolitical bid)  Bonds ↑  USD ↑  BTC ↓  EM ↓
MID  (50-70): Equities →  Gold ↑  Oil →                    Bonds →  USD ↑  BTC →  EM ↓
LOW  (30-50): Equities →  Gold →  Oil →                    Bonds →  USD →  BTC →  EM →
OFF  (<30):   Equities ↑  Gold ↓  Oil →(demand)            Bonds ↓  USD ↓  BTC ↑  EM ↑
```

---

## NLP Keyword Categories

**HIGH RISK** (0.35 pts/hit): war, invasion, airstrike, nuclear, missile, blockade,
coup, genocide, escalation, drone strike, strait, red sea...

**MEDIUM RISK** (0.15 pts/hit): sanction, tariff, trade war, tension, unrest,
embargo, retaliation, expulsion...

**NEGATIVE RISK** (-0.08 pts/hit): ceasefire, peace deal, agreement, diplomacy,
de-escalate, truce...

---

## Quick Start

```bash
# 1. Clone and install
pip install -r requirements.txt

# 2. Run locally
streamlit run app.py

# 3. Access at:
http://localhost:8501
```

---

## Deployment Options

### Option 1: Local (Recommended for Trading Use)
```bash
streamlit run app.py --server.port 8501 --server.headless false
```

### Option 2: Streamlit Community Cloud (Free)
1. Push to GitHub
2. Visit share.streamlit.io → Deploy
3. Set repo/branch/app.py

### Option 3: Railway.app (Always-On)
```bash
railway init
railway up
# Set start command: streamlit run app.py --server.port $PORT --server.address 0.0.0.0
```

### Option 4: Docker (Self-Hosted)
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.address", "0.0.0.0"]
```
```bash
docker build -t georisk .
docker run -p 8501:8501 georisk
```

### Option 5: Cron / Scheduled (Alert Logging)
```python
# Run as a background process with auto-refresh
# Enable the "Auto (5min)" checkbox in the dashboard UI
# Or schedule externally with cron:
# */5 * * * * curl -s http://localhost:8501 > /dev/null
```

---

## Extending the System

### Add More Data Sources
```python
# In MARKET_TICKERS dict:
"Natural Gas": "NG=F",
"Silver":      "SI=F",
"EUR/USD":     "EURUSD=X",
"USD/INR":     "USDINR=X",
"Nifty 50":    "^NSEI",

# In RSS_FEEDS list:
("FT Markets", "https://www.ft.com/rss/home/uk"),
```

### Add Telegram/Email Alerts
```python
import requests

def send_telegram_alert(score, regime, bot_token, chat_id):
    if score >= 70:
        msg = f"🚨 GEORISK ALERT\nScore: {score:.1f}\nRegime: {regime}"
        requests.post(
            f"https://api.telegram.org/bot{bot_token}/sendMessage",
            json={"chat_id": chat_id, "text": msg}
        )
```

### Persist Historical Scores (SQLite)
```python
import sqlite3

def save_score(score, components):
    conn = sqlite3.connect("georisk.db")
    conn.execute("""
        CREATE TABLE IF NOT EXISTS scores
        (ts TEXT, score REAL, vix REAL, oil REAL, gold REAL, news REAL)
    """)
    conn.execute("INSERT INTO scores VALUES (?,?,?,?,?,?)",
        (datetime.now().isoformat(), score,
         components['VIX Fear Index'], components['Oil Price Shock'],
         components['Gold Safe-Haven'], components['News Sentiment']))
    conn.commit()
    conn.close()
```

---

## Variables That Actually Move Markets

The dashboard deliberately tracks only the **5 highest signal-to-noise macro variables:**

1. **VIX** — The single best real-time fear gauge. Encodes options market's probability-weighted panic.
2. **WTI Oil** — Geopolitical chokepoint proxy (Hormuz, Red Sea, Libya, Iraq). Spike = supply shock.
3. **Gold** — 5,000-year safe-haven. Central banks + institutions front-run crises here.
4. **DXY** — Global capital flight metric. Rising DXY = EM stress, USD shortage.
5. **News NLP** — Keyword-weighted RSS scraping catches conflicts/sanctions before markets fully price.

Everything else (bonds, crypto, equities) is a **consequence** of these 5 inputs.

---

*Not financial advice. Research use only.*
