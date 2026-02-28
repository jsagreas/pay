# ser-wallet 数据模型设计文档

> 编写时间：2026-02-26
> 定位：将产品需求映射为数据模型，为后续建表、GORM Gen 代码生成、业务开发提供基础
> 工程规范基准：基于现有工程（ser-app/ser-item/ser-buser/ser-s3 等）的实际代码分析
> 依据来源：
> - 需求分析：/Users/mac/gitlab/z-readme/result/tmp-2.md（152张产品原型+需求文档）
> - 需求理解：/Users/mac/gitlab/z-readme/需求理解/tmp-2.md
> - 架构设计：/Users/mac/gitlab/z-readme/架构设计/tmp-2.md
> - 实现思路：/Users/mac/gitlab/z-readme/实现思路/tmp-2.md
> - 流程思路：/Users/mac/gitlab/z-readme/流程思路/tmp-2.md
> - 现有工程模型分析：ser-app/ser-item/ser-buser/ser-s3 的 SQL + GORM Gen Model

---

## 一、从产品到数据模型的转换思路

### 1.1 为什么需要这一步

产品原型和需求文档描述的是"现实世界"——用户能看到什么、能做什么、流程怎么走。但代码世界需要的是"数据结构"——怎么存、字段是什么、关系怎么表达。数据模型就是连接这两个世界的桥梁。

**转换的核心逻辑**：

```
产品原型中的"页面"        → 数据模型中的"表"
页面上的"字段/信息"       → 表中的"列"
页面上的"状态流转"        → 列中的"状态枚举"
功能之间的"关联关系"      → 表之间的"逻辑外键"
业务规则中的"唯一性约束"   → 索引中的"唯一索引"
非功能性需求中的"安全要求" → 设计中的"幂等索引/审计字段/不可变约束"
```

### 1.2 数据模型分类总览

根据需求分析，ser-wallet 的数据模型按业务域分为 **4 大类 11 张表**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ser-wallet 数据模型全景                             │
│                                                                     │
│  A. 币种配置域（基础设施层，最先建立）                                  │
│     ① currency_config       — 币种配置主表                           │
│     ② exchange_rate_log     — 汇率变更日志表                         │
│                                                                     │
│  B. 钱包核心域（资金安全核心，依赖 A）                                  │
│     ③ wallet_account        — 钱包账户表（物化余额）                   │
│     ④ wallet_transaction    — 账变流水表（不可变账本）                  │
│                                                                     │
│  C. 业务订单域（业务流程载体，依赖 A + B）                              │
│     ⑤ deposit_order         — 充值订单表                             │
│     ⑥ withdraw_order        — 提现订单表                             │
│     ⑦ exchange_order        — 兑换订单表                             │
│     ⑧ withdraw_account      — 提现账户表                             │
│                                                                     │
│  D. 辅助域（附加业务能力）                                             │
│     ⑨ audit_requirement     — 稽核流水要求表                         │
│     ⑩ deposit_duplicate_log — 重复转账检测记录表                      │
│     ⑪ recharge_route_record — 充值路由追踪表（提现渠道路由依据）        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 产品需求 → 数据模型的映射关系

| 产品需求来源 | 映射到的数据模型 | 映射依据 |
|------------|----------------|---------|
| 需求 4.2-4.5 币种配置页 20 个字段 | ① currency_config | 后台配置页的每个字段 = 表中一列 |
| 需求 4.6 汇率日志 Tab | ② exchange_rate_log | 日志页的每个字段 = 表中一列 |
| 需求 4.1 钱包结构（币种×类型） | ③ wallet_account | 每个(user, currency, walletType) = 一行 |
| 需求 3.10 对账一致性 + 流水追踪 | ④ wallet_transaction | 每笔余额变动 = 一条不可变流水 |
| 需求 3.4 充值模块 + 5.4 充值记录 | ⑤ deposit_order | 每笔充值 = 一条订单记录 |
| 需求 3.8 提现模块 + 5.5 提现记录 | ⑥ withdraw_order | 每笔提现 = 一条订单记录（含7状态流转） |
| 需求 3.7 兑换模块 | ⑦ exchange_order | 每笔兑换 = 一条订单记录 |
| 需求 3.8.3-3.8.4 提现账户 | ⑧ withdraw_account | 每个用户的银行卡/电子钱包信息 |
| 需求 3.6 奖金流水要求 + 7.5 稽核 | ⑨ audit_requirement | 每笔带稽核的操作 = 一条稽核记录 |
| 需求 3.5 重复转账拦截规则 | ⑩ deposit_duplicate_log | 待支付订单计数的辅助表 |
| 需求 3.8.2 提现渠道路由规则 | ⑪ recharge_route_record | "怎么充的怎么提"的历史依据 |

---

## 二、工程规范对齐（建表前必须清楚）

在设计具体表结构之前，必须对齐现有工程的数据模型规范。以下所有规范均来自对 ser-app/ser-item/ser-buser/ser-s3 实际代码的分析。

### 2.1 DDL 规范（从现有 SQL 文件提炼）

| 规范项 | 现有工程惯例 | 来源依据 |
|--------|------------|---------|
| 主键类型 | `bigint unsigned NOT NULL AUTO_INCREMENT` | ser-item/sql/1.table.sql:7 |
| 表名风格 | snake_case，可带模块前缀（如 `sys_user`、`s3_documents`） | 全工程统一 |
| 列名风格 | snake_case | 全工程统一 |
| 字符集 | `utf8mb4`，排序规则 `utf8mb4_bin` | ser-item/sql/1.table.sql:23 |
| 引擎 | `InnoDB` | 全工程统一（支持事务和行锁） |
| 时间字段 | `bigint unsigned NOT NULL DEFAULT '0'`，毫秒级时间戳 | ser-app Banner 用毫秒，ser-item 用秒 |
| 软删除字段 | `delete_at bigint unsigned NOT NULL DEFAULT '0'` | ser-item/sql/1.table.sql:43 |
| 审计字段 | `create_by / update_by / create_by_name / update_by_name` | ser-app Banner |
| 字符串默认值 | `DEFAULT ''` | ser-item/sql/1.table.sql:8 |
| 注释 | 每列必须有 `COMMENT` | 全工程统一 |
| 唯一索引前缀 | `uk_` | ser-item `uk_item_code`、`uk_item_scene` |
| 普通索引前缀 | `idx_` | ser-item `idx_item_name`、`idx_delete_at` |

### 2.2 GORM Gen Model 规范（从现有 .gen.go 文件提炼）

| 规范项 | 现有工程惯例 | 来源依据 |
|--------|------------|---------|
| 主键 Go 类型 | `int64` | ser-app Banner.ID（Gen 生成时 bigint unsigned → int64） |
| 非空字段 | 值类型（`int64`、`int32`、`string`） | Banner.BannerName string |
| 可空字段 | 指针类型（`*string`、`*int64`、`*int32`） | Banner.JumpTarget *string |
| GORM Tag 格式 | `gorm:"column:xxx;not null;comment:xxx"` | 全工程统一 |
| JSON Tag | `json:"xxx"` 与列名一致 | 全工程统一 |
| 索引定义 | **不在 Model 中定义**（`FieldWithIndexTag: false`） | common/pkg/db/cmd/generate.go:71 |
| 时间自动填充 | GORM Plugin（`common/pkg/db/mysql/user_plugin.go`）自动注入 | Plugin Before Create/Update |
| TableName 方法 | 自动生成 `func (*Model) TableName() string` | 全工程统一 |

### 2.3 ser-wallet 特殊约定（与现有工程的差异点）

| 差异项 | 现有工程 | ser-wallet | 原因 |
|--------|---------|-----------|------|
| 时间精度 | 混合（毫秒/秒） | **统一毫秒** | 金融场景需要更高精度，且新建表应统一 |
| 金额类型 | 无金额字段 or `int unsigned` | **`decimal(20,4)`** | 金融场景必须用 DECIMAL 避免精度丢失（架构设计 ADR-05） |
| 软删除 | 部分表有 | **视表而定** | 流水表不可删除；账户表/订单表需软删除；配置表需软删除 |
| 审计字段 | B端表有 `create_by_name` | B端管理的表有，C端操作的表仅需 `user_id` | C端操作的主体是用户自己，不需要 `create_by_name` |

### 2.4 金额存储精度说明

**架构设计 ADR-05 决策：使用 DECIMAL，不用 BIGINT**

```
为什么用 DECIMAL(20,4)：
  · 法定货币（VND/IDR 整数，THB 2位小数） → 4位覆盖
  · 平台货币（BSB 2位小数）               → 4位覆盖
  · 汇率相关计算中间结果                   → 4位满足精度需求
  · DECIMAL 是金融系统标准做法（Stripe、Modern Treasury、支付宝均用 NUMERIC/DECIMAL）
  · TiDB 完全兼容 MySQL 的 DECIMAL 运算，无精度丢失

为什么是 (20,4) 而不是 (20,8)：
  · 钱包存的是业务金额，不是加密货币原始精度
  · 加密货币（USDT）在钱包中以"业务币种金额"体现（如 BSB 数量），不直接存 8 位
  · 汇率字段单独使用 (20,8) 覆盖加密货币的 8 位精度
  · 减少不必要的存储空间

特殊字段：
  · 汇率相关字段 → DECIMAL(20,8)（加密货币精度需要 8 位小数）
  · 百分比字段（浮动%/阈值%） → DECIMAL(10,4)（足够表示 xx.xx%）
```

---

## 三、A 类 — 币种配置域数据模型

### 表 ① currency_config（币种配置主表）

#### 产品需求映射

```
产品原型来源：币种配置后台页 — 列表 20 个字段 + 编辑弹窗 6 个字段
需求文档来源：需求 4.2（币种类型）、4.3（基准币种）、4.4（配置页字段）、4.5（汇率规则）
```

| 产品字段（中文） | 数据库列名 | 类型 | 说明 |
|----------------|----------|------|------|
| 序号 | id | 主键自增 | 序号即 ID |
| 币种代码 | currency_code | varchar(10) | 唯一，如 BSB/VND/IDR/THB/USDT |
| 币种名称 | currency_name | varchar(50) | 如 "越南盾" |
| 币种类型 | currency_type | tinyint | 1=法定货币 2=加密货币 3=平台货币 |
| 币种国家 | country | varchar(50) | 数据字典值，默认"全球" |
| 货币符号 | currency_symbol | varchar(10) | 如 d / Rp / ฿ |
| 千分位符号 | thousands_sep | varchar(5) | `,` 或 `.` |
| 小数点符号 | decimal_sep | varchar(5) | `.` 或 `,` |
| 小数位数 | decimal_digits | tinyint | 法币显示精度(0/2)，加密8位 |
| 实时汇率 | realtime_rate | decimal(20,8) | 三方API平均值 |
| 平台汇率 | platform_rate | decimal(20,8) | 偏差达阈值时更新 |
| 汇率浮动% | rate_float_pct | decimal(10,4) | 如 1.4000 表示 1.4% |
| 入款汇率 | deposit_rate | decimal(20,8) | 平台汇率×(1+浮动%) |
| 出款汇率 | withdraw_rate | decimal(20,8) | 平台汇率×(1-浮动%) |
| 偏差值 | deviation_pct | decimal(10,4) | 当前|实时-平台|/实时×100% |
| 阈值% | threshold_pct | decimal(10,4) | 偏差>=此值触发更新 |
| 币种图标 | icon_url | varchar(255) | S3 URL (SVG/PNG <=5KB) |
| 排序 | sort_value | int | C端显示顺序，越小越靠前 |
| 状态 | status | tinyint | 0=禁用 1=启用 |
| 是否基准币种 | is_base | tinyint | 0=否 1=是（全局仅一个=1） |
| 是否系统内置 | is_builtin | tinyint | 0=否 1=是（BSB不可配置） |
| 操作人ID | update_by | bigint unsigned | 最后修改者ID |
| 操作人名称 | update_by_name | varchar(50) | 最后修改者姓名 |
| 创建人ID | create_by | bigint unsigned | — |
| 创建人名称 | create_by_name | varchar(50) | — |
| 创建时间 | create_at | bigint unsigned | 毫秒时间戳 |
| 更新时间 | update_at | bigint unsigned | 毫秒时间戳 |
| 删除时间 | delete_at | bigint unsigned | 0=未删除 |

#### SQL DDL

```sql
CREATE TABLE `currency_config` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码（唯一），如 BSB/VND/IDR/THB/USDT',
    `currency_name`   varchar(50)      NOT NULL DEFAULT ''     COMMENT '币种名称，如 越南盾',
    `currency_type`   tinyint unsigned NOT NULL                COMMENT '币种类型：1法定货币 2加密货币 3平台货币',
    `country`         varchar(50)      NOT NULL DEFAULT '全球'  COMMENT '币种国家/地区，数据字典值',
    `currency_symbol` varchar(10)      NOT NULL DEFAULT ''     COMMENT '货币符号，如 d / Rp / ฿ / $ ',
    `thousands_sep`   varchar(5)       NOT NULL DEFAULT ','    COMMENT '千分位分隔符',
    `decimal_sep`     varchar(5)       NOT NULL DEFAULT '.'    COMMENT '小数点分隔符',
    `decimal_digits`  tinyint unsigned NOT NULL DEFAULT 2      COMMENT '显示小数位数：法币0或2，加密8',
    `realtime_rate`   decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '实时汇率（三方API平均值，对基准币种）',
    `platform_rate`   decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '平台汇率（偏差>=阈值时从实时汇率同步）',
    `rate_float_pct`  decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '汇率浮动百分比，如1.4000表示1.4%',
    `deposit_rate`    decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '入款汇率 = 平台汇率 × (1 + 浮动%)',
    `withdraw_rate`   decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '出款汇率 = 平台汇率 × (1 - 浮动%)',
    `deviation_pct`   decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '当前偏差百分比 = |实时-平台|/实时×100',
    `threshold_pct`   decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '阈值百分比，偏差>=此值触发平台汇率更新',
    `icon_url`        varchar(255)     NOT NULL DEFAULT ''     COMMENT '币种图标URL（SVG/PNG，<=5KB）',
    `sort_value`      int unsigned     NOT NULL DEFAULT 1000   COMMENT 'C端展示排序值，越小越靠前',
    `status`          tinyint unsigned NOT NULL DEFAULT 0      COMMENT '状态：0禁用 1启用',
    `is_base`         tinyint unsigned NOT NULL DEFAULT 0      COMMENT '是否基准币种：0否 1是（全局仅一条=1，设后不可改）',
    `is_builtin`      tinyint unsigned NOT NULL DEFAULT 0      COMMENT '是否系统内置：0否 1是（BSB=1，不可配置汇率/状态）',
    `create_by`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建人ID',
    `create_by_name`  varchar(50)      NOT NULL DEFAULT ''     COMMENT '创建人名称',
    `update_by`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '最后操作人ID',
    `update_by_name`  varchar(50)      NOT NULL DEFAULT ''     COMMENT '最后操作人名称',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    `delete_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '删除时间（毫秒时间戳，0=未删除）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_currency_code` (`currency_code`),
    KEY `idx_status` (`status`),
    KEY `idx_delete_at` (`delete_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='币种配置表';
```

#### GORM Model（预期 Gen 生成结果）

```go
const TableNameCurrencyConfig = "currency_config"

type CurrencyConfig struct {
    ID            int64    `gorm:"column:id;primaryKey;autoIncrement:true;comment:主键ID" json:"id"`
    CurrencyCode  string   `gorm:"column:currency_code;not null;comment:币种代码" json:"currency_code"`
    CurrencyName  string   `gorm:"column:currency_name;not null;comment:币种名称" json:"currency_name"`
    CurrencyType  int32    `gorm:"column:currency_type;not null;comment:币种类型：1法定 2加密 3平台" json:"currency_type"`
    Country       string   `gorm:"column:country;not null;default:全球;comment:币种国家" json:"country"`
    CurrencySymbol string  `gorm:"column:currency_symbol;not null;comment:货币符号" json:"currency_symbol"`
    ThousandsSep  string   `gorm:"column:thousands_sep;not null;default:,;comment:千分位符号" json:"thousands_sep"`
    DecimalSep    string   `gorm:"column:decimal_sep;not null;default:.;comment:小数点符号" json:"decimal_sep"`
    DecimalDigits int32    `gorm:"column:decimal_digits;not null;default:2;comment:显示小数位数" json:"decimal_digits"`
    RealtimeRate  float64  `gorm:"column:realtime_rate;not null;default:0;comment:实时汇率" json:"realtime_rate"`
    PlatformRate  float64  `gorm:"column:platform_rate;not null;default:0;comment:平台汇率" json:"platform_rate"`
    RateFloatPct  float64  `gorm:"column:rate_float_pct;not null;default:0;comment:汇率浮动%" json:"rate_float_pct"`
    DepositRate   float64  `gorm:"column:deposit_rate;not null;default:0;comment:入款汇率" json:"deposit_rate"`
    WithdrawRate  float64  `gorm:"column:withdraw_rate;not null;default:0;comment:出款汇率" json:"withdraw_rate"`
    DeviationPct  float64  `gorm:"column:deviation_pct;not null;default:0;comment:当前偏差%" json:"deviation_pct"`
    ThresholdPct  float64  `gorm:"column:threshold_pct;not null;default:0;comment:阈值%" json:"threshold_pct"`
    IconURL       string   `gorm:"column:icon_url;not null;comment:币种图标URL" json:"icon_url"`
    SortValue     int32    `gorm:"column:sort_value;not null;default:1000;comment:排序值" json:"sort_value"`
    Status        int32    `gorm:"column:status;not null;default:0;comment:状态 0禁用 1启用" json:"status"`
    IsBase        int32    `gorm:"column:is_base;not null;default:0;comment:是否基准币种 0否 1是" json:"is_base"`
    IsBuiltin     int32    `gorm:"column:is_builtin;not null;default:0;comment:是否系统内置 0否 1是" json:"is_builtin"`
    CreateBy      int64    `gorm:"column:create_by;not null;comment:创建人ID" json:"create_by"`
    CreateByName  string   `gorm:"column:create_by_name;not null;comment:创建人名称" json:"create_by_name"`
    UpdateBy      int64    `gorm:"column:update_by;not null;comment:更新人ID" json:"update_by"`
    UpdateByName  string   `gorm:"column:update_by_name;not null;comment:更新人名称" json:"update_by_name"`
    CreateAt      int64    `gorm:"column:create_at;not null;comment:创建时间（毫秒时间戳）" json:"create_at"`
    UpdateAt      int64    `gorm:"column:update_at;not null;comment:更新时间（毫秒时间戳）" json:"update_at"`
    DeleteAt      int64    `gorm:"column:delete_at;not null;comment:删除时间（毫秒时间戳，0=未删除）" json:"delete_at"`
}
```

**注意**：GORM Gen 对 `decimal` 类型默认映射为 `float64`。在业务代码中需使用 `shopspring/decimal` 库做精确计算，Model 字段类型可在生成后根据需要调整或在业务层转换。

#### 设计说明

```
1. is_base 字段：
   · 全局只能有一条 is_base=1 的记录
   · 代码层面保证：先查是否已有 is_base=1，有则拒绝
   · 不做数据库唯一约束（因为可以有多条 is_base=0）
   · 设置后不可更改（代码层面保证，不是数据库约束）

2. is_builtin 字段：
   · BSB 的 is_builtin=1，其汇率/代码/类型/状态不可修改
   · 编辑接口对 is_builtin=1 的记录做特殊校验

3. 基准币种为什么不单独建表：
   · 之前的实现思路提到了 base_currency_setting 表的可能性
   · 实际上用 is_base 字段即可满足需求（只有一条记录标记为1）
   · 减少一张表和一次 JOIN，更简洁
   · "一次性设置"靠代码保证，不需要额外表结构

4. 汇率精度为什么用 (20,8)：
   · 加密货币（USDT/BTC等）行业惯例保留 8 位小数
   · 法定货币只需 2 位，但统一存储格式简化逻辑
   · 显示时按 decimal_digits 字段做截断/四舍五入
```

---

### 表 ② exchange_rate_log（汇率变更日志表）

#### 产品需求映射

```
产品原型来源：操作日志页 → 汇率日志 Tab
需求文档来源：需求 4.6（汇率日志）
触发时机：任何币种的平台汇率因偏差超阈值而更新时，自动生成一条日志
```

| 产品字段 | 数据库列名 | 类型 | 说明 |
|---------|----------|------|------|
| 序号 | id | 主键自增 | — |
| 日志编号 | log_no | varchar(30) | 唯一编号，可由时间+序列生成 |
| 更新时间 | update_time | bigint unsigned | 汇率更新发生的时间 |
| 币种代码 | currency_code | varchar(10) | 关联 currency_config |
| 实时汇率 | realtime_rate | decimal(20,8) | 触发更新时的实时汇率 |
| 平台汇率 | platform_rate | decimal(20,8) | 更新后的平台汇率 |
| 入款汇率 | deposit_rate | decimal(20,8) | 更新后的入款汇率 |
| 出款汇率 | withdraw_rate | decimal(20,8) | 更新后的出款汇率 |
| 浮动% | rate_float_pct | decimal(10,4) | 当时的浮动配置值 |
| 偏差% | deviation_pct | decimal(10,4) | 触发更新的偏差值 |
| 阈值% | threshold_pct | decimal(10,4) | 当时的阈值配置值 |

#### SQL DDL

```sql
CREATE TABLE `exchange_rate_log` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `log_no`          varchar(30)      NOT NULL                COMMENT '日志编号（唯一）',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码',
    `realtime_rate`   decimal(20,8)    NOT NULL                COMMENT '触发时的实时汇率',
    `platform_rate`   decimal(20,8)    NOT NULL                COMMENT '更新后的平台汇率',
    `deposit_rate`    decimal(20,8)    NOT NULL                COMMENT '更新后的入款汇率',
    `withdraw_rate`   decimal(20,8)    NOT NULL                COMMENT '更新后的出款汇率',
    `rate_float_pct`  decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '当时的汇率浮动%',
    `deviation_pct`   decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '触发更新的偏差%',
    `threshold_pct`   decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '当时的阈值%',
    `update_time`     bigint unsigned  NOT NULL                COMMENT '汇率更新时间（毫秒时间戳）',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '记录创建时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_log_no` (`log_no`),
    KEY `idx_currency_time` (`currency_code`, `update_time`),
    KEY `idx_update_time` (`update_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='汇率变更日志表';
```

#### 设计说明

```
1. 此表为审计日志，只有 INSERT，不做 UPDATE/DELETE
2. 无 delete_at 字段（日志不可删除）
3. 无 update_at 字段（日志不可修改）
4. 无 create_by 字段（由系统定时任务自动生成，无人工操作）
5. 筛选需求：按 currency_code + update_time 范围查询 → 组合索引
```

---

## 四、B 类 — 钱包核心域数据模型

### 表 ③ wallet_account（钱包账户表）

#### 产品需求映射

```
产品原型来源：钱包首页余额展示、消费优先级（中心+奖励）、冻结机制
需求文档来源：需求 4.1（钱包结构 5 种类型）、3.3（币种隔离）、3.8（冻结/解冻）
架构设计来源：账户模型（物化余额）、并发控制（SELECT FOR UPDATE）
```

**核心模型思想**：一个"账户" = `(user_id, currency_code, wallet_type)` 的唯一组合。

| 产品概念 | 数据库列名 | 类型 | 说明 |
|---------|----------|------|------|
| 用户 | user_id | bigint unsigned | 用户ID |
| 币种 | currency_code | varchar(10) | 币种隔离维度 |
| 钱包类型 | wallet_type | tinyint unsigned | 1=中心 2=奖励 3=主播 4=代理 5=场馆 |
| 可用余额 | available_amt | decimal(20,4) | 用户可以花的钱 |
| 冻结余额 | frozen_amt | decimal(20,4) | 提现等操作锁定的钱 |
| 版本号 | version | bigint unsigned | 乐观锁备用（主用悲观锁） |

#### SQL DDL

```sql
CREATE TABLE `wallet_account` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`         bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码：BSB/VND/IDR/THB/USDT',
    `wallet_type`     tinyint unsigned NOT NULL                COMMENT '钱包类型：1中心 2奖励 3主播 4代理 5场馆',
    `available_amt`   decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '可用余额',
    `frozen_amt`      decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '冻结余额',
    `version`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '乐观锁版本号（主方案为悲观锁，此字段备用）',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_currency_type` (`user_id`, `currency_code`, `wallet_type`),
    KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='钱包账户表';
```

#### 设计说明

```
1. 无 total_amt 字段：
   · 总余额 = available_amt + frozen_amt，代码层计算
   · 减少一个冗余字段 = 减少一个不一致风险点
   · 架构设计 ADR 明确决策

2. 无 delete_at 字段：
   · 钱包账户一旦创建不可删除
   · 即使用户注销，账户也应保留（审计需要）
   · 如果需要"冻结整个账户"能力，后续加 account_status 字段

3. 无 create_by / update_by 字段：
   · 钱包账户是C端用户操作产生的
   · "谁创建的"就是 user_id 自己
   · "谁修改的"由关联的 wallet_transaction 流水追溯

4. 账户创建时机 — 懒创建：
   · 不在用户注册时批量创建所有币种×所有类型的账户
   · 在首次对该 (user_id, currency_code, wallet_type) 执行操作时创建
   · 减少无效数据（多数用户可能只用 1-2 个币种）
   · 创建逻辑：先查是否存在 → 不存在则 INSERT → 注意并发下的 duplicate 处理

5. 唯一索引 uk_user_currency_type 的作用：
   · 保证同一用户同一币种同一钱包类型只有一个账户
   · 同时作为 SELECT ... FOR UPDATE 的精确锁定依据
   · 并发安全：INSERT 时如遇唯一冲突不会报错（懒创建场景处理）

6. DECIMAL(20,4) 精度：
   · 最大值：9999999999999999.9999（16位整数 + 4位小数）
   · 足够覆盖越南盾（百万级数值，无小数）和泰铢（万级，2位小数）
   · BSB 2位小数，USDT 如果按业务金额存也是 2-4 位足够
```

---

### 表 ④ wallet_transaction（账变流水表 — 不可变账本）

#### 产品需求映射

```
产品原型来源：账变记录 4 个 Tab（充值/提现/兑换/奖励）+ 详情页
需求文档来源：需求 3.9（账变记录）、3.10（对账一致性 + 幂等性）
架构设计来源：不可变流水（只 INSERT）、before/after 审计链、幂等唯一索引
```

**核心模型思想**：每一笔余额变动都产生一条流水记录，流水只能追加不能修改删除。

| 产品概念 | 数据库列名 | 类型 | 说明 |
|---------|----------|------|------|
| 流水ID | id | bigint | Snowflake ID（不自增） |
| 用户 | user_id | bigint unsigned | 用户ID |
| 币种 | currency_code | varchar(10) | 与 wallet_account 对应 |
| 钱包类型 | wallet_type | tinyint unsigned | 1-5 |
| 变动类型 | tx_type | varchar(30) | deposit/withdraw/exchange/reward/bet/... |
| 收支方向 | direction | tinyint unsigned | 1=收入(+) 2=支出(-) |
| 变动金额 | amount | decimal(20,4) | 绝对值，永远 > 0 |
| 变动前余额 | before_amt | decimal(20,4) | 操作前的 available_amt |
| 变动后余额 | after_amt | decimal(20,4) | 操作后的 available_amt |
| 冻结变动 | frozen_change | decimal(20,4) | 正=冻结增加 负=冻结减少 0=无冻结变动 |
| 关联订单号 | order_no | varchar(50) | C+16/T+16/B+16/A+16/M+16 等 |
| 操作类型 | op_type | varchar(30) | credit/debit/freeze/unfreeze/deduct_frozen |
| 备注 | remark | varchar(255) | 可选说明信息 |
| 创建时间 | create_at | bigint unsigned | 毫秒时间戳 |

#### SQL DDL

```sql
CREATE TABLE `wallet_transaction` (
    `id`              bigint unsigned  NOT NULL                COMMENT '流水ID（Snowflake，非自增）',
    `user_id`         bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码',
    `wallet_type`     tinyint unsigned NOT NULL                COMMENT '钱包类型：1中心 2奖励 3主播 4代理 5场馆',
    `tx_type`         varchar(30)      NOT NULL                COMMENT '变动类型：deposit/withdraw/exchange/reward/bet/manual_credit/manual_debit/补单',
    `direction`       tinyint unsigned NOT NULL                COMMENT '收支方向：1收入(+) 2支出(-)',
    `amount`          decimal(20,4)    NOT NULL                COMMENT '变动金额（绝对值，>0）',
    `before_amt`      decimal(20,4)    NOT NULL                COMMENT '变动前可用余额',
    `after_amt`       decimal(20,4)    NOT NULL                COMMENT '变动后可用余额',
    `frozen_change`   decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '冻结变动（正=冻结增加，负=冻结减少，0=无冻结变动）',
    `order_no`        varchar(50)      NOT NULL                COMMENT '关联业务订单号',
    `op_type`         varchar(30)      NOT NULL                COMMENT '操作类型：credit/debit/freeze/unfreeze/deduct_frozen',
    `remark`          varchar(255)     NOT NULL DEFAULT ''     COMMENT '备注',
    `create_at`       bigint unsigned  NOT NULL                COMMENT '创建时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_op_wallet` (`order_no`, `op_type`, `wallet_type`),
    KEY `idx_user_currency_type_time` (`user_id`, `currency_code`, `tx_type`, `create_at`),
    KEY `idx_user_currency_time` (`user_id`, `currency_code`, `create_at`),
    KEY `idx_order_no` (`order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='账变流水表（不可变账本，只INSERT）';
```

#### 设计说明

```
1. 主键不用 AUTO_INCREMENT 而用 Snowflake ID：
   · 流水表数据量大，后续可能分库分表
   · Snowflake ID 有时间维度，天然有序，分表友好
   · 避免自增ID在分布式环境下的冲突
   · 如果项目内暂无 Snowflake 生成器，也可先用 AUTO_INCREMENT，后续迁移

2. 不可变约束（非数据库层面，代码层面强制）：
   · 此表永远不执行 UPDATE 和 DELETE
   · 余额修正通过"反向流水"实现（如修正多加了10，就再写一条扣10的流水）
   · GORM Gen 生成的 repo 中的 Update/Delete 方法不应被调用
   · 可在 Code Review 或静态检查中强制此规则

3. 无 update_at / delete_at 字段：
   · 不可变 = 不更新 = 不删除
   · 这些字段的缺失本身就是"不可变"的信号

4. 幂等唯一索引 uk_order_op_wallet：
   · 三元组组合 (order_no, op_type, wallet_type)
   · 为什么不只用 order_no：
     人工加款（A+16位订单号）可能同时操作中心+奖励+代理+主播 4 个钱包
     同一 order_no 下有 4 条流水，wallet_type 不同
   · 为什么不只用 (order_no, wallet_type)：
     同一订单可能有 freeze + deduct_frozen 两步，op_type 不同
   · 这是 Redis 幂等键的数据库层兜底

5. 索引设计：
   · idx_user_currency_type_time：C端账变记录查询（按币种+类型+时间）
   · idx_user_currency_time：C端钱包总流水查询（不区分类型）
   · idx_order_no：通过订单号反查流水
   · 都是实际业务查询场景驱动的索引

6. before_amt / after_amt 的审计价值：
   · 第 N 条流水的 after_amt == 第 N+1 条的 before_amt（连续性校验）
   · 任何一条流水都能回答"操作前多少钱？操作后多少钱？"
   · 对账脚本可据此逐条验证流水链完整性
```

---

## 五、C 类 — 业务订单域数据模型

### 表 ⑤ deposit_order（充值订单表）

#### 产品需求映射

```
产品原型来源：C端充值页面、B端充值记录列表（24个字段）
需求文档来源：需求 3.4（充值模块）、5.4（充值记录 24 字段）
```

| 产品字段 | 数据库列名 | 类型 | 说明 |
|---------|----------|------|------|
| 平台订单号 | order_no | varchar(20) | C+16位随机数 |
| 通道订单号 | channel_order_no | varchar(50) | 财务模块回传（可能为空） |
| 用户编号 | user_id | bigint unsigned | — |
| 订单币种 | currency_code | varchar(10) | — |
| 钱包类型 | wallet_type | tinyint unsigned | 默认中心钱包 |
| 充值方式 | pay_method | varchar(50) | USDT_TRC20/BANK/EWALLET 等 |
| 订单金额 | order_amt | decimal(20,4) | 用户输入的充值金额 |
| 入款汇率快照 | rate_snapshot | decimal(20,8) | 创建订单时锁定的入款汇率 |
| 支付币种 | pay_currency | varchar(10) | 实际支付的币种（如USDT） |
| 支付金额 | pay_amt | decimal(20,4) | 按汇率换算后的支付金额 |
| 活动类型/ID | activity_id | bigint unsigned | 关联奖金活动（0=无活动） |
| 活动优惠金额 | bonus_amt | decimal(20,4) | 奖金活动赠送金额 |
| 稽核倍数 | audit_multiplier | decimal(10,2) | 奖金需完成的流水倍数 |
| 到账总金额 | total_credit_amt | decimal(20,4) | 订单金额 + 活动优惠 |
| 手续费 | fee_amt | decimal(20,4) | 充值手续费 |
| 费率 | fee_rate | decimal(10,4) | 充值费率% |
| 订单状态 | status | tinyint unsigned | 1=待支付 2=支付成功 3=支付失败 4=超时关闭 |
| 基准币种订单金额 | base_order_amt | decimal(20,4) | 换算为基准币种的金额 |
| 基准币种活动优惠 | base_bonus_amt | decimal(20,4) | 换算为基准币种的优惠 |
| 基准币种手续费 | base_fee_amt | decimal(20,4) | 换算为基准币种的手续费 |
| 平台汇率快照 | platform_rate_snapshot | decimal(20,8) | 用于基准币种换算 |
| 完成时间 | complete_at | bigint unsigned | 支付完成时间 |
| 失败原因 | fail_reason | varchar(255) | 失败时的原因说明 |
| 是否游客 | is_guest | tinyint unsigned | 0=登录用户 1=游客（USDT充值） |

#### SQL DDL

```sql
CREATE TABLE `deposit_order` (
    `id`                      bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`                varchar(20)      NOT NULL                COMMENT '平台订单号（C+16位随机数）',
    `channel_order_no`        varchar(50)      NOT NULL DEFAULT ''     COMMENT '通道订单号（财务模块回传）',
    `user_id`                 bigint unsigned  NOT NULL DEFAULT 0      COMMENT '用户ID（游客=0）',
    `currency_code`           varchar(10)      NOT NULL                COMMENT '订单币种',
    `wallet_type`             tinyint unsigned NOT NULL DEFAULT 1      COMMENT '钱包类型：默认1中心',
    `pay_method`              varchar(50)      NOT NULL                COMMENT '充值方式：USDT_TRC20/USDT_ERC20/BANK/EWALLET等',
    `order_amt`               decimal(20,4)    NOT NULL                COMMENT '订单金额（用户输入）',
    `rate_snapshot`           decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '入款汇率快照（创建订单时锁定）',
    `pay_currency`            varchar(10)      NOT NULL DEFAULT ''     COMMENT '实际支付币种（如USDT）',
    `pay_amt`                 decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '支付金额（汇率换算后）',
    `activity_id`             bigint unsigned  NOT NULL DEFAULT 0      COMMENT '奖金活动ID（0=无活动）',
    `bonus_amt`               decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '活动赠送金额',
    `audit_multiplier`        decimal(10,2)    NOT NULL DEFAULT 0      COMMENT '稽核倍数（奖金流水要求）',
    `total_credit_amt`        decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '到账总金额 = 订单金额 + 活动优惠',
    `fee_amt`                 decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '充值手续费',
    `fee_rate`                decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '充值费率%',
    `status`                  tinyint unsigned NOT NULL DEFAULT 1      COMMENT '状态：1待支付 2支付成功 3支付失败 4超时关闭',
    `base_order_amt`          decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种订单金额',
    `base_bonus_amt`          decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种活动优惠',
    `base_fee_amt`            decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种手续费',
    `platform_rate_snapshot`  decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '平台汇率快照（基准币种换算用）',
    `fail_reason`             varchar(255)     NOT NULL DEFAULT ''     COMMENT '失败原因',
    `is_guest`                tinyint unsigned NOT NULL DEFAULT 0      COMMENT '是否游客充值：0否 1是',
    `complete_at`             bigint unsigned  NOT NULL DEFAULT 0      COMMENT '完成时间（毫秒时间戳，0=未完成）',
    `create_at`               bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`               bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`),
    KEY `idx_user_create_at` (`user_id`, `create_at`),
    KEY `idx_status_create_at` (`status`, `create_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='充值订单表';
```

#### 设计说明

```
1. 无 delete_at：充值订单不允许删除，审计需保留
2. rate_snapshot（汇率快照）：
   · 遵循 Quote-Lock 模式（架构设计第六章）
   · 创建订单时锁定当时的入款汇率，后续计算都用此快照
   · 避免"用户确认时看到的金额"和"实际到账"不一致
3. base_xxx_amt 字段：
   · B端充值记录页需要展示基准币种换算后的金额
   · 在订单创建时就计算好写入，而不是查询时动态算
   · 因为查询时的汇率可能已经变了
4. idx_user_currency_status：重复转账检测需要查"同用户+同币种+待支付状态"的订单
5. 订单号格式 C+16位：产品需求明确规定，见需求 5.4
```

---

### 表 ⑥ withdraw_order（提现订单表）

#### 产品需求映射

```
产品原型来源：C端提现页面、B端提现记录列表
需求文档来源：需求 3.8（提现模块 7 个状态）、5.5（提现记录）
架构设计来源：TCC 冻结三步模式
```

#### SQL DDL

```sql
CREATE TABLE `withdraw_order` (
    `id`                      bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`                varchar(20)      NOT NULL                COMMENT '平台订单号（T+16位随机数）',
    `channel_order_no`        varchar(50)      NOT NULL DEFAULT ''     COMMENT '通道订单号（财务模块回传）',
    `user_id`                 bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`           varchar(10)      NOT NULL                COMMENT '提现币种',
    `wallet_type`             tinyint unsigned NOT NULL DEFAULT 1      COMMENT '钱包类型：默认1中心',
    `withdraw_method`         varchar(50)      NOT NULL                COMMENT '提现方式：BANK/EWALLET_GOPAY/EWALLET_OVO/USDT等',
    `order_amt`               decimal(20,4)    NOT NULL                COMMENT '提现金额（用户输入）',
    `rate_snapshot`           decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '出款汇率快照',
    `target_currency`         varchar(10)      NOT NULL DEFAULT ''     COMMENT '到账目标币种（BSB提现时需要，如VND/USDT）',
    `target_amt`              decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '目标币种到账金额',
    `fee_amt`                 decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '手续费',
    `fee_rate`                decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '手续费率%',
    `actual_amt`              decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '实际到账金额 = 提现金额 - 手续费',
    `frozen_amt`              decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '冻结金额（= 提现金额，冻结时写入）',
    `status`                  tinyint unsigned NOT NULL DEFAULT 1      COMMENT '状态：1待风控审核 2风控驳回 3待财务审核 4财务驳回 5待出款 6出款失败 7出款成功',
    `risk_reviewer`           bigint unsigned  NOT NULL DEFAULT 0      COMMENT '风控审核人ID',
    `risk_review_at`          bigint unsigned  NOT NULL DEFAULT 0      COMMENT '风控审核时间',
    `risk_remark`             varchar(255)     NOT NULL DEFAULT ''     COMMENT '风控审核备注',
    `finance_reviewer`        bigint unsigned  NOT NULL DEFAULT 0      COMMENT '财务审核人ID',
    `finance_review_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '财务审核时间',
    `finance_remark`          varchar(255)     NOT NULL DEFAULT ''     COMMENT '财务审核备注',
    `fail_reason`             varchar(255)     NOT NULL DEFAULT ''     COMMENT '失败/驳回原因',
    `lock_by`                 bigint unsigned  NOT NULL DEFAULT 0      COMMENT '当前锁定人ID（0=未锁定）',
    `lock_at`                 bigint unsigned  NOT NULL DEFAULT 0      COMMENT '锁定时间',
    `platform_rate_snapshot`  decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '平台汇率快照（基准币种换算用）',
    `base_order_amt`          decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种提现金额',
    `base_fee_amt`            decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种手续费',
    `complete_at`             bigint unsigned  NOT NULL DEFAULT 0      COMMENT '完成时间（出款成功/失败时间）',
    `create_at`               bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`               bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`),
    KEY `idx_user_create_at` (`user_id`, `create_at`),
    KEY `idx_status_create_at` (`status`, `create_at`),
    KEY `idx_lock_by` (`lock_by`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='提现订单表';
```

#### 状态机映射

```
产品需求中的提现 7 状态流转 → status 字段枚举值：

    1（待风控审核） ─→ 2（风控驳回） [终态，触发 UnfreezeBalance]
          │
          └──→ 3（待财务审核） ─→ 4（财务驳回） [终态，触发 UnfreezeBalance]
                     │
                     └──→ 5（待出款） ─→ 6（出款失败） [保持冻结，等人工]
                                │
                                └──→ 7（出款成功） [终态，触发 DeductFrozenAmount]

Go 枚举定义方向：
  const (
      WithdrawStatusRiskPending    = 1  // 待风控审核
      WithdrawStatusRiskRejected   = 2  // 风控驳回（终态）
      WithdrawStatusFinPending     = 3  // 待财务审核
      WithdrawStatusFinRejected    = 4  // 财务驳回（终态）
      WithdrawStatusPayPending     = 5  // 待出款
      WithdrawStatusPayFailed      = 6  // 出款失败（需人工）
      WithdrawStatusPaySuccess     = 7  // 出款成功（终态）
  )
```

#### 设计说明

```
1. frozen_amt 记录冻结的金额：
   · 创建订单时立即冻结（wallet_account.available -= frozen_amt, frozen += frozen_amt）
   · 这个值写入订单后不再变化，是TCC Try阶段的凭证
   · 解冻/扣冻结时以此值为准

2. lock_by / lock_at：
   · 产品需求明确有"锁定/解锁/强制解锁"机制
   · lock_by=0 表示未被锁定
   · 锁定后其他操作员不可同时审核此订单

3. 审核人和审核时间分开存储（risk/finance）：
   · 因为有两层审核，且可能不同人
   · B端需要显示"谁在什么时候审的"

4. 无 delete_at：提现订单不可删除
```

---

### 表 ⑦ exchange_order（兑换订单表）

#### 产品需求映射

```
产品原型来源：C端兑换页面
需求文档来源：需求 3.7（兑换模块 — 法币→BSB 单向）
```

#### SQL DDL

```sql
CREATE TABLE `exchange_order` (
    `id`                bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`          varchar(20)      NOT NULL                COMMENT '兑换订单号',
    `user_id`           bigint unsigned  NOT NULL                COMMENT '用户ID',
    `from_currency`     varchar(10)      NOT NULL                COMMENT '源币种（法币：VND/IDR/THB）',
    `to_currency`       varchar(10)      NOT NULL DEFAULT 'BSB'  COMMENT '目标币种（固定BSB）',
    `from_amt`          decimal(20,4)    NOT NULL                COMMENT '兑换金额（法币扣减额）',
    `rate_snapshot`     decimal(20,8)    NOT NULL                COMMENT '兑换汇率快照（当时的平台汇率）',
    `to_amt`            decimal(20,4)    NOT NULL                COMMENT '兑换所得BSB（不含赠送）',
    `bonus_pct`         decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '赠送百分比，如10.0000表示10%',
    `bonus_amt`         decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '赠送BSB金额',
    `total_credit_amt`  decimal(20,4)    NOT NULL                COMMENT '实际到账BSB = 兑换所得 + 赠送',
    `audit_multiplier`  decimal(10,2)    NOT NULL DEFAULT 0      COMMENT '赠送部分的稽核倍数',
    `status`            tinyint unsigned NOT NULL DEFAULT 1      COMMENT '状态：1处理中 2成功 3失败',
    `create_at`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_create_at` (`user_id`, `create_at`),
    KEY `idx_user_from_currency` (`user_id`, `from_currency`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='兑换订单表';
```

#### 设计说明

```
1. to_currency 默认 BSB：
   · 产品需求明确"只允许法币→BSB"，不可反向
   · 固定默认值，代码层也做方向校验

2. 兑换是同步操作（纯内部）：
   · 不涉及三方通道，一个事务内完成
   · 事务内同时操作两个币种的 wallet_account + 写两条 wallet_transaction
   · 所以 status 字段在正常情况下直接就是"成功"
   · 预留"失败"状态用于异常情况

3. bonus_amt 入奖励钱包：
   · 兑换所得 to_amt → BSB 中心钱包
   · 赠送金额 bonus_amt → BSB 奖励钱包
   · total_credit_amt = to_amt + bonus_amt
```

---

### 表 ⑧ withdraw_account（提现账户表）

#### 产品需求映射

```
产品原型来源：C端提现页面 — 银行卡信息/电子钱包信息
需求文档来源：需求 3.8.3（银行卡提现字段）、3.8.4（电子钱包提现字段）
关键规则：首次保存后用户不可修改，持有人必须与KYC姓名一致
```

#### SQL DDL

```sql
CREATE TABLE `withdraw_account` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`         bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码',
    `method_type`     varchar(30)      NOT NULL                COMMENT '提现方式类型：BANK/EWALLET_GOPAY/EWALLET_OVO/EWALLET_DANA/EWALLET_SHOPEEPAY/EWALLET_PROMPTPAY/USDT',
    `account_name`    varchar(100)     NOT NULL                COMMENT '账户持有人姓名（必须与KYC姓名一致）',
    `account_no`      varchar(100)     NOT NULL                COMMENT '账户号码（银行账号/手机号/USDT地址）',
    `bank_name`       varchar(100)     NOT NULL DEFAULT ''     COMMENT '银行名称（银行提现时必填）',
    `branch_name`     varchar(100)     NOT NULL DEFAULT ''     COMMENT '开户网点（选填）',
    `ewallet_type`    varchar(30)      NOT NULL DEFAULT ''     COMMENT '电子钱包类型：GoPay/OVO/DANA/ShopeePay/PromptPay',
    `usdt_network`    varchar(20)      NOT NULL DEFAULT ''     COMMENT 'USDT网络：TRC20/ERC20/BEP20（USDT提现时填）',
    `is_verified`     tinyint unsigned NOT NULL DEFAULT 0      COMMENT '是否已验证：0否 1是',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_currency_method` (`user_id`, `currency_code`, `method_type`),
    KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='提现账户表';
```

#### 设计说明

```
1. 唯一索引 (user_id, currency_code, method_type)：
   · 同一用户同一币种同一提现方式只保存一套账户信息
   · 如果用户有 VND 银行卡和 IDR 银行卡，是两条记录

2. 不可修改的实现方式：
   · 不是数据库约束，是代码逻辑
   · 保存接口：先查是否已存在 → 已存在则拒绝（返回错误码）
   · 只有"保存"没有"编辑"接口
   · 后续如需运营修改能力，再开 B 端接口

3. account_name 与 KYC 姓名一致性校验：
   · 保存时调 ser-kyc 获取 KYC 姓名
   · 比对逻辑：忽略大小写 + 忽略首尾空格 + 忽略连续空格压缩为单空格
   · strings.EqualFold(normalize(accountName), normalize(kycName))

4. 敏感信息处理：
   · account_no（银行账号/手机号）在返回给前端时需脱敏
   · 脱敏逻辑在 Service 层处理，不影响存储
```

---

## 六、D 类 — 辅助域数据模型

### 表 ⑨ audit_requirement（稽核流水要求表）

#### 产品需求映射

```
产品原型来源：充值奖金说明 "需完成xx倍流水要求"、兑换赠送说明
需求文档来源：需求 3.6（奖金机制）、3.7（兑换赠送稽核）、7.5（稽核规则）
```

#### SQL DDL

```sql
CREATE TABLE `audit_requirement` (
    `id`                bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`           bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`     varchar(10)      NOT NULL                COMMENT '币种代码',
    `wallet_type`       tinyint unsigned NOT NULL                COMMENT '关联钱包类型（通常2=奖励钱包）',
    `source_order_no`   varchar(50)      NOT NULL                COMMENT '来源订单号（充值/兑换/补单/加款的订单号）',
    `source_type`       varchar(30)      NOT NULL                COMMENT '来源类型：deposit_bonus/exchange_bonus/补单/manual_credit',
    `bonus_amt`         decimal(20,4)    NOT NULL                COMMENT '赠送/奖金金额',
    `audit_multiplier`  decimal(10,2)    NOT NULL                COMMENT '稽核倍数（如3.00表示3倍）',
    `required_turnover` decimal(20,4)    NOT NULL                COMMENT '需完成流水金额 = 赠送金额 × 稽核倍数',
    `completed_turnover`decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '已完成流水金额',
    `status`            tinyint unsigned NOT NULL DEFAULT 1      COMMENT '状态：1进行中 2已完成 3已过期',
    `create_at`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`),
    KEY `idx_source_order` (`source_order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='稽核流水要求表';
```

#### 设计说明

```
1. completed_turnover 的更新时机：
   · 每次用户消费（投注/购买等）时，检查是否有未完成的稽核记录
   · 消费金额累加到 completed_turnover
   · completed_turnover >= required_turnover 时，status 改为"已完成"
   · 具体更新逻辑取决于消费扣款是由钱包内部处理还是外部通知

2. 为什么不用 wallet_transaction 累加计算：
   · 性能原因——每次消费都 SUM 流水太重
   · 用物化字段 completed_turnover 做增量累加更高效

3. 多条稽核记录并存：
   · 用户可能有多次充值奖金、多次兑换赠送
   · 每次产生一条独立的稽核记录
   · 消费时按 FIFO（先进先出）或全部等比分摊——具体规则待产品确认
```

---

### 表 ⑩ deposit_duplicate_log（重复转账检测辅助表）

#### 产品需求映射

```
需求文档来源：需求 3.5（重复转账拦截规则）
规则：同UserID + 同币种 + 同金额，待支付/处理中 1-2笔警告，3笔强拦截
```

**此表是否需要单独建**——取决于设计选择：

```
方案 A（推荐）：直接查 deposit_order 表
  · SELECT COUNT(*) FROM deposit_order
    WHERE user_id=? AND currency_code=? AND order_amt=? AND status IN (1)
  · deposit_order 已有 idx_user_currency_status 索引，查询高效
  · 不需要额外建表
  · 不适用 USDT 链上充值（代码层判断 pay_method 跳过）

方案 B：单独建辅助表做快速计数
  · 在高并发场景下减少对 deposit_order 的查询压力
  · 用 Redis 计数也可以替代（key: wal:dup:{userId}:{currency}:{amt}）
```

**结论：此表不单独建，复用 deposit_order 表的索引查询。如需极端性能优化，改用 Redis 原子计数。**

---

### 表 ⑪ recharge_route_record（充值路由追踪表）

#### 产品需求映射

```
需求文档来源：需求 3.8.2（提现渠道路由规则 — "怎么充的怎么提"）
规则：法币充值→法币提现，USDT充值→USDT提现，混合充值→法币提现
```

**此表是否需要单独建**——同样取决于设计选择：

```
方案 A（推荐）：直接查 deposit_order 表
  · 提现前查用户历史充值记录：
    SELECT DISTINCT pay_method FROM deposit_order
    WHERE user_id=? AND status=2 (支付成功)
  · 如果有法币充值记录 → 允许法币提现
  · 如果只有 USDT 充值记录 → 仅允许 USDT 提现
  · 如果都有 → 法币提现（混合规则）
  · deposit_order 表已有 user_id 索引

方案 B：维护一张轻量级的用户充值路由标记表
  · 记录每个用户曾经使用过的充值方式类别
  · 首次充值时写入，后续只需查此表
  · 查询更快，但增加了维护成本
```

**结论：初期复用 deposit_order 表查询。如果后续发现查询性能不足，再考虑加缓存或辅助表。**

---

## 七、枚举值定义（所有表共用）

数据模型中的枚举字段需要在代码中有统一的常量定义。以下是需要定义的所有枚举：

### 7.1 钱包类型（wallet_type）

```go
// 来源：需求 4.1.1 钱包结构
const (
    WalletTypeCenter  = 1  // 中心钱包 — 用户主资金
    WalletTypeReward  = 2  // 奖励钱包 — 活动/奖金/赠送
    WalletTypeAnchor  = 3  // 主播钱包 — 直播/语聊收入
    WalletTypeAgent   = 4  // 代理钱包 — 代理佣金（预留）
    WalletTypeVenue   = 5  // 场馆钱包 — iGaming 临时（预留）
)
```

### 7.2 币种类型（currency_type）

```go
// 来源：需求 4.2 币种类型划分
const (
    CurrencyTypeFiat    = 1  // 法定货币（VND/IDR/THB）
    CurrencyTypeCrypto  = 2  // 加密货币（USDT/BTC/ETH）
    CurrencyTypePlatform = 3 // 平台货币（BSB）
)
```

### 7.3 流水变动类型（tx_type）

```go
// 来源：需求 3.9 账变记录 4 个 Tab + 所有资金操作场景
const (
    TxTypeDeposit       = "deposit"        // 充值到账
    TxTypeWithdraw      = "withdraw"       // 提现扣款
    TxTypeExchange      = "exchange"       // 兑换
    TxTypeReward        = "reward"         // 活动/任务奖励
    TxTypeBet           = "bet"            // 投注扣款
    TxTypeBetReward     = "bet_reward"     // 投注返奖
    TxTypeGift          = "gift"           // 购买礼物
    TxTypePurchase      = "purchase"       // 购买道具
    TxTypeManualCredit  = "manual_credit"  // 人工加款
    TxTypeManualDebit   = "manual_debit"   // 人工减款
    TxTypeRepair        = "repair"         // 充值补单
    TxTypeFreeze        = "freeze"         // 冻结
    TxTypeUnfreeze      = "unfreeze"       // 解冻
    TxTypeDeductFrozen  = "deduct_frozen"  // 扣除冻结
)
```

### 7.4 收支方向（direction）

```go
const (
    DirectionIn  = 1  // 收入（余额增加）
    DirectionOut = 2  // 支出（余额减少）
)
```

### 7.5 操作类型（op_type）

```go
// 钱包核心 5 方法的操作标识，用于幂等索引
const (
    OpTypeCredit       = "credit"         // 加款
    OpTypeDebit        = "debit"          // 减款
    OpTypeFreeze       = "freeze"         // 冻结
    OpTypeUnfreeze     = "unfreeze"       // 解冻
    OpTypeDeductFrozen = "deduct_frozen"  // 扣除冻结
)
```

### 7.6 充值订单状态（deposit status）

```go
// 来源：需求 6.5 状态流转
const (
    DepositStatusPending  = 1  // 待支付
    DepositStatusSuccess  = 2  // 支付成功
    DepositStatusFailed   = 3  // 支付失败
    DepositStatusTimeout  = 4  // 超时关闭
)
```

### 7.7 提现订单状态（withdraw status）

```go
// 来源：需求 5.5 提现 7 状态流转
const (
    WithdrawStatusRiskPending  = 1  // 待风控审核
    WithdrawStatusRiskRejected = 2  // 风控驳回（终态）
    WithdrawStatusFinPending   = 3  // 待财务审核
    WithdrawStatusFinRejected  = 4  // 财务驳回（终态）
    WithdrawStatusPayPending   = 5  // 待出款
    WithdrawStatusPayFailed    = 6  // 出款失败（需人工）
    WithdrawStatusPaySuccess   = 7  // 出款成功（终态）
)
```

### 7.8 稽核状态（audit status）

```go
const (
    AuditStatusActive    = 1  // 进行中
    AuditStatusCompleted = 2  // 已完成
    AuditStatusExpired   = 3  // 已过期
)
```

---

## 八、表间关系图

```
                          ┌──────────────────────┐
                          │  currency_config ①   │
                          │  (币种配置主表)        │
                          │  PK: id              │
                          │  UK: currency_code   │
                          └──────┬───────────────┘
                                 │
                    ┌────────────┼────────────────────────────────┐
                    │            │                                │
                    ▼            ▼                                ▼
          ┌─────────────┐ ┌──────────────┐              ┌──────────────────┐
          │exchange_rate │ │wallet_account│              │ deposit_order ⑤ │
          │  _log ②     │ │     ③       │              │ (充值订单)        │
          │(汇率日志)    │ │(钱包账户)    │              │ FK: currency_code│
          │FK:curr_code │ │FK:curr_code │              │ FK: user_id      │
          └─────────────┘ │UK:(uid,cc,wt)│              └──────────────────┘
                          └──────┬───────┘
                                 │
                    ┌────────────┼────────────────────────┐
                    │            │                        │
                    ▼            ▼                        ▼
          ┌──────────────┐ ┌──────────────┐     ┌──────────────────┐
          │  wallet_     │ │withdraw_order│     │ exchange_order ⑦│
          │  transaction │ │     ⑥       │     │ (兑换订单)        │
          │     ④       │ │(提现订单)    │     │ FK: from_currency│
          │(账变流水)    │ │FK:curr_code │     │ FK: user_id      │
          │FK:uid,cc,wt │ │FK: user_id  │     └──────────────────┘
          │UK:(on,op,wt)│ └──────┬───────┘
          └──────────────┘        │
                                  │
                    ┌─────────────┼────────────────┐
                    ▼             ▼                 ▼
          ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
          │  withdraw_   │ │   audit_     │ │ (被财务模块   │
          │  account ⑧  │ │ requirement  │ │  的审核操作   │
          │(提现账户)    │ │     ⑨       │ │  驱动状态变更)│
          │FK:uid,cc     │ │(稽核流水)    │ └──────────────┘
          │UK:(uid,cc,mt)│ │FK: user_id  │
          └──────────────┘ │FK: order_no │
                           └──────────────┘

关系说明（全部是逻辑外键，不建物理外键约束）：

wallet_account.currency_code  → currency_config.currency_code
wallet_transaction.user_id    → wallet_account.user_id
wallet_transaction.currency_code → wallet_account.currency_code
wallet_transaction.wallet_type → wallet_account.wallet_type
wallet_transaction.order_no   → deposit_order.order_no / withdraw_order.order_no / exchange_order.order_no
deposit_order.currency_code   → currency_config.currency_code
withdraw_order.currency_code  → currency_config.currency_code
withdraw_account.user_id      → wallet_account.user_id
audit_requirement.source_order_no → deposit_order.order_no / exchange_order.order_no
exchange_rate_log.currency_code → currency_config.currency_code
```

---

## 九、数据量预估与分表预判

| 表名 | 增长模式 | 预估数据量/年 | 分表必要性 |
|------|---------|-------------|----------|
| currency_config | 极低频 | < 100 行 | 不需要 |
| exchange_rate_log | 定时写入 | ~50万（每2分钟×5币种×365天） | 按时间归档即可 |
| wallet_account | 用户数 × 活跃币种 | ~100万（假设20万用户×5币种） | 不需要 |
| **wallet_transaction** | **每笔操作一条** | **~5000万+** | **后续可能需要按时间分表** |
| deposit_order | 每笔充值一条 | ~500万 | 中期可考虑 |
| withdraw_order | 每笔提现一条 | ~200万 | 暂不需要 |
| exchange_order | 每笔兑换一条 | ~100万 | 不需要 |
| withdraw_account | 用户数 × 方式 | ~50万 | 不需要 |
| audit_requirement | 带稽核的操作 | ~200万 | 暂不需要 |

**wallet_transaction 是唯一需要关注分表的表**：
- 当前不做分表（避免过度设计）
- create_at 字段预留为未来分表键
- TiDB 本身支持自动分区（Range Partition by create_at），到时可无缝启用

---

## 十、GORM Gen 配置方向

基于现有工程的 `gorm_gen.go` 配置模式，ser-wallet 的 Gen 配置：

```go
// 文件：ser-wallet/internal/gen/gorm_gen.go
package main

import (
    "common/pkg/db/cmd"
)

func main() {
    cmd.Generate(
        "./internal/gorm_gen",         // 输出目录
        namesp.EtcdWalletDB,           // ETCD 数据库配置键（修正后的）
        // 需要生成 Model/Query/Repo 的表
        "currency_config",
        "exchange_rate_log",
        "wallet_account",
        "wallet_transaction",
        "deposit_order",
        "withdraw_order",
        "exchange_order",
        "withdraw_account",
        "audit_requirement",
    )
}
```

**生成后的目录结构**：

```
ser-wallet/internal/gorm_gen/
├── model/                          ← 自动生成的 GORM Model（*.gen.go）
│   ├── currency_config.gen.go
│   ├── exchange_rate_log.gen.go
│   ├── wallet_account.gen.go
│   ├── wallet_transaction.gen.go
│   ├── deposit_order.gen.go
│   ├── withdraw_order.gen.go
│   ├── exchange_order.gen.go
│   ├── withdraw_account.gen.go
│   └── audit_requirement.gen.go
├── query/                          ← 自动生成的查询接口（*.gen.go）
│   └── ...
└── repo/                           ← 自动生成的 Repository（标准CRUD）
    ├── currency_config_repo.go     ← 每个表的增删改查方法
    ├── wallet_account_repo.go
    ├── wallet_transaction_repo.go
    ├── ...
    └── modelr/
        └── modelr.go
```

**自动生成的 Repo 提供的标准方法**（每张表都有）：

```go
// 以 wallet_account 为例
repo.WalletAccountRepo.Create(ctx, entity)
repo.WalletAccountRepo.GetOneById(ctx, id)
repo.WalletAccountRepo.GetOneByConds(ctx, conds...)
repo.WalletAccountRepo.GetListByConds(ctx, conds...)
repo.WalletAccountRepo.GetPageByConds(ctx, pageNo, pageSize, orderby, conds...)
repo.WalletAccountRepo.Update(ctx, entity)
repo.WalletAccountRepo.UpdateByConds(ctx, entity, conds...)
repo.WalletAccountRepo.DeleteById(ctx, ids...)
repo.WalletAccountRepo.Count(ctx, conds...)
repo.WalletAccountRepo.Exists(ctx, conds...)
```

**需要额外手写的 Repository 方法（Gen 标准方法不覆盖的）**：

| 方法 | 所属 Repo | 说明 |
|------|----------|------|
| `SelectForUpdate(ctx, userId, currencyCode, walletType)` | wallet_account | 行锁查询（Gen不生成FOR UPDATE） |
| `IncrAvailable(ctx, id, amount)` | wallet_account | 原子增加可用余额 |
| `DecrAvailable(ctx, id, amount)` | wallet_account | 原子减少可用余额 |
| `TransferToFrozen(ctx, id, amount)` | wallet_account | 可用→冻结的原子操作 |
| `TransferFromFrozen(ctx, id, amount)` | wallet_account | 冻结→可用的原子操作 |
| `DeductFrozen(ctx, id, amount)` | wallet_account | 扣除冻结（资金离开系统） |
| `InsertOnly(ctx, entity)` | wallet_transaction | 强调此表只有INSERT |
| `GetLatestByAccount(ctx, userId, currencyCode, walletType)` | wallet_transaction | 获取最近一条流水（对账用） |
| `SumByOrderNo(ctx, orderNo)` | wallet_transaction | 按订单号汇总流水（返奖拆分用） |

---

## 十一、初始数据

系统启动前需要初始化的数据：

### 11.1 币种配置初始数据

```sql
-- BSB（系统内置，不可修改）
INSERT INTO currency_config (currency_code, currency_name, currency_type, country,
    currency_symbol, thousands_sep, decimal_sep, decimal_digits,
    realtime_rate, platform_rate, deposit_rate, withdraw_rate,
    sort_value, status, is_base, is_builtin, create_at, update_at)
VALUES ('BSB', 'BSB', 3, '全球',
    'BSB', ',', '.', 2,
    0.1, 0.1, 0.1, 0.1,
    1, 1, 0, 1, UNIX_TIMESTAMP()*1000, UNIX_TIMESTAMP()*1000);
-- BSB 汇率固定：10 BSB = 1 USDT → 1 BSB = 0.1 USDT

-- VND（待启用，运营配置后上线）
INSERT INTO currency_config (currency_code, currency_name, currency_type, country,
    currency_symbol, thousands_sep, decimal_sep, decimal_digits,
    sort_value, status, is_base, is_builtin, create_at, update_at)
VALUES ('VND', '越南盾', 1, '越南',
    'd', '.', ',', 0,
    10, 0, 0, 0, UNIX_TIMESTAMP()*1000, UNIX_TIMESTAMP()*1000);

-- IDR
INSERT INTO currency_config (currency_code, currency_name, currency_type, country,
    currency_symbol, thousands_sep, decimal_sep, decimal_digits,
    sort_value, status, is_base, is_builtin, create_at, update_at)
VALUES ('IDR', '印尼盾', 1, '印尼',
    'Rp', '.', ',', 0,
    20, 0, 0, 0, UNIX_TIMESTAMP()*1000, UNIX_TIMESTAMP()*1000);

-- THB
INSERT INTO currency_config (currency_code, currency_name, currency_type, country,
    currency_symbol, thousands_sep, decimal_sep, decimal_digits,
    sort_value, status, is_base, is_builtin, create_at, update_at)
VALUES ('THB', '泰铢', 1, '泰国',
    '฿', ',', '.', 2,
    30, 0, 0, 0, UNIX_TIMESTAMP()*1000, UNIX_TIMESTAMP()*1000);

-- USDT（基准币种候选）
INSERT INTO currency_config (currency_code, currency_name, currency_type, country,
    currency_symbol, thousands_sep, decimal_sep, decimal_digits,
    sort_value, status, is_base, is_builtin, create_at, update_at)
VALUES ('USDT', 'Tether', 2, '全球',
    '$', ',', '.', 8,
    100, 1, 0, 0, UNIX_TIMESTAMP()*1000, UNIX_TIMESTAMP()*1000);
-- USDT 的 is_base 初始为 0，由运营在后台设置基准币种时改为 1
```

---

## 十二、后续步骤

数据模型完成后，接下来的工作顺序：

```
Step 1: 评审数据模型
  · 与财务模块确认金额字段精度 DECIMAL(20,4) 是否对齐
  · 与财务模块确认 wallet_type 枚举值是否一致
  · 与产品确认稽核规则的详细逻辑（FIFO 还是等比分摊）
  · 确认 deposit_order 归属（目前设计在 ser-wallet 内，需评审确认）

Step 2: 建库建表
  · 在 TiDB 中创建 ser-wallet 数据库
  · 执行本文档中的 SQL DDL
  · 插入初始化数据

Step 3: GORM Gen 代码生成
  · 配置 gorm_gen.go（修正 namesp.EtcdWalletDB）
  · 执行 go run 生成 model/query/repo

Step 4: 手写扩展 Repo
  · 补充 SelectForUpdate、IncrAvailable 等 Gen 不覆盖的方法

Step 5: 定义枚举和错误码
  · 将第七章的枚举值定义为 Go 常量
  · 定义错误码（实现思路第四章的 6022xxx 体系）

Step 6: 进入业务开发
  · 按实现思路的 Phase 顺序：币种配置 → 钱包核心 → C端功能 → 联调
```

---

## 十三、总结

### 13.1 数据模型全览

| # | 表名 | 所属域 | 核心用途 | 行增长 |
|---|------|--------|---------|--------|
| ① | currency_config | 币种配置 | 币种信息+汇率配置 | 极低 |
| ② | exchange_rate_log | 币种配置 | 汇率变更审计日志 | 中 |
| ③ | wallet_account | 钱包核心 | 物化余额（可用+冻结） | 低 |
| ④ | wallet_transaction | 钱包核心 | 不可变账变流水 | **高** |
| ⑤ | deposit_order | 业务订单 | 充值订单全生命周期 | 高 |
| ⑥ | withdraw_order | 业务订单 | 提现订单(7状态流转) | 中 |
| ⑦ | exchange_order | 业务订单 | 兑换订单 | 中 |
| ⑧ | withdraw_account | 业务订单 | 用户提现账户信息 | 低 |
| ⑨ | audit_requirement | 辅助 | 稽核流水要求跟踪 | 中 |

### 13.2 关键设计决策回顾

| 决策 | 选择 | 依据 |
|------|------|------|
| 金额类型 | DECIMAL(20,4) | 架构设计 ADR-05，覆盖所有币种精度 |
| 汇率精度 | DECIMAL(20,8) | 加密货币行业惯例 8 位小数 |
| 时间戳 | bigint unsigned 毫秒 | 金融场景需要更高精度 |
| 主键 | bigint unsigned AUTO_INCREMENT | 现有工程统一规范 |
| 流水主键 | Snowflake ID（非自增） | 分表友好，时间有序 |
| 关联关系 | 逻辑外键，无物理约束 | 现有工程统一规范 |
| 索引 | 仅在 SQL DDL 定义，不在 Go Model 中 | 现有工程 FieldWithIndexTag:false |
| 软删除 | 视表而定 | 配置表/账户表需要，流水表/日志表不需要 |

### 13.3 一句话总结

**9 张核心表构成 ser-wallet 的数据骨架：2 张币种配置表提供基础数据，2 张钱包核心表（账户+流水）保障资金安全，4 张业务订单表承载充值/提现/兑换/提现账户的完整生命周期，1 张稽核表跟踪流水要求。所有表遵循现有工程规范（bigint 主键、毫秒时间戳、snake_case 命名、逻辑外键），金额统一用 DECIMAL 存储，流水表只追加不修改，为后续 GORM Gen 代码生成和业务开发打好基础。**
