# Spec — Admin 驱动用户管理 (V1 北极星 #7, 方案 X)

> **状态**: design-draft-pending-review
> **对应北极星**: §4 中间路径 #7 (用户 onboarding — 减少 admin SSH 干预)
> **核心 5 问**: Q4 (反"依赖人工 admin")
> **依赖**: 现有 `web/auth/` + `User` 模型 + `UserAuditLog` + `web/routers/admin.py` (user CRUD 已 ship)
> **粗估**: ~1-1.5 hr AI 速度 (改 + 加, 主要 reuse 现有 admin user CRUD)

> **本 spec 替代 2026-05-17 早些时候的 v1 (544 行 / 全 email infra), 用户审议后选方案 X — 见北极星 §6 历史决策 2026-05-17 决策行**

---

## 1. 背景 / Motivation

V1 北极星 §4 #7 标记为 **V1 必做**, 直接服务 5 问 Q4 ("是否避免依赖人工 admin 手工操作").

**v1 spec (已废)**: 544 行 / 完整 email infra (Gmail SMTP / verification token 表 / `/signup` + `/forgot-password` 公共端点 / 邮件模板) — 用户审议后认为 over-scope:
- friends-and-family beta 6 人都是熟人, admin 自己拉, **不需要陌生人公开 signup**
- email infra 引入新依赖 (smtplib 配置 / App Password 管理 / 垃圾箱 deliverability 风险) — V1 不值
- 真正痛点 = "admin SSH 改密码" — 不必须邮件解决, **admin UI 一键够**

**方案 X (本 spec)**: 不引入任何 email 基础设施, admin UI 一切搞定:
- admin 在 UI 点 "新建用户" → 系统**生成临时密码** → admin 看到密码截图发用户 (WeChat / Teams / 当面)
- 每个 user 行有 "重置密码" → 同上生成临时密码
- 用户拿到临时密码 login → **强制跳 `/change-password` 改成自己的密码** → 才能用其他功能

**反偏离对照**: V2+ 加 public signup / email verification / forgot-password 自助 时, 现有架构必须留 forward compat hook (见 §8), 不阻断升级路径.

---

## 2. 目标 + 非目标

### In scope (V1)

| # | 功能 | 实施位置 | 备注 |
|---|---|---|---|
| (a) | **Admin UI "新建用户"** 系统生成临时密码 | `web/routers/admin.py::create_user` (改) + `web/templates/admin.html` (改 inline form → modal + 显示临时密码) | admin 填 identifier + role + display_name; 不再填 password |
| (b) | **Admin UI "重置密码"** 系统生成临时密码 | `web/routers/admin.py` 新加 `POST /api/admin/users/{id}/reset-password` + admin.html 加按钮 | 现有 edit-modal `password` 字段保留 (admin 手输也算 reset 路径), 加 "生成临时密码" 按钮快捷路径 |
| (c) | **`force_password_change` boolean column** | `web/db/models.py::User` + `init_db.py::_migrate_*` 新函数 | 默认 False; admin create/reset 路径设 True |
| (d) | **`password_changed_at` DateTime nullable** | 同上 | 审计 last 改密时间 |
| (e) | **`created_via` String(50) nullable** | 同上 | enum-like soft: `'admin_seed'` / `'admin_create'` / `'self_signup'` (V2 留位) |
| (f) | **`/change-password` 强制页面** | `web/routers/auth_routes.py` 新加 `GET /change-password` + `POST /auth/change-password` + `templates/change_password.html` | 用户 login 后若 `force_password_change=True` 必跳此页 |
| (g) | **Middleware 强制跳转** | `web/app.py` 加 middleware 或 `web/auth/deps.py` 加 dependency | 任何 logged-in request 若 fpc=True 且 path 不在白名单 (`/change-password` / `/auth/logout` / 静态资源) → 302 跳 `/change-password` |
| (h) | **临时密码生成器** | `web/auth/auth.py` 加 `generate_temp_password()` | `secrets.token_urlsafe(9)` ≥ 12 char, URL-safe alphanumeric |
| (i) | **Audit log 扩展** | 复用现有 `UserAuditLog` 表 | admin reset 写 `action='password_reset_by_admin'`, 用户改密写 `action='password_changed'` |

### Out of scope (V2+)

- **公开 `/signup` endpoint** — friends-and-family beta 不开放陌生人注册
- **Email 基础设施** (Gmail SMTP / Resend / Mailgun / 邮件模板 / `email_verification_tokens` 表)
- **用户自助"忘记密码"** (V2 商业化才需要, 当前用 admin "重置密码" 兜底)
- **`email_verified_at` 字段** — V1 不发邮件无从验证; email 仅作为 identifier 用
- **SSO / OAuth** (Google / Apple Login) — V3+
- **2FA / TOTP / SMS code** — V3+
- **Magic link 登录** — V3+
- **Billing tier self-upgrade** — V2 商业化

### Admin 既有能力 (已 ship, 本 spec 不重做)

`web/routers/admin.py` 已含完整 user CRUD:
- `POST /api/admin/users` — 创建 user (当前接收 `password` 字段, **本 spec 改为系统生成**)
- `PUT /api/admin/users/{id}` — 改 role/is_active/password/notes (当前 admin 手输 password, **本 spec 保留 + 加一键生成临时密码路径**)
- `DELETE /api/admin/users/{id}` — soft delete (is_active=False)
- `GET /api/admin/users/column-values` — filter popover 用
- `GET /api/admin/user-audit` — UserAuditLog 列表

`web/templates/admin.html` 已含:
- Tab 2 "Users" — Create User inline form (Email + Password + Role + Create) → 改为不要 Password 输入
- Tabulator 用户列表 — 加 "重置密码" 操作列
- Edit User modal — 保留, password 字段语义改"新密码 (留空 = 一键生成临时密码)"
- User audit log section — 自动显示新 action

---

## 3. 5 问 cross-check (北极星决策门禁)

> **答题规则**: 5 问 **是 = 守住, 否 = 偏离**.

| # | 问题 | 答 | 理由 |
|---|---|---|---|
| 1 | 这件事是否服务 Auckland 选房决策闭环? | **是** | admin 拉新用户进 friends-and-family beta → 用 shortlist / candidates 决策状态机. 不引入 generic real estate / 内容平台 |
| 2 | 是否预留商业化能力位? | **是** | `created_via` enum 留 `'self_signup'` 位; `User.role` 3 值 enum 不动; V2 加 public signup 走同 force_password_change=True 路径 (邮件激活时一并强制改) |
| 3 | 是否预留远景跨领域扩展? | **是** | `web/services/email.py` 留 1-line stub (NotImplementedError) — V2 实施时业务代码已对此模块解耦; 临时密码生成器 `generate_temp_password()` V2 可改返 magic link token |
| 4 | **是否避免依赖人工 admin 手工操作?** | **部分** (核心 trade-off) | **守住**: admin SSH 改密码降为 0 (UI 一键搞定); admin 不再需要 SSH 进 VM 跑 `init_db --seed-admin`. **未到完全 self-service**: 用户忘密码仍需 admin 干预 (UI 点 reset, 不是真"用户自助"). V2 加 email forgot-password 才到完全 self-service |
| 5 | 是否专注护城河, 不重做 Trade Me 已有? | **是** | Trade Me 没用户系统, 不偏离差异化 |

**Q4 trade-off 解释**: 本 spec **不达到** 5 问 Q4 "完全 self-service" 标准 — 用户忘密码仍要 admin 干预. 但**确实达到**核心目标 "admin SSH 改密码降为 0" (今天 2026-05-17 痛点). friends-and-family 6 人阶段 admin UI 点击成本 ≈ 0, 收益 = 不引入 email infra 复杂度. V2 商业化到陌生人付费用户时, 再加 email forgot-password 升级为完全 self-service.

**此决策与北极星目标关系**: 落地 §4 #7 的核心收益 (SSH → UI), 战术上推迟 email infra 到 V2 (跟商业化绑同时做更高效).

---

## 4. 数据模型变更

### 4.1 `User` 表加 3 列 (全部 nullable / 有 default, 兼容回填)

| 列 | 类型 | 默认 | 用途 |
|---|---|---|---|
| `force_password_change` | `Boolean nullable, default False` | `False` | True = 用户 login 后必跳 `/change-password`; admin create/reset 临时密码场景设 True |
| `password_changed_at` | `DateTime nullable` | `NULL` | 用户/admin 改密时设 `now()`; 审计 + 未来"密码 N 天未改提醒"用 |
| `created_via` | `String(50) nullable` | `'admin_create'` (新建时); 既有回填 `'admin_seed'` | soft enum (ORM 层不约束 DB level): `'admin_seed'` (init_db CLI) / `'admin_create'` (admin UI) / `'self_signup'` (V2 留位) |

**不加** `email_verification_tokens` 表 — V1 无 email infra.
**不加** `User.email_verified_at` — V1 不发邮件, 没有 verify 概念.
**不加** `User.auth_provider` — 当前只有 password, V2 加 SSO 时再 ALTER COLUMN.

### 4.2 Alembic migration (跟 `init_db.py::_migrate_*` 风格一致)

新加 `_migrate_user_onboarding(engine)` 函数 (跟 `_migrate_add_columns` 一致, 用 `inspect(engine)` 探测列存在 + `ALTER TABLE ADD COLUMN`):

1. 探测 `users.force_password_change` 列? 没有则 `ALTER TABLE users ADD COLUMN force_password_change BOOLEAN DEFAULT 0`
2. 探测 `users.password_changed_at` 列? 没有则 `ALTER TABLE users ADD COLUMN password_changed_at DATETIME`
3. 探测 `users.created_via` 列? 没有则 `ALTER TABLE users ADD COLUMN created_via VARCHAR(50)`
4. 回填: `UPDATE users SET created_via = 'admin_seed' WHERE created_via IS NULL` (既有 user 视为 admin_seed, 因为之前都是 admin SSH 建)
5. 不回填 `force_password_change` / `password_changed_at` (既有 user 不强制改密, 也不知道何时改的)
6. 幂等 — 重跑无副作用 (跟现有 migration 一致)

注册到 `init_db.py::init_db()` 调用链.

**部署**:
- dev: `python -m web.db.init_db` 本地跑 migration
- PROD: deploy.sh 部署后, GitHub Actions 自动跑 `python -m web.db.init_db` (不 SSH)

---

## 5. API contract

### 5.1 改: `POST /api/admin/users` (admin only)

**Request body** (改, 去掉 `password` 字段):
```json
{
  "identifier": "alice@example.com",
  "role": "viewer",
  "display_name": "Alice"   // optional
}
```

**Response 成功** (200 OK):
```json
{
  "id": 42,
  "identifier": "alice@example.com",
  "role": "viewer",
  "temp_password": "Kj7xN2pQmA8r",
  "password_delivery_method": "admin_screenshot_to_user"
}
```

**业务逻辑** (改现有 `create_user`):
1. 校验 identifier 未存在 / role valid / admin count ≤ 3 (现有逻辑保留)
2. **`temp_password = generate_temp_password()`** (新, 取代 body.password)
3. `credential_hash = hash_password(temp_password)`
4. 创建 User: `force_password_change=True`, `created_via='admin_create'`, `password_changed_at=None`
5. `UserAuditLog(action='created', detail=f"role={role}, force_password_change=True")` (现有逻辑保留 + 加 detail)
6. Response 返 `temp_password` (**只这一次, 不再可查**)

**Response 字段 `password_delivery_method`**: V1 写死 `'admin_screenshot_to_user'`; V2 加 email 时可加 `'email'` 值 (forward compat hook).

### 5.2 新加: `POST /api/admin/users/{id}/reset-password` (admin only)

**Request**: 空 body
**Response** (200 OK):
```json
{
  "id": 42,
  "temp_password": "Kj7xN2pQmA8r",
  "password_delivery_method": "admin_screenshot_to_user"
}
```

**业务逻辑**:
1. 查 user by id, 404 if not found
2. `temp_password = generate_temp_password()`
3. `target.credential_hash = hash_password(temp_password)`
4. `target.force_password_change = True`
5. `target.password_changed_at = now()` (admin 改的也算一次)
6. `UserAuditLog(action='password_reset_by_admin', detail=f"reset by admin user_id={actor.id}")`
7. Response 返 `temp_password`

**Why 新增 endpoint 而不是 reuse `PUT /api/admin/users/{id}`**: PUT 现有逻辑 admin 手输 password (不强制 force_password_change=True), 语义不同. 加专门 endpoint 让"一键生成临时密码"路径明确. PUT 路径保留 (admin 想手输密码兼容老习惯).

### 5.3 新加: `GET /change-password` (logged-in only)

渲染 `templates/change_password.html`, form 提交到 `POST /auth/change-password`. 表单字段:
- `new_password` (≥ 6 chars, 跟现有 `/auth/password` 一致 — 临时密码已 12 char, 用户改密下限 6 不算降级)
- `confirm_password` (前端 JS 校验匹配)

**不要求** `current_password` — 用户拿到临时密码不一定记 (admin 截图发的, 用户没主动设).

页面文案: "您的账号设置了临时密码, 请改成自己的密码后继续使用 / Your account has a temporary password. Please set a new password to continue."

### 5.4 新加: `POST /auth/change-password` (logged-in only)

**Request** (form 或 JSON):
```
new_password=NewSecret456
```

**Response 成功**: 302 → `/shortlist`

**Response 失败**: 400 if `new_password < 6 chars`

**业务逻辑**:
1. 设 `user.credential_hash = hash_password(new_password)`
2. 设 `user.force_password_change = False`
3. 设 `user.password_changed_at = now()`
4. `UserAuditLog(target_user_id=user.id, actor_id=user.id, action='password_changed', detail='self-service via /change-password')`
5. **可选**: invalidate 当前 JWT + 发新 JWT (避免 fpc=True 缓存在旧 JWT claim 中 — 见 §9 Q3)
6. 302 → `/shortlist`

**跟现有 `PUT /auth/password` 区别**: 后者要 `current_password` (用户自主改密, 知道当前密码); 本 endpoint 不要 (临时密码场景, 已 login). 两个 endpoint 并存, 不互相替代.

### 5.5 Middleware: 强制 force_password_change 跳转

**位置**: `web/app.py` 加 middleware **或** `web/auth/deps.py::get_current_user` 加检查 (推荐后者, 跟现有 auth 逻辑近).

**逻辑**:
```
if current_user.force_password_change:
    if request.url.path not in WHITELIST:
        return RedirectResponse("/change-password", status_code=302)
```

**白名单 `WHITELIST`**:
- `/change-password` (form 自身)
- `/auth/change-password` (form submit endpoint)
- `/auth/logout` (允许用户登出)
- `/static/*` (CSS / JS 加载)
- `/health` (健康检查不带 auth, 但保险加)

**性能** (见 §9 Q3 完整分析): 推荐 JWT claim 加 `fpc` boolean field, login 时设, change-password 后重发 JWT 清掉. 避免每 request 查 User table.

V1 简单实施: middleware 跟现有 `get_current_user` 一样查一次 User table (sqlalchemy ORM 缓存够用, friends-and-family 6 用户 QPS < 1, 性能不是瓶颈).

---

## 6. 为什么不用 email (替代 v1 spec §6 邮件选型)

v1 spec 花了 100+ 行讨论 Gmail SMTP vs Resend vs Mailgun. 方案 X 全部跳过, 原因:

1. **friends-and-family 6 人**: admin 自己拉, 都是熟人 (WeChat / Teams / 当面), 截图发临时密码足够; 不需要邮件触达
2. **email infra 不是零成本**: Gmail App Password 管理 / 垃圾箱 deliverability 风险 (verify link 易被反钓鱼 filter 拦) / GitHub Secrets 加 4 个变量 / `web/services/email.py` 抽象层 + Jinja2 邮件模板 / send_verification_email + send_reset_email 实现 / unit test mock NullEmailService — V1 引入这些**没有立刻 ROI**
3. **真痛点是"SSH 改密码"不是"用户自助找回"**: 今天 (2026-05-17) 用户痛点 = 忘 PROD admin 密码必须 SSH; admin UI 一键 reset 完全解决, **不需要邮件**
4. **V2 商业化时一并做**: V2 加陌生人付费用户 → 必须 self-service forgot-password → 那时 email infra + signup 一起上, **跟 billing module 同期更高效**, 不是 V1 单独投入

**V1 forward compat hook**: 见 §8, `web/services/email.py` 留 stub, `created_via` 留 `'self_signup'` 值, V2 加 email 时业务代码已解耦.

---

## 7. UI 设计 (文本 mockup)

### 7.1 Admin Tab 2 "Users" 改

**改 (a) — Create User form 去掉 Password 字段**:

```
┌──────────────────────────────────────────────────────────────┐
│  创建新用户 / Create New User                                  │
│                                                              │
│  Email:        [user@example.com_______]                     │
│  Role:         [viewer ▼]                                    │
│  备注 (可选):  [_______________________]                     │
│                                                              │
│  [  创建 (生成临时密码) / Create (generate temp password)  ]   │
└──────────────────────────────────────────────────────────────┘
```

**改 (b) — Create 成功后弹 modal 显示临时密码**:

```
┌──────────────────────────────────────────────────────┐
│  ✓ 用户已创建 / User created                          │
│                                                      │
│  Email:    alice@example.com                         │
│  Role:     viewer                                    │
│                                                      │
│  ⚠ 临时密码 (仅显示一次):                              │
│  ┌─────────────────────────────────────┐  [复制]    │
│  │  Kj7xN2pQmA8r                       │            │
│  └─────────────────────────────────────┘            │
│                                                      │
│  请截图发给用户 (WeChat / Teams / 当面),               │
│  用户首次登录时会被要求改密.                            │
│                                                      │
│                       [我已复制, 关闭]                 │
└──────────────────────────────────────────────────────┘
```

**改 (c) — 用户列表新加"重置密码"操作列**:

Tabulator 列加一个 actions 列, 每行有 "编辑" + "重置密码" + "禁用" 三个按钮.
点 "重置密码" → confirm dialog "确定要为 alice@example.com 重置密码? (生成新临时密码替换旧密码)" → 确认后调 `POST /api/admin/users/{id}/reset-password` → 弹同上 modal 显示新临时密码.

**改 (d) — Edit User modal 保留, password 字段语义不变**:

现有 modal `password` 字段保留 (admin 想手输 password 兼容老习惯), 加注释 "留空 = 不改; 填 = 设 force_password_change=False (admin 知道用户要的密码场景)". 若 admin 想"一键生成临时密码"走"重置密码"按钮路径, 不走 edit modal.

### 7.2 `/change-password` 新页面 (强制改密)

```
┌──────────────────────────────────────────────────────┐
│  APS  [中 / EN]                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│       请修改临时密码 / Change Temporary Password       │
│                                                      │
│   您的账号设置了临时密码, 请改成自己的密码后继续使用.    │
│   Your account has a temporary password. Please      │
│   set a new password to continue.                    │
│                                                      │
│   新密码 / New Password:                              │
│   [________________________]                         │
│   (至少 6 字符 / at least 6 chars)                    │
│                                                      │
│   确认新密码 / Confirm:                               │
│   [________________________]                         │
│                                                      │
│         [  确认修改 / Confirm Change  ]               │
│                                                      │
│  (右上角 Logout 按钮可登出)                            │
└──────────────────────────────────────────────────────┘
```

无 cancel / 跳过按钮 — middleware 强制. 唯一逃逸路径 = `/auth/logout`.

---

## 8. Forward compat hooks (跟北极星 §5 对齐)

| Hook | 防的偏离 | V1 实施 |
|---|---|---|
| `web/services/email.py` 1-line stub | 防 V2 加 email infra 时业务代码已耦合 smtplib; V2 实施时只改这一个文件 | V1 `def send_email(to, subject, body, html_body=None): raise NotImplementedError("Email infra deferred to V2 — see north-star §4 #7+")` |
| `User.created_via` enum 留 `'self_signup'` 值 | 防 V2 加 public signup 时要 ALTER COLUMN | V1 接受 enum 值 (ORM soft, 不约束 DB); 列表/audit 显示文档明示 |
| Admin "新建用户" API response 加 `password_delivery_method` | 防 V2 加 email 时要改 API contract | V1 写死 `'admin_screenshot_to_user'`; V2 可加 `'email'` 值 |
| `generate_temp_password()` 封装在 `web/auth/auth.py` | 防 V2 切 magic link token 时要全文搜替换 | V1 实现 `secrets.token_urlsafe(9)`; V2 可改返 magic link URL |
| `force_password_change` 通用化 (不绑死"临时密码场景") | 防 V2/V3 其他场景需要强制改密 (e.g. admin 怀疑泄漏 / 90 天 rotation) 时要新加字段 | V1 用 boolean; V2 加 `password_must_change_reason` enum 可扩 |
| `/change-password` 跟 `PUT /auth/password` 分离 | 防 V2 self-service forgot-password 时要 fork 逻辑 | 当前两个 endpoint 语义已分; V2 forgot-password reset token 走 `/change-password` 流 |
| JWT claim 加 `fpc` boolean (推荐, 见 §9 Q3) | 防每 request 查 User table 性能问题 | V1 实施需改 `create_jwt` + `get_current_user`; V2 加 SSO 时 JWT 已是扩展点 |

**这些 hook 任何 PR 不能踩坏**. 改动涉及任一 hook 时必须显式说"此改动如何保留 forward compat".

---

## 9. 风险 / Open questions

### Q1: 临时密码截图发 WeChat 是否合规?

**friends-and-family beta**: 可接受. 临时密码 12 char alphanumeric, 用户首次 login 立即被强制改, 窗口短 (~分钟级).
**V2 商业化**: **不可接受** — 陌生付费用户必须走 email forgot-password (V2 加).
**风险缓解**: V1 临时密码 entropy ≥ 12 char (`secrets.token_urlsafe(9)` ≈ 72 bit), 暴力破解不可行短期; admin 截图后**自行删聊天记录** (无法强制).

### Q2: admin 是 single point of failure (admin 不在没人帮 reset)

**V1 现状**: 接受 — friends-and-family 阶段 admin (我) always available; 极端情况 (admin 失联) 仍可 SSH `init_db --seed-admin` 重置 (北极星 §6 历史决策 2026-05-17 已记录这个兜底路径).
**V2 解**: 加 self-service forgot-password (email reset link), admin 不再是 SPOF.

### Q3: middleware 性能 — 每 request 查 User table?

**V1 推荐**: JWT claim 加 `fpc` boolean field (login 时设, change-password 后重发 JWT 清掉). 避免每 request DB hit.
**实施细节**:
- `create_jwt(user_id, role, ...)` → `create_jwt(user_id, role, fpc, ...)` 加参数
- `get_current_user` 读 JWT claim 检 `fpc`, True → return RedirectResponse(`/change-password`)
- `/auth/change-password` 成功后 → 设新 JWT (fpc=False) + 重 set_cookie

**V1 简单实施 (跳过 JWT claim 改)**: middleware 跟现有 `get_current_user` 一样查 User table (ORM 缓存够; friends-and-family QPS < 1). 标 TODO "性能优化 V2 再做".

**推荐**: V1 直接做 JWT claim 方案 (~10 行代码), 一步到位, 同时是 §8 forward compat hook.

### Q4: `/change-password` 跟现有 `PUT /auth/password` 重复?

**答**: 不重复, 语义不同:
- `PUT /auth/password` — 用户自主改密 (知道当前密码), 要 `current_password` 校验
- `POST /auth/change-password` — 临时密码场景 (用户不知 current_password, admin 截图给的), 不要 `current_password` 校验, 但要求 `force_password_change=True` (额外校验, 不接受任意 logged-in user 调)

两个 endpoint 并存.

### Q5: 用户主动想改密码 (不是强制场景) 走哪个 endpoint?

走现有 `PUT /auth/password` (要 `current_password`). 本 spec 新加的 `/change-password` 仅服务"临时密码强制改密"场景.

UI 角度: 个人 profile 页面应有"改密码"按钮 → 走 `PUT /auth/password`; `/change-password` 仅 middleware 强制跳, 不在 nav 暴露.

### Q6: 临时密码长度 / 复杂度?

V1: `secrets.token_urlsafe(9)` ≈ 12 char URL-safe alphanumeric (A-Z, a-z, 0-9, -, _) — ~72 bit entropy.
**为什么不更长**: 截图发用户, 太长用户手输/复制易出错 (12 char 已平衡安全 + 可用).
**为什么不全 alphanumeric (含 `-_`)**: `secrets.token_urlsafe` 含 `-_`, 但少见歧义 (0/O/I/l 用户 confused) 接受.
若用户反馈"密码含奇怪字符不好用", V2 换 `secrets.choice(string.ascii_letters + string.digits)` 纯字母数字.

### Q7: admin 自己 (admin@aps.local / admin) 是否走 force_password_change 流?

**当前 PROD admin** (PR #14 重设的 `admin/changeme`) 是 SSH `init_db --seed-admin` 建的, 走 `created_via='admin_seed'` 路径, **不设** `force_password_change=True` (admin 自己知道密码).
若 admin 想强制自己改密: 走 admin UI "重置密码" 按钮 reset 自己 → 走 force flow. (admin 不能 reset 自己时, 需另一个 admin reset — 北极星 §6 admin 数量限制 ≤ 3 hook 已支持多 admin 互相 reset).

**Migration 回填规则**: 既有所有 user (含 PROD admin) 不设 `force_password_change=True`, 视为已认领账号. 仅新 create 或新 reset 才设.

### Q8: rate limit / abuse 防范?

V1 admin-only endpoint, 已通过 `require_role(["admin"])` 限制. 不需要额外 IP rate limit (admin 自己不会 abuse).
V2 加 public signup / forgot-password 才需要 rate limit middleware.

---

## 10. Acceptance criteria (V1 完成判定)

✅ 必达项:

1. **数据模型 migration ship + 既有 user 回填**:
   - `users` 加 3 列 (`force_password_change` / `password_changed_at` / `created_via`)
   - 既有 user `created_via='admin_seed'` 回填
   - PROD migration 跑过 (走 GitHub Actions deploy, 不 SSH)

2. **Admin UI "新建用户" 流程**:
   - Tab 2 form 去掉 Password 字段, 加 display_name (可选)
   - POST create 成功 → 弹 modal 显示临时密码 + 复制按钮
   - 复制按钮可用 (clipboard API)
   - 新用户 `force_password_change=True` 持久

3. **Admin UI "重置密码" 流程**:
   - 用户列表每行加 "重置密码" 按钮
   - 点击 confirm → 调 reset API → 弹 modal 显示新临时密码
   - 用户 `force_password_change=True` + `password_changed_at=now`

4. **`/change-password` 强制流程**:
   - 用临时密码登录 → 自动跳 `/change-password` (不能去 /shortlist 或其他页)
   - 改密成功 → 跳 /shortlist + `force_password_change=False`
   - JWT claim `fpc` 清掉 (推荐方案)

5. **临时密码生成**:
   - ≥ 12 char (`secrets.token_urlsafe(9)`)
   - URL-safe alphanumeric (含 `-_`)
   - 每次生成不同 (随机性 test ≥ 100 次)

6. **UserAuditLog 写入**:
   - admin create → `action='created', detail='role=X, force_password_change=True'`
   - admin reset → `action='password_reset_by_admin', detail='reset by admin user_id=N'`
   - 用户改密 → `action='password_changed', detail='self-service via /change-password'`

7. **5+ pytest 单测**:
   - `test_admin_create_user_returns_temp_password_and_sets_fpc`
   - `test_admin_reset_password_sets_fpc_and_returns_new_temp_password`
   - `test_user_with_fpc_redirected_to_change_password`
   - `test_change_password_clears_fpc_and_updates_password_changed_at`
   - `test_generate_temp_password_returns_at_least_12_chars`

8. **1-2 e2e Playwright** (在 dev 真跑):
   - **Admin 新建用户 happy path**: admin login → admin Tab 2 → 填 form → 提交 → 看到临时密码 modal → 复制 → 退出 → 用新 user identifier + 临时密码 login → 自动跳 /change-password → 改密 → 跳 /shortlist
   - **用户强制改密 happy path**: 直接走上面流程的后半段 (假设已有 user with fpc=True)

9. **UI snapshot 视觉审视** (前端测试 §4.1 3 步硬门):
   - admin user create modal (含临时密码显示)
   - `/change-password` 页面
   - 5 条异常清单逐条打勾

10. **`web/services/email.py` 1-line stub** + docstring 标 V2 实施

11. **北极星 §6 历史决策回放加 1 行 + §4 #7 状态改 SHIPPED-COMMIT-XXX**

✅ 非强制 (可推 V2):

- Admin 跨 user 批量操作 (e.g. 批量重置 5 user 密码)
- 临时密码过期机制 (e.g. 24h 内不 login 自动失效)
- Email 通知 (admin 创建 user 时 user 自动收邮件) — V2 跟 email infra 同期
- `password_changed_at` 老化提醒 ("90 天未改密码" — V3 商业化才考虑)

---

## 11. 粗估 (AI 速度)

| 阶段 | 内容 | 估时 |
|---|---|---|
| Phase 1 | DB migration `_migrate_user_onboarding` + ORM 模型 3 列 + `web/services/email.py` 1-line stub + `generate_temp_password()` in `web/auth/auth.py` | ~10 min |
| Phase 2 | `web/routers/admin.py::create_user` 改 (去 password 字段, 加 temp_password 生成 + force_password_change=True) + 新加 `POST /api/admin/users/{id}/reset-password` | ~15 min |
| Phase 3 | `web/routers/auth_routes.py` 新加 `GET /change-password` + `POST /auth/change-password` + JWT claim `fpc` 字段 + `get_current_user` middleware 强制跳 | ~20 min |
| Phase 4 | `templates/admin.html` 改 Create User form (去 password) + 加 temp_password modal + 加 reset password 按钮 / `templates/change_password.html` 新建 | ~15 min |
| Phase 5 | 5 unit + 1-2 e2e Playwright 真跑 dev | ~30 min |
| Phase 6 | PROD deploy + smoke (admin 真建一个 user, 走全流程) | ~10 min |

**总估 ~1.5 hr AI 速度** (跟 v1 spec 3-4 hr 对比, 缩短 50%+, 因为没邮件 infra).

**前置依赖**: 无 (无需 Gmail App Password / DNS 改 / 第三方注册).

---

## 12. 关联文档

- 北极星 memory `project_vision_and_north_star.md` §4 #7 + §3 Q4 + §5 (forward compat hook)
- 现有 spec `2026-04-07-aps-cloud-web-design.md` (web platform MVP, 含 auth 模块设计)
- 现有代码: `web/auth/{auth,deps,schemas}.py` + `web/routers/auth_routes.py` + `web/routers/admin.py` (user CRUD 已 ship) + `web/db/models.py::UserAuditLog`
- 跨项目总纲领: `docs/platform/20-identity-and-roles.md` (身份与角色)
- 跨项目总纲领: `docs/platform/40-security-secrets.md` (凭证管理)
- v1 spec (废): 本文件早些时候版本 (544 行 / 全 email infra), git history 可查

---

**END OF DESIGN SPEC**
