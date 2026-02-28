# 钱包 + 币种配置 + 财务管理 — 全局表结构设计

> 编写时间：2026-02-26
> 视角：钱包、币种配置、财务管理三个模块作为一个"资金域"整体，统一审视需要哪些表
> 分级标准：Must（没有就不能运转）/ Should（业务完整性需要）/ Could（锦上添花可延后）
> 工程规范：基于现有工程实际代码分析（ser-app/ser-item/ser-buser DDL + GORM Gen）
> 需求依据：/Users/mac/gitlab/z-readme/result/tmp-2.md（152张产品原型完整分析）
> 数据模型依据：/Users/mac/gitlab/z-readme/数据模型/tmp-2.md
> 架构设计依据：/Users/mac/gitlab/z-readme/架构设计/tmp-2.md

---

## 一、全局表结构总览

### 1.1 三个模块各自需要什么表

产品需求定义了三个模块的完整功能，每个模块背后都需要数据表来承载。先从宏观看全貌：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        资金域 — 全量表结构地图                               │
│                                                                          │
│  ┌── 币种配置模块（我方）──────────────────────────────────────────────┐   │
│  │                                                                    │   │
│  │  [Must] currency_config         币种配置主表（20+字段，全系统基础）   │   │
│  │  [Must] exchange_rate_log       汇率变更日志（审计追踪，不可省略）     │   │
│  │  [Could] rate_api_source        三方汇率API源配置（可用ETCD替代）     │   │
│  │                                                                    │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌── 钱包模块（我方）────────────────────────────────────────────────┐   │
│  │                                                                    │   │
│  │  [Must] wallet_account          钱包账户表（物化余额，系统心脏）       │   │
│  │  [Must] wallet_transaction      账变流水表（不可变账本，审计核心）     │   │
│  │  [Must] deposit_order           充值订单表（充值全生命周期）           │   │
│  │  [Must] withdraw_order          提现订单表（7状态+冻结+审核）         │   │
│  │  [Must] withdraw_account        提现账户表（银行卡/电子钱包信息）      │   │
│  │  [Should] exchange_order        兑换订单表（法币→BSB记录）            │   │
│  │  [Should] audit_requirement     稽核流水要求表（防薅羊毛门槛）         │   │
│  │  [Could] reconciliation_diff    对账差异记录表（对账发现的差异）        │   │
│  │                                                                    │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌── 财务管理模块（他方，列出以供参考和联调对齐）───────────────────────┐   │
│  │                                                                    │   │
│  │  [Must] payment_channel         支付通道配置表（代收/代付）           │   │
│  │  [Must] deposit_method          充值方式配置表                       │   │
│  │  [Must] withdraw_method         提现方式配置表                       │   │
│  │  [Must] channel_order           通道侧订单表（通道订单号+回调记录）    │   │
│  │  [Should] manual_adjustment     人工修正记录表（补单/加款/减款）       │   │
│  │  [Should] polling_rule_config   轮询规则配置表                       │   │
│  │  [Could] channel_statistics     通道统计汇总表                       │   │
│  │                                                                    │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  总计：我方 Must 7张 + Should 2张 + Could 2张 = 11张                      │
│        他方 Must 4张 + Should 2张 + Could 1张 = 7张（参考）               │
│        合计约 18张表支撑整个资金域                                          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 分级依据说明

| 级别 | 判定标准 | 如果没有会怎样 |
|------|---------|-------------|
| **Must** | 产品需求中明确定义的核心功能所必需的数据载体 | 系统无法运转，核心流程断裂 |
| **Should** | 产品需求中明确提到但可以短期通过其他方式替代 | 业务完整性缺失，部分功能受限 |
| **Could** | 产品需求中未直接提及，但工程健壮性或运营便利性需要 | 不影响主流程，可后期补充 |

---

## 二、工程建表规范速查

所有表遵循以下规范（来源：现有工程 ser-app/ser-item/ser-buser 实际 DDL 分析）：

```
主键：        bigint unsigned NOT NULL AUTO_INCREMENT
表名：        snake_case（可带模块前缀）
列名：        snake_case
字符集：      utf8mb4 COLLATE utf8mb4_bin
引擎：        InnoDB（支持事务+行锁）
时间字段：    bigint unsigned，毫秒级时间戳
软删除：      delete_at bigint unsigned DEFAULT 0（0=未删除）
审计字段：    create_by / create_by_name / update_by / update_by_name（B端表）
注释：        每列必须有 COMMENT
唯一索引：    uk_ 前缀
普通索引：    idx_ 前缀
金额字段：    decimal(20,4)（覆盖所有法币+平台币精度）
汇率字段：    decimal(20,8)（覆盖加密货币8位小数）
百分比字段：  decimal(10,4)（表示 xx.xx%）
```

---

## 三、Must 级 — 没有就不能运转的表

---

### 表 1：currency_config（币种配置主表）

**级别：Must** | **所属：币种配置模块（我方）** | **优先级：最高（地基）**

**这张表做什么**：存储系统中所有可用币种的配置信息——代码、名称、类型、汇率、浮动规则、显示规则、状态。它是整个资金域的"基础数据源"，钱包的每个操作（充值/提现/兑换/查余额）都需要读取币种配置。

**产品需求来源**：需求 4.2-4.5 币种配置页 20 个字段 + 编辑弹窗 + 基准币种规则

**如果没有这张表**：所有币种相关的操作都无法进行——不知道支持哪些币种、汇率是多少、精度规则是什么。

```sql
CREATE TABLE `currency_config` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码（唯一），如 BSB/VND/IDR/THB/USDT',
    `currency_name`   varchar(50)      NOT NULL DEFAULT ''     COMMENT '币种名称，如 越南盾',
    `currency_type`   tinyint unsigned NOT NULL                COMMENT '币种类型：1法定货币 2加密货币 3平台货币',
    `country`         varchar(50)      NOT NULL DEFAULT '全球'  COMMENT '币种国家/地区',
    `currency_symbol` varchar(10)      NOT NULL DEFAULT ''     COMMENT '货币符号，如 d / Rp / ฿ / $',
    `thousands_sep`   varchar(5)       NOT NULL DEFAULT ','    COMMENT '千分位分隔符',
    `decimal_sep`     varchar(5)       NOT NULL DEFAULT '.'    COMMENT '小数点分隔符',
    `decimal_digits`  tinyint unsigned NOT NULL DEFAULT 2      COMMENT '显示小数位数：法币0或2，加密8',
    `realtime_rate`   decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '实时汇率（三方API平均值，对基准币种）',
    `platform_rate`   decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '平台汇率（偏差>=阈值时更新）',
    `rate_float_pct`  decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '汇率浮动百分比，如1.4000=1.4%',
    `deposit_rate`    decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '入款汇率 = 平台汇率×(1+浮动%)',
    `withdraw_rate`   decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '出款汇率 = 平台汇率×(1-浮动%)',
    `deviation_pct`   decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '当前偏差% = |实时-平台|/实时×100',
    `threshold_pct`   decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '阈值%，偏差>=此值触发平台汇率更新',
    `icon_url`        varchar(255)     NOT NULL DEFAULT ''     COMMENT '币种图标URL（SVG/PNG <=5KB）',
    `sort_value`      int unsigned     NOT NULL DEFAULT 1000   COMMENT 'C端展示排序，越小越靠前',
    `status`          tinyint unsigned NOT NULL DEFAULT 0      COMMENT '状态：0禁用 1启用',
    `is_base`         tinyint unsigned NOT NULL DEFAULT 0      COMMENT '是否基准币种：0否 1是（全局仅一个=1，设后不可改）',
    `is_builtin`      tinyint unsigned NOT NULL DEFAULT 0      COMMENT '是否系统内置：0否 1是（BSB=1，不可配置）',
    `create_by`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建人ID',
    `create_by_name`  varchar(50)      NOT NULL DEFAULT ''     COMMENT '创建人名称',
    `update_by`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '最后操作人ID',
    `update_by_name`  varchar(50)      NOT NULL DEFAULT ''     COMMENT '最后操作人名称',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    `delete_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '删除时间（0=未删除）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_currency_code` (`currency_code`),
    KEY `idx_status` (`status`),
    KEY `idx_delete_at` (`delete_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='币种配置表';
```

**字段总数：28**（含主键+审计字段）

**关键设计点**：
- `is_base`：全局仅一条=1，代码层保证一次性设置不可改（需求4.3）
- `is_builtin`：BSB=1，汇率/代码/类型/状态均不可修改（需求4.4.4）
- 汇率字段用 `decimal(20,8)` 覆盖加密货币8位精度
- B端管理表，需要完整审计字段（create_by_name/update_by_name）

---

### 表 2：exchange_rate_log（汇率变更日志表）

**级别：Must** | **所属：币种配置模块（我方）** | **优先级：最高**

**这张表做什么**：每次平台汇率因偏差超过阈值而更新时，自动生成一条日志。是运营人员查看汇率变动历史的数据源，也是汇率审计的依据。

**产品需求来源**：需求 4.6 汇率日志 Tab（操作日志页面内）

**如果没有这张表**：运营无法追踪汇率何时变过、变了多少，出现汇率相关纠纷时无法举证。产品原型中明确有这个页面。

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

**字段总数：12**

**关键设计点**：
- 只有 INSERT，没有 UPDATE/DELETE（审计日志不可变）
- 没有 delete_at / update_at（不可修改就不需要这些字段）
- 没有 create_by（系统定时任务自动生成，无人工操作）
- 筛选：按 `currency_code + update_time` 范围查（产品需求4.6）

---

### 表 3：wallet_account（钱包账户表）

**级别：Must** | **所属：钱包模块（我方）** | **优先级：最高（系统心脏）**

**这张表做什么**：存储每个用户在每个币种下每种钱包类型的余额（可用余额 + 冻结余额）。是整个钱包系统的核心数据，所有"加钱/扣钱/冻结/解冻"操作的目标。

**产品需求来源**：需求 4.1（5种钱包结构）、3.3（币种隔离）、3.8（冻结/解冻机制）

**如果没有这张表**：系统不知道用户有多少钱。整个钱包无法运作。

```sql
CREATE TABLE `wallet_account` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`         bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码',
    `wallet_type`     tinyint unsigned NOT NULL                COMMENT '钱包类型：1中心 2奖励 3主播 4代理 5场馆',
    `available_amt`   decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '可用余额（用户可花的钱）',
    `frozen_amt`      decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '冻结余额（提现等操作锁定的钱）',
    `version`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '乐观锁版本号（主用悲观锁，此字段备用）',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_currency_type` (`user_id`, `currency_code`, `wallet_type`),
    KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='钱包账户表';
```

**字段总数：9**

**关键设计点**：
- 唯一索引 `(user_id, currency_code, wallet_type)` = 一个账户的唯一标识
- 不存 `total_amt` → 总余额 = available + frozen，代码计算，减少冗余
- 不存 `delete_at` → 账户一旦创建不可删除（审计需要）
- 不存 `create_by` → C端操作，主体就是 user_id 自己
- 懒创建：首次操作时自动创建，不在注册时批量创建
- 并发控制：SELECT ... FOR UPDATE 锁住这一行再操作（架构设计第三章）

---

### 表 4：wallet_transaction（账变流水表）

**级别：Must** | **所属：钱包模块（我方）** | **优先级：最高（审计核心）**

**这张表做什么**：记录每一笔余额变动的完整信息——谁、什么时候、什么操作、变了多少、变前多少、变后多少、关联哪个订单。是"不可变账本"，只追加不修改不删除。

**产品需求来源**：需求 3.9（账变记录4个Tab）、3.10（对账一致性 + 幂等性）

**如果没有这张表**：无法知道余额是怎么变的、无法给用户展示流水记录、无法对账、无法审计。

```sql
CREATE TABLE `wallet_transaction` (
    `id`              bigint unsigned  NOT NULL                COMMENT '流水ID（Snowflake或自增，分表友好）',
    `user_id`         bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码',
    `wallet_type`     tinyint unsigned NOT NULL                COMMENT '钱包类型：1中心 2奖励 3主播 4代理 5场馆',
    `tx_type`         varchar(30)      NOT NULL                COMMENT '变动类型：deposit/withdraw/exchange/reward/bet/manual_credit/manual_debit等',
    `direction`       tinyint unsigned NOT NULL                COMMENT '收支方向：1收入(+) 2支出(-)',
    `amount`          decimal(20,4)    NOT NULL                COMMENT '变动金额（绝对值，>0）',
    `before_amt`      decimal(20,4)    NOT NULL                COMMENT '变动前可用余额',
    `after_amt`       decimal(20,4)    NOT NULL                COMMENT '变动后可用余额',
    `frozen_change`   decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '冻结变动（正=冻结增加，负=冻结减少）',
    `order_no`        varchar(50)      NOT NULL                COMMENT '关联业务订单号（C/T/B/A/M+16位）',
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

**字段总数：14**

**关键设计点**：
- **不可变**：永远不执行 UPDATE 和 DELETE（代码层强制）
- 没有 update_at / delete_at（不可变就不需要）
- `before_amt / after_amt`：审计链——第N条的 after = 第N+1条的 before
- 幂等唯一索引 `(order_no, op_type, wallet_type)`：
  - 为什么三元组：人工加款一个订单可操作4个钱包（wallet_type不同），同一订单提现有 freeze + deduct_frozen（op_type不同）
  - 这是 Redis 幂等键的数据库层兜底
- 此表是**唯一可能需要分表的表**（数据量增长最快）

---

### 表 5：deposit_order（充值订单表）

**级别：Must** | **所属：钱包模块（我方）** | **优先级：高**

**这张表做什么**：记录每一笔充值订单的完整生命周期——从用户发起充值到最终成功/失败/超时。包含订单金额、汇率快照、充值方式、奖金活动、状态流转等信息。

**产品需求来源**：需求 3.4（充值模块）、5.4（充值记录24个字段）、3.5（重复转账拦截依据）

**如果没有这张表**：充值流程无法运转——不知道用户充了多少钱、订单到什么状态了、是否需要入账。

```sql
CREATE TABLE `deposit_order` (
    `id`                      bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`                varchar(20)      NOT NULL                COMMENT '平台订单号（C+16位随机数）',
    `channel_order_no`        varchar(50)      NOT NULL DEFAULT ''     COMMENT '通道订单号（财务模块回传）',
    `user_id`                 bigint unsigned  NOT NULL DEFAULT 0      COMMENT '用户ID（游客=0）',
    `currency_code`           varchar(10)      NOT NULL                COMMENT '订单币种',
    `wallet_type`             tinyint unsigned NOT NULL DEFAULT 1      COMMENT '钱包类型：默认1中心',
    `pay_method`              varchar(50)      NOT NULL                COMMENT '充值方式：USDT_TRC20/BANK/EWALLET等',
    `order_amt`               decimal(20,4)    NOT NULL                COMMENT '订单金额（用户输入）',
    `rate_snapshot`           decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '入款汇率快照（Quote-Lock）',
    `pay_currency`            varchar(10)      NOT NULL DEFAULT ''     COMMENT '实际支付币种（如USDT）',
    `pay_amt`                 decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '支付金额（汇率换算后）',
    `activity_id`             bigint unsigned  NOT NULL DEFAULT 0      COMMENT '奖金活动ID（0=无活动）',
    `bonus_amt`               decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '活动赠送金额',
    `audit_multiplier`        decimal(10,2)    NOT NULL DEFAULT 0      COMMENT '稽核倍数',
    `total_credit_amt`        decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '到账总金额 = 订单金额 + 活动赠送',
    `fee_amt`                 decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '充值手续费',
    `fee_rate`                decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '充值费率%',
    `status`                  tinyint unsigned NOT NULL DEFAULT 1      COMMENT '状态：1待支付 2支付成功 3支付失败 4超时关闭',
    `base_order_amt`          decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种订单金额（报表用）',
    `base_bonus_amt`          decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种活动赠送（报表用）',
    `base_fee_amt`            decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种手续费（报表用）',
    `platform_rate_snapshot`  decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '平台汇率快照（基准币种换算用）',
    `fail_reason`             varchar(255)     NOT NULL DEFAULT ''     COMMENT '失败原因',
    `is_guest`                tinyint unsigned NOT NULL DEFAULT 0      COMMENT '是否游客充值：0否 1是',
    `complete_at`             bigint unsigned  NOT NULL DEFAULT 0      COMMENT '完成时间（毫秒，0=未完成）',
    `create_at`               bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`               bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`),
    KEY `idx_user_create_at` (`user_id`, `create_at`),
    KEY `idx_status_create_at` (`status`, `create_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='充值订单表';
```

**字段总数：27**

**关键设计点**：
- `rate_snapshot`：Quote-Lock 模式，创建时锁定汇率，后续计算都用此快照
- `base_xxx_amt`：B端报表需要基准币种金额，创建时计算好写入
- `idx_user_currency_status`：重复转账检测查询 "同用户+同币种+待支付" 的索引
- 状态流转简单：待支付 → 成功/失败/超时
- 不需要 delete_at（充值订单不可删除）

**此表同时承载"重复转账检测"功能**（需求3.5）：
- 查询 `WHERE user_id=? AND currency_code=? AND order_amt=? AND status=1` 计数
- 1-2笔：警告可继续；3笔：强拦截
- 不需要额外建表，复用此索引即可

---

### 表 6：withdraw_order（提现订单表）

**级别：Must** | **所属：钱包模块（我方）** | **优先级：高**

**这张表做什么**：记录每一笔提现订单的完整生命周期——7个状态的流转、冻结金额记录、风控审核/财务审核的审核人和审核时间、锁定机制。是提现流程中最复杂的表。

**产品需求来源**：需求 3.8（提现模块）、5.5（7状态流转+锁定/解锁机制）

**如果没有这张表**：提现流程无法运转——不知道冻结了多少钱、审核到哪一步了、出款结果是什么。

```sql
CREATE TABLE `withdraw_order` (
    `id`                      bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`                varchar(20)      NOT NULL                COMMENT '平台订单号（T+16位随机数）',
    `channel_order_no`        varchar(50)      NOT NULL DEFAULT ''     COMMENT '通道订单号（财务模块回传）',
    `user_id`                 bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`           varchar(10)      NOT NULL                COMMENT '提现币种',
    `wallet_type`             tinyint unsigned NOT NULL DEFAULT 1      COMMENT '钱包类型',
    `withdraw_method`         varchar(50)      NOT NULL                COMMENT '提现方式：BANK/EWALLET_GOPAY/USDT等',
    `order_amt`               decimal(20,4)    NOT NULL                COMMENT '提现金额',
    `rate_snapshot`           decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '出款汇率快照',
    `target_currency`         varchar(10)      NOT NULL DEFAULT ''     COMMENT '目标币种（BSB提现时选择的法币/USDT）',
    `target_amt`              decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '目标币种到账金额',
    `fee_amt`                 decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '手续费',
    `fee_rate`                decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '手续费率%',
    `actual_amt`              decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '实际到账 = 提现金额 - 手续费',
    `frozen_amt`              decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '冻结金额（创建时写入，TCC凭证）',
    `status`                  tinyint unsigned NOT NULL DEFAULT 1      COMMENT '1待风控 2风控驳回 3待财务 4财务驳回 5待出款 6出款失败 7出款成功',
    `risk_reviewer`           bigint unsigned  NOT NULL DEFAULT 0      COMMENT '风控审核人ID',
    `risk_review_at`          bigint unsigned  NOT NULL DEFAULT 0      COMMENT '风控审核时间',
    `risk_remark`             varchar(255)     NOT NULL DEFAULT ''     COMMENT '风控审核备注',
    `finance_reviewer`        bigint unsigned  NOT NULL DEFAULT 0      COMMENT '财务审核人ID',
    `finance_review_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '财务审核时间',
    `finance_remark`          varchar(255)     NOT NULL DEFAULT ''     COMMENT '财务审核备注',
    `fail_reason`             varchar(255)     NOT NULL DEFAULT ''     COMMENT '失败/驳回原因',
    `lock_by`                 bigint unsigned  NOT NULL DEFAULT 0      COMMENT '当前锁定人ID（0=未锁定）',
    `lock_at`                 bigint unsigned  NOT NULL DEFAULT 0      COMMENT '锁定时间',
    `platform_rate_snapshot`  decimal(20,8)    NOT NULL DEFAULT 0      COMMENT '平台汇率快照（报表用）',
    `base_order_amt`          decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种提现金额',
    `base_fee_amt`            decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '基准币种手续费',
    `complete_at`             bigint unsigned  NOT NULL DEFAULT 0      COMMENT '完成时间',
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

**字段总数：31**

**状态机（产品需求5.5）**：

```
1(待风控) → 2(风控驳回) [终态, 解冻]
         → 3(待财务)   → 4(财务驳回) [终态, 解冻]
                       → 5(待出款)   → 6(出款失败) [保持冻结, 等人工]
                                     → 7(出款成功) [终态, 扣冻结]
```

**关键设计点**：
- `frozen_amt`：TCC Try 阶段的凭证，创建时写入后不变
- `lock_by / lock_at`：产品明确的锁定/解锁机制（需求5.5）
- 两层审核分开存（risk/finance）：不同人不同时间
- 字段最多的表（31个），因为提现是最复杂的业务流程

**此表同时承载"提现渠道路由追踪"功能**（需求3.8.2）：
- 查询 `deposit_order WHERE user_id=? AND status=2` 判断历史充值方式
- 决定允许的提现渠道："法币充→法币提，USDT充→USDT提"
- 不需要额外建表

---

### 表 7：withdraw_account（提现账户表）

**级别：Must** | **所属：钱包模块（我方）** | **优先级：高**

**这张表做什么**：存储用户保存的提现账户信息——银行卡信息、电子钱包信息、USDT地址。首次提现时保存，后续自动填充，用户不可自行修改。

**产品需求来源**：需求 3.8.3（银行卡字段）、3.8.4（电子钱包字段）、3.10（账户持有人=KYC姓名）

**如果没有这张表**：用户每次提现都要重新填写账户信息，无法实现"首次保存后续自动填充"的产品体验。更重要的是无法保证"用户不可自行修改提现账户"的安全规则。

```sql
CREATE TABLE `withdraw_account` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`         bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码',
    `method_type`     varchar(30)      NOT NULL                COMMENT '提现方式：BANK/EWALLET_GOPAY/EWALLET_OVO/EWALLET_DANA/EWALLET_SHOPEEPAY/EWALLET_PROMPTPAY/USDT',
    `account_name`    varchar(100)     NOT NULL                COMMENT '账户持有人（必须与KYC姓名一致）',
    `account_no`      varchar(100)     NOT NULL                COMMENT '账号（银行账号/手机号/USDT地址）',
    `bank_name`       varchar(100)     NOT NULL DEFAULT ''     COMMENT '银行名称（银行提现时填）',
    `branch_name`     varchar(100)     NOT NULL DEFAULT ''     COMMENT '开户网点（选填）',
    `ewallet_type`    varchar(30)      NOT NULL DEFAULT ''     COMMENT '电子钱包类型',
    `usdt_network`    varchar(20)      NOT NULL DEFAULT ''     COMMENT 'USDT网络：TRC20/ERC20/BEP20',
    `is_verified`     tinyint unsigned NOT NULL DEFAULT 0      COMMENT '是否已验证：0否 1是',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间（毫秒时间戳）',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_currency_method` (`user_id`, `currency_code`, `method_type`),
    KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='提现账户表';
```

**字段总数：13**

**关键设计点**：
- 唯一索引保证同一用户同一币种同一方式只有一套账户
- "不可修改"由代码逻辑保证——保存接口查到已存在则拒绝
- `account_name` 保存时必须与 KYC 姓名做模糊匹配（忽略大小写+多余空格）
- `account_no` 返回前端时需脱敏处理（Service层）

---

## 四、Should 级 — 业务完整性需要的表

---

### 表 8：exchange_order（兑换订单表）

**级别：Should** | **所属：钱包模块（我方）** | **优先级：中**

**这张表做什么**：记录每一笔法币→BSB兑换操作的订单信息——源币种金额、汇率快照、兑换所得、赠送金额、稽核倍数。

**产品需求来源**：需求 3.7（兑换模块）

**为什么是 Should 而不是 Must**：
- 兑换操作本质上已经被 wallet_transaction 记录了（法币出+BSB入各一条流水）
- 这张表更多是"兑换订单的业务视图"——汇率快照、赠送比例、稽核信息
- 如果不建此表，兑换的流水记录仍然存在，只是查询"兑换历史"会麻烦一些
- 但产品原型中账变记录有"兑换"Tab，需要展示兑换金额/赠送%/实际到账，这些信息单靠流水表不够完整

**结论**：实际开发中应该建。

```sql
CREATE TABLE `exchange_order` (
    `id`                bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`          varchar(20)      NOT NULL                COMMENT '兑换订单号',
    `user_id`           bigint unsigned  NOT NULL                COMMENT '用户ID',
    `from_currency`     varchar(10)      NOT NULL                COMMENT '源币种（VND/IDR/THB）',
    `to_currency`       varchar(10)      NOT NULL DEFAULT 'BSB'  COMMENT '目标币种（固定BSB）',
    `from_amt`          decimal(20,4)    NOT NULL                COMMENT '法币扣减金额',
    `rate_snapshot`     decimal(20,8)    NOT NULL                COMMENT '兑换汇率快照',
    `to_amt`            decimal(20,4)    NOT NULL                COMMENT '兑换所得BSB（不含赠送）',
    `bonus_pct`         decimal(10,4)    NOT NULL DEFAULT 0      COMMENT '赠送比例%',
    `bonus_amt`         decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '赠送BSB金额',
    `total_credit_amt`  decimal(20,4)    NOT NULL                COMMENT '实际到账BSB = 兑换所得 + 赠送',
    `audit_multiplier`  decimal(10,2)    NOT NULL DEFAULT 0      COMMENT '赠送部分稽核倍数',
    `status`            tinyint unsigned NOT NULL DEFAULT 2      COMMENT '状态：1处理中 2成功 3失败',
    `create_at`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间',
    `update_at`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_create_at` (`user_id`, `create_at`),
    KEY `idx_user_from_currency` (`user_id`, `from_currency`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='兑换订单表';
```

**字段总数：15**

---

### 表 9：audit_requirement（稽核流水要求表）

**级别：Should** | **所属：钱包模块（我方）** | **优先级：中**

**这张表做什么**：记录每一笔带稽核要求的操作——充值奖金需要完成N倍流水才能提现、兑换赠送需要完成N倍流水才能提现。跟踪"需完成流水"和"已完成流水"。

**产品需求来源**：需求 3.6（奖金流水要求）、3.7（兑换赠送稽核）、7.5（稽核/流水机制）

**为什么是 Should 而不是 Must**：
- 稽核机制的完整实现需要消费（投注/购买）操作的配合
- 如果游戏/商城模块尚未就绪，稽核流水无法累计
- 可以先建表预留结构，但真正运转需要跨模块协作
- 不影响充值/提现/兑换的核心流程运转

**结论**：应尽早建表，但功能可分阶段实现。

```sql
CREATE TABLE `audit_requirement` (
    `id`                bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`           bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`     varchar(10)      NOT NULL                COMMENT '币种代码',
    `wallet_type`       tinyint unsigned NOT NULL                COMMENT '关联钱包类型（通常2=奖励）',
    `source_order_no`   varchar(50)      NOT NULL                COMMENT '来源订单号',
    `source_type`       varchar(30)      NOT NULL                COMMENT '来源类型：deposit_bonus/exchange_bonus/repair/manual_credit',
    `bonus_amt`         decimal(20,4)    NOT NULL                COMMENT '赠送/奖金金额',
    `audit_multiplier`  decimal(10,2)    NOT NULL                COMMENT '稽核倍数（如3.00=3倍）',
    `required_turnover` decimal(20,4)    NOT NULL                COMMENT '需完成流水 = 金额×倍数',
    `completed_turnover`decimal(20,4)    NOT NULL DEFAULT 0      COMMENT '已完成流水',
    `status`            tinyint unsigned NOT NULL DEFAULT 1      COMMENT '状态：1进行中 2已完成 3已过期',
    `create_at`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间',
    `update_at`         bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间',
    PRIMARY KEY (`id`),
    KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`),
    KEY `idx_source_order` (`source_order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='稽核流水要求表';
```

**字段总数：13**

---

## 五、Could 级 — 可延后但提升健壮性的表

---

### 表 10：reconciliation_diff（对账差异记录表）

**级别：Could** | **所属：钱包模块（我方）** | **优先级：低**

**这张表做什么**：每日对账任务执行时，如果发现 wallet_account 的余额与 wallet_transaction 的流水合计不一致，记录差异信息。供运营人员排查。

**产品需求来源**：需求 3.10（对账一致性校验机制）— 需求说"必须有校验机制"但未定义具体页面

**为什么是 Could**：
- 对账任务可以先只写告警日志（log.Error），不需要独立表
- 差异记录可以暂时存在日志系统或消息队列中
- 后期如需页面化展示再建表不迟

```sql
CREATE TABLE `reconciliation_diff` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`         bigint unsigned  NOT NULL                COMMENT '用户ID',
    `currency_code`   varchar(10)      NOT NULL                COMMENT '币种代码',
    `wallet_type`     tinyint unsigned NOT NULL                COMMENT '钱包类型',
    `account_amt`     decimal(20,4)    NOT NULL                COMMENT '账户表记录的余额',
    `computed_amt`    decimal(20,4)    NOT NULL                COMMENT '流水推算的余额',
    `diff_amt`        decimal(20,4)    NOT NULL                COMMENT '差异金额',
    `diff_type`       varchar(30)      NOT NULL                COMMENT '差异类型：balance_mismatch/flow_discontinuity',
    `status`          tinyint unsigned NOT NULL DEFAULT 1      COMMENT '状态：1待处理 2已处理 3已忽略',
    `handle_remark`   varchar(255)     NOT NULL DEFAULT ''     COMMENT '处理说明',
    `recon_date`      varchar(10)      NOT NULL                COMMENT '对账日期（yyyy-MM-dd）',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间',
    PRIMARY KEY (`id`),
    KEY `idx_recon_date` (`recon_date`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='对账差异记录表';
```

**字段总数：13**

---

### 表 11：rate_api_source（三方汇率API源配置表）

**级别：Could** | **所属：币种配置模块（我方）** | **优先级：低**

**这张表做什么**：存储三方汇率API的配置信息——API地址、密钥、权重、状态等。汇率定时任务从此表读取API配置。

**产品需求来源**：需求 4.5.2（从3个三方API取平均值）

**为什么是 Could**：
- 3个API的配置可以直接写在 ETCD 配置中心（现有工程已有 ETCD 配置模式）
- 短期内API数量不多（3个），配置变更频率极低
- 用 ETCD 管理比建表更轻量，且支持热更新
- 后期如果需要更灵活的API管理（动态增减、权重调整、监控成功率），再建表不迟

```sql
CREATE TABLE `rate_api_source` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `api_name`        varchar(50)      NOT NULL                COMMENT 'API名称',
    `api_url`         varchar(500)     NOT NULL                COMMENT 'API请求地址',
    `api_key`         varchar(200)     NOT NULL DEFAULT ''     COMMENT 'API密钥',
    `weight`          int unsigned     NOT NULL DEFAULT 1      COMMENT '权重（取平均值时的权重）',
    `timeout_ms`      int unsigned     NOT NULL DEFAULT 5000   COMMENT '超时毫秒',
    `status`          tinyint unsigned NOT NULL DEFAULT 1      COMMENT '状态：0禁用 1启用',
    `last_success_at` bigint unsigned  NOT NULL DEFAULT 0      COMMENT '最近成功时间',
    `fail_count`      int unsigned     NOT NULL DEFAULT 0      COMMENT '连续失败次数',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '创建时间',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0      COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_api_name` (`api_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='三方汇率API源配置表';
```

**字段总数：11**

---

## 六、财务管理模块表结构（他方负责，我方需了解）

> 以下表由财务管理模块负责建设和维护，我方需要了解其结构以便联调对齐。此处列出的是根据产品需求推导的表结构方向，具体字段由财务方决定。

---

### 表 F1：payment_channel（支付通道配置表）

**级别：Must（财务方）**

**做什么**：存储代收通道（充值用）和代付通道（提现出款用）的配置信息——通道名称、费率、限额、权重、密钥等。

**产品来源**：需求 5.2（通道配置）

```
推导字段（参考）：
  id                 主键
  channel_name       通道名称
  channel_type       通道类型：1代收(充值) 2代付(提现)
  fee_rate           费率%（1位小数）
  min_amount         单笔最低金额
  max_amount         单笔最高金额
  weight             权重（整数，轮询算法用）
  warning_amount     预警金额
  status             状态：0禁用 1启用
  api_url            请求地址
  merchant_id        商户ID
  api_key            密钥
  app_key            AppKey
  pay_code           支付编码
  public_key         公钥
  private_key        私钥
  currency_code      支持的币种
  success_rate       成功率（轮询算法用）
  avg_time           平均处理时间（轮询算法用）
  create_by/at       审计字段
  update_by/at       审计字段
```

**与我方的关系**：我方创建充值订单后调财务 RPC，财务用此表做通道轮询匹配。

---

### 表 F2：deposit_method（充值方式配置表）

**级别：Must（财务方）**

**做什么**：配置每个币种可用的充值方式——方式名称、图标、快捷档位、金额范围、关联通道。

**产品来源**：需求 5.3（支付配置-充值方式）

```
推导字段（参考）：
  id                 主键
  currency_code      币种代码
  method_name        充值方式名称（同币种唯一）
  icon_url           支付图标（SVG/PNG <=10KB）
  amount_presets     快捷金额档位（JSON数组）
  min_amount         最低充值金额
  max_amount         最高充值金额
  sort_value         排序
  status             状态
  is_recommended     是否推荐
  create_by/at       审计字段
  update_by/at       审计字段
  delete_at          软删除
```

**与我方的关系**：C端充值页面调用财务 RPC 获取充值方式列表，展示给用户。

---

### 表 F3：deposit_method_channel（充值方式-通道关联表）

**级别：Must（财务方）**

**做什么**：一个充值方式可以关联多个支付通道（多对多关系）。

```
推导字段（参考）：
  id                 主键
  method_id          充值方式ID
  channel_id         支付通道ID
  create_at          创建时间
```

---

### 表 F4：withdraw_method（提现方式配置表）

**级别：Must（财务方）**

**做什么**：配置每个币种可用的提现方式——比充值方式简单，无档位/范围/通道关联。

**产品来源**：需求 5.3（支付配置-提现方式）

```
推导字段（参考）：
  id                 主键
  currency_code      币种代码
  method_name        提现方式名称（同币种唯一）
  icon_url           支付图标
  status             状态
  create_by/at       审计字段
  update_by/at       审计字段
  delete_at          软删除
```

**与我方的关系**：C端提现页面调用财务 RPC 获取提现方式列表。

---

### 表 F5：channel_order（通道侧订单表）

**级别：Must（财务方）**

**做什么**：记录通道侧的订单信息——通道订单号、回调内容、回调时间、通道返回的支付状态。与我方的 deposit_order / withdraw_order 通过 order_no 关联。

```
推导字段（参考）：
  id                 主键
  platform_order_no  平台订单号（= 我方的 order_no）
  channel_order_no   通道侧订单号
  channel_id         通道ID
  order_type         订单类型：1充值 2提现
  request_body       发送给通道的请求内容
  callback_body      通道回调的原始内容
  callback_time      回调时间
  channel_status     通道返回的状态
  create_at          创建时间
  update_at          更新时间
```

**与我方的关系**：
- 充值回调链路：三方→财务（写此表+更新状态）→调我方 CreditWallet RPC
- 提现出款链路：财务→通道（写此表）→通道回调→财务→调我方 DeductFrozenAmount RPC

---

### 表 F6：manual_adjustment（人工修正记录表）

**级别：Should（财务方）**

**做什么**：记录充值补单（B+16）、人工加款（A+16）、人工减款（M+16）的操作记录——申请人、审核人、操作金额、目标钱包、稽核参数。

**产品来源**：需求 5.6（人工修正三种操作）

```
推导字段（参考）：
  id                 主键
  order_no           操作订单号（B/A/M + 16位）
  adjust_type        类型：1补单 2加款 3减款
  user_id            目标用户
  currency_code      币种
  原始订单号          补单时关联的充值订单号
  center_amt         中心钱包操作金额
  reward_amt         奖励钱包操作金额
  anchor_amt         主播钱包操作金额
  agent_amt          代理钱包操作金额
  center_audit_mul   中心钱包稽核倍数
  reward_audit_mul   奖励钱包稽核倍数
  reason             操作原因
  status             状态：1待审核 2审核拒绝 3操作成功
  apply_by           申请人
  review_by          审核人
  review_at          审核时间
  review_remark      审核备注
  create_at          创建时间
  update_at          更新时间
```

**与我方的关系**：审核通过后，财务调我方 CreditWallet/DebitWallet RPC，带上 order_no 和具体金额。

---

### 表 F7：polling_rule_config（轮询规则配置表）

**级别：Should（财务方）**

**做什么**：存储通道轮询算法的参数配置——成功率系数权重、时间系数权重、预警系数权重。

**产品来源**：需求 5.2（轮询规则）

```
推导字段（参考）：
  id                    主键
  rule_type             规则类型：1代收 2代付
  success_rate_weight   成功率系数权重
  time_weight           时间系数权重
  warning_weight        预警系数权重
  base_weight           基础权重系数
  status                状态
  create_by/at          审计字段
  update_by/at          审计字段
```

---

## 七、表间关系完整图谱

```
我方负责的表（实线框）                        他方负责的表（虚线框）

┌─────────────────────┐                    ┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐
│  currency_config    │                    ╎  payment_channel    ╎
│  [Must] 币种配置     │                    ╎  [Must] 通道配置     ╎
│  UK: currency_code  │                    └╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘
└─────┬───────────────┘                              │
      │ currency_code                                │ channel_id
      │                                              ▼
      ├──────────────────────┐              ┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐
      │                      │              ╎ deposit_method      ╎
      ▼                      ▼              ╎ [Must] 充值方式      ╎
┌──────────────┐    ┌──────────────┐        └╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘
│exchange_rate │    │wallet_account│                  │ RPC
│  _log        │    │ [Must] 账户  │                  ▼
│ 汇率日志      │    │UK:(uid,cc,wt)│        ┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐
└──────────────┘    └──────┬───────┘        ╎ channel_order       ╎
                           │                ╎ [Must] 通道订单      ╎
              ┌────────────┼────────┐       └╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘
              │            │        │                │ order_no
              ▼            ▼        ▼                ▼
     ┌──────────────┐ ┌────────┐ ┌────────┐  ┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐
     │wallet_       │ │deposit │ │withdraw│  ╎ manual_adjustment   ╎
     │transaction   │ │_order  │ │_order  │  ╎ [Should] 人工修正    ╎
     │[Must] 流水    │ │[Must]  │ │[Must]  │  └╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘
     │UK:(on,op,wt) │ │充值订单 │ │提现订单 │         │ RPC
     └──────────────┘ └────────┘ └───┬────┘         │ Credit/Debit
                                     │              ▼
                           ┌─────────┴──────┐  ┌──────────┐
                           │                │  │wallet_   │
                           ▼                ▼  │account   │
                    ┌──────────────┐ ┌──────────────┐     │
                    │withdraw_     │ │exchange_     │     │
                    │account       │ │order         │◄────┘
                    │[Must] 提现户  │ │[Should] 兑换 │
                    └──────────────┘ └──────────────┘

                    ┌──────────────┐  ┌──────────────────┐
                    │audit_        │  │reconciliation_   │
                    │requirement   │  │diff              │
                    │[Should] 稽核 │  │[Could] 对账差异   │
                    └──────────────┘  └──────────────────┘

关系说明（全部为逻辑外键，不建物理外键约束）：

  wallet_account.currency_code   → currency_config.currency_code
  wallet_transaction.order_no    → deposit_order / withdraw_order / exchange_order
  deposit_order.order_no         → channel_order.platform_order_no（跨服务）
  withdraw_order.order_no        → channel_order.platform_order_no（跨服务）
  manual_adjustment 审核通过      → RPC → wallet_account 余额变动
  exchange_rate_log.currency_code → currency_config.currency_code
```

---

## 八、按建设顺序排列的实施路径

```
Phase 1 — 地基（首先建立，其他一切依赖它们）
═══════════════════════════════════════════
  1. currency_config        — 所有操作都依赖币种数据
  2. wallet_account         — 余额存储的容器
  3. wallet_transaction     — 余额变动的账本

Phase 2 — 核心业务（建立在地基之上）
═══════════════════════════════════════════
  4. deposit_order          — 充值流程
  5. withdraw_order         — 提现流程
  6. withdraw_account       — 提现账户管理

Phase 3 — 业务完善
═══════════════════════════════════════════
  7. exchange_rate_log      — 汇率审计
  8. exchange_order         — 兑换订单
  9. audit_requirement      — 稽核流水跟踪

Phase 4 — 锦上添花（按需建立）
═══════════════════════════════════════════
  10. reconciliation_diff   — 对账差异记录
  11. rate_api_source       — API配置（或用ETCD替代）
```

---

## 九、字段数量与数据量预估汇总

| # | 表名 | 级别 | 字段数 | 预估年数据量 | 分表需求 |
|---|------|------|--------|------------|---------|
| 1 | currency_config | Must | 28 | <100 | 不需要 |
| 2 | exchange_rate_log | Must | 12 | ~50万 | 按时间归档 |
| 3 | wallet_account | Must | 9 | ~100万 | 不需要 |
| 4 | wallet_transaction | Must | 14 | **5000万+** | **后续可能** |
| 5 | deposit_order | Must | 27 | ~500万 | 中期可考虑 |
| 6 | withdraw_order | Must | 31 | ~200万 | 暂不需要 |
| 7 | withdraw_account | Must | 13 | ~50万 | 不需要 |
| 8 | exchange_order | Should | 15 | ~100万 | 不需要 |
| 9 | audit_requirement | Should | 13 | ~200万 | 暂不需要 |
| 10 | reconciliation_diff | Could | 13 | ~1万 | 不需要 |
| 11 | rate_api_source | Could | 11 | <10 | 不需要 |

**我方表字段总计：186个字段**（9张核心表+2张可选表）

---

## 十、与财务模块需要对齐的数据约定

以下约定直接影响两个服务之间的数据互通，需要在联调前确认：

| 序号 | 约定项 | 影响范围 | 当前方案 | 需确认 |
|------|--------|---------|---------|-------|
| 1 | 金额精度 | 所有金额字段 | DECIMAL(20,4) | 财务方是否一致 |
| 2 | 钱包类型枚举 | CreditWallet 等 RPC 入参 | 1中心 2奖励 3主播 4代理 5场馆 | 双方对齐 |
| 3 | 订单号格式 | 幂等键组成 | C/T/B/A/M + 16位 | 已明确 |
| 4 | 时间戳精度 | 所有时间字段 | 毫秒级 bigint | 财务方是否一致 |
| 5 | 币种代码长度 | varchar(10) | BSB/VND/IDR/THB/USDT | 是否需要更长 |
| 6 | 汇率快照归属 | 订单中的 rate_snapshot | 我方创建订单时写入 | 还是财务方提供 |
| 7 | 手续费计算归属 | fee_amt / fee_rate | 提现由财务计算回传 | 充值手续费由谁算 |
| 8 | 基准币种金额归属 | base_xxx_amt | 我方创建订单时计算 | 还是统计时再算 |

---

## 十一、总结

### 11.1 分级汇总

| 级别 | 我方表数 | 他方表数 | 说明 |
|------|---------|---------|------|
| **Must** | 7张 | 4-5张 | 没有就不能运转 |
| **Should** | 2张 | 2张 | 业务完整性需要 |
| **Could** | 2张 | 1张 | 锦上添花可延后 |
| **合计** | **11张** | **7张** | 整个资金域约18张表 |

### 11.2 我方 Must 级 7 张表速查

| 表名 | 字段数 | 一句话用途 |
|------|--------|----------|
| currency_config | 28 | 全系统的币种基础数据源 |
| exchange_rate_log | 12 | 汇率变更的不可变审计日志 |
| wallet_account | 9 | 用户的钱在这里（可用+冻结） |
| wallet_transaction | 14 | 钱怎么变的全在这里（不可变账本） |
| deposit_order | 27 | 充值订单全生命周期 |
| withdraw_order | 31 | 提现订单（7状态+锁定+审核） |
| withdraw_account | 13 | 用户的银行卡/钱包账户信息 |

### 11.3 一句话总结

**我方 7 张 Must 表 + 2 张 Should 表 = 9 张核心表构成 ser-wallet 的完整数据骨架；他方约 7 张表支撑财务管理功能；两边通过 order_no 和 RPC 合同关联。表结构设计遵循现有工程规范（bigint主键/毫秒时间戳/snake_case/InnoDB），金额统一 DECIMAL(20,4)，汇率统一 DECIMAL(20,8)，流水表只追加不修改。先建地基（币种+账户+流水），再建业务（充值/提现/兑换），最后补辅助（稽核/对账），与财务模块的联调核心是 6 个 RPC 接口 + 订单号关联。**
