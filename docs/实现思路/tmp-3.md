# ser-wallet 整体实现思路（粗粒度）

> 梳理时间：2026-02-26
> 定位：需求未完全定型前的方向性实现思路，聚焦"绕不过、必须做、要考虑"的方面
> 依据：需求文档 + 产品原型 + 接口评估 + 依赖关系 + 链路关系 + 流程思路 + 工程认知
> 角色：我负责钱包+币种配置模块，配合财务模块联调

---

## 一、IDL 先行：接口契约是一切的起点

### 1.1 为什么绕不过

工程规范要求 IDL 先行。gen_kitex.sh 从 IDL 生成 kitex_gen（类型定义+服务接口），handler.go 和 main.go 的骨架也从这里来。没有 IDL，后续所有代码编写都没有基础。

### 1.2 实现思路

**目录结构：**

```
common/idl/ser-wallet/
├── service.thrift           ← 主文件：WalletService 全部方法声明
├── wallet.thrift            ← 钱包核心类型（余额、流水、钱包信息）
├── currency.thrift          ← 币种配置类型（币种、汇率、日志）
├── deposit.thrift           ← 充值相关类型（订单、方式、奖金）
├── withdraw.thrift          ← 提现相关类型（订单、账户、方式）
├── exchange.thrift          ← 兑换相关类型（汇率、记录）
└── record.thrift            ← 记录查询类型（列表、详情）
```

**service.thrift 方法分组思路：**

```thrift
namespace go ser_wallet

service WalletService {
    // ===== C端接口（gate-font 转发）=====
    // 首页 & 币种
    wallet.GetWalletHomeResp       GetWalletHome(1: wallet.GetWalletHomeReq req)
    wallet.GetCurrencyListResp     GetCurrencyList(1: wallet.GetCurrencyListReq req)

    // 充值
    deposit.GetDepositMethodsResp  GetDepositMethods(1: deposit.GetDepositMethodsReq req)
    deposit.CreateDepositOrderResp CreateDepositOrder(1: deposit.CreateDepositOrderReq req)
    deposit.CheckDuplicateResp     CheckDuplicateDeposit(1: deposit.CheckDuplicateReq req)

    // 兑换
    exchange.GetExchangeRateResp   GetExchangeRate(1: exchange.GetExchangeRateReq req)
    exchange.DoExchangeResp        DoExchange(1: exchange.DoExchangeReq req)

    // 提现
    withdraw.GetWithdrawMethodsResp  GetWithdrawMethods(1: withdraw.GetWithdrawMethodsReq req)
    withdraw.GetWithdrawAccountResp  GetWithdrawAccount(1: withdraw.GetWithdrawAccountReq req)
    withdraw.SaveWithdrawAccountResp SaveWithdrawAccount(1: withdraw.SaveWithdrawAccountReq req)
    withdraw.CreateWithdrawOrderResp CreateWithdrawOrder(1: withdraw.CreateWithdrawOrderReq req)

    // 记录
    record.GetRecordListResp       GetRecordList(1: record.GetRecordListReq req)
    record.GetRecordDetailResp     GetRecordDetail(1: record.GetRecordDetailReq req)

    // ===== B端接口（gate-back 转发）=====
    currency.PageCurrencyResp      PageCurrency(1: currency.PageCurrencyReq req)
    currency.EditCurrencyResp      EditCurrency(1: currency.EditCurrencyReq req)
    currency.SetBaseCurrencyResp   SetBaseCurrency(1: currency.SetBaseCurrencyReq req)
    currency.StatusCurrencyResp    StatusCurrency(1: currency.StatusCurrencyReq req)
    currency.PageRateLogResp       PageExchangeRateLog(1: currency.PageRateLogReq req)
    wallet.PageUserWalletResp      PageUserWallet(1: wallet.PageUserWalletReq req)
    wallet.DetailUserWalletResp    DetailUserWallet(1: wallet.DetailUserWalletReq req)

    // ===== RPC 对外提供（供财务/投注/直播/活动调用）=====
    wallet.CreditWalletResp        CreditWallet(1: wallet.CreditWalletReq req)
    wallet.DebitWalletResp         DebitWallet(1: wallet.DebitWalletReq req)
    wallet.FreezeBalanceResp       FreezeBalance(1: wallet.FreezeBalanceReq req)
    wallet.UnfreezeBalanceResp     UnfreezeBalance(1: wallet.UnfreezeBalanceReq req)
    wallet.ConfirmDebitResp        ConfirmDebit(1: wallet.ConfirmDebitReq req)
    wallet.ManualCreditResp        ManualCredit(1: wallet.ManualCreditReq req)
    wallet.ManualDebitResp         ManualDebit(1: wallet.ManualDebitReq req)
    wallet.SupplementCreditResp    SupplementCredit(1: wallet.SupplementCreditReq req)
    wallet.GetBalanceResp          GetBalance(1: wallet.GetBalanceReq req)
    wallet.RewardCreditResp        RewardCredit(1: wallet.RewardCreditReq req)
}
```

### 1.3 要考虑的点

- **IDL 中不要过早定义具体字段细节**：需求未定型，先把方法名和分组定下来，Req/Resp 内部字段可以后续迭代。IDL 支持 optional，新增字段不影响已有调用方
- **命名规范对齐已有服务**：`namespace go ser_wallet`，服务名 `WalletService`，类型后缀 `Req/Resp/Info`
- **RPC 对外接口要尽早暴露给财务模块**：财务要调用 CreditWallet/ConfirmDebit/UnfreezeBalance 等，IDL 定义越早，对方越早可以集成

---

## 二、公共基础设施注册：四处必改

### 2.1 为什么绕不过

ser-wallet 是新服务，不注册到公共设施就无法被发现、无法被调用、无法编译通过。

### 2.2 四处修改清单

**① namesp.go** — 服务命名常量

```go
// 文件：common/pkg/consts/namesp/namesp.go
// 新增三行：
EtcdWalletService = "wallet_service"
EtcdWalletPort    = "/slg/serv/wallet/port"
EtcdWalletDb      = "/slg/serv/wallet/db"
```

**② rpc_client.go** — RPC 客户端工厂

```go
// 文件：common/rpc/rpc_client.go
// 新增 WalletClient() 工厂方法，模式与 AppClient() 完全一致
// sync.Once + walletservice.NewClient(namesp.EtcdWalletService, InitRpcClientParams()...)
```

**③ go.work** — 工作区声明

```go
// 文件：/Users/mac/gitlab/go.work
// use 块中新增：
./ser-wallet
```

**④ ETCD 配置** — 写入运行时配置

```
/slg/serv/wallet/port  →  "8022"  （端口号，需确认不与已有服务冲突）
/slg/serv/wallet/db    →  "ser_wallet"  （数据库名）
```

### 2.3 要考虑的点

- 端口号需要查看已有服务占用情况，避免冲突
- 数据库名 `ser_wallet` 需要在 TiDB 中提前创建
- rpc_client 注册后，gate-back/gate-font 以及其他想调用钱包的服务都能通过 `rpc.WalletClient()` 发起调用

---

## 三、网关路由注册：双端接入

### 3.1 为什么绕不过

ser-wallet 同时服务 C 端用户和 B 端管理员，需要在两个网关都注册路由。不注册则 HTTP 请求无法到达 ser-wallet。

### 3.2 gate-back（B端）实现思路

```
新增文件：
  gate-back/biz/handler/wallet/   ← B端 handler（薄封装，一行代码）
  gate-back/biz/router/wallet/    ← B端路由注册

路由示例（全部走 my_router.Api，需 Token+URL权限）：
  POST /admin/api/wallet/currency/page     → PageCurrency
  POST /admin/api/wallet/currency/edit     → EditCurrency
  POST /admin/api/wallet/currency/base     → SetBaseCurrency
  POST /admin/api/wallet/currency/status   → StatusCurrency
  POST /admin/api/wallet/rate/log/page     → PageExchangeRateLog
  POST /admin/api/wallet/user/page         → PageUserWallet
  POST /admin/api/wallet/user/detail       → DetailUserWallet

在 register.go 中添加：
  wallet.RouterWallet(r)
```

### 3.3 gate-font（C端）实现思路

```
新增文件：
  gate-font/biz/handler/wallet/   ← C端 handler
  gate-font/biz/router/wallet/    ← C端路由注册

路由规划：
  需登录的（my_router.Api）：
    POST /api/wallet/home              → GetWalletHome
    POST /api/wallet/currency/list     → GetCurrencyList
    POST /api/wallet/deposit/methods   → GetDepositMethods
    POST /api/wallet/deposit/create    → CreateDepositOrder
    POST /api/wallet/deposit/check     → CheckDuplicateDeposit
    POST /api/wallet/exchange/rate     → GetExchangeRate
    POST /api/wallet/exchange/do       → DoExchange
    POST /api/wallet/withdraw/methods  → GetWithdrawMethods
    POST /api/wallet/withdraw/account  → GetWithdrawAccount
    POST /api/wallet/withdraw/account/save → SaveWithdrawAccount
    POST /api/wallet/withdraw/create   → CreateWithdrawOrder
    POST /api/wallet/record/list       → GetRecordList
    POST /api/wallet/record/detail     → GetRecordDetail

  可能公开的（my_router.Open）：
    暂无明确场景。游客可查看充值页但需登录才能操作。
    如果有"游客可USDT充值"的场景，对应接口可能放Open组。

在 register.go 中添加：
  wallet.RouterWallet(r)
```

### 3.4 要考虑的点

- 所有 handler 都是一行代码的 `rpc.Handler(ctx, c, rpc.WalletClient().XxxMethod)` 薄封装
- URL 路径命名遵循 `/{模块}/{资源}/{操作}` 的已有规范
- B端 URL 注册后需要在 ser-buser 的权限系统中配置对应的 URL 权限项，管理员才能访问
- C端充值/提现等涉及资金操作的接口必须走 Api 组（需登录）

---

## 四、数据库表设计方向

### 4.1 为什么绕不过

表结构决定了 GORM Gen 生成的 model/query/repo，而这些是 Service 层和 Repository 层的基础。表结构没定，代码就没有数据支撑。

### 4.2 核心表清单（粗粒度）

```
必须有的表（无这些表模块无法运行）：

① currency_config         — 币种配置表
   核心字段方向：币种代码/名称/类型(法定/加密/平台)/图标/货币符号/
                千分位规则/小数位规则/实时汇率/平台汇率/入款汇率/出款汇率/
                浮动比例/阈值/状态/基准币种标记

② user_wallet             — 用户钱包表
   核心字段方向：用户ID/币种代码/钱包类型(中心/奖励/主播/代理)/
                可用余额/冻结余额/状态
   特点：一个用户可能有多个币种×多个钱包类型的记录

③ wallet_flow             — 钱包流水表
   核心字段方向：流水ID/用户ID/币种代码/钱包类型/变更前余额/变更金额/
                变更后余额/流水类型(充值/扣款/冻结/解冻/确认/兑换/...)/
                关联订单号/备注
   特点：每次余额变更都写一条，是对账核心

④ exchange_rate_log       — 汇率变更日志表
   核心字段方向：日志编号/币种代码/实时汇率/平台汇率/入款汇率/出款汇率/更新时间

⑤ deposit_order           — 充值订单表
   核心字段方向：订单号/用户ID/币种代码/金额/充值方式/状态(待支付/进行中/成功/失败)/
                通道订单号/汇率快照(BSB时)/奖金活动ID/奖金金额

⑥ withdraw_order          — 提现订单表
   核心字段方向：订单号/用户ID/币种代码/金额/提现方式/状态(待审核/...)/
                财务订单号/汇率快照(BSB时)/冻结流水号/手续费

⑦ withdraw_account        — 提现账户表
   核心字段方向：用户ID/提现方式/账户信息JSON(银行名/账号/持有人/网点 或 钱包类型/手机号/持有人)
   特点：每个用户每种方式存一条，后续复用

⑧ exchange_record         — 兑换记录表
   核心字段方向：用户ID/源币种/目标币种(BSB)/兑换金额/获得BSB/赠送BSB/
                使用汇率/赠送比例/流水倍数要求
```

### 4.3 要考虑的点

- **金额字段用什么类型**：涉及资金必须精确。TiDB/MySQL 中推荐 `DECIMAL(20,8)` 或者用整数存储最小单位（如分）。Go 层面可以用 `int64`（存最小单位）或 `string`（精确传输）+ 服务端计算用 `decimal` 库。这个需要团队统一决策
- **软删除规范**：已有工程用 `deleted_at` 字段（0=未删除，毫秒时间戳=已删除），GORM Gen 自动处理，照搬即可
- **表创建在 TiDB 后**，执行 `gorm_gen.go` 自动生成 model/query/repo 18 个通用 CRUD 方法
- **字段还没完全定下来没关系**：先建核心字段的表，跑通 GORM Gen。后续加字段重新执行 gorm_gen.go 即可
- **user_wallet 的主键设计**：建议 `(user_id, currency_code, wallet_type)` 做联合唯一索引，确保不会重复创建
- **wallet_flow 的 order_no 唯一索引**：做幂等校验的基础

---

## 五、服务内部分层结构

### 5.1 为什么绕不过

已有服务遵循严格的分层规范（handler → ser → rep → gorm_gen）。不按规范写，代码无法融入现有体系，后续维护成本高。

### 5.2 目录规划

```
ser-wallet/
├── main.go                          ← 服务入口（标准模板）
├── handler.go                       ← RPC Handler（薄封装，gen_kitex 生成骨架）
├── go.mod
├── internal/
│   ├── cfg/
│   │   ├── mysql.go                 ← sync.Once 初始化 GORM
│   │   └── redis.go                 ← sync.Once 初始化 Redis
│   ├── errs/
│   │   └── code.go                  ← 错误码定义（60{XX}000 起）
│   ├── enum/
│   │   ├── wallet_type.go           ← 钱包类型枚举（中心/奖励/主播/代理）
│   │   ├── currency_type.go         ← 币种类型枚举（法定/加密/平台）
│   │   ├── flow_type.go             ← 流水类型枚举
│   │   ├── order_status.go          ← 订单状态枚举
│   │   └── freeze_status.go         ← 冻结状态枚举（冻结中/已解冻/已确认）
│   ├── ser/                         ← Service 层（核心业务逻辑）
│   │   ├── currency_service.go      ← 币种配置业务
│   │   ├── wallet_service.go        ← 钱包核心业务（余额操作原子方法）
│   │   ├── deposit_service.go       ← 充值业务
│   │   ├── withdraw_service.go      ← 提现业务
│   │   ├── exchange_service.go      ← 兑换业务
│   │   └── record_service.go        ← 记录查询业务
│   ├── rep/                         ← Repository 层（数据访问，在自动生成的 repo 基础上扩展）
│   │   ├── currency_rep.go
│   │   ├── wallet_rep.go
│   │   ├── deposit_rep.go
│   │   ├── withdraw_rep.go
│   │   ├── exchange_rep.go
│   │   └── flow_rep.go
│   ├── cache/                       ← 缓存层
│   │   ├── balance_cache.go         ← 余额缓存
│   │   └── currency_cache.go        ← 币种配置缓存
│   ├── gen/
│   │   └── gorm_gen.go              ← GORM Gen 入口
│   └── gorm_gen/                    ← 自动生成（禁止手动编辑）
│       ├── model/
│       ├── query/
│       └── repo/
```

### 5.3 handler.go 实现思路

handler.go 由 gen_kitex.sh 生成骨架，手动补充 Service 注入：

```go
type WalletServiceImpl struct {
    currencyService *ser.CurrencyService
    walletService   *ser.WalletService
    depositService  *ser.DepositService
    withdrawService *ser.WithdrawService
    exchangeService *ser.ExchangeService
    recordService   *ser.RecordService
}

// 每个 IDL 方法转发到对应 Service
func (s *WalletServiceImpl) GetWalletHome(ctx context.Context, req *ser_wallet.GetWalletHomeReq) (*ser_wallet.GetWalletHomeResp, error) {
    return s.walletService.GetWalletHome(ctx, req)
}

func (s *WalletServiceImpl) PageCurrency(ctx context.Context, req *ser_wallet.PageCurrencyReq) (*ser_wallet.PageCurrencyResp, error) {
    return s.currencyService.PageCurrency(ctx, req)
}

// ... 其余方法同理
```

### 5.4 Service 层的内部依赖关系

```
currencyService ← 独立，不依赖其他 Service
walletService   ← 依赖 currencyService（需要币种校验和汇率）
exchangeService ← 依赖 currencyService + walletService
depositService  ← 依赖 currencyService + walletService + 外部RPC
withdrawService ← 依赖 currencyService + walletService + 外部RPC
recordService   ← 依赖上述所有的数据表（纯读）
```

构造函数中注入依赖，不在 Service 之间循环依赖。walletService 提供原子方法（creditCore/debitCore/freezeCore 等），depositService 和 withdrawService 组合调用。

---

## 六、错误码规划

### 6.1 为什么绕不过

所有业务错误通过 `ret.BizErr(ctx, errorCode, msg)` 抛出，错误码是 i18n 翻译的 key，也是前端判断具体错误类型的依据。没有错误码体系，无法返回规范的业务错误。

### 6.2 实现思路

```go
// internal/errs/code.go

const (
    // 币种配置模块 60{XX}0xx
    CurrencyBaseError = 60220000 + iota  // 编号 022（待确认）
    CurrencyNotFound                     // 币种不存在
    CurrencyDisabled                     // 币种已禁用
    CurrencyBaseCurrencySet              // 基准币种已设置
    CurrencyBaseCurrencyCannotDisable    // 基准币种不可禁用
)

const (
    // 钱包核心模块 60{XX}1xx
    WalletBaseError = 60221000 + iota
    WalletInsufficientBalance            // 余额不足
    WalletFreezeRecordNotFound           // 冻结记录不存在
    WalletFreezeStatusInvalid            // 冻结状态不正确
    WalletDuplicateOrder                 // 重复订单（幂等拦截）
)

const (
    // 充值模块 60{XX}2xx
    DepositBaseError = 60222000 + iota
    DepositDuplicateBlocked              // 重复充值拦截
    DepositDuplicateWarning              // 重复充值提示
    DepositChannelError                  // 通道创建失败
)

const (
    // 提现模块 60{XX}3xx
    WithdrawBaseError = 60223000 + iota
    WithdrawKycRequired                  // 需要KYC认证
    WithdrawExceedDailyLimit             // 超出每日限额
    WithdrawExceedSingleLimit            // 超出单笔限额
    WithdrawAccountNameMismatch          // 账户持有人与KYC不一致
)

const (
    // 兑换模块 60{XX}4xx
    ExchangeBaseError = 60224000 + iota
    ExchangeInsufficientBalance          // 兑换余额不足
    ExchangeRateUnavailable              // 汇率不可用
)
```

### 6.3 要考虑的点

- 编号 022 是推测的下一个可用编号，需确认
- 错误码定义后需要在 ser-i18n 中配置对应的多语言翻译（Redis Hash 方式）
- 前端会根据特定错误码做不同交互（如 DepositDuplicateWarning 弹窗带"继续充值"按钮）

---

## 七、与财务模块的接口对接

### 7.1 为什么绕不过

充值和提现流程都强依赖财务模块。不确定财务接口，充值页和提现页就无法展示方式列表，订单就无法创建。

### 7.2 需要对齐的接口清单

**我调财务（ser-wallet 作为 Client）：**

| 优先级 | 调用方法 | 场景 | 需确认的信息 |
|--------|---------|------|-------------|
| P0 | GetPaymentConfig | 获取充值/提现方式列表 | 服务名、方法签名、请求/响应结构 |
| P0 | CreatePayOrder | 创建充值通道订单 | 服务名、方法签名、传入哪些字段、返回什么 |
| P0 | CreateWithdrawOrder | 创建提现审核订单 | 同上 |

**财务调我（ser-wallet 作为 Server）：**

| 优先级 | 提供方法 | 场景 | 我需要告诉财务的信息 |
|--------|---------|------|-------------------|
| P0 | CreditWallet | 充值成功上账 | 方法签名、必传字段（userID/currencyCode/amount/orderNo） |
| P0 | ConfirmDebit | 出款成功确认扣除 | 方法签名、必传字段 |
| P0 | UnfreezeBalance | 驳回/失败解冻退回 | 方法签名、必传字段 |
| P1 | ManualCredit | 人工加款 | 方法签名、walletType 枚举值 |
| P1 | ManualDebit | 人工减款 | 方法签名、返回实际减款金额 |
| P1 | SupplementCredit | 充值补单 | 方法签名 |

### 7.3 实现思路

- **第一步**：尽快把 IDL 中 RPC 对外接口（R1~R10）定义出来，发给财务模块负责人
- **第二步**：确认财务模块的服务名和 IDL 方法签名，确认后在 common/idl 中是否已有财务模块 IDL
- **第三步**：如果财务模块 IDL 还没定，先用 Mock 开发。在 ser-wallet 内部写 Mock 返回，充值/提现本地流程先跑通
- **Mock 切换真实调用**：只需把 Mock 方法替换为 `rpc.FinanceClient().XxxMethod()`，Service 层逻辑不变

### 7.4 要考虑的点

- 财务模块的服务名、方法签名目前是待定项，不能阻塞钱包的独立开发
- 需要约定的不只是接口签名，还有**调用时序**（谁先调谁、回调时机、失败怎么处理）
- 建议尽早安排一次与财务模块负责人的对接会，重点对齐上述 6 个 P0 接口

---

## 八、已有 RPC 服务的调用集成

### 8.1 为什么绕不过

钱包业务依赖已有的 KYC/用户/日志/S3 等服务。这些服务的 RPC Client 在 common/rpc/rpc_client.go 中已注册，方法已在 IDL 中定义，可以直接调用。

### 8.2 已有服务调用清单

| 调用目标 | 工厂方法 | 使用场景 | 调用方式 |
|---------|---------|---------|---------|
| ser-kyc | `rpc.KycClient()` | 提现时查 KYC 状态 | `KycClient().KycQuery(ctx, req)` |
| ser-user | `rpc.UserClient()` | 获取用户信息/KYC姓名 | `UserClient().GetUserInfoExpend(ctx, req)` |
| ser-blog | `rpc.BLogClient()` | B端操作审计日志 | `BLogClient().CreateLog(ctx, req)` |
| ser-s3 | `rpc.S3Client()` | 币种图标上传 | `S3Client().GetPresignUrl(ctx, req)` |
| ser-i18n | 无（Redis直读） | 错误码翻译 | 框架层自动处理 |

### 8.3 实现思路

- 直接在 Service 层 import `common/rpc`，通过 `rpc.XxxClient().Method()` 调用
- 不需要额外注册或配置，已有 Client 工厂是现成的
- 调用失败时的降级策略：
  - ser-kyc 不可用：视为"未认证"，仅开放 USDT 提现（保守策略）
  - ser-blog 不可用：审计日志失败不阻断业务（降级为本地日志）
  - ser-s3 不可用：图标上传功能不可用，不影响其他功能

---

## 九、钱包核心逻辑：绕不过的六个原子操作

### 9.1 为什么绕不过

钱包的本质是"余额管理"。所有业务场景（充值、提现、兑换、投注、返奖、人工修正）最终都要调用这些原子操作。它们是 ser-wallet 对外输出能力的核心。

### 9.2 六个原子操作

```
操作            作用               余额变化                  幂等
────            ────               ────────                  ────
creditCore      上账（加钱）        指定钱包.余额 += amount   orderNo
debitCore       扣款（减钱）        中心 -= X, 奖励 -= Y     orderNo
freezeCore      冻结               可用 -= amount, 冻结 += amount   orderNo
unfreezeCore    解冻（退回）        冻结 -= amount, 可用 += amount   orderNo + 冻结状态
confirmDebitCore 确认扣除（永久减） 冻结 -= amount                    orderNo + 冻结状态
writeFlow       写流水             无余额变化                        -
```

### 9.3 实现思路

```
每个原子操作遵循相同的模式：

1. 幂等校验：用 orderNo 查是否已处理
   → 已处理：直接返回上次结果
   → 未处理：继续

2. 前置校验：余额充足性（扣款/冻结时）、冻结状态正确性（解冻/确认时）

3. 事务执行：在单个数据库事务中完成余额变更 + 流水写入

4. 缓存失效：DEL 该用户该币种的余额缓存

5. 返回结果
```

### 9.4 要考虑的点

- **并发安全**：两个请求同时扣同一个用户的余额，需要防超扣。方案有二：
  - 悲观锁：`SELECT ... FOR UPDATE` 锁定钱包行再更新
  - 乐观锁：`UPDATE wallet SET balance = balance - amount WHERE balance >= amount`，通过 affected rows 判断
  - 推荐乐观锁方案（`UPDATE ... WHERE balance >= amount`），性能更好，GORM Gen 可支持
- **扣款分配规则**（优先中心再奖励）：这个逻辑在 debitCore 中实现，返回值中要包含各钱包实际扣了多少，后续投注返奖需要用
- **自动创建钱包**：CreditWallet/ManualCredit 等上账操作，如果目标钱包不存在要自动创建（INSERT ... ON DUPLICATE KEY UPDATE 模式）
- **冻结状态机**：Freeze → Confirm/Unfreeze 互斥，用冻结流水的 status 字段控制

---

## 十、币种配置与汇率体系

### 10.1 为什么绕不过

币种配置是钱包的地基。没有币种数据，钱包首页无法渲染，兑换无法计算，BSB充值/提现无法换算汇率。汇率定时更新是维持数据时效性的唯一手段。

### 10.2 实现思路

**B端配置管理（CRUD，参照已有 ser-app 模式）：**
- PageCurrency / EditCurrency / SetBaseCurrency / StatusCurrency / PageExchangeRateLog
- 标准分页查询 + 标准编辑 + 审计日志
- 这些是最常规的 B 端 CRUD，实现上没有特殊难点

**汇率定时刷新（需要额外考虑）：**

```
触发方式：定时任务（通过 ser-cron 注册，或服务内部 goroutine ticker）
执行频率：可配置（如每分钟一次）

核心逻辑：
  遍历启用的非基准币种 →
    并行调 3 家三方汇率 API →
    取平均值得实时汇率 →
    计算偏差值 →
    偏差 >= 阈值：更新平台汇率 + 入款/出款汇率 + 写日志 →
    所有结果写 Redis 缓存
```

### 10.3 要考虑的点

- **三方汇率 API 的选择**：需要确认用哪三家（如 ExchangeRate-API、Open Exchange Rates、CurrencyLayer 等），以及 API Key 管理
- **汇率更新的原子性**：更新平台汇率时，入款汇率和出款汇率要一起更新（同一事务），不能出现平台汇率更新了但入款汇率还是旧的
- **汇率缓存**：C端读汇率走 Redis 缓存，定时任务更新后写入新值。缓存 key 建议 `wallet:rate:{currencyCode}`
- **币种配置缓存**：币种列表、展示规则等变动不频繁的数据，缓存到 Redis 减少数据库查询。B端编辑后主动失效
- **定时任务实现方式**：已有 ser-cron 服务，可以注册定时任务。如果 ser-cron 的调度方式不适合（如需要秒级/分钟级高频），也可以在 ser-wallet 内部用 goroutine + time.Ticker 自行调度

---

## 十一、充值流程实现方向

### 11.1 绕不过的点

- 重复转账检测逻辑（纯内部，可独立实现）
- 本地充值订单的创建和状态管理
- 与财务模块的订单创建对接（P0 依赖，需 Mock 先行）
- 被财务回调时的上账处理（CreditWallet，已在原子操作中覆盖）

### 11.2 实现思路

```
可独立开发的部分：
  - 充值订单表的 CRUD
  - 重复转账检测逻辑（查自己的订单表）
  - BSB 充值时的汇率锁定逻辑
  - 充值成功上账（CreditWallet 原子操作）

需要 Mock 的部分：
  - 获取充值方式列表（Mock 财务模块返回固定数据）
  - 创建通道订单（Mock 返回虚拟订单号和支付信息）
  - 奖金活动校验（Mock 活动模块返回）

联调时替换：
  Mock → rpc.FinanceClient().GetPaymentConfig()
  Mock → rpc.FinanceClient().CreatePayOrder()
  Mock → rpc.ActivityClient().ValidateBonus()
```

### 11.3 要考虑的点

- 充值订单号生成：用 snowflake 分布式 ID，保证全局唯一
- 订单状态流转：待支付 → 进行中 → 成功/失败，状态只能正向流转
- BSB 充值时汇率快照必须在创建订单时锁定并记录到订单中，回调上账时用快照汇率

---

## 十二、提现流程实现方向

### 12.1 绕不过的点

- KYC 状态查询（调 ser-kyc，已有接口）
- 提现方式的三重过滤逻辑（KYC + 充值历史 + 财务配置）
- 提现账户的保存和复用 + KYC 姓名强校验
- 冻结-确认/解冻状态机
- 与财务模块的提现订单对接

### 12.2 实现思路

```
可独立开发的部分：
  - 提现账户表 CRUD
  - KYC 姓名校验逻辑（调 ser-user 获取 → 对比）
  - 冻结/解冻/确认扣除原子操作（已在第九节覆盖）
  - 提现限额校验（单笔/单日）
  - 提现订单表 CRUD

需要 Mock 的部分：
  - 获取提现方式列表（Mock 财务模块返回）
  - 创建提现审核订单（Mock 财务模块返回）

三重过滤的实现思路：
  KYC 状态过滤：rpc.KycClient().KycQuery() → 未认证仅保留 USDT
  充值历史过滤：查自己的充值订单表 → 法币充/USDT充/混合充 → 映射可提现渠道
  财务配置过滤：Mock/真实调财务获取该币种可用提现方式
  三者取交集 = 最终可用方式
```

### 12.3 要考虑的点

- 冻结后调财务创建审核订单失败的回滚策略：调自己的 unfreezeCore 解冻
- 提现账户持有人校验是**后端强校验**，不能只依赖前端置灰
- 提现限额数据来自财务配置，需缓存或在获取提现方式时一起拿到

---

## 十三、兑换流程实现方向

### 13.1 绕不过的点

- 跨币种事务（法币减少 + BSB增加，同一事务）
- 赠送到奖励钱包的处理
- 汇率锁定

### 13.2 实现思路

```
完全可独立开发，不依赖任何外部模块：
  - 获取平台汇率（从币种配置缓存读）
  - 余额校验
  - 事务内：法币中心 -= X → BSB中心 += Y → BSB奖励 += Z → 写流水
  - 失效双币种缓存
```

### 13.3 为什么建议最先实现

- 零外部依赖，可以端到端跑通
- 覆盖了钱包核心能力（余额查询、余额变更、事务、流水、缓存失效）
- 验证了 IDL → handler → service → repository → GORM Gen 全链路
- 如果兑换流程能跑通，充值/提现只是在此基础上增加外部调用

---

## 十四、缓存策略

### 14.1 为什么绕不过

钱包首页和余额查询是高频操作，每次都查数据库不现实。币种配置数据也不应该每次从数据库读。

### 14.2 实现思路

| 缓存对象 | Key 模式 | 有效期 | 失效策略 |
|---------|---------|--------|---------|
| 用户余额 | `wallet:bal:{userID}:{currCode}` | 30-60s | 余额变更时主动 DEL |
| 币种列表 | `wallet:currencies` | 5-10min | B端编辑/启禁后主动 DEL |
| 币种详情 | `wallet:curr:{currCode}` | 5-10min | 同上 |
| 汇率数据 | `wallet:rate:{currCode}` | 由定时任务覆盖 | 定时任务写入新值 |
| 支付配置 | `wallet:pay:{currCode}:{type}` | 1-5min（可选） | 过期自动失效 |

### 14.3 要考虑的点

- 缓存 key 前缀使用 `wallet:` 命名空间，参照 common/pkg/consts/redis_key 的管理方式
- 余额缓存失效用 DEL 而不是更新值（避免缓存与数据库不一致的窗口）
- 缓存击穿：余额缓存 DEL 后大量请求同时查库，可用 singleflight 模式（或短时间内的 mutex）
- 缓存穿透：查询不存在的用户/币种时不缓存空值（或缓存短时间的空标记）

---

## 十五、幂等性实现

### 15.1 为什么绕不过

所有涉及余额变更的 RPC 接口（CreditWallet/DebitWallet/Freeze/Unfreeze/Confirm 等）都可能被重复调用（网络重试、回调重发）。没有幂等保证，就会重复入账或重复扣款。

### 15.2 实现思路

```
方案：利用 orderNo 做唯一标识

实现方式一（推荐）：在 wallet_flow 表的 order_no 字段加唯一索引
  每次操作前先 INSERT flow 记录（含 order_no）
  → 插入成功：首次处理，继续执行余额变更
  → 唯一键冲突：重复请求，查出上次结果直接返回

实现方式二：独立幂等表
  表结构：idempotent_key(PK) / result_json / created_at
  每次操作前先 INSERT
  → 成功：继续
  → 冲突：返回已有结果

推荐方式一，减少一张表，流水记录本身就是幂等证据。
```

### 15.3 要考虑的点

- orderNo 的生成由**调用方**负责（财务模块生成充值订单号传过来），钱包不生成
- 但钱包自发起的操作（如兑换、冻结），orderNo 由钱包自己用 snowflake 生成
- 幂等返回时，不仅返回"已处理"，还要返回上次的处理结果（成功/失败），调用方需要知道

---

## 十六、事务边界

### 16.1 为什么绕不过

涉及资金的操作要么全做要么全不做。特别是兑换（跨币种双向操作）和冻结（可用/冻结余额互转）。

### 16.2 实现思路

```
原则：只在 ser-wallet 自己的数据库内开事务，不做跨服务分布式事务。

单事务内的场景：
  兑换 → 法币 -= X + BSB += Y + 奖励 += Z + 写流水（同一事务）
  冻结 → 可用 -= X + 冻结 += X + 写流水（同一事务）
  上账 → 余额 += X + 写流水（同一事务）

跨服务的场景（不用分布式事务，用补偿）：
  创建提现订单 → 先冻结（本地事务）→ 调财务创建审核订单
    → 财务调用失败？→ 解冻（本地事务，补偿回滚）

  创建充值订单 → 创建本地订单（本地事务）→ 调财务创建通道订单
    → 财务调用失败？→ 更新本地订单为失败（无需补偿，钱还没加）
```

### 16.3 GORM 事务写法

```go
// 参照已有工程的写法
err := cfg.InitMySQL().Transaction(func(tx *gorm.DB) error {
    // 所有操作使用同一个 tx
    if err := tx.Model(&model.UserWallet{}).
        Where("user_id = ? AND currency_code = ? AND wallet_type = ?", userID, currCode, walletType).
        Update("balance", gorm.Expr("balance - ?", amount)).Error; err != nil {
        return err
    }
    // 写流水
    if err := tx.Create(&model.WalletFlow{...}).Error; err != nil {
        return err
    }
    return nil
})
```

---

## 十七、开发阶段划分

### 17.1 为什么绕不过

不能所有功能并行开发，有明确的依赖顺序。后面的阶段依赖前面阶段的产出。

### 17.2 阶段划分

```
阶段一：脚手架搭建（约 1-2 天）
  ├── 创建 IDL（方法签名，Req/Resp 可以先留基本字段）
  ├── 执行 gen_kitex.sh 生成代码
  ├── 注册 namesp / rpc_client / go.work / ETCD
  ├── 创建数据库表
  ├── 执行 gorm_gen.go 生成 model/query/repo
  ├── 编写 main.go / handler.go / cfg/ / errs/
  ├── 注册 gate-back / gate-font 路由
  └── 验证：服务能启动，能接收 RPC 请求（即使返回空数据）

阶段二：币种配置（无外部依赖）
  ├── B端 CRUD：PageCurrency / EditCurrency / SetBaseCurrency / StatusCurrency
  ├── 汇率日志：PageExchangeRateLog
  ├── 币种缓存机制
  └── 验证：B端后台可以管理币种，C端可以读到币种列表

阶段三：钱包核心 + 兑换流程（无外部依赖）
  ├── 六个原子操作（credit/debit/freeze/unfreeze/confirm + writeFlow）
  ├── 幂等机制
  ├── 余额缓存
  ├── 兑换流程端到端（GetExchangeRate + DoExchange）
  ├── RPC 对外接口（CreditWallet/DebitWallet/GetBalance 等）
  └── 验证：兑换能跑通，外部模块能调 RPC 查余额

阶段四：充值流程（Mock 财务 → 联调替换）
  ├── 充值订单管理
  ├── 重复转账检测
  ├── Mock 财务接口 → 实现充值本地流程
  ├── 与财务联调：替换 Mock 为真实 RPC
  └── 验证：充值全链路跑通

阶段五：提现流程（Mock 财务 → 联调替换）
  ├── 提现账户管理
  ├── KYC 校验集成
  ├── 提现方式三重过滤
  ├── Mock 财务接口 → 实现提现本地流程
  ├── 与财务联调：替换 Mock
  └── 验证：提现全链路跑通

阶段六：记录查询 + 汇率定时 + 收尾
  ├── 记录列表/详情查询
  ├── 汇率定时刷新任务
  ├── B端用户钱包查询
  └── 全面自测
```

---

## 十八、需要提前确认 / 与团队对齐的事项

以下事项会影响实现方式，需要在编码前或编码早期确认：

| 事项 | 影响范围 | 建议决策时间 |
|------|---------|------------|
| 金额存储类型（DECIMAL vs 整数最小单位） | 全部涉及金额的表和计算 | 阶段一（建表前） |
| 错误码编号（022 是否可用） | errs/code.go | 阶段一 |
| ETCD 端口号（8022？） | ETCD 配置 + main.go | 阶段一 |
| 财务模块服务名和 IDL 位置 | 调用财务的全部代码 | 阶段四前 |
| 三方汇率 API 选择和 API Key | 汇率定时任务 | 阶段六前 |
| 充值订单归属（钱包存还是只财务存） | 充值订单表设计 + 记录查询 | 阶段四前 |
| 提现账户存储位置（确认存 ser-wallet） | 提现账户表设计 | 阶段五前 |
| 定时任务方式（ser-cron vs 内置 ticker） | 汇率刷新实现 | 阶段六前 |
| 赠送比例和流水倍数的配置来源 | 兑换+充值奖金逻辑 | 阶段三前 |
| 投注扣款/返奖接口细节 | BetDebit/BetReturn 的比例记录 | 阶段三（定 IDL 时） |

---

## 十九、总结：整体实现思路的核心脉络

```
第一条线：自底向上搭建
  IDL → 代码生成 → 基础设施注册 → 数据库建表 → GORM Gen
  → 服务可启动 → 路由可达

第二条线：由内而外实现业务
  币种配置（地基）→ 钱包核心原子操作（能力层）→ 兑换（内部验证）
  → RPC 对外接口（供他人调）→ 充值/提现（外部联调）→ 记录查询

第三条线：渐进式对接外部
  先 Mock 财务接口 → 本地流程跑通 → 联调时替换为真实 RPC
  已有服务（KYC/用户/日志/S3）→ 直接调用，无需 Mock

贯穿始终的横切关注点：
  幂等（orderNo 唯一索引）
  事务（单库事务，不跨服务）
  缓存（余额 + 币种 + 汇率）
  错误码（规范定义 + i18n）
  审计日志（B端操作必记）
  并发安全（乐观锁防超扣）
```
