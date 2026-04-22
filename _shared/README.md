# `_shared/` — 跨项目共享的客户协作方法论

> 这个目录是 docs-hub 里所有项目共用的**客户状态汇报方法论**。
> 新项目启动时直接复用,不用从零再造。

## 文件说明

| 文件 | 用途 | 修改频率 |
|------|------|---------|
| `claude-status-prompt.md` | CC 的指令模板,告诉 CC 怎么扫描、怎么输出 | 低(完成一个大阶段后 review) |
| `claude-status-glossary.md` | 通用术语字典,跨项目都适用的翻译规则 | 中(发现新词时补) |

## 项目专属的内容放哪?

不放这里。每个项目在自己目录下放:

```
docs-hub/
├── _shared/                        ← 本目录(通用)
│   ├── claude-status-prompt.md
│   └── claude-status-glossary.md
├── options_ai_trader/              ← 某项目
│   ├── tracker/
│   │   ├── tracker-state.json      ← 项目状态镜像
│   │   └── glossary-extra.md       ← 项目专属字典
│   └── weekly-reports/
│       └── 2026-W17.md             ← 周报完整原文
└── <其他项目>/
    └── ...
```

## 新项目怎么开始使用

1. 在 docs-hub 下建项目目录:`<项目名>/tracker/` 和 `<项目名>/weekly-reports/`
2. 复制 `.template/tracker-state.json.example`(如果已建) 为 `<项目名>/tracker/tracker-state.json`,填入项目信息
3. 新建 `<项目名>/tracker/glossary-extra.md`,收录该项目特有的技术词和业务词
4. 在该项目代码仓库里对 CC 说:
   > 按 `../docs-hub/_shared/claude-status-prompt.md` 生成 `<项目名>` 本次状态同步

## 工作流总览

```
客户(腾讯文档/微信群)         项目方(你)               CC(代码端)
     │                         │                        │
     │ 提意见/看进度            │                        │
     │◄────────────────────────┼────────────────────────┤
     │                         │                        │
     │                         │ 每周触发 CC:           │
     │                         │ "按 prompt 生成同步"    │
     │                         │───────────────────────►│
     │                         │                        │
     │                         │                        │ 1. 读 prompt/字典/state
     │                         │                        │ 2. 扫描 git/代码/测试
     │                         │                        │ 3. 识别变化
     │                         │                        │ 4. 输出四段式
     │                         │                        │ 5. 更新 state.json
     │                         │                        │ 6. 新增周报原文 md
     │                         │◄───────────────────────┤
     │                         │                        │
     │                         │ Review 并调整:         │
     │                         │ - 客户版发微信群         │
     │                         │ - TSV 粘贴到腾讯文档     │
     │                         │ - commit docs-hub      │
     │ 微信群收到周报           │                        │
     │◄────────────────────────┤                        │
     │ 腾讯文档看详细           │                        │
     │                         │                        │
```

## 两个系统的分工

| 系统 | 角色 |
|------|------|
| 腾讯文档 + 微信群 | **活跃交互层**:客户实时看进度、提意见、讨论 |
| docs-hub (git) | **沉淀归档层**:周报原文、状态历史、跨项目总览 |

腾讯文档没有 API,所以 docs-hub ↔ 腾讯文档 之间是**人工异步同步**:
- docs-hub → 腾讯文档:CC 生成 TSV,你粘贴
- 腾讯文档 → docs-hub:客户改了内容后你告知 CC,CC 更新 state.json
