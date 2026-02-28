# ser-wallet 架构设计方案

> 编写时间：2026-02-26
> 设计原则：安全、高并发、高性能、不过度设计、业界主流
> 工程基础：CloudWeGo Hertz/Kitex + GORM/Gen + TiDB + Redis + ETCD + NATS
> 业界参考：Stripe Ledger、Modern Treasury、Airbnb Orpheus、蚂蚁金服 TCC、Square Cash App
> 依据来源：
> - 工程分析：/Users/mac/gitlab 全工程深度代码分析
> - 需求分析：/Users/mac/gitlab/z-readme/result/tmp-2.md
> - 流程分析：/Users/mac/gitlab/z-readme/流程思路/tmp-2.md
> - 依赖分析：/Users/mac/gitlab/z-readme/依赖关系/tmp-2.md
> - 实现思路：/Users/mac/gitlab/z-readme/实现思路/tmp-2.md

---

## 一、架构全景

### 1.1 三模块宏观定位

钱包、多币种配置、财务管理是同一个"资金域"的三个子系统。从架构层面看，它们构成一个完整的资金处理体系：

```
                          ┌─────────────────────────────────────┐
                          │          资金域全景（钱相关的整体）     │
                          │                                     │
  ┌───────────────────────┼─────────────────────────────────────┼─────────────────────────┐
  │                       │                                     │                         │
  │   币种配置层            │        钱包核心层                     │      支付通道层           │
  │   (Currency Config)   │        (Wallet Core)                │      (Payment Channel)  │
  │                       │                                     │                         │
  │   · 币种注册与管理      │   · 账户模型（余额管理）               │   · 代收通道（充值入口）   │
  │   · 汇率获取与计算      │   · 账变记录（流水追踪）               │   · 代付通道（提现出口）   │
  │   · 汇率阈值更新机制    │   · 冻结/解冻/扣除机制                │   · 通道轮询匹配算法     │
  │   · 精度与显示规则      │   · 消费优先级策略                    │   · 回调接收与验签       │
  │                       │   · 幂等与并发控制                    │   · 支付方式配置         │
  │   【我方·基础设施】     │   · 兑换引擎                        │   · 订单管理与审核       │
  │                       │                                     │                         │
  │                       │   【我方·核心业务】                    │   【他方·财务管理】       │
  └───────────────────────┼─────────────────────────────────────┼─────────────────────────┘
                          │                                     │
                          └──────────── RPC 合同 ───────────────┘
```

**核心架构原则**：

| 原则 | 说明 | 业界依据 |
|------|------|---------|
| **服务自治** | 每个微服务独立数据库，不共享表，通过 RPC 交互 | 微服务基本原则 |
| **接口即合同** | Thrift IDL 定义的 RPC 接口是模块间的唯一交互方式 | CloudWeGo + IDL先行 |
| **数据归属明确** | 谁的数据谁管理 — 余额归钱包，通道归财务，币种归钱包 | DDD 界限上下文 |
| **账本不可篡改** | 账变流水只追加不修改，余额修正通过反向流水实现 | Stripe Ledger |
| **余额可推导** | 余额 = 初始值 + SUM(所有流水)，物化余额只是缓存加速 | Modern Treasury |
| **操作可重入** | 所有资金操作基于幂等键可安全重试 | Stripe/Airbnb |

### 1.2 系统分层架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           接入层 (Gateway)                               │
│                                                                         │
│  gate-font (C端)                          gate-back (B端)               │
│  ├── BlackCheck (IP黑名单)                 ├── WhiteCheck (IP白名单)      │
│  ├── FontAuthMiddleware (Token鉴权)        ├── BackAuthMiddleware (Token)│
│  ├── 路由分发 → RPC 代理                    ├── RBAC (CheckUrlAuth)       │
│  └── 统一响应 R{TraceId,Code,Msg,Data}     └── 路由分发 → RPC 代理       │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                           服务层 (Service)                               │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    ser-wallet (钱包服务)                          │    │
│  │                                                                 │    │
│  │  handler.go ── 薄层入口，零业务逻辑                                │    │
│  │      │                                                          │    │
│  │      ▼                                                          │    │
│  │  internal/ser/ ── 业务逻辑层                                     │    │
│  │      │   ├── wallet_core_ser  ── 余额操作核心引擎                  │    │
│  │      │   ├── currency_ser     ── 币种配置管理                     │    │
│  │      │   ├── exchange_ser     ── 兑换引擎                        │    │
│  │      │   ├── deposit_ser      ── 充值业务编排                     │    │
│  │      │   ├── withdraw_ser     ── 提现业务编排                     │    │
│  │      │   └── record_ser       ── 账变记录查询                     │    │
│  │      │                                                          │    │
│  │      ▼                                                          │    │
│  │  internal/rep/ ── 数据访问层（GORM Gen 生成 + 自定义）              │    │
│  │  internal/cache/ ── 缓存访问层（Redis 封装）                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │  ser-finance  │  │  ser-kyc     │  │  ser-user    │                  │
│  │  (财务管理)    │  │  (KYC认证)   │  │  (用户服务)   │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                           数据层 (Data)                                  │
│                                                                         │
│  TiDB (MySQL兼容)        Redis               ETCD                      │
│  ├── wallet_account      ├── 幂等键           ├── 服务发现               │
│  ├── wallet_transaction  ├── 币种/汇率缓存    ├── 配置中心               │
│  ├── currency_config     ├── 限额计数器       └── 配置热更新             │
│  ├── exchange_rate_log   ├── 分布式锁                                   │
│  ├── deposit_order       └── Token缓存       NATS/JetStream            │
│  ├── withdraw_order                          ├── 异步事件通知            │
│  ├── exchange_order                          └── 延迟消息               │
│  ├── withdraw_account                                                   │
│  └── audit_requirement                                                  │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                           外部依赖层 (External)                          │
│                                                                         │
│  三方汇率API ×3          三方支付通道（代收/代付）    区块链监控服务        │
│  (定时拉取,取平均值)      (财务管理模块负责对接)      (USDT到账监控)       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、账户模型设计

这是整个架构的核心。账户模型的设计直接决定了系统的安全性、正确性和可扩展性。

### 2.1 设计选型：实用主义账户模型

**业界有两种主流做法**：

| 方案 | 说明 | 代表 | 适用场景 |
|------|------|------|---------|
| **完整复式记账** | 每笔操作产生借贷双方分录，所有分录之和 = 0 | Stripe Ledger、Modern Treasury | 大型支付平台、需要完整会计准则 |
| **物化余额 + 不可变流水** | 账户表存余额（物化），流水表只追加，余额 = SUM(流水) 可推导验证 | 多数业务钱包系统、中型平台 | 业务钱包、游戏币、积分系统 |

**本项目选择方案二：物化余额 + 不可变流水**

理由：
1. 我们是业务钱包（用户资产管理），不是专业会计系统
2. 完整复式记账需要维护大量"对手方账户"（平台收入账户、手续费账户、清算账户等），当前需求没有这个维度
3. 物化余额 + 不可变流水已经能覆盖：余额查询 O(1)、审计追踪、对账校验
4. 符合"不过度设计"原则，但保留了向复式记账演进的能力

**关键约束（确保安全）**：
- 流水表只有 INSERT，永远没有 UPDATE 和 DELETE
- 余额修正通过"反向流水"实现，不直接改余额字段
- 定时对账：`当前余额 == SUM(所有流水)` 必须恒等

### 2.2 账户模型核心结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         账户模型                                     │
│                                                                     │
│  一个"账户" = (user_id, currency_code, wallet_type) 的唯一组合       │
│                                                                     │
│  用户 U001                                                          │
│  ├── BSB                                                            │
│  │   ├── 中心钱包账户  ── available: 1000, frozen: 200              │
│  │   ├── 奖励钱包账户  ── available: 500,  frozen: 0               │
│  │   ├── 主播钱包账户  ── available: 0,    frozen: 0               │
│  │   ├── 代理钱包账户  ── available: 0,    frozen: 0  (预留)        │
│  │   └── 场馆钱包账户  ── available: 0,    frozen: 0  (预留)        │
│  ├── VND                                                            │
│  │   ├── 中心钱包账户  ── available: 5000000, frozen: 0            │
│  │   ├── 奖励钱包账户  ── available: 200000,  frozen: 0            │
│  │   └── ...                                                        │
│  ├── IDR                                                            │
│  │   └── ...                                                        │
│  └── THB                                                            │
│      └── ...                                                        │
│                                                                     │
│  关键：每个账户独立维护余额，币种天然隔离，锁粒度精确到单个账户        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**业界依据**：Modern Treasury、Rapyd、Airwallex 等多币种钱包均采用**币种维度独立账户**模型。好处是：
- 每个币种独立余额计算，无需 `GROUP BY currency` 聚合
- 并发锁粒度精确到 `(user, currency, wallet_type)`，不同币种操作互不阻塞
- 兑换操作本质是"A 账户出金 + B 账户入金"，模型天然支持

### 2.3 账户表设计

```sql
CREATE TABLE wallet_account (
    id              BIGINT       PRIMARY KEY AUTO_INCREMENT,
    user_id         BIGINT       NOT NULL,
    currency_code   VARCHAR(10)  NOT NULL,           -- BSB, VND, IDR, THB
    wallet_type     TINYINT      NOT NULL,           -- 1=中心, 2=奖励, 3=主播, 4=代理, 5=场馆
    available_amt   DECIMAL(20,4) NOT NULL DEFAULT 0, -- 可用余额
    frozen_amt      DECIMAL(20,4) NOT NULL DEFAULT 0, -- 冻结余额
    version         BIGINT       NOT NULL DEFAULT 0,  -- 乐观锁版本号（备用）
    create_at       BIGINT       NOT NULL DEFAULT 0,
    update_at       BIGINT       NOT NULL DEFAULT 0,

    UNIQUE INDEX uk_user_currency_type (user_id, currency_code, wallet_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**设计说明**：

| 字段 | 说明 |
|------|------|
| `available_amt` | 可用余额 — 用户可以花的钱 |
| `frozen_amt` | 冻结余额 — 已被提现/场馆等操作锁定，不可花 |
| 无 `total_amt` | **不存总余额字段**，总余额 = available + frozen，由代码计算，减少数据冗余和不一致风险 |
| `version` | 乐观锁版本号。虽然主方案用悲观锁（FOR UPDATE），但保留此字段用于特定场景的乐观锁降级 |
| `DECIMAL(20,4)` | 覆盖法定货币（最多2位小数）和平台币（2位小数）。加密货币如需8位，调整为 `DECIMAL(20,8)` — **此精度需与财务模块对齐确认** |

**为什么不用 BIGINT 存最小单位**：
- 不同币种最小单位不同（VND 无小数、THB 2位、加密 8位），统一为 BIGINT 需要额外的精度元数据
- DECIMAL 是金融系统的标准做法（Stripe、Modern Treasury 都用 NUMERIC/DECIMAL）
- TiDB 完全兼容 MySQL 的 DECIMAL 运算，无精度丢失

### 2.4 账变流水表设计（不可变账本）

```sql
CREATE TABLE wallet_transaction (
    id              BIGINT       PRIMARY KEY,        -- Snowflake ID
    user_id         BIGINT       NOT NULL,
    currency_code   VARCHAR(10)  NOT NULL,
    wallet_type     TINYINT      NOT NULL,
    tx_type         VARCHAR(30)  NOT NULL,           -- 变动类型枚举
    direction       TINYINT      NOT NULL,           -- 1=收入(+), 2=支出(-)
    amount          DECIMAL(20,4) NOT NULL,          -- 变动金额（绝对值）
    before_amt      DECIMAL(20,4) NOT NULL,          -- 变动前可用余额
    after_amt       DECIMAL(20,4) NOT NULL,          -- 变动后可用余额
    frozen_change   DECIMAL(20,4) NOT NULL DEFAULT 0,-- 冻结变动（正=冻结增加，负=冻结减少）
    order_no        VARCHAR(50)  NOT NULL,           -- 关联业务订单号
    op_type         VARCHAR(30)  NOT NULL,           -- 操作类型（充值/提现/兑换/加款/减款...）
    remark          VARCHAR(255) NOT NULL DEFAULT '',
    create_at       BIGINT       NOT NULL,

    -- 幂等兜底：同一订单+同一操作类型+同一钱包类型 不可重复
    UNIQUE INDEX uk_order_op_wallet (order_no, op_type, wallet_type),
    -- 查询索引：用户+币种+类型 分页查
    INDEX idx_user_currency_type (user_id, currency_code, tx_type, create_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**不可变原则**：此表只有 INSERT，没有 UPDATE 和 DELETE。这是架构层面的强制约束。

**before_amt / after_amt 的作用**：
- 审计追踪：任何一笔流水都能看到"操作前多少、操作后多少"
- 对账验证：第 N 条流水的 `after_amt` 必须等于第 N+1 条的 `before_amt`
- 问题排查：数据异常时可以通过流水链逐条追溯
- 业界依据：这是 Stripe、支付宝等金融系统的标准做法

**幂等唯一索引 `(order_no, op_type, wallet_type)`**：
- 为什么是三元组而非 order_no 单字段？因为人工加款（A+16位订单号）可能同时操作多个钱包类型，同一 order_no 下会有多条流水（中心钱包一条、奖励钱包一条），它们的 wallet_type 不同
- 这是 Redis 幂等键之外的数据库层兜底

---

## 三、并发控制架构

余额操作是整个系统中并发最敏感的部分。设计不当会导致超卖（余额扣成负数）或重复入账。

### 3.1 方案选型

**业界三种主流方案对比**：

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **悲观锁 SELECT FOR UPDATE** | 事务内先锁行，再读写 | 强一致，无重试成本 | 持锁时间影响吞吐 | 写多读少、冲突频繁 |
| **乐观锁 Version** | 读时记版本号，写时比对 | 无锁等待，读不阻塞 | 高冲突时重试成本高 | 读多写少、冲突罕见 |
| **Redis 分布式锁** | SetNX 获取全局锁 | 跨库跨服务 | Redis 故障风险 | 跨数据源操作 |

**本项目选择：悲观锁（SELECT FOR UPDATE）为主 + Redis 幂等键为辅**

理由：
1. 余额操作是**写密集**场景（每次充值/消费/提现都要改余额），乐观锁的重试成本过高
2. 所有余额操作在同一数据库（TiDB）同一事务内完成，不需要分布式锁
3. TiDB 完全支持 `SELECT ... FOR UPDATE` 行级锁，与 MySQL 行为一致
4. Redis SetNX 用于幂等快速拦截，不参与余额操作本身

### 3.2 锁定策略

```
余额操作的标准执行序列（所有 5 个核心方法共享）：

┌──────────────────────────────────────────────────────────────────┐
│  Step 1: 幂等快检（Redis）                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  key = "wal:idem:{order_no}:{op_type}:{wallet_type}"      │  │
│  │  SetNX(key, "1", 24h)                                     │  │
│  │  ├── 返回 false → 重复请求，直接返回成功（幂等）             │  │
│  │  └── 返回 true  → 首次请求，继续执行                        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Step 2: 开启数据库事务                                           │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  BEGIN                                                     │  │
│  │                                                            │  │
│  │  Step 3: 行锁定                                            │  │
│  │  SELECT available_amt, frozen_amt                          │  │
│  │  FROM wallet_account                                       │  │
│  │  WHERE user_id=? AND currency_code=? AND wallet_type=?     │  │
│  │  FOR UPDATE;                                               │  │
│  │  -- 此时该行被锁定，其他事务等待                              │  │
│  │                                                            │  │
│  │  Step 4: 业务校验                                           │  │
│  │  ├── Credit: 无需额外校验                                   │  │
│  │  ├── Debit:  available_amt >= amount ?                     │  │
│  │  ├── Freeze: available_amt >= amount ?                     │  │
│  │  ├── Unfreeze: frozen_amt >= amount ?                      │  │
│  │  └── DeductFrozen: frozen_amt >= amount ?                  │  │
│  │                                                            │  │
│  │  Step 5: 更新余额                                           │  │
│  │  UPDATE wallet_account SET available_amt=?, frozen_amt=?   │  │
│  │  WHERE id=?;                                               │  │
│  │                                                            │  │
│  │  Step 6: 写入流水（同一事务）                                 │  │
│  │  INSERT INTO wallet_transaction (...)                       │  │
│  │  -- before_amt, amount, after_amt, order_no, op_type       │  │
│  │                                                            │  │
│  │  COMMIT                                                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Step 7: 返回结果                                                │
│  └── 返回更新后的余额                                             │
│                                                                  │
│  注意: 如果事务失败回滚，Redis 幂等键不需要删除                      │
│  因为下次重试时 Redis 返回 false，但代码会查 DB 确认流水不存在       │
│  则重置 Redis 键并重新执行                                         │
└──────────────────────────────────────────────────────────────────┘
```

**锁顺序规则（防死锁）**：

当一个操作需要同时锁定多个账户（如兑换：扣法币 + 加 BSB）时，必须按固定顺序加锁：

```
规则：按 (user_id, currency_code, wallet_type) 的字典序升序加锁

示例：兑换 VND → BSB
  先锁: (U001, BSB, center)    -- B < V，BSB 先锁
  再锁: (U001, VND, center)    -- 然后 VND

如果反过来，另一个并发兑换操作也先锁 VND 再锁 BSB，就会死锁。
固定顺序后，所有操作都是 BSB → VND，不会交叉等待。
```

### 3.3 幂等架构（双保险）

```
                        请求到达
                           │
                           ▼
                ┌─────────────────────┐
                │  第一层：Redis SetNX  │  ← 快速拦截（毫秒级）
                │  O(1) 检查           │
                │  命中 → 直接返回成功  │
                └─────────┬───────────┘
                          │ 未命中
                          ▼
                ┌─────────────────────┐
                │  第二层：DB 唯一索引  │  ← 最终兜底（事务内）
                │  INSERT 流水记录     │
                │  唯一冲突 → 返回成功  │
                └─────────┬───────────┘
                          │ 插入成功
                          ▼
                     正常执行业务

为什么需要两层？
· Redis 可能宕机或网络抖动 → 第一层失效，DB 兜底
· DB 唯一约束检查有网络往返 → Redis 在前面快速拦截 99%+ 的重复请求
· 业界标准做法（Stripe, Airbnb 均采用类似双层策略）
```

---

## 四、冻结/解冻/扣除 三步模式架构

这是提现流程的核心机制，本质上是 **TCC（Try-Confirm-Cancel）模式**的业务化实现。

### 4.1 业界 TCC 模式在本项目中的映射

| TCC 阶段 | 本项目对应 | 操作 | 触发方 |
|----------|----------|------|--------|
| **Try** | FreezeBalance | 可用 → 冻结（资源预留） | 用户提交提现时，我方内部调用 |
| **Confirm** | DeductFrozenAmount | 冻结 → 扣除（确认执行） | 出款成功后，财务模块调我方 RPC |
| **Cancel** | UnfreezeBalance | 冻结 → 可用（回滚释放） | 审核驳回时，财务模块调我方 RPC |

### 4.2 余额状态转换图

```
                    ┌─────────────────────────────────┐
                    │          wallet_account          │
                    │  available_amt   frozen_amt      │
                    └────────┬────────────────────────┘
                             │
          ┌──────────────────┼──────────────────────┐
          │                  │                      │
    ① 加款(Credit)     ② 扣款(Debit)          ③ 冻结(Freeze)
    available += N     available -= N          available -= N
                                               frozen += N
          │                  │                      │
          ▼                  ▼                      ▼
       完成(终态)        完成(终态)           ┌─────────────┐
                                             │  冻结中状态   │
                                             └──────┬──────┘
                                                    │
                                        ┌───────────┼───────────┐
                                        │                       │
                                  ④ 解冻(Unfreeze)       ⑤ 扣冻结(DeductFrozen)
                                  frozen -= N             frozen -= N
                                  available += N          (资金离开系统)
                                        │                       │
                                        ▼                       ▼
                                    恢复(终态)             完成(终态)

    特殊：⑥ 出款失败 → frozen 不变，等人工处理
```

### 4.3 三步模式的安全保障

```
安全保障一：冻结金额不参与可用余额计算
  → 用户在冻结期间不可能花掉被冻结的钱
  → 即使并发发起多笔提现，每笔都要校验 available_amt >= amount

安全保障二：出款失败不自动解冻
  → 防止通道实际已出款但回调返回失败（网络抖动）时，
    用户余额被错误释放后再次提现，导致平台资损
  → 这是业界标准的保守策略

安全保障三：每步操作都有独立的幂等键
  → FreezeBalance:       idem:{orderNo}:freeze:{walletType}
  → UnfreezeBalance:     idem:{orderNo}:unfreeze:{walletType}
  → DeductFrozenAmount:  idem:{orderNo}:deduct_frozen:{walletType}
  → 财务模块重复调用不会导致多次冻结/解冻/扣除
```

---

## 五、消费优先级架构

### 5.1 扣款策略

```
消费优先级：先中心钱包，再奖励钱包（需求 4.1.2 明确规定）

DebitByPriority(userId, currency, amount, orderNo, opType):

  1. 事务内锁定两个账户（按固定顺序）
     SELECT ... FOR UPDATE WHERE user_id=? AND currency_code=?
       AND wallet_type IN (1, 2)  -- 1=中心, 2=奖励
     ORDER BY wallet_type ASC;    -- 固定锁序

  2. 校验总余额
     center.available + reward.available >= amount

  3. 计算分配
     center_deduct = min(center.available, amount)
     reward_deduct = amount - center_deduct

  4. 更新两个账户余额

  5. 写入两条流水（各自的 before/after）

  6. 返回 { centerDeducted, rewardDeducted }
     → 这个比例信息要落库，用于后续返奖按比例拆分
```

### 5.2 返奖按比例拆分

```
投注返奖时：按原投注的扣款比例拆分返奖金额

BetReward(userId, currency, rewardAmount, betOrderNo):

  1. 查询原投注的扣款记录
     SELECT wallet_type, amount FROM wallet_transaction
     WHERE order_no = betOrderNo AND op_type = 'bet_deduct'

  2. 计算比例
     center_ratio = center_amount / total_amount
     reward_ratio = reward_amount / total_amount

  3. 拆分返奖
     center_reward = rewardAmount × center_ratio
     reward_reward = rewardAmount × reward_ratio

  4. 分别入两个钱包
     CreditBalance(userId, currency, CENTER, center_reward, ...)
     CreditBalance(userId, currency, REWARD, reward_reward, ...)
```

---

## 六、汇率管理架构

### 6.1 三层汇率体系

```
┌─────────────────────────────────────────────────────────────┐
│                       汇率三层体系                            │
│                                                             │
│  第一层：实时汇率（real_time_rate）                            │
│  ├── 来源：3 个三方汇率 API 取平均值                          │
│  ├── 更新频率：定时任务（可配置，如每 2 分钟）                  │
│  ├── 作用：反映真实市场价                                     │
│  └── 存储：currency_config 表 + Redis 缓存                   │
│                                                             │
│  第二层：平台汇率（platform_rate）                             │
│  ├── 来源：当偏差 >= 阈值% 时，从实时汇率同步                  │
│  ├── 触发机制：|实时 - 平台| / 实时 × 100% >= 阈值            │
│  ├── 作用：平台内部核算基准，不频繁变动                        │
│  └── 每次更新必须写 exchange_rate_log                         │
│                                                             │
│  第三层：入款/出款汇率（deposit_rate / withdraw_rate）         │
│  ├── 公式：                                                  │
│  │   入款汇率 = 平台汇率 × (1 + 浮动%)    ← 用户充值用      │
│  │   出款汇率 = 平台汇率 × (1 - 浮动%)    ← 用户提现用      │
│  ├── 作用：浮动差价是平台的汇兑收益                           │
│  └── 随平台汇率联动更新                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 汇率锁定机制（Quote-Lock 模式）

**业界标准**（Stripe FX Quote、Airwallex）：交易时锁定汇率，防止操作过程中汇率变动导致金额差异。

```
用户发起兑换/充值/提现（涉及汇率的操作）：

  1. 获取当前汇率（从缓存或 DB 读取）
  2. 汇率快照写入订单记录
     ├── deposit_order.rate_snapshot = 当时的入款汇率
     ├── withdraw_order.rate_snapshot = 当时的出款汇率
     └── exchange_order.rate_snapshot = 当时的平台汇率
  3. 后续所有金额计算使用订单中的快照汇率，不再查实时汇率
  4. 对账/统计也使用订单快照汇率

为什么不直接用实时汇率？
  → 从用户点"确认"到系统处理完成有时间差
  → 这个时间差内汇率可能更新了
  → 如果前后用不同汇率，用户看到的金额和实际到账不一致
  → 快照锁定保证"所见即所得"
```

### 6.3 汇率更新定时任务架构

```
                     定时触发（ETCD 配置更新间隔）
                              │
                              ▼
                   ┌─────────────────────┐
                   │ 并发调 3 个三方 API   │
                   │ API-A  API-B  API-C │
                   └─────────┬───────────┘
                             │
                    ┌────────┼────────┐
                    ▼        ▼        ▼
                  成功     失败      成功
                    │                 │
                    └────┬────────────┘
                         │
               取成功的 API 结果平均值
               (全部失败 → 保持旧值 + 告警)
                         │
                         ▼
                ┌─────────────────┐
                │ 遍历每个币种     │
                │ (BSB 跳过)      │
                └────────┬────────┘
                         │
              ┌──────────┼──────────┐
              ▼                     ▼
        平台汇率为空              平台汇率已有
        (首次运行)               计算偏差值
              │                     │
              │                ┌────┴────┐
              │                ▼         ▼
              │          偏差 < 阈值   偏差 >= 阈值
              │          不更新平台    更新平台汇率
              │                       + 入款汇率
              │                       + 出款汇率
              │                       + 写日志
              └──────┬────────────────┘
                     │
                     ▼
              刷新 Redis 缓存
```

---

## 七、跨模块交互架构

### 7.1 RPC 接口合同（IDL 层面的架构约束）

```
                    ┌──────────────────────────────────┐
                    │         IDL 合同层                 │
                    │                                  │
                    │  common/idl/ser-wallet/           │
                    │  ├── service.thrift (WalletService)│
                    │  │   ├── 供财务调用的 6 个核心方法   │
                    │  │   ├── 供游戏调用的 4 个方法       │
                    │  │   ├── C端业务方法                │
                    │  │   └── B端管理方法                │
                    │  ├── wallet.thrift   (余额结构)    │
                    │  ├── currency.thrift (币种结构)    │
                    │  └── ...                          │
                    │                                  │
                    │  common/idl/ser-finance/          │
                    │  ├── service.thrift (FinanceService)│
                    │  │   ├── 供钱包调用的 3-4 个方法    │
                    │  │   └── ...                      │
                    │  └── ...                          │
                    └──────────────────────────────────┘

合同规则：
  1. 双方按 IDL 合同开发，不直接访问对方数据库
  2. IDL 变更必须双方确认，不可单方面修改
  3. 新增字段用 optional，不删除已有字段（向后兼容）
  4. 每个 RPC 方法必须支持幂等（通过 orderNo 参数）
```

### 7.2 服务间调用拓扑

```
┌────────────────────────────────────────────────────────────────────────┐
│                        运行时调用拓扑                                    │
│                                                                        │
│                            ┌──────────┐                                │
│                            │ gate-font│                                │
│                            │ (C端网关) │                                │
│                            └────┬─────┘                                │
│                                 │ RPC                                  │
│                                 ▼                                      │
│  ┌──────────┐     RPC     ┌──────────┐     RPC     ┌──────────┐      │
│  │ser-finance│◄───────────│ser-wallet │────────────►│ ser-kyc  │      │
│  │          │────────────►│          │             └──────────┘      │
│  │ 财务调钱包:│            │          │     RPC     ┌──────────┐      │
│  │ ·Credit  │  钱包调财务: │          │────────────►│ ser-user │      │
│  │ ·Debit   │  ·PayMethod │          │             └──────────┘      │
│  │ ·Freeze  │  ·Channel   │          │                                │
│  │ ·Unfreeze│  ·Payout    │          │                                │
│  │ ·Deduct  │             │          │                                │
│  │ ·Query   │             │          │                                │
│  └──────────┘             └────┬─────┘                                │
│                                │ RPC                                  │
│                                ▼                                      │
│                          ┌──────────┐                                  │
│                          │ gate-back│                                  │
│                          │ (B端网关) │                                  │
│                          └──────────┘                                  │
│                                                                        │
│  关键：ser-wallet 和 ser-finance 是双向调用                              │
│  但中间隔着异步环节（三方回调），不会形成同步环路死锁                       │
│                                                                        │
│  充值链路: wallet→finance(同步) ... finance→wallet(异步回调触发)          │
│  提现链路: wallet(内部冻结) ... finance→wallet(审核结果触发)              │
└────────────────────────────────────────────────────────────────────────┘
```

### 7.3 双向依赖的架构解耦

钱包和财务的双向 RPC 调用是架构上必须处理的问题。解决方案：

```
方案：通过 Kitex RPC Client 解耦（现有工程已有此模式）

ser-wallet 调 ser-finance:
  → 通过 common/rpc/rpc_client.go 中的 FinanceClient() 工厂方法
  → ETCD 服务发现自动解析 ser-finance 地址
  → ser-wallet 只依赖 common/pkg/kitex_gen/ser_finance/ （IDL 生成的 stub）
  → 不依赖 ser-finance 的内部实现

ser-finance 调 ser-wallet:
  → 通过 common/rpc/rpc_client.go 中的 WalletClient() 工厂方法
  → 同样只依赖 IDL 生成的 stub

结果：两个服务在代码层面零依赖，仅通过 IDL 合同和 ETCD 服务发现交互
```

---

## 八、数据一致性架构

### 8.1 一致性分级

不是所有数据都需要最强一致性，按场景分级：

| 一致性级别 | 数据 | 保障机制 | 说明 |
|-----------|------|---------|------|
| **强一致** | 余额 + 流水 | DB 事务 + 行锁 | 同一事务内完成，要么都成功要么都回滚 |
| **最终一致** | 充值到账 | 幂等 + 重试 | 回调可能延迟，但最终一定到账 |
| **最终一致** | 提现审核结果 | 幂等 + 人工兜底 | 审核结果通过 RPC 通知，失败可重试 |
| **弱一致** | Redis 缓存 | TTL + 主动刷新 | 缓存短暂不一致可接受，不影响资金安全 |

### 8.2 单服务内的事务保证

```
核心原则：余额变更和流水记录必须在同一个数据库事务中

// 代码层面的保证（使用 GORM Gen 的事务 API）
err := query.Q.Transaction(func(tx *query.Query) error {
    // Step 1: 锁定账户行
    account, err := tx.WalletAccount.WithContext(ctx).
        Where(tx.WalletAccount.UserID.Eq(userId)).
        Where(tx.WalletAccount.CurrencyCode.Eq(currency)).
        Where(tx.WalletAccount.WalletType.Eq(walletType)).
        ForUpdate().   // SELECT ... FOR UPDATE
        First()
    if err != nil { return err }

    // Step 2: 业务校验
    if account.AvailableAmt.LessThan(amount) {
        return ErrInsufficientBalance
    }

    // Step 3: 更新余额
    _, err = tx.WalletAccount.WithContext(ctx).
        Where(tx.WalletAccount.ID.Eq(account.ID)).
        UpdateSimple(
            tx.WalletAccount.AvailableAmt.Sub(amount),
        )
    if err != nil { return err }

    // Step 4: 写入流水（同一事务，原子提交）
    err = tx.WalletTransaction.WithContext(ctx).Create(&model.WalletTransaction{
        // ... before_amt, amount, after_amt
    })
    return err
})
// 事务失败自动回滚，余额和流水要么都变要么都不变
```

### 8.3 跨服务的最终一致性

```
场景：充值成功后，财务模块调钱包 CreditWallet 入账

          ser-finance                          ser-wallet
              │                                    │
  收到三方回调  │                                    │
  支付成功     │                                    │
              │── CreditWallet RPC ───────────────►│
              │                                    │ 幂等检查
              │                                    │ DB 事务(加余额+写流水)
              │                                    │ 返回成功
              │◄── 成功 ──────────────────────────│
              │                                    │
  更新订单状态  │                                    │

如果 RPC 调用失败（网络超时）：
  ser-finance 不知道钱包是否已入账
  → 重试 CreditWallet（带相同 orderNo）
  → 钱包幂等检查发现已处理 → 返回成功
  → 不会重复入账

如果 ser-wallet 处理了但响应丢失：
  → 同上，重试时幂等返回成功

核心保障：幂等 + 重试 = 最终一致
```

### 8.4 定时对账（最后一道防线）

```
定时任务：每日凌晨执行

对账逻辑：
  对每个 wallet_account 记录：

  computed_balance = COALESCE(
    (SELECT after_amt FROM wallet_transaction
     WHERE user_id=? AND currency_code=? AND wallet_type=?
     ORDER BY create_at DESC, id DESC LIMIT 1),
    0
  )

  IF computed_balance != account.available_amt THEN
    → 记录差异日志
    → 发送告警通知
    → 人工排查（不自动修正）

  流水连续性校验：
    第 N 条流水的 after_amt == 第 N+1 条的 before_amt
    → 不连续则说明有流水丢失或数据被篡改

业界依据：
  Stripe 的 DQ Platform 持续监控清算账户余额
  Mercari 的对账系统定时比对内部记录与三方记录
  这是金融系统的标准"安全网"
```

---

## 九、缓存架构

### 9.1 缓存策略总览

| 缓存对象 | Redis Key 模式 | 策略 | TTL | 一致性要求 |
|---------|---------------|------|-----|----------|
| 币种配置列表 | `{wal:cur}:list` | 写时主动刷新 + TTL 兜底 | 5min | 弱（短暂不一致可接受） |
| 单个币种配置 | `{wal:cur}:{code}` | 写时主动刷新 + TTL 兜底 | 5min | 弱 |
| 汇率数据 | `{wal:rate}:{code}` | 定时任务更新后刷新 | 与更新间隔一致 | 弱 |
| 幂等键 | `{wal:idem}:{orderNo}:{op}:{wt}` | SetNX 一次写入 | 24h | 强（不可丢失） |
| 提现每日限额 | `{wal:wlimit}:{uid}:{cur}:{date}` | INCRBY 原子累加 | 当日过期 | 强（不可超限） |
| 基准币种 | `{wal:base}` | 设置后写入，只读 | 无过期 | 强（设一次不变） |

**不缓存余额**：
- 余额是高频写的数据，缓存和 DB 的一致性很难保证
- 余额查询走 DB 直读（单行主键查询，TiDB 性能足够）
- 避免缓存不一致导致的"用户看到的余额和实际不符"问题
- 如果后期确实出现性能瓶颈，再引入 — 但必须同时引入缓存和 DB 的双写事务保证

### 9.2 缓存键命名规范

```
命名规则：{wal:模块}:维度1:维度2:...

  {wal:cur}:list           — 币种配置列表
  {wal:cur}:VND            — VND 币种配置
  {wal:rate}:VND           — VND 汇率数据
  {wal:idem}:C1234:credit:1 — 幂等键
  {wal:wlimit}:10001:VND:20260226 — 提现日限额

{} 是 Redis Cluster hash tag，保证同模块的 key 落在同一个 slot
wal 前缀避免与其他服务的 key 冲突
```

---

## 十、异步事件架构

### 10.1 NATS JetStream 使用场景

基于现有工程已有的 NATS 基础设施，钱包服务需要使用消息队列的场景：

```
┌─────────────────────────────────────────────────────────────────┐
│                   NATS 消息架构                                   │
│                                                                 │
│  场景一：充值订单超时关闭                                          │
│  ├── 创建充值订单时 → JSPublishDelay(超时消息, 30min)              │
│  ├── 30 分钟后消费 → 检查订单状态是否仍为"待支付"                   │
│  └── 是 → 更新为"超时关闭"                                       │
│  好处：比定时扫表精确，不需要扫描全表                                │
│                                                                 │
│  场景二：余额变动通知（可选）                                       │
│  ├── 充值到账 / 提现成功 / 兑换完成后                              │
│  ├── Publish 余额变动事件                                        │
│  └── 其他服务（如推送服务）消费并通知用户                            │
│  好处：钱包服务不需要关心通知逻辑，解耦                              │
│                                                                 │
│  场景三：对账异常告警（可选）                                       │
│  ├── 对账任务发现不一致                                            │
│  ├── Publish 告警事件                                             │
│  └── 告警服务消费并发送通知                                        │
│                                                                 │
│  注意：余额操作本身不走 MQ                                         │
│  原因：余额和流水必须在同一 DB 事务中，MQ 无法保证这一点              │
│  MQ 只用于"操作完成后"的通知和调度，不参与核心资金流                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 消息 Subject 设计

```go
// 遵循现有工程 subjects 命名规范：<domain>.<resource>.<action>
const (
    WalletDepositTimeout  = "wallet.deposit.timeout"    // 延迟消息：充值超时
    WalletBalanceChanged  = "wallet.balance.changed"    // 余额变动事件
    WalletReconcileAlert  = "wallet.reconcile.alert"    // 对账告警
)

// 消费组命名：<service>-<domain>-<purpose>
const (
    GroupDepositTimeout  = "ser-wallet-deposit-timeout"
    GroupBalanceChanged  = "ser-wallet-balance-notify"
    GroupReconcileAlert  = "ser-wallet-reconcile-alert"
)
```

---

## 十一、安全架构

### 11.1 资金安全多层防护

```
┌─────────────────────────────────────────────────────────────────┐
│                     资金安全防护体系                               │
│                                                                 │
│  第一层：接入安全                                                 │
│  ├── gate-font: Token 鉴权 + IP 黑名单                          │
│  ├── gate-back: Token 鉴权 + IP 白名单 + RBAC 权限               │
│  └── RPC 层: Kitex 服务间鉴权（ETCD 服务发现，非公开端口）         │
│                                                                 │
│  第二层：业务安全                                                 │
│  ├── 幂等控制：Redis + DB 双保险防重复操作                         │
│  ├── 余额校验：每次扣款前校验 available >= amount（DB 行锁内）     │
│  ├── 冻结机制：提现不直接扣款，三步确认                            │
│  ├── KYC 校验：法币提现必须完成实名认证                            │
│  ├── 姓名一致性：提现账户持有人必须与 KYC 姓名一致                  │
│  ├── 提现路由：充值路径约束提现路径（法币充→法币提）                 │
│  ├── 重复转账拦截：同金额同币种 3 笔上限强拦截                     │
│  └── 每日限额：Redis 原子计数器 + 校验                            │
│                                                                 │
│  第三层：数据安全                                                 │
│  ├── 流水不可变：wallet_transaction 只 INSERT，代码层强制          │
│  ├── 软删除：所有删除操作通过 delete_at 标记，不物理删除            │
│  ├── 操作审计：所有 B 端操作通过 BLog 记录操作日志                  │
│  ├── 汇率日志：每次平台汇率更新生成不可篡改的日志记录               │
│  └── 上下文传播：tracer 贯穿全链路，每个操作可追溯到人              │
│                                                                 │
│  第四层：对账安全（最后防线）                                      │
│  ├── 定时对账：余额 vs 流水合计                                   │
│  ├── 流水连续性：前一条 after_amt == 后一条 before_amt             │
│  └── 差异告警 + 人工介入（不自动修正）                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 防超卖/超扣的架构保证

```
为什么不会出现余额扣成负数？

1. SELECT FOR UPDATE 锁定行 → 同一时刻只有一个事务能读写该账户
2. 读取 available_amt → 在锁内读，读到的是真实值
3. 校验 available_amt >= amount → 不足直接返回错误
4. UPDATE available_amt = available_amt - amount → 条件成立才执行

即使有 100 个并发请求同时扣同一个账户：
  → 第 1 个获取行锁，执行扣款
  → 第 2-100 个在 FOR UPDATE 处等待
  → 第 1 个 COMMIT 释放锁
  → 第 2 个获取锁，读到的已是扣款后的余额
  → 如果余额不足，校验失败，返回错误
  → 以此类推

这是数据库行锁的天然保障，不需要额外的分布式锁。
```

---

## 十二、高性能设计

### 12.1 性能优化点

| 优化点 | 方案 | 依据 |
|--------|------|------|
| **余额查询** | 物化余额字段直读，O(1) 时间复杂度 | 不需要 SUM 全部流水 |
| **币种/汇率读取** | Redis 缓存，命中率 99%+ | 写入少读取多，缓存效果显著 |
| **幂等检查** | Redis SetNX O(1)，拦截 99% 重复请求 | 避免走 DB |
| **批量查询** | 币种列表一次查全部缓存 | 避免 N+1 查询 |
| **连接池** | TiDB MaxOpen=20, MaxIdle=10 | 现有工程已配置 |
| **预编译语句** | GORM PrepareStmt=true | 现有工程已配置 |
| **跳过默认事务** | GORM SkipDefaultTransaction=true | 现有工程已配置，单条写操作不包隐式事务 |
| **RPC 复用连接** | Kitex Mux 模式 | 现有工程已配置 |
| **延迟初始化** | MySQL/RPC Client 首次使用时创建 | sync.Once 模式 |

### 12.2 不做的"优化"（避免过度设计）

| 不做的事 | 为什么不做 |
|---------|----------|
| 余额缓存到 Redis | 缓存一致性太难保证，出问题代价极大 |
| 流水表分库分表 | 初期数据量不需要，TiDB 本身支持水平扩展 |
| 读写分离（CQRS） | 余额操作是强一致的读写混合，分离意义不大 |
| 异步写流水（MQ） | 流水必须和余额在同一事务中，异步会破坏一致性 |
| 多级缓存 | 本地缓存+Redis 多级缓存增加复杂度，Redis 单层够用 |

---

## 十三、服务初始化架构

### 13.1 启动序列（遵循现有工程规范）

```go
func main() {
    // ① 日志初始化（最先）
    lg.InitKLog()

    // ② ETCD 配置加载 + Watch（第二步，所有配置依赖它）
    etcd.LoadAndWatch("/slg/")

    // ③ Redis 初始化（依赖 ETCD 配置）
    cfg.InitRedis()

    // ④ MySQL 初始化（依赖 ETCD 配置，可懒加载也可显式初始化）
    cfg.InitMySQL()

    // ⑤ NATS 初始化（如果需要消息队列）
    nats.Init(nats.InitOption{Addr: etcd.Get(ctx, env.NatsAddr)})

    // ⑥ 启动消费者（订阅消息）
    nats.StartConsumers(ctx, natsCfg, consumers...)

    // ⑦ 启动定时任务
    startCronJobs()  // 汇率更新、订单超时等

    // ⑧ 端口配置
    port := flag.String("port", etcd.DirectGet(ctx, namesp.EtcdWalletPort), "端口号")
    flag.Parse()

    // ⑨ 服务实例化（手动依赖注入）
    walletCoreSer := ser.NewWalletCoreSer(cache.NewWalletCache())
    currencySer   := ser.NewCurrencySer(cache.NewCurrencyCache())
    depositSer    := ser.NewDepositSer(walletCoreSer, currencySer)
    withdrawSer   := ser.NewWithdrawSer(walletCoreSer, currencySer)
    exchangeSer   := ser.NewExchangeSer(walletCoreSer, currencySer)
    recordSer     := ser.NewRecordSer()

    // ⑩ 创建 RPC 服务器并启动
    srv := walletservice.NewServer(
        &WalletServiceImpl{
            walletCoreSer: walletCoreSer,
            currencySer:   currencySer,
            depositSer:    depositSer,
            withdrawSer:   withdrawSer,
            exchangeSer:   exchangeSer,
            recordSer:     recordSer,
        },
        rpc.InitRpcServerParams(namesp.EtcdWalletService, utils.S2Int(ctx, *port))...,
    )
    srv.Run()
}
```

### 13.2 服务层依赖关系

```
                    handler.go (WalletServiceImpl)
                         │
           ┌─────────────┼──────────────────────┐
           │             │                      │
     walletCoreSer  currencySer            recordSer
     (余额操作核心)  (币种配置)             (记录查询)
           │             │
     ┌─────┼─────┐       │
     │     │     │       │
  depositSer  withdrawSer  exchangeSer
  (充值编排)   (提现编排)    (兑换编排)

依赖方向：
  depositSer  → walletCoreSer + currencySer
  withdrawSer → walletCoreSer + currencySer
  exchangeSer → walletCoreSer + currencySer

walletCoreSer 和 currencySer 是基础服务，被上层业务编排层依赖
handler.go 持有所有 service 的引用，按方法分发
```

---

## 十四、监控与可观测性

### 14.1 日志架构

```
结构化日志（遵循现有 Zap + lg 封装）：

关键日志点：
  · 每笔余额操作：userId, currency, walletType, amount, orderNo, before, after
  · 汇率更新：currency, oldRate, newRate, deviation, threshold
  · RPC 调用：target, method, duration, success/fail
  · 幂等命中：orderNo, opType（表明是重复请求）
  · 校验失败：reason（余额不足/限额超过/KYC未认证等）

上下文传播（已有基础设施）：
  · TraceId: 请求全链路唯一标识
  · UserId: 操作人（GORM Plugin 自动填充审计字段）
  · ClientIP: 来源 IP
```

### 14.2 操作审计日志

```
所有 B 端写操作 + 核心资金操作必须写审计日志：

// 遵循现有工程 BLog 审计日志模式
rpc.BLogClient().AddActLog(ctx, &ser_blog.ActAddReq{
    Backend:   enum.LogBackend,
    Model:     "wallet",
    Page:      "currency_config",
    Action:    "编辑币种 VND: 汇率浮动 1.4% → 1.5%",
    IsSuccess: int32(enum.HandleSuccess),
})

审计日志覆盖范围：
  · 币种配置增删改
  · 基准币种设置
  · 人工加减款（通过财务 → 钱包 RPC，钱包侧也要记）
  · 补单操作
  · 汇率更新（自动+手动）
```

---

## 十五、架构决策记录（ADR）

### ADR-01：选择物化余额而非纯流水推导

- **决策**：wallet_account 存物化余额（available_amt + frozen_amt），wallet_transaction 存不可变流水
- **理由**：余额查询是高频操作（每次打开钱包），SUM 全部流水的 O(N) 不可接受；物化余额 O(1) 直读
- **代价**：需要保证余额和流水的一致性（通过事务 + 对账）
- **业界参考**：Modern Treasury 推荐的做法 — "物化余额是缓存，流水是真相，两者必须可推导一致"

### ADR-02：选择悲观锁（FOR UPDATE）而非乐观锁

- **决策**：余额操作使用 `SELECT ... FOR UPDATE` 行级悲观锁
- **理由**：余额操作是写密集场景，乐观锁在高冲突时重试成本高；悲观锁一次获取锁即可完成，TiDB 原生支持
- **代价**：持锁期间其他操作同一账户的事务需等待
- **缓解**：账户粒度是 `(user, currency, wallet_type)`，锁粒度足够细，不会造成大面积阻塞

### ADR-03：不在初期引入余额 Redis 缓存

- **决策**：余额查询直接走 DB，不缓存到 Redis
- **理由**：缓存和 DB 的双写一致性在资金场景下极难保证；余额查询是单行主键查询，TiDB 性能足够
- **代价**：余额查询每次走 DB
- **后续**：如果 DB 成为瓶颈，再引入 Redis 缓存 + 事务双写 + 缓存失效策略

### ADR-04：流水和余额在同一事务中（不走 MQ 异步）

- **决策**：wallet_transaction 写入和 wallet_account 更新在同一 DB 事务中
- **理由**：流水是资金操作的"证据"，必须和余额变更原子绑定；MQ 异步写流水会导致"余额变了但流水丢了"的窗口
- **业界参考**：Stripe "余额计算必须与不可变流水保持一致" — 实现方式就是事务绑定

### ADR-05：选择 DECIMAL 存储金额而非 BIGINT

- **决策**：金额字段用 `DECIMAL(20,4)`
- **理由**：不同币种精度不同，DECIMAL 自然支持；BIGINT 需要额外的精度元数据和转换逻辑；DECIMAL 是金融系统的行业标准做法
- **代价**：DECIMAL 运算比 BIGINT 略慢（但在本场景中可以忽略不计）
- **注意**：精度的具体值（4 还是 8）需要与财务模块对齐确认

### ADR-06：不在初期引入完整复式记账

- **决策**：采用"物化余额 + 不可变流水"模型，不引入复式记账的借贷双方分录
- **理由**：完整复式需要维护大量系统账户（平台收入、手续费、清算等），当前需求是用户资产管理，不是专业会计系统；物化余额 + 流水已能覆盖审计和对账需求
- **演进路径**：如果后续需要完整的财务报表（资产负债表），可以在 wallet_transaction 之上增加一层 ledger_entry（借方/贷方分录），不影响现有架构

---

## 十六、架构全景总结

### 16.1 一张图总结

```
┌─────────────────── 安全边界 ───────────────────┐
│                                                │
│   gate-font / gate-back                        │
│   (Token + IP黑白名单 + RBAC)                   │
│                                                │
├────────────────── RPC 边界 ────────────────────┤
│                                                │
│   ser-wallet                                   │
│                                                │
│   ┌─────────── 业务编排层 ──────────┐           │
│   │ deposit_ser  withdraw_ser       │           │
│   │ exchange_ser  record_ser        │           │
│   └──────────┬─────────────────────┘           │
│              │                                  │
│   ┌──────────▼──────────────────────┐          │
│   │       核心引擎层                  │          │
│   │                                  │          │
│   │  wallet_core_ser                 │          │
│   │  ├── CreditBalance              │          │
│   │  ├── DebitBalance               │          │
│   │  ├── FreezeBalance              │          │
│   │  ├── UnfreezeBalance            │          │
│   │  ├── DeductFrozenBalance        │          │
│   │  └── DebitByPriority            │          │
│   │                                  │          │
│   │  currency_ser                    │          │
│   │  ├── 币种 CRUD                   │          │
│   │  ├── 汇率管理                    │          │
│   │  └── 基准币种设置                 │          │
│   └──────────┬──────────────────────┘          │
│              │                                  │
│   ┌──────────▼──────────────────────┐          │
│   │       数据层                      │          │
│   │                                  │          │
│   │  rep/ (GORM Gen)                 │          │
│   │  ├── wallet_account  (余额)      │          │
│   │  ├── wallet_transaction (流水)    │          │
│   │  ├── currency_config (币种)      │          │
│   │  └── 各订单表                     │          │
│   │                                  │          │
│   │  cache/ (Redis)                  │          │
│   │  ├── 币种/汇率缓存               │          │
│   │  ├── 幂等键                      │          │
│   │  └── 限额计数器                  │          │
│   └─────────────────────────────────┘          │
│                                                │
├────────────── 数据库边界 ──────────────────────┤
│                                                │
│   TiDB                Redis              ETCD  │
│   (余额+流水+订单)     (缓存+锁+计数)     (配置) │
│                                                │
└────────────────────────────────────────────────┘
```

### 16.2 架构核心要点速查

| 维度 | 决策 | 业界依据 |
|------|------|---------|
| 账户模型 | 币种维度独立账户，物化余额 + 不可变流水 | Modern Treasury / Rapyd |
| 余额字段 | available_amt + frozen_amt，无 total 冗余 | Rapyd 四种余额类型简化版 |
| 金额类型 | DECIMAL(20,4)，禁止 FLOAT | Stripe / 金融系统标准 |
| 并发控制 | SELECT FOR UPDATE 行级悲观锁 | 写密集场景标准做法 |
| 幂等保障 | Redis SetNX（快速层）+ DB 唯一索引（兜底层） | Stripe / Airbnb 双层策略 |
| 冻结模式 | TCC 思想：Freeze→Confirm/Cancel | 蚂蚁金服 TCC 模式 |
| 汇率锁定 | 订单快照汇率，操作期间汇率不变 | Stripe FX Quote |
| 流水设计 | 只 INSERT 不 UPDATE/DELETE，带 before/after | Stripe Immutable Ledger |
| 跨服务一致 | 幂等 + 重试 = 最终一致，不用分布式事务 | 业界微服务标准做法 |
| 缓存策略 | 币种汇率缓存，余额不缓存 | 安全优先，性能可接受 |
| 对账机制 | 定时校验 余额 == SUM(流水)，差异告警不自动修正 | Stripe DQ Platform |
| 消息队列 | NATS 用于异步通知和延迟任务，不参与余额操作 | 资金操作不走 MQ |

### 16.3 这个架构能支撑什么规模

| 指标 | 预估能力 | 瓶颈点 | 扩展方案 |
|------|---------|--------|---------|
| 并发余额操作 | 数千 TPS/单账户 | DB 行锁等待 | 不同用户天然并行，同用户串行但单次 <10ms |
| 账变流水存储 | 亿级 | 单表行数 | TiDB 自动分片 / 后期按时间分区 |
| 汇率查询 | 万级 QPS | Redis 吞吐 | Redis Cluster 横向扩展 |
| 币种配置 | 百级配置数 | 无瓶颈 | 全量缓存 |
| 充值/提现订单 | 千万级 | 单表行数 | TiDB 自动分片 |

**不过度设计**：当前架构基于单 TiDB 实例 + Redis + ETCD 即可支撑，不预设分库分表、不预设 CQRS、不预设多级缓存。遇到瓶颈时再针对性扩展，TiDB 的水平扩展能力为后续增长提供了空间。
