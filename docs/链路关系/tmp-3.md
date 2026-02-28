# ser-wallet 接口调用链路关系（模块内 + 模块间）

> 梳理时间：2026-02-26
> 梳理依据：需求文档 + 产品原型 + 接口评估（接口评估/tmp-3.md）+ 依赖关系分析（依赖关系/tmp-3.md）
> 梳理目的：站在钱包模块负责人视角，清楚每个接口的内部调用链路和跨模块调用关系
> 箭头含义：`→` 表示调用/依赖方向，`>>` 表示顺序步骤

---

## 一、C端接口调用链路（gate-font → ser-wallet → 内部/外部）

### C1 GetWalletHome（钱包首页）

**触发场景：** 用户打开钱包首页，或切换币种后刷新
**需求依据：** 钱包需求4.1-4.2，原型2-1个人中心，原型2-2币种切换

```
C端App
  → gate-font（路由转发）
    → ser-wallet.GetWalletHome(userID, currencyCode)
      → [内部] 币种配置：校验 currencyCode 是否启用
      → [内部] 币种配置：获取币种展示信息（名称/图标/符号/千分位/小数位规则）
      → [内部] 钱包核心：查询该用户该币种的中心钱包余额
      → [内部] 钱包核心：查询该用户该币种的奖励钱包余额
      → [内部] 计算：资金钱包余额 = 中心余额 + 奖励余额
      → [内部] 格式化：按币种展示规则格式化金额（VND/IDR整数，THB/BSB两位小数）
    → 返回：余额、币种信息、钱包分层明细
```

**调用关系总结：**
- 跨模块调用：无
- 模块内调用：币种配置（读币种状态+展示规则）→ 钱包核心（读余额）
- 特点：纯读操作，高频调用，适合缓存

---

### C2 GetCurrencyList（币种列表）

**触发场景：** 用户点击币种切换按钮
**需求依据：** 钱包需求4.2，原型2-2币种切换列表

```
C端App
  → gate-font
    → ser-wallet.GetCurrencyList(userID)
      → [内部] 币种配置：查询所有已启用的币种列表（代码/名称/图标/符号）
      → [内部] 钱包核心：批量查询该用户在每个币种下的资金钱包余额
      → [内部] 组装：每个币种附带对应余额
    → 返回：币种列表（每项含：币种信息 + 该币种余额）
```

**调用关系总结：**
- 跨模块调用：无
- 模块内调用：币种配置（读币种列表）→ 钱包核心（批量读余额）
- 特点：纯读操作，需要遍历多个币种，注意性能

---

### C3 GetDepositMethods（充值方式列表）

**触发场景：** 用户进入充值页面
**需求依据：** 钱包需求5.1规则说明，原型3-1至3-4充值中心

```
C端App
  → gate-font
    → ser-wallet.GetDepositMethods(userID, currencyCode)
      → [内部] 币种配置：校验币种启用状态
      → [跨模块 RPC] 财务模块.GetPaymentConfig(currencyCode, type="deposit")
        ← 返回：该币种下可用充值方式列表（方式名/图标/档位金额/金额范围/排序）
      → [内部] 币种配置：如果是BSB，获取入款汇率（用于展示"1BSB=X法币"）
      → [内部] 过滤/排序：按排序字段排列，标记默认充值方式和默认最小档位
    → 返回：充值方式列表 + 档位 + 汇率信息（BSB时）
```

**调用关系总结：**
- 跨模块调用：**财务模块.GetPaymentConfig** — 充值方式和档位完全来自财务配置
- 模块内调用：币种配置（校验状态 + BSB入款汇率）
- 关键依赖：如果财务模块接口不可用，充值页无法展示任何充值方式

---

### C4 CreateDepositOrder（创建充值订单）

**触发场景：** 用户选择充值方式和金额后，点击"去充值"
**需求依据：** 钱包需求5.2-5.4，财务需求2.3充值流程

```
C端App
  → gate-font
    → ser-wallet.CreateDepositOrder(userID, currencyCode, amount, method, bonusActivityID?)
      → [内部] 步骤1：校验基础参数（币种启用、金额范围合法）
      → [内部] 步骤2：重复转账检测（调用 CheckDuplicateDeposit 逻辑）
        → [内部] 查询该用户+该币种+该金额的待支付/进行中订单数
        → 1-2笔：返回提示信息（前端弹窗，用户可选继续）
        → 3笔：强制拦截，直接返回错误
      → [内部] 步骤3：如果是BSB充值
        → [内部] 币种配置：获取入款汇率（锁定当时汇率作为快照）
        → [内部] 计算实际支付金额（按入款汇率换算）
      → [内部] 步骤4：如果选择了奖金活动
        → [跨模块 RPC] 活动模块.ValidateBonus(activityID, amount)
          ← 返回：奖金金额、流水倍数要求
      → [内部] 步骤5：创建本地充值订单记录（待支付状态）
      → [跨模块 RPC] 步骤6：财务模块.CreatePayOrder(订单信息)
        → 财务内部：轮询匹配通道（成功率/时间/预警三系数评分排序）
        → 财务内部：向三方通道创建代收订单
        ← 返回：通道订单号 + 支付信息（三方承载页URL / USDT收款地址+二维码）
      → [内部] 步骤7：更新本地订单，关联通道订单号
    → 返回：订单信息 + 支付跳转信息

--- 异步后续链路（非本接口内，但属于充值链路的后半段）---

三方通道
  → 财务模块（支付结果回调）
    → 财务内部：更新充值订单状态为"支付成功"
    → [跨模块 RPC] ser-wallet.CreditWallet(userID, currencyCode, amount, orderNo)
      → [内部] 钱包核心：中心钱包余额 += 充值金额
      → [内部] 如果有奖金活动：奖励钱包余额 += 奖金金额
      → [内部] 写入钱包流水记录
```

**调用关系总结：**
- 跨模块调用（同步）：
  - 活动模块.ValidateBonus — 校验奖金活动有效性（可选，有选奖金时才调）
  - **财务模块.CreatePayOrder** — 创建通道订单（必须，核心依赖）
- 跨模块调用（异步后续）：
  - 财务模块回调 → ser-wallet.CreditWallet — 充值成功上账
- 模块内调用：重复转账检测 → 币种配置（汇率）→ 创建本地订单 → 更新订单
- 关键点：这是钱包模块中**同步调用链路最长**的接口

---

### C5 CheckDuplicateDeposit（重复转账检测）

**触发场景：** 创建充值订单前的前置校验（内嵌在C4中，也可独立调用）
**需求依据：** 钱包需求5.5，原型7-3/7-4

```
C端App
  → gate-font
    → ser-wallet.CheckDuplicateDeposit(userID, currencyCode, amount)
      → [内部] 查询条件：同一userID + 同一currencyCode + 同一amount
      → [内部] 筛选状态："待支付"或"进行中"的充值订单
      → [内部] 统计匹配订单数
        → 0笔：通过，可继续
        → 1-2笔：返回提示（含订单数），前端展示弹窗"是否继续"
        → ≥3笔：拦截，不允许继续创建
    → 返回：检测结果（通过/提示/拦截）+ 匹配订单数
```

**调用关系总结：**
- 跨模块调用：无
- 模块内调用：纯粹查询自己的充值订单表
- 适用范围：银行转账、电子钱包支付（不适用于USDT链上充值）
- 特点：完全独立，不依赖任何外部模块

---

### C6 GetBonusActivities（可用奖金活动列表）

**触发场景：** 充值页面加载时并行请求，或用户点击奖金选择区域
**需求依据：** 钱包需求5.6，原型6-1/6-2

```
C端App
  → gate-font
    → ser-wallet.GetBonusActivities(userID, currencyCode)
      → [跨模块 RPC] 活动模块.GetAvailableBonuses(userID, currencyCode, scene="deposit")
        ← 返回：可用奖金活动列表（活动ID/名称/奖金比例/最低充值门槛/流水倍数）
      → [内部] 排序/过滤：按匹配度排序，找出"当前最优匹配活动"
    → 返回：奖金活动列表 + 默认推荐活动 + "无奖金"选项
```

**调用关系总结：**
- 跨模块调用：**活动模块.GetAvailableBonuses** — 奖金数据完全来自活动模块
- 模块内调用：无实质业务逻辑，只是透传+排序
- 降级策略：活动模块不可用时，应降级为"无可用奖金"，不应阻断充值流程

---

### C7 GetExchangeRate（兑换汇率查询）

**触发场景：** 用户进入兑换页面
**需求依据：** 钱包需求6.2，原型8

```
C端App
  → gate-font
    → ser-wallet.GetExchangeRate(currencyCode)
      → [内部] 币种配置：校验币种启用状态
      → [内部] 币种配置：获取该法币对BSB的平台汇率
      → [内部] 计算展示：统一口径"1 BSB = X 法币"
      → [内部] 获取赠送比例配置（如10%）
    → 返回：汇率（1BSB=X法币）、赠送比例、格式化规则
```

**调用关系总结：**
- 跨模块调用：无
- 模块内调用：币种配置（平台汇率）
- 特点：纯读操作，完全在ser-wallet内部闭环

---

### C8 DoExchange（执行兑换）

**触发场景：** 用户确认兑换，点击"兑换"按钮
**需求依据：** 钱包需求6.3-6.4，原型8

```
C端App
  → gate-font
    → ser-wallet.DoExchange(userID, currencyCode, amount)
      → [内部] 步骤1：校验币种启用状态（必须是法币，BSB下无兑换入口）
      → [内部] 步骤2：获取平台汇率并锁定（防止计算过程中汇率变动）
      → [内部] 步骤3：校验用户法币中心钱包余额 >= amount
        → 不足：返回"已达当前可兑换上限"
      → [内部] 步骤4：计算
        → 兑换获得BSB = amount / 汇率
        → 赠送BSB = 兑换获得BSB × 赠送比例
        → 实际到账BSB = 兑换获得 + 赠送
      → [内部] 步骤5：**同一数据库事务内执行**
        → 法币中心钱包余额 -= amount
        → BSB中心钱包余额 += 兑换获得BSB
        → BSB奖励钱包余额 += 赠送BSB（需完成X倍流水方可提现）
        → 写入兑换流水记录（含汇率快照、赠送信息、流水倍数要求）
      → [内部] 步骤6：返回兑换结果
    → 返回：兑换成功（兑换获得/赠送/实际到账/使用汇率）
```

**调用关系总结：**
- 跨模块调用：**无**
- 模块内调用：币种配置（汇率）→ 钱包核心（余额校验 + 多钱包余额变更）
- 关键点：这是**唯一不依赖任何外部模块**的C端写操作
- 事务要求：法币扣减 + BSB中心入账 + BSB奖励入账必须在同一事务内
- 适合作为首个端到端联调验证的场景

---

### C9 GetWithdrawMethods（提现方式列表）

**触发场景：** 用户进入提现页面
**需求依据：** 钱包需求7.1-7.5，原型9-4、10-1至10-4

```
C端App
  → gate-font
    → ser-wallet.GetWithdrawMethods(userID, currencyCode)
      → [跨模块 RPC] 步骤1：ser-kyc.KycQuery(userID)
        ← 返回：KYC认证状态（未认证/审核中/已认证）
      → [内部] 步骤2：根据KYC状态过滤可用提现方式
        → 未认证：仅USDT提现可用
        → 已认证：全渠道（银行卡/电子钱包/USDT等）
      → [内部] 步骤3：查询用户充值历史，确定提现币种规则
        → 法币充值 → 法币提现
        → USDT充值 → USDT提现
        → 法币+USDT混合充值 → 法币提现
      → [跨模块 RPC] 步骤4：财务模块.GetPaymentConfig(currencyCode, type="withdraw")
        ← 返回：该币种下可用提现方式列表（方式名/图标/费率/大额费率/限额/排序）
      → [内部] 步骤5：综合过滤
        → 用KYC状态过滤（步骤2结果）
        → 用充值历史过滤（步骤3结果）
        → 最终得到该用户实际可用的提现方式列表
      → [内部] 步骤6：如果是BSB，获取出款汇率
    → 返回：KYC状态 + 可用提现方式列表 + 限额/费率信息 + 汇率（BSB时）
```

**调用关系总结：**
- 跨模块调用：
  - **ser-kyc.KycQuery** — 查KYC状态，决定可用渠道
  - **财务模块.GetPaymentConfig** — 提现方式和费率来自财务配置
- 模块内调用：币种配置（出款汇率）+ 充值历史查询（确定提现规则）
- 关键点：这个接口的**过滤逻辑最复杂**——需要综合KYC状态+充值历史+财务配置三方面信息

---

### C10 GetWithdrawAccount（获取提现账户信息）

**触发场景：** 用户选择某种提现方式后，展示已保存的账户信息
**需求依据：** 钱包需求7.2-7.4，首次填写/后续复用

```
C端App
  → gate-font
    → ser-wallet.GetWithdrawAccount(userID, withdrawMethod)
      → [内部] 查询该用户该提现方式下已保存的账户信息
        → 银行卡：银行名称/银行账号/账户持有人/开户网点
        → 电子钱包：钱包类型/手机号/账户持有人
      → [内部] 如果有记录：标记为"已填写"，返回只读数据
      → [内部] 如果无记录：标记为"首次"，需引导用户填写
    → 返回：账户信息（有则返回/无则标记首次）
```

**调用关系总结：**
- 跨模块调用：无
- 模块内调用：纯读自己的提现账户表
- 特点：完全独立，不依赖外部模块

---

### C11 SaveWithdrawAccount（保存提现账户信息）

**触发场景：** 用户首次提现时填写账户信息并保存
**需求依据：** 钱包需求7.2-7.4，账户持有人=KYC姓名

```
C端App
  → gate-font
    → ser-wallet.SaveWithdrawAccount(userID, withdrawMethod, accountInfo)
      → [跨模块 RPC] 步骤1：ser-user.GetUserInfoExpend(userID)
        ← 返回：用户KYC认证姓名
      → [内部] 步骤2：强制校验 accountInfo.holderName == KYC姓名
        → 不一致：拒绝保存（账户持有人必须与KYC认证姓名一致）
        → 说明：前端展示时"账户持有人"字段置灰不可编辑，自动填充KYC姓名
      → [内部] 步骤3：保存账户信息到提现账户表
    → 返回：保存成功
```

**调用关系总结：**
- 跨模块调用：**ser-user.GetUserInfoExpend** — 获取KYC姓名用于校验和自动填充
- 模块内调用：写入提现账户表
- 关键点：账户持有人姓名是**后端强校验**，不能只依赖前端置灰

---

### C12 CreateWithdrawOrder（创建提现订单）

**触发场景：** 用户确认提现，点击"确认提现"
**需求依据：** 钱包需求7.1，原型10-5提现进行中

```
C端App
  → gate-font
    → ser-wallet.CreateWithdrawOrder(userID, currencyCode, amount, withdrawMethod, accountInfo)
      → [跨模块 RPC] 步骤1：ser-kyc.KycQuery(userID)
        ← 校验KYC状态，确认该用户有权使用此提现方式
      → [内部] 步骤2：校验提现限额
        → 单笔限额校验（金额 <= 单笔上限，金额 >= 单笔下限）
        → 单日限额校验（今日已提现金额 + 本次金额 <= 单日上限）
        → 不通过：返回对应提示（原型10-6"超出当日提现限额"弹窗）
      → [内部] 步骤3：如果是BSB提现
        → [内部] 币种配置：获取出款汇率（锁定汇率快照）
        → [内部] 计算实际出款金额
      → [内部] 步骤4：校验中心钱包余额 >= 提现金额
      → [内部] 步骤5：**FreezeBalance（冻结）**
        → 中心钱包可用余额 -= 提现金额
        → 中心钱包冻结余额 += 提现金额
        → 写入冻结流水记录
      → [内部] 步骤6：创建本地提现订单记录（待风控审核状态）
      → [跨模块 RPC] 步骤7：财务模块.CreateWithdrawOrder(订单信息)
        → 财务内部：创建提现审核订单
        ← 返回：财务订单号
      → [内部] 步骤8：更新本地订单，关联财务订单号
    → 返回：提现订单信息 + "提现进行中"状态

--- 异步后续链路（后台审核流程）---

财务模块B端
  → 风控审核（通过/驳回）
    → 驳回：财务模块 → [跨模块 RPC] ser-wallet.UnfreezeBalance(userID, currencyCode, amount, orderNo)
      → [内部] 中心钱包冻结余额 -= 提现金额
      → [内部] 中心钱包可用余额 += 提现金额（退回）
      → [内部] 写入解冻流水记录
    → 通过：进入财务审核
  → 财务审核（通过/驳回）
    → 驳回：同上，调用 UnfreezeBalance 解冻退回
    → 通过：进入出款
  → 出款（自动匹配通道/人工出款）
    → 出款失败：调用 UnfreezeBalance 解冻退回
    → 出款成功：财务模块 → [跨模块 RPC] ser-wallet.ConfirmDebit(userID, currencyCode, amount, orderNo)
      → [内部] 中心钱包冻结余额 -= 提现金额（正式扣除）
      → [内部] 写入确认扣除流水记录
      → [内部] 提现金额从用户钱包中永久减少
```

**调用关系总结：**
- 跨模块调用（同步）：
  - **ser-kyc.KycQuery** — KYC状态前置校验
  - **财务模块.CreateWithdrawOrder** — 提交提现审核
- 跨模块调用（异步后续）：
  - 财务模块 → ser-wallet.UnfreezeBalance — 驳回时解冻
  - 财务模块 → ser-wallet.ConfirmDebit — 出款成功时确认扣除
- 模块内调用：限额校验 → 币种配置（汇率）→ 余额校验 → FreezeBalance → 创建订单
- 关键点：这是**前置条件最多、异步链路最长**的接口，冻结-确认/解冻构成严格状态机

---

### C13 GetRecordList（交易记录列表）

**触发场景：** 用户进入记录页面，按Tab切换（提现/充值/兑换/奖励）
**需求依据：** 钱包需求8.1-8.3，原型11-1

```
C端App
  → gate-font
    → ser-wallet.GetRecordList(userID, currencyCode, recordType, page, pageSize)
      → [内部] 根据 recordType 分发查询：
        → recordType="deposit"（充值记录）：
          → [内部/待确认] 查询充值订单表（数据可能来自钱包自己的订单表或财务模块）
          → 字段：金额、方式、时间、状态（已完成/审核中/失败）
        → recordType="withdraw"（提现记录）：
          → [内部/待确认] 查询提现订单表（同上，数据归属待确认）
          → 字段：金额、方式、时间、状态
        → recordType="exchange"（兑换记录）：
          → [内部] 查询兑换流水表（确定是钱包自己的数据）
          → 字段：兑换金额、赠送比例、实际到账、时间
        → recordType="reward"（奖励记录）：
          → [内部] 查询奖励流水表（确定是钱包自己的数据）
          → 字段：奖励金额、奖励类别、时间
      → [内部] 按当前币种过滤，不支持跨币种混合
      → [内部] 按币种格式化规则格式化金额
    → 返回：记录列表（分页）
```

**调用关系总结：**
- 跨模块调用：**待确认** — 充值/提现记录数据归属需要明确
  - 方案A：钱包自己存一份订单数据（创建订单时写入），查自己的表
  - 方案B：充值/提现记录从财务模块读取，通过RPC查询
  - 建议：方案A更合理，减少跨模块依赖，钱包在创建订单时同步维护自己的记录
- 模块内调用：订单/流水表查询 + 币种配置（格式化规则）

---

### C14 GetRecordDetail（交易记录详情）

**触发场景：** 用户在记录列表中点击某条记录
**需求依据：** 钱包需求8.4，原型11-2

```
C端App
  → gate-font
    → ser-wallet.GetRecordDetail(userID, recordType, recordID)
      → [内部] 根据 recordType + recordID 查询详情
      → [内部] 组装详情字段：订单号、金额明细、状态、时间、方式
      → [内部] 如果状态为失败，附带失败原因
        → 充值失败原因：支付超时/支付异常/卡片问题
        → 提现失败原因：信息有误/系统维护/额度受限
    → 返回：记录详情
```

**调用关系总结：**
- 跨模块调用：无（详情数据应该在列表数据的基础上扩展，不需要额外跨模块调用）
- 模块内调用：读订单/流水详情表

---

## 二、B端接口调用链路（gate-back → ser-wallet → 内部/外部）

### B1 PageCurrency（币种列表）

**触发场景：** 后台管理员进入"系统管理 > 币种配置"页面
**需求依据：** 币种配置3.3，原型-币种配置1

```
B端后台
  → gate-back
    → ser-wallet.PageCurrency(筛选条件: currencyCode?, currencyType?, status?)
      → [内部] 币种配置：分页查询币种列表
      → [内部] 币种配置：获取每个币种的实时汇率、平台汇率、入款/出款汇率
      → [内部] 计算每个币种的偏差值 = |实时汇率 - 平台汇率| / 实时汇率 × 100%
      → [内部] 获取汇率刷新倒计时
    → 返回：币种列表（20个字段） + 刷新倒计时
```

**调用关系总结：**
- 跨模块调用：无
- 模块内调用：币种配置数据表查询 + 汇率计算

---

### B2 EditCurrency（编辑币种）

**触发场景：** 管理员点击某币种的"编辑"按钮
**需求依据：** 币种配置3.3编辑弹窗

```
B端后台
  → gate-back
    → ser-wallet.EditCurrency(currencyID, icon?, floatRate?, threshold?, status?)
      → [内部] 步骤1：校验币种存在
      → [内部] 步骤2：如果上传了图标
        → [跨模块 RPC] ser-s3.GetPresignUrl(文件类型, 文件大小)
          ← 返回：预签名上传URL
        → [内部] 校验图标格式（SVG/PNG）和大小（≤5k）
      → [内部] 步骤3：如果修改了汇率浮动%或阈值%
        → [内部] 重新计算入款汇率和出款汇率
        → 入款汇率 = 平台汇率 × (1 + 新浮动%)
        → 出款汇率 = 平台汇率 × (1 - 新浮动%)
      → [内部] 步骤4：更新币种配置数据
      → [跨模块 RPC] 步骤5：ser-blog.CreateOperationLog(操作人, "编辑", "币种配置", 币种编号+代码)
    → 返回：编辑成功
```

**调用关系总结：**
- 跨模块调用：
  - **ser-s3.GetPresignUrl** — 图标上传（可选，只在上传图标时调用）
  - **ser-blog.CreateOperationLog** — 审计日志（必须，工程规范要求）
- 模块内调用：币种配置数据更新 + 汇率重算

---

### B3 SetBaseCurrency（设置基准币种）

**触发场景：** 管理员首次设置基准币种（一次性操作）
**需求依据：** 币种配置3.2，原型-基准币种配置

```
B端后台
  → gate-back
    → ser-wallet.SetBaseCurrency(currencyCode)
      → [内部] 步骤1：校验是否已设置过基准币种
        → 已设置：直接拒绝（不可更改）
      → [内部] 步骤2：二级确认标记校验（前端需弹确认框，后端验证confirm标记）
      → [内部] 步骤3：设置基准币种（写入全局配置）
      → [内部] 步骤4：初始化该币种的平台汇率（首次直接取实时汇率值）
      → [跨模块 RPC] 步骤5：ser-blog.CreateOperationLog
    → 返回：设置成功
```

**调用关系总结：**
- 跨模块调用：**ser-blog** — 审计日志
- 模块内调用：全局配置写入
- 特殊约束：只能执行一次

---

### B4 StatusCurrency（启用/禁用币种）

**触发场景：** 管理员切换币种的启用/禁用状态
**需求依据：** 币种配置3.3

```
B端后台
  → gate-back
    → ser-wallet.StatusCurrency(currencyID, status)
      → [内部] 步骤1：如果是禁用操作
        → 校验是否为基准币种（基准币种不可禁用）
        → [内部] 查询是否有用户在该币种下有余额
          → 有余额：返回警告信息（前端需弹提示）
      → [内部] 步骤2：更新币种状态
      → [跨模块 RPC] 步骤3：ser-blog.CreateOperationLog
    → 返回：状态更新成功（含警告信息，如有）
```

**调用关系总结：**
- 跨模块调用：**ser-blog** — 审计日志
- 模块内调用：币种状态更新 + 用户余额检查
- 副作用：禁用后C端该币种下所有操作（充值/提现/兑换）将不可用

---

### B5 PageExchangeRateLog（汇率日志）

**触发场景：** 管理员进入"操作日志 > 汇率日志"Tab
**需求依据：** 币种配置4.4，原型-操作日志

```
B端后台
  → gate-back
    → ser-wallet.PageExchangeRateLog(筛选条件: timeRange?, currencyCode?)
      → [内部] 查询汇率变更日志表（每次偏差值>阈值触发更新时写入的记录）
    → 返回：日志列表（日志编号/更新时间/币种代码/实时汇率/平台汇率/入款汇率/出款汇率）
```

**调用关系总结：**
- 跨模块调用：无
- 模块内调用：纯读汇率日志表
- 特点：完全独立

---

### B6-B7 PageUserWallet / DetailUserWallet（用户钱包查询）

**触发场景：** 客服/风控查看某用户钱包状况，或人工修正前确认余额
**需求依据：** 财务人工修正场景联动需要

```
B端后台
  → gate-back
    → ser-wallet.PageUserWallet(筛选条件: userID?, currencyCode?, page, pageSize)
      → [内部] 查询用户钱包表，分页返回
      → [跨模块 RPC] ser-user.BatchGetUserInfo(userIDs)（可选，展示用户昵称）
    → 返回：用户钱包列表

    → ser-wallet.DetailUserWallet(userID, currencyCode)
      → [内部] 查询该用户该币种下所有钱包类型的余额明细
        → 中心钱包余额（可用 + 冻结）
        → 奖励钱包余额
        → 主播钱包余额（如有）
        → 代理钱包余额（如有，预留）
    → 返回：钱包余额明细
```

**调用关系总结：**
- 跨模块调用：**ser-user.BatchGetUserInfo**（可选，展示用户信息）
- 模块内调用：用户钱包表查询

---

## 三、RPC对外提供接口的内部链路（其他模块 → ser-wallet）

> 以下接口是 ser-wallet 暴露给外部模块的 RPC 方法
> 调用方通过 rpc.WalletClient().XxxMethod() 发起调用

### R1 CreditWallet（上账 — 充值成功入账）

**调用方：** 财务模块（充值回调处理后）
**需求依据：** 充值流程最后一步

```
财务模块
  → [RPC] ser-wallet.CreditWallet(userID, currencyCode, amount, orderNo, creditType)
    → [内部] 步骤1：幂等校验（orderNo是否已处理过，防止重复入账）
    → [内部] 步骤2：校验币种存在且启用
    → [内部] 步骤3：如果用户没有该币种钱包 → 自动创建
    → [内部] 步骤4：中心钱包余额 += amount
    → [内部] 步骤5：写入钱包流水（类型：充值入账，关联订单号）
  → 返回：上账结果（成功/失败+原因）
```

**内部核心逻辑：** 幂等校验 → 余额变更 → 流水记录（单表事务）

---

### R2 DebitWallet（扣款 — 消费/投注扣款）

**调用方：** 财务模块 / 投注模块
**需求依据：** 钱包需求4.3余额规则（优先扣中心，再扣奖励）

```
调用方
  → [RPC] ser-wallet.DebitWallet(userID, currencyCode, amount, orderNo, debitType)
    → [内部] 步骤1：幂等校验（orderNo）
    → [内部] 步骤2：查询中心钱包余额 + 奖励钱包余额
    → [内部] 步骤3：校验总余额 >= amount
      → 不足：返回余额不足错误
    → [内部] 步骤4：计算扣款分配（优先扣中心，不足部分扣奖励）
      → centerDebit = min(centerBalance, amount)
      → rewardDebit = amount - centerDebit
    → [内部] 步骤5：事务内执行
      → 中心钱包余额 -= centerDebit
      → 奖励钱包余额 -= rewardDebit（如有）
      → 写入钱包流水（记录中心/奖励各扣了多少，用于后续混合返奖计算）
  → 返回：扣款结果（含各钱包实际扣款明细）
```

**内部核心逻辑：** 幂等 → 余额校验 → 分配计算 → 双钱包扣减（事务）

---

### R3 FreezeBalance（冻结 — 提现申请时）

**调用方：** 自己内部（CreateWithdrawOrder流程中调用）/ 财务模块
**需求依据：** 提现流程：申请→冻结→审核→出款

```
调用方
  → [RPC] ser-wallet.FreezeBalance(userID, currencyCode, amount, orderNo)
    → [内部] 步骤1：幂等校验
    → [内部] 步骤2：校验中心钱包可用余额 >= amount
    → [内部] 步骤3：事务内执行
      → 中心钱包可用余额 -= amount
      → 中心钱包冻结余额 += amount
      → 写入冻结流水（状态：冻结中）
  → 返回：冻结结果 + 冻结流水号
```

**状态机前置：** 此接口执行后，后续必须走 ConfirmDebit 或 UnfreezeBalance 其一

---

### R4 UnfreezeBalance（解冻 — 提现驳回/取消时）

**调用方：** 财务模块（风控/财务审核驳回、出款失败时）
**需求依据：** 提现驳回→退回余额

```
财务模块
  → [RPC] ser-wallet.UnfreezeBalance(userID, currencyCode, amount, orderNo)
    → [内部] 步骤1：校验该orderNo对应的冻结记录存在且状态为"冻结中"
    → [内部] 步骤2：事务内执行
      → 中心钱包冻结余额 -= amount
      → 中心钱包可用余额 += amount（退回）
      → 更新冻结流水状态：冻结中 → 已解冻
  → 返回：解冻结果
```

**状态机约束：** 必须先有 FreezeBalance，且状态为"冻结中"才能执行

---

### R5 ConfirmDebit（确认扣除 — 提现出款成功时）

**调用方：** 财务模块（出款成功后）
**需求依据：** 提现成功→正式扣减冻结金额

```
财务模块
  → [RPC] ser-wallet.ConfirmDebit(userID, currencyCode, amount, orderNo)
    → [内部] 步骤1：校验该orderNo对应的冻结记录存在且状态为"冻结中"
    → [内部] 步骤2：事务内执行
      → 中心钱包冻结余额 -= amount（正式扣除，资金永久离开用户钱包）
      → 更新冻结流水状态：冻结中 → 已确认扣除
      → 写入扣款流水（类型：提现扣款）
  → 返回：确认结果
```

**与R4互斥：** ConfirmDebit 和 UnfreezeBalance 针对同一笔冻结只能执行其一

---

### R6 ManualCredit（人工加款）

**调用方：** 财务模块（人工加款审核通过后）
**需求依据：** 财务需求2.52

```
财务模块
  → [RPC] ser-wallet.ManualCredit(userID, currencyCode, walletType, amount, orderNo, creditType)
    → [内部] 步骤1：幂等校验
    → [内部] 步骤2：如果用户没有该币种+该类型的钱包 → 自动创建
    → [内部] 步骤3：指定钱包类型余额 += amount
      → walletType=center → 中心钱包
      → walletType=reward → 奖励钱包
      → walletType=anchor → 主播钱包
      → walletType=agent → 代理钱包
    → [内部] 步骤4：写入流水（A+订单号前缀，记录加款类型）
  → 返回：加款结果
```

**特殊逻辑：** 可以同时向多个钱包类型加款（财务模块一次操作可能调用多次本接口）

---

### R7 ManualDebit（人工减款）

**调用方：** 财务模块（人工减款审核通过后）
**需求依据：** 财务需求2.53

```
财务模块
  → [RPC] ser-wallet.ManualDebit(userID, currencyCode, amount, orderNo, debitType)
    → [内部] 步骤1：幂等校验
    → [内部] 步骤2：查询该用户该币种下所有钱包类型的余额
    → [内部] 步骤3：按余额从高到低排序，级联扣减
      → 例如：中心余额800，奖励余额200，主播余额100，需减1000
      → 扣中心800 → 扣奖励200 → 刚好1000，完成
      → 如果总余额不足：实际减款金额 = 总余额（可能小于设定金额）
    → [内部] 步骤4：事务内执行各钱包扣减
    → [内部] 步骤5：写入流水（M+订单号前缀，记录各钱包实际扣减明细）
  → 返回：减款结果（含实际减款金额、各钱包扣减明细）
```

**特殊逻辑：** 跨钱包类型级联扣减 + 实际减款可能小于设定金额

---

### R8 SupplementCredit（充值补单）

**调用方：** 财务模块（充值补单审核通过后）
**需求依据：** 财务需求2.51

```
财务模块
  → [RPC] ser-wallet.SupplementCredit(userID, currencyCode, supplementAmount, bonusAmount, auditMultiple, orderNo)
    → [内部] 步骤1：幂等校验
    → [内部] 步骤2：事务内执行
      → 中心钱包余额 += supplementAmount（补单金额）
      → 奖励钱包余额 += bonusAmount（补赠送金额，需完成auditMultiple倍流水）
    → [内部] 步骤3：写入流水（B+订单号前缀）
  → 返回：补单结果
```

---

### R9 GetBalance（余额查询）

**调用方：** 财务/投注/直播/活动等多个模块
**需求依据：** 通用能力

```
任意模块
  → [RPC] ser-wallet.GetBalance(userID, currencyCode, walletType?)
    → [内部] 查询指定用户指定币种的钱包余额
      → 不指定walletType：返回所有钱包类型的余额明细
      → 指定walletType：返回特定钱包的余额
    → [内部] 返回：可用余额、冻结余额（中心钱包特有）
  → 返回：余额信息
```

**特点：** 纯读操作，完全独立，最高频的RPC接口

---

### R10 RewardCredit（活动返奖入账）

**调用方：** 活动/任务模块
**需求依据：** 钱包需求4.2资金流向（活动/任务返奖→奖励钱包）

```
活动模块
  → [RPC] ser-wallet.RewardCredit(userID, currencyCode, amount, orderNo, activityInfo)
    → [内部] 步骤1：幂等校验
    → [内部] 步骤2：如果用户没有该币种奖励钱包 → 自动创建
    → [内部] 步骤3：奖励钱包余额 += amount
    → [内部] 步骤4：写入流水（记录活动信息、稽核流水倍数要求）
  → 返回：入账结果
```

---

## 四、内部定时任务链路

### 汇率定时更新任务

**触发方式：** 定时任务（刷新频率可配置）
**需求依据：** 币种配置4.2汇率规则

```
定时触发（ser-cron 或内置 cron）
  → ser-wallet.RefreshExchangeRate()
    → [内部] 步骤1：查询所有启用的非基准币种列表
    → [外部HTTP] 步骤2：对每个币种，并行调用3家三方汇率API
      → API-1：获取汇率
      → API-2：获取汇率
      → API-3：获取汇率
    → [内部] 步骤3：计算实时汇率 = 三家平均值
    → [内部] 步骤4：对每个币种判断是否需要更新平台汇率
      → 计算偏差值 = |实时汇率 - 当前平台汇率| / 实时汇率 × 100%
      → 偏差值 >= 阈值%：
        → 更新平台汇率 = 实时汇率
        → 更新入款汇率 = 实时汇率 × (1 + 浮动%)
        → 更新出款汇率 = 实时汇率 × (1 - 浮动%)
        → 写入汇率变更日志（供B5 PageExchangeRateLog查询）
      → 偏差值 < 阈值%：
        → 保持平台汇率不变
        → 仅更新实时汇率显示值
    → [内部] 步骤5：更新缓存（Redis），供C端接口实时读取
```

**调用关系总结：**
- 外部调用：3家三方汇率API（HTTP请求，非RPC）
- 模块内调用：币种配置表（读配置 + 写汇率更新 + 写变更日志）
- 关键点：这是**唯一涉及外部HTTP调用**的逻辑，需要处理API超时、部分失败等异常

---

## 五、跨模块调用链路汇总（钱包视角）

### 5.1 钱包调用外部（ser-wallet 作为 RPC Client）

```
ser-wallet → 财务模块（服务名待定）
  ├── GetPaymentConfig(currencyCode, type)    场景：C3充值方式 / C9提现方式
  │     获取充值方式列表+档位 或 提现方式列表+费率+限额
  │
  ├── CreatePayOrder(orderInfo)               场景：C4创建充值订单
  │     传入订单信息，财务负责匹配通道+创建代收订单
  │
  └── CreateWithdrawOrder(orderInfo)           场景：C12创建提现订单
        传入提现订单信息，财务创建审核订单

ser-wallet → ser-kyc
  └── KycQuery(userID)                        场景：C9提现方式 / C12创建提现订单
        查询KYC认证状态，决定可用提现渠道

ser-wallet → ser-user
  ├── GetUserInfoExpend(userID)               场景：C11保存提现账户
  │     获取KYC认证姓名，自动填充"账户持有人"
  │
  └── BatchGetUserInfo(userIDs)               场景：B6用户钱包列表（可选）
        B端展示用户昵称等基础信息

ser-wallet → ser-blog
  └── CreateOperationLog(操作信息)             场景：B2/B3/B4所有B端写操作
        审计日志，工程规范要求

ser-wallet → ser-s3
  └── GetPresignUrl(fileType, fileSize)        场景：B2编辑币种上传图标
        获取预签名URL用于上传图标文件

ser-wallet → 活动模块（服务名待定）
  ├── GetAvailableBonuses(userID, currencyCode) 场景：C6获取可用奖金活动
  │     获取充值可选的额外奖金活动列表
  │
  └── ValidateBonus(activityID, amount)        场景：C4创建充值订单（选了奖金时）
        校验奖金活动有效性，返回奖金金额和流水倍数

ser-wallet → 三方汇率API（HTTP，非RPC）
  └── GetExchangeRate(baseCurrency, targetCurrency)  场景：汇率定时任务
        3家API并行调用取平均值
```

### 5.2 外部调用钱包（ser-wallet 作为 RPC Server）

```
财务模块 → ser-wallet
  ├── CreditWallet(userID, currencyCode, amount, orderNo, type)
  │     场景：充值成功回调后上账
  │
  ├── ConfirmDebit(userID, currencyCode, amount, orderNo)
  │     场景：提现出款成功后确认扣除冻结
  │
  ├── UnfreezeBalance(userID, currencyCode, amount, orderNo)
  │     场景：提现驳回/出款失败后解冻退回
  │
  ├── SupplementCredit(userID, currencyCode, amounts, orderNo)
  │     场景：充值补单审核通过后
  │
  ├── ManualCredit(userID, currencyCode, walletType, amount, orderNo, type)
  │     场景：人工加款审核通过后
  │
  └── ManualDebit(userID, currencyCode, amount, orderNo, type)
        场景：人工减款审核通过后

投注模块 → ser-wallet
  ├── DebitWallet / BetDebit(userID, currencyCode, amount, orderNo)
  │     场景：投注下单时扣款（需记录中心/奖励扣款比例）
  │
  ├── CreditWallet / BetReturn(userID, currencyCode, amount, orderNo)
  │     场景：投注返奖时入账（按投注时的扣款比例分配到中心/奖励）
  │
  └── GetBalance(userID, currencyCode)
        场景：下注前校验余额

直播/社交模块 → ser-wallet
  ├── AnchorIncome(userID, currencyCode, amount, orderNo)
  │     场景：主播收入入账到主播钱包
  │
  └── GetBalance(userID, currencyCode)
        场景：送礼前校验余额

活动/任务模块 → ser-wallet
  └── RewardCredit(userID, currencyCode, amount, orderNo, activityInfo)
        场景：活动/任务返奖入账到奖励钱包
```

---

## 六、接口间内部调用关系（模块内复用）

> 以下梳理 ser-wallet 内部接口之间的复用/依赖关系
> 即一个接口的实现中调用了另一个接口的内部逻辑

```
GetWalletHome（C1）
  └── 复用 → GetBalance 内部逻辑（查余额）
  └── 复用 → GetCurrencyList 内部逻辑的子集（查单个币种信息）

GetCurrencyList（C2）
  └── 复用 → GetBalance 内部逻辑（批量查余额）

GetDepositMethods（C3）
  └── 复用 → GetExchangeRate 内部逻辑的子集（BSB时获取入款汇率）

CreateDepositOrder（C4）
  └── 调用 → CheckDuplicateDeposit 内部逻辑（C5）
  └── 复用 → GetExchangeRate 内部逻辑的子集（BSB时锁定汇率）

GetWithdrawMethods（C9）
  └── 依赖 → 充值订单数据（判断充值历史决定提现规则）

CreateWithdrawOrder（C12）
  └── 调用 → FreezeBalance 内部逻辑（R3）
  └── 复用 → GetExchangeRate 内部逻辑的子集（BSB提现时锁定出款汇率）

DoExchange（C8）
  └── 复用 → GetExchangeRate 内部逻辑（C7，获取平台汇率）
  └── 调用 → DebitWallet 类似逻辑（扣法币中心钱包）
  └── 调用 → CreditWallet 类似逻辑（加BSB中心钱包 + 加BSB奖励钱包）

ManualDebit（R7）
  └── 复用 → GetBalance 内部逻辑（查所有钱包类型余额，确定扣减顺序）
  └── 调用 → DebitWallet 类似逻辑（但是跨钱包类型级联扣减，逻辑更复杂）
```

**可抽取的公共内部方法（internal/ser 层）：**

```
公共方法                           被哪些接口复用
─────────────────────────────── ────────────────────────────────
getWalletBalance(userID, curr)   C1, C2, C8, C12, R2, R7, R9
getCurrencyInfo(currCode)        C1, C2, C3, C7, C8, C9, C12, B1
getExchangeRate(currCode, type)  C3, C4, C7, C8, C9, C12
checkCurrencyEnabled(currCode)   C1, C3, C4, C7, C8, C9, C12, R1~R8
creditWalletCore(参数)            R1, R6, R8, R10, R12, R13（上账核心逻辑）
debitWalletCore(参数)             R2, R7, R11（扣款核心逻辑）
freezeCore(参数)                  R3（冻结核心）
unfreezeCore(参数)                R4（解冻核心）
confirmDebitCore(参数)            R5（确认扣除核心）
idempotentCheck(orderNo)         R1~R8, R10~R13（所有写操作的幂等校验）
writeWalletFlow(流水信息)         所有余额变更操作
```

---

## 七、关键认知总结

### 7.1 我的模块（钱包+币种配置）内部调用关系

```
                    ┌─────────────────────────┐
                    │      币种配置层          │
                    │  （基础数据供给，无外依赖）│
                    │                         │
                    │  币种列表 / 汇率数据 /    │
                    │  展示规则 / 币种状态      │
                    └───────────┬─────────────┘
                                │ 被所有钱包接口依赖
                    ┌───────────▼─────────────┐
                    │     钱包公共方法层        │
                    │  （internal/ser 内部）    │
                    │                         │
                    │  余额查询 / 上账 / 扣款 / │
                    │  冻结 / 解冻 / 幂等校验 / │
                    │  流水记录                 │
                    └──┬──────────┬────────────┘
          ┌────────────┤          ├────────────────┐
          ▼            ▼          ▼                ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐
    │ C端接口   │ │ B端接口   │ │ RPC对外   │  │ 定时任务  │
    │ 13-18个   │ │ 7-9个    │ │ 10-17个   │  │ 汇率刷新  │
    │ 组合调用   │ │ 组合调用  │ │ 直接调用   │  │          │
    │ 公共方法   │ │ 公共方法  │ │ 公共方法   │  │          │
    └──────────┘ └──────────┘ └──────────┘  └──────────┘
```

### 7.2 我的模块与外部模块的调用边界

```
我调别人（我是Client，同步调用，在C端用户操作链路上）：
  → 财务模块：3个接口（获取充值方式、获取提现方式、创建通道订单）
  → ser-kyc：1个接口（KYC状态查询）
  → ser-user：1-2个接口（用户信息/KYC姓名）
  → ser-blog：1个接口（审计日志）
  → ser-s3：1个接口（图标上传）
  → 活动模块：1-2个接口（奖金活动）

别人调我（我是Server，异步调用，在回调/审核触发时）：
  ← 财务模块：6个接口（上账/确认扣除/解冻/补单/加款/减款）
  ← 投注模块：2-3个接口（投注扣款/返奖/余额查询）
  ← 直播模块：1-2个接口（主播收入/余额查询）
  ← 活动模块：1个接口（活动返奖）
```

### 7.3 需要和财务模块负责人提前对齐的接口契约

```
第一优先级（充值流程必需）：
  ① 财务模块.GetPaymentConfig    — 我调用，获取充值/提现方式配置
  ② 财务模块.CreatePayOrder      — 我调用，创建充值通道订单
  ③ ser-wallet.CreditWallet       — 财务调我，充值成功上账

第二优先级（提现流程必需）：
  ④ 财务模块.CreateWithdrawOrder — 我调用，创建提现审核订单
  ⑤ ser-wallet.ConfirmDebit       — 财务调我，出款成功确认扣除
  ⑥ ser-wallet.UnfreezeBalance    — 财务调我，驳回解冻退回

第三优先级（人工修正必需）：
  ⑦ ser-wallet.ManualCredit       — 财务调我，人工加款
  ⑧ ser-wallet.ManualDebit        — 财务调我，人工减款
  ⑨ ser-wallet.SupplementCredit   — 财务调我，充值补单
```

### 7.4 接口调用频率与性质分类

| 分类 | 接口 | 频率 | 关注点 |
|------|------|------|--------|
| 高频读 | GetWalletHome, GetCurrencyList, GetBalance, GetExchangeRate | 每次打开页面 | 缓存策略 |
| 中频读 | GetDepositMethods, GetWithdrawMethods, GetRecordList | 进入充值/提现/记录页 | 跨模块RPC超时处理 |
| 低频写（用户触发） | CreateDepositOrder, DoExchange, CreateWithdrawOrder | 用户主动操作 | 事务一致性、幂等 |
| 低频写（系统触发） | CreditWallet, ConfirmDebit, UnfreezeBalance | 回调/审核后 | 幂等性、数据最终一致 |
| 极低频写（B端触发） | ManualCredit, ManualDebit, SupplementCredit | 人工操作 | 双人审核、审计 |
| 定时 | RefreshExchangeRate | 配置周期（如1分钟） | 三方API容错、缓存更新 |
