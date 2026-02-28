# ser-wallet 技术栈选择与分析

> 分析时间：2026-02-27
> 分析原则：现有工程已有的组件优先复用，没有的才引入新依赖；每项技术说明用途和选择依据
> 分析依据：
> - 工程认知：/Users/mac/gitlab/z-readme/工程认知/tmp-1.md（四模块代码级分析）
> - 架构设计：/Users/mac/gitlab/z-readme/架构设计/tmp-1.md（7个ADR决策）
> - 技术挑战：/Users/mac/gitlab/z-readme/技术挑战/tmp-1.md（15个挑战及方案）
> - 实际代码探索：common/pkg/ 下所有已有封装
> 分类标准：
> - **已有复用** — 工程中已存在封装或依赖，直接用
> - **已有依赖需引入** — common/pkg 已声明依赖但未在业务服务中使用，首次接入
> - **新增引入** — 工程中完全没有，需要新增 go.mod 依赖

---

## 目录

1. [技术栈全景总览](#一技术栈全景总览)
2. [RPC框架：Kitex](#二rpc框架kitex)
3. [HTTP框架：Hertz](#三http框架hertz)
4. [接口定义：Thrift IDL](#四接口定义thrift-idl)
5. [数据库：TiDB](#五数据库tidb)
6. [ORM框架：GORM + GORM Gen](#六orm框架gorm--gorm-gen)
7. [缓存：Redis](#七缓存redis)
8. [配置中心与服务发现：ETCD](#八配置中心与服务发现etcd)
9. [消息队列：NATS JetStream](#九消息队列nats-jetstream)
10. [消息队列：Kafka（备选）](#十消息队列kafka备选)
11. [定时任务：robfig/cron](#十一定时任务robfigcron)
12. [高精度十进制运算：shopspring/decimal](#十二高精度十进制运算shopspringdecimal)
13. [分布式ID生成：Snowflake](#十三分布式id生成snowflake)
14. [对象存储：AWS S3](#十四对象存储aws-s3)
15. [操作日志：ser-blog](#十五操作日志ser-blog)
16. [多语言翻译：i18n 服务](#十六多语言翻译i18n-服务)
17. [KYC认证：ser-kyc](#十七kyc认证ser-kyc)
18. [链路追踪：Tracer](#十八链路追踪tracer)
19. [日志框架：lg](#十九日志框架lg)
20. [其他工具包](#二十其他工具包)
21. [技术栈分类总表](#二十一技术栈分类总表)
22. [依赖引入变更清单](#二十二依赖引入变更清单)

---

## 一、技术栈全景总览

### 1.1 一张图看清全部技术栈

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ser-wallet 技术栈全景                                   │
│                                                                              │
│  ┌─ 接入层 ─────────────────────────────────────────────────────────────┐   │
│  │ gate-font (Hertz HTTP)  ←── C端用户请求                                │   │
│  │ gate-back (Hertz HTTP)  ←── B端运营请求                                │   │
│  │ ser-finance (Kitex RPC) ←── 财务模块调用                               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              │ Kitex RPC                                     │
│  ┌─ 服务层 ─────────────────▼──────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  ser-wallet (Kitex RPC Server)                                      │   │
│  │                                                                      │   │
│  │  ┌── 业务代码 ────────────────────────────────────────────────┐    │   │
│  │  │  Thrift IDL → Kitex Gen → handler.go                       │    │   │
│  │  │  Service层: shopspring/decimal (金额运算)                   │    │   │
│  │  │  Repository层: GORM Gen (类型安全查询)                      │    │   │
│  │  │  定时任务: robfig/cron (汇率更新)                           │    │   │
│  │  │  ID生成: Snowflake (订单号)                                 │    │   │
│  │  └────────────────────────────────────────────────────────────┘    │   │
│  │                                                                      │   │
│  │  ┌── 基础设施依赖 ───────────────────────────────────────────┐    │   │
│  │  │  TiDB          数据持久化 (DECIMAL精度保障)                 │    │   │
│  │  │  Redis          缓存 + 分布式锁                             │    │   │
│  │  │  ETCD           配置中心 + 服务注册/发现                     │    │   │
│  │  │  NATS JetStream 异步事件通知 (wallet→finance)               │    │   │
│  │  │  AWS S3          币种图标存储                                │    │   │
│  │  └────────────────────────────────────────────────────────────┘    │   │
│  │                                                                      │   │
│  │  ┌── RPC 外部依赖 ──────────────────────────────────────────┐    │   │
│  │  │  ser-kyc         KYC认证状态查询                            │    │   │
│  │  │  ser-blog        操作审计日志                                │    │   │
│  │  │  ser-s3          文件上传                                   │    │   │
│  │  │  ser-buser       B端操作人名称                              │    │   │
│  │  │  i18n            错误码多语言翻译                            │    │   │
│  │  └────────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─ 工程工具 ──────────────────────────────────────────────────────────┐   │
│  │  gen_kitex.sh     IDL→RPC代码生成                                    │   │
│  │  gorm_gen.go      表→Model/Query代码生成                             │   │
│  │  Tracer           X-Trace-Id 全链路追踪                               │   │
│  │  ret/errs         统一返回值 + 错误码体系                              │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 分类统计

| 类别 | 数量 | 说明 |
|------|------|------|
| **已有复用** | 14项 | 工程中已封装，直接引用 |
| **已有依赖需引入** | 2项 | common 已声明依赖，ser-wallet 首次使用 |
| **新增引入** | 1项 | 需要新增 go.mod 依赖 |

---

## 二、RPC框架：Kitex

> 状态：**已有复用**
> 代码位置：`common/rpc/`、`common/kitex_gen/`
> Go依赖：`github.com/cloudwego/kitex`

### 2.1 是什么

CloudWeGo Kitex 是字节跳动开源的高性能 Go RPC 框架，支持 Thrift 和 Protobuf 协议。在本项目中作为所有微服务间通信的统一 RPC 框架。

### 2.2 在钱包模块中的用途

```
用途1：ser-wallet 作为 RPC Server 接收请求
  来源：gate-font/gate-back（网关转发）、ser-finance（财务模块调用）
  场景：所有钱包接口——币种CRUD、余额查询、充值/提现/兑换、WalletCredit/Freeze/Deduct 等

用途2：ser-wallet 作为 RPC Client 调用其他服务
  目标：ser-kyc（KYC查询）、ser-blog（审计日志）、ser-s3（图标上传）、ser-buser（操作人）
  模式：通过 common/rpc/rpc_client.go 中的工厂函数调用

用途3：其他服务通过 rpc.WalletClient() 调用 ser-wallet
  场景：ser-finance 调用 WalletCredit/WalletFreeze 等资金操作接口
```

### 2.3 为什么使用它

```
① 工程强制约束 — 所有 ser-xxx 服务都使用 Kitex，这是统一架构决策
② 性能优秀 — 字节内部千万级 QPS 验证，远超 gRPC-Go
③ 生态完善 — ETCD 服务发现、链路追踪、负载均衡一体化
④ 代码生成 — gen_kitex.sh 一键从 IDL 生成客户端/服务端代码
```

### 2.4 钱包模块需要做什么

```
1. 编写 IDL：common/idl/ser-wallet/service.thrift
2. 执行生成：bash common/script/gen_kitex.sh → 产出 common/kitex_gen/ser_wallet/
3. 注册客户端：common/rpc/rpc_client.go 新增 WalletClient() 工厂函数
4. 实现服务端：ser-wallet/handler.go 实现 WalletService 的所有方法
5. 启动注册：main.go 中使用 rpc.InitRpcServerParams() 创建 Kitex Server
```

### 2.5 关键配置

```go
// 客户端参数（已有统一配置）
client.WithResolver(etcdResolver)          // ETCD 服务发现
client.WithRPCTimeout(3 * time.Second)     // 3s 超时
client.WithMuxConnection(1)                // 连接复用
client.WithMetaHandler(transmeta.ClientTTHeaderHandler) // 链路追踪透传

// 服务端参数
server.WithRegistry(etcdRegistry)          // ETCD 注册
server.WithMuxTransport()                  // 多路复用
server.WithMetaHandler(transmeta.ServerTTHeaderHandler) // 链路追踪接收
```

---

## 三、HTTP框架：Hertz

> 状态：**已有复用**（钱包模块不直接使用，通过网关间接涉及）
> 代码位置：`gate-font/`、`gate-back/`
> Go依赖：`github.com/cloudwego/hertz`

### 3.1 是什么

CloudWeGo Hertz 是字节跳动开源的高性能 Go HTTP 框架。在本项目中用于两个网关服务。

### 3.2 在钱包模块中的用途

```
ser-wallet 本身不直接使用 Hertz。
但钱包的 HTTP 入口在网关层：

  gate-font（C端）：
    /api/wallet/deposit/create  → rpc.Handler(ctx, c, rpc.WalletClient().CreateDepositOrder)
    /api/wallet/exchange/create → rpc.Handler(ctx, c, rpc.WalletClient().CreateExchangeOrder)
    ...

  gate-back（B端）：
    /admin/api/wallet/currency/page → rpc.Handler(ctx, c, rpc.WalletClient().PageCurrency)
    /admin/api/wallet/currency/update → rpc.Handler(ctx, c, rpc.WalletClient().UpdateCurrency)
    ...

网关的 Handler 用 rpc.Handler() 通用转发，一行代码完成 HTTP→RPC 转发。
```

### 3.3 钱包模块需要做什么

```
在网关层新增路由文件：
  gate-font/biz/handler/wallet/ + gate-font/biz/router/wallet/
  gate-back/biz/handler/wallet/ + gate-back/biz/router/wallet/

在 register.go 中注册：
  wallet.RouterWallet(r)

Handler 全部用通用转发模式，不写业务逻辑。
```

---

## 四、接口定义：Thrift IDL

> 状态：**已有复用**
> 代码位置：`common/idl/`
> 工具：`common/script/gen_kitex.sh`

### 4.1 是什么

Apache Thrift 是接口定义语言（IDL），用于定义服务接口和数据结构。本项目所有微服务的接口契约都通过 Thrift IDL 定义。

### 4.2 在钱包模块中的用途

```
定义 ser-wallet 对外提供的所有 RPC 接口：
  - 请求/响应结构体（字段名、类型、必填/可选）
  - 服务方法签名（如 WalletCredit、PageCurrency 等）
  - 通过 gen_kitex.sh 自动生成 Go 代码
```

### 4.3 为什么使用它

```
① 工程统一约定 — 所有服务的 IDL 都用 Thrift，保持一致
② Kitex 原生支持 — Kitex 对 Thrift 的代码生成最成熟
③ 接口即文档 — IDL 文件本身就是接口契约，前后端、跨服务共享
```

### 4.4 关键规范

```
文件组织：common/idl/ser-wallet/
  ├── service.thrift      主入口（include 其他文件 + WalletService 定义）
  ├── currency.thrift     币种配置相关 struct
  ├── wallet.thrift       钱包操作相关 struct
  ├── deposit.thrift      充值相关 struct
  ├── withdraw.thrift     提现相关 struct
  ├── exchange.thrift     兑换相关 struct
  └── record.thrift       记录相关 struct

金额字段用 string 类型（不用 double，避免浮点精度丢失）
必填字段用 required，可选字段用 optional
字段带 (api.body="fieldName") 注解供 Hertz 绑定
```

---

## 五、数据库：TiDB

> 状态：**已有复用**
> 代码位置：`common/pkg/db/`
> 配置来源：ETCD `/slg/conf/tidb/*`

### 5.1 是什么

TiDB 是 PingCAP 开发的分布式 NewSQL 数据库，兼容 MySQL 协议。项目中所有服务统一使用 TiDB 作为关系型数据库。

### 5.2 在钱包模块中的用途

```
ser-wallet 的主存储，承载 db_wallet 数据库下的 9 张核心表：
  t_currency         币种配置
  t_rate_log         汇率日志
  t_wallet_account   子钱包账户（余额存储核心）
  t_wallet_flow      资金流水（审计+幂等核心）
  t_deposit_order    充值订单
  t_withdraw_order   提现订单
  t_exchange_order   兑换订单
  t_freeze_record    冻结记录
  t_withdraw_account 提现账户
```

### 5.3 为什么使用它

```
① 工程强制约束 — 所有 ser-xxx 共用同一 TiDB 集群，各自独立数据库
② DECIMAL(30,8) 精确支持 — 金额运算的精度保障（数据库引擎级别）
③ 乐观事务天然契合 — TiDB 的乐观事务模型与我们的乐观锁并发控制方案完美匹配
④ 水平扩展无感 — 数据量增长时自动分片，不需要应用层分库分表
⑤ MySQL兼容 — GORM + GORM Gen 无缝使用，无需特殊适配
```

### 5.4 关键配置

```
ETCD 键：
  /slg/conf/tidb/addr      TiDB地址
  /slg/conf/tidb/username   用户名
  /slg/conf/tidb/password   密码
  /slg/serv/wallet/db       数据库名（db_wallet）

初始化模式：sync.Once 单例（参照 ser-app/internal/cfg/mysql.go）
```

### 5.5 钱包模块特别依赖的 TiDB 能力

```
DECIMAL 类型      → 所有金额字段 DECIMAL(30,8)，精确到小数点后8位
UNSIGNED 约束     → 余额/金额字段不允许负数（DB层兜底）
唯一索引          → t_wallet_flow.(related_order_no, flow_type) 幂等保障
乐观事务          → version CAS 更新不锁行，高并发低冲突
事务原子性        → 余额更新+流水写入在同一事务中
```

---

## 六、ORM框架：GORM + GORM Gen

> 状态：**已有复用**
> 代码位置：`common/pkg/db/`、各服务 `internal/gorm_gen/`
> Go依赖：`gorm.io/gorm`、`gorm.io/gen`

### 6.1 是什么

GORM 是 Go 生态最主流的 ORM 框架。GORM Gen 是其代码生成工具，从数据库表结构自动生成类型安全的 Model 和 Query 代码。

### 6.2 在钱包模块中的用途

```
用途1：自动生成数据模型（Model）
  建表 → 运行 gorm_gen.go → 生成 internal/gorm_gen/model/*.gen.go
  每张表对应一个 Go struct（字段类型、标签自动映射）

用途2：自动生成查询 DAO（Query）
  生成 internal/gorm_gen/query/*.gen.go
  提供类型安全的链式查询（编译期检查字段名，避免拼错）

用途3：事务支持
  query.Q.Transaction(func(tx *query.Query) error {
      // 余额更新 + 流水写入在同一事务中
      // 使用 tx 而非 query.Q 确保事务一致性
  })

用途4：乐观锁的 CAS 更新
  w.Where(w.Version.Eq(oldVersion)).
    UpdateSimple(w.AvailableBalance.Add(amount), w.Version.Add(1))
```

### 6.3 为什么使用它

```
① 工程统一约定 — 所有 ser-xxx 都使用 GORM Gen 自动生成模型和查询
② 类型安全 — 编译期检查字段名和类型，避免运行时拼写错误
③ 软删除兼容 — .Where(xxx.DeleteAt.Eq(0)) 统一过滤模式
④ 事务简洁 — query.Q.Transaction() 闭包模式清晰
⑤ DECIMAL 映射 — GORM 自动将 DECIMAL 映射为 Go 的 string 或自定义类型
```

### 6.4 gorm_gen.go 模板

```go
// 文件：ser-wallet/internal/gen/gorm_gen.go

func main() {
    cfg.InitMySQL()  // 初始化数据库连接

    g := gen.NewGenerator(gen.Config{
        OutPath:      "../gorm_gen/query",
        ModelPkgPath: "../gorm_gen/model",
        Mode:         gen.WithDefaultQuery | gen.WithQueryInterface,
        FieldNullable: true,   // 可空字段生成指针类型
    })

    g.UseDB(cfg.DB())

    // 注册所有表
    g.ApplyBasic(
        g.GenerateModel("t_currency"),
        g.GenerateModel("t_rate_log"),
        g.GenerateModel("t_wallet_account"),
        g.GenerateModel("t_wallet_flow"),
        g.GenerateModel("t_deposit_order"),
        g.GenerateModel("t_withdraw_order"),
        g.GenerateModel("t_exchange_order"),
        g.GenerateModel("t_freeze_record"),
        g.GenerateModel("t_withdraw_account"),
    )

    g.Execute()
}
```

### 6.5 执行方式

```bash
# 先建表（SQL执行到 db_wallet）
# 然后运行代码生成
cd /Users/mac/gitlab/ser-wallet/internal/gen
go run gorm_gen.go
```

---

## 七、缓存：Redis

> 状态：**已有复用**
> 代码位置：`common/pkg/rds/`
> 配置来源：ETCD `/slg/conf/redis/*`

### 7.1 是什么

Redis 是高性能内存键值存储，本项目中用于缓存、会话管理、分布式锁等场景。

### 7.2 在钱包模块中的用途

```
用途1：币种配置缓存
  Key: wallet:currency:active_list     启用币种列表
  Key: wallet:currency:config:{code}   单币种配置
  TTL: 较长（如10分钟），CRUD后主动 DEL 失效
  理由：币种配置变更频率极低，缓存收益高

用途2：汇率数据缓存
  Key: wallet:rate:{currency_code}     某币种的平台汇率
  TTL: 与定时任务间隔一致（如60s）
  理由：兑换/充值/提现每次都要查汇率，缓存避免高频DB查询

用途3：分布式锁
  场景：汇率定时任务防多实例重复执行
  Key: wallet:lock:rate_update
  实现：rds.SetNX(key, token, TTL) + rds.Eval(Lua CAS释放)
  理由：多实例部署时只需一个实例执行定时任务

用途4：空结果缓存（防穿透）
  查询不存在的币种也缓存（value="NULL"，短TTL 60s）
  理由：防止恶意请求用不存在的币种代码反复打穿到DB
```

### 7.3 为什么使用它

```
① 工程已有封装 — common/pkg/rds/ 提供 Get/Set/Del/HGet/SetNX/Eval 等全套操作
② 所有服务共用 — 统一 Redis 实例，无额外运维成本
③ 分布式锁原生支持 — SetNX + Lua CAS 释放是业界标准模式
④ 缓存穿透防护 — 空结果缓存模式在 ser-app 中已有实践
```

### 7.4 已有封装函数（common/pkg/rds/）

```go
rds.Get(key)                           // 获取
rds.Set(ctx, key, value, ttl)          // 设置
rds.Del(keys...)                       // 删除
rds.HGet(hashKey, field)               // Hash 获取
rds.SetNX(key, value, ttl)             // 分布式锁获取
rds.Eval(script, keys, args)           // Lua 脚本执行（锁释放）
```

### 7.5 不缓存什么

```
❌ 用户余额 — 实时性要求高，余额变动频繁，缓存一致性代价大于收益
❌ 订单数据 — 写多读少，状态频繁变化
❌ 冻结记录 — 操作频率不高但正确性要求极高
原则：涉及资金的实时数据不缓存
```

---

## 八、配置中心与服务发现：ETCD

> 状态：**已有复用**
> 代码位置：`common/pkg/etcd/`
> Go依赖：`go.etcd.io/etcd/client/v3`

### 8.1 是什么

ETCD 是分布式键值存储，本项目中同时承担配置中心和服务注册/发现两个角色。

### 8.2 在钱包模块中的用途

```
用途1：服务配置管理
  /slg/serv/wallet/port    → 服务端口（如 9100）
  /slg/serv/wallet/db      → 数据库名（db_wallet）
  /slg/conf/tidb/*         → 数据库连接信息
  /slg/conf/redis/*        → Redis 连接信息
  /slg/conf/nats/*         → NATS 连接信息

用途2：服务注册与发现
  ser-wallet 启动 → ETCD 注册 "wallet_service" + 实例地址
  其他服务 → rpc.WalletClient() → ETCD 解析 "wallet_service" → 负载均衡选实例

用途3：汇率API配置（钱包特有）
  /slg/conf/rate/api1_url    → 第一家汇率API地址
  /slg/conf/rate/api1_key    → API密钥
  /slg/conf/rate/api2_url    → 第二家
  /slg/conf/rate/api3_url    → 第三家
  /slg/conf/rate/cron_interval → 更新间隔（如 "60s"）

用途4：热更新
  ETCD Watch 机制 → 配置变更实时推送 → 无需重启服务
```

### 8.3 为什么使用它

```
① 工程统一约定 — 所有服务的配置和注册都走 ETCD，无本地配置文件
② 热更新 — LoadAndWatch() 监听变更，自动刷新内存缓存
③ Kitex 原生集成 — ETCD Resolver 开箱即用
④ 配置安全 — 敏感信息（密码、密钥）不进代码仓库，全存 ETCD
```

### 8.4 已有封装函数

```go
etcd.LoadAndWatch("/slg/")                    // 加载全量 + 监听变更
etcd.Get(ctx, "/slg/conf/redis/addr")         // 从内存缓存读取（无网络开销）
etcd.DirectGet(ctx, "/slg/serv/wallet/port")  // 直接从 ETCD 读取
```

### 8.5 钱包模块需要新增的 ETCD 配置

```
必须新增（common/pkg/consts/namesp/namesp.go）：
  EtcdWalletService = "wallet_service"       // 服务注册名
  EtcdWalletPort    = "/slg/serv/wallet/port" // 端口键
  EtcdWalletDb      = "/slg/serv/wallet/db"   // 数据库名键

可选新增（汇率 API 配置）：
  /slg/conf/rate/api1_url
  /slg/conf/rate/api1_key
  /slg/conf/rate/api2_url
  /slg/conf/rate/api2_key
  /slg/conf/rate/api3_url
  /slg/conf/rate/api3_key
  /slg/conf/rate/cron_interval
```

---

## 九、消息队列：NATS JetStream

> 状态：**已有依赖需引入**
> 代码位置：`common/pkg/nats/`
> Go依赖：`github.com/nats-io/nats.go v1.48.0`（common/pkg/go.mod 已声明）
> 当前状态：封装完整但业务服务尚未使用，ser-wallet 将是首批使用者之一

### 9.1 是什么

NATS JetStream 是 NATS 消息系统的持久化流式消息层，提供 At-Least-Once 投递保证、消息持久化、消费者组等能力。

### 9.2 在钱包模块中的用途

```
用途1：提现创建 → 通知 finance 审核
  Subject: wallet.withdraw.created
  场景：ser-wallet 创建提现订单+冻结后 → 发消息通知 ser-finance 开始审核
  理由：wallet 不需要等 finance 审核结果（异步解耦）

用途2：充值创建 → 通知 finance 选通道（银行/电子钱包场景）
  Subject: wallet.deposit.created
  场景：用户选择银行转账充值 → wallet 创建订单 → 通知 finance 分配支付通道

消息消费方向：
  wallet 发布 → finance 消费
  finance 处理后通过 RPC 调 wallet 的资金接口（入账/冻结/解冻/扣款）
```

### 9.3 为什么使用它（而不是 Kafka）

```
① 工程已有完整封装 — common/pkg/nats/ 提供：
   JSPublish / JSPublishDelay（发布）
   JSQueueSubscribe / JSPullSubscribeBatch（消费）
   JSBatchConsumeLoop（批量消费循环）
   StartConsumers（高层消费者启动）

② 内置重试语义 — NakWithDelay(5s) + MaxDeliver 配置
   天然适合"通知审核失败后重试"的场景

③ 延迟消息 — JSPublishDelay() 可用于"超时未支付自动取消"
   如：充值订单 30 分钟未支付 → 延迟消息触发自动取消

④ 消息量不需要 Kafka 级别吞吐
   钱包的事件消息量远低于 Kafka 的适用场景

⑤ 运维简单 — NATS 单二进制部署，Kafka 需要 ZooKeeper/KRaft

⑥ Stream 自动创建 — 不需要提前创建 Topic/Partition
```

### 9.4 已有封装API

```go
// 初始化
nats.Init(nats.InitOption{Addr: addr, Token: token})

// 发布（JetStream 持久化）
nats.JSPublish("wallet.withdraw.created", jsonBytes)

// 延迟发布
nats.JSPublishDelay("wallet.deposit.timeout", jsonBytes, 30*time.Minute)

// 消费（队列模式，自动负载均衡）
nats.JSQueueSubscribe("wallet.withdraw.created", "ser-finance-withdraw", handler)

// 批量消费循环
nats.JSBatchConsumeLoop(ctx, sub, batchSize, maxWait, concurrency, handler)

// 高层封装：一次性启动多个消费者
nats.StartConsumers(ctx, cfg, []nats.ConsumerConfig{
    {Subject: "wallet.withdraw.created", QueueGroup: "ser-finance-withdraw", ...},
})
```

### 9.5 钱包模块的 Subject 规划

```
wallet.deposit.created      充值订单创建（银行/电子钱包）
wallet.withdraw.created     提现订单创建（通知审核）
wallet.deposit.timeout      充值超时未支付（延迟消息，自动取消）
```

---

## 十、消息队列：Kafka（备选）

> 状态：**已有依赖需引入**（备选，当前不使用）
> 代码位置：`common/pkg/kafka/`
> Go依赖：`github.com/IBM/sarama v1.46.3`（common/pkg/go.mod 已声明）

### 10.1 是什么

Apache Kafka 是分布式流处理平台，适合高吞吐量消息场景。

### 10.2 在钱包模块中的定位

```
当前定位：备选方案，暂不使用。

使用 NATS JetStream 的原因见上一章。

什么情况下会切换到 Kafka：
  - 消息量增长到万级 TPS 以上
  - 需要严格的分区有序性（如按用户分区保证同一用户的消息顺序）
  - 其他团队已大量使用 Kafka 且运维成熟

切换成本：
  封装在 common/pkg/kafka/ 已就绪，切换仅需改 import 和初始化
```

### 10.3 已有封装API

```go
// 生产者
producer := kafka.NewProducer(kafka.ProducerOption{Addr: addr})
producer.NewSend().WithJsonContent(msg).Send("wallet.withdraw.created")

// 消费者
consumer := kafka.NewConsumer(kafka.ConsumerOption{
    Topic:   "wallet.withdraw.created",
    GroupId: "ser-finance-withdraw",
    Handle:  func(msg []byte) error { ... },
})
consumer.Start()
```

---

## 十一、定时任务：robfig/cron

> 状态：**已有复用**
> 代码位置：`common/pkg/cron/`
> Go依赖：`github.com/robfig/cron/v3 v3.0.1`（common/pkg/go.mod 已声明）

### 11.1 是什么

robfig/cron 是 Go 生态最主流的定时任务库，支持秒级精度的 cron 表达式。

### 11.2 在钱包模块中的用途

```
用途1：汇率偏差检测与更新
  表达式：每 N 秒执行一次（如 "*/60 * * * * *" = 每60秒）
  逻辑：
    获取分布式锁 → 拉取3家API → 过滤异常值 → 计算偏差
    → 偏差≥阈值 → 更新平台汇率+入款/出款汇率 → 写汇率日志 → 清缓存
  分布式锁防多实例重复执行

用途2（可选）：离线对账定时任务
  表达式："0 0 3 * * *"（每日凌晨3点）
  逻辑：遍历所有 wallet_account，比对流水 balance_after vs 当前余额
```

### 11.3 为什么使用它

```
① 工程已有封装 — common/pkg/cron/ 提供 Init / AddJob / RemoveJob / Stop
② 秒级精度 — 支持 6 字段 cron 表达式（秒 分 时 日 月 周）
③ 简单够用 — 汇率更新只需一个定时任务，不需要复杂调度框架
④ 进程内运行 — 与 ser-wallet 同进程，汇率更新逻辑和币种数据紧密耦合
```

### 11.4 已有封装API

```go
cron.Init()                                    // 初始化（sync.Once 安全）
cron.AddJob("*/60 * * * * *", updateRates)     // 注册定时任务
cron.RemoveJob(entryID)                        // 移除任务
cron.Stop()                                    // 停止调度器
```

### 11.5 配合分布式锁

```go
func updateRates() {
    token := uuid.New().String()
    if !rds.SetNX("wallet:lock:rate_update", token, 60*time.Second) {
        return // 其他实例已在执行
    }
    defer releaseLock("wallet:lock:rate_update", token)

    // 执行汇率更新逻辑...
}
```

---

## 十二、高精度十进制运算：shopspring/decimal

> 状态：**新增引入**
> Go依赖：`github.com/shopspring/decimal`
> 当前状态：工程中尚未使用，需要在 ser-wallet 的 go.mod 中新增

### 12.1 是什么

shopspring/decimal 是 Go 生态最主流的高精度十进制运算库（30k+ GitHub stars），提供精确的 Decimal 类型，避免 float64 浮点误差。

### 12.2 在钱包模块中的用途

```
所有涉及金额的计算都使用 decimal.Decimal，不使用 float64：

用途1：汇率换算
  兑换获得BSB = decimal.NewFromString(sourceAmount).Div(decimal.NewFromString(rate))
  入款汇率 = platformRate.Mul(decimal.NewFromString("1").Add(fluctuation))
  出款汇率 = platformRate.Mul(decimal.NewFromString("1").Sub(fluctuation))

用途2：手续费计算
  手续费 = amount.Mul(feeRate.Div(decimal.NewFromInt(100)))
  到账金额 = amount.Sub(feeAmount)

用途3：余额校验
  available.LessThan(amount) → 余额不足

用途4：奖金计算
  奖金BSB = exchangeAmount.Mul(bonusRatio.Div(decimal.NewFromInt(100)))

用途5：对账校验
  balanceAfter.Equal(balanceBefore.Add(amount)) → 一致性断言
```

### 12.3 为什么使用它

```
① 精度零丢失 — 10进制运算，不存在 0.1+0.2=0.30000000000000004 问题
② Go 标准库缺失 — Go 没有内置 Decimal 类型，必须用第三方库
③ 生态最成熟 — 30k+ stars，广泛用于金融/支付系统
④ API 完善 — 加减乘除、比较、舍入（RoundBank/RoundHalfUp）、格式化
⑤ 与 DECIMAL(30,8) 配合 — DB 读出 string → decimal.NewFromString() → 计算 → StringFixed(8) → 存回DB
⑥ ADR-06 已决策 — 架构设计中明确采用此方案
```

### 12.4 核心API

```go
// 创建
d := decimal.NewFromString("100.50")
d := decimal.NewFromInt(100)

// 四则运算
result := a.Add(b)      // 加
result := a.Sub(b)      // 减
result := a.Mul(b)      // 乘
result := a.Div(b)      // 除

// 比较
a.LessThan(b)           // <
a.GreaterThan(b)        // >
a.Equal(b)              // ==
a.IsZero()              // == 0
a.IsNegative()          // < 0

// 舍入（银行家舍入，消除系统性偏差）
result := d.RoundBank(8)  // 保留8位小数
result := d.RoundBank(2)  // 保留2位小数

// 格式化输出
s := d.StringFixed(8)    // "100.50000000"
s := d.String()          // "100.5"

// 取最小值
result := decimal.Min(amount, balance)  // 人工减款：实际扣款=min(期望,余额)
```

### 12.5 引入方式

```bash
cd /Users/mac/gitlab/ser-wallet
go get github.com/shopspring/decimal
```

### 12.6 Code Review 红线

```
金额相关代码中出现 float64 → 直接 block
这是团队规范层面的保障，不允许任何金额运算使用浮点数
```

---

## 十三、分布式ID生成：Snowflake

> 状态：**已有复用**
> 代码位置：`common/pkg/myid/`

### 13.1 是什么

Snowflake（雪花算法）是 Twitter 开源的分布式唯一ID生成方案，生成 64 位整数ID，全局唯一、趋势递增。

### 13.2 在钱包模块中的用途

```
用途1：订单号生成
  充值：C + myid.GenID()  → "C1234567890123456"
  提现：T + myid.GenID()  → "T1234567890123456"
  兑换：E + myid.GenID()  → "E1234567890123456"

用途2：事件消息 event_id
  wallet→finance 的 NATS 消息体中携带唯一 event_id
  用于消费端幂等去重

用途3：流水记录 flow_id（如果不用自增主键的话）
```

### 13.3 为什么使用它

```
① 工程已有封装 — myid.GenID() 一行调用
② 全局唯一 — 分布式环境下不会重复
③ 趋势递增 — 对数据库索引友好
④ 高性能 — 纯本地生成，无网络开销
```

### 13.4 已有封装API

```go
id := myid.GenID()  // int64，如 1234567890123456
```

---

## 十四、对象存储：AWS S3

> 状态：**已有复用**
> 代码位置：`common/pkg/mys3/`
> 配置来源：ETCD `/slg/conf/s3/*`

### 14.1 是什么

AWS S3 兼容的对象存储服务，用于存储文件（图片、视频、文档等）。

### 14.2 在钱包模块中的用途

```
用途：币种图标上传和存储

  B端运营上传币种图标（SVG/PNG，≤5KB）
  → 通过 rpc.S3Client().UploadFile() 上传到 S3
  → 返回 URL 存入 t_currency.icon_url
  → C端展示时通过 /s3/* 路由代理访问

  这是唯一的文件存储需求，使用频率极低（币种新增/修改时才会上传）
```

### 14.3 为什么使用它

```
① 工程统一方案 — 所有文件上传都走 ser-s3 + AWS S3
② 已有完整链路 — 上传(UploadFile) + URL拼接(JoinUrl) + 代理访问(RedirectS3)
③ 无额外开发 — 调现有接口即可
```

### 14.4 使用方式

```go
// 上传（通过 RPC 调 ser-s3）
resp, err := rpc.S3Client().UploadFile(ctx, &ser_s3.UploadFileReq{
    Content:  fileBytes,
    FileName: "vnd_icon.svg",
    FilePath: "wallet/currency/",
})
iconUrl := mys3.JoinUrl(enums.GateTypeBack, resp.S3Key)

// URL 存入 t_currency.icon_url
```

---

## 十五、操作日志：ser-blog

> 状态：**已有复用**
> 调用方式：`rpc.BLogClient().AddActLog()`

### 15.1 是什么

ser-blog 是项目中的操作日志服务，记录 B 端运营人员的所有操作行为。

### 15.2 在钱包模块中的用途

```
所有 B 端写操作必须记录审计日志：

  CreateCurrency        → "新增币种（VND）"
  UpdateCurrency        → "编辑币种（VND）"
  ToggleCurrencyStatus  → "禁用币种（VND）" / "启用币种（VND）"
  SetBaseCurrency       → "设置基准币种（USDT）"

调用模式（oneway 异步，不阻塞主流程）：
  rpc.BLogClient().AddActLog(ctx, &ser_blog.ActAddReq{
      Platform:   "运营后台",
      Module:     "钱包管理",
      Page:       "币种配置",
      Action:     "新增币种（VND）",
      HandleType: "成功",
  })
```

### 15.3 为什么使用它

```
① 工程强制规范 — 所有 B 端写操作必须上报 ser-blog（审计要求）
② oneway 异步 — 不影响主流程性能
③ 合规需求 — 运营操作可追溯
```

---

## 十六、多语言翻译：i18n 服务

> 状态：**已有复用**
> 调用方式：`rpc.I18nClient()`

### 16.1 是什么

i18n 是项目中的多语言翻译服务，根据错误码返回对应语言的文案。

### 16.2 在钱包模块中的用途

```
错误码翻译：
  ret.BizErr(ctx, errs.InsufficientBalance, "余额不足")
  → 经 i18n 翻译 → 前端收到的是用户语言对应的文案
  → 英文用户看到 "Insufficient balance"
  → 越南用户看到 "Số dư không đủ"

在网关层的 rpc.Handler() 中自动调用 I18nClient() 翻译。
```

### 16.3 为什么使用它

```
① 平台跨国运营 — 越南、印尼、泰国等多国用户，必须多语言
② 自动翻译 — 错误码定义后，翻译由 i18n 服务统一管理
③ 与 ret.BizErr 联动 — 错误码→翻译→返回，链路自动化
```

---

## 十七、KYC认证：ser-kyc

> 状态：**已有复用**
> 调用方式：`rpc.KycClient()`

### 17.1 是什么

ser-kyc 是项目中的实名认证服务，管理用户的 KYC（Know Your Customer）认证状态。

### 17.2 在钱包模块中的用途

```
用途1：提现渠道决策
  rpc.KycClient().GetUserKycStatus(ctx, &req) → 获取用户 KYC 状态
  未认证 → 仅允许 USDT 提现
  已认证 → 全渠道（银行卡/电子钱包/USDT）

用途2：提现账户持有人校验
  rpc.KycClient().GetUserKycInfo(ctx, &req) → 获取 KYC 认证姓名
  比对：account_holder（忽略大小写+空格）== KYC姓名
  不一致 → 拒绝绑定提现账户
```

### 17.3 为什么使用它

```
① 需求硬要求 — "KYC状态决定可用提现渠道"
② 合规要求 — "提现账户持有人=KYC实名"
③ 已有 RPC 客户端 — common/rpc/rpc_client.go 中已注册 KycClient()
```

---

## 十八、链路追踪：Tracer

> 状态：**已有复用**
> 代码位置：`common/pkg/tracer/`

### 18.1 是什么

项目自建的链路追踪系统，通过 X-Trace-Id 贯穿全链路请求。

### 18.2 在钱包模块中的用途

```
用途1：请求追踪
  每个请求自动分配 traceId
  日志中打印 tracer.GetTraceID(ctx) → 方便排查问题
  从网关→ser-wallet→ser-finance 全链路可追溯

用途2：用户身份获取
  tracer.GetUserId(ctx)    → C端操作获取用户ID（确保操作归属正确）
  tracer.GetUserName(ctx)  → B端操作获取操作人姓名

用途3：审计日志
  所有返回值携带 traceId → 出问题时可追溯到完整调用链
```

### 18.3 已有封装API

```go
tracer.GetTraceID(ctx)     // 追踪ID
tracer.GetUserId(ctx)      // 用户ID（int64）
tracer.GetUserName(ctx)    // 用户名
tracer.GetUserToken(ctx)   // Token
tracer.GetUserLang(ctx)    // 语言偏好
tracer.GetUserDeviceId(ctx) // 设备ID
```

---

## 十九、日志框架：lg

> 状态：**已有复用**
> 代码位置：`common/pkg/lg/`

### 19.1 是什么

项目封装的日志工具，对 Kitex/Hertz 的日志进行统一封装。

### 19.2 在钱包模块中的用途

```
用途1：服务启动日志
  lg.InitKLog()  → Kitex 服务使用

用途2：业务日志
  lg.Log().CtxInfof(ctx, "充值入账成功 orderNo=%s amount=%s", orderNo, amount)
  lg.Log().CtxErrorf(ctx, "乐观锁冲突 userId=%d retry=%d", userId, retryCount)
  lg.Log().CtxWarnf(ctx, "汇率API不可用 api=%s", apiUrl)

用途3：defer 失败日志（Service层标准模式）
  defer func() {
      if retErr != nil {
          lg.Log().CtxErrorf(ctx, "CreateDepositOrder failed: %v", retErr)
      }
  }()
```

---

## 二十、其他工具包

### 20.1 统一返回值 ret

> 状态：**已有复用**
> 代码位置：`common/pkg/ret/`

```go
// Service 层错误包装
ret.BizErr(ctx, errs.InsufficientBalance, "余额不足")

// 返回值结构
type R struct {
    TraceId string      `json:"traceId"`
    Code    int         `json:"code"`      // 0=成功
    Msg     string      `json:"msg"`
    Data    interface{} `json:"data"`
}
```

### 20.2 错误码 errs

> 状态：**已有模式，需新建**
> 位置：`ser-wallet/internal/errs/code.go`

```
ser-wallet 错误码段：6022xxx

  6022000 ~ 6022099  币种配置模块
  6022100 ~ 6022199  钱包核心
  6022200 ~ 6022299  充值模块
  6022300 ~ 6022399  提现模块
  6022400 ~ 6022499  兑换模块
  6022500 ~ 6022599  记录模块
```

### 20.3 枚举 enum

> 状态：**已有模式，需新建**
> 位置：`ser-wallet/internal/enum/`

```
需要定义的枚举文件：
  currency.go   → CurrencyType（法币/加密/平台）
  wallet.go     → WalletType（中心/奖励/主播/代理/场馆）
  order.go      → DepositStatus/WithdrawStatus/ExchangeStatus/FreezeStatus
  flow.go       → FlowType（14种流水类型）
  method.go     → DepositMethod/WithdrawMethod/NetworkType/AccountType
```

### 20.4 通用工具 utils

> 状态：**已有复用**
> 代码位置：`common/pkg/utils/`

```go
utils.GetLocalIPv4()           // 获取本机IP（服务注册用）
utils.S2Int(ctx, str)          // 字符串转int
utils.GenerateOrderNo(prefix)  // 订单号生成（如果已有封装）
```

---

## 二十一、技术栈分类总表

### 21.1 按状态分类

| 状态 | 技术/组件 | 钱包中的用途 | 代码位置/依赖 |
|------|----------|-------------|-------------|
| **已有复用** | Kitex RPC | 服务间通信 | `common/rpc/` |
| **已有复用** | Hertz HTTP | 网关HTTP入口 | `gate-font/`、`gate-back/` |
| **已有复用** | Thrift IDL | 接口定义 | `common/idl/` |
| **已有复用** | TiDB | 数据持久化 | `common/pkg/db/` |
| **已有复用** | GORM + GORM Gen | ORM + 代码生成 | `internal/gorm_gen/` |
| **已有复用** | Redis | 缓存 + 分布式锁 | `common/pkg/rds/` |
| **已有复用** | ETCD | 配置 + 服务发现 | `common/pkg/etcd/` |
| **已有复用** | Snowflake | 订单号生成 | `common/pkg/myid/` |
| **已有复用** | AWS S3 | 币种图标存储 | `common/pkg/mys3/` |
| **已有复用** | ser-blog | 审计日志 | `rpc.BLogClient()` |
| **已有复用** | i18n | 多语言翻译 | `rpc.I18nClient()` |
| **已有复用** | ser-kyc | KYC认证查询 | `rpc.KycClient()` |
| **已有复用** | Tracer | 链路追踪 | `common/pkg/tracer/` |
| **已有复用** | lg | 日志框架 | `common/pkg/lg/` |
| **已有依赖** | NATS JetStream | 异步事件通知 | `common/pkg/nats/`（首次业务使用） |
| **已有依赖** | robfig/cron | 定时任务 | `common/pkg/cron/`（首次业务使用） |
| **新增引入** | shopspring/decimal | 高精度金额运算 | 需新增 go.mod 依赖 |

### 21.2 按职责分类

| 职责分类 | 技术/组件 | 说明 |
|---------|----------|------|
| **服务通信** | Kitex RPC | 同步RPC调用（资金操作） |
| **服务通信** | Hertz HTTP | 网关层HTTP入口 |
| **服务通信** | NATS JetStream | 异步事件通知（提现/充值通知） |
| **接口契约** | Thrift IDL | 接口定义+代码生成 |
| **数据持久化** | TiDB + GORM Gen | 关系型存储+类型安全ORM |
| **数据缓存** | Redis | 币种配置/汇率缓存 |
| **分布式协调** | ETCD | 配置中心+服务发现 |
| **分布式协调** | Redis (SetNX) | 分布式锁（定时任务防重） |
| **定时调度** | robfig/cron | 汇率偏差检测+对账 |
| **金额精度** | shopspring/decimal | 全链路十进制运算 |
| **ID生成** | Snowflake | 订单号/事件ID |
| **文件存储** | AWS S3 | 币种图标 |
| **安全合规** | ser-kyc | KYC认证状态 |
| **安全合规** | ser-blog | 操作审计日志 |
| **国际化** | i18n | 错误码多语言 |
| **可观测性** | Tracer + lg | 链路追踪+日志 |

### 21.3 按使用频率分类

```
高频使用（每个请求都涉及）：
  Kitex RPC      — 每个接口调用
  TiDB + GORM    — 每个数据操作
  Tracer + lg    — 每个请求日志

中频使用（特定业务场景）：
  Redis          — 币种/汇率查询时（缓存命中）
  decimal        — 涉及金额计算时
  Snowflake      — 创建订单时
  ser-kyc        — 提现相关接口
  ser-blog       — B端写操作

低频使用（定时/异步/偶发）：
  NATS JetStream — 提现/充值创建后通知（异步）
  robfig/cron    — 汇率定时更新（每分钟级别）
  AWS S3         — 币种图标上传（极偶发）
  ETCD (写入)    — 仅初始化和配置变更时
```

---

## 二十二、依赖引入变更清单

### 22.1 common 模块需要修改的文件

```
文件：common/pkg/consts/namesp/namesp.go
变更：新增 ser-wallet 的 ETCD 常量
  EtcdWalletService = "wallet_service"
  EtcdWalletPort    = "/slg/serv/wallet/port"
  EtcdWalletDb      = "/slg/serv/wallet/db"

文件：common/rpc/rpc_client.go
变更：新增 WalletClient() 工厂函数（sync.Once + 单例模式）

文件：common/idl/ser-wallet/ (新建目录)
变更：新建 IDL 文件（service.thrift 等）

执行：bash common/script/gen_kitex.sh
产出：common/kitex_gen/ser_wallet/
```

### 22.2 gate-font 需要修改的文件

```
新建：gate-font/biz/handler/wallet/ (handler 文件)
新建：gate-font/biz/router/wallet/ (路由注册文件)
修改：gate-font/biz/router/register.go → 加 wallet.RouterWallet(r)
```

### 22.3 gate-back 需要修改的文件

```
新建：gate-back/biz/handler/wallet/ (handler 文件)
新建：gate-back/biz/router/wallet/ (路由注册文件)
修改：gate-back/biz/router/register.go → 加 wallet.RouterWallet(r)
```

### 22.4 ser-wallet 新建目录的 go.mod

```
需要新增的唯一外部依赖：
  go get github.com/shopspring/decimal

其他依赖都通过 common 模块间接引用（Kitex/GORM/Redis/ETCD/NATS/Cron/Snowflake 等）
```

### 22.5 ETCD 需要新增的配置

```
必须：
  /slg/serv/wallet/port          → 如 "9100"
  /slg/serv/wallet/db            → "db_wallet"

可选（汇率API相关）：
  /slg/conf/rate/api1_url        → 第一家汇率API地址
  /slg/conf/rate/api1_key        → API密钥
  /slg/conf/rate/api2_url        → 第二家
  /slg/conf/rate/api2_key
  /slg/conf/rate/api3_url        → 第三家
  /slg/conf/rate/api3_key
  /slg/conf/rate/cron_interval   → 更新间隔（如 "60"，单位秒）
```

---

## 总结

```
ser-wallet 技术栈的核心特点：

1. 绝大多数组件已有 — 17 项中只有 1 项需要新增 go.mod 依赖
   工程基础设施成熟，开发重心在业务逻辑而非基建

2. 新增依赖仅 1 项 — shopspring/decimal
   这是金额运算的刚需，其他所有组件都已在 common 中就绪

3. 首次业务使用 2 项 — NATS JetStream + robfig/cron
   封装已就绪，代码已在 common/pkg/ 中，只是之前没有业务服务使用
   ser-wallet 将是这两个组件的首个业务用户

4. 技术选型与工程保持一致
   不引入新框架、不替换现有组件、不另起炉灶
   在现有基础设施上构建，降低学习成本和运维复杂度

5. 每项技术都有明确用途
   没有"可能用到"的冗余引入，每项都对应具体的业务场景
```
