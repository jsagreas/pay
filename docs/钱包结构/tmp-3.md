# ser-wallet 工程骨架结构设计

> 设计时间：2026-02-27
> 定位：准实施阶段的工程结构全览，展示 ser-wallet 模块所有目录与文件的骨架布局
> 设计依据：8 个已有服务的工程规范 + 需求理解 + IDL 设计 + 架构设计 + 表结构设计 + 实现思路
> 设计原则：严格对齐已有工程规范，不自创目录/命名/分层模式；所有文件均有工程依据
> 范围：ser-wallet 模块本体 + IDL 目录 + 网关文件 + 公共注册项（覆盖"钱包落地需要改/加的所有文件"）

---

## 一、工程结构全景图

以下为 `tree` 命令视角的完整文件清单。分为四个区域展示：**模块本体**、**IDL 契约**、**网关接入**、**公共注册**。

### 1.1 ser-wallet 模块本体

```
ser-wallet/
├── handler.go                          ← Kitex Handler，WalletServiceImpl
├── main.go                             ← 服务启动入口
├── go.mod
├── go.sum
├── .golangci.yml
├── .gitignore
├── sql/
│   └── 1.table.sql                     ← 全部建表 DDL
│
├── kitex_gen/                           ← [自动生成] gen_kitex.sh 产出
│   └── ser_wallet/
│       ├── service.go                   ← WalletService 接口定义
│       ├── wallet.go                    ← wallet.thrift 类型
│       ├── currency.go                  ← currency.thrift 类型
│       ├── deposit.go                   ← deposit.thrift 类型
│       ├── withdraw.go                  ← withdraw.thrift 类型
│       ├── exchange.go                  ← exchange.thrift 类型
│       ├── record.go                    ← record.thrift 类型
│       ├── wallet_rpc.go               ← wallet_rpc.thrift 类型
│       ├── k-ser_wallet.go
│       ├── k-consts.go
│       └── walletservice/
│           ├── client.go                ← RPC Client Stub
│           ├── invoker.go
│           ├── server.go                ← RPC Server Stub
│           └── walletservice.go
│
└── internal/
    ├── cfg/
    │   ├── mysql.go                     ← GORM 数据库连接
    │   └── redis.go                     ← Redis 连接
    │
    ├── enum/
    │   ├── wallet_type.go               ← 钱包类型枚举
    │   ├── currency_type.go             ← 币种类型枚举
    │   ├── flow_type.go                 ← 流水类型枚举
    │   ├── order_status.go              ← 订单状态枚举
    │   └── withdraw_method.go           ← 提现方式枚举
    │
    ├── errs/
    │   └── code.go                      ← 全模块错误码
    │
    ├── ser/
    │   ├── currency_service.go          ← 币种配置业务
    │   ├── wallet_core.go               ← 账户核心层（6个原子操作）
    │   ├── wallet_service.go            ← 钱包查询业务
    │   ├── deposit_service.go           ← 充值业务
    │   ├── withdraw_service.go          ← 提现业务
    │   ├── exchange_service.go          ← 兑换业务
    │   └── record_service.go            ← 记录查询业务
    │
    ├── rep/
    │   ├── currency_rep.go              ← 币种数据访问
    │   ├── wallet_rep.go                ← 钱包账户数据访问
    │   ├── flow_rep.go                  ← 流水数据访问
    │   ├── deposit_rep.go               ← 充值订单数据访问
    │   ├── withdraw_rep.go              ← 提现订单数据访问
    │   ├── withdraw_account_rep.go      ← 提现账户数据访问
    │   └── exchange_rep.go              ← 兑换记录数据访问
    │
    ├── cache/
    │   ├── currency_cache.go            ← 币种配置缓存
    │   └── balance_cache.go             ← 余额缓存
    │
    ├── gen/
    │   └── gorm_gen.go                  ← GORM Gen 代码生成入口
    │
    └── gorm_gen/                         ← [自动生成] gorm_gen.go 产出
        ├── model/
        │   ├── currency_config.gen.go
        │   ├── user_wallet.gen.go
        │   ├── wallet_flow.gen.go
        │   ├── wallet_freeze_record.gen.go
        │   ├── deposit_order.gen.go
        │   ├── withdraw_order.gen.go
        │   ├── withdraw_account.gen.go
        │   ├── exchange_record.gen.go
        │   ├── exchange_rate_log.gen.go
        │   ├── manual_adjustment_order.gen.go
        │   ├── wallet_config.gen.go
        │   ├── deposit_bonus_snapshot.gen.go
        │   ├── wallet_daily_snapshot.gen.go
        │   └── rate_provider_log.gen.go
        ├── query/
        │   ├── gen.go                    ← SetDefault / Use / Q 全局变量
        │   ├── currency_config.gen.go
        │   ├── user_wallet.gen.go
        │   ├── wallet_flow.gen.go
        │   ├── wallet_freeze_record.gen.go
        │   ├── deposit_order.gen.go
        │   ├── withdraw_order.gen.go
        │   ├── withdraw_account.gen.go
        │   ├── exchange_record.gen.go
        │   ├── exchange_rate_log.gen.go
        │   ├── manual_adjustment_order.gen.go
        │   ├── wallet_config.gen.go
        │   ├── deposit_bonus_snapshot.gen.go
        │   ├── wallet_daily_snapshot.gen.go
        │   └── rate_provider_log.gen.go
        └── repo/
            ├── currencyconfigr/
            │   └── currencyconfigr.go
            ├── userwalletr/
            │   └── userwalletr.go
            ├── walletflowr/
            │   └── walletflowr.go
            ├── walletfreezerecordr/
            │   └── walletfreezerecordr.go
            ├── depositorderr/
            │   └── depositorderr.go
            ├── withdraworderr/
            │   └── withdraworderr.go
            ├── withdrawaccountr/
            │   └── withdrawaccountr.go
            ├── exchangerecordr/
            │   └── exchangerecordr.go
            ├── exchangeratelogr/
            │   └── exchangeratelogr.go
            ├── manualadjustmentorderr/
            │   └── manualadjustmentorderr.go
            ├── walletconfigr/
            │   └── walletconfigr.go
            ├── depositbonussnapshotr/
            │   └── depositbonussnapshotr.go
            ├── walletdailysnapshotr/
            │   └── walletdailysnapshotr.go
            └── rateproviderlogr/
                └── rateproviderlogr.go
```

### 1.2 IDL 契约文件

```
common/idl/ser-wallet/
├── service.thrift                       ← 入口：WalletService 全部方法声明
├── wallet.thrift                        ← 钱包核心类型
├── currency.thrift                      ← 币种配置类型
├── deposit.thrift                       ← 充值相关类型
├── withdraw.thrift                      ← 提现相关类型
├── exchange.thrift                      ← 兑换相关类型
├── record.thrift                        ← 记录查询类型
└── wallet_rpc.thrift                    ← RPC 对外类型
```

### 1.3 网关接入文件

```
gate-back/biz/
├── handler/wallet/
│   └── wallet.go                        ← B端 7 个 Handler 函数
└── router/wallet/
    └── wallet_router.go                 ← B端路由注册 RouterWallet()

gate-font/biz/
├── handler/wallet/
│   └── wallet.go                        ← C端 13 个 Handler 函数
└── router/wallet/
    └── wallet_router.go                 ← C端路由注册 RouterWallet()
```

### 1.4 公共注册变更

```
需变更的已有文件（非新建）：

common/pkg/consts/namesp/namesp.go       ← +3 行：EtcdWalletService/Port/Db
common/rpc/rpc_client.go                 ← +1 工厂方法：WalletClient()
go.work                                  ← +1 行：./ser-wallet
gate-back/biz/router/register.go         ← +1 行：wallet.RouterWallet(r)
gate-font/biz/router/register.go         ← +1 行：wallet.RouterWallet(r)
```

---

## 二、文件统计总览

### 2.1 按分类统计

| 分类 | 新建文件数 | 自动生成文件数 | 变更已有文件数 | 合计 |
|------|-----------|--------------|-------------|------|
| ser-wallet 根目录 | 5 | 0 | 0 | 5 |
| internal/cfg | 2 | 0 | 0 | 2 |
| internal/enum | 5 | 0 | 0 | 5 |
| internal/errs | 1 | 0 | 0 | 1 |
| internal/ser | 7 | 0 | 0 | 7 |
| internal/rep | 7 | 0 | 0 | 7 |
| internal/cache | 2 | 0 | 0 | 2 |
| internal/gen | 0 | 0 | 0 | 0* |
| internal/gorm_gen | 0 | 43 | 0 | 43 |
| kitex_gen | 0 | ~12 | 0 | ~12 |
| sql | 1 | 0 | 0 | 1 |
| IDL（common/idl） | 8 | 0 | 0 | 8 |
| 网关 handler | 2 | 0 | 0 | 2 |
| 网关 router | 2 | 0 | 0 | 2 |
| 公共注册 | 0 | 0 | 5 | 5 |
| **合计** | **~42** | **~55** | **5** | **~102** |

> *gen/gorm_gen.go 已存在于脚手架中，无需新建

### 2.2 手写 vs 自动生成比例

```
┌──────────────────────────────────────────────────┐
│             ser-wallet 代码构成                    │
│                                                   │
│  ██████████████████████░░░░░░░░░░░░  手写 ~42 个  │
│  ░░░░░░░░░░░░░░░░░░░░████████████████ 生成 ~55 个 │
│  ▓▓▓▓▓                               变更 5 个   │
│                                                   │
│  手写文件：核心业务逻辑，需要逐个编写               │
│  生成文件：gorm_gen + kitex_gen，工具自动产出       │
│  变更文件：已有公共文件，各加 1-3 行                │
└──────────────────────────────────────────────────┘
```

---

## 三、根目录文件详解

```
ser-wallet/
├── handler.go
├── main.go
├── go.mod
├── go.sum
├── .golangci.yml
├── .gitignore
└── sql/
    └── 1.table.sql
```

| 文件 | 职责 | 内容概要 | 工程依据 |
|------|------|---------|---------|
| **handler.go** | Kitex RPC 入口 | `WalletServiceImpl` 结构体，持有 6 个 Service 实例，每个 IDL 方法对应一个委托函数 | ser-app/ser-item handler.go 模式一致 |
| **main.go** | 服务启动入口 | 初始化链：InitKLog → InitEtcd → flag 解析端口 → 构造 Services → NewServer → Run | ser-item main.go 模式一致 |
| **go.mod** | 模块依赖 | `module ser-wallet`，replace common/pkg，依赖 kitex/gorm/etcd/redis | ser-app go.mod 模式一致 |
| **go.sum** | 依赖锁文件 | 自动生成 | — |
| **.golangci.yml** | Lint 配置 | 已存在于脚手架中 | — |
| **.gitignore** | Git 忽略 | 已存在于脚手架中 | — |
| **sql/1.table.sql** | 建表 DDL | 14 张表的 CREATE TABLE 语句 | ser-item sql/1.table.sql 模式一致 |

### handler.go 结构体与方法映射

handler.go 是所有 RPC 请求的入口。`WalletServiceImpl` 结构体内嵌 6 个 Service，每个 IDL 方法转发到对应的 Service 方法：

| handler 方法 | 委托到 | 对应 Service |
|-------------|--------|-------------|
| GetWalletHome | walletService.GetWalletHome | wallet_service.go |
| GetCurrencyList | walletService.GetCurrencyList | wallet_service.go |
| PageUserWallet | walletService.PageUserWallet | wallet_service.go |
| DetailUserWallet | walletService.DetailUserWallet | wallet_service.go |
| GetDepositMethods | depositService.GetDepositMethods | deposit_service.go |
| CreateDepositOrder | depositService.CreateDepositOrder | deposit_service.go |
| CheckDuplicateDeposit | depositService.CheckDuplicateDeposit | deposit_service.go |
| GetExchangeRate | exchangeService.GetExchangeRate | exchange_service.go |
| DoExchange | exchangeService.DoExchange | exchange_service.go |
| GetWithdrawMethods | withdrawService.GetWithdrawMethods | withdraw_service.go |
| GetWithdrawAccount | withdrawService.GetWithdrawAccount | withdraw_service.go |
| SaveWithdrawAccount | withdrawService.SaveWithdrawAccount | withdraw_service.go |
| CreateWithdrawOrder | withdrawService.CreateWithdrawOrder | withdraw_service.go |
| GetRecordList | recordService.GetRecordList | record_service.go |
| GetRecordDetail | recordService.GetRecordDetail | record_service.go |
| PageCurrency | currencyService.PageCurrency | currency_service.go |
| EditCurrency | currencyService.EditCurrency | currency_service.go |
| SetBaseCurrency | currencyService.SetBaseCurrency | currency_service.go |
| StatusCurrency | currencyService.StatusCurrency | currency_service.go |
| PageExchangeRateLog | currencyService.PageExchangeRateLog | currency_service.go |
| CreditWallet | walletCore.CreditWallet | wallet_core.go |
| DebitWallet | walletCore.DebitWallet | wallet_core.go |
| FreezeBalance | walletCore.FreezeBalance | wallet_core.go |
| UnfreezeBalance | walletCore.UnfreezeBalance | wallet_core.go |
| ConfirmDebit | walletCore.ConfirmDebit | wallet_core.go |
| ManualCredit | walletCore.ManualCredit | wallet_core.go |
| ManualDebit | walletCore.ManualDebit | wallet_core.go |
| SupplementCredit | walletCore.SupplementCredit | wallet_core.go |
| GetBalance | walletCore.GetBalance | wallet_core.go |
| RewardCredit | walletCore.RewardCredit | wallet_core.go |

> 共 30 个方法，handler.go 内每个方法体仅一行 return 委托。对比已有工程：ser-user ~37 方法、ser-live ~32 方法、ser-app ~25 方法，规模处于合理范围。

### main.go 初始化顺序

```
main()
  │
  ├─ 1. lg.InitKLog()                              日志初始化
  ├─ 2. etcd.InitEtcd()                            ETCD 连接
  ├─ 3. flag 解析端口号（默认从 ETCD 读取）
  ├─ 4. 构造 Service 实例
  │     ├─ currencyService  = ser.NewCurrencyService()
  │     ├─ walletCore       = ser.NewWalletCore(currencyService)
  │     ├─ walletService    = ser.NewWalletService(currencyService, walletCore)
  │     ├─ depositService   = ser.NewDepositService(currencyService, walletCore)
  │     ├─ withdrawService  = ser.NewWithdrawService(currencyService, walletCore)
  │     ├─ exchangeService  = ser.NewExchangeService(currencyService, walletCore)
  │     └─ recordService    = ser.NewRecordService()
  ├─ 5. walletservice.NewServer(&WalletServiceImpl{...}, rpc.InitRpcServerParams(...)...)
  └─ 6. srv.Run()
```

> Service 构造顺序体现了依赖链：currencyService（无依赖）→ walletCore（依赖 currency）→ 其余 Service（依赖 currency + core）。这与架构设计中的"币种配置域是最底层"一致。

---

## 四、internal/ 分层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    ser-wallet internal/ 五层架构                  │
│                                                                 │
│  ┌─── cfg/ ───┐     初始化层：数据库/Redis 连接                    │
│  │ mysql.go   │     sync.Once 单例模式，服务启动时调用              │
│  │ redis.go   │                                                  │
│  └────────────┘                                                  │
│        │                                                         │
│        ▼                                                         │
│  ┌─── enum/ + errs/ ───┐   常量层：枚举值 + 错误码                 │
│  │ wallet_type.go      │   所有层共享引用，不依赖任何层              │
│  │ flow_type.go   ...  │                                         │
│  │ errs/code.go        │                                         │
│  └─────────────────────┘                                         │
│        │                                                         │
│        ▼                                                         │
│  ┌─── ser/ ─────────────────────────────────────────────────┐    │
│  │                   业务逻辑层（手写核心代码）                │    │
│  │                                                          │    │
│  │  ┌─ 业务编排 ──────────────────────────────────────────┐ │    │
│  │  │ deposit_service    withdraw_service                 │ │    │
│  │  │ exchange_service   record_service                   │ │    │
│  │  │ wallet_service（查询编排）                            │ │    │
│  │  └──────────────┬─────────────────────────────────────┘ │    │
│  │                 │ 调用                                   │    │
│  │  ┌─ 账户核心 ──▼───────────────────────────────────────┐ │    │
│  │  │ wallet_core.go                                      │ │    │
│  │  │ creditCore / debitCore / freezeCore / unfreezeCore   │ │    │
│  │  │ confirmDebitCore / getBalance                        │ │    │
│  │  │ （幂等 + 事务 + 并发控制 集中在此）                    │ │    │
│  │  └──────────────┬──────────────────────────────────────┘ │    │
│  │  ┌─ 配置核心 ──▼───────────────────────────────────────┐ │    │
│  │  │ currency_service.go                                 │ │    │
│  │  │ 币种 CRUD / 汇率计算 / 基准币种管理                   │ │    │
│  │  └─────────────────────────────────────────────────────┘ │    │
│  └──────────────────────────────────────────────────────────┘    │
│        │                                                         │
│        ▼                                                         │
│  ┌─── rep/ + cache/ ───┐   数据访问层（手写扩展查询 + 缓存）       │
│  │ wallet_rep.go       │   在 gorm_gen 的通用 CRUD 之上            │
│  │ currency_rep.go ... │   补充复杂查询（分页/联合条件/聚合）        │
│  │ currency_cache.go   │   Redis 缓存热点数据                      │
│  │ balance_cache.go    │                                          │
│  └─────────────────────┘                                         │
│        │                                                         │
│        ▼                                                         │
│  ┌─── gorm_gen/ ───┐   生成代码层（禁止手动编辑）                   │
│  │ model/  → 实体   │   gorm_gen.go 连接数据库后自动生成             │
│  │ query/  → 查询   │   每张表 → 1 个 model + 1 个 query + 1 个 repo│
│  │ repo/   → 仓库   │   repo 内含 200+ 通用 CRUD 方法               │
│  └─────────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
```

**调用方向严格单向向下**：ser/ → rep/ → gorm_gen/，禁止反向依赖。cache/ 被 ser/ 和 rep/ 使用。enum/ 和 errs/ 被所有层引用。

---

## 五、internal/cfg/ — 初始化配置层

```
internal/cfg/
├── mysql.go
└── redis.go
```

| 文件 | 职责 | 关键实现模式 | 被谁调用 |
|------|------|------------|---------|
| mysql.go | 初始化 GORM 数据库连接 | sync.Once 单例；从 ETCD 读取 TiDB 地址/账密 + `namesp.EtcdWalletDb` 获取库名；初始化后调用 `query.SetDefault(db)` 注册到 gorm_gen | rep/ 和 gorm_gen 的查询入口 |
| redis.go | 初始化 Redis 连接 | sync.Once 单例；从 ETCD 读取 Redis 地址/密码；返回 `*rds.InitOption` | cache/ 层使用 |

> 工程依据：与 ser-app/internal/cfg/mysql.go、ser-item/internal/cfg/mysql.go 完全一致的初始化模式。不需要改动任何模式，直接复用模板即可。

---

## 六、internal/enum/ — 枚举常量层

```
internal/enum/
├── wallet_type.go
├── currency_type.go
├── flow_type.go
├── order_status.go
└── withdraw_method.go
```

| 文件 | 定义内容 | 预估枚举数量 | 需求来源 |
|------|---------|------------|---------|
| wallet_type.go | 钱包类型：中心钱包、奖励钱包、主播钱包、代理钱包 | 4 个 | 需求4.1"资金钱包包含中心和奖励" |
| currency_type.go | 币种类型：法定货币、加密货币、平台货币 | 3 个 | 需求3.3"币种类型字段" |
| flow_type.go | 流水类型：充值入账、扣款、冻结、解冻、确认扣款、兑换、手动加款、手动减款、奖金入账、补单 | 10+ 个 | 架构设计"流水类型枚举" |
| order_status.go | 充值/提现订单状态：待支付、进行中、成功、失败、已取消 | 5-8 个 | 需求5/6 各状态流转 |
| withdraw_method.go | 提现方式：银行卡、电子钱包（后续可能扩展加密） | 2-3 个 | 需求6.1"提现方式列表" |

> 工程依据：ser-app/internal/enum/ 有 4 个枚举文件（banner.go、category.go、enum_registry.go、relation_content.go），ser-wallet 5 个文件属于同等规模。命名模式一致：每个文件聚焦一个枚举维度。

---

## 七、internal/errs/ — 错误码层

```
internal/errs/
└── code.go
```

### 错误码分段规划

所有错误码集中在 `code.go` 一个文件中，按业务模块分段。格式遵循已有工程 `60{XX}{YYY}` 模式：

```
60{XX} = 服务编号（待分配，如 022）
{YYY}  = 模块内编号

┌──────────────┬──────────────┬──────────────────────────────────┐
│ 段号          │ 范围          │ 覆盖模块                         │
├──────────────┼──────────────┼──────────────────────────────────┤
│ 60XX 0xx     │ 000-099      │ 币种配置（CurrencyNotFound 等）   │
│ 60XX 1xx     │ 100-199      │ 钱包核心（InsufficientBalance 等）│
│ 60XX 2xx     │ 200-299      │ 充值（DuplicateBlocked 等）       │
│ 60XX 3xx     │ 300-399      │ 提现（KycRequired 等）            │
│ 60XX 4xx     │ 400-499      │ 兑换（RateExpired 等）            │
│ 60XX 5xx     │ 500-599      │ 通用/预留                        │
└──────────────┴──────────────┴──────────────────────────────────┘
```

> 工程依据：ser-app 使用 6011xxx（第 011 号服务），ser-item 使用 6010xxx（第 010 号服务），每个服务独占一个段号。ser-wallet 的段号需与团队确认避免冲突。

---

## 八、internal/ser/ — 业务逻辑层（核心）

```
internal/ser/
├── currency_service.go        ← 币种配置域
├── wallet_core.go             ← 账户核心层
├── wallet_service.go          ← 钱包查询域
├── deposit_service.go         ← 充值业务域
├── withdraw_service.go        ← 提现业务域
├── exchange_service.go        ← 兑换业务域
└── record_service.go          ← 记录查询域
```

这是整个模块**最核心的目录**——所有业务逻辑都在这 7 个文件中。

### 8.1 文件职责详解

| 文件 | 对应域 | 职责描述 | IDL 方法数 | 依赖的 rep | 依赖的其他 service |
|------|--------|---------|-----------|------------|------------------|
| **currency_service.go** | 币种配置 | B端币种 CRUD、基准币种设置、启禁用、汇率日志分页、汇率计算辅助方法 | 5 | currency_rep | 无（最底层） |
| **wallet_core.go** | 账户核心 | 6 个原子余额操作 + RPC 对外接口（CreditWallet/DebitWallet/Freeze/Unfreeze/ConfirmDebit/GetBalance 等） | 10 | wallet_rep, flow_rep | currency_service |
| **wallet_service.go** | 钱包查询 | C端首页、币种列表（含余额）、B端用户钱包分页/详情 | 4 | wallet_rep | currency_service, wallet_core |
| **deposit_service.go** | 充值业务 | 充值方式列表、创建充值订单、重复转账检测、充值回调处理 | 3 | deposit_rep | currency_service, wallet_core, 外部 RPC |
| **withdraw_service.go** | 提现业务 | 提现方式列表、账户管理、创建提现订单、提现回调处理 | 4 | withdraw_rep, withdraw_account_rep | currency_service, wallet_core, 外部 RPC |
| **exchange_service.go** | 兑换业务 | 实时汇率查询、执行兑换（法币→BSB） | 2 | exchange_rep | currency_service, wallet_core |
| **record_service.go** | 记录查询 | 交易记录分页、记录详情 | 2 | flow_rep, deposit_rep, withdraw_rep, exchange_rep | 无（纯读） |

### 8.2 Service 依赖关系图

```
                      ┌─────────────────┐
                      │ currency_service │ ← 最底层，被所有 service 依赖
                      └────────┬────────┘
                               │
                      ┌────────▼────────┐
                      │  wallet_core    │ ← 核心层，封装所有余额变更原子操作
                      └────────┬────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                     │
  ┌───────▼───────┐  ┌────────▼────────┐  ┌────────▼────────┐
  │deposit_service│  │withdraw_service │  │exchange_service │
  └───────────────┘  └─────────────────┘  └─────────────────┘
          │                    │
          ▼                    ▼
    外部 RPC 调用         外部 RPC 调用
    (财务/活动)           (财务/KYC)

  ┌───────────────┐  ┌─────────────────┐
  │wallet_service │  │ record_service  │ ← 纯查询，不调用 wallet_core 写操作
  └───────────────┘  └─────────────────┘
```

### 8.3 wallet_core.go 特别说明

wallet_core.go 是 ser-wallet 区别于其他服务的关键文件。它不是简单的 Service，而是**账务引擎**：

```
wallet_core.go 内部方法：

┌────────────────────┬──────────────────────────────────────────────┐
│ 方法                │ 职责                                         │
├────────────────────┼──────────────────────────────────────────────┤
│ creditCore()       │ 加款：可用余额 += amount，写入流水              │
│ debitCore()        │ 扣款：可用余额 -= amount（余额不足报错），写入流水│
│ freezeCore()       │ 冻结：可用余额 -= amount，冻结余额 += amount    │
│ unfreezeCore()     │ 解冻：冻结余额 -= amount，可用余额 += amount    │
│ confirmDebitCore() │ 确认扣款：冻结余额 -= amount（不还可用余额）     │
│ getBalance()       │ 查询：返回用户指定币种/钱包类型的余额信息         │
└────────────────────┴──────────────────────────────────────────────┘

每个 core 方法内部统一处理：
  1. 幂等校验（order_no 唯一索引）
  2. 事务管理（GORM Transaction）
  3. 并发控制（SELECT ... FOR UPDATE 或乐观锁 version）
  4. 流水写入（wallet_flow 表）
```

> 为什么单独拎出来而不是散在各 service 中：充值调 creditCore、RPC 财务回调也调 creditCore、手动加款也调 creditCore——同一段逻辑被三个入口复用。如果分散在三个 service 中，幂等和并发控制逻辑必然重复或遗漏。集中管理是资金系统的标准做法。

---

## 九、internal/rep/ — 数据访问层

```
internal/rep/
├── currency_rep.go
├── wallet_rep.go
├── flow_rep.go
├── deposit_rep.go
├── withdraw_rep.go
├── withdraw_account_rep.go
└── exchange_rep.go
```

### 9.1 rep 层定位

rep 层是 **gorm_gen 自动生成的通用 CRUD（200+ 方法）** 之上的**自定义查询扩展层**。简单的单表增删改查直接用 `query.Q.Xxx` 即可，只有涉及以下场景时才在 rep 中编写：

- 多条件动态拼接（如 BannerFilter 模式）
- 分页查询（Page + Count 组合）
- 聚合查询（SUM、COUNT 统计）
- 多表关联查询（JOIN）
- 批量操作（BatchCreate、BatchUpdate）

### 9.2 文件与数据表映射

| rep 文件 | 操作的表 | 典型自定义查询 |
|----------|---------|--------------|
| currency_rep.go | currency_config, exchange_rate_log | 分页筛选（状态/类型/名称）、启用币种列表、汇率日志分页 |
| wallet_rep.go | user_wallet | 按用户+币种+类型精确查询、批量查余额、分页查用户钱包、余额更新（FOR UPDATE） |
| flow_rep.go | wallet_flow, wallet_freeze_record | 按订单号查流水（幂等）、按用户+币种+类型分页查流水、冻结记录状态流转 |
| deposit_rep.go | deposit_order | 重复转账检测（同用户同币种同金额待支付订单计数）、订单状态更新、分页查询 |
| withdraw_rep.go | withdraw_order | 订单状态更新、审核状态流转、分页查询 |
| withdraw_account_rep.go | withdraw_account | 按用户+方式查账户、保存/更新账户信息 |
| exchange_rep.go | exchange_record | 创建兑换记录、分页查询 |

> 工程依据：ser-app/internal/rep/ 有 2 个文件（banner.go、home_category.go），ser-item/internal/rep/ 有 2 个文件（item.go、item_publish.go）。ser-wallet 7 个文件对应 14 张表（有合并），符合"一个 rep 文件管理 1-2 张相关表"的已有规范。

---

## 十、internal/cache/ — 缓存层

```
internal/cache/
├── currency_cache.go
└── balance_cache.go
```

| 文件 | 缓存内容 | 缓存策略 | 使用场景 |
|------|---------|---------|---------|
| currency_cache.go | 启用币种列表、币种配置详情、汇率数据 | 读多写少，修改时主动失效 | C端首页、所有需要币种/汇率数据的接口 |
| balance_cache.go | 用户余额快照 | 写穿透（余额变更时同步更新缓存）或延迟失效 | C端首页余额展示（降低 DB 读压力） |

> 工程依据：ser-app/internal/cache/ 有 1 个文件（banner.go），ser-wallet 2 个文件属于同等规模。缓存层为可选优化，核心逻辑不应依赖缓存正确性（缓存失效时回源 DB）。

---

## 十一、internal/gen/ 与 internal/gorm_gen/ — 代码生成层

### 11.1 gen/gorm_gen.go

```
internal/gen/
└── gorm_gen.go                ← 已存在于脚手架中
```

此文件是 GORM Gen 的代码生成入口脚本。执行 `go run internal/gen/gorm_gen.go` 后，根据 TiDB 中的表结构自动生成 internal/gorm_gen/ 下的所有文件。

### 11.2 gorm_gen/ 自动生成结构

```
internal/gorm_gen/
├── model/          ← 14 个 .gen.go，每张表一个实体 struct
├── query/          ← 14 个 .gen.go + 1 个 gen.go，类型安全的查询构建器
└── repo/           ← 14 个子目录，每个含 1 个 .go 文件，200+ 通用 CRUD 方法
```

**14 张表 → 14 组生成文件的对应关系：**

| 序号 | 数据库表名 | model 文件 | repo 目录 | 优先级 |
|------|-----------|-----------|----------|--------|
| 1 | currency_config | currency_config.gen.go | currencyconfigr/ | 必须 |
| 2 | user_wallet | user_wallet.gen.go | userwalletr/ | 必须 |
| 3 | wallet_flow | wallet_flow.gen.go | walletflowr/ | 必须 |
| 4 | wallet_freeze_record | wallet_freeze_record.gen.go | walletfreezerecordr/ | 必须 |
| 5 | deposit_order | deposit_order.gen.go | depositorderr/ | 必须 |
| 6 | withdraw_order | withdraw_order.gen.go | withdraworderr/ | 必须 |
| 7 | withdraw_account | withdraw_account.gen.go | withdrawaccountr/ | 必须 |
| 8 | exchange_record | exchange_record.gen.go | exchangerecordr/ | 必须 |
| 9 | exchange_rate_log | exchange_rate_log.gen.go | exchangeratelogr/ | 应该 |
| 10 | manual_adjustment_order | manual_adjustment_order.gen.go | manualadjustmentorderr/ | 应该 |
| 11 | wallet_config | wallet_config.gen.go | walletconfigr/ | 应该 |
| 12 | deposit_bonus_snapshot | deposit_bonus_snapshot.gen.go | depositbonussnapshotr/ | 可选 |
| 13 | wallet_daily_snapshot | wallet_daily_snapshot.gen.go | walletdailysnapshotr/ | 可选 |
| 14 | rate_provider_log | rate_provider_log.gen.go | rateproviderlogr/ | 可选 |

> gorm_gen 的 repo 目录命名规则：表名去下划线 + 全小写 + 后缀 `r`。如 `currency_config` → `currencyconfigr/`。这是 GORM Gen 自动生成的，不可自定义。

---

## 十二、kitex_gen/ — Kitex 生成代码

```
kitex_gen/                         ← gen_kitex.sh 从 IDL 自动生成
└── ser_wallet/
    ├── service.go                 ← WalletService 接口定义（所有方法签名）
    ├── wallet.go                  ← wallet.thrift 中的 struct 定义
    ├── currency.go
    ├── deposit.go
    ├── withdraw.go
    ├── exchange.go
    ├── record.go
    ├── wallet_rpc.go
    ├── k-ser_wallet.go            ← Kitex 内部辅助文件
    ├── k-consts.go
    └── walletservice/
        ├── client.go              ← RPC Client（其他服务调用钱包用）
        ├── server.go              ← RPC Server（main.go 中 NewServer 用）
        ├── invoker.go
        └── walletservice.go       ← 服务注册辅助
```

**kitex_gen 与 IDL 的对应关系：**

```
common/idl/ser-wallet/service.thrift
         ↓ gen_kitex.sh
kitex_gen/ser_wallet/service.go        → WalletService interface { ... }
kitex_gen/ser_wallet/walletservice/    → Client / Server 实现

common/idl/ser-wallet/wallet.thrift
         ↓ gen_kitex.sh
kitex_gen/ser_wallet/wallet.go         → GetWalletHomeReq / Resp / WalletInfo 等 struct

（其余 6 个 .thrift 文件同理）
```

> kitex_gen 是纯生成代码，**禁止手动编辑**。IDL 修改后重新执行 gen_kitex.sh 即可更新。handler.go 和其他服务的 rpc_client.go 都 import 此目录下的类型。

---

## 十三、IDL 契约文件详解

```
common/idl/ser-wallet/
├── service.thrift           ← 入口（只声明方法，不定义 struct）
├── wallet.thrift            ← 钱包核心类型
├── currency.thrift          ← 币种配置类型
├── deposit.thrift           ← 充值相关类型
├── withdraw.thrift          ← 提现相关类型
├── exchange.thrift          ← 兑换相关类型
├── record.thrift            ← 记录查询类型
└── wallet_rpc.thrift        ← RPC 对外类型
```

### 13.1 文件职责与方法分配

| IDL 文件 | namespace | 定义内容 | 对应的 IDL 方法 | 预估 struct 数 |
|----------|-----------|---------|----------------|---------------|
| service.thrift | ser_wallet | `service WalletService {}` + include 所有域文件 | ~30 个方法声明 | 0 |
| wallet.thrift | ser_wallet | 钱包余额类型、流水类型、B端用户钱包查询类型 | Home(2) + B端(2) | 10-15 |
| currency.thrift | ser_wallet | 币种配置实体、B端 CRUD 的 Req/Resp | B端(5) | 10-15 |
| deposit.thrift | ser_wallet | 充值方式/订单/重复检测的 Req/Resp | C端(3) | 8-12 |
| withdraw.thrift | ser_wallet | 提现方式/账户/订单的 Req/Resp | C端(4) | 10-14 |
| exchange.thrift | ser_wallet | 汇率/兑换的 Req/Resp | C端(2) | 4-6 |
| record.thrift | ser_wallet | 记录列表/详情的 Req/Resp | C端(2) | 4-6 |
| wallet_rpc.thrift | ser_wallet | RPC 对外接口的 Req/Resp（带 api.body 注解） | RPC(10) | 15-20 |

> 合计：8 个 IDL 文件、~30 个方法、约 60-90 个 struct。对比 ser-user（13 文件 ~37 方法）和 ser-live（9 文件 ~32 方法），规模处于合理范围。

### 13.2 service.thrift 方法分组

```
service WalletService {
    // ── C端接口（gate-font 转发，共 13 个）──────────────
    // 首页 & 币种（2个）
    // 充值（3个）
    // 兑换（2个）
    // 提现（4个）
    // 记录（2个）

    // ── B端接口（gate-back 转发，共 7 个）──────────────
    // 币种配置（5个）
    // 用户钱包（2个）

    // ── RPC 对外接口（内部服务直接调用，共 10 个）────────
    // 余额操作（5个）：Credit / Debit / Freeze / Unfreeze / ConfirmDebit
    // 管理操作（4个）：ManualCredit / ManualDebit / SupplementCredit / RewardCredit
    // 查询操作（1个）：GetBalance
}
```

---

## 十四、网关接入文件详解

### 14.1 gate-back（B端管理后台）

```
gate-back/biz/
├── handler/wallet/
│   └── wallet.go              ← 7 个函数
└── router/wallet/
    └── wallet_router.go       ← RouterWallet() 注册 7 个路由
```

**handler/wallet/wallet.go — 7 个 Handler 函数：**

| 函数名 | 转发到 | 路由路径 |
|--------|--------|---------|
| PageCurrency | rpc.WalletClient().PageCurrency | /admin/api/wallet/currency/page |
| EditCurrency | rpc.WalletClient().EditCurrency | /admin/api/wallet/currency/edit |
| SetBaseCurrency | rpc.WalletClient().SetBaseCurrency | /admin/api/wallet/currency/base |
| StatusCurrency | rpc.WalletClient().StatusCurrency | /admin/api/wallet/currency/status |
| PageExchangeRateLog | rpc.WalletClient().PageExchangeRateLog | /admin/api/wallet/rate/log/page |
| PageUserWallet | rpc.WalletClient().PageUserWallet | /admin/api/wallet/user/page |
| DetailUserWallet | rpc.WalletClient().DetailUserWallet | /admin/api/wallet/user/detail |

> 每个函数体为一行：`rpc.Handler(ctx, c, rpc.WalletClient().Method)`

**router/wallet/wallet_router.go — RouterWallet() 函数：**

7 条路由全部挂载到 `my_router.Api`（需 Token + URL 权限验证）。路径遵循 `/admin/api/{模块}/{资源}/{操作}` 规范。

### 14.2 gate-font（C端用户端）

```
gate-font/biz/
├── handler/wallet/
│   └── wallet.go              ← 13 个函数
└── router/wallet/
    └── wallet_router.go       ← RouterWallet() 注册 13 个路由
```

**handler/wallet/wallet.go — 13 个 Handler 函数：**

| 函数名 | 路由路径 | 路由组 |
|--------|---------|--------|
| GetWalletHome | /api/wallet/home | Api（需登录） |
| GetCurrencyList | /api/wallet/currency/list | Api |
| GetDepositMethods | /api/wallet/deposit/methods | Api |
| CreateDepositOrder | /api/wallet/deposit/create | Api |
| CheckDuplicateDeposit | /api/wallet/deposit/check | Api |
| GetExchangeRate | /api/wallet/exchange/rate | Api |
| DoExchange | /api/wallet/exchange/do | Api |
| GetWithdrawMethods | /api/wallet/withdraw/methods | Api |
| GetWithdrawAccount | /api/wallet/withdraw/account | Api |
| SaveWithdrawAccount | /api/wallet/withdraw/account/save | Api |
| CreateWithdrawOrder | /api/wallet/withdraw/create | Api |
| GetRecordList | /api/wallet/record/list | Api |
| GetRecordDetail | /api/wallet/record/detail | Api |

> 所有 C 端钱包接口均需登录（挂 `my_router.Api`），无公开接口（my_router.Open）。资金操作必须鉴权。

### 14.3 公共注册变更

除了网关 handler/router 的新增文件外，还需要修改以下已有文件：

| 文件 | 变更内容 | 变更量 |
|------|---------|--------|
| gate-back/biz/router/register.go | 在 `customizedRegister()` 中添加 `wallet.RouterWallet(r)` | +1 行 |
| gate-font/biz/router/register.go | 在 `customizedRegister()` 中添加 `wallet.RouterWallet(r)` | +1 行 |

---

## 十五、公共基础设施注册变更

ser-wallet 作为新服务，必须注册到公共基础设施后才能被发现和调用。

```
需变更的公共文件（共 3 处）：

common/pkg/consts/namesp/namesp.go     ← 服务命名常量
common/rpc/rpc_client.go               ← RPC 客户端工厂
go.work                                ← Go 工作区声明
```

### 15.1 namesp.go — 新增 3 个常量

| 常量名 | 值 | 用途 |
|--------|----|----|
| EtcdWalletService | "wallet_service" | Kitex 服务注册/发现名称 |
| EtcdWalletPort | "/slg/serv/wallet/port" | ETCD 中存储端口号的 key |
| EtcdWalletDb | "/slg/serv/wallet/db" | ETCD 中存储数据库名的 key |

### 15.2 rpc_client.go — 新增 WalletClient() 工厂方法

与已有的 `AppClient()`、`UserClient()` 等完全一致的模式：

```
sync.Once + walletservice.NewClient(namesp.EtcdWalletService, InitRpcClientParams()...)
```

注册后，任何服务都可通过 `rpc.WalletClient().Method()` 调用钱包服务的 RPC 接口。

### 15.3 go.work — 新增 1 行

```
use (
    ...
    ./ser-wallet        ← 新增
    ...
)
```

---

## 十六、SQL 文件

```
ser-wallet/sql/
└── 1.table.sql
```

包含 14 张表的 CREATE TABLE 语句，按优先级排列：

| 批次 | 表名 | 优先级 | 说明 |
|------|------|--------|------|
| 第一批（必须） | currency_config | 必须 | 币种配置，所有模块的基础数据源 |
| | user_wallet | 必须 | 用户钱包账户，余额管理核心 |
| | wallet_flow | 必须 | 资金流水，对账审计唯一凭证 |
| | wallet_freeze_record | 必须 | 冻结记录，状态机载体 |
| | deposit_order | 必须 | 充值订单 |
| | withdraw_order | 必须 | 提现订单 |
| | withdraw_account | 必须 | 用户绑定的提现账户 |
| | exchange_record | 必须 | 兑换记录 |
| 第二批（应该） | exchange_rate_log | 应该 | 汇率变更日志 |
| | manual_adjustment_order | 应该 | 人工修正订单 |
| | wallet_config | 应该 | 运行时业务配置 |
| 第三批（可选） | deposit_bonus_snapshot | 可选 | 充值奖金快照 |
| | wallet_daily_snapshot | 可选 | 每日快照统计 |
| | rate_provider_log | 可选 | 三方汇率源日志 |

> 完整 DDL 设计详见：表结构设计/tmp-3.md

---

## 十七、文件 → 需求功能映射表

将需求功能映射到具体文件，建立"需求在哪个文件中实现"的全局索引。

### 17.1 C端功能映射

| 需求功能 | 产品原型来源 | gate-font handler | ser/ service | rep/ | 数据表 |
|----------|------------|-------------------|-------------|------|--------|
| 钱包首页 | 原型2-1 | GetWalletHome | wallet_service | wallet_rep, currency_rep | user_wallet, currency_config |
| 币种切换 | 原型2-2 | GetCurrencyList | wallet_service | wallet_rep, currency_rep | user_wallet, currency_config |
| 充值方式 | 原型3-1~3-4 | GetDepositMethods | deposit_service | currency_rep | currency_config |
| 创建充值 | 原型3-1~3-4 | CreateDepositOrder | deposit_service | deposit_rep, flow_rep | deposit_order, wallet_flow |
| 重复检测 | 原型7-3~7-4 | CheckDuplicateDeposit | deposit_service | deposit_rep | deposit_order |
| 兑换汇率 | 原型4-1~4-3 | GetExchangeRate | exchange_service | currency_rep | currency_config |
| 执行兑换 | 原型4-1~4-3 | DoExchange | exchange_service | exchange_rep, flow_rep | exchange_record, wallet_flow, user_wallet |
| 提现方式 | 原型5-1~5-4 | GetWithdrawMethods | withdraw_service | currency_rep | currency_config |
| 提现账户 | 原型5-1~5-4 | Get/SaveWithdrawAccount | withdraw_service | withdraw_account_rep | withdraw_account |
| 创建提现 | 原型5-1~5-4 | CreateWithdrawOrder | withdraw_service | withdraw_rep, flow_rep | withdraw_order, wallet_flow, wallet_freeze_record |
| 交易记录 | 原型6-1~6-3 | GetRecordList/Detail | record_service | flow_rep, deposit_rep, withdraw_rep, exchange_rep | 多表联查 |

### 17.2 B端功能映射

| 需求功能 | 产品原型来源 | gate-back handler | ser/ service | rep/ | 数据表 |
|----------|------------|-------------------|-------------|------|--------|
| 币种分页 | 需求3.3 | PageCurrency | currency_service | currency_rep | currency_config |
| 编辑币种 | 需求3.3 | EditCurrency | currency_service | currency_rep | currency_config |
| 基准设置 | 需求3.3 | SetBaseCurrency | currency_service | currency_rep | currency_config |
| 启禁用 | 需求3.3 | StatusCurrency | currency_service | currency_rep | currency_config |
| 汇率日志 | 需求3.3 | PageExchangeRateLog | currency_service | currency_rep | exchange_rate_log |
| 用户钱包 | 需求4.1 | PageUserWallet | wallet_service | wallet_rep | user_wallet |
| 钱包详情 | 需求4.1 | DetailUserWallet | wallet_service | wallet_rep, flow_rep | user_wallet, wallet_flow |

### 17.3 RPC 对外接口映射

| RPC 方法 | 调用方 | ser/ 处理 | 核心表 |
|----------|--------|----------|--------|
| CreditWallet | 财务模块（充值回调）| wallet_core.creditCore | user_wallet, wallet_flow |
| DebitWallet | 投注模块（下注扣款）| wallet_core.debitCore | user_wallet, wallet_flow |
| FreezeBalance | 提现模块（提现冻结）| wallet_core.freezeCore | user_wallet, wallet_flow, wallet_freeze_record |
| UnfreezeBalance | 财务模块（提现驳回）| wallet_core.unfreezeCore | user_wallet, wallet_flow, wallet_freeze_record |
| ConfirmDebit | 财务模块（提现确认）| wallet_core.confirmDebitCore | user_wallet, wallet_flow, wallet_freeze_record |
| ManualCredit | B端管理（手动加款）| wallet_core.creditCore | user_wallet, wallet_flow, manual_adjustment_order |
| ManualDebit | B端管理（手动减款）| wallet_core.debitCore | user_wallet, wallet_flow, manual_adjustment_order |
| SupplementCredit | 财务模块（补单）| wallet_core.creditCore | user_wallet, wallet_flow |
| GetBalance | 投注/直播/活动 | wallet_core.getBalance | user_wallet |
| RewardCredit | 活动模块（奖金入账）| wallet_core.creditCore | user_wallet, wallet_flow |

---

## 十八、与已有服务的结构对比

### 18.1 目录结构对比表

以 ser-item（有 domain 层）和 ser-app（无 domain 层）为参照：

| 目录 | ser-app | ser-item | ser-wallet | 说明 |
|------|---------|----------|-----------|------|
| handler.go | 有 | 有 | 有 | 标准 |
| main.go | 有 | 有 | 有 | 标准 |
| go.mod | 有 | 有 | 有 | 标准 |
| sql/ | 无 | 有 | 有 | ser-item 有，ser-wallet 需要 |
| internal/cfg/ | 有(2) | 有(2) | 有(2) | 标准 |
| internal/enum/ | 有(4) | 有(1) | 有(5) | 按枚举维度拆分 |
| internal/errs/ | 有(1) | 有(1) | 有(1) | 标准 |
| internal/ser/ | 有(5) | 有(3) | 有(7) | 按业务域拆分 |
| internal/rep/ | 有(2) | 有(2) | 有(7) | 对应表数量更多 |
| internal/cache/ | 有(1) | 无 | 有(2) | 钱包有热点缓存需求 |
| internal/domain/ | 无 | 有(9) | 无* | 见下方说明 |
| internal/gen/ | 有(1) | 有(1) | 有(1) | 已存在 |
| internal/gorm_gen/ | 有 | 有 | 有 | 自动生成 |
| internal/test/ | 无 | 有(4) | 无** | 见下方说明 |
| internal/utils/ | 无 | 有 | 无 | 无自定义工具需求 |
| kitex_gen/ | 有 | 有 | 有 | 自动生成 |

**关于 domain/ 层**：ser-item 的 domain 层做验证器、映射器、缓存管理器等横切关注点的封装。ser-wallet 的对应能力通过 `wallet_core.go`（集中在 ser/ 层内）实现，不单独建 domain/ 目录。原因是：wallet_core 的职责更聚焦（6 个原子操作），不像 ser-item 的 domain 那样需要拆分 9 个文件。如果后续膨胀可再拆出。

**关于 test/ 层**：初始骨架不包含 test/，但建议在编码阶段为 wallet_core.go 的 6 个原子操作编写单元测试（资金安全是第一优先级）。

### 18.2 规模对比

```
┌──────────────┬──────────┬──────────┬──────────┬──────────────┐
│ 指标          │ ser-app  │ ser-item │ ser-user │ ser-wallet   │
├──────────────┼──────────┼──────────┼──────────┼──────────────┤
│ IDL 文件数    │ 5        │ 4        │ 13       │ 8            │
│ IDL 方法数    │ ~25      │ ~15      │ ~37      │ ~30          │
│ 数据表数      │ 2        │ 2        │ ~8       │ 14           │
│ ser/ 文件数   │ 5        │ 3        │ ~6       │ 7            │
│ rep/ 文件数   │ 2        │ 2        │ ~4       │ 7            │
│ enum/ 文件数  │ 4        │ 1        │ ~3       │ 5            │
│ B端路由       │ ~12      │ ~8       │ ~20      │ 7            │
│ C端路由       │ ~5       │ 0        │ ~10      │ 13           │
│ 手写文件总数   │ ~18      │ ~22      │ ~30      │ ~42          │
└──────────────┴──────────┴──────────┴──────────┴──────────────┘
```

> ser-wallet 的手写文件数（~42）是已有服务中最多的，但属于合理范围——它同时覆盖 C/B/RPC 三端、5 个业务域、14 张表，复杂度本身就高于只服务 B 端的 ser-app 或只有 2 张表的 ser-item。

---

## 十九、开发落地建议顺序

骨架结构中的文件，建议按以下顺序逐步建立：

```
阶段 1：基础设施（无业务逻辑，跑通框架）
  ├─ ① common/idl/ser-wallet/*.thrift          IDL 契约定义
  ├─ ② gen_kitex.sh → kitex_gen/               生成 RPC 代码
  ├─ ③ common/namesp + rpc_client + go.work    公共注册
  ├─ ④ sql/1.table.sql → TiDB 建表（必须表）
  ├─ ⑤ internal/gen/gorm_gen.go → gorm_gen/    生成 ORM 代码
  ├─ ⑥ internal/cfg/mysql.go + redis.go        初始化连接
  ├─ ⑦ handler.go + main.go + go.mod           服务骨架
  └─ ⑧ 启动验证：服务可启动、可注册到 ETCD

阶段 2：核心业务（先打通钱包核心 + 币种配置）
  ├─ ⑨ internal/enum/*.go + errs/code.go       枚举与错误码
  ├─ ⑩ internal/ser/currency_service.go         币种配置 CRUD
  ├─ ⑪ internal/ser/wallet_core.go              6 个原子操作
  ├─ ⑫ internal/rep/currency_rep.go + wallet_rep.go + flow_rep.go
  ├─ ⑬ gate-back handler/router（币种配置 B 端）
  └─ ⑭ RPC 接口联调：CreditWallet / GetBalance 等

阶段 3：交易业务（充值、提现、兑换）
  ├─ ⑮ internal/ser/deposit_service.go + withdraw_service.go + exchange_service.go
  ├─ ⑯ internal/rep/deposit_rep.go + withdraw_rep.go + exchange_rep.go + withdraw_account_rep.go
  ├─ ⑰ gate-font handler/router（C端全部路由）
  └─ ⑱ 与财务模块联调（CreatePayOrder / 回调流程）

阶段 4：查询与优化
  ├─ ⑲ internal/ser/record_service.go           记录查询
  ├─ ⑳ internal/cache/currency_cache.go + balance_cache.go
  └─ ㉑ 补充应该/可选级别的表和对应 gorm_gen
```

---

## 二十、总结

ser-wallet 工程骨架遵循已有工程的所有规范，**没有引入任何新模式**：

- **目录结构**：handler.go + main.go + internal/{cfg,enum,errs,ser,rep,cache,gen,gorm_gen} — 与 ser-app/ser-item 一致
- **IDL 结构**：service.thrift + 域文件 + _rpc.thrift — 与 ser-user/ser-live 一致
- **网关模式**：handler 薄封装 + router 路由注册 — 与所有已有服务一致
- **代码生成**：kitex_gen + gorm_gen — 与所有已有服务一致
- **唯一的特殊设计**：wallet_core.go 在 ser/ 层内部引入"账户核心层"，将 6 个原子余额操作集中管理。这不是新的目录层级，而是 ser/ 内部的文件级拆分，类似 ser-item 的 domain/ 层思路但更轻量

**整体规模**：~42 个手写文件 + ~55 个自动生成文件 + 5 处已有文件变更 ≈ 102 个文件，覆盖 C/B/RPC 三端、5 个业务域、14 张数据表、~30 个 IDL 方法。
