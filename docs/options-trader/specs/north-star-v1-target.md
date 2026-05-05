# ⭐ 北极星 §1 — 本版目标

> **决策门禁**: 任何 spec / 功能规划 / "是否在 scope" 决策前先读此文档。任何字段、流程、维度变更回到此文档同步。
>
> 时间锚: 2026-05-06 (M3 UAT live 起)
>
> 数据样本: 来自 2026-05-06 真 IBKR paper 账户实测 (AAPL, 美股盘中)。

---

## 1. 主用户 / 主入口 / 生产市场

| 项 | 内容 |
|---|---|
| 主用户 | 客户 (中文交互, 美股期权实盘, ~$2M RMB 量级) |
| 主入口 | 客户用自然语言中文提交交易意图 (前端 / API) |
| 生产市场 | 美股期权 (NYSE / NASDAQ / ARCA); ASX 仅开发测试; HK 不用 (跟美股市场差别太大) |

---

## 2. 系统构成

由 **Agent 1** (LLM 解析) + **Agent 2** (LLM 决策) + **3 个 Worker** (盘前 30 min 上线 → 盘后睡眠) 构成。

```
              ┌─────────  IB Gateway  ─────────┐
              │ (账户资金 / 时间信号 / 数据 / 下单) │
              └──┬───┬───┬───┬───┬──────────┘
                 │   │   │   │   │
            Agent 1  时间 条件 信息 Agent 2
           (账户资金) wkr  wkr  wkr  (下单)
                 │    │    │    │     │
                 └─创建机会单 → workers 协作 → Agent 2 → IBKR 下单
```

**所有外部数据 / 操作都过 IB Gateway** — 不只 Agent 2 下单。

| 组件 | 用 IB Gateway 做什么 |
|---|---|
| Agent 1 | 仓位 ↔ 金额转换需查账户资金 |
| 时间 worker | 上线/下线时间信号 + 当前市场时间 |
| 条件 worker | 订阅 (流式) / query (一次性) 市场数据 |
| 信息 worker | 拉 10 维 bundle 大部分来自 IBKR |
| Agent 2 | 下单 |

---

## 3. Worker 工作时段

3 个 Worker 不是永远在线, 工作时段 = 美股 (ASX 测试时是 ASX) 开盘前 30 min 至收盘, 盘后睡眠。

**每天上线第一件事** (时间 worker 主导):
1. 重连 IB Gateway (前一日所有订阅已失效, IBKR 标准行为)
2. 检查所有活跃机会单的时间窗口, 已过截止日期的转"失败单"
3. 重新订阅活跃机会单涉及的标的数据
4. 唤醒条件 worker / 信息 worker

---

## 4. Agent 1 — 解析 + validate

### 4.1 作用一: 解析 3 个关键信息

| 字段 | 必填? | 说明 |
|---|---|---|
| **symbol** (标的) | 必填 | LLM 能从中文公司名映射到代码 (如"特斯拉"→TSLA, "苹果"→AAPL); 真无法解析才阻塞 |
| **触发条件** | 必填 | 触发的就是 Agent 2 下单 |
| **direction** (方向) | 可 null | 看涨 / 看跌 / 看波动; null 时 Agent 2 自由发挥, 非 null 时 Agent 2 必须严格遵守 |

**触发条件 4 种** (本版本):

| 类型 | 说明 | 例子 |
|---|---|---|
| 立即单 | 机会单立即转下单 | "现在 AAPL 来个看涨价差 10 手" |
| 时间单 | 在某时间窗口内下单 | "明天下午盘 AAPL 来个 call" |
| 条件单 | 满足条件才下单。本版本仅 2 子类: ① **PRICE_BREACH** 价格突破固定值 ② **MA_CROSSOVER** 价格 vs 单条均线交叉(含上穿/下穿方向) | "AAPL 突破 200 / 向上突破 MA20 / 跌破 MA20" |
| 时间+条件单 | 时间窗口起点后 + 条件满足 | "5 月 10 号后, AAPL 突破 200 下单" |

> 本版本不支持 (留 v2): 隐含波动率 / 成交量条件 / 双均线交叉 / 多条件组合 (and-or)
>
> 客户原话不止这 3 个信息 (可能含手数 / 止盈 / 止损暗示 / 策略偏好), **其他 Agent 1 不解析**。客户**完整原话**整段保存, Agent 2 决策时自己看原话。

### 4.2 作用二: validate 输入 + 阻塞

| 情况 | 行为 |
|---|---|
| symbol 无法解析 / 触发条件缺失 | 阻塞 |
| 仓位 (%) 和金额 ($) 同时指定 | 阻塞 (互链冲突, 一个推算另一个) |
| 仓位/金额都没指定 + 手数有指定 | 接受 |
| 仓位/金额/手数三者都没指定 | 默认 1 手, 通过 |

阻塞 → 机会单只能保存为**草稿**; 阻塞项原文显示给客户。

---

## 5. Agent 2 — 入场决策 + 仓位管理

Agent 2 是**无状态 LLM 调用**, 由信息 worker 召唤。

### 5.1 入场决策 (信息 worker 第一次发 bundle)

LLM 输入 = **10 维 Bundle** (含客户原话 + 触发那一刻的指标值)

输出二选一:
- **不能下单** → 机会单转"失败单", 失败原因 = "Agent-2 拒绝"
- **可以下单** → 给出策略 (含策略类型 / 腿 / 数量) → **风控门 (risk_gate) 三验证** (symbol / direction / 仓位) → 通过则下单 → 成功后转"持仓单"

### 5.2 仓位管理 (持仓单, 每 5 min 触发)

LLM 输入 = **10 维 Bundle + 上次决策的 summary** (rolling summary 模式)

输出 = **本次 summary** (≤600 字, 含历史决策关键点 + 当前决策原因 + 用数据说话) + 三选一:
- **仓位不变** (不做任何操作)
- **加仓** (具体多少手 + 用什么策略买)
- **平仓** (平部分多少手 / 一次全平)

全部平仓完成 → 持仓单转"已完成单"。

> 客户**期权不用机械止损** — 平仓决策 = LLM 综合判断 (赚到目标 / 市场反转 / 接近到期), 不是触及某个固定价位。期权买方天然亏损上限 = 权利金, 客户接受亏光。

---

## 6. 4 种触发流程串联

```
1. 立即单:    Agent 1 → 时间 worker (now 立即) → 信息 worker → Agent 2 → IBKR 下单
2. 时间单:    Agent 1 → 时间 worker (到起点) → 信息 worker → Agent 2 → IBKR 下单
3. 条件单:    Agent 1 → 条件 worker → (满足) → 信息 worker → Agent 2 → IBKR 下单
4. 时间+条件: Agent 1 → 时间 worker (到起点) → 条件 worker → (满足) → 信息 worker → Agent 2 → IBKR 下单
```

---

## 7. 机会单状态机

```
[草稿]  ──客户确认──→  [机会单]  ──注册 worker──┐
                                                  ↓
                                ┌─Agent 2 拒绝──→ [失败单]
                                ├─时间窗口过期──→ [失败单]
                                └─下单成功─────→ [持仓单]
                                                  ↓
                                         信息 worker 每 5 min
                                       Agent 2 平仓/加仓决策
                                                  ↓
                                              全部平仓
                                                  ↓
                                              [已完成单]
```

---

## 8. 10 维 Bundle 详表 (含真实数据样本)

每个维度: **怎么拉 + 业务解释 + 真实数据样本** (来自 2026-05-06 paper 实测)。

入场用 **9 维** (1-4, 6-9 + 10 mock); 持仓 review 用 **10 维** (含维度 5 持仓盈亏 + 维度 10 上次 summary)。

---

### 维度 1 — 客户原话 (raw_input_text)

**怎么拉**: 数据库静态读取 (机会单创建时已存)。
**业务解释**: 让 LLM 决策时知道客户初衷, 包括客户提到但 Agent 1 没解析的偏好 (止盈意图 / 策略偏好 / 时间偏好等)。

**数据样本**:
```
"AAPL 突破 285 就买 call, 100 手, 赚 60% 出"
```

---

### 维度 2 — 标的 Level 1 quote (bid/ask/last + sizes + volume)

**怎么拉**:
- 入场: `reqMktData(snapshot=True)` 一次性
- 持仓: 持续订阅 streaming, review 时取 latest

**业务解释**: 当前买卖价 + 最新成交价 + 各档位量。LLM 据此判断当前价 / 距 TP 多远 / 流动性是否健康。

**数据样本** (AAPL, 2026-05-06 美股盘中):
```json
{
  "bid": 283.94,
  "bid_size": 1,
  "ask": 283.96,
  "ask_size": 4,
  "last": 283.95,
  "last_size": 13,
  "volume": 691373
}
```

---

### 维度 3 — 标的近期 bar (1 min / 5 min / daily 多周期)

**怎么拉**: `reqHistoricalData` 一次性 query, 各周期分别拉:
- 1 min × 1 D (约 390 根 bar, 美股交易日 6.5h)
- 5 min × 5 D (约 369 根)
- 1 day × 20 D (20 根, 用于计算 MA20)

**业务解释**: 显示最近价格趋势 / 关键支撑阻力 / 当前波动率。LLM 据此判断走势是否符合预期, 决定是否平仓或加仓。

**数据样本** (AAPL 1 min × 1 D, 共 307 根):
```json
[
  {
    "date": "20260506  01:30:00",
    "open": 276.92, "high": 277.87, "low": 276.5, "close": 277.13,
    "volume": 19227
  },
  {
    "date": "20260506  01:31:00",
    "open": 277.08, "high": 277.30, "low": 276.53, "close": 276.66,
    "volume": 3612
  }
]
```

---

### 维度 4 — 期权链 metadata (所有 expiry / strike)

**怎么拉**: `reqSecDefOptParams` 一次性, 跨日重拉一次 (新合约上市 / 老合约到期会变化)。

**业务解释**: 列出该 symbol 当前所有可交易的到期日 + 行权价 + 交易所 + multiplier。LLM 选具体策略腿时用 (e.g. 选 ATM ± 几档 strike, 选周期合适的 expiry)。

**数据样本** (AAPL, 2026-05-06):
```
expiries 数: 25
strikes 数: 116
expiries 头 5 个: [20260506, 20260508, 20260511, 20260513, 20260515]
target 20260618 在链中: ✓
```

---

### 维度 5 — 持仓盈亏快照 (派生值)

> ⚠️ 仅持仓 review 阶段使用; 入场 phase 还没持仓为 null。

**怎么拉**: 信息 worker 数据加工:
1. `reqPositions` 拉持仓列表 → 取该 conid 的 avg_cost + pos
2. `reqMktData` 拉持仓合约当前 mid (= (bid + ask) / 2)
3. 派生计算: 总成本 / 总市值 / 未实现盈亏 / 已实现盈亏

**业务解释**: LLM 直接看到"未实现亏损 -$794, 占成本 -0.83%"比看到 4 个合约的 bid/ask 更好判断。是否赚到目标 / 是否需要止损。

**数据样本** (AAPL 285 Call × 100 手, 入场即测):
```json
{
  "raw_avg_cost_per_contract": 955.45,
  "quantity_contracts": 100,
  "total_cost_basis_usd": 95544.90,
  "current_mid_per_share": 9.475,
  "current_value_per_contract_usd": 947.50,
  "total_current_value_usd": 94750.00,
  "unrealized_pnl_usd": -794.90,
  "unrealized_pnl_pct": -0.83
}
```

> 入场后立即出现的 -0.83% 是合理的 — 用 ask 价开仓, 现在用 mid 估值, 自然差一截 bid-ask spread。

---

### 维度 6 — 持仓合约 Greeks (Δ / Γ / Θ / V / IV)

**怎么拉**: `reqMktData` 持续订阅, IBKR 通过 `tickOptionComputation` 推送 Greeks。

**业务解释**:
- **Δ (Delta)** = 标的涨 $1 时, 期权价格变多少 (e.g. 0.5 = 涨 $0.5)
- **Γ (Gamma)** = Delta 自己变化的速度 (越接近 ATM 越大)
- **Θ (Theta)** = 时间价值流失速度 (每日权利金缩水多少, 卖方赚 / 买方亏)
- **V (Vega)** = 隐含波动率每涨 1% 时, 期权价变多少
- **IV** = 隐含波动率 (市场对未来波动幅度的预期, 高 = 期权贵)

LLM 据 Theta 决定是否 close-to-expiry 提前平; 据 Vega 决定是否在 IV 高位平 (vol crush 风险)。

**数据样本** (AAPL 285 Call expiry 20260618, ATM):
```json
{
  "iv": 0.243,
  "delta": 0.517,
  "gamma": 0.0167,
  "theta": -0.125,
  "vega": 0.393,
  "opt_price": 9.55
}
```

> IV 24.3% 是合理的 ATM 期权水平; Delta 0.52 表示该 call 接近 ATM (delta 50 的合约权利金对标的最敏感)。

---

### 维度 7 — IV 曲面 (skew + term structure 摘要)

**怎么拉**: 派生自维度 4 (合约链) + 维度 6 (各 strike Greeks)。批量拉 ATM ± N 档 strike 的 IV, 计算 skew (横向偏度) + term (纵向期限结构)。

**业务解释**:
- **Skew**: 通常 OTM put 的 IV 比 ATM 高 (恐慌 hedge 需求), 大幅 skew = 市场恐慌情绪强
- **Term structure**: 短期 IV 高 / 长期低 = 短期事件待消化; 短低长高 = 远期不确定性大

LLM 据此判断当前波动率水平是否高估 / 低估, 决定是否可获利离场。

**数据样本** (AAPL, ATM=$285, ± 10 档 strikes):
```
strikes_tested: [275, 280, 285, 290, 295]
contracts_resolved: 10 (5 strikes × Call/Put)
sample IV by strike (Call): 275=低 / 285=24.3% (ATM) / 295=低
```

---

### 维度 8 — Order flow (call/put volume 比 + OI 比)

**怎么拉**: 派生自维度 4 (合约链) + 维度 6 (各合约 quote 的 volume + OI)。

**业务解释**: 期权市场情绪指标:
- **call/put volume ratio 高** = 买 call 的人多 → 看涨情绪强
- **call/put OI ratio 高** = 持仓多在 call 端 → 看涨倾向
- 极端比值 (e.g. > 5 或 < 0.2) 常预示反转

**数据样本** (AAPL 2026-05-06):
```json
{
  "call_volume_total": 14761,
  "put_volume_total": 1768,
  "call_put_volume_ratio": 8.35
}
```

> CP ratio 8.35 = call 量是 put 量 8 倍多, 强看涨情绪 (跟今日 AAPL 涨势一致)。

---

### 维度 9 — 市场宏观 (VIXY 替代 VIX / SPY / sector ETF)

**怎么拉**: `reqMktData` 持续订阅, VIXY 替代 VIX index (paper 账户通常无 CBOE Index 订阅)。

**业务解释**:
- **VIXY** (ProShares VIX Short-Term Futures ETF) = 市场恐慌指数 ETF; 高 = 市场避险, 低 = 风险偏好
- **SPY** (S&P 500 ETF) = 大盘走势, 大盘崩 → 个股大概率跟着崩
- **Sector ETF** (XLK 科技 / XLE 能源 / XLF 金融等) = 板块情绪

LLM 据此判断单一标的的决策应否被宏观风险压倒 (如 VIX 飙升时主动减仓)。

**数据样本** (2026-05-06 美股盘中):
```json
{
  "VIXY": {
    "bid": 27.28, "ask": 27.29, "last": 27.29, "volume": 9793
  },
  "SPY": {
    "bid": 724.38, "ask": 724.40, "last": 724.39, "volume": 518886
  }
}
```

> VIXY ~$27 是低位 (近期市场情绪平稳), SPY 在 724 附近震荡。

---

### 维度 10 — 上次 summary (review only)

**怎么拉**: 数据库读取上一次 LLM 输出的 summary。

**业务解释**: 让 LLM 决策时有"历史记忆" — 知道上次为什么决定 HOLD / 加仓 / 平仓, 防止 review 之间出现剧烈摇摆。

**数据样本** (mock 例):
```
"持仓 AAPL 285C × 100 手, 入场后 5 min mid 跌 0.83% 主因 bid-ask spread 已被消化。
当前 underlying $283.95 距 ATM 1 美元内, Delta 0.52 暴露看涨意图;
Theta -$0.125/天 时间价值流失温和。
上次决策 HOLD, 理由: 客户原话明确等突破 285 标志, 当前未越过仍处建仓窗内。
本次 review 关键观察: VIX 低位 + CP ratio 8.35 强看涨情绪, 维持原计划等趋势确立。"
```

---

## 9. 关键约束 (本版本必守)

| 类别 | 约束 |
|---|---|
| **Agent 1 角色** | 给 worker 派活, 不防 LLM 改原话, 不替 Agent 2 决策具体参数 |
| **必填字段** | 只有 symbol / 触发条件 (必填); direction 非必填但非 null 时强约束 |
| **direction 4 值** | BULLISH (看涨) → 必须做看涨策略; BEARISH (看跌) → 看跌; VOLATILITY (看波动) → 中性/波动率; null → Agent 2 自由 |
| **触发条件限定** | 4 类: 立即 / 时间 / 条件 / 时间+条件; 条件类只 PRICE_BREACH 常数 + MA_CROSSOVER 单 MA |
| **Worker 三角色** | 时间 worker (scheduler 计时) / 条件 worker (算条件) / 信息 worker (打 bundle 召唤 Agent 2) — 盘前 30 min 上线, 盘后睡眠 |
| **数据基础设施** | MarketDataBus 全局单例; 入场用 snapshot 不订阅; 持仓后才建持续订阅 (LAST + 持仓合约 + 宏观 VIXY/SPY) |
| **风险门 (risk_gate) 二层防护** | Layer 1: LLM 输出 schema enum 限制策略只能在白名单 9 种内; Layer 2: "策略 → 方向"映射表对照 opp.direction, 不匹配重跑一次, 仍违 → 失败单 |
| **策略白名单 (无裸卖空)** | 见下表 |
| **客户体验** | 前端中文; 客户改解析回输入框改原话重新解析, 不点字段直接改 |
| **不止损** | 期权要么亏光要么赚钱离场, 没有机械止损线 |

### 策略白名单 9 种 (无裸卖空)

| 方向 | 允许策略 | 备注 |
|---|---|---|
| BULLISH | LONG_CALL / BULL_CALL_SPREAD / BULL_PUT_SPREAD (有保护) | 看涨 |
| BEARISH | LONG_PUT / BEAR_PUT_SPREAD / BEAR_CALL_SPREAD (有保护) | 看跌 |
| VOLATILITY 买波动 | LONG_STRADDLE / LONG_STRANGLE | 中性, 波动放大才赚 |
| VOLATILITY 卖波动 (有保护) | IRON_CONDOR / IRON_BUTTERFLY | 中性, 不大动才赚 |

> **裸卖空 (SHORT_CALL / SHORT_PUT / SHORT_STRADDLE 等亏损无底线策略) 永久禁止**, LLM 输出 schema 物理上无法生成。

---

## 10. 数据基础设施真实测试结果 (2026-05-06)

10 维数据可获得性已用 IBKR paper 账户实测 (AAPL ATM 100 手 LONG_CALL 真下单 + 平仓):

| 维度 | 状态 | 备注 |
|---|---|---|
| 1 客户原话 | ✓ | DB 静态 |
| 2 标的 quote | ✓ | bid/ask/last/sizes/volume 全到 (delayed 数据) |
| 3 标的 bar | ✓ | 1 min × 1 D (307 根) / 5 min × 5 D (369 根) / 1 day × 20 D (20 根) |
| 4 期权链 | ✓ | 25 expiries / 116 strikes |
| **5 持仓 P&L** | ✓ | **真下 100 手实测**, 派生公式确认: cost=$95,544.90 / value=$94,750 / pnl=-$794.90 (-0.83%) |
| 6 ATM Greeks | ✓ | IV 24.3% / Δ 0.52 / Γ 0.017 / Θ -0.125 / V 0.393 |
| 7 IV 曲面 | ✓ | ATM ± 10 档 5 strikes 全 resolve |
| 8 Order flow | ✓ | call/put volume ratio = 8.35 |
| 9 macro | ✓ | SPY ✓ + VIXY ✓ (替代 VIX) |
| 10 上次 summary | ✓ | DB 静态 |

**Level 2 / NASDAQ TotalView**: 当前 OPRA $32.75/月 不含 depth, 经实测确认 (error 10089). 第一版不订, depth 边际价值不及成本。

---

## 11. 维护规则

- 任何 invariant / 字段 / 流程 / 维度变更 → 同步本文档
- 数据样本季度 refresh 一次 (跑 phase1 probe 脚本即可)
- 关联文档:
  - 项目内 brainstorming 归档: `docs/superpowers/discussions/2026-05-05-north-star-section1-brainstorming.md`
  - 北极星 §2-§7 (远景 / 5 问 / 中间路径 / forward compat / 历史决策 / 反偏离): 待 §1 锁定后逐节 review

---

**END OF NORTH STAR §1**
