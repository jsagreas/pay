# ser-wallet 钱包模块·全维度实现总结

> 编写时间：2026-02-28
> 定位：需求评审会前的**完整备战文档**——从设计思路到技术细节，从表结构到接口清单，从技术挑战到应对方案，从开放问题到预期Q&A
> 数据来源：20+份分析文档（接口评估×11、链路关系、依赖关系、IDL设计、实现思路、流程思路、钱包结构、前置信息、模块参考A/B、需求理解、评审反馈、RPC暴露、RPC调用、接口统计等）+ 6文件夹 ~155张产品原型图 + 工程源码实证
> 核心目标：**阅读此文档后，能在需求大会上全方位无死角地阐述钱包模块的设计与实现**

---

## 目录

```
一、模块定位与边界 .................. 我们到底负责什么？
二、设计总思路 ...................... 从0到1的思维链路
三、四大子模块架构 .................. 内部如何拆分？
四、14张数据表设计 .................. 存什么？怎么存？
五、完整接口清单（50个） ............ 对外暴露什么能力？
六、技术栈与工程基础设施 ............ 用什么技术？怎么搭？
七、核心技术挑战与解决方案 .......... 难点在哪？怎么解？
八、三大业务流程详解 ................ 充值/提现/兑换怎么跑？
九、外部依赖与协作方 ................ 我调谁？谁调我？
十、开放问题清单（需产品确认） ...... 还有什么没确定？
十一、风险识别与应对 ................ 可能翻车的地方
十二、开发节奏与里程碑 .............. 怎么分批交付？
十三、评审会Q&A预演 ................ 被问到怎么答？
十四、一页纸速览 ................... 30秒说清楚整个模块
```

---

## 一、模块定位与边界

### 1.1 一句话定位

> **ser-wallet 是整个平台的资金中枢**——所有涉及用户钱的操作（充值入账、消费扣款、提现冻结/出款、兑换、活动返奖、人工修正）最终都经过钱包模块完成余额变更。

### 1.2 职责范围

```
我方主责（必须我们实现）：
  ✅ 多币种配置管理（币种 CRUD + 汇率规则 + 基准币种）
  ✅ 用户钱包余额管理（中心钱包 + 奖励钱包，预留主播/代理）
  ✅ 钱包核心操作（上账/扣款/冻结/解冻/确认扣除 6个原子操作）
  ✅ 充值/提现/兑换业务流程（C端）
  ✅ 交易记录查询（C端 + B端）
  ✅ B端币种配置管理 + 用户钱包查看
  ✅ 汇率定时刷新（内置 cron + 三方 API）
  ✅ 对外 RPC 接口（供财务/投注/直播/活动模块调用）

不是我们负责（但需要了解全貌）：
  ❌ 支付通道对接（财务模块负责）
  ❌ 支付回调验签处理（财务模块负责）
  ❌ 风控审核流程（风控模块负责）
  ❌ 财务审核流程（财务模块负责）
  ❌ 活动/任务规则配置（活动模块负责）
  ❌ KYC认证流程（ser-kyc负责）
```

### 1.3 钱包在业务链路中的位置

```
用户操作         │  ser-wallet 角色                  │  后续流程
─────────────────┼──────────────────────────────────┼──────────────────
充值             │  创建订单→调财务匹配通道          │  财务→通道→回调→CreditWallet上账
投注/消费        │  DebitWallet 扣款（被投注模块调）  │  投注模块→结算→CreditWallet返奖
提现             │  创建订单→冻结余额→提交财务审核    │  风控→财务→出款→ConfirmDebit确认
兑换             │  法币→BSB，全部在钱包内部完成      │  无外部依赖
活动返奖         │  RewardCredit入账到奖励钱包        │  活动模块调我们
人工加/减款      │  ManualCredit/ManualDebit          │  财务B端审核通过后调我们
```

### 1.4 核心业务规则（13条确认规则）

| 编号 | 规则 | 来源 |
|------|------|------|
| R1 | 币种隔离：每个币种独立余额、独立流水 | 需求4.1 |
| R2 | 余额公式：可用资金 = 中心钱包 + 奖励钱包（不含冻结） | 需求2.1 |
| R3 | 消费优先级：中心钱包优先→奖励钱包补足 | 需求2.1 |
| R4 | BSB锚定：10 BSB = 1 USDT（固定比例） | 需求3.2 |
| R5 | 基准币种：USDT，设置后不可更改 | 需求3.2 |
| R6 | 兑换方向：法币→BSB 单向（不可反向） | 需求6.1 |
| R7 | 提现前提：必须通过KYC认证（status=3） | 需求7.1 |
| R8 | 提现冻结：申请后立即冻结，驳回/失败则退回 | 需求7.1 |
| R9 | 幂等保障：所有余额变更通过 orderNo 唯一键防重复 | 架构设计 |
| R10 | 金额精度：全链路使用 i64 整数（最小单位），不用浮点数 | IDL设计 |
| R11 | 充值重复检测：同用户+同币种+同金额，1-2笔警告，3笔阻断 | 需求5.4 |
| R12 | 人工减款级联：余额从高到低的钱包类型依次扣减 | 财务2.53 |
| R13 | 汇率刷新：定时查3家API取平均，偏差≥阈值%才更新平台汇率 | 需求4.2 |

---

## 二、设计总思路

### 2.1 从0到1的设计链路

```
第一步：理解产品需求
  → 6文件夹155张原型图 + 需求文档
  → 明确"我负责什么"（钱包+币种配置）vs"别人负责什么"（财务/风控/活动）

第二步：识别确定性骨架
  → 多币种 + 多钱包类型 → 表结构必须币种×钱包类型二维隔离
  → 充提兑三条核心业务线 → 对应三组接口
  → 财务模块双向依赖 → 需要RPC暴露+RPC调用两套接口

第三步：IDL-First 契约定义
  → 8个 Thrift IDL 文件，38个方法声明
  → 先定义接口契约，再实现业务逻辑
  → IDL 确定后即可自动生成框架代码（gen_kitex.sh）

第四步：四子模块分层
  → 币种配置（底层，无依赖）→ 钱包核心（原子操作）→ 交易业务（组合逻辑）→ 记录查询（纯读）
  → 层级清晰，单向依赖，不存在循环调用

第五步：分批迭代开发
  → P0 纯内部逻辑 → P1 接入已有服务 → P2 Mock财务跑通 → P3 联调替换Mock
  → 每批独立可测，不等全部就绪
```

### 2.2 盖房子思维

```
地基 = 基础设施注册（ETCD端口、数据库、Client工厂）
  ↓
承重墙 = 币种配置 + 钱包核心6个原子操作
  ↓
楼层 = 充值/提现/兑换三条业务线
  ↓
装修 = 记录查询 + B端管理 + 汇率定时任务
  ↓
物业对接 = 与财务/投注/活动模块联调
```

### 2.3 两个编码维度

```
维度一：规范标准（工程一致性）
  → IDL命名规范跟随工程惯例（namespace go ser_wallet）
  → 目录结构跟随 ser-app/ser-item 模式（5层：cfg/enum+errs/ser/rep+cache/gen）
  → RPC Client 注册跟随 rpc_client.go 的 sync.Once 模式
  → 审计日志跟随 ser-blog.AddActLog oneway 模式
  → 错误码跟随 ret.BizErr + ser-i18n Redis直读模式

维度二：需求匹配（业务正确性）
  → 每个接口有明确的产品原型页面对应
  → 每条业务规则有需求文档条目佐证
  → 每个技术方案有模块参考（Java p9钱包/TRC20钱包）验证
```

---

## 三、四大子模块架构

### 3.1 模块关系图

```
┌─────────────────────────────────────────────────────────────┐
│                     ser-wallet                              │
│                                                             │
│  ┌──────────────┐                                           │
│  │ A.币种配置     │ ← 底层模块，无依赖                        │
│  │ currency_ser  │   提供币种/汇率数据给所有上层               │
│  └──────┬───────┘                                           │
│         │ 供给币种/汇率数据                                   │
│         ▼                                                   │
│  ┌──────────────┐                                           │
│  │ B.钱包核心     │ ← 6个原子操作（credit/debit/freeze/...）   │
│  │ wallet_core   │   所有余额变更的唯一入口                   │
│  └──────┬───────┘                                           │
│         │ 提供原子余额操作                                    │
│         ▼                                                   │
│  ┌──────────────┐                                           │
│  │ C.交易业务     │ ← 充值/提现/兑换三条业务线                 │
│  │ deposit_ser   │   组合 A+B 实现复杂业务流程                │
│  │ withdraw_ser  │   对接外部服务（财务/KYC/活动）             │
│  │ exchange_ser  │                                           │
│  └──────┬───────┘                                           │
│         │ 产生交易记录                                        │
│         ▼                                                   │
│  ┌──────────────┐                                           │
│  │ D.记录查询     │ ← 纯读模块                               │
│  │ record_ser    │   聚合所有交易记录（充值/提现/兑换/奖励）     │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 各子模块职责

**A. 币种配置模块（currency_service.go）**

```
B端：
  PageCurrency     — 币种列表分页查询（含20+字段）
  EditCurrency     — 编辑币种（浮动%、阈值%、图标、精度规则）
  StatusCurrency   — 启用/禁用币种
  SetBaseCurrency  — 设置基准币种（一次性操作，USDT）
  PageRateLog      — 汇率变更日志查询

C端：
  GetCurrencyList  — 可用币种列表（带用户余额）

内部：
  RefreshExchangeRate — 定时刷新汇率（cron任务）

RPC暴露：
  GetCurrencyConfig    — 币种配置查询（供其他模块）
  GetExchangeRateRpc   — 汇率查询（供其他模块）
```

**B. 钱包核心模块（wallet_core.go）**

```
6个原子操作（所有余额变更的唯一入口）：

  creditCore(uid, currency, walletType, amount, orderNo)
    → 幂等校验 → 自动建钱包 → SQL原子UPDATE加款 → 写流水

  debitCore(uid, currency, amount, orderNo)
    → 幂等校验 → 中心优先扣→奖励补足 → SQL原子UPDATE WHERE balance >= ? → 写流水

  freezeCore(uid, currency, amount, orderNo)
    → 可用余额 -= amount → 冻结余额 += amount → 写冻结记录

  unfreezeCore(uid, currency, amount, orderNo)
    → 校验冻结记录状态 → 冻结余额 -= amount → 可用余额 += amount

  confirmDebitCore(uid, currency, amount, orderNo)
    → 校验冻结记录状态 → 冻结余额 -= amount（资金永久离开）

  writeFlow(...)
    → 写入 wallet_flow 流水表（每个操作都调）

RPC暴露（包装上述原子操作）：
  CreditWallet / DebitWallet / FreezeBalance / UnfreezeBalance / ConfirmDebit
  ManualCredit / ManualDebit / SupplementCredit
  GetBalance / BatchGetBalance / RewardCredit
```

**C. 交易业务模块**

```
deposit_service.go（充值）：
  GetDepositMethods  — 充值方式列表（→调财务接口）
  CreateDepositOrder — 创建充值订单（→调财务创建通道订单）
  CheckDuplicate     — 重复转账检测
  GetDepositStatus   — 充值状态查询

withdraw_service.go（提现）：
  GetWithdrawMethods — 提现方式列表（→调KYC校验+财务接口）
  GetWithdrawAccount — 获取已保存的提现账户
  SaveWithdrawAccount— 保存提现账户（→调KYC+User获取姓名）
  CreateWithdrawOrder— 创建提现订单（→冻结余额→提交财务审核）

exchange_service.go（兑换）：
  GetExchangeRate    — 汇率查询/兑换预览（含赠送计算）
  DoExchange         — 执行兑换（扣法币→加BSB→加赠送→写稽核流水）
```

**D. 记录查询模块（record_service.go）**

```
  GetRecordList  — 按Tab分类分页查询（充值/提现/兑换/奖励）
  GetRecordDetail— 单条记录详情
```

---

## 四、14张数据表设计

### 4.1 表结构总览

```
核心必须（8张）：
  ┌──────────────────────────────────────────────────────────┐
  │  t_currency_config     币种配置表（10+字段）               │
  │  t_user_wallet         用户钱包表（余额核心表）            │
  │  t_wallet_flow         流水记录表（每笔余额变更留痕）       │
  │  t_wallet_freeze       冻结管理表（提现冻结状态机）         │
  │  t_deposit_order       充值订单表                         │
  │  t_withdraw_order      提现订单表                         │
  │  t_withdraw_account    提现账户表                         │
  │  t_exchange_order      兑换订单表                         │
  └──────────────────────────────────────────────────────────┘

应该有（3张）：
  ┌──────────────────────────────────────────────────────────┐
  │  t_exchange_rate_log   汇率变更日志表                      │
  │  t_currency_rate       币种汇率表（独立于config，高频更新） │
  │  t_audit_flow          稽核流水表（奖励钱包解锁追踪）       │
  └──────────────────────────────────────────────────────────┘

可选/预留（3张）：
  ┌──────────────────────────────────────────────────────────┐
  │  t_wallet_snapshot     钱包快照表（日终对账）               │
  │  t_deposit_address     USDT充值地址表（链上场景）           │
  │  t_withdraw_limit      提现限额配置表（按币种/等级差异化）   │
  └──────────────────────────────────────────────────────────┘
```

### 4.2 核心表字段设计

#### 表1：t_currency_config（币种配置）

```sql
CREATE TABLE t_currency_config (
  id              BIGINT       PRIMARY KEY AUTO_INCREMENT,
  currency_code   VARCHAR(10)  NOT NULL UNIQUE,    -- 币种代码（BRL/USDT/PHP...）
  currency_name   VARCHAR(50)  NOT NULL,           -- 币种名称
  currency_type   TINYINT      NOT NULL,           -- 1=法币 2=虚拟币
  icon_url        VARCHAR(255) DEFAULT '',         -- 图标URL
  icon_s3_key     VARCHAR(255) DEFAULT '',         -- S3 Key
  symbol          VARCHAR(10)  DEFAULT '',         -- 货币符号（R$/₮/₱）
  decimal_places  TINYINT      NOT NULL DEFAULT 2, -- 小数精度
  thousand_sep    VARCHAR(5)   DEFAULT ',',        -- 千分位分隔符
  status          TINYINT      NOT NULL DEFAULT 0, -- 0=禁用 1=启用
  is_base         TINYINT      NOT NULL DEFAULT 0, -- 0=非基准 1=基准币种
  float_rate      DECIMAL(5,2) DEFAULT 0.00,       -- 汇率浮动百分比
  deviation_threshold DECIMAL(5,2) DEFAULT 0.00,   -- 偏差触发阈值百分比
  sort_order      INT          DEFAULT 0,          -- 排序权重
  create_by       BIGINT       DEFAULT 0,
  update_by       BIGINT       DEFAULT 0,
  created_at      BIGINT       NOT NULL,           -- 创建时间戳
  updated_at      BIGINT       NOT NULL,           -- 更新时间戳
  deleted_at      BIGINT       DEFAULT 0           -- 软删除
);
```

#### 表2：t_user_wallet（用户钱包 — 最核心的表）

```sql
CREATE TABLE t_user_wallet (
  id                BIGINT  PRIMARY KEY AUTO_INCREMENT,
  uid               BIGINT  NOT NULL,              -- 用户UID
  currency_code     VARCHAR(10) NOT NULL,           -- 币种代码
  wallet_type       TINYINT NOT NULL,               -- 1=中心 2=奖励 3=主播 4=代理
  available_balance BIGINT  NOT NULL DEFAULT 0,     -- 可用余额（最小单位整数）
  frozen_balance    BIGINT  NOT NULL DEFAULT 0,     -- 冻结余额（仅中心钱包有）
  status            TINYINT NOT NULL DEFAULT 1,     -- 1=正常 2=冻结 3=封禁
  created_at        BIGINT  NOT NULL,
  updated_at        BIGINT  NOT NULL,
  deleted_at        BIGINT  DEFAULT 0,

  UNIQUE KEY uk_uid_currency_type (uid, currency_code, wallet_type)
);

-- 核心SQL（原子操作）：
-- 加款：UPDATE t_user_wallet SET available_balance = available_balance + ? WHERE uid=? AND currency_code=? AND wallet_type=?
-- 扣款：UPDATE t_user_wallet SET available_balance = available_balance - ? WHERE uid=? AND currency_code=? AND wallet_type=? AND available_balance >= ?
-- 冻结：UPDATE t_user_wallet SET available_balance = available_balance - ?, frozen_balance = frozen_balance + ? WHERE uid=? AND currency_code=? AND wallet_type=1 AND available_balance >= ?
```

#### 表3：t_wallet_flow（流水记录 — 最重要的审计表）

```sql
CREATE TABLE t_wallet_flow (
  id              BIGINT      PRIMARY KEY AUTO_INCREMENT,
  uid             BIGINT      NOT NULL,
  currency_code   VARCHAR(10) NOT NULL,
  wallet_type     TINYINT     NOT NULL,            -- 变更的钱包类型
  flow_type       TINYINT     NOT NULL,            -- 1=充值 2=提现 3=兑换 4=投注 5=返奖 6=人工加款 7=人工减款 8=冻结 9=解冻 10=确认扣除 ...
  direction       TINYINT     NOT NULL,            -- 1=入 2=出
  amount          BIGINT      NOT NULL,            -- 变更金额（正数）
  before_balance  BIGINT      NOT NULL,            -- 变更前余额
  after_balance   BIGINT      NOT NULL,            -- 变更后余额
  order_no        VARCHAR(32) NOT NULL,            -- 关联订单号（幂等键）
  biz_type        TINYINT     DEFAULT 0,           -- 业务子类型
  remark          VARCHAR(255) DEFAULT '',
  operator_id     BIGINT      DEFAULT 0,           -- 操作人（人工操作时有值）
  created_at      BIGINT      NOT NULL,

  UNIQUE KEY uk_order_no (order_no),               -- 幂等：同一orderNo只能有一条
  INDEX idx_uid_currency (uid, currency_code),
  INDEX idx_created_at (created_at)
);
```

#### 表4：t_wallet_freeze（冻结管理 — 状态机核心）

```sql
CREATE TABLE t_wallet_freeze (
  id              BIGINT      PRIMARY KEY AUTO_INCREMENT,
  uid             BIGINT      NOT NULL,
  currency_code   VARCHAR(10) NOT NULL,
  amount          BIGINT      NOT NULL,            -- 冻结金额
  order_no        VARCHAR(32) NOT NULL,            -- 关联提现订单号
  status          TINYINT     NOT NULL DEFAULT 1,  -- 1=冻结中 2=已确认扣除 3=已解冻退回
  freeze_at       BIGINT      NOT NULL,
  finish_at       BIGINT      DEFAULT 0,           -- 完结时间（确认/解冻时填入）
  created_at      BIGINT      NOT NULL,
  updated_at      BIGINT      NOT NULL,

  UNIQUE KEY uk_order_no (order_no),
  INDEX idx_uid_status (uid, status)
);

-- 状态机约束：
-- 1(冻结中) → 2(已确认扣除)  仅 ConfirmDebit 可触发
-- 1(冻结中) → 3(已解冻退回)  仅 UnfreezeBalance 可触发
-- 2 和 3 是终态，不可再变更
-- 同一 order_no，2 和 3 互斥，只能执行其一
```

#### 表5-8：业务订单表

```sql
-- t_deposit_order（充值订单）
-- 核心字段：order_no(C+16位), uid, currency_code, amount, pay_method,
--           channel_order_no, status(待支付/支付中/成功/失败/超时),
--           bonus_activity_id, exchange_rate_snapshot, pay_url, expired_at

-- t_withdraw_order（提现订单）
-- 核心字段：order_no(T+16位), uid, currency_code, amount, fee_amount,
--           withdraw_method, account_info(JSON), status(待审核/风控中/出款中/成功/驳回/失败),
--           reject_reason, freeze_record_id

-- t_withdraw_account（提现账户）
-- 核心字段：uid, currency_code, method_type, account_name(KYC姓名),
--           account_no(银行卡号/电子钱包号/USDT地址), bank_name, bank_branch

-- t_exchange_order（兑换订单）
-- 核心字段：order_no(E+16位), uid, from_currency, to_currency(BSB),
--           from_amount, exchange_rate, to_amount, bonus_rate, bonus_amount,
--           audit_multiple, status
```

### 4.3 设计要点

| 要点 | 决策 | 理由 |
|------|------|------|
| 金额类型 | BIGINT（i64整数，最小单位） | 避免浮点精度丢失，全链路一致 |
| 时间类型 | BIGINT（毫秒时间戳） | 跟随工程惯例 created_at/updated_at |
| 软删除 | deleted_at BIGINT DEFAULT 0 | 跟随工程惯例（GORM SoftDelete） |
| 幂等键 | order_no UNIQUE KEY | 核心防重复保障 |
| 钱包唯一 | (uid, currency_code, wallet_type) | 一个用户一个币种一个类型只有一个钱包 |
| 余额存储 | available + frozen 分开存 | 明确区分可用和冻结，无需计算 |

---

## 五、完整接口清单（50个）

### 5.1 数量速览

```
C端 HTTP ............. 16 个（核心13 + 应该有3）
B端 HTTP .............  9 个（核心7 + 应该有2）
RPC 对外暴露 ......... 13 个（核心10 + 应该有3）
RPC 调用依赖 ......... 12 个（可直接接入6 + 需Mock4 + 需确认2）
定时任务 .............  1 个
外部 HTTP ............  1 组（三方汇率API × 3家）

合计需要实现 ......... 39 个（C16 + B9 + RPC暴露13 + 定时1）
合计需要对接 ......... 12 个（RPC调用）+ 1组（HTTP调用）
```

### 5.2 C端接口（16个）

```
路由前缀：POST /api/wallet/...  全部需要用户登录Token鉴权

钱包核心：
  POST /api/wallet/home                    [核心]   钱包首页余额信息
  POST /api/wallet/currency/list           [核心]   可用币种列表

充值（4个）：
  POST /api/wallet/deposit/methods         [核心]   充值方式列表（→调财务接口）
  POST /api/wallet/deposit/create          [核心]   创建充值订单（→调财务创建通道订单）
  POST /api/wallet/deposit/check           [核心]   重复转账检测
  POST /api/wallet/deposit/status          [应该有] 充值订单状态查询

兑换（2个）：
  POST /api/wallet/exchange/rate           [核心]   兑换汇率查询/预览
  POST /api/wallet/exchange/do             [核心]   执行兑换（内部完成，无外部依赖）

提现（4个）：
  POST /api/wallet/withdraw/methods        [核心]   提现方式列表（→调KYC+财务接口）
  POST /api/wallet/withdraw/account        [核心]   获取已保存的提现账户
  POST /api/wallet/withdraw/account/save   [核心]   保存提现账户（→KYC姓名自动填充）
  POST /api/wallet/withdraw/create         [核心]   创建提现订单（→冻结+提交审核）

记录（2个）：
  POST /api/wallet/record/list             [核心]   交易记录列表（按Tab分类）
  POST /api/wallet/record/detail           [核心]   交易记录详情

其他：
  POST /api/wallet/bonus/list              [应该有] 可用奖金活动列表
  POST /api/wallet/withdraw/status         [应该有] 提现订单状态查询
```

### 5.3 B端接口（9个）

```
路由前缀：POST /admin/api/wallet/...  全部需要后台Token + RBAC权限
所有写操作须调 ser-blog.AddActLog() 记录审计日志

币种配置管理（5个）：
  POST /admin/api/wallet/currency/page     [核心]   币种列表分页查询
  POST /admin/api/wallet/currency/edit     [核心]   编辑币种
  POST /admin/api/wallet/currency/status   [核心]   启用/禁用币种
  POST /admin/api/wallet/currency/setBase  [核心]   设置基准币种（一次性）
  POST /admin/api/wallet/rate/log/page     [核心]   汇率日志分页查询

用户钱包管理（2个）：
  POST /admin/api/wallet/user/page         [核心]   B端用户钱包列表
  POST /admin/api/wallet/user/detail       [核心]   B端用户钱包详情

应该有：
  POST /admin/api/wallet/rate/log/export   [应该有] 汇率日志导出
  POST /admin/api/wallet/currency/detail   [应该有] 币种详情查询
```

### 5.4 RPC对外暴露接口（13个）

```
余额变更类（5个，核心）：
  CreditWallet      充值成功上账 → 中心钱包
  DebitWallet        消费/投注扣款（中心优先→奖励补足）
  FreezeBalance      提现申请冻结余额
  UnfreezeBalance    提现驳回/失败解冻退回
  ConfirmDebit       提现出款成功确认扣除

人工修正类（3个，核心）：
  ManualCredit       人工加款（A+单号，指定钱包类型）
  ManualDebit        人工减款（M+单号，余额从高到低级联扣减）
  SupplementCredit   充值补单（B+单号，含赠送→奖励钱包）

查询类（2个）：
  GetBalance         余额查询（高频，走Redis缓存）  [核心]
  BatchGetBalance    批量余额查询（≤100人）          [应该有]

返奖类（1个，核心）：
  RewardCredit       活动/任务返奖 → 奖励钱包

币种数据类（2个，应该有）：
  GetCurrencyConfig    币种配置查询（RPC版）
  GetExchangeRateRpc   汇率查询（RPC版，纯数据）
```

### 5.5 RPC调用依赖（12个）

```
可直接接入（IDL已定义，Client已注册，6个）：
  ① ser-blog.AddActLog          B端审计日志（oneway，异步不阻塞）
  ② ser-kyc.KycQuery            提现KYC状态校验
  ③ ser-user.GetUserInfoExpend   用户信息/KYC姓名
  ④ ser-user.BatchGetUserInfo    B端批量用户昵称
  ⑤ ser-s3.UploadIcon           币种图标上传
  ⑥ ser-buser.QueryUserByIds    B端操作人姓名回显

需Mock占位（服务名/IDL待定，4个）：
  ⑦ 财务.GetPaymentConfig       充值方式配置（★★★★★阻塞）
  ⑧ 财务.GetWithdrawConfig      提现方式配置（★★★★★阻塞）
  ⑨ 财务.CreatePayOrder         创建通道订单（★★★★★阻塞）
  ⑩ 活动.GetAvailableBonuses    可用奖金活动（★★低阻塞）

需确认接入方式（2个）：
  ⑪ ser-cron → 建议内置cron+分布式锁
  ⑫ 风控.RiskCheck → 当前版本不接入，预留方法占位
```

---

## 六、技术栈与工程基础设施

### 6.1 技术栈一览

| 层次 | 技术 | 版本 | 用途 |
|------|------|------|------|
| HTTP框架 | CloudWeGo Hertz | v0.9+ | C端/B端 HTTP 网关 |
| RPC框架 | CloudWeGo Kitex | v0.16.0 | 跨服务 RPC 通信 |
| IDL定义 | Apache Thrift | — | 接口契约定义+代码生成 |
| ORM | GORM + Gen | v2 | 数据库操作+代码生成 |
| 数据库 | TiDB（MySQL兼容） | — | 分布式SQL存储 |
| 缓存 | Redis | — | 余额缓存+分布式锁+汇率缓存 |
| 服务发现 | ETCD | — | 服务注册与发现 |
| 定时任务 | robfig/cron（内置） | — | 汇率定时刷新 |
| 日志 | 工程自带lg.Log() | — | 统一日志框架 |
| 国际化 | ser-i18n（Redis直读） | — | 错误码多语言翻译 |

### 6.2 工程基础设施变更（4处新增）

```
① common/pkg/consts/namesp/namesp.go
  + EtcdWalletService  = "wallet_service"
  + EtcdWalletPort     = "wallet_port"
  + EtcdWalletDb       = "ser_wallet"

② common/rpc/rpc_client.go
  + var walletClient serWallet.Client
  + var walletOnce sync.Once
  + func WalletClient() serWallet.Client { ... }

③ go.work
  + use ./ser-wallet

④ ETCD 配置中心
  + wallet_port = 8022（选取未使用的端口）
  + wallet_db = ser_wallet（TiDB数据库名）
```

### 6.3 工程目录结构

```
ser-wallet/
├── main.go                          # 服务入口（初始化顺序：cfg→DB→Redis→Cron→Server）
├── handler.go                       # 38个方法的 handler 入口（委派到 ser 层）
├── internal/
│   ├── cfg/
│   │   ├── mysql.go                 # GORM + TiDB 连接初始化
│   │   └── redis.go                 # Redis 连接初始化
│   ├── enum/
│   │   ├── wallet_type.go           # WalletType: Center=1, Reward=2, Anchor=3, Agent=4
│   │   ├── currency_type.go         # CurrencyType: Fiat=1, Crypto=2
│   │   ├── flow_type.go             # FlowType: Deposit=1, Withdraw=2, Exchange=3, ...
│   │   ├── order_status.go          # OrderStatus: Pending=1, Processing=2, Success=3, ...
│   │   └── log_enum.go              # 审计日志常量
│   ├── errs/
│   │   └── wallet_errors.go         # 错误码定义（60XX范围）
│   ├── ser/
│   │   ├── currency_service.go      # 币种配置业务逻辑
│   │   ├── wallet_core.go           # 6个原子操作（最核心文件）
│   │   ├── wallet_service.go        # 钱包首页/余额查询
│   │   ├── deposit_service.go       # 充值业务逻辑
│   │   ├── withdraw_service.go      # 提现业务逻辑
│   │   ├── exchange_service.go      # 兑换业务逻辑
│   │   ├── record_service.go        # 记录查询
│   │   ├── wallet_rpc_service.go    # RPC对外暴露接口的service层
│   │   ├── log_helper.go            # 审计日志统一封装
│   │   ├── user_helper.go           # 用户信息查询封装
│   │   └── kyc_helper.go            # KYC校验封装
│   ├── rep/                         # Repository层（扩展 gorm_gen 生成的基础操作）
│   ├── cache/
│   │   ├── balance_cache.go         # 余额缓存（30s TTL）
│   │   └── currency_cache.go        # 币种配置缓存 + 汇率缓存
│   └── gen/
│       └── gorm_gen.go              # GORM代码生成入口
├── common/idl/ser-wallet/           # 8个 Thrift IDL 文件
│   ├── service.thrift               # 38个方法声明
│   ├── wallet.thrift                # 钱包核心类型
│   ├── currency.thrift              # 币种配置类型
│   ├── deposit.thrift               # 充值类型
│   ├── withdraw.thrift              # 提现类型
│   ├── exchange.thrift              # 兑换类型
│   ├── record.thrift                # 记录类型
│   └── wallet_rpc.thrift            # RPC对外类型
└── SQL/
    └── init.sql                     # DDL建表语句

文件统计：
  手写核心文件    ~42 个
  自动生成文件    ~55 个（kitex_gen + gorm_gen）
  修改已有文件     5 个（namesp/rpc_client/go.work/register×2）
  合计           ~102 个
```

### 6.4 错误码体系

```
ser-wallet 使用 60XX 段（XX 为分配的服务编号）：

60XX000 ~ 60XX099  币种配置错误
  001 CurrencyNotFound      币种不存在
  002 CurrencyDisabled      币种已禁用
  003 BaseCurrencyAlreadySet 基准币种已设置
  004 BaseCurrencyCannotDisable 基准币种不可禁用

60XX100 ~ 60XX199  钱包核心错误
  101 WalletNotFound        钱包不存在
  102 InsufficientBalance   余额不足
  103 DuplicateOrder        重复操作（幂等拦截）
  104 FreezeRecordNotFound  冻结记录不存在
  105 FreezeStatusConflict  冻结状态冲突（已解冻/已扣除）

60XX200 ~ 60XX299  充值错误
  201 DepositAmountInvalid  充值金额不合法
  202 DuplicateDeposit      存在重复待支付订单
  203 PaymentConfigUnavail  充值方式获取失败

60XX300 ~ 60XX399  提现错误
  301 KycNotPassed          请先完成KYC认证
  302 WithdrawLimitExceeded 超出提现限额
  303 WithdrawDailyExceeded 超出单日提现次数
  304 WithdrawAccountInvalid 提现账户信息不完整
  305 KycServiceUnavailable KYC服务异常

60XX400 ~ 60XX499  兑换错误
  401 ExchangeRateExpired   汇率已过期
  402 ExchangeAmountInvalid 兑换金额不合法
```

---

## 七、核心技术挑战与解决方案

### 7.1 挑战一：并发余额操作的数据一致性

```
问题场景：
  同一用户同时发起投注扣款 + 充值入账 + 活动返奖，三个操作并发修改同一行余额
  如果用"先读后写"模式 → 数据竞争 → 余额错乱

解决方案：双保险机制

  第一道防线 — Redis分布式锁：
    锁Key：wallet:lock:{uid}:{currencyCode}
    等待时间：20秒
    持有时间：30秒
    → 将并发串行化，减少冲突概率

  第二道防线 — SQL原子UPDATE + WHERE条件：
    UPDATE t_user_wallet
    SET available_balance = available_balance - ?
    WHERE uid = ? AND currency_code = ? AND wallet_type = ?
      AND available_balance >= ?    -- ← 关键！数据库级兜底
    → 即使分布式锁失效，SQL层也能保证不会透支

  参考来源：模块参考A（Java p9钱包 Redisson + SQL WHERE 双保险模式）

为什么不只用一种？
  只用锁：锁失效（宕机/超时释放）时无兜底
  只用SQL：高并发下大量失败重试，性能差
  双保险：锁减少冲突频率 + SQL兜底一致性 = 最优解
```

### 7.2 挑战二：幂等性保障（防止重复入账/扣款）

```
问题场景：
  财务模块调CreditWallet给用户充值入账，网络超时 → 财务侧不知道是否成功 → 重试
  如果没有幂等 → 用户被入账两次

解决方案：orderNo唯一键机制

  1. 所有余额变更RPC的Req必须包含 orderNo 字段
  2. wallet_flow 表的 order_no 设为 UNIQUE KEY
  3. 处理流程：
     → 收到请求 → 查 wallet_flow 是否已有该 orderNo
       → 已存在且成功 → 直接返回成功（幂等响应），不重复操作
       → 已存在且失败 → 返回上次的失败信息
       → 不存在 → 正常处理 → 写入 wallet_flow
  4. 由于 UNIQUE KEY 约束，即使并发重复请求也只有一个能插入成功

订单号规则：
  充值：C + 16位（如 C2026022800000001）
  提现：T + 16位
  兑换：E + 16位
  人工加款：A + 16位
  人工减款：M + 16位
  补单：B + 16位
  外部模块：由调用方自行保证全局唯一
```

### 7.3 挑战三：冻结状态机的一致性

```
问题场景：
  用户提现申请 → 冻结1000元 → 财务审核驳回（应解冻）
  但同时出款通道误发成功回调（应确认扣除）
  → ConfirmDebit 和 UnfreezeBalance 不能同时执行

解决方案：状态机 + 数据库级互斥

  状态机：
    冻结中(1) → 已确认扣除(2)   仅 ConfirmDebit 可触发
    冻结中(1) → 已解冻退回(3)   仅 UnfreezeBalance 可触发
    2 和 3 是终态

  实现：
    UPDATE t_wallet_freeze
    SET status = 2, finish_at = ?    -- 或 status = 3
    WHERE order_no = ? AND status = 1  -- ← 关键：只有状态=1才能变更
    → affected rows = 0 → 说明已被另一个操作消费 → 返回状态冲突错误

  为什么不用程序逻辑判断？
    → "先读status再判断再写" 在并发下有时间窗口
    → SQL WHERE status=1 是原子的，数据库保证
```

### 7.4 挑战四：汇率系统的正确性与可靠性

```
问题场景：
  汇率刷新时，3家API返回不同汇率，其中1家可能异常
  刷新过程中有用户正在兑换，汇率不能突变

解决方案：

  1. 多源取平均 + 异常过滤：
     → 并行调3家API，超时5秒/家
     → 至少2家成功才视为有效
     → 全失败则保持当前汇率不变 + 触发告警

  2. 偏差阈值触发机制：
     → |实时汇率 - 平台汇率| / 实时汇率 × 100% = 偏差%
     → 偏差 ≥ 配置阈值% → 更新平台汇率（同时更新入款/出款汇率）
     → 偏差 < 阈值% → 仅刷新显示用的实时汇率，不变更平台汇率
     → 避免微小波动频繁更新

  3. 汇率快照机制：
     → 用户进入兑换/充值页面时锁定当前汇率
     → 提交订单时校验汇率是否过期（如5分钟内有效）
     → 过期则要求用户刷新页面重新获取

  4. 入款/出款汇率公式：
     → 入款汇率 = 平台汇率 × (1 + 浮动%)  → 用户充值时用（平台多收一点）
     → 出款汇率 = 平台汇率 × (1 - 浮动%)  → 用户提现时用（平台少付一点）

  5. 防多实例重复执行：
     → 内置cron + Redis分布式锁
     → 锁Key：wallet:cron:rate_refresh
     → 某一实例获锁后执行，其他实例跳过
```

### 7.5 挑战五：跨服务事务（充值/提现链路）

```
问题场景：
  充值链路涉及 钱包 + 财务 + 三方通道 三个系统
  提现链路涉及 钱包 + 财务 + 风控 + 三方通道 四个系统
  无法用数据库事务保证跨服务一致性

解决方案：最终一致性 + 补偿机制

  充值链路：
    → 用户发起 → 钱包创建本地订单(status=待支付)
    → 钱包调财务创建通道订单 → 获取支付URL
    → 用户跳转支付 → 三方通道回调财务
    → 财务确认后调 CreditWallet → 钱包上账
    → 关键保障：CreditWallet 幂等，财务可无限重试

  提现链路：
    → 用户发起 → 钱包冻结余额(FreezeBalance) → 创建订单(status=待审核)
    → 提交财务审核流程
    → 审核通过 → 财务出款 → 成功 → 调 ConfirmDebit → 扣除冻结
    → 审核驳回 / 出款失败 → 调 UnfreezeBalance → 退回冻结
    → 关键保障：冻结状态机互斥 + 各RPC幂等

  兑换链路：
    → 全部在钱包内部完成 → 同库同事务 → 无跨服务问题
    → 扣法币 + 加BSB + 加赠送 + 写流水 → 一个DB事务搞定
```

### 7.6 挑战六：金额精度（全链路整数化）

```
问题：浮点数做金融计算会丢精度
  例：0.1 + 0.2 = 0.30000000000000004（IEEE 754）

解决方案：全链路 i64 整数（最小单位）

  存储：BIGINT（数据库）
  传输：i64（Thrift IDL）
  计算：整数运算（Go int64）
  展示：前端根据币种精度配置做格式化

  转换规则（以BRL为例，精度2位）：
    用户看到：R$ 1,234.56
    系统存储：123456（整数，单位=分）
    API传输：amount = 123456, decimalPlaces = 2

  汇率存储：
    汇率本身用 float64 存储（仅用于乘法计算）
    计算结果立即转回 i64
    或者汇率也用整数表示（如 1 USDT = 5123400 BRL分 → rate = 5123400）

  参考：模块参考A（Java p9钱包 amount 字段用 BigDecimal，我们用 i64 更简洁）
```

---

## 八、三大业务流程详解

### 8.1 充值流程（最长链路，涉及多方）

```
┌──────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐     ┌──────────┐
│ 用户  │     │ ser-wallet │     │ 财务模块   │     │ 三方通道   │     │ 区块链    │
└──┬───┘     └─────┬─────┘     └─────┬─────┘     └─────┬─────┘     └────┬─────┘
   │               │                 │                 │                │
   │ ① 打开充值页面  │                 │                 │                │
   │──────────────→│                 │                 │                │
   │               │ ② 调财务获取充值方式                │                │
   │               │────────────────→│                 │                │
   │               │ ③ 返回方式列表   │                 │                │
   │ ④ 展示方式列表 │←────────────────│                 │                │
   │←──────────────│                 │                 │                │
   │               │                 │                 │                │
   │ ⑤ 选择方式+金额│                 │                 │                │
   │──────────────→│                 │                 │                │
   │               │ ⑥ 重复转账检测    │                 │                │
   │               │ ⑦ 创建本地订单    │                 │                │
   │               │ ⑧ 调财务创建通道单 │                 │                │
   │               │────────────────→│                 │                │
   │               │ ⑨ 返回支付URL     │                 │                │
   │ ⑩ 跳转支付页面 │←────────────────│                 │                │
   │←──────────────│                 │                 │                │
   │               │                 │                 │                │
   │ ⑪ 用户支付     │                 │                 │                │
   │─────────────────────────────────────────────────→│                │
   │               │                 │ ⑫ 通道回调结果   │                │
   │               │                 │←────────────────│                │
   │               │ ⑬ CreditWallet   │                 │                │
   │               │←────────────────│                 │                │
   │               │ ⑭ 用户钱包余额+= │                 │                │
   │ ⑮ 充值成功     │                 │                 │                │
   │←──────────────│                 │                 │                │

USDT充值特殊流程（替代⑪-⑫）：
   │ ⑪' 用户转USDT到平台地址                                            │
   │───────────────────────────────────────────────────────────────────→│
   │               │                 │                 │  ⑫' 链上确认   │
   │               │                 │←──────────────────────────────────│
   │               │ ⑬' CreditWallet  │                 │                │
   │               │←────────────────│                 │                │
```

### 8.2 提现流程（最多前置条件，审核链最长）

```
┌──────┐     ┌───────────┐     ┌────────┐     ┌────────┐     ┌───────────┐
│ 用户  │     │ ser-wallet │     │ ser-kyc │     │ 财务审核 │     │ 三方通道   │
└──┬───┘     └─────┬─────┘     └───┬────┘     └───┬────┘     └─────┬─────┘
   │               │               │               │               │
   │ ① 打开提现页面  │               │               │               │
   │──────────────→│               │               │               │
   │               │ ② KYC校验     │               │               │
   │               │──────────────→│               │               │
   │               │ ③ status=3?    │               │               │
   │               │←──────────────│               │               │
   │               │               │               │               │
   │               │ status≠3 → 返回"请先完成KYC"    │               │
   │               │ status=3 → 继续                │               │
   │               │               │               │               │
   │               │ ④ 调财务获取提现方式            │               │
   │ ⑤ 展示方式列表 │               │               │               │
   │←──────────────│               │               │               │
   │               │               │               │               │
   │ ⑥ 选择方式+填写│               │               │               │
   │  账户信息+金额 │               │               │               │
   │──────────────→│               │               │               │
   │               │ ⑦ 二次KYC校验  │               │               │
   │               │ ⑧ 校验限额（单笔/日/次数）      │               │
   │               │ ⑨ 校验余额充足  │               │               │
   │               │ ⑩ FreezeBalance（冻结）         │               │
   │               │ ⑪ 创建提现订单  │               │               │
   │               │ ⑫ 提交财务审核  │               │               │
   │               │────────────────────────────────→│               │
   │ ⑬ 提现已提交   │               │               │               │
   │←──────────────│               │               │               │
   │               │               │               │               │
   │               │               │    ⑭ 审核通过   │               │
   │               │               │               │──────────────→│
   │               │               │               │  ⑮ 出款结果    │
   │               │               │               │←──────────────│
   │               │               │               │               │
   │               │ ⑯a ConfirmDebit│（出款成功）     │               │
   │               │←───────────────────────────────│               │
   │               │ ⑯b UnfreezeBalance（驳回/失败）  │               │
   │               │←───────────────────────────────│               │

提现前置条件（全部必须通过才能提现）：
  ✓ KYC 认证通过（status=3）
  ✓ 提现方式可用（财务模块配置）
  ✓ 提现账户已保存
  ✓ 单笔限额校验通过
  ✓ 单日限额校验通过
  ✓ 单日次数校验通过
  ✓ 可用余额 ≥ 提现金额
```

### 8.3 兑换流程（最简单，全部内部完成）

```
┌──────┐     ┌───────────┐
│ 用户  │     │ ser-wallet │
└──┬───┘     └─────┬─────┘
   │               │
   │ ① 打开兑换页面  │
   │──────────────→│
   │               │ 查询平台汇率 + 赠送比例
   │ ② 展示汇率预览 │
   │←──────────────│
   │               │
   │ ③ 输入法币金额  │
   │──────────────→│
   │               │ 计算：BSB数量 = 法币金额 × 汇率
   │               │       赠送数量 = BSB数量 × 赠送比例
   │               │       流水要求 = 赠送数量 × 流水倍数
   │ ④ 展示到账预览  │
   │←──────────────│
   │               │
   │ ⑤ 确认兑换     │
   │──────────────→│
   │               │ 同一事务内执行：
   │               │   1. 锁定汇率（5分钟有效期内）
   │               │   2. 扣法币中心钱包余额
   │               │   3. 加BSB中心钱包余额
   │               │   4. 加BSB奖励钱包余额（赠送部分）
   │               │   5. 写稽核流水（流水倍数要求）
   │               │   6. 写交易流水
   │               │   7. 创建兑换订单
   │ ⑥ 兑换成功     │
   │←──────────────│

优势：无外部依赖，全在钱包库内事务完成，开发和测试最简单
```

---

## 九、外部依赖与协作方

### 9.1 我调谁（出方向）

```
┌──────────────────────────────────────────────────────────────┐
│                      ser-wallet                              │
│                                                              │
│  直接接入（6个）         Mock占位（4个）         确认/预留（2个）│
│  ├── ser-blog           ├── 财务.GetPayConfig   ├── ser-cron  │
│  ├── ser-kyc            ├── 财务.GetWithdraw    └── 风控模块   │
│  ├── ser-user ×2        ├── 财务.CreatePayOrder               │
│  ├── ser-s3             └── 活动.GetBonuses                   │
│  └── ser-buser                                               │
│                                                              │
│  非RPC依赖：                                                  │
│  ├── Redis（缓存+锁）                                         │
│  ├── ETCD（服务发现）                                          │
│  ├── ser-i18n（Redis直读，非RPC）                              │
│  └── 三方汇率API ×3（HTTP调用）                                │
└──────────────────────────────────────────────────────────────┘
```

### 9.2 谁调我（入方向）

```
┌──────────────────────────────────────────────────────────────┐
│  调用方          │ 调用接口                      │ 频率       │
├──────────────────┼──────────────────────────────┼───────────┤
│  财务模块（主要） │ CreditWallet                  │ 低频       │
│                  │ ConfirmDebit                  │ 低频       │
│                  │ UnfreezeBalance               │ 低频       │
│                  │ ManualCredit                  │ 极低频     │
│                  │ ManualDebit                   │ 极低频     │
│                  │ SupplementCredit              │ 极低频     │
│                  │ GetBalance                    │ 中频       │
│                  │ GetCurrencyConfig             │ 低频       │
│                  │ GetExchangeRateRpc            │ 中频       │
├──────────────────┼──────────────────────────────┼───────────┤
│  投注模块        │ DebitWallet                   │ 高频       │
│                  │ GetBalance                    │ 高频       │
├──────────────────┼──────────────────────────────┼───────────┤
│  活动模块        │ RewardCredit                  │ 中频       │
│                  │ GetBalance                    │ 中频       │
├──────────────────┼──────────────────────────────┼───────────┤
│  直播模块        │ GetBalance                    │ 中频       │
└──────────────────┴──────────────────────────────┴───────────┘

唯一的双向依赖：wallet ←→ 财务模块
  → 我们调财务：获取充值方式、创建通道订单
  → 财务调我们：上账、冻结确认/解冻、人工修正
  → 这是需要最优先对齐接口契约的协作方
```

---

## 十、开放问题清单（需产品确认）

### 10.1 必须确认（影响数据模型/核心逻辑，6个）

| 编号 | 问题 | 影响范围 | 建议方案 |
|------|------|---------|---------|
| Q1 | 奖励钱包稽核流水规则？奖励金额需完成X倍投注后才能转入中心钱包——这个X怎么配置？在哪配？ | t_audit_flow表设计、兑换/返奖接口参数 | 建议：兑换赠送的倍数跟随币种配置，活动返奖的倍数由活动模块传入 |
| Q2 | 混合投注（中心+奖励余额混合扣款）的返奖比例怎么算？例：投100（中心80+奖励20），赢200，怎么分？ | DebitWallet返回值、CreditWallet入账逻辑 | 建议：按扣款比例返还（中心得160、奖励得40），DebitWallet返回比例明细 |
| Q5 | 充值订单数据归谁管？钱包本地建表，还是财务模块建表？ | deposit_order表是否在钱包库、C13记录查询的数据来源 | 倾向归钱包（记录查询本地读更高效），但需财务方确认 |
| Q9 | "充什么提什么"的判定逻辑？用户充BRL只能提BRL？还是充BRL兑换成BSB后也能提BSB？ | 提现校验规则 | 需产品明确。当前理解：提现从中心钱包扣，不限充值来源 |
| Q13 | 兑换赠送比例（如法币→BSB送10%）和流水倍数，在哪配置？ | EditCurrency字段、DoExchange计算逻辑 | 倾向归币种配置（与汇率同级别的基础数据），作为currency_config的字段 |
| Q15 | 入款/出款汇率的基数是实时汇率还是平台汇率？即浮动%应用在哪个基数上？ | 汇率计算公式正确性 | 建议：基于平台汇率（更稳定），即 入款汇率 = 平台汇率 × (1+浮动%) |

### 10.2 应该确认（影响产品体验/功能完整度，5个）

| 编号 | 问题 | 影响范围 |
|------|------|---------|
| Q3 | 主播钱包/代理钱包当前版本是否实现？ | 如实现则RPC暴露接口增加3-4个 |
| Q4 | 币种是预置列表还是运营手动新增？ | 是否需要CreateCurrency接口 |
| Q6 | 提现账户保存后能否自行修改/删除？ | 是否需要编辑/删除提现账户接口 |
| Q10 | 提现手续费规则：固定+比例？由谁配置？ | 提现创建订单时的费用计算 |
| Q12 | USDT充值时是选择链类型还是只支持TRC20？ | 充值页面交互+地址管理 |

### 10.3 低优先确认（不影响首版开发，5个）

| 编号 | 问题 |
|------|------|
| Q7 | 连续汇率刷新失败是否需要自动预警？ |
| Q8 | B端汇率日志是否需要导出功能？ |
| Q11 | 是否支持管理员手动刷新汇率（而非只能等定时任务）？ |
| Q14 | 充值/提现是否需要全局开关（运维层面暂停功能）？ |
| Q16 | 提现订单超时多久自动关闭并解冻？ |

---

## 十一、风险识别与应对

### 11.1 高风险项（🔴）

```
R1："充什么提什么"规则可能锁死用户

  场景：用户充BRL → 兑换成BSB → 想提BRL → 被"充什么提什么"规则阻断
  影响：用户投诉，合规风险
  应对：评审会上确认此规则的准确边界。建议：提现从中心钱包扣，不限充值币种来源

R2：财务模块 IDL 未定义（最大阻塞项）

  现状：财务模块的服务名、IDL、接口契约完全空白
  影响：充值方式列表(C3)、创建充值(C4)、提现方式(C9)、创建提现(C12) 四个核心接口无法真正跑通
  应对：Mock接口开发，定义 FinanceClient interface 隔离依赖；评审会上推动财务模块尽快出 IDL
```

### 11.2 中风险项（🟡）

```
R3：人工减款级联扣减可能扣错钱包
  → 应对：返回明细让财务B端展示确认

R4：汇率刷新期间并发兑换导致使用过渡期汇率
  → 应对：用户下单时快照锁定汇率，5分钟有效期

R5：禁用币种时仍有未完成的充值/提现订单
  → 应对：禁用前检查在途订单，有则提示"仍有N笔进行中订单"

R6：KYC服务不可用时的提现降级策略
  → 应对：绝不默认放行，返回"KYC服务异常请稍后重试"

R7：三方汇率API全部故障
  → 应对：保持当前汇率不变 + 触发告警，连续N次失败升级告警
```

### 11.3 低风险项（🟢）

```
R8：USDT充值地址安全管理 → 由财务/链上模块负责，不在钱包职责内
R9：时区处理 → UTC存储，前端展示转换，每日限额按UTC 0点重置
R10：大文件上传 → 币种图标限制5KB SVG/PNG，不存在大文件问题
```

---

## 十二、开发节奏与里程碑

### 12.1 四批迭代交付

```
P0 — 骨架阶段（纯内部逻辑，不依赖外部服务）
─────────────────────────────────────────────────
  B端：币种列表/编辑/启禁用/设基准/汇率日志（5个）
  C端：钱包首页/币种列表/汇率查询/执行兑换/记录列表/记录详情（6个）
  RPC暴露：CreditWallet/DebitWallet/FreezeBalance/UnfreezeBalance/ConfirmDebit/GetBalance（6个）
  定时任务：汇率刷新（先Mock汇率API）
  可验证：币种CRUD + 余额操作 + 兑换 + 记录查询全部可独立测试

P1 — 接入已有IDL的外部服务
─────────────────────────────────────────────────
  C端：提现方式/提现账户获取/保存/创建提现（4个）→ 依赖ser-kyc/ser-user
  B端：用户钱包列表/详情（2个）→ 依赖ser-user/ser-buser
  RPC暴露：ManualCredit/ManualDebit/SupplementCredit/RewardCredit（4个）
  可验证：提现链路（含KYC校验）+ B端管理功能

P2 — Mock财务模块，跑通充提主链路
─────────────────────────────────────────────────
  C端：充值方式/创建充值/重复检测（3个）→ Mock财务接口
  C端应该有：充值状态/奖金活动/提现状态（3个）
  RPC暴露应该有：BatchGetBalance/GetCurrencyConfig/GetExchangeRateRpc（3个）
  可验证：充值+提现完整链路（用Mock数据跑通）

P3 — 联调替换Mock
─────────────────────────────────────────────────
  替换 MockFinanceClient → RealFinanceClient
  替换 Mock汇率API → 真实三方API
  联调充值/提现完整链路
  性能优化 + 缓存调优
```

### 12.2 里程碑检查点

```
✅ M1：IDL定义完成 + gen_kitex.sh 生成成功 + 服务能启动
✅ M2：币种配置CRUD可通过B端操作 + 汇率日志写入正常
✅ M3：6个原子操作全部通过单元测试（含幂等+并发）
✅ M4：兑换流程端到端跑通（C端发起→余额变更→流水记录）
✅ M5：提现链路跑通（KYC校验→冻结→订单创建）
✅ M6：充值链路跑通（Mock财务→订单创建→CreditWallet入账）
✅ M7：财务模块联调成功（真实RPC替换Mock）
✅ M8：全量接口联调 + 性能测试通过
```

---

## 十三、评审会Q&A预演

### 13.1 架构设计类

**Q：为什么用整数存金额而不用DECIMAL？**
> 全链路 i64 整数（最小单位），避免 IEEE 754 浮点精度丢失。例如 BRL 精度2位，1234.56 存为 123456。Thrift IDL 原生支持 i64，传输无精度损失。参考了 Java p9 钱包使用 BigDecimal 的方案，Go 用 int64 更简洁且性能更好。

**Q：并发扣款怎么保证不超扣？**
> 双保险机制：Redis 分布式锁（减少冲突频率）+ SQL 原子 UPDATE WHERE balance >= ?（数据库级兜底）。即使锁失效，SQL 条件也能保证不透支。这个模式来自 Java p9 钱包的 Redisson + SQL WHERE 双保险实践。

**Q：幂等怎么做的？**
> 所有余额变更 RPC 入参包含 orderNo 字段，wallet_flow 表 order_no 设为 UNIQUE KEY。处理前查是否已存在——已存在直接返回上次结果，不重复操作。调用方可安全重试。

**Q：冻结/解冻/确认扣除三个操作的关系？**
> 这是一个严格的状态机：FreezeBalance 把钱从可用余额移到冻结余额；之后只能二选一——ConfirmDebit（出款成功，永久扣除）或 UnfreezeBalance（驳回/失败，退回可用）。两者互斥，通过 SQL WHERE status=1 原子保证不会同时执行。

**Q：为什么不用消息队列？**
> 工程内没有消息队列基础设施（无 Kafka/RabbitMQ/RocketMQ），项目统一用 RPC 同步调用。如果后续有需求再引入，但当前 RPC + 幂等 + 重试已能满足最终一致性要求。

### 13.2 业务逻辑类

**Q：钱包有几种类型？**
> 4种：中心钱包（主钱包，充值/消费/提现都走这里）、奖励钱包（活动奖励/兑换赠送，需完成稽核流水才能转出）、主播钱包（预留，当前版本不实现）、代理钱包（预留，当前版本不实现）。

**Q：消费时先扣哪个钱包？**
> 中心钱包优先，中心余额不够时从奖励钱包补足。扣款结果包含各钱包实际扣减明细，返回给调用方（投注模块需要这个比例来计算返奖）。

**Q：兑换为什么不需要调外部服务？**
> 兑换是法币→BSB单向转换，汇率数据在钱包本地（由定时任务维护），扣减和增加的都是同一用户不同币种的余额——全部在钱包库内同一事务完成。这是三条业务线中最简单的。

**Q：充值回调是钱包处理还是财务处理？**
> 财务处理。三方通道回调涉及验签、通道匹配、订单状态更新，这些是财务模块的职责。财务确认后调用 ser-wallet.CreditWallet 给用户上账。钱包只负责"收到指令→加钱"。

**Q：BSB是什么？和USDT什么关系？**
> BSB是平台积分币，锚定关系 10 BSB = 1 USDT（固定比例）。用户可以用法币兑换BSB（享受赠送奖励），BSB用于平台内消费。BSB不能直接提现，需在平台内使用。

### 13.3 外部依赖类

**Q：钱包依赖哪些外部服务？**
> 可直接接入6个（ser-blog/ser-kyc/ser-user×2/ser-s3/ser-buser，IDL已有）；需Mock占位4个（财务×3/活动×1，服务名未定）；需确认2个（ser-cron/风控）。最大阻塞项是财务模块——充值方式配置、创建通道订单、提现方式配置三个接口的IDL和服务名完全空白。

**Q：财务模块没好怎么办？**
> 定义 FinanceClient interface 做依赖隔离，先实现 MockFinanceClient 返回硬编码测试数据。财务模块 IDL 确定后实现 RealFinanceClient 替换 Mock。两者实现同一接口，通过配置切换。

**Q：汇率数据从哪来？**
> 定时任务并行调用3家三方汇率API（具体选型需确认，常见有 ExchangeRate-API/Open Exchange Rates/CurrencyLayer），取平均值。至少2家成功才视为有效，全失败保持当前汇率不变并触发告警。

### 13.4 风险与边界类

**Q：这个模块最大的风险是什么？**
> 两个：①财务模块IDL未定义，充值/提现主链路的外部依赖无法真正对接，需尽快推动财务方出IDL；②"充什么提什么"规则的边界不清晰，可能导致用户充BRL→兑换BSB后无法提现BRL的死锁场景。这两个都需要在评审会上确认。

**Q：如果KYC服务挂了怎么办？**
> 提现相关接口（方式列表/账户保存/创建订单）全部返回"KYC服务异常请稍后重试"，绝不默认放行。这是金融安全红线。充值和兑换不受影响（不依赖KYC）。

**Q：如果Redis挂了影响什么？**
> 余额缓存失效→直接读数据库（性能下降但功能不受影响）；分布式锁失效→退化为只依赖SQL乐观锁（性能下降但一致性不受影响）。Redis不是单点故障。

**Q：当前版本不做什么？**
> 主播钱包/代理钱包功能（预留walletType字段）、投注专用扣款接口（通用DebitWallet已覆盖核心需求）、风控模块接入（预留riskCheck方法占位）、充提全局开关（后续评估）。

---

## 十四、一页纸速览

```
┌─────────────────────────────────────────────────────────────────┐
│                    ser-wallet 钱包模块                            │
│                                                                 │
│  定位：平台资金中枢，所有涉及用户钱的操作最终经过钱包              │
│                                                                 │
│  技术栈：Go + Kitex(RPC) + Hertz(HTTP) + GORM + TiDB + Redis    │
│         + ETCD(服务发现) + Thrift IDL                            │
│                                                                 │
│  四子模块：币种配置(底层) → 钱包核心(原子操作) → 交易业务 → 记录   │
│                                                                 │
│  核心数字：                                                      │
│    14 张数据表（8必须 + 3应该 + 3预留）                           │
│    50 个接口（C端16 + B端9 + RPC暴露13 + RPC调用12）              │
│    38 个IDL方法声明                                              │
│    ~102 个工程文件（42手写 + 55自动生成 + 5修改）                  │
│                                                                 │
│  三条业务线：                                                     │
│    充值（最长链路）：用户→钱包→财务→通道→回调→上账                │
│    提现（最多前置）：KYC→限额→冻结→审核→出款→确认/退回            │
│    兑换（最简单）  ：法币→BSB，全在钱包内部事务完成                │
│                                                                 │
│  核心技术方案：                                                   │
│    余额一致性 = Redis分布式锁 + SQL原子UPDATE双保险               │
│    防重复操作 = orderNo唯一键幂等机制                             │
│    冻结安全   = 状态机 + SQL WHERE status=1 原子互斥              │
│    金额精度   = 全链路i64整数（最小单位）                          │
│    汇率可靠   = 3家API取平均 + 偏差阈值 + 异常兜底                │
│                                                                 │
│  最大风险：                                                      │
│    ① 财务模块IDL未定义（充提主链路阻塞）                          │
│    ② "充什么提什么"规则边界不清                                   │
│                                                                 │
│  开发节奏：P0(骨架) → P1(外部服务) → P2(Mock财务) → P3(联调)     │
│                                                                 │
│  评审会需确认：6个必答题 + 5个应确认 + 5个低优先                   │
└─────────────────────────────────────────────────────────────────┘
```

---

> 本文档整合自20+份分析文档，共计30000+行源文件的深度梳理。每个结论均有需求文档条目、产品原型截图编号、工程源码验证、或模块参考实证作为支撑。
