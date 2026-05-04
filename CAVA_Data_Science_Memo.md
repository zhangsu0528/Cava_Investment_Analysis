# CAVA Group (NYSE: CAVA) — Data Science Write-Up
**Date:** May 4, 2026 | **Author:** Su Zhang

---

## 1. Data Sources & Evaluation

**Primary Data: CAVA Quarterly Fundamentals**  
Source: SEC filings, earnings press releases. Coverage: 15 quarters (Q2 2022–Q4 2025); 12 usable after lag construction and target shifting. Key variables: SSSG%, AUV, restaurant-level margin, digital revenue mix, net new openings, adjusted EBITDA margin, and management guidance events. Revenue was normalized to weekly revenue to account for CAVA's variable fiscal quarter lengths (Q1 = 16 weeks, Q2–Q4 = 12 weeks).

**Alternative Data Tested**

| Source | Coverage | Outcome |
|--------|----------|---------|
| Google Trends (pytrends) | Jan 2022–Apr 2026 | Too low-signal; CAVA search interest max = 2/100 |
| Reddit sentiment (VADER) | Sparse (~5 posts/quarter) | Qualitatively useful; insufficient for modeling |
| News sentiment (Finnhub) | Jan 2025–May 2026 | Only 3 quarters given free tier limitation; too short for time series |
| Analyst actions (yfinance) | Jul 2023–May 2026 | Zero sell ratings; r=-0.16 with SSSG |

All four were included in the initial feature set and zeroed out by Lasso regularization, suggesting limited independent predictive value at quarterly frequency with n=12.

**Earnings Call Transcripts**  
Source: Capital IQ (Q2 2023–Q4 2025). Transcripts were split into prepared remarks and Q&A using Capital IQ section markers; boilerplate disclaimers were filtered. Sentiment was scored using VADER and FinBERT. FinBERT was preferred because VADER assigns near-uniform scores across all quarters (~0.36), while FinBERT captures more variation in financial language. Key finding: prepared-remarks sentiment was negatively correlated with next-quarter SSSG (r=-0.502), suggesting that management tone can remain positive even as fundamentals deteriorate. Transcripts were used as a qualitative overlay, not a model input.

---

## 2. Model Specification & Feature Selection

**Target:** next-quarter SSSG (%).  
**Model:** Lasso regression with Leave-One-Out Cross Validation (LOOCV).

**Why Lasso:** With only 12 usable observations, standard linear regression overfits easily and Random Forest lacks sufficient data. Lasso's L1 regularization shrinks irrelevant coefficients to zero, performing automatic feature selection while preserving interpretability.

**Why LOOCV:** A standard 80/20 split yields only 2–3 test observations, which is too few for reliable evaluation. LOOCV trains on n-1 samples and tests on the remaining one, repeated n times, maximizing data use while providing an out-of-sample-style error estimate.

**Final 4 features:**

| Feature | Coefficient | Interpretation |
|---------|-------------|----------------|
| Digital revenue mix | -5.94 | Higher digital mix preceded weaker next-Q SSSG |
| SSSG lag (2Q ago) | -2.78 | Strong prior comps create harder comparison base |
| Margin QoQ change | +2.02 | Margin improvement may signal stronger demand, pricing power, or better store-level execution |
| Guidance cut flag | -0.94 | Guidance cuts aligned with further deceleration |

I treat digital mix as a directional signal, not a causal conclusion. Higher digital mix may proxy for traffic composition shifts, convenience-driven ordering, or broader demand softness rather than directly causing SSSG weakness.

---

## 3. Performance & Forecast

**LOOCV performance:**

| Metric | Final Model | Naive Baseline |
|--------|-------------|----------------|
| MAE | **3.63%** | 7.12% |
| RMSE | **5.28%** | 8.50% |
| R² | **0.614** | 0.000 |

**Q1 2026 forecast:**

| Scenario | SSSG |
|----------|------|
| **Base case** | **-0.3%** |
| Bear case (guidance cut) | -2.9% |
| ±MAE band | -4.0% to +3.3% |
| Management FY2026 guidance | 3.0–5.0% |

The forecast is driven by digital mix at a record 38.9% (bearish), Q4 2025 margin compression of -3.2pp QoQ (bearish), and no new guidance cut, which removes one negative event-based headwind. CAVA has never reported negative SSSG historically, so -0.3% should be read as a signal of limited recovery support rather than a precise point estimate. The more actionable takeaway is that the model sits below management’s FY2026 3–5% guidance range, suggesting downside risk if Q1 does not show a clear recovery.

---

## 4. Limitations & Further Work

**Limitations:**

- **Small sample (n=12):** Results are directional, not statistically definitive; confidence intervals are wide.
- **No true holdout:** LOOCV approximates out-of-sample performance but does not fully replicate true future out-of-sample testing.
- **Backward-looking:** The model cannot capture menu innovation, promotions, weather effects, or abrupt macro shifts.
- **Structural breaks:** The model is weakest at regime transitions, particularly between CAVA’s high-growth 2022–2024 period and its 2025 deceleration.
- **Guidance comparison:** The model predicts Q1 2026 SSSG, while management guidance is for FY2026; the comparison is directional rather than one-to-one.

**With more time and data:**

- Weekly credit card transaction data to decompose traffic vs. ticket.
- Daily foot traffic data to validate whether digital mix is linked to in-store demand softness.
- Peer-augmented panel model using Chipotle, Wingstop, and Texas Roadhouse to increase effective sample size.
- Store maturity data to separate new-unit contribution from mature-store productivity.

---

*Full code: github.com/zhangsu0528/cava-investment-analysis*