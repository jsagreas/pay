# 钱包×支付/财务 跨模块技术沟通准备文档

> **文档定位**：钱包模块负责人参加与支付/财务组技术交流会议的全面准备材料
> **核心目标**：知己知彼，消除盲区，明确依赖，达成协议约定
> **使用方式**：会前准备 → 会中引导 → 会后跟踪清单

---

## 目录

- [第1章：沟通背景与目标](#第1章沟通背景与目标)
- [第2章：双向RPC接口全景](#第2章双向rpc接口全景)
- [第3章：充值链路跨模块交互](#第3章充值链路跨模块交互)
- [第4章：提现链路跨模块交互](#第4章提现链路跨模块交互)
- [第5章：协议约定与技术契约](#第5章协议约定与技术契约)
- [第6章：我方需要对方澄清的问题清单](#第6章我方需要对方澄清的问题清单)
- [第7章：对方需要了解我方的关键设计](#第7章对方需要了解我方的关键设计)
- [第8章：IDL协调与集成排期](#第8章idl协调与集成排期)
- [第9章：异常场景与补偿机制共识](#第9章异常场景与补偿机制共识)
- [第10章：建议会议议程与准备检查](#第10章建议会议议程与准备检查)
- [附录：速查数字与关键术语](#附录速查数字与关键术语)

---

## 第1章：沟通背景与目标

### 1.1 项目背景

本项目是一个**泛娱乐 + 合规竞彩平台**，B/C双端，分布式微服务架构。技术栈为Go语言，CloudWeGo Hertz(HTTP) + Kitex(RPC/Thrift IDL) + GORM + ETCD服务发现 + TiDB/MySQL。

**ser-wallet（钱包服务）** 是平台的资金中枢，承担以下核心职责：
- **多币种管理**：VND / IDR / THB / USDT / BSB（平台积分）五种货币
- **多子钱包**：每用户每币种4个子钱包（中心 / 奖励 / 代理 / 主播）
- **余额生命周期**：充值入账、余额冻结/解冻、提现扣减、兑换流转
- **币种配置**：精度、汇率、限额等全局参数管理

### 1.2 为什么需要这次沟通

**ser-finance（财务/支付服务）是钱包模块最大的外部依赖，目前该服务尚不存在。**

具体现状：
- 钱包模块的充值、提现两大核心链路**强依赖**财务模块的支付渠道能力
- 双方存在 **12+ 个跨模块RPC调用**（wallet→finance 4个 + finance→wallet 8+个）
- 钱包P4阶段（充提对接）**完全被财务IDL交付阻塞**
- 关键流程归属（如代付发起方）、协议约定（如金额格式）等**尚未对齐**

如果不在开发初期对齐这些问题，后果是：
1. IDL设计不匹配 → 联调时大量返工
2. 流程归属不清 → 重复实现或两边都没做
3. 异常处理不一致 → 资金不一致（最严重后果）

### 1.3 沟通三大目标

| # | 目标 | 具体产出 |
|---|------|---------|
| 1 | **澄清双向RPC契约** | 确认12+个RPC的入参/返回/错误码/幂等键 |
| 2 | **确认关键流程归属** | 明确代付发起方、充值订单归属、奖金活动管理方 |
| 3 | **对齐集成排期** | IDL交付里程碑、Mock→Real切换时间、联调窗口 |

### 1.4 双方关系概述

```
┌──────────────┐                    ┌──────────────┐
│              │  ── 4个RPC调用 ──→  │              │
│  ser-wallet  │                    │ ser-finance  │
│   (钱包)     │  ←── 8个RPC回调 ── │  (财务/支付)  │
│              │                    │              │
└──────────────┘                    └──────────────┘
      │                                    │
      │  钱包还调用:                         │  财务还调用:
      │  · ser-kyc (KYC查询)                │  · 第三方支付渠道
      │  · ser-user (用户信息)               │  · 渠道回调处理
      │  · 工程服务 (雪花ID等)               │  · B端审核管理
      └────────────────────────────────────┘
```

**关系定性**：不是单向依赖，是**双向强耦合**——钱包需要财务的渠道能力，财务需要钱包的余额操作能力。

### 1.5 会议预期产出

- [ ] IDL契约初稿方向确认（双方各自的RPC结构框架）
- [ ] 5个P0致命问题的明确决议
- [ ] 集成里程碑时间表（IDL→代码生成→联调→上线）
- [ ] 异常处理与补偿机制的双方责任划分
- [ ] 遗留问题清单+负责人+下次沟通时间

---

## 第2章：双向RPC接口全景

### 2.1 我方暴露给财务的RPC（12个接口）

#### 写操作（6个）— 余额变动类

| 编号 | 方法名 | 核心入参 | 返回值 | 被调场景 | 优先级 |
|------|--------|---------|--------|---------|--------|
| R-1 | **CreditWallet** | userId, currencyCode, walletType, amount, orderNo, opType | success, newBalance, flowId | 充值入账、奖金发放、人工加款、返奖 | P0 |
| R-2 | **DebitWallet** | userId, currencyCode, walletType, amount, orderNo, opType | success, newBalance, flowId | 人工扣款、指定钱包扣减 | P0 |
| R-3 | **FreezeBalance** | userId, currencyCode, walletType, amount, orderNo | success, frozenBalance | 提现发起时冻结余额 | P0 |
| R-4 | **UnfreezeBalance** | userId, currencyCode, walletType, amount, orderNo | success, availableBalance | 提现驳回/出款失败时解冻 | P0 |
| R-5 | **DeductFrozenAmount** | userId, currencyCode, walletType, amount, orderNo | success, frozenBalance | 出款成功后正式扣除冻结金额 | P0 |
| R-7 | **DebitByPriority** | userId, currencyCode, amount, orderNo, opType | success, deductDetails[], remainingCenter, remainingReward | 投注等场景：center→reward自动优先级扣减 | P1 |

**关键设计要点（需要对方知晓）：**

1. **CreditWallet的多子钱包调用模式**：
   - 一笔充值可能需要调2次：center钱包入账 + reward钱包入奖金
   - 一笔人工加款可能调4次：center/reward/agent/anchor 各一次
   - 幂等键 = `orderNo + walletType` 组合，不是单独的orderNo
   - 示例：订单"A123"加款100 BSB → CreditWallet(A123, center, 70) + CreditWallet(A123, reward, 30)

2. **FreezeBalance的冻结金额**：
   - 冻结金额 = 提现金额 + 手续费（如果手续费方向是从余额额外扣的话）
   - **待确认**：手续费方向决定冻结公式

3. **DebitByPriority的返回值**：
   - 返回 `deductDetails[]` 详细说明从哪个子钱包扣了多少
   - 财务/投注模块**必须**用这个明细做返奖比例计算

#### 读操作（6个）— 查询类

| 编号 | 方法名 | 核心入参 | 返回值 | 被调场景 | 优先级 |
|------|--------|---------|--------|---------|--------|
| R-6 | **QueryWalletBalance** | userId, currencyCode(可选) | wallets[] (currencyCode, walletType, available, frozen, total) | B端创建加减款单前查余额 | P0 |
| R-8 | **GetCurrencyConfig** | onlyEnabled(可选) | currencies[], baseCurrency | 获取全平台币种配置元数据 | P0 |
| R-9 | **GetExchangeRate** | currencyCode, rateType(可选) | currencyCode, rateToBase, rateType, updateTime | 获取汇率（实时/平台/买入/卖出） | P0 |
| R-10 | **ConvertAmount** | amount, fromCurrency, toCurrency, rateType(可选) | convertedAmount, rateUsed, rateType | 跨币种金额换算（统一报表用） | P1 |
| R-11 | **BatchQueryBalance** | userIds[], currencyCode(可选) | balances map<userId, wallets[]> | 批量查余额（限100用户，运营报表） | P2 |
| R-12 | **GetWalletFlowList** | userId, currencyCode, walletType, opType, timeRange, page | total, flows[] | 查用户交易流水（审计/对账） | P2 |

### 2.2 我方需要调用财务的RPC（4个接口）

| 编号 | 方法名 | 期望入参 | 期望返回 | 调用时机 | 当前状态 |
|------|--------|---------|---------|---------|---------|
| O-1 | **GetPaymentMethods** | currencyCode, type(deposit/withdraw) | methods[] (methodId, name, icon, quickAmounts, min/maxAmount, feeRate, dailyLimit) | C端进入充值/提现页面 | **IDL未定义** |
| O-2 | **MatchAndCreateChannelOrder** | userId, currencyCode, amount, methodId, platformOrderNo | channelOrderNo, paymentType, paymentUrl/qrCodeUrl/walletAddress, expireTime | C端确认充值后创建渠道订单 | **IDL未定义** |
| O-3 | **InitiateChannelPayout** | withdrawOrderNo, methodId, currencyCode, amount, accountInfo{} | channelOrderNo, status | 提现审核通过后发起代付 | **归属待确认** |
| O-4 | **GetChannelOrderStatus** | channelOrderNo | status, updateTime, actualAmount(可选) | 定时补偿任务查漏补缺 | **IDL未定义** |

**关键说明**：
- 4个接口**全部**需要财务方定义IDL，我方是client
- P2-P3阶段我方使用Mock实现，P4切换为真实RPC调用
- O-3的**调用归属存在争议**（见第4章详述）

### 2.3 双向调用关系矩阵

```
                    ┌─────────── 充值链路 ───────────┐
                    │                               │
  wallet ──O-1──→ finance   获取充值方式列表         │
  wallet ──O-2──→ finance   创建渠道订单（匹配通道）  │
  finance ──R-1──→ wallet   充值入账（中心钱包）      │
  finance ──R-1──→ wallet   奖金入账（奖励钱包，可选） │
  wallet ──O-4──→ finance   补偿查询渠道状态（可选）   │
                    │                               │
                    └───────────────────────────────┘

                    ┌─────────── 提现链路 ───────────┐
                    │                               │
  wallet ──O-1──→ finance   获取提现方式列表         │
  wallet ──R-3──→ wallet    冻结余额（本地操作）      │
  wallet ──O-3?─→ finance   发起代付（归属待确认）    │
  finance ──R-5──→ wallet   出款成功→扣除冻结         │
  finance ──R-4──→ wallet   出款失败/驳回→解冻        │
                    │                               │
                    └───────────────────────────────┘

                    ┌─────────── 运营管理 ───────────┐
                    │                               │
  finance ──R-6──→ wallet   查询余额（B端表单回显）   │
  finance ──R-1──→ wallet   人工加款（1~4个子钱包）   │
  finance ──R-2──→ wallet   人工扣款（1~4个子钱包）   │
  finance ──R-8──→ wallet   获取币种配置             │
  finance ──R-9──→ wallet   获取汇率                 │
  finance ──R-10─→ wallet   金额换算（报表用）        │
                    │                               │
                    └───────────────────────────────┘
```

### 2.4 RPC优先级汇总

| 优先级 | 数量 | 接口 | 说明 |
|--------|------|------|------|
| P0 (Must) | 10 | R-1~R-6, R-8, R-9, O-1, O-2 | 充提核心链路必需 |
| P1 (Should) | 3 | R-7, R-10, O-3 | 投注扣减、金额换算、代付 |
| P2 (Could) | 3 | R-11, R-12, O-4 | 批量查询、流水查询、状态补偿 |

---

## 第3章：充值链路跨模块交互

### 3.1 充值五阶段流程与模块归属

```
阶段          ┃ 执行方     ┃ 操作                          ┃ 跨模块RPC
━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━
① 展示充值方式 ┃ 钱包(主)   ┃ 获取可用充值方式列表           ┃ wallet → finance: O-1
              ┃ 财务(供方)  ┃ 返回支付方式+渠道信息          ┃ GetPaymentMethods
━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━
② 创建充值订单 ┃ 钱包(主)   ┃ 本地校验+生成平台订单号        ┃ wallet → finance: O-2
              ┃           ┃ 重复转账三级拦截（本地DB查询）   ┃ MatchAndCreateChannelOrder
              ┃ 财务(供方)  ┃ 通道轮询匹配+调第三方创建支付单 ┃ 返回: 渠道订单号+支付信息
━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━
③ 用户支付    ┃ 外部       ┃ 用户在第三方完成支付           ┃ 无（钱包不参与）
              ┃ 财务       ┃ 接收渠道回调，确认支付结果      ┃
━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━
④ 充值入账    ┃ 财务(主)   ┃ 确认支付成功后回调钱包          ┃ finance → wallet: R-1
              ┃ 钱包(供方)  ┃ CreditWallet: 中心钱包入账     ┃ CreditWallet (center)
              ┃           ┃ CreditWallet: 奖金入账(如有)   ┃ CreditWallet (reward, 可选)
━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━
⑤ 补偿查询    ┃ 钱包       ┃ 定时任务查超时未回调的订单      ┃ wallet → finance: O-4
              ┃           ┃ 防止渠道回调丢失               ┃ GetChannelOrderStatus
```

### 3.2 阶段①详解：展示充值方式

**交互细节**：
```
用户打开充值页面
  → C端请求钱包 GET /api/wallet/deposit/methods?currency=VND
    → 钱包调财务 RPC: GetPaymentMethods(currency=VND, type=deposit)
      → 财务返回: [{methodId:1, name:"银行转账", icon:"xxx", quickAmounts:[100K,500K,1M], min:50K, max:50M}, ...]
    → 钱包组装返回C端（可能叠加本地业务逻辑，如排序、过滤）
```

**待澄清问题**：

| # | 问题 | 优先级 | 影响 |
|---|------|--------|------|
| Q3-1 | GetPaymentMethods的返回结构中，快捷金额档位(quickAmounts)是否包含在内？还是需要单独接口获取？ | P1 | 影响C端充值页面展示 |
| Q3-2 | 不同平台(App/H5/Web)是否返回不同的支付方式列表？入参是否需要platform字段？ | P2 | 影响接口设计 |
| Q3-3 | 支付方式列表是否按币种分别配置？VND和USDT的充值方式完全不同 | P1 | 影响业务逻辑 |

### 3.3 阶段②详解：创建充值订单

**交互细节**：
```
用户选择充值方式并输入金额
  → C端请求钱包 POST /api/wallet/deposit/order
    → 钱包本地校验:
      ├─ 重复转账三级拦截（查本地deposit_order表）
      │   ├─ 0笔处理中 → 通过
      │   ├─ 1笔处理中 → 警告，用户可force=true继续
      │   ├─ 2笔处理中 → 建议修改金额
      │   └─ ≥3笔处理中 → 拦截
      ├─ 金额范围校验（min ≤ amount ≤ max）
      └─ 生成平台订单号（C + 16位雪花ID）

    → 钱包写入本地充值订单（状态=待支付）

    → 钱包调财务 RPC: MatchAndCreateChannelOrder(
        userId, currencyCode=VND, amount=500000,
        methodId=1, platformOrderNo="C1234567890123456"
      )
      → 财务内部: 通道轮询匹配（成功率/时间/告警系数加权）→ 调第三方创建支付单
      → 返回: {channelOrderNo:"CH...", paymentType:1, paymentUrl:"https://..."}

    → 钱包更新本地订单（写入channelOrderNo）
    → 返回C端: 支付信息（跳转URL / QR码 / USDT地址）
```

**异常处理**：
- MatchAndCreateChannelOrder RPC失败 → 本地订单标记失败 → C端提示"暂不可用"
- **无资金影响**：充值是先支付后入账模式，此时余额尚未变动

**待澄清问题**：

| # | 问题 | 优先级 | 影响 | 我方倾向 |
|---|------|--------|------|---------|
| Q3-4 | 充值订单归属？钱包创建本地deposit_order + 财务创建channel_order？还是只有财务有订单？ | P0 | 数据模型设计 | 双方各建各的订单，platformOrderNo关联 |
| Q3-5 | MatchAndCreateChannelOrder失败时，是否需要财务提供失败原因码？（通道维护 vs 金额超限 vs 其他） | P1 | C端错误提示 | 需要，至少区分"通道不可用"和"金额超限" |
| Q3-6 | 渠道订单号和平台订单号的映射关系？是1:1还是1:N（一个平台订单可能匹配多个渠道）？ | P1 | 订单状态追踪 | 期望1:1 |

### 3.4 阶段④详解：充值入账回调

**交互细节**：
```
渠道回调财务确认支付成功
  → 财务调钱包 RPC: CreditWallet(
      userId, currencyCode=VND, walletType=CENTER(1),
      amount="500000", orderNo="C1234567890123456",
      opType=DEPOSIT(1)
    )
    → 钱包处理:
      ├─ 第一层幂等: 查deposit_order状态，已成功则直接return success
      ├─ 订单状态校验: 必须是"待支付"
      ├─ [BEGIN TX]
      │   ├─ UPDATE user_balance SET available = available + 500000
      │   │   WHERE user_id=X AND currency='VND' AND wallet_type=1
      │   ├─ INSERT balance_flow (type=CREDIT, opType=DEPOSIT, amount=500000, ...)
      │   │   UNIQUE KEY (orderNo, changeType) → 第二层幂等兜底
      │   └─ UPDATE deposit_order SET status=SUCCESS
      │ [COMMIT TX]
      └─ return: {success:true, newBalance:"1500000", flowId:"F..."}

  → 如有充值奖金活动:
    → 财务调钱包 RPC: CreditWallet(
        userId, currencyCode=BSB, walletType=REWARD(2),
        amount="50000", orderNo="C1234567890123456",
        opType=BONUS(5),
        auditMultiplier=3, auditTurnover=0  ← 流水要求倍数
      )
```

**待澄清问题**：

| # | 问题 | 优先级 | 影响 | 我方倾向 |
|---|------|--------|------|---------|
| Q3-7 | 充值入账的回调方式？财务直接调CreditWallet RPC？还是通过消息队列/HTTP webhook？ | **P0** | 架构设计 | 直接RPC调用，简单可靠 |
| Q3-8 | 充值奖金活动数据归谁管理？钱包查本地活动表？还是财务传入奖金参数？ | **P1** | 数据模型+接口设计 | 期望财务传入奖金参数（amount+multiplier），钱包只负责入账 |
| Q3-9 | 充值回调中actualAmount（实际到账金额）是否可能与请求金额不同？（如部分支付） | P2 | 金额校验逻辑 | 需要财务确认 |

### 3.5 USDT充值特殊流程

USDT充值与法币充值有本质区别：

```
法币充值: 选方式 → 输金额 → 跳转第三方支付 → 回调入账
USDT充值: 获取地址 → 用户转账 → 链上确认 → 回调入账
```

**待澄清问题**：

| # | 问题 | 优先级 | 影响 | 我方倾向 |
|---|------|--------|------|---------|
| Q3-10 | USDT充值地址分配模型？固定地址(每用户一个) vs 动态生成(每笔一个) vs 共享地址+memo？ | **P1** | 地址管理方式 | 由财务决定，钱包只展示地址 |
| Q3-11 | USDT充值是否走同一套MatchAndCreateChannelOrder？还是单独接口？ | P1 | 接口设计 | 期望走同一接口，paymentType区分 |
| Q3-12 | USDT充值的链网络(TRC20/ERC20)选择在哪一层？C端选择 vs 财务自动匹配？ | P2 | 用户交互设计 | 由产品+财务决定 |

---

## 第4章：提现链路跨模块交互

### 4.1 提现四阶段流程与模块归属

```
阶段           ┃ 执行方     ┃ 操作                          ┃ 跨模块RPC
━━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━
① 展示提现页面  ┃ 钱包(主)   ┃ 并行获取: KYC状态/提现方式/     ┃ wallet → finance: O-1
               ┃           ┃ 限额信息/已保存账户             ┃ wallet → ser-kyc
               ┃           ┃ 本地过滤提现方式（合规规则）      ┃ wallet → ser-user
━━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━
② 创建提现订单  ┃ 钱包(主)   ┃ 校验→冻结余额→创建订单→提交审核 ┃ wallet本地: R-3 FreezeBalance
               ┃           ┃                               ┃ wallet → finance: 提交审核
━━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━
③ 审核+出款    ┃ 财务(主)   ┃ 三级审核（风控→财务→出款）      ┃ 财务内部流程
               ┃           ┃ 调度渠道出款                   ┃ finance → 第三方渠道
━━━━━━━━━━━━━━╋━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━
④ 结果回调     ┃ 财务(主)   ┃ 出款成功: 通知钱包扣除冻结      ┃ finance → wallet: R-5
               ┃           ┃ 出款失败/驳回: 通知钱包解冻      ┃ finance → wallet: R-4
```

### 4.2 阶段①详解：展示提现页面

**交互细节（4个并行请求）**：
```
用户打开提现页面 → C端请求钱包 → 钱包并行发起:

  [并行1] RPC: wallet → ser-kyc.GetKYCStatus(userId)
           返回: kyc_status = VERIFIED | NOT_VERIFIED
           规则: 法币提现必须KYC VERIFIED; USDT提现不需要KYC

  [并行2] RPC: wallet → finance.GetPaymentMethods(currency, type=withdraw)
           返回: 全部提现方式候选集
           → 钱包本地过滤（合规规则: 资金从哪进就从哪出）:
             ├─ 查用户充值历史: SELECT DISTINCT payment_method FROM deposit_order
             │   WHERE user_id=X AND status=成功
             ├─ 只充过法币 → 只保留法币提现方式
             ├─ 只充过USDT → 只保留USDT提现方式
             ├─ 法币+USDT都充过 → 只保留法币提现（混合以法币为准？）
             └─ 从没充过 → 规则待定

  [并行3] 本地查询: 币种配置参数（提现限额、手续费、T+N天数）

  [并行4] 本地查询: 已保存的提现账户信息（首次提现后自动回填）
```

**待澄清问题**：

| # | 问题 | 优先级 | 影响 | 我方倾向 |
|---|------|--------|------|---------|
| Q4-1 | 混合充值来源的提现渠道过滤规则？法币+USDT都充过→只允许法币提现？还是两种都可以？ | **P1** | 提现方式过滤逻辑 | 需要合规+产品+财务三方确认 |
| Q4-2 | 提现方式列表中的限额信息(单笔min/max, 每日限额)是否包含在GetPaymentMethods返回中？还是需要单独接口？ | P1 | 接口数量 | 期望包含在内，减少RPC次数 |
| Q4-3 | USDT提现是否需要KYC？（加密货币合规要求因地区不同） | P1 | KYC校验逻辑 | 需要合规确认 |

### 4.3 阶段②详解：创建提现订单（最复杂环节）

**交互细节**：
```
用户确认提现
  → C端请求钱包 POST /api/wallet/withdraw/order

  ═══ 第一段: 并行校验（任一失败则拒绝）═══

  [并行] 限额校验:
    ├─ 单笔金额 ≥ 最低限额?
    ├─ 单笔金额 ≤ 最高限额?
    └─ 今日已提现累计 + 本笔 ≤ 每日限额?
        SELECT COALESCE(SUM(amount), 0) FROM withdraw_order
        WHERE user_id=X AND status IN (审核中,处理中)
        AND DATE(FROM_UNIXTIME(created_at)) = TODAY

  [并行] 户名一致性校验:
    ├─ RPC: wallet → ser-user.GetUserInfo(userId) 获取KYC真实姓名
    ├─ 比对: 提现账户户名 vs KYC认证姓名
    └─ 不一致 → 返回错误"提现账户户名与实名信息不符"

  [并行] 余额校验:
    └─ 中心钱包.可用余额 ≥ 提现金额 + 手续费?

  ═══ 第二段: 串行执行（取决于校验全部通过）═══

  Step 1: 冻结余额（关键步骤）
    UPDATE user_balance SET
      available = available - (amount + fee),
      frozen = frozen + (amount + fee)
    WHERE user_id=X AND currency=Y AND wallet_type=CENTER
      AND available >= (amount + fee)  ← CAS乐观锁
    如果 affected_rows = 0 → 余额不足（并发竞争），拒绝

  Step 2: 生成本地提现订单
    ├─ 平台订单号: T + 16位雪花ID
    ├─ 写入表: user_id/currency/amount/fee/method/status=审核中
    └─ 写入冻结流水（balance_flow）

  Step 3: 保存/更新提现账户信息（首次提现自动保存）

  Step 4: 提交财务审核（★ 关键跨模块交互）
    RPC: wallet → finance.SubmitWithdrawalAudit(
      userId, currency, amount, fee, method,
      platformOrderNo, withdrawalAccountInfo
    )
    返回: audit_order_no

    ★ 如果RPC失败（关键异常处理）:
      此时余额已冻结但财务没收到审核请求 → 用户的钱"卡住了"
      补偿流程:
      ├─ 解冻余额: available += (amount+fee), frozen -= (amount+fee)
      ├─ 更新订单状态 = 失败
      ├─ 写流水记录（冻结回退）
      └─ 返回C端: "提现暂不可用，请稍后重试"
      （本地补偿模式，Saga简化版）
```

**待澄清问题**：

| # | 问题 | 优先级 | 影响 | 我方倾向 |
|---|------|--------|------|---------|
| Q4-4 | 提现手续费方向？方案A: 从余额额外扣(用户提100，实扣102) vs 方案B: 从提现金额扣(用户提100，实收98)？ | **P0** | 冻结金额计算公式、用户体验 | 需要产品+财务确认 |
| Q4-5 | 提交财务审核用什么接口？财务是否有专门的SubmitWithdrawalAudit RPC？还是我方创建提现订单后财务B端自己拉取？ | **P0** | 接口设计、流程归属 | 期望推送模式（钱包主动提交），拉取模式会导致延迟 |
| Q4-6 | 提现订单在财务侧是否也会创建一条记录？两边订单状态如何同步？ | P1 | 数据一致性 | 各自维护订单，platformOrderNo关联 |

### 4.4 阶段③-④详解：审核+结果回调

**交互细节**：
```
财务三级审核流程（钱包不参与，只等结果）:
  风控审核(第一级) → 财务审核(第二级) → 出款审核(第三级)
    ↓
  调度渠道出款 → 第三方渠道 → 渠道回调结果
    ↓
  财务获得最终结果 → 回调钱包

═══ 结果A: 审核全通过 + 渠道出款成功 ═══

  finance → wallet RPC: DeductFrozenAmount(
    userId, currencyCode, walletType=CENTER,
    amount=(amount+fee), orderNo="T..."
  )
  钱包处理:
    ├─ 幂等校验: 查订单状态是否已处理
    ├─ [BEGIN TX]
    │   ├─ UPDATE user_balance SET frozen = frozen - (amount+fee)
    │   │   注意: 不动available，冻结时已减过
    │   ├─ UPDATE withdraw_order SET status=SUCCESS
    │   └─ INSERT balance_flow (type=DEBIT, opType=WITHDRAW_DEDUCT)
    │ [COMMIT TX]
    └─ return success
  结果: 冻结金额彻底离开系统，用户钱到了银行

═══ 结果B: 任一级审核驳回 OR 渠道出款失败 ═══

  finance → wallet RPC: UnfreezeBalance(
    userId, currencyCode, walletType=CENTER,
    amount=(amount+fee), orderNo="T..."
  )
  钱包处理:
    ├─ 幂等校验
    ├─ [BEGIN TX]
    │   ├─ UPDATE user_balance SET
    │   │   available = available + (amount+fee),
    │   │   frozen = frozen - (amount+fee)
    │   ├─ UPDATE withdraw_order SET status=FAILED, failReason=...
    │   └─ INSERT balance_flow (type=CREDIT, opType=WITHDRAW_REFUND)
    │ [COMMIT TX]
    └─ return success
  结果: 冻结的钱退回可用余额，用户看到提现失败
```

### 4.5 关键归属争议：InitiateChannelPayout

```
                     提现审核通过后，谁发起代付？

Option A（概率70%，更合理）:
┌──────────────────────────────────────────────────────────────┐
│ 财务审核通过 → 财务自行调度渠道出款 → 财务自行调第三方         │
│ 钱包不参与代付环节                                           │
│ 钱包只在"冻结"和"最终结果回调"两个点被调用                    │
│                                                             │
│ 好处: 出款逻辑内聚在财务模块，钱包不需要知道渠道细节          │
│ 坏处: 钱包不知道出款进度，只能等回调                          │
└──────────────────────────────────────────────────────────────┘

Option B（概率30%）:
┌──────────────────────────────────────────────────────────────┐
│ 财务审核通过 → 通知钱包 → 钱包调财务InitiateChannelPayout     │
│ 钱包在审核通过后主动发起代付请求                               │
│                                                             │
│ 好处: 钱包掌握主动权，可以做更多本地逻辑                      │
│ 坏处: 出款能力跨模块分裂，增加复杂度                          │
└──────────────────────────────────────────────────────────────┘

★ 我方倾向: Option A（财务自行发起代付）
★ 理由: 出款涉及渠道调度、通道轮询、费率计算等，
         这些都是财务模块的核心能力，不应该分散到钱包
★ 必须本次会议确认
```

**待澄清问题**：

| # | 问题 | 优先级 | 影响 | 我方倾向 |
|---|------|--------|------|---------|
| Q4-7 | InitiateChannelPayout调用归属？Option A(财务自行发起) vs Option B(钱包发起)？ | **P0** | 架构设计、代码归属 | Option A: 财务自行发起 |
| Q4-8 | 提现结果回调方式？直接调钱包RPC(DeductFrozenAmount/UnfreezeBalance)？还是消息队列/webhook？ | **P0** | 集成方式 | 直接RPC调用 |
| Q4-9 | 财务审核过程中，钱包能否查到审核进度？是否需要GetWithdrawalAuditStatus接口？ | P2 | C端体验 | 可选，优先级低 |
| Q4-10 | 提现被驳回时，驳回原因从哪里获取？财务回调传入 vs 钱包主动查询？ | P1 | C端展示 | 期望回调时传入failReason字段 |

### 4.6 提现状态转移图

```
用户发起提现
     ↓
 冻结余额成功?
  ↙       ↘
 是        否
 ↓         ↓
审核中    失败(余额不足)
 ↓
提交财务审核成功?
 ↙       ↘
 是       否
 ↓        ↓
等待审核  解冻余额→失败
 ↓       (审核提交RPC失败,本地补偿)
财务审核结果
 ↙          ↘
通过+出款成功  拒绝/出款失败
 ↓              ↓
扣除冻结       解冻余额
状态=成功      状态=失败
(DeductFrozen) (Unfreeze)
```

---

## 第5章：协议约定与技术契约

### 5.1 金额传输格式（P0）

**争议焦点**：RPC接口中金额字段用什么类型传输？

| 方案 | 类型 | 示例 | 优点 | 缺点 |
|------|------|------|------|------|
| A | **string** (Decimal文本) | "500000.00" | 无精度丢失、人类可读、跨语言安全 | 需要解析、占用空间稍大 |
| B | **i64** (最小单位整数) | 50000000 (VND精度2位) | 计算高效、无解析开销 | 双方必须知道精度位数、大数溢出风险 |

**我方倾向：string (方案A)**

理由：
1. 钱包内部用BIGINT最小单位存储，但RPC传输层用string更安全
2. 不同币种精度不同(VND:0, USDT:6, BSB:2)，i64要求双方都清楚精度映射
3. string在Thrift IDL中天然支持，无精度丢失
4. 钱包侧统一做 `ToMinUnit(amountStr, precision)` 和 `ToDisplayAmount(minUnit, precision)` 转换

**示例**：
```
CreditWallet请求: amount = "500000"      (VND, 精度0位, 表示50万越南盾)
CreditWallet请求: amount = "100.50"      (USDT, 精度2位, 表示100.50 USDT)
CreditWallet请求: amount = "1000.00"     (BSB, 精度2位, 表示1000积分)

钱包内部存储:
  VND 500000 → BIGINT 500000    (精度0, 乘以10^0)
  USDT 100.50 → BIGINT 10050000 (精度6, 乘以10^6)
  BSB 1000.00 → BIGINT 100000   (精度2, 乘以10^2)
```

**需要对方确认**：财务侧金额字段也用string？还是有自己的内部格式？

### 5.2 walletType枚举对齐（P0）

```
enum WalletType {
    CENTER = 1      // 中心钱包（充值/提现主钱包）
    REWARD = 2      // 奖励钱包（活动奖金、返奖）
    AGENT  = 3      // 代理钱包（代理佣金）
    ANCHOR = 4      // 主播钱包（主播收入）
}
```

**对齐要点**：
- 财务调CreditWallet/DebitWallet时必须传正确的walletType
- 一笔操作可能涉及多个walletType（如充值 + 奖金 = center + reward）
- 钱包侧用 `(orderNo, walletType)` 组合做幂等键
- **双方必须使用完全一致的枚举值**

### 5.3 opType枚举完整清单（P0）

**信用类（增加余额）**：
```
enum CreditOpType {
    DEPOSIT         = 1   // 充值入账
    BONUS           = 2   // 充值奖金
    MANUAL_CREDIT   = 3   // 人工加款
    RETURN_AWARD    = 4   // 返奖
    ACTIVITY_REWARD = 5   // 活动奖励
    ANCHOR_INCOME   = 6   // 主播收入
    AGENT_COMMISSION= 7   // 代理佣金
    WITHDRAW_REFUND = 8   // 提现退回（解冻回可用）
    EXCHANGE_IN     = 9   // 兑换转入
    ADJUSTMENT      = 10  // 系统调账
}
```

**借记类（减少余额）**：
```
enum DebitOpType {
    WITHDRAW        = 1   // 提现扣款
    MANUAL_DEBIT    = 2   // 人工扣款
    BET_DEDUCT      = 3   // 投注扣款
    PURCHASE        = 4   // 购买（礼物等）
    EXCHANGE_OUT    = 5   // 兑换转出
    FREEZE          = 6   // 冻结（提现冻结）
    DEDUCT_FROZEN   = 7   // 扣除冻结（出款成功）
    ADJUSTMENT      = 8   // 系统调账
}
```

**需要对方确认**：
1. 上述枚举值是否覆盖了财务侧的所有调用场景？
2. 是否需要增加新的opType？（如手续费扣款、利息等）
3. 枚举值的数字编号双方是否需要强一致？

### 5.4 错误码契约（P1）

**钱包侧RPC返回的业务错误码**：

| 错误码 | 含义 | 触发场景 | 财务侧处理建议 |
|--------|------|---------|--------------|
| SUCCESS | 操作成功 | 正常完成 或 幂等命中 | 正常处理 |
| BALANCE_NOT_ENOUGH | 余额不足 | DebitWallet/FreezeBalance时可用余额不够 | 提示用户/标记失败 |
| FROZEN_NOT_ENOUGH | 冻结余额不足 | DeductFrozenAmount时冻结金额不够 | 异常告警，人工介入 |
| DUPLICATE_ORDER | 重复订单（兜底） | 唯一键冲突（正常不应该到这里） | 视为成功 |
| USER_NOT_FOUND | 用户不存在 | 用户ID无效 | 检查数据 |
| CURRENCY_NOT_FOUND | 币种不存在/未启用 | 币种配置问题 | 检查币种配置 |
| INVALID_WALLET_TYPE | 钱包类型无效 | walletType不在1-4范围 | 检查参数 |
| INVALID_AMOUNT | 金额无效 | 金额≤0 或 解析失败 | 检查参数 |
| SYSTEM_ERROR | 系统内部错误 | DB异常等 | 重试或人工介入 |

**关键约定**：
- `SUCCESS` 包含两种情况：首次成功 和 幂等命中（都返回SUCCESS）
- 财务侧收到`SUCCESS`后不需要区分是首次还是幂等命中
- 收到`SYSTEM_ERROR`时，幂等接口可安全重试

### 5.5 幂等键设计（P0）

**核心规则**：幂等键 = `orderNo + walletType` 组合

```
场景1: 充值入账（1笔订单 = 1次CreditWallet）
  CreditWallet(orderNo="C001", walletType=CENTER, amount="500000")
  幂等键: ("C001", CENTER) → balance_flow UNIQUE KEY

场景2: 充值入账 + 奖金（1笔订单 = 2次CreditWallet）
  CreditWallet(orderNo="C001", walletType=CENTER, amount="500000")
  CreditWallet(orderNo="C001", walletType=REWARD, amount="50000")
  幂等键: ("C001", CENTER) + ("C001", REWARD) → 两条独立流水

场景3: 人工加款（1笔订单 = 最多4次CreditWallet）
  CreditWallet(orderNo="ADJ001", walletType=CENTER, amount="70000")
  CreditWallet(orderNo="ADJ001", walletType=REWARD, amount="30000")
  CreditWallet(orderNo="ADJ001", walletType=AGENT,  amount="10000")
  CreditWallet(orderNo="ADJ001", walletType=ANCHOR, amount="5000")
  幂等键: 4个独立组合，互不干扰

场景4: 重试安全
  第一次调 CreditWallet(orderNo="C001", walletType=CENTER) → 成功
  第二次调 CreditWallet(orderNo="C001", walletType=CENTER) → 幂等命中，返回SUCCESS
  余额只增加一次
```

**对方需要遵守的规则**：
1. 同一笔操作，不同子钱包必须用**相同的orderNo + 不同的walletType**
2. 不同笔操作必须用**不同的orderNo**
3. 重试时必须用**完全相同的参数**（orderNo+walletType+amount）
4. **禁止**用同一个orderNo+walletType但不同amount调用（行为未定义）

### 5.6 超时与重试约定（P1）

| 项目 | 约定值 | 说明 |
|------|--------|------|
| RPC超时 | **3秒** | 所有跨模块RPC调用的统一超时阈值 |
| 连接超时 | **1秒** | Kitex连接建立超时 |
| 重试次数 | **1次** | 超时后重试1次（仅限幂等接口） |
| 幂等接口 | R-1~R-5, R-7 | CreditWallet/DebitWallet/Freeze/Unfreeze/DeductFrozen/DebitByPriority |
| 非幂等接口 | O-2 | MatchAndCreateChannelOrder（可能创建重复渠道订单） |
| 查询接口 | R-6, R-8~R-12, O-1, O-4 | 天然幂等，可安全重试 |

**关键约定**：
- 写操作(R-1~R-5)收到超时，不确定是否成功时 → **应该重试**（因为幂等）
- O-2 MatchAndCreateChannelOrder收到超时 → **不应重试**（可能创建重复渠道订单）
- 如果O-2超时且不确定是否成功 → 用O-4查询状态，或由财务侧做补偿

### 5.7 回调协议（P0）

**当前方案：直接RPC调用**

```
财务确认充值成功 → 直接调钱包 CreditWallet RPC
财务确认出款成功 → 直接调钱包 DeductFrozenAmount RPC
财务审核驳回     → 直接调钱包 UnfreezeBalance RPC
```

**替代方案对比**：

| 方案 | 实时性 | 可靠性 | 复杂度 | 适合场景 |
|------|--------|--------|--------|---------|
| 直接RPC调用 | 毫秒级 | 中（依赖对方在线） | 低 | 低并发、强一致要求 |
| 消息队列(Kafka等) | 秒级 | 高（持久化+重试） | 高 | 高并发、最终一致可接受 |
| HTTP Webhook | 秒级 | 中 | 中 | 跨网络、第三方集成 |

**我方倾向：直接RPC调用**
- 理由：项目规模尚小，团队统一使用Kitex RPC，不引入额外中间件
- 可靠性补充：财务侧在调用失败时需实现重试逻辑
- 如果未来并发量上升，可升级为消息队列方案

**需要确认**：财务侧是否同意直接RPC模式？是否已有消息队列基础设施？

### 5.8 金额精度映射表（P0）

| 币种 | 精度位数 | 最小单位 | 示例 | BIGINT存储 | 说明 |
|------|---------|---------|------|-----------|------|
| VND  | **待确认** | **待确认** | 500,000 VND | 500000 (如精度0) 或 50000000 (如精度2) | 越南盾通常0位小数 |
| IDR  | **待确认** | **待确认** | 100,000 IDR | 同上 | 印尼盾通常0位小数 |
| THB  | **待确认** | **待确认** | 1,000.00 THB | 100000 (如精度2) | 泰铢通常2位小数 |
| USDT | **待确认** | **待确认** | 100.50 USDT | 100500000 (如精度6) | 通常6位小数 |
| BSB  | **待确认** | **待确认** | 1,000.00 BSB | 100000 (如精度2) | 平台积分，建议2位 |

**需要确认**：每个币种的精度位数是固定不变的设计参数，一旦确定上线后不可修改（否则历史数据全部失效）。

---

## 第6章：我方需要对方澄清的问题清单

### 6.1 P0 — 致命级（阻塞架构决策，必须本次会议确认）

#### Q-P0-1: ser-finance IDL交付时间节点

| 维度 | 内容 |
|------|------|
| **问题** | 财务/支付模块的Thrift IDL文件什么时候能提交到 `common/idl/ser-finance/`？ |
| **影响** | 钱包P4阶段（充提对接）完全被此阻塞。没有IDL → 无法生成Kitex代码 → 无法切换Mock→Real |
| **我方现状** | P2-P3阶段使用Mock实现（FinanceService接口+MockImpl），P4需要切换为真实RPC |
| **期望** | 给出具体日期，或至少给出"我方IDL初稿比你方IDL晚/早几周"的相对时间 |
| **底线** | 如果短期无法交付完整IDL，能否先交付 GetPaymentMethods + MatchAndCreateChannelOrder 两个最核心的？ |

#### Q-P0-2: InitiateChannelPayout调用归属

| 维度 | 内容 |
|------|------|
| **问题** | 提现审核通过后，代付（出款）由谁发起？ |
| **选项** | Option A: 财务审核通过后自行调度渠道出款（钱包不参与）<br>Option B: 财务审核通过后通知钱包，钱包调财务InitiateChannelPayout |
| **影响** | 决定钱包是否需要实现O-3接口调用。影响代码量约200-300行 |
| **我方倾向** | **Option A**：出款是财务核心能力，不应分散到钱包。钱包只负责"冻结"和"最终结果处理" |
| **需要对方** | 明确选择A或B，如果选B需要说明理由 |

#### Q-P0-3: 充值入账回调方式

| 维度 | 内容 |
|------|------|
| **问题** | 渠道充值成功后，财务如何通知钱包入账？ |
| **选项** | A: 直接调钱包CreditWallet RPC<br>B: 通过消息队列<br>C: HTTP webhook |
| **影响** | 决定钱包入站RPC的实现方式和可靠性保障策略 |
| **我方倾向** | **A: 直接RPC调用**，简单可靠，与现有Kitex体系一致 |
| **需要对方** | 确认方式。如果选A，财务需承诺在调用失败时做重试 |

#### Q-P0-4: 提现结果回调方式

| 维度 | 内容 |
|------|------|
| **问题** | 提现出款成功/失败后，财务如何通知钱包？ |
| **选项** | 同Q-P0-3 |
| **影响** | 出款成功→调DeductFrozenAmount，失败/驳回→调UnfreezeBalance |
| **我方倾向** | **直接RPC调用**，与充值入账保持一致 |
| **需要对方** | 确认方式。失败场景下，回调是否传入failReason字段？ |

#### Q-P0-5: 每个币种的精度位数确认

| 维度 | 内容 |
|------|------|
| **问题** | VND/IDR/THB/USDT/BSB 各自的小数精度位数是多少？ |
| **影响** | 影响所有金额存储（BIGINT换算系数）、所有金额传输（string解析）、所有金额展示 |
| **我方倾向** | VND:0, IDR:0, THB:2, USDT:6, BSB:2（但需要产品+财务确认） |
| **不可修改** | 精度位数是currency_config表的immutable字段，上线后不可修改 |
| **需要对方** | 给出明确数字，如与我方预设不同需说明原因 |

### 6.2 P1 — 严重级（影响详细设计，尽快确认）

#### Q-P1-6: 充值奖金活动数据归属

| 维度 | 内容 |
|------|------|
| **问题** | 充值奖金活动（如"首充送50%"）的活动数据归谁管理？ |
| **选项** | A: 钱包管理（本地活动表）<br>B: 财务管理（财务告诉钱包奖多少）<br>C: 独立活动服务 |
| **影响** | 决定C-15(GetBonusActivities)接口的数据源 |
| **我方倾向** | **B: 财务管理**。财务调CreditWallet时直接传入bonus_amount和auditMultiplier，钱包只负责入账 |
| **需要对方** | 确认活动配置在哪里维护（财务B端？独立后台？） |

#### Q-P1-7: 提现手续费扣减方向

| 维度 | 内容 |
|------|------|
| **问题** | 提现手续费从哪里扣？ |
| **选项** | A: 从余额额外扣（用户提100，实扣100+2手续费=102，冻结102，到账100）<br>B: 从提现金额扣（用户提100，冻结100，到账98，手续费2在渠道扣） |
| **影响** | 冻结金额 = amount（方案B） vs amount+fee（方案A），直接影响FreezeBalance/DeductFrozenAmount的amount参数 |
| **我方倾向** | **方案A（额外扣）**更清晰，冻结金额=amount+fee，到账金额=amount |
| **需要对方** | 确认方案。如果选B，手续费是渠道扣还是财务扣？ |

#### Q-P1-8: 混合充值来源的提现渠道过滤

| 维度 | 内容 |
|------|------|
| **问题** | 用户既充过法币又充过USDT，提现时能选什么方式？ |
| **选项** | A: 只允许法币提现（混合以法币为准）<br>B: 两种都可以<br>C: 按充值比例限制 |
| **影响** | C-7(GetWithdrawMethods)的过滤逻辑 |
| **我方倾向** | 需要合规+产品确认，我方倾向**A（只法币）**，更安全 |
| **需要对方** | 财务侧是否有合规要求？不同地区规则是否不同？ |

#### Q-P1-9: 财务B端人工加减款审核流程

| 维度 | 内容 |
|------|------|
| **问题** | 运营在财务B端做人工加款/扣款，审核流程是什么样的？ |
| **选项** | A: 财务B端发起→财务内审→审核通过后调钱包RPC<br>B: 财务B端发起→直接调钱包RPC（无审核） |
| **影响** | 钱包是否需要做额外的权限/审核校验 |
| **我方倾向** | **A（先审核后调用）**更安全。钱包信任财务的调用（调了就执行），审核在财务侧完成 |
| **需要对方** | 确认审核流程。如果无审核（B），钱包是否需要做额外的风控检查？ |

#### Q-P1-10: USDT充值地址分配模型

| 维度 | 内容 |
|------|------|
| **问题** | USDT充值的钱包地址怎么分配？ |
| **选项** | A: 每用户固定一个地址<br>B: 每笔充值动态生成一个临时地址<br>C: 共享地址+memo区分 |
| **影响** | 地址管理归财务，但钱包需要在C端展示地址信息 |
| **我方倾向** | 由财务决定，钱包只展示。但需要知道返回结构（address + network + memo?） |
| **需要对方** | 选择方案，并确认返回结构 |

### 6.3 P2 — 重要级（影响开发排期，可后续确认）

#### Q-P2-11: GetPaymentMethods返回结构

| 维度 | 内容 |
|------|------|
| **问题** | 返回中是否包含限额信息(min/max/dailyLimit)？手续费率？快捷金额档位？ |
| **影响** | 钱包是否需要单独调另一个接口获取限额 |
| **我方倾向** | 全部包含在一个接口返回中，减少RPC调用次数 |

#### Q-P2-12: 订单号映射关系

| 维度 | 内容 |
|------|------|
| **问题** | 平台订单号(platformOrderNo)与渠道订单号(channelOrderNo)是1:1还是1:N？ |
| **影响** | 订单状态追踪和对账逻辑 |
| **我方倾向** | 1:1，一个充值订单对应一个渠道订单 |

#### Q-P2-13: 汇率方向性

| 维度 | 内容 |
|------|------|
| **问题** | 汇率是否区分买入(inbound)和卖出(outbound)方向？ |
| **影响** | GetExchangeRate的rateType参数设计 |
| **我方倾向** | 支持4种类型（实时/平台/买入/卖出），但需要确认业务是否真的需要区分 |

#### Q-P2-14: 现有IDL参考

| 维度 | 内容 |
|------|------|
| **问题** | 财务模块是否有其他项目的现成IDL可参考？ |
| **影响** | 加速IDL设计和对齐 |
| **我方倾向** | 如果有，希望能共享参考 |

---

## 第7章：对方需要了解我方的关键设计

### 7.1 我方的设计承诺（安全保证）

#### 并发安全：CAS乐观锁

```sql
-- 每一笔余额操作都使用条件更新，绝不超扣
UPDATE user_balance SET
  available = available - #{amount}
WHERE user_id = #{userId}
  AND currency_code = #{currencyCode}
  AND wallet_type = #{walletType}
  AND available >= #{amount}        -- CAS条件：可用余额必须足够

-- affected_rows = 0 → 余额不足或并发冲突，返回BALANCE_NOT_ENOUGH
-- affected_rows = 1 → 成功
```

**含义**：即使财务同时调多次DebitWallet，也不会出现超扣。钱包保证。

#### 幂等保证：双层防护

```
第一层（应用层，快速路径）：
  收到CreditWallet请求 → 先查balance_flow表是否存在(orderNo, walletType)
  → 已存在且成功 → 直接返回SUCCESS（不做任何操作）
  → 不存在 → 继续执行

第二层（数据库层，兜底）：
  balance_flow表唯一键: UNIQUE(ref_order_no, change_type)
  → 极端并发下两个请求都过了第一层 → 只有一个能插入成功
  → 另一个触发唯一键冲突 → catch → 返回SUCCESS
```

**含义**：财务调CreditWallet时如果不确定上次是否成功，可以安全重试。

#### 事务原子性

```
每笔余额操作的事务边界：
[BEGIN TX]
  1. 更新 user_balance（余额变动）
  2. 插入 balance_flow（变动流水，包含before/after快照）
  3. 更新订单状态（deposit_order / withdraw_order）
[COMMIT TX]

三者要么全成功，要么全失败，绝不出现"余额变了但流水没写"的情况
```

#### 事务边界铁律

```
正确做法:
  ① 获取数据（RPC调用、查询等）  ← 事务外
  ② 计算和验证                   ← 事务外
  ③ [BEGIN TX]                   ← 事务开始
  ④   DB操作（余额+流水+订单）
  ⑤ [COMMIT TX]                 ← 事务结束
  ⑥ 返回结果                     ← 事务外

错误做法（绝不允许）:
  ① [BEGIN TX]
  ②   RPC调用（如果超时，事务持续持有锁）  ← 严禁!
  ③   DB操作
  ④ [COMMIT TX]
```

**含义**：钱包RPC不会因为长事务导致DB锁超时。财务调CreditWallet的响应时间约束在毫秒级。

### 7.2 我方的约束和限制

| 约束 | 说明 | 对调用方的影响 |
|------|------|--------------|
| **余额不缓存** | 每次查余额都从DB实时读 | QueryWalletBalance响应略慢（~5ms）但保证实时准确 |
| **冻结三步模型** | Freeze → DeductFrozen/Unfreeze，不支持直接扣冻结 | 提现必须走冻结→扣除/解冻的完整流程 |
| **批量查询限100** | BatchQueryBalance入参userIds最多100个 | 超过100需分批调用 |
| **子钱包不直接转账** | center/reward/agent/anchor之间不能直接转 | 跨子钱包需要先扣再加（两次操作） |
| **币种不可运行时新增** | 新增币种需要走配置+重启 | 如果要新增币种需提前沟通 |

### 7.3 我方期望对方遵守的调用规范

**规范1：多子钱包操作模式**
```
正确: 人工加款100 BSB（center:70 + reward:30）
  Call 1: CreditWallet(orderNo="A001", walletType=CENTER, amount="70")
  Call 2: CreditWallet(orderNo="A001", walletType=REWARD, amount="30")
  → 两次调用用同一个orderNo，不同walletType

错误:
  Call 1: CreditWallet(orderNo="A001-center", walletType=CENTER, amount="70")
  Call 2: CreditWallet(orderNo="A001-reward", walletType=REWARD, amount="30")
  → 不要在orderNo里拼walletType后缀，我方用(orderNo, walletType)组合做幂等
```

**规范2：处理幂等命中的返回值**
```
第一次调用: CreditWallet(orderNo="A001", ...) → SUCCESS, newBalance="1000"
第二次调用: CreditWallet(orderNo="A001", ...) → SUCCESS, newBalance="1000" (幂等命中)

两次都返回SUCCESS，财务侧不需要区分是首次成功还是幂等命中。
newBalance可能不同（因为中间有其他操作），但不影响本次操作的正确性。
```

**规范3：不要批量循环调用写操作**
```
错误做法（可能导致性能问题）:
  for users in user_list:
      DebitWallet(userId=user, ...)  // 循环1000次

正确做法:
  逐个调用，每次检查返回值
  如需批量操作，考虑使用BatchQueryBalance先查再逐个操作
  如果确实需要批量写操作，提前沟通，我方可能需要做限流
```

---

## 第8章：IDL协调与集成排期

### 8.1 IDL文件归属与目录结构

```
common/idl/
├── ser-wallet/                  ← 我方负责
│   ├── service.thrift           (服务定义，所有method)
│   ├── wallet_c.thrift          (C端请求/响应结构)
│   ├── currency_b.thrift        (B端币种配置结构)
│   ├── wallet_rpc.thrift        (RPC领域结构，★ 与财务交互的核心)
│   ├── common.thrift            (公共结构: WalletBalanceInfo等)
│   └── enum.thrift              (枚举: WalletType, OpType, RateType等)
│
├── ser-finance/                 ← 财务方负责
│   ├── service.thrift           (服务定义)
│   ├── finance_rpc.thrift       (RPC领域结构)
│   └── ...
```

**关键说明**：
- `wallet_rpc.thrift` 是双方交互的核心IDL文件，定义了12个RPC的所有请求/响应结构
- 我方负责设计并提交到common，财务方review
- `ser-finance/` 目录需要财务方创建并提交，我方是client

### 8.2 IDL交付里程碑建议

| 周次 | 里程碑 | 我方(钱包) | 对方(财务) | 产出物 |
|------|--------|-----------|-----------|--------|
| Week 1 | IDL初稿 | 完成wallet_rpc.thrift初稿 | 完成finance_rpc.thrift初稿 | 双方IDL草稿 |
| Week 2 | 联合评审 | 提交review意见 | 提交review意见 | 修订后的IDL |
| Week 3 | 代码生成 | 运行gen_kitex.sh生成代码 | 同步生成代码 | kitex_gen/目录 |
| Week 3 | Client注册 | 在common/rpc/注册FinanceClient | 在common/rpc/注册WalletClient | RPC Client可用 |
| Week 4 | 联调环境 | 部署wallet到测试环境 | 部署finance到测试环境 | 联调就绪 |
| Week 5-6 | 联调测试 | 切换Mock→Real | 实现RPC服务端 | 端到端通过 |

### 8.3 Mock→Real切换策略

**接口抽象层设计**：

```go
// internal/external/finance.go — 接口定义
type FinanceService interface {
    GetPaymentMethods(ctx context.Context, currency, methodType string) ([]PaymentMethod, error)
    MatchAndCreateChannelOrder(ctx context.Context, req ChannelOrderReq) (*ChannelOrderResp, error)
    GetChannelOrderStatus(ctx context.Context, channelOrderNo string) (*ChannelOrderStatus, error)
    // InitiateChannelPayout — 视Q-P0-2决议决定是否需要
}

// internal/external/finance_mock.go — P2-P3阶段使用
type financeMockImpl struct{}
func (f *financeMockImpl) GetPaymentMethods(...) ([]PaymentMethod, error) {
    return []PaymentMethod{
        {MethodId: 1, MethodName: "银行转账", MethodType: 1, ...},
        {MethodId: 2, MethodName: "USDT (TRC20)", MethodType: 3, ...},
    }, nil
}

// internal/external/finance_rpc.go — P4阶段切换
type financeRpcImpl struct{}
func (f *financeRpcImpl) GetPaymentMethods(...) ([]PaymentMethod, error) {
    resp, err := rpc.FinanceClient().GetPaymentMethods(ctx, &finance.GetPaymentMethodsReq{...})
    // ...
}

// 配置驱动切换（config.yaml中一行配置）
// use_mock_finance: true  → financeMockImpl
// use_mock_finance: false → financeRpcImpl
```

**好处**：
- P2-P3阶段钱包可以独立开发和测试，不被财务阻塞
- P4切换时只需改配置，不改业务代码
- 如果财务部分接口先就绪，可以逐个接口切换

### 8.4 ETCD服务注册协调

**需要对齐的信息**：

| 项目 | 钱包侧 | 财务侧 | 状态 |
|------|--------|--------|------|
| ETCD Namespace | `ser-wallet` | `ser-finance` (待确认) | 需确认 |
| 服务端口 | `:8xxx` (待分配) | `:8xxx` (待分配) | 需确认 |
| 命名规范 | `common/pkg/consts/namesp/namesp.go` 注册 | 同上 | 需对齐 |
| 环境区分 | dev / test / staging / prod | 同上 | 需对齐 |

```go
// common/pkg/consts/namesp/namesp.go 需要新增:
const (
    EtcdFinanceService = "ser-finance"    // 财务方注册
    EtcdFinancePort    = ":8xxx"          // 端口待分配
)
```

---

## 第9章：异常场景与补偿机制共识

### 9.1 异常场景全景

```
                   充值链路异常点
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  ①获取方式 ──→ ②创建渠道订单 ──→ ③用户支付 ──→ ④入账回调     │
  │     ↓              ↓                              ↓         │
  │  O-1失败        O-2失败                       R-1失败       │
  │  场景1          场景2                         场景3         │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

                   提现链路异常点
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  ①冻结 ──→ ②提交审核 ──→ ③审核+出款 ──→ ④结果回调           │
  │              ↓                              ↓               │
  │          审核RPC失败                   R-5/R-4失败           │
  │          场景4                         场景5                │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 9.2 场景详解与双方责任

#### 场景1：GetPaymentMethods RPC失败

| 维度 | 内容 |
|------|------|
| **触发条件** | 财务服务不可用，或RPC超时 |
| **影响** | C端无法显示充值/提现方式列表 |
| **我方处理** | 返回友好提示"暂时无法获取支付方式，请稍后重试" |
| **资金影响** | **无**（用户还没操作） |
| **对方责任** | 保障服务可用性 |
| **需要约定** | 是否需要降级方案（如缓存上次成功的方式列表）？ |

#### 场景2：MatchAndCreateChannelOrder RPC失败

| 维度 | 内容 |
|------|------|
| **触发条件** | 通道匹配失败、财务RPC超时、渠道API异常 |
| **影响** | 充值订单创建失败 |
| **我方处理** | 本地充值订单标记为"失败" → C端提示"暂不可用" |
| **资金影响** | **无**（充值是先支付后入账，此时未扣余额） |
| **对方责任** | 返回区分的错误码（通道不可用 vs 金额超限 vs 系统错误） |
| **特殊情况** | 如果RPC**超时**（不确定是否成功），能否用platformOrderNo反查？ |

#### 场景3：CreditWallet RPC失败（充值入账失败）⚠️ 高风险

| 维度 | 内容 |
|------|------|
| **触发条件** | 财务确认充值成功后调钱包CreditWallet失败（DB异常、网络断连等） |
| **影响** | **用户已支付但余额未增加** → 用户资金损失感知 |
| **我方处理** | 事务回滚，余额不变，订单保持"待支付" |
| **对方责任（关键）** | 财务必须实现**重试机制**：<br>① 收到失败/超时 → 等待1秒后重试（幂等安全）<br>② 重试仍失败 → 记录异常状态<br>③ 异常状态 → 运营"人工补单"功能处理 |
| **人工补单流程** | 运营在财务B端找到异常订单 → 确认渠道确实收到款 → 手动调CreditWallet → 完成入账 |
| **需要约定** | ① 重试次数和间隔<br>② 人工补单的操作界面在财务B端还是钱包B端？<br>③ 补单是否需要额外的审核流程？ |

#### 场景4：提现冻结成功但提交审核RPC失败 ⚠️ 高风险

| 维度 | 内容 |
|------|------|
| **触发条件** | 钱包成功冻结用户余额后，提交财务审核的RPC调用失败 |
| **影响** | **用户余额被冻结但财务不知道这笔提现** → 资金"卡住" |
| **我方处理（本地补偿）** | 立即执行补偿：<br>① 解冻余额：available += (amount+fee), frozen -= (amount+fee)<br>② 更新订单状态 = 失败<br>③ 写流水记录（冻结回退）<br>④ 返回C端"提现暂不可用，请稍后重试" |
| **对方责任** | 无（因为财务根本没收到请求） |
| **设计模式** | Saga简化版——本地补偿，不需要分布式事务框架 |

#### 场景5：出款结果回调失败 ⚠️ 最高风险

| 维度 | 内容 |
|------|------|
| **触发条件** | 财务确认出款成功后调钱包DeductFrozenAmount失败，或出款失败后调UnfreezeBalance失败 |
| **影响** | **用户余额持续冻结**（出款成功但冻结未扣除 或 出款失败但冻结未解冻） |
| **这是双方共同的最高风险点** | |
| **需要约定的补偿策略** | |

**方案A（推荐）：财务侧重试**
```
财务调钱包RPC失败
  → 等待2秒
  → 重试（钱包侧幂等，安全重试）
  → 仍失败 → 等待5秒 → 再重试
  → 三次全失败 → 记录异常 → 告警通知运营
  → 运营人工处理
```

**方案B：定时对账补偿**
```
钱包定时任务（每5分钟）:
  查询所有"审核中"状态超过N小时的提现订单
  → 主动调财务查询审核结果
  → 如果已出款成功 → 本地执行DeductFrozen
  → 如果已驳回 → 本地执行Unfreeze
```

**我方倾向：A + B 结合**
- A是主路径（即时性好）
- B是兜底（防止A的重试也全失败）
- **需要双方确认此方案**

### 9.3 对账机制

| 维度 | 内容 |
|------|------|
| **对账目标** | 确保钱包余额变动流水 与 财务订单状态 完全一致 |
| **对账频率** | 日终（T+1凌晨执行） |
| **钱包提供数据** | balance_flow表：所有充值入账(DEPOSIT)和提现扣除(WITHDRAW_DEDUCT)流水 |
| **财务提供数据** | 充值订单（状态=成功）和提现订单（状态=成功/失败）清单 |
| **差异场景** | ① 财务有成功充值但钱包无对应流水（入账回调丢失）<br>② 钱包有入账流水但财务无对应成功订单（脏数据）<br>③ 提现金额不一致 |
| **差异处理** | 人工核实 → 确认后补单或调账 |

**待确认**：
1. 对账由谁主导？财务侧？还是独立对账服务？
2. 对账差异报告发给谁？格式是什么？
3. 对账数据接口是否需要额外RPC（如GetDailyFlowSummary）？还是直接查数据库？

### 9.4 监控与告警约定

| 监控项 | 阈值 | 告警对象 | 说明 |
|--------|------|---------|------|
| CreditWallet RPC成功率 | < 99.9% | 双方 | 充值入账异常 |
| DeductFrozenAmount/UnfreezeBalance成功率 | < 99.9% | 双方 | 提现结果处理异常 |
| 提现"审核中"超过4小时 | > 0笔 | 财务 | 审核流程可能卡住 |
| 充值"待支付"超过30分钟 | > N笔 | 双方 | 可能回调丢失 |
| 日终对账差异 | > 0笔 | 双方 | 数据不一致 |

---

## 第10章：建议会议议程与准备检查

### 10.1 建议会议议程（90分钟）

| 时间段 | 主题 | 目标 | 主讲方 |
|--------|------|------|--------|
| 0-10min | **双方模块概述** | 介绍各自模块定位、当前进度、团队分工 | 双方 |
| 10-30min | **充值链路走查** | 逐步确认5个阶段的模块归属和交互方式<br>解决Q3系列问题 | 钱包主讲 |
| 30-50min | **提现链路走查** | 逐步确认4个阶段的模块归属<br>**重点**：InitiateChannelPayout归属(Q-P0-2)<br>解决Q4系列问题 | 钱包主讲 |
| 50-65min | **协议约定确认** | amount类型(string vs i64)<br>walletType/opType枚举<br>错误码<br>超时/重试策略<br>回调方式 | 双方讨论 |
| 65-80min | **IDL排期+集成里程碑** | IDL交付日期(Q-P0-1)<br>Mock→Real切换时间<br>联调窗口 | 双方讨论 |
| 80-90min | **遗留问题+后续计划** | 整理未决问题清单<br>分配负责人和截止日期<br>约定下次沟通时间 | 双方 |

### 10.2 会前准备检查清单

**自检项（确保会前完成）**：

- [ ] 本文档通读一遍，标记自己不确定的点
- [ ] 5个P0问题的"我方倾向"和"理由"准备好
- [ ] 充值/提现流程图打印或投屏准备
- [ ] wallet_rpc.thrift IDL初稿准备好（可以在会上展示结构框架）
- [ ] 了解财务侧的团队成员和分工（谁负责渠道、谁负责审核、谁负责B端）
- [ ] 准备一个共享文档/表格，用于记录会议决议

### 10.3 预判对方可能问的问题及应答

#### Q: "钱包为什么不用分布式事务框架？"

**应答要点**：
- 我们用的是**本地补偿模式（Saga简化版）**，不需要重量级框架（如Seata）
- 原因：① 项目规模不需要，引入框架增加复杂度 ② 所有跨模块操作都是"一方主导"模式（先本地操作、再调外部、失败则本地补偿）③ 所有RPC接口都是幂等的，重试安全
- 具体补偿策略：提现冻结成功但审核提交失败 → 本地立即解冻

#### Q: "CAS乐观锁重试上限是多少？超过怎么办？"

**应答要点**：
- 重试上限：3次
- 超过3次 → 返回BALANCE_NOT_ENOUGH或SYSTEM_ERROR
- 实际触发概率极低：同一用户同一时刻并发操作余额的概率很小
- 即使触发了，失败是安全的（不会超扣，只是本次操作失败，用户重试即可）

#### Q: "余额为什么不缓存？查询性能怎么保证？"

**应答要点**：
- 余额是高频写的热数据，缓存一致性成本极高（每次写操作都要失效缓存）
- 不缓存 = 每次查DB → 保证实时准确（资金场景不能容忍脏读）
- 性能保障：TiDB/MySQL单行查询通过主键/索引，响应时间 < 5ms
- 只有**币种配置**（低频写、高频读）才走Cache-Aside缓存

#### Q: "如果我们RPC挂了（财务不可用），钱包怎么处理？"

**应答要点**：
- **充值方向**：财务挂了 → 用户无法看到充值方式 → 友好提示"暂不可用" → 无资金损失
- **提现方向**：审核提交RPC失败 → 本地补偿解冻 → 用户资金安全 → 提示重试
- **入账回调方向**：是财务调我们，如果我们挂了 → 需要你们重试
- **核心原则**：任何外部依赖不可用时，绝不损害用户资金安全

#### Q: "你们支持多少并发？QPS上限？"

**应答要点**：
- 当前阶段不做提前优化，先保证正确性
- 预估QPS：充值/提现各 < 100 TPS（初期用户量有限）
- CAS乐观锁在低并发下几乎无冲突
- 如果未来需要高并发，可以考虑：分库分表、Redis预扣减、热点账户拆分
- 但这些都是后续优化，当前不做

#### Q: "兑换功能和我们有关系吗？"

**应答要点**：
- **无关**。兑换是钱包内部闭环操作（VND/IDR/THB → BSB 或 反向）
- 不经过财务模块，不涉及第三方渠道
- 汇率从钱包本地exchange_rate表获取
- 事务在钱包内完成（扣源币种+加目标币种，同一事务）

### 10.4 会后跟踪模板

```
┌──────────────────────────────────────────────────────────────────┐
│ 会议日期:                                                        │
│ 参会人员:                                                        │
│                                                                  │
│ ═══ 决议事项 ═══                                                 │
│                                                                  │
│ 1. [决议内容]                                                     │
│    负责人:           截止日期:                                      │
│                                                                  │
│ 2. [决议内容]                                                     │
│    负责人:           截止日期:                                      │
│                                                                  │
│ ═══ 遗留问题 ═══                                                 │
│                                                                  │
│ 1. [问题描述]                                                     │
│    需要: [谁提供什么信息]    截止日期:                               │
│                                                                  │
│ ═══ 下次沟通 ═══                                                 │
│                                                                  │
│ 时间:                                                            │
│ 主题:                                                            │
│ 准备事项:                                                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## 附录：速查数字与关键术语

### A.1 核心数字速查

| 维度 | 数字 | 明细 |
|------|------|------|
| **双向RPC总数** | 16个 | wallet暴露12 + wallet调用4 |
| **P0必须接口** | 10个 | R-1~R-6, R-8, R-9, O-1, O-2 |
| **P0必确认问题** | 5个 | IDL时间/代付归属/充值回调/提现回调/币种精度 |
| **P1严重问题** | 5个 | 奖金归属/手续费方向/提现过滤/审核流程/USDT地址 |
| **P2重要问题** | 4个 | 返回结构/订单映射/汇率方向/现有IDL |
| **异常场景** | 5个 | 方式获取失败/渠道创建失败/入账失败/冻结后审核失败/回调失败 |
| **会议建议时长** | 90分钟 | 6个议程段 |

### A.2 关键术语对齐表

| 钱包侧术语 | 可能的财务侧术语 | 含义 |
|-----------|----------------|------|
| CreditWallet | 入账/加款 | 增加用户余额 |
| DebitWallet | 扣款/减款 | 减少用户余额 |
| FreezeBalance | 冻结 | available→frozen |
| UnfreezeBalance | 解冻/退回 | frozen→available |
| DeductFrozenAmount | 扣冻结/出款确认 | frozen减少 |
| platformOrderNo | 平台订单号/内部订单号 | 钱包生成的订单号（C/T+16位） |
| channelOrderNo | 渠道订单号/通道订单号 | 财务侧/第三方的订单号 |
| walletType | 子钱包类型 | CENTER/REWARD/AGENT/ANCHOR |
| opType | 操作类型/变动类型 | DEPOSIT/WITHDRAW/MANUAL_CREDIT等 |
| balance_flow | 余额流水/变动记录 | 每笔余额变动的完整记录 |
| CAS | 乐观锁/条件更新 | Compare And Swap |
| 最小单位 | 分/satoshi | BIGINT存储的换算单位 |

### A.3 参考文档索引

| 文档 | 路径 | 内容 | 行数 |
|------|------|------|------|
| 综合实现方案 | 总结实现/tmp-2.md | 钱包模块从0到1完整方案 | 1115 |
| RPC暴露详解 | rpc暴露/tmp-2.md | 12个暴露RPC详细设计 | 1093 |
| RPC调用详解 | rpc调用/tmp-2.md | ~11个调用RPC详细分析 | 1193 |
| 依赖关系分析 | 依赖关系/tmp-2.md | 双向依赖完整分析 | - |
| 业务流程详解 | 流程思路/c-5.md | 6条流程线详细设计 | 1127 |
| 接口统计 | 接口统计/tmp-2.md | 46接口+财务53接口参考 | 887 |
| 技术挑战 | 技术挑战/c-5.md | 12项技术挑战解决方案 | 1126 |
| 表结构设计 | 表结构设计/c-5.md | 11张表完整字段设计 | 1187 |

---

> **文档结束**
>
> 本文档共覆盖：10章 + 1附录
> - 16个跨模块RPC完整分析
> - 14个待确认问题（5个P0 + 5个P1 + 4个P2）
> - 5个异常场景与补偿策略
> - 5个预判Q&A应答
> - 1份90分钟会议议程建议
> - 1份会后跟踪模板
