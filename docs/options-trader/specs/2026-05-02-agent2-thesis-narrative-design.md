# Agent 2 Thesis Narrative — Spec

> **状态**：v1.3 design 经五专家评审，P0/P1 已纳入。等用户审 spec 后进 Plan。
>
> **路径分类**（项目 §13）：本 spec 描述 Agent 2 持续决策的"脉络传递"机制 — 用户每 5 min 看到的不再是无记忆的独立决策，而是承接开仓至今脉络的连续推演。

---

## 1. 目的与背景

### 1.1 真问题

当前 Agent 2 review 是**无记忆的**：
- `prompt_builder.build_review_prompt` 只塞当前 10 维 bundle 进 user message（`prompt_builder.py:49-53`）
- DB 里 `Agent2Decision.reasoning` 字段每次都存 LLM 给的事件级理由，但**下次 review 不读它**
- 后果：Agent 2 看不到"30 分钟前我已经 PARTIAL_CLOSE 了 30%"的脉络，每次都从空白开始判断剩余仓位

举例失忆决策：T+30 PARTIAL_CLOSE 30% → T+35 review 看不到刚平的事，判断"剩 7 手 + 浮盈 28%"想再 PARTIAL_CLOSE → 5 分钟内连续平仓。R5 频率限制（每小时不超过 N 次调整）只能 mechanical 兜底，不解决决策质量问题。

### 1.2 解决方向 — 滚动单条 thesis chain (architecture α)

每次 Agent 2 决策同时产出一段 `thesis_summary_zh`（**整笔交易开仓至今的当下脉络浓缩**）。下次 review 读上次的 thesis_summary + 当前 bundle → 写新一条。

**关键设计哲学**（用户原话 "不是简单叠加，是关键总结"）：每条 thesis 不是流水账，而是 **数据化的关键转折叙事** — LLM 自己决定哪些细节足够重要值得保留，哪些自然衰减。

---

## 2. 北极星对齐

**5 问 cross-check 全过**（每问标注本 spec 决策结果）：

| # | 北极星问 | 本 spec 答 |
|---|---|---|
| 1 | 客户提交 vs AI 自主流共用 schema？ | ✓ thesis_summary_zh 是 origin-agnostic, USER_SUBMITTED / AUTO_FROM_NEWS 两条流 Agent 2 接管后走相同链路 |
| 2 | 能扩展到 AI 自主流？ | ✓ Discovery Agent ship 后无人介入闭环, thesis 演进恰恰是机器跟自己对话的命脉 |
| 3 | Forward compat hooks 不踩？ | ✓ 不动 origin_type / cancel_reason 模板 / effective_until / MonitorWindow / 时区单一入口 |
| 4 | 不强加用户必填？ | ✓ thesis 是 LLM 自我产出, 用户 0 接触 |
| 5 | 不前置 AI 决策？ | ✓ 反而给 AI 决策更好的脉络上下文 |

**与北极星目标关系**：直接补 §2 远景的核心前提 — Discovery Agent 闭环大脑要求"机器跟健忘的自己反复"不发生。

---

## 3. 架构（α 滚动）

```
T0 (entry decision):
  bundle (10 维) → LLM → entry_decision { ..., thesis_summary_zh: "T0 浓缩" }
  → DB: agent2_decisions row { thesis_summary_zh: "T0 浓缩", ... }

T+5min (review 1):
  bundle + last_thesis ("T0 浓缩") → LLM
  → review_decision { ..., thesis_summary_zh: "T+5 浓缩 (基于 T0)" }
  → DB: agent2_decisions row { thesis_summary_zh: "T+5 浓缩", ... }

T+10min (review 2):
  bundle + last_thesis ("T+5 浓缩") → LLM
  → review_decision { ..., thesis_summary_zh: "T+10 浓缩 (基于 T+5)" }
  ...
```

**read window = last 1**（仅最近一条 thesis_summary_zh）。entry 框架由 bundle 维度 ⑩ `user_intent.raw_input_text` 每次 review 都重新呈现，**不**依靠 thesis chain 锚定。

---

## 4. 数据模型变更

### 4.1 Pydantic — Agent2Decision discriminated union

文件：`domain/agent2_decisions.py`

`_BaseDecision` 加字段（**4 种 action 全部继承**）：

```python
class _BaseDecision(BaseModel):
    reasoning: str
    confidence: float = Field(ge=0.0, le=1.0)
    next_check_minutes: int
    thesis_summary_zh: str  # 新增 — 必填，本期不允许 null
```

**为什么必填而不是 Optional**：strict schema → LLM 输出 schema 验证强制存在。失败兜底由 prompt + 后置处理（§6）做，不放进 schema 默认。

### 4.2 DB — agent2_decisions 表

文件：`db/models.py` `Agent2Decision`

```python
thesis_summary_zh: Mapped[str | None] = mapped_column(Text, nullable=True, default=None)
```

**为什么 DB nullable=True 而 Pydantic 必填**：
- DB 层 nullable → migration 三环境兼容（旧行不需回填）
- Pydantic 层必填 → 新行写入校验严格
- repo 层 insert 时 trace.decision.thesis_summary_zh 已经是 str（Pydantic 已校验）

### 4.3 Bundle — PositionState 加 realized_pnl（期权实战专家 P1）

文件：`domain/agent2_bundle.py`

```python
class PositionState(BaseModel):
    position_id: UUID
    strategy_type: str
    legs: list[LegPosition]
    entry_cost: float
    unrealized_pnl: float
    unrealized_pnl_pct: float
    realized_pnl: float = 0.0  # 新增 — partial close 累计已实现
    opened_at: datetime
    time_stop_at: datetime | None = None
```

**packager 改动**：`collectors/position.py` 从 `Position.realized_pnl` 列读取（已经在 `worker/jobs.py:408 _compute_realized_pnl` 算好入库）。

**为什么必须加**：thesis 5 要素第 3 条"当前持仓状态浓缩"包含"已锁利润"。bundle 不给 LLM 这个数字，LLM 只能编造 — 违反"数据 not 概念"原则。

### 4.4 Alembic Migration

新文件：`alembic/versions/20260502_0021_agent2_decisions_thesis_summary.py`

```python
"""agent2_decisions thesis_summary_zh + position realized_pnl
Revision ID: 0021
Revises: 0020
"""

def upgrade() -> None:
    op.add_column(
        "agent2_decisions",
        sa.Column("thesis_summary_zh", sa.Text(), nullable=True),
    )
    # positions 表已有 realized_pnl（model 已存在，不需加列）
    # 仅 ORM/Pydantic 层暴露给 bundle

def downgrade() -> None:
    op.drop_column("agent2_decisions", "thesis_summary_zh")
```

云部署专家 P0 兼容性：nullable=True → dev/uat/prod 三环境跑 `alembic upgrade head` 0 风险。

---

## 5. Prompt 变更

### 5.1 共享文件 — prompts/agent2_shared/thesis_writing_rules.md（新建）

完整内容见**附录 A**（spec 末）。被 entry / review prompt include。

要点：
- 长度 600 字软上限（prompt 约束，**不**事后截断）
- 数据化叙事 — 每句涉及市场状态/演进/转折必须有具体数字
- 5 要素回答（开仓判断 / 至今转折 / 持仓快照 / thesis 评估 / 下一步预期）
- 转折点 + 异常用 **加粗** / ⚠️ 标识
- 写作示范（good + bad）
- 自检清单

### 5.2 entry prompt — agent2_entry_system.md（修改）

文件：`prompts/agent2_entry_system.md`

**新增**末尾段：

```markdown
## thesis_summary_zh 输出要求

每次入场决策必须同时产出 `thesis_summary_zh` 字段：你为什么开这个仓位、
当前选的策略类型、你看到的市场关键数据、下一步预期触发条件。

{{include:agent2_shared/thesis_writing_rules.md}}

**Entry 特殊点**: 你是 thesis chain 的起点（thesis_0），后续所有 review
都基于你的输出延展。把开仓判断写得**具体到数字**：哪些指标值让你做了
这个决定，未来什么数字变化会让你改主意。
```

### 5.3 review prompt — agent2_review_system.md（修改，CC AI 专家 P1）

文件：`prompts/agent2_review_system.md`

**新增**段（区别于 entry）：

````markdown
## thesis_summary_zh 输入 + 输出

### 输入：上次的 thesis_summary_zh
你将在 user message 末尾收到上次决策时产出的 `thesis_summary_zh`：
```
=== PRIOR THESIS (T-5min) ===
{prior_thesis_text}
=== END PRIOR THESIS ===
```

### 输出：本次的 thesis_summary_zh
你必须基于上次 thesis + 当前 10 维 bundle → 写**一条全新的当下浓缩**。

{{include:agent2_shared/thesis_writing_rules.md}}

**Review 特殊纪律**:
1. **必须复述 dimension ⑩ user_intent.raw_input_text 的开仓初衷** —
   纵使 thesis chain 已经传了几十层，开仓初衷不能漂移
2. **不要复制粘贴上次 thesis** — 即使本期没有大变化, 也要数据化重述
   "无新转折, 各指标在 X-Y 区间小幅波动"
3. **数据更新必须反映** — bundle 里 PnL / VIX / 标的价等 changed 必须重写
````

### 5.4 prompt_builder 改造

文件：`services/agent2/prompt_builder.py`

```python
def build_entry_prompt(bundle: Agent2DataBundle) -> tuple[str, str]:
    system = _load_prompt_file("agent2_entry_system.md")
    user = bundle.model_dump_json(indent=2)
    return system, user

def build_review_prompt(
    bundle: Agent2DataBundle,
    *,
    prior_thesis: str | None = None,  # 新增参数
) -> tuple[str, str]:
    system = _load_prompt_file("agent2_review_system.md")
    user_parts = [bundle.model_dump_json(indent=2)]
    if prior_thesis:
        user_parts.append(
            f"\n=== PRIOR THESIS (T-5min) ===\n{prior_thesis}\n=== END PRIOR THESIS ==="
        )
    user = "\n".join(user_parts)
    return system, user
```

### 5.5 Orchestrator 接入

文件：`services/agent2/orchestrator.py`

review 路径在调 `build_review_prompt` 前先查 last thesis_summary_zh：

```python
# orchestrator review path
prior_thesis = db_reader.get_last_thesis_summary(
    opportunity_id=opportunity_id,
    position_id=position_id,
)  # 返 str | None
system, user = build_review_prompt(bundle, prior_thesis=prior_thesis)
```

新增 `db_reader` 方法：

```python
# services/agent2/db_reader.py
def get_last_thesis_summary(
    self, *, opportunity_id: UUID, position_id: UUID | None,
) -> str | None:
    """Most recent executed thesis_summary_zh for chain propagation."""
    stmt = (
        select(Agent2Decision.thesis_summary_zh)
        .where(
            Agent2Decision.opportunity_id == opportunity_id,
            Agent2Decision.executed.is_(True),
            Agent2Decision.thesis_summary_zh.isnot(None),
        )
        .order_by(Agent2Decision.created_at.desc())
        .limit(1)
    )
    if position_id is not None:
        stmt = stmt.where(Agent2Decision.position_id == position_id)
    return self.db.scalar(stmt)
```

---

## 6. Failure Handling

### 6.1 LLM 输出 thesis_summary_zh 异常处理

| 情形 | 行为 |
|---|---|
| 字段缺失 / Pydantic 校验失败 | 整个 LLM 输出失败，触发 orchestrator 既有 retry 链路（与 reasoning 缺失同处理）|
| 空字符串 `""` | Pydantic 通过（str 允许空），warning 进 alert_sink，**决策正常通过** |
| 长度 > 600 字 | warning 进 alert_sink（不截断），**决策正常通过**（依赖 LLM 下次自我修正）|
| 超长 ≥ 1500 字 | warning 升级 + 保留入库（防 token 异常 spike 时仍能调试）|
| LLM 复制粘贴上次（cosine 相似度 > 0.9）| Phase 2 polish, 当前不检测 |

invariant 16 兜底："止盈止损全自动不甩给用户" — thesis 字段不能让决策被卡住。

### 6.2 prior_thesis 不存在的处理（review 首次）

第一次 review（之前没 entry thesis 因为是 adopt 仓位 / migration 历史持仓）：
- `db_reader.get_last_thesis_summary` 返 None
- `build_review_prompt(bundle, prior_thesis=None)` 不拼 PRIOR THESIS 段
- review prompt 显式说 "如果没有 PRIOR THESIS 段，按 entry 风格写 thesis_0 起点"

---

## 7. Feature Flag（云部署专家 P1）

文件：`src/options_event_trader/settings.py`

```python
class Settings(BaseSettings):
    ...
    enable_agent2_thesis: bool = True  # kill switch — false 时跳过 thesis 产出 + 读取
```

`.env.{dev,uat,prod}` 添加 `ENABLE_AGENT2_THESIS=true`。

**降级路径**：盘中突发 prompt bug / token cost 异常时翻 false → orchestrator 走旧路径（不读 prior，不验证 thesis 字段必填，DB 列写 NULL）。

---

## 8. 测试（软件工程专家 P1）

### 8.1 单测路径（4 路径必覆盖）

文件：`tests/test_agent2_thesis_propagation.py`（新建）

1. **Normal**：mock orchestrator 跑完整 entry → review × 2 链路。assert review 2 的 user message 含 review 1 的 thesis 文本片段
2. **Empty thesis tolerance**：LLM mock 返 `thesis_summary_zh=""`。assert decision_repository 写 NULL 入库 + alert_sink 收到 warning + dispatcher 正常 dispatch
3. **Over-length warning**：LLM mock 返 1200 字 thesis。assert alert_sink warning + 完整 1200 字入库不截断
4. **Migration roundtrip**：alembic upgrade head 跑 0021 + insert pending agent2_decisions 行 thesis_summary_zh=NULL → 读出仍 NULL；再 alembic downgrade 0020 → schema 不含 thesis_summary_zh 列

### 8.2 集成测试

文件：`tests/test_agent2_review_chain_integration.py`（新建）

完整链路（mock IBKR + mock Anthropic，真 DB）：
- Step 1: entry decision → DB 写 thesis_0
- Step 2: review 1 → db_reader 读 thesis_0 → mock Anthropic 输入 prompt 含 PRIOR THESIS 段 → 写 thesis_1
- Step 3: review 2 → db_reader 读 thesis_1 → mock 输入含 thesis_1
- Assert: 每次 LLM input prompt 含正确 prior thesis；DB 链 [thesis_0, thesis_1, thesis_2] 顺序正确

### 8.3 现有测试 regression

- 现有 `tests/test_agent2_*.py`（约 47 测试）必须 0 regression
- mock fixture 的 LLM 响应需补 `thesis_summary_zh` 字段（fixture 改约 5-10 处）

---

## 9. 五专家评审记录

| # | 专家 | 等级 | 意见 | 修复落点 |
|---|---|---|---|---|
| 1 | 软件工程 | P1 | 失败路径单测必显 | §8.1 4 路径 |
| 2 | CC AI 规则 | P1 | review prompt 区别于 entry，强调"必须复述开仓初衷" | §5.3 review 特殊纪律第 1 条 |
| 3 | 云部署 | P0 | Alembic migration 三环境兼容 | §4.4 nullable=True |
| 4 | 云部署 | P1 | `enable_agent2_thesis` kill switch | §7 |
| 5 | 期权实战 | P1 | bundle ① 加 realized_pnl 否则 LLM 编故事 | §4.3 |
| 6 | IB/TWS | — | 0 影响 | — |

P0 + 全 P1 已纳入。

---

## 10. 已知限制 / Phase 2 polish

- **L1**：thesis 偷懒检测（cosine 相似度 / 完全相同检测）— 当前不实施，盘中观察后 phase 2 补
- **L2**：thesis chain DB 增长监控 — 持仓 1 hour × 12 reviews × 600 字 ≈ 7.2 KB / 持仓; trace_archiver 已有 retention 30 天，足够
- **L3**：thesis_0 entry 与后续 review 的 prompt 模板差异未来或可进一步分化（当前 5 要素共用够用）
- **L4**：与 north star §2 Discovery Agent 远景的契合点 — Discovery 抓事件 → AUTO_FROM_NEWS opportunity 走相同 thesis 链。当前 spec 不需为 Discovery 加任何特殊处理，origin-agnostic

---

## 11. Definition of Done

实施完成需打满以下打钩：

- [ ] D1. Pydantic `_BaseDecision.thesis_summary_zh: str` 必填生效
- [ ] D2. DB migration 0021 跑通本地 + dev + uat（prod 等用户运维窗口）
- [ ] D3. `PositionState.realized_pnl` 字段新增 + packager 装填
- [ ] D4. `prompts/agent2_shared/thesis_writing_rules.md` 创建，被 entry/review 共用 include
- [ ] D5. `agent2_entry_system.md` + `agent2_review_system.md` 修改完毕，include 链验证
- [ ] D6. `prompt_builder.build_review_prompt(prior_thesis=...)` 新参数生效
- [ ] D7. `db_reader.get_last_thesis_summary(...)` 实现 + orchestrator 接入
- [ ] D8. `settings.enable_agent2_thesis` flag 加，三环境 .env 加该 var
- [ ] D9. 单测 4 路径全 PASS
- [ ] D10. 集成测试链路 PASS
- [ ] D11. 现有 47 个 agent2 测试 0 regression
- [ ] D12. paper 盘中实测：跑 1-2 个 LONG_CALL 至少 3 次 review（≥15 分钟），导出 thesis chain 人审"是否有数据化叙事 + 是否传递有效"
- [ ] D13. Findings.md 删除"无 thesis 传递"待办（如有）+ 加 phase 2 polish L1 finding
- [ ] D14. spec/decisions/discussions 三处归档（项目 + docs-hub）

---

## 附录 A — thesis_writing_rules.md 完整内容

````markdown
# thesis_summary_zh 写作规则

## 一、长度
**600 字（中文字符）以内**。这是 prompt 软约束 — 由你自己控制，
**绝不允许事后截断**。若上次产出超过 600，本次必须主动缩短回到范围内。

## 二、风格 — 数据化叙事
**多用数据，少用空泛或概念性语言**。每一句涉及"市场状态 / thesis 演进 / 转折"
的描述，必须可对应到 bundle 里的具体数字。

### 禁止（无数据支撑的纯结论）
- "市场转弱" / "情绪较差" / "波动加剧" / "成交活跃"
- "thesis 仍然成立" / "风险升温" / "走势符合预期"

### 要求（数据 + 结论）
- "VIX 从 14.2 跳到 18.7（+31%）+ SPY -1.2% → **大盘转弱**"
- "Put/Call ratio 从 0.7 升至 1.3 → 期权情绪转空"
- "ATR(14) 从 2.1 升到 3.4（+62%）→ 波动幅度扩大"
- "本仓 strike 周边成交 4500 手 vs 5 日均 1100 手（×4）→ **异常活跃**"

## 三、必须回答的 5 个问题
1. **开仓判断**：一句话从 `user_intent.raw_input_text` 提炼，含 symbol +
   方向 + 主逻辑
2. **至今关键转折**：≤3 个，按时间序列。每个含 **时点 + 具体指标数字 +
   对持仓影响**
3. **当前持仓数据快照**：剩余手数 / 入场均价 / 当前价 / **已锁利润
   (realized_pnl)** / 浮盈%
4. **当前 thesis 评估**：仍成立 / 部分动摇 / 失效 + 至少 2 个数据点佐证
5. **下一步预期触发**：具体阈值（"VIX > 22 或 SPY < 495"），不要"如果
   市场恶化"

## 四、转折点与异常优先
- 用 **加粗** 或 ⚠️ 标识异常事件 + 大转折，让下次读者扫读即可定位
- bundle 维度 ⑨ `unusual_activity` + 维度 ⑤ `technical_indicators` 的
  突变 **必须显式提及** — thesis 是给下次 review 的 LLM 看的脉络叙述，
  不是给当下的 LLM 自查（当下的 LLM 自己有 bundle 全量，下次的 LLM 没有
  过去的 bundle）

## 五、写作示范

### ✅ 好范例
```
AAPL 开仓 230 突破做多 call（1 手, 入场 $5.30）, 用户原意"突破后跟涨"
+ 财报前 15 天。

至今 3 转折:
- T+18min: 标的 232.5（+1.1%）, 浮盈 +35%, ATR(14) 2.1→2.6（+24%）但
  仍在正常区间
- T+47min: ⚠️ VIX 14.2→18.7（+31%）+ SPY -1.2%, 本仓浮盈回吐到 +18%,
  **大盘风险变量出现**
- T+52min: PARTIAL_CLOSE 30%（3 手）@ $7.10, 锁利约 $540, 剩 7 手

当前: 7 手, 均价 $5.30, 现价 $6.80, 浮盈 +28%, 已实现 +30%。
评估: **thesis 部分动摇**。开仓"突破跟涨"未失效（标的仍站 230 上方）,
但市场背景从"宽松"切到"VIX 升温", 风险/收益比变差。
预期: VIX > 22 或 SPY < 495 → FULL_CLOSE; VIX 回 16 下方且 AAPL 重回
232 → ADJUST_STOP 收紧到 +15%。
```

### ❌ 烂范例
```
AAPL 看涨, 走势符合预期。市场略有波动, 但整体可控。已部分锁利, 持仓
状态健康。
```
（无任何数据 / 无具体阈值 / 无转折点 — 下次 review 等于没看）

## 六、自检（产出前快速过一遍）
- [ ] 提到的每个市场/指标变化都有具体数字
- [ ] 至少识别 1 个转折点（如真无则诚实说"无明显转折，HOLD 区间"）
- [ ] 持仓状态全部用数字表达（手数 / 价 / PnL%）
- [ ] 下一步预期写了具体阈值不是模糊语言
- [ ] 字数 ≤ 600
````

---

**END OF SPEC**
