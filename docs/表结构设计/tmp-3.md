# ser-wallet 表结构设计（完整分析）

> 设计时间：2026-02-26
> 设计依据：产品需求文档 + 产品原型 + 架构设计方案 + 现有工程实际建表规范（ser-item/ser-buser/ser-app/ser-s3 的 DDL）
> 设计范围：钱包模块（主责）+ 币种配置模块（主责）+ 与财务模块协作相关的表
> 分析视角：必须 / 应该 / 可选 三级分类
> DDL 规范对齐：现有工程使用 `bigint unsigned`、`tinyint`、`utf8mb4_bin` 等规范

---

## 一、表设计总览与优先级分类

### 1.1 三级分类标准

```
【必须】没有这张表，模块的核心功能无法运行
  → 不建此表 = 模块无法上线

【应该】满足产品需求的重要功能，但核心链路不完全依赖它
  → 不建此表 = 功能缺失或体验降级，但核心交易流程仍可运行

【可选】优化性/扩展性/运维支撑性的表
  → 不建此表 = 不影响业务功能，但缺少可观测性或未来扩展能力
```

### 1.2 表清单总览

```
┌────┬──────────────────────────┬────────┬──────────────────────────────────────┐
│ 序号│ 表名                      │ 优先级  │ 一句话说明                             │
├────┼──────────────────────────┼────────┼──────────────────────────────────────┤
│  1 │ currency_config           │ 必须   │ 币种配置，所有模块的基础数据源             │
│  2 │ user_wallet               │ 必须   │ 用户钱包账户，余额管理的核心               │
│  3 │ wallet_flow               │ 必须   │ 资金流水，对账/审计/幂等的唯一凭证          │
│  4 │ wallet_freeze_record      │ 必须   │ 冻结记录，提现冻结→确认/解冻的状态机载体     │
│  5 │ deposit_order             │ 必须   │ 充值订单，C端充值记录和财务回调的载体        │
│  6 │ withdraw_order            │ 必须   │ 提现订单，C端提现全生命周期管理             │
│  7 │ withdraw_account          │ 必须   │ 提现账户，用户绑定的银行卡/电子钱包信息      │
│  8 │ exchange_record           │ 必须   │ 兑换记录，法币→BSB兑换凭证                 │
│  9 │ exchange_rate_log         │ 应该   │ 汇率变更日志，B端运营审计需要               │
│ 10 │ manual_adjustment_order   │ 应该   │ 人工修正订单，补单/加款/减款统一管理         │
│ 11 │ wallet_config             │ 应该   │ 钱包业务配置，兑换赠送/提现限额等运行时配置   │
│ 12 │ deposit_bonus_snapshot    │ 可选   │ 充值奖金快照，关联活动奖金的详细追溯         │
│ 13 │ wallet_daily_snapshot     │ 可选   │ 每日钱包快照，运营统计和对账辅助             │
│ 14 │ rate_provider_log         │ 可选   │ 三方汇率源日志，汇率来源追溯               │
└────┴──────────────────────────┴────────┴──────────────────────────────────────┘
```

---

## 二、DDL 规范说明（对齐现有工程）

在进入具体表设计前，明确本项目的建表规范。以下所有规范从现有工程的实际 DDL 中提取（ser-item `item`表、ser-buser `sys_user`表、ser-app `banner`表等）。

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ 规范项                现有工程实际做法                    ser-wallet 遵循     │
├──────────────────────────────────────────────────────────────────────────────┤
│ 主键ID               bigint unsigned NOT NULL AUTO_INC   同规范             │
│ 字符串               varchar(N), NOT NULL 或 DEFAULT ''  同规范             │
│ 状态/枚举             tinyint 或 int                     使用 tinyint       │
│ 布尔值               tinyint (0/1)                       同规范             │
│ 时间戳               bigint unsigned (毫秒或秒)           统一用毫秒         │
│ 金额(价格)           int unsigned (ser-item base_price)   bigint unsigned    │
│                      钱包金额范围更大，需 bigint                             │
│ 可空字段             DEFAULT NULL                        同规范             │
│ 非空字段             NOT NULL + DEFAULT 或 NOT NULL      同规范             │
│ 字符集               utf8mb4                             同规范             │
│ 排序规则             utf8mb4_bin                         同规范             │
│ 引擎                 InnoDB                              同规范             │
│ 软删除               delete_at bigint unsigned DEFAULT 0  同规范             │
│ 审计字段             create_by/update_by/create_at/...    同规范             │
│ 索引命名             uk_xxx / idx_xxx                     同规范             │
│ 注释                 COMMENT '中文说明'                   同规范             │
│ 外键约束             不使用物理外键                        同规范             │
└──────────────────────────────────────────────────────────────────────────────┘

金额存储特别说明：
  现有工程 ser-item 的 base_price 用 int unsigned（int32 范围 ~21亿），
  对于道具价格足够了。但钱包模块的金额可能很大：
    VND 越南盾：100,000,000 VND = 1亿（已接近 int 上限）
    USDT 8位小数：100 USDT = 10,000,000,000（100亿，溢出 int）
  所以钱包金额字段统一使用 bigint unsigned。

时间戳统一说明：
  现有工程不一致：ser-item 用秒级，ser-app(banner) 用毫秒级。
  ser-wallet 统一使用毫秒级时间戳（bigint unsigned），原因：
    1. 金融系统需要更高精度的时间区分
    2. 前端 JS 的 Date.now() 返回毫秒，减少转换
    3. ser-app 新表已经在用毫秒，趋势统一
```

---

## 三、【必须】核心表详细设计

以下 8 张表是 ser-wallet 无法绕过的。没有它们，钱包模块的任何核心功能都无法实现。

---

### 表1：currency_config — 币种配置表

**为什么是【必须】：**

```
产品依据：
  - 币种配置需求3.3 列出了后台配置页面的20个字段
  - 钱包需求4.1："钱包按币种维度隔离，每个币种一套独立余额"
  - 没有币种配置 → 不知道平台支持什么币种 → 钱包首页无法渲染
  - 没有汇率数据 → BSB充值/提现/兑换无法计算金额
  - 所有其他表都引用 currency_code，此表是基础数据源

功能依赖：
  C端：钱包首页(读币种列表) / 充值页(读入款汇率) / 兑换页(读平台汇率) / 提现页(读出款汇率)
  B端：币种配置页(CRUD) / 汇率监控
  定时任务：汇率刷新(读写)
  其他模块RPC：投注模块(汇率换算) / 财务模块(汇率数据)
```

**DDL：**

```sql
CREATE TABLE `currency_config` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `currency_code`   varchar(16)      NOT NULL COMMENT '币种代码，全局唯一，如 VND/IDR/THB/BSB/USDT',
    `currency_name`   varchar(64)      NOT NULL COMMENT '币种名称，如 越南盾/印尼盾/泰铢',
    `currency_type`   tinyint unsigned NOT NULL COMMENT '币种类型：1法定货币 2加密货币 3平台货币',
    `country`         varchar(64)      DEFAULT NULL COMMENT '币种国家，加密/平台币为"全球"',
    `icon_url`        varchar(512)     DEFAULT NULL COMMENT '币种图标URL，SVG/PNG，≤5k',
    `symbol`          varchar(16)      NOT NULL COMMENT '货币符号，如 d/Rp/B/₮/USDT',
    `thousand_sep`    varchar(4)       NOT NULL COMMENT '千分位符号，如 . 或 ,',
    `decimal_sep`     varchar(4)       NOT NULL COMMENT '小数点符号，如 .',
    `decimal_digits`  tinyint unsigned NOT NULL COMMENT '小数位数：VND/IDR=0, THB/BSB=2, USDT=8',
    `realtime_rate`   bigint unsigned  NOT NULL DEFAULT 0 COMMENT '实时汇率（×10^8整数），3家三方API取均值',
    `float_rate`      bigint unsigned  NOT NULL DEFAULT 0 COMMENT '汇率浮动比例（×10^4整数），如2.50%存25000',
    `deposit_rate`    bigint unsigned  NOT NULL DEFAULT 0 COMMENT '入款汇率（×10^8整数）= 实时×(1+浮动%)',
    `withdraw_rate`   bigint unsigned  NOT NULL DEFAULT 0 COMMENT '出款汇率（×10^8整数）= 实时×(1-浮动%)',
    `platform_rate`   bigint unsigned  NOT NULL DEFAULT 0 COMMENT '平台汇率（×10^8整数），偏差超阈值时更新',
    `deviation`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '偏差值（×10^4整数），= |实时-平台|/实时×100',
    `threshold`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '阈值（×10^4整数），偏差>=阈值触发更新',
    `is_base`         tinyint unsigned NOT NULL DEFAULT 0 COMMENT '是否基准币种：0否 1是，全局唯一且不可更改',
    `status`          tinyint unsigned NOT NULL DEFAULT 1 COMMENT '状态：0禁用 1启用，基准币种不可禁用',
    `create_by`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建人ID',
    `create_by_name`  varchar(64)      NOT NULL DEFAULT '' COMMENT '创建人姓名',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    `update_by`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新人ID',
    `update_by_name`  varchar(64)      NOT NULL DEFAULT '' COMMENT '更新人姓名',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新时间（毫秒时间戳）',
    `delete_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '删除时间（毫秒时间戳，0=未删除）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_currency_code` (`currency_code`),
    KEY `idx_status` (`status`),
    KEY `idx_currency_type` (`currency_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='币种配置表';
```

**字段与产品需求的映射：**

| 产品需求原文 | 对应字段 | 说明 |
|---|---|---|
| 3.3字段1"序号" | id | 自增主键即序号 |
| 3.3字段2"币种代码-唯一" | currency_code + UK约束 | 全局唯一 |
| 3.3字段3"币种名称" | currency_name | |
| 3.3字段4"币种类型-法定/加密/平台" | currency_type tinyint | 1/2/3枚举 |
| 3.3字段5"币种国家" | country | 加密币为"全球" |
| 3.3字段6"币种图标-SVG/PNG≤5k" | icon_url | S3 URL |
| 3.3字段7"货币符号" | symbol | |
| 3.3字段8"千分位符号" | thousand_sep | VND用`.` THB用`,` |
| 3.3字段9"小数点符号" | decimal_sep | |
| 3.3字段10"实时汇率-三方提供" | realtime_rate | ×10^8整数存储 |
| 3.3字段11"汇率浮动-2位小数%" | float_rate | ×10^4整数存储 |
| 3.3字段12"入款汇率" | deposit_rate | = 平台×(1+浮动%) |
| 3.3字段13"出款汇率" | withdraw_rate | = 平台×(1-浮动%) |
| 3.3字段14"平台汇率" | platform_rate | 偏差触发更新 |
| 3.3字段15"偏差值" | deviation | 计算值 |
| 3.3字段16"阈值" | threshold | 控制更新触发 |
| 3.3字段17"状态" | status | 基准币种不可禁用 |
| 3.3字段18"操作人" | update_by_name | |
| 3.3字段19"操作时间" | update_at | |
| 3.3字段20"操作-编辑" | — | 前端行为，无需字段 |
| 3.2"基准币种只可设1次" | is_base | 应用层校验唯一 |
| 4.1"小数位规则" | decimal_digits | 0/2/8 |

**汇率精度方案：**

```
所有汇率字段统一 ×10^8 整数存储（匹配 USDT 8位小数精度）。

存储示例：
  1 BSB = 0.10000000 USDT → realtime_rate = 10000000
  1 VND = 0.00003820 USDT → realtime_rate = 3820
  1 THB = 0.02850000 USDT → realtime_rate = 2850000

浮动/偏差/阈值 ×10^4 整数存储（百分比最多两位小数）：
  浮动 2.50% → float_rate = 25000
  阈值 5.00% → threshold = 50000

入款汇率计算公式（来自需求4.2）：
  deposit_rate = realtime_rate + realtime_rate × float_rate / 1000000
出款汇率计算公式：
  withdraw_rate = realtime_rate - realtime_rate × float_rate / 1000000
```

---

### 表2：user_wallet — 用户钱包表

**为什么是【必须】：**

```
产品依据：
  - 钱包需求4.1："钱包按币种维度隔离，每个币种一套独立余额"
  - 币种配置2.1资金钱包定义："中心钱包/奖励钱包/主播钱包/代理钱包/场馆钱包"
  - "资金钱包余额 = 中心钱包 + 奖励钱包"
  - 没有此表 → 不知道用户有多少钱 → 钱包的核心意义不存在

功能依赖：
  所有涉及余额的操作都读写此表：
  充值上账/投注扣款/提现冻结/兑换/返奖/人工修正/余额查询/...
  这是整个钱包系统的心脏。
```

**DDL：**

```sql
CREATE TABLE `user_wallet` (
    `id`                bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`           bigint unsigned  NOT NULL COMMENT '用户ID',
    `currency_code`     varchar(16)      NOT NULL COMMENT '币种代码',
    `wallet_type`       tinyint unsigned NOT NULL COMMENT '钱包类型：1中心 2奖励 3主播 4代理(预留) 5场馆(预留)',
    `available_balance` bigint unsigned  NOT NULL DEFAULT 0 COMMENT '可用余额（最小货币单位整数）',
    `frozen_balance`    bigint unsigned  NOT NULL DEFAULT 0 COMMENT '冻结余额（最小货币单位整数）',
    `status`            tinyint unsigned NOT NULL DEFAULT 1 COMMENT '状态：0冻结 1正常',
    `create_at`         bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    `update_at`         bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新时间（毫秒时间戳）',
    `delete_at`         bigint unsigned  NOT NULL DEFAULT 0 COMMENT '删除时间（毫秒时间戳，0=未删除）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_currency_type` (`user_id`, `currency_code`, `wallet_type`),
    KEY `idx_user_id` (`user_id`),
    KEY `idx_currency_code` (`currency_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='用户钱包表';
```

**核心设计说明：**

```
账户模型：一个用户 × 一个币种 × 一个钱包类型 = 一个钱包账户
  联合唯一索引 uk_user_currency_type 是此表的核心约束。

  示例用户 10001：
    (10001, 'VND', 1)  → VND中心钱包  available=500000  frozen=0
    (10001, 'VND', 2)  → VND奖励钱包  available=50000   frozen=0
    (10001, 'BSB', 1)  → BSB中心钱包  available=100000  frozen=20000
    (10001, 'BSB', 2)  → BSB奖励钱包  available=10000   frozen=0

金额存储：
  bigint unsigned 最小货币单位整数（和 currency_config.decimal_digits 配合）
  VND(0位小数)：500,000 VND → 存 500000
  THB(2位小数)：1,500.50 THB → 存 150050
  USDT(8位小数)：100.50 USDT → 存 10050000000

余额规则映射：
  "资金钱包余额 = 中心 + 奖励"
    → SELECT SUM(available_balance) WHERE user_id=? AND currency_code=? AND wallet_type IN (1,2)

  "消费时优先扣中心再扣奖励"
    → Service 层：先 debit wallet_type=1，不够部分再 debit wallet_type=2

  "如用户下没有该币种钱包，系统需自动创建"（财务人工加款需求）
    → creditCore 中 RowsAffected=0 时自动 INSERT

并发安全（来自架构设计第四节）：
  扣款：UPDATE SET available_balance = available_balance - ? WHERE available_balance >= ?
  上账：UPDATE SET available_balance = available_balance + ? (无条件)
  冻结：UPDATE SET available_balance -= ?, frozen_balance += ? WHERE available_balance >= ?
  全部用 gorm.Expr 条件更新，数据库行级保证原子性。

  为什么 bigint unsigned 不用 signed：
    余额不应该为负数。unsigned 提供数据库层面的兜底（虽然 WHERE >= 已经保证了，双重保障）。
```

---

### 表3：wallet_flow — 钱包流水表

**为什么是【必须】：**

```
产品依据：
  - 钱包需求8："记录模块：提现/充值/兑换/奖励四Tab"
  - 需求9："对账一致性：明确账务流水与账户余额的强一致性校验机制"
  - 需求9："幂等性：充值回调必须做幂等处理"
  - 没有此表 → 不知道余额怎么来的怎么走的 → 无法对账 → 无法审计 → 无法幂等

功能依赖：
  C端：记录列表（四个Tab的数据源）、记录详情
  核心：幂等保证（order_no+flow_type唯一索引）
  运维：对账（SUM(amount) = 当前余额）、排查（每笔变更可追溯）
  B端：用户钱包详情查看
```

**DDL：**

```sql
CREATE TABLE `wallet_flow` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `flow_no`         bigint unsigned  NOT NULL COMMENT '流水编号（snowflake分布式ID）',
    `user_id`         bigint unsigned  NOT NULL COMMENT '用户ID',
    `currency_code`   varchar(16)      NOT NULL COMMENT '币种代码',
    `wallet_type`     tinyint unsigned NOT NULL COMMENT '钱包类型：1中心 2奖励 3主播 4代理 5场馆',
    `flow_type`       tinyint unsigned NOT NULL COMMENT '流水类型：1充值 2投注扣款 3投注返奖 4提现冻结 5提现确认 6提现解冻 7兑换扣减 8兑换到账 9兑换赠送 10补单 11加款 12减款 13活动奖励 14主播收益 15充值奖金 16转入 17转出',
    `amount`          bigint           NOT NULL COMMENT '变更金额（正数=入账，负数=出账，最小单位整数）',
    `balance_before`  bigint unsigned  NOT NULL COMMENT '变更前可用余额',
    `balance_after`   bigint unsigned  NOT NULL COMMENT '变更后可用余额',
    `order_no`        varchar(64)      NOT NULL COMMENT '关联业务订单号（幂等键，调用方传入）',
    `remark`          varchar(512)     DEFAULT NULL COMMENT '备注说明',
    `extra`           text             DEFAULT NULL COMMENT '扩展信息（JSON格式，不同业务场景附加数据）',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_flow_no` (`flow_no`),
    UNIQUE KEY `uk_order_flow_type` (`order_no`, `flow_type`),
    KEY `idx_user_currency_time` (`user_id`, `currency_code`, `create_at` DESC),
    KEY `idx_user_flow_type` (`user_id`, `flow_type`),
    KEY `idx_create_at` (`create_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='钱包流水表';
```

**核心设计说明：**

```
幂等核心（架构设计第五节）：
  uk_order_flow_type (order_no, flow_type) 是幂等的数据库层保证。

  为什么加 flow_type：同一笔提现的 order_no 会产生多条不同类型流水：
    order_no='W001', flow_type=4 (提现冻结)  → 用户申请提现时
    order_no='W001', flow_type=5 (提现确认)  → 出款成功后
  或者：
    order_no='W001', flow_type=4 (提现冻结)  → 用户申请提现时
    order_no='W001', flow_type=6 (提现解冻)  → 提现被驳回时
  flow_type 不同 = 不同操作，允许共存。
  flow_type 相同 = 重复请求，唯一索引冲突拦截。

amount 为什么用 signed bigint：
  入账为正数（+），出账为负数（-）。
  这是业界标准（Stripe、蚂蚁的 journal entry 都是正负表示方向）。
  好处：SUM(amount) 直接得到净变化。

balance_before / balance_after 的价值：
  对账：最后一条流水的 balance_after = user_wallet.available_balance
  排查：出了问题可以逐条回溯余额变化链
  审计：完整的变更上下文

流水类型与产品记录Tab的映射：
  提现 Tab → flow_type IN (4,5,6)
  充值 Tab → flow_type IN (1,10,15)
  兑换 Tab → flow_type IN (7,8,9)
  奖励 Tab → flow_type IN (3,13)

extra 字段用途示例：
  兑换：{"source_currency":"VND","rate":3820,"bonus_pct":10}
  充值：{"method":"USDT_TRC20","channel_no":"CH001"}
  人工加款：{"type":"运营补偿","operator":"admin","reason":"活动补偿"}

此表不做软删除。流水一旦写入永远不删除（金融审计要求）。
```

---

### 表4：wallet_freeze_record — 冻结记录表

**为什么是【必须】：**

```
产品依据：
  - 提现流程核心："冻结余额 → 审核 → 确认扣除/解冻退回"
  - 财务需求2.4提现审核流程涉及：待风控→待财务→待出款→出款成功/失败
  - 每个审核结果都对应冻结记录的状态变更
  - 没有此表 → 无法跟踪提现冻结状态 → 无法区分"确认"和"解冻" → 提现流程断裂

功能依赖：
  提现冻结(freezeCore) / 提现确认(confirmDebitCore) / 提现解冻(unfreezeCore)
  状态机转换的载体：Frozen→Confirmed 或 Frozen→Unfrozen
```

**DDL：**

```sql
CREATE TABLE `wallet_freeze_record` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`        varchar(64)      NOT NULL COMMENT '关联业务订单号（一笔提现对应一条冻结）',
    `user_id`         bigint unsigned  NOT NULL COMMENT '用户ID',
    `currency_code`   varchar(16)      NOT NULL COMMENT '币种代码',
    `wallet_type`     tinyint unsigned NOT NULL COMMENT '钱包类型',
    `amount`          bigint unsigned  NOT NULL COMMENT '冻结金额（最小单位整数）',
    `status`          tinyint unsigned NOT NULL DEFAULT 1 COMMENT '冻结状态：1冻结中 2已解冻(退回) 3已确认(扣除)',
    `freeze_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '冻结时间（毫秒时间戳）',
    `finish_at`       bigint unsigned  DEFAULT NULL COMMENT '完结时间（解冻或确认的时间）',
    `finish_remark`   varchar(512)     DEFAULT NULL COMMENT '完结备注（驳回原因等）',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_status` (`user_id`, `status`),
    KEY `idx_user_currency` (`user_id`, `currency_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='钱包冻结记录表';
```

**状态机说明：**

```
状态转换（来自架构设计第六节）：

  [1冻结中] ──解冻──→ [2已解冻]  ← 提现被驳回 / 出款失败
  [1冻结中] ──确认──→ [3已确认]  ← 出款成功

  互斥约束：通过 WHERE status=1 条件更新实现。
    UPDATE SET status=3 WHERE order_no=? AND status=1
    RowsAffected=0 说明已被另一个操作处理（解冻或确认过了）。
  终态不可逆：已解冻/已确认后不可再变。

此表不做软删除。冻结记录是资金审计凭证。
```

---

### 表5：deposit_order — 充值订单表

**为什么是【必须】：**

```
产品依据：
  - 钱包需求5.1-5.6：充值全流程（选方式→创建订单→支付→回调上账）
  - 钱包需求5.5重复转账检测需要查询同用户+同币种+同金额的"待支付/进行中"订单
  - 钱包需求8.3充值记录Tab列表/详情
  - 没有此表 → 充值订单无处记录 → 重复检测无法实现 → 充值记录页无数据

功能依赖：
  C端：创建充值订单 / 重复转账检测 / 充值记录列表+详情
  回调：财务模块通知上账时关联此订单号
```

**DDL：**

```sql
CREATE TABLE `deposit_order` (
    `id`                 bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`           varchar(64)      NOT NULL COMMENT '平台订单号（snowflake）',
    `user_id`            bigint unsigned  NOT NULL COMMENT '用户ID',
    `currency_code`      varchar(16)      NOT NULL COMMENT '币种代码',
    `order_amount`       bigint unsigned  NOT NULL COMMENT '订单金额（钱包币种最小单位整数）',
    `deposit_method`     varchar(32)      NOT NULL COMMENT '充值方式：USDT_TRC20/USDT_ERC20/BANK_VN/BANK_ID/BANK_TH/QRIS/PROMPTPAY等',
    `status`             tinyint unsigned NOT NULL DEFAULT 1 COMMENT '状态：1待支付 2进行中 3成功 4失败 5失效',
    `channel_order_no`   varchar(128)     DEFAULT NULL COMMENT '通道订单号（三方支付通道返回）',
    `rate_snapshot`      bigint unsigned  DEFAULT NULL COMMENT '入款汇率快照（×10^8，BSB充值时锁定）',
    `pay_currency`       varchar(16)      DEFAULT NULL COMMENT '支付币种（BSB充值时=实际支付的币种）',
    `pay_amount`         bigint unsigned  DEFAULT NULL COMMENT '支付金额（支付币种最小单位整数）',
    `bonus_activity_id`  bigint unsigned  DEFAULT NULL COMMENT '关联奖金活动ID',
    `bonus_amount`       bigint unsigned  DEFAULT NULL COMMENT '奖金金额（最小单位整数）',
    `bonus_turnover`     tinyint unsigned DEFAULT NULL COMMENT '奖金流水倍数要求',
    `fail_reason`        varchar(256)     DEFAULT NULL COMMENT '失败原因：支付超时/支付异常/卡片问题',
    `paid_at`            bigint unsigned  DEFAULT NULL COMMENT '支付完成时间（毫秒时间戳）',
    `expire_at`          bigint unsigned  DEFAULT NULL COMMENT '订单过期时间（毫秒时间戳）',
    `create_at`          bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    `update_at`          bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新时间（毫秒时间戳）',
    `delete_at`          bigint unsigned  NOT NULL DEFAULT 0 COMMENT '删除时间（0=未删除）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`, `create_at` DESC),
    KEY `idx_user_time` (`user_id`, `create_at` DESC),
    KEY `idx_status` (`status`),
    KEY `idx_duplicate_check` (`user_id`, `currency_code`, `order_amount`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='充值订单表';
```

**字段说明：**

```
重复转账检测（需求5.5）：
  idx_duplicate_check 索引专为此场景设计：
  SELECT COUNT(*) FROM deposit_order
  WHERE user_id=? AND currency_code=? AND order_amount=? AND status IN (1,2)
  → 0笔通过，1-2笔提示，≥3笔拦截。

汇率快照（BSB充值场景）：
  原型5-2/5-3/5-4 展示 "1BSB=X法币"。
  用户下单时锁定当时的入款汇率存入 rate_snapshot，
  通道回调上账时用此快照汇率计算（避免汇率波动导致金额不一致）。

奖金字段（需求5.6）：
  bonus_activity_id / bonus_amount / bonus_turnover
  充值成功后：主金额→中心钱包，奖金金额→奖励钱包。
  bonus_turnover 记录"赠送部分需完成X倍流水后可提"的要求。

deposit_method 用 varchar 而非 tinyint：
  充值方式种类多且可能扩展（USDT_TRC20/USDT_ERC20/USDT_BEP20/各国银行/各种电子钱包）。
  字符串可读性更好，无需维护枚举映射。
```

---

### 表6：withdraw_order — 提现订单表

**为什么是【必须】：**

```
产品依据：
  - 钱包需求7.1-7.6：提现全流程
  - 财务需求2.4提现记录：7种状态流转
  - 钱包需求8.3提现记录Tab列表/详情
  - 没有此表 → 提现无处记录 → 无法跟踪审核状态 → 提现流程断裂

功能依赖：
  C端：创建提现订单 / 提现记录列表+详情
  财务：审核流程状态更新 / 出款回调
  核心：与 wallet_freeze_record 关联（冻结→确认/解冻）
```

**DDL：**

```sql
CREATE TABLE `withdraw_order` (
    `id`                   bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`             varchar(64)      NOT NULL COMMENT '平台订单号（snowflake）',
    `user_id`              bigint unsigned  NOT NULL COMMENT '用户ID',
    `currency_code`        varchar(16)      NOT NULL COMMENT '币种代码',
    `order_amount`         bigint unsigned  NOT NULL COMMENT '提现金额（钱包币种最小单位整数）',
    `withdraw_method`      tinyint unsigned NOT NULL COMMENT '提现方式：1银行卡 2电子钱包 3USDT',
    `status`               tinyint unsigned NOT NULL DEFAULT 1 COMMENT '状态：1待风控 2风控驳回 3待财务 4财务驳回 5待出款 6出款成功 7出款失败',
    `withdraw_account_id`  bigint unsigned  NOT NULL COMMENT '关联提现账户ID',
    `rate_snapshot`        bigint unsigned  DEFAULT NULL COMMENT '出款汇率快照（×10^8，BSB提现时锁定）',
    `pay_currency`         varchar(16)      DEFAULT NULL COMMENT '出款币种（BSB提现时的实际出款币种）',
    `pay_amount`           bigint unsigned  DEFAULT NULL COMMENT '出款金额（出款币种最小单位整数）',
    `fee_amount`           bigint unsigned  NOT NULL DEFAULT 0 COMMENT '手续费（最小单位整数）',
    `fee_rate`             bigint unsigned  DEFAULT NULL COMMENT '手续费率（×10^4整数）',
    `freeze_order_no`      varchar(64)      DEFAULT NULL COMMENT '冻结记录订单号（关联wallet_freeze_record）',
    `reject_reason`        varchar(512)     DEFAULT NULL COMMENT '驳回原因（风控/财务驳回时填写）',
    `audit_remark`         varchar(512)     DEFAULT NULL COMMENT '审核备注',
    `channel_order_no`     varchar(128)     DEFAULT NULL COMMENT '通道订单号（财务出款后回填）',
    `finish_at`            bigint unsigned  DEFAULT NULL COMMENT '完成时间（毫秒时间戳）',
    `create_at`            bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    `update_at`            bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新时间（毫秒时间戳）',
    `delete_at`            bigint unsigned  NOT NULL DEFAULT 0 COMMENT '删除时间（0=未删除）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`, `create_at` DESC),
    KEY `idx_user_time` (`user_id`, `create_at` DESC),
    KEY `idx_status` (`status`),
    KEY `idx_freeze_order` (`freeze_order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='提现订单表';
```

**7种状态完整映射（来自财务需求2.4）：**

```
1 待风控审核 ──通过──→ 3 待财务审核 ──通过──→ 5 待出款 ──成功──→ 6 出款成功
     │                      │                    │
     │ 驳回                  │ 驳回                │ 失败
     ▼                      ▼                    ▼
2 风控驳回              4 财务驳回           7 出款失败

驳回/失败时：触发 UnfreezeBalance（解冻退回 frozen→available）
出款成功时：触发 ConfirmDebit（确认扣除 frozen→永久减少）
```

---

### 表7：withdraw_account — 提现账户表

**为什么是【必须】：**

```
产品依据：
  - 钱包需求7.1："首次提现需输入账户信息，平台记录，后续无需重复填写"
  - 钱包需求7.1："账户持有人姓名 = KYC认证姓名（不可编辑）"
  - 原型10-1~10-4：各国各方式的提现表单
  - 没有此表 → 用户每次提现都要重新填写信息 → 不符合产品要求

功能依赖：
  C端：获取提现账户（复用）/ 保存提现账户（首次）/ 创建提现订单时引用
  安全：KYC姓名强校验在保存时执行
```

**DDL：**

```sql
CREATE TABLE `withdraw_account` (
    `id`               bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `user_id`          bigint unsigned  NOT NULL COMMENT '用户ID',
    `currency_code`    varchar(16)      NOT NULL COMMENT '币种代码',
    `withdraw_method`  tinyint unsigned NOT NULL COMMENT '提现方式：1银行卡 2电子钱包 3USDT',
    `bank_name`        varchar(256)     DEFAULT NULL COMMENT '银行名称（加密存储）',
    `bank_account`     varchar(256)     DEFAULT NULL COMMENT '银行账号（加密存储）',
    `bank_branch`      varchar(256)     DEFAULT NULL COMMENT '开户网点',
    `ewallet_type`     varchar(32)      DEFAULT NULL COMMENT '电子钱包类型：GoPay/OVO/DANA/ShopeePay/PromptPay',
    `phone_number`     varchar(256)     DEFAULT NULL COMMENT '手机号（加密存储，电子钱包用）',
    `network_type`     varchar(16)      DEFAULT NULL COMMENT 'USDT网络类型：TRC20/ERC20/BEP20',
    `wallet_address`   varchar(256)     DEFAULT NULL COMMENT 'USDT钱包地址',
    `account_holder`   varchar(256)     NOT NULL COMMENT '账户持有人（= KYC姓名，加密存储）',
    `is_default`       tinyint unsigned NOT NULL DEFAULT 1 COMMENT '是否默认：0否 1是',
    `create_at`        bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    `update_at`        bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新时间（毫秒时间戳）',
    `delete_at`        bigint unsigned  NOT NULL DEFAULT 0 COMMENT '删除时间（0=未删除）',
    PRIMARY KEY (`id`),
    KEY `idx_user_currency_method` (`user_id`, `currency_code`, `withdraw_method`),
    KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='提现账户表';
```

**设计说明：**

```
多态表设计（一张表覆盖三种提现方式）：
  银行卡：bank_name + bank_account + bank_branch + account_holder
  电子钱包：ewallet_type + phone_number + account_holder
  USDT：network_type + wallet_address + account_holder
  不用的字段留 NULL。

为什么不拆三张表：
  三种方式共性大于差异（都有 user_id/currency_code/account_holder）。
  一张表查询简单（不需要 UNION），代码复杂度低。

敏感字段加密：
  bank_account / phone_number / account_holder 需加密存储。
  使用 common/pkg/utils/crypto EncryptedString 模式。
  varchar(256) 预留加密后密文长度（密文比明文长）。

产品需求映射：
  原型10-1越南银行卡：银行名称/银行账号/账户持有人(KYC)/开户网点
  原型10-2印尼电子钱包：GoPay/OVO/DANA/ShopeePay + 手机号 + 持有人
  原型10-3泰国电子钱包：PromptPay + 手机号 + 持有人
  原型10-4 BSB提现：复用各方式 + 汇率展示
```

---

### 表8：exchange_record — 兑换记录表

**为什么是【必须】：**

```
产品依据：
  - 钱包需求6.1-6.5：兑换模块（法币→BSB）
  - 钱包需求8.3兑换记录Tab："兑换金额、赠送x%、实际到账、时间"
  - 原型8"兑换BSB"：余额+汇率+兑换金额+获得+赠送+实际到账
  - 没有此表 → 兑换记录页无数据 → 无法追溯兑换历史

功能依赖：
  C端：执行兑换 / 兑换记录列表
  兑换事务中写入此记录（和余额变更在同一事务）
```

**DDL：**

```sql
CREATE TABLE `exchange_record` (
    `id`                 bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `record_no`          varchar(64)      NOT NULL COMMENT '兑换记录编号（snowflake）',
    `user_id`            bigint unsigned  NOT NULL COMMENT '用户ID',
    `source_currency`    varchar(16)      NOT NULL COMMENT '源币种（法币：VND/IDR/THB）',
    `target_currency`    varchar(16)      NOT NULL DEFAULT 'BSB' COMMENT '目标币种（固定BSB）',
    `source_amount`      bigint unsigned  NOT NULL COMMENT '兑换金额-法币（源币种最小单位整数）',
    `target_amount`      bigint unsigned  NOT NULL COMMENT '兑换获得-BSB（BSB最小单位整数）',
    `bonus_percent`      tinyint unsigned NOT NULL DEFAULT 0 COMMENT '赠送比例（整数，如10表示10%）',
    `bonus_amount`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '赠送金额-BSB（最小单位整数）',
    `total_amount`       bigint unsigned  NOT NULL COMMENT '实际到账-BSB = 兑换获得 + 赠送',
    `rate_snapshot`      bigint unsigned  NOT NULL COMMENT '使用的平台汇率快照（×10^8）',
    `turnover_multiple`  tinyint unsigned DEFAULT NULL COMMENT '赠送部分需完成的流水倍数',
    `create_at`          bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_record_no` (`record_no`),
    KEY `idx_user_source_time` (`user_id`, `source_currency`, `create_at` DESC),
    KEY `idx_user_time` (`user_id`, `create_at` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='兑换记录表';
```

**设计说明：**

```
兑换计算（需求6.3）：
  target_amount = source_amount × 换算系数（按平台汇率）
  bonus_amount = target_amount × bonus_percent / 100
  total_amount = target_amount + bonus_amount

  兑换事务（一个数据库事务内）：
    INSERT exchange_record
    UPDATE user_wallet (法币,center) SET balance -= source_amount
    UPDATE user_wallet (BSB,center) SET balance += target_amount
    UPDATE user_wallet (BSB,reward) SET balance += bonus_amount（有赠送时）
    INSERT wallet_flow (法币扣减)
    INSERT wallet_flow (BSB到账)
    INSERT wallet_flow (BSB赠送)（有赠送时）

赠送规则（需求6.4）：
  "赠送部分需完成X倍流水后可提现"
  → turnover_multiple 存储此要求
  → 赠送到奖励钱包（wallet_type=2），而非中心钱包

此表不做软删除（兑换记录不可删除，金融审计要求）。
此表只有 INSERT，没有 UPDATE（兑换是即时完成的，无状态变更）。
```

---

## 四、【应该】重要支撑表详细设计

以下 3 张表满足产品的重要功能需求。核心交易链路不完全依赖它们，但缺少它们会导致功能缺失或运营能力不足。

---

### 表9：exchange_rate_log — 汇率变更日志表

**为什么是【应该】而非必须：**

```
产品依据：
  - 币种配置需求4.4："在操作日志页增加汇率日志页签"，明确列出7个字段
  - 产品原型3"操作日志 > 汇率日志Tab"
  → 产品需求明确要求此功能，但它是B端管理功能，
    不影响C端充值/提现/兑换等核心交易流程。
    即使没有汇率日志，汇率本身的更新和使用不受影响。
    所以是"应该"而非"必须"。
```

**DDL：**

```sql
CREATE TABLE `exchange_rate_log` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `log_no`          varchar(64)      NOT NULL COMMENT '日志编号（snowflake）',
    `currency_code`   varchar(16)      NOT NULL COMMENT '币种代码',
    `realtime_rate`   bigint unsigned  NOT NULL COMMENT '实时汇率（×10^8）',
    `platform_rate`   bigint unsigned  NOT NULL COMMENT '更新后的平台汇率（×10^8）',
    `deposit_rate`    bigint unsigned  NOT NULL COMMENT '更新后的入款汇率（×10^8）',
    `withdraw_rate`   bigint unsigned  NOT NULL COMMENT '更新后的出款汇率（×10^8）',
    `float_rate`      bigint unsigned  NOT NULL COMMENT '当时的浮动比例（×10^4）',
    `deviation`       bigint unsigned  NOT NULL COMMENT '触发更新的偏差值（×10^4）',
    `threshold`       bigint unsigned  NOT NULL COMMENT '当时的阈值（×10^4）',
    `update_time`     bigint unsigned  NOT NULL COMMENT '汇率更新时间（毫秒时间戳）',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_log_no` (`log_no`),
    KEY `idx_currency_time` (`currency_code`, `update_time` DESC),
    KEY `idx_update_time` (`update_time` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='汇率变更日志表';
```

```
写入时机：定时任务刷新汇率，偏差值>=阈值触发更新时同时写入。
此表只 INSERT 不 UPDATE 不 DELETE，纯追加日志。
```

---

### 表10：manual_adjustment_order — 人工修正订单表

**为什么是【应该】而非必须：**

```
产品依据：
  - 财务需求2.5人工修正：充值补单(B+前缀) / 人工加款(A+前缀) / 人工减款(M+前缀)
  - "均需双人审核"
  → 这是 B端运营/财务管理功能。核心的充值/提现/兑换不依赖此表。
    但产品明确要求了人工修正功能，且涉及资金变动，属于重要功能。

  此表由 ser-wallet 管理（因为最终要调 CreditWallet/DebitWallet/SupplementCredit），
  但创建和审核的交互可能在财务模块的 B端页面发起。
  需要与财务模块确认：人工修正订单存在 ser-wallet 还是财务模块。
  以下按存在 ser-wallet 的方案设计。
```

**DDL：**

```sql
CREATE TABLE `manual_adjustment_order` (
    `id`               bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `order_no`         varchar(64)      NOT NULL COMMENT '订单号（B+补单/A+加款/M+减款 前缀+snowflake）',
    `adjust_type`      tinyint unsigned NOT NULL COMMENT '修正类型：1充值补单 2人工加款 3人工减款',
    `user_id`          bigint unsigned  NOT NULL COMMENT '目标用户ID',
    `currency_code`    varchar(16)      NOT NULL COMMENT '币种代码',
    `wallet_type`      tinyint unsigned NOT NULL DEFAULT 1 COMMENT '目标钱包类型',
    `amount`           bigint unsigned  NOT NULL COMMENT '修正金额（最小单位整数）',
    `actual_amount`    bigint unsigned  NOT NULL DEFAULT 0 COMMENT '实际执行金额（减款时可能小于申请金额）',
    `reason_type`      varchar(32)      NOT NULL COMMENT '原因类型：运营补偿/运营奖励/财务修正/测试加款/异常追回/风控处罚/费用调整',
    `reason_remark`    varchar(512)     DEFAULT NULL COMMENT '原因说明',
    `ref_order_no`     varchar(64)      DEFAULT NULL COMMENT '关联原始订单号（补单时关联原充值订单）',
    `status`           tinyint unsigned NOT NULL DEFAULT 1 COMMENT '状态：1待审核 2审核通过 3审核拒绝',
    `apply_by`         bigint unsigned  NOT NULL COMMENT '申请人ID',
    `apply_by_name`    varchar(64)      NOT NULL DEFAULT '' COMMENT '申请人姓名',
    `audit_by`         bigint unsigned  DEFAULT NULL COMMENT '审核人ID（不可自审自批）',
    `audit_by_name`    varchar(64)      DEFAULT NULL COMMENT '审核人姓名',
    `audit_at`         bigint unsigned  DEFAULT NULL COMMENT '审核时间',
    `audit_remark`     varchar(512)     DEFAULT NULL COMMENT '审核备注',
    `create_at`        bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    `update_at`        bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新时间（毫秒时间戳）',
    `delete_at`        bigint unsigned  NOT NULL DEFAULT 0 COMMENT '删除时间（0=未删除）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_type` (`user_id`, `adjust_type`),
    KEY `idx_status` (`status`),
    KEY `idx_apply_by` (`apply_by`),
    KEY `idx_create_at` (`create_at` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='人工修正订单表';
```

**设计说明：**

```
三种修正类型统一管理（来自财务需求2.5）：
  1=充值补单(B+前缀)：原充值少入账，需要补差额
  2=人工加款(A+前缀)：运营补偿/运营奖励/财务修正/测试加款
  3=人工减款(M+前缀)：异常追回/风控处罚/财务冲正/费用调整

双人审核约束：
  apply_by ≠ audit_by（应用层校验：不可自审自批）

加款特殊逻辑（需求2.52）：
  "可同时向中心钱包/奖励钱包/代理钱包/主播钱包加款"
  → 一次加款申请可能拆分为多条记录（每个钱包类型一条），
    或用一条记录+JSON扩展字段记录各钱包分配。
    简单方案：一种钱包类型一条记录。

减款特殊逻辑（需求2.53）：
  "当用户钱包余额不足时，从最高金额的钱包开始依次扣减"
  "实际减款金额可能小于设定的减款金额"
  → actual_amount 记录实际执行金额（可能 < amount）

审核通过后执行：
  补单 → 调 SupplementCredit（上账）
  加款 → 调 ManualCredit（上账）
  减款 → 调 ManualDebit（扣款）
  全部通过 wallet_flow 记录流水，幂等保证。
```

---

### 表11：wallet_config — 钱包业务配置表

**为什么是【应该】而非必须：**

```
产品依据：
  - 需求6.4 "赠送比例如10%""需完成X倍流水后可提现" → 赠送比例和流水倍数在哪配置？
  - 需求7.5 "单笔限额、单日限额" → 提现限额在哪配置？
  - 需求5.5 重复拦截的 3笔上限 → 拦截阈值在哪配置？
  → 这些都是运行时可调的业务配置。
    如果没有此表，这些值只能硬编码在代码中或写在 ETCD 中。
    有此表则 B端可以动态调整，运营灵活度更高。
    但核心交易流程在硬编码也能跑通，所以是"应该"。
```

**DDL：**

```sql
CREATE TABLE `wallet_config` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `config_key`      varchar(64)      NOT NULL COMMENT '配置键',
    `config_value`    varchar(512)     NOT NULL COMMENT '配置值（JSON或简单值）',
    `config_desc`     varchar(256)     DEFAULT NULL COMMENT '配置说明',
    `currency_code`   varchar(16)      DEFAULT NULL COMMENT '关联币种（NULL=全局配置）',
    `status`          tinyint unsigned NOT NULL DEFAULT 1 COMMENT '状态：0禁用 1启用',
    `create_by`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建人',
    `create_by_name`  varchar(64)      NOT NULL DEFAULT '' COMMENT '创建人姓名',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间（毫秒时间戳）',
    `update_by`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新人',
    `update_by_name`  varchar(64)      NOT NULL DEFAULT '' COMMENT '更新人姓名',
    `update_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '更新时间（毫秒时间戳）',
    `delete_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '删除时间（0=未删除）',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_key_currency` (`config_key`, `currency_code`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='钱包业务配置表';
```

**配置项规划：**

```
预期配置项：

config_key                    | currency_code | config_value         | 来源需求
─────────────────────────────|───────────────|─────────────────────|──────────
exchange_bonus_percent        | VND           | 10                   | 需求6.4赠送10%
exchange_bonus_percent        | IDR           | 10                   |
exchange_bonus_percent        | THB           | 10                   |
exchange_turnover_multiple    | NULL(全局)    | 3                    | 需求6.4流水倍数
withdraw_single_min           | VND           | 100000               | 需求7.5单笔最低
withdraw_single_max           | VND           | 500000000            | 需求7.5单笔限额
withdraw_daily_max            | VND           | 2000000000           | 需求7.5单日限额
withdraw_fee_rate             | VND           | 10000                | 需求7.5手续费(1%=10000)
withdraw_arrive_days          | VND           | 1                    | 需求7.5 T+N到账
deposit_duplicate_limit       | NULL(全局)    | 3                    | 需求5.5拦截上限
rate_refresh_interval_ms      | NULL(全局)    | 60000                | 汇率刷新间隔

以 KV 方式存储，灵活扩展，无需频繁加字段。
B端提供配置管理界面，运营可动态调整。
缓存到 Redis，C端读取走缓存。
```

---

## 五、【可选】扩展表设计

以下 3 张表不影响业务功能，但提供更好的可观测性、统计能力和运维支撑。可在后续版本中按需添加。

---

### 表12：deposit_bonus_snapshot — 充值奖金快照表（可选）

**为什么是【可选】：**

```
当前方案：充值奖金信息存在 deposit_order 的 3 个字段中（bonus_activity_id/bonus_amount/bonus_turnover）。
扩展需求：如果未来奖金规则变复杂（多活动叠加、分阶段发放、流水进度跟踪），
          需要独立表记录每笔充值关联的奖金详情。

当前阶段不需要，deposit_order 中的字段足够。
```

```sql
-- 可选：未来需要时再建
CREATE TABLE `deposit_bonus_snapshot` (
    `id`                bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `deposit_order_no`  varchar(64)      NOT NULL COMMENT '关联充值订单号',
    `activity_id`       bigint unsigned  NOT NULL COMMENT '活动ID',
    `activity_name`     varchar(128)     DEFAULT NULL COMMENT '活动名称',
    `bonus_percent`     tinyint unsigned NOT NULL COMMENT '奖金比例(%)',
    `bonus_amount`      bigint unsigned  NOT NULL COMMENT '奖金金额（最小单位）',
    `turnover_required` bigint unsigned  NOT NULL DEFAULT 0 COMMENT '需完成流水金额',
    `turnover_done`     bigint unsigned  NOT NULL DEFAULT 0 COMMENT '已完成流水金额',
    `status`            tinyint unsigned NOT NULL DEFAULT 1 COMMENT '1进行中 2已完成 3已过期',
    `create_at`         bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间',
    PRIMARY KEY (`id`),
    KEY `idx_deposit_order` (`deposit_order_no`),
    KEY `idx_user_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='充值奖金快照表';
```

---

### 表13：wallet_daily_snapshot — 每日钱包快照表（可选）

**为什么是【可选】：**

```
用途：每日定时快照所有用户的钱包余额，用于：
  - 运营统计（每日活跃余额、资金池大小）
  - 对账辅助（快照余额 vs 流水累计 = 当日余额）
  - 风控（异常余额变动检测）

当前阶段不需要。对账可以通过 wallet_flow 实时计算。
未来用户量大、对账需求频繁时可以引入。
```

```sql
-- 可选：运营统计需要时再建
CREATE TABLE `wallet_daily_snapshot` (
    `id`                bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `snapshot_date`     int unsigned     NOT NULL COMMENT '快照日期（YYYYMMDD整数）',
    `user_id`           bigint unsigned  NOT NULL COMMENT '用户ID',
    `currency_code`     varchar(16)      NOT NULL COMMENT '币种代码',
    `wallet_type`       tinyint unsigned NOT NULL COMMENT '钱包类型',
    `available_balance` bigint unsigned  NOT NULL COMMENT '当日结余-可用',
    `frozen_balance`    bigint unsigned  NOT NULL COMMENT '当日结余-冻结',
    `create_at`         bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_date_user_currency_type` (`snapshot_date`, `user_id`, `currency_code`, `wallet_type`),
    KEY `idx_snapshot_date` (`snapshot_date`),
    KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='每日钱包快照表';
```

---

### 表14：rate_provider_log — 三方汇率源日志表（可选）

**为什么是【可选】：**

```
用途：记录每次从三方 API 获取的原始汇率数据，用于：
  - 追溯汇率来源（哪家 API 返回了什么值）
  - 排查汇率异常（某家 API 返回异常值导致平均值偏离）
  - 三方 API 质量评估

当前阶段不需要。exchange_rate_log 记录了最终结果。
排查三方问题时可以查服务日志。未来需要精细化运维时再建。
```

```sql
-- 可选：汇率排查需要时再建
CREATE TABLE `rate_provider_log` (
    `id`              bigint unsigned  NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `currency_code`   varchar(16)      NOT NULL COMMENT '币种代码',
    `provider_name`   varchar(64)      NOT NULL COMMENT '三方API名称',
    `provider_rate`   bigint unsigned  NOT NULL COMMENT '该API返回的汇率（×10^8）',
    `avg_rate`        bigint unsigned  NOT NULL COMMENT '最终取均值后的汇率（×10^8）',
    `response_ms`     int unsigned     NOT NULL DEFAULT 0 COMMENT 'API响应耗时(毫秒)',
    `is_success`      tinyint unsigned NOT NULL DEFAULT 1 COMMENT '请求是否成功：0失败 1成功',
    `error_msg`       varchar(512)     DEFAULT NULL COMMENT '失败原因',
    `create_at`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '创建时间',
    PRIMARY KEY (`id`),
    KEY `idx_currency_time` (`currency_code`, `create_at` DESC),
    KEY `idx_provider` (`provider_name`, `create_at` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='三方汇率源日志表';
```

---

## 六、表之间的关联关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           表关联关系图                                   │
│                                                                         │
│            ┌──────────────────┐                                         │
│            │ currency_config   │ ← 被所有表的 currency_code 引用（基础表）│
│            │ UK:currency_code  │                                         │
│            └────────┬─────────┘                                         │
│                     │                                                   │
│        ┌────────────┼───────────────────────────┐                       │
│        │            │                           │                       │
│        ▼            ▼                           ▼                       │
│  ┌────────────┐ ┌──────────────┐ ┌─────────────────────┐               │
│  │ user_wallet │ │ wallet_flow  │ │ exchange_rate_log    │               │
│  │ UK:(uid,    │ │ UK:(order_no,│ │ UK:log_no            │               │
│  │  code,type) │ │  flow_type)  │ └─────────────────────┘               │
│  └──────┬─────┘ └──────────────┘                                       │
│         │                                                               │
│    被以下表操作                                                          │
│    (balance变更)                                                        │
│         │                                                               │
│  ┌──────┴──────┬─────────────────┬──────────────────┐                   │
│  │             │                 │                  │                   │
│  ▼             ▼                 ▼                  ▼                   │
│ ┌────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────────────┐          │
│ │deposit │ │withdraw  │ │exchange      │ │manual_adjustment │          │
│ │_order  │ │_order    │ │_record       │ │_order            │          │
│ └────────┘ └─────┬────┘ └──────────────┘ └──────────────────┘          │
│                  │                                                      │
│           ┌──────┴──────────┐                                          │
│           │                 │                                          │
│           ▼                 ▼                                          │
│  ┌─────────────────┐ ┌──────────────────┐                              │
│  │wallet_freeze    │ │withdraw_account  │                              │
│  │_record          │ │                  │                              │
│  │UK:order_no      │ │                  │                              │
│  └─────────────────┘ └──────────────────┘                              │
│                                                                         │
│  ┌────────────────┐                                                     │
│  │wallet_config   │  ← 独立配置表，不直接关联其他表                       │
│  └────────────────┘                                                     │
│                                                                         │
│  关联方式：全部为逻辑外键（不建物理 FK 约束）                              │
│  原因：TiDB 推荐 + 微服务架构 + 数据完整性在应用层保证                     │
│                                                                         │
│  关键关联：                                                              │
│    withdraw_order.withdraw_account_id  → withdraw_account.id            │
│    withdraw_order.freeze_order_no      → wallet_freeze_record.order_no  │
│    manual_adjustment.ref_order_no      → deposit_order.order_no (补单时) │
│    所有表.currency_code                → currency_config.currency_code   │
│    所有表.user_id                      → ser-user 的用户ID (跨服务)       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 七、金额字段精度统一方案

```
┌────────────────────────────────────────────────────────────────────────┐
│                     金额存储方案（贯穿所有表）                            │
│                                                                        │
│  【数据库层】bigint unsigned — 最小货币单位整数                          │
│  【Go 代码层】int64（unsigned 映射后） — 直接整数运算                    │
│  【RPC 传输层】string — IDL 中金额字段定义为 string 类型                │
│  【前端展示层】按 currency_config.decimal_digits 格式化                  │
│                                                                        │
│  币种     decimal_digits  放大倍数     示例                              │
│  ────     ─────────────   ────────     ────                              │
│  VND      0               1            500,000 VND  → 存 500000          │
│  IDR      0               1            8,000,000 IDR → 存 8000000        │
│  THB      2               100          1,500.50 THB → 存 150050          │
│  BSB      2               100          1,000.00 BSB → 存 100000          │
│  USDT     8               10^8         100.50 USDT  → 存 10050000000     │
│                                                                        │
│  为什么 bigint 而不是 DECIMAL：                                         │
│    1. 整数运算无精度损失（0.1+0.2≠0.3 的问题不存在）                     │
│    2. 整数比较和索引更高效                                               │
│    3. 现有工程 base_price 就是整数存储                                   │
│    4. 业界标准：Stripe/PayPal/蚂蚁金服的核心账务都用整数                 │
│                                                                        │
│  为什么 bigint 而不是 int：                                             │
│    int unsigned 最大 4,294,967,295 ≈ 42亿                               │
│    VND 10亿越南盾 = 10^9（已超 int 一半）                                │
│    USDT 100 USDT = 10,050,000,000（8位小数展开后溢出 int）               │
│    bigint unsigned 最大 ~1.8×10^19，足够所有场景                         │
│                                                                        │
│  汇率存储：bigint unsigned ×10^8                                        │
│  百分比存储：bigint unsigned ×10^4                                       │
│  详见 currency_config 表设计说明                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 八、建表顺序与执行计划

```
阶段一：基础表（开发环境搭建时立即建）
  1. currency_config        ← 所有模块的基础
  2. user_wallet             ← 余额管理核心
  3. wallet_flow             ← 流水/幂等核心
  4. wallet_freeze_record    ← 冻结状态机

阶段二：交易表（业务开发时建）
  5. exchange_record         ← 兑换功能（最先实现，零外部依赖）
  6. deposit_order           ← 充值功能
  7. withdraw_account        ← 提现前置
  8. withdraw_order          ← 提现功能

阶段三：管理表（B端功能开发时建）
  9. exchange_rate_log       ← 汇率日志B端页面
  10. manual_adjustment_order ← 人工修正功能
  11. wallet_config           ← 业务配置动态化

阶段四：扩展表（按需，可后续版本）
  12. deposit_bonus_snapshot  ← 奖金精细化管理
  13. wallet_daily_snapshot   ← 运营统计
  14. rate_provider_log       ← 汇率排查
```

---

## 九、与财务模块的表边界说明

```
┌──────────────────────────────────────────────────────────────────────────┐
│           ser-wallet 和 财务模块 的数据边界                                │
│                                                                          │
│  ser-wallet 管（存在 ser_wallet 数据库）：                                 │
│    currency_config         — 币种+汇率是钱包的基础设施                    │
│    user_wallet             — 用户余额                                    │
│    wallet_flow             — 资金流水                                    │
│    wallet_freeze_record    — 冻结记录                                    │
│    deposit_order           — 钱包侧的充值订单（金额、方式、状态、奖金）     │
│    withdraw_order          — 钱包侧的提现订单（金额、方式、状态、冻结）     │
│    withdraw_account        — 提现账户信息                                │
│    exchange_record         — 兑换记录                                    │
│    exchange_rate_log       — 汇率日志                                    │
│    manual_adjustment_order — 人工修正订单（如果存钱包侧）                  │
│    wallet_config           — 钱包业务配置                                │
│                                                                          │
│  财务模块管（存在 ser_finance 数据库，非我负责）：                          │
│    payment_channel         — 代收通道配置（通道名/费率/限额/密钥等）       │
│    payout_channel          — 代付通道配置                                │
│    channel_polling_rule    — 通道轮询规则（三系数+区间）                  │
│    payment_method          — 充值方式配置（档位/范围/关联通道/排序/状态）   │
│    withdraw_method_config  — 提现方式配置（费率/大额费率/关联通道/排序）    │
│    finance_deposit_record  — 财务侧充值记录（24字段，含通道费率/手续费等） │
│    finance_withdraw_record — 财务侧提现记录（27字段，含审核/出款详情）     │
│    channel_statistics      — 通道统计数据                                │
│    deposit_statistics      — 充值统计数据                                │
│    withdraw_statistics     — 提现统计数据                                │
│                                                                          │
│  两者通过 order_no 关联：                                                 │
│    ser-wallet.deposit_order.order_no = 财务.finance_deposit_record 的引用 │
│    ser-wallet.withdraw_order.order_no = 财务.finance_withdraw_record 引用 │
│                                                                          │
│  数据流向：                                                               │
│    充值：财务收到三方回调 → RPC调 ser-wallet.CreditWallet → 钱包上账+写流水│
│    提现：钱包冻结余额 → RPC调 财务创建审核订单 → 财务审核/出款 → 回调钱包  │
│    人工：财务B端发起 → RPC调 ser-wallet.ManualCredit/Debit → 钱包执行      │
│                                                                          │
│  原则：ser-wallet 只管「账」，不管「通道」                                 │
│    钱包不存储通道配置、通道密钥、通道费率                                  │
│    钱包不直接和三方支付通道交互                                            │
│    钱包通过 RPC 从财务获取支付方式列表                                     │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 十、总结

```
表结构设计统计：

  【必须】8 张表 — 核心交易链路，缺一不可
    currency_config / user_wallet / wallet_flow / wallet_freeze_record
    deposit_order / withdraw_order / withdraw_account / exchange_record

  【应该】3 张表 — 重要管理功能，产品明确要求
    exchange_rate_log / manual_adjustment_order / wallet_config

  【可选】3 张表 — 扩展优化，后续版本按需引入
    deposit_bonus_snapshot / wallet_daily_snapshot / rate_provider_log

  合计 14 张表，阶段一只需建前 4 张即可启动开发。

设计原则验证：
  ✓ 完全对齐现有工程 DDL 规范（bigint unsigned / tinyint / utf8mb4_bin / delete_at）
  ✓ 金额统一 bigint unsigned 整数存储（匹配 ser-item 的 int 模式，但范围更大）
  ✓ 汇率统一 ×10^8 整数存储（覆盖 USDT 8位小数精度）
  ✓ 幂等通过 wallet_flow 唯一索引保证（无需额外幂等表）
  ✓ 状态机通过 WHERE 条件更新实现（无额外框架）
  ✓ 不使用物理外键（对齐 TiDB 推荐 + 现有工程做法）
  ✓ 每张表都有产品需求原文作为设计依据
```
