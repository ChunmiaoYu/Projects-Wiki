# 完整设计 — Auckland Property Shortlist

本目录是项目所有设计文档的总览。V1 决策闭环 / 远景子能力 / forward compat hooks / 历史决策的完整规格都在这里。

---

## 推荐阅读顺序

| 序号 | 文档 | 谁该读 | 何时读 |
|---|---|---|---|
| **1** | ⭐ [**北极星 — V1 目标 + 最终远景 + 决策门禁**](north-star-v1-target.md) | **所有人必读** | 看任何其他 spec / 决定任何"是否在 scope" 之前 |
| 2 | [项目概述](../01_overview.md) | 业务方 / 新工程师 | 北极星看完后想理解"代码事实"时 |
| 3 | [双形态架构对比](../02_architecture.md) | 工程师 / 架构师 | 想理解桌面版 vs 云端版差异时 |
| 4 | [云端 Web 平台](../07_web_platform.md) | Web 开发 | 想了解 FastAPI + Worker + Pipeline 细节时 |

---

## ⭐ 为什么北极星是首位

**北极星是决策门禁**:

- 任何新功能 / 字段 / 流程变更前必先回查
- 含 V1 目标 + 远景子能力 + 5 问 cross-check + 中间路径 + forward compat hooks + 历史决策 + 反偏离警示
- 如有冲突, **北极星优先**

其他 spec 是分支细节, 都应跟北极星一致。如果不一致, 以北极星为准, 其他 spec 标记 stale 待更新。

---

## 各 spec 的角色

| 文档 | 角色 | 状态 |
|---|---|---|
| ⭐ [北极星](north-star-v1-target.md) | **总纲** — V1 目标 + 远景 + 决策门禁 | 2026-05-17 首版 |

> 未来子 spec (Candidate state machine UI / Admin aggregate 报表 / Onboarding 自助 / 邮件通知 / Billing / 跨城市 / Loan / Contract / Renovation / Rental) 落地时, 在此清单加行。

---

## 镜像说明 (技术层)

本目录是项目 private repo `docs/superpowers/specs/` 的**公开镜像**。脱敏前过 [全局 CLAUDE.md §18 红线](https://github.com/anthropics/claude-code) — API key / 账户号 / 客户姓名 / 具体金额 / VM IP / 实盘参数具体值 都不进 public。

镜像模板: [`docs-hub/templates/spec-mirror-template.md`](https://github.com/ChunmiaoYu/Projects-Wiki/blob/main/templates/spec-mirror-template.md)

相关:

- [决策记录 (客户友好提炼)](../decisions/README.md)
- [客户看板 (实时运营)](../dashboard/index.md)
- 原始讨论 (internal, 不在此公开): 项目 repo `docs/superpowers/discussions/`
