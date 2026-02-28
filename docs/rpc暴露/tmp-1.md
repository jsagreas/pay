# ser-wallet 需暴露的内部 RPC 接口 — 综合分析

> **文档定位**：ser-wallet 除 B端/C端 HTTP 接口外，需要暴露给工程内其他模块调用的 RPC 接口清单
> **分析依据**：136张需求/原型文档 + 工程全量IDL分析（73个thrift文件/280+方法） + 前期需求分析（c-2/c-3） + 工程代码级分析（c-2/c-3会话） + 模块参考A/B + 已有RPC分析（c-4/d-1/d-2）
> **分析范围**：仅内部 RPC 接口（Kitex TTHeader协议，service-to-service），不含 B端/C端 HTTP 接口（Hertz协议，走gate-back/gate-font网关）
> **核心原则**：余额只有钱包能改，所有涉及余额变更的模块都必须通过 ser-wallet 的 RPC 接口

---

## 一、为什么要梳理 RPC 暴露接口

### 1.1 与 B端/C端接口的本质区别

| 维度 | B端/C端接口 | 内部RPC接口 |
|------|-----------|------------|
| 调用方 | 用户（通过浏览器/App） | 工程内其他 ser-xxx 服务 |
| 入口 | gate-back / gate-font 网关路由 | Kitex RPC 直连（ETCD服务发现） |
| 协议 | HTTP（Hertz框架） | Thrift（Kitex TTHeader） |
| 触发方式 | 用户操作触发 | 服务间程序触发 |
| IDL位置 | service.thrift 中带 `api.post`/`api.get` 注解 | wallet_rpc.thrift 中无HTTP注解 |
| 典型场景 | 用户查余额、用户发起充值 | 财务通知钱包入账、提现冻结/解冻 |

### 1.2 不梳理清楚的后果

- 别人不知道该调我哪个接口 → 联调时才发现没有 → 返工
- 接口定义不对齐（参数/返回值/语义） → 联调时才发现对不上 → 返工
- 遗漏关键接口，某条业务链路跑不通 → 阻塞整个项目进度

### 1.3 工程中已有的RPC接口模式参考

| 模块 | RPC专用文件 | RPC方法示例 | 特点 |
|------|-----------|-----------|------|
| ser-user | user_rpc.thrift | GetUserInfoExpend, BatchGetUserInfo, SearchUserByNickname | 单独文件定义RPC结构体 |
| ser-live | live_rpc.thrift | BatchGetLiveUser, GetLiveRoomInfoByUserId | 返回map/struct |
| ser-blog | model.thrift + service.thrift | `oneway void AddActLog` | 异步单向调用 |
| ser-item | service.thrift | GetItemsByType, BatchGetItemsByIds | 批量查询接口 |
| ser-bag | bag.thrift + service.thrift | AddBagItem, UseBagItem | 资产类操作 |

**ser-wallet 应遵循此模式**：在 `/common/idl/ser-wallet/` 下创建 `wallet_rpc.thrift` 专门定义内部RPC接口的请求/响应结构体。

---

## 二、调用方全景 — 谁需要我提供什么

### 2.1 调用方识别

```
                            ┌─────────────────────┐
                            │     ser-wallet       │
                            │   (RPC接口提供方)     │
                            └──────────┬──────────┘
                                       │
         ┌──────────┬──────────┬───────┴───┬──────────┬──────────┐
         │          │          │           │          │          │
    ┌────┴────┐┌────┴────┐┌───┴────┐ ┌────┴────┐┌───┴────┐┌───┴────┐
    │ser-     ││ser-user ││ 投注/  │ │ 直播/  ││ 活动/ ││ 其他  │
    │finance  ││(用户)   ││ 竞彩   │ │ 礼物   ││ 奖励  ││ 模块  │
    │(财务)   ││         ││ 模块   │ │ 模块   ││ 模块  ││       │
    └─────────┘└─────────┘└────────┘ └────────┘└───────┘└───────┘
     当前需求    当前需求    未来必需    未来必需   未来必需  可预见
     P0/已明确   P0/已明确   可预见      可预见     可预见
```

### 2.2 调用方详细分析

| 调用方 | 优先级 | 依据来源 | 需要的能力 |
|--------|--------|---------|-----------|
| **ser-finance（财务管理）** | **P0 当前需求** | 136张需求文档明确定义充值/提现/人工修正流程 | 余额入账、冻结、解冻、确认扣款、人工加减款、查余额、查币种、查汇率 |
| **ser-user（用户服务）** | **P0 当前需求** | 用户注册必须初始化钱包 | 钱包初始化 |
| **投注/竞彩模块** | **P1 平台核心** | 项目背景：核心域=投注、赔率、结算返奖 | 查余额（下注前校验）、扣余额（下注扣款）、返奖入账 |
| **直播/礼物模块** | **P1 业务必需** | 项目背景：泛娱乐=直播打赏；ser-live IDL中有消费金额/钻石字段 | 打赏扣款、主播收益入账 |
| **活动/奖励模块** | **P2 运营必需** | 需求文档：充值奖金、活动返奖入奖励钱包 | 活动奖金发放到奖励钱包 |
| **结算/返奖模块** | **P2 核心必需** | 项目背景：核心域=结算、返奖 | 返奖金额入账 |

**关键发现**（基于ser-live IDL代码实证）：
- ser-live 的 live_rpc.thrift 中**无任何钱包/余额相关接口**
- ser-live 的 live_consume.thrift 中有 `totalAmount`（消费金额）、`totalDiamond`（消费钻石）字段，但都是**只读记录**
- 说明直播模块的资金操作（打赏扣款/收益上账）不在 ser-live 内部完成，而是通过调用 ser-wallet 的 RPC 接口完成

---

## 三、RPC 接口完整清单（12个 + 1个审计调用）

### 3.1 全景分类

```
ser-wallet 对外暴露 RPC 能力
│
├── 第一类：资金操作原语（5个，核心写操作）
│   ├── W-R1. CreditWallet        上账/加款（充值入账、补单、加款、奖金发放）
│   ├── W-R2. DebitWallet          扣款/减款（人工减款、消费扣款）
│   ├── W-R3. FreezeBalance        冻结余额（提现申请冻结、风控冻结）
│   ├── W-R4. UnfreezeBalance      解冻余额（审核驳回、出款失败返还）
│   └── W-R5. ConfirmDebit         确认扣除已冻结金额（出款成功）
│
├── 第二类：钱包管理能力（1个，写操作）
│   └── W-R6. InitWallet           用户注册初始化多币种钱包
│
├── 第三类：查询能力（6个，只读）
│   ├── W-R7. GetBalance           单用户余额查询
│   ├── W-R8. BatchGetBalance      批量用户余额查询
│   ├── W-R9. GetWalletFlow        资金流水查询
│   ├── CC-R1. GetCurrencyList     获取可用币种列表
│   ├── CC-R2. GetExchangeRate     获取币种汇率
│   └── CC-R3. GetBaseCurrency     获取基准币种信息
│
└── 审计调用（ser-wallet → ser-blog）
    └── oneway void AddActLog      所有写操作完成后异步记录审计日志
```

**总计：12个对外RPC接口 + 1个对外RPC调用**

### 3.2 接口编号与命名对照

| 统一编号 | 接口名称 | c-4.md编号 | d-1.md编号 | d-2.md编号 | 操作类型 |
|---------|---------|-----------|-----------|-----------|---------|
| W-R1 | CreditWallet | W-R2/W-R6/W-R8合并 | CreditBalance | CreditWallet | 写/入账 |
| W-R2 | DebitWallet | W-R7(ManualDebit) | DebitBalance | DebitWallet | 写/扣款 |
| W-R3 | FreezeBalance | W-R3 | FreezeBalance | FreezeBalance | 写/冻结 |
| W-R4 | UnfreezeBalance | W-R4 | UnfreezeBalance | UnfreezeBalance | 写/解冻 |
| W-R5 | ConfirmDebit | W-R5 | ConfirmWithdrawalDeduct | ConfirmDebit | 写/确认 |
| W-R6 | InitWallet | — | — | InitWallet | 写/初始化 |
| W-R7 | GetBalance | W-R1 | GetBalance | GetBalance | 只读 |
| W-R8 | BatchGetBalance | — | — | BatchGetBalance | 只读 |
| W-R9 | GetWalletFlow | — | — | GetWalletFlow | 只读 |
| CC-R1 | GetCurrencyList | CC-R1 | GetCurrencyList | GetCurrencyList | 只读 |
| CC-R2 | GetExchangeRate | CC-R2 | GetExchangeRate | GetExchangeRate | 只读 |
| CC-R3 | GetBaseCurrency | CC-R3 | GetBaseCurrency | GetBaseCurrency | 只读 |

**整合说明**：
- c-4.md 中 W-R2(CreditDeposit) + W-R6(ManualCredit) + W-R8(CreditCorrection) → 统一为 **W-R1 CreditWallet**（通过 bizType 区分充值/补单/加款/奖金等不同入账场景）
- c-4.md 中 W-R7(ManualDebit) → 统一为 **W-R2 DebitWallet**（通过 bizType 区分人工减款/消费扣款等不同扣款场景）
- **统一原因**：上账就是上账、扣款就是扣款，核心操作是同一件事，不同的只是业务来源（bizType）。拆成多个接口会导致逻辑重复、维护成本高。参考工程A（p9-services）也是统一的 ModifyBalance 接口 + bizType 区分。
- d-2.md 提出的 InitWallet、BatchGetBalance、GetWalletFlow 是必要补充，c-4.md 未覆盖但业务上需要。

---

## 四、第一类：资金操作原语（5个）

> 核心设计原则：原子性、幂等性、可审计、非负保护
> 这是 ser-wallet 对外最核心的能力。平台内任何模块需要变动用户余额，都必须通过这5个原语接口。

### 4.1 W-R1. CreditWallet — 上账/加款

```
接口名称：CreditWallet
操作类型：写（余额增加）
幂等要求：必须（refOrderNo + bizType 唯一索引）
重要程度：★★★★★
```

**调用方与触发场景**：

| 调用方 | 场景 | bizType | 目标钱包(walletType) | 依据 |
|--------|------|---------|---------------------|------|
| ser-finance | 充值支付成功后入账 | 1（充值） | 1（中心钱包） | 需求文档5-1~5-7充值模块 |
| ser-finance | 充值奖金入账 | 2（充值奖金） | 2（奖励钱包） | 需求文档5-6额外奖金选择规则 |
| ser-finance | 充值补单审核通过 | 3（补单） | 1（中心钱包） | 财务需求2-5人工修正 |
| ser-finance | 人工加款审核通过 | 4（人工加款） | 1/2/3/4（按操作指定） | 财务需求2-5人工修正 |
| 活动系统 | 活动奖金发放 | 5（活动奖金） | 2（奖励钱包） | 需求文档额外奖金、投注返奖 |
| 兑换内部 | 法币→BSB兑换入账 | 6（兑换入） | 1（中心钱包） | 需求文档6兑换模块 |
| 兑换内部 | 兑换赠送部分 | 7（兑换赠送） | 2（奖励钱包） | 需求文档6兑换模块 |
| 直播模块 | 主播打赏收益上账 | 8（打赏收入） | 3（主播钱包） | 项目背景：泛娱乐直播打赏 |
| 结算模块 | 投注返奖入账 | 9（投注返奖） | 按投注比例分入对应钱包 | 项目背景：核心域=结算返奖 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| userId | i64 | 是 | 目标用户ID |
| currencyCode | string | 是 | 币种代码（BSB/VND/IDR/THB） |
| walletType | i32 | 是 | 钱包类型 1中心 2奖励 3主播 4代理 5场馆 |
| amount | i64 | 是 | 金额（最小单位整数，必须>0） |
| bizType | i32 | 是 | 业务类型（区分入账原因，见上表） |
| refOrderNo | string | 是 | 关联业务订单号（幂等键） |
| remark | string | 否 | 备注信息 |
| turnoverMultiple | i32 | 否 | 流水倍数要求（奖金场景：需完成N倍流水才可提现） |

**返回**：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 操作是否成功 |
| afterBalance | i64 | 操作后该钱包可用余额 |
| message | string | 失败原因（如"订单已处理"） |

**内部执行逻辑**：
```
1. 幂等检查：(refOrderNo, bizType) 是否已存在于 wallet_transaction 表 → 已存在直接返回成功
2. 开启 DB 事务
3. SELECT FOR UPDATE 锁定 wallet_account 行（userId + currencyCode + walletType）
4. available_balance += amount
5. INSERT wallet_transaction 流水记录（balance_before, balance_after, refOrderNo, bizType）
6. 如有 turnoverMultiple → INSERT wallet_audit 稽核记录
7. 提交事务
8. 异步调用 ser-blog.AddActLog 记录审计日志
9. 返回成功
```

**关键设计要点**：
- 幂等性：`(refOrderNo, bizType)` 唯一索引，三方回调可能重复推送
- 并发控制：SELECT FOR UPDATE 行级锁，防止同一用户同时充值导致余额覆盖
- 流水不可变：wallet_transaction 记录只 INSERT 不 UPDATE/DELETE
- 充值奖金时调用方需同时发两次 CreditWallet（一次 walletType=1 入中心，一次 walletType=2 入奖励），或 ser-finance 先计算好分别入账

---

### 4.2 W-R2. DebitWallet — 扣款/减款

```
接口名称：DebitWallet
操作类型：写（余额减少）
幂等要求：必须（refOrderNo + bizType 唯一索引）
重要程度：★★★★★
```

**调用方与触发场景**：

| 调用方 | 场景 | bizType | 扣款规则 | 依据 |
|--------|------|---------|---------|------|
| ser-finance | 人工减款审核通过 | 10（人工减款） | 按操作指定的子钱包扣 | 财务需求2-5人工修正 |
| 兑换内部 | 法币→BSB兑换扣除法币 | 11（兑换出） | 扣中心钱包 | 需求文档6兑换模块 |
| 直播模块 | 用户打赏送礼扣款 | 12（打赏支出） | 先中心后奖励 | 需求文档4-1消费优先级 |
| 商城模块 | 商品购买扣款 | 13（商城消费） | 先中心后奖励 | 项目背景 |
| 投注模块 | 投注扣款 | 14（投注扣款） | 先中心后奖励 | 项目背景 |

**接口契约**：与 CreditWallet 结构基本一致（userId, currencyCode, walletType, amount, bizType, refOrderNo, remark）

**关键差异点**：
- **防超扣**：SQL级保证 `available_balance - amount >= 0`，WHERE条件中加 `AND (available_balance - frozen_amount) >= #{amount}`
- **消费优先级**：先扣中心钱包(walletType=1)，不足部分扣奖励钱包(walletType=2)
  - 依据：需求文档4-1钱包总体设计明确"消费优先扣中心钱包"
  - 实现方式：调用方可指定 walletType 精确扣款，也可不指定由钱包内部按优先级自动分配
- **余额不足处理**：两种策略（由调用方通过参数决定）
  - 策略A：余额不足→拒绝（投注、消费场景）
  - 策略B：余额不足→扣实际可用余额（人工减款场景，需求明确"减款不足则扣实际"）

---

### 4.3 W-R3. FreezeBalance — 冻结余额

```
接口名称：FreezeBalance
操作类型：写（可用余额 → 冻结余额）
幂等要求：必须（refOrderNo 唯一索引）
重要程度：★★★★★
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-wallet内部（函数调用） | 用户提交提现订单 | 需求文档7-1~7-6提现模块 |
| ser-finance | 人工冻结操作（风控场景） | 财务需求2-5人工修正 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| userId | i64 | 是 | 目标用户ID |
| currencyCode | string | 是 | 币种代码 |
| amount | i64 | 是 | 冻结金额 |
| refOrderNo | string | 是 | 关联提现订单号（幂等键） |

**返回**：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 操作是否成功 |
| availableBalance | i64 | 冻结后可用余额 |
| frozenBalance | i64 | 冻结后冻结总额 |
| message | string | 失败原因（如"余额不足"） |

**内部执行逻辑**：
```
1. 幂等检查
2. 开启事务
3. SELECT FOR UPDATE 锁定 wallet_account
4. 校验 available_balance - frozen_amount >= amount
5. frozen_amount += amount （available_balance 不变，但可用余额 = balance - frozen 减少了）
6. INSERT wallet_transaction (tx_type=冻结)
7. INSERT freeze_detail (记录本次冻结明细：从哪个钱包冻了多少)
8. 提交事务
```

**关键设计要点**：
- 冻结≠扣款：balance不变，frozen_amount增加，available余额减少
- 冻结来源分配：先冻中心钱包，不足再冻奖励钱包（与消费优先级一致）
- 需记录冻结明细（freeze_detail），后续解冻/确认时需要知道从哪些子钱包冻了多少

---

### 4.4 W-R4. UnfreezeBalance — 解冻余额

```
接口名称：UnfreezeBalance
操作类型：写（冻结余额 → 可用余额）
幂等要求：必须（refOrderNo 唯一索引）
重要程度：★★★★★
互斥关系：与 W-R5 ConfirmDebit 互斥 — 同一笔冻结只能走其中一条路
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-finance | 提现风控审核驳回 | 依赖关系文档：提现状态机 |
| ser-finance | 提现财务审核驳回 | 需求文档：提现审核流程 |
| ser-finance | 三方出款失败回调 | 需求文档：提现出款失败处理 |
| ser-finance | 人工解冻（风控场景） | 财务需求2-5人工修正 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| userId | i64 | 是 | 目标用户ID |
| currencyCode | string | 是 | 币种代码 |
| refOrderNo | string | 是 | 关联提现订单号（通过此查找冻结明细） |
| reason | string | 是 | 解冻原因（"风控驳回"/"财务驳回"/"出款失败"） |

**内部执行逻辑**：
```
1. 幂等检查
2. 状态校验：查找该 refOrderNo 的 freeze_detail，必须为"已冻结"状态
3. 互斥校验：如果已被 ConfirmDebit 处理 → 拒绝（不可能既解冻又扣除）
4. 开启事务
5. 按 freeze_detail 记录的各子钱包冻结金额，分别还原
   - wallet_account.frozen_amount -= freeze_detail.amount
6. UPDATE freeze_detail 状态 → 已解冻
7. INSERT wallet_transaction (tx_type=解冻，记录解冻原因)
8. 提交事务
```

---

### 4.5 W-R5. ConfirmDebit — 确认扣除已冻结金额

```
接口名称：ConfirmDebit
操作类型：写（冻结余额 → 实际扣除，资金永久离开用户账户）
幂等要求：必须（refOrderNo 唯一索引）
重要程度：★★★★★
互斥关系：与 W-R4 UnfreezeBalance 互斥
这是提现链路的终点接口
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-finance | 三方出款成功回调 | 链路关系文档：出款成功→ConfirmDebit |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| userId | i64 | 是 | 目标用户ID |
| currencyCode | string | 是 | 币种代码 |
| refOrderNo | string | 是 | 关联提现订单号 |

**内部执行逻辑**：
```
1. 幂等检查
2. 状态校验：freeze_detail 必须为"已冻结"状态
3. 互斥校验：如果已被 UnfreezeBalance 处理 → 拒绝
4. 开启事务
5. 按 freeze_detail 记录的各子钱包冻结金额：
   - wallet_account.balance -= freeze_detail.amount （余额真正减少）
   - wallet_account.frozen_amount -= freeze_detail.amount （冻结释放）
6. UPDATE freeze_detail 状态 → 已扣除
7. INSERT wallet_transaction (tx_type=提现确认扣除，balance_before/after)
8. 提交事务
```

### 4.6 冻结三阶段完整状态流转

```
用户提交提现 ──► FreezeBalance（W-R3 冻结）
                        │
          ┌─────────────┴─────────────┐
          │                           │
    审核通过 + 出款成功           审核驳回 / 出款失败
          │                           │
    ConfirmDebit（W-R5）       UnfreezeBalance（W-R4）
    确认扣除                     解冻返还
          │                           │
    balance 减少               balance 不变
    frozenAmount 减少          frozenAmount 减少
    资金永久离开用户账户       资金恢复可用
```

**为什么采用三阶段**：
- 提现存在多级审核（风控→财务→出款），审核周期可能长达数小时甚至天
- 冻结期间用户不能用这笔钱消费（防止余额不足），但审核不过可全额返还
- 参考工程A（p9-services wallet-service）和工程B（tg-services wallet-server）均采用此三阶段模式，生产验证成熟

---

## 五、第二类：钱包管理能力（1个）

### 5.1 W-R6. InitWallet — 钱包初始化

```
接口名称：InitWallet
操作类型：写（创建钱包账户记录）
幂等要求：必须（userId 去重，重复调用不报错不重复创建）
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-user | 用户注册成功后 | 每个用户必须有钱包才能进行充值/消费/投注等操作 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| userId | i64 | 是 | 新注册用户ID |

**返回**：

| 字段 | 类型 | 说明 |
|------|------|------|
| success | bool | 操作是否成功 |

**内部逻辑**：
```
1. 幂等检查：该 userId 是否已有钱包记录 → 有则直接返回成功
2. 查询当前所有启用状态的币种列表
3. 为每个币种创建钱包账户记录（wallet_account）：
   - 每个币种创建 N 个子钱包（中心/奖励/主播/代理等已启用的钱包类型）
   - 初始余额 = 0，冻结金额 = 0
4. 设置用户默认币种偏好（按系统配置的默认币种）
```

**关键设计要点**：
- 后续新增币种时，存量用户的钱包初始化走另一个内部逻辑（懒加载或批处理），不走此RPC
- 参考工程B：tg-services wallet-server 通过 `POST /wallet/create` 为用户创建链上地址，我方是创建多币种账户记录

---

## 六、第三类：查询能力（6个）

### 6.1 W-R7. GetBalance — 单用户余额查询

```
接口名称：GetBalance
操作类型：只读
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-finance | 人工加/减款弹窗展示当前各子钱包余额 | 依赖关系文档5.4人工修正链路 |
| ser-finance | 充值/提现记录页展示用户余额 | 财务需求文档 |
| 直播模块 | 判断用户余额是否足够送礼物 | 项目背景 |
| 投注模块 | 下注前校验余额 | 项目背景 |
| 活动系统 | 判断用户是否满足活动余额门槛 | 需求文档 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| userId | i64 | 是 | 用户ID |
| currencyCode | string | 否 | 币种代码（不传返回所有币种） |
| walletType | i32 | 否 | 钱包类型（不传返回所有子钱包） |

**返回**：

| 字段 | 类型 | 说明 |
|------|------|------|
| userId | i64 | 用户ID |
| wallets | list\<WalletBalanceInfo\> | 各子钱包余额列表 |
| totalAvailable | i64 | 资金钱包余额（中心+奖励的可用合计） |

WalletBalanceInfo：

| 字段 | 类型 | 说明 |
|------|------|------|
| currencyCode | string | 币种代码 |
| walletType | i32 | 钱包类型 |
| walletTypeName | string | "中心钱包" |
| balance | i64 | 账面余额 |
| frozenAmount | i64 | 冻结金额 |
| availableBalance | i64 | 可用余额 = balance - frozenAmount |

---

### 6.2 W-R8. BatchGetBalance — 批量用户余额查询

```
接口名称：BatchGetBalance
操作类型：只读
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-finance | B端用户列表页展示多个用户余额 | 避免N+1调用 |
| 运营后台 | 批量查看活跃用户资金情况 | 运营需求 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| userIds | list\<i64\> | 是 | 用户ID列表（建议上限100） |
| currencyCode | string | 是 | 币种代码 |

**返回**：map\<i64, list\<WalletBalanceInfo\>\>（userId → 余额列表）

**设计依据**：参考工程中 ser-user 的 `BatchGetUserInfo(list<i64>)` 和 ser-live 的 `BatchGetLiveUser` 模式，批量查询是工程标准模式。

---

### 6.3 W-R9. GetWalletFlow — 资金流水查询

```
接口名称：GetWalletFlow
操作类型：只读
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-finance | B端查看用户资金流水明细 | 财务审计需求 |
| ser-finance | 对账核实时查询流水 | 需求文档非功能性要求 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| userId | i64 | 是 | 用户ID |
| currencyCode | string | 否 | 币种代码 |
| walletType | i32 | 否 | 钱包类型 |
| bizType | i32 | 否 | 业务类型筛选 |
| startTime | i64 | 否 | 开始时间戳 |
| endTime | i64 | 否 | 结束时间戳 |
| pageNo | i32 | 是 | 页码 |
| pageSize | i32 | 是 | 每页条数 |

**返回**：分页流水列表（含 refOrderNo, bizType, amount, direction, balanceBefore, balanceAfter, createTime）

---

### 6.4 CC-R1. GetCurrencyList — 获取可用币种列表

```
接口名称：GetCurrencyList
操作类型：只读
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-finance | 所有B端页面的币种下拉筛选 | 财务管理产品原型：通道配置/充值记录/提现记录/统计页面均含币种筛选 |
| 其他模块 | 任何需要币种列表的场景 | 币种配置是平台基础设施 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| status | i32 | 否 | 不传=只返回启用的，传0=全部 |

**返回**：list\<CurrencyInfo\>

CurrencyInfo：

| 字段 | 类型 | 说明 |
|------|------|------|
| currencyCode | string | "VND"/"BSB"/"USDT" |
| currencyName | string | "越南盾" |
| currencyType | i32 | 1=法币 2=加密 3=平台币 |
| currencySymbol | string | "₫"/"$"/"₿" |
| decimalPlaces | i32 | 小数位数（法币2位，加密8位） |
| thousandSep | string | 千分位符号 |
| iconUrl | string | 币种图标URL |
| sortOrder | i32 | 排序权重 |
| status | i32 | 1=启用 0=禁用 |

---

### 6.5 CC-R2. GetExchangeRate — 获取币种汇率

```
接口名称：GetExchangeRate
操作类型：只读
```

**调用方与触发场景**：

| 调用方 | 场景 | 依据 |
|--------|------|------|
| ser-finance | 报表换算（基准币种统一汇总） | 财务需求E3/E4充值/提现记录的"基准换算"列 |
| 结算模块 | 跨币种结算时的汇率参考 | 项目背景 |

**接口契约**：

| 字段 | 类型 | 必传 | 说明 |
|------|------|------|------|
| currencyCode | string | 是 | 币种代码 |
| rateType | i32 | 否 | 1=实时 2=平台 3=入款 4=出款（不传=返回全部） |

**返回**：

| 字段 | 类型 | 说明 |
|------|------|------|
| currencyCode | string | 币种代码 |
| rateRealtime | string | 实时汇率（decimal字符串） |
| ratePlatform | string | 平台汇率 |
| rateDeposit | string | 入款汇率（= 实时 × (1+浮动%)） |
| rateWithdrawal | string | 出款汇率（= 实时 × (1-浮动%)） |
| updatedAt | i64 | 最后更新时间戳 |

**依据**：币种配置需求文档3-2平台基准币种、4-1汇率应用场景

---

### 6.6 CC-R3. GetBaseCurrency — 获取基准币种

```
接口名称：GetBaseCurrency
操作类型：只读
调用频率：低频（基准币种设置后不变，可长期缓存）
```

**调用方**：ser-finance（统计报表确定换算基准）

**接口契约**：

| 字段 | 类型 | 说明 |
|------|------|------|
| 入参 | 无 | — |
| currencyCode | string | "USDT" |
| currencyName | string | "泰达币" |
| decimalPlaces | i32 | 8 |
| anchorRate | string | BSB锚定比例（"10"表示10BSB=1USDT） |

**依据**：币种配置需求文档B3——基准币种推荐USDT，配置后不可更改

---

## 七、审计调用（ser-wallet → ser-blog）

ser-wallet 的所有写操作（W-R1~W-R6）完成后，需异步调用 ser-blog 的审计日志接口：

```thrift
// 参照工程现有 ser-blog/service.thrift
oneway void AddActLog(1: model.ActAddReq req)
```

- 调用方式：`oneway`（单向，不等待响应）
- 调用时机：事务提交成功后异步发送
- 记录内容：操作人、操作模块、操作动作、操作IP、是否成功
- 不阻塞主流程：即使 ser-blog 不可用也不影响资金操作

---

## 八、接口间的关联关系与约束

### 8.1 互斥关系

```
W-R4 (UnfreezeBalance) ←互斥→ W-R5 (ConfirmDebit)
  同一笔冻结（同一个 refOrderNo）只能走其中一条路
  已解冻的不能再确认扣除，已确认扣除的不能再解冻
```

### 8.2 依赖关系

```
W-R3 (FreezeBalance)
  ├── 前置条件：用户已有钱包（W-R6 InitWallet 已执行）
  ├── 后续操作：W-R4 (UnfreezeBalance) 或 W-R5 (ConfirmDebit)
  └── 不可单独存在：冻结后必须有终态处理

W-R1 (CreditWallet)
  └── 前置条件：用户已有钱包（W-R6 InitWallet 已执行）

W-R2 (DebitWallet)
  └── 前置条件：用户已有钱包且余额充足
```

### 8.3 业务链路中的RPC调用序列

#### 充值链路：
```
三方回调成功 → ser-finance 验签
  → ser-finance 调 W-R1 CreditWallet (bizType=1, walletType=1, 充值金额入中心钱包)
  → 如有充值奖金 → ser-finance 调 W-R1 CreditWallet (bizType=2, walletType=2, 奖金入奖励钱包)
  → 完成
涉及RPC：W-R1（1~2次调用）
```

#### 提现链路：
```
用户提交提现 → ser-wallet 内部调 W-R3 FreezeBalance (冻结金额)
  → ser-wallet 创建提现订单 → 通知 ser-finance
  → ser-finance 风控审核
    ├── 驳回 → ser-finance 调 W-R4 UnfreezeBalance (解冻返还)
    └── 通过 → ser-finance 财务审核
        ├── 驳回 → ser-finance 调 W-R4 UnfreezeBalance
        └── 通过 → ser-finance 调三方出款
            ├── 失败 → ser-finance 调 W-R4 UnfreezeBalance
            └── 成功 → ser-finance 调 W-R5 ConfirmDebit (确认扣除)
涉及RPC：W-R3 + (W-R4 或 W-R5)
```

#### 兑换链路（内部，不走RPC）：
```
用户法币→BSB兑换
  → ser-wallet 内部调 DebitWallet逻辑 (bizType=11, 扣法币)
  → ser-wallet 内部调 CreditWallet逻辑 (bizType=6, BSB入中心)
  → 如有赠送 → CreditWallet逻辑 (bizType=7, 赠送入奖励)
涉及RPC：0（全部内部函数调用，同一事务内完成）
```

#### 人工修正链路：
```
财务B端创建加/减款 → 审核人审批通过
  → 加款：ser-finance 调 W-R1 CreditWallet (bizType=4, 按指定子钱包加款)
  → 减款：ser-finance 调 W-R2 DebitWallet (bizType=10, 按指定子钱包减款)
  → 修正前查余额：ser-finance 调 W-R7 GetBalance
涉及RPC：W-R7 + (W-R1 或 W-R2)
```

---

## 九、与现有工程IDL规范的对照

### 9.1 需遵循的规范

基于对工程73个thrift文件的完整分析，ser-wallet 的 RPC IDL 必须遵循：

| 规范项 | 工程标准 | ser-wallet 应用 |
|--------|---------|----------------|
| Service命名 | `{Module}Service` | `WalletService` |
| Namespace | `namespace go ser_{module}` | `namespace go ser_wallet` |
| 请求结构命名 | `{Operation}Req` | `CreditWalletReq`, `GetBalanceReq` |
| 响应结构命名 | `{Operation}Resp` | `CreditWalletResp`, `GetBalanceResp` |
| 字段编号 | 从1开始连续 | 遵循 |
| 金额类型 | i64（最小单位整数） | 遵循 |
| 分页字段 | pageNo/pageSize/total/pageList | 遵循（W-R9 GetWalletFlow） |
| 批量查询 | `Batch` 前缀 + 返回 `map` | `BatchGetBalance` → `map<i64, list>` |
| 异步调用 | `oneway void` | 审计日志调用 ser-blog |
| RPC专用文件 | `xxx_rpc.thrift` | `wallet_rpc.thrift` |

### 9.2 IDL文件组织建议

```
/common/idl/ser-wallet/
├── service.thrift           # 主服务入口（include所有子文件，定义 WalletService）
├── wallet.thrift            # 钱包基础模型（WalletBalanceInfo等）
├── wallet_rpc.thrift        # RPC专用请求/响应结构体（12个接口的Req/Resp）
├── currency.thrift          # 币种配置模型（CurrencyInfo等）
├── transaction.thrift       # 交易流水模型（TransactionInfo等）
└── freeze.thrift            # 冻结明细模型（FreezeDetail等）
```

---

## 十、写操作的统一安全约束

以下约束适用于所有5个资金操作原语（W-R1~W-R5）+ 初始化（W-R6）：

| 约束项 | 要求 | 实现方式 |
|--------|------|---------|
| **幂等性** | 相同操作不重复执行 | `(refOrderNo, bizType)` 唯一索引 |
| **原子性** | 余额变更+流水写入同一事务 | DB事务 + SELECT FOR UPDATE |
| **非负保护** | 余额不能扣成负数 | SQL WHERE `available_balance - frozen_amount >= #{amount}` |
| **并发控制** | 同用户同币种操作串行 | SELECT FOR UPDATE 行级锁 |
| **可审计** | 所有操作可追溯 | wallet_transaction 不可变流水 + ser-blog 审计日志 |
| **可追溯** | 每笔操作关联业务订单 | refOrderNo 关联 + balance_before/after 记录 |

---

## 十一、待确认事项

以下事项在当前需求文档中未完全明确，需要在后续迭代中确认：

| 编号 | 待确认项 | 影响范围 | 当前处理策略 |
|------|---------|---------|-------------|
| Q1 | 币种配置模块（CC-R1~R3）是否和钱包在同一个 ser-wallet 服务中 | IDL文件组织 | 暂定合在一起（同一个 WalletService），如需拆分后续调整 |
| Q2 | 充值奖金入账是调用方拆成两次 CreditWallet，还是一次调用内部拆分 | W-R1 接口设计 | 暂定调用方拆两次（更清晰），接口保持简单 |
| Q3 | 投注模块具体的 bizType 编号范围 | W-R1/W-R2 的 bizType 枚举 | 预留 14~20 给投注相关，具体等投注模块需求明确 |
| Q4 | 场馆钱包（walletType=5）的转入/转出是否需要单独 RPC | 接口是否新增 | 需求标注"预留"，暂不实现，后续按需新增 |
| Q5 | 主播钱包转入中心钱包是否需要独立 RPC | 是否新增 TransferBetweenWallets | 需求提及"可转入中心钱包"，待确认是走 C端接口还是 RPC |
| Q6 | 汇率定时拉取是 ser-cron 调 ser-wallet 还是 ser-wallet 内部定时 | 架构选择 | 暂定 ser-wallet 内部定时任务（减少跨服务依赖） |

---

## 十二、总结

### 12.1 数量统计

| 类别 | 数量 | 接口 |
|------|------|------|
| 资金操作原语（写） | 5个 | CreditWallet, DebitWallet, FreezeBalance, UnfreezeBalance, ConfirmDebit |
| 钱包管理（写） | 1个 | InitWallet |
| 钱包查询（读） | 3个 | GetBalance, BatchGetBalance, GetWalletFlow |
| 币种配置查询（读） | 3个 | GetCurrencyList, GetExchangeRate, GetBaseCurrency |
| **合计** | **12个** | — |

### 12.2 按调用方分布

| 调用方 | 会调用的接口 | 数量 |
|--------|-----------|------|
| ser-finance（财务） | W-R1~R5, W-R7~R9, CC-R1~R3 | 11个（全部） |
| ser-user（用户） | W-R6 InitWallet | 1个 |
| 投注/竞彩（未来） | W-R1, W-R2, W-R3~R5, W-R7 | 6个 |
| 直播/礼物（未来） | W-R1, W-R2, W-R7 | 3个 |
| 活动/奖励（未来） | W-R1, W-R7 | 2个 |

### 12.3 按优先级排序

| 优先级 | 接口 | 理由 |
|--------|------|------|
| P0 必须首批实现 | W-R1 CreditWallet | 充值核心链路：没有入账能力，充值流程跑不通 |
| P0 必须首批实现 | W-R3 FreezeBalance | 提现核心链路：没有冻结能力，提现流程跑不通 |
| P0 必须首批实现 | W-R4 UnfreezeBalance | 提现核心链路：审核驳回必须能解冻 |
| P0 必须首批实现 | W-R5 ConfirmDebit | 提现核心链路：出款成功必须能确认扣除 |
| P0 必须首批实现 | W-R7 GetBalance | 人工修正前置：查余额展示 |
| P0 必须首批实现 | CC-R1 GetCurrencyList | 财务所有页面的币种下拉 |
| P1 紧随其后 | W-R2 DebitWallet | 人工减款、消费扣款 |
| P1 紧随其后 | W-R6 InitWallet | 用户注册流程需要 |
| P1 紧随其后 | CC-R2 GetExchangeRate | 财务报表换算 |
| P2 按需实现 | W-R8 BatchGetBalance | 批量查询优化 |
| P2 按需实现 | W-R9 GetWalletFlow | 财务审计查询 |
| P2 按需实现 | CC-R3 GetBaseCurrency | 统计基准 |
