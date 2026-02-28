# ser-wallet 对外暴露 RPC 接口分析

> 撰写时间：2026-02-28
> 分析视角：ser-wallet 模块需要对外暴露哪些 RPC 接口供工程内其他模块调用
> 范围界定：仅分析 **RPC 接口**（内部服务间调用），不含 B 端 HTTP 接口和 C 端 HTTP 接口
> 分析依据：
> - 需求分析：/Users/mac/gitlab/z-readme/result/tmp-2.md（152 张图片提取）
> - 依赖关系：/Users/mac/gitlab/z-readme/依赖关系/tmp-2.md（模块间调用方向全景）
> - 链路关系：/Users/mac/gitlab/z-readme/链路关系/tmp-2.md（46 个接口逐个调用链路）
> - 财务结构：/Users/mac/gitlab/z-readme/财务结构/tmp-2.md（财务模块工程骨架推演）
> - 工程规范：已有 ser-live / ser-item / ser-buser 的 service.thrift 实际 RPC 定义
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-2.md（第三版，46 个接口）

---

## 一、为什么需要单独分析 RPC 暴露接口

### 1.1 问题定义

ser-wallet 对外有三类接口：
- **B 端接口**（6 个）：运营后台管理币种配置，调用方是 B 端前端
- **C 端接口**（19 个）：用户端钱包操作，调用方是 C 端前端
- **RPC 接口**（本文分析对象）：工程内部其他服务调用，调用方是 ser-finance、ser-live、游戏模块等

B 端和 C 端接口的调用方是前端，接口定义权完全在我方。但 RPC 接口的调用方是**其他后端服务**，这意味着：
- **接口定义即合同**：一旦 IDL 定义发布，其他服务就会按此开发，变更成本高
- **参数设计必须精准**：传什么字段、用什么类型、返回什么结构，直接影响联调效率
- **幂等和安全必须内建**：RPC 被外部服务调用，无法控制调用频率和时序

### 1.2 分析目的

1. 确定 ser-wallet 需要暴露哪些 RPC 方法
2. 每个方法被谁调用、在什么场景下调用、带什么参数
3. 参照项目现有 RPC 定义惯例，推导 IDL 结构设计
4. 识别优先级，区分"必须首批暴露"和"可以后续补充"

---

## 二、项目现有 RPC 定义惯例

### 2.1 IDL 文件组织方式

通过分析 common/idl/ 下已有的 23 个服务目录，提取出 RPC 定义的统一惯例：

**ser-live（最完整的参考）：**
```
common/idl/ser-live/
├── service.thrift              # 总入口，定义 LiveService
├── live_room.thrift            # 房间管理域
├── live_consume.thrift         # 消费域
├── live_audience.thrift        # 观众域
├── live_convention.thrift      # 公约域
├── live_audit.thrift           # 审核域
├── live_user.thrift            # 用户域
├── live_api.thrift             # C端API域
└── live_rpc.thrift             # ★ RPC供他方域（独立文件）★
```

关键模式：
- RPC 接口的请求/响应结构定义在独立的 `live_rpc.thrift` 文件中
- service.thrift 中用注释分隔：`// ========== RPC接口（供其他服务调用） ==========`
- RPC 方法和 B/C 端方法在同一个 Service 定义中，但通过注释明确区分

**ser-item（简化版参考）：**
```
service ItemService {
    // 道具配置管理
    ...（B端接口）

    // 道具查询接口（供其他服务调用）     ← 注释标识 RPC 区域
    GetItemsByType(...)                    ← 按类型批量获取
    BatchGetItemsByIds(...)                ← 按ID批量获取
}
```

关键模式：
- RPC 方法以 `Get/Batch` 前缀为主，面向查询
- 请求/响应结构在已有的 `item_publish.thrift` 域文件中定义（不需要独立 RPC 文件）

**ser-buser（用户服务参考）：**
- QueryUserByIds：批量查用户
- QueryUserIdByName：按名字查ID
- 同样是查询型 RPC，供其他服务获取用户信息

### 2.2 提取的命名和设计规范

| 维度 | 惯例 | 来源 |
|------|------|------|
| Service 名 | `XxxService`（首字母大写，驼峰） | LiveService / ItemService |
| 方法名 | 动词+名词，驼峰命名 | BatchGetLiveUser / GetItemsByType |
| 请求结构名 | `XxxReq` | BatchGetLiveUserReq |
| 响应结构名 | `XxxResp` | BatchGetLiveUserResp |
| 字段注解 | `(api.body="fieldName")` | 所有已有 IDL 均遵循 |
| RPC 区域标识 | `// ========== RPC接口（供其他服务调用） ==========` | ser-live |
| 独立 RPC 文件 | 当 RPC 结构较多时，拆出 `xxx_rpc.thrift` | ser-live 的 live_rpc.thrift |
| namespace | `namespace go ser_xxx` | 所有服务统一 |

### 2.3 现有 RPC 的共性特征

```
已有服务的 RPC 接口特征：
├── 方向：几乎全是"被动提供数据"，即查询型
│     BatchGetLiveUser       → 批量查直播用户信息
│     GetItemsByType         → 按类型查道具
│     BatchGetItemsByIds     → 按ID批查道具
│     QueryUserByIds         → 按ID批查用户
│     QueryUserIdByName      → 按名字查用户ID
│
├── 操作类型：只读（不修改数据）
│
└── 设计思路：
    · 批量优先（Batch/List 前缀）
    · 参数尽量简单（ID 列表或类型码）
    · 返回结构完整（一次返回所有需要的字段，减少二次调用）
```

**ser-wallet 的 RPC 特殊之处：不仅有查询型，还有大量的写操作型 RPC（入账/扣款/冻结/解冻）。这在现有工程中是独一无二的，因为钱包是唯一管理用户资金的服务。**

---

## 三、ser-wallet 需暴露的 RPC 接口总览

基于依赖关系分析和链路关系分析的综合推导，ser-wallet 需要暴露 **12 个 RPC 接口**。

### 3.1 全景一览

```
ser-wallet RPC 接口（供其他服务调用）
│
├── 余额操作类（6 个）— 写操作，改变用户余额
│   ├── R-1  CreditWallet          入账/加款（可用余额 +）
│   ├── R-2  DebitWallet           扣款/减款（可用余额 -）
│   ├── R-3  FreezeBalance         冻结余额（可用→冻结）
│   ├── R-4  UnfreezeBalance       解冻余额（冻结→可用）
│   ├── R-5  DeductFrozenAmount    扣除冻结（冻结减少，钱真的出去了）
│   └── R-7  DebitByPriority       按优先级扣款（先扣中心再扣奖励）
│
├── 余额查询类（2 个）— 读操作
│   ├── R-6  QueryWalletBalance    查询单用户各钱包余额
│   └── R-11 BatchQueryBalance     批量查询多用户余额
│
├── 币种信息类（3 个）— 读操作
│   ├── R-8  GetCurrencyConfig     获取币种配置列表
│   ├── R-9  GetExchangeRate       获取指定币种汇率
│   └── R-10 ConvertAmount         金额换算（A币种→B币种）
│
└── 流水查询类（1 个）— 读操作
    └── R-12 GetWalletFlowList     查询用户账变流水
```

### 3.2 按调用方分类

```
调用方                  调用的 RPC 接口                    场景
─────────────────────────────────────────────────────────────────
ser-finance（财务）     CreditWallet                      充值入账/补单/加款
                       DebitWallet                        人工减款
                       FreezeBalance                      (如提现归财务创建)
                       UnfreezeBalance                    提现驳回解冻
                       DeductFrozenAmount                 出款成功扣冻结
                       QueryWalletBalance                 加减款/补单表单查余额
                       GetCurrencyConfig                  统计报表需要币种信息
                       GetExchangeRate                    统计报表需要汇率
                       ConvertAmount                      金额换算为基准币种

游戏/投注模块           DebitByPriority                   投注扣款（自动按优先级）
                       CreditWallet                       返奖入账（按扣款比例拆分）
                       QueryWalletBalance                 余额校验

直播/礼物模块           DebitByPriority                   打赏/送礼扣款
                       CreditWallet                       主播收入入账

活动/运营模块           CreditWallet                       活动奖励发放
                       GetCurrencyConfig                  活动关联币种校验

日志/运营报表           BatchQueryBalance                  批量查余额做统计
                       GetWalletFlowList                  查资金流水
                       ConvertAmount                      金额换算

其他潜在调用方          GetCurrencyConfig                  任何需要知道平台支持哪些币种的模块
                       GetExchangeRate                    任何需要汇率信息的模块
```

---

## 四、逐个 RPC 接口详解

### R-1: CreditWallet（入账/加款）

#### 被谁调用

| 调用方 | 场景 | 频率 | 来源文件(财务侧) |
|--------|------|------|------------------|
| ser-finance | 充值成功入账 | 高频 | callback.go |
| ser-finance | 充值补单入账 | 低频 | adjustment.go |
| ser-finance | 人工加款 | 低频 | adjustment.go |
| 游戏模块 | 投注返奖 | 高频 | — |
| 直播模块 | 主播收入入账 | 中频 | — |
| 活动模块 | 活动奖励发放 | 低频 | — |

#### 调用参数设计

```
CreditWalletReq {
    userId:        i64      // 目标用户ID
    currencyCode:  string   // 币种代码（VND/IDR/THB/USDT/BSB）
    walletType:    i32      // 子钱包类型（1中心/2奖励/3主播/4代理）
    amount:        string   // 入账金额（Decimal字符串，精度由币种配置决定）
    orderNo:       string   // 关联订单号（幂等键的一部分）
    opType:        i32      // 操作类型（充值/补单/加款/返奖/奖励/主播收入/...）
    auditMultiplier: optional string  // 稽核倍数（仅奖励钱包）
    auditTurnover:   optional string  // 要求流水金额（仅奖励钱包）
    remark:        optional string    // 操作备注
}

CreditWalletResp {
    success:      bool      // 是否成功
    newBalance:   string    // 操作后新余额
    flowId:       string    // 生成的流水ID
}
```

#### 核心逻辑

```
收到 CreditWallet 调用
  ├→ 幂等校验：orderNo + walletType 组合是否已处理
  │    → 已处理 → 直接返回成功（不重复入账）
  ├→ 参数校验：amount > 0，walletType 合法，币种已启用
  ├→ 余额操作：指定子钱包可用余额 += amount
  ├→ 写账变流水：(userId, currencyCode, walletType, +amount, orderNo, opType)
  ├→ 如有稽核信息：写稽核要求记录
  └→ 返回(成功, 新余额, 流水ID)
```

#### 关键设计点

1. **幂等键 = orderNo + walletType**。人工加款场景下，同一个 A+16 位订单号会调用多次（中心+奖励+主播+代理各一次），必须用 orderNo + walletType 组合做幂等，而非单独 orderNo
2. **amount 用 string 传递 Decimal**。避免浮点精度问题，由钱包内部统一解析和精度校验
3. **opType 枚举必须覆盖所有入账场景**。充值=1/补单=2/加款=3/返奖=4/活动奖励=5/主播收入=6/兑换=7/...（需与财务模块对齐）

依据：[需求 3.10] "充值回调必须实现幂等处理"；链路关系/tmp-2.md R-1 章节。

---

### R-2: DebitWallet（扣款/减款）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| ser-finance | 人工减款 | 低频 |
| 游戏模块 | 投注扣款（指定钱包时） | 中频 |
| 直播/礼物模块 | 送礼扣款（指定钱包时） | 中频 |

#### 调用参数设计

```
DebitWalletReq {
    userId:        i64
    currencyCode:  string
    walletType:    i32      // 指定从哪个子钱包扣
    amount:        string
    orderNo:       string
    opType:        i32      // 操作类型（减款/投注/购买/...）
    remark:        optional string
}

DebitWalletResp {
    success:      bool
    newBalance:   string    // 扣款后新余额
    flowId:       string
}
```

#### 核心逻辑

```
收到 DebitWallet 调用
  ├→ 幂等校验：orderNo + walletType 是否已处理
  ├→ 查询指定子钱包可用余额
  │    → 可用余额 < amount → 返回余额不足错误
  ├→ 余额操作：指定子钱包可用余额 -= amount
  ├→ 写账变流水
  └→ 返回(成功, 新余额, 流水ID)
```

#### 关键设计点

1. **余额不足必须明确拒绝**。返回具体错误码，调用方需要处理此场景
2. **与 DebitByPriority 的区别**：DebitWallet 是"指定钱包扣"，DebitByPriority 是"自动按优先级分配扣"。财务减款用 DebitWallet（运营指定了扣哪个钱包），游戏投注通常用 DebitByPriority

依据：链路关系/tmp-2.md R-2 章节；财务结构/tmp-2.md 场景七。

---

### R-3: FreezeBalance（冻结余额）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| 内部（C-8 CreateWithdrawOrder） | 提现冻结 | 中频 |
| ser-finance（如提现归财务创建） | 提现冻结 | 中频 |

#### 调用参数设计

```
FreezeBalanceReq {
    userId:        i64
    currencyCode:  string
    walletType:    i32      // 目前只有中心钱包可提现
    amount:        string
    orderNo:       string   // 提现订单号 T+16位
}

FreezeBalanceResp {
    success:      bool
    frozenBalance: string   // 冻结后的冻结余额总额
}
```

#### 核心逻辑

```
收到 FreezeBalance 调用
  ├→ 查询可用余额 >= amount？
  │    → 不足 → 拒绝
  ├→ 可用余额 -= amount
  ├→ 冻结余额 += amount
  ├→ 写账变流水（类型=冻结）
  └→ 返回(成功)
```

#### 关键设计点

**FreezeBalance / UnfreezeBalance / DeductFrozenAmount 是配套的三步操作：**
```
正常提现：Freeze → (审核通过+出款成功) → DeductFrozen
驳回提现：Freeze → (审核驳回) → Unfreeze
出款失败：Freeze → (出款失败) → 保持冻结（不调任何接口，等人工）
```

这三个接口必须一起设计、一起测试。

依据：[需求 5.5] "创建订单→立即冻结"；依赖关系/tmp-2.md §3.2。

---

### R-4: UnfreezeBalance（解冻余额）

#### 被谁调用

| 调用方 | 场景 | 频率 | 来源文件(财务侧) |
|--------|------|------|------------------|
| ser-finance | 风控审核驳回 | 低频 | withdraw_record.go |
| ser-finance | 财务审核驳回 | 低频 | withdraw_record.go |
| ser-finance | 出款驳回 | 低频 | withdraw_record.go |

#### 调用参数设计

```
UnfreezeBalanceReq {
    userId:        i64
    currencyCode:  string
    walletType:    i32
    amount:        string   // 解冻金额 = 原冻结金额
    orderNo:       string   // 原提现订单号 T+16位
}

UnfreezeBalanceResp {
    success:      bool
    availableBalance: string  // 解冻后的可用余额
}
```

#### 核心逻辑

```
收到 UnfreezeBalance 调用
  ├→ 校验冻结余额 >= amount
  ├→ 冻结余额 -= amount
  ├→ 可用余额 += amount（钱回到可用状态）
  ├→ 写账变流水（类型=解冻）
  └→ 返回(成功)
```

依据：[需求 5.5] "风控驳回→解冻钱包金额"，"财务驳回→解冻钱包金额"。

---

### R-5: DeductFrozenAmount（扣除冻结金额）

#### 被谁调用

| 调用方 | 场景 | 频率 | 来源文件(财务侧) |
|--------|------|------|------------------|
| ser-finance | 三方通道出款成功回调 | 低频 | callback.go |
| ser-finance | 人工确认出款成功 | 低频 | withdraw_record.go |

#### 调用参数设计

```
DeductFrozenAmountReq {
    userId:        i64
    currencyCode:  string
    walletType:    i32
    amount:        string
    orderNo:       string   // 原提现订单号 T+16位
}

DeductFrozenAmountResp {
    success:      bool
    frozenBalance: string   // 扣除后的冻结余额
}
```

#### 核心逻辑

```
收到 DeductFrozenAmount 调用
  ├→ 幂等校验：orderNo 是否已执行过扣冻结
  ├→ 校验冻结余额 >= amount
  ├→ 冻结余额 -= amount（不加回可用余额，钱真的出去了）
  ├→ 写账变流水（类型=提现扣除）
  └→ 返回(成功)
```

#### 与 UnfreezeBalance 的本质区别

```
UnfreezeBalance:  冻结 → 可用   （钱回到用户手里，驳回/取消场景）
DeductFrozen:     冻结 → 消失   （钱离开了系统，出款成功场景）
```

依据：[需求 5.5] "出款成功→扣除冻结金额"。

---

### R-6: QueryWalletBalance（查询余额）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| ser-finance | 补单创建表单展示余额 | 低频 |
| ser-finance | 加款创建表单展示余额 | 低频 |
| ser-finance | 减款创建表单展示余额（校验减款上限） | 低频 |
| 游戏模块 | 投注前余额校验 | 高频 |
| 其他模块 | 按需查余额 | 中频 |

#### 调用参数设计

```
QueryWalletBalanceReq {
    userId:        i64
    currencyCode:  string   // 可选：不传则返回所有币种
}

QueryWalletBalanceResp {
    wallets: list<WalletBalanceInfo>
}

WalletBalanceInfo {
    currencyCode:    string
    walletType:      i32      // 1中心/2奖励/3主播/4代理
    walletTypeName:  string   // 钱包类型名称
    availableBalance: string  // 可用余额
    frozenBalance:    string  // 冻结余额
    totalBalance:     string  // 总余额 = 可用 + 冻结
}
```

#### 关键设计点

1. **返回所有子钱包**。财务加款表单需要同时展示中心/奖励/主播/代理四个子钱包的余额
2. **冻结余额也要返回**。减款场景下前端需要校验"可用余额 - 减款金额 >= 0"
3. **纯读操作，无副作用**。可以安全缓存，但缓存时间不宜过长（资金数据时效性要求高）

依据：财务结构/tmp-2.md 场景五（补单）、场景六（加款）、场景七（减款）。

---

### R-7: DebitByPriority（按消费优先级扣款）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| 游戏模块 | 投注扣款 | 高频 |
| 直播/礼物模块 | 打赏/送礼扣款 | 高频 |
| 商城/道具模块 | 道具购买扣款 | 中频 |

#### 调用参数设计

```
DebitByPriorityReq {
    userId:        i64
    currencyCode:  string
    amount:        string   // 总扣款金额
    orderNo:       string
    opType:        i32      // 操作类型（投注/购买/打赏/...）
}

DebitByPriorityResp {
    success:          bool
    deductDetails:    list<DeductDetail>  // 扣款明细（关键！）
    remainingCenter:  string              // 中心钱包剩余
    remainingReward:  string              // 奖励钱包剩余
}

DeductDetail {
    walletType:   i32      // 从哪个钱包扣的
    amount:       string   // 扣了多少
}
```

#### 核心逻辑

```
收到 DebitByPriority 调用(amount=100)
  ├→ 查询中心钱包可用余额 centerBalance
  ├→ 查询奖励钱包可用余额 rewardBalance
  ├→ 判断扣款分配：
  │    centerBalance=80, rewardBalance=50
  │    → 80 < 100 → 中心不够
  │    → 80 + 50 = 130 >= 100 → 总余额够
  │    → center 扣 80, reward 扣 20
  ├→ 分别执行扣款 + 写流水（1~2条流水）
  └→ 返回(成功, [{center,80}, {reward,20}])
```

#### 为什么返回扣款明细

**这是此接口最关键的设计。** 因为 [需求 4.1.2] 规定：
- 中心钱包投注 → 返奖归中心
- 奖励钱包投注 → 返奖归奖励
- 混合投注（中心80+奖励20）→ 返奖按比例拆分（中心80%+奖励20%）

调用方拿到 `deductDetails` 后，返奖时需按此比例分别调 CreditWallet。如果不返回明细，返奖就无法正确拆分。

#### 扣款优先级

```
扣款顺序（固定）：
1. 中心钱包（用户真金白银充的钱，优先消耗）
2. 奖励钱包（兑换/活动获得，稽核流水要求高，后消耗）
注：主播钱包和代理钱包不参与消费扣款
```

依据：[需求 4.1.2] "资金钱包余额 = 中心 + 奖励"，消费场景先扣中心再扣奖励。

---

### R-8: GetCurrencyConfig（获取币种配置）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| ser-finance | 统计报表显示币种信息 | 中频 |
| 活动模块 | 活动关联币种校验 | 低频 |
| 游戏模块 | 奖池币种配置 | 低频 |
| 任何模块 | 需要知道平台支持哪些币种 | 中频 |

#### 调用参数设计

```
GetCurrencyConfigReq {
    onlyEnabled: optional bool    // 是否只返回已启用的（默认true）
}

GetCurrencyConfigResp {
    currencies: list<CurrencyConfigInfo>
    baseCurrency: string              // 基准币种代码
}

CurrencyConfigInfo {
    currencyCode:    string    // 币种代码（VND/IDR/THB/USDT/BSB）
    currencyName:    string    // 币种名称
    currencySymbol:  string    // 货币符号（₫/Rp/฿/$）
    iconUrl:         string    // 币种图标URL
    decimalPlaces:   i32       // 小数精度
    thousandSep:     string    // 千分位分隔符
    status:          i32       // 1启用/0禁用
    sortOrder:       i32       // 排序权重
    isBaseCurrency:  bool      // 是否是基准币种
}
```

#### 关键设计点

1. **高频读接口，建议调用方缓存**。币种配置变更频率极低（可能只在上线初期配置一次）
2. **返回基准币种信息**。统计模块需要知道基准币种是什么，才能做金额换算

依据：[需求 4.3] 币种配置字段定义；[需求 4.5.1] 基准币种概念。

---

### R-9: GetExchangeRate（获取汇率）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| ser-finance | 统计报表金额换算 | 中频 |
| 任何模块 | 需要知道币种间汇率 | 中频 |

#### 调用参数设计

```
GetExchangeRateReq {
    currencyCode:  string    // 币种代码
    rateType:      optional i32  // 汇率类型：1实时/2平台/3入款/4出款（默认2平台）
}

GetExchangeRateResp {
    currencyCode:  string
    rateToBase:    string    // 对基准币种的汇率
    rateType:      i32       // 返回的汇率类型
    updateTime:    i64       // 汇率更新时间戳
}
```

#### 关键设计点

1. **四种汇率类型**：实时汇率（三方API直取均值）、平台汇率（阈值触发更新）、入款汇率（平台+浮动上浮）、出款汇率（平台+浮动下浮）
2. **建议带更新时间**。调用方可据此判断汇率新鲜度

依据：[需求 4.5.1] 四种汇率定义；[需求 4.5.2-4.5.3] 汇率更新机制。

---

### R-10: ConvertAmount（金额换算）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| ser-finance | 统计报表：所有币种汇总为基准币种 | 中频 |
| 任何模块 | 跨币种金额换算 | 低频 |

#### 调用参数设计

```
ConvertAmountReq {
    amount:          string  // 原始金额
    fromCurrency:    string  // 源币种
    toCurrency:      string  // 目标币种
    rateType:        optional i32  // 使用哪种汇率（默认2平台汇率）
}

ConvertAmountResp {
    convertedAmount: string  // 换算后金额
    rateUsed:        string  // 使用的汇率值
    rateType:        i32     // 使用的汇率类型
}
```

#### 核心逻辑

```
ConvertAmount(100 VND → USDT, 平台汇率)
  ├→ 获取 VND 对 USDT 的平台汇率
  ├→ 计算换算结果 = 100 / rate
  ├→ 按目标币种精度截断/四舍五入
  └→ 返回(换算结果, 使用的汇率)
```

依据：[需求 4.3] "所有币种金额在核算层必须换算为基准币种进行统计"。

---

### R-11: BatchQueryBalance（批量查询余额）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| 运营报表 | 批量查用户余额做统计 | 低频 |
| B端管理 | 用户管理页面展示余额 | 低频 |

#### 调用参数设计

```
BatchQueryBalanceReq {
    userIds:       list<i64>
    currencyCode:  optional string  // 不传则返回所有币种
}

BatchQueryBalanceResp {
    balances: map<i64, list<WalletBalanceInfo>>  // userId → 余额列表
}
```

#### 关键设计点

1. **限制批量大小**。防止超大批次查询导致 DB 压力，建议限制 userIds 上限（如 100 个）
2. **复用 WalletBalanceInfo 结构**。与 R-6 共用同一个响应子结构

---

### R-12: GetWalletFlowList（查询账变流水）

#### 被谁调用

| 调用方 | 场景 | 频率 |
|--------|------|------|
| B端管理 | 运营查看用户资金明细 | 低频 |
| 审计/合规 | 资金流水审计 | 低频 |

#### 调用参数设计

```
GetWalletFlowListReq {
    userId:        i64
    currencyCode:  optional string
    walletType:    optional i32
    opType:        optional i32      // 按操作类型过滤
    startTime:     optional i64
    endTime:       optional i64
    page:          i32
    pageSize:      i32
}

GetWalletFlowListResp {
    total:   i64
    flows:   list<WalletFlowInfo>
}

WalletFlowInfo {
    flowId:        string
    userId:        i64
    currencyCode:  string
    walletType:    i32
    opType:        i32
    opTypeName:    string    // 操作类型名称
    amount:        string    // 变动金额（+/-）
    balanceBefore: string    // 变动前余额
    balanceAfter:  string    // 变动后余额
    orderNo:       string    // 关联订单号
    remark:        string    // 备注
    createdAt:     i64       // 时间戳
}
```

---

## 五、IDL 文件结构设计

### 5.1 推荐的文件组织

参照 ser-live 的模式（RPC 结构独立文件）+ ser-item 的模式（混合定义），推荐如下结构：

```
common/idl/ser-wallet/               ← 新建目录
├── service.thrift                   # 总入口，定义 WalletService（37个方法）
├── wallet_c.thrift                  # C端域：充值/提现/兑换/余额 的请求响应结构
├── currency_b.thrift                # B端域：币种配置 CRUD 的请求响应结构
├── wallet_rpc.thrift                # ★ RPC域：12个RPC方法的请求响应结构 ★
├── common.thrift                    # 公共结构：WalletBalanceInfo / WalletFlowInfo / CurrencyConfigInfo
└── enum.thrift                      # 公共枚举：WalletType / OpType / RateType 等
```

### 5.2 service.thrift 中的 RPC 区域

```thrift
namespace go ser_wallet

include "wallet_c.thrift"
include "currency_b.thrift"
include "wallet_rpc.thrift"
include "common.thrift"

service WalletService {
    // ========== C端接口 ==========
    wallet_c.GetWalletOverviewResp GetWalletOverview(1: wallet_c.GetWalletOverviewReq req)
    // ...（19个C端方法）

    // ========== B端接口（币种配置）==========
    currency_b.GetCurrencyListResp GetCurrencyList(1: currency_b.GetCurrencyListReq req)
    // ...（6个B端方法）

    // ========== RPC接口（供其他服务调用）==========

    // --- 余额操作 ---
    wallet_rpc.CreditWalletResp CreditWallet(1: wallet_rpc.CreditWalletReq req)
    wallet_rpc.DebitWalletResp DebitWallet(1: wallet_rpc.DebitWalletReq req)
    wallet_rpc.FreezeBalanceResp FreezeBalance(1: wallet_rpc.FreezeBalanceReq req)
    wallet_rpc.UnfreezeBalanceResp UnfreezeBalance(1: wallet_rpc.UnfreezeBalanceReq req)
    wallet_rpc.DeductFrozenAmountResp DeductFrozenAmount(1: wallet_rpc.DeductFrozenAmountReq req)
    wallet_rpc.DebitByPriorityResp DebitByPriority(1: wallet_rpc.DebitByPriorityReq req)

    // --- 余额查询 ---
    wallet_rpc.QueryWalletBalanceResp QueryWalletBalance(1: wallet_rpc.QueryWalletBalanceReq req)
    wallet_rpc.BatchQueryBalanceResp BatchQueryBalance(1: wallet_rpc.BatchQueryBalanceReq req)

    // --- 币种信息 ---
    wallet_rpc.GetCurrencyConfigResp GetCurrencyConfig(1: wallet_rpc.GetCurrencyConfigReq req)
    wallet_rpc.GetExchangeRateResp GetExchangeRate(1: wallet_rpc.GetExchangeRateReq req)
    wallet_rpc.ConvertAmountResp ConvertAmount(1: wallet_rpc.ConvertAmountReq req)

    // --- 流水查询 ---
    wallet_rpc.GetWalletFlowListResp GetWalletFlowList(1: wallet_rpc.GetWalletFlowListReq req)
}
```

### 5.3 wallet_rpc.thrift 概要

此文件定义 12 个 RPC 方法各自的 Req/Resp 结构。按照项目惯例，所有字段带 `(api.body="fieldName")` 注解。

结构定义在第四章已逐个列出，此处不重复展开。

---

## 六、RPC 接口分组与优先级

### 6.1 按开发阶段分组

```
Phase 2（核心能力层，最先实现）
════════════════════════════════════
  必须首批暴露：
  ├── R-1  CreditWallet         ← 充值入账依赖它，P0
  ├── R-2  DebitWallet           ← 人工减款依赖它，P0
  ├── R-3  FreezeBalance         ← 提现冻结依赖它，P0
  ├── R-4  UnfreezeBalance       ← 提现驳回依赖它，P0
  ├── R-5  DeductFrozenAmount    ← 出款成功依赖它，P0
  └── R-6  QueryWalletBalance    ← 财务表单展示余额，P0

  说明：这 6 个接口是财务模块联调的硬依赖。没有它们，
  充值入账、提现全链路、人工加减款全部无法运行。

Phase 2（核心能力层，同批实现）
════════════════════════════════════
  同批暴露：
  ├── R-8  GetCurrencyConfig     ← 其他模块获取币种信息，P0
  ├── R-9  GetExchangeRate       ← 其他模块获取汇率，P0
  └── R-10 ConvertAmount         ← 金额换算，P1

  说明：币种和汇率是基础数据，几乎所有模块都需要。

Phase 3（C端功能层）
════════════════════════════════════
  按需暴露：
  └── R-7  DebitByPriority       ← 游戏投注/直播打赏场景，P1

  说明：投注和打赏场景依赖它，但这些场景在 Phase 3 或
  更后期才会联调。可以在 Phase 2 后期同步实现。

Phase 4（联调验证层）
════════════════════════════════════
  补充暴露：
  ├── R-11 BatchQueryBalance     ← 运营报表批量查余额，P2
  └── R-12 GetWalletFlowList     ← B端查流水明细，P2

  说明：这两个是辅助性的读取接口，不阻塞任何核心业务链路。
```

### 6.2 优先级矩阵

| 接口 | 优先级 | 被谁强依赖 | 断了的后果 |
|------|--------|-----------|-----------|
| R-1 CreditWallet | **P0** | 财务：充值入账/补单/加款 | 充值成功但钱进不了账 |
| R-2 DebitWallet | **P0** | 财务：人工减款 | 减款审核通过但执行不了 |
| R-3 FreezeBalance | **P0** | 提现流程：冻结步骤 | 提现无法发起 |
| R-4 UnfreezeBalance | **P0** | 财务：提现驳回 | 驳回后钱回不来 |
| R-5 DeductFrozenAmount | **P0** | 财务：出款成功 | 出款成功但冻结不释放 |
| R-6 QueryWalletBalance | **P0** | 财务：加减款表单 | 表单无法展示余额 |
| R-7 DebitByPriority | **P1** | 游戏：投注扣款 | 投注功能不可用 |
| R-8 GetCurrencyConfig | **P0** | 所有模块 | 无法获取币种信息 |
| R-9 GetExchangeRate | **P0** | 所有需要汇率的模块 | 金额换算失效 |
| R-10 ConvertAmount | **P1** | 财务统计 | 统计报表无法跨币种汇总 |
| R-11 BatchQueryBalance | **P2** | 运营报表 | 报表功能不全 |
| R-12 GetWalletFlowList | **P2** | B端管理 | B端无法查看用户流水 |

---

## 七、跨模块 RPC 调用时序全景

### 7.1 财务调钱包 — 完整调用时序

```
时间线 ──────────────────────────────────────────────────────────────►

[三方通道]           [ser-finance]                    [ser-wallet]

                                                        RPC 接口
充值成功回调 ─────→ callback.go                          │
                    ├ 验签+更新订单                       │
                    └ CreditWallet ────────────────────→ R-1 入账
                      (中心钱包, 充值金额)                │
                    └ CreditWallet ────────────────────→ R-1 入账
                      (奖励钱包, 奖金金额, 稽核)          │
                                                        │
出款成功回调 ─────→ callback.go                          │
                    └ DeductFrozenAmount ──────────────→ R-5 扣冻结
                      (T+16位订单号)                      │
                                                        │
                  withdraw_record.go                     │
审核驳回(B端) ──→  └ UnfreezeBalance ─────────────────→ R-4 解冻
                      (T+16位订单号)                      │
                                                        │
                  adjustment.go                          │
创建补单(B端) ──→  └ QueryWalletBalance ──────────────→ R-6 查余额
审核补单通过 ───→  └ CreditWallet ────────────────────→ R-1 入账
                                                        │
创建加款(B端) ──→  └ QueryWalletBalance ──────────────→ R-6 查余额
审核加款通过 ───→  └ CreditWallet × 1~4 ─────────────→ R-1 入账×N
                    (每个有金额的子钱包各一次)              │
                                                        │
创建减款(B端) ──→  └ QueryWalletBalance ──────────────→ R-6 查余额
审核减款通过 ───→  └ DebitWallet × 1~4 ──────────────→ R-2 扣款×N
```

### 7.2 游戏/直播调钱包 — 调用时序

```
[游戏模块]                          [ser-wallet]

用户下注 ──→ DebitByPriority ─────→ R-7 按优先级扣款
             ← 返回扣款明细          │  (center扣80, reward扣20)
             (记录明细留待返奖)       │
                                    │
投注中奖 ──→ CreditWallet ─────────→ R-1 入账
             (center, 奖金×80%)     │  (按扣款比例拆分)
          ──→ CreditWallet ─────────→ R-1 入账
             (reward, 奖金×20%)     │
```

```
[直播模块]                          [ser-wallet]

观众送礼 ──→ DebitByPriority ─────→ R-7 按优先级扣款
                                    │
主播结算 ──→ CreditWallet ─────────→ R-1 入账
             (主播钱包, 收入金额)     │
```

---

## 八、联调前待对齐事项

这些是编写 IDL 前必须与财务模块开发者达成一致的关键约定：

### 8.1 P0 对齐项（阻塞 IDL 定义）

| 序号 | 事项 | 影响范围 | 说明 |
|------|------|---------|------|
| 1 | **amount 字段类型** | 所有余额操作RPC | string（Decimal字符串）还是 i64（最小单位整数，如"分"）？推荐 string，精度由钱包内部处理 |
| 2 | **walletType 枚举值** | 所有余额操作RPC | 1=中心/2=奖励/3=主播/4=代理，双方必须完全一致 |
| 3 | **幂等键组成** | CreditWallet / DebitWallet | orderNo 单独 vs orderNo+walletType 组合？推荐后者（加款场景一个orderNo对应多次调用） |
| 4 | **opType 枚举值** | CreditWallet / DebitWallet | 充值=1/补单=2/加款=3/返奖=4/活动=5/主播收入=6/减款=7/投注=8/购买=9/... |

### 8.2 P1 对齐项（影响联调质量）

| 序号 | 事项 | 影响范围 |
|------|------|---------|
| 5 | 充值入账时 CreditWallet 的完整参数列表（特别是稽核字段） | callback.go → wallet |
| 6 | 提现驳回时 UnfreezeBalance 的完整参数列表 | withdraw_record.go → wallet |
| 7 | 财务模块的 IDL 什么时候出（影响我方 C 端调财务的联调时间） | Phase 3 C端功能 |
| 8 | 出款失败后人工处理流程（是否需要新增 RPC 方法） | withdraw_record.go 重试逻辑 |

### 8.3 错误码对齐

钱包 RPC 需要返回的关键错误码（供调用方处理分支逻辑）：

| 错误码 | 含义 | 调用方处理 |
|--------|------|-----------|
| BALANCE_NOT_ENOUGH | 余额不足 | DebitWallet/FreezeBalance 调用失败时 |
| FROZEN_NOT_ENOUGH | 冻结余额不足 | UnfreezeBalance/DeductFrozen 调用失败时 |
| DUPLICATE_ORDER | 幂等拦截（已处理过） | 调用方视为成功 |
| CURRENCY_DISABLED | 币种已禁用 | 调用方提示用户 |
| WALLET_NOT_FOUND | 钱包不存在 | 调用方检查参数 |
| INVALID_AMOUNT | 金额非法（≤0或精度超限） | 调用方检查参数 |

---

## 九、与项目已有 RPC 的对比

### 9.1 差异分析

| 维度 | 已有服务 RPC | ser-wallet RPC | 差异原因 |
|------|-------------|---------------|---------|
| 操作类型 | 几乎全是只读查询 | 读+写各半（6个写+6个读） | 钱包是资金核心，必须暴露写操作 |
| 接口数量 | 2~3 个 | 12 个 | 钱包承载了全平台资金能力 |
| 幂等要求 | 无（查询无副作用） | 写操作必须幂等 | 资金操作不允许重复执行 |
| 调用频率 | 低~中频 | 高频（投注扣款/入账） | 投注是最高频的资金操作 |
| 事务要求 | 无 | 余额变更+流水必须原子 | 资金一致性是硬性要求 |

### 9.2 ser-wallet RPC 在项目中的特殊地位

```
项目中所有涉及"钱"的操作，最终都汇聚到 ser-wallet 的 RPC 接口：

  充值 → (财务回调) → CreditWallet
  提现 → FreezeBalance → (审核) → DeductFrozen / Unfreeze
  投注 → DebitByPriority → (结算) → CreditWallet
  打赏 → DebitByPriority → (结算) → CreditWallet
  加款 → CreditWallet
  减款 → DebitWallet
  兑换 → (内部操作，不走RPC)

ser-wallet 的 RPC 是整个平台的"资金总开关"。
```

---

## 十、总结

### 10.1 数量统计

| 分类 | 数量 | 具体 |
|------|------|------|
| 余额操作（写） | 6 | Credit / Debit / Freeze / Unfreeze / DeductFrozen / DebitByPriority |
| 余额查询（读） | 2 | QueryBalance / BatchQueryBalance |
| 币种信息（读） | 3 | GetCurrencyConfig / GetExchangeRate / ConvertAmount |
| 流水查询（读） | 1 | GetWalletFlowList |
| **合计** | **12** | |

### 10.2 核心结论

1. **12 个 RPC 接口，其中 8 个是 P0（必须首批实现）**。财务模块的充值入账、提现审核、人工修正全部依赖这 8 个接口。

2. **写操作 RPC 是 ser-wallet 区别于项目其他服务的核心特征**。需要特别注意幂等性、事务原子性、错误码明确性。

3. **IDL 结构推荐独立 wallet_rpc.thrift 文件**。参照 ser-live 的 live_rpc.thrift 模式，将 RPC 请求/响应结构集中管理。

4. **联调前 4 个 P0 对齐项必须优先解决**：amount 类型、walletType 枚举、幂等键组成、opType 枚举。这 4 个未对齐就写 IDL 等于"在流沙上盖房子"。

5. **DebitByPriority 的扣款明细返回是最关键的设计决策**。它直接影响了投注返奖的正确性，是跨模块协作的"隐藏依赖"。

---

## 附录 A：12 个 RPC 接口速查卡

```
接口名                    方向    类型    被谁调            最高频场景
────────────────────────────────────────────────────────────────────
R-1  CreditWallet         写入    P0     财务/游戏/直播    充值入账
R-2  DebitWallet           扣减    P0     财务/游戏         人工减款
R-3  FreezeBalance         冻结    P0     钱包内部/财务     提现发起
R-4  UnfreezeBalance       解冻    P0     财务              提现驳回
R-5  DeductFrozenAmount    扣冻结  P0     财务              出款成功
R-6  QueryWalletBalance    查询    P0     财务              加减款表单
R-7  DebitByPriority       扣减    P1     游戏/直播         投注扣款
R-8  GetCurrencyConfig     查询    P0     所有模块          获取币种列表
R-9  GetExchangeRate       查询    P0     所有模块          获取汇率
R-10 ConvertAmount         计算    P1     财务统计          金额换算
R-11 BatchQueryBalance     查询    P2     运营报表          批量查余额
R-12 GetWalletFlowList     查询    P2     B端管理           查资金流水
```

## 附录 B：钱包调外部的 RPC 接口（反向参考）

钱包自身也需要调用其他模块的 RPC，这些不是本文重点（本文聚焦"暴露给别人"），但列出供完整性参考：

| 目标服务 | 接口 | 调用时机 | 我方调用文件 |
|---------|------|---------|------------|
| ser-finance | GetPaymentMethods | 获取充值/提现方式 | deposit.go / withdraw.go |
| ser-finance | MatchAndCreateChannelOrder | 充值匹配通道 | deposit.go |
| ser-finance | InitiateChannelPayout | 发起代付 | withdraw.go |
| ser-finance | GetChannelOrderStatus | 查通道订单状态 | deposit.go |
| ser-kyc | QueryKycStatus | 提现前查KYC | withdraw.go |
| ser-user | GetUserInfo | 提现前查用户信息 | withdraw.go |

## 附录 C：IDL 文件创建对照表

```
需要创建的文件                    状态        负责方
───────────────────────────────────────────────────
common/idl/ser-wallet/           ❌ 不存在    我方
  ├── service.thrift             待创建       我方
  ├── wallet_c.thrift            待创建       我方
  ├── currency_b.thrift          待创建       我方
  ├── wallet_rpc.thrift          待创建       我方（本文重点）
  ├── common.thrift              待创建       我方
  └── enum.thrift                待创建       我方

common/idl/ser-finance/          ❌ 不存在    同事方
  ├── service.thrift             待创建       同事
  ├── finance_rpc.thrift         待创建       同事（我方需调用的4个接口）
  └── ...其他域文件              待创建       同事

common/pkg/consts/namesp/        ✏️ 需新增    我方 + 同事
  └── namesp.go                  新增 wallet/finance 常量

common/rpc/rpc_client.go         ✏️ 需新增    我方 + 同事
  └── WalletClient() / FinanceClient()

go.work                          ✏️ 需新增    我方 + 同事
  └── ./ser-wallet / ./ser-finance 条目
```
