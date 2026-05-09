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

### 5. Short-Term Sentiment Volume Stability (测试中)

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

### 6. Debt Trend Health

- **Hypothesis** Debt of a firm represents the amount of loans it has taken. Lower debt generally means lower financing risk; therefore, firms with decreasing debt tend to be healthier (all else equal), so we long them, and short firms with rising debt.（降债偏多、加债偏空。）
- **Implementation** Measure medium-term debt change and negate it so debt reduction gets higher score: `rank(-ts_delta(debt,90))`.（`90` 日抓中期资产负债表变化，先做方向再 rank。）
- **Hints to Implement** Debt levels vary a lot across sectors and update at low frequency; apply sector neutralization and test shorter/longer change windows to balance responsiveness and noise.

`rank(-ts_delta(debt,90))` → `rank(-ts_delta(debt,21))` (先做行业内比较，降低结构偏置。）

**思路（如何优化）**

- 先固定中性化：`Sector Neutralization`，避免把“行业资本结构差异”误当 alpha。
- 再扫时间窗：`63 / 90 / 126`，看哪一档在 `Sharpe-Fitness-Turnover` 上更平衡。
- 若子宇宙掉得快：加流动性门槛（如 `volume/adv20 > 0.6`）或提高 `Decay` 抑制低流动性噪声。
- 若覆盖不足或跳变明显：可对 debt 先做 backfill（若平台可用）再 `ts_delta`。

Settings（建议）: `USA | TOP3000 | Decay 4~6 | Delay 1 | Truncation 0.08 | Neutralization Sector | Pasteurization On`.

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

`-ts_corr(est_ptp,est_fcf,252)` → `-ts_corr(est_ptp,est_fcf,21)`（短窗更快，通常更贴近“过度定价后回归”节奏。）

Settings（from source）: `USA | TOP3000 | Decay 0 | Delay 1 | Truncation 0.08 | Neutralization Market | Pasteurization On`.
![[Pasted image 20260508145248.png]]

### 9. Volatility arbitrage

- **Hypothesis** Higher volatility is often observed in bearish regimes, while lower realized volatility with relatively high implied volatility may indicate stronger forward bullish sentiment.（隐波相对高、历波相对低时偏多。）
- **Implementation** Build implied-vs-realized volatility ratio: long when implied volatility significantly exceeds historical volatility, short the opposite.
- **Hints to Implement** Use `ts_backfill` to reduce missing-data breaks in option-implied volatility fields.

`implied_volatility_call_120/parkinson_volatility_120` → `ts_backfill(implied_volatility_call_120,120)/parkinson_volatility_120`（先补齐隐波缺失，减少 NaN 导致的权重断裂。）

Settings（from source）: `USA | TOP200 | Decay 0 | Delay 1 | Truncation 0.08 | Neutralization Sector | Pasteurization On`.

### 10. Implied Volatility Spread as a predictor

- **Hypothesis** If call open interest is higher than put open interest, upside sentiment may dominate; the larger call-put IV spread, the stronger potential upside (and vice versa).（看涨持仓占优时，IV spread 更有方向信息。）
- **Implementation** Use `trade_when` with `pcr_oi_270 < 1` as participation filter, then trade on `(implied_volatility_call_270 - implied_volatility_put_270)`.
- **Hints to Implement** Try custom neutralization based on self-created groups (e.g., historical volatility buckets) to improve sub-universe robustness.

`trade_when(pcr_oi_270 < 1, (implied_volatility_call_270-implied_volatility_put_270), -1)` → `group_neutralize(trade_when(pcr_oi_270 < 1, (implied_volatility_call_270-implied_volatility_put_270), -1), bucket(rank(ts_std_dev(returns,60)), range="0,1,0.1"))`（按历史波动分桶中性化，减少结构偏置。）

Settings（from source）: `USA | TOP3000 | Decay 4 | Delay 1 | Truncation 0.08 | Neutralization Market | Pasteurization On`.

### 11. 6-Month Call-Put Volatility Skew

- **Hypothesis** When call IV is higher than put IV relative to average ATM IV, options traders may be focusing more on upside than downside risk.（6个月 call-put skew 越高，偏多情绪越强。）
- **Implementation** Use normalized skew: `(implied_volatility_call_180 - implied_volatility_put_180) / implied_volatility_mean_180`.
- **Hints to Implement** Preprocess with `ts_backfill()` to pass weight/coverage tests; reduce turnover via smoothing/decay.

`(implied_volatility_call_180-implied_volatility_put_180)/implied_volatility_mean_180` → `ts_decay_linear((ts_backfill(implied_volatility_call_180,60)-ts_backfill(implied_volatility_put_180,60))/max(ts_backfill(implied_volatility_mean_180,60),0.01),8)`（先补缺失，再加平滑降换手。）

Settings（from source）: `USA | TOP3000 | Decay 0 | Delay 0 | Truncation 0.08 | Neutralization Subindustry | Pasteurization On`.

### 12. 5-Day Peer vs. Stock Performance Gap

- **Hypothesis** If peer group outperforms the stock over 5 days, the stock may be a short-term laggard with mean-reversion upside.（同伴涨得更多而个股滞后，短期补涨概率上升。）
- **Implementation** Compare 5-day compounded returns of peers (`rel_ret_all`) vs stock (`returns`) and trade the gap.
- **Hints to Implement** Use `trade_when` so trades happen only when the gap is meaningful, to reduce noise turnover.

`cum_rel_return-cum_return` → `gap = cum_rel_return-cum_return; trade_when(abs(gap) > ts_std_dev(gap,60), gap, -1)`（只在显著偏离时交易。）

Settings（from source）: `USA | TOP3000 | Decay 0 | Delay 1 | Truncation 0.08 | Neutralization Sector | Pasteurization On`.

### 13. Investing for the Future

- **Hypothesis** Firms with persistently rising long-term investment are more likely to generate future profit growth.（长期投资趋势上行，未来盈利能力可能更强。）
- **Implementation** Build a yearly investment series from `fnd6_newqv1300_ivltq` via backfill+sum, then estimate trend using `ts_regression(..., ts_step(1), 756, rettype=2)`.
- **Hints to Implement** Boost signal by giving more weight to firms with recently improving revenue.

`ts_regression(ts_sum(ts_backfill(fnd6_newqv1300_ivltq,60),252),ts_step(1),756,rettype=2)` → `ts_regression(ts_sum(ts_backfill(fnd6_newqv1300_ivltq,60),252),ts_step(1),756,rettype=2) * rank(ts_delta(ts_backfill(revenue,60),63))`（投资趋势 * 近期收入改善权重。）

Settings（from source）: `USA | TOP3000 | Decay 0 | Delay 1 | Truncation 0.08 | Neutralization Subindustry | Pasteurization On`.

### 14. Free Cash Flow Quality and Inventory Efficiency Signal

- **Hypothesis** Firms with persistently strong operating cashflow relative to capex should outperform; if inventory efficiency also improves sharply, the edge may strengthen.（高质量 FCF + 库存效率改善，偏多。）
- **Implementation** Use `est_cashflow_op - est_capex` as FCF-quality proxy, normalize over time, then smooth with `ts_decay_linear`.
- **Hints to Implement** Amplify signal when inventory turnover improves materially vs last year.

`ts_decay_linear(ts_scale(est_cashflow_op,252),22)-ts_decay_linear(ts_scale(est_capex,252),22)` → `base = ts_decay_linear(ts_scale(est_cashflow_op,252),22)-ts_decay_linear(ts_scale(est_capex,252),22); base * (1 + (ts_delta(inventory_turnover,252) > 0.5 ? 1 : 0))`（库存周转同比显著改善时放大。）

Settings（from source）: `USA | TOP3000 | Decay 2 | Delay 1 | Truncation 0.08 | Neutralization Industry | Pasteurization On`.

### 15. Bull Trap

- **Hypothesis** When short-horizon first-minute reaction trend is deteriorating but a large upside spike appears, that spike may be a bull trap.（短线反应斜率转弱 + 当日冲高，易是假突破。）
- **Implementation** Compute 5-day slope of `news_pct_1min` using `ts_regression(..., rettype=2)`, combine with recent `news_max_up_ret`, then winsorize extremes.
- **Hints to Implement** Reduce turnover by smoothing or event-gating.

`slope = ts_regression(ts_backfill(news_pct_1min,60), ts_step(1), 5, rettype=2); winsorize(-ts_backfill(news_max_up_ret,60) * abs(slope),std = 4)` → `slope = ts_regression(ts_backfill(news_pct_1min,60), ts_step(1), 5, rettype=2); trap = winsorize(-ts_backfill(news_max_up_ret,60) * abs(slope),std=4); ts_decay_linear(trap,5)`（加衰减降换手。）

Settings（from source）: `USA | TOP3000 | Decay 0 | Delay 1 | Truncation 0.08 | Neutralization Industry | Pasteurization On`.