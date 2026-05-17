# Spec — 用户 onboarding 自助 (2026-05-17, V1 北极星 #7)

> **状态**: design-draft-pending-review
> **对应北极星**: §4 中间路径 #7 (用户 onboarding 自助 — signup + email verify + 密码自助重置, 减少 admin SSH 干预)
> **核心 5 问**: Q4 (反"依赖人工 admin")
> **依赖**: 现有 `web/auth/` + `User` 模型 + admin user CRUD (已 ship)
> **粗估**: ~3-4 hr AI 速度 (含 Gmail SMTP infra + 2 alembic migration + 4 endpoint + 5 unit + 2 e2e)

---

## 1. 背景 / Motivation

V1 北极星 §4 #7 标记为 **IDEA — V1 必做**, 直接服务 5 问 Q4 ("是否避免依赖人工 admin 手工操作").

**用户痛点 (explicit, 2026-05-17)**: "有时候还是会依赖 SSH" — 今天 (2026-05-17) 用户忘了 PROD admin 密码, 通过 SSH 进 VM 跑 `python -m web.db.init_db --seed-admin admin changeme` 重置, 同时简化为 `admin/changeme` (PR #14). 这是 V1 北极星反偏离明确指出的 anti-pattern:

> §4 Q4 偏离信号: "admin 必须每天 SSH 跑命令 / **密码重置必走 SSH** / 数据回填必须人工触发 / 用户 onboarding 必须 admin 手动建账号"

**当前状态 (anti-pattern)**:
- 新增用户: admin 在 `/admin` UI 手动建 → 用户拿到初始密码 → 必须自己改 (但忘了怎么办?)
- 忘密码: 必须 SSH 进 VM 跑 init_db 重置 (今天就发生了)
- 邮箱地址无校验: 任何字符串都能填 (`bob`, `admin`, `not-an-email`)

**目标**: V1 ship 4 端点 (signup / verify-email / forgot-password / reset-password) + Gmail SMTP 真发邮件, 把"SSH 改密码"从 daily ops 降为罕见运维 (e.g. 整站 disaster recovery 才用).

**反偏离对照 (避免 over-scope)**: SSO / OAuth (Google/Apple login) / 2FA / Magic Link / 客户 self-billing tier upgrade 都是 V2-V3+ 范围, 本 spec 不动. forward compat hook 留 (见 §8).

---

## 2. 目标 + 非目标

### In scope (V1)

| # | 功能 | 端点 | 备注 |
|---|---|---|---|
| (a) | **`/signup` 公共页** | `GET /signup` + `POST /auth/signup` | 任何人能注册, 默认 `role=viewer` + `is_active=False`, 待 email verify |
| (b) | **Email 验证** | `GET /auth/verify-email?token=XXX` | Token 邮件 → 点链激活 (`is_active=True`, `email_verified_at=now`), 跳 `/login` |
| (c) | **忘密码 (forgot password)** | `POST /auth/forgot-password` | 输 identifier → 永返 200 (防 enumeration) → 后台发 reset 邮件 |
| (d) | **重置密码** | `GET /auth/reset-password?token=XXX` (form) + `POST /auth/reset-password` | 验证 token → 设新 `credential_hash` → 跳 `/login` |
| (e) | **Email infra 抽象** | `web/services/email.py` | 提供 `send_email(to, subject, html, text)` 统一接口, V1 实现 = Gmail SMTP, V2+ 可换 Resend/Mailgun |
| (f) | **`/login` 加链接** | template 修改 | 加 "Sign up" / "Forgot password?" 链接, UX 完整 |

### Out of scope (V2+)

- **SSO / OAuth** (Google / Apple Login) — V3+ 远景 (商业化付费 tier 后做)
- **2FA / MFA** (TOTP / SMS code) — V3+ (admin account 真敏感时再加)
- **Magic Link 登录** (passwordless) — V3+
- **客户 self-billing tier upgrade** (viewer → premium 自助付费) — V2 billing 模块
- **批量 invite (CSV upload)** — 当前 friends-and-family 6 人手工够
- **多语言邮件模板 i18n 后端切换** — V1 模板 bilingual (中英文一起渲染), 不按 user.lang 切

### Admin 既有能力 (已 ship, 本 spec 不重做)

- `web/routers/admin.py` 已实现 user CRUD (`POST /api/admin/users` / `PUT /api/admin/users/{id}` / 删除 = `is_active=False`)
- 5 Tab Admin Dashboard 已含用户管理 Tab
- 本 spec 加 1 个新动作: admin 可批量"清理 30 天未验证账号" (P2, 见 §9 open question)

---

## 3. 5 问 cross-check (北极星决策门禁)

> **答题规则**: 5 问 **是 = 守住, 否 = 偏离**. 全 5 是才能进 plan.

| # | 问题 | 答 | 理由 |
|---|---|---|---|
| 1 | 这件事是否服务 Auckland 选房决策闭环? | **是** | 用户能自助 signup → 邀请友人加入 friends-and-family beta → 用 shortlist / candidates 决策状态机. 不引入跨城市 / generic real estate / 内容平台 |
| 2 | 是否预留商业化能力位? | **是** | signup 默认 `role=viewer` (基础免费层); V2 加 `invite_code` 字段决定 `role=premium` (付费 tier 用 billing module 发码). 保留 `User.role` 3 值 enum 不动 |
| 3 | 是否预留远景跨领域扩展? | **是** | `User` 模型加 `auth_provider` enum (默认 `password`, 未来 `google_sso` / `apple_sso` / `magic_link`), email infra 抽象层 (`web/services/email.py`) — 远景 V2 candidate 状态变更邮件通知 / 出价过期提醒复用同一基础设施 |
| 4 | **是否避免依赖人工 admin 手工操作?** | **是 (核心 — 本 spec 就是解 Q4)** | signup 自助 + 密码自助重置 → admin SSH 改密码降为罕见运维; admin 仍保留 manual override 能力 (admin 可强制 reset 任意 user 密码 走 audit log) |
| 5 | 是否专注护城河, 不重做 Trade Me 已有? | **是** | Trade Me 没用户系统 (是房源 marketplace), 这是 APS 独立用户域; 不偏离差异化 |

**5 问全过 ✓**. **此决策与北极星目标关系**: 直接落地 §4 #7 + §3 Q4 守自动化 hook; admin SSH 干预从 daily 降为 disaster recovery only.

---

## 4. 数据模型变更

### 4.1 `User` 表加 3 列

| 列 | 类型 | 默认 | 用途 |
|---|---|---|---|
| `email_verified_at` | `DateTime nullable` | `NULL` | NULL = 未验证, **不允许 login**. 验证成功后 = `now()` |
| `created_via` | `String(50)` | `'admin_seed'` | enum 3 值: `'admin_seed'` (init_db CLI) / `'admin_create'` (admin UI 建) / `'self_signup'` (公共 signup) — 区分账号来源, audit + 未来 billing tier 判断用 |
| `auth_provider` | `String(50)` | `'password'` | enum (扩展位): `'password'` (当前唯一) / `'google_sso'` / `'apple_sso'` / `'magic_link'` — V3+ SSO 接入时用 |

**回填迁移规则** (`init_db.py::_migrate_*` 风格):
- 既有 user (admin / 5 friends-and-family beta) 全部回填: `email_verified_at = created_at` (视为"已验证, 因为之前是 admin 手建"), `created_via = 'admin_seed'`, `auth_provider = 'password'`
- 回填不需要发验证邮件 — 这是历史数据迁移

### 4.2 新表 `email_verification_tokens`

```
email_verification_tokens
├─ id            INTEGER PK autoincrement
├─ user_id       INTEGER FK → users.id (NOT NULL, INDEX)
├─ token         CHAR(64) UNIQUE INDEX   -- secrets.token_hex(32) → 64 chars hex
├─ purpose       VARCHAR(32) NOT NULL    -- 'verify_email' | 'password_reset'
├─ created_at    DATETIME NOT NULL DEFAULT now()
├─ expires_at    DATETIME NOT NULL       -- verify_email = +24h, password_reset = +1h
├─ consumed_at   DATETIME nullable       -- NULL = 未使用; 使用后填 now() (防重放)
└─ ip_address    VARCHAR(45) nullable    -- 申请时 IP (audit, rate limit 用)
```

**索引**:
- `ix_evt_token` UNIQUE on `token` (lookup 主路径)
- `ix_evt_user_id_purpose` on `(user_id, purpose, consumed_at)` (查"该 user 是否有未消费的 reset token", rate limit 用)
- `ix_evt_expires_at` on `expires_at` (cron 清理过期 token 用)

**生命周期**:
- 创建: `POST /auth/signup` (purpose=verify_email) / `POST /auth/forgot-password` (purpose=password_reset)
- 使用: `GET /auth/verify-email?token=XXX` / `POST /auth/reset-password` — 校验 `expires_at > now` + `consumed_at IS NULL` → 业务动作 → 设 `consumed_at=now()`
- 清理: cron daily 删 `expires_at < now - 30 days` 的 (audit 留 30 天足够)

### 4.3 Alembic migration 实施 (跟 `init_db.py::_migrate_*` 风格一致)

新加 `_migrate_user_onboarding(engine)` 函数:

1. 用 `inspect(engine)` 探测 `users` 表是否已有 `email_verified_at` 列, 没有则 `ALTER TABLE users ADD COLUMN email_verified_at DATETIME`
2. 类似加 `created_via VARCHAR(50) DEFAULT 'admin_seed'` + `auth_provider VARCHAR(50) DEFAULT 'password'`
3. 回填: `UPDATE users SET email_verified_at = created_at WHERE email_verified_at IS NULL` (既有 user 视为已验证)
4. 检测 `email_verification_tokens` 表存在? 不存在则 `Base.metadata.create_all` 创建 (SQLAlchemy ORM 自动)
5. 幂等 — 重跑无副作用 (跟现有 `_migrate_add_columns` / `_migrate_drop_candidate_unique` 一致)

**部署**:
- dev: `python -m web.db.init_db` 本地跑 migration
- PROD: deploy.sh 部署后, GitHub Actions 自动跑 `python -m web.db.init_db` (跟现有 reenrich migration 流程一致, 不 SSH)

---

## 5. API contract

### 5.1 `GET /signup` (HTML page)

渲染 `templates/signup.html`, form 提交到 `POST /auth/signup`. 表单字段:
- `identifier` (email format, HTML5 `type="email"` 但后端用 `type="text"` 一致性 — 详 forward compat hook)
- `password` (≥ 8 chars, HTML5 `minlength=8`)
- `display_name` (optional)

### 5.2 `POST /auth/signup`

**Request** (form-encoded, 跟现有 login 一致):
```
identifier=alice@example.com&password=Secret123&display_name=Alice
```

**Response 成功** (201 Created, HTML page):
```html
<h1>验证邮件已发送</h1>
<p>请查收 alice@example.com 邮箱, 24 小时内点击链接激活账号.</p>
<a href="/login">回登录页</a>
```

**Response 失败**:
- `409 Conflict` — identifier 已存在 (返回 signup 页 + error "该 email 已注册, 请登录或重置密码")
- `400 Bad Request` — password < 8 chars / identifier 不符 email format (邮箱格式校验放 backend 用 `email-validator` 库, 见 §6 依赖)
- `429 Too Many Requests` — 同 IP 60s 内 ≥ 3 次 signup 请求 (rate limit, 防 abuse)

**业务逻辑**:
1. 校验 identifier 是 valid email format
2. 校验 identifier 未存在 (含已存在但 `is_active=False` 未验证 — 同样 409, 提示用户重发验证邮件而非重新注册)
3. `hash_password(password)` → 创建 User: `is_active=False`, `email_verified_at=NULL`, `created_via='self_signup'`, `role='viewer'`
4. 生成 token: `secrets.token_hex(32)` → 写 `email_verification_tokens(purpose='verify_email', expires_at=now+24h)`
5. 调 `web.services.email.send_verification_email(to=identifier, token=token, display_name=display_name)`
6. log audit: `UserAuditLog(target_user_id=new_user.id, actor_id=new_user.id, action='self_signup', detail='via /auth/signup')`
7. 返 201 + 渲染 "邮件已发" 页

### 5.3 `GET /auth/verify-email?token=XXX`

**Response 成功**: 302 → `/login?verified=1` (login 页顶部显示 "邮箱已验证, 请登录" banner)

**Response 失败**:
- Token 不存在 / 已 consumed / expired → 渲染 error page "链接已失效, 请重新申请" + 重发链接按钮
- Purpose 不匹配 (不是 `verify_email`) → 同上 error page

**业务逻辑**:
1. 查 token in DB, 校验 `purpose='verify_email'` + `consumed_at IS NULL` + `expires_at > now`
2. 设 `user.is_active=True`, `user.email_verified_at=now()`
3. 设 `token.consumed_at=now()`
4. log audit: `action='email_verified'`
5. 302 跳 `/login?verified=1`

### 5.4 `POST /auth/forgot-password`

**Request** (form 或 JSON):
```
identifier=alice@example.com
```

**Response**: **永远 200** (防 user enumeration — 不告诉对方该 email 是否注册过)
```json
{"ok": true, "message": "如果该 email 已注册, 重置邮件已发出"}
```

**Response 失败仅**:
- `429 Too Many Requests` — 同 identifier 或 IP 60s 内 ≥ 3 次 (rate limit)

**业务逻辑** (后台异步):
1. 查 user by identifier, **不告诉前端是否存在**
2. 若存在 + `is_active=True`: 生成 token `secrets.token_hex(32)`, 写 `email_verification_tokens(purpose='password_reset', expires_at=now+1h)`, 调 `email.send_reset_email(to=identifier, token=token)`
3. 若存在但 `is_active=False` (未验证): 不发 reset, 发"先验证邮箱"提示邮件 (含重发 verify_email token 链接)
4. 若不存在: silent — 不发任何邮件 (但 response 仍 200)
5. log audit: `action='password_reset_requested'` (不管是否成功发出邮件)

### 5.5 `GET /auth/reset-password?token=XXX` (HTML form)

**Response 成功**: 渲染 `templates/reset_password.html`, form 字段:
- `token` (hidden, prefilled from query string)
- `new_password` (≥ 8 chars)
- `confirm_password` (前端 JS 校验匹配)

**Response 失败**: 同 verify-email — error page 提示链接失效

### 5.6 `POST /auth/reset-password`

**Request** (form):
```
token=abc123...&new_password=NewSecret456
```

**Response 成功**: 302 → `/login?reset=1`

**Response 失败**:
- Token invalid / expired / consumed → error page
- `new_password < 8 chars` → 返 form + error

**业务逻辑**:
1. 校验 token (purpose=`password_reset`, 未消费, 未过期)
2. 设 `user.credential_hash = hash_password(new_password)`
3. 设 `token.consumed_at=now()`
4. log audit: `action='password_reset_completed'`, `actor_id=user.id` (self-service)
5. 302 → `/login?reset=1`

---

## 6. 邮件基础设施选型 (关键决策)

> **这是 V1 唯一引入第三方依赖的地方**, 必须想清楚.

### 选项对比

| 维度 | A: Gmail SMTP | B: Resend free | C: Mailgun free |
|---|---|---|---|
| **费用** | 免费 (用户 Gmail App Password) | 3000 邮件/月 free, 之后 $20/月 1万邮件 | 5000 邮件/月 free 前 3 个月, 之后按量付费 |
| **设置成本** | Gmail 账号开 App Password (~5 min) | 注册 Resend + verify domain (~30 min) + DNS 改 | 注册 Mailgun + verify domain + DNS 改 (~30 min) |
| **deliverability** | 一般 (Gmail 可能拦"no-reply" 邮件入垃圾箱) | 好 (transactional infra, SPF/DKIM/DMARC 自动) | 好 (transactional infra) |
| **依赖** | 0 (Python stdlib `smtplib` + `email.mime`) | `resend` Python SDK (新增 1 包) | `requests` (已有) 直调 API |
| **限额** | Gmail 500 邮件/day (够 friends-and-family 6 人 + signup 浪潮) | 3000 邮件/月 (够 V1+早期 V2) | 5000 邮件/月 (V1+V2 都够) |
| **用户预算 §13 兼容** | ✓ 0 付费 | ⚠️ free tier, 但有付费产品 — 仍可考虑 (用户预算反对付费 — free tier OK) | ⚠️ 同 B, 且 free 期短 |
| **forward compat** | 用 `email.py` 抽象, V2 可切 B/C 不改业务 | 同 | 同 |
| **隐患** | Gmail 自动拦 transactional 邮件 (尤其 verify link); 用户邮箱配额耗光; 用 Gmail = 个人邮箱跟生产邮件混 | Resend 公司倒闭风险 (创业公司, 不是 SendGrid 大厂) | Mailgun 涨价历史 (2-3 年前关 free tier 引争议) |

### 推荐: **选项 A (Gmail SMTP) for V1**

**理由**:
1. **零依赖 + 零费用** — 完全符合用户预算 §13 (Oracle Always Free + 不付费组件)
2. **friends-and-family 6 人 + 日均 < 50 邮件**, Gmail 500/day 远远够用
3. **V1 forward compat hook 已留** — `web/services/email.py` 抽象接口, V2 用户感觉"垃圾箱拦得多" 时切 Resend (改一个文件, 不改业务)
4. **生产邮件用 admin alias** (`aps-noreply@yuanshan1900.com` Gmail alias), 不污染用户个人邮箱

### 实现细节 (V1 Gmail SMTP)

```
web/services/email.py
├─ class EmailService (抽象基类, 定义 send_email / send_verification_email / send_reset_email)
├─ class GmailSMTPEmailService (V1 实现, 用 smtplib + ssl + email.mime.multipart)
├─ class NullEmailService (单测用, 不发邮件只 log)
└─ get_email_service() → singleton, 按 settings.email_provider 工厂返
```

**新增 settings (`.env.example` 加)**:
```
# Email infrastructure (V1: gmail_smtp)
EMAIL_PROVIDER=gmail_smtp           # gmail_smtp | resend | mailgun | null
EMAIL_FROM_ADDRESS=aps-noreply@yuanshan1900.com
EMAIL_FROM_NAME=APS Auckland Property Shortlist
GMAIL_SMTP_USER=<gmail_address>
GMAIL_SMTP_PASSWORD=<gmail_app_password>   # 16-char App Password, 不是 Gmail 登录密码
GMAIL_SMTP_HOST=smtp.gmail.com
GMAIL_SMTP_PORT=587
PUBLIC_BASE_URL=https://property.yuanshan1900.com   # token 链接用
```

**GitHub Secrets 同步** (跟 `APS_VM_IP` 一样): `EMAIL_PROVIDER` / `GMAIL_SMTP_USER` / `GMAIL_SMTP_PASSWORD` / `PUBLIC_BASE_URL` 加入 GitHub Secrets, deploy.sh 注入 PROD `.env`.

### 邮件模板 (V1, bilingual 中英文一起)

`web/templates/emails/verification_email.html`:
```
Subject: APS — 请验证您的邮箱 / Please verify your email

[中文]
您好 {display_name or '用户'},
感谢注册 Auckland Property Shortlist. 请点击以下链接验证邮箱 (24 小时内有效):
{PUBLIC_BASE_URL}/auth/verify-email?token={token}

[English]
Hello {display_name or 'there'},
Thanks for signing up to Auckland Property Shortlist. Click the link below to verify your email (valid for 24 hours):
{PUBLIC_BASE_URL}/auth/verify-email?token={token}

---
此邮件由系统自动发送, 请勿回复. / This is an automated email, please do not reply.
```

`web/templates/emails/password_reset_email.html` 类似 (1 小时有效, /auth/reset-password?token=).

---

## 7. UI 设计 (文本 mockup)

### 7.1 `/signup` 新页面

```
┌──────────────────────────────────────────┐
│  APS  [中 / EN]                          │
├──────────────────────────────────────────┤
│                                          │
│       创建账号 / Sign Up                  │
│                                          │
│   Email:        [_________________]      │
│   Password:     [_________________]      │
│                 (至少 8 字符)             │
│   昵称 (可选): [_________________]       │
│                                          │
│         [  注 册 / Sign Up  ]            │
│                                          │
│   已有账号? [登录 / Login]                │
│                                          │
└──────────────────────────────────────────┘
```

### 7.2 `/login` 修改 (加 2 链接)

```
┌──────────────────────────────────────────┐
│  APS  [中 / EN]                          │
├──────────────────────────────────────────┤
│                                          │
│        登录 / Login                       │
│                                          │
│   Email/User:   [_________________]      │
│   Password:     [_________________]      │
│                                          │
│         [   登 录 / Login   ]            │
│                                          │
│   还没账号? [注册 / Sign Up]              │
│   忘记密码? [找回密码 / Forgot Password]  │
│                                          │
└──────────────────────────────────────────┘
```

### 7.3 `/auth/reset-password?token=XXX` 新页面

```
┌──────────────────────────────────────────┐
│  APS  [中 / EN]                          │
├──────────────────────────────────────────┤
│                                          │
│      重置密码 / Reset Password            │
│                                          │
│   新密码:        [_________________]     │
│                 (至少 8 字符)             │
│   确认新密码:    [_________________]     │
│                                          │
│         [  重 置 / Reset  ]              │
│                                          │
└──────────────────────────────────────────┘
```

### 7.4 邮件已发出提示页 (signup 后 / forgot-password 后)

```
┌──────────────────────────────────────────┐
│  APS                                     │
├──────────────────────────────────────────┤
│                                          │
│   ✉️  验证邮件已发送 / Email Sent          │
│                                          │
│   请查收 alice@example.com 邮箱,           │
│   24 小时内点击链接激活账号.                │
│                                          │
│   没收到? 检查垃圾箱, 或 [重新发送]        │
│                                          │
│   [回登录页 / Back to Login]              │
│                                          │
└──────────────────────────────────────────┘
```

---

## 8. Forward compat hooks (跟北极星 §5 对齐)

| Hook | 防的偏离 | 实施 |
|---|---|---|
| `web/services/email.py` 抽象接口 (`EmailService` 基类 + 工厂) | 防 V2 切 Resend/Mailgun 要全文搜替换 `smtplib`; 业务代码只调 `get_email_service().send_verification_email()` | V1 实现 GmailSMTPEmailService + NullEmailService 单测用 |
| `User.auth_provider` enum (默认 `'password'`) | 防 V3+ SSO (Google/Apple/Magic Link) 接入要重写 User 模型 | 加列 + alembic 回填 `'password'` |
| `User.created_via` enum (`admin_seed`/`admin_create`/`self_signup`) | 防未来 billing tier 自动 upgrade 时要全文搜 / 防审计需求新增 | 加列, audit 用 |
| `email_verification_tokens.purpose` enum (`verify_email`/`password_reset`) | 防 V2 加 `email_change_confirm` / `invite_accept` 等新 purpose 要新建表 | 复用同表只加 enum 值 |
| `POST /auth/signup` 加 `invite_code` 字段 (V1 可空, 不强制) | 防 V2 billing tier "premium 用 invite code 注册" 要重写 signup | V1 schema 接收 `invite_code` 字段但忽略 (TODO V2 实施); 文档明示 |
| Rate limit middleware (在 `web/app.py` 加, signup/forgot-password endpoint 60s ≥ 3 次拦) | 防 signup 风暴 / token enumeration | V1 实现简单 in-memory dict + IP / identifier 双键; V2 加 `slowapi` 库或 Redis |
| Token 一律 `secrets.token_hex(32)` (64 hex chars, 256 bit entropy) | 防猜测攻击 | 不接受 user-supplied token / 不接受 base64url 等其他 encoding |
| Email content 走 Jinja2 模板 (`templates/emails/`) | 防硬编码 HTML 字符串拼接难维护 / 防 XSS (Jinja2 autoescape) | 跟主站模板共用 Jinja2Environment |

**这些 hook 任何 PR 不能踩坏**. 改动涉及任一 hook 时必须显式说"此改动如何保留 forward compat".

---

## 9. 风险 / Open questions

### Q1 (用户决): **Email provider 选 A / B / C?**

- **A (Gmail SMTP)**: 推荐, 零依赖零费用; 风险 = Gmail 可能拦垃圾箱 (尤其 verify link 可能被反钓鱼 filter 误判)
- **B (Resend free 3000/月)**: 真正 transactional infra, deliverability 最好; 风险 = 创业公司, 需 verify domain (~30 min DNS 改)
- **C (Mailgun free)**: free tier 短 (3 个月), 之后按量付费, 不推荐

→ **默认 A**, 用户回 "选 B" 或 "选 C" 才改 spec.

### Q2: signup 默认 role 是 `viewer` 还是 `admin must approve` 才激活?

**推荐**: `role=viewer` + `is_active=True after email verify` (邮箱已验证就能 login, 默认 viewer 只能看 shortlist + candidates, 不能 Excel 导出 — 已有 role gate).

**Why not admin approve**: friends-and-family beta 6 人手工建够, 但 V2 商业化时 viewer 是免费层, 必须 frictionless. **admin 仍保留 `PUT /api/admin/users/{id}` 把 viewer 升 premium / 禁用**.

替代方案 (open): signup 默认 `role=viewer` + `is_active=False` 即使邮箱验证也要 admin manual approve — 更安全但 friction 高. V1 不做.

### Q3: token 过期时间合理?

- `verify_email`: **24h** (允许用户睡一觉再点)
- `password_reset`: **1h** (短窗口防泄漏)

→ 都进 settings, 改 .env 可调.

### Q4: 注册风暴防范多严?

V1 简单 in-memory rate limit: 同 IP 60s 内 ≥ 3 次 signup/forgot-password → 429. 不阻拦真实使用 (一个用户 60s 内不会按 3 次).

V2 升级到 `slowapi` 或 Redis 后端持久化 rate limit (跨 worker 进程).

### Q5: admin 是否能批量清理 30 天未验证账号?

**推荐**: V1 加 cron daily 跑 `python -m web.maint.cleanup_unverified_users --older-than-days 30 --dry-run` (默认 dry-run, 改 `--apply` 真删). admin 可在 `/admin` UI 看到"待清理"列表 + 手动 trigger.

→ 是 P2, V1 不强求, 但 schema 设计要支持 (查 `WHERE email_verified_at IS NULL AND created_at < now - 30 days`).

### Q6: signup 是否开放 (任何人都能注册) 还是要 invite code?

V1: **开放 + admin 手动 review** (signup 后 admin 看 `/admin` 用户 Tab 新用户 → 决定是否禁用). friends-and-family beta 阶段 admin 一周看一次没问题.

V2 加 `invite_code` 字段 (forward compat hook 已留, 见 §8) — billing tier signup 用 invite code 决定 role.

### Q7: identifier 仍允许非 email 吗 (跟 login `type="text"` 一致)?

**signup 强制 email format** (用 `email-validator` 库校验), 因为要发邮件验证. **login 仍 `type="text"`** (forward compat 给 SSO username 不带 @ 用).

不一致是有意的: signup 必 email (要发邮件), login 接受非 email (未来 SSO).

### Q8: Gmail App Password 怎么管理 + rotation?

V1: 用户 Gmail 账号开 App Password (Google Account → Security → 2-Step Verification → App passwords), 16 字符贴 GitHub Secret `GMAIL_SMTP_PASSWORD`.

Rotation: 跟 `APS_VM_IP` 一样 — 改 Secret + memory 同步 (北极星 §6 历史决策 2026-05-17 教训).

---

## 10. Acceptance criteria (V1 完成判定)

✅ 必达项:

1. **4 个新端点全 ship + tests pass**:
   - `POST /auth/signup` (含 409 / 400 / 429)
   - `GET /auth/verify-email?token=XXX` (含 expired / consumed error)
   - `POST /auth/forgot-password` (永 200, rate limit 429)
   - `GET /auth/reset-password` + `POST /auth/reset-password`

2. **数据模型 migration ship + 既有 user 回填**:
   - `users` 加 3 列 (`email_verified_at` / `created_via` / `auth_provider`)
   - 新表 `email_verification_tokens` 创建
   - 既有 user `email_verified_at = created_at` 回填
   - PROD migration 跑过 (走 GitHub Actions deploy, 不 SSH)

3. **Gmail SMTP 真发邮件 verify pass**:
   - `EMAIL_PROVIDER=gmail_smtp` PROD 配置完
   - 测试: 用 dev 邮箱 signup → 真收到 verify 邮件 → 点链接成功激活 → login 成功
   - 测试: dev 邮箱 forgot-password → 真收到 reset 邮件 → 点链接设新密码 → login 成功

4. **5 单测 (pytest)**:
   - `test_signup_creates_user_and_sends_email` (用 NullEmailService mock)
   - `test_signup_duplicate_email_returns_409`
   - `test_verify_email_activates_user_and_consumes_token`
   - `test_verify_email_expired_token_returns_error`
   - `test_reset_password_updates_credential_hash_and_consumes_token`

5. **2 e2e Playwright** (在 dev 环境真跑):
   - **Signup happy path**: 打开 `/signup` → 填 form → submit → 收到 "邮件已发" 页 → 看 mock email log → 取 verify link → 跳 `/login?verified=1` → 用新账号 login 成功
   - **Forgot password happy path**: 打开 `/login` → 点 "Forgot password?" → 填 email → submit → 收到 "邮件已发" 页 → 看 mock email log → 取 reset link → 设新密码 → 用新密码 login 成功

6. **`/login` 模板加 2 链接** (Sign up / Forgot password)

7. **UI screenshot 视觉审视** (前端测试 §4.1 3 步硬门):
   - `/signup` page snapshot
   - `/auth/reset-password` page snapshot
   - 5 条异常清单逐条打勾

8. **北极星 §6 历史决策回放加 1 行 + §4 #7 状态改 SHIPPED-COMMIT-XXX**

✅ 非强制 (可推 V2):

- Admin UI 加"未验证用户"过滤 + 手动重发 verify 邮件按钮
- Cleanup unverified users cron (Q5)
- Bilingual email 模板按 user.lang 切 (V1 中英文一起渲染)

---

## 11. 粗估 (AI 速度)

| 阶段 | 内容 | 估时 |
|---|---|---|
| Phase 1 | `web/services/email.py` (抽象 + GmailSMTPService + NullService) + Jinja2 邮件模板 2 个 | ~30 min |
| Phase 2 | DB migration (`_migrate_user_onboarding`) + `email_verification_tokens` 模型 + unit test 模型 | ~30 min |
| Phase 3 | 4 endpoints + Pydantic schemas + rate limit middleware + audit log | ~45-60 min |
| Phase 4 | Templates (`signup.html` / `reset_password.html` / `email_sent.html` / login.html 改) + i18n | ~30 min |
| Phase 5 | 5 unit + 2 e2e Playwright 真跑 dev + dev 邮箱真测 Gmail SMTP | ~60-90 min |
| Phase 6 | `.env.example` + GitHub Secrets + PROD deploy + smoke test | ~20 min |

**总估 ~3-4 hr AI 速度** (假设 Gmail App Password ~5 min 开完, 无 DNS 改).

**前置依赖**:
- 用户 Gmail 账号开 App Password (~5 min, 文档列步骤)
- 用户决定 §9 Q1 (email provider A/B/C)

---

## 12. 关联文档

- 北极星 memory `project_vision_and_north_star.md` §4 #7 + §3 Q4 + §5 (forward compat hook)
- 现有 spec `2026-04-07-aps-cloud-web-design.md` (web platform MVP, 含 auth 模块设计)
- 现有代码: `web/auth/{auth,deps,schemas}.py` + `web/routers/auth_routes.py` + `web/routers/admin.py` (user CRUD)
- 跨项目总纲领: `docs/platform/20-identity-and-roles.md` (身份与角色)
- 跨项目总纲领: `docs/platform/40-security-secrets.md` (凭证管理)
- 用户预算 memory `user_budget.md` (反对付费组件 — Gmail SMTP / Resend free tier OK)

---

**END OF DESIGN SPEC**
