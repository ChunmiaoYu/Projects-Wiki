<!-- PAGE_ID: options_08_dependencies -->
<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [pyproject.toml:1-45](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L1-L45)
- [settings.py:1-59](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L1-L59)
- [INSTALL_LOCAL_WINDOWS.md:1-150](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/docs/INSTALL_LOCAL_WINDOWS.md#L1-L150)

</details>

# 依赖项与配置

> **Related Pages**: [[系统架构|02_architecture.md]]

---

<!-- BEGIN:AUTOGEN options_08_dependencies_python -->
## Python 依赖清单

项目要求 Python >= 3.11，使用 setuptools 构建（[pyproject.toml:2-3](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L2-L3)）。所有依赖在 `pyproject.toml` 的 `[project] dependencies` 中声明（[pyproject.toml:11-27](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L11-L27)）。

### 核心依赖

| 包名 | 版本约束 | 用途 |
|------|----------|------|
| `fastapi` | >= 0.115.0 | Web API 框架，提供 REST 端点 ([pyproject.toml:12](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L12)) |
| `uvicorn[standard]` | >= 0.30.0 | ASGI 服务器，运行 FastAPI 应用 ([pyproject.toml:13](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L13)) |
| `sqlalchemy` | >= 2.0.30 | ORM 框架，定义 17 张数据表模型 ([pyproject.toml:14](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L14)) |
| `psycopg[binary]` | >= 3.2.0 | PostgreSQL 驱动（psycopg3） ([pyproject.toml:15](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L15)) |
| `alembic` | >= 1.13.2 | 数据库迁移工具（已有 0001-0004 四次迁移） ([pyproject.toml:16](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L16)) |
| `pydantic` | >= 2.7.0 | 数据验证与序列化，定义 API schema ([pyproject.toml:17](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L17)) |
| `pydantic-settings` | >= 2.3.0 | 从 `.env` 文件和环境变量加载配置 ([pyproject.toml:18](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L18)) |
| `openai` | >= 1.40.0 | OpenAI API 客户端，用于 Agent1/Agent2 的 Structured Outputs 调用 ([pyproject.toml:19](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L19)) |
| `python-dotenv` | >= 1.0.1 | `.env` 文件加载 ([pyproject.toml:20](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L20)) |
| `httpx` | >= 0.27.0 | 异步 HTTP 客户端 ([pyproject.toml:21](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L21)) |
| `orjson` | >= 3.10.0 | 高性能 JSON 序列化/反序列化 ([pyproject.toml:22](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L22)) |
| `tenacity` | >= 8.4.0 | 重试机制（用于 OpenAI API 调用等） ([pyproject.toml:23](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L23)) |
| `langgraph` | >= 0.4.0 | LangChain 图工作流框架，实现 Agent1 的 7 节点 intake 工作流 ([pyproject.toml:24](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L24)) |
| `langchain-core` | >= 0.3.0 | LangChain 核心抽象层 ([pyproject.toml:25](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L25)) |
| `PyYAML` | >= 6.0.2 | YAML 配置文件解析（business_lexicon、trigger_catalog、default_policies） ([pyproject.toml:26](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L26)) |

### 测试依赖

| 包名 | 版本约束 | 用途 |
|------|----------|------|
| `pytest` | >= 8.2.0 | 测试框架 ([pyproject.toml:33-34](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L33-L34)) |

### 未在 pyproject.toml 中声明的运行时依赖

- **ibapi**（IBKR TWS API）：通过 IBKR 官方安装包手动安装，不在 PyPI 上分发
- **Docker**：运行 PostgreSQL 容器

### 包数据

项目打包时包含 `config/*.yml` 和 `prompts/*.md` 文件（[pyproject.toml:29-30](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L29-L30)）。

Sources: [pyproject.toml:1-45](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/pyproject.toml#L1-L45)
<!-- END:AUTOGEN options_08_dependencies_python -->

---

<!-- BEGIN:AUTOGEN options_08_dependencies_env -->
## 环境变量配置

所有配置通过 `pydantic-settings` 的 `Settings` 类管理，支持 `.env` 文件和环境变量两种方式加载（[settings.py:10](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L10)）。未知的环境变量会被忽略（`extra="ignore"`）。

### 应用配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `APP_NAME` | str | `options-event-trader` | 应用名称 ([settings.py:12](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L12)) |
| `APP_ENV` | str | `dev` | 运行环境（dev/uat/prod） ([settings.py:13](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L13)) |
| `API_HOST` | str | `0.0.0.0` | API 服务监听地址 ([settings.py:14](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L14)) |
| `API_PORT` | int | `8080` | API 服务端口 ([settings.py:15](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L15)) |
| `LOG_LEVEL` | str | `INFO` | 日志级别 ([settings.py:16](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L16)) |

### 数据库配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `POSTGRES_HOST` | str | `localhost` | PostgreSQL 主机 ([settings.py:18](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L18)) |
| `POSTGRES_PORT` | int | `5432` | PostgreSQL 端口 ([settings.py:19](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L19)) |
| `POSTGRES_DB` | str | `options_event_trader` | 数据库名 ([settings.py:20](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L20)) |
| `POSTGRES_USER` | str | `postgres` | 数据库用户名 ([settings.py:21](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L21)) |
| `POSTGRES_PASSWORD` | str | `postgres` | 数据库密码 ([settings.py:22](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L22)) |
| `DATABASE_URL` | str | `None` | 完整连接串（如提供则覆盖上述分项配置） ([settings.py:23](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L23)) |

当 `DATABASE_URL` 未提供时，系统自动拼接连接串：`postgresql+psycopg://{user}:{password}@{host}:{port}/{db}`（[settings.py:42-49](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L42-L49)）。

### OpenAI 配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `OPENAI_API_KEY` | str | `None` | OpenAI API 密钥（为空时 AI 功能禁用） ([settings.py:25](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L25)) |
| `OPENAI_MODEL` | str | `gpt-5` | 使用的 OpenAI 模型名 ([settings.py:26](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L26)) |

`openai_enabled` 计算属性：当 `OPENAI_API_KEY` 非空且非纯空白时返回 `True`（[settings.py:52-54](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L52-L54)）。

### IBKR 配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `IBKR_HOST` | str | `127.0.0.1` | TWS/IB Gateway 主机地址 ([settings.py:28](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L28)) |
| `IBKR_PORT` | int | `7497` | TWS 连接端口 ([settings.py:29](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L29)) |
| `IBKR_CLIENT_ID` | int | `27` | 客户端标识（同一 TWS 实例上不可重复） ([settings.py:30](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L30)) |
| `IBKR_ACCOUNT_MODE` | str | `paper` | 账户模式：paper（模拟）或 live（实盘） ([settings.py:31](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L31)) |
| `IBKR_REQUEST_TIMEOUT_SEC` | int | `15` | 单次 IBKR 请求超时（秒） ([settings.py:32](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L32)) |
| `IBKR_CONNECT_TIMEOUT_SEC` | int | `10` | 连接超时（秒） ([settings.py:33](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L33)) |
| `IBKR_MARKET_DATA_TYPE` | int | `3` | 市场数据类型（1=实时，2=冻结，3=延迟，4=延迟冻结） ([settings.py:34](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L34)) |
| `IBKR_DRY_RUN` | bool | `False` | Dry-run 模式，执行到下单前停止 ([settings.py:35](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L35)) |
| `IBKR_MOCK` | bool | `False` | Mock 模式，使用 MockIBKRClient 替代真实连接 ([settings.py:36](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L36)) |

### Worker 配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `WORKER_POLL_SECONDS` | int | `15` | Worker 轮询间隔（秒） ([settings.py:38](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L38)) |
| `MONITOR_POLL_SECONDS` | int | `10` | 监控轮询间隔（秒） ([settings.py:39](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L39)) |
| `PROMPT_VERSION` | str | `v1` | Prompt 版本标识 ([settings.py:40](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L40)) |

### 配置加载机制

`Settings` 类使用 `@lru_cache(maxsize=1)` 包装的 `get_settings()` 函数，确保全局单例（[settings.py:57-59](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L57-L59)）。配置优先级为：环境变量 > `.env` 文件 > 代码默认值。

Sources: [settings.py:1-59](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L1-L59)
<!-- END:AUTOGEN options_08_dependencies_env -->

---

<!-- BEGIN:AUTOGEN options_08_dependencies_ibkr-setup -->
## IBKR TWS 设置

### 端口配置

| 客户端 | 模拟交易端口 | 实盘端口 |
|--------|-------------|----------|
| TWS (Trader Workstation) | 7497 | 7496 |
| IB Gateway | 4002 | 4001 |

Paper trading 使用真实账户凭证登录 TWS 的"模拟交易"标签页，模拟账户号以 "DU" 开头。

### TWS API 权限设置

以下为已验证的 TWS 配置（[INSTALL_LOCAL_WINDOWS.md:120-121](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/docs/INSTALL_LOCAL_WINDOWS.md#L120-L121)）：

| 设置项 | 值 | 说明 |
|--------|-----|------|
| 启用 ActiveX 和套接字客户端 | 开启 | 允许 API 连接 |
| 只读 API | 关闭 | 需要下单权限 |
| 跳过 API 委托单预防设置 | 开启 | 避免 TWS 弹出确认对话框阻塞自动化流程 |
| 其他预防项 | 全不勾选 | 避免额外的人工确认 |

### 代码层面的订单设置

每个订单必须设置 `order.eTradeOnly = False` 和 `order.firmQuoteOnly = False`，否则 ibapi 默认值 `True` 会导致错误码 10268。

### 市场数据订阅

项目已开通 OPRA 实时期权数据订阅（$32.75/月），解决了 snapshot 模式下 OPRA 数据不可靠的问题，改用 streaming 模式采集期权行情数据。

### 本地安装步骤

安装分为两层（[INSTALL_LOCAL_WINDOWS.md:16-30](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/docs/INSTALL_LOCAL_WINDOWS.md#L16-L30)）：

**手工前置项（只做一次）：**
1. 安装 Docker Desktop
2. 安装 Python 3.11+
3. 安装 IB Gateway/TWS 并登录
4. 填写 `.env.dev` / `.env.uat` 中的 `OPENAI_API_KEY` 和数据库名

**一键初始化（`scripts/bootstrap_local.bat`）：**
1. 检查 Docker / Python
2. 创建 `.venv` 并安装依赖
3. 启动 PostgreSQL 容器
4. 创建 Dev / UAT 数据库
5. 执行 Alembic migration

**启动服务：**
- Dev API：`scripts/start_dev_api.bat`（默认端口 8080）
- Dev Worker：`scripts/start_dev_worker.bat`
- 健康检查：`http://127.0.0.1:8080/health`

**环境验证脚本：**

```powershell
powershell -ExecutionPolicy Bypass -File scripts/verify_prereqs.ps1
```

检查 Docker、Python、IB Gateway 端口（127.0.0.1:7497）、OpenAI key 是否配置（[INSTALL_LOCAL_WINDOWS.md:107-119](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/docs/INSTALL_LOCAL_WINDOWS.md#L107-L119)）。

Sources: [INSTALL_LOCAL_WINDOWS.md:1-150](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/docs/INSTALL_LOCAL_WINDOWS.md#L1-L150), [settings.py:28-36](https://github.com/ChunmiaoYu/options_ai_trader/blob/f5f3ac84e9c5d963fc1450f12306ea264183dfad/src/options_event_trader/settings.py#L28-L36)
<!-- END:AUTOGEN options_08_dependencies_ibkr-setup -->

---
