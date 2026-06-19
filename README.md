# alpha-f5-member-position-concentration

商品期货会员持仓集中度 Alpha（Factor ID: F5）。计算前5大净多/净空席位的持仓保证金集中度，生成横截面 buy/sell/hold 信号。

## 目录结构

```
alpha-f5-member-position-concentration/        开发产物（计算、验证、回测）
  scripts/
    factor.py       因子计算，输出 Parquet 结果
    validate.py     无未来函数检测、字段完整性验证
    backtest.py     IC/ICIR/ARR/MDD/多空收益等回测指标
  references/
    data_guide.md   数据源与字段说明、因子公式、信号规则

alpha-f5-member-position-concentration-production/   生产产物（只读）
  database.parquet  每日收盘后更新的因子结果
  SKILL.md          生产 skill 说明

README.md           本文件
```

## 因子逻辑

- **核心假设**：前5大净多（或净空）席位持仓保证金占全市场超过60%时，主力方向高度一致，价格跟随主力运动。
- **多头集中度**：`bull_ratio = sum(top5 净多 net_margin) / sum(abs(net_margin) 全部席位)`
- **空头集中度**：`bear_ratio = sum(abs(top5 净空 net_margin)) / sum(abs(net_margin) 全部席位)`
- **原始权重**：`raw_weight = bear_ratio - bull_ratio`（正值代表空方更集中，负值代表多方更集中）
- **Level 路径**：`raw_weight` 做 60 日滚动 Z-score → `zscore_level`（`min_periods=20`）
- **Delta 路径**：`raw_weight` 一阶差分后再做 60 日滚动 Z-score → `zscore_delta`（`min_periods=20`）
- **混合**：`factor_value = 0.5 × zscore_level + 0.5 × zscore_delta`（两路同时有值才保留，等权）

## 信号规则

| 信号 | 触发条件 |
|------|----------|
| `buy` | `bull_ratio > 0.6` 且 `factor_value > 0` 且 横截面 `rank ≤ 前90%` |
| `sell` | `bear_ratio > 0.6` 且 `factor_value < 0` 且 横截面 `rank ≥ 后10%` |
| `hold` | 其余情况 |

## 数据源

使用 Panda data SDK，不读取本地表格作为正式输入。

| 环境变量 | 说明 |
|----------|------|
| `PANDA_DATA_USERNAME` | 必填 |
| `PANDA_DATA_PASSWORD` | 必填 |
| `PANDA_DATA_START_DATE` | 可选，格式 `YYYY-MM-DD` |
| `PANDA_DATA_END_DATE` | 可选，格式 `YYYY-MM-DD` |
| `PANDA_DATA_UNDERLYING` | 可选，逗号分隔品种代码，如 `A,AG,AU` |

## 运行方式

```bash
cd alpha-f5-member-position-concentration
python scripts/factor.py
python scripts/validate.py
python scripts/backtest.py
```

## 输出字段

| 字段 | 说明 |
|------|------|
| `trade_date` | 交易日期 `YYYY-MM-DD` |
| `asset_type` | 固定 `future` |
| `symbol` | 期货品种代码 |
| `factor_id` | 固定 `F5` |
| `factor_name` | 固定 `会员持仓集中度` |
| `factor_value` | Level + Delta Z-score 等权混合值 |
| `score` | 横截面百分位 `[0, 100]` |
| `rank` | 横截面整数排名，1 为最优 |
| `signal` | `buy` / `sell` / `hold` |
| `confidence` | `score / 100` |
| `data_version` | 固定 `real-v2` |
| `update_time` | ISO8601 生成时间 |

## 验收标准

- `factor.py` 输出非空，包含 12 个必需字段，`factor_id=F5`，`data_version=real-v2`
- `validate.py` 验证通过（无未来函数、字段完整、取值合法）
- `backtest.py` 输出研究口径和可交易口径两组指标，含多周期 IC 衰减（1D-20D）、Bootstrap IC 95% CI、成本四档分析
- 不通过验证不得进入生产

## 生产查询

交易 agent 使用 `alpha-f5-member-position-concentration-production` skill 读取 `database.parquet`，不在调用时重新计算因子。

## 依赖

- Python 3.10+
- panda-data
- pandas
- scipy
- pyarrow 或 fastparquet
