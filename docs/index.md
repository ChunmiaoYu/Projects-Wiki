# 项目文档中心

欢迎来到项目技术文档聚合平台。这里集中管理所有项目的架构文档、API 参考、数据库设计和开发指南。

## 项目列表

| 项目 | 描述 | 状态 |
|------|------|------|
| [期权 AI 交易助手](options-trader/dashboard/) | 自然语言 → AI 策略生成 → IBKR 自动下单 | 开发中 (进度 65%) |
| [Auckland Property Shortlist](auckland-property-shortlist/dashboard/) | Auckland 房地产候选 → 离线 GIS 富化 → 评分 → 短名单 (桌面版 + 云端 Web) | 云上线稳定期 (进度 80%) |

## 文档说明

每个项目文档分三层：

- **自动文档**：由 [deepwiki-skill](https://github.com/natsu1211/deepwiki-skill) 从源码自动生成，描述代码结构。代码引用链接到对应 commit 的 GitHub 源码。
- **决策记录**：重大改动的客户友好版（为什么这样设计、有哪些候选、风险和回滚）。按全局规则每命中触发条件就归档一条。
- **设计 Spec**：项目内源头设计文档的脱敏公开镜像。

原始讨论 / Q&A 过程 / 内部细节保留在项目 repo 内（private），不在此 public wiki。

- Mermaid 图表支持在线渲染
- 使用 MkDocs Material 主题，支持全文搜索

---

*最后更新：2026-04-14*
