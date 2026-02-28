# ser-wallet 模块间依赖关系与链路分析

> 分析时间：2026-02-26
> 分析依据：
> - 需求理解：/Users/mac/gitlab/z-readme/需求理解/tmp-1.md
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-1.md
> - 原始需求提取：/Users/mac/gitlab/z-readme/result/tmp-1.md（~130张需求图片）
> - 需求总结：/Users/mac/gitlab/z-readme/result/c-3.md
> 分析目的：梳理清楚模块间的依赖调用关系、接口前后置关系、耦合度，为架构设计和开发排期提供决策依据
> 职责边界：我负责币种配置 + 多币种钱包，财务管理为他人负责（需了解依赖）

---

## 目录

1. [模块层级依赖总览](#一模块层级依赖总览)
2. [核心业务链路拆解](#二核心业务链路拆解)
3. [接口前后置依赖矩阵](#三接口前后置依赖矩阵)
4. [耦合度分析](#四耦合度分析)
5. [数据流向与调用方向图](#五数据流向与调用方向图)
6. [实现顺序建议](#六实现顺序建议)
7. [风险点与待确认项](#七风险点与待确认项)

---

## 一、模块层级依赖总览

### 1.1 三层架构关系

三个模块天然形成**基础设施层 → 核心业务层 → 运营管理层**的分层结构，依赖方向**自上而下单向**为主，但存在反向回调：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        运营管理层（B端）                              │
│                                                                     │
│                     ┌─────────────────┐                             │
│                     │   ser-finance   │                             │
│                     │   (财务管理)     │                             │
│                     └────────┬────────┘                             │
│                              │                                      │
│              ┌───────────────┼───────────────────┐                  │
│              │ WalletCredit  │ WalletFreeze       │ GetUserBalance  │
│              │ WalletCreditR │ WalletUnfreeze     │ GetCurrencyConf │
│              │ WalletManual+ │ WalletDeduct       │                 │
│              ▼               ▼                    ▼                 │
├─────────────────────────────────────────────────────────────────────┤
│                        核心业务层（C端+RPC）                          │
│                                                                     │
│                     ┌─────────────────┐                             │
│                     │   ser-wallet    │                             │
│                     │ (多币种钱包模块) │                             │
│                     └────────┬────────┘                             │
│                              │                                      │
│              ┌───────────────┼──────────────┐                       │
│              │ GetCurrencyC  │ CalcExchange  │ ListActiveCurr      │
│              │ GetExchangeR  │ GetPlatformR  │                      │
│              ▼               ▼               ▼                      │
├─────────────────────────────────────────────────────────────────────┤
│                        基础设施层（B端+RPC）                          │
│                                                                     │
│                     ┌─────────────────┐                             │
│                     │   ser-wallet    │                             │
│                     │  (币种配置模块)  │                             │
│                     └─────────────────┘                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 依赖方向汇总表

| 调用方 | 被调用方 | 调用内容 | 方向性质 |
|--------|---------|---------|---------|
| 钱包模块 | 币种配置模块 | 汇率查询、币种配置、换算计算 | **单向下行**（业务层→基础层）|
| 财务模块 | 钱包模块 | 入账/冻结/解冻/扣款/加减款/余额查询 | **单向下行**（管理层→业务层）|
| 财务模块 | 币种配置模块 | 币种配置/精度/汇率（通过钱包透传或直接调） | **单向下行**（管理层→基础层）|
| 钱包模块 | ser-kyc | KYC认证状态查询 | **外部单向** |
| 钱包模块 | ser-blog | 审计日志上报 | **外部单向** |
| 钱包模块 | ser-s3 | 图标上传 | **外部单向** |
| 财务模块 | 三方支付 | 代收/代付 + 回调接收 | **双向**（请求+回调）|

### 1.3 一个关键认知

> **币种配置是地基，不依赖任何业务模块；钱包模块是中间层，对下依赖币种配置，对上被财务调用；财务模块是最上层消费者。**
>
> 但存在一个**反向依赖**需要注意：钱包C端充值时，需要知道"当前币种有哪些可用的充值方式"，这个配置由**财务模块的支付配置**管理。所以钱包→财务存在一条**配置读取**的反向依赖。

---

## 二、核心业务链路拆解

共6条核心业务链路，逐条拆解每一步的接口调用和模块交互。

### 2.1 充值链路

```
用户(C端)         ser-wallet(钱包)        ser-wallet(币种)      ser-finance(财务)       三方支付
   │                   │                      │                     │                    │
   │ ①进入充值页面      │                      │                     │                    │
   ├──────────────────>│                      │                     │                    │
   │                   │ ②获取可用币种         │                     │                    │
   │                   ├─────────────────────>│                     │                    │
   │                   │  ListActiveCurrencies │                     │                    │
   │                   │<─────────────────────│                     │                    │
   │                   │                      │                     │                    │
   │                   │ ③获取充值配置（充值方式+限额+汇率）            │                    │
   │                   │ GetDepositConfig ────┼─ GetExchangeRate ──>│                    │
   │                   │                      │                     │                    │
   │                   │ (?) 获取可用充值方式 ─────────────────────>│                    │
   │                   │     ↑ 待确认：是wallet调finance读支付配置    │                    │
   │                   │       还是wallet本地缓存finance推送的配置    │                    │
   │                   │                      │                     │                    │
   │ ④选择奖金活动      │                      │                     │                    │
   │ (可选)             │                      │                     │                    │
   │ ListAvailableBonus│                      │                     │                    │
   │                   │                      │                     │                    │
   │ ⑤输入金额+发起充值 │                      │                     │                    │
   ├──────────────────>│                      │                     │                    │
   │                   │ ⑥重复转账拦截检查      │                     │                    │
   │                   │ (同UserID+同币种+同金额，查pending/processing订单数)              │
   │                   │ 1-2笔→软拦截(可继续)   │                     │                    │
   │                   │ ≥3笔→硬拦截(禁止)      │                     │                    │
   │                   │                      │                     │                    │
   │                   │ ⑦创建充值订单          │                     │                    │
   │                   │ CreateDepositOrder    │                     │                    │
   │                   │                      │                     │                    │
   │                   │ ──── 分支A: USDT充值（平台承载页）────       │                    │
   │                   │ ⑧生成收款地址          │                     │                    │
   │                   │ GetDepositAddress     │                     │                    │
   │                   │ (TRC20/ERC20/BEP20)   │                     │                    │
   │                   │                      │                     │     区块链监听服务   │
   │                   │<─────────────────────┼─────────────────────┼──── 链上确认回调 ───│
   │                   │ DepositNotify(到账)   │                     │                    │
   │                   │ 自行入账WalletCredit   │                     │                    │
   │                   │                      │                     │                    │
   │                   │ ──── 分支B: 银行/电子钱包充值（三方托管页）──  │                    │
   │                   │ ⑧请求创建支付 ─────────┼───────────────────>│                    │
   │                   │                      │                     │ ⑨向三方发起代收 ──>│
   │                   │                      │                     │<── ⑩三方回调 ──────│
   │                   │                      │                     │ CollectNotify      │
   │                   │                      │                     │                    │
   │                   │<── ⑪finance调wallet入账 ──────────────────│                    │
   │                   │    WalletCredit(金额→中心钱包)              │                    │
   │                   │    WalletCreditReward(奖金→奖励钱包)        │                    │
   │                   │                      │                     │                    │
   │<── ⑫返回结果 ─────│                      │                     │                    │
```

**链路涉及的接口前后关系：**

| 步骤 | 接口 | 前置依赖 | 性质 |
|------|------|---------|------|
| ② | `ListActiveCurrencies` | 币种配置数据已存在 | **前置**（无依赖，最底层） |
| ③ | `GetDepositConfig` | 依赖 `ListActiveCurrencies` + `GetExchangeRate` + 财务支付配置 | **前置**（聚合多个底层数据） |
| ③ | `GetExchangeRate` | 依赖币种配置+汇率定时任务已运行 | **前置** |
| ④ | `ListAvailableBonus` | 独立于充值流程，可并行获取 | **独立** |
| ⑥ | 重复转账拦截 | 依赖充值订单表存在 | `CreateDepositOrder` 的**内部前置逻辑** |
| ⑦ | `CreateDepositOrder` | 依赖 ②③④ 全部完成 | **核心动作** |
| ⑧ | `GetDepositAddress` | 仅USDT场景，依赖 ⑦ 订单已创建 | **后置**（USDT分支） |
| ⑪ | `WalletCredit` | 依赖三方回调成功 | **后置回调** |
| ⑪ | `WalletCreditReward` | 依赖 ⑪ 入账成功 + 用户选了奖金活动 | **后置回调** |

---

### 2.2 提现链路

```
用户(C端)         ser-wallet(钱包)        ser-kyc           ser-finance(财务)       三方支付
   │                   │                    │                    │                    │
   │ ①进入提现页面      │                    │                    │                    │
   ├──────────────────>│                    │                    │                    │
   │                   │ ②查KYC状态          │                    │                    │
   │                   ├───────────────────>│                    │                    │
   │                   │  GetUserKycStatus   │                    │                    │
   │                   │<───────────────────│                    │                    │
   │                   │                    │                    │                    │
   │                   │ ③根据KYC+充值历史 → 决定可用提现方式       │                    │
   │                   │ GetWithdrawConfig  │                    │                    │
   │                   │ 未认证→仅USDT       │                    │                    │
   │                   │ 已认证→全渠道       │                    │                    │
   │                   │ 提现币种规则：       │                    │                    │
   │                   │  法币充→法币提       │                    │                    │
   │                   │  USDT充→USDT提      │                    │                    │
   │                   │  混合→法币提         │                    │                    │
   │                   │                    │                    │                    │
   │ ④查询/填写提现账户  │                    │                    │                    │
   │ GetWithdrawAccount│                    │                    │                    │
   │ SaveWithdrawAccount│(首次，持有人=KYC实名)│                    │                    │
   │                   │                    │                    │                    │
   │ ⑤输入金额+发起提现 │                    │                    │                    │
   ├──────────────────>│                    │                    │                    │
   │                   │ ⑥校验限额           │                    │                    │
   │                   │ (单笔min/max、单日限额、余额充足性)        │                    │
   │                   │                    │                    │                    │
   │                   │ ⑦创建提现订单+冻结   │                    │                    │
   │                   │ CreateWithdrawOrder │                    │                    │
   │                   │ → 内部调 WalletFreeze(冻结对应金额)       │                    │
   │                   │                    │                    │                    │
   │                   │ ⑧通知财务审核 ──────┼──────────────────>│                    │
   │                   │ (RPC调用或事件通知，待确认)                │                    │
   │                   │                    │                    │                    │
   │                   │                    │    ⑨风控审核        │                    │
   │                   │                    │   ReviewWithdrawRisk│                    │
   │                   │                    │      │              │                    │
   │                   │                    │      ├─ 驳回 ──────>│                    │
   │                   │<── WalletUnfreeze(解冻) ─────────────────│                    │
   │                   │                    │      │              │                    │
   │                   │                    │      ├─ 通过 ──────>│                    │
   │                   │                    │                    │                    │
   │                   │                    │    ⑩财务审核        │                    │
   │                   │                    │  ReviewWithdrawFin  │                    │
   │                   │                    │      │              │                    │
   │                   │                    │      ├─ 驳回 ──────>│                    │
   │                   │<── WalletUnfreeze(解冻) ─────────────────│                    │
   │                   │                    │      │              │                    │
   │                   │                    │      ├─ 通过 ──────>│                    │
   │                   │                    │                    │ ⑪发起代付 ────────>│
   │                   │                    │                    │<── ⑫回调 ──────────│
   │                   │                    │                    │ PayoutNotify       │
   │                   │                    │                    │    │               │
   │                   │                    │                    │    ├─ 成功          │
   │                   │<── WalletDeduct(最终扣款) ───────────────│                    │
   │                   │                    │                    │    │               │
   │                   │                    │                    │    ├─ 失败          │
   │                   │<── WalletUnfreeze(解冻归还) ─────────────│                    │
   │                   │                    │                    │                    │
   │<── ⑬状态更新 ─────│                    │                    │                    │
```

**链路涉及的接口前后关系：**

| 步骤 | 接口 | 前置依赖 | 性质 |
|------|------|---------|------|
| ② | `GetUserKycStatus`(ser-kyc) | 无 | **外部前置** |
| ③ | `GetWithdrawConfig` | 依赖 KYC状态 + 财务支付配置 | **前置** |
| ④ | `GetWithdrawAccount` | 无强依赖 | **独立查询** |
| ④ | `SaveWithdrawAccount` | 依赖 KYC实名信息（比对姓名） | **首次前置** |
| ⑦ | `CreateWithdrawOrder` | 依赖 ③④⑥ 全部完成 | **核心动作** |
| ⑦ | `WalletFreeze` | `CreateWithdrawOrder` 内部调用 | **核心动作的原子步骤** |
| ⑨⑩ | `WalletUnfreeze` | 依赖审核驳回事件 | **后置（失败分支）** |
| ⑫ | `WalletDeduct` | 依赖代付成功回调 | **后置（成功分支）** |
| ⑫ | `WalletUnfreeze` | 依赖代付失败回调 | **后置（失败分支）** |

---

### 2.3 兑换链路

```
用户(C端)         ser-wallet(钱包)        ser-wallet(币种)
   │                   │                      │
   │ ①进入兑换页面      │                      │
   │ (仅法币钱包下可见)  │                      │
   ├──────────────────>│                      │
   │                   │ ②获取汇率             │
   │                   ├─────────────────────>│
   │                   │  GetExchangeRate      │
   │                   │  (口径：1 BSB = X 法币)│
   │                   │<─────────────────────│
   │                   │                      │
   │ ③输入法币金额       │                      │
   ├──────────────────>│                      │
   │                   │ ④试算预览             │
   │                   │ PreviewExchange       │
   │                   │ → CalcExchangeAmount  │
   │                   │   (兑换BSB + 赠送BSB + 实际到账BSB)
   │                   │<─────────────────────│
   │<── 展示结果 ───────│                      │
   │                   │                      │
   │ ⑤确认兑换          │                      │
   ├──────────────────>│                      │
   │                   │ ⑥执行兑换             │
   │                   │ CreateExchangeOrder   │
   │                   │ → CalcExchangeAmount  │
   │                   │ → 法币子钱包扣减       │
   │                   │ → BSB子钱包增加        │
   │                   │ → 赠送部分标记流水要求  │
   │                   │                      │
   │<── ⑦兑换成功(3s内) │                      │
```

**链路特点：最简单、最独立的链路**

| 步骤 | 接口 | 前置依赖 | 性质 |
|------|------|---------|------|
| ② | `GetExchangeRate` | 币种配置+汇率数据 | **前置** |
| ④ | `PreviewExchange` → `CalcExchangeAmount` | 依赖 ② 汇率 | **前置**（试算） |
| ⑥ | `CreateExchangeOrder` → `CalcExchangeAmount` | 依赖 ④ 确认 | **核心动作** |

> **兑换链路不涉及财务模块，不涉及三方支付，不涉及审核流程**。是钱包模块内部闭环操作，仅依赖币种配置的汇率数据。

---

### 2.4 人工修正链路（补单 / 加款 / 减款）

```
B端运营           ser-finance(财务)          ser-wallet(钱包)
   │                   │                         │
   │ ──────────── 补单流程 ────────────           │
   │ ①发起补单         │                         │
   ├──────────────────>│                         │
   │                   │ CreateSupplementOrder    │
   │                   │ (填写：用户ID、币种、     │
   │                   │  充值金额、额外奖金)      │
   │                   │                         │
   │ ②审核通过         │                         │
   ├──────────────────>│                         │
   │                   │ ReviewCorrectionOrder    │
   │                   │ ─── WalletCredit ──────>│ 金额→中心钱包
   │                   │ ─── WalletCreditReward ─>│ 奖金→奖励钱包(如有)
   │                   │                         │
   │ ──────────── 加款流程 ────────────           │
   │ ①发起加款         │                         │
   ├──────────────────>│                         │
   │                   │ ②查用户余额 ───────────>│
   │                   │   GetUserBalance         │ 返回各子钱包余额
   │                   │<─────────────────────────│
   │                   │ ③展示余额弹窗            │
   │                   │ CreateManualAddOrder     │
   │                   │ (分别填写4种子钱包加款额)  │
   │                   │                         │
   │ ④审核通过         │                         │
   ├──────────────────>│                         │
   │                   │ ReviewCorrectionOrder    │
   │                   │ ─── WalletManualAdd ───>│ 按子钱包分别加款
   │                   │                         │
   │ ──────────── 减款流程 ────────────           │
   │ (同加款，区别在于)  │                         │
   │                   │ ─── WalletManualSub ───>│ 按子钱包分别减款
   │                   │                         │ ※ 减款>余额时,实扣=余额
```

**链路特点：纯B端操作，财务模块发起、钱包模块被动执行**

| 步骤 | 接口 | 前置依赖 | 性质 |
|------|------|---------|------|
| 查余额 | `GetUserBalance` | 钱包模块余额数据存在 | **前置** |
| 补单入账 | `WalletCredit` + `WalletCreditReward` | 审核通过事件 | **后置** |
| 加款 | `WalletManualAdd` | 审核通过事件 | **后置** |
| 减款 | `WalletManualSub` | 审核通过事件 | **后置** |

---

### 2.5 币种配置链路（基础设施维护）

```
B端运营           ser-wallet(币种配置)
   │                   │
   │ ①配置币种         │
   │ CreateCurrency    │ (法币/加密/平台，20个字段)
   │ UpdateCurrency    │ (编辑浮动%/阈值%/限额等)
   │ SetBaseCurrency   │ (一次性设USDT，不可改)
   │ ToggleCurrencyStatus │ (启用/禁用)
   │                   │
   │                   │ ②定时任务自动执行
   │                   │ 汇率偏差检测(T1)
   │                   │ 拉3家API取平均 → 比对阈值 → 更新平台汇率
   │                   │ → 重算入款/出款汇率
   │                   │ → 生成汇率日志
   │                   │
   │ ③查看汇率日志      │
   │ QueryRateLogPage  │
```

**链路特点：完全独立，不依赖其他业务模块，是其他所有链路的根基**

---

### 2.6 记录查询链路

```
用户(C端)         ser-wallet(钱包)
   │                   │
   │ 查充值/提现/兑换/奖励记录
   │ QueryRecordPage   │ (按当前币种隔离，4个Tab)
   │ GetRecordDetail   │ (订单详情，含失败原因)
```

**链路特点：纯读取操作，依赖充值/提现/兑换产生的订单数据，是所有写操作的后置查询**

---

## 三、接口前后置依赖矩阵

### 3.1 按依赖层级分类

将所有接口按**"是否是其他接口的前置条件"**分为4层：

```
                        依赖层级图

Layer 0（地基层）── 不依赖任何业务接口，是其他一切的前提
│
├── CreateCurrency          (币种必须先存在)
├── SetBaseCurrency         (基准币种必须先设定)
├── 汇率偏差检测定时任务(T1)  (汇率数据必须先有)
│
▼
Layer 1（配置读取层）── 依赖 Layer 0 的数据，为业务层提供配置
│
├── ListActiveCurrencies    (依赖：币种已创建+已启用)
├── GetExchangeRate         (依赖：汇率数据已产生)
├── GetCurrencyConfig       (依赖：币种已配置)
├── CalcExchangeAmount      (依赖：汇率数据)
├── GetPlatformRate         (依赖：汇率数据)
├── QueryCurrencyPage       (依赖：币种已创建)
├── QueryRateLogPage        (依赖：汇率日志已产生)
│
▼
Layer 2（业务操作层）── 依赖 Layer 1 提供的配置，执行核心业务
│
├── GetDepositConfig        (依赖：ListActiveCurrencies + GetExchangeRate + 支付配置)
├── CreateDepositOrder      (依赖：GetDepositConfig)
├── GetDepositAddress       (依赖：CreateDepositOrder，仅USDT)
├── PreviewExchange         (依赖：GetExchangeRate + CalcExchangeAmount)
├── CreateExchangeOrder     (依赖：PreviewExchange)
├── GetWithdrawConfig       (依赖：ser-kyc.GetUserKycStatus + 支付配置)
├── GetWithdrawAccount      (独立查询)
├── SaveWithdrawAccount     (依赖：KYC实名信息)
├── CreateWithdrawOrder     (依赖：GetWithdrawConfig + GetWithdrawAccount + WalletFreeze)
├── GetWalletOverview       (依赖：钱包账户已初始化)
├── GetUserBalance          (依赖：钱包账户已初始化)
│
▼
Layer 3（后置响应层）── 依赖 Layer 2 的操作结果，执行后续处理
│
├── WalletCredit            (依赖：充值订单已创建 + 三方回调成功)
├── WalletCreditReward      (依赖：WalletCredit 成功 + 用户选了奖金)
├── WalletFreeze            (依赖：CreateWithdrawOrder 内部调用)
├── WalletUnfreeze          (依赖：审核驳回 或 出款失败)
├── WalletDeduct            (依赖：出款成功回调)
├── WalletManualAdd         (依赖：人工加款审核通过)
├── WalletManualSub         (依赖：人工减款审核通过)
├── DepositNotify           (依赖：链上确认回调)
├── QueryRecordPage         (依赖：有交易记录数据)
├── GetRecordDetail         (依赖：有交易记录数据)
```

### 3.2 前后置关系矩阵（核心接口）

下表展示**我负责的接口**之间的直接依赖关系（✓ = 行接口依赖列接口）：

```
被依赖方（列）→        Create  SetBase  汇率    ListActive  GetExch  CalcExch  GetCurr
依赖方（行）↓          Curr    Curr     T1定时  Currencies  Rate     Amount    Config
─────────────────────┼───────┼────────┼───────┼───────────┼────────┼─────────┼────────
ListActiveCurrencies  │  ✓    │        │       │           │        │         │
GetExchangeRate       │  ✓    │        │  ✓    │           │        │         │
CalcExchangeAmount    │       │        │  ✓    │           │  ✓     │         │
GetCurrencyConfig     │  ✓    │        │       │           │        │         │
GetDepositConfig      │       │        │       │    ✓      │  ✓     │         │
CreateDepositOrder    │       │        │       │           │        │  ✓      │
GetDepositAddress     │       │        │       │           │        │         │  (依赖CreateDepositOrder)
PreviewExchange       │       │        │       │           │  ✓     │  ✓      │
CreateExchangeOrder   │       │        │       │           │        │  ✓      │  (依赖PreviewExchange)
GetWithdrawConfig     │       │        │       │    ✓      │        │         │  (依赖KYC)
CreateWithdrawOrder   │       │        │       │           │        │         │  (依赖GetWithdrawConfig)
WalletCredit          │       │        │       │           │        │         │  ✓ (金额精度)
WalletFreeze          │       │        │       │           │        │         │  ✓ (金额精度)
WalletManualAdd/Sub   │       │        │       │           │        │         │  ✓ (金额精度)
GetUserBalance        │       │        │       │           │        │         │  ✓ (格式化)
```

---

## 四、耦合度分析

### 4.1 耦合等级定义

| 等级 | 含义 | 特征 |
|------|------|------|
| **强耦合** | 必须同时实现，缺一不可 | 同一事务中原子调用、共享状态 |
| **中耦合** | 有数据依赖但可独立部署 | A产生数据B消费，但不在同一事务 |
| **弱耦合** | 仅配置/格式依赖 | 读取配置或格式化规则 |
| **独立** | 完全无关联 | 可完全独立实现和测试 |

### 4.2 接口耦合度分类

#### 强耦合组（必须一起设计、一起实现）

```
┌──────────────────────────────────────────────────────────────┐
│ 强耦合组1: 提现事务链                                         │
│                                                              │
│  CreateWithdrawOrder ←→ WalletFreeze ←→ WalletUnfreeze      │
│                                         ←→ WalletDeduct     │
│                                                              │
│  原因：冻结/解冻/扣款是同一笔提现的状态流转，                    │
│        金额必须严格一致，状态机必须闭环                          │
│  共享状态：冻结金额、订单状态                                   │
│  事务要求：冻结和创建订单必须原子操作                            │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ 强耦合组2: 充值入账                                           │
│                                                              │
│  CreateDepositOrder ←→ WalletCredit ←→ WalletCreditReward   │
│                                                              │
│  原因：订单和入账必须关联，入账和奖金必须同一事务，               │
│        幂等性要求（防重复入账）                                  │
│  共享状态：订单号、入账状态                                     │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ 强耦合组3: 人工加减款                                         │
│                                                              │
│  GetUserBalance ←→ WalletManualAdd ←→ WalletManualSub       │
│                                                              │
│  原因：加减款前必须查余额（展示+校验），                         │
│        减款有"不超过余额"的硬约束                               │
│  共享状态：各子钱包余额                                        │
└──────────────────────────────────────────────────────────────┘
```

#### 中耦合组（有数据依赖但可独立开发）

```
┌──────────────────────────────────────────────────────────────┐
│ 中耦合组1: 兑换流程                                           │
│                                                              │
│  GetExchangeRate → PreviewExchange → CreateExchangeOrder    │
│        ↑                                                     │
│  CalcExchangeAmount (被前两者调用)                             │
│                                                              │
│  原因：数据流向单一，但汇率变化可能导致试算和实际执行不一致         │
│  注意点：CreateExchangeOrder 需要校验汇率是否过期                │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ 中耦合组2: 充值配置获取                                       │
│                                                              │
│  ListActiveCurrencies → GetDepositConfig                    │
│  GetExchangeRate      → GetDepositConfig                    │
│  (财务支付配置)        → GetDepositConfig                    │
│                                                              │
│  原因：GetDepositConfig 聚合了多个底层数据源                    │
│  注意点：任一数据源不可用，充值页面都无法正常展示                  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ 中耦合组3: 提现配置获取                                       │
│                                                              │
│  ser-kyc.GetUserKycStatus → GetWithdrawConfig               │
│  (财务提现方式配置)        → GetWithdrawConfig               │
│  提现币种规则(充值历史)    → GetWithdrawConfig               │
│                                                              │
│  原因：提现可用渠道由 KYC + 充值历史 + 财务配置 三方共同决定     │
└──────────────────────────────────────────────────────────────┘
```

#### 弱耦合（仅配置依赖）

| 接口 | 依赖的配置 | 说明 |
|------|-----------|------|
| 所有涉及金额的接口 | `GetCurrencyConfig` | 精度、千分位、小数点符号 |
| 所有写操作 | `ser-blog.AddActLog` | 审计日志，oneway异步，不影响主流程 |
| B端列表接口 | `ser-buser.QueryUserByIds` | 操作人名称展示，不影响功能 |

#### 完全独立接口

| 接口 | 说明 |
|------|------|
| `QueryCurrencyPage` | B端币种列表，纯查询 |
| `QueryRateLogPage` | B端汇率日志，纯查询 |
| `QueryRecordPage` / `GetRecordDetail` | C端记录查询，纯查询 |
| `GetWalletOverview` / `GetWalletDetail` | C端余额查询，纯查询 |
| `UpdateCurrency` | 币种编辑，独立操作 |
| `ToggleCurrencyStatus` | 启禁用，独立操作 |

---

## 五、数据流向与调用方向图

### 5.1 完整调用关系图

```
                              ┌─────────────┐
                              │  三方支付     │
                              │ (代收/代付)   │
                              └──────┬───────┘
                                     │ 回调
                                     ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         ser-finance (财务管理)                          │
│                                                                        │
│  通道配置 ◄──── 支付配置 ──── 充值记录 ──── 提现记录(审核) ──── 人工修正 │
│                                                                        │
│  ┌─ 回调处理后，调用 wallet 的 RPC 接口 ───────────────────────────┐    │
│  │                                                                │    │
│  │  充值成功 → WalletCredit + WalletCreditReward                  │    │
│  │  提现创建 → WalletFreeze                                       │    │
│  │  审核驳回 → WalletUnfreeze                                     │    │
│  │  出款成功 → WalletDeduct                                       │    │
│  │  出款失败 → WalletUnfreeze                                     │    │
│  │  补单通过 → WalletCredit + WalletCreditReward                  │    │
│  │  加款通过 → WalletManualAdd                                    │    │
│  │  减款通过 → WalletManualSub                                    │    │
│  │  加减弹窗 → GetUserBalance                                     │    │
│  │  金额处理 → GetCurrencyConfig                                  │    │
│  └────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                     ┌──────────────┼──────────────────────┐
                     │ RPC调用(9个)  │                      │ (?)读支付配置
                     ▼              ▼                      │ (反向依赖,待确认)
┌────────────────────────────────────────────────────────────────────────┐
│                         ser-wallet (钱包 + 币种)                       │
│                                                                        │
│  ┌─────────────────────────────┐    ┌──────────────────────────────┐  │
│  │      币种配置 (基础设施)      │    │       多币种钱包 (核心业务)    │  │
│  │                             │    │                              │  │
│  │  CreateCurrency             │    │  ┌── C端接口 ──────────────┐ │  │
│  │  UpdateCurrency             │    │  │ GetWalletOverview       │ │  │
│  │  SetBaseCurrency            │    │  │ GetDepositConfig        │ │  │
│  │  ToggleCurrencyStatus       │    │  │ CreateDepositOrder      │ │  │
│  │  QueryCurrencyPage          │◄───│  │ GetDepositAddress       │ │  │
│  │  QueryRateLogPage           │    │  │ ListAvailableBonus      │ │  │
│  │  GetCurrencyDetail          │    │  │ PreviewExchange         │ │  │
│  │                             │    │  │ CreateExchangeOrder     │ │  │
│  │  ── RPC对外 ──              │    │  │ GetWithdrawConfig       │ │  │
│  │  ListActiveCurrencies       │◄───│  │ GetWithdrawAccount      │ │  │
│  │  GetExchangeRate            │◄───│  │ SaveWithdrawAccount     │ │  │
│  │  GetCurrencyConfig          │◄───│  │ CreateWithdrawOrder     │ │  │
│  │  CalcExchangeAmount         │◄───│  │ QueryRecordPage         │ │  │
│  │  GetPlatformRate            │    │  │ GetRecordDetail         │ │  │
│  │                             │    │  └────────────────────────┘ │  │
│  │  ── 定时任务 ──             │    │                              │  │
│  │  汇率偏差检测(T1)           │    │  ┌── RPC对外(供finance调) ─┐ │  │
│  │                             │    │  │ WalletCredit            │ │  │
│  └─────────────────────────────┘    │  │ WalletCreditReward      │ │  │
│                                      │  │ WalletFreeze            │ │  │
│                                      │  │ WalletUnfreeze          │ │  │
│                                      │  │ WalletDeduct            │ │  │
│                                      │  │ WalletManualAdd         │ │  │
│                                      │  │ WalletManualSub         │ │  │
│                                      │  │ GetUserBalance          │ │  │
│                                      │  │ WalletConsume (预留)    │ │  │
│                                      │  │ WalletReward (预留)     │ │  │
│                                      │  └────────────────────────┘ │  │
│                                      └──────────────────────────────┘  │
└────────────────────────────┬──────────────────┬────────────────────────┘
                             │                  │
              ┌──────────────┼──────────────────┼──────────┐
              ▼              ▼                  ▼          ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
        │  ser-kyc  │  │ ser-blog │  │  ser-s3  │  │ser-buser │
        │ KYC状态   │  │ 审计日志  │  │ 图标上传  │  │ 操作人    │
        └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### 5.2 六条链路的调用方向汇总

```
链路1 充值:  C端 ──→ wallet ──→ finance ──→ 三方 ──→ finance(回调) ──→ wallet(入账)
             右→右→右→左←←←（先正向再反向回调）

链路2 提现:  C端 ──→ wallet(冻结) ──→ finance(审核) ──→ 三方(代付) ──→ finance(回调) ──→ wallet(扣/解)
             右→右→右→右→左←←←（更长的链路，更多反向回调）

链路3 兑换:  C端 ──→ wallet ──→ 币种配置(汇率) ──→ wallet(内部完成)
             右→右→左（最短链路，模块内闭环）

链路4 修正:  B端 ──→ finance ──→ wallet(加/减/入账)
             右→右（单向，无回调）

链路5 配置:  B端 ──→ 币种配置(CRUD) ──→ 定时任务(汇率更新)
             右→右（独立链路）

链路6 查询:  C端 ──→ wallet(记录) / B端 ──→ finance(记录)
             右（末端纯读取）
```

---

## 六、实现顺序建议

### 6.1 基于依赖关系的开发阶段划分

```
┌──────────────────────────────────────────────────────────────────┐
│  Phase 1: 地基（币种配置 + 钱包账户基础）                          │
│  ─────────────────────────────────────                           │
│  无任何外部依赖，可以最先启动                                      │
│                                                                  │
│  1. 数据库表：币种表、汇率日志表、钱包账户表、子钱包余额表           │
│  2. 币种CRUD：CreateCurrency / UpdateCurrency / QueryCurrencyPage │
│  3. 基准币种：SetBaseCurrency                                     │
│  4. 启禁用：ToggleCurrencyStatus                                  │
│  5. RPC对外：GetCurrencyConfig / ListActiveCurrencies             │
│  6. 钱包账户初始化逻辑（用户首次访问时创建子钱包）                   │
│  7. 余额查询：GetWalletOverview / GetUserBalance                  │
│                                                                  │
│  ✅ 完成标志：能配置币种，能查到用户子钱包余额                      │
│  ⏱  预计独立可测                                                  │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 2: 汇率体系                                               │
│  ───────────                                                     │
│  依赖 Phase 1 的币种数据                                          │
│                                                                  │
│  1. 汇率偏差检测定时任务(T1)：拉3家API → 取平均 → 比对阈值 → 更新  │
│  2. 入款/出款汇率自动计算（平台汇率 × (1±浮动%)）                   │
│  3. RPC对外：GetExchangeRate / CalcExchangeAmount / GetPlatformRate│
│  4. 汇率日志：QueryRateLogPage                                    │
│                                                                  │
│  ✅ 完成标志：汇率能自动更新，能换算金额                            │
│  ⏱  依赖 Phase 1，但不依赖其他业务模块                             │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 3: 兑换功能（钱包模块内部闭环）                              │
│  ──────────────────────────────────                              │
│  依赖 Phase 1 + 2                                                │
│                                                                  │
│  1. 兑换预览：PreviewExchange                                     │
│  2. 兑换执行：CreateExchangeOrder                                 │
│  3. 兑换记录（含在 QueryRecordPage 的兑换Tab）                     │
│                                                                  │
│  ✅ 完成标志：法币→BSB兑换全流程可走通                              │
│  ⏱  完全不依赖财务模块，可独立测试                                  │
│  ⚡ 建议：兑换是最简单独立的业务，适合作为第一个端到端验证的功能       │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 4A: 充值功能（需与财务模块协同）                             │
│  ──────────────────────────────                                  │
│  依赖 Phase 1 + 2 + 财务模块的支付配置                             │
│                                                                  │
│  1. 充值配置：GetDepositConfig                                    │
│  2. 创建充值订单：CreateDepositOrder（含重复转账拦截）               │
│  3. USDT收款地址：GetDepositAddress                               │
│  4. 额外奖金：ListAvailableBonus                                  │
│  5. 入账RPC：WalletCredit / WalletCreditReward                   │
│  6. 充值回调：DepositNotify（USDT链上场景）                        │
│  7. 充值记录（含在 QueryRecordPage 的充值Tab）                     │
│                                                                  │
│  ⚠️ 阻塞点：需要财务模块提供支付配置数据 + 回调协议                 │
│  ✅ 完成标志：至少USDT充值全流程可走通（USDT不依赖三方代收通道）     │
└──────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────────────────────────────────────────┐
│  Phase 4B: 提现功能（需与财务模块协同）                             │
│  ──────────────────────────────                                  │
│  依赖 Phase 1 + 2 + ser-kyc + 财务模块审核流程                     │
│                                                                  │
│  1. 提现配置：GetWithdrawConfig（含KYC判断+提现币种规则）            │
│  2. 提现账户：GetWithdrawAccount / SaveWithdrawAccount            │
│  3. 创建提现订单：CreateWithdrawOrder（含WalletFreeze）             │
│  4. 冻结/解冻/扣款RPC：WalletFreeze / WalletUnfreeze / WalletDeduct│
│  5. 提现记录（含在 QueryRecordPage 的提现Tab）                     │
│                                                                  │
│  ⚠️ 阻塞点：需要 ser-kyc 提供KYC查询接口 + 财务模块审核+出款流程   │
│  ✅ 完成标志：提现发起→冻结→审核→扣款/解冻 全状态可流转             │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 5: 人工修正对接 + 兜底能力                                  │
│  ──────────────────────                                          │
│  依赖 Phase 1 + 4A + 4B                                          │
│                                                                  │
│  1. WalletManualAdd（人工加款）                                    │
│  2. WalletManualSub（人工减款，含余额不足兜底）                     │
│  3. 补单入账（复用 WalletCredit + WalletCreditReward）             │
│                                                                  │
│  ✅ 完成标志：财务模块能通过RPC对钱包执行加减款                      │
│  ⏱  钱包侧实现简单，主要是接口契约对齐                              │
└──────────────────────────────────────────────────────────────────┘
```

### 6.2 阶段依赖关系图

```
Phase 1 ──→ Phase 2 ──→ Phase 3 (兑换，独立可测)
                │
                ├──→ Phase 4A (充值，需finance协同)
                │
                ├──→ Phase 4B (提现，需finance+kyc协同)
                │
                └──→ Phase 5 (人工修正，需finance协同)

※ Phase 4A 和 4B 互相独立，可并行开发
※ Phase 3 完全独立，可作为第一个端到端验证
※ Phase 5 依赖 Phase 4A/4B 的订单/余额结构，但实现简单
```

### 6.3 开发优先级排序

| 优先级 | 内容 | 理由 | 是否可独立测试 |
|--------|------|------|-------------|
| **P0** | 币种CRUD + 基准币种 + 币种RPC | 所有模块的地基，0依赖 | ✅ 完全独立 |
| **P0** | 钱包账户初始化 + 余额查询 | 所有钱包操作的前提 | ✅ 完全独立 |
| **P1** | 汇率定时任务 + 汇率RPC | 充值/兑换/提现都需要汇率 | ✅ 仅依赖P0 |
| **P2** | 兑换全流程 | 最简单的端到端业务，不依赖外部 | ✅ 仅依赖P0+P1 |
| **P3** | 充值全流程 | 核心收入链路，但需要finance协同 | ⚠️ USDT可独立，银行/电子钱包需finance |
| **P3** | 提现全流程 | 核心支出链路，需要finance+kyc协同 | ⚠️ 需要KYC和财务审核 |
| **P4** | 人工加减款RPC | 兜底能力，接口简单 | ⚠️ 需要finance触发 |
| **P4** | 记录查询 | 纯读取，跟着业务功能自然完成 | ✅ 依赖有数据 |

---

## 七、风险点与待确认项

### 7.1 架构层面的关键待确认项

这些问题直接影响模块间的依赖方向和接口设计，必须在正式开发前与财务模块负责人对齐：

| # | 待确认项 | 两种可能 | 影响范围 |
|---|---------|---------|---------|
| **1** | **充值发起时wallet和finance的调用方向** | A: wallet创建订单后RPC调finance获取支付通道和跳转URL<br>B: wallet创建订单后发事件，finance订阅后主动拉取 | 影响充值链路的接口契约和数据流向 |
| **2** | **提现创建后wallet怎么通知finance** | A: wallet内部直接RPC调finance的"提交审核"接口<br>B: wallet写消息队列，finance消费 | 影响提现链路的耦合度 |
| **3** | **USDT链上充值的回调归谁处理** | A: 区块链监听服务直接回调wallet，wallet自行入账<br>B: 回调先到finance，finance统一处理后调wallet入账 | 影响DepositNotify的归属 |
| **4** | **充值/提现订单数据放哪个库** | A: wallet库（wallet是订单owner）<br>B: finance库（finance管订单，wallet只管钱）<br>C: 各自存一份（双写+最终一致） | 影响数据归属和查询效率 |
| **5** | **支付配置数据wallet怎么获取** | A: wallet启动时从finance拉取配置缓存本地<br>B: wallet实时RPC调finance查询<br>C: 配置存公共配置中心，两边各自读 | 影响充值/提现页面的配置获取链路 |

### 7.2 依赖风险

| 风险 | 说明 | 影响 | 缓解措施 |
|------|------|------|---------|
| **finance模块进度阻塞** | 充值和提现链路依赖finance的回调/审核，如果finance延期，wallet无法端到端测试 | 充值入账、提现出款无法验证 | Phase 3（兑换）先行；充值/提现钱包侧可先实现，用mock模拟finance回调 |
| **ser-kyc接口不可用** | 提现依赖KYC状态查询 | 提现渠道判断失效 | 提现模块做降级处理：KYC不可用时按"未认证"处理（仅USDT） |
| **汇率API不可用** | 三家汇率API都挂了 | 汇率无法更新 | 使用最近一次有效的平台汇率（最终一致性兜底） |
| **反向依赖不清晰** | wallet需要读finance的支付配置，但finance也调wallet | 循环依赖风险 | 支付配置建议下沉为公共配置或wallet缓存，避免运行时循环调用 |

### 7.3 接口契约对齐清单

以下接口需要与财务模块负责人共同定义，确保双方理解一致：

| 接口 | wallet侧职责 | finance侧职责 | 需对齐的内容 |
|------|-------------|-------------|-------------|
| `WalletCredit` | 校验订单号幂等 → 入账中心钱包 → 记录流水 | 充值成功/补单通过后调用 | 入参结构（订单号/用户ID/币种/金额/来源类型） |
| `WalletCreditReward` | 入账奖励钱包 → 标记流水稽核要求 | 奖金发放时调用 | 入参结构（关联订单号/奖金金额/流水倍数） |
| `WalletFreeze` | 冻结金额 → 更新可用余额 | 创建提现订单时调用 | 入参结构（用户ID/币种/金额/关联提现单号） |
| `WalletUnfreeze` | 解冻归还 → 恢复可用余额 | 审核驳回/出款失败时调用 | 入参结构（冻结记录ID或提现单号） |
| `WalletDeduct` | 最终扣除冻结资金 → 记录流水 | 出款成功回调后调用 | 入参结构（冻结记录ID或提现单号） |
| `WalletManualAdd` | 按子钱包分别加款 → 记录流水 | 加款审核通过后调用 | 入参结构（用户ID/币种/各子钱包加款金额map/关联修正单号） |
| `WalletManualSub` | 按子钱包分别减款（不超余额） → 记录流水 | 减款审核通过后调用 | 入参结构（同上） + 返回实际扣款金额 |
| `GetUserBalance` | 返回各子钱包余额（含冻结） | 加减款弹窗展示 | 返回结构（各子钱包可用/冻结/总额） |
| `GetCurrencyConfig` | 返回币种完整配置 | 金额格式化时调用 | 返回结构（精度/符号/千分位/类型） |

---

## 附录：一页纸速查

### A. 接口按独立性分类速查

```
╔══════════════════════════════════════════════════════════════════╗
║  完全独立（可任意时间实现，不阻塞也不被阻塞）                       ║
║                                                                  ║
║  · QueryCurrencyPage        · QueryRateLogPage                  ║
║  · GetCurrencyDetail        · UpdateCurrency                    ║
║  · ToggleCurrencyStatus     · GetWalletOverview                 ║
║  · GetWalletDetail          · QueryRecordPage                   ║
║  · GetRecordDetail                                              ║
╠══════════════════════════════════════════════════════════════════╣
║  前置接口（必须先实现，后续功能依赖它们）                           ║
║                                                                  ║
║  · CreateCurrency ──────┐                                       ║
║  · SetBaseCurrency ─────┤→ 一切金额操作的前提                    ║
║  · 汇率定时任务(T1) ────┘                                       ║
║  · ListActiveCurrencies ─→ C端所有页面的币种下拉                  ║
║  · GetExchangeRate ──────→ 充值/兑换/提现的汇率展示               ║
║  · GetCurrencyConfig ───→ 所有金额处理的精度/格式                 ║
║  · CalcExchangeAmount ──→ 兑换/充值的金额换算                    ║
║  · GetUserKycStatus ────→ 提现渠道判断                           ║
╠══════════════════════════════════════════════════════════════════╣
║  核心业务接口（中间层，承上启下）                                   ║
║                                                                  ║
║  · GetDepositConfig ────→ 充值页面渲染                           ║
║  · CreateDepositOrder ──→ 充值订单创建                           ║
║  · GetDepositAddress ───→ USDT收款地址                           ║
║  · PreviewExchange ─────→ 兑换试算                               ║
║  · CreateExchangeOrder ─→ 兑换执行                               ║
║  · GetWithdrawConfig ───→ 提现页面渲染                           ║
║  · SaveWithdrawAccount ─→ 提现账户绑定                           ║
║  · CreateWithdrawOrder ─→ 提现订单创建+冻结                      ║
║  · GetUserBalance ──────→ 供finance读取余额                      ║
╠══════════════════════════════════════════════════════════════════╣
║  后置接口（被动响应，由外部事件触发）                               ║
║                                                                  ║
║  · WalletCredit ────────← finance充值成功/补单通过               ║
║  · WalletCreditReward ──← finance奖金发放                       ║
║  · WalletFreeze ────────← 提现订单创建（内部）                    ║
║  · WalletUnfreeze ──────← finance审核驳回/出款失败               ║
║  · WalletDeduct ────────← finance出款成功                       ║
║  · WalletManualAdd ─────← finance加款审核通过                    ║
║  · WalletManualSub ─────← finance减款审核通过                    ║
║  · DepositNotify ───────← 区块链监听服务回调                     ║
╚══════════════════════════════════════════════════════════════════╝
```

### B. 模块间9条RPC调用链路速查

```
finance ──→ wallet 的调用（9条）：

  ① WalletCredit         充值入账/补单入账 → 中心钱包+金额
  ② WalletCreditReward   奖金入账         → 奖励钱包+金额+流水倍数
  ③ WalletFreeze         提现冻结         → 冻结金额
  ④ WalletUnfreeze       提现解冻         → 解冻金额
  ⑤ WalletDeduct         提现扣款         → 扣除冻结
  ⑥ WalletManualAdd      人工加款         → 各子钱包+金额
  ⑦ WalletManualSub      人工减款         → 各子钱包+金额(不超余额)
  ⑧ GetUserBalance       查余额           → 各子钱包余额(含冻结)
  ⑨ GetCurrencyConfig    查币种配置        → 精度/符号/类型

wallet ──→ 外部的调用（4条）：

  ① ser-kyc.GetUserKycStatus   提现渠道判断
  ② ser-blog.AddActLog         审计日志(oneway)
  ③ ser-s3.UploadFile           图标上传
  ④ ser-buser.QueryUserByIds   操作人名称

wallet内部调用（币种配置 → 钱包）（5条）：

  ① ListActiveCurrencies       C端可用币种
  ② GetExchangeRate            当前汇率
  ③ GetCurrencyConfig          币种精度/格式
  ④ CalcExchangeAmount         汇率换算
  ⑤ GetPlatformRate            批量平台汇率
```

---
---

# 补充：三大模块间依赖关系文字描述

> 追加时间：2026-02-26
> 目的：用纯文字+箭头的方式，从业务视角完整描述币种配置、多币种钱包、财务管理三大模块之间的依赖调用关系
> 信息源：产品需求文档（币种配置24张、钱包需求42张+原型32张、财务管理25张+原型26张）

---

## 一、三模块的角色定位与总体依赖方向

### 1.1 各模块角色一句话定义

**币种配置模块**是"规则制定者"——它定义平台用什么货币、汇率怎么算、精度怎么处理。自身不产生任何业务行为，只提供基础数据和计算能力。

**多币种钱包模块**是"资金执行者"——它管理用户的钱，负责充值入账、消费扣款、提现冻结、兑换换算。面向C端用户提供操作界面，面向其他模块暴露资金操作RPC接口。

**财务管理模块**是"运营管控者"——它管理支付通道、审核提现订单、处理异常修正、统计财务数据。面向B端运营和财务人员，通过调用钱包的RPC接口来间接操作用户资金。

### 1.2 总体依赖方向

从产品需求文档中可以明确提炼出三个模块的依赖方向：

**币种配置 ← 钱包 ← 财务**

具体来说：

- 币种配置模块**不依赖**钱包和财务，它是独立的底层基础设施。
- 钱包模块**依赖**币种配置（获取汇率、精度、币种列表），**不依赖**财务模块（但有一个待确认的反向读取，后面细说）。
- 财务模块**依赖**钱包模块（入账、冻结、解冻、扣款、加减款、查余额），也**依赖**币种配置（金额精度处理）。

用一个比喻：**币种配置是地基，钱包是楼体，财务是物业管理**。物业管理可以操控楼里的设施（调钱包接口），但不能改地基（币种规则由币种配置独立管理）；楼体必须建在地基上（钱包所有金额操作依赖币种配置），但不需要物业先就位就能盖起来。

---

## 二、币种配置模块的依赖关系

### 2.1 币种配置对外提供什么

币种配置模块是纯粹的"被依赖方"，其他模块单方面读取它的数据，它不需要调用任何业务模块。

**提供给钱包模块的能力：**

- 可用币种列表 → 钱包C端的币种切换下拉框需要读取哪些币种已启用
- 当前汇率（入款/出款方向）→ 充值页面展示汇率、提现页面展示换算、兑换页面计算到账金额
- 汇率换算计算 → 兑换时法币转BSB的精确计算、充值USDT时的金额换算
- 币种精度和格式化规则 → 所有金额展示和计算都需要知道保留几位小数、千分位符号是什么

**提供给财务模块的能力：**

- 币种配置信息 → 财务模块处理订单金额时需要知道精度规则
- 平台汇率 → 统计报表需要将各币种金额统一转换为基准币种USDT

### 2.2 币种配置依赖什么

**外部服务依赖（非业务模块）：**

- 三方汇率API → 定时拉取实时汇率，这是汇率体系的数据源头
- ser-s3 → 币种图标上传存储
- ser-blog → B端配置操作的审计日志

**不依赖任何业务模块**。这意味着币种配置可以完全独立开发、独立测试、独立部署。

### 2.3 币种配置的内部依赖链

币种配置模块内部也有前后置关系：

- `CreateCurrency`（创建币种）→ 是一切的起点，没有币种数据，后续所有操作都无从谈起
- `SetBaseCurrency`（设定基准币种USDT）→ 必须在汇率体系运转前完成，因为所有汇率换算都以USDT为锚点
- 汇率定时任务 → 依赖币种已创建且已配置汇率浮动%和阈值%，否则无法计算偏差和触发更新
- `GetExchangeRate` / `CalcExchangeAmount` → 依赖汇率定时任务已产出有效的平台汇率数据
- `QueryRateLogPage` → 依赖汇率定时任务产生的日志记录

用箭头表示：

`CreateCurrency` → `SetBaseCurrency` → 汇率定时任务启动 → `GetExchangeRate` / `CalcExchangeAmount` 可用 → 所有依赖汇率的业务功能才能运转

---

## 三、钱包模块的依赖关系

### 3.1 钱包模块向下依赖（钱包 → 币种配置）

钱包模块的几乎每个功能都需要读取币种配置的数据，这是最基础的依赖关系：

**充值功能依赖币种配置：**
- 用户进入充值页面 → 需要调用 `ListActiveCurrencies` 获取可切换的币种列表
- 用户选择BSB充值 → 页面需要展示"1 BSB = X USDT"的汇率，调用 `GetExchangeRate`
- 用户输入金额后系统换算USDT → 调用 `CalcExchangeAmount` 精确计算
- 充值金额的展示格式 → 调用 `GetCurrencyConfig` 获取千分位符号和小数精度

**兑换功能依赖币种配置：**
- 兑换预览试算 → `PreviewExchange` 内部调用 `CalcExchangeAmount`，基于当前汇率计算"法币金额 / 汇率 = BSB数量"
- 兑换执行 → `CreateExchangeOrder` 再次调用 `CalcExchangeAmount` 确认最终汇率和金额
- 兑换页面展示口径"1 BSB = X 法币"→ 调用 `GetExchangeRate`

**提现功能依赖币种配置：**
- BSB提现时需要展示换算后的法币/USDT金额 → 调用 `GetExchangeRate`
- 提现金额的精度处理 → 调用 `GetCurrencyConfig`

**余额展示依赖币种配置：**
- 用户看到的余额数字格式（VND用`.`做千分位无小数，THB用`,`做千分位保留2位小数）→ 依赖 `GetCurrencyConfig` 返回的格式化规则

**依据（需求文档）：** 币种配置需求文档明确指出"精度规则：法币2位小数，加密货币8位小数，不足补0超过四舍五入"，钱包原型中所有金额展示都遵循此规则。汇率体系的四种汇率（实时/平台/入款/出款）在充值、提现、兑换页面有不同的使用场景。

### 3.2 钱包模块向外依赖（钱包 → 外部服务）

**钱包 → ser-kyc（提现场景）：**
- 依据：钱包需求文档"提现"章节明确规定"KYC未认证用户仅支持USDT提现，已认证用户支持全渠道"
- 用户点击提现 → 钱包调用 `ser-kyc.GetUserKycStatus` 查询认证状态 → 决定展示哪些提现方式
- 用户首次保存提现账户 → 账户持有人必须等于KYC实名（忽略大小写和多余空格比对），需要读取KYC姓名信息
- 这是**强依赖**：没有KYC状态，提现流程无法正常展示可用渠道

**钱包 → ser-blog（所有写操作）：**
- 依据：工程强制规范，不是需求文档要求，但工程中每个服务都遵守
- 币种的创建/编辑/启禁用、充值/提现/兑换订单的创建、余额变动等所有写操作 → 异步上报审计日志
- 这是**弱依赖**：oneway异步调用，日志服务不可用不影响主流程

**钱包 → ser-s3（币种图标上传）：**
- 依据：币种配置需求文档"编辑弹窗支持上传币种图标，格式SVG/PNG不超过5KB"
- B端编辑币种时上传图标文件 → 调用 ser-s3 存储 → 返回图标URL写入币种配置
- 这是**弱依赖**：仅B端配置场景用到，C端不触发

### 3.3 钱包模块对上提供（钱包 → 被财务调用）

这是钱包模块最核心的对外接口契约，也是与财务模块最关键的依赖关系。钱包作为"资金执行者"，必须暴露以下RPC接口供财务模块调用：

**入账类：**

- `WalletCredit`：充值金额入中心钱包。财务模块在收到三方代收成功回调后调用此接口。也在补单审核通过后调用。
  - 依据：钱包需求文档"充值流程"明确了"三方回调 → 平台更新状态 → 入账"的链路；财务需求文档"补单"章节描述"审核通过后通知钱包入账"

- `WalletCreditReward`：额外奖金入奖励钱包。充值时如果用户选择了奖金活动，充值入账的同时需要把奖金部分单独入奖励钱包并标记流水稽核要求。
  - 依据：钱包需求文档"额外奖金"章节"入账规则：充值金额→中心钱包，奖金金额→奖励钱包"

**冻结/解冻/扣款类（提现事务链）：**

- `WalletFreeze`：提现发起时冻结对应金额。用户创建提现订单后，钱包立即冻结该笔金额，冻结期间资金不可使用。
  - 依据：钱包需求文档"提现流程"明确写了"用户发起 → 钱包冻结对应金额 → 风控审核 → 财务审核 → 出款"

- `WalletUnfreeze`：三种场景触发解冻归还——风控审核驳回、财务审核驳回、代付出款失败。
  - 依据：财务需求文档"提现记录管理"章节"驳回后：通知钱包解冻金额"

- `WalletDeduct`：代付出款成功后，最终扣除已冻结的资金。这一步才是真正的"钱走了"。
  - 依据：钱包需求文档"审核通过+出款成功 → 最终扣除冻结资金"

- 这三个接口构成一个**状态机闭环**：冻结(Freeze) → 解冻(Unfreeze) 或 扣款(Deduct)。每一笔提现必须以 Unfreeze 或 Deduct 终结，不能停在 Freeze 状态。

**人工修正类：**

- `WalletManualAdd`：人工加款审核通过后，按指定的子钱包（中心/奖励/主播/代理）分别加款。
  - 依据：财务需求文档"人工加款"章节描述了"选择用户 → 查询展示当前四种子钱包余额 → 分别填写加款金额 → 提交审核 → 审核通过 → 通知钱包按子钱包分别加款"

- `WalletManualSub`：人工减款审核通过后，按指定的子钱包分别减款。有一条特殊规则：如果减款金额 > 当前余额，实际扣款 = 当前余额（不会变成负数）。
  - 依据：财务需求文档"人工减款"章节"特殊规则：如果减款金额 > 当前余额，实际扣款 = 当前余额"

**查询类：**

- `GetUserBalance`：返回用户各子钱包当前余额（含冻结金额）。财务模块在人工加/减款时需要先展示用户余额弹窗。
  - 依据：财务需求文档"加款/减款弹窗需要展示用户当前各子钱包余额"

- `GetCurrencyConfig`：返回币种配置信息。财务模块在处理订单金额时需要知道精度和格式化规则。

### 3.4 钱包模块可能的反向依赖（钱包 → 财务，待确认）

产品需求文档中存在一个隐含的反向依赖，需要特别说明：

**支付配置的读取方向：**

钱包C端的充值页面需要展示"当前币种有哪些可用的充值方式（USDT/银行转账/电子钱包）"以及"每种方式的限额、手续费"。这些配置数据是在**财务模块的支付配置**中管理的（财务需求文档"支付配置"章节明确由财务B端配置各币种可用的充值方式和提现方式）。

同理，提现页面需要展示"当前币种有哪些可用的提现方式"以及"单笔限额、手续费率、到账时间"，这些也是财务的支付配置数据。

这就产生了一个问题：钱包C端渲染页面时，是否需要实时RPC调用财务模块读取这些配置？如果是，就形成了**钱包→财务→钱包**的循环依赖。

这个问题待确认，可能的解决方案有三种：
1. 财务配置变更时主动推送/同步到钱包侧缓存，钱包不实时调财务
2. 支付配置下沉到公共配置中心，两个模块各自读取
3. 钱包确实实时调财务（技术上可行但架构上不优雅）

---

## 四、财务管理模块的依赖关系

### 4.1 财务模块向下依赖（财务 → 钱包）

财务模块是钱包模块最大的"消费方"，通过9个RPC接口调用钱包的资金操作能力。按业务场景逐一说明：

**充值成功入账场景：**

三方代收回调到达财务模块 → 财务模块验证回调合法性、更新订单状态 → 调用钱包的 `WalletCredit` 将充值金额入用户中心钱包 → 如果用户选了额外奖金活动，再调用 `WalletCreditReward` 将奖金金额入用户奖励钱包。

箭头表示：三方回调 → 财务.CollectNotify → 钱包.WalletCredit → 钱包.WalletCreditReward

依据：财务需求文档"充值记录"章节描述了订单状态从"待支付"到"已完成"的流转，钱包需求文档"充值回调"章节要求幂等处理防重复入账。

**提现审核场景：**

用户在C端发起提现 → 钱包创建订单并冻结金额 → 通知财务开始审核 → 财务侧进行风控审核（通过/驳回）→ 财务侧进行财务审核（通过/驳回）→ 审核全部通过后发起代付出款 → 三方代付回调 → 财务根据结果调用钱包的扣款或解冻接口。

箭头表示（审核通过+出款成功的完整链路）：
C端 → 钱包.CreateWithdrawOrder → 钱包.WalletFreeze（冻结）→ 财务.风控审核通过 → 财务.财务审核通过 → 财务.发起代付 → 三方回调 → 财务.PayoutNotify → 钱包.WalletDeduct（最终扣款）

箭头表示（任一环节失败的链路）：
财务.风控驳回 → 钱包.WalletUnfreeze（解冻归还）
财务.财务驳回 → 钱包.WalletUnfreeze（解冻归还）
三方出款失败 → 财务.PayoutNotify → 钱包.WalletUnfreeze（解冻归还）

依据：财务需求文档"提现记录管理"章节"双审核流程：风控审核 → 财务审核"，"创建人与审核人不能是同一人"，"驳回后：通知钱包解冻金额，通过后：发起代付出款"。

**补单场景：**

B端运营在财务模块发起补单（用户充值了但系统未到账）→ 提交审核 → 审核通过 → 财务调用钱包的 `WalletCredit` 补充入账 → 如果需要补充额外奖金，再调用 `WalletCreditReward`。

箭头表示：B端 → 财务.CreateSupplementOrder → 财务.ReviewCorrectionOrder（审核通过）→ 钱包.WalletCredit + 钱包.WalletCreditReward

依据：财务需求文档"补单"章节"场景：用户充值但系统未到账（回调丢失等），操作：补充充值金额+可补充额外奖金"。

**人工加款场景：**

B端运营在财务模块发起加款 → 首先调用钱包的 `GetUserBalance` 查询该用户当前各子钱包余额 → 弹窗展示余额 → 运营分别填写各子钱包的加款金额 → 提交审核 → 审核通过 → 财务调用钱包的 `WalletManualAdd` 按子钱包分别加款。

箭头表示：B端 → 财务.CreateManualAddOrder → 钱包.GetUserBalance（展示余额）→ 财务.ReviewCorrectionOrder（审核通过）→ 钱包.WalletManualAdd

依据：财务需求文档"人工加款"章节"选择用户 → 查询展示当前四种子钱包余额 → 分别填写加款金额"，加款类型包括"系统补偿、活动补发、财务冲正、其他"。

**人工减款场景：**

同加款流程，区别在于最后调用 `WalletManualSub`，并且有一条硬规则：减款金额不能把余额扣成负数。

箭头表示：B端 → 财务.CreateManualSubOrder → 钱包.GetUserBalance（展示余额）→ 财务.ReviewCorrectionOrder（审核通过）→ 钱包.WalletManualSub

依据：财务需求文档"人工减款"章节"特殊规则：如果减款金额 > 当前余额，实际扣款 = 当前余额（不会变成负数）"。

### 4.2 财务模块向下依赖（财务 → 币种配置）

财务模块也需要读取币种配置数据，主要在以下场景：

- 订单金额的精度处理和格式化展示 → 读取 `GetCurrencyConfig`
- 统计报表中将各币种金额统一转换为基准币种USDT → 读取 `GetPlatformRate` 或 `CalcExchangeAmount`
- 通道配置中每个通道绑定的币种信息 → 读取币种基础数据

依据：财务需求文档"数据统计"章节"所有金额统一转换为基准币种（USDT）展示"，以及"时区处理：服务器数据以UTC基准时间记录"。

### 4.3 财务模块向外依赖（财务 → 三方支付）

这是财务模块最核心的外部依赖：

- 代收（充值）：财务模块向三方支付通道发起代收请求，接收三方回调
- 代付（提现）：财务模块向三方代付通道发起付款请求，接收三方回调
- 通道轮询：财务模块根据权重+成功率+预警系数，自动选择最优通道

箭头表示：财务 ←→ 三方支付（双向：请求+回调）

依据：财务需求文档"通道配置"章节详细描述了代收/代付通道的配置字段（请求地址、商户ID、密钥、AppKey、支付编码、公钥、私钥等）以及"通道轮询规则"（三维度评分：成功率系数+时间系数+预警系数=总分数）。

### 4.4 财务模块不被业务模块反向调用

从产品需求文档来看，财务模块是纯粹的"调用方"和"管理方"，不需要暴露RPC接口给钱包或币种配置调用（支付配置的读取方向待确认除外）。它的所有功能都是B端操作或回调处理，不直接面向C端用户。

---

## 五、三模块之间的具体联动场景分析

### 5.1 充值场景的模块联动

充值涉及三个模块的联动，是最能体现依赖关系的场景。

**用户视角**：用户在C端选择一种充值方式，输入金额，完成支付后资金到账。

**模块间发生了什么**：

第一步，页面渲染：钱包 → 币种配置。钱包模块需要从币种配置获取当前有哪些可用币种（`ListActiveCurrencies`），以及当前汇率（`GetExchangeRate`）。同时需要知道当前币种有哪些可用的充值方式——这个数据来源是财务的支付配置（依赖方向待确认）。

第二步，创建订单：钱包内部完成。钱包校验重复转账拦截规则（同UserID+同币种+同金额，查询pending/processing状态的订单数），通过后创建充值订单。

第三步，支付发起：钱包 → 财务 → 三方（仅银行转账/电子钱包场景）。钱包创建订单后，需要通知财务模块选择合适的支付通道、向三方发起代收请求、获取三方的支付跳转URL或二维码。USDT充值是例外——USDT走平台承载页，由钱包自己生成收款地址，不经过财务模块的代收通道。

第四步，支付回调入账：三方 → 财务 → 钱包。三方支付回调到达财务模块（`CollectNotify`），财务验证后调用钱包的 `WalletCredit` 将金额入用户中心钱包。如果有额外奖金，还调用 `WalletCreditReward` 入奖励钱包。USDT充值的回调可能直接到达钱包（区块链监听服务直接通知钱包的 `DepositNotify`），这个归属待确认。

**联动总结**：

- 币种配置 → 提供汇率和币种数据（前置，被动被调）
- 钱包 → 创建订单、管理重复拦截、最终入账（核心执行者）
- 财务 → 选通道、发起代收、接收回调、触发入账（中间协调者）
- 三方 → 实际收款和回调（外部系统）

### 5.2 提现场景的模块联动

提现的链路最长，涉及的模块交互最多，也是依赖最复杂的场景。

**第一步，提现资格判断**：钱包 → ser-kyc + 币种配置 + 财务（配置）。钱包需要综合三方面信息来决定用户能用什么方式提现：(1) KYC认证状态决定渠道范围，(2) 用户的充值历史决定提现币种规则，(3) 财务的提现方式配置决定限额和手续费。这三者缺一不可。

**第二步，提现账户校验**：钱包 → ser-kyc。用户首次提现需要填写账户信息（银行卡号/电子钱包/USDT地址），其中账户持有人必须等于KYC实名姓名，钱包需要从KYC获取实名信息做比对。

**第三步，创建订单+冻结**：钱包内部完成。钱包校验限额后创建提现订单，同时调用内部的 `WalletFreeze` 冻结对应金额。冻结和创建订单是原子操作——要么都成功，要么都失败。

**第四步，审核**：钱包 → 财务。钱包创建订单后需要通知财务开始审核流程。财务侧执行风控审核和财务审核两道关卡，创建人与审核人不能是同一人。

**第五步，审核结果处理**：财务 → 钱包。如果任一审核环节驳回，财务调用钱包的 `WalletUnfreeze` 解冻归还冻结金额。如果全部通过，财务发起代付出款。

**第六步，出款结果处理**：三方 → 财务 → 钱包。出款成功回调到达后，财务调用钱包的 `WalletDeduct` 最终扣除冻结资金。出款失败回调到达后，财务调用钱包的 `WalletUnfreeze` 解冻归还。

**联动总结**：

提现链路跨越了所有三个模块加上外部KYC和三方支付，是依赖最重的场景。其中冻结→解冻/扣款这条线是**强耦合的事务链**，金额必须严格一致、状态必须闭环。

### 5.3 兑换场景的模块联动

兑换是三个场景中最简单的，**完全不涉及财务模块**，也**不涉及三方支付**。

**全流程**：用户在法币钱包页面点击兑换 → 钱包从币种配置获取当前汇率 → 用户输入金额 → 钱包调用 `CalcExchangeAmount` 试算（兑换BSB + 赠送BSB + 实际到账BSB）→ 用户确认 → 钱包执行兑换（法币子钱包扣减 + BSB子钱包增加）→ 赠送部分标记流水稽核要求 → 3秒内返回成功。

**联动总结**：

- 币种配置 → 提供汇率和换算计算（前置，被动被调）
- 钱包 → 从头到尾自己完成（核心执行者）
- 财务 → 不参与（无关）

这意味着兑换功能可以在财务模块还没开发的时候就独立测试验证。

### 5.4 人工修正场景的模块联动

人工修正（补单/加款/减款）是纯B端操作，由财务模块发起，钱包模块被动执行。

**方向单一**：所有操作都是 B端运营 → 财务模块 → 钱包模块，不存在反向调用，不存在回调，不涉及三方。

**加/减款的特殊依赖**：财务模块在发起加/减款申请时，需要先调用钱包的 `GetUserBalance` 查询用户当前各子钱包余额，在弹窗中展示给运营人员看，运营人员依据余额分别填写各子钱包的加/减金额。这意味着**加减款弹窗的渲染依赖钱包的余额查询接口**。

**联动总结**：

- 币种配置 → 提供精度处理（弱依赖）
- 钱包 → 提供余额查询 + 执行加减款（被动响应方）
- 财务 → 发起操作 + 审核 + 触发执行（主动方）

---

## 六、跨模块依赖的关键约束总结

### 6.1 数据一致性约束

以下是从产品需求文档中提炼出的、跨模块必须遵守的数据一致性约束：

**充值幂等约束**：同一笔充值订单的入账操作（`WalletCredit`）必须幂等处理。如果财务模块因网络抖动重复调用，钱包不能重复入账。判断依据是订单号唯一性。

依据：钱包需求文档"非功能性需求"章节"充值回调必须做幂等处理，防止重复入账"。

**冻结余额约束**：提现冻结的金额必须从用户可用余额中扣除，冻结期间其他操作（充值入账、消费扣款）不影响已冻结部分。最终结果只有两种：扣款（`WalletDeduct`）或解冻（`WalletUnfreeze`），不允许冻结后永久挂起。

依据：钱包需求文档"提现流程"章节明确了"冻结 → 审核通过+出款成功 → 最终扣除冻结资金"和"审核驳回/出款失败 → 解冻归还冻结金额"。

**减款不负约束**：人工减款时，如果减款金额大于当前子钱包余额，实际扣款等于当前余额，不会产生负余额。这个约束在钱包侧执行，财务侧传入的是"期望减款金额"，钱包返回"实际减款金额"。

依据：财务需求文档"人工减款"章节"特殊规则：如果减款金额 > 当前余额，实际扣款 = 当前余额"。

**订单号唯一性约束**：充值(C)、提现(T)、补单(B)、加款(A)、减款(M)五类订单号格式不同（前缀+16位随机数），跨模块调用时以订单号作为关联凭据和幂等键。

依据：财务需求文档明确给出了五种订单号前缀规范。

### 6.2 时序约束

以下是接口调用必须遵守的时序顺序，打破顺序会导致业务异常：

- `SetBaseCurrency` 必须在系统首次使用前完成，且不可修改 → 否则后续所有汇率换算没有锚点
- 汇率定时任务必须在充值/兑换/提现功能上线前运行 → 否则没有平台汇率数据可用
- `WalletFreeze` 必须在 `CreateWithdrawOrder` 的同一事务中完成 → 否则可能出现创建了订单但钱没冻结的不一致
- `WalletCredit` 必须在 `WalletCreditReward` 之前执行 → 充值金额先入中心钱包，再入奖金到奖励钱包
- `GetUserBalance` 必须在 `WalletManualAdd` / `WalletManualSub` 之前调用 → 运营需要先看到余额才能填写合理的加减金额

### 6.3 独立性确认

以下功能之间**完全独立，没有任何依赖关系**，可以由不同的人并行开发：

- 币种CRUD ↔ 钱包账户初始化（互不影响，各自建各自的表）
- 兑换功能 ↔ 充值功能 ↔ 提现功能（三者之间无数据依赖，只共享币种配置和余额数据）
- 充值记录查询 ↔ 提现记录查询 ↔ 兑换记录查询（三种记录各自独立，只是前端放在同一个Tab组件里）
- 币种配置的B端管理 ↔ 财务的通道配置（各管各的配置，互不干扰）
- 汇率日志查询 ↔ 业务记录查询（不同类型的日志/记录，互不影响）

---

## 七、一句话总结每条依赖线

| 依赖关系 | 一句话描述 | 依赖强度 |
|---------|-----------|---------|
| 钱包 → 币种配置（汇率） | 钱包的充值、兑换、提现都需要从币种配置获取实时汇率来做金额换算 | **强依赖**，无汇率则无法操作 |
| 钱包 → 币种配置（精度） | 钱包所有金额的展示和计算都要遵循币种配置定义的精度规则（法币2位/加密8位） | **强依赖**，无精度则金额显示错误 |
| 钱包 → 币种配置（币种列表） | 钱包C端的币种切换下拉依赖币种配置中哪些币种是启用状态 | **强依赖**，无列表则页面无法渲染 |
| 钱包 → ser-kyc | 提现时需要查KYC认证状态来决定可用渠道，保存提现账户时需比对KYC实名 | **强依赖**（仅提现场景） |
| 财务 → 钱包（入账） | 充值成功和补单通过后，财务调钱包的入账接口把钱放进用户中心钱包和奖励钱包 | **强依赖**，不调则用户充了钱但余额不变 |
| 财务 → 钱包（冻结/解冻/扣款） | 提现的完整状态机（冻结→审核→扣款/解冻）需要财务在不同节点调用钱包的对应接口 | **强依赖**，不调则提现流程断裂 |
| 财务 → 钱包（加减款） | 人工修正审核通过后，财务调钱包的加减款接口修改用户余额 | **强依赖**，不调则修正无法生效 |
| 财务 → 钱包（查余额） | 人工加减款的弹窗需要先展示用户当前余额 | **中依赖**，不调则运营盲填 |
| 财务 → 币种配置（精度） | 财务的订单金额展示和统计报表转换需要币种精度 | **中依赖** |
| 钱包 → 财务（支付配置） | 钱包C端需要知道当前币种有哪些可用充值/提现方式（配置在财务侧管理） | **待确认**，可能通过缓存/配置中心解耦 |
| 钱包 → ser-blog | 所有写操作的审计日志上报 | **弱依赖**，异步不阻塞主流程 |
| 钱包 → ser-s3 | 币种图标上传 | **弱依赖**，仅B端配置场景 |
| 钱包 → ser-buser | B端列表展示操作人名称 | **弱依赖**，仅影响展示 |
