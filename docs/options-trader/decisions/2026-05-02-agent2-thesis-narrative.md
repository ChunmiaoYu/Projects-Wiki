# 2026-05-02 — Agent 2 决策脉络（thesis chain）

## 简介

让每次 5 分钟一次的持仓决策不再"失忆"。过去每次 review 都是独立判断剩余仓位，
现在每次决策都基于上次的"脉络浓缩"+ 当前市场数据继续推演。

## 改动

每次 Agent 2 决策（开仓那次 + 后续每 5 分钟那次）同时输出一段
`thesis_summary_zh`（≤ 600 字，数据驱动）。下次决策读上次的浓缩 + 当前
10 维 bundle → 写新的浓缩。链条按持仓延续传到平仓。

## 客户视角

"Agent 2 详细测试结果"页面将展示完整 thesis chain timeline — 从开仓到平仓
每 5 分钟一段数据驱动的状态浓缩。客户能看到：
- 开仓初衷如何被 Agent 持续印证或动摇
- 每个关键转折（VIX 突变、PARTIAL_CLOSE 锁利等）何时发生
- Agent 当前评估和下一步预期触发条件

## 设计哲学（用户与 AI 协作产出）

- **架构 α 滚动单条**: 每条 thesis 是"开仓至今脉络的当下完整浓缩"，不是
  事件流水。下次只读最近 1 条
- **有损压缩是 feature**: 不重要细节自然衰减，重要事件由 LLM 反复强化
- **数据化叙事**: 每句涉及市场状态/演进/转折必须有具体数字，禁用空泛
  语言（如"市场转弱"必须写成"VIX 14.2→18.7（+31%）+ SPY -1.2%"）
- **600 字 prompt 软约束**: LLM 自我控制，不事后截断

## 风险 & 限制

- thesis 由 LLM 自我产出 — 质量依赖 prompt 工程（已固化 5 要素 + 数据化
  要求 + good/bad 范例 + 自检清单）
- v1 不做 cosine 相似度防偷懒检测 — 盘中观察后 phase 2 决定
- 紧急 kill switch: `enable_agent2_thesis=false` 即可降级回老路径

## 实施

- Spec: 2026-05-02-agent2-thesis-narrative-design.md
- Plan: 13 task TDD 拆分
- Tests: 116 PASS, 0 regression

## 五专家评审

软件工程 / CC AI 规则 / 云部署 / 期权实战 / IB-TWS — 全员 PASS。
P0 (migration 兼容) + 4 P1 (单测覆盖、prompt 差异化、kill switch、
realized_pnl 字段) 全部纳入实施。
