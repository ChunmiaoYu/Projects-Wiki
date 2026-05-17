# Spec — Candidate 决策状态机 UI (2026-05-17, V1 北极星 #5)

> **related_north_star_path**: §4 #5 (Candidate 决策状态机 UI — V1 必做)
> **related_north_star_section**: §1 V1 必做 + §5 forward compat hook (`Candidate.progress` enum)
> **status**: design-draft, pending user ack → plan
> **预计工程量**: ~30-60 min (AI 速度, 见 §10)

---

## 1. 背景 / Motivation

北极星 §4 #5 列 **Candidate 决策状态机 UI** 为 V1 必做。当前实施情况:

**已有**:
- `Candidate.progress` 字段已存在 (nullable String(50), 在 `web/db/models.py:213`)
- 实际 enum 是 **10 值** (非北极星 §1 写的 8 值): `new / watching / research / negotiating / offered / offer_failed / offer_cancelled / contracted / settled / dropped` — 见 `web/routers/candidates.py:80` `PROGRESS_CHOICES`
- `PUT /api/candidates/{id}` 已接受 `progress` 字段更新
- 表格行内 progress 列可点击切换 (`openProgressPicker`)
- 详情面板有 `<select>` + Save Progress 按钮

**V1 真实缺口** (不是"UI 不能编辑", 是"决策闭环不可审计"):
1. **没有变更历史** — 改了 progress, 旧值丢失, 无法知道"何时从 X 改到 Y / 改的时候写了什么 notes"
2. **notes 跟 progress 解耦** — notes 是单一 Text 字段, 所有阶段共用, 无法按阶段记笔记 (e.g. "在 offered 阶段出价 $850k, 在 negotiating 阶段对方反 $920k" 无法区分)
3. **没有时间线视图** — 用户看不到决策走过了哪些阶段, 卡在哪
4. **没有跨用户协作 hook** — V2 多用户共看同一 candidate (北极星 §4 #11) 时, 必须知道"谁改的"

V1 本 spec ship 的是 **audit log + timeline UI**, 让 progress 改动结构化留痕, 支撑 V2 多人协作 + V3+ 跨阶段子能力 (loan/contract 各阶段挂事件).

**北极星 §1 跟代码事实差异**: §1 写"8 值", 代码 10 值. 本 spec 沿用代码 10 值 (用户实际用的真相), spec 通过后**同 PR 修北极星 §1 + §5 hook 描述对齐**.

---

## 2. 目标 + 非目标

### In-scope (V1)

- **新表 `candidate_progress_history`** 留每次 progress 变更
- **`PATCH /api/candidates/{id}`** (沿用现有 `PUT` 端点 + 扩 body schema) 写 progress 同时支持附 `transition_notes` 字段, 同事务插一行 history
- **`GET /api/candidates/{id}/history`** 返回该 candidate 全部 progress 变更记录
- **UI 详情面板加 timeline 段** 倒序展示历史变更 (时间 + from → to + transition_notes + 谁改的)
- **UI Save Progress 按钮旁加 transition_notes textarea** (可选填), 提交时同 PATCH body 一起送

### Out-of-scope (V2+, 本 spec 不实施)

- 状态机合法性约束 (e.g. `settled` 后不能改回 `research`) — V1 不限, 用户自由改 (见 §8 open question Q1)
- 文件附件 (合同 PDF / 看房照片 attach 到 transition) — 北极星 §4 #9 V2
- 多 user 协作 (家庭成员共看同一 candidate) — 北极星 §4 #11 V2 (但 history.user_id 留 hook)
- 邮件通知 (状态变更触发邮件) — 北极星 §4 #8 V2
- 跨阶段子能力挂载 (loan/contract/renovation 挂到 transition) — 北极星 §4 #13-15 V3+
- 批量改 progress (多选行一次改) — V1 单条改, 批量留 V2 看真需求
- 回滚操作 (撤销最近一次 progress 变更) — V1 重新点一下即可, 不做专门 undo
- Admin 跨用户 aggregate 报表 — 是北极星 §4 #6 另一个独立 spec, 本 spec 只做底层 history 表 (报表会查它)

---

## 3. 北极星 §3 5 问 cross-check

| # | 问题 (是 = 守住) | 答 | 一句话理由 |
|---|---|---|---|
| 1 | 服务 Auckland 选房决策闭环? | **是** | progress 是决策闭环骨架, history 是闭环可审计的前提 (北极星 §1 V1 必做项之一) |
| 2 | 预留商业化能力位? | **是** | history.user_id 留 hook 支撑未来 premium tier "多人协作" 收费功能 + role-gated 改权 (viewer 不能改 progress 见 §5) |
| 3 | 预留远景跨领域扩展? | **是** | history 表加 `transition_notes` Text 不限阶段, V3+ loan/contract/renovation 加新 progress 值时无须改 schema; progress enum 跟代码解耦由 `PROGRESS_CHOICES` 集中定义, 加新值改一处 |
| 4 | 避免依赖人工 admin? | **是** | 全部走 Web UI / API, 无 SSH 操作 |
| 5 | 专注护城河, 不重做 Trade Me? | **是** | Trade Me 不做"我跟这房子谈到哪一步" — 是 APS 决策闭环独有价值 |

**5 问全过. 此设计与北极星目标关系**: 直接落地 §4 #5 + 补 §1 "Candidate 决策状态机 UI 完整闭环" 条款; 同时为 §4 #6 (admin aggregate 报表) + §4 #11 (多人协作) 留底层 hook.

---

## 4. 数据模型变更

### 4.1 `Candidate.progress` (已存在, 不改)

保留 `String(50) nullable`, 不改类型不加 CHECK 约束 — 跟北极星 §7 "不要给 Candidate.progress 加 if-elif 写死 8 值业务逻辑" 一致, 留远景加新值的灵活性.

### 4.2 新表 `candidate_progress_history`

```
CREATE TABLE candidate_progress_history (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    candidate_id    INTEGER NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
    user_id         INTEGER NOT NULL REFERENCES users(id),
    from_progress   VARCHAR(50),       -- NULL = 初次设置 (e.g. 收藏时自动设 'new')
    to_progress     VARCHAR(50),       -- NULL = 清空 progress (允许)
    transition_notes TEXT,             -- 这次变更附的说明 (e.g. "对方反价 $920k")
    changed_at      DATETIME NOT NULL  -- UTC
);
CREATE INDEX ix_cph_candidate_id ON candidate_progress_history(candidate_id);
CREATE INDEX ix_cph_user_id ON candidate_progress_history(user_id);
CREATE INDEX ix_cph_changed_at ON candidate_progress_history(changed_at);
```

字段决策:
- **`user_id` 非空**: 永远从 `get_current_user` 拿; 防御未来 SSO/API key 路径 user_id 缺失
- **`from_progress` 可空**: 第一次 progress 从无到有 (e.g. 旧数据 progress=NULL 改成 'new'), `from_progress=NULL`
- **`to_progress` 可空**: 现 API 允许清空 progress (`body.progress == ""` → 设 NULL), history 也允许记 `to_progress=NULL`
- **不加 `actor_role`**: 当时 role 跟 user_id 可 join 拿; 即使 role 后续改也能反推; 不冗余
- **不加 `ip_address` / `user_agent`**: 北极星 §1 friends-and-family beta 不需要审计级别; V2 真需要再加
- **`transition_notes` 跟 `Candidate.notes` 关系**: `Candidate.notes` 仍是"当前总览备注" (现状不动); `transition_notes` 是"这次变更附的说明", 两者分开. 详 §6 UI.

### 4.3 Migration 策略

跟 PR #13 `_migrate_drop_candidate_unique` 一致 — 在 `web/db/init_db.py` 加新函数:

```
def _migrate_create_progress_history(engine):
    """Create candidate_progress_history table if absent. Idempotent."""
    from sqlalchemy import inspect
    insp = inspect(engine)
    if "candidate_progress_history" not in insp.get_table_names():
        from web.db.models import CandidateProgressHistory
        CandidateProgressHistory.__table__.create(engine)
        print("  Created candidate_progress_history table")
```

`init_tables()` 末尾追加 `_migrate_create_progress_history(engine)`. 不做"历史 backfill" — 已有 candidates 的当前 progress 不补 history 行, 新 PATCH 才开始留痕. (回填要么需要 created_at/updated_at 推断, 要么需要选一个 actor_id, 都是猜测; 不值得复杂化.)

---

## 5. API contract

### 5.1 `PATCH /api/candidates/{id}` (扩现有 `PUT`)

!!! note "为什么 PATCH 不是 PUT"
    现有端点是 `PUT`, 严格 REST 语义是 "PUT = 全量替换, PATCH = 部分更新". 当前用法本质是 PATCH (只改 notes/progress 子集). 本 spec **同 PR 改 PUT → PATCH** 并保留 `PUT` 作为 alias 1-2 个 release (无前端调用方需要兼容外, 主要防御 bookmark/客户端缓存).

**Request body**:

```json
{
  "notes": "optional, 当前总览备注 (现有行为不变)",
  "progress": "optional, 新 progress 值 (现有行为不变, 空字符串 = 清空)",
  "transition_notes": "optional, 仅当 progress 改变时记入 history"
}
```

**行为**:
- `progress` 字段提交且 **跟当前值不同** → 同事务插一行 `candidate_progress_history` (`from = 旧值, to = 新值, transition_notes = body.transition_notes 或 NULL, user_id = current user`)
- `progress` 提交但跟当前值相同 → **不插 history** (避免噪声), 即使 `transition_notes` 给了也不写 (返回 200 + 提示 "progress unchanged")
- `transition_notes` 单独提交 (不带 progress) → 拒绝 422 "transition_notes requires progress change"
- `notes` 提交 → 跟现状一样, 直接覆盖 `Candidate.notes`, 不进 history (notes 是总览不是 transition)

**Response**: 200 + 完整 candidate dict (同现状)

### 5.2 `GET /api/candidates/{id}/history`

**Response**:

```json
{
  "candidate_id": 42,
  "history": [
    {
      "id": 17,
      "from_progress": "negotiating",
      "to_progress": "contracted",
      "transition_notes": "签了 sale & purchase agreement, settlement date 2026-06-15",
      "changed_at": "2026-05-17T03:22:14Z",
      "user": {
        "id": 3,
        "identifier": "alice@aps.local",
        "display_name": "Alice"
      }
    },
    {
      "id": 14,
      "from_progress": "offered",
      "to_progress": "negotiating",
      "transition_notes": "对方反价 $920k (我们出 $850k)",
      "changed_at": "2026-05-12T19:08:33Z",
      "user": {"id": 3, "identifier": "alice@aps.local", "display_name": "Alice"}
    }
  ]
}
```

排序: `changed_at DESC` (最新在上, 跟 UI timeline 一致).

权限: 复用 `_check_owner` (owner + admin), V2 多人协作时改为 "shared_with" 集合判断.

---

## 6. UI 设计

### 6.1 Inline progress 切换 (现有, 微调)

现有 `openProgressPicker` 保留. **微调**: dropdown 提交时 **不附 transition_notes** (快速路径, 用户只想快速改). 想加说明走详情面板路径 §6.2.

### 6.2 详情面板 — Progress 段重设计

```
┌─ 进展阶段 / Progress ─────────────────────────┐
│ [当前 badge: 5 已出价 (offered)         ]    │
│                                               │
│ 改到: [<select PROGRESS_CHOICES>      ▾]      │
│ 说明 (可选): ┌───────────────────────────┐   │
│             │ 对方反价 $920k             │   │
│             └───────────────────────────┘    │
│             [ 保存进展 / Save Progress  ]    │
└───────────────────────────────────────────────┘

┌─ 变更历史 / Timeline ─────────────────────────┐
│ ● 2026-05-17 15:22  Alice                    │
│ │ negotiating → contracted                    │
│ │ "签了 S&P, settlement 2026-06-15"          │
│ │                                             │
│ ● 2026-05-12 07:08  Alice                    │
│ │ offered → negotiating                       │
│ │ "对方反价 $920k (我们出 $850k)"             │
│ │                                             │
│ ● 2026-05-05 22:14  Alice                    │
│   research → offered                          │
│   "出价 $850k"                                │
└───────────────────────────────────────────────┘
```

**实施细节**:
- 改保存按钮 onClick → POST 时 body 加 `transition_notes: textarea.value || null`
- Timeline 段在 `renderDetailPanel` 末尾追加, 异步 fetch `/api/candidates/{id}/history` 后渲染
- 空 history → 显示 "暂无变更记录 / No history yet"
- Timeline 项 hover 高亮; click 暂不做交互 (V2 可加 edit/delete)

### 6.3 Role-gated

- `viewer` 角色: progress 列只显示不可点, 详情面板 progress 段隐藏 select + button, **timeline 可看**
- `premium` / `admin`: 全开
- 后端 PATCH 端点也加 role check (`viewer` → 403)

### 6.4 Notes 字段保留现状

`Candidate.notes` (总览备注) 不动. 详情面板 Notes 段 + Save Notes 按钮 跟现状一样.

---

## 7. Forward compat hooks (给 V2/V3+ 留门)

| Hook | 防的偏离 | 怎么留 |
|---|---|---|
| `PROGRESS_CHOICES` 集中定义 (`web/routers/candidates.py:80`) | V3+ 加 `loan_approved` / `contract_signed` / `renovation_started` / `rented` 等阶段时只改一处 | 已嵌入, 本 spec 不改; 后续考虑挪到 `web/constants.py` 统一管 |
| `candidate_progress_history.user_id` 非空 | V2 多人协作 (北极星 §4 #11) 需要 "谁改的" | 本 spec 加 |
| `transition_notes` Text 不限阶段 | V3+ loan/contract/renovation 在 transition 上挂结构化数据时, 不要硬编码 schema | 先用 Text; V3+ 真要结构化时加 `transition_metadata JSONB` (SQLite 用 TEXT + json1) |
| `ON DELETE CASCADE` 候选删除自动清 history | candidates 物理删时 (现有 DELETE 端点) 不留孤儿; archive 软删 (V2+ candidates 也加 archived_at) 时 history 自然保留 | 本 spec 加 |
| 没加状态机合法性约束 | 远景加新 progress 值时不要现在写死转移图 (e.g. dropped 不能回 research 这种约束) | 本 spec 不加, V2 看真需求 |
| `from_progress` / `to_progress` 都可空 | 允许"初次设置"和"清空" 两种边界变更入 history | 本 spec 加 |

**北极星 §5 现有 hook 不能踩坏**:
- `Candidate.progress` enum 仍是 nullable String — 本 spec 不加 CHECK / 不加 enum 表
- SQLAlchemy ORM 必走 — 新 model `CandidateProgressHistory` 走 ORM, query history 也走 ORM
- 60s debounce hook 不动 — history 只针对 progress 变化, 跟 candidate 创建 debounce 解耦

---

## 8. 风险 / Open questions (留给用户决)

!!! warning "需用户决策"
    这些问题 V1 默认值已选, 用户可重定. 影响实施分支.

**Q1**: 允许 progress 倒退? (e.g. `settled` 改回 `research`)
- **默认**: 允许, 不加状态机合法性约束
- **理由**: V1 friends-and-family beta 6 人, 信任用户; 加约束反而限制真实场景 (e.g. 误点回退 / 重新评估 / settled 黄了改回 dropped)
- **替代**: 加 hard guard (settled → 不可改) / 加 soft warning (UI 确认对话框)

**Q2**: `transition_notes` 大小上限?
- **默认**: 不加上限 (Text 字段, SQLite 实际 ~1GB 上限)
- **风险**: 用户粘贴超大文本污染 timeline. 极低概率.
- **替代**: 加 ~2000 字符前端 + 后端硬限

**Q3**: 同一秒内连续 2 次点同一个 progress 切换应不应该都记 history?
- **默认**: 跟现状一致 — 跟当前值不同就记, 不做 debounce (跟 candidate 创建的 60s debounce 不同 namespace)
- **风险**: 误点 spam 几下 → history 几条噪声. 概率低 (用户没动机).

**Q4**: V2 多人协作时 (北极星 §4 #11), 别人改了我的 candidate, history 应该 push 通知我吗?
- **范围外**: 通知机制是 V2 #8 (邮件通知) + 多人协作 #11 一起设计, 本 spec 只留 user_id 字段 hook

**Q5**: Timeline 在 admin "看全用户 candidates" 视图 (北极星 §4 #6 admin aggregate) 怎么展示?
- **范围外**: 那是 admin aggregate spec 的设计点; 本 spec 只保证 history 表可被任意查询访问

**Q6**: Progress 跟 enrichment status (`Candidate.status` = pending/enriched/failed) 关系?
- **现状**: 两个独立维度 — `status` 是富化进度 (机器), `progress` 是决策进度 (用户). UI 已在同一 cell 显示 (status 优先, complete 后显 progress badge). 本 spec 不改.

---

## 9. Acceptance criteria

V1 完成判定 (全部 pass 才算 done):

1. `candidate_progress_history` 表创建迁移可重复跑 (idempotent), `python -m web.db.init_db` 跑两次不报错
2. `PATCH /api/candidates/{id}` 改 progress 同事务插 history; pytest 覆盖:
   - progress 改变 + transition_notes 存在 → 1 行 history, 字段对齐
   - progress 改变 + transition_notes 缺 → 1 行 history, `transition_notes=NULL`
   - progress 不变 + transition_notes 给 → 0 行 history, 200 OK (或 422, 见 §5 决定)
   - progress 从 NULL 改成 'new' → 1 行 history, `from_progress=NULL`
   - progress 从 'offered' 改成 NULL (空字符串) → 1 行 history, `to_progress=NULL`
   - viewer 角色 PATCH → 403
3. `GET /api/candidates/{id}/history` 返回 history 列表, owner-check 生效, admin 可看任意, viewer 可看自己的
4. UI 详情面板显示 timeline (倒序), 空状态显示空文案
5. UI Save Progress 旁的 transition_notes textarea 存在, 提交后 timeline 立刻刷新
6. Inline progress picker (快速切换) 保留, 提交不带 transition_notes (仍记 history, transition_notes=NULL)
7. ≥ 1 个 Playwright e2e 跑全链路: 改 progress + 写 transition_notes → reload 页面 → timeline 仍在
8. 北极星 §1 "progress 8 值" 同 PR 改成 "10 值" 对齐代码; §4 #5 状态从 IDEA → SHIPPED-COMMIT-XXX

---

## 10. 粗估 (AI 速度, 见全局 CLAUDE.md §20.1)

| 模块 | 估时 |
|---|---|
| `models.py` 加 `CandidateProgressHistory` 类 + `init_db.py` 加 migration | ~5 min |
| `routers/candidates.py` 扩 PATCH + 加 GET history endpoint + 路由 alias PUT→PATCH | ~10 min |
| `templates/candidates.html` 加 timeline 段 + transition_notes textarea + role gate | ~15 min |
| `tests/web/test_candidates_history.py` 新建 ~6 case (见 §9 #2) | ~15 min |
| Playwright e2e 1 个 | ~10 min |
| 同步北极星 memory + wiki | ~5 min |
| **合计** | **~60 min** (一个晚上场段) |

无云端等待 / 无外部依赖审核 / 无 IBKR 类 paper smoke 等待 — 全离线工程, 一次 ship.

---

## 相关文档

- 北极星 wiki: [auckland-property-shortlist/specs/north-star-v1-target](../specs/north-star-v1-target.md) §4 #5
- 现有 progress 实施 (project repo): `web/routers/candidates.py:80` + `web/templates/candidates.html:292`
- Migration 范式参考 (project repo): `web/db/init_db.py::_migrate_drop_candidate_unique` (PR #13)
- 关联 V2 spec (未来): admin aggregate 报表 (北极星 §4 #6) + 多人协作 (§4 #11) + 邮件通知 (§4 #8) + 文件附件 (§4 #9)
