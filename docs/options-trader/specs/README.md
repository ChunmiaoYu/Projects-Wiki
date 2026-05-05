# 完整设计 — 期权 AI 交易助手

本目录是项目所有设计文档的总览。<span class="term-agent">Agent 1</span> / <span class="term-agent">Agent 2</span> / 工人体系 / 数据基础设施的完整规格都在这里。

---

## 推荐阅读顺序

| 序号 | 文档 | 谁该读 | 何时读 |
|---|---|---|---|
| **1** | ⭐ [**北极星 §1 — 项目第一版目标**](north-star-v1-target.md) | **所有人必读** | 看任何其他 spec 之前 |
| 2 | [2026-04-21 AI 自动盯盘 (完整版 v5)](2026-04-21-agentic-trading-redesign-design.md) | 想了解 <span class="term-agent">Agent 2</span> 持续决策细节的人 | 北极星 §1 看完后 |
| 3 | [2026-04-29 UI 重设计 v2](2026-04-29-ui-redesign-design.md) | 想了解前端架构的人 | 看完上面两个后 |

---

## ⭐ 为什么北极星 §1 是首位

**北极星 §1 是决策门禁**:
- 任何新功能 / 字段 / 流程变更前必先回查
- 含本版本目标 + 系统构成 + <span class="term-agent">Agent 1</span> + <span class="term-agent">Agent 2</span> + 3 个工人 + 10 维数据 bundle 全部规格
- 如有冲突, 北极星 §1 优先

其他 spec 是分支细节, 都应跟北极星 §1 一致。如果不一致, 以北极星 §1 为准, 其他 spec 标记 stale 待更新。

---

## 各 spec 的角色

| 文档 | 角色 | 状态 |
|---|---|---|
| ⭐ 北极星 §1 | **总纲** — 当前版本目标 + 全系统规格 | 2026-05-06 最新 |
| 2026-04-21 AI 自动盯盘 | <span class="term-agent">Agent 2</span> 持续决策子设计 | 待跟北极星 §1 同步 |
| 2026-04-29 UI 重设计 | 前端 Dashboard + 暗色 + Greeks 普通话子设计 | 待跟北极星 §1 同步 |

---

## 镜像说明 (技术层)

本目录是项目 private repo `docs/superpowers/specs/` 的**公开镜像**。脱敏前过 [全局 CLAUDE.md §18 红线](https://github.com/anthropics/claude-code) — API key / 账户号 / 客户姓名 / 具体金额 / VM IP / 实盘参数具体值 都不进 public。

镜像模板: [`docs-hub/templates/spec-mirror-template.md`](https://github.com/ChunmiaoYu/Projects-Wiki/blob/main/templates/spec-mirror-template.md)

相关:
- [决策记录 (客户友好提炼)](../decisions/)
- [客户看板 (实时运营)](../dashboard/)
