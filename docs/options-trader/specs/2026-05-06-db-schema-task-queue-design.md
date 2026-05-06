---
title: DB Schema 重构 — 任务队列驱动新架构 Design Spec
date: 2026-05-06
status: design-approved-pending-plan
related_north_star_path: §4 #1.5 (2026-05-06 加; 阻塞 §4 #6 触发条件扩展 / #8 Agent 2 升级 / #11 多账户多 LLM)
related_north_star_section: §1 系统构成范式升级 (4 Pool/Client + 8 event_type + 任务队列驱动)
related_architecture_walkthrough: docs-hub/options-trader/specs/architecture-walkthrough.md §9 数据库架构
related_invariants_referenced:
  - 11 (主队列状态机, V2 已 ship 0023)
  - 13 (修改 = 取消原单 + 创建复制单, 跟 split_order_batch_id 区分)
  - 21 (限价单 mid-based 4 档 spread_ratio, 跟 algo_strategy 协同)
related_findings_to_create:
  - F-2026-05-06-DB-SCHEMA-MIGRATION-PENDING (P0, 6 alembic migrations 待 ship)
  - F-2026-05-06-STRATEGY-WHITELIST-SEED-DATA (P0, 12 策略 seed 跟北极星 §1 + wiki §9 同步, hard test 已 design)
  - F-2026-05-06-SYSTEM-SLEEP-TRIGGER-CONDITION (P1, SE 二轮提, SYSTEM_SLEEP 触发条件 spec 未定义)
  - F-2026-05-06-CONID-FALLBACK (P1, IB 二轮提, reqContractDetails 返空时 ContractNotFoundError 处理)
  - F-2026-05-06-NEWSTICKER-REQID-ISOLATION (P1, IB 二轮提, newsTicker 用独立 reqId 池防 cancelMktData 误杀)
  - F-2026-05-06-OPEX-EVENT-CALENDAR (P2, 期权实战提, 加 OPEX 月期权到期日 event_type pre=30/post=0)
  - F-2026-05-06-DEPLOY-HEALTH-GATE (P2, Cloud 提, deploy.sh 加 alembic upgrade 失败时 worker 不自动起)
  - F-2026-05-06-ALEMBIC-NOT-VALID-CHECK (P2, Cloud 提, 0029 CHECK 用 NOT VALID + 后台 VALIDATE 防全表扫)
five_expert_review:
  software_engineer: pass
  cc_ai_rules: pass-after-amendments
  oci_deployment: pass
  options_practitioner: pass
  ib_tws: pass
---

# DB Schema 重构 — 任务队列驱动新架构

## 1. 背景

2026-05-06 范式级架构梳理 (memory `project_vision_and_north_star.md` §1) 把"3 工人 actor" 视角升级为 **APScheduler + 8 handler + 4 Pool/Client + 任务队列驱动**。该升级要求 DB schema 配套补充 5 张新表 + 1 张表加列 + 应用层扩展 task_type 枚举。

**已 ship 的现有架构** (本 spec **不动**):
- `WorkflowTask` (workflow_tasks 表) — 已是完整任务队列实现 (含 task_type / task_status / payload jsonb / idempotency_key / locked_at / lock_token / attempts / max_attempts / not_before_at)
- `Agent2Decision` (agent2_decisions 表) — 已含 bundle_json / parsed_decision_json / model_name / cost / risk_gate_result / executed_at_utc / thesis_summary_zh
- `LLMCall` — Agent 1 LLM trace
- `Opportunity.origin_type / origin_metadata / effective_until` — 已 ship 在 alembic 0017
- `BrokerOrder` / `Fill` / `Position` / `BrokerOrderEvent` / `AccountSnapshot` / `WorkerHeartbeat` / `AuditEvent` / `MonitorConfig` / `BrokerCapabilitySnapshot` / `PnlSnapshot` / `OptionChainSnapshot` 等 — 全已 ship

**真正需要做的 (本 spec scope)**:
1. 新建 5 张表 (active_reviews / market_data_subscriptions / pool_health_log / event_calendar / strategy_whitelist)
2. BrokerOrder 加 2 列 (algo_strategy / split_order_batch_id) — 为 §4 #1.6 分批下单依赖
3. WorkflowTask `task_type` 应用层扩展到 8 种 event_type (schema 不变, Pydantic enum 锁)

## 2. 北极星 §13b 决策门禁 5 问 cross-check

| # | 问题 | 答 | 偏离? |
|---|---|---|---|
| 1 | 用户提交流 / AI 自主流共用 schema? | 是, `Opportunity.origin_type` 区分来源, 5 张新表跟来源无关 (基础设施层) | ✓ 不偏离 |
| 2 | 加的新概念 Discovery Agent ship 时是否要重写? | 不, 5 张表都是 Pool/任务队列基础设施, Discovery 来源由 origin_type 区分 | ✓ 不偏离 |
| 3 | Forward compat hook 留齐? | 是, 见 §5 全表; 本 spec 落地北极星 §5 中"待嵌入"的 6 个 hook (引用计数持久化 / 任务必持久化 / 策略白名单查表 / 风险门 mapping 查表 / 5 min review 持久化 / Pool 状态可观测) | ✓ 不偏离 |
| 4 | 是否硬编码"用户必填"? | 否, 不增 Opportunity 字段, 不动 intake | ✓ 不偏离 |
| 5 | 是否前置 AI 自动决策为"用户必给"? | 否, 任务队列 / Pool 状态 / 策略 catalog 都是基础设施 | ✓ 不偏离 |

**5 问全过, 此设计与北极星目标关系**: 直接落地北极星 §4 #1.5 + §5 中 6 个待嵌入 hook, 不偏离任何子能力, 是 §4 #6 / #8 / #11 远景演进的**共用基础设施层前置**。

## 3. 系统构成 (新表跟现有架构关系)

```mermaid
graph TB
    subgraph "已有架构 (不动)"
        OPP["Opportunity<br/>(origin_type/effective_until 已 ship)"]
        WT["WorkflowTask<br/>(任务队列已完整)"]
        A2D["Agent2Decision<br/>(bundle 审计已完整)"]
        BO["BrokerOrder"]
        POS["Position"]
    end

    subgraph "新加 5 张表"
        AR["active_reviews<br/><i>5 min review 持久化排程</i>"]
        MDS["market_data_subscriptions<br/><i>MarketDataPool ref_count 持久化</i>"]
        PHL["pool_health_log<br/><i>4 Pool 状态切换历史</i>"]
        EC["event_calendar<br/><i>事件熔断窗口数据源</i>"]
        SW["strategy_whitelist<br/><i>12 策略 + direction mapping 查表</i>"]
    end

    subgraph "改 1 张表"
        BO_NEW["BrokerOrder + algo_strategy + split_order_batch_id<br/><i>(为 §1.6 分批下单)</i>"]
    end

    OPP -.持仓后→ INSERT.-> AR
    OPP -.监控/持仓→ subscribe.-> MDS
    APS["⏰ APScheduler"] -->|每 ~10s 扫 due| AR
    APS -->|塞任务| WT
    APS -->|查事件窗口| EC
    EC -->|事件 ±15min| AR
    SW -.risk_gate Layer 1+2.-> A2D
    POOL["📡 4 Pool/Client"] -->|状态切换| PHL
    POOL -->|启动遍历| MDS
    BO_NEW -.分批下单| WT

    classDef new fill:#e8f5e9
    classDef changed fill:#fff3e0
    classDef existing fill:#e1f5fe
    class AR,MDS,PHL,EC,SW new
    class BO_NEW changed
    class OPP,WT,A2D,BO,POS existing
```

## 4. 5 个核心设计决策

### D1. `active_reviews` — 5 min review 持续化排程

**问题**: 北极星 §1 锁"5 min/次 review (事件窗口连续 loop)", 当前 review 排程在 in-memory dict 还是已有 DB 表? 重启会丢吗?

**决策**: 新建 `active_reviews` 表持久化, 一行 = 一个持仓 opp 的下次 review 时刻。

**字段**:
- `id` UUID PK
- `opportunity_id` UUID FK → opportunities.id **ON DELETE CASCADE** (UNIQUE — 一个 opp 只有一行)
- `next_review_due_at` timestamptz (btree index, 见下)
- `review_interval_sec` int (default 300; **语义 = handler 完成后下次 review 至少等多少秒, 不是固定间隔**; 见下)
- `last_review_at` timestamptz nullable
- `claimed_by` varchar(64) nullable — APScheduler 实例 ID, 防多实例双触发
- `claimed_at` timestamptz nullable — 取走时戳; 超 60s 未完成视为崩溃, 释放重做
- `created_at` timestamptz
- `updated_at` timestamptz

**`review_interval_sec` 精确语义** (Q4 澄清):
| 模式 | 字段值 | handler 完成 → next_due | 实际间隔 |
|---|---|---|---|
| 正常时段 | **300** | NOW() + 300s | **5 min/次** (handler 远小于 300s) |
| 事件熔断窗口 | **1** (1 秒最小 floor 防 LLM API rate limit + APScheduler tick 不重叠) | NOW() + 1s | **~15-30s/次** (LLM 决策延迟 ~15-30s 主导节奏, 不是固定 1s) — 即"前次完成→立即下一次"的连续 loop 实现 |

**生命周期**:
- ENTRY_FILLED handler INSERT (next_due = NOW() + 300)
- EXIT_FILLED 全平 handler DELETE
- 事件熔断 trigger handler UPDATE review_interval_sec 300 → 1
- 事件熔断窗口结束 trigger handler UPDATE 回 300

**索引** (partial 防全表扫):
- `CREATE INDEX active_reviews_due_idx ON active_reviews (next_review_due_at) WHERE claimed_at IS NULL`
- 持仓单只占 opp 5-10%, partial index 更快

**Concurrency 模型** (防多 APScheduler 实例双触发):
- APScheduler 扫表用 `SELECT ... FROM active_reviews WHERE next_review_due_at <= NOW() AND (claimed_at IS NULL OR claimed_at < NOW() - INTERVAL '60 seconds') FOR UPDATE SKIP LOCKED`
- 取到的行立即 `UPDATE active_reviews SET claimed_by = X, claimed_at = NOW()`
- handler 完成时 `UPDATE active_reviews SET next_review_due_at = NOW() + review_interval_sec, claimed_by = NULL, claimed_at = NULL`
- 60s 超时机制: 如果 handler crash, 60s 后 claimed_at 过期被另一实例 claim 重做 (handler 必 idempotent)

**幂等保证**: `(opportunity_id) UNIQUE` 防重复 INSERT; row-level lock 防双 dispatch

### D2. `market_data_subscriptions` — MarketDataPool ref_count 持久化

**问题**: MarketDataPool 引用计数 (北极星 §1) 必持久化, 否则 Pool 重启 (cloud-dev 部署 / 周日维护后) 业务订阅意图丢失。

**决策**: 新建 `market_data_subscriptions` 表, 一行 = 一个 (con_id, subscriber_id) 唯一组合; ref_count = `COUNT(*)` WHERE unsubscribed_at IS NULL GROUP BY con_id。

**字段** (用 IBKR ConId 作 contract identifier, **不用自定义 string** — 避免漏字段订阅错合约):
- `id` UUID PK
- `con_id` BIGINT NOT NULL — **IBKR 全局唯一 ConId** (reqContractDetails 解析后拿, 同 strike+expiry 不同 multiplier/exchange 全区分)
- `symbol` varchar(32) (反范式, 方便人读 + symbol 级聚合查询)
- `contract_descriptor` JSONB — 审计用 (含 secType/strike/expiry/right/exchange/currency 等, 人读)
- `subscriber_id` varchar(64) — opportunity_id 或 'DASHBOARD' / 'AGENT1_INTAKE' 等
- `subscriber_kind` varchar(32) — **OPP_MONITOR / OPP_HOLDING / CONDITION_TRIGGER / AGENT1_INTAKE / DASHBOARD / BUNDLE_REVIEW** (CONDITION_TRIGGER 是条件工人 callback 算条件用; 同 (con_id, opp_id) 可能 OPP_MONITOR + CONDITION_TRIGGER 两 kind 都订阅, ref_count 计 2 是对的)
- `first_subscribed_at` timestamptz
- `last_active_at` timestamptz
- `unsubscribed_at` timestamptz nullable (软删, 留审计)

**Pool 启动 / DEGRADED → READY 重连恢复 (SYSTEM_WAKE_UP handler 或 Pool 内部 connectionRestored callback)**:
1. `SELECT DISTINCT con_id, contract_descriptor FROM market_data_subscriptions WHERE unsubscribed_at IS NULL`
2. 对每个 con_id, **重发 reqMktData 用新 reqId** (IBKR socket 重连后旧 reqId 失效, 必须新 reqId)
3. Pool 内存维护 `con_id → 当前活跃 reqId` 映射 (运行时, 不持久化, reqId 是 connection-scoped)

**业务调用**:
- `subscribe(con_id, contract_descriptor, subscriber_id, kind)` → `INSERT ... ON CONFLICT (con_id, subscriber_id) WHERE unsubscribed_at IS NULL DO NOTHING` (race-safe)
- `unsubscribe(con_id, subscriber_id)` → `UPDATE SET unsubscribed_at = NOW() WHERE con_id = X AND subscriber_id = Y AND unsubscribed_at IS NULL` (软删)
- `get_ref_count(con_id)` → `SELECT COUNT(*) WHERE con_id = X AND unsubscribed_at IS NULL`

**ref_count → 0 时是否真调 cancelMktData** (北极星 §1 锁的行为):
- **不立刻调**, 保留 IBKR 端订阅 (反正 IBKR 端订阅免费 + 可能马上又有新 subscriber)
- **真 cancel 时机** = SYSTEM_SLEEP handler 时遍历 ref_count=0 的 con_id 一次性 cancelMktData; 或者 IB Gateway 自动断 (周日维护 / daily 05:30 ET 重启) 自然清理
- 防 5 min review 来回订阅打架浪费 100 stream 配额

**索引**:
- partial UNIQUE: `CREATE UNIQUE INDEX mds_active_uniq ON market_data_subscriptions (con_id, subscriber_id) WHERE unsubscribed_at IS NULL` — 防活跃订阅重复
- btree on (con_id) — get_ref_count + Pool 启动扫表
- btree on (last_active_at) — 清理 stale (90 天 hard delete unsubscribed_at IS NOT NULL 的行)

**为什么软删不硬删**: 审计 — 1 周内查"AAPL 这个 stream 历史上谁订阅过, 何时退订" 用得上; 90 天后定期 cleanup task hard delete 防 PAYG 存储膨胀。

### D3. `pool_health_log` — 4 Pool 连接状态切换历史

**问题**: Pool 自治重连屏蔽业务层 (北极星 §1), 但用户/SRE 需要"事后审计 Pool 何时 degrade 多久"。

**决策**: 新建 `pool_health_log` 表, 仅记录**状态切换** (不记心跳, 防爆表), 一行一切换。**仅 4 个常驻 Pool (client_id 10-40) 写**, ad-hoc 90+ (e.g. 2FA probe) 不归此表。

**字段**:
- `id` UUID PK
- `pool_name` varchar(32) — MarketDataPool / QueryClient / OrderClient / AccountSnapshotClient
- `from_state` varchar(16) — DISCONNECTED / DEGRADED / READY (NULL = 初始化)
- `to_state` varchar(16) — 同
- `changed_at` timestamptz (btree index)
- `error_msg` text nullable — DISCONNECTED/DEGRADED 时填 IBKR errorString 原文
- `ibkr_error_code` int nullable — IBKR 标准化 error code (100/162/354/1100/2104 等), 方便聚合统计 ("最近 24h 出过几次 100 限流")
- `ibkr_req_id` int nullable — 关联 IBKR error 的 reqId, 调试用
- `active_stream_count` int nullable — MarketDataPool 状态切换时填当前活跃 stream 数 (Dashboard 看 "73/100" 接近 100 上限提前告警)
- `recovery_seconds` int nullable — DEGRADED → READY 时填本次 outage 总秒数

**索引**:
- btree on (changed_at)
- composite (pool_name, changed_at) — 单 Pool 历史查询
- partial btree on (ibkr_error_code) WHERE ibkr_error_code IS NOT NULL — 错误码聚合统计

**Retention** (防 PAYG 存储膨胀): 估算一年 ~3000 行 (4 Pool × 平均 2 切换/天 × 365 天), 单表无问题, 不必 partition. 远期 1+ 年后可加 90 天 hard delete cleanup task。

**用途**: 用户 NZ 早起 `SELECT * FROM pool_health_log WHERE pool_name='MarketDataPool' AND changed_at > NOW() - INTERVAL '24 hours' ORDER BY changed_at DESC` 看夜里 Pool 有没有断, 多久恢复 (跟 oatworker 2FA probe 互补)。

### D4. `event_calendar` — 事件熔断窗口数据源

**问题**: 北极星 §1 锁"事件 ±15 min 窗口连续 loop", 但事件数据从哪来? 当前没数据源。Agent 2 是否感知事件?

**决策**:
1. 新建 `event_calendar` 表, 一行 = 一个已知事件
2. 本期 source 全 MANUAL (用户/Claude 手录), 远景 §4 #3 EventCalendarCollector ship 时加 AUTO_THIRD_PARTY (Polygon / Finnhub / Yahoo Finance API; **IBKR 不直接提供** earnings calendar)
3. **Agent 2 本期不感知事件** (Q2 a 用户 ack) — 系统层在 active_reviews 上加快 review 频率, Agent 2 只是被更频繁喂 bundle 反应当前数据。**dim 11 "事件维度" 入 bundle 是远景 §4 #8 Agent 2 升级 scope, 本 spec 不动**

**字段**:
- `id` UUID PK
- `symbol` varchar(32) nullable — NULL 表示宏观事件 (CPI / FOMC 不绑定 symbol, 全市场触发)
- `event_type` varchar(32) — EARNINGS / CPI / FOMC / DIVIDEND / SPLIT / NEWS_BREAKING / OTHER
- `event_at` timestamptz (btree index)
- `pre_window_min` int — 事件前进入熔断窗口的分钟数 (**按 event_type 不同 default 见下表**)
- `post_window_min` int — 事件后保持熔断窗口的分钟数 (**按 event_type 不同 default**)
- `source` varchar(32) — MANUAL / AUTO_IBKR_NEWS / AUTO_THIRD_PARTY / AUTO_DISCOVERY
- `description_zh` text nullable
- `created_at` timestamptz
- `created_by` varchar(64) — 'manual:user@email' / 'collector:news_module' / 'newsticker:reqId=42' 等

**Per-event_type pre/post default** (期权专家给的实战值, 应用层 lookup dict 实施, schema 不存; INSERT 时按 event_type 查 default 填):
| event_type | pre_window_min | post_window_min | 理由 |
|---|---|---|---|
| EARNINGS | 15 | **60** | IV crush 5min 完成, 但 analyst conference call 30-60min 后可能二次反应 (公司答 Q&A 给业绩 guidance) |
| FOMC | **30** | **120** | 14:00 ET 公布声明 → 14:30 Powell 讲话 (即兴回答记者问, 措辞一变市场翻转), 至少 post=120 才安全 |
| CPI | 15 | **30** | 8:30 ET 盘前公布, 9:30 开盘前消化完, post=30 够 |
| NEWS_BREAKING | **0** | **15** | 突发新闻 (并购/FDA), 公布瞬间已发生, 只需 post=15 反应 |
| DIVIDEND / SPLIT | 5 | 5 | 行为温和 |
| OTHER | 15 | 15 | 兜底 |

**APScheduler 集成** (每分钟扫):
- `SELECT * FROM event_calendar WHERE event_at - pre_window_min*INTERVAL'1 min' <= NOW() AND event_at + post_window_min*INTERVAL'1 min' >= NOW()`
- 对每个命中事件: 找匹配 `active_reviews WHERE opportunity.symbol = event.symbol OR event.symbol IS NULL`; **UPDATE review_interval_sec 300 → 1** (1 秒 floor, 见 D1 Q4 语义)
- 窗口结束 (event_at + post_window_min < NOW()): UPDATE 回 300

**newsTicker callback 接入路径** (远景 §4 #4 真 collector ship 后, 本期留扩展点):
- IBKR `reqMktData` with `genericTickList=292` (news tick) 收到突发新闻 → MarketDataPool 内 callback handler 调 `INSERT INTO event_calendar (symbol, event_type='NEWS_BREAKING', event_at=NOW(), pre_window_min=0, post_window_min=15, source='AUTO_IBKR_NEWS', created_by='newsticker:reqId=X')`
- APScheduler 下次扫表 (1 分钟内) 自动触发熔断
- **本期不实施 newsTicker handler**, 但 schema/source enum 留扩展点

**索引**:
- btree on (event_at)
- composite (symbol, event_at) — 单 symbol 查询
- **UNIQUE on (symbol, event_at, event_type)** WITH `INSERT ... ON CONFLICT DO NOTHING` — 防手录重复

**Forward compat**: source 字段为远景 §4 #3 EventCalendarCollector + #4 SentimentCollector 留扩展点。本期客户告知"AAPL 5/8 16:30 ET 财报" → 你/Claude `INSERT` 一行即可。本周已知事件 5 min 录完。

### D5. `strategy_whitelist` — 12 策略 + direction mapping 查表 (合并 risk_gate_mapping)

**问题**: 北极星 §1 锁 12 种策略 + Layer 2 mapping (BULL_PUT_SPREAD → BULLISH 等反直觉), 当前散落 hardcoded 在代码里。

**决策**: 新建 `strategy_whitelist` 表, 一行 = 一个策略, **同表含 direction mapping (无需另建 risk_gate_mapping 单独表)**。

**注意 (Q1 用户 ack 反转)**: 4 腿流动性约束 (IRON_CONDOR/BUTTERFLY 限 SPY/QQQ/IWM + OI≥500/1000) **从北极星 hard gate 改为 soft hint** — Agent 2 LLM 看 10 维 bundle (dim 4 期权链 + dim 8 OI/volume) 自决, 不在 schema 写硬约束。本 spec 同步 **删 `liquidity_constraint_symbols` + `min_oi_per_leg` 字段**, 流动性指引改为 prompt 文件 (`prompts/agent2_shared/risk_guidelines.md`)。

**字段** (5 列, 简化):
- `strategy_name` varchar(32) PK — LONG_CALL / BULL_CALL_SPREAD / IRON_CONDOR / CALENDAR_SPREAD / DIAGONAL_SPREAD 等 12 个
- `direction_class` varchar(16) — BULLISH / BEARISH / VOLATILITY (Layer 2 mapping 数据源)
- `legs_count` smallint — 1 / 2 / 4 等 (审计/统计用, 不做 hard gate)
- `is_naked_short` boolean default false — 全部 false (本版禁裸卖空); 字段留 forward compat 防未来 SHORT_PUT_HEDGED 等加入
- `enabled` boolean default true — 临时禁某策略 (e.g. 流动性危机时)
- `description_zh` text — 客户友好简短描述
- `created_at` timestamptz
- `updated_at` timestamptz

**Seed 数据 (12 行, 必跟北极星 §1 同步, 见同步 test 下)**:
```
LONG_CALL          BULLISH    1  false  true  '看涨, 买 call, 亏损上限=权利金'
BULL_CALL_SPREAD   BULLISH    2  false  true  '看涨价差, 净 debit, 有限亏'
BULL_PUT_SPREAD    BULLISH    2  false  true  '看涨, 卖 put 收权利金, 价差宽度有限亏'
LONG_PUT           BEARISH    1  false  true  '看跌, 买 put'
BEAR_PUT_SPREAD    BEARISH    2  false  true  '看跌价差, 净 debit'
BEAR_CALL_SPREAD   BEARISH    2  false  true  '看跌, 卖 call 收权利金'
LONG_STRADDLE      VOLATILITY 2  false  true  '买波动, 同 strike call+put'
LONG_STRANGLE      VOLATILITY 2  false  true  '买波动, OTM call + OTM put'
IRON_CONDOR        VOLATILITY 4  false  true  '卖波动有保护 (Agent 2 看 dim 8 OI 自决标的)'
IRON_BUTTERFLY     VOLATILITY 4  false  true  '同上, 中心 strike'
CALENDAR_SPREAD    VOLATILITY 2  false  true  '赚时间价值, 卖近月买远月'
DIAGONAL_SPREAD    VOLATILITY 2  false  true  '不同 strike + expiry'
```

**风控 Enforcement 路径**:

| Layer | 用什么 hard gate | 拦什么 |
|---|---|---|
| **Layer 1** (schema enum) | 应用启动时 `SELECT strategy_name FROM strategy_whitelist WHERE enabled=true` 生成 enum 注入 LLM prompt schema (Structured Outputs) | LLM 物理无法输出禁止策略 (e.g. SHORT_CALL 不在 enum 里 LLM 输出不出来); 临时禁某策略改 enabled=false 即可 |
| **Layer 2** (mapping check) | `SELECT direction_class FROM strategy_whitelist WHERE strategy_name=X` 跟 opp.direction 比对 | LLM 给 LONG_CALL 但 opp.direction=BEARISH → 拒绝重跑一次; 仍违 → 失败单 |
| **流动性 (soft hint, 非 layer)** | **prompt 文件** `prompts/agent2_shared/risk_guidelines.md` 写: "100 手以上 4 腿策略, 必先看 dim 8 各腿 OI ≥ 1000 + 当日 volume ≥ 100 + bid-ask spread ≤ 5% mid; 不达标主动降级 SPREAD 或拒绝" | LLM 自己看数据判断 |

**Cache + Hot reload**:
- 应用启动时 cache enum + mapping (避免每次 risk_gate 查 DB)
- 改 strategy_whitelist 表后 = **重启服务 reload** (本期); 远景做 cache invalidation
- **二选一锁**: 选**重启**, 不做 hot reload (不增加并发风险)

**Seed 跟北极星同步 hard test** (机制硬化, 替代"自觉同步"):

**锁定数据源 = wiki `north-star-v1-target.md` §9 markdown 表** (不解析 memory §1 散文, 因为 markdown 表更结构化稳定; memory §1 是决策门禁骨架, wiki §9 是规格细节单一真相源)。

`tests/test_strategy_whitelist_north_star_sync.py`:
```python
import re
from pathlib import Path

WIKI_PATH = Path("docs-hub/docs/options-trader/specs/north-star-v1-target.md")
# 或者 git submodule / repo 路径; 项目内 wiki mirror 路径如不存在, fallback 直接读
# https://chunmiaoyu.github.io/Projects-Wiki/options-trader/specs/north-star-v1-target/

def parse_north_star_strategies_from_wiki(wiki_md_path: Path) -> dict[str, dict]:
    """解析 wiki §9 markdown 表 (固定 4 列: 方向/允许策略/备注), 提取 12 策略。

    锚定 §9 标题 '## <span class="h-num">9.</span> 方向策略白名单' 之后第一个表格。
    每行格式: | DIRECTION (中文) | STRATEGY_NAME / STRATEGY_NAME / ... | 共 N 种 |
    解析为: {STRATEGY_NAME: {direction_class: BULLISH/BEARISH/VOLATILITY, legs_count: 推断}}
    """
    text = wiki_md_path.read_text(encoding='utf-8')
    # 找 §9 表格 (markdown table 用 | 分隔)
    h9_match = re.search(r'## <span class="h-num">9\.</span>(.+?)## <span class="h-num">10', text, re.DOTALL)
    assert h9_match, "wiki §9 not found — wiki structure changed, sync test broken"
    section = h9_match.group(1)
    # parse rows
    rows = [line for line in section.split('\n') if line.strip().startswith('|') and 'BULLISH' in line or 'BEARISH' in line or 'VOLATILITY' in line]
    result = {}
    for row in rows:
        cells = [c.strip() for c in row.split('|')[1:-1]]
        direction_zh = cells[0]
        if 'BULLISH' in direction_zh: direction_class = 'BULLISH'
        elif 'BEARISH' in direction_zh: direction_class = 'BEARISH'
        elif 'VOLATILITY' in direction_zh: direction_class = 'VOLATILITY'
        else: continue  # null 行跳过
        strategies = [s.strip() for s in cells[1].split('/')]
        for s_name in strategies:
            # 移除可能的 "(有保护)" 等后缀
            clean = re.sub(r'\s*\(.+?\)\s*', '', s_name).strip()
            result[clean] = {'direction_class': direction_class}
    return result

def test_strategy_whitelist_seed_matches_north_star_wiki(db_session):
    """wiki §9 12 策略列表跟 DB seed 必一致, 不一致 fail."""
    wiki_strategies = parse_north_star_strategies_from_wiki(WIKI_PATH)
    db_strategies = {s.strategy_name: s for s in db_session.query(StrategyWhitelist).all()}

    # 名字集合一致
    assert set(wiki_strategies.keys()) == set(db_strategies.keys()), \
        f"Wiki/DB strategy name mismatch: wiki only={set(wiki_strategies.keys()) - set(db_strategies.keys())}, db only={set(db_strategies.keys()) - set(wiki_strategies.keys())}"

    # 每个策略 direction_class 一致
    for name, wiki_s in wiki_strategies.items():
        db_s = db_strategies[name]
        assert db_s.direction_class == wiki_s['direction_class'], \
            f"{name}: wiki={wiki_s['direction_class']} vs db={db_s.direction_class}"
        assert db_s.is_naked_short == False, f"{name}: 北极星永久禁裸卖空, db is_naked_short=True 违规"
```

**Wiki 文件路径解析**:
- 优先项目内 mirror (e.g. `docs-hub-mirror/north-star-v1-target.md` git submodule)
- Fallback `urllib.request.urlopen("https://chunmiaoyu.github.io/...")` 拉远程渲染前 markdown
- 实施时 plan 阶段确定哪种 (memory `project_client_dashboard.md` 提 docs-hub 镜像策略)

**CI 跑这个 test, 不一致 fail** → 改 wiki §9 必同步 DB seed 反之亦然 (机制硬化, 不靠人工自觉)。如果 wiki 散文格式变化导致 parser 失败, test 也 fail (锚定 `## <span class="h-num">9.</span>` 标题), 提示 wiki 结构变化要先评估再改 parser。

**索引**: 单表 12 行, 不需要额外索引 (PK 够)。

**Forward compat**: `enabled` / `is_naked_short` 字段为远景留 (临时禁策略 / 加裸卖空 hedged 变种, 都改表不改代码)。

### D6. `BrokerOrder` 改造 — 加 algo_strategy + split_order_batch_id + batch_intent_json

**问题**: 北极星 §1.6 分批下单 (本期 ship), 单仓位 ≥ 50 手必拆 → 一个业务下单 = N 个 BrokerOrder。需要标记同 batch + 算法类型 + batch-level 元数据。

**决策**: BrokerOrder 加 3 列:
- `algo_strategy` varchar(32) nullable — 5 值枚举: NULL (普通 LMT) / 'Adaptive' (IBKR Adaptive Algorithm 单腿) / 'COMBO_PATIENT' / 'COMBO_NORMAL' / 'COMBO_URGENT' (BAG combo 自实现拆腿 + 4 档调价)
- `split_order_batch_id` UUID nullable — 同 batch_id 的 orders 是同一组拆单 (NULL = 整单不拆)
- `batch_intent_json` JSONB nullable — batch-level 元数据 (原始 N 手 / 拆几批 / 算法参数 / 当前 batch 序号), 用于部分腿失败时 reconciliation: e.g. `{"total_qty": 100, "batch_size": 10, "batch_idx": 3, "interval_sec": 8, "spread_ratio": "patient"}`

**索引**:
- btree on (split_order_batch_id) — 按 batch 聚合查询 (e.g. "opp #5 这次入场拆了几批, 每批成交价")

**幂等**: 现有 `client_order_id` UNIQUE 已保证下单 idempotent; batch_id 不影响。

**为什么不另建 OrderBatch 表**: KISS + JSONB — batch_id 做 grouping marker + batch_intent_json 装 batch 元数据, 双一起够用。如果将来要做 batch-level 实体查询 (e.g. "全部 patient 模式拆单"), 再加表。

### D7. WorkflowTask `task_type` 加 DB CHECK + 应用层枚举扩展

**现状**: `WorkflowTask.task_type` 是 `String(32)` 软约束 (无 DB enum 也无 CHECK), 现有代码可能已用一些 task_type 字符串。

**决策**:
1. **alembic 0028 加 DB CHECK 约束** (硬约束防 silent typo bug, e.g. `'AGENT2_REVEW_TICK'` 入库后 handler dispatch miss → task 卡 PENDING 直到 max_attempts):
   ```sql
   ALTER TABLE workflow_tasks ADD CONSTRAINT workflow_tasks_task_type_check
     CHECK (task_type IN (
       'SYSTEM_WAKE_UP', 'SYSTEM_SLEEP', 'CONDITION_MET', 'AGENT2_REVIEW_TICK',
       'EXECUTE_DECISION', 'ENTRY_FILLED', 'EXIT_FILLED', 'EXPIRE_OPPORTUNITY',
       -- 加 legacy 列表防现有 task 数据炸 (gradual migrate)
       <现有 task_type 全部值, 实施 PR 前 grep 出来>
     ));
   ```
2. 应用层 (`src/options_event_trader/queue/event_types.py` 新文件) 定义 8 种 event_type Pydantic enum:

```python
class WorkflowEventType(str, Enum):
    SYSTEM_WAKE_UP = "SYSTEM_WAKE_UP"
    SYSTEM_SLEEP = "SYSTEM_SLEEP"
    CONDITION_MET = "CONDITION_MET"
    AGENT2_REVIEW_TICK = "AGENT2_REVIEW_TICK"
    EXECUTE_DECISION = "EXECUTE_DECISION"
    ENTRY_FILLED = "ENTRY_FILLED"
    EXIT_FILLED = "EXIT_FILLED"
    EXPIRE_OPPORTUNITY = "EXPIRE_OPPORTUNITY"
```

+ `EVENT_HANDLERS: dict[WorkflowEventType, Callable]` 集中 dispatch 表 — 新加 event_type 必加一行。

**对现有 task_type 兼容**: 实施 PR 前 grep 全项目找现有 task_type 字符串清单 (`grep -rn "task_type\s*=\s*['\"]" src/`), 加入 CHECK 列表防数据炸。Pydantic enum 加 `LegacyWorkflowEventType` 标 deprecated, gradual migrate。

**Risk**: 现有代码可能某处 hardcode `task_type="..."`, 需要 grep + 一次性 PR 全替换 + 加 invariant test (`find . -name '*.py' | xargs grep -E "task_type\s*=\s*['\"]" | wc -l == 0` 之外的固定 import) 防回归。

**lifecycle_status vs task_type 命名空间区分** (防 reviewer 混淆): `task_type` 是**任务事件类型** (e.g. AGENT2_REVIEW_TICK), `Opportunity.lifecycle_status` 是**业务实体状态** (e.g. HOLDING / COMPLETED / FAILED, 见 invariant 11 V2 0023)。两 namespace 完全独立, 8 event_type 不在 lifecycle_status 里也不在 task_status 里 (后者是 PENDING/RUNNING/DONE/FAILED 单独 namespace)。

## 5. 迁移策略 — Alembic 6 个 migration (拆 D5/D6 + D7 CHECK 独立)

按顺序 ship (各自独立可回滚, 拆细粒度防误操作):

1. `0024_add_active_reviews` — D1
2. `0025_add_market_data_subscriptions` — D2
3. `0026_add_pool_health_log` — D3
4. `0027_add_event_calendar` — D4
5. `0028_add_strategy_whitelist_seed` — D5 only (含 12 行 INSERT ON CONFLICT DO NOTHING data migration; 不 UPDATE 已存在行 → 客户改 PROD enabled=false 后下次 upgrade 不被 overwrite)
6. `0029_brokerorder_algo_split_batch_and_workflow_check` — D6 (BrokerOrder 加 3 列) + D7 (workflow_tasks 加 task_type CHECK 约束)

**应用层 D7 Pydantic enum** 不走 alembic (代码 PR 即可), 但 0029 的 DB CHECK 约束是 schema 改动必走 alembic。

**三环境部署顺序** (cloud-dev → UAT → PROD, 各 VM 独立 PostgreSQL 实例 — memory `project_oracle_cloud.md`三 VM 独立, PG 跟着各 VM 本地, 不共享 RDS):
1. **cloud-dev** → 各 VM 内 `docker exec` 跑 `alembic upgrade head`, 跑 unit + integration test 全过 → CI gate 自动触发
2. **UAT** → 同 cloud-dev + e2e test (paper IBKR 全管道) → 用户审通过手 ack
3. **PROD** → **必走周日 NZT 维护窗口** (memory `project_ibkr_ops.md` IBKR 强制重启时段 + 美股盘外, 双安全): 先停 worker → docker exec alembic upgrade head → 验证表结构 + seed 数据 → 启 worker; **不在窗口内不动 PROD**

**回滚 — PROD 一旦写入业务数据 = 事故恢复手段, 不是常规回滚**:
- 0024 active_reviews / 0025 market_data_subscriptions / 0026 pool_health_log: 一旦 PROD 跑起来就有持仓 opp 行, downgrade = **业务状态丢失** (持仓 review 排程丢, Pool 重启订阅意图丢)
- 0028 strategy_whitelist downgrade: **不能 TRUNCATE** (会立刻破坏 risk_gate Layer 1+2). downgrade 改为 **no-op + 手工 INSERT 历史 seed** 模式; 文档明示 "PROD 改 enabled=false 后, **永不再 alembic downgrade 跨 0028**"
- 0029 BrokerOrder 加列: 删列前必备份 PROD orders 表
- **结论**: PROD downgrade 仅作为 "ship 后立即发现致命 bug 必须紧急回退" 的事故恢复手段, 部署前必本地 + cloud-dev 全测过

## 6. 测试策略

| 层 | 测什么 | 文件 |
|---|---|---|
| **Unit** | 每个新 model + repository CRUD; ref_count 计算正确; active_reviews INSERT/UPDATE/DELETE idempotent | `tests/db/test_active_reviews.py` 等 5 个 |
| **Unit (sync gate)** | strategy_whitelist seed 跟北极星 §1 12 策略一致性 (D5 hard test) | `tests/test_strategy_whitelist_north_star_sync.py` |
| **Unit (handler idempotency)** | 每个 8 event_type handler 必有 idempotency unit test (同 task 重做 2 次结果一致, 防崩溃恢复破坏数据) | `tests/handlers/test_*_idempotent.py` × 8 个 |
| **集成** | WorkflowTask handler dispatch 8 种 event_type 全跑通; APScheduler tick 正确触发 AGENT2_REVIEW_TICK; market_data_subscriptions ref_count 多消费者去重正确 | `tests/integration/test_task_queue_dispatch.py` |
| **集成 (Pool 重启)** | MarketDataPool 重启后从 market_data_subscriptions 表恢复 ref_count, 全部 reqMktData re-subscribe 用新 reqId | `tests/integration/test_pool_restart_resubscribe.py` |
| **集成 (APScheduler 重叠)** | 多 APScheduler 实例并发扫 active_reviews, FOR UPDATE SKIP LOCKED 防双触发 | `tests/integration/test_scheduler_concurrency.py` |
| **集成 (Race condition)** | 多 worker 同时取同一 workflow_task, lock_token 防 double-claim | `tests/integration/test_workflow_task_race.py` |
| **集成 (DB CHECK)** | 故意 INSERT `task_type='AGENT2_REVEW_TICK'` (typo) 必抛 IntegrityError (D7 CHECK 真生效) | `tests/integration/test_task_type_check_constraint.py` |
| **e2e** | 北极星 architecture-walkthrough §4 Step 0-9 完整场景跑通 (AAPL 突破 280 → 入场 → review × N → 部分平仓 → 全平 → 取消订阅) — 用 paper IBKR | `tests/e2e/test_full_lifecycle_aapl_280_breach.py` |
| **e2e UI** | Dashboard 能正确显示 active_reviews / pool_health_log (持仓队列 + Pool 状态可观测) | `tests/e2e/test_smoke_frontend.py` 加 SC-W-7 |

**测试数据隔离**: e2e 仍用 `ai_trader_e2e` DB (跟 dev DB 隔离, invariant 22)。

## 7. 风险

1. **现有 WorkflowTask schema 不能被破坏** — 仅加 task_type CHECK 约束, 不改字段。CHECK 约束含现有 task_type 全清单 (实施 PR 前 grep 出), 防现有数据炸
2. **strategy_whitelist seed 跟北极星 §1 必同步** — 已加 hard test `tests/test_strategy_whitelist_north_star_sync.py` (D5), CI fail 自动挡, 不靠人工自觉. finding F-2026-05-06-STRATEGY-WHITELIST-SEED-DATA 跟踪
3. **6 alembic migration 顺序敏感** — 0029 (BrokerOrder + workflow CHECK) 必在前 5 个 migration 之后 (技术上独立, 但部署文档锁顺序防误操作)
4. **三环境 backfill** — 现有 opportunities 行 `origin_type` 已有 server_default='USER_SUBMITTED' (0017 ship), 不需要数据 backfill。新表全是 INSERT-only, 无 backfill。BrokerOrder 加列也无 backfill (新列 nullable, 旧 order 行 algo_strategy=NULL 表示"非分批")
5. **strategy_whitelist hot reload — 锁定为重启 (不做 hot reload)**: 改表后 risk_gate Layer 1 schema enum 必重启服务 reload。**不做 hot reload** (不增加并发风险, KISS); 改 strategy_whitelist 是低频操作 (一年 < 5 次), 重启代价可接受
6. **PROD downgrade ≠ 常规回滚** — 见 §5 详, 一旦写入业务数据 PROD downgrade = 事故恢复手段, 部署前必本地 + cloud-dev 全测过 (PROD 一定要先在两个 lower env 跑过)
7. **strategy_whitelist legs_count 字段不做 hard gate** — 防意外复活 invariant 17 (max_legs 已废为 soft hint 2026-04-30); legs_count 仅审计/统计/Dashboard 显示用, risk_gate 不读这字段
8. **流动性约束改为 prompt 文件 (Q1 反转)** — 改 `prompts/agent2_shared/risk_guidelines.md` 阈值 (e.g. OI 1000 → 1500) **不需 alembic, 也不需重启** (LLM 每次 review 都重读 prompt 文件), 但要更新北极星 §1 + 项目 CLAUDE.md 同步描述

## 8. 关联文档

- 北极星 memory `project_vision_and_north_star.md` §1 (4 Pool/Client + 8 event_type + 12 策略) + §4 #1.5 (本 spec) + §5 hooks (本 spec 落地 6 个待嵌入)
- wiki `architecture-walkthrough.md` §9 (DB 5 类表骨架, 本 spec 详化)
- 现有 model: `src/options_event_trader/db/models.py`
- 现有 alembic: `alembic/versions/` (本 spec 加 0024-0028 5 个)

## 9. 五专家评审待办 (本 spec 写完后即派, 全 pass 才进 plan)

按全局 §15 + 项目 §8 = 5 位:

| 专家 | review 重点 |
|---|---|
| 软件工程 | schema 设计 / 索引 / FK 约束 / 是否漏掉关键字段; D5 strategy_whitelist 跟代码 hardcoded 迁移路径; D7 task_type 软约束 vs DB enum 取舍 |
| AI 规则记忆 | 跟北极星 §5 hooks 对齐 (6 个待嵌入是否真落地); 跨 session memory 不漂移; risk_gate 二层完整性; ER 图跟 architecture-walkthrough 一致 |
| 云部署 | 5 个 alembic migration 三环境部署顺序; strategy_whitelist seed 跟代码同步策略; pool_health_log 体量 (一年多少行?); event_calendar manual 录入操作流程 |
| 期权实战 | strategy_whitelist seed 12 种是否覆盖客户常见场景; CALENDAR/DIAGONAL liquidity 约束是否够 (没限 SPY/QQQ/IWM?); event_calendar pre/post window 默认 15min 是否对所有事件类型合理 |
| IB/TWS | market_data_subscriptions ref_count 持久化跟 IBKR reconnect 行为对齐; subscriber_kind 是否够细 (含 BUNDLE_REVIEW); event_calendar 跟 IBKR newsTicker 加塞机制对齐; pool_health_log error_msg 字段够装 IBKR error code 细节 |

任一反对必修, 不走多数票。

## 10. Plan (5 专家全 pass + 用户审后)

按 superpowers:writing-plans 框架写 implementation plan, ship 时间预估:
- Phase 1 (2-3 hr): 5 alembic migration 写 + 5 model 加 + repository 层 + unit test
- Phase 2 (1-2 hr): WorkflowEventType enum + handler dispatch 表 + 集成 test
- Phase 3 (1 hr): 三环境部署
- Phase 4 (2 hr): e2e 完整场景测试 (paper IBKR + UI snapshot)

**总估 6-8 hr 工程, 不阻塞 §1.6 分批下单 spec (那个 spec 依赖本 spec 的 BrokerOrder 改动, 但 spec 本身可以并行写)**。

---

**END OF DESIGN SPEC**
