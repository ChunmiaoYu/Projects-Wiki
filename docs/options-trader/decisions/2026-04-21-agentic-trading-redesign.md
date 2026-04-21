# 决策记录 — Agentic Trading 重构（Agent 2 每 bar 自主决策）

> **日期**：2026-04-21
> **参与评审**：5 位全域专家（软件工程 / Claude Code 规则 / Oracle 云部署 / 美股期权实战 / IBKR 工程师）
> **相关设计 Spec**：[`../specs/2026-04-21-agentic-trading-redesign-design.md`](../specs/2026-04-21-agentic-trading-redesign-design.md)
> **原始讨论（internal，对外不可见）**：保留在项目 repo 内

---

## 触发的 §18 条款

按全局 CLAUDE.md §18 重大决策留痕规则，本方案 **同时命中全部 7 条触发清单**：

1. ✅ 触发五专家 brainstorming
2. ✅ invariant / 核心规则修改（项目 CLAUDE.md §5 多条 invariant 要改）
3. ✅ 新增主要依赖（Anthropic SDK / pandas-ta / pandas-market-calendars）
4. ✅ 架构级选型（LLM 厂商 OpenAI → Claude + 决策范式反转）
5. ✅ 数据模型 / API 契约重大变更（新表 agent2_decisions + 新 prompt 文件 + 新 collectors 模块）
6. ✅ 运维策略调整（UAT/PROD 从 paper→live + IBKR secondary username for API）
7. ✅ 安全 / 风控规则变更（新风控层 7 条硬规则）

7/7 命中 —— 这是项目历史上最大规模的一次方向调整。

---

## 背景（为什么要改）

### 原设计
项目原本定位为 **"AI 辅助用户决策"**：用户用中文描述交易意图，Agent 1 解析成结构化 JSON，Agent 2 一次性生成 2-3 个策略方案，用户按按钮确认，系统下单。止盈止损由规则化触发（价格到达阈值自动平仓）。

### 转变契机
客户已授权 **$10,000 USD 级别测试资金**（允许亏损），要求 **实盘可跑（paper 先跑通，再切 live）**。开源 Lumibot 项目（Lumiwealth，4 万 GitHub stars，MIT 协议）官方主打 **agentic trading** 方向，验证了 "LLM 每 bar 自主决策持仓" 的可行性和市场兴趣。

客户的明确需求升级为：**"LLM 在每个 bar（默认 5 分钟）自主决策持仓，系统只负责执行和风控兜底"**。用户只在建仓时提供初始意图，之后 AI 全自主管理。

### 不改的代价
如果继续老方案，客户的 $10k 测试资金无法体现"AI 自动交易"的价值：用户仍要守着屏幕按按钮，项目沦为"AI 翻译下单工具"，和已有 IBKR 官方 app 无本质区别。

---

## 决策内容（改了什么）

### 核心变化一览表

| 维度 | 从 | 到 |
|---|---|---|
| 决策主体 | 用户按按钮 | Claude LLM 自主判断 |
| Agent 2 触发 | 提交机会单后一次 | 入场后每 5 min 循环 |
| 止盈止损 | 规则化触发 | LLM 看 9 类市场数据自判 |
| LLM 厂商 | OpenAI | Claude（Haiku 常规 / Sonnet 关键节点） |
| 反馈/词典 | 活闭环 | 功能开关冻结（保留代码和数据） |
| 触发类型 | 立即 + 时间 + 部分条件 | 4 种（立即 / 时间 / 条件 / 时间+条件组合） |
| 环境枚举 | 3 档（dev / uat / prod） | 4 档（local-dev + cloud-dev + uat + prod） |
| UAT/PROD 账户 | 都 paper | **UAT = 小额 live / PROD = 大额 live** |

### 10 大改造模块

1. **Agent 2 彻底重写** —— 从"一次生成策略"变"每 bar 持续决策"，输出 4 种 action：HOLD / PARTIAL_CLOSE / FULL_CLOSE / ADJUST_STOP
2. **数据采集扩容到 9 类** —— 持仓状态 / 标的价 / 期权希腊 / 多粒度 K 线 / 技术指标 / 期权流动性 / 大盘（SPY+VIX）/ 期权链异常活动 / 用户意图（原始 Agent 1 解析结果）
3. **新风控层 7 条硬规则** —— 单笔仓位 / 总仓位 / 流动性分层 / Sanity check / 频率 / 熔断 / assignment 避免
4. **限价单 mid-based 4 档定价** —— 借鉴 Lumibot 的 SmartLimit 算法，从 bid-based 改 mid-based（`limit = mid + ratio × (ask - mid)`）
5. **条件触发 runtime 实现** —— MA_CROSSOVER（均线交叉）和 PRICE_BREACH（价格突破）本次实现，同时新增"时间+条件组合"触发
6. **Prompt Caching 优化** —— 利用 Anthropic 5 min TTL 缓存 system prompt 和用户意图，估算节省 30-50% input token
7. **Worker 异步化** —— asyncio + AsyncAnthropic，避免 LLM 调用阻塞其他任务
8. **新审计基建** —— 新表 agent2_decisions + trace JSON 落盘，修复历史审计缺口
9. **反馈系统冻结** —— 4 个 feature flag 关闭词典写入 / 反馈消费 / 编辑信号 API / Agent 1 prompt 词典拼接（代码和数据保留）
10. **三环境重命名 + 账户模式变更** —— 新增 local-dev 环境用于灵活开发；UAT 从 paper 切 live（小额 $10k 量级）

### 成本估算

- 单仓位每日约 **78 次** LLM review（美股 6.5 小时 × 60 / 5 分钟）
- Haiku 单次成本约 $0.005，Sonnet 升级约 15-20%，混合平均约 **$0.006/次**
- 10 仓位/月估算：**$120-150/月** LLM 成本（含 prompt caching 节省）

---

## Why（为什么选这个方案）

### 选项对比

| 选项 | 优点 | 缺点 | 结果 |
|---|---|---|---|
| **A. 全面改造为 agentic（本方案）** | 符合客户实盘自动化需求；借鉴 Lumibot 成熟模式；未来可扩展 | 开发量大（4-5 周）；首次接入 Claude 新依赖 | ✅ **选择** |
| B. 直接用 Lumibot 作为依赖 | 省 SmartLimit / indicators / 日历等基础代码 | **License 矛盾（README MIT / LICENSE 文件 GPL-3.0）**，GPL 传染会强制整个项目开源，不接受商业闭源 | ❌ |
| C. 维持老方案 + 小改 | 工作量小 | 不解决客户核心需求；$10k 测试资金无法验证 agentic 价值 | ❌ |
| D. Agent 2 保持规则化止盈止损 + 只在建仓用 AI | 风控边界清晰 | 违反 invariant 16（止盈止损不甩给用户）；AI 无法处理复杂市场变化（IV pump / 流动性突变）| ❌ |

### 关键判断

- **Lumibot License 矛盾**：本 Claude agent 通过 `cat LICENSE | head -5` 核实，`LICENSE` 文件是完整 GPL-3.0 文本，`setup.py` 和 `README.md` 虽写 MIT 但法律上 LICENSE 文件 override。不能作为商业项目依赖
- **选项 A 借鉴 6 个设计**：`sleeptime` + 交易时间感知 hook / `@agent_tool` 装饰器 / SmartLimit 定价（mid-based）/ Agent Replay Cache / JSON trace 审计 / Broker 抽象边界
- **选项 A 避免 3 个 Lumibot 盲点**：Lumibot 自身缺持仓监控+自动止损升级 / 欧式 Greeks 模型不适合美式期权 / 多腿部分成交无原子回滚

---

## 风险和回滚

### 已知风险和应对

| 风险 | 严重度 | 应对 |
|---|---|---|
| Anthropic API 限流 / 故障 | 高 | Haiku + Sonnet 双模型备选；失败重试 3 次 → 保守 HOLD + 告警 |
| Worker 异步化重构引入死锁 | 高 | Step C 完成后专项压测；回滚到同步版本备份 |
| Prompt caching 5 min TTL 边界命中率不稳 | 中 | 监控 cache hit rate；<70% 告警，持续 3 日低于阈值触发复查 |
| IBKR secondary username 2FA 仍触发 | 中 | UAT live 启动前专项验证 |
| 盘中 Agent 2 决策迭代堆积 | 高 | 迭代延迟观测 + 连续 3 次 ERROR 自动降级到 "Haiku only" |
| Claude 输出格式漂移（非合规 JSON） | 高 | Pydantic discriminated union 严格验证；校验失败 → HOLD + 告警 |
| IBKR market data 订阅超 100 并发上限 | 高 | 订阅池复用（underlying/SPY/VIX 全项目共享）+ 期权合约 snapshot 模式 |
| 多腿 combo 订单部分成交 | 中 | 部分成交立即触发计划外 Agent 2 review；BAG 不走 MKT 档 |

### 回滚策略

- **代码层**：所有新模块有独立 feature flag（`enable_agent2_autonomous` 总开关），关闭即回到老 Agent 2 路径
- **数据层**：`agent2_decisions` 新表独立，不影响 opportunities / strategy_runs / positions。回滚不删表
- **配置层**：`.env.uat` / `.env.prod` 改回 paper + `enable_agent2_autonomous=false`，systemd 重启 Worker 即回滚完成
- **分阶段验收**：Step G paper 跑 5 天 → Step G+ UAT small live 跑 2-3 天 → 确认无问题后才 PROD live 上线

---

## 实施时间线

```
Step A (1.2天) 反馈冻结 + Agent 1 简化 + 环境配置
  ↓
Step B (3.6天) 数据采集 9 类 + 鲜度标记 + 订阅池
  ↓
Step C (2.4天) LLM 决策层 + 状态机 + Anthropic SDK
  ↓
Step D (1.2天) 风控层 7 条
  ↓
Step E (2.4天) 执行层 mid-based 定价 + 升级链
  ↓
Step F (2.4天) 条件触发 runtime (MA_CROSSOVER + PRICE_BREACH + 时间+条件组合)
  ↓
Step G (1.8天) Paper 端到端测试
  ↓
Step G+ (2-3天) UAT small live 验证 (Paper vs Live 差异)
  ↓
Step H (0.6天) 文档 + runbook
```

**开发工作量**：~18 工作日（+ 20% buffer）
**实际排期**：**4-5 周**（含 session 中断 / 方案迭代 / bug 修复）

---

## 五专家评审结果

| Round | 专家 | P0 问题 | P1 问题 | 结果 |
|---|---|---|---|---|
| 1 | 软件工程 | 7 | 8 | ✅ 通过（全部应用） |
| 2 | Claude Code 规则 / 记忆系统 | 4 | 6 | ✅ 通过（全部应用） |
| 3 | Oracle 云部署 | 3 | 6 | ✅ 通过（全部应用） |
| 4 | 美股期权实战 | 4 | 6 | ✅ 通过（全部应用） |
| 5 | IBKR / TWS 工程师 | 5 | 6 | ✅ 通过（全部应用） |
| **总计** | | **23** | **32** | **全员认可** |

全部 55 个 P0/P1 问题已应用到 Spec v5 最终版。

---

## 脱敏自查 ✓

提交前确认：
- [x] 无 API key / token / secret
- [x] 无 IBKR 具体账户号
- [x] 无客户真实姓名 / 邮箱 / 具体金额（只用"$10k 量级"等脱敏表述）
- [x] 无 VM IP / SSH key 路径
- [x] 无内部实盘策略参数具体数值（默认值只作示例，真值在 config）
