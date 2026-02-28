# ser-finance 工程骨架结构设计（同事模块·协作参考）

> 设计时间：2026-02-27
> 定位：站在钱包模块开发者视角，推演 ser-finance 的工程结构全貌，做到"了然于胸"
> 设计依据：财务管理需求文档（8个子模块）+ 产品原型 + 依赖关系分析 + 链路关系分析 + 已有工程规范
> 设计原则：严格对齐已有工程规范推演，不是"我想让它长什么样"，而是"按工程规范它必然长什么样"
> 视角说明：**我不负责这个模块**，但钱包和财务有双向 RPC 依赖，了解对方结构是顺畅联调的前提
> 与 ser-wallet 的关系：资金体系的"通道层"，钱包管"账"，财务管"通道"，二者共同构成资金进出闭环

---

## 零、为什么需要这份文档

```
资金体系三层架构：

  ┌─────────────────────────────────────────────────────────────┐
  │  【账户层】ser-wallet（我负责）                               │
  │   管理用户账户余额、记录每一笔资金变动                         │
  ├─────────────────────────────────────────────────────────────┤
  │  【配置层】ser-wallet 币种配置子模块（我负责）                  │
  │   管理币种基础数据、汇率体系                                  │
  ├─────────────────────────────────────────────────────────────┤
  │  【通道层】ser-finance（同事负责）← 本文档推演对象              │
  │   管理支付通道、充提订单审核、资金进出的"管道"                  │
  └─────────────────────────────────────────────────────────────┘

钱包和财务的双向 RPC 调用：
  钱包 → 财务：2-3 个接口（获取支付配置、创建通道订单）
  财务 → 钱包：6-7 个接口（充值上账、确认扣除、解冻退回、人工修正）

不清楚财务模块的结构 = 不清楚这些 RPC 调用在对方的哪个 Service 处理、哪个表存储、什么时候回调。
```

---

## 一、工程结构全景图

### 1.1 ser-finance 模块本体

```
ser-finance/
├── handler.go                          ← Kitex Handler，FinanceServiceImpl
├── main.go                             ← 服务启动入口
├── go.mod
├── go.sum
├── .golangci.yml
├── .gitignore
├── sql/
│   └── 1.table.sql                     ← 全部建表 DDL
│
├── kitex_gen/                           ← [自动生成] gen_kitex.sh 产出
│   └── ser_finance/
│       ├── service.go                   ← FinanceService 接口定义
│       ├── channel.go                   ← channel.thrift 类型
│       ├── payment.go                   ← payment.thrift 类型
│       ├── deposit.go                   ← deposit.thrift 类型
│       ├── withdraw.go                  ← withdraw.thrift 类型
│       ├── adjustment.go               ← adjustment.thrift 类型
│       ├── statistics.go               ← statistics.thrift 类型
│       ├── finance_rpc.go              ← finance_rpc.thrift 类型
│       ├── callback.go                 ← callback.thrift 类型
│       ├── k-ser_finance.go
│       ├── k-consts.go
│       └── financeservice/
│           ├── client.go                ← RPC Client Stub
│           ├── invoker.go
│           ├── server.go                ← RPC Server Stub
│           └── financeservice.go
│
└── internal/
    ├── cfg/
    │   ├── mysql.go                     ← GORM 数据库连接
    │   └── redis.go                     ← Redis 连接
    │
    ├── enum/
    │   ├── channel_type.go              ← 通道支付类型枚举
    │   ├── order_status.go              ← 订单状态枚举（充值/提现）
    │   ├── audit_status.go              ← 审核状态枚举
    │   ├── adjustment_type.go           ← 修正类型枚举
    │   └── deposit_method.go            ← 充值/提现方式类型枚举
    │
    ├── errs/
    │   └── code.go                      ← 全模块错误码
    │
    ├── ser/
    │   ├── channel_service.go           ← 通道配置管理
    │   ├── payment_service.go           ← 支付配置管理
    │   ├── deposit_service.go           ← 充值订单+回调处理
    │   ├── withdraw_service.go          ← 提现订单+审核流程+出款
    │   ├── adjustment_service.go        ← 人工修正+双人审核
    │   ├── statistics_service.go        ← 数据统计+导出
    │   ├── polling_engine.go            ← 通道轮询算法引擎（特有）
    │   └── channel_dispatcher.go        ← 三方通道调度+适配（特有）
    │
    ├── rep/
    │   ├── channel_rep.go               ← 通道数据访问
    │   ├── payment_rep.go               ← 支付配置数据访问
    │   ├── deposit_rep.go               ← 充值订单数据访问
    │   ├── withdraw_rep.go              ← 提现订单数据访问
    │   ├── adjustment_rep.go            ← 修正订单数据访问
    │   ├── statistics_rep.go            ← 统计数据访问
    │   └── polling_rep.go               ← 轮询规则数据访问
    │
    ├── cache/
    │   ├── channel_cache.go             ← 通道配置缓存
    │   └── payment_cache.go             ← 支付配置缓存
    │
    ├── gen/
    │   └── gorm_gen.go                  ← GORM Gen 代码生成入口
    │
    └── gorm_gen/                         ← [自动生成] gorm_gen.go 产出
        ├── model/
        │   ├── deposit_channel.gen.go
        │   ├── payout_channel.gen.go
        │   ├── polling_rule.gen.go
        │   ├── deposit_method_config.gen.go
        │   ├── withdraw_method_config.gen.go
        │   ├── finance_deposit_order.gen.go
        │   ├── finance_withdraw_order.gen.go
        │   ├── supplement_order.gen.go
        │   ├── manual_credit_order.gen.go
        │   ├── manual_debit_order.gen.go
        │   ├── channel_daily_stat.gen.go
        │   ├── deposit_daily_stat.gen.go
        │   ├── withdraw_daily_stat.gen.go
        │   └── export_record.gen.go
        ├── query/
        │   ├── gen.go
        │   ├── deposit_channel.gen.go
        │   ├── payout_channel.gen.go
        │   ├── polling_rule.gen.go
        │   ├── deposit_method_config.gen.go
        │   ├── withdraw_method_config.gen.go
        │   ├── finance_deposit_order.gen.go
        │   ├── finance_withdraw_order.gen.go
        │   ├── supplement_order.gen.go
        │   ├── manual_credit_order.gen.go
        │   ├── manual_debit_order.gen.go
        │   ├── channel_daily_stat.gen.go
        │   ├── deposit_daily_stat.gen.go
        │   ├── withdraw_daily_stat.gen.go
        │   └── export_record.gen.go
        └── repo/
            ├── depositchannelr/
            │   └── depositchannelr.go
            ├── payoutchannelr/
            │   └── payoutchannelr.go
            ├── pollingruler/
            │   └── pollingruler.go
            ├── depositmethodconfigr/
            │   └── depositmethodconfigr.go
            ├── withdrawmethodconfigr/
            │   └── withdrawmethodconfigr.go
            ├── financedepositorderr/
            │   └── financedepositorderr.go
            ├── financewithdraworderr/
            │   └── financewithdraworderr.go
            ├── supplementorderr/
            │   └── supplementorderr.go
            ├── manualcreditorderr/
            │   └── manualcreditorderr.go
            ├── manualdebitorderr/
            │   └── manualdebitorderr.go
            ├── channeldailystatr/
            │   └── channeldailystatr.go
            ├── depositdailystatr/
            │   └── depositdailystatr.go
            ├── withdrawdailystatr/
            │   └── withdrawdailystatr.go
            └── exportrecordr/
                └── exportrecordr.go
```

### 1.2 IDL 契约文件

```
common/idl/ser-finance/
├── service.thrift                       ← 入口：FinanceService 全部方法声明
├── channel.thrift                       ← 通道配置类型
├── payment.thrift                       ← 支付配置类型
├── deposit.thrift                       ← 充值订单类型
├── withdraw.thrift                      ← 提现订单类型
├── adjustment.thrift                    ← 人工修正类型
├── statistics.thrift                    ← 统计类型
├── finance_rpc.thrift                   ← RPC 对外类型（供钱包调用）
└── callback.thrift                      ← 三方通道回调类型
```

### 1.3 网关接入文件

```
gate-back/biz/
├── handler/finance/
│   ├── channel.go                       ← 通道配置 Handler（8个函数）
│   ├── payment.go                       ← 支付配置 Handler（6个函数）
│   ├── deposit.go                       ← 充值记录 Handler（2个函数）
│   ├── withdraw.go                      ← 提现记录 Handler（6个函数）
│   ├── adjustment.go                    ← 人工修正 Handler（9个函数）
│   └── statistics.go                    ← 数据统计 Handler（7个函数）
└── router/finance/
    └── finance_router.go                ← B端路由注册 RouterFinance()

gate-font/biz/
├── handler/finance/
│   └── callback.go                      ← 三方通道回调 Handler（2-3个函数）
└── router/finance/
    └── finance_router.go                ← 回调路由注册（Open组，无需鉴权）
```

### 1.4 公共注册变更

```
需变更的已有文件（非新建）：

common/pkg/consts/namesp/namesp.go       ← +3 行：EtcdFinanceService/Port/Db
common/rpc/rpc_client.go                 ← +1 工厂方法：FinanceClient()
go.work                                  ← +1 行：./ser-finance
gate-back/biz/router/register.go         ← +1 行：finance.RouterFinance(r)
gate-font/biz/router/register.go         ← +1 行：finance.RouterFinance(r)
```

---

## 二、文件统计总览

| 分类 | 新建文件数 | 自动生成文件数 | 变更已有文件数 | 合计 |
|------|-----------|--------------|-------------|------|
| ser-finance 根目录 | 5 | 0 | 0 | 5 |
| internal/cfg | 2 | 0 | 0 | 2 |
| internal/enum | 5 | 0 | 0 | 5 |
| internal/errs | 1 | 0 | 0 | 1 |
| internal/ser | 8 | 0 | 0 | 8 |
| internal/rep | 7 | 0 | 0 | 7 |
| internal/cache | 2 | 0 | 0 | 2 |
| internal/gorm_gen | 0 | 43 | 0 | 43 |
| kitex_gen | 0 | ~14 | 0 | ~14 |
| sql | 1 | 0 | 0 | 1 |
| IDL（common/idl） | 9 | 0 | 0 | 9 |
| 网关 handler | 7 | 0 | 0 | 7 |
| 网关 router | 2 | 0 | 0 | 2 |
| 公共注册 | 0 | 0 | 5 | 5 |
| **合计** | **~49** | **~57** | **5** | **~111** |

> ser-finance 的手写文件数（~49）比 ser-wallet（~42）更多，因为：财务有 8 个子模块（vs 钱包 5 个域），B 端路由更多（~38 vs 7），gate-back handler 需要按子模块拆分为 6 个文件。

---

## 三、核心差异：ser-finance vs ser-wallet

在展开每个目录的详解之前，先明确 ser-finance 区别于 ser-wallet（以及其他已有服务）的**两个特有层**：

### 3.1 结构对比图

```
               ser-wallet                          ser-finance
               ──────────                          ───────────
  handler.go   → WalletServiceImpl                → FinanceServiceImpl
  ┌────────────────────────┐        ┌────────────────────────────┐
  │ ser/ 业务逻辑层         │        │ ser/ 业务逻辑层              │
  │                        │        │                            │
  │  业务编排               │        │  业务编排                    │
  │  (deposit/withdraw/    │        │  (deposit/withdraw/         │
  │   exchange/record)     │        │   adjustment/statistics)    │
  │         │              │        │         │                   │
  │  ┌──────▼─────┐       │        │  ┌──────▼──────────┐       │
  │  │ wallet_core│ ★特有  │        │  │ polling_engine  │ ★特有  │
  │  │ 6个原子操作 │       │        │  │ 通道轮询算法     │        │
  │  └────────────┘       │        │  └─────────────────┘        │
  │                        │        │  ┌───────────────────┐      │
  │     （无对外通道对接）   │        │  │ channel_dispatcher│ ★特有 │
  │                        │        │  │ 三方通道调度+适配  │      │
  │                        │        │  └───────────────────┘      │
  └────────────────────────┘        └────────────────────────────┘
  │                                 │
  rep/ → 自己的表                    rep/ → 自己的表
  │                                 │
  gorm_gen/ → 14张表                 gorm_gen/ → 14张表
```

### 3.2 两个特有层说明

| 特有层 | 类比 ser-wallet | 职责 | 为什么需要 |
|--------|----------------|------|-----------|
| **polling_engine.go** | wallet_core.go | 通道轮询评分算法：成功率×系数 + 时间×系数 + 预警×系数，综合评分排序选出最优通道 | 每次创建充值/提现订单都要选通道，算法是核心竞争力 |
| **channel_dispatcher.go** | 无（钱包不对接通道） | 三方支付通道的统一调度：根据选中的通道，调用对应通道的 API 创建代收/代付订单，处理回调 | 每个三方通道的 API 格式不同，需要适配层统一封装 |

> 这两个文件是 ser-finance 区别于所有已有服务的核心。就像 wallet_core 是钱包的"账务引擎"，polling_engine + channel_dispatcher 是财务的"支付引擎"。

---

## 四、internal/ 分层架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ser-finance internal/ 架构                       │
│                                                                     │
│  ┌─── cfg/ ───┐     初始化层                                        │
│  │ mysql.go   │     TiDB + Redis 连接（与 ser-wallet 共用相同模式）    │
│  │ redis.go   │                                                     │
│  └────────────┘                                                     │
│        │                                                            │
│        ▼                                                            │
│  ┌─── enum/ + errs/ ───┐   常量层                                   │
│  │ channel_type.go     │   通道类型 / 订单状态 / 审核状态 / 修正类型    │
│  │ order_status.go ... │                                            │
│  │ errs/code.go        │                                            │
│  └─────────────────────┘                                            │
│        │                                                            │
│        ▼                                                            │
│  ┌─── ser/ ──────────────────────────────────────────────────────┐  │
│  │                                                               │  │
│  │  ┌─ B端业务 ──────────────────────────────────────────────┐  │  │
│  │  │ channel_service    通道CRUD + 启禁用                    │  │  │
│  │  │ payment_service    充值/提现方式CRUD                    │  │  │
│  │  │ adjustment_service 人工修正（补单/加款/减款+双人审核）    │  │  │
│  │  │ statistics_service 三种统计 + 导出                      │  │  │
│  │  └──────────────────────────────────────────────────────-─┘  │  │
│  │                                                               │  │
│  │  ┌─ 订单流程 ─────────────────────────────────────────────┐  │  │
│  │  │ deposit_service    充值订单生命周期 + 回调处理            │  │  │
│  │  │ withdraw_service   提现订单生命周期 + 审核 + 出款         │  │  │
│  │  └──────────────┬─────────────────────────────────────────┘  │  │
│  │                 │ 调用                                        │  │
│  │  ┌─ 支付引擎 ──▼───────────────────────────────────────┐    │  │
│  │  │ polling_engine.go                                    │    │  │
│  │  │ 三系数评分：成功率×a + 时间×b + 预警×c → 通道排序选优   │    │  │
│  │  │                                                      │    │  │
│  │  │ channel_dispatcher.go                                │    │  │
│  │  │ 通道适配：统一接口 → 路由到具体通道API → 创建订单/查询  │    │  │
│  │  │ 回调解析：不同通道回调格式 → 统一订单状态更新            │    │  │
│  │  └──────────────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│        │                                                            │
│        ▼                                                            │
│  ┌─── rep/ + cache/ ───┐   数据访问层                                │
│  │ channel_rep.go      │   7 个 rep 文件 + 2 个 cache 文件           │
│  │ deposit_rep.go  ... │                                            │
│  └─────────────────────┘                                            │
│        │                                                            │
│        ▼                                                            │
│  ┌─── gorm_gen/ ───┐   生成代码层                                    │
│  │ model/query/repo │   14 张表自动生成                               │
│  └─────────────────┘                                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、internal/ser/ — 业务逻辑层详解

```
internal/ser/
├── channel_service.go           ← 通道配置域
├── payment_service.go           ← 支付配置域
├── deposit_service.go           ← 充值订单域
├── withdraw_service.go          ← 提现订单域
├── adjustment_service.go        ← 人工修正域
├── statistics_service.go        ← 数据统计域
├── polling_engine.go            ← 通道轮询引擎（特有）
└── channel_dispatcher.go        ← 三方通道调度（特有）
```

### 5.1 文件职责详解

| 文件 | 对应需求子模块 | 职责 | 预估 IDL 方法 | 调用钱包接口 |
|------|--------------|------|-------------|-------------|
| **channel_service.go** | 2.1 通道配置 | 代收通道 CRUD + 代付通道 CRUD + 轮询规则 CRUD + 通道敏感配置加密存储 | B端 8 | 无 |
| **payment_service.go** | 2.2 支付配置 | 充值方式 CRUD + 提现方式 CRUD + 关联通道绑定 + 图标上传 | B端 6 | 无 |
| **deposit_service.go** | 2.3 充值记录 | 充值订单分页查询/导出 + 创建通道订单 + 接收三方回调 + 回调后调用钱包上账 | B端 2 + RPC 1 + 回调 1 | **CreditWallet**（充值成功上账） |
| **withdraw_service.go** | 2.4 提现记录 | 提现订单管理 + 风控审核 + 财务审核 + 出款（自动匹配通道/人工出款）+ 锁定/解锁 | B端 6 + RPC 1 | **ConfirmDebit**（出款成功）/ **UnfreezeBalance**（审核驳回/出款失败） |
| **adjustment_service.go** | 2.5 人工修正 | 充值补单（B+前缀）+ 人工加款（A+前缀）+ 人工减款（M+前缀）+ 双人审核 | B端 9 | **SupplementCredit** / **ManualCredit** / **ManualDebit** |
| **statistics_service.go** | 3. 数据统计 | 代收/代付通道统计 + 充值统计 + 提现统计 + 基准币种折算 + 导出 | B端 7 | 无 |
| **polling_engine.go** | 2.13 轮询规则 | 通道评分算法：三系数加权排序 + 同分时按权重>时间排序 | 无（内部调用） | 无 |
| **channel_dispatcher.go** | — | 三方通道统一适配：创建代收/代付订单 + 查询订单状态 + 解析回调 | 无（内部调用） | 无 |

### 5.2 Service 依赖关系图

```
  ┌──────────────────┐     ┌──────────────────┐
  │ channel_service  │     │ payment_service  │
  │ （通道CRUD）      │     │ （支付方式CRUD）  │
  └────────┬─────────┘     └────────┬─────────┘
           │                        │
           │  被引用                  │  被引用
           ▼                        ▼
  ┌────────────────────────────────────────────┐
  │           polling_engine                    │
  │  评分排序 → 选出最优通道                      │
  └───────────────────┬────────────────────────┘
                      │
                      ▼
  ┌────────────────────────────────────────────┐
  │         channel_dispatcher                  │
  │  调用选中通道的 API → 创建订单 / 处理回调      │
  └───────────────────┬────────────────────────┘
                      │
           ┌──────────┴──────────┐
           ▼                     ▼
  ┌─────────────────┐  ┌─────────────────┐
  │deposit_service  │  │withdraw_service │
  │ 创建充值订单     │  │ 创建提现订单     │
  │ 回调 → 调钱包    │  │ 审核 → 调钱包    │
  │ CreditWallet    │  │ Confirm/Unfreeze│
  └─────────────────┘  └─────────────────┘
           │                     │
           ▼                     ▼
       ser-wallet             ser-wallet
      (RPC 调用)             (RPC 调用)

  ┌─────────────────┐  ┌─────────────────┐
  │adjustment_service│  │statistics_service│
  │ 审核→调钱包      │  │ 纯查询+统计      │
  │ 补单/加款/减款   │  │ 不调用钱包       │
  └─────────────────┘  └─────────────────┘
```

### 5.3 polling_engine.go 特别说明

通道轮询是财务模块的核心算法，每次创建充值/提现订单都要执行：

```
通道轮询算法（需求 2.13）：

输入：币种 + 订单类型（代收/代付）+ 订单金额
输出：最优通道

评分规则：
  综合评分 = 成功率 × 成功率系数 + 时间评分 × 时间系数 + 预警评分 × 预警系数

  ┌─────────────┬───────────────────────────────────────────┐
  │ 维度         │ 计算方式                                   │
  ├─────────────┼───────────────────────────────────────────┤
  │ 成功率       │ 该通道近期成功率 → 映射到区间分值            │
  │ 时间         │ 该通道平均入款/出款时间 → 映射到区间分值      │
  │ 预警         │ 通道当日累计金额 vs 预警金额 → 映射到区间分值 │
  └─────────────┴───────────────────────────────────────────┘

  三个系数由 B 端配置（polling_rule 表），可动态调整权重

  排序规则：
    1. 综合评分降序
    2. 评分相同 → 代收权重降序
    3. 权重相同 → 时间优先级

  额外过滤：
    - 通道状态必须启用
    - 订单金额在通道单笔限额范围内
    - 代付通道每日限额未超
```

### 5.4 channel_dispatcher.go 特别说明

这是 ser-finance 最独特的文件——它是整个系统中**唯一直接对接三方支付通道**的代码：

```
channel_dispatcher 职责：

  ┌──────────────────────────────────────────────────┐
  │         统一调度接口                               │
  │                                                   │
  │  CreateDeposit(channel, order)  → 创建代收订单      │
  │  CreatePayout(channel, order)   → 创建代付订单      │
  │  QueryOrder(channel, orderNo)   → 查询订单状态      │
  │  ParseCallback(channel, body)   → 解析回调数据      │
  └───────────────────┬──────────────────────────────┘
                      │ 路由到具体通道实现
          ┌───────────┼───────────────┐
          ▼           ▼               ▼
    ┌──────────┐ ┌──────────┐  ┌──────────┐
    │ 通道 A   │ │ 通道 B   │  │ 通道 C   │
    │ API 格式A│ │ API 格式B│  │ API 格式C│
    │ 签名方式A│ │ 签名方式B│  │ 签名方式C│
    │ 回调格式A│ │ 回调格式B│  │ 回调格式C│
    └──────────┘ └──────────┘  └──────────┘

  每接入一个新通道，增加一个适配实现
  通道的 API 地址/商户ID/密钥 等从 deposit_channel/payout_channel 表加密读取
```

> **为什么这对钱包开发很重要**：钱包调 `CreatePayOrder` 后，财务的 `deposit_service` 会调 `polling_engine` 选通道，再调 `channel_dispatcher` 创建三方订单。三方回调到 `channel_dispatcher` 解析后，`deposit_service` 更新订单状态，然后调钱包的 `CreditWallet`。**整个链路中的每一步对你来说都是黑盒**——了解这个结构，至少知道"我的回调为什么还没来"应该去看哪个环节。

---

## 六、internal/enum/ — 枚举常量层

```
internal/enum/
├── channel_type.go
├── order_status.go
├── audit_status.go
├── adjustment_type.go
└── deposit_method.go
```

| 文件 | 定义内容 | 预估枚举值 | 需求来源 |
|------|---------|-----------|---------|
| channel_type.go | 通道支付类型：唤醒、原生、扫码、混合 | 4 个 | 需求 2.11 "支付类型" |
| order_status.go | 充值状态：待支付/支付成功/支付失败；提现状态：待风控审核/风控驳回/待财务审核/财务驳回/待出款/出款失败/出款成功 | 3 + 7 = 10 个 | 需求 2.3 / 2.4 |
| audit_status.go | 人工修正审核状态：待审核/审核通过/审核拒绝 | 3 个 | 需求 2.5 |
| adjustment_type.go | 加款类型：运营补偿/运营奖励/财务修正/测试加款；减款类型：异常追回/风控处罚/财务冲正/费用调整 | 4 + 4 = 8 个 | 需求 2.52 / 2.53 |
| deposit_method.go | 充值/提现方式类型标识 | 按需扩展 | 需求 2.21 / 2.22 |

---

## 七、internal/errs/ — 错误码层

```
internal/errs/
└── code.go
```

### 错误码分段规划

```
60{YY} = 财务模块服务编号（待分配，如 023）

┌──────────────┬──────────────┬──────────────────────────────────────┐
│ 段号          │ 范围          │ 覆盖模块                              │
├──────────────┼──────────────┼──────────────────────────────────────┤
│ 60YY 0xx     │ 000-099      │ 通道配置（ChannelNotFound 等）         │
│ 60YY 1xx     │ 100-199      │ 支付配置（MethodNotFound 等）          │
│ 60YY 2xx     │ 200-299      │ 充值订单（ChannelCreateFailed 等）     │
│ 60YY 3xx     │ 300-399      │ 提现订单（AuditPermissionDenied 等）   │
│ 60YY 4xx     │ 400-499      │ 人工修正（SelfAuditForbidden 等）      │
│ 60YY 5xx     │ 500-599      │ 统计/导出（ExportVerifyFailed 等）     │
│ 60YY 6xx     │ 600-699      │ 通道对接（CallbackSignInvalid 等）     │
└──────────────┴──────────────┴──────────────────────────────────────┘
```

---

## 八、internal/rep/ — 数据访问层

```
internal/rep/
├── channel_rep.go
├── payment_rep.go
├── deposit_rep.go
├── withdraw_rep.go
├── adjustment_rep.go
├── statistics_rep.go
└── polling_rep.go
```

| rep 文件 | 操作的表 | 典型自定义查询 |
|----------|---------|--------------|
| channel_rep.go | deposit_channel, payout_channel | 按币种+状态分页、按通道名称模糊搜索、敏感字段解密读取 |
| payment_rep.go | deposit_method_config, withdraw_method_config | 按币种+方式+状态筛选、排序、关联通道查询 |
| deposit_rep.go | finance_deposit_order | 多条件分页（24字段查询）、按通道订单号查询、按状态统计、导出查询 |
| withdraw_rep.go | finance_withdraw_order | 多条件分页（27字段查询）、状态流转更新、锁定/解锁操作、审核记录写入 |
| adjustment_rep.go | supplement_order, manual_credit_order, manual_debit_order | 三种修正订单各自的分页查询 + 创建 + 审核状态更新 |
| statistics_rep.go | channel_daily_stat, deposit_daily_stat, withdraw_daily_stat | 按日期+币种+通道聚合统计、基准币种折算查询、汇总卡片数据 |
| polling_rep.go | polling_rule | 轮询规则 CRUD + 三个系数区间配置 |

---

## 九、数据库表设计推演

### 9.1 表清单总览

```
┌────┬──────────────────────────────┬────────┬──────────────────────────────────────┐
│ 序号│ 表名                          │ 优先级  │ 一句话说明                             │
├────┼──────────────────────────────┼────────┼──────────────────────────────────────┤
│  1 │ deposit_channel               │ 必须   │ 代收通道配置（含加密敏感信息）            │
│  2 │ payout_channel                │ 必须   │ 代付通道配置（含加密敏感信息）            │
│  3 │ polling_rule                  │ 必须   │ 通道轮询规则（三系数区间配置）            │
│  4 │ deposit_method_config         │ 必须   │ 充值方式配置（档位/范围/关联通道）        │
│  5 │ withdraw_method_config        │ 必须   │ 提现方式配置（费率/限额/关联通道）        │
│  6 │ finance_deposit_order         │ 必须   │ 充值订单（24字段完整财务记录）            │
│  7 │ finance_withdraw_order        │ 必须   │ 提现订单（27字段完整财务记录+审核轨迹）    │
│  8 │ supplement_order              │ 必须   │ 充值补单（B+前缀，双人审核）             │
│  9 │ manual_credit_order           │ 必须   │ 人工加款（A+前缀，四种类型，双人审核）     │
│ 10 │ manual_debit_order            │ 必须   │ 人工减款（M+前缀，四种类型，双人审核）     │
│ 11 │ channel_daily_stat            │ 应该   │ 通道日统计（代收+代付维度）              │
│ 12 │ deposit_daily_stat            │ 应该   │ 充值日统计（按币种维度）                │
│ 13 │ withdraw_daily_stat           │ 应该   │ 提现日统计（含审核维度）                │
│ 14 │ export_record                 │ 可选   │ 导出记录（密码+谷歌验证码验证追踪）       │
└────┴──────────────────────────────┴────────┴──────────────────────────────────────┘
```

### 9.2 关键表字段方向

**finance_deposit_order（充值订单表）— 需求列出 24 个字段：**

```
核心字段方向：
  平台订单号 / 通道订单号 / 用户编号 / 昵称
  通道名称 / 订单币种 / 充值方式 / 订单状态
  订单金额 / 活动类型 / 活动优惠 / 充值费率 / 手续费 / 到账总金额
  入款汇率 / 支付币种 / 支付金额
  平台汇率 / 基准币种订单金额 / 基准币种活动优惠 / 基准币种手续费
  创建时间 / 完成时间
```

**finance_withdraw_order（提现订单表）— 需求列出 27 个字段：**

```
核心字段方向：
  平台订单号 / 通道订单号 / 用户编号 / 昵称
  通道名称 / 订单币种 / 提现方式 / 订单状态（7种）
  订单金额 / 提现费率 / 手续费
  出款汇率 / 出款币种 / 出款金额
  平台汇率 / 基准币种订单金额 / 基准币种手续费
  拒绝原因 / 审核备注
  锁定状态 / 锁定人 / 锁定时间
  风控审核人 / 风控审核时间 / 财务审核人 / 财务审核时间
  创建时间 / 完成时间
```

**deposit_channel / payout_channel（通道配置表）— 含敏感字段：**

```
常规字段：
  币种 / 通道名称 / 支付类型 / 费率 / 单笔限额 / 权重 / 预警金额 / 状态

敏感字段（需 AES-256 加密存储）：
  请求地址 / 商户ID / 密钥 / AppKey / 支付编码 / 公钥 / 私钥

代付通道额外字段：
  关联代收通道 / 每日限额
```

> **与钱包表的关系**：财务的 `finance_deposit_order` 和钱包的 `deposit_order` 是**两张不同的表**。财务表记录完整的财务维度（通道信息、费率、汇率折算、基准币种金额），钱包表记录简化的用户维度（金额、状态、关联订单号）。两者通过平台订单号关联。

---

## 十、IDL 契约文件详解

```
common/idl/ser-finance/
├── service.thrift           ← 入口
├── channel.thrift           ← 通道配置
├── payment.thrift           ← 支付配置
├── deposit.thrift           ← 充值订单
├── withdraw.thrift          ← 提现订单
├── adjustment.thrift        ← 人工修正
├── statistics.thrift        ← 统计
├── finance_rpc.thrift       ← RPC 对外
└── callback.thrift          ← 三方回调
```

### 10.1 service.thrift 方法分组

```
service FinanceService {
    // ── B端接口（gate-back 转发，共 ~38 个）────────────────────

    // 通道配置（8个）
    channel.PageDepositChannelResp    PageDepositChannel(...)
    channel.EditDepositChannelResp    EditDepositChannel(...)
    channel.PagePayoutChannelResp     PagePayoutChannel(...)
    channel.EditPayoutChannelResp     EditPayoutChannel(...)
    channel.PagePollingRuleResp       PagePollingRule(...)
    channel.CreatePollingRuleResp     CreatePollingRule(...)
    channel.EditPollingRuleResp       EditPollingRule(...)
    channel.DeletePollingRuleResp     DeletePollingRule(...)

    // 支付配置（6个）
    payment.PageDepositMethodResp     PageDepositMethod(...)
    payment.CreateDepositMethodResp   CreateDepositMethod(...)
    payment.EditDepositMethodResp     EditDepositMethod(...)
    payment.PageWithdrawMethodResp    PageWithdrawMethod(...)
    payment.CreateWithdrawMethodResp  CreateWithdrawMethod(...)
    payment.EditWithdrawMethodResp    EditWithdrawMethod(...)

    // 充值记录（2个）
    deposit.PageDepositOrderResp      PageDepositOrder(...)
    deposit.ExportDepositOrderResp    ExportDepositOrder(...)

    // 提现记录（6个）
    withdraw.PageWithdrawOrderResp    PageWithdrawOrder(...)
    withdraw.AuditWithdrawOrderResp   AuditWithdrawOrder(...)
    withdraw.LockWithdrawOrderResp    LockWithdrawOrder(...)
    withdraw.UnlockWithdrawOrderResp  UnlockWithdrawOrder(...)
    withdraw.ForceUnlockResp          ForceUnlockWithdrawOrder(...)
    withdraw.ExportWithdrawOrderResp  ExportWithdrawOrder(...)

    // 人工修正（9个）
    adjustment.PageSupplementResp         PageSupplement(...)
    adjustment.CreateSupplementResp       CreateSupplement(...)
    adjustment.AuditSupplementResp        AuditSupplement(...)
    adjustment.PageManualCreditResp       PageManualCredit(...)
    adjustment.CreateManualCreditResp     CreateManualCredit(...)
    adjustment.AuditManualCreditResp      AuditManualCredit(...)
    adjustment.PageManualDebitResp        PageManualDebit(...)
    adjustment.CreateManualDebitResp      CreateManualDebit(...)
    adjustment.AuditManualDebitResp       AuditManualDebit(...)

    // 数据统计（7个）
    statistics.GetDepositChannelStatResp  GetDepositChannelStat(...)
    statistics.GetPayoutChannelStatResp   GetPayoutChannelStat(...)
    statistics.ExportChannelStatResp      ExportChannelStat(...)
    statistics.GetDepositStatResp         GetDepositStat(...)
    statistics.ExportDepositStatResp      ExportDepositStat(...)
    statistics.GetWithdrawStatResp        GetWithdrawStat(...)
    statistics.ExportWithdrawStatResp     ExportWithdrawStat(...)

    // ── RPC 对外接口（供 ser-wallet 调用，共 3 个）─────────────

    finance_rpc.GetPaymentConfigResp      GetPaymentConfig(...)
    finance_rpc.CreatePayOrderResp        CreatePayOrder(...)
    finance_rpc.SubmitWithdrawOrderResp   SubmitWithdrawOrder(...)

    // ── 三方通道回调（gate-font Open 转发，共 2 个）────────────

    callback.DepositCallbackResp          DepositCallback(...)
    callback.WithdrawCallbackResp         WithdrawCallback(...)
}
```

> 合计：~43 个方法（B端 38 + RPC 3 + 回调 2）。对比 ser-wallet ~30 个方法，ser-finance 的方法数更多，主要因为 B 端管理功能更繁重（8 个子模块的 CRUD + 审核 + 统计 + 导出）。

### 10.2 finance_rpc.thrift — 钱包最关心的 3 个 RPC 接口

这是**钱包开发者最需要关注的 IDL 文件**——钱包调用财务模块的接口全在这里。

```
file: finance_rpc.thrift

┌──────────────────────┬──────────────────────────────────────────────────┐
│ 方法                  │ 钱包什么时候调                                    │
├──────────────────────┼──────────────────────────────────────────────────┤
│ GetPaymentConfig     │ 钱包 C端 GetDepositMethods / GetWithdrawMethods  │
│                      │ 用户打开充值/提现页时，获取可用方式+档位+费率+限额  │
│                      │                                                  │
│ CreatePayOrder       │ 钱包 C端 CreateDepositOrder                      │
│                      │ 用户确认充值时，传递订单信息，财务匹配通道+创建订单 │
│                      │                                                  │
│ SubmitWithdrawOrder  │ 钱包 C端 CreateWithdrawOrder                     │
│                      │ 用户确认提现时（余额已冻结），提交给财务进入审核流程 │
└──────────────────────┴──────────────────────────────────────────────────┘
```

**GetPaymentConfig 请求/响应方向：**

```
请求（钱包 → 财务）：
  currencyCode  ← 币种代码（VND/IDR/THB/BSB）
  configType    ← "deposit" 或 "withdraw"

响应（财务 → 钱包）：
  configType = "deposit" 时：
    list<DepositMethodItem>  充值方式列表
      ├── methodName         方式名称（如"越南银行转账"）
      ├── methodIcon         方式图标 URL
      ├── amountTiers        档位金额列表（如 [100000, 200000, 500000, ...]）
      ├── minAmount          最低金额
      ├── maxAmount          最高金额
      └── sort               排序值

  configType = "withdraw" 时：
    list<WithdrawMethodItem>  提现方式列表
      ├── methodName          方式名称（如"银行卡"）
      ├── methodIcon          方式图标 URL
      ├── feeRate             手续费比例
      ├── largeAmountFeeRate  大额手续费比例
      ├── minAmount           单笔最低
      ├── maxAmount           单笔最高
      ├── dailyLimit          每日限额
      └── sort                排序值
```

**CreatePayOrder 请求/响应方向：**

```
请求（钱包 → 财务）：
  userId         ← 用户ID
  currencyCode   ← 币种
  amount         ← 充值金额
  depositMethod  ← 充值方式
  bonusId        ← 奖金活动ID（可选）
  callbackInfo   ← 钱包侧订单号等（用于回调匹配）

响应（财务 → 钱包）：
  financeOrderNo   ← 财务平台订单号
  channelOrderNo   ← 通道订单号
  paymentInfo      ← 支付信息
    ├── type        "redirect" / "qrcode" / "address"
    ├── url         三方承载页 URL（redirect 类型）
    ├── qrcodeUrl   二维码图片 URL（qrcode 类型）
    ├── address     USDT 收款地址（address 类型）
    └── expireTime  支付超时时间
```

---

## 十一、网关路由详解

### 11.1 gate-back（B端管理后台）

B 端 handler 按子模块拆分为 6 个文件（对比 ser-wallet 只有 1 个 wallet.go），因为财务的 B 端功能远多于钱包。

```
gate-back/biz/handler/finance/

┌──────────────┬─────────────────────────────────────────────────────────────┐
│ 文件          │ 路由路径示例                                                 │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ channel.go   │ POST /admin/api/finance/channel/deposit/page                │
│   (8个函数)  │ POST /admin/api/finance/channel/deposit/edit                │
│              │ POST /admin/api/finance/channel/payout/page                  │
│              │ POST /admin/api/finance/channel/payout/edit                  │
│              │ POST /admin/api/finance/channel/polling/page                 │
│              │ POST /admin/api/finance/channel/polling/create               │
│              │ POST /admin/api/finance/channel/polling/edit                 │
│              │ POST /admin/api/finance/channel/polling/delete               │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ payment.go   │ POST /admin/api/finance/payment/deposit/page                │
│   (6个函数)  │ POST /admin/api/finance/payment/deposit/create              │
│              │ POST /admin/api/finance/payment/deposit/edit                 │
│              │ POST /admin/api/finance/payment/withdraw/page                │
│              │ POST /admin/api/finance/payment/withdraw/create              │
│              │ POST /admin/api/finance/payment/withdraw/edit                │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ deposit.go   │ POST /admin/api/finance/deposit/order/page                  │
│   (2个函数)  │ POST /admin/api/finance/deposit/order/export                │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ withdraw.go  │ POST /admin/api/finance/withdraw/order/page                 │
│   (6个函数)  │ POST /admin/api/finance/withdraw/order/audit                │
│              │ POST /admin/api/finance/withdraw/order/lock                  │
│              │ POST /admin/api/finance/withdraw/order/unlock                │
│              │ POST /admin/api/finance/withdraw/order/force-unlock          │
│              │ POST /admin/api/finance/withdraw/order/export                │
├──────────────┼─────────────────────────────────────────────────────────────┤
│adjustment.go │ POST /admin/api/finance/adjustment/supplement/page          │
│   (9个函数)  │ POST /admin/api/finance/adjustment/supplement/create        │
│              │ POST /admin/api/finance/adjustment/supplement/audit          │
│              │ POST /admin/api/finance/adjustment/credit/page               │
│              │ POST /admin/api/finance/adjustment/credit/create             │
│              │ POST /admin/api/finance/adjustment/credit/audit              │
│              │ POST /admin/api/finance/adjustment/debit/page                │
│              │ POST /admin/api/finance/adjustment/debit/create              │
│              │ POST /admin/api/finance/adjustment/debit/audit               │
├──────────────┼─────────────────────────────────────────────────────────────┤
│statistics.go │ POST /admin/api/finance/statistics/channel/deposit          │
│   (7个函数)  │ POST /admin/api/finance/statistics/channel/payout           │
│              │ POST /admin/api/finance/statistics/channel/export            │
│              │ POST /admin/api/finance/statistics/deposit                   │
│              │ POST /admin/api/finance/statistics/deposit/export            │
│              │ POST /admin/api/finance/statistics/withdraw                  │
│              │ POST /admin/api/finance/statistics/withdraw/export           │
└──────────────┴─────────────────────────────────────────────────────────────┘

共 38 个路由，全部挂载 my_router.Api（需 Token + URL 权限验证）
```

### 11.2 gate-font（三方通道回调入口）

```
gate-font/biz/handler/finance/
└── callback.go                  ← 2-3 个函数

路由（挂载 my_router.Open，无需鉴权）：
  POST /open/finance/callback/deposit     ← 充值回调（三方通道→平台）
  POST /open/finance/callback/withdraw    ← 提现/出款回调（三方通道→平台）
```

> **为什么回调走 gate-font Open 组**：三方支付通道的回调是服务端到服务端的 HTTP 请求，不携带平台用户 Token。gate-font 的 Open 组不做鉴权，适合接收外部回调。回调的安全性通过**签名验证**保障（每个通道有不同的签名算法，在 `channel_dispatcher.go` 中实现）。

---

## 十二、与 ser-wallet 的交互全景

这是本文档最核心的章节——钱包和财务之间所有 RPC 调用的完整映射。

### 12.1 交互方向一览

```
               ser-wallet                    ser-finance
               ──────────                    ───────────

  ┌─ 钱包→财务（同步，C端用户操作时）──────────────────────────────┐
  │                                                              │
  │  wallet/deposit_service ──RPC──→ finance/payment_service     │
  │  GetDepositMethods()       GetPaymentConfig(deposit)         │
  │                                                              │
  │  wallet/withdraw_service ──RPC──→ finance/payment_service    │
  │  GetWithdrawMethods()       GetPaymentConfig(withdraw)       │
  │                                                              │
  │  wallet/deposit_service ──RPC──→ finance/deposit_service     │
  │  CreateDepositOrder()       CreatePayOrder()                 │
  │                                                              │
  │  wallet/withdraw_service ──RPC──→ finance/withdraw_service   │
  │  CreateWithdrawOrder()      SubmitWithdrawOrder()            │
  └──────────────────────────────────────────────────────────────┘

  ┌─ 财务→钱包（异步，回调/审核后触发）──────────────────────────────┐
  │                                                              │
  │  finance/deposit_service ──RPC──→ wallet/wallet_core         │
  │  （充值回调成功后）          CreditWallet()                    │
  │                                                              │
  │  finance/withdraw_service ──RPC──→ wallet/wallet_core        │
  │  （出款成功后）              ConfirmDebit()                    │
  │                                                              │
  │  finance/withdraw_service ──RPC──→ wallet/wallet_core        │
  │  （审核驳回/出款失败后）     UnfreezeBalance()                 │
  │                                                              │
  │  finance/adjustment_service ──RPC──→ wallet/wallet_core      │
  │  （补单审核通过后）          SupplementCredit()                │
  │                                                              │
  │  finance/adjustment_service ──RPC──→ wallet/wallet_core      │
  │  （加款审核通过后）          ManualCredit()                    │
  │                                                              │
  │  finance/adjustment_service ──RPC──→ wallet/wallet_core      │
  │  （减款审核通过后）          ManualDebit()                     │
  └──────────────────────────────────────────────────────────────┘
```

### 12.2 交互详细映射表

| 方向 | 触发时机 | 钱包侧文件 | 财务侧文件 | RPC 方法 | 调用方式 |
|------|---------|-----------|-----------|---------|---------|
| 钱包→财务 | C端打开充值页 | deposit_service.go | payment_service.go | GetPaymentConfig | 同步 |
| 钱包→财务 | C端打开提现页 | withdraw_service.go | payment_service.go | GetPaymentConfig | 同步 |
| 钱包→财务 | C端确认充值 | deposit_service.go | deposit_service.go | CreatePayOrder | 同步 |
| 钱包→财务 | C端确认提现 | withdraw_service.go | withdraw_service.go | SubmitWithdrawOrder | 同步 |
| 财务→钱包 | 充值回调成功 | wallet_core.go | deposit_service.go | CreditWallet | 异步 |
| 财务→钱包 | 出款成功 | wallet_core.go | withdraw_service.go | ConfirmDebit | 异步 |
| 财务→钱包 | 审核驳回/出款失败 | wallet_core.go | withdraw_service.go | UnfreezeBalance | 异步 |
| 财务→钱包 | 补单审核通过 | wallet_core.go | adjustment_service.go | SupplementCredit | 异步 |
| 财务→钱包 | 加款审核通过 | wallet_core.go | adjustment_service.go | ManualCredit | 异步 |
| 财务→钱包 | 减款审核通过 | wallet_core.go | adjustment_service.go | ManualDebit | 异步 |

### 12.3 关键链路时序

**充值全链路**：

```
C端App     ser-wallet              ser-finance              三方通道
  │                                    │                       │
  │─ GetDepositMethods ─→│             │                       │
  │                      │──RPC──→ GetPaymentConfig            │
  │                      │←───── 充值方式列表 ────│             │
  │← 充值页数据 ─────────│             │                       │
  │                                    │                       │
  │─ CreateDepositOrder ─→│            │                       │
  │                      │──RPC──→ CreatePayOrder              │
  │                      │            │── polling_engine 选通道  │
  │                      │            │── channel_dispatcher ──→│ 创建代收订单
  │                      │            │←───── 支付信息 ─────────│
  │                      │←──── 订单+支付信息 ───│              │
  │← 跳转支付 ───────────│             │                       │
  │                                    │                       │
  │  （用户去三方页面支付）              │           回调 ────────→│
  │                                    │←────── 支付结果 ──────│
  │                                    │                       │
  │                      │←──RPC── CreditWallet                │
  │                      │  （更新余额+写入流水）               │
  │                                    │                       │
```

**提现审核全链路**：

```
C端App     ser-wallet              ser-finance              三方通道
  │                                    │                       │
  │─ CreateWithdrawOrder ─→│           │                       │
  │                       │ 冻结余额    │                       │
  │                       │──RPC──→ SubmitWithdrawOrder        │
  │                       │←──── 订单号 ─────────│             │
  │← 提现进行中 ──────────│             │                       │
  │                                    │                       │
  │           B端管理员操作              │                       │
  │                         风控审核 ──→│                       │
  │                                    │ 通过 → 财务审核        │
  │                                    │ 通过 → 出款            │
  │                                    │── channel_dispatcher──→│ 创建代付订单
  │                                    │←────── 出款结果 ──────│
  │                                    │                       │
  │                      │←──RPC── ConfirmDebit (成功)         │
  │                      │  （扣除冻结，正式减少余额）           │
  │                   或者：                                    │
  │                      │←──RPC── UnfreezeBalance (驳回/失败)  │
  │                      │  （解冻退回，恢复可用余额）           │
```

### 12.4 钱包侧需要提前准备的

为了顺畅联调，钱包侧需要在开发阶段就做好以下准备：

| 准备项 | 说明 | 在钱包哪个文件 |
|--------|------|-------------|
| 注册 FinanceClient | 在 rpc_client.go 添加 FinanceClient() 工厂方法 | common/rpc/rpc_client.go |
| 定义回调契约 | 与财务确认 CreditWallet 等 RPC 的 Req 字段（哪些参数财务会传） | wallet_rpc.thrift |
| Mock 财务接口 | 开发期间 Mock GetPaymentConfig / CreatePayOrder 的返回值 | deposit_service.go / withdraw_service.go |
| 幂等设计 | CreditWallet 等接口必须幂等（财务可能重试回调） | wallet_core.go |
| 订单号关联 | 钱包本地订单号和财务订单号的双向关联存储 | deposit_rep.go / withdraw_rep.go |

---

## 十三、与 ser-wallet 的结构对比

| 维度 | ser-wallet | ser-finance | 差异原因 |
|------|-----------|------------|---------|
| **模块定位** | 管"账"（余额+流水） | 管"通道"（支付进出） | 资金体系分工 |
| **IDL 文件数** | 8 | 9 | 财务多一个 callback.thrift |
| **IDL 方法数** | ~30 | ~43 | 财务 B 端管理功能多 |
| **数据表数** | 14 | 14 | 规模相近 |
| **ser/ 文件数** | 7 | 8 | 财务多 polling_engine + channel_dispatcher |
| **rep/ 文件数** | 7 | 7 | 相同 |
| **特有核心层** | wallet_core.go（账务引擎） | polling_engine + channel_dispatcher（支付引擎） | 各自的核心能力 |
| **B端路由数** | 7 | ~38 | 财务 B 端管理远多于钱包 |
| **C端路由数** | 13 | 0（回调 2-3） | 钱包面向用户，财务面向通道 |
| **RPC 对外** | 10（供财务/投注/直播调用） | 3（供钱包调用） | 钱包是被调方，财务是被钱包调的方 |
| **外部通道对接** | 无 | 有（三方支付通道） | 钱包不直接和三方交互 |
| **双人审核** | 无 | 有（人工修正三类） | 财务的合规要求 |
| **导出功能** | 无 | 有（4处导出+2FA验证） | 财务报表需求 |
| **敏感信息加密** | 无（不存通道密钥） | 有（通道配置含密钥/私钥） | 通道对接安全需求 |

```
规模对比柱状图：

         ser-wallet    ser-finance
         ──────────    ───────────
IDL方法   ████████       █████████████
B端路由   ██             ████████████████████
C端路由   ████████       ░  (回调不算C端)
RPC对外   ██████         ██
手写文件   ████████████   ██████████████████
```

---

## 十四、需求子模块 → 文件映射表

将财务管理的 8 个子模块映射到具体文件，建立全局索引。

| 需求子模块 | gate-back handler | ser/ service | rep/ | 核心数据表 |
|-----------|-------------------|-------------|------|-----------|
| 2.1 通道配置 | channel.go | channel_service.go | channel_rep.go | deposit_channel, payout_channel, polling_rule |
| 2.2 支付配置 | payment.go | payment_service.go | payment_rep.go | deposit_method_config, withdraw_method_config |
| 2.3 充值记录 | deposit.go | deposit_service.go | deposit_rep.go | finance_deposit_order |
| 2.4 提现记录 | withdraw.go | withdraw_service.go | withdraw_rep.go | finance_withdraw_order |
| 2.5 人工修正 | adjustment.go | adjustment_service.go | adjustment_rep.go | supplement_order, manual_credit_order, manual_debit_order |
| 3.1 通道统计 | statistics.go | statistics_service.go | statistics_rep.go | channel_daily_stat |
| 3.2 充值统计 | statistics.go | statistics_service.go | statistics_rep.go | deposit_daily_stat |
| 3.3 提现统计 | statistics.go | statistics_service.go | statistics_rep.go | withdraw_daily_stat |

---

## 十五、财务模块的开发阶段推演

从钱包协作视角，推演财务模块的开发顺序和联调时间窗口。

```
阶段 1：基础设施（与钱包并行，各自独立）
  ├─ ① IDL 定义 + gen_kitex.sh + 公共注册
  ├─ ② 建表 + gorm_gen
  ├─ ③ handler.go + main.go + cfg/ + enum/ + errs/
  └─ ④ 服务可启动、可注册到 ETCD
  → 🔗 此阶段与钱包无交互

阶段 2：配置管理（钱包可 Mock 联调）
  ├─ ⑤ channel_service.go — 通道 CRUD（含敏感信息加密）
  ├─ ⑥ payment_service.go — 充值/提现方式 CRUD
  ├─ ⑦ polling_engine.go — 轮询算法实现
  └─ ⑧ gate-back 路由注册（通道+支付配置）
  → 🔗 ⑥完成后，钱包可联调 GetPaymentConfig（替换 Mock）

阶段 3：充值链路（钱包核心联调窗口）
  ├─ ⑨ channel_dispatcher.go — 三方通道适配（至少接入 1 个通道）
  ├─ ⑩ deposit_service.go — 充值订单创建 + 回调处理
  ├─ ⑪ gate-font callback 路由
  └─ ⑫ 充值回调 → CreditWallet RPC 调用
  → 🔗 钱包联调重点：CreatePayOrder + CreditWallet 回调

阶段 4：提现链路（钱包第二轮联调）
  ├─ ⑬ withdraw_service.go — 提现订单 + 审核流程 + 出款
  └─ ⑭ 审核/出款后 → ConfirmDebit / UnfreezeBalance RPC 调用
  → 🔗 钱包联调重点：SubmitWithdrawOrder + ConfirmDebit/UnfreezeBalance

阶段 5：人工修正 + 统计（钱包最后联调）
  ├─ ⑮ adjustment_service.go — 补单/加款/减款 + 双人审核
  ├─ ⑯ statistics_service.go — 三种统计 + 导出
  └─ ⑰ 审核通过 → SupplementCredit / ManualCredit / ManualDebit RPC 调用
  → 🔗 钱包联调重点：三种人工修正的 RPC 回调
```

### 联调时间线对齐

```
          钱包开发进度          财务开发进度           联调窗口
          ────────────          ────────────           ────────
Week 1-2  基础设施+核心RPC      基础设施+配置管理
Week 3    充值前端流程(Mock)     通道接入+充值链路      ← GetPaymentConfig 可联调
Week 4    提现前端流程(Mock)     提现链路+审核          ← CreatePayOrder + 回调
Week 5    联调充值全链路         联调充值全链路          ← 充值端到端验证
Week 6    联调提现全链路         联调提现全链路          ← 提现端到端验证
Week 7    人工修正联调           人工修正+统计          ← 补单/加减款验证
```

---

## 十六、总结

### ser-finance 结构要点

1. **8 个子模块**（通道配置 / 支付配置 / 充值记录 / 提现记录 / 人工修正 / 通道统计 / 充值统计 / 提现统计）覆盖资金体系"通道层"的全部功能
2. **2 个特有层**（polling_engine 通道轮询 + channel_dispatcher 通道适配）是区别于其他服务的核心，类似 ser-wallet 的 wallet_core
3. **纯 B 端 + RPC 模块**：不直接服务 C 端用户，C 端通过 ser-wallet 间接使用；gate-font 只有三方回调入口
4. **重 B 端管理**：38 个 B 端路由（vs 钱包的 7 个），管理界面复杂度高
5. **三方通道对接**：整个系统中唯一直接和外部支付通道交互的模块

### 对钱包开发者的核心启示

```
你需要记住的 3 个文件名：
  1. finance/payment_service.go  ← 你调它拿充值/提现方式
  2. finance/deposit_service.go  ← 你调它创建充值订单 + 它回调你 CreditWallet
  3. finance/withdraw_service.go ← 你调它提交提现 + 它回调你 Confirm/Unfreeze

你需要记住的 1 个 IDL 文件：
  finance_rpc.thrift  ← 钱包调财务的 3 个 RPC 接口契约全在这里

你需要最早确认的 1 件事：
  finance_rpc.thrift 和 wallet_rpc.thrift 的字段定义需要双方提前对齐
  → IDL 是接口契约，越早确定越好，改 IDL 意味着重新生成代码
```
