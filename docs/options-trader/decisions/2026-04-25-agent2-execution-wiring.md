# 决策记录 — Agent 2 执行接入（让 AI 决策真的改单）

> **归档位置**：`docs-hub/docs/options-trader/decisions/2026-04-25-agent2-execution-wiring.md`（public）
> **日期**：2026-04-25
> **参与者**：用户 + Claude + 五专家
> **相关 spec**：[2026-04-25-step-f-execution-wiring-design.md](../specs/2026-04-25-step-f-execution-wiring-design.md)
> **相关原始讨论**（internal，客户不可见）：项目 repo `docs/superpowers/discussions/2026-04-25-step-f-scenarios-v2-5expert-review.md`

---

## 触发的 §18 条款

- ① 触发五专家 brainstorming
- ⑤ 数据模型重大变更（新增 `position_adjustments` 表 + `positions` 加一列）
- ⑥ 运维策略调整（三环境 `.env` 新增配置 + `ENABLE_AGENT2_DISPATCH` feature flag）
- ⑦ 安全/风控规则变更（Paper/Live 硬门，首次 live 部署必须人工开关）

---

## 背景（为什么要改）

上一阶段（Phase 11 Step A-E）把 Agent 2 的决策大脑全部搭好：10 维市场数据 Bundle、Haiku/Sonnet 双层模型、7 条风控规则、紧急平仓判定。但在今日盘中准备测试时发现：

**决策大脑的输出没有接到下单的手上。** Agent 2 每 5 分钟扫一次持仓、产出"部分平仓 30%"这类决策，但调度器拿到决策后**直接丢弃**，订单从来没真发出去过。同时审计表 `agent2_decisions` 永远是空的，事后想回溯"AI 当时为什么这么判断"完全没依据。

不改后果：**Agent 2 新范式在客户账户上只是"陪跑"，止盈止损还是走老的用户硬规则路径**，与 invariant 10/20（Agent 2 动态管 TP/SL）的产品承诺不一致。

---

## 决策内容（改了什么）

1. **新增 `decision_dispatcher`**：把 AI 决策的 4 种动作（维持/部分平/全平/调止损）路由到真正的执行模块
2. **新增 `decision_repository`**：把每次 AI 决策落到 `agent2_decisions` 表做事实真相层，事后可审计
3. **新增 `trace_archiver`**：完整决策过程（含市场数据、提示词、AI 原始输出）gzip 冷存，本地 7 天、云端 30 天自动轮转
4. **新增 `position_adjustments` 独立表**：AI 每次调整止损都留痕不覆盖（金融可审计要求）
5. **新增 `NewsCollector` 接口**：本版返 null，但 schema 字段预留，未来接入新闻源不用动数据结构
6. **Paper/Live 硬门**：生产环境首次针对某持仓下 AI 单前，必须先在 paper 账户走通同流程（`paper_smoke_passed_at` 字段 + feature flag 双重保护）
7. **监控周期从 5 分钟降到 ≤60 秒**（仅止损检查，cheap query）：AI 调整止损后，老的轮询周期最长 5 分钟才发现新值，对突发行情保护不足

---

## Why（为什么选这个方案）

**ADJUST_STOP 的做法有三个候选**：

| 方案 | 优点 | 缺点 | 决策 |
|---|---|---|---|
| A. IBKR 服务器侧 bracket modifyOrder | 止损实时生效，不受轮询周期影响 | 改动大，涉及订单状态机、BAG combo 的 bracket 语义复杂 | **不选本版**（长期迁移） |
| B. 不做，Agent 2 说 ADJUST_STOP 就忽略 | 零改动 | 违反 invariant 16（止损核心价值是不在场也能控损）| 不选 |
| C. DB + 监控轮询读新值（本版选）| 改动小，5 天能落地；配合轮询周期降到 60s overshoot 从 5 分钟缩到 1 分钟 | 仍有 ≤60s 的理论 overshoot 窗口 | **选这个** |

选 C 的核心理由：**先 ship 90% 的价值，剩下 10%（从 60s 再压到毫秒级）记长期技术债 `F-2026-04-25-ADJUST-STOP-SERVER-SIDE` 改 bracket**。保持推进速度而不是追求一次完美。

**其他选型**：
- 独立 `position_adjustments` 表 vs `positions` 加列：独立表赢在**可审计性**（每次调整历史都留痕，FK 到 AI 决策形成闭环）
- DB 先写后下单 vs 反向：**DB 先写**，保证事实层永远是真相，订单失败可重试
- Paper/Live 硬门硬挡 vs 软警告：**硬挡**，首次 live 部署不能在客户真仓上 debug

---

## 风险和回滚

- **风险 1**：AI 第一次真改单可能产生意料外的订单（spread 太宽 / 数量取整错误）→ 应对：`enable_agent2_dispatch` feature flag 默认 false，UAT 小金额 live 2-3 天验收后人工打开
- **风险 2**：`agent2_decisions` 表写入失败 → 应对：DB 先写优先级高于订单执行，写失败直接短路 review 不下单
- **风险 3**：轮询降到 60s 增加 IBKR API 限流压力 → 应对：仅止损检查走便宜 query，不触发 reqPositions（后者已知 50 msg/s 限制）
- **回滚**：`ENABLE_AGENT2_DISPATCH=false` 一行配置即可回退到"AI 只 log 不下单"的老行为；Alembic downgrade 全部两路可逆

---

## 脱敏自查 ✓

- [x] 无 API key / token / secret
- [x] 无 IBKR 账户号
- [x] 无客户真实姓名/邮箱/具体金额
- [x] 无 VM IP / SSH key 路径
- [x] 无内部策略参数具体数值
