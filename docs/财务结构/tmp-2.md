# ser-finance 模块工程结构设计（同事模块，联调参考）

> 编写时间：2026-02-27
> 定位：他方财务管理模块的工程骨架推演 — 供我方联调时「知己知彼」
> 核心目的：清楚同事模块的目录结构、文件职责、接口分布，以便我方在联调时精准对接
> 视角说明：**我不写这个模块的代码**，但必须清楚它"长什么样"，才能写好自己的钱包模块
> 设计依据：
> - 需求来源：ser-wallet/tmp/财务管理产品原型（26 张）+ 财务管理需求文档（25 张）
> - 接口评估：接口评估/tmp-2.md 第六章（财务模块 ~53 个接口）
> - 表结构设计：表结构设计/tmp-2.md（财务侧 Must 4 + Should 2 + Could 1 = 7 张表）
> - 流程思路：流程思路/tmp-2.md 第四~五部分（财务调钱包 RPC 全链路）
> - 工程规范：与 ser-wallet 结构文档保持同一分析框架，遵循 ser-app/ser-user/ser-live 实际工程规范

---

## 一、为什么我需要清楚 ser-finance 的结构

```
ser-wallet（我方）                     ser-finance（同事）
┌──────────────────┐                 ┌──────────────────┐
│  钱包核心          │ ←── 6 个 RPC ──│  充值回调/审核     │
│  (余额/流水/冻结)   │                │  (入账/冻解/扣冻)  │
│                    │                │                    │
│  充值/提现 C端      │ ── 4 个 RPC ──→│  通道/支付配置     │
│  (方式获取/通道匹配) │                │  (方式/通道/轮询)  │
└──────────────────┘                 └──────────────────┘
        双向 10 个 RPC 调用
```

**10 个 RPC 调用贯穿充值/提现/人工修正三条核心链路**，不了解对方的文件在哪、逻辑在哪，联调时就是盲人摸象。

---

## 二、完整工程目录结构

```
ser-finance/
│
├── main.go                          # 服务入口：初始化 + 启动 RPC Server
├── handler.go                       # RPC Handler：~47 个方法的请求分发层
├── go.mod                           # 模块依赖声明
├── go.sum                           # 依赖校验
├── .gitignore
├── .golangci.yml
│
├── internal/
│   │
│   ├── cfg/                         # ── 基础设施初始化 ──
│   │   ├── mysql.go                 # TiDB 连接（namesp.EtcdFinanceDb）
│   │   └── redis.go                 # Redis 连接
│   │
│   ├── errs/                        # ── 错误码定义 ──
│   │   └── code.go                  # 按域分段：channel/payment/deposit/withdraw/adjust/stats
│   │
│   ├── enum/                        # ── 枚举与常量 ──
│   │   ├── channel_type.go          # 通道类型（1代收/2代付）+ 状态
│   │   ├── order_status.go          # 充值订单4状态 + 提现订单7状态 + 人工修正3状态
│   │   ├── adjust_type.go           # 人工修正类型（1补单/2加款/3减款）
│   │   ├── payment_method.go        # 支付方式类型（银行/电子钱包/USDT）
│   │   └── enum_registry.go         # 枚举注册表
│   │
│   ├── ser/                         # ── 服务层（核心业务逻辑）──
│   │   ├── channel.go               # 通道配置：代收/代付通道 CRUD + 轮询规则
│   │   ├── payment.go               # 支付配置：充值方式 + 提现方式 CRUD
│   │   ├── deposit_record.go        # 充值记录：列表 + 详情 + 导出
│   │   ├── withdraw_record.go       # 提现记录：列表 + 详情 + 审核 + 锁定 + 导出
│   │   ├── adjustment.go            # 人工修正：补单 + 加款 + 减款（创建/列表/审核）
│   │   ├── callback.go              # 回调处理：代收回调 + 代付回调（调钱包RPC）
│   │   ├── statistics.go            # 数据统计：通道统计 + 充值统计 + 提现统计
│   │   └── rpc_provider.go          # RPC供他方：4个接口（支付方式/通道匹配/代付/状态查询）
│   │
│   ├── rep/                         # ── 数据访问层 ──
│   │   ├── payment_channel.go       # payment_channel 表（代收+代付通道）
│   │   ├── deposit_method.go        # deposit_method 表（充值方式配置）
│   │   ├── withdraw_method.go       # withdraw_method 表（提现方式配置）
│   │   ├── channel_order.go         # channel_order 表（通道侧订单）
│   │   ├── manual_adjustment.go     # manual_adjustment 表（补单/加款/减款记录）
│   │   ├── polling_rule.go          # polling_rule_config 表（轮询算法配置）
│   │   └── channel_statistics.go    # channel_statistics 表（通道统计汇总）
│   │
│   ├── cache/                       # ── 缓存层 ──
│   │   ├── channel.go               # 通道配置缓存（轮询时读频繁）
│   │   ├── payment.go               # 支付方式缓存（C端查方式时高频读）
│   │   └── callback.go              # 回调幂等缓存（防重复处理）
│   │
│   ├── channel/                     # ── 三方通道适配层（财务模块特有）──
│   │   ├── adapter.go               # 通道适配器接口定义（统一抽象）
│   │   ├── polling.go               # 轮询算法实现（综合得分 = 成功率×时间×预警×权重）
│   │   ├── sign.go                  # 签名/验签工具（各通道密钥管理）
│   │   └── provider/                # 具体通道实现（每接入一个通道加一个文件）
│   │       ├── provider_a.go        # 通道A适配（请求构建+回调解析）
│   │       └── provider_b.go        # 通道B适配
│   │
│   ├── gen/                         # ── 代码生成 ──
│   │   └── gorm_gen.go              # GORM Gen 驱动
│   │
│   ├── gorm_gen/                    # ── 自动生成（禁止手动修改）──
│   │   ├── model/                   # 生成的数据模型
│   │   ├── query/                   # 生成的查询构建器
│   │   └── repo/                    # 生成的基础仓储
│   │
│   └── utils/                       # ── 工具函数 ──
│       └── order_no.go              # 订单号生成（B/A/M + 16位）
│
└── tmp/                             # 产品文档（不参与编译）
```

---

## 三、关联的 IDL 目录结构

```
common/idl/ser-finance/              # 同事负责创建
├── service.thrift                   # 服务入口：FinanceService 定义 ~47 个方法
├── channel.thrift                   # 通道配置域：代收/代付通道 CRUD + 轮询规则
├── payment.thrift                   # 支付配置域：充值方式 + 提现方式 CRUD
├── deposit_record.thrift            # 充值记录域：列表/详情/导出
├── withdraw_record.thrift           # 提现记录域：列表/详情/审核/锁定/导出
├── adjustment.thrift                # 人工修正域：补单/加款/减款
├── statistics.thrift                # 数据统计域：通道/充值/提现统计
├── callback.thrift                  # 回调域：代收/代付回调数据结构
└── finance_rpc.thrift               # RPC供他方域：4个供钱包调用的接口
```

**共 9 个文件（1 service + 8 domain）**

---

## 四、需要修改的 common 仓库文件

```
common/
├── idl/ser-finance/                  # ❌ 新建：9 个 thrift 文件（同事负责）
├── pkg/consts/namesp/namesp.go       # ✏️ 新增：EtcdFinanceService/Port/Db
├── pkg/kitex_gen/ser_finance/        # 🔄 gen_kitex.sh 自动生成
├── rpc/rpc_client.go                 # ✏️ 新增：FinanceClient() 工厂方法
└── ../go.work                        # ✏️ 新增：./ser-finance 条目
```

---

## 五、文件职责分层详解

### 5.1 根目录文件

| 文件 | 职责说明 |
|------|---------|
| **main.go** | 初始化日志→ETCD→MySQL/Redis→创建 8 个 Service 实例→注入 Handler→启动 RPC Server |
| **handler.go** | 实现 FinanceService 接口的 ~47 个方法，纯委托分发到 8 个 Service |
| **go.mod** | `module ser-finance`，go 1.25，依赖 kitex/gorm/etcd 等 |

**handler.go 的 Service 注入结构**：

```
FinanceServiceImpl struct {
    channelSer        *ser.ChannelService          // 通道配置
    paymentSer        *ser.PaymentService          // 支付配置
    depositRecordSer  *ser.DepositRecordService     // 充值记录
    withdrawRecordSer *ser.WithdrawRecordService    // 提现记录
    adjustmentSer     *ser.AdjustmentService        // 人工修正
    callbackSer       *ser.CallbackService          // 回调处理
    statisticsSer     *ser.StatisticsService        // 数据统计
    rpcProviderSer    *ser.RpcProviderService        // RPC供他方
}
```

---

### 5.2 internal/ser/ — 服务层（8 个文件）

#### 文件与接口对应关系

| 文件 | 接口数 | 覆盖功能 | 是否调钱包 RPC |
|------|--------|---------|--------------|
| **channel.go** | ~8 | 代收通道列表/编辑/启禁用 + 代付通道列表/编辑/启禁用 + 轮询规则查询/配置 | ❌ |
| **payment.go** | ~8 | 充值方式列表/新增/编辑/启禁用 + 提现方式列表/新增/编辑/启禁用 | ❌ |
| **deposit_record.go** | ~3 | 充值记录列表（24字段）+ 详情 + 导出 | ❌ |
| **withdraw_record.go** | ~9 | 提现记录列表/详情/导出 + 风控审核 + 财务审核 + 出款确认 + 锁定/解锁/强制解锁 | ✅ UnfreezeBalance |
| **adjustment.go** | ~9 | 补单(列表/创建/审核) + 加款(列表/创建/审核) + 减款(列表/创建/审核) | ✅ CreditWallet / DebitWallet / QueryBalance |
| **callback.go** | 2 | 代收回调处理 + 代付回调处理 | ✅ CreditWallet / DeductFrozen |
| **statistics.go** | ~4 | 代收通道统计 + 代付通道统计 + 充值统计 + 提现统计 | ❌ |
| **rpc_provider.go** | 4 | GetPaymentMethods + MatchAndCreateChannelOrder + InitiateChannelPayout + GetChannelOrderStatus | ❌（被钱包调用） |

**合计**：8+8+3+9+9+2+4+4 = **~47**

#### 核心文件详解

**callback.go — 回调处理（联调最关键的文件）**

这个文件承载了充值和提现链路中「三方通道 → 财务 → 钱包」的关键中转逻辑：

```
callback.go 内部结构：

┌─ CallbackService struct ────────────────────────────┐
│  channelOrderRep  *rep.ChannelOrderRepo             │
│  cache            *cache.CallbackCache              │
│  walletClient     WalletServiceClient（RPC客户端）   │
└─────────────────────────────────────────────────────┘
         │
         ├── HandleDepositCallback(ctx, rawData)
         │   ├── 1. 解析回调数据（根据通道类型适配）
         │   ├── 2. 验签（通道密钥 → sign.Verify）
         │   ├── 3. 查询通道订单 → 找到平台订单号
         │   ├── 4. 幂等检查（同一回调不重复处理）
         │   ├── 5. 支付成功 → 更新通道订单状态
         │   ├── 6. ★ RPC 调钱包 CreditWallet ★
         │   │       参数：userId, currencyCode, walletType=中心
         │   │              amount=订单金额, orderNo=C+16位
         │   │       如有奖金活动：
         │   │              第二次调用 CreditWallet → walletType=奖励
         │   │              + auditMultiplier + auditTurnover
         │   └── 7. 返回成功给三方通道
         │
         └── HandleWithdrawCallback(ctx, rawData)
             ├── 1. 解析回调数据
             ├── 2. 验签
             ├── 3. 查询通道订单 → 找到平台提现订单号
             ├── 4. 幂等检查
             ├── 5. 分支处理：
             │   ├── 出款成功：
             │   │   ├── 更新订单状态 → 7(出款成功)
             │   │   └── ★ RPC 调钱包 DeductFrozenAmount ★
             │   │         参数：userId, currencyCode, walletType=中心
             │   │                amount=提现金额, orderNo=T+16位
             │   └── 出款失败：
             │       ├── 更新订单状态 → 6(出款失败)
             │       └── ★ 不调钱包任何接口 ★（保持冻结，等人工处理）
             └── 6. 返回成功给三方通道
```

**withdraw_record.go — 提现审核（调钱包解冻的文件）**

```
withdraw_record.go 关键方法：

├── RiskAudit(ctx, req)              // 风控审核
│   ├── 通过 → 状态: 待风控 → 待财务
│   └── 驳回 → 状态: 待风控 → 风控驳回(终态)
│       └── ★ RPC 调钱包 UnfreezeBalance ★
│              参数：userId, currencyCode, walletType, amount, orderNo=T+16位
│
├── FinanceAudit(ctx, req)           // 财务审核
│   ├── 通过 → 状态: 待财务 → 待出款
│   │   └── 触发发起代付（调通道）
│   └── 驳回 → 状态: 待财务 → 财务驳回(终态)
│       └── ★ RPC 调钱包 UnfreezeBalance ★
│
└── ConfirmPayout(ctx, req)          // 出款确认（人工确认通道已出款）
    └── 状态: 待出款 → 出款成功
        └── ★ RPC 调钱包 DeductFrozenAmount ★
```

**adjustment.go — 人工修正（调钱包加减款的文件）**

```
adjustment.go 关键方法：

├── CreateSupplementOrder(ctx, req)  // 创建补单
│   └── 查用户余额：RPC QueryWalletBalance → 展示在表单
│
├── AuditSupplementOrder(ctx, req)   // 审核补单
│   └── 通过 → ★ RPC CreditWallet ★（orderNo=B+16, opType=补单）
│
├── CreateAddFundsOrder(ctx, req)    // 创建加款
│   └── 查用户余额：RPC QueryWalletBalance → 展示在表单
│
├── AuditAddFundsOrder(ctx, req)     // 审核加款
│   └── 通过 → 逐个钱包调用：
│       ├── ★ CreditWallet(walletType=中心, amount=X, orderNo=A+16) ★
│       ├── ★ CreditWallet(walletType=奖励, amount=Y, orderNo=A+16) ★
│       ├── ★ CreditWallet(walletType=代理, amount=Z, orderNo=A+16) ★
│       └── ★ CreditWallet(walletType=主播, amount=W, orderNo=A+16) ★
│       注意：同一 orderNo + 不同 walletType = 不同幂等键
│
├── CreateDeductFundsOrder(ctx, req) // 创建减款
│   └── 查用户余额：RPC QueryWalletBalance → 展示在表单 + 校验上限
│
└── AuditDeductFundsOrder(ctx, req)  // 审核减款
    └── 通过 → 逐个钱包调用：
        └── ★ DebitWallet(walletType=X, amount=Y, orderNo=M+16) ★
            注意：减款金额 <= 当前余额，否则钱包 RPC 拒绝
```

**rpc_provider.go — RPC 供钱包调用（联调接口定义在这里）**

```
rpc_provider.go 4 个方法：

├── GetPaymentMethods(ctx, req)
│   │  入参：currencyCode + type("deposit"/"withdraw")
│   │  出参：方式列表（名称/图标/档位/限额/关联通道）
│   └── 数据来源：deposit_method 表 / withdraw_method 表
│
├── MatchAndCreateChannelOrder(ctx, req)
│   │  入参：depositMethodId + currencyCode + amount
│   │  处理：
│   │    1. 查询充值方式关联的所有代收通道
│   │    2. 轮询算法选最优通道（polling.go）
│   │    3. 调用通道 API 创建订单（channel/provider/*.go）
│   │    4. 写入 channel_order 表
│   │  出参：channelOrderNo + paymentInfo（跳转URL/二维码/地址）
│   └── 数据来源：payment_channel 表 + polling_rule 表
│
├── InitiateChannelPayout(ctx, req)
│   │  入参：withdrawOrderNo + withdrawMethodId + amount + 收款账户信息
│   │  处理：
│   │    1. 选择代付通道
│   │    2. 构建代付请求（channel/provider/*.go）
│   │    3. 调用通道 API 发起代付
│   │    4. 写入 channel_order 表
│   │  出参：channelOrderNo + status（processing/success/failed）
│   └── 后续通过回调异步通知结果
│
└── GetChannelOrderStatus(ctx, req)
    │  入参：channelOrderNo
    │  出参：状态 + 更新时间
    └── 数据来源：channel_order 表
```

---

### 5.3 internal/rep/ — 数据访问层（7 个文件）

#### 文件与数据表对应关系

| 文件 | 对应表 | 分级 | 核心操作 |
|------|-------|------|---------|
| **payment_channel.go** | payment_channel | Must | ListByType / GetById / Update / ToggleStatus / SelectForPolling |
| **deposit_method.go** | deposit_method | Must | ListByCurrency / Create / Update / ToggleStatus |
| **withdraw_method.go** | withdraw_method | Must | ListByCurrency / Create / Update / ToggleStatus |
| **channel_order.go** | channel_order | Must | Create / UpdateStatus / GetByChannelOrderNo / GetByPlatformOrderNo |
| **manual_adjustment.go** | manual_adjustment | Should | Create / PageByFilter / UpdateAuditResult |
| **polling_rule.go** | polling_rule_config | Should | Get / Update |
| **channel_statistics.go** | channel_statistics | Could | Upsert / PageByFilter / AggregateByDateRange |

---

### 5.4 internal/channel/ — 三方通道适配层（财务模块特有）

这是 ser-finance **区别于所有其他服务**的特殊目录。其他服务（ser-app/ser-user/ser-wallet）都没有这一层。

| 文件 | 职责 |
|------|------|
| **adapter.go** | 定义通道适配器接口（统一抽象） |
| **polling.go** | 轮询算法实现：综合得分 = 成功率系数 × 时间系数 × 预警系数 × 权重 |
| **sign.go** | 签名/验签工具：每个通道的密钥管理 + 回调验签逻辑 |
| **provider/provider_a.go** | 通道 A 的具体实现：请求构建 + 回调解析 + 错误映射 |
| **provider/provider_b.go** | 通道 B 的具体实现（每接入一个三方通道增加一个文件） |

**适配器接口设计**：

```
ChannelAdapter interface {
    // 代收：创建充值支付订单
    CreatePaymentOrder(ctx, params) → (paymentInfo, error)

    // 代付：发起提现出款
    CreatePayoutOrder(ctx, params) → (payoutResult, error)

    // 回调：解析通道回调数据
    ParseCallback(ctx, rawData) → (callbackResult, error)

    // 验签：校验回调签名
    VerifySign(ctx, rawData, publicKey) → (bool, error)

    // 查询：查询通道订单状态
    QueryOrderStatus(ctx, channelOrderNo) → (status, error)
}
```

**这层的意义**：将三方通道的差异封装在 provider/ 里，ser/ 层的 callback.go 和 rpc_provider.go 只面向统一接口，不关心具体通道细节。后续接入新通道只需新增 provider 文件，不改业务逻辑。

---

### 5.5 internal/cache/ — 缓存层（3 个文件）

| 文件 | 缓存内容 | 为什么需要缓存 |
|------|---------|-------------|
| **channel.go** | 通道配置列表 + 轮询参数 | 每次充值都触发通道匹配，读频极高 |
| **payment.go** | 充值方式列表 + 提现方式列表 | C 端用户打开充值/提现页面即查询，读频高 |
| **callback.go** | 回调幂等键（SetNX 防重复处理） | 三方通道可能重复发送回调 |

---

### 5.6 internal/enum/ — 枚举（5 个文件）

| 文件 | 典型常量 |
|------|---------|
| **channel_type.go** | `ChannelTypeDeposit=1, ChannelTypeWithdraw=2` + `ChannelStatusEnabled=1, Disabled=0` |
| **order_status.go** | 充值：`DepositPending=1, Success=2, Failed=3, Timeout=4`；提现：`WithdrawRiskPending=1, RiskRejected=2, FinancePending=3, FinanceRejected=4, PayoutPending=5, PayoutFailed=6, PayoutSuccess=7` |
| **adjust_type.go** | `AdjustSupplement=1, AdjustAddFunds=2, AdjustDeductFunds=3` + 审核状态 |
| **payment_method.go** | `MethodBankTransfer=1, MethodEWallet=2, MethodUSDT=3` |
| **enum_registry.go** | 注册所有枚举供 GetEnumOptions 查询 |

---

## 六、数据表结构概览

### 6.1 表清单

```
┌── ser-finance 数据表 ───────────────────────────────────────────────────┐
│                                                                        │
│  [Must] payment_channel         支付通道配置表（代收+代付统一）           │
│  [Must] deposit_method          充值方式配置表（币种×方式）              │
│  [Must] withdraw_method         提现方式配置表（币种×方式）              │
│  [Must] channel_order           通道侧订单表（通道订单号+回调状态）       │
│  [Should] manual_adjustment     人工修正记录表（补单/加款/减款）          │
│  [Should] polling_rule_config   轮询规则配置表（算法参数）               │
│  [Could] channel_statistics     通道统计汇总表（预计算报表数据）          │
│                                                                        │
│  合计：Must 4 + Should 2 + Could 1 = 7 张表                            │
└────────────────────────────────────────────────────────────────────────┘
```

### 6.2 核心表字段概要

#### payment_channel（通道配置）

| 字段类别 | 关键字段 | 说明 |
|----------|---------|------|
| 标识 | channel_name, channel_code | 通道名称和唯一编码 |
| 类型 | channel_type(1 代收 / 2 代付) | 区分充值通道和提现通道 |
| 认证 | api_url, merchant_id, api_key, public_key, private_key | 对接三方通道的密钥信息 |
| 费率 | fee_rate, min_amount, max_amount | 通道手续费和限额 |
| 轮询 | weight, success_rate, avg_response_time, warning_amount | 轮询算法参数 |
| 币种 | currency_code | 该通道支持的币种 |
| 状态 | status(0 禁用 / 1 启用) | 通道开关 |

#### channel_order（通道订单）

| 字段类别 | 关键字段 | 说明 |
|----------|---------|------|
| 关联 | platform_order_no, channel_order_no | 平台订单号（C/T+16）↔ 通道订单号 |
| 类型 | order_type(1 代收 / 2 代付) | 充值 or 提现 |
| 金额 | request_amount, actual_amount, fee_amount | 请求金额/实际到账/手续费 |
| 通道 | channel_id, channel_code | 使用的通道 |
| 回调 | callback_raw, callback_time | 原始回调数据和时间 |
| 状态 | status | 处理中/成功/失败 |

#### manual_adjustment（人工修正）

| 字段类别 | 关键字段 | 说明 |
|----------|---------|------|
| 标识 | order_no(B/A/M+16) | 修正订单号 |
| 类型 | adjust_type(1 补单 / 2 加款 / 3 减款) | 修正类型 |
| 关联 | user_id, currency_code, source_order_no | 用户/币种/原订单（补单时） |
| 金额 | center_amt, reward_amt, anchor_amt, agent_amt | 4 个子钱包分别的操作金额 |
| 稽核 | center_audit_mul, reward_audit_mul | 各钱包稽核倍数 |
| 审核 | status(1 待审核 / 2 拒绝 / 3 通过), review_by, review_remark | 审核流程 |

---

## 七、层次调用关系图

```
三方支付通道（代收/代付）
       │ HTTP 回调
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  handler.go — ~47 个方法，纯委托转发                                  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
       ┌──────────┬──────────┼──────────┬──────────┐
       ▼          ▼          ▼          ▼          ▼
┌──────────┐ ┌────────┐ ┌──────────┐ ┌────────┐ ┌──────────────┐
│ channel  │ │payment │ │ callback │ │adjust  │ │ rpc_provider │
│ .go      │ │.go     │ │ .go      │ │ment.go │ │ .go          │
│          │ │        │ │ 回调处理  │ │ 人工修正│ │ 供钱包调用    │
└────┬─────┘ └───┬────┘ └────┬─────┘ └───┬────┘ └──────┬───────┘
     │           │           │           │             │
     │           │      ┌────┘           │             │
     ▼           ▼      ▼               ▼             ▼
┌──────────────────────────────────────────────────────────────┐
│  rep/ — 7 个 Repo 文件                                        │
│  channel_order.go / payment_channel.go / manual_adjustment.go │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼ GORM
┌──────────────────────────────────────────────────────────────┐
│  gorm_gen/ — 自动生成 Model + Query                           │
│  ↕ TiDB (ser-finance 独立数据库)                              │
└──────────────────────────────────────────────────────────────┘

     ║ callback.go / adjustment.go / withdraw_record.go
     ║ 调用钱包 RPC
     ▼
┌──────────────────┐        ┌──────────────┐
│ ser-wallet       │        │ channel/     │
│ (via RPC Client) │        │ adapter.go   │
│ CreditWallet     │        │ polling.go   │
│ DebitWallet      │        │ sign.go      │
│ FreezeBalance    │        │ provider/    │
│ UnfreezeBalance  │        │  ↕ 三方通道   │
│ DeductFrozen     │        └──────────────┘
│ QueryBalance     │
└──────────────────┘
```

---

## 八、功能域 → 文件映射全景图

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                      ser-finance ~53 个接口的文件落地分布                              │
│                                                                                    │
│  ┌─ 通道配置域（B端 ~8 个）─────────────────────────────────────────────────────┐  │
│  │ IDL: channel.thrift    Ser: channel.go    Rep: payment_channel.go           │  │
│  │                                                polling_rule.go             │  │
│  │ 代收通道列表 / 编辑 / 启禁用                                                 │  │
│  │ 代付通道列表 / 编辑 / 启禁用                                                 │  │
│  │ 轮询规则查询 / 配置                                                          │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 支付配置域（B端 ~8 个）─────────────────────────────────────────────────────┐  │
│  │ IDL: payment.thrift    Ser: payment.go    Rep: deposit_method.go            │  │
│  │                                                withdraw_method.go           │  │
│  │ 充值方式列表 / 新增 / 编辑 / 启禁用                                           │  │
│  │ 提现方式列表 / 新增 / 编辑 / 启禁用                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 充值记录域（B端 ~3 个）─────────────────────────────────────────────────────┐  │
│  │ IDL: deposit_record.thrift  Ser: deposit_record.go  Rep: channel_order.go   │  │
│  │                                                     (+ 读钱包 deposit_order)│  │
│  │ 充值记录列表（24字段筛选）/ 详情 / 导出                                       │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 提现记录域（B端 ~9 个）─────────────────────────────────────────────────────┐  │
│  │ IDL: withdraw_record.thrift  Ser: withdraw_record.go                        │  │
│  │                               Rep: channel_order.go(复用)                   │  │
│  │                                                                              │  │
│  │ 提现记录列表 / 详情 / 导出                                                    │  │
│  │ ★ 风控审核（驳回→调钱包 UnfreezeBalance）                                     │  │
│  │ ★ 财务审核（驳回→调钱包 UnfreezeBalance）                                     │  │
│  │ ★ 出款确认（成功→调钱包 DeductFrozenAmount）                                  │  │
│  │ 锁定订单 / 解锁订单 / 强制解锁                                                │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 人工修正域（B端 ~9 个）─────────────────────────────────────────────────────┐  │
│  │ IDL: adjustment.thrift  Ser: adjustment.go  Rep: manual_adjustment.go       │  │
│  │                                                                              │  │
│  │ 补单：列表 / 创建（查余额 QueryBalance）/ ★ 审核（CreditWallet）              │  │
│  │ 加款：列表 / 创建（查余额 QueryBalance）/ ★ 审核（CreditWallet × N个钱包）    │  │
│  │ 减款：列表 / 创建（查余额 QueryBalance）/ ★ 审核（DebitWallet × N个钱包）     │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 回调域（2 个 HTTP 回调）────────────────────────────────────────────────────┐  │
│  │ IDL: callback.thrift    Ser: callback.go    内部依赖: channel/adapter+sign   │  │
│  │                                             Rep: channel_order.go           │  │
│  │                                                                              │  │
│  │ ★ 代收回调 → 验签 → 更新通道订单 → CreditWallet（充值入账）                    │  │
│  │ ★ 代付回调 → 验签 → 更新通道订单 → DeductFrozen（出款成功）/ 不动（失败）       │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 数据统计域（B端 ~4 个）─────────────────────────────────────────────────────┐  │
│  │ IDL: statistics.thrift  Ser: statistics.go  Rep: channel_statistics.go       │  │
│  │                                             (+ 读 channel_order 聚合)       │  │
│  │ 代收通道统计 / 代付通道统计 / 充值统计 / 提现统计                               │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ RPC 供他方域（4 个 RPC 方法）───────────────────────────────────────────────┐  │
│  │ IDL: finance_rpc.thrift  Ser: rpc_provider.go                               │  │
│  │                          内部依赖: channel/polling.go + channel/provider/    │  │
│  │                                                                              │  │
│  │ GetPaymentMethods            ← 钱包 C-2/C-7 调用                            │  │
│  │ MatchAndCreateChannelOrder   ← 钱包 C-3 调用                                │  │
│  │ InitiateChannelPayout        ← 提现出款时调用                                 │  │
│  │ GetChannelOrderStatus        ← 查询通道订单状态                               │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ 调钱包 RPC（6 个，不在财务 IDL 定义，在钱包 IDL 定义）──────────────────────┐  │
│  │ 调用位置分布：                                                                │  │
│  │  callback.go       → CreditWallet / DeductFrozenAmount                      │  │
│  │  withdraw_record.go → UnfreezeBalance / DeductFrozenAmount                  │  │
│  │  adjustment.go     → CreditWallet / DebitWallet / QueryWalletBalance        │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、与 ser-wallet 的 RPC 交互详解

这是我方联调时最需要熟悉的部分：哪个文件在什么时机调我的哪个 RPC。

### 9.1 财务调钱包（6 个 RPC）

| 财务文件 | 调用时机 | 调我方 RPC | 我方处理文件 |
|----------|---------|-----------|------------|
| callback.go | 三方充值回调成功 | **CreditWallet** | wallet.go |
| callback.go | 三方代付回调成功 | **DeductFrozenAmount** | wallet.go |
| withdraw_record.go | 风控/财务驳回提现 | **UnfreezeBalance** | wallet.go |
| withdraw_record.go | 人工确认出款成功 | **DeductFrozenAmount** | wallet.go |
| adjustment.go | 补单/加款审核通过 | **CreditWallet**（可能多次） | wallet.go |
| adjustment.go | 减款审核通过 | **DebitWallet**（可能多次） | wallet.go |
| adjustment.go | 创建补单/加款/减款时 | **QueryWalletBalance** | wallet.go |

### 9.2 钱包调财务（4 个 RPC）

| 我方文件 | 调用时机 | 调财务 RPC | 财务处理文件 |
|----------|---------|-----------|------------|
| deposit.go | 用户打开充值页面 | **GetPaymentMethods** | rpc_provider.go |
| deposit.go | 用户确认充值提交 | **MatchAndCreateChannelOrder** | rpc_provider.go |
| withdraw.go | 用户打开提现页面 | **GetPaymentMethods** | rpc_provider.go |
| withdraw.go | 提现审核通过后出款 | **InitiateChannelPayout** | rpc_provider.go |

### 9.3 调用链路一览

```
充值链路：
  用户充值 → 我方 deposit.go → RPC → 财务 rpc_provider.go（匹配通道）
         → 三方通道支付 → 回调 → 财务 callback.go → RPC → 我方 wallet.go（入账）

提现链路：
  用户提现 → 我方 withdraw.go（冻结余额）→ 财务 B端审核
         → 财务 withdraw_record.go（审核通过）→ RPC → 财务 rpc_provider.go（发起代付）
         → 三方通道代付 → 回调 → 财务 callback.go → RPC → 我方 wallet.go（扣冻结）
  或：驳回 → 财务 withdraw_record.go → RPC → 我方 wallet.go（解冻）

人工修正链路：
  运营操作 → 财务 adjustment.go → RPC QueryBalance（查余额）
         → 审核通过 → RPC CreditWallet / DebitWallet → 我方 wallet.go
```

---

## 十、文件总量统计

### 10.1 需手动编写的文件

| 目录层 | 文件数 | 文件列表 |
|--------|-------|---------|
| 根目录 | 2 | main.go, handler.go |
| cfg/ | 2 | mysql.go, redis.go |
| errs/ | 1 | code.go |
| enum/ | 5 | channel_type.go, order_status.go, adjust_type.go, payment_method.go, enum_registry.go |
| ser/ | 8 | channel.go, payment.go, deposit_record.go, withdraw_record.go, adjustment.go, callback.go, statistics.go, rpc_provider.go |
| rep/ | 7 | payment_channel.go, deposit_method.go, withdraw_method.go, channel_order.go, manual_adjustment.go, polling_rule.go, channel_statistics.go |
| cache/ | 3 | channel.go, payment.go, callback.go |
| channel/ | 3+N | adapter.go, polling.go, sign.go + provider/\*.go（每通道 1 文件） |
| utils/ | 1 | order_no.go |
| **合计** | **32+N** | N = 三方通道数量 |

### 10.2 与 ser-wallet 的规模对比

| 对比维度 | ser-wallet（我方） | ser-finance（同事） | 差异原因 |
|----------|------------------|-------------------|---------|
| IDL 方法数 | 37 | ~47 | 财务 B 端操作多（审核/锁定/导出） |
| IDL 域文件数 | 6 | 8 | 财务多了 callback/statistics/rpc 三个域 |
| ser/ 文件数 | 7 | 8 | 财务多了 callback.go |
| rep/ 文件数 | 9 | 7 | 钱包表更多（9 张 vs 7 张） |
| 特有层 | — | channel/（通道适配层 3+N 文件）| 三方通道对接是财务独有的 |
| 手动文件数 | 30 | 32+N | 财务因通道适配层略多 |
| 数据表 | 9 | 7 | 钱包表结构更复杂 |
| 接口总数 | 46 | ~53 | 财务含 2 个 HTTP 回调 |
| 调对方 RPC | 4（调财务） | 6（调钱包） | 财务是钱包的"上游操作者" |

---

## 十一、联调时我需要关注的关键点

### 11.1 充值入账 — 对方调我 CreditWallet 时传什么

```
我需要确认的字段：
  ├── userId:        int64    ← 哪个用户
  ├── currencyCode:  string   ← 哪种币
  ├── walletType:    int32    ← 1=中心（充值金额）/ 2=奖励（奖金部分）
  ├── amount:        string   ← 金额（DECIMAL字符串传输 or 最小单位整数？★待对齐★）
  ├── orderNo:       string   ← C+16位（充值）/ B+16位（补单）/ A+16位（加款）
  ├── opType:        int32    ← 操作类型枚举（充值=1/补单=4/加款=5）★待对齐★
  └── auditInfo:              ← 稽核信息（仅奖励钱包时携带）
      ├── auditMultiplier: decimal  ← 稽核倍数
      └── auditTurnover:   decimal  ← 要求流水金额
```

### 11.2 提现扣冻结 — 对方调我 DeductFrozenAmount 时传什么

```
我需要确认的字段：
  ├── userId, currencyCode, walletType  ← 同上
  ├── amount:        string   ← 提现金额
  └── orderNo:       string   ← T+16位
```

### 11.3 加款/减款 — 对方一次修正可能调我多次

```
一次加款操作（A+16位）最多调用 4 次 CreditWallet：
  CreditWallet(walletType=1, amount=X, orderNo=A+16)  ← 中心钱包
  CreditWallet(walletType=2, amount=Y, orderNo=A+16)  ← 奖励钱包
  CreditWallet(walletType=3, amount=Z, orderNo=A+16)  ← 主播钱包
  CreditWallet(walletType=4, amount=W, orderNo=A+16)  ← 代理钱包

幂等键 = orderNo + walletType 组合（★必须确认这一点★）
  同一 orderNo + 不同 walletType → 允许（4次调用各自独立）
  同一 orderNo + 同一 walletType → 拒绝（幂等拦截）
```

### 11.4 联调前必须与同事对齐的清单

| 序号 | 对齐项 | 影响 | 优先级 |
|------|--------|------|--------|
| 1 | CreditWallet 的 amount 字段类型（string decimal vs int64 最小单位） | 所有入账操作 | P0 |
| 2 | 钱包类型枚举值（1~5 的确切定义） | 所有 RPC 调用 | P0 |
| 3 | 幂等键组成（orderNo 单独 vs orderNo+walletType） | 加款/减款多次调用场景 | P0 |
| 4 | 操作类型枚举值（充值/补单/加款/减款/投注/奖励...） | CreditWallet/DebitWallet 的 opType | P0 |
| 5 | 充值回调后调 CreditWallet 的完整参数列表 | callback.go → wallet.go | P1 |
| 6 | 提现驳回时 UnfreezeBalance 的完整参数列表 | withdraw_record.go → wallet.go | P1 |
| 7 | GetPaymentMethods 的返回结构（充值/提现方式的字段定义） | rpc_provider.go → deposit.go/withdraw.go | P1 |
| 8 | MatchAndCreateChannelOrder 的入参出参 | rpc_provider.go → deposit.go | P1 |
| 9 | 财务 IDL 什么时候出？ | 影响我方 Phase 5/6 的启动时间 | P1 |
| 10 | 出款失败后人工处理流程 | withdraw_record.go 的重试逻辑 | P2 |

---

## 十二、与前序文档的衔接

| 本文内容 | 来源文档 | 关键段落 |
|----------|---------|---------|
| ~53 个接口清单 | 接口评估/tmp-2.md | 第六章 财务管理模块 |
| 7 张表的分级 | 表结构设计/tmp-2.md | 第一章 全局表结构地图（他方部分） |
| 充值回调→入账链路 | 流程思路/tmp-2.md | 第四部分 4.1 补单 / 第五部分 5.2 调用方向 |
| 提现审核→冻解链路 | 流程思路/tmp-2.md | 第二部分 2.4 提现流程 / 第五部分 5.3 三步模式 |
| 人工加减款→多次 RPC | 流程思路/tmp-2.md | 第四部分 4.2 加款 / 4.3 减款 |
| 通道轮询算法 | 需求理解/tmp-2.md | 第五部分 5.2 通道配置 |
| 提现 7 状态流转 | 需求理解/tmp-2.md | 第五部分 5.4 提现记录 |
| 6 大功能块划分 | 需求理解/tmp-2.md | 第五部分 5.2 财务管理 6 大功能块 |
| RPC 双向调用矩阵 | 接口评估/tmp-2.md | 第八章 模块间调用关系图 |
| channel/ 适配层设计 | 工程规范推演 | 参考策略模式 + 现有工程分层惯例 |
