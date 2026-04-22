# 通用技术 → 业务术语翻译字典

> 位置:`docs-hub/_shared/claude-status-glossary.md`
> 跨项目通用部分。项目专属术语放各项目的 `tracker/glossary-extra.md`。
> CC 生成客户版汇报时,**两个字典都要加载**,项目专属优先。

---

## 📐 通用技术概念

| 技术词 | 客户可见表述 |
|--------|-------------|
| API / endpoint | 系统接口 |
| parser / compiler | 解析器 / 编译处理 |
| runtime | 运行环境 |
| pipeline | 处理流程 |
| test / unit test / fixture | 自检 / 功能验证 |
| deploy / deployment | 上线部署 |
| migration | 数据迁移 / 结构升级 |
| refactor | 代码结构优化 |
| hardening | 稳定性打磨 / 细节收口 |
| rollback | 版本回退 |
| backup | 备份 |
| logging | 运行日志 |
| monitoring | 系统监控 |
| alert / alarm | 告警 |

---

## 🎯 常见技术动作 → 业务说法

| 技术动作 | 业务表述 |
|---------|---------|
| 扩展字段 / 新增字段 | 完善数据格式,让系统可以理解更多信息 |
| 修复 bug | 修复 XXX 问题 / 解决 XXX 异常 |
| 重构 | 为后续扩展做准备 |
| 加测试 | 增加自检覆盖,降低出错风险 |
| 补注释 / 改文档 | 完善开发文档 |
| 调参数 | 调优系统参数 |
| 性能优化 | 提升响应速度 / 降低延迟 |
| 加日志 | 增强系统可观测性 / 便于排错 |
| 规则化 / 抽象化 | 把 XXX 升级成通用能力,避免一次一补 |
| 联调 | 与 XXX 对接测试 |
| 打包 / 构建 | 生成部署文件 |

---

## 🚦 通用状态/进度词

| 技术状态 | 客户可见表述 |
|---------|-------------|
| MVP / 最小可用版本 | 基础版 / 雏形 |
| POC / Proof of Concept | 原型验证 |
| alpha / beta | 内测版 / 试用版 |
| RC / release candidate | 预发布版本 |
| GA / 正式版 | 正式上线 |
| in progress | 进行中 |
| blocked | 暂时卡住(后接原因) |
| pending review | 等待复核 / 等待确认 |
| deprecated | 已弃用(对客户少用,换成"改用新方案") |
| breaking change | 重要变更(可能影响现有使用方式) |

---

## ☁️ 云/部署通用词

| 技术词 | 客户可见表述 |
|--------|-------------|
| VM / 虚拟机 | 云服务器 |
| cloud / 云平台 | 云平台 |
| CI/CD | 自动化测试与部署流程 |
| GitHub Actions | 自动化流程 |
| systemd | 系统服务管理 |
| nginx | 网关/反向代理(一般不暴露,说"访问入口") |
| PostgreSQL / MySQL / SQLite | 数据库 |
| Redis | 缓存 |
| Docker | 容器(一般不暴露) |

---

## 🗣 通用技术栈词

| 技术词 | 客户可见表述 |
|--------|-------------|
| Python / Java / Node.js | 后端开发语言(一般不提) |
| React / Vue / Angular | 前端框架(一般不提,说"操作界面") |
| FastAPI / Flask / Django | 后台系统 |
| SQLAlchemy / ORM | 数据访问层 |
| pytest / unittest | 自动化测试 |
| pydantic | 数据校验 |

---

## ❗ 永远不要出现在客户版的词

**一旦出现,视为翻译失败,必须改写**:

```
commit, branch, merge, PR, hash, diff, checkout, revert, rebase
TODO, FIXME, XXX, HACK
schema, parser, compiler, blocker, fixture, endpoint, middleware
JSON, YAML, XML, SQL (除非客户是技术人员)
PascalCase / camelCase 标识符 (如 UserIntent, ParsedResult)
文件路径 (如 src/services/parser.py)
版本号式代号 (如 P1/P2/P3,除非客户已熟悉)
技术栈名 (如 LangGraph, FastAPI, React, pytest, uvicorn)
代码字段名 (如 user_id, created_at, max_risk_pct)
git 命令 (如 git pull, git push)
```

---

## 🎨 推荐句式

### 描述进展
- "完善了 XXX,现在可以 YYY"
- "加强了 XXX,使 YYY 更可靠"
- "新增了 XXX 能力,支持 YYY 场景"
- "优化了 XXX 的处理逻辑,减少 YYY 的发生"

### 描述阻塞
- "XXX 卡在 YYY,希望您帮忙确认 ZZZ"
- "目前发现 XXX 存在歧义,希望您补充说明:..."
- "XXX 功能需要您先明确 YYY 的具体要求,以便我们对应实现"

### 描述延期 (需要时)
- "原计划本周完成 XXX,因遇到 YYY,预计延至下周 ZZZ 完成"
- "XXX 比预期复杂,增加了约 X 天工作量,整体里程碑影响不大"

### 描述完成里程碑
- "🎉 本周完成里程碑:M<N> · <里程碑名>"
- "至此,<业务能力> 已可演示/可试用"

---

## 📝 字典维护说明

### 何时更新

- CC 在汇报时发现新术语(内部版会提示)
- 项目引入新技术栈
- 业务表述被客户反馈"听不懂"

### 怎么更新

1. 在对应分类下加一行
2. 左列:代码/commit/文档里的原词;右列:给客户看的表述
3. 原则:**短、准、不装**

### 特殊约定

- 客户已熟悉的词可直接使用 (如"IBKR"、"机会单")
- 技术词如客户已学会 (如"期权 Level 权限"),可直接用
- 判断基准:**这个词如果客户听不懂,会不会引发他的焦虑或质疑?**
  - 会 → 必须翻译
  - 不会 → 可以保留

---

## 🔗 项目专属字典

每个项目还有自己的 `glossary-extra.md`,含:
- 项目特有的架构层级(如六层架构)
- 项目特有的字段名(如 intake_v2)
- 项目特有的阶段/版本标记(如 Step 1.2, Phase 10+)
- 项目特有的业务领域词(如期权、机会单)

CC 加载时:
- 先读本通用字典
- 再读项目 `glossary-extra.md`(项目定义覆盖通用定义)
