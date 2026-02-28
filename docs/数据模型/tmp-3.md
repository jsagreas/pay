# ser-wallet 数据模型设计（产品需求 → 数据模型映射）

> 设计时间：2026-02-26
> 设计依据：需求文档 + 产品原型 + 架构设计方案 + 现有工程 GORM Gen 模型规范
> 设计目的：将产品原型中的现实世界描述，映射为可直接落地的数据模型，作为"现实层面→代码层面"的桥梁
> 设计原则：
>   - 字段类型对齐现有工程规范（int64 时间戳、int32 状态/枚举、*string 可选字段）
>   - 金额字段统一 BIGINT 整数存储（最小货币单位）
>   - 汇率字段统一 BIGINT × 10^8 存储（保留8位精度）
>   - 软删除用 delete_at int64（0=未删除，毫秒时间戳=已删除）
>   - 主键用自增 int64，业务单号用 snowflake int64

---

## 一、现实世界 → 数据模型 映射总览

产品需求中有明确的现实世界概念，每个概念需要对应一个或多个数据模型。以下是完整映射：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  产品原型概念                    数据模型                   所属业务域        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  币种（BSB/VND/IDR/THB/USDT）  → currency_config           币种配置域       │
│  基准币种（USDT）              → currency_config.is_base    币种配置域       │
│  实时汇率/平台汇率/入出款汇率  → currency_config + 缓存     币种配置域       │
│  汇率变更日志                  → exchange_rate_log          币种配置域       │
│                                                                              │
│  用户钱包（中心/奖励/主播/代理）→ user_wallet               账户域          │
│  可用余额 / 冻结余额          → user_wallet 两个字段        账户域          │
│  资金变更记录（流水）          → wallet_flow                账户域          │
│  冻结记录                     → wallet_freeze_record        账户域          │
│                                                                              │
│  充值订单                     → deposit_order               交易域          │
│  提现订单                     → withdraw_order              交易域          │
│  提现账户（银行卡/电子钱包）   → withdraw_account           交易域          │
│  兑换记录（法币→BSB）          → exchange_record            交易域          │
│                                                                              │
│  充值方式 / 提现方式配置       → 财务模块管（非 ser-wallet） ——             │
│  通道配置 / 轮询规则          → 财务模块管（非 ser-wallet） ——             │
│  KYC 认证信息                 → ser-kyc 管（非 ser-wallet） ——             │
│  奖金活动                     → 活动模块管（非 ser-wallet） ——             │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**说明：** ser-wallet 只管「账」不管「通道」，因此充值方式配置、通道配置、KYC 数据不在 ser-wallet 建模。ser-wallet 通过 RPC 从对应模块获取这些数据。

---

## 二、枚举定义（所有模型共用）

在定义具体模型前，先明确所有枚举值。这些枚举对应代码中 `internal/enum/` 目录下的常量定义。

### 2.1 钱包类型 WalletType

```
产品原型概念 → 枚举值映射：

产品说法                  枚举名称              值     说明
──────                    ──────               ──      ──────
中心钱包                  WalletTypeCenter      1      主资金钱包，充值/投注/提现
奖励钱包                  WalletTypeReward      2      活动/任务奖金暂存，需完成流水后转入中心
主播钱包                  WalletTypeAnchor      3      直播/连麦/短视频收益
代理钱包（预留）           WalletTypeAgent       4      代理分佣收益（当前版本预留）
场馆钱包（预留）           WalletTypeVenue       5      igaming场馆临时钱包（当前版本预留）

产品原型依据：
  - 币种配置需求2.1资金钱包定义："中心钱包→奖励钱包→场馆钱包(预留)→代理钱包→主播钱包"
  - 钱包需求4.1："资金钱包余额 = 中心钱包 + 奖励钱包"
  - 消费时优先扣中心再扣奖励
```

### 2.2 币种类型 CurrencyType

```
产品原型概念 → 枚举值映射：

产品说法                  枚举名称                值     说明
──────                    ──────                  ──     ──────
法定货币                  CurrencyTypeFiat         1     VND/IDR/THB 等
加密货币                  CurrencyTypeCrypto       2     USDT/BTC/ETH 等
平台货币                  CurrencyTypePlatform     3     BSB

产品原型依据：
  - 币种配置需求3.1："法定货币/加密货币/平台货币 三类"
  - "所有类型的币种在配置层统一为'币种'，在规则层区分"
```

### 2.3 流水类型 FlowType

```
产品原型概念 → 枚举值映射：

产品说法                  枚举名称                  值     说明
──────                    ──────                    ──     ──────
充值到账                  FlowTypeDeposit            1     充值成功后上账
投注扣款                  FlowTypeBetDebit           2     投注扣减余额
投注返奖                  FlowTypeBetCredit          3     投注结算返奖
提现冻结                  FlowTypeWithdrawFreeze     4     提现申请冻结金额
提现确认                  FlowTypeWithdrawConfirm    5     出款成功确认扣除冻结
提现解冻                  FlowTypeWithdrawUnfreeze   6     提现驳回/失败退回冻结
兑换扣减                  FlowTypeExchangeDebit      7     兑换时法币扣减
兑换到账                  FlowTypeExchangeCredit     8     兑换时BSB到账
兑换赠送                  FlowTypeExchangeBonus      9     兑换赠送部分到奖励钱包
充值补单                  FlowTypeSupplement         10    财务人工补单
人工加款                  FlowTypeManualCredit       11    运营/财务人工加款
人工减款                  FlowTypeManualDebit        12    异常追回/风控处罚
活动奖励                  FlowTypeRewardCredit       13    活动/任务返奖
主播收益                  FlowTypeAnchorIncome       14    直播打赏收益
充值奖金                  FlowTypeDepositBonus       15    充值活动奖金到奖励钱包
钱包转入                  FlowTypeTransferIn         16    从主播/代理钱包转入中心
钱包转出                  FlowTypeTransferOut        17    从中心钱包转出

产品原型依据：
  - 钱包需求8.1记录分类："提现、充值、兑换、奖励 四Tab"
  - 币种配置需求2.1资金流向：充值→中心，返奖→中心，活动→奖励，打赏收益→主播
  - 财务需求2.5人工修正：补单(B+)、加款(A+)、减款(M+)
  - 钱包需求4.2余额规则："消费时优先扣中心再扣奖励" → 需区分每笔来源
```

### 2.4 充值订单状态 DepositOrderStatus

```
产品说法                  枚举名称                  值     说明
──────                    ──────                    ──     ──────
待支付                    DepositStatusPending       1     订单已创建，等待支付
进行中                    DepositStatusProcessing    2     已发起支付，等待回调
支付成功                  DepositStatusSuccess       3     支付成功，已上账
支付失败                  DepositStatusFailed        4     支付失败/超时
订单失效                  DepositStatusExpired       5     超过支付时限

产品原型依据：
  - 钱包需求5.2-5.4：通道回调 → 成功/失败
  - 钱包需求8.3充值字段："状态（颜色区分：已完成/审核中/失败）"
  - 产品原型7-2："充值进行中"弹窗
  - 产品原型1-三方充值页面：右侧"订单失效"状态
```

### 2.5 提现订单状态 WithdrawOrderStatus

```
产品说法                  枚举名称                    值     说明
──────                    ──────                      ──     ──────
待风控审核                WithdrawStatusRiskPending     1     提交后初始状态
风控驳回                  WithdrawStatusRiskRejected    2     风控审核不通过
待财务审核                WithdrawStatusFinPending      3     风控通过，等财务审核
财务驳回                  WithdrawStatusFinRejected     4     财务审核不通过
待出款                    WithdrawStatusPayPending      5     财务通过，等出款
出款成功                  WithdrawStatusPaySuccess      6     出款完成
出款失败                  WithdrawStatusPayFailed       7     出款执行失败

产品原型依据：
  - 财务需求2.4提现记录："订单状态：待风控审核/风控驳回/待财务审核/财务驳回/待出款/出款失败/出款成功"
  - 这7个状态与产品文档一一对应
```

### 2.6 冻结状态 FreezeStatus

```
产品说法                  枚举名称                  值     说明
──────                    ──────                    ──     ──────
冻结中                    FreezeStatusFrozen         1     余额已从可用转到冻结
已解冻                    FreezeStatusUnfrozen       2     驳回/失败，冻结退回可用
已确认扣除                FreezeStatusConfirmed      3     出款成功，冻结永久减少

产品原型依据：
  - 架构设计方案六-状态机：Frozen → Unfrozen / Confirmed 互斥
  - 提现驳回/失败 → UnfreezeBalance（退回）
  - 出款成功 → ConfirmDebit（永久扣除）
```

### 2.7 提现方式 WithdrawMethod

```
产品说法                  枚举名称                  值     说明
──────                    ──────                    ──     ──────
银行卡                    WithdrawMethodBank         1     各国银行卡转账
电子钱包                  WithdrawMethodEWallet      2     GoPay/OVO/DANA/PromptPay等
USDT                     WithdrawMethodUSDT          3     链上USDT提现

产品原型依据：
  - 钱包需求7.2-7.4：银行卡提现 / 电子钱包提现 / BSB提现
  - 产品原型10-1~10-4：各国提现方式
```

### 2.8 币种状态 CurrencyStatus

```
产品说法                  枚举名称                  值     说明
──────                    ──────                    ──     ──────
启用                      CurrencyStatusEnabled      1     C端可使用/充值/提现
禁用                      CurrencyStatusDisabled     0     C端不可用

产品原型依据：
  - 币种配置需求3.3："状态 - 基准币种无法禁用"
  - "禁用后C端不可使用/充值/提现"
```

---

## 三、核心数据模型（逐表详细设计）

### 3.1 currency_config — 币种配置表

**现实世界对应：** 产品原型中"系统管理 > 币种配置"页面的每一行数据。产品文档列出了20个字段。

**产品需求映射：**

```
产品原型字段                  →  数据库字段              类型         约束
─────────────                    ──────────             ─────        ─────
序号                          →  id                     int64        PK, 自增
币种代码（唯一）               →  currency_code          string       NOT NULL, 唯一索引
币种名称                      →  currency_name          string       NOT NULL
币种类型（法定/加密/平台）      →  currency_type          int32        NOT NULL (1=法定,2=加密,3=平台)
币种国家                      →  country                *string      加密/平台币为"全球"
币种图标（SVG/PNG≤5k）         →  icon_url               *string      S3 URL
货币符号                      →  symbol                 string       NOT NULL (如 "d","Rp","B","₮")
千分位符号                    →  thousand_sep           string       NOT NULL (如 ".","," )
小数点符号                    →  decimal_sep            string       NOT NULL (如 ".","," )
小数位数规则                  →  decimal_digits         int32        NOT NULL (0=VND/IDR, 2=THB/BSB, 8=USDT)
实时汇率（三方API平均值）      →  realtime_rate          int64        以 10^8 倍整数存储
汇率浮动%                     →  float_rate             int64        以 10^4 倍整数存储 (如 2.50% → 25000)
入款汇率                      →  deposit_rate           int64        = realtime × (1 + float%) 整数
出款汇率                      →  withdraw_rate          int64        = realtime × (1 - float%) 整数
平台汇率                      →  platform_rate          int64        偏差值触发更新
偏差值                        →  deviation              int64        计算值，以 10^4 倍存储
阈值%                         →  threshold              int64        以 10^4 倍整数存储
是否基准币种                  →  is_base                int32        0=否, 1=是 (全局唯一1条为1)
状态                          →  status                 int32        0=禁用, 1=启用
操作人                        →  update_by              int64        最后操作人ID
操作人名称                    →  update_by_name         string       最后操作人姓名
操作时间                      →  update_at              int64        毫秒时间戳
创建人                        →  create_by              int64        创建人ID
创建人名称                    →  create_by_name         string       创建人姓名
创建时间                      →  create_at              int64        毫秒时间戳
删除时间                      →  delete_at              int64        0=未删除
```

**Go 模型结构预览（GORM Gen 将从建表语句生成）：**

```go
type CurrencyConfig struct {
    ID             int64   `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    CurrencyCode   string  `gorm:"column:currency_code;not null;uniqueIndex;comment:币种代码(唯一)" json:"currency_code"`
    CurrencyName   string  `gorm:"column:currency_name;not null;comment:币种名称" json:"currency_name"`
    CurrencyType   int32   `gorm:"column:currency_type;not null;comment:币种类型 1法定 2加密 3平台" json:"currency_type"`
    Country        *string `gorm:"column:country;comment:币种国家(加密/平台为全球)" json:"country"`
    IconUrl        *string `gorm:"column:icon_url;comment:币种图标URL" json:"icon_url"`
    Symbol         string  `gorm:"column:symbol;not null;comment:货币符号" json:"symbol"`
    ThousandSep    string  `gorm:"column:thousand_sep;not null;comment:千分位符号" json:"thousand_sep"`
    DecimalSep     string  `gorm:"column:decimal_sep;not null;comment:小数点符号" json:"decimal_sep"`
    DecimalDigits  int32   `gorm:"column:decimal_digits;not null;comment:小数位数(0/2/8)" json:"decimal_digits"`
    RealtimeRate   int64   `gorm:"column:realtime_rate;not null;default:0;comment:实时汇率(×10^8整数)" json:"realtime_rate"`
    FloatRate      int64   `gorm:"column:float_rate;not null;default:0;comment:汇率浮动比例(×10^4,如2.50%=25000)" json:"float_rate"`
    DepositRate    int64   `gorm:"column:deposit_rate;not null;default:0;comment:入款汇率(×10^8整数)" json:"deposit_rate"`
    WithdrawRate   int64   `gorm:"column:withdraw_rate;not null;default:0;comment:出款汇率(×10^8整数)" json:"withdraw_rate"`
    PlatformRate   int64   `gorm:"column:platform_rate;not null;default:0;comment:平台汇率(×10^8整数)" json:"platform_rate"`
    Deviation      int64   `gorm:"column:deviation;not null;default:0;comment:偏差值(×10^4)" json:"deviation"`
    Threshold      int64   `gorm:"column:threshold;not null;default:0;comment:阈值(×10^4,如5%=50000)" json:"threshold"`
    IsBase         int32   `gorm:"column:is_base;not null;default:0;comment:是否基准币种 0否 1是" json:"is_base"`
    Status         int32   `gorm:"column:status;not null;default:1;comment:状态 0禁用 1启用" json:"status"`
    CreateBy       int64   `gorm:"column:create_by;not null;comment:创建人ID" json:"create_by"`
    CreateByName   string  `gorm:"column:create_by_name;not null;comment:创建人名称" json:"create_by_name"`
    CreateAt       int64   `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
    UpdateBy       int64   `gorm:"column:update_by;not null;comment:更新人ID" json:"update_by"`
    UpdateByName   string  `gorm:"column:update_by_name;not null;comment:更新人名称" json:"update_by_name"`
    UpdateAt       int64   `gorm:"column:update_at;not null;comment:更新时间(毫秒时间戳)" json:"update_at"`
    DeleteAt       int64   `gorm:"column:delete_at;not null;default:0;comment:删除时间(0=未删除)" json:"delete_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
UNIQUE INDEX idx_currency_code (currency_code)          -- 币种代码唯一
INDEX idx_status (status)                               -- 按状态筛选
INDEX idx_currency_type (currency_type)                 -- 按类型筛选
```

**关键设计说明：**

```
汇率精度方案：
  所有汇率字段用 int64 存储，乘以 10^8 后取整。

  示例（VND→USDT汇率，实际值 0.00003820）：
    存储值 = 0.00003820 × 10^8 = 3820
    读取时 = 3820 / 10^8 = 0.00003820

  示例（BSB→USDT汇率，实际值 0.10000000）：
    存储值 = 0.10000000 × 10^8 = 10000000
    读取时 = 10000000 / 10^8 = 0.10000000

  为什么 10^8：加密货币汇率需要8位小数精度（USDT就是8位），
  法币2位，统一按最高精度存储，避免多套方案。

浮动比例/阈值精度方案：
  用 int64 存储，乘以 10^4 后取整。

  示例（浮动 2.50%）：
    存储值 = 2.50 × 10^4 = 25000
    计算时：deposit_rate = realtime_rate + realtime_rate * 25000 / 1000000

  为什么 10^4：百分比最多两位小数（如 2.50%），10^4 足够。

入款/出款汇率计算规则（来自产品需求4.2）：
  入款汇率 = 实时汇率 × (1 + 汇率浮动%)
  出款汇率 = 实时汇率 × (1 - 汇率浮动%)
  偏差值 = |实时汇率 - 平台汇率| / 实时汇率 × 100
  偏差值 >= 阈值 → 更新平台汇率 + 入出款汇率 + 写汇率日志

is_base 约束：
  产品要求"基准币种只可设置1次，提交时需二级确认"。
  数据库层面只有一条 is_base=1 的记录。
  应用层面做校验：SetBaseCurrency 时检查是否已有 is_base=1。
  设置后不可更改（不可将已有的 is_base 置为0，也不可再设另一条为1）。
```

---

### 3.2 user_wallet — 用户钱包表

**现实世界对应：** 用户在每个币种下的每种钱包。产品原型2-2"币种切换列表"中每个币种后面显示的余额，就来自这张表。

**产品需求映射：**

```
产品原型概念                   →  数据库字段              类型         约束
─────────────                     ──────────             ─────        ─────
用户                           →  user_id                int64        NOT NULL
币种                           →  currency_code          string       NOT NULL
钱包类型（中心/奖励/主播/代理） →  wallet_type            int32        NOT NULL (1~5)
可用余额                       →  available_balance      int64        NOT NULL, 默认0, >= 0
冻结余额                       →  frozen_balance         int64        NOT NULL, 默认0, >= 0
钱包状态                       →  status                 int32        NOT NULL, 默认1（启用）
```

**Go 模型结构预览：**

```go
type UserWallet struct {
    ID               int64  `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    UserID           int64  `gorm:"column:user_id;not null;comment:用户ID" json:"user_id"`
    CurrencyCode     string `gorm:"column:currency_code;not null;comment:币种代码" json:"currency_code"`
    WalletType       int32  `gorm:"column:wallet_type;not null;comment:钱包类型 1中心 2奖励 3主播 4代理 5场馆" json:"wallet_type"`
    AvailableBalance int64  `gorm:"column:available_balance;not null;default:0;comment:可用余额(最小单位整数)" json:"available_balance"`
    FrozenBalance    int64  `gorm:"column:frozen_balance;not null;default:0;comment:冻结余额(最小单位整数)" json:"frozen_balance"`
    Status           int32  `gorm:"column:status;not null;default:1;comment:状态 0禁用 1启用" json:"status"`
    CreateAt         int64  `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
    UpdateAt         int64  `gorm:"column:update_at;not null;comment:更新时间(毫秒时间戳)" json:"update_at"`
    DeleteAt         int64  `gorm:"column:delete_at;not null;default:0;comment:删除时间(0=未删除)" json:"delete_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
UNIQUE INDEX idx_user_currency_type (user_id, currency_code, wallet_type)  -- 核心唯一约束
INDEX idx_user_id (user_id)                                                -- 查用户所有钱包
INDEX idx_currency_code (currency_code)                                    -- 按币种查询
```

**关键设计说明：**

```
账户模型（来自架构设计）：
  一个用户 × 一个币种 × 一个钱包类型 = 一个钱包账户

  联合唯一索引 (user_id, currency_code, wallet_type) 是本表的核心。

  示例用户 10001 可能有以下记录：
    (10001, "VND", 1)  → VND 中心钱包，可用 500000，冻结 0
    (10001, "VND", 2)  → VND 奖励钱包，可用 50000，冻结 0
    (10001, "BSB", 1)  → BSB 中心钱包，可用 1000，冻结 200
    (10001, "BSB", 2)  → BSB 奖励钱包，可用 100，冻结 0
    (10001, "IDR", 1)  → IDR 中心钱包，可用 8000000，冻结 0

金额存储规则：
  available_balance 和 frozen_balance 使用 BIGINT 存储最小货币单位的整数值。

  VND/IDR（0位小数）：1 VND = 1（最小单位就是1）
    用户看到 d 500,000 → 存储 500000

  THB/BSB（2位小数）：1.50 THB = 150（存放"分"）
    用户看到 B 1,000.50 → 存储 100050

  USDT（8位小数）：1.00000001 USDT = 100000001（存放 satoshi）
    用户看到 USDT 100.50 → 存储 10050000000

  每个币种的"放大倍数" = 10^decimal_digits，从 currency_config.decimal_digits 获取。

产品需求映射验证：
  "资金钱包余额 = 中心钱包余额 + 奖励钱包余额"
    → 查询：SELECT SUM(available_balance) WHERE user_id=? AND currency_code=? AND wallet_type IN (1,2)

  "消费时优先扣除中心钱包余额，再扣奖励钱包余额"
    → 应用层先尝试中心钱包 debit，不够部分再 debit 奖励钱包

  "切换币种即切换余额"
    → 查询条件加上 currency_code 即可切换

  "如用户下没有该币种钱包，系统需自动创建"（财务人工加款需求）
    → creditCore 中 RowsAffected=0 时自动 INSERT

自动创建时机：
  - 用户首次充值某币种到账时（CreditWallet）
  - 财务人工加款到不存在的钱包时（ManualCredit）
  - 活动/任务首次给该币种发奖时（RewardCredit）
  不建议用户注册时批量创建所有币种的钱包（按需创建，减少空数据）。
```

---

### 3.3 wallet_flow — 钱包流水表

**现实世界对应：** 每一次余额变更的记录。产品原型11-1"记录列表"和11-2"记录详情"的数据来源。同时也是对账和审计的唯一凭证。

**产品需求映射：**

```
产品原型概念                   →  数据库字段              类型         约束
─────────────                     ──────────             ─────        ─────
流水唯一标识                   →  id                     int64        PK, 自增
流水业务编号                   →  flow_no                int64        snowflake, 唯一索引
用户                           →  user_id                int64        NOT NULL
币种                           →  currency_code          string       NOT NULL
钱包类型                       →  wallet_type            int32        NOT NULL
流水类型（充值/扣款/冻结/...）  →  flow_type              int32        NOT NULL (见2.3枚举)
变更金额                       →  amount                 int64        正=入账, 负=出账
变更前余额                     →  balance_before         int64        变更前 available_balance
变更后余额                     →  balance_after          int64        变更后 available_balance
关联业务订单号（幂等键）        →  order_no               string       NOT NULL
备注                           →  remark                 *string      操作说明
扩展字段                       →  extra                  *string      JSON格式扩展信息
创建时间                       →  create_at              int64        毫秒时间戳
```

**Go 模型结构预览：**

```go
type WalletFlow struct {
    ID            int64   `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    FlowNo        int64   `gorm:"column:flow_no;not null;comment:流水编号(snowflake)" json:"flow_no"`
    UserID        int64   `gorm:"column:user_id;not null;comment:用户ID" json:"user_id"`
    CurrencyCode  string  `gorm:"column:currency_code;not null;comment:币种代码" json:"currency_code"`
    WalletType    int32   `gorm:"column:wallet_type;not null;comment:钱包类型" json:"wallet_type"`
    FlowType      int32   `gorm:"column:flow_type;not null;comment:流水类型" json:"flow_type"`
    Amount        int64   `gorm:"column:amount;not null;comment:变更金额(正入负出,最小单位)" json:"amount"`
    BalanceBefore int64   `gorm:"column:balance_before;not null;comment:变更前余额" json:"balance_before"`
    BalanceAfter  int64   `gorm:"column:balance_after;not null;comment:变更后余额" json:"balance_after"`
    OrderNo       string  `gorm:"column:order_no;not null;comment:关联业务订单号(幂等键)" json:"order_no"`
    Remark        *string `gorm:"column:remark;comment:备注说明" json:"remark"`
    Extra         *string `gorm:"column:extra;comment:扩展信息(JSON)" json:"extra"`
    CreateAt      int64   `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
UNIQUE INDEX idx_flow_no (flow_no)                                          -- 流水编号唯一
UNIQUE INDEX idx_order_flow_type (order_no, flow_type)                      -- 幂等保证
INDEX idx_user_currency_time (user_id, currency_code, create_at DESC)       -- C端记录列表查询
INDEX idx_user_flow_type (user_id, flow_type)                               -- 按类型筛选
```

**关键设计说明：**

```
幂等核心（来自架构设计第五节）：
  唯一索引 (order_no, flow_type) 是幂等的数据库层保证。

  为什么加 flow_type：同一个 order_no 可能产生多种类型的流水。
  例如提现操作：
    order_no="W20260226001", flow_type=4 (提现冻结) → 冻结时写入
    order_no="W20260226001", flow_type=5 (提现确认) → 出款成功时写入
  这两条是同一笔提现的不同阶段，flow_type 不同，算不同操作。

  但同一个 order_no + 同一个 flow_type 不允许重复写入。
  重复请求时唯一索引冲突 → 查出已有记录直接返回。

balance_before / balance_after 的价值：
  - 对账：任意时刻，最后一条流水的 balance_after 应等于 user_wallet 的 available_balance
  - 排查：出了问题可以回溯每一步的余额变化
  - 审计：完整的变更链条，每条流水都记录了上下文

产品记录模块的映射：
  产品需求8.1"记录分类：提现、充值、兑换、奖励四Tab"
  → 按 flow_type 分类筛选：
    提现 Tab：flow_type IN (4,5,6)
    充值 Tab：flow_type IN (1,10)
    兑换 Tab：flow_type IN (7,8,9)
    奖励 Tab：flow_type IN (3,13,15)

extra 字段用途：
  存储不同业务场景的附加信息（JSON格式），避免为每种场景加列。
  例如：
    兑换流水：{"source_currency":"VND","target_currency":"BSB","rate":3820,"bonus_percent":10}
    充值流水：{"deposit_method":"USDT","channel_order_no":"CH20260226001"}
    人工加款：{"manual_type":"运营补偿","operator_id":888,"operator_name":"admin"}

注意：wallet_flow 不做软删除。资金流水一旦写入永远不删除。没有 delete_at 字段。
```

---

### 3.4 wallet_freeze_record — 冻结记录表

**现实世界对应：** 用户提现时，系统从可用余额冻结一部分资金。这条记录跟踪冻结的生命周期：冻结 → 确认扣除/解冻退回。

**产品需求映射：**

```
产品原型概念                   →  数据库字段              类型         约束
─────────────                     ──────────             ─────        ─────
冻结记录标识                   →  id                     int64        PK, 自增
关联业务订单号                 →  order_no               string       NOT NULL, 唯一索引
用户                           →  user_id                int64        NOT NULL
币种                           →  currency_code          string       NOT NULL
钱包类型                       →  wallet_type            int32        NOT NULL
冻结金额                       →  amount                 int64        NOT NULL
冻结状态                       →  status                 int32        1=冻结中, 2=已解冻, 3=已确认
冻结时间                       →  freeze_at              int64        毫秒时间戳
完结时间                       →  finish_at              *int64       解冻/确认的时间
完结备注                       →  finish_remark          *string      驳回原因等
创建时间                       →  create_at              int64        毫秒时间戳
更新时间                       →  update_at              int64        毫秒时间戳
```

**Go 模型结构预览：**

```go
type WalletFreezeRecord struct {
    ID           int64   `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    OrderNo      string  `gorm:"column:order_no;not null;uniqueIndex;comment:关联业务订单号" json:"order_no"`
    UserID       int64   `gorm:"column:user_id;not null;comment:用户ID" json:"user_id"`
    CurrencyCode string  `gorm:"column:currency_code;not null;comment:币种代码" json:"currency_code"`
    WalletType   int32   `gorm:"column:wallet_type;not null;comment:钱包类型" json:"wallet_type"`
    Amount       int64   `gorm:"column:amount;not null;comment:冻结金额(最小单位整数)" json:"amount"`
    Status       int32   `gorm:"column:status;not null;default:1;comment:冻结状态 1冻结中 2已解冻 3已确认" json:"status"`
    FreezeAt     int64   `gorm:"column:freeze_at;not null;comment:冻结时间(毫秒时间戳)" json:"freeze_at"`
    FinishAt     *int64  `gorm:"column:finish_at;comment:完结时间(毫秒时间戳)" json:"finish_at"`
    FinishRemark *string `gorm:"column:finish_remark;comment:完结备注(驳回原因等)" json:"finish_remark"`
    CreateAt     int64   `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
    UpdateAt     int64   `gorm:"column:update_at;not null;comment:更新时间(毫秒时间戳)" json:"update_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
UNIQUE INDEX idx_order_no (order_no)                           -- 一个订单对应一条冻结
INDEX idx_user_status (user_id, status)                        -- 查用户的冻结中记录
INDEX idx_user_currency (user_id, currency_code)               -- 查某币种冻结
```

**关键设计说明：**

```
为什么需要独立的冻结记录表：
  1. wallet_flow 记录的是已经发生的余额变更，冻结是一个"进行中"的状态
  2. 冻结有状态机（Frozen → Unfrozen/Confirmed），需要 WHERE status=? 做状态转换
  3. order_no 唯一索引确保同一笔提现只有一条冻结记录

状态机实现（来自架构设计第六节）：
  冻结 → 解冻（驳回/失败时）：
    UPDATE wallet_freeze_record SET status=2, finish_at=?, finish_remark=?
    WHERE order_no=? AND status=1
    RowsAffected=0 说明状态已变（已被确认或已解冻），互斥保证。

  冻结 → 确认（出款成功时）：
    UPDATE wallet_freeze_record SET status=3, finish_at=?
    WHERE order_no=? AND status=1
    同理，RowsAffected=0 说明状态已变。

  终态不可逆：已解冻/已确认后不可再变。

注意：此表不做软删除。冻结记录是资金审计凭证，不删除。
```

---

### 3.5 deposit_order — 充值订单表

**现实世界对应：** 用户每次发起充值产生的一条订单。产品原型7-2"充值进行中"弹窗中展示的订单信息、产品原型11-1"充值"Tab的列表数据。

**产品需求映射：**

```
产品原型概念                   →  数据库字段              类型         约束
─────────────                     ──────────             ─────        ─────
订单标识                       →  id                     int64        PK, 自增
平台订单号                     →  order_no               string       NOT NULL, snowflake, 唯一
用户                           →  user_id                int64        NOT NULL
币种                           →  currency_code          string       NOT NULL
订单金额（用户看到的金额）      →  order_amount           int64        最小单位整数
充值方式                       →  deposit_method         string       NOT NULL (如"USDT_TRC20","BANK_VN","QRIS")
订单状态                       →  status                 int32        NOT NULL (1~5 见枚举)
通道订单号（三方）             →  channel_order_no       *string      财务模块返回
— 汇率相关（BSB充值时使用）—
使用的入款汇率快照             →  rate_snapshot          *int64       创建订单时锁定的入款汇率
支付币种（实际支付的币种）      →  pay_currency           *string      BSB充值时为VND/IDR/THB/USDT
支付金额（实际支付金额）        →  pay_amount             *int64       按汇率换算后的支付金额
— 奖金相关 —
奖金活动ID                     →  bonus_activity_id      *int64       选择的奖金活动
奖金金额                       →  bonus_amount           *int64       奖金到账金额
奖金流水倍数要求               →  bonus_turnover         *int32       完成X倍流水后可提
— 时间 —
创建时间                       →  create_at              int64        毫秒时间戳
支付完成时间                   →  paid_at                *int64       支付成功/失败的时间
过期时间                       →  expire_at              *int64       订单失效时间
— 审计 —
失败原因                       →  fail_reason            *string      支付超时/支付异常/卡片问题
更新时间                       →  update_at              int64        毫秒时间戳
删除时间                       →  delete_at              int64        0=未删除
```

**Go 模型结构预览：**

```go
type DepositOrder struct {
    ID              int64   `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    OrderNo         string  `gorm:"column:order_no;not null;uniqueIndex;comment:平台订单号(snowflake)" json:"order_no"`
    UserID          int64   `gorm:"column:user_id;not null;comment:用户ID" json:"user_id"`
    CurrencyCode    string  `gorm:"column:currency_code;not null;comment:币种代码" json:"currency_code"`
    OrderAmount     int64   `gorm:"column:order_amount;not null;comment:订单金额(最小单位整数)" json:"order_amount"`
    DepositMethod   string  `gorm:"column:deposit_method;not null;comment:充值方式" json:"deposit_method"`
    Status          int32   `gorm:"column:status;not null;default:1;comment:订单状态 1待支付 2进行中 3成功 4失败 5失效" json:"status"`
    ChannelOrderNo  *string `gorm:"column:channel_order_no;comment:通道订单号(三方)" json:"channel_order_no"`
    RateSnapshot    *int64  `gorm:"column:rate_snapshot;comment:入款汇率快照(×10^8,BSB充值时)" json:"rate_snapshot"`
    PayCurrency     *string `gorm:"column:pay_currency;comment:支付币种(BSB充值时)" json:"pay_currency"`
    PayAmount       *int64  `gorm:"column:pay_amount;comment:支付金额(最小单位整数)" json:"pay_amount"`
    BonusActivityID *int64  `gorm:"column:bonus_activity_id;comment:奖金活动ID" json:"bonus_activity_id"`
    BonusAmount     *int64  `gorm:"column:bonus_amount;comment:奖金金额(最小单位整数)" json:"bonus_amount"`
    BonusTurnover   *int32  `gorm:"column:bonus_turnover;comment:奖金流水倍数要求" json:"bonus_turnover"`
    FailReason      *string `gorm:"column:fail_reason;comment:失败原因" json:"fail_reason"`
    PaidAt          *int64  `gorm:"column:paid_at;comment:支付完成时间(毫秒时间戳)" json:"paid_at"`
    ExpireAt        *int64  `gorm:"column:expire_at;comment:订单过期时间(毫秒时间戳)" json:"expire_at"`
    CreateAt        int64   `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
    UpdateAt        int64   `gorm:"column:update_at;not null;comment:更新时间(毫秒时间戳)" json:"update_at"`
    DeleteAt        int64   `gorm:"column:delete_at;not null;default:0;comment:删除时间(0=未删除)" json:"delete_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
UNIQUE INDEX idx_order_no (order_no)                                           -- 订单号唯一
INDEX idx_user_currency_status (user_id, currency_code, status, create_at DESC) -- C端记录列表 + 重复充值检测
INDEX idx_user_time (user_id, create_at DESC)                                  -- 按时间查
INDEX idx_status (status)                                                      -- 按状态查（后台管理）
```

**关键设计说明：**

```
重复转账检测（产品需求5.5）依赖此表：
  查询条件：user_id=? AND currency_code=? AND order_amount=? AND status IN (1,2)
  → 统计匹配订单数 → 1-2笔提示，≥3笔拦截
  索引 idx_user_currency_status 精准覆盖此查询。

汇率快照字段（rate_snapshot）：
  BSB充值时，用户看到的汇率是下单时的入款汇率。
  从下单到通道回调可能过了几分钟甚至更久，汇率可能已经变了。
  所以必须在创建订单时锁定当时的汇率，回调上账时用这个快照汇率计算。
  产品原型5-2/5-3/5-4中BSB充值展示的"1BSB=X法币"就是这个快照值。

奖金相关字段：
  产品需求5.6"额外奖金选择规则"：
  - bonus_activity_id：用户选择的奖金活动
  - bonus_amount：该活动给的奖金金额
  - bonus_turnover：赠送部分需完成X倍流水后可提现
  充值成功后，主金额 → 中心钱包，奖金金额 → 奖励钱包。

失败原因映射：
  产品需求8.4详情页：
  "充值失败原因：支付超时、支付异常、卡片问题"
  → fail_reason 存文本，对应 i18n key。

充值订单与财务充值记录的关系：
  财务需求2.3"充值记录"有24个字段，包含通道名称/充值费率/手续费/基准币种折算等。
  这些通道侧信息由财务模块存储在自己的表中，ser-wallet 只存钱包侧的订单信息。
  两者通过 order_no 关联。
```

---

### 3.6 withdraw_order — 提现订单表

**现实世界对应：** 用户每次发起提现产生的一条订单。包含从创建到审核到出款的全生命周期。

**产品需求映射：**

```
产品原型概念                   →  数据库字段              类型         约束
─────────────                     ──────────             ─────        ─────
订单标识                       →  id                     int64        PK, 自增
平台订单号                     →  order_no               string       NOT NULL, snowflake, 唯一
用户                           →  user_id                int64        NOT NULL
币种                           →  currency_code          string       NOT NULL
提现金额                       →  order_amount           int64        最小单位整数
提现方式                       →  withdraw_method        int32        NOT NULL (1银行卡 2电子钱包 3USDT)
订单状态（7种）                →  status                 int32        NOT NULL (1~7 见枚举)
关联提现账户ID                 →  withdraw_account_id    int64        NOT NULL
— 汇率相关（BSB提现时使用）—
使用的出款汇率快照             →  rate_snapshot          *int64       创建订单时锁定的出款汇率
出款币种                       →  pay_currency           *string      BSB提现时实际出款的币种
出款金额                       →  pay_amount             *int64       按出款汇率换算后的金额
— 费用 —
手续费                         →  fee_amount             int64        默认0
手续费率                       →  fee_rate               *int64       百分比 × 10^4
— 冻结关联 —
冻结记录订单号                 →  freeze_order_no        *string      关联 wallet_freeze_record
— 审核 —
拒绝原因                       →  reject_reason          *string      驳回原因
审核备注                       →  audit_remark           *string      审核人备注
— 财务通道 —
通道订单号                     →  channel_order_no       *string      财务出款后回填
— 时间 —
创建时间                       →  create_at              int64        毫秒时间戳
完成时间                       →  finish_at              *int64       出款完成/最终失败时间
更新时间                       →  update_at              int64        毫秒时间戳
删除时间                       →  delete_at              int64        0=未删除
```

**Go 模型结构预览：**

```go
type WithdrawOrder struct {
    ID                int64   `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    OrderNo           string  `gorm:"column:order_no;not null;uniqueIndex;comment:平台订单号(snowflake)" json:"order_no"`
    UserID            int64   `gorm:"column:user_id;not null;comment:用户ID" json:"user_id"`
    CurrencyCode      string  `gorm:"column:currency_code;not null;comment:币种代码" json:"currency_code"`
    OrderAmount       int64   `gorm:"column:order_amount;not null;comment:提现金额(最小单位整数)" json:"order_amount"`
    WithdrawMethod    int32   `gorm:"column:withdraw_method;not null;comment:提现方式 1银行卡 2电子钱包 3USDT" json:"withdraw_method"`
    Status            int32   `gorm:"column:status;not null;default:1;comment:状态 1待风控 2风控驳回 3待财务 4财务驳回 5待出款 6出款成功 7出款失败" json:"status"`
    WithdrawAccountID int64   `gorm:"column:withdraw_account_id;not null;comment:提现账户ID" json:"withdraw_account_id"`
    RateSnapshot      *int64  `gorm:"column:rate_snapshot;comment:出款汇率快照(×10^8,BSB提现时)" json:"rate_snapshot"`
    PayCurrency       *string `gorm:"column:pay_currency;comment:出款币种(BSB提现时)" json:"pay_currency"`
    PayAmount         *int64  `gorm:"column:pay_amount;comment:出款金额(最小单位整数)" json:"pay_amount"`
    FeeAmount         int64   `gorm:"column:fee_amount;not null;default:0;comment:手续费(最小单位整数)" json:"fee_amount"`
    FeeRate           *int64  `gorm:"column:fee_rate;comment:手续费率(×10^4)" json:"fee_rate"`
    FreezeOrderNo     *string `gorm:"column:freeze_order_no;comment:冻结记录订单号" json:"freeze_order_no"`
    RejectReason      *string `gorm:"column:reject_reason;comment:驳回原因" json:"reject_reason"`
    AuditRemark       *string `gorm:"column:audit_remark;comment:审核备注" json:"audit_remark"`
    ChannelOrderNo    *string `gorm:"column:channel_order_no;comment:通道订单号" json:"channel_order_no"`
    FinishAt          *int64  `gorm:"column:finish_at;comment:完成时间(毫秒时间戳)" json:"finish_at"`
    CreateAt          int64   `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
    UpdateAt          int64   `gorm:"column:update_at;not null;comment:更新时间(毫秒时间戳)" json:"update_at"`
    DeleteAt          int64   `gorm:"column:delete_at;not null;default:0;comment:删除时间(0=未删除)" json:"delete_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
UNIQUE INDEX idx_order_no (order_no)                                           -- 订单号唯一
INDEX idx_user_currency_status (user_id, currency_code, status, create_at DESC) -- C端记录列表
INDEX idx_user_time (user_id, create_at DESC)                                  -- 按时间查
INDEX idx_status (status)                                                      -- 后台按状态筛选（待审核等）
INDEX idx_freeze_order (freeze_order_no)                                       -- 关联冻结记录
```

**关键设计说明：**

```
提现状态机（7种状态，来自财务需求2.4）：
  1 待风控审核 → 2 风控驳回（触发解冻）
  1 待风控审核 → 3 待财务审核
  3 待财务审核 → 4 财务驳回（触发解冻）
  3 待财务审核 → 5 待出款
  5 待出款     → 6 出款成功（触发确认扣除）
  5 待出款     → 7 出款失败（触发解冻）

  驳回/失败 → ser-wallet.UnfreezeBalance(order_no) → 冻结退回可用
  出款成功 → ser-wallet.ConfirmDebit(order_no) → 冻结永久减少

提现订单创建流程中的冻结关联：
  1. ser-wallet 创建本地提现订单（status=1）
  2. ser-wallet.freezeCore(order_no, amount) → 冻结金额
  3. freeze_order_no 指向 wallet_freeze_record 的 order_no
  4. 调财务模块创建审核订单
  5. 财务审核/出款后回调 → 根据 freeze_order_no 找到冻结记录 → 确认/解冻

产品原型映射：
  产品原型10-5"提现进行中"弹窗：提现方式、订单号、提现金额、订单时间
  产品原型11-1"提现Tab"：金额、方式、时间、状态（已完成/审核中/失败）
  产品原型11-2"提现详情"：订单号、金额明细、状态、时间、方式、失败原因

  提现失败原因（产品需求8.4）：
  "信息有误、系统维护、额度受限" → reject_reason / 通道失败原因

手续费规则：
  产品需求7.5"提现限制（后台配置）"：手续费规则
  财务需求2.22提现方式配置：提现费率（手续费+百分比）/ 大额提现费率
  手续费 = 固定费用 + 提现金额 × 费率百分比
  fee_amount 存计算后的最终手续费，fee_rate 存使用的费率。
```

---

### 3.7 withdraw_account — 提现账户表

**现实世界对应：** 用户绑定的提现渠道信息。产品原型10-1~10-4中"首次提现需填写账户信息"，"后续复用（只读）"对应的数据。

**产品需求映射：**

```
产品原型概念                       →  数据库字段              类型         约束
─────────────                         ──────────             ─────        ─────
账户标识                           →  id                     int64        PK, 自增
用户                               →  user_id                int64        NOT NULL
币种                               →  currency_code          string       NOT NULL
提现方式                           →  withdraw_method        int32        NOT NULL (1银行卡 2电子钱包 3USDT)
— 银行卡信息 —
银行名称                           →  bank_name              *string      加密存储
银行账号                           →  bank_account           *string      加密存储
开户网点                           →  bank_branch            *string
— 电子钱包信息 —
钱包类型（GoPay/OVO/DANA等）       →  ewallet_type           *string
手机号                             →  phone_number           *string      加密存储
— USDT信息 —
网络类型（TRC20/ERC20/BEP20）      →  network_type           *string
钱包地址                           →  wallet_address         *string
— 通用字段 —
账户持有人（= KYC姓名）            →  account_holder         string       NOT NULL, 加密存储
是否默认                           →  is_default             int32        0否 1是
创建时间                           →  create_at              int64        毫秒时间戳
更新时间                           →  update_at              int64        毫秒时间戳
删除时间                           →  delete_at              int64        0=未删除
```

**Go 模型结构预览：**

```go
type WithdrawAccount struct {
    ID             int64   `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    UserID         int64   `gorm:"column:user_id;not null;comment:用户ID" json:"user_id"`
    CurrencyCode   string  `gorm:"column:currency_code;not null;comment:币种代码" json:"currency_code"`
    WithdrawMethod int32   `gorm:"column:withdraw_method;not null;comment:提现方式 1银行卡 2电子钱包 3USDT" json:"withdraw_method"`
    BankName       *string `gorm:"column:bank_name;comment:银行名称(加密)" json:"bank_name"`
    BankAccount    *string `gorm:"column:bank_account;comment:银行账号(加密)" json:"bank_account"`
    BankBranch     *string `gorm:"column:bank_branch;comment:开户网点" json:"bank_branch"`
    EwalletType    *string `gorm:"column:ewallet_type;comment:电子钱包类型" json:"ewallet_type"`
    PhoneNumber    *string `gorm:"column:phone_number;comment:手机号(加密)" json:"phone_number"`
    NetworkType    *string `gorm:"column:network_type;comment:网络类型(TRC20/ERC20/BEP20)" json:"network_type"`
    WalletAddress  *string `gorm:"column:wallet_address;comment:USDT钱包地址" json:"wallet_address"`
    AccountHolder  string  `gorm:"column:account_holder;not null;comment:账户持有人(=KYC姓名,加密)" json:"account_holder"`
    IsDefault      int32   `gorm:"column:is_default;not null;default:0;comment:是否默认 0否 1是" json:"is_default"`
    CreateAt       int64   `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
    UpdateAt       int64   `gorm:"column:update_at;not null;comment:更新时间(毫秒时间戳)" json:"update_at"`
    DeleteAt       int64   `gorm:"column:delete_at;not null;default:0;comment:删除时间(0=未删除)" json:"delete_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
INDEX idx_user_currency_method (user_id, currency_code, withdraw_method)   -- 查用户特定方式的账户
INDEX idx_user_id (user_id)                                                -- 查用户所有提现账户
```

**关键设计说明：**

```
为什么不拆分成三张表（银行卡表/电子钱包表/USDT表）：
  虽然三种方式的字段不同，但共性大于差异（都有 user_id/currency_code/account_holder）。
  拆三张表会增加查询复杂度（查用户所有提现方式需要 UNION 三张表）。
  用一张表 + 各方式特有字段设为可空（*string），更简单。
  这也是业界支付系统常见的"多态表"模式。

敏感字段加密：
  银行账号、手机号、账户持有人属于敏感信息。
  现有工程 ser-kyc 使用 EncryptedString 类型实现 GORM 自动加解密。

  在应用层使用时，GORM 自动解密读出明文，写入时自动加密。
  数据库中存储的是密文，即使被拖库也不会泄露。

  实现方式（参考 common/pkg/utils/crypto）：
  bank_account / phone_number / account_holder 在写入时用 AES 加密，
  读出时自动解密。具体加密方式取决于 EncryptedString 的实现。

账户持有人与 KYC 强关联：
  产品需求7.1："账户持有人姓名 = KYC认证姓名（提现页自动填充已实名的姓名且不可编辑）"
  产品原型10-3注释："账户持有人名称必须与KYC一致，忽略大小写和多余空格"

  后端校验逻辑：
    SaveWithdrawAccount 时：
      1. 调 ser-user.GetUserInfoExpend 获取 KYC 认证姓名
      2. 对比 account_holder 与 KYC 姓名（trim + 忽略大小写）
      3. 不一致 → 返回 WithdrawAccountNameMismatch 错误

  这是后端强校验，不能只依赖前端置灰。

每种方式的首次/复用逻辑：
  产品需求："首次提现需输入账户信息，提现账户信息平台记录，后续提现无需重复填写"
  → GetWithdrawAccount(user_id, currency_code, withdraw_method)
    → 有记录：返回账户信息（只读展示），直接进入金额输入
    → 无记录：返回空，前端展示完整表单
```

---

### 3.8 exchange_record — 兑换记录表

**现实世界对应：** 用户每次法币→BSB兑换产生的一条记录。产品原型8"兑换BSB"和原型11-1"兑换Tab"的数据。

**产品需求映射：**

```
产品原型概念                       →  数据库字段              类型         约束
─────────────                         ──────────             ─────        ─────
记录标识                           →  id                     int64        PK, 自增
兑换记录编号                       →  record_no              string       NOT NULL, snowflake, 唯一
用户                               →  user_id                int64        NOT NULL
源币种（法币）                      →  source_currency        string       NOT NULL (VND/IDR/THB)
目标币种（固定BSB）                 →  target_currency        string       NOT NULL (默认"BSB")
兑换金额（法币金额）               →  source_amount          int64        最小单位整数
兑换获得（BSB金额）                →  target_amount          int64        最小单位整数
赠送比例（如10%）                  →  bonus_percent          int32        百分比整数 (10=10%)
赠送金额（BSB）                    →  bonus_amount           int64        最小单位整数
实际到账（= 兑换获得 + 赠送）       →  total_amount           int64        最小单位整数
使用的平台汇率快照                 →  rate_snapshot          int64        兑换时锁定的汇率 ×10^8
流水倍数要求                       →  turnover_multiple      *int32       赠送部分需完成X倍流水
创建时间                           →  create_at              int64        毫秒时间戳
```

**Go 模型结构预览：**

```go
type ExchangeRecord struct {
    ID               int64   `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    RecordNo         string  `gorm:"column:record_no;not null;uniqueIndex;comment:兑换记录编号(snowflake)" json:"record_no"`
    UserID           int64   `gorm:"column:user_id;not null;comment:用户ID" json:"user_id"`
    SourceCurrency   string  `gorm:"column:source_currency;not null;comment:源币种(法币)" json:"source_currency"`
    TargetCurrency   string  `gorm:"column:target_currency;not null;default:BSB;comment:目标币种(BSB)" json:"target_currency"`
    SourceAmount     int64   `gorm:"column:source_amount;not null;comment:兑换金额-法币(最小单位)" json:"source_amount"`
    TargetAmount     int64   `gorm:"column:target_amount;not null;comment:兑换获得-BSB(最小单位)" json:"target_amount"`
    BonusPercent     int32   `gorm:"column:bonus_percent;not null;default:0;comment:赠送比例(10=10%)" json:"bonus_percent"`
    BonusAmount      int64   `gorm:"column:bonus_amount;not null;default:0;comment:赠送金额-BSB(最小单位)" json:"bonus_amount"`
    TotalAmount      int64   `gorm:"column:total_amount;not null;comment:实际到账-BSB(最小单位)" json:"total_amount"`
    RateSnapshot     int64   `gorm:"column:rate_snapshot;not null;comment:使用的平台汇率(×10^8)" json:"rate_snapshot"`
    TurnoverMultiple *int32  `gorm:"column:turnover_multiple;comment:赠送流水倍数要求" json:"turnover_multiple"`
    CreateAt         int64   `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
UNIQUE INDEX idx_record_no (record_no)                                    -- 记录号唯一
INDEX idx_user_source_time (user_id, source_currency, create_at DESC)     -- C端兑换记录列表
INDEX idx_user_time (user_id, create_at DESC)                             -- 按时间查
```

**关键设计说明：**

```
兑换计算公式映射：
  产品需求6.3"兑换计算"：
    兑换获得 BSB = source_amount / 汇率（1BSB = X法币）
    赠送金额 = 兑换获得 × bonus_percent%
    实际到账 = 兑换获得 + 赠送金额

  整数计算示例（VND→BSB，1BSB=2620VND，赠送10%）：
    用户输入：d 262,000（VND）
    source_amount = 262000（VND最小单位就是1）
    汇率快照 rate_snapshot = 2620_00000000（2620 × 10^8）
    target_amount = 262000 / 2620 = 100（BSB，2位小数 → 100 = 1.00 BSB × 100 ...

    注意：这里的计算涉及两个币种不同的 decimal_digits，需要在应用层处理。
    具体：
      VND decimal_digits=0，所以 262000 就是 262000 VND
      BSB decimal_digits=2，所以 10000 代表 100.00 BSB
      target_amount = source_amount × 10^(BSB.digits) / rate_in_source_units
    详细的计算逻辑在 Service 层实现，数据模型只存结果。

兑换事务范围：
  一次兑换在同一个数据库事务中：
    1. INSERT exchange_record
    2. UPDATE user_wallet (法币, center) SET balance -= source_amount
    3. UPDATE user_wallet (BSB, center) SET balance += target_amount
    4. UPDATE user_wallet (BSB, reward) SET balance += bonus_amount（如果有赠送）
    5. INSERT wallet_flow (法币扣减)
    6. INSERT wallet_flow (BSB到账)
    7. INSERT wallet_flow (BSB赠送到账)（如果有赠送）

产品原型映射：
  原型8"兑换BSB"四个页面：余额、汇率、兑换金额、兑换获得、赠送10%、实际到账
  原型11-1"兑换Tab"：兑换金额、赠送x%、实际到账、时间

注意：此表不做软删除。兑换记录一旦写入不删除，是资金审计凭证。
```

---

### 3.9 exchange_rate_log — 汇率变更日志表

**现实世界对应：** 产品原型"操作日志 > 汇率日志"Tab 的数据。每次平台汇率因偏差值超过阈值而更新时，记一条日志。

**产品需求映射：**

```
产品原型概念（产品需求4.4）     →  数据库字段              类型         约束
─────────────                     ──────────             ─────        ─────
序号                           →  id                     int64        PK, 自增
日志编号（唯一编码）            →  log_no                 string       NOT NULL, 唯一索引
更新时间                       →  update_time            int64        汇率更新的时间, 毫秒时间戳
币种代码                       →  currency_code          string       NOT NULL
实时汇率                       →  realtime_rate          int64        ×10^8
平台汇率                       →  platform_rate          int64        ×10^8
入款汇率                       →  deposit_rate           int64        ×10^8
出款汇率                       →  withdraw_rate          int64        ×10^8
浮动比例                       →  float_rate             int64        ×10^4
偏差值                         →  deviation              int64        ×10^4
阈值                           →  threshold              int64        ×10^4
创建时间                       →  create_at              int64        毫秒时间戳
```

**Go 模型结构预览：**

```go
type ExchangeRateLog struct {
    ID           int64  `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    LogNo        string `gorm:"column:log_no;not null;uniqueIndex;comment:日志编号(snowflake)" json:"log_no"`
    UpdateTime   int64  `gorm:"column:update_time;not null;comment:汇率更新时间(毫秒时间戳)" json:"update_time"`
    CurrencyCode string `gorm:"column:currency_code;not null;comment:币种代码" json:"currency_code"`
    RealtimeRate int64  `gorm:"column:realtime_rate;not null;comment:实时汇率(×10^8)" json:"realtime_rate"`
    PlatformRate int64  `gorm:"column:platform_rate;not null;comment:平台汇率(×10^8)" json:"platform_rate"`
    DepositRate  int64  `gorm:"column:deposit_rate;not null;comment:入款汇率(×10^8)" json:"deposit_rate"`
    WithdrawRate int64  `gorm:"column:withdraw_rate;not null;comment:出款汇率(×10^8)" json:"withdraw_rate"`
    FloatRate    int64  `gorm:"column:float_rate;not null;comment:浮动比例(×10^4)" json:"float_rate"`
    Deviation    int64  `gorm:"column:deviation;not null;comment:偏差值(×10^4)" json:"deviation"`
    Threshold    int64  `gorm:"column:threshold;not null;comment:阈值(×10^4)" json:"threshold"`
    CreateAt     int64  `gorm:"column:create_at;not null;comment:创建时间(毫秒时间戳)" json:"create_at"`
}
```

**索引设计：**

```
PRIMARY KEY (id)
UNIQUE INDEX idx_log_no (log_no)                              -- 日志编号唯一
INDEX idx_currency_time (currency_code, update_time DESC)     -- 按币种+时间查
INDEX idx_update_time (update_time DESC)                      -- 按时间查（B端列表）
```

**关键设计说明：**

```
产品需求完全对应：
  币种配置需求4.4"汇率日志"明确列出了7个字段：
    序号、日志编号、更新时间、币种代码、实时汇率、平台汇率、入款汇率、出款汇率
  模型完整覆盖这7个字段，额外增加了浮动比例/偏差值/阈值用于审计追溯。

写入时机：
  定时任务刷新汇率时，如果偏差值 >= 阈值，在更新 currency_config 的同一事务中写入此日志。
  保证汇率变更和日志记录的原子性。

产品原型映射：
  币种配置原型3"操作日志 > 汇率日志Tab"：
  筛选条件（更新时间/币种代码），表格含 VDN/THB/BTC/IDR/TRX 示例数据

注意：此表只插入不更新不删除，是纯追加日志表。
```

---

## 四、模型间关系总图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        数据模型关系图                                     │
│                                                                         │
│  ┌──────────────────┐                                                   │
│  │ currency_config   │                                                   │
│  │ (币种配置)        │◄─────────────────────────────────────────┐        │
│  │                   │                                          │        │
│  │ PK: id            │                                          │        │
│  │ UK: currency_code │                                          │        │
│  └────────┬──────────┘                                          │        │
│           │                                                     │        │
│           │ currency_code                                       │        │
│           │ (被所有表引用)                                       │        │
│           │                                                     │        │
│  ┌────────▼──────────┐        ┌───────────────────┐            │        │
│  │ user_wallet        │        │ wallet_flow        │            │        │
│  │ (用户钱包)         │        │ (钱包流水)         │            │        │
│  │                    │        │                    │            │        │
│  │ PK: id             │        │ PK: id             │            │        │
│  │ UK: user_id +      │        │ UK: order_no +     │            │        │
│  │     currency_code +│        │     flow_type      │            │        │
│  │     wallet_type    │        │ FK: user_id        │            │        │
│  │                    │        │ FK: currency_code   │            │        │
│  └────────────────────┘        └───────────────────┘            │        │
│                                                                  │        │
│  ┌────────────────────┐        ┌───────────────────────┐        │        │
│  │ deposit_order       │        │ wallet_freeze_record   │        │        │
│  │ (充值订单)          │        │ (冻结记录)             │        │        │
│  │                     │        │                        │        │        │
│  │ PK: id              │        │ PK: id                 │        │        │
│  │ UK: order_no        │        │ UK: order_no           │        │        │
│  │ FK: user_id         │        │ FK: user_id            │        │        │
│  │ FK: currency_code──►│────────│ FK: currency_code──────│────────┘        │
│  └─────────────────────┘        └───────────┬────────────┘                 │
│                                              │                              │
│  ┌─────────────────────┐        ┌────────────▼───────────┐                 │
│  │ exchange_record      │        │ withdraw_order          │                 │
│  │ (兑换记录)           │        │ (提现订单)              │                 │
│  │                      │        │                         │                 │
│  │ PK: id               │        │ PK: id                  │                 │
│  │ UK: record_no        │        │ UK: order_no             │                 │
│  │ FK: user_id          │        │ FK: user_id              │                 │
│  │ FK: source_currency  │        │ FK: currency_code        │                 │
│  │ FK: target_currency  │        │ FK: withdraw_account_id──│─┐               │
│  └──────────────────────┘        │ FK: freeze_order_no ─────│─│─→ freeze.order_no
│                                  └──────────────────────────┘ │               │
│  ┌──────────────────────┐                                     │               │
│  │ withdraw_account      │◄────────────────────────────────────┘               │
│  │ (提现账户)            │                                                     │
│  │                       │                                                     │
│  │ PK: id                │                                                     │
│  │ FK: user_id           │                                                     │
│  │ FK: currency_code     │                                                     │
│  └───────────────────────┘                                                     │
│                                                                                │
│  ┌────────────────────────┐                                                    │
│  │ exchange_rate_log       │                                                    │
│  │ (汇率变更日志)          │                                                    │
│  │                         │                                                    │
│  │ PK: id                  │                                                    │
│  │ UK: log_no              │                                                    │
│  │ FK: currency_code       │                                                    │
│  └─────────────────────────┘                                                    │
│                                                                                │
│  关系说明：                                                                    │
│    所有表的 currency_code → currency_config.currency_code（逻辑外键，不加物理FK）│
│    withdraw_order.withdraw_account_id → withdraw_account.id（逻辑外键）        │
│    withdraw_order.freeze_order_no → wallet_freeze_record.order_no（逻辑外键）  │
│    所有表的 user_id → ser-user 服务的用户ID（跨服务，无物理FK）                │
│                                                                                │
│  为什么不用物理外键：                                                          │
│    1. TiDB 推荐不使用外键约束（影响分布式事务性能）                              │
│    2. 微服务架构下 user_id 是跨服务引用，无法建物理外键                          │
│    3. 数据完整性在应用层保证（Service 层校验）                                  │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、建表语句参考（DDL）

以下 DDL 基于 TiDB（兼容 MySQL 语法），字段顺序、类型、注释完整，可直接用于建表后执行 GORM Gen 生成代码。

### 5.1 currency_config

```sql
CREATE TABLE `currency_config` (
    `id`              BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `currency_code`   VARCHAR(16)  NOT NULL COMMENT '币种代码(唯一)',
    `currency_name`   VARCHAR(64)  NOT NULL COMMENT '币种名称',
    `currency_type`   INT          NOT NULL COMMENT '币种类型 1法定 2加密 3平台',
    `country`         VARCHAR(64)  DEFAULT NULL COMMENT '币种国家(加密/平台为全球)',
    `icon_url`        VARCHAR(512) DEFAULT NULL COMMENT '币种图标URL',
    `symbol`          VARCHAR(16)  NOT NULL COMMENT '货币符号',
    `thousand_sep`    VARCHAR(4)   NOT NULL COMMENT '千分位符号',
    `decimal_sep`     VARCHAR(4)   NOT NULL COMMENT '小数点符号',
    `decimal_digits`  INT          NOT NULL COMMENT '小数位数(0/2/8)',
    `realtime_rate`   BIGINT       NOT NULL DEFAULT 0 COMMENT '实时汇率(×10^8整数)',
    `float_rate`      BIGINT       NOT NULL DEFAULT 0 COMMENT '汇率浮动比例(×10^4)',
    `deposit_rate`    BIGINT       NOT NULL DEFAULT 0 COMMENT '入款汇率(×10^8整数)',
    `withdraw_rate`   BIGINT       NOT NULL DEFAULT 0 COMMENT '出款汇率(×10^8整数)',
    `platform_rate`   BIGINT       NOT NULL DEFAULT 0 COMMENT '平台汇率(×10^8整数)',
    `deviation`       BIGINT       NOT NULL DEFAULT 0 COMMENT '偏差值(×10^4)',
    `threshold`       BIGINT       NOT NULL DEFAULT 0 COMMENT '阈值(×10^4)',
    `is_base`         INT          NOT NULL DEFAULT 0 COMMENT '是否基准币种 0否 1是',
    `status`          INT          NOT NULL DEFAULT 1 COMMENT '状态 0禁用 1启用',
    `create_by`       BIGINT       NOT NULL COMMENT '创建人ID',
    `create_by_name`  VARCHAR(64)  NOT NULL COMMENT '创建人名称',
    `create_at`       BIGINT       NOT NULL COMMENT '创建时间(毫秒时间戳)',
    `update_by`       BIGINT       NOT NULL COMMENT '更新人ID',
    `update_by_name`  VARCHAR(64)  NOT NULL COMMENT '更新人名称',
    `update_at`       BIGINT       NOT NULL COMMENT '更新时间(毫秒时间戳)',
    `delete_at`       BIGINT       NOT NULL DEFAULT 0 COMMENT '删除时间(0=未删除)',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_currency_code` (`currency_code`),
    INDEX `idx_status` (`status`),
    INDEX `idx_currency_type` (`currency_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='币种配置表';
```

### 5.2 user_wallet

```sql
CREATE TABLE `user_wallet` (
    `id`                BIGINT      NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`           BIGINT      NOT NULL COMMENT '用户ID',
    `currency_code`     VARCHAR(16) NOT NULL COMMENT '币种代码',
    `wallet_type`       INT         NOT NULL COMMENT '钱包类型 1中心 2奖励 3主播 4代理 5场馆',
    `available_balance` BIGINT      NOT NULL DEFAULT 0 COMMENT '可用余额(最小单位整数)',
    `frozen_balance`    BIGINT      NOT NULL DEFAULT 0 COMMENT '冻结余额(最小单位整数)',
    `status`            INT         NOT NULL DEFAULT 1 COMMENT '状态 0禁用 1启用',
    `create_at`         BIGINT      NOT NULL COMMENT '创建时间(毫秒时间戳)',
    `update_at`         BIGINT      NOT NULL COMMENT '更新时间(毫秒时间戳)',
    `delete_at`         BIGINT      NOT NULL DEFAULT 0 COMMENT '删除时间(0=未删除)',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_user_currency_type` (`user_id`, `currency_code`, `wallet_type`),
    INDEX `idx_user_id` (`user_id`),
    INDEX `idx_currency_code` (`currency_code`),
    CHECK (`available_balance` >= 0),
    CHECK (`frozen_balance` >= 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='用户钱包表';
```

### 5.3 wallet_flow

```sql
CREATE TABLE `wallet_flow` (
    `id`             BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `flow_no`        BIGINT       NOT NULL COMMENT '流水编号(snowflake)',
    `user_id`        BIGINT       NOT NULL COMMENT '用户ID',
    `currency_code`  VARCHAR(16)  NOT NULL COMMENT '币种代码',
    `wallet_type`    INT          NOT NULL COMMENT '钱包类型',
    `flow_type`      INT          NOT NULL COMMENT '流水类型',
    `amount`         BIGINT       NOT NULL COMMENT '变更金额(正入负出,最小单位)',
    `balance_before` BIGINT       NOT NULL COMMENT '变更前余额',
    `balance_after`  BIGINT       NOT NULL COMMENT '变更后余额',
    `order_no`       VARCHAR(64)  NOT NULL COMMENT '关联业务订单号(幂等键)',
    `remark`         VARCHAR(512) DEFAULT NULL COMMENT '备注说明',
    `extra`          TEXT         DEFAULT NULL COMMENT '扩展信息(JSON)',
    `create_at`      BIGINT       NOT NULL COMMENT '创建时间(毫秒时间戳)',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_flow_no` (`flow_no`),
    UNIQUE INDEX `idx_order_flow_type` (`order_no`, `flow_type`),
    INDEX `idx_user_currency_time` (`user_id`, `currency_code`, `create_at` DESC),
    INDEX `idx_user_flow_type` (`user_id`, `flow_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='钱包流水表';
```

### 5.4 wallet_freeze_record

```sql
CREATE TABLE `wallet_freeze_record` (
    `id`             BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`       VARCHAR(64)  NOT NULL COMMENT '关联业务订单号',
    `user_id`        BIGINT       NOT NULL COMMENT '用户ID',
    `currency_code`  VARCHAR(16)  NOT NULL COMMENT '币种代码',
    `wallet_type`    INT          NOT NULL COMMENT '钱包类型',
    `amount`         BIGINT       NOT NULL COMMENT '冻结金额(最小单位整数)',
    `status`         INT          NOT NULL DEFAULT 1 COMMENT '冻结状态 1冻结中 2已解冻 3已确认',
    `freeze_at`      BIGINT       NOT NULL COMMENT '冻结时间(毫秒时间戳)',
    `finish_at`      BIGINT       DEFAULT NULL COMMENT '完结时间(毫秒时间戳)',
    `finish_remark`  VARCHAR(512) DEFAULT NULL COMMENT '完结备注(驳回原因等)',
    `create_at`      BIGINT       NOT NULL COMMENT '创建时间(毫秒时间戳)',
    `update_at`      BIGINT       NOT NULL COMMENT '更新时间(毫秒时间戳)',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_order_no` (`order_no`),
    INDEX `idx_user_status` (`user_id`, `status`),
    INDEX `idx_user_currency` (`user_id`, `currency_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='钱包冻结记录表';
```

### 5.5 deposit_order

```sql
CREATE TABLE `deposit_order` (
    `id`                BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`          VARCHAR(64)  NOT NULL COMMENT '平台订单号(snowflake)',
    `user_id`           BIGINT       NOT NULL COMMENT '用户ID',
    `currency_code`     VARCHAR(16)  NOT NULL COMMENT '币种代码',
    `order_amount`      BIGINT       NOT NULL COMMENT '订单金额(最小单位整数)',
    `deposit_method`    VARCHAR(32)  NOT NULL COMMENT '充值方式',
    `status`            INT          NOT NULL DEFAULT 1 COMMENT '状态 1待支付 2进行中 3成功 4失败 5失效',
    `channel_order_no`  VARCHAR(128) DEFAULT NULL COMMENT '通道订单号(三方)',
    `rate_snapshot`     BIGINT       DEFAULT NULL COMMENT '入款汇率快照(×10^8,BSB充值时)',
    `pay_currency`      VARCHAR(16)  DEFAULT NULL COMMENT '支付币种(BSB充值时)',
    `pay_amount`        BIGINT       DEFAULT NULL COMMENT '支付金额(最小单位整数)',
    `bonus_activity_id` BIGINT       DEFAULT NULL COMMENT '奖金活动ID',
    `bonus_amount`      BIGINT       DEFAULT NULL COMMENT '奖金金额(最小单位整数)',
    `bonus_turnover`    INT          DEFAULT NULL COMMENT '奖金流水倍数要求',
    `fail_reason`       VARCHAR(256) DEFAULT NULL COMMENT '失败原因',
    `paid_at`           BIGINT       DEFAULT NULL COMMENT '支付完成时间(毫秒时间戳)',
    `expire_at`         BIGINT       DEFAULT NULL COMMENT '订单过期时间(毫秒时间戳)',
    `create_at`         BIGINT       NOT NULL COMMENT '创建时间(毫秒时间戳)',
    `update_at`         BIGINT       NOT NULL COMMENT '更新时间(毫秒时间戳)',
    `delete_at`         BIGINT       NOT NULL DEFAULT 0 COMMENT '删除时间(0=未删除)',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_order_no` (`order_no`),
    INDEX `idx_user_currency_status` (`user_id`, `currency_code`, `status`, `create_at` DESC),
    INDEX `idx_user_time` (`user_id`, `create_at` DESC),
    INDEX `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='充值订单表';
```

### 5.6 withdraw_order

```sql
CREATE TABLE `withdraw_order` (
    `id`                  BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`            VARCHAR(64)  NOT NULL COMMENT '平台订单号(snowflake)',
    `user_id`             BIGINT       NOT NULL COMMENT '用户ID',
    `currency_code`       VARCHAR(16)  NOT NULL COMMENT '币种代码',
    `order_amount`        BIGINT       NOT NULL COMMENT '提现金额(最小单位整数)',
    `withdraw_method`     INT          NOT NULL COMMENT '提现方式 1银行卡 2电子钱包 3USDT',
    `status`              INT          NOT NULL DEFAULT 1 COMMENT '状态 1待风控 2风控驳回 3待财务 4财务驳回 5待出款 6成功 7失败',
    `withdraw_account_id` BIGINT       NOT NULL COMMENT '提现账户ID',
    `rate_snapshot`       BIGINT       DEFAULT NULL COMMENT '出款汇率快照(×10^8,BSB提现时)',
    `pay_currency`        VARCHAR(16)  DEFAULT NULL COMMENT '出款币种(BSB提现时)',
    `pay_amount`          BIGINT       DEFAULT NULL COMMENT '出款金额(最小单位整数)',
    `fee_amount`          BIGINT       NOT NULL DEFAULT 0 COMMENT '手续费(最小单位整数)',
    `fee_rate`            BIGINT       DEFAULT NULL COMMENT '手续费率(×10^4)',
    `freeze_order_no`     VARCHAR(64)  DEFAULT NULL COMMENT '冻结记录订单号',
    `reject_reason`       VARCHAR(512) DEFAULT NULL COMMENT '驳回原因',
    `audit_remark`        VARCHAR(512) DEFAULT NULL COMMENT '审核备注',
    `channel_order_no`    VARCHAR(128) DEFAULT NULL COMMENT '通道订单号',
    `finish_at`           BIGINT       DEFAULT NULL COMMENT '完成时间(毫秒时间戳)',
    `create_at`           BIGINT       NOT NULL COMMENT '创建时间(毫秒时间戳)',
    `update_at`           BIGINT       NOT NULL COMMENT '更新时间(毫秒时间戳)',
    `delete_at`           BIGINT       NOT NULL DEFAULT 0 COMMENT '删除时间(0=未删除)',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_order_no` (`order_no`),
    INDEX `idx_user_currency_status` (`user_id`, `currency_code`, `status`, `create_at` DESC),
    INDEX `idx_user_time` (`user_id`, `create_at` DESC),
    INDEX `idx_status` (`status`),
    INDEX `idx_freeze_order` (`freeze_order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='提现订单表';
```

### 5.7 withdraw_account

```sql
CREATE TABLE `withdraw_account` (
    `id`               BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`          BIGINT       NOT NULL COMMENT '用户ID',
    `currency_code`    VARCHAR(16)  NOT NULL COMMENT '币种代码',
    `withdraw_method`  INT          NOT NULL COMMENT '提现方式 1银行卡 2电子钱包 3USDT',
    `bank_name`        VARCHAR(256) DEFAULT NULL COMMENT '银行名称(加密)',
    `bank_account`     VARCHAR(256) DEFAULT NULL COMMENT '银行账号(加密)',
    `bank_branch`      VARCHAR(256) DEFAULT NULL COMMENT '开户网点',
    `ewallet_type`     VARCHAR(32)  DEFAULT NULL COMMENT '电子钱包类型',
    `phone_number`     VARCHAR(256) DEFAULT NULL COMMENT '手机号(加密)',
    `network_type`     VARCHAR(16)  DEFAULT NULL COMMENT '网络类型(TRC20/ERC20/BEP20)',
    `wallet_address`   VARCHAR(256) DEFAULT NULL COMMENT 'USDT钱包地址',
    `account_holder`   VARCHAR(256) NOT NULL COMMENT '账户持有人(=KYC姓名,加密)',
    `is_default`       INT          NOT NULL DEFAULT 0 COMMENT '是否默认 0否 1是',
    `create_at`        BIGINT       NOT NULL COMMENT '创建时间(毫秒时间戳)',
    `update_at`        BIGINT       NOT NULL COMMENT '更新时间(毫秒时间戳)',
    `delete_at`        BIGINT       NOT NULL DEFAULT 0 COMMENT '删除时间(0=未删除)',
    PRIMARY KEY (`id`),
    INDEX `idx_user_currency_method` (`user_id`, `currency_code`, `withdraw_method`),
    INDEX `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='提现账户表';
```

### 5.8 exchange_record

```sql
CREATE TABLE `exchange_record` (
    `id`                BIGINT      NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `record_no`         VARCHAR(64) NOT NULL COMMENT '兑换记录编号(snowflake)',
    `user_id`           BIGINT      NOT NULL COMMENT '用户ID',
    `source_currency`   VARCHAR(16) NOT NULL COMMENT '源币种(法币)',
    `target_currency`   VARCHAR(16) NOT NULL DEFAULT 'BSB' COMMENT '目标币种(BSB)',
    `source_amount`     BIGINT      NOT NULL COMMENT '兑换金额-法币(最小单位)',
    `target_amount`     BIGINT      NOT NULL COMMENT '兑换获得-BSB(最小单位)',
    `bonus_percent`     INT         NOT NULL DEFAULT 0 COMMENT '赠送比例(10=10%)',
    `bonus_amount`      BIGINT      NOT NULL DEFAULT 0 COMMENT '赠送金额-BSB(最小单位)',
    `total_amount`      BIGINT      NOT NULL COMMENT '实际到账-BSB(最小单位)',
    `rate_snapshot`     BIGINT      NOT NULL COMMENT '使用的平台汇率(×10^8)',
    `turnover_multiple` INT         DEFAULT NULL COMMENT '赠送流水倍数要求',
    `create_at`         BIGINT      NOT NULL COMMENT '创建时间(毫秒时间戳)',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_record_no` (`record_no`),
    INDEX `idx_user_source_time` (`user_id`, `source_currency`, `create_at` DESC),
    INDEX `idx_user_time` (`user_id`, `create_at` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='兑换记录表';
```

### 5.9 exchange_rate_log

```sql
CREATE TABLE `exchange_rate_log` (
    `id`             BIGINT      NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `log_no`         VARCHAR(64) NOT NULL COMMENT '日志编号(snowflake)',
    `update_time`    BIGINT      NOT NULL COMMENT '汇率更新时间(毫秒时间戳)',
    `currency_code`  VARCHAR(16) NOT NULL COMMENT '币种代码',
    `realtime_rate`  BIGINT      NOT NULL COMMENT '实时汇率(×10^8)',
    `platform_rate`  BIGINT      NOT NULL COMMENT '平台汇率(×10^8)',
    `deposit_rate`   BIGINT      NOT NULL COMMENT '入款汇率(×10^8)',
    `withdraw_rate`  BIGINT      NOT NULL COMMENT '出款汇率(×10^8)',
    `float_rate`     BIGINT      NOT NULL COMMENT '浮动比例(×10^4)',
    `deviation`      BIGINT      NOT NULL COMMENT '偏差值(×10^4)',
    `threshold`      BIGINT      NOT NULL COMMENT '阈值(×10^4)',
    `create_at`      BIGINT      NOT NULL COMMENT '创建时间(毫秒时间戳)',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `idx_log_no` (`log_no`),
    INDEX `idx_currency_time` (`currency_code`, `update_time` DESC),
    INDEX `idx_update_time` (`update_time` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='汇率变更日志表';
```

---

## 六、金额精度统一方案总结

这是贯穿所有模型的核心规则，单独整理以确保团队理解一致。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     金额精度统一方案                                      │
│                                                                         │
│  【层次】         【类型】        【说明】                                │
│                                                                         │
│  数据库存储层      BIGINT          最小货币单位的整数值                    │
│  Go 计算层        int64           直接运算，无精度问题                     │
│  RPC 传输层       string          IDL 中金额字段用 string 类型            │
│  前端展示层       string→格式化    按 decimal_digits + symbol 格式化       │
│                                                                         │
│  【币种】    【decimal_digits】 【放大倍数】 【示例】                      │
│                                                                         │
│  VND            0                 1          500000 VND → 存 500000      │
│  IDR            0                 1          8000000 IDR → 存 8000000    │
│  THB            2                 100        1500.50 THB → 存 150050     │
│  BSB            2                 100        1000.00 BSB → 存 100000     │
│  USDT           8                 10^8       100.50 USDT → 存 10050000000│
│                                                                         │
│  【汇率】    统一用 ×10^8 存储                                           │
│                                                                         │
│  1 BSB = 0.1 USDT   → 存 10000000   (0.1 × 10^8)                       │
│  1 BSB = 2620 VND   → 存 262000000000000  (2620 × 10^8)                 │
│  1 USDT = 25300 VND → 存 2530000000000000 (25300 × 10^8)                │
│                                                                         │
│  【百分比】  统一用 ×10^4 存储                                           │
│                                                                         │
│  2.50%  → 存 25000  (2.50 × 10^4)                                       │
│  10%    → 存 100000 (10.00 × 10^4)                                      │
│  0.15%  → 存 1500   (0.15 × 10^4)                                       │
│                                                                         │
│  【IDL 传输约定】                                                        │
│  金额字段在 Thrift IDL 中定义为 string：                                  │
│    1: required string orderAmount      // "150050"                       │
│    2: required string availableBalance // "500000"                       │
│  前端收到 string 后，结合 decimal_digits 做展示格式化。                    │
│  为什么不用 i64：JSON 序列化后 JS Number 精度只有 2^53，                  │
│  VND 大额可能超出。string 最安全。                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 七、模型与产品原型页面的对应关系

产品原型中的每个页面/组件，其数据分别来自哪个模型：

```
┌────────────────────────────────────────────────────────────────────────────┐
│  产品原型页面                           数据来源模型                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  【个人中心/钱包首页】                                                      │
│  原型2-1 个人中心主页                                                       │
│    余额显示 "BSB 1,000,000.00"     ← user_wallet (SUM center + reward)    │
│    币种信息                         ← currency_config (symbol, 格式化规则) │
│                                                                            │
│  原型2-2 币种切换列表                                                       │
│    BSB/IDR/VND/THB 各带余额        ← currency_config (列表)               │
│                                       + user_wallet (每币种余额)           │
│                                                                            │
│  【充值页面】                                                               │
│  原型3-1~3-4 各币种充值中心                                                 │
│    余额                             ← user_wallet                          │
│    充值方式列表 + 档位              ← 财务模块 RPC (GetPaymentConfig)       │
│    汇率展示 (BSB时)                 ← currency_config (deposit_rate)       │
│                                                                            │
│  原型5-2~5-4 BSB选中不同支付方式                                            │
│    汇率 "1BSB=X法币"               ← currency_config (deposit_rate)       │
│                                                                            │
│  原型6-1~6-2 选择奖金                                                       │
│    奖金活动列表                     ← 活动模块 RPC (外部数据)               │
│    奖金角标 "+50%"                  ← 活动模块返回的 bonus_percent          │
│                                                                            │
│  原型7-2 充值进行中                                                         │
│    充值方式/订单号/金额/时间        ← deposit_order                         │
│                                                                            │
│  原型7-3~7-4 重复充值拦截                                                   │
│    同金额待支付订单数               ← deposit_order (条件查询统计)          │
│                                                                            │
│  【兑换页面】                                                               │
│  原型8 兑换BSB                                                              │
│    余额                             ← user_wallet                          │
│    汇率 "1BSB=X法币"               ← currency_config (platform_rate)      │
│    兑换结果                         ← exchange_record (计算后写入)          │
│                                                                            │
│  【提现页面】                                                               │
│  原型9-2 KYC弹窗                                                            │
│    KYC状态                          ← ser-kyc RPC (外部数据)               │
│                                                                            │
│  原型9-4 USDT提现方式                                                       │
│    限额/费率                        ← 财务模块 RPC (外部配置)               │
│                                                                            │
│  原型10-1~10-4 各国提现                                                     │
│    提现账户信息（首次/复用）         ← withdraw_account                     │
│    账户持有人（KYC姓名置灰）        ← ser-user RPC → withdraw_account      │
│    BSB汇率展示                      ← currency_config (withdraw_rate)      │
│                                                                            │
│  原型10-5 提现进行中                                                        │
│    提现方式/订单号/金额/时间        ← withdraw_order                        │
│                                                                            │
│  【记录页面】                                                               │
│  原型11-1 四个Tab列表                                                       │
│    提现Tab                          ← withdraw_order                        │
│    充值Tab                          ← deposit_order                         │
│    兑换Tab                          ← exchange_record                       │
│    奖励Tab                          ← wallet_flow (flow_type IN 奖励类)    │
│                                                                            │
│  原型11-2 四个详情弹窗                                                      │
│    提现详情                         ← withdraw_order                        │
│    充值详情                         ← deposit_order                         │
│    失败原因                         ← deposit_order.fail_reason /           │
│                                       withdraw_order.reject_reason         │
│                                                                            │
│  【B端后台】                                                                │
│  币种配置原型1 币种列表              ← currency_config                      │
│  币种配置原型2 编辑弹窗              ← currency_config (UPDATE)             │
│  币种配置原型3 汇率日志              ← exchange_rate_log                    │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、初始数据与数据导入

### 8.1 币种初始数据

产品需求明确"币种数据由开发导入"，以下是当前版本需要的初始币种：

```
需导入的币种（来自产品需求4.1 + 3.1）：

currency_code | currency_name | currency_type | country | symbol | thousand_sep | decimal_sep | decimal_digits
─────────────|───────────────|───────────────|─────────|────────|──────────────|─────────────|──────────────
BSB           | BSB           | 3 (平台)      | 全球    | BSB    | ,            | .           | 2
VND           | 越南盾         | 1 (法定)      | 越南    | d      | .            | .           | 0
IDR           | 印尼盾         | 1 (法定)      | 印尼    | Rp     | .            | .           | 0
THB           | 泰铢           | 1 (法定)      | 泰国    | B      | ,            | .           | 2
USDT          | USDT          | 2 (加密)      | 全球    | USDT   | ,            | .           | 8

初始状态：全部 status=1 (启用)
基准币种：USDT (is_base=1)，需在系统初始化时设置

产品原型依据：
  钱包需求4.1：当前支持BSB/VND/IDR/THB
  币种配置需求3.2：基准币种推荐USDT
  原型2-2：BSB/IDR/VND/THB 四个币种
  BSB锚定汇率：10BSB = 1USDT → 1BSB = 0.1USDT
```

### 8.2 汇率初始值

```
币种     | 初始平台汇率（相对 USDT）     | realtime_rate 存储值
─────────|─────────────────────────────|─────────────────────
BSB      | 0.10000000 (1BSB=0.1USDT)   | 10000000
VND      | 0.00003820 (1VND=0.0000382U)| 3820
IDR      | 0.00006130 (1IDR=0.0000613U)| 6130
THB      | 0.02850000 (1THB=0.0285U)   | 2850000
USDT     | 1.00000000 (基准)           | 100000000

注：以上为示例初始值，实际值需在上线前从三方汇率API获取。
BSB汇率锚定 USDT，不参与定时刷新。
```

---

## 九、后续步骤

数据模型设计完成后，落地到代码的步骤：

```
步骤1：在 TiDB 中创建 ser_wallet 数据库
  CREATE DATABASE ser_wallet CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

步骤2：执行上述 9 个建表 DDL
  按顺序执行（无物理外键，无顺序依赖）

步骤3：导入币种初始数据
  INSERT INTO currency_config (...) VALUES (...), (...), (...), (...), (...);

步骤4：执行 GORM Gen 代码生成
  运行 ser-wallet/internal/gen/gorm_gen.go
  生成 gorm_gen/model/ + gorm_gen/query/ + gorm_gen/repo/ （18个CRUD方法）

步骤5：在 internal/enum/ 下定义所有枚举常量
  wallet_type.go / currency_type.go / flow_type.go / order_status.go / freeze_status.go

步骤6：在 internal/rep/ 下封装自定义查询
  在 GORM Gen 生成的 repo 基础上，编写条件更新、聚合查询等自定义方法

步骤7：在 internal/ser/ 下实现业务逻辑
  使用 model 结构体和 repo 方法组合业务操作
```

---

## 十、表汇总与数据量预估

```
┌──────────────────────┬──────────┬────────────────────────────────────────────┐
│ 表名                  │ 数据增长  │ 说明                                       │
├──────────────────────┼──────────┼────────────────────────────────────────────┤
│ currency_config       │ 极低     │ 只有几个到十几个币种，几乎不增长             │
│ user_wallet           │ 低       │ 用户数 × 币种数 × 钱包类型数，按需创建       │
│ wallet_flow           │ 高       │ 每次余额变更一条，是增长最快的表             │
│ wallet_freeze_record  │ 中       │ 每次提现一条，量级 = 提现订单量               │
│ deposit_order         │ 中       │ 每次充值一条                                │
│ withdraw_order        │ 中       │ 每次提现一条                                │
│ withdraw_account      │ 低       │ 用户数 × 提现方式数，首次创建后复用           │
│ exchange_record       │ 中       │ 每次兑换一条                                │
│ exchange_rate_log     │ 中       │ 定时任务触发，频率取决于汇率波动              │
└──────────────────────┴──────────┴────────────────────────────────────────────┘

wallet_flow 是唯一需要关注数据量的表：
  如果日活 10 万用户，平均每人每天 5 次余额变更 → 日新增 50 万条
  月新增 1500 万条，年新增 1.8 亿条
  TiDB 可以透明处理这个量级（自动分片），当前阶段无需额外分表处理。
  如果未来需要优化，可以按 create_at 做时间范围分区。
```
