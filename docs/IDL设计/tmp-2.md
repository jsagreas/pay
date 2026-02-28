# ser-wallet IDL 设计方案

> 编写时间：2026-02-27
> 定位：IDL 文件结构设计 + 方法定义 + 结构体规划（粗粒度骨架，屏蔽字段细节）
> 依据来源：
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-2.md（第三版，我方 46 个接口）
> - 实现思路：/Users/mac/gitlab/z-readme/实现思路/tmp-2.md（6 个核心 RPC + 5 个余额方法）
> - 现有 IDL 工程规范：common/idl/ 下 24 个服务的 75+ .thrift 文件实际分析
> - gen_kitex.sh 脚本逻辑：common/script/gen_kitex.sh
> - 阶段：预评审，核心骨架确定，字段细节保留弹性

---

## 一、设计方法论

### 1.1 设计原则

| 原则 | 说明 | 依据 |
|------|------|------|
| **规范先行** | 100% 遵循现有工程 IDL 规范，不发明新约定 | 24 个服务的 IDL 实际分析 |
| **适度拆分** | 按功能域拆分文件，不过度细碎也不堆成一个大文件 | ser-live（8个域文件）、ser-user（13个域文件）实践参考 |
| **骨架优先** | 先定方法签名和结构体名称，字段细节评审后再填充 | 预评审阶段策略 |
| **对齐接口** | 每个 IDL 方法对应接口评估中的一个明确接口编号 | 接口评估 tmp-2.md |
| **兼顾生成** | service.thrift 命名和结构必须兼容 gen_kitex.sh 自动扫描 | gen_kitex.sh 第31行逻辑 |

### 1.2 设计输入

```
接口评估（46个接口）
    │
    ├── C端 HTTP 19个（C-1 ~ C-19）
    ├── B端 HTTP 6个（B-1 ~ B-6）
    ├── RPC 供他方 12个（R-1 ~ R-12）
    ├── RPC 调他方 6个（O-1 ~ O-6）← 不在我方 IDL 中定义
    └── 定时任务 3个（T-1 ~ T-3）← 不在 IDL 中定义

    需要在 IDL 中定义的 = C端 + B端 + RPC供他方 = 19 + 6 + 12 = 37 个方法
    RPC调他方 → 在对方的 IDL 中定义，我方只是调用方
    定时任务 → 内部实现，不暴露为 RPC 方法
```

---

## 二、现有工程 IDL 规范提炼

> 以下规范从 24 个服务的实际 IDL 文件中归纳，不是臆测

### 2.1 目录与文件规范

| 规范项 | 实际规则 | 依据样本 |
|--------|---------|---------|
| 目录位置 | `common/idl/ser-{name}/` | 全部 24 个服务 |
| 入口文件 | `service.thrift`（必须包含 "service" 关键字） | gen_kitex.sh 第31行：`find ... -name "*service.thrift"` |
| 域文件 | `{domain}.thrift`，按功能域拆分 | ser-app: banner.thrift / s3.thrift / home_category.thrift |
| 跨服务引用 | `include "../ser-xxx/yyy.thrift"` | ser-kyc/service.thrift: `include "../common/enum.thrift"` |

### 2.2 命名规范

| 规范项 | 实际规则 | 依据样本 |
|--------|---------|---------|
| namespace | `namespace go ser_{name}`（下划线连接） | ser_app、ser_user、ser_live、ser_kyc |
| Service | `{Name}Service` — 大驼峰 | AppService、UserService、LiveService |
| 结构体 | 大驼峰，名词性 | BannerItem、UserKycInfo、LiveRoomListReq |
| 请求 | `{Action}{Domain}Req` 或 `{Domain}{Action}Req` | CreateBannerReq、KycSubmitReq、LiveRoomListReq |
| 响应 | `{Action}{Domain}Resp` 或 `{Domain}{Action}Resp` | CreateBannerResp、KycQueryResp、LiveRoomListResp |
| 方法 | 大驼峰动词开头 | CreateBanner、GetLiveRoomList、QueryKycStatus |

### 2.3 字段规范

| 规范项 | 实际规则 | 依据样本 |
|--------|---------|---------|
| 编号 | 从 1 开始，顺序递增 | 全部文件 |
| 修饰符 | 显式标注 `required` / `optional` | banner.thrift、user_kyc.thrift |
| 类型 | i64=ID/时间戳/金额，i32=状态/枚举/页码，string=文本/代码 | 全部文件 |
| 注释 | 每个字段行尾 `//` 注释说明 | banner.thrift、user_kyc.thrift |
| 分页请求 | pageNo(i32) + pageSize(i32) | PageBannerReq |
| 分页响应 | pageList(list) + total(i64) + pageNo(i32) + pageSize(i32) | PageBannerResp |

### 2.4 方法定义规范

| 规范项 | 实际规则 | 依据样本 |
|--------|---------|---------|
| 参数 | `1: domain.XxxReq req` — 单参数 struct | 全部 service.thrift |
| 返回 | `domain.XxxResp` — 单返回 struct | 全部 service.thrift |
| 注释 | 方法前一行 `// 中文说明` | 全部 service.thrift |
| 空参数 | 允许无参数 `MethodName()` | UserService: `GetUserDetail()` |
| 前缀引用 | 使用域文件名前缀 `banner.CreateBannerResp` | AppService |

### 2.5 gen_kitex.sh 关键逻辑

```bash
# 第31行：只扫描匹配 *service.thrift 的文件
find "$idl_dir" -maxdepth 1 -name "*service.thrift"

# 第47行：-module common/pkg -service $service_name
kitex -module common/pkg -I $IDL_BASE_DIR/$service_name -service $service_name -gen-path ./kitex_gen/ $rel_thrift_path
```

**关键点**：
1. 只有 `*service.thrift` 匹配的文件会被 kitex 处理，其他 `.thrift` 文件通过 `include` 被间接引入
2. `-I` 参数指向服务自己的 IDL 目录，跨服务引用需要用相对路径
3. 生成代码输出到 `common/pkg/kitex_gen/ser_{name}/`
4. 如果 `$WORKSPACE_DIR/ser-wallet/handler.go` 和 `main.go` 已存在，会先备份再覆盖再恢复

---

## 三、ser-wallet IDL 文件结构设计

### 3.1 文件清单

```
common/idl/ser-wallet/
├── service.thrift          ← 服务入口（WalletService 所有方法定义）
├── wallet.thrift           ← 钱包核心（余额查询/加减款/冻结解冻/消费优先级）
├── currency.thrift         ← 币种配置（B端CRUD/汇率/基准币种/RPC供他方）
├── deposit.thrift          ← 充值（充值方式/创建订单/USDT/重复检测/状态查询）
├── withdraw.thrift         ← 提现（提现方式/创建订单/账户管理/限额）
├── exchange.thrift         ← 兑换（预览/执行）
└── record.thrift           ← 账变记录（列表/详情）
```

**共 7 个文件（1 service + 6 domain）**

### 3.2 文件数量论证

| 对比维度 | ser-app | ser-user | ser-live | ser-wallet（设计） |
|----------|---------|----------|----------|-------------------|
| 域文件数 | 4 | 13 | 8 | **6** |
| 服务方法数 | 21 | 34 | 45 | **~37** |
| 文件/方法比 | 1:5.2 | 1:2.6 | 1:5.6 | **1:5.3** |

- ser-wallet 的 6 个域文件对应 6 个清晰的功能域：钱包核心、币种配置、充值、提现、兑换、记录
- 每个文件平均 ~6 个方法的结构体定义，体量适中
- 与 ser-app（4个域文件/21方法）和 ser-live（8个域文件/45方法）处于同一量级
- **不多不少**：比 ser-user 的 13 个文件更聚焦（不拆太碎），比堆成 1~2 个大文件更清晰

### 3.3 为什么不合并 / 不继续拆分

**不合并的理由**：
- 如果把 deposit/withdraw/exchange 合成一个 `transaction.thrift`，单文件结构体数量过多（预估 20+），查找困难
- 充值、提现、兑换三者的业务概念和生命周期完全不同，合并违反单一职责

**不继续拆分的理由**：
- ser-user 的 13 个文件中有些文件只有 1~2 个结构体（如 user_kyc.thrift 只有 2 个 struct），过于碎片化
- wallet 的 6 个域恰好对应接口评估中的 6 个功能域，粒度天然匹配
- 不需要单独的 `s3.thrift`（图标上传可以复用 common 的 S3 能力或内嵌到 currency.thrift）

---

## 四、service.thrift 设计

### 4.1 整体结构

```thrift
namespace go ser_wallet

include "wallet.thrift"
include "currency.thrift"
include "deposit.thrift"
include "withdraw.thrift"
include "exchange.thrift"
include "record.thrift"

service WalletService {

    // ========== 钱包核心 — RPC 供他方调用 ==========

    // 加款入账（充值成功/补单/人工加款/活动奖金/投注返奖）
    wallet.CreditWalletResp CreditWallet(1: wallet.CreditWalletReq req)
    // 减款扣款（人工减款/指定钱包扣款）
    wallet.DebitWalletResp DebitWallet(1: wallet.DebitWalletReq req)
    // 冻结余额（提现订单创建时冻结）
    wallet.FreezeBalanceResp FreezeBalance(1: wallet.FreezeBalanceReq req)
    // 解冻余额（提现审核驳回）
    wallet.UnfreezeBalanceResp UnfreezeBalance(1: wallet.UnfreezeBalanceReq req)
    // 扣除冻结金额（出款成功后终态扣款）
    wallet.DeductFrozenResp DeductFrozen(1: wallet.DeductFrozenReq req)
    // 查询钱包余额（各子钱包可用+冻结）
    wallet.QueryBalanceResp QueryBalance(1: wallet.QueryBalanceReq req)
    // 按消费优先级自动扣款（先中心后奖励）
    wallet.DebitByPriorityResp DebitByPriority(1: wallet.DebitByPriorityReq req)
    // 批量查询多用户余额
    wallet.BatchQueryBalanceResp BatchQueryBalance(1: wallet.BatchQueryBalanceReq req)
    // 查询用户账变流水（RPC版，供B端用户管理模块调用）
    wallet.GetWalletFlowResp GetWalletFlow(1: wallet.GetWalletFlowReq req)

    // ========== 钱包核心 — C端接口 ==========

    // 钱包首页余额总览（各子钱包余额+资金钱包合计）
    wallet.WalletOverviewResp GetWalletOverview(1: wallet.WalletOverviewReq req)

    // ========== 币种配置 — B端管理接口 ==========

    // 币种列表查询（分页+筛选+实时汇率）
    currency.CurrencyListResp GetCurrencyList(1: currency.CurrencyListReq req)
    // 编辑币种（汇率浮动/阈值/图标/状态等）
    currency.EditCurrencyResp EditCurrency(1: currency.EditCurrencyReq req)
    // 启用/禁用币种
    currency.ToggleCurrencyStatusResp ToggleCurrencyStatus(1: currency.ToggleCurrencyStatusReq req)
    // 设置基准币种（一次性，设后不可改）
    currency.SetBaseCurrencyResp SetBaseCurrency(1: currency.SetBaseCurrencyReq req)
    // 汇率日志查询（分页+筛选）
    currency.RateLogListResp GetRateLogList(1: currency.RateLogListReq req)
    // 获取单个币种详情
    currency.CurrencyDetailResp GetCurrencyDetail(1: currency.CurrencyDetailReq req)

    // ========== 币种配置 — RPC 供他方调用 ==========

    // 获取可用币种配置列表
    currency.CurrencyConfigResp GetCurrencyConfig(1: currency.CurrencyConfigReq req)
    // 获取指定币种汇率信息
    currency.ExchangeRateResp GetExchangeRate(1: currency.ExchangeRateReq req)
    // 金额币种换算
    currency.ConvertAmountResp ConvertAmount(1: currency.ConvertAmountReq req)

    // ========== 充值 — C端接口 ==========

    // 获取充值方式列表
    deposit.DepositMethodsResp GetDepositMethods(1: deposit.DepositMethodsReq req)
    // 创建充值订单
    deposit.CreateDepositResp CreateDeposit(1: deposit.CreateDepositReq req)
    // 获取USDT链上充值信息（游客可用）
    deposit.USDTDepositInfoResp GetUSDTDepositInfo(1: deposit.USDTDepositInfoReq req)
    // 重复转账检测
    deposit.DuplicateCheckResp CheckDuplicateDeposit(1: deposit.DuplicateCheckReq req)
    // 获取充值可选奖金活动列表
    deposit.BonusActivitiesResp GetBonusActivities(1: deposit.BonusActivitiesReq req)
    // 查询充值订单支付状态（前端轮询）
    deposit.DepositStatusResp GetDepositStatus(1: deposit.DepositStatusReq req)
    // 取消充值订单
    deposit.CancelDepositResp CancelDeposit(1: deposit.CancelDepositReq req)

    // ========== 提现 — C端接口 ==========

    // 获取提现方式列表（KYC状态过滤+充值路由规则）
    withdraw.WithdrawMethodsResp GetWithdrawMethods(1: withdraw.WithdrawMethodsReq req)
    // 创建提现订单（校验+冻结+生成订单）
    withdraw.CreateWithdrawResp CreateWithdraw(1: withdraw.CreateWithdrawReq req)
    // 获取已绑定提现账户信息
    withdraw.WithdrawAccountResp GetWithdrawAccount(1: withdraw.WithdrawAccountReq req)
    // 首次保存提现账户信息
    withdraw.SaveWithdrawAccountResp SaveWithdrawAccount(1: withdraw.SaveWithdrawAccountReq req)
    // 获取提现限额信息
    withdraw.WithdrawLimitsResp GetWithdrawLimits(1: withdraw.WithdrawLimitsReq req)

    // ========== 兑换 — C端接口 ==========

    // 兑换预览（输入金额→返回预计到账+赠送+流水要求）
    exchange.ExchangePreviewResp GetExchangePreview(1: exchange.ExchangePreviewReq req)
    // 执行兑换（法币→BSB）
    exchange.CreateExchangeResp CreateExchange(1: exchange.CreateExchangeReq req)

    // ========== 账变记录 — C端接口 ==========

    // 账变记录列表（按Tab类型分页查询）
    record.TransactionListResp GetTransactionList(1: record.TransactionListReq req)
    // 账变记录详情
    record.TransactionDetailResp GetTransactionDetail(1: record.TransactionDetailReq req)

    // ========== 通用 — C端接口 ==========

    // 获取当前币种汇率信息
    currency.CurrentRateResp GetCurrentRate(1: currency.CurrentRateReq req)
    // 获取C端可用币种列表
    currency.AvailableCurrenciesResp GetAvailableCurrencies(1: currency.AvailableCurrenciesReq req)
}
```

### 4.2 方法汇总与接口编号映射

| IDL 方法 | 接口编号 | 维度 | 优先级 |
|----------|---------|------|--------|
| CreditWallet | R-1 | RPC供他方 | Must |
| DebitWallet | R-2 | RPC供他方 | Must |
| FreezeBalance | R-3 | RPC供他方 | Must |
| UnfreezeBalance | R-4 | RPC供他方 | Must |
| DeductFrozen | R-5 | RPC供他方 | Must |
| QueryBalance | R-6 | RPC供他方 | Must |
| DebitByPriority | R-7 | RPC供他方 | Should |
| BatchQueryBalance | R-11 | RPC供他方 | Could |
| GetWalletFlow | R-12 | RPC供他方 | Could |
| GetWalletOverview | C-1 | C端 | Must |
| GetCurrencyList | B-1 | B端 | Must |
| EditCurrency | B-2 | B端 | Must |
| ToggleCurrencyStatus | B-3 | B端 | Must |
| SetBaseCurrency | B-4 | B端 | Must |
| GetRateLogList | B-5 | B端 | Must |
| GetCurrencyDetail | B-6 | B端 | Should |
| GetCurrencyConfig | R-8 | RPC供他方 | Should |
| GetExchangeRate | R-9 | RPC供他方 | Should |
| ConvertAmount | R-10 | RPC供他方 | Should |
| GetDepositMethods | C-2 | C端 | Must |
| CreateDeposit | C-3 | C端 | Must |
| GetUSDTDepositInfo | C-4 | C端 | Must |
| CheckDuplicateDeposit | C-14 | C端 | Should |
| GetBonusActivities | C-15 | C端 | Should |
| GetDepositStatus | C-16 | C端 | Should |
| CancelDeposit | C-18 | C端 | Could |
| GetWithdrawMethods | C-7 | C端 | Must |
| CreateWithdraw | C-8 | C端 | Must |
| GetWithdrawAccount | C-9 | C端 | Must |
| SaveWithdrawAccount | C-10 | C端 | Must |
| GetWithdrawLimits | C-17 | C端 | Should |
| GetExchangePreview | C-5 | C端 | Must |
| CreateExchange | C-6 | C端 | Must |
| GetTransactionList | C-11 | C端 | Must |
| GetTransactionDetail | C-12 | C端 | Must |
| GetCurrentRate | C-13 | C端 | Must |
| GetAvailableCurrencies | C-19 | C端 | Could |

**方法总计：37 个**（= 接口评估 46 - RPC调他方 6 - 定时任务 3 = 37）

---

## 五、各域文件结构体规划

> 以下只列出结构体名称和用途，不展开具体字段。
> 字段细节在评审确认需求后再填充。
> 遵循原则：每个方法对应一个 Req + 一个 Resp，共用的数据实体单独定义。

### 5.1 wallet.thrift — 钱包核心

```thrift
namespace go ser_wallet

// ========== 数据实体 ==========

// 钱包账户信息（余额展示用）
struct WalletAccountInfo { ... }

// 账变流水记录项
struct WalletFlowItem { ... }

// ========== 供他方调用 RPC — 联调合同接口 ==========

// R-1 加款
struct CreditWalletReq { ... }     // userId/currency/walletType/amount/orderNo/opType/稽核参数(optional)
struct CreditWalletResp { ... }    // 操作后余额

// R-2 减款
struct DebitWalletReq { ... }      // userId/currency/walletType/amount/orderNo/opType
struct DebitWalletResp { ... }

// R-3 冻结
struct FreezeBalanceReq { ... }    // userId/currency/walletType/amount/orderNo
struct FreezeBalanceResp { ... }

// R-4 解冻
struct UnfreezeBalanceReq { ... }  // userId/currency/walletType/amount/orderNo
struct UnfreezeBalanceResp { ... }

// R-5 扣除冻结
struct DeductFrozenReq { ... }     // userId/currency/walletType/amount/orderNo
struct DeductFrozenResp { ... }

// R-6 查询余额
struct QueryBalanceReq { ... }     // userId/currency(optional)
struct QueryBalanceResp { ... }    // list<WalletAccountInfo>

// R-7 消费优先级扣款
struct DebitByPriorityReq { ... }  // userId/currency/amount/orderNo/opType
struct DebitByPriorityResp { ... } // 各钱包实际扣款明细

// R-11 批量查询余额
struct BatchQueryBalanceReq { ... }  // list<userId>
struct BatchQueryBalanceResp { ... }

// R-12 查询账变流水（RPC版）
struct GetWalletFlowReq { ... }    // userId/currency/type/分页
struct GetWalletFlowResp { ... }   // list<WalletFlowItem>/分页信息

// ========== C端接口 ==========

// C-1 钱包余额总览
struct WalletOverviewReq { ... }   // currency(optional)
struct WalletOverviewResp { ... }  // list<WalletAccountInfo>/资金钱包合计
```

**结构体数量：约 22 个**（10 个 Req + 10 个 Resp + 2 个数据实体）

### 5.2 currency.thrift — 币种配置

```thrift
namespace go ser_wallet

// ========== 数据实体 ==========

// 币种配置信息（B端完整版，20个配置字段）
struct CurrencyInfo { ... }

// 币种配置简要信息（C端/RPC供他方，只含必要字段）
struct CurrencyBrief { ... }

// 汇率信息
struct RateInfo { ... }

// 汇率日志条目
struct RateLogItem { ... }

// ========== B端管理接口 ==========

// B-1 币种列表
struct CurrencyListReq { ... }     // 筛选条件+分页
struct CurrencyListResp { ... }    // pageList<CurrencyInfo>/分页信息

// B-2 编辑币种
struct EditCurrencyReq { ... }     // id/汇率浮动/阈值/图标/状态等
struct EditCurrencyResp { ... }

// B-3 启用/禁用
struct ToggleCurrencyStatusReq { ... }  // id/status
struct ToggleCurrencyStatusResp { ... }

// B-4 设置基准币种
struct SetBaseCurrencyReq { ... }  // currencyCode
struct SetBaseCurrencyResp { ... }

// B-5 汇率日志
struct RateLogListReq { ... }      // 日期范围+币种筛选+分页
struct RateLogListResp { ... }     // pageList<RateLogItem>/分页信息

// B-6 币种详情
struct CurrencyDetailReq { ... }   // id
struct CurrencyDetailResp { ... }  // CurrencyInfo

// ========== RPC 供他方调用 ==========

// R-8 获取可用币种配置
struct CurrencyConfigReq { ... }   // 可选筛选条件
struct CurrencyConfigResp { ... }  // list<CurrencyBrief>

// R-9 获取汇率
struct ExchangeRateReq { ... }     // currencyCode
struct ExchangeRateResp { ... }    // RateInfo

// R-10 金额换算
struct ConvertAmountReq { ... }    // amount/fromCurrency/toCurrency
struct ConvertAmountResp { ... }   // convertedAmount/使用的汇率

// ========== C端通用接口 ==========

// C-13 当前币种汇率
struct CurrentRateReq { ... }      // currencyCode
struct CurrentRateResp { ... }     // 入款/出款/BSB换算比

// C-19 C端可用币种列表
struct AvailableCurrenciesReq { ... }
struct AvailableCurrenciesResp { ... } // list<CurrencyBrief>
```

**结构体数量：约 26 个**（11 个 Req + 11 个 Resp + 4 个数据实体）

### 5.3 deposit.thrift — 充值

```thrift
namespace go ser_wallet

// ========== 数据实体 ==========

// 充值方式信息
struct DepositMethodItem { ... }

// 奖金活动信息
struct BonusActivityItem { ... }

// ========== C端接口 ==========

// C-2 充值方式列表
struct DepositMethodsReq { ... }   // currency
struct DepositMethodsResp { ... }  // list<DepositMethodItem>

// C-3 创建充值订单
struct CreateDepositReq { ... }    // currency/amount/methodId/bonusActivityId(optional)
struct CreateDepositResp { ... }   // orderNo/支付信息(URL/QR/地址等)

// C-4 USDT充值信息
struct USDTDepositInfoReq { ... }  // networkType(optional)
struct USDTDepositInfoResp { ... } // 网络列表/收款地址/QR码

// C-14 重复转账检测
struct DuplicateCheckReq { ... }   // currency/amount
struct DuplicateCheckResp { ... }  // 待处理同金额订单数/弹窗策略

// C-15 奖金活动
struct BonusActivitiesReq { ... }  // currency/amount
struct BonusActivitiesResp { ... } // list<BonusActivityItem>

// C-16 充值订单状态
struct DepositStatusReq { ... }    // orderNo
struct DepositStatusResp { ... }   // status/完成信息

// C-18 取消充值
struct CancelDepositReq { ... }    // orderNo
struct CancelDepositResp { ... }
```

**结构体数量：约 16 个**（7 个 Req + 7 个 Resp + 2 个数据实体）

### 5.4 withdraw.thrift — 提现

```thrift
namespace go ser_wallet

// ========== 数据实体 ==========

// 提现方式信息
struct WithdrawMethodItem { ... }

// 提现账户信息
struct WithdrawAccountInfo { ... }

// ========== C端接口 ==========

// C-7 提现方式列表
struct WithdrawMethodsReq { ... }    // currency
struct WithdrawMethodsResp { ... }   // list<WithdrawMethodItem>

// C-8 创建提现订单
struct CreateWithdrawReq { ... }     // currency/amount/methodId/accountInfo
struct CreateWithdrawResp { ... }    // orderNo/冻结确认

// C-9 获取提现账户
struct WithdrawAccountReq { ... }    // currency/methodType(optional)
struct WithdrawAccountResp { ... }   // WithdrawAccountInfo(optional)

// C-10 保存提现账户
struct SaveWithdrawAccountReq { ... }  // currency/methodType/账户字段
struct SaveWithdrawAccountResp { ... }

// C-17 提现限额
struct WithdrawLimitsReq { ... }     // currency/methodId
struct WithdrawLimitsResp { ... }    // 每日限额/单笔最低/手续费率/今日已用额度
```

**结构体数量：约 12 个**（5 个 Req + 5 个 Resp + 2 个数据实体）

### 5.5 exchange.thrift — 兑换

```thrift
namespace go ser_wallet

// ========== C端接口 ==========

// C-5 兑换预览
struct ExchangePreviewReq { ... }    // fromCurrency/amount
struct ExchangePreviewResp { ... }   // BSB到账/赠送金额/流水要求/使用汇率

// C-6 执行兑换
struct CreateExchangeReq { ... }     // fromCurrency/amount/确认标识
struct CreateExchangeResp { ... }    // orderNo/实际到账
```

**结构体数量：4 个**（2 个 Req + 2 个 Resp）

### 5.6 record.thrift — 账变记录

```thrift
namespace go ser_wallet

// ========== 数据实体 ==========

// 账变记录列表项
struct TransactionItem { ... }

// 账变记录详情
struct TransactionDetailInfo { ... }

// ========== C端接口 ==========

// C-11 账变记录列表
struct TransactionListReq { ... }     // currency/tabType/分页
struct TransactionListResp { ... }    // pageList<TransactionItem>/分页信息

// C-12 账变记录详情
struct TransactionDetailReq { ... }   // id 或 orderNo
struct TransactionDetailResp { ... }  // TransactionDetailInfo
```

**结构体数量：6 个**（2 个 Req + 2 个 Resp + 2 个数据实体）

### 5.7 全局结构体统计

| 文件 | 数据实体 | Req | Resp | 合计 |
|------|---------|-----|------|------|
| wallet.thrift | 2 | 10 | 10 | **22** |
| currency.thrift | 4 | 11 | 11 | **26** |
| deposit.thrift | 2 | 7 | 7 | **16** |
| withdraw.thrift | 2 | 5 | 5 | **12** |
| exchange.thrift | 0 | 2 | 2 | **4** |
| record.thrift | 2 | 2 | 2 | **6** |
| **合计** | **12** | **37** | **37** | **86** |

---

## 六、跨服务 IDL 依赖分析

### 6.1 我方需要引用他方 IDL 吗？

**不需要。**

```
现有跨服务引用模式：
  ser-kyc/service.thrift → include "../common/enum.thrift"   ✅ 引用公共枚举
  ser-app/service.thrift → include "../common/enum.thrift"   ✅ 引用公共枚举

ser-wallet 的跨服务调用方式：
  调 ser-kyc  → 通过 rpc_client.go 的 KycClient() 调用，不需要 include KYC 的 IDL
  调 ser-user → 通过 rpc_client.go 的 UserClient() 调用，不需要 include User 的 IDL
  调 财务模块  → 财务模块尚不存在，后续通过 rpc_client 调用
```

**结论**：ser-wallet 的 service.thrift 只 include 自己的 6 个域文件，不引用其他服务的 IDL。跨服务通过 RPC Client 调用，类型已在 `kitex_gen` 中独立生成。

### 6.2 他方需要引用我方 IDL 吗？

**需要。**

财务模块和其他模块调用我方 RPC 时，需要引用 `kitex_gen/ser_wallet/` 下生成的代码。但这是代码层面的依赖（Go import），不是 IDL 层面的 include。

```
财务模块调用链路：
  财务代码 → import "common/pkg/kitex_gen/ser_wallet" → 使用 CreditWalletReq 等类型

  这意味着：
  1. 我方 IDL 定义 + gen_kitex.sh 生成代码后
  2. 其他模块通过 Go import 引用 kitex_gen 下的类型
  3. 无需在其他模块的 IDL 中 include 我方 thrift 文件
```

### 6.3 是否需要引用 common/enum.thrift？

```
common/idl/common/enum.thrift 现有内容 — 通用枚举定义
```

**视情况决定**：
- 如果 ser-wallet 需要使用公共枚举（如通用状态码），则 `include "../common/enum.thrift"`
- 目前钱包模块的枚举（钱包类型、币种类型、订单状态等）都是领域专属的，大概率自行定义
- **建议**：初始不引用 common/enum.thrift，除非发现有可复用的枚举值

---

## 七、关键设计考量

### 7.1 namespace 统一为 ser_wallet

```thrift
// 所有 6 个域文件 + service.thrift 统一使用同一 namespace
namespace go ser_wallet
```

**依据**：ser-app 所有域文件（banner.thrift、s3.thrift、home_category.thrift）的 namespace 都是 `ser_app`，不按域分 namespace。生成的 Go 代码全部在同一个包 `ser_wallet` 下。

### 7.2 6 个核心 RPC 是联调合同

wallet.thrift 中的 R-1 ~ R-6 是与财务模块的**接口合同**：

```
CreditWallet / DebitWallet / FreezeBalance / UnfreezeBalance / DeductFrozen / QueryBalance
```

这 6 个方法的 Req/Resp 结构在 IDL 中定义后，等效于双方开发的"合同"：
- 财务模块按此 IDL 构造请求
- 我方按此 IDL 解析并处理
- **评审时重点关注这 6 个接口的入参出参**

### 7.3 C端 vs B端 vs RPC 在同一 Service 中

**设计决策**：所有方法定义在一个 `WalletService` 中，不按 C端/B端/RPC 拆分多个 Service。

**理由**：
- ser-live 同样将后台接口、主播端接口、观众端接口、RPC接口全部放在一个 `LiveService` 中
- ser-user 同样将 C端、B端、RPC 全部放在一个 `UserService` 中
- Kitex 生成代码时一个 service.thrift 对应一个 handler.go，多 Service 需要多文件管理，增加复杂度
- C端/B端的路由区分在 gateway 层（gate-font / gate-back）处理，不在 RPC 层区分

**用分组注释区分**：
```thrift
// ========== 钱包核心 — RPC 供他方调用 ==========
// ========== 充值 — C端接口 ==========
// ========== 币种配置 — B端管理接口 ==========
```

遵循 ser-live 的 `// ========== 主播端接口 ==========` 分组注释风格。

### 7.4 Req/Resp 命名一致性

**命名模式选择**：`{Domain}{Action}Req` / `{Domain}{Action}Resp`

对比现有工程两种风格：
- ser-app: `CreateBannerReq` / `PageBannerReq`（Action + Domain）
- ser-kyc: `KycSubmitReq` / `KycQueryReq`（Domain + Action）
- ser-live: `LiveRoomListReq`（Domain + Action）

**ser-wallet 采用混合但一致的策略**：
- RPC 核心方法：以动作命名 → `CreditWalletReq`、`FreezeBalanceReq`（与方法名对齐）
- B端 CRUD：以动作命名 → `EditCurrencyReq`、`SetBaseCurrencyReq`
- C端查询：以场景命名 → `WalletOverviewReq`、`DepositMethodsReq`

核心原则：**Req/Resp 名称与 Service 方法名保持强对应关系**，避免名称分离造成混乱。

### 7.5 分页遵循统一模式

```thrift
// 请求
struct XxxListReq {
    // ... 筛选条件 ...
    N: optional i32 pageNo       // 页码，从1开始
    N+1: optional i32 pageSize   // 每页条数
}

// 响应
struct XxxListResp {
    1: list<XxxItem> pageList    // 业务数据
    2: i64 total                 // 总记录数
    3: i32 pageNo                // 当前页码
    4: i32 pageSize              // 每页记录数
}
```

完全遵循 ser-app `PageBannerReq` / `PageBannerResp` 的既有模式。

### 7.6 字段类型选择参考

| 数据类型 | Thrift 类型 | 依据 |
|----------|-------------|------|
| ID/主键 | `i64` | 全工程统一 |
| 时间戳 | `i64`（毫秒） | banner.thrift: createAt/updateAt |
| 状态/枚举 | `i32` | banner.thrift: status/jumpType |
| 金额 | `string` | 金融场景推荐字符串传输，避免浮点精度问题 |
| 币种代码 | `string` | "VND"/"THB"/"BSB"等 |
| 订单号 | `string` | "C+16位"/"T+16位" |
| 页码/页数 | `i32` | 全工程统一 |
| 总记录数 | `i64` | PageBannerResp |
| 列表 | `list<T>` | 全工程统一 |

**金额为什么用 string 而不是 i64 或 double**：
- `double`：浮点精度问题，金融场景禁用
- `i64`：最小单位整数方案可行，但需要前后端约定精度换算规则
- `string`：字符串传输最安全，由接收方用 decimal 库解析，避免传输过程精度丢失
- **最终选择需评审确认**，IDL 层先用 string 预留最大兼容性

---

## 八、开发阶段与 IDL 的关系

### 8.1 IDL 不需要一次写完

```
Phase 1 — 工程搭建时的 IDL（最小可用集）
═══════════════════════════════════════════
必须定义：
  ├── service.thrift（至少包含 6 个核心 RPC 方法签名）
  ├── wallet.thrift（6 个核心 RPC 的 Req/Resp）
  └── currency.thrift（基本结构，可以只有空壳）

目的：gen_kitex.sh 能跑通 → 生成 handler.go 骨架 → 其他模块可以引用 kitex_gen

Phase 2 — 币种配置开发时补充
═══════════════════════════════
补充定义：
  └── currency.thrift 完整填充（B端 CRUD + RPC 供他方）

Phase 3 — 钱包核心开发时补充
═══════════════════════════════
补充定义：
  └── wallet.thrift 完整填充（C-1 + R-7/R-11/R-12）

Phase 4-6 — 业务功能开发时补充
═══════════════════════════════
按需补充：
  ├── deposit.thrift
  ├── withdraw.thrift
  ├── exchange.thrift
  └── record.thrift
```

**重要**：每次修改 IDL 后需要重新执行 gen_kitex.sh 重新生成代码。

### 8.2 Phase 1 最小 IDL 集

Phase 1 只需要让 gen_kitex.sh 能生成代码 + 财务模块能看到核心 RPC 接口定义。

最小集 = service.thrift（含 6 个核心 RPC 方法）+ wallet.thrift（6 个 Req/Resp）

其他域文件可以先创建空壳（只有 namespace 声明），后续按开发进度填充。

---

## 九、与现有工程的对比验证

### 9.1 规模对比

| 维度 | ser-app | ser-user | ser-live | **ser-wallet** |
|------|---------|----------|----------|----------------|
| 域文件数 | 4 | 13 | 8 | **6** |
| Service 方法数 | 21 | 34 | 45 | **37** |
| 文件总数 | 5 | 14 | 9 | **7** |
| 结构体总数（估） | ~60 | ~120 | ~100 | **~86** |

ser-wallet 的 IDL 规模与 ser-live 相当，处于工程中偏大的服务级别，符合其作为核心业务模块的定位。

### 9.2 命名规范对比

| 检查项 | 规范 | ser-wallet 设计 | 符合 |
|--------|------|----------------|------|
| namespace | `namespace go ser_{name}` | `namespace go ser_wallet` | Yes |
| Service 名 | `{Name}Service` | `WalletService` | Yes |
| 入口文件 | `service.thrift` | `service.thrift` | Yes |
| 方法签名 | `domain.XxxResp Method(1: domain.XxxReq req)` | 全部遵循 | Yes |
| 域文件前缀引用 | `domain.XxxReq` | `wallet.CreditWalletReq` | Yes |
| 注释 | `// 中文说明` | 全部方法有注释 | Yes |
| 分组注释 | `// =====` | 按维度分组 | Yes（参照 ser-live） |

### 9.3 gen_kitex.sh 兼容验证

```bash
# gen_kitex.sh 第31行匹配条件
find "$idl_dir" -maxdepth 1 -name "*service.thrift"

# 我方文件：common/idl/ser-wallet/service.thrift
# 匹配结果：✅ 匹配成功（文件名含 "service"）

# 生成命令：
kitex -module common/pkg \
      -I common/idl/ser-wallet \
      -service ser-wallet \
      -gen-path ./kitex_gen/ \
      common/idl/ser-wallet/service.thrift

# 生成结果：
# common/pkg/kitex_gen/ser_wallet/  ← 包含所有类型和客户端代码
# ser-wallet/handler.go             ← 37 个方法的空实现
# ser-wallet/main.go                ← 服务启动入口
```

---

## 十、不在 IDL 中定义的内容

### 10.1 定时任务（T-1 ~ T-3）

| 任务 | 说明 | 为什么不在 IDL 中 |
|------|------|------------------|
| T-1 ExchangeRateUpdate | 汇率定时更新 | 内部 cron 任务，不暴露为 RPC |
| T-2 DepositOrderTimeout | 充值订单超时 | 内部 cron 任务 |
| T-3 BalanceReconciliation | 余额对账 | 内部 cron 任务 |

### 10.2 RPC 调他方（O-1 ~ O-6）

| 调用 | 目标服务 | 说明 | 为什么不在我方 IDL 中 |
|------|---------|------|---------------------|
| O-1 | ser-kyc | QueryKycStatus | 在 ser-kyc 的 IDL 中已定义，我方通过 KycClient() 调用 |
| O-2 | ser-user | GetUserInfo | 在 ser-user 的 IDL 中已定义，我方通过 UserClient() 调用 |
| O-3~O-5 | 财务模块 | 支付方式/通道匹配/代付 | 在财务模块的 IDL 中定义，我方后续添加 FinanceClient() |
| O-6 | 活动模块 | 奖金活动查询 | 活动模块尚不存在，先预留接口结构 |

### 10.3 枚举值（在代码中定义，不在 IDL 中）

```
钱包类型：center=1, reward=2, anchor=3, agent=4, venue=5
变动类型：deposit=1, withdraw=2, exchange=3, reward=4, debit=5, ...
订单状态：pending=0, success=1, failed=2, expired=3, ...
币种类型：fiat=1, crypto=2, platform=3
```

**现有工程做法**：枚举值在 Go 代码的 `enum/` 目录中定义为 const，不在 Thrift IDL 中用 `enum` 关键字。
IDL 中状态/类型字段统一用 `i32`，具体值由业务代码控制。

---

## 十一、评审重点事项

### 11.1 IDL 评审优先级

| 优先级 | 内容 | 关注点 |
|--------|------|--------|
| **P0** | wallet.thrift 中 R-1~R-6 的 Req/Resp 字段 | 联调合同，双方必须对齐 |
| **P1** | 金额字段类型（string vs i64） | 影响全部金额传输 |
| **P1** | 钱包类型枚举值约定 | 影响所有 RPC 调用 |
| **P2** | currency.thrift 的 B端管理字段 | 影响后台功能开发 |
| **P3** | C端接口的 Req/Resp 细节 | 与前端对齐，可后续调整 |

### 11.2 需要财务模块确认的 IDL 字段

| 字段 | 归属 | 待确认内容 |
|------|------|----------|
| `walletType` | CreditWalletReq 等 | i32 枚举值定义 — 必须双方一致 |
| `opType` | CreditWalletReq / DebitWalletReq | 操作类型枚举 — 充值/补单/加款/返奖如何区分 |
| `orderNo` | 所有资金操作 Req | 订单号格式约定 — 谁生成，什么格式 |
| `amount` | 所有资金操作 Req | string 还是 i64？精度到几位？ |
| `auditMultiplier` | CreditWalletReq | 稽核倍数参数 — 是否由调用方传入 |
| `currency` | 所有 Req | 币种代码格式 — "VND" 还是 "vnd"？ISO 4217？ |

### 11.3 可能的结构调整场景

| 场景 | 影响 | 应对 |
|------|------|------|
| 评审新增接口 | service.thrift 加方法 + 对应域文件加结构体 | 按需添加，不影响已有定义 |
| 评审删减接口 | 删除对应方法和结构体 | 直接删除，重新 gen |
| 充值回调归属变化 | 如果充值回调归钱包，需要新增回调处理方法 | deposit.thrift 加 DepositCallbackReq/Resp |
| 提现审核操作移入钱包 | 需要新增审核相关方法和结构体 | withdraw.thrift 扩展 |
| 合并 Could 接口到 Must 接口 | 如 C-19 合入 C-1，减少方法数 | 调整 Resp 结构体包含更多字段 |

---

## 十二、文件创建检查清单

执行 gen_kitex.sh 前，确认以下文件都已就位：

```
✅ 检查清单

[ ] common/idl/ser-wallet/ 目录已创建
[ ] common/idl/ser-wallet/service.thrift — 已定义 WalletService + 所有方法
[ ] common/idl/ser-wallet/wallet.thrift — 已定义核心 Req/Resp
[ ] common/idl/ser-wallet/currency.thrift — 已定义币种相关 Req/Resp
[ ] common/idl/ser-wallet/deposit.thrift — 已定义充值相关 Req/Resp
[ ] common/idl/ser-wallet/withdraw.thrift — 已定义提现相关 Req/Resp
[ ] common/idl/ser-wallet/exchange.thrift — 已定义兑换相关 Req/Resp
[ ] common/idl/ser-wallet/record.thrift — 已定义记录相关 Req/Resp
[ ] 所有文件 namespace 统一为 ser_wallet
[ ] 所有 include 路径正确
[ ] service.thrift 中所有方法的 domain 前缀与文件名一致
```

执行 gen_kitex.sh 后，确认以下生成结果：

```
[ ] common/pkg/kitex_gen/ser_wallet/ 目录已生成
[ ] ser-wallet/handler.go 已生成（包含 37 个方法的空实现）
[ ] ser-wallet/main.go 已生成
[ ] 编译通过：cd ser-wallet && go build ./...
```

---

## 十三、总结

### 13.1 设计概览

```
common/idl/ser-wallet/（7 个文件）
│
├── service.thrift ─── WalletService（37 个方法）
│       │
│       ├── include wallet.thrift ──── 钱包核心（~22 个结构体）
│       │     └── R-1~R-6 联调合同 + R-7/R-11/R-12 + C-1
│       │
│       ├── include currency.thrift ── 币种配置（~26 个结构体）
│       │     └── B-1~B-6 管理 + R-8~R-10 供他方 + C-13/C-19
│       │
│       ├── include deposit.thrift ─── 充值（~16 个结构体）
│       │     └── C-2~C-4/C-14~C-16/C-18
│       │
│       ├── include withdraw.thrift ── 提现（~12 个结构体）
│       │     └── C-7~C-10/C-17
│       │
│       ├── include exchange.thrift ── 兑换（~4 个结构体）
│       │     └── C-5~C-6
│       │
│       └── include record.thrift ──── 账变记录（~6 个结构体）
│             └── C-11~C-12
│
总计：37 个方法，~86 个结构体，6 个域文件
```

### 13.2 确定性骨架

1. **7 个文件**：1 service + 6 domain，粒度与 ser-live（9文件/45方法）一致
2. **37 个方法**：覆盖接口评估 46 个接口中需要 IDL 定义的全部（排除调他方 6 + 定时 3）
3. **namespace 统一**：`ser_wallet`，遵循全工程规范
4. **gen_kitex.sh 兼容**：service.thrift 命名满足 `*service.thrift` 匹配
5. **6 个核心 RPC 是联调合同**：wallet.thrift 中 R-1~R-6 需要与财务模块双方对齐
6. **渐进式填充**：Phase 1 只需最小 IDL 集，后续按开发阶段逐步补充字段

### 13.3 弹性空间

1. 具体字段定义（评审后填充）
2. 金额类型（string vs i64，评审确认）
3. 枚举值约定（walletType / opType / status，双方对齐）
4. Could 接口可能合并或删除
5. 评审可能新增接口（如充值回调归属变化）
6. 是否引用 common/enum.thrift（视具体枚举需求）
