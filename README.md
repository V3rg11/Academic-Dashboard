# Adaptive Time-Series Performance & Error Diagnostic Model

A Google Sheets statistical forecasting and academic performance diagnostic workbook. It tracks session-by-session test scores, categorizes lost marks, and predicts future performance using adaptive exponential smoothing, Holt's linear trend method, anomaly-weighted ensembles, and expanding-window multiple linear regression.(The template is setup for Edexcel International Advanced Level subjects/units, but its entirely customisable)

## Workbook Architecture

The workbook is organized into four interconnected sheets forming a sequential data pipeline:

| **`Sheet1`** | **Data Entry & Normalization** | Primary input ledger for session scores, unit tags, and categorized mark deductions. Supports up to 5 customisable categories of mark deduction. Calculates normalized scores ($X_t$) and pulls live forecasts back from `Sheet2`. |

| **`Sheet2`** | **Statistical Backend** | Mathematical formulae computing rolling volatility, dynamic smoothing parameters ($\alpha_t, \beta_t$), Holt's trend, EWMA baselines, z-score anomaly weights, and two expanding-window `LINEST` regression models. |

| **`Sheet3`** | **Forecast & Anomaly Dashboard** | Side-by-side comparison of actual vs. predicted scores (`EWMA Base` vs. `Weighted Regression`), charts, and a filtered anomaly table flagging high-residual sessions alongside their error breakdowns. |

| **`Sheet4`** | **Executive & Unit Summary** | High-level KPI overview, global error taxonomy distribution, and an interactive unit-level diagnostic filter identifying the single largest source of lost marks per unit. |

---

## How It Works

### 1. Input & Error Taxonomy (`Sheet1`)
Users log each assessment session chronologically across columns `A:J`:
* **Identifiers & Raw Scores:** `Session` (`A`), `Unit` (`B`), `Marks Scored` (`C`), and `Total Unit Marks` (`D`).
* **Error Categorization (`E:I`):** Lost marks are decomposed into five distinct failure modes:
  1. `Computational Error`
  2. `Setup Misunderstanding`
  3. `Unseen Problem`
  4. `Modelling Error`
  5. `No Idea How to Model`
* **Normalization (`K`):** Raw scores are standardized to a $[0, 1]$ scale:
  $$X_t = \frac{\text{Marks Scored}}{\text{Total Unit Marks}}$$
* **Live Predictions (`L:M`):** Dynamically filters and displays `EWMA Base Prediction` and `Weighted Regression Prediction` from `Sheet2` for all active sessions as these two yield the most accurate results for academic test scores.

---

### 2. Statistical Forecasting Engine (`Sheet2`)
`Sheet2` ingests $X_t$ (`A2:A100`) and initializes over a warm-up window (rows `2–5`) before running full predictive calculations from **Row 6 (Session 5)** onward:

1. **Rolling Volatility ($\sigma_t$, Col `B`):** 5-session moving sample standard deviation:
   $$\sigma_t = \text{STDEV}(X_{t-4}:X_t)$$
2. **Adaptive Smoothing Parameters ($\alpha_t, \beta_t$, Cols `C:D`):**
   * **Level Smoothing ($\alpha_t$):** Scales dynamically between $0.2$ and $0.8$ relative to peak historical volatility so the model reacts faster during turbulent periods:
     $$\alpha_t = 0.2 + 0.6 \left(\frac{\sigma_t}{\max_{6 \le i \le t} \sigma_i}\right)$$
   * **Trend Smoothing ($\beta_t$):** Scales dynamically with session-to-session score jumps:
     $$\beta_t = 0.1 + 0.4 |X_t - X_{t-1}|$$
3. **Component Forecasts (Cols `E:J`):**
   * **Holt's Linear Method (`E:G`):** Tracks smoothed level $L_t = \alpha_t X_t + (1 - \alpha_t)(L_{t-1} + T_{t-1})$ and trend $T_t = \beta_t(L_t - L_{t-1}) + (1 - \beta_t)T_{t-1}$, producing forecast $G_t = L_t + T_t$.
   * **Adaptive EWMA (`H`):** $S_t = \alpha_t X_t + (1 - \alpha_t)S_{t-1}$.
   * **Recent Mean (`I`):** 3-session simple moving average $M_t = \frac{1}{3}(X_{t-2} + X_{t-1} + X_t)$.
   * **Holt Base Ensemble (`J`):** Weighted blend $J_t = 0.5 G_t + 0.3 S_t + 0.2 M_t$.
4. **Anomaly Detection & Weighting (Cols `K:L, O`):**
   * **Standardized Residual ($Z_t$):** $Z_t = \frac{X_t - J_t}{\sigma_t}$ (clamped to $[-2.5, 2.5]$ during rows `6–15`).
   * **Outlier Down-Weighting ($w_t$):** $w_t = e^{-|Z_t|}$, which dampens sudden anomalous drops/spikes in the `Weighted Ensemble` ($O_t = w_t J_t + (1 - w_t) S_{t-1}$).
5. **Final Predictive Models (Cols `P:R`):**
   * **`EWMA Base` (`P`):** Trend- and mean-reversion-adjusted EWMA baseline:
     $$\hat{X}_{t,\text{EWMA}} = S_t + 0.2 T_t + 0.1(M_t - S_t)$$
     *(Note: $T_t$ is clamped to $[-0.1, 0.1]$ for rows `6–15`.)*
   * **`Regression EWMA` (`Q`):** Starting at Row 16 (Session 15), fits an expanding-window ordinary least squares (OLS) regression via `LINEST` predicting $(X_t - S_t)$ from trend $T_t$ and prior `EWMA Base` values.
   * **`Weighted Regression` (`R`):** Starting at Row 16, applies Weighted Least Squares (WLS) by scaling variables by $\sqrt{w_t}$ (`Cols X:Z`: $\sqrt{w_t}(X_t - S_t)$, $\sqrt{w_t}T_t$, and $\sqrt{w_t}(M_t - S_t)$) before running `LINEST`. This prevents outlier sessions from skewing regression slope coefficients.

---

### 3. Visualization & Diagnostics (`Sheet3` & `Sheet4`)
* **Performance Tracking (`Sheet3`):**
  * Plots `Actual Score`, `Predicted Score (EWMA)`, and `Predicted Score (Weighted Regression)` across sessions (Line & Bar charts).
  * Extracts session numbers, absolute residuals $|Z_t|$, and corresponding error category counts (`E:I`) into a dedicated anomaly inspection table (`H2:N100`).
* **Summary & Unit Drill-Down (`Sheet4`):**
  * Displays overall metrics (`Current Overall Average`, `Total Marks Lost Ever`) and an **Error Index** calculating total and percentage marks lost per error category, visualized with a **Radar Chart**.
  * Includes a dropdown selector (`B9`: e.g., `Unit 4`, `Unit 5`, `Unit 6`) that dynamically computes the unit's average score (`AVERAGEIFS`) and identifies the `Worst Error for this Unit` (`INDEX/MATCH` over `SUMIFS`).

---

## Usage & Compatibility Notes

1. **Input Data:** Enter session details in `Sheet1` starting at Row 2. Ensure `Total Unit Marks` (`D`) is greater than `0` to avoid `#DIV/0!` errors in normalized score calculations.
2. **Minimum Session Thresholds(Can be changed):**
   * **Sessions 1–4 (Rows `2–5`):** Initialization period; predictions default to the actual normalized score.
   * **Sessions 5–14 (Rows `6–15`):** `EWMA Base` predictions become active.
   * **Session 15+ (Row `16+`):** `Weighted Regression` predictions activate once enough observations accumulate for `LINEST`.
