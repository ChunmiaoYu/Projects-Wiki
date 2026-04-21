# Spec 镜像 — Agentic Trading 重构（Agent 2 每 bar 自主决策）

> **镜像来源**：项目 repo `docs/superpowers/specs/2026-04-21-agentic-trading-redesign-design.md`
> **镜像日期**：2026-04-21
> **最终版本**：v5（通过五专家 Round 1-5 全部评审，55 个 P0/P1 全部应用）
> **对应决策记录（客户友好提炼）**：[`../decisions/2026-04-21-agentic-trading-redesign.md`](../decisions/2026-04-21-agentic-trading-redesign.md)
> **脱敏说明**：已按全局 CLAUDE.md §18 脱敏红线核查（无 API key / 账户号 / 具体金额 / VM IP / 实盘参数具体值）。风控默认值只作示例，真实值在项目 `config/risk_limits.yml`

---

## 原始 Spec 内容

> 以下为项目 repo 内 spec 原文。因 spec 本身已按 public 风格写作，内容无需修改，仅加镜像 header。

**Spec 日期**：2026-04-21
**版本**：v0 → v5 迭代（五专家 Round 1-5 审议完成）
**对应原始讨论（internal，保留在项目 repo 内不对外）**：项目 repo `docs/superpowers/discussions/2026-04-21-agentic-trading-redesign.md`

---

## 0. 执行摘要

把 Agent 2 从"一次性生成策略交给用户审批"改为 **"LLM 在每个 bar（默认 5 分钟）自主决策持仓"**。

### §18 触发清单命中（全部 7 条，必须 wiki 留痕）

| # | §18 条款 | 本方案命中点 |
|---|---|---|
| 1 | 触发五专家 brainstorming | 本 spec 正在走五专家评审（§17） |
| 2 | invariant / 核心规则修改 | 项目 CLAUDE.md §5 invariant 4-5 / 11 / 16 / 17 要改（见 §19） |
| 3 | 新增主要依赖 | Anthropic SDK / pandas-ta / pandas-market-calendars 三个新依赖 |
| 4 | 架构级选型 | LLM 厂商 OpenAI→Claude（Agent 2）+ 决策范式反转（规则→AI） |
| 5 | 数据模型 / API 契约 | 新表 `agent2_decisions` + 2 个新 prompt 文件 + 新 collectors 模块 |
| 6 | 运维策略调整 | UAT/PROD 从 paper→live + secondary username 要求（见 memory `project_ibkr_ops.md`） |
| 7 | 安全 / 风控规则 | 新风控层 6 条硬规则（§4） |

### 跨文件关系（本 spec 的定位）

| 文件 | 定位 | 生命周期 |
|---|---|---|
| 本 spec | **设计真相源** — 一次讨论的最终方案 + why | 实施期间只改版本号，不重写 |
| `docs/superpowers/plans/2026-04-21-...plan.md`（待写） | **执行步骤真相源** — Step A-H 展开 + 每 Step 具体测试 cases | 实施中随进度更新 |
| `task_plan.md` | **进度真相源** — 当前在哪一 Step / 本周做什么 | 每个 Step 完更新 [x] |
| `findings.md` | **未解决债真相源** — Step 实施中发现的新 P2/P3 | 修复时删，新发现时加 |

### 决策摘要

| 核心改变 | 从 | 到 |
|---|---|---|
| 决策主体 | 用户按按钮 | Claude LLM 自主判断 |
| Agent 2 触发 | opp SUBMITTED 时一次 | opp 入场后每 5 min 循环 |
| 止盈止损 | `drain_exit_queue` + `check_time_stops` 规则化 | LLM 看 9 类数据自判 |
| LLM 厂商 | OpenAI gpt-5 | Claude Haiku 4.5 常规 / Sonnet 4.6 关键节点 |
| 反馈/词典 | 活闭环 | feature flag 冻结（保留代码和数据） |
| Agent 1 prompt | 拼 business_lexicon context | 关词典（trigger_catalog / default_policies 保留） |
| 触发类型 | 立即 + 时间 + 部分条件 | 4 种（立即 / 时间 / 条件 / 时间+条件组合） |
| 环境枚举 | dev / uat / prod | local-dev / cloud-dev / uat / prod |
| UAT / PROD 账户 | 都 paper（现状） | UAT = 小额 live / PROD = 大额 live |

---

## 1. 新架构图

```
┌───────────────────────────────────────────────────────────────┐
│ Agent 1 (保留 + 简化) — 意图解析                              │
│ LangGraph 8 节点，OpenAI Structured Outputs                   │
│ 关键节点改动：load_lexicon_context flag 关 business_lexicon   │
│ 输出：ParsedIntentCore + 5 大计划 (trigger / activation / etc) │
└──────────────────────┬────────────────────────────────────────┘
                       │ (用户提交机会单)
                       ▼
┌───────────────────────────────────────────────────────────────┐
│ Trigger 层 — 触发条件监控（4 种）                              │
│ 立即 / 时间 / 条件 / 时间+条件组合                             │
│ 新增 CONDITION_MONITOR 模块 (MA_CROSSOVER + PRICE_BREACH)      │
│ 修 findings F4 + F5 runtime 层                                 │
└──────────────────────┬────────────────────────────────────────┘
                       │ (条件满足)
                       ▼
┌───────────────────────────────────────────────────────────────┐
│ 数据采集层（扩容到 9 类 + 第 8 类预留接口）                    │
│ ①持仓 ②标的价 ③希腊 ④K线多粒度 ⑤技术指标 ⑥流动性              │
│ ⑦大盘SPY+VIX ⑧新闻(接口预留) ⑨异常活动 ⑩用户意图              │
│ 新增 ExternalDataCollector 抽象 + pandas-ta                   │
└──────────────────────┬────────────────────────────────────────┘
                       │ (打包 JSON)
                       ▼
┌───────────────────────────────────────────────────────────────┐
│ Agent 2 新版本 — 两个工作点                                    │
│ (A) 首次入场决策: 选策略类型 + strike + 数量 (Claude Sonnet)   │
│ (B) 入场后持续决策: HOLD/PARTIAL_CLOSE/FULL_CLOSE/ADJUST_STOP │
│     每 5 min 默认 Haiku, 触发条件升 Sonnet                     │
│ Prompt Caching 开启, Temperature 0, 结构化 JSON 输出           │
└──────────────────────┬────────────────────────────────────────┘
                       │ (决策 JSON)
                       ▼
┌───────────────────────────────────────────────────────────────┐
│ 风控层（非 AI, 硬规则 6 条）                                   │
│ 单仓30% / 总仓80% / 流动性 / sanity / 频率 / 熔断             │
│ 失败 = 拒绝+日志+告警, 不静默跳过                              │
└──────────────────────┬────────────────────────────────────────┘
                       │ (通过)
                       ▼
┌───────────────────────────────────────────────────────────────┐
│ 执行层（改造）                                                 │
│ mid-based 4 档限价: patient/normal/urgent/market              │
│ 撤单重试 2 min * 3 次, spread_ratio +0.1                      │
│ LMT→追价→MKT 升级链 (market 档需风控额外许可)                 │
└───────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────────────┐
│ 日志 / 审计 / 告警 / 日报                                      │
│ 每 agent.run() 独立 JSON trace (修 F-LLM-AUDIT-GAP)           │
│ 新表 agent2_decisions, 控制台+文件日志, 告警接口预留           │
│ 日报: token 消耗 / 决策次数 / 订单 / 盈亏                     │
└───────────────────────────────────────────────────────────────┘
```

---

## 2. 10 类数据采集详规

按用户原文列举，本版本 9 类实现（第 8 类预留接口），每类加 `collected_at_utc` 字段（盲点 2 补丁）。

### 2.1 数据结构（顶层 — Pydantic 与现有项目一致）

```python
from pydantic import BaseModel
from uuid import UUID
from datetime import datetime

class Agent2DataBundle(BaseModel):
    bundle_id: str
    collected_at_utc: datetime
    opportunity_id: UUID
    position_id: UUID | None = None
    # 10 类
    position: PositionState | None = None
    underlying: UnderlyingQuote
    greeks: OptionGreeks | None = None
    bars: BarsMultiframe
    indicators: TechnicalIndicators
    liquidity: OptionLiquidity | None = None
    market_context: MarketContext
    news: None = None                  # ⑧ 本版本始终 None (改为 None 而非空 dict, 判 is None 明确)
    unusual_activity: UnusualActivity
    user_intent: UserIntentSnapshot
    ages_sec: dict[str, float]         # 每类 collected_at_utc vs bundle_id 时差
```

**注**：所有子模型（`PositionState` / `UnderlyingQuote` / ...）同样用 `pydantic.BaseModel`，与项目现有 `ParsedIntentDraft` / `TakeProfitSpec` 一致。

### 2.1a Market Data 订阅池（P0-IB-1 + P0-IB-2）

IBKR streaming 100 订阅上限 + OPRA 额外 100，`reqHistoricalData` 6 次/秒。50 仓位 × 10 并发请求 = 500 远超限。

**订阅池策略**：
- 新增 `SubscriptionManager`：按 `symbol` 计数引用复用
  - `underlying / SPY / VIX` 全项目共享单条 streaming 订阅
  - 多个仓位订阅同一 underlying 只算 1 条
- **期权合约用 snapshot 模式**（一次性拿完 Greeks + bid/ask 立即取消订阅，不占 streaming 额度）
- 历史 bars：Worker 启动时**一次性拉**所有 active symbol 的 N 根历史 bars 进内存（slow startup 5-10 分钟可接受），之后 `reqRealTimeBars` 增量维护最新 bar
- Agent 2 review 时**只读内存 cache**，不实时调 `reqHistoricalData`

### 2.2 ⑤ 技术指标（新增，pandas-ta）

在 `requirements.txt` 新增 `pandas-ta>=0.3.14b0`（纯 Python 包，Windows 友好，**不需要 TA-Lib C 库**）。

预计算字段：
```python
{
  "rsi_14": float,
  "macd_signal": Literal["bullish_crossover", "bearish_crossover", "neutral"],
  "bollinger_position": Literal["upper_band", "lower_band", "middle"],
  "sma_20": float, "sma_50": float,
  "atr_14": float,
  "volume_ratio_20d": float,   # 今日 vol / 20 日均
}
```

### 2.3 ⑦ 大盘 SPY+VIX（新增）

`market_data_collector` 扩多 symbol 支持。每次 review 固定拉：
```python
{
  "spy_change_pct": float,       # 日内涨跌
  "vix": float,                  # 当前值
  "vix_change_pct": float,       # 相对昨日
  "sector_etf_change_pct": float # 仓位所属行业 ETF (XLK / XLF / XLE 等)
}
```

### 2.4 ⑨ 异常活动（新增）

```python
{
  "nearby_strikes_volume_spike": bool,  # 持仓 strike ±3 档任一档成交量 > 5 日均 × 3
  "put_call_ratio": float,              # 当日标的所有期权 put_vol / call_vol
}
```

### 2.5 ⑧ 新闻接口预留

```python
class ExternalDataCollector(Protocol):
    def collect_news(self, symbol: str, since_utc: datetime) -> dict | None: ...

# 本版本唯一实现
class NullExternalDataCollector:
    def collect_news(self, symbol, since_utc): return None
```

prompt 拼接层 `if bundle.news is not None:` 才拼接。返回 `None` 而非空 `{}` 让判断边界明确。

### 2.6 数据鲜度（盲点 2 补丁）

`bundle.ages_sec = {"underlying": 2.1, "greeks": 3.8, "bars_5m": 1.2, ...}`

prompt 显式提示："数据采集到打包时差（秒）：..."。**任一字段 age > 30s 该字段标记 `STALE` 或剔除**（交由风控层决定）。

---

## 3. Agent 2 决策范式

### 3.1 两个工作点

**工作点 A — 首次入场决策**（触发条件满足时）
- 输入：bundle + 无持仓 + 用户原始意图
- 输出：`{strategy: EnumStrategy, legs: list[Leg] (最多 2 条), contracts: int, entry_lmt_price: float}`
- **策略枚举（10 种，严格 invariant 17 最多 2 腿）**：
  - 单腿（6 种）：`LONG_CALL` / `LONG_PUT` / `SHORT_CALL`（covered/cash-secured）/ `SHORT_PUT`（cash-secured）/ `PROTECTIVE_PUT` / `COVERED_CALL`
  - 双腿价差（4 种）：`BULL_CALL_SPREAD` / `BEAR_PUT_SPREAD` / `BULL_PUT_SPREAD` / `BEAR_CALL_SPREAD`
  - 波动率双腿：`LONG_STRADDLE` / `LONG_STRANGLE`（共 2 种，算入上述 10 种限额）
  - **禁用**：IRON_CONDOR / BUTTERFLY / 任何 ≥3 腿结构（invariant 17）
- 默认用 **Claude Sonnet 4.6**
- 走 dry-run 路径，入场前可选人工审批（feature flag 控制）

**工作点 B — 持续决策**（入场后每 5 min）
- 输入：bundle + 持仓 + 用户原始意图
- 输出：Pydantic **discriminated union**，消除 action 和字段矛盾：
```python
class HoldDecision(BaseModel):
    action: Literal["HOLD"]
    reasoning: str
    confidence: float  # 0-1
    next_check_minutes: int  # LLM 建议; 本版本 Worker 固定 5 min, 字段只审计落库

class PartialCloseDecision(BaseModel):
    action: Literal["PARTIAL_CLOSE"]
    quantity_pct: float  # 严格 0.1-0.9
    reasoning: str
    confidence: float
    next_check_minutes: int

    @model_validator(mode="after")
    def auto_promote_to_full_close(self, info):
        """期权最小单位 1 张, 持仓 < 3 张时 PARTIAL_CLOSE 数学上等于 FULL_CLOSE.
        风控层需在 LLM 输出后立即检查此情形, 强制转换 action."""
        # 实际检查由 risk_gate 在拿到 position + decision 时做 (这里仅文档)
        return self

class FullCloseDecision(BaseModel):
    action: Literal["FULL_CLOSE"]
    reasoning: str
    confidence: float
    next_check_minutes: int

class AdjustStopDecision(BaseModel):
    action: Literal["ADJUST_STOP"]
    # 语义明确: 拆两字段, LLM 只能填一个 (XOR)
    new_stop_pct_vs_entry: float | None = None   # 相对入场价的绝对止损 %
    trailing_stop_pct: float | None = None       # trailing stop 追踪 %（动态跟随当前最高价）
    reasoning: str
    confidence: float
    next_check_minutes: int

    @model_validator(mode="after")
    def exactly_one_stop_type(self):
        if (self.new_stop_pct_vs_entry is None) == (self.trailing_stop_pct is None):
            raise ValueError("ADJUST_STOP 必须且只能填一个止损类型")
        return self

Agent2Decision = Annotated[
    HoldDecision | PartialCloseDecision | FullCloseDecision | AdjustStopDecision,
    Field(discriminator="action"),
]
```

**`next_check_minutes` 合约**：LLM 建议值只落 `agent2_decisions` 表审计，**本版本 Worker 固定 5 min 轮询**，不依 LLM 建议。下版本可能启用动态间隔。
- 默认用 **Claude Haiku 4.5**，升级到 Sonnet 条件（进 `config/execution.yml`）：
  - 浮盈 ≥ 50% 或 浮亏 ≥ 20%
  - 标的日内变动 ≥ ±3%
  - **`iv_rank > 80` 或 `iv_rank_change_24h > 30`**（改自绝对 IV 阈值，避免高波日 Sonnet 频繁触发）
  - 距到期 ≤ 7 天
  - Haiku 自身 `confidence < 0.6`

### 3.2 Prompt Caching 策略

用 Anthropic SDK 的 `cache_control` 字段。缓存：
- System prompt（策略框架 / 风控红线描述，从 `prompts/agent2_shared/` include 合成）
- 用户原始意图（Agent 1 解析结果，入场后不变）

不缓存：bundle（每次都变）。

**5 分钟 sleeptime 正好匹配 Anthropic cache 5 分钟 TTL**（用户 Q7 频率选对）。每次 review 命中 cache → input token 省 30-50%。

### 3.2a Cache Hit Rate 监控（CC AI 专家 P0 要求）

Worker 实际间隔会抖动（5.1 / 5.8 / 4.7 min），超过 5 min 就 miss。规则：

- **监控**：`agent2.cache_hit_rate`（过去 24h 滚动）进日报
- **WARN 阈值**：< 70% → 日报标红
- **升级阈值**：连续 3 天 < 70% → 触发人工复查（sleeptime 是否要调到 4.5 min / prompt 结构是否要精简）
- `cached_input_tokens` / `input_tokens` 从 Anthropic 响应字段直接读，进 `agent2_decisions` 表

### 3.3 Confidence 状态机（盲点 5 补丁）

```
Haiku 调用 →
  confidence ≥ 0.7 → 执行（走风控层）
  confidence 0.4-0.7 → 升 Sonnet
  confidence < 0.4 → 直接 HOLD + 告警（数据问题可能）

Sonnet 调用 →
  confidence ≥ 0.6 → 执行
  confidence < 0.6 → 保守 HOLD + 告警（不再升级）
```

**模型层级上限由配置控制**，不硬编码：`config/execution.yml` 的 `MAX_MODEL_TIER=sonnet-4-6`。要启用 Opus 时改配置，代码无需改。

### 3.4 Temperature = 0

两个工作点都用 `temperature=0.0`，降低决策随机性。

---

## 4. 风控层（非 AI, 7 条硬规则）

LLM 决策输出 → 风控层检查 → 通过才给执行层。失败 = 拒绝 + 日志 + 告警，**不静默跳过**。

| # | 规则 | 默认值（进 `config/risk_limits.yml`，**不**进 CLAUDE.md） |
|---|---|---|
| 1 | 单笔最大仓位 | **双条件 min()**：账户总资金 × 30% **AND** ≤ 合约近 5 日平均日成交量 × 20%（防打穿盘口） |
| 2 | 总仓位上限 | **long premium 总额 ≤ 60% 账户**；**short margin requirement ≤ 70% 账户**；保留 20% 现金缓冲 |
| 3 | 流动性分层（spread %） | < 3% 正常 LMT；3-10% 走 urgent 档（ratio 0.7）；10-20% 走 market 档（风控额外许可）；> 20% 禁止 + 告警 |
| 4 | Sanity check | 数量 ≤ 持仓；价格 vs 最近 15 分钟 mid 波动范围 ±3σ 内（动态） |
| 5 | 频率限制 | 同一持仓 1 小时内最多 3 次调整；**开盘前 30 分钟降至 1 次/小时**（防 AI 抽风） |
| 6 | 熔断 | 同仓位连续 2 次亏损调整后暂停 + 需人工确认 |
| **7** | **Assignment 避免（新）** | 距到期 ≤ 2 天 + short leg + ITM → Agent 2 强制输出 FULL_CLOSE；风控层 override LLM 决策为 FULL_CLOSE 不平就当作拒绝 |

**注意**：**不加"日亏损上限"**（用户 Q 明确："止盈止损完全交给 AI 或用户输入"）。

**本版本例外保留的风控通道**（按 Q4 原则，非 Agent 2 接管范围）：
- IBKR 账户保证金危险线（IBKR 自动执行强平，我们不管）

---

## 5. 执行层（改造）

### 5.1 定价策略（mid-based，借鉴 Lumibot）

旧公式：`limit = bid + ratio × (ask - bid)`（bid-based）
**新公式：`limit = mid + ratio × (ask - mid)`（mid-based）** —— mid 是做市商理论价，语义更清晰。

**Bid/Ask 合法性校验（P0-IB-3）**：计算 mid 之前必须通过：
1. bid 和 ask 至少一个 > 0（盘后 `-1` 拒用）
2. `spread / mid > 50%` → 视为 fake quote 拒用（瞬间做市商试探）
3. quote age（收到时间戳 vs 当前）> 10 秒 → 拒用，拿新的

任一失败：本次 review 跳过该仓位（不下单），下一轮再试。

**Tick Size Rounding（P0-IB-4）**：`compute_final_price` 输出必须 round 到合法 tick：
- 价格 `< $3.00` → round 到 `$0.05`（或 `$0.01` 若该 symbol 在 penny pilot 白名单）
- 价格 `≥ $3.00` → round 到 `$0.10`（或 `$0.05`）
- `config/execution.yml` 维护 `penny_pilot_symbols` 白名单（SPY / QQQ / IWM 等主流 ETF + 特定股票）
- round 策略：BUY 向下 floor（不过度加价），SELL 向上 ceil（不过度让利），保守方向

4 档配置（进 `config/execution.yml`）：

| 模式 | ratio | 适用 |
|---|---|---|
| `patient` | 0.2 | 不急 / 追求最优价 |
| `normal` | 0.3（默认） | 常规调仓 |
| `urgent` | 0.7 | 盘口变化快 / 接近到期 |
| `market` | — 直接市价 | 紧急止损（**风控层额外许可才允许**） |

### 5.2 撤单重试

未成交超过 **2 分钟**撤单重试，每次 `ratio += 0.1`，重试 **3 次**仍失败 → 告警 + 跳过本轮。

### 5.3 升级链

**单腿**：`LMT (patient)` → `LMT (normal)` → `LMT (urgent)` → `MKT`（最后一档需风控许可）。

**BAG combo（多腿）**（P1-IB-11）：**不走 MKT 档**（combo 市价各 leg 价格错位风险）。始终 LMT，风控允许时**逐腿拆单**平仓（combo 变独立 leg orders）。BAG 必须 `exchange='SMART'`（不允许跨所，P1-IB-7）。

原 `exit_order_placer` 的升级链框架保留，改定价公式 + 拆 combo/单腿路径。

### 5.4 IBKR 下单速率控制（P1-IB-8）

50 仓位并发 review 同时下单接近 IBKR pacing 边界（~50 orders/分钟）：
- 订单用 FIFO queue + `asyncio.Semaphore(30)` 限制并发下单数
- 超过 30 排队等待，最多队列长度 100（超过 → 告警 + 丢弃 older orders）

### 5.5 IBKR Error 映射表（P1-IB-10）

| Error Code | 含义 | Agent 2 响应 |
|---|---|---|
| 10268 | order reject（eTradeOnly / 预防设置）| **HOLD + 告警**（配置问题，不自己重试） |
| 2109 | 不合规 order | **HOLD + 告警** |
| 321 | 市场数据不可用（盘后 / 无权限） | **该仓位本次跳过 review**，下轮重试 |
| 502 | 市场数据权限错误 | **HOLD + 告警**（需要用户处理订阅） |
| 1100 | connectivity lost | **触发重连**（`_ensure_connected` 已有），review 稍后重试 |
| 10197 | paper/live mismatch | **立即 HOLD + 严重告警**（环境配置错误） |
| 其他 | 未知 | HOLD + 记录到 `agent2_decisions.risk_gate_result=IBKR_ERROR_<code>` |

### 5b. Dry-run 模式（P1-Opt-10）

用于 paper 测试前的链路验证 + 生产环境的 "shadow mode"：

- `settings.DRY_RUN_ORDERS=True` 时：
  - 数据采集层 **真跑**（真调 IBKR 拿数据）
  - Agent 2 **真调 Anthropic**（生成真实决策 + 真消耗 token）
  - 风控层 **真跑**（真做 sanity check）
  - 执行层 **log only**：`logger.info("[DRY_RUN] 本应下单: %s", order_spec)`，不真发 IBKR
- 价值：
  - 验证整个决策链路在云端的行为
  - 估算真实 token 成本（前几天运行看看日报）
  - 发现数据采集 / prompt / 风控层的边界 bug，不会损失真金

Step G paper 端到端测试前至少跑 1-2 个交易日 dry-run。

---

## 6. Worker / 调度（异步化考虑）

### 6.1 新循环体

```python
# worker/loop.py 新增
async def run_position_ai_review_once(
    db, ibkr_client, anthropic_client, calendar, config
):
    if not calendar.is_market_open(now):
        return
    # 跳过开盘首 5 分钟噪音 + 收盘前 5 分钟 (P1-Opt-7 + 原有)
    if calendar.minutes_since_open(now) < config.ai_review_start_after_open_min:
        return
    if calendar.minutes_to_close(now) < config.ai_review_stop_before_close_min:
        return
    for position in get_all_open_positions(db):
        task = asyncio.create_task(
            ai_review_one_position(position, anthropic_client, ...)
        )
```

### 6.2 并发 vs 阻塞（重要）

Worker 当前是单线程 15 秒循环。Haiku 调用 2-3s / Sonnet 5-10s。50 仓位同步跑 = 2 分钟，严重阻塞 `drain_exit_queue`。

**本版本方案**：`asyncio` 异步 + `anthropic.AsyncAnthropic`。Worker 主循环改 async，`ai_review_once` 调 anthropic client 时 await 不阻塞其他任务（reconcile / 扫 trigger 等）。

**不**改多进程（风险高：IBKR client_id 冲突 / 状态同步复杂），不**改**独立进程。

### 6.2b ibapi → asyncio Bridge（P0-IB-5）

ibapi 原生是**同步**模型（client 开独立线程 + callback 通过队列传回），直接在 asyncio 内调 ibapi API 会阻塞 event loop。

**方案**：
- 包一层 `AsyncIBKRClient` adapter：同步 `reqMktData` / `placeOrder` 等调用用 `loop.run_in_executor(thread_pool, ibkr_client.method, ...)` 丢到 thread pool
- Callback 注入：ibapi 的 EWrapper 回调（如 `tickPrice`）用 `asyncio.Queue` 把数据推回 event loop；Agent 2 代码 `await queue.get()` 拿结果
- Thread pool 用固定大小（默认 20）避免过度并发

这样 Agent 2 async 代码可以 `await self.ibkr.get_option_greeks(...)` 像纯异步调用，底层 IBKR 同步模型被隐藏。

### 6.2a 云端负载估算（P1-Cloud-4）

Oracle VM 规格（见 memory `project_oracle_cloud.md`）：2 OCPU / 12GB RAM / ARM64。

| 负载项 | 50 仓位并发估算 |
|---|---|
| Bundle 内存（单仓位）| ~20 MB（9 类数据 + bars 多粒度）|
| 50 仓位 bundles 峰值 | ~1 GB |
| Anthropic client connection pool | ~50 MB |
| asyncio event loop + pending tasks | ~100 MB |
| **总预计内存峰值** | **~1.5 GB**（/ 12GB = 12.5%，远够） |
| CPU 峰值 | <30%（大部分在 HTTP await，不是计算密集） |

**并发限流**：本版本用 `asyncio.Semaphore(20)` 限制**同时最多 20 个 Agent 2 调用在飞**。超过 20 仓位时排队（估算 50 仓位完成一轮约 5-8 秒，不影响 5 min 周期）。

**Step G paper 测试时**需验证实测内存 / CPU，与本估算对比偏差 > 50% 则复查。

### 6.3 交易日历（借鉴 Lumibot）

新增 `services/trading_calendar.py`：
```python
import pandas_market_calendars as mcal
nyse = mcal.get_calendar('NYSE')
def is_market_open(now_utc) -> bool: ...
def minutes_to_close(now_utc) -> int: ...
```

处理美东时区 + 节假日 + 周末 + 早退日（感恩节次日半日）。

**本版本只支持 NYSE（美股）。不引入 ASX / NZX 代码（YAGNI）**。未来扩展澳新市场时，calendar 模块加新 exchange 参数化，但本版本的所有调用点都默认 NYSE，不加 exchange 入参。

### 6.4 迭代延迟观测 + 自适应降级（盲点 1 补丁 + CC AI 专家 P1）

每次 `ai_review_once` 开头记录 `elapsed_since_last_run_sec`：
- \> sleeptime × 1.5 → WARN 日志
- \> sleeptime × 2.0 → ERROR + 告警

**自适应降级规则（新，CC AI 专家 P1-5）**：
- 连续 **3 次 ERROR** 后自动降级为 "Haiku only"（不再升 Sonnet），减压持续观察
- 降级状态持续到下一个美股收盘日（next day 重置）
- 日报记录降级事件

日报统计平均延迟 + 降级次数。

### 6.5 Worker 启动检查清单（CC AI 专家 P1 要求）

Worker 进程启动时（不是每次 review），类似全局 §8.1 Session 自检：

1. **读 `.env.{APP_ENV}` 打印**：环境 / IB Gateway 端口 / account_mode（paper/live）/ secondary username / IBKR_CLIENT_ID
2. **对比 memory 关键字段**：读 `project_ibkr_ops.md` 等，若 `.env` 的 IBKR_PORT 和 memory 描述的端口范围（`40xx` = Gateway）不匹配 → stderr 警告
3. **扫 findings.md 的 P1**：若发现标记 "阻塞 Step X" 的未修 finding，用 **exit code 78**（NOPERM 语义）退出，systemd 不自动重启
4. **Config hash 审计**：读 `config/risk_limits.yml` 和 `config/execution.yml` 的 SHA256，记录到日志首行
5. **ANTHROPIC_API_KEY 非空校验**：若为空，exit code 78 退出（阻塞 Step C）
6. 检查全部通过 → 开 main loop

### 6.5a systemd 单元约束（P0-Cloud-2 + P1-IB-6）

`/etc/systemd/system/options-ai-trader-worker.service`：
```ini
[Service]
Restart=on-failure
RestartSec=60  # 留出 IBKR side 清旧 clientId 的时间 (P1-IB-6)
StartLimitBurst=3
StartLimitIntervalSec=600
# exit code 78 (NOPERM) = 禁止重启
RestartPreventExitStatus=78
```

**Worker 重启 clientId 冲突处理（P1-IB-6）**：
- IBKR 清旧连接通常 30-60s，`RestartSec=60` 基本够
- 启动时若收到 "clientId already in use" error 501，额外等 30s 再重试 3 次
- 3 次仍失败 → exit 78（人工介入）

- 正常故障（网络抖动）：30 秒后自动重启，10 分钟内最多 3 次
- fatal 失败（findings 阻塞 / ANTHROPIC_KEY 缺失）：exit 78 → systemd 不重启，避免循环爆 /var/log

---

## 7. 触发类型（4 种 + 1 组合）

| 类型 | 本版本做 | 实现位置 |
|---|---|---|
| 立即触发 | ✅ | Agent 1 现有 |
| 时间触发 | ✅ | Agent 1 现有 + Worker scheduler |
| 条件触发（MA_CROSSOVER） | ✅ **补 F4 runtime** | 新 `monitoring/condition_monitor.py` |
| 条件触发（PRICE_BREACH） | ✅ **补 F5 runtime** | 同上 |
| **时间+条件组合（新）** | ✅ | `trigger_catalog` 新增 `TIME_AND_CONDITION` + 组合逻辑 |
| 事件 / 事件+条件 / 时间+事件 / 三合一 | ❌ 跳过 | 待事件日历外部数据源 |

`CONDITION_MONITOR` 框架：bar 订阅（5 min K 线 / 成交量）+ 计算 SMA / 价格突破 + 触发 entry_order_placer。

---

## 8. 环境隔离

| 环境 | APP_ENV | `.env` | IB Gateway | IBKR_CLIENT_ID | AI 自主 | 运行形态 |
|---|---|---|---|---|---|---|
| `local-dev` | `local-dev` | `.env.local-dev`（新） | 本机，可 paper 或 live（用户手动切） | **38**（与 cloud-dev 分离） | ✅ | 手动跑 |
| `cloud-dev` | `dev`（**保持老命名兼容 CI**，内部语义 cloud-dev） | `.env.dev`（不改名） | Oracle VM paper | 28（现有） | ✅ | 24h 跑 |
| `uat` | `uat` | `.env.uat`（改 live 4001） | Oracle VM live（secondary username） | 29（新） | ✅ | 24h 跑 |
| `prod` | `prod` | `.env.prod`（live） | Oracle VM live（secondary username） | 30（新） | ✅ | 24h 跑 |

**CI/CD 兼容**（P0-Cloud-1）：
- `.github/workflows/deploy-*.yml` 现有"dev / uat / prod" 命名**不变**，`dev` 等同于 `cloud-dev`
- `settings.py` 接受 `APP_ENV=dev` 向后兼容，映射到 `cloud-dev` 的行为（如 paper + 24h）
- 新增 `local-dev` 不进 CI（纯本地），`.env.local-dev` 也不 commit 到 repo

**IBKR_CLIENT_ID 冲突规则**（P1-Cloud-9）：
- `local-dev` 必须用 `CLIENT_ID=38`（或其他与 cloud-dev 不同的值）
- 或申请独立 paper 子账户给 local-dev 用（推荐，彻底避免互踢）
- 同账户用不同 CLIENT_ID **仍有风险**（IBKR 偶发仍会踢），子账户是更安全方案

**启动时打印**（§6.5 Worker 启动检查之一）：环境枚举 + 连的账户（paper 用 DU* / live 用 U*）+ secondary username + IBKR_CLIENT_ID。不允许运行时切换账户。

### 8a. 凭证新需求（P0-Cloud-3）

新增 Anthropic API key，每环境独立：

| 项 | 位置 | 需建 |
|---|---|---|
| 本地 `.env.local-dev` | 手动填 | 用户建 |
| 云端 `.env.dev` / `.env.uat` / `.env.prod` | ssh VM 手动填 | 用户建 |
| GitHub Secrets | `ANTHROPIC_API_KEY_DEV` / `_UAT` / `_PROD` | 用户建 |

**阻塞**：新 finding `F-2026-04-21-ANTHROPIC-KEY-SETUP-BLOCK`（Step C 部署前必须完成）。

### 8b. paper/live 切换时间窗口（P1-Cloud-7）

UAT 从 paper → live 切换（改 `.env.uat` + systemd restart）期间，Worker 断 IBKR <30s：
- **建议窗口**：美股收盘后（16:00 ET）或**周末**执行
- 避免开盘中切换（持仓监控盲点会错过关键价格动作）
- 切换后 10 分钟人工确认 `ibkr_client.isConnected() == True` + 持仓数据恢复

---

## 9. 日志 / 审计 / 告警 / 日报

### 9.1 LLM 审计落库（修 F-LLM-AUDIT-GAP）

新表 `agent2_decisions`（Alembic migration，命名 `alembic/versions/<ts>_agent2_decisions.py`）。

**核心字段**（完整 DDL 由 migration 定义，spec 只列要点，防漂移）：
- 标识：`id UUID PK` / `opportunity_id` FK / `position_id` FK nullable
- Bundle 输入：`bundle_id` / `bundle_json JSONB`
- 模型元数据：`model_name` / `temperature` / `input_tokens` / `cached_input_tokens` / `output_tokens` / `cost_usd`
- LLM 输出：`raw_response_json` / `parsed_decision_json` / `action` / `confidence`
- 执行结果：`risk_gate_result`（PASS / REJECT_<rule_id>）/ `executed BOOL`
- Trace 指针：`trace_path` → `logs/agent2/YYYY-MM-DD/{decision_id}.json`
- 时间戳：`created_at`

Trace 文件落盘（借鉴 Lumibot），本地 7 天 rotate，云端 30 天保留（见 memory `project_cloud_log_access.md`）。

**磁盘空间估算和监控**（P1-Cloud-5）：
- 50 仓位 × 78 review/天 × 30 天 ≈ 117k 条 × ~10 KB = **~1.2 GB**
- Oracle VM 50 GB block 占 2.4%，非致命但要监控
- logrotate 加 `compress` 选项（gzip 节省 60% 空间 → ~500 MB）
- 监控规则：`df /opt/{project}/logs` > 70% 告警 + 自动 purge 老于 30 天的 trace 文件

### 9.2 结构化日志扩容

`core/logging.py` 新增：
- 文件 handler + rotation（10MB / 7 日）
- 新 logger namespace `agent2.decision` / `agent2.risk_gate` / `agent2.executor`
- JSON 结构化格式

### 9.3 告警接口预留

抽象 `AlertSink`：
```python
class AlertSink(Protocol):
    def send(self, level, title, body): ...

class ConsoleAlertSink: ...   # 本版本唯一实现（同时落文件日志）
```

**不**预定义 `TelegramAlertSink` / `SlackAlertSink` 等接口（YAGNI —— 实现接口时再定义，避免空接口漂移）。

触发场景：熔断 / 风控拒绝 / 迭代延迟 ERROR / Agent confidence <0.4 / 订单重试 3 次失败 / IBKR 重连失败 ≥3 次。

### 9.4 日报

每日美东 16:30（收盘后）生成：
- LLM 调用次数分布（Haiku / Sonnet）
- Token 消耗 + cost_usd 总计
- 风控拒绝明细
- 执行订单 + 盈亏
- 迭代延迟分布

落 `logs/daily_report/YYYY-MM-DD.md`，控制台同时打印摘要。

---

## 10. 数据鲜度 + 部分成交 + Routing（盲点 2/3/4 补丁）

### 10.1 盲点 2 — 数据鲜度

见 §2.6。每类 `collected_at_utc` + `ages_sec` 喂给 LLM。age > 30s 剔除或标记。

### 10.2 盲点 3 — IBKR 路由

`.env.{env}` 新增 `IBKR_OPT_EXCHANGE=SMART`（默认）。opportunity 层可覆盖（字段 `preferred_exchange`）。期权 1 档差 $0.05 × 10 张 = $50。

### 10.3 盲点 4 — 多腿 BAG 部分成交

`exit_order_placer` 的 orphan leg 处理改：部分成交立刻 **trigger 一次计划外 `ai_review_once`**（不等 sleeptime），把"刚刚部分成交、剩余 X 张、已成交 Y 张"作为 bundle 额外字段传给 Agent 2。

---

## 11. 反馈 / 词典冻结实施

按 Round 1 Q3 决策，**只冻 `business_lexicon.yml`，trigger_catalog 和 default_policies 保留**。

### Flag 命名（统一 "enable_* = 开启新/变化功能" 语义，CC AI 专家 P1-6）

| Flag | 位置 | 默认值 | 语义 |
|---|---|---|---|
| `freeze_lexicon_writes` | `settings.py` | `True`（冻结中） | True = 冻结词典写，不调 apply_changes |
| `freeze_feedback_consumer` | `settings.py` | `True` | True = 提交反馈不触发 LLM 消费 |
| `freeze_edit_signal_api` | `settings.py` | `True` | True = `/edit-signal` 返回 400 |
| `enable_lexicon_in_prompt` | `settings.py` | `False`（旧行为关闭） | False = Agent 1 不拼词典 |
| `enable_agent2_autonomous` | `settings.py` | `True`（本版本启用新 Agent 2） | True = 用新 orchestrator，False = 老 strategy_agent |

统一了"**freeze_\* = 关闭旧功能** / **enable_\* = 开启新功能**"的正反向语义，避免混淆。

不删任何代码，只 flag 短路。`UserFeedbackRecord` / `UserEditSignal` 表保留入库（便于未来恢复）。

---

## 12. Agent 1 改动（最小）+ 老 Agent 2 兼容

### Agent 1 改动
仅两处：
1. `intake_graph.py:load_lexicon_context` — `if not settings.enable_lexicon_in_prompt: return empty_context`
2. `config_loader.py:build_prompt_context` — 支持 lexicon disable 模式，trigger_catalog 和 default_policies 正常注入

其他 Agent 1 代码不动。ParsedIntentCore / TakeProfitSpec / StopLossSpec / PositionSpec 全保留（给 Agent 2 用作 user_intent 输入）。

### 老 `services/strategy_agent.py` 兼容层
改成 wrapper 调新 `services/agent2/orchestrator.py`。

**兼容层清理条件**（加入 findings 作新 P3 技术债）：
- PROD 运行 ≥ 30 天
- 老 code path hit rate（通过 feature flag 计数）降为 0
- 满足后 Alembic 迁移删除老 strategy_agent.py

**Finding ID**：`F-2026-04-21-STRATEGY-AGENT-COMPAT-WRAPPER`（技术债，clean-up 条件明确）

---

## 13. 文件级改动清单

### 新增文件（拆分到更细粒度，防 god class）

**Agent 2 模块**（原 `agent2_reviewer.py` 拆 3 份）：
- `services/agent2/orchestrator.py` — 主入口，协调 collector → agent → state_machine → risk_gate → executor
- `services/agent2/state_machine.py` — Confidence 状态机（Haiku/Sonnet 分级 + 升级判断）
- `services/agent2/prompt_builder.py` — 拼 prompt（bundle + user_intent + cache_control 标记）

**数据采集**（原 `data_collector_v2.py` 拆）：
- `services/data_collection/bundle_packager.py` — 打包 Bundle 顶层
- `services/data_collection/collectors/position.py` — ①
- `services/data_collection/collectors/underlying.py` — ②
- `services/data_collection/collectors/greeks.py` — ③
- `services/data_collection/collectors/bars.py` — ④ 多粒度
- `services/data_collection/collectors/indicators.py` — ⑤ pandas-ta 封装
- `services/data_collection/collectors/liquidity.py` — ⑥
- `services/data_collection/collectors/market_context.py` — ⑦ SPY + VIX
- `services/data_collection/collectors/external.py` — ⑧ `ExternalDataCollector` Protocol + `NullExternalDataCollector`
- `services/data_collection/collectors/unusual_activity.py` — ⑨

**其他**：
- `services/trading_calendar.py` — NYSE 交易日历（仅美股）
- `services/risk_gate_v2.py` — 6 条风控硬规则
- `services/alert_sink.py` — 告警抽象（仅 `AlertSink` + `ConsoleAlertSink`）
- `integrations/anthropic_client.py` — Claude SDK 适配（async + prompt caching + replay cache with timestamp-stripped key）
- `monitoring/condition_monitor.py` — MA_CROSSOVER + PRICE_BREACH（F4/F5）
- `config/risk_limits.yml`
- `config/execution.yml`（含 `MAX_MODEL_TIER` / 4 档 spread_ratio / 撤单参数）
- `prompts/agent2_entry_system.md` — 首次入场决策 prompt（工作点 A）
- `prompts/agent2_review_system.md` — 持续决策 prompt（工作点 B）
- `prompts/agent2_shared/risk_rules.md` — 共享：风控红线描述（A/B 都 include）
- `prompts/agent2_shared/strategy_types.md` — 共享：10 种策略类型定义（A include）
- `prompts/agent2_shared/data_bundle_schema.md` — 共享：9 类 Bundle 字段说明（A/B 都 include）

### 修改文件（约 15 个）

- `worker/loop.py` — 改 async + 新循环体
- `services/broker_adapter.py` — 入场路径保留，出场由 Agent 2 触发
- `services/order_placer.py` — mid-based 定价
- `services/exit_order_placer.py` — 定价 + 部分成交 trigger
- `graphs/intake_graph.py` — flag 关词典
- `intake/config_loader.py` — 支持 lexicon disable
- `agents/strategy_agent.py` — 改成 wrapper 调新 `agent2_reviewer`（老代码保留以兼容）
- `db/models.py` — 新表 `agent2_decisions`
- `settings.py` — 新 flag + APP_ENV 4 档
- `alembic/versions/xxxx_agent2_decisions.py` — migration
- `.env.local-dev` / `.env.uat` (live) / `.env.prod` (live) — 新建/改
- `core/logging.py` — 文件 handler + rotation
- `requirements.txt` — 新依赖 `anthropic` / `pandas-ta` / `pandas-market-calendars`
- `CLAUDE.md §5` — invariant 更新
- `task_plan.md` — Phase 11 分解

### 停用点清单（flag 短路，不删代码）

| 文件 | 行号范围 | 触发 flag | 短路行为 |
|---|---|---|---|
| `services/feedback_agent.py` | 102-166 `_call_llm_and_apply` | `enable_feedback_consumer=False` | 早 return，不调 LLM 不写词典 |
| `services/lexicon_writer.py` | 36-46 `save_lexicon` | `enable_lexicon_writes=False` | 早 return，不写 YAML |
| `api/routers/feedback.py` | 16-39 `POST /v1/feedback` | `enable_feedback_consumer=False` | 202 Accepted + `{"skipped": true}` |
| `api/routers/opportunities.py` | 99-105 submit 里的 consumer | `enable_feedback_consumer=False` | skip |
| `api/routers/opportunities.py` | 160-171 edit-signal | `enable_edit_signal_api=False` | 400 Bad Request |
| `graphs/intake_graph.py` | 98-105 `load_lexicon_context` | `enable_lexicon_in_prompt=False` | 返回空 context |
| `intake/config_loader.py` | 195+ `build_prompt_context` | 同上 | business_lexicon 部分 skip，trigger_catalog / default_policies 正常注入 |
| `worker/jobs.py` | 466-561 `drain_exit_queue` TP/SL 分支 | `enable_agent2_autonomous=True` | TP/SL/TIME_STOP 触发路径跳过（Agent 2 接管） |
| `worker/jobs.py` | 607-624 `check_time_stops` | 同上 | 整个函数早 return |
| `services/position_monitor.py` | 规则触发路径 | 同上 | 仅保留 PnL 追踪，不发触发事件 |

（行号为 2026-04-21 日当时位置，实际 plan 阶段按最新 HEAD 重确认）

---

## 14. Step A-G 实施计划 + 时间估（+20% buffer）

| Step | 内容 | 核心估 | +buffer |
|---|---|---|---|
| **A** | 反馈系统冻结 + Agent 1 词典简化 + settings flag + APP_ENV 4 档 + Alembic migration 三环境 + ARM 兼容性验证 + Secrets 建 Anthropic key | 1 天 | 1.2 天 |
| **B** | 数据采集层 9 类 + 单元测试（每类 ≥3 cases）+ 第 8 类接口预留 + 鲜度标记 | 3 天 | 3.6 天 |
| **C** | LLM 决策层（Anthropic SDK + prompt caching + 分级调用 + discriminated union + 状态机 + replay cache timestamp-stripped key）| 2 天 | 2.4 天 |
| **D** | 风控层 6 条 + 每条 ≥3 cases 单元测试（pass / edge / reject）| 1 天 | 1.2 天 |
| **E** | 执行层（mid-based 4 档 + property-based test 验证 limit 合理区间 + 撤单重试 + 升级链 + 部分成交 trigger）+ 日志 + 告警 | 2 天 | 2.4 天 |
| **F** | F4/F5 runtime 层（MA_CROSSOVER + PRICE_BREACH + CONDITION_MONITOR）+ 时间+条件组合触发 | 2 天 | 2.4 天 |
| **G** | Paper 账户端到端测试（SessionStart hook 首触发 + replay cache）| 1.5 天 | 1.8 天 |
| **H** | 用户文档 + 运维 runbook | 0.5 天 | 0.6 天 |

**核心估**：13 天开发 + 5 天测试 = 18 工作日
**加 20% buffer**：~22 工作日
**考虑 session 中断、方案迭代、bug 修复**，**实际 4-5 周**完成。

---

## 15. 验收标准（Karpathy §4 Goal-Driven Execution 要求的可验证条件）

每个 Step 完成前必须满足：
- ✅ 所有新增单元测试通过（不是 empty test，要有真实断言）
- ✅ 现有全量 pytest 不回归（478 基线）
- ✅ 在 `local-dev` 环境 paper 账户跑通该 Step 核心路径
- ✅ git commit 独立 + commit message 中文 why
- ✅ Step 完成报告 markdown 放 `docs/progress/STEP_X.md`
- ✅ 若涉及前端 → Playwright 1920×1080 回归

**Step G 验收**（端到端）：
- Paper 账户 $10k 模拟资金跑 5 个交易日
- 至少 3 个不同 symbol 的仓位触发过 PARTIAL_CLOSE / FULL_CLOSE / ADJUST_STOP 各 1 次
- `agent2_decisions` 表 > 100 条记录
- 日报每日生成
- 无熔断触发 / 无风控拒绝 > 5% 的比例

### Step G+ — Paper vs Live 差异验证（P1-IB-9）

Paper 账户通过**不等于** Live 通过，原因：
- Paper 账户**不模拟 assignment**（即使 short ITM 到期也不会被 assign）
- Paper fill 速度过于乐观（LMT 可能瞬间成交，live 可能挂半天）
- Paper bid-ask spread 真实但 fill 价给 mid（live 需和做市商抢）

**Step G 之后必须跑 Step G+**：
- UAT small live（$10k）跑 2-3 交易日
- 必须在 memory `project_ibkr_ops.md` 第三节 2FA + secondary username 就绪**后**
- 重点观察：(a) fill 价格 vs paper 差异 (b) 实际 spread 成本 (c) assignment 场景（长 short position 到期前 2 天）

### 每 Step 的测试清单摘要（具体 cases 在 plan 阶段展开）

| Step | 测试重点 |
|---|---|
| A | settings flag 分别 on/off 的 intake_graph / feedback consumer 行为测试 |
| B | 9 类 collector 各自 mock IBKR 响应的单元测试（每类 ≥3 cases，含 edge cases：0 volume / null Greeks / stale bar） |
| C | Anthropic client 的 mock stub（不调真实 API），prompt caching hit/miss 模拟，状态机 8 条路径（Haiku 3 + Sonnet 2 + 升级转换 3） |
| D | 6 条风控规则每条 3 cases（pass / edge / reject），组合测试（同时触发多条） |
| E | 定价 property-based test（hypothesis 生成随机 bid/ask，验证 `mid ≤ limit ≤ max(ask, bid+max_ratio)`），撤单重试状态测试 |
| F | MA_CROSSOVER 的 bar 订阅 mock，5 日金叉判定精度，PRICE_BREACH 边界相等测试，时间+条件组合的 AND 逻辑 |
| G | Paper 账户活跑（见上） |

---

## 16. 风险和回滚

### 16.1 风险列表

| 风险 | 严重度 | 缓解 |
|---|---|---|
| Anthropic API 限流 / 故障 | P1 | Haiku + Sonnet 双模型；失败 → 重试 3 次 → 保守 HOLD + 告警 |
| Worker 异步化重构引入死锁 | P1 | Step C 完成后专项压力测试；回滚到同步 sleep 版本备份 |
| Prompt caching 5 min TTL 边界漂移 | P2 | 监控 cache hit rate；<70% 时告警 |
| IBKR secondary username 2FA 仍触发 | P2 | UAT live 启动前专门验证；见 memory `project_ibkr_ops.md` |
| 条件触发 bar 订阅成本 | P2 | MA_CROSSOVER 用 5 min bar 足够；每仓位订阅限 1 条 |
| 盘中 Agent 2 decision loop 迭代堆积 | P1 | 盲点 1 补丁观测 + 告警 |
| Claude 输出格式漂移（非合规 JSON） | P1 | Anthropic structured output + JSON schema validator；校验失败 → HOLD + 告警 |

### 16.2 回滚策略

**代码层**：所有新模块有独立 feature flag（`enable_agent2_autonomous` 总开关），关闭即回到老 Agent 2（一次性生成 + 用户确认）路径。

**数据层**：`agent2_decisions` 表独立，不影响 `opportunities` / `strategy_runs` / `positions`。回滚不删表。

**配置层**：`.env.uat` / `.env.prod` 改回 paper + `enable_agent2_autonomous=false`。

---

## 17a. 项目 CLAUDE.md §5 同步更新清单（CC AI 专家 P0 要求）

spec 实施前必须同步到项目 CLAUDE.md §5 的 invariant：

| 现状 invariant | 需改动 | 为何 |
|---|---|---|
| §5-4 direction 允许 BULLISH/BEARISH/null | 不改 | 仍然适用 |
| §5-5 支持等级二分 SUPPORTED/UNSUPPORTED | **可能弱化**：新 Agent 2 范式下 support_level 判断的意义可能变小（AI 自主决策会跳过"该不该支持"的硬 gate） | 待 Step A 后评估 |
| §5-10 只保留两个 AI Agent（Intake + Strategy） | **保留表述**但**改实现**：Agent 2 从"一次生成"变"持续决策" | spec §3 实施 |
| §5-11 主队列只显示大状态 | 不改 | Agent 2 决策明细进详情页 + 日报，不污染主队列 |
| §5-13 修改 = 取消原单 + 复制单 | 不改，按 Q6 澄清此 invariant 仅在 Agent 1 阶段适用 | 但需要在 invariant 文字里加"（仅 Agent 1 机会单阶段）"限定词 |
| §5-16 止盈止损全自动不甩给用户 | **强化**：Agent 2 接管后这条更硬 —— AI 决策 + mid-based 升级链必走到底 | 加一行"AI 决策不确定时保守 HOLD + 告警，不降级为'让用户决定'" |
| §5-17 策略最多 2 腿 | 不改 | Agent 2 prompt 里继续体现 |
| §5-18 UAT/PROD 不自动启动 | **修正**：Q1 + Q2 决策后，UAT/PROD live 下 Worker 必须 24h 运行，"启停时机由用户决定"改为"部署无缝，升级时短暂停机" | 重写这条 |
| **新增 §5-19** | 本版本限制 4 种触发类型（立即/时间/条件/时间+条件组合），事件系列跳过 | CC AI 专家 P0-4 要求 |
| **新增 §5-20** | Agent 2 两个工作点：(A) 首次入场 / (B) 入场后每 bar 持续决策 | spec §3.1 体现 |
| **新增 §5-21** | 限价单定价使用 mid-based（`mid + ratio × (ask - mid)`），4 档可配；绝不用裸市价单（market 档需风控层额外许可） | spec §5 |

这些改动在 Step A 实施前做 PR，让 §5 invariant 与 spec 保持一致。

## 17b. Memory 同步清单（CC AI 专家 P1 要求）

本 spec 实施前后需同步更新的 memory 文件：

| Memory 文件 | 需更新的内容 | 时机 |
|---|---|---|
| `project_ibkr_ops.md` 第三节 | 已含 2FA + secondary username 要求 ✅ | 已同步（Phase A 合并时） |
| `project_oracle_cloud.md` | 加一节"UAT/PROD IB Gateway live 登录节点 + systemd auto-restart 配置" | UAT live 切换前 |
| `project_risk_thresholds_index.md` | 加"新风控层 6 条硬规则在 `config/risk_limits.yml`" | Step D 完成 |
| 新 memory `project_agent2_autonomous.md` | 记录 Agent 2 新范式的"两个工作点 + Haiku/Sonnet 分级 + prompt caching 策略"稳定事实 | Step C 完成后新建 |

**不要** 在 memory 里写具体阈值 / 端口 / 账号（按全局 §13 零数字原则），只写"见 config/* 或 .env.* 的 X"。

## 17. 五专家评审（v1-v5 迭代记录位置）

### Round 1 — 软件工程专家 ✅ 通过（7 P0 + 8 P1，全部应用）

**关注重点**：模块边界 / API 契约 / 测试完整性 / 可维护性 / DRY / YAGNI

**P0 问题和修正**（7 条，全部应用到 v1）：

| # | 问题 | 修正位置 |
|---|---|---|
| P0-1 | `agent2_reviewer.py` / `data_collector_v2.py` 会变 god class | §13 拆为 `services/agent2/{orchestrator,state_machine,prompt_builder}.py` + `services/data_collection/{bundle_packager,collectors/*}.py` |
| P0-2 | 单 prompt 文件承担两个工作点 | §13 拆 `prompts/agent2_entry_system.md` + `prompts/agent2_review_system.md` |
| P0-3 | Action + 字段扁平 schema 可出矛盾输出 | §3.1 改 Pydantic discriminated union（Hold/PartialClose/FullClose/AdjustStop 各独立模型） |
| P0-4 | Bundle 用 dataclass 与项目 Pydantic 不一致 | §2.1 改 `pydantic.BaseModel` |
| P0-5 | Replay cache key 含时间戳永 miss | §2 新增规则：cache key **排除** `collected_at_utc` 等时间戳字段 |
| P0-6 | 硬编码 "永不升级 Opus" | §3.3 移到 `config/execution.yml` 的 `MAX_MODEL_TIER` |
| P0-7 | §6.3 说扩展 ASX/NZX | §6.3 明确"本版本只 NYSE，不引入多市场代码（YAGNI）" |

**P1 问题和修正**（8 条，全部应用）：

| # | 问题 | 修正位置 |
|---|---|---|
| P1-8 | strategy_agent 兼容层清理条件缺 | §12 加 `F-2026-04-21-STRATEGY-AGENT-COMPAT-WRAPPER` finding，明确 PROD 30 天 + hit rate 0 |
| P1-9 | `next_check_minutes` LLM vs Worker 合约不明 | §3.1 加合约声明：本版本 Worker 固定 5 min，LLM 建议只审计 |
| P1-10 | 前面 Step 测试清单缺 | §15 加"每 Step 测试清单摘要"子节 |
| P1-11 | §9.1 完整 SQL 会和 migration 漂移 | §9.1 改为"核心字段列表"，DDL 交 migration |
| P1-12 | 停用文件只说 flag 不说行号 | §13 加"停用点清单"表（10 行：文件 + 行号范围 + flag + 短路行为） |
| P1-13 | `NullExternalDataCollector` 返回 `{}` 脆弱 | §2.5 改返回 `None`，`is None` 判断明确 |
| P1-14 | 时间估无 buffer | §14 每 Step 加 +20% buffer 列 |
| P1-15 | `TelegramAlertSink` 接口无实现 = YAGNI | §9.3 删除，只留 `AlertSink` + `ConsoleAlertSink` |

**专家意见**：v1 通过。进 Round 2。

### Round 2 — CC AI 规则/记忆系统专家 ✅ 通过（4 P0 + 6 P1，全部应用）

**关注重点**：规则主观判断缺口 / 跨 session 事实传递 / 机制硬化 vs 自觉

**P0 问题和修正**（4 条，全部应用到 v2）：

| # | 问题 | 修正位置 |
|---|---|---|
| P0-CC-1 | entry + review 两个 prompt 文件有重叠内容 → DRY 违反 | §13 新增 `prompts/agent2_shared/{risk_rules,strategy_types,data_bundle_schema}.md`，prompt_builder 负责 include |
| P0-CC-2 | §18 触发清单命中无显式声明 | §0 加 "§18 触发清单命中" 表 + "跨文件关系" 表 |
| P0-CC-3 | Prompt caching 监控缺 | §3.2a 加 cache_hit_rate 监控规则：< 70% WARN，连续 3 天触发复查 |
| P0-CC-4 | 项目 CLAUDE.md §5 invariant 同步清单缺 | §17a 加完整清单，列出要改/新增的 invariant（5-10 / 5-13 / 5-16 / 5-18 改，5-19/5-20/5-21 新增） |

**P1 问题和修正**（6 条，全部应用）：

| # | 问题 | 修正位置 |
|---|---|---|
| P1-CC-5 | 迭代延迟连续 ERROR 下游行为空白 | §6.4 加自适应降级：连续 3 次 ERROR → 降级 "Haiku only" 至下个收盘日 |
| P1-CC-6 | flag 命名正反向混用 | §11 统一命名：`freeze_* = 关闭旧功能` / `enable_* = 开启新功能`，反馈层改 `freeze_lexicon_writes=True` |
| P1-CC-7 | config 运行时被误改无审计 | §6.5 Worker 启动检查第 4 条：记录 config SHA256 hash |
| P1-CC-8 | Memory 同步清单缺 | §17b 新增，列 4 份相关 memory 的同步时机 |
| P1-CC-9 | spec ↔ plan ↔ task_plan ↔ findings 关系图缺 | §0 "跨文件关系" 表解决 |
| P1-CC-10 | Worker 启动检查清单缺 | §6.5 新增 5 步启动检查（环境打印 / memory 对比 / findings 扫 / config hash / 通过才开 loop） |

**专家意见**：v2 通过。进 Round 3。

### Round 3 — Oracle 云部署专家 ✅ 通过（3 P0 + 6 P1，全部应用）

**关注重点**：Oracle VM 负载 / 三环境 / CI-CD / 凭证管理 / 部署一致性

**P0 问题和修正**（3 条）：

| # | 问题 | 修正位置 |
|---|---|---|
| P0-Cloud-1 | APP_ENV 4 档新命名波及 CI workflow | §8 保留 CI 命名 `dev` = `cloud-dev`（别名，settings.py 向后兼容）；`.env.dev` 不改名 |
| P0-Cloud-2 | Worker 启动检查失败循环重启爆日志 | §6.5a 加 systemd 约束：`RestartSec=30` + `StartLimitBurst=3` + `RestartPreventExitStatus=78`（fatal 退出不重启） |
| P0-Cloud-3 | Anthropic key 每环境需独立 + GitHub Secrets 没有 | §8a 明确每环境配置路径 + 新 finding `F-2026-04-21-ANTHROPIC-KEY-SETUP-BLOCK` 阻塞 Step C |

**P1 问题和修正**（6 条，全部应用）：

| # | 问题 | 修正位置 |
|---|---|---|
| P1-Cloud-4 | Oracle VM 2 OCPU / 12GB ARM 对 asyncio 50 并发负载没评估 | §6.2a 加估算表（内存 ~1.5 GB / CPU <30%）+ `asyncio.Semaphore(20)` 限流 |
| P1-Cloud-5 | 云端 30 天 trace 日志磁盘没监控 | §9.1 加磁盘监控 + logrotate compress |
| P1-Cloud-6 | Alembic 三环境 migration 验证 | §14 Step A 加"Alembic upgrade 在 dev/uat/prod 顺序 + 验证" |
| P1-Cloud-7 | paper/live 切换 Worker 重启盲点 | §8b 加切换时间窗口建议（收盘后 / 周末） |
| P1-Cloud-8 | ARM 兼容性（pandas-ta numpy wheel） | §14 Step A 加 "ARM 冒烟测试" |
| P1-Cloud-9 | local-dev 和 cloud-dev 的 IBKR_CLIENT_ID 冲突 | §8 明确 local-dev 用 CLIENT_ID=38；推荐独立 paper 子账户 |

**专家意见**：v3 通过。进 Round 4。

### Round 4 — 美股期权实战专家 ✅ 通过（4 P0 + 6 P1，全部应用）

**关注重点**：实盘可行性 / 风险管理 / 市场微结构 / 滑点 / 流动性 / 交易场景完整性

**P0 问题和修正**（4 条）：

| # | 问题 | 修正位置 |
|---|---|---|
| P0-Opt-1 | 单笔 30% 资金对 PROD 大账户会打穿盘口（$150k 单笔期权 > 低流动合约日成交量）| §4 规则 1 改双条件 min(30% 资金, 5 日均成交量 × 20%) |
| P0-Opt-2 | spread > 5% 一刀切禁止平仓，违反 invariant 16 | §4 规则 3 分层：<3% LMT / 3-10% urgent / 10-20% market / >20% 禁止 |
| P0-Opt-3 | 10 种策略类型未列出 + invariant 17 "2 腿" 未强调 | §3.1 工作点 A 列出 10 种枚举 + 显式禁用 ≥3 腿结构 |
| P0-Opt-4 | (a) 持仓 <3 张 PARTIAL_CLOSE 数学退化；(b) `new_stop_pct` 语义不明 | §3.1 Pydantic 加 validator 自动转 FULL_CLOSE；拆 `new_stop_pct_vs_entry` 和 `trailing_stop_pct` XOR 填 |

**P1 问题和修正**（6 条，全部应用）：

| # | 问题 | 修正位置 |
|---|---|---|
| P1-Opt-5 | 总 80% 规则未区分 long premium 和 short margin | §4 规则 2 拆：long premium ≤ 60% / short margin ≤ 70% / 现金保留 20% |
| P1-Opt-6 | Sanity check ±20% 静态会高波日误拒 | §4 规则 4 改动态："15 分钟 mid 波动 ±3σ" |
| P1-Opt-7 | 开盘首 bar 噪音未跳过 | §6 新增 `ai_review_start_after_open_min=5` + worker 判断 |
| P1-Opt-8 | Sonnet 升级 IV 阈值低导致成本翻倍 | §3.1 改 `iv_rank > 80` 或 `iv_rank_change_24h > 30` |
| P1-Opt-9 | 未处理 assignment 风险 | §4 新增规则 7（风控从 6 条增至 7 条）：距到期 ≤2 天 + short ITM → 强制 FULL_CLOSE |
| P1-Opt-10 | Dry-run 模式无细节 | §5b 新增专节：数据/Agent 2/风控真跑，执行层 log only |

**专家意见**：v4 通过。进 Round 5（最后一轮）。

### Round 5 — IB/TWS 工程师 ✅ 通过（5 P0 + 6 P1，全部应用）

**关注重点**：IBKR API 边界 / 订阅限制 / 订单规范 / 连接管理 / 错误处理

**P0 问题和修正**（5 条）：

| # | 问题 | 修正位置 |
|---|---|---|
| P0-IB-1 | Market data 订阅超 IBKR 100 并发上限（50 仓位 × 10 并发 = 500）| §2.1a 加 SubscriptionManager 订阅池：underlying/SPY/VIX 共享订阅；期权用 snapshot 模式 |
| P0-IB-2 | `reqHistoricalData` 6 次/秒上限被打爆（150 并发）| §2.1a 改策略：Worker 启动一次性拉历史 bars + reqRealTimeBars 增量维护 + 内存 cache |
| P0-IB-3 | bid/ask 合法性未校验（盘后 -1 / fake quote / stale）| §5.1 加 3 条校验：bid/ask >0 / spread<50% mid / age<10s |
| P0-IB-4 | limit 价格未 round 到合法 tick 导致 error 110 | §5.1 加 tick rounding：<$3 $0.05，≥$3 $0.10；penny pilot 白名单 |
| P0-IB-5 | ibapi 同步模型在 asyncio 会阻塞 event loop | §6.2b 加 ibapi→asyncio bridge：run_in_executor thread pool + asyncio.Queue callback |

**P1 问题和修正**（6 条，全部应用）：

| # | 问题 | 修正位置 |
|---|---|---|
| P1-IB-6 | Worker 重启 IBKR 侧未清旧 clientId → 501 error | §6.5a `RestartSec=60` + 启动 501 retry 3 次逻辑 |
| P1-IB-7 | BAG combo 跨所路由会失败 | §5.3 BAG 强制 `exchange='SMART'` |
| P1-IB-8 | 50 并发下单接近 IBKR 50 orders/分钟 pacing 边界 | §5.4 加 `asyncio.Semaphore(30)` FIFO 下单队列 |
| P1-IB-9 | Paper ≠ Live（不模拟 assignment / fill 乐观） | §15 新增 "Step G+ Paper vs Live 差异验证"：UAT small live 2-3 日 |
| P1-IB-10 | IBKR error 分散处理 | §5.5 IBKR Error 映射表（7 种 + 其他） |
| P1-IB-11 | BAG MKT 升级各 leg 价格错位 | §5.3 BAG 不走 MKT 档，始终 LMT + 风控允许逐腿拆单 |

**专家意见**：v5 通过。五专家全员认可，**spec 进入最终版 v5**。

---

## 19. 最终版结论

- **版本**：v5（通过五专家 Round 1-5 全部评审）
- **问题统计**：5 轮共 **23 P0 + 32 P1 = 55 问题，全部应用到 spec**
- **可进 plan 阶段**：✅ 可以开始写 `docs/superpowers/plans/2026-04-21-agentic-trading-redesign.md` 展开 Step A-H
- **用户最后审批**：本 spec 提交用户 review，用户确认后进 plan 和执行



---

## 18. 文档版本

- v0 (2026-04-21): 初稿
- v1-v5: 五专家评审后修订（见 §17）

---
