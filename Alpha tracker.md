# Alpha Tracker

> 编号规则：每个母因子从 `.0` 开始；衍生因子按 `.1`、`.2`、`.3` 递增。  
> 1.x 是 CLV 系列，2.x 是行业中性反转系列。

## 1.x CLV + 市场状态

### 1.0 

```python
clv = ((close-low)-(high-low))/(high-low);
trade_when(volume/adv20>0.6, ts_rank(-clv, 250) * ts_sum(returns>0? 1:0, 252), -1)
```

- `clv`“收盘位置系数（Close Location Value）”：`clv = ((close-low)-(high-low))/(high-low)`；收盘越靠近低位，`-clv` 越大，反转做多倾向越强。
- `sig = ts_rank(-clv, 250)`：对“收盘位置系数”做 250 日时序排序，慢信号；抗噪、风格稳定，but 新行情响应偏慢。
- `reg = ts_sum(returns>0?1:0, 252)`：近 252 日正收益日数（市场强弱计数器）；强市放大信号，弱市收缩信号。
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

- `Fitness` 可近似看成“==收益质量 / 交易噪声==”；==稳定收益、低无效交易 -> Fitness 上升==。
- 1.3 核心：`alpha = sig * reg / vol_adj`；表达式已带降噪，平台 `Decay/Truncation` 属于二次平滑和去尾。
- `Decay` 链条：上调 -> 更平滑 -> ==换手==降 -> 成本降；但过高 -> ==信号钝化 -> Sharpe/Returns 下滑== -> Fitness 下滑。
- 你的样本：`Decay=9`、`Turnover=38.03%`、`Fitness=0.70`；典型“降噪过头，信号被削”。
- `Truncation` 链条：变小（更严）-> 尾部风险降；但过严 -> 强信号被截断 -> 收益弹性降 -> Fitness 降。
- 联动结论：`高 Decay + 严 Truncation` 易“没信号”；`低 Decay + 松 Truncation` 易“高噪声”。
- 实操顺序：固定 `Truncation=0.06` 扫 `Decay=6/7/8`；再固定最佳 Decay 扫 `Truncation=0.05/0.06/0.07`；仍 `<1` 再微调表达式窗口/门槛。
- 快速判据：`Turnover <45%` 且 `Fitness <1` -> 过平滑；`Turnover >65%` 且 `Fitness <1` -> 噪声过高。
- 目标区间：`Turnover 50%~60%`，`Fitness > 1.0`。

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
sig = decay_linear(-ts_zscore(group_neutralize(clv, subindustry), 120), 6);
liq = rank(volume/adv20);
vol_adj = 1 + ts_std_dev(returns, 30);
trade_when(volume/adv20 > 0.6, sig * liq / vol_adj, -1)
```

- `group_neutralize + decay_linear`：保留平滑主线；市场因子化降低。
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
core = -ts_zscore(group_neutralize(returns, subindustry), 90);
size_tilt = signed_power(rank(assets/cap), 0.5);
trade_when(volume/adv20 > 0.8, core * size_tilt, -1)
```

- `ts_rank(250)` -> `ts_zscore(90)`：速度更快，响应更灵敏。
- `signed_power(rank(assets/cap), 0.5)`：压缩极值票影响。
- 相比 2.0：同源但不同暴露路径；相关性通常下降。
- `Decay=4, Truncation=0.07`
- `Decay=5, Truncation=0.07`

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
trade_when(volume/adv20 > 0.6, decay_linear(core * size_tilt * liq, 5), -1)
```

- `zscore(45)` + `decay_linear(..., 5)`：快反转执行层风格。
- `size_tilt` 和 `liq` 保留；表达式内衰减抑制噪声抖动。
- 相比 2.0：时间频率、执行风格都分叉；更易满足相关性约束。
- `Decay=6, Truncation=0.06`
- `Decay=5, Truncation=0.07`

## 回测筛选建议（简版）

1. 每组先用 `.0` 做基线，再比较 `.1~.5`（或 `.1~.4`）的增量。
2. 与母因子相关性建议保留在 `0.35~0.68`，避免超过平台提交阈值 `0.70`。
3. 同时看 Sharpe、Fitness、Turnover、容量，避免只优化单一指标。
4. 最后从 1.x 和 2.x 各选 1~2 条做小组合，看稳定性是否提升。

