# ser-wallet 模块工程结构设计

> 编写时间：2026-02-27
> 定位：模块级工程骨架设计 — 只定目录和文件，不写具体代码
> 核心目标：让人看到 tree 结构就知道"这个模块长什么样、每个文件干什么"
> 设计依据：
> - 工程规范：ser-app（21方法/5服务/125行handler）、ser-user（34方法/13域文件）、ser-live（45方法/8域文件）、ser-item（domain模式）实际代码分析
> - 需求规模：46 个接口（19C端 + 6B端 + 12RPC供 + 6RPC调 + 3定时）、37 个 IDL 方法、9 张数据表
> - 业务域划分：6 个功能域（钱包核心/币种配置/充值/提现/兑换/记录）+ 3 个定时任务
> - IDL 设计：7 个 thrift 文件（1 service + 6 domain）— 来源 IDL设计/tmp-2.md
> - 表结构设计：Must 7 张 + Should 2 张 — 来源 表结构设计/tmp-2.md

---

## 一、完整工程目录结构

```
ser-wallet/
│
├── main.go                          # 服务入口：初始化 + 启动 RPC Server
├── handler.go                       # RPC Handler：37 个方法的请求分发层
├── go.mod                           # 模块依赖声明
├── go.sum                           # 依赖校验
├── .gitignore                       # ✅ 已存在
├── .golangci.yml                    # ✅ 已存在
│
├── internal/                        # 内部实现（不对外暴露）
│   │
│   ├── cfg/                         # ── 基础设施初始化 ──
│   │   ├── mysql.go                 # TiDB 连接初始化（sync.Once 单例）
│   │   └── redis.go                 # Redis 连接初始化（sync.Once 单例）
│   │
│   ├── errs/                        # ── 错误码定义 ──
│   │   └── code.go                  # 按模块分段：wallet/currency/deposit/withdraw/exchange/record
│   │
│   ├── enum/                        # ── 枚举与常量 ──
│   │   ├── wallet_type.go           # 钱包类型（center/reward/anchor/agent/venue）
│   │   ├── currency_type.go         # 币种类型（fiat/crypto/platform）
│   │   ├── transaction_type.go      # 交易类型（deposit/withdraw/exchange/bonus/manual...）
│   │   ├── order_status.go          # 订单状态（充值状态/提现7状态/兑换状态）
│   │   └── enum_registry.go         # 枚举注册表（对接通用枚举查询）
│   │
│   ├── ser/                         # ── 服务层（核心业务逻辑）──
│   │   ├── wallet.go                # 钱包核心：余额查询 + 6个RPC余额操作 + 消费优先级
│   │   ├── currency.go              # 币种配置：B端CRUD + 基准币种 + 汇率日志 + RPC供他方
│   │   ├── deposit.go               # 充值流程：充值方式 + 创建订单 + USDT + 重复检测
│   │   ├── withdraw.go              # 提现流程：提现方式 + 创建订单 + 冻结 + KYC校验
│   │   ├── exchange.go              # 兑换流程：预览 + 执行（法币→BSB）
│   │   ├── record.go                # 账变记录：列表 + 详情（按Tab分类）
│   │   └── task.go                  # 定时任务：汇率更新 + 订单超时 + 余额对账
│   │
│   ├── rep/                         # ── 数据访问层（仓储）──
│   │   ├── wallet_account.go        # wallet_account 表操作（余额CRUD/冻结/行锁）
│   │   ├── wallet_transaction.go    # wallet_transaction 表操作（流水写入/查询）
│   │   ├── currency_config.go       # currency_config 表操作（币种CRUD）
│   │   ├── exchange_rate_log.go     # exchange_rate_log 表操作（汇率日志写入/查询）
│   │   ├── deposit_order.go         # deposit_order 表操作（充值订单生命周期）
│   │   ├── withdraw_order.go        # withdraw_order 表操作（提现订单7状态流转）
│   │   ├── withdraw_account.go      # withdraw_account 表操作（提现账户保存/查询）
│   │   ├── exchange_order.go        # exchange_order 表操作（兑换记录）
│   │   └── audit_requirement.go     # audit_requirement 表操作（稽核要求）
│   │
│   ├── cache/                       # ── 缓存层 ──
│   │   ├── wallet.go                # 余额缓存（热点用户余额 + 幂等键 SetNX）
│   │   ├── currency.go              # 币种配置缓存 + 汇率缓存（读多写少场景）
│   │   └── withdraw.go              # 提现相关缓存（每日限额计数 INCRBY）
│   │
│   ├── gen/                         # ── 代码生成 ──
│   │   └── gorm_gen.go              # ⚠️ 已存在，需修正 DB 引用
│   │
│   ├── gorm_gen/                    # ── GORM 自动生成（禁止手动修改）──
│   │   ├── model/                   # 生成的数据模型 *.gen.go
│   │   ├── query/                   # 生成的查询构建器 *.gen.go
│   │   └── repo/                    # 生成的基础仓储 *.gen.go
│   │
│   └── utils/                       # ── 工具函数 ──
│       └── order_no.go              # 订单号生成（C/T/B/A/M + 16位）
│
└── tmp/                             # 产品文档和原型图（不参与编译）
    ├── 多币种钱包产品原型/           # 37 张 C端原型截图
    ├── 多币种钱包需求文档/           # 64 张需求文档截图
    ├── 币种配置产品原型/             # 3 张 B端原型截图
    ├── 币种配置需求文档/             # 22 张需求文档截图
    ├── 财务管理产品原型/             # 26 张 B端原型截图（参考）
    └── 财务管理需求文档/             # 25 张需求文档截图（参考）
```

---

## 二、关联的 IDL 目录结构

ser-wallet 的 IDL 文件存放在 common 仓库，与服务代码分离：

```
common/idl/ser-wallet/               # ❌ 当前不存在，需要创建
├── service.thrift                    # 服务入口：WalletService 定义 37 个方法
├── wallet.thrift                     # 钱包核心域：余额操作 + 批量查询
├── currency.thrift                   # 币种配置域：CRUD + 汇率 + 基准币种
├── deposit.thrift                    # 充值域：方式获取 + 创建订单 + USDT
├── withdraw.thrift                   # 提现域：方式获取 + 创建订单 + 账户管理
├── exchange.thrift                   # 兑换域：预览 + 执行
└── record.thrift                     # 记录域：列表 + 详情
```

**gen_kitex.sh 生成的代码输出到**：

```
common/pkg/kitex_gen/ser_wallet/      # 自动生成，禁止手动修改
├── wallet.go                         # wallet.thrift 的 Go 类型
├── k-wallet.go                       # wallet.thrift 的序列化代码
├── currency.go
├── k-currency.go
├── deposit.go
├── k-deposit.go
├── withdraw.go
├── k-withdraw.go
├── exchange.go
├── k-exchange.go
├── record.go
├── k-record.go
├── service.go                        # WalletService 接口定义
├── k-service.go                      # 序列化代码
└── walletservice/                    # Client/Server 脚手架代码
    ├── client.go
    ├── server.go
    └── invoker.go
```

---

## 三、需要修改的 common 仓库文件

```
common/
├── idl/ser-wallet/                   # ❌ 新建：7 个 thrift 文件
├── pkg/consts/namesp/namesp.go       # ✏️ 新增：EtcdWalletService/Port/Db 三个常量
├── pkg/kitex_gen/ser_wallet/         # 🔄 自动生成：gen_kitex.sh 执行后产生
├── rpc/rpc_client.go                 # ✏️ 新增：WalletClient() 工厂方法
└── ../go.work                        # ✏️ 新增：./ser-wallet 条目
```

---

## 四、文件职责分层详解

### 4.1 根目录文件

| 文件 | 当前状态 | 职责说明 |
|------|---------|---------|
| **main.go** | ❌ 不存在 | 服务启动入口。职责：初始化日志 → ETCD 加载配置 → MySQL/Redis 初始化 → 注册定时任务 → 创建 Service 实例 → 注入 Handler → 启动 RPC Server |
| **handler.go** | ❌ 不存在 | RPC 请求分发层。职责：实现 WalletService 接口的 37 个方法，每个方法是一行委托调用（纯转发，无业务逻辑） |
| **go.mod** | ❌ 不存在 | 模块声明。`module ser-wallet`，go 1.25，引入 kitex/gorm/etcd/decimal 等依赖 |
| **.gitignore** | ✅ 已存在 | 忽略 gorm_gen 生成代码、IDE 文件等 |
| **.golangci.yml** | ✅ 已存在 | 代码检查规则（11 条规则，与其他服务一致） |

**main.go 与 handler.go 的关系（参考 ser-app 实际模式）**：

```
main.go                              handler.go
─────────                            ────────────
创建 Service 实例                      定义 WalletServiceImpl 结构体
  walletSer  = ser.NewWalletService()    持有 6 个 Service 引用
  currencySer = ser.NewCurrencyService()
  depositSer  = ser.NewDepositService()  37 个方法各自委托对应 Service
  withdrawSer = ser.NewWithdrawService()   func CreditWallet() → walletSer.CreditWallet()
  exchangeSer = ser.NewExchangeService()   func GetBalance()   → walletSer.GetBalance()
  recordSer   = ser.NewRecordService()     func EditCurrency() → currencySer.EditCurrency()
                                           ...（每个方法 1 行委托）
注入 → &WalletServiceImpl{...}
启动 → walletservice.NewServer(impl, opts...)
```

---

### 4.2 internal/cfg/ — 基础设施初始化

| 文件 | 职责 | 初始化内容 |
|------|------|-----------|
| **mysql.go** | TiDB 数据库连接 | sync.Once 单例 → 从 ETCD 读取地址/用户名/密码/库名 → 初始化 GORM → 设置 query.SetDefault(db) |
| **redis.go** | Redis 连接 | sync.Once 单例 → 从 ETCD 读取地址/密码 → 初始化 go-redis 客户端 |

**关键差异（对比 ser-app）**：
- ser-app 无 MySQL 初始化（无数据库表），ser-wallet 必须有
- ETCD 路径使用 `namesp.EtcdWalletDb`（当前不存在，需先注册）

---

### 4.3 internal/errs/ — 错误码定义

**单文件 code.go，按功能域分段**：

| 域 | 错误码范围 | 覆盖场景（举例） |
|------|-----------|----------------|
| wallet | 60XX000 ~ 60XX099 | 余额不足、冻结金额不足、钱包不存在、幂等重复 |
| currency | 60XX100 ~ 60XX199 | 币种不存在、币种已禁用、基准币种已设置 |
| deposit | 60XX200 ~ 60XX299 | 重复转账拦截、订单超时、充值金额不在范围 |
| withdraw | 60XX300 ~ 60XX399 | KYC未认证、超日限额、姓名不匹配、账户已存在 |
| exchange | 60XX400 ~ 60XX499 | 余额不足、BSB不可兑换、汇率过期 |
| record | 60XX500 ~ 60XX599 | 记录不存在 |

> 注：`XX` 是 ser-wallet 的服务编号（待分配，参考 ser-app=011）

**命名规范（遵循 ser-app 的 iota 模式）**：

```
WalletBaseError           = 60XX000 + iota
WalletInsufficientBalance                    // 60XX001 余额不足
WalletFrozenInsufficient                     // 60XX002 冻结余额不足
WalletAccountNotFound                        // 60XX003 钱包账户不存在
WalletIdempotentDuplicate                    // 60XX004 幂等重复操作
```

---

### 4.4 internal/enum/ — 枚举与常量

| 文件 | 内容 | 典型常量 |
|------|------|---------|
| **wallet_type.go** | 5 种钱包子类型 | `WalletTypeCenter=1, WalletTypeReward=2, WalletTypeAnchor=3, WalletTypeAgent=4, WalletTypeVenue=5` |
| **currency_type.go** | 3 种币种类型 + 币种代码 | `CurrencyTypeFiat=1, CurrencyTypeCrypto=2, CurrencyTypePlatform=3` + `CurrencyVND="VND"` 等 |
| **transaction_type.go** | 账变交易类型 | `TransDeposit=1, TransWithdraw=2, TransExchange=3, TransBonus=4, TransManualAdd=5, TransManualDeduct=6, TransBet=7, TransReward=8` |
| **order_status.go** | 各订单状态机 | 充值：待支付/支付中/成功/超时/失败；提现：待风控/待财务/待出款/成功/风控驳回/财务驳回/出款失败 |
| **enum_registry.go** | 枚举注册表 | 对接通用 GetEnumOptions 接口，注册上述 4 组枚举 |

**为什么拆 4 个文件而不是 1 个**：
- ser-app 的 enum/ 有 4 个文件（banner.go / category.go / relation_content.go / enum_registry.go）
- 每个文件对应一个业务概念域，便于查找
- 合成 1 个文件会超过 200 行，不利于维护

---

### 4.5 internal/ser/ — 服务层（业务逻辑核心）

这是整个模块最重要的一层。每个文件对应一个功能域，与 IDL 域文件一一映射。

#### 文件与接口对应关系

| 文件 | 接口数量 | 覆盖的接口编号 | 核心职责 |
|------|---------|-------------|---------|
| **wallet.go** | 9 | R-1~R-7, C-1, R-12 | **模块心脏**。6 个核心 RPC（Credit/Debit/Freeze/Unfreeze/DeductFrozen/QueryBalance）+ 消费优先级 DebitByPriority + C端余额查询 + 批量查询 |
| **currency.go** | 9 | B-1~B-6, R-8~R-10 | 币种 CRUD + 基准币种设置 + 汇率日志查询 + RPC供他方（GetCurrencyConfig/GetExchangeRate/ConvertAmount） |
| **deposit.go** | 7 | C-2~C-4, C-14~C-16, C-19 | 充值方式获取 + 创建充值订单 + USDT充值信息 + 重复转账检测 + 奖金活动 + 订单状态/取消 |
| **withdraw.go** | 6 | C-7~C-10, C-17, C-18 | 提现方式获取 + 创建提现订单(含冻结) + 提现账户管理 + 限额查询 + 历史提现方式 |
| **exchange.go** | 2 | C-5, C-6 | 兑换预览（实时计算到账金额）+ 执行兑换（法币→BSB，含赠送） |
| **record.go** | 2 | C-11, C-12 | 账变记录列表（按 Tab 分类 + 分页）+ 记录详情 |
| **task.go** | 3 | T-1~T-3 | 汇率定时更新 + 充值订单超时处理 + 余额对账校验 |

> 合计：9+9+7+6+2+2+3 = **38**（37 IDL 方法 + 1 内部定时任务调度）

#### 每个文件的内部结构模式

以 **wallet.go** 为例，遵循 ser-app 的 BannerService 模式：

```
wallet.go 内部结构：

┌─ WalletService struct ─────────────────────────────┐
│  repo   *rep.WalletAccountRepo                     │
│  txnRep *rep.WalletTransactionRepo                 │
│  cache  *cache.WalletCache                         │
└────────────────────────────────────────────────────┘
         │
         ├── NewWalletService(cache) → 构造函数
         │
         ├── CreditWallet(ctx, req)  → 加款（事务：余额+流水+幂等校验）
         ├── DebitWallet(ctx, req)   → 扣款（事务：余额校验+扣减+流水）
         ├── FreezeBalance(ctx, req) → 冻结（事务：可用→冻结+流水）
         ├── UnfreezeBalance(ctx, req)  → 解冻（事务：冻结→可用+流水）
         ├── DeductFrozen(ctx, req)  → 扣冻结（事务：冻结扣除+流水）
         ├── QueryBalance(ctx, req)  → 查余额（缓存优先→DB兜底）
         ├── DebitByPriority(ctx, req)  → 消费优先级扣款（先中心后奖励）
         ├── GetBalance(ctx, req)    → C端余额查询（多钱包聚合）
         └── BatchQueryBalance(ctx, req) → 批量查询（RPC供他方）
```

**关键特征（从 ser-app 代码分析得出）**：
- 每个 Service struct 持有自己的 Repo + Cache 引用
- 构造函数 `New*Service(cache)` 内部创建 Repo（Cache 从外部注入）
- 公开方法签名统一：`(ctx context.Context, req *thrift.XxxReq) (*thrift.XxxResp, error)`
- 资金操作在事务内完成：余额变更 + 流水写入同一事务
- 错误返回使用 `ret.BizErr(ctx, errs.Code, msg)`

---

### 4.6 internal/rep/ — 数据访问层

#### 文件与数据表对应关系

| 文件 | 对应表 | 核心操作 |
|------|-------|---------|
| **wallet_account.go** | wallet_account | GetByUserCurrencyType / Create / UpdateBalance / UpdateFrozen / **SelectForUpdate**（行锁） |
| **wallet_transaction.go** | wallet_transaction | Create（写入不可变流水）/ PageByFilter / GetByTxnNo / SumByUserCurrency |
| **currency_config.go** | currency_config | GetByCode / ListEnabled / Update / Create / SetBaseCurrency |
| **exchange_rate_log.go** | exchange_rate_log | Create / PageByFilter |
| **deposit_order.go** | deposit_order | Create / UpdateStatus / GetByOrderNo / CountPending（重复检测）/ BatchExpireTimeout |
| **withdraw_order.go** | withdraw_order | Create / UpdateStatus / GetByOrderNo / SumTodayAmount（日限额计算） |
| **withdraw_account.go** | withdraw_account | Create / GetByUserCurrency / ExistsByUser |
| **exchange_order.go** | exchange_order | Create / PageByFilter |
| **audit_requirement.go** | audit_requirement | Create / GetByOrderNo / UpdateProgress |

**关键模式（从 ser-app/internal/rep/banner.go 分析得出）**：

```
每个 Repo 文件的通用结构：

1. Filter struct — 封装查询条件
   type WalletAccountFilter struct {
       UserId       int64
       CurrencyCode string
       WalletType   int32
   }

2. Repo struct — 无状态
   type WalletAccountRepo struct{}
   func NewWalletAccountRepo() *WalletAccountRepo

3. 查询方法 — 使用 GORM Gen 构建器
   func (r *WalletAccountRepo) GetByUserCurrencyType(ctx, userId, code, wType) (*model.WalletAccount, error) {
       // query.Q.WalletAccount.WithContext(ctx).Where(...).First()
   }

4. 事务方法 — 传入 tx 参数
   func (r *WalletAccountRepo) UpdateBalanceInTx(tx *query.Query, ctx, id, amount) error {
       // tx.WalletAccount.WithContext(ctx).Where(...).UpdateColumn(...)
   }
```

**wallet_account.go 的特殊性**：
- 是唯一需要 `SELECT ... FOR UPDATE` 行锁的 Repo
- 余额更新必须在事务内：`query.Q.Transaction(func(tx) error { ... })`
- 每次余额变更同时写入 wallet_transaction 流水

---

### 4.7 internal/cache/ — 缓存层

| 文件 | 缓存内容 | 策略 |
|------|---------|------|
| **wallet.go** | 幂等键（SetNX 防重复）、热点用户余额 | 幂等键 TTL 24h；余额缓存写入时刷新 |
| **currency.go** | 币种配置列表、汇率数据（实时/平台/入款/出款） | 币种列表 TTL 较长；汇率随定时任务刷新 |
| **withdraw.go** | 每日提现累计额（INCRBY 计数） | 每日 0 点自动过期（TTL 到次日 0 点） |

**设计原则（从 ser-app/internal/cache/banner.go 分析得出）**：
- 返回 `nil` 表示缓存未命中（调用方需查 DB）
- 空结果也缓存（防穿透）
- 数据变更时主动清除相关缓存
- Key 构造由 cache 层负责，使用 common/pkg/consts/redis_key 包定义前缀

---

### 4.8 internal/gen/ — GORM 代码生成

| 文件 | 状态 | 说明 |
|------|------|------|
| **gorm_gen.go** | ⚠️ 已存在但引用错误 | 当前引用 `namesp.EtcdAppDb`，需改为 `namesp.EtcdWalletDb` |

**执行方式**：`go run internal/gen/gorm_gen.go`（手动触发，非编译时自动执行）

**输出**：扫描 ser-wallet 数据库的所有表 → 生成 model/query/repo 到 `internal/gorm_gen/` 目录

---

### 4.9 internal/gorm_gen/ — 自动生成代码（禁止手动修改）

```
gorm_gen/
├── model/                           # 表→结构体映射
│   ├── wallet_account.gen.go        # WalletAccount struct
│   ├── wallet_transaction.gen.go    # WalletTransaction struct
│   ├── currency_config.gen.go       # CurrencyConfig struct
│   ├── exchange_rate_log.gen.go     # ExchangeRateLog struct
│   ├── deposit_order.gen.go         # DepositOrder struct
│   ├── withdraw_order.gen.go        # WithdrawOrder struct
│   ├── withdraw_account.gen.go      # WithdrawAccount struct
│   ├── exchange_order.gen.go        # ExchangeOrder struct
│   └── audit_requirement.gen.go     # AuditRequirement struct
│
├── query/                           # 类型安全查询构建器
│   ├── gen.go                       # query.Q 全局入口 + SetDefault()
│   ├── wallet_account.gen.go
│   ├── wallet_transaction.gen.go
│   ├── currency_config.gen.go
│   ├── ... (每张表一个)
│
└── repo/                            # 基础仓储（可选，视 Gen 配置）
    ├── walletaccountr/
    ├── wallettransactionr/
    └── ... (每张表一个子目录)
```

**重要**：
- 所有 `*.gen.go` 文件头部标注 `// Code generated by gorm.io/gen. DO NOT EDIT.`
- 该目录已在 .gitignore 中排除（确认：已存在的 .gitignore 包含 `internal/gorm_gen`）
- 表结构变更后需重新运行 `go run internal/gen/gorm_gen.go`

---

### 4.10 internal/utils/ — 工具函数

| 文件 | 职责 |
|------|------|
| **order_no.go** | 订单号生成：`GenOrderNo(prefix string) string`，格式 = 前缀(C/T/B/A/M/E) + 16 位 |

**为什么只有 1 个文件**：
- 大部分工具函数已在 `common/pkg/utils/` 中（JSON 序列化、类型转换、雪花 ID 等）
- ser-wallet 仅需补充订单号生成这一个钱包特有的工具
- 如后续有其他特有工具，可在此目录扩展

---

## 五、文件总量统计

### 5.1 需手动编写的文件

| 目录层 | 文件数 | 文件列表 |
|--------|-------|---------|
| 根目录 | 2 | main.go, handler.go |
| cfg/ | 2 | mysql.go, redis.go |
| errs/ | 1 | code.go |
| enum/ | 5 | wallet_type.go, currency_type.go, transaction_type.go, order_status.go, enum_registry.go |
| ser/ | 7 | wallet.go, currency.go, deposit.go, withdraw.go, exchange.go, record.go, task.go |
| rep/ | 9 | wallet_account.go, wallet_transaction.go, currency_config.go, exchange_rate_log.go, deposit_order.go, withdraw_order.go, withdraw_account.go, exchange_order.go, audit_requirement.go |
| cache/ | 3 | wallet.go, currency.go, withdraw.go |
| utils/ | 1 | order_no.go |
| **合计** | **30** | |

### 5.2 需初始化/修改的文件

| 类别 | 文件数 | 说明 |
|------|-------|------|
| go.mod + go.sum | 2 | 模块初始化 |
| IDL 文件（common） | 7 | 1 service.thrift + 6 domain.thrift |
| common 注册文件 | 3 | namesp.go + rpc_client.go + go.work |
| gen 修正 | 1 | gorm_gen.go 修正 DB 引用 |
| **合计** | **13** | |

### 5.3 自动生成的文件

| 类别 | 估算文件数 | 说明 |
|------|-----------|------|
| kitex_gen | ~15 | 7 domain + service + k-文件 + walletservice/ |
| gorm_gen | ~20 | 9 model + 9 query + gen.go + repo/ |
| **合计** | **~35** | 禁止手动修改 |

### 5.4 总览

```
手动编写        30 个 Go 文件
初始化/修改     13 个文件（含 7 thrift + 3 注册 + go.mod 等）
自动生成       ~35 个文件（kitex_gen + gorm_gen）
─────────────────────────
总计          ~78 个文件（含自动生成）
```

---

## 六、层次调用关系图

```
外部请求（RPC/Cron）
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  handler.go — 37 个方法，纯委托转发，零业务逻辑                         │
│  WalletServiceImpl.CreditWallet() → walletSer.CreditWallet()       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       ▼                     ▼                     ▼
┌────────────┐     ┌────────────────┐     ┌──────────────┐
│ ser/       │     │ ser/           │     │ ser/         │
│ wallet.go  │     │ deposit.go     │     │ task.go      │
│ currency.go│     │ withdraw.go    │     │ (定时任务)    │
│ exchange.go│     │ record.go      │     │              │
└─────┬──────┘     └───────┬────────┘     └──────┬───────┘
      │                    │                     │
      │        ┌───────────┼───────────┐         │
      ▼        ▼           ▼           ▼         ▼
┌──────────────────────────────────────────────────────┐
│  rep/ — 数据访问层（9 个 Repo 文件）                     │
│  使用 GORM Gen 查询构建器（query.Q.xxx）                 │
│  事务操作在此层完成（query.Q.Transaction）                │
└─────────────────────────┬────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────┐
│  gorm_gen/ — 自动生成的 Model + Query + Repo          │
│  ↕ TiDB                                              │
└──────────────────────────────────────────────────────┘

      ║ 同时                    ║ 同时
      ▼                        ▼
┌──────────────┐        ┌──────────────┐
│ cache/       │        │ 外部 RPC     │
│ wallet.go    │        │ ser-kyc      │
│ currency.go  │        │ ser-user     │
│ withdraw.go  │        │ ser-finance  │
│ ↕ Redis      │        │ (via common/ │
└──────────────┘        │  rpc client) │
                        └──────────────┘
```

**调用方向**：handler → ser → rep + cache + external RPC

**不允许的调用**：
- handler 直接调 rep（必须经过 ser）
- ser 文件之间互相调用（保持域独立，共享逻辑下沉到 rep 或 utils）
- rep 调 ser（禁止反向依赖）
- cache 调 ser 或 rep（cache 是被动调用的工具层）

---

## 七、功能域 → 文件映射全景图

将 46 个接口完整映射到文件层：

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                          46 个接口的文件落地分布                                       │
│                                                                                    │
│  ┌─ 钱包核心域 ──────────────────────────────────────────────────────────────────┐  │
│  │ IDL: wallet.thrift   Ser: wallet.go   Rep: wallet_account.go                │  │
│  │                                            wallet_transaction.go            │  │
│  │                                                                              │  │
│  │ R-1 CreditWallet      ← 充值入账/补单/加款/奖金                               │  │
│  │ R-2 DebitWallet        ← 减款/指定钱包扣款                                    │  │
│  │ R-3 FreezeBalance      ← 提现冻结                                            │  │
│  │ R-4 UnfreezeBalance    ← 提现驳回解冻                                         │  │
│  │ R-5 DeductFrozen       ← 出款成功终态扣款                                     │  │
│  │ R-6 QueryBalance       ← 财务模块查余额                                       │  │
│  │ R-7 DebitByPriority    ← 消费优先级扣款（先中心后奖励）                          │  │
│  │ R-12 BatchQueryBalance ← 批量余额查询                                         │  │
│  │ C-1 GetBalance         ← C端余额展示                                          │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 币种配置域 ──────────────────────────────────────────────────────────────────┐  │
│  │ IDL: currency.thrift  Ser: currency.go  Rep: currency_config.go             │  │
│  │                                              exchange_rate_log.go           │  │
│  │                                                                              │  │
│  │ B-1 GetCurrencyList    ← B端币种列表                                          │  │
│  │ B-2 EditCurrency       ← B端编辑币种（浮动/图标/阈值/状态）                      │  │
│  │ B-3 GetCurrencyDetail  ← B端币种详情                                          │  │
│  │ B-4 SetBaseCurrency    ← 设置基准币种（一次性）                                 │  │
│  │ B-5 GetRateLogList     ← B端汇率日志列表                                      │  │
│  │ B-6 GetRateLogDetail   ← B端汇率日志详情                                      │  │
│  │ R-8 GetCurrencyConfig  ← RPC供他方（其他模块获取币种配置）                       │  │
│  │ R-9 GetExchangeRate    ← RPC供他方（获取指定币种汇率）                           │  │
│  │ R-10 ConvertAmount     ← RPC供他方（金额换算）                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 充值域 ──────────────────────────────────────────────────────────────────────┐  │
│  │ IDL: deposit.thrift   Ser: deposit.go   Rep: deposit_order.go               │  │
│  │                                                                              │  │
│  │ C-2 GetDepositMethods      ← 充值方式列表（RPC调财务 GetPaymentMethods）       │  │
│  │ C-3 CreateDepositOrder     ← 创建充值订单（RPC调财务 MatchAndCreateChannel）    │  │
│  │ C-4 GetUSDTDepositInfo     ← USDT充值地址+二维码                              │  │
│  │ C-14 CheckDuplicateDeposit ← 重复转账检测（3笔拦截）                           │  │
│  │ C-15 GetBonusActivities    ← 奖金活动列表                                     │  │
│  │ C-16 GetDepositOrderStatus ← 充值订单状态查询                                  │  │
│  │ C-19 CancelDepositOrder    ← 取消充值订单                                     │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 提现域 ──────────────────────────────────────────────────────────────────────┐  │
│  │ IDL: withdraw.thrift  Ser: withdraw.go  Rep: withdraw_order.go              │  │
│  │                                              withdraw_account.go            │  │
│  │                                              audit_requirement.go           │  │
│  │                                                                              │  │
│  │ C-7 GetWithdrawMethods     ← 提现方式列表（含KYC过滤+路由规则）                 │  │
│  │ C-8 CreateWithdrawOrder    ← 创建提现订单（校验+冻结+建单）                     │  │
│  │ C-9 GetWithdrawAccounts    ← 提现账户列表                                     │  │
│  │ C-10 SaveWithdrawAccount   ← 保存提现账户（首次+姓名匹配）                      │  │
│  │ C-17 GetWithdrawLimits     ← 提现限额查询（日限/单笔最低/手续费）               │  │
│  │ C-18 GetDepositHistory     ← 历史充值方式（路由规则依据）                        │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 兑换域 ──────────────────────────────────────────────────────────────────────┐  │
│  │ IDL: exchange.thrift  Ser: exchange.go  Rep: exchange_order.go              │  │
│  │                                                                              │  │
│  │ C-5 PreviewExchange        ← 兑换预览（实时汇率+赠送金额+到账金额）             │  │
│  │ C-6 ExecuteExchange        ← 执行兑换（法币扣减+BSB入账+赠送入奖励钱包）         │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 记录域 ──────────────────────────────────────────────────────────────────────┐  │
│  │ IDL: record.thrift    Ser: record.go    Rep: wallet_transaction.go(复用)     │  │
│  │                                              deposit_order.go(复用)          │  │
│  │                                              withdraw_order.go(复用)         │  │
│  │                                              exchange_order.go(复用)         │  │
│  │                                                                              │  │
│  │ C-11 GetTransactionList    ← 账变记录列表（充值/提现/兑换/奖励 Tab）            │  │
│  │ C-12 GetTransactionDetail  ← 账变记录详情                                     │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 定时任务 ────────────────────────────────────────────────────────────────────┐  │
│  │ 无IDL（内部实现）  Ser: task.go   Rep: currency_config.go(复用)               │  │
│  │                                       exchange_rate_log.go(复用)             │  │
│  │                                       deposit_order.go(复用)                 │  │
│  │                                       wallet_account.go(复用)                │  │
│  │                                       wallet_transaction.go(复用)            │  │
│  │                                                                              │  │
│  │ T-1 ExchangeRateUpdate     ← 汇率定时更新（调3方API→比较偏差→更新平台汇率）     │  │
│  │ T-2 DepositOrderTimeout    ← 充值订单超时关闭（扫描超期待支付订单）              │  │
│  │ T-3 BalanceReconciliation  ← 余额对账（流水累计 vs 账户余额，检测不一致）        │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 调用外部RPC（不在我方文件中定义，通过 common/rpc client 调用）───────────────┐  │
│  │ O-1 ser-kyc   QueryKycStatus  ← withdraw.go 内部调用                        │  │
│  │ O-2 ser-user  GetUserInfo     ← withdraw.go 内部调用                        │  │
│  │ O-3 ser-finance GetPaymentMethods        ← deposit.go 内部调用              │  │
│  │ O-4 ser-finance MatchAndCreateChannel    ← deposit.go 内部调用              │  │
│  │ O-5 ser-finance InitiateChannelPayout    ← withdraw.go 内部调用             │  │
│  │ O-6 ser-finance GetChannelOrderStatus    ← deposit.go/withdraw.go 内部调用  │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、与现有服务的规模对比

| 对比维度 | ser-app | ser-user | ser-live | **ser-wallet** |
|----------|---------|----------|----------|----------------|
| IDL 方法数 | 21 | 34 | 45 | **37** |
| IDL 域文件数 | 4 | 13 | 8 | **6** |
| handler.go 方法数 | 21 | 34 | 45 | **37** |
| ser/ 文件数 | 5 | ~12 | ~8 | **7** |
| rep/ 文件数 | ~3 | ~10 | ~6 | **9** |
| cache/ 文件数 | 1 | ~3 | ~2 | **3** |
| enum/ 文件数 | 4 | ~6 | ~4 | **5** |
| 数据表数 | 0* | ~8 | ~6 | **9** |
| 手动 Go 文件数 | ~14 | ~35 | ~24 | **30** |

> *ser-app 通过 ser-bcom 共享数据库

**结论**：ser-wallet 的规模介于 ser-user（最大）和 ser-app（最小）之间，与 ser-live 最接近。30 个手动文件、9 张表、7 个 IDL 文件，属于中等偏上的服务体量。

---

## 九、开发阶段与文件创建顺序

将 30 个文件按 7 个开发阶段排列，展示每个阶段需要创建的文件：

### Phase 1 — 工程搭建

```
本阶段创建 / 修改 ──────────────────────────────────

[新建 - ser-wallet 内]
  go.mod                       ← 模块初始化
  main.go                      ← 服务入口骨架
  handler.go                   ← 37 个空方法桩
  internal/cfg/mysql.go        ← DB 初始化
  internal/cfg/redis.go        ← Redis 初始化
  internal/errs/code.go        ← 错误码定义（全部）
  internal/utils/order_no.go   ← 订单号生成

[新建 - common 仓库]
  common/idl/ser-wallet/ (7 thrift 文件)

[修改 - common 仓库]
  go.work                      ← 添加 ./ser-wallet
  namesp.go                    ← 添加 3 个常量
  rpc_client.go                ← 添加 WalletClient()

[修正]
  internal/gen/gorm_gen.go     ← 修正 DB 引用

[自动生成]
  运行 gen_kitex.sh            → kitex_gen 生成
  运行 gorm_gen.go             → gorm_gen 生成
  创建 9 张数据表              → DDL 执行
```

### Phase 2 — 币种配置 + 汇率

```
internal/enum/currency_type.go
internal/enum/enum_registry.go
internal/ser/currency.go
internal/rep/currency_config.go
internal/rep/exchange_rate_log.go
internal/cache/currency.go
internal/ser/task.go            ← T-1 汇率更新部分
```

### Phase 3 — 钱包核心

```
internal/enum/wallet_type.go
internal/enum/transaction_type.go
internal/ser/wallet.go
internal/rep/wallet_account.go
internal/rep/wallet_transaction.go
internal/cache/wallet.go
```

### Phase 4 — C端基础功能

```
internal/ser/exchange.go
internal/ser/record.go
internal/rep/exchange_order.go
internal/enum/order_status.go    ← 兑换状态部分
```

### Phase 5 — 充值流程

```
internal/ser/deposit.go
internal/rep/deposit_order.go
internal/ser/task.go             ← T-2 订单超时部分
internal/enum/order_status.go    ← 充值状态部分（补充）
```

### Phase 6 — 提现流程

```
internal/ser/withdraw.go
internal/rep/withdraw_order.go
internal/rep/withdraw_account.go
internal/rep/audit_requirement.go
internal/cache/withdraw.go
internal/enum/order_status.go    ← 提现状态部分（补充）
```

### Phase 7 — 联调验证

```
internal/ser/task.go             ← T-3 余额对账部分
（无新文件，全链路联调测试）
```

---

## 十、与前序文档的衔接关系

| 本文内容 | 来源文档 | 具体章节 |
|----------|---------|---------|
| 7 个 IDL 文件结构 | IDL设计/tmp-2.md | 第三章 文件清单 |
| 37 个方法的分配 | IDL设计/tmp-2.md | 第四~九章 各域方法定义 |
| 46 个接口编号映射 | 接口评估/tmp-2.md | 全文（C-1~C-19, B-1~B-6, R-1~R-12, O-1~O-6, T-1~T-3） |
| 9 张表→9 个 rep 文件 | 表结构设计/tmp-2.md | 第一章 全局表结构地图 |
| 7 阶段开发计划 | 实现思路/tmp-2.md | 第十一章 阶段建议 |
| handler 委托模式 | ser-app/handler.go | 实际代码（125行/21方法） |
| main.go 初始化模式 | ser-app/main.go | 实际代码（59行） |
| Service 构造模式 | ser-app/internal/ser/banner.go | NewBannerService 模式 |
| Repo 查询模式 | ser-app/internal/rep/banner.go | GORM Gen 构建器模式 |
| Cache 模式 | ser-app/internal/cache/banner.go | nil 未命中 + 空结果防穿透 |
| 错误码分段模式 | ser-app/internal/errs/code.go | 6011000+iota 分段模式 |
| 枚举注册模式 | ser-app/internal/enum/enum_registry.go | EnumsOptionsRegistry 模式 |
| gorm_gen 生成模式 | ser-app/internal/gen/gorm_gen.go | GenCodeWithAll 调用 |
