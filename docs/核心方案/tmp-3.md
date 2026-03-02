# ser-wallet 核心挑战与解决方案 — 钱包模块编码级技术攻坚手册

> **定位**：钱包是资金中枢，与钱相关的编码零容忍缺陷。本文档将所有"绕不过的硬骨头"分门别类摆上桌面，每个议题给出**工业级最优方案** — 不是讲理论，而是直接给出可落地的Go代码模式、SQL语句、Redis命令和异常处理策略。
> **原则**：每个方案必须满足 — 最优(业界最佳实践) × 最恰当(契合当前技术栈) × 最可行(工程量可控) × 最流行(主流选择) × 最标准(有据可循)。

---

## 目录

- [第一章：并发安全 — 防超扣是生死线](#第一章并发安全--防超扣是生死线)
- [第二章：幂等性设计 — 重复调用零副作用](#第二章幂等性设计--重复调用零副作用)
- [第三章：事务一致性 — 要么全做要么全不做](#第三章事务一致性--要么全做要么全不做)
- [第四章：冻结状态机 — 互斥终态的工程保证](#第四章冻结状态机--互斥终态的工程保证)
- [第五章：跨服务最终一致性 — 补偿代替分布式事务](#第五章跨服务最终一致性--补偿代替分布式事务)
- [第六章：金额精度 — 全链路整数化方案](#第六章金额精度--全链路整数化方案)
- [第七章：缓存策略 — 高频读场景的一致性保障](#第七章缓存策略--高频读场景的一致性保障)
- [第八章：汇率系统 — 多源聚合与快照锁定](#第八章汇率系统--多源聚合与快照锁定)
- [第九章：分布式锁 — Redis锁的正确姿势](#第九章分布式锁--redis锁的正确姿势)
- [第十章：防重与限流 — 业务级防刷策略](#第十章防重与限流--业务级防刷策略)
- [第十一章：异常补偿与对账 — 资金安全的最后防线](#第十一章异常补偿与对账--资金安全的最后防线)
- [第十二章：审计与可追溯 — 每笔资金有据可查](#第十二章审计与可追溯--每笔资金有据可查)
- [第十三章：Mock与渐进集成 — 外部依赖隔离策略](#第十三章mock与渐进集成--外部依赖隔离策略)
- [第十四章：安全编码 — 资金系统的红线清单](#第十四章安全编码--资金系统的红线清单)
- [第十五章：性能与可扩展 — 为增长预留空间](#第十五章性能与可扩展--为增长预留空间)
- [第十六章：开发检查清单 — 编码前必过的关卡](#第十六章开发检查清单--编码前必过的关卡)

---

## 第一章：并发安全 — 防超扣是生死线

### 1.1 问题场景

```
场景A: 同一用户同时发起2笔投注扣款
  T1: DebitWallet(uid=1001, amount=800) — 查余额=1000, 够扣
  T2: DebitWallet(uid=1001, amount=500) — 查余额=1000, 够扣
  T1: UPDATE balance -= 800 → 余额=200 ✓
  T2: UPDATE balance -= 500 → 余额=-300 ✗ 超扣！

场景B: 同一用户同时提现+投注
  T1: FreezeBalance(amount=800) — 查available=1000, 够冻
  T2: DebitWallet(amount=500) — 查available=1000, 够扣
  T1: available -= 800, frozen += 800 → available=200
  T2: available -= 500 → available=-300 ✗ 超扣！

场景C: 兑换过程中并发扣款
  T1: DoExchange — 法币center -= 5000
  T2: DebitWallet — 法币center -= 3000
  两者同时执行, 余额=6000, 结果可能=-2000
```

**核心矛盾**：先查余额再扣款的两步操作之间存在时间窗口，并发请求在这个窗口内都读到相同余额。

### 1.2 解决方案：双保险机制（分布式锁 + SQL乐观锁）

#### 第一道防线：SQL乐观锁（数据库层，绝对保证）

```go
// 核心SQL：UPDATE + WHERE条件原子性保证不超扣
// 这是不可妥协的底线——即使锁失败、缓存错误，SQL层也能兜住

// 扣款
result := tx.Model(&model.UserWallet{}).
    Where("user_id = ? AND currency_code = ? AND wallet_type = ? AND available_balance >= ?",
        userID, currencyCode, walletType, amount).
    Update("available_balance", gorm.Expr("available_balance - ?", amount))

if result.RowsAffected == 0 {
    // 两种可能：余额不足 OR 并发竞争导致条件不满足
    // 无论哪种，都应拒绝本次操作
    return errs.ErrInsufficientBalance
}

// 冻结
result := tx.Model(&model.UserWallet{}).
    Where("user_id = ? AND currency_code = ? AND wallet_type = ? AND available_balance >= ?",
        userID, currencyCode, walletType, amount).
    Updates(map[string]interface{}{
        "available_balance": gorm.Expr("available_balance - ?", amount),
        "frozen_balance":    gorm.Expr("frozen_balance + ?", amount),
    })

// 上账（加钱不需要WHERE条件约束，但需要防止负数传入）
if amount <= 0 {
    return errs.ErrInvalidAmount
}
result := tx.Model(&model.UserWallet{}).
    Where("user_id = ? AND currency_code = ? AND wallet_type = ?",
        userID, currencyCode, walletType).
    Update("available_balance", gorm.Expr("available_balance + ?", amount))
```

**为什么乐观锁优于悲观锁（SELECT FOR UPDATE）？**

```
悲观锁（SELECT ... FOR UPDATE）：
  ✗ 锁定行直到事务结束，高并发时大量请求排队等待
  ✗ 如果事务中有外部RPC调用（如查汇率），锁持有时间不可控
  ✗ 死锁风险：多表操作时锁顺序不一致可能死锁
  ✗ TiDB分布式场景下，悲观锁性能开销更大

乐观锁（UPDATE ... WHERE balance >= amount）：
  ✓ 不锁行，并发请求都能执行UPDATE
  ✓ 只有满足条件的UPDATE才会成功（affected_rows > 0）
  ✓ 失败的请求立即返回，不阻塞
  ✓ TiDB天然支持，性能最优
  ✓ 参考来源：模块参考A（Java法币钱包）的p9方案，已验证

结论：乐观锁是金融级钱包的标准选择
```

#### 第二道防线：Redis分布式锁（应用层，减少冲突）

```go
// 分布式锁：同一用户同一币种的余额操作串行化
// 不是必须的（SQL乐观锁已保证安全），但能减少无效SQL竞争

const (
    lockKeyPrefix = "wallet:lock:"
    lockWaitTime  = 20 * time.Second  // 等锁最长20秒
    lockHoldTime  = 30 * time.Second  // 持锁最长30秒
)

func walletLockKey(userID int64, currencyCode string) string {
    return fmt.Sprintf("%s%d:%s", lockKeyPrefix, userID, currencyCode)
}

// 使用示例（伪代码）
func (s *WalletCore) DebitWallet(ctx context.Context, req *DebitReq) (*DebitResp, error) {
    lockKey := walletLockKey(req.UserID, req.CurrencyCode)

    // 尝试获取分布式锁
    locked, unlock, err := acquireLock(ctx, lockKey, lockWaitTime, lockHoldTime)
    if err != nil {
        // 锁服务异常 → 降级到只用SQL乐观锁（不阻断业务）
        lg.Log().CtxWarnf(ctx, "分布式锁获取异常，降级执行: %v", err)
    } else if !locked {
        // 等锁超时 → 系统繁忙
        return nil, errs.ErrSystemBusy
    } else {
        defer unlock() // 确保释放锁
    }

    // 执行扣款（无论锁是否获取成功，SQL乐观锁都是最终保障）
    return s.debitCore(ctx, req)
}
```

**为什么是双保险而不是只用一个？**

```
只用分布式锁（不用SQL乐观锁）：
  ✗ 锁服务(Redis)宕机 → 所有余额操作无法执行 → 业务完全中断
  ✗ 锁过期(持有30s后自动释放) → 长事务场景下锁失效，并发问题重现
  ✗ 锁实现bug(如主从切换) → 两个进程同时持锁 → 超扣

只用SQL乐观锁（不用分布式锁）：
  ✓ 安全性有保证（数据库是最终裁判）
  ✗ 高并发时大量UPDATE竞争 → affected_rows=0的比例高 → 用户体验差
  ✗ 每次失败的UPDATE也消耗数据库资源

双保险（分布式锁 + SQL乐观锁）：
  ✓ 分布式锁：减少并发冲突，大部分请求串行执行，成功率高
  ✓ SQL乐观锁：兜底保证，即使锁失效也不会超扣
  ✓ 锁服务故障时降级到只用SQL → 业务不中断，只是并发能力下降
  ✓ 来源：模块参考A的Redisson+SQL双保险方案，已在生产验证
```

### 1.3 双钱包扣款的并发安全

```go
// DebitWallet特殊场景：中心+奖励双钱包扣款
// 两个钱包的UPDATE必须在同一事务内，且都带WHERE条件

func (s *WalletCore) debitCore(ctx context.Context, req *DebitReq) (*DebitResp, error) {
    // 1. 幂等检查（见第二章）
    // ...

    // 2. 查询双钱包余额（用于计算分配）
    centerBalance := s.getBalance(ctx, req.UserID, req.CurrencyCode, WalletTypeCenter)
    rewardBalance := s.getBalance(ctx, req.UserID, req.CurrencyCode, WalletTypeReward)
    totalBalance := centerBalance + rewardBalance

    if totalBalance < req.Amount {
        return nil, errs.ErrInsufficientBalance
    }

    // 3. 计算分配：优先扣中心，不足再扣奖励
    centerDebit := min(centerBalance, req.Amount)
    rewardDebit := req.Amount - centerDebit

    // 4. 单事务内执行双钱包扣款
    err := s.db.Transaction(func(tx *gorm.DB) error {
        // 扣中心钱包
        if centerDebit > 0 {
            result := tx.Model(&model.UserWallet{}).
                Where("user_id = ? AND currency_code = ? AND wallet_type = ? AND available_balance >= ?",
                    req.UserID, req.CurrencyCode, WalletTypeCenter, centerDebit).
                Update("available_balance", gorm.Expr("available_balance - ?", centerDebit))
            if result.RowsAffected == 0 {
                return errs.ErrInsufficientBalance // 并发竞争导致余额不足
            }
        }

        // 扣奖励钱包
        if rewardDebit > 0 {
            result := tx.Model(&model.UserWallet{}).
                Where("user_id = ? AND currency_code = ? AND wallet_type = ? AND available_balance >= ?",
                    req.UserID, req.CurrencyCode, WalletTypeReward, rewardDebit).
                Update("available_balance", gorm.Expr("available_balance - ?", rewardDebit))
            if result.RowsAffected == 0 {
                return errs.ErrInsufficientBalance
            }
        }

        // 写流水（在同一事务内）
        if err := s.writeFlow(tx, ...); err != nil {
            return err
        }

        return nil
    })

    // 5. 失效缓存
    s.invalidateBalanceCache(ctx, req.UserID, req.CurrencyCode)

    // 6. 返回分配信息（投注模块需要这个信息计算返奖）
    return &DebitResp{
        Success:     true,
        CenterDebit: centerDebit,
        RewardDebit: rewardDebit,
    }, nil
}
```

### 1.4 并发安全总结

```
┌──────────────────────────────────────────────────────────────────┐
│                    并发安全三层防御                                │
│                                                                  │
│  Layer 1: 分布式锁 (Redis)                                      │
│    目的: 减少并发冲突，提高成功率                                  │
│    失败降级: 跳过，仅依赖Layer 2                                  │
│                                                                  │
│  Layer 2: SQL乐观锁 (WHERE balance >= amount)                   │
│    目的: 数据库级保证不超扣                                       │
│    affected_rows=0: 余额不足或并发竞争                            │
│    这是不可妥协的底线                                             │
│                                                                  │
│  Layer 3: 业务前置校验 (查余额)                                  │
│    目的: 快速拒绝明显不足的请求，减少无效SQL                       │
│    不是安全保证，只是优化手段                                     │
│                                                                  │
│  三层的关系: Layer 3 快速拒绝 → Layer 1 串行化 → Layer 2 兜底    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 第二章：幂等性设计 — 重复调用零副作用

### 2.1 问题场景

```
场景A: 财务模块调CreditWallet充值入账
  T=0s: 财务调 CreditWallet(orderNo="C2026030200001", amount=1000) → 网络超时
  T=3s: 财务未收到响应，自动重试
  T=4s: 第一次请求其实已执行成功（余额+1000）
  T=5s: 第二次请求到达 → 如果没有幂等保证 → 余额再+1000 → 重复入账！

场景B: 投注模块调DebitWallet扣款
  同理: 网络抖动导致重试 → 可能重复扣款

场景C: 财务模块调ConfirmDebit确认扣除
  打款成功后回调 → 网络问题重试 → 不能重复扣除冻结金额

规律: 所有跨服务RPC调用都可能因网络问题被重试
     所有余额变动操作都必须幂等
```

### 2.2 解决方案：orderNo唯一键 + 先查后写

#### 方案核心：wallet_flow表的order_no字段作为幂等键

```sql
-- wallet_flow 表设计（幂等基础）
CREATE TABLE t_wallet_flow (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    currency_code   VARCHAR(16) NOT NULL,
    wallet_type     TINYINT NOT NULL,          -- 1=center, 2=reward, 3=streamer, 4=agent
    flow_type       TINYINT NOT NULL,          -- 1=deposit, 2=withdraw, 3=exchange, ...
    direction       TINYINT NOT NULL,          -- 1=in, 2=out
    amount          BIGINT NOT NULL,           -- 最小单位整数
    before_balance  BIGINT NOT NULL,           -- 变动前余额（审计用）
    after_balance   BIGINT NOT NULL,           -- 变动后余额（审计用）
    order_no        VARCHAR(64) NOT NULL,      -- ★ 幂等键
    biz_type        VARCHAR(32) DEFAULT '',    -- 业务类型标识
    remark          VARCHAR(256) DEFAULT '',
    operator_id     BIGINT DEFAULT 0,          -- B端操作人（人工操作时）
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE KEY uk_order_no (order_no),         -- ★ 唯一索引保证幂等
    INDEX idx_uid_currency (user_id, currency_code),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 幂等处理流程（所有原子操作统一模式）

```go
// 幂等检查 + 业务执行的统一模式
func (s *WalletCore) creditCore(ctx context.Context, req *CreditReq) (*CreditResp, error) {
    // ========== Step 1: 幂等检查 ==========
    existingFlow, err := s.flowRep.GetByOrderNo(ctx, req.OrderNo)
    if err != nil && !errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, errs.ErrSystemInternal // DB查询异常
    }

    if existingFlow != nil {
        // 已处理过 → 直接返回上次结果（幂等响应）
        return &CreditResp{
            Success:    true,
            Idempotent: true, // 标记为幂等重复请求
            FlowID:     existingFlow.ID,
            NewBalance: existingFlow.AfterBalance,
        }, nil
    }

    // ========== Step 2: 正常业务处理 ==========
    // ... 校验、余额变动、写流水（事务内）

    // ========== Step 3: 唯一键冲突兜底 ==========
    // 即使Step 1未查到，并发请求可能同时到达Step 2
    // INSERT flow时唯一键冲突 → 说明另一个请求已成功
    // GORM会返回 duplicate entry error → 捕获并返回幂等响应

    return resp, nil
}

// 唯一键冲突捕获
func isDuplicateKeyError(err error) bool {
    if err == nil {
        return false
    }
    return strings.Contains(err.Error(), "Duplicate entry") ||
           strings.Contains(err.Error(), "1062")
}
```

### 2.3 orderNo规则设计

```
┌───────────────────────────────────────────────────────────────┐
│                    orderNo 生成规则                            │
│                                                               │
│  钱包内部生成（钱包自发起的操作）：                               │
│    充值订单: C + snowflake     (如 C738159264000001)           │
│    提现订单: T + snowflake                                    │
│    兑换订单: E + snowflake                                    │
│    前缀用于区分来源，snowflake保证全局唯一                       │
│                                                               │
│  外部传入（其他模块调RPC时传入）：                               │
│    财务充值回调: 传财务侧订单号（全局唯一即可）                    │
│    投注扣款: 传投注订单号                                      │
│    活动奖励: 传活动订单号                                      │
│    人工加款: A + snowflake（财务审批后生成）                     │
│    人工减款: M + snowflake                                    │
│    充值补单: B + snowflake                                    │
│                                                               │
│  核心约束：                                                    │
│    ① orderNo由调用方负责生成和保证唯一性                        │
│    ② 钱包不生成外部操作的orderNo                                │
│    ③ 同一orderNo永远只处理一次（唯一键保证）                     │
│    ④ 幂等返回必须包含上次的处理结果                              │
└───────────────────────────────────────────────────────────────┘
```

### 2.4 冻结/解冻/确认的幂等特殊性

```go
// 冻结三件套的幂等更复杂 — 需要检查冻结状态

func (s *WalletCore) unfreezeCore(ctx context.Context, req *UnfreezeReq) (*UnfreezeResp, error) {
    // Step 1: 查冻结记录
    freezeRecord, err := s.freezeRep.GetByOrderNo(ctx, req.OrderNo)
    if err != nil {
        return nil, errs.ErrFreezeRecordNotFound
    }

    // Step 2: 状态检查 + 幂等判断
    switch freezeRecord.Status {
    case FreezeStatusFrozen: // status=1, 正常处理
        // 继续执行解冻
    case FreezeStatusUnfrozen: // status=3, 已解冻（幂等）
        return &UnfreezeResp{Success: true, Idempotent: true}, nil
    case FreezeStatusConfirmed: // status=2, 已确认扣除（互斥冲突）
        return nil, errs.ErrFreezeAlreadyConfirmed
    default:
        return nil, errs.ErrInvalidFreezeStatus
    }

    // Step 3: 执行解冻...
}
```

### 2.5 幂等性总结

```
幂等性设计核心三要素:
  ① 唯一标识: orderNo (由调用方生成, wallet_flow唯一键)
  ② 先查后写: 每次操作前先查是否已处理
  ③ 幂等返回: 重复请求返回上次结果, 不是错误

9个必须幂等的接口:
  CreditWallet / DebitWallet / FreezeBalance / UnfreezeBalance / ConfirmDebit
  ManualCredit / ManualDebit / SupplementCredit / RewardCredit

幂等覆盖率: 所有余额变动RPC接口 = 100%
```

---

## 第三章：事务一致性 — 要么全做要么全不做

### 3.1 问题场景

```
场景A: 兑换操作 — 跨币种三钱包操作
  法币center -= 5000 ✓
  BSB center += 50   ← 此处DB异常
  BSB reward += 5    ← 未执行
  结果: 用户法币扣了但BSB没加 → 资金丢失！

场景B: 扣款操作 — 余额变动+流水写入
  center -= 800 ✓
  INSERT flow  ← 此处失败
  结果: 余额变了但没有审计记录 → 无法追溯！

场景C: 冻结操作 — 可用/冻结余额互转
  available -= 1000 ✓
  frozen += 1000     ← 此处失败
  结果: available减了但frozen没加 → 金额凭空消失！
```

### 3.2 解决方案：单库事务 + 跨服务补偿

#### 原则：只在ser-wallet自己的数据库内开事务，绝不做跨服务分布式事务

```
为什么不用分布式事务（如Saga/TCC/2PC）？

① 工程复杂度：Saga需要定义每步的补偿操作，代码量翻倍
② 性能开销：2PC需要两阶段提交，延迟不可接受（投注扣款要求毫秒级）
③ 技术栈不匹配：Kitex生态没有成熟的分布式事务框架
④ 场景不需要：钱包的跨服务操作本质上是"异步回调"模式，天然适合最终一致性
⑤ 业界共识：支付系统普遍采用"本地事务+补偿+对账"而非分布式事务
```

#### 事务边界分类

```
┌──────────────────────────────────────────────────────────────────────┐
│                    事务边界分类                                       │
│                                                                      │
│  Type A: 单表单操作（最简单）                                         │
│    上账: UPDATE wallet balance += amount + INSERT flow               │
│    查询: 纯读, 无事务需求                                            │
│                                                                      │
│  Type B: 多表同库操作（需事务）                                       │
│    冻结: UPDATE wallet(available/frozen) + INSERT freeze + INSERT flow│
│    兑换: UPDATE wallet×3(法币/-,BSB中心/+,BSB奖励/+) + INSERT flow×3 │
│           + INSERT exchange_order                                    │
│    扣款: UPDATE wallet×2(中心/-,奖励/-) + INSERT flow×2              │
│                                                                      │
│  Type C: 跨服务操作（不用事务, 用补偿）                                │
│    充值: 本地建单 → 调财务CreatePayOrder → 失败则更新本地单为失败       │
│    提现: FreezeBalance → 调财务CreateWithdrawOrder → 失败则Unfreeze   │
│    提现回调: 财务调ConfirmDebit/Unfreeze → 本地事务执行即可             │
│                                                                      │
│  口诀: 同库同事务, 跨服务用补偿, 绝不跨库事务                          │
└──────────────────────────────────────────────────────────────────────┘
```

#### 兑换事务（最复杂的Type B场景）

```go
// DoExchange: 单事务内7条SQL, 跨2币种3钱包
// 这是原子性要求最高的操作

func (s *ExchangeService) DoExchange(ctx context.Context, req *ExchangeReq) (*ExchangeResp, error) {
    // 1. 事务外: 前置校验 + 计算
    rate := s.currencySer.LockPlatformRate(ctx, req.CurrencyCode)
    exchangeBSB := req.Amount * ratePrecision / rate  // 整数计算
    bonusBSB := exchangeBSB * bonusRatio / 100

    // 2. 单事务执行所有变更
    var resp *ExchangeResp
    err := s.db.Transaction(func(tx *gorm.DB) error {
        // 2a. 扣法币中心钱包（带乐观锁）
        result := tx.Model(&model.UserWallet{}).
            Where("user_id = ? AND currency_code = ? AND wallet_type = ? AND available_balance >= ?",
                req.UserID, req.CurrencyCode, WalletTypeCenter, req.Amount).
            Update("available_balance", gorm.Expr("available_balance - ?", req.Amount))
        if result.RowsAffected == 0 {
            return errs.ErrInsufficientBalance
        }

        // 获取变动前余额（用于流水记录）
        // 注: 变动前余额 = 当前余额 + 刚扣的金额

        // 2b. 加BSB中心钱包
        if err := tx.Model(&model.UserWallet{}).
            Where("user_id = ? AND currency_code = 'BSB' AND wallet_type = ?",
                req.UserID, WalletTypeCenter).
            Update("available_balance", gorm.Expr("available_balance + ?", exchangeBSB)).Error; err != nil {
            return err
        }

        // 2c. 加BSB奖励钱包
        if bonusBSB > 0 {
            if err := tx.Model(&model.UserWallet{}).
                Where("user_id = ? AND currency_code = 'BSB' AND wallet_type = ?",
                    req.UserID, WalletTypeReward).
                Update("available_balance", gorm.Expr("available_balance + ?", bonusBSB)).Error; err != nil {
                return err
            }
        }

        // 2d. 写法币扣款流水
        // 2e. 写BSB入账流水
        // 2f. 写BSB奖励流水（含流水倍率）
        // 2g. 写兑换记录（含汇率快照）
        // ...（每条INSERT在此事务内）

        return nil
    })

    if err != nil {
        return nil, err // 事务失败，所有变更自动回滚
    }

    // 3. 事务外: 失效双币种缓存
    s.invalidateBalanceCache(ctx, req.UserID, req.CurrencyCode)
    s.invalidateBalanceCache(ctx, req.UserID, "BSB")

    return resp, nil
}
```

### 3.3 跨服务补偿模式（提现场景）

```go
// CreateWithdrawOrder: 先本地事务, 再跨服务调用, 失败则补偿
// 这是Type C场景的标准模式

func (s *WithdrawService) CreateWithdrawOrder(ctx context.Context, req *CreateWithdrawReq) (*CreateWithdrawResp, error) {
    // Phase 1: 前置校验（KYC、限额、余额）
    // ... 省略

    // Phase 2: 本地事务 — 冻结 + 建单
    var orderNo string
    err := s.db.Transaction(func(tx *gorm.DB) error {
        // 冻结余额
        if err := s.walletCore.freezeCore(tx, req.UserID, req.CurrencyCode, req.Amount, orderNo); err != nil {
            return err
        }
        // 创建本地提现订单
        if err := s.orderRep.CreateWithdrawOrder(tx, &order); err != nil {
            return err
        }
        return nil
    })
    if err != nil {
        return nil, err // 冻结或建单失败，事务回滚，无需补偿
    }

    // Phase 3: 跨服务调用 — 调财务创建审核订单
    financeResp, err := rpc.FinanceClient().CreateWithdrawOrder(ctx, &financeReq)
    if err != nil {
        // ★★★ 关键补偿逻辑 ★★★
        // 财务调用失败 → 必须回滚冻结
        lg.Log().CtxErrorf(ctx, "财务CreateWithdrawOrder失败，执行解冻补偿: %v", err)

        unfreezeErr := s.walletCore.UnfreezeBalance(ctx, &UnfreezeReq{
            UserID:       req.UserID,
            CurrencyCode: req.CurrencyCode,
            Amount:       req.Amount,
            OrderNo:      orderNo,
        })
        if unfreezeErr != nil {
            // 解冻也失败 → 严重异常，记录告警，进入人工处理
            lg.Log().CtxErrorf(ctx, "解冻补偿也失败！资金悬空！orderNo=%s, err=%v", orderNo, unfreezeErr)
            // TODO: 发送告警通知运维
        }

        // 更新本地订单状态
        s.orderRep.UpdateStatus(ctx, orderNo, OrderStatusCreationFailed)

        return nil, errs.ErrFinanceServiceUnavailable
    }

    // Phase 4: 关联财务订单号
    s.orderRep.UpdateFinanceOrderNo(ctx, orderNo, financeResp.FinanceOrderNo)

    return &CreateWithdrawResp{OrderNo: orderNo, Status: "pending_review"}, nil
}
```

### 3.4 事务安全编码规范

```
规范1: defer tx.Rollback() 兜底
  即使GORM的Transaction方法内部已处理，显式defer是好习惯

规范2: 事务内不做外部RPC调用
  事务持锁期间调RPC → 如果RPC超时3s → 事务持锁3s → 其他请求等待
  正确做法: 事务外调RPC，事务内只做DB操作

规范3: 事务内SQL数量控制
  兑换: 7条SQL（已是上限）
  超过10条SQL的事务需要拆分或重新设计

规范4: 事务超时设置
  GORM事务默认无超时 → 应设置合理超时（如5s）
  防止死锁或网络问题导致事务挂起

规范5: 先做会失败的操作
  事务内: 先UPDATE(可能affected_rows=0) → 后INSERT(通常成功)
  而不是: 先INSERT → 后UPDATE失败 → 需要回滚INSERT
```

---

## 第四章：冻结状态机 — 互斥终态的工程保证

### 4.1 问题场景

```
提现流程中的冻结状态机:

  FreezeBalance → 状态=1(freezing)
       │
       ├──→ ConfirmDebit → 状态=2(confirmed) → 资金永久离开
       │
       └──→ UnfreezeBalance → 状态=3(unfrozen) → 资金回到可用

  核心约束: 状态2和状态3互斥，同一笔冻结只能走一条路

  危险场景:
    T=0: 风控驳回 → 财务调UnfreezeBalance
    T=0: 同时三方通道返回成功 → 财务调ConfirmDebit
    如果两个请求几乎同时到达 → 必须保证只有一个成功
```

### 4.2 解决方案：SQL WHERE条件原子互斥

```sql
-- 冻结表设计
CREATE TABLE t_wallet_freeze (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    currency_code   VARCHAR(16) NOT NULL,
    amount          BIGINT NOT NULL,
    order_no        VARCHAR(64) NOT NULL,
    status          TINYINT NOT NULL DEFAULT 1, -- 1=freezing, 2=confirmed, 3=unfrozen
    freeze_at       DATETIME NOT NULL,
    finish_at       DATETIME DEFAULT NULL,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_order_no (order_no),
    INDEX idx_uid_status (user_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 解冻操作SQL（原子互斥）
UPDATE t_wallet_freeze
SET status = 3, finish_at = NOW()
WHERE order_no = ? AND status = 1;  -- ★ 关键: 只有status=1才能变

-- 确认扣除SQL（原子互斥）
UPDATE t_wallet_freeze
SET status = 2, finish_at = NOW()
WHERE order_no = ? AND status = 1;  -- ★ 同样的条件

-- 并发场景:
--   T1: UPDATE SET status=3 WHERE status=1 → affected_rows=1 ✓ (先执行)
--   T2: UPDATE SET status=2 WHERE status=1 → affected_rows=0 ✗ (后执行, status已=3)
--   数据库层面保证了互斥性, 无需应用层加锁
```

```go
// Go代码实现
func (s *WalletCore) confirmDebitCore(ctx context.Context, req *ConfirmDebitReq) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 1. 原子更新冻结状态: 只有status=1才能变为2
        result := tx.Model(&model.WalletFreeze{}).
            Where("order_no = ? AND status = ?", req.OrderNo, FreezeStatusFrozen).
            Updates(map[string]interface{}{
                "status":    FreezeStatusConfirmed,
                "finish_at": time.Now(),
            })

        if result.RowsAffected == 0 {
            // 查一下当前状态，给出精确错误
            freeze, _ := s.freezeRep.GetByOrderNo(ctx, req.OrderNo)
            if freeze == nil {
                return errs.ErrFreezeRecordNotFound
            }
            switch freeze.Status {
            case FreezeStatusConfirmed:
                // 已确认（幂等场景）
                return nil // 幂等返回成功
            case FreezeStatusUnfrozen:
                // 已解冻（互斥冲突）
                return errs.ErrFreezeAlreadyUnfrozen
            default:
                return errs.ErrInvalidFreezeStatus
            }
        }

        // 2. 扣减冻结余额（资金永久离开）
        result = tx.Model(&model.UserWallet{}).
            Where("user_id = ? AND currency_code = ? AND wallet_type = ? AND frozen_balance >= ?",
                req.UserID, req.CurrencyCode, WalletTypeCenter, req.Amount).
            Update("frozen_balance", gorm.Expr("frozen_balance - ?", req.Amount))
        if result.RowsAffected == 0 {
            return errs.ErrInsufficientFrozenBalance
        }

        // 3. 写扣款流水
        return s.writeFlow(tx, /* ... */)
    })
}
```

### 4.3 状态机正确性验证清单

```
验证场景1: 正常冻结→确认
  Freeze(orderNo=T001) → status=1 ✓
  ConfirmDebit(orderNo=T001) → status=2 ✓
  再次ConfirmDebit(orderNo=T001) → 幂等返回成功 ✓

验证场景2: 正常冻结→解冻
  Freeze(orderNo=T002) → status=1 ✓
  Unfreeze(orderNo=T002) → status=3 ✓
  再次Unfreeze(orderNo=T002) → 幂等返回成功 ✓

验证场景3: 互斥冲突
  Freeze(orderNo=T003) → status=1
  Unfreeze(orderNo=T003) → status=3 ✓
  ConfirmDebit(orderNo=T003) → 返回 ErrFreezeAlreadyUnfrozen ✓

验证场景4: 并发互斥
  Freeze(orderNo=T004) → status=1
  并发: Unfreeze(T004) + ConfirmDebit(T004) 同时执行
  → 只有一个affected_rows=1，另一个=0 ✓

验证场景5: 不存在的冻结
  ConfirmDebit(orderNo=T999) → 返回 ErrFreezeRecordNotFound ✓

验证场景6: 未冻结直接解冻
  Unfreeze(orderNo=没冻结过的) → 返回 ErrFreezeRecordNotFound ✓
```

---

## 第五章：跨服务最终一致性 — 补偿代替分布式事务

### 5.1 三条业务链路的跨服务策略

```
┌──────────────────────────────────────────────────────────────────────┐
│                三条业务链路的一致性策略                                 │
│                                                                      │
│  充值链路:                                                           │
│    wallet建单 → 调财务CreatePayOrder → 用户支付 → 财务回调CreditWallet│
│    一致性保证: CreditWallet幂等 + 财务可无限重试                      │
│    补偿方向: 财务 → 钱包（单向回调，无需钱包补偿）                     │
│                                                                      │
│  提现链路:                                                           │
│    wallet冻结 → 调财务CreateWithdrawOrder → 审核 → 回调Confirm/Unfreeze│
│    一致性保证: 冻结先于财务调用 + 失败时UnfreezeBalance补偿            │
│    补偿方向: 双向（钱包补偿冻结，财务回调结果）                        │
│                                                                      │
│  兑换链路:                                                           │
│    纯内部单事务，无跨服务问题                                         │
│    一致性保证: 数据库ACID事务                                         │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 充值链路的最终一致性

```
时间线分析:
  T1: 用户确认充值 → wallet创建本地订单(pending) → 调财务CreatePayOrder
  T2: 用户在三方页面支付
  T3: 三方回调财务 → 财务回调wallet的CreditWallet

  T1→T2: 可能数秒到数小时（用户操作时间不可控）
  T2→T3: 通常数秒（三方回调延迟）

  在T1-T3期间，资金还没进入wallet → 无需保证一致性
  T3时刻: CreditWallet是幂等的 → 财务可安全重试

充值一致性唯一风险点:
  Q: 如果CreditWallet执行成功但响应超时？
  A: 财务重试 → orderNo幂等 → 返回已处理 → 无影响

  Q: 如果wallet宕机，财务回调失败？
  A: 财务应有重试队列，wallet恢复后重试成功

  Q: 如果财务也宕机，三方回调丢失？
  A: 日终对账发现差异 → 人工补单（SupplementCredit）
```

### 5.3 提现链路的最终一致性

```
提现是跨服务一致性最复杂的场景，需要逐步分析每个失败点:

Step 5a: FreezeBalance成功, CreateLocalOrder成功, CreateWithdrawOrder失败
  → wallet自动补偿: UnfreezeBalance(orderNo)
  → 更新本地订单: status=creation_failed
  → 用户余额: 恢复原状

Step 5b: FreezeBalance成功, CreateLocalOrder成功, CreateWithdrawOrder超时(不确定)
  → 不确定财务是否创建了审核订单
  → wallet做法: 先UnfreezeBalance(补偿), 更新本地订单失败
  → 后续: 如果财务确实创建了订单，审核通过后调ConfirmDebit
    → 此时freeze已解冻 → ConfirmDebit会失败(status≠1)
    → 财务发现失败 → 进入人工处理（对账发现）
  → 或者: 财务发现创建成功但wallet冻结已解冻 → 财务标记异常

Step 6-7: 审核通过, 财务调ConfirmDebit/UnfreezeBalance失败
  → 财务应有重试机制（同一orderNo, wallet幂等）
  → 极端情况: 出款成功但ConfirmDebit始终失败
    → 资金已出但frozen没扣 → 日终对账发现差异 → 人工处理

核心原则:
  "宁可多冻不可多扣" — 冻结是可逆的(能解冻), 但错误的ConfirmDebit不可逆
  所以: 不确定的情况下, 选择解冻(返还给用户), 而不是确认扣除(永久减少)
```

### 5.4 补偿机制的实现框架

```go
// 跨服务调用的标准补偿模式

type CompensableOperation struct {
    Execute    func(ctx context.Context) error   // 正向操作
    Compensate func(ctx context.Context) error   // 补偿操作
    Name       string                            // 操作名（用于日志）
}

// 执行带补偿的操作序列
func ExecuteWithCompensation(ctx context.Context, ops []CompensableOperation) error {
    var completedOps []CompensableOperation

    for _, op := range ops {
        if err := op.Execute(ctx); err != nil {
            // 当前步骤失败 → 逆序补偿已完成的步骤
            lg.Log().CtxErrorf(ctx, "%s 执行失败: %v, 开始补偿", op.Name, err)

            for i := len(completedOps) - 1; i >= 0; i-- {
                compOp := completedOps[i]
                if compErr := compOp.Compensate(ctx); compErr != nil {
                    // 补偿也失败 → 严重告警
                    lg.Log().CtxErrorf(ctx, "%s 补偿失败: %v", compOp.Name, compErr)
                    // TODO: 发送告警，进入人工处理队列
                }
            }
            return err
        }
        completedOps = append(completedOps, op)
    }
    return nil
}
```

---

## 第六章：金额精度 — 全链路整数化方案

### 6.1 问题本质

```
浮点数精度问题（IEEE 754）:
  0.1 + 0.2 = 0.30000000000000004  (不是0.3！)

金融场景的后果:
  用户充值 0.1 USDT + 0.2 USDT = 0.30000000000000004 USDT
  用户提现 0.3 USDT → 余额剩 0.00000000000000004 → 永远提不干净

  更严重: 大量小数误差累积 → 系统对账不平 → 平台资金损失

业界共识: 金融系统绝不能用浮点数表示金额
```

### 6.2 解决方案：全链路i64整数（最小单位存储）

```
┌──────────────────────────────────────────────────────────────────────┐
│                    全链路整数化方案                                    │
│                                                                      │
│  存储层 (TiDB):    BIGINT        123456                             │
│  传输层 (Thrift):  i64           123456                             │
│  计算层 (Go):      int64         123456                             │
│  展示层 (前端):    格式化显示     1,234.56                           │
│                                                                      │
│  全程不出现float/double/decimal（金额字段）                           │
└──────────────────────────────────────────────────────────────────────┘

精度规则（按币种）:
  VND (越南盾):  0位小数 → 1 VND = 1 (最小单位=1盾)
  IDR (印尼盾):  0位小数 → 1 IDR = 1
  THB (泰铢):    2位小数 → 1 THB = 100 (最小单位=1萨当)
  USDT:          8位小数 → 1 USDT = 100000000 (最小单位=1聪)
  BSB (平台币):  2位小数 → 1 BSB = 100

转换示例:
  用户看到: ₫262,000     → 系统存储: 262000    (VND, 0位)
  用户看到: ฿1,234.56    → 系统存储: 123456    (THB, 2位)
  用户看到: 10.00000000 USDT → 系统存储: 1000000000 (USDT, 8位)
```

### 6.3 汇率计算的精度处理

```go
// 汇率本身不是"金额"，可以用更大精度
// 但计算结果必须立即转回int64

const ratePrecision = 1e8 // 汇率精度: 8位小数

// 兑换计算示例: 5000 VND → ? BSB
// 平台汇率: 1 BSB = 2620 VND (存储为: rate = 262000000000, 即2620.00000000)
func calculateExchange(fiatAmount int64, rate int64, fiatDecimal int, bsbDecimal int) int64 {
    // fiatAmount = 5000 (VND, 已是最小单位, 0位小数)
    // rate = 262000000000 (2620 * 1e8)
    // 计算: bsbAmount = fiatAmount * 1e8 / rate * bsbPrecision
    // 5000 * 100000000 / 262000000000 * 100 = 190 (即1.90 BSB)

    // 使用big.Int避免溢出
    result := new(big.Int).Mul(
        big.NewInt(fiatAmount),
        big.NewInt(ratePrecision),
    )
    result.Div(result, big.NewInt(rate))

    // 转换到BSB最小单位
    bsbPrecisionFactor := int64(math.Pow10(bsbDecimal))
    result.Mul(result, big.NewInt(bsbPrecisionFactor))
    result.Div(result, big.NewInt(int64(math.Pow10(fiatDecimal))))

    return result.Int64()
}
```

### 6.4 金额校验规则

```go
// 所有金额入口的统一校验

func ValidateAmount(amount int64) error {
    if amount <= 0 {
        return errs.ErrInvalidAmount // 金额必须为正整数
    }
    if amount > maxAmount { // 设定单笔上限, 如 1e15 (百万亿级)
        return errs.ErrAmountExceedsLimit
    }
    return nil
}

// IDL层面约束
// struct CreditWalletReq {
//     1: required i64 amount  // 使用i64, 不用double/string
// }
```

---

## 第七章：缓存策略 — 高频读场景的一致性保障

### 7.1 问题场景

```
场景A: 投注前查余额（全平台最高频调用）
  每秒可能数千次 GetBalance → 直接打DB → 数据库撑不住

场景B: 充值入账后立即查余额
  CreditWallet成功 → 用户立即查余额 → 缓存还是旧值 → 用户看到没到账

场景C: 缓存失效瞬间大量请求（缓存击穿/雪崩）
  缓存TTL到期 → 同一时刻100个请求查同一用户余额 → 全部打DB

场景D: 查询不存在的用户（缓存穿透）
  攻击者构造大量不存在的uid → 全部穿透到DB
```

### 7.2 解决方案：Cache-Aside + 主动失效 + singleflight

```
┌──────────────────────────────────────────────────────────────────┐
│                    缓存策略设计                                   │
│                                                                  │
│  读路径 (Cache-Aside):                                          │
│    查Redis → 命中 → 返回                                        │
│    查Redis → 未命中 → singleflight → 查DB → 写Redis(TTL) → 返回│
│                                                                  │
│  写路径 (主动失效):                                              │
│    余额变动 → DB事务提交 → DEL Redis缓存                         │
│    下次查询 → 未命中 → 重建缓存                                  │
│                                                                  │
│  为什么DEL而不是SET？                                            │
│    SET: 事务提交 → SET缓存 → 如果此时有并发写 → 缓存值可能错误   │
│    DEL: 事务提交 → DEL缓存 → 下次读时从DB重建 → 一定是最新值     │
│    DEL是业界标准做法（Cache-Aside Pattern）                       │
└──────────────────────────────────────────────────────────────────┘
```

#### 缓存Key设计

```
┌────────────────────────────────────────────────────────────────┐
│ 缓存对象      │ Key                                  │ TTL    │
├────────────────────────────────────────────────────────────────┤
│ 用户余额      │ wallet:bal:{uid}:{currCode}           │ 60s    │
│ 币种列表      │ wallet:currencies                     │ 5min   │
│ 币种详情      │ wallet:curr:{currCode}                │ 5min   │
│ 汇率数据      │ wallet:rate:{currCode}                │ 由定时覆盖│
│ 分布式锁      │ wallet:lock:{uid}:{currCode}          │ 30s    │
│ 定时任务锁    │ wallet:cron:rate_refresh              │ 90s    │
└────────────────────────────────────────────────────────────────┘

命名空间: wallet: 前缀统一管理
TTL选择: 余额缓存60s(高频变动) > 币种缓存5min(低频变动)
```

#### singleflight防击穿

```go
import "golang.org/x/sync/singleflight"

var balanceSF singleflight.Group

func (s *WalletCore) GetBalanceCached(ctx context.Context, uid int64, currCode string) (int64, error) {
    cacheKey := fmt.Sprintf("wallet:bal:%d:%s", uid, currCode)

    // 1. 查Redis
    val, err := rds.Get(ctx, cacheKey)
    if err == nil && val != "" {
        return strconv.ParseInt(val, 10, 64)
    }

    // 2. 缓存未命中 → singleflight合并并发请求
    sfKey := cacheKey // 用相同key的请求会被合并
    result, err, _ := balanceSF.Do(sfKey, func() (interface{}, error) {
        // 同一用户同一币种，只有一个请求到达DB
        balance, err := s.walletRep.GetBalance(ctx, uid, currCode)
        if err != nil {
            return nil, err
        }

        // 写回Redis
        rds.Set(ctx, cacheKey, strconv.FormatInt(balance, 10), 60*time.Second)

        return balance, nil
    })

    if err != nil {
        return 0, err
    }
    return result.(int64), nil
}
```

#### 余额变动后的缓存失效

```go
// 所有原子操作执行完毕后，必须失效相关缓存

func (s *WalletCore) invalidateBalanceCache(ctx context.Context, uid int64, currCode string) {
    cacheKey := fmt.Sprintf("wallet:bal:%d:%s", uid, currCode)

    if err := rds.Del(ctx, cacheKey); err != nil {
        // DEL失败不阻断业务，TTL会兜底（最多60s后自动过期）
        lg.Log().CtxWarnf(ctx, "缓存失效失败，TTL兜底: key=%s, err=%v", cacheKey, err)
    }
}

// 兑换场景: 需要失效双币种缓存
func (s *ExchangeService) afterExchange(ctx context.Context, uid int64, fiatCode string) {
    s.invalidateBalanceCache(ctx, uid, fiatCode) // 法币缓存
    s.invalidateBalanceCache(ctx, uid, "BSB")     // BSB缓存
}
```

### 7.3 缓存穿透防护

```go
// 查询不存在的用户/币种时，缓存空标记防止重复穿透

const emptyMark = "__EMPTY__"
const emptyTTL = 30 * time.Second // 空标记只缓存30s

func (s *WalletCore) GetBalanceCached(ctx context.Context, uid int64, currCode string) (int64, error) {
    cacheKey := fmt.Sprintf("wallet:bal:%d:%s", uid, currCode)

    val, err := rds.Get(ctx, cacheKey)
    if err == nil {
        if val == emptyMark {
            return 0, nil // 已知不存在，返回0余额
        }
        return strconv.ParseInt(val, 10, 64)
    }

    // 查DB
    balance, err := s.walletRep.GetBalance(ctx, uid, currCode)
    if errors.Is(err, gorm.ErrRecordNotFound) {
        rds.Set(ctx, cacheKey, emptyMark, emptyTTL) // 缓存空标记
        return 0, nil
    }
    if err != nil {
        return 0, err
    }

    rds.Set(ctx, cacheKey, strconv.FormatInt(balance, 10), 60*time.Second)
    return balance, nil
}
```

---

## 第八章：汇率系统 — 多源聚合与快照锁定

### 8.1 三个核心问题

```
问题1: 三方API返回异常值怎么办？
  API-A: 1 BSB = 2620 VND ✓
  API-B: 1 BSB = 2615 VND ✓
  API-C: 1 BSB = 26200 VND ✗ (10倍异常)
  如果直接取平均 → (2620+2615+26200)/3 = 10478 → 完全错误

问题2: 汇率频繁波动导致平台损失
  实时汇率每秒变化 → 如果每次变化都更新平台汇率 → 用户利用微小波动套利

问题3: 用户查看汇率后确认时汇率已变
  T=0: 用户看到 1 BSB = 2620 VND
  T=5s: 后台刷新, 汇率变为 2680 VND
  T=10s: 用户确认兑换 → 用2620还是2680？
```

### 8.2 解决方案

#### 多源聚合 + 异常值过滤

```go
// 汇率定时刷新核心逻辑

func (s *CurrencyService) RefreshExchangeRate(ctx context.Context, currCode string) error {
    // 1. 并行调3家API（每个5s超时）
    type rateResult struct {
        rate float64
        err  error
    }
    results := make([]rateResult, 3)
    var wg sync.WaitGroup

    apis := []func() (float64, error){
        func() (float64, error) { return s.exchangeRateAPI.Fetch(ctx, currCode) },
        func() (float64, error) { return s.openExchangeAPI.Fetch(ctx, currCode) },
        func() (float64, error) { return s.currencyLayerAPI.Fetch(ctx, currCode) },
    }

    for i, api := range apis {
        wg.Add(1)
        go func(idx int, fetchFn func() (float64, error)) {
            defer wg.Done()
            rate, err := fetchFn()
            results[idx] = rateResult{rate, err}
        }(i, api)
    }
    wg.Wait()

    // 2. 收集成功结果
    var validRates []float64
    for _, r := range results {
        if r.err == nil && r.rate > 0 {
            validRates = append(validRates, r.rate)
        }
    }

    // 3. 至少2个成功才有效
    if len(validRates) < 2 {
        lg.Log().CtxWarnf(ctx, "汇率API成功数不足2，保持当前汇率: currency=%s", currCode)
        return nil // 不更新，保持旧值
    }

    // 4. 异常值过滤（中位数法）
    sort.Float64s(validRates)
    median := validRates[len(validRates)/2]
    var filteredRates []float64
    for _, r := range validRates {
        deviation := math.Abs(r-median) / median
        if deviation < 0.1 { // 偏离中位数超过10%的视为异常
            filteredRates = append(filteredRates, r)
        }
    }

    if len(filteredRates) == 0 {
        lg.Log().CtxWarnf(ctx, "所有汇率值偏差过大，保持当前汇率")
        return nil
    }

    // 5. 取平均值作为实时汇率
    var sum float64
    for _, r := range filteredRates {
        sum += r
    }
    realTimeRate := sum / float64(len(filteredRates))

    // 6. 偏差阈值判断：是否更新平台汇率
    currentPlatformRate := s.getCurrentPlatformRate(ctx, currCode)
    deviation := math.Abs(realTimeRate-currentPlatformRate) / currentPlatformRate * 100

    if deviation >= s.getDeviationThreshold(ctx, currCode) {
        // 超过阈值 → 更新平台汇率 + 入款/出款汇率
        s.updatePlatformRate(ctx, currCode, realTimeRate)
    }

    // 7. 无论是否更新平台汇率，都刷新实时汇率缓存
    s.updateRealTimeRateCache(ctx, currCode, realTimeRate)

    return nil
}
```

#### 汇率快照锁定

```go
// 用户操作时锁定汇率，整个操作使用锁定值

func (s *ExchangeService) DoExchange(ctx context.Context, req *ExchangeReq) (*ExchangeResp, error) {
    // 锁定当前平台汇率（读取后不再变）
    rateSnapshot := s.currencySer.GetPlatformRate(ctx, req.CurrencyCode)

    // 后续所有计算都使用 rateSnapshot, 不再查询最新汇率
    exchangeBSB := calculateExchange(req.Amount, rateSnapshot, ...)

    // 事务内使用 rateSnapshot
    // 兑换记录中保存 rateSnapshot（审计追溯用）
    exchangeOrder := &model.ExchangeOrder{
        RateSnapshot: rateSnapshot,
        // ...
    }
}

// 对于充值/提现的汇率快照，保存在订单记录中
// 充值: deposit_order.rate_snapshot
// 提现: withdraw_order.rate_snapshot
// 后续回调时使用订单中保存的快照汇率，不用当前实时汇率
```

#### 定时任务防多实例重复执行

```go
// 多实例部署时，只有一个实例执行汇率刷新

func (s *CurrencyService) StartRateRefreshCron() {
    ticker := time.NewTicker(1 * time.Minute) // 每分钟执行

    go func() {
        for range ticker.C {
            s.tryRefreshRates()
        }
    }()
}

func (s *CurrencyService) tryRefreshRates() {
    ctx := context.Background()
    lockKey := "wallet:cron:rate_refresh"

    // 尝试获取执行锁（90s，覆盖一个刷新周期）
    locked, err := rds.SetNX(ctx, lockKey, "1", 90*time.Second)
    if err != nil || !locked {
        return // 其他实例正在执行，跳过
    }

    // 遍历所有启用的非基准币种
    currencies := s.getEnabledNonBaseCurrencies(ctx)
    for _, curr := range currencies {
        if err := s.RefreshExchangeRate(ctx, curr.CurrencyCode); err != nil {
            lg.Log().CtxErrorf(ctx, "汇率刷新失败: %s, err=%v", curr.CurrencyCode, err)
        }
    }
}
```

---

## 第九章：分布式锁 — Redis锁的正确姿势

### 9.1 锁的使用场景

```
场景1: 同一用户余额操作串行化（第一章）
  Key: wallet:lock:{uid}:{currCode}
  用途: DebitWallet / FreezeBalance 等并发控制

场景2: 定时任务防重复执行（第八章）
  Key: wallet:cron:rate_refresh
  用途: 多实例只有一个执行

场景3: 订单创建防并发
  Key: wallet:order:{uid}:{type}
  用途: 同一用户不能同时创建两个充值/提现订单
```

### 9.2 Redis锁实现要点

```go
// 基于Redis的分布式锁实现要点

// 加锁: SET key value NX PX timeout
// 解锁: Lua脚本保证原子性（只有持有者能解锁）

const unlockScript = `
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
`

func AcquireLock(ctx context.Context, key string, waitTime, holdTime time.Duration) (bool, func(), error) {
    value := uuid.New().String() // 唯一值，防止误删别人的锁

    deadline := time.Now().Add(waitTime)
    for time.Now().Before(deadline) {
        ok, err := rds.SetNX(ctx, key, value, holdTime)
        if err != nil {
            return false, nil, err
        }
        if ok {
            // 加锁成功
            unlock := func() {
                // 使用Lua脚本原子解锁（只有自己能删自己的锁）
                rds.Eval(ctx, unlockScript, []string{key}, value)
            }
            return true, unlock, nil
        }

        // 加锁失败，短暂等待后重试
        time.Sleep(50 * time.Millisecond)
    }

    return false, nil, nil // 等锁超时
}
```

### 9.3 锁降级策略

```
核心原则: 锁是优化手段，不是安全保证
  SQL乐观锁才是安全保证
  所以: 锁失败 → 业务降级继续, 而不是业务中断

降级场景:
  ① Redis宕机 → 跳过加锁, 直接执行SQL
  ② 等锁超时 → 返回"系统繁忙", 用户重试
  ③ 加锁成功但业务执行中Redis主从切换 → 锁可能丢失
     → SQL乐观锁兜底, 不会超扣
```

---

## 第十章：防重与限流 — 业务级防刷策略

### 10.1 重复转账检测

```go
// 充值重复转账检测
// 规则: 同一用户+同一币种+同一金额, N分钟内不超过3笔

func (s *DepositService) CheckDuplicate(ctx context.Context, req *CheckDuplicateReq) (*CheckDuplicateResp, error) {
    // USDT链上充值不适用
    if req.Method == DepositMethodUSDT {
        return &CheckDuplicateResp{IsDuplicate: false}, nil
    }

    // 查询近N分钟内相同条件的订单数
    cutoffTime := time.Now().Add(-30 * time.Minute)
    count, err := s.orderRep.CountPendingDeposits(ctx,
        req.UserID, req.CurrencyCode, req.Amount, cutoffTime)
    if err != nil {
        return nil, err
    }

    switch {
    case count == 0:
        return &CheckDuplicateResp{IsDuplicate: false}, nil
    case count < 3:
        return &CheckDuplicateResp{
            IsDuplicate:  true,
            CanContinue:  true,  // 前端弹窗确认
            PendingCount: count,
        }, nil
    default: // count >= 3
        return &CheckDuplicateResp{
            IsDuplicate:  true,
            CanContinue:  false, // 强制拦截
            PendingCount: count,
        }, nil
    }
}
```

### 10.2 提现日限额校验

```go
// 提现日限额: 今日已提现 + 本次 <= 日限额

func (s *WithdrawService) validateDailyLimit(ctx context.Context, uid int64, currCode string, amount int64) error {
    // 查今日已提现总额（包含待审核、处理中、已成功的，排除已拒绝的）
    today := time.Now().Format("2006-01-02")
    todayWithdrawn, err := s.orderRep.SumTodayWithdrawals(ctx, uid, currCode, today)
    if err != nil {
        return err
    }

    // 获取日限额配置
    dailyLimit := s.getDailyLimit(ctx, currCode)

    if todayWithdrawn + amount > dailyLimit {
        return errs.ErrExceedsDailyLimit
    }

    return nil
}
```

### 10.3 并发提现防护

```
同一用户快速连续点击"确认提现":
  方案1: 前端防抖(提交后按钮置灰3s) — 用户体验层
  方案2: Redis分布式锁(同一用户提现操作串行化) — 应用层
  方案3: SQL乐观锁(WHERE available >= amount) — 数据库层

三层都要做, 但安全保证在方案3
```

---

## 第十一章：异常补偿与对账 — 资金安全的最后防线

### 11.1 异常分级

```
┌──────────────────────────────────────────────────────────────────────┐
│                    异常分级与处理策略                                  │
│                                                                      │
│  Level 1 (自动恢复):                                                │
│    缓存DEL失败 → TTL兜底(60s) → 自动恢复                            │
│    RPC超时重试 → 幂等保证 → 自动恢复                                 │
│                                                                      │
│  Level 2 (自动补偿):                                                │
│    提现创建失败 → 自动UnfreezeBalance → 余额恢复                     │
│    充值建单失败 → 本地订单标记失败 → 无资金影响                       │
│                                                                      │
│  Level 3 (重试队列):                                                │
│    财务回调CreditWallet失败 → 财务侧重试队列 → 重试成功              │
│    ConfirmDebit/UnfreezeBalance失败 → 财务侧重试 → 重试成功          │
│                                                                      │
│  Level 4 (人工处理):                                                │
│    补偿也失败(冻结无法解冻) → 告警通知 → 运维人工处理                  │
│    对账差异 → 人工核实 → ManualCredit/ManualDebit补偿                 │
│    回调金额不一致 → 记录异常日志 → 人工核实                           │
│                                                                      │
│  Level 5 (日终对账):                                                │
│    充值: 财务"已上账清单" vs 钱包"已入账清单" → 差异处理              │
│    提现: 财务"已出款清单" vs 钱包"已确认扣除清单" → 差异处理           │
│    兑换: 纯内部 → 余额总和=0(法币减+BSB增) → 自检                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 11.2 对账方案设计

```
日终对账的基本思路:

① 充值对账
  钱包: 查wallet_flow WHERE flow_type=deposit AND date=today → 汇总{orderNo, amount}
  财务: 提供"已成功充值清单" → 汇总{orderNo, amount}
  比对: 差集 = 财务有但钱包没有 → 漏入账（需补单SupplementCredit）
       差集 = 钱包有但财务没有 → 多入账（理论不可能，需严查）

② 提现对账
  钱包: 查wallet_freeze WHERE status=2(confirmed) AND date=today → 汇总{orderNo, amount}
  财务: 提供"已成功出款清单" → 汇总{orderNo, amount}
  比对: 差集 = 财务出了但钱包没确认 → ConfirmDebit漏执行（需补执行）
       差集 = 钱包确认了但财务没出 → 理论不可能（先出款后确认）

③ 余额自检
  每日: SUM(wallet_flow.direction=1的amount) - SUM(direction=2的amount) == 当前余额?
  如果不等 → 有异常流水或余额被非原子操作修改

对账执行方式:
  可以做成定时任务, 也可以做成B端手动触发
  初期建议: 每日凌晨自动执行 + 差异告警到运维群
```

### 11.3 关键异常处理代码模式

```go
// 通用异常处理wrapper

func (s *WalletCore) safeExecute(ctx context.Context, operationName string, fn func() error) error {
    err := fn()
    if err != nil {
        // 记录详细错误日志（含traceID、用户、操作、错误）
        traceID := tracer.GetTraceID(ctx)
        lg.Log().CtxErrorf(ctx, "[%s] operation=%s, traceID=%s, error=%v",
            "WALLET_ERROR", operationName, traceID, err)

        // 资金相关错误额外记录到独立的审计表或告警通道
        if isFundRelatedError(err) {
            s.alertFundError(ctx, operationName, err)
        }
    }
    return err
}

// 资金异常告警
func (s *WalletCore) alertFundError(ctx context.Context, operation string, err error) {
    // 写入异常审计表
    // 发送告警（企业微信/钉钉/邮件）
    // 这类告警必须被处理，不能忽略
}
```

---

## 第十二章：审计与可追溯 — 每笔资金有据可查

### 12.1 设计原则

```
资金系统的审计三原则:
  ① 每笔余额变动都有流水记录（wallet_flow）
  ② 每条流水记录都有关联的业务单号（order_no）
  ③ 每条流水记录都有变动前后余额（before_balance / after_balance）

审计追溯链:
  用户余额异常 → 查wallet_flow → 找到orderNo
  → 通过orderNo查对应订单（deposit_order/withdraw_order/exchange_order）
  → 确认操作来源和上下文
  → 如果是外部调用 → 通过orderNo在对方系统查原始操作
```

### 12.2 before_balance / after_balance 的写法

```go
// 在事务内获取变动前余额，计算变动后余额

func (s *WalletCore) creditCoreWithFlow(tx *gorm.DB, uid int64, currCode string,
    walletType int, amount int64, orderNo string, flowType int) error {

    // 1. 查当前余额（事务内，读的是事务视图，一致的）
    var wallet model.UserWallet
    if err := tx.Where("user_id = ? AND currency_code = ? AND wallet_type = ?",
        uid, currCode, walletType).First(&wallet).Error; err != nil {
        return err
    }
    beforeBalance := wallet.AvailableBalance
    afterBalance := beforeBalance + amount

    // 2. 更新余额
    if err := tx.Model(&model.UserWallet{}).
        Where("user_id = ? AND currency_code = ? AND wallet_type = ?",
            uid, currCode, walletType).
        Update("available_balance", gorm.Expr("available_balance + ?", amount)).Error; err != nil {
        return err
    }

    // 3. 写流水（含变动前后余额）
    flow := &model.WalletFlow{
        UserID:        uid,
        CurrencyCode:  currCode,
        WalletType:    walletType,
        FlowType:      flowType,
        Direction:     DirectionIn,
        Amount:        amount,
        BeforeBalance: beforeBalance,
        AfterBalance:  afterBalance,
        OrderNo:       orderNo,
        CreatedAt:     time.Now(),
    }
    return tx.Create(flow).Error
}
```

### 12.3 B端审计日志

```go
// B端所有写操作必须记录审计日志

func (s *CurrencyService) EditCurrency(ctx context.Context, req *EditCurrencyReq) error {
    // 1. 查当前值（记录"改前"）
    oldCurrency, _ := s.currencyRep.GetByCode(ctx, req.CurrencyCode)

    // 2. 执行修改
    if err := s.currencyRep.Update(ctx, req); err != nil {
        return err
    }

    // 3. 记录审计日志（fire-and-forget, 不阻塞主流程）
    go func() {
        logCtx := context.Background()
        rpc.BLogClient().AddActLog(logCtx, &blog.AddActLogReq{
            OperatorID: req.OperatorID,
            Module:     "currency_config",
            Action:     "edit",
            TargetID:   req.CurrencyCode,
            OldValue:   utils.MarshalStr(oldCurrency),
            NewValue:   utils.MarshalStr(req),
            Remark:     req.Remark,
        })
    }()

    return nil
}
```

---

## 第十三章：Mock与渐进集成 — 外部依赖隔离策略

### 13.1 Mock策略

```
当前外部依赖状态:
  ✅ 可直接使用: ser-kyc, ser-user, ser-blog, ser-s3, ser-buser, ser-bcom
  ⚠️ 需要Mock:   财务模块(服务名未定, IDL未定义)
  ⚠️ 需要Mock:   活动模块(服务名未定, IDL未定义)
  ⚠️ 需确认:     ser-cron(IDL为空壳)
```

### 13.2 Mock实现方式

```go
// 使用Go接口实现Mock切换

// 定义财务模块接口
type FinanceService interface {
    GetPaymentConfig(ctx context.Context, currCode string, configType string) (*PaymentConfig, error)
    CreatePayOrder(ctx context.Context, req *CreatePayOrderReq) (*CreatePayOrderResp, error)
    CreateWithdrawOrder(ctx context.Context, req *CreateWithdrawOrderReq) (*CreateWithdrawOrderResp, error)
}

// Mock实现
type MockFinanceService struct{}

func (m *MockFinanceService) GetPaymentConfig(ctx context.Context, currCode string, configType string) (*PaymentConfig, error) {
    // 返回预设的充值/提现方式配置
    return &PaymentConfig{
        Methods: []PaymentMethod{
            {Name: "Bank Transfer", Fee: 0, MinAmount: 10000, MaxAmount: 50000000},
            {Name: "E-Wallet", Fee: 100, MinAmount: 5000, MaxAmount: 10000000},
        },
    }, nil
}

func (m *MockFinanceService) CreatePayOrder(ctx context.Context, req *CreatePayOrderReq) (*CreatePayOrderResp, error) {
    return &CreatePayOrderResp{
        ChannelOrderNo: "MOCK_" + req.OrderNo,
        PayURL:         "https://mock-payment.example.com/pay?order=" + req.OrderNo,
    }, nil
}

// 真实实现（IDL可用后替换）
type RealFinanceService struct{}

func (r *RealFinanceService) GetPaymentConfig(ctx context.Context, currCode string, configType string) (*PaymentConfig, error) {
    resp, err := rpc.FinanceClient().GetPaymentConfig(ctx, &finance.GetPaymentConfigReq{
        CurrencyCode: currCode,
        Type:         configType,
    })
    // ... 转换并返回
}

// 切换: 通过依赖注入在Service构造时决定
func NewDepositService(rep *rep.Repository, financeSer FinanceService) *DepositService {
    return &DepositService{rep: rep, financeSer: financeSer}
}

// 开发阶段: NewDepositService(rep, &MockFinanceService{})
// 联调阶段: NewDepositService(rep, &RealFinanceService{})
```

### 13.3 渐进集成路径

```
P0 → P1 → P2 → P3 的集成路径:

P0: 纯内部逻辑（无需任何外部依赖）
  ✅ 币种配置CRUD
  ✅ 6大原子操作（credit/debit/freeze/unfreeze/confirm/writeFlow）
  ✅ 兑换全流程（GetExchangeRate + DoExchange）
  ✅ RPC暴露接口
  ✅ 记录查询
  验证方式: 单元测试 + 本地集成测试

P1: 接入已有服务（IDL已定义）
  ✅ ser-kyc → 提现KYC校验
  ✅ ser-user → 提现账户姓名校验
  ✅ ser-blog → B端审计日志
  ✅ ser-s3 → 币种图标上传
  验证方式: 联调环境测试

P2: Mock财务模块做端到端
  ⚠️ Mock GetPaymentConfig → 充值/提现方式列表
  ⚠️ Mock CreatePayOrder → 充值下单
  ⚠️ Mock CreateWithdrawOrder → 提现下单
  ⚠️ 模拟CreditWallet回调 → 充值全链路
  ⚠️ 模拟ConfirmDebit/Unfreeze回调 → 提现全链路
  验证方式: 用Mock走完所有Happy Path + Error Path

P3: 替换Mock为真实财务模块
  🔄 获取财务真实IDL → 实现RealFinanceService
  🔄 联调所有接口
  🔄 压力测试
  验证方式: 真实环境全链路测试
```

---

## 第十四章：安全编码 — 资金系统的红线清单

### 14.1 绝对禁止事项（红线）

```
RED LINE 1: 绝不在Service层直接写SQL修改balance
  ✗ tx.Exec("UPDATE user_wallet SET balance = 500 WHERE ...")  // 绝对值赋值
  ✓ tx.Update("balance", gorm.Expr("balance + ?", amount))     // 增量操作
  原因: 绝对值赋值在并发下覆盖其他操作的结果

RED LINE 2: 绝不在事务外修改余额
  ✗ db.Update(balance) → 然后 db.Create(flow)  // 两个独立操作
  ✓ tx.Update(balance) + tx.Create(flow)         // 同一事务
  原因: 余额变了但流水没写 → 无法审计

RED LINE 3: 绝不用浮点数计算金额
  ✗ amount := 1.1 + 2.2  // float64精度丢失
  ✓ amount := int64(110) + int64(220)  // 整数计算
  原因: 浮点精度问题导致资金对不平

RED LINE 4: 绝不信任前端传入的金额计算结果
  ✗ bsbAmount = req.FrontendCalculatedBSB  // 前端传什么就是什么
  ✓ bsbAmount = calculateExchange(req.FiatAmount, serverRate)  // 后端重新计算
  原因: 前端数据可被篡改

RED LINE 5: 绝不跳过幂等检查
  ✗ func CreditWallet() { updateBalance(); writeFlow() }  // 无幂等
  ✓ func CreditWallet() { checkIdempotent(); updateBalance(); writeFlow() }
  原因: 重复调用导致重复入账/扣款

RED LINE 6: 绝不在余额变动操作中忽略affected_rows
  ✗ tx.Update(...); // 不检查结果
  ✓ result := tx.Update(...); if result.RowsAffected == 0 { return err }
  原因: affected_rows=0意味着条件不满足（余额不足），忽略会导致逻辑错误

RED LINE 7: 绝不在事务内调用外部RPC
  ✗ tx.Begin(); rpc.FinanceClient().Call(); tx.Commit()
  ✓ tx.Begin(); dbOps(); tx.Commit(); rpc.FinanceClient().Call()
  原因: RPC超时会导致事务持锁过长，影响并发性能

RED LINE 8: 绝不硬编码金额/汇率
  ✗ if currencyCode == "VND" { rate = 2620 }
  ✓ rate = currencyService.GetPlatformRate(ctx, currencyCode)
  原因: 汇率是动态的，硬编码将导致计算错误
```

### 14.2 防注入与参数校验

```go
// 所有外部输入必须校验

func validateCurrencyCode(code string) error {
    // 只允许大写字母+数字，长度3-16
    matched, _ := regexp.MatchString(`^[A-Z0-9]{3,16}$`, code)
    if !matched {
        return errs.ErrInvalidCurrencyCode
    }
    return nil
}

func validateOrderNo(orderNo string) error {
    // 只允许字母+数字+下划线，长度1-64
    matched, _ := regexp.MatchString(`^[A-Za-z0-9_]{1,64}$`, orderNo)
    if !matched {
        return errs.ErrInvalidOrderNo
    }
    return nil
}

// GORM参数化查询（防SQL注入）
// GORM默认使用参数化查询，但手写SQL时要注意
// ✗ tx.Where(fmt.Sprintf("user_id = %d", uid))  // 字符串拼接
// ✓ tx.Where("user_id = ?", uid)                  // 参数化
```

---

## 第十五章：性能与可扩展 — 为增长预留空间

### 15.1 性能关键路径

```
性能热点排序:
  ① GetBalance — 全平台最高频（投注前必查）
    目标: <5ms (缓存命中), <50ms (缓存未命中)
    方案: Redis缓存 + singleflight

  ② DebitWallet — 投注扣款（实时性要求高）
    目标: <100ms
    方案: 分布式锁减少冲突 + SQL乐观锁

  ③ DoExchange — 兑换（单事务7条SQL）
    目标: <500ms
    方案: 事务内纯DB操作，无外部调用

  ④ CreditWallet — 充值入账（异步回调，容忍度高）
    目标: <200ms
    方案: 幂等检查 + 原子UPDATE

  ⑤ GetWalletHome/GetCurrencyList — 首页（缓存覆盖）
    目标: <50ms
    方案: 缓存命中率>95%
```

### 15.2 数据库索引设计

```sql
-- user_wallet 索引
UNIQUE KEY uk_uid_curr_type (user_id, currency_code, wallet_type)  -- 最核心查询
INDEX idx_uid (user_id)                                            -- B端查某用户所有钱包

-- wallet_flow 索引
UNIQUE KEY uk_order_no (order_no)                                  -- 幂等键
INDEX idx_uid_currency (user_id, currency_code)                    -- 用户维度查流水
INDEX idx_created_at (created_at)                                  -- 时间范围查询

-- wallet_freeze 索引
UNIQUE KEY uk_order_no (order_no)                                  -- 幂等+查找
INDEX idx_uid_status (user_id, status)                             -- 查某用户冻结中的记录

-- deposit_order / withdraw_order 索引
UNIQUE KEY uk_order_no (order_no)
INDEX idx_uid_status_date (user_id, status, created_at)            -- 日限额查询+记录列表
```

### 15.3 可扩展设计

```
扩展点1: 新增钱包类型
  当前: center(1) + reward(2)，预留 streamer(3) + agent(4)
  设计: wallet_type用TINYINT枚举，新增类型只需加枚举值
  影响: 所有原子操作通过wallet_type参数化，无需改代码逻辑

扩展点2: 新增币种
  当前: VND/IDR/THB/USDT/BSB
  设计: currency_config表驱动，新增币种=新增配置行
  影响: 所有逻辑通过currency_code参数化，无需改代码

扩展点3: 新增流水类型
  当前: deposit/withdraw/exchange/bet/reward/manual_credit/manual_debit/freeze/unfreeze/confirm
  设计: flow_type用TINYINT枚举，新增类型只需加枚举值+对应的Service方法

扩展点4: 新增RPC接口
  当前: 13个RPC暴露接口
  设计: IDL新增方法 → gen_kitex.sh → handler路由 → Service实现
  影响: 不改现有代码，纯新增

扩展点5: 水平扩展
  TiDB: 天然支持水平扩展
  Redis: 可集群部署
  ser-wallet: 无状态服务，多实例部署
  定时任务: Redis分布式锁保证单实例执行
```

---

## 第十六章：开发检查清单 — 编码前必过的关卡

### 16.1 每个接口开发前的检查清单

```
□ 1. 该接口是否涉及余额变动？
     是 → 必须走walletCore六大原子操作之一
     否 → 无需特殊处理

□ 2. 该接口是否需要幂等？
     是 → orderNo从哪来？唯一键在哪张表？幂等返回值是什么？
     否 → 仅查询接口无需幂等

□ 3. 该接口是否需要事务？
     是 → 事务边界是什么？几条SQL在事务内？有没有外部RPC？
     否 → 单条SQL无需显式事务

□ 4. 该接口是否有并发风险？
     是 → SQL乐观锁WHERE条件是什么？需要分布式锁吗？
     否 → 纯读接口无并发风险

□ 5. 该接口是否调用外部RPC？
     是 → 目标服务IDL可用吗？超时处理？失败降级？需要Mock吗？
     否 → 纯内部逻辑

□ 6. 该接口失败时如何回滚？
     纯内部 → 事务自动回滚
     跨服务 → 需要补偿操作（如UnfreezeBalance）

□ 7. 缓存策略
     读操作 → 走缓存吗？缓存Key是什么？TTL多少？
     写操作 → 需要DEL哪些缓存Key？

□ 8. 该接口的返回值
     调用方需要什么信息？（如DebitWallet需返回扣款分配比例）
     错误码是否已定义？

□ 9. 该接口在哪个Phase可以开发？
     P0(纯内部) / P1(已有服务) / P2(Mock联调) / P3(真实联调)
```

### 16.2 代码Review核查点

```
金额安全:
  □ 所有金额字段使用int64
  □ 所有余额UPDATE使用增量表达式（gorm.Expr）
  □ 所有扣款UPDATE带WHERE balance >= amount
  □ 所有UPDATE检查affected_rows
  □ 金额入口有正整数校验

事务安全:
  □ 余额变动和流水写入在同一事务内
  □ 事务内无外部RPC调用
  □ 有defer tx.Rollback()兜底
  □ 事务内先做可能失败的操作

幂等安全:
  □ 所有余额变动RPC有orderNo参数
  □ 操作前检查orderNo是否已存在
  □ 幂等返回包含上次结果
  □ wallet_flow.order_no有唯一索引

缓存安全:
  □ 余额变动后DEL对应缓存
  □ DEL失败有日志+TTL兜底
  □ 缓存读取有singleflight防击穿

状态机安全:
  □ 冻结状态变更用WHERE status=1
  □ 确认和解冻互斥(SQL层保证)
  □ 终态(status=2/3)不可再变

错误处理:
  □ 资金相关错误有独立告警
  □ 跨服务调用失败有补偿逻辑
  □ 所有error都有上下文(traceID, uid, orderNo)
```

### 16.3 接口→方案映射速查

```
┌──────────────────────────┬─────────────────────────────────────────────┐
│ 接口                      │ 涉及的核心方案                               │
├──────────────────────────┼─────────────────────────────────────────────┤
│ CreditWallet (R1)        │ 幂等(ch2) + 事务(ch3) + 缓存失效(ch7)       │
│ DebitWallet (R2)         │ 并发(ch1) + 幂等(ch2) + 事务(ch3) + 缓存(ch7)│
│ FreezeBalance (R3)       │ 并发(ch1) + 幂等(ch2) + 状态机(ch4) + 缓存  │
│ UnfreezeBalance (R4)     │ 幂等(ch2) + 状态机(ch4) + 互斥终态          │
│ ConfirmDebit (R5)        │ 幂等(ch2) + 状态机(ch4) + 互斥终态          │
│ ManualCredit (R6)        │ 幂等(ch2) + 自动建钱包 + 审计(ch12)         │
│ ManualDebit (R7)         │ 幂等(ch2) + 级联扣款(ch1) + 审计(ch12)      │
│ GetBalance (R9)          │ 缓存(ch7) + singleflight + 性能(ch15)       │
│ DoExchange (C8)          │ 事务(ch3,最复杂) + 精度(ch6) + 汇率(ch8)    │
│ CreateDepositOrder (C4)  │ 跨服务(ch5) + Mock(ch13) + 防重(ch10)       │
│ CreateWithdrawOrder (C12)│ 并发(ch1) + 状态机(ch4) + 补偿(ch5) +       │
│                          │ 跨服务(ch5) + 防重(ch10) — 全系统最复杂      │
│ GetWithdrawMethods (C9)  │ 三重过滤逻辑 + Mock(ch13) + 降级            │
│ 所有B端写操作             │ 审计日志(ch12) + 缓存失效(ch7)              │
└──────────────────────────┴─────────────────────────────────────────────┘
```

---

## 附录A：核心方案一页纸总结

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    ser-wallet 核心方案一页纸                              │
│                                                                          │
│  并发安全    : SQL乐观锁(WHERE balance>=) + Redis分布式锁 = 双保险       │
│  幂等性      : orderNo唯一键(wallet_flow) + 先查后写 + 幂等返回          │
│  事务一致性  : 同库单事务(GORM) + 跨服务补偿(不用分布式事务)              │
│  状态机      : SQL WHERE status=1 原子互斥(数据库保证, 非程序逻辑)        │
│  最终一致性  : 本地事务 + 补偿回滚 + 幂等重试 + 日终对账                 │
│  金额精度    : 全链路i64整数(最小单位存储), 零浮点数                      │
│  缓存策略    : Cache-Aside + 写后DEL(非SET) + singleflight + TTL兜底     │
│  汇率系统    : 3API取中位数 + 偏差阈值更新 + 快照锁定                    │
│  分布式锁    : SET NX PX + Lua解锁 + 降级到SQL乐观锁                    │
│  防重限流    : 重复转账检测 + 日限额 + 前端防抖 + 后端串行化              │
│  异常补偿    : 4级分类(自恢复→自补偿→重试→人工) + 日终对账              │
│  审计追溯    : wallet_flow(before/after_balance) + B端审计日志            │
│  Mock策略    : Go接口抽象 + Mock/Real切换 + P0→P3渐进集成                │
│  安全红线    : 8条绝对禁止 + 参数校验 + GORM参数化                       │
│  性能设计    : GetBalance<5ms + 缓存命中>95% + 无状态水平扩展            │
│                                                                          │
│  记住: SQL乐观锁是底线, 幂等是生命线, 事务是安全线                       │
└──────────────────────────────────────────────────────────────────────────┘
```

## 附录B：方案来源与业界参考

```
┌────────────────────────────────────────────────────────────────────────┐
│ 方案                │ 来源/参考                                        │
├────────────────────────────────────────────────────────────────────────┤
│ SQL乐观锁           │ 模块参考A(Java法币钱包p9) + 业界标准               │
│ Redis分布式锁       │ 模块参考A(Redisson) → 改为Go原生实现               │
│ orderNo幂等         │ 模块参考A + 模块参考B(TRC20)                      │
│ Cache-Aside        │ 业界标准(AWS/阿里云/美团最佳实践)                   │
│ singleflight       │ Go标准库(golang.org/x/sync/singleflight)          │
│ 补偿代替分布式事务   │ 支付行业共识(Stripe/支付宝/微信支付)               │
│ i64整数化           │ 金融系统标准(ISO 4217最小单位)                     │
│ 状态机SQL互斥       │ 数据库ACID特性的直接应用                           │
│ 日终对账            │ 银行/支付行业标准流程                              │
│ 异常4级分类         │ 运维最佳实践(SRE理念)                              │
│ Mock接口切换        │ Go依赖注入标准模式                                 │
│ GORM事务模式        │ 工程已有代码模式(cfg.InitMySQL().Transaction)      │
└────────────────────────────────────────────────────────────────────────┘
```

---

> **本文档定位**：编码阶段的"安全手册" — 每个方案直接可落地，每条红线不可逾越，每个检查清单必须过关。
> **使用方式**：开发任何接口前，先查16.3的映射表，对照对应章节的方案实施。
> **核心口诀**：SQL乐观锁是底线，幂等是生命线，事务是安全线。
