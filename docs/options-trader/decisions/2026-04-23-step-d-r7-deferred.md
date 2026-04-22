# 决策记录 — Step D 风控层完成 + R7 阈值改动延后到 Step E

> **日期**：2026-04-23
> **参与评审**：5 位项目专家（软件工程 / Claude Code 规则 / Oracle 云部署 / 美股期权实战 / IBKR 工程师），其中 R7 阈值决策另请期权实战专家单独 review
> **相关设计 Spec**：[`../specs/2026-04-23-risk-gate-v2-design.md`](../specs/2026-04-23-risk-gate-v2-design.md)
> **关联前置决策**：[`2026-04-21-agentic-trading-redesign.md`](2026-04-21-agentic-trading-redesign.md)
> **原始讨论（internal，对外不可见）**：保留在项目 repo 内

---

## 触发的 §18 条款

| 条款 | 命中 | 说明 |
|---|---|---|
| 2. invariant / 核心规则修改 | ✅ | 风控规则编排次序确立 |
| 5. 数据模型 / API 契约重大变更 | ✅ | `Agent2DecisionTrace` 加 `risk_gate_result` 字段；`account_state` 新增必填 contract |
| 7. 安全 / 风控规则变更 | ✅✅ | 整个 Step D 是 7 条硬风控规则 + R7 阈值是风险偏好选择 |

3 条触发条件命中 → 必须归档。

---

## 背景

Phase 11 Step D 实装"风控层 7 条硬规则"（spec §4），把 Agent 2 的 LLM 决策包在确定性的 risk gate 后。Step D 实装到一半（D9 evaluate() 编排时）遇到 4 个非平凡设计选择，加上**期权实战派**对"R7 Assignment 规避"阈值的反向 review，形成 5 个设计决策点。

---

## 决策内容（改了什么 / 不改什么）

### 1. R7 在评估链的位置 — 放最前 + override 后重跑 R1-R6

**问题**：R1-R6 都是"扫一眼，有问题就拒绝"的 reject 规则。R7 不一样 —— 它**强制改写** AI 决定（防止被强制行权），不 reject。R7 放最后会被 R1/R2 误拦下；放最前但跳过 R1-R6 又少了流动性兜底。

**决定**：R7 第一个跑 → 若触发 override 把决定改成全平 → 用新决定再跑 R1-R6，确保流动性、频率等检查仍生效。

### 2. 单笔仓位 + 总仓位规则当前是占位（保留 + 留 TODO Phase 12）

**问题**：单笔 / 总仓位规则的本意检查"开仓"，但 Agent 2 当前的 4 个决策动作（持有 / 部分平仓 / 全平 / 改止损）里**没有"开仓"或"加仓"**。这两条规则在 Agent 2 决策环里实际从来不会真正拦下任何东西。

**决定**：保留现规则 + 加注释，等 Phase 12 给 Agent 2 加"开仓"动作时再让两条规则真起作用。最终目标是 AI 完全自主决定开仓加注，全链路风控覆盖。

### 3. 入口加输入校验 + 配置不变量校验

**问题**：每条规则从一个 dict 取多个 key，少一个就崩。

**决定**：评估入口一次性校验所有 key 齐全（缺哪个清楚说），同时校验配置 yaml 的 tier 阈值不变量（防止配错出现 gap）。

### 4. 熔断触发只写 ERROR 日志（告警机制留 Step E）

**问题**：熔断规则触发时 reason 写"需人工确认"，但当前只返回结果，不发邮件/Slack。

**决定**：当前只写 ERROR 日志（运维 SSH 查日志即可处理）。告警机制（`alert_sink.py`）由 Step E 实装时一并接入。

### 5. R7 阈值改 5 天的提议 → 经期权实战专家 review，本 Step 不动，配 Step E P0 一起做

**用户初始提议**：R7 阈值从 2 天改 5 天，理由"等到必须 R7 触发时已经没流动性了"。

**期权实战专家 verdict**：
- 5 天数字本身合理（与 TastyTrade 业界主流"5 DTE rule"完全吻合）
- 但**单改阈值不够** —— 必须配套"R7 紧急平仓路径局部豁免组合策略不许市价单"的规则，否则 BAG 组合策略遇到流动性枯竭依然会卡死
- 项目目前两条 invariant 在 BAG 紧急场景下打架：
  - "止盈止损升级链 LMT → 追价 → MKT 必须走到底"
  - "组合策略不走 MKT 档"
- 需要在 R7 紧急平仓路径开"局部豁免"后门（仅此紧急路径，不全面推翻）

**最终决定（用户拍板）**：本 Step D 保持 R7 阈值原值 2 天，把 (a) 阈值改 5 天 + (b) 紧急路径 MKT 豁免**配套留 Step E P0 一起做**，严格遵循专家"缺一不可"verdict。

---

## 影响分析

### 不变的部分

- 7 条硬规则的全部参数都通过 `config/risk_limits.yml` 调，代码不变
- Agent 2 LLM 决策流程不动，只在末端加 risk gate 闸门
- 所有 invariant 1-21 兼容，唯一冲突点（16 vs 21 在 BAG 紧急场景）由 Step E (b) 局部豁免化解

### 新增 Step E 必接条目

- (a) `config/risk_limits.yml` `rule_7_assignment_avoidance.days_to_expiry_threshold: 2 → 5`
- (b) R7 紧急平仓路径局部豁免：BAG 组合在 R7 触发的紧急平仓时，升级链 LMT 多档失败后允许走 MKT 兜底（仅此紧急路径）
- 实装 `_build_account_state` 真实数据源（10 个 key，从 IBKR 账户 / DB / 时间逻辑取数）
- `services/alert_sink.py` + R6 reject 自动告警接入

### 新增 Step E P1 实战派建议

- (c) 标的流动性分级表（约 20 行 YAML：高流动性如 SPY/QQQ/大盘股、其他归低流动性）+ 入场前规则（中小盘 + 距到期 < 21 天直接拦截）。比维护"30 日 spread baseline"便宜得多。

### 新增 Phase 12 P1 远景

- (d) 持仓时间止损默认值改成距到期 5 天（让 Agent 2 首次决策保守，减少 R7 触发频率）
- (e) 财报事件策略豁免 R7 / 差异化阈值
- (f) Agent 2 加"开仓"动作 → 让单笔/总仓位规则真正起作用

---

## 测试现状

Step D 完成后，Phase 11 项目级别 targeted 测试 **197 全绿**（Step D 新增 49 个，Step C agent2 测试 0 回归）。

49 个新测试覆盖：每条规则 ≥3 case + 边界值 / 顺序编排 / R7 override + R3 reject / 配置不变量 / Schema 缺 key / Cache hit + R3 reject 集成测试 等。

---

## 与现有 invariant 的关系

| 项目 invariant | 受影响 | 关系 |
|---|---|---|
| 第 10 条（Risk Gate 是确定性代码非 AI） | ✓ 兑现 | 7 条规则全是 dataclass + 函数，无 LLM 调用 |
| 第 16 条（升级链 LMT → 追价 → MKT 必须走到底） | 间接 | R7 触发的全平走升级链 |
| 第 18 条（Worker 24h 持续决策，部署后不停机） | ✓ 兼容 | risk_gate 是同步函数调用，不影响循环 |
| 第 20 条（Agent 2 两个工作点） | 间接 | risk_gate 工作点 B 全跑；工作点 A 走 Step E 入场 entry gate |
| **第 21 条**（组合策略不许 MKT） | **冲突** | Step E (b) 局部豁免解决 |

---

## 下一步

进入 **Step E 执行层**（约 2.4 天 / ~10 commit）。Step E 起手第一件事：实装 `_build_account_state` 10 个真实数据源；第二件事：(a)+(b) 配套实装。
