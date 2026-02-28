# ser-wallet 钱包模块：从 0 到 1 全景设计纲要

> 综合 30+ 份分析文档、产品原型、参考实现（Java法币钱包 + TRC20虚拟币钱包）、工程代码库实际模式
> 目标：需求评审会上全方位无死角阐述钱包模块的设计思路、技术方案、风险预案
> 日期：2026-02-28

---

## 目录

- [一、设计思路总纲](#一设计思路总纲)
- [二、业务域全景](#二业务域全景)
- [三、表结构设计](#三表结构设计)
- [四、接口全景](#四接口全景)
- [五、技术栈选型与工程规范](#五技术栈选型与工程规范)
- [六、十大技术挑战与解法](#六十大技术挑战与解法)
- [七、核心业务流程状态机](#七核心业务流程状态机)
- [八、跨模块协作与依赖](#八跨模块协作与依赖)
- [九、需要与产品/评审确认的问题清单](#九需要与产品评审确认的问题清单)
- [十、潜在提问与应对预案](#十潜在提问与应对预案)
- [十一、编码节奏与落地策略](#十一编码节奏与落地策略)

---

## 一、设计思路总纲

### 1.1 核心定位

钱包模块是平台的**资金引擎**，不直接面向运营后台（B端运营页面由 ser-finance 负责），而是作为金融操作的底层原语层存在。所有余额变更**只能**通过 ser-wallet 的 RPC 接口完成——这是整个资金安全体系的根基。

一句话概括：**余额只有钱包能改，钱包只做余额管理。**

### 1.2 职责边界三层切割

```
┌────────────────────────────────────────────────────────────────┐
│  第一层：币种配置（Currency Config）                             │
│  · 币种元数据管理（法币/虚拟币/平台币）                          │
│  · 汇率体系（实时/平台/入款/出款）                               │
│  · 全局基准币种设定（USDT，一次性不可改）                        │
│  · 定时任务：汇率自动刷新                                       │
│  → 数据源角色，为所有模块提供币种和汇率信息                      │
├────────────────────────────────────────────────────────────────┤
│  第二层：钱包核心（Wallet Core）                                 │
│  · 多币种 × 多子钱包的余额管理                                  │
│  · 6 个金融操作原语：加款/扣款/冻结/解冻/确认扣款/初始化          │
│  · C端用户流程：充值/提现/兑换/记录查询                          │
│  · 幂等 + 事务 + 并发控制 = 资金安全铁三角                      │
│  → 执行角色，所有余额变更的唯一入口                              │
├────────────────────────────────────────────────────────────────┤
│  第三层：财务管理（Finance，别人负责，我们配合）                    │
│  · 支付通道对接、三方回调、审核流程                               │
│  · B端运营：人工加款/减款/补单、交易记录查看                      │
│  · 通过 RPC 调用我们的加款/扣款/冻结/解冻接口                    │
│  → 协作角色，我们提供能力，他们编排流程                           │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 设计原则

| 原则 | 具体含义 | 工程落地 |
|------|---------|---------|
| **余额只有钱包能改** | 任何模块不得直接操作钱包表 | 所有写操作走 RPC 接口，DB 权限隔离 |
| **金额不用浮点** | 杜绝精度丢失 | DECIMAL(20,4) 存储，i64 最小单位传输 |
| **每笔变更可追溯** | 流水不可变，记录 before/after | wallet_transaction 表 INSERT-ONLY |
| **幂等是底线** | 重试不会多扣多加 | Redis SetNX + DB 唯一索引双层保障 |
| **悲观锁保并发** | 写多读少场景用 FOR UPDATE | SELECT FOR UPDATE 行级锁串行化 |
| **冻结不等于扣款** | 冻结三阶段：冻结→(解冻 OR 确认扣款) | 状态机互斥约束 |
| **降级不降核心** | 日志/图标等非核心失败不阻塞主流程 | oneway 调用 + 降级兜底 |

### 1.4 盖房子比喻

```
地基层 → 币种配置 + wallet_account 表 + 基础 CRUD
框架层 → 6 个金融原语 RPC（加/扣/冻结/解冻/确认/初始化）
装修层 → C端流程（充值/提现/兑换/记录）
软装层 → 汇率定时刷新、对账、监控告警
```

先打地基，再搭框架，装修可以后补，但地基和框架的设计必须一步到位。

---

## 二、业务域全景

### 2.1 多子钱包体系

| 子钱包类型 | walletType | 资金来源 | 资金用途 | 首版状态 |
|-----------|-----------|---------|---------|---------|
| 中心钱包 | 1 | 充值、兑换入、补单 | 投注/消费/提现（需审计） | **核心** |
| 奖励钱包 | 2 | 活动奖金、投注返奖、充值奖金 | 达到流水要求后可提现 | **核心** |
| 主播钱包 | 3 | 礼物打赏收入、视频收益 | 直接提现/转入中心 | **核心** |
| 代理钱包 | 4 | 代理佣金 | 直接提现/转入中心 | 预留 |
| 场馆钱包 | 5 | 中心+奖励转入 | 游戏场馆内临时使用，退出归还 | 预留 |

**关键公式**：
- 资源钱包余额 = 中心钱包可用额 + 奖励钱包可用额
- 可用余额 = available_amt（不含冻结）
- 消费优先级：中心钱包先扣 → 不足再扣奖励钱包
- 混合投注返奖：按各钱包投注金额比例分配回各钱包

### 2.2 多币种体系

| 币种类型 | 具体币种 | 精度 | 千分位 | 小数位 |
|---------|---------|------|--------|--------|
| 法定货币 | VND（越南盾） | 整数 | `.` | 0 |
| 法定货币 | IDR（印尼盾） | 整数 | `.` | 0 |
| 法定货币 | THB（泰铢） | 2位 | `,` | 2 |
| 虚拟货币 | USDT（泰达币） | 8位 | — | 8 |
| 平台币 | BSB | 2位 | `,` | 2 |

**币种隔离原则**：每个用户 × 每个币种 × 每种子钱包 = 一条 wallet_account 记录。币种之间不混用，跨币种操作只能通过「兑换」完成。

### 2.3 汇率体系

```
                    三方API均值
                        │
                        ▼
              ┌─────────────────┐
              │   实时汇率 (R)   │  ← 定时任务轮询（如每5分钟）
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          │                         │
  R × (1 + 浮动%)          R × (1 - 浮动%)
          │                         │
          ▼                         ▼
   ┌──────────────┐         ┌──────────────┐
   │  入款汇率(D)  │         │  出款汇率(W)  │
   └──────────────┘         └──────────────┘

偏差检测：
  偏差值 = |实时 - 平台| / 实时 × 100%
  if 偏差值 ≥ 阈值% → 自动更新平台汇率
  触发条件：入款/出款汇率任一超过阈值即触发
```

**基准币种**：USDT，设置后不可修改。BSB 锚定比例：10 BSB = 1 USDT。

### 2.4 充值方式（3种）

| 方式 | 承载方 | 确认机制 | KYC要求 |
|------|--------|---------|---------|
| USDT 链上充值 | 平台自托管 | 链上确认（区块确认数） | 不需要 |
| 银行转账 | 三方通道 | 三方回调 | 需要 |
| 电子钱包扫码 | 三方通道 | 三方回调 + 倒计时 | 需要 |

**防重复充值**：同一用户 + 同币种 + 同金额，1-2次软拦截（可重试），≥3次硬拦截。

### 2.5 提现规则

| 规则项 | 说明 |
|--------|------|
| KYC 影响 | 未认证 → 仅 USDT 提现；已认证 → 全渠道 |
| 币种规则 | 法币充 → 法币提；USDT充 → USDT提；混合充 → 法币提 |
| 账户信息 | 首次必须填写，后续复用不可自改（需客服介入修改） |
| 姓名校验 | 持卡人姓名 = KYC 实名（大小写不敏感、去首尾空格） |
| 限额 | 单笔最小/最大、单日累计（B端配置） |
| 结算周期 | T+N 结算 |
| 流水审计 | 奖励钱包余额需完成 N 倍流水要求后方可提现 |

### 2.6 兑换逻辑

- 方向：法币 → BSB（单向，不可反向）
- 计算：兑换额 = 法币金额 × 入款汇率 / BSB锚定比例
- 赠送：按活动规则，赠送部分进奖励钱包
- 事务：纯内部操作，同一事务内完成（扣法币 + 加 BSB + 加赠送），不依赖外部 RPC

---

## 三、表结构设计

### 3.1 表清单概览

| # | 表名 | 说明 | 记录量级/年 | 优先级 |
|---|------|------|-----------|--------|
| 1 | currency_config | 币种配置主表 | <50 条 | 必须 |
| 2 | exchange_rate_log | 汇率变更日志 | ~50万条 | 必须 |
| 3 | wallet_account | 钱包账户余额 | <100万条 | 必须 |
| 4 | wallet_transaction | 交易流水（不可变） | **5000万+** | 必须 |
| 5 | deposit_order | 充值订单 | ~500万条 | 必须 |
| 6 | withdraw_order | 提现订单 | ~200万条 | 必须 |
| 7 | withdraw_account | 提现账户信息 | <100万条 | 必须 |
| 8 | exchange_order | 兑换订单 | ~100万条 | 应有 |
| 9 | audit_requirement | 流水审计要求 | <100万条 | 应有 |
| 10 | freeze_detail | 冻结明细 | ~200万条 | 必须 |

### 3.2 核心表结构详述

#### 表1：currency_config（币种配置）

```sql
CREATE TABLE currency_config (
    id              bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    currency_code   varchar(10)     NOT NULL COMMENT '币种代码 VND/IDR/THB/USDT/BSB',
    currency_name   varchar(50)     NOT NULL COMMENT '币种名称',
    currency_type   tinyint         NOT NULL COMMENT '1=法币 2=虚拟币 3=平台币',
    currency_symbol varchar(10)     NOT NULL COMMENT '符号 ₫/$',
    decimal_places  tinyint         NOT NULL DEFAULT 2 COMMENT '显示小数位',
    thousand_sep    varchar(5)      NOT NULL DEFAULT ',' COMMENT '千分位分隔符',
    icon_url        varchar(255)    NOT NULL DEFAULT '' COMMENT '图标URL',
    is_base         tinyint         NOT NULL DEFAULT 0 COMMENT '是否基准币种 0=否 1=是',
    anchor_rate     decimal(20,8)   NOT NULL DEFAULT 0 COMMENT 'BSB锚定比例(仅BSB有值)',
    rate_realtime   decimal(20,8)   NOT NULL DEFAULT 0 COMMENT '实时汇率(对USDT)',
    rate_platform   decimal(20,8)   NOT NULL DEFAULT 0 COMMENT '平台汇率',
    rate_deposit    decimal(20,8)   NOT NULL DEFAULT 0 COMMENT '入款汇率',
    rate_withdraw   decimal(20,8)   NOT NULL DEFAULT 0 COMMENT '出款汇率',
    float_deposit   decimal(10,4)   NOT NULL DEFAULT 0 COMMENT '入款浮动比例(%)',
    float_withdraw  decimal(10,4)   NOT NULL DEFAULT 0 COMMENT '出款浮动比例(%)',
    deviation_threshold decimal(10,4) NOT NULL DEFAULT 0 COMMENT '偏差阈值(%)',
    sort_order      int             NOT NULL DEFAULT 1000 COMMENT '排序权重',
    status          tinyint         NOT NULL DEFAULT 1 COMMENT '1=启用 0=禁用',
    create_by       bigint unsigned NOT NULL DEFAULT 0,
    create_by_name  varchar(50)     NOT NULL DEFAULT '',
    update_by       bigint unsigned NOT NULL DEFAULT 0,
    update_by_name  varchar(50)     NOT NULL DEFAULT '',
    create_at       bigint unsigned NOT NULL DEFAULT 0 COMMENT '毫秒时间戳',
    update_at       bigint unsigned NOT NULL DEFAULT 0,
    delete_at       bigint unsigned NOT NULL DEFAULT 0 COMMENT '0=未删除',
    UNIQUE KEY uk_currency_code (currency_code, delete_at)
) ENGINE=InnoDB COMMENT='币种配置表';
```

#### 表2：exchange_rate_log（汇率变更日志）

```sql
CREATE TABLE exchange_rate_log (
    id              bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    currency_code   varchar(10)     NOT NULL,
    rate_before     decimal(20,8)   NOT NULL COMMENT '变更前汇率',
    rate_after      decimal(20,8)   NOT NULL COMMENT '变更后汇率',
    rate_type       tinyint         NOT NULL COMMENT '1=实时 2=平台 3=入款 4=出款',
    trigger_type    tinyint         NOT NULL COMMENT '1=定时自动 2=手动刷新 3=偏差触发',
    deviation_value decimal(10,4)   NOT NULL DEFAULT 0 COMMENT '触发时偏差值',
    api_source      varchar(100)    NOT NULL DEFAULT '' COMMENT 'API来源',
    create_by       bigint unsigned NOT NULL DEFAULT 0,
    create_by_name  varchar(50)     NOT NULL DEFAULT '',
    create_at       bigint unsigned NOT NULL DEFAULT 0,
    INDEX idx_currency_code (currency_code),
    INDEX idx_create_at (create_at)
) ENGINE=InnoDB COMMENT='汇率变更日志(INSERT-ONLY)';
```

#### 表3：wallet_account（钱包账户）

```sql
CREATE TABLE wallet_account (
    id              bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id         bigint unsigned NOT NULL COMMENT '用户ID',
    wallet_type     tinyint         NOT NULL COMMENT '1=中心 2=奖励 3=主播 4=代理 5=场馆',
    currency_code   varchar(10)     NOT NULL COMMENT '币种代码',
    available_amt   decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '可用余额',
    frozen_amt      decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '冻结金额',
    version         int unsigned    NOT NULL DEFAULT 1 COMMENT '乐观锁版本号',
    create_at       bigint unsigned NOT NULL DEFAULT 0,
    update_at       bigint unsigned NOT NULL DEFAULT 0,
    UNIQUE KEY uk_user_wallet_currency (user_id, wallet_type, currency_code),
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB COMMENT='钱包账户表';
```

**设计要点**：
- `available_amt` 是可用余额，不含冻结。总余额 = available_amt + frozen_amt
- 不用 BIGINT 存最小单位而用 DECIMAL(20,4)，因为法币精度2位、BSB精度2位、汇率精度8位，4位是存储精度（传输时用 i64 最小单位）
- `version` 乐观锁字段，配合 SELECT FOR UPDATE 使用

#### 表4：wallet_transaction（交易流水，不可变）

```sql
CREATE TABLE wallet_transaction (
    id               bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id          bigint unsigned NOT NULL,
    wallet_id        bigint unsigned NOT NULL COMMENT '关联wallet_account.id',
    wallet_type      tinyint         NOT NULL,
    currency_code    varchar(10)     NOT NULL,
    order_no         varchar(64)     NOT NULL COMMENT '业务订单号',
    op_type          tinyint         NOT NULL COMMENT '操作类型:1=充值 2=充值奖金 3=补单 4=人工加款 5=活动奖金 6=兑换入 7=兑换赠送 8=打赏收入 9=返奖 10=消费 11=兑换出 12=提现冻结 13=提现确认 14=提现解冻 15=人工减款',
    direction        tinyint         NOT NULL COMMENT '1=收入 2=支出 3=冻结 4=解冻',
    amount           decimal(20,4)   NOT NULL COMMENT '变动金额(绝对值)',
    available_before decimal(20,4)   NOT NULL COMMENT '变动前可用余额',
    available_after  decimal(20,4)   NOT NULL COMMENT '变动后可用余额',
    frozen_before    decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '变动前冻结金额',
    frozen_after     decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '变动后冻结金额',
    remark           varchar(256)    NOT NULL DEFAULT '' COMMENT '备注',
    create_at        bigint unsigned NOT NULL DEFAULT 0,
    UNIQUE KEY uk_order_op_wallet (order_no, op_type, wallet_type),
    INDEX idx_user_id (user_id),
    INDEX idx_wallet_id (wallet_id),
    INDEX idx_create_at (create_at)
) ENGINE=InnoDB COMMENT='交易流水(INSERT-ONLY，禁止UPDATE/DELETE)';
```

**设计要点**：
- **INSERT-ONLY**：代码层面禁止 UPDATE/DELETE，保证审计链完整
- **before/after 快照**：每笔流水记录变动前后的可用余额和冻结金额，可回溯任意时点余额
- **幂等唯一索引**：`(order_no, op_type, wallet_type)` 三元组唯一，防止重复记账
- **年增量 5000万+**：未来需考虑按月/季度归档（wallet_transaction_archive_YYYYMM）

#### 表5：deposit_order（充值订单）

```sql
CREATE TABLE deposit_order (
    id               bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_no         varchar(64)     NOT NULL COMMENT '充值订单号 C+16位',
    user_id          bigint unsigned NOT NULL,
    currency_code    varchar(10)     NOT NULL,
    amount           decimal(20,4)   NOT NULL COMMENT '充值金额',
    actual_amount    decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '实际到账金额',
    bsb_amount       decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '兑换BSB金额(如有)',
    pay_method       tinyint         NOT NULL COMMENT '1=USDT 2=银行转账 3=电子钱包',
    channel_code     varchar(50)     NOT NULL DEFAULT '' COMMENT '支付通道编码',
    channel_order_no varchar(100)    NOT NULL DEFAULT '' COMMENT '三方订单号',
    rate_snapshot    decimal(20,8)   NOT NULL DEFAULT 0 COMMENT '下单时汇率快照',
    bonus_plan_id    bigint unsigned NOT NULL DEFAULT 0 COMMENT '奖金活动ID',
    bonus_amount     decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '奖金金额',
    bonus_multiple   int             NOT NULL DEFAULT 0 COMMENT '流水倍数要求',
    status           tinyint         NOT NULL DEFAULT 0 COMMENT '0=待支付 1=支付中 2=已到账 3=已失败 4=已关闭',
    pay_time         bigint unsigned NOT NULL DEFAULT 0 COMMENT '支付完成时间',
    callback_time    bigint unsigned NOT NULL DEFAULT 0 COMMENT '回调时间',
    expire_time      bigint unsigned NOT NULL DEFAULT 0 COMMENT '过期时间',
    client_ip        varchar(50)     NOT NULL DEFAULT '',
    device_type      varchar(20)     NOT NULL DEFAULT '' COMMENT 'APP/H5/PC',
    remark           varchar(256)    NOT NULL DEFAULT '',
    create_at        bigint unsigned NOT NULL DEFAULT 0,
    update_at        bigint unsigned NOT NULL DEFAULT 0,
    delete_at        bigint unsigned NOT NULL DEFAULT 0,
    UNIQUE KEY uk_order_no (order_no),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_create_at (create_at)
) ENGINE=InnoDB COMMENT='充值订单表';
```

#### 表6：withdraw_order（提现订单，最复杂）

```sql
CREATE TABLE withdraw_order (
    id               bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_no         varchar(64)     NOT NULL COMMENT '提现订单号 T+16位',
    user_id          bigint unsigned NOT NULL,
    currency_code    varchar(10)     NOT NULL,
    amount           decimal(20,4)   NOT NULL COMMENT '提现金额',
    fee_amount       decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '手续费',
    actual_amount    decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '实际到账=提现-手续费',
    withdraw_method  tinyint         NOT NULL COMMENT '1=银行卡 2=电子钱包 3=USDT',
    account_id       bigint unsigned NOT NULL COMMENT '关联withdraw_account.id',
    rate_snapshot    decimal(20,8)   NOT NULL DEFAULT 0 COMMENT '下单时汇率快照',
    freeze_order_no  varchar(64)     NOT NULL DEFAULT '' COMMENT '冻结关联单号',
    center_freeze    decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '中心钱包冻结金额',
    reward_freeze    decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '奖励钱包冻结金额',
    status           tinyint         NOT NULL DEFAULT 0 COMMENT '见状态机',
    risk_status      tinyint         NOT NULL DEFAULT 0 COMMENT '风控审核: 0=待审 1=通过 2=驳回',
    risk_reviewer    bigint unsigned NOT NULL DEFAULT 0,
    risk_review_time bigint unsigned NOT NULL DEFAULT 0,
    risk_remark      varchar(256)    NOT NULL DEFAULT '',
    finance_status   tinyint         NOT NULL DEFAULT 0 COMMENT '财务审核: 0=待审 1=通过 2=驳回',
    finance_reviewer bigint unsigned NOT NULL DEFAULT 0,
    finance_review_time bigint unsigned NOT NULL DEFAULT 0,
    finance_remark   varchar(256)    NOT NULL DEFAULT '',
    payout_time      bigint unsigned NOT NULL DEFAULT 0 COMMENT '出款时间',
    payout_tx_id     varchar(128)    NOT NULL DEFAULT '' COMMENT '出款交易号/链上hash',
    fail_reason      varchar(256)    NOT NULL DEFAULT '',
    client_ip        varchar(50)     NOT NULL DEFAULT '',
    device_type      varchar(20)     NOT NULL DEFAULT '',
    create_at        bigint unsigned NOT NULL DEFAULT 0,
    update_at        bigint unsigned NOT NULL DEFAULT 0,
    delete_at        bigint unsigned NOT NULL DEFAULT 0,
    UNIQUE KEY uk_order_no (order_no),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_create_at (create_at)
) ENGINE=InnoDB COMMENT='提现订单表';
```

#### 表7：withdraw_account（提现账户）

```sql
CREATE TABLE withdraw_account (
    id               bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id          bigint unsigned NOT NULL,
    account_type     tinyint         NOT NULL COMMENT '1=银行卡 2=电子钱包 3=USDT地址',
    currency_code    varchar(10)     NOT NULL,
    account_name     varchar(100)    NOT NULL COMMENT '持卡人/账户名(与KYC实名比对)',
    account_no       varchar(100)    NOT NULL COMMENT '卡号/钱包号/链地址',
    bank_name        varchar(100)    NOT NULL DEFAULT '' COMMENT '开户行名称',
    bank_branch      varchar(100)    NOT NULL DEFAULT '' COMMENT '支行',
    chain_network    varchar(20)     NOT NULL DEFAULT '' COMMENT 'TRC20/ERC20/BEP20',
    is_default       tinyint         NOT NULL DEFAULT 1 COMMENT '是否默认',
    create_at        bigint unsigned NOT NULL DEFAULT 0,
    update_at        bigint unsigned NOT NULL DEFAULT 0,
    delete_at        bigint unsigned NOT NULL DEFAULT 0,
    INDEX idx_user_id (user_id),
    INDEX idx_account_type (account_type)
) ENGINE=InnoDB COMMENT='提现账户信息(首次填写后锁定)';
```

#### 表8：freeze_detail（冻结明细）

```sql
CREATE TABLE freeze_detail (
    id              bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id         bigint unsigned NOT NULL,
    wallet_id       bigint unsigned NOT NULL,
    wallet_type     tinyint         NOT NULL,
    currency_code   varchar(10)     NOT NULL,
    ref_order_no    varchar(64)     NOT NULL COMMENT '关联业务单号(提现订单号)',
    freeze_amount   decimal(20,4)   NOT NULL,
    status          tinyint         NOT NULL DEFAULT 1 COMMENT '1=冻结中 2=已解冻 3=已确认扣款',
    freeze_time     bigint unsigned NOT NULL DEFAULT 0,
    resolve_time    bigint unsigned NOT NULL DEFAULT 0 COMMENT '解冻/确认时间',
    resolve_reason  varchar(256)    NOT NULL DEFAULT '',
    create_at       bigint unsigned NOT NULL DEFAULT 0,
    UNIQUE KEY uk_ref_order_wallet (ref_order_no, wallet_type),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
) ENGINE=InnoDB COMMENT='冻结明细表';
```

#### 表9：exchange_order（兑换订单）

```sql
CREATE TABLE exchange_order (
    id              bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_no        varchar(64)     NOT NULL COMMENT '兑换订单号',
    user_id         bigint unsigned NOT NULL,
    from_currency   varchar(10)     NOT NULL COMMENT '源币种(法币)',
    to_currency     varchar(10)     NOT NULL DEFAULT 'BSB' COMMENT '目标币种(BSB)',
    from_amount     decimal(20,4)   NOT NULL COMMENT '扣除法币金额',
    to_amount       decimal(20,4)   NOT NULL COMMENT '获得BSB金额',
    bonus_amount    decimal(20,4)   NOT NULL DEFAULT 0 COMMENT '赠送BSB金额',
    rate_snapshot   decimal(20,8)   NOT NULL COMMENT '兑换时汇率快照',
    anchor_snapshot decimal(20,8)   NOT NULL COMMENT 'BSB锚定比例快照',
    bonus_multiple  int             NOT NULL DEFAULT 0 COMMENT '赠送部分流水倍数',
    status          tinyint         NOT NULL DEFAULT 1 COMMENT '1=成功',
    create_at       bigint unsigned NOT NULL DEFAULT 0,
    UNIQUE KEY uk_order_no (order_no),
    INDEX idx_user_id (user_id),
    INDEX idx_create_at (create_at)
) ENGINE=InnoDB COMMENT='兑换订单表';
```

#### 表10：audit_requirement（流水审计要求）

```sql
CREATE TABLE audit_requirement (
    id              bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id         bigint unsigned NOT NULL,
    currency_code   varchar(10)     NOT NULL,
    source_order_no varchar(64)     NOT NULL COMMENT '来源订单号',
    source_type     tinyint         NOT NULL COMMENT '1=充值奖金 2=活动奖金 3=兑换赠送',
    bonus_amount    decimal(20,4)   NOT NULL COMMENT '奖金金额',
    required_turnover decimal(20,4) NOT NULL COMMENT '需完成流水=奖金×倍数',
    completed_turnover decimal(20,4) NOT NULL DEFAULT 0 COMMENT '已完成流水',
    multiple        int             NOT NULL COMMENT '流水倍数',
    status          tinyint         NOT NULL DEFAULT 0 COMMENT '0=进行中 1=已完成',
    create_at       bigint unsigned NOT NULL DEFAULT 0,
    update_at       bigint unsigned NOT NULL DEFAULT 0,
    UNIQUE KEY uk_source_order (source_order_no),
    INDEX idx_user_id_status (user_id, status)
) ENGINE=InnoDB COMMENT='流水审计要求表';
```

### 3.3 表设计决策说明

| 决策 | 选型 | 为什么不用替代方案 |
|------|------|------------------|
| 金额精度 | DECIMAL(20,4) | BIGINT 最小单位需要业务层反复换算，FLOAT/DOUBLE 有精度丢失。4位兼顾法币(2位)+BSB(2位)的存储精度 |
| 汇率精度 | DECIMAL(20,8) | 虚拟币汇率需要8位小数精度 |
| 时间字段 | bigint 毫秒 | 工程规范一致（全项目统一），JSON 序列化友好 |
| 主键 | bigint AUTO_INCREMENT | Snowflake ID 在 TiDB 下不如 AUTO_INCREMENT（TiDB 自动分配避免热点）|
| 软删除 | delete_at=0 | 工程规范统一（全项目统一 GORM Gen 模式） |
| 流水不可变 | INSERT-ONLY | 金融审计铁律：流水一旦写入不可修改，只能追加冲正记录 |
| 冻结明细独立表 | freeze_detail | wallet_account 只记总冻结额，明细在 freeze_detail 中管理（支持多笔并行冻结） |

---

## 四、接口全景

### 4.1 接口统计

| 维度 | 币种配置 | 钱包模块 | 合计 |
|------|---------|---------|------|
| C端 HTTP | 2 | 19 | **21** |
| B端 HTTP | 7 | 0 | **7** |
| RPC 暴露 | 3 | 9 | **12** |
| RPC 调用 | 3 | 4 | **7** |
| 定时任务 | 1 | 0 | **1** |
| **合计** | **16** | **32** | **48** |

### 4.2 RPC 暴露接口（12个，供其他模块调用）

#### 核心金融原语（5个写操作，全部 P0）

| # | 方法 | 说明 | bizType 枚举 | 幂等键 |
|---|------|------|-------------|--------|
| W-R1 | CreditWallet | 加款/上分 | 1=充值/2=充值奖金/3=补单/4=人工加款/5=活动奖金/6=兑换入/7=兑换赠送/8=打赏/9=返奖 | (refOrderNo, bizType) |
| W-R2 | DebitWallet | 扣款/下分 | 10=消费/11=兑换出/15=人工减款 | (refOrderNo, bizType) |
| W-R3 | FreezeBalance | 冻结余额 | — | refOrderNo |
| W-R4 | UnfreezeBalance | 解冻余额 | — | refOrderNo |
| W-R5 | ConfirmDebit | 确认扣款冻结额 | — | refOrderNo |

**互斥约束**：W-R4（解冻）和 W-R5（确认扣款）对同一笔冻结互斥，只能走一条路。

**安全约束（5个操作共性）**：
- 幂等性：唯一索引 (order_no, op_type, wallet_type)
- 原子性：余额变更 + 流水写入在同一 DB 事务
- 非负约束：SQL WHERE `available_amt >= amount` 防透支
- 并发控制：SELECT FOR UPDATE 行级锁
- 审计链：wallet_transaction 不可变 + ser-blog 审计日志

#### 管理操作（1个）

| # | 方法 | 说明 | 调用方 |
|---|------|------|--------|
| W-R6 | InitWallet | 用户注册初始化钱包 | ser-user |

#### 查询操作（3+3个）

| # | 方法 | 说明 | 优先级 |
|---|------|------|--------|
| W-R7 | GetBalance | 单用户余额查询 | P0 |
| W-R8 | BatchGetBalance | 批量用户余额查询 | P2 |
| W-R9 | GetWalletFlow | 交易流水分页查询 | P2 |
| CC-R1 | GetCurrencyList | 币种列表 | P0 |
| CC-R2 | GetExchangeRate | 汇率查询 | P1 |
| CC-R3 | GetBaseCurrency | 基准币种 | P2 |

### 4.3 RPC 调用接口（7个，依赖外部模块）

| # | 目标模块 | 方法 | 依赖强度 | 说明 |
|---|---------|------|---------|------|
| 1 | ser-user | GetUserInfoExpend | **强** | 校验用户状态/国家 |
| 2 | ser-user | BatchGetUserInfo | 标准 | B端列表批量查用户 |
| 3 | ser-kyc | KycQuery | **强** | 提现KYC校验+实名比对 |
| 4 | ser-blog | AddActLog (oneway) | 标准 | 审计日志（异步不阻塞） |
| 5 | ser-blog | QueryActs | 标准 | B端操作日志查询 |
| 6 | ser-s3 | UploadFile | 弱 | 币种图标上传 |
| 7 | ser-buser | QueryUserByIds | 弱 | B端操作人名称 |

**全部5个RPC Client已在 `common/rpc/rpc_client.go` 注册，无需新增。**

### 4.4 C端 HTTP 接口（21个）

| 分类 | 接口数 | 路由前缀 |
|------|--------|---------|
| 钱包基础 | 2 | `/api/wallet/balance`, `/api/wallet/currencies` |
| 充值 | 5 | `/api/wallet/deposit/*` |
| 兑换 | 2 | `/api/wallet/exchange/*` |
| 提现 | 5 | `/api/wallet/withdraw/*` |
| 记录查询 | 5 | `/api/wallet/records/*` |
| 币种查询 | 2 | `/api/wallet/currency/*` |

### 4.5 B端 HTTP 接口（7个）

| 接口 | 路由 | 说明 |
|------|------|------|
| B1 | `GET /admin/api/wallet/currency/list` | 币种列表分页 |
| B2 | `POST /admin/api/wallet/currency/create` | 新增币种 |
| B3 | `PUT /admin/api/wallet/currency/update` | 编辑币种 |
| B4 | `PUT /admin/api/wallet/currency/status` | 启禁用币种 |
| B5 | `PUT /admin/api/wallet/currency/base` | 设置基准币种 |
| B6 | `GET /admin/api/wallet/currency/rate/log` | 汇率日志 |
| B7 | `GET /admin/api/wallet/currency/detail` | 币种详情 |

**钱包模块不设 B 端接口**——B 端运营操作由 ser-finance 发起，通过 RPC 调用我们。

---

## 五、技术栈选型与工程规范

### 5.1 技术栈全景

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| HTTP框架 | CloudWeGo Hertz | 通过 gateway 代理，实际是 Kitex RPC |
| RPC框架 | CloudWeGo Kitex | Thrift IDL + TTHeader 协议 |
| IDL | Apache Thrift | 位于 `/common/idl/ser-wallet/` |
| 数据库 | TiDB（MySQL兼容） | GORM + GORM Gen 自动生成 |
| 缓存 | Redis（UniversalClient） | 支持单机/集群自适应 |
| 配置中心 | ETCD | LoadAndWatch 模式，动态配置 |
| 服务发现 | ETCD Resolver | Kitex 自动负载均衡 |
| 分布式ID | Snowflake | int64 单调递增（或 TiDB AUTO_INCREMENT）|
| 日志 | Klog（Kitex日志） | 结构化 JSON + TraceID 串联 |

### 5.2 工程分层（对齐现有工程规范）

```
ser-wallet/
├── main.go                     # 入口：ETCD初始化 → Redis → MySQL → 注册RPC服务
├── handler.go                  # RPC Service 实现，委托给 internal/ser
├── internal/
│   ├── cfg/                    # 配置初始化（redis.go, mysql.go）
│   ├── ser/                    # 业务逻辑层 ← 核心代码在这里
│   │   ├── wallet_service.go   # 钱包核心服务（6个金融原语）
│   │   ├── currency_service.go # 币种配置服务
│   │   ├── deposit_service.go  # 充值流程
│   │   ├── withdraw_service.go # 提现流程
│   │   ├── exchange_service.go # 兑换流程
│   │   └── record_service.go   # 记录查询
│   ├── rep/                    # 数据访问层（Repository）
│   ├── cache/                  # Redis 缓存层
│   ├── errs/                   # 服务错误码
│   ├── enum/                   # 枚举常量
│   ├── task/                   # 定时任务（汇率刷新）
│   ├── gen/                    # GORM Gen 生成入口
│   └── gorm_gen/               # 自动生成的模型和查询
│       ├── model/              # 表模型 *.gen.go
│       └── query/              # 查询方法 *.gen.go
└── tmp/                        # 临时文件
```

### 5.3 关键工程规范对齐

| 规范项 | 现有工程标准 | ser-wallet 遵循 |
|--------|------------|----------------|
| 错误码格式 | `6{NNN}{MMM}` (NNN=服务号) | `6{walletServiceNo}{modulNo}` |
| 响应格式 | `{traceId, code, msg, data}`，code=0成功 | 完全遵循 ret.HttpOne() |
| 时间字段 | int64 毫秒时间戳 | 完全遵循 |
| 软删除 | delete_at=0 未删除 | 完全遵循 |
| 审计字段 | create_by/create_by_name/update_by/update_by_name | B端表完全遵循 |
| RPC 客户端 | sync.Once 单例 + ETCD Resolver | 已注册，直接使用 |
| RPC 超时 | 3秒 | 遵循默认 |
| 日志 | `lg.Log().CtxErrorf(ctx, ...)` | 完全遵循 |
| 缓存 | Get/Set/DelAll 三操作 + 空结果缓存 | 完全遵循 |
| 分页 | `dto.PageResult[T]` 泛型分页 | 完全遵循 |
| 服务层 | 构造器注入 repo + cache | 完全遵循 |

### 5.4 Thrift IDL 组织规划

```
/common/idl/ser-wallet/
├── service.thrift          # 主入口：include 所有子文件，定义 WalletService
├── wallet.thrift           # 钱包基础模型：WalletBalanceInfo, WalletAccount
├── wallet_rpc.thrift       # RPC 请求/响应：CreditWalletReq/Resp 等12个
├── currency.thrift         # 币种模型：CurrencyInfo, ExchangeRateInfo
├── deposit.thrift          # 充值相关：DepositOrder, DepositMethod
├── withdraw.thrift         # 提现相关：WithdrawOrder, WithdrawAccount
├── exchange.thrift         # 兑换相关：ExchangeOrder, ExchangePreview
└── record.thrift           # 记录查询：RecordFilter, RecordItem
```

---

## 六、十大技术挑战与解法

### 挑战①：并发安全与防超卖 🔴

**场景**：同一用户同时发起提现和投注，或同一笔奖金被MQ重复投递。

**解法**：双层锁 + SQL原子操作

```
第一层：Redis 分布式锁（快速拦截）
  key = wallet:lock:{userId}:{walletType}:{currencyCode}
  SetNX + 10s TTL

第二层：DB 悲观锁（最终保障）
  SELECT * FROM wallet_account
  WHERE user_id = ? AND wallet_type = ? AND currency_code = ?
  FOR UPDATE

第三层：SQL 原子操作（防透支兜底）
  UPDATE wallet_account
  SET available_amt = available_amt - ?
  WHERE id = ? AND available_amt >= ?
  → affected_rows = 0 则余额不足
```

**为什么用悲观锁不用乐观锁**：钱包是写多场景（投注/充值/提现高频），乐观锁重试率高反而降低性能。悲观锁串行化虽有等待但保证一次成功。

**防死锁**：多子钱包扣款时按 walletType 固定顺序加锁（先中心后奖励），永远不交叉。

### 挑战②：数据一致性保障 🔴

**场景**：加款 + 流水写入必须同时成功或同时失败。

**解法**：本地事务（GORM Transaction）

```go
db.Transaction(func(tx *gorm.DB) error {
    // 1. SELECT FOR UPDATE 锁行
    // 2. UPDATE available_amt（SQL原子操作）
    // 3. INSERT wallet_transaction（流水）
    // 4. INSERT freeze_detail（冻结场景）
    // 全部在同一个 tx 中，要么全成功要么全回滚
    return nil
})
```

**不用 2PC/XA 的原因**：
- 钱包操作的余额和流水在同一个数据库同一个事务内，不需要分布式事务
- 跨模块协作（如提现涉及 ser-finance 审核）用 TCC 模式（冻结=Try，确认扣款=Confirm，解冻=Cancel）
- 充值入账由 ser-finance 回调触发，用幂等重试即可保证最终一致

### 挑战③：幂等性保障 🔴

**场景**：网络抖动导致 RPC 重试，同一笔充值被入账两次。

**解法**：双层幂等

```
第一层：Redis SetNX（毫秒级拦截99%重复请求）
  key = wallet:idempotent:{bizType}:{refOrderNo}
  TTL = 24h
  → SetNX 失败 = 重复请求，直接返回成功

第二层：DB 唯一索引（兜底保障）
  UNIQUE KEY uk_order_op_wallet (order_no, op_type, wallet_type)
  → 插入冲突 = 重复操作，捕获 DuplicateEntry 返回成功
```

**为什么两层不能只要一层**：
- 只有 Redis：Redis 故障/重启后丢失，无法保障
- 只有 DB：每次都到 DB 层检测，高并发下压力大
- 双层组合：Redis 挡住 99%+，DB 兜底 1%

### 挑战④：资金安全四层防线 🔴

```
┌─────────────────────────────────────────┐
│ 第一层：接入防线                          │
│  · 用户状态校验（封号禁止操作）            │
│  · KYC 校验（未认证限制提现渠道）          │
│  · 频率限制（防刷接口）                    │
├─────────────────────────────────────────┤
│ 第二层：业务逻辑防线                      │
│  · 金额非负校验（amount > 0）             │
│  · 余额充足校验（available_amt >= amount） │
│  · 限额校验（单笔/单日）                  │
│  · 姓名比对（持卡人 = KYC实名）           │
├─────────────────────────────────────────┤
│ 第三层：数据层防线                        │
│  · SQL原子操作（UPDATE WHERE >= amount）   │
│  · 悲观锁串行化（SELECT FOR UPDATE）      │
│  · 唯一索引幂等（防重复记账）             │
│  · 流水不可变（INSERT-ONLY）              │
├─────────────────────────────────────────┤
│ 第四层：事后防线                          │
│  · 每日对账（余额 = Σ流水验证）           │
│  · 异常告警（余额变为负数即告警）          │
│  · 审计日志（ser-blog 全操作留痕）         │
└─────────────────────────────────────────┘
```

### 挑战⑤：多币种与汇率管理 🟡

**精度方案**：

| 场景 | 存储精度 | 传输格式 |
|------|---------|---------|
| 金额（法币/BSB） | DECIMAL(20,4) | i64 最小单位 |
| 汇率 | DECIMAL(20,8) | string 小数字符串 |
| 计算中间值 | Go decimal 库 | shopspring/decimal |

**汇率快照锁定**：
- 用户发起充值/提现/兑换时，将当时汇率快照存入订单（rate_snapshot 字段）
- 后续所有计算基于快照汇率，不受实时汇率波动影响
- "所见即所得"——用户看到的汇率 = 最终结算汇率

**汇率定时刷新流程**：
1. 定时任务轮询 3 家三方 API
2. 取均值作为实时汇率
3. 计算偏差：|实时 - 平台| / 实时 × 100%
4. 偏差 ≥ 阈值 → 更新入款/出款汇率
5. 写入 exchange_rate_log 记录变更

### 挑战⑥：高性能与容量规划 🟢

**热点分析**：

| 操作 | 频率 | 瓶颈 | 优化策略 |
|------|------|------|---------|
| 余额查询 | 极高频 | DB 读 | Redis 缓存余额（30s TTL，写操作时主动失效）|
| 币种列表 | 高频 | DB 读 | Redis 缓存（5min TTL，变更时清除）|
| 汇率查询 | 高频 | DB 读 | Redis 缓存（跟随定时任务刷新）|
| 加款/扣款 | 中高频 | DB 写 | FOR UPDATE 行级锁（不锁表）|
| 流水写入 | 高频 | DB 写 | wallet_transaction 仅 INSERT，无锁竞争 |

**大表策略**：
- wallet_transaction 年增 5000万+，TiDB 自动分片处理
- 未来可按月归档：wallet_transaction_archive_YYYYMM
- 查询加 create_at 时间范围索引，避免全表扫描

### 挑战⑦：跨服务协调与容错 🟡

**重试策略**：

| 场景 | 策略 | 参数 |
|------|------|------|
| RPC 调用失败 | 幂等重试 | 2次，指数退避 100ms→200ms |
| ser-blog 写日志失败 | 仅 warn 不阻塞 | oneway 异步，无需重试 |
| ser-s3 上传失败 | 降级返回空URL | 不阻塞币种保存 |
| ser-finance 未开发 | TODO/Mock 占位 | 骨架先行 |

**超时设置**：
- RPC 默认 3s（工程规范）
- DB 事务超时 5s
- Redis 操作 2s

### 挑战⑧：冻结三阶段（TCC 模式） 🔴

```
提现场景 TCC 映射：

  Try    = FreezeBalance     (冻结金额，预留资源)
  Confirm = ConfirmDebit     (确认扣款，释放冻结)
  Cancel  = UnfreezeBalance  (解冻退回，恢复可用)

  ★ Confirm 和 Cancel 互斥：同一笔冻结只能走一条路
  ★ 通过 freeze_detail 表 + status 字段保证状态转换正确性
  ★ 状态转换：冻结中(1) → 已解冻(2) 或 已确认扣款(3)
```

**兑换场景**：纯本地事务（扣法币+加BSB+加赠送在同一个 tx 中），无需 TCC。

**充值场景**：ser-finance 回调触发 CreditWallet，用幂等保证不重复入账。

### 挑战⑨：可观测性与故障排查 🟡

**结构化日志**：
```go
lg.Log().CtxInfof(ctx, "CreditWallet userId=%d currency=%s walletType=%d "+
    "amount=%s bizType=%d refOrderNo=%s before=%s after=%s",
    req.UserId, req.CurrencyCode, req.WalletType,
    req.Amount, req.BizType, req.RefOrderNo,
    wallet.AvailableAmt, newAvailable)
```

**关键监控指标**：
- 余额变为负数 → 立即告警（理论不可能，出现说明有 BUG）
- 冻结超过 24h 未解决 → 提醒运营
- RPC 调用失败率 > 1% → 告警
- DB 事务耗时 > 1s → 告警

**审计链**：
- wallet_transaction（业务流水）+ ser-blog（操作日志）双轨审计
- 每笔流水含 traceId，可串联完整调用链

### 挑战⑩：可扩展性设计 🟢

**配置驱动而非代码驱动**：
- 新增币种 → currency_config 表加一条记录，无需改代码
- 调整汇率参数 → B端页面修改，无需重启
- 新增子钱包类型 → walletType 枚举扩展，wallet_account 加记录

**预留弹性**：
- walletType 预留了 4(代理) 和 5(场馆)，首版只实现 1-3
- bizType 枚举预留空间，新业务场景只需新增枚举值
- 金额精度 DECIMAL(20,4) 覆盖到百亿级别

---

## 七、核心业务流程状态机

### 7.1 充值订单状态机

```
┌──────────────┐
│  待支付 (0)   │  用户创建充值订单
└──────┬───────┘
       │ 用户发起支付
       ▼
┌──────────────┐
│  支付中 (1)   │  等待三方回调 / 链上确认
└──────┬───────┘
       │
       ├──── 超时未支付 ──→ 已关闭 (4)
       │
       ├──── 三方回调失败 ──→ 已失败 (3)
       │
       └──── 回调成功
             │
             ▼
      ┌──────────────┐
      │  已到账 (2)   │  CreditWallet 入账完成（终态）
      └──────────────┘
```

### 7.2 提现订单状态机（最复杂）

```
┌──────────────┐
│  已创建 (0)   │  用户提交提现申请
└──────┬───────┘
       │ 校验通过 + FreezeBalance
       ▼
┌──────────────┐
│  已冻结 (1)   │  余额已冻结，等待风控审核
└──────┬───────┘
       │
       ├──── 风控驳回 ──→ 已驳回(5) + UnfreezeBalance
       │
       ▼
┌──────────────┐
│  风控通过 (2)  │  等待财务审核
└──────┬───────┘
       │
       ├──── 财务驳回 ──→ 已驳回(5) + UnfreezeBalance
       │
       ▼
┌──────────────┐
│  出款中 (3)   │  已提交三方出款
└──────┬───────┘
       │
       ├──── 出款失败 ──→ 出款失败(6) + UnfreezeBalance
       │
       └──── 出款成功
             │
             ▼
      ┌──────────────┐
      │  已完成 (4)   │  ConfirmDebit 确认扣款（终态）
      └──────────────┘

驳回/失败(5/6) 均触发 UnfreezeBalance，资金退回可用
```

### 7.3 冻结生命周期

```
freeze_detail.status 转换：

  ┌───────────┐
  │ 冻结中 (1) │
  └─────┬─────┘
        │
    ┌───┴───┐
    │       │
    ▼       ▼
┌──────┐ ┌──────────┐
│解冻(2)│ │确认扣款(3)│
└──────┘ └──────────┘
(互斥：一条记录只能走其中一个终态)
```

---

## 八、跨模块协作与依赖

### 8.1 全景协作图

```
                          ┌─────────────────────┐
                          │      ser-wallet      │
                          │  (我们负责的模块)      │
                          └──────────┬──────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
        ▼                            ▼                            ▼
   ┌─────────┐              ┌──────────────┐            ┌──────────────┐
   │ 我们暴露  │              │ 我们调用的    │            │ 待对接的      │
   │ RPC(12)  │              │ RPC(7)       │            │ ser-finance   │
   └────┬────┘              └──────┬───────┘            └──────┬───────┘
        │                          │                           │
   ser-finance                ser-user(2)                  充值方式列表
   ser-user                   ser-kyc(1)                   提现方式列表
   投注/结算                  ser-blog(2)                  提现限额/费率
   活动系统                   ser-s3(1)                    提交提现审核
                              ser-buser(1)
```

### 8.2 关键调用链路

**充值入账链路**（ser-finance 驱动）：
```
三方回调成功 → ser-finance 验证 → RPC CreditWallet(bizType=1, 中心钱包)
                                → [有奖金时] RPC CreditWallet(bizType=2, 奖励钱包)
```

**提现全链路**（ser-wallet + ser-finance 协作）：
```
用户提交提现 → ser-wallet 校验(KYC+状态+余额)
            → ser-wallet FreezeBalance(冻结)
            → ser-wallet 创建 withdraw_order
            → ser-wallet 通知 ser-finance(TODO)
            → ser-finance 风控审核
              ├ 驳回 → ser-finance RPC UnfreezeBalance
              └ 通过 → ser-finance 财务审核
                ├ 驳回 → ser-finance RPC UnfreezeBalance
                └ 通过 → ser-finance 调用三方出款
                  ├ 失败 → ser-finance RPC UnfreezeBalance
                  └ 成功 → ser-finance RPC ConfirmDebit
```

**兑换链路**（纯内部）：
```
用户兑换法币→BSB → ser-wallet 内部事务:
  ① DebitWallet(法币中心钱包, bizType=11)
  ② CreditWallet(BSB中心钱包, bizType=6)
  ③ CreditWallet(BSB奖励钱包, bizType=7) [有赠送时]
  → 0 次外部 RPC 调用
```

### 8.3 ser-finance 未开发的阻塞项

| 阻塞项 | 影响范围 | 编码策略 |
|--------|---------|---------|
| 充值方式列表 | C端充值页面 | TODO + Mock 返回测试数据 |
| 提现方式列表 | C端提现页面 | TODO + Mock |
| 提现限额/费率 | C端提现限额展示 | TODO + Mock |
| 提交提现审核 | 提现流程最后一步 | TODO + Mock |

**策略**：这些是"缺火"级别——核心链路不完整但骨架照常实现，调用处留清晰标注。

---

## 九、需要与产品/评审确认的问题清单

### 9.1 业务规则确认

| # | 问题 | 背景 | 影响 |
|---|------|------|------|
| 1 | 充值回调归属 | 充值成功的三方回调由 ser-finance 接收还是 ser-wallet 接收？ | 决定回调接口放在哪个模块 |
| 2 | 奖金活动数据源 | 充值页面展示的奖金活动列表，数据由钱包模块还是活动模块提供？ | 影响接口归属和表结构 |
| 3 | 记录 Tab 合并 | 4 个记录 Tab（充值/提现/兑换/奖励）各有不同字段，是否确认独立接口？ | 接口数量和实现复杂度 |
| 4 | 场馆/代理钱包首版 | walletType=4(代理)/5(场馆) 首版是否需要实现？还是仅预留？ | 开发工作量 |
| 5 | 提现限额配置归属 | 单笔最小/最大、单日累计限额由钱包配置还是财务配置？ | 表结构和接口归属 |
| 6 | 手续费计算 | 提现手续费由钱包模块计算还是 ser-finance 告知？ | 业务逻辑放置位置 |
| 7 | 人工加减款审批流 | 人工加款/减款是否需要审批流？审批在 ser-finance 还是钱包？ | 当前假设在 ser-finance |
| 8 | 提现账户修改 | 需求说首次填写后锁定，但客服是否有后台修改入口？ | 是否需要 B 端修改接口 |

### 9.2 技术方案确认

| # | 问题 | 选项 | 建议 |
|---|------|------|------|
| 9 | 金额传输格式 | A) i64 最小单位 B) string 小数字符串 | 建议 A，减少精度问题 |
| 10 | 汇率刷新频率 | 需求未明确具体间隔 | 建议 5 分钟，可配置化 |
| 11 | 充值订单超时 | 需求未明确超时时间 | 建议 30 分钟，可配置 |
| 12 | 防重复充值阈值 | "1-2次软拦截，≥3次硬拦截"的具体时间窗口？ | 建议同币种同金额 5 分钟窗口 |
| 13 | 余额缓存策略 | 强一致 vs 最终一致 | 建议写操作主动失效 + 30s TTL |

### 9.3 跨模块对齐

| # | 问题 | 需要对齐方 |
|---|------|-----------|
| 14 | 订单号格式规范 | ser-finance：C/T/B/A/M + 16位是否确认？ |
| 15 | bizType 枚举值 | 全模块统一的操作类型枚举 |
| 16 | walletType 枚举值 | 全模块统一的钱包类型枚举 |
| 17 | 汇率精度标准 | ser-finance 侧汇率字段精度是否也用 DECIMAL(20,8)？ |
| 18 | KYC 状态码含义 | ser-kyc：status=1/2/3/4 的确切含义和转换条件 |

---

## 十、潜在提问与应对预案

### Q1：为什么金额不用 BIGINT 存最小单位？

**答**：两种方案都可行。我们选 DECIMAL(20,4) 的原因：
- 多币种精度不同（法币0-2位、BSB 2位），BIGINT 需要每次乘除换算，容易出错
- DECIMAL 在 SQL 层面直接可读（`WHERE available_amt >= 100.50` 比 `WHERE available_amt >= 10050` 直观）
- TiDB/MySQL 的 DECIMAL 运算是精确的，不存在浮点精度问题
- 传输层（Thrift IDL）使用 i64 最小单位或 string 小数，两层精度独立管理

### Q2：为什么用悲观锁不用乐观锁？

**答**：钱包是典型的**写多读少**且**写冲突概率高**的场景。
- 乐观锁：读 → 计算 → 写（CAS），冲突时重试。高并发下重试率高，性能反而差
- 悲观锁：SELECT FOR UPDATE 直接锁行，后续请求排队等待，每次都能一次成功
- 实际上：行级锁只锁同一用户同一币种同一子钱包的那一行，不同用户完全并行
- 用户级别的并发不高（同一个人不会同时做100笔操作），悲观锁等待时间极短

### Q3：如果 Redis 宕机了怎么办？

**答**：
- 分布式锁降级：Redis 不可用时，降级为 DB 悲观锁（SELECT FOR UPDATE 本身就能保证串行化）
- 幂等检测降级：Redis SetNX 失败时，DB 唯一索引兜底（双层设计的意义）
- 缓存降级：查询直接走 DB，只是性能下降，不影响正确性
- 核心原则：**Redis 是加速层，DB 是保障层**，两层都能独立工作

### Q4：余额怎么保证不会出现负数？

**答**：三层防护：
1. **业务层**：Go 代码校验 `if available < amount then reject`
2. **SQL 层**：`UPDATE ... SET available_amt = available_amt - ? WHERE available_amt >= ?`，受影响行数=0 则余额不足
3. **数据库约束**：可选加 CHECK 约束 `CHECK (available_amt >= 0)`（TiDB 支持）
4. **事后监控**：定时扫描 `available_amt < 0` 的记录，发现即告警

### Q5：冻结和扣款的区别是什么？

**答**：冻结不等于扣款。
- **冻结**：总余额不变，available_amt 减少，frozen_amt 增加。用户看到"冻结中"，钱还在账户里
- **确认扣款**：frozen_amt 减少，钱真正离开账户。总余额减少
- **解冻**：frozen_amt 减少，available_amt 增加。恢复可用，钱回到可用余额

这是 TCC 模式的实际应用：Try(冻结) → Confirm(确认扣款) / Cancel(解冻)。

### Q6：兑换为什么不需要外部 RPC？

**答**：兑换是法币→BSB 的内部换算，整个过程在 ser-wallet 内部完成：
1. 查汇率和锚定比例（内部表）
2. 扣法币中心钱包（内部操作）
3. 加 BSB 中心钱包（内部操作）
4. 加 BSB 赠送到奖励钱包（内部操作，如有）
5. 全部在同一个 DB 事务中

不需要调用外部服务，也不需要通道/三方。这是最"干净"的流程。

### Q7：wallet_transaction 年增 5000万条，怎么处理？

**答**：
- **当前**：TiDB 自动分片（Region Split），单表亿级无压力
- **中期**：查询加 create_at 索引 + 时间范围过滤，避免全表扫描
- **长期**：按月/季度归档到 wallet_transaction_archive_YYYYMM，近期数据热查，历史数据归档查
- **注意**：归档不是删除，而是迁移到归档表（保留审计链完整性）

### Q8：消费优先级"中心先扣奖励后补"怎么实现？

**答**：
```go
// 伪代码
func debitWithPriority(userId, currency, totalAmount) {
    // 1. 先尝试全部从中心钱包扣
    centerWallet := getWallet(userId, CENTER, currency)
    if centerWallet.available >= totalAmount {
        debit(centerWallet, totalAmount)
        return
    }
    // 2. 中心不够，中心全扣 + 奖励补差
    rewardWallet := getWallet(userId, REWARD, currency)
    centerDebit := centerWallet.available  // 中心能扣多少扣多少
    rewardDebit := totalAmount - centerDebit
    if rewardWallet.available < rewardDebit {
        return ErrInsufficientBalance  // 两个加起来也不够
    }
    // 3. 同一事务内扣两个钱包（固定顺序：先中心后奖励，防死锁）
    debit(centerWallet, centerDebit)
    debit(rewardWallet, rewardDebit)
}
```

### Q9：提现时怎么校验持卡人姓名？

**答**：
```go
// 提现账户的持卡人姓名必须与 KYC 实名一致
kycResp := rpc.KycClient().KycQuery(ctx, uid)
accountName := strings.TrimSpace(req.AccountName)
kycName := strings.TrimSpace(kycResp.Name)

if !strings.EqualFold(accountName, kycName) {
    return ErrNameMismatch  // "持卡人姓名与实名认证不一致"
}
```
- `EqualFold`：大小写不敏感比较
- `TrimSpace`：去除首尾空格
- 首次填写后锁定，后续提现自动回显，不允许自行修改

### Q10：ser-finance 还没开发，怎么推进？

**答**：分层解耦。
- **我们能独立完成的**：
  - 币种配置全套（B端+RPC+定时任务）
  - 6个金融原语 RPC（加/扣/冻/解/确/初始化）
  - 兑换全流程（纯内部）
  - 记录查询全套（纯查询）
  - 余额管理全套
- **需要 ser-finance 的部分**：
  - 充值方式列表、提现方式列表 → TODO/Mock
  - 充值三方回调 → 预留 CreditWallet 接口等调用
  - 提现审核流程 → 预留 Freeze/Unfreeze/Confirm 接口等调用
- **策略**：骨架先行，调用处写清 TODO 标注和 Mock 返回值，联调时替换即可

### Q11：参考实现里发现了哪些问题？我们怎么避免？

**答**：分析了两套参考实现（Java法币钱包 + TRC20虚拟币），共发现 12 个问题：

| 参考实现问题 | 我们的规避方案 |
|-------------|-------------|
| Java：coin_record.order_no 无唯一索引 | 加 UNIQUE KEY uk_order_op_wallet |
| Java：锁放在 Controller 层，Service 层裸奔 | 锁放在 Service 层 |
| Java：批量操作只锁第一个用户 | 每个用户独立加锁 |
| Java：MQ 消费者 75% 无锁保护 | 所有写入口统一走锁保护的 Service 方法 |
| Java：无 version 字段 | 加 version 乐观锁字段备用 |
| Java：MQ 无死信队列 | 用 NATS JetStream MaxDeliver + 错误主题 |
| TRC20：5步提现不在事务内 | 所有步骤同一事务 |
| TRC20：AES/ECB 弱加密 | 如有加密需求用 AES/GCM |
| TRC20：RSA 512位不够 | 最低 RSA 2048 |
| TRC20：硬编码 coin_name='USDT' | 参数化查询，支持多币种 |
| TRC20：gas fee 硬编码 | 配置化 |
| Java：DECIMAL(20,2) 精度不够 | 用 DECIMAL(20,4) + 汇率 DECIMAL(20,8) |

---

## 十一、编码节奏与落地策略

### 11.1 四梯队优先级

```
第一梯队：地基（必须先行，其他一切依赖它）
  ① 币种配置 B端 CRUD (B1-B6)
  ② 币种配置 RPC 暴露 (CC-R1, CC-R2)
  ③ wallet_account 表 + 6个金融原语 RPC (W-R1~R5, W-R7)
  ④ 汇率定时任务 (T1)

第二梯队：C端核心流程
  ⑤ 钱包基础 (W1, W2)
  ⑥ 兑换全流程 (W8, W9) — 无外部依赖，最干净
  ⑦ 记录查询 (W15-W19) — 纯查询，无外部依赖
  ⑧ 提现流程 (W10-W13) — 依赖 ser-kyc/ser-user（已有）

第三梯队：外部依赖项
  ⑨ 充值流程 (W3-W5) — 依赖 ser-finance (TODO/Mock)
  ⑩ 钱包初始化 (W-R6) — 依赖 ser-user 注册回调
  ⑪ 充值奖金/状态轮询 (W6, W7)

第四梯队：增强项
  ⑫ 币种详情 (B7), 基准币种 RPC (CC-R3)
  ⑬ BatchGetBalance, GetWalletFlow RPC
  ⑭ 提现限额独立接口 (W14)
```

### 11.2 编码方法论

```
每个模块的交付节奏：

  骨架阶段 → 编写接口签名 + 表模型 + Service 空方法
           → TODO 标注未实现部分
           → 能编译通过

  填充阶段 → 实现核心逻辑 + 单元测试
           → Mock 外部依赖
           → 能跑通 happy path

  完善阶段 → 补充 error path + 边界条件
           → 加缓存 + 日志 + 审计
           → 能跑通所有 case

  联调阶段 → 替换 Mock 为真实 RPC
           → 端到端验证
           → 能跑通全链路
```

### 11.3 风险识别

| 风险 | 概率 | 影响 | 预案 |
|------|------|------|------|
| ser-finance 开发延期 | 高 | 充值/提现C端流程不完整 | 骨架+Mock先行，不阻塞钱包核心 |
| 汇率三方API不稳定 | 中 | 汇率刷新失败 | 3源取均值 + 降级用上次成功值 |
| 币种精度争议 | 低 | 金额计算出错 | DECIMAL(20,4) + 传输层统一最小单位 |
| 大表性能 | 低（TiDB） | 查询变慢 | 时间索引 + 未来归档 |
| KYC/用户服务不可用 | 低 | 提现页面无法加载 | 超时降级 + 友好提示 |

---

## 附录A：订单号规范

| 类型 | 前缀 | 格式 | 示例 |
|------|------|------|------|
| 充值 | C | C + 16位数字 | C1234567890123456 |
| 提现 | T | T + 16位数字 | T1234567890123456 |
| 补单 | B | B + 16位数字 | B1234567890123456 |
| 人工加款 | A | A + 16位数字 | A1234567890123456 |
| 人工减款 | M | M + 16位数字 | M1234567890123456 |
| 兑换 | E | E + 16位数字 | E1234567890123456 |

## 附录B：bizType 操作类型枚举

| 值 | 含义 | 方向 | 目标钱包 |
|----|------|------|---------|
| 1 | 充值入账 | 收入 | 中心 |
| 2 | 充值奖金 | 收入 | 奖励 |
| 3 | 补单 | 收入 | 中心 |
| 4 | 人工加款 | 收入 | 指定 |
| 5 | 活动奖金 | 收入 | 奖励 |
| 6 | 兑换入 | 收入 | 中心 |
| 7 | 兑换赠送 | 收入 | 奖励 |
| 8 | 打赏收入 | 收入 | 主播 |
| 9 | 投注返奖 | 收入 | 按比例分配 |
| 10 | 消费扣款 | 支出 | 优先中心 |
| 11 | 兑换出 | 支出 | 中心 |
| 12 | 提现冻结 | 冻结 | 优先中心 |
| 13 | 提现确认 | 确认扣款 | 对应冻结 |
| 14 | 提现解冻 | 解冻 | 对应冻结 |
| 15 | 人工减款 | 支出 | 指定 |

## 附录C：错误码规划

```
ser-wallet 服务号待分配（假设为 015）

6015000 — 基础错误
6015001 — 参数校验失败
6015002 — 用户不存在
6015003 — 钱包未初始化

6015100 — 余额操作错误
6015101 — 余额不足
6015102 — 冻结金额不足
6015103 — 重复操作（幂等拦截）
6015104 — 冻结记录不存在
6015105 — 冻结已完结（重复解冻/确认）

6015200 — 充值错误
6015201 — 充值订单不存在
6015202 — 充值订单已关闭
6015203 — 重复充值拦截

6015300 — 提现错误
6015301 — KYC未认证（仅USDT可用）
6015302 — 持卡人姓名不一致
6015303 — 超过单笔限额
6015304 — 超过单日累计限额
6015305 — 提现账户不存在
6015306 — 用户已封号

6015400 — 兑换错误
6015401 — 源币种未启用
6015402 — 汇率已失效（请刷新）

6015500 — 币种配置错误
6015501 — 币种代码已存在
6015502 — 币种不存在
6015503 — 基准币种已设置不可修改
6015504 — 币种已启用不可删除
6015505 — 币种名称已存在
```
