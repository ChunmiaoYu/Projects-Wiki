# UI 重设计 — Dashboard 入口 + 暗色专业风格 + 分层展开

> Spec date: 2026-04-29 NZ 周三
> 五专家评审通过: 软件工程 / CC AI / 云部署 / 美股期权 / IB-TWS

---

## 1. 目标与背景

### 1.1 项目目标
将 `frontend/index.html`（2524 行单文件 SPA，3 tab：新建 / 队列 / 详情）重设计为面向**私募 + 散户**双类客户的暗色专业 dashboard，承载 7 个模块 + drill-down 渐进披露 + "+新建"弹窗。

### 1.2 触发因素
- 当前 UI（vm-dev-frontend-a19-shipped.png）1920×1080 下底部 65% 留白（finding `F-2026-04-25-FRONTEND-ENTRY-BOTTOM-UNDERFILL-REVIEW`）
- 持仓 / Agent 2 review 决策 / 风险监控 / PnL 曲线全部不可见
- 客户面对的核心信息（在场资金、当前持仓、最近决策）需要在登录时立刻可见

### 1.3 客户画像
- **A 私募基金 / 资管经理**：全天盯盘，要快速决策、看深层数据、自己配置策略
- **B 个人散户 / 长尾客户**：看一天一两次，要信任感 + 解释性，不被复杂数据淹没

两类客户共用同一产品，调和策略：**分层展开 drill-down**（默认简洁，每行右"展开 ▾"露出深层数据）。

---

## 2. 核心设计决策（五项）

| 决策点 | 选择 | 理由 |
|---|---|---|
| 客户画像 | A 私募 + B 散户 | 两类同一产品共用，分层展开兼顾 |
| 调和策略 | 分层展开 drill-down | 一套设计两套深度，最 lean |
| 入口 | Dashboard | 在场资金 + PnL 是双方共同核心 |
| Dashboard 模块 | 7 个：账户 / 持仓 / 队列 / Agent2 timeline / 风险监控 / PnL 曲线 / 历史统计 | 私募 + 散户都需要的最小完整集 |
| 布局 | 上窄下宽（3 row, 1:2:1 / 1:1 / 2:1） | §17 数据密集每行填满；宽内容（持仓表/PnL/历史）放对应宽位置 |
| 视觉风格 | 暗色专业（深蓝灰 #0f172a + 青色 #06b6d4 + 红绿盈亏） | Bloomberg/TradingView 一派；私募盯盘不刺眼 + 散户觉得"高级" |
| Tab 结构 | Dashboard / 持仓 / 队列 / 详情 + 顶部右"+新建"按钮（弹窗） | 节省 tab 槽位；"+新建"作为"快速记一笔意图"动作 |

---

## 3. 信息架构

### 3.1 Tab 结构

```
顶部 nav: [期权交易助手 logo]  [Dashboard★] [持仓] [队列] [详情]   [+新建机会] [账户切换]
```

- **Dashboard**（默认入口）：7 模块仪表盘
- **持仓**：所有 OPEN 持仓的 expanded 列表 + 单笔详情侧栏（drill-down 永久展开）
- **队列**：所有 opportunity 按生命周期 grouped（DRAFT / SUBMITTED / EXECUTING / OPEN / CLOSED / FAILED）
- **详情**：drill-down 进单个 opportunity 的完整状态 + Agent 2 决策完整 trace + 修改链
- **"+新建机会"**：弹窗（不占 tab 槽位）

### 3.2 路由

```
/             → /dashboard (默认重定向)
/dashboard    → Dashboard 7 模块
/positions    → 持仓 tab
/queue        → 队列 tab
/opportunities/{id}    → 详情 tab
```

---

## 4. Dashboard 模块详细规范（7 个）

### 4.1 布局

```
Row 1 (1:2:1):  [账户卡片]    [当前持仓表]    [队列摘要]
Row 2 (1:1):    [Agent 2 timeline]   [风险监控]
Row 3 (2:1):    [PnL 曲线]           [历史统计]
```

**响应式**: 1920×1080 主屏。<1280px 触发 2 列折行（v2 不做）。

### 4.2 账户卡片

| 字段 | 来源 | drill-down |
|---|---|---|
| 账户 ID | IBKR managedAccounts | — |
| 净值 | reqAccountSummary NetLiquidation | — |
| 可用资金 | reqAccountSummary AvailableFunds | — |
| 购买力 | reqAccountSummary BuyingPower | — |
| 今日 PnL | DailyPnL（pnlSingle 聚合） | hover 看分时图 |

### 4.3 当前持仓表

每行显示：`symbol leg`（如"AAPL 220C 0428"）/ `qty` / `avg_cost` / `unrealized_pnl` / `展开 ▾`

**展开后露出（drill-down）**:
- Greeks（**P0-OPT-1 必须含普通话解释**）：
  - `Δ +0.5` → "**价格敏感度**: underlying 涨 $1, 期权涨 $0.50"
  - `Γ +0.08` → "**敏感度的加速度**: underlying 涨 $1, Δ 增 0.08"
  - `Θ -$45` → "**时间衰减**: 每天少赚 $45（持仓时间是敌人）"
  - `Vega +0.21` → "**波动敏感度**: IV 涨 1%，期权涨 $0.21"
- 当前 IV vs 入场 IV 对比
- TP/SL 设置 + 触发距离
- Agent 2 当前 review 决策（最近一条）

### 4.4 队列摘要

显示进行中 opportunity 数量 + 状态条（DRAFT / SUBMITTED / EXECUTING）+ 最近 3 条标题。点击单条 → 跳"详情"tab。

### 4.5 Agent 2 timeline（最近 5 条）

每条：`HH:MM action ON symbol — 信心 X — "rationale 摘要"`

action 颜色:
- `HOLD` 灰
- `PARTIAL_CLOSE` 黄
- `FULL_CLOSE` 绿
- `ADJUST_STOP` 橙

**drill-down**: 点击单条 → 弹出 modal 显示完整 trace + bundle JSON（私募 24h 历史在 v2，本版本 5 条够）。

### 4.6 风险监控

账户级 Total Greeks（净 Δ / Γ / Θ / Vega，散户友好普通话见 4.3）+ 最大亏损（已挂在场金额 / 净值占比） + 警报：
- 临近到期（≤7 天）红色
- 超过单仓位风控阈值（>5% 净值）橙色
- OPEN position 超 5 min 没新 Agent 2 review 决策（review_fn cycle 故障）灰色

### 4.7 PnL 曲线

切换：今日（分时）/ 7d / 30d。SVG line chart + gradient fill。今日数据来源 `pnl_snapshots` 表（worker 写, API 读, 见 §6）。

### 4.8 历史统计

- 胜率（已平仓 OPEN→CLOSED with realized_pnl）
- 平均盈亏比（赢:亏的金额比）
- 最大回撤（按 `pnl_snapshots` 滚动算）
- 本月 PnL（month-to-date 已实现 + 未实现）

---

## 5. 视觉规范

### 5.1 配色

| 用途 | 颜色 | 备注 |
|---|---|---|
| 背景 page | `#0f172a` | slate-900 |
| 背景 card | `#1e293b` | slate-800 |
| 边界 | `#334155` | slate-700 |
| 文字 primary | `#f1f5f9` | slate-100 |
| 文字 secondary | `#94a3b8` | slate-400 |
| accent / 焦点 | `#06b6d4` | cyan-500 |
| 盈利绿 | `#22c55e` | green-500 |
| 亏损红 | `#f87171` | red-400 |
| 警告黄 | `#fbbf24` | amber-400 |

### 5.2 字体

- 中文：`-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif`
- 数字：`"SF Mono", "JetBrains Mono", monospace`（数字等宽对齐）
- Heading 13px / Body 11px / Label 10px

### 5.3 spacing

- card padding: 8px 内 / 16px 外
- grid gap: 8px
- row gap: 8px

---

## 6. 数据流

### 6.1 实时 PnL（解 finding `B4` 跨进程问题）

**前置依赖（spec 实施前必做）**:
- 新建表 `pnl_snapshots(position_id, ts, unrealized_pnl, daily_pnl)`，worker 写每秒一条
- API 加 `GET /v1/dashboard/pnl?position_id=&since=` 读最近 N 秒
- Dashboard polling 每 1s 调上述 API

### 6.2 detail schema 修复（finding `F-2026-04-29-OPP-DETAIL-SCHEMA-DROPS-SPECS`）

**前置依赖**: `OpportunityDraftRead` 加 `take_profit_spec / stop_loss_spec / position_spec / max_risk_dollars` 字段，详情 tab drill-down 改用 schema 而非 DB 直查。

### 6.3 polling 频率

| 数据 | 频率 | API |
|---|---|---|
| 账户净值 | 30s | `GET /v1/account/summary` |
| 持仓 + PnL | 1s | `GET /v1/dashboard/pnl` |
| Agent 2 timeline | 5s | `GET /v1/agent2/decisions?limit=5` |
| 风险监控 | 30s | `GET /v1/dashboard/risk` |
| PnL 曲线 | 5s | `GET /v1/dashboard/pnl-curve?range=today` |
| 历史统计 | 60s | `GET /v1/dashboard/stats` |

### 6.4 drill-down Greeks（finding `F-2026-04-29-OPP-DETAIL-SCHEMA-DROPS-SPECS` 关联）

展开持仓行触发 `GET /v1/positions/{id}/greeks` → API 内部 reqMktData snapshot + 缓存 30s TTL（IB-P2 节流）。

---

## 7. 错误处理

| 场景 | UI 表现 |
|---|---|
| 网络断（API 失败） | 顶部红 banner: "网络断开, 数据可能 stale (上次更新: HH:MM:SS)" |
| IBKR 断 | 账户卡片 banner: "⚠ 实时行情中断, PnL 暂停更新" |
| 数据 stale > 60s | 模块右上角灰色 dot + tooltip "X 秒前更新" |
| Agent 2 review 失败 | timeline 该条标 "❌ 决策失败 — 点击看 trace" |
| 持仓 row IBKR 拿不到 Greeks | 展开后显示 "Greeks 数据暂不可用 (IBKR 限流或盘后)" |

---

## 8. 测试策略

### 8.1 e2e（invariant 22 gate）

新增 SC（在 `tests/e2e/`）：

| SC | 场景 | 断言 |
|---|---|---|
| SC-D01 | Dashboard 默认加载 7 模块 | 7 个 module 容器 visible + 文字 "Dashboard" 在 nav 高亮 |
| SC-D02 | 账户卡片显示净值 | 拿到 `$\d+,\d+` 格式 + 今日 PnL 颜色对（盈绿/亏红） |
| SC-D03 | 持仓表渲染当前持仓 | 至少 1 row + 展开按钮可点 |
| SC-D04 | 持仓 row 展开露出 Greeks 普通话 | 文字含 "时间衰减" / "价格敏感度" |
| SC-D05 | Agent 2 timeline 显示最近 5 条 | 5 row（或 < 5 时显示"暂无决策"） |
| SC-D06 | 风险监控 4 个 Greek 显示 | Δ/Γ/Θ/Vega 4 个 metric |
| SC-D07 | PnL 曲线 SVG 渲染 | `<polyline>` 元素存在 |
| SC-D08 | "+新建" 弹窗触发 | 点按钮 → modal 可见 + 输入框自动聚焦 |
| SC-D09 | 持仓 tab 跳转 | 点 nav "持仓" → URL `/positions` + 列表渲染 |
| SC-D10 | 队列 tab 状态 grouping | DRAFT / SUBMITTED / EXECUTING / OPEN / CLOSED / FAILED 6 组 |
| SC-D11 | 详情 tab drill-down | 点队列单条 → URL `/opportunities/{id}` + 完整 trace 加载 |
| SC-D12 | 1920×1080 视口占用率 ≥ 95% | screenshot 验证 (frontend-testing skill §空间密度) |

### 8.2 视觉回归

每个 SC 配 snapshot PNG 落 `tests/e2e/__snapshots__/`，frontend-testing skill §4.1 3 步硬门强制 (Read PNG + 5 条异常清单)。

### 8.3 单测

- 持仓表 drill-down 状态机（折叠/展开/记忆偏好）
- Greeks 普通话翻译纯函数
- pnl_snapshots polling 增量 merge

---

## 9. 实施阶段

### Phase 1: 前置依赖（spec 实施前必做）
1. 新建 `pnl_snapshots` 表 + alembic migration
2. Worker 加 `pnl_writer.py` 每秒写 snapshots
3. API 加 `/v1/dashboard/*` 6 个 endpoint
4. 修 `OpportunityDraftRead` schema 加 spec 字段

### Phase 2: 前端拆分（P1-SE-1）
原 `frontend/index.html` 拆成：
```
frontend/
├── index.html          (shell, < 200 行)
├── js/
│   ├── app.js          (router + 公共)
│   ├── dashboard.js    (Dashboard 7 模块)
│   ├── positions.js    (持仓 tab)
│   ├── queue.js        (队列 tab)
│   ├── detail.js       (详情 tab)
│   └── new-modal.js    (新建弹窗)
└── styles/
    ├── theme.css       (暗色变量)
    └── components.css  (复用组件)
```

### Phase 3: Dashboard 实施
按 4.1 layout 顺序实现 7 模块。drill-down 行为优先（P0-OPT-1 普通话解释必有）。

### Phase 4: 其他 tab
持仓 / 队列 / 详情 tab 重设计实施。

### Phase 5: e2e + 视觉回归
12 个 SC 全跑通 + snapshot 全审核。

---

## 10. v2 待做（本版本不做）

- 临近事件 calendar（财报 / FOMC，期权专家 P1）
- Agent 2 timeline drill-down 24h 完整历史
- 多账户切换（公司账户托管场景）
- "+新建" tab 模式（私募 power 用户多策略并行编辑）
- 移动端适配（<1280px 折行）
- 个人偏好记忆（drill-down 永久折叠/展开）

---

## 11. 五专家评审 P0/P1 issues 修订追溯

| ID | 来源 | 状态 | 修订位置 |
|---|---|---|---|
| P0-OPT-1 | 期权 | ✅ 已纳入 | §4.3 drill-down Greeks 普通话解释 |
| P1-SE-1 | 软件工程 | ✅ 已纳入 | §9 Phase 2 frontend 拆分 |
| P1-SE-2 | 软件工程 | ✅ 已纳入 | §6.2 detail schema 前置依赖 |
| P1-IB-1 / P1-SE-3 | IB-TWS / 软件工程 | ✅ 已纳入 | §6.1 pnl_snapshots 表前置依赖 |
| P1-CC-1 | CC AI | ✅ 本 spec 即修 | docs/superpowers/specs/2026-04-29-ui-redesign-design.md |
| P1-OPT-1 | 期权 | ⏸ v2 | §10 临近事件 calendar v2 |
| P2 (云部署/IB/期权 多项) | — | 见 §6 polling 频率 + §10 v2 | — |

---

## 12. Definition of Done

- [ ] Phase 1-5 全部 ship
- [ ] 12 个 e2e SC 全绿
- [ ] frontend-testing skill §4.1 3 步硬门走完（每个 PR 走一次）
- [ ] §17 数据密集 1920×1080 视口占用 ≥ 95%
- [ ] 私募 + 散户两类客户各跑 1 次 UAT 用户测试（v2 前可跳过，但记 finding）
- [ ] memory `project_ui_design_v2.md` 指针写入
- [ ] docs-hub 镜像 spec + 决策记录（§18 决策留痕）

---

## 13. 资源

- 当前 UI 截图: `vm-dev-frontend-a19-shipped.png`
- 持仓 UI brainstorming 历史: `task_plan.md` 持仓 UI 快照（部分纳入本 spec）
- e2e 框架基础: `tests/e2e/test_smoke_frontend.py` (5 SC 已有)
- frontend-design skill: `~/.claude/plugins/.../skills/frontend-design/`
