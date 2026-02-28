# ser-wallet IDL 设计方案

> 设计时间：2026-02-27
> 定位：IDL 先行的接口契约设计，是后续 kitex_gen 代码生成、handler 编写、网关路由注册的基础
> 设计依据：需求理解（result/tmp-3.md）+ 接口评估（接口评估/tmp-3.md）+ 流程思路（流程思路/tmp-3.md）+ 实现思路（实现思路/tmp-3.md）+ 现有 71 个 IDL 文件的工程规范
> 设计原则：对齐已有规范、屏蔽字段细节、方法分组清晰、文件拆分合理、保留迭代弹性
> 角色定位：我负责钱包+币种配置模块，配合财务模块联调

---

## 一、设计前提：为什么必须仔细考虑 IDL 设计

### 1.1 IDL 在工程中的地位

```
IDL 定义 → gen_kitex.sh 生成 kitex_gen → handler.go 骨架 → 业务代码填充
                                        → rpc_client.go 注册 → 其他服务可调用
                                        → gate 路由注册 → HTTP 可达
```

IDL 是一切的起点。文件怎么拆、方法怎么分、命名怎么定——这些决策一旦落地就会扩散到整个链路（kitex_gen、handler、router、调用方）。改 IDL 意味着重新生成代码、调用方同步修改，成本随接入方增加而放大。

### 1.2 ser-wallet 的特殊性

与已有服务对比，ser-wallet 有三个特殊之处：

| 特殊点 | 说明 | 对 IDL 设计的影响 |
|--------|------|------------------|
| **三端服务** | 同时服务 C端（App用户）、B端（管理员）、RPC（内部服务） | 需要清晰的方法分组和命名策略 |
| **接口数量大** | 预估 40-50 个方法（核心 ~36，应该有 ~10，可选 ~7） | 需要合理的文件拆分，单文件不能过于臃肿 |
| **跨模块调用方多** | 财务/投注/直播/活动等多个服务会调 RPC | RPC 接口的稳定性要求最高，需独立管理 |

对比已有服务规模：

| 服务 | IDL 文件数 | 方法数 | 类比参考 |
|------|-----------|--------|---------|
| ser-user | 13 | ~37 | 最接近 ser-wallet 的规模 |
| ser-live | 9 | ~32 | 多端（主播端/观众端/B端/RPC）模式可参考 |
| ser-app | 5 | ~25 | 中等规模，多域聚合模式 |
| **ser-wallet（预估）** | **8** | **~40-50** | 三端 + 多域，介于 ser-user 和 ser-live 之间 |

---

## 二、已有工程 IDL 规范提取（71 个文件的共性规则）

基于对 `common/idl/` 下 20 个服务目录、71 个 Thrift 文件的完整分析，提取以下确定性规范。ser-wallet 必须遵守，不可自创规则。

### 2.1 文件组织规范

| 规范项 | 已有工程规则 | 证据来源 |
|--------|------------|---------|
| 目录位置 | `common/idl/ser-{name}/` | 所有 20 个服务一致 |
| 入口文件 | `service.thrift`（每个服务必有且唯一） | 20/20 服务一致 |
| 入口文件职责 | 只声明 `service XxxService {}`，只 include 域文件，**不定义 struct** | ser-app/ser-user/ser-live 等一致 |
| 域文件 | 按业务域拆分，定义该域的所有 struct（实体+Req+Resp） | banner.thrift、users.thrift、live_room.thrift 等 |
| RPC 专用文件 | `*_rpc.thrift`，供其他服务调用的接口类型单独管理 | user_rpc.thrift、live_rpc.thrift |
| 公共类型 | `../common/enum.thrift` 跨服务共享 | ser-app 引用 common/enum.thrift |

### 2.2 命名规范

| 规范项 | 已有工程规则 | 示例 |
|--------|------------|------|
| namespace | `namespace go ser_{name}` | `namespace go ser_app`、`namespace go ser_user` |
| 服务名 | `{Name}Service`（PascalCase） | `AppService`、`UserService`、`LiveService` |
| 方法名 | `{Action}{Entity}`（PascalCase） | `CreateBanner`、`PageBanner`、`StatusBanner` |
| Req/Resp | `{Action}{Entity}Req` / `{Action}{Entity}Resp` | `CreateBannerReq`、`CreateBannerResp` |
| 实体 struct | `{Entity}Item`（B端）/ `{Entity}ItemC`（C端简化版） | `BannerItem`、`BannerItemC` |
| 文件名 | snake_case.thrift | `live_room.thrift`、`user_rpc.thrift` |
| 字段名 | camelCase | `bannerName`、`pagePosition`、`createAt` |

### 2.3 方法签名规范

```thrift
// 标准签名：返回类型在前，参数编号从 1 开始
domain.ActionEntityResp ActionEntity(1: domain.ActionEntityReq req)

// 无参方法：省略参数列表
users.UserDetailResp GetUserDetail()

// 返回类型引用域文件前缀
banner.CreateBannerResp CreateBanner(1: banner.CreateBannerReq req)
```

### 2.4 字段类型与注解规范

| 规范项 | 规则 | 示例 |
|--------|------|------|
| ID 类型 | 一律 `i64` | `1: i64 id`、`2: i64 userId` |
| 时间戳 | `i64`（毫秒时间戳） | `15: i64 createAt`、`18: i64 updateAt` |
| 金额 | `i64`（整数存储，最小单位） | 与架构设计一致：BIGINT 整数精度链 |
| 状态/类型 | `i32`（枚举值） | `7: i32 status`、`4: i32 jumpType` |
| 必填/可选 | Req 用 `required`/`optional`，Resp 和实体 struct 不标注 | `1: required string bannerName` |
| 字段编号 | 从 1 开始连续递增，不跳号 | 所有文件一致 |
| API 注解 | `(api.body = "field")` 用于 RPC 类型文件 | user_rpc.thrift 全部字段带注解 |
| 分页默认值 | `pageNo = 1`、`pageSize = 10`（可选） | user_rpc.thrift 的 SearchUserReq |

### 2.5 分页模式

```thrift
// 请求：
struct PageXxxReq {
    // ... 业务筛选字段 ...
    N: optional i32 pageNo;     // 页码，从1开始
    M: optional i32 pageSize;   // 每页条数
}

// 响应：
struct PageXxxResp {
    1: list<XxxItem> pageList;  // 业务数据（字段名 pageList 或 list）
    2: i64 total;               // 总记录数
    3: i32 pageNo;              // 当前页码
    4: i32 pageSize;            // 每页记录数
}
```

### 2.6 空响应模式

```thrift
// 无返回值的操作使用显式空 struct
struct DeleteBannerResp {}
struct StatusBannerResp {}
```

---

## 三、ser-wallet IDL 文件拆分方案

### 3.1 拆分策略推导

**核心问题**：8 个文件还是 6 个？service.thrift 单服务还是多服务？

**已有工程的答案**——全部 20 个服务都是 **单 service.thrift + 单 Service 声明**，没有任何一个服务拆成多个 Service。这个约束很明确。

文件拆分的关键考量：

```
拆分维度 ① ——按业务子模块
  ser-wallet 有四个子模块：币种配置、钱包核心、交易业务、记录查询
  → 每个子模块的类型定义放在独立文件中

拆分维度 ② ——按调用端
  C端和B端的实体 struct 不同（C端简化、B端详细）
  → 但已有工程的做法是放在同一个文件中（BannerItem vs BannerItemC 同在 banner.thrift）
  → 遵循已有规范，不按端拆分文件

拆分维度 ③ ——RPC 单独管理
  供外部服务调用的 RPC 接口，参照 user_rpc.thrift / live_rpc.thrift 独立一个文件
  → wallet_rpc.thrift
```

### 3.2 最终文件清单：8 个文件

```
common/idl/ser-wallet/
├── service.thrift           ← 入口：WalletService 所有方法声明（~40-50 方法）
├── wallet.thrift            ← 钱包核心类型（余额、钱包信息、流水）
├── currency.thrift          ← 币种配置类型（币种、汇率、日志）
├── deposit.thrift           ← 充值相关类型（订单、方式、奖金）
├── withdraw.thrift          ← 提现相关类型（订单、账户、方式）
├── exchange.thrift          ← 兑换相关类型（汇率、记录）
├── record.thrift            ← 记录查询类型（列表、详情）
└── wallet_rpc.thrift        ← RPC 对外类型（供财务/投注/直播/活动调用）
```

### 3.3 每个文件的职责与对应子模块

| 文件 | 对应子模块 | 职责 | 预估 struct 数 |
|------|-----------|------|---------------|
| service.thrift | — | 统一入口，只 include + 只声明方法 | 0（不定义 struct） |
| wallet.thrift | B-钱包核心 + D-记录相关 | 钱包余额查询、首页信息、B端用户钱包查询的类型 | 10-15 |
| currency.thrift | A-币种配置 | 币种列表、编辑、基准设置、启禁用、汇率日志的类型 | 10-15 |
| deposit.thrift | C-交易业务（充值） | 充值方式、充值订单、重复检测、奖金活动的类型 | 8-12 |
| withdraw.thrift | C-交易业务（提现） | 提现方式、提现账户、提现订单的类型 | 10-14 |
| exchange.thrift | C-交易业务（兑换） | 兑换汇率查询、执行兑换的类型 | 4-6 |
| record.thrift | D-记录查询 | 记录列表、记录详情的类型 | 4-6 |
| wallet_rpc.thrift | B-钱包核心（对外能力输出） | CreditWallet/DebitWallet/Freeze/Unfreeze 等 RPC 的类型 | 15-20 |

**总计：约 60-90 个 struct**（含 Req/Resp/Item），分散在 7 个域文件中，每个文件 4-20 个 struct，规模可控。

### 3.4 拆分合理性验证

与已有服务的文件数对比：

| 服务 | 方法数 | 文件数 | 文件/方法比 |
|------|--------|--------|-----------|
| ser-user | ~37 | 13 | 0.35 |
| ser-live | ~32 | 9 | 0.28 |
| ser-app | ~25 | 5 | 0.20 |
| ser-rtc | ~15 | 4 | 0.27 |
| **ser-wallet** | **~45** | **8** | **0.18** |

ser-wallet 的文件/方法比是最低的（最紧凑），在合理范围内。如果后续某个域膨胀（如提现业务复杂化），可以按已有规范继续拆分（如 `withdraw_account.thrift`、`withdraw_method.thrift`），不影响已有文件。

---

## 四、service.thrift 完整方法设计

### 4.1 设计原则

- **方法按调用端分组**：C端 → B端 → RPC 对外，用注释分隔
- **方法名不带 C/B 前缀**：参照已有服务（ser-live 不用 `CGetLiveRoomInfo`，直接 `GetLiveRoomInfo`），端的区分通过网关路由实现
- **方法名动词对齐已有规范**：`Create`/`Page`/`Detail`/`Edit`/`Status`/`List`/`Get`/`Do`
- **先列核心必须，再列应该有，可选/预留暂不写入 IDL**（IDL 支持向后追加，保留弹性）

### 4.2 完整方法声明

```thrift
namespace go ser_wallet

include "wallet.thrift"
include "currency.thrift"
include "deposit.thrift"
include "withdraw.thrift"
include "exchange.thrift"
include "record.thrift"
include "wallet_rpc.thrift"

service WalletService {

    // ==========================================
    // C端接口（gate-font 转发，面向 App/H5 用户）
    // ==========================================

    // --- 首页 & 币种 ---

    // 钱包首页：当前币种余额（中心+奖励）、钱包分层信息
    wallet.GetWalletHomeResp GetWalletHome(1: wallet.GetWalletHomeReq req)

    // 可用币种列表（名称/图标/余额/状态）
    wallet.GetCurrencyListResp GetCurrencyList(1: wallet.GetCurrencyListReq req)

    // --- 充值 ---

    // 当前币种下充值方式列表 + 档位 + 默认方式
    deposit.GetDepositMethodsResp GetDepositMethods(1: deposit.GetDepositMethodsReq req)

    // 创建充值订单（统一入口，内部按充值方式分流）
    deposit.CreateDepositOrderResp CreateDepositOrder(1: deposit.CreateDepositOrderReq req)

    // 重复转账检测（同用户 + 同币种 + 同金额的待支付订单）
    deposit.CheckDuplicateDepositResp CheckDuplicateDeposit(1: deposit.CheckDuplicateDepositReq req)

    // 可用奖金活动列表（充值关联的额外奖金）
    deposit.GetBonusActivitiesResp GetBonusActivities(1: deposit.GetBonusActivitiesReq req)

    // 充值订单状态查询（创建后前端轮询）
    deposit.GetDepositOrderStatusResp GetDepositOrderStatus(1: deposit.GetDepositOrderStatusReq req)

    // --- 兑换 ---

    // 兑换汇率查询（法币→BSB，含赠送比例）
    exchange.GetExchangeRateResp GetExchangeRate(1: exchange.GetExchangeRateReq req)

    // 执行兑换（法币→BSB 单向）
    exchange.DoExchangeResp DoExchange(1: exchange.DoExchangeReq req)

    // --- 提现 ---

    // 提现方式列表 + 限额 + 手续费
    withdraw.GetWithdrawMethodsResp GetWithdrawMethods(1: withdraw.GetWithdrawMethodsReq req)

    // 获取已保存的提现账户信息（银行卡/电子钱包）
    withdraw.GetWithdrawAccountResp GetWithdrawAccount(1: withdraw.GetWithdrawAccountReq req)

    // 首次保存 / 更新提现账户信息
    withdraw.SaveWithdrawAccountResp SaveWithdrawAccount(1: withdraw.SaveWithdrawAccountReq req)

    // 创建提现订单（含 KYC 校验、限额校验、冻结余额）
    withdraw.CreateWithdrawOrderResp CreateWithdrawOrder(1: withdraw.CreateWithdrawOrderReq req)

    // 提现订单状态查询
    withdraw.GetWithdrawOrderStatusResp GetWithdrawOrderStatus(1: withdraw.GetWithdrawOrderStatusReq req)

    // --- 记录 ---

    // 交易记录列表（充值/提现/兑换/奖励四 Tab，按币种隔离）
    record.GetRecordListResp GetRecordList(1: record.GetRecordListReq req)

    // 交易记录详情
    record.GetRecordDetailResp GetRecordDetail(1: record.GetRecordDetailReq req)

    // ==========================================
    // B端接口（gate-back 转发，面向管理后台）
    // ==========================================

    // --- 币种配置管理 ---

    // 币种列表分页查询（含实时汇率、平台汇率、偏差值）
    currency.PageCurrencyResp PageCurrency(1: currency.PageCurrencyReq req)

    // 编辑币种（图标/汇率浮动%/阈值%）
    currency.EditCurrencyResp EditCurrency(1: currency.EditCurrencyReq req)

    // 设置基准币种（一次性，二级确认）
    currency.SetBaseCurrencyResp SetBaseCurrency(1: currency.SetBaseCurrencyReq req)

    // 启用/禁用币种
    currency.StatusCurrencyResp StatusCurrency(1: currency.StatusCurrencyReq req)

    // 汇率日志分页查询
    currency.PageExchangeRateLogResp PageExchangeRateLog(1: currency.PageExchangeRateLogReq req)

    // --- 用户钱包管理 ---

    // B端用户钱包分页查询（客服/风控场景查余额）
    wallet.PageUserWalletResp PageUserWallet(1: wallet.PageUserWalletReq req)

    // B端用户钱包详情（按币种 + 钱包类型展开）
    wallet.DetailUserWalletResp DetailUserWallet(1: wallet.DetailUserWalletReq req)

    // ==========================================
    // RPC 对外提供（供财务/投注/直播/活动等服务调用）
    // ==========================================

    // --- 余额变更（原子操作）---

    // 上账：充值成功入账到中心钱包
    wallet_rpc.CreditWalletResp CreditWallet(1: wallet_rpc.CreditWalletReq req)

    // 扣款：消费/投注扣款（优先中心→再奖励）
    wallet_rpc.DebitWalletResp DebitWallet(1: wallet_rpc.DebitWalletReq req)

    // 冻结：提现申请时冻结对应金额
    wallet_rpc.FreezeBalanceResp FreezeBalance(1: wallet_rpc.FreezeBalanceReq req)

    // 解冻：提现驳回/取消时解冻
    wallet_rpc.UnfreezeBalanceResp UnfreezeBalance(1: wallet_rpc.UnfreezeBalanceReq req)

    // 确认扣除：提现出款成功后确认扣除冻结金额
    wallet_rpc.ConfirmDebitResp ConfirmDebit(1: wallet_rpc.ConfirmDebitReq req)

    // --- 人工修正 ---

    // 人工加款（A+ 单号，指定钱包类型）
    wallet_rpc.ManualCreditResp ManualCredit(1: wallet_rpc.ManualCreditReq req)

    // 人工减款（M+ 单号，余额不足时按最高金额钱包依次扣减）
    wallet_rpc.ManualDebitResp ManualDebit(1: wallet_rpc.ManualDebitReq req)

    // 充值补单（B+ 单号，修正充值金额 + 赠送金额）
    wallet_rpc.SupplementCreditResp SupplementCredit(1: wallet_rpc.SupplementCreditReq req)

    // --- 查询 ---

    // 查询用户钱包余额（按币种 + 钱包类型）
    wallet_rpc.GetBalanceResp GetBalance(1: wallet_rpc.GetBalanceReq req)

    // --- 返奖 ---

    // 活动/任务返奖入账（→奖励钱包）
    wallet_rpc.RewardCreditResp RewardCredit(1: wallet_rpc.RewardCreditReq req)
}
```

### 4.3 方法统计

| 分组 | 方法数 | 对应接口评估层级 |
|------|--------|-----------------|
| C端接口 | 16 | 核心必须 13 + 应该有 3 |
| B端接口 | 7 | 核心必须 7 |
| RPC 对外 | 10 | 核心必须 10 |
| **合计** | **33** | 全部为核心必须 + 应该有 |

**暂不写入 IDL 的可选/预留方法（7 个）**：

| 方法 | 原因 | 何时加入 |
|------|------|---------|
| TransferToCenter | 主播/代理钱包转入中心，当前版本可能未做 | 确认需求后加入 |
| GetWalletFlow | 资金流水明细，对账追溯用 | 对账需求明确后加入 |
| BetDebit | 投注扣款（需记录中心/奖励比例） | 与投注模块联调时加入 |
| BetReturn | 投注返奖 | 与投注模块联调时加入 |
| AnchorIncome | 主播收入入账 | 直播模块接入时加入 |
| BatchGetBalance | 批量查询余额 | 统计报表需求明确后加入 |
| AgentIncome / VenueTransfer | 代理佣金/场馆转入 | 标注"预留"的功能，暂不实现 |

> 设计依据：IDL 支持 optional 和方法追加，不影响已有调用方。先保持 IDL 精简，按需渐进扩展。

---

## 五、各域文件 struct 设计概要

> 注意：此处**只列出 struct 名称和用途**，不展开字段细节。
> 原因：① 需求处于预评审阶段，字段细节可能变动；② IDL 支持 optional 追加字段，后续迭代无断裂成本；③ 对齐设计原则"屏蔽细节、保留弹性"。

### 5.1 wallet.thrift（钱包核心类型）

```
namespace go ser_wallet

对应方法：GetWalletHome / GetCurrencyList / PageUserWallet / DetailUserWallet

struct 清单：
├── WalletHomeInfo          ← 钱包首页聚合信息（余额、钱包分层、币种信息）
├── WalletItem              ← 单个钱包信息（B端展示用）
├── WalletItemC             ← 单个钱包信息（C端简化版）
├── CurrencyItemC           ← C端币种列表项（名称/图标/余额/状态）
│
├── GetWalletHomeReq        ← C端：入参（币种标识）
├── GetWalletHomeResp       ← C端：返回首页聚合数据
├── GetCurrencyListReq      ← C端：入参（可选过滤）
├── GetCurrencyListResp     ← C端：返回可用币种列表
│
├── PageUserWalletReq       ← B端：分页查询用户钱包（筛选条件）
├── PageUserWalletResp      ← B端：分页返回（pageList/total/pageNo/pageSize）
├── DetailUserWalletReq     ← B端：用户钱包详情入参
└── DetailUserWalletResp    ← B端：按币种+钱包类型展开的详情
```

### 5.2 currency.thrift（币种配置类型）

```
namespace go ser_wallet

对应方法：PageCurrency / EditCurrency / SetBaseCurrency / StatusCurrency / PageExchangeRateLog

struct 清单：
├── CurrencyItem            ← 币种配置详情（B端展示：实时汇率/平台汇率/偏差值/浮动%/阈值%）
├── ExchangeRateLogItem     ← 汇率日志条目
│
├── PageCurrencyReq         ← B端：币种列表分页请求
├── PageCurrencyResp        ← B端：币种列表分页响应
├── EditCurrencyReq         ← B端：编辑币种请求
├── EditCurrencyResp        ← B端：编辑币种响应
├── SetBaseCurrencyReq      ← B端：设置基准币种请求（含确认标记）
├── SetBaseCurrencyResp     ← B端：设置基准币种响应
├── StatusCurrencyReq       ← B端：启用/禁用币种请求
├── StatusCurrencyResp      ← B端：启用/禁用币种响应
├── PageExchangeRateLogReq  ← B端：汇率日志分页请求
└── PageExchangeRateLogResp ← B端：汇率日志分页响应
```

### 5.3 deposit.thrift（充值类型）

```
namespace go ser_wallet

对应方法：GetDepositMethods / CreateDepositOrder / CheckDuplicateDeposit / GetBonusActivities / GetDepositOrderStatus

struct 清单：
├── DepositMethodItem       ← 充值方式项（方式类型/档位/默认标记）
├── BonusActivityItem       ← 奖金活动项（活动名称/奖金比例/门槛）
├── DepositOrderInfo        ← 充值订单信息（订单号/金额/状态/时间）
│
├── GetDepositMethodsReq    ← C端：入参（币种标识）
├── GetDepositMethodsResp   ← C端：返回充值方式列表
├── CreateDepositOrderReq   ← C端：创建充值订单请求
├── CreateDepositOrderResp  ← C端：返回订单号 + 支付跳转信息
├── CheckDuplicateDepositReq  ← C端：重复检测请求
├── CheckDuplicateDepositResp ← C端：是否存在重复 + 已有订单信息
├── GetBonusActivitiesReq   ← C端：入参（币种/充值金额）
├── GetBonusActivitiesResp  ← C端：返回可选奖金活动列表
├── GetDepositOrderStatusReq  ← C端：订单状态查询请求
└── GetDepositOrderStatusResp ← C端：返回订单当前状态
```

### 5.4 withdraw.thrift（提现类型）

```
namespace go ser_wallet

对应方法：GetWithdrawMethods / GetWithdrawAccount / SaveWithdrawAccount / CreateWithdrawOrder / GetWithdrawOrderStatus

struct 清单：
├── WithdrawMethodItem      ← 提现方式项（方式类型/限额/手续费率）
├── WithdrawAccountInfo     ← 提现账户信息（银行卡/电子钱包/持有人）
├── WithdrawOrderInfo       ← 提现订单信息（订单号/金额/状态/时间）
│
├── GetWithdrawMethodsReq   ← C端：入参（币种标识）
├── GetWithdrawMethodsResp  ← C端：返回提现方式列表
├── GetWithdrawAccountReq   ← C端：入参（提现方式类型）
├── GetWithdrawAccountResp  ← C端：返回已保存的账户信息
├── SaveWithdrawAccountReq  ← C端：保存/更新账户信息请求
├── SaveWithdrawAccountResp ← C端：保存结果
├── CreateWithdrawOrderReq  ← C端：创建提现订单请求
├── CreateWithdrawOrderResp ← C端：返回订单号
├── GetWithdrawOrderStatusReq  ← C端：订单状态查询请求
└── GetWithdrawOrderStatusResp ← C端：返回订单当前状态
```

### 5.5 exchange.thrift（兑换类型）

```
namespace go ser_wallet

对应方法：GetExchangeRate / DoExchange

struct 清单：
├── ExchangeRateInfo        ← 兑换汇率信息（汇率值/赠送比例/有效时间）
│
├── GetExchangeRateReq      ← C端：入参（源币种/目标币种=BSB）
├── GetExchangeRateResp     ← C端：返回汇率信息
├── DoExchangeReq           ← C端：执行兑换请求
└── DoExchangeResp          ← C端：返回兑换结果（兑换后金额/流水号）
```

### 5.6 record.thrift（记录查询类型）

```
namespace go ser_wallet

对应方法：GetRecordList / GetRecordDetail

struct 清单：
├── RecordItem              ← 记录列表项（类型/金额/状态/时间/简要描述）
├── RecordDetail            ← 记录详情（完整信息：订单号/到账金额/手续费/失败原因等）
│
├── GetRecordListReq        ← C端：入参（记录类型Tab/币种/分页）
├── GetRecordListResp       ← C端：返回记录列表（pageList/total/pageNo/pageSize）
├── GetRecordDetailReq      ← C端：入参（记录ID/记录类型）
└── GetRecordDetailResp     ← C端：返回记录详情
```

### 5.7 wallet_rpc.thrift（RPC 对外类型）

```
namespace go ser_wallet

对应方法：CreditWallet / DebitWallet / FreezeBalance / UnfreezeBalance / ConfirmDebit
         / ManualCredit / ManualDebit / SupplementCredit / GetBalance / RewardCredit

struct 清单：

// --- 余额变更 ---
├── CreditWalletReq         ← 上账请求（userId/currencyCode/amount/walletType/orderNo/flowType/remark）
├── CreditWalletResp        ← 上账结果（是否成功/变更后余额/幂等标记）
├── DebitWalletReq          ← 扣款请求（userId/currencyCode/amount/orderNo/flowType）
├── DebitWalletResp         ← 扣款结果（是否成功/各钱包实际扣款明细）
├── FreezeBalanceReq        ← 冻结请求（userId/currencyCode/amount/orderNo）
├── FreezeBalanceResp       ← 冻结结果
├── UnfreezeBalanceReq      ← 解冻请求（userId/currencyCode/amount/orderNo）
├── UnfreezeBalanceResp     ← 解冻结果
├── ConfirmDebitReq         ← 确认扣除请求（userId/currencyCode/orderNo）
├── ConfirmDebitResp        ← 确认结果

// --- 人工修正 ---
├── ManualCreditReq         ← 人工加款请求（userId/currencyCode/amount/walletType/orderNo/operatorId/remark）
├── ManualCreditResp        ← 人工加款结果
├── ManualDebitReq          ← 人工减款请求
├── ManualDebitResp         ← 人工减款结果
├── SupplementCreditReq     ← 充值补单请求（原充值订单号/补正金额/赠送金额/operatorId）
├── SupplementCreditResp    ← 充值补单结果

// --- 查询 ---
├── GetBalanceReq           ← 余额查询请求（userId/currencyCode/walletType 可选）
├── GetBalanceResp          ← 余额查询结果（可用余额/冻结余额/各钱包余额明细）

// --- 返奖 ---
├── RewardCreditReq         ← 活动/任务返奖请求（userId/currencyCode/amount/orderNo/activityId）
└── RewardCreditResp        ← 返奖结果
```

**wallet_rpc.thrift 特殊注意**：

- 所有字段带 `(api.body = "field")` 注解，对齐 user_rpc.thrift 规范
- 所有 Req 都必须包含 `orderNo`（幂等键），这是架构设计决策（幂等挑战 P0）
- 金额字段类型为 `i64`（最小单位整数），对齐 BIGINT 精度链
- 调用方（财务/投注等）import 这个文件的类型来构建请求

---

## 六、service.thrift 的 include 依赖关系

```
service.thrift
├── include "wallet.thrift"          ← 钱包核心
├── include "currency.thrift"        ← 币种配置
├── include "deposit.thrift"         ← 充值
├── include "withdraw.thrift"        ← 提现
├── include "exchange.thrift"        ← 兑换
├── include "record.thrift"          ← 记录
└── include "wallet_rpc.thrift"      ← RPC 对外

域文件之间：无交叉引用
  wallet.thrift      → 不 include 任何其他域文件
  currency.thrift    → 不 include 任何其他域文件
  deposit.thrift     → 不 include 任何其他域文件
  withdraw.thrift    → 不 include 任何其他域文件
  exchange.thrift    → 不 include 任何其他域文件
  record.thrift      → 不 include 任何其他域文件
  wallet_rpc.thrift  → 不 include 任何其他域文件
```

**设计决策：域文件之间不互相引用**

理由：
1. 已有工程中，域文件几乎不互相 include（banner.thrift 不引用 s3.thrift，各自独立）
2. 如果 `deposit.thrift` 需要 `wallet.thrift` 的类型，说明耦合了——应该由 service 层（Go 代码）组合，不在 IDL 层耦合
3. 如果有真正需要跨域共享的类型（如通用金额结构），提取到 `../common/` 下

**唯一的跨服务引用（如果需要的话）**：

```thrift
// 如果 ser-wallet 需要引用 common 枚举
include "../common/enum.thrift"
```

当前评估：ser-wallet 暂不需要引用 `../common/enum.thrift`，因为钱包的枚举（WalletType、FlowType 等）是服务内部定义，不是全局共享枚举。

---

## 七、IDL 与网关路由的映射关系

### 7.1 gate-font（C端路由）

```
路由注册位置：gate-font/biz/router/wallet/  （新增目录）

路由分组：
  my_router.Open.POST(...)   ← 无需登录的接口（暂无，钱包操作均需登录）
  my_router.Api.POST(...)    ← 需要登录的接口

路由 → handler → RPC 映射：
  POST /wallet/getWalletHome         → wallet.GetWalletHome     → rpc.Handler(ctx, c, rpc.WalletClient().GetWalletHome)
  POST /wallet/getCurrencyList       → wallet.GetCurrencyList   → rpc.Handler(ctx, c, rpc.WalletClient().GetCurrencyList)
  POST /wallet/getDepositMethods     → deposit.GetDepositMethods→ rpc.Handler(ctx, c, rpc.WalletClient().GetDepositMethods)
  POST /wallet/createDepositOrder    → deposit.CreateDeposit... → rpc.Handler(ctx, c, rpc.WalletClient().CreateDepositOrder)
  POST /wallet/checkDuplicateDeposit → ...
  POST /wallet/getBonusActivities    → ...
  POST /wallet/getDepositOrderStatus → ...
  POST /wallet/getExchangeRate       → ...
  POST /wallet/doExchange            → ...
  POST /wallet/getWithdrawMethods    → ...
  POST /wallet/getWithdrawAccount    → ...
  POST /wallet/saveWithdrawAccount   → ...
  POST /wallet/createWithdrawOrder   → ...
  POST /wallet/getWithdrawOrderStatus→ ...
  POST /wallet/getRecordList         → ...
  POST /wallet/getRecordDetail       → ...
```

**16 条 C端路由，全部 POST，全部走 `my_router.Api`（需登录）**

### 7.2 gate-back（B端路由）

```
路由注册位置：gate-back/biz/router/wallet/  （新增目录）

路由 → handler → RPC 映射：
  POST /wallet/currency/page            → rpc.Handler(ctx, c, rpc.WalletClient().PageCurrency)
  POST /wallet/currency/edit            → rpc.Handler(ctx, c, rpc.WalletClient().EditCurrency)
  POST /wallet/currency/setBase         → rpc.Handler(ctx, c, rpc.WalletClient().SetBaseCurrency)
  POST /wallet/currency/status          → rpc.Handler(ctx, c, rpc.WalletClient().StatusCurrency)
  POST /wallet/currency/rateLog/page    → rpc.Handler(ctx, c, rpc.WalletClient().PageExchangeRateLog)
  POST /wallet/user/page                → rpc.Handler(ctx, c, rpc.WalletClient().PageUserWallet)
  POST /wallet/user/detail              → rpc.Handler(ctx, c, rpc.WalletClient().DetailUserWallet)
```

**7 条 B端路由，全部 POST，全部走 `my_router.Api`（需 B端鉴权）**

### 7.3 RPC 方法无路由

RPC 对外方法（CreditWallet、DebitWallet 等 10 个）不经过网关，由其他服务直接通过 `rpc.WalletClient().CreditWallet(ctx, req)` 调用，走 Kitex RPC 协议 + ETCD 服务发现。

---

## 八、IDL 与代码生成的对应关系

### 8.1 gen_kitex.sh 的生成产物

```bash
# 执行命令（参照已有服务的生成脚本）
cd common/idl/ser-wallet
kitex -module gitlab.com/xxx/common idl/ser-wallet/service.thrift
```

生成目录：

```
common/kitex_gen/ser_wallet/
├── walletservice/          ← 服务接口 + 客户端代码
│   ├── client.go           ← WalletServiceClient
│   ├── invoker.go
│   ├── server.go
│   └── walletservice.go    ← 接口定义
├── k-ser_wallet.go
├── k-wallet.go
├── k-currency.go
├── k-deposit.go
├── k-withdraw.go
├── k-exchange.go
├── k-record.go
└── k-wallet_rpc.go
```

### 8.2 handler.go 骨架

```go
// ser-wallet/handler.go（自动生成骨架，手动填充业务逻辑）
type WalletServiceImpl struct{}

// C端方法
func (s *WalletServiceImpl) GetWalletHome(ctx context.Context, req *ser_wallet.GetWalletHomeReq) (*ser_wallet.GetWalletHomeResp, error) {
    // TODO: 业务逻辑
}

// B端方法
func (s *WalletServiceImpl) PageCurrency(ctx context.Context, req *ser_wallet.PageCurrencyReq) (*ser_wallet.PageCurrencyResp, error) {
    // TODO: 业务逻辑
}

// RPC方法
func (s *WalletServiceImpl) CreditWallet(ctx context.Context, req *ser_wallet.CreditWalletReq) (*ser_wallet.CreditWalletResp, error) {
    // TODO: 业务逻辑
}

// ... 共 33 个方法骨架
```

### 8.3 rpc_client.go 注册

```go
// common/rpc/rpc_client.go 新增：
var walletOnce sync.Once
var walletClient walletservice.Client

func WalletClient() walletservice.Client {
    walletOnce.Do(func() {
        walletClient, _ = walletservice.NewClient(
            namesp.EtcdWalletService,
            InitRpcClientParams()...,
        )
    })
    return walletClient
}
```

注册后，所有服务（gate-font、gate-back、ser-finance 等）可通过 `rpc.WalletClient()` 调用 ser-wallet 的全部 33 个方法。

---

## 九、设计决策记录（ADR）

| # | 决策 | 选项 | 选定 | 理由 |
|---|------|------|------|------|
| 1 | 单 Service 还是多 Service | A. 单 WalletService B. 拆 WalletService + CurrencyService | A | 20/20 已有服务均为单 Service，不破坏规范 |
| 2 | RPC 类型放哪 | A. 放 wallet.thrift B. 独立 wallet_rpc.thrift | B | 参照 user_rpc.thrift、live_rpc.thrift 规范 |
| 3 | C端/B端类型是否拆文件 | A. 同文件（BannerItem vs BannerItemC 模式） B. 分文件 | A | 已有工程 100% 选 A，C端用 XxxItemC 后缀 |
| 4 | 可选方法是否先写入 | A. 先不写 B. 全部写入 | A | 需求预评审，可选方法不确定，IDL 支持追加 |
| 5 | 域文件是否交叉引用 | A. 不引用 B. 按需引用 | A | 降低耦合，组合在 Go 代码层完成 |
| 6 | 金额字段类型 | A. i64 B. string C. double | A | 对齐 BIGINT 精度链，架构设计 ADR 已决 |
| 7 | 字段 required/optional 策略 | A. Req 区分，Resp 不标 B. 全部标注 | A | 对齐 banner.thrift 等已有规范 |
| 8 | include 是否引 common/enum | A. 暂不引 B. 引入 | A | 钱包枚举为服务内部定义，不是全局共享 |

---

## 十、扩展性评估

### 10.1 已考虑的扩展场景

| 场景 | 扩展方式 | 影响范围 |
|------|---------|---------|
| 新增投注扣款 RPC（BetDebit） | wallet_rpc.thrift 新增 struct + service.thrift 新增方法 | 不影响已有方法和调用方 |
| 新增主播收入 RPC（AnchorIncome） | 同上 | 同上 |
| 充值方式增加新类型 | deposit.thrift 新增 struct 或在已有 struct 中 optional 追加字段 | 不影响已有字段 |
| 提现流程变更 | withdraw.thrift 修改 Req/Resp 的 optional 字段 | optional 追加不破坏兼容 |
| 新增对账接口 | 新增 reconcile.thrift 域文件 + service.thrift include 并追加方法 | 不影响已有文件 |
| 币种配置功能膨胀 | currency.thrift 继续扩展，或拆分为 currency.thrift + currency_rate.thrift | 灵活拆分 |

### 10.2 向后兼容性保证

Thrift IDL 的兼容性规则：
- **新增 optional 字段**：旧客户端不传新字段 → 服务端收到零值 → 兼容
- **新增方法**：旧客户端不调用新方法 → 兼容
- **不可做的事**：删除字段、修改字段编号、修改字段类型 → 破坏兼容

因此，当前设计策略是：
- Req 中"可能变的字段"用 `optional`
- 字段编号一旦分配不再修改
- 弃用字段保留编号但加 `// deprecated` 注释

---

## 十一、与现有分析文档的一致性验证

| 验证项 | 参考文档 | 一致性 |
|--------|---------|--------|
| 方法名称和分组 | 实现思路/tmp-3.md §1.2 | ✅ 完全一致，方法名和分组策略直接沿用 |
| 接口数量（核心+应该有） | 接口评估/tmp-3.md 迭代版 | ✅ C端16 + B端7 + RPC10 = 33，在评估范围内 |
| 四子模块对应关系 | 流程思路/tmp-3.md §2.1 | ✅ 币种配置→currency / 钱包核心→wallet+wallet_rpc / 交易业务→deposit+withdraw+exchange / 记录查询→record |
| 金额类型 i64 | 架构设计/tmp-3.md ADR#3 | ✅ BIGINT 精度链，IDL 用 i64 |
| 幂等键 orderNo | 架构设计/tmp-3.md §4.2 | ✅ wallet_rpc.thrift 所有变更 Req 均含 orderNo |
| C端简化 struct | 工程规范（BannerItemC） | ✅ wallet.thrift 设计 WalletItemC/CurrencyItemC |

---

## 十二、开发阶段的 IDL 编写顺序建议

```
阶段一：基础能力（必须先行）
  ① wallet_rpc.thrift    ← RPC 对外类型先定义，让财务模块可以尽早集成
  ② wallet.thrift         ← 钱包核心类型（余额查询依赖此文件）
  ③ currency.thrift       ← 币种配置类型（B端管理的基础数据层）
  ④ service.thrift        ← 入口文件（include 已有的 3 个域文件，先声明 RPC + 币种方法）

阶段二：交易业务（依赖阶段一）
  ⑤ deposit.thrift        ← 充值相关类型
  ⑥ withdraw.thrift       ← 提现相关类型
  ⑦ exchange.thrift       ← 兑换相关类型
  更新 service.thrift     ← 追加 C端充值/提现/兑换方法

阶段三：查询展示（依赖阶段二）
  ⑧ record.thrift         ← 记录查询类型
  更新 service.thrift     ← 追加记录查询方法
```

**核心优先级**：`wallet_rpc.thrift` 必须最先完成。
- 财务模块需要调用 CreditWallet/DebitWallet/FreezeBalance 等接口
- 越早暴露 IDL，对方越早可以基于类型生成代码、编写集成逻辑
- 这是跨团队协作的关键路径

---

## 十三、总结

### 核心数字

| 维度 | 数量 |
|------|------|
| IDL 文件数 | 8 个（1 入口 + 6 域文件 + 1 RPC 文件） |
| service 方法数 | 33 个（C端 16 + B端 7 + RPC 10） |
| 预估 struct 数 | 60-90 个 |
| 可选/预留（暂不写入） | 7 个方法 |

### 设计全景

```
common/idl/ser-wallet/
│
├── service.thrift ─────────── WalletService（33 方法）
│   ├── include "wallet.thrift"
│   │   └── 钱包核心：GetWalletHome / GetCurrencyList / PageUserWallet / DetailUserWallet
│   │
│   ├── include "currency.thrift"
│   │   └── 币种配置：PageCurrency / EditCurrency / SetBaseCurrency / StatusCurrency / PageExchangeRateLog
│   │
│   ├── include "deposit.thrift"
│   │   └── 充值：GetDepositMethods / CreateDepositOrder / CheckDuplicateDeposit / GetBonusActivities / GetDepositOrderStatus
│   │
│   ├── include "withdraw.thrift"
│   │   └── 提现：GetWithdrawMethods / GetWithdrawAccount / SaveWithdrawAccount / CreateWithdrawOrder / GetWithdrawOrderStatus
│   │
│   ├── include "exchange.thrift"
│   │   └── 兑换：GetExchangeRate / DoExchange
│   │
│   ├── include "record.thrift"
│   │   └── 记录：GetRecordList / GetRecordDetail
│   │
│   └── include "wallet_rpc.thrift"
│       └── RPC：CreditWallet / DebitWallet / FreezeBalance / UnfreezeBalance / ConfirmDebit
│             ManualCredit / ManualDebit / SupplementCredit / GetBalance / RewardCredit
│
└── 域文件间无交叉引用 → 组合在 Go 业务代码层完成
```

### 一句话总结

> ser-wallet 采用 **1 入口 + 7 域文件** 的标准拆分策略，33 个方法按 C端/B端/RPC 三分组，完全对齐已有 20 个服务的 IDL 规范，字段细节保留弹性待需求定型后渐进补充，wallet_rpc.thrift 最先交付以解除跨团队协作阻塞。
