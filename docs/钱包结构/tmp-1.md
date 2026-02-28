# ser-wallet 工程目录结构设计

> 设计时间：2026-02-27
> 定位：不写一行代码，只回答一个问题——ser-wallet 这个工程长什么样
> 设计原则：
> - **严格遵循现有工程规范**（逐文件对照 ser-app/ser-item 的实际模式）
> - **每个文件都有需求依据**（不凭空添加）
> - **图示 + 表格 + 文字分层讲解**（tree图看全貌，表格看对照，文字看原因）
> 设计依据：
> - 现有工程代码级扫描：ser-app（25个Go文件）、ser-item（31个Go文件）的实际目录结构
> - IDL设计：8个thrift文件、30个RPC方法
> - 表结构设计：9张必须表 + 14种枚举
> - 架构设计：四层分层架构 + 7个ADR决策

---

## 目录

1. [30秒速览：一张图看全貌](#一30秒速览)
2. [根目录文件详解](#二根目录文件详解)
3. [internal/cfg/ — 基础设施初始化层](#三internalcfg基础设施初始化层)
4. [internal/errs/ — 错误码定义层](#四internalerrs错误码定义层)
5. [internal/enum/ — 业务枚举常量层](#五internalenum业务枚举常量层)
6. [internal/ser/ — 服务逻辑层（核心）](#六internalser服务逻辑层)
7. [internal/rep/ — 数据仓库层](#七internalrep数据仓库层)
8. [internal/cache/ — 缓存操作层](#八internalcache缓存操作层)
9. [internal/utils/ — 工具函数层](#九internalutils工具函数层)
10. [internal/gen/ — 代码生成入口](#十internalgen代码生成入口)
11. [internal/gorm_gen/ — 自动生成代码（勿手编）](#十一internalgorm_gen自动生成代码)
12. [工程外部触点：IDL + 网关 + common](#十二工程外部触点)
13. [文件与需求的映射关系](#十三文件与需求的映射关系)
14. [文件与实现阶段的映射关系](#十四文件与实现阶段的映射关系)
15. [与现有工程的模式对照](#十五与现有工程的模式对照)

---

## 一、30秒速览

### 1.1 完整目录树

```
ser-wallet/
│
├── main.go                                     # 服务入口：初始化 → 依赖注入 → 启动Kitex
├── handler.go                                  # RPC委托层：30个方法 → 6个Service
├── go.mod                                      # Go模块定义
├── .gitignore                                  # Git忽略规则
├── .golangci.yml                               # 代码检查配置
│
├── internal/                                   # 核心业务代码（不对外暴露）
│   │
│   ├── cfg/                                    #【基础设施初始化】
│   │   ├── mysql.go                            #   TiDB连接 + GORM Gen全局Q绑定
│   │   └── redis.go                            #   Redis连接
│   │
│   ├── errs/                                   #【错误码】
│   │   └── code.go                             #   6022000起，按模块分段
│   │
│   ├── enum/                                   #【业务枚举】
│   │   ├── currency.go                         #   币种类型、币种状态
│   │   ├── wallet.go                           #   子钱包类型、流水类型、流水方向
│   │   ├── deposit.go                          #   充值方式、充值状态
│   │   ├── withdraw.go                         #   提现方式、提现状态、账户类型
│   │   ├── exchange.go                         #   兑换状态
│   │   └── enum_registry.go                    #   枚举下拉选项注册表
│   │
│   ├── ser/                                    #【服务逻辑层 — 核心业务】
│   │   ├── currency_service.go                 #   币种配置：CRUD + 汇率查询 + 对外RPC
│   │   ├── wallet_service.go                   #   钱包核心：余额概览 + 6种原子操作
│   │   ├── deposit_service.go                  #   充值：配置查询 + 下单 + 状态查询
│   │   ├── withdraw_service.go                 #   提现：配置 + 账户管理 + 下单
│   │   ├── exchange_service.go                 #   兑换：预览 + 执行
│   │   └── record_service.go                   #   记录：分页 + 详情（四类Tab路由）
│   │
│   ├── rep/                                    #【数据仓库层 — DB操作】
│   │   ├── currency.go                         #   t_currency 查询
│   │   ├── rate_log.go                         #   t_rate_log 查询
│   │   ├── wallet_account.go                   #   t_wallet_account 查询（含CAS乐观锁）
│   │   ├── wallet_flow.go                      #   t_wallet_flow 插入（含幂等唯一索引）
│   │   ├── deposit_order.go                    #   t_deposit_order 查询
│   │   ├── withdraw_order.go                   #   t_withdraw_order 查询
│   │   ├── exchange_order.go                   #   t_exchange_order 查询
│   │   ├── freeze_record.go                    #   t_freeze_record 查询
│   │   └── withdraw_account.go                 #   t_withdraw_account 查询
│   │
│   ├── cache/                                  #【缓存操作层 — Redis】
│   │   ├── currency.go                         #   币种配置缓存（高读低写）
│   │   └── rate.go                             #   汇率缓存（定时刷新）
│   │
│   ├── utils/                                  #【工具函数】
│   │   └── decimal.go                          #   shopspring/decimal 金额运算封装
│   │
│   ├── gen/                                    #【代码生成入口】
│   │   └── gorm_gen.go                         #   GORM Gen代码生成（go run触发）
│   │
│   └── gorm_gen/                               #【自动生成 — .gitignore忽略】
│       ├── model/                              #   GORM模型（9个.gen.go）
│       │   ├── currency.gen.go                 #     t_currency → Currency struct
│       │   ├── rate_log.gen.go                 #     t_rate_log → RateLog struct
│       │   ├── wallet_account.gen.go           #     t_wallet_account → WalletAccount struct
│       │   ├── wallet_flow.gen.go              #     t_wallet_flow → WalletFlow struct
│       │   ├── deposit_order.gen.go            #     t_deposit_order → DepositOrder struct
│       │   ├── withdraw_order.gen.go           #     t_withdraw_order → WithdrawOrder struct
│       │   ├── exchange_order.gen.go           #     t_exchange_order → ExchangeOrder struct
│       │   ├── freeze_record.gen.go            #     t_freeze_record → FreezeRecord struct
│       │   └── withdraw_account.gen.go         #     t_withdraw_account → WithdrawAccount struct
│       ├── query/                              #   类型安全查询构造器
│       │   ├── gen.go                          #     Q全局变量 + SetDefault() + Transaction()
│       │   ├── currency.gen.go                 #     币种表查询构造器
│       │   ├── rate_log.gen.go                 #     汇率日志表查询构造器
│       │   ├── wallet_account.gen.go           #     钱包账户表查询构造器
│       │   ├── wallet_flow.gen.go              #     流水表查询构造器
│       │   ├── deposit_order.gen.go            #     充值订单表查询构造器
│       │   ├── withdraw_order.gen.go           #     提现订单表查询构造器
│       │   ├── exchange_order.gen.go           #     兑换订单表查询构造器
│       │   ├── freeze_record.gen.go            #     冻结记录表查询构造器
│       │   └── withdraw_account.gen.go         #     提现账户表查询构造器
│       └── repo/                               #   自动生成的通用CRUD
│           ├── currencyr/currencyr.go
│           ├── ratelogr/ratelogr.go
│           ├── walletaccountr/walletaccountr.go
│           ├── walletflowr/walletflowr.go
│           ├── depositoorderr/depositoorderr.go
│           ├── withdraworderr/withdraworderr.go
│           ├── exchangeorderr/exchangeorderr.go
│           ├── freezerecordr/freezerecordr.go
│           └── withdrawaccountr/withdrawaccountr.go
│
└── tmp/                                        # 产品文档（非代码，仅参考）
    ├── 多币种钱包产品原型/                       #   C端原型截图（30张）
    ├── 多币种钱包需求文档/                       #   C端需求文档截图（46张）
    ├── 币种配置产品原型/                         #   B端币种原型（3张）
    ├── 币种配置需求文档/                         #   B端币种需求文档（24张）
    ├── 财务管理产品原型/                         #   财务管理原型（26张）
    └── 财务管理需求文档/                         #   财务管理需求文档（25张）
```

### 1.2 数量统计

| 指标 | 数量 | 说明 |
|------|------|------|
| 手写Go文件 | **25个** | 需要人工编写的代码文件 |
| 自动生成Go文件 | ~28个 | gorm_gen/ 目录下，由工具自动生成 |
| 目录总数 | 15个 | internal/ 下的分包数量 |
| 根目录配置文件 | 3个 | go.mod + .gitignore + .golangci.yml |

### 1.3 与现有工程规模对比

```
ser-app:   25 个Go文件，20个方法，2张表
ser-item:  31 个Go文件，10个方法，2张表
ser-wallet: 25 个Go文件，30个方法，9张表  ← 方法多，表多，但文件数合理
```

方法虽然是 ser-app 的1.5倍，但文件数相当——因为钱包的6个业务域边界清晰，每域一个 service + 一个 repo + 一个 enum，不需要碎文件。

---

## 二、根目录文件详解

### 2.1 目录树

```
ser-wallet/
├── main.go
├── handler.go
├── go.mod
├── .gitignore
└── .golangci.yml
```

### 2.2 逐文件说明

| 文件 | 职责 | 对标工程文件 | 内部做什么 |
|------|------|-------------|-----------|
| `main.go` | 服务启动入口 | ser-app/main.go | 初始化日志 → 加载ETCD → 初始化Redis → 创建6个Service实例 → 启动Kitex Server |
| `handler.go` | RPC方法委托 | ser-app/handler.go | 定义 `WalletServiceImpl` 结构体，持有6个Service指针，30个方法全部一行委托 |
| `go.mod` | Go模块定义 | ser-app/go.mod | `module ser-wallet`，依赖 common/pkg、common/rpc、shopspring/decimal |
| `.gitignore` | 忽略规则 | ser-app/.gitignore | 模板相同 + 忽略 `internal/gorm_gen/` |
| `.golangci.yml` | 代码检查 | ser-app/.golangci.yml | 工程统一模板，直接复制 |

### 2.3 main.go 内部结构示意

```
main()
  ├── lg.InitKLog()                          // 1. 日志
  ├── etcd.LoadAndWatch("/slg/")             // 2. ETCD配置
  ├── cfg.InitRedis()                        // 3. Redis
  ├── flag.String("port", ...)               // 4. 端口
  │
  ├── 创建6个Service实例：                     // 5. 手动依赖注入
  │   ├── currencyService  = ser.NewCurrencyService(cache)
  │   ├── walletService    = ser.NewWalletService(currencyService)    ← 依赖币种
  │   ├── depositService   = ser.NewDepositService(walletService, currencyService)
  │   ├── withdrawService  = ser.NewWithdrawService(walletService, currencyService)
  │   ├── exchangeService  = ser.NewExchangeService(walletService, currencyService)
  │   └── recordService    = ser.NewRecordService()
  │
  └── walletservice.NewServer(               // 6. 启动Kitex
        &WalletServiceImpl{...},
        rpc.InitRpcServerParams(namesp.EtcdWalletService, port)...,
      ).Run()
```

**关键设计点**：`currencyService` 被注入到其他4个Service中（进程内调用，不走网络），因为汇率查询、精度配置贯穿所有金额操作。`walletService` 封装了6种原子余额操作，被 deposit/withdraw/exchange 依赖。

### 2.4 handler.go 内部结构示意

```go
type WalletServiceImpl struct {
    currencyService  *ser.CurrencyService     // → 方法 1-9, 30
    walletService    *ser.WalletService        // → 方法 10, 22-29
    depositService   *ser.DepositService       // → 方法 11-13
    withdrawService  *ser.WithdrawService      // → 方法 16-19
    exchangeService  *ser.ExchangeService      // → 方法 14-15
    recordService    *ser.RecordService        // → 方法 20-21
}

// 30个方法全部一行委托，零业务逻辑。示例：
// func (s *WalletServiceImpl) CreateCurrency(ctx, req) { return s.currencyService.CreateCurrency(ctx, req) }
// func (s *WalletServiceImpl) WalletCredit(ctx, req)   { return s.walletService.WalletCredit(ctx, req) }
```

**为什么handler.go不写逻辑**：这是现有工程的硬规范。ser-app/handler.go 的20个方法全部是单行委托。handler只是IDL接口的"接线板"，保证 Service 层可以独立单元测试。

---

## 三、internal/cfg/ — 基础设施初始化层

### 3.1 目录树

```
internal/cfg/
├── mysql.go          # TiDB 连接初始化
└── redis.go          # Redis 连接初始化
```

### 3.2 文件说明

| 文件 | 对标 | 职责 | 核心机制 |
|------|------|------|---------|
| `mysql.go` | ser-app/internal/cfg/mysql.go | 读ETCD获取db_wallet连接串，初始化GORM，绑定 `query.SetDefault(db)` | `sync.Once` 保证只初始化一次 |
| `redis.go` | ser-app/internal/cfg/redis.go | 读ETCD获取Redis地址密码，初始化Redis客户端 | `sync.Once` 保证只初始化一次 |

**为什么只有2个文件**：这是工程统一模板。每个服务的 cfg/ 都是 mysql.go + redis.go，格式几乎相同，只有数据库名不同（此处为 `namesp.EtcdWalletDb` → `db_wallet`）。

---

## 四、internal/errs/ — 错误码定义层

### 4.1 目录树

```
internal/errs/
└── code.go           # 全部错误码定义
```

### 4.2 错误码段位分配

ser-wallet 的服务编号为 022，错误码前缀为 `6022`，按模块分段：

| 段位 | 范围 | 模块 | 预留数量 |
|------|------|------|---------|
| 币种配置 | 6022000 - 6022099 | currency | 100个 |
| 钱包核心 | 6022100 - 6022199 | wallet | 100个 |
| 充值 | 6022200 - 6022299 | deposit | 100个 |
| 提现 | 6022300 - 6022399 | withdraw | 100个 |
| 兑换 | 6022400 - 6022449 | exchange | 50个 |
| 记录 | 6022450 - 6022499 | record | 50个 |
| 预留 | 6022500 - 6022599 | 扩展 | 100个 |

**依据**：现有工程 ser-app 用 `6011` 前缀、ser-item 用 `6010` 前缀，均采用 iota 递增 + 按模块分段。钱包模块多，每段100个错误码给充值/提现等复杂流程预留充足空间。

### 4.3 code.go 内部结构示意

```
const (
    // ========== 币种配置模块 6022000-6022099 ==========
    CurrencyBaseError         = 6022000 + iota   // 基础错误
    CurrencyNotFound                              // 币种不存在
    CurrencyCodeDuplicate                         // 币种编码重复
    CurrencyCannotDisable                         // 有活跃用户不可禁用
    ...

    // ========== 钱包核心模块 6022100-6022199 ==========
    WalletBaseError           = 6022100 + iota
    WalletAccountNotFound                         // 账户不存在
    WalletInsufficientBalance                     // 余额不足
    WalletVersionConflict                         // 乐观锁冲突（需重试）
    WalletFlowDuplicate                           // 流水重复（幂等拦截）
    ...
)
```

---

## 五、internal/enum/ — 业务枚举常量层

### 5.1 目录树

```
internal/enum/
├── currency.go          # 币种类型、币种状态
├── wallet.go            # 子钱包类型、流水类型、流水方向
├── deposit.go           # 充值方式、充值状态
├── withdraw.go          # 提现方式、提现状态、提现账户类型
├── exchange.go          # 兑换状态
└── enum_registry.go     # 枚举选项注册表（B端下拉框用）
```

### 5.2 逐文件枚举清单

| 文件 | 定义的枚举 | 来源 |
|------|-----------|------|
| `currency.go` | CurrencyType（1法币/2虚拟币/3平台币）、CurrencyStatus（0禁用/1启用）、RateSourceType（1三方/2平台调整/3手动） | 表结构设计 §15 |
| `wallet.go` | WalletType（1中心/2奖励/3主播/4代理/5场馆）、FlowType（1充值/2提现/3兑换/4奖金/5人工加/6人工减/7冻结/8解冻/9扣款）、FlowDirection（1收入/2支出） | 表结构设计 §15 |
| `deposit.go` | DepositMethod（1银行转账/2电子钱包/3USDT）、DepositStatus（1待支付/2进行中/3成功/4失败/5超时） | 需求文档 §5 |
| `withdraw.go` | WithdrawMethod（1银行卡/2电子钱包/3USDT）、WithdrawStatus（1待审核/2审核中/3出款中/4成功/5失败/6驳回）、AccountType（1银行卡/2电子钱包/3USDT） | 需求文档 §7 |
| `exchange.go` | ExchangeStatus（1成功/2失败） | 需求文档 §6 |
| `enum_registry.go` | `EnumsOptionsRegistry` map — 注册所有枚举供 B端下拉框动态获取 | ser-app/enum/enum_registry.go 模式 |

### 5.3 每个文件内部包含什么

以 `wallet.go` 为例说明内部结构规范：

```
wallet.go 内部结构：
├── WalletType 常量组（const + iota）
├── WalletTypeLabel map（数字→中文名映射，日志/展示用）
├── FlowType 常量组
├── FlowTypeLabel map
├── FlowDirection 常量组
└── IsValidWalletType() / IsValidFlowType() 校验函数
```

**依据**：完全对照 ser-app/internal/enum/banner.go 的模式——常量定义 + Label map + 校验函数。

---

## 六、internal/ser/ — 服务逻辑层（核心）

### 6.1 目录树

```
internal/ser/
├── currency_service.go          # 币种配置服务
├── wallet_service.go            # 钱包核心服务（最重要）
├── deposit_service.go           # 充值服务
├── withdraw_service.go          # 提现服务
├── exchange_service.go          # 兑换服务
└── record_service.go            # 记录查询服务
```

**为什么是6个文件**：正好对应6个业务域。不多不少——每个文件一个 `XxxService` 结构体，不出现一个文件塞两个Service的情况，也不出现一个域拆成多个文件的碎片化。

### 6.2 逐文件详解

#### currency_service.go — 币种配置服务

| 属性 | 说明 |
|------|------|
| **结构体** | `CurrencyService` |
| **依赖** | `*rep.CurrencyRepo` + `*rep.RateLogRepo` + `*cache.CurrencyCache` + `*cache.RateCache` |
| **对应IDL方法** | PageCurrency、CreateCurrency、UpdateCurrency、ToggleCurrencyStatus、SetBaseCurrency、DetailCurrency、PageRateLog、ListActiveCurrency、GetExchangeRate、GetCurrencyConfig |
| **方法数** | 10个（B端7个 + C端2个 + RPC 1个） |
| **核心职责** | 币种CRUD、汇率查询、币种配置缓存管理、对外提供币种信息 |
| **设计要点** | 被注入到其他4个Service，作为"计价引擎"角色 |

```
currency_service.go 方法清单：
├── NewCurrencyService(cache, rateCache) → 构造函数
│
├── B端方法（gate-back 调用）：
│   ├── PageCurrency()            → 币种分页列表
│   ├── CreateCurrency()          → 创建币种（校验编码唯一）
│   ├── UpdateCurrency()          → 编辑币种
│   ├── ToggleCurrencyStatus()    → 启禁用币种
│   ├── SetBaseCurrency()         → 设置基准币种（全局唯一）
│   ├── DetailCurrency()          → 币种详情
│   └── PageRateLog()             → 汇率变更日志分页
│
├── C端方法（gate-font 调用）：
│   ├── ListActiveCurrency()      → 可用币种列表（状态=启用）
│   └── GetExchangeRate()         → 获取两币种间汇率
│
└── RPC方法（ser-finance 调用）：
    └── GetCurrencyConfig()       → 查币种配置信息
```

#### wallet_service.go — 钱包核心服务（最重要的文件）

| 属性 | 说明 |
|------|------|
| **结构体** | `WalletService` |
| **依赖** | `*rep.WalletAccountRepo` + `*rep.WalletFlowRepo` + `*rep.FreezeRecordRepo` + `*CurrencyService` |
| **对应IDL方法** | GetWalletOverview、WalletCredit、WalletCreditReward、WalletFreeze、WalletUnfreeze、WalletDeduct、WalletManualAdd、WalletManualSub、GetUserBalance |
| **方法数** | 9个（C端1个 + RPC 8个） |
| **核心职责** | **余额变动的唯一入口** — 所有金额的增、减、冻结、解冻都且只通过这个Service |
| **设计要点** | 6种原子操作，每次操作 = 一个事务内同时更新 wallet_account(CAS) + 插入 wallet_flow(幂等) |

```
wallet_service.go 方法清单：
├── NewWalletService(currencyService) → 构造函数
│
├── C端方法：
│   └── GetWalletOverview()           → 用户钱包余额概览（5种子钱包汇总）
│
├── RPC方法（资金操作 — 仅限 ser-finance 调用）：
│   ├── WalletCredit()                → 入账（充值到账/补单）
│   ├── WalletCreditReward()          → 奖金入账（→奖励钱包）
│   ├── WalletFreeze()                → 冻结（提现发起时）
│   ├── WalletUnfreeze()              → 解冻归还（审核驳回/出款失败）
│   ├── WalletDeduct()                → 确认扣款（提现成功）
│   ├── WalletManualAdd()             → 人工加款
│   └── WalletManualSub()             → 人工减款
│
├── RPC方法（查询）：
│   └── GetUserBalance()              → 查指定用户余额
│
└── 内部私有方法（不对外暴露）：
    ├── ensureAccount()               → 按需自动开户（首次操作时创建子钱包）
    ├── doCredit()                    → 原子加款（CAS更新余额 + 写流水）
    ├── doDebit()                     → 原子减款
    ├── doFreeze()                    → 原子冻结（可用→冻结）
    ├── doUnfreeze()                  → 原子解冻（冻结→可用）
    └── doDeduct()                    → 原子扣款（冻结→扣除）
```

**为什么这是最重要的文件**：

架构设计约束1明确规定——所有能修改 `t_wallet_account` 余额字段的代码，只存在于 `WalletService` 的6种原子操作方法中。其他 Service（deposit/withdraw/exchange）必须通过调用这些方法来完成资金变动。这保证并发控制、流水记录、幂等检查不会被绕过。

#### deposit_service.go — 充值服务

| 属性 | 说明 |
|------|------|
| **结构体** | `DepositService` |
| **依赖** | `*rep.DepositOrderRepo` + `*WalletService` + `*CurrencyService` |
| **对应IDL方法** | GetDepositConfig、CreateDepositOrder、GetDepositStatus |
| **方法数** | 3个（全部C端） |
| **核心职责** | 充值配置展示、创建充值订单（含重复转账拦截）、查询充值状态 |

```
deposit_service.go 方法清单：
├── NewDepositService(walletService, currencyService) → 构造函数
├── GetDepositConfig()        → 充值页配置（充值方式+快捷金额+汇率）
├── CreateDepositOrder()      → 创建充值订单（含重复转账拦截逻辑）
└── GetDepositStatus()        → 充值状态查询
```

**内部逻辑重点**：`CreateDepositOrder` 内含重复转账拦截——查询同用户+同币种+同金额的待处理订单数量，返回拦截类型（0正常/1软拦截/2硬拦截），由前端决定弹窗行为。

#### withdraw_service.go — 提现服务

| 属性 | 说明 |
|------|------|
| **结构体** | `WithdrawService` |
| **依赖** | `*rep.WithdrawOrderRepo` + `*rep.WithdrawAccountRepo` + `*WalletService` + `*CurrencyService` |
| **对应IDL方法** | GetWithdrawConfig、GetWithdrawAccount、SaveWithdrawAccount、CreateWithdrawOrder |
| **方法数** | 4个（全部C端） |
| **核心职责** | 提现配置展示、提现账户管理（银行卡/电子钱包/USDT地址）、创建提现订单（冻结余额） |

```
withdraw_service.go 方法清单：
├── NewWithdrawService(walletService, currencyService) → 构造函数
├── GetWithdrawConfig()       → 提现页配置（KYC状态+提现方式+限额+手续费）
├── GetWithdrawAccount()      → 查询已保存的提现账户
├── SaveWithdrawAccount()     → 保存提现账户（首次填写，不可自行修改）
└── CreateWithdrawOrder()     → 创建提现订单（校验限额→冻结余额→通知finance审核）
```

**内部逻辑重点**：
- `SaveWithdrawAccount` 含 KYC 姓名一致性校验（需RPC调 ser-kyc 查姓名）
- `CreateWithdrawOrder` 的核心动作 = 调 `walletService.WalletFreeze()` 冻结对应金额，然后通过MQ通知 ser-finance 开始审核

#### exchange_service.go — 兑换服务

| 属性 | 说明 |
|------|------|
| **结构体** | `ExchangeService` |
| **依赖** | `*rep.ExchangeOrderRepo` + `*WalletService` + `*CurrencyService` |
| **对应IDL方法** | PreviewExchange、CreateExchange |
| **方法数** | 2个（全部C端） |
| **核心职责** | 兑换预览（试算金额）+ 执行兑换（法币→BSB单向） |

```
exchange_service.go 方法清单：
├── NewExchangeService(walletService, currencyService) → 构造函数
├── PreviewExchange()         → 预览试算（输入法币金额→返回可得BSB+赠送BSB）
└── CreateExchange()          → 执行兑换（法币减款→BSB加款→写订单→写流水）
```

**内部逻辑重点**：`CreateExchange` 是内部闭环操作（不涉及三方/审核），在同一事务中：扣法币余额 + 加BSB余额 + 写兑换订单 + 写两条流水。这是最简单的业务闭环，适合作为Phase 4验证余额体系的用例。

#### record_service.go — 记录查询服务

| 属性 | 说明 |
|------|------|
| **结构体** | `RecordService` |
| **依赖** | `*rep.DepositOrderRepo` + `*rep.WithdrawOrderRepo` + `*rep.ExchangeOrderRepo` |
| **对应IDL方法** | PageRecord、DetailRecord |
| **方法数** | 2个（全部C端） |
| **核心职责** | 四类记录的统一分页查询（充值/提现/兑换/奖励），按 recordType 路由到不同表 |

```
record_service.go 方法清单：
├── NewRecordService() → 构造函数
├── PageRecord()       → 统一分页（根据recordType路由到不同订单表）
├── DetailRecord()     → 统一详情
└── mapToRecordItem()  → 私有：将不同订单模型统一映射为 RecordItem
```

### 6.3 Service层依赖关系图

```
                     ┌─────────────────────┐
                     │  CurrencyService    │  ← 被所有业务Service依赖
                     │  (计价引擎)          │
                     └───────┬─────────────┘
                             │ 进程内注入
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
   ┌──────────────┐ ┌───────────────┐ ┌──────────────┐
   │DepositService│ │WithdrawService│ │ExchangeService│
   └──────┬───────┘ └──────┬────────┘ └──────┬───────┘
          │                │                  │
          ▼                ▼                  ▼
     ┌─────────────────────────────────────────┐
     │          WalletService                   │  ← 余额变动唯一入口
     │  (6种原子操作：credit/freeze/deduct/...) │
     └─────────────────────────────────────────┘

   ┌──────────────┐
   │RecordService │  ← 纯查询，不依赖 WalletService
   └──────────────┘
```

**核心规则**：所有资金流转都经过 `WalletService` 的原子操作 → 保证并发安全 + 流水完整 + 幂等。

---

## 七、internal/rep/ — 数据仓库层

### 7.1 目录树

```
internal/rep/
├── currency.go              # t_currency
├── rate_log.go              # t_rate_log
├── wallet_account.go        # t_wallet_account（含CAS）
├── wallet_flow.go           # t_wallet_flow（含幂等）
├── deposit_order.go         # t_deposit_order
├── withdraw_order.go        # t_withdraw_order
├── exchange_order.go        # t_exchange_order
├── freeze_record.go         # t_freeze_record
└── withdraw_account.go      # t_withdraw_account
```

**对应规则**：每张数据库表对应一个 repo 文件。9张表 → 9个文件，一一对应。

### 7.2 逐文件说明

| 文件 | 对应表 | 结构体 | 核心操作 | 特殊机制 |
|------|--------|--------|---------|---------|
| `currency.go` | t_currency | `CurrencyRepo` | 分页、按code查、按状态筛选、更新 | 无 |
| `rate_log.go` | t_rate_log | `RateLogRepo` | 批量插入、分页查询 | 仅追加写入（日志表不更新） |
| `wallet_account.go` | t_wallet_account | `WalletAccountRepo` | 按(userId,currency,type)查、余额更新 | **CAS乐观锁**（`WHERE version = ?` + `SET version = version + 1`） |
| `wallet_flow.go` | t_wallet_flow | `WalletFlowRepo` | 插入流水、按用户分页 | **唯一索引幂等**（`related_order_no + flow_type` 重复插入会报错） |
| `deposit_order.go` | t_deposit_order | `DepositOrderRepo` | 创建订单、按状态查、重复转账计数 | `CountPending()` 用于重复转账拦截 |
| `withdraw_order.go` | t_withdraw_order | `WithdrawOrderRepo` | 创建订单、状态更新、按用户分页 | 今日已提金额聚合查询 |
| `exchange_order.go` | t_exchange_order | `ExchangeOrderRepo` | 创建订单、按用户分页 | 无复杂机制 |
| `freeze_record.go` | t_freeze_record | `FreezeRecordRepo` | 创建冻结、按orderNo查、状态更新 | 关联 wallet_account 的冻结金额 |
| `withdraw_account.go` | t_withdraw_account | `WithdrawAccountRepo` | 按用户查、按类型筛选、创建 | 姓名比对辅助查询 |

### 7.3 关键Repo的特殊操作说明

**wallet_account.go — CAS乐观锁（最核心的并发控制）**：

```
UpdateBalanceWithCAS 方法的SQL语义：

UPDATE t_wallet_account
SET available_balance = ?,
    frozen_balance = ?,
    version = version + 1,
    update_at = ?
WHERE id = ?
  AND version = ?           ← CAS条件
  AND delete_at = 0

如果 affected_rows = 0 → 说明被其他请求抢先修改了 → 返回 WalletVersionConflict 错误
Service 层收到此错误后重试（最多3次）
```

**wallet_flow.go — 唯一索引幂等**：

```
InsertFlow 方法遇到唯一索引冲突时：

INSERT INTO t_wallet_flow (...) VALUES (...)
→ 如果 (related_order_no, flow_type) 重复 → MySQL报 Duplicate Entry
→ Repo 层捕获此错误 → 返回 WalletFlowDuplicate 错误
→ Service 层收到此错误 → 判定为幂等拦截，返回成功（不重复执行）
```

这两个机制是架构设计 ADR-03（乐观锁）和 ADR-04（流水幂等）的代码落地位置。

---

## 八、internal/cache/ — 缓存操作层

### 8.1 目录树

```
internal/cache/
├── currency.go          # 币种配置缓存
└── rate.go              # 汇率缓存
```

### 8.2 文件说明

| 文件 | 结构体 | 缓存什么 | 读写频率 | TTL策略 |
|------|--------|---------|---------|---------|
| `currency.go` | `CurrencyCache` | 启用状态的币种列表（C端高频读取）、单个币种配置 | 读：极高（每次充值/提现/兑换都要查）；写：极低（B端修改币种才变） | 5-10分钟，B端修改后主动清除 |
| `rate.go` | `RateCache` | 各币种对BSB的当前汇率（C端每次展示都要查） | 读：极高；写：定时任务每次拉取后刷新 | 与汇率更新周期对齐（如1小时） |

**为什么只有2个缓存文件**：

不是所有数据都需要缓存。钱包余额不缓存（必须实时查DB保证一致性），订单数据不缓存（写多读少）。只有「配置类数据」（高读低写）才值得缓存，即币种配置和汇率。

**对照**：ser-app 也只有1个缓存文件（banner.go），说明缓存应该精简、不滥用。

---

## 九、internal/utils/ — 工具函数层

### 9.1 目录树

```
internal/utils/
└── decimal.go           # 高精度金额运算封装
```

### 9.2 文件说明

| 文件 | 职责 | 为什么需要 |
|------|------|-----------|
| `decimal.go` | 封装 `shopspring/decimal` 的常用金额运算：字符串→Decimal 转换、加减乘除、四舍五入到指定精度、比较、格式化输出 | 架构设计 ADR-06 决定用 `DECIMAL(30,8)` + `string` + `shopspring/decimal` 三位一体方案。所有金额运算禁止 float64，必须走 decimal 库 |

```
decimal.go 预计包含的工具函数：
├── ParseAmount(s string) → decimal.Decimal        // string → Decimal（含错误处理）
├── Add(a, b string) → string                      // 加法
├── Sub(a, b string) → string                      // 减法
├── Mul(a, b string) → string                      // 乘法（汇率换算）
├── Div(a, b string, precision int32) → string     // 除法（指定精度）
├── Compare(a, b string) → int                     // 比较（-1/0/1）
├── IsPositive(s string) → bool                    // 是否大于0
├── IsZero(s string) → bool                        // 是否为0
└── FormatAmount(s string, precision int32) → string // 格式化输出
```

**为什么放在 internal/utils/ 而不是 common/pkg/utils/**：

因为 `shopspring/decimal` 是 ser-wallet 新引入的依赖，目前其他服务不需要。如果后续其他服务也需要高精度金额运算，可以提升到 common 层。先放在 internal 降低对公共模块的影响。

---

## 十、internal/gen/ — 代码生成入口

### 10.1 目录树

```
internal/gen/
└── gorm_gen.go          # GORM Gen 代码生成入口
```

### 10.2 文件说明

| 文件 | 职责 | 触发方式 |
|------|------|---------|
| `gorm_gen.go` | 连接 db_wallet 数据库，扫描所有表，自动生成 model/ + query/ + repo/ 代码到 `internal/gorm_gen/` | `cd ser-wallet && go run internal/gen/gorm_gen.go` |

**当前状态**：此文件已存在但引用了 ser-app 的数据库配置（`namesp.EtcdAppDb`），需要改为 `namesp.EtcdWalletDb`。

---

## 十一、internal/gorm_gen/ — 自动生成代码

### 11.1 目录树

```
internal/gorm_gen/                     # 整个目录被 .gitignore 忽略
├── model/                             # 9个表 → 9个模型文件
│   ├── currency.gen.go
│   ├── rate_log.gen.go
│   ├── wallet_account.gen.go
│   ├── wallet_flow.gen.go
│   ├── deposit_order.gen.go
│   ├── withdraw_order.gen.go
│   ├── exchange_order.gen.go
│   ├── freeze_record.gen.go
│   └── withdraw_account.gen.go
├── query/                             # 1个总入口 + 9个查询构造器
│   ├── gen.go                         # Q全局变量、SetDefault()、Transaction()
│   ├── currency.gen.go
│   ├── rate_log.gen.go
│   ├── wallet_account.gen.go
│   ├── wallet_flow.gen.go
│   ├── deposit_order.gen.go
│   ├── withdraw_order.gen.go
│   ├── exchange_order.gen.go
│   ├── freeze_record.gen.go
│   └── withdraw_account.gen.go
└── repo/                              # 9个通用CRUD包
    ├── currencyr/currencyr.go
    ├── ratelogr/ratelogr.go
    ├── walletaccountr/walletaccountr.go
    ├── walletflowr/walletflowr.go
    ├── depositorderr/depositorderr.go
    ├── withdraworderr/withdraworderr.go
    ├── exchangeorderr/exchangeorderr.go
    ├── freezerecordr/freezerecordr.go
    └── withdrawaccountr/withdrawaccountr.go
```

### 11.2 三个子目录的角色

| 子目录 | 内容 | 谁使用 | 能不能手改 |
|--------|------|--------|-----------|
| `model/` | 表结构对应的Go struct（含gorm tag） | `rep/` 层的返回类型、`ser/` 层的数据传递 | **禁止手改**（重新生成会覆盖） |
| `query/` | 类型安全的查询构造器（`Q.WalletAccount.WithContext(ctx).Where(...)`） | `rep/` 层直接调用 | **禁止手改** |
| `repo/` | 自动生成的通用 CRUD 模板（Create/Update/Delete/GetPage...） | 可直接用，也可在 `rep/` 层写更定制化的查询 | **禁止手改** |

**与 internal/rep/ 的分工**：`gorm_gen/repo/` 是机器生成的通用操作，`internal/rep/` 是人工编写的业务定制查询（如 CAS 更新、聚合统计、复杂筛选）。两者互补而非替代。

---

## 十二、工程外部触点

ser-wallet 不是孤立的，它需要在工程中的其他位置注册和对接。以下是需要修改的外部文件，但**不属于 ser-wallet 目录内部**。

### 12.1 IDL文件（位于 common/idl/）

```
common/idl/ser-wallet/
├── service.thrift               # 主入口：定义 WalletService（30个方法）
├── currency.thrift              # 币种配置域 struct
├── wallet.thrift                # 钱包核心域 struct
├── deposit.thrift               # 充值域 struct
├── withdraw.thrift              # 提现域 struct
├── exchange.thrift              # 兑换域 struct
├── record.thrift                # 记录域 struct
└── wallet_rpc.thrift            # 跨服务RPC域 struct（供finance调用）
```

| 文件 | struct数量 | 对应方法 |
|------|-----------|---------|
| service.thrift | 0（纯方法定义） | 30个方法声明 |
| currency.thrift | ~10 | B端7 + C端2 |
| wallet.thrift | ~5 | C端1 |
| deposit.thrift | ~8 | C端3 |
| withdraw.thrift | ~10 | C端4 |
| exchange.thrift | ~5 | C端2 |
| record.thrift | ~6 | C端2 |
| wallet_rpc.thrift | ~15 | RPC 9 |

### 12.2 common模块修改点

| 文件 | 修改内容 |
|------|---------|
| `common/pkg/consts/namesp/namesp.go` | 新增 `EtcdWalletService`、`EtcdWalletPort`、`EtcdWalletDb` 常量 |
| `common/rpc/rpc_client.go` | 新增 `WalletClient()` 工厂函数（sync.Once懒初始化） |
| `go.work` | 新增 `./ser-wallet` 到 use 块 |

### 12.3 网关路由修改点

**gate-back（B端网关）**：

```
gate-back/biz/
├── handler/wallet/
│   └── currency_handler.go      # B端币种管理 handler（7个路由）
├── router/wallet/
│   └── wallet.go                # 路由注册：/api/wallet/currency/*
└── router/register.go           # 加一行：wallet.RouterWallet(r)
```

**gate-font（C端网关）**：

```
gate-font/biz/
├── handler/wallet/
│   ├── wallet_handler.go        # 钱包概览 handler
│   ├── deposit_handler.go       # 充值 handler（3个路由）
│   ├── withdraw_handler.go      # 提现 handler（4个路由）
│   ├── exchange_handler.go      # 兑换 handler（2个路由）
│   └── record_handler.go        # 记录 handler（2个路由）
├── router/wallet/
│   └── wallet.go                # 路由注册：/api/wallet/*
└── router/register.go           # 加一行：wallet.RouterWallet(r)
```

### 12.4 ETCD配置项

| Key | 值示例 | 说明 |
|-----|--------|------|
| `/slg/serv/wallet/port` | `9100` | ser-wallet 监听端口 |
| `/slg/serv/wallet/db` | `db_wallet` | 数据库名 |

---

## 十三、文件与需求的映射关系

### 13.1 需求功能 → 代码文件映射表

每个需求功能点由「哪些文件协同完成」，纵览数据流向：

| 需求功能 | enum/ | ser/ | rep/ | cache/ | utils/ | 表 |
|---------|-------|------|------|--------|--------|-----|
| B端创建币种 | currency.go | currency_service.go | currency.go | currency.go | - | t_currency |
| B端设置基准币种 | currency.go | currency_service.go | currency.go | currency.go | - | t_currency |
| B端汇率日志 | currency.go | currency_service.go | rate_log.go | - | - | t_rate_log |
| C端币种列表 | currency.go | currency_service.go | currency.go | currency.go | - | t_currency |
| C端钱包概览 | wallet.go | wallet_service.go | wallet_account.go | - | decimal.go | t_wallet_account |
| C端充值下单 | deposit.go | deposit_service.go | deposit_order.go | rate.go | decimal.go | t_deposit_order |
| 重复转账拦截 | deposit.go | deposit_service.go | deposit_order.go | - | - | t_deposit_order |
| C端兑换 | exchange.go | exchange_service.go + wallet_service.go | exchange_order.go + wallet_account.go + wallet_flow.go | rate.go | decimal.go | t_exchange_order + t_wallet_account + t_wallet_flow |
| C端提现下单 | withdraw.go | withdraw_service.go + wallet_service.go | withdraw_order.go + wallet_account.go + wallet_flow.go + freeze_record.go | - | decimal.go | t_withdraw_order + t_wallet_account + t_wallet_flow + t_freeze_record |
| C端保存提现账户 | withdraw.go | withdraw_service.go | withdraw_account.go | - | - | t_withdraw_account |
| C端记录查询 | deposit.go + withdraw.go + exchange.go | record_service.go | deposit_order.go + withdraw_order.go + exchange_order.go | - | - | 三张订单表 |
| RPC入账 | wallet.go | wallet_service.go | wallet_account.go + wallet_flow.go | - | decimal.go | t_wallet_account + t_wallet_flow |
| RPC冻结 | wallet.go | wallet_service.go | wallet_account.go + wallet_flow.go + freeze_record.go | - | decimal.go | t_wallet_account + t_wallet_flow + t_freeze_record |
| RPC解冻 | wallet.go | wallet_service.go | wallet_account.go + wallet_flow.go + freeze_record.go | - | decimal.go | t_wallet_account + t_wallet_flow + t_freeze_record |
| RPC扣款 | wallet.go | wallet_service.go | wallet_account.go + wallet_flow.go + freeze_record.go | - | decimal.go | t_wallet_account + t_wallet_flow + t_freeze_record |
| RPC人工加减款 | wallet.go | wallet_service.go | wallet_account.go + wallet_flow.go | - | decimal.go | t_wallet_account + t_wallet_flow |

### 13.2 一笔「提现」的完整文件穿越

以最复杂的业务场景——用户创建提现订单——为例，追踪请求穿越哪些文件：

```
用户发起提现请求
    │
    ▼
handler.go                          → CreateWithdrawOrder() 一行委托
    │
    ▼
ser/withdraw_service.go             → 校验币种/限额/KYC → 计算手续费
    │
    ├─→ ser/currency_service.go     → GetExchangeRate()（BSB提现需汇率）
    │       └─→ cache/rate.go       → 优先读缓存
    │           └─→ rep/currency.go → 缓存miss查DB
    │
    ├─→ rep/withdraw_order.go       → 创建提现订单（状态=待审核）
    │
    ├─→ ser/wallet_service.go       → WalletFreeze()（冻结对应金额）
    │       │
    │       ├─→ rep/wallet_account.go → CAS更新余额（available↓ frozen↑）
    │       ├─→ rep/wallet_flow.go    → 插入冻结流水（幂等唯一索引）
    │       └─→ rep/freeze_record.go  → 插入冻结记录
    │
    └─→ NATS/Kafka                  → 发消息通知 ser-finance 开始审核
```

涉及文件：`handler.go` → `ser/withdraw_service.go` → `ser/currency_service.go` → `ser/wallet_service.go` → `cache/rate.go` → `rep/currency.go` + `rep/withdraw_order.go` + `rep/wallet_account.go` + `rep/wallet_flow.go` + `rep/freeze_record.go` + `utils/decimal.go`。共 **11个文件协同**。

---

## 十四、文件与实现阶段的映射关系

### 14.1 按Phase标注每个文件何时编写

| Phase | 阶段名称 | 新增/修改的文件 |
|-------|---------|---------------|
| **1** | 工程脚手架 | `main.go`、`handler.go`、`go.mod`、`.gitignore`、`.golangci.yml`、`cfg/mysql.go`、`cfg/redis.go`、`gen/gorm_gen.go`（修正）、`errs/code.go`（框架）、全部 `enum/*.go`（框架）、全部 IDL thrift 文件 |
| **2** | 币种配置 | `ser/currency_service.go`、`rep/currency.go`、`rep/rate_log.go`、`cache/currency.go`、`cache/rate.go`、`utils/decimal.go` |
| **3** | 余额体系 | `ser/wallet_service.go`、`rep/wallet_account.go`、`rep/wallet_flow.go`、`rep/freeze_record.go` |
| **4** | 兑换模块 | `ser/exchange_service.go`、`rep/exchange_order.go` |
| **5** | 充值模块 | `ser/deposit_service.go`、`rep/deposit_order.go` |
| **6** | 提现模块 | `ser/withdraw_service.go`、`rep/withdraw_order.go`、`rep/withdraw_account.go` |
| **7** | 记录+联调 | `ser/record_service.go` |

### 14.2 文件依赖与编写顺序可视化

```
Phase 1 ─── main.go, handler.go, go.mod, cfg/*, errs/*, enum/*, IDL
                │
Phase 2 ─── currency_service.go, currency.go(rep), rate_log.go(rep)
            cache/currency.go, cache/rate.go, utils/decimal.go
                │
Phase 3 ─── wallet_service.go, wallet_account.go(rep)
            wallet_flow.go(rep), freeze_record.go(rep)
                │
Phase 4 ─── exchange_service.go, exchange_order.go(rep)
                │
Phase 5 ─── deposit_service.go, deposit_order.go(rep)
                │
Phase 6 ─── withdraw_service.go, withdraw_order.go(rep)
            withdraw_account.go(rep)
                │
Phase 7 ─── record_service.go
```

**为什么这个顺序**：
- Phase 1 产出骨架，所有后续Phase才有东西可写
- Phase 2 币种配置是地基——汇率、精度被所有金额操作依赖
- Phase 3 余额体系是核心——6种原子操作被充值/提现/兑换依赖
- Phase 4 兑换最简单（内部闭环），用来验证 Phase 2+3 的正确性
- Phase 5→6 复杂度递增（充值不需冻结，提现需要）
- Phase 7 记录是纯查询，等前面的订单数据有了才有意义

---

## 十五、与现有工程的模式对照

### 15.1 ser-wallet vs ser-app 目录对比

```
ser-app/                              ser-wallet/
├── main.go                    ✓      ├── main.go
├── handler.go                 ✓      ├── handler.go
├── go.mod                     ✓      ├── go.mod
├── .gitignore                 ✓      ├── .gitignore
├── .golangci.yml              ✓      ├── .golangci.yml
├── internal/                         ├── internal/
│   ├── cfg/                          │   ├── cfg/
│   │   ├── mysql.go           ✓      │   │   ├── mysql.go
│   │   └── redis.go           ✓      │   │   └── redis.go
│   ├── errs/                         │   ├── errs/
│   │   └── code.go            ✓      │   │   └── code.go
│   ├── enum/                         │   ├── enum/
│   │   ├── banner.go          ≈      │   │   ├── currency.go
│   │   ├── category.go        ≈      │   │   ├── wallet.go
│   │   ├── relation_content.go≈      │   │   ├── deposit.go / withdraw.go / exchange.go
│   │   └── enum_registry.go   ✓      │   │   └── enum_registry.go
│   ├── ser/                          │   ├── ser/
│   │   ├── banner.go          ≈      │   │   ├── currency_service.go
│   │   ├── home_category.go   ≈      │   │   ├── wallet_service.go
│   │   ├── s3_service.go      ≈      │   │   ├── deposit_service.go
│   │   ├── enum_service.go           │   │   ├── withdraw_service.go
│   │   └── content_strategy.go       │   │   ├── exchange_service.go
│   │                                 │   │   └── record_service.go
│   ├── rep/                          │   ├── rep/
│   │   ├── banner.go          ≈      │   │   ├── currency.go ... (9个)
│   │   └── home_category.go          │   │
│   ├── cache/                        │   ├── cache/
│   │   └── banner.go          ≈      │   │   ├── currency.go
│   │                                 │   │   └── rate.go
│   ├── gen/                          │   ├── gen/
│   │   └── gorm_gen.go        ✓      │   │   └── gorm_gen.go
│   ├── gorm_gen/              ✓      │   ├── gorm_gen/ (9表)
│   └── (无utils)              ←      │   └── utils/
│                                     │       └── decimal.go    ← 新增（金额精度需要）
```

✓ = 完全一致的文件   ≈ = 模式相同，内容不同

### 15.2 差异点说明

| 差异 | 原因 | 是否符合规范 |
|------|------|-------------|
| ser-wallet 多了 `utils/decimal.go` | 金额精度是钱包的特有需求，其他服务不涉及 | 符合（ser-ip 也有 `internal/utils/`） |
| ser-wallet enum/ 文件更多（6个 vs 4个） | 业务域更多（6个域 vs 2个域），每域一个enum文件 | 符合（文件数与域数成正比） |
| ser-wallet rep/ 文件更多（9个 vs 2个） | 表更多（9张 vs 2张），每表一个repo文件 | 符合（一表一repo是工程惯例） |
| ser-wallet 没有 `domain/` 目录 | domain/ 只在 ser-item 使用，且ser-item的domain/主要用于validator/mapper分离。wallet的验证逻辑在ser层足够清晰 | 符合（不为了用而用） |

### 15.3 为什么不引入 domain/ 层

ser-item 引入 `domain/` 是因为它需要策略模式的 Validator 接口来处理不同物品类型的校验差异。ser-wallet 的校验逻辑相对直接（金额校验、状态校验、限额校验），不需要策略模式抽象。如果后续业务复杂度增长（如新增子钱包类型有不同规则），可以考虑提取 domain 层，但现阶段不引入 = 不过度设计。
