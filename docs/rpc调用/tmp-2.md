# ser-wallet 对外 RPC 依赖分析（我方主动调用他方接口）

> 撰写时间：2026-02-28
> 分析视角：ser-wallet 模块主动调用工程内其他模块的哪些 RPC 接口
> 与前文关系：前文（rpc暴露/tmp-2.md）分析"我们暴露给别人"，本文分析"我们依赖别人"
> 分析依据：
> - 需求分析：/Users/mac/gitlab/z-readme/result/tmp-2.md（152 张图片提取）
> - 依赖关系：/Users/mac/gitlab/z-readme/依赖关系/tmp-2.md（谁调谁全景图）
> - 链路关系：/Users/mac/gitlab/z-readme/链路关系/tmp-2.md（每个接口的调用链路）
> - 财务结构：/Users/mac/gitlab/z-readme/财务结构/tmp-2.md（财务模块推演）
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-2.md（O-1~O-6 外调接口定义）
> - 工程实际：common/idl/ 下 23 个服务的 service.thrift 实际 IDL 定义
> - RPC Client：common/rpc/rpc_client.go（22 个已注册的 RPC 客户端工厂方法）

---

## 一、分析背景

### 1.1 "暴露"与"依赖"的对称关系

```
上一篇（rpc暴露/tmp-2.md）：
  别人 ───调用──→ 我方 12 个 RPC 接口
  我方角色：被调用方（Server）
  关注点：接口设计、幂等性、参数约定

本篇（rpc调用/tmp-2.md）：
  我方 ───调用──→ 别人的 RPC 接口
  我方角色：调用方（Client）
  关注点：对方接口是否存在、参数怎么传、失败怎么处理
```

### 1.2 为什么必须清楚对外依赖

1. **联调排期**：知道依赖谁，才能跟对方约好联调时间
2. **Mock 策略**：对方未就绪时，我方需要 mock 哪些接口才能独立开发
3. **容错设计**：对方接口挂了，我方业务链路如何降级
4. **初始化顺序**：main.go 中需要初始化哪些 RPC Client

---

## 二、项目 RPC Client 基础设施现状

### 2.1 已有的 RPC 客户端（common/rpc/rpc_client.go）

项目已注册 **22 个 RPC 客户端**，均采用 sync.Once 单例模式 + ETCD 服务发现：

```
编号  Client 工厂方法           目标服务          ser-wallet 是否需要
────────────────────────────────────────────────────────────────────
002   BUserClient()            ser-buser          ✅ 是（B端操作者查询）
003   BComClient()             ser-bcom           ⚠️ 可能（开关配置）
004   I18nClient()             ser-i18n           ⚠️ 可能（多语言）
005   IpClient()               ser-ip             ❌ 否
006   BLogClient()             ser-blog           ✅ 是（B端操作日志）
007   S3Client()               ser-s3             ❌ 否
008   UserClient()             ser-user           ✅ 是（用户信息查询）
009   ExpClient()              ser-exp            ⚠️ 可能（VIP等级）
010   ItemClient()             ser-item           ❌ 否（钱包不涉及道具）
011   AppClient()              ser-app            ❌ 否
012   KycClient()              ser-kyc            ✅ 是（KYC认证查询）
017   WebSocketClient()        ser-socket         ✅ 是（余额变更推送）
019   LiveClient()             ser-live           ❌ 否（直播调钱包，不是钱包调直播）
020   CronClient()             ser-cron           ❌ 否
021   MallClient()             ser-mall           ❌ 否
022   BagClient()              ser-bag            ❌ 否
---   FinanceClient()          ser-finance        ✅ 是 ← 需新建（不存在）
```

### 2.2 需要新增的 RPC Client

```
需新增：
  common/rpc/rpc_client.go    → 新增 FinanceClient() 工厂方法
  common/pkg/consts/namesp/   → 新增 EtcdFinanceService / EtcdFinancePort
```

ser-finance 的 IDL 目录和 kitex 生成代码尚未创建，这是联调前的阻塞项。

---

## 三、对外 RPC 依赖总览

按确定性分为三个层级：

```
ser-wallet 对外 RPC 依赖（总计 ~13 个调用点）
│
├── 确定依赖（需求文档明确记录，必须调用）
│   │
│   ├── ser-finance（4 个调用 — 尚未创建）
│   │   ├── O-1  GetPaymentMethods           获取充值/提现方式
│   │   ├── O-2  MatchAndCreateChannelOrder   充值通道匹配+创建通道订单
│   │   ├── O-3  InitiateChannelPayout        发起代付（提现出款）
│   │   └── O-4  GetChannelOrderStatus        查询通道订单状态
│   │
│   ├── ser-kyc（1 个调用 — 已存在）
│   │   └── O-5  KycQuery                     查KYC状态+获取认证姓名
│   │
│   └── ser-user（1 个调用 — 已存在）
│       └── O-6  GetUserInfoExpend            获取用户信息
│
├── 工程规范依赖（项目通用惯例，强烈建议调用）
│   │
│   ├── ser-blog（1 个调用 — 已存在）
│   │   └── O-7  AddActLog                    B端操作日志记录
│   │
│   ├── ser-buser（1 个调用 — 已存在）
│   │   └── O-8  QueryUserByIds               B端操作者名称查询
│   │
│   └── ser-socket（1 个调用 — 已存在）
│       └── O-9  WebSocket推送                余额变更实时通知
│
├── 待确认依赖（预评审可能新增/移除）
│   │
│   ├── 活动模块（1 个调用 — 可能不存在）
│   │   └── O-10 GetMatchingActivities        充值奖金活动查询
│   │
│   └── ser-i18n（1 个调用 — 已存在）
│       └── O-11 LangTransList                多语言文案翻译
│
└── 外部 HTTP 依赖（不是 RPC，但是外部调用）
    │
    └── 三方汇率 API × 3（HTTP 调用）
        └── O-EXT 定时拉取各币种对 USDT 汇率
```

---

## 四、确定依赖 — 逐个详解

### O-1: ser-finance → GetPaymentMethods（获取充值/提现方式）

#### IDL 现状

**ser-finance IDL 目录不存在。** 此接口由同事在 ser-finance 模块中实现，我方作为调用方。

#### 钱包侧调用场景

| 我方接口 | 调用时机 | 传入参数 | 期望返回 |
|---------|---------|---------|---------|
| C-2 GetDepositMethods | 用户进入充值页面 | currencyCode + type="deposit" | 该币种可用的充值方式列表 |
| C-7 GetWithdrawMethods | 用户进入提现页面 | currencyCode + type="withdraw" | 该币种可用的提现方式列表 |
| C-17 GetWithdrawLimits | 获取提现限额信息 | currencyCode + type="withdraw" | 提现方式配置（含限额/手续费） |

#### 调用时序

```
[用户 App]              [ser-wallet]                    [ser-finance]
    │ 进入充值页面         │                               │
    │──────────────────→│                               │
    │                    │  RPC: GetPaymentMethods       │
    │                    │  (currencyCode=VND,           │
    │                    │   type="deposit")             │
    │                    │──────────────────────────────→│
    │                    │                               │ 查 deposit_method 表
    │                    │                               │ 过滤: 已启用 + 匹配币种
    │                    │←──────────────────────────────│
    │                    │  返回: 充值方式列表              │
    │                    │  (名称/图标/档位/限额/排序)      │
    │←──────────────────│                               │
    │  展示充值方式列表    │                               │
```

#### 期望的返回结构（推测，需与同事对齐）

```
GetPaymentMethodsReq {
    currencyCode:  string       // VND / IDR / THB / USDT / BSB
    type:          string       // "deposit" 或 "withdraw"
}

GetPaymentMethodsResp {
    methods: list<PaymentMethodInfo>
}

PaymentMethodInfo {
    methodId:       i64         // 方式ID
    methodName:     string      // 方式名称（如"银行转账"）
    methodIcon:     string      // 图标URL
    methodType:     i32         // 方式类型（银行/电子钱包/USDT）
    quickAmounts:   list<string>  // 快捷金额档位
    minAmount:      string      // 最低金额
    maxAmount:      string      // 最高金额
    feeRate:        optional string  // 手续费率（提现用）
    dailyLimit:     optional string  // 每日限额（提现用）
    sortOrder:      i32         // 排序
    isRecommended:  bool        // 是否推荐
}
```

#### 失败处理

- 财务模块不可用 → 充值/提现页面无法加载方式列表 → 返回友好提示"暂时无法获取充值方式"
- 返回空列表 → 展示"暂无可用充值方式"

#### Mock 策略

Phase 2~3 独立开发期间，可 mock 返回固定的方式列表数据：
```go
// mock: 返回2个充值方式供开发调试
mockMethods := []PaymentMethodInfo{
    {MethodId: 1, MethodName: "Bank Transfer", MethodType: 1, ...},
    {MethodId: 2, MethodName: "USDT (TRC20)", MethodType: 3, ...},
}
```

依据：[需求 5.3] 充值方式配置字段；链路关系/tmp-2.md C-2/C-7 章节。

---

### O-2: ser-finance → MatchAndCreateChannelOrder（通道匹配+创建通道订单）

#### 钱包侧调用场景

| 我方接口 | 调用时机 | 说明 |
|---------|---------|------|
| C-3 CreateDepositOrder | 用户确认充值，点击"去充值" | 钱包先校验（重复转账/金额/活动），然后调财务匹配通道 |

#### 调用时序

```
[用户 App]              [ser-wallet]                    [ser-finance]
    │ 确认充值(VND,      │                               │
    │  500000, 银行转账)  │                               │
    │──────────────────→│                               │
    │                    │  内部校验:                     │
    │                    │  ├ 重复转账检测                 │
    │                    │  ├ 金额范围校验                 │
    │                    │  └ 奖金活动校验                 │
    │                    │                               │
    │                    │  RPC: MatchAndCreateChannelOrder│
    │                    │  (depositMethodId=1,           │
    │                    │   currencyCode=VND,            │
    │                    │   amount=500000,               │
    │                    │   platformOrderNo=C+16位)      │
    │                    │──────────────────────────────→│
    │                    │                               │ 轮询算法选通道
    │                    │                               │ 创建通道侧订单
    │                    │                               │ 调三方通道 API
    │                    │←──────────────────────────────│
    │                    │  返回: channelOrderNo          │
    │                    │  + paymentInfo(支付URL/QR码)   │
    │                    │                               │
    │                    │  创建平台充值订单(状态=待支付)    │
    │                    │                               │
    │←──────────────────│                               │
    │  跳转支付页面       │                               │
```

#### 期望的参数和返回

```
MatchAndCreateChannelOrderReq {
    depositMethodId:   i64      // 充值方式ID（从 O-1 获取的）
    currencyCode:      string   // 币种
    amount:            string   // 充值金额
    platformOrderNo:   string   // 我方平台订单号（C+16位）
    userId:            i64      // 用户ID
}

MatchAndCreateChannelOrderResp {
    channelOrderNo:  string     // 通道侧订单号
    channelName:     string     // 匹配到的通道名称
    paymentType:     i32        // 支付类型（跳转/QR码/地址/...）
    paymentUrl:      optional string  // 支付跳转URL
    qrCodeUrl:       optional string  // QR码图片URL
    bankAccount:     optional string  // 收款银行账号
    walletAddress:   optional string  // 收款钱包地址（USDT）
    expireTime:      i64        // 支付过期时间戳
}
```

#### 失败处理

这是**充值链路中最关键的外部调用**：
- 通道匹配失败（所有通道不可用）→ 返回"当前充值方式暂时不可用，请稍后再试"
- 通道创建订单超时 → 重试 1 次，仍失败则返回错误
- **不要在通道匹配失败时就创建平台订单**，避免产生无效订单

#### Mock 策略

```go
// mock: 返回一个虚拟的支付跳转URL
mockResp := MatchAndCreateChannelOrderResp{
    ChannelOrderNo: "CH" + generateRandom16(),
    PaymentType:    1, // 跳转
    PaymentUrl:     "https://mock-payment.example.com/pay?order=xxx",
    ExpireTime:     time.Now().Add(30 * time.Minute).Unix(),
}
```

依据：[需求 3.4.1] "点击去充值后生成订单匹配通道"；[需求 5.2] 轮询算法；链路关系/tmp-2.md C-3 章节。

---

### O-3: ser-finance → InitiateChannelPayout（发起代付/出款）

#### 钱包侧调用场景

| 调用时机 | 说明 |
|---------|------|
| 提现审核全部通过后 | 风控审核通过 → 财务审核通过 → 发起代付 |

#### 调用归属的关键疑问

**这个调用的归属存在不确定性：**

```
方案 A（大概率）：
  提现审核在财务 B 端完成 → 财务模块自己发起代付 → 钱包不调此接口
  ──→ 钱包只在创建提现订单时冻结余额，后续交给财务处理

方案 B（小概率）：
  提现审核通过后财务通知钱包 → 钱包发起代付请求 → 钱包调此接口
  ──→ 钱包需要额外维护提现状态流转
```

**根据财务结构/tmp-2.md 的分析，方案 A 的可能性更大**：提现审核和出款流程完全在财务 B 端完成，财务内部调通道发起代付，成功后通过 RPC 回调钱包的 DeductFrozenAmount。

#### 不论哪种方案，我方需要了解的参数

```
InitiateChannelPayoutReq {
    withdrawOrderNo:   string   // 提现订单号（T+16位）
    withdrawMethodId:  i64      // 提现方式ID
    currencyCode:      string   // 币种
    amount:            string   // 出款金额
    accountInfo: {              // 收款账户信息
        holderName:    string   // 持有人姓名
        bankName:      optional string  // 银行名称
        bankAccount:   optional string  // 银行账号
        walletAddress: optional string  // USDT钱包地址
    }
}

InitiateChannelPayoutResp {
    channelOrderNo:  string     // 通道侧订单号
    status:          i32        // processing/success/failed
}
```

#### Mock 策略

如果确认是方案 A，此接口我方不调用，无需 mock。
如果是方案 B，mock 返回 `status=processing`，后续通过财务回调模拟出款结果。

依据：[需求 5.5] 提现流程状态流转；财务结构/tmp-2.md §5.4 rpc_provider.go。

**此项为预评审待确认事项。**

---

### O-4: ser-finance → GetChannelOrderStatus（查询通道订单状态）

#### 钱包侧调用场景

| 我方接口 | 调用时机 | 说明 |
|---------|---------|------|
| C-16 GetDepositOrderStatus | 用户查询充值订单状态 | 如果本地状态仍是"待支付"，可能需要主动查通道确认 |
| 定时任务（补偿查询） | 回调可能丢失时的主动轮询 | 补偿机制：定时查通道状态，如果通道已成功但我方未入账 |

#### 期望的参数和返回

```
GetChannelOrderStatusReq {
    channelOrderNo:  string     // 通道侧订单号
}

GetChannelOrderStatusResp {
    status:       i32           // 处理中/成功/失败
    updateTime:   i64           // 状态更新时间
    actualAmount: optional string  // 实际到账金额（可能与请求金额不同）
}
```

#### 调用频率

低频。主要用于：
1. 用户主动刷新充值订单状态时（C-16）
2. 定时补偿任务发现超时订单时

#### Mock 策略

```go
// mock: 返回处理中状态
mockResp := GetChannelOrderStatusResp{
    Status:     1, // processing
    UpdateTime: time.Now().Unix(),
}
```

依据：财务结构/tmp-2.md §5.4 rpc_provider.go 的 GetChannelOrderStatus 方法。

---

### O-5: ser-kyc → KycQuery（查询 KYC 认证状态）

#### IDL 现状

**已存在。** 文件路径：`common/idl/ser-kyc/service.thrift`
RPC Client 已注册：`common/rpc/rpc_client.go` → `KycClient()`

#### 钱包侧调用场景

| 我方接口 | 调用时机 | 目的 |
|---------|---------|------|
| C-7 GetWithdrawMethods | 用户进入提现页面 | 根据 KYC 状态过滤可用提现方式 |
| C-8 CreateWithdrawOrder | 用户提交提现申请 | 校验法币提现必须已通过 KYC |
| C-10 SaveWithdrawAccount | 用户保存提现账户 | 获取 KYC 认证姓名做一致性校验 |

#### 实际可用的接口

```thrift
// 已有的 KycQuery 方法
KycQueryReq {
    uid: optional i64    // 用户 UID（注意是 uid 不是 userId）
}

KycQueryResp {
    kycInfo: optional UserKycInfo
}

UserKycInfo {
    id:           i64
    uid:          i64
    name:         string      // ★ KYC认证姓名 — 提现账户校验用
    country:      string      // 国家
    phone:        string      // 手机号
    idType:       i32         // 证件类型
    idNumber:     string      // 证件号码
    status:       i32         // ★ 认证状态 — 决定可用提现方式
    submitCount:  i32         // 提交次数
    createAt:     i64
    updateAt:     i64
}
```

#### 调用时序（以提现方式过滤为例）

```
[用户 App]              [ser-wallet]                    [ser-kyc]
    │ 进入提现页面         │                               │
    │──────────────────→│                               │
    │                    │  RPC: KycQuery(uid=用户UID)    │
    │                    │──────────────────────────────→│
    │                    │←──────────────────────────────│
    │                    │  返回: kycInfo                 │
    │                    │  {status=已认证, name="张三"}   │
    │                    │                               │
    │                    │  内部过滤逻辑:                  │
    │                    │  ├ 未认证 → 只保留 USDT 提现    │
    │                    │  └ 已认证 → 保留全部提现方式     │
    │                    │                               │
    │                    │  RPC: GetPaymentMethods(...)   │
    │                    │─────────────────────→ ser-finance
    │                    │                               │
    │                    │  合并: KYC过滤 + 充值路由规则    │
    │←──────────────────│                               │
    │  展示过滤后的提现方式│                               │
```

#### 关键注意点

1. **uid vs userId**：ser-kyc 的 KycQuery 使用 `uid`（业务编号），不是 `userId`（自增主键）。我方在调用前需要确认用 uid 还是 userId，可能需要先从 ser-user 获取 uid
2. **name 字段用途**：KYC 认证姓名用于提现账户持有人校验（忽略大小写+多余空格后比对）
3. **status 含义**：需要确认状态值枚举（未提交/审核中/已通过/已拒绝）

#### Mock 策略

```go
// mock: 返回已认证状态
mockKycInfo := UserKycInfo{
    Uid:    userUid,
    Name:   "Test User",
    Status: 3, // 假设3=已认证（需确认枚举值）
}
```

依据：[需求 3.8.1] "KYC 与提现关系"；[需求 3.8.4] "账户持有人名称必须与 KYC 认证姓名一致"。

---

### O-6: ser-user → GetUserInfoExpend（获取用户信息）

#### IDL 现状

**已存在。** 文件路径：`common/idl/ser-user/service.thrift` + `user_rpc.thrift`
RPC Client 已注册：`common/rpc/rpc_client.go` → `UserClient()`

#### 钱包侧调用场景

| 我方接口 | 调用时机 | 目的 |
|---------|---------|------|
| C-8 CreateWithdrawOrder | 用户提交提现申请 | 获取用户基本信息（国家码、账户类型等） |
| 钱包账户初始化 | 用户首次进入钱包 | 根据用户国家码确定默认币种 |

#### 实际可用的接口

```thrift
// 获取单个用户信息
GetUserInfoReq {
    userId: i64 (api.query="userId", api.body="userId")
    uid:    i64 (api.query="uid", api.body="uid")
}

UserInfoExpendResp {
    userId:      i64       // 用户ID
    uid:         i64       // 业务UID
    nickname:    string    // 昵称
    mobile:      string    // 手机号
    email:       string    // 邮箱
    status:      i32       // 账号状态（正常/冻结/封禁）
    accountType: i32       // 账号类型（游客/正式）
    avatar:      string    // 头像URL
    exp:         i64       // 经验值
    level:       i32       // 用户等级
    totalFans:   i32       // 粉丝数
    countryCode: string    // ★ 国家码 — 用于确定默认币种
    createAt:    i64       // 注册时间
    gender:      i32       // 性别
}
```

另有批量查询方法 `BatchGetUserInfo`，如果需要批量操作可用。

#### 关键使用场景

**场景 1：提现姓名校验**
```
提现时：
  ├ 先调 ser-kyc.KycQuery → 获取 kycInfo.name（认证姓名）
  ├ 再调 ser-user.GetUserInfoExpend → 获取用户基本信息
  └ 校验：提现账户持有人姓名 ≈ KYC 认证姓名
```

**场景 2：钱包初始化时确定默认币种**
```
用户首次进入钱包：
  ├ 调 ser-user.GetUserInfoExpend → 获取 countryCode
  ├ 根据 countryCode 映射默认币种：
  │   VN → VND
  │   ID → IDR
  │   TH → THB
  │   其他 → USDT
  └ 初始化对应币种的钱包账户
```

**场景 3：账号状态校验**
```
任何资金操作前：
  ├ 检查用户 status
  └ 如果账号已冻结/封禁 → 拒绝操作
```

#### Mock 策略

```go
// mock: 返回测试用户信息
mockUser := UserInfoExpendResp{
    UserId:      testUserId,
    Uid:         10001,
    Nickname:    "TestUser",
    Status:      1, // 正常
    AccountType: 1, // 正式
    CountryCode: "VN",
}
```

依据：链路关系/tmp-2.md C-8 章节；[需求 3.2] 币种与国家码的关联。

---

## 五、工程规范依赖 — 逐个详解

以下接口虽然需求文档未显式提及，但从项目工程规范和通用需求可推导出必须调用。

### O-7: ser-blog → AddActLog（B 端操作日志记录）

#### IDL 现状

**已存在。** `common/idl/ser-blog/service.thrift`
RPC Client 已注册：`BLogClient()`
**特殊：oneway 调用（发了就走，不等响应）**

#### 钱包侧调用场景

B 端（币种配置模块）的所有写操作都应该记录操作日志：

| B端操作 | model | page | action |
|---------|-------|------|--------|
| B-2 EditCurrency | "钱包管理" | "币种配置" | "编辑币种: {currencyCode}" |
| B-3 ToggleCurrencyStatus | "钱包管理" | "币种配置" | "启用/禁用币种: {currencyCode}" |
| B-4 SetBaseCurrency | "钱包管理" | "币种配置" | "设置基准币种: {currencyCode}" |

#### 调用方式

```thrift
// oneway 语法 — 异步发送，不阻塞业务
oneway void AddActLog(1: ActAddReq req)

ActAddReq {
    backend:    string    // "钱包管理后台"
    model:      string    // "币种配置"
    page:       string    // "币种列表" / "基准币种设置"
    action:     string    // 具体操作描述
    isSuccess:  i32       // 1成功 / 0失败
}
```

#### 关键特性

1. **oneway 不阻塞**：AddActLog 是 oneway 调用，即使 ser-blog 不可用也不会影响钱包业务
2. **所有 B 端写操作都应记录**：这是项目审计合规的通用要求
3. **IP 和操作者信息**：由 ser-blog 内部从 context 提取，我方不需要传

依据：参考 ser-buser/ser-live 等已有模块的 B 端操作均调用 AddActLog。

---

### O-8: ser-buser → QueryUserByIds（B 端操作者名称查询）

#### IDL 现状

**已存在。** `common/idl/ser-buser/service.thrift`
RPC Client 已注册：`BUserClient()`

#### 钱包侧调用场景

B 端列表接口中需要显示"操作者"名称时：

| B端操作 | 场景 |
|---------|------|
| B-5 GetExchangeRateLogs | 汇率日志列表中如有手动触发记录，需显示操作者 |
| B-1 GetCurrencyList | 币种列表中显示"最后修改人" |

#### 实际可用的接口

```thrift
// 批量查后台用户名称
QueryUserByIds(RpcSysUserReq req) → map<i64, string>
// 返回: {userId1: "admin", userId2: "operator1"}

// 单个查后台用户ID
QueryUserIdByName(string name) → i64
```

#### 调用方式

```go
// 示例：查操作者名称
nameMap, _ := rpc.BUserClient().QueryUserByIds(ctx, &RpcSysUserReq{
    UserIds: operatorIds,
})
// nameMap = {1001: "张三", 1002: "李四"}
```

#### 关键特性

这个依赖不是"非用不可"，但项目惯例是 B 端列表接口都会带上操作者名称。

---

### O-9: ser-socket → WebSocket 推送（余额变更实时通知）

#### IDL 现状

**已存在但接口几乎为空。** `common/idl/ser-socket/service.thrift` 目前只有 `OnBlackList()` 一个方法。

#### 钱包侧需求

余额变更后需要实时推送到用户 App：
- 充值成功入账 → 推送"到账 +500,000 VND"
- 提现成功 → 推送"提现已到账"
- 兑换完成 → 推送余额更新

#### 当前状态分析

```
ser-socket 当前状态：
  ├ service.thrift 几乎为空（只有 OnBlackList）
  ├ 但 RPC Client 已注册（WebSocketClient）
  └ 说明 WebSocket 基础设施存在，但推送方法可能在开发中
```

#### 推送方式推测

根据项目 WebSocket 基础设施存在的事实，推送方式可能是：
1. 调 ser-socket 的 RPC 方法，由 ser-socket 转发给前端（最可能）
2. 直接通过 Redis Pub/Sub 发消息（备选）

**此项需进一步了解 ser-socket 的推送机制后确认。**

#### Mock 策略

Phase 2~3 开发期间可跳过推送，在业务逻辑中预留 Hook 点：
```go
// 预留推送 Hook，后续填充实现
func notifyBalanceChange(ctx context.Context, userId int64, event BalanceChangeEvent) {
    // TODO: 调 ser-socket 推送余额变更通知
}
```

---

## 六、待确认依赖

### O-10: 活动模块 → GetMatchingActivities（充值奖金活动查询）

#### 需求背景

[需求 3.6] 充值时用户可以选择奖金活动，获得额外赠送。C-15 GetBonusActivities 接口需要获取匹配当前币种和金额的活动列表。

#### 不确定性

```
不确定点：
  ├ 活动模块是否已存在？ → common/idl/ 下没有 ser-activity 目录
  ├ 活动数据由谁管理？   → 可能在 ser-finance 内部，也可能是独立服务
  └ 活动数据结构是什么？ → 需求文档只给了示例（3个活动），未定义完整结构
```

#### 两种可能

**方案 A：活动数据在 ser-finance 内管理**
```
钱包 → ser-finance.GetPaymentMethods 时一并返回关联的活动列表
不需要额外调用
```

**方案 B：独立的活动模块**
```
钱包 → activity.GetMatchingActivities(currencyCode, amount)
返回：活动列表（门槛金额、奖金百分比、稽核倍数）
```

#### Mock 策略

Phase 2 开发期间，活动数据可以写死或从配置读取：
```go
// mock: 返回3个固定活动
mockActivities := []BonusActivity{
    {Threshold: "100000", BonusPercent: "5%",  AuditMultiplier: "8x"},
    {Threshold: "500000", BonusPercent: "10%", AuditMultiplier: "10x"},
    {Threshold: "1000000", BonusPercent: "15%", AuditMultiplier: "12x"},
}
```

**此项为预评审待确认事项。**

---

### O-11: ser-i18n → LangTransList（多语言文案翻译）

#### IDL 现状

**已存在。** `common/idl/ser-i18n/service.thrift`
RPC Client 已注册：`I18nClient()`

#### 可能的调用场景

```thrift
// 已有方法
LangTransOne(i64 reqKey) → string               // 单个翻译
LangTransList(list<i64> reqKeys) → map<i64, string>  // 批量翻译
```

钱包模块如果需要返回多语言文案（如错误提示、操作类型名称、币种名称），可调此接口。

#### 当前判断

**大概率不需要直接调用。** 多语言翻译通常由前端/网关层处理，后端接口返回 key 或枚举值，前端自行翻译。除非产品明确要求后端返回翻译后的文案。

**暂标记为"待观察"，进入编码后根据实际需要决定。**

---

## 七、外部 HTTP 依赖（非 RPC）

### O-EXT: 三方汇率 API × 3

#### 需求背景

[需求 4.5.2] ser-wallet 需要从 3 个三方汇率 API 定时拉取各币种对 USDT 的实时汇率。

#### 调用方式

```
定时任务 T-1 ExchangeRateUpdate
  ├→ HTTP GET: 汇率API-1 (如 exchangeratesapi.io)
  ├→ HTTP GET: 汇率API-2 (如 coinmarketcap)
  ├→ HTTP GET: 汇率API-3 (如 binance)
  │   （3个并行调用，提高效率+容错）
  │
  ├→ 计算实时汇率 = 三者平均值
  │    → 某个 API 失败：用其余 2 个的平均值
  │    → 2 个以上失败：本轮跳过，保持旧汇率
  └→ 比较偏差，触发平台汇率更新
```

#### 关键设计点

1. **不走 RPC，走 HTTP**：三方 API 是外部互联网服务，通过 HTTP Client 调用
2. **容错设计**：3 个 API 互为备份，任一失败不影响整体
3. **频率可配**：通过配置控制定时拉取间隔
4. **API Key 管理**：三方 API 的 Key 通过配置中心或环境变量管理

#### 待确认项

| 事项 | 说明 |
|------|------|
| 使用哪 3 个 API | 需产品/运营确认数据源 |
| API Key 从哪获取 | 需运维确认密钥管理方式 |
| 调用频率 | 需产品确认（每分钟/每5分钟/每15分钟） |
| BSB 汇率 | BSB 锚定价（10 BSB = 1 USDT）是否固定，还是也需要从外部获取 |

---

## 八、调用全景图

### 8.1 按业务链路看调用

```
充值链路（我方主动调外部 = 2~3 次 RPC）
════════════════════════════════════════════════════════
用户打开充值页面
  └→ [O-1] ser-finance.GetPaymentMethods ← 获取充值方式
  └→ [O-10?] activity.GetMatchingActivities ← 获取奖金活动（待确认）

用户确认充值
  └→ [O-2] ser-finance.MatchAndCreateChannelOrder ← 通道匹配

用户查充值状态
  └→ [O-4] ser-finance.GetChannelOrderStatus ← 查通道状态（补偿用）


提现链路（我方主动调外部 = 3~4 次 RPC）
════════════════════════════════════════════════════════
用户打开提现页面
  └→ [O-5] ser-kyc.KycQuery ← 查KYC状态
  └→ [O-1] ser-finance.GetPaymentMethods ← 获取提现方式

用户提交提现
  └→ [O-5] ser-kyc.KycQuery ← 再次校验KYC
  └→ [O-6] ser-user.GetUserInfoExpend ← 获取用户信息

用户保存提现账户
  └→ [O-5] ser-kyc.KycQuery ← 获取KYC认证姓名做校验


兑换链路（我方主动调外部 = 0 次 RPC）
════════════════════════════════════════════════════════
纯内部操作，不依赖任何外部模块。


B端操作（我方主动调外部 = 1~2 次 RPC / 每次操作）
════════════════════════════════════════════════════════
每次 B 端写操作
  └→ [O-7] ser-blog.AddActLog ← 记录操作日志（oneway，不阻塞）
  └→ [O-8] ser-buser.QueryUserByIds ← 列表接口查操作者名称


定时任务（外部 HTTP = 3 次 / 每轮）
════════════════════════════════════════════════════════
汇率更新任务
  └→ [O-EXT] 三方汇率API × 3 ← HTTP并行调用
```

### 8.2 按目标服务看调用

```
                    ┌──────────────────────────────────────────────┐
                    │              ser-wallet                       │
                    │                                              │
                    │  充值         提现          B端       定时     │
                    │  ┌───┐      ┌───┐        ┌───┐     ┌───┐   │
                    │  │C-2│      │C-7│        │B-2│     │T-1│   │
                    │  │C-3│      │C-8│        │B-3│     └─┬─┘   │
                    │  └─┬─┘      │C-10│       │B-4│       │     │
                    │    │        └─┬─┘        └─┬─┘       │     │
                    └────┼──────────┼────────────┼─────────┼─────┘
                         │          │            │         │
              ┌──────────┘          │            │         │
              │    ┌────────────────┘            │         │
              │    │    ┌───────────────────────┘         │
              │    │    │                                  │
              ▼    ▼    ▼                                  ▼
    ┌──────────────────────┐                     ┌──────────────┐
    │     ser-finance       │                     │  三方汇率API  │
    │   （尚未创建 ❌）      │                     │  × 3 (HTTP)  │
    │                      │                     └──────────────┘
    │  O-1 GetPaymentMethods│
    │  O-2 MatchChannel     │
    │  O-3 Payout (待确认)  │
    │  O-4 GetChannelStatus │
    └──────────────────────┘
              │
              │ (财务侧也需要)
              │
    ┌─────────┴──────────┐   ┌──────────────┐   ┌──────────────┐
    │     ser-kyc         │   │  ser-user     │   │  ser-blog    │
    │   （已存在 ✅）      │   │ （已存在 ✅）  │   │ （已存在 ✅） │
    │                    │   │              │   │              │
    │  O-5 KycQuery      │   │ O-6 GetUser  │   │ O-7 AddActLog│
    │  (KYC状态+姓名)     │   │ InfoExpend   │   │ (oneway)     │
    └────────────────────┘   └──────────────┘   └──────────────┘

    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
    │  ser-buser    │   │  ser-socket   │   │  活动模块     │
    │ （已存在 ✅）  │   │ （已存在 ✅）  │   │ （不存在 ❌）  │
    │              │   │              │   │              │
    │ O-8 QueryUser│   │ O-9 WS推送   │   │ O-10 活动查询 │
    │ ByIds        │   │ (余额通知)    │   │ (待确认)      │
    └──────────────┘   └──────────────┘   └──────────────┘
```

---

## 九、各依赖的就绪状态和 Mock 策略

### 9.1 就绪状态汇总

| 编号 | 目标服务 | 方法 | IDL存在 | Client存在 | 服务运行 | 我方可调 |
|------|---------|------|---------|-----------|---------|---------|
| O-1 | ser-finance | GetPaymentMethods | ❌ | ❌ | ❌ | 需 mock |
| O-2 | ser-finance | MatchAndCreateChannelOrder | ❌ | ❌ | ❌ | 需 mock |
| O-3 | ser-finance | InitiateChannelPayout | ❌ | ❌ | ❌ | 需 mock（且调用归属待确认） |
| O-4 | ser-finance | GetChannelOrderStatus | ❌ | ❌ | ❌ | 需 mock |
| O-5 | ser-kyc | KycQuery | ✅ | ✅ | ✅ | **可直接调用** |
| O-6 | ser-user | GetUserInfoExpend | ✅ | ✅ | ✅ | **可直接调用** |
| O-7 | ser-blog | AddActLog | ✅ | ✅ | ✅ | **可直接调用** |
| O-8 | ser-buser | QueryUserByIds | ✅ | ✅ | ✅ | **可直接调用** |
| O-9 | ser-socket | WebSocket推送 | ⚠️ 接口缺 | ✅ | ✅ | 需等接口完善 |
| O-10 | 活动模块 | GetMatchingActivities | ❌ | ❌ | ❌ | 需 mock |
| O-11 | ser-i18n | LangTransList | ✅ | ✅ | ✅ | 待观察 |
| O-EXT | 三方API | HTTP GET | — | — | — | 需确认 API 源 |

### 9.2 Mock 策略矩阵

```
Phase 2（核心能力层）：
═══════════════════════════════════════════════════════
  真调：O-5 ser-kyc, O-6 ser-user, O-7 ser-blog, O-8 ser-buser
  → 这4个服务已存在且稳定运行，可直接调用

  Mock：无（Phase 2 的接口不涉及财务调用）
  → Phase 2 做的是钱包核心余额操作+币种配置，不依赖财务

Phase 3（C端功能层 — 独立部分）：
═══════════════════════════════════════════════════════
  不需要外部调用的功能（可先做）：
    C-1  GetWalletOverview        ← 纯内部
    C-5  GetExchangePreview       ← 纯内部
    C-6  CreateExchange           ← 纯内部
    C-9  GetWithdrawAccount       ← 纯内部
    C-11 GetTransactionList       ← 纯内部
    C-12 GetTransactionDetail     ← 纯内部
    C-13 GetCurrentRate           ← 纯内部
    C-19 GetAvailableCurrencies   ← 纯内部

Phase 3（C端功能层 — 依赖财务部分）：
═══════════════════════════════════════════════════════
  Mock 财务：O-1, O-2, O-4
    C-2  GetDepositMethods        ← mock 充值方式列表
    C-3  CreateDepositOrder       ← mock 通道匹配返回
    C-16 GetDepositOrderStatus    ← mock 通道状态返回

  Mock 财务 + 真调 KYC：O-1, O-5
    C-7  GetWithdrawMethods       ← mock 提现方式 + 真调 KYC
    C-8  CreateWithdrawOrder      ← 真调 KYC + 真调 User
    C-10 SaveWithdrawAccount      ← 真调 KYC

Phase 4（联调验证层）：
═══════════════════════════════════════════════════════
  全部真调，不再 mock
  与财务模块联调：O-1 ~ O-4 全部替换为真实调用
```

---

## 十、开发阶段需要做的 RPC Client 准备工作

### 10.1 已有 Client 的直接使用

以下 Client 在 `common/rpc/rpc_client.go` 中已注册，ser-wallet 的 main.go 初始化后可直接使用：

```go
// ser-wallet/main.go 中的 RPC Client 初始化
func initRpcClients() {
    // 已有的，直接用
    _ = rpc.KycClient()     // ser-kyc：KYC查询
    _ = rpc.UserClient()    // ser-user：用户信息
    _ = rpc.BLogClient()    // ser-blog：操作日志
    _ = rpc.BUserClient()   // ser-buser：B端用户
    // _ = rpc.WebSocketClient()  // ser-socket：推送（等接口完善）
    // _ = rpc.I18nClient()       // ser-i18n：多语言（待观察）
}
```

### 10.2 需要新建的 Client

```
需新建 FinanceClient：
─────────────────────────────────────────────────

1. 同事方创建 IDL：
   common/idl/ser-finance/service.thrift
   common/idl/ser-finance/finance_rpc.thrift

2. 生成 Kitex 代码：
   gen_kitex.sh → common/pkg/kitex_gen/ser_finance/

3. 注册命名空间常量：
   common/pkg/consts/namesp/namesp.go
     + EtcdFinanceService = "ser-finance"
     + EtcdFinancePort    = ":8XXX"

4. 注册 RPC Client 工厂：
   common/rpc/rpc_client.go
     + FinanceClient() financeservice.Client { ... }

5. 注册 go.work：
   go.work → use ./ser-finance
```

### 10.3 Mock 层设计建议

为了在 ser-finance 未就绪时独立开发，建议在 ser-wallet 内部做一层调用抽象：

```go
// internal/external/finance.go — 财务模块调用抽象层
type FinanceService interface {
    GetPaymentMethods(ctx context.Context, currencyCode, methodType string) ([]PaymentMethod, error)
    MatchAndCreateChannelOrder(ctx context.Context, req ChannelOrderReq) (*ChannelOrderResp, error)
    GetChannelOrderStatus(ctx context.Context, channelOrderNo string) (*ChannelOrderStatus, error)
}

// internal/external/finance_rpc.go — 真实 RPC 实现
type financeRpcImpl struct{}
func (f *financeRpcImpl) GetPaymentMethods(...) { /* 调 rpc.FinanceClient() */ }

// internal/external/finance_mock.go — Mock 实现
type financeMockImpl struct{}
func (f *financeMockImpl) GetPaymentMethods(...) { /* 返回固定数据 */ }

// main.go 中通过配置切换
if config.UseMockFinance {
    financeSer = &financeMockImpl{}
} else {
    financeSer = &financeRpcImpl{}
}
```

同理可为其他未就绪的依赖（活动模块、WebSocket 推送）做类似抽象。

---

## 十一、联调前待对齐事项

### 11.1 与 ser-finance 同事对齐（P0）

| 序号 | 事项 | 影响 |
|------|------|------|
| 1 | **GetPaymentMethods 的返回结构** | 我方 C-2/C-7 的前端展示完全依赖此结构 |
| 2 | **MatchAndCreateChannelOrder 的入参出参** | 我方 C-3 的订单创建和支付跳转依赖此接口 |
| 3 | **InitiateChannelPayout 的调用归属** | 由谁发起代付？我方还是财务模块自己？ |
| 4 | **财务 IDL 的交付时间** | 决定我方何时从 mock 切换为真实调用 |
| 5 | **充值回调流程** | 通道回调到财务后，财务如何通知钱包入账（直接调 CreditWallet？还是需要钱包监听？） |

### 11.2 与 ser-kyc / ser-user 确认（P1）

| 序号 | 事项 | 影响 |
|------|------|------|
| 6 | **KycQuery 用 uid 还是 userId** | 我方调用时传参不同 |
| 7 | **KYC status 枚举值定义** | 0=未提交/1=审核中/2=已拒绝/3=已通过？需确认 |
| 8 | **用户 countryCode 与币种的映射关系** | 钱包初始化时需要此映射 |

### 11.3 通用确认项（P2）

| 序号 | 事项 | 影响 |
|------|------|------|
| 9 | **WebSocket 推送机制** | 余额变更后如何通知前端 |
| 10 | **活动模块归属** | 奖金活动数据从哪获取 |
| 11 | **三方汇率 API 选择** | 用哪 3 个 API、Key 从哪来 |

---

## 十二、总结

### 12.1 数量统计

| 分类 | 数量 | 说明 |
|------|------|------|
| 确定依赖（需求明确） | 6 | 4个 ser-finance + 1个 ser-kyc + 1个 ser-user |
| 工程规范依赖 | 3 | 1个 ser-blog + 1个 ser-buser + 1个 ser-socket |
| 待确认依赖 | 2 | 1个活动模块 + 1个 ser-i18n |
| 外部 HTTP | 1 | 三方汇率API（3个并行） |
| **合计** | **~12** | |

### 12.2 核心结论

1. **最大阻塞项是 ser-finance**。4 个核心调用全部指向一个尚未创建的服务，IDL 不存在、Client 不存在、服务不存在。这意味着充值和提现的完整链路在 ser-finance 就绪前无法联调。

2. **ser-kyc 和 ser-user 已就绪**。提现相关的 KYC 校验和用户信息获取可以直接调用，不需要 mock。

3. **兑换链路完全不依赖外部**。这是唯一不需要任何外部 RPC 调用的业务链路，可以最先端到端跑通。

4. **Mock 层设计是独立开发的关键**。建议对 ser-finance 的 4 个调用做接口抽象 + Mock 实现，Phase 2~3 用 Mock 跑通内部逻辑，Phase 4 联调时切换为真实调用。

5. **工程规范依赖（日志/操作者/推送）不阻塞核心功能**。但为了代码合规，建议从 Phase 2 开始就接入 AddActLog。

### 12.3 与前文（rpc暴露/tmp-2.md）的对称关系

```
暴露给别人：12 个 RPC（6写 + 6读）
  └ 我方是 Server，定义 IDL，实现处理逻辑

依赖别人的：~12 个调用点（6确定 + 3规范 + 2待确认 + 1HTTP）
  └ 我方是 Client，按对方 IDL 调用

双向关系最复杂的：ser-finance
  └ 我方暴露 6 个 RPC 给它（入账/扣款/冻结/解冻/扣冻结/查余额）
  └ 我方调用它 4 个 RPC（支付方式/通道匹配/代付/状态查询）
  └ 合计 10 个 RPC 调用横跨两个模块
```

---

## 附录 A：对外 RPC 调用速查卡

```
编号  目标服务       方法                        调用场景           状态
──────────────────────────────────────────────────────────────────────────
O-1   ser-finance   GetPaymentMethods           充值/提现方式获取  ❌需mock
O-2   ser-finance   MatchAndCreateChannelOrder   充值通道匹配      ❌需mock
O-3   ser-finance   InitiateChannelPayout        发起代付          ❌需mock+待确认
O-4   ser-finance   GetChannelOrderStatus        查通道订单状态    ❌需mock
O-5   ser-kyc       KycQuery                     KYC状态+姓名      ✅可直接调
O-6   ser-user      GetUserInfoExpend            用户信息          ✅可直接调
O-7   ser-blog      AddActLog (oneway)           B端操作日志       ✅可直接调
O-8   ser-buser     QueryUserByIds               操作者名称        ✅可直接调
O-9   ser-socket    WebSocket推送                余额变更通知      ⚠️待接口完善
O-10  活动模块       GetMatchingActivities        奖金活动查询      ❌不存在+待确认
O-11  ser-i18n      LangTransList                多语言翻译        ⚠️待观察
O-EXT 三方API       HTTP GET × 3                 汇率拉取          ⚠️待确认API源
```

## 附录 B：依赖方向对比（暴露 vs 调用）

```
                ser-wallet 的 RPC 双向关系
                ═══════════════════════════

        暴露给别人（12个）          我方调用别人（~12个）
        ──────────────             ──────────────
        CreditWallet    ←──┐  ┌──→ GetPaymentMethods
        DebitWallet     ←──│  │──→ MatchAndCreateChannelOrder
        FreezeBalance   ←──│  │──→ InitiateChannelPayout
        UnfreezeBalance ←──│  │──→ GetChannelOrderStatus
        DeductFrozen    ←──│  │
        QueryBalance    ←──┘  │    (以上 4 个 ↔ ser-finance)
        DebitByPriority ←     │
                              │──→ KycQuery
        GetCurrencyConfig←    │    (ser-kyc)
        GetExchangeRate  ←    │
        ConvertAmount    ←    │──→ GetUserInfoExpend
                              │    (ser-user)
        BatchQueryBalance←    │
        GetWalletFlowList←    │──→ AddActLog (ser-blog)
                              │──→ QueryUserByIds (ser-buser)
                              │──→ WebSocket推送 (ser-socket)
                              └──→ 三方汇率API (HTTP)
```

## 附录 C：main.go 中需初始化的 RPC Client 清单

```go
// ser-wallet/main.go — RPC Client 初始化
func initExternalClients() {
    // ===== 确定使用 =====
    _ = rpc.KycClient()     // O-5: KYC认证查询
    _ = rpc.UserClient()    // O-6: 用户信息查询
    _ = rpc.BLogClient()    // O-7: B端操作日志
    _ = rpc.BUserClient()   // O-8: B端用户查询

    // ===== 待 ser-finance 就绪后启用 =====
    // _ = rpc.FinanceClient()  // O-1~O-4: 需新建 Client

    // ===== 待确认后启用 =====
    // _ = rpc.WebSocketClient()  // O-9: 余额推送
    // _ = rpc.I18nClient()       // O-11: 多语言
}
```
