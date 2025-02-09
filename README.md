# Quant Research Project – Pairs Trading Strategy on Nifty and Bank Nifty

**Author:** Arsh Singh

---

## Overview

This project implements and compares two pairs trading strategies based on implied volatility (IV) data of the Nifty and Bank Nifty indices. The objective is to test the research thesis that, because these two indices share significant overlap in constituents and market conditions, their volatilities should also be closely linked. Any temporary divergence in their volatilities can be exploited using a pairs trading approach.

**Key Deliverables:**

1. **Base Model:** A standard z-score based trading system using the spread defined as
   
$$
\\text{Spread} = \\text{Bank Nifty IV} - \\text{Nifty IV}
$$
   
   and a profit/loss (P/L) calculation given by
   
$$
\\text{P/L} = \\text{Spread} \\times (\\text{Time To Expiry})^{0.7}
$$
   
3. **Enhanced Model:** An improved model that modifies the spread definition by incorporating the effect of time-to-expiry (TTE) directly into the spread. In this model, the “weighted spread” is defined as
   
$$
W = (\\text{Bank Nifty IV} - \\text{Nifty IV}) \\times (\\text{TTE})^{0.7}
$$
   
   This weighting implies that when TTE is high, the profit potential per unit deviation is larger. In our implementation, we also adjust the trade size by effectively scaling it in proportion to $(\\text{TTE})^{0.7}$. Moreover, a robust, modified z-score is computed using the median and the median absolute deviation (MAD) to reduce the influence of outliers.
5. **Comparison and Optimization:** We compare the two models with the goal of optimizing absolute P/L, Sharpe Ratio, and Drawdown. In addition, we document experiments with linear regression-based approaches (which yielded unsatisfactory results) and discuss the potential benefits of a Kalman Filter-based pairs trading strategy. However, we chose to develop our own modifications rather than directly employing a premade solution.

---

## Dataset

The dataset is provided as `data.parquet` and includes:

- **Minute-level implied volatilities (IVs)** for both Bank Nifty and Nifty.
- **Time To Expiry (TTE)** for the options series.
- Data is recorded during Indian market trading hours (09:15–15:30).
- There are missing values and holiday effects (days where IVs do not change), which are handled in preprocessing.

---

## Project Structure

- **Preprocessing:**\
  The `preprocess_data` function:

  - Forward-fills missing values.
  - Filters the dataset to trading hours (09:15–15:30).
  - Removes holiday days (defined as days when both Bank Nifty and Nifty values remain constant).

- **Base Model (Z-score Trading System):**\
  The `calculate_spread_and_zscore` function computes:

  - The spread as the simple difference $S = \\text{BankNifty IV} - \\text{Nifty IV}$ .
  - Rolling mean and standard deviation (over a specified window, e.g. 1875 minutes representing one full trading week).
  - Standard z-score signal for entry/exit decisions.

  The `generate_positions` function creates trade signals based on these z-scores. The `calculate_pnl` function then calculates the P/L using the given formula:

$$
\\text{P/L} = S \\times (\\text{TTE})^{0.7}
$$

  Positions are assumed to be fixed (unit notional), making performance metrics directly comparable across trades.

- **Enhanced Model (Weighted Spread & Robust Z-score):**\
  The enhanced method modifies the spread calculation to account for TTE by defining:

$$
W = (\\text{BankNifty IV} - \\text{Nifty IV}) \\times (\\text{TTE})^{0.7}
$$

  This effectively means that if TTE is higher, the “spread” is magnified. In practical terms, this is analogous to buying more units of the asset when the profit equation is more favorable. In addition, rather than using the conventional mean and standard deviation, a robust z-score is computed using the rolling median and median absolute deviation (MAD) as follows:

$$
\\text{Modified Z-score} = 0.6745 \\times \\frac{W - \\text{median}(W)}{\\text{MAD}(W)}
$$

  The rest of the functions (`generate_positions_new`, `calculate_pnl_new`, `extract_trades_new`) follow a similar structure but use the enhanced spread. Note that TTE is also used to adjust the effective quantity (i.e. trading units are scaled by $(\\text{TTE})^{0.7})$ , which further refines the P/L calculation.

- **Trade Extraction & Metrics:**\\
  Discrete trade events are extracted (using `extract_trades` or `extract_trades_new`) to record trade entry/exit times, net profit, and invested capital (using the absolute raw_pnl at entry as a proxy for capital deployment). Performance metrics (total P/L, Sharpe Ratio, maximum drawdown, average invested value, etc.) are then computed. For the Sharpe ratio, we assume weekly returns (using an annualization factor based on 375 minutes/day and 5 trading days/week).

- **Parameter Optimization:**\\
  Although not detailed in the code snippets here, we explored different parameter settings. We found that parameters such as an entry threshold of approximately 1.75 and an exit threshold of about 0.75 yielded better profit at a higher risk level.

- **Alternate Methods Explored:**\\
  We also experimented with a linear regression-based approach to estimate a dynamic relationship between Bank Nifty and Nifty IVs (including a TTE term). However, this approach did not yield satisfactory results. Additionally, we came across Kalman Filter-Based Pairs Trading strategies, which appeared promising. Nonetheless, we decided against directly implementing these prepackaged models in favor of developing our own modifications.

---

## Assumptions

- **Trading Hours:** Data is considered only during 09:15 to 15:30.
- **Holidays:** Days where both Bank Nifty and Nifty IVs remain constant are removed.
- **Lookback Window:** For rolling statistics and signal generation, a window of 1875 minutes (approx. one trading week) is used. This parameter is adjustable.
- **P/L Formula:** The base P/L is computed as $\\text{P/L} = \\text{Spread} \\times (\\text{TTE})^{0.7}$ . For the enhanced model, the spread is weighted by $(\\text{TTE})^{0.7}$ .
- **Capital Deployment:** For trade-level performance, the invested capital is proxied by the absolute raw_pnl at trade entry.
- **Sharpe Ratio:** Calculated on a weekly return basis, with an annualization factor derived from the assumption of 375 minutes/day and 5 trading days/week.
- **Trade Signal Generation:** The strategy enforces that a new trade is only entered after the previous trade is closed, with immediate reversals allowed if the signal strongly flips.
- **Parameter Tuning:** Preliminary backtesting indicated that entry thresholds around 1.75 and exit thresholds around 0.75 may deliver improved performance, albeit with higher risk.

---

## Results & Findings

**Base Model (Z‑Score Strategy):**

- **Total P&L:** 198.90 (units)
- **Sharpe Ratio:** 1.48 (weekly returns)
- **Max Drawdown:** 3.55 (units), representing 1.78% drawdown
- **Average Invested Value:** 0.73 (units)
- **Total Trades:** 575
- **Observations:**  
  The base model generated many small spikes in P&L. Given the fixed notion per trade, the performance metrics reflect stable but modest returns.

**Enhanced Model (Weighted Spread with Robust Z‑Score):**

- **Total P&L:** 4742.18 (units)
- **Sharpe Ratio:** 2.16 (weekly returns)
- **Max Drawdown:** 35.04 (units), representing 0.74% drawdown
- **Average Invested Value:** 5.96 (units)
- **Total Trades:** 485
- **Observations:**  
  In the enhanced model, fewer but significantly larger spikes were observed. By incorporating TTE into the spread and effectively scaling the trade size (buying units proportional to $(\\text{TTE})^{0.7}$ , the model achieved markedly higher profits. The robust modified z‑score (using median and MAD) reduced noise and outlier effects, yielding a higher Sharpe ratio and lower relative drawdown compared to the base model.

**Other Methods Considered:**

- **Linear Regression-Based Spread:**  
  We attempted to estimate a dynamic hedge ratio using linear regression (with Bank Nifty IV as a function of Nifty IV and $(\\text{TTE})^{0.7}$ . However, this approach did not improve performance relative to our enhanced method.
- **Kalman Filter-Based Strategy:**  
  A Kalman Filter approach for dynamic hedge ratio estimation appeared theoretically promising but was not directly implemented. We opted to develop our own enhancements rather than adopt a prebuilt solution, emphasizing our ability to innovate.

---

## Mathematical Explanation

Let:

$$
S_t = \\text{BankNifty IV}_t - \\text{Nifty IV}_t\\
$$

$$
T_t = (\\text{TTE}\\_t)^{0.7}\\
$$

**Base Model:**

- **Spread:** $S_t$
- **P/L per minute:** $\\text{P/L}_t = S_t \\times T_t$

**Enhanced Model:**

- **Weighted Spread:**
  
$$
W_t = S_t \\times T_t
$$

- **Robust Z-Score Computation:**\
  Instead of using the rolling mean $(\\mu)$ and standard deviation $(\\sigma)$ of $(W_t)$, we use the rolling median $(m)$ and MAD:

$$
\\text{Modified Z-score}_t = 0.6745 \\times \\frac{W_t - m}{\\text{MAD}}
$$

  This scaling constant (0.6745) ensures that, for a normally distributed dataset, the modified z-score is comparable to the conventional z-score.
- **Position Sizing:**\
  Positions are generated based on the robust z-score. Moreover, since the profit equation is now $(W_t)$ rather than $(S_t)$, the actual number of “units” traded is effectively scaled by $(T_t)$. In other words, a trade with a longer TTE implies a larger exposure because:

$$
\\text{Effective Units} \\propto T_t
$$

- **Cumulative P/L:**\
  The net profit is computed as the cumulative change in the weighted spread, adjusted by the position held.

---

## Final Remarks

- **Flexibility of Parameters:**  
  The lookback window, entry threshold, and exit threshold are all tunable parameters. In our experiments, we observed that thresholds around 1.75 for entry and 0.75 for exit deliver higher profits at a higher risk level.
- **Performance Improvement:**  
  The enhanced model, which incorporates TTE adjustments and uses a robust (median/MAD) z‑score, outperforms the base z‑score model in terms of both absolute P&L and risk-adjusted metrics.
- **Innovation over Prebuilt Models:**  
  While alternatives like linear regression and Kalman Filter-based strategies were considered, our approach—developed in-house—demonstrated superior performance relative to the base model while showcasing creative problem-solving.

---

## How to Run the Code

1. **Dependencies:**

   - Python 3.x
   - Pandas, NumPy, Matplotlib, Statsmodels (if using regression-based methods)
   - The dataset (`data.parquet`)

2. **Execution:**

   - Clone the repository.
   - Ensure the dataset is placed in the project directory or modify the file path in the code accordingly.
   - Run the main script (or Jupyter Notebook) to process the data, generate trading signals, compute P/L, extract trade details, and output performance metrics and plots.

3. **Output:**
   - CSV files: e.g., `Zoutput.csv` for the base model and `NewOutput.csv` for the enhanced model; similarly, trade details are output in `Ztrades.csv` and `NewTrades.csv`.
   - Plots illustrating cumulative P&L, the spread, and z‑score over time.

---

## Summary of Results

- **Base Model:**  
  Total P&L: 198.90, Sharpe Ratio: 1.48, Max Drawdown: 3.55 units (1.78%), Average Invested Value: 0.73, Total Trades: 575.
- **Enhanced Model:**  
  Total P&L: 4742.18, Sharpe Ratio: 2.16, Max Drawdown: 35.04 units (0.74%), Average Invested Value: 5.96, Total Trades: 485.

The enhanced model, which factors in TTE through a weighted spread and employs a robust modified z‑score, results in fewer but larger trades. This leads to significantly improved absolute and risk-adjusted performance. Parameter exploration indicated that using an entry threshold of approximately 1.75 and an exit threshold of 0.75 produces even better profit potential, albeit at a higher risk level.

---

## Conclusion

This project demonstrates a methodical approach to developing and enhancing a pairs trading strategy. By starting with a simple z-score model and then extending it to incorporate time-to-expiry effects and robust statistics, we have shown that creative modifications can lead to significant performance improvements. While alternative methods such as linear regression and Kalman Filter-based strategies were explored, the enhanced model presented here was chosen for its simplicity, transparency, and superior empirical results. This work not only meets the assignment’s requirements but also provides insights into how additional market dynamics can be effectively captured in a trading strategy.
