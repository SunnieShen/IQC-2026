### 1. Operating Earnings Yield
- **Hypothesis** If the operating income of a company is currently higher than its past 1 year history, buy the company’s stock and vice-versa.（相对自身约一年盈利位置做多空。）
- **Implementation** Using `ts_rank` to identify current performance of the company compared to its own history, using the fundamental data field `operating_income`.（252d≈一年；季报低频，披露间常平台。）
- **Hints to Implement** Rather than comparing the value directly, can calculating a ratio that includes stock market moves, improve the signal?

`ts_rank(operating_income,252)` → `ts_rank(operating_income/cap,252)`.（分子营业利润，分母 `cap` 纳入市值/价格，更接近 earnings yield。）

### 2. Appreciation of liabilities（负债公允价值）
- **Hypothesis** An increase in the fair value of liabilities could indicate a higher cost than expected. This may deteriorate the company's financial health, potentially leading to lower profitability or financial distress.（负债公允价值上升偏空。）
- **Implementation** Go short when there is an increase in the fair value of liabilities within a year and long when the opposite occurs using fundamental data.（用 L1 年度侧字段刻画一年内水平。）
- **Hints to Implement** Could observing the increase over a shorter period improve accuracy?

`-ts_rank(fn_liab_fair_val_l1_a,252)` → `-ts_rank(fn_liab_fair_val_l1_a,126)`.（更短窗更敏感、更噪，可对拍。）

### 3. Power of leverage
- **Hypothesis** Companies with high liability-to-asset ratios – excluding those with poor financial health or weak cashflows – often leverage debt as a strategic tool to pursue aggressive growth initiatives. By effectively utilizing financial leverage, these firms are more likely to deliver outsized returns, as they reinvest borrowed capital into high-potential opportunities.（高杠杆在排除差健康度后或对应扩张；需自行 filter。）
- **Implementation** Use the `liabilities` and `assets` to design the ratio.（核心：`liabilities/assets`。）
- **Hints to Implement** This ratio can vary significantly across industries. Would it be worth considering alternative neutralization settings?

`liabilities/assets` → `group_rank(liabilities/assets,industry)`.（跨行业尺度与结构差异大，行业 rank 后再交易。）

### 4. Earnings Yield Momentum
- **Hypothesis** Stocks whose earnings yield has been high more often over the last quarter, relative to their own history, may be undervalued thus we should long them.（近段相对自身历史更常高 earnings yield → 偏多。）
- **Implementation** Use EPS-to-price ratio as earnings yield proxy, compare over its own past, and compare it within its industry.（`est_eps/close`，先时序再行业。）
- **Hints to Implement** Use NAN HANDLING to preprocess data and boost the performance.

`ts_rank(est_eps/close,60)` → `group_rank(ts_rank(est_eps/close,60),industry)`.（行业横截面；缺失值按平台规则预处理后再跑。）

### 5. Short-Term Sentiment Volume Stability
- **Hypothesis** A high 10-day standard deviation of sentiment volume for a stock means that investor attention is unstable, with frequent spikes and drops in how much the stock is discussed. This unstable attention is often driven by short-lived news or hype and may lead to noisy, unsustainable price moves, causing the stock to underperform afterward.（讨论热度波动大 → 后或跑输。）
- **Implementation** Take the 10-day rolling standard deviation of relative sentiment volume `scl12_buzz` and negate it.（高 std → 取负偏好稳定。）
- **Hints to Implement** Would observing stability over a shorter horizon be more effective for more liquid stocks?

`-ts_std_dev(scl12_buzz,10)` → `-ts_std_dev(scl12_buzz,5)`.（高流动性可试更短 horizon。）

**Quick check（附录）** 日频门控 vs 财报披露节奏；尺度用比率 / `ts_rank` / `group_rank`；NaN；`Decay` / `Truncation` 与流动性过滤。
