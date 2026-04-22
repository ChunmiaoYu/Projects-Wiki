# 10 · 配置 · 环境变量 · 依赖

> 对应文件：`config.json`、`config.py`、`web/settings.py`、`.env.example`、`requirements.txt`、`build_release.bat`。本章把 APS 所有"可配置点"集中列出，方便查阅。

---

## 1. 两套配置系统

| 系统 | 用于 | 文件 | 读取 |
|---|---|---|---|
| 桌面版 | `main.py` 流水线 | `config.json`（客户可编辑）+ 环境变量 | `config.SETTINGS` 单例 |
| 云端版 | FastAPI / worker / pipeline | `.env` | `web.settings.get_settings()` |

**共享**：两边都会用 `config.SETTINGS` 读 `config.json`（比如 `TRADEME_SEARCH` / `ZONE_ALLOWLIST` / scoring weights）。云端部署时 `.env` 和 `config.json` 都要在 `/opt/aps/` 下。

---

## 2. `config.json`（桌面版客户可编辑）

**设计原则**（见 `config.py:90-96` 的注释）：
- `config.json` 只放**客户可编辑的业务设置**
- **技术默认**（logging / HTTP throttle / scoring weights / internal paths）**写死在代码里**
- 进阶用户可以用**环境变量**覆盖（但不推荐在客户版暴露）

### 2.1 模板

```json
{
  "output_path": "output",
  "shortlist": {
    "top_n": 100000
  },
  "providers": {
    "trademe": {
      "search": {
        "params": {
          "price_max": 50000000,
          "land_area_min": 0.02,
          "land_area_max": 10
        },
        "startdate": "",
        "listed_within_days": 1,
        "max_pages": 2000
      }
    }
  },
  "zone_allowlist": {
    "18": "Residential - Mixed Housing Suburban",
    "60": "Residential - Mixed Housing Urban",
    "8":  "Residential - THAB (Terrace Housing & Apartments)"
  }
}
```

### 2.2 字段说明

| 路径 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `output_path` | string | `output` | Excel 输出目录或完整文件路径（以 `.xlsx` 结尾视为文件） |
| `shortlist.top_n` | int | 10 | Top N 短名单大小；**≤ 0 表示保留全部**（evaluation 用 100000 表示 all） |
| `providers.trademe.search.params.price_max` | int | - | Trade Me 价格上限 NZD |
| `providers.trademe.search.params.land_area_min/max` | float | - | 土地面积单位**公顷**（0.02 ha ≈ 200 m²） |
| `providers.trademe.search.startdate` | string `YYYY-MM-DD` | `""` | 锚定日期（空=今天）；用于**复现历史运行**。必须在 [today-365, today] |
| `providers.trademe.search.listed_within_days` | int | - | 最近 N 天窗口；0/负 = 不限 |
| `providers.trademe.search.max_pages` | int | - | 硬性页数上限 |
| `zone_allowlist` | dict<str,str> | `{18, 60, 8, 19}` | Unitary Plan Base Zone Code → 名字。只保留列出的 zone |

### 2.3 命名兼容

代码里对多个 key 有 fallback（`config.py:99-123`）：
- `output_path` → fallback `output.output_dir`
- `configure.json` → fallback `config.json`
- `startdate` → fallback `start_date`

---

## 3. 桌面版环境变量（进阶）

这些**不写进 config.json**，只在需要调优时通过环境变量传。

### 3.1 输入 / 输出

| ENV | 默认 | 含义 |
|---|---|---|
| `APS_CONFIG` | `` | 指定 config.json 位置（绝对路径） |
| `APS_DATA_DIR` | `./data` | 离线 GIS 数据目录 |
| `APS_LOG_DIR` | `./log` | 日志目录 |
| `APS_LOG_LEVEL` | `INFO` | `DEBUG`/`INFO`/`WARNING`/`ERROR` |
| `APS_LOG_FILE_PREFIX` | `shortlist` | 日志文件名前缀 |
| `APS_SAVE_DEBUG_HTML` | `0` | 保存抓取过程的 HTML（仅调试） |

### 3.2 Trade Me 抓取

| ENV | 默认 | 含义 |
|---|---|---|
| `APS_DELAY_SECONDS` | `0.35` | 每页请求之间延迟（礼貌 + 防反爬） |
| `APS_TIMEOUT_SECONDS` | `30.0` | 每次 HTTP 请求 timeout |
| `APS_USER_AGENT` | Chrome 122 | 自定义 UA |
| `APS_TOP_N` | `10` | config 之外再兜底 |

### 3.3 GIS 调优

| ENV | 默认 | 含义 |
|---|---|---|
| `APS_GIS_FIRST_ONLY` | `0` | 只对第一条做 full GIS（debug 模式） |
| `APS_GIS_BBOX_BUFFER_M` | `600.0` | 窗口读 buffer |
| `APS_GIS_PARCEL_SEARCH_START_M` | `150.0` | Parcel 匹配起始窗口 |
| `APS_GIS_PARCEL_SEARCH_MAX_M` | `1200.0` | Parcel 匹配最大窗口 |
| `APS_GIS_PIPE_SEARCH_RADIUS_M` | `100.0` | 管线距离搜索半径 |
| `APS_GIS_OVERLAND_SEARCH_RADIUS_M` | `100.0` | 地表径流搜索半径 |
| `APS_GIS_POLYGON_WINDOW_PAD_M` | `30.0` | 多边形图层窗口 padding |
| `APS_GIS_SLOPE_TIMING` | `0` | 打印 slope raster 每阶段 ms |
| `APS_GIS_TIMING` | `0` | 打印各 GIS 阶段 ms |
| `APS_GIS_EARLY_EXIT` | `1` | 硬过滤前早退（省时间） |

### 3.4 并行度

| ENV | 默认 | 含义 |
|---|---|---|
| `APS_GIS_WORKERS` | auto | Stage 1 并行 worker 数 |
| `APS_GIS_CHUNK_SIZE` | auto | Stage 1 chunksize |
| `APS_OUTLINE_WORKERS` | auto | Stage 2 worker 数 |
| `APS_OUTLINE_CHUNK_SIZE` | auto | Stage 2 chunksize |
| `APS_CV_THREADS` | auto | CV 查询线程数 |

### 3.5 Scoring 权重与归一化

全部在 `config.py:286-316`，都有对应环境变量：

| ENV | 默认 | 含义 |
|---|---|---|
| `WEIGHT_FRONTAGE` | `0.20` | frontage 权重 |
| `WEIGHT_SEWER_DISTANCE` | `0.20` | 污水管距离权重 |
| `WEIGHT_STORMWATER_DISTANCE` | `0.10` | 雨水管距离权重 |
| `WEIGHT_RENT_3BR` | `0.15` | 3 卧租金权重 |
| `WEIGHT_SLOPE` | `0.10` | 坡度权重 |
| `WEIGHT_LAND_AREA` | `0.20` | 土地面积权重 |
| `WEIGHT_OLDNESS` | `0.05` | 房龄权重 |
| `NORM_FRONTAGE_MIN/MAX` | `8 / 25` | 归一化区间（m） |
| `NORM_SEWER_DISTANCE_MIN/MAX` | `0 / 30` | （m） |
| `NORM_STORMWATER_DISTANCE_MIN/MAX` | `0 / 30` | （m） |
| `NORM_RENT_3BR_MIN/MAX` | `500 / 1200` | 周租金（NZD） |
| `NORM_SLOPE_MIN/MAX` | `0 / 15` | 度 |
| `NORM_LAND_AREA_MIN/MAX` | `500 / 1500` | m² |
| `NORM_OLDNESS_MIN/MAX` | `0 / 80` | 年 |
| `MISSING_NEUTRAL` | `0.5` | 缺失值中性分 |
| `MISSING_PENALTY` | `0.0` | 缺失额外扣分 |

---

## 4. 云端版 `.env`（`web/settings.py`）

### 4.1 `.env.example`

```bash
APP_NAME=aps
APP_ENV=dev
LOG_LEVEL=INFO
DATABASE_URL=sqlite:///data/aps.db
JWT_SECRET_KEY=<generate-with-python3 -c "import secrets; print(secrets.token_urlsafe(64))">
JWT_EXPIRE_MINUTES=1440
API_HOST=0.0.0.0
API_PORT=8080

# In-process pipeline scheduler (0 = disabled).
# Leave 0 when cron triggers pipeline (PROD / any host with a cron entry).
# Set to e.g. 60 only if the host has NO cron.
PIPELINE_INTERVAL_MINUTES=0
```

### 4.2 字段解释（`web/settings.py:5-22`）

| 字段 | 默认 | 用途 |
|---|---|---|
| `APP_NAME` | `aps` | 标识符（日志前缀 / health JSON） |
| `APP_ENV` | `dev` | `dev` / `production`。`dev` 开启 uvicorn reload + FastAPI `/docs` |
| `LOG_LEVEL` | `INFO` | Python logging level |
| `DATABASE_URL` | `sqlite:///data/aps.db` | SQLAlchemy URL。PROD 用绝对路径 `sqlite:////opt/aps/data/aps.db` |
| `JWT_SECRET_KEY` | placeholder | HS256 密钥。PROD 必须换（init-vm.sh 自动生成 64 字节） |
| `JWT_EXPIRE_MINUTES` | `1440` | JWT 过期（1440 = 24h） |
| `API_HOST` | `0.0.0.0` | Uvicorn bind（all interfaces） |
| `API_PORT` | `8080` | Uvicorn port |
| `PIPELINE_INTERVAL_MINUTES` | `0` | 0 = 禁用 in-process scheduler（PROD 用 cron）；> 0 = 开 daemon thread |
| `PIPELINE_START_HOUR` | `6` | scheduler active window 起点（包含） |
| `PIPELINE_END_HOUR` | `24` | scheduler active window 终点（不含；24 = midnight） |

### 4.3 pydantic-settings

`web/settings.py` 用 `pydantic_settings.BaseSettings`（Pydantic v2）：

```python
class WebSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )
    app_name: str = "aps"
    ...
```

环境变量优先级：OS env var > `.env` 文件 > 默认值。

### 4.4 `extra="ignore"` 允许多余字段

意味着 `.env` 里有 `APS_GIS_WORKERS=...` 这种桌面版变量不会 crash（云端不用但不报错）。便于一份 `.env` 两边共用。

---

## 5. Python 依赖（`requirements.txt`）

```
requests                  # HTTP client（Trade Me 抓取 + internal notify）
beautifulsoup4            # HTML fallback 解析
pandas                    # Excel 处理辅助
openpyxl                  # .xlsx 读写

# GIS
geopandas                 # 矢量（gpkg 读写）
shapely                   # 几何运算
pyproj                    # CRS 投影
rtree                     # 空间索引
rasterio                  # 栅格（DEM tif）
matplotlib                # parcel outline PNG 渲染

# --- Web platform ---
fastapi>=0.115.0
uvicorn[standard]>=0.30.0
sqlalchemy>=2.0.0
alembic>=1.13.0
pydantic-settings>=2.0.0
bcrypt>=4.2.0
pyjwt>=2.9.0
jinja2>=3.1.0
python-multipart>=0.0.9   # form data（登录 form）

# --- Testing ---
pytest>=8.0.0
httpx>=0.27.0             # TestClient
```

### 5.1 没列出但间接装的

- `cryptography` — 通过 `pyjwt` 和 `aps_license` 间接依赖（Ed25519 + DPAPI 不需要它，但如果用 RS256 就需要）
- `python-dotenv` — config.py 里 soft import（`config.py:10-14`）用于加载 `.env`
- `numpy` — geopandas / rasterio 的底层依赖

### 5.2 PyInstaller 依赖（桌面版构建）

`build_release.bat` 用 PyInstaller 打包 onedir：
- `pyinstaller` — 构建 .exe
- `pywin32` — 非必须（DPAPI 用 ctypes 实现，避免依赖）
- `playwright` — **不打包**（太大），保留运行时 fallback 但默认不用

---

## 6. 构建桌面版（`build_release.bat`）

```batch
REM 概要（完整见 build_release.bat）
call venv\Scripts\activate
pyinstaller ^
    --noconfirm --clean ^
    --name APS ^
    --onedir ^
    --icon Property_Icon.ico ^
    --hidden-import geopandas ^
    --collect-submodules rasterio ^
    aps_bootstrap.py

pyinstaller ^
    --noconfirm --clean ^
    --name activate ^
    --onefile ^
    activate.py

REM Copy data/ config.json into release/
xcopy data release\APS\_internal\data\ /E /I /Y
copy config.json release\APS\
```

输出：
- `release/APS/APS.exe` + `_internal/` 目录（含 Python runtime + deps + 数据）
- `release/activate.exe`（单文件，给客户先激活用）

### 6.1 onedir vs onefile

- `APS.exe` 用 **onedir**：启动快（免每次解压临时目录）、体积对 GIS stack 可接受
- `activate.exe` 用 **onefile**：小巧便携，用户双击就能跑

### 6.2 发布前自查

`build_release.bat` 尾部通常跑一次 smoke test：
- 新 VM / 新用户目录下运行一次
- 确认 license gate 弹窗（没 license.bin 时）
- 确认 `activate.exe --machine-id` 能产出 txt
- 确认安装 license.lic 后 APS.exe 能跑

---

## 7. 配置冲突 / 优先级

### 7.1 桌面版

1. 环境变量（`APS_*`, `WEIGHT_*`, `NORM_*`）
2. `config.json` 的 `providers.trademe.search` / `zone_allowlist`
3. 代码默认（`config.py` class body）

### 7.2 云端版

1. OS 环境变量
2. `.env` 文件
3. `WebSettings` 类默认

### 7.3 注意

**不会**去读：
- `~/.env`（只读当前工作目录的）
- `/etc/environment`（除非 systemd EnvironmentFile 引入）

systemd `aps-web.service` 有 `EnvironmentFile=/opt/aps/.env`，所以 `.env` 在 PROD 生效是靠这条。如果手动 `python run_web.py` 不传 env，`.env` 文件在当前目录就会被 pydantic-settings 读到。

---

## 8. 典型部署三元组

### 8.1 开发机（Windows，运行桌面版）

- `config.json`（客户实际用的版本）
- `./data/` 完整 GIS 数据
- `./log/`（自动创建）
- `./output/`（自动创建）

### 8.2 开发机（任意 OS，运行云端版）

- `.env`：`APP_ENV=dev`, `PIPELINE_INTERVAL_MINUTES=60`（自带 scheduler）
- `data/aps.db`（自动创建；测试用 in-memory）
- 可选 `./data/` 放 GIS（但测试不要求）

### 8.3 PROD VM

- `/opt/aps/.env`（`APP_ENV=production`, `PIPELINE_INTERVAL_MINUTES=0`, DATABASE_URL 绝对路径）
- `/opt/aps/data/aps.db` + `/opt/aps/data/*.gpkg + .tif`
- `/opt/aps/log/`
- `/opt/aps/venv/`（systemd unit 指向这个 Python）

---

## 9. Secrets 管理

| Secret | 哪里生成 | 哪里存 |
|---|---|---|
| JWT_SECRET_KEY | `init-vm.sh:step_create_env` 用 `secrets.token_urlsafe(64)` | `/opt/aps/.env`（chmod 600） |
| Admin 密码 | init 时 CLI 传 `APS_ADMIN_PASS` 或自动生成 | DB `users.credential_hash`（bcrypt） |
| Ed25519 license 私钥 | 发行方本地 `license_gen.py gen-keys --passphrase ...` | 发行方电脑（private key passphrase 加密） |
| SSH key to VM | 用户本地 `~/.ssh/id_ed25519` | 本地 + GitHub Secrets `APS_SSH_KEY` |
| VM IP | Oracle Cloud Console | GitHub Secrets `APS_VM_IP` |
| Trade Me cookie | 无（APS 不登录 Trade Me，只用公开 API） | - |

### 9.1 不进 git 的文件

`.gitignore` 拦的：
- `.env`（但 `.env.example` 在）
- `keys/`（Ed25519 私钥）
- `license.lic`（客户收到的 license 文件；repo 只有 example `license.lic` 用于 dev）
- `machine_id.txt`
- `data/` / `.venv/` / `release/` / `log/` / `output/`
- `*.db*`（SQLite 文件）

---

## 10. 参考文件

- `config.py`（320 行）— 桌面版 SETTINGS 单例
- `config.json`（28 行）— 客户可编辑
- `web/settings.py`（27 行）— 云端 pydantic-settings
- `.env.example`（13 行）
- `requirements.txt`（27 行）
- `build_release.bat`（桌面版构建脚本）
- `deploy/init-vm.sh`（376 行，云端初始化）
