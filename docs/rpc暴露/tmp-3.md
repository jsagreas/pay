# ser-wallet RPC 对外暴露接口分析（供其他模块调用）

> 分析时间：2026-02-28
> 分析范围：仅聚焦 ser-wallet 需要暴露给**工程内其他模块**的 RPC 接口（不含 C端/B端 HTTP 接口）
> 分析依据：需求文档（6文件夹 ~155张图片）+ 产品原型 + 接口评估 + 依赖关系 + 链路关系 + IDL设计 + 模块参考A/B + 工程现状（22个已注册 RPC Client 的通信模式）
> 分析角色：后端开发，主责钱包+币种配置模块
> 核心问题：**ser-wallet 需要对外暴露哪些 RPC 接口？谁来调？什么时候调？为什么必须由我们暴露？**

---

## 零、结论速览

```
ser-wallet 需要对外暴露的 RPC 接口：

┌──────────────────────────────────────────────────────────────────────────────┐
│  分类          │ 接口数 │ 调用方               │ 核心关注点                    │
├──────────────────────────────────────────────────────────────────────────────┤
│  余额变更类    │   5    │ 财务模块             │ 幂等+事务+状态机              │
│  人工修正类    │   3    │ 财务模块             │ 幂等+自动建钱包+级联扣减      │
│  余额查询类    │   2    │ 财务/投注/直播/活动   │ 高频读+缓存                   │
│  返奖入账类    │   1    │ 活动/任务模块        │ 幂等+自动建钱包               │
│  币种数据类    │   2    │ 财务/其他模块        │ 缓存+状态过滤                 │
├──────────────────────────────────────────────────────────────────────────────┤
│  合计          │  13    │                      │                               │
│  其中核心必须  │  10    │                      │ 没有就无法联调                │
│  其中应该有    │   3    │                      │ 不做不影响核心链路，但高概率需要│
└──────────────────────────────────────────────────────────────────────────────┘

暂不暴露（可选/预留，当前版本不写入 IDL）：4 个
```

---

## 一、为什么需要单独分析 RPC 暴露接口

### 1.1 RPC 接口的特殊性

C端/B端接口走 HTTP 网关（gate-font / gate-back），只被前端调用，改动成本仅限前后端协商。

RPC 对外接口走 Kitex RPC + ETCD 服务发现，被**工程内多个服务**调用。一旦暴露，调用方会基于 IDL 生成客户端代码、编写集成逻辑。**改动 RPC 接口 = 所有调用方同步改动**，成本随接入方数量放大。

所以 RPC 暴露接口必须：
- **想清楚再暴露**：接口语义、参数结构、返回值要足够稳定
- **幂等性设计**：其他模块可能重试、可能网络抖动重发
- **通用性设计**：一个接口可能被多种场景调用（如 GetBalance 同时被财务、投注、直播调用）
- **最小暴露原则**：不暴露内部实现细节，只暴露必要的能力

### 1.2 工程中的 RPC 通信模式（实证）

基于对工程内 22 个已注册 RPC Client 的分析，当前项目的跨模块 RPC 通信有以下特征：

| 特征 | 实际做法 | 证据 |
|------|---------|------|
| 注册方式 | `rpc_client.go` 中 `sync.Once` 单例模式 | 22 个 Client 全部如此 |
| 调用方式 | `rpc.XxxClient().Method(ctx, req)` | ser-item/ser-app/ser-bcom 等均如此 |
| IDL 组织 | 对外 RPC 类型放 `*_rpc.thrift` 独立管理 | user_rpc.thrift、live_rpc.thrift |
| 服务发现 | ETCD + 3秒超时 + TTHeader 链路追踪 | `init.go` InitRpcClientParams |
| 审计日志 | 几乎所有 B端写操作都调 `rpc.BLogClient().AddActLog()` | ser-item/ser-buser/ser-bcom 等 |

**ser-wallet 要做的**：在 `rpc_client.go` 注册 `WalletClient()`，在 `namesp.go` 注册 `EtcdWalletService`，其他模块即可通过 `rpc.WalletClient().CreditWallet(ctx, req)` 调用。

---

## 二、核心必须暴露的 RPC 接口（10 个）

> 判定标准：需求文档明确存在该业务场景，且该场景中**必须由其他模块调用钱包来操作余额**，不暴露此接口则对应业务流程无法跑通。

### R1 CreditWallet — 充值成功上账

```
调用方：    财务模块（充值回调处理后）
触发时机：  三方支付通道回调→财务模块确认支付成功→调此接口给用户上账
调用方式：  RPC 异步（不在 C端用户实时链路上，是后台异步处理）
```

**需求依据：**

- 钱包需求 5.2 节（USDT 充值）："达到确认数后自动上账"
- 钱包需求 5.3 节（银行转账充值）："支付状态回传平台 → 平台更新订单状态"
- 财务需求 2.3 节充值流程三泳道图：通道回传支付结果 → 服务端更新状态
- 产品原型 7-2：充值进行中→充值成功弹窗

**为什么必须是 RPC 而不是钱包自己处理：**

充值的实际支付回调是三方通道回传给财务模块的（因为通道配置、代收订单都在财务模块管理），财务模块处理完回调后需要通知钱包"这笔充值成功了，给用户上账"。钱包不直接对接三方通道。

**接口职责：**

```
输入：  userId, currencyCode, amount, orderNo, creditType(充值/补单/返奖等), remark
输出：  是否成功, 幂等标记(是否重复请求), 变更后余额

内部逻辑：
  1. 幂等校验（orderNo 是否已处理）
  2. 校验币种存在且启用
  3. 用户无该币种钱包时→自动创建
  4. 中心钱包可用余额 += amount（SQL 原子 UPDATE）
  5. 写入 wallet_flow 流水记录（类型：充值入账，关联 orderNo）
```

**设计要点：**
- **幂等**：财务模块可能因网络超时重试，同一 orderNo 只能上账一次
- **自动建钱包**：用户首次充某币种时可能还没有钱包记录
- **原子性**：余额变更+流水写入在同一事务内

---

### R2 DebitWallet — 消费/投注扣款

```
调用方：    投注模块 / 财务模块（视具体场景）
触发时机：  用户下注时扣减余额 / 其他消费扣款场景
调用方式：  RPC 同步（在用户操作实时链路上，需要快速响应）
```

**需求依据：**

- 钱包需求 4.3 节余额使用规则："优先扣除中心钱包余额，中心余额不足时从奖励钱包扣除"
- 钱包需求 4.2 节资金流向图：投注/游戏下注→从钱包扣款
- 核心业务链路（项目背景 know:项目背景.md）："充值→入账→钱包入账→投注/游戏下注→锁单→..."

**为什么是通用扣款而不是针对投注的专用接口：**

扣款逻辑本质相同——从用户钱包扣减指定金额。投注扣款、消费扣款、礼物扣款的区别仅在于 flowType（流水类型），核心的余额变更逻辑一致。提供通用扣款接口，通过 flowType 参数区分场景，避免为每种场景单独开接口。

**接口职责：**

```
输入：  userId, currencyCode, amount, orderNo, flowType(投注/消费/礼物等)
输出：  是否成功, 各钱包实际扣款明细(中心扣了多少/奖励扣了多少)

内部逻辑：
  1. 幂等校验
  2. 查询中心钱包余额 + 奖励钱包余额
  3. 校验总余额 >= amount
     → 不足：返回余额不足错误
  4. 计算扣款分配：
     → centerDebit = min(centerBalance, amount)
     → rewardDebit = amount - centerDebit
  5. 事务内执行：
     → 中心钱包余额 -= centerDebit（SQL 原子 UPDATE WHERE balance >= debit）
     → 奖励钱包余额 -= rewardDebit（如有）
  6. 写入 wallet_flow（记录各钱包扣款明细，供后续返奖比例计算）
```

**设计要点：**
- **返回扣款明细**：投注场景需要知道中心/奖励各扣了多少，因为返奖时需要按比例返还（需求 4.3 混合投注返奖规则）
- **双钱包扣减事务**：中心+奖励的扣减必须在同一事务内
- **SQL 乐观锁**：UPDATE ... SET balance = balance - ? WHERE balance >= ?，防止并发透支

---

### R3 FreezeBalance — 提现冻结

```
调用方：    ser-wallet 内部（CreateWithdrawOrder 流程中）/ 财务模块
触发时机：  用户提交提现申请时，冻结中心钱包对应金额
调用方式：  内部调用为主，也暴露 RPC 供财务模块在特殊场景调用
```

**需求依据：**

- 钱包需求 7.1 节提现流程：用户提交提现申请→系统冻结对应金额
- 财务需求 2.4 节提现流程三泳道图：申请提现→冻结金额→进入审核
- 产品原型 10-5：提现进行中弹窗（用户能看到冻结状态）
- 模块参考A分析：p9 wallet-service 的 freezeCoin 三步法（freeze / subFreeze / rollback），验证了冻结作为独立操作的必要性

**接口职责：**

```
输入：  userId, currencyCode, amount, orderNo
输出：  是否成功, 冻结流水号

内部逻辑：
  1. 幂等校验
  2. 校验中心钱包可用余额 >= amount
  3. 事务内执行：
     → 中心钱包可用余额 -= amount
     → 中心钱包冻结余额 += amount
  4. 写入 wallet_freeze_record（状态：冻结中）
  5. 写入 wallet_flow（类型：提现冻结）
```

**状态机前置**：此接口执行后，后续**必须且只能**走 R4（解冻退回）或 R5（确认扣除）其一。

---

### R4 UnfreezeBalance — 提现驳回/取消解冻

```
调用方：    财务模块（风控审核驳回 / 财务审核驳回 / 出款失败时）
触发时机：  提现订单在审核流程中被驳回，需要将冻结金额退还给用户
调用方式：  RPC 异步（后台审核操作触发）
```

**需求依据：**

- 财务需求 2.4 节提现流程：订单状态包含"风控驳回""财务驳回"
- 产品原型"提现记录-3"：审核弹窗展示通过/驳回两态
- 钱包需求 7.1 节提现状态说明：驳回→冻结金额退回可用余额

**接口职责：**

```
输入：  userId, currencyCode, amount, orderNo
输出：  是否成功

内部逻辑：
  1. 校验 orderNo 对应的冻结记录存在且状态为"冻结中"
  2. 事务内执行：
     → 中心钱包冻结余额 -= amount
     → 中心钱包可用余额 += amount（退回）
  3. 更新 wallet_freeze_record 状态：冻结中 → 已解冻
  4. 写入 wallet_flow（类型：提现退回）
```

**与 R5 互斥**：同一笔冻结记录，ConfirmDebit 和 UnfreezeBalance 只能执行其一。

---

### R5 ConfirmDebit — 提现出款成功确认扣除

```
调用方：    财务模块（出款成功后）
触发时机：  提现订单经过风控审核→财务审核→出款操作后，三方通道确认出款成功
调用方式：  RPC 异步（后台出款完成触发）
```

**需求依据：**

- 财务需求 2.4 节提现流程：出款成功→更新订单状态
- 资金链路（项目背景）："...投注/游戏下注→锁单→赛果→结算→返奖→账本→对账→风控→合规审计"中，"锁单"即冻结，"结算"即确认扣除

**接口职责：**

```
输入：  userId, currencyCode, amount, orderNo
输出：  是否成功

内部逻辑：
  1. 校验 orderNo 对应的冻结记录存在且状态为"冻结中"
  2. 事务内执行：
     → 中心钱包冻结余额 -= amount（正式扣除，资金永久离开用户钱包）
  3. 更新 wallet_freeze_record 状态：冻结中 → 已确认扣除
  4. 写入 wallet_flow（类型：提现扣款）
```

**三件套小结**：FreezeBalance(R3) → ConfirmDebit(R5) 或 UnfreezeBalance(R4) 构成**冻结状态机**：

```
           FreezeBalance(R3)
                 │
            ┌────▼────┐
            │  冻结中   │
            └────┬────┘
           ┌─────┴─────┐
    ConfirmDebit(R5)  UnfreezeBalance(R4)
           │                  │
     ┌─────▼─────┐    ┌──────▼──────┐
     │ 已确认扣除  │    │  已解冻退回  │
     │（资金离开） │    │（余额恢复）  │
     └───────────┘    └─────────────┘

约束：同一 orderNo，R5 和 R4 只能执行其一，且必须在 R3 之后
```

---

### R6 ManualCredit — 人工加款

```
调用方：    财务模块（人工加款审核通过后）
触发时机：  B端操作员创建加款单→另一名操作员审核通过→调此接口给用户加款
调用方式：  RPC 异步（B端审核操作触发）
```

**需求依据：**

- 财务需求 2.52 节"人工加款"：加款类型包括运营补偿/运营奖励/财务修正/测试加款
- "可同时向中心钱包/奖励钱包/代理钱包/主播钱包加款"
- "如用户下没有该币种钱包，系统需自动创建"
- 产品原型"人工修正-5至7"：四个钱包分别加款的界面

**接口职责：**

```
输入：  userId, currencyCode, walletType(center/reward/anchor/agent),
        amount, orderNo, creditType(补偿/奖励/修正/测试), operatorId, remark
输出：  是否成功, 幂等标记

内部逻辑：
  1. 幂等校验（A+ 前缀 orderNo）
  2. 用户无该币种+该类型钱包时→自动创建
  3. 指定 walletType 余额 += amount
  4. 写入 wallet_flow（类型：人工加款，记录操作人+加款类型+备注）
```

**设计要点：**
- **walletType 参数**：财务模块一次操作可能需要向多个钱包类型分别加款（界面上四个钱包各自输入金额），每个钱包类型调用一次本接口
- **自动建钱包**：需求明确"如用户下没有该币种钱包，系统需自动创建"

---

### R7 ManualDebit — 人工减款

```
调用方：    财务模块（人工减款审核通过后）
触发时机：  B端操作员创建减款单→另一名操作员审核通过→调此接口从用户钱包扣款
调用方式：  RPC 异步（B端审核操作触发）
```

**需求依据：**

- 财务需求 2.53 节"人工减款"：减款类型包括异常追回/风控处罚/财务冲正/费用调整
- **核心规则**："当用户钱包余额不足时，从最高金额的钱包开始依次扣减"
- "实际减款金额可能小于设定的减款金额"
- 产品原型"人工修正-8至10"：减款流程

**接口职责：**

```
输入：  userId, currencyCode, amount, orderNo, debitType(追回/处罚/冲正/调整), operatorId, remark
输出：  是否成功, 实际减款金额, 各钱包扣减明细列表

内部逻辑：
  1. 幂等校验（M+ 前缀 orderNo）
  2. 查询用户该币种下所有钱包类型的余额
  3. 按余额从高到低排序，级联扣减：
     → 例如：中心800 + 奖励200 + 主播100 = 总计1100
     → 需减1000：扣中心800 → 扣奖励200 → 合计1000
     → 如需减1500：扣中心800 + 奖励200 + 主播100 = 实际减1100 < 1500
  4. 事务内执行各钱包扣减
  5. 写入 wallet_flow（M+ orderNo，记录各钱包实际扣减明细）
```

**设计要点：**
- **级联扣减**：这是人工减款独有的逻辑，普通 DebitWallet 只在中心+奖励间分配
- **实际金额可能小于设定金额**：需要在返回值中明确返回实际扣了多少
- **跨钱包类型事务**：多个钱包的扣减必须在同一事务内

---

### R8 SupplementCredit — 充值补单

```
调用方：    财务模块（充值补单审核通过后）
触发时机：  充值金额有误需要补差→B端创建补单→双人审核通过→调此接口补差额
调用方式：  RPC 异步（B端审核操作触发）
```

**需求依据：**

- 财务需求 2.51 节"充值补单"：审核通过即"修正金额并入流水"
- 产品原型"人工修正-1至4"：补单完整流程（输入原充值订单号→查询→填写补单信息→审核）
- 补单包含两部分：补充金额（→中心钱包）+ 赠送金额（→奖励钱包，需稽核流水）

**接口职责：**

```
输入：  userId, currencyCode, supplementAmount, bonusAmount, auditMultiple,
        originalOrderNo, orderNo, operatorId, remark
输出：  是否成功, 幂等标记

内部逻辑：
  1. 幂等校验（B+ 前缀 orderNo）
  2. 事务内执行：
     → 中心钱包余额 += supplementAmount
     → 奖励钱包余额 += bonusAmount（如有，需关联稽核倍数）
  3. 写入 wallet_flow（B+ orderNo，记录原订单关联+补正信息）
```

---

### R9 GetBalance — 余额查询

```
调用方：    财务模块 / 投注模块 / 直播模块 / 活动模块 / 统计模块 ...
触发时机：  各种需要校验或展示用户余额的场景
调用方式：  RPC 同步（被多个模块高频调用）
```

**需求依据：**

- 投注场景：下注前需校验余额是否够（核心业务链路"钱包入账→投注/游戏下注"）
- 直播场景：送礼前需校验余额
- 财务场景：人工修正前需确认当前余额
- 活动场景：判断用户是否有足够余额参与活动

**为什么这是必须暴露的：**

工程内的 22 个已注册 RPC Client 中，最高频的跨模块调用就是**查询类接口**（如 `rpc.UserClient().GetUserInfoExpend()`）。ser-wallet 作为资产管理核心，余额数据被多个上层业务模块依赖，不暴露此接口则各业务模块无法进行余额前置校验。

**接口职责：**

```
输入：  userId, currencyCode, walletType(可选，不传=返回所有类型)
输出：  余额明细列表：[{walletType, availableBalance, frozenBalance}]

内部逻辑：
  1. 查询指定用户指定币种的钱包记录
  2. 如指定 walletType：返回该类型余额
  3. 如不指定：返回所有钱包类型的余额明细
     → 中心钱包：可用余额 + 冻结余额
     → 奖励钱包：可用余额
     → 主播钱包：可用余额（如有）
     → 代理钱包：可用余额（如有，预留）
```

**设计要点：**
- **纯读操作**，无副作用，可做缓存优化
- **最高频的 RPC 接口**，性能要求高
- 返回值包含**冻结余额**（中心钱包独有），调用方可据此判断用户实际可用资金

---

### R10 RewardCredit — 活动/任务返奖入账

```
调用方：    活动模块 / 任务模块
触发时机：  用户完成活动条件/任务目标后，发放奖励到用户奖励钱包
调用方式：  RPC 异步（活动系统结算后触发）
```

**需求依据：**

- 钱包需求 4.2 节资金流向图："活动/任务返奖→奖励钱包"
- 奖励钱包的定义："存放平台活动奖励、首充奖励、任务奖励等，需完成稽核流水后可转入中心钱包"
- 产品原型 2-1 钱包结构：奖励钱包作为独立层级存在

**为什么单独一个接口而不复用 CreditWallet：**

CreditWallet(R1) 的默认目标是中心钱包，且语义是"充值入账"。返奖入账有两个区别：
1. 目标钱包是**奖励钱包**（不是中心钱包）
2. 需要关联**稽核流水倍数**要求（奖励金额需完成 X 倍投注才能转入中心钱包）

虽然技术上可以用 CreditWallet + walletType=reward 实现，但从语义清晰度和调用方便利性考虑，独立接口更合理——活动模块调 `RewardCredit` 比调 `CreditWallet(walletType=reward, auditMultiple=X)` 更直觉。

**接口职责：**

```
输入：  userId, currencyCode, amount, orderNo, activityId, activityType, auditMultiple
输出：  是否成功, 幂等标记

内部逻辑：
  1. 幂等校验
  2. 用户无该币种奖励钱包→自动创建
  3. 奖励钱包余额 += amount
  4. 写入 wallet_flow（类型：活动返奖，关联 activityId + auditMultiple）
```

---

## 三、应该暴露的 RPC 接口（3 个）

> 判定标准：需求文档未直接提及该接口名称，但根据业务逻辑推导和工程规范，高概率在联调阶段必需。不做不影响核心链路启动，但会在业务对接时暴露缺失。

### R11 BatchGetBalance — 批量余额查询

```
调用方：    统计/报表模块、活动模块（批量校验）、B端财务模块（用户列表展示余额）
触发时机：  需要一次性获取多个用户余额的场景
调用方式：  RPC 同步
```

**推导依据：**

- 工程规范参照：ser-user 暴露了 `BatchGetUserInfo`（支持按 userId 列表批量查询），ser-item 暴露了 `BatchGetItemsByIds`——**批量查询是工程内的标准 RPC 模式**
- 业务场景：活动模块发放奖励时可能需要批量校验用户余额、财务报表需要汇总统计

**接口职责：**

```
输入：  userIds(列表), currencyCode
输出：  map<userId, BalanceInfo> 或 list<UserBalanceItem>

内部逻辑：
  1. 按 userIds 批量查询钱包表
  2. 组装结果（用户无钱包记录的返回零值）
```

**设计要点：**
- 限制单次查询数量上限（如 100），防止滥用
- 参照 ser-user 的 BatchGetUserInfo：`user_rpc.BatchGetUserInfoReq { userIds: list<i64> }`

---

### R12 GetCurrencyConfig — 币种配置查询（RPC 版）

```
调用方：    财务模块 / 其他需要币种信息的模块
触发时机：  财务模块配置充值/提现方式时需要获取可用币种列表和汇率信息
调用方式：  RPC 同步
```

**推导依据：**

- 依赖关系分析（依赖关系/tmp-3.md §8.1）："币种配置对钱包是纯供给关系"——但这个供给不仅限于钱包自己的 C端/B端接口，**财务模块在配置充值方式时也需要知道有哪些币种可用**
- 实际场景：财务模块的"充值方式新增"需要选择关联的币种（产品原型"支付配置-1"展示了按币种分组的充值方式列表），这个币种数据从哪来？从 ser-wallet 的币种配置来

**接口职责：**

```
输入：  status(可选: 全部/启用/禁用), currencyType(可选: 法币/加密)
输出：  币种列表: [{currencyCode, currencyName, icon, symbol, decimalPlaces,
         thousandSeparator, status, platformRate, depositRate, withdrawRate}]

内部逻辑：
  1. 查询 currency_config 表，按条件过滤
  2. 如需汇率信息，从缓存/库中获取实时/平台/入款/出款汇率
```

**设计要点：**
- 该接口的数据变更频率低（币种配置基本稳定），调用方可做本地缓存
- 和 B端的 `PageCurrency` 区别：B端接口含分页+偏差值计算等管理信息，RPC 版只返回核心配置数据

---

### R13 GetExchangeRateRpc — 汇率查询（RPC 版）

```
调用方：    财务模块 / 其他需要汇率换算的模块
触发时机：  财务模块处理充值/提现订单时需要汇率换算
调用方式：  RPC 同步
```

**推导依据：**

- 依赖关系分析："币种配置→钱包→财务"的汇率供给链中，财务模块处理 BSB 相关的充值/提现订单时，需要获取当前的入款汇率/出款汇率进行金额换算
- 充值场景：BSB 充值时用户实际支付法币，财务模块在创建通道订单时需要知道入款汇率
- 提现场景：BSB 提现时用户提取法币，财务模块在计算出款金额时需要出款汇率

**接口职责：**

```
输入：  currencyCode, rateType(入款/出款/平台)
输出：  rate(汇率值), baseCurrency, updateTime

内部逻辑：
  1. 从缓存/库获取指定币种指定类型的汇率
  2. 返回汇率值+更新时间
```

**和 C端 GetExchangeRate(C7) 的区别**：
- C7 面向用户展示，包含赠送比例、格式化规则等展示信息
- R13 面向内部服务，只返回纯数据（汇率值+精度），用于计算

---

## 四、暂不暴露的接口（可选/预留，4 个）

> 判定标准：需求文档标注"预留"、当前版本不确定是否实现、或对应调用方模块尚未开发。后续按需通过 IDL optional 追加即可，不影响已有接口。

| 序号 | 接口名 | 调用方 | 暂不暴露的原因 | 何时考虑加入 |
|------|--------|--------|---------------|-------------|
| P1 | BetDebit | 投注模块 | 投注模块尚未开发，投注扣款的特殊需求（中心/奖励比例记录用于返奖计算）需与投注模块联调时确认 | 投注模块进入联调阶段 |
| P2 | BetReturn | 投注模块 | 同上，投注返奖规则（按投注时比例返还到中心/奖励）需与投注模块确认 | 投注模块进入联调阶段 |
| P3 | AnchorIncome | 直播/社交模块 | 直播模块尚未开发，主播收入入账的钱包类型（主播钱包）和规则需确认 | 直播模块进入联调阶段 |
| P4 | AgentIncome / VenueTransfer | 代理/场馆模块 | 需求文档标注"预留"，当前版本明确不实现 | 产品明确要求后加入 |

**暂不暴露 ≠ 不考虑**：

这些接口虽然暂不写入 IDL，但在代码架构设计时已经考虑了扩展性：
- `CreditWallet(R1)` 的 `creditType` 参数和 `walletType` 参数支持传入不同类型，BetReturn/AnchorIncome 本质上可以通过 CreditWallet + 特定 type 实现
- `DebitWallet(R2)` 返回的扣款明细已包含中心/奖励分配，BetDebit 的核心需求已被覆盖
- 如果投注/直播模块联调时发现通用接口不能满足特殊需求，再追加专用接口

---

## 五、RPC 接口与调用方的关系矩阵

### 5.1 谁调用了谁

```
                    R1    R2    R3    R4    R5    R6    R7    R8    R9    R10   R11   R12   R13
                    上账  扣款  冻结  解冻  确认  加款  减款  补单  查余额 返奖  批量  币种  汇率
                    ────  ────  ────  ────  ────  ────  ────  ────  ────  ────  ────  ────  ────
财务模块             ●     ○     △     ●     ●     ●     ●     ●     ●           ○     ●     ●
投注模块                   ●                                         ●
直播/社交模块                                                        ●
活动/任务模块                                                        ●     ●
统计/报表模块                                                        ●           ●

● = 核心调用（明确需要）
○ = 可能调用（视具体设计）
△ = 钱包内部调用为主，也暴露 RPC
```

### 5.2 调用频率与触发性质

| 接口 | 调用频率 | 触发性质 | 核心关注点 |
|------|---------|---------|-----------|
| R1 CreditWallet | 低频 | 异步（三方回调后） | **幂等性**、数据最终一致 |
| R2 DebitWallet | 中频 | 同步（用户操作时） | **响应速度**、余额一致性 |
| R3 FreezeBalance | 低频 | 同步（提现申请时） | 事务原子性 |
| R4 UnfreezeBalance | 低频 | 异步（审核驳回后） | 幂等性、状态机一致 |
| R5 ConfirmDebit | 低频 | 异步（出款成功后） | 幂等性、状态机一致 |
| R6 ManualCredit | 极低频 | 异步（B端审核后） | 幂等性、自动建钱包 |
| R7 ManualDebit | 极低频 | 异步（B端审核后） | 级联扣减、实际金额返回 |
| R8 SupplementCredit | 极低频 | 异步（B端审核后） | 幂等性 |
| R9 GetBalance | **高频** | 同步（多场景查询） | **性能**、缓存策略 |
| R10 RewardCredit | 中频 | 异步（活动结算后） | 幂等性、稽核流水关联 |
| R11 BatchGetBalance | 中频 | 同步（批量查询） | 批量性能、上限控制 |
| R12 GetCurrencyConfig | 低频 | 同步（配置读取） | 缓存、变更频率低 |
| R13 GetExchangeRateRpc | 中频 | 同步（订单计算时） | 缓存、汇率实时性 |

---

## 六、所有 RPC 接口的共性设计要求

### 6.1 幂等性设计（所有写操作必须）

```
涉及接口：R1, R2, R3, R4, R5, R6, R7, R8, R10（共 9 个写操作）

幂等策略：基于 orderNo 唯一键

流程：
  1. 调用方传入全局唯一的 orderNo
  2. ser-wallet 在处理前查询 wallet_flow 表：该 orderNo 是否已存在
     → 已存在且状态=成功：直接返回成功（幂等响应），不重复操作
     → 已存在且状态=处理中：返回"处理中"状态
     → 不存在：正常处理
  3. 处理完成后 wallet_flow 记录关联 orderNo

依据：
  - 架构设计/tmp-3.md ADR#3：所有 RPC 变更 Req 均含 orderNo
  - IDL设计/tmp-3.md §5.7：wallet_rpc.thrift 所有 Req 都有 orderNo 字段
  - 模块参考B分析：tg-services 使用 ssOrderId 实现幂等防重入
```

### 6.2 统一返回结构

```
所有 RPC Resp 应包含：
  1. success: bool          — 是否处理成功
  2. idempotent: bool       — 是否为幂等重复请求（已处理过的相同 orderNo）
  3. errorCode: i32         — 错误码（0=成功）
  4. errorMsg: string       — 错误描述

写操作额外返回：
  5. flowId: i64            — 流水记录 ID（供调用方关联）

查询操作返回对应的业务数据
```

### 6.3 SQL 原子操作（余额变更必须）

```
所有余额变更使用 SQL 级原子 UPDATE：
  UPDATE user_wallet
  SET available_balance = available_balance + ?
  WHERE user_id = ? AND currency_code = ? AND wallet_type = ?

扣款操作增加余额校验：
  UPDATE user_wallet
  SET available_balance = available_balance - ?
  WHERE user_id = ? AND currency_code = ? AND wallet_type = ?
    AND available_balance >= ?

依据：
  - 模块参考A分析：p9 wallet-service 的 SQL 级原子 UPDATE（WHERE amount >= 扣减额）
  - 架构设计/tmp-3.md："SQL 乐观锁方案"被评为可直接采纳的模式
```

### 6.4 分布式锁（高并发场景）

```
对同一用户的同一币种的并发写操作，使用 Redis 分布式锁：
  锁 Key：wallet:lock:{userId}:{currencyCode}
  等待时间：参照 p9 模式 20s
  持有时间：参照 p9 模式 30s

与 SQL 原子操作形成双保险：
  分布式锁 = 第一道防线（减少并发冲突）
  SQL WHERE 条件 = 第二道防线（兜底保证数据一致）

依据：
  - 模块参考A分析：Redisson 分布式锁 + SQL 原子 UPDATE 双保险
  - 评定结论：✅ 可直接采纳
```

---

## 七、RPC 接口的开发优先级

基于依赖关系分析中的分层结论：

```
第一优先级 — 基础能力（其他模块联调的前提）
  ┌─────────────────────────────────────────────────┐
  │  R9  GetBalance         — 被所有模块依赖的基础查询 │
  │  R1  CreditWallet       — 充值链路的终点            │
  │  R2  DebitWallet        — 投注/消费链路的起点        │
  │  R3  FreezeBalance      — 提现链路的起点             │
  │  R4  UnfreezeBalance    — 提现链路的回退路径          │
  │  R5  ConfirmDebit       — 提现链路的确认路径          │
  │  R12 GetCurrencyConfig  — 财务模块配置依赖            │
  │  R13 GetExchangeRateRpc — 财务模块汇率依赖            │
  │                                                     │
  │  这 8 个接口优先交付 IDL + 空壳实现                   │
  │  让财务模块可以尽早基于类型生成客户端代码              │
  └─────────────────────────────────────────────────┘

第二优先级 — 人工修正 + 返奖（可在财务联调中期交付）
  ┌─────────────────────────────────────────────────┐
  │  R6  ManualCredit       — 人工加款                 │
  │  R7  ManualDebit        — 人工减款                 │
  │  R8  SupplementCredit   — 充值补单                 │
  │  R10 RewardCredit       — 活动返奖                 │
  │  R11 BatchGetBalance    — 批量余额查询              │
  └─────────────────────────────────────────────────┘

后续按需 — 投注/直播/代理（对应模块联调时交付）
  ┌─────────────────────────────────────────────────┐
  │  P1  BetDebit           — 投注专用扣款             │
  │  P2  BetReturn          — 投注专用返奖             │
  │  P3  AnchorIncome       — 主播收入                 │
  │  P4  AgentIncome / ...  — 预留                     │
  └─────────────────────────────────────────────────┘
```

---

## 八、RPC 接口与 IDL 文件的对应关系

所有 RPC 对外接口的类型定义统一放在 `wallet_rpc.thrift`：

```thrift
// common/idl/ser-wallet/wallet_rpc.thrift
namespace go ser_wallet

// ==========================================
// 余额变更类
// ==========================================

struct CreditWalletReq {
    1: required i64 userId
    2: required string currencyCode
    3: required i64 amount           // 最小单位整数
    4: required string orderNo       // 幂等键
    5: required i32 creditType       // 充值=1/补单=2/返奖=3/...
    6: optional i32 walletType       // 默认中心钱包
    7: optional string remark
}
struct CreditWalletResp { ... }

struct DebitWalletReq {
    1: required i64 userId
    2: required string currencyCode
    3: required i64 amount
    4: required string orderNo
    5: required i32 flowType         // 投注=1/消费=2/礼物=3/...
}
struct DebitWalletResp {
    // ...
    // 返回各钱包实际扣款明细：[{walletType, debitAmount}]
}

struct FreezeBalanceReq { ... }
struct FreezeBalanceResp { ... }
struct UnfreezeBalanceReq { ... }
struct UnfreezeBalanceResp { ... }
struct ConfirmDebitReq { ... }
struct ConfirmDebitResp { ... }

// ==========================================
// 人工修正类
// ==========================================

struct ManualCreditReq {
    // ... + walletType + operatorId + creditType(补偿/奖励/修正/测试)
}
struct ManualCreditResp { ... }
struct ManualDebitReq { ... }
struct ManualDebitResp {
    // ... + actualAmount(实际减款) + debitDetails(各钱包明细)
}
struct SupplementCreditReq {
    // ... + supplementAmount + bonusAmount + auditMultiple + originalOrderNo
}
struct SupplementCreditResp { ... }

// ==========================================
// 查询类
// ==========================================

struct GetBalanceReq {
    1: required i64 userId
    2: required string currencyCode
    3: optional i32 walletType       // 不传=返回所有类型
}
struct GetBalanceResp {
    1: list<WalletBalanceItem> balances
}
struct WalletBalanceItem {
    1: i32 walletType
    2: i64 availableBalance
    3: i64 frozenBalance
}

struct BatchGetBalanceReq {
    1: required list<i64> userIds    // 上限 100
    2: required string currencyCode
}
struct BatchGetBalanceResp {
    1: list<UserBalanceItem> items
}

struct GetCurrencyConfigReq {
    1: optional i32 status           // 全部/启用/禁用
    2: optional i32 currencyType     // 法币/加密
}
struct GetCurrencyConfigResp {
    1: list<CurrencyConfigItem> currencies
}

struct GetExchangeRateRpcReq {
    1: required string currencyCode
    2: required i32 rateType         // 入款=1/出款=2/平台=3
}
struct GetExchangeRateRpcResp {
    1: i64 rate                      // 最小单位整数表示
    2: string baseCurrency
    3: i64 updateTime
}

// ==========================================
// 返奖类
// ==========================================

struct RewardCreditReq {
    1: required i64 userId
    2: required string currencyCode
    3: required i64 amount
    4: required string orderNo
    5: required i64 activityId
    6: optional i32 activityType
    7: optional i32 auditMultiple    // 稽核流水倍数
}
struct RewardCreditResp { ... }
```

在 `service.thrift` 的 RPC 分组中声明方法：

```thrift
service WalletService {
    // ... C端 / B端方法 ...

    // ==========================================
    // RPC 对外提供（供财务/投注/直播/活动等服务调用）
    // ==========================================

    // 余额变更
    wallet_rpc.CreditWalletResp CreditWallet(1: wallet_rpc.CreditWalletReq req)
    wallet_rpc.DebitWalletResp DebitWallet(1: wallet_rpc.DebitWalletReq req)
    wallet_rpc.FreezeBalanceResp FreezeBalance(1: wallet_rpc.FreezeBalanceReq req)
    wallet_rpc.UnfreezeBalanceResp UnfreezeBalance(1: wallet_rpc.UnfreezeBalanceReq req)
    wallet_rpc.ConfirmDebitResp ConfirmDebit(1: wallet_rpc.ConfirmDebitReq req)

    // 人工修正
    wallet_rpc.ManualCreditResp ManualCredit(1: wallet_rpc.ManualCreditReq req)
    wallet_rpc.ManualDebitResp ManualDebit(1: wallet_rpc.ManualDebitReq req)
    wallet_rpc.SupplementCreditResp SupplementCredit(1: wallet_rpc.SupplementCreditReq req)

    // 查询
    wallet_rpc.GetBalanceResp GetBalance(1: wallet_rpc.GetBalanceReq req)
    wallet_rpc.BatchGetBalanceResp BatchGetBalance(1: wallet_rpc.BatchGetBalanceReq req)
    wallet_rpc.GetCurrencyConfigResp GetCurrencyConfig(1: wallet_rpc.GetCurrencyConfigReq req)
    wallet_rpc.GetExchangeRateRpcResp GetExchangeRateRpc(1: wallet_rpc.GetExchangeRateRpcReq req)

    // 返奖
    wallet_rpc.RewardCreditResp RewardCredit(1: wallet_rpc.RewardCreditReq req)
}
```

---

## 九、与已有分析文档的一致性验证

| 验证项 | 参考文档 | 本文档结论 | 一致性 |
|--------|---------|-----------|--------|
| RPC 对外核心 10 个 | 接口评估/tmp-3.md §三 | R1-R10 完全对应 | ✅ |
| 冻结三件套状态机 | 依赖关系/tmp-3.md §3.2 | R3→R4/R5 互斥关系一致 | ✅ |
| 财务调钱包 6 条链路 | 依赖关系/tmp-3.md §8.3 | R1+R4+R5+R6+R7+R8 = 6条 | ✅ |
| wallet_rpc.thrift 独立管理 | IDL设计/tmp-3.md §3.2 | 所有 RPC 类型放 wallet_rpc.thrift | ✅ |
| 幂等键 orderNo | 架构设计/tmp-3.md §4.2 | 所有写 Req 含 orderNo | ✅ |
| SQL 原子 UPDATE | 模块参考A/tmp-3.md §五 | ✅ 采纳 WHERE balance >= ? | ✅ |
| 分布式锁双保险 | 模块参考A/tmp-3.md §五 | ✅ 采纳 Redis锁 + SQL乐观锁 | ✅ |
| 金额 i64 整数 | IDL设计/tmp-3.md §2.4 | 所有金额字段 i64 | ✅ |
| 扣款返回明细 | 链路关系/tmp-3.md §R2 | DebitWalletResp 含各钱包扣款明细 | ✅ |
| 级联扣减逻辑 | 链路关系/tmp-3.md §R7 | ManualDebit 按余额从高到低扣减 | ✅ |
| 活动返奖→奖励钱包 | 需求理解/tmp-3.md §2.2 | RewardCredit 目标=奖励钱包 | ✅ |

**新增 3 个（相比 IDL设计文档）**：

| 新增接口 | 新增理由 | IDL设计时为何未包含 |
|---------|---------|-------------------|
| R11 BatchGetBalance | 工程规范中批量查询是标准模式（user_rpc / item_publish 均有） | IDL设计时归入"应该有"暂未写入 |
| R12 GetCurrencyConfig | 财务模块配置充值/提现方式时需要币种数据 | IDL设计时未单独考虑币种RPC版 |
| R13 GetExchangeRateRpc | 财务模块处理订单时需要汇率数据 | IDL设计时未单独考虑汇率RPC版 |

这 3 个接口在之前的 IDL 设计中归入"可选/后续"，本次分析从**联调实际需求**出发将其提升为"应该有"。

---

## 十、需要与财务模块负责人提前对齐的事项

### 10.1 接口契约对齐（最高优先级）

```
我们暴露，财务调用（需约定 Req/Resp 细节）：
  ① CreditWallet    — orderNo 格式？creditType 枚举值？
  ② ConfirmDebit    — 是否需要额外传实际出款金额（可能因手续费和冻结金额不一致）？
  ③ UnfreezeBalance — 是否需要传退回原因（风控驳回/财务驳回/出款失败的区分）？
  ④ ManualCredit    — operatorId 如何传递？加款类型枚举值对齐？
  ⑤ ManualDebit     — 返回的实际减款金额，财务侧如何使用？
  ⑥ SupplementCredit — 原订单号的校验逻辑在谁那边？
```

### 10.2 时序约定

```
充值链路的关键时序问题：
  Q: 充值订单是谁创建的？
     → 方案A：钱包创建本地订单 + 调财务创建通道订单（两份数据）
     → 方案B：财务统一管理订单，钱包只管上账
     → 影响：R1 CreditWallet 的 orderNo 来源

提现链路的关键时序问题：
  Q: 冻结是钱包自己在 CreateWithdrawOrder 中做，还是财务调 FreezeBalance 做？
     → 当前设计：钱包自己冻结，因为冻结是余额操作
     → 需确认：财务模块是否接受这个时序
```

### 10.3 错误处理约定

```
当 RPC 调用失败时的处理策略：
  ① 财务调 CreditWallet 超时 → 财务应重试（钱包有幂等保证）
  ② 财务调 UnfreezeBalance 失败 → 冻结金额卡住，需要补偿机制
  ③ 财务调 ManualDebit 返回实际金额<设定金额 → 财务侧如何展示？
```

---

## 十一、总结

### 一句话定位

> ser-wallet 对外暴露的 RPC 接口本质上是**资产操作能力的输出**——让工程内其他模块可以安全地操作用户钱包（查余额、加钱、扣钱、冻结、解冻），而不需要直接访问钱包的数据库。

### 数字总结

```
核心必须暴露：10 个（不做则联调无法进行）
  余额变更 5 个：CreditWallet / DebitWallet / FreezeBalance / UnfreezeBalance / ConfirmDebit
  人工修正 3 个：ManualCredit / ManualDebit / SupplementCredit
  查询 1 个：    GetBalance
  返奖 1 个：    RewardCredit

应该暴露：3 个（高概率需要）
  BatchGetBalance / GetCurrencyConfig / GetExchangeRateRpc

暂不暴露：4 个（可选/预留）
  BetDebit / BetReturn / AnchorIncome / AgentIncome+VenueTransfer

调用方统计：
  财务模块调我们：8 个接口（R1-R8, 余额变更全套 + 人工修正全套）
  投注模块调我们：2 个接口（R2 DebitWallet + R9 GetBalance）
  直播模块调我们：1 个接口（R9 GetBalance）
  活动模块调我们：2 个接口（R9 GetBalance + R10 RewardCredit）
  其他模块调我们：3 个接口（R9 + R11 + R12）
```

### 工程落地清单

```
① common/idl/ser-wallet/wallet_rpc.thrift   — 定义 13 个 RPC 接口的 Req/Resp
② common/idl/ser-wallet/service.thrift      — RPC 分组中声明 13 个方法
③ common/pkg/consts/namesp/namesp.go        — 新增 EtcdWalletService + Port
④ common/rpc/rpc_client.go                  — 新增 WalletClient() 单例
⑤ ser-wallet/handler.go                     — 13 个 RPC 方法的 handler 实现
⑥ ser-wallet/internal/ser/wallet_rpc_*.go   — RPC 业务逻辑层
```
