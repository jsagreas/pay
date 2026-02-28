# ser-wallet 表结构设计分析

> 设计时间：2026-02-26
> 分析视角：从三大模块（钱包、币种配置、财务）的业务需求出发，逐表论证"为什么需要这张表、它存什么、字段怎么定"
> 分析维度：必须（没有它业务跑不通）、应该（正常系统都会有）、可选（评审后决定）
> 设计依据：
> - 需求理解：/Users/mac/gitlab/z-readme/需求理解/tmp-1.md
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-1.md
> - 架构设计：/Users/mac/gitlab/z-readme/架构设计/tmp-1.md
> - 流程思路：/Users/mac/gitlab/z-readme/流程思路/tmp-1.md
> - 工程认知：/Users/mac/gitlab/z-readme/工程认知/tmp-1.md
> 工程约束：TiDB(MySQL兼容) + GORM Gen + 软删除(delete_at=0) + int64时间戳 + utf8mb4

---

## 目录

1. [分析方法论](#一分析方法论)
2. [表结构全景清单](#二表结构全景清单)
3. [必须级：币种配置表 t_currency](#三必须级币种配置表)
4. [必须级：汇率日志表 t_rate_log](#四必须级汇率日志表)
5. [必须级：子钱包账户表 t_wallet_account](#五必须级子钱包账户表)
6. [必须级：资金流水表 t_wallet_flow](#六必须级资金流水表)
7. [必须级：充值订单表 t_deposit_order](#七必须级充值订单表)
8. [必须级：提现订单表 t_withdraw_order](#八必须级提现订单表)
9. [必须级：兑换订单表 t_exchange_order](#九必须级兑换订单表)
10. [必须级：冻结记录表 t_freeze_record](#十必须级冻结记录表)
11. [必须级：提现账户表 t_withdraw_account](#十一必须级提现账户表)
12. [应该级：财务模块相关表分析](#十二应该级财务模块相关表分析)
13. [可选级：扩展表分析](#十三可选级扩展表分析)
14. [公共字段规范](#十四公共字段规范)
15. [枚举值速查表](#十五枚举值速查表)
16. [表间关系总图](#十六表间关系总图)
17. [建表顺序与依赖](#十七建表顺序与依赖)

---

## 一、分析方法论

### 1.1 怎么判断一张表是否"必须"

```
一张表是否必须存在，取决于三个问题：

Q1: 有没有业务场景必须读写这张表的数据？
    → 如果有明确的需求功能点依赖它 → 必须

Q2: 如果不建这张表，能否用其他表的字段兼容？
    → 如果能兼容且不引入复杂性 → 可以合并
    → 如果强行合并会导致数据冗余或查询困难 → 必须独立

Q3: 这张表的数据的生命周期和写入频率是否与其他表不同？
    → 生命周期或写入模式明显不同的数据应该独立建表
```

### 1.2 三个级别的定义

| 级别 | 含义 | 判断依据 |
|------|------|---------|
| **必须** | 没有这张表，核心业务流程跑不通 | 需求文档中有明确功能依赖 |
| **应该** | 正常系统会有，缺了会导致管理困难 | 行业惯例或运维需要 |
| **可选** | 当前阶段可以不做，未来可能需要 | 预留功能或评审后再定 |

### 1.3 分析范围

本文覆盖三大模块的全部表结构需求：

```
┌─────────────────────────────────────────────────────────────┐
│  db_wallet（我负责建表）                                       │
│  ├── 币种配置域：t_currency, t_rate_log                       │
│  ├── 钱包核心域：t_wallet_account, t_wallet_flow,            │
│  │              t_freeze_record                              │
│  ├── 充值域：t_deposit_order                                  │
│  ├── 提现域：t_withdraw_order, t_withdraw_account            │
│  └── 兑换域：t_exchange_order                                 │
│                                                              │
│  db_finance（财务负责人建表，我需要了解）                       │
│  ├── t_collect_channel（代收通道）                             │
│  ├── t_payout_channel（代付通道）                              │
│  ├── t_deposit_method_config（充值方式配置）                   │
│  ├── t_withdraw_method_config（提现方式配置）                  │
│  └── t_correction_order（人工修正订单）                        │
│                                                              │
│  可选/预留                                                    │
│  ├── t_audit_config（稽核流水配置）                            │
│  ├── t_deposit_address（USDT充值地址池）                      │
│  └── t_daily_statistics（每日统计快照）                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、表结构全景清单

### 2.1 按级别分类

| 级别 | 所属库 | 表名 | 现实含义 | 需求来源 |
|------|--------|------|---------|---------|
| **必须** | db_wallet | t_currency | 平台支持哪些货币 | 币种配置需求24张图 |
| **必须** | db_wallet | t_rate_log | 每次汇率更新的记录 | 需求文档3.4汇率更新+3.7汇率日志 |
| **必须** | db_wallet | t_wallet_account | 用户×币种×子钱包的余额 | 需求文档2.1五种子钱包 |
| **必须** | db_wallet | t_wallet_flow | 每笔资金变动的审计记录 | 需求文档4.9对账一致性 |
| **必须** | db_wallet | t_deposit_order | 充值订单 | 钱包需求42张图 |
| **必须** | db_wallet | t_withdraw_order | 提现订单 | 钱包需求42张图 |
| **必须** | db_wallet | t_exchange_order | 兑换订单 | 需求文档4.6兑换 |
| **必须** | db_wallet | t_freeze_record | 提现冻结生命周期 | 需求文档4.7.6冻结流程 |
| **必须** | db_wallet | t_withdraw_account | 用户绑定的提现账户 | 需求文档4.7.3+4.7.4 |
| **应该** | db_finance | t_collect_channel | 代收支付通道 | 财务需求5.2 |
| **应该** | db_finance | t_payout_channel | 代付支付通道 | 财务需求5.2 |
| **应该** | db_finance | t_deposit_method_config | 充值方式配置 | 财务需求5.4 |
| **应该** | db_finance | t_withdraw_method_config | 提现方式配置 | 财务需求5.4 |
| **应该** | db_finance | t_correction_order | 人工修正订单 | 财务需求5.7 |
| **可选** | db_wallet | t_audit_config | 稽核流水规则配置 | 需求文档2.4稽核 |
| **可选** | db_wallet | t_deposit_address | USDT充值地址池 | 需求文档4.3.1 |
| **可选** | db_wallet/db_finance | t_daily_statistics | 每日统计聚合快照 | 财务需求5.8统计 |
| **可选** | db_wallet | t_bonus_activity | 额外奖金活动配置 | 需求文档4.5额外奖金 |

---

## 三、必须级：币种配置表

### t_currency — 平台支持的所有货币

**为什么必须**：
```
整个系统的所有金额操作都依赖币种配置：
- 充值页面需要知道当前币种的汇率 → 读 t_currency.deposit_rate
- 提现页面需要知道精度和手续费 → 读 t_currency.precision
- 兑换需要汇率换算 → 读 t_currency.platform_rate
- C端钱包页切换币种 → 读 t_currency（已启用列表）
- B端运营管理币种 → CRUD t_currency

没有这张表，系统连"用什么钱"都不知道。它是地基中的地基。
```

**需求追溯**：需求文档3.2三类币种 + 3.4汇率体系 + 3.5精度规则 + 3.6金额显示规则 + 3.7后台配置页

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `currency_code` | varchar(16) | 是 | 币种代码如VND/USDT/BSB | **唯一标识**，所有关联表都用这个字段关联 |
| `currency_name` | varchar(64) | 是 | 币种名称 | B端列表展示+C端展示 |
| `currency_type` | tinyint | 是 | 1法币/2加密/3平台 | 业务规则区分：法币2位精度、加密8位精度；兑换仅法币→BSB |
| `country` | varchar(32) | 否 | 所属国家 | 法币关联国家（VND→越南），加密和平台币无国家 |
| `icon_url` | varchar(512) | 否 | 图标URL | C端钱包页展示币种图标（SVG/PNG≤5KB，经ser-s3上传） |
| `currency_symbol` | varchar(16) | 是 | 货币符号如₫ Rp ฿ | C端金额展示前缀/后缀 |
| `thousands_sep` | varchar(4) | 是 | 千分位符号 | 需求文档3.6：VND/IDR用`.` THB/BSB用`,` |
| `decimal_sep` | varchar(4) | 是 | 小数点符号 | 需求文档3.6：格式化金额显示 |
| `precision` | tinyint | 是 | 小数位数 | 需求文档3.5：法币=2位，加密=8位，用于展示截断和运算舍入 |
| `real_rate` | decimal(30,8) | 是 | 实时汇率 | 需求文档3.4：3家API取平均值，定时任务刷新 |
| `platform_rate` | decimal(30,8) | 是 | 平台汇率 | 需求文档3.4：系统内部实际使用的汇率，偏差达阈值才更新 |
| `rate_fluctuation` | decimal(10,4) | 是 | 汇率浮动比例(%) | 需求文档3.4：入款/出款汇率的浮动计算因子 |
| `rate_threshold` | decimal(10,4) | 是 | 偏差阈值(%) | 需求文档3.4：偏差≥阈值才触发平台汇率更新 |
| `deposit_rate` | decimal(30,8) | 是 | 入款汇率 | = platform_rate × (1 + fluctuation%)，充值时使用 |
| `withdraw_rate` | decimal(30,8) | 是 | 出款汇率 | = platform_rate × (1 - fluctuation%)，提现时使用 |
| `is_base` | tinyint | 是 | 是否基准币种 | 需求文档3.3：USDT为基准，整表仅一条为1，一次性设置不可改 |
| `sort_value` | int | 是 | 排序值 | C端列表按此排序展示 |
| `status` | tinyint | 是 | 1启用/2禁用 | 需求文档3.1：禁用后C端不可见；基准币种不允许禁用 |
| `create_by` | bigint | 是 | 创建人 | 工程规范：B端操作审计 |
| `update_by` | bigint | 是 | 修改人 | 工程规范 |
| `create_at` | bigint | 是 | 创建时间 | 工程规范 |
| `update_at` | bigint | 是 | 更新时间 | 工程规范 |
| `delete_at` | bigint | 是 | 软删除 | 工程规范：0=未删除 |

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `uk_currency_code` | (currency_code) | UNIQUE | 币种代码全局唯一，需求硬要求"不可重复" |
| `idx_status_sort` | (status, sort_value) | INDEX | C端 ListActiveCurrencies 查 status=1 + 按sort排序 |

**核心约束**：
- `currency_code` 创建后不可修改（需求文档："币种代码唯一不可重复"）
- `currency_type` 创建后不可修改
- `is_base=1` 的记录不可禁用（需求文档："基准币种无法禁用"）
- `is_base=1` 整表最多一条（需求文档："只能设置一次，不可修改"）

---

## 四、必须级：汇率日志表

### t_rate_log — 每次汇率自动更新的历史记录

**为什么必须**：
```
需求文档3.7明确要求后台"汇率日志"Tab页面，展示每次平台汇率更新的记录。
这是B端运营的审计需求——需要看到"汇率什么时候变了、变成了多少"。

如果不建这张表：
- B端汇率日志页面没有数据源
- 运营无法追溯汇率变更历史
- 出现汇率争议时无法举证
```

**需求追溯**：需求文档3.4汇率更新机制 + 3.7后台"汇率日志"Tab

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `currency_code` | varchar(16) | 是 | 变更的币种 | B端按币种筛选日志 |
| `real_rate_before` | decimal(30,8) | 是 | 更新前实时汇率 | 追溯变更前后差异 |
| `real_rate_after` | decimal(30,8) | 是 | 更新后实时汇率（本次API平均值） | 需求文档3.7字段"实时汇率" |
| `platform_rate_before` | decimal(30,8) | 是 | 更新前平台汇率 | 追溯变更幅度 |
| `platform_rate_after` | decimal(30,8) | 是 | 更新后平台汇率 | 需求文档3.7字段"平台汇率" |
| `deposit_rate` | decimal(30,8) | 是 | 更新后入款汇率 | 需求文档3.7字段"入款汇率" |
| `withdraw_rate` | decimal(30,8) | 是 | 更新后出款汇率 | 需求文档3.7字段"出款汇率" |
| `deviation_pct` | decimal(10,4) | 是 | 触发此次更新的偏差百分比 | 审计：证明此次更新确实因偏差达阈值触发 |
| `trigger_type` | tinyint | 是 | 1自动/2手动 | 区分定时任务自动触发和后续可能的手动触发 |
| `create_at` | bigint | 是 | 日志时间 | 需求文档3.7字段"时间" |
| `delete_at` | bigint | 是 | 软删除 | 工程规范 |

**特性**：只插入不更新——日志是不可变审计记录

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `idx_currency_create` | (currency_code, create_at DESC) | INDEX | B端按币种筛选+时间倒序翻页 |
| `idx_create_at` | (create_at DESC) | INDEX | B端不带筛选条件时按时间翻页 |

---

## 五、必须级：子钱包账户表

### t_wallet_account — 用户在某币种下某子钱包的余额

**为什么必须**：
```
这是整个钱包系统的心脏——"用户有多少钱"。

需求文档2.1明确定义了五种子钱包（中心/奖励/主播/代理/场馆），
需求文档2.5明确要求"每个币种独立一套子钱包"。

所有资金操作的最终效果都是修改这张表的余额字段：
  充值成功 → available_balance += amount
  提现冻结 → available_balance -= amount, frozen_balance += amount
  提现扣款 → frozen_balance -= amount
  兑换    → 源币种 available -= amount, 目标币种 available += amount

如果不建这张表，用户余额无处存储，整个钱包系统不存在。
```

**需求追溯**：需求文档2.1五种子钱包 + 2.2余额计算 + 2.5每个币种独立

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `user_id` | bigint | 是 | 用户ID | 账户三维唯一的维度1 |
| `currency_code` | varchar(16) | 是 | 币种代码 | 维度2："每个币种独立一套" |
| `wallet_type` | tinyint | 是 | 子钱包类型(1中心/2奖励/3主播/4代理/5场馆) | 维度3："五种子钱包各有独立规则" |
| `available_balance` | decimal(30,8) unsigned | 是 | 可用余额 | 用户当前能操作的钱 |
| `frozen_balance` | decimal(30,8) unsigned | 是 | 冻结余额 | 提现审核中暂不可操作的钱 |
| `version` | bigint | 是 | 乐观锁版本号 | 架构决策ADR-03：并发控制，每次余额变动version+1 |
| `create_at` | bigint | 是 | 创建时间 | 工程规范 |
| `update_at` | bigint | 是 | 更新时间 | 工程规范 |
| `delete_at` | bigint | 是 | 软删除 | 工程规范 |

**为什么 available_balance 和 frozen_balance 要分开**：
```
需求文档4.7.6提现流程：
  "用户发起→钱包冻结对应金额→审核→出款→成功/失败"

如果只有一个 balance 字段：
  用户提现1000 → 余额减少1000 → 审核驳回 → 余额恢复1000
  问题：如果在审核期间用户又消费了，那"恢复"回来的钱从哪扣？

分开存储：
  提现1000 → available -= 1000, frozen += 1000
  审核期间用户只能操作 available，frozen 不可碰
  驳回 → available += 1000, frozen -= 1000（精确归还）
  成功 → frozen -= 1000（直接扣除冻结金额）

这是资金安全的基本要求，业界所有钱包系统都这样设计。
```

**为什么需要 version 字段**：
```
架构决策ADR-03选择了乐观锁。

场景：两个请求同时给用户充值100
  请求A读到余额=500, version=10
  请求B读到余额=500, version=10
  请求A写入：UPDATE SET balance=600, version=11 WHERE version=10 → 成功
  请求B写入：UPDATE SET balance=600, version=11 WHERE version=10 → 失败（version已变）
  请求B重试：重新读取余额=600, version=11，写入700, version=12 → 成功

没有version字段 → 两个请求都写入600 → 丢失一笔充值 → 资金安全事故
```

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `uk_user_currency_type` | (user_id, currency_code, wallet_type) | UNIQUE | **核心约束**：一个用户一个币种一种子钱包只能有一个账户 |
| `idx_user_currency` | (user_id, currency_code) | INDEX | C端钱包概览：查某用户某币种下所有子钱包余额 |

---

## 六、必须级：资金流水表

### t_wallet_flow — 每一笔余额变动的不可变审计记录

**为什么必须**：
```
需求文档4.9明确要求："对账一致性：账务流水与账户余额必须强一致性校验"
需求文档7.7："流水与余额强一致性"
架构决策ADR-04："流水表唯一索引做幂等保障"

这张表承担三个关键职责：
1. 审计记录 — 每笔钱的变动都有据可查（"谁的钱、为什么动、变了多少"）
2. 对账依据 — balance_after 必须等于当前余额（发现不一致=有bug或被攻击）
3. 幂等保障 — (related_order_no, flow_type) 唯一索引防止重复入账

如果不建这张表：
- 无法对账 → 不知道余额对不对
- 无法幂等 → 重复回调导致多次入账 → 资金凭空增加
- 无法审计 → 出问题时无法追溯每一分钱的去向
```

**需求追溯**：需求文档4.9 + 7.7 + 架构设计4.2 + ADR-04

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `user_id` | bigint | 是 | 用户ID | 查某用户的流水 |
| `currency_code` | varchar(16) | 是 | 币种 | 流水按币种隔离 |
| `wallet_type` | tinyint | 是 | 子钱包类型 | 精确到哪个子钱包的变动 |
| `flow_type` | tinyint | 是 | 流水类型（枚举14种） | 区分变动原因（充值/奖金/冻结/扣款/...），幂等索引的组成部分 |
| `amount` | decimal(30,8) **SIGNED** | 是 | 变动金额(正增负减) | 核心：这笔操作动了多少钱 |
| `balance_before` | decimal(30,8) unsigned | 是 | 变动前可用余额 | 对账：balance_before + amount = balance_after |
| `balance_after` | decimal(30,8) unsigned | 是 | 变动后可用余额 | 对账：最后一条流水的after应等于当前余额 |
| `frozen_before` | decimal(30,8) unsigned | 是 | 变动前冻结余额 | freeze/unfreeze/deduct操作会改冻结余额，需要快照 |
| `frozen_after` | decimal(30,8) unsigned | 是 | 变动后冻结余额 | 同上 |
| `related_order_no` | varchar(32) | 是 | 关联订单号 | 按订单追溯流水；幂等索引的组成部分 |
| `related_biz_type` | tinyint | 是 | 关联业务(1充值/2提现/3兑换/4修正) | 分类查询 |
| `remark` | varchar(512) | 否 | 备注 | 人工加减款的原因说明 |
| `create_at` | bigint | 是 | 创建时间 | 工程规范 |

**注意**：此表没有 `update_at` 和 `delete_at`。流水一旦写入就不可修改、不可删除，这是审计的基本要求。`amount` 是唯一的 **SIGNED** 金额字段（可以为负表示减少）。

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `uk_order_flow` | (related_order_no, flow_type) | UNIQUE | **幂等核心**：同一订单+同一流水类型只能有一条记录，防重复入账 |
| `idx_user_currency_time` | (user_id, currency_code, wallet_type, create_at DESC) | INDEX | C端记录查询+对账按时间序列 |
| `idx_order_no` | (related_order_no) | INDEX | 按订单号追溯所有关联流水（一笔充值可能有入账+奖金两条流水） |

**为什么幂等索引是 (order_no, flow_type) 而不是仅 order_no**：
```
一笔充值订单 C123 可能产生两条流水：
  flow(C123, flow_type=1充值入账) → 1000入中心钱包
  flow(C123, flow_type=2奖金入账) → 500入奖励钱包

如果唯一索引仅 order_no → 第二条流水会被当成"重复"而拒绝 → 奖金无法入账
所以必须 (order_no + flow_type) 联合唯一，同一订单的不同类型流水各自独立幂等
```

---

## 七、必须级：充值订单表

### t_deposit_order — 每笔充值操作的完整记录

**为什么必须**：
```
充值是钱进入系统的入口。需求文档4.3定义了三种充值方式（USDT/银行/电子钱包），
需求文档4.4定义了重复转账拦截，需求文档4.5定义了额外奖金。

没有充值订单表：
- 无法记录充值状态（待支付→处理中→完成/失败）
- 无法实现重复转账拦截（需要查"同用户+同币种+同金额+待支付"的订单数）
- C端充值记录Tab无数据
- 三方回调找不到对应订单
```

**需求追溯**：需求文档4.3充值模块 + 4.4重复拦截 + 4.5额外奖金 + 4.8记录 + 5.9订单号

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `order_no` | varchar(32) | 是 | 订单号(C+16位) | 需求文档5.9订单号规范，全局唯一标识 |
| `user_id` | bigint | 是 | 用户ID | 核心维度 |
| `currency_code` | varchar(16) | 是 | 充值币种 | 需求文档4.2"按币种隔离" |
| `deposit_method` | tinyint | 是 | 1USDT/2银行/3电子钱包 | 需求文档4.3三种方式，技术实现路径不同 |
| `amount` | decimal(30,8) unsigned | 是 | 充值金额 | 核心字段 |
| `actual_amount` | decimal(30,8) unsigned | 是 | 实际到账金额 | 扣除手续费后的实际到账 |
| `usdt_amount` | decimal(30,8) unsigned | 否 | USDT换算金额 | 仅USDT充值场景：需求文档4.3.1"系统根据汇率换算USDT" |
| `exchange_rate` | decimal(30,8) unsigned | 否 | 使用汇率快照 | 汇率随时变，需在下单时快照固定 |
| `network_type` | tinyint | 否 | 1TRC20/2ERC20/3BEP20 | 仅USDT：需求文档4.3.1"网络选择" |
| `tx_hash` | varchar(128) | 否 | 链上交易哈希 | 仅USDT到账后填入，区块链确认凭证 |
| `deposit_address` | varchar(128) | 否 | USDT收款地址 | 仅USDT：需求文档4.3.1"展示收款地址" |
| `channel_id` | bigint | 否 | 支付通道ID | 银行/电子钱包场景由finance分配通道 |
| `channel_order_no` | varchar(64) | 否 | 三方通道订单号 | 三方回调时用此字段匹配；对账使用 |
| `bonus_activity_id` | bigint | 否 | 额外奖金活动ID | 需求文档4.5：0=无奖金，与活动模块关联 |
| `bonus_amount` | decimal(30,8) unsigned | 否 | 奖金金额 | 需求文档4.5"奖金金额→奖励钱包" |
| `bonus_turnover_multi` | decimal(10,2) unsigned | 否 | 奖金流水倍数 | 需求文档4.5"奖金需完成N倍流水" |
| `status` | tinyint | 是 | 1待支付/2处理中/3完成/4失败 | 充值状态机，需求文档4.8记录展示 |
| `fail_reason` | varchar(256) | 否 | 失败原因 | 需求文档4.8"失败原因分类：支付超时/支付异常/卡片问题" |
| `paid_at` | bigint | 否 | 支付完成时间 | C端详情页展示 |
| `create_at` | bigint | 是 | 下单时间 | 工程规范 |
| `update_at` | bigint | 是 | 更新时间 | 工程规范 |
| `delete_at` | bigint | 是 | 软删除 | 工程规范 |

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `uk_order_no` | (order_no) | UNIQUE | 订单号全局唯一 |
| `idx_user_currency_status` | (user_id, currency_code, status) | INDEX | C端充值记录列表 |
| `idx_user_currency_amount_status` | (user_id, currency_code, amount, status) | INDEX | **重复转账拦截**：查同user+同币种+同金额+status∈(1,2) |
| `idx_channel_order` | (channel_order_no) | INDEX | 三方回调时按通道订单号查找 |
| `idx_create_at` | (create_at DESC) | INDEX | B端按时间倒序查询 |

---

## 八、必须级：提现订单表

### t_withdraw_order — 每笔提现的完整记录（含双审核流程）

**为什么必须**：
```
提现是钱离开系统的出口，是合规要求最多、流程最复杂的模块。
需求文档5.6定义了双审核流程（风控→财务），需要记录每个审核环节的人、时间、备注。

没有提现订单表：
- 无法追踪提现生命周期（审核中→通过/驳回→出款→成功/失败）
- 无法记录审核意见
- finance模块无法关联提现单进行审核操作
```

**需求追溯**：需求文档4.7提现模块 + 5.6双审核 + 4.8记录

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `order_no` | varchar(32) | 是 | 订单号(T+16位) | 需求文档5.9 |
| `user_id` | bigint | 是 | 用户ID | 核心维度 |
| `currency_code` | varchar(16) | 是 | 提现币种 | 币种隔离 |
| `wallet_type` | tinyint | 是 | 来源子钱包 | 从哪个子钱包提现（通常中心钱包，主播可从主播钱包提） |
| `withdraw_method` | tinyint | 是 | 1银行卡/2电子钱包/3USDT | 需求文档4.7.3三种方式 |
| `withdraw_account_id` | bigint | 是 | 提现账户ID | 关联 t_withdraw_account |
| `amount` | decimal(30,8) unsigned | 是 | 提现金额 | 核心字段 |
| `fee_rate` | decimal(10,4) unsigned | 是 | 手续费率(%) | 需求文档4.7.5"手续费率" |
| `fee_amount` | decimal(30,8) unsigned | 是 | 手续费 | 需求文档4.7.3"手续费" |
| `actual_amount` | decimal(30,8) unsigned | 是 | 到账金额=amount-fee | 需求文档4.7.3"到账金额" |
| `exchange_rate` | decimal(30,8) unsigned | 否 | 使用汇率 | BSB提现时需汇率换算 |
| `freeze_record_id` | bigint | 是 | 冻结记录ID | 关联 t_freeze_record，一一对应 |
| `status` | tinyint | 是 | 8种状态（见枚举） | 提现状态机 |
| `risk_reviewer_id` | bigint | 否 | 风控审核人 | 需求文档5.6"风控审核" |
| `risk_review_at` | bigint | 否 | 风控审核时间 | 审核时间线追溯 |
| `risk_review_remark` | varchar(512) | 否 | 风控备注 | 需求文档5.6"驳回需填写备注" |
| `finance_reviewer_id` | bigint | 否 | 财务审核人 | 需求文档5.6"财务审核" |
| `finance_review_at` | bigint | 否 | 财务审核时间 | 审核时间线追溯 |
| `finance_review_remark` | varchar(512) | 否 | 财务备注 | 需求文档5.6 |
| `channel_id` | bigint | 否 | 代付通道ID | 出款使用的通道 |
| `channel_order_no` | varchar(64) | 否 | 三方通道订单号 | 对账使用 |
| `fail_reason` | varchar(256) | 否 | 失败原因 | 需求文档4.8"失败原因" |
| `paid_at` | bigint | 否 | 出款完成时间 | C端展示 |
| `create_at` | bigint | 是 | 下单时间 | 工程规范 |
| `update_at` | bigint | 是 | 更新时间 | 工程规范 |
| `delete_at` | bigint | 是 | 软删除 | 工程规范 |

**为什么审核字段放在订单表里而不独立建审核表**：
```
分析了两种方案：

方案A：审核字段内嵌订单表（当前选择）
  优点：查订单时一次查询就拿到全部信息，简单高效
  缺点：如果未来审核环节增多（如增加第三道审核），需要加字段

方案B：独立建 t_withdraw_review 审核记录表
  优点：审核环节可灵活扩展
  缺点：查订单详情需要 JOIN 或多次查询

选择方案A的依据：
  - 当前需求明确只有两道审核（风控+财务），不会频繁变化
  - 一笔提现最多只有2条审核记录，数据量不大
  - 避免过度设计——如果未来真的加第三道，再迁移也不晚
```

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `uk_order_no` | (order_no) | UNIQUE | 订单号唯一 |
| `idx_user_currency_status` | (user_id, currency_code, status) | INDEX | C端记录列表 |
| `idx_status_create` | (status, create_at DESC) | INDEX | B端按状态筛选（如"审核中"列表） |
| `idx_create_at` | (create_at DESC) | INDEX | B端时间排序 |

---

## 九、必须级：兑换订单表

### t_exchange_order — 法币→BSB兑换的记录

**为什么必须**：
```
需求文档4.6定义了兑换功能（仅法币→BSB单向）。
需求文档4.8记录模块有"兑换"Tab页面展示兑换历史。

兑换是同步原子操作（一个事务内完成），但仍需订单记录因为：
- C端"兑换记录"Tab需要数据源
- 需要记录兑换时使用的汇率快照（事后可审计"按什么价换的"）
- 流水表的 related_order_no 需要关联到兑换订单
```

**需求追溯**：需求文档4.6兑换模块 + 4.8记录

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `order_no` | varchar(32) | 是 | 订单号(E+16位) | 全局唯一标识 |
| `user_id` | bigint | 是 | 用户ID | 核心维度 |
| `source_currency` | varchar(16) | 是 | 源币种(法币) | 需求文档4.6"仅法币→BSB" |
| `target_currency` | varchar(16) | 是 | 目标币种(BSB) | 固定BSB，但字段保留扩展性 |
| `source_amount` | decimal(30,8) unsigned | 是 | 法币扣减金额 | 用户输入的金额 |
| `exchange_rate` | decimal(30,8) unsigned | 是 | 使用汇率快照 | 审计："按什么价换的" |
| `exchange_amount` | decimal(30,8) unsigned | 是 | 兑换获得BSB | = source_amount / exchange_rate |
| `bonus_ratio` | decimal(10,4) unsigned | 否 | 赠送比例(%) | 需求文档4.6"赠送比例%" |
| `bonus_amount` | decimal(30,8) unsigned | 否 | 赠送BSB | = exchange_amount × bonus_ratio% |
| `actual_amount` | decimal(30,8) unsigned | 是 | 实际到账BSB | = exchange + bonus |
| `bonus_turnover_multi` | decimal(10,2) unsigned | 否 | 赠送流水倍数 | 需求文档4.6"赠送部分需完成X倍流水" |
| `status` | tinyint | 是 | 1成功/2失败 | 兑换是同步操作，创建即终态 |
| `create_at` | bigint | 是 | 兑换时间 | 工程规范 |
| `update_at` | bigint | 是 | 更新时间 | 工程规范 |
| `delete_at` | bigint | 是 | 软删除 | 工程规范 |

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `uk_order_no` | (order_no) | UNIQUE | 订单号唯一 |
| `idx_user_source_time` | (user_id, source_currency, create_at DESC) | INDEX | C端兑换记录列表 |

---

## 十、必须级：冻结记录表

### t_freeze_record — 提现冻结金额的生命周期追踪

**为什么必须**：
```
需求文档4.7.6："用户发起→钱包冻结→审核→出款→成功/失败"

冻结记录独立于提现订单表的原因：
1. 生命周期不同——提现订单有8种状态流转，冻结记录只有3种（冻结中/已解冻/已扣款）
2. 操作对象不同——WalletUnfreeze/WalletDeduct操作的是冻结记录，不是订单
3. 并发安全——冻结解冻的状态守卫需要独立的 WHERE status=冻结中 原子操作

如果不独立建表而是用提现订单表的字段兼容：
  WalletUnfreeze 需要同时校验提现订单状态+冻结状态 → 逻辑耦合
  如果未来有其他冻结场景（如风控冻结），订单表无法承载
```

**需求追溯**：需求文档4.7.6 + 架构设计6.3冻结状态机

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `user_id` | bigint | 是 | 用户ID | 查用户所有冻结 |
| `currency_code` | varchar(16) | 是 | 币种 | 跟随账户 |
| `wallet_type` | tinyint | 是 | 子钱包类型 | 跟随账户 |
| `freeze_amount` | decimal(30,8) unsigned | 是 | 冻结金额 | 核心：冻了多少钱 |
| `related_order_no` | varchar(32) | 是 | 关联提现订单号 | 与t_withdraw_order对应 |
| `status` | tinyint | 是 | 1冻结中/2已解冻/3已扣款 | 冻结状态机 |
| `unfreeze_at` | bigint | 否 | 解冻/扣款时间 | 终态时间戳 |
| `create_at` | bigint | 是 | 冻结时间 | 工程规范 |
| `update_at` | bigint | 是 | 更新时间 | 工程规范 |
| `delete_at` | bigint | 是 | 软删除 | 工程规范 |

**状态机约束**：
```
冻结中(1) → 已解冻(2)  -- WalletUnfreeze
冻结中(1) → 已扣款(3)  -- WalletDeduct
终态(2/3) → 不可再操作

代码实现：UPDATE ... WHERE status=1 AND id=?
         affected_rows=0 → 拒绝操作（已处理过）
```

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `idx_order_no` | (related_order_no) | INDEX | 按提现订单号查冻结记录 |
| `idx_user_status` | (user_id, status) | INDEX | 查用户所有"冻结中"的记录 |

---

## 十一、必须级：提现账户表

### t_withdraw_account — 用户绑定的提现收款账户

**为什么必须**：
```
需求文档4.7.3三种提现方式各需要不同信息：
  银行卡：银行名称、账号、持有人、网点
  电子钱包：钱包类型、手机号、持有人
  USDT：链上地址

需求文档4.7.4明确要求：
  "首次填写后平台记录，后续复用"
  "不可自行修改"（风控要求）
  "账户持有人必须=KYC认证姓名"

如果不独立建表而是放在提现订单里：
  每次提现都要重复存一整套银行信息 → 数据冗余
  用户查"我绑定了什么提现账户"没有直接数据源
  "不可修改"的约束无处依附
```

**需求追溯**：需求文档4.7.3 + 4.7.4

**字段设计**：

| 字段 | 类型 | 必填 | 说明 | 为什么需要这个字段 |
|------|------|------|------|-----------------|
| `id` | bigint | PK | 主键 | 工程规范 |
| `user_id` | bigint | 是 | 用户ID | 核心维度 |
| `account_type` | tinyint | 是 | 1银行卡/2电子钱包/3USDT | 三种方式字段不同 |
| `currency_code` | varchar(16) | 是 | 关联币种 | 银行卡/电子钱包与特定国家币种绑定 |
| `bank_name` | varchar(128) | 否 | 银行名称 | 银行卡时必填：需求文档4.7.3"银行名称(下拉选择)" |
| `bank_account` | varchar(64) | 否 | 银行账号 | 银行卡时必填 |
| `bank_branch` | varchar(128) | 否 | 开户网点 | 银行卡时选填 |
| `account_holder` | varchar(128) | 是 | 账户持有人 | 需求文档4.7.4"自动填充KYC姓名"，必须=KYC实名 |
| `e_wallet_type` | varchar(32) | 否 | 电子钱包类型 | 电子钱包时：GoPay/OVO/DANA/ShopeePay/PromptPay |
| `e_wallet_phone` | varchar(32) | 否 | 电子钱包手机号 | 电子钱包时必填 |
| `chain_address` | varchar(128) | 否 | USDT链上地址 | USDT时必填 |
| `chain_network` | tinyint | 否 | USDT网络类型 | USDT时：1TRC20/2ERC20/3BEP20 |
| `create_at` | bigint | 是 | 绑定时间 | 工程规范 |
| `update_at` | bigint | 是 | 更新时间 | 正常不更新 |
| `delete_at` | bigint | 是 | 软删除 | 工程规范 |

**核心约束**：
- 只写一次不可修改（风控要求）
- `account_holder` 必须与 KYC 实名一致（忽略大小写+空格比对）

**核心索引**：

| 索引 | 字段 | 类型 | 必要性论证 |
|------|------|------|----------|
| `uk_user_type_currency` | (user_id, account_type, currency_code) | UNIQUE | 每种方式+每种币种只能绑定一个账户 |
| `idx_user_id` | (user_id) | INDEX | 查用户所有提现账户 |

---

## 十二、应该级：财务模块相关表分析

> 以下表由财务模块负责人建在 db_finance 库中。我不建这些表，但需要了解它们，因为钱包模块与财务模块有大量数据交互。

### 12.1 t_collect_channel — 代收通道表

**为什么应该有**：
```
需求文档5.2：平台对接多家三方支付通道，代收通道负责收钱。
每个通道有20+字段（名称、币种、费率、限额、权重、预警、技术配置等）。
没有这张表，充值无法选择支付通道。

不在db_wallet里，因为通道配置是财务的职责域。
钱包模块通过RPC读取通道信息或由finance选好通道后告知wallet。
```

**关键字段方向**：通道名称、币种、支付类型、费率、单笔限额、权重、预警金额、状态、请求地址、商户ID、密钥、AppKey、公钥、私钥等

### 12.2 t_payout_channel — 代付通道表

**为什么应该有**：
```
需求文档5.2：代付通道负责付钱（提现出款）。
与代收通道类似但增加了成功率字段。
没有这张表，提现无法发起出款。
```

### 12.3 t_deposit_method_config — 充值方式配置表

**为什么应该有**：
```
需求文档5.4："按币种维度配置可用的充值方式，每种可配单笔限额、显示排序、状态"
这个配置直接影响钱包C端展示的可用充值方式。

例如：VND币种下可用"银行转账"（限额100-50000VND）和"电子钱包QRIS"（限额50-10000VND）
wallet的 GetDepositConfig 接口需要读取这些配置来告诉C端"当前有哪些充值方式"。
```

**关键字段方向**：币种、充值方式类型、单笔最低/最高限额、显示排序、状态

### 12.4 t_withdraw_method_config — 提现方式配置表

**为什么应该有**：
```
需求文档5.4："每种提现方式可配单笔限额、手续费、到账时间、状态"
wallet的 GetWithdrawConfig 需要读取这些配置来校验限额和计算手续费。
```

**关键字段方向**：币种、提现方式类型、单笔最低/最高限额、单日限额、手续费率、T+N到账时间、状态

### 12.5 t_correction_order — 人工修正订单表

**为什么应该有**：
```
需求文档5.7定义了三种人工修正（补单B/加款A/减款M），都有审核流程。
finance负责创建和审核修正单，审核通过后RPC调wallet执行资金操作。

这张表存在finance库，wallet不直接读写，但wallet收到的RPC请求中
related_order_no 会关联到这张表的订单号。
```

**关键字段方向**：订单号(B/A/M+16位)、修正类型、用户ID、币种、各子钱包金额、原因类型、审核人、审核状态、备注

---

## 十三、可选级：扩展表分析

### 13.1 t_audit_config — 稽核流水规则配置表

**为什么可选**：
```
需求文档2.4提到了"稽核流水"概念：
  "充值进中心钱包的钱需要完成稽核流水（如投注达到一定金额）才能提现"
  "奖励钱包的钱需要完成流水要求后才能转入中心钱包"

但当前需求处于预评审状态，稽核的具体规则（倍数、计算方式、哪些行为计入流水）
还没有完全定义。可以在评审确认后再建表。

如果确认要做，大概需要：
  规则ID、关联场景（充值/奖金/兑换赠送）、流水倍数、生效条件、状态
```

### 13.2 t_deposit_address — USDT充值地址池表

**为什么可选**：
```
需求文档4.3.1描述USDT充值需要"生成收款地址和二维码"。
具体实现有两种方案：
  方案A：地址池——预先生成一批地址存表，用户充值时分配
  方案B：动态生成——实时调用区块链服务生成地址

哪种方案取决于对接的区块链服务的能力，当前方案未定。
如果走方案A，需要：
  地址、网络类型、是否已分配、分配给的用户ID、分配时间
```

### 13.3 t_daily_statistics — 每日统计聚合快照表

**为什么可选**：
```
需求文档5.8描述了充值统计/提现统计/通道统计三种报表。

两种实现方案：
  方案A：实时聚合查询——直接 GROUP BY 订单表计算，不存快照
  方案B：定时快照——每日凌晨跑定时任务将统计结果写入快照表

如果订单量不大（早期），方案A足够。
如果订单量增长到查询变慢，再加方案B。
所以当前可选。
```

### 13.4 t_bonus_activity — 额外奖金活动配置表

**为什么可选**：
```
需求文档4.5描述了充值额外奖金功能：
  "默认无奖金，点击弹出半屏浮窗选择"
  "有最低充值门槛，互斥规则，流水倍数"

但奖金活动数据的来源有两种可能：
  方案A：由钱包模块自行管理 → 需要建 t_bonus_activity 表
  方案B：由活动模块（ser-activity或ser-bcom）管理 → wallet通过RPC读取

当前无法确定，评审后再定。
充值订单表已预留 bonus_activity_id 字段做关联。
```

---

## 十四、公共字段规范

> 依据：工程认知 + 对 ser-app/ser-buser/ser-item 等模块的代码分析

### 14.1 通用公共字段

所有表都包含以下公共字段（t_wallet_flow 例外，见说明）：

| 字段 | 类型 | 默认值 | GORM标签 | 说明 |
|------|------|--------|---------|------|
| `id` | bigint unsigned | AUTO_INCREMENT | `primaryKey;autoIncrement:true` | 自增主键 |
| `create_at` | bigint unsigned | 0 | `not null` | 创建时间（Unix秒级时间戳） |
| `update_at` | bigint unsigned | 0 | `not null` | 更新时间（Unix秒级时间戳） |
| `delete_at` | bigint unsigned | 0 | `not null` | 软删除标记（0=未删除，非0=删除时间戳） |

### 14.2 例外说明

**t_wallet_flow 不包含 update_at 和 delete_at**：
```
原因：流水是不可变审计记录，一旦写入永不修改、永不删除。
这是资金安全的基本要求。
如果允许修改/删除流水 → 对账结果不可信 → 资金审计体系崩塌。
```

**t_rate_log 不包含 update_at**：
```
原因：汇率日志也是不可变记录，只插入不更新。
保留 delete_at 是为了兼容工程的通用查询模式（.Where(xxx.DeleteAt.Eq(0))）。
```

### 14.3 金额字段规范

| 规范项 | 规则 | 依据 |
|--------|------|------|
| 存储类型 | `DECIMAL(30,8)` | 架构决策ADR-06：30位整数+8位小数，覆盖所有精度需求 |
| UNSIGNED | 除 t_wallet_flow.amount 外全部 UNSIGNED | 余额/金额不允许负数，DB层兜底 |
| t_wallet_flow.amount | SIGNED | 正数=增加，负数=减少 |
| IDL传输 | string | 避免Thrift double精度丢失 |
| 服务内运算 | shopspring/decimal | Go层精确计算 |
| 默认值 | `0.00000000` | 不用NULL，简化查询 |

### 14.4 表名与字段名规范

| 规范项 | 规则 | 示例 |
|--------|------|------|
| 表名 | 小写+下划线，t_ 前缀 | t_currency, t_wallet_account |
| 字段名 | 小写+下划线 | currency_code, available_balance |
| Go struct | PascalCase（GORM自动映射） | CurrencyCode, AvailableBalance |
| 索引名 | uk_ / idx_ 前缀 + 字段缩写 | uk_currency_code, idx_user_currency |
| 编码 | utf8mb4 + utf8mb4_bin | 全局统一 |

---

## 十五、枚举值速查表

### 15.1 全部枚举一览

| 枚举名 | Go常量前缀 | 值 → 含义 | 使用表 |
|--------|-----------|----------|--------|
| CurrencyType | `CurrencyType` | 1法币 2加密 3平台 | t_currency.currency_type |
| WalletType | `WalletType` | 1中心 2奖励 3主播 4代理 5场馆 | t_wallet_account/flow/freeze |
| DepositStatus | `DepositStatus` | 1待支付 2处理中 3已完成 4已失败 | t_deposit_order.status |
| WithdrawStatus | `WithdrawStatus` | 1审核中 2风控通过 3风控驳回 4财务通过 5财务驳回 6出款中 7出款成功 8出款失败 | t_withdraw_order.status |
| ExchangeStatus | `ExchangeStatus` | 1成功 2失败 | t_exchange_order.status |
| FreezeStatus | `FreezeStatus` | 1冻结中 2已解冻 3已扣款 | t_freeze_record.status |
| FlowType | `FlowType` | 1充值入账 2奖金入账 3兑换扣减 4兑换入账 5兑换赠送 6提现冻结 7提现解冻 8提现扣款 9人工加款 10人工减款 11消费扣款 12返奖入账 13转入 14转出 | t_wallet_flow.flow_type |
| DepositMethod | `DepositMethod` | 1USDT 2银行转账 3电子钱包 | t_deposit_order.deposit_method |
| WithdrawMethod | `WithdrawMethod` | 1银行卡 2电子钱包 3USDT | t_withdraw_order.withdraw_method |
| NetworkType | `NetworkType` | 1TRC20 2ERC20 3BEP20 | t_deposit_order/t_withdraw_account |
| AccountType | `AccountType` | 1银行卡 2电子钱包 3USDT | t_withdraw_account.account_type |
| BizType | `BizType` | 1充值 2提现 3兑换 4修正 | t_wallet_flow.related_biz_type |
| CommonStatus | `CommonStatus` | 1启用 2禁用 | t_currency.status |
| TriggerType | `TriggerType` | 1自动 2手动 | t_rate_log.trigger_type |

### 15.2 状态机转换规则速查

```
充值订单：1(待支付) → 2(处理中) → 3(完成)/4(失败)
                1 → 4（超时未支付直接失败）

提现订单：1(审核中) → 2(风控通过)/3(风控驳回*)
                2 → 4(财务通过)/5(财务驳回*)
                4 → 6(出款中)
                6 → 7(出款成功*)/8(出款失败*)
                *号=终态，不可变

冻结记录：1(冻结中) → 2(已解冻*)/3(已扣款*)
                *号=终态，互斥

兑换订单：创建即终态，1(成功)/2(失败)
```

---

## 十六、表间关系总图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        db_wallet 表间关系                                 │
│                                                                          │
│                                                                          │
│  ┌────────────┐                                                          │
│  │ t_currency │──── currency_code ──────────────────────────┐            │
│  │ (必须)     │    被以下所有表引用                             │            │
│  └─────┬──────┘                                              │            │
│        │ 1:N                                                  │            │
│        ▼                                                      │            │
│  ┌────────────┐                                              │            │
│  │ t_rate_log │                                              │            │
│  │ (必须)     │                                              │            │
│  └────────────┘                                              │            │
│                                                               │            │
│                                                               ▼            │
│  ┌──────────────────┐  1:N   ┌──────────────────┐                        │
│  │ t_wallet_account │───────>│  t_wallet_flow   │                        │
│  │ (必须·核心)       │        │  (必须·审计)      │                        │
│  │ uk:(user,curr,    │        │  uk:(order_no,    │                        │
│  │     wallet_type)  │        │      flow_type)   │                        │
│  │ version乐观锁     │        │  只插入不修改      │                        │
│  └──────────────────┘        └────────┬─────────┘                        │
│                                        │ related_order_no                 │
│                               ┌────────┼─────────────┐                   │
│                               │        │             │                   │
│                               ▼        ▼             ▼                   │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                 │
│  │t_deposit_    │   │t_withdraw_   │   │t_exchange_   │                 │
│  │  order       │   │  order       │   │  order       │                 │
│  │ (必须)       │   │ (必须)       │   │ (必须)       │                 │
│  │ C+16位       │   │ T+16位       │   │ E+16位       │                 │
│  └──────────────┘   └──────┬───────┘   └──────────────┘                 │
│                             │                                             │
│              ┌──────────────┼──────────────┐                             │
│              │ 1:1          │              │ N:1                          │
│              ▼              │              ▼                              │
│  ┌──────────────┐          │    ┌──────────────────┐                    │
│  │t_freeze_     │          │    │t_withdraw_       │                    │
│  │  record      │          │    │  account         │                    │
│  │ (必须)       │          │    │ (必须)            │                    │
│  │ 状态机       │          │    │ 只写一次不可改     │                    │
│  └──────────────┘          │    └──────────────────┘                    │
│                             │                                             │
│  - - - - - - - - - - - - - ┼ - - - - - - - - - - - - - - - - - -       │
│  db_finance (财务负责人)      │                                             │
│                             │ RPC调用                                     │
│  ┌─────────────────┐  ┌────┴─────────┐  ┌─────────────────────┐        │
│  │t_collect_channel│  │t_correction_ │  │t_deposit/withdraw_  │        │
│  │t_payout_channel │  │  order       │  │  method_config      │        │
│  │ (应该)          │  │ (应该)       │  │ (应该)               │        │
│  └─────────────────┘  └──────────────┘  └─────────────────────┘        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 十七、建表顺序与依赖

### 17.1 建表顺序（遵循依赖方向）

```
Phase 1  基础层（无依赖，最先建）
├── t_currency          ← 所有表都依赖它的 currency_code
└── t_rate_log          ← 仅依赖 t_currency 的 currency_code

Phase 2  核心层（依赖 Phase 1）
├── t_wallet_account    ← 引用 currency_code
└── t_wallet_flow       ← 引用 currency_code + 关联订单号

Phase 3  业务层（依赖 Phase 1 + 2）
├── t_deposit_order     ← 入账操作写 t_wallet_account + t_wallet_flow
├── t_withdraw_account  ← 独立表，无强依赖，但逻辑上与提现相关
├── t_withdraw_order    ← 引用 t_withdraw_account.id + 创建 t_freeze_record
├── t_freeze_record     ← 与 t_withdraw_order 同时创建
└── t_exchange_order    ← 操作写 t_wallet_account + t_wallet_flow
```

### 17.2 建表后的下一步

```
1. 执行建表SQL → db_wallet 数据库
2. 确认 ETCD 配置 /slg/serv/wallet/db → db_wallet
3. 编写 gorm_gen.go → 引用所有9张表
4. 执行 go run gorm_gen.go → 生成 model/ 和 query/
5. 在 internal/enum/ 下定义枚举常量
6. 开始 Repository 层 → Service 层编码

建表SQL参考：/Users/mac/gitlab/z-readme/数据模型/tmp-1.md 第十四章
```

---

## 总结

### 按级别统计

| 级别 | 数量 | 所属 | 表名 |
|------|------|------|------|
| **必须** | **9张** | db_wallet（我建） | t_currency, t_rate_log, t_wallet_account, t_wallet_flow, t_deposit_order, t_withdraw_order, t_exchange_order, t_freeze_record, t_withdraw_account |
| **应该** | **5张** | db_finance（他建） | t_collect_channel, t_payout_channel, t_deposit_method_config, t_withdraw_method_config, t_correction_order |
| **可选** | **4张** | 待定 | t_audit_config, t_deposit_address, t_daily_statistics, t_bonus_activity |

### 设计依据链

```
每张表的存在都可以追溯到需求文档的具体章节：

需求文档3.2三类币种           → t_currency
需求文档3.4+3.7汇率更新+日志  → t_rate_log
需求文档2.1五种子钱包         → t_wallet_account
需求文档4.9+7.7对账一致性     → t_wallet_flow
需求文档4.3充值三种方式       → t_deposit_order
需求文档4.7提现+5.6双审核     → t_withdraw_order
需求文档4.6兑换              → t_exchange_order
需求文档4.7.6冻结流程         → t_freeze_record
需求文档4.7.3+4.7.4账户管理   → t_withdraw_account

每个字段的存在都可以追溯到需求文档的具体功能点。
没有一张"拍脑袋"加的表，没有一个"可能用到"的冗余字段。
```
