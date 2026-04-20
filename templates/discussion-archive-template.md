# 讨论归档 — [标题]

> ⚠ **Internal only** — 此文件仅在项目 repo `docs/superpowers/discussions/` 目录，**不镜像到 docs-hub**。
> **日期**：YYYY-MM-DD
> **Session**：Claude Code session（transcript 路径可选附）
> **对应 spec**：`docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
> **对应 public 决策**：`docs-hub/docs/<project>/decisions/YYYY-MM-DD-<topic>.md`

---

## 保留目的

此文件保留**原始 Q&A 和讨论过程**，用于：
1. 未来新 session 快速理解"为什么这样设计"，避开已讨论过的坑
2. 回顾五专家评审的原始反对意见，确认是否仍有效
3. 内部复盘 / 培训 / 新人 onboarding

Public decisions/ 是**提炼版**，省去了 Q&A 过程和被否决的方案。如果要知道"当时为啥没选方案 X"，看本文件。

---

## 参与的专家（若走了 brainstorming）

按 §14 / §15 全局三位 + 项目两位：
- 软件工程专家
- CC AI 规则/记忆系统专家
- 云部署专家（项目 memory 的具体云厂商）
- [项目专家 1，如 options 项目的美股期权实战专家]
- [项目专家 2，如 options 项目的 IB/TWS 工程师]

---

## 用户澄清和 Q&A 过程

### Round 1 — [主题，例如 方向定义 / 边界澄清]

**Claude 问**：...
**用户答**：...
**Claude 问**：...
**用户答**：...

### Round 2 — [主题]

...

（保持原汁原味，不改写用户话术 —— 未来 review 时要看得出用户当时的真实考虑）

---

## 五专家评审过程（若适用）

### 软件工程专家提出的 P0/P1 问题

- P0-1：...
- P1-1：...
- **如何修正**：...

### CC AI 规则/记忆系统专家

- P0-1：...
- **如何修正**：...

### 云部署专家

...

### 项目专家 1

...

### 项目专家 2

...

---

## 被否决的候选方案

- **方案 A**：... **为什么否决**：...
- **方案 B**：... **为什么否决**：...

最终方案见对应 decisions/ 和 specs/。

---

## 此讨论的后续任务（如有）

- TODO 1 ...（归入 task_plan.md）
- TODO 2 ...（归入 findings.md）
