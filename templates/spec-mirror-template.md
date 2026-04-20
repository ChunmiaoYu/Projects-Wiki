# [Spec 标题]

> **镜像状态**：此文件镜像自项目 repo `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
> **镜像日期**：YYYY-MM-DD
> **源文件 commit**：`<git short sha>`（若源头更新需重新镜像）
> **对应决策记录**：`[decisions/YYYY-MM-DD-<topic>.md](../decisions/YYYY-MM-DD-<topic>.md)`

---

## 镜像方式

两种实现（项目决定选哪个，记在 docs-hub/README.md）：

### 方式 A — 文件拷贝（简单）

直接把项目 spec 文件内容拷贝到此文件。每次项目源头更新需手动同步。适合 spec 稳定、变更不频繁的场景。

### 方式 B — mkdocs include 插件

用 `mkdocs-include-markdown-plugin`，此文件只写元信息 + include 指令，build 时拉取项目源头。要求 docs-hub repo 和项目 repo 在同一本地开发机上，且路径可预测。

---

## 粘贴内容时的脱敏要求

镜像到 docs-hub 前，按全局 CLAUDE.md §18 脱敏红线检查：
- 项目内 spec 可能含真实数值（如 `max_risk_dollars: 1000`），镜像到 public 时改为 `<REDACTED — 见项目配置>`
- 项目内 spec 可能含账户号（DU/U），镜像时改为 `<DU-PAPER>` / `<U-LIVE>` 占位

---

<!-- BEGIN:SPEC_CONTENT -->
<!-- 此处粘贴项目 spec 主体，或放 include 指令 -->
<!-- END:SPEC_CONTENT -->
