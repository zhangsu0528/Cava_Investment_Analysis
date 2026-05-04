# CAVA Group (NYSE: CAVA) — Consumer Sector Investment Analysis

**Date:** May 2026 | **Current Price:** $90.98 | **Sector:** Consumer — Restaurants

---

## Investment Recommendation

**Direction:** Cautious | **Magnitude:** Medium | **Conviction:** Medium

CAVA Group is the leading fast-casual Mediterranean restaurant chain in the US, with a differentiated brand and a long runway for unit expansion. However, data-driven analysis reveals a structural deceleration in same-restaurant sales growth (SSSG) that the market may not have fully priced in.

Key findings:
- **Q1 2026 SSSG Base Case Forecast: -0.3%** (±3.6% MAE) — below management's FY2026 guidance of 3–5%
- SSSG has decelerated from a peak of 28.4% (Q1 2023) to 0.5% (Q4 2025)
- Three consecutive earnings misses in 2025, accompanied by two rounds of guidance cuts
- Earnings call NLP analysis provides a more mixed read — cautiously optimistic signals from Q4 2025 transcript

---

## Analysis Framework

```
Quarterly Fundamentals ──┐
                          ├──► Lasso SSSG Forecast ──► Investment Recommendation
Google Trends          ──┤         ▲                        ▲
Reddit Sentiment       ──┤         │ feature selection       │ qualitative validation
News Sentiment (NLP)   ──┤         │                         │
Analyst Actions        ──┘         │               Earnings Transcript NLP
                                    │               (VADER + FinBERT)
```

---

## Repository Structure

```
cava-investment-analysis/
├── README.md
├── requirements.txt
├── .env                              # credentials (gitignored)
├── .gitignore
│
├── data_preprocessing/
│   ├── 00_data_collection.ipynb     # Financial data, Google Trends, Reddit, News
│   └── 01_preprocessing.ipynb       # Cleaning, feature engineering, master dataset
│
├── modeling/
│   ├── 02_modeling.ipynb            # Lasso SSSG forecasting model
│   └── 03_earnings_nlp.ipynb        # Earnings transcript NLP (VADER + FinBERT)
│
├── data/
│   ├── raw/                         # Raw data files (gitignored)
│   └── processed/                   # Cleaned datasets
│       ├── cava_master_quarterly.csv
│       ├── cava_quarterly_fundamentals.csv
│       ├── cava_google_trends_monthly.csv
│       ├── cava_reddit_combined_sentiment.csv
│       └── cava_earnings_nlp.csv
│
├── outputs/
│   ├── figures/                     # All charts and exhibits
│   ├── final_model_results.csv
│   ├── loocv_predictions.csv
│   └── q1_2026_forecast.csv
│
├── CAVA_Investment_Writeup.md       # Investment recommendation write-up
└── CAVA_DataScience_Writeup.md      # Data science methodology write-up
```

---

## Data Sources

| Source | What I Use It For | Access |
|--------|------------------|--------|
| CAVA SEC Filings (10-K/10-Q) | Quarterly SSSG, AUV, restaurant count, margins | Free — SEC Edgar / ir.cava.com |
| yfinance | Daily stock price, analyst upgrades/downgrades | Free — Python library |
| Google Trends (pytrends) | Weekly consumer search interest | Free — Python library |
| Reddit Public API | Social media sentiment (r/wallstreetbets, r/investing, r/food) | Free — no auth required |
| Finnhub API | Financial news headlines and sentiment | Free tier |
| Capital IQ | Earnings call transcripts (Q2 2023–Q4 2025) | Institutional access |

---

## Modeling Approach

### SSSG Forecasting (02_modeling.ipynb)

**Model:** Lasso Regression with Leave-One-Out Cross Validation (LOOCV)

**Final features (selected via iterative Lasso regularization):**

| Feature | Coefficient | Interpretation |
|---------|------------|----------------|
| digital_revenue_mix_pct | -5.94 | Rising digital mix signals traffic softness |
| sssg_pct_lag2 | -2.78 | High base 2Q ago creates harder comps |
| margin_qoq | +2.02 | Margin improvement signals demand health |
| guidance_cut | -0.94 | Management guidance cuts predict deceleration |

**Performance:**
- LOOCV MAE: **3.63%** (vs. naive baseline 7.12% — 49% improvement)
- LOOCV R²: **0.614**

**Q1 2026 Forecast:**
- Base case: **-0.3%** (±3.6%)
- Bear case (guidance cut): **-2.9%**
- Management FY2026 guidance: 3–5%

### Earnings Call NLP (03_earnings_nlp.ipynb)

Analyzed 11 earnings call transcripts (Q2 2023–Q4 2025) using:
- **VADER**: Rule-based sentiment baseline
- **FinBERT** (ProsusAI/finbert): BERT fine-tuned on financial text

**Key NLP finding:** `vader_prepared` sentiment is the strongest NLP leading indicator of next-quarter SSSG with r = -0.502 (negative). When management uses the most positive scripted language, next-quarter SSSG tends to disappoint — a contrarian signal.

---

## How to Run

### 1. Clone the repo
```bash
git clone https://github.com/zhangsu0528/cava-investment-analysis.git
cd cava-investment-analysis
```

### 2. Create virtual environment
```bash
python -m venv venv
source venv/bin/activate  # Mac/Linux
pip install -r requirements.txt
```

### 3. Set up credentials
Create a `.env` file in the root directory:
```
FINNHUB_KEY=your_finnhub_api_key
NEWSAPI_KEY=your_newsapi_key
```
- Finnhub API (free): https://finnhub.io/register
- NewsAPI (free): https://newsapi.org/register

### 4. Add earnings transcripts
Place Capital IQ PDF transcripts in `data/earnings_transcripts/`:
```
CAVA Group Inc._Earnings Call_2023-08-15_English.pdf
CAVA Group Inc._Earnings Call_2023-11-07_English.pdf
... (Q2 2023 through Q4 2025)
```

### 5. Run notebooks in order
```
data_preprocessing/00_data_collection.ipynb
data_preprocessing/01_preprocessing.ipynb
modeling/02_modeling.ipynb
modeling/03_earnings_nlp.ipynb
```

---

## Key Results

| Analysis | Finding |
|----------|---------|
| SSSG trend | Decelerated from 28.4% (Q1 2023) to 0.5% (Q4 2025) |
| Best SSSG predictor | Digital revenue mix (r = -0.59 with next-Q SSSG) |
| Model performance | LOOCV MAE = 3.63%, R² = 0.614 |
| Q1 2026 forecast | -0.3% base case (below mgmt guidance of 3–5%) |
| NLP leading indicator | vader_prepared (r = -0.502 with next-Q SSSG) |
| Alt data verdict | Google Trends, Reddit, news lack power at quarterly frequency |
| Investment stance | Cautious — downside risk to consensus SSSG estimates |