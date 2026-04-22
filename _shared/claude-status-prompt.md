# 客户 Dashboard 生成/更新指令 (v2.1 · 双轨 + 评论镜像 + 自动剪贴板)

> **给 Claude Code 的指令**
> 位置: `docs-hub/_shared/claude-status-prompt.md`
> 跨项目通用,所有项目共用同一份方法论。

## 🎯 触发指令

用户在项目仓库里对 CC 说:

| 指令 | CC 行为 |
|---|---|
| **"更新 Dashboard"** / **"更新 dash"** | 完整流程: 扫变化 → 问 2 题 → 双版生成 → 归档 → 自动复制 |
| **"初始化 \<项目名\> Dashboard"** | 新项目首版生成 (CC 会先问背景信息) |

## 📐 v2.1 双轨架构

| 层 | 文件 | 用途 |
|---|------|------|
| **GitHub Pages 美化版** | `docs/<项目>/dashboard/index.md` | 客户主入口, Material 美化 |
| **腾讯文档摘要版** | `<项目>/tracker/tencent-doc-version.md` | 评论入口, 纯 markdown |
| **快照归档** | `docs/<项目>/dashboard-snapshots/YYYY-MM-DD.md` | 历史 + 评论镜像 |
| **状态 JSON** | `<项目>/tracker/tracker-state.json` | CC 读写, 客户/项目方都不直接看 |

旧版 v1.0 (周报 + 6 Sheet TSV) **已废弃**。

---

## 📏 客户视角设计原则

### 客户读 3 分钟看完
长度上限: GitHub Pages 美化版 ≤ 200 行总, 腾讯文档版 ≤ 100 行

### 群聊语气 · 去 "您"
群里多人, 用中性表述: "本周待处理" 不是 "您需要处理"; "项目方收到通知" 不是 "我会通知您"

### 客户已知术语 (不翻译)
- IBKR / 机会单 / 期权 / Call / Put / Bullish / Bearish
- 行权价 / 到期日 / Delta / 保证金 / 实盘 / 模拟 / 持仓 / PnL
- 期权 Level 权限 (Level 1/2/3/4)

### 客户禁词清单 (严禁出现)

**项目内部术语**:
- Phase / Step / Sprint / Milestone / Release / hardening / refactor
- Intake / parser / compiler / Agent1 / Agent2 / LangGraph / Strategy Agent
- DRAFT / SUBMIT / submit_blockers / support_level / effective_mode
- F1/F2/F3 / M1-M13 / Sprint-0A / Sprint-0B
- HOLD / PARTIAL_CLOSE (英文 enum 值)
- 五专家评审 / 提示词缓存 / 分级调用

**技术栈词**:
- FastAPI / React / PostgreSQL / Alembic / pytest / Playwright
- commit / PR / branch / deploy / hook
- schema / endpoint / middleware / fixture
- Claude Sonnet / Haiku (说 "AI" 就够)

**项目管理词**:
- blocker / ticket / story point / backlog / kanban
- 任务编号 (T001 / M12 / D014)
- 意见编号 (O001 / O002)
- "owner" / "负责人"

### 翻译速查

| 技术原文 | 客户版 |
|---------|--------|
| "Phase 11 Agentic Trading Redesign" | "AI 自动盯盘方向升级" |
| "五专家评审通过 spec v5" | "经过专业讨论, 方案已确定" |
| "execution-status refactor Release N" | "系统反馈逻辑更清晰" |
| "intake 完整性 + TP/SL UI 三合一" | "复杂指令识别更稳" |
| "mid-based 4 档定价" | "智能挂单" |
| "478 pytest 全绿" | (不提, 客户不关心数字) |

---

## 📐 Dashboard 7 区块固定结构

每次更新必须包含, 顺序不变:

1. **顶部 · 元信息** — 标题 + 最近更新日期 + 历史版本链接
2. **📍 当前状态** — 进度条 + 下一节点卡片 + 1-2 句总体状态
3. **⚠️ 本周待处理** — admonition warning, 最多 3 条
4. **📆 最近 1.5 周进展** — grid cards (3-5 张, ⭐ 1 张高亮)
5. **🗺 未来计划** — 自制 HTML timeline (3-5 节点)
6. **💬 反馈与留言** — 跳腾讯文档大按钮
7. **📂 历史版本** — 跳 snapshots/ admonition

---

## 🔄 完整流程 (CC 执行)

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
git log --since="<last_sync>" --pretty=format:"%ad %h %s" --date=short
head -50 task_plan.md
cat findings.md | grep -E "未解决|未修|P[012]"
git log --since="<last_sync>" --name-only -- 'docs/superpowers/'
```

### Step 2: 主动问用户 2 题

```
问题 1: 客户在腾讯文档有新评论吗?
  - 用户回 "有, 客户说 XXX" → 写入 client_comments_mirror + snapshot 顶部
  - 用户回 "没有" / "没看" → 跳过
  - 用户给截图 → 描述截图内容写入

问题 2: 有哪些事希望特别高亮 (⭐) 让客户看?
  - 用户给 1-3 条 → 选最重要 1 条作为本期 ⭐
  - 用户说 "按 git log 判断" → CC 自己选
```

### Step 3: 生成 GitHub Pages 美化版

写到 `docs/<项目路径>/dashboard/index.md`, **必须用 Material 美化模板**:
- next-milestone HTML 卡片 (左色条 + 图标 + 日期 + 事件)
- HTML 进度条 (渐变 indigo → cyan)
- admonition warning 框
- grid cards (Material 语法)
- 自制 HTML timeline (4 节点, 渐变竖线)
- comment-cta 大按钮 (跳腾讯文档绝对 URL)
- 徽章用 `<span class="badge badge-xxx">xxx</span>` (绝不用 `[xxx]{.class}`)

### Step 4: 生成腾讯文档摘要版

写到 `<项目>/tracker/tencent-doc-version.md`, **纯 markdown,无 HTML/CSS**:
- 链接用 `[文本](URL)` 标准 markdown
- 无徽章 (改用 emoji + 文字: 🟢 云 DEV Paper, 🟠 UAT Live, 🔴 PROD Live)
- 无 grid cards (改用 ### 标题分隔, 顺序排列)
- 无进度条 HTML (用文字 "整体进度约 X%")
- 内容**完整覆盖** Dashboard 业务陈述 (让客户能选中具体段落评论)

### Step 5: 自动复制腾讯文档版到剪贴板

```bash
powershell.exe -Command "Get-Content '../docs-hub/<项目>/tracker/tencent-doc-version.md' -Raw -Encoding UTF8 | Set-Clipboard"
```

执行完告诉用户: "✓ 腾讯文档版已复制到剪贴板, 去 Ctrl+A → Ctrl+V 粘贴"

### Step 6: 归档旧版 + 评论镜像

```bash
# 归档旧 dashboard 到 snapshot
cp docs/<项目路径>/dashboard/index.md docs/<项目路径>/dashboard-snapshots/<YYYY-MM-DD>.md
```

编辑 snapshot, **顶部加 2 段**:
```markdown
# 📂 看板快照 · YYYY-MM-DD
> 这是项目看板在 YYYY-MM-DD 的状态归档。最新版请看 [客户看板](../dashboard/)。

## 📌 上期客户留言镜像

> **客户**: <选段位置>: "<原话>"
> **项目方回应**: <你的回复>

(如果本期无新评论, 写: "本期无新留言")

---

(以下是本期看板内容)
```

更新归档索引:
```bash
# dashboard-snapshots/index.md 表格末尾追加一行
| YYYY-MM-DD | <一句摘要> + (评论 N 条) | [查看](YYYY-MM-DD.md) |
```

### Step 7: 更新 tracker-state.json

```python
# 更新字段:
_meta.last_sync = "YYYY-MM-DD"
_meta.last_sync_by = "by CC - <重点>"

# 全量替换:
project_info.progress_pct
next_milestone
upcoming_timeline
this_week_highlights
action_items

# 追加:
client_comments_mirror.append({
    "date": "YYYY-MM-DD",
    "section": "<评论位置>",
    "content": "<客户原话>",
    "response": "<项目方回应>"
})
snapshots.append({
    "date": "YYYY-MM-DD",
    "path": "docs/<项目路径>/dashboard-snapshots/YYYY-MM-DD.md",
    "summary": "<一句摘要>"
})
```

### Step 8: commit + push

```bash
cd ../docs-hub
git add -A
git commit -m "docs(<项目>): Dashboard 更新 YYYY-MM-DD — <一句摘要>"
git push origin main
```

### Step 9: 给用户最终输出

简短 chat 消息:
```
✅ Dashboard 已更新部署 (commit: <sha>)
✓ 腾讯文档版已复制到剪贴板, 去 Ctrl+A → Ctrl+V 粘贴

GitHub Pages 预览: https://chunmiaoyu.github.io/Projects-Wiki/<项目>/dashboard/
```

---

## 🎨 Material 美化模板速查

### 进度条 HTML

```html
<div class="progress-wrap" markdown>
<div class="progress-bar">
  <div class="progress-fill" style="width: 65%"></div>
  <span class="progress-label">整体进度 · 65%</span>
</div>
</div>
```

### 下一节点大卡片 HTML

```html
<div class="next-milestone" markdown>
<div class="icon">🚀</div>
<div class="content" markdown>
<div class="date">下一节点 · YYYY-MM-DD</div>
<div class="event">XXX 事件标题</div>
<div class="sub" markdown>环境: <span class="badge badge-dev">云 DEV</span> <span class="badge badge-paper">Paper</span></div>
</div>
</div>
```

### 自制 HTML 时间线 (4 节点)

```html
<div class="timeline" markdown>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-1">🎯</div>
<div class="tl-date">YYYY-MM-DD · 描述</div>
<div class="tl-title">节点标题</div>
<div class="tl-desc" markdown>描述 + <span class="badge badge-dev">环境标签</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-2">🔧</div>
<!-- ... -->
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-3">💰</div>
<!-- ... -->
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-4">🚀</div>
<!-- ... -->
</div>

</div>
```

### Grid Cards (本周进展)

```markdown
<div class="grid cards" markdown>

-   :material-star-circle: &nbsp; __标题 1__ · <span class="badge badge-star">本期重点</span>

    ---

    **以前**: 旧情况
    **现在**: 新情况
    *好处描述*

-   :material-speedometer: &nbsp; __标题 2__

    ---

    内容

</div>
```

### 评论入口大按钮 HTML

```html
<a class="comment-cta" href="<腾讯文档绝对 URL>" target="_blank">
<span class="label">:material-comment-text-outline: · 反馈与留言</span>
有任何想法、疑问、或想看的方向 → 点这里跳转腾讯文档评论
</a>
```

### 徽章 span 速查

| 用途 | HTML |
|---|---|
| 本期重点 ⭐ | `<span class="badge badge-star">本期重点</span>` |
| 能做 (绿) | `<span class="badge badge-done">能做</span>` |
| 暂时做不了 (灰) | `<span class="badge badge-wait">暂时做不了</span>` |
| 云 DEV (绿) | `<span class="badge badge-dev">云 DEV</span>` |
| Paper (灰) | `<span class="badge badge-paper">Paper</span>` |
| UAT Live (橙) | `<span class="badge badge-uat">UAT Live</span>` |
| PROD Live (红) | `<span class="badge badge-prod">PROD Live</span>` |
| 进行中 (蓝) | `<span class="badge badge-doing">进行中</span>` |

⚠️ **绝对禁用** `[xxx]{.badge .badge-yyy}` — python-markdown attr_list 不支持纯文本 inline class, 会被当字面字符串渲染。

---

## 📝 腾讯文档摘要版模板速查

```markdown
# <项目名> · 项目协作台

> **📊 完整项目看板（含进度条 / 时间线 / 卡片）**
> [https://chunmiaoyu.github.io/Projects-Wiki/<项目>/dashboard/](https://chunmiaoyu.github.io/Projects-Wiki/<项目>/dashboard/)
>
> **📂 历史版本归档**
> [https://chunmiaoyu.github.io/Projects-Wiki/<项目>/dashboard-snapshots/](https://chunmiaoyu.github.io/Projects-Wiki/<项目>/dashboard-snapshots/)

---

## 📍 当前状态

整体进度约 **N%**。<1-2 句总体状态>。

🚀 **下一节点**: YYYY-MM-DD · 节点标题

---

## ⚠️ 本周待处理

**待处理项 1 标题**

详细说明...

---

## 📆 最近 1.5 周进展

### ⭐ 重点项标题（本期重点）

**以前**: ...
**今后**: ...
**好处**: ...

### 第二项标题

内容...

### 第三项标题

内容...

---

## 🗺 未来计划

🎯 **YYYY-MM-DD**
节点标题
描述

🔧 **YYYY-MM-DD**
节点标题
描述

💰 **YYYY-MM-DD**
节点标题
描述

🚀 **YYYY-MM-DD**
节点标题
描述

---

## 💬 反馈与留言

有任何想法、疑问、或想看的方向 → **直接在这份文档里**:

1. **选中** 上面任意文字或段落
2. **右键** → "评论"
3. 输入反馈, 按回车提交

项目方会立即收到通知并回复。也欢迎在微信群里直接讨论。
```

---

## 📋 tracker-state.json v2.0 schema

```json
{
  "_meta": {
    "version": "2.0",
    "last_sync": "YYYY-MM-DD",
    "last_sync_by": "by CC - <本次重点>"
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
      "summary": "<一句摘要>",
      "comments_count": 0
    }
  ]
}
```

---

## ⚠️ 硬性约束

1. **必须双轨同步**: 每次更新 GitHub Pages 美化版 + 腾讯文档摘要版
2. **必须自动复制剪贴板**: powershell Set-Clipboard 后告知用户粘贴位置
3. **必须归档旧版**: 不归档 = 违规
4. **必须问 2 题**: 评论 + 高亮, 不能跳过
5. **禁词清单严格执行**: 命中即改, 不讨价还价
6. **徽章必须 `<span class="badge ...">`**: 不要 `[xxx]{.class}` (会被当字面字符串)
7. **腾讯文档版无任何 HTML/CSS**: 纯 markdown, 链接 `[文本](URL)`, 徽章用 emoji + 文字
8. **commit message 用中文**

---

## 🚀 新项目初始化流程 ("初始化 X Dashboard")

### 1. 收集背景信息 (问用户)

```
- 项目英文名 (用作目录名)?
- 项目中文名 (客户友好版,显示在标题)?
- 客户是谁 (1 个 vs 团队)?
- 项目核心目的 (1 句话)?
- 客户已知术语 (项目特有的)?
- 当前阶段?
- 第一个里程碑日期 / 内容?
```

### 2. 建目录骨架

```bash
mkdir -p ../docs-hub/<项目名>/tracker/
mkdir -p ../docs-hub/docs/<项目路径>/dashboard/
mkdir -p ../docs-hub/docs/<项目路径>/dashboard-snapshots/
```

### 3. 创建 4 个文件

- `<项目名>/tracker/tracker-state.json` (按 v2.0 schema)
- `<项目名>/tracker/glossary-extra.md` (项目特有术语)
- `<项目名>/tracker/tencent-doc-version.md` (腾讯文档摘要版)
- `docs/<项目路径>/dashboard/index.md` (GitHub Pages 美化版)
- `docs/<项目路径>/dashboard-snapshots/index.md` (归档索引,空表格)

### 4. 更新 mkdocs.yml nav

```yaml
- <项目中文名>:
  - 客户看板: <项目路径>/dashboard/index.md
  - 看板历史: <项目路径>/dashboard-snapshots/index.md
```

### 5. commit + push + 告知用户

- 给客户看板 URL
- 给腾讯文档**新建**指引 (用户自己建 Doc, 把内容粘贴, 配权限, 发链接给 CC 写到 tencent_doc_url)
- 给微信群置顶模板 (按本项目定制化)

---

## 💡 触发"评论镜像"的场景

每次"更新 Dashboard" 必走 Step 2 问 2 题。

特别地, 如果用户主动说 **"客户评论了 XXX"** / **"客户在腾讯文档说 YYY"**, 不必等下次正式 update, CC 应:
1. 立即把评论写入 `client_comments_mirror`
2. 询问: "需要我现在就更新 Dashboard 把客户留言镜像 push 上去吗? 还是等下次正式更新?"

用户确认后再走完整 Step 流程。
