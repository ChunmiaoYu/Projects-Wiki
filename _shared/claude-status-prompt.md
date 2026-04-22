# 客户状态汇报生成指令 (通用模板)

> **给 Claude Code 的指令**
> 位置:`docs-hub/_shared/claude-status-prompt.md`
> 跨项目通用,所有项目共用同一份方法论。

## 🎯 使用方式

在对应项目的仓库(如 `options_ai_trader/`)里打开 CC,对它说:

> 按 `../docs-hub/_shared/claude-status-prompt.md` 生成 `<项目名>` 本次状态同步

CC 会:
1. 读取本文件(方法论)
2. 读取 `../docs-hub/_shared/claude-status-glossary.md`(通用字典)
3. 读取 `../docs-hub/<项目名>/tracker/glossary-extra.md`(项目专属字典)
4. 读取 `../docs-hub/<项目名>/tracker/tracker-state.json`(状态基准)
5. 扫描当前项目代码、git log、TODO、测试结果
6. 产出四段式输出,更新 JSON 镜像,归档周报

---

## 📂 跨仓库文件读写

CC 运行在项目仓库(如 `options_ai_trader/`)工作目录下,以相对路径访问 `docs-hub`。

**读取**:
- `../docs-hub/_shared/claude-status-prompt.md`(本文件)
- `../docs-hub/_shared/claude-status-glossary.md`(通用字典)
- `../docs-hub/<项目名>/tracker/glossary-extra.md`(项目专属字典)
- `../docs-hub/<项目名>/tracker/tracker-state.json`(状态基准)

**写入**:
- `../docs-hub/<项目名>/tracker/tracker-state.json`(更新后的 JSON)
- `../docs-hub/<项目名>/weekly-reports/<YYYY-WNN>.md`(本周周报完整原文)

**重要**:如果 `../docs-hub/` 路径不存在,停止并告知用户。不要自己猜路径。

---

## 🧠 核心工作流程

### Step 1:读取基准状态

读取项目对应的 `tracker-state.json`,获取:
- 当前所有任务/意见/决议/里程碑/交付物
- ID 计数器(`id_counters`)
- `last_sync` 时间戳

### Step 2:扫描项目实际状态

```bash
# 自上次同步以来的 commit
git log --since="<last_sync>" --pretty=format:"%h | %ad | %s%n%b" --date=iso

# 近 7 天 commit (后备)
git log --since="7 days ago" --pretty=format:"%h | %ad | %s" --date=short

# TODO/FIXME 扫描 (覆盖项目实际文件类型)
grep -rn -E "TODO|FIXME|XXX|HACK" \
  --include="*.py" --include="*.md" --include="*.yml" --include="*.yaml" \
  --include="*.html" --include="*.js" --include="*.json" \
  --exclude-dir=".venv" --exclude-dir="node_modules" --exclude-dir=".git" \
  --exclude-dir="testResults" --exclude-dir=".playwright-mcp" .

# 自上次同步以来新增的文件
git log --since="<last_sync>" --diff-filter=A --name-only --pretty=format: | sort -u | grep -v "^$"

# testResults 目录 (被 .gitignore,本地才有)
ls -la testResults/ 2>/dev/null && ls testResults/*.json 2>/dev/null | head -20

# 项目关键配置/文档变化 (对本项目尤其重要)
git log --since="<last_sync>" --name-only -- 'config/*.yml' 'docs/*.md' 'prompts/*.md' 2>/dev/null
```

### Step 3:识别变化 (diff)

对比 JSON 镜像和实际状态,分三类:

- **新增项**:JSON 里没有但代码/讨论里新出现的
- **更新项**:JSON 里有但状态应变化的
- **无变化项**:不生成输出,保持镜像

### Step 4:产出四段式输出 (见下方"输出格式")

### Step 5:更新 JSON 镜像 + 归档周报

- 更新 `tracker-state.json`(last_sync、id_counters、各对象状态)
- 在 `../docs-hub/<项目名>/weekly-reports/` 新增 `YYYY-WNN.md`(完整周报原文)

---

## ⏱ 时间估算规则

**不追求精确,只给客户量级感**。

| 工作量特征 | 估算 |
|-----------|------|
| 单个小 commit | ~半天 |
| 单个中等 commit (改一个文件) | ~1 天 |
| 多个相关 commit (改一个模块) | ~2-3 天 |
| 跨模块重构、大新功能 | ~半周/一整周 |
| 涉及调试、测试反复 | 基础估算 ×1.5 |

**表述**:客户版用"约 2 天"、"半周左右"、"本周主要精力";内部版具体些。

**时间窗口**:本周 = 最近 7 天(滚动)。

---

## 🗣 业务语言翻译

所有面向客户的内容必须走字典翻译:
- 通用字典:`../docs-hub/_shared/claude-status-glossary.md`
- 项目专属字典:`../docs-hub/<项目名>/tracker/glossary-extra.md`

**两个字典都要加载**,项目专属优先。

**绝对不允许出现在客户版的词**:
- 变量名/类名/函数名/commit hash/文件路径
- schema/parser/compiler/blocker/fixture/endpoint
- PascalCase/camelCase 标识符(如 ParsedIntentLLMOutput)
- 技术栈名 (LangGraph/FastAPI/React/pytest/等)
- 代码字段名 (intake_v2/submit_blockers/等)

字典未命中时:**宁可模糊,也不暴露技术词**。在内部版末尾列出新术语,提醒项目方补字典。

---

## 📄 输出格式

### 第一段:客户版微信播报 (Markdown)

```markdown
# 项目进展同步 · <YYYY-MM-DD>

## 本周重点进展
(3-5 条,业务语言,每条含"约花费时间")
- 完善了 XXX,现在可以 YYY (约 X 天)
- ...

## 当前整体进度
- 当前阶段:<阶段名>
- 阶段进展:约 X%
- 距离下一里程碑:预计 X 周

## 需要您关注 / 决策的事项
(无则写"暂无,按原计划推进")
1. **<事项>** - 背景... - 希望您:...

## 下周计划
- ...

## 时间总览
- 本周实际投入:约 X 天
- 本阶段已投入:约 X 周
- 距离最终交付:预计 X 周 (目标:YYYY-MM-DD)

## 完整周报原文
👉 https://<你的用户名>.github.io/docs-hub/<项目名>/weekly-reports/YYYY-WNN/
```

### 第二段:腾讯文档粘贴块 (TSV)

**规则**:真实 Tab 分隔、严格列数、空值保留空(连续两个 Tab)、日期用 YYYY-MM-DD。

按每个 Sheet 分组输出:

```
## 📋 项目进度 Sheet

### 新增行 (粘贴到最下方)
列顺序: 编号|所属阶段|任务名称|状态|进度%|负责人|预计开始|预计完成|实际完成|依赖任务#|备注
<tab 分隔的行>

### 更新指令 (需手动定位到对应行修改)
格式: 编号 | 字段 | 新值 | 原因
T003 | 状态 | 已完成 | 27 字段扩展完成
```

```
## 💬 意见清单 Sheet (13 列)

### 新增行
列顺序: 编号|提出时间|提出人|意见内容|类型|优先级|当前状态|处理人|处理结论|预计完成日期|关联任务#|更新时间|状态流转历史

### 更新指令
O003 | 当前状态 | 已采纳 | 客户确认方案
O003 | 状态流转历史 | <追加> 4/20 讨论中 → 4/22 已采纳 | (在原有历史末尾追加,不覆盖)
```

```
## 📝 决议记录 Sheet - 新增行
列顺序: 日期|议题|决议内容|客户确认|相关意见#|备注
```

```
## 📅 里程碑 Sheet - 更新指令
M3 | 状态 | 已完成 | Step 1.2 hardening 收口
```

```
## 📦 交付物清单 Sheet - 新增行/更新指令
```

```
## 🔄 变更日志 Sheet - 新增行 (核心追踪)

列顺序: 变更时间|对象类型|对象编号|变更字段|变更前|变更后|变更原因/说明

2026-04-22 10:00\t任务\tT003\t状态\t进行中\t已完成\t27 字段扩展完成
2026-04-22 10:00\t意见\tO003\t状态\t讨论中\t已采纳\t客户确认方案
```

**强制规则**:每一条"更新指令"必须对应一条"变更日志"条目。

```
## 📰 周报归档 Sheet - 新增行

列顺序: 周次|日期|三行摘要|关键里程碑/决策|完整原文链接

W17\t2026-04-22\t1) ... \n2) ...\n3) ...\t<里程碑/决策>\thttps://<你的用户名>.github.io/docs-hub/<项目名>/weekly-reports/2026-W17/
```

摘要严格 3 行以内。

### 第三段:更新后的 JSON 镜像

```markdown
## 请把以下内容写入 ../docs-hub/<项目名>/tracker/tracker-state.json,覆盖旧文件
```

然后给出完整新 JSON。

更新规则:
- `last_sync` 改为今日
- `last_sync_by` 改为本次操作说明
- `id_counters` 如有新增则递增
- 各 map 只改变化的 key,其他保持
- `weekly_reports` 数组末尾追加本周摘要对象:
  ```json
  {
    "week": "W17",
    "date": "2026-04-22",
    "summary": ["...", "...", "..."],
    "highlights": ["..."],
    "full_report_path": "options_ai_trader/weekly-reports/2026-W17.md"
  }
  ```

### 第四段:周报完整原文 + 内部说明

```markdown
## 请把以下内容写入 ../docs-hub/<项目名>/weekly-reports/YYYY-WNN.md

# 周报 · <项目名> · YYYY-WNN
> 日期: YYYY-MM-DD | 阶段: <阶段名>

## 客户版原文
<客户版 Markdown,完整保留>

## 内部版技术细节

### 本次扫描数据源摘要
- Git commit: X 条 (自 <last_sync> 以来)
- 新增文件: X 个
- TODO 变化: 新增 X / 解决 X
- 测试状态: X passed / Y failed

### 字典未命中的新术语 (建议加入 glossary-extra)
- <技术词> → <建议业务表述>

### 本次不确定的地方 (需项目方确认)
- <某任务归到哪个阶段>

### 手改提醒
⚠️ 项目方请确认:自上次同步以来,是否在腾讯文档里直接改过任何内容?
- 若有,请告知,以便同步到 JSON 镜像
- 若无,回"没有"

### 风险提示
- ...
```

**同时在 chat 里给出短版内部说明**,方便项目方快速看到。

---

## ⚠️ 底线规则

1. **不编造**:扫描不到的东西不凭印象写
2. **不污染客户版**:技术词一律翻译
3. **更新指令必配变更日志**:缺一不可
4. **JSON 和周报原文必须同步更新**
5. **保守估时**:给预计时间时往后放缓冲
6. **阻塞项必须说"希望客户做 X"**,不能只说"卡住了"
7. **TSV 严格格式**:真实 Tab、严格列数
8. **每步汇报,等用户确认再继续**:不要一次做完

---

## 📌 特殊场景

**本周无重大进展**: 直接说"本周主要精力在 XXX 稳定性打磨,没有大新功能"

**本周遇阻**: 客户版"需要您关注"置顶,加粗 `**⚠️ ...**`

**本周完成里程碑**: 客户版顶部加 `🎉 本周完成里程碑:M<N> · ...`

**JSON 与实际严重不符**: 停下来,在内部版标出差异,问项目方以 JSON 还是以代码为准

---

## 🔁 触发时的简短模式

如果项目方说"按 prompt 生成状态同步",默认执行完整 5 步流程。

如果项目方说"快速模式"或"只更新某几条",CC 可以:
- 跳过全量扫描,只处理指定项
- 但 JSON 和变更日志仍须更新
- 不生成周报原文(因为不是完整周报)

---

*本文件由项目方维护。每完成一个大阶段后 review 一次。*
