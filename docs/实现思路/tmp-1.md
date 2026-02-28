# ser-wallet 整体实现思路

> 梳理时间：2026-02-26
> 定位：粗粒度实现方向，不固化细节，聚焦"绕不过的、必须考虑的"
> 依据：
> - 工程认知：/Users/mac/gitlab/z-readme/工程认知/tmp-1.md（四模块代码级分析）
> - 需求理解：/Users/mac/gitlab/z-readme/需求理解/tmp-1.md
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-1.md
> - 依赖关系：/Users/mac/gitlab/z-readme/依赖关系/tmp-1.md
> - 链路关系：/Users/mac/gitlab/z-readme/链路关系/tmp-1.md
> - 流程思路：/Users/mac/gitlab/z-readme/流程思路/tmp-1.md
> 职责：我负责币种配置 + 多币种钱包后端实现，配合财务模块联调

---

## 目录

1. [实现阶段总览](#一实现阶段总览)
2. [第一步：工程脚手架搭建](#二第一步工程脚手架搭建)
3. [第二步：IDL 接口定义](#三第二步idl-接口定义)
4. [第三步：数据库表设计方向](#四第三步数据库表设计方向)
5. [第四步：服务内部分层架构](#五第四步服务内部分层架构)
6. [第五步：币种配置模块实现](#六第五步币种配置模块实现)
7. [第六步：子钱包余额体系实现](#七第六步子钱包余额体系实现)
8. [第七步：业务子模块实现（兑换→充值→提现）](#八第七步业务子模块实现)
9. [第八步：对外 RPC 接口实现（配合财务联调）](#九第八步对外-rpc-接口实现)
10. [汇率定时任务实现思路](#十汇率定时任务实现思路)
11. [缓存策略](#十一缓存策略)
12. [资金安全绕不过的硬要求](#十二资金安全绕不过的硬要求)
13. [错误码规划](#十三错误码规划)
14. [与财务模块联调的边界约定](#十四与财务模块联调的边界约定)
15. [待评审确认后再定的事项](#十五待评审确认后再定的事项)

---

## 一、实现阶段总览

### 1.1 整体节奏

```
Phase 1  工程脚手架 + IDL + 数据库 + 代码生成
         ↓ 产出：ser-wallet 能启动，RPC 能通，表能建
Phase 2  币种配置模块（CRUD + 定时任务 + 对外查询能力）
         ↓ 产出：B端能配币种，汇率自动更新，RPC可查配置
Phase 3  子钱包余额体系（余额 CRUD + 内部资金操作）
         ↓ 产出：子钱包能初始化，余额能增减冻结
Phase 4  兑换子模块（最简单的业务闭环）
         ↓ 产出：法币→BSB兑换全流程通
Phase 5  充值子模块 + 入账 RPC
         ↓ 产出：充值下单+USDT回调+WalletCredit 通
Phase 6  提现子模块 + 冻结/解冻/扣款 RPC
         ↓ 产出：提现下单+冻结+WalletUnfreeze/Deduct 通
Phase 7  人工加减款 RPC + 记录查询 + 联调收尾
         ↓ 产出：与财务模块全链路联调通过
```

### 1.2 为什么是这个顺序

- **Phase 1 → 2**：币种配置是地基，所有金额操作依赖它的汇率和精度
- **Phase 2 → 3**：子钱包余额表是所有资金变动的载体
- **Phase 3 → 4**：兑换最简单（内部闭环、不涉及三方/审核），用来验证余额体系
- **Phase 4 → 5 → 6**：充值不需要冻结机制，提现需要，复杂度递增
- **Phase 7**：人工加减款和记录依赖前面的数据

---

## 二、第一步：工程脚手架搭建

> 依据：工程认知 第七章（从0到1开发 ser-wallet 完整指南）

### 2.1 需要动的文件和位置

```
要新建的：
/Users/mac/gitlab/ser-wallet/           ← 整个服务目录（参照 ser-app 结构）
/Users/mac/gitlab/common/idl/ser-wallet/ ← IDL 定义

要修改的（往 common 里注册）：
common/pkg/consts/namesp/namesp.go      ← 加 EtcdWalletService/Port/Db 常量
common/rpc/rpc_client.go                ← 加 WalletClient() 工厂函数

要修改的（网关路由）：
gate-back/biz/handler/wallet/           ← B端 handler（币种管理）
gate-back/biz/router/wallet/            ← B端 路由注册
gate-back/biz/router/register.go        ← 加 wallet.RouterWallet(r)
gate-font/biz/handler/wallet/           ← C端 handler（充值/兑换/提现/记录）
gate-font/biz/router/wallet/            ← C端 路由注册
gate-font/biz/router/register.go        ← 加 wallet.RouterWallet(r)

要配置的（ETCD）：
/slg/serv/wallet/port                   ← 服务端口（如 9100）
/slg/serv/wallet/db                     ← 数据库名（如 db_wallet）
```

### 2.2 ser-wallet 服务目录结构

```
/Users/mac/gitlab/ser-wallet/
├── main.go                              # 入口：ETCD/Redis/MySQL 初始化 → Kitex Server 启动
├── handler.go                           # RPC Handler 委托层
├── go.mod                               # 依赖管理（引用 common）
├── internal/
│   ├── cfg/
│   │   ├── mysql.go                     # MySQL sync.Once 初始化
│   │   └── redis.go                     # Redis sync.Once 初始化
│   ├── errs/
│   │   └── code.go                      # 错误码 6022xxx
│   ├── enum/
│   │   ├── wallet.go                    # 子钱包类型、订单状态等枚举
│   │   ├── currency.go                  # 币种类型枚举
│   │   └── ...
│   ├── gen/
│   │   └── gorm_gen.go                  # GORM 模型生成入口
│   ├── gorm_gen/                        # 自动生成（勿手编）
│   │   ├── model/
│   │   └── query/
│   ├── rep/                             # Repository 层
│   │   ├── currency_repo.go
│   │   ├── wallet_repo.go
│   │   ├── deposit_repo.go
│   │   ├── withdraw_repo.go
│   │   ├── exchange_repo.go
│   │   └── ...
│   ├── ser/                             # Service 层
│   │   ├── currency_service.go
│   │   ├── wallet_service.go
│   │   ├── deposit_service.go
│   │   ├── withdraw_service.go
│   │   ├── exchange_service.go
│   │   ├── record_service.go
│   │   └── ...
│   └── cache/                           # Cache 层
│       ├── currency_cache.go
│       └── wallet_cache.go
```

### 2.3 common 侧注册

**namesp.go** 新增：

```go
// ser-wallet (编号022)
EtcdWalletService = "wallet_service"
EtcdWalletPort    = "/slg/serv/wallet/port"
EtcdWalletDb      = "/slg/serv/wallet/db"
```

**rpc_client.go** 新增 `WalletClient()` 工厂函数，模式与现有 `AppClient()` 一致：sync.Once + 惰性单例 + ETCD 服务发现。

### 2.4 网关路由规划

**gate-back（B端）**：

```
POST /admin/api/wallet/currency/page       # 币种列表
POST /admin/api/wallet/currency/create     # 新增币种
POST /admin/api/wallet/currency/update     # 编辑币种
POST /admin/api/wallet/currency/toggle     # 启禁用币种
POST /admin/api/wallet/currency/base       # 设置基准币种
POST /admin/api/wallet/currency/detail     # 币种详情
POST /admin/api/wallet/rate-log/page       # 汇率日志
```

**gate-font（C端）**：

```
POST /api/wallet/currency/list             # 可用币种列表
POST /api/wallet/currency/rate             # 获取汇率
POST /api/wallet/overview                  # 钱包余额概览
POST /api/wallet/deposit/config            # 充值配置
POST /api/wallet/deposit/create            # 创建充值订单
POST /api/wallet/deposit/address           # USDT收款地址
POST /api/wallet/deposit/status            # 充值状态查询
POST /api/wallet/exchange/preview          # 兑换预览
POST /api/wallet/exchange/create           # 执行兑换
POST /api/wallet/withdraw/config           # 提现配置
POST /api/wallet/withdraw/account          # 提现账户查询
POST /api/wallet/withdraw/account/save     # 保存提现账户
POST /api/wallet/withdraw/create           # 创建提现订单
POST /api/wallet/record/page               # 记录分页
POST /api/wallet/record/detail             # 记录详情
```

handler 实现全部用 `rpc.Handler(ctx, c, rpc.WalletClient().XXX)` 通用转发模式，网关不写业务逻辑。

### 2.5 这一步的验证标准

- ser-wallet 能启动，注册到 ETCD
- gate-back / gate-font 的路由能打通（返回参数校验错误即可，说明链路通了）
- `rpc.WalletClient()` 在其他服务中可调用

---

## 三、第二步：IDL 接口定义

> 依据：工程认知 3.2（IDL 规范）+ 接口评估（接口清单）

### 3.1 IDL 文件组织

```
common/idl/ser-wallet/
├── service.thrift          # 主入口：include + WalletService 定义
├── currency.thrift         # 币种配置相关 struct
├── wallet.thrift           # 钱包操作相关 struct（余额/冻结/入账）
├── deposit.thrift          # 充值相关 struct
├── withdraw.thrift         # 提现相关 struct
├── exchange.thrift         # 兑换相关 struct
└── record.thrift           # 记录相关 struct
```

一个 ser 对应一个 `WalletService`，所有方法在同一个 service 块内。按业务域拆分 thrift 文件只是为了可维护性，最终都 include 进 service.thrift。

### 3.2 方法命名遵循工程规范

```
动词 + 资源名，PascalCase

CRUD 标准动词：
  Create / Update / Delete / Toggle(状态切换)
  Page(分页查询) / Detail(单条详情) / List(全量列表) / Get(获取)

示例：
  CreateCurrency / UpdateCurrency / ToggleCurrencyStatus / SetBaseCurrency
  PageCurrency / DetailCurrency / ListActiveCurrencies
  PageRateLog
  GetWalletOverview / GetUserBalance
  CreateDepositOrder / GetDepositAddress / GetDepositConfig
  PreviewExchange / CreateExchangeOrder
  GetWithdrawConfig / CreateWithdrawOrder
  WalletCredit / WalletFreeze / WalletUnfreeze / WalletDeduct
  WalletManualAdd / WalletManualSub
  PageRecord / DetailRecord
```

### 3.3 IDL 编写要点

- 字段加 `(api.body="fieldName")` 注解，用于 Hertz 自动绑定
- `required` 用于必填字段，`optional` 用于可选筛选条件和可空字段
- 分页请求统一带 `pageNo` + `pageSize` 字段
- 金额字段用 `string` 而非 `double`，避免浮点精度问题（或用 `i64` 存分/最小单位）
- 枚举值用 `i32` 传输，在服务端做映射

### 3.4 金额字段类型决策（绕不过的）

```
方案A：string 传输（如 "100.50"）
  优点：前端友好，精度可控
  缺点：服务端每次需要 parse，计算前转 Decimal

方案B：i64 传输最小单位（如 10050 = 100.50 的 100倍）
  优点：整数运算无精度丢失
  缺点：前后端都需要按精度转换

建议：IDL 中金额用 string，服务端内部用 Decimal 库运算，
     落库用 DECIMAL(N,M) 类型，回传前端时按币种精度格式化为 string。
     这样最安全，也与币种配置的精度规则（法币2位/加密8位）一致。
```

### 3.5 这一步完成后执行代码生成

```bash
cd /Users/mac/gitlab/common
bash script/gen_kitex.sh
```

产物：`common/kitex_gen/ser_wallet/walletservice/` 下的客户端和服务端接口。

---

## 四、第三步：数据库表设计方向

> 依据：需求理解（业务规则）+ 工程认知 3.14（GORM Gen 模式）

### 4.1 需要的表（粗粒度）

```
币种配置域：
├── t_currency                # 币种配置表（~20个字段）
└── t_rate_log                # 汇率日志表

钱包核心域：
├── t_wallet_account          # 子钱包账户表（用户×币种×子钱包类型）
├── t_wallet_flow             # 资金流水表（所有余额变动的流水记录）
└── t_freeze_record           # 冻结记录表（提现冻结的生命周期追踪）

充值域：
└── t_deposit_order           # 充值订单表

提现域：
├── t_withdraw_order          # 提现订单表
└── t_withdraw_account        # 提现账户表（用户绑定的银行卡/电子钱包/USDT地址）

兑换域：
└── t_exchange_order          # 兑换订单表
```

### 4.2 关键表的设计方向

#### t_currency（币种配置表）

```
核心字段方向：
  id, currency_code(唯一), currency_name, currency_type(法币/加密/平台)
  country, icon_url, currency_symbol, thousands_separator, decimal_separator
  precision(小数位数), real_rate(实时汇率), platform_rate(平台汇率)
  rate_fluctuation(浮动%), threshold(阈值%), deposit_rate(入款汇率), withdraw_rate(出款汇率)
  is_base(是否基准币种), sort_value, status(启用/禁用)
  create_by, create_at, update_at, delete_at(软删除)

要点：
  - currency_code 建唯一索引
  - is_base 整表只能有一条为 true
  - 汇率相关字段由定时任务维护
  - 所有金额/汇率字段用 DECIMAL 类型
```

#### t_wallet_account（子钱包账户表）

```
核心字段方向：
  id, user_id, currency_code, wallet_type(中心/奖励/主播/代理/场馆)
  available_balance(可用余额), frozen_balance(冻结余额)
  version(乐观锁版本号)
  create_at, update_at

要点：
  - 联合唯一索引：(user_id, currency_code, wallet_type)
  - 用户首次访问某币种时初始化一组账户
  - available_balance 和 frozen_balance 用 DECIMAL
  - version 字段用于乐观锁，每次更新余额时 version+1，防并发
```

#### t_wallet_flow（资金流水表）

```
核心字段方向：
  id, user_id, currency_code, wallet_type
  flow_type(充值入账/奖金入账/兑换扣减/兑换入账/提现冻结/提现解冻/提现扣款/
           人工加款/人工减款/消费扣款/返奖入账)
  amount(变动金额，正数增/负数减)
  balance_after(变动后余额快照)
  related_order_no(关联订单号)
  remark
  create_at

要点：
  - 只插入不更新（流水是不可变的审计记录）
  - related_order_no 索引，用于按订单追溯流水
  - balance_after 用于对账：最后一条流水的 balance_after 应等于当前余额
```

#### t_freeze_record（冻结记录表）

```
核心字段方向：
  id, user_id, currency_code, wallet_type
  freeze_amount, related_order_no(关联提现单号)
  status(冻结中/已解冻/已扣款)
  create_at, update_at

要点：
  - 每次提现产生一条冻结记录
  - status 是状态机：冻结中 → 已解冻 或 已扣款（终态，互斥）
  - WalletUnfreeze/WalletDeduct 操作前必须校验 status=冻结中
```

#### t_deposit_order / t_withdraw_order / t_exchange_order

```
共同字段方向：
  id, order_no(唯一索引), user_id, currency_code
  amount, status, create_at, update_at

充值订单额外字段：
  deposit_type(USDT/银行转账/电子钱包), channel_id
  usdt_amount(USDT换算金额), exchange_rate(使用汇率)
  bonus_activity_id, bonus_amount, network_type(TRC20/ERC20/BEP20)
  tx_hash(链上交易哈希，USDT场景)

提现订单额外字段：
  withdraw_type(银行卡/电子钱包/USDT), account_id
  fee_amount(手续费), actual_amount(到账金额)
  freeze_record_id

兑换订单额外字段：
  source_currency, target_currency(BSB)
  exchange_rate, exchange_amount, bonus_ratio, bonus_amount, actual_amount
```

#### t_withdraw_account（提现账户表）

```
核心字段方向：
  id, user_id, account_type(银行卡/电子钱包/USDT)
  bank_name, bank_account, account_holder, branch(银行卡)
  wallet_type_name, phone(电子钱包)
  chain_address(USDT)
  create_at

要点：
  - 联合唯一索引：(user_id, account_type) — 每种方式只能绑定一个
  - 不可修改（只有首次创建，无 update 操作）
```

### 4.3 订单号生成规则

```
充值：C + 16位    如 C1234567890123456
提现：T + 16位    如 T1234567890123456
补单：B + 16位    (财务模块生成)
加款：A + 16位    (财务模块生成)
减款：M + 16位    (财务模块生成)

实现：可用 Snowflake ID 生成 16 位数字部分，前面拼接前缀字母
或者用时间戳+随机数组合，保证唯一即可
兑换订单号：可自定义前缀（如 E+16位），需求文档未明确规定
```

### 4.4 这一步完成后执行 gorm_gen

建好表后在 ETCD 配置 `/slg/serv/wallet/db` → `db_wallet`，然后：

```bash
cd /Users/mac/gitlab/ser-wallet/internal/gen
go run gorm_gen.go
```

产物：`internal/gorm_gen/model/` 和 `internal/gorm_gen/query/` 下的自动生成代码。

---

## 五、第四步：服务内部分层架构

> 依据：工程认知 第六章（ser-app 范本解析）

### 5.1 严格遵循四层架构

```
handler.go (RPC Handler)
    │ 职责：接收 RPC 请求 → 委托给 Service → 返回结果
    │ 规则：一行委托，不写业务逻辑
    ▼
internal/ser/ (Service 层)
    │ 职责：业务逻辑核心 — 校验→检查→转换→持久化→缓存→日志
    │ 规则：
    │   - defer 失败日志（方法返回前自动记录）
    │   - 参数校验独立函数 validateXxx()
    │   - 错误用 ret.BizErr(ctx, errCode, msg) 包装
    │   - 操作人用 tracer.GetUserName(ctx) / tracer.GetUserId(ctx)
    │   - 写操作后清缓存 + 调 ser-blog 审计日志
    ▼
internal/rep/ (Repository 层)
    │ 职责：数据访问 — 封装 GORM 查询，支持事务
    │ 规则：
    │   - 所有查询加 .Where(xxx.DeleteAt.Eq(0)) 软删除过滤
    │   - 不存在返回 nil, nil（不是 error）
    │   - 事务用 query.Q.Transaction(func(tx *query.Query) error { ... })
    │   - 更新用 .Select(fields...).Updates(model) 显式字段
    ▼
internal/gorm_gen/ (Query + Model)
    │ 自动生成，不手编
    ▼
Database (TiDB)
```

### 5.2 Service 划分

```
CurrencyService      负责：币种 CRUD + 汇率查询 + 换算计算
WalletService        负责：子钱包初始化 + 余额查询 + 入账/冻结/解冻/扣款/加减款
DepositService       负责：充值配置 + 创建订单 + USDT地址 + 状态查询 + 回调
WithdrawService      负责：提现配置 + 提现账户管理 + 创建订单
ExchangeService      负责：兑换预览 + 执行兑换
RecordService        负责：记录分页 + 详情

WalletService 是核心，其他 Service 在执行资金变动时调用 WalletService 的方法。
例如 ExchangeService.CreateExchangeOrder 内部调用 WalletService 的扣减和入账方法。
```

### 5.3 main.go 中的依赖注入

```go
// 伪代码示意（非最终代码）
currencyCache := cache.NewCurrencyCache()
walletCache := cache.NewWalletCache()

currencyService := ser.NewCurrencyService(currencyCache)
walletService := ser.NewWalletService(walletCache, currencyService)
depositService := ser.NewDepositService(walletService, currencyService)
withdrawService := ser.NewWithdrawService(walletService, currencyService)
exchangeService := ser.NewExchangeService(walletService, currencyService)
recordService := ser.NewRecordService(currencyService)

srv := walletservice.NewServer(
    &WalletServiceImpl{
        currencyService: currencyService,
        walletService:   walletService,
        depositService:  depositService,
        withdrawService: withdrawService,
        exchangeService: exchangeService,
        recordService:   recordService,
    },
    rpc.InitRpcServerParams(namesp.EtcdWalletService, port)...,
)
```

CurrencyService 被所有其他 Service 依赖（因为需要汇率和精度），这是同进程内的方法调用，不走网络。

---

## 六、第五步：币种配置模块实现

### 6.1 绕不过的实现内容

```
B端接口（7个）：
  PageCurrency / CreateCurrency / UpdateCurrency / ToggleCurrencyStatus
  SetBaseCurrency / DetailCurrency / PageRateLog

C端接口（2个）：
  ListActiveCurrencies / GetExchangeRate

RPC对外（3个）：
  GetCurrencyConfig / CalcExchangeAmount / GetPlatformRate

定时任务（1个）：
  汇率偏差检测与更新
```

### 6.2 实现要点

**CreateCurrency / UpdateCurrency**
- 图标上传通过 `rpc.S3Client().UploadFile()` → 拿回 URL 存表
- 币种代码唯一性校验（CreateCurrency 时查表）
- 币种代码和类型创建后不可修改（UpdateCurrency 校验）

**SetBaseCurrency**
- 查表确认是否已设置过，已设置直接返回错误
- 一次性写入后不可修改
- BSB 与 USDT 的 10:1 锚定关系是硬编码逻辑，不进配置

**ToggleCurrencyStatus**
- 基准币种不允许禁用
- 禁用后 ListActiveCurrencies 不返回该币种

**ListActiveCurrencies / GetExchangeRate / GetCurrencyConfig / CalcExchangeAmount**
- 纯查询+计算，无状态修改
- 被调用频率高，必须做缓存（见缓存策略章节）
- CalcExchangeAmount 是统一换算入口，所有涉及跨币种计算的地方都要走它，不能各处自行计算

### 6.3 审计日志

币种配置的所有写操作（Create/Update/Toggle/SetBase）都要调 `rpc.BLogClient().AddActLog()`，参数模式：

```go
rpc.BLogClient().AddActLog(ctx, &ser_blog.ActAddReq{
    Platform:   "运营后台",
    Module:     "钱包管理",
    Page:       "币种配置",
    Action:     "新增币种（XXX）",
    HandleType: "成功",
})
```

oneway 异步，不阻塞主流程。

---

## 七、第六步：子钱包余额体系实现

### 7.1 这是钱包模块的心脏

所有充值/兑换/提现/加减款，最终都是对 `t_wallet_account` 表做余额增减。这个体系必须先搭好。

### 7.2 账户初始化流程

```
用户首次访问某币种的钱包 → 检查 t_wallet_account 表是否有该用户+该币种的记录
  无记录 → 初始化创建3条（当前实现）或5条（含预留）账户记录：
    (user_id, currency_code, center,  0, 0)
    (user_id, currency_code, reward,  0, 0)
    (user_id, currency_code, streamer, 0, 0)
    # 代理和场馆预留，评审后决定是否创建
```

### 7.3 余额变动的核心方法（WalletService 内部）

```
所有余额变动归纳为以下原子操作，其他 Service 通过调用这些方法完成资金变动：

1. credit(用户, 币种, 子钱包类型, 金额, 关联单号, 流水类型)
     可用余额 += 金额，写流水

2. debit(用户, 币种, 子钱包类型, 金额, 关联单号, 流水类型)
     校验可用余额 ≥ 金额，可用余额 -= 金额，写流水

3. freeze(用户, 币种, 子钱包类型, 金额, 关联单号)
     校验可用余额 ≥ 金额
     可用余额 -= 金额，冻结余额 += 金额
     创建冻结记录，写流水

4. unfreeze(冻结记录ID)
     校验冻结记录状态=冻结中
     冻结余额 -= 金额，可用余额 += 金额
     更新冻结记录状态=已解冻，写流水

5. deductFrozen(冻结记录ID)
     校验冻结记录状态=冻结中
     冻结余额 -= 金额（最终扣除）
     更新冻结记录状态=已扣款，写流水
```

### 7.4 并发安全（绕不过的）

```
问题：两个请求同时修改同一用户的余额，可能导致数据不一致

方案选择（二选一）：

方案A：乐观锁（version 字段）
  UPDATE t_wallet_account
  SET available_balance = available_balance + ?, version = version + 1
  WHERE id = ? AND version = ?
  如果 affected_rows = 0 → 重试（说明被其他请求先更新了）

方案B：数据库行级锁（SELECT FOR UPDATE）
  在事务中 SELECT ... FOR UPDATE 锁定行
  更新余额
  事务提交释放锁

建议方案A（乐观锁），因为金额操作的并发冲突概率不高，
乐观锁的整体性能更好，重试成本可控。
```

---

## 八、第七步：业务子模块实现

### 8.1 兑换（先做，最简单）

```
需要实现：
  PreviewExchange  — 试算（纯计算，不写DB）
  CreateExchangeOrder — 执行（事务：法币减+BSB加+写流水+写订单）

内部调用：
  CurrencyService.GetExchangeRate()
  CurrencyService.CalcExchangeAmount()
  WalletService.debit()  — 扣减法币
  WalletService.credit() — 增加BSB
  WalletService.credit() — 赠送BSB入奖励钱包（如有赠送）

事务要求：
  法币扣减 + BSB入账 + 订单写入 必须在同一个DB事务中
  任一步失败整体回滚

关键校验：
  - 源币种必须是法币（VND/IDR/THB），目标固定BSB
  - 金额 ≤ 法币中心钱包可用余额
  - 执行时必须重新计算汇率（不能用预览时的快照）
```

### 8.2 充值（中等复杂度）

```
需要实现：
  GetDepositConfig    — 聚合接口（可用方式+汇率+奖金活动）
  CreateDepositOrder  — 创建订单（含重复转账拦截）
  GetDepositAddress   — USDT收款地址
  QueryDepositStatus  — 状态查询（C端轮询）
  ListAvailableBonus  — 奖金活动列表

需要实现的RPC（供finance和区块链回调）：
  WalletCredit       — 入账到中心钱包
  WalletCreditReward — 奖金入账到奖励钱包

重复转账拦截逻辑（CreateDepositOrder 内部）：
  查 t_deposit_order：同 user_id + 同 currency + 同 amount + 状态=待支付/进行中
  0笔 → 正常
  1-2笔 → 返回软拦截标记（前端弹窗警告，允许继续）
  ≥3笔 → 返回硬拦截标记（前端弹窗禁止）
  仅银行转账和电子钱包适用，USDT跳过拦截

USDT充值收款地址（GetDepositAddress）：
  技术方案待定（地址池 vs 链上动态生成）
  这里可能需要对接区块链服务，具体方案评审后确定

WalletCredit / WalletCreditReward 的幂等实现：
  以 related_order_no 为幂等key
  入账前查 t_wallet_flow：该订单号是否已有入账流水
  有 → 直接返回成功
  无 → 正常执行入账
```

### 8.3 提现（最复杂）

```
需要实现：
  GetWithdrawConfig    — 聚合接口（KYC+充值历史+方式配置+限额）
  GetWithdrawAccount   — 查询已绑定提现账户
  SaveWithdrawAccount  — 首次保存提现账户（KYC姓名校验）
  CreateWithdrawOrder  — 创建订单 + 冻结（事务）

需要实现的RPC（供finance调用）：
  WalletFreeze   — 冻结
  WalletUnfreeze — 解冻
  WalletDeduct   — 最终扣款

KYC 对接：
  rpc.KycClient().GetUserKycStatus()  — 查认证状态
  rpc.KycClient().GetUserKycInfo()    — 获取认证姓名（比对持有人）
  （KycClient 在 common/rpc/rpc_client.go 中已注册）

提现币种规则（GetWithdrawConfig 内部）：
  查用户充值历史，确定"从哪来回哪去"：
  仅法币充值 → 法币提现
  仅USDT充值 → USDT提现
  混合充值 → 法币提现
  法币→兑换BSB → 追溯为法币提现

CreateWithdrawOrder 事务：
  限额校验 → 创建订单(T+16位) → WalletFreeze冻结
  三步在同一个事务中

通知财务审核的机制（待与财务对齐）：
  方案A: 事务提交后RPC调 finance 的审核入口
  方案B: 事务提交后发消息队列（Kafka/NATS）
```

---

## 九、第八步：对外 RPC 接口实现

> 这是财务模块联调的核心接口，必须提供

### 9.1 接口清单和调用方

```
┌─────────────────────┬────────────────────────────────────────┐
│ RPC 接口             │ finance 侧调用场景                       │
├─────────────────────┼────────────────────────────────────────┤
│ WalletCredit        │ 充值成功/补单通过 → 入中心钱包              │
│ WalletCreditReward  │ 充值奖金/活动奖励 → 入奖励钱包             │
│ WalletFreeze        │ 提现订单创建 → 冻结                       │
│ WalletUnfreeze      │ 审核驳回/出款失败 → 解冻归还                │
│ WalletDeduct        │ 出款成功 → 最终扣款                       │
│ WalletManualAdd     │ 人工加款审核通过 → 按子钱包分别加款          │
│ WalletManualSub     │ 人工减款审核通过 → 按子钱包分别减款          │
│ GetUserBalance      │ 人工加减款弹窗 → 展示各子钱包余额           │
│ GetCurrencyConfig   │ 金额格式化 → 获取精度和符号                 │
└─────────────────────┴────────────────────────────────────────┘
```

### 9.2 WalletManualAdd / WalletManualSub 特殊处理

```
入参设计方向：
  user_id, currency_code
  center_amount, reward_amount, streamer_amount, agent_amount  (各子钱包独立)
  related_order_no (修正单号，A+16位或M+16位)
  correction_type (补偿/活动补发/财务冲正/费用调整等)

WalletManualAdd：各子钱包独立加款（金额>0的才操作）
WalletManualSub：各子钱包独立减款
  关键：实际扣款 = min(期望扣款, 当前余额)
  返回值必须包含每个子钱包的实际扣款金额（可能与期望不同）
```

### 9.3 联调对接模式

```
ser-wallet 作为被调用方，接口定义主动权在我这边。
与财务模块联调时：

1. 我先定义好 IDL，生成代码
2. 财务模块通过 rpc.WalletClient() 调用我
3. 我提供接口文档（每个接口的入参/出参/错误码/幂等说明）
4. 先用 mock 数据跑通链路，再对接真实数据
```

---

## 十、汇率定时任务实现思路

### 10.1 部署方式

```
方案A：放在 ser-wallet 进程内
  用 Go 的 cron 库（如 robfig/cron）在 main.go 启动时注册定时任务
  优点：简单，不需要额外服务
  缺点：如果 ser-wallet 多实例部署，需要分布式锁防重复执行

方案B：放在 ser-cron 定时任务服务
  项目中已有 ser-cron 服务，定时任务统一管理
  ser-cron 通过 RPC 调用 ser-wallet 的汇率更新接口
  优点：与工程现有模式一致
  缺点：需要额外暴露一个汇率更新 RPC 接口

建议方案A + 分布式锁（Redis SETNX），因为汇率更新逻辑与币种数据紧密耦合，
放在同进程内减少网络开销和故障点。
```

### 10.2 三方汇率 API 对接

```
需要对接3家汇率API取平均值。

容错策略：
  3家都成功 → 取平均
  2家成功 → 取2家平均
  1家成功 → 直接用该值（降级）
  0家成功 → 跳过本轮，不更新，记告警日志

API 凭证和地址建议放 ETCD：
  /slg/conf/rate/api1_url
  /slg/conf/rate/api1_key
  /slg/conf/rate/api2_url
  ...
  /slg/conf/rate/cron_interval  (更新间隔，如 "60s")
```

---

## 十一、缓存策略

### 11.1 需要缓存的数据

```
高频读、低频写 → 缓存收益大：

├── 币种配置列表（ListActiveCurrencies）
│     Redis Key: wallet:currency:active_list
│     TTL: 较长（如10分钟），币种CRUD后主动失效
│
├── 单币种配置（GetCurrencyConfig）
│     Redis Key: wallet:currency:config:{code}
│     TTL: 较长，CRUD后失效
│
├── 平台汇率（GetExchangeRate/CalcExchangeAmount 内部使用）
│     Redis Key: wallet:rate:{code}
│     TTL: 与定时任务更新间隔一致，定时任务更新后主动刷新
│
└── 额外奖金活动列表（ListAvailableBonus）
      如果活动数据不常变 → 缓存
      如果经常调整 → 视情况决定
```

### 11.2 不适合缓存的数据

```
├── 用户余额（实时性要求高，余额变动频繁）
│     不缓存，直接查DB
│     如果性能需要，可考虑 Redis 做余额缓存，但需要严格的缓存一致性方案
│
├── 订单数据（写多读少的业务数据）
│     不缓存
│
└── 冻结记录（操作频率不高，实时性要求高）
      不缓存
```

### 11.3 缓存失效策略

```
写操作后主动失效（与 ser-app 的 cache.DelAll() 模式一致）：

CreateCurrency / UpdateCurrency / ToggleCurrencyStatus
  → 清除币种列表缓存 + 对应币种配置缓存

汇率定时任务更新平台汇率后
  → 清除对应币种的汇率缓存

空结果缓存：
  查询结果为空时也要缓存（防缓存穿透）
  TTL 可短一些
```

---

## 十二、资金安全绕不过的硬要求

### 12.1 幂等性

```
必须幂等的接口：
  WalletCredit / WalletCreditReward / WalletManualAdd / WalletManualSub
  DepositNotify（USDT链上回调）

实现方式：
  每个接口以 related_order_no（关联订单号/修正单号/链上交易ID）为幂等 key
  执行前查 t_wallet_flow：该 key 是否已有对应流水
  有 → 返回上次结果（不重复执行）
  无 → 正常执行

  额外保险：订单状态也做检查
  已完成状态的充值订单不再接受入账
```

### 12.2 事务原子性

```
必须在同一个DB事务中的操作组：

1. 兑换：法币子钱包扣减 + BSB子钱包入账 + 订单写入 + 流水写入
2. 提现创建：提现订单写入 + 冻结操作 + 流水写入
3. 所有入账：余额增加 + 订单状态更新 + 流水写入
4. 冻结/解冻/扣款：冻结余额变动 + 可用余额变动 + 冻结记录状态 + 流水写入
5. 人工加减款：多子钱包余额变动 + 流水写入

使用 GORM 事务：
  query.Q.Transaction(func(tx *query.Query) error {
      // 所有DB操作使用 tx 而非 query.Q
      return nil
  })
```

### 12.3 流水与余额一致性

```
原则：任意时刻，t_wallet_account 的 available_balance
      应等于 t_wallet_flow 中该账户所有流水的净值之和

做法：
  每次余额变动必须同时写一条流水记录
  流水中记录 balance_after（变动后余额快照）
  可定期用脚本对账：sum(flow.amount) == account.available_balance
```

### 12.4 防双花（提现冻结机制）

```
提现发起 → 立即冻结 → 可用余额减少 → 审核期间用户无法消费冻结的钱
审核通过+出款成功 → WalletDeduct 最终扣除
审核驳回/出款失败 → WalletUnfreeze 解冻归还

冻结记录状态机必须严格：
  冻结中 → 已解冻（不可逆）
  冻结中 → 已扣款（不可逆）
  已解冻/已扣款 → 不可再操作

WalletUnfreeze 和 WalletDeduct 操作前必须校验 status=冻结中
```

### 12.5 金额精度

```
所有金额运算使用 Decimal 库（如 shopspring/decimal）
不使用 float64 做金额计算

数据库字段用 DECIMAL(20,8) 或按需求精度
  法币字段可用 DECIMAL(20,2)
  加密货币字段用 DECIMAL(20,8)

统一用 CalcExchangeAmount 做汇率换算（不各处自行计算）
最终展示前按 GetCurrencyConfig 的精度规则格式化
```

---

## 十三、错误码规划

> 依据：工程认知 3.8（错误码体系）

### 13.1 ser-wallet 错误码段

```
服务编号 022 → 基础码 6022000

按模块分段：
  6022000 ~ 6022099  币种配置模块
  6022100 ~ 6022199  钱包核心（余额/冻结/入账）
  6022200 ~ 6022299  充值模块
  6022300 ~ 6022399  提现模块
  6022400 ~ 6022499  兑换模块
  6022500 ~ 6022599  记录模块
```

### 13.2 典型错误码示例

```go
// internal/errs/code.go

// 币种配置 6022000
const (
    CurrencyBaseError          = 6022000 + iota
    CurrencyCodeAlreadyExist   // 6022001 币种代码已存在
    CurrencyNotFound           // 6022002 币种不存在
    BaseCurrencyAlreadySet     // 6022003 基准币种已设置
    BaseCurrencyCannotDisable  // 6022004 基准币种不能禁用
    CurrencyCodeNotEditable    // 6022005 币种代码不可修改
)

// 钱包核心 6022100
const (
    WalletBaseError            = 6022100 + iota
    InsufficientBalance        // 6022101 余额不足
    FreezeRecordNotFound       // 6022102 冻结记录不存在
    FreezeAlreadyProcessed     // 6022103 冻结已处理（解冻或扣款）
    DuplicateCredit            // 6022104 重复入账（幂等拦截）
    WalletAccountNotInit       // 6022105 钱包账户未初始化
)

// 充值 6022200
const (
    DepositBaseError           = 6022200 + iota
    DepositAmountTooLow        // 6022201 充值金额低于最低限额
    DepositHardBlock           // 6022202 重复转账硬拦截
    DepositSoftBlock           // 6022203 重复转账软拦截
    DepositOrderNotFound       // 6022204 充值订单不存在
    BonusThresholdNotMet       // 6022205 未达奖金最低充值门槛
)

// 提现 6022300
const (
    WithdrawBaseError          = 6022300 + iota
    WithdrawAmountBelowMin     // 6022301 低于单笔最低
    WithdrawAmountExceedMax    // 6022302 超出单笔最高
    WithdrawDailyLimitExceed   // 6022303 超出单日限额
    WithdrawAccountNotFound    // 6022304 提现账户不存在
    WithdrawAccountExists      // 6022305 提现账户已存在(不可重复绑定)
    KycNameMismatch            // 6022306 KYC姓名与持有人不一致
    WithdrawMethodNotAllowed   // 6022307 提现方式不可用(KYC/充值方式限制)
)

// 兑换 6022400
const (
    ExchangeBaseError          = 6022400 + iota
    ExchangeOnlyFiatToBSB      // 6022401 仅支持法币→BSB
    ExchangeExceedBalance      // 6022402 超出可兑换余额
    ExchangeRateChanged        // 6022403 汇率已变化(可选)
)
```

---

## 十四、与财务模块联调的边界约定

### 14.1 职责边界

```
┌──────────────────────────────┬──────────────────────────────┐
│ 我的职责（ser-wallet）         │ 财务的职责（ser-finance）       │
├──────────────────────────────┼──────────────────────────────┤
│ 币种配置、汇率管理              │ 支付通道配置                   │
│ 子钱包余额增减冻结             │ 充值/提现方式配置（限额/手续费）  │
│ 充值订单创建                   │ 通道选择+三方代收               │
│ USDT收款地址+链上回调           │ 三方代收/代付回调处理           │
│ 兑换全流程                    │ 风控审核+财务审核               │
│ 提现订单创建+冻结              │ 代付出款                      │
│ 记录查询                     │ 人工修正（补单/加款/减款）审核    │
│ 提供资金操作RPC给finance       │ 统计报表                      │
├──────────────────────────────┼──────────────────────────────┤
│ 被动：接受finance的入账/冻结    │ 主动：调wallet的RPC执行资金变动  │
│ /解冻/扣款/加减款RPC调用        │                              │
└──────────────────────────────┴──────────────────────────────┘
```

### 14.2 需要与财务对齐的接口契约

```
finance调wallet（我定义，他调用）：
  ✅ 我控制IDL定义和实现
  ✅ 我提供入参/出参/错误码/幂等说明

wallet调finance（待确认方向）：
  ❓ 充值时获取可用支付方式+通道 → 是wallet调finance？还是finance推配置给wallet？
  ❓ 提现创建后通知finance审核 → 是RPC直调？还是消息队列？
  ❓ 充值/提现方式的限额/手续费配置 → 存在哪边的库？谁维护？

  这些需要评审后与财务负责人对齐。
  当前策略：先把wallet侧能独立实现的部分做完，
  待确认的调用方向用接口预留，不硬编码。
```

### 14.3 联调顺序建议

```
第一批联调：GetCurrencyConfig + GetUserBalance
  → 最简单的查询接口，验证RPC链路是否通

第二批联调：WalletCredit + WalletCreditReward
  → 充值入账链路，验证资金操作

第三批联调：WalletFreeze + WalletUnfreeze + WalletDeduct
  → 提现冻结链路，验证状态机

第四批联调：WalletManualAdd + WalletManualSub
  → 人工修正链路
```

---

## 十五、待评审确认后再定的事项

### 15.1 评审前可以先做的（确定性高）

```
✅ 工程脚手架搭建（Phase 1）
✅ 币种配置 CRUD + 汇率定时任务（Phase 2）
✅ 子钱包余额体系（Phase 3）
✅ 兑换全流程（Phase 4）
✅ 对外RPC接口的IDL定义和基本实现
✅ 错误码规划和定义
```

### 15.2 评审后才能完全确定的

```
❓ wallet ↔ finance 充值时的调用方向
     当前影响：CreateDepositOrder 的分发逻辑暂时预留

❓ 充值/提现方式配置数据归属
     当前影响：GetDepositConfig / GetWithdrawConfig 的数据来源暂时预留

❓ 提现创建后通知财务的机制（RPC vs MQ）
     当前影响：CreateWithdrawOrder 通知部分暂时预留

❓ USDT链上充值回调归属（wallet直接处理 vs 经finance中转）
     当前影响：DepositNotify 接口的暴露位置

❓ 代理钱包和场馆钱包是否本期实现
     当前影响：子钱包初始化时创建几条记录、ManualAdd/Sub 支持几种类型

❓ 快捷金额档位、奖金比例、流水倍数、限额手续费等具体数值
     当前影响：不影响接口设计，只影响校验逻辑中的阈值

❓ 额外奖金活动的数据来源（本地配置 vs 活动模块RPC）
     当前影响：ListAvailableBonus 的实现方式
```

### 15.3 策略：确定的先做，不确定的预留接口

```
不确定的部分，在 Service 层预留方法签名和接口，
内部实现暂用 TODO 标记或返回 mock 数据。
评审确认后再填充具体逻辑。

这样不会阻塞确定部分的开发进度，
也不会因为过早做决定而在评审后返工。
```

---

## 总结：实现思路核心脉络

```
1. 脚手架：IDL → gen_kitex.sh → 建表 → gorm_gen → 服务能启动、RPC能通

2. 地基先行：币种配置先做完，因为所有金额操作都依赖它的汇率和精度

3. 余额体系是心脏：t_wallet_account 表 + credit/debit/freeze/unfreeze/deductFrozen
   五个原子操作，所有业务子模块都调用它们

4. 自底向上实现业务：兑换（内部闭环验证）→ 充值（对接三方）→ 提现（最复杂）

5. 资金安全是红线：幂等、事务、乐观锁、流水对账，一个不能少

6. 确定的先做，不确定的预留：评审前聚焦coin币种配置+余额体系+兑换+RPC接口定义，
   评审后补充充值/提现的三方对接细节和finance联调方向

7. 与财务的接口契约由我定义：IDL在我这边，我先定义好入参/出参/错误码/幂等规则，
   财务模块通过 rpc.WalletClient() 调用
```
