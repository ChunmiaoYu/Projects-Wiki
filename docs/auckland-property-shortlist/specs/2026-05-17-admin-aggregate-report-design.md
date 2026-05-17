# Spec — Admin 多用户 aggregate 报表 (2026-05-17, V1 北极星 #6)

> **状态**: SPEC-DRAFTED
> **北极星对应**: §4 #6 (Admin 多用户 aggregate 报表 — V1 必做项)
> **依赖**: §4 #5 (Candidate progress UI 编辑) — 报表数据源, **但不强阻塞**: progress 字段 enum 已 ship, 即使无 UI 编辑 admin 仍能看 `new` 默认分布
> **前置阅读**: `memory/project_vision_and_north_star.md` (5 问 cross-check), `web/db/models.py` (Candidate.progress / User.role), `web/routers/admin.py` (现有 5 Tab)

---

## 1. 背景 / Motivation

APS Web 平台当前 admin 面板已有 5 Tab Dashboard (Users / Candidates / Popular Properties / User Activity / Pipeline + Audit), 见 `web/routers/admin.py` + `web/templates/admin.html`. **缺口**: admin (项目所有者) 没法一眼看清"全 6 family-and-friends 用户的 candidate 决策状态分布":

- 谁还在 `research` 阶段卡了很久?
- 全用户加总有多少 `offered` / `contracted` / `settled`?
- 哪个用户 `dropped` 最多 (筛房标准是不是有问题)?

**北极星 §4 #6 明确**: V1 必做"跨 6 user 决策状态 rollup, e.g. '全用户共 N offered / M contracted'". 是 V1 决策闭环 (北极星 5 问 Q1) 的关键可见性工具 — 没这个, admin 只能逐用户切 Candidates 页手工数, 决策闭环没闭口.

**当前已 ship 基础**: `Candidate.progress` 列已 enum (`new/research/offered/negotiating/contracted/settled/watching/dropped` 8 值, 见 `models.py:213`), 数据存得下来. 缺的是**聚合查询 + UI 呈现**.

---

## 2. 目标 + 非目标

### 2.1 目标 (V1 in-scope)

1. **跨用户决策状态矩阵** (rows = users, cols = progress 8 值, cells = count)
2. **行/列加总** (右侧 user 总数列 + 底部 progress 总数行)
3. **drill-down**: 点击矩阵单元 → 跳转到 Candidates 页, 自动加 `user_id` + `progress` filter (复用现有 `/api/admin/candidates`)
4. **归档过滤**: 默认排除 archived (跟 V1 北极星 "180 天 archived 默认隐藏" 一致), 可 toggle 显示
5. **角色显示**: 用户名旁标注 role (admin/premium/viewer), 帮 admin 区分 friends-and-family 6 人是谁
6. **空状态优雅**: 6 用户没 candidate 时显示提示, 不渲染空矩阵

### 2.2 非目标 (V2+ 推后)

- **时间序列趋势图** (e.g. "过去 30 天 contracted 增长曲线") — 需新数据模型 (progress_history 表), V2 加邮件通知 + 时间轴时一起
- **Candidates Excel 导出** — 当前只 shortlist 可导出 (export_logs.source='shortlist'), candidates 导出需额外 spec (V1 决策闭环不阻塞)
- **per-property 跨用户 analytics** — Popular Properties Tab 已 cover ("哪个 listing 最多人收藏"), 不重做
- **多租户隔离** — V1 全 admin 看全 candidate (含 notes), 商业化分级 V2 加 (在 forward compat hook 预留, 不实施)
- **报表 PDF/Excel 导出** — V1 用 admin 现有 5 Tab 一致 (UI 内查看 + Tabulator export 兜底)

---

## 3. 北极星 §3 5 问 cross-check

| # | 问题 | 答 | 偏离? |
|---|---|---|---|
| 1 | 这件事是否服务 Auckland 选房决策闭环? | **是** — 直接落地 §4 #6 "跨用户决策状态 rollup", 让 admin 看到 candidates 决策状态机的群体分布 (research → offered → contracted → settled), 是决策闭环可见性核心工具 | ✓ 不偏离 |
| 2 | 是否预留商业化能力位? | **是** — API 设计**不写死 `user_id=admin`**, 用 `require_role(["admin"])` 守门; future 加 `customer_admin` 角色只需扩 role enum + filter `WHERE tenant_id = current_user.tenant_id`; 矩阵 schema 通用 (`progress_counts: {enum_value: N}`), 不假设全租户共享 | ✓ 不偏离 |
| 3 | 是否预留远景"跨领域"扩展? | **是** — `Candidate.progress` 已 enum 8 值; 报表代码按 enum 动态生成列, V3+ 加 `loan_approved` / `contract_signed` / `renovation_started` / `tenant_secured` 等新阶段时**只改 enum + UI 列宽**, 报表 SQL 不动 (`GROUP BY user_id, progress` 自适应) | ✓ 不偏离 |
| 4 | 是否避免依赖人工 admin 手工操作? | **是** — admin 在 UI 内一键打开新 Tab 看到聚合, 不需 SSH 进 VM 跑 `SELECT user_id, progress, COUNT(*) FROM candidates GROUP BY ...`; drill-down 也走 UI, 不需手工拼 query | ✓ 不偏离 |
| 5 | 是否专注护城河, 不重做 Trade Me 已有功能? | **是** — Trade Me 只是 listing 源, 不知道用户私下决策状态; "跨用户决策 rollup" 是 APS 独有的决策状态机 + admin 视图, Trade Me 完全不做. 跟 §5 护城河 (GIS 富化 + 评分 + **决策状态机**) 直接对齐 | ✓ 不偏离 |

**5 问全过. 此设计与北极星目标关系**: 直接落地 §4 #6, 是 V1 决策闭环 (北极星 5 问 Q1) 的最后一块可见性补丁; 不引入新数据模型 / 不引入第三方依赖 / 不破坏 §5 forward compat hooks.

---

## 4. 数据模型变更

### 4.1 schema 变更: **无** (V1)

`Candidate.progress` 列已 enum 8 值 (`models.py:213`), 已 ix_candidates_user_id (`models.py:204`). 现有 schema 已满足 `GROUP BY user_id, progress` 的查询需求.

### 4.2 索引讨论

**现状**:
- `ix_candidates_user_id` (单列索引, `models.py:204`)
- 无 `(user_id, progress)` 复合索引

**评估**: V1 规模 6 用户 × 平均 ~20 candidates/user = ~120 行. SQLite 全表扫秒杀. 复合索引 V1 **不加** (KISS, 不预先优化).

**升级触发**: 用户增到 ~100, candidates 总量 ~5000+ 时, 加复合索引 `(user_id, progress)`. 加索引不需 spec, 直接迁移 (`_migrate_add_idx_candidates_user_progress`).

### 4.3 可选: `last_progress_at` 列 (推后 V2)

理想可加列 `Candidate.last_progress_at: datetime` 追踪"上次状态变更时间", 让报表加"卡 research 超 N 天"告警. **V1 不加**, 理由:
- 数据模型变更 = 跟 §4 #5 (Candidate progress UI 编辑) 同 spec 一起做, 节省 1 次 migration
- V1 报表 KPI 不需要时长 (只需 count)
- forward compat hook 已留: 加列时 ORM 兼容, 矩阵 API 不变

---

## 5. API contract

### 5.1 新增: GET `/api/admin/candidates/aggregate`

**Query params**:
- `include_archived: bool` (default `false`) — 是否计入 archived (V1 默认排除, 跟北极星 archived 默认隐藏一致)
- `exclude_inactive_users: bool` (default `true`) — 是否排除 `User.is_active=false` 的用户行

**Auth**: `require_role(["admin"])` (现有依赖, 见 `web/auth/deps.py`)

**Response schema**:

```json
{
  "users": [
    {
      "user_id": 1,
      "identifier": "admin@aps.local",
      "display_name": "Yumiao",
      "role": "admin",
      "is_active": true,
      "progress_counts": {
        "new": 3,
        "research": 5,
        "offered": 1,
        "negotiating": 0,
        "contracted": 0,
        "settled": 0,
        "watching": 2,
        "dropped": 4,
        "_null": 1
      },
      "total": 16
    },
    {
      "user_id": 2,
      "identifier": "alice@example.com",
      "display_name": "Alice",
      "role": "premium",
      "is_active": true,
      "progress_counts": { "new": 8, "research": 2, "offered": 0, ... },
      "total": 12
    }
  ],
  "totals": {
    "new": 11,
    "research": 7,
    "offered": 1,
    "negotiating": 0,
    "contracted": 0,
    "settled": 0,
    "watching": 2,
    "dropped": 4,
    "_null": 1,
    "_all": 26
  },
  "progress_order": ["new", "research", "offered", "negotiating", "contracted", "settled", "watching", "dropped", "_null"],
  "filters_applied": {
    "include_archived": false,
    "exclude_inactive_users": true
  }
}
```

**说明**:
- `_null` bucket: progress 字段尚未设值 (旧数据 / 用户没编辑) 的 candidates. UI 列名 "(未设置)".
- `progress_order`: 后端定义稳定排序, 前端按此顺序渲染列. V3+ 加新 enum 值时只追加, 不破坏前端.
- `_all`: 列总数兜底 (前端表底 "全部用户共 N candidates").
- archived 判断: candidate 没有 `archived_at` 列, 通过 join 对应 `Property.archived_at` 判断 (按 `listing_id`); 手工 candidates (无 listing_id, address_input 录入) 不归档, 始终计入.

### 5.2 drill-down: **复用** GET `/api/admin/candidates`

现有 endpoint `/api/admin/candidates` (`admin.py:307`) 已支持 `user_id` query 参数 + `filter[col]=val` 通用过滤 (`admin.py:319`). 矩阵单元点击 → 前端跳转:

```
/admin?tab=candidates&user_id=2&filter[progress]=offered
```

**不另起新 endpoint**, 节省代码 + 跟现有 5 Tab 行为一致.

### 5.3 erorr handling

- 非 admin 调用: 403 (`require_role` 守门)
- DB 错误: 500 + log
- 0 用户 0 candidate: 返回 `{users: [], totals: {_all: 0, ...}, ...}` (前端渲染空状态提示)

---

## 6. UI 设计

### 6.1 新增第 6 Tab: "决策汇总" (Decision Summary)

`web/templates/admin.html` 在现有 5 Tab 后追加 "决策汇总" Tab:

```
[Dashboard] [Users] [Candidates] [Popular Properties] [User Activity] [Pipeline] [决策汇总]
                                                                                  ^^^^^^^^^ NEW
```

### 6.2 矩阵 table (text mockup)

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│ 决策汇总 (Decision Summary)                                       [☐ 含归档]  [刷新]  [更新于 X]   │
├──────────────────────────────────────────────────────────────────────────────────────────────────┤
│ User              | Role    | new | research | offered | negot. | contr. | settled | watch | drop | (未设) | 总  │
├───────────────────┼─────────┼─────┼──────────┼─────────┼────────┼────────┼─────────┼───────┼──────┼────────┼────│
│ admin@aps.local   | admin   |  3  |    5     |    1    |   0    |   0    |    0    |   2   |  4   |   1    | 16 │
│ alice@ex.com      | premium |  8  |    2     |    0    |   0    |   0    |    0    |   0   |  2   |   0    | 12 │
│ bob@ex.com        | premium |  0  |    0     |    0    |   0    |   1    |    0    |   0   |  0   |   0    |  1 │
│ carol@ex.com      | viewer  |  0  |    0     |    0    |   0    |   0    |    0    |   0   |  0   |   0    |  0 │  ← 空状态行
│ ...                                                                                                          │
├───────────────────┼─────────┼─────┼──────────┼─────────┼────────┼────────┼─────────┼───────┼──────┼────────┼────│
│ 全部 (6 users)    |   —     | 11  |    7     |    1    |   0    |   1    |    0    |   2   |  6   |   1    | 29 │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 视觉细节

- **暗色主题**: 沿用 `admin-container` / `kpi-card` 现有 CSS variables (`--bg-card` / `--border` / `--accent` / `--text-primary`)
- **Heatmap 着色** (可选, V1 简单实现): cell count > 0 时按颜色梯度上色 (0 = 灰; 1-2 = 淡蓝; 3-5 = 中蓝; 6+ = 亮蓝). 颜色仅作辅助, 数字仍为主. V1 简单做法: 用 CSS `background: rgba(56, 189, 248, calc(count * 0.1))` linear scale; 6+ 触顶
- **行高亮**: 总计行加粗 + 边框上方 2px solid accent
- **空 cell**: 显示 `0` 但灰色 (`color: var(--text-muted)`), 不空字符避免误判
- **空用户**: 用户 0 candidate 时仍显示该行 (全 0), 帮 admin 看到"alice 这周完全没活动" 信号

### 6.4 交互

- **点击 cell** (count > 0): 跳转 `/admin?tab=candidates&user_id=X&filter[progress]=Y` — 切到 Candidates Tab + 自动应用 filter
- **点击 user 名** (左侧首列): 跳转 `/admin?tab=candidates&user_id=X` — 看该用户全 candidates
- **点击 progress 列头**: 跳转 `/admin?tab=candidates&filter[progress]=Y` — 看全用户该 progress 阶段
- **`☐ 含归档` toggle**: 触发 API 调用 `?include_archived=true`, 重渲染矩阵
- **`刷新` 按钮**: 调用 API 重新拉数据 (非自动轮询, 数据低频变更; SSE 不需要)
- **`更新于 X`**: 显示上次拉取时间 (本地时间 HH:MM:SS, 用户能判断 stale)

### 6.5 i18n

沿用 `base.html` 中英文 toggle, 标签对照:

| 中文 | English |
|---|---|
| 决策汇总 | Decision Summary |
| 含归档 | Include Archived |
| 刷新 | Refresh |
| 全部 | Total |
| 角色 | Role |
| (未设置) | (Unset) |

Progress 值本身保留英文 (跟 §4 #5 Candidate progress UI 编辑 spec 对齐, 不在本 spec 翻译).

---

## 7. Forward compat hooks (北极星 §5 对齐)

| Hook | 防的偏离 | 本 spec 落地 |
|---|---|---|
| `Candidate.progress` enum 8 值 | 防硬编码 if-elif 业务逻辑; 加状态只改 enum + UI | 报表 `GROUP BY progress` SQL 不假设 8 值固定; `progress_order` 后端动态生成 |
| `User.role` enum (`admin/premium/viewer`) | 防"全用户同权限"; 商业化时 premium-only 查 role | API 用 `require_role(["admin"])` 守门; future `customer_admin` 角色加只需扩 list |
| `web/auth/` 独立模块 | 防 SSO 升级要改 router | 复用现有 `require_role`, 不嵌入新 auth 逻辑 |
| SQLAlchemy ORM 必走 | 防 PG 迁移要全文 sed | 用 `db.query(Candidate).group_by(Candidate.user_id, Candidate.progress)`, 不写 raw SQL |
| `archived_at` nullable | 防数据 lifecycle 不可逆 | `include_archived` 默认 false, 跟 shortlist API 行为一致 |
| 多租户预留 | V1 无, V2 商业化加 | API 不假设全局可见; 加 `WHERE tenant_id = X` 时矩阵 schema 不变 |

**新加 hook (本 spec 引入)**:
- **`progress_order` 后端导出**: 前端按后端给定顺序渲染列, 加新 enum 值时**不改前端代码**. 录入北极星 §5 表.
- **drill-down 复用现有 candidates API**: 不另起 schema. 录入北极星 §5 表.

---

## 8. 风险 / Open questions

### 8.1 已决定 (本 spec 内 close)

| Q | 决定 | 理由 |
|---|---|---|
| archived 是否计入? | **默认排除**, 可 toggle | 跟北极星 "180 天 archived 默认隐藏" 一致 |
| watching / dropped 是否单独列? | **单独列** | 数据保真, admin 自己看出 "dropped 多 = 筛房标准有问题" 信号 |
| 复合索引是否本 spec 加? | **不加** (V1 规模 ~120 行无需要) | KISS, 不预先优化 |
| 报表是否要导出 Excel? | **不导** (V1) | Candidates Excel 导出本身就是另外议题, 这里看 UI 够用 |

### 8.2 留 V2 / 后续 spec (Open)

1. **跨用户隐私 — admin 能看 viewer 用户的 candidate notes?**
   - V1 简单 = 全可见 (单 tenant friends-and-family)
   - V2 商业化分级时考虑: viewer-only candidate notes 可能要隐藏 (privacy by default)
   - 留给 §4 #10 (Billing / 订阅基础设施) spec 决
2. **报表 stale 时长? 是否要后端 cache?**
   - V1 不 cache (~120 行 SQLite GROUP BY < 10ms)
   - 用户 ~100 时 reconsider, 但实测过 50ms 才 cache
3. **`last_progress_at` 列是否要本 spec 同时加?**
   - 倾向**不加** (V1 报表用不到, 跟 §4 #5 spec 一起做节省 migration)
   - 但若 §4 #5 spec 先 ship 已加了, 本 spec API 可顺手返回 "最久未更新" KPI
4. **drill-down 是 modal 还是路由跳转?**
   - 倾向**路由跳转** (复用 Candidates Tab 全部 UI + filter popover)
   - modal 会复制 Candidates 表格代码 = 反 DRY
5. **空用户行是否显示?** (e.g. carol 0 candidate)
   - 倾向**显示** (帮 admin 看到 "carol 这周没活动" 信号, 不是 noise)
   - V1 实测后若 6 用户都常空 → 可加 `?hide_empty_users=true` toggle

---

## 9. Acceptance criteria

V1 完成判定 (DoD):

- [ ] 新 API `GET /api/admin/candidates/aggregate` 返回正确矩阵 schema (§5.1 JSON)
- [ ] `?include_archived=true/false` 行为正确 (默认 false 排除有 `archived_at` 的 listing_id)
- [ ] 非 admin 调用返回 403
- [ ] admin.html 第 6 Tab "决策汇总" 显示矩阵 + 总计行 + 总计列
- [ ] cell 点击跳转到 Candidates Tab + 自动应用 user_id + progress filter
- [ ] i18n 中英文 toggle 正确
- [ ] **pytest 1 集成测**: 建 3 user × 不同 progress candidates, 调 API 断言矩阵正确
- [ ] **Playwright 1 e2e** (frontend-testing skill 触发, 含截图 review 5 条异常清单): admin 登录 → 切第 6 Tab → 矩阵渲染 → 点 cell 跳转 → URL 含 filter param
- [ ] 北极星 §6 历史决策表追加一行 ("2026-05-?? Admin aggregate 报表 SHIPPED-COMMIT-XXX")
- [ ] 北极星 §4 中间路径 #6 状态从 IDEA → SHIPPED-COMMIT-XXX
- [ ] 北极星 §5 forward compat hooks 表追加 2 行 (`progress_order` 后端导出 + drill-down 复用 candidates API)

---

## 10. 粗估 (AI 速度)

| 子任务 | 估时 |
|---|---|
| 后端 API `/api/admin/candidates/aggregate` + ORM query + filter 逻辑 | ~15-25 min |
| pytest 集成测 (fixture 建 3 user + N candidate, 断言矩阵) | ~10-15 min |
| 前端 admin.html 新 Tab + 矩阵 table + 着色 + i18n | ~25-40 min |
| 前端 JS: fetch API + 渲染 + 点击跳转 + toggle | ~15-25 min |
| Playwright e2e + 截图 review | ~15-25 min |
| 北极星 memory 更新 (§4 / §5 / §6) | ~5 min |
| **合计** | **~85-135 min (~1.5-2.25 hr)** |

**全 AI 速度, 不含真实 deploy + 跨 session block**. Subagent 可并行 (后端 + 前端 + 测试 3 路并行约 ~40 min 总).

---

## 相关文档

- 北极星: `memory/project_vision_and_north_star.md` §4 #6
- 现有 admin: `web/routers/admin.py` + `web/templates/admin.html`
- 数据模型: `web/db/models.py` (Candidate + User)
- 总纲领: `docs/platform/00-overview.md`
- 项目档案: `docs/project-cloud-profile.md`
- 关联 spec (并行 V1 必做): §4 #5 Candidate progress UI 编辑 (尚未 spec)
- 关联 spec (并行 V1 必做): §4 #7 用户 onboarding 自助 (尚未 spec)
