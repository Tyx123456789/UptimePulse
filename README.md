# UptimePulse

> 服务可用性与响应时间监控平台——盯住你关心的那些网址：什么时候挂了、慢到什么程度、已经挂了多久。

用户添加若干监控目标，后端按各自设定的间隔自动探测，记录每次的连通性、状态码与响应耗时；前端用看板呈现**可用率**、**延迟趋势**、**状态时间线**和**故障事故**，并通过 **SSE 实时推送**自动更新，无需手动刷新。

<!-- 部署完成后替换以下占位 -->
- 🔗 **Demo**：`TODO`（前端地址）
- 📡 **API 健康检查**：`TODO`（`/health`）
- 📷 **截图 / GIF**：`TODO`
- 📖 **架构说明**：[docs/architecture.md](docs/architecture.md)（待补）

---

## 目录

- [它解决什么问题](#它解决什么问题)
- [功能清单](#功能清单)
- [技术栈](#技术栈)
- [系统架构](#系统架构)
- [目录结构](#目录结构)
- [数据库设计](#数据库设计)
- [接口设计](#接口设计)
- [核心业务规则](#核心业务规则)
- [本地开发](#本地开发)
- [环境变量](#环境变量)
- [部署](#部署)
- [测试](#测试)
- [六周开发计划](#六周开发计划)
- [关键技术难点与解法](#关键技术难点与解法)
- [已知限制](#已知限制)
- [不做什么](#不做什么)
- [可延展方向](#可延展方向)

---

## 它解决什么问题

| 问题 | 这个项目给出的答案 |
| --- | --- |
| 学校教务网是不是又崩了？ | 状态时间线一眼看出哪段时间不可用 |
| 我的博客这周稳定性怎么样？ | 可用率 KPI，形如 `99.2%` |
| 网站变慢了还是网络问题？ | 记录每次响应耗时，看 P95 与均值趋势 |
| 到底挂了多久、什么时候恢复的？ | 自动生成「事故」记录，有明确的起止时间与持续时长 |
| 不想每次手动刷？ | SSE 实时推送，看板自己动 |

**使用场景**：校园服务监控（教务系统、选课、图书馆检索）、招聘公告页可用性与更新监控、个人站点稳定性、第三方依赖 API 健康度。

v1 只覆盖**少量目标、分钟级间隔**的场景；不做大规模、秒级、分布式探测。

---

## 功能清单

### P0（必须完成）

| 编号 | 功能 | 说明 |
| --- | --- | --- |
| F1 | 监控目标管理 | 新增、编辑、删除、启用/停用；字段含名称、URL、探测间隔、超时、期望状态码、可选关键字 |
| F2 | 定时探测 | 调度器按每个目标自己的间隔发起探测，结果入库 |
| F3 | 探测结果记录 | 状态码、耗时、成功与否、错误类型与错误信息 |
| F4 | 总览看板 | 每目标一张卡片：当前状态、可用率、P95 延迟、最近探测时间；顶部全局汇总条 |
| F5 | 状态时间线 | 以时间块展示最近 24 小时的成功/失败 |
| F6 | 延迟趋势图 | 折线图展示选中目标在某时间段内的响应耗时 |
| F7 | 事故自动生成 | 连续失败达阈值自动建事故，恢复后自动关闭并计算持续时长 |
| F8 | 实时推送 | 探测结果通过 SSE 推送到前端 |

### P1（有余力再加）

- 告警防抖参数可配置（连续失败次数、恢复确认次数）
- 关键字校验（响应 200 但内容报错也能识别）
- 手动立即探测按钮
- 时间段筛选与 CSV 导出
- TLS 证书剩余天数检测

### P2（后续方向）

- 邮件 / webhook 通知
- 多用户与权限体系
- 区域多节点探测

---

## 技术栈

| 层 | 选型 | 用途 |
| --- | --- | --- |
| 前端 | Vue 3 + Vite + TypeScript + Pinia | 页面与状态管理 |
| 图表 | ECharts | 延迟折线、状态时间线 |
| 后端 | Python 3.11 + FastAPI + SQLAlchemy 2 | HTTP 接口与调度 |
| 异步客户端 | httpx.AsyncClient + `asyncio.gather` | 并发探测 |
| 调度 | APScheduler（AsyncIOScheduler） | 周期性扫描到期目标 |
| 数据库 | PostgreSQL | 时序记录与聚合 |
| 迁移 | Alembic | 表结构版本管理 |
| 部署 | Vercel（前端）+ Render（后端）+ Supabase（数据库） | 免费额度内上线 |
| 工程 | Docker Compose、GitHub Actions | 本地一键启动、lint/test/build |

---

## 系统架构

三层结构，外加一条推送通道：

```mermaid
flowchart LR
    subgraph Browser["浏览器"]
        V["Vue 3 看板<br/>总览 / 详情 / 管理"]
    end

    subgraph Backend["FastAPI 单进程"]
        API["REST API<br/>/api/targets /api/checks ..."]
        SCH["调度器<br/>APScheduler · 每 60s 扫描"]
        PROBE["探测器<br/>httpx + asyncio.gather"]
        INC["事故判定<br/>连续失败/恢复防抖"]
        SSE["SSE 广播<br/>/api/stream"]
    end

    DB[("PostgreSQL<br/>targets · checks · incidents")]
    TARGETS["外部目标站点"]

    V -- "REST 拉取" --> API
    SSE -- "事件推送" --> V
    SCH --> PROBE
    PROBE -- "HTTP/HTTPS 探测" --> TARGETS
    PROBE --> DB
    PROBE --> INC
    INC --> DB
    PROBE --> SSE
    API --> DB
    GH["GitHub Actions<br/>每 15 分钟"] -- "POST /api/cron/run" --> API
```

**接入层**：FastAPI 提供 REST 接口；后台调度器与 Web 服务跑在同一进程里——项目规模小，不单独拆 worker。
**数据层**：PostgreSQL 三张表；探测记录是典型时序数据（写得频繁、按时间范围读），用复合索引支撑。
**展示层**：Vue 三个页面，REST 拉数据，SSE 收推送。

### 一次探测的完整数据流

1. 调度器每 60 秒扫描 `targets`，挑出 `next_check_at` 已到期的目标，并重算下一次探测时间。
2. 对选中目标用 `asyncio.gather` 并发发起请求，每个请求带独立超时；拿到状态码、耗时、错误类型。
3. 逐条写入 `checks`，并更新目标的 `last_checked_at`。
4. 读取该目标最近若干条记录，按防抖规则判断是否开启或关闭事故，写 `incidents`。
5. 把本次结果作为事件推送给所有在线的前端连接（SSE）。
6. 前端更新卡片状态、追加曲线上一个点，状态由通转断时高亮提示。

### 外部触发通道

`POST /api/cron/run` 让外部定时器主动触发一轮探测。存在的理由：免费实例会休眠，进程一睡调度器就停。用 GitHub Actions 每 15 分钟调一次该接口，既唤醒实例又补上探测（详见 [.github/workflows](#部署)）。

---

## 目录结构

```
uptime-pulse/
├── frontend/                  # Vue 3
│   ├── src/views/             # Overview / TargetDetail / Manage
│   ├── src/components/        # StatusBadge、UptimeBar、LatencyChart
│   └── src/api/               # 请求封装
├── backend/
│   ├── app/
│   │   ├── api/               # targets.py、checks.py、stream.py、cron.py
│   │   ├── models/            # SQLAlchemy 模型
│   │   ├── schemas/           # Pydantic 模型
│   │   ├── core/              # config.py、security.py
│   │   ├── services/          # prober.py、scheduler.py、incident.py
│   │   └── main.py            # 应用入口，启动时拉起调度器
│   ├── alembic/
│   └── tests/
├── scripts/                   # seed_targets.py 初始化示例目标
├── docs/                      # 架构说明、截图、开发笔记
├── docker-compose.yml
└── README.md
```

`backend/app/services/` 是业务核心：`prober.py` 负责单次探测与错误分类，`scheduler.py` 负责扫描到期目标并并发派发，`incident.py` 负责防抖判定。把它们与路由层拆开，一是好测试，二是让「业务逻辑」与「接口」解耦。

---

## 数据库设计

### `targets`（监控目标）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | serial PK | 主键 |
| `name` | text not null | 显示名称 |
| `url` | text not null | 目标地址，**唯一约束**防重复添加 |
| `interval_sec` | int not null default 300 | 探测间隔（60–3600） |
| `timeout_ms` | int not null default 10000 | 单次请求超时 |
| `expected_status` | int default 200 | 期望状态码；也接受 2xx 区间 |
| `keyword` | text null | 响应体需包含的关键字，空则跳过校验 |
| `enabled` | bool default true | 是否参与探测 |
| `last_checked_at` | timestamptz null | 最近一次探测时间 |
| `next_check_at` | timestamptz not null | 下次应探测时间，调度器据此筛选 |
| `created_at` | timestamptz default now() | 创建时间 |

### `checks`（探测记录，时序表）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | bigserial PK | 主键 |
| `target_id` | int FK → `targets` | 所属目标 |
| `checked_at` | timestamptz not null | 探测时刻，统一存 UTC |
| `ok` | bool not null | 是否判定成功 |
| `status_code` | int null | HTTP 状态码，请求失败时为空 |
| `latency_ms` | int null | 响应耗时，失败时为空 |
| `error_kind` | text null | 错误分类（见下） |
| `error_msg` | text null | 原始错误信息，截断保存 |

索引：`(target_id, checked_at DESC)`——延迟曲线与可用率计算的性能关键。

### `incidents`（事故）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | serial PK | 主键 |
| `target_id` | int FK → `targets` | 所属目标 |
| `started_at` | timestamptz not null | 事故开始（第 N 次连续失败的那次） |
| `ended_at` | timestamptz null | 恢复时间，未恢复为空 |
| `cause` | text null | 代表性错误类型 |
| `failed_checks` | int | 期间失败次数 |

索引：`(target_id, started_at DESC)`。

**两个约定**：

- **时间一律用 `timestamptz` 存 UTC**，前端按本地时区渲染。
- **探测记录有保留策略**：超过 180 天由定时任务清理，避免把免费额度写满。

### 错误分类（固定七类）

`dns` · `connect`（连接被拒/超时）· `tls` · `http_4xx` · `http_5xx` · `keyword_missing` · `other`

分类的意义：前端能显示「超时」而不只是「失败」，事故原因也可归因。

---

## 接口设计

| 方法 | 路径 | 用途 | 鉴权 |
| --- | --- | --- | --- |
| `POST` | `/api/targets` | 新增监控目标 | 管理口令 |
| `GET` | `/api/targets` | 目标列表，含当前状态摘要 | 无 |
| `PATCH` | `/api/targets/{id}` | 修改间隔、超时、关键字、启停 | 管理口令 |
| `DELETE` | `/api/targets/{id}` | 删除目标及其记录 | 管理口令 |
| `POST` | `/api/targets/{id}/check` | 立即探测一次 | 管理口令 |
| `GET` | `/api/checks` | 按目标与时间段查探测记录 | 无 |
| `GET` | `/api/uptime` | 可用率、P95、平均延迟 | 无 |
| `GET` | `/api/incidents` | 事故列表 | 无 |
| `GET` | `/api/stream` | SSE 实时事件流 | 无 |
| `POST` | `/api/cron/run` | 外部触发一轮探测 | 管理口令 |
| `GET` | `/health` | 健康检查，供保活使用 | 无 |

管理口令通过请求头 `X-Admin-Token` 传递，与后端环境变量 `ADMIN_TOKEN` 比对。

**新增目标**

```http
POST /api/targets
Content-Type: application/json
X-Admin-Token: <ADMIN_TOKEN>

{
  "name": "学校教务系统",
  "url": "https://jw.example.edu.cn",
  "interval_sec": 300,
  "timeout_ms": 10000,
  "expected_status": 200,
  "keyword": "登录"
}
```

**可用率统计**

```http
GET /api/uptime?target_id=1&window=24h
```

```json
{
  "target_id": 1,
  "window": "24h",
  "total": 288,
  "ok": 286,
  "uptime_ratio": 0.9931,
  "p95_latency_ms": 420,
  "avg_latency_ms": 268
}
```

**SSE 事件**

`GET /api/stream` 每条事件为一行 `data:`，负载形如：

```json
{
  "type": "check",
  "target_id": 1,
  "checked_at": "2025-01-01T08:00:00Z",
  "ok": false,
  "status_code": null,
  "latency_ms": null,
  "error_kind": "timeout",
  "incident": "opened"
}
```

> 流式端点必须带 `X-Accel-Buffering: no`，且压缩中间件不能作用于 `/api/stream`，否则反向代理会缓冲响应，导致「本地能推、线上不推」。

---

## 核心业务规则

1. **成功的判定**：连接建立成功；状态码落在期望区间（默认 2xx）；耗时未超时；若配置了关键字，响应体包含该关键字。任一不满足即失败。
2. **七类错误码**：见上表，前端据此展示「超时 / DNS 失败 / 5xx」等具体原因。
3. **告警防抖**：连续 **3** 次失败判为故障并建事故；事故期间不重复建；连续 **2** 次成功才关事故。阈值可配置（`INCIDENT_FAIL_THRESHOLD` / `INCIDENT_RECOVER_THRESHOLD`），默认值即上文。目的是不让一次网络抖动就报故障。
4. **可用率**：窗口内成功次数 / 总探测次数。v1 要求同一目标间隔固定，因此按次数比即可；若将来支持间隔变化，需改为按时间加权——这是**已知简化**。
5. **P95 延迟**：不自行排序，直接用 PostgreSQL 的 `percentile_cont(0.95) within group (order by latency_ms)`。
6. **在线状态**：最近一次探测成功、且距今不超过两个探测间隔 → 在线；否则异常或未知。不存布尔字段，由数据推导。

---

## 本地开发

### 方式一：Docker Compose（推荐）

```bash
git clone https://github.com/Tyx123456789/UptimePulse.git
cd UptimePulse
cp .env.example .env          # 按需修改 ADMIN_TOKEN、数据库口令
docker compose up --build
```

- 前端：http://localhost:5173
- 后端：http://localhost:8000 （文档 `/docs`，健康检查 `/health`）
- 数据库：localhost:5432

首次启动后执行迁移并写入示例目标：

```bash
docker compose exec backend alembic upgrade head
docker compose exec backend python ../scripts/seed_targets.py
```

### 方式二：前后端分别启动

```bash
# 1) 数据库（也可用本机已装的 PostgreSQL）
docker compose up -d db

# 2) 后端
cd backend
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp ../.env.example ../.env                          # 填 DATABASE_URL / ADMIN_TOKEN
alembic upgrade head
uvicorn app.main:app --reload --port 8000

# 3) 前端
cd frontend
npm install
npm run dev
```

前端通过 `VITE_API_BASE_URL`（默认 `http://localhost:8000`）访问后端。

### 常用命令

| 位置 | 命令 | 作用 |
| --- | --- | --- |
| `backend/` | `ruff check .` | lint |
| `backend/` | `pytest -q` | 单元测试 |
| `backend/` | `alembic revision --autogenerate -m "msg"` | 生成迁移 |
| `frontend/` | `npm run lint` | lint |
| `frontend/` | `npm run build` | 生产构建 |

---

## 环境变量

### 后端（`backend`）

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DATABASE_URL` | — | PostgreSQL 连接串，如 `postgresql+asyncpg://user:pass@host:5432/uptime` |
| `ADMIN_TOKEN` | — | 管理口令，写接口校验用（`X-Admin-Token`） |
| `CORS_ORIGINS` | `http://localhost:5173` | 允许的前端域名，逗号分隔 |
| `SCHEDULER_ENABLED` | `true` | 是否在本进程启动调度器 |
| `SCHEDULER_TICK_SEC` | `60` | 调度器扫描周期 |
| `INCIDENT_FAIL_THRESHOLD` | `3` | 连续失败多少次开事故 |
| `INCIDENT_RECOVER_THRESHOLD` | `2` | 连续成功多少次关事故 |
| `DATA_RETENTION_DAYS` | `180` | 探测记录保留天数 |
| `PROBE_CONCURRENCY` | `20` | 一轮探测的最大并发数 |

### 前端（`frontend`）

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `VITE_API_BASE_URL` | `http://localhost:8000` | 后端地址；生产环境填 Render 域名 |

> `.env` 不入库；仓库只提交 `.env.example`。

---

## 部署

三件套：**Supabase** 提供 PostgreSQL，**Render** 跑 FastAPI，**Vercel** 托管 Vue 静态站。

1. **数据库**：在 Supabase 新建项目，取连接串填入 Render 的 `DATABASE_URL`（注意用 pooler 主机名，免费实例连接数有限）。
2. **后端（Render）**：新建 Web Service，Root Directory `backend`，启动命令 `uvicorn app.main:app --host 0.0.0.0 --port $PORT`，Build Command `pip install -r requirements.txt && alembic upgrade head`。配置上表的后端环境变量。
3. **前端（Vercel）**：Import 仓库，Root Directory `frontend`，Framework 选 Vite，环境变量 `VITE_API_BASE_URL` 指向 Render 域名。构建完成后把 Vercel 域名填回后端的 `CORS_ORIGINS`。
4. **保活 + 补探测**：GitHub Actions 每 15 分钟调一次 `POST /api/cron/run`（带 `X-Admin-Token`）。密钥放仓库 Secrets：`BACKEND_URL`、`ADMIN_TOKEN`。
5. **CI**：`lint`（ruff + eslint）→ `test`（pytest）→ `build`（`npm run build`）三项全绿才算通过。

> 也可以把整套服务跑在任意容器平台上（仓库自带 `docker-compose.yml`），只要 `DATABASE_URL` 可达、`/api/cron/run` 能被外部定时器访问即可。

---

## 测试

至少覆盖三块：

1. **探测判定**（`prober.py`）：2xx 成功、非期望状态码失败、超时、关键字缺失、错误分类映射。
2. **防抖**（`incident.py`）：连续 N 次失败开事故、事故期间不重复开、连续 M 次成功关闭并算出持续时长。
3. **可用率**（`/api/uptime`）：把手工算出的 `ok/total` 与接口返回的 `uptime_ratio`、P95 对齐。

```bash
cd backend && pytest -q
```

---

## 六周开发计划

| 阶段 | 目标 | 交付物 | 验收标准 |
| --- | --- | --- | --- |
| W1 | 骨架上线 | Vue 空页面 + FastAPI `/health` | 前后端各有公网地址 |
| W2 | 打通一条链路 | 三张表 + 迁移 + 新增目标接口 + 手动探测接口 | 点一次按钮，`checks` 出现一条真实记录，页面可见 |
| W3 | 看板成型 | 目标列表、状态徽标、延迟折线、SSE 推送 | 手动探测后看板 3 秒内更新 |
| W4 | 调度与判定 | 自动探测、并发、错误分类、防抖事故 | 指向不可达地址，自动产生事故并恢复关闭 |
| W5 | 工程化 | Docker Compose、GitHub Actions、日志、测试 | CI 全绿，≥3 个测试 |
| W6 | 交付 | README、截图与 GIF、架构说明、复盘 | 陌生人照 README 能在本地跑起来 |

W1 就上线是刻意的：部署踩坑越早遇到成本越低；拖到最后，环境变量、跨域、时区会一起来。

---

## 关键技术难点与解法

1. **SSE 在免费 PaaS 上不推送**：本地正常、线上无事件。原因是反向代理缓冲了响应。解法：响应头 `X-Accel-Buffering: no`，并确保压缩中间件不作用于流式端点。
2. **并发探测拖垮一轮任务**：串行请求 20 个目标可能耗时数十秒。解法：`asyncio.gather` 并发、每请求独立超时、`return_exceptions=True` 防止单个异常打断整轮。
3. **误报**：单次偶发失败就报警会让人不再信任看板。解法：连续 N 次失败才开事故、连续 M 次成功才关，阈值可配。
4. **时序查询变慢**：免费实例 CPU 弱，数据量上来后全表扫描很明显。解法：复合索引、强制按时间范围查询、长区间走小时级聚合而非逐条返回。
5. **免费实例休眠**：闲置后调度器停摆、首访 30 秒冷启动。解法：GitHub Actions 定时调 `/api/cron/run` 保活兼补探测，前端用骨架屏兜住冷启动。
6. **监控系统不能监控自己**：后端若探测自身，服务挂掉时既无数据也无告警。解法：监控目标选外部地址，或前后端分两处互探（见「已知限制」）。

---

## 已知限制

- **无多用户**：v1 用单一管理口令（`ADMIN_TOKEN`），没有注册、登录与角色权限。
- **可用率按次数比**：前提是同目标探测间隔固定；间隔变化时未做时间加权。
- **单点探测**：只有一处探测节点，无法区分「目标挂了」还是「我这边的网络到不了目标」。
- **不监控自身**：平台本身不可用时不会产生告警，需要外部（如 GitHub Actions 失败通知）兜底。
- **免费实例休眠**：休眠期间调度器不运行，探测间隔靠外部触发补齐，时间线可能出现空档。
- **无通知渠道**：v1 只在页面看，不发邮件/短信/webhook。

---

## 不做什么

写下「不做什么」比写下「做什么」更重要——它是控制工期的工具。

- 用户注册、登录、多角色权限（v1 单一管理口令）
- 秒级探测、分布式多节点探测、大规模目标池
- 邮件或短信通知（P2 再说）
- 移动 App
- 微服务拆分、消息队列、缓存中间件

---

## 可延展方向

数据层稳定后，可以在同一架构上做垂直增强，不需要重构：

- 多用户与权限体系
- 区域多节点探测（对比不同地区访问质量）
- 通知渠道（邮件、webhook、企微机器人）
- 指标下钻（DNS 耗时、TLS 握手耗时拆分）

---

## 许可证

[MIT](LICENSE)

---

<p align="center">用真实数据说话 —— 每条曲线都来自一次真实的网络请求。</p>