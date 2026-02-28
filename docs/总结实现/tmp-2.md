# ser-wallet 钱包模块 — 从0到1综合实现方案（终极版）

> 定位：从0到1负责钱包模块后端开发，在需求评审大会上向部门负责人全方位展示对模块的理解认知
> 目标：设计思路、表结构、接口、技术栈、技术挑战、待确认问题、开发计划、风险应对 — 一网打尽
> 阅读方式：全文按汇报节奏编排（what → how → risks），可作为评审讲稿使用
> 数据来源：需求文档(152张原型) + 工程代码(22个服务) + 54份分析文档(55000+行)
> 版本说明：整合 c-5 系列(9份8567行) + tmp-2 系列(20份19700行) + 接口统计(887行) 全维度精华

---

# 开场定位（30秒说清楚）

我负责 **ser-wallet** 的从0到1实现。一句话概括：

**管用户的钱 — 钱怎么进来（充值）、怎么流转（兑换）、怎么出去（提现）、怎么查（记录）、怎么配（币种/汇率）。**

ser-wallet 在平台中的角色类似"支付宝的余额模块" — 不直接对接支付渠道（那是财务模块的事），但管理用户的资金账本。

```
核心规模数据
═══════════════════════
接口总量      46 个（19 C端 + 6 B端 + 12 RPC暴露 + 3 定时任务 + 6 RPC调出）
数据库表      11 张（4 币种配置 + 7 钱包）
IDL 文件      7 个（1 service + 6 entity）
代码文件      71 个（40 手写 + 31 自动生成）
新技术引入    0 个（全部复用现有技术栈）
开发分期      4 期（从独立到依赖）
与财务依赖    我调他 4 个 RPC，他调我 6 个 RPC
```

---

---

# 第一章：模块定位与设计哲学

## 1.1 业务定位

从 C 端用户视角：

```
打开钱包 → 看余额（5 币种 × 4 子钱包）
充值     → 法币/USDT 进钱包
兑换     → 法币/USDT 换成 BSB（平台币）
提现     → BSB/法币 出钱包
查记录   → 充值/提现/兑换/奖励的流水
```

从 B 端运营视角：

```
配币种   → 新增/编辑/启禁用币种（VND/IDR/THB/USDT/BSB）
配汇率   → 设置"1 BSB = X 法币"，定时从三方 API 拉取实时汇率
看日志   → 操作日志、汇率变更日志
查用户钱包 → 客服/风控场景查看用户余额和流水
```

## 1.2 五大核心概念

**多币种**：VND（越南盾）、IDR（印尼盾）、THB（泰铢）、USDT（加密货币）、BSB（平台基准币种）。BSB 是平台内部统一结算单位，用户充值法币/USDT 后兑换 BSB 用于消费。

**多子钱包**：每用户每币种下 4 个子钱包 — 中心钱包（主账户，充值/兑换/消费）、奖励钱包（活动奖金/充值赠送）、代理钱包（佣金）、主播钱包（打赏收入）。余额矩阵 = **5币种 × 4子钱包 = 最多20个余额记录**。

**冻结模型**：提现不直接扣钱，先"冻结"（available→frozen），审核通过出款成功后才正式扣除（frozen减），审核驳回则解冻退回（frozen→available）。三步操作：Freeze → DeductFrozen/Unfreeze。

**最小单位**：所有金额用 BIGINT 存储最小货币单位整数（如 10.50 BSB → 存 1050），全链路无浮点运算，零精度丢失。

**汇率快照**：订单创建时锁定当前汇率，后续汇率变更不影响已创建订单。历史订单按创建时汇率审计。

## 1.3 职责边界

```
┌──────────────────────────────────────────────┐
│  我负责（主责，完整实现）                       │
│  ① 币种配置模块（B端）：6个B端接口 + 内置RPC    │
│  ② 钱包模块（C端+RPC）：19个C端 + 12个RPC暴露   │
│  合计：46个方法                                │
├──────────────────────────────────────────────┤
│  他人负责，我配合联调                           │
│  ③ 财务模块（B端）：~53个接口                   │
│  我调财务 4 个 RPC，财务调我 6 个 RPC            │
│  双向依赖，必须协同                             │
└──────────────────────────────────────────────┘
```

## 1.4 设计哲学 — 四条主线

```
安全第一：乐观锁防超扣 + 双层幂等防重复 + 事务铁律防丢失 + BIGINT防精度丢失
简单可靠：单服务不拆分，零新组件引入，本地补偿不用分布式事务框架
粗粒度：通用RPC方法 + opType参数区分场景，参照项目现有模式
从独立到依赖：先做不依赖别人的，再做别人依赖我的，最后做我依赖别人的
```

---

---

# 第二章：技术架构设计

## 2.1 架构决策总览

| 决策项 | 选择 | 一句话理由 |
|--------|------|-----------|
| 服务形态 | **单服务** ser-wallet | ser-buser(31方法)、ser-live(36方法)都是单服务，46方法同一量级 |
| 分层架构 | **四层**：Handler→Service→Rep→Model | 照搬 ser-item、ser-buser 成熟范式 |
| 金额存储 | **BIGINT 最小单位整数** | Stripe/支付宝同款，避免浮点精度问题 |
| 并发控制 | **乐观锁**（CAS 条件更新） | 无死锁、并发度高、单条 SQL 原子完成 |
| 幂等保障 | **两层防护**（应用层 + DB唯一键） | 防重复入账/扣款，金融级底线 |
| 缓存策略 | 币种配置 **Cache-Aside**，余额**不缓存** | 配置读多写少适合缓存，余额必须实时准确 |
| 分布式事务 | **不用框架**，本地事务 + 补偿 | 两服务两步操作，Seata/DTM杀鸡用牛刀 |
| 新组件引入 | **零** | common/pkg 完全覆盖所有需求，不需 Kafka/ES/Seata |

## 2.2 为什么是单服务不拆分

有人可能会问：币种配置和钱包功能不同，要不要拆两个服务？**不拆。三个理由：**

**数据内聚**：余额操作需要实时读取币种参数（精度/限额/手续费），同服务内走本地方法调用，不走RPC，减少一次网络开销和一个故障点。

**事务需要**：兑换涉及两个币种余额变动（扣法币 + 加BSB），必须同一数据库事务。拆了就变分布式事务，复杂度暴增。

**工程一致**：ser-buser（31方法，管菜单+角色+用户+认证）、ser-app（19方法，管banner+策略+分类+S3+枚举）都是单服务多职责。

## 2.3 四层架构

```
┌───────────────────────────────────────────────────────┐
│  Handler 层（handler.go）                               │
│  一行委托，不写任何业务逻辑，参照 ser-item/handler.go      │
├───────────────────────────────────────────────────────┤
│  Service 层（internal/ser/*.go，8个文件）                │
│  业务编排中心 — 校验、调RPC、计算、发起事务               │
│  currency / rate / balance / deposit / withdraw /      │
│  exchange / record / rpc_service                       │
├───────────────────────────────────────────────────────┤
│  Repository 层（internal/rep/*.go）                     │
│  数据访问 — 复杂SQL、事务内多表操作                       │
│  基于 GORM Gen query 扩展自定义方法                      │
├───────────────────────────────────────────────────────┤
│  Model 层（internal/gorm_gen/model/*.go）               │
│  自动生成，不手写                                        │
└───────────────────────────────────────────────────────┘
```

**层间调用铁律**：
- Handler 只调 Service
- Service 可以互调（如 DepositService 调 BalanceService）
- Service 调 Rep（数据访问）和外部 RPC
- **Rep 不调 RPC** — 数据层不能有远程调用
- **核心：RPC 在事务之外做，DB 操作在事务之内做**

## 2.4 技术栈 — 全部复用，零引入

| 需求 | 用什么 | 来源 |
|------|--------|------|
| RPC 框架 | Kitex + Thrift IDL + gen_kitex.sh | 已有 |
| HTTP 网关 | Hertz（gate-back + gate-font） | 已有 |
| 数据库 | TiDB + GORM + Gen + UserPlugin | 已有 |
| 缓存 | Redis（common/pkg/rds/） | 已有 |
| 配置/注册 | ETCD（etcd.LoadAndWatch） | 已有 |
| 订单号生成 | Snowflake（utils.GenerateSnowflakeID） | 已有 |
| 敏感数据加密 | AES EncryptedString（utils/crypto_util） | 已有 |
| 错误码 | ret + errs + i18n | 已有 |
| 审计日志 | ser-blog RPC（AddActLog，oneway） | 已有 |
| 链路透传 | tracer（metainfo 8 个 Header） | 已有 |

**不需要 Kafka**（没有异步消息场景）、**不需要 ES**（没有全文搜索）、**不需要 Seata/DTM**（本地补偿够用）。

## 2.5 代码目录结构

```
ser-wallet/
├── main.go                         启动入口
├── handler.go                      Kitex Handler（46方法，一行委托）
├── sql/
│   ├── 01.currency.sql             币种配置模块DDL（4张表）
│   └── 02.wallet.sql               钱包模块DDL（7张表）
└── internal/
    ├── cfg/                        DB/Redis 初始化
    ├── errs/code.go                错误码 6022xxx
    ├── enum/                       枚举定义（子钱包类型/订单状态/变动类型）
    ├── gen/gorm_gen.go             GORM Gen 代码生成入口
    ├── gorm_gen/                   ══ 自动生成 ══
    │   ├── model/                  表结构体（11个文件）
    │   └── query/                  类型安全查询对象
    ├── rep/                        ══ 业务Repository层 ══
    │   ├── currency.go / balance.go / deposit.go
    │   ├── withdraw.go / exchange.go
    │   └── balance_op.go           余额操作统一抽象（核心）
    ├── ser/                        ══ Service业务逻辑层 ══
    │   ├── currency_service.go     B端币种配置
    │   ├── wallet_service.go       C端首页+余额+记录
    │   ├── deposit_service.go      C端充值
    │   ├── withdraw_service.go     C端提现
    │   ├── exchange_service.go     C端兑换
    │   └── rpc_service.go          财务回调RPC
    ├── cache/                      币种+汇率缓存管理
    └── utils/                      金额精度换算等工具
```

## 2.6 缓存策略

```
币种配置 → Redis Cache-Aside
  ├─ 币种列表 TTL 5 分钟
  ├─ 汇率 TTL 1-3 分钟
  ├─ B 端修改后主动删缓存，下次查询自动重建
  └─ 走 common/pkg/rds/ 封装

余额 → 不缓存
  └─ 金融数据必须实时准确，直接查 DB
     余额表走唯一键索引，TiDB查询 < 1ms
```

---

---

# 第三章：数据库表结构设计

## 3.1 11张表全景

```
币种配置模块（4张，我负责）          钱包模块（7张，我负责）
───────────────────────         ───────────────────────
[必须] currency_config 币种配置   [必须] user_balance    余额表（心脏）
[必须] exchange_rate   汇率配置   [必须] balance_flow    流水表（账本）
[应该] currency_log    配置日志   [必须] deposit_order   充值订单
[应该] rate_log        汇率日志   [必须] withdraw_order  提现订单
                                 [必须] exchange_order  兑换订单
                                 [应该] withdraw_account 提现账户
                                 [可选] reward_flow_rule 奖励规则
```

## 3.2 核心表一：user_balance（系统心脏）

```
user_balance — 所有业务最终都操作这张表
──────────────────────────────
user_id     BIGINT NOT NULL          ┐
currency    VARCHAR(10) NOT NULL     ├─ 三元组唯一键
wallet_type TINYINT NOT NULL         ┘
available   BIGINT NOT NULL DEFAULT 0    可用余额（最小单位整数）
frozen      BIGINT NOT NULL DEFAULT 0    冻结余额

UNIQUE KEY (user_id, currency, wallet_type)
```

**三个关键设计决策**：

**金额用 BIGINT 不用 DECIMAL**：
- "10.50 BSB"（precision=2）→ 存 1050；"100000 VND"（precision=0）→ 存 100000
- 整数加减无精度丢失（0.1+0.2≠0.3 的问题不存在）
- 展示层做换算：available ÷ 10^precision = 展示金额

**三元组唯一键设计**：user_id + currency + wallet_type，每个子钱包独立一行。币种可动态新增不影响表结构。

**available 和 frozen 同行**：提现冻结需原子地"available减+frozen加"，同一行 UPDATE 最简单安全。

**并发安全 SQL 模式**：

```sql
-- 扣减余额（乐观锁）：一条SQL完成"检查+扣减"
UPDATE user_balance SET available = available - ?
WHERE user_id = ? AND currency = ? AND wallet_type = ? AND available >= ?
-- affected_rows=0 → 余额不足；=1 → 成功

-- 提现冻结（乐观锁）
UPDATE user_balance SET available = available - ?, frozen = frozen + ?
WHERE user_id = ? AND currency = ? AND wallet_type = ? AND available >= ?

-- 提现确认扣款
UPDATE user_balance SET frozen = frozen - ?
WHERE user_id = ? AND currency = ? AND wallet_type = ? AND frozen >= ?

-- 提现解冻退回
UPDATE user_balance SET available = available + ?, frozen = frozen - ?
WHERE user_id = ? AND currency = ? AND wallet_type = ? AND frozen >= ?
```

## 3.3 核心表二：balance_flow（系统账本）

```
balance_flow — 不可变追加记录，审计底线
──────────────────────────────
user_id        BIGINT NOT NULL
currency       VARCHAR(10)
wallet_type    TINYINT
change_type    TINYINT NOT NULL     变动类型枚举（11种）
amount         BIGINT NOT NULL      变动金额（正=增 负=减）
before_balance BIGINT NOT NULL      变动前余额
after_balance  BIGINT NOT NULL      变动后余额
ref_order_no   VARCHAR(32)          关联订单号

UNIQUE KEY (ref_order_no, change_type)    ← 幂等兜底
INDEX (user_id, currency)                 ← 记录查询
```

**change_type 枚举**：1=充值入账 2=充值奖金 3=提现冻结 4=提现确认 5=提现解冻 6=兑换扣减 7=兑换增加 8=兑换赠送 9=人工加款 10=人工减款 11=人工补单

**核心价值**：①审计追溯 ②对账校验（flow_N.after = flow_N+1.before，不等有异常）③幂等兜底（唯一键阻止重复写入）

## 3.4 币种配置表

```
currency_config — 系统配置地基
  currency_code   VARCHAR(10) UNIQUE    BSB/VND/IDR/THB/USDT
  currency_type   TINYINT               1=法币 2=数字货币 3=平台币
  precision       TINYINT               小数精度（上线后不可改）
  deposit_min/max BIGINT                充值限额
  withdraw_min/max/daily BIGINT         提现限额
  fee_type/fee_value                    手续费（固定/百分比）
  arrive_days     TINYINT               T+N 到账天数

exchange_rate — 汇率配置
  currency_code   VARCHAR(10) UNIQUE    每个币种一条汇率
  rate            BIGINT                汇率值（BIGINT扩大后存储）
  rate_precision  TINYINT               汇率精度
  gift_ratio      INT                   兑换赠送比例（万分比）
```

## 3.5 订单表

```
deposit_order   充值订单 — 状态机：1待支付 → 2成功/3失败/4超时/5补单成功
withdraw_order  提现订单 — 状态机：1审核中 → 2成功/3失败
exchange_order  兑换订单 — 状态机：1成功/2失败（即时完成）
```

三张订单表结构类似：平台订单号（雪花ID）+ 用户 + 币种 + 金额 + 状态 + 方式 + 汇率快照 + 关联信息。充值订单额外有 bonus_activity_id/bonus_amount；兑换订单额外有 from/to_currency、gift_ratio/gift_amount。

## 3.6 辅助表

```
currency_log     — 币种配置操作日志（Append-Only，审计合规）
rate_log         — 汇率变更日志（旧值/新值/操作人/时间）
withdraw_account — 提现账户（银行卡号 AES 加密存储）
                   UNIQUE (user_id, currency, method)
```

## 3.7 设计铁律

```
1. 金额一律 BIGINT（int64），存最小单位整数
2. 时间一律 BIGINT（int64），毫秒级时间戳（UserPlugin自动填充）
3. 状态/枚举用 INT（int32），预留码段
4. 订单号 VARCHAR(32)，雪花ID填充
5. 敏感信息（银行卡号等）AES加密存储
6. 配置表加软删除（delete_at），订单/流水/余额表不加
7. 不建外键，用应用层保证引用完整性
8. 索引按查询驱动，不先建再说
```

---

---

# 第四章：接口完整清单 — 46个方法

## 4.1 速查数字

```
接口总数     46 个（1 个 WalletService）
  C端接口    19 个（首页2 + 充值5 + 兑换2 + 提现4 + 记录2 + 补充4）
  B端接口     6 个（币种配置全部）
  RPC暴露    12 个（余额操作6写 + 查询2读 + 币种3读 + 流水1读）
  定时任务     3 个（汇率更新 + 订单超时 + 对账校验）
  RPC调出     6 个（确定依赖）+ 5 个（工程规范/待确认）

优先级分布：Must 27 + Should 14 + Could 5
```

## 4.2 C 端接口（19个，gate-font）

### 钱包首页（2个）

```
C-1  POST /api/wallet/overview              GetWalletOverview        Must
     → 返回当前币种下各子钱包余额，纯读操作
C-13 POST /api/wallet/rate/current          GetCurrentRate           Must
     → 返回当前币种入款/出款/BSB换算汇率
```

### 充值（5个）

```
C-2  POST /api/wallet/deposit/methods       GetDepositMethods        Must
     → 调财务RPC获取可用充值方式，直接透传
C-3  POST /api/wallet/deposit/create        CreateDepositOrder       Must
     → 校验→重复转账拦截→生成订单→调财务创建渠道订单→返回支付信息
C-4  POST /open/wallet/deposit/usdt         GetUSDTDepositInfo       Must
     → USDT链上充值信息：网络/地址/QR码
C-14 POST /api/wallet/deposit/dup-check     CheckDuplicateDeposit    Should
     → 同金额同币种处理中订单数：1笔警告/2笔警告/≥3笔拦截
C-16 POST /api/wallet/deposit/status        GetDepositOrderStatus    Should
     → 前端从支付页返回后轮询订单状态
```

### 兑换（2个）

```
C-5  POST /api/wallet/exchange/preview      GetExchangePreview       Must
     → 输入法币金额→返回BSB到账数+赠送+流水要求
C-6  POST /api/wallet/exchange/create       CreateExchange           Must
     → 完全闭环，无外部依赖。扣法币+入BSB+写流水+写订单
```

### 提现（4个）

```
C-7  POST /api/wallet/withdraw/methods      GetWithdrawMethods       Must
     → 调财务RPC+KYC过滤+充值来源过滤（钱从哪进从哪出）
C-8  POST /api/wallet/withdraw/create       CreateWithdrawOrder      Must
     → 校验KYC+限额+户名→冻结余额→生成订单→调财务提交审核
C-9  POST /api/wallet/withdraw/account      GetWithdrawAccount       Must
     → 已绑定提现账户信息回显
C-10 POST /api/wallet/withdraw/account/save SaveWithdrawAccount      Must
     → 首次保存提现账户，户名须与KYC姓名一致
```

### 记录+补充（6个）

```
C-11 POST /api/wallet/transaction/list      GetTransactionList       Must
C-12 POST /api/wallet/transaction/detail    GetTransactionDetail     Must
C-15 POST /api/wallet/deposit/bonus         GetBonusActivities       Should
C-17 POST /api/wallet/withdraw/limits       GetWithdrawLimits        Should
C-18 POST /api/wallet/deposit/cancel        CancelDepositOrder       Could
C-19 POST /api/wallet/currencies            GetAvailableCurrencies   Could
```

## 4.3 B 端接口（6个，gate-back）

```
B-1  POST /api/currency/list                GetCurrencyList          Must
B-2  POST /api/currency/edit                EditCurrency             Must
B-3  POST /api/currency/toggle-status       ToggleCurrencyStatus     Must
B-4  POST /api/currency/set-base            SetBaseCurrency          Must
B-5  POST /api/currency/rate-log/list       GetExchangeRateLogs      Must
B-6  POST /api/currency/detail              GetCurrencyDetail        Should
```

所有B端写操作调 ser-blog.AddActLog 记审计日志。精度字段一旦设定**不可修改**。基准币种设定后不可变更。

## 4.4 RPC 对外暴露（12个）

### 写操作（6个）

```
R-1  CreditWallet         入账/加款（充值/补单/加款/返奖/活动/主播收入）  Must
R-2  DebitWallet          扣款/减款（人工减款/指定钱包扣款）              Must
R-3  FreezeBalance        冻结余额（提现发起时 available→frozen）        Must
R-4  UnfreezeBalance      解冻余额（提现驳回时 frozen→available）        Must
R-5  DeductFrozenAmount   扣除冻结（出款成功时 frozen减少）              Must
R-7  DebitByPriority      按优先级扣款（先中心→再奖励，返deductDetails） Should
```

### 读操作（6个）

```
R-6  QueryWalletBalance   查询余额（含可用+冻结，四子钱包明细）           Must
R-8  GetCurrencyConfig    获取币种配置（列表+详情）                      Must
R-9  GetExchangeRate      获取汇率（实时/平台/入款/出款四种）             Must
R-10 ConvertAmount        金额换算（跨币种转换+精度截断）                 Should
R-11 BatchQueryBalance    批量查询余额（限100个userId）                  Could
R-12 GetWalletFlowList    查询账变流水                                   Could
```

所有写方法必须带 refOrderNo（幂等键），userId 显式传参。统一处理模式：幂等校验→参数校验→事务操作→返回。

## 4.5 定时任务（3个）

```
T-1  ExchangeRateUpdate   Must    3个三方API取均值→偏差检测→触发汇率更新
T-2  DepositOrderTimeout  Should  超时订单自动失效
T-3  BalanceReconciliation Should 流水与余额一致性定时对账
```

## 4.6 RPC 主动调用（确定6个 + 工程规范3个 + 待确认2个）

```
O-1  ser-finance  GetPaymentMethods         Should  ❌需mock
O-2  ser-finance  MatchAndCreateChannelOrder Should  ❌需mock
O-3  ser-finance  InitiateChannelPayout      Should  ❌需mock+归属待确认
O-4  ser-finance  GetChannelOrderStatus      Could   ❌需mock
O-5  ser-kyc      KycQuery                   Must    ✅可直接调
O-6  ser-user     GetUserInfoExpend          Must    ✅可直接调
O-7  ser-blog     AddActLog (oneway)         Must*   ✅可直接调
O-8  ser-buser    QueryUserByIds             Should* ✅可直接调
O-9  ser-socket   WebSocket推送              Should* ⚠️待接口完善
```

---

---

# 第五章：核心业务流程

## 5.1 流程一：币种配置管理（B端，最基础，最先做）

**Happy Path**：B端运营 CRUD 币种配置 → 写DB → 删Redis缓存 → 写审计日志 → RPC暴露供他方查询。

**Error Path**：
- 币种代码重复 → 唯一键冲突 → 返回"币种已存在"
- 修改精度 → 上线后应禁止修改（影响所有历史金额换算）
- 禁用币种 → 已有余额用户不受影响，只是不能新操作
- 审计日志写入失败 → 不应阻塞主流程（降级容错）

**没有难点，就是标准的 B 端管理模块**。参照 ser-buser 的 RBAC 管理风格。

## 5.2 流程二：余额查询（C端，纯读取）

**Happy Path**：userId 从 JWT→metainfo→tracer.GetUserId(ctx) 获取 → 查 user_balance → 返回4子钱包余额。

**Error Path**：
- 新用户无余额记录 → 返回全部为0（不预创建余额行，首次操作时自动创建）
- 查询超时 → 余额直接查DB，不走缓存，TiDB唯一键索引 <1ms

## 5.3 流程三：充值（C端，跨模块，五阶段）

我负责"头和尾"（发起 + 入账），中间渠道对接交给财务。

```
[阶段一] 进入充值页
  → 调财务RPC获取可用充值方式 → 透传C端
  ⚠ 财务RPC失败 → 返回"充值暂不可用"（降级处理）

[阶段二] 用户选方式、输金额、确认
  → 参数校验（币种/金额/限额）
  → 重复转账三级拦截（0笔通过/1笔警告/2笔警告/≥3笔拦截）
  → 生成本地充值订单（状态=待支付，此时余额不变）
  → 调财务RPC创建渠道支付订单 → 拿到支付信息返回C端
  ⚠ 创建渠道订单RPC失败 → 本地订单标记失败，返回C端错误
     （无余额影响，充值是先支付后入账）

[阶段三] 等待支付 → 渠道回调 → 财务确认（我不参与）

[阶段四] 充值入账（财务调我的RPC接口）
  → 幂等校验（订单号查状态，已入账直接返回成功）
  → 事务内：更新订单状态=成功 + 中心钱包余额增加 + 写流水
  → 如有奖金活动：奖励钱包余额增加 + 写奖金流水
  ⚠ 入账事务失败 → 返回失败，财务侧记录异常，等待人工补单

[阶段五] C端轮询/推送确认到账
```

**关键难点**：阶段四入账操作必须幂等（财务可能超时重试），事务内保证余额+订单+流水三者一致。

## 5.4 流程四：兑换（C端，完全闭环 — 烟囱测试）

只依赖汇率，不跨模块，全在本服务内完成。**这是最先验证的交易链路。**

```
[步骤一] 获取汇率预览（RPC/内部）→ "1 BSB = 23000 VND" + 赠送5%

[步骤二] 用户输金额确认
  → 重新获取最新汇率（不用页面旧汇率）
  → 计算：230000 VND ÷ 23000 = 10 BSB，赠送5% = 0.5 BSB
  → 余额预检：VND可用 ≥ 230000？

[步骤三] 事务操作（核心）
  ┌─ 事务 ──────────────────────────────┐
  │ ① VND中心钱包 available - 230000（乐观锁） │
  │ ② BSB中心钱包 available + 1000 (10 BSB)   │
  │ ③ BSB奖励钱包 available + 50 (0.5 BSB)    │
  │ ④ 写兑换订单（锁定汇率快照+赠送比例快照）    │
  │ ⑤ 写余额变动流水（3条）                      │
  └──────────────────────────────────────┘
```

**Error Path**：
- 余额不足 → 乐观锁 affected_rows=0 → 返回"余额不足"
- 汇率获取失败 → 返回"汇率暂时不可用"
- 事务失败 → 全部回滚，用户余额不变

**为什么兑换最先做**：它涉及两个币种余额变动、事务操作、流水记录、乐观锁 — 覆盖余额操作的所有核心逻辑，但不依赖任何外部模块。兑换跑通 = 充提逻辑可复用。

## 5.5 流程五：提现（C端，跨模块最复杂）

跨度最长、模块最多、状态流转最复杂。

```
[阶段一] 进入提现页（并行4个请求）
  → KYC状态查询（调ser-kyc，非USDT提现必须KYC）
  → 可用提现方式（调财务RPC + 按充值来源过滤"钱从哪进从哪出"）
  → 提现限额查询（读币种配置）
  → 提现账户信息（读本地表，首次填写后回显）
  ⚠ KYC未认证 → C端引导认证，阻断提现

[阶段二] 确认提现（最复杂的订单创建）
  → 并行校验：限额 + KYC户名比对 + 余额
  → 冻结余额：available -= (amount+fee), frozen += (amount+fee)（乐观锁）
  → 生成提现订单（状态=审核中）
  → 调财务RPC提交审核
  ⚠ 提交审核RPC失败 → 补偿：解冻余额 + 订单标记失败 + 写回退流水
  ⚠ 解冻也失败（极端） → 记日志告警，人工介入

[阶段三] 财务三级审核+渠道出款（我不参与）

[阶段四] 财务回调我（两种结果）
  审核通过+出款成功 → 调"提现确认" → frozen -= (amount+fee) → 钱正式离开
  审核拒绝/出款失败 → 调"提现解冻" → frozen → available（钱回到用户手里）
```

**关键难点一：冻结→提交审核的补偿**。冻结成功后如果调财务RPC失败，用户钱"卡住了"。方案：立即解冻 + 订单标记失败 + 写回退流水。

**关键难点二：提现方式过滤**。合规要求"钱从哪进从哪出"：只充过法币→法币提现；只充过USDT→USDT提现；混合→法币优先。

## 5.6 流程六：RPC 被调用（统一处理模式）

财务调我的 6 个写接口，全部遵循统一模式：

```
① 幂等校验（订单号/修正单号查状态，已完成→直接返回成功）
② 参数校验（用户/币种/金额/子钱包类型）
③ 事务操作（余额变动 + 订单状态更新 + 写流水）
④ 返回成功/失败

不同接口只是参数不同：
  充值入账    → available + amount
  提现确认    → frozen - amount
  提现解冻    → available + amount, frozen - amount
  人工加款    → available + amount（指定子钱包）
  人工减款    → available - amount（指定子钱包，带乐观锁）
  人工补单    → available + amount（关联原充值订单，更新原订单状态）
```

---

---

# 第六章：技术挑战与解决方案

## 6.1 挑战总览

```
致命级（资损）：
  1. 并发余额操作安全      100%会遇到
  2. 重复入账/扣款（幂等）  100%会遇到
  3. 金额精度与溢出         100%会遇到

严重级（资金卡死/系统不可用）：
  4. 跨模块分布式一致性     100%会遇到
  5. 事务边界与连接池耗尽   高概率
  6. 账务一致性与对账       100%需要

重要级（性能/体验/安全）：
  7. 热点账户（高并发单用户） 中概率
  8. 缓存与DB数据一致性     高概率
  9. 敏感数据安全           100%需要
  10. 防刷与风控            高概率

关注级（后期影响）：
  11. 高可用与降级策略      中概率
  12. 数据增长与存储治理    后期
```

## 6.2 致命级挑战及方案

### 挑战一：并发余额操作安全

**问题**：两个请求并发操作同一余额行 → 超扣或更新丢失。

**方案对比**：

| 方案 | 优势 | 劣势 |
|------|------|------|
| **CAS条件更新（选择）** | 无显式锁、无死锁、单SQL原子完成 | 竞争激烈时失败率高 |
| 悲观锁 SELECT FOR UPDATE | 支持复杂计算 | 锁持有时间长、死锁风险 |
| 分布式锁（Redis/ETCD） | 跨服务安全 | 复杂、性能差 |

**最终方案**：CAS条件更新，不重试。`UPDATE SET available=available-? WHERE available>=?`。InnoDB行级排他锁在UPDATE瞬间持有，微秒级释放。available >= amount 本身就是版本条件，比额外version字段更有业务语义。

### 挑战二：重复入账/扣款（幂等）

**问题**：RPC超时重试 → 同一笔充值入账两次 = 直接资损。

**方案对比**：

| 方案 | 优势 | 劣势 |
|------|------|------|
| **应用层+DB唯一键（选择）** | 双层防护，99.9%+0.1% | 需处理唯一键冲突异常 |
| Redis SETNX | 快速 | Redis宕机或过期后失效 |
| 幂等表 | 通用 | 额外一张表 |

**最终方案**：两层幂等。第一层（应用层）：查订单状态，已完成直接返回。第二层（DB兜底）：balance_flow 的 `(ref_order_no, change_type)` 唯一键，极端并发穿透时唯一键冲突 → 捕获异常 → 返回成功。

### 挑战三：金额精度与溢出

**问题**：浮点数 0.1+0.2=0.30000000000000004 → 积累成资损。

**最终方案**：BIGINT最小单位整数，全链路无浮点。Stripe/支付宝同款做法。

```
存储：int64（BIGINT）最小单位
计算：Go int64 整数运算
传输：Thrift i64
展示：amount ÷ 10^precision = 用户看到的金额

精度换算集中管理：
  ToMinUnit(displayAmount, precision) → int64
  ToDisplayAmount(minUnit, precision) → string
```

溢出风险评估：int64 最大 9.2×10^18，即使 VND 10亿=10^9，距上限还有 10^9 倍空间 → 不会溢出。

## 6.3 严重级挑战及方案

### 挑战四：跨模块分布式一致性

**典型场景**：提现冻结成功 → 调财务提交审核RPC失败 → 用户钱"卡住"了。

**最终方案**：本地补偿 + 人工补单兜底。不引入 Seata/DTM（两服务三步操作，框架是杀鸡用牛刀）。

```
场景1: 充值创建→调财务失败 → 本地订单标记失败（无余额影响）
场景2: 提现冻结→调财务失败 → 立即解冻+订单失败+写回退流水
场景3: 提现冻结→调财务超时 → 视为失败解冻（简单优先）
场景4: 财务调入账→钱包DB异常 → 人工补单（需求文档明确设计的功能）
```

### 挑战五：事务边界与连接池耗尽

**铁律**：RPC在事务外，DB操作在事务内。事务内只做纯本地DB操作。

```
正确：获取汇率(RPC) → 计算 → [开事务] → 操作余额 → 写流水 → [提交事务]
错误：[开事务] → 获取汇率(RPC) → 操作余额 → [提交事务]
```

以兑换为例：事务外获取汇率+计算 → 事务内5条SQL（2扣减+2增加+1订单+3流水） → 纯DB操作预估1-5ms → 连接池利用率极高。

### 挑战六：账务一致性与对账

**最终方案**：事务内同步写流水 + before/after连续性校验。

每一次余额变动，余额UPDATE + 流水INSERT + 订单UPDATE 三者在同一事务中，全部成功或全部回滚。不选异步写流水 — 消息队列存在丢失可能，资金流水绝对不能丢。

## 6.4 重要级挑战简要方案

| 挑战 | 方案 |
|------|------|
| 热点账户 | 当前不特殊处理。万级DAU单用户并发概率<1%，InnoDB行锁串行够用 |
| 缓存一致性 | Cache-Aside+短TTL+主动删缓存，余额绝不缓存 |
| 敏感数据安全 | 应用层AES加密（复用现有crypto_util）+ 脱敏展示 |
| 防刷风控 | 重复转账三级拦截+限额校验+KYC强制校验+户名比对+充值来源过滤 |

## 6.5 关注级挑战简要方案

| 挑战 | 方案 |
|------|------|
| 高可用降级 | 功能独立降级（财务不可用→只影响充提，不影响兑换/查询），RPC超时3s |
| 数据增长 | TiDB单表千万级无需分表，按当前增速3年不需要考虑。后续可按月分流水表 |

## 6.6 技术保障五道防线

```
防线一：乐观锁 → 不超扣
防线二：双层幂等 → 不重复
防线三：事务铁律 → 不丢失
防线四：BIGINT → 不丢精度
防线五：统一余额变动方法 → 不写错
```

---

---

# 第七章：跨模块依赖与集成策略

## 7.1 依赖全景

```
ser-finance（新模块，尚不存在）— 4 个调用，全部 Should，第四期
  ├── GetPaymentMethods          充值/提现方式查询
  ├── MatchAndCreateChannelOrder 创建渠道支付订单
  ├── InitiateChannelPayout      发起通道代付（归属待确认）
  └── GetChannelOrderStatus      查询通道订单状态

ser-kyc（已存在）— 1 个调用，Must，第四期
  └── KycQuery                   KYC状态+姓名查询

ser-user（已存在）— 1 个调用，Must，第四期
  └── GetUserInfoExpend          用户信息查询

ser-blog（已存在）— 1 个调用，Must*，第二期起
  └── AddActLog (oneway)         B端审计日志
```

## 7.2 双向依赖对照（ser-wallet ↔ ser-finance）

```
我暴露给财务（6个RPC）                  我调用财务（4个RPC）
──────────────────────                 ──────────────────────
R-1  CreditWallet（入账）     ←──┐  ┌──→ O-1  GetPaymentMethods
R-2  DebitWallet（扣款）      ←──│  │──→ O-2  MatchAndCreateChannelOrder
R-3  FreezeBalance（冻结）    ←──│  │──→ O-3  InitiateChannelPayout
R-4  UnfreezeBalance（解冻）  ←──│  │──→ O-4  GetChannelOrderStatus
R-5  DeductFrozenAmount（扣冻结）←─┘  │
R-6  QueryWalletBalance（查余额）     │

    合计 10 个 RPC 横跨两模块
```

## 7.3 依赖分期分布

```
第一期（骨架）：0 个外部 RPC — 纯框架搭建
第二期（币种）：1 个 — ser-blog.AddActLog
第三期（核心）：0 个 — 兑换完全闭环（故意设计）
第四期（充提）：7-9 个 — 集中爆发（ser-finance + ser-kyc + ser-user）
```

**核心思路**：外部依赖集中在最后一期，前三期完全自主可控。

## 7.4 集成策略

**IDL先行**：评审通过后立即提交全部46方法的IDL定义，gen_kitex.sh生成桩代码。财务pull后有 `rpc.WalletClient()` 类型定义，可先写调用代码。同理我们基于 `rpc.FinanceClient()` mock先行。

**Mock策略**：对 ser-finance 做接口抽象层：

```
internal/external/finance.go       → FinanceService interface
internal/external/finance_rpc.go   → 真实RPC实现
internal/external/finance_mock.go  → Mock实现
main.go 通过ETCD配置切换
```

可直接调用（不需mock）：ser-kyc/ser-user/ser-blog/ser-buser。

---

---

# 第八章：开发分期与构建顺序

## 8.1 四期计划

```
┌────────────────────────────────────────────────────────┐
│ 第一期：骨架搭建                                          │
│ ────────────────                                        │
│  ① 编写全部 IDL（重点：RPC暴露接口先给财务）                │
│  ② gen_kitex.sh 生成桩代码                                │
│  ③ 注册到 common（namesp + rpc_client + go.work）         │
│  ④ 建表建库配 ETCD                                        │
│  ⑤ GORM Gen 生成 model/query                              │
│  ⑥ 搭建骨架（main.go / handler.go / cfg / errs）          │
│  ⑦ 接入 gateway 路由                                      │
│                                                           │
│  验收：服务能启动、注册ETCD、网关能路由、空接口能通           │
│  依赖：不依赖任何人                                         │
│  产出：财务可 pull 代码用 rpc.WalletClient() 编译通过        │
├────────────────────────────────────────────────────────┤
│ 第二期：币种配置（~13个实现）                                │
│ ────────────────────                                    │
│  ① 币种 CRUD（B端全部6个接口）                              │
│  ② 汇率设置/查询 + 变更日志                                 │
│  ③ RPC暴露（R-8 币种配置 + R-9 汇率 + R-10 换算）           │
│  ④ Redis 缓存 + T-1 汇率定时更新                            │
│                                                           │
│  验收：B端能配置币种和汇率，RPC接口返回正确数据                │
│  依赖：不依赖任何人                                         │
│  外部RPC：1个（ser-blog.AddActLog）                        │
├────────────────────────────────────────────────────────┤
│ 第三期：钱包核心能力（~22个实现）                             │
│ ────────────────────────                                │
│  ① 余额查询（C端 C-1, C-13）                               │
│  ② 兑换完整实现（C-5, C-6）← 最早端到端验证（烟囱测试）       │
│  ③ 记录查询（C端 C-11, C-12）                               │
│  ④ 提现账户管理（C-9, C-10）                                │
│  ⑤ RPC暴露全部写接口（R-1~R-7）+ 读接口（R-6, R-11, R-12）  │
│  ⑥ 通用枚举 + 补充接口                                      │
│                                                           │
│  验收：兑换端到端跑通，RPC接口全部就绪→通知财务联调          │
│  依赖：只依赖自己的币种配置                                  │
│  外部RPC：0个（故意设计）                                   │
├────────────────────────────────────────────────────────┤
│ 第四期：充值+提现（~12个实现）                               │
│ ────────────────────────                                │
│  ① 充值功能（C端 C-2~C-4, C-14~C-16, C-18）                │
│  ② 提现功能（C端 C-7, C-8, C-17）                           │
│  ③ 联调对接 + T-2, T-3 定时任务                              │
│                                                           │
│  策略：先写好钱包侧逻辑，mock财务返回→财务就绪后替换联调    │
│  依赖：强依赖 ser-finance + ser-kyc + ser-user               │
│  外部RPC：7-9个（集中爆发）                                  │
└────────────────────────────────────────────────────────┘
```

## 8.2 为什么这个顺序

```
第一期（骨架）→ 不依赖任何人，产出 IDL 给财务
第二期（币种）→ 不依赖任何人，产出被其他模块依赖的基础数据
第三期（核心）→ 低依赖，兑换验证核心逻辑，RPC接口给财务联调
第四期（充提）→ 强依赖放最后，可用 mock 先行，财务就绪后联调
```

**核心思路：先把不依赖别人的做完，再把别人依赖我的做好，最后做我依赖别人的。**

## 8.3 协同时间线

```
[评审通过后立即]
  我：提交 IDL + 桩代码 → 财务 pull 后可编译 rpc.WalletClient()
  财务：提供 IDL → 我 pull 后可编译 rpc.FinanceClient()

[第一二期] 各自独立开发，不互相阻塞

[第三期]
  我：完成全部RPC接口实现 → 通知财务可以调了

[第四期]
  联调：充值全链路（用户支付→渠道回调→财务确认→我入账）
  联调：提现全链路（我冻结→提交审核→三级审核→我确认/解冻）
```

---

---

# 第九章：待确认问题与评审要点

## 9.1 致命级（影响数据模型，确认错了要推翻重来）

**问题1：五个币种各自的精度定义**

| 币种 | 小数精度 | 1的最小单位存储值 | 示例 |
|------|---------|-----------------|------|
| BSB | 2位？0位？ | 100？1？ | "10.50 BSB"存1050还是"10 BSB"存10？ |
| VND | 0位 | 1 | "100000 VND"存100000 |
| IDR | 0位 | 1 | "50000 IDR"存50000 |
| THB | 2位 | 100 | "100.50 THB"存10050 |
| USDT | 2位？6位？ | ？ | 链上最多6位小数，我们保留几位？ |

**为什么致命**：精度决定所有金额字段的换算系数。上线后几乎不可修改 — 改精度=迁移全部历史金额数据。

**问题2：四个子钱包的资金流转规则**

- 充值入哪个钱包？（我理解：统一入中心钱包）
- 兑换从哪个扣、入哪个？（我理解：中心扣法币，BSB入中心，赠送入奖励）
- 提现只能从中心钱包提吗？奖励/代理/主播能否直接提现？
- 四个钱包之间能互转吗？代理佣金怎么提现？
- 奖励钱包X倍流水规则：X是多少？全局统一还是每活动不同？谁校验？

**为什么致命**：子钱包规则是余额表设计基础。规则不清会出现"钱进去出不来"。

**问题3：提现手续费方向**

- 方案A（从余额扣）：提现100实际到手100，另外从余额扣5手续费
- 方案B（从提现金额扣）：提现100实际到手95，手续费5

影响全部提现逻辑：冻结金额计算、订单金额含义、流水记录、限额校验。

## 9.2 严重级（影响接口逻辑）

| # | 问题 | 影响 |
|---|------|------|
| 4 | BSB定位：结算币 vs 展示币？ | 兑换是"可选"还是"必经之路" |
| 5 | 充值奖金活动由谁管理？ | C-15数据源 |
| 6 | 混合充值用户的提现方式过滤规则？ | 法币+USDT都充过的用户提现时展示什么 |
| 7 | 汇率是否区分买入/卖出方向？ | 兑换汇率和提现展示汇率是否不同 |
| 8 | USDT提现是否需要KYC？ | 提现前置校验逻辑 |
| 9 | 汇率修改后进行中的操作用哪个？ | 需要汇率锁定机制？ |

## 9.3 重要级（影响开发节奏）

| # | 问题 | 影响 |
|---|------|------|
| 10 | ser-finance IDL何时提交？ | 直接阻塞第四期 |
| 11 | 提现订单超时和过期机制？ | 长期"审核中"的订单怎么处理 |
| 12 | 每日限额的时区基准？ | UTC/用户时区/服务器时区 |
| 13 | ETCD命名空间/数据库名/端口号？ | 第一期前置条件 |
| 14 | 错误码服务编号（拟用022）？ | 全部错误码定义 |

## 9.4 按对象分类速查

```
产品侧确认：#1精度 #2子钱包规则 #3手续费方向 #4BSB定位 #5奖金归属
            #6混合提现过滤 #7汇率方向 #8USDT-KYC #9汇率锁定 #11超时
            #12时区
后端侧协同：#10财务IDL #13ETCD/DB配置 #14错误码
前端侧对齐：amount字段类型(string vs i64) walletType枚举值 opType枚举值
```

---

---

# 第十章：风险评估与缓解措施

## 10.1 技术风险

| 风险 | 概率 | 影响 | 缓解方案 | 应急预案 |
|------|------|------|---------|---------|
| 并发余额超扣 | 高 | 资损 | CAS条件更新 | 100%防超扣 |
| 重复入账/扣款 | 高 | 资损 | 双层幂等 | 100%防重复 |
| 提现冻结后审核RPC失败 | 中 | 资金卡死 | 即时补偿解冻 | 解冻失败→日志告警→人工 |
| 事务内含RPC致连接池耗尽 | 低 | 系统不可用 | 铁律：RPC外DB内 | 连接池监控告警 |
| 金额精度丢失 | 低 | 对账差异 | BIGINT全链路无浮点 | 精度换算集中管理 |

## 10.2 协调风险

| 风险 | 概率 | 影响 | 缓解方案 | 应急预案 |
|------|------|------|---------|---------|
| 财务接口迟迟不出 | 中 | 充提无法联调 | 前三期不依赖+mock先行 | 最后mock→真实只需替换 |
| 需求评审后细节变更 | 高 | 部分返工 | 骨架不变+字段级弹性 | 表结构/IDL方法签名大概率不变 |
| 双方接口理解不一致 | 中 | 联调对不上 | IDL先行+评审后对齐协议 | 提前定义枚举值和字段格式 |
| IDL评审延迟 | 中 | 第一期阻塞 | 评审后立即提交IDL | 先按设计写，评审微调 |

## 10.3 业务风险

| 风险 | 概率 | 影响 | 缓解方案 |
|------|------|------|---------|
| 需求大改（增删模块） | 低 | 大范围返工 | 预评审确认骨架+弹性设计 |
| 合规要求变更 | 中 | KYC/提现规则调整 | 规则可配置（ETCD/DB） |
| 币种扩展 | 高 | 新增币种 | 零成本扩展（currency_config加一行即可） |

---

---

# 第十一章：关键指标与监控

## 11.1 业务指标

```
日交易量：充值笔数/金额、兑换笔数/金额、提现笔数/金额
成功率：充值成功率、提现成功率（审核通过+出款成功/总申请）
平均处理时间：充值从创建到入账、提现从申请到最终结果
异常率：入账失败率、补偿触发率、人工补单频率
```

## 11.2 技术指标

```
CAS重试/失败率：乐观锁竞争情况（超过5%需关注热点问题）
幂等命中率：重复请求被拦截的比例（正常应<1%，高了说明调用方有问题）
缓存命中率：币种/汇率缓存命中率（应>95%）
事务耗时：DB事务执行时间（应<10ms，超50ms需优化）
RPC调用耗时：外部RPC响应时间（超3s触发超时）
```

## 11.3 告警阈值建议

```
P0（立即处理）：余额为负数、流水不连续（after≠下一条before）
P1（30分钟内）：入账/扣款失败率>5%、DB连接池使用>80%
P2（当天处理）：缓存命中率<90%、CAS竞争率>5%
```

## 11.4 预判评审可能被问到的刁钻问题及应答

**Q：为什么不用 DECIMAL 存金额？**
A：BIGINT更快更安全。0.1+0.2≠0.3是浮点经典问题，整数运算从根源消除。Stripe/支付宝同款做法。

**Q：为什么不用悲观锁 SELECT FOR UPDATE？**
A：钱包余额操作都是简单加减，不需要"先读再算再写"。CAS一条SQL搞定，锁持有=单条UPDATE执行时间（微秒级）。

**Q：并发场景下怎么保证余额不出错？**
A：三重保障。①CAS乐观锁防超扣。②双层幂等防重复入账。③事务铁律保证余额+订单+流水原子性。

**Q：为什么不用分布式事务框架？**
A：两个服务、最多三步操作，Seata/DTM是为5+服务10+步编排设计的。本地补偿+人工补单完全够用。

**Q：余额为什么不走缓存？**
A：金融数据必须实时准确。缓存不一致=资损风险。读余额走唯一键索引<1ms，DB完全能扛。

**Q：提现冻结后审核RPC失败怎么办？**
A：三步补偿。①立即解冻。②订单标记失败。③写回退流水。极端情况记日志+告警人工介入。

**Q：为什么不拆成两个服务？**
A：①数据内聚（余额实时依赖币种参数）②事务需要（兑换跨两币种必须同事务）③工程一致（ser-buser 31方法同量级）。

**Q：同一用户多端同时操作怎么办？**
A：CAS天然支持。两请求同时扣余额，先到的成功，后到的available<amount失败返"余额不足"。

**Q：怎么扩展新币种？**
A：零成本。currency_config新增一行，user_balance的currency是VARCHAR，新币种余额行首次操作时自动创建。

**Q：怎么做对账？**
A：balance_flow天然支持。相邻流水after=下一条before，不等即异常。T-3定时任务扫描。

---

---

# 附录：速查数字表

| 维度 | 数据 |
|------|------|
| 我负责的接口数 | **46** 个（19C端 + 6B端 + 12RPC暴露 + 3定时 + 6RPC调出） |
| 我负责的表数 | **11** 张（4币种配置 + 7钱包） |
| IDL文件数 | **7** 个（1 service + 6 entity） |
| 代码文件总数 | **71** 个（40手写 + 31自动生成） |
| Service层文件数 | **8** 个 |
| 新技术引入 | **0** 个 |
| 开发分期 | **4** 期（骨架→币种→核心→充提） |
| 与财务RPC接口 | 我调他 **4** + 他调我 **6** = **10** 个 |
| 外部RPC总依赖 | **11** 个（finance 4 + kyc 1 + user 1 + blog 1 + buser 1 + socket 1 + 待确认 2） |
| 错误码段 | **6022**000 ~ 6022999 |
| 优先级分布 | Must **27** + Should **14** + Could **5** |
| 网关路由 | gate-back **6** + gate-font **19** + RPC only **12** |
| 技术挑战 | **12** 项（3致命 + 3严重 + 4重要 + 2关注） |
| 致命级待确认 | **3** 个（精度/子钱包规则/手续费方向） |
| 严重级待确认 | **6** 个 |
| 重要级待确认 | **5** 个 |
| 前置阻塞项 | 第一期4个 + 第二期3个 + 第三期3个 + 第四期5个 |

---

# 一句话总结

**ser-wallet 是一个 46 接口的单体微服务，四期开发从独立到依赖，核心是用乐观锁+双层幂等+事务铁律保证余额操作的绝对正确性，技术栈全部复用现有工程零引入，与财务模块通过 IDL 先行实现并行开发。**
