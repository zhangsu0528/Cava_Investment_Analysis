# CAVA Group — Data Science Write-Up
**Date:** May 3, 2026 | **Author:** Su Zhang

---

## Overview

This write-up documents the data science methodology underlying my investment
analysis of CAVA Group (NYSE: CAVA). It covers data source selection,
preprocessing decisions, modeling approach, and key technical findings.
All code is available in the accompanying GitHub repository.

**Repository structure:**
```
cava-investment-analysis/
├── data_preprocessing/
│   ├── 00_data_collection.ipynb
│   └── 01_preprocessing.ipynb
├── modeling/
│   ├── 02_sssg_forecasting.ipynb
│   └── 03_earnings_nlp.ipynb
├── data/
│   ├── raw/
│   └── processed/
└── outputs/
    └── figures/
```

---

## 1. Data Sources

### 1.1 Primary Data: Quarterly Fundamentals

**Source:** CAVA SEC filings (10-Q, 10-K) and earnings press releases via
CAVA Investor Relations (ir.cava.com) and SEC Edgar.

**Why this source:** CAVA's key performance metrics — Same-Restaurant Sales
Growth (SSSG), Average Unit Volume (AUV), restaurant-level profit margin,
and digital revenue mix — are only available from the company's own
disclosures. There is no third-party data provider that publishes these
restaurant-specific KPIs at the required granularity.

**Data collected:** 15 quarters (Q2 2022–Q4 2025), manually verified from
earnings releases. Key metrics: SSSG%, revenue, AUV, restaurant count, net
new openings, restaurant-level margin, digital revenue mix, adjusted EBITDA
margin.

**Key challenge:** CAVA uses a fiscal calendar with variable quarter lengths
(Q1 = 16 weeks, Q2-Q4 = 12 weeks). Raw revenue figures are not directly
comparable across quarters. I normalized using weekly revenue (revenue /
weeks_in_period) for time-series analysis, while using percentage-based
metrics (SSSG%, margin%) directly as they are already normalized.

**Data limitation:** CAVA IPO'd in June 2023, so stock price data is only
available from Q2 2023 onward. Pre-IPO quarters (Q2–Q4 2022, Q1 2023) have
fundamentals from 10-Q comparison periods but no market data.

### 1.2 Alternative Data: Google Trends

**Source:** Google Trends API via `pytrends` Python library (free).

**Why this source:** Search interest is a high-frequency (weekly) proxy for
consumer awareness and brand momentum. I hypothesized that rising search
interest for "CAVA restaurant" would lead SSSG by 1–2 quarters, as
increasing consumer curiosity should precede visit frequency growth.

**Data collected:** Weekly search interest (0–100 scale) for "CAVA
restaurant", "Chipotle", and "Sweetgreen" from January 2022 to April 2026.

**Finding:** CAVA's search interest is extremely low in absolute terms
(max value = 2 on Google's 0–100 scale vs. Chipotle's consistent 52–100),
reflecting CAVA's still-regional brand awareness. Share of search averaged
~1% across the period with no meaningful trend. Correlation with next-quarter
SSSG was near-zero (r = 0.06). This data was retained in the feature set
but Lasso regularization correctly zeroed it out.

**Lesson:** Low-signal data does not improve models — Lasso's automatic
feature selection handled this appropriately without manual intervention.

### 1.3 Alternative Data: Reddit Sentiment

**Source:** Reddit public API (no authentication required) via `requests`
library. Sentiment scored using VADER (Valence Aware Dictionary and
sEntiment Reasoner).

**Why this source:** Reddit communities (r/wallstreetbets, r/investing,
r/stocks, r/food, r/AskNYC) contain organic consumer and investor
discussions about CAVA that may reflect real-time sentiment shifts
not captured in lagging financial metrics.

**Data collected:** 93 posts initially; filtered to 61 strictly
CAVA-related posts after relevance filtering. Comments scraped for
high-upvote posts, with upvote-weighted sentiment aggregation.

**Key finding:** Post volume is sparse (avg ~5 posts per quarter),
making quarterly aggregation statistically unreliable. However, the
dataset captured the August 12, 2025 earnings miss in real time:
posts titled "Cava down -20% after hours" (2,168 upvotes, 528 comments)
and "Cava stock plummets after company lowers forecast" (433 upvotes)
were the most viral posts, consistent with the Q2 2025 SSSG miss.

**Modeling decision:** Reddit sentiment was zeroed out by Lasso across
all feature sets, confirming it lacks independent predictive power at
quarterly frequency. Used as qualitative context rather than model input.

### 1.4 Alternative Data: News Sentiment

**Source:** Finnhub API (free tier) for financial news headlines and
summaries. Sentiment scored using VADER on title (60% weight) + summary
(40% weight).

**Why this source:** Financial news coverage reflects analyst and media
sentiment that may lead or coincide with fundamental changes. Finnhub
provides structured access to financial news with ticker-level filtering.

**Data collected:** 373 articles (after deduplication and CAVA filtering)
from January 2025 to May 2026. Earlier data is not available on the
Finnhub free tier.

**Key limitation:** Coverage starts only in 2025, meaning news sentiment
is available for only 3 of 15 quarters (Q2–Q4 2025). This prevents
meaningful time-series modeling.

**Modeling decision:** News sentiment was zeroed out by Lasso, consistent
with the limited coverage. However, the Q2 2025 news sentiment decline
(-0.14 average compound score) directionally aligned with the SSSG miss,
providing qualitative validation.

### 1.5 Alternative Data: Analyst Actions

**Source:** `yfinance` Python library (free), pulling analyst
upgrade/downgrade history from Yahoo Finance.

**Data collected:** 164 analyst actions (July 2023–May 2026) from 15+
sell-side firms including Morgan Stanley, Citigroup, TD Cowen, and Barclays.

**Key finding:** All 164 analyst actions were either Buy/Outperform/
Overweight or Neutral/Hold — zero Sell ratings. Analyst sentiment showed
near-zero correlation with SSSG (r = -0.16 current quarter). This confirms
sell-side analysts are not useful predictors of CAVA's fundamental
performance, likely due to structural biases in sell-side coverage of
growth stocks.

### 1.6 Primary Data: Earnings Call Transcripts

**Source:** Capital IQ (institutional access) — 11 earnings call transcripts
from Q2 2023 to Q4 2025. Extracted using `pdfplumber` Python library.

**Why this source:** Earnings calls contain management's forward-looking
language and responses to analyst pressure — both scripted (prepared remarks)
and unscripted (Q&A). This provides qualitative signal not available in
structured financial data.

**Processing:** Transcripts were split into prepared remarks and Q&A sections
using Capital IQ's standard section markers ("Presentation" and
"Question and Answer"). Boilerplate legal disclaimers were filtered using
keyword exclusion before sentiment scoring.

---

## 2. Data Preprocessing

### 2.1 Feature Engineering

From the quarterly fundamentals, I engineered the following features:

```python
# Lag features (capture momentum)
sssg_pct_lag1  # Previous quarter SSSG
sssg_pct_lag2  # Two quarters ago SSSG
sssg_rolling_3q  # 3-quarter rolling average

# Rate-of-change features (capture direction of change)
auv_qoq        # AUV quarter-over-quarter % change
margin_qoq     # Restaurant margin QoQ pp change
digital_mix_qoq  # Digital mix QoQ pp change

# Event flags (capture structural breaks)
ipo_quarter    # Q2 2023 IPO
sssg_beat      # Quarters with significant positive surprise
sssg_miss      # Quarters with significant negative surprise
guidance_cut   # Quarters with FY guidance reduction
ath_quarter    # All-time high stock price quarter
```

**Design principle:** Rate-of-change features (QoQ changes) consistently
outperformed absolute-level features in Lasso selection. This reflects
that investors react to the *direction* of change in fundamentals, not
their absolute level.

### 2.2 Missing Value Treatment

| Column | Missing Quarters | Reason | Treatment |
|--------|-----------------|--------|-----------|
| Stock price metrics | Q2–Q4 2022, Q1 2023 | Pre-IPO | Keep NaN, exclude from price models |
| News sentiment | Q2 2022–Q1 2025 | Finnhub free tier limit | Keep NaN, add has_news_data flag |
| Reddit sentiment | 3 quarters | Low post volume | Keep NaN |
| Analyst score | Q2–Q4 2022, Q1–Q2 2023 | Pre-IPO, no coverage | Keep NaN |
| search_interest_qoq | Q2 2022 | First row, no prior period | Fill 0 |

**Key principle:** Sentiment-related NaN values were kept as NaN rather
than filled with 0, since 0 implies "neutral sentiment" while NaN implies
"no data" — fundamentally different meanings that would mislead a model.

### 2.3 Seasonality

CAVA exhibits strong quarterly seasonality: Q2–Q3 consistently outperform
Q1 and Q4 in SSSG and margins. I tested sine/cosine encoding and one-hot
quarter dummies as explicit seasonality features. Lasso zeroed out all
seasonal encodings, suggesting the financial features already implicitly
capture seasonal patterns. Explicit seasonality features introduced
redundant variance that hurt out-of-sample performance.

---

## 3. Modeling: SSSG Forecasting

### 3.1 Problem Formulation

**Target:** `sssg_pct_next` — next quarter's SSSG percentage
**Features:** 19 candidates (10 fundamentals + 5 alt data + 4 event flags)
**Sample size:** 12 usable quarters (after dropping NaN targets and lag NaNs)
**Validation:** Leave-One-Out Cross Validation (LOOCV)

**Why LOOCV:** With n=12, standard train/test splits (e.g., 80/20) would
yield only 2–3 test samples — insufficient for reliable performance
estimation. LOOCV uses n-1 samples for training and 1 for testing,
repeated n times, maximizing use of limited data while providing
unbiased performance estimates.

### 3.2 Model Selection

I tested four model families across multiple feature sets:

| Model | MAE (best config) | Notes |
|-------|------------------|-------|
| Naive Baseline (mean) | 7.12% | Benchmark |
| Linear Regression | 3.75% | Works well with 4 features |
| Ridge CV | 4.27% | Unstable alpha selection |
| **Lasso CV** | **3.63%** | **Best overall** |
| Random Forest | 6.12% | Insufficient data for trees |

**Why Lasso over Ridge:** Ridge CV consistently selected alpha=0.01
(near-zero regularization), indicating instability on small samples.
Lasso's L1 penalty aggressively zeros irrelevant features, preventing
overfitting while retaining interpretability.

**Why not PCA:** PCA destroys feature interpretability — principal
components have no economic meaning. An investment team needs to know
*which* factors drive SSSG predictions, not just that some linear
combination of features is predictive.

**Why not Random Forest:** Tree-based methods require substantially more
data to learn meaningful decision boundaries. With n=12, Random Forest
performed at or below baseline in all configurations.

### 3.3 Iterative Feature Selection

I used an iterative Lasso-based feature selection process:

```
Step 1: All 19 features → Lasso selects 4, MAE=6.33%, R²=0.141
Step 2: 10 fundamentals only → Lasso selects 5, MAE=4.84%, R²=0.425
Step 3: 5 selected + guidance_cut → Lasso selects 5, MAE=4.55%, R²=0.481
Step 4: Remove digital_mix_qoq (redundant with digital_revenue_mix_pct)
        → MAE=4.07%, R²=0.518
Step 5: Remove auv_qoq → MAE=3.63%, R²=0.614 ← FINAL MODEL
```

**Key insight from Step 1:** Adding alt data (Google Trends, Reddit, news)
to fundamentals *worsened* model performance (MAE: 6.33% vs. 4.84%),
confirming these signals lack independent predictive power at quarterly
frequency with n=12.

**Key insight from Step 4:** `digital_revenue_mix_pct` (absolute level)
and `digital_mix_qoq` (rate of change) appeared redundant — removing
the rate-of-change feature improved performance, suggesting the absolute
level already captures the relevant signal.

### 3.4 Final Model

**Model:** Lasso Regression (alpha=0.5126, selected via LassoCV with LOOCV)

**Features:**

| Feature | Coefficient (standardized) | Direction | Interpretation |
|---------|--------------------------|-----------|----------------|
| digital_revenue_mix_pct | -5.94 | ↓ | Rising digital mix → lower next-Q SSSG |
| sssg_pct_lag2 | -2.78 | ↓ | High base 2Q ago → harder comps |
| margin_qoq | +2.02 | ↑ | Margin improvement → stronger demand |
| guidance_cut | -0.94 | ↓ | Guidance cut → further deceleration |

**Performance:**

| Metric | Value | vs. Baseline |
|--------|-------|-------------|
| LOOCV MAE | 3.63% | -3.49pp (49% improvement) |
| LOOCV RMSE | 5.28% | — |
| LOOCV R² | 0.614 | vs. 0.000 |

**Model limitations:**
- n=12: all results are directional, not statistically definitive
- Struggles at structural inflection points (largest errors at Q4 2022,
  Q4 2023, Q4 2024 — all regime-change quarters)
- No holdout test set; LOOCV is the only available validation method
- Alt data signals lack power at quarterly frequency; higher-frequency
  data (weekly credit card transactions, daily foot traffic) would
  significantly improve performance

---

## 4. Earnings Call NLP Analysis

### 4.1 Approach

I analyzed 11 earnings call transcripts (Q2 2023–Q4 2025) sourced from
Capital IQ using two complementary sentiment models:

**VADER** (rule-based): Fast, no training required. Weakness: does not
understand financial jargon ("navigating a fog" scores near-neutral).

**FinBERT** (ProsusAI/finbert): BERT model fine-tuned on financial text.
Correctly interprets domain-specific language ("headwind", "cautious
outlook", "miss"). Applied via Hugging Face `transformers` pipeline with
chunking to handle the 512-token limit.

**Why both:** When VADER and FinBERT agree on direction, conviction is
higher. Divergences reveal cases where financial context materially
changes the sentiment interpretation.

### 4.2 Transcript Processing

Each transcript was split into:
- **Prepared remarks:** Scripted management narrative (avg ~3,000 words)
- **Q&A:** Analyst questions and management responses (avg ~3,500 words)

Capital IQ transcripts use standard section markers ("Presentation" and
"Question and Answer") for splitting. Boilerplate legal disclaimers were
filtered using keyword exclusion before sentiment scoring.

Sentiment was computed by chunking text into ~400-word segments, scoring
each chunk with FinBERT, and averaging compound scores (positive = +confidence,
negative = -confidence, neutral = 0).

### 4.3 Key Technical Findings

**Lead-lag correlations with next-quarter SSSG:**

| NLP Signal | r (next-Q SSSG) | Direction | Interpretation |
|------------|----------------|-----------|----------------|
| vader_prepared | -0.502 | Negative | More positive scripted tone → lower next-Q SSSG |
| caution_signals | -0.441 | Negative | More caution keywords → lower next-Q SSSG |
| value_pricing | +0.431 | Positive | More value/pricing discussion → higher next-Q SSSG |
| vader_qa | +0.361 | Positive | More positive Q&A → higher next-Q SSSG |

**The contrarian prepared remarks signal (r=-0.502):** Management's
scripted prepared remarks sentiment is negatively correlated with
next-quarter SSSG. This reflects a systematic pattern where management
maintains positive scripted language even as fundamentals deteriorate —
most evident in Q2 2025 (FinBERT prepared = 0.716, actual SSSG = 2.1%).

**The Prepared vs Q&A gap:** FinBERT Q&A scores are consistently below
prepared remarks scores across all 11 quarters, confirming that analyst
questioning draws out more cautious responses than the scripted narrative
suggests. The gap is largest in Q3 2024 (-0.641) and Q2 2025 (-0.644),
both significant quarters from an investment perspective.

**VADER vs FinBERT divergence:** VADER assigns near-uniform scores to
prepared remarks (~0.36 across all quarters), failing to differentiate
between strong and weak quarters. FinBERT shows meaningful variation
(range: 0.422–0.720), correctly identifying Q3–Q4 2024 as the most
positively-toned period and Q4 2023 as relatively more cautious.

### 4.4 Alternative Data Synthesis

The NLP analysis provides qualitative validation of the quantitative model's
conclusions rather than replacing it:

- Caution keyword frequency peaked in Q3 2025 (5.79/1000w) — consistent
  with continued SSSG deceleration into Q4 2025
- Value/pricing discussion elevated in Q4 2025 (3.55/1000w vs. avg 2.73)
  — consistent with management's competitive repositioning narrative
- Q4 2025 NLP signals (lower caution, higher value/pricing, improving
  gc_ratio) are more constructive than the quantitative model's -0.3%
  base case, suggesting a potential recovery narrative is forming

---

## 5. What More Time & Resources Would Add

**Data:**
- Weekly credit card transaction data (Earnest Research, Second Measure):
  the highest-value addition — enables real-time SSSG nowcasting by
  decomposing traffic vs. ticket size at weekly frequency
- Daily foot traffic data (Placer.ai): independent validation of in-store
  visit trends, providing an early read on the traffic vs. digital mix dynamic
- Store-level geographic and demographic data (US Census): enables unit
  expansion quality modeling — predicting which markets support strong AUVs

**Modeling:**
- With more data: Bayesian structural time series (BSTS) or state-space
  models to explicitly model regime changes (the model's current weakness)
- Earnings transcript entity and topic modeling (LDA or BERTopic) to
  identify specific themes management discusses more/less over time
- Multi-task learning: jointly predicting SSSG, AUV, and margins to
  exploit correlations between targets

**Validation:**
- True out-of-sample holdout test (requires more historical quarters)
- Peer company benchmarking: apply the same model to Chipotle, Sweetgreen,
  and Wingstop to validate whether the identified features are
  industry-wide or CAVA-specific

---

*All code available in the accompanying GitHub repository.
Key dependencies: pandas, numpy, scikit-learn, statsmodels, pdfplumber,
transformers (FinBERT), vaderSentiment, pytrends, praw, finnhub-python.*
