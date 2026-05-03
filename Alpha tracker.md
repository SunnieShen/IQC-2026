# Alpha Tracker

> 编号规则：每个母因子从 `.0` 开始；衍生因子按 `.1`、`.2`、`.3` 递增。  
> 1.x 是 CLV 系列，2.x 是行业中性反转系列，3.x 是 10 日价格反转系列，4.x 是简单价量候选系列。

## 1.x CLV + 市场状态

### 1.0

```python
clv = ((close-low)-(high-low))/(high-low);
trade_when(volume/adv20>0.6, ts_rank(-clv, 250) * ts_sum(returns>0? 1:0, 252), -1)
```

- `clv`“收盘位置系数（Close Location Value）”：`clv = ((close-low)-(high-low))/(high-low)`；收盘越靠近低位，`-clv` 越大，反转做多倾向越强。
- `sig = ts_rank(-clv, 250)`：对“收盘位置系数”做 250 日时序排序，慢信号；抗噪、风格稳定，but 新行情响应偏慢。
- `reg = ts_sum(returns>0?1:0, 252)`：近 252 日正收益日数（市场强弱计数器）；强市放大信号，弱市收缩信号。
  - 因此策略在顺风(reg大)期更激进，在逆风(reg小)期更保守
- 组合公式：`alpha_t = I(volume_t/adv20_t>0.6) * rank_t^{250}(-clv_t) * N_t^{up}(252)`，`N_t^{up}(252)=\sum_{i=0}^{251} 1[r_{t-i}>0]`。
- 数学理解：核心是“反转强度 × 市场状态权重”；`rank(-clv)` 给个股反转排序，`N^{up}` 给市场级放大系数。
- 风险点：`N^{up}` 未归一化时尺度会累积变大；可改 `N^{up}/252`，提升版本可比性与稳定性。
![[Pasted image 20260503185531.png]]
![[Pasted image 20260503185721.png]]
![[Pasted image 20260503185734.png]]

### 1.1

```python
clv = ((close-low)-(high-low))/(high-low);
vol_ok = (volume/adv20 > 0.6);
sig = -ts_zscore(clv, 120);
posRet = (returns > 0 ? returns : 0);
regime = ts_sum(posRet, 180);
trade_when(vol_ok, sig * (regime/180), -1)
```

- `sig = -ts_zscore(clv, 120)`：替代 `ts_rank(250)`；更快、更灵敏，滞后更小。
- `regime = ts_sum(posRet, 180)`：替代上涨日计数；对大阳线更敏感，趋势环境放大更明显。
- 相比 1.0：短中期机会更容易捕捉；稳定性更依赖 `Decay/Truncation`。
- 权衡：收益弹性上升，信号波动也上升。
![[Pasted image 20260503185811.png]]
![[Pasted image 20260503185851.png]]

### 1.2 标准 CLV 写法 + 概率门控

```python
clv = ((close-low)-(high-close))/max(high-low, 0.01*close);
sig = -ts_zscore(group_neutralize(clv, subindustry), 60);
reg = ts_mean(returns < 0 ? 1 : 0, 120);
liq = rank(volume/adv20);
trade_when(volume/adv20 > 0.7, sig * reg * liq, -1)
```

- `group_neutralize(clv, subindustry)`：市场共振 -> 行业内相对强弱。
- `reg = ts_mean(returns < 0 ? 1 : 0, 120)`：顺风门控 -> 逆风门控，弱市反转风格。
- `liq = rank(volume/adv20)`：流动性直接入权重，降低边缘交易影响。
- 相比 1.0：结构分叉更明显，相关性更容易压到 `<0.70`。
- `Decay=5, Truncation=0.07`
- `Decay=4, Truncation=0.08`

### 1.3 保留原风格的同源增强

```python
clv = ((close-low)-(high-close))/max(high-low, 0.01*close);
sig = ts_rank(-clv, 40);
reg = ts_mean(returns > 0 ? 1 : 0, 40);
vol_adj = 1 + ts_std_dev(returns, 20);
trade_when(volume/adv20 > 0.6, sig * reg / vol_adj, -1)
```

- `ts_rank(-clv, 40)`：慢反转 -> 中短反转，节奏明显提速。
- `/ (1 + ts_std_dev(returns, 20))`：高波动降权，抑制噪声暴露。
- 相比 1.0：时间结构差异大，相关性更易下降。
- 风险：窗口过短 + 参数过激，Fitness 易波动。
- `Decay=7, Truncation=0.06`
- `Decay=6, Truncation=0.07`

#### ==1.3 实验矩阵（参数待扫描, 6组，目标：Fitness > 1 且 Corr < 0.70）==

- G1（原 1.3 表达式）
  - 表达式：
  ```python
  clv = ((close-low)-(high-close))/max(high-low, 0.01*close);
  sig = ts_rank(-clv, 40);
  reg = ts_mean(returns > 0 ? 1 : 0, 40);
  vol_adj = 1 + ts_std_dev(returns, 20);
  trade_when(volume/adv20 > 0.6, sig * reg / vol_adj, -1)
  ```
  - 参数：`Decay=8, Truncation=0.06`
- G2（原 1.3 表达式，强风控）
  - 表达式同 G1
  - 参数：`Decay=9, Truncation=0.05`
- G3（原 1.3 表达式，中等风控）
  - 表达式同 G1
  - 参数：`Decay=8, Truncation=0.05`
- G4（降频版 1）
  - 表达式：
  ```python
  clv = ((close-low)-(high-close))/max(high-low, 0.01*close);
  sig = ts_rank(-clv, 60);
  reg = ts_mean(returns > 0 ? 1 : 0, 60);
  vol_adj = 1 + ts_std_dev(returns, 20);
  trade_when(volume/adv20 > 0.65, sig * reg / vol_adj, -1)
  ```
  - 参数：`Decay=7, Truncation=0.06`
- G5（降频版 1，强风控）
  - 表达式同 G4
  - 参数：`Decay=8, Truncation=0.05`
- G6（降频版 2，更稳）
  - 表达式：
  ```python
  clv = ((close-low)-(high-close))/max(high-low, 0.01*close);
  sig = ts_rank(-clv, 80);
  reg = ts_mean(returns > 0 ? 1 : 0, 80);
  vol_adj = 1 + ts_std_dev(returns, 20);
  trade_when(volume/adv20 > 0.65, sig * reg / vol_adj, -1)
  ```
  - 参数：`Decay=7, Truncation=0.05`
- 跑法顺序：`G1 -> G2 -> G3 -> G4 -> G5 -> G6`
- 保留标准：
  - `Fitness > 1.00`
  - 与 `1.0/1.1` 的相关性 `< 0.70`
  - 在满足前两条时，优先选择 Turnover 更低的版本

#### 1.3 提高 Fitness 的逻辑链条（Decay / Truncation）

> - `Neutralization`：按分组（如 `Subindustry`）做组内权重和为 0；行业共振暴露下降，纯个股信号占比上升。
> - `Decay`：对过去 N 天输入做==线性递减加权平均==；N 越大越==平滑==，换手越低，but 反应越慢
> - `Truncation`：==限制单日单票最大权重==；值越小越保守，尾部风险更低，but 强信号也更容易被截断。

- `Fitness` 可近似看成“==收益质量 / 交易噪声==”；==稳定收益、低无效交易 -> Fitness 上升==。
- 1.3 核心：`alpha = sig * reg / vol_adj`；表达式已带降噪，平台 `Decay/Truncation` 属于二次平滑和去尾。
- `Decay` 链条：上调 -> 更平滑 -> ==换手==降 -> 成本降；但过高 -> ==信号钝化 -> Sharpe/Returns 下滑== -> Fitness 下滑。
- 你的样本：`Decay=9`、`Turnover=38.03%`、`Fitness=0.70`；典型“降噪过头，信号被削”。
- `Truncation` 链条：变小（更严）-> 尾部风险降；但过严 -> 强信号被截断 -> 收益弹性降 -> Fitness 降。
- 联动结论：`高 Decay + 严 Truncation` 易“没信号”；`低 Decay + 松 Truncation` 易“高噪声”。
- 实操顺序：固定 `Truncation=0.06` 扫 `Decay=6/7/8`；再固定最佳 Decay 扫 `Truncation=0.05/0.06/0.07`；仍 `<1` 再微调表达式窗口/门槛。
- 快速判据：`Turnover <45%` 且 `Fitness <1` -> 过平滑；`Turnover >65%` 且 `Fitness <1` -> 噪声过高。
- 目标区间：`Turnover 50%~60%`，`Fitness > 1.0`。

#### 1.3 Neutralization / Universe 对 Fitness 的影响

- `Neutralization=Subindustry`：组内权重和为 0；行业共振暴露更低，个股 alpha 纯度更高。
- `Neutralization=Industry`：中性化更弱；信号保留更多，but 行业暴露更容易抬回撤。
- 对 1.3 的常见影响：`Subindustry` 更稳、回撤更低；`Industry` 有时收益更高，但 Fitness 不一定更高。
- `Universe=TOP3000`：覆盖更广，机会更多，but 噪声和摩擦也更多。
- `Universe=TOP2000/TOP1000`：流动性更好、噪声更低；常见是 Turnover/Drawdown 改善，Fitness 更容易上升。
- 代价：Universe 变小，容量和收益上限可能下降。

#### 1.3 四组最小测试表（先不改表达式）

- U1：`Neutralization=Subindustry`，`Universe=TOP3000`
- U2：`Neutralization=Subindustry`，`Universe=TOP2000`
- U3：`Neutralization=Industry`，`Universe=TOP3000`
- U4：`Neutralization=Industry`，`Universe=TOP2000`
- 其他参数固定：用你当前 1.3 最优 `Decay/Truncation`，`Delay=1` 不变。
- 比较顺序：先看 `Fitness`，再看 `Sharpe/Drawdown`，最后看 `Turnover`。
- 保留规则：`Fitness > 1` 且与 `1.0/1.1` 相关性 `<0.70`；同分时选 Turnover 更低版本。

### 1.4 极值增强版本

```python
clv = ((close-low)-(high-close))/max(high-low, 0.01*close);
sig = signed_power(-ts_zscore(clv, 90), 1.6);
down_reg = ts_mean(returns < 0 ? 1 : 0, 90);
trade_when(volume/adv20 > 0.8, sig * down_reg, -1)
```

- `signed_power(..., 1.6)`：强信号放大、弱信号压缩，极值特征更强。
- `down_reg`：顺风门控 -> 逆风门控，偏下跌环境反转。
- 相比 1.0：事件型反转更明显，相关性更容易拉开。
- 风险：尾部暴露更强，需更严 `Truncation`。
- `Decay=8, Truncation=0.05`
- `Decay=7, Truncation=0.06`

### 1.5 衰减降噪版本

```python
clv = ((close-low)-(high-close))/max(high-low, 0.01*close);
sig = ts_mean(-ts_zscore(group_neutralize(clv, subindustry), 120), 6);
liq = rank(volume/adv20);
vol_adj = 1 + ts_std_dev(returns, 30);
trade_when(volume/adv20 > 0.6, sig * liq / vol_adj, -1)
```

- `group_neutralize + ts_mean(..., 6)`：短窗平滑；BRAIN 无 `decay_linear` 时用 `ts_mean` 替代。
- `liq` + `vol_adj`：交易可行性和风险同时入模。
- 相比 1.0：由“门控增强”变为“风险调整后反转”，结构差异更大。
- 风险：表达式已衰减，平台 `Decay` 过高会过平滑。
- `Decay=3, Truncation=0.08`
- `Decay=4, Truncation=0.07`

## 2.x 行业中性反转 + 规模倾向系列

### 2.0 原始母因子

```python
trade_when(volume/adv20>0.8,
  ts_rank(-group_neutralize(returns,subindustry),250) * assets/cap,
  -1)
```

- 公式：`alpha = ts_rank(-group_neutralize(returns, subindustry), 250) * (assets/cap)`。
- 逻辑：行业内反转 + 价值倾向；估值更便宜的票权重更高。
- `group_neutralize`：先去行业共振，避免行业 beta 伪 alpha。
- 时间结构：`250` 日慢信号；稳定，but 风格切换响应慢。
- 优点：可解释性强，适合作为 2.x 母因子。
- 风险：`assets/cap` 极值放大；需 `rank` 或幂次压缩控尾部。
![[Pasted image 20260503190426.png]]
![[Pasted image 20260503190446.png]]

### 2.1 秩化规模倾向（防极值）

```python
core = -ts_zscore(group_neutralize(returns, subindustry), 120);
size_tilt = signed_power(rank(assets/cap), 0.4);
vol_adj = 1 + ts_std_dev(returns, 20);
liq = rank(volume/adv20);
trade_when(volume/adv20 > 0.85, core * size_tilt * liq / vol_adj, -1)
```

- `zscore` 窗口 `90 -> 120`：降短期噪声，回撤压力更小。
- `size_tilt` 幂次 `0.5 -> 0.4`：进一步压极值暴露，减少单票冲击。
- 新增 `vol_adj` + `liq`：高波动降权，边缘流动性降权。
- 相比旧 2.1：更稳、更防噪，目标是抬 `Sharpe/Fitness`。
- `Decay=5, Truncation=0.06`
- `Decay=6, Truncation=0.05`

### 2.2 zscore 响应增强

```python
core = ts_rank(-group_neutralize(returns, subindustry), 60);
liq = rank(volume/adv20);
reg = ts_mean(returns < 0 ? 1 : 0, 120);
trade_when(volume/adv20 > 0.7, core * liq * reg, -1)
```

- 窗口 `250/180` 级别 -> `60`；慢反转 -> 中短反转。
- 乘子 `assets/cap` -> `liq * reg`；价值倾向 -> 流动性 + 弱市门控。
- 相比 2.0：经济含义分叉更大，低相关补充更有效。
- `Decay=5, Truncation=0.07`
- `Decay=6, Truncation=0.06`

### 2.3 非线性压缩规模暴露

```python
core = ts_rank(-group_neutralize(returns, subindustry), 180);
size_tilt = signed_power(rank(assets/cap), 0.7);
vol_adj = 1 + ts_std_dev(returns, 30);
trade_when(volume/adv20 > 0.8, core * size_tilt / vol_adj, -1)
```

- 保留 `assets/cap` 倾向；同源性不丢。
- 增加 `vol_adj`；高噪声行情降权。
- 相比 2.0：纯放大 -> 风险调整后放大；曲线更稳。
- `Decay=3, Truncation=0.08`
- `Decay=4, Truncation=0.08`

### 2.4 连续流动性加权

```python
liq = rank(volume/adv20);
core = -ts_zscore(group_neutralize(returns, subindustry), 45);
size_tilt = rank(assets/cap);
trade_when(volume/adv20 > 0.6, ts_mean(core * size_tilt * liq, 5), -1)
```

- `zscore(45)` + `ts_mean(..., 5)`：快反转 + 短窗平滑；无 `decay_linear` 时用 `ts_mean`。
- `size_tilt` 和 `liq` 保留；短窗均值抑制噪声抖动。
- 相比 2.0：时间频率、执行风格都分叉；更易满足相关性约束。
- `Decay=6, Truncation=0.06`
- `Decay=5, Truncation=0.07`

## 3.x 10 日价格反转系列

### 3.0 原始母因子

```python
-1 * (close - ts_delay(close, 10))
```

- 核心逻辑：过去 10 日涨太多则反转偏空，跌太多则反转偏多。
- 风险点：直接使用价格差，天然偏向高价股；更适合作为想法原型，不建议直接提交。

### 3.1 10 日收益反转

```python
ret10 = close / ts_delay(close, 10) - 1;
sig = -ts_zscore(ret10, 60);
trade_when(volume/adv20 > 0.7, sig, -1)
```

- `ret10`：把价格差改成 10 日收益率，去掉价格尺度问题。
- `ts_zscore(ret10, 60)`：用 60 日历史分布衡量当前 10 日涨跌是否极端。
- 相比 3.0：逻辑不变，但横截面更公平，结果更稳定。
- `Decay=5, Truncation=0.07`
- `Decay=4, Truncation=0.07`

### 3.2 放量增强版

```python
ret10 = close / ts_delay(close, 10) - 1;
sig = -ts_zscore(ret10, 60);
liq = rank(volume/adv20);
trade_when(volume/adv20 > 0.8, sig * liq, -1)
```

- `liq = rank(volume/adv20)`：成交量越异常，价格冲击越可能过度，反转权重越高。
- 相比 3.1：增加价量确认，可能提高收益弹性，但 Turnover 也可能上升。
- `Decay=5, Truncation=0.06`
- `Decay=6, Truncation=0.06`

### 3.3 行业内 10 日反转

```python
ret10 = close / ts_delay(close, 10) - 1;
sig = -ts_zscore(group_neutralize(ret10, subindustry), 60);
trade_when(volume/adv20 > 0.7, sig, -1)
```

- `group_neutralize(ret10, subindustry)`：去掉行业共振，只保留行业内相对超涨/超跌。
- 相比 3.1：通常更稳，行业 beta 暴露更低。
- `Decay=4, Truncation=0.07`
- `Decay=5, Truncation=0.07`

### 3.4 高波动降权版

```python
ret10 = close / ts_delay(close, 10) - 1;
sig = -ts_zscore(ret10, 60);
vol_adj = 1 + ts_std_dev(returns, 20);
trade_when(volume/adv20 > 0.7, sig / vol_adj, -1)
```

- `vol_adj`：高波动股票降权，降低噪声和尾部风险。
- 相比 3.1：收益弹性可能下降，但 Fitness 和回撤稳定性可能更好。
- `Decay=4, Truncation=0.08`
- `Decay=5, Truncation=0.07`

### 3.5 趋势环境过滤

```python
ret10 = close / ts_delay(close, 10) - 1;
sig = -ts_zscore(ret10, 60);
regime = ts_mean(returns > 0 ? 1 : 0, 120);
trade_when(volume/adv20 > 0.7, sig * regime, -1)
```

- `regime`：近期上涨环境更好时，超跌反弹更容易延续；弱环境下降低权重。
- 相比 3.1：从纯反转变成“有环境过滤的反转”。
- `Decay=5, Truncation=0.07`
- `Decay=6, Truncation=0.06`

#### 3.x 跑法建议

- 优先顺序：`3.3 -> 3.1 -> 3.2 -> 3.4 -> 3.5`
- 若 `Turnover` 过高：优先提高 `Decay` 或把 `ret10` 改成 `ret15`。
- 若 `Fitness` 不够但 Sharpe 还行：优先试 `3.3` 的行业中性版本，或给 `3.2` 降低成交量门槛到 `0.7`。

## 4.x 简单优质价量候选系列

### 4.0 行业内 10 日反转

```python
ret10 = close / ts_delay(close, 10) - 1;
sig = -ts_zscore(group_neutralize(ret10, subindustry), 60);
trade_when(volume/adv20 > 0.7, sig, -1)
```

- 逻辑：行业内超涨回落、超跌反弹。
- 特点：简单、稳定，适合作为 4.x 的稳健基线；与 3.3 同式，可优先只保留表现更好的编号。
- `Decay=4, Truncation=0.07`

### 4.1 VWAP 偏离反转

```python
disloc = (close - vwap) / vwap;
sig = -ts_zscore(group_neutralize(disloc, subindustry), 40);
trade_when(volume/adv20 > 0.8, sig, -1)
```

- `disloc`：收盘价相对当日成交均价的偏离。
- 逻辑：收盘价明显高于 VWAP，短期偏过热；明显低于 VWAP，短期偏恐慌。
- 优点：交易含义清晰，比裸收益反转更像“价量失衡”。
- `Decay=5, Truncation=0.07`
- `Decay=4, Truncation=0.08`

### 4.2 放量 5 日反转

```python
ret5 = close / ts_delay(close, 5) - 1;
vol_shock = rank(volume/adv20);
sig = -ts_zscore(ret5, 40);
trade_when(volume/adv20 > 0.8, sig * vol_shock, -1)
```

- `ret5`：更短周期的价格冲击。
- `vol_shock`：放量越明显，短期过度反应权重越高。
- 相比 4.0：节奏更快，收益弹性更高，但噪声也更高。
- `Decay=4, Truncation=0.06`
- `Decay=5, Truncation=0.06`

### 4.3 价格涨跌与量能衰减背离

```python
ret20 = close / ts_delay(close, 20) - 1;
vol_chg = volume / ts_mean(volume, 20);
sig = -ts_zscore(ret20, 60) * ts_rank(-vol_chg, 20);
trade_when(volume/adv20 > 0.6, sig, -1)
```

- 逻辑：价格涨很多但量能不跟，反转风险更高；价格跌很多但量能萎缩，抛压可能衰竭。
- 风险：`ts_rank(-vol_chg, 20)` 会偏好缩量，需观察容量和成交过滤是否足够。
- `Decay=6, Truncation=0.06`
- `Decay=7, Truncation=0.05`

### 4.4 量价冲击过度反应

```python
shock = returns * rank(volume/adv20);
sig = -ts_zscore(shock, 40);
trade_when(volume/adv20 > 0.8, sig, -1)
```

- 逻辑：大成交量配合大涨跌，常见短期过度反应。
- 优点：表达式非常短，适合作为低复杂度候选。
- `Decay=4, Truncation=0.07`
- `Decay=5, Truncation=0.06`

### 4.5 低波动反转

```python
ret10 = close / ts_delay(close, 10) - 1;
sig = -ts_zscore(ret10, 60);
risk = 1 + ts_std_dev(returns, 20);
trade_when(volume/adv20 > 0.7, sig / risk, -1)
```

- 逻辑：同样的超涨超跌，高波动票更噪，低波动票反转更干净。
- 相比 3.4：同类思路，可作为 3.4 的复跑或参数替代。
- `Decay=4, Truncation=0.08`
- `Decay=5, Truncation=0.07`

### 4.6 振幅异常反转

```python
day_range = (high - low) / close;
sig = -ts_zscore(returns, 20) * ts_rank(day_range, 20);
trade_when(volume/adv20 > 0.8, sig, -1)
```

- `day_range`：日内振幅，捕捉情绪冲击。
- 逻辑：大振幅日的涨跌更可能是短期情绪过度，后续容易回归。
- `Decay=5, Truncation=0.06`
- `Decay=6, Truncation=0.06`

### 4.7 稳健动量

```python
mom60 = close / ts_delay(close, 60) - 1;
vol = ts_std_dev(returns, 20);
sig = ts_zscore(mom60, 120) / (1 + vol);
trade_when(volume/adv20 > 0.7, sig, -1)
```

- 逻辑：中期强者恒强，但高波动趋势降权。
- 作用：和短反转系列方向不同，可用于测试组合互补性。
- `Decay=8, Truncation=0.05`
- `Decay=7, Truncation=0.06`

### 4.8 短反转 + 中趋势过滤

```python
ret5 = close / ts_delay(close, 5) - 1;
mom60 = close / ts_delay(close, 60) - 1;
sig = -ts_zscore(ret5, 40) * ts_rank(mom60, 120);
trade_when(volume/adv20 > 0.7, sig, -1)
```

- 逻辑：只在中期趋势较好的股票里做短期回调反转，偏“趋势中买回撤”。
- 相比纯短反转：可能减少逆大趋势交易，但会引入动量暴露。
- `Decay=5, Truncation=0.07`
- `Decay=6, Truncation=0.06`

#### 4.x 跑法建议

- 优先顺序：`4.1 -> 4.0 -> 4.5 -> 4.4 -> 4.8 -> 4.2`
- 质量筛选：先看 `Fitness > 1`，再看与 1.x/2.x/3.x 的相关性是否 `<0.70`。
- 若 `4.0` 与 `3.3` 重复，只保留表现更好的一个；`4.1` 更适合做独立母因子。

## 回测筛选建议（简版）

1. 每组先用 `.0` 做基线，再比较 `.1~.5`（或 `.1~.4`）的增量。
2. 与母因子相关性建议保留在 `0.35~0.68`，避免超过平台提交阈值 `0.70`。
3. 同时看 Sharpe、Fitness、Turnover、容量，避免只优化单一指标。
4. 最后从 1.x 和 2.x 各选 1~2 条做小组合，看稳定性是否提升。

