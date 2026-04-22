# Spec 镜像 — Risk Gate V2 设计 (Phase 11 Step D)

> **镜像来源**：项目 repo `docs/superpowers/specs/2026-04-23-risk-gate-v2-design.md`
> **镜像日期**：2026-04-23
> **状态**: Step D 实装完成
> **对应决策记录（客户友好提炼）**：[`../decisions/2026-04-23-step-d-r7-deferred.md`](../decisions/2026-04-23-step-d-r7-deferred.md)
> **关联前置 spec**：[`2026-04-21-agentic-trading-redesign-design.md`](2026-04-21-agentic-trading-redesign-design.md)
> **脱敏说明**：已按全局 §18 脱敏红线核查（无 API key / 账户号 / 具体金额 / VM IP / 实盘参数具体值）。风控阈值只作示例，真实值在项目 `config/risk_limits.yml`。原始 commit SHA / 内部讨论原始 Q&A 不镜像。

---

## 原始 Spec 内容

> 以下为项目 repo 内 spec 原文（已按 public 风格写作）。

**Spec 日期**: 2026-04-23
**前置 plan**: 见关联前置 spec

---

## 1. 目标

提供 7 条**确定性硬规则**（非 AI）作为 Agent 2 决策的最后一道闸门。规则按 spec §4 定义，所有阈值通过 `config/risk_limits.yml` 调参，代码不变。

设计原则：
- **每条规则一个独立函数**，签名统一 `(decision, bundle, account_state, config) -> RuleResult`
- **顺序评估，first-reject 短路**（除 R7 例外）
- **R7 唯一 override 规则**：不 reject，强制改写 decision 为 FullCloseDecision 防止 assignment

---

## 2. 7 条规则总览

| ID | 名称 | 类型 | 数据源 | Skip 模式 |
|---|---|---|---|---|
| R1 | 单笔最大仓位 (账户 30% AND 5 日均量 × 20%) | reject | account_state | action-driven (PARTIAL_CLOSE only) |
| R2 | 总仓位上限 (long 60% / short 70% / cash buffer 20%) | reject | account_state | action-driven (PARTIAL_CLOSE only) |
| R3 | 流动性分层 (spread% 4 档：normal/urgent/market/reject) | reject + tier metadata | bundle.liquidity | bundle-driven (skip if liquidity is None) |
| R4 | sanity check (entry_lmt_price ±3σ vs 最近 15min mid) | reject | bundle.bars + account_state.entry_lmt_price (optional) | bundle-driven + entry_lmt_price-driven |
| R5 | 频率限制 (1h ≤ 3, 开盘前 30min ≤ 1) | reject | account_state | action-driven (HOLD only) |
| R6 | 熔断 (连续 N 次亏损调整暂停) | reject + ERROR log | account_state.last_two_decisions_all_loss (pre-computed) | action-driven (HOLD only) |
| R7 | Assignment 规避 (距到期 ≤ N 天 + short + ITM → 强制 FULL_CLOSE) | **override** | account_state.assignment_risk_short_legs (pre-computed) | bundle-driven (skip if position is None) |

---

## 3. evaluate() 编排（Q1 决策）

**R7 放最前 + override 后用新 decision 重跑 R1-R6**：

```python
def evaluate(decision, bundle, account_state) -> RiskGateResult:
    config = load_risk_config()
    _validate_account_state_schema(account_state)  # Q3
    _validate_config_invariants(config)             # Min-4

    all_results = []
    final_decision = decision

    # R7 first
    r7_result = check_rule_7_assignment_avoidance(final_decision, bundle, account_state, config)
    all_results.append(r7_result)
    if r7_result.override_decision is not None:
        final_decision = r7_result.override_decision

    # Then R1-R6 with possibly-overridden decision
    for rule_func in [R1, R2, R3, R4, R5, R6]:
        result = rule_func(final_decision, bundle, account_state, config)
        all_results.append(result)
        if not result.passed:
            return RiskGateResult(
                passed=False, first_reject=result,
                all_results=all_results, final_decision=final_decision,
            )

    return RiskGateResult(passed=True, first_reject=None, all_results=all_results, final_decision=final_decision)
```

**关键不变量**:
1. R7 始终先跑（因为 R7 可能 override，让 R1/R2 拦截 PARTIAL_CLOSE 错地拦下原本应被改成 FULL_CLOSE 的决策）
2. R7 永远 `passed=True`（不 reject，只 override）
3. R1-R6 任一 reject 立即短路返回
4. 短路时 `final_decision` 仍是（可能被 R7 覆盖后的）值

---

## 4. 数据契约

### 4.1 account_state schema（Q3 决策）

`evaluate()` 入口校验 `_REQUIRED_ACCOUNT_STATE_KEYS_BY_RULE` 列出的所有 key 必须存在。任一缺失 → ValueError 一次性列全。

```python
_REQUIRED_ACCOUNT_STATE_KEYS_BY_RULE = {
    "R1": ("account_cash", "position_notional_usd", "five_day_avg_volume"),
    "R2": ("account_cash", "total_long_premium", "total_short_margin", "used_cash_pct"),
    # R3 reads bundle only
    # R4 reads bundle.bars + optional entry_lmt_price (R4 uses .get(), key may absent)
    "R5": ("adjustments_last_hour", "is_pre_open_30min"),
    "R6": ("last_two_decisions_all_loss",),
    "R7": ("assignment_risk_short_legs",),
}
```

### 4.2 config 不变量校验（Min-4）

`evaluate()` 入口校验：`config["rule_3_liquidity_tiers"]["market_spread_max_pct"] == config["rule_3_liquidity_tiers"]["reject_spread_min_pct"]`，否则 tier 模型有 gap → ValueError。

### 4.3 RuleResult / RiskGateResult dataclass

```python
@dataclass
class RuleResult:
    rule_id: str            # "R1" .. "R7"
    passed: bool
    reason_zh: str          # 中文 audit 字符串, 必含具体数字
    override_decision: Agent2Decision | None = None  # R7 才用

@dataclass
class RiskGateResult:
    passed: bool
    first_reject: RuleResult | None
    all_results: list[RuleResult]   # 顺序所有规则结果, 审计用
    final_decision: Agent2Decision  # R7 override 后的实际 decision
```

---

## 5. R1/R2 当前是占位（Q2 决策）

**R1/R2 当前不会真正 reject 任何 Agent 2 决策**。原因：

- R1（单笔最大仓位）和 R2（总仓位上限）本意检查"开仓/加仓"
- Agent 2 的 4 个 action 没有"开仓"
- 当前 skip 谓词 `if action != "PARTIAL_CLOSE": skip`
- PARTIAL_CLOSE 是缩仓，不会突破最大仓位

**保留 R1/R2 + 加 TODO 注释**，等 Phase 12 Agent 2 union 加 `OPEN_POSITION` action 后改 skip 谓词为 `if action not in ("OPEN_POSITION", "PARTIAL_CLOSE"): skip`，让 R1/R2 真起作用。

最终目标：让 AI 完全自主决定开仓/加注，全链路风控覆盖。

---

## 6. R6 reject → ERROR log（Q4 决策）

R6 reject reason 写"连续 N 次亏损调整 → 熔断暂停, 需人工确认"。"需人工确认"是状态描述非行为声明。

**D9 实装**: orchestrator `_finalize_trace` 检测 `first_reject.rule_id == "R6"` → `logger.error(...)`。

**Step E 兑现**: 实装 `services/alert_sink.py` + 接入"R6 reject → 自动告警邮件/Slack"。

---

## 7. R7 阈值 = 2 天 + Step E 配套（决策五）

期权实战专家 review verdict：5 天数字本身合理（业界标准 TastyTrade 5 DTE rule + TOS DTE 5 默认提醒），但**单改阈值不够**——必须配套 R7 紧急平仓路径局部豁免 invariant 21。

**Step D 决定**：保 plan 原值 2 天，(a) 改 5 天 + (b) MKT 豁免**配套留 Step E P0 一起做**。

### Step E P0 必接条目（task_plan.md 已落地）

- (a) `config/risk_limits.yml` `rule_7_assignment_avoidance.days_to_expiry_threshold: 2 → 5`
- (b) R7 紧急平仓路径局部豁免 invariant 21：BAG combo 在 R7 触发的紧急平仓时，升级链 LMT 多档失败后允许走 MKT 兜底（仅此紧急路径，不全面推翻）

### Step E P1 实战派建议

- (c) 标的流动性分级表（20 行 YAML：SPY/QQQ/AAPL/TSLA/NVDA = high-liquidity，其他 = low-liquidity）+ entry gate 一条规则（中小盘 + DTE < 21 拦截）

### Phase 12 P1（远景）

- (d) TIME_STOP 默认距到期 5 天（intake 层默认值）
- (e) Earnings 事件策略豁免 R7
- (f) Agent 2 union 加 `OPEN_POSITION` action（承接 Q2 R1/R2 真正起作用的前提）

---

## 8. orchestrator 集成

### 8.1 `_build_account_state` (Step E 接口契约)

D9 暂置 placeholder 值，每行带 `# TODO Step E:` 注释指真实数据源。10 个 key 的来源：

| Key | 数据源 (Step E 实装) | 服务于 |
|---|---|---|
| `account_cash` | `ibkr_client.get_account_value("NetLiquidation")` | R1, R2 |
| `position_notional_usd` | bundle.position.legs 求和 | R1 |
| `five_day_avg_volume` | bundle.bars 或外部 | R1 |
| `total_long_premium` | IBKR 账户值聚合 | R2 |
| `total_short_margin` | IBKR 账户值聚合 | R2 |
| `used_cash_pct` | IBKR 账户值 | R2 |
| `adjustments_last_hour` | DB query agent2_decisions 表 | R5 |
| `is_pre_open_30min` | `datetime.now(ET)` vs 09:30 ET | R5 |
| `last_two_decisions_all_loss` | 查最近 2 次 executed decisions PnL | R6 |
| `assignment_risk_short_legs` | bundle.position.legs filter (is_short=True) + IBKR contract details (DTE + 标的价 vs strike ITM 判定) + threshold | R7 |

`entry_lmt_price` 是 R4 可选 key，仅入场下单时由 orchestrator 注入；R4 用 `.get()` safe default。

### 8.2 `_finalize_trace` 5 return sites 全部接入

`ai_review_one_position` 的 5 个 return site（cache hit / Haiku EXECUTE / Haiku HOLD_AND_ALERT / Sonnet EXECUTE / Sonnet HOLD_AND_ALERT）全部经 `_finalize_trace`。

**关键**: 缓存命中也跑 risk gate（cached LLM output 仍是 LLM output, risk depends on current bundle data not cache-time data）。HOLD_AND_ALERT 降级也跑（R7 仍可能 override）。

### 8.3 Agent2DecisionTrace 加新字段

```python
risk_gate_result: RiskGateResult | None = None
```

默认 `None` 是为兼容直接构造 trace 的测试 fixture（生产路径全部经 `_finalize_trace` 设值）。

---

## 9. 跨规则模式锁定（review 累积）

| 模式 | 来源 | 内容 |
|---|---|---|
| N-1 | D3 polish | 所有 reason_zh 阈值从 params 插值，绝不硬编码 |
| M-4 | D2 review | 测试 fixture 干净，禁止 thinking-out-loud 注释 |
| D4 Min-1 | D4 review | tier 分支按"最有 consequence 的先检查"排序 |
| Bundle/action skip 双轨 | D4 lock-in | bundle-using 规则 → bundle-driven skip；非 bundle 规则 → action-driven skip |
| D6 Min-1 | D6 review | 测试 substring assert 用区分度强字符串，不裸用 "1"/"2"/"3" |
| D6 Min-2 | D6 review | 测试用 module-level functions, 不用 class wrapper |
| §5 trust internal | D9 I-1 revert | 不为不可能场景加 defensive guard，签名声明的契约由 caller 保证 |

---

## 10. 测试覆盖

49 个新测试（Step D）：

| 文件 | 测试数 | 覆盖 |
|---|---|---|
| test_risk_gate_v2_base.py | 2 | config load + dataclass 默认值 |
| test_risk_gate_v2_rule_1.py | 4 | pass + reject account_pct + reject volume + skip HOLD |
| test_risk_gate_v2_rule_2.py | 5 | pass + reject 3 个分支 + skip HOLD |
| test_risk_gate_v2_rule_3.py | 6 | 4 tier + skip + 20% boundary |
| test_risk_gate_v2_rule_4.py | 6 | pass within / reject below / reject above / skip 2 路径 / boundary |
| test_risk_gate_v2_rule_5.py | 6 | pass + reject 2 路径 + skip + 2 boundary |
| test_risk_gate_v2_rule_6.py | 4 | pass + reject + skip + non-consecutive scenario doc |
| test_risk_gate_v2_rule_7.py | 4 | skip no_position + pass no_risk + override + override preserves next_check_minutes |
| test_risk_gate_v2_evaluate.py | 8 | all_pass + R1 short-circuit + R7 override + override + R3 reject + 顺序 + schema 缺 key + config invariant + HOLD pass-through |
| test_agent2_orchestrator_risk_gate.py | 4 | Haiku EXECUTE + R6 ERROR log + R7 override + cache hit + R3 reject |

**测试基线**：Phase 11 targeted 144 → 193 (+49)。Step C 47 agent2 测试**不回归**。

---

## 11. 风险与已知限制

1. **R1/R2 占位**: 当前 Agent 2 决策环里不真起作用。Phase 12 (f) 加 OPEN_POSITION 后兑现。
2. **R7 阈值 2 天**: 流动性枯竭场景下 BAG combo 卡死风险。Step E P0 (a)+(b) 配套兑现。
3. **`_build_account_state` placeholder**: 10 个 key 全 placeholder（permissive defaults 偏向"通过"），生产 risk gate 在 Step E 真实数据接入前实质 disabled。Step E 必须先于 deploy 完成数据接入。
4. **invariant 21 (BAG MKT 禁) vs 16 (升级链必须 MKT 兜底)** 在 BAG 紧急场景打架：Step E (b) 局部豁免解决，仅 R7 紧急路径开 MKT 后门。

---

## 12. 与现有 invariants 的关系

| invariant | 受影响 | 关系 |
|---|---|---|
| §5 第 10 (Risk Gate 是确定性代码非 AI) | ✓ 兑现 | 7 条规则全是 dataclass + 函数, 无 LLM 调用 |
| §5 第 16 (升级链 LMT → 追价 → MKT 必须走到底) | 间接 | R7 触发的 FULL_CLOSE 走升级链 |
| §5 第 18 (Worker 24h 持续决策, 部署后不停机) | ✓ 兼容 | risk_gate 是 evaluate() 同步调用, 不影响循环 |
| §5 第 20 (Agent 2 两个工作点) | 间接 | risk_gate 工作点 B (持仓决策) 全跑; 工作点 A (首次入场) 走 Step E 入场 entry gate |
| §5 第 21 (BAG 不许 MKT) | **冲突** | Step E (b) 局部豁免解决 |
