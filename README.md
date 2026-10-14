# 冷链物流温控与合规平台（cold-chain-platform）

![Java](https://img.shields.io/badge/Java-17-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-2.7.18-brightgreen)
![MyBatis-Plus](https://img.shields.io/badge/MyBatis--Plus-3.5.5-blue)
![Vue](https://img.shields.io/badge/Vue-3.5-42b883)
![Vite](https://img.shields.io/badge/Vite-5.4-646CFF)
![MySQL](https://img.shields.io/badge/MySQL-8.0.33-4479A1)
![Redis](https://img.shields.io/badge/Redis-7.x-DC382D)
![License](https://img.shields.io/badge/License-See%20below-lightgrey)

> 独立立项、独立交付的冷链温控与追溯系统（V1.0 单体模块化）。技术栈与版本以 `pom.xml`、
> `web/package.json` 与 `docs/01_系统架构与技术方案.md` 为准。

---

## 项目简介

本平台面向**食品、生鲜、医药**行业，提供「入库感知 → 库内温控 → 在途监控 → 交付签收 → 全链路追溯」
闭环，用**可采信的温度证据链**支撑货损责任界定与合规检查。

系统为**独立私有化部署**，不依赖、不集成园区物流云或其它系统，可部署于客户机房 / 政务云。
（依据 `docs/01_系统架构与技术方案.md` §1.1、§7）

核心设计取向（详见 `docs/04_温控证据链与合规设计（ADR）.md`）：

- **温度记录只增不改**：`cc_temp_record` 仅 INSERT/SELECT，数据库账号层 `REVOKE UPDATE, DELETE`。
- **哈希链防篡改**：`hash = SHA256(hash_prev + '|' + device_id + '|' + temp + '|' + device_ts + '|' + signature)`，每设备独立链。
- **断链不可证**：断链期间不写任何记录，报告显式标注"该时段温度不可证"，**严禁估算/插值**。
- **设备签名双源可信**：设备私钥签名（Ed25519 / SM2 可配置），平台验签。
- **校准台账**：无有效校准或精度不足的设备数据标"不可举证"，不参与合规举证。

---

## Features（按模块）

业务分包前缀 `com.coldchain.module.`（见 `docs/01_系统架构与技术方案.md` §4）：

| 模块 | 能力 |
|---|---|
| `sys` | 认证、用户、角色、权限、只追加审计日志；登录失败锁定（5 次 / 15 分钟） |
| `tenant` | 多租户隔离（业务表带 `tenant_id`，跨租户拒绝 `TENANT_MISMATCH`） |
| `master` | 企业 / 冷藏车 / 驾驶员 / 证照台账 / 温控设备与校准台账（证照过期派单熔断） |
| `warehouse` | 冷库 / 温区 / 库位 / 品类温区阈值 / 库存（批次级）/ 出入库 / 效期 / FIFO / 盘点 |
| `temp` | 温度记录哈希链、温度曲线、哈希链校验、断链、取证导出包 |
| `alarm` | 四级告警分级矩阵（L1~L4）、确认 / 处置 / 升级 / 闭环 |
| `trace` | 一物一码赋码、追溯链（产地→入库→装车→发车→到达→交付）、GS1 映射、公开查询（脱敏） |
| `order` | 冷链运输订单与状态机、装车绑定温控设备、电子签收、订单节点留痕 |
| `track` | GPS / 北斗定位接入、实时位置、地理围栏（含温度围栏 `TEMP`、禁行围栏 `NOWAY`） |
| `ai` | 销量预测（内置基线 `MOVING_AVG_TREND`）、库容与能耗优化建议 |
| `data` | 指标字典 / 指标值、合规报告生成（日报 / 月报 / GSP / 货损举证） |
| `cold-iot-gateway` | 独立 IoT 接入网关（Netty，多协议、设备验签、断链检测、读数上报） |

---

## 系统架构

以独立仓库、独立数据库（`coldchain`）、独立中间件集群方式运行（依据 `docs/01_系统架构与技术方案.md` §2）：

```
┌────────────────────────────────────────────────────────────┐
│ 端侧：质量/合规 Web · 冷库管理 Web · 调度 Web · 收货方查询页  │
│       司机端小程序（装车确认 / 断链上报 / 到货签收）           │
└───────────────────────────┬────────────────────────────────┘
                            │ HTTPS  Header: cold-token
┌───────────────────────────▼────────────────────────────────┐
│ cold-app（Spring Boot 2.7.18，端口 8080）                    │
│ Sa-Token 鉴权 → 租户隔离 → 全局异常 → 审计切面               │
│ 业务模块：sys/tenant/master/warehouse/temp/alarm/trace/     │
│           order/track/ai/data                               │
└───────┬───────────────────┬───────────────────┬────────────┘
        │                   │                   │
┌───────▼──────┐  ┌─────────▼────────┐  ┌───────▼────────────┐
│ MySQL 8.0.33 │  │ TDengine 3.x     │  │ cold-iot-gateway   │
│ 业务+哈希链  │  │ 温度高频时序     │  │ 8090(HTTP)/9001(TCP)│
│ +审计        │  │ (10s 采样)       │  │ Netty + 验签 + 断链 │
└──────────────┘  └──────────────────┘  └───────┬────────────┘
        │                                        │ 上报读数/断链
┌───────▼──────┐  ┌──────────────┐  ┌────────────▼───────────┐
│ Redis 7.x    │  │ RocketMQ 5.x │  │ MinIO（报告/证书/归档） │
│ 令牌/在线态  │  │ 事件总线     │  │ 可选：本地磁盘降级      │
└──────────────┘  └──────────────┘  └────────────────────────┘
```

---

## 技术栈

| 层 | 选型 | 版本 | 来源 |
|---|---|---|---|
| 语言 / 运行时 | Java | 17 | `pom.xml`（`java.version=17`） |
| 后端框架 | Spring Boot | 2.7.18 | `pom.xml` |
| ORM | MyBatis-Plus | 3.5.5 | `pom.xml` |
| 鉴权 | Sa-Token | 1.37.0 | `pom.xml` |
| 工具库 | Hutool | 5.8.25 | `pom.xml` |
| 接口文档 | springdoc-openapi | 1.6.15 | `pom.xml` |
| 表格导出 | EasyExcel | 3.3.4 | `pom.xml` |
| 对象存储 | MinIO SDK | 8.5.7 | `pom.xml` |
| IoT 通信 | Netty | 4.1.100.Final | `pom.xml` |
| 调度 | XXL-Job | 2.4.1（默认关闭） | `pom.xml` |
| 消息 | RocketMQ | 2.2.3（starter） | `pom.xml` |
| 时序库驱动 | TDengine JDBC | 3.2.9 | `pom.xml` |
| 国密 / 签名 | Bouncy Castle | 1.77 | `pom.xml` |
| 覆盖率 | JaCoCo | 0.8.11（行覆盖率门禁 0.60） | `pom.xml` |
| 前端 | Vue 3 + Vite | 3.5.11 / 5.4.8 | `web/package.json` |
| 前端 UI | Element Plus | 2.8.4 | `web/package.json` |
| 图表 | ECharts | 5.5.1 | `web/package.json` |
| 状态管理 / 路由 | Pinia 2.2.4 / Vue Router 4.4.5 | — | `web/package.json` |
| 可选 AI 预测服务 | FastAPI + Uvicorn + statsmodels + numpy | Python 3.10+ | `ai-forecast-service/requirements.txt` |

> 数据库：MySQL `coldchain`（utf8mb4，InnoDB）；温度高频时序可另存 TDengine 3.x。
> 详见 `docs/02_数据库设计说明书.md`。

---

## Quick Start

> 面向新手的完整图文步骤见 [`docs/环境搭建说明.md`](docs/环境搭建说明.md)；跨平台差异（macOS/Linux）在该文档中以备注给出。

### 1. 环境要求

| 软件 | 版本 | 必需 |
|---|---|---|
| JDK | 17 | ✅ |
| Maven | 3.8+ | ✅ |
| MySQL | 8.x | ✅ |
| Redis | 7.x（6+ 亦可） | ✅ |
| Node.js | 18+（含 npm） | ✅ |
| Git | 任意近期版本 | ✅ |
| Python | 3.10+ | ⭕（仅可选 AI 预测服务） |
| Docker / Compose | 可选 | ⭕ |

> 本机默认 profile 为 `local`（`event-bus=local`、`storage=local`、`position-store=mysql`）：
> **只装 MySQL + Redis 即可完整跑起后端**，MinIO / TDengine / RocketMQ / AI 服务本机可先不启动。

### 2. 建库与 SQL 执行顺序（重要）

库名固定 `coldchain`。`schema.sql` **不含 `CREATE DATABASE`**，需手工建库：

```sql
CREATE DATABASE IF NOT EXISTS `coldchain`
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_0900_ai_ci;
```

按序执行（顺序很重要）：

```powershell
# 1) 建表
mysql -h 127.0.0.1 -P 3316 -u county -p coldchain < sql\schema.sql
# 2) 种子数据（admin/quality 账号、角色、品类阈值、告警矩阵；幂等）
mysql -h 127.0.0.1 -P 3316 -u county -p coldchain < sql\seed.sql
# 3) （生产强烈建议）只追加表授权：对 8 张只追加表 REVOKE UPDATE, DELETE
mysql -h 127.0.0.1 -P 3316 -u county -p coldchain < sql\grant_append_only.sql
```

> 端口请与 `cold-app` 配置一致：本机 `local` profile 为 **3316**，生产为 `3306`。
> 只追加表包括 `cc_temp_record`、`cc_inventory_txn`、`cc_sys_audit_log`、`cc_alarm_action`、
> `cc_trace_link`、`cc_vehicle_position`、`cc_order_node`、`cc_report`。

### 3. 启动后端（cold-app）

```powershell
cd <项目根目录>
# 首次先构建公共模块（-am 连带构建 cold-common）
mvn -q -pl cold-app -am clean package -DskipTests
# 启动（二选一）
mvn -pl cold-app -am spring-boot:run
# 或：java -jar cold-app\target\cold-app-1.0.0.jar
```

- 启动类：`com.coldchain.ColdChainApplication`；默认端口 **8080**，默认 profile `local`。
- 健康检查：`curl http://127.0.0.1:8080/actuator/health` → `{"status":"UP",...}`
- 接口文档（非 prod）：`http://127.0.0.1:8080/doc.html`

### 4. 启动前端（web）

```powershell
cd web
npm install
npm run dev
```

- 开发端口 **5173**（`vite.config.js`），访问 `http://127.0.0.1:5173`。
- 开发环境通过 Vite 代理将 `/api` 转发到 `http://127.0.0.1:8080`，后端在 8080 即无需额外配置。

### 5. 默认账号（以 `sql/seed.sql` 为准）

| 用户名 | 密码 | 角色 |
|---|---|---|
| `admin` | `admin123` | 平台管理员（全量权限） |
| `quality` | `admin123` | 质量合规负责人 |

> 仅用于本地验证，生产环境请务必修改（登录失败 5 次锁定 15 分钟）。

### 6. 启动可选 AI 预测服务（Python，默认不启用）

```powershell
cd ai-forecast-service
pip install -r requirements.txt
uvicorn app:app --host 0.0.0.0 --port 8500
# 或：docker build -t cold-ai-forecast:1.0.0 . && docker run -p 8500:8500 cold-ai-forecast:1.0.0
```

主后端通过 `cold.ai.remote-enabled=true` + `cold.ai.predict-endpoint` 指向该服务；**调用失败会自动降级为内置基线，
不报错、不阻断业务**。详见 [`docs/07_可选外部AI预测服务说明.md`](docs/07_可选外部AI预测服务说明.md)。

### 7. 启动 IoT 网关（可选，无真实传感器时可不启动）

```powershell
mvn -pl cold-iot-gateway -am spring-boot:run
# 或：java -jar cold-iot-gateway\target\cold-iot-gateway-1.0.0.jar
```

- HTTP **8090** / 传感器 TCP **9001**（`cold.iot.port` / `cold.iot.tcp-port`）。

---

## 配置说明

配置位置：`cold-app/src/main/resources/application.yml`（主）、`application-local.yml`（本机）、
`application-prod.yml`（生产，全部走环境变量）；`cold-iot-gateway` 使用独立 `application.yml`。

**关键配置键（部分，完整见 `docs/环境搭建说明.md` §7 / §15.2）：**

| 配置键 | 默认 | 说明 |
|---|---|---|
| `server.port` | `8080` | 后端端口 |
| `cold.crypto.secret` | `cold-chain-default-aes-key-0001` | 敏感数据加密密钥，**生产必须用环境变量/KMS 注入** |
| `cold.login.max-fail-count` / `lock-minutes` | `5` / `15` | 登录失败锁定策略 |
| `cold.temp.gap-threshold-seconds` | `120` | 设备静默 >2 分钟判定断链 |
| `cold.alarm.*` | L1 接近 1℃ / L3 越限 30 分钟 / L4 断链 120 分钟 | 告警分级矩阵 |
| `cold.device.sign-algorithm` | `ed25519` | 设备签名算法（可选 `sm2`，需密评确认） |
| `cold.ai.remote-enabled` | `false` | 外部 AI 预测开关 |
| `cold.ai.predict-endpoint` | 空 | 外部 AI 预测地址（如 `http://127.0.0.1:8500/predict`） |
| `cold.event-bus.type` | `local`（prod 默认 `rocketmq`） | 事件总线类型 |
| `cold.storage.type` | `local`（prod 默认 `minio`） | 存储类型 |
| `cold.position-store.type` | `mysql`（prod 默认 `tdengine`） | 定位存储类型 |
| `cold.retention.food-years` / `medicine-years` | `3` / `5` | 食品 / 医药温控数据留存年限 |
| `sa-token.token-name` | `cold-token` | 登录令牌请求头名 |
| `minio.endpoint` / `access-key` / `secret-key` / `bucket` | `http://127.0.0.1:9000` / `minioadmin` / `minioadmin` / `cold-chain` | 对象存储 |

> 覆盖优先级：命令行参数 > 环境变量 > `application-{profile}.yml` > `application.yml`。
> 生产环境请勿使用默认密钥与默认密码。

---

## API 概览

- 统一返回结构：`{ "code": 0, "msg": "成功", "data": { } }`（`code=0` 成功）。
- 认证：登录令牌，请求头 **`cold-token`**；`timeout=43200s`、`active-timeout=1800s`。
- 免登录白名单：`/api/auth/login`、`/api/public/**`、`/api/openapi/**`、`/doc.html`、`/actuator/health` 等。
- 温度记录接口**只读**（仅查询 / 校验 / 导出，无编辑 / 删除接口）。

**接口模块一览：**

| 前缀 | 说明 |
|---|---|
| `/api/auth` | 登录 / 登出 / 当前用户 |
| `/api/master` | 企业 / 车辆 / 驾驶员 / 证照 / 温控设备与校准 |
| `/api/warehouse` | 冷库 / 温区 / 库位 / 品类阈值 / 库存与流水 / 盘点 |
| `/api/temp` | 温度记录 / 曲线 / 哈希链校验 / 断链 / 取证导出 |
| `/api/order` | 订单、状态流转、装车绑定设备、电子签收、节点 |
| `/api/track` | 定位 / 看板 / 围栏（含温度围栏） |
| `/api/alarms` | 告警列表 / 详情 / 确认 / 处置 / 闭环 / 分级矩阵 |
| `/api/trace`、`/api/public/trace` | 追溯赋码 / 查链 / 映射；公开查询（脱敏） |
| `/api/report` | 报告生成 / 台账列表 |
| `/api/ai` | 销量预测 / 能耗建议 |
| `/api/file` | 文件上传 / 下载 |
| `/api/openapi/iot` | 开放 IoT 接入（免登录，依赖设备签名） |

> 完整路径、入参、出参与错误码（含 `TEMP_RECORD_IMMUTABLE`、`TEMP_CHAIN_BROKEN`、`LINK_GAP_PRESENT`、
> `DEVICE_BIND_REQUIRED` 等）见 **[`docs/03_对外API文档.md`](docs/03_对外API文档.md)**。

---

## 部署

> **当前仓库状态（据实说明）**：截至本文编写，仓库中**不存在 `deploy/` 目录、`docker-compose.yml`、
> `.env.example` 或 Nginx 配置**；全仓库仅 `ai-forecast-service/Dockerfile` 一个 Dockerfile。
> 一键部署脚本在 `docs/05_部署与运维手册.md` 中标记为 **⬜ 未开始（待 P1-T01）**，其 `docker-compose.yml`
> 为**占位示意**。

当前可通过**手工方式**搭建：分别启动 MySQL（本机 3316）、Redis（本机 6390）、后端（8080）、
前端（`npm run build` 后由 Nginx 托管 `dist/`）。

**部署形态（规划，见 `docs/01_系统架构与技术方案.md` §6、`docs/05_部署与运维手册.md` §2）：**

| 形态 | 适用 |
|---|---|
| 单机版（Docker Compose） | 试点客户（≤10 冷库 / ≤100 冷藏车），1 台服务器（建议 8C16G 起） |
| 集群版 | 多冷库 / 多车队推广，应用无状态水平扩展 |
| 私有化 | 客户内网机房 / 政务云，按等保与网络要求部署 |

上线前必做项（含只追加表授权、密钥注入、阈值签字等）见
**[`docs/05_部署与运维手册.md`](docs/05_部署与运维手册.md) §7 上线检查清单**。

---

## 目录结构

```
cold-chain-platform/
├── pom.xml                     # Maven 聚合父 POM（Spring Boot 2.7.18 / JDK 17 / 版本锁定）
├── cold-common/                # 公共库：统一返回 R、异常、租户上下文、加密、哈希链、脱敏
├── cold-app/                   # 后端主应用
│   └── src/main/resources/
│       ├── application.yml         # 主配置
│       ├── application-local.yml   # 本机开发（DB 3316 / Redis 6390）
│       └── application-prod.yml    # 生产（环境变量注入）
├── cold-iot-gateway/           # IoT 接入网关（Netty，HTTP 8090 / TCP 9001）
├── web/                        # 管理端前端（Vue3 + Vite + Element Plus + Pinia）
├── driver-mini/                # 司机端小程序（uni-app，非 Maven 模块，按需）
├── ai-forecast-service/        # 可选 Python AI 预测服务（FastAPI，默认不启用）
├── sql/
│   ├── schema.sql              # 建表（utf8mb4，约 30+ 张业务表）
│   ├── seed.sql                # 种子数据（admin/quality、角色、品类阈值、告警矩阵）
│   └── grant_append_only.sql   # 只追加表 REVOKE 授权
└── docs/                       # 项目文档（01~08）
```

> 注意：`pom.xml` 的 `<modules>` 仅包含 `cold-common`、`cold-app`、`cold-iot-gateway` 三个 Maven 模块；
> `web/` 与 `driver-mini/` 是独立前端工程，不走 Maven 构建。

---

## 文档索引

| 文档 | 内容 |
|---|---|
| [`docs/环境搭建说明.md`](docs/环境搭建说明.md) | 从零搭建、启动、端口、默认账号（面向新手） |
| [`docs/功能使用说明.md`](docs/功能使用说明.md) | 管理端逐页操作说明、角色权限、端到端场景 |
| [`docs/01_系统架构与技术方案.md`](docs/01_系统架构与技术方案.md) | 系统定位、总体架构、技术选型、模块划分 |
| [`docs/02_数据库设计说明书.md`](docs/02_数据库设计说明书.md) | 表结构、字段、只追加表约定 |
| [`docs/03_对外API文档.md`](docs/03_对外API文档.md) | 对外 REST 接口、角色、错误码 |
| [`docs/04_温控证据链与合规设计（ADR）.md`](docs/04_温控证据链与合规设计（ADR）.md) | 哈希链、设备签名、断链不可证、校准台账 |
| [`docs/05_部署与运维手册.md`](docs/05_部署与运维手册.md) | 部署形态、监控、备份、上线检查清单 |
| [`docs/06_等保2.0三级差距分析.md`](docs/06_等保2.0三级差距分析.md) | 等保三级差距分析 |
| [`docs/07_可选外部AI预测服务说明.md`](docs/07_可选外部AI预测服务说明.md) | 可选外部 AI 预测服务 |
| [`docs/P0-T02_SRS需求规格说明书.md`](docs/P0-T02_SRS需求规格说明书.md) | 需求规格（140 条） |
| [`docs/P0-T04_品类温区阈值表.md`](docs/P0-T04_品类温区阈值表.md) | 品类温区阈值（基线，待合规顾问签字） |
| [`docs/复盘问题清单与修复计划.md`](docs/复盘问题清单与修复计划.md) | 复盘审计与修复记录 |
| [`ai-forecast-service/README.md`](ai-forecast-service/README.md) | 可选 AI 预测服务启动与契约 |

---

## Roadmap

> 以下为仓库文档中**已登记的规划 / 待办项**，非承诺时间表；状态以相关文档为准。

- [ ] **一键部署脚本与 CI**：Docker Compose 一键部署、CI（单测 + lint）——文档标注 ⬜ 未开始（`docs/05_部署与运维手册.md` §8，P1-T01）。
- [ ] **集群版 / 私有化部署**：应用无状态水平扩展，MQ / 缓存 / 时序库独立集群（`docs/01_系统架构与技术方案.md` §6）。
- [ ] **单机容量压测验证**：8C16G 在 V1 规模下压测（P1-T20，`docs/01_系统架构与技术方案.md` §8）。
- [ ] **AI 能力扩展**：设备故障预测（V3.1）、温控异常预测（V3.2）——本期不做，已登记变更（`docs/04_温控证据链与合规设计（ADR）.md` ADR-005）。
- [ ] **监管平台上报对接**：外部平台未接入时按设计返回 `NOT_IMPLEMENTED`（P3-T05，依赖外部）。

---

## Contributing

欢迎提交 Issue 与 Pull Request。为保证证据链与合规红线不被破坏，请遵守以下约定：

1. **不得绕过只追加约束**：`cc_temp_record` 等只追加表**禁止**新增 UPDATE/DELETE 逻辑或接口。
2. **不得对断链时段做估算 / 插值**：这是合规红线。
3. **接口改动三处对齐**：后端 Controller、前端 `web/src/api/modules/*.js`、`docs/03_对外API文档.md` 需同步更新。
4. **签名口径统一**：设备签名被签载荷与编码口径须在网关、后端、契约文档三处一致（见 `docs/开发约定与公共契约.md` §5）。
5. **测试与覆盖率**：提交前请确保 `mvn clean verify` 通过；核心算法类（温度哈希链、设备验签、告警判定引擎、身份鉴别核心）行覆盖率门禁为 **0.60**（`pom.xml`）。

---

## License

> 【占位】请替换：本项目许可证（当前仓库**未包含 `LICENSE` 文件**）。
> 建议在确定开源/商用授权方式后，补充 `LICENSE` 文件（如 MIT / Apache-2.0 / 商业授权），并在此说明。

---

## 联系方式

> 【占位】请替换：GitHub / 代码仓库地址。
> 【占位】请替换：官网或产品落地页。
> 【占位】请替换：联系邮箱。
> 【占位】请替换：企业微信 / 客服联系方式。

---

> **说明**：本文档中的功能、端口、版本、默认账号与配置键均引自本仓库现有文件（`pom.xml`、`web/package.json`、
> `sql/*.sql`、`docs/*.md` 等）并尽量标注来源；**未在仓库文档中出现的数据（如实测性能、客户案例、用户数）
> 一律未写入**。
---

## 🏢 关于我们

<p align="center">
  <a href="http://www.net188.net">
    <img src="http://www.net188.net/images/logo1.png" alt="Net188 Logo" width="200" />
  </a>
</p>

<p align="center">
  <strong>Net188 · 互联网技术服务</strong>
</p>

<p align="center">
  专注于跨平台应用开发、AI Agent 集成与大模型应用落地。<br/>
  提供从产品设计、开发实施到部署运维的全栈技术解决方案。
</p>

<p align="center">
  🌐 <a href="http://www.net188.net"><strong>www.net188.net</strong></a>
</p>

---
