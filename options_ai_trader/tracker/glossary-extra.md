# options_ai_trader 项目专属术语字典

> 位置:`docs-hub/options_ai_trader/tracker/glossary-extra.md`
> 这是本项目独有的技术词 → 业务表述映射。
> CC 加载时先读 `_shared/claude-status-glossary.md`,再读本文件。
> **本文件的定义覆盖通用字典中的同名条目**。

---

## 🏗️ 项目核心架构(六层)

| 技术词 | 客户可见表述 |
|--------|-------------|
| Opportunity / 机会单 / 交易机会 | 交易机会 (客户已熟悉,可直接用"机会单") |
| Trigger Rule / 触发规则 | 触发条件 |
| Strategy Run / 策略运行 | 策略生成过程 |
| Execution Run / 执行运行 | 下单执行过程 |
| Fact Layer / 事实层 | 交易记录层 |
| Task Queue / WorkflowTask | 后台任务队列 |
| MarketContext | 市场环境信息 |
| StrategyDecision | 策略结论 |
| AccountSnapshot | 账户快照 |
| BrokerOrder | 券商订单记录 |

---

## 🤖 Agent 与 Intake 相关

| 技术词 | 客户可见表述 |
|--------|-------------|
| Agent1 / Intake Compiler / IntakeService | 交易意图解析引擎 |
| Agent2 / Strategy Agent | 策略生成引擎 |
| IntakeState / IntakeGraph | 意图解析流程 |
| ParsedIntentDraft / ParsedIntentCore | 解析后的交易意图 |
| ParsedIntentLLMOutput | **绝不暴露**;统一说"解析结果" |
| intake_v2 / schema_version | 指令格式规范 / 格式版本 |
| LangGraph | **不暴露**;说"多阶段处理流程" |

---

## 📋 五个 Plan (intake_v2 核心概念)

| 技术词 | 客户可见表述 |
|--------|-------------|
| trigger_plan / TriggerPlanPreview | 触发条件方案 (什么情况算命中) |
| activation_plan / ActivationPlanPreview | 值班启动方案 (什么时候开始盯) |
| subscription_plan / SubscriptionPlanItem | 数据订阅方案 (要不要持续订阅行情) |
| compute_plan / ComputePlanItem | 指标计算方案 (要不要持续算指标) |
| query_plan / QueryPlanPreview | 临时查询方案 (关键阶段要补查什么) |

**原则**:如果客户已经从需求沟通里熟悉这些词,可直接用"触发方案/值班方案/订阅方案/指标方案/查询方案"这种半口语表述。

---

## 🔍 解析结果相关字段

| 技术词 | 客户可见表述 |
|--------|-------------|
| submit_blockers / submit_blockers_zh | 下单前自检项 / 拦截规则 |
| validation_errors / validation_errors_zh | 指令错误提示 |
| missing_fields | 缺失信息 |
| human_summary_zh | 人话版总结 (客户已熟悉概念,可直接说"总结") |
| support_level | 支持程度 (SUPPORTED→完全支持 / PARTIAL→部分支持 / UNSUPPORTED→暂不支持) |
| can_submit_as_is | 是否可直接提交 |
| can_submit_after_adjustment | 补充信息后可提交 |
| effective_mode / requested_mode | 执行模式 |

### 三种模式
| 技术词 | 客户可见表述 |
|--------|-------------|
| ADVICE_ONLY | 仅建议模式 (不下单,只给策略建议) |
| SEMI_AUTO | 半自动模式 (下单前需人工确认) |
| AUTO_EXECUTE | 全自动模式 (满足条件自动下单) |

---

## 🗂️ 配置与词典

| 技术词 | 客户可见表述 |
|--------|-------------|
| business_lexicon.yml / 商业词典 | 业务短语识别词典 |
| trigger_catalog.yml | 触发条件目录 |
| default_policies.yml | 默认策略配置 |
| lexicon_version / trigger_catalog_version | 词典版本号 / 目录版本号 |
| REQUIRE_CLARIFICATION | 需要用户澄清 |
| STRICT / PARTIAL 翻译 | 严格匹配 / 部分匹配 |
| matched_terms | 识别到的业务短语 |

---

## 🎯 项目阶段标记 (关键!)

**这些是项目内部命名,客户已经习惯听到这些,可以直接用**:

| 技术词 | 客户可见表述 |
|--------|-------------|
| Step 1 | 第一步:Intake parse 基础版 |
| Step 1.1 | 第一步收尾:错误处理收紧 |
| Step 1.2 | 第二步:intake_v2 升级 |
| Step 1.2 hardening | 第二步收口:稳定性打磨 |
| Step 2 | 第三步:两段式提交 + IBKR 集成 |
| Phase 1-8 | 阶段 1-8 (基建期,已完成) |
| Phase 9B | 阶段 9B:盘中实测(已完成) |
| Phase 10 / Phase 10+ | 阶段 10+ 反馈系统(已交付) |
| Phase 11 / Agentic Trading Redesign | **策略代理范式升级**(当前进行中) |
| Step A/B/C/D/E/F/G/H | Phase 11 实施步骤 A-H (基础设施/数据层/决策层/风控层/执行层/触发层/验证/交付) |
| Release N | 支持等级二值化重构发布代号 (2026-04-19) |
| 三合一重构 | 三个历史债合并一次修完(2026-04-20) |
| F1/F2/F3 反馈规则 | 三类反馈机制 (解析→策略→执行) —— 注意区别于"元规则 F1/F2" |
| 元规则 F1/F2 | 系统核心不变量的代码守护(对客户说"系统宪法守护") |
| 元规则 M1/M2/M8/M9 | 核心不变量条款 |
| B-plus 方案变体 | 部署方案 B-plus (保留 PostgreSQL + 多进程 host 直装) |

---

## 🔌 券商集成

| 技术词 | 客户可见表述 |
|--------|-------------|
| IBKR / Interactive Brokers | IBKR (客户已熟悉) |
| IB Gateway / TWS | 券商连接通道 |
| IBKRClient | 券商接入模块 |
| ib_async | 券商通讯库 |
| IBC | 券商登录自动化组件 |
| Paper account / Paper trading | 测试账户 / 模拟账户 (客户已熟悉) |
| Live account | 实盘账户 (客户已熟悉) |
| Option Level 1/2/3/4 | 期权权限等级 (客户已熟悉) |
| Margin account | 保证金账户 |
| Cash account | 现金账户 |
| qualify (contract) | 合约验证 |
| BAG construct | 组合单构建 |
| parent-child orders | 主子订单 |
| fill | 成交 |
| partial fill | 部分成交 |
| PnL | 盈亏 (客户已熟悉) |
| Risk Gate | 风控关卡 |
| position | 持仓 (客户已熟悉) |
| leg | 订单腿 (单腿/多腿策略,客户已熟悉) |

---

## 🔧 运行时与 Worker

| 技术词 | 客户可见表述 |
|--------|-------------|
| Worker / worker_loop | 后台工作进程 / 后台值班 |
| scheduler | 调度器 |
| dispatcher | 分发器 |
| monitor_loop | 监控循环 |
| reqPositions | 账户持仓查询 |
| order_updates | 订单状态更新流 |

---

## 📦 Step 2 相关(已完成)

| 技术词 | 客户可见表述 |
|--------|-------------|
| Alembic migration | 数据库结构升级 |
| DRAFT / SUBMIT 两段式 | 草稿 / 正式提交 两段式流程 |
| deterministic blocker recompute | 提交时的确定性自检 |
| Customer Queue | 客户主队列 |
| Opportunity Detail | 机会单详情页 |
| Big Status / Small Status | 大状态 / 小状态 |
| idempotency_key | 幂等键 (不暴露;说"防重复提交标识") |
| lease_owner / lease_expires_at | 任务锁 (不暴露;说"任务占用机制") |

---

## 🤖 Phase 11 Agentic Trading 范式升级(新增)

**本周 2026-04-21 方向大调整引入的新概念。客户沟通时这些是核心。**

### 范式本身
| 技术词 | 客户可见表述 |
|--------|-------------|
| Agentic Trading / agentic | 自主决策交易 (AI 定期看盘自己做决定) |
| Bar / 每 bar 决策 | 每 5 分钟周期决策 |
| continuous decision loop | 持续决策循环 |
| Lumibot | 借鉴设计的开源框架(**只借鉴设计,不引入依赖**) |

### AI 决策动作(Agent 2 输出四选一)
| 技术词 | 客户可见表述 |
|--------|-------------|
| HOLD | 持有(什么都不做,继续观察) |
| PARTIAL_CLOSE | 部分平仓 |
| FULL_CLOSE | 全部平仓 |
| ADJUST_STOP | 调整止损位 |
| Bundle / Bundle schema | 决策包(一次决策的完整输出结构) |

### AI 分级调用(Claude 模型)
| 技术词 | 客户可见表述 |
|--------|-------------|
| Sonnet 4.6 | 强力模型 (用于首次入场决策 + 触发条件升级) |
| Haiku 4.5 | 快速模型 (用于每 5 分钟持续决策) |
| prompt caching | 提示词缓存 (降低 API 成本) |
| Anthropic API key | AI 接口密钥 |
| Anthropic SDK | AI 调用库 |
| discriminated union | 类型明确的输出结构 (不暴露,说"结构化输出") |
| replay cache | 决策回放缓存 |

### 执行层新概念(mid-based 4 档定价)
| 技术词 | 客户可见表述 |
|--------|-------------|
| mid-based 定价 | 按中间价定价 |
| spread_ratio | 中间价偏移比例 |
| patient / normal / urgent / market 4 档 | 4 档耐心度:耐心/正常/紧急/直发市价 |
| 升级链 LMT → 追价 → MKT | 平仓三级升级:限价→追价→市价 (**不允许中途降级给用户**) |
| combo / BAG combo | 组合单 (2 腿策略) |
| 逐腿拆单 | 大仓位分腿拆小单下 |
| property-based test | 覆盖式自检 (针对定价逻辑) |
| 部分成交 trigger | 部分成交后的自动后续动作 |

### 触发类型(本版本限 4 种,事件类整组跳过)
| 技术词 | 客户可见表述 |
|--------|-------------|
| MA_CROSSOVER | 均线交叉触发 |
| PRICE_BREACH | 价位突破触发 |
| CONDITION_MONITOR | 条件监控框架 |
| ENTRY_TIME / 时间触发 | 时间到点触发 |
| 时间+条件组合触发 | 混合触发 |
| EVENT / EVENT_RESPONSE / 事件日历 | **事件类触发** (本版本整组跳过,等外部事件日历数据源就绪) |

### 风控层
| 技术词 | 客户可见表述 |
|--------|-------------|
| Risk Gate (7 条规则) | 风控关卡(7 条硬规则) |
| feature flag / enable_agent2_autonomous | 功能开关 (紧急时关闭 AI 自主决策降级为告警) |
| 单笔成本上限 | 单次下单金额上限 |
| KillSwitch | 紧急停机开关 |

### 前置依赖
| 技术词 | 客户可见表述 |
|--------|-------------|
| IBKR secondary username for API | IBKR API 专用第二账号 (审核 3-7 天) |
| GitHub Secrets | 云端密钥管理 |
| UAT / PROD live | 测试环境实盘 / 生产环境实盘 |
| 4 档 APP_ENV | 4 档环境区分(本地/开发/测试/生产) |
| ARM 兼容性 | ARM 芯片架构兼容 (Oracle 云部分 VM 用 ARM) |
| 2FA | 双因素认证 |

### 系统规则加强(本周新增)
| 技术词 | 客户可见表述 |
|--------|-------------|
| CLAUDE.md §18 wiki 留痕规则 | 重大决策的文档归档规则 (internal/public 分层) |
| SessionStart hook | 每次会话开头自动注入上下文机制 |
| docs-hub / discussions / decisions / specs | 四层文档分层:内部讨论 / 客户友好决策 / 设计文档 / 自动文档 |
| Karpathy 四原则 | AI 系统设计四条原则(**不暴露,说"业界 AI 设计最佳实践"**) |
| 五专家评审 | 五角色评审:软件工程 / Claude 规则 / 云部署 / 美股期权实战 / IBKR 工程 |

### 结构化字段(本周三合一重构引入)
| 技术词 | 客户可见表述 |
|--------|-------------|
| take_profit_spec / stop_loss_spec | 止盈止损结构化配置 |
| position_spec | 仓位结构化配置 |
| agent_can_adjust flag | AI 策略可调整标志 |
| SpecEditor / AdjustCheckbox / SubmitConfirmModal | 前端:配置编辑器 / 可调整勾选框 / 提交确认弹窗 |

---

## 🛠 开发相关

| 技术词 | 客户可见表述 |
|--------|-------------|
| testResults/ | 测试运行记录 |
| intent_parse_runs | 意图解析运行历史 |
| audit log | 审计日志 |
| STABLE snapshot | 稳定版本快照 |
| rollback | 版本回退 |
| dry-run | 预演模式 (不真实下单) |

---

## 📌 特殊约定

### 优先使用项目里已有的中文说法

项目 `project_handoff_background_zh.md` / `PROJECT_MIGRATION_PACK.md` 等文档里出现的中文表述,**优先复用**,不要再造新词。例如:
- 项目里说"机会单" → 用"机会单",不要翻译成"交易机会订单"
- 项目里说"人话版" → 用"人话版总结",不要翻译成"自然语言摘要"

### 客户已熟悉的技术词

以下词客户已在需求沟通时接触过,**可直接使用不翻译**:
- IBKR / 机会单 / 期权 / 策略 / 持仓 / PnL / 实盘 / 模拟
- 期权 Level 权限 (Level 1/2/3/4)
- Call/Put/Bullish/Bearish
- 行权价 / 到期日 / Delta / 保证金

### 持续补充

CC 在生成周报时如发现:
- 项目里高频出现但本字典没收录的技术词
- 某业务表述客户反馈"听不懂"

请在内部版报告末尾列出,提醒项目方更新本字典。
