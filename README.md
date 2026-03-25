# 🧠 AI Market Intelligence & Trading Signal Pipeline

> An automated, multi-stage AI pipeline that transforms daily financial news into structured, actionable trading signals — built with n8n, LLM orchestration, and Google Sheets output.

---

## 📌 Project Overview

This system ingests financial news every morning, runs two sequential AI passes to rank and score articles for market impact, and outputs structured trading signals to Google Sheets for daily review and strategy integration.

**Core objective:** Identify high-probability, market-moving news signals and quantify their impact to improve trading decision-making on the US S&P 500 / equity index futures.

---

## 🏗️ System Architecture

```
RSS Feeds → Deduplicate → AI Pass 1 (Triage) → Extract Top 10
    → Fetch Full Content → Fallback Logic → AI Pass 2 (Scoring)
        → Quality Check → Google Sheets Output
```

### Stage-by-Stage Breakdown

| Stage | Node | Description |
|-------|------|-------------|
| 1 | Schedule Trigger | Runs pipeline on a daily schedule |
| 2 | Fetch News RSS Feeds | Ingests articles from CNBC, Reuters (expanding to BLS, BEA, AP) |
| 3 | Deduplicate Articles | Removes near-duplicate stories to prevent signal over-weighting |
| 4 | Combine Articles for AI Ranking | Prepares ~30 candidates for Pass 1 |
| 5 | **AI Pass 1 — Triage** | LLM selects top 10 most market-relevant articles using strict ID-matching |
| 6 | Extract Top 10 Articles | Maps AI-selected IDs back to original candidates (hallucination guard) |
| 7 | Fetch Full Article Content | HTTP requests for full article text |
| 8 | Detect Blocked / Paywalled Pages | Identifies hard blocks (403) and soft blocks (captcha, bot detection) |
| 9 | Merge + Content Preparation | Assigns text source priority: full text → snippet → title fallback |
| 10 | Prepare Article Text for AI Pass 2 | Formats content for structured scoring |
| 11 | **AI Pass 2 — Market Impact Scoring** | LLM converts each article into a structured trading signal |
| 12 | Merge Pass 2 Output with Metadata | Recombines scores with original article data |
| 13 | Quality Check | Flags runs where blocked content exceeds acceptable threshold |
| 14 | Format + Append to Google Sheets | Writes article data and run metadata to two separate sheets |

---

## 🤖 AI Pass 1 — Triage Logic

**Purpose:** From ~30 candidate articles, select exactly 10 with the highest market relevance.

**Output per article:**
- `id` — must match input candidate list exactly (hallucination prevention)
- `reason` — why this article is market-relevant
- `urgency` — speed of potential market impact
- `tags` — affected sectors, assets, or macro themes

**Key design rule:** The AI cannot invent articles. It must select IDs from the provided candidate list only. IDs are mapped back to originals after selection to guarantee data integrity.

---

## 📊 AI Pass 2 — Market Impact Scoring

**Purpose:** Convert each article into a structured trading signal.

**Output fields:**

| Field | Type | Description |
|-------|------|-------------|
| `summary` | string | Concise market-relevant summary |
| `impact_score` | 1–10 | Estimated strength of market impact |
| `market_bias` | bullish / bearish / neutral | Directional signal |
| `confidence` | 1–10 | AI confidence in the assessment |
| `market_scope` | broad / sector / company | How wide the impact is expected to be |
| `affected_assets` | list | Specific indices, sectors, or tickers affected |
| `trade_implication` | string | Actionable insight for trading decisions |

---

## 🛡️ Content Fallback Hierarchy

Many financial sources (Reuters, Bloomberg) block automated requests. The system handles this gracefully:

```
1. Full article text     ← preferred
2. Snippet fallback      ← usable for macro signals
3. Title-only fallback   ← last resort, still captures macro events
```

Each article is tagged with its `text_source` and `fetch_block_reason` so signal quality can be assessed per run.

---

## 📋 Google Sheets Output

### Sheet 1: `Daily_News_Articles`

| Column | Description |
|--------|-------------|
| `run_timestamp` | When the pipeline ran |
| `id`, `title`, `source`, `url` | Article metadata |
| `text_source` | full_text / snippet_fallback / title_only_fallback |
| `published` | Article publish time |
| `fetch_blocked`, `soft_block_detected` | Block detection flags |
| `triage_reason`, `triage_urgency`, `triage_tags` | Pass 1 output |
| `summary`, `impact_score`, `market_bias` | Pass 2 output |
| `confidence`, `market_scope`, `affected_assets_flat` | Pass 2 output |
| `trade_implication` | Actionable trading signal |

### Sheet 2: `Run_Metadata`

Tracks system-level quality per execution:

| Column | Description |
|--------|-------------|
| `total_selected` | Articles processed |
| `full_text_count` | Articles with full content |
| `blocked_count` | Articles where fetch failed |
| `quality_warning` | Flag if blocked rate exceeds threshold |
| `quality_message` | Human-readable quality summary |

---

## ✅ Current Status

| Component | Status |
|-----------|--------|
| End-to-end pipeline | ✅ Working |
| Google Sheets output | ✅ Writing daily |
| AI triage (Pass 1) | ✅ Functioning |
| AI scoring (Pass 2) | ✅ Functioning |
| Block detection & fallback | ✅ Working |
| Full-text coverage | ⚠️ Limited — paywall issues being addressed |
| Pass 2 output consistency | ⚠️ Being tightened |

---

## 🗺️ Roadmap

### Phase 1 — Current (Optimisation)
- Expand RSS sources (BLS, BEA, AP News)
- Improve Pass 2 prompt strictness and output consistency
- Increase full-text fetch success rate

### Phase 2 — Dashboard
- Visualise average daily impact score
- Bullish vs bearish ratio over time
- Sector distribution heatmap
- Built with Google Sheets charts or Streamlit

### Phase 3 — Trading Dataset Integration
- Combine news signals with time-based intraday features
- Integrate with existing IBKR data pipeline (XGBoost model, 70% directional accuracy)
- Build direction prediction model combining sentiment + price data

### Phase 4 — Real-Time Alerts
- Trigger notifications when:
  - `impact_score ≥ 7`
  - `confidence ≥ 8`
  - `market_scope = broad_market`

---

## 🔧 Tech Stack

| Tool | Role |
|------|------|
| **n8n** | Workflow orchestration and automation |
| **LLM (AI nodes)** | Pass 1 triage + Pass 2 market scoring |
| **Python** | Predictive modelling (XGBoost, Scikit-Learn, Pandas) |
| **IBKR API** | Market data ingestion |
| **Google Sheets API** | Structured output storage |
| **RSS Feeds** | News ingestion (CNBC, Reuters, expanding) |

---

## 🧠 Design Philosophy

**1. Something is better than nothing**
Snippets and titles still carry macro signal value. The fallback hierarchy ensures the pipeline never fails silently.

**2. Eliminate hallucination at all costs**
AI cannot invent articles. Strict ID-matching between Pass 1 output and the original candidate list is enforced at every run.

**3. Structured outputs over raw text**
Every piece of content is converted into scores, classifications, and signals — not paragraphs. Built for decision support, not reading.

**4. Build for trading, not just analysis**
Every output answers one question: *does this affect market direction, and how confident are we?*

---

## 📈 Related Work

This pipeline is part of a broader trading strategy project that includes:
- Predictive modelling using XGBoost and clustering on the US 500 index (70% directional accuracy)
- Feature engineering with SelectKBest for signal optimisation
- Intraday dataset collection via IBKR API

---

## 👤 Author

**Luke M. Jones**
MBA Candidate — AI Strategy & Project Management, Anglia Ruskin University, Cambridge UK
[LinkedIn](https://linkedin.com/in/lukemjones1) | LMJSplash@gmail.com

---

*This project is under active development. See roadmap above for planned enhancements.*
