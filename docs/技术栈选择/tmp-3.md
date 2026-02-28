# ser-wallet 技术栈选择与分析（完整梳理）

> 分析时间：2026-02-27
> 分析依据：现有工程实际代码（go.mod + common/pkg/ + 各服务实现）+ 架构设计方案 + 技术挑战分析 + 业界钱包系统实践
> 分析原则：
>   - **已有的复用** — 现有工程已引入且已使用的技术，直接复用，不重复造轮子
>   - **没有的评估** — 现有工程未引入但 ser-wallet 可能需要的技术，逐个评估是否引入
>   - **不想当然** — 每项技术给出"用在哪里 + 为什么用它"的明确依据
> 分析范围：钱包模块（主责）+ 币种配置（主责）+ 与财务模块联调相关

---

## 目录

```
一、技术栈总览：复用 vs 新增
二、核心框架层 — RPC 与 HTTP 通信
三、数据持久化层 — 数据库与 ORM
四、缓存层 — Redis
五、服务治理层 — 注册发现与配置
六、消息与异步层 — NATS / Kafka
七、ID 生成 — 雪花 ID
八、安全与加密 — AES / Token / 签名
九、国际化 — I18n 错误码翻译
十、日志与可观测 — 结构化日志 + 链路追踪
十一、定时任务 — 汇率刷新
十二、文件存储 — S3
十三、代码生成 — Thrift IDL + GORM Gen
十四、工具与通用能力 — 公共工具包
十五、不引入的技术（及原因）
十六、技术栈全景图
十七、总结
```

---

## 一、技术栈总览：复用 vs 新增

### 1.1 一句话结论

**ser-wallet 所需的全部技术栈，现有工程已 100% 覆盖，不需要引入任何新的中间件或框架。**

### 1.2 总览表

```
┌────┬──────────────────────────┬──────────┬──────────────────────────────────┬─────────────────────────┐
│ 序号│ 技术 / 组件               │ 状态     │ 在 ser-wallet 中的用途             │ 现有工程使用情况         │
├────┼──────────────────────────┼──────────┼──────────────────────────────────┼─────────────────────────┤
│  1 │ CloudWeGo Kitex v0.16.0  │ ✅ 复用  │ RPC 服务端 + 客户端通信           │ 全部 ser-* 服务使用      │
│  2 │ CloudWeGo Hertz v0.10.3  │ ✅ 复用  │ HTTP 网关转发（gate层）           │ gate-font / gate-back   │
│  3 │ TiDB (MySQL 协议)         │ ✅ 复用  │ 数据持久化（14张表）              │ 全部服务使用             │
│  4 │ GORM v1.31.1             │ ✅ 复用  │ ORM 数据访问                     │ 全部服务使用             │
│  5 │ GORM Gen v0.3.27         │ ✅ 复用  │ 代码生成（model/query/repo）      │ 全部服务使用             │
│  6 │ Redis (go-redis v9.17.2) │ ✅ 复用  │ 余额/币种/汇率缓存               │ 全部服务使用             │
│  7 │ ETCD v3.6.7              │ ✅ 复用  │ 服务注册发现 + 配置中心           │ 全部服务使用             │
│  8 │ Thrift IDL (thriftgo)    │ ✅ 复用  │ 接口契约定义 + 代码生成           │ 全部服务使用             │
│  9 │ Snowflake ID             │ ✅ 复用  │ 订单号/流水ID生成                │ utils.GenerateUniqueID  │
│ 10 │ AES-256-CBC 加密          │ ✅ 复用  │ 银行账号/手机号加密存储           │ ser-kyc / EncryptedString│
│ 11 │ Zap + Lumberjack 日志    │ ✅ 复用  │ 结构化日志 + 日志轮转            │ 全部服务使用             │
│ 12 │ OpenTelemetry 链路追踪    │ ✅ 复用  │ 请求链路追踪 + TraceID 传播      │ Kitex + Hertz 集成       │
│ 13 │ Tracer 上下文传播         │ ✅ 复用  │ 提取 user_id / trace_id 等       │ 全部服务使用             │
│ 14 │ NATS JetStream           │ ✅ 复用  │ 异步事件通知（预留）              │ common/pkg/nats 已封装   │
│ 15 │ Kafka (Sarama v1.46.3)   │ ✅ 复用  │ 异步消息（预留）                  │ common/pkg/kafka 已封装  │
│ 16 │ S3 (AWS SDK v2)          │ ✅ 复用  │ 币种图标上传                     │ ser-s3 / mys3 已封装     │
│ 17 │ Cron (robfig v3)         │ ✅ 复用  │ 汇率定时刷新                     │ common/pkg/cron 已封装   │
│ 18 │ XXL-JOB                  │ ✅ 复用  │ 分布式定时任务（可选）             │ common/pkg/xxljob 已封装 │
│ 19 │ go-i18n                  │ ✅ 复用  │ 错误码多语言翻译                  │ ser-i18n 服务 + Redis    │
│ 20 │ HTTP 工具 (utils)         │ ✅ 复用  │ 调用三方汇率 API                 │ utils.NewHttpUt() 已封装 │
│ 21 │ RPC Client 工厂           │ ✅ 复用  │ 调用 ser-kyc/user/blog/s3 等     │ common/rpc/rpc_client.go │
│ 22 │ ret 响应封装              │ ✅ 复用  │ 统一 RPC/HTTP 响应格式           │ common/pkg/ret 已封装    │
│ 23 │ 分页 DTO                  │ ✅ 复用  │ B端列表分页查询                   │ db.PageResult[T] 泛型   │
│ 24 │ Copier (jinzhu v0.4.0)   │ ✅ 复用  │ 结构体字段拷贝（model→resp）      │ 全部服务使用             │
│ 25 │ Sonic JSON (bytedance)   │ ✅ 复用  │ 高性能 JSON 序列化               │ utils.MarshalStr 已封装  │
│ 26 │ Testify v1.11.1          │ ✅ 复用  │ 单元测试断言                     │ 已有依赖                │
├────┼──────────────────────────┼──────────┼──────────────────────────────────┼─────────────────────────┤
│    │ 新增引入                  │          │                                  │                         │
├────┼──────────────────────────┼──────────┼──────────────────────────────────┼─────────────────────────┤
│    │ （无）                    │ —        │ 全部复用现有，不需要新引入         │                         │
└────┴──────────────────────────┴──────────┴──────────────────────────────────┴─────────────────────────┘
```

---

## 二、核心框架层 — RPC 与 HTTP 通信

### 2.1 CloudWeGo Kitex v0.16.0 — RPC 框架【复用】

```
用在哪里：
  ser-wallet 作为 RPC 服务端，接收来自 gate-font / gate-back / 财务模块 / 投注模块 的调用
  ser-wallet 作为 RPC 客户端，调用 ser-kyc / ser-user / ser-blog / ser-s3 / 财务模块

为什么用它：
  ✓ 全部现有服务（12个模块）都使用 Kitex，没有第二选择
  ✓ 基于 Thrift IDL 生成类型安全的接口代码
  ✓ 支持 ETCD 服务发现（kitex-etcd-registry v0.3.0）
  ✓ 支持 TTHeader 元数据传播（TraceID / UserID 等跨服务透传）
  ✓ 内置超时控制（3秒默认）+ 多路复用（MuxConnection）

现有代码位置：
  RPC 服务端配置：每个 ser-* 的 main.go（server.NewServer + InitRpcServerParams）
  RPC 客户端工厂：common/rpc/rpc_client.go（21 个服务的 sync.Once 单例客户端）

ser-wallet 需要做的：
  1. 在 common/rpc/rpc_client.go 中新增 WalletClient() 工厂方法
  2. 在 ser-wallet/main.go 中配置 RPC 服务端
  3. 在 handler.go 中实现 WalletService 接口方法
```

### 2.2 CloudWeGo Hertz v0.10.3 — HTTP 框架【复用，不直接使用】

```
用在哪里：
  ser-wallet 不直接使用 Hertz。
  Hertz 运行在 gate-font 和 gate-back 中，负责 HTTP→RPC 的协议转换。
  ser-wallet 的接口通过 Kitex RPC 暴露，由 gate 层的 Hertz handler 转发。

为什么提它：
  虽然 ser-wallet 不直接依赖 Hertz，但需要在两个 gate 中新增路由和 handler：
  - gate-font: /api/wallet/... → ser-wallet RPC
  - gate-back: /admin/api/wallet/... → ser-wallet RPC

ser-wallet 需要做的：
  1. gate-font/biz/handler/wallet/ — C端 handler（薄封装一行代码）
  2. gate-font/biz/router/wallet/ — C端路由注册
  3. gate-back/biz/handler/wallet/ — B端 handler
  4. gate-back/biz/router/wallet/ — B端路由注册
  5. 两个 gate 的 register.go 中添加 wallet.RouterWallet(r)
```

---

## 三、数据持久化层 — 数据库与 ORM

### 3.1 TiDB (MySQL 协议兼容) — 分布式数据库【复用】

```
用在哪里：
  ser-wallet 的全部 14 张表存储在 TiDB 中
  包括：currency_config / user_wallet / wallet_flow / wallet_freeze_record /
        deposit_order / withdraw_order / withdraw_account / exchange_record /
        exchange_rate_log / manual_adjustment_order / wallet_config 等

为什么用它：
  ✓ 现有工程全部服务都使用 TiDB（MySQL 协议兼容，GORM 无需适配）
  ✓ TiDB 自带分布式能力，无需应用层做分库分表
  ✓ 支持 CHECK 约束（可加 CHECK(available_balance >= 0) 兜底防超扣）
  ✓ 支持行级锁和条件更新（我们的并发控制核心机制）
  ✓ 水平扩展对应用层透明（数据量增长时 DBA 操作即可）

现有代码位置：
  数据库连接：common/pkg/db/mysql/mysql.go（Init + 连接池配置）
  各服务初始化：internal/cfg/mysql.go（sync.Once 单例初始化）
  连接参数配置：ETCD 中存储 DSN（如 /slg/serv/wallet/db → "ser_wallet"）

ser-wallet 需要做的：
  1. 在 TiDB 中创建 ser_wallet 数据库
  2. 执行 DDL 创建 14 张表
  3. 在 ETCD 中写入 /slg/serv/wallet/db → DSN 配置
  4. 在 ser-wallet/internal/cfg/mysql.go 中复用 db.Init() 模式

关键配置（来自现有工程）：
  SkipDefaultTransaction: true   — 显式事务控制（资金操作必须手动开事务）
  PrepareStmt: true              — 预编译语句缓存（提升性能）
  MaxIdleConnections: 10         — 连接池最小空闲
  MaxOpenConnections: 20         — 连接池最大连接数
```

### 3.2 GORM v1.31.1 — ORM 框架【复用】

```
用在哪里：
  ser-wallet 的全部数据库操作通过 GORM 进行
  包括：CRUD / 条件更新 / 事务 / 批量查询

为什么用它：
  ✓ 现有工程全部服务使用 GORM（无第二 ORM 选择）
  ✓ 支持 gorm.Expr() 做字段级原子操作（条件更新的核心）
  ✓ 支持 Transaction() 显式事务（余额变更 + 流水写入）
  ✓ 支持 RowsAffected 判断（条件更新是否生效）
  ✓ 与 GORM Gen 配合，类型安全

ser-wallet 重度依赖的 GORM 特性：

  ① gorm.Expr — 条件更新（并发控制核心）
    tx.Updates(map[string]interface{}{
        "available_balance": gorm.Expr("available_balance - ?", amount),
    })

  ② Transaction — 显式事务（余额+流水原子性）
    db.Transaction(func(tx *gorm.DB) error {
        // UPDATE user_wallet ...
        // INSERT wallet_flow ...
        return nil
    })

  ③ RowsAffected — 判断更新是否生效（余额充足性校验）
    if result.RowsAffected == 0 {
        return ErrInsufficientBalance
    }

  ④ Where 条件链 — 状态机控制
    tx.Where("order_no = ? AND status = ?", orderNo, FreezeStatusFrozen).
      Update("status", FreezeStatusConfirmed)
```

### 3.3 GORM Gen v0.3.27 — ORM 代码生成【复用】

```
用在哪里：
  从 TiDB 表结构自动生成 Go model / query / repo 代码
  生成路径：ser-wallet/internal/gorm_gen/（model/ + query/ + repo/）

为什么用它：
  ✓ 现有工程全部服务使用 GORM Gen（统一代码生成流程）
  ✓ 自动生成 18 个通用 CRUD 方法（每张表）
  ✓ 类型安全（字段名拼写错误编译期报错）
  ✓ FieldNullable: true — 可空字段自动生成指针类型（*string / *int64）

现有代码位置：
  生成器入口：每个 ser-*/internal/gen/gorm_gen.go
  生成器配置：common/pkg/db/mysql/mysql_gen.go

  生成入口模式：
    func main() {
        etcd.InitEtcd()
        mysql_gen.GenCodeWithAll(
            etcd.DirectGet(context.Background(), namesp.EtcdWalletDb),
            "ser-wallet", "",
        )
    }

ser-wallet 需要做的：
  1. 创建 ser-wallet/internal/gen/gorm_gen.go
  2. 先建表（DDL）→ 再执行 go run gorm_gen.go → 自动生成代码
  3. 后续加字段只需重新执行即可
```

---

## 四、缓存层 — Redis

### 4.1 Redis (go-redis v9.17.2) — 缓存中间件【复用】

```
用在哪里：
  ① 余额缓存 — wallet:bal:{uid}:{code}（TTL 30s，余额变更后 DEL）
  ② 币种列表缓存 — wallet:currencies（TTL 5min，B端编辑后 DEL）
  ③ 币种详情缓存 — wallet:curr:{code}（TTL 5min，B端编辑后 DEL）
  ④ 汇率缓存 — wallet:rate:{code}（定时任务覆盖写入）
  ⑤ 基准币种缓存 — wallet:base_currency（TTL 1h）

为什么用它：
  ✓ 现有工程全部服务使用 Redis（common/pkg/rds/ 已封装）
  ✓ UniversalClient 支持 Standalone / Cluster / Sentinel 三种模式
  ✓ 钱包首页余额查询是高频操作，缓存可将数据库查询减少 90%+
  ✓ Cache-Aside 模式是现有工程已有的缓存策略

现有代码位置：
  Redis 初始化：common/pkg/rds/redis_init.go（rds.Init + 全局 client）
  Redis Key 管理：common/pkg/consts/redis_key/redis_key.go
  各服务使用：internal/cfg/redis.go（sync.Once 初始化）

ser-wallet 需要做的：
  1. ser-wallet/internal/cfg/redis.go — 复用 rds.Init() 模式
  2. ser-wallet/internal/cache/balance_cache.go — 余额缓存读写+失效
  3. ser-wallet/internal/cache/currency_cache.go — 币种配置缓存读写+失效
  4. 在 common/pkg/consts/redis_key/ 中新增 wallet 相关的 key 前缀常量

缓存策略（来自架构设计方案）：
  写操作：先写数据库（事务提交）→ 再 DEL Redis Key
  读操作：先读 Redis → 未命中则读 DB → 回填 Redis（带 TTL）
  写操作的前置校验：不依赖缓存，由数据库 WHERE 条件保证
```

---

## 五、服务治理层 — 注册发现与配置

### 5.1 ETCD v3.6.7 — 服务注册发现 + 配置中心【复用】

```
用在哪里：
  ① 服务注册 — ser-wallet 启动后注册到 ETCD，其他服务可发现并调用
  ② 配置读取 — 数据库 DSN、端口号、Redis 地址等运行时配置
  ③ RPC 客户端发现 — rpc.WalletClient() 通过 ETCD 发现 ser-wallet 实例

为什么用它：
  ✓ 现有工程全部服务使用 ETCD（Kitex + Hertz 的 registry 插件）
  ✓ 配置集中管理，不硬编码在代码中
  ✓ 支持多实例部署和负载均衡（Kitex ETCD resolver 自动实现）

现有代码位置：
  ETCD 初始化：common/pkg/etcd/etcd_init.go（etcd.InitEtcd，sync.Once 单例）
  ETCD 读取：etcd.Get(ctx, key) / etcd.DirectGet(ctx, key)
  命名空间：common/pkg/consts/namesp/namesp.go

ser-wallet 需要写入 ETCD 的配置：
  /slg/serv/wallet/port  →  "8022"（端口号，需确认不冲突）
  /slg/serv/wallet/db    →  数据库 DSN 或数据库名

ser-wallet 需要在 namesp.go 中新增：
  EtcdWalletService = "wallet_service"
  EtcdWalletPort    = "/slg/serv/wallet/port"
  EtcdWalletDb      = "/slg/serv/wallet/db"
```

---

## 六、消息与异步层 — NATS / Kafka

### 6.1 NATS JetStream v1.48.0 — 持久化消息队列【复用，预留】

```
用在哪里（当前阶段预留，不主动使用）：
  ① 批量发奖异步化 — 活动模块批量调用 RewardCredit 时，通过 NATS 排队处理
  ② 余额变更事件通知 — 余额变更后通知统计模块更新报表
  ③ 大额操作风控通知 — 大额提现/兑换通知风控系统

为什么预留而不立即使用：
  × 余额变更是同步操作（调用方需要知道结果），不适合异步化
  × 当前需求中没有明确的异步场景
  × 不过度设计

为什么选 NATS 而不是 Kafka：
  现有工程两者都有（common/pkg/nats/ + common/pkg/kafka/）
  NATS JetStream 更轻量，适合业务事件通知
  Kafka 更适合大数据量的日志流、事件溯源

现有代码位置：
  NATS 封装：common/pkg/nats/nats.go
    nats.Init()             — 初始化
    nats.JSPublish()        — JetStream 发布（持久化）
    nats.JSSubscribe()      — JetStream 订阅
    nats.JSBatchConsumeLoop() — 批量消费

什么时候启用：
  → 监控发现批量发奖导致瞬时高并发 → 引入 NATS 队列削峰
  → 产品需求增加"余额变更推送" → 引入 NATS 事件通知
  → 只需在 Service 层加一行 nats.JSPublish()，不影响核心逻辑
```

### 6.2 Kafka (Sarama v1.46.3) — 高吞吐消息队列【复用，预留】

```
用在哪里（当前阶段预留）：
  可能用途：审计日志异步写入、交易数据同步到数据仓库

为什么预留：
  当前审计日志通过同步 RPC 调用 ser-blog 实现，够用
  数据仓库同步需求不明确

现有代码位置：
  Kafka 封装：common/pkg/kafka/producer.go + consumer.go
  生产者模式：kafka.NewProducer(opt).NewSend().WithJsonContent(data).Send("topic")
```

---

## 七、ID 生成 — 雪花 ID

### 7.1 Snowflake ID (bwmarrin v0.3.0 + 自定义封装) — 分布式 ID【复用】

```
用在哪里：
  ① 订单号生成 — 充值订单号 / 提现订单号 / 兑换单号 / 人工修正单号
  ② 流水ID生成 — wallet_flow 的 flow_id
  ③ 冻结记录ID — wallet_freeze_record 的记录标识

为什么用它：
  ✓ 现有工程已封装（common/pkg/utils/snowflake.go）
  ✓ 64位整数，全局唯一，时间有序
  ✓ 不依赖数据库自增（避免主键暴露业务量和性能瓶颈）
  ✓ 生成速度快（纯内存运算，无IO）

现有代码位置：
  工具函数：common/pkg/utils/snowflake.go
    utils.GenerateSnowflakeID()  — 生成雪花ID（int64）
    utils.GenerateUniqueID()     — 生成唯一ID（带回退机制）
    utils.ParseSnowflakeID()     — 解析ID中的时间戳等信息

  雪花ID结构（自定义 Epoch = 2024-01-01）：
    41位时间戳 + 5位数据中心 + 5位机器码 + 12位序列号
    每毫秒可生成 4096 个 ID（单节点）

ser-wallet 的使用场景：
  充值订单：deposit_order.order_no = utils.GenerateUniqueID()
  提现订单：withdraw_order.order_no = utils.GenerateUniqueID()
  兑换记录：exchange_record.record_no = utils.GenerateUniqueID()
  钱包流水：wallet_flow.flow_id = utils.GenerateUniqueID()
  冻结记录：wallet_freeze_record.freeze_no = utils.GenerateUniqueID()

  注意：外部传入的 orderNo（财务回调、投注扣款）由调用方生成，
  ser-wallet 只为自发起的操作生成 ID
```

---

## 八、安全与加密

### 8.1 AES-256-CBC 加密 — 敏感字段加密存储【复用】

```
用在哪里：
  ① 提现账户银行账号 — withdraw_account.bank_account_no（加密存储）
  ② 提现账户手机号 — withdraw_account.phone_number（加密存储）

为什么用它：
  ✓ 现有工程 ser-kyc 已有此模式（EncryptedString 类型，GORM 自动加解密）
  ✓ AES-256-CBC 是业界标准的对称加密算法
  ✓ GORM 钩子自动化（写入时加密 / 读取时解密），对业务代码透明

现有代码位置：
  加密工具：common/pkg/utils/crypto_util.go
    utils.AesEncrypt(plaintext, key, iv)
    utils.AesDecrypt(ciphertext, key, iv)
    utils.EncryptSensitiveData(text, key, iv)  — 含 Base64 编码
    utils.DecryptSensitiveData(text, key, iv)

  GORM 自动加解密类型：
    type EncryptedString string
    — 实现 Valuer / Scanner 接口
    — INSERT/UPDATE 时自动加密
    — SELECT 时自动解密

  密钥管理：
    DefaultKey / DefaultIV 全局变量
    实际生产环境从 ETCD 读取，不硬编码

ser-wallet 的用法：
  type WithdrawAccount struct {
      BankAccountNo EncryptedString `gorm:"column:bank_account_no" json:"bank_account_no"`
      PhoneNumber   EncryptedString `gorm:"column:phone_number" json:"phone_number"`
  }
  // 写入时自动加密，读取时自动解密，业务代码无感
```

### 8.2 Token / 认证机制 — 接入层安全【复用，不自己实现】

```
用在哪里：
  ① C端用户认证 — gate-font 的 FontAuthMiddleware 从 Token 提取 user_id
  ② B端管理员认证 — gate-back 的 BackAuthMiddleware 从 Token 提取管理员信息
  ③ B端 URL 权限 — CheckUrlAuth 校验管理员是否有权访问该接口

为什么不自己实现：
  ✓ 认证在 gate 层完成，ser-wallet 只需从 context 中取用户信息
  ✓ tracer.GetUserId(ctx) 即可获取经过验证的 user_id
  ✓ 不需要在 ser-wallet 中处理 Token 验证逻辑

现有代码位置：
  认证中间件：common/pkg/middleware/gate-auth.go
    FontAuthMiddleware() — C端认证（Token → Redis 验证 → 注入 context）
    BackAuthMiddleware() — B端认证（Token + URL 权限校验）
    BlackCheck()         — IP 黑名单检查
    WhiteCheck()         — IP 白名单检查

  上下文提取：common/pkg/tracer/tracer.go
    tracer.GetUserId(ctx)      → int64（C端 user_id，已验证，不可伪造）
    tracer.GetUserName(ctx)    → string（B端管理员姓名）
    tracer.GetTraceID(ctx)     → string（链路追踪ID）
    tracer.GetClientIp(ctx)    → string（客户端IP）
    tracer.GetUserLang(ctx)    → string（用户语言偏好）
```

---

## 九、国际化 — I18n 错误码翻译

### 9.1 go-i18n + Redis — 多语言错误码【复用】

```
用在哪里：
  ser-wallet 返回的业务错误码（如 WalletInsufficientBalance）需要翻译为用户语言

为什么用它：
  ✓ 现有工程已有完整的 i18n 体系
  ✓ 错误码通过 ret.BizErr(ctx, errorCode, msg) 抛出
  ✓ gate 层的 ret.HttpErr() 自动从 ser-i18n（Redis）获取对应语言的翻译文本
  ✓ 前端根据翻译后的文本展示错误信息

现有代码位置：
  i18n 库：common/pkg/i18n/i18n.go
  错误码定义：各 ser-*/internal/errs/code.go
  翻译存储：Redis Hash（由 ser-i18n 管理）
  翻译调用：ret.HttpErr(ctx, err, i18nTransService)

ser-wallet 需要做的：
  1. ser-wallet/internal/errs/code.go — 定义 wallet 模块的错误码
  2. 在 ser-i18n 中配置对应错误码的多语言翻译
  3. Service 层通过 ret.BizErr(ctx, errs.WalletInsufficientBalance) 抛出
```

---

## 十、日志与可观测

### 10.1 Zap + Lumberjack — 结构化日志【复用】

```
用在哪里：
  ① 业务日志 — 余额变更、订单创建、状态流转等关键操作记录
  ② 错误日志 — RPC 调用失败、数据库异常等错误记录
  ③ 审计辅助 — 结合 TraceID 追踪完整请求链路

为什么用它：
  ✓ 现有工程全部服务使用 Zap（common/pkg/lg/ 已封装）
  ✓ JSON 格式结构化日志，便于 ELK 采集和分析
  ✓ Lumberjack 自动轮转（100MB 切割，30天保留，10份备份，gzip 压缩）
  ✓ 自动注入 TraceID（从 context 提取）

现有代码位置：
  日志初始化：common/pkg/lg/log.go
    lg.InitKLog() — Kitex 日志适配器
    lg.InitHLog() — Hertz 日志适配器

  使用方式：
    lg.Log().CtxInfof(ctx, "余额变更: uid=%d, amount=%d", uid, amount)
    lg.Log().CtxErrorf(ctx, "RPC调用失败: %v", err)

ser-wallet 的日志规范：
  所有余额变更操作 → CtxInfof 记录（用户ID + 金额 + 订单号 + 操作类型）
  所有外部 RPC 调用 → 记录请求和响应（成功 Info，失败 Error）
  状态机流转 → CtxInfof 记录（from_status → to_status）
```

### 10.2 OpenTelemetry — 分布式链路追踪【复用】

```
用在哪里：
  ① 跨服务调用追踪 — gate-font → ser-wallet → ser-kyc 的完整链路
  ② 性能分析 — 每个 RPC 调用的耗时统计

为什么用它：
  ✓ 现有工程已集成 OpenTelemetry（Kitex + Hertz 插件自动采集）
  ✓ TraceID 自动传播（TTHeader 元数据）
  ✓ ser-wallet 不需要额外配置，只需正确传递 context

现有代码位置：
  追踪集成：Kitex server/client options 中自动注册
  TraceID 传播：common/pkg/tracer/tracer.go（X-Trace-Id header）

ser-wallet 的做法：
  确保所有操作传递 context（ctx）→ 框架自动处理追踪上报
  不需要额外的追踪代码
```

---

## 十一、定时任务 — 汇率刷新

### 11.1 Cron (robfig v3) — 服务内定时任务【复用，主选】

```
用在哪里：
  ① 汇率定时刷新 — 每分钟调 3 家三方 API 更新实时汇率
  ② 超时订单扫描 — 每分钟扫描超时未回调的充值订单，标记失效
  ③ 对账定时任务 — 每天凌晨运行余额 vs 流水一致性校验（后期）

为什么用它：
  ✓ 现有工程已封装（common/pkg/cron/cron.go）
  ✓ 支持秒级精度的 cron 表达式
  ✓ 轻量，服务内运行，无额外部署
  ✓ 简单直接：cron.AddJob("*/1 * * * *", refreshRatesFunc)

现有代码位置：
  Cron 封装：common/pkg/cron/cron.go
    cron.Init()           — 初始化（秒级精度）
    cron.AddJob(spec, fn) — 添加定时任务
    cron.RemoveJob(id)    — 移除任务
    cron.Stop()           — 优雅停止

ser-wallet 的用法示例：
  // 在 main.go 或 service 初始化时
  cron.Init()
  cron.AddJob("0 */1 * * * *", func() {
      // 每分钟执行一次汇率刷新
      currencyService.RefreshExchangeRates(context.Background())
  })
  cron.AddJob("0 */5 * * * *", func() {
      // 每5分钟扫描超时订单
      depositService.ExpireStaleOrders(context.Background())
  })
```

### 11.2 XXL-JOB — 分布式任务调度【复用，可选备选】

```
用在哪里（可选）：
  如果 ser-wallet 部署多实例，cron 任务会在每个实例上重复执行
  XXL-JOB 可以保证每次只有一个实例执行任务

为什么是可选：
  × 汇率刷新每分钟执行一次，多实例重复执行的影响不大（写入结果幂等）
  × 超时订单扫描也是幂等的（已标记的不会重复标记）
  × 引入 XXL-JOB 需要 Admin Center 部署和配置

什么时候使用：
  → 如果出现多实例重复执行导致三方 API 配额消耗过快
  → 如果需要在 B 端后台手动触发/暂停定时任务
  → 将 cron 替换为 xxljob 只需改初始化代码

现有代码位置：
  XXL-JOB 封装：common/pkg/xxljob/xxljob.go
    xxljob.RegisterTask(pattern, handler) — 注册任务
    xxljob.Init(cfg)                      — 初始化
```

---

## 十二、文件存储 — S3

### 12.1 AWS S3 SDK v2 — 对象存储【复用】

```
用在哪里：
  ① 币种图标上传 — B端管理员上传币种图标（SVG/PNG ≤ 5KB）

为什么用它：
  ✓ 现有工程已有完整的 S3 体系（common/pkg/mys3/ + ser-s3 服务）
  ✓ 图标上传走 ser-s3 的 GetPresignUrl 获取预签名 URL → 前端直传
  ✓ ser-wallet 不直接操作 S3，通过 rpc.S3Client() 调用 ser-s3

现有代码位置：
  S3 封装：common/pkg/mys3/mys3.go
  S3 服务：ser-s3（管理预签名 URL、文件类型校验等）
  RPC 调用：rpc.S3Client().GetPresignUrl(ctx, req)

ser-wallet 的用法：
  B端 EditCurrency 接口 → 如果包含图标文件 →
    调 rpc.S3Client().GetPresignUrl() 获取上传 URL →
    前端用 URL 直传 S3 →
    ser-wallet 只保存 icon_url 字段
```

---

## 十三、代码生成 — Thrift IDL + GORM Gen

### 13.1 Thrift IDL + gen_kitex.sh — RPC 代码生成【复用】

```
用在哪里：
  ① IDL 定义接口契约 — service.thrift / wallet.thrift / currency.thrift 等
  ② 代码生成 — 从 IDL 生成 kitex_gen（类型定义 + 服务接口）

为什么用它：
  ✓ 现有工程全部服务使用 Thrift IDL（IDL-first 开发流程）
  ✓ gen_kitex.sh 自动生成服务端骨架（handler.go / main.go）和客户端代码
  ✓ 接口变更只需修改 IDL → 重新生成 → 编译检查即可

现有代码位置：
  IDL 目录：common/idl/ser-*/（每个服务的 IDL 定义）
  生成脚本：各 ser-*/gen_kitex.sh
  生成结果：common/pkg/kitex_gen/ser_xxx/（生成的 Go 代码）

ser-wallet 需要做的：
  1. 创建 common/idl/ser-wallet/ 目录
  2. 编写 service.thrift + wallet.thrift + currency.thrift + deposit.thrift +
     withdraw.thrift + exchange.thrift + record.thrift
  3. 执行 gen_kitex.sh 生成代码
  4. 实现 handler.go 中的接口方法
```

### 13.2 GORM Gen — 数据库代码生成【复用】

```
（已在 3.3 节详述，此处不重复）

关键点：建表 → go run gorm_gen.go → 自动生成 model/query/repo → Service 层使用
```

---

## 十四、工具与通用能力 — 公共工具包

### 14.1 HTTP 客户端工具 — 调用三方汇率 API【复用】

```
用在哪里：
  汇率定时刷新 — 并行调用 3 家三方汇率 API（ExchangeRate-API / CoinGecko 等）

为什么用它：
  ✓ 现有工程已封装 HTTP 客户端（common/pkg/utils/http_util.go）
  ✓ Fluent Builder 模式，简洁易用
  ✓ 支持 GET/POST、Header 设置、JSON 解析

现有代码位置：
  HTTP 工具：common/pkg/utils/http_util.go

  使用示例：
    resp := utils.NewHttpUt().
        SetUrl("https://api.exchangerate-api.com/v4/latest/USD").
        SetMethodGet().
        SetHeaderKv("Authorization", "Bearer " + apiKey).
        Send().
        GetStringRes()
```

### 14.2 结构体拷贝 — model 到 resp 的转换【复用】

```
用在哪里：
  将 GORM Gen 生成的 model 结构体转换为 IDL 定义的 Resp 结构体

为什么用它：
  ✓ 现有工程使用 jinzhu/copier（自动按字段名匹配拷贝）
  ✓ 避免手写大量赋值代码

现有位置：
  go.mod 依赖：github.com/jinzhu/copier v0.4.0

  使用示例：
    var resp ser_wallet.CurrencyInfo
    copier.Copy(&resp, &currencyModel)
```

### 14.3 分页查询工具 — B端列表分页【复用】

```
用在哪里：
  B端管理接口：PageCurrency / PageExchangeRateLog / PageUserWallet

为什么用它：
  ✓ 现有工程已封装泛型分页 DTO（common/pkg/db/dto/page_result.go）

现有代码位置：
  分页 DTO：common/pkg/db/dto/page_result.go
    type PageResult[T any] struct {
        Data      []*T  `json:"data"`
        Total     int64 `json:"total"`
        PageTotal int64 `json:"page_total"`
        PageNo    int   `json:"page_no"`
        PageSize  int   `json:"page_size"`
    }
```

### 14.4 RPC Client 工厂 — 调用已有服务【复用】

```
用在哪里：
  ① rpc.KycClient().KycQuery()          — 提现时查 KYC 状态
  ② rpc.UserClient().GetUserInfoExpend() — 获取用户信息 / KYC 姓名
  ③ rpc.BLogClient().CreateLog()         — B端操作审计日志
  ④ rpc.S3Client().GetPresignUrl()       — 币种图标上传
  ⑤ rpc.FinanceClient().XxxMethod()      — 调用财务模块（待新增）

为什么用它：
  ✓ 现有工程已封装 21 个服务的 RPC Client 工厂（sync.Once 单例模式）
  ✓ 一行代码即可调用任何已注册的服务

现有代码位置：
  RPC 客户端：common/rpc/rpc_client.go

ser-wallet 需要做的：
  1. 新增 WalletClient() 工厂方法 — 供其他服务调用 ser-wallet
  2. 新增 FinanceClient() 工厂方法 — 供 ser-wallet 调用财务模块（如果财务模块已有 IDL）
```

### 14.5 响应封装 — 统一错误/成功响应格式【复用】

```
用在哪里：
  所有 RPC 接口的响应格式统一

为什么用它：
  ✓ 现有工程使用 ret.BizErr / ret.HttpOk / ret.RpcOne 等标准方法

现有代码位置：
  响应封装：common/pkg/ret/project_ret.go
    ret.BizErr(ctx, errorCode, msg...)  — 抛出业务错误
    ret.HttpOk(ctx, data)               — HTTP 成功响应
    ret.HttpErr(ctx, err, transServ)     — HTTP 错误响应（含 i18n 翻译）
    ret.RpcOne[T](ctx, data, err, code) — RPC 统一响应
```

---

## 十五、不引入的技术（及原因）

以下技术在钱包系统设计中常被提及，但经过评估，我们决定**不引入**：

```
┌──────────────────────┬──────────────────────────────────┬──────────────────────────────────┐
│ 技术                  │ 为什么不引入                      │ 什么情况下重新评估               │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ 分布式事务框架        │ 现有工程无 Seata/DTM；            │ 跨服务事务场景变得复杂且补偿     │
│ (Seata/DTM)          │ 本地事务+补偿已满足需求           │ 模式无法覆盖时                   │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ 分布式锁框架          │ 数据库条件更新已解决并发问题；     │ 出现单账户秒级 1000+ 次并发写入  │
│ (Redlock/Redsync)    │ 不需要跨服务互斥                  │                                  │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ Decimal 库            │ 全链路整数运算方案已确定；         │ 如果有非整除运算需要高精度       │
│ (shopspring/decimal)  │ Go int64 完全满足需求             │ 中间结果的场景（当前不存在）     │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ 状态机框架            │ Go 生态无主流状态机框架；          │ 状态路径变得极其复杂             │
│                      │ WHERE 条件控制简单有效             │ （超过 10 种状态 × 20+ 种流转） │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ 消息队列做余额变更    │ 余额变更是同步操作，调用方需要     │ 出现极高频批量写入需要削峰       │
│ (NATS/Kafka 主路径)  │ 即时知道结果，异步不适合           │                                  │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ 读写分离              │ TiDB 自带分布式读写能力，          │ DBA 评估需要应用层路由读写       │
│ (GORM DB Resolver)   │ 不需要应用层处理                  │ （极少见情况）                   │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ GraphQL               │ 现有工程全部 RESTful + RPC；      │ 不会引入                         │
│                      │ 不符合现有架构                     │                                  │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ WebSocket 实时推送    │ 余额变更不需要实时推送；           │ 产品需求增加"余额变更实时推送"  │
│ (ser-websocket)      │ 刷新页面即可看到最新余额           │ 时可接入 ser-websocket           │
├──────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ Elasticsearch         │ 钱包的查询场景不需要全文搜索；     │ B端需要复杂的流水搜索/聚合统计  │
│                      │ 分页查询用 MySQL 索引足够          │ 且 MySQL 无法满足时              │
└──────────────────────┴──────────────────────────────────┴──────────────────────────────────┘
```

---

## 十六、技术栈全景图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ser-wallet 技术栈全景                                    │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                          接入层（gate 复用）                             │    │
│  │                                                                         │    │
│  │  Hertz v0.10.3          FontAuthMiddleware        BackAuthMiddleware    │    │
│  │  HTTP 路由转发           C端 Token 认证            B端 Token+URL权限     │    │
│  │                          BlackCheck IP黑名单       WhiteCheck IP白名单   │    │
│  └─────────────────────────────────┬───────────────────────────────────────┘    │
│                                    │ Kitex RPC                                  │
│  ┌─────────────────────────────────▼───────────────────────────────────────┐    │
│  │                        RPC 通信层（Kitex 复用）                          │    │
│  │                                                                         │    │
│  │  Kitex v0.16.0                 Thrift IDL (thriftgo)                   │    │
│  │  RPC 服务端/客户端              接口契约定义 + 代码生成                   │    │
│  │  TTHeader 元数据传播            gen_kitex.sh 自动生成                    │    │
│  │  ETCD 服务发现 (v0.3.0)        类型安全的 Req/Resp 结构                 │    │
│  │  3秒超时 + 多路复用             WalletService 全部方法                   │    │
│  └─────────────────────────────────┬───────────────────────────────────────┘    │
│                                    │                                            │
│  ┌─────────────────────────────────▼───────────────────────────────────────┐    │
│  │                         业务逻辑层（ser-wallet 核心）                     │    │
│  │                                                                         │    │
│  │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐   │    │
│  │  │ 安全 & 认证        │  │ ID 生成            │  │ 错误码 & I18n     │   │    │
│  │  │                   │  │                   │  │                   │   │    │
│  │  │ tracer.GetUserId  │  │ Snowflake ID      │  │ errs/code.go     │   │    │
│  │  │ (context 提取)    │  │ (utils 已封装)    │  │ ret.BizErr()     │   │    │
│  │  │ EncryptedString   │  │ 订单号/流水ID     │  │ go-i18n + Redis  │   │    │
│  │  │ (AES-256-CBC)     │  │                   │  │ 多语言翻译        │   │    │
│  │  └───────────────────┘  └───────────────────┘  └───────────────────┘   │    │
│  │                                                                         │    │
│  │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐   │    │
│  │  │ 日志 & 追踪        │  │ 定时任务           │  │ HTTP 客户端       │   │    │
│  │  │                   │  │                   │  │                   │   │    │
│  │  │ Zap 结构化日志    │  │ Cron (robfig v3) │  │ utils.NewHttpUt  │   │    │
│  │  │ Lumberjack 轮转   │  │ 汇率每分钟刷新    │  │ 调三方汇率 API   │   │    │
│  │  │ OpenTelemetry     │  │ 超时订单扫描      │  │ Fluent Builder   │   │    │
│  │  │ TraceID 传播      │  │ 对账任务（后期）  │  │                   │   │    │
│  │  └───────────────────┘  └───────────────────┘  └───────────────────┘   │    │
│  │                                                                         │    │
│  │  ┌───────────────────┐  ┌───────────────────┐                          │    │
│  │  │ 工具库             │  │ RPC Client 工厂   │                          │    │
│  │  │                   │  │                   │                          │    │
│  │  │ copier 结构体拷贝 │  │ KycClient()      │                          │    │
│  │  │ sonic JSON 编解码 │  │ UserClient()     │                          │    │
│  │  │ PageResult[T]     │  │ BLogClient()     │                          │    │
│  │  │ 字符串/时间工具   │  │ S3Client()       │                          │    │
│  │  │                   │  │ FinanceClient()  │                          │    │
│  │  └───────────────────┘  └───────────────────┘                          │    │
│  └─────────────────────────────────┬───────────────────────────────────────┘    │
│                                    │                                            │
│  ┌─────────────────────────────────▼───────────────────────────────────────┐    │
│  │                         数据访问层                                       │    │
│  │                                                                         │    │
│  │  ┌───────────────────────────┐  ┌───────────────────────────────────┐   │    │
│  │  │ GORM v1.31.1              │  │ GORM Gen v0.3.27                  │   │    │
│  │  │                           │  │                                   │   │    │
│  │  │ ORM 数据访问              │  │ 自动生成 model/query/repo         │   │    │
│  │  │ gorm.Expr 条件更新        │  │ 18 个 CRUD 方法/每张表            │   │    │
│  │  │ Transaction 显式事务      │  │ FieldNullable: true               │   │    │
│  │  │ RowsAffected 结果判断     │  │ FieldSignable: false              │   │    │
│  │  └───────────────────────────┘  └───────────────────────────────────┘   │    │
│  └─────────────────────────────────┬───────────────────────────────────────┘    │
│                                    │                                            │
│  ┌─────────────────────────────────▼───────────────────────────────────────┐    │
│  │                         基础设施层                                       │    │
│  │                                                                         │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │    │
│  │  │ TiDB         │  │ Redis        │  │ ETCD v3.6.7  │  │ S3         │  │    │
│  │  │ (MySQL协议)  │  │ go-redis     │  │              │  │ (AWS SDK)  │  │    │
│  │  │              │  │ v9.17.2      │  │ 服务注册发现  │  │            │  │    │
│  │  │ 14张业务表   │  │              │  │ 配置中心      │  │ 币种图标   │  │    │
│  │  │ 连接池管理   │  │ 余额缓存     │  │ DSN/端口存储 │  │ 通过ser-s3 │  │    │
│  │  │ 事务+条件更新│  │ 币种缓存     │  │              │  │ 间接使用   │  │    │
│  │  │              │  │ 汇率缓存     │  │              │  │            │  │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘  │    │
│  │                                                                         │    │
│  │  ┌──────────────┐  ┌──────────────┐                                    │    │
│  │  │ NATS         │  │ Kafka        │  ← 两者当前预留，不主动使用         │    │
│  │  │ JetStream    │  │ (Sarama)     │    业务需要时随时可启用              │    │
│  │  │ 异步事件通知 │  │ 数据管道     │                                    │    │
│  │  │ （预留）     │  │ （预留）     │                                    │    │
│  │  └──────────────┘  └──────────────┘                                    │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 十七、总结

### 17.1 核心结论

```
1. 零新增引入 — ser-wallet 所需的全部技术栈在现有工程中已 100% 覆盖
   这是最理想的状态：不引入新依赖 = 不增加运维复杂度 = 不增加学习成本

2. 全面复用 — 26 项技术/组件全部复用，从框架到工具到中间件到公共包
   现有工程经过 12 个微服务模块的验证，技术选型已经成熟稳定

3. 预留扩展 — NATS / Kafka / XXL-JOB / Elasticsearch 等已引入但当前不使用
   需要时一行代码启用，不需要额外部署和配置

4. 不过度设计 — 不引入分布式事务框架、分布式锁、Decimal 库、状态机框架等
   用最简单的方案解决问题：条件更新、唯一索引、整数运算、WHERE 条件
```

### 17.2 技术栈按用途分类汇总

```
通信层（2项）：
  Kitex v0.16.0     — 微服务 RPC 通信（ser-wallet 的全部接口暴露和调用）
  Hertz v0.10.3     — HTTP 网关（gate 层路由转发，ser-wallet 间接使用）

数据层（3项）：
  TiDB              — 分布式数据库（14张表的持久化存储）
  GORM v1.31.1      — ORM 框架（条件更新/事务/查询）
  GORM Gen v0.3.27  — 代码生成（model/query/repo 自动生成）

缓存层（1项）：
  Redis v9.17.2     — 缓存中间件（余额/币种/汇率三类缓存）

服务治理（1项）：
  ETCD v3.6.7       — 服务注册发现 + 配置中心

消息层（2项，预留）：
  NATS JetStream    — 异步事件通知（批量发奖削峰等）
  Kafka             — 高吞吐数据管道（审计日志等）

安全层（2项）：
  AES-256-CBC       — 敏感字段加密（银行账号/手机号）
  Token 认证机制     — gate 中间件复用（FontAuth/BackAuth）

工具层（8项）：
  Snowflake ID      — 分布式唯一 ID 生成
  Zap + Lumberjack  — 结构化日志 + 轮转
  OpenTelemetry     — 分布式链路追踪
  Cron (robfig v3)  — 定时任务（汇率刷新/超时扫描）
  go-i18n           — 错误码多语言翻译
  HTTP 工具          — 调用三方汇率 API
  Copier + Sonic    — 结构体拷贝 + 高性能 JSON
  PageResult[T]     — 泛型分页 DTO

代码生成（2项）：
  Thrift IDL        — RPC 接口契约定义
  gen_kitex.sh      — 从 IDL 生成服务代码

文件存储（1项）：
  S3 (AWS SDK v2)   — 币种图标存储（通过 ser-s3 间接使用）
```

### 17.3 一句话总结

```
ser-wallet 站在一个成熟的技术平台上开发：
  通信有 Kitex/Hertz，存储有 TiDB/Redis/S3，治理有 ETCD，
  日志有 Zap，追踪有 OpenTelemetry，定时有 Cron，加密有 AES，
  生成有 Thrift IDL + GORM Gen，工具有 Snowflake/Copier/Sonic/i18n。

不需要引入任何新技术，只需要正确地复用已有的一切。
这不是"限制"，而是"优势"——同一套技术栈，12 个服务验证过的稳定性和一致性。
```
