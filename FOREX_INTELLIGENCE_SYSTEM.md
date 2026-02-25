# ỌGBỌN — Master Design Document
> **Status:** Living Document — Room deliberately left for unanswered questions and future discoveries  
> **Last Reasoned:** February 2026  
> **Pair in Focus:** GBP/USD  
> **Timeframe:** Daily  
> **Operating Window:** ~5 hours/day  
> **Hardware:** Low-spec PC (Nigerian power constraints considered as architectural input, not limitation)

---

## 1. The Vision

ỌGBỌN is a focused, self-contained trading intelligence system that does what a disciplined manual trader does — reads price, understands macro context, weighs risk, and makes a reasoned decision — but does it consistently, without emotion, without fatigue, and within the constraints of the environment it runs in.

This is **not** a get-rich-quick bot. It is not a signal copier. It is a system that reasons and explains itself. When it says trade, you know why. When it says don't, you know why. Over time, it becomes smarter because of data it collected itself.

**The first version of ỌGBỌN is a decision engine — not a trading bot. She tells you trade or don't trade, and exactly why. You decide whether to act on it. This is intentional. It means you are validating her intelligence before trusting her with execution. The brain is built before the hands.**

The system must be:
- **Focused** — GBP/USD, Daily timeframe, one job done well
- **Explainable** — every decision comes with a reason
- **Resilient** — PC going off mid-day must not break anything
- **Honest** — it disagrees with the user when the data says so
- **Growable** — designed from day one to become smarter over time

---

## 2. Core Decisions Made (and Why)

### 2.1 Daily Timeframe — Not a Compromise, A Superior Choice

**Decision:** The system analyzes and acts on Daily candles only.

**Why:** The original instinct was "5 hours a day is my limit." But the real reason to choose Daily goes beyond power constraints:

- Daily candles filter out intraday noise that causes false signals
- Institutional money — the entities that actually move GBP/USD — thinks in Daily and Weekly terms. Aligning with this means reading the same chart the big players respond to
- One high-quality analysis per day beats ten low-quality ones
- Gives the system time to be right — not every move happens in minutes

**Power constraint alignment:** On the Daily timeframe, analysis happens once per session. Trades are fully defined (entry, stop loss, take profit) before being sent to the broker server. Once placed, the broker holds the trade. The PC going off is architecturally irrelevant.

---

### 2.2 Swing Trading — Positions Held 2 to 7 Days

**Decision:** Positions are held across multiple days. The system does not scalp. It does not day trade.

**Why:** Any approach requiring the PC to monitor and manage an open trade in real time is incompatible with unreliable power. Scalping is eliminated immediately — a position open for seconds or minutes with no PC monitoring is an unmanaged risk. Day trading has the same vulnerability.

Swing trading on the Daily timeframe means:
- Analysis runs once per day during the active window
- Full trade parameters are set before placement
- The broker manages the open trade server-side
- The PC's on/off state has zero impact on the trade's integrity

This is not a workaround. This is the correct approach for the environment.

---

### 2.3 GBP/USD — One Pair to Start

**Decision:** The system focuses exclusively on GBP/USD at launch.

**Why:**
- Most liquid pair globally — tightest spreads, lowest slippage
- Enormous historical data available for training
- Well-documented economic drivers (Bank of England, UK CPI, US Fed, NFP)
- Reduces complexity, storage requirements, and analytical noise dramatically

More pairs can be added after the system proves itself on one. Adding pairs before the foundation is solid is how systems become unfocused and unreliable.

---

### 2.4 No General-Purpose LLM in the Core System

**Decision:** No Mistral, no ChatGPT API, no Ollama as the brain of this system.

**Why:** A general-purpose LLM is trained on everything — cooking, history, poetry, random internet content. Using it as the core of a Forex system means carrying enormous irrelevant knowledge that adds noise, increases hardware requirements, and creates unpredictable reasoning paths.

The system's intelligence comes from purpose-built components, each trained or designed for one specific job. A focused assembly of specialists beats one generalist in every dimension that matters here.

**Where a language model could eventually fit:** Only for generating human-readable explanations of decisions — the "why" output. Even then, it would be tightly constrained with structured inputs and outputs. This is a Phase 3 consideration, not Phase 1.

---

### 2.5 Training From Scratch for Numerical Components

**Decision:** The price analysis and pattern recognition models are trained from scratch on Forex data only.

**Why:** There is no pretrained numerical model to "inherit garbage from" in the same way as LLMs, but the principle holds — models trained only on GBP/USD Daily price history will learn relationships specific to that pair and timeframe. No transfer learning, no borrowed assumptions. Clean data in, clean reasoning out.

The analogy that crystallized this: *AI is like a baby — the architecture is the brain, the training data is the upbringing. Raise it only in a trading environment and it thinks like a trader.*

---

## 3. System Architecture

The system has four components. Each has one job.

```
┌─────────────────────────────────────────────────────┐
│                  DAILY TRIGGER                       │
│         (Runs once per active session)               │
└──────────────┬──────────────────────────────────────┘
               │
       ┌───────▼────────┐
       │  DATA ENGINE   │  ← Pulls GBP/USD Daily candles
       │  Component 1   │  ← Pulls economic calendar (next 72hrs)
       └───────┬────────┘
               │
       ┌───────▼────────┐
       │  ANALYSIS      │  ← Trained from scratch on GBP/USD history
       │  ENGINE        │  ← Reads price action, patterns, structure
       │  Component 2   │  ← Produces technical signal + confidence
       └───────┬────────┘
               │
       ┌───────▼────────┐
       │  CONTEXT       │  ← Economic calendar impact scoring
       │  LAYER         │  ← Filters or adjusts signal based on
       │  Component 3   │    upcoming high-impact events
       └───────┬────────┘
               │
       ┌───────▼────────┐
       │  DECISION      │  ← Combines technical signal + context
       │  LAYER         │  ← Applies risk rules
       │  Component 4   │  ← Outputs: Direction / Confidence / Entry
       └───────┬────────┘   → Stop Loss / Take Profit / Position Size
               │             → Plain reason why
               ▼
       ┌───────────────┐
       │  BROKER API   │  ← Order placed server-side
       │  (MT5/OANDA)  │  ← PC can go off. Trade runs itself.
       └───────────────┘
```

---

### Component 1 — Data Engine

**Job:** Fetch and store clean data. Nothing else.

**What it pulls:**
- GBP/USD Daily OHLCV candles (Open, High, Low, Close, Volume)
- Economic calendar events affecting GBP or USD for the next 72 hours
- Impact rating of each event (High / Medium / Low)
- Historical outcomes of similar events on GBP/USD (built over time)

**What it does NOT pull:**
- News articles
- Social media sentiment
- Any other currency pair

**Storage:** SQLite locally. Lightweight, no server required, runs on old hardware without complaint.

**APIs to consider:**
- MetaTrader 5 Python API (free, connects to most brokers)
- OANDA REST API (free tier available)
- Investing.com economic calendar (scrape or API)
- ForexFactory calendar API (community maintained)

---

### Component 2 — Analysis Engine

**Job:** Read price structure and produce a technical signal.

**What it analyzes:**
- Recent Daily candle patterns (last 50-200 candles as rolling window)
- Key support and resistance zones (calculated, not drawn manually)
- Trend direction and strength
- A small, deliberate set of indicators — to be finalized before build
- Volume behavior relative to price movement

**What it is:**
- A model trained from scratch on GBP/USD Daily historical data
- Trained to recognize setups that historically preceded significant moves
- Outputs a signal: Long / Short / No Trade, with a confidence score (0-100)

**What it is NOT:**
- A rule-based "if RSI < 30 buy" script
- A borrowed pretrained model
- An indicator dashboard

**Training data requirement:** Minimum 10 years of GBP/USD Daily data. More is better.

---

### Component 3 — Context Layer

**Job:** Filter or adjust the technical signal based on macro risk.

**Logic:**
- High-impact event (NFP, BOE rate decision, US CPI) within 24 hours → signal confidence penalized regardless of technical quality
- High-impact event within 48-72 hours → noted, slight penalty applied
- No significant events in window → technical signal passes through unmodified

**Why this matters:** A technically perfect setup entering 4 hours before a surprise Bank of England statement is not a good trade. The system must know what it doesn't know about upcoming risk. This is the component that prevents confident stupidity.

**This is NOT:**
- A news sentiment reader
- An NLP model
- A complex scoring algorithm

It is a clean rules-based filter that cross-references the technical signal against the economic calendar. Simple. Reliable. Understandable.

---

### Component 4 — Decision Layer

**Job:** Make the final call and explain it.

**Inputs:**
- Technical signal + confidence from Component 2
- Context risk score from Component 3
- Current account balance and open positions

**Outputs — every single time, no exceptions:**
- **Decision:** Trade / No Trade
- **Direction:** Long / Short (if trade)
- **Entry Price:** Specific level
- **Stop Loss:** Specific level with pip distance
- **Take Profit:** Specific level (minimum 1:2 risk/reward enforced)
- **Position Size:** Calculated based on defined risk per trade (e.g., 1-2% of account)
- **Reason:** Plain language explanation of why this decision was made

**Non-negotiable rules hardcoded into this layer:**
- Never risk more than 2% of account on a single trade
- Never place a trade with less than 1:2 risk/reward ratio
- Never trade within 2 hours of a High-impact event
- If account drawdown exceeds 10% in a rolling period → system pauses and flags for review

**What this layer does NOT do:**
- Override its own rules because the signal looks "really good"
- Place trades without a defined stop loss
- Agree with the user if the data says otherwise

---

## 4. Backtesting Framework — Must Be Built First

> **This is not optional. This is not Phase 2. This is the foundation.**

**Why first:**

Every signal the Analysis Engine produces needs to be validated against historical data before it is trusted with real money or even a demo account. Without a backtesting framework, there is no way to know if the system is intelligent or just confidently wrong. Building components without this is building on sand.

**What the backtesting framework does:**

Takes historical GBP/USD Daily data, runs the system's logic against it as if it were making live decisions day by day, records every signal generated, records what actually happened after each signal, calculates performance metrics.

**Metrics the framework must produce:**
- Win rate (% of trades that hit take profit)
- Average Risk/Reward achieved
- Maximum drawdown (worst losing streak in % terms)
- Profit factor (gross profit divided by gross loss)
- Sharpe ratio (return relative to risk taken)
- Performance by market condition (trending vs ranging)

**The honest rule:** If the backtest results are not satisfactory, the system does not move to live testing. Period. The backtest is the system's CV — it must earn the right to trade real money.

---

## 5. Technology Stack

| Component | Tool | Why |
|-----------|------|-----|
| Language | Python 3.10+ | Industry standard for ML/finance, vast library support |
| Data Storage | SQLite | Lightweight, no server, runs on old PC |
| Broker Connection | MetaTrader 5 Python API | Free, widely supported, executes trades server-side |
| Price Data | MT5 API or OANDA | Historical + live Daily candles |
| Economic Calendar | ForexFactory / Investing.com | High-impact event tracking |
| ML Framework | scikit-learn (start) → PyTorch (scale) | scikit-learn for initial models, PyTorch when complexity grows |
| Backtesting | Backtrader or custom-built | Backtrader is Python-native, well documented |
| Scheduling | APScheduler | Runs analysis once per active session |
| Logging | Python logging module | Every decision logged with timestamp and reason |

---

## 6. Build Order (Phases)

### Phase 0 — Foundation (Before Writing One Line of System Code)
- [ ] Collect minimum 10 years of GBP/USD Daily OHLCV data
- [ ] Collect GBP/USD relevant economic event history with outcomes
- [ ] Set up SQLite database schema
- [ ] Set up project folder structure
- [ ] Define and document the specific indicators the Analysis Engine will use
- [ ] **Build the backtesting framework**

> Nothing else starts until Phase 0 is complete.

---

### Phase 1 — Data Engine
- [ ] MT5 or OANDA API connection working
- [ ] Daily candle pull and storage running cleanly
- [ ] Economic calendar pull and storage running cleanly
- [ ] Data validation (no gaps, no corrupted candles)
- [ ] Scheduler running reliably within the 5-hour window

---

### Phase 2 — Analysis Engine (First Version)
- [ ] Feature engineering from raw price data
- [ ] Initial model trained on historical GBP/USD data
- [ ] Signal output: Long / Short / No Trade + confidence score
- [ ] Backtested against Phase 0 framework
- [ ] Performance metrics reviewed honestly before proceeding

---

### Phase 3 — Context Layer
- [ ] Economic calendar risk scoring logic implemented
- [ ] Integration with Analysis Engine signal
- [ ] Backtested with context layer active
- [ ] Performance comparison: with vs without context layer

---

### Phase 4 — Decision Layer
- [ ] Full parameter generation (entry, SL, TP, position size)
- [ ] Risk rules hardcoded and tested
- [ ] Plain language reason generation (rule-based to start)
- [ ] End-to-end system test on paper trading / demo account
- [ ] Minimum 3 months demo trading before live consideration

---

### Phase 5 — Live + Learning
- [ ] Live trading with minimum capital
- [ ] System logs every decision and outcome
- [ ] Data accumulates for future model retraining
- [ ] Monthly review of performance metrics
- [ ] Model retraining schedule established

---

### Phase 6 — Intelligence Growth (Future, Not Now)
- [ ] Fine-tuning Analysis Engine on self-collected outcome data
- [ ] Evaluating whether a constrained language layer adds genuine value for explanations
- [ ] Evaluating second currency pair addition (only after Phase 5 is stable)
- [ ] Evaluating more sophisticated ML architectures (LSTM, Temporal Fusion Transformer)

---

## 7. Open Questions (Deliberately Unanswered)

These are questions identified during design that need answers before or during build. They are listed here so they are not forgotten.

**Technical:**
- Which specific indicators will the Analysis Engine use? (Must be decided before Phase 1 training — adding indicators after training changes the model entirely)
- What rolling candle window does the Analysis Engine look back? (50, 100, 200 candles — each changes what the model learns)
- What exact risk percentage per trade? (1% is conservative, 2% is moderate — must be fixed before backtesting)
- Which broker will be used and do they support MT5 Python API from Nigeria without restrictions?
- What minimum account size makes the position sizing logic meaningful?

**System behavior:**
- How does the system handle a trade already open when the next daily analysis runs? Does it stay silent, does it trail the stop, or does it reassess?
- What happens when the economic calendar data feed is unavailable? Does the system trade with technical signal only or does it stand down?
- How does the system define "market condition" — trending vs ranging — and does it behave differently in each?

**Data:**
- Where exactly is the 10+ years of GBP/USD Daily data coming from and how is its quality validated?
- How far back does the economic calendar historical data go and is outcome data (what GBP/USD did after each event) available or does it need to be constructed manually?

**Operational:**
- What happens to the system during the Nigerian market hours when spread widening occurs on GBP/USD? (Less relevant on Daily but worth understanding)
- Internet connectivity reliability — does the system need a fallback if the calendar API call fails mid-session?

---

## 8. What This System Is Not

Stated clearly so there is no confusion during build:

- It is not a get-rich system
- It is not a fully autonomous trading bot that should be trusted blindly
- It is not finished when it starts making money — it is finished when it is consistently right *for understood reasons*
- It is not a system that agrees with its owner to make them feel good
- It is not something to rush — a system built fast is a system built wrong

---

## 9. The System's Character

This was discussed and matters enough to document:

The system reasons independently. If the user says "I think GBP/USD is going up tomorrow" — the system does not factor that in. It shows its own analysis and its own conclusion. If the user's instinct and the system's analysis agree — good. If they disagree — the system explains why it sees it differently and the user decides with full information.

The system does not have moods. It does not get excited during a winning streak or fearful during a losing one. Its risk rules do not bend because a setup "looks really good." Consistency is the feature, not a bug.

Over time, as the system accumulates its own trade data, there is a version of this system that literally learns from its own history — what it got right, what it got wrong, and why. That version is the goal. This document is the road to it.

---

## 10. Discovered Later — Space For Future Entries

> *This section is intentionally blank. As build progresses, questions not yet asked will surface. Discoveries, corrections, pivots and new understanding belong here. The document grows with the system.*

---

*"Less tools. More focus. Grows with you."*

