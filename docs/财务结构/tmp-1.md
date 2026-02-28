# ser-finance 工程目录结构设计（联调视角）

> 设计时间：2026-02-27
> 定位：站在 ser-wallet 开发者的视角，推演同事的 ser-finance 模块「应该长什么样」
> 设计目的：了熟于心对方的工程结构，才能高效联调、提前预判接口格式、对齐开发节奏
> 设计依据：
> - 需求理解文档 §五（财务管理模块：8大功能全景 + 51张需求/原型图分析）
> - 表结构设计文档 §十二（db_finance 5张应该级表）
> - 评审反馈文档 §二（5个必须拍板的架构问题，含充值调用方向、通知机制）
> - IDL设计文档 §十一（wallet_rpc.thrift — finance调wallet的9个RPC接口）
> - 现有工程代码级扫描（ser-app/ser-item 的目录模式严格遵循）
> 重要说明：此文档是**推演**，不是定稿。实际结构由 finance 负责人决定。
> 但核心接口契约（finance调wallet的9个RPC）是双方共同约定，不会因结构变化而改变。

---

## 目录

1. [为什么我需要了解 ser-finance 的结构](#一为什么我需要了解)
2. [30秒速览：一张图看全貌](#二30秒速览)
3. [根目录文件详解](#三根目录文件详解)
4. [internal/cfg/ — 基础设施初始化层](#四internalcfg)
5. [internal/errs/ — 错误码定义层](#五internalerrs)
6. [internal/enum/ — 业务枚举常量层](#六internalenum)
7. [internal/ser/ — 服务逻辑层（核心）](#七internalser)
8. [internal/rep/ — 数据仓库层](#八internalrep)
9. [internal/cache/ — 缓存操作层](#九internalcache)
10. [internal/domain/ — 领域逻辑层（通道路由）](#十internaldomain)
11. [internal/utils/ — 工具函数层](#十一internalutils)
12. [internal/gen/ + gorm_gen/ — 代码生成](#十二代码生成)
13. [工程外部触点：IDL + 网关 + common](#十三工程外部触点)
14. [finance 与 wallet 的接口契约全景](#十四接口契约全景)
15. [联调时序：谁先谁后、怎么对接](#十五联调时序)
16. [与 ser-wallet 的结构对比](#十六与-ser-wallet-的结构对比)

---

## 一、为什么我需要了解

### 1.1 钱和钱的关系

```
币种配置 = 计价引擎（告诉系统"钱怎么算"）           ← 我负责
钱包     = 账户引擎（管理"用户有多少钱、钱怎么动"）   ← 我负责
财务     = 交易引擎（驱动"钱从外部进出系统"）         ← 同事负责
```

三者构成一个支付账户体系的完整闭环。wallet负责"内部"（余额增减冻结），finance负责"外部"（三方通道进出钱），两者通过9个RPC + MQ消息紧密联动。

### 1.2 不了解对方会导致什么问题

| 场景 | 不了解的后果 |
|------|-------------|
| 充值入账联调 | 不知道 finance 什么时候、以什么格式调 `WalletCredit`，mock数据写不对 |
| 提现审核联调 | 不知道 finance 审核通过后是调 `WalletDeduct` 还是 `WalletUnfreeze`，时序搞错 |
| 支付配置查询 | 不知道 finance 的配置接口叫什么名、返回什么结构，wallet 的 `GetDepositConfig` 就不知道数据从哪来 |
| 回调处理 | 不知道三方回调是打到 finance 还是 wallet，回调handler写错位置 |
| 人工修正 | 不知道 finance 的修正单审核流程，wallet 的 `WalletManualAdd/Sub` 的调用时机理解错误 |

### 1.3 我需要了解到什么程度

```
✅ 清楚 finance 有哪些Service、每个Service做什么（知道对方"有什么能力"）
✅ 清楚 finance 的表结构和数据归属（知道"数据在哪里"）
✅ 清楚 finance 调 wallet 的9个RPC的触发时机（知道"什么时候会调我"）
✅ 清楚 wallet 调 finance 的接口有哪些（知道"我什么时候要调他"）
✅ 清楚 finance 处理三方回调的链路（知道"钱怎么从外面进来"）
❌ 不需要知道 finance 内部每个方法的具体实现细节
❌ 不需要知道通道路由算法的评分公式
❌ 不需要知道 finance 统计报表的SQL怎么写
```

---

## 二、30秒速览

### 2.1 完整目录树

```
ser-finance/
│
├── main.go                                     # 服务入口：初始化 → 依赖注入 → 启动Kitex
├── handler.go                                  # RPC委托层：~28个方法 → 7个Service
├── go.mod                                      # Go模块定义
├── .gitignore                                  # Git忽略规则
├── .golangci.yml                               # 代码检查配置
│
├── internal/
│   │
│   ├── cfg/                                    #【基础设施初始化】
│   │   ├── mysql.go                            #   TiDB(db_finance)连接 + GORM Gen全局Q
│   │   └── redis.go                            #   Redis连接
│   │
│   ├── errs/                                   #【错误码】
│   │   └── code.go                             #   6023000起，按模块分段
│   │
│   ├── enum/                                   #【业务枚举】
│   │   ├── channel.go                          #   通道类型、通道状态、支付类型
│   │   ├── payment.go                          #   充值方式、提现方式
│   │   ├── correction.go                       #   修正类型、修正审核状态
│   │   ├── review.go                           #   提现审核阶段、审核动作
│   │   └── enum_registry.go                    #   枚举下拉选项注册表
│   │
│   ├── ser/                                    #【服务逻辑层 — 核心业务】
│   │   ├── channel_service.go                  #   通道配置：代收/代付通道CRUD
│   │   ├── payment_config_service.go           #   支付配置：充值/提现方式配置
│   │   ├── deposit_record_service.go           #   充值记录：B端查看管理
│   │   ├── withdraw_review_service.go          #   提现记录+审核：风控→财务双审核
│   │   ├── correction_service.go               #   人工修正：补单/加款/减款+审核
│   │   ├── statistics_service.go               #   数据统计：三种报表+CSV导出
│   │   └── payment_service.go                  #   支付核心：创建支付+回调处理+通道路由
│   │
│   ├── rep/                                    #【数据仓库层 — DB操作】
│   │   ├── collect_channel.go                  #   t_collect_channel 查询
│   │   ├── payout_channel.go                   #   t_payout_channel 查询
│   │   ├── deposit_method_config.go            #   t_deposit_method_config 查询
│   │   ├── withdraw_method_config.go           #   t_withdraw_method_config 查询
│   │   └── correction_order.go                 #   t_correction_order 查询
│   │
│   ├── cache/                                  #【缓存操作层 — Redis】
│   │   ├── channel.go                          #   通道配置缓存（轮询路由用）
│   │   └── payment_config.go                   #   充值/提现方式配置缓存
│   │
│   ├── domain/                                 #【领域逻辑层 — 复杂算法抽取】
│   │   ├── channel_router.go                   #   通道智能路由：三维评分轮询算法
│   │   └── callback_handler.go                 #   三方回调统一处理：验签+状态映射
│   │
│   ├── utils/                                  #【工具函数】
│   │   └── decimal.go                          #   shopspring/decimal 金额运算（同wallet）
│   │
│   ├── gen/                                    #【代码生成入口】
│   │   └── gorm_gen.go                         #   GORM Gen代码生成
│   │
│   └── gorm_gen/                               #【自动生成 — .gitignore忽略】
│       ├── model/                              #   5个表 → 5个模型文件
│       │   ├── collect_channel.gen.go          #     t_collect_channel
│       │   ├── payout_channel.gen.go           #     t_payout_channel
│       │   ├── deposit_method_config.gen.go    #     t_deposit_method_config
│       │   ├── withdraw_method_config.gen.go   #     t_withdraw_method_config
│       │   └── correction_order.gen.go         #     t_correction_order
│       ├── query/                              #   1个总入口 + 5个查询构造器
│       │   ├── gen.go
│       │   ├── collect_channel.gen.go
│       │   ├── payout_channel.gen.go
│       │   ├── deposit_method_config.gen.go
│       │   ├── withdraw_method_config.gen.go
│       │   └── correction_order.gen.go
│       └── repo/                               #   5个通用CRUD包
│           ├── collectchannelr/collectchannelr.go
│           ├── payoutchannelr/payoutchannelr.go
│           ├── depositmethodconfigr/depositmethodconfigr.go
│           ├── withdrawmethodconfigr/withdrawmethodconfigr.go
│           └── correctionorderr/correctionorderr.go
│
└── tmp/                                        # 产品文档（参考用，非代码）
```

### 2.2 数量统计

| 指标 | 数量 | 与 ser-wallet 对比 |
|------|------|-------------------|
| 手写Go文件 | **22个** | wallet: 25个 |
| 自动生成Go文件 | ~16个 | wallet: ~28个（表少所以少） |
| 服务层Service | 7个 | wallet: 6个 |
| 数据库表 | 5张 (db_finance) | wallet: 9张 (db_wallet) |
| IDL方法（预估） | ~28个 | wallet: 30个 |

### 2.3 关键差异点

```
ser-wallet:  表多(9张)、面向C端用户、资金内部操作（余额增减冻结）
ser-finance: 表少(5张)、面向B端运营、资金外部操作（三方通道进出钱）+ 读wallet表数据(RPC)

ser-wallet 的核心复杂度 = 并发安全（乐观锁+幂等）
ser-finance 的核心复杂度 = 三方对接（多通道路由+回调处理+审核流程）
```

---

## 三、根目录文件详解

### 3.1 目录树

```
ser-finance/
├── main.go
├── handler.go
├── go.mod
├── .gitignore
└── .golangci.yml
```

### 3.2 逐文件说明

| 文件 | 职责 | 与 ser-wallet 的区别 |
|------|------|---------------------|
| `main.go` | 初始化日志→ETCD→Redis→创建7个Service→启动Kitex | 结构相同，服务名/端口/DB名不同 |
| `handler.go` | 定义 `FinanceServiceImpl`，~28个方法委托到7个Service | 结构相同，方法数和Service名不同 |
| `go.mod` | `module ser-finance`，依赖 common/pkg、common/rpc、shopspring/decimal | 额外依赖：可能需要三方支付SDK |
| `.gitignore` | 标准模板 | 相同 |
| `.golangci.yml` | 工程统一模板 | 相同 |

### 3.3 main.go 依赖注入示意

```
main()
  ├── lg.InitKLog()
  ├── etcd.LoadAndWatch("/slg/")
  ├── cfg.InitRedis()
  ├── flag.String("port", ...)
  │
  ├── 创建7个Service实例：
  │   ├── channelService       = ser.NewChannelService(cache)
  │   ├── paymentConfigService = ser.NewPaymentConfigService(cache)
  │   ├── paymentService       = ser.NewPaymentService(channelService)    ← 依赖通道路由
  │   ├── depositRecordService = ser.NewDepositRecordService()
  │   ├── withdrawReviewService= ser.NewWithdrawReviewService(paymentService)
  │   ├── correctionService    = ser.NewCorrectionService()
  │   └── statisticsService    = ser.NewStatisticsService()
  │
  └── financeservice.NewServer(...).Run()
```

**关键设计点**：`channelService` 被注入到 `paymentService`（通道路由依赖通道配置数据），`paymentService` 被注入到 `withdrawReviewService`（审核通过后发起代付出款需要支付能力）。

### 3.4 handler.go 内部结构示意

```go
type FinanceServiceImpl struct {
    channelService        *ser.ChannelService         // → 通道配置方法
    paymentConfigService  *ser.PaymentConfigService    // → 支付配置方法
    depositRecordService  *ser.DepositRecordService    // → 充值记录方法
    withdrawReviewService *ser.WithdrawReviewService   // → 提现记录+审核方法
    correctionService     *ser.CorrectionService       // → 人工修正方法
    statisticsService     *ser.StatisticsService       // → 统计方法
    paymentService        *ser.PaymentService          // → 支付核心方法
}
```

---

## 四、internal/cfg/

### 4.1 目录树

```
internal/cfg/
├── mysql.go          # TiDB(db_finance) 连接初始化
└── redis.go          # Redis 连接初始化
```

与 ser-wallet 完全相同的模式，唯一区别是数据库名：`namesp.EtcdFinanceDb` → `db_finance`。

---

## 五、internal/errs/

### 5.1 目录树

```
internal/errs/
└── code.go           # 全部错误码定义
```

### 5.2 错误码段位分配（推演）

假设 ser-finance 服务编号为 023，错误码前缀 `6023`：

| 段位 | 范围 | 模块 |
|------|------|------|
| 通道配置 | 6023000 - 6023099 | channel |
| 支付配置 | 6023100 - 6023149 | payment_config |
| 充值记录 | 6023150 - 6023199 | deposit_record |
| 提现审核 | 6023200 - 6023299 | withdraw_review |
| 人工修正 | 6023300 - 6023399 | correction |
| 支付核心 | 6023400 - 6023499 | payment |
| 数据统计 | 6023500 - 6023549 | statistics |

---

## 六、internal/enum/

### 6.1 目录树

```
internal/enum/
├── channel.go          # 通道类型、通道状态、支付类型编码
├── payment.go          # 充值/提现方式枚举
├── correction.go       # 修正类型、修正审核状态
├── review.go           # 提现审核阶段、审核动作
└── enum_registry.go    # B端枚举下拉注册表
```

### 6.2 逐文件枚举清单

| 文件 | 定义的枚举 | 来源 |
|------|-----------|------|
| `channel.go` | ChannelType（1代收/2代付）、ChannelStatus（1启用/2禁用/3维护）、PaymentType（1银行转账/2电子钱包/3USDT） | 需求文档 §5.2 通道配置 |
| `payment.go` | DepositMethodType（同wallet的DepositMethod）、WithdrawMethodType（同wallet的WithdrawMethod） | 需求文档 §5.4 支付配置 |
| `correction.go` | CorrectionType（1补单/2人工加款/3人工减款）、CorrectionStatus（1待审核/2审核通过/3审核驳回）、CorrectionReason（1系统补偿/2活动补发/3财务冲正/4异常追回/5风控处罚/6费用调整/7其他） | 需求文档 §5.7 人工修正 |
| `review.go` | ReviewStage（1风控审核/2财务审核）、ReviewAction（1通过/2驳回）、WithdrawReviewStatus（复用wallet的WithdrawStatus枚举值） | 需求文档 §5.6 提现审核 |

### 6.3 与 wallet 枚举的关系

```
共用的枚举值（两边保持一致）：
├── DepositMethod:    1银行转账 2电子钱包 3USDT
├── WithdrawMethod:   1银行卡 2电子钱包 3USDT
├── WithdrawStatus:   1审核中...8出款失败（8种状态）
├── CurrencyType:     1法币 2加密 3平台
└── WalletType:       1中心 2奖励 3主播 4代理 5场馆

finance独有的枚举值：
├── ChannelType/Status（wallet不关心通道细节）
├── CorrectionType/Status（wallet只关心审核通过后的RPC调用）
└── 通道评分相关常量（路由算法内部用）
```

**联调关键**：`WithdrawStatus` 枚举是双方共享的状态机。wallet创建提现订单时状态=1(审核中)，finance 审核/出款过程推进状态到2→4→6→7/8，过程中finance通过RPC通知wallet执行冻结/解冻/扣款。**两边的状态值必须完全一致**。

---

## 七、internal/ser/ — 服务逻辑层（核心）

### 7.1 目录树

```
internal/ser/
├── channel_service.go              # 通道配置
├── payment_config_service.go       # 支付配置
├── deposit_record_service.go       # 充值记录管理
├── withdraw_review_service.go      # 提现记录+双审核
├── correction_service.go           # 人工修正（补单/加款/减款）
├── statistics_service.go           # 数据统计
└── payment_service.go              # 支付核心（三方对接）
```

### 7.2 逐文件详解

#### channel_service.go — 通道配置服务

| 属性 | 说明 |
|------|------|
| **结构体** | `ChannelService` |
| **依赖** | `*rep.CollectChannelRepo` + `*rep.PayoutChannelRepo` + `*cache.ChannelCache` |
| **功能来源** | 需求文档 §5.2 通道配置 + §5.3 通道轮询规则 |
| **与wallet关系** | **无直接关系**。wallet不关心通道细节，只关心最终支付URL |

```
channel_service.go 方法清单：
├── B端方法：
│   ├── PageCollectChannel()          → 代收通道分页列表
│   ├── UpdateCollectChannel()        → 编辑代收通道（含技术配置：密钥等）
│   ├── ToggleCollectChannelStatus()  → 启禁用代收通道
│   ├── PagePayoutChannel()           → 代付通道分页列表
│   ├── UpdatePayoutChannel()         → 编辑代付通道
│   └── TogglePayoutChannelStatus()   → 启禁用代付通道
│
└── 内部方法（供 PaymentService 调用）：
    ├── GetAvailableCollectChannels()  → 获取某币种+某支付类型的可用代收通道列表
    └── GetAvailablePayoutChannels()   → 获取某币种的可用代付通道列表
```

**需求原文要点**：通道不可新建（由开发导入数据），后台只能编辑配置。每个通道有技术配置（请求地址、商户ID、密钥、AppKey、支付编码、公钥、私钥）。

#### payment_config_service.go — 支付配置服务

| 属性 | 说明 |
|------|------|
| **结构体** | `PaymentConfigService` |
| **依赖** | `*rep.DepositMethodConfigRepo` + `*rep.WithdrawMethodConfigRepo` + `*cache.PaymentConfigCache` |
| **功能来源** | 需求文档 §5.4 支付配置 |
| **与wallet关系** | **高度相关** — wallet的 `GetDepositConfig`/`GetWithdrawConfig` 需要读取这里的配置数据 |

```
payment_config_service.go 方法清单：
├── B端方法：
│   ├── PageDepositMethodConfig()         → 充值方式配置列表
│   ├── UpdateDepositMethodConfig()       → 编辑充值方式（限额/排序/状态）
│   ├── ToggleDepositMethodStatus()       → 启禁用某币种的某充值方式
│   ├── PageWithdrawMethodConfig()        → 提现方式配置列表
│   ├── UpdateWithdrawMethodConfig()      → 编辑提现方式（限额/手续费/到账时间）
│   └── ToggleWithdrawMethodStatus()      → 启禁用某币种的某提现方式
│
└── RPC方法（供 wallet 调用）：
    ├── GetDepositMethods()   → 返回某币种可用充值方式列表 + 限额 + 排序
    └── GetWithdrawMethods()  → 返回某币种可用提现方式列表 + 限额 + 手续费 + 到账时间
```

**联调重点**：
- wallet 的 `GetDepositConfig` 内部会 RPC 调 finance 的 `GetDepositMethods` 获取充值方式
- wallet 的 `GetWithdrawConfig` 内部会 RPC 调 finance 的 `GetWithdrawMethods` 获取提现方式
- **这两个RPC的入参出参格式是联调必须提前对齐的**

#### payment_service.go — 支付核心服务（最重要的文件）

| 属性 | 说明 |
|------|------|
| **结构体** | `PaymentService` |
| **依赖** | `*ChannelService`（通道路由）+ `*domain.ChannelRouter`（评分算法）+ `*domain.CallbackHandler`（回调处理） |
| **功能来源** | 需求文档 §5.2~5.4（通道选择+支付创建）+ §充值/提现流程中的三方对接部分 |
| **与wallet关系** | **最紧密** — 充值回调后调wallet入账，代付回调后调wallet扣款/解冻 |

```
payment_service.go 方法清单：
├── RPC方法（供 wallet 调用）：
│   └── CreatePayment()           → wallet充值下单时调用：选通道→创建三方订单→返回支付URL
│
├── 回调处理方法：
│   ├── HandleDepositCallback()   → 三方代收回调：验签→更新订单→RPC调wallet.WalletCredit
│   └── HandlePayoutCallback()    → 三方代付回调：验签→更新订单→RPC调wallet.WalletDeduct/Unfreeze
│
└── 内部方法：
    ├── selectCollectChannel()    → 调 domain.ChannelRouter 选最优代收通道
    ├── selectPayoutChannel()     → 调 domain.ChannelRouter 选最优代付通道
    ├── callThirdPartyCreate()    → 调三方支付API创建订单（HTTP请求）
    └── callThirdPartyPayout()    → 调三方代付API发起出款（HTTP请求）
```

**这是整个系统"钱从外部进出"的核心枢纽**：

```
充值链路：
  wallet.CreateDepositOrder → finance.CreatePayment → 三方支付
  三方回调 → finance.HandleDepositCallback → wallet.WalletCredit(入账)

提现链路：
  wallet.CreateWithdrawOrder → wallet.WalletFreeze → MQ通知finance
  finance审核通过 → finance.callThirdPartyPayout → 三方代付
  三方回调 → finance.HandlePayoutCallback → wallet.WalletDeduct(扣款)
                                     或 → wallet.WalletUnfreeze(出款失败归还)
```

#### deposit_record_service.go — 充值记录管理服务

| 属性 | 说明 |
|------|------|
| **结构体** | `DepositRecordService` |
| **依赖** | 无本地表依赖（充值订单在 db_wallet），通过 RPC 调 wallet 查询 |
| **功能来源** | 需求文档 §5.5 充值记录管理 |
| **与wallet关系** | 需要RPC调 wallet 查询充值订单数据 |

```
deposit_record_service.go 方法清单：
├── PageDepositRecord()     → B端充值记录分页（RPC调wallet查订单 或 直读wallet提供的查询接口）
└── DetailDepositRecord()   → B端充值记录详情
```

**设计选择点**：充值订单数据在 db_wallet，finance 的 B端页面要展示充值记录。有两种方案：
- **方案A**：finance RPC调 wallet 的查询接口 → 数据库私有，规范清晰
- **方案B**：finance 直连 db_wallet 读取 → 破坏微服务边界，但查询更灵活

评审反馈已建议方案A。但这意味着 wallet 可能需要额外提供 B端充值记录的分页RPC接口（不在当前30个方法中，联调时需确认）。

#### withdraw_review_service.go — 提现记录+审核服务

| 属性 | 说明 |
|------|------|
| **结构体** | `WithdrawReviewService` |
| **依赖** | `*PaymentService`（审核通过后发起出款） |
| **功能来源** | 需求文档 §5.6 提现记录管理和审核 |
| **与wallet关系** | **高度相关** — 审核结果直接触发 wallet 的冻结/解冻/扣款 |

```
withdraw_review_service.go 方法清单：
├── PageWithdrawRecord()            → B端提现记录分页（同样需要RPC调wallet查订单）
├── DetailWithdrawRecord()          → B端提现记录详情
├── ReviewWithdrawRisk()            → 风控审核（通过/驳回）
└── ReviewWithdrawFinance()         → 财务审核（通过/驳回）
```

**双审核状态流转与RPC调用**：

```
提现订单状态机（finance驱动推进）：

  1(审核中) ──风控审核──→ 2(风控通过) 或 3(风控驳回)
                           │                    │
                           │                    └→ RPC: wallet.WalletUnfreeze() ← 解冻归还
                           ▼
                        4(财务通过) 或 5(财务驳回)
                           │                    │
                           │                    └→ RPC: wallet.WalletUnfreeze() ← 解冻归还
                           ▼
                        6(出款中) → finance.callThirdPartyPayout()
                           │
                     三方回调后 ──→ 7(出款成功) 或 8(出款失败)
                                    │                    │
                                    │                    └→ RPC: wallet.WalletUnfreeze()
                                    └→ RPC: wallet.WalletDeduct() ← 最终扣款
```

**联调关键规则**：
- **创建人与审核人不能是同一人**（需求硬规则）
- 风控驳回 → 直接 `WalletUnfreeze`，不再进入财务审核
- 财务通过 → 自动发起代付，不需要人工操作出款
- 出款失败 → `WalletUnfreeze` 归还冻结金额，用户可重新发起提现

#### correction_service.go — 人工修正服务

| 属性 | 说明 |
|------|------|
| **结构体** | `CorrectionService` |
| **依赖** | `*rep.CorrectionOrderRepo` |
| **功能来源** | 需求文档 §5.7 人工修正（补单/加款/减款） |
| **与wallet关系** | 审核通过后RPC调 wallet 的3个接口 |

```
correction_service.go 方法清单：
├── CreateSupplementOrder()       → 创建补单（B+16位订单号）
├── CreateManualAddOrder()        → 创建人工加款单（A+16位）
├── CreateManualSubOrder()        → 创建人工减款单（M+16位）
├── ReviewCorrectionOrder()       → 审核修正单（通过/驳回）
└── PageCorrectionOrder()         → 修正单分页列表
```

**审核通过后的RPC调用映射**：

| 修正类型 | 审核通过后调用 | 说明 |
|---------|-------------|------|
| 补单(B) | `wallet.WalletCredit` + `wallet.WalletCreditReward`（如有奖金） | 充值金额入中心钱包，奖金入奖励钱包 |
| 人工加款(A) | `wallet.WalletManualAdd` | 按指定子钱包分别加款 |
| 人工减款(M) | `wallet.WalletManualSub` | 按指定子钱包分别减款，实际扣款=min(期望, 余额) |

**联调重点**：
- 加款/减款弹窗需要展示用户当前各子钱包余额 → 先 RPC 调 `wallet.GetUserBalance`
- 减款的 `WalletManualSubResp` 会返回实际扣款金额（可能小于期望），finance 需要根据实际金额更新修正单

#### statistics_service.go — 数据统计服务

| 属性 | 说明 |
|------|------|
| **结构体** | `StatisticsService` |
| **依赖** | 可能需要跨库聚合查询，或依赖快照表 |
| **功能来源** | 需求文档 §5.8 数据统计 |
| **与wallet关系** | **间接关系** — 统计数据来源于充值/提现订单（在db_wallet），可能需要RPC获取 |

```
statistics_service.go 方法清单：
├── GetChannelStatistics()       → 通道维度统计（订单数/金额/成功率/手续费/净入账）
├── GetDepositStatistics()       → 充值维度统计（按日期/币种）
├── GetWithdrawStatistics()      → 提现维度统计（按日期/币种）
└── ExportStatisticsCSV()        → CSV导出
```

**时区处理要点**：服务器UTC存储 → B端按站点时区查询 → 前端传时区偏移，后端还原UTC查DB。

### 7.3 Service层依赖关系图

```
                ┌────────────────────┐
                │  ChannelService    │  ← 通道配置数据
                │  (通道数据提供者)    │
                └─────────┬──────────┘
                          │ 注入
                          ▼
                ┌────────────────────┐
                │  PaymentService    │  ← 三方支付核心
                │  (支付+回调+路由)   │
                └────┬──────────┬────┘
                     │          │ 注入
                     │          ▼
                     │  ┌───────────────────────┐
                     │  │WithdrawReviewService   │  ← 审核通过→发起出款
                     │  │(提现审核+出款)          │
                     │  └───────────────────────┘
                     │
    ┌────────────────┤
    │                │
    ▼                ▼
┌───────────┐  ┌─────────────┐   ┌───────────────┐   ┌─────────────┐
│PaymentConf│  │DepositRecord│   │CorrectionServ │   │StatisticsServ│
│  igService│  │  Service    │   │  ice          │   │  ice        │
└───────────┘  └─────────────┘   └───────────────┘   └─────────────┘
      ↑              ↑                   ↑                   ↑
   独立模块        独立模块            独立模块             独立模块


         ====== 跨服务RPC调用方向 ======

   finance ──→ wallet.WalletCredit/Freeze/Unfreeze/Deduct/ManualAdd/Sub/GetUserBalance
   wallet  ──→ finance.GetDepositMethods/GetWithdrawMethods/CreatePayment
```

---

## 八、internal/rep/

### 8.1 目录树

```
internal/rep/
├── collect_channel.go          # t_collect_channel
├── payout_channel.go           # t_payout_channel
├── deposit_method_config.go    # t_deposit_method_config
├── withdraw_method_config.go   # t_withdraw_method_config
└── correction_order.go         # t_correction_order
```

### 8.2 逐文件说明

| 文件 | 对应表 | 核心操作 | 特殊机制 |
|------|--------|---------|---------|
| `collect_channel.go` | t_collect_channel | 分页、按币种+支付类型筛选、更新配置、更新成功率 | 成功率统计字段需原子更新 |
| `payout_channel.go` | t_payout_channel | 分页、按币种筛选、更新配置、更新成功率 | 同上 |
| `deposit_method_config.go` | t_deposit_method_config | 按币种查、更新限额/状态 | 无复杂机制 |
| `withdraw_method_config.go` | t_withdraw_method_config | 按币种查、更新限额/费率/状态 | 无复杂机制 |
| `correction_order.go` | t_correction_order | 创建修正单、审核状态更新、分页 | 创建人≠审核人的DB级校验 |

### 8.3 finance 的 rep 与 wallet 的 rep 的关键区别

```
wallet 的 rep：
├── 9张表，wallet直接读写
├── 核心机制：CAS乐观锁(wallet_account) + 幂等唯一索引(wallet_flow)
└── 事务复杂度高（一个操作同时写2-3张表）

finance 的 rep：
├── 5张表，finance直接读写
├── 核心机制：相对简单的CRUD（配置表的增删改查）
├── correction_order 有审核流转但没有并发竞争问题
└── 需要跨服务读取数据时走RPC，不直连db_wallet
```

**重要**：finance 需要读取的充值订单、提现订单、用户余额等数据，**全部通过 RPC 调 wallet 获取**，不直接查 db_wallet。这是微服务数据库私有原则。

---

## 九、internal/cache/

### 9.1 目录树

```
internal/cache/
├── channel.go              # 通道配置缓存
└── payment_config.go       # 充值/提现方式配置缓存
```

### 9.2 文件说明

| 文件 | 缓存什么 | 为什么缓存 |
|------|---------|-----------|
| `channel.go` | 可用通道列表 + 通道评分数据 | 通道路由算法每次充值/出款都要计算，不能每次都查DB |
| `payment_config.go` | 各币种的充值方式列表、提现方式列表 | wallet 高频查询（每次用户打开充值/提现页面），缓存减少RPC→DB的链路 |

**与 wallet 缓存的联动**：wallet 缓存了从 finance 获取的支付配置（TTL 5分钟）。如果 finance 的 B端修改了配置，需要考虑 wallet 侧缓存过期的问题。建议方案：finance 修改配置后 RPC 通知 wallet 清缓存，或 wallet 用短TTL自然过期。

---

## 十、internal/domain/

### 10.1 目录树

```
internal/domain/
├── channel_router.go          # 通道智能路由算法
└── callback_handler.go        # 三方回调统一处理
```

### 10.2 为什么 finance 需要 domain/ 层

ser-wallet 不需要 domain/（业务逻辑在 ser/ 层足够清晰），但 finance 有两块逻辑复杂到值得抽取：

**① 通道路由算法**（需求文档 §5.3）：

```
三维评分模型：
  总分 = 成功率系数 × W1 + 时间系数 × W2 + 预警系数 × W3

  成功率系数 = 通道近N小时成功率
  时间系数   = 距上次使用的时间（越久分越高，负载均衡）
  预警系数   = 通道余额/预警金额比值（越低越危险）

  降级规则：成功率 < 阈值 → 自动移出候选
  兜底规则：所有通道不可用 → B端强制指定
```

这个算法涉及多个字段的加权计算、排序、降级判断、兜底逻辑，放在 service 层会让 `PaymentService` 膨胀。抽到 domain 层单独管理。

**② 三方回调处理**：

```
回调处理流程（每家三方支付的格式不同）：
  1. 验签（不同通道用不同密钥和算法）
  2. 解析回调体（字段名、格式各异）
  3. 提取标准字段（订单号、金额、状态、时间）
  4. 映射为内部状态（三方的"成功"→平台的DepositStatus.完成）
```

回调处理本质上是"适配器模式"——把N家三方的不同格式统一适配为内部格式。抽到 domain 层用策略模式/接口隔离，service 层只需调统一入口。

### 10.3 与 wallet 联调的相关性

wallet 开发者不需要关心通道路由和回调验签的细节。但需要知道：

| domain文件 | wallet需要知道的 |
|-----------|----------------|
| `channel_router.go` | 不需要关心内部算法，只需知道"finance 会自动选最优通道" |
| `callback_handler.go` | 需要知道回调处理完成后，finance 会调 wallet 的哪些RPC、以什么参数调用 |

---

## 十一、internal/utils/

### 11.1 目录树

```
internal/utils/
└── decimal.go          # shopspring/decimal 金额运算
```

与 wallet 的 `utils/decimal.go` 功能相同。finance 也需要高精度金额运算（手续费计算、汇率换算、统计聚合）。

**建议**：两个服务用同一套 decimal 工具函数。如果后续使用频率高，可以提升到 `common/pkg/utils/decimal.go`。

---

## 十二、代码生成

### 12.1 目录树

```
internal/gen/
└── gorm_gen.go               # 连接 db_finance，扫描5张表

internal/gorm_gen/             # 自动生成（.gitignore忽略）
├── model/    (5个 .gen.go)
├── query/    (1 + 5个 .gen.go)
└── repo/     (5个子目录)
```

与 wallet 的生成机制完全相同，只是表数量从9张变成5张。

---

## 十三、工程外部触点

### 13.1 IDL文件（推演）

```
common/idl/ser-finance/
├── service.thrift               # 主入口：定义 FinanceService（~28个方法）
├── channel.thrift               # 通道配置域 struct
├── payment_config.thrift        # 支付配置域 struct
├── deposit_record.thrift        # 充值记录域 struct
├── withdraw_review.thrift       # 提现记录+审核域 struct
├── correction.thrift            # 人工修正域 struct
├── statistics.thrift            # 统计域 struct
└── finance_rpc.thrift           # 供 wallet 调用的RPC struct
```

| 文件 | struct数量（预估） | 说明 |
|------|-------------------|------|
| service.thrift | 0（纯方法定义） | ~28个方法声明 |
| channel.thrift | ~8 | 代收/代付通道配置相关 |
| payment_config.thrift | ~6 | 充值/提现方式配置相关 |
| deposit_record.thrift | ~4 | B端充值记录查看 |
| withdraw_review.thrift | ~8 | B端提现记录+审核 |
| correction.thrift | ~8 | 三种修正类型+审核 |
| statistics.thrift | ~6 | 三种统计报表 |
| finance_rpc.thrift | ~6 | GetDepositMethods/GetWithdrawMethods/CreatePayment |

### 13.2 finance_rpc.thrift — wallet 需要调用的接口（联调核心）

```thrift
// 以下是 wallet 需要调用 finance 的RPC方法 struct（推演）

// ============ 获取充值方式配置 ============
struct GetDepositMethodsReq {
    1: required string currencyCode       // 币种编码
}
struct DepositMethodItem {
    1: i32 depositMethod                  // 1银行转账 2电子钱包 3USDT
    2: string methodName                  // 方式名称
    3: string minAmount                   // 单笔最低
    4: string maxAmount                   // 单笔最高
    5: i32 sortOrder                      // 显示排序
    6: list<string> quickAmounts          // 快捷金额档位
}
struct GetDepositMethodsResp {
    1: list<DepositMethodItem> methods
}

// ============ 获取提现方式配置 ============
struct GetWithdrawMethodsReq {
    1: required string currencyCode
}
struct WithdrawMethodItem {
    1: i32 withdrawMethod                 // 1银行卡 2电子钱包 3USDT
    2: string methodName
    3: string minAmount
    4: string maxAmount
    5: string dailyLimit
    6: string feeRate                     // 手续费率(%)
    7: string arrivalTime                 // 到账时间描述
}
struct GetWithdrawMethodsResp {
    1: list<WithdrawMethodItem> methods
}

// ============ 创建三方支付订单 ============
struct CreatePaymentReq {
    1: required string orderNo            // 充值订单号（C+16位）
    2: required i64 userId
    3: required string currencyCode
    4: required string amount
    5: required i32 depositMethod         // 充值方式
    6: optional i32 networkType           // USDT网络（TRC20/ERC20/BEP20）
}
struct CreatePaymentResp {
    1: string paymentUrl                  // 三方支付URL（银行/电子钱包跳转用）
    2: string depositAddress              // USDT收款地址（USDT方式用）
    3: string qrCode                      // 二维码内容
    4: string channelOrderNo              // 通道方订单号
    5: i32 expireSeconds                  // 支付有效期
}
```

### 13.3 网关路由（全部B端）

```
gate-back/biz/
├── handler/finance/
│   ├── channel_handler.go              # 通道配置（6个路由）
│   ├── payment_config_handler.go       # 支付配置（6个路由）
│   ├── deposit_record_handler.go       # 充值记录（2个路由）
│   ├── withdraw_review_handler.go      # 提现记录+审核（4个路由）
│   ├── correction_handler.go           # 人工修正（5个路由）
│   └── statistics_handler.go           # 数据统计（4个路由）
├── router/finance/
│   └── finance.go                      # 路由注册
└── router/register.go                  # 加一行 finance.RouterFinance(r)
```

**注意**：finance 纯B端服务（后台管理），gate-font（C端）不需要 finance 的路由。C端的充值/提现/兑换/记录全部通过 ser-wallet 提供。

### 13.4 回调路由（特殊）

三方支付的回调通常是 HTTP POST，不走 Kitex RPC。有两种方案：

```
方案A：回调打到 gate-back → 转发到 finance（现有架构模式内）
方案B：finance 自己起一个 Hertz HTTP Server 直接接收回调（需要额外端口）
```

具体方案需评审确认。如果走方案A，gate-back 需要新增回调路由（不需要鉴权）。

### 13.5 common模块修改点

| 文件 | 修改内容 |
|------|---------|
| `common/pkg/consts/namesp/namesp.go` | 新增 `EtcdFinanceService`、`EtcdFinancePort`、`EtcdFinanceDb` |
| `common/rpc/rpc_client.go` | 新增 `FinanceClient()` 工厂函数 |
| `go.work` | 新增 `./ser-finance` |

### 13.6 ETCD配置项

| Key | 值示例 | 说明 |
|-----|--------|------|
| `/slg/serv/finance/port` | `9101` | ser-finance 监听端口 |
| `/slg/serv/finance/db` | `db_finance` | 数据库名 |

---

## 十四、finance 与 wallet 的接口契约全景

### 14.1 双向调用关系一览

```
┌─────────────────────────────────────────────────────────┐
│                                                          │
│   ser-wallet ────RPC───→ ser-finance                     │
│      │                       │                           │
│      ├── GetDepositMethods() │  "当前VND有哪些充值方式？" │
│      ├── GetWithdrawMethods()│  "当前VND提现限额多少？"   │
│      └── CreatePayment()     │  "帮我创建一笔支付订单"    │
│                              │                           │
│   ser-finance ───RPC───→ ser-wallet                      │
│      │                       │                           │
│      ├── WalletCredit()      │  "充值到账了，给用户入账"  │
│      ├── WalletCreditReward()│  "充值奖金，入奖励钱包"    │
│      ├── WalletFreeze()      │  "提现发起，冻结金额"      │
│      ├── WalletUnfreeze()    │  "驳回/出款失败，解冻归还" │
│      ├── WalletDeduct()      │  "出款成功，最终扣款"      │
│      ├── WalletManualAdd()   │  "加款审核通过，分子钱包加"│
│      ├── WalletManualSub()   │  "减款审核通过，分子钱包减"│
│      ├── GetUserBalance()    │  "查余额（加减款弹窗展示）"│
│      └── GetCurrencyConfig() │  "查币种配置（格式化金额）"│
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 14.2 接口契约详细表

#### wallet → finance（3个RPC）

| 接口 | wallet什么时候调 | 入参核心字段 | 出参核心字段 | 超时建议 |
|------|----------------|-------------|-------------|---------|
| `GetDepositMethods` | 用户打开充值页面 | currencyCode | methods列表(方式+限额+快捷金额) | 1秒 |
| `GetWithdrawMethods` | 用户打开提现页面 | currencyCode | methods列表(方式+限额+费率+到账时间) | 1秒 |
| `CreatePayment` | 用户点击"充值"按钮 | orderNo, userId, currencyCode, amount, depositMethod | paymentUrl, depositAddress, qrCode, expireSeconds | 5秒（涉及三方HTTP调用） |

#### finance → wallet（9个RPC）

| 接口 | finance什么时候调 | 入参核心字段 | 出参核心字段 | 幂等key |
|------|-----------------|-------------|-------------|---------|
| `WalletCredit` | 充值回调成功 / 补单审核通过 | userId, currencyCode, amount, orderNo | success, newBalance | orderNo |
| `WalletCreditReward` | 充值有额外奖金 | userId, currencyCode, amount, orderNo, turnoverMulti | success | orderNo |
| `WalletFreeze` | 提现订单创建后（MQ消费） | userId, currencyCode, walletType, amount, orderNo | success, freezeRecordId | orderNo |
| `WalletUnfreeze` | 审核驳回 / 出款失败 | orderNo | success | orderNo |
| `WalletDeduct` | 代付回调成功 | orderNo | success | orderNo |
| `WalletManualAdd` | 加款审核通过 | userId, currencyCode, orderNo, center/reward/streamerAmount | success | orderNo |
| `WalletManualSub` | 减款审核通过 | userId, currencyCode, orderNo, center/reward/streamerAmount | success, actualXxxAmount | orderNo |
| `GetUserBalance` | 人工加减款弹窗 | userId, currencyCode | subWallets列表, totalAvailable, totalFrozen | - |
| `GetCurrencyConfig` | 金额格式化 | currencyCode | data(配置信息) | - |

### 14.3 异步消息契约

| 消息方向 | 触发场景 | 建议NATS Subject | 消息体核心字段 |
|---------|---------|-----------------|---------------|
| wallet → finance | 提现订单创建成功 | `Nats.wallet.withdraw.created` | orderNo, userId, currencyCode, amount, withdrawMethod |
| finance → wallet | 提现状态变更通知（可选） | `Nats.finance.withdraw.status_changed` | orderNo, newStatus, remark |

**为什么提现用MQ而非RPC**：wallet冻结完成后它的工作已完成，不需要等finance审核结果。MQ解耦让wallet不关心finance的处理速度。

---

## 十五、联调时序：谁先谁后、怎么对接

### 15.1 开发节奏对齐

| 时间线 | ser-wallet（我） | ser-finance（同事） | 联调点 |
|--------|-----------------|-------------------|--------|
| **Phase 1** | 工程脚手架 + IDL + 建表 | 工程脚手架 + IDL + 建表 | 双方同时定义IDL，**对齐RPC接口格式** |
| **Phase 2** | 币种配置模块 | 通道配置 + 支付配置 | finance完成支付配置后，wallet可以RPC读取 |
| **Phase 3** | 余额体系（6种原子操作） | 充值记录 + 人工修正 | wallet的9个RPC就绪后，finance可以开始Mock调用 |
| **Phase 4** | 兑换模块（内部闭环） | 提现审核 + 出款逻辑 | 各自独立开发 |
| **Phase 5** | 充值模块 | 支付核心（CreatePayment + 回调） | **第一次真正联调**：wallet调finance.CreatePayment + finance回调调wallet.WalletCredit |
| **Phase 6** | 提现模块 | 完善代付出款 | **第二次联调**：wallet.WalletFreeze + MQ + finance审核 + finance调wallet.Deduct/Unfreeze |
| **Phase 7** | 记录查询 + 联调收尾 | 人工修正联调 + 统计报表 | finance调wallet.ManualAdd/Sub + GetUserBalance |

### 15.2 联调前的准备清单

| 序号 | 准备事项 | 谁做 | 什么时候完成 |
|------|---------|------|-------------|
| 1 | 双方 IDL 定稿（包含 finance_rpc.thrift + wallet_rpc.thrift） | 双方 | Phase 1 结束前 |
| 2 | 对齐 WithdrawStatus 枚举值（8种状态两边完全一致） | 双方 | Phase 1 |
| 3 | 对齐订单号格式（C/T/B/A/M + 16位） | 双方 | Phase 1 |
| 4 | wallet 的9个RPC可调通（哪怕返回Mock数据） | 我 | Phase 3 结束 |
| 5 | finance 的 GetDepositMethods/GetWithdrawMethods 可调通 | 同事 | Phase 2 结束 |
| 6 | 确定提现通知方式（NATS Subject + 消息格式） | 双方 | Phase 4 结束前 |
| 7 | finance 的 CreatePayment 可调通（至少沙箱环境） | 同事 | Phase 5 开始前 |

### 15.3 联调流程建议

```
第一阶段：接口联通（Phase 3完成后）
├── wallet 启动 → finance 启动
├── finance 调 wallet.GetUserBalance → 确认RPC链路通
├── wallet 调 finance.GetDepositMethods → 确认RPC链路通
└── 目标：双方RPC能调通，参数格式正确

第二阶段：充值全链路（Phase 5）
├── 用户发起充值 → wallet创建订单 → wallet调finance.CreatePayment
├── finance选通道 → 调三方 → 拿到支付URL → 返回wallet → wallet给前端
├── 用户支付 → 三方回调finance → finance调wallet.WalletCredit
├── wallet入账成功 → 余额变化 → 流水记录
└── 目标：一笔充值从头到尾走通

第三阶段：提现全链路（Phase 6）
├── 用户发起提现 → wallet创建订单 → wallet.WalletFreeze → MQ通知finance
├── finance消费MQ → 进入风控审核队列
├── B端风控通过 → 财务通过 → finance发起代付
├── 三方代付回调 → finance调wallet.WalletDeduct → 最终扣款
├── 中间驳回场景：finance调wallet.WalletUnfreeze → 解冻归还
└── 目标：提现正常+驳回两条路径都走通

第四阶段：人工修正（Phase 7）
├── finance 创建补单/加款/减款
├── 加减款弹窗：finance调wallet.GetUserBalance → 展示余额
├── 审核通过 → finance调wallet.WalletManualAdd/Sub
├── 减款特殊：验证返回的actualAmount是否正确处理
└── 目标：三种修正类型全部走通
```

---

## 十六、与 ser-wallet 的结构对比

### 16.1 并排对比

```
ser-wallet/                           ser-finance/
├── main.go                    ✓      ├── main.go
├── handler.go                 ✓      ├── handler.go
├── go.mod                     ✓      ├── go.mod
├── internal/                         ├── internal/
│   ├── cfg/ (mysql+redis)     ✓      │   ├── cfg/ (mysql+redis)
│   ├── errs/ (6022xxx)        ≈      │   ├── errs/ (6023xxx)
│   ├── enum/ (6个文件)         ≈      │   ├── enum/ (5个文件)
│   ├── ser/ (6个Service)      ≈      │   ├── ser/ (7个Service)
│   ├── rep/ (9个Repo)         ≈      │   ├── rep/ (5个Repo)
│   ├── cache/ (2个文件)        ≈      │   ├── cache/ (2个文件)
│   ├── utils/ (decimal.go)    ✓      │   ├── utils/ (decimal.go)
│   ├── gen/                   ✓      │   ├── gen/
│   ├── gorm_gen/ (9表)        ≈      │   ├── gorm_gen/ (5表)
│   └── (无domain/)            ←      │   ├── domain/ (router+callback)  ← 新增
```

### 16.2 差异分析

| 维度 | ser-wallet | ser-finance | 原因 |
|------|-----------|-------------|------|
| 表数量 | 9张 | 5张 | wallet管理用户资金全链路数据，finance管理通道和配置 |
| Service数量 | 6个 | 7个 | finance 多了 payment_service（三方对接核心） |
| 有无 domain/ | 无 | **有** | finance 有通道路由算法和回调适配器，复杂度值得抽取 |
| 面向端 | C端(gate-font) + B端(gate-back) + RPC | **纯B端(gate-back)** + RPC + 回调 | C端操作全部通过wallet暴露 |
| 并发复杂度 | 高（乐观锁+幂等） | 低（配置类CRUD为主） | wallet 操作余额有竞争，finance 操作配置无竞争 |
| 外部依赖 | ser-kyc, ser-blog | **三方支付通道**（HTTP）, ser-wallet(RPC), ser-blog | finance 核心依赖是三方支付，wallet 核心依赖是内部DB |

### 16.3 总结：两个模块的定位

```
┌───────────────────────────────────────────────────────────────┐
│                        支付账户体系                              │
│                                                                │
│  ┌──────────────────┐          ┌──────────────────┐           │
│  │   ser-wallet      │          │   ser-finance     │           │
│  │                   │          │                   │           │
│  │  "内部管家"        │   RPC    │  "外部出纳"        │           │
│  │                   │◄────────►│                   │           │
│  │  管理用户账本      │          │  管理资金进出       │           │
│  │  ・余额增减冻结    │          │  ・三方通道对接     │           │
│  │  ・流水记录        │          │  ・回调处理         │           │
│  │  ・幂等/并发安全   │          │  ・通道智能路由     │           │
│  │  ・币种/汇率配置   │          │  ・审核流程         │           │
│  │                   │          │  ・人工修正         │           │
│  │  db_wallet(9表)   │          │  db_finance(5表)   │           │
│  └──────────────────┘          └──────────────────┘           │
│          ▲                              ▲                      │
│          │                              │                      │
│    C端用户操作                      B端运营管理                   │
│    (充值/提现/兑换)                (通道/审核/统计)               │
└───────────────────────────────────────────────────────────────┘
```

**一句话总结**：wallet 是"内部管家"——管账本、算余额、保安全；finance 是"外部出纳"——对接银行、处理进出、管审核。两者通过9+3=12个RPC接口和1个MQ消息形成完整闭环。清楚对方的结构，联调时才能一步到位。
