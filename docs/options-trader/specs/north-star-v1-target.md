# ⭐ 北极星 §1 — 项目第一版目标

> **决策门禁**: 任何 spec / 功能规划 / "是否在 scope" 决策前先读此文档。任何字段、流程、维度变更回到此文档同步。
>
> 时间锚: 2026-05-06 (M3 UAT live 起)
>
> 数据样本: 来自 2026-05-06 真 <span class="term-service">IBKR</span> paper 账户实测 (AAPL, 美股盘中)。

---

## 1. 主用户 / 主入口 / 生产市场

| 项 | 内容 |
|---|---|
| 主用户 | 客户 (中文交互, 美股期权实盘, ~$2M RMB 量级) |
| 主入口 | 客户用自然语言中文提交交易意图 (前端 / API) |
| 生产市场 | 美股期权 (NYSE / NASDAQ / ARCA); ASX 仅开发测试; HK 不用 (跟美股市场差别太大) |

---

## 2. 系统构成

由 <span class="term-agent">Agent 1</span> (LLM 解析) + <span class="term-agent">Agent 2</span> (LLM 决策) + **3 个工人** (盘前 30 min 上线 → 盘后睡眠) 构成。

```
              ┌─────────  IB Gateway  ─────────┐
              │ (账户资金 / 时间信号 / 数据 / 下单) │
              └──┬───┬───┬───┬───┬──────────┘
                 │   │   │   │   │
            Agent 1  时间 条件 信息 Agent 2
           (账户资金) 工人 工人 工人 (下单)
                 │    │    │    │     │
                 └─创建机会单 → 工人协作 → Agent 2 → IBKR 下单
```

**所有外部数据 / 操作都过 <span class="term-service">IB Gateway</span>** — 不只 <span class="term-agent">Agent 2</span> 下单。

| 组件 | 用 <span class="term-service">IB Gateway</span> 做什么 |
|---|---|
| <span class="term-agent">Agent 1</span> | 仓位 ↔ 金额转换需查账户资金 |
| <span class="term-worker">时间工人</span> | 上线/下线时间信号 + 当前市场时间 |
| <span class="term-worker">条件工人</span> | 订阅 (流式) / query (一次性) 市场数据 |
| <span class="term-worker">信息工人</span> | 拉 10 维 bundle 大部分来自 <span class="term-service">IBKR</span> |
| <span class="term-agent">Agent 2</span> | 下单 |

---

## 3. 工人工作时段

3 个工人不是永远在线, 工作时段 = 美股 (ASX 测试时是 ASX) 开盘前 30 min 至收盘, 盘后睡眠。

**每天上线第一件事** (<span class="term-worker">时间工人</span>主导):

1. 重连 <span class="term-service">IB Gateway</span> (前一日所有订阅已失效, <span class="term-service">IBKR</span> 标准行为)
2. 检查所有活跃<span class="term-state">机会单</span>的时间窗口, 已过截止日期的转 <span class="term-state">失败单</span>
3. **重建数据获取** — 分两类:
    - **持续订阅** (流式, 整天活跃): 活跃<span class="term-state">机会单</span>涉及的标的 Level 1 quote / <span class="term-state">持仓单</span>的合约 quote + Greeks / 宏观 VIXY + SPY
    - **一次性 query 但每天重拉** (历史数据不变但需 fresh, 因为前一日 close 进了新一日的窗口): 各<span class="term-state">机会单</span>的条件计算所需的历史 bar (e.g. MA20 需要前 19 天日 bar, 一次拉到再加今天 LAST 算) / 期权链 metadata (新合约上市/老到期会变) / 标的近期 N 根 bar
4. 唤醒 <span class="term-worker">条件工人</span> / <span class="term-worker">信息工人</span>

---

## 4. <span class="term-agent">Agent 1</span> — 解析 + validate

### 4.1 作用一: 解析 3 个关键信息

| 字段 | 必填? | 说明 |
|---|---|---|
| **symbol** (标的) | 必填 | LLM 能从中文公司名映射到代码 (如"特斯拉"→TSLA, "苹果"→AAPL); 真无法解析才阻塞 |
| **触发条件** | 必填 | 触发的就是 <span class="term-agent">Agent 2</span> 下单 |
| **direction** (方向) | 可 null | 看涨 / 看跌 / 看波动; null 时 <span class="term-agent">Agent 2</span> 自由发挥, 非 null 时 <span class="term-agent">Agent 2</span> 必须严格遵守 |

**触发条件 4 种** (本版本):

| 类型 | 说明 | 例子 |
|---|---|---|
| 立即单 | <span class="term-state">机会单</span>立即转下单 | "现在 AAPL 来个看涨价差 10 手" |
| 时间单 | 在某时间窗口内下单 | "明天下午盘 AAPL 来个 call" |
| 条件单 | 满足条件才下单。本版本仅 2 子类: ① **PRICE_BREACH** 价格突破固定值 ② **MA_CROSSOVER** 价格 vs 单条均线交叉 (含上穿/下穿方向) | "AAPL 突破 200 / 向上突破 MA20 / 跌破 MA20" |
| 时间+条件单 | 时间窗口起点后 + 条件满足 | "5 月 10 号后, AAPL 突破 200 下单" |

> 本版本不支持 (留 v2): 隐含波动率条件 / 成交量条件 / 双均线交叉 / 多条件组合 (and-or)
>
> 客户原话不止这 3 个信息 (可能含手数 / 止盈 / 止损暗示 / 策略偏好), **其他 <span class="term-agent">Agent 1</span> 不解析**。客户**完整原话**整段保存, <span class="term-agent">Agent 2</span> 决策时自己看原话。

### 4.2 作用二: validate 输入 + 阻塞

| 情况 | 行为 |
|---|---|
| symbol 无法解析 | 阻塞 |
| 触发条件缺失 | 阻塞 |
| **触发条件不在本版本范围** (如客户说"隐含波动率超 50%") | 阻塞 (本版本仅支持 PRICE_BREACH 常数 / MA_CROSSOVER 单 MA, 其他类型留 v2) |
| 仓位 (%) 和金额 ($) 同时指定 | 阻塞 (互链冲突, 一个推算另一个) |
| 仓位/金额都没指定 + 手数有指定 | 接受 |
| 仓位/金额/手数三者都没指定 | 默认 1 手, 通过 |

阻塞 → <span class="term-state">机会单</span>只能保存为 <span class="term-state">草稿</span>; **阻塞原因要展示给客户** (<span class="term-agent">Agent 1</span> 给出中文阻塞原因)。

---

## 5. <span class="term-agent">Agent 2</span> — 入场决策 + 仓位管理

<span class="term-agent">Agent 2</span> 是**无状态 LLM 调用**, 由 <span class="term-worker">信息工人</span>召唤。

### 5.1 入场决策 (<span class="term-worker">信息工人</span>第一次发 bundle)

LLM 输入 = **10 维 Bundle** (含客户原话 + 触发那一刻的指标值)

输出二选一:
- **不能下单** → <span class="term-state">机会单</span>转 <span class="term-state">失败单</span>, 失败原因 = "Agent-2 拒绝"
- **可以下单** → 给出策略 (含策略类型 / 腿 / 数量) → <span class="term-service">风险门</span> (risk_gate) 三验证 (symbol / direction / 仓位)
    - **三验证全过** → 下单 → 成功后转 <span class="term-state">持仓单</span>
    - **任一验证失败** (如 LLM 给的策略方向不符 opp.direction) → **要求 LLM 重跑一次** (给一次机会, prompt 提醒 LLM 上次违反了什么)
        - 重跑通过 → 下单 → <span class="term-state">持仓单</span>
        - **重跑仍违 → 转 <span class="term-state">失败单</span>**, 失败原因 = "Agent-2 风控不通过"

### 5.2 仓位管理 (<span class="term-state">持仓单</span>, 每 5 min 触发)

LLM 输入 = **10 维 Bundle + 上次决策的 summary** (rolling summary 模式)

输出 = **本次 summary** (≤600 字, 含历史决策关键点 + 当前决策原因 + 用数据说话) + 三选一:
- **仓位不变** (不做任何操作)
- **加仓** (具体多少手 + 用什么策略买)
- **平仓** (平部分多少手 / 一次全平)

全部平仓完成 → <span class="term-state">持仓单</span>转 <span class="term-state">已完成单</span>。

### 5.3 客户期权不用机械止损

平仓决策 = LLM 综合判断 (赚到目标 / 市场反转 / 接近到期), 不是触及某个固定价位。期权买方天然亏损上限 = 权利金, 客户接受亏光。

---

## 6. 4 种触发流程串联

```
1. 立即单:    Agent 1 → 时间工人 (now 立即) → 信息工人 → Agent 2 → IBKR 下单
2. 时间单:    Agent 1 → 时间工人 (到起点) → 信息工人 → Agent 2 → IBKR 下单
3. 条件单:    Agent 1 → 条件工人 → (满足) → 信息工人 → Agent 2 → IBKR 下单
4. 时间+条件: Agent 1 → 时间工人 (到起点) → 条件工人 → (满足) → 信息工人 → Agent 2 → IBKR 下单
```

---

## 7. <span class="term-state">机会单</span>状态机

```
[草稿]  ──客户确认──→  [机会单]  ──注册工人──┐
                                                  ↓
                                ┌─Agent 2 拒绝──→ [失败单]
                                ├─风控重试仍违──→ [失败单]
                                ├─时间窗口过期──→ [失败单]
                                └─下单成功─────→ [持仓单]
                                                  ↓
                                          信息工人每 5 min
                                       Agent 2 平仓/加仓决策
                                                  ↓
                                              全部平仓
                                                  ↓
                                              [已完成单]
```

---

## 8. 10 维 Bundle 详表

入场用 **9 维** (1-4, 6-9 + 10 mock); <span class="term-state">持仓单</span> review 用 **10 维** (含维度 5 持仓盈亏 + 维度 10 上次 summary)。

数据样本来自 2026-05-06 真 <span class="term-service">IBKR</span> paper 实测 (AAPL, 美股盘中, 真下 100 手 ATM LONG_CALL + 自动平仓)。

| # | 维度 | 入场拉法 | 持仓 review 拉法 | JSON 格式和解释 | 实际样本和解释 | 业务解释 + 为什么对仓位管理重要 |
|---|---|---|---|---|---|---|
| 1 | **客户原话** | DB 静态读 | DB 静态读 | `string` (一段中文文本, 客户原文) | `"AAPL 突破 285 就买 call, 100 手, 赚 60% 出"` — 客户的真实意图原话 | 客户的真实意图 + <span class="term-agent">Agent 1</span> 没解析的偏好 (如止盈暗示 "赚 60% 出" / 策略偏好); LLM 仓位管理时也看, 决定按客户原意是否平仓 |
| 2 | **标的 Level 1 quote** | `reqMktData(snapshot=True)` 一次 | 持续订阅 streaming, 取 latest | `{bid, ask, last, bid_size, ask_size, last_size, volume}` 全 float/int<br>**mid = (bid+ask)/2** 是常用估值 | `bid=283.94 ask=283.96 last=283.95 volume=691373` — 价差 2 分流动性好, mid≈$283.95 | 实时市场报价; 算未实现盈亏 / 距 TP 多远 / 当前流动性是否健康 |
| 3 | **标的近期 bar** | `reqHistoricalData` 一次 (1 min × 1 D + 5 min × 5 D + 1 day × 20 D) | 增量拉新 bar | `{1m_1D: [bar...], 5m_5D: [bar...], 1d_20D: [bar...]}`<br>每根 bar = `{date, open, high, low, close, volume}`<br>**OHLCV** = 开/高/低/收/成交量 | `1 min × 1 D = 307 根, 1 day × 20 D = 20 根; 第 1 根: {open:276.92, high:277.87, low:276.5, close:277.13, volume:19227}` | 各时间尺度的价格趋势; 趋势是否符合入场预期, 决定持仓 / 加仓 / 平仓 |
| 4 | **期权链 metadata** | `reqSecDefOptParams` 一次 | 跨日重拉 (新合约上市/老到期) | `{expiries_count, strikes_count, expiries: [YYYYMMDD,...], strikes: [float,...], target_in_chain: bool}`<br>**expiries** = 到期日列表; **strikes** = 行权价列表 | `25 expiries (20260506→20260918) × 116 strikes; AAPL 285 strike 在链中 ✓` | 该 symbol 当前所有可交易合约清单; review 时调仓 (加仓不同 strike/expiry) 从此清单选 |
| 5 | **持仓盈亏快照** (派生) | entry 没持仓 → null | `reqPositions` + `reqMktData` 算 mid → 派生 | `{raw_avg_cost_per_contract, quantity_contracts, total_cost_basis_usd, current_mid_per_share, total_current_value_usd, unrealized_pnl_usd, unrealized_pnl_pct}`<br>**avg_cost** = <span class="term-service">IBKR</span> 每张合约平均成本 (含 multiplier 100); **cost_basis** = 入场总成本; **value** = 当前总市值; **pnl** = 差值 | `cost=$95,544.90 / value=$94,750 / pnl=-$794.90 (-0.83%)` — 入场即 -0.83% 是 bid-ask spread 必然 | 持仓真实财务状态; LLM 直接看到亏损/盈利%, 决定该不该止盈 / 调仓 |
| 6 | **持仓 Greeks** | 候选 ATM ± k strike snapshot | 持续订阅 (跟 quote 一起推) | `{iv, delta, gamma, theta, vega, opt_price}` 全 float<br>**IV** = 隐含波动率 (0.243=24.3%); **Δ** = 标的涨1美元期权变多少; **Γ** = Δ自身变化速度; **Θ** = 时间价值/天 流失 (负数); **V** = IV 涨1% 期权变多少 | `IV=24.3% Δ=0.52 Γ=0.017 Θ=-$0.125 V=0.393 opt_price=$9.55` — 合理 ATM 水平 | 期权对标的价/时间/IV 敏感度; Θ 大 → 临到期前考虑平; V 大 → IV 涨跌敏感; Δ 反映方向暴露 |
| 7 | **IV 曲面** | 派生自 4+6 (一次性) | 几分钟更新 | `{atm_strike, iv_by_strike_call: {strike: iv}, iv_by_strike_put: {strike: iv}, skew_summary, term_structure_summary}`<br>**skew** = 横向 (不同 strike 的 IV 形状); **term** = 纵向 (不同 expiry 的 IV 形状) | `ATM=$285, ±10 档 5 strikes 各 C/P = 10 合约; OTM put 略高 ATM 1.5 vol pts (常态)` | 当前 IV 高估/低估; skew 大 = 恐慌可能反转 → 决定是否离场 |
| 8 | **Order flow** | 派生自合约链 (一次性) | 几分钟更新 | `{call_volume_total, put_volume_total, call_put_volume_ratio, call_oi_total, put_oi_total, call_put_oi_ratio, interpretation}`<br>**OI** = 持仓量 open interest; **ratio** = call/put | `call_vol=14761 / put_vol=1768 / ratio=8.35` — 强看涨情绪 (跟当日涨势一致) | 期权市场情绪指标; ratio 极端 (>5 或 <0.2) 常预示反转 → 决定是否减仓避险 |
| 9 | **市场宏观** | snapshot 一次 | 持续订阅 | `{VIXY: {bid, ask, last, volume, interpretation}, SPY: {...}}`<br>**VIXY** = ProShares VIX Short-Term Futures ETF (paper 没 CBOE Index 用 ETF 替代); **SPY** = S&P 500 ETF (大盘) | `VIXY $27.29 (低位 = 市场情绪平稳)` + `SPY $724.39 (大盘强势)` | 整体市场恐慌 + 大盘走势; 单标的决策应否被宏观风险压倒 (VIX 飙升时主动减仓) |
| 10 | **上次 summary** (review only) | entry 没历史 → null | DB 取上次 LLM 输出 | `string ≤600 字` (一段文本) | 文本含上次决策关键点 + 当前决策原因 + 用数据说话 | 历史决策上下文; 防 review 之间剧烈摇摆 (上次 HOLD 这次突然 FULL_CLOSE 没理由), 决策连贯 |

### 完整 Bundle JSON 样本 (<span class="term-state">持仓单</span> review phase, 10 维齐全)

```json
{
  "dim_1_customer_raw_text": "AAPL 突破 285 就买 call, 100 手, 赚 60% 出",

  "dim_2_underlying_quote": {
    "symbol": "AAPL",
    "bid": 283.94, "bid_size": 1,
    "ask": 283.96, "ask_size": 4,
    "last": 283.95, "last_size": 13,
    "volume": 691373,
    "as_of": "2026-05-06T18:52:00Z"
  },

  "dim_3_underlying_bars": {
    "1m_1D": [
      {"date": "20260506 01:30:00", "open": 276.92, "high": 277.87, "low": 276.50, "close": 277.13, "volume": 19227},
      "... 305 more bars ..."
    ],
    "5m_5D": "... 369 bars ...",
    "1d_20D": [
      {"date": "20260408", "open": 258.40, "high": 259.75, "low": 256.53, "close": 258.90, "volume": 502278},
      "... 19 more bars ..."
    ]
  },

  "dim_4_option_chain_metadata": {
    "expiries_count": 25,
    "strikes_count": 116,
    "expiries_sample": ["20260506", "20260508", "20260511", "20260513", "20260515", "20260618"],
    "target_expiry_in_chain": true
  },

  "dim_5_position_pnl_snapshot": {
    "raw_avg_cost_per_contract": 955.45,
    "quantity_contracts": 100,
    "total_cost_basis_usd": 95544.90,
    "current_mid_per_share": 9.475,
    "current_value_per_contract_usd": 947.50,
    "total_current_value_usd": 94750.00,
    "unrealized_pnl_usd": -794.90,
    "unrealized_pnl_pct": -0.83
  },

  "dim_6_position_greeks": {
    "atm_strike": 285,
    "expiry": "20260618",
    "iv": 0.243,
    "delta": 0.517,
    "gamma": 0.0167,
    "theta": -0.125,
    "vega": 0.393,
    "opt_price": 9.55
  },

  "dim_7_iv_surface": {
    "atm_strike": 285,
    "strikes_tested": [275, 280, 285, 290, 295],
    "iv_by_strike_call": {"275": 0.215, "280": 0.227, "285": 0.243, "290": 0.250, "295": 0.260},
    "iv_by_strike_put": {"275": 0.231, "280": 0.234, "285": 0.241, "290": 0.244, "295": 0.237},
    "skew_summary": "OTM put 略高于 ATM 约 1.5 vol pts (常态)",
    "term_structure_summary": "前端 IV 略高于后端 (无明显 contango)"
  },

  "dim_8_order_flow": {
    "call_volume_total": 14761,
    "put_volume_total": 1768,
    "call_put_volume_ratio": 8.35,
    "call_oi_total": 234567,
    "put_oi_total": 89012,
    "call_put_oi_ratio": 2.63,
    "interpretation": "强看涨情绪 (CV/PV 8.35), 跟当日涨势一致"
  },

  "dim_9_macro": {
    "VIXY": {"bid": 27.28, "ask": 27.29, "last": 27.29, "volume": 9793, "interpretation": "低位, 市场情绪平稳"},
    "SPY": {"bid": 724.38, "ask": 724.40, "last": 724.39, "volume": 518886, "interpretation": "大盘强势"}
  },

  "dim_10_last_summary": "上次 review (5 min 前) 决策 HOLD。理由: 持仓 AAPL 285C × 100 手, mid 跌 0.83% 主因开仓 spread 已被消化。当前 underlying $283.95 距 ATM $1 内, Δ 0.52 暴露看涨意图; Θ -$0.125/天 时间价值流失温和。客户原话明确'赚 60% 出', 当前 -0.83% 远未到目标。VIXY 低位 + CP ratio 8.35 强看涨情绪, 维持原计划。"
}
```

### 一个完整端到端示例

**客户输入** (前端): `"AAPL 突破 285 就买 call, 100 手, 赚 60% 出"`

**1. <span class="term-agent">Agent 1</span> 派活**:

- symbol: AAPL ✓
- 触发条件: PRICE_BREACH (level=285, direction=ABOVE) ✓ (在本版本支持范围内)
- direction: BULLISH ✓
- 仓位/金额/手数: 客户指定 100 手, 通过
- validate: 通过, 进 <span class="term-state">机会单</span>

**2. server compile** 入库, 注册 <span class="term-worker">时间工人</span> 立即激活

**3. <span class="term-worker">条件工人</span>**:

- <span class="term-service">MarketDataBus</span> 订阅 AAPL Level 1
- 每 tick 检查 `latest_price > 285`
- 假设 09:42 (ET) AAPL 涨到 $285.15 → 触发
- 唤醒 <span class="term-worker">信息工人</span>, 传 metadata: `{event:"PRICE_BREACH", price:285.15, at:"...09:42:13Z"}`
- 自己结束 (一次性触发)

**4. <span class="term-worker">信息工人</span> (entry phase)**:

- 拉 9 维 bundle
- 调 <span class="term-agent">Agent 2</span> LLM

**5. <span class="term-agent">Agent 2</span> entry 决策**:

- 看 9 维信息, 遵守标的、仓位和方向门禁下单
- LLM 输出: `{strategy: "LONG_CALL", legs: [{symbol:"AAPL", expiry:"20260618", strike:285, right:"C", quantity:100, action:"BUY"}], thesis_summary: "客户突破 285 入场, IV 24% 合理, CP ratio 8.35 强看涨, 选 ATM call 一致看涨意图"}`
- <span class="term-service">风险门</span>三验证: symbol=AAPL ✓ / direction LONG_CALL→BULLISH ✓ / 仓位 100 手 ≤ 客户指定 ✓ → 通过
- 下单 LMT @ ask = $9.55

**6. 下单成功 → 转 <span class="term-state">持仓单</span>**, 注册 5 min review self-loop

**7. 5 min 后, <span class="term-worker">信息工人</span> review phase**:

- 拉完整 10 维 bundle (含维度 5 持仓盈亏 + 维度 10 上次 summary)
- 调 <span class="term-agent">Agent 2</span> LLM
- LLM 输出: `{decision: "HOLD", new_summary: "...", reason: "未实现 -0.83% 远未到 60% 目标, 趋势没反转, 维持持仓"}`
- 持仓不动, 下次 review 用本次 summary

**8. 反复 review**, 直到 LLM 决策"平仓" → SELL 100 LMT → <span class="term-state">持仓单</span>转 <span class="term-state">已完成单</span>

---

## 9. 关键约束 (本版本必守)

| 类别 | 约束 |
|---|---|
| <span class="term-agent">Agent 1</span> 角色 | 给工人派活, 不防 LLM 改原话, 不替 <span class="term-agent">Agent 2</span> 决策具体参数 |
| 必填字段 | symbol / 触发条件 (必填); direction 非必填但非 null 时强约束 |
| direction 4 值 | BULLISH (看涨) → 必须做看涨策略; BEARISH (看跌) → 看跌; VOLATILITY (看波动) → 中性/波动率; null → <span class="term-agent">Agent 2</span> 自由 |
| 触发条件限定 | 4 类: 立即 / 时间 / 条件 / 时间+条件; 条件类**只**支持 PRICE_BREACH 常数 + MA_CROSSOVER 单 MA, 不在范围 → 阻塞 |
| 3 工人角色 | <span class="term-worker">时间工人</span> (scheduler 计时) / <span class="term-worker">条件工人</span> (算条件) / <span class="term-worker">信息工人</span> (打 bundle 召唤 <span class="term-agent">Agent 2</span>) — 盘前 30 min 上线, 盘后睡眠 |
| 数据基础设施 | <span class="term-service">MarketDataBus</span> 全局单例; 入场用 snapshot 不订阅; 持仓后才建持续订阅 (LAST + 持仓合约 + 宏观 VIXY/SPY) |
| <span class="term-service">风险门</span> (risk_gate) 二层防护 | Layer 1: LLM 输出 schema enum 限制策略只能在白名单 9 种内; Layer 2: "策略 → 方向"映射表对照 opp.direction, 不匹配**先要求 LLM 重跑一次**, 重跑仍违 → <span class="term-state">失败单</span> |
| 策略白名单 (无裸卖空) | 见下表 |
| 客户体验 | 前端中文; 客户改解析回输入框改原话重新解析, 不点字段直接改; 阻塞原因要展示给客户 |
| 不止损 | 期权要么亏光要么赚钱离场, 没有机械止损线 |

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

10 维数据可获得性已用 <span class="term-service">IBKR</span> paper 账户实测 (AAPL ATM 100 手 LONG_CALL 真下单 + 平仓):

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

**Level 2 / NASDAQ TotalView**: 当前 <span class="term-service">OPRA</span> $32.75/月 不含 depth, 经实测确认 (error 10089)。第一版不订, depth 边际价值不及成本。

---

## 11. 维护规则

- 任何 invariant / 字段 / 流程 / 维度变更 → 同步本文档
- 数据样本季度 refresh 一次 (跑 phase1 probe 脚本即可)
- 关联文档:
    - 项目内 brainstorming 归档: `docs/superpowers/discussions/2026-05-05-north-star-section1-brainstorming.md`
    - 北极星 §2-§7 (远景 / 5 问 / 中间路径 / forward compat / 历史决策 / 反偏离): 待 §1 锁定后逐节 review

---

**END OF NORTH STAR §1**
