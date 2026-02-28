# ser-wallet IDL 设计方案

> 设计时间：2026-02-27
> 核心问题：钱包/币种/财务三大模块从0到1实现，IDL先行——怎么设计、设计多少文件、怎么组织方法
> 设计原则：符合现有工程规范、有依据、不过度死板、屏蔽不确定细节但骨架清晰
> 信息依据：
> - 工程认知：/Users/mac/gitlab/z-readme/工程认知/tmp-1.md（IDL规范 + gen_kitex.sh + 现有19服务模式分析）
> - 需求理解：/Users/mac/gitlab/z-readme/需求理解/tmp-1.md（三模块功能全景）
> - 架构设计：/Users/mac/gitlab/z-readme/架构设计/tmp-1.md（七大ADR决策 + 服务分层 + RPC接口清单）
> - 实现思路：/Users/mac/gitlab/z-readme/实现思路/tmp-1.md（七阶段实现 + IDL编写要点）
> - 表结构设计：/Users/mac/gitlab/z-readme/表结构设计/tmp-1.md（9必须表 + 14枚举）
> - 数据模型：/Users/mac/gitlab/z-readme/数据模型/tmp-1.md（完整字段 + 建表SQL）
> - 现有工程IDL探索：全工程71个thrift文件、19个服务模块的实际代码模式

---

## 目录

1. [现有工程IDL模式分析（设计依据）](#一现有工程idl模式分析)
2. [ser-wallet IDL设计决策](#二ser-wallet-idl设计决策)
3. [文件结构总览](#三文件结构总览)
4. [service.thrift — 主入口文件](#四servicethrift-主入口文件)
5. [currency.thrift — 币种配置域](#五currencythrift-币种配置域)
6. [wallet.thrift — 钱包核心域](#六walletthrift-钱包核心域)
7. [deposit.thrift — 充值域](#七depositthrift-充值域)
8. [withdraw.thrift — 提现域](#八withdrawthrift-提现域)
9. [exchange.thrift — 兑换域](#九exchangethrift-兑换域)
10. [record.thrift — 记录域](#十recordthrift-记录域)
11. [wallet_rpc.thrift — 跨服务RPC域](#十一wallet_rpcthrift-跨服务rpc域)
12. [方法全景清单](#十二方法全景清单)
13. [IDL设计约定速查](#十三idl设计约定速查)
14. [金额字段类型决策](#十四金额字段类型决策)
15. [与现有服务的include关系](#十五与现有服务的include关系)
16. [代码生成与验证](#十六代码生成与验证)
17. [设计弹性空间](#十七设计弹性空间)
18. [IDL设计决策记录](#十八idl设计决策记录)

---

## 一、现有工程IDL模式分析

> 不拍脑袋设计，先看工程里已有的19个服务是怎么做的

### 1.1 工程IDL全局数据

通过代码扫描 `/Users/mac/gitlab/common/idl/` 目录下的全部Thrift文件，得到以下事实数据：

| 指标 | 数值 |
|------|------|
| 总 .thrift 文件数 | 71 个 |
| 服务目录数（每目录一个service） | 19 个 |
| 平均每服务 thrift 文件数 | ~3.7 个 |
| 最少文件数 | 1 个（ser-ip，仅 service.thrift） |
| 最多文件数 | 13 个（ser-user） |

### 1.2 各服务的文件拆分情况

| 服务 | .thrift 文件数 | 方法数 | 拆分策略 |
|------|---------------|--------|---------|
| ser-user | 13 | 20+ | 按功能域细拆（users/rpc/kyc/follow/blacklist/login_log/...） |
| ser-live | 9 | 37 | 按功能域拆（room/consume/audience/audit/api/rpc） |
| ser-app | 5 | 25+ | 按资源类型拆（banner/s3/home_category/content_strategy） |
| ser-kyc | 4 | 9 | 按流程拆（kyc/s3/audit） |
| ser-mall | 4 | 10+ | 按模块拆（mall/order/s3） |
| ser-item | 4 | 8 | 按模块拆（item/publish/s3） |
| ser-exp | 3 | 10+ | 较简单，按配置/用户拆 |
| ser-blog | 2 | 5+ | service + 数据结构 |
| ser-s3 | 2 | 12 | service + 数据结构 |
| ser-ip | 1 | 5 | 极简，全放一个文件 |

### 1.3 核心模式归纳

**模式一：一个service定义，多个struct文件**

所有19个服务无一例外遵循此模式：

```
common/idl/ser-xxx/
├── service.thrift          ← 定义 service XxxService { ... }
├── feature_a.thrift        ← 定义 feature_a 相关的 struct/enum
├── feature_b.thrift        ← 定义 feature_b 相关的 struct/enum
└── xxx_rpc.thrift          ← 定义供其他服务RPC调用的 struct（如果有）
```

- `service.thrift` 是唯一入口，include 其他文件
- 每个服务只有**一个** service 定义（不拆多个service）
- struct/enum 按**功能域**拆分到不同 .thrift 文件

**模式二：命名规范统一**

```thrift
namespace go ser_xxx                    // 蛇形命名
service XxxService { ... }              // PascalCase + Service后缀
MethodNameReq / MethodNameResp          // 方法名 + Req/Resp
```

**模式三：struct字段注解**

```thrift
struct SomeReq {
    1: required string name    (api.body="name")       // 必填
    2: optional i32 status     (api.body="status")     // 可选
    3: i32 pageNo = 1         (api.body="pageNo")     // 带默认值
    4: i32 pageSize = 10      (api.body="pageSize")   // 带默认值
}
```

**模式四：分页标准格式**

```thrift
// 请求
struct PageXxxReq {
    // ...筛选条件
    N: i32 pageNo = 1   (api.body="pageNo")
    N+1: i32 pageSize = 10 (api.body="pageSize")
}

// 响应
struct PageXxxResp {
    1: list<XxxItem> list       (api.body="list")
    2: i64 total                (api.body="total")
    3: i32 pageNo               (api.body="pageNo")
    4: i32 pageSize             (api.body="pageSize")
}
```

**模式五：_rpc.thrift 用于跨服务调用**

ser-user 和 ser-live 都有独立的 `_rpc.thrift`，专门定义供其他服务 RPC 调用的 struct：

```thrift
// ser-user/user_rpc.thrift — 供其他服务调用的用户查询
struct BatchGetUserInfoReq { ... }
struct BatchUserInfoResp { ... }
struct SearchUserReq { ... }
struct SearchUserResp { ... }
```

这种拆分的好处：把"对内接口"（B/C端）和"对外接口"（供其他服务RPC调用）的数据结构分开管理，职责清晰。

### 1.4 数据类型使用惯例

通过扫描现有IDL的实际用法：

| 语义 | 实际使用类型 | 示例 |
|------|-------------|------|
| 业务ID | `i64` | userId, sessionId, orderId |
| 枚举/状态 | `i32` | status, type, level |
| 时间戳 | `i64` | createAt, updateAt（Unix秒） |
| 金额（现有） | `i64` | totalIncome（直播收入，按最小单位存） |
| 文本 | `string` | name, remark, code |
| 布尔 | `bool` | isSuccess（少见） |
| 列表 | `list<T>` | list<BannerItem> |
| 键值对 | `map<K,V>` | map<i64, UserInfo>（少见） |

**重要发现**：现有工程中金额字段用 `i64`（最小单位），但我们的架构设计ADR-06明确选择了 `string` + `DECIMAL(30,8)` 方案。这是有意识的差异，原因见第十四章。

---

## 二、ser-wallet IDL设计决策

### 2.1 几个文件？

**决策：7个 .thrift 文件**

```
service.thrift       1个主入口
currency.thrift      1个币种配置域
wallet.thrift        1个钱包核心域
deposit.thrift       1个充值域
withdraw.thrift      1个提现域
exchange.thrift      1个兑换域
record.thrift        1个记录域
wallet_rpc.thrift    1个跨服务RPC域（供finance/其他服务调用）
```

总计 **8个文件**。

### 2.2 为什么是8个

**依据一：业务域天然分界**

钱包系统有6个明确的业务域（币种、钱包核心、充值、提现、兑换、记录），再加上对外RPC接口和主入口，正好8个。这不是拍脑袋定的数量，而是业务结构决定的。

**依据二：与工程规模匹配**

- ser-wallet 预计30个方法（与 ser-live 的37个和 ser-app 的25个同量级）
- ser-live 用了9个文件管理37个方法
- ser-app 用了5个文件管理25个方法
- ser-wallet 8个文件管理30个方法，密度合理

**依据三：不过度拆分**

如果只有2-3个struct的域（如兑换只有PreviewExchange和CreateExchange两个方法的struct），仍然独立一个文件，因为：
- 逻辑边界清晰，维护时能快速定位
- 评审后可能增加方法/struct，预留空间不亏

### 2.3 一个service还是多个service？

**决策：一个 `WalletService`**

与工程中所有19个服务保持一致——每个 `ser-xxx` 只有一个 service 定义。不拆多service的原因：

- Kitex 代码生成以 service 为单位，一个 service 生成一套 client/server
- `rpc.WalletClient()` 工厂函数返回一个 client，对调用方最简单
- 多 service 需要多个 client 实例和端口，增加运维复杂度
- 通过文件拆分 struct 已经足够清晰，不需要拆 service 来组织

### 2.4 struct拆到哪个文件的原则

```
原则：struct 跟着它最紧密相关的业务域走

币种的 struct → currency.thrift
充值的 struct → deposit.thrift
供 finance 调用的 struct → wallet_rpc.thrift

当一个 struct 被多个域使用时：
  → 放在它"产生"的那个域中
  → 其他域通过 include 引用
```

---

## 三、文件结构总览

```
/Users/mac/gitlab/common/idl/ser-wallet/
│
├── service.thrift              # 主入口
│   namespace go ser_wallet
│   include "currency.thrift"
│   include "wallet.thrift"
│   include "deposit.thrift"
│   include "withdraw.thrift"
│   include "exchange.thrift"
│   include "record.thrift"
│   include "wallet_rpc.thrift"
│
│   service WalletService {
│       // --- B端：币种配置 (7个方法) ---
│       // --- C端：钱包操作 (12个方法) ---
│       // --- C端：记录查询 (2个方法) ---
│       // --- RPC：供finance等服务调用 (9个方法) ---
│   }
│
├── currency.thrift             # 币种配置域 struct
│   namespace go ser_wallet
│   // CurrencyItem, CreateCurrencyReq/Resp, UpdateCurrencyReq/Resp,
│   // PageCurrencyReq/Resp, ToggleCurrencyStatusReq/Resp,
│   // SetBaseCurrencyReq/Resp, DetailCurrencyReq/Resp,
│   // ListActiveCurrencyResp, GetExchangeRateReq/Resp
│   // RateLogItem, PageRateLogReq/Resp
│
├── wallet.thrift               # 钱包核心域 struct
│   namespace go ser_wallet
│   // WalletOverviewReq/Resp, SubWalletInfo
│
├── deposit.thrift              # 充值域 struct
│   namespace go ser_wallet
│   // GetDepositConfigReq/Resp, CreateDepositOrderReq/Resp,
│   // GetDepositStatusReq/Resp, DepositConfigItem, DepositOrderItem
│
├── withdraw.thrift             # 提现域 struct
│   namespace go ser_wallet
│   // GetWithdrawConfigReq/Resp, GetWithdrawAccountReq/Resp,
│   // SaveWithdrawAccountReq/Resp, CreateWithdrawOrderReq/Resp,
│   // WithdrawAccountInfo, WithdrawConfigItem, WithdrawOrderItem
│
├── exchange.thrift             # 兑换域 struct
│   namespace go ser_wallet
│   // PreviewExchangeReq/Resp, CreateExchangeReq/Resp
│
├── record.thrift               # 记录域 struct
│   namespace go ser_wallet
│   // PageRecordReq/Resp, DetailRecordReq/Resp,
│   // RecordItem, RecordDetail
│
└── wallet_rpc.thrift           # 跨服务RPC域 struct
    namespace go ser_wallet
    // WalletCreditReq/Resp, WalletCreditRewardReq/Resp,
    // WalletFreezeReq/Resp, WalletUnfreezeReq/Resp,
    // WalletDeductReq/Resp, WalletManualAddReq/Resp,
    // WalletManualSubReq/Resp, GetUserBalanceReq/Resp,
    // GetCurrencyConfigReq/Resp
    // UserBalanceInfo, SubWalletBalance, CurrencyConfigInfo
```

---

## 四、service.thrift — 主入口文件

### 4.1 文件定位

这是 Kitex 代码生成的入口文件，定义 `WalletService` 的全部方法签名。所有 struct 通过 include 引入，本文件只有 namespace + include + service 块。

### 4.2 完整设计

```thrift
namespace go ser_wallet

include "currency.thrift"
include "wallet.thrift"
include "deposit.thrift"
include "withdraw.thrift"
include "exchange.thrift"
include "record.thrift"
include "wallet_rpc.thrift"

service WalletService {

    // ========== B端：币种配置管理 ==========

    // 币种分页列表（B端后台）
    currency.PageCurrencyResp PageCurrency(1: currency.PageCurrencyReq req)

    // 创建币种
    currency.CreateCurrencyResp CreateCurrency(1: currency.CreateCurrencyReq req)

    // 编辑币种
    currency.UpdateCurrencyResp UpdateCurrency(1: currency.UpdateCurrencyReq req)

    // 启用/禁用币种
    currency.ToggleCurrencyStatusResp ToggleCurrencyStatus(1: currency.ToggleCurrencyStatusReq req)

    // 设置基准币种（一次性操作）
    currency.SetBaseCurrencyResp SetBaseCurrency(1: currency.SetBaseCurrencyReq req)

    // 币种详情
    currency.DetailCurrencyResp DetailCurrency(1: currency.DetailCurrencyReq req)

    // 汇率日志分页
    currency.PageRateLogResp PageRateLog(1: currency.PageRateLogReq req)


    // ========== C端：钱包操作 ==========

    // 可用币种列表（C端用户）
    currency.ListActiveCurrencyResp ListActiveCurrency(1: currency.ListActiveCurrencyReq req)

    // 获取汇率信息
    currency.GetExchangeRateResp GetExchangeRate(1: currency.GetExchangeRateReq req)

    // 钱包余额概览
    wallet.GetWalletOverviewResp GetWalletOverview(1: wallet.GetWalletOverviewReq req)

    // 充值配置（可用方式+汇率+快捷金额）
    deposit.GetDepositConfigResp GetDepositConfig(1: deposit.GetDepositConfigReq req)

    // 创建充值订单
    deposit.CreateDepositOrderResp CreateDepositOrder(1: deposit.CreateDepositOrderReq req)

    // 充值状态查询（C端轮询）
    deposit.GetDepositStatusResp GetDepositStatus(1: deposit.GetDepositStatusReq req)

    // 兑换预览（试算）
    exchange.PreviewExchangeResp PreviewExchange(1: exchange.PreviewExchangeReq req)

    // 执行兑换
    exchange.CreateExchangeResp CreateExchange(1: exchange.CreateExchangeReq req)

    // 提现配置（KYC状态+可用方式+限额）
    withdraw.GetWithdrawConfigResp GetWithdrawConfig(1: withdraw.GetWithdrawConfigReq req)

    // 查询提现账户
    withdraw.GetWithdrawAccountResp GetWithdrawAccount(1: withdraw.GetWithdrawAccountReq req)

    // 保存提现账户（首次绑定）
    withdraw.SaveWithdrawAccountResp SaveWithdrawAccount(1: withdraw.SaveWithdrawAccountReq req)

    // 创建提现订单
    withdraw.CreateWithdrawOrderResp CreateWithdrawOrder(1: withdraw.CreateWithdrawOrderReq req)


    // ========== C端：记录查询 ==========

    // 记录分页（充值/提现/兑换/奖励 四Tab）
    record.PageRecordResp PageRecord(1: record.PageRecordReq req)

    // 记录详情
    record.DetailRecordResp DetailRecord(1: record.DetailRecordReq req)


    // ========== RPC：供 ser-finance 等服务调用 ==========

    // 入账到中心钱包（充值成功/补单通过）
    wallet_rpc.WalletCreditResp WalletCredit(1: wallet_rpc.WalletCreditReq req)

    // 奖金入账到奖励钱包
    wallet_rpc.WalletCreditRewardResp WalletCreditReward(1: wallet_rpc.WalletCreditRewardReq req)

    // 冻结（提现发起）
    wallet_rpc.WalletFreezeResp WalletFreeze(1: wallet_rpc.WalletFreezeReq req)

    // 解冻（审核驳回/出款失败）
    wallet_rpc.WalletUnfreezeResp WalletUnfreeze(1: wallet_rpc.WalletUnfreezeReq req)

    // 扣款（出款成功最终扣除冻结金额）
    wallet_rpc.WalletDeductResp WalletDeduct(1: wallet_rpc.WalletDeductReq req)

    // 人工加款（按子钱包分别加）
    wallet_rpc.WalletManualAddResp WalletManualAdd(1: wallet_rpc.WalletManualAddReq req)

    // 人工减款（按子钱包分别减）
    wallet_rpc.WalletManualSubResp WalletManualSub(1: wallet_rpc.WalletManualSubReq req)

    // 查询用户各子钱包余额
    wallet_rpc.GetUserBalanceResp GetUserBalance(1: wallet_rpc.GetUserBalanceReq req)

    // 获取币种配置信息
    wallet_rpc.GetCurrencyConfigResp GetCurrencyConfig(1: wallet_rpc.GetCurrencyConfigReq req)
}
```

### 4.3 方法分组说明

| 分组 | 方法数 | 调用方 | 经过的网关 |
|------|--------|--------|----------|
| B端币种配置 | 7 | gate-back → ser-wallet | gate-back |
| C端钱包操作 | 12 | gate-font → ser-wallet | gate-font |
| C端记录查询 | 2 | gate-font → ser-wallet | gate-font |
| RPC跨服务 | 9 | ser-finance → ser-wallet | 无（直接RPC） |
| **合计** | **30** | | |

30个方法在工程中属于中上规模，与 ser-live（37个）和 ser-app（25个）同量级，合理。

---

## 五、currency.thrift — 币种配置域

### 5.1 文件定位

承载币种配置相关的所有请求/响应结构体——B端CRUD、C端查询、汇率日志。

### 5.2 设计

```thrift
namespace go ser_wallet

// ===================== 公共数据结构 =====================

// 币种信息项（B端列表展示用）
struct CurrencyItem {
    1:  i64 id                      (api.body="id")
    2:  string currencyCode         (api.body="currencyCode")         // 币种代码 VND/USDT/BSB
    3:  string currencyName         (api.body="currencyName")         // 币种名称
    4:  i32 currencyType            (api.body="currencyType")         // 1法币 2加密 3平台
    5:  string country              (api.body="country")              // 所属国家
    6:  string iconUrl              (api.body="iconUrl")              // 图标URL
    7:  string currencySymbol       (api.body="currencySymbol")       // 货币符号 ₫ Rp ฿
    8:  string thousandsSep         (api.body="thousandsSep")         // 千分位符号
    9:  string decimalSep           (api.body="decimalSep")          // 小数点符号
    10: i32 precision               (api.body="precision")            // 小数位数
    11: string realRate             (api.body="realRate")             // 实时汇率
    12: string platformRate         (api.body="platformRate")         // 平台汇率
    13: string rateFluctuation      (api.body="rateFluctuation")     // 汇率浮动(%)
    14: string rateThreshold        (api.body="rateThreshold")       // 偏差阈值(%)
    15: string depositRate          (api.body="depositRate")          // 入款汇率
    16: string withdrawRate         (api.body="withdrawRate")         // 出款汇率
    17: i32 isBase                  (api.body="isBase")               // 是否基准币种 0否 1是
    18: i32 sortValue               (api.body="sortValue")            // 排序值
    19: i32 status                  (api.body="status")               // 1启用 2禁用
    20: i64 createBy                (api.body="createBy")             // 创建人
    21: string createByName         (api.body="createByName")         // 创建人姓名
    22: i64 createAt                (api.body="createAt")             // 创建时间
    23: i64 updateAt                (api.body="updateAt")             // 更新时间
}

// C端币种简要信息
struct CurrencyBriefItem {
    1: string currencyCode          (api.body="currencyCode")
    2: string currencyName          (api.body="currencyName")
    3: i32 currencyType             (api.body="currencyType")
    4: string iconUrl               (api.body="iconUrl")
    5: string currencySymbol        (api.body="currencySymbol")
    6: string thousandsSep          (api.body="thousandsSep")
    7: string decimalSep            (api.body="decimalSep")
    8: i32 precision                (api.body="precision")
    9: i32 sortValue                (api.body="sortValue")
}

// 汇率日志项
struct RateLogItem {
    1:  i64 id                      (api.body="id")
    2:  string currencyCode         (api.body="currencyCode")
    3:  string realRateBefore       (api.body="realRateBefore")      // 更新前实时汇率
    4:  string realRateAfter        (api.body="realRateAfter")       // 更新后实时汇率
    5:  string platformRateBefore   (api.body="platformRateBefore")  // 更新前平台汇率
    6:  string platformRateAfter    (api.body="platformRateAfter")   // 更新后平台汇率
    7:  string depositRate          (api.body="depositRate")          // 更新后入款汇率
    8:  string withdrawRate         (api.body="withdrawRate")         // 更新后出款汇率
    9:  string deviationPct         (api.body="deviationPct")        // 触发偏差百分比
    10: i32 triggerType             (api.body="triggerType")          // 1自动 2手动
    11: i64 createAt                (api.body="createAt")
}


// ===================== B端：币种 CRUD =====================

// 创建币种
struct CreateCurrencyReq {
    1:  required string currencyCode    (api.body="currencyCode")
    2:  required string currencyName    (api.body="currencyName")
    3:  required i32 currencyType       (api.body="currencyType")    // 1法币 2加密 3平台
    4:  optional string country         (api.body="country")
    5:  optional string iconUrl         (api.body="iconUrl")
    6:  required string currencySymbol  (api.body="currencySymbol")
    7:  required string thousandsSep    (api.body="thousandsSep")
    8:  required string decimalSep      (api.body="decimalSep")
    9:  required i32 precision          (api.body="precision")
    10: required string rateFluctuation (api.body="rateFluctuation")
    11: required string rateThreshold   (api.body="rateThreshold")
    12: required i32 sortValue          (api.body="sortValue")
    13: required i32 status             (api.body="status")
}

struct CreateCurrencyResp {
    1: i64 id                           (api.body="id")
}

// 编辑币种
struct UpdateCurrencyReq {
    1:  required i64 id                 (api.body="id")
    2:  optional string currencyName    (api.body="currencyName")
    3:  optional string country         (api.body="country")
    4:  optional string iconUrl         (api.body="iconUrl")
    5:  optional string currencySymbol  (api.body="currencySymbol")
    6:  optional string thousandsSep    (api.body="thousandsSep")
    7:  optional string decimalSep      (api.body="decimalSep")
    8:  optional i32 precision          (api.body="precision")
    9:  optional string rateFluctuation (api.body="rateFluctuation")
    10: optional string rateThreshold   (api.body="rateThreshold")
    11: optional i32 sortValue          (api.body="sortValue")
}

struct UpdateCurrencyResp {
}

// 启禁用
struct ToggleCurrencyStatusReq {
    1: required i64 id                  (api.body="id")
    2: required i32 status              (api.body="status")          // 1启用 2禁用
}

struct ToggleCurrencyStatusResp {
}

// 设置基准币种
struct SetBaseCurrencyReq {
    1: required string currencyCode     (api.body="currencyCode")    // 只能设一次
}

struct SetBaseCurrencyResp {
}

// 币种详情
struct DetailCurrencyReq {
    1: required i64 id                  (api.body="id")
}

struct DetailCurrencyResp {
    1: CurrencyItem data                (api.body="data")
}

// 币种分页（B端）
struct PageCurrencyReq {
    1: optional string currencyCode     (api.body="currencyCode")
    2: optional string currencyName     (api.body="currencyName")
    3: optional i32 currencyType        (api.body="currencyType")
    4: optional i32 status              (api.body="status")
    5: i32 pageNo = 1                   (api.body="pageNo")
    6: i32 pageSize = 10                (api.body="pageSize")
}

struct PageCurrencyResp {
    1: list<CurrencyItem> list          (api.body="list")
    2: i64 total                        (api.body="total")
    3: i32 pageNo                       (api.body="pageNo")
    4: i32 pageSize                     (api.body="pageSize")
}


// ===================== 汇率日志 =====================

struct PageRateLogReq {
    1: optional string currencyCode     (api.body="currencyCode")
    2: optional i64 startTime           (api.body="startTime")
    3: optional i64 endTime             (api.body="endTime")
    4: i32 pageNo = 1                   (api.body="pageNo")
    5: i32 pageSize = 10                (api.body="pageSize")
}

struct PageRateLogResp {
    1: list<RateLogItem> list           (api.body="list")
    2: i64 total                        (api.body="total")
    3: i32 pageNo                       (api.body="pageNo")
    4: i32 pageSize                     (api.body="pageSize")
}


// ===================== C端：币种查询 =====================

// 可用币种列表
struct ListActiveCurrencyReq {
}

struct ListActiveCurrencyResp {
    1: list<CurrencyBriefItem> list     (api.body="list")
}

// 获取汇率
struct GetExchangeRateReq {
    1: required string sourceCurrency   (api.body="sourceCurrency")  // 源币种
    2: required string targetCurrency   (api.body="targetCurrency")  // 目标币种
}

struct GetExchangeRateResp {
    1: string exchangeRate              (api.body="exchangeRate")    // 汇率
    2: string depositRate               (api.body="depositRate")     // 入款汇率
    3: string withdrawRate              (api.body="withdrawRate")    // 出款汇率
}
```

### 5.3 为什么CurrencyItem和CurrencyBriefItem分开

B端需要看到完整20+字段（汇率、阈值、操作人等管理字段），C端只需要展示用的7-8个字段。一个Item塞所有字段会导致C端接口携带大量无用数据，且C端不该看到阈值、浮动率等运营参数。

---

## 六、wallet.thrift — 钱包核心域

### 6.1 文件定位

钱包余额概览——C端用户看到的"我有多少钱"。

### 6.2 设计

```thrift
namespace go ser_wallet

// 子钱包余额信息
struct SubWalletInfo {
    1: i32 walletType               (api.body="walletType")         // 1中心 2奖励 3主播
    2: string walletTypeName        (api.body="walletTypeName")     // 中心钱包/奖励钱包/主播钱包
    3: string availableBalance      (api.body="availableBalance")   // 可用余额
    4: string frozenBalance         (api.body="frozenBalance")      // 冻结余额
}

// 钱包概览请求
struct GetWalletOverviewReq {
    1: required string currencyCode (api.body="currencyCode")       // 当前选中的币种
}

// 钱包概览响应
struct GetWalletOverviewResp {
    1: string currencyCode              (api.body="currencyCode")
    2: string totalBalance              (api.body="totalBalance")       // 资金钱包余额=中心+奖励
    3: string totalFrozen               (api.body="totalFrozen")        // 总冻结金额
    4: list<SubWalletInfo> subWallets   (api.body="subWallets")         // 各子钱包明细
    5: string currencySymbol            (api.body="currencySymbol")     // 货币符号
    6: i32 precision                    (api.body="precision")          // 精度
}
```

### 6.3 为什么 wallet.thrift 这么小

钱包核心域的"重头戏"是六种原子操作（credit/debit/freeze/unfreeze/deduct/debitSafe），但这些是服务内部方法，不走IDL。IDL中暴露的是：

- C端余额概览 → wallet.thrift（本文件）
- RPC供finance调用的资金操作 → wallet_rpc.thrift（独立文件）

所以 wallet.thrift 只管"查余额"这一件事，是合理的精简。

---

## 七、deposit.thrift — 充值域

### 7.1 文件定位

充值业务的全部请求/响应——充值配置、创建订单、状态查询。

### 7.2 设计

```thrift
namespace go ser_wallet

// ===================== 公共数据结构 =====================

// 充值方式配置项
struct DepositMethodItem {
    1: i32 depositMethod            (api.body="depositMethod")      // 1USDT 2银行 3电子钱包
    2: string methodName            (api.body="methodName")         // 方式名称
    3: string minAmount             (api.body="minAmount")          // 单笔最低
    4: string maxAmount             (api.body="maxAmount")          // 单笔最高
    5: i32 sortValue                (api.body="sortValue")
    6: list<string> quickAmounts    (api.body="quickAmounts")       // 快捷金额档位
}

// 额外奖金活动项
struct BonusActivityItem {
    1: i64 activityId               (api.body="activityId")
    2: string activityName          (api.body="activityName")
    3: string bonusRatio            (api.body="bonusRatio")         // 奖金比例(%)
    4: string minDepositAmount      (api.body="minDepositAmount")   // 最低充值门槛
    5: string turnoverMulti         (api.body="turnoverMulti")      // 流水倍数
}

// 充值订单项（记录列表用）
struct DepositOrderItem {
    1: string orderNo               (api.body="orderNo")
    2: string amount                (api.body="amount")
    3: i32 depositMethod            (api.body="depositMethod")
    4: string methodName            (api.body="methodName")
    5: i32 status                   (api.body="status")             // 1待支付 2处理中 3完成 4失败
    6: string statusText            (api.body="statusText")
    7: i64 createAt                 (api.body="createAt")
}


// ===================== 充值配置 =====================

struct GetDepositConfigReq {
    1: required string currencyCode (api.body="currencyCode")
}

struct GetDepositConfigResp {
    1: string currencyCode              (api.body="currencyCode")
    2: list<DepositMethodItem> methods  (api.body="methods")        // 可用充值方式
    3: list<BonusActivityItem> bonuses  (api.body="bonuses")        // 可用奖金活动
    4: string exchangeRate              (api.body="exchangeRate")    // 当前汇率（BSB场景用）
    5: string currencySymbol            (api.body="currencySymbol")
}


// ===================== 创建充值订单 =====================

struct CreateDepositOrderReq {
    1: required string currencyCode     (api.body="currencyCode")
    2: required i32 depositMethod       (api.body="depositMethod")  // 1USDT 2银行 3电子钱包
    3: required string amount           (api.body="amount")         // 充值金额
    4: optional i64 bonusActivityId     (api.body="bonusActivityId") // 奖金活动ID（0=无）
    5: optional i32 networkType         (api.body="networkType")    // USDT网络 1TRC20 2ERC20 3BEP20
}

struct CreateDepositOrderResp {
    1: string orderNo                   (api.body="orderNo")
    2: i32 interceptType                (api.body="interceptType")  // 0正常 1软拦截 2硬拦截
    3: i32 interceptCount               (api.body="interceptCount") // 已有同类订单数
    4: string paymentUrl                (api.body="paymentUrl")     // 三方支付页URL（银行/电子钱包）
    5: string depositAddress            (api.body="depositAddress") // USDT收款地址
    6: string qrCode                    (api.body="qrCode")        // 二维码内容
    7: string usdtAmount                (api.body="usdtAmount")    // 换算后USDT金额
    8: string exchangeRate              (api.body="exchangeRate")   // 使用的汇率快照
    9: string bonusAmount               (api.body="bonusAmount")   // 奖金金额
}


// ===================== 充值状态查询 =====================

struct GetDepositStatusReq {
    1: required string orderNo          (api.body="orderNo")
}

struct GetDepositStatusResp {
    1: string orderNo                   (api.body="orderNo")
    2: i32 status                       (api.body="status")
    3: string statusText                (api.body="statusText")
    4: string amount                    (api.body="amount")
    5: string actualAmount              (api.body="actualAmount")
    6: i32 depositMethod                (api.body="depositMethod")
    7: string methodName                (api.body="methodName")
    8: i64 createAt                     (api.body="createAt")
    9: i64 paidAt                       (api.body="paidAt")
    10: string failReason               (api.body="failReason")
}
```

### 7.3 关于重复转账拦截

`CreateDepositOrderResp` 中的 `interceptType` 字段（0正常/1软拦截/2硬拦截）是需求文档4.4的直接映射。当 `interceptType=2` 时，前端展示阻断弹窗禁止继续；`interceptType=1` 时前端弹窗警告但允许继续。

拦截逻辑在 Service 层实现（查同用户+同币种+同金额+状态∈(1,2)的订单数），IDL只需传递判断结果。

---

## 八、withdraw.thrift — 提现域

### 8.1 文件定位

提现业务的全部请求/响应——提现配置、提现账户管理、创建提现订单。

### 8.2 设计

```thrift
namespace go ser_wallet

// ===================== 公共数据结构 =====================

// 提现方式配置项
struct WithdrawMethodItem {
    1: i32 withdrawMethod           (api.body="withdrawMethod")     // 1银行卡 2电子钱包 3USDT
    2: string methodName            (api.body="methodName")
    3: string minAmount             (api.body="minAmount")          // 单笔最低
    4: string maxAmount             (api.body="maxAmount")          // 单笔最高
    5: string dailyLimit            (api.body="dailyLimit")         // 单日限额
    6: string feeRate               (api.body="feeRate")            // 手续费率(%)
    7: string arrivalTime           (api.body="arrivalTime")        // 到账时间描述
}

// 提现账户信息
struct WithdrawAccountInfo {
    1:  i64 id                      (api.body="id")
    2:  i32 accountType             (api.body="accountType")        // 1银行卡 2电子钱包 3USDT
    3:  string currencyCode         (api.body="currencyCode")
    4:  string bankName             (api.body="bankName")           // 银行名称
    5:  string bankAccount          (api.body="bankAccount")        // 银行账号（脱敏）
    6:  string bankBranch           (api.body="bankBranch")         // 开户网点
    7:  string accountHolder        (api.body="accountHolder")      // 账户持有人
    8:  string eWalletType          (api.body="eWalletType")        // 电子钱包类型
    9:  string eWalletPhone         (api.body="eWalletPhone")       // 电子钱包手机号（脱敏）
    10: string chainAddress         (api.body="chainAddress")       // USDT地址（脱敏）
    11: i32 chainNetwork            (api.body="chainNetwork")       // USDT网络类型
}

// 提现订单项（记录列表用）
struct WithdrawOrderItem {
    1:  string orderNo              (api.body="orderNo")
    2:  string amount               (api.body="amount")
    3:  string feeAmount            (api.body="feeAmount")
    4:  string actualAmount         (api.body="actualAmount")
    5:  i32 withdrawMethod          (api.body="withdrawMethod")
    6:  string methodName           (api.body="methodName")
    7:  i32 status                  (api.body="status")
    8:  string statusText           (api.body="statusText")
    9:  i64 createAt                (api.body="createAt")
}


// ===================== 提现配置 =====================

struct GetWithdrawConfigReq {
    1: required string currencyCode (api.body="currencyCode")
}

struct GetWithdrawConfigResp {
    1: i32 kycStatus                    (api.body="kycStatus")          // KYC状态
    2: i32 allowedWithdrawRule          (api.body="allowedWithdrawRule") // 可提币种规则 1法币 2USDT 3全部
    3: list<WithdrawMethodItem> methods (api.body="methods")            // 可用提现方式
    4: string availableBalance          (api.body="availableBalance")   // 当前可提余额
    5: string todayWithdrawn            (api.body="todayWithdrawn")     // 今日已提金额
    6: string exchangeRate              (api.body="exchangeRate")       // BSB→法币汇率
    7: string currencySymbol            (api.body="currencySymbol")
}


// ===================== 提现账户管理 =====================

struct GetWithdrawAccountReq {
    1: required string currencyCode (api.body="currencyCode")
    2: optional i32 accountType     (api.body="accountType")        // 筛选方式
}

struct GetWithdrawAccountResp {
    1: list<WithdrawAccountInfo> list (api.body="list")
}

struct SaveWithdrawAccountReq {
    1:  required string currencyCode    (api.body="currencyCode")
    2:  required i32 accountType        (api.body="accountType")    // 1银行卡 2电子钱包 3USDT
    3:  optional string bankName        (api.body="bankName")       // 银行卡时必填
    4:  optional string bankAccount     (api.body="bankAccount")
    5:  optional string bankBranch      (api.body="bankBranch")
    6:  required string accountHolder   (api.body="accountHolder")  // KYC姓名自动填充
    7:  optional string eWalletType     (api.body="eWalletType")    // 电子钱包时必填
    8:  optional string eWalletPhone    (api.body="eWalletPhone")
    9:  optional string chainAddress    (api.body="chainAddress")   // USDT时必填
    10: optional i32 chainNetwork       (api.body="chainNetwork")   // USDT时必填
}

struct SaveWithdrawAccountResp {
    1: i64 id                           (api.body="id")
}


// ===================== 创建提现订单 =====================

struct CreateWithdrawOrderReq {
    1: required string currencyCode     (api.body="currencyCode")
    2: required i32 withdrawMethod      (api.body="withdrawMethod") // 1银行卡 2电子钱包 3USDT
    3: required string amount           (api.body="amount")
    4: required i64 withdrawAccountId   (api.body="withdrawAccountId")
    5: optional i32 walletType          (api.body="walletType")     // 来源子钱包(默认中心)
}

struct CreateWithdrawOrderResp {
    1: string orderNo                   (api.body="orderNo")
    2: string amount                    (api.body="amount")
    3: string feeAmount                 (api.body="feeAmount")
    4: string actualAmount              (api.body="actualAmount")
}
```

### 8.3 设计说明

- `GetWithdrawConfigResp.allowedWithdrawRule` 封装了"从哪来回哪去"的提现币种规则判断结果，Service层根据充值历史计算后返回一个简单的枚举值，前端据此展示可用方式
- `SaveWithdrawAccountReq` 中 `accountHolder` 是 required 的，前端自动填充KYC姓名，Service层做二次校验
- optional 字段按 accountType 不同有不同的"事实必填"规则，在 Service 层做条件校验，IDL层不强制（因为Thrift无法表达条件required）

---

## 九、exchange.thrift — 兑换域

### 9.1 文件定位

兑换业务——预览（试算）和执行兑换。

### 9.2 设计

```thrift
namespace go ser_wallet

// ===================== 兑换预览 =====================

struct PreviewExchangeReq {
    1: required string sourceCurrency   (api.body="sourceCurrency")  // 源币种(法币)
    2: required string amount           (api.body="amount")          // 兑换金额
}

struct PreviewExchangeResp {
    1: string sourceCurrency            (api.body="sourceCurrency")
    2: string targetCurrency            (api.body="targetCurrency")  // BSB
    3: string amount                    (api.body="amount")          // 输入金额
    4: string exchangeRate              (api.body="exchangeRate")    // 使用汇率（1 BSB = X 法币）
    5: string exchangeAmount            (api.body="exchangeAmount")  // 兑换获得BSB
    6: string bonusRatio                (api.body="bonusRatio")      // 赠送比例(%)
    7: string bonusAmount               (api.body="bonusAmount")     // 赠送BSB
    8: string actualAmount              (api.body="actualAmount")    // 实际到账=兑换+赠送
    9: string availableBalance          (api.body="availableBalance") // 当前法币可用余额
}


// ===================== 执行兑换 =====================

struct CreateExchangeReq {
    1: required string sourceCurrency   (api.body="sourceCurrency")
    2: required string amount           (api.body="amount")
}

struct CreateExchangeResp {
    1: string orderNo                   (api.body="orderNo")
    2: string exchangeAmount            (api.body="exchangeAmount")  // 兑换获得
    3: string bonusAmount               (api.body="bonusAmount")     // 赠送
    4: string actualAmount              (api.body="actualAmount")    // 实际到账
    5: string exchangeRate              (api.body="exchangeRate")    // 实际使用的汇率
}
```

### 9.3 为什么兑换域最小

兑换业务是内部闭环操作（不涉及三方支付、无审核流程），逻辑最简单：

- 预览 = 纯计算，不写DB
- 执行 = 一个事务内完成（法币扣+BSB加+写订单+写流水）

两个方法足够覆盖。如果评审后兑换功能增加（如兑换记录独立查询、兑换限额配置），struct可以在此文件中扩展。

---

## 十、record.thrift — 记录域

### 10.1 文件定位

C端"记录"页面——四个Tab（充值/提现/兑换/奖励）的分页查询和详情查看。

### 10.2 设计

```thrift
namespace go ser_wallet

// ===================== 记录列表项 =====================

// 通用记录列表项（四Tab统一结构，按recordType区分不同Tab的展示字段）
struct RecordItem {
    1:  string orderNo              (api.body="orderNo")            // 订单号
    2:  i32 recordType              (api.body="recordType")         // 1充值 2提现 3兑换 4奖励
    3:  string amount               (api.body="amount")             // 金额
    4:  string methodName           (api.body="methodName")         // 方式名称
    5:  i32 status                  (api.body="status")             // 状态码
    6:  string statusText           (api.body="statusText")         // 状态文本
    7:  i32 statusColor             (api.body="statusColor")        // 状态色标 1默认 2橙色 3红色
    8:  i64 createAt                (api.body="createAt")
    // 兑换特有
    9:  string exchangeAmount       (api.body="exchangeAmount")     // 兑换获得
    10: string bonusInfo            (api.body="bonusInfo")          // 赠送x%
    11: string actualAmount         (api.body="actualAmount")       // 实际到账
    // 奖励特有
    12: string rewardCategory       (api.body="rewardCategory")     // 奖励类别
    // 提现特有
    13: string feeAmount            (api.body="feeAmount")          // 手续费
}


// ===================== 记录详情 =====================

struct RecordDetail {
    1:  string orderNo              (api.body="orderNo")
    2:  i32 recordType              (api.body="recordType")
    3:  string amount               (api.body="amount")
    4:  i32 method                  (api.body="method")             // 方式枚举
    5:  string methodName           (api.body="methodName")
    6:  i32 status                  (api.body="status")
    7:  string statusText           (api.body="statusText")
    8:  i64 createAt                (api.body="createAt")
    9:  i64 completedAt             (api.body="completedAt")        // 完成时间
    10: string failReason           (api.body="failReason")         // 失败原因
    11: string feeAmount            (api.body="feeAmount")
    12: string actualAmount         (api.body="actualAmount")
    13: string exchangeRate         (api.body="exchangeRate")
    14: string exchangeAmount       (api.body="exchangeAmount")
    15: string bonusAmount          (api.body="bonusAmount")
    16: string currencyCode         (api.body="currencyCode")
    17: string currencySymbol       (api.body="currencySymbol")
}


// ===================== 请求响应 =====================

struct PageRecordReq {
    1: required string currencyCode     (api.body="currencyCode")
    2: required i32 recordType          (api.body="recordType")     // 1充值 2提现 3兑换 4奖励
    3: optional i32 status              (api.body="status")         // 筛选状态
    4: i32 pageNo = 1                   (api.body="pageNo")
    5: i32 pageSize = 10                (api.body="pageSize")
}

struct PageRecordResp {
    1: list<RecordItem> list            (api.body="list")
    2: i64 total                        (api.body="total")
    3: i32 pageNo                       (api.body="pageNo")
    4: i32 pageSize                     (api.body="pageSize")
}

struct DetailRecordReq {
    1: required string orderNo          (api.body="orderNo")
    2: required i32 recordType          (api.body="recordType")
}

struct DetailRecordResp {
    1: RecordDetail data                (api.body="data")
}
```

### 10.3 设计说明：统一结构 vs 四套独立结构

**决策**：用统一的 `RecordItem` + `recordType` 字段区分四个Tab，而非四套独立struct。

**依据**：
- 需求文档4.8：四个Tab的列表字段大量重叠（都有订单号、金额、方式、时间、状态）
- 差异字段很少（兑换多了汇率和赠送、奖励多了类别、提现多了手续费）
- 一个struct的几个optional字段即可覆盖差异
- Service层根据 recordType 查不同的订单表然后统一映射为 RecordItem

**好处**：前端只需一个列表组件，后端只需一个分页接口，按 recordType 路由到不同查询逻辑。避免了四套几乎相同的struct定义。

---

## 十一、wallet_rpc.thrift — 跨服务RPC域

### 11.1 文件定位

这是**最关键**的IDL文件之一——定义了 ser-finance（和其他服务）调用 ser-wallet 的全部资金操作接口的数据结构。

这些接口是"接口契约"：我定义、我实现，finance按此调用。

### 11.2 设计

```thrift
namespace go ser_wallet

// ===================== 公共数据结构 =====================

// 用户单子钱包余额
struct SubWalletBalance {
    1: i32 walletType               (api.body="walletType")         // 1中心 2奖励 3主播 4代理 5场馆
    2: string walletTypeName        (api.body="walletTypeName")
    3: string availableBalance      (api.body="availableBalance")
    4: string frozenBalance         (api.body="frozenBalance")
}

// 币种配置信息（供外部服务读取）
struct CurrencyConfigInfo {
    1: string currencyCode          (api.body="currencyCode")
    2: string currencyName          (api.body="currencyName")
    3: i32 currencyType             (api.body="currencyType")
    4: string currencySymbol        (api.body="currencySymbol")
    5: i32 precision                (api.body="precision")
    6: string thousandsSep          (api.body="thousandsSep")
    7: string decimalSep            (api.body="decimalSep")
    8: string platformRate          (api.body="platformRate")
    9: string depositRate           (api.body="depositRate")
    10: string withdrawRate         (api.body="withdrawRate")
    11: i32 status                  (api.body="status")
}


// ===================== 入账（充值成功/补单） =====================

struct WalletCreditReq {
    1: required i64 userId              (api.body="userId")
    2: required string currencyCode     (api.body="currencyCode")
    3: required string amount           (api.body="amount")         // 入账金额
    4: required string orderNo          (api.body="orderNo")        // 关联订单号（幂等key）
    5: optional string remark           (api.body="remark")
}

struct WalletCreditResp {
    1: bool success                     (api.body="success")
    2: string newBalance                (api.body="newBalance")     // 入账后余额
}


// ===================== 奖金入账 =====================

struct WalletCreditRewardReq {
    1: required i64 userId              (api.body="userId")
    2: required string currencyCode     (api.body="currencyCode")
    3: required string amount           (api.body="amount")
    4: required string orderNo          (api.body="orderNo")        // 幂等key
    5: optional string turnoverMulti    (api.body="turnoverMulti")  // 流水倍数要求
    6: optional string remark           (api.body="remark")
}

struct WalletCreditRewardResp {
    1: bool success                     (api.body="success")
}


// ===================== 冻结（提现发起） =====================

struct WalletFreezeReq {
    1: required i64 userId              (api.body="userId")
    2: required string currencyCode     (api.body="currencyCode")
    3: required i32 walletType          (api.body="walletType")     // 冻结哪个子钱包
    4: required string amount           (api.body="amount")
    5: required string orderNo          (api.body="orderNo")        // 关联提现单号
}

struct WalletFreezeResp {
    1: bool success                     (api.body="success")
    2: i64 freezeRecordId               (api.body="freezeRecordId") // 冻结记录ID
}


// ===================== 解冻（审核驳回/出款失败） =====================

struct WalletUnfreezeReq {
    1: required string orderNo          (api.body="orderNo")        // 关联提现单号
    2: optional string remark           (api.body="remark")
}

struct WalletUnfreezeResp {
    1: bool success                     (api.body="success")
}


// ===================== 扣款（出款成功最终扣除） =====================

struct WalletDeductReq {
    1: required string orderNo          (api.body="orderNo")        // 关联提现单号
    2: optional string remark           (api.body="remark")
}

struct WalletDeductResp {
    1: bool success                     (api.body="success")
}


// ===================== 人工加款 =====================

struct WalletManualAddReq {
    1: required i64 userId              (api.body="userId")
    2: required string currencyCode     (api.body="currencyCode")
    3: required string orderNo          (api.body="orderNo")        // 修正单号 A+16位
    4: optional string centerAmount     (api.body="centerAmount")   // 中心钱包加款金额
    5: optional string rewardAmount     (api.body="rewardAmount")   // 奖励钱包加款金额
    6: optional string streamerAmount   (api.body="streamerAmount") // 主播钱包加款金额
    7: optional string agentAmount      (api.body="agentAmount")    // 代理钱包加款金额（预留）
    8: optional i32 correctionReason    (api.body="correctionReason") // 原因类型
    9: optional string remark           (api.body="remark")
}

struct WalletManualAddResp {
    1: bool success                     (api.body="success")
}


// ===================== 人工减款 =====================

struct WalletManualSubReq {
    1: required i64 userId              (api.body="userId")
    2: required string currencyCode     (api.body="currencyCode")
    3: required string orderNo          (api.body="orderNo")        // 修正单号 M+16位
    4: optional string centerAmount     (api.body="centerAmount")
    5: optional string rewardAmount     (api.body="rewardAmount")
    6: optional string streamerAmount   (api.body="streamerAmount")
    7: optional string agentAmount      (api.body="agentAmount")
    8: optional i32 correctionReason    (api.body="correctionReason")
    9: optional string remark           (api.body="remark")
}

struct WalletManualSubResp {
    1: bool success                         (api.body="success")
    2: optional string actualCenterAmount   (api.body="actualCenterAmount")   // 实际扣款=min(期望,余额)
    3: optional string actualRewardAmount   (api.body="actualRewardAmount")
    4: optional string actualStreamerAmount  (api.body="actualStreamerAmount")
    5: optional string actualAgentAmount    (api.body="actualAgentAmount")
}


// ===================== 查询用户余额 =====================

struct GetUserBalanceReq {
    1: required i64 userId              (api.body="userId")
    2: required string currencyCode     (api.body="currencyCode")
}

struct GetUserBalanceResp {
    1: i64 userId                           (api.body="userId")
    2: string currencyCode                  (api.body="currencyCode")
    3: list<SubWalletBalance> subWallets    (api.body="subWallets")
    4: string totalAvailable                (api.body="totalAvailable")  // 中心+奖励可用
    5: string totalFrozen                   (api.body="totalFrozen")
}


// ===================== 获取币种配置 =====================

struct GetCurrencyConfigReq {
    1: required string currencyCode     (api.body="currencyCode")
}

struct GetCurrencyConfigResp {
    1: CurrencyConfigInfo data          (api.body="data")
}
```

### 11.3 RPC接口设计的关键考量

**幂等key设计**：

| 接口 | 幂等key | 说明 |
|------|---------|------|
| WalletCredit | orderNo（充值单号C/补单号B） | 同一订单只入账一次 |
| WalletCreditReward | orderNo + 奖金flow_type | 同一订单奖金只入一次 |
| WalletFreeze | orderNo（提现单号T） | 同一提现只冻结一次 |
| WalletUnfreeze | orderNo | 同一冻结只解冻一次 |
| WalletDeduct | orderNo | 同一冻结只扣款一次 |
| WalletManualAdd | orderNo（加款单号A） | 同一修正只加一次 |
| WalletManualSub | orderNo（减款单号M） | 同一修正只减一次 |

所有资金变动接口都以 `orderNo` 作为幂等key，这是架构设计ADR-04的直接体现：流水表 `(related_order_no, flow_type)` 唯一索引保证幂等。

**WalletManualSubResp 返回实际扣款金额**：

需求文档5.7："如果减款金额 > 当前余额，实际扣款 = 当前余额"。所以响应必须返回实际扣了多少，finance 需要知道这个信息（可能与期望金额不同）。

**WalletUnfreeze/WalletDeduct 只需 orderNo**：

不需要传 userId/currencyCode/amount，因为 wallet 内部通过 orderNo 关联 freeze_record，冻结记录中已包含所有信息。这简化了 finance 的调用逻辑。

---

## 十二、方法全景清单

### 12.1 按分组整理

| 序号 | 分组 | 方法名 | 功能 | 调用方 |
|------|------|--------|------|--------|
| 1 | B端币种 | PageCurrency | 币种分页列表 | gate-back |
| 2 | B端币种 | CreateCurrency | 创建币种 | gate-back |
| 3 | B端币种 | UpdateCurrency | 编辑币种 | gate-back |
| 4 | B端币种 | ToggleCurrencyStatus | 启禁用币种 | gate-back |
| 5 | B端币种 | SetBaseCurrency | 设置基准币种 | gate-back |
| 6 | B端币种 | DetailCurrency | 币种详情 | gate-back |
| 7 | B端币种 | PageRateLog | 汇率日志分页 | gate-back |
| 8 | C端钱包 | ListActiveCurrency | 可用币种列表 | gate-font |
| 9 | C端钱包 | GetExchangeRate | 获取汇率 | gate-font |
| 10 | C端钱包 | GetWalletOverview | 钱包余额概览 | gate-font |
| 11 | C端充值 | GetDepositConfig | 充值配置 | gate-font |
| 12 | C端充值 | CreateDepositOrder | 创建充值订单 | gate-font |
| 13 | C端充值 | GetDepositStatus | 充值状态查询 | gate-font |
| 14 | C端兑换 | PreviewExchange | 兑换预览 | gate-font |
| 15 | C端兑换 | CreateExchange | 执行兑换 | gate-font |
| 16 | C端提现 | GetWithdrawConfig | 提现配置 | gate-font |
| 17 | C端提现 | GetWithdrawAccount | 查询提现账户 | gate-font |
| 18 | C端提现 | SaveWithdrawAccount | 保存提现账户 | gate-font |
| 19 | C端提现 | CreateWithdrawOrder | 创建提现订单 | gate-font |
| 20 | C端记录 | PageRecord | 记录分页 | gate-font |
| 21 | C端记录 | DetailRecord | 记录详情 | gate-font |
| 22 | RPC | WalletCredit | 入账 | ser-finance |
| 23 | RPC | WalletCreditReward | 奖金入账 | ser-finance |
| 24 | RPC | WalletFreeze | 冻结 | ser-finance |
| 25 | RPC | WalletUnfreeze | 解冻 | ser-finance |
| 26 | RPC | WalletDeduct | 扣款 | ser-finance |
| 27 | RPC | WalletManualAdd | 人工加款 | ser-finance |
| 28 | RPC | WalletManualSub | 人工减款 | ser-finance |
| 29 | RPC | GetUserBalance | 查余额 | ser-finance |
| 30 | RPC | GetCurrencyConfig | 查币种配置 | ser-finance |

### 12.2 按实现阶段映射

| Phase | 包含的方法 | 说明 |
|-------|----------|------|
| Phase 1（脚手架） | 全部方法的IDL定义 | IDL先行，30个方法都定义好 |
| Phase 2（币种配置） | 1-7, 8-9, 30 | B端币种CRUD + C端查询 + 对外GetCurrencyConfig |
| Phase 3（余额体系） | 10, 29 | 钱包概览 + GetUserBalance |
| Phase 4（兑换） | 14-15 | 预览+执行兑换 |
| Phase 5（充值） | 11-13, 22-23 | 充值+WalletCredit |
| Phase 6（提现） | 16-19, 24-26 | 提现+冻结/解冻/扣款 |
| Phase 7（修正+联调） | 20-21, 27-28 | 记录+人工加减款 |

---

## 十三、IDL设计约定速查

### 13.1 命名速查表

| 元素 | 规则 | 示例 |
|------|------|------|
| namespace | `go ser_wallet` | 蛇形，与目录名一致 |
| service | `WalletService` | PascalCase + Service |
| 方法 | `<动词><资源>` PascalCase | CreateCurrency, PageRateLog |
| 请求struct | `<方法名>Req` | CreateCurrencyReq |
| 响应struct | `<方法名>Resp` | CreateCurrencyResp |
| 列表项struct | `<资源>Item` | CurrencyItem, RecordItem |
| 详情struct | `<资源>Detail` | RecordDetail |
| 信息struct | `<资源>Info` | SubWalletInfo, WithdrawAccountInfo |

### 13.2 动词规范

| 动词 | 语义 | 示例 |
|------|------|------|
| Page | 分页查询 | PageCurrency, PageRateLog, PageRecord |
| List | 全量列表（不分页） | ListActiveCurrency |
| Get | 获取单个/聚合信息 | GetWalletOverview, GetDepositConfig |
| Detail | 获取单条详情 | DetailCurrency, DetailRecord |
| Create | 创建资源 | CreateCurrency, CreateDepositOrder |
| Update | 更新资源 | UpdateCurrency |
| Toggle | 状态切换 | ToggleCurrencyStatus |
| Set | 一次性设置 | SetBaseCurrency |
| Save | 保存（含创建语义） | SaveWithdrawAccount |
| Preview | 预览/试算 | PreviewExchange |
| Wallet | RPC资金操作前缀 | WalletCredit, WalletFreeze |

### 13.3 分页标准格式

```thrift
// 请求——筛选条件 + pageNo(默认1) + pageSize(默认10)
struct PageXxxReq {
    1: optional string someFilter   (api.body="someFilter")
    N: i32 pageNo = 1              (api.body="pageNo")
    N+1: i32 pageSize = 10          (api.body="pageSize")
}

// 响应——list + total + pageNo + pageSize
struct PageXxxResp {
    1: list<XxxItem> list           (api.body="list")
    2: i64 total                    (api.body="total")
    3: i32 pageNo                   (api.body="pageNo")
    4: i32 pageSize                 (api.body="pageSize")
}
```

### 13.4 字段类型选择

| 语义 | 类型 | 说明 |
|------|------|------|
| 业务ID | `i64` | userId, orderId, accountId |
| 金额 | `string` | "100.50", "0.00012345"（ADR-06决策） |
| 汇率 | `string` | 同金额，高精度场景 |
| 百分比 | `string` | "10.50" 表示10.50% |
| 枚举/状态 | `i32` | status, type, method |
| 时间 | `i64` | Unix秒级时间戳 |
| 标识码 | `string` | currencyCode, orderNo |
| 文本 | `string` | name, remark, address |
| 布尔 | `bool` | success |
| 列表 | `list<T>` | list, methods, subWallets |

### 13.5 required vs optional 用法

```
required 用于：
  - 接口调用的必要条件（没有它请求无意义）
  - 如：userId, currencyCode, amount, orderNo

optional 用于：
  - 筛选条件（可不传，表示不筛选）
  - 分类条件决定的字段（银行卡/电子钱包/USDT 各自有不同必填项）
  - 响应中的可空字段

不标注（默认optional）用于：
  - 响应体的数据字段（大多数情况）
  - 带默认值的字段（如 pageNo = 1）
```

---

## 十四、金额字段类型决策

### 14.1 工程现状 vs 我们的选择

| 对比 | 现有工程用法 | 我们的选择 |
|------|-------------|----------|
| IDL中金额类型 | `i64`（最小单位，如分） | `string`（原始小数） |
| 现有使用场景 | 直播打赏/钻石消费 | 多币种钱包 |
| 精度需求 | 单精度（固定2位） | 多精度（法币2位/加密8位） |
| 汇率换算 | 无 | 有（中间结果需要高精度） |

### 14.2 为什么选 string 而不是沿用 i64

```
理由一：精度不统一
  法币（VND）精度=2位 → 放大100倍 = i64
  加密币（USDT）精度=8位 → 放大100000000倍 = i64
  同一个字段无法同时用两种放大倍数
  → 必须在IDL层区分精度，或者统一用string

理由二：汇率换算中间结果
  100 VND × 汇率 = BSB
  汇率可能是 0.00004123
  如果用i64存VND=10000（×100），乘以汇率后仍需高精度中间结果
  i64无法优雅地处理精度不同的币种间换算

理由三：行业惯例
  支付领域（Stripe、PayPal的API）传递金额都用string
  避免前后端JSON序列化时的浮点精度丢失

理由四：ADR-06已做决策
  "DB用DECIMAL(30,8)，IDL传输用string，内部运算用shopspring/decimal"
  这是安全性最高的方案
```

### 14.3 string金额的格式约定

```
合法格式：
  "100"        → 整数（法币场景常见）
  "100.50"     → 2位小数
  "0.00012345" → 8位小数
  "0"          → 零值

非法格式（Service层校验拒绝）：
  ""           → 空串
  "abc"        → 非数字
  "-100"       → 负数（金额不允许负数，流水的正负由flow_type决定）
  ".5"         → 缺少整数部分

转换链路：
  前端 "100.50" → IDL string → Service decimal.NewFromString("100.50")
  → 运算后 decimal.StringFixed(precision) → 落库 DECIMAL(30,8)
  → 查询后 model.Amount.String() → IDL string → 前端 "100.50"
```

---

## 十五、与现有服务的include关系

### 15.1 ser-wallet IDL不include其他服务

```
ser-wallet 的 IDL 文件不引用其他服务的 thrift 文件。

原因：
  1. 钱包的请求/响应结构体是自包含的
  2. 虽然业务上依赖 ser-kyc（查KYC状态）和 ser-blog（写审计日志），
     但这些依赖在 Go 代码层通过 rpc.KycClient() 和 rpc.BLogClient() 调用，
     不在 IDL 层 include
  3. 跨服务 include 会增加 gen_kitex.sh 的复杂度和编译依赖

如果需要引用公共类型：
  可 include "../common/enum.thrift"
  但当前分析 ser-wallet 不需要公共枚举（自己的枚举在Go的enum包中定义）
```

### 15.2 其他服务引用 ser-wallet 的场景

```
未来可能出现：
  ser-finance 的 IDL 需要引用 wallet_rpc.thrift 中的某些 struct
  如：GetUserBalanceResp 中的 SubWalletBalance

但按现有工程模式：
  ser-finance 直接 include "../ser-wallet/wallet_rpc.thrift" 即可
  gen_kitex.sh 的 -I 参数已覆盖此路径

当前不需要提前规划，finance需要时再引用。
```

---

## 十六、代码生成与验证

### 16.1 IDL编写完成后的执行步骤

```bash
# 1. 确保文件已创建
ls /Users/mac/gitlab/common/idl/ser-wallet/
# 应该有8个文件：service.thrift + 7个域文件

# 2. 执行代码生成
cd /Users/mac/gitlab/common
bash script/gen_kitex.sh

# 3. 验证生成产物
ls common/kitex_gen/ser_wallet/
# 应该有：walletservice/ 目录
ls common/kitex_gen/ser_wallet/walletservice/
# 应该有：client.go, server.go, walletservice.go 等

# 4. 确认方法签名
# walletservice.go 中应包含30个方法签名
```

### 16.2 生成后需要同步的位置

```
1. common/rpc/rpc_client.go — 新增 WalletClient() 工厂函数
2. common/pkg/consts/namesp/namesp.go — 新增 EtcdWalletService 常量
3. ser-wallet/handler.go — 实现 WalletService 的30个方法
4. gate-back/biz/handler/wallet/ — B端handler转发
5. gate-font/biz/handler/wallet/ — C端handler转发
```

### 16.3 IDL修改的工程流程

```
IDL修改 → gen_kitex.sh → 编译检查 → 更新handler
         ↑ 这一步会覆盖 kitex_gen/ 下的生成代码
         ↑ 所以永远不要手编 kitex_gen/ 下的文件

IDL变更规则：
  - 新增方法 → 重新生成 + handler新增实现
  - 修改struct字段 → 重新生成 + 调整Service逻辑
  - 删除方法 → 不建议直接删除，标记deprecated后下个版本清理
```

---

## 十七、设计弹性空间

> 当前处于预评审阶段，以下场景可能在评审后变化，IDL已预留空间

### 17.1 可能新增的方法

| 场景 | 可能新增的方法 | 预留在哪 |
|------|--------------|---------|
| 代理/场馆钱包上线 | GetAgentWalletOverview, TransferToVenue | wallet.thrift 扩展 |
| 充值地址管理 | GetDepositAddress, AssignDepositAddress | deposit.thrift 扩展 |
| 额外奖金CRUD | CreateBonusActivity, PageBonusActivity | 新增 bonus.thrift |
| 稽核流水查询 | GetAuditStatus, PageAuditRecord | 新增 audit.thrift |
| 对账接口 | ReconcileAccount | wallet_rpc.thrift 扩展 |
| B端订单管理 | B端PageDepositOrder, B端PageWithdrawOrder | 对应域文件扩展 |

### 17.2 可能调整的struct

```
评审后可能调整的点：
  - CreateDepositOrderResp 的字段（取决于充值流程调用方向最终确认）
  - GetWithdrawConfigResp 的字段（取决于充值/提现方式配置数据归属）
  - BonusActivityItem 的字段（取决于奖金活动数据来源确认）
  - WalletFreezeReq 中冻结发起方是wallet还是finance

这些调整只影响struct的字段增减，不影响文件结构和方法命名。
所以当前的8文件30方法的骨架是稳定的。
```

### 17.3 不会变的（确定性骨架）

```
不管评审怎么调整，以下设计不会变：
  ✅ 8个文件的组织方式（业务域不变 → 文件拆分不变）
  ✅ WalletService 单一service定义
  ✅ 方法命名规范
  ✅ 金额用string类型
  ✅ 分页用 pageNo/pageSize 标准格式
  ✅ RPC接口的幂等key=orderNo
  ✅ 9个RPC方法是finance→wallet的必须接口
  ✅ namespace go ser_wallet
```

---

## 十八、IDL设计决策记录

### IDL-ADR-01：8个文件按业务域拆分

```
背景：ser-wallet 涉及6个业务域+跨服务RPC
决策：8个.thrift文件（1主入口 + 6域 + 1 RPC）
替代：全放一个文件（太长难维护）、按B/C端拆（跨域不清晰）
依据：ser-user 13文件/ser-live 9文件/ser-app 5文件——按域拆是工程惯例
```

### IDL-ADR-02：单一 WalletService

```
背景：30个方法是否拆成多个service
决策：一个WalletService包含全部30个方法
替代：CurrencyService + WalletService + FinanceRpcService 分三个
依据：工程中19个服务全部是单service模式；拆service需要多端口多client，运维成本高
```

### IDL-ADR-03：金额用 string 不用 i64

```
背景：现有工程用i64存最小单位，钱包场景多精度+汇率换算
决策：IDL中金额字段统一用string
替代：i64存最小单位（需要不同币种不同放大倍数）
依据：ADR-06决策"DECIMAL+string+decimal库"；法币2位和加密8位精度不同，
     i64方案需要在前后端都做精度转换且换算时可能溢出
```

### IDL-ADR-04：记录页面统一 RecordItem

```
背景：C端记录页四个Tab（充值/提现/兑换/奖励），各Tab字段大量重叠
决策：用统一的RecordItem + recordType区分，而非四套独立struct
替代：DepositRecordItem + WithdrawRecordItem + ExchangeRecordItem + RewardRecordItem
依据：80%字段重叠，差异字段用optional覆盖；一个接口+一个struct比四套简洁得多
```

### IDL-ADR-05：wallet_rpc.thrift 独立管理RPC接口

```
背景：RPC接口（供finance调用）和B/C端接口混在一起还是分开
决策：独立 wallet_rpc.thrift 文件
替代：按业务域分散到各域文件中
依据：参考ser-user的user_rpc.thrift和ser-live的live_rpc.thrift；
     RPC接口的调用方、关注点、变更节奏与B/C端不同，独立管理更清晰；
     finance的开发者只需看wallet_rpc.thrift就能了解全部可调用接口
```

---

## 总结

### 一张图看清全局

```
┌──────────────────────────────────────────────────────────────────┐
│              ser-wallet IDL 全景                                   │
│                                                                   │
│  service.thrift (主入口)                                          │
│  └── service WalletService { 30个方法 }                          │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │ currency    │  │ wallet       │  │ deposit      │            │
│  │ .thrift     │  │ .thrift      │  │ .thrift      │            │
│  │             │  │              │  │              │            │
│  │ B端:7方法    │  │ C端:1方法    │  │ C端:3方法    │            │
│  │ C端:2方法    │  │              │  │              │            │
│  │ ~12 struct  │  │ ~3 struct    │  │ ~8 struct    │            │
│  └─────────────┘  └──────────────┘  └──────────────┘            │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │ withdraw    │  │ exchange     │  │ record       │            │
│  │ .thrift     │  │ .thrift      │  │ .thrift      │            │
│  │             │  │              │  │              │            │
│  │ C端:4方法    │  │ C端:2方法    │  │ C端:2方法    │            │
│  │ ~10 struct  │  │ ~4 struct    │  │ ~5 struct    │            │
│  └─────────────┘  └──────────────┘  └──────────────┘            │
│                                                                   │
│  ┌─────────────────────────────────────────────────┐             │
│  │ wallet_rpc.thrift                                │             │
│  │                                                  │             │
│  │ RPC供finance调用:9方法   ~20 struct              │             │
│  │ WalletCredit / CreditReward / Freeze / Unfreeze │             │
│  │ / Deduct / ManualAdd / ManualSub                │             │
│  │ / GetUserBalance / GetCurrencyConfig            │             │
│  └─────────────────────────────────────────────────┘             │
│                                                                   │
│  总计：8个文件 | 30个方法 | ~62个struct                           │
│  骨架确定性：高（业务域不变 → 文件结构不变）                       │
│  细节弹性：中（字段增减不影响骨架）                                 │
└──────────────────────────────────────────────────────────────────┘
```

### 核心数据

| 指标 | 数值 |
|------|------|
| 文件数 | 8个 |
| 方法总数 | 30个 |
| struct总数 | ~62个 |
| B端方法 | 7个 |
| C端方法 | 14个 |
| RPC方法 | 9个 |
| 金额类型 | string |
| 分页格式 | pageNo + pageSize |
| 幂等key | orderNo |
