# 决策记录 — Scenarios v2 测试框架（10 大类 × 3 层验证）

> **归档位置**：`docs-hub/docs/options-trader/decisions/2026-04-25-scenarios-v2-framework.md`（public）
> **日期**：2026-04-25
> **参与者**：用户 + Claude + 五专家
> **相关 spec**：[2026-04-25-scenarios-v2-framework-design.md](../specs/2026-04-25-scenarios-v2-framework-design.md)
> **相关原始讨论**（internal）：项目 repo `docs/superpowers/discussions/2026-04-25-step-f-scenarios-v2-5expert-review.md`

---

## 触发的 §18 条款

- ① 触发五专家 brainstorming
- ⑤ 测试基础设施重大变更（新 YAML schema + 独立 runner + CI 集成）
- ⑦ 质量门变更（新增 `check_scenario_counts.py` CI gate 卡 PR）

---

## 背景（为什么要改）

项目已有一套 **100 scenarios 机制**（2026-04-19 上线），110 条 YAML 覆盖用户自然语言输入 → Agent 1 结构化解析的全套场景。但两件事让它不够用了：

1. **范式变了**：老 scenarios schema 要求每条必填 `take_profit_rule_zh` 和 `stop_loss_rule_zh`（用户必须写死止盈止损）。2026-04-21 起 Agent 2 agentic 范式上线后，止盈止损改为 AI 基于 10 维市场数据每 5 分钟动态决策，用户不再需要给死规则。老 schema 的核心断言点全部失效
2. **覆盖窄**：老 scenarios 只验 Agent 1 的自然语言解析，完全不碰后面的 Agent 2 决策、风控、下单、监控、平仓升级链。这些才是用户真正关心的"AI 靠不靠谱"

不改的后果：**新范式 Agent 2 每天在客户账户上跑却没有自动化测试覆盖**。每次改 prompt 或决策逻辑都靠人肉盘中观察，bug 累积到一定程度改一个踩三个。

---

## 决策内容（改了什么）

1. **新建独立 `scenarios_v2/` 目录**：旧 100 scenarios **冻结不删**（仍作 Agent 1 intake 回归），新的完全独立设计，避免两套互相漂移
2. **3 层验证分层**：
   - **L1**：纯决策层（mock 全链，秒级跑），专测"AI 收到这堆数据会做什么判断"
   - **L2**：加执行层（MockIBKR + 真数据库），测"决策变成订单了吗"
   - **L3**：真 IBKR paper 账户 + 真 AI 调用，盘前 nightly 跑，测"整链路在真实 API 下能跑完吗"
3. **10 大类覆盖**：入场决策 / 维持 / 部分平 / 全平 / 调止损 / R7 强制平 / 低信心升级 / 降级告警 / 临近到期特殊处理 / 全生命周期回放
4. **止盈止损场景按 long/short vol 拆**：IV 下降对 long 策略不利、对 short 策略利好，混一起会让 AI 学错捷径（"IV 降必平"这种过度简化规则）
5. **回放场景强制分层抽样**：必须至少 30% 亏损持仓 + 20% 时间到期 + 10% 风控 override + 1 次 earnings 失败。防止只回放赢的持仓导致 AI 过度乐观
6. **数量门卡 CI**：每条 PR 必须 L1 ≥ 50 / L2 ≥ 20 / 每大类 L1 ≥ 3 + L2 ≥ 1，`scripts/check_scenario_counts.py` 硬断言
7. **AI 原因文字不做白名单断言**：改为 `reasoning_must_not_include` 黑名单（"不能出现这句话"），避免断言过拟合当前 prompt 措辞，将来 prompt 微调大面积红

---

## Why（为什么选这个方案）

**Scenarios 的"原子单元"有三个候选**：

| 方案 | 优点 | 缺点 | 决策 |
|---|---|---|---|
| A. 单点 bundle → 单次决策 | 写快、覆盖广、CI 秒跑 | 错过"时间演化"（入场后第 3 次 review 心态） | 部分选（作 L1）|
| B. 全生命周期剧本 | 最接近真实交易 | 维护成本高，单测 500+ 行 YAML | 部分选（作 L3 少量）|
| C. 两层矩阵（A+B） | 代价分摊 | 运行环境要分三种 | **选这个**（演化为 L1/L2/L3）|

选 C 的核心理由：**L1 保覆盖广度（新维度如 news 上线只改 L1 fixture 字段），L3 保真实度（paper + 真 AI），L2 是两者的桥（mock API 但真 DB）**。单走 A 错过时间维度，单走 B 维护爆炸。

**其他选型**：
- 新旧 scenarios 合并 vs 分离：**分离**，旧的 schema 不兼容新范式，强行迁移会破坏 intake 回归
- YAML schema 统一 vs 按层拆：**按层拆**（discriminated union）；L1 不该有执行断言、L3 不该只有单点快照
- 允许 AI 原因文字白名单断言 vs 黑名单：**黑名单**；白名单绑 prompt 当前措辞，微调就大面积红，这正是老 scenarios 变僵尸字段的根因

---

## 风险和回滚

- **风险 1**：L3 nightly 烧 token（真调 Claude Haiku + Sonnet × 50 场景）→ 应对：workflow 加 `timeout-minutes: 30` + budget 上限 env var，超了 fail-fast
- **风险 2**：L3 周日撞 IBKR 维护窗口假警报 → 应对：workflow 跳周日 + IBKR probe 失败 skip-with-success
- **风险 3**：scenarios 数量门阻断正常开发（某次改动本身不需要加 scenarios 但被门卡住）→ 应对：`[skip-scenarios-v2: 理由]` 逃生口 + findings 跟进
- **回滚**：删 `tests/fixtures/scenarios_v2/` + `scripts/run_scenarios_v2.py` + 撤 CI workflow 即可；旧 scenarios 从未动过，持续作 Agent 1 回归

---

## 脱敏自查 ✓

- [x] 无 API key / token / secret
- [x] 无 IBKR 账户号
- [x] 无客户真实姓名/邮箱/具体金额
- [x] 无 VM IP / SSH key 路径
- [x] 无内部策略参数具体数值
