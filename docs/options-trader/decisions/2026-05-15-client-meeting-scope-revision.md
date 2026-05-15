# 2026-05-15 客户开会 — v1 范围 6 项变更 (D1-D6)

> **类型**: 决策记录 (公开 — 客户友好版, 脱敏)
> **触发**: 客户实际使用场景反馈 → 影响北极星远景 + 中间路径 + 状态机流转 + 系统能力层
> **关联**: [北极星 §1 第一版目标](../specs/north-star-v1-target.md) + 内部 discussion (项目 repo private, 不外发)
> **2026-05-16 invariant 24 真意 sync**: D2 entry review loop 真意 = 3 语义分支 (显式窗口 / 隐晦窗口 30 天默认 / 显式立即不进 loop), 详见 D2 关键设计段

---

## 决策清单

| # | 决策 | 类别 |
|---|---|---|
| D1 | 用户表达层条件触发**永久锁死 v1 范围**, 不再"无穷扩展" | scope 收缩 |
| D2 | **Entry phase review loop** — 时间触发 Agent 2 临时拒绝不立即转失败 (3 语义分支, 2026-05-16 真意 sync) | 状态机流转 |
| D3 | **AI 决策时间线 UI** — 持仓详情页加可解释性能力 | 产品能力 (新) |
| D4 | **系统自愈全面化** — 16 类故障矩阵 + chaos test + NZ 凌晨告警触达 | 系统可靠性 (升级) |
| D5 | **BundleV2 dim 11 长线指标 21 项** — Agent 2 输入维度从 10 → 12 | 决策输入扩展 |
| D6 | **BundleV2 dim 12 事件影响 + Agent 3 Event Refresh** — 3 Agent 体系 | 决策输入扩展 + 新 Agent |

---

## D1 — 用户表达层条件触发永久锁死

### 背景

第一版北极星远景列出"触发条件无穷扩展"作为远景子能力, 列举 10+ 种条件类型 (隐含波动率 / 成交量 / 双 MA / 多条件 and-or / 跨标的相对强度 / 期权希腊字母异动 等)。

### 客户反馈

真实交易场景下, 客户**几乎不下纯条件触发单**, 实际意图分布:

- 时间锚定 (e.g. "财报后买 straddle")
- 事件锚定 (e.g. "FOMC 后看跌")
- 立即决策 (e.g. "立即买 1 张 NVDA call")
- 简单条件 (e.g. "突破 280 买" — 已在 v1 范围内 PRICE_BREACH)

"MA5 上穿 MA20 就下单" 这种条件单在真实使用中**基本不出现**。

### 决策

**用户表达层** (客户输入框 + 前端解析层 + Agent 1 schema) 永久锁死 v1 范围:

| 触发类型 | 状态 |
|---|---|
| 立即 (effective_now) | v1 已做 ✓ |
| 时间 (effective_window) | v1 已做 ✓ |
| 条件 PRICE_BREACH 常数 | v1 已做 ✓ |
| 条件 MA_CROSSOVER 单 MA (含上/下穿) | v1 已做 ✓ |
| 时间 + 条件组合 | v1 已做 ✓ |
| 其他条件类型 (IV / 成交量 / 双 MA / and-or / 跨标的 / 希腊字母 等) | **永久不做 (用户层)** |

### 重要保留

**AI 自主流内部 (Discovery Agent)** 仍可使用任意条件 logic, 但**不暴露给客户 UI**:
- 例: Discovery 抓 earnings 公布 + IV crush 信号触发自动减仓
- 例: Discovery 抓 SEC filing 异常触发自动建仓
- 这些是 Discovery Agent 内部条件 logic, 跟客户输入层无关

### 影响

- 北极星 §2 远景子能力 #1 重写
- 北极星 §4 #6 状态: IDEA → **PERMANENTLY-DEFERRED-USER-LAYER**
- 北极星 §7 反偏离加 1 条
- 节省成本: 客户输入框 prompt 设计 / 前端解析展示组件 / Agent 1 schema 扩展

---

## D2 — Entry phase review loop (时间触发不再一次拒绝即失败)

### 背景

当前 v1 状态机流转 (时间触发):

```
机会单 (ACTIVE_MONITORING)
  → 时间窗口到 → Agent 2 入场决策
  → 拒绝 → 失败单 (FAILED)
  → 通过 → 风险门 → 下单 → 持仓单
```

### 客户反馈

**真实市场 5 min 内可完全反转**, Agent 2 一次拒绝就转失败太脆弱:
- 流动性临时差 (e.g. 大单刚走完 spread 还没恢复)
- IV 临时偏高 (e.g. 突发新闻后 IV spike)
- spread 太宽 (e.g. 开盘前 5 min)

下次 review 这些条件可能恢复, 让 Agent 2 临时拒绝就放弃机会是浪费。

客户期望: **窗口内轮询 Agent 2 决策, 窗口过期才转失败**, 类似仓位管理工人的 review loop 模式。

### 决策

引入 **Entry phase review loop** — 时间触发的入场决策也走持续 review 机制:

#### 流程改造

```
机会单 (ACTIVE_MONITORING)
  → 时间窗口到 → Agent 2 入场决策
  → 拒绝 (TEMPORARY) → 留在 ACTIVE_MONITORING + INSERT active_reviews(review_phase=ENTRY)
  → 下次 review interval 到 → Agent 2 再决策
  → ... (循环) ...
  → 拒绝 (PERMANENT) → 立即转失败单
  → 通过 → 风险门 → 下单 → 持仓单
  → 窗口过期 → EXPIRE_OPPORTUNITY → 失败单
```

#### 关键设计

- **rejection_type 字段** (Agent 2 entry decision 输出):
  - `TEMPORARY` — 流动性临时差 / spread 太宽 / IV 临时偏高 / 价格刚触发等下 confirm
  - `PERMANENT` — 用户意图无效 / 策略根本不应该执行 / symbol 退市 / 风险预算耗尽
  - `null` — 决策通过
- **Review 节奏 = 连续 loop** (2026-05-15 round 3 补充澄清, 跟北极星 §1 / wiki §5.2 事件熔断窗口同 pattern, **不是**自适应 interval 公式):
  - 前一次 Agent 2 判断 + 决策实施完成 (HOLD / 加仓 fill / 减仓 fill) → **立即下一轮 review**
  - 周期由 LLM 决策延迟主导 (~15-30s/轮, 可能不到 1 min)
  - 单次 review 全管道 **timeout = 60s for entry phase** (round 4 用户 ack; vs 事件熔断 25s — entry window 30 min - 1 day 不像 ±15 min 紧迫, 给 LLM 充分时间; 真 hang 跳过该轮继续 loop)
- **持仓 review 5 min 默认不动**: 用户提"仓位管理工人已有此设定"是误解现有行为 (现有持仓 review 默认 5 min/次, 仅事件窗口连续 loop). entry phase 复用**事件熔断 pattern**, 持仓 review 默认行为不改 (若期望持仓也改连续 loop = 另开 brainstorm, LLM 调用量级 10x)
- **3 语义分支** (2026-05-16 invariant 24 真意 sync — 用户 Tier 1 纠正):
  - **(a) 显式窗口** ("7 天内买 TSLA" / "本周" / "5/20 之前") → `effective_until = 用户给截止`, **进 loop**
  - **(b) 隐晦表达窗口** ("看情况下单" / "等机会" / "合适时候" / 没明示时间但非立即) → **默认 `effective_until = NOW + 30 天`**, **进 loop**
  - **(c) 显式立即** ("立即" / "马上" / "现在" / "今天就") → **不进 loop**, Agent 2 一次决策即 EXECUTE (下单, 撞 IBKR 端拒走 ORDER_FAILED → FAILED) 或一次拒绝即 FAILED
- **PRICE_BREACH / MA_CROSSOVER 触发后**: 进 ENTRY_REVIEW_PHASE 直至窗口过期, **不依赖再次满足条件** (客户 "突破 280 买" 一旦回落 279.99 不应又得等突破)
- **架构层复用 active_reviews 表** + 加 `review_phase` 列区分 ENTRY/POSITION

#### 影响

- 北极星 §1 加 entry review loop 段
- 北极星 §7 反偏离加 1 条
- 新加 invariant 24 (待 spec 后落)
- 真实场景下: 入场成功率提升 (一次拒绝不再丢机会)

---

## D3 — AI 决策时间线 UI (客户可解释性)

### 背景

当前持仓详情页:
- 显示订单基本信息 + timeline (业务状态变化)
- AI 决策**只能看到最近一次** review summary
- 历史 entry 决策 / 历史加减仓决策 summary **不可见**

### 客户反馈

客户审计需要看到完整的 "AI 为什么这次决定平 / 没平 / 没下", 这是客户信任系统的关键。当前披露太少。

### 决策

持仓详情页加 **"AI 决策时间线" tab** (跟现有 timeline tab 并列):

#### 显示内容 (2026-05-15 10:01 NZ 用户补充简化)

**只列开仓 + 加仓 + 减仓决策** (state-changing only, 用户原话"所以不会很多"):

- ENTRY (成功开仓 + 永久拒绝失败)
- ADD (未来加仓, 待 invariant 10 扩展)
- PARTIAL_CLOSE (部分平仓)
- FULL_CLOSE (全平)

**不显示**:
- HOLD 决策 (噪声) — **不需要 "显示所有" toggle**, 永远不显示
- entry review loop 中每次临时拒绝 — 用户原话"所以不会很多", 这些跟 HOLD 同噪声级别
- ADJUST_STOP — invariant 16 客户期权不止损, 实际不会有此类决策

#### 列表交互 (用户澄清)

- **鼠标划过触发自动滚动**显示卡片
- **点击 popup** 看详情
- 时间倒序 + ENTRY sticky 顶部, color-coded:

| 决策类型 | 颜色 |
|---|---|
| ENTRY (成功) | 紫色 |
| ENTRY 失败 / 永久拒绝 | 红色 |
| ADD (加仓) | 橙色 |
| PARTIAL_CLOSE | 蓝色 |
| FULL_CLOSE | 绿色 |

每张卡片显示: 时间 + 决策类型 + reasoning 一行摘要。

#### 点击 popup 详情 (2026-05-15 10:01 NZ 用户补充简化)

用户原话"其他细节应该不用看":

- **AI 决策 summary** + **决策的思考过程** (≤600 字 reasoning + 关键 thesis 点)
- **不需要**: 输入 10 维 Bundle JSON viewer / AI raw output / risk gate trace

客户看的是**产品视角** (AI 决定了什么 + 为什么), 不是工程师视角 (调试细节)。

#### 跟 timeline tab 区分

- **Timeline tab** (现有) = **业务状态变化** (草稿 / 机会 / 持仓 / 已平仓 + 失败原因) — 客观事件
- **AI 决策时间线** (新加) = **AI 思考过程** (开仓 + 加减仓决策) — 主观判断

两个 tab 并列, 不重叠。

### 影响

- 北极星 §2 远景子能力新加 #6 (客户可解释性)
- 后端: `GET /v1/positions/{id}/decisions` 新 API
- 前端: 持仓 + 失败机会单详情页 tab 改造

---

## D4 — 系统自愈全面化 (system resilience / self-healing)

### 背景

当前自愈能力:
- ✓ IBKR API socket 自动重连 (Pool 自治, wave 11 ship)
- ✓ IB Gateway 进程自动重启 (IBC, 周日维护后实测过)
- ✓ Market data subscription 自动 resubscribe
- ✓ Pre-existing position adoption (process restart)
- ⏳ systemd unit (待 deploy)
- ❌ 其他 10+ 类故障未覆盖

### 客户反馈

> "断线重连一定要做好, 不然用户毫无办法。... 美股都是在新西兰晚上才开盘的, 这个时候除了问题, 基本找不到我修复的。做出来还要充分测试各种场景的短线重连。"

**痛点本质**: 不是"IBKR 断线", 是 "NZ 凌晨美股盘中任何故障 dev 不能 reach"。

### 决策

把 "断线重连" 升级为 **system resilience / self-healing 全面化**, 覆盖 16 类故障:

#### 16 类故障矩阵

| 类别 | 当前状态 | 期望恢复路径 |
|---|---|---|
| IBKR API socket / Gateway 进程 / Network 抖动 | wave 11 ✓ | chaos test 补 |
| IBC daily restart 2FA push timeout | ⏳ 兜底闹钟 | 1-2 周实测 P50/P95 |
| VM 重启 / Worker crash / OOM | ⏳ systemd | restart=on-failure + MemoryHigh 限 + heartbeat |
| DB 连接 lost | 部分 | 显式 retry policy |
| LLM API 限流 / 超时 / 断网 / quota / key 失效 | 部分 | + **LLM fallback chain** env 切换 |
| Market data subscription 丢失 | wave 11 ✓ | chaos test 补 |
| **orderStatus callback 丢失 (silent drift)** | ❌ | + **Reconcile loop** periodic reqExecutions |
| **active_reviews stale** (worker 死时 review 没 fire) | ❌ | + APScheduler misfire handle + 重启扫表 catch-up |
| Disk full / NTP drift | ❌ | log rotation + 监控 |
| MarketDataPool 100 stream 配额 | 部分 | + LRU evict |
| Pre-existing position adoption | 本 session ✓ | chaos test 补 |

#### 三件套硬要求

每类故障必须有:
1. **自动恢复路径** (代码层)
2. **监控告警** (passive 模式, 见下)
3. **chaos test 覆盖** (`tests/chaos/` 真实 kill -9 / network drop / docker stop)

#### 告警 3 类邮件 (2026-05-15 round 3+4 用户补充, 不轰炸)

用户 round 3 原话: "不是信息轰炸客户不停的提示他, 而是 banner 显示出来就行了"
用户 round 4 原话: "有些失败是不用发邮件提醒的 ... 不要同样的问题一直推送 ... 能自愈的 bug 就不要推送了 ... 每天结收盘后给一份 summary 给 IT 人员, 主要关于运行状态的 summary"

| 邮件类型 | 触发 | 频率 | 内容 |
|---|---|---|---|
| **A. 自愈失败 event-driven** | 系统尝试自愈**失败** (LLM fallback chain 全失败 / Pool 重连 30 min 仍失败 / orderStatus reconcile 失败 / IBC 2FA 5 次推送都没接 等) | 实时, **60 min 去重窗口** | 故障类型 + 时间 + 自愈尝试记录 + 当前系统状态 + 建议人工介入步骤 |
| **B. 跳过 (已有 native channel 不重复)** | #4 **IBC daily restart 2FA push timeout** — IBKR app push 已通知 | **不发邮件** (`skip_email_on_failure=true`) | — |
| **C. Daily summary** | 美股**收盘后 16:30 ET** (= NZ 09:30 / 08:30 depending DST, 用户起床看) | 每交易日 1 次 | 6 类: (1) 业务 (持仓+开仓/平仓+失败机会单分布) (2) 故障+自愈记录 (即使成功也列) (3) LLM 调用+cost (4) Pool/Connection (uptime+重连+恢复时间) (5) KPI (心跳/orderStatus 丢失次数/NTP/Disk/Memory) (6) 明日前瞻 (earnings/FOMC/CPI 标的) |

**能自愈成功的不实时推送** (Pool 重连成功 / LLM fallback 切到备用成功 / IBC push 通过), 仍在 daily summary 里列。**16 类故障矩阵每类加 `skip_email_on_failure: bool` flag** 默认 false 仅个别故障 (e.g. #4 IBC) 标 true。

**客户端**: 只 banner 显示 (passive — 客户打开 dashboard 才看到). **不做**客户端推送 / Slack 客户群 / 邮件告警客户。

**自动 fallback** (LLM chain 切换 / Pool 重连 / reconcile loop) 仍执行不依赖人工 — 邮件只是知会。NZ 凌晨故障 = 系统自动 fallback; IT 人员早上看 daily summary 知道发生了什么 (失败需介入则前一晚已收 A 类实时邮件); 客户打开 dashboard 看到 banner — **三者都不被打扰**。

#### 新加机制

- **Reconcile loop** (修 silent drift): 每 N min 主动 reqPositions + reqExecutions vs DB 对比 → 自动修复 (orderStatus 丢失典型场景)
- **LLM fallback chain**: 主 → 备 1 → 备 2, env 切换代码不动
- **月度 chaos drill**: cron 自动诱导一次故障 + 测量恢复时间 + 写 health 报告

#### 关联 invariant 升级 (待 spec 后落)

- 新加 invariant 25: "系统自愈 NZ 凌晨硬要求 — 16 类故障三件套覆盖。任何'要人工介入'路径前必须先穷尽自动化"
- 升级 invariant 18: 加 oncall 触达子句

---

---

## D5 — BundleV2 dim 11 长线指标 21 项 (2026-05-15 round 4 加)

### 背景

10 维 Bundle 没有"长线 perspective" 维度 (现有 dim 3 仅 1m_1D + 5m_5D + 1d_20D 短中线), Agent 2 决策时缺过去 2 年的标的特征 (高低点 / 趋势 / 波动率 / 动量 / 回撤等)。

### 客户反馈

"10 维要扩充, 加历史数据 — 过去 2 年标的每天的价格以及每天的成交量数据 ... 把 2 年数据发给 AI 有些笨, 不如根据这些数据算出**很多有用的指标**再传给 AI 做决策 ... 既要数据要全面 ... 至少这两年的最高点最低点等等信息都是需要的 ... 这些信息是需要 query 到 2 年的日线自己算出来的, 还是直接 query 到的?"

### 决策

**BundleV1 (10 维) → BundleV2 (12 维)**, 新加 **dim 11 = 长线指标 21 项** (按指标族算, 美股期权实战专家审核):

| 类 | family | 内容 |
|---|---|---|
| **A. 长期价格 levels** | 3 | **5 时间段** (2y/1y/6m/3m/1m) × (high/low + 日期 + 成交量) + 长期 (2y) mean/median/stdev + 当前 vs (high/low/median/mean) of 各时段 % + 当前 2y 分位 |
| **B. MA trend** | 3 | MA20/50/100/200 当前值 + MA 多空排列分类 + 距 MA200 % + MA50 vs MA200 % + **MA200 斜率** (上行/横盘/下行) |
| **C. 波动率** | 4 | RV 30/90/252 年化 + ATR(14) + ATR% + RV 30d 分位 + **HV/IV ratio** (跟 dim 7 联合, 期权"贵不贵"实战核心) |
| **D. Volume** | 3 | DAV 2y + 30d + 30d/2y 比 + 当日 vol 分位 |
| **E. Momentum** | 4 | 1m/3m/6m/12m 涨跌 + RSI(14) + MACD(12,26,9) 状态 + 连续涨跌天数 |
| **F. 回撤 + 相对 SPY** | 4 | MDD 2y + 持续天数 + 恢复天数 + 当前回撤 + outperform SPY 1m/3m/12m + β 2y |

**删除 4 项** (实战不真用): A4 支撑/阻力位 (简单聚类噪声) / D4 OBV (期权不用) / E5 30d win rate (相关性弱) / F5 α (回归模型敏感)。

**加 1 项**: C4 HV/IV ratio (期权实战核心)。

### 数据来源 (客户问"自己算 vs 直接 query")

**IBKR API 只直接给原始 500 根日 OHLCV bar**, 所有指标都要**自己算**:

| 指标族 | 来源 |
|---|---|
| raw 日线 OHLCV (~500 根) | IBKR `QueryClient.reqHistoricalData(durationStr='2 Y', barSizeSetting='1 day', whatToShow='TRADES')` 直接 query |
| High / Low / Median / Mean / Stdev / Percentile | 自己算 (`numpy.max/min/median/mean/std/percentile`) |
| MA20/50/100/200 + 斜率 | 自己算 (`pandas.rolling(N).mean()`) |
| RV / HV / ATR | 自己算 (`pct_change().std()` / `pandas_ta.atr()`) |
| RSI / MACD | 自己算 (`pandas_ta.rsi()` / `pandas_ta.macd()`) |
| 回撤 MDD | 自己算 (`cummax`-based) |
| β / outperform SPY | 自己算 (额外 query SPY 500 根日 bar + `scipy.stats.linregress`) |
| HV/IV ratio | HV in dim 11 算 + IV from dim 7 现成 + ratio = HV / IV |

**实现**:
- 2 次 IBKR query per opp: 标的 + SPY 各 500 根日 bar
- 1 次 Python 计算: pandas + numpy + pandas_ta + scipy, 21 指标 ~10-50 ms
- **新依赖**: `pandas_ta` (纯 Python 替代 ta-lib C 库) + `scipy.stats.linregress`

**计算时机**:
- 入场前一次性算 (Agent 2 第一次拉 bundle 时)
- 每日 SYSTEM_WAKE_UP 时重新 query + 重算 (50 ms 不是瓶颈)

### 远景 (依赖 events 数据)

Earnings 历史反应 (过去 4-8 季度财报后 1/3/5 天涨跌平均) — 跟 §4 #3 EventCalendarCollector 一起做。

---

## D6 — BundleV2 dim 12 事件影响 + 新加 Agent 3 (2026-05-15 round 4 加)

### 背景

现有 10 维 Bundle **没有事件/新闻维度** (Agent 2 决策时不考虑世界上发生的事情)。客户期望系统能自动考虑新闻 + 热点 + 财报 / 政策 / 行业事件等。

### 客户反馈

"system prompt 里面要提醒一句: 判断还要包括最近的新闻和热点的影响 ... 我想要 AI 能自动考虑世界上发生的事情, 因为一个炒期权的人也会关注这个因素, 而且它是重要因素 ... 但是我有个问题, 这回造成 AI 每次都去搜索吗? 还是说我应该在 Agent 1 的时候就让 AI 单独思考这个新闻热点因素然后记录下来, 以后只是把它放在 10 个维度中的事件维度就好了 (现在本来也是空的) ..."

Round 3 修订: "得 Agent 3 专门负责 AI 刷事件 ... 你怎么知道事件什么时候公布? ... 入场前必拉最新一份 ... 盘前 30 min 为什么要刷? 这时候还没开盘 ... 收盘后就不刷了"

### 决策

**dim 12 = 事件影响** (per-symbol + macro 子字段):
- **per-symbol scope**: (机会单 ACTIVE_MONITORING) ∪ (持仓单 HOLDING) symbol 合集去重
- **macro 子字段**: FOMC / CPI / 政策等影响所有标的的宏观事件
- **schema**: `{as_of, next_refresh_due, summary_zh (≤500 字), key_events: [...], upcoming_events: [...], macro_events_summary}`

**新加 Agent 3 (事件刷新 agent)** — Agent 体系扩为 3:

| Agent | 职责 | LLM 调用时机 |
|---|---|---|
| **Agent 1 (Intake)** | 中文意图 → 声明式 JSON | 客户提交时 1 次 |
| **Agent 2 (Strategy)** | 入场决策 + 仓位管理 review | 持续 (entry review loop + 持仓 5 min / 事件熔断 连续 loop) |
| **Agent 3 (Event Refresh)** ← 新加 | 定期 web search 摘要写 dim 12 per-symbol + macro | 每 15 min + 开盘瞬间 + 入场前阻塞触发 |

**实现层** (跟北极星 §1 范式一致): 新加 1 handler (`agent3_event_refresh_handler`) + 1 event_type (`AGENT3_EVENT_REFRESH_TICK(symbol)`); 北极星 §1 **8 → 9 handler / 8 → 9 event_type**。Agent 3 跟 Agent 1/2 完全分离, search 慢/失败不影响主决策。

### 刷新触发时机 (客户提问回答)

| 时段 | Agent 3 行为 | 理由 |
|---|---|---|
| 盘前 (09:00-09:30 ET) | **不刷** | 用户判断: 不交易不需事件数据 |
| **开盘瞬间 09:30 ET** | **立即刷一次** | 盘外 16 hr 缺口需补, 开盘后第一时间给 09:30-09:45 间可能的 entry 决策用 |
| 盘中 (09:45-15:45 ET) | **每 15 min** | 比 5 min 节省 API quota, 比 30 min 反应快 |
| **入场决策前** | **阻塞触发刷** (鲜度门槛 5 min, >5 min 旧则刷, ≤5 min 用现成) | 用户原话: "所谓的入场就是 Agent 2 决策的, 所以当然要事件数据, 要触发刷" |
| 收盘 16:00 ET | **停止刷** | 用户判断: 收盘后就不刷了 |
| 盘外 (16:00 ET - 次日 09:30 ET) | 不刷 | — |

**"事件公布瞬间立即刷"** — 用户问"你怎么知道事件什么时候公布?". 实事求是回答:
- FOMC / CPI 时间精确公开 (跟 §1 事件熔断窗口同源) — 预先 cron 调度可知
- Earnings 日期公开, 但 pre/post-market + 精确分钟不一定
- 一般市场新闻 / 政策意外 / 黑天鹅 — **无法预知**, 需事后 newsTicker push 接收
- 本期 ship **不实现** (没真 newsTicker push 数据源, 留远景 §4 #2 NewsCollector ship 后)

### 本期数据源 vs 远景

| 阶段 | 数据源 |
|---|---|
| **本期 ship** | **Anthropic SDK web search tool** in dedicated worker (1 LLM 调用做"搜近 24 hr {symbol} 相关新闻 + 摘要 ≤500 字") |
| **远景** (§4 #2 NewsCollector ship 后) | 真 IBKR Reuters/DJN + 第三方 (Polygon/Finnhub) + per-source 限流 + 跨源去重 → 真新闻流入 dim 12 + newsTicker push 触发紧急刷新 |

### Agent 1/2 prompt 不改 (用户判断对)

跟 invariant 5 v3 (整段传 Agent 2 自决) + 不强制止损 (LLM 看数据综合判断) **同哲学** — 信任 LLM 看 dim 12 数据自然综合"事件影响"判断, **不需** explicit prompt 提醒"考虑新闻"。加 prompt 教指令反而违反"信任 LLM" (跟 4 腿流动性 hard→soft 反转同理)。

Agent 2 system prompt 已经说"看 10 维 bundle 综合决策", dim 12 加上后 Agent 2 自然读, **不需要改 prompt**。

---

## 后续工作 (留下轮)

各项决策需独立 spec brainstorm + 五专家评审:

- D2: `2026-05-15-entry-review-loop-design.md`
- D3: `2026-05-15-decision-disclosure-ui-design.md`
- D4: `2026-05-15-system-resilience-self-healing-design.md`
- **D5: `2026-05-15-long-term-stats-dim11-design.md`** (新加, 21 指标 + IBKR query + pandas_ta 实现)
- **D6: `2026-05-15-agent3-event-impact-dim12-design.md`** (新加, Agent 3 + handler/event_type + 刷新触发 + web search worker)
- CLAUDE.md invariant 24 / 25 / 18 + Bundle 维度数描述 更新待 spec ship 后

---

## 五专家评审

基础 3 (软件工程 / CC AI 规则系统 / 云部署) + 项目专属 2 (美股期权实战 / IB-TWS) = 全员 PASS, 备注:
- D2 review interval 公式待 spec 阶段实测调整
- D4 监控 stack 选型待 spec 阶段决 (self-hosted vs hosted)
- D4 orderStatus callback 丢失场景 chaos test 必须覆盖 (IB-TWS 专家 P0 fix)

---

## 5 问 cross-check (北极星 §3)

**0 偏离信号**, 全 PASS。4 项变更跟客户主动流 vs AI 自主流 schema 共用一致, forward compat hook 不破坏, 不引入新必填字段, 不前置 AI 决策到 intake。

---

**Decision archived 2026-05-15. 同 commit 改动**: 北极星 memory + wiki 北极星 + 项目 discussion 归档。
