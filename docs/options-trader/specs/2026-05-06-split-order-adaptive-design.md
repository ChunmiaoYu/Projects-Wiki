---
title: 分批下单 + Adaptive 算法本期 ship — Design Spec
date: 2026-05-06
status: design-approved-pending-plan
related_north_star_path: §4 #1.6 (2026-05-06 加; 大单滑点是先决条件不是后期优化)
related_north_star_section: §1 (单仓位 ≥ 50 手必拆 / 单腿 Adaptive / 多腿走 invariant 21 4 档)
related_invariants_referenced:
  - 21 (限价单 mid-based 4 档 spread_ratio, 本 spec 复用 + 单腿 Adaptive 是局部豁免 + partial leg R7 应急 MKT 是 invariant 21 R7 类豁免)
  - 13 (修改 = 取消原单 + 创建复制单, 跟 split_order_batch_id 是不同 namespace)
related_dependencies:
  - Phase B spec docs/superpowers/specs/2026-05-06-db-schema-task-queue-design.md D6 (BrokerOrder 加 algo_strategy/split_order_batch_id/batch_intent_json 列, alembic 0029)
related_findings_to_create:
  - F-2026-05-06-SPLIT-ORDER-PAPER-VALIDATION (P1, 100 手 LONG_CALL + IRON_CONDOR paper 真测)
  - F-2026-05-06-PARTIAL-LEG-TIMEOUT-PAPER-VALIDATION (P1, 90s 硬 timeout MKT 应急 paper 验)
  - F-2026-05-06-COMPUTE-SPLIT-THRESHOLD-TUNE (P1, per-symbol 阈值公式 paper 调)
  - F-2026-05-06-INTERVAL-JITTER-TUNE (P2, jitter ±2-5s 实测最优值)
  - F-2026-05-06-DEPLOY-HEALTH-GATE (P1, deploy.sh 加 alembic revision check, 跟 Phase B 同 finding)
  - F-2026-05-06-SYMBOL-ROUTES-EXEC-OVERRIDE (P2, 未来 per-symbol min_qty 覆盖看 config/symbol_routes.yml 加字段)
five_expert_review:
  software_engineer: pass
  cc_ai_rules: pass
  oci_deployment: pass-after-amendments
  options_practitioner: pass-after-amendments
  ib_tws: pass-after-amendments
---

# 分批下单 + Adaptive 算法本期 ship

## 1. 背景

北极星 §1.6 (2026-05-06 加) 锁定: **单仓位 ≥ 50 手必拆**。客户实盘单仓位 100+ 手 (~$280k USD 账户), 直接整单 LMT 滑点 $0.10-0.30/手 = 单笔 $1k-3k = 交易成本 2-6%。**这不是优化是先决条件**, 不能放远景。

**好消息**: 现有架构已 ship 80% 基础设施, 只需 wrapper 串起来:
- `services/entry_escalation.py` — 入场链 4 档 spread_ratio (patient → normal → urgent → market) ✓
- `services/exit_order_placer.py:132` + `:322` — Adaptive Algorithm 调用 (`order.algoStrategy="Adaptive"`) ✓
- `services/order_placer.py:572` + `:666` — 4 档 escalation 集成入口 + mode 参数透传 ✓
- `services/pricing.py:77` — mid-based LMT + spread_ratio 配置查 `config["pricing"]["spread_ratio"]` ✓
- `services/upgrade_chain.py` — 升级路径定义 (单腿 4 档 / BAG combo 3 档 invariant 21) ✓
- `services/emergency_close.py` — R7 紧急 MKT 豁免 (唯一 invariant 21 局部豁免) ✓
- `services/liquidity_gate.py` — 流动性门 ✓

**Phase C 真正 scope** (~4-5 hr 工程):
1. **新建 `services/split_order_dispatcher.py`** — wrapper 接口 + 触发 + 路由
2. **集成入口** 在 `order_placer.py` / `exit_order_placer.py` 加 `if total_qty >= 50: dispatcher else: 原路径`
3. **依赖 Phase B 的 BrokerOrder 加列** (algo_strategy / split_order_batch_id / batch_intent_json 已 spec)
4. **paper 真测 ≥ 1 次** (LONG_CALL 100 手 Adaptive + IRON_CONDOR 100 手拆腿)

## 2. 北极星 §13b 5 问 cross-check

| # | 问题 | 答 | 偏离? |
|---|---|---|---|
| 1 | 用户提交流 / AI 自主流共用? | 是, 分批下单是 OrderClient 抽象内的执行机制, 跟 origin_type 无关 (Discovery Agent ship 时同一个 dispatcher) | ✓ 不偏离 |
| 2 | Discovery Agent ship 时是否要重写? | 不, dispatcher 只看 contract + qty, 不关心来源 | ✓ 不偏离 |
| 3 | Forward compat hook 留齐? | 是, 落地北极星 §5 中"业务代码必走 OrderClient 抽象" hook (Phase B 已 spec, dispatcher 仍走 OrderClient) | ✓ 不偏离 |
| 4 | 是否硬编码"用户必填"? | 否, 客户原话不变, 拆单是底层执行细节 | ✓ 不偏离 |
| 5 | 是否前置 AI 自动决策为"用户必给"? | 否, Agent 2 输出策略 + qty, dispatcher 接管拆单, 客户/Agent 2 都不感知 | ✓ 不偏离 |

**5 问全过, 此设计与北极星目标关系**: 直接落地 §4 #1.6 + §1 "分批下单 + Adaptive 算法本期 ship" 条款; 复用现有 80% 基础设施, KISS 不卷入新概念。

## 3. 单腿 vs 多腿路径分裂

| 场景 | 算法 | 实施路径 |
|---|---|---|
| **单腿** (LONG_CALL / LONG_PUT / SHORT_PUT_HEDGED 未来) ≥ 50 手 | **IBKR Adaptive Algorithm** (`order.algoStrategy="Adaptive"` + `algoParams=[TagValue("adaptivePriority", X)]`) | 调 `OrderClient.placeOrder` 一次, IBKR 内部拆单; 我们**不再批 batch_size 拆**, 信任 IBKR 算法 |
| **多腿** (BULL_CALL_SPREAD 2 腿 / IRON_CONDOR 4 腿等 BAG combo) ≥ 50 手 | **自实现拆腿挂单 + invariant 21 3 档** (patient → normal → urgent, BAG 禁 market 档) | 100 手 IRON_CONDOR → 拆 10 批 × 10 手, 每批 4 腿 BAG 同时挂; 每批走 entry_escalation 升级链 |
| **任何策略** < 50 手 | 整单, 不拆 (走现有 order_placer 路径) | 不动现有逻辑 |

**为什么单腿用 Adaptive 不自实现拆**: IBKR Adaptive Algorithm 是 IBKR Native Algorithm Server 行为, 它看 OPRA 实时数据 + 做市商行为自动决定拆策略 + 跟单, 比我们应用层 8s interval 启发式拆更智能。**单腿场景信任 IBKR**。

**为什么多腿不用 Adaptive**: IBKR Adaptive **不支持 BAG combo** (官方限制), 多腿必须自实现。

## 4. 5 个核心设计决策

### D1. `split_order_dispatcher.py` 接口

新文件 `services/split_order_dispatcher.py`:

```python
def place_split_order(
    contract: Contract,                # IBKR contract (单腿 STK/OPT 或 BAG combo)
    total_qty: int,                    # 总手数
    side: Literal["BUY", "SELL"],      # 入场 / 平仓
    batch_size: int = 10,              # 多腿场景每批手数 (单腿 Adaptive 忽略)
    interval_sec: int = 8,             # 多腿场景批间隔 (单腿忽略)
    mode: Literal["patient", "normal", "urgent"] = "patient",  # 起步 spread_ratio
    algo: Literal["auto", "adaptive", "combo"] = "auto",       # 强制路径
    opp_id: UUID | None = None,        # 关联 opportunity (BrokerOrder.opp_id 填)
) -> SplitOrderResult:
    """
    Returns:
        SplitOrderResult(
            batch_id: UUID,
            total_filled: int,
            avg_price: float,
            broker_orders: list[BrokerOrder],  # 所有 batch 的 BrokerOrder
            status: "FULL_FILL" | "PARTIAL_FILL" | "FAILED"
        )
    """
```

`algo="auto"` 路由 (默认):
- contract.secType in ("STK", "OPT") + 单腿 → algorithmic = "adaptive"
- contract.secType == "BAG" → algorithmic = "combo"
- 强制覆盖: `algo="adaptive"` / `algo="combo"`

### D2. 触发逻辑 — **per-symbol 流动性挂钩阈值** (不固定 50) + jitter + 三环境 config

**P0 修订 (期权实战)**: 50 手固定阈值不合理 — SPY weekly 50 手不算大 (单 OI 5k+), NVDA/TSLA weekly 50 手已显眼, PLTR/RIVN 30 手已扫光 best ask。**阈值必须查 `services/liquidity_gate.py` 算 per-symbol/per-OI 当前 bar**。

入口集成 (依赖 grep 现有 `OrderRequest` 字段名 — 实施 PR 前 grep 确认 `quantity` vs `total_qty`):

```python
# services/order_placer.py:place_entry_order() 顶部加:
from services.split_order_dispatcher import place_split_order
from services.execution_config import get_execution_config
from services.liquidity_gate import compute_split_threshold

config = get_execution_config()

# 阈值计算: per-symbol 流动性查 liquidity_gate, fallback config default
threshold = compute_split_threshold(
    contract=contract,
    fallback=config["execution"]["split_order"]["min_qty_default"],
)
# 例: SPY ATM strike OI=5000 → threshold=100; NVDA OI=2000 → 50; PLTR OI=400 → 20

if order_request.quantity >= threshold:  # grep OrderRequest 字段名
    return place_split_order(contract, total_qty=order_request.quantity, ...)
else:
    return _legacy_place_single_order(...)  # 原路径不动
```

同样集成在 `services/exit_order_placer.py:place_exit_order()`.

**`compute_split_threshold` 算法** (新加在 `liquidity_gate.py`):
```python
def compute_split_threshold(contract, fallback=50) -> int:
    """per-symbol 流动性挂钩阈值. 查最近 bar OI / volume / bid-ask spread."""
    # 单腿 STK/OPT: 阈值 = min(OI / 50, fallback * 2)  (高流动性容忍更大整单)
    # BAG combo: 阈值 = min(每腿 OI / 100, fallback * 1.5)
    # 数据拿不到 (盘外/新合约) → fallback (50)
```

**Config 文件 三环境分版本** `config/execution.{dev,uat,prod}.yml` (跟 .env.{env} 加载机制对齐, 不进 GitHub Secrets — 进 repo 即可无敏感):

```yaml
# config/execution.dev.yml (cloud-dev, 易触发测)
execution:
  split_order:
    min_qty_default: 10  # cloud-dev 容易触发拆单 paper 测
    default_batch_size: 5
    default_interval_sec: 3
    interval_jitter_sec: 1  # ±1s 随机防被盯单
    batch_size_jitter: 1    # batch_size ±1 随机
    combo_fill_timeout_sec: 30
    adaptive_priority_map:
      patient: Patient
      normal: Normal
      urgent: Urgent

# config/execution.uat.yml: min_qty_default=30, batch_size=8, interval=5
# config/execution.prod.yml: min_qty_default=50, batch_size=10, interval=8, jitter=2-5s
```

**Jitter 防做市商识破 (期权实战 + IB 共同提)**:
- `interval_sec` 实际 = `default_interval_sec ± random(0, interval_jitter_sec * 2)` (e.g. PROD 8 ± 0-4 = 6-10s 随机)
- `batch_size` 实际 = `default_batch_size ± random(0, batch_size_jitter * 2)` (e.g. PROD 10 ± 0-2 = 8-12 随机, 且 sum 仍 = total_qty)

阈值/jitter 改 config 不改代码 + 三环境隔离 + per-symbol 流动性挂钩。

### D3. 单腿 Adaptive 实现 (P0 修订: lmtPrice=mid + RTH gate + tif=DAY)

**P0 #2 修订 (期权实战)**: 我们传 `lmtPrice=mid+0.2*(ask-mid)` 跟 Adaptive 算法**冲突** — Adaptive 内部按 OPRA NBBO + 历史 fill 自决, 起步价我们传偏 patient 它可能立刻跨到 mid+0.5。**单腿 Adaptive 场景 lmtPrice=mid (不加 ratio), 让 Adaptive 全权**, 否则 double-pricing。

**P0 #3 修订 (Cloud + IB)**: Adaptive Algorithm **仅 RTH** (regular trading hours, IBKR 官方硬限制 + outsideRth=False)。盘前/盘后单走 Adaptive 会被 IBKR reject。**dispatcher 代码层加 RTH gate + 文档双保险**。

**P1 修订 (IB)**: Adaptive 必须 `tif="DAY"` (不支持 GTC); lmtPrice 远离 mid > 5% 会被 IBKR 算法 reject, 加 sanity clamp。

```python
def _place_single_leg_adaptive(contract, total_qty, side, mode, opp_id):
    # P0 RTH gate: Adaptive 仅 RTH, 盘前/盘后退化 LMT 4 档
    if not is_rth_now(contract.exchange):
        return _place_single_leg_lmt_escalation(contract, total_qty, side, mode, opp_id)

    if mode == "market":
        # market 档 Adaptive 不适用, 走 emergency_close (invariant 21 唯一豁免, R7 only)
        raise UnsupportedModeError("single-leg market 档不走 Adaptive, 应走 emergency_close R7 路径")

    # P0 #2: lmtPrice = mid (不加 ratio), 让 Adaptive 全权
    mid = get_current_mid(contract)
    lmt_price = mid

    # P1 sanity clamp: lmt 远离 mid > 5% 会被 IBKR Adaptive reject
    nbbo_spread = get_bid_ask_spread(contract)
    max_deviation = max(mid * 0.05, nbbo_spread * 2)
    if abs(lmt_price - mid) > max_deviation:
        lmt_price = mid  # clamp to mid 防 reject

    order = LimitOrder(
        action=side,
        totalQuantity=total_qty,
        lmtPrice=lmt_price,
        tif="DAY",        # P1 IB: Adaptive 不支持 GTC
        outsideRth=False, # P0 RTH gate (双保险, 即使错过 is_rth_now 检查 IBKR 也 reject 不裸跑)
    )
    order.algoStrategy = "Adaptive"
    adaptive_priority = config["execution"]["split_order"]["adaptive_priority_map"][mode]
    order.algoParams = [
        TagValue("adaptivePriority", adaptive_priority)  # Patient / Normal / Urgent
    ]

    # 通过 OrderClient 抽象 (北极星 §5 hook 必走)
    broker_order = order_client.place_order(
        contract, order, opp_id=opp_id, algo_strategy="Adaptive"
    )
    # 等 orderStatus FILLED (累计 IBKR 推) → ENTRY_FILLED handler 塞任务
    return broker_order
```

**关键点**:
- **`lmtPrice = mid` (不加 ratio)**, 让 Adaptive 全权决策跟单 + ±tick — 应用层不double-pricing
- **`tif="DAY"`** (Adaptive 不支持 GTC, IB 强约束)
- **`outsideRth=False`** + 应用层 `is_rth_now` 双保险 — Adaptive 仅 RTH
- 盘前/盘后单**自动退化 LMT 4 档 escalation** (`_place_single_leg_lmt_escalation` 复用 entry_escalation), Adaptive 不裸跑
- `adaptivePriority`: Patient (耐心等更好价) / Normal (折中) / Urgent (快成交牺牲价格)
- IBKR Adaptive 内部拆单 + 跟单 + 反挂, 我们应用层不再 batch_size 拆 (信任 IBKR)
- BrokerOrder.algo_strategy = "Adaptive" 标记审计

### D4. 多腿 BAG combo 4 档调价

```python
def _place_combo_legs_split(contract_bag, total_qty, side, batch_size, interval_sec, mode, opp_id):
    batch_id = uuid4()
    batches = total_qty // batch_size  # 100 / 10 = 10 batch
    remainder = total_qty % batch_size

    all_broker_orders = []
    cumulative_filled = 0

    for batch_idx in range(batches):
        batch_qty = batch_size
        result = _place_one_combo_batch(
            contract_bag, batch_qty, side, mode,
            batch_id, batch_idx, opp_id
        )
        all_broker_orders.extend(result.broker_orders)
        cumulative_filled += result.filled_qty

        if result.filled_qty < batch_qty:
            # 部分 fill — 升级 mode 重挂剩余
            remaining = batch_qty - result.filled_qty
            mode_next = upgrade_chain.next_mode(mode, secType="BAG")  # invariant 21 BAG 3 档
            if mode_next is None:
                # urgent 档已用尽 — 部分 fill ENTRY_FILLED handler 处理裸腿
                break
            retry_result = _place_one_combo_batch(contract_bag, remaining, side, mode_next, batch_id, batch_idx, opp_id)
            all_broker_orders.extend(retry_result.broker_orders)
            cumulative_filled += retry_result.filled_qty

        time.sleep(interval_sec)  # 防被盘单, 间隔 8s

    if remainder > 0:
        # 最后零头
        ...

    return SplitOrderResult(
        batch_id=batch_id, total_filled=cumulative_filled,
        avg_price=_compute_vwap(all_broker_orders),
        broker_orders=all_broker_orders,
        status="FULL_FILL" if cumulative_filled == total_qty else "PARTIAL_FILL"
    )

def _place_one_combo_batch(bag, batch_qty, side, mode, batch_id, batch_idx, opp_id):
    """单批 BAG combo 挂单 + 等成交, 复用 entry_escalation 4 档逻辑"""
    lmt_price = mid_based_lmt(bag, mode)
    bag_order = LimitOrder(action=side, totalQuantity=batch_qty, lmtPrice=lmt_price)

    broker_order = order_client.place_order(
        bag, bag_order, opp_id=opp_id,
        algo_strategy=f"COMBO_{mode.upper()}",  # COMBO_PATIENT / COMBO_NORMAL / COMBO_URGENT
        split_order_batch_id=batch_id,
        batch_intent_json={
            "total_qty": batch_qty * len(self.batches),
            "batch_size": batch_qty,
            "batch_idx": batch_idx,
            "interval_sec": 8,
            "mode": mode
        }
    )
    # 等 orderStatus 累计 fill (timeout 30s)
    ...
    return BatchResult(filled_qty=X, broker_orders=[broker_order])
```

**关键点**:
- 拆腿 = 拆 batch (不是 leg), 每批 N 手的 4 腿 BAG combo
- 每批走 `entry_escalation` 4 档升级链 (patient → normal → urgent; BAG 禁 market 档 invariant 21)
- `interval_sec=8` 批间隔防被盯单
- BrokerOrder.algo_strategy = "COMBO_{mode}" + split_order_batch_id + batch_intent_json

### D5. 部分 fill + 裸腿处理 (P0 修订: combo-level 累计 + 90s 硬 timeout 应急 MKT)

**P0 修订 (IB)**: BAG combo 的 IBKR `orderStatus.filled` 是 **combo 单位** (e.g. 60 个 combo = 4×60 leg), 不会出现腿间不齐 — combo 通常**原子撮合**。**真正的裸腿风险**仅来自 combo 被 IBKR 拆 multiple child orders 撮合时 (rare)。**所以按 `parentOrderId` 累计 combo filled, 不按 leg 累**, 简化逻辑。

**P0 修订 (期权实战 升 P0)**: D5 partial leg 5 min review 暴露过长 — IRON_CONDOR 净裸 short call/put 在事件窗口 5 min 可亏 30%+。**应用层加 90s 硬 timeout 应急 MKT 平已成交腿** (作为 invariant 21 R7 类豁免, 修复故障不算违反"信任 LLM" 因为不是策略方向决策)。

**处理**:
1. 现有 `services/broker_adapter.py` 的 `BrokerOrderEvent` 跟踪 combo orderStatus (已 ship)
2. dispatcher `_place_one_combo_batch` 按 **`parentOrderId` 累计 `orderStatus.filled`** (combo-level), 等 `combo_fill_timeout_sec` (config, default 30, patient mode 90) 内不全 fill → 返 `PartialBatchResult(filled_combo_qty=X)`
3. 升级 mode 重挂剩余 combo (next mode patient → normal → urgent), **重挂前必 refresh quote** — `lmt_price = mid_based_lmt(contract, mode_next, refresh_quote=True)`, 防 mid 缓存 stale 导致挂单方向错 (P1 SE 提)
4. urgent 仍部分 fill → 不再升 market 档 (invariant 21), 进 step 5 应急
5. **应急 90s 硬 timeout 应急 MKT 平已成交腿** (P0 期权实战):
   - 检测 partial leg 状态 (BrokerOrderEvent 显示某些 leg 已成交某些未成交)
   - 计时 90s, 若 90s 内 cumulative orderStatus 未达 total_combo_qty
   - **dispatcher 自动塞 emergency_close R7 任务** (现有 `services/emergency_close.py` 已 ship invariant 21 唯一豁免) → 立即 MKT 平已成交腿
   - 不等 Agent 2 5 min review (5 min 太长, 期权事件窗口可能亏 30%+)
   - 同时塞 audit + 告警 Agent 2 review 紧急唤醒看后续决策
6. **R7 应急 MKT 不算违反"信任 LLM"哲学** — 因为修复 dispatcher 故障不是策略方向决策, 跟客户接受期权亏权利金 (策略层风险) 不同; partial leg 是基础设施层故障, 应用层应自动恢复
7. SplitOrderResult.status: FULL_FILL / PARTIAL_FILL / EMERGENCY_CLOSED (90s timeout 触发) / FAILED 4 值

**关键决策对照**:

| 场景 | 处理 | 哲学 |
|---|---|---|
| Agent 2 决策持仓 → HOLD/CLOSE/ADJUST | LLM 自主, 5 min review 节奏 | 信任 LLM (策略方向) |
| Agent 2 决策入场 → 风险门 reject | 重跑一次 仍违 失败单 | 信任 LLM (策略方向) |
| dispatcher partial leg 90s timeout | 应用层自动 R7 emergency MKT | 修复故障 (基础设施层) |
| 客户期权浮亏权利金 | 让 LLM 自决平仓时机 | 信任 LLM (策略方向) |

## 5. 测试策略

| 层 | 测什么 | 文件 |
|---|---|---|
| **Unit** | dispatcher 路由逻辑 (auto → adaptive vs combo); 拆 batch 数量 + batch_intent_json 正确; 升级 mode 链 | `tests/services/test_split_order_dispatcher.py` |
| **Unit (config)** | min_qty 阈值 = 50 (default) / 改 config 生效 | `tests/services/test_execution_config_split.py` |
| **集成** | mock IBKR 单腿 Adaptive (algoStrategy 字段填对) + 多腿 BAG 拆腿 + partial fill 升级 mode 重挂 | `tests/integration/test_split_order_integration.py` |
| **e2e (paper)** | **真盘**: 100 手 LONG_CALL Adaptive (1 次) + 100 手 IRON_CONDOR 拆腿 (1 次) | finding F-2026-05-06-SPLIT-ORDER-PAPER-VALIDATION |
| **性能** | 100 手拆 10 批 × 10 手 + interval_sec=8 ≈ 80s 全部成交; 单腿 Adaptive ~30s | 同上 finding |

## 6. 风险 (P0 已修, 仅余 P1-P2)

1. **IBKR Adaptive 行为不可预测** — IBKR 算法服务器自决跟单, 滑点报告依赖 IBKR 给的成交价。**Mitigation**: BrokerOrder.algo_strategy="Adaptive" 标记 + slippage_audit.py 跟踪 (已 ship line 83); paper 测确认实际跟我们 patient/normal/urgent 期望表现一致 (期权实战 review)
2. **partial leg 90s timeout MKT 应急** (D5 P0 修后) — 应急 MKT 仍有滑点 (但比 5 min 暴露好), paper 测调 90s 是否合理; finding F-2026-05-06-PARTIAL-LEG-TIMEOUT-PAPER-VALIDATION 跟踪
3. **interval_sec jitter ±2-5s** — paper 测验做市商真否被规律 8s 识破, 决定 jitter 范围合理值
4. **Per-symbol 阈值 compute_split_threshold 算法** — 公式 `min(OI / 50, fallback * 2)` 是经验值, paper 实测调
5. **跟现有 order_placer 路径回归** — 集成入口 `if quantity >= threshold` 必须不影响 threshold 以下单 (现有逻辑); 测试覆盖
6. **Phase B alembic 0029 部署 hard gate** — Phase C 上 PROD 前必先 Phase B 0029 在三环境 alembic upgrade head 完, 否则 D5 字段 algo_strategy/split_order_batch_id/batch_intent_json 写入报错。**部署文档明示 + deploy.sh 加 alembic revision check** (Cloud P1)

## 7. 五专家评审待办

| 专家 | review 重点 |
|---|---|
| 软件工程 | dispatcher 模块边界 / 跟现有 order_placer + exit_order_placer 集成路径 / partial fill 状态机 / config 驱动 vs hardcode |
| AI Mem | 跟北极星 §1.6 + invariant 21 + Phase B BrokerOrder 字段一致; algo_strategy 5 值枚举跟 Phase B D6 spec 一致 |
| Cloud | Adaptive Algorithm 在云端 IB Gateway 跑稳 (网络抖动时 algoParams 是否传错); 部署不需 alembic (纯代码 PR) |
| 期权实战 | 50 手阈值合理? 8s interval 实战正确? Adaptive (Patient/Normal/Urgent) 跟我们 4 档 spread_ratio 映射对吗? IRON_CONDOR 100 手拆 10 批是否被做市商识破 |
| IB/TWS | Adaptive Algorithm IBKR API 接口正确 (algoParams TagValue 用法 / outsideRth 兼容) / BAG combo 拆腿后 BrokerOrderEvent 跟踪每腿 / IBKR 实时成交回调跟 SplitOrderResult 累计逻辑 |

任一反对必修, 不走多数票。

## 8. Plan

按 superpowers:writing-plans 框架写 implementation plan, ship 时间预估:
- **Phase 1** (1 hr): split_order_dispatcher.py 实现 + unit + config 化 min_qty
- **Phase 2** (1 hr): order_placer.py + exit_order_placer.py 集成入口 + 跟现有 entry_escalation 复用
- **Phase 3** (30 min): BrokerOrder 字段填值 (依赖 Phase B alembic 0029 ship 完); slippage_audit 标记
- **Phase 4** (1-2 hr paper, 美股盘中): 真测 100 手 LONG_CALL Adaptive + IRON_CONDOR 拆腿; 调 interval_sec 实测最优值; 验证 partial fill → mode 升级路径

**总估 4-5 hr 工程**. **依赖 Phase B 的 BrokerOrder alembic 0029 ship 完** (algo_strategy / split_order_batch_id / batch_intent_json 列), 否则 D5 字段填值无 schema 支持。

## 9. 关联文档

- 北极星 memory `project_vision_and_north_star.md` §1 (分批下单本期 ship 条款) + §4 #1.6 (本 spec)
- wiki `architecture-walkthrough.md` (4 Pool/Client + OrderClient)
- Phase B spec `2026-05-06-db-schema-task-queue-design.md` D6 (BrokerOrder 加列)
- 现有代码: `services/{entry_escalation,exit_order_placer,order_placer,pricing,upgrade_chain,emergency_close,liquidity_gate,broker_adapter}.py`
- invariant 21 (CLAUDE.md §5)

---

**END OF DESIGN SPEC**
