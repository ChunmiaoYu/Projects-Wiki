---
hide:
  - toc
---

# :material-home-city: 奥克兰在售房产搜索工具 · 项目看板

> **最近更新**: 2026-05-17 · [:material-history: 查看历史版本](../dashboard-snapshots/)
>
> _自用项目 · 无外部客户 · 内部进度追踪_

---

## :material-target: 当前状态

<div class="progress-wrap" markdown>
<div class="progress-bar">
  <div class="progress-fill" style="width: 85%"></div>
  <span class="progress-label">整体进度 · 85%</span>
</div>
</div>

<div class="next-milestone" markdown>
<div class="icon">🎯</div>
<div class="content" markdown>
<div class="date">下一节点 · 待启动</div>
<div class="event">Candidate 决策状态机 UI — 草稿 spec + brainstorm</div>
<div class="sub" markdown>环境: <span class="badge badge-prod">PROD Live</span> · `Candidate.progress` 枚举 8 值 (new → research → offered → negotiating → contracted → settled + watching/dropped) 后端已落, 缺前端 inline 编辑 + 状态转移时间轴。北极星 V1 完成态 3 项剩余之一。</div>
</div>
</div>

云端 Web 平台核心已稳跑 — Caddy + property.yuanshan1900.com auto HTTPS 在线, 180 天历史数据 5,306 房源 + 6 个月自动归档全跑通, admin/premium/viewer 三角色权限矩阵 ship。剩余 15% 集中在 V1 收尾的三件 IDEA: Candidate 决策状态机 UI / Admin 跨用户加总报表 / 用户 onboarding 自助 (减少 admin SSH 干预)。

---

## :material-alert-circle: 本周待处理

!!! info "V1 收尾三件套 — 等用户挑主线"
    北极星 V1 完成态判定剩三项 IDEA, 都未排具体日期, 待用户定下一主线:

    - **Candidate 决策状态机 UI** (`progress` 字段 inline edit + 状态转移时间轴)
    - **Admin 多用户 aggregate 报表** (跨 6 用户决策状态 rollup, e.g. "全用户共 N offered / M contracted")
    - **用户 onboarding 自助** (signup + email verify + 密码自助重置, 减少 admin SSH 干预)

---

## :material-calendar-check: 最近进展 (5/15 - 5/17)

<div class="grid cards" markdown>

-   :material-compass-rose: &nbsp; __项目北极星 memory + wiki 首版 ship__ · <span class="badge badge-star">本期重点</span>

    ---

    **以前**: 无统一"V1 完成态判定" / "远景子能力" / "永不做"清单, 每次 brainstorm 容易长尾偏离 (做 generic real estate / 跨城市 / 经纪撮合 / AVM 等)。

    **今后**: `project_vision_and_north_star.md` memory + wiki spec 双版 ship — V1 目标 + 最终远景 (V3+ 跨贷款/合同/装修/招租) + 决策 5 问 cross-check + 中间路径 16 条 spec 状态表 + forward compat hooks + 历史决策 + 反偏离警示。所有 spec / plan / brainstorm 前必读。

    *从"凭感觉接需求"升级到"5 问门禁过滤"; 跟 options-ai-trader 同结构对齐, APS 精简到 ~350 行 (无 real money / agent / LLM / market hours)。*

-   :material-key-variant: &nbsp; __Admin 账号简化 + login form 放开 (PR #14)__

    ---

    **以前**: PROD admin 用 email 形式 (`admin@aps.local`) + 用户忘了原密码; login form `<input type="email">` 浏览器对纯 `admin` 强制 @ 校验拦, 无法提交简短 username。

    **今后**: 模板 `login.html` 改 `type="text"` + label "Email or username"; PROD DB 重置 `admin@aps.local → admin / changeme` (其他 5 用户不动); end-to-end 验证 POST /auth/login → 303 + JWT cookie role=admin。

    *forward compat 顺带预留 — 商业化未来 SSO username 可能是 phone/UUID 不带 @, login form 现在就放开。*

-   :material-server-network: &nbsp; __VM IP 漂移教训 + Caddy HTTPS 文档回写__

    ---

    **以前**: 上次 commit 92d2aa8 把 ephemeral `161.33.80.117` → Reserved `152.69.183.186` 同步了 memory/docs, **漏改 GitHub Secret `APS_VM_IP`** → PR #14 第一次 deploy `dial tcp ***:22: i/o timeout`。同时发现 PROD 早就上了 Caddy + property.yuanshan1900.com + auto HTTPS, **文档完全没记录** (旧 docs 还说"暂不配 HTTPS")。

    **今后**: `gh secret set APS_VM_IP --body 152.69.183.186` 修后第二次 deploy 成功; 北极星 §5 forward compat hook 加一条 "GitHub Secret 跟 memory/docs 必同 commit sync"; §6 历史决策回放补 Caddy + HTTPS 落地记录 (落地时间已不可追溯)。

    *系统性教训 — 配置类事实 (IP / port / 密钥路径) 改动必须 grep memory + GitHub Secrets 同 commit 同步, 不允许"下个 session 再说"。*

-   :material-archive-clock: &nbsp; __回顾: 6 个月归档 + Cron TZ 系统级修复__

    ---

    **以前** (4 月底前): PROD 数据无 lifecycle 概念无限增长; cron schedule 时区错乱 (NZ 01-05 AM 错跑 + NZ 13-17 PM 缺失, PR #11 改 `CRON_TZ=` 没用 — vixie-cron user crontab 不识别)。

    **今后** (PR #13, 2026-04-23): `Property.archived_at` 列 + pipeline 幂等 `archive_old_listings()` 跑 180 天阈值; shortlist API / admin KPI 加 `WHERE archived_at IS NULL`; UPSERT 复活机制 (Trade Me 再上架同 listing_id 清 archived_at)。Cron TZ 终极方案: `timedatectl set-timezone Pacific/Auckland` 把 VM 系统 TZ 切到 NZ, DST tzdata 自动处理。

    *V1 数据规模可控 + 时间感知正确, 为 candidate 决策状态机 UI 铺路。*

</div>

---

## :material-map-marker-path: 未来计划

<div class="timeline" markdown>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-1">🎯</div>
<div class="tl-date">近期 · 待启动</div>
<div class="tl-title">Candidate 决策状态机 UI</div>
<div class="tl-desc" markdown>`Candidate.progress` enum 8 值 inline edit + 状态转移时间轴 (new → research → offered → negotiating → contracted → settled, 分支 watching/dropped) · <span class="badge badge-wait">V1 必做</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-2">📊</div>
<div class="tl-date">近期 · 待启动</div>
<div class="tl-title">Admin 多用户 aggregate 报表</div>
<div class="tl-desc" markdown>跨 friends-and-family beta ~6 用户的决策状态 rollup (全用户共 N offered / M contracted), admin 看小群体加总视图 · <span class="badge badge-wait">V1 必做</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-3">🪪</div>
<div class="tl-date">近期 · 待启动</div>
<div class="tl-title">用户 onboarding 自助</div>
<div class="tl-desc" markdown>signup + email verify + 密码自助重置, 减少 admin SSH 干预 (PR #14 admin 重置走的就是 SSH 一次性运维, 应自动化) · <span class="badge badge-wait">V1 必做</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-4">📧</div>
<div class="tl-date">V2 · 远景</div>
<div class="tl-title">邮件通知 + 文件附件 + 多用户协作</div>
<div class="tl-desc" markdown>candidate 状态变更 / daily digest / 出价过期提醒; 合同 PDF / 看房照片 attach; 家庭成员共看同一 candidate · <span class="badge badge-wait">远景</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-5">🏘️</div>
<div class="tl-date">V3+ · 跨领域闭环</div>
<div class="tl-title">贷款 / 合同 / 装修 / 招租模块</div>
<div class="tl-desc" markdown>从"选房筛选"演进到"完整买/卖/装修/出租闭环", 出租实际租金回报反馈到评分模型. **永远聚焦 Auckland 不跨国** · <span class="badge badge-wait">远景</span></div>
</div>

</div>

---

## :material-bug: 当前已知问题

!!! warning "技术债务 (未解决, 不影响主流程)"

    - **Tabulator remote filter 对数字列无效** — `headerFilter: "input"` 客户端会先把数字列过滤成空, 需用自定义 filter popover (已在 shortlist 页用 Excel 风格 popover 绕过)
    - **Pipeline 60 秒防重入靠文件锁** — dev 模式 uvicorn reload 会起两个调度器线程; 生产环境无此问题, 但 dev 体验略糙
    - **统一通知架构未完工** — Candidates SSE 内部仍 1 秒 DB 轮询, 计划改用 `asyncio.Event` 推送 (worker / pipeline HTTP 回调架构已 ship, 这步是收尾)
    - **Trade Me 云端 IP 限制** — Oracle Cloud Melbourne IP 直接 curl `trademe.co.nz` 返回 403, 但 `api.trademe.co.nz` JSON API 正常; 如果将来需要网页抓取 (非 API) 需代理或其他方案

---

## :material-archive: 查看历史版本

!!! tip ""
    每次看板更新会自动归档一份快照, 按日期可查。

    :material-arrow-right-bold: **[看板历史归档](../dashboard-snapshots/)**
