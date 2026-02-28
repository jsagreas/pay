# ser-wallet 接口调用链路关系

> 撰写时间：2026-02-26
> 分析依据：
> - 需求提取：/Users/mac/gitlab/z-readme/result/tmp-2.md（152张图片分析）
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-2.md（第三版，46个接口）
> - 依赖关系：/Users/mac/gitlab/z-readme/依赖关系/tmp-2.md
> 目的：梳理每个接口被调用时，内部经过了哪些步骤、调了哪些内部方法、调了哪些外部服务，做到链路清晰可追踪

---

## 阅读说明

**符号约定：**
- `→` 表示同步调用
- `-->` 表示异步/回调
- `[内部]` 表示 ser-wallet 服务内部的方法/数据调用
- `[财务RPC]` 表示调用财务管理模块的 RPC 接口
- `[钱包RPC]` 表示财务模块调用钱包暴露的 RPC 接口
- `[KYC RPC]` 表示调用 ser-kyc 服务
- `[User RPC]` 表示调用 ser-user 服务
- `[三方API]` 表示调用外部三方服务

**接口编号沿用接口评估文档：** C-1~C-19（C端）、B-1~B-6（B端）、R-1~R-12（RPC供他方）、O-1~O-6（RPC调他方）、T-1~T-3（定时任务）

---

# 第一部分：钱包模块内部 — 每个 C 端接口的调用链路

## C-1: GetWalletOverview（钱包首页余额查询）

**触发：** 用户打开钱包首页，或切换币种时

**调用链路：**
```
客户端请求(userID, currencyCode)
  → [内部] 币种配置.查询该币种是否已启用
  → [内部] 币种配置.获取币种显示规则(货币符号, 千分位, 小数精度)
  → [内部] 钱包余额表.查询该用户该币种下所有子钱包余额
      → 中心钱包余额(可用 + 冻结)
      → 奖励钱包余额(可用 + 冻结)
      → 主播钱包余额(可用 + 冻结)   [仅主播角色]
      → 代理钱包余额(可用 + 冻结)   [仅代理角色]
  → [内部] 计算资金钱包合计 = 中心可用 + 奖励可用
  → [内部] 按精度规则格式化所有金额
  → 返回
```

**跨模块调用：无。** 纯内部读操作。

**内部依赖：** 币种配置数据（币种状态、显示规则）+ 钱包余额数据。

依据：[需求 3.2-3.3] 币种维度隔离，切换币种立即切换余额；[需求 4.1.2] 资金钱包余额 = 中心 + 奖励。

---

## C-2: GetDepositMethods（获取充值方式列表）

**触发：** 用户进入充值页面

**调用链路：**
```
客户端请求(userID, currencyCode)
  → [内部] 币种配置.确认该币种已启用
  → [财务RPC] GetPaymentMethods(currencyCode, type=deposit)
      → 财务返回：该币种可用的充值方式列表
         (方式名称, 图标URL, 快捷金额档位, 金额范围min~max, 排序, 推荐标记)
  → [内部] 组装返回数据(充值方式列表 + 币种显示信息)
  → 返回
```

**跨模块调用：财务管理 × 1 次。** 充值方式的数据源在财务模块的支付配置中。

**内部依赖：** 币种配置（确认币种可用）。

依据：[需求 5.3] 充值方式字段（币种/名称/图标/档位/范围/关联通道/排序/状态）由财务 B 端配置。

---

## C-3: CreateDepositOrder（创建充值订单）

**触发：** 用户选择充值方式、输入金额、选择奖金活动后，点击"去充值"

**调用链路：**
```
客户端请求(userID, currencyCode, depositMethodID, amount, bonusActivityID?)
  │
  ├→ [内部] 参数校验：金额 >= 该方式最低限额
  │
  ├→ [内部] 重复转账检测 (即 C-14 的逻辑)
  │    → 查询订单表：同userID + 同currencyCode + 同amount + 状态为(待支付/处理中)的订单数
  │    → count >= 3 → 拒绝创建，返回强拦截
  │    → count 1~2 → 标记为"有重复警告"（前端已弹窗确认过，请求中带 confirmOverride=true）
  │    → count 0 → 正常继续
  │    [注意：USDT链上充值跳过此检测]
  │
  ├→ [内部] 币种配置.获取当前汇率（BSB充值需要展示汇率换算）
  │
  ├→ [内部] 如有奖金活动：校验活动门槛（金额 >= 活动最低要求）
  │    → 计算奖金金额 = amount × 奖金百分比
  │    → 计算稽核流水 = (amount + 奖金) × 稽核倍数
  │
  ├→ [财务RPC] MatchAndCreateChannelOrder(depositMethodID, currencyCode, amount)
  │    → 财务内部：轮询算法选通道(成功率×时间×预警×权重)
  │    → 财务内部：创建通道侧订单
  │    → 返回：通道名称, 通道订单号, 支付信息(跳转URL/QR码/收款地址)
  │
  ├→ [内部] 创建平台充值订单记录
  │    → 订单号: C + 16位随机数
  │    → 状态: 待支付
  │    → 记录：金额, 币种, 充值方式, 通道, 汇率快照, 活动信息, 稽核信息
  │
  └→ 返回(订单号, 支付信息)
```

**跨模块调用：财务管理 × 1 次**（通道匹配+创建通道订单）。

**内部依赖的其他接口：** C-14 CheckDuplicateDeposit 的逻辑被内嵌调用。

依据：[需求 3.4.1] "点击去充值后生成订单匹配通道"；[需求 3.5] 重复转账拦截规则；[需求 3.6] 一笔订单只能参与一个奖金活动。

---

## C-4: GetUSDTDepositInfo（获取USDT充值信息）

**触发：** 用户在支持 USDT 的币种下选择 USDT 充值方式

**调用链路：**
```
请求(userID?, currencyCode, network=TRC20)   [userID可选，游客也可访问]
  → [内部] 币种配置.获取 USDT 汇率（用于展示"约等于 X USDT"）
  → [内部] 获取/生成该网络的收款地址
  → [内部] 生成收款地址 QR 码
  → 返回(网络类型, 收款地址, QR码, 汇率信息)
```

**跨模块调用：无。** 但收款地址的来源可能涉及链上钱包服务（待评审确认）。

**特殊点：** 这是 Open 路由（免鉴权），游客也能访问。

依据：[需求 3.4.2] "游客（未登录）也支持 USDT 充值"，展示网络类型/收款地址/QR码。

---

## C-5: GetExchangePreview（兑换预览）

**触发：** 用户在法币钱包中输入兑换金额

**调用链路：**
```
客户端请求(userID, currencyCode, fiatAmount)
  → [内部] 币种配置.获取当前币种对 BSB 的平台汇率
  → [内部] 检查：fiatAmount <= 用户当前法币可用余额
      → 超过 → 返回"已达当前可兑换上限"
  → [内部] 计算 BSB 兑换所得 = fiatAmount / 汇率
  → [内部] 计算赠送奖金 = BSB兑换所得 × 赠送百分比
  → [内部] 计算实际到账 = 兑换所得 + 赠送奖金
  → [内部] 计算稽核流水要求 = 赠送部分 × 稽核倍数
  → 返回(汇率, 兑换所得, 赠送奖金, 实际到账, 稽核流水要求)
```

**跨模块调用：无。** 纯内部计算。

依据：[需求 3.7] 兑换计算逻辑，赠送部分需完成 X 倍流水。

---

## C-6: CreateExchange（执行法币→BSB兑换）

**触发：** 用户确认兑换

**调用链路：**
```
客户端请求(userID, currencyCode, fiatAmount)
  │
  ├→ [内部] 重新获取汇率并计算（同 C-5 逻辑，防止预览后汇率变动）
  │
  ├→ [内部] 再次校验法币余额充足
  │
  ├→ [内部] 扣减法币余额（中心钱包）
  │    → 钱包余额表.update: 中心钱包可用余额 -= fiatAmount
  │    → 写账变流水：类型=兑换支出, 币种=法币
  │
  ├→ [内部] 增加BSB余额（分两笔）
  │    → 钱包余额表.update: BSB中心钱包可用余额 += 兑换所得
  │    → 写账变流水：类型=兑换收入, 币种=BSB, 子钱包=中心
  │    → 钱包余额表.update: BSB奖励钱包可用余额 += 赠送奖金
  │    → 写账变流水：类型=兑换赠送, 币种=BSB, 子钱包=奖励
  │
  ├→ [内部] 记录稽核流水要求（赠送部分关联的流水门槛）
  │
  ├→ [内部] 创建兑换记录
  │
  └→ 返回(兑换成功, 各金额明细)
```

**跨模块调用：无。** 纯内部操作，这是最独立的业务链路。

**内部调用关系：** 内部使用了与 R-1 CreditWallet 和 R-2 DebitWallet 相同的余额操作核心逻辑。建议在实现时抽取公共的余额操作方法，C-6 和 R-1/R-2 共用。

依据：[需求 3.7] 兑换范围、汇率显示、兑换计算、赠送+流水要求。

---

## C-7: GetWithdrawMethods（获取提现方式列表）

**触发：** 用户进入提现页面

**调用链路：**
```
客户端请求(userID, currencyCode)
  │
  ├→ [KYC RPC] QueryKycStatus(userID)
  │    → 返回：认证状态(未认证/已认证) + KYC姓名
  │
  ├→ [财务RPC] GetPaymentMethods(currencyCode, type=withdraw)
  │    → 返回：该币种全量提现方式列表
  │
  ├→ [内部] 按 KYC 状态过滤
  │    → 未认证 → 只保留 USDT 提现
  │    → 已认证 → 保留全部
  │
  ├→ [内部] 按充值路由规则过滤
  │    → 查询用户历史充值记录的充值方式类型
  │    → 法币充值 → 法币提现
  │    → USDT充值 → USDT提现
  │    → 法币+USDT → 法币提现
  │    → 法币→BSB → 法币提现
  │
  └→ 返回(过滤后的提现方式列表)
```

**跨模块调用：ser-kyc × 1 + 财务管理 × 1 = 2 次。**

**内部依赖：** 用户历史充值记录（判断充值路由规则）。

依据：[需求 3.8.1] KYC与提现关系；[需求 3.8.2] 提现规则（基于币种路由）。

---

## C-8: CreateWithdrawOrder（创建提现订单）

**触发：** 用户填写提现信息后点击确认

**调用链路：**
```
客户端请求(userID, currencyCode, withdrawMethodID, amount, accountInfo)
  │
  ├→ [KYC RPC] QueryKycStatus(userID)
  │    → 校验：法币提现必须已认证
  │
  ├→ [User RPC] GetUserInfo(userID)
  │    → 获取用户基本信息
  │
  ├→ [内部] 校验提现账户持有人姓名 = KYC 姓名
  │    → 忽略大小写、忽略多余空格比对
  │    → 不一致 → 拒绝
  │
  ├→ [内部] 校验提现限额
  │    → 查询用户今日已提现总额
  │    → 今日已提 + 本次金额 > 每日限额 → 拒绝
  │    → 本次金额 < 单笔最低 → 拒绝
  │
  ├→ [内部] 校验余额
  │    → 查询中心钱包可用余额
  │    → 可用余额 < 提现金额 → 拒绝
  │
  ├→ [内部] 冻结余额（FreezeBalance 核心逻辑）
  │    → 中心钱包可用余额 -= amount
  │    → 中心钱包冻结余额 += amount
  │    → 写账变流水：类型=提现冻结
  │
  ├→ [内部] 创建提现订单
  │    → 订单号: T + 16位随机数
  │    → 状态: 待风控审核
  │    → 记录：金额, 币种, 提现方式, 账户信息, 汇率快照, 手续费
  │
  └→ 返回(订单号, 状态=待审核)
```

**跨模块调用：ser-kyc × 1 + ser-user × 1 = 2 次。**

**内部调用关系：** 使用了与 R-3 FreezeBalance 相同的冻结逻辑。

依据：[需求 3.8.1] KYC校验；[需求 3.8.3-3.8.6] 提现流程和校验规则；[需求 5.5] "创建订单 → 立即冻结用户钱包金额"。

---

## C-9: GetWithdrawAccount（获取已绑定提现账户）

```
客户端请求(userID, currencyCode, withdrawMethodType)
  → [内部] 查询提现账户表(userID + currencyCode + methodType)
  → 返回(账户信息 或 空，标识是否首次)
```

**跨模块调用：无。**

---

## C-10: SaveWithdrawAccount（保存提现账户）

```
客户端请求(userID, currencyCode, withdrawMethodType, accountInfo)
  → [内部] 检查是否已存在账户（用户不可自行修改）
      → 已存在 → 拒绝
  → [KYC RPC] QueryKycStatus(userID) → 获取 KYC 姓名
  → [内部] 校验 accountInfo.holderName ≈ KYC姓名（模糊匹配）
  → [内部] 保存账户信息到提现账户表
  → 返回(保存成功)
```

**跨模块调用：ser-kyc × 1。**

依据：[需求 3.8.4] "账户持有人名称必须与 KYC 认证姓名一致"；[需求 3.10] "用户不允许自行修改提现账户"。

---

## C-11: GetTransactionList（账变记录列表）

```
客户端请求(userID, currencyCode, tabType[充值/提现/兑换/奖励], page, pageSize)
  → [内部] 查询账变记录表(userID + currencyCode + type + 分页)
  → [内部] 按币种精度格式化金额
  → 返回(列表数据)
```

**跨模块调用：无。** 纯读操作。

---

## C-12: GetTransactionDetail（账变记录详情）

```
客户端请求(userID, orderNo)
  → [内部] 查询订单/记录详情(orderNo)
  → [内部] 校验 orderNo 属于该 userID
  → 返回(详情数据：订单号, 金额, 状态, 时间, 方式, 失败原因等)
```

**跨模块调用：无。**

---

## C-13 ~ C-19：简要链路

| 接口 | 调用链路要点 | 跨模块调用 |
|------|------------|----------|
| C-13 GetCurrentRate | [内部]币种配置.获取指定币种汇率(实时/平台/入款/出款) → 格式化返回 | 无 |
| C-14 CheckDuplicateDeposit | [内部]查询订单表(同user+同currency+同amount+状态=待支付/处理中) → 返回count+策略 | 无 |
| C-15 GetBonusActivities | [财务RPC或活动模块]获取匹配当前币种+金额的活动列表 → 返回 | 财务/活动 × 1 |
| C-16 GetDepositOrderStatus | [内部]查询充值订单表(orderNo) → 返回当前状态 | 无 |
| C-17 GetWithdrawLimits | [财务RPC]获取提现方式配置(含限额/手续费) 或 读缓存 → 返回 | 财务 × 1 |
| C-18 CancelDepositOrder | [内部]校验订单状态=待支付 → 更新为已取消 | 无 |
| C-19 GetAvailableCurrencies | [内部]币种配置.查询已启用币种列表(含排序/图标) → 返回 | 无 |

---

# 第二部分：币种配置 B 端接口调用链路

## B-1 ~ B-6：全部为纯内部操作

| 接口 | 调用链路 | 跨模块调用 |
|------|---------|----------|
| B-1 GetCurrencyList | [内部]查询币种配置表(筛选条件) + [内部]关联最新汇率数据 → 返回列表 | 无 |
| B-2 EditCurrency | [内部]校验(浮动率小数位/图标文件大小≤5KB/阈值范围) → 更新币种配置记录 → 返回 | 无 |
| B-3 ToggleCurrencyStatus | [内部]校验不是基准币种(基准不可禁用) → 更新状态(启用/禁用) → 返回 | 无 |
| B-4 SetBaseCurrency | [内部]校验基准币种未设置(只能设一次) → 设置基准币种 → 返回 | 无 |
| B-5 GetExchangeRateLogs | [内部]查询汇率日志表(日期范围+币种筛选+分页) → 返回列表 | 无 |
| B-6 GetCurrencyDetail | [内部]查询单个币种详情 → 返回 | 无 |

**币种配置模块不调用任何外部服务，也不调用钱包余额相关操作。它是纯粹的基础数据管理。**

---

# 第三部分：钱包 RPC 接口调用链路（供他方调用）

这些接口被财务模块、游戏模块、直播模块等外部服务调用。它们的内部实现链路如下：

## R-1: CreditWallet（入账/加款）

```
调用方请求(userID, currencyCode, walletType, amount, orderNo, opType, auditMultiplier?, auditTurnover?)
  │
  ├→ [内部] 幂等校验：orderNo 是否已处理过
  │    → 已处理 → 直接返回成功（防重复入账）
  │
  ├→ [内部] 参数校验：amount > 0, walletType 合法
  │
  ├→ [内部] 钱包余额表.update: 指定子钱包可用余额 += amount
  │
  ├→ [内部] 写账变流水记录(userID, currencyCode, walletType, +amount, orderNo, opType)
  │
  ├→ [内部] 如有稽核信息：记录稽核流水要求(auditMultiplier, auditTurnover)
  │
  └→ 返回(成功, 新余额)
```

**被谁调用：**
- 财务模块：充值成功入账、补单入账、人工加款
- 游戏模块：投注返奖
- 活动模块：活动奖励发放
- 直播模块：主播收入入账

**关键点：幂等性。** 同一个 orderNo 只能入账一次，重复调用直接返回成功。这是 [需求 3.10] "充值回调必须实现幂等处理"的核心保障。

---

## R-2: DebitWallet（扣款/减款）

```
调用方请求(userID, currencyCode, walletType, amount, orderNo, opType)
  │
  ├→ [内部] 幂等校验：orderNo 是否已处理过
  │
  ├→ [内部] 查询指定子钱包可用余额
  │    → 可用余额 < amount → 返回余额不足
  │
  ├→ [内部] 钱包余额表.update: 指定子钱包可用余额 -= amount
  │
  ├→ [内部] 写账变流水记录
  │
  └→ 返回(成功, 新余额)
```

**被谁调用：**
- 财务模块：人工减款
- 游戏模块：投注扣款（指定钱包）
- 直播/礼物模块：购买扣款（指定钱包）

---

## R-3: FreezeBalance（冻结余额）

```
调用方请求(userID, currencyCode, walletType, amount, orderNo)
  │
  ├→ [内部] 查询指定子钱包可用余额
  │    → 可用余额 < amount → 返回余额不足
  │
  ├→ [内部] 钱包余额表.update:
  │    → 可用余额 -= amount
  │    → 冻结余额 += amount
  │
  ├→ [内部] 写账变流水记录(类型=冻结)
  │
  └→ 返回(成功)
```

**被谁调用：** 提现订单创建时（C-8 内部调用，不经过 RPC）。如果提现订单创建在财务模块侧，则财务通过 RPC 调用。

---

## R-4: UnfreezeBalance（解冻余额）

```
调用方请求(userID, currencyCode, walletType, amount, orderNo)
  │
  ├→ [内部] 校验冻结余额 >= amount
  │
  ├→ [内部] 钱包余额表.update:
  │    → 冻结余额 -= amount
  │    → 可用余额 += amount
  │
  ├→ [内部] 写账变流水记录(类型=解冻)
  │
  └→ 返回(成功)
```

**被谁调用：** 财务模块在提现被驳回（风控驳回/财务驳回/出款驳回）时调用。

---

## R-5: DeductFrozenAmount（扣除冻结金额）

```
调用方请求(userID, currencyCode, walletType, amount, orderNo)
  │
  ├→ [内部] 校验冻结余额 >= amount
  │
  ├→ [内部] 钱包余额表.update:
  │    → 冻结余额 -= amount
  │    → （不加回可用余额，钱真的出去了）
  │
  ├→ [内部] 写账变流水记录(类型=提现扣除)
  │
  └→ 返回(成功)
```

**被谁调用：** 财务模块在出款成功时调用。

**R-3/R-4/R-5 的配套关系：**
```
正常提现：FreezeBalance → (审核通过，出款成功) → DeductFrozenAmount
驳回提现：FreezeBalance → (审核驳回) → UnfreezeBalance
出款失败：FreezeBalance → (出款失败) → 保持冻结（不调任何接口，等人工）
```

---

## R-6: QueryWalletBalance（查询余额）

```
调用方请求(userID, currencyCode)
  → [内部] 查询所有子钱包余额(可用 + 冻结)
  → 返回(中心{可用,冻结}, 奖励{可用,冻结}, 主播{可用,冻结}, 代理{可用,冻结})
```

**被谁调用：** 财务模块在人工加款/减款/补单的表单页面调用，展示用户当前余额。

---

## R-7: DebitByPriority（按消费优先级扣款）

```
调用方请求(userID, currencyCode, amount, orderNo, opType)
  │
  ├→ [内部] 查询中心钱包可用余额 centerBalance
  ├→ [内部] 查询奖励钱包可用余额 rewardBalance
  │
  ├→ [内部] 判断扣款分配：
  │    → centerBalance >= amount
  │        → 全从中心扣：center -= amount
  │    → centerBalance < amount 且 centerBalance + rewardBalance >= amount
  │        → 中心扣完：center -= centerBalance
  │        → 剩余从奖励扣：reward -= (amount - centerBalance)
  │    → 总余额不足 → 返回余额不足
  │
  ├→ [内部] 写账变流水（可能 1~2 条，按实际扣款的子钱包分别记录）
  │
  └→ 返回(成功, 扣款明细{中心扣了多少, 奖励扣了多少})
```

**被谁调用：** 游戏投注、礼物购买、道具购买等消费场景。调用方不需要知道钱从哪个钱包扣，钱包内部按优先级自动分配。

**返回的扣款明细很重要**：因为 [需求 4.1.2] "中心钱包投注返奖归中心，奖励钱包投注返奖归奖励，混合投注按比例拆分"。调用方拿到明细后，返奖时需按此比例分别调 CreditWallet。

---

## R-8 ~ R-12：简要链路

| 接口 | 调用链路 | 说明 |
|------|---------|------|
| R-8 GetCurrencyConfig | [内部]币种配置表.查询已启用列表 → 返回 | 供其他模块获取平台支持的币种信息 |
| R-9 GetExchangeRate | [内部]币种配置.获取指定币种汇率(实时/平台/入款/出款) → 返回 | 供其他模块获取汇率 |
| R-10 ConvertAmount | [内部]获取汇率 → 计算换算 → 返回结果 | 供其他模块做金额换算 |
| R-11 BatchQueryBalance | [内部]批量查询多用户余额 → 返回 | 运营报表等场景 |
| R-12 GetWalletFlowList | [内部]查询指定用户账变流水(分页) → 返回 | B端查看用户资金明细 |

---

# 第四部分：定时任务调用链路

## T-1: ExchangeRateUpdate（汇率定时更新）

```
定时触发(可配置频率)
  │
  ├→ [三方API] 调用汇率API-1 → 获取各币种对USDT的汇率
  ├→ [三方API] 调用汇率API-2 → 获取各币种对USDT的汇率
  ├→ [三方API] 调用汇率API-3 → 获取各币种对USDT的汇率
  │    （3个API并行调用，提高效率+容错）
  │
  ├→ [内部] 计算实时汇率 = 三者平均值
  │    → 如某个API失败/超时：用其余2个的平均值
  │    → 如2个以上失败：本轮跳过，保持旧汇率
  │
  ├→ [内部] 遍历每个已启用币种（BSB除外，BSB汇率固定）：
  │    │
  │    ├→ 平台汇率是否为空（首次）？
  │    │    → 是 → 直接设实时汇率为平台汇率
  │    │    → 否 → 计算偏差 = |实时 - 平台| / 实时 × 100%
  │    │
  │    ├→ 偏差 >= 阈值？
  │    │    → 是 →
  │    │        → 更新平台汇率 = 实时汇率
  │    │        → 更新入款汇率 = 实时 × (1 + 浮动%)
  │    │        → 更新出款汇率 = 实时 × (1 - 浮动%)
  │    │        → [内部] 写汇率日志记录
  │    │    → 否 → 不更新，只更新实时汇率展示值
  │    │
  │    └→ 下一个币种
  │
  └→ 完成
```

**跨模块调用：三方汇率API × 3（外部HTTP）。不调钱包余额接口。**

依据：[需求 4.5.2] 从3个三方API取平均值；[需求 4.5.3] 阈值更新机制。

---

## T-2: DepositOrderTimeout（充值订单超时）

```
定时触发(如每分钟)
  → [内部] 查询状态=待支付 且 创建时间 < (当前时间 - 超时阈值)的订单
  → [内部] 批量更新状态为"支付失败/已过期"
  → 完成
```

## T-3: BalanceReconciliation（余额对账）

```
定时触发(如每日凌晨)
  → [内部] 遍历每个用户+币种+子钱包：
      → 计算：该子钱包所有入账流水之和 - 所有出账流水之和 = 理论余额
      → 对比：理论余额 vs 当前实际余额
      → 不一致 → 记录差异报告，告警
  → 完成
```

---

# 第五部分：跨模块完整调用链路（财务侧视角）

以下从财务管理模块的视角，描述它在各业务场景中如何调用钱包 RPC。这些不是我方实现的代码，但我方必须清楚这些调用会在什么时机、带什么参数到达。

## 场景一：充值成功入账

```
三方通道 --回调--> 财务模块.代收回调接口
  → 财务：验签(通道密钥)
  → 财务：更新通道订单状态
  → 财务：更新平台订单状态 = 支付成功
  → 财务 → [钱包RPC] CreditWallet
      参数: userID, currencyCode, walletType=中心, amount=订单金额, orderNo=C+16位, opType=充值
  → 如有活动奖金：
      财务 → [钱包RPC] CreditWallet
          参数: userID, currencyCode, walletType=奖励, amount=奖金金额, orderNo=同上, opType=充值奖金, auditMultiplier=N, auditTurnover=计算值
```

**钱包侧接收到的调用：CreditWallet × 1~2 次（有奖金则2次）。**

---

## 场景二：提现审核驳回

```
运营人员在财务B端点击"驳回"
  → 财务：更新提现订单状态 = 风控驳回/财务驳回
  → 财务 → [钱包RPC] UnfreezeBalance
      参数: userID, currencyCode, walletType=中心, amount=提现金额, orderNo=T+16位
```

**钱包侧接收到的调用：UnfreezeBalance × 1 次。**

---

## 场景三：出款成功

```
三方通道 --回调--> 财务模块.代付回调接口
  → 财务：验签
  → 财务：更新提现订单状态 = 出款成功
  → 财务 → [钱包RPC] DeductFrozenAmount
      参数: userID, currencyCode, walletType=中心, amount=提现金额, orderNo=T+16位
```

**钱包侧接收到的调用：DeductFrozenAmount × 1 次。**

---

## 场景四：出款失败

```
三方通道 --回调--> 财务模块.代付回调接口
  → 财务：更新提现订单状态 = 出款失败
  → 财务：不调钱包任何接口
  → 冻结金额保持不动，等人工处理
```

**钱包侧接收到的调用：无。** 冻结余额持续存在直到人工介入。

---

## 场景五：充值补单

```
运营在财务B端创建补单
  → 财务 → [钱包RPC] QueryWalletBalance(userID, currencyCode) → 展示余额
  → 运营填写补单信息 → 提交 → 审核通过
  → 财务 → [钱包RPC] CreditWallet
      参数: userID, currencyCode, walletType=中心, amount=补单金额, orderNo=B+16位, opType=补单
  → 如有补单赠送：
      财务 → [钱包RPC] CreditWallet
          参数: walletType=奖励, amount=赠送金额, auditMultiplier, auditTurnover
```

**钱包侧接收到的调用：QueryWalletBalance × 1 + CreditWallet × 1~2。**

---

## 场景六：人工加款

```
运营在财务B端打开加款页面
  → 财务 → [钱包RPC] QueryWalletBalance(userID, currencyCode) → 展示当前各钱包余额
  → 运营填写(中心+X, 奖励+Y, 主播+Z, 代理+W) → 提交 → 审核通过
  → 对每个金额>0的子钱包：
      财务 → [钱包RPC] CreditWallet(walletType=中心, amount=X, orderNo=A+16位, opType=加款, auditMultiplier, auditTurnover)
      财务 → [钱包RPC] CreditWallet(walletType=奖励, amount=Y, ...)
      财务 → [钱包RPC] CreditWallet(walletType=主播, amount=Z, ...)
      财务 → [钱包RPC] CreditWallet(walletType=代理, amount=W, ...)
```

**钱包侧接收到的调用：QueryWalletBalance × 1 + CreditWallet × 1~4 次（取决于几个子钱包有金额）。**

---

## 场景七：人工减款

```
运营在财务B端打开减款页面
  → 财务 → [钱包RPC] QueryWalletBalance(userID, currencyCode) → 展示当前各钱包余额
  → 运营填写(中心-X, 奖励-Y, ...) → 前端校验减款后余额>=0 → 提交 → 审核通过
  → 对每个金额>0的子钱包：
      财务 → [钱包RPC] DebitWallet(walletType=中心, amount=X, orderNo=M+16位, opType=减款)
      财务 → [钱包RPC] DebitWallet(walletType=奖励, amount=Y, ...)
```

**钱包侧接收到的调用：QueryWalletBalance × 1 + DebitWallet × 1~4 次。**

**后端必须再次校验余额充足，不能信任前端的校验结果。**

---

# 第六部分：接口间内部复用关系

以下梳理 ser-wallet 服务内部，哪些接口共用了底层逻辑，建议在实现时抽取为公共方法。

## 6.1 余额操作核心方法（最底层，被多个接口复用）

```
内部公共方法                    被谁调用
─────────────────────────────────────────────
addBalance(user,cur,wallet,amt)  → R-1 CreditWallet
                                 → C-6 CreateExchange(BSB入账部分)

deductBalance(user,cur,wallet,amt) → R-2 DebitWallet
                                    → C-6 CreateExchange(法币扣减部分)
                                    → R-7 DebitByPriority(内部拆分后调用)

freezeBalance(user,cur,wallet,amt) → R-3 FreezeBalance
                                    → C-8 CreateWithdrawOrder(内部冻结)

unfreezeBalance(user,cur,wallet,amt) → R-4 UnfreezeBalance

deductFrozen(user,cur,wallet,amt) → R-5 DeductFrozenAmount
```

## 6.2 账变流水记录方法

```
内部公共方法                    被谁调用
─────────────────────────────────────────────
writeTransactionLog(...)        → addBalance 内调用
                                → deductBalance 内调用
                                → freezeBalance 内调用
                                → unfreezeBalance 内调用
                                → deductFrozen 内调用
                                → C-6 CreateExchange(兑换记录)
```

**所有改变余额的操作，必须同时写流水。这两个操作应该在同一个数据库事务中。**

## 6.3 币种配置读取方法

```
内部公共方法                        被谁调用
────────────────────────────────────────────────
getCurrencyConfig(currencyCode)   → C-1 GetWalletOverview(币种校验+显示规则)
                                  → C-2 GetDepositMethods(币种校验)
                                  → C-3 CreateDepositOrder(币种校验)
                                  → C-5/C-6 兑换(币种校验)
                                  → C-7/C-8 提现(币种校验)
                                  → R-8 GetCurrencyConfig(对外暴露)
                                  → 几乎所有接口都用

getExchangeRate(currencyCode)     → C-3 CreateDepositOrder(汇率快照)
                                  → C-4 GetUSDTDepositInfo(USDT换算)
                                  → C-5/C-6 兑换(汇率计算)
                                  → C-8 CreateWithdrawOrder(汇率快照)
                                  → C-13 GetCurrentRate(直接返回)
                                  → R-9 GetExchangeRate(对外暴露)
                                  → R-10 ConvertAmount(换算计算)
                                  → T-1 ExchangeRateUpdate(更新汇率)
```

## 6.4 幂等校验方法

```
内部公共方法                    被谁调用
─────────────────────────────────────────────
checkIdempotent(orderNo)        → R-1 CreditWallet
                                → R-2 DebitWallet
                                → R-3 FreezeBalance
                                → R-5 DeductFrozenAmount
```

**所有涉及资金变动的 RPC 接口都必须做幂等校验。**

---

# 第七部分：调用链路全景表

以下用表格汇总每个接口的"调了谁 + 被谁调"的完整关系。

## 7.1 C 端接口调用全景

| 接口 | 内部调用的公共方法 | 调用的外部RPC | 被谁调用 |
|------|------------------|-------------|---------|
| C-1 GetWalletOverview | getCurrencyConfig, 查余额表 | 无 | 客户端 |
| C-2 GetDepositMethods | getCurrencyConfig | 财务.GetPaymentMethods | 客户端 |
| C-3 CreateDepositOrder | getCurrencyConfig, getExchangeRate, 查订单表(重复检测), 写订单表 | 财务.MatchAndCreateChannelOrder | 客户端 |
| C-4 GetUSDTDepositInfo | getExchangeRate | 无 | 客户端(Open) |
| C-5 GetExchangePreview | getExchangeRate, 查余额表 | 无 | 客户端 |
| C-6 CreateExchange | getExchangeRate, deductBalance, addBalance×2, writeLog | 无 | 客户端 |
| C-7 GetWithdrawMethods | getCurrencyConfig, 查充值记录表 | KYC.QueryKycStatus, 财务.GetPaymentMethods | 客户端 |
| C-8 CreateWithdrawOrder | getCurrencyConfig, freezeBalance, writeLog, 写订单表 | KYC.QueryKycStatus, User.GetUserInfo | 客户端 |
| C-9 GetWithdrawAccount | 查提现账户表 | 无 | 客户端 |
| C-10 SaveWithdrawAccount | 写提现账户表 | KYC.QueryKycStatus | 客户端 |
| C-11 GetTransactionList | 查账变记录表 | 无 | 客户端 |
| C-12 GetTransactionDetail | 查订单/记录表 | 无 | 客户端 |
| C-13 GetCurrentRate | getExchangeRate | 无 | 客户端 |
| C-14 CheckDuplicateDeposit | 查订单表 | 无 | 客户端(或C-3内部) |
| C-15 GetBonusActivities | — | 财务/活动模块 | 客户端 |
| C-16 GetDepositOrderStatus | 查订单表 | 无 | 客户端 |
| C-17 GetWithdrawLimits | — | 财务.GetPaymentMethods | 客户端 |
| C-18 CancelDepositOrder | 更新订单表 | 无 | 客户端 |
| C-19 GetAvailableCurrencies | getCurrencyConfig | 无 | 客户端 |

## 7.2 RPC 接口被调用全景

| 接口 | 内部调用的公共方法 | 被谁调用 | 调用频率预估 |
|------|------------------|---------|------------|
| R-1 CreditWallet | checkIdempotent, addBalance, writeLog | 财务(充值/补单/加款), 游戏(返奖), 活动(奖励), 直播(主播收入) | 高频 |
| R-2 DebitWallet | checkIdempotent, deductBalance, writeLog | 财务(减款), 游戏(投注指定钱包) | 中频 |
| R-3 FreezeBalance | freezeBalance, writeLog | C-8内部 / 财务(如提现归财务创建) | 中频 |
| R-4 UnfreezeBalance | unfreezeBalance, writeLog | 财务(提现驳回) | 低频 |
| R-5 DeductFrozenAmount | checkIdempotent, deductFrozen, writeLog | 财务(出款成功) | 低频 |
| R-6 QueryWalletBalance | 查余额表 | 财务(加减款/补单表单) | 中频 |
| R-7 DebitByPriority | deductBalance×1~2, writeLog×1~2 | 游戏(投注), 礼物(购买), 道具(购买) | 高频 |
| R-8 GetCurrencyConfig | getCurrencyConfig | 所有需要币种信息的模块 | 高频(建议缓存) |
| R-9 GetExchangeRate | getExchangeRate | 所有需要汇率的模块 | 高频(建议缓存) |
| R-10 ConvertAmount | getExchangeRate + 计算 | 财务统计报表 | 中频 |

## 7.3 跨模块调用次数汇总

| 业务场景 | 钱包→财务 | 钱包→KYC | 钱包→User | 财务→钱包 |
|---------|----------|---------|----------|----------|
| 充值展示 | 1次(支付方式) | — | — | — |
| 充值下单 | 1次(通道匹配) | — | — | — |
| 充值成功入账 | — | — | — | 1~2次(CreditWallet) |
| 提现展示 | 1次(提现方式) | 1次(KYC状态) | — | — |
| 提现下单 | — | 1次(KYC状态) | 1次(用户信息) | — |
| 提现驳回 | — | — | — | 1次(Unfreeze) |
| 出款成功 | — | — | — | 1次(DeductFrozen) |
| 补单 | — | — | — | 1次(QueryBalance) + 1~2次(Credit) |
| 人工加款 | — | — | — | 1次(QueryBalance) + 1~4次(Credit) |
| 人工减款 | — | — | — | 1次(QueryBalance) + 1~4次(Debit) |
| 兑换 | — | — | — | — |
| 保存提现账户 | — | 1次(KYC姓名) | — | — |