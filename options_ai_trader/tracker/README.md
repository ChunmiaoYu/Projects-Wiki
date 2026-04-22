# options_ai_trader / tracker

> 本目录是 `options_ai_trader` 项目的客户协作追踪中心。

## 目录内容

| 文件 | 用途 |
|------|------|
| `tracker-state.json` | 状态镜像,CC 的"记忆"。**不要手工编辑** |
| `glossary-extra.md` | 本项目专属术语字典(补充通用字典) |
| `腾讯文档链接.md` | 记录协作台 URL(首次填入后一般不变) |
| `README.md` | 本文件 |

## 同级目录 `weekly-reports/`

CC 每次生成状态同步时,把**当周周报的完整原文**(客户版 + 内部版)写入:
```
docs-hub/options_ai_trader/weekly-reports/YYYY-WNN.md
```

## 使用方式

在 `options_ai_trader` 代码仓库里打开 CC,对它说:

> 按 `../docs-hub/_shared/claude-status-prompt.md` 生成 options_ai_trader 本次状态同步

CC 会:
1. 读 `../docs-hub/_shared/claude-status-prompt.md`(方法论)
2. 读 `../docs-hub/_shared/claude-status-glossary.md`(通用字典)
3. 读 `../docs-hub/options_ai_trader/tracker/glossary-extra.md`(本项目字典)
4. 读 `../docs-hub/options_ai_trader/tracker/tracker-state.json`(状态基准)
5. 扫描当前项目代码、git log、TODO、testResults
6. 产出四段式输出,更新 JSON 镜像,新增周报原文

## 前提

- `options_ai_trader/` 和 `docs-hub/` 两个仓库要在本地同一层目录下(比如都在 `C:\Users\yumia\Downloads\`)
- 否则 CC 用相对路径 `../docs-hub/` 访问不到

## 你的日常动作

1. **客户在腾讯文档或微信群的变动** → 你人肉告诉 CC(因为腾讯文档无 API)
2. **CC 生成的输出** → 你粘贴到腾讯文档对应 Sheet
3. **CC 生成的 JSON 和周报原文** → commit + push 到 docs-hub

## JSON 镜像的纪律

`tracker-state.json` 是 CC 的唯一可信基准。维护它的几条规则:

1. **不要手工编辑** JSON。有变化告诉 CC,让 CC 更新
2. **腾讯文档里的变化** → 下次触发 CC 时明确告知
3. **JSON 跟着 git 走**:每次 commit 都留下历史,自带追踪能力
4. **JSON 和腾讯文档不一致时**:CC 会在内部版报告里警告,不要忽略

## 首次初始化

`tracker-state.json` 目前是空架。首次使用前必须由 CC 用真实项目数据填充:
- 扫描 git log 和 CHANGELOG,识别已完成的任务
- 识别真实里程碑(Step 1 / 1.1 / 1.2 / Step 2 / ...)
- 识别真实客户意见(从 task_plan / project_handoff 等提炼)
- 识别真实交付物

见《首次启动 CC 指令》(项目方会单独提供)。

## 腾讯文档链接模板

`腾讯文档链接.md` 建议内容:

```markdown
# options_ai_trader 客户协作台

- **腾讯文档链接**: https://docs.qq.com/sheet/xxxxxxxx
- **创建时间**: YYYY-MM-DD
- **客户账号**: (如有专用的,记录用户名,不记密码)
- **项目方账号**: yumia
- **权限设置**: 获得链接的人可编辑
- **微信群公告链接**: (已置顶到微信群)

## 备注
- ...
```
