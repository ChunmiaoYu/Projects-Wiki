# 决策记录 — [标题]

> **归档位置**：`docs-hub/docs/<project>/decisions/YYYY-MM-DD-<topic>.md`（public，客户可见）
> **日期**：YYYY-MM-DD
> **参与者**：用户 + Claude + 五专家（如涉及 brainstorming）
> **相关 spec**：`[spec 文件名](../specs/YYYY-MM-DD-<topic>-design.md)`
> **相关原始讨论（internal，客户不可见）**：项目 repo `docs/superpowers/discussions/YYYY-MM-DD-<topic>.md`

---

## 触发的 §18 条款

列出命中的清单编号（1-7），例如：
- ① 触发五专家 brainstorming
- ③ 新增 Anthropic SDK 依赖
- ④ LLM 厂商 OpenAI→Claude

---

## 背景（为什么要改）

用 3-5 句大白话说清 **原先为什么那样设计** + **遇到了什么问题 / 新需求** + **不改会怎样**。给客户看能让他理解脉络。

---

## 决策内容（改了什么）

精简列出核心改动：

1. **XXX 组件**：从 A 模式改为 B 模式
2. **YYY 配置**：新增可配置参数 ...
3. ...

具体实现细节放 spec 文档，这里只列"what changed"。

---

## Why（为什么选这个方案，不选别的）

对比至少 2 个候选方案的优劣，解释**为什么最终选这个**。包含：

- 选项 A（最终选）：优点 X / 缺点 Y
- 选项 B（备选）：优点 X / 缺点 Y（为什么没选）

---

## 风险和回滚

- 已知风险 1：... → 应对 ...
- 已知风险 2：... → 应对 ...
- 如何回滚：如果 PROD 发现问题，怎么退回老方案？

---

## 涉及文件（对内文件清单）

- 项目源文件：`path/to/file.py`
- 数据库 migration：`alembic/versions/XXX`
- 配置变更：`.env.{env}` / `config/*.yml`

（此节可选，给内部开发者用；客户只看前面）

---

## 脱敏自查 ✓

提交前确认：
- [ ] 无 API key / token / secret
- [ ] 无 IBKR 账户号（DU/U 开头）/ Oracle tenancy ID
- [ ] 无客户真实姓名/邮箱/具体金额数字
- [ ] 无 VM IP / SSH key 路径
- [ ] 无内部实盘策略参数具体数值

若其中任一需要出现，把该文档改放项目 `docs/superpowers/discussions/`（internal）。
