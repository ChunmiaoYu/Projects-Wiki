---
hide:
  - navigation
  - toc
---

# :material-home-city: Auckland Property Shortlist · 项目看板

> **最近更新**: 2026-04-22 · [:material-history: 查看历史版本](../dashboard-snapshots/)
>
> _自用项目 · 无外部客户 · 内部进度追踪_

---

## :material-target: 当前状态

<div class="progress-wrap" markdown>
<div class="progress-bar">
  <div class="progress-fill" style="width: 80%"></div>
  <span class="progress-label">整体进度 · 80%</span>
</div>
</div>

<div class="next-milestone" markdown>
<div class="icon">🚀</div>
<div class="content" markdown>
<div class="date">下一节点 · 2026-04-23</div>
<div class="event">PR #8 合并 + Deploy to PROD 回归验证</div>
<div class="sub" markdown>环境: <span class="badge badge-prod">PROD Live</span> · 验证 deploy.sh 顺序修复是否彻底解决 venv 被 git clean 误删的连锁问题</div>
</div>
</div>

核心交付已经完成 — 桌面版 + 云端 Web 平台都在运行, 180 天历史数据 5,306 条房源加载完毕。剩余 20% 是增量功能 (lifecycle 管理 / HTTPS / 私信) 与长期运维迭代。

---

## :material-alert-circle: 本周待处理

!!! warning "PR #8 等合并"
    `deploy.sh` 顺序调整 — `git clean` 挪到 `reset --hard` 之后, 防止 clean 在读旧 `.gitignore` 时把 `venv/bin/` 当未追踪文件删掉。合并后需要跑一次 Deploy to PROD 做回归, 确认 venv 不再被误删。

---

## :material-calendar-check: 最近 1.5 周进展 (4/11 - 4/22)

<div class="grid cards" markdown>

-   :material-cloud-check: &nbsp; __云端 Web 平台已上线 Oracle Cloud__ · <span class="badge badge-star">本期重点</span>

    ---

    **以前**: 仅 Windows 桌面单机版 (PyInstaller onedir + Ed25519 离线授权), 使用场景受限于发行方签发 license。

    **今后**: 云端 Web 平台已上 Oracle VM `aps-web` (161.33.80.117)。技术栈:

    - FastAPI + Jinja2 + Tabulator 6.x
    - SQLite (WAL) + SQLAlchemy 2.0 ORM
    - systemd 管理三进程 (web / worker / pipeline)
    - cron 每小时 NZ 0:00+6:00-23:00 自动抓取
    - GitHub Actions CI/CD (dev→PR→main→Deploy to PROD)

    *里程碑级 — 从单机工具跨到多用户云产品, 180 天历史数据 5,306 条房源完整加载。后续可挂 HTTPS/域名/私信做 SaaS 形态。*

-   :material-refresh-auto: &nbsp; __Re-enrich 机制全流程实施__

    ---

    **以前**: GIS 数据更新后需要手动 SSH 重跑 pipeline, 无独立 CLI, 无 cron 托管。

    **今后**: `reenrich.py` CLI + service 化 + cron `try/finally` lifecycle + `sudoers` 自动安装 + PROD smoke 17m44s 端到端成功 (2946 accepted + 2332 rejected)。

    *GIS 数据或评分算法升级后可一条命令触发全量重算, 不再手动处理。*

-   :material-image-check: &nbsp; __PROD 缩略图恢复 + schools 分隔符修复__

    ---

    **以前**: PROD 2947 行全无缩略图 (`requirements.txt` 漏 `matplotlib`) + 学校列 `\n` 分隔不渲染挤成一团。

    **今后**: PR #6/#7/#8 连锁修完 — 加 matplotlib + `APS.fmtSchoolList(\n→; )` + `.gitignore` 加 `venv/` + `deploy.sh` 顺序调整。Reenrich 跑完 `has_png=2947/2947`, Playwright 22/22 验证通过。

    *前端体验恢复完整, 同时暴露并修复了 deploy.sh 的 git clean 误删 venv 连锁 bug。*

-   :material-shield-check: &nbsp; __Deploy.sh 加固 (PR #5)__

    ---

    **以前**: `checkout + pull` 组合遇到 VM 本地漂移 (文件 mode 变化, 未追踪的 `.gitignore`) 会报错阻塞部署。

    **今后**: `reset --hard` 取代 `checkout+pull` (VM 漂移自动抹平), sudoers 自动安装 (`_systemctl` 检查真实 returncode)。

    *CI/CD 流水线更鲁棒, VM 人工运维残留可自愈。*

</div>

---

## :material-map-marker-path: 未来计划

<div class="timeline" markdown>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-1">🎯</div>
<div class="tl-date">2026-04-23 · 近期</div>
<div class="tl-title">PR #8 合并 + Deploy to PROD 回归验证</div>
<div class="tl-desc" markdown>验证 deploy.sh 顺序修复是否彻底解决 venv 被误删问题 · <span class="badge badge-prod">PROD Live</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-2">🔧</div>
<div class="tl-date">2026-04 月底</div>
<div class="tl-title">Listing lifecycle 管理</div>
<div class="tl-desc" markdown>加 `is_active` flag + 下架房源检测, 避免 PROD 保留已从 Trade Me 下架的房源 · <span class="badge badge-doing">规划中</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-3">💰</div>
<div class="tl-date">2026-05 中</div>
<div class="tl-title">HTTPS + 自定义域名</div>
<div class="tl-desc" markdown>当前仅 IP 直连 (`161.33.80.117:8080`), 切 HTTPS + 域名提升可用性 · <span class="badge badge-wait">待启动</span></div>
</div>

<div class="tl-item" markdown>
<div class="tl-dot tl-dot-4">🚀</div>
<div class="tl-date">2026-05 下</div>
<div class="tl-title">Private messaging + 外部集成探索</div>
<div class="tl-desc" markdown>Admin ↔ User 站内私信 + Teams/WeChat 集成探索 · <span class="badge badge-wait">待启动</span></div>
</div>

</div>

---

## :material-archive: 查看历史版本

!!! tip ""
    每次看板更新会自动归档一份快照, 按日期可查。

    :material-arrow-right-bold: **[看板历史归档](../dashboard-snapshots/)**
