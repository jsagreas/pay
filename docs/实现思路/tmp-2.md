# ser-wallet 整体实现思路

> 编写时间：2026-02-26
> 定位：粗粒度实现方向，抓确定性骨架，留弹性空间
> 依据来源：
> - 需求分析：/Users/mac/gitlab/z-readme/result/tmp-2.md（152张图片分析）
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-2.md（第三版 46个接口）
> - 依赖关系：/Users/mac/gitlab/z-readme/依赖关系/tmp-2.md（调用方向+耦合分析）
> - 链路关系：/Users/mac/gitlab/z-readme/链路关系/tmp-2.md
> - 工程认知：/Users/mac/gitlab/z-readme/工程认知/tmp-2.md（技术规范+开发步骤）
> - 流程思路：/Users/mac/gitlab/z-readme/流程思路/tmp-2.md（完整处理流程）
> 阶段：预评审，核心骨架确定，细节保留弹性

---

## 一、实现总纲

### 1.1 一句话概括

**ser-wallet 是一个多币种、多子钱包的资金管理服务，对内处理余额增减冻解和账变记录，对外通过 RPC 为财务/游戏/直播等模块提供统一的资金操作能力，同时包含币种配置和汇率管理两个基础设施子模块。**

### 1.2 实现的三条主线

```
主线一：基础设施（币种配置 + 汇率管理）
  → 为所有资金操作提供币种数据、汇率数据、精度规则
  → 独立于钱包核心，可最先实现，无外部依赖

主线二：钱包核心（余额操作 + 账变流水 + RPC服务）
  → 5个核心方法 (Credit/Debit/Freeze/Unfreeze/DeductFrozen)
  → 这是整个模块的"心脏"，财务模块联调的基础

主线三：业务流程（充值/提现/兑换 + C端接口 + B端接口）
  → 建立在前两条主线之上，串联用户端和管理端
  → 部分流程需要财务模块配合联调
```

### 1.3 核心设计原则（贯穿实现全程）

| 原则 | 说明 | 依据 |
|------|------|------|
| 币种隔离 | 所有数据和操作按币种维度隔离，不跨币种混合 | 需求3.3 |
| 余额一致 | 账变流水与余额必须强一致，每笔操作必须记流水 | 需求3.10 |
| 幂等处理 | 所有资金变动操作基于 orderNo 幂等，防止重复执行 | 需求3.10 |
| 消费优先 | 扣款时先中心钱包再奖励钱包，返奖按投注比例拆分 | 需求4.1.2 |
| 冻结模式 | 提现不直接扣款，走"冻结→审核→扣除/解冻"三步 | 需求5.5 |
| 事务保证 | 余额操作必须在数据库事务内完成，先加锁再读写 | 资金安全要求 |

---

## 二、绕不过的工程前置动作

在写任何业务代码之前，以下动作是必须完成的"地基工程"，跳过任何一项都无法正常开发。

### 2.1 IDL 定义（第一步，一切的起点）

**依据**：工程认知第二章 — IDL先行开发模式，gen_kitex.sh 脚本扫描 common/idl/ 目录

**要做的事**：

```
common/idl/ser-wallet/
├── service.thrift      ← 服务入口，定义 WalletService 的所有方法
├── wallet.thrift       ← 钱包相关结构（余额、冻结、加减款请求/响应）
├── currency.thrift     ← 币种配置相关结构（币种信息、汇率信息）
├── deposit.thrift      ← 充值相关结构（可后续按需拆分）
├── withdraw.thrift     ← 提现相关结构（可后续按需拆分）
├── exchange.thrift     ← 兑换相关结构
└── record.thrift       ← 账变记录相关结构
```

**核心考虑**：
- namespace 命名为 `ser_wallet`，遵循现有工程 `ser_xxx` 规范
- service 命名为 `WalletService`
- RPC 方法覆盖三类：C端业务方法 + B端管理方法 + 供他方调用的核心方法
- 供他方调用的 6 个核心 RPC（CreditWallet/DebitWallet/FreezeBalance/UnfreezeBalance/DeductFrozenAmount/QueryWalletBalance）是**与财务模块联调的合同**，IDL 定义必须双方对齐
- 请求/响应结构遵循 `方法名Req` / `方法名Resp` 命名

**IDL 定义后执行 gen_kitex.sh**：
- 自动生成 `common/pkg/kitex_gen/ser_wallet/` 代码
- 自动生成 ser-wallet 的 handler.go 和 main.go 骨架

### 2.2 工程注册（让 ser-wallet 融入整个体系）

**依据**：工程认知第十一章 — ser-wallet 已知缺失清单

必须修改的 4 个文件：

| 文件 | 要做的事 | 为什么绕不过 |
|------|---------|------------|
| `go.work` | 添加 `./ser-wallet` | 不加入 workspace，模块无法被其他模块引用 |
| `common/pkg/consts/namesp/namesp.go` | 添加 EtcdWalletService / EtcdWalletPort / EtcdWalletDB 等常量 | ETCD 服务注册和配置读取的标识，没有就启动不了 |
| `common/rpc/rpc_client.go` | 添加 WalletClient() 工厂方法 | 其他模块（财务/游戏/直播）调用我方 RPC 的入口 |
| `ser-wallet/internal/gen/gorm_gen.go` | 修正 namesp.EtcdAppDb → namesp.EtcdWalletDB | 当前引用了错误的数据库配置，会连错库 |

**这 4 项改动是开发的绝对前置，缺一不可。**

### 2.3 数据库表结构准备（GORM Gen 生成）

**依据**：工程认知第二章 — GORM Gen 代码生成流程

流程：先在数据库中建好表 → 在 gorm_gen.go 中配置表名 → 执行 `go run` 自动生成 model/query/repo 代码

**这意味着**：在动手写业务代码前，需要先完成表结构设计（至少核心表），然后通过 Gen 生成基础的 CRUD 代码，再在此基础上编写业务逻辑。

---

## 三、数据存储设计方向

### 3.1 核心表清单（绕不过的表）

以下是根据需求分析推导的必须建的表，不分先后都需要：

```
┌────────────────────────────────────────────────────────────┐
│                    ser-wallet 核心表                         │
│                                                            │
│  ① currency_config         — 币种配置表                     │
│     币种代码/名称/类型/国家/符号/精度/汇率/浮动/阈值/排序/状态  │
│                                                            │
│  ② base_currency_setting   — 基准币种设置表（或合入config）    │
│     基准币种代码/设置时间/操作人（一旦设置不可变更）             │
│                                                            │
│  ③ exchange_rate_log        — 汇率变更日志表                 │
│     日志编号/币种/更新时间/实时汇率/平台汇率/入款汇率/出款汇率   │
│                                                            │
│  ④ wallet_account           — 钱包账户表                     │
│     用户ID/币种/钱包类型/可用余额/冻结余额/总余额/版本号        │
│     唯一索引：(user_id, currency_code, wallet_type)          │
│                                                            │
│  ⑤ wallet_transaction       — 账变流水表                     │
│     流水ID/用户ID/币种/钱包类型/变动类型/关联订单号             │
│     变动前余额/变动金额/变动后余额/冻结变动/备注/创建时间        │
│     唯一索引：(order_no, op_type) — 幂等保障兜底              │
│                                                            │
│  ⑥ deposit_order            — 充值订单表                     │
│     订单号(C+16)/用户ID/币种/金额/充值方式/汇率快照             │
│     奖金活动ID/赠送金额/稽核倍数/状态/创建时间/完成时间         │
│                                                            │
│  ⑦ withdraw_order           — 提现订单表                     │
│     订单号(T+16)/用户ID/币种/金额/提现方式/汇率快照             │
│     手续费/冻结金额/状态/创建时间                              │
│                                                            │
│  ⑧ exchange_order           — 兑换订单表                     │
│     订单号/用户ID/源币种/目标币种(BSB)/兑换金额/汇率快照         │
│     兑换所得/赠送金额/稽核倍数/创建时间                        │
│                                                            │
│  ⑨ withdraw_account         — 提现账户表                     │
│     用户ID/币种/提现方式类型/账户信息(银行名/账号/持有人等)      │
│     唯一索引：(user_id, currency_code, method_type)           │
│                                                            │
│  ⑩ audit_requirement        — 稽核流水要求表                  │
│     用户ID/币种/关联订单号/需完成流水金额/已完成流水金额/状态    │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 3.2 表设计的核心考量

**金额存储**：
- 需要在联调前与财务模块统一确认：用"最小单位整数"（分/越南盾整数）还是"元级 Decimal"
- 推荐方案：数据库用 `DECIMAL(20,8)` 或 `BIGINT`（最小单位），业务层统一处理精度
- 不同币种精度不同（BSB 2位小数、VND/IDR 整数、THB 2位小数、加密货币 8位），存储层必须能覆盖最高精度

**并发控制**：
- wallet_account 表需要加`版本号`字段做乐观锁，或在事务中用 `SELECT ... FOR UPDATE` 行锁
- 推荐 `SELECT ... FOR UPDATE`，因为余额操作频率高，乐观锁重试成本高

**幂等保障**：
- wallet_transaction 表的 `(order_no, op_type)` 唯一索引是最终兜底
- 业务层先用 Redis SetNX 做快速拦截，数据库唯一约束做最后防线

**分表预留**：
- wallet_transaction（账变流水）数据量会持续增长，后续可能需要按时间分表
- 当前可先单表，但表设计预留时间维度（如 create_time 作为分表键），不做过度设计

### 3.3 缓存设计方向

```
Redis 缓存策略（粗粒度方向）：

① 币种配置缓存
   key: "wallet:currency:{code}" 或 "wallet:currency:list"
   场景: 所有接口都要读币种/汇率信息，频率极高
   策略: 写入/更新配置时主动刷新，设合理 TTL

② 幂等键
   key: "wallet:idempotent:{orderNo}:{opType}"
   场景: 所有资金操作的快速幂等拦截
   策略: SetNX + 24h TTL

③ 用户余额缓存（可选，需谨慎）
   场景: 钱包首页查余额频率高
   策略: 如果用，必须保证数据库操作和缓存更新在同一事务中
   风险: 缓存和数据库不一致会导致资金显示错误
   建议: 初期直接查数据库，性能瓶颈出现后再引入缓存

④ 提现每日限额计数
   key: "wallet:withdraw:daily:{userId}:{currency}:{date}"
   场景: 提现时校验每日限额
   策略: INCRBY + 当日过期
```

---

## 四、服务内部分层架构

### 4.1 目录结构（遵循现有工程规范）

**依据**：工程认知第九章 — 标准服务模块结构

```
ser-wallet/
├── main.go                    ← 服务启动入口（gen_kitex.sh 生成骨架）
├── handler.go                 ← RPC方法入口（薄层，委托给 service）
├── Makefile                   ← 构建脚本
├── go.mod
├── internal/
│   ├── gen/
│   │   └── gorm_gen.go        ← GORM Gen 代码生成配置
│   ├── cfg/                   ← 配置管理（ETCD 配置加载 + Redis 初始化）
│   ├── ser/                   ← 业务逻辑层（核心实现）
│   │   ├── wallet_ser.go      ← 钱包余额操作核心
│   │   ├── currency_ser.go    ← 币种配置管理
│   │   ├── deposit_ser.go     ← 充值业务
│   │   ├── withdraw_ser.go    ← 提现业务
│   │   ├── exchange_ser.go    ← 兑换业务
│   │   └── record_ser.go      ← 账变记录查询
│   ├── rep/                   ← 数据访问层（GORM Gen 生成 + 自定义查询）
│   ├── cache/                 ← 缓存操作封装
│   ├── enum/                  ← 枚举定义（钱包类型、变动类型、订单状态等）
│   └── errs/                  ← 错误码定义
└── kitex_gen/                 ← Kitex 自动生成代码（不手动编辑）
```

### 4.2 分层职责（严格遵守，不越层调用）

```
handler.go（入口层）
    │  职责：接收 RPC 请求 → 委托给 ser 层 → 返回响应
    │  规则：不写业务逻辑，只做参数传递
    │
    ▼
ser/（业务逻辑层）
    │  职责：业务规则校验 → 调用 rep 层操作数据 → 调用 cache 层操作缓存
    │  规则：核心逻辑全部在此层，事务控制在此层
    │
    ▼
rep/（数据访问层）
    │  职责：数据库 CRUD 操作
    │  规则：不包含业务逻辑，只负责数据读写
    │  来源：大部分由 GORM Gen 生成，复杂查询手写
    │
    ▼
cache/（缓存层）
    │  职责：Redis 缓存读写
    │  规则：封装缓存 key 和操作，供 ser 层调用
```

### 4.3 错误码分配

**依据**：工程认知第二章 — 错误码规范 `6 + 服务编号(3位) + 模块(3位)`

ser-wallet 需要申请一个服务编号。参考现有分配：
- ser-app: 011
- ser-item: 不详

假设 ser-wallet 分配编号 022（需确认），则错误码范围：

```go
// 文件：ser-wallet/internal/errs/code.go

const (
    // 钱包核心错误 6022 + 000 段
    ErrBalanceInsufficient = 6022001  // 余额不足
    ErrFrozenInsufficient  = 6022002  // 冻结余额不足
    ErrDuplicateOperation  = 6022003  // 重复操作（幂等拦截）
    ErrWalletNotFound      = 6022004  // 钱包账户不存在

    // 币种配置错误 6022 + 100 段
    ErrCurrencyDisabled    = 6022101  // 币种已禁用
    ErrCurrencyNotFound    = 6022102  // 币种不存在
    ErrBaseCurrencySet     = 6022103  // 基准币种已设置过
    ErrBaseCurrencyNoChange= 6022104  // 基准币种不可更改

    // 充值错误 6022 + 200 段
    ErrDepositDuplicate    = 6022201  // 重复充值拦截
    ErrDepositLimitReached = 6022202  // 达到重复充值上限
    ErrDepositOrderNotFound= 6022203  // 充值订单不存在

    // 提现错误 6022 + 300 段
    ErrWithdrawDailyLimit  = 6022301  // 超出每日提现限额
    ErrWithdrawMinAmount   = 6022302  // 低于单笔最低金额
    ErrWithdrawKycRequired = 6022303  // 需要KYC认证
    ErrWithdrawNameMismatch= 6022304  // 账户持有人与KYC姓名不一致
    ErrWithdrawRouteInvalid= 6022305  // 提现路由校验失败

    // 兑换错误 6022 + 400 段
    ErrExchangeAmountExceed= 6022401  // 兑换金额超过余额
    ErrExchangeRateExpired = 6022402  // 汇率已过期

    // 每个功能域预留 100 个号段，防止后续不够用
)
```

---

## 五、核心实现模块思路

### 5.1 模块一：币种配置（最先实现，无外部依赖）

**地位**：地基中的地基。所有资金操作都依赖币种和汇率数据。

**实现方向**：

```
B端接口（gate-back 路由）：
  ├── 币种列表查询（分页 + 筛选）
  ├── 币种编辑（汇率浮动%、阈值%、图标、状态）
  ├── 币种启用/禁用（基准币种不可禁用）
  ├── 基准币种设置（一次性，设后不可改）
  └── 汇率日志查询（分页 + 筛选）

RPC 接口（供其他模块调用）：
  ├── GetCurrencyConfig — 获取可用币种列表
  ├── GetExchangeRate — 获取指定币种汇率
  └── ConvertAmount — 金额币种换算

定时任务：
  └── 汇率定时更新
      ├── 调 3 个三方 API 取平均值
      ├── 偏差 >= 阈值% 时更新平台汇率
      ├── 联动计算入款/出款汇率
      ├── 生成汇率变更日志
      └── 刷新 Redis 缓存
```

**关键设计点**：
- 基准币种设置的一次性约束：数据库层面保证（表中只能有一条记录，或者 flag 字段 + 唯一约束）
- BSB 特殊处理：BSB 的汇率/代码/类型/状态不可配置，代码中硬编码或数据库中标记为系统内置
- 汇率 API 容错：3 个 API 任一失败时取可用 API 的平均值，全部失败时保持旧值不更新 + 告警
- 汇率精度：法定货币 2 位小数，加密货币 8 位小数 — 需要在 currency_config 表中配置每个币种的精度

### 5.2 模块二：钱包核心（心脏，最关键）

**地位**：所有资金操作最终都汇聚到这里。财务模块联调的基础。

**5 个核心余额方法**：

```
CreditBalance(userId, currency, walletType, amount, orderNo, opType)
  → 加款：可用余额 += amount
  → 适用：充值到账、补单、人工加款、投注返奖

DebitBalance(userId, currency, walletType, amount, orderNo, opType)
  → 减款：校验可用余额 >= amount → 可用余额 -= amount
  → 适用：人工减款、指定钱包扣款

FreezeBalance(userId, currency, walletType, amount, orderNo)
  → 冻结：校验可用余额 >= amount → 可用余额 -= amount，冻结余额 += amount
  → 适用：提现订单创建

UnfreezeBalance(userId, currency, walletType, amount, orderNo)
  → 解冻：校验冻结余额 >= amount → 冻结余额 -= amount，可用余额 += amount
  → 适用：提现审核驳回

DeductFrozenBalance(userId, currency, walletType, amount, orderNo)
  → 扣冻结：校验冻结余额 >= amount → 冻结余额 -= amount，总余额 -= amount
  → 适用：出款成功（资金真正离开钱包）
```

**每个方法的内部执行流程（统一模式）**：

```
1. 幂等校验（Redis SetNX: wallet:idempotent:{orderNo}:{opType}）
   └── 已存在 → 直接返回成功
2. 开启数据库事务
3. 行锁定（SELECT ... FOR UPDATE WHERE user_id=? AND currency=? AND wallet_type=?）
4. 读取当前余额
5. 校验（余额充足 / 冻结充足）
6. 计算新余额
7. 更新余额
8. 写入账变流水（前余额、变动额、后余额、关联订单号）
9. 提交事务
10. 返回结果
```

**消费优先级方法（DebitByPriority）**：

```
DebitByPriority(userId, currency, amount, orderNo, opType)
  → 查询中心钱包余额 A + 奖励钱包余额 B
  → 校验 A + B >= amount
  → 中心扣款 = min(A, amount)
  → 奖励扣款 = amount - 中心扣款
  → 事务内分别扣两个钱包 + 分别写两条流水
  → 返回各钱包实际扣款金额（用于后续返奖按比例拆分）
```

**关键设计点**：
- 所有方法必须在事务中，先锁再读
- 每次操作必须写流水（审计需求，不可省略）
- 幂等双保险：Redis 快速拦截 + 数据库唯一约束兜底
- wallet_account 表的 `(user_id, currency_code, wallet_type)` 必须有唯一索引

### 5.3 模块三：充值流程

**地位**：核心业务，涉及跨模块协作。

**我方职责边界**：

```
我方负责的部分：
  ├── 充值页面数据组装（调财务获取充值方式 + 查币种汇率）
  ├── 重复转账检测（查本地订单表）
  ├── 创建平台充值订单（C+16位订单号）
  ├── 调财务模块匹配通道并创建通道订单
  ├── 充值回调处理（被财务模块调 CreditWallet RPC）
  └── 充值订单状态更新

财务模块负责的部分：
  ├── 配置充值方式和支付通道
  ├── 通道轮询匹配算法
  ├── 创建通道侧订单
  ├── 接收三方支付回调
  └── 回调后调我方 RPC 入账
```

**实现要点**：
- 充值订单表由我方建（平台订单），通道订单表由财务方建 — 两表分属两个服务
- 充值成功入账走 CreditWallet RPC，不是我方直接处理回调，而是财务模块收到三方回调后调我方
- 奖金活动选择：如果活动模块尚未开发，先预留接口结构，实现时先不对接
- USDT 充值支持游客（走 gate-font 的 Open 路由组，免鉴权）

### 5.4 模块四：提现流程

**地位**：最复杂的流程，跨 KYC + 多级审核 + 冻结/解冻/扣除。

**我方职责边界**：

```
我方负责的部分：
  ├── 提现方式获取（调财务获取 + KYC状态过滤 + 充值路由规则过滤）
  ├── 提现账户信息管理（首次保存、后续只读）
  ├── 创建提现订单（T+16位）+ 冻结余额
  ├── 提现校验（KYC/限额/姓名一致性/充值路由规则）
  └── 被动接受财务模块的 UnfreezeBalance / DeductFrozenAmount 调用

财务模块负责的部分：
  ├── 配置提现方式
  ├── 风控审核、财务审核
  ├── 发起通道代付
  ├── 审核驳回时调我方 UnfreezeBalance
  └── 出款成功时调我方 DeductFrozenAmount
```

**实现要点**：
- 冻结操作在创建提现订单时由我方内部完成，不需要财务模块来调
- 审核和出款结果由财务模块通过 RPC 通知我方执行余额变动
- 出款失败时**保持冻结不动**，等人工处理 — 这是产品明确要求
- 充值路由规则校验（"怎么充的怎么提"）：需要查用户历史充值记录判断路由
- 提现账户持有人必须与 KYC 姓名一致（忽略大小写和多余空格）— 校验逻辑在我方

### 5.5 模块五：兑换流程

**地位**：纯内部操作，最简单，不依赖外部模块。

**实现方向**：

```
法币(VND/IDR/THB) → BSB（单向）
  1. 查币种配置获取平台汇率
  2. 校验兑换金额 <= 可用余额
  3. 锁定汇率（取当前平台汇率作为本次兑换汇率）
  4. 事务内操作两个币种的余额：
     - 法币中心钱包 -= 兑换金额
     - BSB 中心钱包 += 兑换所得
     - BSB 奖励钱包 += 赠送金额（如有）
  5. 写入账变流水（法币出、BSB入各一条）
  6. 记录稽核信息（赠送部分）
```

**实现要点**：
- 虽然操作两个币种的余额，但属于同一用户，在同一数据库事务中完成
- BSB 页面无兑换入口 — 前端控制，后端也要做方向校验（源币种不能是 BSB）
- 赠送比例由运营配置（具体配在哪待确认，先预留参数位）

### 5.6 模块六：账变记录

**地位**：纯查询功能，不涉及资金变动。

**实现方向**：
- 4 个 Tab 对应 4 种 type 过滤：充值/提现/兑换/奖励
- 严格按当前币种过滤，切换币种时切换数据集
- 分页查询 + 时间倒序
- 详情页：关联订单完整信息 + 失败原因（如有）
- 数据来源：wallet_transaction 表 + 关联订单表 JOIN

---

## 六、RPC 接口设计方向

### 6.1 供财务模块调用的核心 RPC（联调合同）

这 6 个接口是**与财务模块的合同接口**，IDL 定义需要双方对齐：

```
1. CreditWallet(userId, currency, walletType, amount, orderNo, opType, auditMultiplier?, auditTurnover?)
   → 加款入账
   → 场景：充值成功、补单、人工加款、活动奖金

2. DebitWallet(userId, currency, walletType, amount, orderNo, opType)
   → 减款扣款
   → 场景：人工减款

3. FreezeBalance(userId, currency, walletType, amount, orderNo)
   → 冻结余额
   → 场景：提现订单创建（通常我方内部调用，但也暴露 RPC 供外部）

4. UnfreezeBalance(userId, currency, walletType, amount, orderNo)
   → 解冻余额
   → 场景：审核驳回

5. DeductFrozenAmount(userId, currency, walletType, amount, orderNo)
   → 扣除冻结金额
   → 场景：出款成功

6. QueryWalletBalance(userId, currency?)
   → 查询余额（各子钱包的可用+冻结）
   → 场景：加减款表单展示、B端用户查看
```

**设计要点**：
- 每个方法的 orderNo 是幂等键，重复调用返回成功不重复执行
- CreditWallet 需要支持可选的稽核参数（充值奖金、补单、加款场景需要）
- 响应中应返回操作后的余额（方便调用方确认）
- 所有方法必须返回统一的错误码体系

### 6.2 供游戏/直播等模块调用的 RPC

```
7. DebitByPriority(userId, currency, amount, orderNo, opType)
   → 按消费优先级自动扣款（先中心后奖励）
   → 返回各钱包实际扣款金额

8. GetCurrencyConfig()
   → 获取所有可用币种配置

9. GetExchangeRate(currencyCode)
   → 获取指定币种的汇率信息

10. ConvertAmount(amount, fromCurrency, toCurrency)
    → 金额币种换算
```

### 6.3 我方调用他方的 RPC

```
调财务模块：
  ├── GetPaymentMethods(currency, type) — 获取充值/提现方式
  ├── MatchAndCreateChannelOrder(orderInfo) — 通道匹配+创建通道订单
  └── InitiateChannelPayout(orderInfo) — 发起代付

调 KYC 模块：
  ├── CheckKycStatus(userId) — 提现前KYC状态检查
  └── GetKycInfo(userId) — 获取KYC姓名

调用户模块：
  └── GetUserInfo(userId) — 获取用户基础信息
```

---

## 七、网关路由设计方向

### 7.1 C端路由（gate-font）

**依据**：工程认知第三/四章 — gate-font 路由分组规范

```go
// 路由分组：/open/wallet/xxx（免鉴权）+ /api/wallet/xxx（需登录）

// Open 组（免鉴权）
walletOpen := openGroup.Group("/wallet")
{
    walletOpen.POST("/usdt-deposit-info", ...)   // C-4: USDT充值信息（游客可用）
}

// Api 组（需登录鉴权）
walletApi := apiGroup.Group("/wallet")
{
    walletApi.POST("/overview", ...)              // C-1: 钱包余额总览
    walletApi.POST("/deposit/methods", ...)       // C-2: 充值方式列表
    walletApi.POST("/deposit/create", ...)        // C-3: 创建充值订单
    walletApi.POST("/exchange/preview", ...)      // C-5: 兑换预览
    walletApi.POST("/exchange/create", ...)       // C-6: 执行兑换
    walletApi.POST("/withdraw/methods", ...)      // C-7: 提现方式列表
    walletApi.POST("/withdraw/create", ...)       // C-8: 创建提现订单
    walletApi.POST("/withdraw/account", ...)      // C-9: 获取提现账户
    walletApi.POST("/withdraw/account/save", ...) // C-10: 保存提现账户
    walletApi.POST("/transaction/list", ...)      // C-11: 账变记录列表
    walletApi.POST("/transaction/detail", ...)    // C-12: 账变记录详情
    walletApi.POST("/rate", ...)                  // C-13: 汇率查询
    // ... 其他 Should/Could 接口
}
```

### 7.2 B端路由（gate-back）

```go
// 路由分组：/admin/api/wallet/xxx（需后台登录 + RBAC 权限）

currencyApi := adminApiGroup.Group("/currency")
{
    currencyApi.POST("/list", ...)               // B-1: 币种列表
    currencyApi.POST("/edit", ...)               // B-2: 编辑币种
    currencyApi.POST("/toggle-status", ...)      // B-3: 启用/禁用
    currencyApi.POST("/set-base", ...)           // B-4: 设置基准币种
    currencyApi.POST("/rate-logs", ...)          // B-5: 汇率日志
    currencyApi.POST("/detail", ...)             // B-6: 币种详情
}
```

**要点**：
- C端全部 POST 请求（遵循现有工程规范，gate-font 只用 POST）
- B端同样 POST（遵循 gate-back 规范）
- 路由路径遵循 `/模块/功能` 的二级结构
- gate-font 路由需要在 BlackCheck（IP黑名单）之后
- gate-back 路由需要在 WhiteCheck + BackAuthMiddleware + RBAC（CheckUrlAuth）之后

---

## 八、与财务模块联调策略

### 8.1 联调的本质：双方按 IDL 合同开发

```
              联调前

我方                           财务方
┌────────────────┐            ┌────────────────┐
│ 定义 IDL        │            │ 定义 IDL        │
│ (ser-wallet)   │            │ (ser-finance)  │
│                │            │                │
│ Mock 财务RPC   │            │ Mock 钱包RPC   │
│ 内部逻辑先跑通  │            │ 内部逻辑先跑通  │
└────────────────┘            └────────────────┘

              联调时

┌─────────────────────────────────────────────┐
│  双方 RPC 对接，Mock 替换为真实调用            │
│                                             │
│  我方提供：CreditWallet / DebitWallet / ...  │
│  财务提供：GetPaymentMethods / ...           │
│                                             │
│  按链路跑通：充值全链路 → 提现全链路 → 人工修正 │
└─────────────────────────────────────────────┘
```

### 8.2 可以先独立做的部分（不等财务）

| 内容 | 为什么可以独立 |
|------|-------------|
| 币种配置全部功能 | 纯内部，无外部依赖 |
| 钱包账户 + 5个核心余额方法 | 纯内部数据操作 |
| 6个核心 RPC 实现 | 我方暴露，不需要调外部 |
| 兑换全流程 | 纯内部操作，不涉及财务 |
| 账变记录查询 | 纯查询，数据来自本地表 |
| 提现账户管理 | 纯 CRUD |
| 汇率定时任务 | 只调三方汇率 API |
| B端币种管理全部接口 | 纯内部 CRUD |

### 8.3 需要等财务联调的部分

| 内容 | 依赖财务什么 |
|------|------------|
| C端充值方式列表 | 财务的支付方式配置数据 |
| C端创建充值订单 | 财务的通道匹配能力 |
| 充值到账 | 财务收到回调后调我方 CreditWallet |
| C端提现方式列表 | 财务的提现方式配置数据 |
| 提现解冻/扣冻结 | 财务审核后调我方 Unfreeze/DeductFrozen |
| 人工加减款 | 财务审核后调我方 Credit/Debit |

### 8.4 Mock 策略

联调前，对财务模块的依赖可以通过 Mock 解决：

```
方式一：在 ser 层写 Mock 实现
  ├── 获取充值方式 → 返回硬编码的测试数据
  ├── 通道匹配 → 返回固定通道信息
  └── 联调时替换为真实 RPC 调用

方式二：在 rpc_client 层做条件分支
  └── 配置项控制：mock_finance=true 时走 Mock，false 走真实 RPC
```

---

## 九、定时任务实现方向

### 9.1 汇率定时更新（核心，必须有）

```
触发频率：可配置（ETCD 配置项）
执行逻辑：
  1. 并发调 3 个三方汇率 API
  2. 过滤失败的，取成功的平均值作为实时汇率
  3. 遍历每个需要汇率的币种：
     a. 更新实时汇率字段
     b. 计算偏差 = |实时 - 平台| / 实时 × 100%
     c. 偏差 >= 阈值% → 更新平台汇率 + 入款汇率 + 出款汇率 + 写日志
     d. 偏差 < 阈值% → 仅更新实时汇率，平台汇率不变
  4. 刷新 Redis 缓存
  5. 全部 API 失败 → 保持旧值 + 告警日志

注意：
  - 初次运行（平台汇率为空）→ 直接以实时汇率设置平台汇率
  - BSB 汇率不从三方 API 获取（锚定 USDT，10 BSB = 1 USDT）
```

### 9.2 充值订单超时处理

```
触发频率：每分钟或每5分钟
执行逻辑：
  扫描 deposit_order 表中 状态=待支付 且 创建时间超过有效期 的订单
  → 批量更新状态为"支付超时"
  → 不操作余额（待支付订单没有入过账）
```

### 9.3 余额对账（可后期实现）

```
触发频率：每日凌晨
执行逻辑：
  对每个 (user_id, currency, wallet_type)：
  → 计算：初始余额 + SUM(所有入账流水) - SUM(所有出账流水)
  → 对比：wallet_account 表中的当前余额
  → 不一致 → 记录差异日志 + 告警通知
  → 纯检测，不自动修正
```

---

## 十、关键技术决策清单（需确认）

以下事项影响实现方向，需要在开发前确认或在评审中讨论：

### 10.1 必须与财务模块对齐的

| 序号 | 事项 | 影响 | 当前推测 |
|------|------|------|---------|
| 1 | **金额单位**：存储用最小单位整数还是元级 Decimal？ | 所有金额计算和传输 | 推荐 DECIMAL(20,8)，覆盖加密货币精度 |
| 2 | **钱包类型枚举值**：center=1, reward=2, anchor=3, agent=4, venue=5？ | RPC 入参 | 待双方确定，写入 IDL |
| 3 | **幂等键组成**：orderNo 单独还是 orderNo + walletType 组合？ | 人工加款一次可能操作多个钱包 | 推荐 orderNo + walletType 组合 |
| 4 | **提现冻结由谁发起**：我方内部冻结还是财务模块调 FreezeBalance？ | 提现链路设计 | 推测我方内部冻结（用户提交时立即冻结更安全） |
| 5 | **充值回调链路**：三方→财务→(RPC)→钱包，还是三方→钱包直接处理？ | 代码放在哪个服务 | 推测三方→财务→钱包（财务持有通道密钥） |
| 6 | **错误码服务编号分配** | 错误码范围 | ser-wallet 需要分配一个 3 位编号 |

### 10.2 我方内部需要决定的

| 序号 | 事项 | 选项 | 推荐 |
|------|------|------|------|
| 1 | 余额并发控制方式 | 悲观锁(FOR UPDATE) vs 乐观锁(版本号) | 悲观锁 — 资金操作频率高，乐观锁重试成本大 |
| 2 | 订单号生成方式 | UUID vs Snowflake vs 前缀+随机数 | 前缀+16位随机数（需求明确：C/T/B/A/M+16位） |
| 3 | 钱包账户创建时机 | 用户注册时批量创建 vs 首次操作时懒创建 | 首次操作时懒创建 — 避免为不活跃用户创建无用记录 |
| 4 | 汇率缓存策略 | 每次查DB vs Redis缓存 | Redis缓存 + 写入时刷新 — 汇率读频率远高于写频率 |
| 5 | 账变流水是否需要消息队列异步写入 | 同步写 vs MQ异步 | 同步写 — 流水是审计要求，必须和余额操作在同一事务 |
| 6 | IDL 拆分粒度 | 一个大文件 vs 按功能域拆分 | 按功能域拆分（wallet/currency/deposit/withdraw/exchange/record） |

---

## 十一、实现阶段建议

### 11.1 阶段总览

```
Phase 1 ─ 工程搭建 + 基础设施
═══════════════════════════════
  · IDL 定义（至少核心 RPC 部分）
  · gen_kitex.sh 生成代码
  · 4 个文件注册（go.work / namesp / rpc_client / gorm_gen 修正）
  · 数据库建表（至少核心表）
  · GORM Gen 生成 model/query/repo
  · cfg/cfg.go（ETCD 配置加载 + Redis 初始化）
  · 错误码定义
  · 枚举定义

Phase 2 ─ 币种配置 + 汇率
═══════════════════════════════
  · B端：币种 CRUD + 基准币种设置 + 汇率日志
  · RPC：GetCurrencyConfig / GetExchangeRate / ConvertAmount
  · 定时任务：汇率更新
  · Redis 缓存：币种/汇率缓存

Phase 3 ─ 钱包核心
═══════════════════════════════
  · wallet_account 表操作
  · 5 个核心余额方法 + 幂等机制 + 账变流水
  · DebitByPriority（消费优先级）
  · 6 个核心 RPC 接口实现
  → 此阶段完成后，财务模块可以开始联调

Phase 4 ─ C端基础功能
═══════════════════════════════
  · 钱包余额查询（C-1）
  · 汇率查询（C-13）
  · 币种列表
  · 兑换预览 + 执行（C-5, C-6）— 纯内部，可独立验证
  · 账变记录（C-11, C-12）
  · 提现账户管理（C-9, C-10）

Phase 5 ─ 充值流程（需财务配合）
═══════════════════════════════
  · 充值方式获取（C-2）— 依赖财务
  · 创建充值订单（C-3）— 依赖财务通道匹配
  · USDT 充值信息（C-4）
  · 重复转账检测
  · 充值回调处理（CreditWallet 已在 Phase 3 实现）
  · 订单超时定时任务

Phase 6 ─ 提现流程（需财务 + KYC 配合）
═══════════════════════════════
  · 提现方式获取（C-7）— 依赖财务 + KYC
  · 创建提现订单 + 冻结（C-8）
  · 提现校验（KYC/限额/姓名/路由规则）
  · Unfreeze / DeductFrozen 已在 Phase 3 实现

Phase 7 ─ 联调验证
═══════════════════════════════
  · 充值全链路：用户→通道→回调→入账
  · 提现全链路：用户→冻结→审核→出款→扣冻结/解冻
  · 人工修正全链路：补单/加款/减款→审核→余额变动
  · 余额对账验证
```

### 11.2 阶段间的依赖关系

```
Phase 1（工程搭建）
    │
    ├──→ Phase 2（币种配置）───┐
    │                         │
    └──→ Phase 3（钱包核心）───┤
                              │
                              ├──→ Phase 4（C端基础）
                              │
                              ├──→ Phase 5（充值）──→ Phase 7（联调）
                              │                         ↑
                              └──→ Phase 6（提现）───────┘

关键路径：Phase 1 → Phase 3 → Phase 5/6 → Phase 7
并行路径：Phase 2 和 Phase 3 可以并行
         Phase 4 可以在 Phase 2+3 完成后立即开始
         Phase 5 和 Phase 6 可以并行但都需要 Phase 3
```

### 11.3 与财务模块的衔接点

```
我方 Phase 3 完成 → 财务模块可以开始联调 6 个核心 RPC
我方 Phase 5 开始 → 需要财务模块的支付方式配置 + 通道匹配 RPC 就绪
我方 Phase 6 开始 → 需要财务模块的提现方式配置 + KYC 服务就绪
Phase 7 联调   → 双方功能都基本就绪
```

---

## 十二、风险与应对

| 风险 | 影响 | 应对方向 |
|------|------|---------|
| 财务模块 RPC 延迟交付 | 充值/提现 C端无法联调 | Phase 2-4 先独立推进，Mock 财务 RPC |
| 需求评审后细节变动 | 接口参数/流程可能调整 | IDL 结构预留扩展性（optional 字段），核心骨架不动 |
| 金额精度问题 | 不同币种精度不同，计算可能出现精度丢失 | 统一用 DECIMAL 存储，业务层统一处理四舍五入 |
| 余额并发操作 | 高并发下可能出现超卖/超扣 | 行锁(FOR UPDATE) + 余额校验在事务内 |
| 汇率 API 全部不可用 | 无法更新实时汇率 | 保持旧值 + 告警，平台汇率有缓存不影响业务 |
| ser-kyc 接口未就绪 | 提现 KYC 校验无法实现 | 先确认 ser-kyc IDL 是否已存在，提前对齐 |
| 活动模块不存在 | 充值奖金选择无数据源 | 预留接口结构，先跳过奖金逻辑 |
| 充值订单表归属不明 | 影响表设计和查询职责 | 推测：我方建平台订单表，财务建通道订单表，评审确认 |

---

## 十三、需要在评审中讨论/确认的事项

### 13.1 与财务模块的接口约定

| 序号 | 约定项 | 当前状态 | 重要程度 |
|------|--------|---------|---------|
| 1 | 6 个核心 RPC 的入参出参（IDL 定义） | 待定义 | 最高 — 联调合同 |
| 2 | 金额单位和精度 | 待确认 | 最高 — 影响所有金额计算 |
| 3 | 钱包类型枚举值 | 待确认 | 高 |
| 4 | 订单号生成规则细节 | 部分确定（前缀+16位） | 中 |
| 5 | 充值回调链路确认 | 推测三方→财务→钱包 | 高 |
| 6 | 提现冻结发起方确认 | 推测我方内部 | 高 |
| 7 | 提现限额和手续费配置归属 | 推测归财务 | 中 |
| 8 | 错误码服务编号分配 | 待分配 | 中 |

### 13.2 产品层面待确认

| 序号 | 问题 | 影响 |
|------|------|------|
| 1 | 钱包账户何时创建？用户注册时还是首次操作时？ | 影响用户注册流程是否需要调钱包 |
| 2 | 兑换赠送比例由谁配置？在哪配？ | 影响兑换流程的数据来源 |
| 3 | 奖金活动数据由谁维护？ | 影响充值流程是否需要对接活动模块 |
| 4 | 提现账户信息的 B 端修改通道？ | 需求说用户不可修改，但运营是否可以？ |
| 5 | 游客 USDT 充值后如何关联到账？ | 游客没有 userId，到账逻辑需要特殊处理 |

---

## 十四、总结

### 14.1 确定性骨架（绕不过的）

1. **IDL 先行** → gen_kitex.sh 生成代码 → 这是一切的起点
2. **4 文件注册** → go.work / namesp / rpc_client / gorm_gen → 融入工程体系
3. **10 张表** → 币种配置/钱包账户/账变流水/充值订单/提现订单/兑换订单/提现账户/汇率日志/稽核要求/基准币种
4. **5 个核心方法** → Credit/Debit/Freeze/Unfreeze/DeductFrozen → 钱包的心脏
5. **6 个核心 RPC** → 与财务模块的联调合同
6. **币种配置 + 汇率管理** → 所有资金操作的数据基础
7. **幂等 + 事务 + 流水** → 资金安全的三道防线
8. **冻结三步模式** → 提现的核心机制

### 14.2 弹性空间（可能变动的）

1. 具体的 IDL 字段定义（评审后可能增减）
2. 接口参数细节（如校验规则的阈值）
3. 奖金活动对接方式（活动模块是否存在）
4. 提现限额/手续费的配置归属
5. 具体的汇率 API 选择
6. 表的字段细节（如是否需要额外的扩展字段）
7. 定时任务的具体频率

### 14.3 实施路径一句话

**先搭工程骨架（IDL + 注册 + 建表），再建核心能力（余额操作 + RPC），然后铺业务功能（币种→兑换→充值→提现），最后联调验证（与财务模块对接全链路）。能独立做的先做，需要联调的 Mock 先行。**
