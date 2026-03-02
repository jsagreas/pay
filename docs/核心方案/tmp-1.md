# 钱包模块核心技术方案——绕不过的问题与最优解

> **定位**：这不是挑战清单，而是**工程决策书**。每个议题给出：问题本质 → 影响范围 → 方案设计（含伪代码）→ 为什么是最优解 → 陷阱与边界 → 验证方法。
> **基线**：所有方案基于现有工程基线（GORM Gen + query.Q.Transaction + rds.SetNX + ret.BizErr + Kitex 3s超时），不引入工程中不存在的框架。

---

## 目录

- [议题一 并发安全与资金防超卖](#议题一)
- [议题二 幂等性保障](#议题二)
- [议题三 本地事务一致性](#议题三)
- [议题四 冻结三阶段(TCC)互斥安全](#议题四)
- [议题五 金额精度与计算安全](#议题五)
- [议题六 死锁预防](#议题六)
- [议题七 缓存一致性](#议题七)
- [议题八 状态机正确性保障](#议题八)
- [议题九 资金安全纵深防线](#议题九)
- [议题十 高频接口性能优化](#议题十)
- [议题十一 跨模块故障隔离与降级](#议题十一)
- [议题十二 审计可追溯性](#议题十二)
- [议题十三 可扩展性设计](#议题十三)
- [附录A 方案选型对比总表](#附录a)
- [附录B 各议题与接口映射矩阵](#附录b)
- [附录C 关键Go代码骨架汇总](#附录c)

---

<a id="议题一"></a>
## 议题一 并发安全与资金防超卖

### 1.1 问题本质

**为什么普通CRUD不会遇到，钱包会遇到？**

普通业务：Banner名称重复了，最多数据脏；钱包：同一用户余额100，两笔50的扣款同时到达，如果不做并发控制，两笔都读到余额100，两笔都扣成功，余额变-50——**资金超卖**。

核心矛盾：**读-判断-写** 三步不在一个原子操作内。两个请求都通过了"余额≥50"的判断，然后各自执行了扣减。

**触发场景：**
- 用户同时发起提现和投注（两笔扣款竞争同一钱包行）
- 充值回调重试（MQ/网络重试导致CreditWallet被并发调用）
- 结算批量返奖（同一用户的多笔返奖几乎同时抵达）
- 兑换操作（同一事务内操作2~3个钱包行）

### 1.2 影响范围

| 接口 | 并发风险等级 | 说明 |
|------|------------|------|
| CreditWallet | 🔴 高 | 充值回调可能重试/并发 |
| DebitWallet | 🔴 高 | 投注扣款是最高频写操作 |
| FreezeBalance | 🔴 高 | 提现冻结必须精确 |
| ExecuteExchange | 🔴 高 | 多行写入，死锁风险 |
| UnfreezeBalance | 🟡 中 | 与ConfirmDebit互斥 |
| ConfirmDebit | 🟡 中 | 与UnfreezeBalance互斥 |
| InitWallet | 🟢 低 | 注册一次，幂等保护即可 |

### 1.3 方案设计：三层递进防护

```
请求到达
    │
    ▼
┌─ Layer 1: Redis分布式锁 (应用层串行化) ─────────────────────┐
│  目的: 将同一用户的并发请求在应用层串行化，减轻DB压力           │
│  粒度: 每用户每钱包类型每币种                                  │
│  99%的并发冲突在这里被拦截                                    │
└──────────┬──────────────────────────────────────────────────┘
           │ 获取锁成功
           ▼
┌─ Layer 2: DB悲观锁 (数据层串行化) ──────────────────────────┐
│  目的: 即使Redis锁失效(宕机/过期)，DB层仍能保证安全           │
│  手段: SELECT ... FOR UPDATE 行级锁                         │
│  在事务内锁定目标行，其他事务阻塞等待                          │
└──────────┬──────────────────────────────────────────────────┘
           │ 行锁获取成功
           ▼
┌─ Layer 3: SQL原子条件更新 (最终防线) ──────────────────────┐
│  目的: 即使前两层都被绕过(极端场景)，SQL本身不会超卖           │
│  手段: UPDATE ... SET amt = amt - ? WHERE amt >= ?          │
│  affected_rows = 0 → 余额不足，拒绝操作                     │
└─────────────────────────────────────────────────────────────┘
```

**Layer 1: Redis分布式锁**

```go
// ============= 分布式锁封装 =============
// 文件: internal/pkg/dlock/dlock.go

const (
    lockPrefix = "wallet:lock:"
    lockTTL    = 8 * time.Second  // 锁持有时间上限
)

// WalletLockKey 生成锁key
// 粒度: userId + walletType + currencyCode
func WalletLockKey(userId int64, walletType int32, currencyCode string) string {
    return fmt.Sprintf("%s%d:%d:%s", lockPrefix, userId, walletType, currencyCode)
}

// WithLock 获取锁→执行→释放锁
// 使用Lua脚本保证释放时的所有权校验
func WithLock(ctx context.Context, key string, fn func() error) error {
    // 生成唯一标识(防止误释放别人的锁)
    value := tracer.GetTraceID(ctx) + ":" + strconv.FormatInt(time.Now().UnixNano(), 36)

    // 尝试获取锁
    acquired := rds.SetNX(key, value, lockTTL)
    if !acquired {
        return ret.BizErr(ctx, errs.WalletOperationBusy, "操作频繁，请稍后重试")
    }

    // 确保释放: Lua脚本原子校验value后删除
    defer func() {
        unlockScript := `
            if redis.call("GET", KEYS[1]) == ARGV[1] then
                return redis.call("DEL", KEYS[1])
            end
            return 0`
        rds.Eval(ctx, unlockScript, []string{key}, value)
    }()

    return fn()
}
```

**为什么不用 Redlock / redsync？**
- 当前工程是单Redis实例，不是Redis集群。SetNX + Lua释放在单实例下已经是最佳方案。
- Redlock是为多Redis节点设计的（多数派投票），引入它反而增加复杂度。
- 如果未来迁移到Redis集群，只需将SetNX替换为redsync.NewMutex()，接口不变。

**为什么锁粒度是 userId + walletType + currencyCode？**
- 粒度太粗（只锁userId）：同一用户的VND充值会阻塞BSB投注 → 不合理。
- 粒度太细（锁到行ID）：需要先查DB才能获取行ID → 失去锁的意义。
- 当前粒度：同一用户同一钱包同一币种串行化，不同钱包/币种互不阻塞 → 最优平衡。

**Layer 2: DB悲观锁（SELECT FOR UPDATE）**

```go
// ============= 仓储层: 加锁查询 =============
// 文件: internal/rep/wallet_account.go

func (r *WalletAccountRepo) GetForUpdate(
    ctx context.Context,
    tx *query.Query,  // 必须使用事务内的tx，不能用全局query.Q
    userId int64,
    walletType int32,
    currencyCode string,
) (*model.WalletAccount, error) {
    w := tx.WalletAccount
    account, err := w.WithContext(ctx).
        Where(w.UserID.Eq(userId)).
        Where(w.WalletType.Eq(walletType)).
        Where(w.CurrencyCode.Eq(currencyCode)).
        ForUpdate(). // SELECT ... FOR UPDATE
        First()
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ret.BizErr(ctx, errs.WalletAccountNotFound, "钱包账户不存在")
        }
        return nil, err
    }
    return account, nil
}
```

**关键陷阱：必须用事务内的 tx，不能用全局 query.Q**

```go
// ❌ 错误: FOR UPDATE在事务外无意义，锁立即释放
account, _ := query.Q.WalletAccount.WithContext(ctx).ForUpdate().First()

// ✅ 正确: 锁持有到事务COMMIT/ROLLBACK
query.Q.Transaction(func(tx *query.Query) error {
    account, _ := tx.WalletAccount.WithContext(ctx).ForUpdate().First()
    // ... 锁在这里持有 ...
    return nil // COMMIT释放锁
})
```

**Layer 3: SQL原子条件更新**

```go
// ============= 仓储层: 原子扣减 =============

// DebitAvailable 原子扣减可用余额
// SQL: UPDATE wallet_account SET available_amt = available_amt - ?
//      WHERE id = ? AND available_amt >= ?
// 返回 affected_rows, 0表示余额不足
func (r *WalletAccountRepo) DebitAvailable(
    ctx context.Context,
    tx *query.Query,
    accountId int64,
    amount decimal.Decimal,
) (int64, error) {
    w := tx.WalletAccount
    result, err := w.WithContext(ctx).
        Where(w.ID.Eq(accountId)).
        Where(w.AvailableAmt.Gte(amount)). // WHERE available_amt >= amount
        UpdateSimple(
            w.AvailableAmt.Sub(amount), // SET available_amt = available_amt - amount
        )
    return result.RowsAffected, err
}

// CreditAvailable 原子增加可用余额 (无需条件检查，加钱永远合法)
func (r *WalletAccountRepo) CreditAvailable(
    ctx context.Context,
    tx *query.Query,
    accountId int64,
    amount decimal.Decimal,
) error {
    w := tx.WalletAccount
    _, err := w.WithContext(ctx).
        Where(w.ID.Eq(accountId)).
        UpdateSimple(
            w.AvailableAmt.Add(amount),
        )
    return err
}
```

**为什么不用乐观锁（Version字段）作为主方案？**

| 对比 | 悲观锁 (FOR UPDATE) | 乐观锁 (Version) |
|------|---------------------|-------------------|
| 适用场景 | 写冲突概率高 | 写冲突概率低 |
| 钱包写冲突概率 | 高（同用户频繁操作） | - |
| 失败处理 | 等待→必定成功 | 检测冲突→重试 |
| 重试代价 | 无需重试 | 重新读→重新算→重试，可能多次 |
| 代码复杂度 | 简单（一次性成功） | 复杂（需重试循环+退避） |
| **结论** | **主方案** | **辅助安全网** |

钱包是**写密集型高冲突场景**，乐观锁会导致频繁重试 → 性能劣化。悲观锁虽然有等待成本，但保证一次性成功。我们用悲观锁做主力，Version字段做辅助安全网（如果某处代码绕过了FOR UPDATE，Version会捕获异常）。

### 1.4 三层协同的完整调用链

```go
// ============= Service层: 完整的扣款流程 =============
// 文件: internal/ser/wallet.go

func (s *WalletService) DebitWallet(ctx context.Context, req *DebitWalletReq) (*DebitWalletResp, error) {
    // === Layer 0: 幂等检查 (详见议题二) ===
    if cached := s.checkIdempotent(ctx, req.BizType, req.RefOrderNo); cached != nil {
        return cached, nil
    }

    // === Layer 1: Redis分布式锁 ===
    lockKey := dlock.WalletLockKey(req.UserId, req.WalletType, req.CurrencyCode)
    var resp *DebitWalletResp

    err := dlock.WithLock(ctx, lockKey, func() error {
        // === Layer 2 + 3: DB事务 ===
        return query.Q.Transaction(func(tx *query.Query) error {
            // Layer 2: SELECT FOR UPDATE 锁行
            account, err := s.accountRepo.GetForUpdate(ctx, tx,
                req.UserId, req.WalletType, req.CurrencyCode)
            if err != nil {
                return err
            }

            // 业务校验
            amount, _ := decimal.NewFromString(req.Amount)
            if account.AvailableAmt.LessThan(amount) {
                return ret.BizErr(ctx, errs.InsufficientBalance, "余额不足")
            }

            // Layer 3: SQL原子条件更新
            affected, err := s.accountRepo.DebitAvailable(ctx, tx, account.ID, amount)
            if err != nil {
                return err
            }
            if affected == 0 {
                // 极端情况: FOR UPDATE拿到的余额够，但并发SQL更新失败
                return ret.BizErr(ctx, errs.InsufficientBalance, "余额不足(并发)")
            }

            // 写入不可变流水 (详见议题三)
            newAvailable := account.AvailableAmt.Sub(amount)
            err = s.txnRepo.InsertFlow(ctx, tx, &FlowParams{
                UserId:          req.UserId,
                WalletType:      req.WalletType,
                CurrencyCode:    req.CurrencyCode,
                OrderNo:         req.RefOrderNo,
                OpType:          req.BizType,
                Direction:       DirectionExpense, // 2=支出
                Amount:          amount,
                AvailableBefore: account.AvailableAmt,
                AvailableAfter:  newAvailable,
                FrozenBefore:    account.FrozenAmt,
                FrozenAfter:     account.FrozenAmt, // 冻结不变
            })
            if err != nil {
                return err
            }

            resp = &DebitWalletResp{
                Success:      true,
                AfterBalance: newAvailable.String(),
            }
            return nil
        })
    })

    if err != nil {
        return nil, err
    }

    // === 事务外操作(不影响主流程) ===
    s.invalidateBalanceCache(ctx, req.UserId, req.CurrencyCode) // 缓存失效
    s.asyncAuditLog(ctx, "DebitWallet", req)                     // 异步审计
    s.cacheIdempotentResult(ctx, req.BizType, req.RefOrderNo, resp) // 幂等缓存

    return resp, nil
}
```

### 1.5 陷阱与边界

| 陷阱 | 说明 | 应对 |
|------|------|------|
| Redis宕机 | Layer1失效，锁获取失败 | SetNX返回false → **降级为仅DB锁**，不拒绝请求 |
| 锁过期但事务未完成 | 业务执行>8s，锁自动释放 | DB层FOR UPDATE兜底；监控事务耗时>5s告警 |
| 误释放别人的锁 | A的锁过期→B获取→A的defer释放了B的锁 | Lua脚本校验value所有权后才删除 |
| 加钱不需要检查余额 | CreditWallet永远合法(amount>0已校验) | CreditAvailable不加WHERE条件，直接+=amount |

### 1.6 验证方法

```
1. 单元测试: 模拟10个goroutine并发扣同一钱包，验证最终余额=初始-成功扣除总额
2. 集成测试: 关闭Redis，验证仅DB锁仍能正确防超卖
3. 压力测试: 100并发对同一用户投注，验证零超卖 + 吞吐量>500TPS
4. 混沌测试: 随机kill Redis连接，验证不出现资金异常
```

---

<a id="议题二"></a>
## 议题二 幂等性保障

### 2.1 问题本质

**为什么钱包的幂等比普通业务更致命？**

普通业务：创建两条重复Banner → 删掉一条就行。
钱包：充值100元被入账两次 → 用户多了100块真金白银，无法自动回退。

**触发场景：**
- 网络超时→调用方自动重试（Kitex默认可能重试）
- MQ消息重投（至少一次语义）
- 前端用户快速双击提交按钮
- ser-finance回调ser-wallet，回调超时→ser-finance重试

### 2.2 方案设计：双层幂等

```
请求到达
    │
    ▼
┌─ Layer 1: Redis快速拦截 (毫秒级) ──────────────────────────┐
│  key = wallet:idem:{bizType}:{refOrderNo}:{walletType}      │
│  SetNX + TTL 24h                                             │
│  → 已存在 = 重复请求 → 立即返回缓存结果                       │
│  → 不存在 = 新请求 → 继续                                    │
│  ★ 拦截99%+的重复请求，DB压力近零                             │
└──────────┬─────────────────────────────────────────────────┘
           │ 新请求
           ▼
┌─ Layer 2: DB唯一索引兜底 (最终保障) ──────────────────────┐
│  wallet_transaction表:                                      │
│  UNIQUE KEY uk_order_op_wallet(order_no, op_type, wallet_type)│
│  INSERT失败(Duplicate Entry) → 幂等保护生效 → 返回成功       │
│  ★ Redis宕机/重启后，DB层仍能防重                            │
└─────────────────────────────────────────────────────────────┘
```

**为什么必须是双层？**

```
仅Redis:  Redis重启 → 幂等key丢失 → 重复请求通过 → 重复入账 ❌
仅DB:     每个请求都INSERT尝试 → DB压力大，高并发下性能差 ❌
双层:     Redis挡99%，DB挡剩余1% → 性能+安全兼得 ✅
```

### 2.3 幂等Key设计

```
key = wallet:idem:{bizType}:{refOrderNo}:{walletType}

为什么包含walletType？
  → 人工加款可能同时给同一用户的中心钱包和奖励钱包加款
  → refOrderNo相同(同一加款单)，但walletType不同(1和2)
  → 这两笔是不同的操作，不应互相幂等拦截
  → 加入walletType后，(A+16, 4, 1) 和 (A+16, 4, 2) 是两个独立的幂等key
```

### 2.4 完整实现

```go
// ============= 幂等检查 =============
// 文件: internal/pkg/idempotent/idempotent.go

const (
    idemPrefix = "wallet:idem:"
    idemTTL    = 24 * time.Hour
)

// IdempotentKey 构造幂等key
func IdempotentKey(bizType int32, refOrderNo string, walletType int32) string {
    return fmt.Sprintf("%s%d:%s:%d", idemPrefix, bizType, refOrderNo, walletType)
}

// CheckAndMark 检查幂等 + 标记占位
// 返回值: (isRepeat bool, cachedResp string)
// isRepeat=true → 重复请求，cachedResp为上次结果
// isRepeat=false → 新请求，已标记占位(value="processing")
func CheckAndMark(ctx context.Context, key string) (bool, string) {
    // 先尝试SetNX占位
    acquired := rds.SetNX(key, "processing", idemTTL)
    if acquired {
        return false, "" // 新请求
    }

    // 已存在，读取缓存结果
    val := rds.Get(key)
    if val == "processing" {
        // 上一次请求还在处理中(极端情况: 另一个请求正在执行)
        // 当前请求应该等待或快速失败
        return true, ""
    }
    return true, val // 已完成的缓存结果
}

// MarkCompleted 请求完成后，将结果写入幂等缓存
func MarkCompleted(ctx context.Context, key string, resultJSON string) {
    rds.Set(ctx, key, resultJSON, idemTTL)
}

// MarkFailed 请求失败后，删除占位(允许重试)
func MarkFailed(ctx context.Context, key string) {
    rds.Del(key)
}
```

**Service层集成：**

```go
func (s *WalletService) CreditWallet(ctx context.Context, req *CreditWalletReq) (*CreditWalletResp, error) {
    // 幂等检查
    idemKey := idempotent.IdempotentKey(req.BizType, req.RefOrderNo, req.WalletType)
    isRepeat, cached := idempotent.CheckAndMark(ctx, idemKey)
    if isRepeat {
        if cached != "" {
            var resp CreditWalletResp
            json.Unmarshal([]byte(cached), &resp)
            return &resp, nil  // 返回缓存结果
        }
        // 正在处理中 → 让调用方稍后重试
        return nil, ret.BizErr(ctx, errs.RequestProcessing, "请求处理中，请勿重复提交")
    }

    // 执行业务逻辑...
    resp, err := s.doCreditWallet(ctx, req)
    if err != nil {
        idempotent.MarkFailed(ctx, idemKey)  // 失败→删除占位，允许重试
        return nil, err
    }

    // 成功→缓存结果
    resultJSON, _ := json.Marshal(resp)
    idempotent.MarkCompleted(ctx, idemKey, string(resultJSON))
    return resp, nil
}
```

### 2.5 幂等范围明细

| 接口 | 幂等Key组成 | 说明 |
|------|------------|------|
| CreditWallet | bizType + refOrderNo + walletType | 同一充值单同一钱包只入一次 |
| DebitWallet | bizType + refOrderNo + walletType | 同一投注单同一钱包只扣一次 |
| FreezeBalance | "freeze" + refOrderNo | 同一提现单只冻一次 |
| UnfreezeBalance | "unfreeze" + refOrderNo | 同一单只解冻一次 |
| ConfirmDebit | "confirm" + refOrderNo | 同一单只确认一次 |
| InitWallet | "init" + userId | 同一用户只初始化一次 |

### 2.6 DB唯一索引兜底

```sql
-- wallet_transaction表
CREATE UNIQUE INDEX uk_order_op_wallet
ON wallet_transaction(order_no, op_type, wallet_type);

-- freeze_detail表
CREATE UNIQUE INDEX uk_ref_order
ON freeze_detail(ref_order_no);
```

**INSERT失败捕获：**

```go
func (r *WalletTxnRepo) InsertFlow(ctx context.Context, tx *query.Query, params *FlowParams) error {
    flow := &model.WalletTransaction{
        OrderNo:  params.OrderNo,
        OpType:   params.OpType,
        // ... 其他字段
    }
    err := tx.WalletTransaction.WithContext(ctx).Create(flow)
    if err != nil {
        // 检测是否是唯一索引冲突
        if isDuplicateKeyError(err) {
            // 幂等保护: 该流水已存在，视为成功
            lg.Log().CtxWarnf(ctx, "幂等拦截(DB层): orderNo=%s opType=%d walletType=%d",
                params.OrderNo, params.OpType, params.WalletType)
            return nil  // 不返回error，让事务继续
        }
        return err
    }
    return nil
}

func isDuplicateKeyError(err error) bool {
    // MySQL错误码1062 = Duplicate entry
    return strings.Contains(err.Error(), "Duplicate entry") ||
           strings.Contains(err.Error(), "1062")
}
```

### 2.7 陷阱与边界

| 陷阱 | 说明 | 应对 |
|------|------|------|
| 失败不清理幂等标记 | CreditWallet执行失败但Redis占位未删 → 后续重试被误拦截 | MarkFailed()删除占位 |
| processing状态残留 | 服务crash在执行中 → 幂等key永远是processing | 24h TTL自动过期；或启动时扫描清理 |
| 不同walletType共用key | 人工加款同时加center和reward → 第二笔被误拦截 | key中包含walletType |
| DB幂等判断为成功但数据不一致 | INSERT被拦截=成功，但account没更新 | 不可能：INSERT和UPDATE在同一事务，要么都成功要么都回滚 |

---

<a id="议题三"></a>
## 议题三 本地事务一致性

### 3.1 问题本质

**余额更新和流水写入必须原子：要么都成功，要么都回滚。**

如果余额扣了但流水没写 → 账务对不上，无法审计。
如果流水写了但余额没扣 → 账务记了但钱没扣，资金漏洞。

**好消息：** 钱包核心数据（wallet_account + wallet_transaction + freeze_detail）在同一个数据库(TiDB)中，只需本地事务，不需要分布式事务。

### 3.2 什么情况需要分布式事务？什么情况不需要？

```
不需要分布式事务的场景 (本地事务即可):
  ├─ CreditWallet: 更新wallet_account + 写wallet_transaction → 同一DB ✓
  ├─ DebitWallet: 同上 ✓
  ├─ FreezeBalance: 更新wallet_account + 写freeze_detail + 写wallet_transaction → 同一DB ✓
  ├─ ExecuteExchange: 更新多行wallet_account + 写多条wallet_transaction → 同一DB ✓
  └─ 所有核心操作都在同一DB → 本地GORM事务 ✓

需要"类分布式"协调的场景 (但不需要2PC/XA):
  ├─ 提现: FreezeBalance(我方) + SubmitWithdrawOrder(ser-finance)
  │        → 不用分布式事务，用TCC模式(详见议题四)
  ├─ 充值: ser-finance回调CreditWallet
  │        → 不用分布式事务，用幂等回调重试(最终一致性)
  └─ 充值补单/人工操作: ser-finance调CreditWallet/DebitWallet
           → 不用分布式事务，用幂等保护
```

### 3.3 事务模板

```go
// ============= 标准事务模板 =============
// 所有钱包写操作必须使用此模板

func (s *WalletService) doXxxOperation(ctx context.Context, params *XxxParams) error {
    return query.Q.Transaction(func(tx *query.Query) error {
        // Step 1: 锁行 (悲观锁)
        account, err := s.accountRepo.GetForUpdate(ctx, tx,
            params.UserId, params.WalletType, params.CurrencyCode)
        if err != nil {
            return err
        }

        // Step 2: 业务校验 (在锁保护下)
        if err := s.validateBusinessRules(ctx, account, params); err != nil {
            return err // ROLLBACK
        }

        // Step 3: 原子更新余额
        affected, err := s.accountRepo.XxxUpdate(ctx, tx, account.ID, params.Amount)
        if err != nil {
            return err // ROLLBACK
        }
        if affected == 0 {
            return ret.BizErr(ctx, errs.ConcurrentConflict, "并发冲突")
        }

        // Step 4: 写不可变流水 (同一事务)
        err = s.txnRepo.InsertFlow(ctx, tx, &FlowParams{...})
        if err != nil {
            return err // ROLLBACK
        }

        // Step 5: [可选] 写冻结明细 / 更新订单状态 (同一事务)
        // ...

        return nil // COMMIT
    })
}
```

### 3.4 事务边界原则

```
事务内只做DB操作:
  ✅ SELECT FOR UPDATE
  ✅ UPDATE wallet_account
  ✅ INSERT wallet_transaction
  ✅ INSERT/UPDATE freeze_detail
  ✅ INSERT/UPDATE deposit_order / withdraw_order / exchange_order

事务外做其他操作:
  ✅ Redis缓存失效 (DEL)
  ✅ 审计日志 (RPC: ser-blog, oneway)
  ✅ 余额推送 (WebSocket)
  ✅ 幂等结果缓存 (Redis SET)

绝不在事务内做:
  ❌ RPC调用 (可能超时，延长事务持有时间)
  ❌ HTTP请求 (同上)
  ❌ 大量Redis操作 (可能阻塞)
  ❌ 复杂计算 (延长锁持有时间)
```

**为什么事务内不能做RPC？**
事务持有FOR UPDATE锁，锁的生命周期=事务时间。如果事务内做RPC调用(3s超时)，那么行锁持有3+秒，所有对同一行的其他请求都要等3+秒。高并发下=灾难。

### 3.5 wallet_transaction的INSERT-ONLY设计

```
为什么流水表是INSERT-ONLY（只插入，不更新，不删除）？

1. 审计安全: 任何人无法篡改已记录的交易流水
2. 无锁竞争: INSERT操作不与其他INSERT竞争(无行锁)
3. 性能优越: 不需要UPDATE，不需要SELECT FOR UPDATE
4. 对账基础: 任意时刻 SUM(流水) 应该 = 当前余额

字段设计关键:
  available_before / available_after  → 余额快照(审计溯源)
  frozen_before / frozen_after        → 冻结快照
  direction (1收入/2支出/3冻结/4解冻) → 明确资金方向
  amount (绝对值)                     → 永远为正数，配合direction使用
```

### 3.6 事务超时控制

```go
// TiDB事务超时设置
// 建议在GORM连接池配置中添加:
db.Exec("SET SESSION innodb_lock_wait_timeout = 5")  // 行锁等待5秒超时
db.Exec("SET SESSION max_execution_time = 5000")     // 单语句5秒超时

// 连接池配置
sqlDB.SetMaxOpenConns(100)     // 最大100连接
sqlDB.SetMaxIdleConns(20)      // 空闲20连接
sqlDB.SetConnMaxLifetime(time.Minute * 5)  // 5分钟回收
```

---

<a id="议题四"></a>
## 议题四 冻结三阶段(TCC)互斥安全

### 4.1 问题本质

提现链路跨越两个服务(ser-wallet + ser-finance)，不能用本地事务保证一致性。采用TCC模式：

```
Try    = FreezeBalance     → 冻结资金(预留)
Confirm = ConfirmDebit     → 确认扣款(最终执行)
Cancel = UnfreezeBalance   → 解冻返还(回滚)

核心约束: Confirm和Cancel对同一refOrderNo互斥，只能执行一个
```

**这不是学术上的TCC，是轻量化的TCC思想应用。我们不引入DTM框架，因为：**
1. 只有提现链路需要TCC，引入框架杀鸡用牛刀
2. 操作只涉及ser-wallet自身DB，协调逻辑由ser-finance控制
3. 用freeze_detail.status状态机即可实现互斥

### 4.2 互斥保障的核心：freeze_detail状态机

```sql
CREATE TABLE freeze_detail (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    currency_code   VARCHAR(10) NOT NULL,
    wallet_type     INT NOT NULL DEFAULT 1,
    amount          DECIMAL(20,4) NOT NULL,
    ref_order_no    VARCHAR(32) NOT NULL,
    status          TINYINT NOT NULL DEFAULT 1,  -- 1=frozen, 2=unfrozen, 3=confirmed
    reason          VARCHAR(255) DEFAULT '',
    create_at       BIGINT NOT NULL,
    update_at       BIGINT NOT NULL,
    UNIQUE KEY uk_ref_order (ref_order_no)  -- 每个订单只有一条冻结记录
);

状态转换:
  Frozen(1) ──→ Unfrozen(2)   [UnfreezeBalance]
  Frozen(1) ──→ Confirmed(3)  [ConfirmDebit]
  Unfrozen(2) ──→ (终态，不可变)
  Confirmed(3) ──→ (终态，不可变)
```

### 4.3 互斥实现核心

```go
// ============= ConfirmDebit 核心逻辑 =============

func (s *WalletService) doConfirmDebit(ctx context.Context, tx *query.Query,
    userId int64, currencyCode string, refOrderNo string,
) error {
    // Step 1: 查冻结记录 + 状态校验
    fd := tx.FreezeDetail
    freeze, err := fd.WithContext(ctx).
        Where(fd.RefOrderNo.Eq(refOrderNo)).
        First()
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return ret.BizErr(ctx, errs.FreezeRecordNotFound, "冻结记录不存在")
        }
        return err
    }

    // ★ 互斥核心检查 ★
    switch freeze.Status {
    case FreezeStatusFrozen: // 1 → 正常，继续执行
        break
    case FreezeStatusConfirmed: // 3 → 幂等：已确认，直接返回成功
        return nil
    case FreezeStatusUnfrozen: // 2 → 冲突：已被解冻，拒绝确认
        return ret.BizErr(ctx, errs.FreezeAlreadyUnfrozen,
            "冻结已被解冻，无法确认扣款")
    }

    // Step 2: 原子更新冻结状态 (WHERE status=1 防并发)
    result, err := fd.WithContext(ctx).
        Where(fd.RefOrderNo.Eq(refOrderNo)).
        Where(fd.Status.Eq(FreezeStatusFrozen)). // ★ 关键: 只有status=1才能更新
        UpdateSimple(
            fd.Status.Value(FreezeStatusConfirmed),
            fd.UpdateAt.Value(time.Now().UnixMilli()),
        )
    if err != nil {
        return err
    }
    if result.RowsAffected == 0 {
        // 并发: 在Step1和Step2之间，另一个请求已将status改为2
        return ret.BizErr(ctx, errs.FreezeConcurrentConflict,
            "冻结状态已变更(并发)")
    }

    // Step 3: 扣减冻结金额 (frozen_amt -= amount)
    w := tx.WalletAccount
    _, err = w.WithContext(ctx).
        Where(w.UserID.Eq(userId)).
        Where(w.WalletType.Eq(freeze.WalletType)).
        Where(w.CurrencyCode.Eq(currencyCode)).
        Where(w.FrozenAmt.Gte(freeze.Amount)). // 安全条件
        UpdateSimple(
            w.FrozenAmt.Sub(freeze.Amount),
        )
    if err != nil {
        return err
    }

    // Step 4: 写流水
    return s.txnRepo.InsertFlow(ctx, tx, &FlowParams{
        Direction: DirectionExpense, // 2=支出(最终扣款)
        OpType:    BizTypeWithdrawConfirm, // 13
        // ... 其他字段
    })
}
```

**UnfreezeBalance的互斥逻辑完全对称：**

```go
// 互斥检查
switch freeze.Status {
case FreezeStatusFrozen:    // 1 → 正常
case FreezeStatusUnfrozen:  // 2 → 幂等返回
    return nil
case FreezeStatusConfirmed: // 3 → 冲突拒绝
    return ret.BizErr(ctx, errs.FreezeAlreadyConfirmed, "已确认扣款，无法解冻")
}

// UPDATE ... WHERE status = 1
// + available_amt += freeze.Amount, frozen_amt -= freeze.Amount
```

### 4.4 为什么不用数据库乐观锁(Version)做互斥？

```
Version方案:  freeze_detail增加version字段
  → Confirm: UPDATE WHERE version=1, SET status=3, version=2
  → Cancel:  UPDATE WHERE version=1, SET status=2, version=2
  → affected_rows=0 → 冲突

status方案(我们的选择):
  → Confirm: UPDATE WHERE status=1, SET status=3
  → Cancel:  UPDATE WHERE status=1, SET status=2
  → affected_rows=0 → 冲突

两者等效! 但status方案更直观:
  - status本身就是业务语义，不需要额外字段
  - 运维排查时看status=2/3就知道发生了什么
  - version只是个数字，没有业务含义
```

### 4.5 空回滚与悬挂防护

```
空回滚: Cancel被调用，但Try从未执行成功
  → 场景: FreezeBalance超时，ser-finance调UnfreezeBalance回滚
  → freeze_detail不存在 → 查不到记录
  → 处理: 返回成功(幂等语义)，记录warn日志

悬挂: Try比Cancel后到达
  → 场景: FreezeBalance网络延迟，Cancel先到并删除，FreezeBalance后到又创建
  → 处理: freeze_detail有唯一索引uk_ref_order
  → UnfreezeBalance先到 → 插入一条status=2的记录(标记已取消)
  → FreezeBalance后到 → INSERT冲突 → 检测到已取消 → 拒绝冻结

  实现方式(防悬挂):
```

```go
// UnfreezeBalance: 如果冻结记录不存在，插入一条status=2的"空回滚"记录
func (s *WalletService) doUnfreezeBalance(ctx context.Context, tx *query.Query, ...) error {
    freeze, err := s.freezeRepo.GetByRefOrderNo(ctx, tx, refOrderNo)
    if err != nil && errors.Is(err, gorm.ErrRecordNotFound) {
        // 空回滚: 插入已取消标记，防止后续Try悬挂
        return s.freezeRepo.InsertCancelledMark(ctx, tx, &model.FreezeDetail{
            RefOrderNo: refOrderNo,
            Status:     FreezeStatusUnfrozen, // 2
            Reason:     "空回滚标记",
            // 其他字段为零值
        })
    }
    // ... 正常解冻逻辑
}

// FreezeBalance: INSERT时如果冲突，检查是否为空回滚标记
func (s *WalletService) doFreezeBalance(ctx context.Context, tx *query.Query, ...) error {
    err := s.freezeRepo.InsertFreeze(ctx, tx, freeze)
    if isDuplicateKeyError(err) {
        // 检查已有记录状态
        existing, _ := s.freezeRepo.GetByRefOrderNo(ctx, tx, refOrderNo)
        if existing != nil && existing.Status == FreezeStatusUnfrozen {
            // 悬挂: Cancel已先到，拒绝冻结
            return ret.BizErr(ctx, errs.FreezeRejectedByCancel,
                "提现已取消，冻结被拒绝")
        }
        // 幂等: 已经冻结过了
        return nil
    }
    return err
}
```

---

<a id="议题五"></a>
## 议题五 金额精度与计算安全

### 5.1 问题本质

```
float64的致命缺陷:
  0.1 + 0.2 = 0.30000000000000004  (IEEE 754)

钱包影响:
  用户充值 0.1 VND，查余额显示 0.30000000000000004 → 用户投诉
  百万次交易累积误差 → 对账差异 → 财务审计不过
```

### 5.2 方案设计：全链路精度控制

```
存储层:     DECIMAL(20,4) / DECIMAL(20,8)   → DB原生精度
传输层:     i64最小单位 / string小数          → 无精度损失
计算层:     shopspring/decimal               → 任意精度
展示层:     按currency_config.decimal格式化   → 用户友好

全链路没有任何环节使用float64
```

### 5.3 存储层规范

```sql
-- 金额字段: DECIMAL(20,4)
-- 总共20位，小数4位，整数最大16位 = 9999999999999999.9999
-- 足够覆盖100万亿级别金额
available_amt   DECIMAL(20,4) NOT NULL DEFAULT 0.0000,
frozen_amt      DECIMAL(20,4) NOT NULL DEFAULT 0.0000,

-- 汇率字段: DECIMAL(20,8)
-- 8位小数覆盖绝大多数汇率精度需求
-- 例: 1 USDT = 25432.12345678 VND
platform_rate   DECIMAL(20,8) NOT NULL DEFAULT 0.00000000,
deposit_rate    DECIMAL(20,8) NOT NULL DEFAULT 0.00000000,

-- 手续费率: DECIMAL(10,4)
-- 例: 0.0200 = 2%
fee_rate        DECIMAL(10,4) NOT NULL DEFAULT 0.0000,
```

### 5.4 传输层规范

```thrift
// Thrift IDL 金额传输约定:

// 方案A: i64最小单位 (推荐用于RPC)
// 优势: 整数传输零精度损失，高性能
// 劣势: 需要前后端约定decimal位数进行换算
struct CreditWalletReq {
    1: required i64 amount,           // 单位: 最小精度单位(如VND=分, BSB=0.0001)
    2: required string currencyCode,  // 配合currency_config.decimal换算
}

// 方案B: string小数 (推荐用于HTTP/前端)
// 优势: 人类可读，调试方便
// 劣势: 解析需要多一步
struct ExchangePreviewResp {
    1: required string fromAmount,    // "100.0000"
    2: required string toAmount,      // "0.0039"
    3: required string rate,          // "25432.12345678"
}
```

**换算规则：**

```go
// i64 → decimal
func Int64ToDecimal(val int64, decimalPlaces int32) decimal.Decimal {
    return decimal.NewFromInt(val).Shift(-decimalPlaces)
    // 例: val=100000, decimal=4 → 10.0000
}

// decimal → i64
func DecimalToInt64(val decimal.Decimal, decimalPlaces int32) int64 {
    return val.Shift(decimalPlaces).IntPart()
    // 例: val=10.0000, decimal=4 → 100000
}
```

### 5.5 计算层规范

```go
import "github.com/shopspring/decimal"

// ============= 金额计算安全规范 =============

// ❌ 绝对禁止
var a float64 = 0.1
var b float64 = 0.2
c := a + b // 0.30000000000000004

// ✅ 必须使用
a := decimal.NewFromString("0.1")
b := decimal.NewFromString("0.2")
c := a.Add(b) // 精确 0.3

// ============= 兑换计算示例 =============

func CalculateExchange(fiatAmount string, depositRate string, bsbAnchorRatio string) (
    exchangeAmount decimal.Decimal, err error,
) {
    fiat, err := decimal.NewFromString(fiatAmount)
    if err != nil {
        return decimal.Zero, fmt.Errorf("无效金额: %s", fiatAmount)
    }
    rate, err := decimal.NewFromString(depositRate)
    if err != nil {
        return decimal.Zero, fmt.Errorf("无效汇率: %s", depositRate)
    }
    anchor, err := decimal.NewFromString(bsbAnchorRatio)
    if err != nil {
        return decimal.Zero, fmt.Errorf("无效锚定比: %s", bsbAnchorRatio)
    }

    if anchor.IsZero() {
        return decimal.Zero, errors.New("锚定比不能为零")
    }

    // exchangeAmount = fiatAmount × depositRate / bsbAnchorRatio
    // 使用DivRound指定精度，避免无限小数
    exchangeAmount = fiat.Mul(rate).DivRound(anchor, 8) // 8位小数精度

    return exchangeAmount, nil
}

// ============= 手续费计算示例 =============

func CalculateFee(amount string, feeRate string) (feeAmount, actualAmount decimal.Decimal, err error) {
    amt, _ := decimal.NewFromString(amount)
    rate, _ := decimal.NewFromString(feeRate)

    feeAmount = amt.Mul(rate).RoundFloor(4)  // 向下取整(对用户有利)
    actualAmount = amt.Sub(feeAmount)

    return feeAmount, actualAmount, nil
}
```

### 5.6 核心纪律

```
金额计算八条铁律:

1. 永远从string创建decimal，绝不从float64创建
   ✅ decimal.NewFromString("19.99")
   ❌ decimal.NewFromFloat(19.99)

2. 永远用.Equal()比较，绝不用==
   ✅ a.Equal(b)
   ❌ a == b  // 结构体比较，可能误判

3. 除法必须指定精度
   ✅ a.DivRound(b, 8)
   ❌ a.Div(b)  // 可能产生无限循环小数

4. 除法前必须检查除数非零
   if divisor.IsZero() { return error }

5. 金额必须≥0，入口处校验
   if amount.LessThanOrEqual(decimal.Zero) { return error }

6. DB存储用DECIMAL类型，GORM自动映射
   Balance decimal.Decimal `gorm:"type:decimal(20,4)"`

7. JSON序列化默认带引号(string)，不要改为unquoted
   "amount": "19.9900"  ✅
   "amount": 19.99      ❌ (JS解析可能丢精度)

8. 所有中间计算保持最高精度，仅在最终输出时truncate/round
```

---

<a id="议题六"></a>
## 议题六 死锁预防

### 6.1 问题本质

```
场景: 兑换操作在同一事务内需要操作3行:
  行A: wallet_account (userId=1, walletType=1, currency=VND)   → 扣法币
  行B: wallet_account (userId=1, walletType=1, currency=BSB)   → 加BSB
  行C: wallet_account (userId=1, walletType=2, currency=BSB)   → 加赠金

如果两个并发兑换请求:
  请求1: 锁A → 锁B → 锁C
  请求2: 锁B → 锁A → (等待A → 死锁!)

MySQL/TiDB检测到死锁后会自动回滚其中一个事务(ROLLBACK)，
但我们不应该依赖这个机制——死锁=性能损失+用户体验差。
```

### 6.2 方案设计：固定加锁顺序

```
规则: 所有多行加锁操作，按照 (walletType ASC, currencyCode ASC) 的固定顺序加锁。

兑换示例:
  行A: (walletType=1, currency=BSB)  → 先锁
  行B: (walletType=1, currency=VND)  → 后锁
  行C: (walletType=2, currency=BSB)  → 最后锁

按此顺序:
  请求1: 锁A → 锁B → 锁C ✓
  请求2: 锁A(等待1释放) → 锁B → 锁C ✓
  永远不会出现循环等待 = 不会死锁
```

### 6.3 实现

```go
// ============= 多行加锁工具 =============
// 文件: internal/pkg/lockorder/lockorder.go

// LockTarget 加锁目标
type LockTarget struct {
    WalletType   int32
    CurrencyCode string
}

// SortLockTargets 按固定顺序排序加锁目标
// 规则: walletType ASC → currencyCode ASC
func SortLockTargets(targets []LockTarget) {
    sort.Slice(targets, func(i, j int) bool {
        if targets[i].WalletType != targets[j].WalletType {
            return targets[i].WalletType < targets[j].WalletType
        }
        return targets[i].CurrencyCode < targets[j].CurrencyCode
    })
}

// LockMultipleAccounts 按固定顺序锁定多个钱包行
// 返回按加锁顺序排列的account列表
func (r *WalletAccountRepo) LockMultipleAccounts(
    ctx context.Context,
    tx *query.Query,
    userId int64,
    targets []LockTarget,
) ([]*model.WalletAccount, error) {
    // 排序
    SortLockTargets(targets)

    accounts := make([]*model.WalletAccount, 0, len(targets))
    for _, t := range targets {
        acc, err := r.GetForUpdate(ctx, tx, userId, t.WalletType, t.CurrencyCode)
        if err != nil {
            return nil, err
        }
        accounts = append(accounts, acc)
    }
    return accounts, nil
}
```

**兑换Service使用：**

```go
func (s *WalletService) doExecuteExchange(ctx context.Context, params *ExchangeParams) error {
    return query.Q.Transaction(func(tx *query.Query) error {
        // 定义需要锁的行
        targets := []lockorder.LockTarget{
            {WalletType: 1, CurrencyCode: params.FromCurrency}, // 法币中心
            {WalletType: 1, CurrencyCode: "BSB"},               // BSB中心
        }
        if params.BonusAmount.GreaterThan(decimal.Zero) {
            targets = append(targets, lockorder.LockTarget{
                WalletType: 2, CurrencyCode: "BSB",            // BSB奖励
            })
        }

        // 按固定顺序加锁
        accounts, err := s.accountRepo.LockMultipleAccounts(ctx, tx, params.UserId, targets)
        if err != nil {
            return err
        }

        // 后续操作按accounts顺序处理...
        // accounts[0] = 排序后第一个 → 操作
        // accounts[1] = 排序后第二个 → 操作
        // ...

        return nil
    })
}
```

### 6.4 陷阱

| 陷阱 | 说明 | 应对 |
|------|------|------|
| 只有兑换需要多行锁 | CreditWallet/DebitWallet只操作1行，无死锁风险 | 单行操作不需要排序 |
| Redis锁粒度与DB锁不一致 | Redis锁是per-walletType-per-currency，兑换涉及多个 | 兑换场景获取多个Redis锁(也要固定顺序) |
| 用户同时兑换和提现 | 兑换锁(center,VND) + 提现锁(center,VND) | 同一行=自然串行(FOR UPDATE等待) |

---

<a id="议题七"></a>
## 议题七 缓存一致性

### 7.1 问题本质

```
缓存不一致的后果:
  用户充值100元 → DB余额200 → 但Redis缓存还是100 → 用户看到余额100 → 投诉

比普通业务更敏感:
  Banner缓存不一致 → 用户看到旧Banner → 无所谓
  余额缓存不一致 → 用户看到旧余额 → 恐慌("我的钱去哪了？")
```

### 7.2 方案设计：写后失效 + 短TTL双保障

```
策略: Cache-Aside (旁路缓存)
  读: 先查缓存 → 命中返回 → 未命中查DB → 写缓存
  写: 先写DB(事务) → 事务提交后立即删缓存

不采用: Cache-Through / Write-Behind
  原因: 钱包数据一致性要求极高，不能异步/延迟写入
```

### 7.3 余额缓存实现

```go
// ============= 余额缓存 =============
// 文件: internal/cache/balance.go

const (
    balancePrefix = "wallet:bal:"
    balanceTTL    = 30 * time.Second  // 30秒短TTL
)

func balanceKey(userId int64, currencyCode string) string {
    return fmt.Sprintf("%s%d:%s", balancePrefix, userId, currencyCode)
}

// GetBalance 查询缓存
// 返回nil表示未命中
func (c *BalanceCache) GetBalance(ctx context.Context, userId int64, currencyCode string) *BalanceDTO {
    data := rds.Get(balanceKey(userId, currencyCode))
    if data == "" {
        return nil // 未命中
    }
    var dto BalanceDTO
    if err := json.Unmarshal([]byte(data), &dto); err != nil {
        return nil // 反序列化失败=视为未命中
    }
    return &dto
}

// SetBalance 写入缓存(空结果也缓存防穿透)
func (c *BalanceCache) SetBalance(ctx context.Context, userId int64, currencyCode string, dto *BalanceDTO) {
    data, _ := json.Marshal(dto)
    rds.Set(ctx, balanceKey(userId, currencyCode), string(data), balanceTTL)
}

// InvalidateBalance 失效缓存
// 在所有写操作(Credit/Debit/Freeze/Unfreeze/Confirm/Exchange)后调用
func (c *BalanceCache) InvalidateBalance(ctx context.Context, userId int64, currencyCode string) {
    rds.Del(balanceKey(userId, currencyCode))
}
```

### 7.4 为什么是删缓存而不是更新缓存？

```
更新缓存的问题:
  请求A: DB余额200 → 准备更新缓存为200
  请求B: DB余额300 → 更新缓存为300
  请求A: 更新缓存为200 (延迟到达)
  → 缓存=200, DB=300 → 不一致!

删缓存的优势:
  请求A: DB余额200 → 删缓存
  请求B: DB余额300 → 删缓存
  下次读 → 缓存未命中 → 查DB=300 → 写缓存300
  → 永远一致!
```

### 7.5 为什么TTL是30秒？

```
TTL太长(如5分钟):
  写操作后删缓存失败(Redis网络抖动) → 5分钟内用户看到旧余额 → 不可接受

TTL太短(如1秒):
  缓存命中率极低 → 等于没有缓存 → DB压力大

30秒平衡点:
  正常情况: 写后删缓存成功 → 下次读从DB加载最新值 → 缓存30秒
  异常情况: 删缓存失败 → 最多30秒后TTL过期 → 自动从DB重载
  缓存命中率: 余额查询频率远高于写操作 → 30秒内多次读取只查1次DB → 足够
```

### 7.6 币种和汇率缓存

```go
// ============= 币种缓存(长TTL) =============
const currencyTTL = 5 * time.Minute  // 5分钟

// 币种配置变更极少(B端操作)，5分钟TTL足够
// B端修改后主动清除:
func (c *CurrencyCache) InvalidateAll(ctx context.Context) {
    keys := []string{
        "wallet:currency:list:enabled",
        "wallet:currency:list:all",
    }
    rds.Del(keys...)
}

// ============= 汇率缓存(与刷新任务同步) =============
const rateTTL = 10 * time.Minute  // 10分钟(> 刷新周期5分钟)

// 汇率刷新任务更新DB后，同步更新缓存:
func (c *RateCache) SetRate(ctx context.Context, currencyCode string, rate *RateDTO) {
    data, _ := json.Marshal(rate)
    rds.Set(ctx, "wallet:rate:"+currencyCode, string(data), rateTTL)
}
```

### 7.7 缓存穿透防护

```go
// 空结果也缓存
func (s *WalletService) GetBalance(ctx context.Context, userId int64, currencyCode string) (*BalanceDTO, error) {
    cached := s.cache.GetBalance(ctx, userId, currencyCode)
    if cached != nil {
        return cached, nil // 命中(可能是空结果)
    }

    // 查DB
    accounts, err := s.accountRepo.ListByUserAndCurrency(ctx, userId, currencyCode)
    if err != nil {
        return nil, err
    }

    dto := assembleBalanceDTO(accounts) // 可能是全零的DTO
    s.cache.SetBalance(ctx, userId, currencyCode, dto) // 空结果也缓存
    return dto, nil
}
```

---

<a id="议题八"></a>
## 议题八 状态机正确性保障

### 8.1 问题本质

```
钱包有3个核心状态机:
  1. 充值订单: Pending → Paying → Credited/Failed/Closed
  2. 提现订单: Created → Frozen → RiskOK → Paying → Completed/Rejected/PayoutFailed
  3. 冻结明细: Frozen → Unfrozen/Confirmed

如果状态跳转没有严格约束:
  Paying直接跳到Closed(跳过了Failed) → 数据不一致
  Completed又跳回Paying → 已出款的钱又变成"出款中" → 灾难
```

### 8.2 方案设计：状态转换白名单 + DB原子条件更新

```go
// ============= 状态机定义 =============
// 文件: internal/enum/order_status.go

// 提现订单状态
const (
    WithdrawStatusCreated     int32 = 0
    WithdrawStatusFrozen      int32 = 1
    WithdrawStatusRiskOK      int32 = 2
    WithdrawStatusPaying      int32 = 3
    WithdrawStatusCompleted   int32 = 4
    WithdrawStatusRejected    int32 = 5
    WithdrawStatusPayoutFailed int32 = 6
)

// 合法状态转换白名单
// key=当前状态, value=允许转换到的目标状态列表
var withdrawTransitions = map[int32][]int32{
    WithdrawStatusCreated:      {WithdrawStatusFrozen},
    WithdrawStatusFrozen:       {WithdrawStatusRiskOK, WithdrawStatusRejected},
    WithdrawStatusRiskOK:       {WithdrawStatusPaying, WithdrawStatusRejected},
    WithdrawStatusPaying:       {WithdrawStatusCompleted, WithdrawStatusPayoutFailed},
    WithdrawStatusCompleted:    {},  // 终态，不可变
    WithdrawStatusRejected:     {},  // 终态
    WithdrawStatusPayoutFailed: {},  // 终态
}

// CanTransition 校验状态转换是否合法
func CanTransition(current, target int32, transitions map[int32][]int32) bool {
    allowed, exists := transitions[current]
    if !exists {
        return false
    }
    for _, s := range allowed {
        if s == target {
            return true
        }
    }
    return false
}
```

**DB层双保险：**

```go
// ============= 状态更新必须带WHERE条件 =============

func (r *WithdrawOrderRepo) UpdateStatus(
    ctx context.Context,
    tx *query.Query,
    orderNo string,
    fromStatus int32,  // 当前期望状态
    toStatus int32,    // 目标状态
) error {
    // 应用层校验
    if !CanTransition(fromStatus, toStatus, withdrawTransitions) {
        return fmt.Errorf("非法状态转换: %d → %d", fromStatus, toStatus)
    }

    // DB层原子更新(WHERE status = fromStatus)
    wo := tx.WithdrawOrder
    result, err := wo.WithContext(ctx).
        Where(wo.OrderNo.Eq(orderNo)).
        Where(wo.Status.Eq(fromStatus)). // ★ 关键: 只有当前状态匹配才更新
        UpdateSimple(
            wo.Status.Value(toStatus),
            wo.UpdateAt.Value(time.Now().UnixMilli()),
        )
    if err != nil {
        return err
    }
    if result.RowsAffected == 0 {
        return ret.BizErr(ctx, errs.OrderStatusConflict,
            fmt.Sprintf("订单状态已变更，预期%d实际已不是", fromStatus))
    }
    return nil
}
```

### 8.3 为什么两层校验？

```
仅应用层校验:
  → 并发情况下，两个请求都通过校验，都执行UPDATE
  → 一个成功将status从1→2，另一个也成功将status从2→5(跳过了3,4)
  → 因为第二个请求读到的status是1(校验时)，但UPDATE时已经是2了
  → ❌ 不安全

仅DB层校验(WHERE status = fromStatus):
  → 安全，但错误信息不友好(affected_rows=0不知道原因)
  → 每次都走DB查询

两层结合:
  → 应用层: 快速拒绝明显非法的转换(如terminal→anything)
  → DB层: WHERE条件原子保证并发安全
  → ✅ 安全+友好
```

---

<a id="议题九"></a>
## 议题九 资金安全纵深防线

### 9.1 四层防线体系

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: 入口防线 (恶意请求拦截)                                  │
│   ├─ IP黑名单 (BlackCheck)                                      │
│   ├─ Token校验 (FontAuth/BackAuth)                               │
│   ├─ 用户状态校验 (封禁用户禁止操作)                                │
│   ├─ KYC校验 (未实名限制提现渠道)                                  │
│   └─ 频率限制 (同一用户同一操作1秒内不能重复)                        │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: 业务逻辑防线 (业务规则校验)                                │
│   ├─ 金额校验: amount > 0                                        │
│   ├─ 余额校验: available_amt >= debit_amount                     │
│   ├─ 限额校验: 单笔/日/月限额                                     │
│   ├─ 姓名匹配: 提现账户名 == KYC实名                               │
│   ├─ 币种校验: 币种存在且已启用                                     │
│   └─ 状态机校验: 状态转换合法性                                     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: 数据层防线 (技术机制保障)                                  │
│   ├─ SQL原子条件: UPDATE WHERE available_amt >= amount             │
│   ├─ 悲观锁: SELECT FOR UPDATE                                    │
│   ├─ 唯一索引幂等: uk_order_op_wallet                              │
│   ├─ TCC互斥: freeze_detail.status状态机                          │
│   └─ 流水不可变: wallet_transaction INSERT-ONLY                    │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: 事后防线 (发现与修复)                                     │
│   ├─ 余额对账: 定期 SUM(流水) vs 当前余额                          │
│   ├─ 异常告警: 负余额/冻结超24h未处理                               │
│   ├─ 审计日志: ser-blog全操作记录                                   │
│   └─ 流水溯源: traceId贯穿全链路                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 余额对账方案

```go
// ============= 定时对账任务 =============
// 每日凌晨执行，对比余额与流水

func (s *ReconciliationService) DailyReconcile(ctx context.Context) error {
    // 查所有有余额的账户
    accounts, err := s.accountRepo.ListNonZero(ctx)
    if err != nil {
        return err
    }

    for _, acc := range accounts {
        // 计算: 流水净额 = SUM(收入) - SUM(支出)
        netFlow, err := s.txnRepo.SumNetFlow(ctx, acc.UserID, acc.WalletType, acc.CurrencyCode)
        if err != nil {
            lg.Log().CtxErrorf(ctx, "对账查询流水失败: userId=%d err=%v", acc.UserID, err)
            continue
        }

        // 当前余额 = available + frozen
        currentBalance := acc.AvailableAmt.Add(acc.FrozenAmt)

        // 对比
        if !currentBalance.Equal(netFlow) {
            // ★ 严重告警: 余额与流水不一致
            diff := currentBalance.Sub(netFlow)
            lg.Log().CtxErrorf(ctx,
                "⚠️ 对账异常! userId=%d walletType=%d currency=%s "+
                    "balance=%s flowNet=%s diff=%s",
                acc.UserID, acc.WalletType, acc.CurrencyCode,
                currentBalance.String(), netFlow.String(), diff.String())

            // 发送告警通知(钉钉/Slack/邮件)
            s.alertService.SendReconcileAlert(ctx, acc, currentBalance, netFlow, diff)
        }
    }
    return nil
}
```

**流水净额SQL：**

```sql
SELECT
    COALESCE(SUM(CASE WHEN direction = 1 THEN amount ELSE 0 END), 0) -- 收入
    - COALESCE(SUM(CASE WHEN direction = 2 THEN amount ELSE 0 END), 0) -- 支出
    AS net_flow
FROM wallet_transaction
WHERE user_id = ? AND wallet_type = ? AND currency_code = ?
```

### 9.3 负余额监控

```go
// 实时监控: 每次余额更新后检查
func (s *WalletService) afterBalanceUpdate(ctx context.Context, account *model.WalletAccount) {
    if account.AvailableAmt.LessThan(decimal.Zero) {
        // ★ 极其严重: 余额为负，说明防超卖机制有漏洞
        lg.Log().CtxErrorf(ctx,
            "🔴 负余额告警! userId=%d walletType=%d currency=%s available=%s",
            account.UserID, account.WalletType, account.CurrencyCode,
            account.AvailableAmt.String())
        s.alertService.SendNegativeBalanceAlert(ctx, account)
    }
}
```

---

<a id="议题十"></a>
## 议题十 高频接口性能优化

### 10.1 热点分析

```
频率排行:
  Top 1: GetBalance (余额查询)     → 每次打开App/下注/结算都触发
  Top 2: DebitWallet (投注扣款)    → 每笔投注
  Top 3: CreditWallet (结算返奖)   → 每笔中奖
  Top 4: GetCurrencyList (币种查)  → 各模块通用
  Top 5: GetExchangeRate (汇率查)  → 兑换/充值/提现

优化核心原则: 读操作走缓存，写操作最小化锁持有时间
```

### 10.2 读操作优化

```
余额查询 (Top 1):
  ├─ Redis缓存(30s TTL, 写后失效)
  ├─ 缓存命中率预估 > 90% (读远多于写)
  ├─ DB查询有索引: (user_id, wallet_type, currency_code) UNIQUE
  └─ 预估: 缓存命中15ms, 未命中25ms

币种列表 (Top 4):
  ├─ Redis缓存(5min TTL)
  ├─ 数据量小(≤10条), 全量缓存
  └─ 预估: 缓存命中10ms

汇率查询 (Top 5):
  ├─ Redis缓存(与刷新任务同步, 10min TTL)
  └─ 预估: 缓存命中10ms
```

### 10.3 写操作优化

```
投注扣款 (Top 2, 最关键):

  不该做的优化:
    ❌ 余额放Redis(内存)做扣减 → Redis宕机=数据丢失
    ❌ 异步写DB → 崩溃=丢交易
    ❌ 去掉FOR UPDATE → 超卖

  应该做的优化:
    ✅ Redis分布式锁拦截99%并发(不打DB)
    ✅ FOR UPDATE行级锁(不是表锁), 只锁1行
    ✅ wallet_transaction INSERT-ONLY(无锁竞争)
    ✅ 事务内不做RPC(最小化锁持有时间)
    ✅ 事务外做缓存失效/审计日志(异步不阻塞)

  预估:
    单笔扣款: ~30ms (含Redis锁+DB事务)
    单用户TPS: 受限于串行化, ~30 TPS/用户
    全局TPS: 不同用户并行, 受限于DB连接池, ~3000 TPS
```

### 10.4 wallet_transaction大表策略

```
预估数据量:
  每用户每天10笔交易(投注/返奖/充值)
  10万DAU × 10笔 = 100万条/天
  100万 × 365 = 3.65亿条/年

TiDB优势:
  ├─ 自动分片, 亿级表无需手动分表
  ├─ 写入性能不随数据量线性下降(Region分裂)
  └─ 索引查询O(logN), 3亿条≈28次IO, <10ms

查询优化:
  ├─ 必须带 user_id 条件(命中索引)
  ├─ 必须带 create_at 时间范围(减少扫描行)
  ├─ 分页查询: LIMIT + OFFSET (小页, 每页20条)
  └─ 未来: 按月归档到 wallet_transaction_archive_YYYYMM
```

---

<a id="议题十一"></a>
## 议题十一 跨模块故障隔离与降级

### 11.1 依赖分类

```
强依赖 (故障=链路阻断):
  ├─ TiDB: 所有写操作
  ├─ Redis: 分布式锁, 幂等, 缓存
  └─ ser-finance: 充值/提现链路 (TODO, 尚未开发)

弱依赖 (故障=降级不阻断):
  ├─ ser-blog: 审计日志 (oneway, 失败忽略)
  ├─ ser-s3: 图标上传 (失败返回空URL)
  ├─ ser-user: 用户状态查询 (提现前置检查)
  ├─ ser-kyc: KYC查询 (提现前置检查)
  ├─ ser-ip: IP校验 (C端fail-open)
  └─ 三方汇率API: 汇率刷新 (保持旧值)
```

### 11.2 降级策略实现

```go
// ============= 弱依赖降级模板 =============

// 审计日志: 失败不阻塞
func (s *WalletService) asyncAuditLog(ctx context.Context, action string, req interface{}) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                lg.Log().CtxErrorf(ctx, "审计日志panic: %v", r)
            }
        }()
        // oneway调用, 不等响应
        if err := rpc.BLogClient().AddActLog(ctx, buildLogReq(action, req)); err != nil {
            lg.Log().CtxWarnf(ctx, "审计日志写入失败(不影响业务): %v", err)
        }
    }()
}

// 用户状态查询: 超时降级
func (s *WalletService) checkUserStatus(ctx context.Context, userId int64) error {
    // 设置短超时context
    timeoutCtx, cancel := context.WithTimeout(ctx, 1*time.Second)
    defer cancel()

    userInfo, err := rpc.UserClient().GetUserInfoExpend(timeoutCtx, userId)
    if err != nil {
        // 降级: 无法确认用户状态 → 记录warn, 允许继续
        lg.Log().CtxWarnf(ctx, "用户状态查询失败(降级放行): userId=%d err=%v", userId, err)
        return nil // 不阻塞
    }

    if userInfo.Status == UserStatusBlocked {
        return ret.BizErr(ctx, errs.UserBlocked, "账户已被封禁")
    }
    return nil
}
```

### 11.3 Redis故障降级

```go
// Redis不可用时的降级策略

func (s *WalletService) DebitWalletWithDegradation(ctx context.Context, req *DebitWalletReq) (*DebitWalletResp, error) {
    // 尝试获取Redis分布式锁
    lockKey := dlock.WalletLockKey(req.UserId, req.WalletType, req.CurrencyCode)
    acquired := rds.SetNX(lockKey, "...", dlock.LockTTL)

    if !acquired {
        // 可能是竞争，也可能是Redis不可用
        // 检测Redis是否可用
        if !rds.Ping() {
            // Redis不可用 → 降级: 跳过Redis锁，仅依赖DB锁
            lg.Log().CtxWarnf(ctx, "Redis不可用，降级为仅DB锁: userId=%d", req.UserId)
            return s.doDebitWalletDBOnly(ctx, req) // 只用FOR UPDATE
        }
        // Redis正常，是真正的锁竞争
        return nil, ret.BizErr(ctx, errs.WalletOperationBusy, "操作频繁")
    }

    defer rds.Eval(ctx, unlockScript, []string{lockKey}, "...")
    return s.doDebitWallet(ctx, req)
}
```

### 11.4 ser-finance未就绪的Mock策略

```go
// ser-finance接口尚未开发，使用TODO标记 + Mock数据

func (s *DepositService) GetDepositMethods(ctx context.Context, currencyCode string, userId int64) ([]*DepositMethod, error) {
    // TODO: 替换为真实RPC调用
    // resp, err := rpc.FinanceClient().GetDepositMethods(ctx, &finance.GetDepositMethodsReq{...})
    // if err != nil { return nil, err }

    // Mock数据(开发阶段)
    return []*DepositMethod{
        {MethodId: 1, Name: "银行卡", MinAmount: "100", MaxAmount: "50000"},
        {MethodId: 2, Name: "电子钱包", MinAmount: "50", MaxAmount: "20000"},
    }, nil
}
```

---

<a id="议题十二"></a>
## 议题十二 审计可追溯性

### 12.1 双重审计体系

```
Layer 1: wallet_transaction (业务流水)
  ├─ 记录每一笔资金变动
  ├─ INSERT-ONLY, 不可篡改
  ├─ 字段: order_no, op_type, direction, amount,
  │        available_before/after, frozen_before/after
  └─ 用途: 对账, 资金溯源, 余额重建

Layer 2: ser-blog (操作日志)
  ├─ 记录谁做了什么操作
  ├─ 通过RPC(oneway)写入
  ├─ 字段: operator, action, target, result, traceId
  └─ 用途: 操作审计, 责任追溯
```

### 12.2 traceId贯穿全链路

```
一笔充值的traceId传播:

用户HTTP请求
  → gate-font生成 traceId=abc-123
  → metainfo.WithPersistentValue(ctx, "X-Trace-Id", "abc-123")
  → RPC传播到 ser-wallet (TTHeader)
  → ser-wallet日志: <abc-123> CreditWallet userId=1 amount=100
  → wallet_transaction.trace_id = "abc-123" (持久化)
  → ser-blog日志: traceId=abc-123, action="充值入账"
  → 响应: {traceId: "abc-123", code: 0}

事后排查:
  用户投诉"充值没到账"
  → 用户提供traceId (或从前端日志提取)
  → grep traceId → 完整还原请求链路
```

### 12.3 关键日志规范

```go
// 所有钱包写操作必须记录结构化日志

// 成功日志
lg.Log().CtxInfof(ctx,
    "CreditWallet成功 userId=%d currency=%s walletType=%d "+
        "amount=%s bizType=%d refOrderNo=%s before=%s after=%s",
    req.UserId, req.CurrencyCode, req.WalletType,
    amount.String(), req.BizType, req.RefOrderNo,
    account.AvailableAmt.String(), newAvailable.String())

// 失败日志
lg.Log().CtxErrorf(ctx,
    "DebitWallet失败 userId=%d currency=%s walletType=%d "+
        "amount=%s bizType=%d refOrderNo=%s reason=%s",
    req.UserId, req.CurrencyCode, req.WalletType,
    amount.String(), req.BizType, req.RefOrderNo,
    err.Error())

// before/after快照是关键: 事后可以重建余额变化轨迹
```

---

<a id="议题十三"></a>
## 议题十三 可扩展性设计

### 13.1 配置驱动而非代码驱动

```
新增币种:
  ✅ B端增加一条currency_config记录 → 全局可用
  ❌ 改代码 + 加枚举 + 重新部署

新增子钱包类型:
  ✅ wallet_account增加walletType枚举值 → 对应用户增加行
  ❌ 改表结构 + 迁移数据

新增业务类型:
  ✅ bizType枚举增加值 → CreditWallet/DebitWallet直接支持
  ❌ 新写一套接口
```

### 13.2 预留弹性空间

```
已预留但v1不实现:
  ├─ walletType=4(代理钱包): 代理佣金结算
  ├─ walletType=5(场馆钱包): 临时场馆资金
  ├─ bizType有保留区间: 未来新业务类型
  ├─ DECIMAL(20,4): 20位总长度够100万亿
  └─ TiDB自动分片: 亿级数据无需手动分表

v1不做但架构支持:
  ├─ 多币种钱包之间的转账(C2C)
  ├─ 子钱包之间的资金调拨
  ├─ 定时解锁(奖励钱包流水达到倍数后自动释放)
  └─ 余额变动Webhook通知(外部系统订阅)
```

### 13.3 接口版本化

```
// Thrift IDL 向前兼容:
// 新增字段使用optional + 高编号

struct CreditWalletReq {
    1: required i64 userId,
    2: required string currencyCode,
    3: required i32 walletType,
    4: required string amount,
    5: required i32 bizType,
    6: required string refOrderNo,
    7: optional string remark,
    8: optional i32 turnoverMultiple,
    // v2 新增字段(不影响v1调用方):
    20: optional string sourceTraceId,   // 来源链路追踪
    21: optional map<string,string> ext, // 扩展字段
}
```

---

<a id="附录a"></a>
## 附录A 方案选型对比总表

| 议题 | 我们的方案 | 备选方案 | 为什么选这个 |
|------|-----------|---------|------------|
| 并发控制(主) | 悲观锁 FOR UPDATE | 乐观锁 Version | 写密集高冲突，悲观锁一次成功，乐观锁频繁重试 |
| 并发控制(辅) | Redis分布式锁 | Go sync.Mutex | 多实例部署，进程锁无效 |
| 幂等 | Redis SetNX + DB唯一索引 | 仅Redis / 仅DB | 双层互补：Redis快+DB可靠 |
| 本地事务 | GORM query.Q.Transaction | 手动Begin/Commit | 函数式事务自动Rollback，不会遗漏 |
| 分布式事务 | TCC(手动实现) | DTM框架 / 2PC / Saga | 只有提现需要，引入框架过重 |
| 金额精度 | shopspring/decimal | float64 / math/big | Go生态事实标准，GORM原生支持 |
| 金额存储 | DECIMAL(20,4) | BIGINT最小单位 | DECIMAL可读性好，TiDB原生支持 |
| 金额传输 | i64最小单位(RPC) / string(HTTP) | 统一string | RPC用i64性能好，HTTP用string可读 |
| 缓存策略 | Cache-Aside + 写后删除 | Cache-Through | 钱包一致性要求高，不能异步更新 |
| 缓存TTL | 30秒(余额) / 5分钟(币种) | 长TTL / 无TTL | 30秒兼顾一致性和命中率 |
| 状态机 | 白名单 + DB WHERE条件 | 枚举FSM库 | 简单直接，无需引入额外库 |
| 死锁预防 | 固定加锁顺序 | NOWAIT + 重试 | 固定顺序从根本上消除死锁 |
| 对账 | 定时任务 SUM(流水) vs 余额 | 实时触发器 | 定时即可，实时性要求不高 |
| 审计 | 流水表 + ser-blog双重 | 仅流水表 | 流水=资金审计，日志=操作审计 |
| 降级 | 分级(强/弱依赖) | 全部强依赖 | 弱依赖故障不应阻塞核心链路 |

---

<a id="附录b"></a>
## 附录B 各议题与接口映射矩阵

```
                    并发 幂等 事务 TCC 精度 死锁 缓存 状态机 安全
CreditWallet         ●    ●    ●        ●              ●
DebitWallet          ●    ●    ●        ●              ●
FreezeBalance        ●    ●    ●   ●    ●              ●
UnfreezeBalance      ●    ●    ●   ●    ●         ●    ●
ConfirmDebit         ●    ●    ●   ●    ●         ●    ●
ExecuteExchange      ●    ●    ●        ●    ●         ●
GetBalance                          ●         ●
GetCurrencyList                                   ●
GetExchangeRate                     ●         ●
InitWallet                ●    ●
CreateDepositOrder             ●              ●    ●
CreateWithdrawOrder  ●         ●        ●         ●    ●
B端币种CRUD                    ●              ●
汇率刷新                                     ●

● = 该议题方案在该接口中必须应用
```

---

<a id="附录c"></a>
## 附录C 关键Go代码骨架汇总

### C.1 项目结构

```
ser-wallet/
├── main.go                        # 启动入口
├── handler.go                     # WalletServiceImpl(薄代理)
├── internal/
│   ├── ser/                       # Service层(业务核心)
│   │   ├── wallet.go              # CreditWallet/DebitWallet/Freeze/Unfreeze/Confirm
│   │   ├── exchange.go            # ExchangePreview/ExecuteExchange
│   │   ├── deposit.go             # CreateDepositOrder/QueryDepositStatus
│   │   ├── withdraw.go            # CreateWithdrawOrder/WithdrawInfo
│   │   ├── balance.go             # GetBalance/BatchGetBalance
│   │   ├── currency.go            # 币种CRUD
│   │   └── rate.go                # 汇率管理+刷新
│   ├── rep/                       # Repository层(数据持久化)
│   │   ├── wallet_account.go      # GetForUpdate/CreditAvailable/DebitAvailable/Freeze/Unfreeze
│   │   ├── wallet_transaction.go  # InsertFlow/SumNetFlow
│   │   ├── freeze_detail.go       # InsertFreeze/UpdateStatus
│   │   ├── deposit_order.go       # CRUD
│   │   ├── withdraw_order.go      # CRUD + UpdateStatus
│   │   ├── exchange_order.go      # CRUD
│   │   ├── currency_config.go     # CRUD
│   │   └── exchange_rate_log.go   # Insert/Page
│   ├── cache/                     # Cache层
│   │   ├── balance.go             # 余额缓存(30s)
│   │   ├── currency.go            # 币种缓存(5min)
│   │   └── rate.go                # 汇率缓存(10min)
│   ├── pkg/                       # 内部工具包
│   │   ├── dlock/dlock.go         # 分布式锁封装
│   │   ├── idempotent/idem.go     # 幂等检查封装
│   │   └── lockorder/order.go     # 多行加锁顺序工具
│   ├── enum/                      # 枚举与常量
│   │   ├── biz_type.go            # bizType(1-15)
│   │   ├── wallet_type.go         # walletType(1-5)
│   │   ├── direction.go           # direction(1-4)
│   │   └── order_status.go        # 订单状态+转换白名单
│   ├── errs/                      # 错误码
│   │   └── code.go                # 6015000+
│   └── gorm_gen/                  # GORM Gen生成代码
│       ├── query/
│       └── model/
```

### C.2 核心Service骨架

```go
// ============= WalletService 依赖注入 =============

type WalletService struct {
    accountRepo *rep.WalletAccountRepo
    txnRepo     *rep.WalletTransactionRepo
    freezeRepo  *rep.FreezeDetailRepo
    balanceCache *cache.BalanceCache
}

func NewWalletService(
    accountRepo *rep.WalletAccountRepo,
    txnRepo *rep.WalletTransactionRepo,
    freezeRepo *rep.FreezeDetailRepo,
    balanceCache *cache.BalanceCache,
) *WalletService {
    return &WalletService{
        accountRepo:  accountRepo,
        txnRepo:      txnRepo,
        freezeRepo:   freezeRepo,
        balanceCache: balanceCache,
    }
}
```

### C.3 标准写操作模板

```go
// ============= 所有写操作遵循此模板 =============

func (s *WalletService) XxxWriteOperation(ctx context.Context, req *XxxReq) (*XxxResp, error) {
    // ① 幂等检查
    idemKey := idempotent.IdempotentKey(req.BizType, req.RefOrderNo, req.WalletType)
    if isRepeat, cached := idempotent.CheckAndMark(ctx, idemKey); isRepeat {
        return parseCachedResp(cached), nil
    }

    // ② Redis分布式锁
    lockKey := dlock.WalletLockKey(req.UserId, req.WalletType, req.CurrencyCode)
    var resp *XxxResp

    err := dlock.WithLock(ctx, lockKey, func() error {
        // ③ DB事务(悲观锁+原子更新+写流水)
        return query.Q.Transaction(func(tx *query.Query) error {
            // SELECT FOR UPDATE
            account, err := s.accountRepo.GetForUpdate(ctx, tx, ...)
            if err != nil { return err }

            // 业务校验
            if err := validate(account, req); err != nil { return err }

            // 原子更新
            affected, err := s.accountRepo.XxxUpdate(ctx, tx, ...)
            if err != nil { return err }
            if affected == 0 { return bizErr("并发冲突") }

            // 写流水
            err = s.txnRepo.InsertFlow(ctx, tx, ...)
            if err != nil { return err }

            resp = &XxxResp{...}
            return nil
        })
    })

    if err != nil {
        idempotent.MarkFailed(ctx, idemKey) // 失败清除幂等占位
        return nil, err
    }

    // ④ 事务外操作(不阻塞)
    s.balanceCache.InvalidateBalance(ctx, req.UserId, req.CurrencyCode)
    s.asyncAuditLog(ctx, "Xxx", req)
    idempotent.MarkCompleted(ctx, idemKey, marshal(resp))

    return resp, nil
}
```

### C.4 错误码规划

```go
// ser-wallet 错误码: 6015000 ~ 6015999
// 微服务编号: 015

const (
    WalletBaseError          = 6015000 + iota
    WalletAccountNotFound    // 6015001 钱包账户不存在
    InsufficientBalance      // 6015002 余额不足
    WalletOperationBusy      // 6015003 操作频繁，请稍后重试
    ConcurrentConflict       // 6015004 并发冲突
    RequestProcessing        // 6015005 请求处理中，请勿重复提交
    InvalidAmount            // 6015006 金额无效
    CurrencyNotFound         // 6015007 币种不存在
    CurrencyDisabled         // 6015008 币种已禁用
    FreezeRecordNotFound     // 6015009 冻结记录不存在
    FreezeAlreadyUnfrozen    // 6015010 已解冻，无法确认
    FreezeAlreadyConfirmed   // 6015011 已确认扣款，无法解冻
    FreezeConcurrentConflict // 6015012 冻结状态并发冲突
    FreezeRejectedByCancel   // 6015013 已取消，冻结被拒绝
    OrderStatusConflict      // 6015014 订单状态冲突
    UserBlocked              // 6015015 用户已封禁
    WithdrawNameMismatch     // 6015016 提现账户名与实名不匹配
    WithdrawLimitExceeded    // 6015017 超出提现限额
    ExchangeRateExpired      // 6015018 汇率已过期
    BaseCurrencyAlreadySet   // 6015019 基准币种已设置不可变更
)

// 币种配置错误码: 6015100 ~ 6015199
const (
    CurrencyConfigBaseError     = 6015100 + iota
    CurrencyCodeAlreadyExist    // 6015101 币种代码已存在
    CurrencyNameAlreadyExist    // 6015102 币种名称已存在
    CurrencyHasBalance          // 6015103 该币种下有用户持有余额，不可禁用
    CurrencyFieldsIncomplete    // 6015104 启用币种需完善必填信息
)
```

---

> **文档价值**：13个议题覆盖了钱包编码中所有"绕不过的问题"。每个议题不是空谈原理，而是给出了：
> - 基于现有工程基线（rds.SetNX, query.Q.Transaction, ret.BizErr）的具体Go代码
> - 为什么选这个方案而不是备选方案的清晰理由
> - 陷阱与边界条件的预警
> - 验证方法
>
> 这份文档是编码阶段的"技术决策指南"——遇到任何议题，翻到对应章节，方案就在那里。
