# ser-wallet 核心工程挑战与解决方案 — 实现蓝图

> 定位：聚焦"和钱打交道"时绕不过的核心工程挑战——不是普通CRUD，而是幂等性、并发安全、事务一致性、精度、安全等必须在编码前就想清楚的问题
> 视角：从技术挑战/c-5.md的方案选型结论出发，深入到**实现蓝图级**——具体怎么写、Go/SQL伪代码、错误处理模式、跨机制协同
> 区别：c-5.md = "选什么药"（方案对比与选型）→ 本文档 = "怎么用药、完整疗程"（实现蓝图与编码指导）
> 依据：技术挑战/c-5.md(1126行) + 调用链路图/tmp-2.md(1367行) + 表结构设计/c-5.md(1187行) + 支付沟通/tmp-2.md(1409行) + 存储会话/c-3.md + 实际代码库
> 阶段：预评审期，编码前的核心机制完整蓝图

---

# 第1章：核心工程挑战总览

## 1.1 钱包 vs 普通CRUD的本质区别

```
普通CRUD接口（如新闻列表、用户信息管理）：
  ├─ 同一请求执行两次？无所谓，覆盖就行
  ├─ 两个请求同时写？最后一个赢，业务可接受
  ├─ 数据偶尔不一致？刷新一下就好
  └─ 接口报错？提示用户重试

钱包模块的资金操作：
  ├─ 同一请求执行两次？灾难——充值两次=多给钱，扣款两次=多扣钱
  ├─ 两个请求同时写？灾难——超扣=余额变负，更新丢失=少入账
  ├─ 数据不一致？灾难——余额和流水对不上=审计无法通过
  └─ 接口报错？不能简单重试——重试可能造成重复操作

核心区别：
  1. 不可逆性：钱一旦多给或多扣，追回成本极高
  2. 零容忍：资损不是"偶尔出现可接受"，是"一次都不行"
  3. 并发争抢：同一用户余额被多方同时操作（充值回调+兑换+人工加款）
  4. 跨模块依赖：充值/提现涉及财务模块，没有全局事务
```

## 1.2 12个挑战的编码分类

按"你写代码时什么时候会遇到它"分类：

```
┌─────────────────────────────────────────────────────────────────────┐
│ 分类                        │ 挑战                │ 影响级别       │
├─────────────────────────────────────────────────────────────────────┤
│ 每一行余额代码都面对          │ 1. 并发安全(CAS)    │ 致命 ★★★      │
│                             │ 2. 金额精度(BIGINT)  │ 致命 ★★★      │
│                             │ 3. 事务边界(铁律)    │ 严重 ★★★      │
├─────────────────────────────────────────────────────────────────────┤
│ 每一个写接口都面对            │ 4. 幂等性(双层)      │ 致命 ★★★      │
│                             │ 5. 账务一致性(三件套) │ 严重 ★★       │
├─────────────────────────────────────────────────────────────────────┤
│ 跨模块操作时面对              │ 6. 分布式一致性(补偿) │ 严重 ★★★      │
├─────────────────────────────────────────────────────────────────────┤
│ 特定业务场景面对              │ 7. 冻结模型(三步)    │ 严重 ★★       │
│                             │ 8. 安全与防刷        │ 重要 ★★       │
│                             │ 9. 缓存一致性        │ 重要 ★        │
├─────────────────────────────────────────────────────────────────────┤
│ 系统层面（当前阶段不处理）     │ 10. 热点账户         │ 关注          │
│                             │ 11. 高可用降级        │ 关注          │
│                             │ 12. 数据增长治理      │ 关注          │
└─────────────────────────────────────────────────────────────────────┘
```

## 1.3 安全铁三角：三个挑战的协同关系

```
并发安全(CAS) ─→ 保证"一次操作原子完成"
       │
       ├─→ 事务边界(铁律) ─→ 保证"事务内只有DB操作，极短极快"
       │         │
       │         └─→ 账务一致性(三件套) ─→ 保证"余额+流水+订单同时成功"
       │
       └─→ 幂等性(双层) ─→ 保证"重复调用不会重复执行"

三者缺一不可：
  没有CAS → 并发超扣
  没有事务边界 → 长事务拖垮连接池
  没有三件套 → 余额改了流水没写，对账永远对不上
  没有幂等 → 重复入账/扣款
```

## 1.4 本文档结构

```
第2-5章：编码基础机制（每行代码都用到）
  → CAS乐观锁、幂等双层、金额精度、事务边界

第6-7章：跨模块与特定场景机制
  → 本地补偿、冻结模型

第8-9章：安全与缓存
  → 数据安全、防刷风控、Cache-Aside

第10章：系统级考量（认知储备，当前不实现）
  → 热点账户、高可用、数据增长

第11章：跨机制协同（所有机制如何在一次操作中串联）
  → 以真实接口为例，展示完整调用栈

第12章：验证矩阵与编码指南
  → 按开发阶段的验证路径 + Code Review检查清单
```

---

# 第2章：并发安全工程 — CAS乐观锁完整实现

> 方案选型结论（详见技术挑战/c-5.md 第2.1节）：选择CAS条件更新，不选悲观锁/分布式锁/队列串行化
> 本章重点：怎么写、4种场景的完整SQL、Go代码模式、错误处理、测试策略

## 2.1 核心原理

```
CAS（Compare And Swap）在数据库层面的实现：
  UPDATE ... SET field = field ± amount WHERE field >= amount

  数据库（InnoDB/TiDB）执行这条SQL时：
  1. 加行级排他锁（微秒级）
  2. 检查 WHERE 条件（field >= amount?）
  3. 条件满足 → 执行更新 → 释放锁 → affected_rows = 1
  4. 条件不满足 → 不更新 → 释放锁 → affected_rows = 0

  整个过程是原子的：不会出现"读到旧值→算出新值→写入"之间被插入的情况
  这就是为什么一条SQL就能安全处理并发——数据库帮你做了"比较+交换"
```

## 2.2 四种场景的完整SQL

### 场景1：扣减余额（兑换扣源币、投注扣款、人工减款）

```sql
UPDATE user_balance
SET available = available - @amount,
    update_at = @now
WHERE user_id = @userId
  AND currency = @currency
  AND wallet_type = @walletType
  AND available >= @amount    -- CAS条件：够扣才扣
```

### 场景2：增加余额（充值入账、兑换加BSB、人工加款）

```sql
UPDATE user_balance
SET available = available + @amount,
    update_at = @now
WHERE user_id = @userId
  AND currency = @currency
  AND wallet_type = @walletType
-- 注意：增加操作没有 available >= 条件（加钱不会失败）
```

### 场景3：冻结余额（提现申请）

```sql
UPDATE user_balance
SET available = available - @amount,
    frozen = frozen + @amount,
    update_at = @now
WHERE user_id = @userId
  AND currency = @currency
  AND wallet_type = @walletType
  AND available >= @amount    -- CAS条件：可用余额够才能冻结
```

### 场景4：解冻/扣除冻结（提现结果回调）

```sql
-- 4a: 提现成功 → 扣除冻结金额（钱真正离开系统）
UPDATE user_balance
SET frozen = frozen - @amount,
    update_at = @now
WHERE user_id = @userId
  AND currency = @currency
  AND wallet_type = @walletType
  AND frozen >= @amount       -- CAS条件：冻结金额够才能扣除

-- 4b: 提现失败 → 解冻（冻结金额退回可用）
UPDATE user_balance
SET frozen = frozen - @amount,
    available = available + @amount,
    update_at = @now
WHERE user_id = @userId
  AND currency = @currency
  AND wallet_type = @walletType
  AND frozen >= @amount       -- CAS条件：冻结金额够才能解冻
```

## 2.3 Go实现伪代码

```go
// BalanceRepo — Repository层方法

// DeductAvailable 扣减可用余额（CAS安全）
// 返回 affected rows: 1=成功, 0=余额不足
func (r *BalanceRepo) DeductAvailable(
    ctx context.Context, tx *query.Query,
    userId int64, currency string, walletType int32, amount int64,
) (int64, error) {
    ub := tx.UserBalance
    now := time.Now().UnixMilli()

    result, err := ub.WithContext(ctx).
        Where(ub.UserID.Eq(userId)).
        Where(ub.Currency.Eq(currency)).
        Where(ub.WalletType.Eq(walletType)).
        Where(ub.Available.Gte(amount)). // CAS条件
        UpdateColumns(map[string]interface{}{
            "available": gorm.Expr("available - ?", amount),
            "update_at": now,
        })
    if err != nil {
        return 0, err
    }
    return result.RowsAffected, nil
}

// AddAvailable 增加可用余额
// 增加操作理论上不会失败（不检查上限）
func (r *BalanceRepo) AddAvailable(
    ctx context.Context, tx *query.Query,
    userId int64, currency string, walletType int32, amount int64,
) (int64, error) {
    ub := tx.UserBalance
    now := time.Now().UnixMilli()

    result, err := ub.WithContext(ctx).
        Where(ub.UserID.Eq(userId)).
        Where(ub.Currency.Eq(currency)).
        Where(ub.WalletType.Eq(walletType)).
        UpdateColumns(map[string]interface{}{
            "available": gorm.Expr("available + ?", amount),
            "update_at": now,
        })
    if err != nil {
        return 0, err
    }
    return result.RowsAffected, nil
}
```

## 2.4 affected_rows 的处理策略

```
场景          │ affected_rows = 0 的含义         │ 处理方式
────────────────────────────────────────────────────────────────
扣减(场景1)   │ 余额不足                         │ 返回 ErrInsufficientBalance
冻结(场景3)   │ 可用余额不足                     │ 返回 ErrInsufficientBalance
解冻(场景4)   │ 冻结金额不足（不应发生）          │ 日志告警 + 返回系统错误
增加(场景2)   │ 余额行不存在（首次操作该币种）    │ 触发"首次创建余额行"逻辑
```

## 2.5 首次余额行不存在的处理

```go
// 用户首次操作某个币种/子钱包时，user_balance表里没有对应行
// 解决：先尝试UPDATE，affected=0时检查是否因为行不存在

func (r *BalanceRepo) EnsureBalanceRow(
    ctx context.Context, tx *query.Query,
    userId int64, currency string, walletType int32,
) error {
    ub := tx.UserBalance

    // INSERT IGNORE：行存在则不操作，不存在则创建
    // 利用唯一键 (user_id, currency, wallet_type) 保证幂等
    err := ub.WithContext(ctx).Create(&model.UserBalance{
        UserID:     userId,
        Currency:   currency,
        WalletType: walletType,
        Available:  0,
        Frozen:     0,
        // CreateAt/UpdateAt 由 UserPlugin 自动填充
    })
    if err != nil {
        // 唯一键冲突 → 行已存在 → 忽略（这是预期的）
        if isDuplicateKeyError(err) {
            return nil
        }
        return err
    }
    return nil
}

// 调用时机：
// 1. 充值入账前：EnsureBalanceRow → AddAvailable
// 2. 兑换时BSB子钱包可能不存在：EnsureBalanceRow → AddAvailable
// 3. 其他增加操作前
```

## 2.6 为什么不重试

```
关键认知：CAS的 affected_rows=0 不是"竞争失败可重试"

  扣减场景：affected_rows=0 → 余额不足 → 重试也不够 → 直接返回业务错误
  冻结场景：同上

  增加场景：affected_rows=0 → 行不存在（不是竞争问题）→ 创建行后重试一次

  对比"版本号乐观锁"：
    version乐观锁的 affected_rows=0 = "有人抢先更新了版本号"
    → 可以重试：重新读版本号再UPDATE
    CAS条件更新的 available >= amount 是业务条件，不是版本条件
    → 不需要重试
```

## 2.7 并发测试策略

```
测试1：10个goroutine同时扣减同一用户，每笔100，初始余额500
  预期：5笔成功（500/100=5），5笔返回余额不足
  验证：最终余额=0，流水=5条，每条100

测试2：5个goroutine同时充值（增加），每笔1000
  预期：5笔全部成功
  验证：最终余额=初始+5000，流水=5条

测试3：1个goroutine冻结500，同时1个goroutine扣减500，初始余额500
  预期：只有一个成功，另一个返回余额不足
  验证：最终 available+frozen=500
```

## 2.8 出现在哪些链路

```
链路3-充值回调：场景2（增加）→ CreditWallet
链路4-兑换：场景1（扣减源币）+ 场景2（增加BSB）
链路5-提现：场景3（冻结）+ 场景4a/4b（扣除冻结/解冻）
链路6-RPC被调：场景1/2/3/4 全部覆盖（取决于具体RPC方法）
```

---

# 第3章：幂等性工程 — 双层防护完整实现

> 方案选型结论（详见技术挑战/c-5.md 第2.2节）：选择两层幂等（应用层+DB唯一键），不选Redis SETNX/Token令牌/独立幂等表
> 本章重点：每一层怎么实现、幂等键设计、并发穿透处理、返回值设计

## 3.1 两层幂等的架构

```
请求到达（外部模块RPC调用）
  │
  ├─ 第一层：应用层状态检查（快速路径）
  │   查询 balance_flow 表：是否已有 ref_order_no + change_type 的记录
  │   ├─ 存在 → 直接返回上次成功结果（不执行任何操作）
  │   └─ 不存在 → 继续执行
  │
  ├─ 正常业务逻辑执行...
  │
  └─ 第二层：DB唯一键兜底（安全网）
      写入 balance_flow 时：
      UNIQUE KEY uk_order_type (ref_order_no, change_type)
      ├─ 写入成功 → 正常流程
      └─ 唯一键冲突 → 说明并发穿透，另一个请求已成功写入
         → 返回成功（幂等语义）

拦截比例：
  第一层：拦截 99.9% 的重复请求（一次DB查询）
  第二层：拦截 0.1% 极端并发穿透（DB约束兜底）
```

## 3.2 第一层实现：应用层状态检查

```go
// CreditWallet — 充值入账（RPC暴露接口）
func (s *WalletService) CreditWallet(ctx context.Context, req *wallet.CreditRequest) (*wallet.CreditResponse, error) {
    // ═══ 第一层幂等：查流水表 ═══
    existFlow, err := s.flowRepo.FindByOrderAndType(ctx, req.OrderNo, ChangeTypeDeposit)
    if err != nil {
        return nil, err // DB查询异常，返回错误（调用方会重试）
    }
    if existFlow != nil {
        // 幂等命中：返回上次成功的结果
        return &wallet.CreditResponse{
            Success:    true,
            NewBalance: existFlow.BalanceAfter,
            FlowId:     existFlow.ID,
            Message:    "already processed", // 可选：告知调用方这是幂等返回
        }, nil
    }

    // ═══ 继续执行正常业务逻辑 ═══
    // 参数校验 → 事务（余额+流水+订单）→ 返回
    // ...
}
```

```go
// flowRepo.FindByOrderAndType — 幂等查询方法
func (r *FlowRepo) FindByOrderAndType(
    ctx context.Context, orderNo string, changeType int32,
) (*model.BalanceFlow, error) {
    bf := r.db.BalanceFlow
    flow, err := bf.WithContext(ctx).
        Where(bf.RefOrderNo.Eq(orderNo)).
        Where(bf.ChangeType.Eq(changeType)).
        First()
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, nil // 不存在 → 非幂等请求
        }
        return nil, err // DB异常
    }
    return flow, nil
}
```

## 3.3 第二层实现：DB唯一键兜底

```
表定义（balance_flow表）：
  UNIQUE KEY uk_order_type (ref_order_no, change_type)

当两个并发请求同时穿透第一层：
  goroutine A: 查流水不存在 → 执行业务 → 写流水
  goroutine B: 查流水不存在（A还没写完）→ 执行业务 → 写流水

  结果：
  A写流水成功 → 正常返回
  B写流水触发唯一键冲突 → catch错误 → 返回成功（幂等语义）
```

```go
// 在事务内写流水时处理唯一键冲突
func (s *WalletService) executeInTransaction(ctx context.Context, ...) error {
    return s.balanceRepo.Transaction(ctx, func(tx *query.Query) error {
        // ... 余额更新 ...

        // 写流水
        err := tx.BalanceFlow.WithContext(ctx).Create(flow)
        if err != nil {
            if isDuplicateKeyError(err) {
                // ★ 唯一键冲突 = 另一个并发请求已成功
                // 返回nil（不是错误），上层判断后返回幂等成功
                return ErrIdempotentHit // 自定义sentinel error
            }
            return err
        }
        // ... 订单状态更新 ...
        return nil
    })
}

// isDuplicateKeyError — 判断是否是MySQL/TiDB唯一键冲突
func isDuplicateKeyError(err error) bool {
    // MySQL Error Code 1062: Duplicate entry
    var mysqlErr *mysql.MySQLError
    if errors.As(err, &mysqlErr) && mysqlErr.Number == 1062 {
        return true
    }
    return false
}
```

## 3.4 每个接口的幂等键映射表

```
接口                     │ ref_order_no        │ change_type          │ 第一层检查点
─────────────────────────────────────────────────────────────────────────────────
CreditWallet (充值入账)   │ 平台充值订单号       │ DEPOSIT              │ balance_flow查询
DebitWallet (人工减款)    │ 修正单号             │ MANUAL_DEDUCT        │ balance_flow查询
FreezeBalance (提现冻结)  │ 提现订单号           │ WITHDRAW_FREEZE      │ balance_flow查询
UnfreezeBalance (解冻)    │ 提现订单号           │ WITHDRAW_UNFREEZE    │ balance_flow查询
DeductFrozenAmount (扣冻) │ 提现订单号           │ WITHDRAW_DEDUCT      │ balance_flow查询
DebitByPriority (投注扣减)│ 投注单号             │ BET_DEDUCT           │ balance_flow查询
SupplementCredit (补单)   │ 补单号(B+16位)       │ SUPPLEMENT           │ balance_flow查询
─────────────────────────────────────────────────────────────────────────────────
C端兑换(Exchange)        │ 兑换订单号           │ EXCHANGE_OUT/IN/BONUS│ exchange_order.status
C端充值创建              │ 平台充值订单号        │ —                   │ deposit_order查重
C端提现创建              │ 提现订单号           │ —                    │ withdraw_order查重
```

## 3.5 幂等返回值设计

```
核心原则：幂等返回 = 返回上次成功的结果，不是返回"已处理"

  正确做法：
    CreditWallet 幂等返回 → {success: true, newBalance: 上次入账后的余额, flowId: 上次的流水ID}
    调用方（财务）收到这个响应，和正常成功的响应格式完全一致
    它不需要知道"这是第一次还是第N次调用"

  错误做法：
    返回 {code: "ALREADY_PROCESSED", message: "订单已处理"}
    调用方需要额外判断这个特殊code，增加耦合

为什么这样设计：
  幂等的语义是"调用N次和调用1次的效果完全一致"
  包括返回值也应该一致
  调用方不需要区分"首次成功"和"幂等成功"
```

## 3.6 极端并发穿透场景分析

```
时间线：

  T0: goroutine A 查流水 → 不存在
  T1: goroutine B 查流水 → 不存在（A还没写入）
  T2: goroutine A 开始事务 → 更新余额 → 写流水成功 → COMMIT
  T3: goroutine B 开始事务 → 更新余额 → 写流水 → 唯一键冲突！

  问题：B已经更新了余额但流水写入失败，会不会导致余额多加？
  答案：不会。因为事务回滚（唯一键冲突在事务内触发，整个事务回滚）

  流程：
    B的事务内：UPDATE余额 → INSERT流水(冲突!) → 事务ROLLBACK → 余额恢复原值
    上层catch到ErrIdempotentHit → 返回幂等成功（查A写入的流水数据）
```

## 3.7 为什么不用Redis SETNX

```
1. 幂等键天然存在于DB表中
   balance_flow.(ref_order_no, change_type) 已经是唯一键
   不需要在Redis中额外维护一份

2. Redis有失效风险
   SETNX需要设TTL → 过期后幂等失效 → 重复操作
   主从切换时可能丢数据 → 已设置的key消失 → 幂等失效
   对资金操作，幂等的兜底必须是持久化的、不会过期的

3. 增加Redis依赖 = 增加故障点
   Redis不可用 → 幂等检查跳过 → 所有重复请求都能穿透
   DB唯一键不受Redis状态影响
```

---

# 第4章：金额精度工程 — BIGINT全链路实现

> 方案选型结论（详见技术挑战/c-5.md 第2.3节）：BIGINT最小单位整数，不选DECIMAL/shopspring/decimal/字符串
> 本章重点：精度换算函数、汇率整数计算、溢出防护、API层约定

## 4.1 存储规则

```
所有金额存储为 int64（BIGINT），表示该币种的最小货币单位：

币种    precision    换算关系                         示例
────────────────────────────────────────────────────────────
VND     0           1 VND = 1（无小数）               1,000,000 VND → 存 1000000
IDR     0           1 IDR = 1（无小数）               500,000 IDR → 存 500000
THB     2           1 THB = 100                      10.50 THB → 存 1050
USDT    2           1 USDT = 100                     100.50 USDT → 存 10050
BSB     2           1 BSB = 100                      10.50 BSB → 存 1050

precision 由 currency_config 表的 precision 字段配置
上线后不可修改（修改会导致历史数据全部错误）
```

## 4.2 精度换算核心函数

```go
// package utils — 集中管理精度换算

// ToMinUnit 展示金额 → 最小单位整数
// displayAmount: 字符串形式的金额（如 "10.50"）
// precision: 币种精度（如 BSB=2）
// 返回: int64 最小单位（如 1050）
func ToMinUnit(displayAmount string, precision int32) (int64, error) {
    // 1. 解析字符串为 big.Rat（精确有理数）
    rat := new(big.Rat)
    if _, ok := rat.SetString(displayAmount); !ok {
        return 0, fmt.Errorf("invalid amount: %s", displayAmount)
    }

    // 2. 乘以 10^precision
    multiplier := new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(precision)), nil)
    result := new(big.Rat).Mul(rat, new(big.Rat).SetInt(multiplier))

    // 3. 检查是否为整数（不应有更多小数位）
    if !result.IsInt() {
        return 0, fmt.Errorf("amount %s exceeds precision %d", displayAmount, precision)
    }

    // 4. 转为 int64
    return result.Num().Int64(), nil
}

// ToDisplayAmount 最小单位整数 → 展示字符串
// minUnit: int64 最小单位（如 1050）
// precision: 币种精度（如 BSB=2）
// 返回: 字符串（如 "10.50"）
func ToDisplayAmount(minUnit int64, precision int32) string {
    if precision == 0 {
        return strconv.FormatInt(minUnit, 10)
    }
    divisor := int64(math.Pow10(int(precision)))
    intPart := minUnit / divisor
    fracPart := minUnit % divisor
    format := fmt.Sprintf("%%d.%%0%dd", precision)
    return fmt.Sprintf(format, intPart, fracPart)
}
```

## 4.3 汇率计算的整数公式

```
汇率存储方式：
  exchange_rate.rate = int64      （如 230000000）
  exchange_rate.rate_precision = 4 （汇率精度）
  实际汇率 = rate / 10^rate_precision = 230000000 / 10000 = 23000

兑换计算（法币→BSB）：
  from_amount: 源币种最小单位（已经是int64）
  to_amount = from_amount × 10^rate_precision / rate

  示例：1,000,000 VND → BSB（汇率 1 BSB = 23000 VND）
    to_amount = 1000000 × 10000 / 230000000
    to_amount = 10000000000 / 230000000
    to_amount ≈ 43   （43最小单位 = 0.43 BSB）

关键：先乘后除
  错误：from_amount / rate × 10^precision → 整数除法先截断，精度丢失
  正确：from_amount × 10^precision / rate → 先乘大再除，减少截断误差
```

## 4.4 溢出防护

```go
// SafeAdd 安全加法（防溢出）
func SafeAdd(a, b int64) (int64, error) {
    if b > 0 && a > math.MaxInt64-b {
        return 0, errors.New("integer overflow")
    }
    if b < 0 && a < math.MinInt64-b {
        return 0, errors.New("integer underflow")
    }
    return a + b, nil
}

// SafeSub 安全减法（防下溢）
func SafeSub(a, b int64) (int64, error) {
    if b > 0 && a < math.MinInt64+b {
        return 0, errors.New("integer underflow")
    }
    return a - b, nil
}

// 实际风险评估：
// int64 最大 = 9.2 × 10^18
// 最大的币种 VND（precision=0）：1亿VND = 10^8，距上限还有10^10倍
// 即使用户有千亿VND余额也不会溢出
// SafeAdd/SafeSub 是防御性编程，不是解决实际问题
```

## 4.5 API层约定

```
入参：C端/RPC传入展示金额用 string 类型
  → 避免JSON序列化/反序列化时的浮点精度问题
  → "10.50" 而不是 10.5 或 10.50

内部：全程 int64 计算
  → 加减乘除全是整数运算
  → 不引入 float64 做中间计算

出参：返回给C端/RPC也用 string 类型
  → ToDisplayAmount(balance.Available, config.Precision)

IDL定义：
  i64 amount → 最小单位int64（服务间RPC）
  string display_amount → 展示金额（C端API）
```

## 4.6 常见精度Bug预防

```
Bug1: 用float64做中间变量
  ✗ amount := float64(req.Amount) / 100.0  // 精度丢失
  ✓ 全程int64，展示时才调ToDisplayAmount

Bug2: 在展示层做业务计算
  ✗ totalDisplay := "10.50" + "20.30"  // 字符串拼接？
  ✓ total := amount1 + amount2（int64加法），展示时一次转换

Bug3: 汇率计算先除后乘
  ✗ bsb = vnAmount / rate * 10000  // 先除截断
  ✓ bsb = vnAmount * 10000 / rate  // 先乘后除

Bug4: 修改已上线币种的precision
  ✗ VND从precision=0改为precision=2 → 所有历史金额意义改变
  ✓ precision上线后immutable，代码层面禁止修改
```

---

# 第5章：事务边界工程 — 铁律与实现模式

> 方案选型结论（详见技术挑战/c-5.md 第3.2节）：严格"RPC在事务外，DB在事务内"
> 本章重点：标准事务模板、事务内禁止清单、提现的特殊事务边界、连接池配置

## 5.1 铁律

```
事务内只做DB操作，事务外做一切其他IO

为什么：
  事务 = 占用DB连接 + 持有行锁
  事务越长，连接占用越久，行锁持有越久
  RPC调用超时3秒 = DB连接和行锁被占用3秒
  10个并发请求卡在事务内RPC = 连接池耗尽 = 系统雪崩

量化：
  事务内纯DB操作（5条SQL）→ 耗时 1-5ms → 连接占用 5ms
  事务内含RPC调用 → 耗时 3000ms（超时）→ 连接占用 3000ms
  差距 600 倍
```

## 5.2 标准事务模板（以兑换为例）

```go
func (s *ExchangeService) Execute(ctx context.Context, req *ExchangeRequest) error {

    // ═══════════ 事务之外（可以慢，不影响DB）═══════════

    // 1. 获取最新汇率（可能涉及Redis/DB查询）
    rate, err := s.currencyRepo.GetLatestRate(ctx, req.FromCurrency)
    if err != nil {
        return fmt.Errorf("get rate: %w", err)
    }

    // 2. 计算兑换金额（纯CPU计算）
    toAmount := req.Amount * int64(math.Pow10(int(rate.RatePrecision))) / rate.Rate
    bonusAmount := toAmount * rate.BonusPercent / 100

    // 3. 预校验余额（快速失败，减少无效事务启动）
    balance, err := s.balanceRepo.GetBalance(ctx, req.UserId, req.FromCurrency, WalletCenter)
    if err != nil {
        return err
    }
    if balance.Available < req.Amount {
        return ErrInsufficientBalance // 快速失败，不进事务
    }

    // 4. 准备流水数据（在事务外组装好所有数据）
    orderNo := generateOrderNo("E") // 兑换订单号
    now := time.Now().UnixMilli()

    // ═══════════ 事务之内（必须快，毫秒级完成）═══════════

    return s.db.Transaction(func(tx *query.Query) error {

        // Step1: CAS扣减源币种可用余额（参见第2章）
        affected, err := s.balanceRepo.DeductAvailable(ctx, tx,
            req.UserId, req.FromCurrency, WalletCenter, req.Amount)
        if err != nil {
            return err
        }
        if affected == 0 {
            return ErrInsufficientBalance // 事务内精确校验
        }

        // Step2: 确保BSB余额行存在（参见第2章 2.5节）
        if err := s.balanceRepo.EnsureBalanceRow(ctx, tx,
            req.UserId, "BSB", WalletCenter); err != nil {
            return err
        }

        // Step3: 增加BSB center余额
        _, err = s.balanceRepo.AddAvailable(ctx, tx,
            req.UserId, "BSB", WalletCenter, toAmount)
        if err != nil {
            return err
        }

        // Step4: 增加BSB reward余额（赠送部分）
        if bonusAmount > 0 {
            if err := s.balanceRepo.EnsureBalanceRow(ctx, tx,
                req.UserId, "BSB", WalletReward); err != nil {
                return err
            }
            _, err = s.balanceRepo.AddAvailable(ctx, tx,
                req.UserId, "BSB", WalletReward, bonusAmount)
            if err != nil {
                return err
            }
        }

        // Step5: 创建兑换订单
        order := &model.ExchangeOrder{
            OrderNo:      orderNo,
            UserID:       req.UserId,
            FromCurrency: req.FromCurrency,
            ToCurrency:   "BSB",
            FromAmount:   req.Amount,
            ToAmount:     toAmount,
            BonusAmount:  bonusAmount,
            Rate:         rate.Rate,
            Status:       OrderStatusSuccess,
        }
        if err := tx.ExchangeOrder.WithContext(ctx).Create(order); err != nil {
            return err
        }

        // Step6: 写3条流水（参见第7章 三件套）
        flows := s.buildExchangeFlows(req, order, balance, toAmount, bonusAmount, now)
        if err := tx.BalanceFlow.WithContext(ctx).CreateInBatches(flows, len(flows)); err != nil {
            return err
        }

        return nil
    })

    // ═══════════ 事务之后（异步操作）═══════════
    // 无需额外操作（兑换不跨模块）
}
```

## 5.3 事务内禁止清单

```
事务内绝对不允许的操作：

  ✗ RPC调用（Kitex Client）
    → 超时3秒 = 连接占用3秒 = 灾难
    → 所有RPC必须在事务开始前或提交后调用

  ✗ Redis操作
    → Redis超时可能几百毫秒
    → 缓存更新放在事务COMMIT后（defer/goroutine）

  ✗ HTTP请求（任何外部网络IO）
    → 同RPC，不可预测的延迟

  ✗ 大量日志打印（高频场景下）
    → 同步写日志可能有磁盘IO延迟
    → 事务内日志打印应该最小化

  ✗ 文件IO
    → 磁盘操作延迟不可控

事务内允许的操作：
  ✓ SQL查询（SELECT）
  ✓ SQL更新（UPDATE/INSERT/DELETE）
  ✓ 纯CPU计算（金额计算、条件判断）
  ✓ 内存操作（组装对象、填充字段）
```

## 5.4 提现的特殊事务边界

```
提现与兑换不同：兑换是内部闭环（一个事务搞定），提现需要调财务RPC

正确的事务边界：

  ┌── 事务1 ──────────────────────────────────────┐
  │ Step1: CAS冻结余额（available--, frozen++）      │
  │ Step2: 创建提现订单（status=处理中）              │
  │ Step3: 写冻结流水（含before/after）               │
  └────────────────────────────────────────────────┘
       │
       │ COMMIT成功
       │
       ▼ 事务之外
  Step4: [RPC] 调财务.SubmitWithdrawalAudit
       │
       ├─ 成功 → 完成（等待财务回调）
       └─ 失败 → 补偿（参见第6章）
           ┌── 事务2（补偿事务）────────────────────┐
           │ Step5: CAS解冻（frozen--, available++）  │
           │ Step6: 更新订单状态（status=失败）        │
           │ Step7: 写解冻流水                        │
           └────────────────────────────────────────┘

关键：Step4(RPC) 在事务1的COMMIT之后
  不是在事务1内部调RPC
  RPC失败时启动新的事务2做补偿
```

## 5.5 连接池配置

```
现有工程参考值：
  MaxOpenConns  = 20    最大打开连接数
  MaxIdleConns  = 10    最大空闲连接数
  ConnMaxLifetime = 1h  连接最大存活时间
  ConnMaxIdleTime = 10m 空闲连接最大存活时间

计算：
  每个事务占用连接 ~5ms（5条SQL）
  20个连接 → 每秒处理 20/0.005 = 4000 个事务
  远超万级DAU的实际并发量

如果事务内有RPC（错误做法）：
  每个事务占用连接 ~3000ms
  20个连接 → 每秒处理 20/3 ≈ 6.7 个事务
  差距 ~600 倍 → 这就是铁律存在的原因
```

---

# 第6章：分布式一致性工程 — 本地补偿完整实现

> 方案选型结论（详见技术挑战/c-5.md 第3.1节）：本地补偿+人工补单兜底，不引入Seata/DTM/TCC
> 本章重点：4个具体补偿场景的完整代码、补偿失败的兜底、定时对账设计

## 6.1 补偿场景总览

```
场景    │ 触发条件                     │ 补偿动作          │ 复杂度
────────────────────────────────────────────────────────────────────
场景1   │ 充值创建→财务RPC失败          │ 标记订单失败       │ ★☆☆
场景2   │ 提现冻结→财务审核RPC失败      │ 解冻+标记失败      │ ★★★（最关键）
场景3   │ 提现冻结→财务审核RPC超时      │ 视为失败解冻       │ ★★☆
场景4   │ 财务回调→钱包DB异常          │ 人工补单           │ ★★☆
```

## 6.2 场景1：充值创建→财务RPC失败

```go
func (s *DepositService) CreateDeposit(ctx context.Context, req *DepositRequest) error {
    // Step1: 本地创建充值订单（status=待支付）
    order := &model.DepositOrder{
        OrderNo:  generateOrderNo("D"),
        UserID:   req.UserId,
        Amount:   req.Amount,
        Currency: req.Currency,
        Status:   OrderStatusPending,
    }
    if err := s.orderRepo.Create(ctx, order); err != nil {
        return err
    }

    // Step2: 调财务创建渠道订单
    channelResp, err := s.financeRPC.MatchAndCreateChannelOrder(ctx, &finance.CreateOrderReq{
        PlatformOrderNo: order.OrderNo,
        Amount:          req.Amount,
        Currency:        req.Currency,
        PayMethod:       req.PayMethod,
    })
    if err != nil {
        // ★ 补偿：标记本地订单为失败
        // 此时没有动过余额，只需标记订单状态
        _ = s.orderRepo.UpdateStatus(ctx, order.ID, OrderStatusFailed)
        return fmt.Errorf("create channel order failed: %w", err)
    }

    // Step3: 更新本地订单（保存渠道订单号、支付信息）
    s.orderRepo.UpdateChannelInfo(ctx, order.ID, channelResp.ChannelOrderNo, channelResp.PayUrl)
    return nil
}

// 为什么简单？因为充值是"先支付后入账"
// 财务RPC失败时，用户还没有付钱，余额也没有变动
// 只需要把本地订单标记为失败即可，无资金影响
```

## 6.3 场景2：提现冻结→财务审核RPC失败（★★★ 最关键）

```go
func (s *WithdrawService) SubmitWithdraw(ctx context.Context, req *WithdrawRequest) error {

    // ═══ Step1: 冻结余额（事务内）═══
    var order *model.WithdrawOrder
    err := s.db.Transaction(func(tx *query.Query) error {
        // CAS冻结余额
        affected, err := s.balanceRepo.FreezeBalance(ctx, tx,
            req.UserId, req.Currency, WalletCenter, req.Amount)
        if err != nil {
            return err
        }
        if affected == 0 {
            return ErrInsufficientBalance
        }

        // 创建提现订单
        order = &model.WithdrawOrder{
            OrderNo:  generateOrderNo("W"),
            UserID:   req.UserId,
            Amount:   req.Amount,
            Currency: req.Currency,
            Fee:      req.Fee,
            Status:   OrderStatusProcessing,
        }
        if err := tx.WithdrawOrder.WithContext(ctx).Create(order); err != nil {
            return err
        }

        // 写冻结流水
        flow := s.buildFreezeFlow(req, order)
        return tx.BalanceFlow.WithContext(ctx).Create(flow)
    })
    if err != nil {
        return err // 冻结失败，直接返回（无需补偿）
    }

    // ═══ Step2: 调财务提交审核（事务外！）═══
    err = s.financeRPC.SubmitWithdrawalAudit(ctx, &finance.AuditReq{
        PlatformOrderNo: order.OrderNo,
        UserId:          req.UserId,
        Amount:          req.Amount,
        Currency:        req.Currency,
        AccountInfo:     req.AccountInfo,
    })
    if err != nil {
        // ★★★ 关键补偿：冻结成功但审核RPC失败
        // 必须解冻，否则用户的钱永远卡住
        s.compensateUnfreeze(ctx, order, req)
        return fmt.Errorf("submit audit failed (compensated): %w", err)
    }

    return nil
}

// compensateUnfreeze — 补偿解冻
func (s *WithdrawService) compensateUnfreeze(
    ctx context.Context, order *model.WithdrawOrder, req *WithdrawRequest,
) {
    // 补偿事务：解冻 + 标记订单失败 + 写解冻流水
    compensateErr := s.db.Transaction(func(tx *query.Query) error {
        // CAS解冻
        affected, err := s.balanceRepo.UnfreezeBalance(ctx, tx,
            req.UserId, req.Currency, WalletCenter, req.Amount)
        if err != nil {
            return err
        }
        if affected == 0 {
            return fmt.Errorf("unfreeze failed: frozen amount insufficient")
        }

        // 更新订单状态
        tx.WithdrawOrder.WithContext(ctx).
            Where(tx.WithdrawOrder.ID.Eq(order.ID)).
            UpdateColumn(tx.WithdrawOrder.Status, OrderStatusFailed)

        // 写解冻流水
        flow := s.buildUnfreezeFlow(req, order)
        return tx.BalanceFlow.WithContext(ctx).Create(flow)
    })

    if compensateErr != nil {
        // ★★★ 补偿也失败了 — 最高风险！
        // 用户余额持续冻结，没有人来解冻
        // 三重兜底：
        //   1. 日志：CRITICAL级别，包含完整上下文
        //   2. 告警：推送到监控系统（钉钉/飞书/短信）
        //   3. 人工：运营通过B端"人工修正"功能解冻
        log.Error("CRITICAL: compensate unfreeze failed",
            "userId", req.UserId,
            "orderNo", order.OrderNo,
            "amount", req.Amount,
            "freezeErr", "none",
            "unfreezeErr", compensateErr,
        )
        // 触发告警（复用现有告警通道）
        alert.Critical(ctx, "withdraw_compensate_failed", map[string]interface{}{
            "userId":  req.UserId,
            "orderNo": order.OrderNo,
            "amount":  req.Amount,
        })
    }
}
```

## 6.4 场景3：提现冻结→财务审核RPC超时

```
超时的特殊性：不知道财务是否收到了

推荐策略（简单优先）：视为失败，执行补偿解冻

理由：
  1. Kitex RPC 默认超时 3 秒，超时意味着大概率网络问题
  2. 即使财务实际收到了，我们已经解冻了余额，订单标记为失败
  3. 财务回调钱包时，发现订单已失败 → 走异常处理通道
  4. 极端情况通过"定时对账"发现并人工处理

代码实现：与场景2完全一致（err != nil 包含超时错误）
```

## 6.5 场景4：财务回调→钱包DB异常

```
场景：财务确认充值成功 → 调钱包CreditWallet → 钱包DB异常（TiDB不可用等）

处理链路：
  1. 钱包返回系统错误 → 财务记录"回调失败"
  2. 财务自动重试（由财务侧决定重试策略）
  3. 重试仍失败 → 财务标记异常 → 运营人工查看
  4. 运营通过"人工补单"功能（B端已有设计）→ 调 SupplementCredit RPC
  5. 人工补单有独立的幂等键（补单号 B+16位）

关键认知：这个场景的主动权不在钱包侧
  钱包只需要保证：
  - CreditWallet 接口支持幂等重试（参见第3章）
  - SupplementCredit 接口可用（人工补单入口）
```

## 6.6 定时对账任务设计

```go
// 定时任务1：扫描异常冻结订单
// 执行频率：每5分钟
// 逻辑：查找"冻结超过30分钟且状态仍为处理中"的提现订单
//       对比财务侧是否有对应记录

func (s *ReconcileService) ScanAbnormalFreezeOrders(ctx context.Context) {
    threshold := time.Now().Add(-30 * time.Minute).UnixMilli()

    // 查找超时的处理中订单
    orders, _ := s.orderRepo.FindProcessingBefore(ctx, threshold)

    for _, order := range orders {
        // 调财务查询该订单状态
        status, err := s.financeRPC.GetWithdrawStatus(ctx, order.OrderNo)
        if err != nil {
            continue // 查询失败，下次再试
        }

        if status == "NOT_FOUND" {
            // 财务侧没有这个订单 → 确认是提交审核RPC失败的遗留
            // 执行补偿解冻
            s.compensateUnfreeze(ctx, order, ...)
            log.Warn("reconcile: unfroze orphan order", "orderNo", order.OrderNo)
        }
        // 如果财务侧有记录，说明只是回调还没到，继续等
    }
}

// 定时任务2：扫描异常充值订单
// 逻辑：查找"创建超过2小时但状态仍为处理中"的充值订单
//       调财务查询渠道订单状态

func (s *ReconcileService) ScanAbnormalDepositOrders(ctx context.Context) {
    // 类似逻辑...
    // 如果财务侧确认已到账但钱包未入账 → 标记异常 → 告警人工处理
}
```

## 6.7 为什么不用Seata/DTM

```
1. 场景简单：只涉及2个服务（钱包+财务），每个跨模块操作最多3步
   Seata/DTM是为5+服务、10+步骤的复杂编排设计的

2. 补偿逻辑明确：
   场景1：标记订单失败（一行代码）
   场景2：解冻余额（一次DB操作）
   场景4：人工补单（已有产品功能）
   用 if err != nil { compensate() } 就搞定了

3. 额外成本高：
   Seata需要TC（事务协调器）服务 → 额外部署运维
   从0学习 → 团队无经验
   增加故障点 → TC宕机影响所有事务

4. 务实决策：
   "能用代码解决的不引入框架，能用人工兜底的不自动化"
   当前规模下，人工处理每天0-1笔异常订单是完全可接受的
```

---

# 第7章：账务一致性工程 — 三件套同事务实现

> 方案选型结论（详见技术挑战/c-5.md 第3.3节）：事务内同步写流水 + before/after快照
> 本章重点：三件套的具体实现、before/after获取方式、流水连续性校验

## 7.1 三件套模式

```
每一次余额变动，以下三类数据在同一个事务中更新：

  ┌── BEGIN TX ──────────────────────────────────┐
  │ ① UPDATE user_balance（余额变动）              │
  │ ② INSERT balance_flow（流水，含before/after）   │
  │ ③ UPDATE *_order（订单状态，如有关联订单）       │
  └── COMMIT TX ─────────────────────────────────┘

三者要么全部成功，要么全部回滚。
不存在"余额改了但流水没写"或"流水写了但订单没更新"的中间状态。
```

## 7.2 before/after 快照的获取方式

```go
// 推荐方式：事务内先读余额（before），更新后计算得到after

func (s *WalletService) creditInTransaction(
    ctx context.Context, tx *query.Query,
    userId int64, currency string, walletType int32, amount int64,
    orderNo string, changeType int32,
) (*model.BalanceFlow, error) {

    // 1. 先读当前余额 → beforeBalance
    ub := tx.UserBalance
    currentBalance, err := ub.WithContext(ctx).
        Where(ub.UserID.Eq(userId)).
        Where(ub.Currency.Eq(currency)).
        Where(ub.WalletType.Eq(walletType)).
        First()
    if err != nil {
        return nil, err
    }
    beforeBalance := currentBalance.Available

    // 2. CAS增加余额（参见第2章）
    affected, err := s.balanceRepo.AddAvailable(ctx, tx, userId, currency, walletType, amount)
    if err != nil {
        return nil, err
    }
    if affected == 0 {
        return nil, fmt.Errorf("balance row not found for user %d", userId)
    }

    // 3. 写流水（含before/after快照）
    afterBalance := beforeBalance + amount
    flow := &model.BalanceFlow{
        UserID:        userId,
        Currency:      currency,
        WalletType:    walletType,
        ChangeType:    changeType,
        Amount:        amount,
        BeforeBalance: beforeBalance,
        AfterBalance:  afterBalance,
        RefOrderNo:    orderNo,
        Remark:        fmt.Sprintf("credit %d to %s/%d", amount, currency, walletType),
    }
    if err := tx.BalanceFlow.WithContext(ctx).Create(flow); err != nil {
        return nil, err
    }

    return flow, nil
}
```

## 7.3 为什么用先读后写而不是反推

```
方案A（推荐，先读后写）：
  before = SELECT available → after = before + amount
  优点：before是实际读到的值，准确可靠
  缺点：多一次SELECT查询（但事务极短，1ms内完成）

方案B（反推）：
  UPDATE成功后 → SELECT当前余额 → before = current - amount
  缺点：如果有并发操作在UPDATE和SELECT之间修改了余额，反推不准

方案C（RETURNING子句）：
  UPDATE ... RETURNING available
  缺点：MySQL不支持UPDATE RETURNING，TiDB部分支持

选择方案A：事务内多一次SELECT的性能损失可忽略（<1ms）
  而准确性是刚需——对账依赖before/after连续性
```

## 7.4 流水连续性校验

```
校验公式：
  同一用户、同一币种、同一子钱包的相邻两条流水：
  flow_N.after_balance == flow_N+1.before_balance

  如果不等 → 说明有异常（漏记流水、重复记录、并发问题）

实时校验（可选，写入时检查）：
  写流水前查上一条流水的after_balance
  if lastFlow.AfterBalance != currentFlow.BeforeBalance {
      log.Error("balance flow discontinuity detected", ...)
      // 不阻塞业务，但记录告警
  }

T+1对账（后续可做）：
  SELECT SUM(amount) FROM balance_flow
  WHERE user_id=? AND currency=? AND wallet_type=?
  → 结果应等于 user_balance.available + user_balance.frozen

  不等 → 生成异常报告 → 人工核查
```

## 7.5 不选异步写流水的理由

```
如果用消息队列异步写流水：
  余额UPDATE成功 → 发消息 → 消费者写流水

  风险：
  1. 消息丢失（MQ宕机、网络断连）→ 流水永久缺失 → 对账永远对不上
  2. 消费延迟 → 短时间内余额和流水不一致 → 查询时出现偏差
  3. 消费失败 → 需要重试+死信队列处理 → 增加系统复杂度

  同步写流水的"代价"：
  多一条INSERT语句 → 增加 ~1ms 延迟
  对于钱包操作（响应时间50-200ms），增量可忽略

  结论：简单可靠 > 性能极致，这是金融系统的价值观
```

---

# 第8章：安全工程 — 资金操作的安全防护

> 方案选型结论（详见技术挑战/c-5.md 第4.3-4.4节）：应用层AES加密 + 业务规则硬编码防刷
> 本章重点：加密存储实现、脱敏展示、身份校验、三级拦截、提现安全校验矩阵

## 8.1 敏感数据加密存储

```go
// withdraw_account 表的 account_no 字段加密存储

// 写入时加密
func (r *AccountRepo) SaveAccount(ctx context.Context, account *model.WithdrawAccount) error {
    // AES加密（复用 common/pkg/utils 已有工具）
    encryptedNo, err := utils.AESEncrypt(account.AccountNo, r.sensitiveKey)
    if err != nil {
        return fmt.Errorf("encrypt account_no: %w", err)
    }
    account.AccountNo = encryptedNo
    return r.db.WithdrawAccount.WithContext(ctx).Create(account)
}

// 读取时解密
func (r *AccountRepo) GetAccount(ctx context.Context, id int64) (*model.WithdrawAccount, error) {
    account, err := r.db.WithdrawAccount.WithContext(ctx).
        Where(r.db.WithdrawAccount.ID.Eq(id)).First()
    if err != nil {
        return nil, err
    }
    // AES解密
    decryptedNo, err := utils.AESDecrypt(account.AccountNo, r.sensitiveKey)
    if err != nil {
        return nil, fmt.Errorf("decrypt account_no: %w", err)
    }
    account.AccountNo = decryptedNo
    return account, nil
}

// 密钥来源：ETCD /slg/conf/security/sensitive_data_key
// 复用现有加密基础设施，不引入额外密钥管理系统
```

## 8.2 脱敏展示

```go
// API返回给C端时脱敏处理

func MaskBankCard(cardNo string) string {
    // 6222****1234 — 保留前4后4
    if len(cardNo) <= 8 {
        return "****"
    }
    return cardNo[:4] + strings.Repeat("*", len(cardNo)-8) + cardNo[len(cardNo)-4:]
}

func MaskEWallet(walletId string) string {
    // 09****7890 — 保留前2后4
    if len(walletId) <= 6 {
        return "****"
    }
    return walletId[:2] + strings.Repeat("*", len(walletId)-6) + walletId[len(walletId)-4:]
}

func MaskUSDTAddress(address string) string {
    // TN7S****Ks3j — 保留前4后4
    if len(address) <= 8 {
        return "****"
    }
    return address[:4] + strings.Repeat("*", len(address)-8) + address[len(address)-4:]
}
```

## 8.3 身份校验

```
C端请求的身份校验原则：
  ✗ 不信任C端传的userId参数
  ✓ 从Token中解析真实userId

实现：
  userId := tracer.GetUserId(ctx)  // 从Token中间件解析后写入context的userId
  // 后续所有操作使用这个userId，不使用请求参数中的userId

B端请求的权限校验：
  1. BackAuthMiddleware → B端Token认证
  2. CheckUrlAuth → URL级别权限校验
  3. IP白名单 → BlackCheck中间件

RPC接口的调用方认证：
  Kitex TTHeader 透传 traceId + userId
  ser-wallet 信任来自内部服务的RPC调用（通过服务发现ETCD注册的才能调用）
```

## 8.4 充值重复转账三级拦截

```go
// 需求文档5-5明确要求的三级拦截策略

func (s *DepositService) CheckDuplicateTransfer(
    ctx context.Context, userId int64, currency string, amount int64,
) (int, error) {
    // 查询：同用户 + 同币种 + 同金额 + 状态=处理中 的充值订单数
    do := s.db.DepositOrder
    count, err := do.WithContext(ctx).
        Where(do.UserID.Eq(userId)).
        Where(do.Currency.Eq(currency)).
        Where(do.Amount.Eq(amount)).
        Where(do.Status.In(OrderStatusPending, OrderStatusProcessing)).
        Count()
    if err != nil {
        return 0, err
    }

    return int(count), nil
}

// 调用方（Handler/Service层）的处理逻辑：
//   count == 0 → 通过，正常继续
//   count == 1 → 返回提醒码 DUPLICATE_REMIND（C端弹确认框"您有一笔相同金额订单正在处理，是否继续？"）
//   count == 2 → 返回警告码 DUPLICATE_WARN（C端提示"建议修改金额"）
//   count >= 3 → 返回拦截码 DUPLICATE_BLOCK（不允许继续）
```

## 8.5 提现安全校验矩阵

```
提现提交前的全量校验（并行执行，参见调用链路图/tmp-2.md 第7章）：

校验项              │ 检查方式                          │ 失败处理
──────────────────────────────────────────────────────────────────
KYC认证状态         │ [RPC] ser-kyc.KycQuery            │ 拒绝："请先完成实名认证"
户名一致性          │ 提现户名 vs KYC认证姓名            │ 拒绝："户名与实名不一致"
单笔限额            │ currency_config.withdraw_min/max  │ 拒绝："超出单笔限额"
每日限额            │ SUM(今日成功+处理中提现)           │ 拒绝："超出每日限额"
余额充足            │ user_balance.available >= amount   │ 拒绝："余额不足"
提现方式源头过滤     │ 充值方式记录 → 过滤规则           │ 缩小可选提现方式列表

过滤规则（合规要求，"资金从哪进就从哪出"）：
  只充过法币 → 只能法币提现
  只充过USDT → 只能USDT提现
  法币+USDT混合充值 → 只保留法币提现
```

## 8.6 审计日志

```
所有B端写操作必须记录审计日志：

调用方式：rpc.BLogClient().AddActLog(ctx, &blog.ActLogReq{
    UserId:   tracer.GetUserId(ctx),  // 操作人
    Module:   "wallet",               // 模块
    Action:   "update_currency",      // 动作
    TargetId: currencyId,             // 操作对象
    Before:   beforeJSON,             // 变更前快照
    After:    afterJSON,              // 变更后快照
})

需要审计的操作：
  - 新增/编辑/启禁用币种
  - 修改汇率
  - 人工加款/减款/补单
  - 调整提现限额
```

---

# 第9章：缓存工程 — Cache-Aside完整实现

> 方案选型结论（详见技术挑战/c-5.md 第4.2节）：Cache-Aside + 主动删缓存 + 短TTL
> 本章重点：读写流程实现、Key设计、TTL策略、绝不缓存清单、降级处理

## 9.1 Cache-Aside读操作实现

```go
func (r *CurrencyRepo) GetAvailableCurrencies(ctx context.Context) ([]*model.CurrencyConfig, error) {
    key := "wallet:currency:list"

    // 1. 尝试从Redis读取
    cached, err := r.redis.Get(ctx, key).Bytes()
    if err == nil {
        // 缓存命中 → 反序列化返回
        var configs []*model.CurrencyConfig
        if err := json.Unmarshal(cached, &configs); err == nil {
            return configs, nil
        }
        // 反序列化失败 → 当作缓存未命中，走DB
    }

    // 2. 缓存未命中 → 查DB
    cc := r.db.CurrencyConfig
    configs, err := cc.WithContext(ctx).
        Where(cc.Status.Eq(StatusEnabled)).
        Order(cc.SortOrder).
        Find()
    if err != nil {
        return nil, err
    }

    // 3. 写入Redis（TTL=5分钟）
    data, _ := json.Marshal(configs)
    r.redis.Set(ctx, key, data, 5*time.Minute)
    // 写入失败不影响业务，忽略错误

    return configs, nil
}
```

## 9.2 Cache-Aside写操作（删缓存）

```go
func (r *CurrencyRepo) UpdateCurrency(ctx context.Context, config *model.CurrencyConfig) error {
    // 1. 更新DB
    err := r.db.CurrencyConfig.WithContext(ctx).Save(config)
    if err != nil {
        return err
    }

    // 2. 异步删缓存（参照ser-item的ClearAllItemCache模式）
    go func() {
        r.redis.Del(context.Background(), "wallet:currency:list")
        r.redis.Del(context.Background(), "wallet:currency:detail:"+config.Code)
    }()

    return nil
}
```

## 9.3 缓存Key与TTL设计

```
Key命名规范                          │ TTL      │ 说明
─────────────────────────────────────────────────────────────
wallet:currency:list                │ 5分钟    │ 可用币种列表JSON
wallet:currency:detail:{code}       │ 5分钟    │ 单币种完整配置JSON
wallet:rate:{code}                  │ 1-3分钟  │ 单币种汇率
wallet:rate:list                    │ 1-3分钟  │ 全部汇率列表

汇率TTL比币种配置短：汇率变更频率=小时级，比币种配置（天级）频繁
```

## 9.4 绝不缓存的数据

```
以下数据绝对不使用缓存，直查DB：

  ✗ user_balance（用户余额）
    理由：高频写数据，缓存与DB几乎永远不一致
    用户看到过时余额（充值后余额没变）→ 投诉/恐慌
    走唯一键索引查DB < 1ms，无需缓存

  ✗ *_order（订单状态）
    理由：订单状态频繁变更（待支付→处理中→成功/失败）
    缓存过期前展示旧状态 → 用户困惑

  ✗ balance_flow（余额流水）
    理由：追加写入、查询必须实时
    分页查询走索引，性能充足
```

## 9.5 Redis不可用降级

```go
func (r *CurrencyRepo) GetConfigWithFallback(ctx context.Context, code string) (*model.CurrencyConfig, error) {
    key := "wallet:currency:detail:" + code

    // 尝试Redis
    cached, err := r.redis.Get(ctx, key).Bytes()
    if err == nil {
        var config model.CurrencyConfig
        if json.Unmarshal(cached, &config) == nil {
            return &config, nil
        }
    }

    // Redis不可用或未命中 → 直查DB（不区分Redis错误类型）
    config, err := r.db.CurrencyConfig.WithContext(ctx).
        Where(r.db.CurrencyConfig.Code.Eq(code)).
        First()
    if err != nil {
        return nil, err
    }

    // 尝试写回Redis（Redis不可用时写入失败，忽略）
    if data, err := json.Marshal(config); err == nil {
        _ = r.redis.Set(ctx, key, data, 5*time.Minute)
    }

    return config, nil
}

// 设计原则：Redis是加速器，不是依赖
// Redis宕机 → 所有币种配置查询回源DB → 数据量极小（<10条）→ 性能无影响
```

---

# 第10章：冻结模型工程 — 三步操作完整实现

> 参见调用链路图/tmp-2.md 第7章（提现全链路）及支付沟通/tmp-2.md 第4章（提现交互协议）
> 本章重点：三步操作的SQL实现、状态转移图、与订单状态联动、展示规则

## 10.1 冻结模型三步全景

```
user_balance表有两个金额字段：available（可用）和 frozen（冻结中）

三步操作：

  Step1: Freeze（提现申请时）
    available 减少 → frozen 增加
    语义：钱还在用户名下，但不能用了（预留给提现）

  Step2a: DeductFrozen（提现成功时）
    frozen 减少
    语义：冻结的钱真正离开系统（渠道已打款）

  Step2b: Unfreeze（提现失败/拒绝时）
    frozen 减少 → available 增加
    语义：冻结的钱退回可用（审核没通过或渠道打款失败）

全程守恒：
  total = available + frozen（始终不变，除非Freeze/DeductFrozen）
  Freeze:       total不变（available⇄frozen内部转移）
  Unfreeze:     total不变（frozen⇄available内部转移）
  DeductFrozen: total减少（frozen离开系统）
```

## 10.2 三步操作的SQL实现

```sql
-- Step1: Freeze（CAS安全）
UPDATE user_balance
SET available = available - @amount,
    frozen = frozen + @amount,
    update_at = @now
WHERE user_id = @userId
  AND currency = @currency
  AND wallet_type = @walletType
  AND available >= @amount
-- affected_rows = 0 → 可用余额不足

-- Step2a: DeductFrozen（CAS安全）
UPDATE user_balance
SET frozen = frozen - @amount,
    update_at = @now
WHERE user_id = @userId
  AND currency = @currency
  AND wallet_type = @walletType
  AND frozen >= @amount
-- affected_rows = 0 → 冻结金额不足（不应发生，如果发生说明有bug）

-- Step2b: Unfreeze（CAS安全）
UPDATE user_balance
SET frozen = frozen - @amount,
    available = available + @amount,
    update_at = @now
WHERE user_id = @userId
  AND currency = @currency
  AND wallet_type = @walletType
  AND frozen >= @amount
-- affected_rows = 0 → 冻结金额不足（不应发生）
```

## 10.3 状态转移图

```
  ┌─────────────┐
  │  正常状态     │  available > 0, frozen = 0
  │  (无冻结)     │
  └──────┬──────┘
         │ Freeze（提现申请）
         ▼
  ┌─────────────┐
  │  冻结中      │  available ↓, frozen ↑
  │  (审核等待)   │
  └──────┬──────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│ 成功出款│ │ 失败退回│
│DeductF │ │Unfreeze│
│frozen↓ │ │frozen↓ │
│        │ │avail↑  │
└────────┘ └────────┘

注意：一个用户可以同时有多笔冻结
  第一笔提现冻结100 → frozen=100, available=900
  第二笔提现冻结200 → frozen=300, available=700
  第一笔成功DeductFrozen(100) → frozen=200, available=700
  第二笔失败Unfreeze(200) → frozen=0, available=900
```

## 10.4 与订单状态的联动

```
提现订单状态      │ 触发操作       │ 余额影响
─────────────────────────────────────────────
已创建/处理中     │ Freeze         │ available--, frozen++
审核通过→出款中   │ （无余额操作）  │ 不变
出款成功          │ DeductFrozen   │ frozen--
审核驳回/出款失败 │ Unfreeze       │ frozen--, available++

关键：订单状态变更和余额操作在同一事务内
  Freeze + 订单(处理中) → 同一事务
  DeductFrozen + 订单(成功) → 同一事务（财务回调触发）
  Unfreeze + 订单(失败) → 同一事务（财务回调触发）
```

## 10.5 展示规则

```
C端钱包页面展示：
  可用余额：user_balance.available     → 可操作的（可兑换、可消费）
  冻结中：  user_balance.frozen        → 不可操作的（提现审核中）
  总资产：  available + frozen          → 用户名下总金额

操作时只看available：
  兑换校验：available >= exchangeAmount
  投注扣减：available >= betAmount
  提现冻结：available >= withdrawAmount

  frozen字段只有Freeze/DeductFrozen/Unfreeze三个操作会触碰
  其他操作完全不涉及frozen
```

## 10.6 补偿与冻结的关系

```
冻结模型的补偿本质就是触发Unfreeze：

  正常路径：Freeze → (审核) → DeductFrozen
  异常路径：Freeze → (审核RPC失败) → 补偿Unfreeze

  补偿Unfreeze和正常Unfreeze的实现完全相同
  区别只在于触发条件：
    正常Unfreeze：财务回调通知"审核驳回/出款失败"
    补偿Unfreeze：钱包检测到审核RPC失败，主动触发
```

---

# 第11章：跨机制协同 — 一次完整写操作的全流程

> 本章目的：展示第2-10章的所有核心机制在一次真实操作中如何串联工作
> 不是重复讲解，而是"把散装零件组装成一台完整机器"

## 11.1 示例一：CreditWallet（充值入账）— 展示幂等+CAS+事务+三件套

```
请求到达：财务模块RPC调用 ser-wallet.CreditWallet
参数：{orderNo: "D202603021234", userId: 100, currency: "VND", walletType: 1, amount: 1000000}

Step1: 幂等检查 [第3章]
  │ → 查 balance_flow: ref_order_no="D202603021234" AND change_type=DEPOSIT
  │ → 不存在 → 继续
  │ → (如果存在 → 返回上次的 {success, newBalance, flowId})

Step2: 参数校验
  │ → userId=100 存在？ → 查user表 ✓
  │ → currency="VND" 启用？ → 查currency_config(缓存) ✓ [第9章]
  │ → amount=1000000 > 0？ ✓ [第4章 精度: VND precision=0, 1000000即100万VND]
  │ → walletType=1 (center) 合法？ ✓

Step3: 确保余额行存在 [第2章 2.5节]
  │ → INSERT IGNORE user_balance (userId=100, currency="VND", walletType=1)
  │ → 已存在则忽略（幂等安全）

Step4: 开始事务 [第5章 事务边界]
  │ BEGIN TX
  │   ├─ 读当前余额 → beforeBalance = 5000000 [第7章 快照]
  │   │
  │   ├─ CAS增加余额 [第2章 场景2]
  │   │   UPDATE user_balance SET available = available + 1000000
  │   │   WHERE user_id=100 AND currency='VND' AND wallet_type=1
  │   │   → affected_rows = 1 ✓
  │   │
  │   ├─ 写流水 [第7章 三件套 + 第3章 第二层幂等]
  │   │   INSERT balance_flow (
  │   │     user_id=100, currency='VND', wallet_type=1,
  │   │     change_type=DEPOSIT, amount=1000000,
  │   │     before_balance=5000000, after_balance=6000000,
  │   │     ref_order_no='D202603021234'
  │   │   )
  │   │   → 唯一键 (ref_order_no, change_type) 保证幂等兜底
  │   │
  │   └─ 更新充值订单状态 [第7章 三件套]
  │       UPDATE deposit_order SET status=SUCCESS
  │       WHERE order_no='D202603021234'
  │
  │ COMMIT TX

Step5: 返回结果
  │ → {success: true, newBalance: 6000000, flowId: 12345}

整个过程涉及的章节：
  第2章(CAS) + 第3章(幂等双层) + 第4章(精度) + 第5章(事务边界)
  + 第7章(三件套+快照) + 第9章(缓存查币种)
```

## 11.2 示例二：提现Submit — 展示冻结+补偿+事务边界

```
请求到达：C端用户提交提现
参数：{userId: 100, currency: "VND", amount: 500000, method: "bank", accountInfo: {...}}

Step1: 安全校验（并行）[第8章]
  │ goroutine 1: KYC校验 → [RPC] ser-kyc.Query → 已认证 ✓
  │ goroutine 2: 户名校验 → 提现户名 vs KYC姓名 → 一致 ✓
  │ goroutine 3: 限额校验 → amount在min/max范围 + 今日未超限 ✓
  │ goroutine 4: 余额校验 → available >= 500000 ✓
  │ → 全部通过 → 继续（任一失败 → 聚合错误返回）

Step2: 事务1 — 冻结+创建订单+写流水 [第10章 + 第5章]
  │ BEGIN TX
  │   ├─ CAS冻结 [第10章 Step1 + 第2章 场景3]
  │   │   UPDATE user_balance SET available=available-500000, frozen=frozen+500000
  │   │   WHERE ... AND available >= 500000
  │   │   → affected_rows = 1 ✓
  │   │
  │   ├─ 创建提现订单
  │   │   INSERT withdraw_order (orderNo='W...', status=PROCESSING, ...)
  │   │
  │   └─ 写冻结流水 [第7章]
  │       INSERT balance_flow (change_type=WITHDRAW_FREEZE, amount=500000, ...)
  │ COMMIT TX

Step3: 调财务审核RPC（事务外！）[第5章 铁律 + 第6章 补偿]
  │ → financeRPC.SubmitWithdrawalAudit(orderNo, ...)
  │
  │ ├─ 成功 → 完成，等待财务回调
  │ │
  │ └─ 失败 → ★★★ 触发补偿 [第6章 场景2]
  │     Step4: 事务2 — 补偿解冻
  │     │ BEGIN TX
  │     │   ├─ CAS解冻 [第10章 Step2b]
  │     │   ├─ 更新订单状态为失败
  │     │   └─ 写解冻流水
  │     │ COMMIT TX
  │     │
  │     └─ 补偿也失败？→ 日志+告警+人工处理 [第6章 三重兜底]
```

## 11.3 示例三：兑换Execute — 展示多表事务+CAS

```
省略校验步骤，聚焦事务内操作：

事务外：
  → 获取汇率 [第9章 缓存]
  → 计算金额 [第4章 精度]
  → 预校验余额

事务内（5条SQL，~5ms）[第5章 模板]：
  1. CAS扣减 VND center available [第2章 场景1]
  2. CAS增加 BSB center available [第2章 场景2]
  3. CAS增加 BSB reward available [第2章 场景2]
  4. INSERT exchange_order [第7章 三件套]
  5. INSERT 3条 balance_flow (VND扣减+BSB增加+BSB奖励) [第7章 三件套]

涉及表：user_balance(3次UPDATE) + exchange_order(1次INSERT) + balance_flow(3次INSERT)
总SQL数：7条 → 事务耗时 ~3-7ms
```

## 11.4 错误处理统一模式

```
所有写操作共用的错误处理分类：

错误类型                    │ 处理方式                        │ 涉及章节
──────────────────────────────────────────────────────────────────────
余额不足(affected_rows=0)  │ 返回 ErrInsufficientBalance     │ 第2章
幂等命中(流水已存在)        │ 返回上次成功结果                 │ 第3章
唯一键冲突(并发穿透)        │ 视为幂等成功                    │ 第3章
DB异常(连接/超时)           │ 事务自动回滚，返回系统错误       │ 第5章
RPC异常(事务外)            │ 触发补偿逻辑                    │ 第6章
参数校验失败               │ 返回明确业务错误码               │ 第8章
精度溢出(理论不触发)        │ 返回系统错误                    │ 第4章

关键原则：
  业务错误 → 明确错误码，不重试（余额不足不会因为重试变够）
  幂等命中 → 返回成功（对调用方透明）
  系统错误 → 让调用方决定是否重试（我们保证幂等安全）
```

---

# 第12章：实现验证矩阵与开发指南

## 12.1 按开发阶段的验证路径

```
P2期：币种配置CRUD + 余额查询
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  验证核心机制：
  ✓ 缓存一致性 [第9章]：修改配置后查询返回新值
  ✓ 精度字段不可变：尝试修改precision → 应被拒绝

  测试用例：
  - B端创建币种 → Redis缓存生成 → C端查询命中缓存
  - B端修改币种 → Redis缓存删除 → C端查询回源DB → 新缓存生成
  - 查余额（余额行不存在时返回0）

P3期：兑换端到端（★★★ 关键烟囱测试）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  验证核心机制：
  ✓ CAS并发安全 [第2章]：10并发同时兑换
  ✓ 精度计算 [第4章]：VND→BSB的换算准确性
  ✓ 事务原子性 [第5章]：中途模拟异常 → 全回滚
  ✓ 三件套一致性 [第7章]：余额 == SUM(流水)
  ✓ before/after连续性 [第7章]：相邻流水before==上一条after

  测试用例：
  - 正常兑换：1,000,000 VND → BSB（验证金额计算）
  - 并发兑换：10个goroutine同时兑换同用户（验证CAS）
  - 余额不足：兑换金额>available（验证CAS拒绝）
  - 中途panic：模拟事务内异常（验证回滚）
  - 流水校验：兑换后检查3条流水的before/after连续性

P3期：RPC暴露接口（CreditWallet/DebitWallet等）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  验证核心机制：
  ✓ 幂等 [第3章]：同一orderNo调两次 → 结果相同
  ✓ 并发幂等 [第3章]：同一orderNo两个goroutine同时调
  ✓ CAS [第2章]：余额操作的正确性

  测试用例：
  - CreditWallet正常入账 → 余额增加 + 流水记录
  - CreditWallet重复调用（同orderNo）→ 幂等返回
  - CreditWallet并发调用（同orderNo）→ 只入账一次
  - DebitWallet余额不足 → 返回业务错误
  - DebitByPriority优先级扣减 → 验证扣减明细返回

P4期：提现端到端
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  验证核心机制：
  ✓ 冻结模型 [第10章]：Freeze→DeductFrozen/Unfreeze完整流程
  ✓ 补偿 [第6章]：mock财务RPC失败 → 自动解冻
  ✓ 安全校验 [第8章]：KYC/限额/户名校验

  测试用例：
  - 提现正常流程：冻结 → mock审核通过 → DeductFrozen
  - 提现拒绝流程：冻结 → mock审核拒绝 → Unfreeze
  - 补偿测试：冻结成功 → mock审核RPC失败 → 验证自动解冻
  - 安全校验：未KYC → 拒绝；超限额 → 拒绝

P4期：充值端到端
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  验证核心机制：
  ✓ 回调幂等 [第3章]：mock财务重复回调
  ✓ 三级拦截 [第8章]：重复转账拦截
  ✓ 补偿 [第6章]：创建渠道订单失败

  测试用例：
  - 充值正常流程：创建订单 → mock回调CreditWallet → 余额增加
  - 重复回调：同一订单回调两次 → 幂等（余额只增一次）
  - 三级拦截：连续创建3笔相同金额订单 → 第3笔被拦截
```

## 12.2 每个核心机制的测试方法

```
CAS并发安全：
  方法：sync.WaitGroup + N个goroutine同时调用DeductAvailable
  断言：成功笔数 × 单笔金额 ≤ 初始余额
  断言：最终余额 ≥ 0
  断言：最终余额 + 成功笔数 × 单笔金额 = 初始余额

幂等性：
  方法：同一参数连续调用2次
  断言：两次返回结果完全一致（包括newBalance和flowId）
  断言：balance_flow表只有1条记录

精度计算：
  方法：边界值测试
  断言：ToMinUnit("0", 2) == 0
  断言：ToMinUnit("99999999.99", 2) == 9999999999
  断言：ToDisplayAmount(1050, 2) == "10.50"
  断言：ToDisplayAmount(0, 0) == "0"

事务原子性：
  方法：在事务内模拟panic/返回error
  断言：所有操作回滚 → 余额不变、无流水、无订单

补偿机制：
  方法：mock financeRPC返回error
  断言：冻结后RPC失败 → 余额自动解冻
  断言：订单状态=失败
  断言：有解冻流水记录
```

## 12.3 编码检查清单（Code Review必查项）

```
每次Code Review时必须逐项检查：

  ═══ 致命级（不通过则不能合并）═══

  □ 余额操作是否用了CAS模式？
    ✗ 先SELECT再UPDATE（read-modify-write）
    ✓ 单条UPDATE带WHERE available >= ?

  □ 写操作RPC是否有幂等检查？
    ✗ 没有任何幂等机制
    ✓ 第一层(查流水) + 第二层(唯一键)

  □ 金额字段是否用int64？
    ✗ 使用float64或decimal
    ✓ int64 BIGINT最小单位

  □ RPC调用是否在事务外？
    ✗ 事务内调用RPC
    ✓ 事务外调RPC，事务内只有DB操作

  □ 余额+流水+订单是否在同一事务？
    ✗ 余额更新和流水写入分开提交
    ✓ 同一个Transaction闭包内

  ═══ 严重级（应修复后合并）═══

  □ balance_flow是否有before/after字段？
    ✗ 流水只记amount
    ✓ 流水包含before_balance和after_balance

  □ 跨模块RPC失败是否有补偿？
    ✗ RPC失败只记日志
    ✓ 有明确的补偿逻辑（解冻、标记失败等）

  □ 敏感数据（银行卡号等）是否加密存储？
    ✗ 明文存入DB
    ✓ AES加密后存储

  ═══ 重要级（建议修复）═══

  □ C端userId是否从Token提取？
    ✗ 使用请求参数中的userId
    ✓ tracer.GetUserId(ctx)

  □ 精度换算是否走统一函数？
    ✗ 业务代码中散落换算逻辑
    ✓ 统一使用 ToMinUnit/ToDisplayAmount
```

---

# 附录：核心方案速查表

## A.1 12个挑战→12个方案速查

```
#   │ 挑战              │ 方案                              │ 一句话
────────────────────────────────────────────────────────────────────────
1   │ 并发安全           │ CAS条件更新                       │ UPDATE ... WHERE available >= ?
2   │ 幂等性             │ 应用层查流水 + DB唯一键             │ 两层防护：快速拦截+绝对兜底
3   │ 金额精度           │ BIGINT最小单位整数                  │ int64存储，零浮点运算
4   │ 事务边界           │ RPC外/DB内铁律                     │ 事务5ms vs 3000ms
5   │ 账务一致性         │ 三件套同事务 + before/after          │ 余额+流水+订单原子提交
6   │ 分布式一致性       │ 本地补偿 + 人工兜底                  │ if err { compensate() }
7   │ 冻结模型           │ Freeze→DeductFrozen/Unfreeze       │ available⇄frozen三步转移
8   │ 安全防护           │ AES加密 + 脱敏 + 三级拦截            │ 加密存储、脱敏展示、规则拦截
9   │ 缓存一致性         │ Cache-Aside + 短TTL + 余额不缓存     │ 配置缓存5min，余额永不缓存
10  │ 热点账户           │ 当前不处理                          │ 万级DAU行锁够用
11  │ 高可用             │ 功能独立降级                        │ 财务不可用≠兑换不可用
12  │ 数据增长           │ 不处理，预留空间                     │ TiDB千万级无压力
```

## A.2 核心SQL速查（4种CAS场景）

```sql
-- 扣减：UPDATE SET available = available - ? WHERE available >= ?
-- 增加：UPDATE SET available = available + ?
-- 冻结：UPDATE SET available = available - ?, frozen = frozen + ? WHERE available >= ?
-- 解冻：UPDATE SET frozen = frozen - ?, available = available + ? WHERE frozen >= ?
-- 扣冻：UPDATE SET frozen = frozen - ? WHERE frozen >= ?
```

## A.3 关键Go函数签名速查

```go
// 精度换算
utils.ToMinUnit(displayAmount string, precision int32) (int64, error)
utils.ToDisplayAmount(minUnit int64, precision int32) string
utils.SafeAdd(a, b int64) (int64, error)

// 余额操作
balanceRepo.DeductAvailable(ctx, tx, userId, currency, walletType, amount) (affected int64, err error)
balanceRepo.AddAvailable(ctx, tx, userId, currency, walletType, amount) (affected int64, err error)
balanceRepo.FreezeBalance(ctx, tx, userId, currency, walletType, amount) (affected int64, err error)
balanceRepo.UnfreezeBalance(ctx, tx, userId, currency, walletType, amount) (affected int64, err error)
balanceRepo.DeductFrozen(ctx, tx, userId, currency, walletType, amount) (affected int64, err error)
balanceRepo.EnsureBalanceRow(ctx, tx, userId, currency, walletType) error

// 幂等检查
flowRepo.FindByOrderAndType(ctx, orderNo string, changeType int32) (*BalanceFlow, error)
isDuplicateKeyError(err error) bool
```

## A.4 编码铁律速查（5条不可违反）

```
铁律1：余额操作必须CAS — UPDATE WHERE available >= ?，不是先读后写
铁律2：写操作必须幂等 — 应用层查流水 + DB唯一键双层
铁律3：金额必须int64  — 全链路无float64，展示时才转字符串
铁律4：RPC在事务外    — 事务内只有DB操作，微秒级完成
铁律5：三件套同事务    — 余额+流水+订单原子提交，不允许分开
```

---

> **文档结束**
>
> 本文档共覆盖：12章 + 1附录
> - 12个核心工程挑战的完整实现蓝图
> - 4种CAS场景的SQL + Go伪代码
> - 3个完整业务示例（充值入账/提现/兑换）展示跨机制协同
> - 4个补偿场景的完整代码
> - 按P2/P3/P4开发阶段的验证矩阵
> - Code Review必查清单（9项：5致命+2严重+2重要）
> - 5条编码铁律速查
