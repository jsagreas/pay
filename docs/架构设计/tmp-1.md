# ser-wallet 架构设计方案

> 设计时间：2026-02-26
> 设计原则：安全、高并发、高性能、不过度设计、业界主流
> 工程基础：Go Workspace + Kitex RPC + Hertz HTTP + GORM Gen + TiDB + Redis + ETCD + NATS/Kafka
> 职责范围：我负责币种配置 + 多币种钱包，配合财务模块联调
> 信息源：
> - 工程代码级分析（common/gate-back/gate-font/ser-app 四模块）
> - 需求理解 + 接口评估 + 依赖关系 + 链路关系 + 流程思路 + 实现思路

---

## 目录

1. [架构设计总纲](#一架构设计总纲)
2. [系统拓扑架构](#二系统拓扑架构)
3. [账户模型架构（核心中的核心）](#三账户模型架构)
4. [记账模型架构](#四记账模型架构)
5. [并发控制架构](#五并发控制架构)
6. [订单状态机架构](#六订单状态机架构)
7. [幂等性架构](#七幂等性架构)
8. [跨模块通信架构](#八跨模块通信架构)
9. [缓存架构](#九缓存架构)
10. [资金安全架构](#十资金安全架构)
11. [数据库架构](#十一数据库架构)
12. [服务内部分层架构](#十二服务内部分层架构)
13. [部署与可观测性](#十三部署与可观测性)
14. [架构决策记录（ADR）](#十四架构决策记录)

---

## 一、架构设计总纲

### 1.1 设计约束

```
硬约束（不可选择的）：
├── 现有工程栈：Go Workspace + Kitex + Hertz + GORM Gen + TiDB + Redis + ETCD
├── 现有通信模式：网关(Hertz HTTP) → 服务(Kitex RPC)，ETCD 服务发现
├── 现有运维模式：ETCD 配置中心，无本地配置文件
├── 现有审计要求：所有写操作 → ser-blog.AddActLog (oneway)
└── 现有 MQ 基础设施：Kafka + NATS JetStream 双可用

设计原则：
├── 安全第一：资金操作零容忍数据不一致
├── 不过度设计：不引入当前规模不需要的组件
├── 复用现有：在现有工程模式上扩展，不另起炉灶
└── 可演进：关键位置留扩展口，但不提前实现
```

### 1.2 这套系统的本质

从架构角度看，钱包 + 币种配置 + 财务 这三个模块本质上构成一个**支付账户体系**：

```
币种配置 = 计价引擎（告诉系统"钱怎么算"）
钱包     = 账户引擎（管理"用户有多少钱、钱怎么动"）
财务     = 交易引擎（驱动"钱从外部进出系统"）
```

业界对这类系统有成熟的架构模式，核心是三件事：
1. **账户模型**——钱存在哪里、怎么组织
2. **记账模型**——每笔资金变动怎么记录、怎么对账
3. **并发控制**——多个操作同时改同一个账户时怎么保证正确

下面逐一展开。

---

## 二、系统拓扑架构

### 2.1 整体拓扑

```
                        ┌──────────────┐
                        │   C端用户     │
                        └──────┬───────┘
                               │ HTTPS
                        ┌──────▼───────┐
                        │  gate-font   │  C端网关(Hertz)
                        │  黑名单+鉴权  │  FontAuthMiddleware
                        └──────┬───────┘
                               │ RPC (Kitex)
            ┌──────────────────┼──────────────────┐
            │                  │                  │
     ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐
     │  ser-wallet  │  │  ser-kyc     │  │  ser-user    │
     │  钱包+币种    │  │  KYC认证     │  │  用户服务     │
     └──────┬───────┘  └──────────────┘  └──────────────┘
            │ 同进程方法调用（不走网络）
     ┌──────┴──────────────────────────┐
     │  ser-wallet 内部                  │
     │  ┌────────────┐ ┌────────────┐  │
     │  │ 币种配置模块 │ │ 钱包业务模块│  │
     │  │ (计价引擎)  │→│ (账户引擎)  │  │
     │  └────────────┘ └─────┬──────┘  │
     └───────────────────────┼─────────┘
                             │ RPC (被调用)
                      ┌──────▼───────┐
                      │ ser-finance  │
                      │ 财务(交易引擎)│
                      └──────┬───────┘
                             │ HTTP回调
                      ┌──────▼───────┐
                      │  三方支付/链上 │
                      └──────────────┘

     ┌──────────────┐
     │   B端运营     │
     └──────┬───────┘
            │ HTTPS
     ┌──────▼───────┐
     │  gate-back   │  B端网关(Hertz)
     │  白名单+鉴权  │  BackAuthMiddleware
     └──────┬───────┘
            │ RPC
            ▼
     ser-wallet (币种配置B端接口)
     ser-finance (财务管理B端接口)
```

### 2.2 数据流向

```
资金入方向（充值）：
  C端 → gate-font → ser-wallet(创建订单)
  → ser-finance(选通道) → 三方支付 → 回调 ser-finance
  → ser-finance RPC调 ser-wallet(WalletCredit 入账)

资金出方向（提现）：
  C端 → gate-font → ser-wallet(创建订单+冻结)
  → ser-finance(审核+代付) → 三方支付 → 回调 ser-finance
  → ser-finance RPC调 ser-wallet(WalletDeduct 扣款 / WalletUnfreeze 解冻)

资金内转方向（兑换）：
  C端 → gate-font → ser-wallet(内部闭环，不涉及 ser-finance)

配置方向：
  B端 → gate-back → ser-wallet(币种CRUD)
  定时任务 → ser-wallet(汇率更新)
```

### 2.3 架构要点

**ser-wallet 是单一服务进程**，币种配置和钱包业务在同一进程内，通过 Go 包方法调用（不走网络 RPC）。这是正确的——它们共享数据库和缓存，拆成两个服务会引入不必要的分布式事务问题。

**ser-finance 是独立服务进程**，与 ser-wallet 通过 Kitex RPC 通信。这也是正确的——财务对接三方支付通道，职责边界清晰，独立部署。

---

## 三、账户模型架构

> 这是整个钱包系统最核心的架构决策

### 3.1 业界两种主流账户模型

```
模型A：扁平账户模型（一个账户一个余额）
  user_id → balance
  简单，但无法区分资金性质

模型B：多子账户模型（一个用户多个账户，按资金性质隔离）
  user_id → [中心账户, 奖励账户, 主播账户, ...]
  每个子账户独立余额，资金性质清晰

本系统采用 模型B，原因：
  - 业务明确要求五种子钱包（中心/奖励/主播/代理/场馆）
  - 不同子钱包有不同的使用规则（消费优先级、流水要求、提现限制）
  - 资金隔离是合规硬要求
```

### 3.2 账户模型设计

```
┌─────────────────────────────────────────────────────────────┐
│ 账户模型（Account Model）                                     │
│                                                              │
│ 一个"钱包账户"由三个维度唯一确定：                               │
│   (user_id, currency_code, wallet_type)                      │
│                                                              │
│ 每个账户有两个余额字段：                                        │
│   available_balance  可用余额（用户可操作的钱）                  │
│   frozen_balance     冻结余额（提现审核中的钱，暂时不可操作）      │
│                                                              │
│ 账户总余额 = available_balance + frozen_balance                │
│ 但 C 端用户看到的"余额" = 中心.available + 奖励.available      │
│                                                              │
│ 示例：用户 1001 在 VND 币种下的账户结构                         │
│                                                              │
│   (1001, VND, center)   available=50000  frozen=10000        │
│   (1001, VND, reward)   available=8000   frozen=0            │
│   (1001, VND, streamer) available=12000  frozen=0            │
│                                                              │
│   用户看到的余额 = 50000 + 8000 = 58000                       │
│   冻结的 10000 是正在提现审核的钱                               │
│                                                              │
│ 每个币种一套完整结构，币种间完全隔离                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 账户生命周期

```
创建：用户首次访问某币种时，惰性初始化该币种下的子账户组
      INSERT ON DUPLICATE KEY IGNORE（防并发重复创建）
      初始化 3 个账户（center/reward/streamer），预留 2 个（agent/venue）

读取：查询 available_balance + frozen_balance
      C端返回精简视图，内部RPC返回完整视图

更新：只能通过下面 6 种原子操作修改余额（Service 层统一入口）
      不允许直接 UPDATE 余额字段

销毁：不销毁。账户一旦创建永久存在（余额可能为0但账户不删除）
```

### 3.4 六种余额原子操作

这是账户模型的操作接口，所有业务逻辑最终都归结为这 6 种操作的组合：

```
┌────────────────────────────────────────────────────────────────────┐
│ 操作             │ available_balance │ frozen_balance │ 触发场景     │
├──────────────────┼───────────────────┼────────────────┼─────────────┤
│ ① credit (入账)  │ += amount         │ 不变            │ 充值/奖金/加款│
│ ② debit (扣减)   │ -= amount         │ 不变            │ 兑换/消费    │
│ ③ freeze (冻结)  │ -= amount         │ += amount       │ 提现发起     │
│ ④ unfreeze(解冻) │ += amount         │ -= amount       │ 审核驳回/失败│
│ ⑤ deduct (扣款)  │ 不变              │ -= amount       │ 出款成功     │
│ ⑥ debitSafe(减款)│ -= min(amt,bal)   │ 不变            │ 人工减款     │
└────────────────────────────────────────────────────────────────────┘

每种操作都是一个独立方法，内部包含：
  1. 前置校验（余额充足性）
  2. 余额变动（带并发控制）
  3. 写流水记录
  4. 全部在同一个 DB 事务中
```

### 3.5 为什么这么设计

```
六种操作覆盖了所有可能的余额变动场景：
  充值入账      = credit
  奖金入账      = credit（目标=奖励钱包）
  兑换扣法币    = debit
  兑换加BSB     = credit
  提现冻结      = freeze
  审核驳回      = unfreeze
  出款成功      = deduct
  人工加款      = credit
  人工减款      = debitSafe（不超余额）
  消费扣款(未来) = debit（先中心后奖励）
  返奖(未来)    = credit（按比例分配）

把余额操作收拢到 6 个方法，好处：
  - 所有资金变动都有统一的并发控制入口
  - 所有资金变动都必须写流水（不可能遗漏）
  - 审计简单：只需要审计这 6 个方法
  - 测试简单：只需要测这 6 个方法的正确性
```

---

## 四、记账模型架构

> 回答"每一笔钱的变动怎么记录、怎么证明没有出错"

### 4.1 业界两种记账模型的选择

```
模型A：复式记账（Double-Entry Bookkeeping）
  每笔交易至少两条记录（一借一贷），借贷必须平衡
  用途：银行核心系统、企业财务系统
  复杂度高，需要设计借方/贷方科目体系

模型B：增强单式记账（Enhanced Single-Entry）
  每笔余额变动记录一条流水，流水中含变动前后余额快照
  用途：互联网钱包、游戏虚拟货币、电商账户
  复杂度适中，满足对账需求

本系统采用 模型B，原因：
  - 我们是用户钱包系统，不是银行核心系统
  - 账户结构清晰（子钱包已天然隔离资金性质）
  - 复式记账需要设计会计科目体系，是过度设计
  - 增强单式记账 + 严格的流水校验 = 足够保证资金安全
```

### 4.2 流水记录设计

```
每次调用六种原子操作之一，必须同时写一条流水记录：

┌──────────────────────────────────────────────────────────────┐
│ 流水记录（Wallet Flow）                                        │
│                                                              │
│ 核心字段：                                                    │
│   flow_id          雪花ID（全局唯一）                          │
│   user_id          用户                                      │
│   currency_code    币种                                      │
│   wallet_type      子钱包类型                                 │
│   flow_type        变动类型（枚举：充值入账/奖金入账/兑换扣减    │
│                    /兑换入账/冻结/解冻/扣款/人工加款/人工减款）   │
│   amount           变动金额（正=增加，负=减少）                  │
│   balance_before   变动前余额                                 │
│   balance_after    变动后余额                                 │
│   frozen_before    变动前冻结余额                              │
│   frozen_after     变动后冻结余额                              │
│   related_order_no 关联订单号                                 │
│   related_biz_type 关联业务类型（充值/提现/兑换/修正）           │
│   remark           备注                                      │
│   create_at        创建时间                                   │
│                                                              │
│ 约束：                                                        │
│   - 只插入，不更新，不删除（不可变审计记录）                     │
│   - balance_after = balance_before + amount                  │
│   - 每条流水都在余额变动的同一个事务中写入                      │
└──────────────────────────────────────────────────────────────┘
```

### 4.3 对账机制

```
一致性公式（任何时刻都必须成立）：

  account.available_balance
    == SUM(flow.amount WHERE wallet_type=X AND flow影响available)

  account.frozen_balance
    == SUM(flow.amount WHERE wallet_type=X AND flow影响frozen)

简化校验方式：
  取该账户最后一条流水的 balance_after，应等于 account.available_balance
  取该账户最后一条流水的 frozen_after，应等于 account.frozen_balance

对账实施：
  方式1：定时任务离线对账（每日凌晨扫描所有账户，比对流水和余额）
  方式2：实时校验（每次余额变动后，assert balance_after == 新余额）
  建议：方式2 在开发阶段启用（快速发现 bug），上线后保留方式1
```

---

## 五、并发控制架构

> 回答"两个请求同时操作同一个账户的钱时怎么办"

### 5.1 并发场景分析

```
高频场景（发生概率高，必须处理）：
  同一用户同时充值+消费 → 两个请求同时 credit 和 debit 同一账户
  同一用户快速连续兑换 → 两个 debit 同一账户

低频场景（发生概率低，但后果严重）：
  同一用户提现+消费同时发生 → freeze 和 debit 竞争同一余额
  finance 重试入账 → 两次 credit 同一订单（幂等问题）

极端场景（极低概率）：
  同一用户在不同设备同时操作 → 需要被正确处理但不需要优化性能
```

### 5.2 采用乐观锁（Optimistic Concurrency Control）

```
这是业界钱包系统最主流的并发控制方案。

原理：
  1. 读取账户当前余额和版本号 (balance, version)
  2. 计算新余额
  3. UPDATE SET balance=新余额, version=version+1 WHERE id=? AND version=旧版本
  4. 如果 affected_rows == 0 → 说明被其他请求先改了 → 重试
  5. 如果 affected_rows == 1 → 成功

在现有工程中的实现方式：

  // 事务内执行
  query.Q.Transaction(func(tx *query.Query) error {
      w := tx.WalletAccount

      // 1. 读取当前状态
      account, err := w.WithContext(ctx).
          Where(w.UserID.Eq(userID)).
          Where(w.CurrencyCode.Eq(currency)).
          Where(w.WalletType.Eq(walletType)).
          First()

      // 2. 校验余额充足
      if account.AvailableBalance < amount {
          return ErrInsufficientBalance
      }

      // 3. 乐观锁更新
      result, err := w.WithContext(ctx).
          Where(w.ID.Eq(account.ID)).
          Where(w.Version.Eq(account.Version)).                 // 版本号匹配
          UpdateSimple(
              w.AvailableBalance.Add(-amount),                   // 原子减
              w.Version.Add(1),                                  // 版本号+1
          )
      if result.RowsAffected == 0 {
          return ErrConcurrencyConflict                          // 触发重试
      }

      // 4. 写流水（在同一事务中）
      return tx.WalletFlow.WithContext(ctx).Create(&flow)
  })
```

### 5.3 为什么选乐观锁而不是悲观锁

```
对比：

┌──────────────┬─────────────────────────┬─────────────────────────┐
│              │ 乐观锁(version)          │ 悲观锁(SELECT FOR UPDATE) │
├──────────────┼─────────────────────────┼─────────────────────────┤
│ 锁粒度        │ 无锁，CAS 更新          │ 行级锁，事务内独占        │
│ 冲突处理      │ 冲突时重试              │ 排队等待                  │
│ 性能（低冲突）│ ★★★ 无锁开销           │ ★★ 有锁开销              │
│ 性能（高冲突）│ ★★ 重试有开销           │ ★★ 等待有开销            │
│ 死锁风险      │ 无                      │ 有（多行锁顺序不一致时）   │
│ 实现复杂度    │ 低（加 version 字段即可）│ 中（需注意锁顺序）        │
│ 适用场景      │ 读多写少，冲突率低       │ 写频繁，冲突率高          │
└──────────────┴─────────────────────────┴─────────────────────────┘

选择乐观锁的依据：
  1. 同一用户同一币种同一子钱包的并发写入概率不高
  2. 无死锁风险（钱包系统不能接受死锁导致的余额锁定）
  3. TiDB 对乐观事务有原生支持
  4. 重试逻辑简单（最多重试 2-3 次，仍失败则返回错误让上游重试）
```

### 5.4 重试策略

```
当乐观锁冲突时：

func withRetry(ctx context.Context, maxRetries int, fn func() error) error {
    for i := 0; i <= maxRetries; i++ {
        err := fn()
        if err == ErrConcurrencyConflict && i < maxRetries {
            continue  // 重试，无需 sleep（乐观锁冲突是瞬时的）
        }
        return err
    }
    return ErrConcurrencyConflict
}

// 调用方式
err := withRetry(ctx, 2, func() error {
    return walletService.credit(ctx, params)
})

最多重试 2 次（共 3 次尝试）。
不需要 sleep——乐观锁冲突是因为另一个事务刚提交，再试一次大概率成功。
3 次都失败说明并发极高（不正常），返回错误让调用方决定。
```

### 5.5 分布式锁的使用场景

```
乐观锁解决"同一账户并发余额更新"。
但有些场景需要分布式锁：

场景1：汇率定时任务（防多实例重复执行）
  锁 Key：wallet:lock:rate_update
  用 Redis SetNX + TTL + Lua释放

场景2：充值订单回调处理（防重复入账，幂等的另一层保障）
  锁 Key：wallet:lock:deposit:{order_no}
  配合DB幂等检查双重保障

实现方式（基于现有 rds.SetNX + rds.Eval）：

  // 加锁
  func AcquireLock(key, token string, ttl time.Duration) bool {
      return rds.SetNX(key, token, ttl)
  }

  // 释放（Lua CAS，只有持有者能释放）
  func ReleaseLock(key, token string) bool {
      script := "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end"
      result := rds.Eval(script, []string{key}, token)
      return result == int64(1)
  }

  现有工程中 rds.SetNX 和 rds.Eval 均已可用，只需封装上层函数。
```

---

## 六、订单状态机架构

> 回答"每种订单经历哪些状态、怎么流转、怎么防止非法跳转"

### 6.1 充值订单状态机

```
        ┌─────────┐
        │  待支付  │ ← 创建订单后的初始状态
        │ pending  │
        └────┬────┘
             │ 用户完成支付 / 三方回调 / 链上确认
             ▼
        ┌─────────┐
        │ 处理中   │ ← 收到支付通知，正在入账
        │processing│
        └────┬────┘
             │
        ┌────┴────┐
        │         │
        ▼         ▼
  ┌─────────┐ ┌─────────┐
  │  成功    │ │  失败    │
  │completed│ │ failed  │
  └─────────┘ └─────────┘

合法转换：
  pending → processing → completed
  pending → processing → failed
  pending → failed（超时未支付）

不合法转换（代码级防御）：
  completed → 任何状态（终态不可变）
  failed → 任何状态（终态不可变）
```

### 6.2 提现订单状态机

```
        ┌─────────┐
        │ 审核中   │ ← 创建订单+冻结后的初始状态
        │reviewing │
        └────┬────┘
             │
        ┌────┴────────────┐
        │                 │
        ▼                 ▼
  ┌─────────┐       ┌─────────┐
  │风控通过  │       │风控驳回  │ → WalletUnfreeze → 终态
  │risk_pass│       │risk_deny│
  └────┬────┘       └─────────┘
       │
  ┌────┴────────────┐
  │                 │
  ▼                 ▼
┌─────────┐   ┌─────────┐
│财务通过  │   │财务驳回  │ → WalletUnfreeze → 终态
│fin_pass │   │fin_deny │
└────┬────┘   └─────────┘
     │
     ▼ 发起代付
┌─────────┐
│ 出款中   │
│paying   │
└────┬────┘
     │
┌────┴────┐
│         │
▼         ▼
┌────────┐ ┌────────┐
│出款成功 │ │出款失败│
│paid    │ │pay_fail│
└────┬───┘ └────┬───┘
     │          │
     ▼          ▼
WalletDeduct  WalletUnfreeze
(最终扣款)    (解冻归还)

关键：
  提现的每个状态只有 1-2 个合法的下一步
  每次状态变更前必须校验当前状态是否是合法的前置状态
```

### 6.3 冻结记录状态机

```
   ┌──────────┐
   │  冻结中   │ ← WalletFreeze 创建
   │ freezing │
   └────┬─────┘
        │
   ┌────┴────┐
   │         │
   ▼         ▼
┌────────┐ ┌────────┐
│已解冻  │ │已扣款  │
│unfrozen│ │deducted│
└────────┘ └────────┘

两个终态互斥：同一笔冻结记录只能走到其中一个终态。
代码防御：操作前必须 WHERE status = 'freezing'，affected_rows=0 则拒绝。
```

### 6.4 状态机的代码实现模式

```go
// 状态转换守卫（通用模式）
func (r *OrderRepo) TransitStatus(ctx context.Context, orderNo string, from, to int32) (bool, error) {
    o := query.Q.WithdrawOrder
    result, err := o.WithContext(ctx).
        Where(o.OrderNo.Eq(orderNo)).
        Where(o.Status.Eq(from)).       // 前置状态必须匹配
        UpdateSimple(o.Status.Value(to))
    if err != nil {
        return false, err
    }
    return result.RowsAffected > 0, nil // false = 状态不匹配，转换失败
}

// 调用方
ok, err := repo.TransitStatus(ctx, orderNo, StatusReviewing, StatusRiskPass)
if !ok {
    return errors.New("订单状态不允许此操作")
}
```

WHERE 条件中带前置状态，利用数据库的原子性保证状态转换不会非法跳转。这是业界标准做法。

---

## 七、幂等性架构

> 回答"同一请求被重复执行时，怎么保证只生效一次"

### 7.1 需要幂等保障的接口

```
所有被外部系统调用且会修改余额的 RPC 接口：
  WalletCredit / WalletCreditReward / WalletManualAdd / WalletManualSub
  DepositNotify（链上回调）

为什么：
  finance 调 WalletCredit 时如果网络超时，finance 不知道 wallet 是否成功，
  会按照"最多重试3次"的策略再调一次。
  如果 wallet 没有幂等保护 → 重复入账 → 资金凭空增加。
```

### 7.2 幂等实现方案

```
采用 "业务单号唯一约束" 方案（业界最主流，不依赖额外组件）：

┌──────────────────────────────────────────────────────────────┐
│ 方案：在流水表的 (related_order_no, flow_type) 上建唯一索引     │
│                                                              │
│ 流程：                                                        │
│   1. 收到请求（如 WalletCredit, order_no=C123）               │
│   2. 开启事务                                                 │
│   3. 更新余额                                                 │
│   4. 插入流水（related_order_no=C123, flow_type=充值入账）     │
│   5. 如果插入成功 → 事务提交 → 返回成功                        │
│   6. 如果唯一索引冲突 → 事务回滚 → 说明已处理过 → 返回成功      │
│                                                              │
│ 优势：                                                        │
│   - 利用数据库唯一索引保证原子性，没有 TOCTOU 竞态问题          │
│   - 不需要额外的 Redis 锁或幂等表                              │
│   - 流水表本身就是幂等记录                                     │
│   - 与余额更新在同一个事务中，要么都成功要么都失败               │
└──────────────────────────────────────────────────────────────┘

对比其他方案：
  Redis SetNX → 和DB事务不在同一原子单元，可能Redis成功但DB失败
  独立幂等表  → 额外维护一张表，本质和流水表唯一索引一样
  所以直接用流水表最简洁可靠
```

### 7.3 幂等 + 乐观锁的协同

```
完整的入账操作路径：

1. 收到 WalletCredit(order_no, user_id, currency, amount)
2. withRetry(2, func() {
3.     Transaction {
4.         读取账户余额和版本号
5.         UPDATE 余额 WHERE version=旧版本 (乐观锁)
6.         INSERT 流水 (related_order_no + flow_type 唯一索引)
7.     }
8. })

可能的结果：
  - 乐观锁冲突 → 重试（步骤 2 的 retry）
  - 唯一索引冲突 → 幂等拦截，返回成功
  - 正常成功 → 返回成功
  - 其他DB错误 → 返回错误
```

---

## 八、跨模块通信架构

> 回答"ser-wallet 和 ser-finance 之间怎么通信"

### 8.1 通信方式选择

```
┌──────────────────────────────────────────────────────────────┐
│ 决策：关键资金操作走同步 RPC，非关键通知走异步 MQ              │
│                                                              │
│ 同步 RPC（Kitex，3s超时）：                                   │
│   finance → wallet 的所有资金操作：                            │
│     WalletCredit / CreditReward / Freeze / Unfreeze /        │
│     Deduct / ManualAdd / ManualSub / GetUserBalance /        │
│     GetCurrencyConfig                                        │
│   原因：资金操作必须知道结果（成功/失败），不能"发出去不管"      │
│                                                              │
│ 异步 MQ（NATS JetStream）：                                  │
│   wallet → finance 的事件通知：                                │
│     提现订单创建 → 通知 finance 开始审核                       │
│     充值订单创建 → 通知 finance 选通道（银行/电子钱包场景）     │
│   原因：wallet 不需要等 finance 的处理结果，                    │
│         发出去就行，finance 自己消费处理                        │
│                                                              │
│ 同进程方法调用（Go 包调用）：                                  │
│   钱包模块 → 币种配置模块：                                    │
│     GetCurrencyConfig / GetExchangeRate / CalcExchangeAmount  │
│   原因：同一个 ser-wallet 进程，不走网络                       │
└──────────────────────────────────────────────────────────────┘
```

### 8.2 为什么用 NATS JetStream 而不是 Kafka

```
对比（基于现有工程中两者的实现质量）：

NATS JetStream 优势（在本场景下）：
  - 现有工程已实现 NakWithDelay(5s) + MaxDeliver 重试语义
  - 内置消息持久化和 At-Least-Once 投递保证
  - 自动创建 Stream（省运维配置）
  - 延迟消费（JSPublishDelay）可用于超时未支付自动取消

Kafka 优势（在本场景下不需要）：
  - 更高吞吐量（钱包的消息量不需要 Kafka 级别的吞吐）
  - 更强的分区有序性（钱包消息不严格要求全局有序）

结论：用 NATS JetStream。
  - 已有的 batch consumer + NakWithDelay + MaxDeliver 模式
    天然适合"提现通知财务审核"这种需要重试保障的场景
  - 不需要再引入额外依赖
```

### 8.3 事件消息设计

```
wallet → finance 的事件消息（NATS Subject）：

Subject: wallet.deposit.created    充值订单创建（银行/电子钱包场景）
Subject: wallet.withdraw.created   提现订单创建（通知审核）

消息体（统一格式）：

{
    "event_id": "雪花ID",           // 事件唯一标识（用于消费端幂等）
    "event_type": "withdraw.created",
    "order_no": "T1234567890123456",
    "user_id": 1001,
    "currency_code": "VND",
    "amount": "50000.00",
    "extra": { ... },               // 业务扩展字段
    "timestamp": 1709000000
}

finance 消费时：
  1. 以 event_id 做消费端幂等（防重复处理）
  2. 处理成功 → Ack
  3. 处理失败 → NakWithDelay(5s) → 自动重试
  4. 超过 MaxDeliver 次数 → 进入死信（告警+人工处理）
```

### 8.4 超时与补偿

```
同步 RPC 超时处理：

  finance 调 wallet.WalletCredit 超时时：
  ├── finance 不知道 wallet 是否成功
  ├── 安全做法：直接重试（因为 wallet 有幂等保障）
  └── wallet 如果已处理 → 幂等拦截返回成功；未处理 → 正常执行

  finance 调 wallet.WalletFreeze 超时时：
  ├── finance 不知道钱是否冻结了
  ├── 安全做法：wallet 提供 QueryFreezeStatus(order_no) 查询接口
  └── finance 先查再决定是否重试

跨服务事务补偿：
  不使用分布式事务框架（如 Seata），过度设计。
  用"本地事务 + 重试 + 幂等"模式（业界称为 Best-Effort Delivery）。
  极端情况下不一致 → 对账发现 → 人工修正（WalletManualAdd/Sub）。
```

---

## 九、缓存架构

### 9.1 缓存分层

```
┌─────────────────────────────────────────────────────────────┐
│ 第一层：进程内缓存（Go map/sync.Map，命中率最高，延迟最低）     │
│                                                              │
│   适用数据：币种配置列表、币种精度规则                           │
│   特点：变化频率极低（天/周级别），启动时加载，CRUD后刷新         │
│   实现：CurrencyService 内部持有 sync.Map                     │
│   失效：币种 CRUD 后重新加载                                   │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│ 第二层：Redis 缓存（跨实例共享，延迟毫秒级）                    │
│                                                              │
│   适用数据：汇率数据（按币种缓存）                              │
│   特点：定时任务更新后主动刷新                                  │
│   Key 规范：wallet:rate:{currency_code}                      │
│   TTL：与定时任务间隔一致（如 60s）                             │
│   失效：汇率定时任务更新后 DEL key                             │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│ 第三层：数据库（TiDB，最终数据源）                              │
│                                                              │
│   所有数据的 Source of Truth                                  │
│   缓存未命中时回源查询                                        │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 不缓存什么

```
❌ 用户余额 — 实时性要求高，余额变动频繁，缓存一致性成本高于收益
❌ 订单数据 — 写多读少，状态频繁变化
❌ 冻结记录 — 操作频率不高，但正确性要求极高

原则：涉及资金的实时数据不缓存，宁可多查一次DB，不冒数据不一致的风险。
```

### 9.3 缓存穿透防护

```
空结果缓存：
  查询币种不存在时，也写入缓存（value="NULL"，短 TTL 如 60s）
  防止恶意请求用不存在的币种代码反复穿透到DB

这是现有工程已有的模式（ser-app cache 层对空列表也缓存）。
```

---

## 十、资金安全架构

### 10.1 安全防线体系

```
┌─ 第一道防线：接口层 ──────────────────────────────────────────┐
│ 参数校验 + 权限验证 + 频率限制                                   │
│ 网关层已有：黑名单/白名单 + Token鉴权                            │
│ 服务层补充：金额正数校验、币种有效性校验、用户归属校验             │
└──────────────────────────────────────────────────────────────┘
                              │
┌─ 第二道防线：业务逻辑层 ──────────────────────────────────────┐
│ 余额充足性校验 + 限额校验 + 状态机守卫 + 业务规则校验            │
│ 提现限额（单笔+单日）+ KYC校验 + 重复转账拦截                   │
└──────────────────────────────────────────────────────────────┘
                              │
┌─ 第三道防线：数据层 ─────────────────────────────────────────┐
│ 乐观锁 + 唯一索引幂等 + 事务原子性 + 不可变流水               │
│ 这一层保证即使业务逻辑有 bug，数据层也能兜住                    │
└──────────────────────────────────────────────────────────────┘
                              │
┌─ 第四道防线：事后审计 ───────────────────────────────────────┐
│ 不可变流水记录 + 定期对账 + 操作日志(ser-blog) + 告警          │
│ 发现任何不一致 → 告警 → 人工介入                               │
└──────────────────────────────────────────────────────────────┘
```

### 10.2 金额精度方案

```
┌──────────────────────────────────────────────────────────────┐
│ 决策：数据库和内部传输用 DECIMAL / string，不用 float64         │
│                                                              │
│ 数据库字段：DECIMAL(30,8)                                     │
│   法币精度 2 位，加密精度 8 位                                  │
│   统一用 8 位存储（法币精度由展示层截断）                        │
│   30 位整数部分足够覆盖最大金额场景                             │
│                                                              │
│ IDL 传输：string 类型                                         │
│   避免 Thrift double 类型的精度丢失                            │
│   前后端传输示例："100.50"、"0.00012345"                       │
│                                                              │
│ 服务内部运算：shopspring/decimal 库                            │
│   所有金额运算使用 decimal.Decimal 类型                        │
│   decimal.NewFromString("100.50")                            │
│   结果按币种精度 RoundBank() 后转回 string                     │
│                                                              │
│ 现有工程参考：已有 int64 存最小单位的模式（DiamondConsumeData） │
│   但汇率换算场景不适合 int64（中间结果需要高精度小数），         │
│   所以 DECIMAL + string + decimal 库是更稳妥的方案              │
└──────────────────────────────────────────────────────────────┘
```

### 10.3 防资金溢出

```
所有 debit / freeze 操作前必须校验：
  available_balance >= amount

所有 debitSafe（人工减款）：
  actual = min(amount, available_balance)
  不允许余额变成负数

数据库层约束：
  DECIMAL(30,8) UNSIGNED  → 数据库层面禁止负数
  UPDATE ... SET available_balance = available_balance - ?
  如果结果为负 → TiDB 会报错回滚事务
  这是最后的安全网
```

---

## 十一、数据库架构

### 11.1 数据库选型（现有约束）

```
TiDB（MySQL 兼容）— 项目已在使用，不做变更

TiDB 对钱包系统的关键优势：
  - 水平扩展能力（未来数据量增长时不需要分库分表）
  - MySQL 兼容（GORM + GORM Gen 无缝使用）
  - 乐观事务模型（与我们的乐观锁方案天然契合）
  - DECIMAL 类型完整支持
```

### 11.2 库表隔离

```
ser-wallet 使用独立数据库：db_wallet
ETCD 配置：/slg/serv/wallet/db → db_wallet

ser-finance 使用独立数据库：db_finance（由财务模块负责人管理）

两个服务不直接读取对方的数据库，只通过 RPC 接口交互。
这是微服务的基本原则——数据库私有。
```

### 11.3 核心表关系

```
┌──────────────┐
│ t_currency   │  币种配置（20+字段）
│              │  唯一索引：currency_code
└──────┬───────┘
       │ currency_code
       ▼
┌──────────────────┐     ┌──────────────────┐
│ t_wallet_account │     │ t_rate_log       │
│                  │     │ 汇率变更日志      │
│ 子钱包账户        │     └──────────────────┘
│ 联合唯一：        │
│ (user_id,        │
│  currency_code,  │
│  wallet_type)    │
│                  │
│ version 乐观锁   │
└──────┬───────────┘
       │ account_id (逻辑关联)
       ▼
┌──────────────────┐
│ t_wallet_flow    │  资金流水（只插入，不可变）
│                  │  唯一索引：(related_order_no, flow_type)  ← 幂等保障
│                  │  索引：(user_id, currency_code, wallet_type, create_at)
└──────────────────┘

┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ t_deposit_order  │     │ t_withdraw_order │     │ t_exchange_order │
│ 充值订单          │     │ 提现订单          │     │ 兑换订单          │
│ 唯一：order_no   │     │ 唯一：order_no   │     │ 唯一：order_no   │
└──────────────────┘     └────────┬─────────┘     └──────────────────┘
                                  │
                           ┌──────▼─────────┐     ┌──────────────────┐
                           │ t_freeze_record│     │t_withdraw_account│
                           │ 冻结记录        │     │ 提现账户          │
                           │ 状态机          │     │ 唯一：(user_id,  │
                           └────────────────┘     │  account_type)   │
                                                  └──────────────────┘
```

### 11.4 索引设计原则

```
核心索引（围绕查询模式）：

t_wallet_account：
  PRIMARY KEY (id)
  UNIQUE (user_id, currency_code, wallet_type)   ← 账户唯一性
  INDEX (user_id, currency_code)                  ← 按币种查所有子钱包

t_wallet_flow：
  PRIMARY KEY (id)
  UNIQUE (related_order_no, flow_type)            ← 幂等保障
  INDEX (user_id, currency_code, wallet_type, create_at DESC)  ← 用户流水查询
  INDEX (related_order_no)                        ← 按订单号追溯流水

t_deposit_order / t_withdraw_order / t_exchange_order：
  PRIMARY KEY (id)
  UNIQUE (order_no)                               ← 订单号唯一
  INDEX (user_id, currency_code, status)          ← 用户订单列表
  INDEX (user_id, currency_code, amount, status)  ← 重复转账拦截查询

t_freeze_record：
  PRIMARY KEY (id)
  INDEX (related_order_no)                        ← 按提现单号查冻结
  INDEX (user_id, status)                         ← 查用户所有冻结中的记录
```

---

## 十二、服务内部分层架构

### 12.1 分层图

```
┌─────────────────────────────────────────────────────────────┐
│                     ser-wallet 服务进程                       │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ handler.go (RPC Handler)                             │    │
│  │ 职责：接收RPC请求 → 委托Service → 返回结果            │    │
│  │ 规则：一行委托，不写任何业务逻辑                       │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                          │                                   │
│  ┌──────────────────────▼──────────────────────────────┐    │
│  │ internal/ser/ (Service 层)                           │    │
│  │                                                      │    │
│  │  CurrencyService   ←── 被其他所有 Service 依赖       │    │
│  │    │                                                 │    │
│  │    ▼                                                 │    │
│  │  WalletService   ←── 六种原子操作在这里               │    │
│  │    │                                                 │    │
│  │    ├── DepositService  (调用 WalletService.credit)   │    │
│  │    ├── WithdrawService (调用 WalletService.freeze)   │    │
│  │    ├── ExchangeService (调用 WalletService.debit     │    │
│  │    │                    + WalletService.credit)       │    │
│  │    └── RecordService   (纯查询)                      │    │
│  │                                                      │    │
│  │  职责：业务逻辑 — 校验→计算→持久化→缓存→日志          │    │
│  │  规则：                                               │    │
│  │    - 余额变动只能通过 WalletService 的六种原子操作     │    │
│  │    - defer 失败日志                                   │    │
│  │    - 参数校验独立函数                                  │    │
│  │    - ret.BizErr(ctx, errCode, msg) 统一错误包装       │    │
│  │    - 写操作后清缓存 + ser-blog 审计日志               │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                          │                                   │
│  ┌──────────────────────▼──────────────────────────────┐    │
│  │ internal/rep/ (Repository 层)                        │    │
│  │ 职责：数据访问 — GORM Gen 查询、事务、软删除           │    │
│  │ 规则：                                               │    │
│  │   - 所有查询 .Where(xxx.DeleteAt.Eq(0))             │    │
│  │   - 事务 query.Q.Transaction(func(tx) error)        │    │
│  │   - 不存在返回 nil, nil                              │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                          │                                   │
│  ┌──────────────────────▼──────────────────────────────┐    │
│  │ internal/gorm_gen/ (自动生成，勿手编)                 │    │
│  │   model/  — 数据模型                                  │    │
│  │   query/  — 类型安全查询 DAO                          │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                          │                                   │
│                     TiDB (db_wallet)                         │
│                                                              │
│  旁路：                                                      │
│  ┌────────────────┐  ┌────────────────┐                     │
│  │ internal/cache/ │  │ 定时任务(cron) │                     │
│  │ Redis缓存操作   │  │ 汇率偏差检测   │                     │
│  └────────────────┘  └────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 关键设计约束

```
约束1：余额变动的唯一入口
  所有代码中，能修改 t_wallet_account 余额字段的代码
  只存在于 WalletService 的六种原子操作方法中。
  其他 Service 必须通过调用这些方法来完成资金变动。
  这保证了并发控制、流水记录、幂等检查不会被绕过。

约束2：事务边界在 Repository 层
  Service 层调用 Repo 方法，Repo 方法内部开启事务。
  这与现有工程模式（ser-app、ser-buser）一致。

  但 WalletService 的原子操作比较特殊——
  它需要在一个事务中同时操作 wallet_account 和 wallet_flow 两张表。
  解决方式：WalletRepo 提供 WithTx() 方法，
  在同一个事务中调用多个 Repo 方法。

约束3：CurrencyService 是纯读取
  币种配置模块的查询能力（GetCurrencyConfig/CalcExchangeAmount/GetExchangeRate）
  在 CurrencyService 中实现，被其他所有 Service 作为依赖注入。
  同进程调用，不走网络。
```

---

## 十三、部署与可观测性

### 13.1 部署架构

```
ser-wallet 部署为 N 个实例（N ≥ 2），ETCD 自动注册和发现。
Kitex 客户端内置负载均衡，自动在多实例间分配请求。

多实例下需要注意的：
  - 汇率定时任务：需要分布式锁保证只有一个实例执行
  - 进程内缓存：每个实例独立，CRUD 后需要通知所有实例刷新
    （简单方案：Redis pub/sub 通知；更简单方案：短 TTL 自然过期）
  - 其他无状态：乐观锁和DB事务天然支持多实例
```

### 13.2 可观测性

```
链路追踪（已有基础设施）：
  X-Trace-Id 贯穿全链路
  tracer.GetTraceID(ctx) 在日志中打印

操作日志（已有基础设施）：
  所有写操作 → ser-blog.AddActLog (oneway 异步)
  记录：平台、模块、页面、操作、结果

业务监控（建议新增）：
  关键指标打日志，后续可接入 Prometheus/Grafana：
  - 入账笔数/金额（按币种）
  - 冻结/解冻/扣款笔数
  - 乐观锁重试次数
  - 幂等拦截次数
  - 汇率更新成功/失败次数

资金对账（建议新增）：
  定时任务（每日凌晨）：
  扫描所有 wallet_account，比对 wallet_flow 的 sum
  不一致 → 告警日志 + 通知
```

---

## 十四、架构决策记录（ADR）

> 记录关键架构决策的原因，供后续回溯

### ADR-01：账户模型选择多子账户模型

```
背景：需要区分充值资金、奖励资金、主播收入等不同性质的钱
决策：每个(用户, 币种, 子钱包类型)是一个独立账户
替代方案：单账户+标签、单账户+虚拟子账户
选择原因：业务明确要求五种子钱包各有独立规则，独立账户最清晰
风险：账户记录数量 = 用户数 × 币种数 × 子钱包类型数，量级可控
```

### ADR-02：记账模型选择增强单式记账

```
背景：需要可靠的资金变动记录
决策：每笔余额变动写一条流水，流水含 before/after 快照
替代方案：复式记账（Double-Entry Bookkeeping）
选择原因：我们是用户钱包不是银行核心系统，子钱包已天然隔离资金性质，
         复式记账需要设计会计科目体系，是过度设计
风险：如果未来需要升级为复式记账，流水表的数据可以作为迁移基础
```

### ADR-03：并发控制选择乐观锁

```
背景：同一账户可能被并发修改
决策：version 字段 + CAS 更新 + 有限重试
替代方案：SELECT FOR UPDATE 悲观锁
选择原因：同用户同币种同子钱包的并发冲突概率低，乐观锁性能更好，
         无死锁风险，TiDB 原生支持乐观事务
风险：极端高并发下重试次数增加。应对：监控重试率，如果持续高于5%再考虑悲观锁
```

### ADR-04：幂等性选择流水表唯一索引

```
背景：RPC 调用可能重试导致重复入账
决策：t_wallet_flow 的 (related_order_no, flow_type) 唯一索引
替代方案：Redis SetNX、独立幂等表
选择原因：与余额更新在同一个 DB 事务中，原子性天然保证，
         不引入额外组件，流水表本身就是幂等记录
风险：如果同一个 order_no 有多种 flow_type（如充值入账+奖金入账），
     需要 flow_type 参与唯一索引（而非仅 order_no）。这已在设计中考虑。
```

### ADR-05：跨模块通信选择 RPC + MQ 混合

```
背景：ser-wallet 和 ser-finance 需要通信
决策：资金操作走同步 RPC，事件通知走 NATS JetStream
替代方案：全部 RPC、全部 MQ、分布式事务框架(Seata)
选择原因：
  - 资金操作必须知道结果 → 同步 RPC
  - 事件通知不需要等结果 → 异步 MQ
  - 分布式事务框架引入新依赖和复杂度 → 过度设计
  - NATS JetStream 已有成熟的重试语义（NakWithDelay + MaxDeliver）
风险：RPC 超时时 finance 不确定 wallet 的状态。
     应对：wallet 提供幂等接口 + 查询接口，finance 安全重试。
```

### ADR-06：金额类型选择 DECIMAL + string + decimal库

```
背景：金额运算需要精确，不能有浮点误差
决策：DB 用 DECIMAL(30,8)，IDL 传输用 string，内部运算用 shopspring/decimal
替代方案：int64 最小单位（分/聪）
选择原因：
  - 汇率换算的中间结果需要高精度小数，int64 容易溢出
  - 法币2位和加密币8位精度不同，int64 需要不同的放大倍数，易出错
  - DECIMAL + string 是支付行业标准方案
  - 现有工程虽有 int64 模式（礼物/钻石），但那是单精度场景
风险：string 解析有少量性能开销。实际影响可忽略（金额操作不是计算密集型）。
```

### ADR-07：币种配置和钱包在同一服务进程

```
背景：币种配置和钱包业务是否拆成两个服务
决策：不拆，放在同一个 ser-wallet 进程中
替代方案：ser-currency + ser-wallet 两个独立服务
选择原因：
  - 钱包每次操作都需要调用币种配置的汇率/精度/换算
  - 如果拆成两个服务，每次操作多一次网络 RPC 调用（性能损耗）
  - 拆开后兑换操作涉及两个服务的DB → 分布式事务问题
  - 两个模块共享同一个数据库，拆服务没有数据隔离收益
风险：单服务的代码量较大。应对：通过包结构清晰划分模块边界。
```

---

## 架构总图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           ser-wallet 架构总图                            │
│                                                                          │
│  外部入口                                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │gate-font │  │gate-back │  │ser-finance│  │区块链监听 │               │
│  │  C端HTTP  │  │  B端HTTP  │  │   RPC    │  │  回调    │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│       └──────────────┴─────────────┴─────────────┘                      │
│                              │ Kitex RPC                                 │
│  ┌───────────────────────────▼───────────────────────────────────┐      │
│  │ handler.go                                                     │      │
│  │ ┌───────────────┬───────────────┬─────────────────────────┐   │      │
│  │ │ B端币种CRUD    │ C端钱包操作    │ RPC(供finance调用)       │   │      │
│  │ └───────┬───────┴───────┬───────┴─────────────┬───────────┘   │      │
│  └─────────┼───────────────┼─────────────────────┼───────────────┘      │
│            │               │                     │                       │
│  ┌─────────▼───────────────▼─────────────────────▼───────────────┐      │
│  │ Service 层                                                     │      │
│  │                                                                │      │
│  │  CurrencyService ──────────────────────────────────┐          │      │
│  │  (汇率/精度/换算)                                     │          │      │
│  │       │                                              │          │      │
│  │  WalletService ◄─── 六种原子操作 ──────────┐         │          │      │
│  │  (余额管理核心)                              │         │          │      │
│  │       │         ┌──────────┐  ┌──────────┐│┌────────┤          │      │
│  │       ├─────────│ Deposit  │  │ Withdraw ││ │Exchange│          │      │
│  │       │         │ Service  │  │ Service  │││Service │          │      │
│  │       │         └──────────┘  └──────────┘│└────────┘          │      │
│  │       │                                    │                    │      │
│  │       │         ┌──────────┐              │                    │      │
│  │       │         │ Record   │              │                    │      │
│  │       │         │ Service  │              │                    │      │
│  │       │         └──────────┘              │                    │      │
│  └───────┼───────────────────────────────────┼────────────────────┘      │
│          │                                    │                           │
│  ┌───────▼────────────────────────────────────▼────────────────────┐     │
│  │ Repository 层 + GORM Gen                                        │     │
│  │ 事务：query.Q.Transaction  乐观锁：version CAS  幂等：唯一索引   │     │
│  └───────┬─────────────────────────────────────────────────────────┘     │
│          │                                                               │
│  ┌───────▼─────────────┐  ┌─────────────┐  ┌──────────────────┐        │
│  │ TiDB (db_wallet)   │  │ Redis       │  │ NATS JetStream  │        │
│  │ 账户/流水/订单/冻结  │  │ 缓存/分布式锁│  │ 事件通知         │        │
│  └─────────────────────┘  └─────────────┘  └──────────────────┘        │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │ 外部RPC依赖                                                   │       │
│  │ ser-kyc(KYC) │ ser-blog(审计) │ ser-s3(图标) │ ser-buser(操作人)│      │
│  └──────────────────────────────────────────────────────────────┘       │
│                                                                          │
│  ┌──────────────────────┐                                               │
│  │ 定时任务               │                                               │
│  │ 汇率偏差检测(分布式锁) │                                               │
│  └──────────────────────┘                                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```
