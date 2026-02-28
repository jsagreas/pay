# ser-wallet 数据模型设计

> 设计时间：2026-02-26
> 设计定位：将产品原型和需求文档中的现实世界描述，映射为可直接指导建表和编码的数据模型
> 设计依据：
> - 需求理解：/Users/mac/gitlab/z-readme/需求理解/tmp-1.md
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-1.md
> - 架构设计：/Users/mac/gitlab/z-readme/架构设计/tmp-1.md（ADR决策）
> - 实现思路：/Users/mac/gitlab/z-readme/实现思路/tmp-1.md
> - 工程认知：/Users/mac/gitlab/z-readme/工程认知/tmp-1.md（GORM Gen + 软删除 + 命名规范）
> 工程约束：TiDB(MySQL兼容) + GORM Gen 自动生成 model/query + DECIMAL(30,8) + 软删除(delete_at=0)
> 数据库名：db_wallet（ETCD配置 /slg/serv/wallet/db）

---

## 目录

1. [从现实世界到数据模型的映射总览](#一从现实世界到数据模型的映射总览)
2. [领域模型全景](#二领域模型全景)
3. [枚举与常量定义](#三枚举与常量定义)
4. [币种配置域](#四币种配置域)
5. [钱包账户域](#五钱包账户域)
6. [资金流水域](#六资金流水域)
7. [充值订单域](#七充值订单域)
8. [提现订单域](#八提现订单域)
9. [兑换订单域](#九兑换订单域)
10. [冻结记录域](#十冻结记录域)
11. [提现账户域](#十一提现账户域)
12. [模型间关系图](#十二模型间关系图)
13. [索引设计总表](#十三索引设计总表)
14. [建表语句参考](#十四建表语句参考)
15. [后续怎么做](#十五后续怎么做)

---

## 一、从现实世界到数据模型的映射总览

### 1.1 映射思路

产品原型和需求文档描述的是"现实世界"——用户充值、提现、兑换、运营配置币种。数据模型要做的是把这些现实世界的概念抽象为"表+字段+关系+约束"，让代码层面能正确承载业务逻辑。

```
现实世界概念                 数据模型                    说明
────────────────────────────────────────────────────────────────
平台支持哪些货币             → t_currency               币种配置表
汇率什么时候变了             → t_rate_log               汇率变更日志
用户在某币种下有多少钱        → t_wallet_account         子钱包账户表
每一笔资金变动的记录          → t_wallet_flow            资金流水表（审计+对账）
用户充了一笔钱               → t_deposit_order          充值订单表
用户提了一笔钱               → t_withdraw_order         提现订单表
用户把法币换成BSB            → t_exchange_order         兑换订单表
提现时冻结的那笔钱            → t_freeze_record          冻结记录表
用户绑定的银行卡/电子钱包     → t_withdraw_account       提现账户表
```

### 1.2 模型分域

按业务边界将全部数据模型划分为 **5 个域**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        db_wallet 数据库                               │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────────────────────────────┐ │
│  │  币种配置域        │  │  钱包核心域                                │ │
│  │  ┌──────────────┐│  │  ┌────────────────┐ ┌────────────────┐ │ │
│  │  │ t_currency   ││  │  │t_wallet_account│ │ t_wallet_flow  │ │ │
│  │  ├──────────────┤│  │  └────────────────┘ └────────────────┘ │ │
│  │  │ t_rate_log   ││  │  ┌────────────────┐                    │ │
│  │  └──────────────┘│  │  │t_freeze_record │                    │ │
│  └──────────────────┘  │  └────────────────┘                    │ │
│                         └──────────────────────────────────────────┘ │
│  ┌──────────────────┐  ┌──────────────────┐ ┌───────────────────┐  │
│  │  充值域            │  │  提现域           │ │  兑换域            │  │
│  │  ┌──────────────┐│  │  ┌──────────────┐│ │  ┌─────────────┐ │  │
│  │  │t_deposit_order││ │  │t_withdraw_order│ │  │t_exchange_order│ │
│  │  └──────────────┘│  │  ├──────────────┤│ │  └─────────────┘ │  │
│  └──────────────────┘  │  │t_withdraw_   ││ └───────────────────┘  │
│                         │  │ account      ││                        │
│                         │  └──────────────┘│                        │
│                         └──────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 共性约定

以下约定适用于所有表，不再逐表重复：

| 约定项 | 规则 | 依据 |
|--------|------|------|
| 主键 | `id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY` | 工程规范：GORM Gen 默认 |
| 软删除 | `delete_at BIGINT UNSIGNED NOT NULL DEFAULT 0` | 工程规范：`.Where(xxx.DeleteAt.Eq(0))` |
| 创建时间 | `create_at BIGINT UNSIGNED NOT NULL DEFAULT 0` | 工程规范：Unix 时间戳（秒） |
| 更新时间 | `update_at BIGINT UNSIGNED NOT NULL DEFAULT 0` | 工程规范：Unix 时间戳（秒） |
| 金额类型 | `DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000` | 架构决策 ADR-06 |
| 字符编码 | `utf8mb4` | TiDB 默认 |
| 引擎 | InnoDB | TiDB 兼容 |

---

## 二、领域模型全景

### 2.1 全部数据模型一览

| 序号 | 表名 | 域 | 现实含义 | 记录规模估算 | 读写特性 |
|------|------|-----|---------|-------------|---------|
| 1 | `t_currency` | 币种配置 | 平台支持的所有货币 | 极小（<20条） | 读远多于写 |
| 2 | `t_rate_log` | 币种配置 | 每次汇率更新的历史记录 | 中等（日增数百） | 只写+翻页查 |
| 3 | `t_wallet_account` | 钱包核心 | 用户×币种×子钱包类型=一个账户 | 大（用户数×币种数×3） | 高频读写 |
| 4 | `t_wallet_flow` | 钱包核心 | 每笔余额变动的审计记录 | 最大（只增不删） | 只写+条件查 |
| 5 | `t_freeze_record` | 钱包核心 | 提现冻结的生命周期追踪 | 中等（提现笔数） | 低频读写 |
| 6 | `t_deposit_order` | 充值 | 充值订单 | 大（充值笔数） | 写后多读 |
| 7 | `t_withdraw_order` | 提现 | 提现订单 | 中大（提现笔数） | 写后多读 |
| 8 | `t_exchange_order` | 兑换 | 兑换订单 | 中等（兑换笔数） | 写后少读 |
| 9 | `t_withdraw_account` | 提现 | 用户绑定的提现账户 | 大（用户数×账户类型数） | 写一次读多次 |

---

## 三、枚举与常量定义

> 对应 Go 代码位置：`ser-wallet/internal/enum/`
> 枚举是数据模型的"血肉"——表结构是骨骼，枚举定义了字段的合法取值范围

### 3.1 币种类型（CurrencyType）

```
来源：需求文档3.2三类币种

值  含义        举例
1   法定货币     VND, IDR, THB
2   加密货币     USDT, BTC, ETH
3   平台货币     BSB
```

### 3.2 子钱包类型（WalletType）

```
来源：需求文档2.1五种子钱包

值  含义        资金来源                使用规则              当前状态
1   中心钱包     充值                   投注/消费/提现(稽核)   核心必实现
2   奖励钱包     活动/任务返奖/投注返奖   流水达标→转中心钱包    核心必实现
3   主播钱包     礼物打赏/短视频/活动     可直接提现/转中心       核心必实现
4   代理钱包     代理佣金(下级分佣)       可直接提现/转中心       预留暂不实现
5   场馆钱包     中心+奖励转入           igaming场馆临时使用    预留暂不实现
```

### 3.3 充值订单状态（DepositStatus）

```
来源：需求文档充值流程 + 架构设计6.1充值状态机

值  含义        说明
1   待支付       创建订单后初始状态
2   处理中       收到支付通知/链上确认中
3   已完成       入账成功（终态）
4   已失败       支付超时/异常/卡片问题（终态）

合法转换：
  1(待支付) → 2(处理中) → 3(已完成)
  1(待支付) → 2(处理中) → 4(已失败)
  1(待支付) → 4(已失败)  -- 超时未支付
```

### 3.4 提现订单状态（WithdrawStatus）

```
来源：需求文档7.6提现流程 + 架构设计6.2提现状态机

值  含义         说明
1   审核中        创建订单+冻结后初始状态
2   风控通过      风控审核通过
3   风控驳回      风控审核驳回 → 触发解冻（终态）
4   财务通过      财务审核通过
5   财务驳回      财务审核驳回 → 触发解冻（终态）
6   出款中        已发起代付
7   出款成功      代付成功 → 触发最终扣款（终态）
8   出款失败      代付失败 → 触发解冻（终态）

合法转换：
  1 → 2(风控通过) 或 3(风控驳回)
  2 → 4(财务通过) 或 5(财务驳回)
  4 → 6(出款中)
  6 → 7(出款成功) 或 8(出款失败)
  终态(3/5/7/8) → 不可变
```

### 3.5 兑换订单状态（ExchangeStatus）

```
来源：需求文档6.1兑换流程（内部闭环，无审核）

值  含义        说明
1   成功         兑换立即完成（终态，兑换是同步原子操作）
2   失败         兑换失败（余额不足/系统异常）（终态）

说明：兑换是同步原子操作（事务内完成），不存在中间状态。
     创建即终态，要么成功要么失败。
```

### 3.6 冻结记录状态（FreezeStatus）

```
来源：架构设计6.3冻结状态机

值  含义        说明
1   冻结中       WalletFreeze创建
2   已解冻       WalletUnfreeze（审核驳回/出款失败）
3   已扣款       WalletDeduct（出款成功）

两个终态互斥：同一条记录只能走到2或3其中一个。
```

### 3.7 流水类型（FlowType）

```
来源：架构设计3.4六种原子操作 + 实现思路7.3

值   含义            对应原子操作    触发场景
1    充值入账         credit        充值成功/补单通过 → 中心钱包
2    奖金入账         credit        额外奖金/活动奖励 → 奖励钱包
3    兑换扣减         debit         法币→BSB兑换时扣减法币
4    兑换入账         credit        法币→BSB兑换时增加BSB
5    兑换赠送入账     credit        兑换赠送部分入账 → 奖励钱包
6    提现冻结         freeze        提现发起冻结
7    提现解冻         unfreeze      审核驳回/出款失败解冻归还
8    提现扣款         deduct        出款成功最终扣除
9    人工加款         credit        finance审核通过后加款
10   人工减款         debitSafe     finance审核通过后减款
11   消费扣款         debit         投注/打赏扣款（预留）
12   返奖入账         credit        投注返奖（预留）
13   转入入账         credit        子钱包间转入（预留）
14   转出扣减         debit         子钱包间转出（预留）
```

### 3.8 充值方式（DepositMethod）

```
来源：需求文档4.3充值模块三种方式

值  含义            技术实现
1   USDT链上充值     平台承载页，自生成收款地址+二维码
2   银行转账         三方托管页，跳转三方支付
3   电子钱包         三方托管页，跳转三方扫码
```

### 3.9 提现方式（WithdrawMethod）

```
来源：需求文档4.7.3三种提现方式

值  含义            前置条件
1   银行卡提现       需KYC认证
2   电子钱包提现     需KYC认证
3   USDT提现        无需KYC
```

### 3.10 USDT网络类型（NetworkType）

```
来源：需求文档4.3.1 USDT充值

值  含义
1   TRC20（默认）
2   ERC20
3   BEP20
```

### 3.11 人工修正类型（CorrectionType）

```
来源：需求文档5.7人工修正

值  含义            订单号前缀
1   补单             B
2   人工加款          A
3   人工减款          M
```

### 3.12 加减款原因类型（CorrectionReason）

```
来源：需求文档5.7人工加款/减款场景

值  含义
1   系统补偿
2   活动补发
3   财务冲正
4   费用调整
5   异常追回
6   风控处罚
7   其他
```

### 3.13 订单号前缀规范

```
来源：需求文档5.9订单号规范

前缀  类型        生成方         格式
C     充值订单     ser-wallet    C + 16位
T     提现订单     ser-wallet    T + 16位
E     兑换订单     ser-wallet    E + 16位（需求未规定前缀，自定义）
B     补单        ser-finance   B + 16位
A     人工加款     ser-finance   A + 16位
M     人工减款     ser-finance   M + 16位
```

---

## 四、币种配置域

### 4.1 t_currency — 币种配置表

**现实含义**：平台支持哪些货币，每种货币怎么计价、怎么展示、汇率怎么算

**需求来源**：
- 需求文档3.2三类币种 + 3.4汇率体系 + 3.5精度规则 + 3.6金额显示规则
- 后台配置页面20个字段

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键 | 工程规范 |
| `currency_code` | VARCHAR(16) | NOT NULL | - | 币种代码（如VND/USDT/BSB） | 需求文档3.2 "币种代码唯一，不可重复" |
| `currency_name` | VARCHAR(64) | NOT NULL | '' | 币种名称（如"越南盾"） | 需求文档3.2 |
| `currency_type` | TINYINT UNSIGNED | NOT NULL | 0 | 币种类型（1法币/2加密/3平台） | 需求文档3.2三类币种 |
| `country` | VARCHAR(32) | NOT NULL | '' | 所属国家（法币关联国家，加密和平台为空） | 需求文档3.2 |
| `icon_url` | VARCHAR(512) | NOT NULL | '' | 币种图标URL（SVG/PNG，≤5KB） | 需求文档3.1 "编辑弹窗支持上传图标" |
| `currency_symbol` | VARCHAR(16) | NOT NULL | '' | 货币符号（如 ₫, Rp, ฿, $） | 需求文档3.6金额显示 |
| `thousands_sep` | VARCHAR(4) | NOT NULL | ',' | 千分位分隔符（VND/IDR用`.` THB/BSB用`,`） | 需求文档3.6 |
| `decimal_sep` | VARCHAR(4) | NOT NULL | '.' | 小数点符号（VND/IDR无小数） | 需求文档3.6 |
| `precision` | TINYINT UNSIGNED | NOT NULL | 2 | 小数位数（法币2位/加密8位） | 需求文档3.5精度规则 |
| `real_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 实时汇率（3家API取平均值，定时刷新） | 需求文档3.4 "3家三方API取平均值" |
| `platform_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 平台汇率（偏差≥阈值时快照更新） | 需求文档3.4 "系统内部实际使用的汇率" |
| `rate_fluctuation` | DECIMAL(10,4) UNSIGNED | NOT NULL | 0 | 汇率浮动比例（%，支持2位小数） | 需求文档3.4 "入款汇率=平台汇率×(1+浮动%)" |
| `rate_threshold` | DECIMAL(10,4) UNSIGNED | NOT NULL | 0 | 汇率偏差阈值（%） | 需求文档3.4 "偏差≥阈值%时更新平台汇率" |
| `deposit_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 入款汇率 = platform_rate × (1 + rate_fluctuation/100) | 需求文档3.4 |
| `withdraw_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 出款汇率 = platform_rate × (1 - rate_fluctuation/100) | 需求文档3.4 |
| `is_base` | TINYINT UNSIGNED | NOT NULL | 0 | 是否基准币种（0否/1是，整表仅一条为1） | 需求文档3.3 "只能设置一次，不可修改" |
| `sort_value` | INT UNSIGNED | NOT NULL | 0 | 排序值（C端列表排序） | 需求文档3.1后台配置 |
| `status` | TINYINT UNSIGNED | NOT NULL | 1 | 状态（1启用/2禁用） | 需求文档3.1 "启用/禁用" |
| `create_by` | BIGINT UNSIGNED | NOT NULL | 0 | 创建人ID | 工程规范 |
| `update_by` | BIGINT UNSIGNED | NOT NULL | 0 | 最后修改人ID | 工程规范 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 创建时间 | 工程规范 |
| `update_at` | BIGINT UNSIGNED | NOT NULL | 0 | 更新时间 | 工程规范 |
| `delete_at` | BIGINT UNSIGNED | NOT NULL | 0 | 软删除时间 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `uk_currency_code` | `(currency_code)` | UNIQUE | 币种代码全局唯一 |
| `idx_status_sort` | `(status, sort_value)` | INDEX | C端查可用币种列表按排序 |

**约束规则**：
- `currency_code` 创建后不可修改
- `currency_type` 创建后不可修改
- `is_base=1` 的币种不可禁用
- `is_base` 整表只允许一条记录为 1
- 所有汇率和金额字段使用 DECIMAL，不用 float
- 币种数据初始由开发导入，后台只能编辑/启禁用/上传图标

---

### 4.2 t_rate_log — 汇率变更日志表

**现实含义**：每次平台汇率因偏差达到阈值而自动更新时，留下的一条变更记录

**需求来源**：需求文档3.4汇率更新机制 + 3.7后台配置页 "汇率日志" Tab

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键 | 工程规范 |
| `currency_code` | VARCHAR(16) | NOT NULL | - | 变更的币种代码 | 需求文档3.7 "日志编号、时间、币种代码" |
| `real_rate_before` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 更新前的实时汇率 | 便于审计追溯 |
| `real_rate_after` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 更新后的实时汇率（本次3家API平均值） | 需求文档3.7 "实时汇率" |
| `platform_rate_before` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 更新前的平台汇率 | 便于审计追溯 |
| `platform_rate_after` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 更新后的平台汇率 | 需求文档3.7 "平台汇率" |
| `deposit_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 更新后的入款汇率 | 需求文档3.7 "入款汇率" |
| `withdraw_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 更新后的出款汇率 | 需求文档3.7 "出款汇率" |
| `deviation_pct` | DECIMAL(10,4) | NOT NULL | 0 | 偏差百分比（触发本次更新的偏差值） | 需求文档3.4 "偏差值≥阈值%时更新" |
| `trigger_type` | TINYINT UNSIGNED | NOT NULL | 1 | 触发方式（1自动定时/2手动） | 预留手动触发场景 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 日志时间 | 需求文档3.7 "时间" |
| `delete_at` | BIGINT UNSIGNED | NOT NULL | 0 | 软删除 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `idx_currency_create` | `(currency_code, create_at DESC)` | INDEX | B端按币种查汇率日志 |
| `idx_create_at` | `(create_at DESC)` | INDEX | B端按时间倒序翻页 |

**特点**：
- 只插入，不更新（日志是不可变的审计记录）
- 每次汇率定时任务检测到偏差达阈值时自动生成
- B端"汇率日志"Tab 的数据源

---

## 五、钱包账户域

### 5.1 t_wallet_account — 子钱包账户表

**现实含义**：用户在某个币种下某种子钱包中有多少钱。这是整个钱包系统的核心数据——"钱存在哪里"

**需求来源**：
- 需求文档2.1五种子钱包结构
- 需求文档2.2余额计算 "资金钱包余额 = 中心 + 奖励"
- 需求文档2.5 "每个币种独立一套"
- 架构设计3.2账户模型 "三维度唯一确定：(user_id, currency_code, wallet_type)"

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键 | 工程规范 |
| `user_id` | BIGINT UNSIGNED | NOT NULL | 0 | 用户ID | 核心维度1 |
| `currency_code` | VARCHAR(16) | NOT NULL | - | 币种代码 | 需求文档2.5 "每个币种独立一套" |
| `wallet_type` | TINYINT UNSIGNED | NOT NULL | 0 | 子钱包类型（枚举3.2） | 需求文档2.1五种子钱包 |
| `available_balance` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 可用余额（用户当前能操作的钱） | 架构设计3.2 "用户可操作的钱" |
| `frozen_balance` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 冻结余额（提现审核中的钱，暂不可操作） | 架构设计3.2 "提现审核中的钱" |
| `version` | BIGINT UNSIGNED | NOT NULL | 0 | 乐观锁版本号（每次余额变动+1） | 架构决策ADR-03乐观锁 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 账户创建时间 | 工程规范 |
| `update_at` | BIGINT UNSIGNED | NOT NULL | 0 | 最后变动时间 | 工程规范 |
| `delete_at` | BIGINT UNSIGNED | NOT NULL | 0 | 软删除 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `uk_user_currency_type` | `(user_id, currency_code, wallet_type)` | UNIQUE | 账户唯一性：一个用户+一个币种+一个子钱包类型=唯一账户 |
| `idx_user_currency` | `(user_id, currency_code)` | INDEX | 按币种查该用户所有子钱包余额（钱包概览页） |

**核心设计要点**：

```
1. 账户三维唯一
   (user_id=1001, currency_code='VND', wallet_type=1)  → VND中心钱包
   (user_id=1001, currency_code='VND', wallet_type=2)  → VND奖励钱包
   (user_id=1001, currency_code='VND', wallet_type=3)  → VND主播钱包
   (user_id=1001, currency_code='BSB', wallet_type=1)  → BSB中心钱包
   ...
   不同币种完全隔离，各自一套

2. C端"余额" = 中心.available_balance + 奖励.available_balance
   主播钱包不计入C端展示余额

3. 总余额 = available_balance + frozen_balance
   available_balance 是可操作的
   frozen_balance 是提现审核中暂时锁定的

4. 所有余额变动只能通过六种原子操作（见架构设计3.4）
   不允许直接 UPDATE 余额字段
   每次变动 version+1，乐观锁防并发

5. DECIMAL(30,8) UNSIGNED
   UNSIGNED 是数据库层面禁止余额变负的最后安全网
   即使代码有bug，DB层面也能拦住

6. 账户惰性创建
   用户首次访问某币种时 INSERT ON DUPLICATE KEY IGNORE 初始化
   初始化3条记录(center/reward/streamer)，代理和场馆预留
```

---

## 六、资金流水域

### 6.1 t_wallet_flow — 资金流水表

**现实含义**：每一笔钱的变动都留下的不可篡改记录——谁的钱、在哪个钱包、什么原因、变了多少、变前多少、变后多少

**需求来源**：
- 需求文档4.9 "对账一致性：账务流水与账户余额必须强一致性校验"
- 架构设计4.2流水记录设计
- 架构决策ADR-04幂等性 "流水表唯一索引做幂等"

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键（雪花ID或自增） | 工程规范 |
| `user_id` | BIGINT UNSIGNED | NOT NULL | 0 | 用户ID | 核心维度 |
| `currency_code` | VARCHAR(16) | NOT NULL | - | 币种代码 | 跟随账户 |
| `wallet_type` | TINYINT UNSIGNED | NOT NULL | 0 | 子钱包类型 | 跟随账户 |
| `flow_type` | TINYINT UNSIGNED | NOT NULL | 0 | 流水类型（枚举3.7） | 架构设计4.2 |
| `amount` | DECIMAL(30,8) | NOT NULL | 0 | 变动金额（正数=增加，负数=减少） | 架构设计4.2 |
| `balance_before` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 变动前可用余额快照 | 架构设计4.2 "balance_after=balance_before+amount" |
| `balance_after` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 变动后可用余额快照 | 架构设计4.2 |
| `frozen_before` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 变动前冻结余额快照 | 架构设计4.2 |
| `frozen_after` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 变动后冻结余额快照 | 架构设计4.2 |
| `related_order_no` | VARCHAR(32) | NOT NULL | '' | 关联订单号（C/T/E/B/A/M+16位） | 架构设计4.2 "关联订单号" |
| `related_biz_type` | TINYINT UNSIGNED | NOT NULL | 0 | 关联业务类型（1充值/2提现/3兑换/4修正） | 架构设计4.2 |
| `remark` | VARCHAR(512) | NOT NULL | '' | 备注（人工加减款的原因说明等） | 架构设计4.2 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 流水创建时间 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `uk_order_flow` | `(related_order_no, flow_type)` | UNIQUE | **幂等保障**：同一订单同一类型只能有一条流水 |
| `idx_user_currency_time` | `(user_id, currency_code, wallet_type, create_at DESC)` | INDEX | 用户流水查询（C端记录页） |
| `idx_order_no` | `(related_order_no)` | INDEX | 按订单号追溯所有关联流水 |

**核心设计要点**：

```
1. 只插入，不更新，不删除
   流水是不可变的审计记录
   没有 update_at 和 delete_at 字段（流水表不做软删除）
   注意：amount 字段是 SIGNED（可以为负数）

2. uk_order_flow 唯一索引是幂等的核心
   当 finance 重试调用 WalletCredit(order_no=C123) 时：
   - 第一次：正常插入流水，余额变动 → 成功
   - 第二次：INSERT 触发唯一索引冲突 → 事务回滚 → 返回"已处理" → 幂等保障
   见架构决策 ADR-04

3. balance_before + amount = balance_after 恒等式
   这是对账的基础公式
   定时对账任务验证：最后一条流水的 balance_after == account.available_balance

4. 同一订单可以有多条流水（flow_type 不同）
   例如一笔充值带额外奖金：
     flow(order_no=C123, flow_type=1充值入账) → 中心钱包
     flow(order_no=C123, flow_type=2奖金入账) → 奖励钱包
   唯一索引是 (order_no, flow_type) 而非仅 order_no

5. frozen_before / frozen_after 用于追踪冻结余额变动
   freeze/unfreeze/deduct 操作会改变冻结余额
   普通的 credit/debit 操作中 frozen_before=frozen_after（冻结不变）
```

---

## 七、充值订单域

### 7.1 t_deposit_order — 充值订单表

**现实含义**：用户发起的每一笔充值操作的完整记录

**需求来源**：
- 需求文档4.3充值模块（三种充值方式）
- 需求文档4.4重复转账拦截
- 需求文档4.5额外奖金
- 需求文档4.8记录模块（充值Tab）
- 需求文档5.9订单号 "C + 16位"

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键 | 工程规范 |
| `order_no` | VARCHAR(32) | NOT NULL | - | 充值订单号（C+16位） | 需求文档5.9 |
| `user_id` | BIGINT UNSIGNED | NOT NULL | 0 | 用户ID | 核心维度 |
| `currency_code` | VARCHAR(16) | NOT NULL | - | 充值币种 | 需求文档4.2 "按币种隔离" |
| `deposit_method` | TINYINT UNSIGNED | NOT NULL | 0 | 充值方式（枚举3.8） | 需求文档4.3三种充值方式 |
| `amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 充值金额（用户输入的，以当前币种为单位） | 需求文档4.3 |
| `actual_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 实际到账金额（扣除手续费后） | 预留手续费场景 |
| `usdt_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | USDT换算金额（仅USDT充值时有值） | 需求文档4.3.1 "系统根据汇率换算USDT" |
| `exchange_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 使用的汇率（充值时快照） | 汇率随时变，需要快照记录 |
| `network_type` | TINYINT UNSIGNED | NOT NULL | 0 | USDT网络（枚举3.10，仅USDT时有值） | 需求文档4.3.1 "TRC20/ERC20/BEP20" |
| `tx_hash` | VARCHAR(128) | NOT NULL | '' | 链上交易哈希（仅USDT到账后填入） | USDT充值确认凭证 |
| `deposit_address` | VARCHAR(128) | NOT NULL | '' | USDT收款地址（仅USDT时有值） | 需求文档4.3.1 "展示收款地址和二维码" |
| `channel_id` | BIGINT UNSIGNED | NOT NULL | 0 | 支付通道ID（银行/电子钱包由finance分配） | 需求文档5.3通道轮询 |
| `channel_order_no` | VARCHAR(64) | NOT NULL | '' | 三方通道订单号（三方返回的） | 对账使用 |
| `bonus_activity_id` | BIGINT UNSIGNED | NOT NULL | 0 | 额外奖金活动ID（0=无奖金） | 需求文档4.5 "选择参加额外奖金活动" |
| `bonus_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 奖金金额（入奖励钱包） | 需求文档4.5 "奖金金额→奖励钱包" |
| `bonus_turnover_multi` | DECIMAL(10,2) UNSIGNED | NOT NULL | 0 | 奖金流水倍数要求 | 需求文档4.5 "奖金部分需完成N倍流水" |
| `status` | TINYINT UNSIGNED | NOT NULL | 1 | 订单状态（枚举3.3） | 需求文档4.8记录状态 |
| `fail_reason` | VARCHAR(256) | NOT NULL | '' | 失败原因（支付超时/支付异常/卡片问题） | 需求文档4.8 "失败原因分类" |
| `paid_at` | BIGINT UNSIGNED | NOT NULL | 0 | 支付完成时间 | C端展示 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 下单时间 | 工程规范 |
| `update_at` | BIGINT UNSIGNED | NOT NULL | 0 | 最后更新时间 | 工程规范 |
| `delete_at` | BIGINT UNSIGNED | NOT NULL | 0 | 软删除 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `uk_order_no` | `(order_no)` | UNIQUE | 订单号全局唯一 |
| `idx_user_currency_status` | `(user_id, currency_code, status)` | INDEX | C端按币种查充值记录列表 |
| `idx_user_currency_amount_status` | `(user_id, currency_code, amount, status)` | INDEX | 重复转账拦截查询：同user+同币种+同金额+待支付/处理中 |
| `idx_channel_order` | `(channel_order_no)` | INDEX | 三方回调时按通道订单号查找 |
| `idx_create_at` | `(create_at DESC)` | INDEX | B端按时间倒序查询 |

**重复转账拦截查询模式**（需求文档4.4）：

```sql
-- 创建订单前检查：同一用户+同一币种+同一金额，状态为待支付或处理中
SELECT COUNT(*) FROM t_deposit_order
WHERE user_id = ? AND currency_code = ? AND amount = ?
  AND status IN (1, 2)  -- 待支付, 处理中
  AND delete_at = 0;

-- count=0 → 正常
-- count=1~2 → 软拦截（返回警告，允许继续）
-- count>=3 → 硬拦截（禁止创建新订单）
-- 仅适用于银行转账和电子钱包，USDT跳过此检查
```

---

## 八、提现订单域

### 8.1 t_withdraw_order — 提现订单表

**现实含义**：用户发起的每一笔提现操作的完整记录，包含双审核流程的状态流转

**需求来源**：
- 需求文档4.7提现模块（KYC/限额/三种方式/双审核）
- 需求文档4.8记录模块（提现Tab）
- 需求文档5.6提现审核 "双审核流程"
- 需求文档5.9订单号 "T + 16位"

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键 | 工程规范 |
| `order_no` | VARCHAR(32) | NOT NULL | - | 提现订单号（T+16位） | 需求文档5.9 |
| `user_id` | BIGINT UNSIGNED | NOT NULL | 0 | 用户ID | 核心维度 |
| `currency_code` | VARCHAR(16) | NOT NULL | - | 提现币种 | 需求文档4.7 |
| `wallet_type` | TINYINT UNSIGNED | NOT NULL | 1 | 提现来源子钱包（默认中心钱包） | 需求文档2.1 "中心钱包可提现" |
| `withdraw_method` | TINYINT UNSIGNED | NOT NULL | 0 | 提现方式（枚举3.9） | 需求文档4.7.3 |
| `withdraw_account_id` | BIGINT UNSIGNED | NOT NULL | 0 | 关联的提现账户ID（t_withdraw_account.id） | 需求文档4.7.4 "账户信息管理" |
| `amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 提现金额 | 需求文档4.7 |
| `fee_rate` | DECIMAL(10,4) UNSIGNED | NOT NULL | 0 | 手续费率（%） | 需求文档4.7.5 "手续费率" |
| `fee_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 手续费金额 | 需求文档4.7.5 |
| `actual_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 到账金额 = amount - fee_amount | 需求文档4.7.3 "提现金额、手续费、到账金额" |
| `exchange_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 使用的汇率（BSB提现时需要换算） | 需求文档4.7.3 "BSB提现需展示汇率换算" |
| `freeze_record_id` | BIGINT UNSIGNED | NOT NULL | 0 | 关联的冻结记录ID | 架构设计6.3 |
| `status` | TINYINT UNSIGNED | NOT NULL | 1 | 订单状态（枚举3.4） | 架构设计6.2提现状态机 |
| `risk_reviewer_id` | BIGINT UNSIGNED | NOT NULL | 0 | 风控审核人ID | 需求文档5.6 "风控审核" |
| `risk_review_at` | BIGINT UNSIGNED | NOT NULL | 0 | 风控审核时间 | 需求文档5.6 |
| `risk_review_remark` | VARCHAR(512) | NOT NULL | '' | 风控审核备注 | 需求文档5.6 "驳回需填写备注" |
| `finance_reviewer_id` | BIGINT UNSIGNED | NOT NULL | 0 | 财务审核人ID | 需求文档5.6 "财务审核" |
| `finance_review_at` | BIGINT UNSIGNED | NOT NULL | 0 | 财务审核时间 | 需求文档5.6 |
| `finance_review_remark` | VARCHAR(512) | NOT NULL | '' | 财务审核备注 | 需求文档5.6 |
| `channel_id` | BIGINT UNSIGNED | NOT NULL | 0 | 代付通道ID | 需求文档5.2通道配置 |
| `channel_order_no` | VARCHAR(64) | NOT NULL | '' | 三方通道订单号 | 对账使用 |
| `fail_reason` | VARCHAR(256) | NOT NULL | '' | 失败原因（信息有误/系统维护/额度受限） | 需求文档4.8 "失败原因分类" |
| `paid_at` | BIGINT UNSIGNED | NOT NULL | 0 | 出款完成时间 | C端展示 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 下单时间 | 工程规范 |
| `update_at` | BIGINT UNSIGNED | NOT NULL | 0 | 最后更新时间 | 工程规范 |
| `delete_at` | BIGINT UNSIGNED | NOT NULL | 0 | 软删除 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `uk_order_no` | `(order_no)` | UNIQUE | 订单号全局唯一 |
| `idx_user_currency_status` | `(user_id, currency_code, status)` | INDEX | C端按币种查提现记录列表 |
| `idx_status_create` | `(status, create_at DESC)` | INDEX | B端按状态筛选（如"审核中"列表） |
| `idx_create_at` | `(create_at DESC)` | INDEX | B端按时间倒序查询 |

**审核规则约束**（需求文档5.6）：
- 创建人（user_id）与审核人（risk/finance_reviewer_id）不能是同一人
- 风控和财务是两个独立审核环节，顺序执行
- 状态变更通过 WHERE status=前置状态 保证状态机合法转换

---

## 九、兑换订单域

### 9.1 t_exchange_order — 兑换订单表

**现实含义**：用户把法币余额兑换为BSB的操作记录

**需求来源**：
- 需求文档4.6兑换模块
- 需求文档4.8记录模块（兑换Tab）

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键 | 工程规范 |
| `order_no` | VARCHAR(32) | NOT NULL | - | 兑换订单号（E+16位） | 自定义前缀 |
| `user_id` | BIGINT UNSIGNED | NOT NULL | 0 | 用户ID | 核心维度 |
| `source_currency` | VARCHAR(16) | NOT NULL | - | 源币种（法币：VND/IDR/THB） | 需求文档4.6 "仅法币→BSB" |
| `target_currency` | VARCHAR(16) | NOT NULL | 'BSB' | 目标币种（固定BSB） | 需求文档4.6 "单向兑换" |
| `source_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 源币种扣减金额（用户输入的法币金额） | 需求文档4.6 |
| `exchange_rate` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 使用的汇率（执行时快照） | 需求文档4.6 "1 BSB = X 法币" |
| `exchange_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 兑换获得BSB = source_amount / exchange_rate | 需求文档4.6计算公式 |
| `bonus_ratio` | DECIMAL(10,4) UNSIGNED | NOT NULL | 0 | 赠送比例（%） | 需求文档4.6 "赠送比例%" |
| `bonus_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 赠送BSB = exchange_amount × bonus_ratio / 100 | 需求文档4.6 "赠送BSB" |
| `actual_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 实际到账BSB = exchange_amount + bonus_amount | 需求文档4.6 "实际到账" |
| `bonus_turnover_multi` | DECIMAL(10,2) UNSIGNED | NOT NULL | 0 | 赠送部分流水倍数要求 | 需求文档4.6 "赠送部分需完成X倍流水" |
| `status` | TINYINT UNSIGNED | NOT NULL | 1 | 订单状态（枚举3.5） | 兑换是同步操作 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 兑换时间 | 工程规范 |
| `update_at` | BIGINT UNSIGNED | NOT NULL | 0 | 更新时间 | 工程规范 |
| `delete_at` | BIGINT UNSIGNED | NOT NULL | 0 | 软删除 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `uk_order_no` | `(order_no)` | UNIQUE | 订单号全局唯一 |
| `idx_user_source_time` | `(user_id, source_currency, create_at DESC)` | INDEX | C端按源币种查兑换记录 |

**设计要点**：
- 兑换是同步原子操作：在一个DB事务中完成（法币debit + BSB credit + 订单写入 + 流水写入）
- 不存在中间状态，创建即终态（成功或失败）
- `target_currency` 固定为 `BSB`，但仍作为字段存储（未来可能开放其他方向）
- 执行时必须重新计算汇率（不能用预览时的快照，防止时间差导致汇率变化）

---

## 十、冻结记录域

### 10.1 t_freeze_record — 冻结记录表

**现实含义**：提现发起时冻结的那笔钱——从冻结到最终"归还"或"扣除"的完整生命周期

**需求来源**：
- 需求文档4.7.6 "用户发起→钱包冻结对应金额→审核→出款→成功/失败"
- 架构设计6.3冻结状态机
- 实现思路12.4防双花

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键 | 工程规范 |
| `user_id` | BIGINT UNSIGNED | NOT NULL | 0 | 用户ID | 核心维度 |
| `currency_code` | VARCHAR(16) | NOT NULL | - | 币种代码 | 跟随账户 |
| `wallet_type` | TINYINT UNSIGNED | NOT NULL | 0 | 子钱包类型 | 跟随账户 |
| `freeze_amount` | DECIMAL(30,8) UNSIGNED | NOT NULL | 0 | 冻结金额 | 需求文档4.7.6 "冻结对应金额" |
| `related_order_no` | VARCHAR(32) | NOT NULL | '' | 关联的提现订单号（T+16位） | 与提现订单关联 |
| `status` | TINYINT UNSIGNED | NOT NULL | 1 | 冻结状态（枚举3.6） | 架构设计6.3 |
| `unfreeze_at` | BIGINT UNSIGNED | NOT NULL | 0 | 解冻/扣款时间 | 终态时间戳 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 冻结时间 | 工程规范 |
| `update_at` | BIGINT UNSIGNED | NOT NULL | 0 | 更新时间 | 工程规范 |
| `delete_at` | BIGINT UNSIGNED | NOT NULL | 0 | 软删除 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `idx_order_no` | `(related_order_no)` | INDEX | 按提现订单号查冻结记录 |
| `idx_user_status` | `(user_id, status)` | INDEX | 查用户所有"冻结中"的记录 |

**状态机规则**：

```
        ┌──────────┐
        │  冻结中   │ ← WalletFreeze 创建（status=1）
        │ status=1 │
        └────┬─────┘
             │
        ┌────┴────┐
        │         │
        ▼         ▼
  ┌──────────┐ ┌──────────┐
  │  已解冻   │ │  已扣款   │
  │ status=2 │ │ status=3 │
  └──────────┘ └──────────┘

关键约束：
  - WalletUnfreeze 操作前：WHERE status=1 → affected_rows=0 则拒绝
  - WalletDeduct 操作前：WHERE status=1 → affected_rows=0 则拒绝
  - 两个终态互斥：同一条记录不可能既解冻又扣款
  - 利用数据库原子性保证状态转换正确
```

---

## 十一、提现账户域

### 11.1 t_withdraw_account — 提现账户表

**现实含义**：用户绑定的提现收款账户（银行卡/电子钱包/USDT地址）

**需求来源**：
- 需求文档4.7.3三种提现方式（各自需要的字段不同）
- 需求文档4.7.4 "首次填写后平台记录，后续复用"、"不可自行修改"
- 需求文档4.7.4 "账户持有人必须=KYC认证姓名"

**字段定义**：

| 字段名 | 类型 | 约束 | 默认值 | 现实含义 | 需求来源 |
|--------|------|------|--------|---------|---------|
| `id` | BIGINT UNSIGNED | PK AUTO_INCREMENT | - | 主键 | 工程规范 |
| `user_id` | BIGINT UNSIGNED | NOT NULL | 0 | 用户ID | 核心维度 |
| `account_type` | TINYINT UNSIGNED | NOT NULL | 0 | 账户类型（1银行卡/2电子钱包/3USDT） | 需求文档4.7.3三种方式 |
| `bank_name` | VARCHAR(128) | NOT NULL | '' | 银行名称（银行卡时必填） | 需求文档4.7.3 "银行名称(下拉选择)" |
| `bank_account` | VARCHAR(64) | NOT NULL | '' | 银行账号（银行卡时必填） | 需求文档4.7.3 "银行账号" |
| `bank_branch` | VARCHAR(128) | NOT NULL | '' | 开户网点（银行卡时选填） | 需求文档4.7.3 "开户网点" |
| `account_holder` | VARCHAR(128) | NOT NULL | '' | 账户持有人（必须=KYC实名） | 需求文档4.7.4 "自动填充KYC姓名，灰显" |
| `e_wallet_type` | VARCHAR(32) | NOT NULL | '' | 电子钱包类型（GoPay/OVO/DANA/ShopeePay/PromptPay） | 需求文档4.7.3 |
| `e_wallet_phone` | VARCHAR(32) | NOT NULL | '' | 电子钱包手机号 | 需求文档4.7.3 "手机号" |
| `chain_address` | VARCHAR(128) | NOT NULL | '' | USDT链上地址 | USDT提现场景 |
| `chain_network` | TINYINT UNSIGNED | NOT NULL | 0 | USDT网络类型（枚举3.10） | USDT提现场景 |
| `currency_code` | VARCHAR(16) | NOT NULL | '' | 关联币种（银行卡/电子钱包关联特定国家币种） | 按币种区分 |
| `create_at` | BIGINT UNSIGNED | NOT NULL | 0 | 首次绑定时间 | 工程规范 |
| `update_at` | BIGINT UNSIGNED | NOT NULL | 0 | 更新时间（正常不更新） | 工程规范 |
| `delete_at` | BIGINT UNSIGNED | NOT NULL | 0 | 软删除 | 工程规范 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| `uk_user_type_currency` | `(user_id, account_type, currency_code)` | UNIQUE | 每种方式每种币种只能绑定一个账户 |
| `idx_user_id` | `(user_id)` | INDEX | 查用户所有提现账户 |

**核心约束**（需求文档4.7.4）：

```
1. 只写一次，不可自行修改
   SaveWithdrawAccount 只处理首次创建
   如果已存在（唯一索引命中）→ 返回"已绑定，不可修改"
   这是风控要求

2. 持有人姓名校验
   account_holder 必须与 KYC 认证姓名一致
   比对规则：忽略大小写 + 忽略多余空格
   SaveWithdrawAccount 时调 ser-kyc.GetUserKycInfo() 获取实名
   strings.EqualFold(trim(account_holder), trim(kyc_name))

3. 一个 account_type 可存多种币种的账户
   例如：user_id=1001 + account_type=1(银行卡) + currency_code='VND' → 越南银行
         user_id=1001 + account_type=1(银行卡) + currency_code='IDR' → 印尼银行
   所以唯一索引包含 currency_code
```

---

## 十二、模型间关系图

### 12.1 全局关系图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           模型关系全景图                                    │
│                                                                          │
│  ┌──────────┐                                                            │
│  │t_currency│ ─────────────────────────────────────────────────────┐     │
│  │          │  currency_code 被以下所有表引用                        │     │
│  └──────┬───┘                                                      │     │
│         │                                                          │     │
│         │ 1:N (一个币种对应多条汇率日志)                              │     │
│         ▼                                                          │     │
│  ┌──────────┐                                                      │     │
│  │t_rate_log│                                                      │     │
│  └──────────┘                                                      │     │
│                                                                    │     │
│  ┌──────────────────┐                                              │     │
│  │t_wallet_account  │ ← 核心：被所有资金操作引用                      │     │
│  │(user_id,         │                                              │     │
│  │ currency_code,   │ ←──────────────────────────────────────────  │     │
│  │ wallet_type)唯一  │                                              │     │
│  └────────┬─────────┘                                              │     │
│           │                                                        │     │
│           │ 1:N (一个账户对应多条流水)                                │     │
│           ▼                                                        │     │
│  ┌──────────────────┐                                              │     │
│  │ t_wallet_flow    │ ← 审计：不可变的资金变动记录                     │     │
│  │ uk: (order_no,   │                                              │     │
│  │     flow_type)   │ ← 幂等保障                                    │     │
│  └────────┬─────────┘                                              │     │
│           │                                                        │     │
│           │ related_order_no 关联到以下订单表                        │     │
│           │                                                        │     │
│  ┌────────┼──────────────┬──────────────────┐                      │     │
│  │        │              │                  │                      │     │
│  ▼        ▼              ▼                  ▼                      │     │
│ ┌─────────────┐  ┌──────────────┐  ┌──────────────┐              │     │
│ │t_deposit_   │  │t_withdraw_   │  │t_exchange_   │              │     │
│ │  order      │  │  order       │  │  order       │              │     │
│ │ C+16位      │  │ T+16位       │  │ E+16位       │              │     │
│ └─────────────┘  └──────┬───────┘  └──────────────┘              │     │
│                         │                                         │     │
│                         │ 1:1 (一笔提现对应一条冻结)                │     │
│                         ▼                                         │     │
│                  ┌──────────────┐                                  │     │
│                  │t_freeze_     │                                  │     │
│                  │  record      │                                  │     │
│                  └──────────────┘                                  │     │
│                                                                    │     │
│                  ┌──────────────┐                                  │     │
│                  │t_withdraw_   │                                  │     │
│                  │  account     │  ← t_withdraw_order.             │     │
│                  │ (user绑定)   │     withdraw_account_id 引用     │     │
│                  └──────────────┘                                  │     │
│                                                                    │     │
└──────────────────────────────────────────────────────────────────────────┘
```

### 12.2 关系说明

| 关系 | 类型 | 说明 |
|------|------|------|
| t_currency → t_wallet_account | 1:N | 一个币种下有多个用户的多个子钱包账户 |
| t_currency → t_rate_log | 1:N | 一个币种有多条汇率变更日志 |
| t_wallet_account → t_wallet_flow | 1:N | 一个账户有多条流水记录 |
| t_wallet_flow → t_deposit_order | N:1 | 多条流水可以关联同一个充值订单（如入账+奖金） |
| t_wallet_flow → t_withdraw_order | N:1 | 多条流水关联同一个提现订单（冻结+解冻/扣款） |
| t_wallet_flow → t_exchange_order | N:1 | 多条流水关联同一个兑换订单（法币扣减+BSB入账+赠送） |
| t_withdraw_order → t_freeze_record | 1:1 | 一笔提现对应一条冻结记录 |
| t_withdraw_order → t_withdraw_account | N:1 | 多笔提现可以用同一个提现账户 |

### 12.3 关联方式

所有表之间使用**逻辑外键**（字段引用），不建物理外键约束。原因：
- TiDB 对外键支持有限
- 物理外键会降低写入性能
- 工程中现有服务均不使用物理外键
- 由代码层（Service + Repository）保证引用完整性

---

## 十三、索引设计总表

### 13.1 全部索引一览

| 表名 | 索引名 | 字段 | 类型 | 服务的查询场景 |
|------|--------|------|------|-------------|
| t_currency | uk_currency_code | (currency_code) | UNIQUE | 币种代码唯一性 |
| t_currency | idx_status_sort | (status, sort_value) | INDEX | C端可用币种列表 |
| t_rate_log | idx_currency_create | (currency_code, create_at DESC) | INDEX | B端按币种查日志 |
| t_rate_log | idx_create_at | (create_at DESC) | INDEX | B端按时间翻页 |
| t_wallet_account | uk_user_currency_type | (user_id, currency_code, wallet_type) | UNIQUE | 账户三维唯一 |
| t_wallet_account | idx_user_currency | (user_id, currency_code) | INDEX | 按币种查所有子钱包 |
| t_wallet_flow | uk_order_flow | (related_order_no, flow_type) | UNIQUE | 幂等保障 |
| t_wallet_flow | idx_user_currency_time | (user_id, currency_code, wallet_type, create_at DESC) | INDEX | 用户流水查询 |
| t_wallet_flow | idx_order_no | (related_order_no) | INDEX | 按订单号追溯流水 |
| t_deposit_order | uk_order_no | (order_no) | UNIQUE | 订单号唯一 |
| t_deposit_order | idx_user_currency_status | (user_id, currency_code, status) | INDEX | C端充值记录列表 |
| t_deposit_order | idx_user_currency_amount_status | (user_id, currency_code, amount, status) | INDEX | 重复转账拦截 |
| t_deposit_order | idx_channel_order | (channel_order_no) | INDEX | 三方回调查找 |
| t_deposit_order | idx_create_at | (create_at DESC) | INDEX | B端按时间查询 |
| t_withdraw_order | uk_order_no | (order_no) | UNIQUE | 订单号唯一 |
| t_withdraw_order | idx_user_currency_status | (user_id, currency_code, status) | INDEX | C端提现记录列表 |
| t_withdraw_order | idx_status_create | (status, create_at DESC) | INDEX | B端按状态筛选 |
| t_withdraw_order | idx_create_at | (create_at DESC) | INDEX | B端按时间查询 |
| t_exchange_order | uk_order_no | (order_no) | UNIQUE | 订单号唯一 |
| t_exchange_order | idx_user_source_time | (user_id, source_currency, create_at DESC) | INDEX | C端兑换记录列表 |
| t_freeze_record | idx_order_no | (related_order_no) | INDEX | 按提现单号查冻结 |
| t_freeze_record | idx_user_status | (user_id, status) | INDEX | 查用户冻结中记录 |
| t_withdraw_account | uk_user_type_currency | (user_id, account_type, currency_code) | UNIQUE | 每种方式每种币种唯一 |
| t_withdraw_account | idx_user_id | (user_id) | INDEX | 查用户所有提现账户 |

### 13.2 索引设计原则

```
1. 唯一索引优先用于业务约束
   uk_currency_code → 币种代码不能重复
   uk_user_currency_type → 账户不能重复创建
   uk_order_flow → 幂等保障不能重复入账
   uk_order_no → 订单号不能重复

2. 复合索引遵循最左前缀原则
   查询条件从左到右匹配
   例如 idx_user_currency_status 同时服务于：
     WHERE user_id = ?
     WHERE user_id = ? AND currency_code = ?
     WHERE user_id = ? AND currency_code = ? AND status = ?

3. 覆盖高频查询模式
   C端列表查询 → (user_id, currency_code, ...) 开头
   B端管理查询 → (status, create_at) 或 (create_at) 开头
   重复拦截查询 → (user_id, currency_code, amount, status) 精确匹配

4. 避免过度索引
   每个表的写操作都会更新索引
   只为实际查询场景建索引，不预建"可能用到"的索引
```

---

## 十四、建表语句参考

> 以下为完整的建表 SQL，可直接在 TiDB 中执行
> 执行后配置 gorm_gen.go 生成 model 和 query 代码

```sql
-- ============================================================
-- 币种配置域
-- ============================================================

CREATE TABLE `t_currency` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `currency_code` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '币种代码(VND/USDT/BSB)',
  `currency_name` VARCHAR(64) NOT NULL DEFAULT '' COMMENT '币种名称',
  `currency_type` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '类型:1法币 2加密 3平台',
  `country` VARCHAR(32) NOT NULL DEFAULT '' COMMENT '所属国家',
  `icon_url` VARCHAR(512) NOT NULL DEFAULT '' COMMENT '图标URL',
  `currency_symbol` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '货币符号',
  `thousands_sep` VARCHAR(4) NOT NULL DEFAULT ',' COMMENT '千分位分隔符',
  `decimal_sep` VARCHAR(4) NOT NULL DEFAULT '.' COMMENT '小数点符号',
  `precision` TINYINT UNSIGNED NOT NULL DEFAULT 2 COMMENT '小数位数',
  `real_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '实时汇率(API平均)',
  `platform_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '平台汇率',
  `rate_fluctuation` DECIMAL(10,4) UNSIGNED NOT NULL DEFAULT 0.0000 COMMENT '汇率浮动比例%',
  `rate_threshold` DECIMAL(10,4) UNSIGNED NOT NULL DEFAULT 0.0000 COMMENT '汇率偏差阈值%',
  `deposit_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '入款汇率',
  `withdraw_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '出款汇率',
  `is_base` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '是否基准币种:0否 1是',
  `sort_value` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '排序值',
  `status` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态:1启用 2禁用',
  `create_by` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '创建人ID',
  `update_by` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '修改人ID',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '创建时间',
  `update_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '更新时间',
  `delete_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '软删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_currency_code` (`currency_code`),
  KEY `idx_status_sort` (`status`, `sort_value`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='币种配置表';

CREATE TABLE `t_rate_log` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `currency_code` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '币种代码',
  `real_rate_before` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '更新前实时汇率',
  `real_rate_after` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '更新后实时汇率',
  `platform_rate_before` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '更新前平台汇率',
  `platform_rate_after` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '更新后平台汇率',
  `deposit_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '入款汇率',
  `withdraw_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '出款汇率',
  `deviation_pct` DECIMAL(10,4) NOT NULL DEFAULT 0.0000 COMMENT '偏差百分比',
  `trigger_type` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '触发方式:1自动 2手动',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '日志时间',
  `delete_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '软删除',
  PRIMARY KEY (`id`),
  KEY `idx_currency_create` (`currency_code`, `create_at` DESC),
  KEY `idx_create_at` (`create_at` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='汇率变更日志表';

-- ============================================================
-- 钱包核心域
-- ============================================================

CREATE TABLE `t_wallet_account` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户ID',
  `currency_code` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '币种代码',
  `wallet_type` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '子钱包类型:1中心 2奖励 3主播 4代理 5场馆',
  `available_balance` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '可用余额',
  `frozen_balance` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '冻结余额',
  `version` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '创建时间',
  `update_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '更新时间',
  `delete_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '软删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_currency_type` (`user_id`, `currency_code`, `wallet_type`),
  KEY `idx_user_currency` (`user_id`, `currency_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='子钱包账户表';

CREATE TABLE `t_wallet_flow` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户ID',
  `currency_code` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '币种代码',
  `wallet_type` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '子钱包类型',
  `flow_type` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '流水类型',
  `amount` DECIMAL(30,8) NOT NULL DEFAULT 0.00000000 COMMENT '变动金额(正增负减)',
  `balance_before` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '变动前可用余额',
  `balance_after` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '变动后可用余额',
  `frozen_before` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '变动前冻结余额',
  `frozen_after` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '变动后冻结余额',
  `related_order_no` VARCHAR(32) NOT NULL DEFAULT '' COMMENT '关联订单号',
  `related_biz_type` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '关联业务类型:1充值 2提现 3兑换 4修正',
  `remark` VARCHAR(512) NOT NULL DEFAULT '' COMMENT '备注',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_flow` (`related_order_no`, `flow_type`),
  KEY `idx_user_currency_time` (`user_id`, `currency_code`, `wallet_type`, `create_at` DESC),
  KEY `idx_order_no` (`related_order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='资金流水表';

CREATE TABLE `t_freeze_record` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户ID',
  `currency_code` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '币种代码',
  `wallet_type` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '子钱包类型',
  `freeze_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '冻结金额',
  `related_order_no` VARCHAR(32) NOT NULL DEFAULT '' COMMENT '关联提现订单号',
  `status` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态:1冻结中 2已解冻 3已扣款',
  `unfreeze_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '解冻/扣款时间',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '冻结时间',
  `update_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '更新时间',
  `delete_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '软删除',
  PRIMARY KEY (`id`),
  KEY `idx_order_no` (`related_order_no`),
  KEY `idx_user_status` (`user_id`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='冻结记录表';

-- ============================================================
-- 充值域
-- ============================================================

CREATE TABLE `t_deposit_order` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `order_no` VARCHAR(32) NOT NULL DEFAULT '' COMMENT '充值订单号(C+16位)',
  `user_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户ID',
  `currency_code` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '充值币种',
  `deposit_method` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '充值方式:1USDT 2银行转账 3电子钱包',
  `amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '充值金额',
  `actual_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '实际到账金额',
  `usdt_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT 'USDT换算金额',
  `exchange_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '使用汇率快照',
  `network_type` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'USDT网络:1TRC20 2ERC20 3BEP20',
  `tx_hash` VARCHAR(128) NOT NULL DEFAULT '' COMMENT '链上交易哈希',
  `deposit_address` VARCHAR(128) NOT NULL DEFAULT '' COMMENT 'USDT收款地址',
  `channel_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '支付通道ID',
  `channel_order_no` VARCHAR(64) NOT NULL DEFAULT '' COMMENT '三方通道订单号',
  `bonus_activity_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '额外奖金活动ID',
  `bonus_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '奖金金额',
  `bonus_turnover_multi` DECIMAL(10,2) UNSIGNED NOT NULL DEFAULT 0.00 COMMENT '奖金流水倍数',
  `status` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态:1待支付 2处理中 3已完成 4已失败',
  `fail_reason` VARCHAR(256) NOT NULL DEFAULT '' COMMENT '失败原因',
  `paid_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '支付完成时间',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '下单时间',
  `update_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '更新时间',
  `delete_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '软删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),
  KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`),
  KEY `idx_user_currency_amount_status` (`user_id`, `currency_code`, `amount`, `status`),
  KEY `idx_channel_order` (`channel_order_no`),
  KEY `idx_create_at` (`create_at` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='充值订单表';

-- ============================================================
-- 提现域
-- ============================================================

CREATE TABLE `t_withdraw_order` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `order_no` VARCHAR(32) NOT NULL DEFAULT '' COMMENT '提现订单号(T+16位)',
  `user_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户ID',
  `currency_code` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '提现币种',
  `wallet_type` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '来源子钱包类型',
  `withdraw_method` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '提现方式:1银行卡 2电子钱包 3USDT',
  `withdraw_account_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '提现账户ID',
  `amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '提现金额',
  `fee_rate` DECIMAL(10,4) UNSIGNED NOT NULL DEFAULT 0.0000 COMMENT '手续费率%',
  `fee_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '手续费金额',
  `actual_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '到账金额',
  `exchange_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '使用汇率(BSB提现换算)',
  `freeze_record_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '冻结记录ID',
  `status` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态:1审核中 2风控通过 3风控驳回 4财务通过 5财务驳回 6出款中 7出款成功 8出款失败',
  `risk_reviewer_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '风控审核人ID',
  `risk_review_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '风控审核时间',
  `risk_review_remark` VARCHAR(512) NOT NULL DEFAULT '' COMMENT '风控审核备注',
  `finance_reviewer_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '财务审核人ID',
  `finance_review_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '财务审核时间',
  `finance_review_remark` VARCHAR(512) NOT NULL DEFAULT '' COMMENT '财务审核备注',
  `channel_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '代付通道ID',
  `channel_order_no` VARCHAR(64) NOT NULL DEFAULT '' COMMENT '三方通道订单号',
  `fail_reason` VARCHAR(256) NOT NULL DEFAULT '' COMMENT '失败原因',
  `paid_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '出款完成时间',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '下单时间',
  `update_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '更新时间',
  `delete_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '软删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),
  KEY `idx_user_currency_status` (`user_id`, `currency_code`, `status`),
  KEY `idx_status_create` (`status`, `create_at` DESC),
  KEY `idx_create_at` (`create_at` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='提现订单表';

CREATE TABLE `t_withdraw_account` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户ID',
  `account_type` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '账户类型:1银行卡 2电子钱包 3USDT',
  `bank_name` VARCHAR(128) NOT NULL DEFAULT '' COMMENT '银行名称',
  `bank_account` VARCHAR(64) NOT NULL DEFAULT '' COMMENT '银行账号',
  `bank_branch` VARCHAR(128) NOT NULL DEFAULT '' COMMENT '开户网点',
  `account_holder` VARCHAR(128) NOT NULL DEFAULT '' COMMENT '账户持有人(=KYC实名)',
  `e_wallet_type` VARCHAR(32) NOT NULL DEFAULT '' COMMENT '电子钱包类型',
  `e_wallet_phone` VARCHAR(32) NOT NULL DEFAULT '' COMMENT '电子钱包手机号',
  `chain_address` VARCHAR(128) NOT NULL DEFAULT '' COMMENT 'USDT链上地址',
  `chain_network` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'USDT网络类型',
  `currency_code` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '关联币种',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '绑定时间',
  `update_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '更新时间',
  `delete_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '软删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_type_currency` (`user_id`, `account_type`, `currency_code`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='提现账户表';

-- ============================================================
-- 兑换域
-- ============================================================

CREATE TABLE `t_exchange_order` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `order_no` VARCHAR(32) NOT NULL DEFAULT '' COMMENT '兑换订单号(E+16位)',
  `user_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户ID',
  `source_currency` VARCHAR(16) NOT NULL DEFAULT '' COMMENT '源币种(法币)',
  `target_currency` VARCHAR(16) NOT NULL DEFAULT 'BSB' COMMENT '目标币种(固定BSB)',
  `source_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '源币种扣减金额',
  `exchange_rate` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '使用汇率快照',
  `exchange_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '兑换获得BSB',
  `bonus_ratio` DECIMAL(10,4) UNSIGNED NOT NULL DEFAULT 0.0000 COMMENT '赠送比例%',
  `bonus_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '赠送BSB',
  `actual_amount` DECIMAL(30,8) UNSIGNED NOT NULL DEFAULT 0.00000000 COMMENT '实际到账BSB',
  `bonus_turnover_multi` DECIMAL(10,2) UNSIGNED NOT NULL DEFAULT 0.00 COMMENT '赠送流水倍数',
  `status` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态:1成功 2失败',
  `create_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '兑换时间',
  `update_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '更新时间',
  `delete_at` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '软删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),
  KEY `idx_user_source_time` (`user_id`, `source_currency`, `create_at` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='兑换订单表';
```

---

## 十五、后续怎么做

### 15.1 建表 → 代码生成 → 开始编码

```
Step 1: 在 TiDB 中执行上述建表 SQL（连接 db_wallet 数据库）
Step 2: 确认 ETCD 配置 /slg/serv/wallet/db → db_wallet
Step 3: 编写 ser-wallet/internal/gen/gorm_gen.go（参照 ser-app 的 gorm_gen.go）
Step 4: 执行 go run gorm_gen.go → 自动生成 internal/gorm_gen/model/ 和 query/
Step 5: 在 internal/enum/ 下定义本文第三章的所有枚举常量
Step 6: 开始编写 Repository 层（internal/rep/），封装对生成代码的调用
Step 7: 开始编写 Service 层（internal/ser/），实现业务逻辑
```

### 15.2 数据模型与代码结构的映射

```
t_currency           → model.TCurrency          → CurrencyRepo      → CurrencyService
t_rate_log           → model.TRateLog           → RateLogRepo       → CurrencyService
t_wallet_account     → model.TWalletAccount     → WalletRepo        → WalletService
t_wallet_flow        → model.TWalletFlow        → FlowRepo          → WalletService
t_freeze_record      → model.TFreezeRecord      → FreezeRepo        → WalletService
t_deposit_order      → model.TDepositOrder      → DepositRepo       → DepositService
t_withdraw_order     → model.TWithdrawOrder     → WithdrawRepo      → WithdrawService
t_withdraw_account   → model.TWithdrawAccount   → WithdrawAcctRepo  → WithdrawService
t_exchange_order     → model.TExchangeOrder     → ExchangeRepo      → ExchangeService
```

### 15.3 本文给出了什么

| 你得到了 | 后续用它来 |
|---------|-----------|
| 9张表的完整字段定义（类型+约束+默认值+注释） | 直接建表 |
| 14种枚举的完整取值范围 | 编写 enum 常量代码 |
| 24个索引的精确定义 | 建表时同步创建 |
| 全部表间关系 | 编写 Repository 和 Service 时知道数据怎么关联 |
| 完整的建表 SQL | 直接复制执行 |
| 每个设计决策的需求来源标注 | 评审时可追溯"为什么这么设计" |

### 15.4 注意事项

```
1. 所有金额字段统一 DECIMAL(30,8)
   即使法币只需要2位小数，存储统一8位
   展示层按 t_currency.precision 截断显示
   这避免了不同币种需要不同表结构的问题

2. t_wallet_flow.amount 是唯一一个 SIGNED DECIMAL 字段
   其他所有金额字段都是 UNSIGNED
   amount 正数=增加，负数=减少

3. t_wallet_flow 没有 update_at 和 delete_at
   流水是不可变记录，不更新不删除
   这是审计和对账的基础

4. 订单表的状态变更通过 WHERE status=前置状态 控制
   不是代码级的 if-else，是数据库原子性保证
   affected_rows=0 即状态不合法

5. 评审后可能调整的点
   - 代理/场馆钱包是否初始化（影响 wallet_type 枚举使用范围）
   - 充值/提现方式配置字段（影响 deposit_method/withdraw_method 枚举）
   - 额外奖金数据来源（影响 bonus_activity_id 的关联方式）
   - 这些不影响表结构，只影响字段取值范围
```
