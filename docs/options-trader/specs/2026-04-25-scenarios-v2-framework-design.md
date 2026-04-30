# Scenarios v2 完整框架 — Agent 2 新范式 agentic 测试体系

> **日期**：2026-04-25
> **状态**：Approved（brainstorming 2026-04-25 下午完成，五专家全员 approve with P0 addressed，13 组 P0 + 精选 P1 全部吸收）
> **前置**：`2026-04-25-step-f-execution-wiring-design.md`（L2/L3 断执行层依赖 Step F ship）
> **后置**：`scenarios v2 Phase 2`（Phase 12+，news collector 真实接入后扩展 REVIEW_NEWS_* 类）

---

## 1. 目的

1. **取代旧 intake-only scenarios**：`tests/fixtures/scenarios/scenarios_batch_{1,2,3}.yml` 的 `expect.intake.take_profit_rule_zh Field(min_length=1)` 与 Agent 2 新范式 invariant 10/20（TP/SL 动态决策）冲突，不可复用
2. **验 Agent 2 agentic 决策全链路**：10 维 bundle → 4 种 action → 执行层改单 → agent2_decisions 持久化 的闭环
3. **CI 回归门**：改 `src/.../agent2/**` 或 `tests/fixtures/scenarios_v2/**` 触发 PreToolUse hook + CI scenario count gate
4. **News 维度接口预留**：scenarios YAML schema 留 `bundle.news: null | NewsSnapshot` 字段，未来 collector 上线不改 schema

## 2. 非目的

- ❌ 真接入 news collector（等 Phase 12+）
- ❌ 旧 `scenarios_batch_*.yml` 迁移（冻结保留独立路径，见 §11）
- ❌ Gamma scalping / skew trade / order flow 场景（Phase 2 扩展，见 §14 技术债）

## 3. 架构

```
tests/fixtures/scenarios_v2/
    ├── entry_decision/                  (L1 + L2, 10 条)
    ├── review_hold/                     (L1 + L2)
    ├── review_partial_close_long_vol/   (L1 + L2)   ← 按 vol_stance 拆
    ├── review_partial_close_short_vol/  (L1 + L2)
    ├── review_full_close_long_vol/
    ├── review_full_close_short_vol/
    ├── review_adjust_stop_long_vol/
    ├── review_adjust_stop_short_vol/
    ├── risk_override_r7/
    ├── low_confidence_escalation/
    ├── hold_and_alert/
    ├── dte_terminal_handling/           (0DTE / DTE ≤ 1, 独立分类)  ← 新增 P1 吸收
    └── lifecycle_replay/                (L3 only, 分层抽样)

scripts/run_scenarios_v2.py       # pytest-based runner
scripts/check_scenario_counts.py  # CI gate (§8)
src/options_event_trader/
    ├── scenarios/types.py        # Action / Category / VolStance 单一真相 (吸收 CC-P0-2)
    └── scenarios/schema.py       # Pydantic discriminated union by layer (吸收 SWE-P1-7)
.github/workflows/
    └── scenarios_v2_paper_nightly.yml    # L3 nightly
```

## 4. YAML Schema（Pydantic discriminated union by layer）

### 4.1 共享字段
```yaml
id: AG2-L1-ENTRY-001                     # 格式: AG2-{L1|L2|L3}-{CATEGORY}-{NNN}
layer: L1 | L2 | L3
category: ENTRY_DECISION | REVIEW_HOLD | REVIEW_PARTIAL_CLOSE | REVIEW_FULL_CLOSE
        | REVIEW_ADJUST_STOP | RISK_OVERRIDE_R7 | LOW_CONFIDENCE_ESCALATION
        | HOLD_AND_ALERT | DTE_TERMINAL_HANDLING | LIFECYCLE_REPLAY
vol_stance: LONG_VOL | SHORT_VOL | NEUTRAL     # 吸收 Opt-P0-4
description: string                            # > 20 chars
bundle:
  position: null | PositionSnapshot            # 工作点 A null
  underlying:
    symbol: str
    last / bid / ask / day_change_pct / day_volume: ...
    next_earnings_date: null | "YYYY-MM-DD"    # 吸收 Opt-P0-5
    has_earnings_within_dte: bool
  greeks: null | GreeksSnapshot
  bars:
    bars_5m / bars_1h / bars_1d: list[{t,o,h,l,c,v}]
  indicators: rsi_14 / macd_signal / bollinger_position / sma_20 / sma_50 / atr_14 / volume_ratio_20d
  liquidity: null | {bid_size / ask_size / open_interest / day_volume / spread_pct}
  market_context: {spy_change_pct / vix / vix_change_pct / sector_etf_change_pct}
  news: null | NewsSnapshot                    # 本版永远 null，接口预留
  unusual_activity: null | {nearby_strikes_volume_spike / put_call_ratio}
  user_intent: {raw_input_text / symbol / direction / preferred_strategies / max_risk_dollars}
  ages_sec: dict[str, int]                     # 每个 top-level 字段对应秒差

# PositionSnapshot.legs[i]:
#   con_id / quantity / avg_cost / is_short
#   days_to_expiry: int                        # 吸收 Opt-P0-5，每 leg 必填
#   strike / right / expiry (YYYYMMDD)
```

### 4.2 L1 Schema（纯决策层断言，禁 `expect_execution` + `timeline`）
```yaml
expect:
  decision:
    action: [PARTIAL_CLOSE, ADJUST_STOP]       # 允许集合（OR）
    confidence_min: 0.5                        # 0-1
    quantity_pct_allowed: [25, 33, 50, 67, 100]  # 吸收 Opt-P1-1 收敛 5 档
    reasoning_must_not_include: ["IV 降必平"]  # 黑名单（吸收 SWE-P0-4）
  tier_used: any | haiku | sonnet | cache
  risk_gate_overridden: bool
  emergency_close_directive_present: bool | null
```

### 4.3 L2 Schema（`expect_execution` 必填，禁 `timeline`）
```yaml
expect:
  ... (同 L1)
expect_execution:
  if_action=PARTIAL_CLOSE:
    - execute_exit_escalation_called: true
    - quantity_override_range: [3, 7]          # 具体范围（10 张 × 25-67%）
    - liquidity_pre_check_passed: bool         # 吸收 Opt-P0-2
    - bag_per_leg_compensation_invoked: bool   # BAG 2 腿不对称补偿（IB-P0-5）
  if_action=ADJUST_STOP:
    - position_adjustments_row_inserted: true
    - old_value_captured: not_null
    - superseded_at_on_prior_row: not_null
  agent2_decisions_row:
    executed: true
    bundle_id_unique_enforced: true            # 重放幂等（SWE-P0-2）
```

### 4.4 L3 Schema（`timeline` 必填，`expect_execution` 可选每 tick）
```yaml
timeline:
  - t_offset_min: 0
    bundle: {... entry 状态}
    expect:
      decision: {action: [ENTRY_DECISION_TYPE], ...}
      (L3 入场 decision)
  - t_offset_min: 5
    bundle: {... vix_change +35%, greeks theta decayed 2%}
    expect:
      decision: {action: [PARTIAL_CLOSE], quantity_pct_allowed: [33, 50]}
  - t_offset_min: 10
    bundle: {... post-partial_close}
    expect:
      decision: {action: [HOLD]}
  ... (N 个 tick)
final_state:
  position_status: CLOSED | OPEN
  realized_pnl_range: [-500, +2000]            # 最终 PnL 区间断言
  decision_chain_length_range: [3, 8]          # 总决策次数
stratification_tag: LOSING_POSITION | TIME_STOP_TRIGGERED | R7_OVERRIDE | EARNINGS_IV_CRUSH_FAILURE | PROFITABLE  # 吸收 Opt-P0-3
```

### 4.5 Pydantic discriminated union（吸收 SWE-P1-7）
```python
ScenarioL1 | ScenarioL2 | ScenarioL3 = discriminator("layer")
# L1 类 禁用 expect_execution / timeline
# L2 类 必须有 expect_execution, 禁用 timeline
# L3 类 必须有 timeline + final_state + stratification_tag
```

## 5. 10 大类覆盖矩阵（吸收 Opt-P0-4 + P1 新增 DTE_TERMINAL_HANDLING）

| 大类 | 描述 | L1 下限 | L2 下限 | L3 下限 |
|---|---|---|---|---|
| `ENTRY_DECISION` | 入场决策（10 维 → 10 种 strategy） | 8 | 2 | 0 |
| `REVIEW_HOLD` | 稳定持仓 → HOLD | 3 | 1 | 0 |
| `REVIEW_PARTIAL_CLOSE_LONG_VOL` | long debit spread IV↓ 不利 → PARTIAL 锁利 | 4 | 1 | 0 |
| `REVIEW_PARTIAL_CLOSE_SHORT_VOL` | short credit IV↓ 利好 → HOLD / PARTIAL 理由反向 | 4 | 1 | 0 |
| `REVIEW_FULL_CLOSE_LONG_VOL` | TP 达成 / theta 衰减 | 3 | 1 | 0 |
| `REVIEW_FULL_CLOSE_SHORT_VOL` | 反向触发 | 3 | 1 | 0 |
| `REVIEW_ADJUST_STOP_LONG_VOL` | 浮盈突破 → 提升 stop | 3 | 1 | 0 |
| `REVIEW_ADJUST_STOP_SHORT_VOL` | delta 扩大 → 放宽 | 3 | 1 | 0 |
| `RISK_OVERRIDE_R7` | short + ITM + DTE ≤ 5 → 强制 FULL_CLOSE override | 3 | 1 | 0 |
| `LOW_CONFIDENCE_ESCALATION` | Haiku 0.5 → 升 Sonnet → execute | 3 | 0 | 0 |
| `HOLD_AND_ALERT` | 双 tier 低 conf → 降级 HOLD | 3 | 1 | 0 |
| `DTE_TERMINAL_HANDLING` | 0DTE / DTE ≤ 1 特殊：pin risk / gamma 爆炸 / 1500ET assignment 规避 | 4 | 1 | 0 |
| `LIFECYCLE_REPLAY` | 入场→N review→平仓完整剧本（分层抽样） | 0 | 0 | 5-10 |

**总下限**：L1 ≥ 44 / L2 ≥ 12 / L3 ≥ 5。**目标**：L1 ≥ 55 / L2 ≥ 20 / L3 ≥ 8。

**每大类 gate**（吸收 CC-P0-4 + SWE-P1-3）：**L1 ≥ 3 + L2 ≥ 1**（`LIFECYCLE_REPLAY` 和 `LOW_CONFIDENCE_ESCALATION` 豁免）。CI 的 `check_scenario_counts.py` 强制断言矩阵下限。

## 6. LIFECYCLE_REPLAY 分层抽样（吸收 Opt-P0-3）

L3 强制 stratification：
- **≥ 30% 最终亏损**（stratification_tag=LOSING_POSITION）
- **≥ 20% time_stop 触发**（TIME_STOP_TRIGGERED）
- **≥ 10% R7 override**（R7_OVERRIDE）
- **≥ 1 次 earnings IV crush 失败**（EARNINGS_IV_CRUSH_FAILURE）
- 其余可为 PROFITABLE

禁止只 replay 盈利持仓（prompt few-shot calibration 过度乐观化风险）。

`check_scenario_counts.py` 加 stratification 断言。

## 7. Runner 设计

### 7.1 `scripts/run_scenarios_v2.py`

pytest-based，YAML → 每条 scenario 一个 pytest case（pytest parametrize）。

```python
# 伪代码
@pytest.fixture
def mock_anthropic(monkeypatch): ...
@pytest.fixture
def mock_ibkr(): ...
@pytest.fixture
def e2e_db(): ...

@pytest.mark.parametrize("scenario", load_all_scenarios("tests/fixtures/scenarios_v2/"))
def test_scenario(scenario, mock_anthropic, mock_ibkr, e2e_db):
    if scenario.layer == "L1":
        run_l1(scenario, mock_anthropic, mock_ibkr=InlineMock(), mock_db=InlineMock())
    elif scenario.layer == "L2":
        run_l2(scenario, mock_anthropic, mock_ibkr=MockIBKRClient(), db=e2e_db)
    elif scenario.layer == "L3":
        pytest.skip_unless_paper_env()
        run_l3(scenario, real_anthropic=ClaudeClient(), real_ibkr=IBKRClient(client_id=29), db=e2e_db)
```

### 7.2 三层执行环境

| Layer | anthropic | ibkr | db | 秒级 | 何处跑 |
|---|---|---|---|---|---|
| **L1** | mock（fabricated decision JSON）| mock | mock | ✓ 秒级 | CI PR check |
| **L2** | mock（fabricated decision，省 token）| `MockIBKRClient` | 真 e2e DB（`ai_trader_e2e`）| ✓ 秒级 | CI PR check |
| **L3** | 真 Claude（Haiku + Sonnet）| 真 IBKR paper | 真 DB | 分钟级 | nightly workflow |

### 7.3 Paper 盘中 smoke report（吸收 CC-P1-5）

L3 / paper smoke 跑完生成 `testResults/scenarios_v2/agent2_paper_smoke_report_{date}.json`，push repo。下次 session 读 report 不靠记忆。

## 8. CI 集成

### 8.1 PR check
`.github/workflows/pr.yml` 加 job：
```yaml
- name: scenarios_v2 L1 + L2
  services:
    postgres:
      image: postgres:16
  run: |
    alembic upgrade head
    pytest tests/fixtures/scenarios_v2/ -v --layer=L1,L2
- name: scenario counts gate
  run: python scripts/check_scenario_counts.py --assert-matrix
```

### 8.2 L3 nightly workflow（吸收 Cloud-P0-1 + IB-P0-3/4）

`.github/workflows/scenarios_v2_paper_nightly.yml`：
```yaml
on:
  schedule:
    - cron: "30 17 * * *"   # UTC 17:30 = 13:30 ET 盘前（非周日）
jobs:
  l3_paper:
    if: github.event.schedule && fromJSON(format('{{"day":{0}}}', ...)).day != 7   # 跳周日
    timeout-minutes: 30
    env:
      IBKR_CLIENT_ID: 29                   # 吸收 IB-P0-3 专用 ID
      ANTHROPIC_API_KEY_BUDGET_USD: 20     # 吸收 Cloud-P1-5 成本上限
    steps:
      - name: IBKR probe
        run: python scripts/ibkr_probe.py || exit 0   # 维护窗口 skip-with-success (non-fail)
      - name: Pre-cleanup orphan orders
        run: python scripts/cancel_all_open_orders.py --client-id 29   # 吸收 IB-P1-6
      - name: Run L3
        run: pytest tests/fixtures/scenarios_v2/ -v --layer=L3
      - name: Post-cleanup
        run: python scripts/cancel_all_open_orders.py --client-id 29
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          path: testResults/scenarios_v2/
```

### 8.3 PreToolUse hook
改 `src/options_event_trader/services/agent2/**` 或 `tests/fixtures/scenarios_v2/**` → hook 要求 `pytest tests/fixtures/scenarios_v2/ -v` 全绿 + commit msg 贴命令输出（类 invariant 22 Frontend Testing Gate）。逃生口：`[skip-scenarios-v2: 理由]` + findings 跟进项。

Hook 文件：`~/.claude/hooks/check_scenarios_v2_intent.py`（UserPromptSubmit 扫意图词） + `~/.claude/hooks/check_scenarios_v2_gate.py`（PreToolUse Edit/Write 路径硬触发）。

## 9. 旧 scenarios 处置（吸收 SWE-P1-2）

- `tests/fixtures/scenarios/scenarios_batch_{1,2,3}.yml` **冻结**，加 `tests/fixtures/scenarios/README.md`：
  ```
  # ⚠️ FROZEN 2026-04-25
  # These scenarios are intake-layer only and reflect pre-agent2 semantics
  # (take_profit_rule_zh required, etc.). DO NOT modify. Run `scripts/run_scenarios_intake.py`
  # for intake regression. New Agent 2 scenarios live in tests/fixtures/scenarios_v2/.
  ```
- `scripts/run_scenarios_intake.py` 保留运行（intake 回归）
- 未来删 → 等 intake 层迁 Claude（finding `F-2026-04-25-AGENT1-INTAKE-STILL-OPENAI` 解决）+ 所有 intake 测试收敛

## 10. 单一真相：`scenarios/types.py`（吸收 CC-P0-2）

```python
# src/options_event_trader/scenarios/types.py
from typing import Literal

Action = Literal["HOLD", "PARTIAL_CLOSE", "FULL_CLOSE", "ADJUST_STOP"]
# ↑ 与 domain/agent2_decisions.py 的 Action 必须保持同一 Literal！CI parity test。

Category = Literal[
    "ENTRY_DECISION", "REVIEW_HOLD",
    "REVIEW_PARTIAL_CLOSE_LONG_VOL", "REVIEW_PARTIAL_CLOSE_SHORT_VOL",
    "REVIEW_FULL_CLOSE_LONG_VOL", "REVIEW_FULL_CLOSE_SHORT_VOL",
    "REVIEW_ADJUST_STOP_LONG_VOL", "REVIEW_ADJUST_STOP_SHORT_VOL",
    "RISK_OVERRIDE_R7", "LOW_CONFIDENCE_ESCALATION", "HOLD_AND_ALERT",
    "DTE_TERMINAL_HANDLING", "LIFECYCLE_REPLAY",
]

VolStance = Literal["LONG_VOL", "SHORT_VOL", "NEUTRAL"]

# CATEGORY_TO_EXPECTED_ACTIONS 映射
CATEGORY_TO_EXPECTED_ACTIONS: dict[Category, set[Action]] = {
    "ENTRY_DECISION": {"HOLD"},                   # 入场不算 action, HOLD 占位
    "REVIEW_HOLD": {"HOLD"},
    "REVIEW_PARTIAL_CLOSE_LONG_VOL": {"PARTIAL_CLOSE"},
    ... (完整映射)
}
```

**`tests/test_scenarios_v2_parity.py`**：断言 `set(Action.__args__)` 出现在至少一个 category 映射里，`decision_dispatcher` 所有 if-branch 的 action 都在 Literal 里（exhaustiveness check）。

## 11. 与 Step F 的交互

- **L1 不依赖 Step F**：纯决策层，mock 整链，Step F 未 ship 也能跑（Phase A 并行）
- **L2 依赖 Step F**：`expect_execution` 断言 `execute_exit_escalation_called` / `position_adjustments_row_inserted`，需要 Step F 的 dispatcher 落地
- **L3 依赖 Step F**：真 paper 改单，Step F 必须 ship 才能跑
- Step F ship 前：L2/L3 scenarios 写好但 CI 只跑 L1；Step F ship 后 CI 解锁 L2 + nightly 解锁 L3

## 12. 测试策略

### 12.1 Meta-test
- `tests/test_scenarios_v2_schema.py`：所有 YAML 过 Pydantic discriminated union；每大类 L1 ≥ 3 + L2 ≥ 1 矩阵断言
- `tests/test_scenarios_v2_parity.py`：Action / Category / VolStance 与 domain 层单一真相同步
- `tests/test_scenarios_v2_runner.py`：runner 正确 dispatch 到 L1/L2/L3 路径

### 12.2 Runner 自测
- `scripts/run_scenarios_v2.py --dry-run --scenario tests/fixtures/scenarios_v2/entry_decision/AG2-L1-ENTRY-001.yml`：单 scenario 跑通

### 12.3 Paper 盘中 smoke
`scripts/agent2_paper_smoke.py`（Step F spec §10.4 已列，此处复用）+ 选一条 LIFECYCLE_REPLAY L3 scenario 盘中跑。

## 13. Definition of Done

- [ ] Schema + Runner + Meta-test 全绿
- [ ] L1 scenarios ≥ 44（下限），目标 ≥ 55，10 大类 L1 ≥ 3 达标
- [ ] L2 scenarios ≥ 12（下限），目标 ≥ 20，10 大类 L2 ≥ 1 达标
- [ ] L3 scenarios ≥ 5，分层抽样（LOSING ≥ 30% / TIME_STOP ≥ 20% / R7 ≥ 10% / EARNINGS_IV_CRUSH ≥ 1）达标
- [ ] **L3 含 ≥ 1 条 live 小金额 smoke**（< $100 risk，吸收 Opt-P1-6）
- [ ] CI PR check 跑 L1 + L2 全绿
- [ ] `scenarios_v2_paper_nightly.yml` nightly workflow 在 DEV 环境连续 3 日不红（周日跳过）
- [ ] PreToolUse hook 本地验证：改 scenarios_v2/ 触发 gate
- [ ] 旧 `scenarios_batch_*.yml` README 冻结
- [ ] `scripts/check_scenario_counts.py` CI 集成
- [ ] `agent2_paper_smoke_report.{date}.json` 产物 push repo
- [ ] CLAUDE.md `invariant 22` 更新（加 scenarios_v2 gate）或新开 invariant 23

## 14. 设计决策（Why）

| 决策 | 原因 |
|---|---|
| L1/L2/L3 三层分开 + discriminated union | L1 快 CI 跑 / L2 加执行 / L3 真实度。schema 强约束避免混用（SWE-P1-7）|
| REVIEW_* 按 LONG_VOL / SHORT_VOL 拆大类 | IV 降 对 long debit 利空 / 对 short credit 利好，混一起 Agent 2 学错捷径（Opt-P0-4）|
| LIFECYCLE_REPLAY 强制分层抽样 | 防 Agent 2 few-shot calibration 过度乐观（Opt-P0-3）|
| DTE_TERMINAL_HANDLING 独立大类 | 0DTE 的 pin risk / gamma 爆炸是独立失效模式，用通用 5min review 处理会翻车（Opt-P1-2）|
| `reasoning_must_not_include` 黑名单替代 `must_include` 白名单 | 白名单绑 LLM 当前措辞, prompt 微调大面积红; 黑名单更稳 (SWE-P0-4) |
| bundle.news 字段预留但本版永远 null | 未来接入不改 schema / 不重写历史 scenarios，权衡"死字段" vs "schema 改动成本"偏后者（五专家共识）|
| 旧 scenarios 冻结不 migrate | 110 条 intake-only 仍是 valid 回归；新范式不可拉回旧结构；两套分离避免漂移（SWE-P1-2）|
| PARTIAL_CLOSE 分档收敛 5 档 | trader 常用 1/3, 1/2, 全平，Agent 2 从 enum 选降决策噪声（Opt-P1-1）|
| Action 在 `scenarios/types.py` 单一真相 | 新 action 增删时 scenarios runner 和 dispatcher 同源，避免 silent drift（CC-P0-2）|
| L3 nightly 跳周日 + client ID 独立 + probe fallback | IBKR 周日维护窗口 / client ID 冲突踢 worker / 连续假警报（Cloud-P0-1 + IB-P0-3/4）|

## 15. 未解决 / 后续技术债

| 项 | 优先级 | 何时做 |
|---|---|---|
| `F-2026-04-25-SCENARIOS-NEWS-INTEGRATION`：news 接入后补 `REVIEW_NEWS_*` 大类 scenarios | P2 | 等 Step F spec §14 news collector ship |
| `F-2026-04-25-SCENARIOS-GAMMA-SCALPING`：加 gamma scalping / skew trade 大类 | P2 | Phase 2 |
| `F-2026-04-25-SCENARIOS-ORDER-FLOW`：bundle 加 `order_flow.large_prints_nearby` 后补 scenarios | P3 | Phase 2 |
| `F-2026-04-25-SCENARIOS-L3-PAPER-LIVE-DELTA`：L3 paper vs live 差异量化持续验证（对应 `F-2026-04-21-PAPER-TO-LIVE-DELTA-UNKNOWN`）| P1 | 随 UAT live smoke |

---

**五专家签字**（brainstorming 2026-04-25）：全员 approve with P0 addressed。本 spec 吸收 13 组 P0 + 精选 P1。

**归档**：内部讨论 → `discussions/2026-04-25-step-f-scenarios-v2-5expert-review.md`（项目内部 repo, 不在公开 docs-hub）。脱敏公开版 → `docs-hub/docs/options-trader/specs/2026-04-25-scenarios-v2-framework-design.md`（本 spec 镜像）+ `docs-hub/docs/options-trader/decisions/2026-04-25-scenarios-v2-framework.md`（客户友好摘要）。
