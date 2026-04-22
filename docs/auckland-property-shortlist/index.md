# Auckland Property Shortlist — 技术文档索引

> 面向 **内部开发团队 + 客户技术顾问** 的完整技术文档。基于 `c:/Users/yumia/Downloads/akl_property_shortlist/` 源码（2026-04-22 快照）。

---

## 文档列表

| # | 文档 | 摘要 | 核心读者 |
|---|---|---|---|
| 01 | [项目概述](01_overview.md) | 业务定位、两种交付形态（桌面 vs 云）、高层流水线、代码结构速览 | 所有人 |
| 02 | [双形态架构对比](02_architecture.md) | 桌面版 vs 云端版数据流对比、共享代码 vs 分叉点、技术栈 | 工程师 / 架构师 |
| 03 | [桌面业务流水线](03_desktop_pipeline.md) | `main.py:run()` 详解、两条候选来源、评分模型、硬过滤、两阶段 GIS、Excel 累积输出 | 桌面版开发 |
| 04 | [离线 GIS 富化](04_gis_enrichers.md) | `enrichers/` 11 个模块、窗口读、CRS 2193、Stage 1/2 并行、领域富化模块、数据文件清单 | GIS 工程师 |
| 05 | [数据源抓取](05_providers.md) | Trade Me v1 search API、listing_to_row、CV 查询、Playwright fallback、离线 Candidates.xlsx | 抓取 / 集成 |
| 06 | [授权系统](06_licensing.md) | Ed25519 + machine_id + DPAPI + 时间回拨；activate.exe 激活流程；license_gen 签发 | 发行 / 安全 |
| 07 | [云端 Web 平台](07_web_platform.md) | FastAPI + 三进程（web / worker / cron pipeline）、JWT + RBAC、SSE、Admin 面板、Re-enrich CLI | Web 开发 |
| 08 | [数据库](08_database.md) | SQLite WAL、6 张表 ER、字段映射（中英文）、Upsert 模式、Alembic / 备份 | DBA / 后端 |
| 09 | [部署与 CI/CD](09_deploy_cicd.md) | Oracle Cloud VM 初始化、systemd / cron / UFW、GitHub Actions（ci + deploy-prod）、分支策略、故障排查 | DevOps |
| 10 | [配置与依赖](10_config_dependencies.md) | `config.json` / `.env` / 全部 env var、requirements、PyInstaller 构建 | 运维 / 开发环境 |
| 11 | [路线图](11_roadmap.md) | Backlog（listing lifecycle / HTTPS / private msg / WeChat）、技术债、开放问题、性能边界 | 产品 / 架构 |

---

## 阅读路径建议

### 产品经理 / 业务方
1. `01_overview.md` §1-3（业务定位 + 两种形态 + 高层流水线）
2. `07_web_platform.md` §9 Candidates 工作流
3. `11_roadmap.md`

### 工程师（新来接手项目）
1. `01_overview.md`（全）
2. `02_architecture.md`（全）
3. `03_desktop_pipeline.md`（如果主要改桌面版）或 `07_web_platform.md`（如果主要改 Web）
4. `04_gis_enrichers.md`（了解 GIS 核心）
5. `10_config_dependencies.md`（跑起来）

### GIS 工程师
1. `01_overview.md` §6（数据资产）
2. `04_gis_enrichers.md`（全，重点）
3. `03_desktop_pipeline.md` §6 两阶段 GIS
4. `08_database.md` §3.2 Property 表字段

### DevOps / SRE
1. `02_architecture.md` §3 云端三进程
2. `09_deploy_cicd.md`（全）
3. `10_config_dependencies.md` §4 `.env`
4. `08_database.md` §7 备份策略

### 安全 / 合规
1. `06_licensing.md`（桌面版授权）
2. `07_web_platform.md` §3 JWT + RBAC
3. `10_config_dependencies.md` §9 Secrets

### 客户技术顾问
1. `01_overview.md`（全）
2. `02_architecture.md` §1 边界
3. `11_roadmap.md` §1 Backlog + §5 开放问题

---

## 核心事实卡片

| 项 | 值 |
|---|---|
| 代码仓库 | `c:/Users/yumia/Downloads/akl_property_shortlist/` |
| 主要语言 | Python 3.11 |
| 桌面版入口 | `aps_bootstrap.py` → `main.py:run()` |
| 云端版入口 | `run_web.py` → `web.app:create_app` |
| 云端部署路径 | `/opt/aps/`（Oracle Cloud Ubuntu 22.04 VM） |
| 默认端口 | 8080 |
| 数据库 | SQLite WAL, 6 张表 |
| 抓取源 | Trade Me v1 JSON API（同源） |
| GIS 数据 | 本地 `./data/` or `/opt/aps/data/`，~17 GB |
| 核心 GIS 投影 | EPSG:2193（NZTM 2000） |
| 评分模型 | 7 子项加权归一化（0-1）加权求和 |
| 默认 zone allowlist | `18 / 60 / 8 / (19)` |
| 硬过滤 slope 阈值 | > 1/3.73（约 15°） |
| 授权（桌面） | Ed25519 + machine_id + DPAPI + 时间回拨 |
| 鉴权（云端） | JWT HS256 cookie + bcrypt + 3 种角色 |
| Pipeline cron | 每小时 0 分，06:00-23:00 + 00:00 NZT |
| CI | GitHub Actions pytest（PR to main/dev 触发） |
| Deploy | GitHub Actions 手动 workflow_dispatch |

---

## 源码行数分布

| 模块 | 行数 | 说明 |
|---|---|---|
| `main.py` | 2090 | 桌面版主流水线 |
| `providers/trademe.py` | 1644 | Trade Me 抓取 + CV |
| `enrichers/` | ~7600 | 11 个 GIS 富化模块 |
| `aps_license/` | ~800 | 离线授权 |
| `web/` Python | ~3200 | FastAPI + Worker + Pipeline + DB |
| `web/templates/` | ~4400 HTML | Jinja2 + Tabulator |
| `web/static/` | ~1900 JS/CSS | filter popover + Tabulator |
| `tests/web/` | 21 个 test 文件 | 云端测试 |
| `deploy/` | ~800 bash | 部署脚本 + systemd + cron |

合计约 **20k+ 行 Python**（不含 venv 和生成物）。

---

## 变更约定

- 加 enricher → 流程见 `04_gis_enrichers.md` §10
- 加 DB 列 → 流程见 `08_database.md` §6.3
- 改评分权重 → env var 覆盖或改 `config.py:286-316`（见 `10_config_dependencies.md` §3.5）
- 改 zone 过滤 → 改 `config.json` 的 `zone_allowlist`（`03_desktop_pipeline.md` §5）
- 发新版桌面 → `build_release.bat`（见 `10_config_dependencies.md` §6）
- 发新版云端 → GitHub Actions "Deploy to PROD"（见 `09_deploy_cicd.md` §6.2）

---

## 外部文件引用（在源码 repo 内）

| 文件 | 内容 |
|---|---|
| `README.md` | ~500 行用户文档（含使用指南、FAQ、troubleshooting） |
| `deploy/README.md` | 548 行部署详解（含 Phase A-E checklist） |
| `Akl Property Shortlist - Delivery Introduction.md` | 客户交付介绍 |
| `AklPropertyShortlist_Release_Postmortem_2026-02-12.docx` | 2 月发布复盘 |
| `findings.md` | 当前未解决技术债 |
| `CLAUDE.md` | 项目特定的工作规则 |

---

## 反馈

发现错误或文档过期：
- 源码变更是 source of truth，docs-hub 是派生品
- 重要错误 → 同时修源码对应 comment + docs 该节
- 少量拼写 / 格式 → 只改 docs

Generated: 2026-04-22
