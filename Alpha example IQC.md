### 1. Operating Earnings Yield

- **Hypothesis** If the operating income of a company is currently higher than its past 1 year history, buy the company’s stock and vice-versa.（相对自身约一年盈利位置做多空。）
- **Implementation** Using `ts_rank` to identify current performance of the company compared to its own history, using the fundamental data field `operating_income`.（252d≈一年；季报低频，披露间常平台。）
- **Hints to Implement** Rather than comparing the value directly, can calculating a ratio that includes stock market moves, improve the signal?

`ts_rank(operating_income,252)` → `ts_rank(operating_income/cap,252)`.（分子营业利润，分母 `cap` 纳入**市值**/价格，更接近 earnings yield。）

Sharpe1.42Turnover13.09%Fitness1.06Returns7.30%Drawdown6.66%Margin11.15‱

### 2. Appreciation of liabilities（负债公允价值）

- **Hypothesis** An increase in the fair value of liabilities could indicate a higher cost than expected. This may deteriorate the company's financial health, potentially leading to lower profitability or financial distress.（**负债公允价值**上升偏空。）
- **Implementation** Go short when there is an increase in the fair value of liabilities within a year and long when the opposite occurs using fundamental data.（用 L1 年度侧字段刻画一年内水平。）
- **Hints to Implement** Could observing the increase over a shorter period improve accuracy?

`-ts_rank(fn_liab_fair_val_l1_a,252)` → `-ts_rank(fn_liab_fair_val_l1_a,126)`.（更短窗**更敏感**、更噪，可对拍。）

![[Pasted image 20260506202956.png]]

### 3. Power of leverage

- **Hypothesis** Companies with high liability-to-asset ratios – excluding those with poor financial health or weak cashflows – often leverage debt as a strategic tool to pursue aggressive growth initiatives. By effectively utilizing financial leverage, these firms are more likely to deliver outsized returns, as they reinvest borrowed capital into high-potential opportunities.（高杠杆在排除差健康度后或对应扩张；需自行 filter。）
- **Implementation** Use the `liabilities` and `assets` to design the ratio.（核心：`liabilities/assets`。）
- **Hints to Implement** This ratio can vary significantly across industries. Would it be worth considering alternative neutralization settings?
`liabilities/assets` -> set neutralization = **industry**（跨行业尺度与结构差异大，行业 rank 后再交易。）
![[Pasted image 20260506203334.png]]

### 4. Earnings Yield Momentum

- **Hypothesis** Stocks whose **earnings yield** has been high more often over the last quarter, relative to their own history, may be undervalued thus we should long them.（**近段"预期每股收益"相对自身历史更常高 earnings yield** → 偏多。）
- **Implementation** Use EPS-to-price ratio as earnings yield proxy, compare over its own past, and compare it within its industry.（`est_eps/close`，先时序再行业。）
- **Hints to Implement** Use NAN HANDLING to preprocess data and boost the performance. EPS 数据缺失

`ts_rank(est_eps/close,60)` → `temp = group_backfill(est_eps/close, industry, 20); group_rank(ts_rank(temp, 60), industry)`
![[Pasted image 20260506210049.png]]

### 5. Short-Term Sentiment Volume Stability

- **Hypothesis** A high 10-day standard deviation of sentiment volume for a stock means that investor attention is unstable, with frequent spikes and drops in how much the stock is discussed. This unstable attention is often driven by short-lived news or hype and may lead to noisy, unsustainable price moves, causing the stock to underperform afterward.（讨论热度波动大 → 后或跑输。）
- **Implementation** Take the 10-day rolling standard deviation of relative sentiment volume `scl12_buzz` and negate it.（高 std → 取负偏好稳定。）
- **Hints to Implement** Would observing stability over a shorter horizon be more effective for more liquid stocks?
  - [Weight concentration](https://support.worldquantbrain.com/hc/en-us/articles/19248385997719) **10.25%** is above cutoff of **10%** on **10/3/2022**.
  - [Sub-universe Sharpe](https://support.worldquantbrain.com/hc/en-us/articles/6568644868375) of **0.83** is below cutoff of **0.92**.
  - **Issue A: Sub-universe Sharpe 不达标（核心）** 主宇宙有效但更高流动性子宇宙（如 TOP3000 -> TOP1000）衰减过大；官方门槛可写作 `sub_sharpe >= 0.75 * sqrt(sub_size/alpha_size) * alpha_sharpe`。
  - **Issue A 解决（优先顺序）** ① 去掉强 size/liquidity 倾斜乘子；② 流动性分层衰减（液体快、非液体慢）；③ 每次改动复查 `TOP3000/1000/500/200`，看子宇宙 Sharpe 是否同步改善。
  - **Issue B: Weight concentration 超限（核心）** 权重过度集中在少数股票/日期，常由极值、过短窗口、或流动性尾部暴露导致。
  - **Issue B 解决（优先顺序）** ① 降低极值影响（rank/zscore、适当提高 decay）；② 加流动性门控或分层；③ 调整 Truncation 与中性化，优先把单票峰值压回 cutoff 下。

`-ts_std_dev(scl12_buzz,10)` → `-ts_std_dev(scl12_buzz,5)`（短窗版，提升液体段响应）

`-ts_std_dev(scl12_buzz,8)` → `ts_decay_linear(-ts_std_dev(scl12_buzz,8),4)*rank(volume*close) + ts_decay_linear(-ts_std_dev(scl12_buzz,8),10)*(1-rank(volume*close))`（Sub-universe 常用修复）

`-ts_std_dev(scl12_buzz,8)` → `group_rank(-ts_std_dev(scl12_buzz,8),sector)`（降低集中度与结构偏置）

**Quick check（附录）** 日频门控 vs 财报披露节奏；尺度用比率 / `ts_rank` / `group_rank`；NaN；`Decay` / `Truncation` 与流动性过滤。

- **Idea:** Debt of a firm represents the amount of loans a firm has taken. Lower Debt generally means lower risk. Hence, a firm with decreasing debt tends to be a healthier firm (all things being equal) and we should go long on it and vice versa for a firm with increasing debt.
- **Expression:** Rank(-ts_delta(debt,90))
- **Settings:** Select TOP3000 with Sector Neutralization

### 7. Valuation based on cash flow

- **Hypothesis** A lower `EV/CF` usually suggests the company is becoming cheaper relative to its cash-generating ability; a higher multiple suggests it is getting more expensive.（`EV/CF` 下降偏多、上升偏空。）
- **Implementation** Use `ts_zscore` to standardize the time-series change of valuation ratio, then use `group_rank` to control cross-sectional turnover and industry structure.（先时序标准化，再行业横截面排序。）
- **Hints to Implement** There are various types of cash flow; switching cashflow type in the denominator may improve robustness and coverage. 

`group_rank(-ts_zscore(enterprise_value/cashflow,63),industry)` → `group_rank(-ts_zscore(enterprise_value/cashflow_dividends,63),industry)`**可分配、对股东真正可兑现**的现金能力，所以在估值里信息密度更高

![[Pasted image 20260507172642.png]]

Settings（from source）: `USA | TOP3000 | Decay 0 | Delay 1 | Truncation 0.08 | Neutralization Industry | Pasteurization On`.

### 8. Overpriced stocks

- **Hypothesis** When analyst price target estimates (`est_ptp`) and free cashflow estimates (`est_fcf`) move highly in sync, the market may have already priced in the cash flow story, leaving limited upside.（高正相关偏“已定价”，信号取负。）
- **Implementation** Use `ts_corr(est_ptp, est_fcf, window)` to capture the co-movement dynamics between target price and cashflow expectation.
- **Hints to Implement** A 1-year window can be too slow for correction trades; test shorter windows for faster reaction.

`-ts_corr(est_ptp,est_fcf,252)` → `-ts_corr(est_ptp,est_fcf,63)`（短窗更快，通常更贴近“过度定价后回归”节奏。）

Settings（from source）: `USA | TOP3000 | Decay 0 | Delay 1 | Truncation 0.08 | Neutralization Market | Pasteurization On`.

### 9. Volatility arbitrage

- **Hypothesis** Higher volatility is often observed in bearish regimes, while lower realized volatility with relatively high implied volatility may indicate stronger forward bullish sentiment.（隐波相对高、历波相对低时偏多。）
- **Implementation** Build implied-vs-realized volatility ratio: long when implied volatility significantly exceeds historical volatility, short the opposite.
- **Hints to Implement** Use `ts_backfill` to reduce missing-data breaks in option-implied volatility fields.

`implied_volatility_call_120/parkinson_volatility_120` → `ts_backfill(implied_volatility_call_120,120)/parkinson_volatility_120`（先补齐隐波缺失，减少 NaN 导致的权重断裂。）

Settings（from source）: `USA | TOP200 | Decay 0 | Delay 1 | Truncation 0.08 | Neutralization Sector | Pasteurization On`.