---
hide:
  - toc
---

# :material-chart-box: 期权 AI 交易助手 · 项目看板

> **最近更新**: 2026-04-22 · [:material-history: 查看历史版本](../dashboard-snapshots/)

---

## :material-target: 当前状态

<div class="progress-wrap" markdown>
<div class="progress-bar">
  <div class="progress-fill" style="width: 65%"></div>
  <span class="progress-label">整体进度 · 65%</span>
</div>
</div>

<div class="next-milestone" markdown>
<div class="icon">🚀</div>
<div class="content" markdown>
<div class="date">下一节点 · 2026-04-24 ~ 04-28 · 实测期</div>
<div class="event">AI 自动盯盘首次盘中实测</div>
<div class="sub" markdown>环境: <span class="badge badge-dev">云 DEV</span> <span class="badge badge-paper">模拟账户 Paper</span></div>
</div>
</div>

系统核心功能已建完, AI 自动盯盘的设计也通过。接下来重点是 **实盘验证这一关** — 从模拟账户 → UAT 小额 → PROD 大额 逐级验证, 这段路还有约 35%。

---

## :material-alert-circle: 本周待处理

!!! warning "IBKR API 专用账号 · 审核中"
    已发起申请, 预计 **4/25 - 4/29** 下来。这是切 UAT 真钱实盘的前置条件, 到账后会立即通知。

---

## :material-calendar-check: 最近 1.5 周进展 (4/11 - 4/22)

<div class="grid cards" markdown>

-   :material-star-circle: &nbsp; __AI 下单后自己盯盘__ · <span class="badge badge-star">本期重点</span>

    ---

    **以前**: AI 选策略 → 下单 → 死板止盈止损 → 结束

    **今后**: AI 下单后每 5 分钟自主看盘, 决定:

    - 继续持有
    - 部分平仓
    - 全部平仓
    - 调整止损位

    *不用手动盯盘, 关键时点 AI 主动出手, 止损也能自动走完不漏*

-   :material-speedometer: &nbsp; __系统反馈逻辑更清晰__

    ---

    **以前**: 三档 "完全支持 / 仅建议 / 部分支持" 容易混淆

    **现在**: 两档 <span class="badge badge-done">能做</span> <span class="badge badge-wait">暂时做不了</span>

    *一眼看懂*

-   :material-message-text-outline: &nbsp; __复杂中文指令识别更稳__

    ---

    像 *"NVDA 财报那天涨破 \$900 就买 call, 涨一倍平仓, 跌到 \$850 止损"* 这种复合指令

    现在能稳定识别 **每一个细节**:

    - 入场条件
    - 止盈
    - 止损

-   :material-check-decagram: &nbsp; __盘中实盘已通过测试__

    ---

    4 月 17 日在 <span class="badge badge-paper">模拟账户</span> 完成真实下单全流程测试

    发现并修复了 **10 多个边界问题**, 为上 UAT 实盘铺好路

</div>

---

## :material-map-marker-path: 未来计划

<div class="timeline" markdown>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-1">🎯</div>
<div class="tl-date">2026-04-24 ~ 04-28 · 实测期</div>
<div class="tl-title">AI 自动盯盘首次盘中实测</div>
<div class="tl-desc" markdown>周五首次盘中跑起来看实时决策效果, 周一周二继续观察盘中表现 · <span class="badge badge-dev">云 DEV Paper</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-2">🔧</div>
<div class="tl-date">4/29 ~ 5/10</div>
<div class="tl-title">云 UAT Live 启动 (边补齐边跑)</div>
<div class="tl-desc" markdown>基于实测期结果调优, 边跑真钱边补上分级调用 / 智能挂单 / 条件触发 · <span class="badge badge-uat">云 UAT Live</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-3">💰</div>
<div class="tl-date">2026 年 5 月中</div>
<div class="tl-title">PROD 上线准备 (UAT 稳跑后切 PROD)</div>
<div class="tl-desc" markdown>UAT Live 稳跑验证 + PROD 配置准备, 等条件齐备切生产 · <span class="badge badge-uat">UAT Live</span></div>
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
