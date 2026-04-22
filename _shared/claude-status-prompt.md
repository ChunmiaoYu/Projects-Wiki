# 客户 Dashboard 生成/更新指令 (v2.0 · 单文档模式)

> **给 Claude Code 的指令**
> 位置: `docs-hub/_shared/claude-status-prompt.md`
> 跨项目通用,所有项目共用同一份方法论。

## 🎯 使用方式

用户在对应项目的仓库(如 `options_ai_trader/`)里对 CC 说:

> **"更新 Dashboard"** 或 **"更新 dash"**

或更完整:

> 按 `../docs-hub/_shared/claude-status-prompt.md` 更新 `<项目名>` Dashboard

CC 会:

1. 读 `../docs-hub/<项目名>/tracker/tracker-state.json` (v2.0 schema) 作为基准
2. 读 `../docs-hub/_shared/claude-status-glossary.md` (通用字典) + `../docs-hub/<项目名>/tracker/glossary-extra.md` (项目字典)
3. 扫项目代码 / git log / task_plan / findings 的变化
4. 主动问用户 2 个问题 (评论镜像 + 高亮)
5. 写新版 `docs/<项目路径>/dashboard/index.md`
6. 旧版归档到 `docs/<项目路径>/dashboard-snapshots/YYYY-MM-DD.md`
7. 更新归档索引 `docs/<项目路径>/dashboard-snapshots/index.md`
8. 更新 `tracker-state.json`
9. commit + push 到 docs-hub
10. 展示新 Markdown 给用户复制到腾讯文档替换

---

## 📏 核心原则

### 客户视角设计

Dashboard 是给 **客户 + 客户团队** 在微信群里看的,不是给项目经理看的:

- **3 分钟读完** — 不是 10 分钟
- **视觉直观** — 进度条 / 时间表 / emoji, 不是段落堆砌
- **零技术术语** — 客户懂业务, 不懂 IT 和项目开发
- **去 "您"** — 群里多人, 用中性表述 ("本周待处理" 不是 "您需要处理")
- **最近 1.5 周聚焦** — 更早的不汇总, 都在历史归档里
- **固定结构** — 每次更新填同一个模板, 客户有熟悉感

### 承载分层

| 层 | 载体 | 用户 |
|---|------|------|
| **客户主入口** | 腾讯文档在线文档 (支持评论) | 客户 + 客户团队 |
| **历史归档 + 备份** | GitHub Pages (MkDocs 渲染) | 项目方参考 / 客户深挖时用 |
| **状态镜像** | `tracker-state.json` v2.0 | CC 读写, 客户/项目方都不直接看 |

旧的 "周报 TSV + 6 Sheet 表格" 模式 **已废弃**。

### 客户已知术语

可直接使用不翻译 (客户业务背景熟):
- IBKR / 机会单 / 期权 / Call / Put / Bullish / Bearish
- 行权价 / 到期日 / Delta / 保证金 / 实盘 / 模拟 / 持仓 / PnL
- 期权 Level 权限 (Level 1/2/3/4)

### 客户绝对看不懂的词 (严禁出现)

**项目内部术语**:
- Phase / Step / Sprint / Milestone / Release / hardening / refactor
- Intake / parser / compiler / Agent1 / Agent2 / LangGraph / Strategy Agent
- DRAFT/SUBMIT / submit_blockers / support_level / effective_mode
- F1/F2/F3 / M1-M13 / Sprint-0A / Sprint-0B
- Bundle / HOLD / PARTIAL_CLOSE (英文 enum 值)
- 五专家评审 / 提示词缓存 / 分级调用

**技术栈词**:
- FastAPI / React / PostgreSQL / Alembic / pytest / Playwright
- commit / PR / branch / deploy / hook
- schema / endpoint / middleware / fixture
- Claude Sonnet/Haiku (说 "AI" 就够, 不必说具体模型名)

**项目管理词**:
- blocker / ticket / story point / backlog / kanban
- 任务编号 (T001/M12/D014 这些不给客户看)
- 意见编号 (O001/O002)
- 负责人 (不说 "owner")

### 翻译示例

| 技术原文 | 客户版 |
|---------|--------|
| "Phase 11 Agentic Trading Redesign" | "AI 自动盯盘方向升级" |
| "五专家评审通过 spec v5" | "经过全方位专业讨论, 方案已确定" |
| "execution-status refactor Release N" | "系统反馈逻辑更清晰了" |
| "F2-P2 + TP-SL-UI + COMPLETENESS 三合一重构" | "几个老问题一次修完,指令识别更稳" |
| "Intake 完整性" | "中文指令理解" |
| "mid-based 4 档定价" | "智能挂单" |
| "Step A 反馈冻结" | (不提, 这是内部实施步骤, 客户不关心) |
| "478 pytest 全绿" | (不提, 客户不关心自检数字, 只关心 "实盘能不能跑") |

---

## 📐 固定结构 (7 个区块)

每次更新 Dashboard 必须包含这 7 块, 顺序不变:

### 1. 顶部 · 元信息
```markdown
# 📊 <项目名客户友好版> · 项目看板

> **最近更新**: YYYY-MM-DD · [查看历史版本](../dashboard-snapshots/)
```

### 2. 📍 当前状态
- 整体进度条 (字符 `█` 和 `░`, 10 格) + 百分比数字
- 下一节点 (日期 + 一句话描述)
- 1-2 句话总体状态

### 3. ⚠️ 本周待处理
- 最多 3 条, 每条 1-2 行
- 如果是客户要做的事, 明确说 "已发起 / 进行中 / 待跟进"
- 如果无, 写 "无"

### 4. 📆 最近 1.5 周进展
- 3-5 条, 每条用简短 heading (带 ⭐ 给最重要的 1 条)
- 每条正文 2-5 行, 业务语言
- 用 **以前 / 现在 / 好处** 的对比结构 (最有感)
- 避免技术细节, 避免 pytest 数字

### 5. 🗺 未来计划
- Markdown 表格, 3-5 行
- 时间 + 动作, 不出现任何技术术语
- 关键节点用粗体强调

### 6. 💬 反馈与留言
- 固定文案, 引导客户评论:
  > 有任何想法、疑问、或想看的方向: **选中对应文字或段落 → 右键"评论"**。项目方会实时收到通知并回复。也欢迎在微信群里直接讨论。

### 7. 📂 查看历史版本
- 固定指向 `../dashboard-snapshots/`
- 文案: "每次看板更新会自动归档一份快照,按日期可查"

---

## 🔄 更新流程 (CC 执行)

### Step 0: 读基准

```
读入:
  ../docs-hub/<项目名>/tracker/tracker-state.json   (v2.0 schema)
  ../docs-hub/_shared/claude-status-glossary.md    (通用字典)
  ../docs-hub/<项目名>/tracker/glossary-extra.md    (项目字典)
  ../docs-hub/docs/<项目路径>/dashboard/index.md    (当前 Dashboard)
```

### Step 1: 扫项目变化

```bash
# 自 last_sync 起的 commit
git log --since="<last_sync>" --pretty=format:"%ad %h %s" --date=short

# task_plan 五问
head -50 task_plan.md

# findings 未修项
cat findings.md | grep -E "未解决|未修|P[012]"

# 新增的 spec / decisions 文档
git log --since="<last_sync>" --name-only -- 'docs/superpowers/'
```

### Step 2: 问用户 2 个问题

```
问题 1: 客户在腾讯文档有新评论吗?
  - 用户回 "有, 客户说 XXX" → CC 把评论记到 client_comments_mirror
  - 用户回 "没有" / "没看" → 跳过
  - 有评论时,在 Dashboard 底部"反馈与留言"区前补一个"📌 上期客户留言回应"段

问题 2: 有哪些事希望特别高亮让客户看的?
  - 用户给 1-3 条 → 作为本期 ⭐ 高亮候选
  - 用户说 "按 git log 判断" → CC 自己选最重要的
```

### Step 3: 生成新 Dashboard

- **固定 7 区块结构**, 不随意改
- **所有文案走字典翻译**, 命中禁词必替换
- **语气去 "您"**, 用中性表述
- 进度条 + 日期 + 表格视觉对齐
- 长度控制在 **3 分钟内读完**

### Step 4: 归档旧版

```bash
# 复制当前 dashboard/index.md 到 snapshots/YYYY-MM-DD.md
cp docs/<项目路径>/dashboard/index.md docs/<项目路径>/dashboard-snapshots/<YYYY-MM-DD>.md

# 编辑 snapshot 顶部加归档说明
# 更新 dashboard-snapshots/index.md 索引表新加一行
```

### Step 5: 写新版 + 更新 JSON

- 覆盖 `docs/<项目路径>/dashboard/index.md` 为新版
- 更新 `tracker-state.json`:
  - `_meta.last_sync` = 今天
  - `_meta.last_sync_by` = "by CC - <本次重点一句>"
  - `next_milestone` / `upcoming_timeline` / `this_week_highlights` / `action_items` 全量替换
  - `snapshots` 数组末尾追加新快照条目

### Step 6: commit + push

```bash
cd ../docs-hub
git add docs/<项目路径>/dashboard/ docs/<项目路径>/dashboard-snapshots/ <项目名>/tracker/tracker-state.json
git commit -m "docs(<项目路径>): Dashboard 更新 YYYY-MM-DD — <一句摘要>"
git push origin main
```

### Step 7: 给用户复制块

展示 **完整新版 Dashboard Markdown** 在 chat 里 (代码块形式), 让用户:
1. 打开腾讯文档 Dashboard
2. 全选 Ctrl+A → 删除 → 粘贴新内容 → 保存
3. 耗时约 15 秒

同时告知用户:
- GitHub Pages 渲染版: `https://chunmiaoyu.github.io/Projects-Wiki/<项目路径>/dashboard/` (预览用)
- 历史归档: `https://chunmiaoyu.github.io/Projects-Wiki/<项目路径>/dashboard-snapshots/`

---

## ⚠️ 硬性约束

1. **禁词清单上面那张表必须全部替换**, 命中即改, 不讨价还价
2. **一切 emoji 都可以用** (腾讯文档 + MkDocs 都渲染 OK)
3. **不用 Material 专属语法** (`!!! note` / `??? tip`) — 腾讯文档不识别, 会直接显示字符
4. **不超过 50 行纯内容** (不含标题和分隔符), 客户 3 分钟必须看完
5. **时间表格最多 5 行**
6. **本周进展最多 5 条**, 每条 heading 10 字以内
7. **每次更新必须归档旧版** — 不归档就是违规
8. **commit message 用中文** (见 Step 6 模板)

---

## 📋 tracker-state.json v2.0 schema

```json
{
  "_meta": {
    "version": "2.0",
    "last_sync": "YYYY-MM-DD",
    "last_sync_by": "by CC - <重点>"
  },
  "project_info": {
    "name": "<英文名>",
    "name_zh": "<客户友好中文名>",
    "client_name": "<客户名>",
    "start_date": "YYYY-MM-DD",
    "target_delivery": "<客户语言描述>",
    "current_stage": "<客户语言描述>",
    "progress_pct": 0
  },
  "next_milestone": {
    "name": "<客户语言>",
    "date": "YYYY-MM-DD",
    "environment": "云 DEV / UAT / PROD / 模拟账户 / 实盘",
    "note": "<一句补充>"
  },
  "upcoming_timeline": [
    {"date": "...", "label": "...", "event": "..."}
  ],
  "this_week_highlights": [
    {
      "title": "...",
      "detail_before": "...",
      "detail_after": "...",
      "benefit": "...",
      "is_star": true
    }
  ],
  "action_items": [
    {
      "id": "A001",
      "content": "<客户语言>",
      "status": "进行中/已完成/待启动",
      "owner": "客户/项目方",
      "created_at": "YYYY-MM-DD"
    }
  ],
  "client_comments_mirror": [
    {
      "date": "YYYY-MM-DD",
      "section": "<评论位置>",
      "content": "<客户原话>",
      "response": "<项目方回应>"
    }
  ],
  "snapshots": [
    {
      "date": "YYYY-MM-DD",
      "path": "docs/<项目路径>/dashboard-snapshots/YYYY-MM-DD.md",
      "summary": "<一句摘要>"
    }
  ]
}
```

---

## 🎯 判断何时主动高亮 ⭐

一期只给 **1 条** ⭐ 高亮, 判断标准:

- **客户感知强**: 能直接影响客户怎么用这个系统
- **本期关键变化**: 这期做的事 3 条里最重要的那个
- **方向性改动**: 比"修小问题"更重要的事

错误示范:
- ⭐ "478 pytest 全绿"  → 客户不关心数字
- ⭐ "重构模块 XYZ"  → 客户不知道 XYZ 是啥

正确示范:
- ⭐ "AI 下单后自己盯盘" → 客户能立刻理解这个好处
- ⭐ "新增均线突破自动触发" → 客户熟的业务概念
