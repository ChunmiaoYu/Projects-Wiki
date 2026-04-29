# 决策记录 — UI 重设计 v2(Dashboard 入口 + 暗色专业风格 + 分层展开)

> **归档位置**:`docs-hub/docs/options-trader/decisions/2026-04-29-ui-redesign.md`(public)
> **日期**:2026-04-29
> **参与者**:用户 + Claude + 五专家(软件工程 / CC AI / 云部署 / 美股期权 / IB-TWS)
> **相关 spec**:[2026-04-29-ui-redesign-design.md](../specs/2026-04-29-ui-redesign-design.md)
> **相关原始讨论**(internal,客户不可见):项目 repo `docs/superpowers/discussions/2026-04-29-ui-redesign-brainstorm.md` + `docs/superpowers/discussions/2026-04-29-step-f-collector-chain-research.md`

---

## 触发的 §18 条款

- **①** 触发五专家 brainstorming(项目 CLAUDE.md §8 全 5 位专家 review approve)
- **⑤** 数据模型 / API 契约重大变更(新增 `pnl_snapshots` 表 + 6 个 `/v1/dashboard/*` endpoint + `OpportunityDraftRead` schema 加 4 个 spec 字段)

---

## 背景(为什么要改)

旧 UI 是 2524 行单文件 React SPA,3 个 tab(新建机会 / 客户队列 / 机会详情),浅色风格博客式布局。两件事让它撑不住客户演示:

1. **底部大片留白**:1920×1080 视口下内容只占顶部 ~35%,下方 ~65% 全空。客户打开看到空荡荡的页面会怀疑系统没在工作。这是 §17 数据密集 UI 不该用窄 max-width 的反例。
2. **核心信息不可见**:持仓 / Agent 2 决策 timeline / 风险监控 / PnL 曲线 / 历史统计——这 5 个客户登录最想第一眼看到的东西全都没有,只能在"客户队列"tab 里翻"机会详情"才能看到一点点散在状态字段里的碎片信息。

不改的后果:**演示当天客户登录看不到自己的钱在干嘛**,只能看到一个新建表单。私募基金客户(全天盯盘需求)和散户(偶尔看一眼需要信任感)两类都不满意。

---

## 决策内容(改了什么)

### 视觉

1. **暗色专业 palette**:`#0f172a` page bg + `#06b6d4` cyan accent + 红绿盈亏(Bloomberg/TradingView 一派),长时间盯盘不刺眼 + 散户觉得"高级"
2. **3 行紧凑 grid**:`1:2:1 / 1:1 / 2:1` 列比,viewport 高度撑满 95%+,数字密集监控页风格

### 结构

3. **Dashboard 作为默认入口**(新增 tab),取代原"新建机会"作首页。Dashboard 包 7 个模块:账户 / 当前持仓 / 队列摘要 / Agent 2 timeline / 风险监控 / PnL 曲线 / 历史统计
4. **4 tab 改为**:Dashboard / 持仓 / 队列 / 详情 + 顶部右"+新建机会"button(弹窗,不占 tab 槽位)
5. **持仓表 drill-down 分层展开**:默认每行简洁(symbol / qty / cost / unrealized PnL / 展开 ▾),点开露出 Greeks + TP/SL spec + 最近 Agent 2 决策

### 客户友好

6. **Greeks 普通话翻译**(P0-OPT-1):
   - Δ +0.5 → "**价格敏感度**:underlying 涨 $1, 期权涨 $0.50"
   - Γ +0.08 → "**敏感度的加速度**:underlying 涨 $1, Δ 增 0.08"
   - Θ -$45 → "**时间衰减**:每天少赚 $45 (持仓时间是敌人)"
   - Vega +0.21 → "**波动敏感度**:IV 涨 1%, 期权涨 $0.21"

   这是项目 CLAUDE.md §12 "面向中国客户, 客户可见内容必须中文" 的硬要求

### 工程

7. **不引入 bundler**:仍走 `<script type="text/babel">` per-file 模型 + `@babel/standalone` 浏览器编译。`window.OET.*` 命名空间挂组件实现跨文件协作。理由:§5 Simplicity First, 现项目无 build step, 引入 bundler 是侵入级改动
8. **frontend 拆分**:`index.html` 从 2524 → 37 行 shell + `js/runtime.jsx` (公共 hooks) + `js/router.jsx` + `js/dashboard.jsx` + 7 个 `js/modules/*.jsx` + 3 个 `js/tabs/*.jsx` + `js/new-modal.jsx` + `js/legacy-composer.jsx` (临时,v2 重写)

### 测试

9. **12 个 e2e SC** 全过(SC-D01 ~ SC-D12),包括 P0 验证项:
   - SC-D04: 展开持仓行 Greeks 普通话翻译可见(或 fallback "Greeks 数据暂不可用")
   - SC-D12: 1920×1080 视口占用率 ≥ 95%(§17 硬闸)
   - 其余 10 条覆盖 7 模块 + 4 tab + +新建 modal 路由

10. **invariant 22 e2e gate** 已 enforce(2026-04-24 落地, frontend 改动 commit 前必跑)

### 后端配套(Plan 1)

11. **新增 `pnl_snapshots` 表**(每秒 worker 写一行 per OPEN position),解 finding B4(B4: `/v1/monitoring/live-pnl` 跨进程返 `[]`)
12. **加 6 个 `/v1/dashboard/*` endpoint**:`/pnl` `/account` `/pnl-curve` `/stats` `/risk` `/agent2/decisions`
13. **加 `/v1/dashboard/positions` 聚合 endpoint**(Position 表 LEFT JOIN 最新 `pnl_snapshot.unrealized_pnl` + 派生 contract_label / TP/SL summary)
14. **`OpportunityDraftRead` schema 加 4 个字段**(`take_profit_spec` / `stop_loss_spec` / `position_spec` / `max_risk_dollars`)解 detail tab drill-down 拿不到 spec 的 finding

具体实现细节见 spec 文档。

---

## Why(为什么选这个方案,不选别的)

### 调和私募 + 散户两类客户:分层展开 vs 双产品

- **选项 A(最终选)**:**单一 dashboard + 分层展开 drill-down**——默认简洁,每行右"展开 ▾"露出深层数据
  - 优点:1 套设计 2 套深度;开发量 1 倍;客户切换无认知负担
  - 缺点:私募 power 用户多策略并行场景下展开多行视野零散
- **选项 B(备选)**:专业版 / 简版 双产品(类似 TradingView Pro)
  - 优点:每类用户专属体验
  - 缺点:开发 2 倍 + 维护双倍 + 测试 2 套, 当前期不值
- **决策**:选 A;v2 给私募加"+新建" tab 模式 + drill-down 24h 历史(§10 v2 待做)

### 不引入 bundler

- **选项 A(最终选)**:维持 `<script type="text/babel">` + `window.OET.*`
  - 优点:0 build step, 部署还是 `StaticFiles(directory=frontend_dir, html=True)`,无侵入;调试改 .jsx 即时刷新
  - 缺点:babel-standalone 在浏览器编译, 14 个 .jsx 串行加载,首屏比 bundler 慢 ~200ms;无 TypeScript
- **选项 B(备选)**:Vite + esbuild + ESM imports
  - 优点:tree-shaking + TypeScript + dev server HMR
  - 缺点:本期纯 UI 重设计目的不是优化首屏 200ms,加 bundler 引入 deploy.sh + CI build step + npm 依赖,远超 ROI
- **决策**:选 A;v2 真要做产品化前考虑 bundler(留 finding 不阻塞)

### Dashboard 入口 vs "新建机会"入口

- **选项 A(最终选)**:Dashboard 默认入口,新建走顶部 "+新建" button 弹窗
  - 优点:登录第一眼看到资金状态 + 持仓 + Agent 2 决策——这是双类客户共同核心
  - 缺点:重度产品化场景(私募一天开 30 单)新建被弹窗限制,不能多笔并行编辑
- **选项 B(备选)**:保留"新建机会"作 tab,Dashboard 加为第 4 tab
  - 优点:私募 power 用户多笔并行
  - 缺点:演示时打开看到空表单的"系统在干嘛"问题不解
- **决策**:选 A;v2 给"+新建" tab 模式开关满足 power 用户

---

## 风险和回滚

### 已知风险 1:legacy-composer.jsx 临时占位 922 行

- 描述:Task 9 +新建 modal 嵌入 LegacyComposer(机械搬迁原 ComposerView + 15 个依赖组件), AppCtx.Provider 已 wrap, 功能正确但样式未对齐暗色 theme(modal 内部分元素仍用旧 light theme CSS variables, hardcode `#fff` 已改 `var(--bg-card)` 但完整 audit 未做)
- 应对:留 finding `F-2026-04-29-NEW-MODAL-AUTOCLOSE-MISSING` (P3) + v2 follow-up "ComposerView 重写新 spec 风格"
- 回滚:删 `js/new-modal.jsx` script tag,新建按钮 set state 无对应组件 short-circuit, button 点击无反应(降级为只看不能新建)

### 已知风险 2:Plan 1 几个 endpoint 是 stub

- `/v1/dashboard/account` 返默认 0 + `_warning: "v2 待接入 (类似 pnl_snapshots)"`,前端识别 `_warning` 显示提示
- `/v1/dashboard/risk` Greeks 聚合 stub(Plan 1 设计错误:Position.raw_payload 顶层无 Greeks),返默认 0 + warning
- 应对:前端识别 `_warning` 字段显示 "v2 待接入" 而非假装显示有效数据(后者比无更危险);v2 加 `position_greeks_snapshots` 表 + `account_snapshots` 表 类似 `pnl_snapshots` 模式
- 回滚:不需要,这些是 stub 不是 broken

### 已知风险 3:Dashboard 7 模块 polling 频率组合

- 账户 30s / 持仓 5s / pnl 1s / Agent2 5s / 风险 30s / 曲线 5s / 统计 60s
- 单页 polling 6-7 路并发, IBKR 限流可能压;但当前实现都走 `pnl_snapshots` 表(已聚合,不直接调 IBKR),只有 risk 算 net Greeks 时未来会真调 IBKR
- 应对:每模块 stale dot 显示数据 freshness, 前端识别 stale 状态;v2 真接入 IBKR 时再做限流策略(`60s avg <= 50 msg/s`)
- 回滚:任一模块禁用 → fallback PlaceholderCard "暂未实现"

### 已知风险 4:VM DEV 部署因 CI 失败不到位(2026-04-29 暴露)

- Plan 2 commits 全部 push 到 origin/dev 但 deploy-dev.yml 因 `tests/e2e/test_dashboard.py` import playwright 在 CI 没装 → ImportError → test job exit 1 → deploy job skipped
- VM DEV 当前仍是 5 天前的旧 UI commit,Plan 1 + Plan 2 都没真上线
- 应对:补 1 行 `deploy-dev.yml` 加 `--ignore=tests/e2e`(已本地 commit 待 push)
- 回滚:回滚 deploy-dev.yml 那一行 → CI 再次 fail → VM 不会变(无破坏)

---

## 涉及文件(对内文件清单, 客户可跳过)

- 前端:`frontend/index.html` (2524→37) + `frontend/js/{runtime,router,dashboard,new-modal,legacy-composer}.jsx` + `frontend/js/modules/{account-card,positions-table,queue-summary,agent2-timeline,risk-monitor,pnl-curve,stats-history}.jsx` + `frontend/js/tabs/{positions,queue,detail}-tab.jsx` + `frontend/styles/{theme,components}.css`
- 后端:`src/options_event_trader/api/routers/dashboard.py` (Plan 1 + 7 个 endpoint) + `src/options_event_trader/services/pnl_writer.py` + `src/options_event_trader/domain/schemas.py` (`OpportunityDraftRead` 4 spec 字段)
- 数据库:`alembic/versions/20260429_0015_pnl_snapshots.py`
- 测试:`tests/e2e/test_dashboard.py` (12 SC) + `tests/test_dashboard_router.py` (9 endpoint tests) + `tests/test_pnl_writer.py` + `tests/test_opportunity_draft_read_schema.py`
- CI:`.github/workflows/deploy-dev.yml`(本地 commit 未 push, 待 lexicon 窗口先 push 后 rebase)

Plan 1 commit chain:`6b97dee/3b3f6cd/2e1bd9b/7a9db71/7cc9d4e/d9474cd`(2026-04-29 早 ship)
Plan 2 commit chain:`27b9e47` → `33e3968`(共 16 commits,2026-04-29 晚 ship)

---

## 脱敏自查 ✓

- [x] 无 API key / token / secret(只引用 GitHub Secret 名)
- [x] 无 IBKR 账户号 / Oracle tenancy ID(memory 里有, 客户文档不出现)
- [x] 无客户真实姓名 / 邮箱 / 具体金额数字(只写"约 200 万 RMB 量级")
- [x] 无 VM IP(memory 有, 不写客户文档)
- [x] 无内部实盘策略参数具体数值(只写"4 档 spread_ratio / 风控 7 条规则")

Internal 讨论(项目 repo `discussions/`)可以出现上述内容, docs-hub 是 public 永远不出现。
