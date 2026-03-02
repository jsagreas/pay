# 钱包模块 × 支付/财务模块：跨组技术沟通准备

> 身份：钱包模块（ser-wallet）负责人
> 场景：与支付/财务组（ser-finance）进行首次跨模块技术对齐会议
> 目标：消除双方盲区、明确接口契约、划清职责边界、暴露所有待确认事项
> 日期：2026-03-02

---

## 目录

- [一、开场：我方模块概述与职责声明](#一开场我方模块概述与职责声明)
- [二、我方需要你们提供什么（我方依赖清单）](#二我方需要你们提供什么)
- [三、你们需要调用我方什么（我方暴露清单）](#三你们需要调用我方什么)
- [四、充值全链路：逐步拆解谁做什么](#四充值全链路逐步拆解谁做什么)
- [五、提现全链路：逐步拆解谁做什么](#五提现全链路逐步拆解谁做什么)
- [六、人工操作链路：补单/加款/减款](#六人工操作链路补单加款减款)
- [七、必须当面对齐的核心问题（20问）](#七必须当面对齐的核心问题)
- [八、接口契约草案（供讨论）](#八接口契约草案供讨论)
- [九、数据格式与枚举对齐](#九数据格式与枚举对齐)
- [十、风险与阻塞项提前预警](#十风险与阻塞项提前预警)

---

## 一、开场：我方模块概述与职责声明

### 1.1 我方是谁

ser-wallet = **钱包模块 + 币种配置模块**，核心定位一句话：

> **余额只有钱包能改，钱包只做余额管理。**

我方不碰支付通道、不接三方回调、不做审核流程。我方提供 6 个金融操作原语（加款/扣款/冻结/解冻/确认扣款/钱包初始化），你方通过 RPC 调用这些原语来编排业务流程。

### 1.2 我方管什么

| 管什么 | 具体内容 |
|--------|---------|
| 用户钱包 | 5 种子钱包（中心/奖励/主播/代理/场馆）× 多币种的余额管理 |
| 币种配置 | 法币(VND/IDR/THB)、虚拟币(USDT)、平台币(BSB) 的全局配置 |
| 汇率体系 | 实时汇率（三方API均值）、平台汇率、入款汇率、出款汇率 |
| C 端页面 | 充值/提现/兑换的用户交互页面（方式选择、金额输入、账户填写） |
| 交易流水 | wallet_transaction（不可变流水表，记录每笔余额变更的 before/after） |
| 冻结管理 | 提现冻结三阶段（冻结→解冻 OR 确认扣款），freeze_detail 明细 |

### 1.3 我方不管什么（需要你方负责）

| 不管什么 | 为什么是你方 |
|----------|------------|
| 支付通道对接 | 通道 API 调用、密钥管理、通道轮询算法 |
| 三方回调接收 | 回调验签、幂等处理、回调路由 |
| 充值方式配置 | 哪些币种开放哪些充值方式（银行卡/电子钱包/USDT） |
| 提现方式配置 | 哪些币种开放哪些提现方式、限额配置 |
| 审核流程 | 提现的风控审核、财务审核、审批链 |
| 出款执行 | 调三方通道发起代付转账 |
| 人工操作页面 | B 端的补单/加款/减款的录入、审批界面 |
| 手续费计算 | 提现手续费率配置、费用计算 |

### 1.4 我方现状

- 币种配置模块：设计完成，待编码
- 钱包核心 RPC：设计完成（6 个金融原语 + 6 个查询），待编码
- C 端充值/提现页面：**骨架可做，但充值方式列表、提现方式列表、限额费率等数据依赖你方接口**
- 你方服务（ser-finance）：**目前代码库中不存在，IDL 未定义**
- RPC Client：你方的 RPC Client 尚未在 `common/rpc/rpc_client.go` 中注册

---

## 二、我方需要你们提供什么

### 2.1 RPC 接口需求清单（我方作为客户端调用你方）

| # | 我需要的能力 | 调用场景 | 紧急程度 | 说明 |
|---|------------|---------|---------|------|
| **F1** | 获取充值方式列表 | C 端用户进入充值页面 | 🔴 阻塞 | 返回该币种下可用的充值方式（银行卡/电子钱包/USDT/…） |
| **F2** | 获取提现方式列表 | C 端用户进入提现页面 | 🔴 阻塞 | 返回该币种下可用的提现方式及各方式的限额 |
| **F3** | 获取提现限额与费率 | C 端提现页面展示限额 | 🔴 阻塞 | 单笔最小/最大、单日累计、手续费率 |
| **F4** | 提交提现订单到财务审核 | 用户提交提现后 | 🔴 阻塞 | 钱包冻结金额后，需要通知你方启动审核流程 |
| **F5** | 创建通道订单（充值） | 用户选完方式金额后 | 🟡 重要 | 你方匹配通道、调三方 API 创建支付订单、返回支付信息 |
| **F6** | 查询通道订单状态 | 充值等待页面轮询 | 🟢 一般 | 用户等待支付结果时轮询订单状态 |

### 2.2 每个接口我期望的入参和出参（草案，供讨论）

#### F1：获取充值方式列表

```
我方调用时机：用户进入充值页面
我方传入：
  - currencyCode: string   // 当前选择的币种，如 "VND"
  - userId: i64            // 用户ID（可能影响可用方式，如新用户限制）

我方期望返回：
  - methods: list<DepositMethod>
    - methodId: i64         // 方式ID（后续创建订单时回传）
    - methodName: string    // 方式名称："银行卡转账"/"USDT"/"PromptPay"
    - methodType: i32       // 1=银行卡 2=电子钱包 3=USDT
    - icon: string          // 图标URL
    - minAmount: i64        // 最小充值金额（最小单位）
    - maxAmount: i64        // 最大充值金额
    - amountOptions: list<i64>  // 快捷金额选项 [100,500,1000,5000]
    - supportChains: list<string>  // USDT 时：["TRC20","ERC20"]
    - enabled: bool
```

**我方疑问**：
1. 充值方式是否按币种隔离？VND 能看到银行卡，但 USDT 币种能否看到银行卡？
2. 是否有用户级别限制？比如新注册用户 24h 内不能充值？
3. 快捷金额选项是固定的还是可配置的？配置在你方还是我方？

#### F2：获取提现方式列表

```
我方调用时机：用户进入提现页面
我方传入：
  - currencyCode: string
  - userId: i64
  - kycStatus: i32         // KYC状态(1=未认证 2=审核中 3=已通过 4=需重认)
                           // 我方已查过 ser-kyc，传给你方做方式过滤

我方期望返回：
  - methods: list<WithdrawMethod>
    - methodId: i64
    - methodName: string    // "银行卡"/"电子钱包"/"USDT"
    - methodType: i32       // 1=银行卡 2=电子钱包 3=USDT
    - icon: string
    - minAmount: i64        // 单笔最小
    - maxAmount: i64        // 单笔最大
    - dailyLimit: i64       // 单日累计限额
    - feeRate: string       // 手续费率，如 "0.02" (2%)
    - feeMin: i64           // 最低手续费
    - feeMax: i64           // 最高手续费（封顶）
    - supportChains: list<string>  // USDT时的链网络
    - requireKyc: bool      // 是否需要KYC已通过
    - enabled: bool
```

**我方疑问**：
1. KYC 过滤逻辑谁做？我方已有 KYC 状态，是我方过滤后只传"已通过/未通过"，还是全部传给你方，你方按 kycStatus 过滤可用方式？
2. 手续费是固定金额还是比例？还是阶梯（金额越大费率越低）？
3. 单日累计限额的"天"按哪个时区算？

#### F3：获取提现限额与费率

```
可能与 F2 合并，但如果独立：

我方传入：
  - currencyCode: string
  - userId: i64
  - withdrawMethodId: i64   // 用户选择的提现方式

我方期望返回：
  - singleMin: i64          // 单笔最小
  - singleMax: i64          // 单笔最大
  - dailyUsed: i64          // 今日已用额度
  - dailyRemain: i64        // 今日剩余额度
  - feeRate: string         // 费率
  - feeAmount: i64          // 按当前金额预算的手续费（如果前端需要实时展示）
```

**我方疑问**：
1. F2 和 F3 是否合并成一个接口？还是 F2 返回方式级别的限额，F3 返回用户级别的已用额度？
2. 今日已用额度怎么算——是你方查自己的提现记录统计，还是我方提供查询？

#### F4：提交提现订单到财务审核

```
我方调用时机：用户提现 → 我方冻结余额成功 → 创建 withdraw_order → 通知你方

我方传入：
  - withdrawOrderNo: string     // T+16位，我方生成
  - userId: i64
  - currencyCode: string
  - amount: i64                 // 提现总金额（最小单位）
  - feeAmount: i64              // 手续费（我方根据 F2/F3 返回的费率预算）
  - actualAmount: i64           // 实际到账 = amount - feeAmount
  - withdrawMethodId: i64       // 用户选的方式
  - accountInfo: WithdrawAccountInfo  // 收款账户信息
    - accountType: i32          // 1=银行卡 2=电子钱包 3=USDT
    - accountName: string       // 持卡人姓名
    - accountNo: string         // 卡号/钱包号/链地址
    - bankName: string          // 银行名称（银行卡时）
    - chainNetwork: string      // TRC20/ERC20（USDT时）
  - rateSnapshot: string        // 汇率快照（出款汇率）
  - clientIp: string
  - deviceType: string          // APP/H5/PC

我方期望返回：
  - success: bool
  - message: string             // 失败原因
```

**我方疑问**：
1. 手续费由谁最终确定？我方根据 F2/F3 返回的费率预算一个 feeAmount 传给你方，还是你方重新按费率计算？
2. 你方收到后是否再验证金额和限额？还是信任我方传入的值？
3. 提现审核有几级？风控 + 财务两级？还是可配置N级？
4. 你方审核通过后直接发起出款，还是需要我方再确认？

#### F5：创建通道订单（充值）

```
我方调用时机：用户选完充值方式和金额，点击"确认充值"

我方传入：
  - depositOrderNo: string      // C+16位，我方生成的充值订单号
  - userId: i64
  - currencyCode: string
  - amount: i64                 // 充值金额
  - depositMethodId: i64        // 用户选的充值方式
  - clientIp: string
  - deviceType: string

我方期望返回：
  - channelOrderNo: string      // 通道侧订单号
  - paymentType: i32            // 1=跳转URL 2=二维码 3=转账信息 4=链上地址
  - paymentUrl: string          // 跳转支付的URL（type=1时）
  - qrCodeUrl: string           // 二维码图片URL或内容（type=2时）
  - transferInfo: TransferInfo  // 转账详情（type=3时）
    - bankName: string
    - accountNo: string
    - accountName: string
    - branch: string
  - chainAddress: string        // 收款链上地址（type=4时，USDT充值）
  - chainNetwork: string        // TRC20/ERC20
  - expireTime: i64             // 支付超时时间
```

**我方疑问**：
1. 创建通道订单失败（通道不可用/限额超出）怎么处理？我方的充值订单已创建，是标记失败还是允许重选方式？
2. USDT 充值的收款地址是固定的（平台地址）还是每笔订单动态生成？
3. 通道轮询选择是你方内部逻辑，我方不感知？还是你方返回多个可用通道让用户选？
4. 支付超时后你方是否主动回调通知，还是我方轮询？

#### F6：查询通道订单状态

```
我方调用时机：充值等待页面轮询、或充值记录页面刷新

我方传入：
  - depositOrderNo: string      // 我方充值订单号

我方期望返回：
  - status: i32                 // 0=待支付 1=支付中 2=已成功 3=已失败 4=已超时
  - channelOrderNo: string
  - paidTime: i64               // 支付成功时间
  - paidAmount: i64             // 实际支付金额（可能与请求金额不同？）
```

---

## 三、你们需要调用我方什么

### 3.1 我方暴露给你方的 RPC 接口清单

| # | 方法 | 什么时候调 | 参数概要 | 返回概要 |
|---|------|-----------|---------|---------|
| **W-R1** | CreditWallet | 充值成功入账 / 奖金发放 / 补单 / 人工加款 | userId, currencyCode, walletType, amount, bizType, refOrderNo | success, afterBalance |
| **W-R2** | DebitWallet | 人工减款 | userId, currencyCode, walletType, amount, bizType, refOrderNo | success, afterBalance |
| **W-R3** | FreezeBalance | 提现冻结（一般我方内部调用） | userId, currencyCode, amount, refOrderNo | success, availableBalance, frozenBalance |
| **W-R4** | UnfreezeBalance | 审核驳回 / 出款失败 → 解冻退回 | userId, currencyCode, refOrderNo, reason | success |
| **W-R5** | ConfirmDebit | 出款成功 → 确认扣除冻结金额 | userId, currencyCode, refOrderNo | success |
| **W-R7** | GetBalance | 人工加减款弹窗查余额 | userId, currencyCode(可选) | wallets 列表(各子钱包余额) |
| **CC-R1** | GetCurrencyList | 币种下拉 / 报表筛选 | status(可选) | CurrencyInfo 列表 |
| **CC-R2** | GetExchangeRate | 报表汇率换算 | currencyCode, rateType(可选) | 四种汇率 + 更新时间 |

### 3.2 你方调用我方时的关键约束

**幂等保证**：
- CreditWallet/DebitWallet 的幂等键 = `(refOrderNo, bizType)`
- 你方用同样的 refOrderNo + bizType 重复调用 → 我方直接返回成功，不会重复入账
- **请务必保证同一笔业务使用同一个 refOrderNo 调用**

**互斥约束**：
- 对同一笔 refOrderNo，UnfreezeBalance 和 ConfirmDebit **互斥**
- 调了 ConfirmDebit 后再调 UnfreezeBalance → 我方返回错误"冻结已完结"
- 反之亦然

**余额不足处理**：
- DebitWallet 默认策略：余额不足 → 拒绝，返回错误码 6015101
- 如果人工减款场景需要"不足则扣实际可用"，需要在请求中传 `insufficientStrategy=2`
- 请你方根据业务场景选择策略

**bizType 枚举**（你方需要用的）：

| bizType 值 | 含义 | 你方什么场景用 |
|-----------|------|-------------|
| 1 | 充值入账 | 充值回调成功 → CreditWallet(bizType=1, walletType=1) |
| 2 | 充值奖金 | 充值有奖金 → CreditWallet(bizType=2, walletType=2) |
| 3 | 补单 | 补单审核通过 → CreditWallet(bizType=3) |
| 4 | 人工加款 | 人工加款审核通过 → CreditWallet(bizType=4) |
| 15 | 人工减款 | 人工减款审核通过 → DebitWallet(bizType=15) |

### 3.3 我方的 CreditWallet 详细参数

```thrift
struct CreditWalletReq {
    1: required i64 userId,
    2: required string currencyCode,     // "VND" / "BSB" / "USDT"
    3: required i32 walletType,          // 1=中心 2=奖励 3=主播 4=代理
    4: required i64 amount,              // 金额(最小单位，必须>0)
    5: required i32 bizType,             // 业务类型(见枚举)
    6: required string refOrderNo,       // 关联业务订单号(幂等键)
    7: optional string remark,           // 备注
    8: optional i32 turnoverMultiple,    // 流水倍数(奖金场景)
}

struct CreditWalletResp {
    1: required bool success,
    2: optional i64 afterBalance,        // 操作后可用余额
    3: optional string message,          // 失败原因
}
```

---

## 四、充值全链路：逐步拆解谁做什么

```
步骤    操作内容                           负责方          接口/RPC
────    ──────                             ──────          ────────
  1     用户打开充值页面                    我方C端
  2     展示币种选择 + 切换                 我方C端         (内部：查币种列表)
  3     查询该币种的充值方式列表            我方→你方       RPC F1: GetDepositMethods
  4     展示充值方式、金额输入              我方C端
  5     用户选方式、输金额、点击充值        我方C端
  6     我方创建充值订单 deposit_order      我方内部        (我方数据库)
  7     请求创建通道订单                    我方→你方       RPC F5: CreateChannelOrder
  8     你方轮询通道、调三方API创建订单     你方内部        (你方逻辑)
  9     返回支付信息(二维码/转账/链地址)    你方→我方       F5 返回
 10     我方展示支付信息给用户              我方C端
 11     用户完成支付(链外操作)              用户/三方
 12     三方回调通知支付结果                三方→你方       (你方回调接口)
 13     你方验签、幂等检查                  你方内部
 14     你方更新通道订单状态                你方内部
 15     充值成功→你方调我方加款             你方→我方       RPC W-R1: CreditWallet
 16     我方入账(中心钱包) + 写流水         我方内部
 17     有奖金→你方再调一次加款             你方→我方       RPC W-R1: CreditWallet(奖励)
 18     我方入账(奖励钱包) + 记录流水倍数   我方内部
 19     我方异步写审计日志                  我方→ser-blog   oneway AddActLog
 20     我方推送余额变更给前端              我方→C端        (WebSocket/消息)
```

### 充值链路中需要当面确认的问题

| # | 问题 | 为什么重要 |
|---|------|-----------|
| C1 | 步骤 6 和 7 的顺序：我方先创建自己的充值订单再调你方？还是先调你方成功后我方才创建？ | 如果先调你方失败了，我方已有空订单需处理 |
| C2 | 步骤 7 失败（通道不可用）后，用户是否可以重选方式重试？我方同一个 depositOrderNo 是否可以多次调 F5？ | 幂等和重试策略 |
| C3 | 步骤 12 三方回调，回调地址是你方提供还是统一配置？ | 部署和运维 |
| C4 | 步骤 15 调 CreditWallet，refOrderNo 用什么值？我方的 depositOrderNo(C+16) 还是你方的 channelOrderNo？ | 幂等键对齐 |
| C5 | 步骤 17 充值奖金，奖金金额和流水倍数谁计算？是你方计算好告诉我方金额，还是我方自己算？ | 奖金逻辑归属 |
| C6 | 充值成功后，我方 deposit_order 状态谁来改？是你方回调时一并告知我方改状态，还是我方 CreditWallet 成功后自己改？ | 状态同步 |
| C7 | USDT 充值是否走不同链路？平台自托管 vs 三方通道？ | USDT 可能不经过你方 |

---

## 五、提现全链路：逐步拆解谁做什么

```
步骤    操作内容                            负责方          接口/RPC
────    ──────                              ──────          ────────
  1     用户打开提现页面                     我方C端
  2     查用户信息(状态、国家)               我方→ser-user   RPC GetUserInfoExpend
  3     查用户KYC(状态、实名)                我方→ser-kyc    RPC KycQuery
  4     查提现方式列表(按KYC过滤)            我方→你方       RPC F2: GetWithdrawMethods
  5     查提现限额和费率                     我方→你方       RPC F3: GetWithdrawLimits
  6     展示可用方式、限额、费率             我方C端
  7     用户选方式、输金额、填/选账户信息    我方C端
  8     持卡人姓名 vs KYC实名校验            我方内部        (大小写不敏感 trim)
  9     计算手续费和实际到账                 ??? (需确认)    (用F2/F3返回的费率)
 10     我方冻结用户余额                    我方内部         FreezeBalance(内部)
 11     我方创建 withdraw_order              我方内部        (我方数据库)
 12     提交提现订单到你方审核               我方→你方       RPC F4: SubmitWithdrawOrder
 13     你方记录提现订单、进入风控审核       你方内部
 14     风控审核(人工/规则)                  你方B端
        ├── 驳回 → 你方调我方解冻            你方→我方       RPC W-R4: UnfreezeBalance
        └── 通过 → 进入财务审核
 15     财务审核(人工)                       你方B端
        ├── 驳回 → 你方调我方解冻            你方→我方       RPC W-R4: UnfreezeBalance
        └── 通过 → 发起出款
 16     你方调三方通道发起代付               你方内部
 17     三方通道执行转账(可能T+N)            三方
 18     三方回调代付结果                     三方→你方
        ├── 成功 → 你方调我方确认扣款        你方→我方       RPC W-R5: ConfirmDebit
        └── 失败 → ???(需确认)
 19     我方更新 withdraw_order 状态         我方内部
 20     我方异步写审计日志                   我方→ser-blog
 21     我方推送余额变更给前端               我方→C端
```

### 提现链路中需要当面确认的问题

| # | 问题 | 为什么重要 |
|---|------|-----------|
| W1 | 步骤 9 手续费计算：谁最终确定 feeAmount？我方根据 F2/F3 预算后传给你方，还是你方重新计算？ | 金额一致性 |
| W2 | 步骤 12 提交审核后，你方是否需要验证金额/限额？还是信任我方传的值？ | 双重校验 vs 单点 |
| W3 | 步骤 14/15 审核驳回时，你方调 UnfreezeBalance 的 refOrderNo 用什么？我方的 withdrawOrderNo(T+16)？ | 幂等键对齐 |
| W4 | 步骤 18 出款失败后：是你方调 UnfreezeBalance 解冻？还是保持冻结等人工处理？ | 参考实现中有两种做法，需统一 |
| W5 | 审核状态变更是否需要通知我方？你方驳回/通过后，我方 withdraw_order 的状态怎么同步？ | 我方需要更新自己的订单状态 |
| W6 | 提现订单是否两边都存一份？你方有自己的提现记录表，我方也有 withdraw_order，如何保持一致？ | 数据一致性 |
| W7 | 你方调 ConfirmDebit 或 UnfreezeBalance 时，是否附带额外信息（审核备注、出款交易号等）？我方需要记录 | 审计完整性 |
| W8 | 出款成功后，实际到账金额是否可能与申请金额不同？（比如通道扣了额外费用） | 金额差异处理 |

---

## 六、人工操作链路：补单/加款/减款

### 6.1 流程概要

```
三种人工操作都是同一模式：
  你方B端创建 → 你方审核 → 你方调我方RPC → 我方执行余额变更

补单(充值补单)：
  你方创建补单记录(B+16) → 审核通过 → 调 CreditWallet(bizType=3, walletType=1)
  → 如有奖金：再调 CreditWallet(bizType=2, walletType=2, turnoverMultiple=N)

人工加款：
  你方创建加款记录(A+16) → 审核通过 → 调 CreditWallet(bizType=4, walletType=按指定)
  → 可能一次操作加多个子钱包（中心+奖励各加不同金额）
  → 同一个 refOrderNo + 不同 walletType = 不同幂等键，可分别重试

人工减款：
  你方创建减款记录(M+16) → 审核通过 → 调 DebitWallet(bizType=15, walletType=按指定)
  → 余额不足策略由你方指定：拒绝 or 扣实际可用
```

### 6.2 人工操作需确认的问题

| # | 问题 | 说明 |
|---|------|------|
| M1 | 人工加减款弹窗中需要展示用户余额，你方调 GetBalance 时需要哪些信息？所有币种所有子钱包？还是指定币种？ | 影响 GetBalance 的返回粒度 |
| M2 | 人工加款是否支持同时给多个子钱包加不同金额？如果是，refOrderNo 怎么区分？ | 同一 refOrderNo + 不同 walletType 我方可以接受 |
| M3 | 人工减款余额不足时，你方期望什么行为？A=拒绝并返回错误 B=扣实际可用余额并返回实际扣了多少？ | 影响 DebitWallet 的策略参数 |
| M4 | 补单是否需要记录原始充值订单号？如果是，refOrderNo 用补单号(B+16)还是原充值号(C+16)？ | 幂等键和关联关系 |

---

## 七、必须当面对齐的核心问题（20问）

### 职责边界类（谁做什么）

| # | 问题 | 背景 |
|---|------|------|
| 1 | **充值订单归属**：充值订单(deposit_order)是我方存还是你方存？还是两边各存一份？ | 我方设计了 deposit_order 表，但你方也有通道订单。是否冗余？ |
| 2 | **提现订单归属**：同上，withdraw_order 两边是否各存一份？ | 如果都存，状态同步怎么做？ |
| 3 | **回调接收方**：三方回调（充值成功/出款结果）的 HTTP 端点在你方？ | 我方理解是你方接收回调，验签后通过 RPC 通知我方 |
| 4 | **USDT充值归属**：USDT 充值是走你方通道还是我方自托管？ | 需求原型里 USDT 是平台自托管地址，不走三方 |
| 5 | **充值奖金计算**：奖金金额和流水倍数谁算？你方还是我方？还是活动模块提供？ | 需要明确数据源 |

### 接口契约类（怎么对接）

| # | 问题 | 背景 |
|---|------|------|
| 6 | **refOrderNo 对齐**：你方调我方 CreditWallet 时，refOrderNo 传什么？你方的通道订单号？我方的 depositOrderNo？还是你方自己生成的业务单号？ | 幂等键必须全局唯一且稳定 |
| 7 | **金额格式**：金额是用 i64 最小单位还是 string 小数？ | 我方内部用 DECIMAL(20,4)，RPC 传输倾向 i64 最小单位 |
| 8 | **汇率获取方式**：你方需要汇率时，是每次 RPC 查我方？还是我方推送/你方缓存？ | 影响你方对我方的调用频率 |
| 9 | **你方 RPC 服务名**：你方的 ETCD 服务名是什么？我方需要注册 RPC Client | 比如 `finance_service` 或 `pay_service`？ |
| 10 | **你方 IDL 位置**：你方的 Thrift IDL 文件会放在 `/common/idl/ser-finance/` 下吗？ | 需要统一管理 |

### 流程细节类（怎么走通）

| # | 问题 | 背景 |
|---|------|------|
| 11 | **充值创建失败回滚**：通道订单创建失败后，我方已创建的 deposit_order 怎么处理？标记失败？允许重试？ | 需要定义失败处理策略 |
| 12 | **提现审核状态同步**：你方审核驳回/通过后，如何通知我方更新 withdraw_order 状态？RPC 回调？还是我方轮询？ | 我方需要展示最新状态给用户 |
| 13 | **出款失败处理**：出款失败后是自动解冻还是保持冻结等人工？ | 自动解冻有风险（用户立即重新提现，短时间大量请求），人工需要额外流程 |
| 14 | **充值超时处理**：充值订单超时（如30分钟未支付），谁关闭订单？你方通道侧关闭后通知我方？还是我方定时任务扫描？ | 需要定义超时机制 |
| 15 | **防重复充值**：同用户+同币种+同金额的重复充值检测，逻辑在我方还是你方？ | 我方设计了 1-2 次软拦截、≥3 次硬拦截的规则 |

### 数据对齐类（格式统一）

| # | 问题 | 背景 |
|---|------|------|
| 16 | **订单号格式**：是否确认充值 C+16、提现 T+16、补单 B+16、加款 A+16、减款 M+16？ | 全局唯一性和可读性 |
| 17 | **币种代码标准**：统一用 ISO 4217？如 VND、IDR、THB、USDT、BSB？ | 两方必须一致 |
| 18 | **精度标准**：金额字段你方也用 DECIMAL(20,4)？汇率字段也用 DECIMAL(20,8)？ | 跨模块金额运算不能有精度差 |
| 19 | **时间格式**：时间字段是否统一用 int64 毫秒时间戳？ | 工程规范一致性 |
| 20 | **错误码范围**：你方的服务错误码号段是多少？我方是 6015xxx | 避免冲突 |

---

## 八、接口契约草案（供讨论）

### 8.1 你方需要定义的 Thrift IDL（建议结构）

```thrift
// /common/idl/ser-finance/service.thrift

namespace go ser_finance

include "deposit.thrift"
include "withdraw.thrift"

service FinanceService {
    // 充值相关
    deposit.GetDepositMethodsResp GetDepositMethods(
        1: deposit.GetDepositMethodsReq req)

    deposit.CreateChannelOrderResp CreateChannelOrder(
        1: deposit.CreateChannelOrderReq req)

    deposit.QueryChannelOrderResp QueryChannelOrder(
        1: deposit.QueryChannelOrderReq req)

    // 提现相关
    withdraw.GetWithdrawMethodsResp GetWithdrawMethods(
        1: withdraw.GetWithdrawMethodsReq req)

    withdraw.GetWithdrawLimitsResp GetWithdrawLimits(
        1: withdraw.GetWithdrawLimitsReq req)

    withdraw.SubmitWithdrawOrderResp SubmitWithdrawOrder(
        1: withdraw.SubmitWithdrawOrderReq req)
}
```

### 8.2 我方已定义的 Thrift IDL（通知你方）

```thrift
// /common/idl/ser-wallet/wallet_rpc.thrift (草案)

service WalletService {
    // 写操作
    CreditWalletResp CreditWallet(1: CreditWalletReq req)
    DebitWalletResp DebitWallet(1: DebitWalletReq req)
    FreezeBalanceResp FreezeBalance(1: FreezeBalanceReq req)
    UnfreezeBalanceResp UnfreezeBalance(1: UnfreezeBalanceReq req)
    ConfirmDebitResp ConfirmDebit(1: ConfirmDebitReq req)
    InitWalletResp InitWallet(1: InitWalletReq req)

    // 查询操作
    GetBalanceResp GetBalance(1: GetBalanceReq req)
    BatchGetBalanceResp BatchGetBalance(1: BatchGetBalanceReq req)
    GetWalletFlowResp GetWalletFlow(1: GetWalletFlowReq req)

    // 币种配置查询
    GetCurrencyListResp GetCurrencyList(1: GetCurrencyListReq req)
    GetExchangeRateResp GetExchangeRate(1: GetExchangeRateReq req)
    GetBaseCurrencyResp GetBaseCurrency()
}
```

### 8.3 双方交互的时序图总结

```
充值入账：
  你方 ───CreditWallet(refOrderNo=充值订单号, bizType=1)──→ 我方
  你方 ───CreditWallet(refOrderNo=充值订单号, bizType=2)──→ 我方 (有奖金时)

提现流程：
  我方 ───SubmitWithdrawOrder(withdrawOrderNo)──→ 你方
  你方 ───UnfreezeBalance(refOrderNo=提现订单号)──→ 我方 (驳回/失败)
  你方 ───ConfirmDebit(refOrderNo=提现订单号)──→ 我方 (成功)

人工操作：
  你方 ───GetBalance(userId)──→ 我方 (查余额)
  你方 ───CreditWallet(refOrderNo=补单/加款单号, bizType=3/4)──→ 我方
  你方 ───DebitWallet(refOrderNo=减款单号, bizType=15)──→ 我方

汇率/币种查询：
  你方 ───GetCurrencyList()──→ 我方
  你方 ───GetExchangeRate(currencyCode)──→ 我方
```

---

## 九、数据格式与枚举对齐

### 9.1 需要双方统一的枚举

| 枚举 | 值 | 含义 | 谁定义 |
|------|---|------|--------|
| **walletType** | 1 | 中心钱包 | 我方定义 |
| | 2 | 奖励钱包 | |
| | 3 | 主播钱包 | |
| | 4 | 代理钱包 | |
| | 5 | 场馆钱包 | |
| **bizType** | 1 | 充值入账 | 我方定义，你方使用 |
| | 2 | 充值奖金 | |
| | 3 | 补单 | |
| | 4 | 人工加款 | |
| | 5 | 活动奖金 | |
| | 15 | 人工减款 | |
| **depositMethod** | 1 | 银行卡转账 | 你方定义 |
| | 2 | 电子钱包 | |
| | 3 | USDT | |
| **withdrawMethod** | 1 | 银行卡 | 你方定义 |
| | 2 | 电子钱包 | |
| | 3 | USDT | |
| **orderPrefix** | C | 充值 | 双方统一 |
| | T | 提现 | |
| | B | 补单 | |
| | A | 加款 | |
| | M | 减款 | |
| | E | 兑换 | |

### 9.2 金额精度标准（双方必须一致）

| 场景 | DB 存储 | RPC 传输 | 说明 |
|------|---------|---------|------|
| 金额 | DECIMAL(20,4) | i64 最小单位 | 法币乘以 10^精度位数 |
| 汇率 | DECIMAL(20,8) | string 小数字符串 | 避免 i64 溢出 |
| 手续费率 | DECIMAL(10,4) | string 小数字符串 | 如 "0.0200" 表示 2% |

### 9.3 时间戳标准

- 全部使用 **int64 毫秒时间戳**（工程统一规范）
- 时区：服务端统一 UTC，前端根据用户所在地转换

---

## 十、风险与阻塞项提前预警

### 10.1 阻塞风险

| 风险 | 影响 | 预案 |
|------|------|------|
| 你方接口未定义 | 我方 C 端充值/提现页面无法完整实现 | 我方先用 TODO/Mock 占位，骨架先行 |
| 你方 IDL 未创建 | 我方无法生成 RPC Client | 请尽快创建 Thrift IDL 并放入 `/common/idl/ser-finance/` |
| 你方 ETCD 服务名未确定 | 我方无法注册 RPC Client | 请确认服务名（如 `finance_service`） |
| 双方金额精度不一致 | 跨模块计算出现分差 | 今天必须对齐：DECIMAL(20,4) + i64 传输 |
| 回调幂等未对齐 | 三方重复回调导致重复入账 | 你方在调 CreditWallet 前做幂等检查，我方也有唯一索引兜底 |

### 10.2 我方可以独立推进的部分

| 模块 | 是否依赖你方 | 说明 |
|------|------------|------|
| 币种配置 B 端 CRUD | 不依赖 | 完全独立 |
| 币种配置 RPC 暴露 | 不依赖 | 你方可以用来查币种/汇率 |
| 汇率定时刷新 | 不依赖 | 纯内部逻辑 |
| 6 个金融原语 RPC | 不依赖 | 我方先实现，你方随时可调用 |
| 兑换全流程 | 不依赖 | 纯内部，无外部 RPC |
| 记录查询 | 不依赖 | 纯查询 |
| 充值 C 端页面 | **依赖 F1, F5** | 方式列表 + 通道订单 |
| 提现 C 端页面 | **依赖 F2, F3, F4** | 方式列表 + 限额 + 审核提交 |

### 10.3 建议的推进节奏

```
第一步（本次会后立即）：
  · 双方确认金额精度、枚举值、订单号格式
  · 你方确认 ETCD 服务名和 IDL 存放位置
  · 双方确认 refOrderNo 的生成规则和传递约定

第二步（1周内）：
  · 你方输出 F1~F6 的 Thrift IDL 草案
  · 我方输出 W-R1~R9 + CC-R1~R3 的 Thrift IDL
  · 双方 Code Review IDL 设计

第三步（2周内）：
  · 双方各自实现骨架代码
  · 我方先实现金融原语 RPC + 币种配置
  · 你方先实现充值方式/提现方式查询接口

第四步（3-4周）：
  · 联调充值全链路
  · 联调提现全链路
  · 联调人工操作链路
```

---

## 附录：我方的沟通态度

> 我是钱包模块负责人，钱包是资金引擎——**你方是流程编排者，我方是执行者**。
>
> 我方的原则很简单：
> 1. 你告诉我加多少钱给谁（CreditWallet），我保证幂等、原子、可追溯地执行
> 2. 你告诉我冻结/解冻/确认扣款（Freeze/Unfreeze/Confirm），我保证状态机正确互斥
> 3. 你查余额查汇率查币种（GetBalance/GetRate/GetCurrency），我保证数据准确及时
> 4. 我需要你提供充值方式、提现方式、限额费率、通道订单创建能力
>
> 边界清晰了，对接就不会有盲区。今天的目标就是把这个边界钉死。
