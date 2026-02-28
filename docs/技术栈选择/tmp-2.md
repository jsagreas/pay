# ser-wallet 技术栈选择与分析

> 编写时间：2026-02-27
> 定位：完整梳理实现钱包模块需要用到的所有技术组件/中间件/库，区分"工程已有可复用"和"需新引入"，说明每项技术的用途和选择理由
> 分析原则：**已有的坚决复用，没有的才提出引入，不重复造轮子**
> 依据来源：
> - 工程代码深度分析：/Users/mac/gitlab 全工程 12 模块实际代码
> - 架构设计：/Users/mac/gitlab/z-readme/架构设计/tmp-2.md
> - 技术挑战：/Users/mac/gitlab/z-readme/技术挑战/tmp-2.md
> - 表结构设计：/Users/mac/gitlab/z-readme/表结构设计/tmp-2.md

---

## 一、技术栈总览

### 1.1 一句话结论

**钱包模块需要的核心技术能力，现有工程已覆盖 90% 以上。只需在现有基础上引入 1 个新库（shopspring/decimal 高精度运算），其余全部复用已有组件。** 这不是巧合——当初的技术选型（CloudWeGo + TiDB + Redis + NATS + ETCD）就是为了支撑全业务域的，钱包只是其中一个落地场景。

### 1.2 分类总览

```
┌────────────────────────────────────────────────────────────────────────┐
│              ser-wallet 技术栈全景图                                      │
│                                                                        │
│  ┌── 已有·直接复用（90%）──────────────────────────────────────────┐  │
│  │                                                                  │  │
│  │  RPC 框架      │  Kitex v0.16.0 + Thrift IDL                    │  │
│  │  HTTP 框架     │  Hertz v0.10.3（C端/B端网关已有）               │  │
│  │  ORM          │  GORM v1.31.1 + Gen v0.3.27                    │  │
│  │  数据库        │  TiDB（MySQL 兼容，行锁/事务/DECIMAL 支持）     │  │
│  │  缓存          │  Redis go-redis v9.17.2（UniversalClient）     │  │
│  │  配置中心      │  ETCD v3.6.7（LoadAndWatch 热更新）             │  │
│  │  消息队列      │  NATS/JetStream v1.48.0                        │  │
│  │  定时任务      │  robfig/cron v3.0.1（秒级精度）                 │  │
│  │  ID 生成      │  Snowflake（自研，common/pkg/utils）             │  │
│  │  日志          │  Zap v1.27.1 + Lumberjack 切割                 │  │
│  │  链路追踪      │  TraceId（UUID）+ Metainfo 上下文传播           │  │
│  │  服务发现      │  kitex-contrib/registry-etcd v0.3.0             │  │
│  │  审计插件      │  UserPlugin（GORM 回调自动填充审计字段）         │  │
│  │  代码生成      │  GORM Gen + gen_kitex.sh                       │  │
│  │  JSON 处理    │  bytedance/sonic（高性能）                      │  │
│  │  对象拷贝      │  jinzhu/copier v0.4.0                          │  │
│  │  错误处理      │  pkg/errors v0.9.1                             │  │
│  │                                                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌── 需新引入（仅 1 项）──────────────────────────────────────────┐  │
│  │                                                                  │  │
│  │  高精度运算    │  shopspring/decimal（金额/汇率精确计算）         │  │
│  │                                                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌── 已有·暂不需要（工程有但钱包用不到）────────────────────────┐  │
│  │                                                                  │  │
│  │  Elasticsearch │  钱包无全文搜索需求                             │  │
│  │  S3/AWS SDK    │  钱包无文件上传需求                             │  │
│  │  Kafka         │  NATS 已够用，无需两套 MQ                      │  │
│  │  XXL-JOB       │  robfig/cron 已够用                            │  │
│  │  Excel/CSV     │  初期无报表导出需求（可后续按需引入）            │  │
│  │                                                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 二、RPC 框架 — Kitex

### 2.1 已有情况

| 项目 | 信息 |
|------|------|
| **包路径** | `github.com/cloudwego/kitex v0.16.0` |
| **IDL 协议** | Apache Thrift |
| **代码生成** | `gen_kitex.sh` 脚本扫描 `common/idl/` 目录自动生成 |
| **服务发现** | `kitex-contrib/registry-etcd v0.3.0`，通过 ETCD 注册/解析 |
| **连接模式** | Mux 多路复用（`WithMuxConnection(1)`） |
| **超时** | RPC 超时 3 秒（`WithRPCTimeout(3 * time.Second)`） |
| **元数据传播** | TTHeader 协议（`transmeta.ClientTTHeaderHandler`），支持 TraceId/UserId 等上下文传递 |
| **现有客户端** | 20+ 个服务的 RPC Client 已注册在 `common/rpc/rpc_client.go` |
| **工厂模式** | `sync.Once` 单例 + 懒初始化 |

**文件位置**：
- 初始化参数：`/Users/mac/gitlab/common/rpc/init.go`
- 客户端工厂：`/Users/mac/gitlab/common/rpc/rpc_client.go`

### 2.2 钱包模块如何复用

```
复用清单：
  ① IDL 定义：在 common/idl/ser-wallet/ 下编写 Thrift IDL
  ② gen_kitex.sh：执行后自动生成 handler.go / main.go 骨架 + kitex_gen 代码
  ③ 服务端注册：复用 InitRpcServerParams()，传入 namesp.EtcdWalletService
  ④ 客户端工厂：在 rpc_client.go 中新增 WalletClient() 方法
  ⑤ 中间件日志：服务端自带 RPC 访问日志（含 TraceId / 方法名 / 耗时）
  ⑥ 元数据传播：自动传递 TraceId、UserId 等上下文信息

需要做的改动：
  · common/rpc/rpc_client.go 新增 WalletClient() — 供财务/游戏等模块调用我方 RPC
  · common/idl/ser-wallet/ 目录新建（当前不存在）
```

### 2.3 为什么用 Kitex

| 理由 | 说明 |
|------|------|
| **工程统一** | 全工程 20+ 服务均使用 Kitex，统一技术栈零学习成本 |
| **IDL 先行** | Thrift IDL 是模块间的"合同"，自动生成代码减少手写错误 |
| **高性能** | Mux 连接复用、Thrift 二进制序列化，性能优于 JSON HTTP |
| **服务治理** | 内置超时/重试/熔断能力，Kitex 官方文档已提供配置方式 |
| **元数据传播** | TTHeader 机制天然支持 TraceId 跨服务传递，审计需要 |

---

## 三、HTTP 框架 — Hertz

### 3.1 已有情况

| 项目 | 信息 |
|------|------|
| **包路径** | `github.com/cloudwego/hertz v0.10.3` |
| **网关服务** | gate-font（C 端）、gate-back（B 端）已存在 |
| **中间件链** | IP 黑白名单 → Token 鉴权 → RBAC → 路由分发 → RPC 代理 |
| **链路追踪** | IGlTracer 生成 UUID TraceId，注入请求上下文 |
| **统一响应** | `R{TraceId, Code, Msg, Data}` 格式 |
| **请求限制** | Body 最大 20MB |

**文件位置**：
- C 端网关：`/Users/mac/gitlab/gate-font/`
- B 端网关：`/Users/mac/gitlab/gate-back/`
- 中间件集合：`/Users/mac/gitlab/common/pkg/middleware/`
- 链路追踪：`/Users/mac/gitlab/common/pkg/tracer/tracer.go`

### 3.2 钱包模块如何复用

```
复用清单：
  ① gate-font 中新增钱包 C 端路由组：/api/wallet/xxx（需登录）+ /open/wallet/xxx（免登录）
  ② gate-back 中新增币种管理 B 端路由组：/admin/api/currency/xxx
  ③ 复用现有鉴权中间件（FontAuthMiddleware / BackAuthMiddleware）
  ④ 复用 IGlTracer 链路追踪（TraceId 自动生成和传播）
  ⑤ 复用 RBAC 权限校验（B 端币种管理需要角色权限控制）
  ⑥ 复用统一响应格式

需要做的改动：
  · gate-font/router 中注册钱包 C 端路由（POST 方法）
  · gate-back/router 中注册币种管理 B 端路由
  · 无需修改中间件代码
```

### 3.3 为什么用 Hertz

| 理由 | 说明 |
|------|------|
| **工程统一** | gate-font / gate-back 已基于 Hertz 搭建完毕 |
| **中间件生态** | 鉴权、黑白名单、RBAC、日志中间件已经完善 |
| **与 Kitex 一体** | 同属 CloudWeGo 体系，Hertz 收到 HTTP 请求后直接转发 RPC |
| **上下文传递** | IGlTracer 生成 TraceId 后通过 Metainfo 传递到 Kitex RPC |

---

## 四、数据库 — TiDB + GORM

### 4.1 已有情况

**TiDB（MySQL 兼容分布式数据库）**：

| 项目 | 信息 |
|------|------|
| **驱动** | `gorm.io/driver/mysql v1.6.0`（TiDB 完全兼容 MySQL 协议） |
| **连接管理** | `common/pkg/db/mysql/mysql.go`，MaxOpen=20, MaxIdle=10, MaxLifeTime=1h |
| **GORM 配置** | PrepareStmt=true（预编译缓存）, SkipDefaultTransaction=true（跳过隐式事务） |
| **GORM 插件** | UserPlugin — 自动填充 CreateBy/UpdateBy/CreateAt/UpdateAt |
| **GORM Gen** | `gorm.io/gen v0.3.27` — 自动生成 Model/Query/Repo 代码 |
| **Gen 配置** | `common/pkg/db/cmd/generate.go`，支持全库扫描或指定表名生成 |
| **Gen 选项** | FieldNullable=true（可空字段用指针）, FieldWithIndexTag=false（索引不在 Go Model 中） |
| **附加包** | gorm.io/plugin/dbresolver（读写分离）, gorm.io/datatypes, gorm.io/hints |
| **ETCD 配置** | TiDB 连接信息从 ETCD 读取（addr/port/username/password/dbname） |

**文件位置**：
- 连接初始化：`/Users/mac/gitlab/common/pkg/db/mysql/mysql.go`
- UserPlugin：`/Users/mac/gitlab/common/pkg/db/mysql/user_plugin.go`
- Gen 代码生成：`/Users/mac/gitlab/common/pkg/db/cmd/generate.go`
- Gen 仓储模板：`/Users/mac/gitlab/common/pkg/db/cmd/repo_template.go`
- ser-wallet Gen 入口：`/Users/mac/gitlab/ser-wallet/internal/gen/gorm_gen.go`（已有但引用了错误的 DB）

### 4.2 钱包模块如何复用

```
复用清单：
  ① 复用 MySQL 连接初始化（mysql.Init + ETCD 配置读取）
  ② 复用 UserPlugin（B 端操作自动填充审计字段）
  ③ 复用 GORM Gen 代码生成流程（建表 → gorm_gen.go 配置 → go run 生成）
  ④ 复用 GORM Transaction API（余额+流水同一事务）
  ⑤ 复用 PrepareStmt=true 和 SkipDefaultTransaction=true

需要做的改动：
  · ser-wallet/internal/gen/gorm_gen.go：修正 namesp.EtcdAppDb → namesp.EtcdWalletDB
  · common/pkg/consts/namesp/namesp.go：新增 EtcdWalletDB 常量
  · TiDB 中创建 ser_wallet 数据库 + 建表
```

### 4.3 为什么用 TiDB + GORM

| 理由 | 说明 |
|------|------|
| **TiDB：行锁支持** | 完全兼容 MySQL 的 `SELECT ... FOR UPDATE`，悲观锁并发控制的基础 |
| **TiDB：ACID 事务** | 支持完整事务，余额+流水同事务操作的保障 |
| **TiDB：DECIMAL 精度** | 原生支持 `DECIMAL(20,4)` / `DECIMAL(20,8)`，金融级精度无损 |
| **TiDB：水平扩展** | 底层 TiKV 自动分片，流水表数据增长后不需要手动分表 |
| **GORM：Gen 代码生成** | 自动生成 Model + Query + Repo，减少手写 SQL 的错误风险 |
| **GORM：事务 API** | `query.Q.Transaction(func(tx *query.Query) error { ... })` 清晰易用 |
| **GORM：Plugin 机制** | UserPlugin 自动审计字段，钱包的 B 端操作天然有审计追踪 |
| **工程统一** | 全工程 12 模块均使用 GORM + TiDB |

### 4.4 TiDB 对钱包的关键特性

```
钱包核心依赖的 TiDB 能力：

  1. SELECT ... FOR UPDATE（悲观锁）
     → 余额操作的并发控制基础
     → TiDB 文档明确支持，与 MySQL 语义一致

  2. ACID 事务
     → 余额更新 + 流水写入 + 幂等索引检查 = 同一事务
     → 失败自动回滚，不会出现余额改了但流水没写的情况

  3. DECIMAL 精度
     → DECIMAL(20,4) 金额：20位总长度、4位小数
     → DECIMAL(20,8) 汇率：覆盖加密货币 8 位小数
     → 数据库层面保证精确计算，无浮点误差

  4. 唯一索引（幂等兜底）
     → wallet_transaction 表的 uk_order_op_wallet 唯一索引
     → INSERT 冲突时返回 Duplicate Entry，幂等最终防线

  5. 自动分片（容量规划）
     → wallet_transaction 使用 Snowflake ID（分散写入不同 Region）
     → TiKV 自动 Rebalance，数据增长不需要手动分表
```

---

## 五、缓存 — Redis

### 5.1 已有情况

| 项目 | 信息 |
|------|------|
| **包路径** | `github.com/redis/go-redis/v9 v9.17.2` |
| **客户端类型** | UniversalClient（自动适配 Standalone/Cluster） |
| **集群支持** | RouteByLatency=true, MaxRedirects=8 |
| **超时** | ReadTimeout=2s, WriteTimeout=2s |
| **连接池** | 可配置 PoolSize / MinIdleConns |
| **Hash Tag** | 已有使用（`{item:list}:*`, `{banner:list}:*`） |
| **已有键模式** | user:token:* / enum:* / i18n:* / IP:* / exp:* / {item:list}:* 等 |
| **ETCD 配置** | Redis 地址和密码从 ETCD 读取 |

**文件位置**：
- 初始化：`/Users/mac/gitlab/common/pkg/rds/redis_init.go`

### 5.2 钱包模块如何复用

```
复用清单：
  ① 复用 Redis 初始化（rds.Init + ETCD 配置）
  ② 复用 UniversalClient（集群/单机自动适配）
  ③ 复用 Hash Tag 模式（{wal:cur}:* 保证同 slot）

钱包新增的 Redis 用途：

  ┌─────────────────────────────────────────────────────────────────┐
  │  键模式                                  │ 用途          │ TTL   │
  ├─────────────────────────────────────────────────────────────────┤
  │  {wal:idem}:{orderNo}:{op}:{wt}        │ 幂等快速拦截   │ 24h   │
  │  {wal:cur}:list                         │ 币种配置列表   │ 5min  │
  │  {wal:cur}:{code}                       │ 单个币种配置   │ 5min  │
  │  {wal:rate}:{code}                      │ 汇率数据      │ 同步  │
  │  {wal:wlimit}:{uid}:{cur}:{date}       │ 提现每日限额   │ 当日  │
  │  {wal:base}                             │ 基准币种      │ 永久  │
  └─────────────────────────────────────────────────────────────────┘

Redis 操作类型：
  · SetNX — 幂等键原子写入（核心操作，防重复入账）
  · Get/Set — 币种配置/汇率缓存读写
  · INCRBY — 提现每日限额原子累加
  · Del — 缓存主动刷新
```

### 5.3 为什么用 Redis

| 理由 | 说明 |
|------|------|
| **幂等快速拦截** | SetNX O(1) 操作，毫秒级拦截 99%+ 的重复请求，保护数据库 |
| **高频读缓存** | 币种配置/汇率数据读取频率极高（所有接口都要用），Redis 缓存命中率 99%+ |
| **原子计数器** | 提现每日限额用 INCRBY 原子累加，天然支持并发，无竞态条件 |
| **工程统一** | 全工程已使用 go-redis UniversalClient，键命名规范已有模式可参照 |
| **不缓存余额** | 余额是高频写数据，缓存一致性风险大于收益，选择直接查 TiDB |

---

## 六、配置中心 — ETCD

### 6.1 已有情况

| 项目 | 信息 |
|------|------|
| **包路径** | `go.etcd.io/etcd/client/v3 v3.6.7` |
| **初始化** | `etcd.InitEtcd()` + `etcd.LoadAndWatch("/slg/")` |
| **配置获取** | `etcd.DirectGet(ctx, key)` 直读 / `etcd.Get(ctx, key)` 读缓存 |
| **热更新** | LoadAndWatch 自动监听配置变更并更新本地缓存 |
| **配置路径** | `/slg/conf/tidb/*`, `/slg/conf/redis/*`, `/slg/serv/*` |
| **服务常量** | `common/pkg/consts/namesp/namesp.go` — 全部服务名/端口/DB 名 |
| **服务注册** | 通过 `registry-etcd` 自动注册到 ETCD |

**文件位置**：
- ETCD 初始化：`/Users/mac/gitlab/common/pkg/etcd/etcd_init.go`
- 服务命名空间：`/Users/mac/gitlab/common/pkg/consts/namesp/namesp.go`
- 环境变量：`/Users/mac/gitlab/common/pkg/consts/env/`

### 6.2 钱包模块如何复用

```
复用清单：
  ① 复用 ETCD 初始化和 LoadAndWatch
  ② 复用 TiDB/Redis 连接信息读取（已有的 env 常量）
  ③ 复用服务注册发现机制

需要做的改动：
  · namesp.go 新增 3 个常量：
    - EtcdWalletService = "ser-wallet"    // 服务名
    - EtcdWalletPort = "/slg/serv/wallet/port"  // 端口
    - EtcdWalletDB = "/slg/serv/wallet/db"      // 数据库名

钱包专属 ETCD 配置项（可选）：
  · /slg/serv/wallet/rate_update_interval  — 汇率更新间隔（分钟）
  · /slg/serv/wallet/rate_threshold        — 汇率偏差阈值（%）
  · /slg/serv/wallet/deposit_timeout       — 充值订单超时时间（分钟）
```

### 6.3 为什么用 ETCD

| 理由 | 说明 |
|------|------|
| **服务注册发现** | Kitex 服务间互相发现的基础设施，ser-wallet 启动后自动注册 |
| **配置中心** | 数据库/Redis/汇率参数等配置集中管理，支持运行时热更新 |
| **工程统一** | 全工程 12 个模块的配置和服务发现均基于 ETCD |
| **运行时调参** | 汇率更新间隔、阈值等参数可在 ETCD 中修改后实时生效，无需重启服务 |

---

## 七、消息队列 — NATS/JetStream

### 7.1 已有情况

| 项目 | 信息 |
|------|------|
| **包路径** | `github.com/nats-io/nats.go v1.48.0` |
| **JetStream** | 已启用，支持持久化消息 |
| **发布能力** | JSPublish（即时）/ JSPublishDelay（延迟消息） |
| **消费模式** | Pull + 手动 ACK / Queue 消费组 / Batch 批量消费 |
| **命名空间** | 环境变量 `NATS_NAMESPACE` 隔离（开发/测试/生产） |
| **自动建 Stream** | 发布时自动检查并创建 Stream |
| **连接配置** | ReconnectWait=2s, MaxReconnects=100, PingInterval=20s |

**文件位置**：
- 初始化：`/Users/mac/gitlab/common/pkg/nats/nats.go`

### 7.2 钱包模块如何复用

```
复用清单：
  ① 复用 NATS 初始化（nats.Init + ETCD 配置）
  ② 复用 JetStream 发布/消费 API
  ③ 复用延迟消息能力（JSPublishDelay）

钱包使用场景：

  场景一：充值订单超时关闭（核心场景）
  ├── 创建充值订单时 → JSPublishDelay("wallet.deposit.timeout", 30min)
  ├── 30 分钟后消费 → 检查订单是否仍为"待支付"
  └── 是 → 更新为"超时关闭"

  场景二：余额变动通知（可选）
  ├── 充值/提现/兑换完成后
  ├── JSPublish("wallet.balance.changed", payload)
  └── 推送服务消费并通知用户

  场景三：对账差异告警（可选）
  ├── 对账发现不一致
  └── JSPublish("wallet.reconcile.alert", payload)

  Subject 命名：<domain>.<resource>.<action>（遵循现有规范）
  消费组命名：ser-wallet-<domain>-<purpose>
```

### 7.3 为什么用 NATS

| 理由 | 说明 |
|------|------|
| **延迟消息** | JSPublishDelay 天然支持充值超时关闭，比定时扫表精确 |
| **可靠投递** | JetStream 持久化 + 手动 ACK，消息不丢失 |
| **解耦** | 余额变动后发事件，通知服务订阅处理，钱包服务不关心下游逻辑 |
| **工程统一** | 全工程已使用 NATS，基础设施完备 |
| **轻量** | 比 Kafka 更轻量，适合当前规模 |

### 7.4 重要约束

```
⚠️ 余额操作本身不走 MQ ⚠️

原因：
  · 余额更新和流水写入必须在同一个 DB 事务中
  · MQ 异步消费无法保证与 DB 事务的原子性
  · 如果 MQ 消费失败，余额已改但流水没写 = 数据不一致

MQ 的定位：
  · 只用于"操作完成后"的通知和调度
  · 不参与核心资金流
```

---

## 八、定时任务 — Cron

### 8.1 已有情况

| 项目 | 信息 |
|------|------|
| **包路径** | `github.com/robfig/cron/v3 v3.0.1` |
| **精度** | 秒级（`cron.WithSeconds()`） |
| **API** | `cron.AddJob(spec, func())` / `cron.RemoveJob(id)` |
| **生命周期** | 全局单例，Init() + Start() / Stop() |
| **备选** | XXL-JOB（`github.com/xxl-job/xxl-job-executor-go v1.2.0`） |

**文件位置**：
- Cron 封装：`/Users/mac/gitlab/common/pkg/cron/cron.go`
- XXL-JOB 封装：`/Users/mac/gitlab/common/pkg/xxljob/xxljob.go`

### 8.2 钱包模块如何复用

```
复用清单：
  ① 复用 robfig/cron 全局实例
  ② 复用 AddJob API

钱包定时任务：

  任务一：汇率定时更新（核心，必须有）
  ├── 表达式："0 */2 * * * *"（每 2 分钟，可从 ETCD 配置）
  ├── 逻辑：调 3 个三方 API → 取平均值 → 偏差检查 → 更新平台汇率 → 刷新缓存
  └── 容错：全部 API 失败 → 保持旧值 + 告警

  任务二：充值订单超时扫描（补充方案，配合 NATS 延迟消息）
  ├── 表达式："0 */5 * * * *"（每 5 分钟）
  ├── 逻辑：扫描 deposit_order 中超时的待支付订单 → 批量关闭
  └── 作用：NATS 延迟消息的兜底

  任务三：每日余额对账（v2）
  ├── 表达式："0 0 3 * * *"（每日凌晨 3 点）
  ├── 逻辑：遍历 wallet_account → 对比 SUM(流水) → 差异告警
  └── 作用：资金安全最后防线
```

### 8.3 为什么用 Cron

| 理由 | 说明 |
|------|------|
| **简单可靠** | 钱包的定时任务逻辑明确（汇率更新/订单超时/对账），robfig/cron 足够 |
| **秒级精度** | 汇率更新需要分钟级触发，秒级精度有余 |
| **工程统一** | 全工程已封装好，Init + AddJob 即用 |
| **不需要 XXL-JOB** | 钱包定时任务数量少（3 个），不需要分布式任务调度的复杂性 |

---

## 九、ID 生成 — Snowflake

### 9.1 已有情况

| 项目 | 信息 |
|------|------|
| **位置** | `common/pkg/utils/snowflake.go` |
| **算法** | 标准 Snowflake：timestamp(41) + datacenter(5) + machine(5) + sequence(12) |
| **Epoch** | 2024-01-01 00:00:00 UTC |
| **并发安全** | sync.Mutex 保护 |
| **全局实例** | sync.Once 单例模式 |
| **降级方案** | 未初始化时自动用 timestamp + random 降级 |
| **API** | `utils.GenerateSnowflakeID() int64` / `utils.GenerateUniqueString() string` |

### 9.2 钱包模块如何复用

```
复用清单：
  ① 直接调用 utils.GenerateSnowflakeID()

钱包使用场景：
  · wallet_transaction 表主键 — Snowflake ID（非自增，分散写入 TiDB Region）
  · 订单号生成的随机部分 — 也可用 Snowflake 的低位做随机种子

为什么 wallet_transaction 用 Snowflake 而不是 AUTO_INCREMENT：
  · TiDB 的 AUTO_INCREMENT 会导致写热点（所有 INSERT 集中在最后一个 Region）
  · Snowflake ID 天然离散，写入分散到多个 Region
  · TiDB 官方文档推荐对高写入表使用非连续 ID
```

### 9.3 为什么用 Snowflake

| 理由 | 说明 |
|------|------|
| **TiDB 写热点避免** | 分散写入到不同 Region，避免 AUTO_INCREMENT 的尾部热点 |
| **全局唯一** | 64 位 ID 全局唯一，跨服务安全 |
| **时间有序** | 高位为时间戳，天然按时间排序，有利于范围查询 |
| **工程已有** | 无需引入新依赖，直接调用 |

---

## 十、日志 — Zap + Lumberjack

### 10.1 已有情况

| 项目 | 信息 |
|------|------|
| **日志库** | `go.uber.org/zap v1.27.1` |
| **文件切割** | `gopkg.in/natefinch/lumberjack.v2 v2.2.1` |
| **格式** | JSON 结构化日志 |
| **输出** | 双通道：文件 + Stdout |
| **集成** | Kitex 集成 `kitex-contrib/obs-opentelemetry/logging/zap` |
| **TraceId** | 通过 `tracer.GetTraceID(ctx)` 获取并注入日志 |
| **API** | `lg.Log().CtxInfof(ctx, format, args...)` |

**文件位置**：
- 日志初始化：`/Users/mac/gitlab/common/pkg/lg/log.go`
- 链路追踪：`/Users/mac/gitlab/common/pkg/tracer/tracer.go`

### 10.2 钱包模块如何复用

```
复用清单：
  ① 复用 lg.Log() 全局日志实例
  ② 复用 tracer.GetTraceID(ctx) 获取链路 ID
  ③ 复用 tracer.GetUserId(ctx) 获取操作人 ID
  ④ 复用 JSON 结构化格式

钱包专属日志增强（在 ser 层调用时添加）：
  · 每笔余额操作：userId / currency / walletType / amount / orderNo / before / after
  · 幂等命中：orderNo / opType / source(redis/db)
  · 汇率更新：currency / oldRate / newRate / deviation / threshold
  · 对账差异：userId / currency / dbBalance / computedBalance / diff

示例代码模式（复用现有 API）：
  lg.Log().CtxInfof(ctx, "<%s> wallet_debit | user=%d cur=%s wt=%d amt=%s before=%s after=%s order=%s",
      tracer.GetTraceID(ctx), userId, currency, walletType, amount, before, after, orderNo)
```

### 10.3 为什么用 Zap

| 理由 | 说明 |
|------|------|
| **高性能** | Zap 是 Go 生态最快的结构化日志库（零分配） |
| **JSON 格式** | 结构化日志便于 ELK/Loki 等系统采集和查询 |
| **TraceId 贯穿** | 资金链路追踪的基础——一个 TraceId 串联从网关到 RPC 的完整链路 |
| **工程统一** | 全工程统一日志接口，无需额外集成 |

---

## 十一、审计插件 — UserPlugin

### 11.1 已有情况

| 项目 | 信息 |
|------|------|
| **位置** | `common/pkg/db/mysql/user_plugin.go` |
| **机制** | GORM Callback 插件，注册在 Create/Update 前 |
| **自动填充字段** | CreateBy / CreateAt / UpdateBy / UpdateAt |
| **用户来源** | `tracer.GetUserId(ctx)` 从上下文获取操作人 ID |
| **时间格式** | `time.Now().UnixMilli()`（毫秒级时间戳） |
| **批量支持** | 支持 Slice/Array 批量创建时逐行填充 |

### 11.2 钱包模块如何复用

```
复用清单：
  ① 直接复用，无需任何改动
  ② MySQL 连接初始化时自动注册 UserPlugin（db.Use(&UserPlugin{})）

钱包中的审计场景：
  · B 端币种编辑 → 自动记录谁改的、什么时候改的
  · B 端人工修正汇率 → 审计追踪
  · 所有带 create_by / update_by 字段的表自动填充

注意：
  · wallet_transaction（流水表）不需要 update_by（因为它 INSERT Only，永不 UPDATE）
  · wallet_transaction 的 create_at 由业务层显式设置（Snowflake ID 已包含时间信息）
```

### 11.3 为什么用 UserPlugin

| 理由 | 说明 |
|------|------|
| **自动化** | 无需在每个 Create/Update 调用中手动设置审计字段 |
| **一致性** | 全工程统一机制，不会出现有的表填了有的没填 |
| **资金审计需求** | 钱包的 B 端操作必须可追溯到人，UserPlugin 天然满足 |

---

## 十二、上下文传播 — Tracer + Metainfo

### 12.1 已有情况

| 项目 | 信息 |
|------|------|
| **位置** | `common/pkg/tracer/tracer.go` |
| **TraceId** | UUID，在 gate 层 IGlTracer 生成 |
| **传播机制** | `github.com/bytedance/gopkg/cloud/metainfo` — Kitex TTHeader 传递 |
| **可获取信息** | TraceId / UserId / UserToken / ClientIP / UserLang / DeviceID / DeviceType / UserName |
| **Header 常量** | X-Trace-Id / X-User-ID / X-Client-IP 等 |

### 12.2 钱包模块如何复用

```
复用清单：
  ① 所有 RPC 方法内通过 tracer.GetTraceID(ctx) 获取链路 ID
  ② 通过 tracer.GetUserId(ctx) 获取操作人（UserPlugin 也依赖它）
  ③ 通过 tracer.GetClientIp(ctx) 获取客户端 IP（安全日志需要）

钱包专属用途：
  · 每笔资金操作日志必须包含 TraceId → 全链路追踪
  · UserPlugin 自动从 context 获取 UserId 填充 CreateBy/UpdateBy
  · 出现对账差异时，可通过 TraceId 反查完整调用链路
```

### 12.3 为什么用 Metainfo

| 理由 | 说明 |
|------|------|
| **跨服务透传** | gate → ser-wallet → ser-finance，TraceId 自动传递，不需要手动携带 |
| **审计基础** | UserId 透传是 UserPlugin 自动填充审计字段的前提 |
| **工程统一** | 全工程统一的上下文传播机制 |

---

## 十三、代码生成 — GORM Gen + gen_kitex.sh

### 13.1 已有情况

**GORM Gen**：
| 项目 | 信息 |
|------|------|
| **包路径** | `gorm.io/gen v0.3.27` |
| **生成内容** | Model（struct 定义）+ Query（类型安全的查询构建器）+ Repo（封装的 CRUD 方法） |
| **生成入口** | 各服务的 `internal/gen/gorm_gen.go` |
| **Repo 模板** | `common/pkg/db/cmd/repo_template.go`（PageResult[T] 泛型分页等） |
| **配置规则** | FieldNullable=true, FieldWithIndexTag=false, FieldSignable=false |

**gen_kitex.sh**：
| 项目 | 信息 |
|------|------|
| **功能** | 扫描 `common/idl/*/` 目录，自动生成 Kitex 服务端代码 |
| **输出** | handler.go / main.go 骨架 + kitex_gen/ 目录 |
| **IDL 格式** | Apache Thrift |

### 13.2 钱包模块如何复用

```
GORM Gen 流程：
  1. 在 TiDB 中创建 ser_wallet 库和表
  2. 修正 ser-wallet/internal/gen/gorm_gen.go 的数据库引用
  3. 执行 go run internal/gen/gorm_gen.go
  4. 自动生成 internal/gorm_gen/model/ + query/ + repo/ 代码

gen_kitex.sh 流程：
  1. 在 common/idl/ser-wallet/ 下编写 Thrift IDL
  2. 执行 gen_kitex.sh
  3. 自动生成 common/pkg/kitex_gen/ser_wallet/ + ser-wallet 骨架代码

两个代码生成互不依赖，但都是业务开发前的必做步骤。
```

---

## 十四、新引入 — shopspring/decimal（高精度运算）

### 14.1 为什么需要新引入

当前工程中 **没有任何高精度运算库**（经搜索确认，无 shopspring/decimal、无 cockroachdb/apd、无任何 decimal 相关依赖）。

钱包模块涉及大量金额计算：
- 余额加减（1000000.0000 + 500.5000）
- 汇率换算（500000 × 0.00041000）
- 百分比计算（1000 × 1.025）
- 手续费计算（10000 × 0.02）
- 按比例拆分返奖

**如果用 float64 做这些计算**：

```go
// 经典精度丢失示例
a := 0.1 + 0.2   // 结果是 0.30000000000000004，不是 0.3
b := 1.0 - 0.9    // 结果是 0.09999999999999998，不是 0.1

// 在钱包中：
rate := 0.00041
amount := 500000.0
result := amount * rate  // 可能出现 204.99999... 而不是 205.0
// 差 0.00001 看起来小，但累积 100 万笔交易后差额就是 10 元
```

**这在金融系统中是绝对不可接受的。**

### 14.2 方案选择

| 方案 | 包路径 | Stars | 说明 | 适用场景 |
|------|--------|-------|------|---------|
| **① shopspring/decimal** | `github.com/shopspring/decimal` | 6.3k+ | Go 社区最广泛使用的高精度 decimal 库 | 金融计算、钱包、支付 |
| **② cockroachdb/apd** | `github.com/cockroachdb/apd/v3` | 600+ | CockroachDB 出品，性能极致 | 数据库引擎内部 |
| **③ math/big** | 标准库 | — | Go 标准库大数运算 | 需要手动管理精度 |
| **④ ericlagergren/decimal** | `github.com/ericlagergren/decimal` | 500+ | 高精度，API 相对复杂 | 科学计算 |

**推荐 ① shopspring/decimal**：

理由：
- Go 金融/支付项目的**事实标准**（Stripe Go SDK、多数 Go 钱包系统均使用）
- API 简洁直观：`decimal.NewFromString("100.50")`
- 支持 GORM 字段映射：可以定义 `type Amount = decimal.Decimal` 直接与 `DECIMAL(20,4)` 映射
- 6300+ Stars，社区活跃，维护良好
- 无外部 C 依赖，纯 Go 实现

### 14.3 使用场景示例

```go
import "github.com/shopspring/decimal"

// 余额计算
available := decimal.NewFromString("1000000.0000")
amount := decimal.NewFromString("500.5000")
newBalance := available.Sub(amount)  // 999500.0000（精确）

// 汇率换算
rate := decimal.NewFromString("0.00041000")
fiatAmount := decimal.NewFromString("500000")
bsbAmount := fiatAmount.Mul(rate)  // 205.0000（精确）

// 百分比计算（浮动比例）
platformRate := decimal.NewFromString("0.00041000")
floatPercent := decimal.NewFromString("0.025")  // 2.5%
depositRate := platformRate.Mul(decimal.NewFromInt(1).Add(floatPercent))

// 按比例拆分返奖
centerDeducted := decimal.NewFromString("800.0000")
rewardDeducted := decimal.NewFromString("200.0000")
totalDeducted := centerDeducted.Add(rewardDeducted)  // 1000.0000
rewardAmount := decimal.NewFromString("500.0000")  // 返奖总额
centerReward := rewardAmount.Mul(centerDeducted).Div(totalDeducted)  // 400.0000
rewardReward := rewardAmount.Sub(centerReward)  // 100.0000

// 四舍五入到指定精度（根据币种配置）
rounded := bsbAmount.Round(2)  // BSB 保留 2 位小数
```

### 14.4 引入影响

```
影响范围：仅 ser-wallet 模块

引入方式：
  cd ser-wallet && go get github.com/shopspring/decimal

依赖大小：纯 Go 库，无 C 依赖，编译体积增加 < 1MB

对现有工程的影响：
  · 零影响 — 只在 ser-wallet 的 go.mod 中新增
  · 不修改 common/pkg 的 go.mod
  · 其他模块不受任何影响

GORM 兼容性：
  · shopspring/decimal 原生支持 GORM 的 database/sql Scanner/Valuer 接口
  · 可直接用 decimal.Decimal 类型定义 Model 字段，自动与 DECIMAL 列映射
```

---

## 十五、工程已有但钱包不需要的组件

为了完整性，列出现有工程中钱包**暂时不需要**的组件及原因：

| 组件 | 现有位置 | 为什么钱包不需要 | 未来可能需要的场景 |
|------|---------|----------------|------------------|
| **Elasticsearch** | `common/pkg/es/` | 钱包无全文搜索需求，账变记录查询用 DB 索引足够 | 如果需要复杂的交易记录搜索/聚合分析 |
| **AWS S3** | `common/pkg/mys3/` | 钱包无文件上传/存储需求 | 如果需要上传提现凭证/KYC 影像 |
| **Kafka** | `common/pkg/kafka/` | NATS/JetStream 已满足异步消息需求，无需两套 MQ | 如果需要大吞吐量的交易事件流 |
| **XXL-JOB** | `common/pkg/xxljob/` | 钱包定时任务少（3 个），robfig/cron 足够 | 如果定时任务需要分布式协调/可视化管理 |
| **Excel 导出** | `xuri/excelize` | 初期无报表导出需求 | B 端需要导出交易记录/对账报告时 |
| **CSV** | `gocarina/gocsv` | 同上 | 同上 |
| **WebSocket** | `gorilla/websocket` | 钱包无实时推送需求（余额变动通知走 MQ + 推送服务） | 如果需要实时余额推送 |
| **i18n** | `go-i18n` | 钱包的错误信息走统一错误码体系，前端负责多语言 | 如果需要后端直接返回多语言文案 |
| **OTP/2FA** | `pquerna/otp` | 钱包不处理 2FA（由用户服务/KYC 服务负责） | — |
| **手机号解析** | `nyaruka/phonenumbers` | 钱包不处理手机号 | — |

---

## 十六、技术栈依赖关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ser-wallet 技术栈依赖全景                          │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                      ETCD（配置中心 + 服务注册）               │      │
│   │   /slg/conf/tidb/*  /slg/conf/redis/*  /slg/serv/wallet/*  │      │
│   └────┬───────────────────┬──────────────────────┬─────────────┘      │
│        │ 连接配置            │ 连接配置              │ 服务发现           │
│        ▼                   ▼                      ▼                    │
│   ┌─────────┐        ┌─────────┐           ┌──────────┐               │
│   │  TiDB   │        │  Redis  │           │  Kitex   │               │
│   │ (GORM)  │        │(go-redis)│          │  (RPC)   │               │
│   └────┬────┘        └────┬────┘           └────┬─────┘               │
│        │                  │                      │                     │
│   ┌────┴─────────────────┴──────────────────────┴─────┐               │
│   │                ser-wallet 业务层                      │               │
│   │                                                     │               │
│   │  ┌──────────────────────────────────────────────┐  │               │
│   │  │  wallet_core_ser（余额操作核心引擎）            │  │               │
│   │  │                                              │  │               │
│   │  │  依赖的技术能力：                              │  │               │
│   │  │  · TiDB:  FOR UPDATE + Transaction + DECIMAL │  │               │
│   │  │  · Redis: SetNX（幂等）+ INCRBY（限额）       │  │               │
│   │  │  · decimal: 高精度金额运算                     │  │               │
│   │  │  · Snowflake: 流水表 ID 生成                  │  │               │
│   │  │  · Zap: 资金操作结构化日志                     │  │               │
│   │  │  · Tracer: TraceId + UserId 上下文            │  │               │
│   │  └──────────────────────────────────────────────┘  │               │
│   │                                                     │               │
│   │  ┌──────────────────┐  ┌──────────────────────┐   │               │
│   │  │ currency_ser     │  │ deposit/withdraw/     │   │               │
│   │  │ (币种配置)        │  │ exchange_ser（业务）  │   │               │
│   │  │                  │  │                      │   │               │
│   │  │ · Redis: 缓存    │  │ · NATS: 延迟消息     │   │               │
│   │  │ · Cron: 汇率定时 │  │ · Kitex: 调财务RPC   │   │               │
│   │  │ · ETCD: 配置热更  │  │ · decimal: 汇率换算  │   │               │
│   │  └──────────────────┘  └──────────────────────┘   │               │
│   │                                                     │               │
│   └─────────────────────────────────────────────────────┘               │
│        │                  │                      │                     │
│        ▼                  ▼                      ▼                     │
│   ┌─────────┐        ┌─────────┐           ┌──────────┐               │
│   │  NATS   │        │  Cron   │           │ Snowflake│               │
│   │(JetStream)│      │(robfig) │           │ (自研)   │               │
│   └─────────┘        └─────────┘           └──────────┘               │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                  GORM Gen（代码生成层）                        │      │
│   │   Model + Query + Repo 自动生成                              │      │
│   │   UserPlugin 自动审计字段填充                                  │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                  新引入                                       │      │
│   │   shopspring/decimal（高精度金额/汇率运算）                    │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 十七、技术栈版本清单（精确版本号）

### 17.1 直接复用（已有，含版本号）

| # | 技术/组件 | 版本 | 包路径 | 钱包中的用途 |
|---|----------|------|--------|------------|
| 1 | Go | 1.25.5 | — | 运行时 |
| 2 | Kitex | v0.16.0 | `github.com/cloudwego/kitex` | RPC 框架，模块间通信 |
| 3 | Hertz | v0.10.3 | `github.com/cloudwego/hertz` | HTTP 网关（gate-font/gate-back） |
| 4 | GORM | v1.31.1 | `gorm.io/gorm` | ORM，数据库操作 |
| 5 | GORM Gen | v0.3.27 | `gorm.io/gen` | 代码生成（Model/Query/Repo） |
| 6 | GORM MySQL Driver | v1.6.0 | `gorm.io/driver/mysql` | TiDB 连接驱动 |
| 7 | go-redis | v9.17.2 | `github.com/redis/go-redis/v9` | 缓存、幂等、限额 |
| 8 | ETCD Client | v3.6.7 | `go.etcd.io/etcd/client/v3` | 配置中心、服务发现 |
| 9 | registry-etcd | v0.3.0 | `github.com/kitex-contrib/registry-etcd` | Kitex 服务注册 |
| 10 | NATS | v1.48.0 | `github.com/nats-io/nats.go` | 消息队列（延迟消息/事件通知） |
| 11 | Cron | v3.0.1 | `github.com/robfig/cron/v3` | 定时任务（汇率更新/对账） |
| 12 | Zap | v1.27.1 | `go.uber.org/zap` | 结构化日志 |
| 13 | Lumberjack | v2.2.1 | `gopkg.in/natefinch/lumberjack.v2` | 日志文件切割 |
| 14 | Snowflake | 自研 | `common/pkg/utils/snowflake.go` | 流水表 ID 生成 |
| 15 | UUID | v1.6.0 | `github.com/google/uuid` | TraceId 生成 |
| 16 | Sonic | v1.15.0 | `github.com/bytedance/sonic` | 高性能 JSON 序列化 |
| 17 | Copier | v0.4.0 | `github.com/jinzhu/copier` | 结构体拷贝（Req→Model 转换） |
| 18 | Errors | v0.9.1 | `github.com/pkg/errors` | 错误包装（堆栈信息） |
| 19 | Metainfo | v0.1.3 | `github.com/bytedance/gopkg` | RPC 上下文传播 |
| 20 | Kitex Zap Logger | — | `github.com/kitex-contrib/obs-opentelemetry/logging/zap` | Kitex 日志集成 |

### 17.2 需新引入

| # | 技术/组件 | 推荐版本 | 包路径 | 钱包中的用途 |
|---|----------|---------|--------|------------|
| 1 | shopspring/decimal | latest (v1.4+) | `github.com/shopspring/decimal` | 金额/汇率高精度运算 |

### 17.3 已有但不需要

| # | 技术/组件 | 原因 |
|---|----------|------|
| 1 | Elasticsearch | 无全文搜索需求 |
| 2 | AWS S3 SDK | 无文件上传需求 |
| 3 | Kafka (sarama) | NATS 已够用 |
| 4 | XXL-JOB | Cron 已够用 |
| 5 | Excelize | 初期无报表导出 |
| 6 | WebSocket | 无实时推送需求 |

---

## 十八、技术选择与挑战的映射

将技术栈与之前分析的 10 大技术挑战关联，说明每项技术解决什么问题：

| 技术挑战 | 依赖的技术组件 | 如何解决 |
|---------|--------------|---------|
| **① 并发与超卖** | TiDB (FOR UPDATE) + GORM (Transaction) | 悲观锁行级锁定，事务内校验+更新 |
| **② 数据一致性** | GORM Transaction + Redis (SetNX) + Kitex (RPC) | 本地事务 + 幂等重试 + TCC |
| **③ 幂等防护** | Redis (SetNX) + TiDB (唯一索引) | 双层防护，Redis 快拦截 + DB 兜底 |
| **④ 资金安全** | Hertz (中间件鉴权) + UserPlugin (审计) + Zap (日志) | 四层纵深防御 |
| **⑤ 多币种汇率** | shopspring/decimal + Redis (缓存) + Cron (定时更新) | DECIMAL 精度 + Quote-Lock + 三层汇率 |
| **⑥ 高性能** | Redis (缓存) + Snowflake (ID) + TiDB (自动分片) | 缓存高频读 + 分散写热点 |
| **⑦ 跨服务协调** | Kitex (RPC+超时+重试) + ETCD (服务发现) | 幂等重试 + 降级策略 |
| **⑧ 分布式事务** | GORM Transaction + Redis + Kitex | TCC（提现）+ 本地事务（兑换）|
| **⑨ 可观测性** | Zap + Tracer + Metainfo + UserPlugin | 结构化日志 + 链路追踪 + 审计 |
| **⑩ 可扩展性** | TiDB (弹性扩展) + ETCD (配置驱动) | 配置化新币种 + 自动分片 |

---

## 十九、总结

### 19.1 核心结论

```
┌──────────────────────────────────────────────────────────┐
│  技术栈总结                                                │
│                                                          │
│  已有复用：20 项（90%+）                                   │
│  新引入：1 项（shopspring/decimal）                        │
│  暂不需要：6 项（ES/S3/Kafka/XXL-JOB/Excel/WebSocket）   │
│                                                          │
│  结论：现有工程技术栈已具备实现钱包模块的                    │
│        全部核心能力，仅需引入 1 个高精度运算库               │
└──────────────────────────────────────────────────────────┘
```

### 19.2 开发前的工程改动清单

基于本技术栈分析，开发前需要做的工程改动：

| # | 改动 | 涉及文件 | 说明 |
|---|------|---------|------|
| 1 | 新增 ETCD 常量 | `common/pkg/consts/namesp/namesp.go` | EtcdWalletService / EtcdWalletPort / EtcdWalletDB |
| 2 | 新增 RPC Client | `common/rpc/rpc_client.go` | WalletClient() 工厂方法 |
| 3 | 加入 Workspace | `go.work` | 添加 `./ser-wallet` |
| 4 | 修正 Gen 配置 | `ser-wallet/internal/gen/gorm_gen.go` | namesp.EtcdAppDb → EtcdWalletDB |
| 5 | 创建 IDL 目录 | `common/idl/ser-wallet/` | 新建 Thrift IDL 文件 |
| 6 | 引入 decimal | `ser-wallet/go.mod` | `go get github.com/shopspring/decimal` |
| 7 | 创建数据库 | TiDB | CREATE DATABASE ser_wallet + 建表 DDL |

### 19.3 一句话总结

**钱包模块站在了一个"巨人的肩膀"上——Kitex 提供 RPC 通信、TiDB 提供事务和行锁、Redis 提供幂等和缓存、NATS 提供异步消息、ETCD 提供配置和发现、GORM Gen 提供代码生成、Zap + Tracer 提供可观测性、UserPlugin 提供审计能力。我们只需要额外引入 shopspring/decimal 做精确金额运算，其余全部复用已有基础设施，即可支撑一个安全、高并发、多币种的钱包系统。**
