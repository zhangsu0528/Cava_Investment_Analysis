# CAVA Group (CAVA) — Consumer Sector Investment Analysis

## Investment Thesis

CAVA is the leading fast-casual Mediterranean restaurant chain in the US, with a differentiated brand, strong unit economics, and a long runway for expansion. This analysis argues for a **positive outlook** on CAVA with **high conviction**, driven by three data-supported observations:

1. **Same-restaurant sales growth (SSSG) is reaccelerating** — after a soft Q1 2024 (2.3%), SSSG recovered sharply to 18.1% in Q3 and 21.2% in Q4 2024, well above consensus expectations.
2. **Google Trends search interest is trending up**, suggesting consumer awareness and demand continue to expand ahead of reported financials.
3. **Reddit sentiment around CAVA is predominantly positive**, reflecting strong brand affinity among the health-conscious demographic that drives CAVA's core customer base.

---

## Analysis Framework

```
Financial Time Series  ──┐
                          ├──► Same-Store Sales Growth Forecast  ──► Investment Recommendation
Google Trends          ──┤         ▲
                          │         │ leading indicator validation
Reddit Sentiment       ──┘         │
                                    │
Earnings Transcript NLP ───────────┘
```

**Direction:** Positive  
**Magnitude:** Large  
**Conviction:** High  

---

## Repository Structure

```
cava-investment-analysis/
├── README.md
├── requirements.txt
├── .gitignore
├── notebooks/
│   ├── 01_data_collection.ipynb     # Pull financial, Google Trends, Reddit data
│   ├── 02_modeling.ipynb            # Time series forecasting of SSSG
│   └── 03_earnings_nlp.ipynb        # Earnings transcript sentiment analysis
├── data/
│   ├── raw/                         # Raw CSVs (gitignored)
│   └── processed/                   # Cleaned, merged datasets
└── outputs/
    └── figures/                     # All charts and exhibits
```

---

## Data Sources

| Source | What We Use It For | Access |
|--------|-------------------|--------|
| CAVA SEC Filings (10-K/10-Q) | Quarterly SSSG, AUV, restaurant count, margins | Free — SEC Edgar |
| yfinance | Daily stock price history | Free — Python library |
| Google Trends (pytrends) | Weekly consumer search interest, leading indicator | Free — Python library |
| Reddit (PRAW + VADER) | Social media sentiment from food/investing communities | Free — Reddit API |
| Earnings transcripts | Management tone & forward guidance via NLP | Free — SEC Edgar 8-K |

---

## How to Run

### 1. Clone the repo
```bash
git clone https://github.com/zhangsu0528/cava-investment-analysis.git
cd cava-investment-analysis
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
```

### 3. Set up credentials
Create a `.env` file in the root directory:
```
REDDIT_CLIENT_ID=your_client_id
REDDIT_CLIENT_SECRET=your_client_secret
REDDIT_USER_AGENT=cava_sentiment_scraper/0.1
```
To get Reddit credentials: https://www.reddit.com/prefs/apps

### 4. Run notebooks in order
```
01_data_collection.ipynb  →  02_modeling.ipynb  →  03_earnings_nlp.ipynb
```

---

## Key Findings

*(To be updated as analysis progresses)*

- SSSG forecast for Q1 2025: **TBD**
- Google Trends lead-lag vs SSSG: **TBD**
- Earnings transcript sentiment trend: **TBD**

---

## Requirements

See `requirements.txt`. Key libraries:
- `yfinance` — financial data
- `pytrends` — Google Trends
- `praw` — Reddit API
- `vaderSentiment` — rule-based sentiment scoring
- `scikit-learn`, `statsmodels` — modeling
- `pandas`, `numpy`, `matplotlib`, `seaborn` — data processing & visualization
