# Templates — 归档模板

这里是**全局 CLAUDE.md §18（重大决策的 docs-hub 留痕）** 用到的三份模板。

## 文件清单

| 文件 | 放哪里 | 公开性 |
|---|---|---|
| `decision-template.md` | `docs-hub/docs/<project>/decisions/YYYY-MM-DD-<topic>.md` | Public（客户可见） |
| `spec-mirror-template.md` | `docs-hub/docs/<project>/specs/YYYY-MM-DD-<topic>-design.md` | Public |
| `discussion-archive-template.md` | 项目 repo `docs/superpowers/discussions/YYYY-MM-DD-<topic>.md` | Internal（项目 repo private 天然保护） |

## 使用流程

1. brainstorming 五专家 Present design 结束 → 用 `discussion-archive-template` 写项目内 discussions/
2. 写 spec 文件到项目 `docs/superpowers/specs/` → 用 `spec-mirror-template` 复制一份到 docs-hub
3. spec 用户确认 / 实施完成 → 用 `decision-template` 写 docs-hub decisions/ 客户友好版

## 脱敏红线（每次粘贴前过一遍）

docs-hub 是 **public GitHub Pages**，绝不出现：
- API key / token / secret
- 账户号（DU/U 开头 IBKR / Oracle tenancy ID）
- 客户真实姓名 / 邮箱 / 具体金额
- VM IP / SSH key 路径 / 数据库密码
- 实盘策略参数具体数值

Internal 文件（discussions/）可以有上述内容。
