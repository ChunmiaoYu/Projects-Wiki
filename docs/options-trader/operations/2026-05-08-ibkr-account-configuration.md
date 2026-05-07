---
title: IBKR 账户配置完整指南
date: 2026-05-08
audience: programmer / 运维
status: living-doc
---

# IBKR 账户配置完整指南

> **本文档面向程序员 + 运维**, 系统性记录 IBKR 账户层级 / 市场数据订阅 / IB Gateway 部署 / 2FA 流程 / 故障排查。
> 经过 2026-04 ~ 2026-05 多轮试错才定型, 写下来防止下次又踩同样的坑。
>
> 敏感信息 (实际 username / 账户号 / IP) 用 `*` 脱敏 — 真值在项目 repo 私有 memory + `.env.{env}` 里。

---

## 1. 账户层级 — 1 个主账户 + 3 个 username

IBKR 设计:
- 1 个**主账户号** (account number, U******) 是计费 + 持仓 + 风控实体
- 主账户下有多个 **username** (login user), 每个 username 是不同 IBKR session 凭证
- 数据订阅按 **per-username** 计费 (不是 per-account)

| Username (脱敏) | 角色 | 模式 | OPRA 订阅持有 |
|---|---|---|---|
| `wa****` | 主用户 (live trading) | live | ✓ 所有数据订阅挂这里 |
| `oat****` | 次用户 (准备替换 wa****) | live | ✓ 单独订一份 (Non-Pro 申请审核中) |
| `vmq****` | paper 用户 (IBKR 自动生成) | paper | ❌ 自身无订阅, 共享 wa**** 的 |

**关键事实**:
- `vmq****` 是 IBKR 在新增 secondary user 后**系统自动生成**的 paper username
- 之前主用户 `wa****` 关联的 paper 登录方式失效 (报错 "当前的真实交易用户有多个与之关联的模拟交易账户")
- 本地测试 IB Gateway 必须用 `vmq****` paper, 不要再用 `wa****` paper 登录
- `wa****` 在 IB Gateway **live mode 能登 ✓**, paper mode 失效 (multi-paper 错)

---

## 2. 4 部署位置的 user 矩阵 — 动态规则

**硬约束**: IBKR 同 username 同时只允许**一个** active session (TWS / IB Gateway 类)。

```
┌──────────────────────────┬──────────────────────────────────────────┐
│ 部署位置                  │ user 选择规则                             │
├──────────────────────────┼──────────────────────────────────────────┤
│ 本地 laptop              │ 永远 vmq**** paper (主开发位)            │
│                          │                                          │
│ VM DEV (云端)            │ ★ 灵活 ★ 跟其他位置看情况:                │
│                          │   - 本地 vmq**** 跑着 → VM DEV 必 live   │
│                          │     (用 wa**** 或 oat****)               │
│                          │   - VM UAT 用 oat**** 跑着 → VM DEV 降级│
│                          │     用 vmq**** paper (本地暂停 paper)    │
│                          │   原因: VM DEV 也是测试位, paper 优先,    │
│                          │   live 仅 UAT 准备阶段使用                │
│                          │                                          │
│ VM UAT (云端)            │ oat**** live (UAT 客户上线前最后一站)    │
│                          │ 待 oat**** Non-Pro review 通过后切换      │
│                          │                                          │
│ VM PROD (云端)           │ 客户的 IBKR 账户 (生产真客户钱)          │
└──────────────────────────┴──────────────────────────────────────────┘
```

**当前临时态 (2026-05-08)**:
- 本地 laptop: `vmq****` paper port 4002 ✓
- VM DEV: `wa****` live port 4001 (临时, 等 oat**** Non-Pro review 通过)
- VM UAT: 尚未部署 (等 oat****)
- VM PROD: 尚未部署 (等客户)

---

## 3. 市场数据订阅

### 3.1 必订清单 (本项目需求)

按 IBKR Portal **Settings → Market Data Subscriptions → Subscriptions 面板齿轮 ⚙ → Level I (NBBO)** 找:

| 订阅 | 覆盖 | NonPro 月费 (大概) |
|---|---|---|
| **OPRA (US Options Exchanges) (NP,L1)** | 美股期权 6 大交易所 (NYSE/CBOE/BOX/Nasdaq/MIAX/MEMX) | 见 Portal (月佣金 ≥ $20 时 Fee Waived) |
| **NASDAQ (Network C/UTP) (NP,L1)** | Nasdaq 上市股票 (AAPL/MSFT/NVDA/TSLA) | 见 Portal |
| **NYSE (Network A/CTA) (NP,L1)** | NYSE 上市普通股 (JPM/BAC/V/JNJ) | 见 Portal |
| **NYSE American + ARCA + BATS + IEX (Network B) (NP,L1)** | NYSE American + ARCA-listed ETF (**SPY/QQQ/IWM**) + IEX | 见 Portal |
| **CBOE Streaming Market Indexes (NP)** | CBOE 自家指数 (**VIX** 全家族 / SKEW) | 见 Portal |
| **ASX Total (NP,L2)** | 澳股 (BHP 测试用) | 见 Portal (AUD 计价) |

> 具体价格不固化在文档, IBKR Portal 实时为准。

### 3.2 IBKR-PRO 是噱头 — 不可替代付费订阅

IBKR 给所有客户**自动免费**送 "**US Real-Time Non Consolidated Streaming Quotes (IBKR-PRO)**", 但实际描述里写的是:

> "A BBO alternative that provides top of book data for equities traded on **five US equity exchanges (BATS, BYX, EDGX, EDGEA, IEX)**"

→ **只覆盖 5 个小众交易所** (BATS / BYX / EDGX / EDGE-A / IEX), **不含主流 NYSE / Nasdaq / NYSE Arca**。所以 AAPL (Nasdaq) / SPY (Arca) / JPM (NYSE) realtime 永远拿不到, 必须订上面 3 个 Network。

### 3.3 Fee Waived 触发

每个订阅旁红字写"A monthly USD X fee will be waived whenever the monthly commissions generated in the account reaches USD Y"。月佣金达到该订阅 waiver 阈值就免费。OPRA / Network 类多数有 waiver, 实际只算一份最高价的 (waiver 不累加)。

### 3.4 Share with Paper 配置

paper 账户 `vmq****` **没自己的订阅**, 必须靠 share 路径:

```
IBKR Portal → Settings → Account Settings → Paper Trading Account
  → Share real-time market data subscriptions with paper trading account? → Yes
  → Select the username whose market data you want shared → wa****
```

底部一条**至关重要的 regulatory note**:

> *Note that due to regulatory laws, only one of these accounts can have an active session at any given time.*

意思: `wa****` (live) 跟 `vmq****` (paper) **同时只能一个** TWS/IB Gateway active session。两边都开 = paper 那边 IBKR 立刻 block 数据 (报 `code=10197 真实账户登录期间无法共享市场数据`)。

### 3.5 怎么知道某 symbol 需要哪个订阅 — Market Data Assistant

IBKR Portal **Support → Quick Support → Market Data Assistant** 工具:

```
Enter symbol: AAPL
Enter exchange: NASDAQ
Choose Type: NonPro
→ Search
```

返回所有能给 AAPL realtime 的订阅 + NonPro 价格。**比 Live Chat 客服快**, 比浏览整个 catalog 快。

---

## 4. IB Gateway 部署

### 4.1 端口 / 账户模式

| 部署 | IB Gateway User | mode | port | 来源 |
|---|---|---|---|---|
| 本地 laptop | `vmq****` | paper | 4002 | 项目 `.env` `IBKR_PORT` / `IBKR_ACCOUNT_MODE` |
| VM DEV | (灵活, §2) | (灵活) | 4001 | VM 上 `/opt/oet/dev/ibgw/env` |
| VM UAT | `oat****` | live | 4001 | VM 上 `/opt/oet/uat/ibgw/env` |
| VM PROD | 客户 | live | 4001 | VM 上 `/opt/oet/prod/ibgw/env` |

> 端口数字 + IBKR_PORT 等以 `.env.{env}` 为唯一真相, 文档不固化。

### 4.2 云端 IB Gateway 容器化

VM 上 IB Gateway 用 **Docker 容器** (`ghcr.io/gnzsnz/ib-gateway:stable`):

```bash
sudo docker run -d --name oet-{env}-ibgw \
  --restart=unless-stopped \
  --network=host \
  --env-file /opt/oet/{env}/ibgw/env \
  ghcr.io/gnzsnz/ib-gateway:stable
```

`--network=host` 必填 (bridge 模式下 docker-proxy 改 source IP, IBC `TrustedIPs=127.0.0.1` 拒绝)。

凭证文件 `/opt/oet/{env}/ibgw/env` (`chmod 600 root:root`) 含:
```
TWS_USERID=...
TWS_PASSWORD=...
TRADING_MODE=paper|live
AUTO_RESTART_TIME=11:59 PM
```

---

## 5. 2FA / IBKey 流程

### 5.1 wa**** LIVE 每次启动 push approve

`wa****` LIVE 模式启动 IB Gateway 时, IBKR 强制 2FA, 用户手机 IBKey app 收 push:

```
Docker 启 → IBC 自动登 → 30-60 sec 后弹 2FA push → 230s 内手机 approve
  → IB Gateway 完整登入 → port 4001 LISTEN
```

**230s timeout 内没接 → IBC 不重试**, 必须重启 docker 触发新 push。

### 5.2 oat**** IBKey device 端激活死循环 (待解)

oat**** 申请 IBKey, 但 device 端 (手机 IBKR Mobile app) 没"Add User"完成:
- Add User 流程要 push 来生 challenge response
- 但 push 要 device 端完成 Add User 才能推
- → 死循环

**解法**: 联系 IBKR Customer Service Live Chat 拿 temporary code, 输入 code 完成 Add User, 之后 push 路径正常。

### 5.3 周日维护窗口

IBKR 每周日晚强制重启所有客户端连接:
- **周六 23:00 ET ~ 周日 03:00 ET** (主维护窗口)
- = NZ **周日下午 16:00-21:00** (DST 时差 16h)
- → 用户清醒时段, 自然介入即可, 不上复杂监控

---

## 6. 常见故障速查

| Error code | 含义 | 真因 / 解法 |
|---|---|---|
| `code=10197` 真实账户登录期间无法共享市场数据 | regulatory share 冲突 | wa**** live 跟 vmq**** paper 不能同时 active. 停一边再测 |
| `code=10089` 请求的市场数据需要额外订阅 | 没买对应 Network 订阅 | 看 §3.1, 把缺的订上, 等 5-15 min 生效 |
| `code=10168` 没有订阅, 延迟数据未启用 | type=1 但没 sub + 没启 delayed | reqMarketDataType(3) 显式启 delayed, 或买实时订阅 |
| `code=326` 客户端 ID 已被使用 | 旧连接没关 | 换 client_id 或杀残留 python |
| `code=200` 未找到该请求的证券定义 | 合约类型错 | 例 VIX 用 STK 而非 IND, SPY 写错 primaryExchange |
| `code=2107` ushmds inactive | 历史数据农场闲置 | 不阻塞实时, 历史调用时 IBKR 自动激活 |

---

## 7. 测试探针 (项目 repo 内)

| 脚本 | 用途 |
|---|---|
| `scripts/_probe_market_data_type.py` | 探 paper 是 realtime 还是 delayed (用 AAPL 5s streaming) |
| `scripts/_probe_wayne1900_live_2026-05-08.py` | 直接连 VM 4001 wa**** LIVE 测自己能不能拿 realtime (绕过 share) |
| `scripts/_smoke_all_collectors_2026-05-07.py` | 7-dim collector 链端到端 smoke (dim1-dim7 全跑) |

---

## 8. 历史教训

| 时间 | 学到 |
|---|---|
| 2026-04-21 | vmq**** 是 IBKR 自动生成 paper user, wa**** paper 已失效 |
| 2026-04-24 | VM IB Gateway 必须用 Docker host networking 模式 |
| 2026-05-07 | wa**** live 在 VM 跑通 + 手机 push approve 路径打通 |
| **2026-05-08** | **关键发现**: IBKR-PRO 只覆盖 5 个小众交易所, 不含 NYSE/Nasdaq/Arca, 必须单独订 Network A/B/C |
| **2026-05-08** | **关键发现**: VIX 是 IND on CBOE, 不是 STK on SMART, collector 代码必须用 IND 合约 + CBOE Streaming Indexes 订阅 |
| **2026-05-08** | **关键发现**: regulatory rule "only one active session" 是 IBKR 后端硬规, 不是软件 bug, 共享 paper 测试时永远要先停 live 端 |

---

## 9. 维护规则 (本文档)

- **真值** (实际 username / 账户号 / IP / 价格) 永远在**项目 repo 内 memory** + `.env.{env}` + IBKR Portal, **不在本文档**
- **本文档负责的**: 规则 / 流程 / 故障应对 / 学到的教训 (audience-agnostic 的 "为什么这样"知识)
- 主要变化触发本文档更新: oat**** 激活路径定型 / 客户账户接入 / 新订阅 SKU 出现 / IBKR Portal UI 改版导致截图陈旧
