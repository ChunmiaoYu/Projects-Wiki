# 2026-05-15 客户开会 — v1 范围 4 项变更

> **类型**: 决策记录 (公开 — 客户友好版, 脱敏)
> **触发**: 客户实际使用场景反馈 → 影响北极星远景 + 中间路径 + 状态机流转 + 系统能力层
> **关联**: [北极星 §1 第一版目标](../specs/north-star-v1-target.md) + 内部 discussion (项目 repo private, 不外发)

---

## 决策清单

| # | 决策 | 类别 |
|---|---|---|
| D1 | 用户表达层条件触发**永久锁死 v1 范围**, 不再"无穷扩展" | scope 收缩 |
| D2 | **Entry phase review loop** — 时间触发 Agent 2 临时拒绝不立即转失败 | 状态机流转 |
| D3 | **AI 决策时间线 UI** — 持仓详情页加可解释性能力 | 产品能力 (新) |
| D4 | **系统自愈全面化** — 16 类故障矩阵 + chaos test + NZ 凌晨告警触达 | 系统可靠性 (升级) |

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
- **Review 节奏 = 连续 loop** (2026-05-15 10:01 NZ 用户补充澄清, 跟北极星 §1 / wiki §5.2 事件熔断窗口同 pattern, **不是**自适应 interval 公式):
  - 前一次 Agent 2 判断 + 决策实施完成 (HOLD / 加仓 fill / 减仓 fill) → **立即下一轮 review**
  - 周期由 LLM 决策延迟主导 (~15-30s/轮, 可能不到 1 min)
  - 单次 review 全管道 timeout = 25s (跟事件熔断同) 超时跳过该轮继续 loop
- **持仓 review 5 min 默认不动**: 用户提"仓位管理工人已有此设定"是误解现有行为 (现有持仓 review 默认 5 min/次, 仅事件窗口连续 loop). entry phase 复用**事件熔断 pattern**, 持仓 review 默认行为不改 (若期望持仓也改连续 loop = 另开 brainstorm, LLM 调用量级 10x)
- **立即触发 (effective_now)** 默认 effective_until = +30 min, 也走此机制
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

#### 告警 passive 模式 (2026-05-15 10:01 NZ 用户补充澄清, 不轰炸)

用户原话: "不是信息轰炸客户不停的提示他, 而是 banner 显示出来就行了。给我 (IT 人员) 发一封邮件说明发生了什么事情, 做提醒就行。"

- **客户端**: 只 banner 显示 (passive — 客户打开 dashboard 才看到). **不做**客户端推送 / Slack 客户群 / 邮件告警客户 (避免轰炸客户)
- **IT 人员 (项目负责人)**: **summary 邮件**告知发生了什么事情 + 提醒 (不是实时每秒告警). **不做** Slack 实时推送 / SMS / 电话
- **自动 fallback** (LLM chain 切换 / Pool 重连 / reconcile loop) 仍执行不依赖人工 — 告警只是知会

这样 NZ 凌晨故障 = 系统自动 fallback / reconcile / Pool 重连; IT 人员早上看邮件 summary 知道发生了什么; 客户打开 dashboard 看到 banner 知道系统状态 — **三者都不被打扰**。

#### 新加机制

- **Reconcile loop** (修 silent drift): 每 N min 主动 reqPositions + reqExecutions vs DB 对比 → 自动修复 (orderStatus 丢失典型场景)
- **LLM fallback chain**: 主 → 备 1 → 备 2, env 切换代码不动
- **月度 chaos drill**: cron 自动诱导一次故障 + 测量恢复时间 + 写 health 报告

#### 关联 invariant 升级 (待 spec 后落)

- 新加 invariant 25: "系统自愈 NZ 凌晨硬要求 — 16 类故障三件套覆盖。任何'要人工介入'路径前必须先穷尽自动化"
- 升级 invariant 18: 加 oncall 触达子句

---

## 后续工作 (留下轮)

各项决策需独立 spec brainstorm + 五专家评审:

- D2: `2026-05-15-entry-review-loop-design.md`
- D3: `2026-05-15-decision-disclosure-ui-design.md`
- D4: `2026-05-15-system-resilience-self-healing-design.md`
- CLAUDE.md invariant 24 / 25 / 18 更新待 spec ship 后

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
