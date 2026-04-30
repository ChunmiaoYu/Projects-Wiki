# Step F 执行接入 — Agent 2 决策真改单 + Trace 持久化 + Paper/Live 硬门

> **日期**：2026-04-25
> **状态**：Approved（brainstorming 2026-04-25 下午完成，五专家全员 approve with P0 addressed，13 组 P0 + 精选 P1 全部吸收）
> **前置**：Phase 11 Step A-E + EI-1/2/3 全部 ship（2026-04-24 100 手 paper smoke 跑通老链路）
> **后置（独立 spec）**：`2026-04-25-scenarios-v2-framework-design.md`（与本 spec Phase A 并行）
> **Blast radius**：Agent 2 PARTIAL_CLOSE / FULL_CLOSE / ADJUST_STOP 首次真改单——盘中部署必须走 "DEV paper → UAT paper → UAT live 小金额 → PROD" 阶梯

---

## 1. 目的

把 `Agent2DecisionTrace` 从"log-only"变成"真执行 + 真审计"：

1. **决策 → 执行的最后一公里**：`worker/agent2_scheduler.py:114-127` `review_fn` 拿到 orchestrator 返回的 trace **直接 return 扔掉**，本 spec 让它路由到 `exit_order_placer`（PARTIAL/FULL_CLOSE）或 UPDATE 新 `position_adjustments` 表（ADJUST_STOP）
2. **决策持久化**：`agent2_decisions` DB 表永远空（ORM model 在 `db/models.py:380`，无 INSERT 代码）→ 本 spec 加 `decision_repository`
3. **Trace 冷存**：完整 trace（bundle + prompt + LLM raw + risk_gate_result）gz 落盘供事后审计（Internal 30 天，Public 脱敏）
4. **News 维度接口预留**：`NewsCollector` stub + `NewsSnapshot` schema，本版返 null，未来接入时无需 bundle schema 变更

## 2. 非目的

- ❌ News collector 真实接入（Phase 12+，需外部事件日历）
- ❌ `ADJUST_STOP` 走 IBKR bracket `modifyOrder` 服务器侧持续监控（本版用 DB + 60s monitoring cycle 兜底，长期迁移记 finding）
- ❌ `agent2_decisions` 表分区 / 历史归档（Phase 12，等行数超 100K 再做）
- ❌ Agent 2 入场决策（工作点 A）的 payoff 蒙卡 sanity check（P1 记录在 §14，Phase 12 实装）

## 3. 架构

```
┌─────────────────────────────────────────────────────────────┐
│ worker/loop.py:157  (每 20 cycle ≈ 5min, 改为 ≤ 60s stop-check)│
└──────────────┬──────────────────────────────────────────────┘
               ↓
┌─────────────────────────────────────────────────────────────┐
│ worker/agent2_scheduler.py::review_fn (重写)                  │
│   1. 调 orchestrator.ai_review_one_position → trace         │
│   2. 调 decision_repository.insert_pending(trace) → row.id  │
│   3. 调 decision_dispatcher.dispatch(trace, row.id) → result│
│   4. 调 decision_repository.mark_executed(row.id, result)   │
│   5. 调 trace_archiver.persist(trace, row.id) (async OK fail)│
└──────────────┬──────────────────────────────────────────────┘
               ↓
         ┌─────┴────────────┬──────────────┬──────────────┐
         ↓                  ↓              ↓              ↓
    HoldDecision    PartialClose     FullClose      AdjustStop
    (log only)      (liquidity      (full qty      (insert 
                     pre-check →     via           position_
                     exit_order_     execute_      adjustments
                     placer.exec_    exit_          + monitoring
                     exit_escalation escalation)   60s 读最新)
                     (qty_override))
```

## 4. 模块设计

### 4.1 `services/agent2/decision_dispatcher.py`（新）

**职责**：消费 `Agent2DecisionTrace`，按 `decision.action` 路由到执行层。幂等边界由 caller (agent2_scheduler) 的 `bundle_id` UNIQUE 保证。

**签名**：
```python
@dataclass
class DispatchResult:
    executed: bool
    order_ids: list[str]
    adjustment_id: UUID | None     # ADJUST_STOP 时的 position_adjustments.id
    skipped_reason: str | None     # liquidity gate 拒或 < 1 张退化时
    error: str | None

async def dispatch(
    *, trace: Agent2DecisionTrace,
    decision_row_id: UUID,     # decision_repository.insert_pending 返回
    db: Session, ibkr_client: Any,
    settings: Settings, alert_sink: AlertSink,
) -> DispatchResult: ...
```

**路由规则**（四分支）：

| Decision | 前置校验 | 执行路径 | 后果 |
|---|---|---|---|
| `HoldDecision` | 无 | log 一行 | DispatchResult(executed=False, skipped_reason="HOLD") |
| `PartialCloseDecision` | ① `spread_pct > 8%` 或 `bid_size < qty_override` → skip 并告警 Agent 2 下 cycle 重决策 ② `floor(pos.qty × pct) < 1` → 降级 FULL_CLOSE ③ `DTE ≤ 2 and leg.is_short and leg.gamma > threshold` → skip（pin risk） | BAG 2 腿逐腿按比例平仓（invariant 21）：每腿独立 `execute_exit_escalation(leg_position, quantity_override=floor(leg.qty × pct))`；两腿不对称 partial fill 时补偿已有 leg（§7 详）| DispatchResult(executed=True, order_ids=[...]) |
| `FullCloseDecision` | `FullCloseDecision` 由 risk_gate_v2 R7 override 时强制通过不校验 | `execute_exit_escalation(position)` 全量 | DispatchResult(executed=True, order_ids=[...]) |
| `AdjustStopDecision` | 无 | INSERT `position_adjustments`（FK 到 `agent2_decisions.id`） | DispatchResult(executed=True, order_ids=[], adjustment_id=...) |

**Paper/Live 硬门**（entry 校验，吸收 P0-I）：
```python
if settings.app_env == "prod":
    if not position.paper_smoke_passed_at:
        alert_sink.send(level="CRITICAL", title="Agent 2 dispatch blocked on live unsmoked position", ...)
        raise DispatchBlocked("live position lacks paper_smoke_passed_at")
if not settings.enable_agent2_dispatch:
    return DispatchResult(executed=False, skipped_reason="flag off")
```

### 4.2 `services/agent2/decision_repository.py`（新）

**职责**：`agent2_decisions` 表的唯一写入真相。`trace_archiver` 失败绝不影响这里。

```python
def insert_pending(db: Session, trace: Agent2DecisionTrace) -> UUID:
    """写入 agent2_decisions 行，executed=False。bundle_id UNIQUE 保证幂等。
    已有同 bundle_id 的 row → 返回该 row.id，不重复插入（幂等键）。"""

def mark_executed(db: Session, row_id: UUID, result: DispatchResult) -> None:
    """UPDATE executed=True, executed_at_utc=now, order_ids=result.order_ids,
    adjustment_id=result.adjustment_id。"""
```

**表结构**（已有 ORM `db/models.py:380`，本 spec 不加列，只新增 **constraint**）：
- `bundle_id` 加 UNIQUE constraint（alembic migration `2026-04-25-agent2-decisions-unique-bundle-id`）
- 新外键 `adjustment_id` → `position_adjustments.id`（可选字段，只 ADJUST_STOP 非 null）

### 4.3 `services/agent2/trace_archiver.py`（新）

**职责**：冷存 / debug 用，失败只 log warning，**不阻塞 dispatch**。

```python
def persist(trace: Agent2DecisionTrace, decision_row_id: UUID,
            trace_dir: Path, retention_days: int) -> Path | None:
    """落盘到 {trace_dir}/YYYY-MM-DD/{symbol}/{bundle_id}.json.gz
    策略：先写 .tmp，成功后 atomic rename(.tmp → .json.gz)。
    Retention 由独立 systemd.timer + python cleanup script 跑（§11 详）。
    返回路径或 None（落盘失败）。"""

def cleanup_stale_tmp(trace_dir: Path) -> int:
    """worker 启动时调：扫 .tmp 残留（worker crash 留下的半截文件），
    告警 alert_sink + 删除。返回清理数量。"""
```

**路径结构**（吸收 SWE-P2-1 / Cloud-P2-1）：`/opt/oet/{env}/traces/YYYY-MM-DD/{symbol}/{bundle_id}.json.gz`
- 本地 dev：`./traces/...`
- 文件名加 `{env}_` 前缀做多 env 拉文件防混淆

### 4.4 `services/data_collection/collectors/news.py`（新 stub）

```python
class NewsCollector(Protocol):
    async def collect(self, *, symbol: str, as_of_utc: datetime) -> NewsSnapshot | None: ...

class NullNewsCollector:
    """V1 默认实现：永远返 null。Agent 2 prompt 必须处理 news=None 分支。"""
    async def collect(self, *, symbol, as_of_utc) -> None:
        return None
```

`domain/agent2_bundle.py` 新增（schema 永久化，未来 collector 接入不改）：
```python
class NewsSnapshot(BaseModel):
    headlines: list[NewsHeadline]
    sentiment_score: float | None = None   # -1 to +1
    ages_sec: int
    source: str                            # "ibkr" / "external_vendor"
```

`bundle_packager.py`：news 采集失败降级 None（非关键 collector）。

### 4.5 `worker/agent2_scheduler.py::review_fn` 重写

旧代码：拿 trace 后 `return None`。新代码（伪代码）：
```python
async def review_fn(*, opportunity_id, position_id):
    trace = await review_impl(opportunity_id=..., strategy_run_id=pos.strategy_run_id, ...)
    # 1. DB 先 (事实层 invariant 对齐)
    decision_row_id = decision_repository.insert_pending(db, trace)
    # 2. 执行 (失败不回滚 DB，dispatch_result 记录 error)
    dispatch_result = await decision_dispatcher.dispatch(
        trace=trace, decision_row_id=decision_row_id,
        db=db, ibkr_client=ibkr_client, settings=settings, alert_sink=alert_sink,
    )
    # 3. 标记执行结果
    decision_repository.mark_executed(db, decision_row_id, dispatch_result)
    # 4. 冷存 (失败只 warn, 不影响前 3 步)
    try: trace_archiver.persist(trace, decision_row_id, settings.trace_dir, settings.trace_retention_days)
    except Exception: LOGGER.warning("trace archive failed, DB still authoritative", exc_info=True)
```

`services/partial_fill.trigger_agent2_review` 同步改（复用 review_fn 包装）。

## 5. 数据模型

### 5.1 新表 `position_adjustments`（吸收 SWE-P0-1 + Opt-P0-1）

```python
class PositionAdjustment(Base, TimestampedUUIDMixin):
    __tablename__ = "position_adjustments"

    position_id: UUID = FK(positions.id, index=True)
    agent2_decision_id: UUID = FK(agent2_decisions.id, index=True)  # 闭环 trace 链
    adjustment_type: Literal["STOP_LOSS_PCT", "TRAILING_STOP_PCT"]
    old_value: float | None        # 调整前（可能是原始 spec 或前一次 adjustment）
    new_value: float               # 调整后
    effective_from: datetime       # 生效时间（= created_at）
    superseded_at: datetime | None # 被下一条 adjustment 取代时填（null = 当前生效）
    reasoning: str                 # Agent 2 决策 reasoning 摘要（便于查）
```

`monitoring/rules.py` 查询："该 position 最新一条 `superseded_at IS NULL` 的 adjustment"，有则用 `new_value`，无则 fallback `opportunities.stop_loss_spec`。

### 5.2 `positions` 表加 1 列（吸收 Cloud-P0-3）

```python
paper_smoke_passed_at: datetime | None  # 该 position 是否通过 paper smoke 验收；dispatcher 在 PROD 强制要求非 null
```

### 5.3 ADJUST_STOP 语义（吸收 Opt-P0-1 硬性写死）

- `stop_loss_pct` 基准 = **期权合约 mid 从 entry_cost 算**（不是 underlying 价格，不是 delta-weighted）
- BAG combo 的 stop 按**组合净 mid**（long legs mid - short legs mid）
- `trailing_stop_pct` 触发用 **mark-to-market bar close**（每 1m bar close 后更新 high_water_mark，非 tick 级避 whipsaw）

### 5.4 Bundle 扩字段（吸收 Opt-P0-5）

`domain/agent2_bundle.py`：
```python
class PositionLeg(BaseModel):
    ...
    days_to_expiry: int   # 每 leg 的 DTE，必填（工作点 B）

class Underlying(BaseModel):
    ...
    next_earnings_date: date | None          # next scheduled earnings
    has_earnings_within_dte: bool            # any leg DTE 内是否跨 earnings
```

`data_bundle_schema.md` 相应更新（markdown 10 条结构不变，字段明细增补）。

## 6. 流程：PARTIAL_CLOSE BAG 2 腿逐腿补偿（吸收 IB-P0-5）

```
Agent 2 决策: PARTIAL_CLOSE 50%, BAG (+1 long call, -1 short call)

Dispatcher:
  1. 对 long call leg: execute_exit_escalation(leg_A, qty_override=floor(qty_A × 0.5))
     → LMT → chase → MKT 升级链 (老链路继承)
  2. 等 leg_A 完全成交或 MKT 兜底
  3. 对 short call leg: execute_exit_escalation(leg_B, qty_override=floor(qty_B × 0.5))
  4. 两腿结束后验对称性:
     - 若 leg_A 成交 5 张，leg_B 成交 3 张 → 仓位 delta-unbalanced
     - 补偿：对 leg_B 再发 2 张 MKT 补平 (invariant 16 不甩给用户)
     - 补偿失败 (spread 极宽或无 fill) → R6 circuit breaker (alert + 停止 Agent 2 dispatch)
```

## 7. 错误处理 + 幂等

- **bundle_id UNIQUE 做幂等 key**：同一 bundle_id 重放（worker restart + 旧 review 重跑）→ `decision_repository.insert_pending` 返回已存在 row，**不重复插入**。dispatcher 看到 decision_row 已 executed=True 直接短路
- **dispatcher 内部幂等**：每腿 order_id 先落 `agent2_decisions.order_ids` 再下单。重放时 `orderStatus=Submitted/Filled` 的 order_id 跳过
- **DB 先于执行**：`insert_pending` 失败 → 整个 review 退出不执行，保 fail-closed
- **liquidity gate 拒绝**：`skipped_reason` 字段记录，alert_sink 发 WARNING。Agent 2 下 cycle 会看到 position unchanged + 新 market state 重决策

## 8. 安全硬门

1. **Paper/Live 硬门**：`positions.paper_smoke_passed_at` 为 null + `app_env=prod` → dispatcher raise `DispatchBlocked`
2. **Feature flag**：`.env.prod` 初始 `ENABLE_AGENT2_DISPATCH=false`，用户手动打开（流程：UAT live 小金额 2-3 天验收 → 打开 flag）
3. **SIGTERM grace**（吸收 Cloud-P0-2）：worker 改 systemd unit `KillSignal=SIGTERM` + `TimeoutStopSec=30`，worker 捕获 SIGTERM 后设 `_shutting_down=True` 不再领新 review，当前 review_fn 跑完落盘再退
4. **`.tmp` 原子 rename**：`trace_archiver` 永远先写 `.json.gz.tmp` 再 `os.rename`，失败扫残留

## 9. 配置 + 环境（吸收 Cloud-P0-4 + J）

三环境 `.env.{dev,uat,prod}` 同 commit 加：
```
TRACE_DIR=/opt/oet/{env}/traces              # 本地 dev = ./traces
TRACE_RETENTION_DAYS=30                      # dev=7 / uat=30 / prod=30
ENABLE_AGENT2_DISPATCH=true                  # prod 初始 false
AGENT2_STOP_CHECK_INTERVAL_SEC=60            # 吸收 IB-P0-2，从 5min 降到 60s
IBKR_CLIENT_ID_L3_RUNNER=29                  # L3 scenarios nightly 专用
```

`deploy.sh` 开头 assert：
```bash
for v in TRACE_DIR TRACE_RETENTION_DAYS ENABLE_AGENT2_DISPATCH AGENT2_STOP_CHECK_INTERVAL_SEC; do
  grep -q "^${v}=" .env || { echo "FATAL: $v missing"; exit 1; }
done
```

memory `project_oracle_cloud.md` 同步指针："trace_dir / retention / dispatch flag 以 .env.{env} 为真相"（不写具体数字）。

## 10. 测试策略（TDD，每段代码先测后码）

### 10.1 单测（`tests/test_*`）
- `test_decision_dispatcher_hold.py`：HoldDecision → 不调 placer
- `test_decision_dispatcher_partial_close.py`：50% on 10 张 → `quantity_override=5`；1 张 < 1 退化 FULL_CLOSE；spread > 8% skip
- `test_decision_dispatcher_partial_close_bag.py`：2 腿 BAG 逐腿 + 不对称补偿
- `test_decision_dispatcher_full_close.py`
- `test_decision_dispatcher_adjust_stop.py`：INSERT `position_adjustments` + 旧 row `superseded_at` 写入
- `test_decision_dispatcher_paper_live_gate.py`：app_env=prod + paper_smoke_passed_at=null → raise
- `test_decision_repository_idempotency.py`：同 bundle_id 重放 → 同一 row.id
- `test_trace_archiver_atomic_rename.py`：.tmp 崩溃场景 + cleanup_stale_tmp
- `test_news_collector_null_stub.py`

### 10.2 集成测（MockIBKR + 真 e2e DB）
- `tests/test_agent2_execution_integration.py`：造 OPEN position → fire review → 断 `agent2_decisions` row executed=True + `position_adjustments` row（若 ADJUST_STOP）+ mock order 下过

### 10.3 e2e（invariant 22 Frontend Gate 延伸）
- `tests/e2e/test_smoke_frontend.py::SC-05`：POST 造持仓 → 手动触 review cycle → 断前端 position detail 页显示 Agent 2 调整记录 + DB 验

### 10.4 Paper 盘中 smoke
- `scripts/agent2_paper_smoke.py`：人工造 1 张 SPY 远月 long call paper position → 触一次 review → 观察真出 PARTIAL_CLOSE 或 ADJUST_STOP → fill + DB + trace 文件齐全。**产出 JSON report push 到 repo**（吸收 CC-P1-5 跨 session 异步）供下 session 读

### 10.5 `scripts/check_scenario_counts.py`（吸收 CC-P0-4）
PR 中 scenarios 数量门：L1 ≥ 50 / L2 ≥ 20 / L3 ≥ 5 / 每大类 L1 ≥ 3 + L2 ≥ 1。CI 自动跑。

## 11. 部署 & 迁移

### 11.1 Alembic migration
1. `2026-04-25-01-agent2-decisions-unique-bundle-id`：加 UNIQUE constraint + `adjustment_id` 外键列
2. `2026-04-25-02-position-adjustments`：新表
3. `2026-04-25-03-positions-paper-smoke-passed-at`：加列（nullable）
4. `2026-04-25-04-bundle-leg-dte-earnings`：bundle schema 字段不在 DB，跳过 migration

### 11.2 部署顺序（三环境）
DEV（push dev 自动 deploy）→ UAT（手动 tag + Friday 21:00 NZ 部署窗）→ PROD（独立手动 + weekend only）。每环境：
1. `alembic upgrade head`
2. 部署 code
3. `systemctl restart oet-{env}-worker oet-{env}-api`
4. 健康检查（deploy.sh retry loop 30s，吸收 P3 `F-2026-04-23-DEPLOY-HEALTH-CHECK-RACE` 顺带修）
5. PROD 部署后 `ENABLE_AGENT2_DISPATCH=false` 保持，**用户在 UAT live 验收后手动开**

### 11.3 systemd trace cleanup timer
`/etc/systemd/system/oet-trace-cleanup-{env}.timer`（Persistent=true）+ `.service` 调用 `python -m options_event_trader.scripts.trace_cleanup --env {env}`，每日 UTC 04:00 跑。

### 11.4 deploy.sh 顺带修
- `F-2026-04-23-DEV-API-PORT-HARDCODED`：systemd unit 改 `EnvironmentFile` + `${API_PORT}` 读取
- `F-2026-04-23-DEPLOY-HEALTH-CHECK-RACE`：health check retry loop 15s max

## 12. Definition of Done

- [ ] 所有 5.x 单测 + 集成测 + e2e SC-05 全绿
- [ ] Alembic 4 个 migration upgrade/downgrade 双向测试通过
- [ ] DEV paper smoke：人工 SPY position + 真触 PARTIAL_CLOSE → DB 行 + trace 文件 + 前端显示全齐
- [ ] UAT paper 24h 连续跑无 CRITICAL
- [ ] UAT live 小金额（< $100 risk）2-3 天验收（吸收 P1 `F-2026-04-21-PAPER-TO-LIVE-DELTA-UNKNOWN`）
- [ ] 三环境 `.env.{env}` 齐 + deploy.sh assert 通过
- [ ] `agent2_paper_smoke_report.{date}.json` 产物 push repo
- [ ] task_plan 五问表顶部加 `agent2_execution_wired: true` + finding `F-2026-04-25-AGENT2-REVIEW-TRACE-ONLY` 同 commit 删（吸收 CC-P0-3）
- [ ] memory `project_oracle_cloud.md` 同步 + `project_agent2_execution.md` 新建（吸收 CC-P1-1）

## 13. 设计决策（Why）

| 决策 | 原因 |
|---|---|
| ADJUST_STOP 用独立 `position_adjustments` 表而非 positions 两列 | 金融可审计：Agent 2 每 5min 决策，一仓位调 10+ 次，覆盖式更新丢历史（SWE-P0-1）|
| ADJUST_STOP 用 DB + 60s monitoring cycle 而非 IBKR bracket modifyOrder | 改动小先落地，overshoot 从 5min→60s 大幅收窄；长期迁 bracket 记 finding F-2026-04-25-ADJUST-STOP-SERVER-SIDE P2 |
| `decision_repository` + `trace_archiver` 拆两个模块 | DB 是事实层不能丢；trace gz 是冷存可丢。绑一起磁盘满会拖垮决策写入（SWE-P0-3）|
| bundle_id UNIQUE 做幂等 key | worker restart / IBKR 重连时同 bundle 重跑，避免 "多砍一半仓位"（SWE-P0-2）|
| BAG 2 腿逐腿按比例平 + 补偿 | 对齐 invariant 21（BAG combo 始终 LMT + 逐腿拆单）；IBKR BAG partial fill 不对称是真实痛点（IB-P0-5）|
| PARTIAL_CLOSE liquidity pre-check（spread > 8% skip） | 500 万 RMB 量级 + OTM 远月滑点能吃 30-50% 利润（Opt-P0-2）|
| Paper/Live feature flag + position.paper_smoke_passed_at 硬门 | 第一次 live PARTIAL_CLOSE 不能直接在客户 200 万真仓 debug（Cloud-P0-3）|
| News=null 但 schema 字段预留 | 修一次，未来接入不改 bundle schema / 不影响历史 scenarios（Opt + IB 共识）|
| trace_archiver 先 .tmp 再 atomic rename | worker SIGKILL 场景保证不留半截 gzip（Cloud-P0-2）|

## 14. 未解决 / 后续技术债

| 项 | 优先级 | 何时做 |
|---|---|---|
| `F-2026-04-25-ADJUST-STOP-SERVER-SIDE`：迁到 IBKR bracket modifyOrder 服务器侧 | P2 | Phase 12 |
| `F-2026-04-25-ENTRY-SONNET-SANITY-MONTE-CARLO`：工作点 A 加 payoff 曲线蒙卡 | P2 | Phase 12（吸收 Opt-P1-5）|
| `F-2026-04-25-NEWS-COLLECTOR-INTEGRATION`：真 news collector 接入 | P2 | Phase 12+ |
| `F-2026-04-25-AGENT2-DECISIONS-PARTITION`：表分区 / 归档（100K 行阈值）| P3 | 半年后 |
| `F-2026-04-25-MID-BASED-LOCKED-CROSSED-MARKET`：locked/crossed fallback NBBO | P2 | Phase 12（吸收 IB-P1-7）|
| `F-2026-04-25-BUNDLE-ORDER-FLOW`：bundle 加 `order_flow.large_prints_nearby` 和 `sector_vs_spy_5d_relative` | P3 | Phase 12+（吸收 Opt-P2）|

---

**五专家签字**（brainstorming 2026-04-25）：全员 approve with P0 addressed。本 spec 吸收 13 组 P0 + 精选 P1。

**归档**：内部讨论 → `discussions/2026-04-25-step-f-scenarios-v2-5expert-review.md`（项目内部 repo, 不在公开 docs-hub）。脱敏公开版 → `docs-hub/docs/options-trader/specs/2026-04-25-step-f-execution-wiring-design.md`（本 spec 镜像）+ `docs-hub/docs/options-trader/decisions/2026-04-25-agent2-execution-wiring.md`（客户友好摘要）。
