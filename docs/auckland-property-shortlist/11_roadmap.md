# 11 · 路线图与扩展点

> 对应文件：README.md §14、findings.md、部分 spec 文档。本章总结 **已知扩展点、待实现功能、重构方向**，供产品规划和工程估算参考。

---

## 1. 已在 backlog 的方向

### 1.1 Listing Lifecycle Tracking（中等优先）

**需求**：跟踪一条 listing 从 active → under_offer → sold/withdrawn 的生命周期，而不是每次 pipeline 都 upsert 覆盖。

**当前限制**：
- `properties` 表只保留最新状态，历史信息丢失
- Pipeline 再跑一遍，如果 Trade Me 已下架，reject_reason 不变（没人检测 delisted）

**设计方向**：
- 新增 `listing_history` 表，记录每次 pipeline 看到该 listing 的 snapshot
- Pipeline 每次比对"上次看到 + 这次没看到" → 标记 sold/withdrawn
- Admin 面板增加"lifecycle funnel" 图
- Candidates 阶段机（`progress` 字段）已存在，只是还没挂上自动 flow

**工作量估计**：~1 周（新表 + pipeline 扩展 + UI）

### 1.2 Trade Me Rentals API 接入

**需求**：让 `rent_avg_3br_pw` 不再永远 `None`。

**当前状态**：`enrichers/rent.py` 是占位（`enrichers/rent.py:4-9` 有 TODO）。

**实现思路**：
- 用同样的 `api.trademe.co.nz/v1/search/property/rental.json` endpoint
- 按 suburb + bedrooms 分组拉一批（caching 24h）
- 计算 median weekly rent
- 填 `rent_avg_{2,3,4}br_pw`
- `scoring.py` 里 `score_rent` 从 MISSING_NEUTRAL=0.5 变成真实分

**工作量估计**：3-5 天（抓取 + 缓存 + 测试）

### 1.3 HTTPS / Domain

**当前**：只有 `http://<vm-ip>:8080`。

**方向**：
- 注册域名（apspropertyshortlist.com 类）
- Caddy / nginx + Let's Encrypt 做反向代理 + TLS
- UFW allow 80/443，关闭 8080 外部访问

**工作量估计**：半天（如果不计 DNS / 域名采购）

### 1.4 Trade Me Private Message 集成

**需求**：用户在 APS UI 里直接给 listing 卖家发私信（不跳到 Trade Me 网站）。

**障碍**：
- Trade Me 公开 API 不含 private message（需要开发者账号申请）
- 私信本质是 IM，要考虑通知 / 收件箱 / 历史记录

**估计**：需要先和 Trade Me 沟通 API 接入；纯前端代码 1 周，总工期取决于 API 许可。

### 1.5 WeChat 联动

**需求**：分享 shortlist 到微信群 / 朋友圈；微信小程序入口；公众号推送新 listing。

**挑战**：
- 微信生态要求 ICP 备案（APS 服务器在 Oracle Cloud，对中国大陆不友好）
- 微信小程序要求 HTTPS（先做 §1.3）
- 可能要用企业微信而不是个人微信

**估计**：作为可选高优先级（客户很想要），但技术栈改动不小。

---

## 2. 代码层面的技术债

### 2.1 `main.py` 单文件 2090 行

**问题**：`run()` 600 行、`process_candidates` 957 行，难读难测。

**不重构的理由**：
- PyInstaller 打包单文件更可靠（避免 frozen 模式下 relative import 的坑）
- 重构成模块化会破坏 `web/services/pipeline.py` 的 `from main import ...` 这种"复用桌面版函数"的模式
- 已有 21 个 tests/web/ 覆盖核心流程，重构收益不大

**如果要重构**：
- 抽 `process_candidates_from_inputfolder` 到 `main/candidates.py`
- 抽 `_parallel_cfg / _split_rows_for_threads` 等 utility 到 `main/parallel.py`
- `run()` 拆成 `read_existing → fetch → enrich → score → write` 几段

### 2.2 `enrichers/gis.py` 1330 行

**问题**：`enrich_frontage_zone_pipes` 单函数 653 行（`gis.py:275-928`）。

**理由**：顺序非常关键（parcel match → zoning → slope → pipes → flood → overland → frontage，每步依赖前一步的 early exit / cached geometry），拆分函数会引入参数传递地狱。

**渐进改进**：把每个阶段的 try/except + timing 块抽成 `_stage_zoning(row, parcel) / _stage_slope(row, parcel)` 等，`enrich_*` 只保留编排。

### 2.3 前端 template 长

- `shortlist.html` 885 行
- `candidates.html` 1187 行
- `admin.html` 2116 行

**原因**：Jinja2 + inline JS + inline CSS 混写；Tabulator 配置很长。

**渐进改进**：
- inline JS 拆到 `web/static/js/shortlist.js` 等
- 公共组件（比如 Tabulator 列定义生成器）提到 `aps-common.js`
- 考虑 Alpine.js 或 HTMX 处理简单交互（但这是**新依赖**，需要评估）

### 2.4 `web/routers/admin.py` 528 行

**问题**：5 个 tab 所有 API 都挤在一个 router。

**改进**：按 tab 拆成 `admin_users.py / admin_candidates.py / admin_pipeline.py` 等子 router。

### 2.5 没用 Alembic 做迁移

见 `08_database.md` §6。当前用 `init_db._migrate_add_columns` 手动列表。每次加列都要改 `models.py` + `init_db.py` 两处。

**长期方向**：切到 Alembic `autogenerate` 流程，init-vm.sh 改成 `alembic upgrade head`。

### 2.6 Rent enricher 占位

见 §1.2。

### 2.7 `web/services/pipeline.py` 的 `from main import ...`

循环依赖风险。目前工作是因为 `main.py` 的函数纯函数式，但如果 `main.py` 在 import 时有副作用（比如 setup_logging），会导致 pipeline 启动时也跑那些副作用。

**改进**：把要复用的函数抽到 `shared/pipeline_utils.py`，`main.py` 和 `web/services/pipeline.py` 都从它 import。

---

## 3. 架构层面的想法

### 3.1 多 Worker

**当前**：`web/worker.py` 单进程、串行处理 job queue。

**问题**：用户连续输入 10 个地址 → 排队 10× 单次时间。

**方向**：
- systemd 跑 N 个 `aps-worker@1.service` / `aps-worker@2.service`
- 需要 DB 层的"job claim"原子性 —— SQLite 不好做 `SELECT FOR UPDATE SKIP LOCKED`
- 更合理的升级路径是**换 Postgres**

**估计**：一周（含 Postgres 迁移）；但当前用户量还撑得住单 worker。

### 3.2 Postgres 升级

触发点：
- 多 worker 需求（§3.1）
- 数据量增长到 SQLite 性能边界（几十万行）
- 需要全文检索（listing description 搜索）

涉及改动：
- `web/db/session.py:_set_sqlite_pragmas` 移除
- `web/db/models.py` 用 Postgres-specific 类型（JSONB / tsvector）
- `init_db.py._migrate_add_columns` 换成 alembic
- `web/services/pipeline.py` 的 `sqlite_insert.on_conflict_do_update` 换 Postgres 对应语法
- 备份策略换成 `pg_dump`

### 3.3 Dashboard 分析

`README.md` 提到"Excel/PowerBI 做二次分析：区域分布、TopN 稳定性、策略回测"。当前只有 Excel 输出。

**方向**：在 admin 面板加 `/admin/analytics`：
- 按 suburb 的 property 分布
- 过去 N 周 pipeline 新增趋势
- 评分分布直方图（quality check）
- Candidates progress funnel（new → contracted 转化率）

用 Tabulator 已经在前端，加 Chart.js / ApexCharts 就能出图。

### 3.4 Email 通知

**需求**：
- Pipeline 完成后通知 admin（邮件）
- 候选状态变化通知订阅者（"Top 10 又变了"）
- 密码重置邮件（当前只能 admin 帮改）

**方向**：SMTP（Gmail App Password / SendGrid / AWS SES） → `web/services/mailer.py`。

### 3.5 Multi-tenant（saas 化）

当前：一个 APS 实例 = 一个团队。

**假设要支持多个团队** 各自独立数据：
- 每个 Property / Candidate / User 加 `tenant_id` 列
- 所有 query 自动加 `filter(tenant_id=current)`
- Pipeline 按 tenant 各自配置 zone_allowlist / 搜索参数
- GIS 数据可共享（都是 AKL 的公共数据）

这是重构级别的改动（~1 个月），通常在有了付费客户后才启动。

---

## 4. 短期可做的小改进

| 改进 | 预计工作量 | 价值 |
|---|---|---|
| Rent 接入 Trade Me Rentals | 3-5 天 | 让 score_rent 真实；评分质量提升 |
| 加 Redis 做 CV 查询缓存（跨 pipeline run） | 2 天 | CV 查询不重复，每次 pipeline 减 50% time |
| shortlist 表格列持久化到 localStorage | 1 天 | 用户体验改善（每次刷新不重置列宽 / 顺序） |
| Pipeline 错误通知（Slack webhook） | 半天 | 运维看得到 pipeline 失败 |
| Admin "impersonate user" debug 功能 | 1 天 | 排查"某用户看不到 X" 问题 |
| 候选地址输入 autocomplete（基于 nz-addresses） | 2 天 | 减少用户手输错 |
| Candidates "compare" 多选对比 | 2 天 | 选 2-3 个候选做并列比较视图 |
| Export CSV（当前只有 xlsx） | 半天 | 有些用户要 CSV 导入其他系统 |

---

## 5. 开放问题 / 待调研

### 5.1 Trade Me 反爬升级

如果 Trade Me 开始给 `api.trademe.co.nz/v1/search/...` 加验证码或更严的反爬：
- 选项 A：Playwright 渲染 + 解析 NGRX `frend-state`（已有代码 fallback，默认不启用）
- 选项 B：申请 Trade Me 开发者 API（要 token，调用限额）
- 选项 C：换数据源（OneRoof / realestate.co.nz）

### 5.2 GIS 数据更新

- Auckland Council / LINZ 数据每年更新 1-2 次
- 当前**手动下载 + rsync 到 VM**
- 可以考虑 pipeline 里加一个 `check_gis_version.py`，对比本地 gpkg 的时间戳和官方数据发布日，提醒更新

### 5.3 租金数据质量

即使接入 Trade Me Rentals：
- Auckland 郊区样本稀疏（某 suburb 一个月只 1-2 条 3BR listing）
- 租赁中介的 listing price 和实际成交价有差距
- 可以考虑用 Tenancy Services 的 bond data（每季度公开）交叉验证

### 5.4 Parcel 匹配失败率

当前 `data_warnings=PARCEL_NOT_FOUND` 的比例：日志里看是个位数百分比。
- 新填海 / 新分割 parcel → LINZ nz-parcels.gpkg 可能滞后 1-3 个月
- 地址在海上 / 错误 geocoding → 本身就是数据异常

**改进**：对 PARCEL_NOT_FOUND 的 listing 做二次 geocoding（`nz-addresses` match，而不是直接用 Trade Me lat/lon）。

### 5.5 Reject cache 是否过时

桌面版 `被淘汰的listingid` 和云端 `properties where reject_reason is not null` 都是 **永久缓存**。
- 如果 `config.json zone_allowlist` 改了 → 原来 `ZONE_NOT_ALLOWED:19` 的 listing 应该重审
- 当前代码**不会自动**重审；要手动清缓存（桌面版删 sheet；云端用 reenrich）

**改进**：reject cache 带 schema version，version 变了自动重审。

---

## 6. 性能边界

### 6.1 当前规模

- 桌面版：一次 pipeline 几百到几千条 listing，20-60 分钟
- 云端版：每小时 pipeline 几十条 new listing，typical run < 3 min
- DB size: properties ~几万行 → SQLite file 几十 MB + 缩略图 ~1-2 GB

### 6.2 瓶颈

- **Trade Me 抓取**（最慢）：每页 500ms-1s，几百条 listing = 几十秒
- **GIS Stage 1**：每 listing 平均 100-300ms；用 6 workers 并行 = 几秒
- **GIS Stage 2 parcel outline render**：每 listing 200-500ms（matplotlib + hillshade）
- **CV 查询**：每条 200-500ms（网络），但线程池并发可以隐藏

### 6.3 Scale 方向

如果哪天 listing 量涨 10×：
1. 先加 CV 查询缓存（跨 run）
2. 再看 Stage 1 GIS 加 workers 或拆分地理区域并行
3. 再看 DB 切 Postgres + 多 worker

---

## 7. 参考

- `README.md` §14 Roadmap
- `findings.md` — 未解决的技术债条目
- `docs/superpowers/specs/*.md` — 已归档的设计决策（项目内 private）
- `Akl Property Shortlist - Delivery Introduction.md` — 客户交付介绍
- `AklPropertyShortlist_Release_Postmortem_2026-02-12.docx` — 2 月发布的复盘（含踩坑和下次改进）
