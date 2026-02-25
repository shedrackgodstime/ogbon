# ỌGBỌN — Data, Schema & Foundation
> **Status:** Living Document  
> **Companion To:** `FOREX_INTELLIGENCE_SYSTEM.md`  
> **Focus:** Data sourcing, data quality, database schema, and project structure  

---

## 1. Where Phase 0 Actually Begins

Three prerequisites before any system code is written:

- Python environment confirmed and ready
- Historical GBP/USD Daily data — sourced, clean, and stored
- Broker API — deprioritized until the decision engine is validated

**On Broker API:** ỌGBỌN's first form is a decision engine. She tells you whether to trade and why. You act on it manually. Execution automation comes later as a bonus, after her intelligence is proven. The brain is built before the hands.

**On Data:** This is where Phase 0 lives. Three questions matter here — should collection be automated, how clean is clean enough, and what exactly does the schema look like.

---

## 2. Should Data Collection Be Automated?

**Yes. But with discipline.**

Not scraping randomly from multiple sources and hoping for consistency. The approach is:

- Identify **one authoritative source**
- Write **one clean script** that pulls from it on a schedule
- Validate what it receives before storing it
- Run it during the active daily session window

Using multiple sources and merging them introduces inconsistency — different timestamps, different price conventions, different handling of holidays. One source. One script. One format. Always.

**Recommended sources for GBP/USD Daily candles:**

| Source | Why |
|--------|-----|
| Stooq | Deep free history, no authentication required, clean CSV |
| OANDA REST API | Free tier, structured, reliable, well documented |
| MetaTrader 5 Python API | Most practical long term, connects to most brokers |

Start with **Stooq** for bulk historical data and **OANDA or MT5** for ongoing daily updates. They can coexist as long as the validation layer catches inconsistencies before they enter the database.

---

## 3. How Clean Is Clean Enough?

Clean data for ỌGBỌN means four rules — no exceptions.

**Rule 1 — No Missing Dates**
Every trading day must have a candle. If a date is missing the system must know it is missing and handle it explicitly. Silent skips corrupt time-based calculations downstream without any visible error. Missing dates must be flagged and logged.

**Rule 2 — No Zero Values**
A candle with a zero open, high, low or close is corrupted data. Full stop. Caught on ingestion and quarantined before touching the database.

**Rule 3 — No Duplicate Entries**
The same date appearing twice with different values breaks everything downstream. Uniqueness is enforced at the database level — not just at the application level.

**Rule 4 — Chronological Integrity**
Data must always be stored and retrieved oldest to newest. Any candle arriving out of sequence must be validated against its neighbors before acceptance.

**The quarantine principle:**
Anything failing those four rules gets **flagged and quarantined, not deleted.** Silent deletion of bad data hides problems. A quarantine table preserves the evidence.

---

## 4. The Database Schema

Four tables. Each has one job. The third one is what most systems never build — and it becomes the most valuable table over time.

---

### Table 1 — price_data

Every GBP/USD Daily candle the system collects.

```sql
CREATE TABLE price_data (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    pair        TEXT NOT NULL,
    date        DATE NOT NULL,
    open        REAL NOT NULL,
    high        REAL NOT NULL,
    low         REAL NOT NULL,
    close       REAL NOT NULL,
    volume      REAL,
    source      TEXT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(pair, date)
);
```

**Why `source` column:** When two sources disagree on a historical price, you need to know where each candle came from. Without this column that investigation is impossible.

**Why `UNIQUE(pair, date)`:** The database itself rejects duplicates before they can cause harm — not just application logic.

---

### Table 2 — economic_events

All GBP and USD economic calendar events with impact ratings and outcomes.

```sql
CREATE TABLE economic_events (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    event_date    DATE NOT NULL,
    event_time    TEXT,
    currency      TEXT NOT NULL,
    event_name    TEXT NOT NULL,
    impact        TEXT NOT NULL CHECK(impact IN ('HIGH', 'MEDIUM', 'LOW')),
    forecast      REAL,
    actual        REAL,
    previous      REAL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Why `forecast`, `actual`, `previous`:** These three numbers together tell the real story of an event. A CPI reading that matches forecast moves markets differently than one that surprises to the upside. This data trains the Context Layer to understand not just that an event happened but how much it mattered.

**Why `CHECK(impact IN ...)`:** The Context Layer only knows three states — HIGH, MEDIUM, LOW. Enforcing this at schema level means it never receives unexpected input.

---

### Table 3 — system_decisions

Every decision ỌGBỌN makes — and what actually happened after.

```sql
CREATE TABLE system_decisions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    decision_date   DATE NOT NULL,
    pair            TEXT NOT NULL,
    signal          TEXT NOT NULL CHECK(signal IN ('LONG', 'SHORT', 'NO_TRADE')),
    confidence      REAL,
    technical_score REAL,
    context_score   REAL,
    entry_price     REAL,
    stop_loss       REAL,
    take_profit     REAL,
    reason          TEXT NOT NULL,
    outcome         TEXT CHECK(outcome IN ('WIN', 'LOSS', 'CANCELLED', 'PENDING')),
    outcome_date    DATE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Why this table matters more than the others:** This is how ỌGBỌN eventually learns from herself — what she got right, what she got wrong, and under what conditions. It is the training data for the smarter version that comes later. Start collecting from day one even when early decisions are rough. The data compounds. A year from now this table is gold.

**The `reason` column is NOT optional:** Every decision must produce a human-readable reason before it is logged. If the system cannot explain why it made a decision it should not be making that decision. Enforced at schema level — `NOT NULL`.

---

### Table 4 — data_quarantine

Any data that failed validation during ingestion.

```sql
CREATE TABLE data_quarantine (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    source_table    TEXT NOT NULL,
    raw_data        TEXT NOT NULL,
    failure_reason  TEXT NOT NULL,
    flagged_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Why this exists:** Bad data that is silently deleted leaves no evidence. Quarantine tells you where sources are failing, how often, and under what conditions. Invisible when things go well — invaluable when things go wrong.

---

## 5. Project Folder Structure

Decided before the first script — not after. Changing folder structure mid-build breaks imports and wastes time.

```
ogbon/
│
├── data/
│   └── ogbon.db                  ← SQLite database lives here
│
├── ingestion/
│   ├── price_fetcher.py          ← Pulls Daily candles from source
│   ├── calendar_fetcher.py       ← Pulls economic events
│   └── validator.py              ← Validates data before storage
│
├── analysis/
│   ├── technical.py              ← Analysis Engine (Component 2)
│   └── features.py               ← Feature engineering from raw candles
│
├── context/
│   └── calendar_filter.py        ← Context Layer (Component 3)
│
├── decision/
│   ├── decision_engine.py        ← Decision Layer (Component 4)
│   └── risk.py                   ← Risk rules — hardcoded, never bypassed
│
├── backtest/
│   ├── runner.py                 ← Runs system logic against historical data
│   └── metrics.py                ← Calculates performance metrics
│
├── models/
│   └── (trained model files live here when ready)
│
├── logs/
│   └── (daily run logs)
│
├── config.py                     ← All settings in one place
├── main.py                       ← Entry point — runs the daily session
├── init_db.py                    ← Run once — creates database and tables
├── fetch_history.py              ← Run once — pulls 40yr GBP/USD history
└── requirements.txt              ← All Python dependencies
```

**Why `config.py` matters:** Every value that might change — risk percentage, lookback window, confidence threshold, database path — lives in one file. When you want to change the risk per trade from 1% to 2%, you change one line in one place.

---

## 6. Open Questions

Questions without final answers yet — kept here so they are not forgotten:

- **Holiday calendar vs missing data** — the validator needs to distinguish expected market closures from actual data failures. A forex trading calendar is needed for this
- **Economic calendar update cadence** — events are scheduled in advance but `actual` values only exist after the event occurs. When and how does the update run?
- **`system_decisions` outcome updates** — what triggers the outcome being written? A separate script? Part of the daily run? Needs a defined answer before the decision table is used seriously
- **Stooq data depth** — free sources reliably cover GBP/USD Daily from approximately 1990. Pre-1990 data may require a paid source. Verify actual coverage before depending on it

---

## 7. Discovered Later — Space For Future Entries

> *This section is intentionally blank. Discoveries, corrections, new questions and pivots during build belong here.*

---

*Companion document to `FOREX_INTELLIGENCE_SYSTEM.md`*

