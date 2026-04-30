---
hide:
  - toc
---

# :material-chart-box: 期权 AI 交易助手 · 项目看板

> **最近更新**: 2026-05-01 · [:material-history: 查看历史版本](../dashboard-snapshots/)

---

!!! tip ":material-presentation-play: 最新成果展示 · 中文意图解析能力（2026-05-01）"

    **8 个真实场景** 一次性测试 — 客户用一句中文表达交易意图，系统全自动提取标的 / 方向 / 仓位 / 止盈止损等所有字段。系统对暂不支持的场景诚实告知，不糊弄。

    [:material-arrow-right-circle: **进入完整演示**](agent-demo-2026-05-01.md)

---

## :material-target: 当前状态

<div class="progress-wrap" markdown>
<div class="progress-bar">
  <div class="progress-fill" style="width: 75%"></div>
  <span class="progress-label">整体进度 · 75%</span>
</div>
</div>

<div class="next-milestone" markdown>
<div class="icon">🚀</div>
<div class="content" markdown>
<div class="date">下一节点 · 2026-04-29 前后 · 实测期</div>
<div class="event">AI 自动盯盘首次盘中实测</div>
<div class="sub" markdown>环境: <span class="badge badge-dev">云 DEV</span> <span class="badge badge-paper">模拟账户 Paper</span></div>
</div>
</div>

AI 自主决策的**全套地基已建完**(数据采集 / 决策思考 / 风控兜底 三层全部打通), 接下来重点是**实盘验证这一关** — 先补完最后一段执行层代码, 再从模拟账户 → UAT 小额 → PROD 大额逐级验证。还剩约 25%。

---

## :material-alert-circle: 本周待处理

!!! warning "IBKR API 专用账号 · 审核中"
    已发起申请, 预计 **4/25 - 4/29** 下来。这是切 UAT 真钱实盘的前置条件, 到账后会立即通知。

---

## :material-calendar-check: 最近 1 周进展 (4/16 - 4/23)

<div class="grid cards" markdown>

-   :material-star-circle: &nbsp; __AI 自主决策地基建完__ · <span class="badge badge-star">本期重点</span>

    ---

    **以前**: 系统能识别中文意图, 但 AI 只在下单前给一次建议, 之后不管。

    **今后**: AI 拿到 9 类市场数据(标的价 / 希腊字母 / K 线 / 技术指标 / 流动性 / 大盘 / 异常活动 等), 每 5 分钟思考一次, 自主决定:

    - 继续持有 / 部分平仓 / 全部平仓 / 调整止损位

    *AI 决策的"眼睛"(看什么) + "大脑"(怎么想) + "刹车"(风险兜底) 三件套今天全部建完, 下一步就是把执行链路接上盘中实测*

-   :material-shield-check: &nbsp; __风险兜底 7 条硬规则上线__

    ---

    AI 每次决策都要过 7 道闸门:

    - **单笔仓位上限**(账户 30%)
    - **总仓位上限**(long premium 60% / short margin 70%)
    - **流动性分层**(spread > 20% 直接禁止入场)
    - **价格 sanity check**(挂单价格 ±3σ 检测)
    - **频率限制**(1h 内 ≤ 3 次, 开盘前 30min ≤ 1 次)
    - **熔断**(连续 2 次亏损调整自动暂停)
    - **Assignment 规避**(快到期 + 卖方 + 行权对自己不利 → 强制全平)

    *AI 即使判断错也跑不出这 7 条边界*

-   :material-target-account: &nbsp; __期权实战派审视, 平仓阈值定调__

    ---

    特意请期权实战派 review 平仓相关参数, 确认:

    - 距到期 5 天就强制平仓(业界主流标准, 给充分时间在好流动性时段慢慢平)
    - 紧急平仓时, 组合策略允许临时走市价单兜底(避免流动性枯竭卡死)

    *实战派背书, 避免理论参数实盘踩坑*

-   :material-test-tube: &nbsp; __质量基线: 测试 +37%, 老隐藏问题修光__

    ---

    本周新增 **49 个自动化测试**(从 144 增至 197), 同时发现并修复 **4 处一直没暴露的老测试问题**(系统升级时影响了 4 个 Phase 9 期写的测试, 之前没人发现)

    *系统每改一行, 测试都能立刻验证, 不靠人脑记*

</div>

---

## :material-map-marker-path: 未来计划

<div class="timeline" markdown>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-1">🎯</div>
<div class="tl-date">2026-04-24 ~ 04-28 · 收尾期</div>
<div class="tl-title">执行层代码补完 + 智能挂单链路</div>
<div class="tl-desc" markdown>把 AI 决策接到真实下单, 含智能价格挂单 / 撤单升级 / 部分成交跟进 · <span class="badge badge-dev">云 DEV</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-2">🔧</div>
<div class="tl-date">2026-04-29 ~ 05-08</div>
<div class="tl-title">模拟账户 5 个交易日实测</div>
<div class="tl-desc" markdown>云 DEV 模拟账户连续 5 天盘中跑, 观察 AI 自主决策效果 · <span class="badge badge-dev">云 DEV Paper</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-3">💰</div>
<div class="tl-date">2026 年 5 月中</div>
<div class="tl-title">UAT 小额真钱实盘启动</div>
<div class="tl-desc" markdown>$10k 量级真钱 Live, 2-3 个交易日量化模拟 vs 真盘的差异 · <span class="badge badge-uat">UAT Live</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-4">🚀</div>
<div class="tl-date">2026 年 5 月下</div>
<div class="tl-title">PROD 大额实盘启动</div>
<div class="tl-desc" markdown>正式交付, 大额实盘运行 · <span class="badge badge-prod">PROD Live</span></div>
</div>

</div>

---

<a class="comment-cta" href="https://docs.qq.com/doc/DVHBSd29vTHR1U1dj" target="_blank">
<span class="label">:material-comment-text-outline: · 反馈与留言</span>
有任何想法、疑问、或想看的方向 → 点这里跳转腾讯文档评论
</a>

---

## :material-archive: 查看历史版本

!!! tip ""
    每次看板更新会自动归档一份快照, 按日期可查。

    :material-arrow-right-bold: **[看板历史归档](../dashboard-snapshots/)**
