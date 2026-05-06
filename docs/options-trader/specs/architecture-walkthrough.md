# ⭐ 架构通俗讲解 — 一个场景跑通整个系统

> **读者**: 不懂软件架构的客户 / 业务方 / 偶尔回看不熟代码的工程师。
>
> **目的**: 用一个"AAPL 突破 280 时买 100 手 call"的真实场景, 把任务队列 / 工人 / 4 个基础设施抽象 / 数据库表全跑一遍, 不需要看代码就能理解系统怎么工作。
>
> 时间锚: 2026-05-06。配套 [⭐ 北极星 §1](north-star-v1-target.md) 看更佳。

---

## <span class="h-num">1.</span> 先理解 "任务队列" — 餐厅厨房挂单板

整个系统的核心是一块**任务队列**。理解它, 后面所有事情都通了。

**类比**: 餐厅厨房的**挂单板**。

```
顾客点单 → 服务员写订单 → 挂到厨房挂单板上
                                ↓
              厨师从挂单板取下订单 → 做菜 → 上菜
```

挂单板的价值:
| # | 价值 | 没挂单板会怎样 |
|---|---|---|
| 1 | **异步** — 服务员写完单就走, 不用等厨师做完 | 服务员要站在厨房等菜出锅才能接下一桌, 一桌卡死全瘫 |
| 2 | **解耦** — 服务员不知道哪个厨师做, 厨师不知道哪个服务员点 | 每点一道菜要喊"老王你来做", 老王休息了就不知道喊谁 |
| 3 | **持久化** — 单子写下来挂着, 厨房着火重启后没做的单还在 | 厨师脑子里记的单, 一晕忘光 |
| 4 | **限流** — 挂单板满了, 服务员不接新单 | 厨房爆单做不完, 客户等 2 小时还在做 |
| 5 | **审计** — 每张单背面写时间, 月底翻查 | 老板不知道哪个时段最忙、哪道菜出错率高 |
| 6 | **重试** — 做坏的单可以重做 | 客户投诉只能凭印象 |
| 7 | **优先级** — VIP 单挂红色, 厨师优先做 | 紧急客户要么被忽略要么要服务员去插话 |

**软件里的任务队列 = 这块挂单板**, 写到数据库表 `workflow_tasks` 里, 多个工人进程都来取任务处理。

**没有任务队列的话**: 就是"服务员叫厨师 hey 来做这单"直接调用, 服务员卡死等厨师做完才能接下一桌, 任何一环挂全餐厅瘫。这就是为什么需要队列。

---

## <span class="h-num">2.</span> 系统真实只有 3 类东西

直觉上你可能以为有 "时间工人 / 条件工人 / 信息工人" 3 个独立的人在跑。**实际不是**。

**真实只有 3 类东西**:

| # | 是什么 | 实际身份 |
|---|---|---|
| 1 | **APScheduler** (排程器) | 时间排程, 唯一会主动产生任务的 |
| 2 | **8 个 handler 函数** | 取任务 → 处理 → 可能塞新任务 |
| 3 | **4 个 <span class="term-service">Pool/Client</span>** (基础设施) | 屏蔽 <span class="term-service">IBKR</span> 细节, 业务层不感知连接 |

**"工人" 这个词指的是 handler 函数的逻辑分组**, 不是独立进程:
- "<span class="term-worker">时间工人</span>" = APScheduler 排程逻辑
- "<span class="term-worker">条件工人</span>" = MarketDataPool 的 tick callback 里的条件计算逻辑
- "<span class="term-worker">信息工人</span>" = `CONDITION_MET` / `AGENT2_REVIEW_TICK` handler 里拉 bundle + 喂 LLM 的代码

下面跟着场景看, 每个名词什么时候出场。

---

## <span class="h-num">3.</span> 4 个基础设施抽象 (Pool/Client)

业务代码**永远不直接调** <span class="term-service">IBKR</span> API。所有 <span class="term-service">IBKR</span> 操作必走这 4 个抽象之一:

| Pool/Client | 职责 | client_id | 模式 |
|---|---|---|---|
| **<span class="term-service">MarketDataPool</span>** | 持续订阅市场数据 (实时 tick / 期权报价), 引用计数去重 | 10-19 | Push (IBKR 推 tick) + Pull (cache 拉最新) |
| **<span class="term-service">QueryClient</span>** | 一次性 query (历史 K 线 / 期权链 / contract details) | 20 | Pull (问一次答一次) |
| **<span class="term-service">OrderClient</span>** | 下单 / 撤单 / 修改, 独占防订单状态混乱 | 30 | Push (IBKR 推 orderStatus) + Pull (我们发 placeOrder) |
| **<span class="term-service">AccountSnapshotClient</span>** | 账户余额 / 持仓 / 订单状态, cache TTL 5 秒 | 40 | Pull (按需拉, cache 去重) |

每个 Pool/Client 一个长连接 (long-lived TCP socket), 永远保持连接。

---

## <span class="h-num">4.</span> 完整场景 — AAPL 突破 280 买 100 手 call

跟着时间线走, 看每一步系统在干什么。

### Step 0 — 客户提交意图 (任何时间)

```
客户输入框: "AAPL 突破 280 时买入 100 手 call"
    ↓
Agent 1 LLM 解析  →  创建 opp #5 in DB:
                       symbol = AAPL
                       trigger = PRICE_BREACH (>= 280)
                       direction = BULLISH
                       raw_input_text = 客户原话整段
                       status = ACTIVE_MONITORING
                       effective_until = 30 天后
```

opp #5 静静躺在数据库里, 等待时间工人盘前唤醒系统去监控它。

---

### Step 1 — 09:00 ET (盘前 30 分钟): SYSTEM_WAKE_UP

美股 regular session 是 09:30-16:00 ET, 所以盘前 30 分钟 = **09:00 ET**。

```
09:00 ET  时间工人 (APScheduler) 排程到点:
    ↓
往挂单板 (workflow_tasks 表) 塞一张订单:
    event_type = SYSTEM_WAKE_UP
    payload = {}
    status = PENDING
```

**SYSTEM_WAKE_UP 的 handler 干什么** (这就是字面意义的"唤醒"):

```
SYSTEM_WAKE_UP handler 取出任务:
  1. MarketDataPool.connect()         → client_id=10, 长连建上
  2. QueryClient.connect()            → client_id=20
  3. OrderClient.connect()            → client_id=30
  4. AccountSnapshotClient.connect()  → client_id=40

  5. 扫 DB 所有 status = ACTIVE_MONITORING 的 opp:
     - opp #5 监控 AAPL → MarketDataPool.subscribe(AAPL,
                                  subscriber=opp_5,
                                  on_tick=check_opp5_condition)
     - opp #6 监控 TSLA → MarketDataPool.subscribe(TSLA,
                                  subscriber=opp_6,
                                  on_tick=check_opp6_condition)
     - ...
  6. 标记任务 status = DONE
```

**"唤醒" = 4 个长连接建上 + 所有该订阅的市场数据订阅上**。

> **常见误解**: "时间工人要喊条件工人来工作呀, 为什么没看到这个任务？"
>
> **答**: 因为**没有"条件工人进程"**! 上面 step 5 注册的 callback `check_opp5_condition` 就是条件计算逻辑, 只要 MarketDataPool 收到 AAPL tick 就自动调它。"<span class="term-worker">条件工人</span>" 这个词指的就是这堆 callback, **不需要单独唤醒**。

---

### Step 2 — 09:30 ET (盘开): MarketDataPool 收 tick

```
09:30:00 ET  IBKR 推 AAPL tick: $278.50
    ↓
MarketDataPool 内部 on_tick 函数被 IBKR API 自动调用 (这是被动响应, 不是任务)
    ↓
on_tick 遍历 callbacks[AAPL] = [check_opp5_condition, ...]
    ↓
调 check_opp5_condition(tick):
    AAPL >= 280 ? 278.50 < 280, 不触发, 啥也不干

09:30:01 ET  下一个 tick: $278.55  → 仍不触发
09:30:02 ET  下一个 tick: $278.40  → 仍不触发
...持续每秒数十个 tick...

09:45:32 ET  tick: $280.10
    ↓
check_opp5_condition(tick) → 280.10 >= 280, 条件触发!
    ↓
往挂单板塞订单:
    event_type = CONDITION_MET
    payload = {opp_id: 5, trigger_price: 280.10, trigger_at: "09:45:32 ET"}
    status = PENDING
    ↓
callback 函数返回 (不等下游 handler 做完, 立即接下一个 tick)
```

> **常见误解**: "tick callback / newsTicker callback 是不是任务？"
>
> **答**: **不是任务, 是事件回调函数**。<span class="term-service">IBKR</span> 推数据来时, MarketDataPool 内部代码会被自动触发。**callback 内部如果发现要做后续大动作, 才塞任务到挂单板**。一个是"被推来" (callback), 一个是"主动写到挂单板" (任务)。

---

### Step 3 — Agent 2 入场决策 (CONDITION_MET handler)

```
CONDITION_MET handler 取出任务 (payload = {opp_id: 5, trigger_price: 280.10}):

  1. 拉 10 维 bundle (拉到的数据后面喂给 LLM):
     ┌─ MarketDataPool.get_latest(AAPL)              → spot = $280.10 (从 cache 拉)
     ├─ MarketDataPool.get_latest_iv(AAPL)           → IV = 35%
     ├─ QueryClient.req_option_chain(AAPL,           → AAPL 285C / 290C 各到期日报价
     │       expiry="2026-05-15")
     ├─ AccountSnapshotClient.get_balance()          → 账户可用资金 (cache TTL 5s)
     ├─ AccountSnapshotClient.get_positions()        → 当前所有持仓
     ├─ DB: opp #5 的 raw_input_text 整段
     ├─ 持仓盈亏维度 = null (还没入场)
     └─ 上次 summary 维度 = null (entry 时无)

  2. 喂 Agent 2 LLM (DeepSeek V4-Flash) → ~10 秒
     LLM 输出: "策略 = LONG_CALL, 行权 285, 到期 2026-05-15, 100 手"

  3. 风险门验证:
     - Layer 1 schema enum: LONG_CALL ∈ 12 种白名单 ✓
     - Layer 2 mapping: LONG_CALL → BULLISH, opp.direction = BULLISH ✓

  4. 往挂单板塞订单:
     event_type = EXECUTE_DECISION
     payload = {opp_id: 5,
                action: "ENTRY",
                strategy: "LONG_CALL",
                legs: [{contract: "AAPL_285C_2026-05-15", qty: 100, side: "BUY"}]}

  5. 写 agent2_decisions 表 (审计)

  6. 标 CONDITION_MET 任务 status = DONE
```

**这就是"<span class="term-worker">信息工人</span>" 逻辑**: 拉 bundle + 喂 LLM + 风险门。**没有信息工人进程**, 这个逻辑就是 `CONDITION_MET` handler 的代码。

---

### Step 4 — 下单, order 在这一步创建 (EXECUTE_DECISION handler)

```
EXECUTE_DECISION handler 取出任务:

  1. OrderClient.place_split_order(
       contract = AAPL_285C_2026-05-15,
       total_qty = 100,
       batch_size = 10,           ← 拆 10 批
       interval_sec = 8,          ← 批间隔 8 秒
       spread_ratio = "patient"   ← LMT mid + 0.2*(ask-mid)
     )

  2. 第 1 批 10 手挂出去 → IBKR 异步推回 orderStatus:
     ┌─ orderStatus = SUBMITTED  → OrderClient 写 orders 表 (order_id=42 创建!)
     └─ orderStatus = FILLED     → 写 fills 表

  3. 第 2 批 10 手 ... 第 10 批 10 手 ...

  4. 全部成交 → OrderClient 内部 orderStatus callback 累计 100 手 FILLED:
     ↓
     往挂单板塞:
        event_type = ENTRY_FILLED
        payload = {opp_id: 5, order_id: 42,
                   total_filled: 100, avg_price: $5.20, total_cost: $52,000}
```

> **关键: order 是 opp 的子产物**。1 个 opp 全程会有 N 个 order (entry 单 + 部分平仓 + 全平 + 撤单重挂等), order_id 会变, opp_id 不变。所以任务 payload 用 `opp_id`, 不用 `order_id`。

---

### Step 5 — 持仓建立 + 启动 5 min review (ENTRY_FILLED handler)

```
ENTRY_FILLED handler 取出任务:

  1. AccountSnapshotClient.invalidate_cache()
     ↑ 强制下次重拉 (持仓刚变, cache 已 stale)

  2. 写 positions 表: opp_id=5, qty=100, avg_cost=$5.20

  3. 改 opp #5 status = HOLDING

  4. 切换 MarketDataPool 订阅模式 (监控阶段 → 持仓阶段):
     - MarketDataPool.unsubscribe_callback(AAPL, subscriber=opp_5)
       ↑ 不再需要条件 callback (条件已触发)
     - MarketDataPool.subscribe(AAPL_285C_2026-05-15, subscriber=opp_5, on_tick=None)
       ↑ 期权报价订阅 (review 时拉 cache 用)
     - AAPL 现货订阅保留 (review 也要 spot 价), 但不再带 callback

  5. INSERT 进 active_reviews 表:
     opp_id=5, next_review_due_at=now()+5min, review_interval_sec=300
```

---

### Step 6 — 5 min review 循环 (AGENT2_REVIEW_TICK handler)

**循环是谁控制的**? 时间工人 (APScheduler) 每 10 秒扫一次 `active_reviews` 表:

```
时间工人每 10 秒 tick 一次:
  扫 active_reviews where next_review_due_at <= now()
  对每行 opp 塞:
     event_type = AGENT2_REVIEW_TICK
     payload = {opp_id, scheduled_at}
  立即 UPDATE next_review_due_at = now() + review_interval_sec
  (handler 完成后再实际执行下一次, 中间 handler 失败也能往前推)
```

**第一次 review** (10:00 ET, 入场后 5 min):

```
AGENT2_REVIEW_TICK(opp_id=5) handler 取出任务:

  1. 拉 10 维 bundle (这次完整 10 维, 含持仓盈亏 + 上次 summary):
     ┌─ MarketDataPool.get_latest(AAPL)             → spot 现 $282.50
     ├─ MarketDataPool.get_latest(AAPL_285C_2026-05-15) → 期权 mid 现 $5.80
     ├─ AccountSnapshotClient.get_positions()       → 持仓 P&L = +$6,000
     │                                                  (5.80-5.20)*100*100=6000
     ├─ DB: 上次 summary 从 agent2_decisions 表拉 → null (这是第一次)
     └─ ...其他 5 维

  2. 喂 Agent 2 LLM
     LLM 输出: "HOLD, 离 TP 还远, 继续观察。
               summary: 入场 $5.20 现 $5.80 浮盈 11.5%, AAPL 上行势头未变..."

  3. 写 agent2_decisions 表 (审计 + 下次 review 拉 summary)

  4. HOLD → 不塞新任务, handler 返回
```

**循环继续**:
- 10:05 ET → HOLD
- 10:10 ET → HOLD
- ...
- 11:30 ET → LLM 输出 PARTIAL_CLOSE 50 手 (见下一步)

---

### Step 7 — 部分平仓 (LLM 决定 PARTIAL_CLOSE 50%)

```
11:30 ET AGENT2_REVIEW_TICK handler:
  ...
  LLM 输出: "PARTIAL_CLOSE 50 手, 已浮盈 25%, 落袋为安一半。
            summary: ..."
  ↓
塞: event_type = EXECUTE_DECISION
    payload = {opp_id: 5, action: "EXIT", qty: 50, ...}

EXECUTE_DECISION handler:
  OrderClient.place_split_order(SELL 50 手, 拆 5×10)
  ↓ 全部成交
  塞: event_type = EXIT_FILLED
      payload = {opp_id: 5, order_id: 67, qty: 50, avg_price: $6.50}

EXIT_FILLED handler:
  1. 写 fills 表
  2. positions 表 qty 100 → 50
  3. AccountSnapshotClient.invalidate_cache()
  4. opp #5 status 仍 HOLDING (还持仓 50 手)
  5. next_review_due 不变 (继续 review 剩下 50 手)
```

---

### Step 8 — 全平仓 + 取消订阅 (LLM 决定 FULL_CLOSE)

```
12:00 ET AGENT2_REVIEW_TICK handler:
  ...
  LLM 输出: "FULL_CLOSE, 触及阻力位, 平剩下 50 手。"
  ↓
塞 EXECUTE_DECISION → OrderClient 卖剩 50 手 → EXIT_FILLED handler:

  1. 写 fills 表
  2. positions 表 qty 50 → 0
  3. 标 opp #5 status = COMPLETED
  4. DELETE 从 active_reviews (不再 review)

  5. 取消订阅 (引用计数自动去重):
     - MarketDataPool.unsubscribe(AAPL, subscriber=opp_5)
       → ref_count[AAPL]--
       → 若 ref_count = 0 (没其他 opp 监控 AAPL): MarketDataPool 真调 cancelMktData(AAPL)
       → 若 ref_count > 0: 仅减计数, IBKR 仍订阅 (其他 opp 共享)
     - MarketDataPool.unsubscribe(AAPL_285C_2026-05-15, subscriber=opp_5)
       → 同理

  6. AccountSnapshotClient.invalidate_cache()
  7. 写归档表 + 释放风险预算
```

---

### Step 9 — 16:00 ET (盘后): SYSTEM_SLEEP

```
16:00 ET 时间工人塞:
  event_type = SYSTEM_SLEEP

SYSTEM_SLEEP handler:
  1. 时间工人停止排新任务 (active_reviews 表保留, 第二天恢复)
  2. MarketDataPool 不主动 cancelMktData
     (保留 ref_count 业务订阅意图, 等 IBKR 自己 05:30 ET auto-restart 时断)
  3. 4 个长连接保持 (反正 IBKR 会自动断, 省一次 API 调用)
```

第二天 09:00 ET 时间工人又塞 SYSTEM_WAKE_UP, Pool 重连 + 遍历 ref_count > 0 的 symbol 全部 reqMktData 重新订阅, 循环开始。

---

## <span class="h-num">5.</span> 4 个 Pool/Client 在场景里各自做什么

| Pool/Client | 主动操作 (我们调它) | 被动响应 (IBKR 推来) |
|---|---|---|
| **MarketDataPool** | Step 1 connect; Step 1 subscribe; Step 3/6 get_latest; Step 8 unsubscribe | **Step 2 收 tick** → 触发 on_tick callback → 塞 CONDITION_MET 任务 |
| **QueryClient** | Step 3 req_option_chain | 收响应 → 直接返给 caller (不塞任务) |
| **OrderClient** | Step 4 place_split_order; Step 7/8 平仓 | Step 4 收 orderStatus → 写 orders/fills 表 → 累计 FILLED 后塞 ENTRY_FILLED / EXIT_FILLED 任务 |
| **AccountSnapshotClient** | Step 3/5/6/8 get_balance / get_positions / invalidate_cache | (相对少, 主要 pull 模式) |

**Step 2 的 tick callback 是 Step 1 subscribe 的结果** — 我们调一次 subscribe 后, IBKR 持续推 tick, 直到 unsubscribe。

---

## <span class="h-num">6.</span> 长连接的本质 — 不是因为"持续推流", 是 IBKR 协议要求

直觉上你可能以为"只有 MarketDataPool 需要长连 (因为持续推 tick), 其他 Pool/Client 一问一答应该短连"。**实际不是**。

**IBKR API 协议是 EWrapper/EClient 异步双向消息模型**:

- 你调 `OrderClient.placeOrder(...)` — 这只是把订单请求**写到 socket** (异步, 不返回结果)
- IBKR 处理后, 通过**同一个 socket 反向推**响应回来 (orderStatus / openOrder / execDetails)
- 客户端的 EWrapper 监听 socket → 收到响应 → 调对应 callback (你预先注册的 `onOrderStatus`)

所以"发请求"和"收响应"是**两个独立动作**, 中间随时可能有响应推回来。**socket 必须持续打开**, 否则:

- 短连每次 connect 要重做 ibapi handshake (auth + version negotiation, ~1-2 秒)
- 短连关闭瞬间如果有响应在路上, 直接丢
- 订单状态推送 / 账户变化推送 / 错误码推送都收不到

**心跳是副产品, 不是主因**: 长连后 IBKR 每 1 秒 ping 一次, 客户端 30 秒不响应就被 IBKR 主动断。心跳让"连接还活着"可检测, 但目的是**维持响应通道**, 不是为了心跳本身。

**回到 4 个 Pool/Client**: 4 个独立 socket, 互不阻塞 (OrderClient 在等下单响应不影响 MarketDataPool 收 tick), 全部用同一份长连协议。

---

## <span class="h-num">7.</span> 掉线了怎么办 — 3 类异常 + 应对

### 7.1 MarketDataPool 跟 IBKR 断 (网络抖动 / IB Gateway 异常重启)

```
Pool 内部状态机:
  CONNECTED → 检测断 → DISCONNECTED → 自动 retry (指数退避 30s → 1min → 2min → 5min)
            → 重连成功 → 遍历 ref_count > 0 的 symbol 全部 resubscribe → CONNECTED
```

期间 review handler 调 `get_latest(AAPL)` 返回 stale cache + 抛 `DataStaleError` → handler abort 这次 review, 下次 tick 重试。

**Pool 对业务层屏蔽连接事件**, 业务层只看到 "Pool ready / Pool degraded" 两个状态。

### 7.2 Worker 进程崩溃 (handler 跑到一半)

```
workflow_tasks 表:
  task_id | event_type | status   | picked_at
  100     | EXECUTE... | RUNNING  | 11:30:15  ← 卡在这, handler 没标 DONE

重启后扫表:
  WHERE status = RUNNING AND picked_at < now() - 60 sec
  → 重置为 PENDING 重做
```

要求 handler **idempotent** (重做不出错): 例如下单 handler 要先查 orders 表是否已有相同 client_order_id 的单, 有则跳过。

### 7.3 APScheduler 进程崩溃

```
重启后从 DB 的 active_reviews 表恢复:
  从 next_review_due_at 算起继续排 review (无丢失)
```

**核心设计**: 所有状态都持久化到 DB, in-memory 只是 cache。任何进程崩溃, 重启后从 DB 恢复, 不丢任务。

---

## <span class="h-num">8.</span> 8 种 event_type 全清单

任务队列里所有任务类型:

| # | event_type | 谁触发 | 谁消费 | 作用域 |
|---|---|---|---|---|
| 1 | `SYSTEM_WAKE_UP` | 时间工人 (盘前 30 min, APScheduler + pandas_market_calendars 排) | 全局 (Pool 重连 + 重订阅) | 全局 |
| 2 | `SYSTEM_SLEEP` | 时间工人 (盘后) | 全局 (停 active_reviews) | 全局 |
| 3 | `CONDITION_MET(opp_id)` | MarketDataPool tick callback | 业务逻辑 → Agent 2 入场决策 | 单 opp |
| 4 | `AGENT2_REVIEW_TICK(opp_id)` | 时间工人 (5 min 周期 / 事件窗口连续) | 信息工人 → Agent 2 LLM | 单 opp |
| 5 | `EXECUTE_DECISION(decision)` | CONDITION_MET / AGENT2_REVIEW_TICK handler (Agent 2 输出后) | OrderClient | 单决策 |
| 6 | `ENTRY_FILLED(opp_id, order_id)` | OrderClient orderStatus callback (累计 FILLED) | 业务逻辑 → 写持仓 + 启动 review | 单 opp |
| 7 | `EXIT_FILLED(opp_id, order_id)` | 同上 | 业务逻辑 → 减持仓 / 全平归档 | 单 opp |
| 8 | `EXPIRE_OPPORTUNITY(opp_id)` | 时间工人 (机会单 effective_until 到点) | 业务逻辑 → 标 FAILED | 单 opp |

---

## <span class="h-num">9.</span> 数据库架构

新架构对 DB 的影响。共 5 类表。

### 9.1 业务事实表 (核心)

| 表 | 用途 | 关键字段 |
|---|---|---|
| `opportunities` | 机会单 | `opp_id`, `symbol`, `trigger`, `direction`, `raw_input_text`, `status` (ACTIVE_MONITORING / HOLDING / COMPLETED / FAILED), `origin_type` (USER_SUBMITTED / AUTO_FROM_NEWS...), `origin_metadata` jsonb, `effective_until`, `cancel_reason_zh` |
| `orders` | 订单 | `order_id`, `opp_id`, `client_order_id` (idempotent key), `side`, `qty`, `limit_price`, `algo_strategy`, `split_order_batch_id` |
| `fills` | 成交 | `fill_id`, `order_id`, `qty`, `price`, `filled_at` |
| `positions` | 持仓 | `opp_id`, `contract`, `qty`, `avg_cost`, `updated_at` |

### 9.2 任务队列表 (核心新增)

| 表 | 用途 | 关键字段 |
|---|---|---|
| `workflow_tasks` | 8 种 event_type 任务总队列, 崩溃恢复用 | `task_id`, `event_type`, `payload` jsonb, `status` (PENDING / RUNNING / DONE / FAILED), `created_at`, `picked_at`, `done_at`, `handler_result` jsonb, `retry_count` |
| `active_reviews` | 5 min review 排程持久化 | `opp_id`, `next_review_due_at`, `review_interval_sec` (300 正常 / 30 事件窗口) |

### 9.3 基础设施层状态表 (新增)

| 表 | 用途 | 关键字段 |
|---|---|---|
| `market_data_subscriptions` | MarketDataPool ref_count 持久化, 重启恢复 | `symbol`, `subscriber_id` (opp_id 或其他), `ref_count`, `first_subscribed_at`, `last_active_at` |
| `pool_health_log` | Pool 连接状态历史 | `timestamp`, `pool_name`, `state` (READY / DEGRADED / DISCONNECTED), `error_msg` |

### 9.4 审计 / Trace 表

| 表 | 用途 | 关键字段 |
|---|---|---|
| `agent1_traces` | Agent 1 LLM input/output | `trace_id`, `opp_id`, `raw_input_text`, `parsed_json`, `blockers`, `model_id`, `latency_ms` |
| `agent2_decisions` | Agent 2 LLM 每次决策 (入场 + review) | `decision_id`, `opp_id`, `bundle_snapshot` jsonb (10 维), `decision` jsonb, `summary_zh` (≤600 字), `model_id`, `latency_ms`, `decided_at` |

### 9.5 配置 / Catalog 表 (查表数据, 跟代码解耦)

| 表 | 用途 | 关键字段 |
|---|---|---|
| `strategy_whitelist` | 12 种策略 + direction mapping (risk_gate Layer 1 schema 从这查表生成) | `strategy_name`, `legs_count`, `direction`, `liquidity_constraint` (e.g. "仅 SPY/QQQ/IWM") |
| `risk_gate_mapping` | "策略 → 方向" mapping (Layer 2 验证查表) | `strategy_name`, `expected_direction`, `notes` |
| `event_calendar` | 事件熔断窗口表 (财报/CPI/FOMC) | `symbol`, `event_type` (EARNINGS / CPI / FOMC), `event_at`, `pre_window_min`, `post_window_min`, `source` |
| `exchange_routing` | per-symbol 路由 (已有, invariant 23) | `symbol`, `exchange`, `currency`, `primary_exchange`, `option_exchange` |

### 9.6 不存的 (in-memory 即可)

- MarketDataPool callback list (重启时从 market_data_subscriptions 表重建)
- AccountSnapshotClient cache TTL (重启时清空, 自然 refresh)
- OrderClient pending order state (跟 IBKR 对齐, 重启时 reqOpenOrders 同步)

---

## <span class="h-num">10.</span> 并发上限 — 你账户能装多少

3 个独立维度, **取最小值**就是真实上限:

### 10.1 资金限制 (通常是真上限)

客户账户量级 (~$2M RMB)。按典型仓位金额估:

| 策略类型 | 单仓位资金占用 | 可装数量 |
|---|---|---|
| LONG_CALL/PUT 100 手, 权利金 $5/手 | ~$50k | 5-6 个 |
| LONG_STRADDLE 100 手 (双腿权利金 $8) | ~$80k | 3-4 个 |
| BULL_CALL_SPREAD 100 手 (净 debit $2) | ~$20k | 14 个 |
| IRON_CONDOR 100 手 (margin = spread 5 - net credit 1) | ~$40k | 7 个 |

**保留 30% buffer 防 margin call** → 实际 **3-5 个并发持仓** 安全。

### 10.2 IBKR 数据订阅 stream 限制 (Pro account 默认 100 stream)

每个机会单订阅哪些 stream:
- 1 underlying spot 价
- N 腿期权报价 (LONG_CALL=1, IRON_CONDOR=4)

**MarketDataPool 引用计数去重** — 多个机会单共享同 underlying 节省 stream。

实际计算: 假设并发 10 个机会单, 涉及 5 个不同 underlying (AAPL/TSLA/SPY/QQQ/NVDA), underlying = 5 (去重后) + 期权 = 10 opp × 平均 2 腿 = 20 → 总 25 stream。**100 配额够装 30+ 机会单, 不是瓶颈**。

### 10.3 IBKR client_id 限制 (32 个并发连接)

我们只用 5 个长连 client (4 基础设施 + 1 临时段位), 32 上限**完全用不完**。

### 10.4 综合上限

```
真实上限 = min(资金限制, 订阅限制, client 限制)
        = min(3-5 个持仓, 30+ 机会单, 32 client)
        = 3-5 个并发持仓 (资金是真瓶颈)
```

**重要区分**:

| 指标 | 上限 | 含义 |
|---|---|---|
| **持仓数量** | **3-5 个** | 已下单未平仓的机会单 (吃资金) |
| **机会单总数** | **20-30 个** | 持仓 + 监控中未触发的草稿 (不吃资金, 只吃订阅 stream) |

监控中的机会单只占订阅配额, 不占资金。所以可以同时**监控 20 个条件**, 但只**持仓 3-5 个**。条件触发后如果资金不够, 风险门拒绝入场或要求等其他仓位平掉。

---

## <span class="h-num">11.</span> 一句话总结

**没有"独立工人 actor"**, 只有:
- **APScheduler** (排程) +
- **8 种任务的 handler 函数** +
- **4 个基础设施抽象** (Pool/Client) +
- **任务队列** (workflow_tasks 表) 串起来。

业务代码永远不直接碰 IBKR API, 全走 Pool/Client; 工人之间不互相调函数, 全走任务队列。这两条是架构稳定性的核心。

---

## 维护规则

- 改架构 (新加 event_type / 新加 Pool/Client / 改 DB schema) → 同时回到此文档更新
- 改 [⭐ 北极星 §1](north-star-v1-target.md) 涉及工人 / 数据 bundle / 风险门 → 同步本文档
- 任何与北极星 §1 冲突, 以北极星为准, 本文档 stale 标记待更新

---

**END OF ARCHITECTURE WALKTHROUGH**
