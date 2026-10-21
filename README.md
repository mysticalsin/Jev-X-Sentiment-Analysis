# Jev X Sentiment Analysis

![Jev X Sentiment Analysis](public/images/frontend.jpeg)

An on-demand crypto market intelligence and decision-support terminal powered by TypeSafe AI's System One model (Jev).

The tool allows you to search any cryptocurrency (such as BTC, SOL, or ETH), select how many tweets you want to analyze (from 50 up to 1,000 tweets), and receive an instant, data-backed trading decision (Buy, Sell, Hold, or Take Profit) based on real-time market data, perpetuals funding rates, and social sentiment.

The platform does not execute trades automatically. It generates a clear decision card with calculated entry ranges, stop losses, and target levels so you can review the reasoning and execute manually on whichever exchange or DEX you prefer.

---

## How Tweets Are Analyzed

When you request sentiment analysis across 50 to 1,000 tweets, dumping hundreds of raw tweets into an LLM would exceed token budgets and introduce latency. Instead, Jev X Sentiment Analysis uses an **intelligent three-stage pipeline**:

```text
User Search (e.g. "SOL", 500 tweets)
   │
   ├──> CCXT: Live Price, 24h Volume, RSI, Funding Rate, Open Interest
   └──> TwitterAPI.io: 500 tweets ingested via cursor pagination
            │
            ▼
   Tier 1: Python Statistical Pre-Processing
   - Computes total engagement velocity (likes, retweets per minute)
   - Measures author diversity ratio (detects bot farms vs organic retail)
   - Computes keyword sentiment polarity (fear/capitulation vs greed/hype)
   - Stratified extraction:
       • Top 25 highest-engaged tweets (KOL & market-moving opinions)
       • 25 most recent breaking tweets (current real-time narrative)
            │
            ▼
   Tier 2: Early-Stopping Database Deduplication (API Cost Optimization)
   - To save API credits, the system stores all ingested tweets in a local SQLite database (`data/market_intel.db`).
   - TwitterAPI.io returns tweets in reverse-chronological order (`queryType="Latest"`).
   - During pagination, **as soon as a returned tweet ID already exists in the local database, the pagination loop halts immediately**.
   - Any remaining tweets required to fulfill your requested sample size (e.g., 500 tweets) are loaded directly from the local database.
   - **Result**: On repeated or intraday searches, you only pay for the few brand-new tweets posted since your last search (often 1 page call = ~$0.006) instead of re-fetching hundreds of tweets you already have.
            │
            ▼
   Tier 3: TypeSafe Jev System One Evaluation (`typesafe-sdk`)
   Evaluates 4 typed questions concurrently on the combined state:
     1. Trade Action (Choice: Strong Buy, Buy, Hold, Take Profit, Sell, Strong Sell)
     2. Sentiment Spectrum (Score: Extreme Panic to Euphoria)
     3. Squeeze Risk (Noul: probability that negative funding + panic indicates a short squeeze)
     4. Catalyst Significance (Score: None, Minor, Moderate, Major)
            │
            ▼
   Decision Card Displayed in Web Terminal
   - Recommended action with calibrated confidence percentage
   - Macro sentiment gauge across all 500 tweets
   - Calculated entry range, stop loss, and target levels
   - User executes manually on their exchange of choice
```

---

## Configurable Social Fetching & Cost Breakdown

You can configure the tweet sample size in the terminal interface based on your needs:

| Sample Size | TwitterAPI.io Cost | TypeSafe Jev Cost | Total Cost per Search | Best For |
| :--- | :--- | :--- | :--- | :--- |
| **50 Tweets** | ~$0.0075 | ~$0.0008 | **< $0.009 (<1¢)** | Quick pulse check on immediate price moves |
| **100 Tweets** | ~$0.0150 | ~$0.0008 | **~$0.016 (1.6¢)** | Standard intraday trading check |
| **250 Tweets** | ~$0.0375 | ~$0.0008 | **~$0.038 (3.8¢)** | Multi-hour swing setup validation |
| **500 Tweets** | ~$0.0750 | ~$0.0008 | **~$0.076 (7.6¢)** | Comprehensive sentiment & news audit |
| **1,000 Tweets** | ~$0.1500 | ~$0.0008 | **~$0.151 (15¢)** | Major regime shift or ETF/catalyst investigation |

*Note: Repeated searches for the same asset within a 10-minute window hit the local cache and cost $0.00.*

---

## Technical Architecture

```text
.
├── app/
│   ├── api/
│   │   └── v1/
│   │       └── analyze.py        # Search and settings endpoints
│   ├── core/
│   │   ├── config.py             # App configuration and environment variables
│   │   ├── cache.py              # In-memory TTL caches
│   │   └── database.py           # Local SQLite tweet storage and retrieval
│   ├── services/
│   │   ├── market_service.py     # Exchange data client (CCXT Kraken/Kraken Futures)
│   │   ├── twitter_service.py    # TwitterAPI.io client with cursor pagination
│   │   ├── stats_service.py      # Tier 1 deterministic statistical pre-processing
│   │   └── typesafe_service.py   # TypeSafe Jev System One client and decision logic
│   ├── static/
│   │   ├── css/
│   │   │   └── style.css         # Dark quantitative terminal styling
│   │   └── js/
│   │       └── app.js            # Frontend controls and result rendering
│   ├── templates/
│   │   └── index.html            # Main web terminal interface
│   └── main.py                   # FastAPI application entrypoint
├── requirements.txt
└── README.md
```

---

## Setup and Installation

### Prerequisites
- Python 3.12 or higher
- A TypeSafe AI API key (from [console.typesafe.ai](https://console.typesafe.ai))
- A TwitterAPI.io API key (from [twitterapi.io](https://twitterapi.io))

### 1. Clone and Install Dependencies

```bash
git clone <repo-url>
cd Jev-X-Sentiment-Analysis

python -m venv venv
source venv/bin/activate

pip install -r requirements.txt
```

### 2. Configure Environment Variables

Create a `.env` file in the root directory:

```env
TYPESAFE_API_KEY=your_typesafe_api_key_here
TWITTER_API_KEY=your_twitterapi_io_key_here

# Optional configuration
PORT=8000
HOST=127.0.0.1
CACHE_TTL_SECONDS=600

# Security & Access Control
ALLOWED_ORIGINS=http://localhost:8000,http://127.0.0.1:8000
# ADMIN_TOKEN=your_secret_admin_token_here
```

### 3. Run the Application

```bash
uvicorn app.main:app --reload --port 8000
```

Open your browser and navigate to:
```text
http://localhost:8000
```

### 4. Running the Automated Test Suite

```bash
pytest -v
```

---

## Usage Workflow

1. **Enter a Symbol**: Type any supported cryptocurrency symbol (e.g. `BTC`, `SOL`, `ETH`).
2. **Select Sample Size**: Choose between 50, 100, 250, 500, or 1,000 tweets using the sample slider.
3. **Review Market & Derivatives Data**: View live spot price, 24-hour volume, perpetuals funding rate, and open interest delta (powered by Kraken & Kraken Futures).
4. **Inspect Social Sentiment**: Check engagement velocity, fear/greed polarity, and top-discussed catalysts across the sample.
5. **Read the Jev System One Decision**: Review the recommended action (`STRONG BUY`, `BUY`, `HOLD`, `TAKE PROFIT`, `SELL`), confidence percentage, and rationale.
6. **Execute Manually**: Review the calculated entry, stop-loss, and target levels and execute manually on your preferred exchange or DEX.

---

## Future Roadmap

The planned next step is to combine traditional technical indicators with Jev's social sentiment analysis, then introduce optional automated trading through a staged rollout. These capabilities are planned and are not implemented yet; the current platform remains a manual decision-support terminal.

### Phase 1 — Expand Traditional Technical Indicators

- Build on the existing RSI (14) calculation with SMA/EMA trend filters, MACD, Bollinger Bands, ATR, and volume indicators such as VWAP and OBV.
- Extend candle history beyond the current 48 hourly candles to support indicator warm-up and configurable timeframes, such as 15 minutes, 1 hour, 4 hours, and 1 day, where the exchange supports them.
- Calculate indicators deterministically from completed candles, label their timeframe and data freshness, and report insufficient history as unavailable.
- Display indicator overlays and a technical summary alongside the existing sentiment and derivatives panels.

### Phase 2 — Combine Technical and Sentiment Signals

- Add structured technical features to the Jev evaluation alongside social sentiment, funding rates, and open interest.
- Introduce configurable strategy rules for trend, momentum, volatility, and sentiment confirmation, with clear explanations when signals agree or conflict.
- Define deterministic entry and exit rules, ATR-based stop distances, position sizing, and minimum signal requirements before a decision can become an executable trade intent.
- Version strategy settings and record the inputs behind every decision so results can be reproduced and reviewed.

### Phase 3 — Backtesting and Paper Trading

- Build historical replay using timestamped market data and archived sentiment available at each decision time, avoiding look-ahead bias. Evaluate combined strategies only where historical sentiment coverage exists.
- Compare technical-only, sentiment-only, and combined strategies against a buy-and-hold baseline, including fees, spread, slippage, and funding costs where applicable.
- Report drawdown, net returns, trade count, win rate, and risk-adjusted performance across out-of-sample periods and different market conditions.
- Run the same strategy and order lifecycle in a paper-trading mode, including partial fills, rejected orders, and simulated exchange outages, before enabling live execution.

### Phase 4 — Optional Exchange Execution

- Introduce an authenticated execution service with CCXT exchange adapters, starting with one exchange and spot trading before considering derivatives.
- Support an approval-required mode first, followed by explicitly enabled automation within user-configured limits. Keep paper trading as the default.
- Store exchange credentials securely on the server, require trading-only keys with withdrawals disabled, and enforce exchange precision, minimum order sizes, and available-balance checks.
- Track submitted orders, fills, cancellations, and positions persistently; prevent duplicate orders on retries and reconcile exchange state after restarts or connection loss.

### Phase 5 — Risk Controls and Operations

- Enforce position and portfolio exposure limits, maximum daily loss, maximum drawdown, and limits on concurrent trades independently of model output.
- Block new orders when data is stale, missing, or marked as fallback/synthetic, or when required market, account, or exchange checks fail.
- Add a kill switch that stops new orders and cancels pending orders, with a separate explicit control for closing existing positions.
- Provide execution alerts, an audit trail, performance monitoring, and recovery procedures. Expand exchange coverage or derivatives support only after the initial integration meets predefined paper-trading and operational acceptance criteria.

Each phase depends on validation of the previous phase; no release dates are committed. Automated execution will require explicit opt-in and will remain subject to deterministic risk controls.

---

## ⚖️ Legal & Financial Disclaimer

**Not Financial Advice**: This software is created strictly for educational, research, and technical demonstration purposes. None of the quantitative models, sentiment scores, trade levels, or verdicts generated by this platform constitute financial, investment, or trading advice. Trading cryptocurrencies carries a significant risk of financial loss. Always perform your own due diligence and never trade with funds you cannot afford to lose.

---

## License

This project is open-source software licensed under the [MIT License](LICENSE).
