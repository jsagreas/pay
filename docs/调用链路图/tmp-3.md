# ser-wallet 全链路请求流转图 — 宏观调用链路深度分析

> **定位**：以"上帝视角"观测每一个请求从入口到终态的完整流转路径，屏蔽内部实现细节，聚焦跨模块边界、关键节点、复杂度热点，作为编码阶段的指导方针。
> **粒度**：请求级（一个接口=一条链路），每条链路标注"谁发起→经过哪些模块→最终状态"。

---

## 目录

- [第一部分：请求基础设施层 — HTTP到RPC的通用链路](#第一部分请求基础设施层--http到rpc的通用链路)
- [第二部分：模块全景关系图 — 谁和谁打交道](#第二部分模块全景关系图--谁和谁打交道)
- [第三部分：C端接口逐一链路分析（16条）](#第三部分c端接口逐一链路分析16条)
- [第四部分：B端接口逐一链路分析（9条）](#第四部分b端接口逐一链路分析9条)
- [第五部分：RPC暴露接口逐一链路分析（13条）](#第五部分rpc暴露接口逐一链路分析13条)
- [第六部分：三大业务主链路时序图](#第六部分三大业务主链路时序图)
- [第七部分：内部子模块调用拓扑](#第七部分内部子模块调用拓扑)
- [第八部分：六大原子操作流转详解](#第八部分六大原子操作流转详解)
- [第九部分：依赖启动顺序与并行开发策略](#第九部分依赖启动顺序与并行开发策略)
- [第十部分：复杂度热力图与挑战识别](#第十部分复杂度热力图与挑战识别)
- [第十一部分：异常链路与降级方案](#第十一部分异常链路与降级方案)
- [第十二部分：关键结论与开发指导](#第十二部分关键结论与开发指导)

---

## 第一部分：请求基础设施层 — HTTP到RPC的通用链路

### 1.1 通用请求链路（所有接口共享）

每个请求不管是C端还是B端，都经过以下固定链路：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        通用请求链路（7层流转）                               │
│                                                                             │
│  Client (App/Web/Admin)                                                     │
│     │                                                                       │
│     ▼ HTTPS                                                                 │
│  ① gate-font / gate-back (Hertz HTTP Server)                               │
│     │  路由匹配 → POST /api/wallet/xxx                                      │
│     │  中间件链：                                                            │
│     │    ├── IP黑白名单检查                                                  │
│     │    ├── Token解析 → 提取userID                                          │
│     │    ├── 权限校验（B端额外RBAC校验）                                      │
│     │    └── 请求参数绑定                                                    │
│     │                                                                       │
│     ▼ RPC (Kitex, TTHeader协议, 3s超时, ETCD服务发现)                        │
│  ② ser-wallet (Kitex RPC Server)                                            │
│     │  handler.go → 方法路由分发                                              │
│     │                                                                       │
│     ▼                                                                       │
│  ③ internal/ser/xxx_service.go (业务逻辑层)                                  │
│     │  参数校验 → 业务逻辑 → 跨模块RPC调用（如需要）                           │
│     │                                                                       │
│     ▼                                                                       │
│  ④ internal/rep/xxx_rep.go (数据访问层)                                      │
│     │  GORM操作 → SQL构建 → 事务管理                                         │
│     │                                                                       │
│     ▼                                                                       │
│  ⑤ TiDB (MySQL兼容) + Redis (缓存/分布式锁)                                 │
│     │                                                                       │
│     ▼                                                                       │
│  ⑥ 返回路径: rep → ser → handler → Kitex Response                           │
│     │                                                                       │
│     ▼                                                                       │
│  ⑦ gate 组装HTTP Response → Client                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 RPC调用基础设施

```
RPC Client工厂模式：
  rpc/rpc_client.go 中的 sync.Once 单例模式
  例: rpc.KycClient() → 首次调用初始化连接(ETCD Resolver + 3s超时)，后续复用

RPC调用参数：
  协议: TTHeader (Kitex原生二进制协议)
  超时: 3秒 (client.WithRPCTimeout)
  发现: ETCD (client.WithResolver)
  序列化: Thrift Binary

RPC调用链路：
  ser-wallet → rpc.XxxClient() → ETCD解析目标地址 → 建立连接 → 发送请求 → 等待响应(≤3s)
```

### 1.3 C端 vs B端链路差异

```
C端请求链路（gate-font）：
  Client → gate-font → [IP检查 → Token → Auth] → RPC → ser-wallet
  特点: 面向终端用户, userID从Token中提取, 不信任前端传入的uid

B端请求链路（gate-back）：
  Admin → gate-back → [IP检查 → Token → RBAC权限] → RPC → ser-wallet
  特点: 面向运营人员, 额外RBAC权限校验, operatorId用于审计日志
  额外动作: 所有写操作需调用 ser-blog.AddActLog 记录审计日志

RPC被调链路（其他服务直接调用）：
  外部ser → rpc.WalletClient() → ETCD → ser-wallet.handler → ser → rep → DB
  特点: 无gate层, 无中间件, 调用方负责身份验证, wallet只验证参数+幂等
```

---

## 第二部分：模块全景关系图 — 谁和谁打交道

### 2.1 模块间调用关系全景

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    平台全景                              │
                    │                                                         │
  ┌──────────┐      │   ┌──────────┐     ┌──────────┐     ┌──────────┐       │
  │ C端 App  │──────│──▷│ gate-font│────▷│ser-wallet│     │ ser-kyc  │       │
  └──────────┘      │   └──────────┘     │          │────▷│ (KYC查询)│       │
                    │                    │          │     └──────────┘       │
  ┌──────────┐      │   ┌──────────┐     │          │     ┌──────────┐       │
  │ B端 Admin│──────│──▷│ gate-back│────▷│          │────▷│ ser-user │       │
  └──────────┘      │   └──────────┘     │          │     │(用户信息) │       │
                    │                    │          │     └──────────┘       │
                    │                    │          │     ┌──────────┐       │
                    │                    │          │────▷│ ser-blog │       │
                    │                    │          │     │(审计日志) │       │
                    │                    │          │     └──────────┘       │
                    │                    │          │     ┌──────────┐       │
                    │                    │          │────▷│  ser-s3  │       │
                    │                    │          │     │(图标上传) │       │
                    │                    │          │     └──────────┘       │
                    │                    │          │                         │
                    │                    │          │◁───────────────┐       │
                    │                    │          │  ┌──────────┐  │       │
                    │                    │          │◁─│ 财务模块  │──┘       │
                    │                    │          │─▷│(支付/配置)│          │
                    │                    │          │  └──────────┘          │
                    │                    │          │       ▲                 │
                    │                    │          │       │                 │
                    │                    │          │  ┌──────────┐          │
                    │                    │          │  │三方支付渠道│          │
                    │                    │          │  └──────────┘          │
                    │                    │          │                         │
                    │                    │          │◁──┌──────────┐         │
                    │                    │          │   │ 投注模块  │         │
                    │                    │          │   │(扣款/返奖)│         │
                    │                    │          │   └──────────┘         │
                    │                    │          │                         │
                    │                    │          │◁──┌──────────┐         │
                    │                    │          │   │ 活动模块  │         │
                    │                    │          │─▷ │(奖励/红利)│         │
                    │                    │          │   └──────────┘         │
                    │                    │          │                         │
                    │                    │          │◁──┌──────────┐         │
                    │                    │          │   │ 直播模块  │         │
                    │                    │          │   │(主播收入) │         │
                    │                    └──────────┘   └──────────┘         │
                    │                                                         │
                    │   ┌──────────────┐                                      │
                    │   │三方汇率API×3 │──HTTP──▷ ser-wallet (定时拉取)       │
                    │   └──────────────┘                                      │
                    └─────────────────────────────────────────────────────────┘
```

### 2.2 依赖方向汇总

```
单向依赖（wallet → 外部）：
  wallet → ser-kyc      ：仅提现流程调用，查KYC状态
  wallet → ser-user     ：仅提现流程调用，取KYC姓名
  wallet → ser-blog     ：B端写操作调用，审计日志（fire-and-forget）
  wallet → ser-s3       ：B端币种编辑调用，图标上传
  wallet → 三方汇率API  ：定时任务调用，拉取汇率

单向依赖（外部 → wallet）：
  投注模块 → wallet    ：调用 DebitWallet / CreditWallet
  直播模块 → wallet    ：调用 CreditWallet（主播钱包）

双向依赖（唯一一对）：
  wallet ⟷ 财务模块    ：
    正向: wallet → 财务 (GetPaymentConfig, CreatePayOrder, CreateWithdrawOrder)  2-3个
    反向: 财务 → wallet (CreditWallet, Freeze/Unfreeze/Confirm, Manual系列)     6-7个

双向依赖（较弱）：
  wallet ⟷ 活动模块    ：
    正向: wallet → 活动 (GetAvailableBonuses, GetBonusConfig)
    反向: 活动 → wallet (RewardCredit)
```

### 2.3 关键认知

```
① 财务模块是唯一强双向依赖 — 必须协同设计IDL合约
② 投注/直播/活动 调用 wallet 都是"单向使用"模式 — wallet暴露接口即可
③ ser-kyc/ser-user/ser-blog/ser-s3 都是已有服务 — IDL已定义，可直接接入
④ 财务模块/活动模块 — 服务名未定，IDL不可用，需Mock占位开发
```

---

## 第三部分：C端接口逐一链路分析（16条）

### 3.1 钱包首页与币种（2条链路）

#### C1: GetWalletHome — 钱包首页

```
链路类型: 纯内部链路（无跨模块调用）
复杂度: ★★☆☆☆

C端App
  │ POST /api/wallet/home
  ▼
gate-font → [Token提取userID] → RPC
  │
  ▼
ser-wallet.handler.GetWalletHome(ctx, req)
  │
  ▼
walletService.GetWalletHome(ctx, req)
  │
  ├──① currencyService.ValidateCurrency(currencyCode)     [内部调用-币种配置]
  │     └── rep.GetCurrencyConfig(currencyCode) → TiDB
  │         校验: status=enabled, 返回displayInfo
  │
  ├──② walletCore.GetBalance(uid, currencyCode, "center")  [内部调用-钱包核心]
  │     └── Redis缓存 → 命中返回 / 未命中 → rep.GetUserWallet → TiDB
  │
  ├──③ walletCore.GetBalance(uid, currencyCode, "reward")  [内部调用-钱包核心]
  │     └── 同上
  │
  ├──④ 计算: resourceBalance = center + reward
  │
  └──⑤ 格式化: 应用币种精度规则(VND/IDR整数, THB/BSB两位小数)
  │
  ▼
← 返回: {balance, currencyInfo, walletTiers}

关键点: 纯读操作, 走缓存, 无写入, 无锁, 最简单的链路之一
```

#### C2: GetCurrencyList — 币种列表

```
链路类型: 纯内部链路
复杂度: ★★☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
walletService.GetCurrencyList(ctx, req)
  │
  ├──① currencyService.ListEnabledCurrencies()
  │     └── rep → TiDB: SELECT * FROM currency_config WHERE status=1
  │
  ├──② walletCore.BatchGetBalance(uid, allCurrencyCodes)
  │     └── 批量查询每个币种的resourceBalance
  │
  └──③ 组装: 每个币种 + 对应余额
  │
  ▼
← 返回: [{currencyInfo, balance}, ...]

关键点: 批量查询性能 — N个币种=N次余额查询，考虑Redis pipeline优化
```

### 3.2 充值相关（4条链路）

#### C3: GetDepositMethods — 充值方式列表

```
链路类型: 跨模块链路（调用财务模块）
复杂度: ★★★☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
depositService.GetDepositMethods(ctx, req)
  │
  ├──① currencyService.ValidateCurrency(currencyCode)
  │     └── rep → TiDB: 校验币种启用
  │
  ├──②【跨模块RPC】rpc.FinanceClient().GetPaymentConfig(currencyCode, "deposit")
  │     └── ser-wallet → ETCD → 财务模块 → 返回: 充值方式列表+手续费+限额+排序
  │     └── ⚠️ 当前状态: 财务IDL未定义, 需Mock
  │     └── 超时: 3s, 失败降级: 返回空列表+错误提示
  │
  └──③ 过滤+排序: 按币种+用户状态过滤可用方式
  │
  ▼
← 返回: [{methodName, fee, minAmount, maxAmount, icon}, ...]

关键点: 这是第一个跨模块调用点，财务模块不可用时整个充值流程阻断
挑战: Mock策略 — 如何设计Mock让开发不阻塞？
```

#### C4: CreateDepositOrder — 创建充值订单

```
链路类型: 跨模块链路（调用财务模块 + 可能调用活动模块）
复杂度: ★★★★☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
depositService.CreateDepositOrder(ctx, req)
  │
  ├──① currencyService.ValidateCurrency(currencyCode)
  │     └── rep → TiDB
  │
  ├──② 金额校验: amount >= min && amount <= max && amount <= dailyLimit
  │     └── rep → TiDB: 查今日已充值总额
  │
  ├──③ (BSB充值) currencyService.GetExchangeRate(currencyCode)
  │     └── 锁定汇率快照，记录到订单
  │
  ├──④ (有红利) 【跨模块RPC?】活动模块.ValidateBonus(activityId, uid)
  │     └── ⚠️ 服务名未定, 需Mock
  │
  ├──⑤【跨模块RPC】rpc.FinanceClient().CreatePayOrder(orderInfo)
  │     └── ser-wallet → 财务模块 → 三方支付渠道
  │     └── 返回: 支付订单号 + 支付URL/地址
  │     └── ⚠️ 需Mock
  │
  ├──⑥ 本地建单: INSERT deposit_order(status=pending_payment)
  │     └── rep → TiDB (事务)
  │
  └──⑦ 返回支付信息给前端
  │
  ▼
← 返回: {orderNo, payUrl, payAddress, expireTime}

关键点:
  - 涉及2个跨模块调用（财务+活动），是充值链路中最复杂的节点
  - 汇率锁定时机：在此步骤锁定，后续CreditWallet回调时使用相同汇率
  - 本地订单状态: pending_payment → 等待三方回调
  - 此步骤不涉及钱包余额变动！余额变动在Stage6（CreditWallet回调）
```

#### C5: CheckDuplicateDeposit — 重复转账检测

```
链路类型: 纯内部链路
复杂度: ★☆☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
depositService.CheckDuplicateDeposit(ctx, req)
  │
  └──① rep → TiDB: 查询近N分钟内相同(uid, amount, method)的订单
  │
  ▼
← 返回: {isDuplicate: bool, existingOrderNo: string}

关键点: 纯DB查询, 无跨模块调用, 前置于CreateDepositOrder
```

#### C6: GetDepositStatus — 充值订单状态查询

```
链路类型: 纯内部链路 / 可能跨模块
复杂度: ★★☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
depositService.GetDepositStatus(ctx, req)
  │
  ├──① rep → TiDB: 查本地deposit_order状态
  │
  └──② (可选)【跨模块RPC】财务模块.QueryOrderStatus(channelOrderNo)
  │     └── 如果本地状态仍为pending, 主动查财务确认
  │
  ▼
← 返回: {status, amount, createTime, updateTime}

关键点: Should-have接口, 主要用于前端轮询支付结果
```

### 3.3 兑换相关（2条链路）

#### C7: GetExchangeRate — 汇率查询/预览

```
链路类型: 纯内部链路（最独立的接口之一）
复杂度: ★★☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
exchangeService.GetExchangeRate(ctx, req)
  │
  ├──① currencyService.ValidateCurrency(currencyCode)
  │     └── 校验: 必须是法币(VND/IDR/THB), BSB不支持兑换
  │
  ├──② currencyService.GetPlatformRate(currencyCode)
  │     └── rep → TiDB/Redis: 获取平台汇率 (1 BSB = X 法币)
  │
  ├──③ 获取红利比例(来源待定)
  │
  └──④ 计算预览:
  │     exchangeBsb = inputAmount / rate
  │     bonusBsb = exchangeBsb × bonusRatio
  │     totalBsb = exchangeBsb + bonusBsb
  │
  ▼
← 返回: {rate, bonusRatio, previewExchange, previewBonus, previewTotal}

关键点: 不依赖财务模块! 纯钱包内部+币种配置, 可最早完成端到端测试
```

#### C8: DoExchange — 执行兑换

```
链路类型: 纯内部链路（跨币种原子操作）
复杂度: ★★★★☆ (内部复杂度高，但无外部依赖)

C端App → gate-font → RPC → ser-wallet
  │
  ▼
exchangeService.DoExchange(ctx, req)
  │
  ├──① currencyService.ValidateCurrency(currencyCode)
  │     └── 校验法币状态
  │
  ├──② currencyService.LockPlatformRate(currencyCode)
  │     └── 锁定当前平台汇率(防止计算过程中汇率变动)
  │
  ├──③ walletCore.GetBalance(uid, currencyCode, "center")
  │     └── 校验: 法币中心钱包余额 >= amount
  │
  ├──④ 计算:
  │     exchangeBsb = amount / lockedRate
  │     bonusBsb = exchangeBsb × bonusRatio
  │
  ├──⑤ 【单事务原子操作 — 全部成功或全部回滚】
  │     ├── 5a. 法币中心钱包 -= amount
  │     │     └── SQL: UPDATE user_wallet SET available = available - ? WHERE uid=? AND currency=? AND wallet_type=1 AND available >= ?
  │     ├── 5b. BSB中心钱包 += exchangeBsb
  │     │     └── SQL: UPDATE user_wallet SET available = available + ? WHERE uid=? AND currency='BSB' AND wallet_type=1
  │     ├── 5c. BSB奖励钱包 += bonusBsb (含流水倍率要求)
  │     │     └── SQL: UPDATE user_wallet SET available = available + ? WHERE uid=? AND currency='BSB' AND wallet_type=2
  │     ├── 5d. 写法币扣款流水 (wallet_flow)
  │     ├── 5e. 写BSB入账流水 (wallet_flow)
  │     ├── 5f. 写BSB奖励流水 (wallet_flow, 含multiplier)
  │     └── 5g. 写兑换记录 (exchange_order, 含汇率快照)
  │     └── rep → TiDB: BEGIN → 7条SQL → COMMIT
  │
  └──⑥ 失效缓存: DEL wallet:{uid}:{法币} + DEL wallet:{uid}:BSB
  │
  ▼
← 返回: {success, exchangedBsb, bonusBsb, totalBsb, rateUsed}

关键点:
  - 单事务内7条SQL, 跨2个币种3个钱包, 是原子性要求最高的操作
  - 无任何外部RPC调用 — 全程内部闭环
  - 但如果事务中任何一步失败, 全部回滚 — 对事务可靠性要求极高
  - 这是开发阶段最适合先跑通的端到端链路（不依赖任何外部模块）
```

### 3.4 提现相关（4条链路）

#### C9: GetWithdrawMethods — 提现方式列表

```
链路类型: 多模块交叉链路（调用财务+KYC+本地数据）
复杂度: ★★★★☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
withdrawService.GetWithdrawMethods(ctx, req)
  │
  ├──① currencyService.ValidateCurrency(currencyCode)
  │
  ├──②【跨模块RPC】rpc.KycClient().KycQuery(uid)
  │     └── ser-wallet → ETCD → ser-kyc → 返回: KYC认证状态
  │     └── 路由:
  │         未认证 → 仅显示USDT提现
  │         已认证 → 显示所有方式
  │
  ├──③【跨模块RPC】rpc.FinanceClient().GetWithdrawConfig(currencyCode)
  │     └── ser-wallet → ETCD → 财务模块 → 返回: 提现方式+手续费+限额
  │     └── ⚠️ 需Mock
  │
  ├──④ 查本地充值历史: rep → TiDB
  │     └── 规则: "充什么提什么" — 查用户历史充值方式
  │     └── 法币充值 → 法币提现; 混合 → 法币优先
  │
  └──⑤ 三重过滤取交集:
  │     available = KYC过滤 ∩ 充值历史过滤 ∩ 财务配置过滤
  │     这是提现流程中最复杂的过滤逻辑
  │
  ▼
← 返回: [{method, fee, min, max, kycStatus}, ...]

关键点:
  - 3个数据源交叉过滤: KYC状态 × 充值历史 × 财务配置
  - 2个跨模块RPC调用（KYC + 财务）
  - 任何一个数据源缺失都会影响过滤结果的正确性
  - 这是整个C端中过滤逻辑最复杂的接口
```

#### C10: GetWithdrawAccount — 获取提现账户

```
链路类型: 纯内部链路
复杂度: ★☆☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
withdrawService.GetWithdrawAccount(ctx, req)
  │
  └── rep → TiDB: SELECT * FROM withdraw_account WHERE uid=? AND method=?
  │
  ▼
← 返回: {accountInfo} 或 空(首次提现)

关键点: 纯DB读操作, 最简单的链路之一
```

#### C11: SaveWithdrawAccount — 保存提现账户

```
链路类型: 跨模块链路（调用ser-user）
复杂度: ★★★☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
withdrawService.SaveWithdrawAccount(ctx, req)
  │
  ├──①【跨模块RPC】rpc.UserClient().GetUserInfoExpend(uid)
  │     └── 获取KYC实名信息 → 姓名
  │
  ├──② 校验: 账户持有人姓名 == KYC认证姓名
  │     └── 不一致 → 拒绝保存, 返回错误
  │
  └──③ rep → TiDB: UPSERT withdraw_account (uid + method 唯一键)
  │
  ▼
← 返回: {success}

关键点: 后端必须强制校验姓名一致性, 不能仅依赖前端校验
```

#### C12: CreateWithdrawOrder — 创建提现订单

```
链路类型: 最复杂的C端链路（7个前置条件, 3个跨模块调用, 涉及冻结操作）
复杂度: ★★★★★

C端App → gate-font → RPC → ser-wallet
  │
  ▼
withdrawService.CreateWithdrawOrder(ctx, req)
  │
  ├──①【跨模块RPC】rpc.KycClient().KycQuery(uid)
  │     └── 防御性重检: 用户可能在填写过程中KYC状态变化
  │
  ├──② 限额重校验:
  │     ├── 单次限额: amount >= min && amount <= max
  │     ├── 日限额: today_withdrawn + amount <= daily_max
  │     └── rep → TiDB: SUM(amount) FROM withdraw_order WHERE uid=? AND date=today AND status IN (pending,processing,success)
  │
  ├──③ (BSB提现) currencyService.GetOutputRate(currencyCode)
  │     └── 锁定出金汇率快照
  │
  ├──④ 余额校验: walletCore.GetBalance(uid, currencyCode, "center")
  │     └── center_available >= amount
  │
  ├──⑤【核心步骤 — 冻结余额】walletCore.FreezeBalance(uid, currencyCode, amount, orderNo)
  │     └── 原子操作: available -= amount, frozen += amount
  │     └── 写freeze记录(status=freezing) + 写wallet_flow
  │     └── ⚠️ 此步骤后用户可用余额已减少, 如果后续步骤失败必须解冻
  │
  ├──⑥ 本地建单: INSERT withdraw_order(status=pending_review)
  │     └── rep → TiDB
  │
  ├──⑦【跨模块RPC】rpc.FinanceClient().CreateWithdrawOrder(orderInfo)
  │     └── ser-wallet → 财务模块 → 返回: 财务订单号
  │     └── ⚠️ 需Mock
  │     └── ⚠️ 如果这步失败:
  │         → 必须调用 UnfreezeBalance 回滚冻结
  │         → 更新本地订单状态为 "creation_failed"
  │
  └──⑧ 更新本地订单: 关联财务订单号
  │
  ▼
← 返回: {orderNo, status: "pending_review"}

关键点:
  - 7个前置条件, 任何一个失败都阻止创建
  - 冻结在财务调用之前 — 先锁资金再通知财务
  - 财务调用失败必须回滚冻结 — 这是最关键的异常处理链路
  - 整个C端中前置依赖最多、异常分支最复杂的接口

异常分支:
  ┌─ KYC检查失败 → 直接返回错误, 无需回滚
  ├─ 限额校验失败 → 直接返回错误, 无需回滚
  ├─ 余额不足 → 直接返回错误, 无需回滚
  ├─ 冻结失败(并发竞争) → 直接返回错误, 无需回滚
  ├─ 本地建单失败 → 需回滚冻结(UnfreezeBalance)
  └─ 财务模块调用失败 → 需回滚冻结(UnfreezeBalance) + 更新本地订单状态
```

### 3.5 记录查询（2条链路）

#### C13: GetRecordList — 交易记录列表

```
链路类型: 纯内部链路
复杂度: ★★☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
recordService.GetRecordList(ctx, req)
  │
  ├── 按tab分类查询:
  │   ├── 充值记录 → rep → TiDB: deposit_order (或查财务模块, 待确认)
  │   ├── 提现记录 → rep → TiDB: withdraw_order
  │   ├── 兑换记录 → rep → TiDB: exchange_order
  │   └── 奖励记录 → rep → TiDB: wallet_flow WHERE type IN (reward...)
  │
  └── 分页 + 时间范围 + 状态筛选
  │
  ▼
← 返回: [{recordSummary}, ...], pagination

关键点: 数据源确认 — 充值/提现记录从本地查还是从财务模块查, 需跨组确认
```

#### C14: GetRecordDetail — 交易记录详情

```
链路类型: 纯内部链路 / 可能跨模块
复杂度: ★★☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
recordService.GetRecordDetail(ctx, req)
  │
  ├── 按recordType路由:
  │   ├── 充值详情 → deposit_order + wallet_flow
  │   ├── 提现详情 → withdraw_order + wallet_freeze + wallet_flow
  │   ├── 兑换详情 → exchange_order + wallet_flow
  │   └── 其他详情 → wallet_flow
  │
  └── 组装: 基础信息 + 状态时间线 + 关联流水
  │
  ▼
← 返回: {detailInfo, statusTimeline, relatedFlows}

关键点: 提现详情最复杂, 需要关联3张表(订单+冻结+流水)
```

### 3.6 补充C端链路（2条, Should-Have）

#### C15: GetBonusList — 红利活动列表

```
链路类型: 跨模块链路
复杂度: ★★☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
depositService.GetBonusList(ctx, req)
  │
  └──【跨模块RPC】活动模块.GetAvailableBonuses(uid, currencyCode)
  │     └── ⚠️ 服务名未定, 需Mock
  │
  ▼
← 返回: [{bonusName, ratio, minDeposit, maxBonus}, ...]
```

#### C16: GetWithdrawStatus — 提现订单状态

```
链路类型: 纯内部链路
复杂度: ★☆☆☆☆

C端App → gate-font → RPC → ser-wallet
  │
  ▼
withdrawService.GetWithdrawStatus(ctx, req)
  │
  └── rep → TiDB: SELECT status FROM withdraw_order WHERE orderNo=?
  │
  ▼
← 返回: {status, updateTime}
```

---

## 第四部分：B端接口逐一链路分析（9条）

### 4.1 币种配置管理（5条链路）

#### B1: PageCurrency — 币种列表分页

```
链路类型: 纯内部链路
复杂度: ★☆☆☆☆

Admin → gate-back → [RBAC] → RPC → ser-wallet
  │
  ▼
currencyService.PageCurrency(ctx, req)
  │
  └── rep → TiDB: SELECT * FROM currency_config + 分页 + 筛选
  │
  ▼
← 返回: {list, total, page, pageSize}
```

#### B2: EditCurrency — 编辑币种

```
链路类型: 跨模块链路（审计日志 + 可能图标上传）
复杂度: ★★★☆☆

Admin → gate-back → [RBAC] → RPC → ser-wallet
  │
  ▼
currencyService.EditCurrency(ctx, req)
  │
  ├──① (有图标) 【跨模块RPC】rpc.S3Client().GetPresignUrl(req)
  │     └── 获取预签名URL用于图标上传
  │
  ├──② 参数校验: 浮动比例/偏差阈值/精度等
  │
  ├──③ rep → TiDB: UPDATE currency_config SET ...
  │
  ├──④ 失效缓存: DEL currency:{code}
  │
  └──⑤【跨模块RPC】rpc.BLogClient().AddActLog(ctx, logReq) [oneway, fire-and-forget]
  │     └── 审计日志: 谁改了什么, 改前值/改后值
  │
  ▼
← 返回: {success}

关键点: oneway审计日志不阻塞主流程, 图标上传是可选步骤
```

#### B3: StatusCurrency — 启停币种

```
链路类型: 跨模块链路（审计日志）
复杂度: ★★★☆☆ (操作简单但有级联影响)

Admin → gate-back → [RBAC] → RPC → ser-wallet
  │
  ▼
currencyService.StatusCurrency(ctx, req)
  │
  ├──① (禁用操作) 检查: 是否有用户持有该币种余额 → 告警但不阻止
  │
  ├──② rep → TiDB: UPDATE currency_config SET status=? WHERE code=?
  │
  ├──③ 失效缓存: DEL currency:{code} + DEL currency:list
  │
  └──④【跨模块RPC】rpc.BLogClient().AddActLog [oneway]
  │
  ▼
← 返回: {success}

关键点: 禁用币种的级联影响 — 该币种的充值/提现/兑换入口都应不可见
边界问题: 禁用时有进行中的充值/提现订单怎么处理? (待和财务确认)
```

#### B4: SetBaseCurrency — 设置基础币种

```
链路类型: 纯内部 + 审计
复杂度: ★★☆☆☆ (但是一次性关键操作)

Admin → gate-back → [RBAC] → RPC → ser-wallet
  │
  ▼
currencyService.SetBaseCurrency(ctx, req)
  │
  ├──① 校验: 当前是否已有基础币种 → 已设置则拒绝(一次性操作)
  │
  ├──② rep → TiDB: UPDATE currency_config SET is_base=1 WHERE code='USDT'
  │
  └──③【跨模块RPC】rpc.BLogClient().AddActLog [oneway]
  │
  ▼
← 返回: {success}

关键点: 绝对前置条件, 必须最先执行, 且只能执行一次
```

#### B5: PageExchangeRateLog — 汇率变更日志

```
链路类型: 纯内部链路
复杂度: ★☆☆☆☆

Admin → gate-back → [RBAC] → RPC → ser-wallet
  │
  ▼
currencyService.PageExchangeRateLog(ctx, req)
  │
  └── rep → TiDB: SELECT * FROM exchange_rate_log + 分页 + 时间范围
  │
  ▼
← 返回: {list, total}
```

### 4.2 用户钱包管理（2条链路）

#### B6: PageUserWallet — 用户钱包列表

```
链路类型: 跨模块链路（需查用户信息）
复杂度: ★★☆☆☆

Admin → gate-back → [RBAC] → RPC → ser-wallet
  │
  ▼
walletService.PageUserWallet(ctx, req)
  │
  ├──① rep → TiDB: SELECT * FROM user_wallet + 分页 + 条件筛选
  │
  └──②【跨模块RPC】rpc.UserClient().BatchGetUserInfo(uids)
  │     └── 批量查用户昵称, 用于列表展示
  │
  ▼
← 返回: {list: [{uid, nickname, currency, balance, ...}], total}
```

#### B7: DetailUserWallet — 用户钱包详情

```
链路类型: 纯内部链路
复杂度: ★★☆☆☆

Admin → gate-back → [RBAC] → RPC → ser-wallet
  │
  ▼
walletService.DetailUserWallet(ctx, req)
  │
  ├──① rep → TiDB: 查该用户所有币种×钱包类型的余额
  │
  └──② rep → TiDB: 查最近N条流水记录
  │
  ▼
← 返回: {wallets: [...], recentFlows: [...]}
```

### 4.3 补充B端链路（2条, Should-Have）

#### B8: ExportRateLog — 汇率日志导出

```
复杂度: ★★☆☆☆
链路同B5, 增加CSV/Excel生成步骤
```

#### B9: DetailCurrency — 币种详情

```
复杂度: ★☆☆☆☆
链路: rep → TiDB: SELECT * FROM currency_config WHERE code=?
```

---

## 第五部分：RPC暴露接口逐一链路分析（13条）

> 注: RPC接口由外部模块直接调用, 不经过gate层, 链路从RPC入口开始。

### 5.1 余额变动类（5条核心链路）

#### R1: CreditWallet — 充值成功入账

```
调用方: 财务模块（三方支付回调确认后）
链路类型: 带幂等的写入链路
复杂度: ★★★☆☆

财务模块
  │ rpc.WalletClient().CreditWallet(ctx, req)
  ▼
ser-wallet.handler.CreditWallet
  │
  ▼
walletCore.CreditWallet(ctx, req)
  │
  ├──① 幂等检查: rep → TiDB: SELECT FROM wallet_flow WHERE order_no=?
  │     └── 已存在 → 返回成功(幂等, 不重复入账)
  │
  ├──② currencyService.ValidateCurrency(currencyCode)
  │     └── 校验币种存在且启用
  │
  ├──③ 自动建钱包: rep → TiDB: 该用户该币种无钱包 → INSERT user_wallet
  │     └── 首次充值该币种时触发
  │
  ├──④ creditCore 原子操作:
  │     └── SQL: UPDATE user_wallet SET available = available + ? WHERE uid=? AND currency=? AND wallet_type=1
  │     └── 写wallet_flow: type=deposit_credit, order_no=orderNo
  │     └── 单事务: balance变动 + flow写入
  │
  └──⑤ 失效缓存: DEL wallet:{uid}:{currency}
  │
  ▼
← 返回: {success, idempotent: bool, newBalance}

关键点:
  - 财务可能因超时重试 → 幂等设计是生命线
  - 自动建钱包: 用户可能从未操作过该币种
  - 这是充值链路的终点 — 资金真正进入用户钱包的时刻
```

#### R2: DebitWallet — 投注/消费扣款

```
调用方: 投注模块 / 财务模块
链路类型: 双钱包扣款链路
复杂度: ★★★★☆

投注模块
  │ rpc.WalletClient().DebitWallet(ctx, req)
  ▼
ser-wallet.handler.DebitWallet
  │
  ▼
walletCore.DebitWallet(ctx, req)
  │
  ├──① 幂等检查: order_no
  │
  ├──② 查询双钱包余额:
  │     centerBalance = rep.GetBalance(uid, currency, center)
  │     rewardBalance = rep.GetBalance(uid, currency, reward)
  │
  ├──③ 余额校验: centerBalance + rewardBalance >= amount
  │     └── 不足 → 返回余额不足错误
  │
  ├──④ 计算分配:
  │     centerDebit = min(centerBalance, amount)
  │     rewardDebit = amount - centerDebit
  │
  ├──⑤ debitCore 原子操作(单事务):
  │     ├── UPDATE user_wallet SET available -= centerDebit WHERE ... AND available >= centerDebit
  │     ├── (如需) UPDATE user_wallet SET available -= rewardDebit WHERE ... AND available >= rewardDebit
  │     ├── 写center_flow: debit
  │     └── (如需) 写reward_flow: debit
  │
  └──⑥ 失效缓存
  │
  ▼
← 返回: {success, centerDebit, rewardDebit}

关键点:
  - 必须返回center/reward扣款分配 — 投注模块需要这个信息做返奖计算
  - 中心钱包优先扣 — 奖励钱包补充不足部分
  - SQL乐观锁 WHERE available >= ? — 防止并发超扣
  - 双钱包扣款在单事务内 — 原子性保证
```

#### R3: FreezeBalance — 提现冻结

```
调用方: ser-wallet内部(CreateWithdrawOrder) / 财务模块(边缘场景)
链路类型: 状态机起始链路
复杂度: ★★★☆☆

ser-wallet 或 财务模块
  │
  ▼
walletCore.FreezeBalance(ctx, req)
  │
  ├──① 幂等检查: order_no
  │
  ├──② 余额校验: center_available >= amount
  │
  ├──③ freezeCore 原子操作(单事务):
  │     ├── UPDATE user_wallet SET available -= amount, frozen += amount
  │     ├── INSERT wallet_freeze (order_no, status=1:freezing)
  │     └── 写wallet_flow: type=withdraw_freeze
  │
  └──④ 失效缓存
  │
  ▼
← 返回: {success, freezeFlowId}

关键点: 这是状态机的起点 → 后续必须走 R4(Unfreeze) 或 R5(ConfirmDebit)
```

#### R4: UnfreezeBalance — 提现拒绝/解冻

```
调用方: 财务模块（风控拒绝/财务拒绝/打款失败）
链路类型: 状态机分支A — 资金回归
复杂度: ★★★☆☆

财务模块
  │
  ▼
walletCore.UnfreezeBalance(ctx, req)
  │
  ├──① 查冻结记录: rep → TiDB: SELECT FROM wallet_freeze WHERE order_no=?
  │     └── 校验: status必须=1(freezing), 否则拒绝
  │
  ├──② unfreezeCore 原子操作(单事务):
  │     ├── UPDATE user_wallet SET frozen -= amount, available += amount
  │     ├── UPDATE wallet_freeze SET status=3(unfrozen), finish_at=now
  │     └── 写wallet_flow: type=withdraw_return
  │
  └──③ 失效缓存
  │
  ▼
← 返回: {success}

关键点: 与R5互斥 — 同一orderNo不可能既解冻又确认扣除
```

#### R5: ConfirmDebit — 打款成功确认

```
调用方: 财务模块（三方打款成功后）
链路类型: 状态机分支B — 资金永久离开
复杂度: ★★★☆☆

财务模块
  │
  ▼
walletCore.ConfirmDebit(ctx, req)
  │
  ├──① 查冻结记录: 同R4, 校验status=1(freezing)
  │
  ├──② confirmDebitCore 原子操作(单事务):
  │     ├── UPDATE user_wallet SET frozen -= amount (永久扣除, 资金离开钱包)
  │     ├── UPDATE wallet_freeze SET status=2(confirmed_debit), finish_at=now
  │     └── 写wallet_flow: type=withdraw_debit
  │
  └──③ 失效缓存
  │
  ▼
← 返回: {success}

状态机全貌:
  FreezeBalance(R3)        → status=1(freezing)     [资金从available移到frozen]
       ├── ConfirmDebit(R5) → status=2(confirmed)    [frozen减少, 资金永久离开]
       └── UnfreezeBalance(R4) → status=3(unfrozen)  [frozen减少, 回到available]

  2和3互斥, 终态不可逆转
```

### 5.2 手动修正类（3条核心链路）

#### R6: ManualCredit — 手动加款

```
调用方: 财务模块（B端审批通过后）
复杂度: ★★☆☆☆

财务模块 → ser-wallet.ManualCredit
  │
  ├── 幂等检查(A+前缀orderNo)
  ├── 自动建钱包(如不存在)
  ├── creditCore: specified_walletType.balance += amount
  ├── 写wallet_flow: type=manual_credit, 记录operator+reason+remark
  └── 失效缓存

← 返回: {success}

关键点: walletType参数 — 可指定中心/奖励/主播/代理任意钱包
```

#### R7: ManualDebit — 手动扣款

```
调用方: 财务模块（B端审批通过后）
复杂度: ★★★☆☆

财务模块 → ser-wallet.ManualDebit
  │
  ├── 幂等检查(M+前缀orderNo)
  ├── 查所有钱包余额
  ├── 级联扣款: 从余额最高的钱包开始扣, 直到扣满
  │   例: debit=1000, center=800, reward=150, anchor=100
  │   → center扣800 + reward扣150 + anchor扣50 = 1000
  │   → 如果总余额仅900, 返回actual=900(部分扣除)
  ├── 事务: 更新每个受影响钱包 + 写流水
  └── 失效缓存

← 返回: {success, actualDebit}

关键点: 实际扣除金额可能 < 请求金额, 差额记录审计
```

#### R8: SupplementCredit — 充值补单

```
调用方: 财务模块（B端审批充值补单后）
复杂度: ★★☆☆☆

财务模块 → ser-wallet.SupplementCredit
  │
  ├── 幂等检查(S+前缀orderNo)
  ├── center_wallet += amount (主金额)
  ├── (如有配置) reward_wallet += bonusAmount (补充奖励)
  ├── 写wallet_flow(关联原订单号)
  └── 失效缓存

← 返回: {success}

关键点: 可同时往中心+奖励两个钱包入账
```

### 5.3 查询类（2条链路）

#### R9: GetBalance — 余额查询

```
调用方: 财务/投注/直播/活动 等所有模块
复杂度: ★☆☆☆☆ (但频率最高)

任何模块 → ser-wallet.GetBalance
  │
  ├── Redis缓存查询: GET wallet:bal:{uid}:{currency}
  │   ├── 命中 → 直接返回
  │   └── 未命中 → rep → TiDB → 写回Redis(TTL=5min)
  │
  ▼
← 返回: {available, frozen, total}

关键点:
  - 全平台调用频率最高的接口(投注前必查)
  - 必须走缓存, 否则DB会成为瓶颈
  - 所有余额变动操作都会DEL缓存 → 下次查询触发重建
```

#### R10: BatchGetBalance — 批量余额查询

```
调用方: 财务/投注（多用户场景）
复杂度: ★★☆☆☆

任何模块 → ser-wallet.BatchGetBalance
  │
  ├── 循环调用 GetBalance (走缓存) 或 Redis Pipeline批量查
  │
  ▼
← 返回: [{uid, available, frozen, total}, ...]

关键点: Should-Have, 减少批量场景的RPC调用次数
```

### 5.4 奖励类（1条链路）

#### R11: RewardCredit — 活动奖励入账

```
调用方: 活动/任务模块
复杂度: ★★☆☆☆

活动模块 → ser-wallet.RewardCredit
  │
  ├── 幂等检查
  ├── 自动建reward_wallet(如不存在)
  ├── reward_wallet.balance += amount
  ├── 写wallet_flow: type=activity_reward, 含flowMultiplier要求
  └── 失效缓存

← 返回: {success}

关键点: 永远进奖励钱包(非中心), 含流水倍率要求(需投注X倍才可提现)
```

### 5.5 币种数据类（2条链路, Should-Have）

#### R12: GetCurrencyConfig — 币种配置查询

```
调用方: 财务/其他模块
复杂度: ★☆☆☆☆

任何模块 → ser-wallet.GetCurrencyConfig
  │
  └── rep/cache → 返回币种基础配置信息

← 返回: {currencyCode, name, type, symbol, status, ...}
```

#### R13: GetExchangeRateRpc — 汇率查询

```
调用方: 财务/其他模块
复杂度: ★☆☆☆☆

任何模块 → ser-wallet.GetExchangeRateRpc
  │
  └── rep/cache → 返回当前平台汇率

← 返回: {currencyCode, platformRate, depositRate, outputRate}
```

---

## 第六部分：三大业务主链路时序图

### 6.1 充值完整链路（最长链路 — 6阶段跨3方）

```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  C端App  │    │ ser-wallet│    │  财务模块  │    │ 三方支付  │    │ 活动模块  │
└────┬─────┘    └─────┬────┘    └─────┬────┘    └─────┬────┘    └────┬─────┘
     │                │               │               │              │
     │ ──Stage 1: 选币种──            │               │              │
     │ GetWalletHome  │               │               │              │
     │───────────────▶│               │               │              │
     │◀──余额+币种───│               │               │              │
     │                │               │               │              │
     │ ──Stage 2: 选充值方式──        │               │              │
     │ GetDepositMethods              │               │              │
     │───────────────▶│               │               │              │
     │                │──GetPaymentConfig──▶│         │              │
     │                │◀──充值方式列表──────│         │              │
     │◀──方式列表────│               │               │              │
     │                │               │               │              │
     │ (可选)GetBonusList             │               │              │
     │───────────────▶│               │               │              │
     │                │────────────────────────────────────GetBonuses─▶│
     │                │◀──────────────────────────────────红利列表────│
     │◀──红利列表────│               │               │              │
     │                │               │               │              │
     │ ──Stage 3: 输入金额──          │               │              │
     │ (前端本地校验)  │               │               │              │
     │                │               │               │              │
     │ ──Stage 4: 确认充值──          │               │              │
     │ CreateDepositOrder              │               │              │
     │───────────────▶│               │               │              │
     │                │──CreatePayOrder──▶│            │              │
     │                │                │──创建渠道单──▶│              │
     │                │◀──支付单号+URL──│◀────────────│              │
     │                │ INSERT deposit_order            │              │
     │◀──支付URL─────│               │               │              │
     │                │               │               │              │
     │ ──Stage 5: 用户支付(脱离平台)── │               │              │
     │────────────────────────────────────用户支付──▶  │              │
     │                │               │  ◀──支付结果──│              │
     │                │               │               │              │
     │ ──Stage 6: 回调入账──          │               │              │
     │                │◀──CreditWallet─│              │              │
     │                │  creditCore     │               │              │
     │                │  center += amount               │              │
     │                │  写wallet_flow  │               │              │
     │                │──success──────▶│               │              │
     │                │               │               │              │

时间线: Stage1-4 秒级(即时) │ Stage5 分钟级(用户操作) │ Stage6 秒级(异步回调)

关键认知:
  ① ser-wallet在Stage5中完全不参与 — 用户在三方平台操作
  ② 资金真正入账在Stage6的CreditWallet回调 — 这是充值链路的终点
  ③ 财务模块是中间枢纽 — 连接wallet和三方支付
  ④ 整条链路中wallet写操作只有2处: Stage4建订单 + Stage6入账
```

### 6.2 提现完整链路（最多前置条件 — 8阶段跨4方）

```
┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  C端App  │   │ ser-wallet│   │  财务模块  │   │  ser-kyc  │   │ ser-user │
└────┬─────┘   └─────┬────┘   └─────┬────┘   └─────┬────┘   └────┬─────┘
     │               │              │              │              │
     │ ──Stage 1: KYC检查──         │              │              │
     │───────────────▶│              │              │              │
     │               │──KycQuery──────────────────▶│              │
     │               │◀──KYC状态─────────────────│              │
     │◀──KYC路由────│              │              │              │
     │               │              │              │              │
     │ ──Stage 2: 获取提现方式──    │              │              │
     │ GetWithdrawMethods            │              │              │
     │───────────────▶│              │              │              │
     │               │──GetWithdrawConfig──▶│      │              │
     │               │◀──提现方式列表────────│      │              │
     │               │  查充值历史(本地)      │      │              │
     │               │  三重过滤取交集        │      │              │
     │◀──方式列表────│              │              │              │
     │               │              │              │              │
     │ ──Stage 3: 提现账户──        │              │              │
     │ SaveWithdrawAccount           │              │              │
     │───────────────▶│              │              │              │
     │               │──GetUserInfoExpend──────────────────────▶│
     │               │◀──KYC姓名───────────────────────────────│
     │               │  校验: 持卡人==KYC姓名  │              │
     │               │  UPSERT withdraw_account │              │
     │◀──保存成功────│              │              │              │
     │               │              │              │              │
     │ ──Stage 4: 输入金额+校验──   │              │              │
     │               │ 单次限额+日限额+余额校验    │              │
     │               │              │              │              │
     │ ──Stage 5: 确认提现(最关键)── │              │              │
     │ CreateWithdrawOrder           │              │              │
     │───────────────▶│              │              │              │
     │               │──KycQuery(防御性重检)─────▶│              │
     │               │◀──确认KYC──────────────────│              │
     │               │  限额重校验(防并发)         │              │
     │               │  余额重校验                 │              │
     │               │                             │              │
     │               │【⚡冻结余额⚡】             │              │
     │               │  FreezeBalance               │              │
     │               │  available -= amount          │              │
     │               │  frozen += amount             │              │
     │               │  INSERT wallet_freeze         │              │
     │               │                             │              │
     │               │  INSERT withdraw_order       │              │
     │               │──CreateWithdrawOrder──▶│     │              │
     │               │◀──财务订单号───────────│     │              │
     │               │  UPDATE withdraw_order(link)│              │
     │◀──提现中──────│              │              │              │
     │               │              │              │              │
     │ ──Stage 6-7: 审核+打款(B端/财务异步处理)──  │              │
     │               │              │              │              │
     │  (审核拒绝)    │◀──UnfreezeBalance──│       │              │
     │               │  frozen -= amount    │       │              │
     │               │  available += amount │       │              │
     │               │              │              │              │
     │  (打款成功)    │◀──ConfirmDebit──────│       │              │
     │               │  frozen -= amount    │       │              │
     │               │  (资金永久离开钱包)   │       │              │
     │               │              │              │              │
     │ ──Stage 8: 最终状态──        │              │              │
     │ GetWithdrawStatus             │              │              │
     │───────────────▶│              │              │              │
     │◀──最终状态────│              │              │              │

时间线: Stage1-5 秒级 │ Stage6-7 分钟至天级(人工审核+银行处理) │ Stage8 秒级

关键认知:
  ① Stage5是最关键节点 — 冻结在财务调用之前(先锁钱再通知)
  ② 如果Stage5中财务调用失败 → 必须回滚冻结(UnfreezeBalance)
  ③ Stage6-7是异步的 — wallet只被动接收Unfreeze或ConfirmDebit回调
  ④ 提现是唯一用到冻结状态机的业务流程
  ⑤ CreateWithdrawOrder有7个前置条件 — 是C端最复杂的接口
```

### 6.3 兑换完整链路（最独立 — 3阶段纯内部）

```
┌─────────┐    ┌──────────┐    ┌──────────┐
│  C端App  │    │ ser-wallet│    │   TiDB   │
└────┬─────┘    └─────┬────┘    └─────┬────┘
     │                │               │
     │ ──Stage 1: 查汇率──           │
     │ GetExchangeRate │               │
     │───────────────▶│               │
     │                │──查币种配置──▶│
     │                │◀──汇率+红利──│
     │                │  计算预览      │
     │◀──预览信息────│               │
     │                │               │
     │ ──Stage 2: 输入金额──         │
     │ (前端本地计算)  │               │
     │                │               │
     │ ──Stage 3: 执行兑换──         │
     │ DoExchange     │               │
     │───────────────▶│               │
     │                │  锁定平台汇率  │
     │                │  校验法币余额  │
     │                │  计算BSB金额   │
     │                │               │
     │                │──BEGIN────────▶│
     │                │  法币中心 -= X  │
     │                │  BSB中心 += Y  │
     │                │  BSB奖励 += Z  │
     │                │  写3条流水      │
     │                │  写兑换记录     │
     │                │──COMMIT───────▶│
     │                │◀──成功─────────│
     │                │               │
     │                │  DEL 法币缓存  │
     │                │  DEL BSB缓存   │
     │◀──兑换结果────│               │

时间线: 全程秒级 (<3秒完成)

关键认知:
  ① 零外部依赖 — 不调用任何其他服务
  ② 单事务7条SQL — 跨2个币种3个钱包, 原子性要求最高
  ③ 最适合作为第一个端到端联调的业务流程
  ④ 唯一的风险点: 事务内SQL较多, 需关注TiDB事务性能
```

---

## 第七部分：内部子模块调用拓扑

### 7.1 四层架构依赖图

```
┌──────────────────────────────────────────────────────────────────┐
│                    handler.go (30方法路由分发)                     │
│  每个IDL方法 → 对应Service方法                                    │
│  例: GetWalletHome → walletService.GetWalletHome                 │
│      PageCurrency → currencyService.PageCurrency                 │
│      CreditWallet → walletCore.CreditWallet                     │
└────────────┬───┬───┬───┬───┬───┬─────────────────────────────────┘
             │   │   │   │   │   │
    ┌────────▼─┐ │   │   │   │   │
    │currency  │ │   │   │   │   │
    │Service   │ │   │   │   │   │
    │(独立)    │ │   │   │   │   │
    └──┬───────┘ │   │   │   │   │
       │  供给    │   │   │   │   │
       │  币种/汇率│   │   │   │   │
       ▼         ▼   │   │   │   │
    ┌──────────────┐ │   │   │   │
    │ walletCore   │ │   │   │   │
    │ (6原子操作)   │ │   │   │   │
    │ credit/debit │ │   │   │   │
    │ freeze/unfreeze│   │   │   │
    │ confirmDebit │ │   │   │   │
    │ writeFlow    │ │   │   │   │
    └──┬───────────┘ │   │   │   │
       │  供给        │   │   │   │
       │  原子操作     │   │   │   │
       ▼              ▼   ▼   ▼   │
    ┌──────────────────────────┐   │
    │   交易业务层              │   │
    │ ┌──────────┐            │   │
    │ │deposit   │ 依赖:      │   │
    │ │Service   │ currency + │   │
    │ │          │ walletCore+│   │
    │ │          │ 财务RPC    │   │
    │ └──────────┘            │   │
    │ ┌──────────┐            │   │
    │ │withdraw  │ 依赖:      │   │
    │ │Service   │ currency + │   │
    │ │          │ walletCore+│   │
    │ │          │ 财务RPC +  │   │
    │ │          │ KYC + User │   │
    │ └──────────┘            │   │
    │ ┌──────────┐            │   │
    │ │exchange  │ 依赖:      │   │
    │ │Service   │ currency + │   │
    │ │          │ walletCore │   │
    │ └──────────┘            │   │
    └──────────┬───────────────┘   │
               │  产出记录数据      │
               ▼                   ▼
    ┌──────────────────────────────┐
    │ recordService (只读)          │
    │ 聚合查询所有表的记录数据       │
    └──────────────────────────────┘

依赖方向: 严格单向, 无循环依赖
  currency → 无依赖
  walletCore → currency
  deposit/withdraw/exchange → currency + walletCore + (外部RPC)
  record → 所有表(只读)
```

### 7.2 外部模块调用wallet时的路由

```
外部模块通过RPC调用wallet时, 直接进入walletCore层, 不经过交易业务层:

投注模块 → RPC → handler.DebitWallet → walletCore.DebitWallet
财务模块 → RPC → handler.CreditWallet → walletCore.CreditWallet
活动模块 → RPC → handler.RewardCredit → walletCore.RewardCredit
财务模块 → RPC → handler.FreezeBalance → walletCore.FreezeBalance

即: 外部调用 = handler直通walletCore, 跳过交易业务层
    C端请求 = handler → 交易业务层 → walletCore

这说明walletCore是真正的"核心", 被内外两条路径共同依赖
```

### 7.3 Service构造注入模式

```go
// handler.go 注入所有Service
type WalletServiceImpl struct {
    currencyService *ser.CurrencyService     // 独立
    walletService   *ser.WalletService        // → currencyService
    depositService  *ser.DepositService       // → currencyService + walletService + rpc
    withdrawService *ser.WithdrawService      // → currencyService + walletService + rpc
    exchangeService *ser.ExchangeService      // → currencyService + walletService
    recordService   *ser.RecordService        // → 只读查询
}

// 构造顺序(按依赖关系)
func NewWalletServiceImpl() *WalletServiceImpl {
    currencySer := ser.NewCurrencyService(rep)       // 1st, 无依赖
    walletSer := ser.NewWalletService(rep, currencySer) // 2nd
    depositSer := ser.NewDepositService(rep, currencySer, walletSer)   // 3rd
    withdrawSer := ser.NewWithdrawService(rep, currencySer, walletSer) // 3rd
    exchangeSer := ser.NewExchangeService(rep, currencySer, walletSer) // 3rd
    recordSer := ser.NewRecordService(rep)           // anytime
    // ...
}
```

---

## 第八部分：六大原子操作流转详解

### 8.1 原子操作总览

```
所有余额变动都必须通过以下6个原子操作之一, 没有其他入口:

┌────────────────────────────────────────────────────────────────────────┐
│                       walletCore 六大原子操作                          │
│                                                                        │
│  ① creditCore  ── 上账(+余额)     → 充值入账/奖励/手动加款/补单       │
│  ② debitCore   ── 扣款(-余额)     → 投注/消费/礼物/手动扣款           │
│  ③ freezeCore  ── 冻结(可用→冻结) → 提现发起                          │
│  ④ unfreezeCore── 解冻(冻结→可用) → 提现拒绝/打款失败                  │
│  ⑤ confirmDebitCore── 确认扣除(冻结→永久减少) → 打款成功               │
│  ⑥ writeFlow   ── 写流水(不改余额) → 所有操作的审计记录               │
│                                                                        │
│  共同模式:                                                             │
│    幂等检查(orderNo) → 前置校验 → 单事务(余额+流水) → 失效缓存         │
└────────────────────────────────────────────────────────────────────────┘
```

### 8.2 每个原子操作的内部流转

#### ① creditCore — 上账

```
入口: CreditWallet / ManualCredit / SupplementCredit / RewardCredit
  │
  ├── 1. 幂等: SELECT FROM wallet_flow WHERE order_no = ? → 已存在则返回
  ├── 2. 校验: 币种存在+启用
  ├── 3. 自动建钱包: user_wallet不存在 → INSERT
  ├── 4. BEGIN事务
  │     ├── UPDATE user_wallet SET available = available + ? WHERE uid AND currency AND wallet_type
  │     └── INSERT wallet_flow (type, amount, before, after, order_no)
  ├── 5. COMMIT
  └── 6. DEL Redis缓存 wallet:bal:{uid}:{currency}

SQL特点: 无WHERE条件约束(加钱不需要校验余额), 最简单的原子操作
```

#### ② debitCore — 扣款

```
入口: DebitWallet / ManualDebit
  │
  ├── 1. 幂等检查
  ├── 2. 查询: center_balance + reward_balance
  ├── 3. 校验: center + reward >= amount → 不足则拒绝
  ├── 4. 计算分配: centerDebit = min(center, amount), rewardDebit = amount - centerDebit
  ├── 5. BEGIN事务
  │     ├── UPDATE user_wallet SET available = available - centerDebit
  │     │     WHERE ... AND available >= centerDebit  ← 乐观锁!
  │     ├── (如需) UPDATE user_wallet SET available = available - rewardDebit
  │     │     WHERE ... AND available >= rewardDebit  ← 乐观锁!
  │     ├── INSERT center_flow
  │     └── (如需) INSERT reward_flow
  ├── 6. COMMIT
  └── 7. DEL缓存

SQL特点: WHERE available >= ? 乐观锁防止并发超扣, 双钱包单事务
返回: {centerDebit, rewardDebit} — 调用方需要这个分配信息
```

#### ③ freezeCore — 冻结

```
入口: FreezeBalance (CreateWithdrawOrder内部调用)
  │
  ├── 1. 幂等检查
  ├── 2. 校验: center_available >= amount
  ├── 3. BEGIN事务
  │     ├── UPDATE user_wallet SET available -= amount, frozen += amount
  │     │     WHERE ... AND available >= amount  ← 乐观锁
  │     ├── INSERT wallet_freeze (order_no, uid, currency, amount, status=1)
  │     └── INSERT wallet_flow (type=freeze)
  ├── 4. COMMIT
  └── 5. DEL缓存

状态: 创建freeze记录, status=1(freezing) — 后续必须走④或⑤终结
```

#### ④ unfreezeCore — 解冻

```
入口: UnfreezeBalance (财务模块回调: 审核拒绝/打款失败)
  │
  ├── 1. 查冻结记录: SELECT FROM wallet_freeze WHERE order_no = ?
  ├── 2. 校验: status 必须 = 1(freezing) → 否则拒绝(已终结)
  ├── 3. BEGIN事务
  │     ├── UPDATE user_wallet SET frozen -= amount, available += amount
  │     ├── UPDATE wallet_freeze SET status=3(unfrozen), finish_at=now
  │     └── INSERT wallet_flow (type=withdraw_return)
  ├── 4. COMMIT
  └── 5. DEL缓存

结果: 冻结金额回到可用余额, 用户可再次使用
与⑤互斥: 同一orderNo的freeze记录, ④和⑤只能执行一个
```

#### ⑤ confirmDebitCore — 确认扣除

```
入口: ConfirmDebit (财务模块回调: 三方打款成功)
  │
  ├── 1-2. 同④
  ├── 3. BEGIN事务
  │     ├── UPDATE user_wallet SET frozen -= amount  ← 只减frozen, 不加available!
  │     ├── UPDATE wallet_freeze SET status=2(confirmed_debit), finish_at=now
  │     └── INSERT wallet_flow (type=withdraw_debit)
  ├── 4. COMMIT
  └── 5. DEL缓存

结果: 冻结金额永久消失(资金已通过三方渠道到达用户银行账户)
这是"不可逆点" — 执行后资金已离开平台
```

#### ⑥ writeFlow — 写流水

```
所有原子操作内部都调用writeFlow, 它不独立存在:
  INSERT wallet_flow (
    uid, currency, wallet_type, flow_type, direction,
    amount, before_balance, after_balance, order_no,
    biz_type, remark, operator_id, created_at
  )

order_no唯一键: 幂等性的数据库级保障
```

### 8.3 原子操作调用关系图

```
                    ┌─────────────────────────────────────────┐
                    │           外部调用方                      │
                    │  财务  投注  活动  直播  手动  补单       │
                    └───┬──┬──┬──┬──┬──┬──────────────────────┘
                        │  │  │  │  │  │
        ┌───────────────▼──▼──▼──▼──▼──▼──────────────────┐
        │              handler.go (路由分发)                │
        └──┬─────┬─────┬─────┬─────┬─────┬────────────────┘
           │     │     │     │     │     │
   CreditWallet │ DebitWallet │ Freeze │ Unfreeze │ ConfirmDebit │ Manual系列
           │     │     │     │     │     │
        ┌──▼─────▼─────▼─────▼─────▼─────▼────────────────┐
        │              walletCore                           │
        │                                                   │
        │  creditCore  ←── CreditWallet, ManualCredit,     │
        │                  SupplementCredit, RewardCredit   │
        │                                                   │
        │  debitCore   ←── DebitWallet, ManualDebit         │
        │                                                   │
        │  freezeCore  ←── FreezeBalance                   │
        │  unfreezeCore←── UnfreezeBalance                 │
        │  confirmDebitCore←── ConfirmDebit                │
        │                                                   │
        │  writeFlow   ←── 所有操作内部调用                 │
        │                                                   │
        └──────────────────────────────────────────────────┘
                        │
                        ▼
                ┌───────────────┐
                │ TiDB + Redis  │
                │ user_wallet   │
                │ wallet_flow   │
                │ wallet_freeze │
                └───────────────┘
```

---

## 第九部分：依赖启动顺序与并行开发策略

### 9.1 四层启动依赖

```
Layer 0: 基础设施 (已就绪)
  ├── ETCD, Redis, TiDB
  ├── ser-blog, ser-s3, ser-i18n, ser-kyc, ser-user
  └── 这些不需要wallet开发, 直接可用

Layer 1: 币种配置初始化 (wallet独立完成, 不依赖任何业务模块)
  ├── ① 导入币种基础数据 (开发脚本)
  ├── ② SetBaseCurrency(USDT, 一次性)
  ├── ③ 启动汇率定时拉取 (3家API取平均)
  └── 完成标志: currency_config表有数据, 汇率定时更新

Layer 2: 钱包核心能力就绪 (依赖Layer1)
  ├── ④ 建表: user_wallet, wallet_flow, wallet_freeze
  ├── ⑤ 6大原子操作可用 (credit/debit/freeze/unfreeze/confirm/writeFlow)
  ├── ⑥ RPC暴露接口可调用 (CreditWallet, GetBalance等)
  └── 完成标志: 外部模块可通过RPC操作钱包

Layer 3: 财务配置就绪 (别人负责, 与Layer2并行)
  ├── ⑦ 支付渠道配置 (财务模块)
  ├── ⑧ 充值/提现方式配置 (财务模块)
  └── 完成标志: GetPaymentConfig可返回有效数据

Layer 4: C端业务流程打通 (依赖Layer1+2+3)
  ├── ⑨ 充值流程 (Layer1汇率 + Layer3支付配置 + Layer2入账)
  ├── ⑩ 提现流程 (Layer1汇率 + Layer3支付配置 + Layer2冻结/扣除 + ser-kyc)
  ├── ⑪ 兑换流程 (Layer1汇率 + Layer2余额操作) ← 最早可跑通!
  └── ⑫ 记录查询 (⑨⑩⑪产生数据后)
```

### 9.2 并行开发策略

```
Phase 1 (P0): 纯内部逻辑 — 不依赖任何外部模块
  ┌─────────────────────────────────────────────────────┐
  │ 可并行开发:                                          │
  │  ├── 币种配置CRUD (B端5个接口)                       │
  │  ├── 钱包核心6大原子操作                              │
  │  ├── 兑换流程(GetExchangeRate + DoExchange)          │
  │  ├── 记录查询(GetRecordList + GetRecordDetail)       │
  │  └── RPC暴露接口(CreditWallet等, 纯逻辑实现)         │
  │                                                      │
  │ 开发完成后可独立测试, 不等任何人                       │
  └─────────────────────────────────────────────────────┘

Phase 2 (P1): 接入已有服务
  ┌─────────────────────────────────────────────────────┐
  │ 接入:                                                │
  │  ├── ser-kyc → 提现KYC检查                           │
  │  ├── ser-user → 提现账户姓名校验                      │
  │  ├── ser-blog → B端审计日志                           │
  │  └── ser-s3 → 币种图标上传                            │
  │                                                      │
  │ 这些服务已有IDL, 接入即用                              │
  └─────────────────────────────────────────────────────┘

Phase 3 (P2): Mock财务模块做端到端联调
  ┌─────────────────────────────────────────────────────┐
  │ Mock:                                                │
  │  ├── FinanceClient.GetPaymentConfig → 返回预设配置    │
  │  ├── FinanceClient.CreatePayOrder → 返回模拟订单号    │
  │  ├── FinanceClient.CreateWithdrawOrder → 返回模拟单号 │
  │                                                      │
  │ 模拟: CreditWallet回调 → 验证充值全链路               │
  │ 模拟: ConfirmDebit/Unfreeze回调 → 验证提现全链路      │
  └─────────────────────────────────────────────────────┘

Phase 4 (P3): 替换Mock为真实财务模块
  ┌─────────────────────────────────────────────────────┐
  │ 集成:                                                │
  │  ├── 获取财务模块真实IDL → 替换Mock                   │
  │  ├── 联调充值全链路(含三方支付)                        │
  │  ├── 联调提现全链路(含审核+打款)                       │
  │  └── 压力测试(并发扣款, 并发冻结等)                    │
  └─────────────────────────────────────────────────────┘
```

### 9.3 最早可端到端验证的链路

```
优先级排序(从最易到最难):

第1条: 兑换流程 (DoExchange)
  ✅ 零外部依赖, 纯内部闭环
  ✅ 涉及跨币种操作, 能验证事务原子性
  ✅ P0阶段即可完成端到端测试

第2条: RPC暴露接口单元测试
  ✅ CreditWallet / DebitWallet / GetBalance
  ✅ 纯钱包操作, 无外部依赖
  ✅ P0阶段可完成

第3条: 充值流程(Mock财务)
  ⚠️ 需Mock财务模块2个接口
  ⚠️ 需模拟CreditWallet回调
  ✅ P2阶段可完成

第4条: 提现流程(Mock财务)
  ⚠️ 需Mock财务+KYC+User 3个模块
  ⚠️ 需模拟审核回调(Unfreeze/ConfirmDebit)
  ⚠️ 7个前置条件, 异常分支最多
  ✅ P2阶段可完成, 但测试用例最复杂
```

---

## 第十部分：复杂度热力图与挑战识别

### 10.1 接口复杂度评级

```
★★★★★ (最复杂):
  C12: CreateWithdrawOrder — 7前置条件, 3跨模块调用, 冻结回滚, 异常分支最多
  C8:  DoExchange — 单事务7条SQL, 跨2币种3钱包, 原子性要求极高

★★★★☆ (复杂):
  C4:  CreateDepositOrder — 2跨模块调用, 汇率锁定, 活动验证
  C9:  GetWithdrawMethods — 3数据源交叉过滤(KYC×充值历史×财务配置)
  R2:  DebitWallet — 双钱包扣款分配, 乐观锁, 返回分配信息

★★★☆☆ (中等):
  C3:  GetDepositMethods — 首个跨模块调用点, Mock策略
  C11: SaveWithdrawAccount — 跨模块姓名校验
  B2:  EditCurrency — 图标上传+审计日志
  R1:  CreditWallet — 幂等+自动建钱包+入账
  R3-R5: Freeze三件套 — 状态机正确性

★★☆☆☆ (一般):
  C1, C2, C7, C13, C14, B1, B5-B7, R6, R8, R9

★☆☆☆☆ (简单):
  C5, C6, C10, C15, C16, B3, B4, B8, B9, R10-R13
```

### 10.2 技术挑战热点

```
挑战1: 兑换事务原子性 ━━━━━━━━━━━━━━━━━━━━ 难度:★★★★★
  场景: DoExchange单事务内7条SQL, 跨2个币种3个钱包
  风险: 部分成功(法币扣了但BSB没加) → 用户资金丢失
  方案: GORM事务 + defer rollback + 写入前后余额到flow表审计
  验证: 必须写并发测试(多用户同时兑换, 汇率刷新并发)

挑战2: 提现冻结回滚 ━━━━━━━━━━━━━━━━━━━━━━ 难度:★★★★☆
  场景: CreateWithdrawOrder中FreezeBalance成功后, 财务调用失败
  风险: 用户余额被冻结但提现未创建 → 资金"悬空"
  方案: freeze后try-catch, 财务失败则立即UnfreezeBalance
  验证: 模拟财务模块超时/返回错误, 验证冻结是否正确回滚

挑战3: 并发扣款防超扣 ━━━━━━━━━━━━━━━━━━━━━ 难度:★★★★☆
  场景: 同一用户同时发起多笔投注扣款
  风险: 并发UPDATE导致余额为负
  方案: SQL乐观锁 WHERE available >= amount + Redis分布式锁双保险
  验证: 压力测试(100并发扣款同一用户)

挑战4: 幂等性设计 ━━━━━━━━━━━━━━━━━━━━━━━━ 难度:★★★☆☆
  场景: 财务模块因超时重试CreditWallet
  风险: 重复入账 → 用户余额多加
  方案: wallet_flow.order_no唯一键, 入库前先查
  验证: 同一orderNo连续调用2次, 验证只入账1次

挑战5: 汇率快照一致性 ━━━━━━━━━━━━━━━━━━━━━ 难度:★★★☆☆
  场景: 用户查看汇率后,在确认前汇率更新了
  风险: 用户看到的预览金额和实际执行金额不一致
  方案: 确认时锁定当前汇率, 用锁定值计算, 记录到订单快照
  验证: 模拟查看→确认期间汇率变化, 验证实际计算用的是哪个值

挑战6: 缓存一致性 ━━━━━━━━━━━━━━━━━━━━━━━━ 难度:★★★☆☆
  场景: 余额变动后缓存失效, 但高并发下可能读到旧值
  风险: 用户看到错误余额
  方案: 写后立即DEL缓存(非更新) + TTL兜底(5min)
  验证: 充值入账后立即查询, 验证余额已更新
```

### 10.3 模块间边界风险

```
风险1: 财务模块IDL完全缺失 ━━━━━━━━━━━━━━━━ 影响:全链路
  现状: 服务名未定, IDL未定义, 无法接入
  影响: 充值/提现流程无法真实联调
  缓解: Mock策略, 但Mock覆盖不了真实的接口契约差异

风险2: 充值/提现记录数据源不明 ━━━━━━━━━━━━━ 影响:记录查询
  问题: C端交易记录从本地查还是从财务模块查?
  如果从本地查 → wallet需要维护完整订单生命周期
  如果从财务查 → wallet需要跨模块RPC, 增加依赖
  需确认: 与财务组讨论

风险3: "充什么提什么"数据来源 ━━━━━━━━━━━━━ 影响:提现方式过滤
  问题: 用户充值历史从哪里查?
  如果wallet记录 → deposit_order表
  如果财务记录 → 需RPC查询财务
  需确认: 充值方式是否保存在wallet本地

风险4: 活动红利接口不明 ━━━━━━━━━━━━━━━━━━ 影响:充值+兑换
  问题: 红利比例、活动校验从哪个模块获取?
  需确认: 活动模块是否有IDL? 红利配置在哪?
```

---

## 第十一部分：异常链路与降级方案

### 11.1 充值链路异常

```
异常1: GetPaymentConfig超时/返回空
  触发: C3 GetDepositMethods
  影响: 用户看不到充值方式
  降级: 返回空列表+友好提示"暂无可用充值方式"
  后续: 用户可稍后重试

异常2: CreatePayOrder超时
  触发: C4 CreateDepositOrder
  影响: 不确定财务是否已创建渠道订单
  处理: 不创建本地订单, 返回"系统繁忙", 用户重试
  关键: 不能创建本地订单(否则可能有"幽灵订单")

异常3: CreditWallet回调重复/乱序
  触发: R1 CreditWallet
  影响: 可能重复入账
  处理: orderNo幂等 → 第二次调用直接返回成功
  关键: 幂等返回值应与首次一致

异常4: 充值成功但缓存未更新
  触发: R1 CreditWallet后DEL缓存失败
  影响: 用户短暂看到旧余额
  处理: TTL 5分钟兜底, 最多5分钟后自动纠正
```

### 11.2 提现链路异常

```
异常5: 冻结成功 + 财务调用失败
  触发: C12 CreateWithdrawOrder 步骤⑤成功, 步骤⑦失败
  影响: 用户余额被冻结但提现未创建 → 资金"悬空"
  处理:
    try {
      FreezeBalance(...)      // ⑤ 成功
      CreateLocalOrder(...)   // ⑥
      FinanceClient.CreateWithdrawOrder(...) // ⑦ 失败!
    } catch {
      UnfreezeBalance(...)    // 回滚冻结
      UpdateLocalOrder(status=creation_failed)
      return error
    }
  关键: 这是最危险的异常分支, 必须100%回滚

异常6: ConfirmDebit/UnfreezeBalance超时
  触发: R4/R5, 财务模块回调wallet
  影响: 财务不确定是否执行成功
  处理: wallet端幂等, 财务重试安全
  关键: freeze记录已变为终态后, 重复调用直接返回成功

异常7: 并发提现
  触发: 同一用户快速连续提交两笔提现
  影响: 第一笔冻结后余额不足, 第二笔应失败
  处理: SQL乐观锁 WHERE available >= amount → 第二笔UPDATE影响行数=0 → 返回余额不足
  关键: 不需要分布式锁, SQL层即可保证

异常8: KYC状态在提现过程中变化
  触发: 用户开始提现时已认证, 确认时被取消认证
  处理: CreateWithdrawOrder中防御性重检KYC
  关键: Stage5中的KycQuery不是冗余的, 是必要的防御
```

### 11.3 兑换链路异常

```
异常9: 兑换事务部分失败
  触发: C8 DoExchange, 7条SQL中某条失败
  影响: 可能法币扣了但BSB没加
  处理: 全部在单事务内, 任何失败自动ROLLBACK
  关键: 绝不能把7条SQL拆成多个事务

异常10: 兑换过程中汇率刷新
  触发: 用户确认兑换时, 后台定时任务正好更新汇率
  处理: 确认时锁定汇率(读取后不再变), 用锁定值计算
  关键: 锁定方式 — 事务开始时读取汇率, 事务内使用该值
```

---

## 第十二部分：关键结论与开发指导

### 12.1 链路复杂度排名

```
最复杂的链路(开发重点):
  1. 提现创建 (C12) — 7前置, 3跨模块, 冻结回滚, 异常分支最多
  2. 兑换执行 (C8) — 单事务7SQL, 跨币种原子操作
  3. 提现方式 (C9) — 3数据源交叉过滤

最关键的链路(系统生命线):
  1. CreditWallet (R1) — 充值入账终点, 幂等性是生命线
  2. DebitWallet (R2) — 投注扣款, 防超扣是底线
  3. Freeze三件套 (R3-R5) — 状态机正确性决定资金安全

最独立的链路(优先开发):
  1. 兑换流程 — 零外部依赖, 最早可验证
  2. 币种配置CRUD — 纯内部逻辑
  3. 余额查询 — 纯读操作
```

### 12.2 开发优先级建议

```
第一批(P0, 可立即开发, 不等任何人):
  ┌─────────────────────────────────────────────────┐
  │ 币种配置层:                                      │
  │   currency_config表 + CRUD + 汇率定时任务         │
  │                                                  │
  │ 钱包核心层:                                      │
  │   user_wallet表 + wallet_flow表 + wallet_freeze表│
  │   6大原子操作(credit/debit/freeze/unfreeze/      │
  │   confirmDebit/writeFlow)                        │
  │                                                  │
  │ 兑换流程:                                        │
  │   GetExchangeRate + DoExchange + exchange_order表│
  │                                                  │
  │ RPC暴露接口:                                     │
  │   CreditWallet, DebitWallet, GetBalance,          │
  │   FreezeBalance, UnfreezeBalance, ConfirmDebit    │
  │                                                  │
  │ 记录查询:                                        │
  │   GetRecordList + GetRecordDetail                │
  └─────────────────────────────────────────────────┘

第二批(P1, 接入已有服务):
  ┌─────────────────────────────────────────────────┐
  │ ser-kyc集成 → 提现KYC校验                        │
  │ ser-user集成 → 提现账户姓名校验                   │
  │ ser-blog集成 → B端审计日志                        │
  │ ser-s3集成 → 币种图标上传                         │
  └─────────────────────────────────────────────────┘

第三批(P2, Mock联调):
  ┌─────────────────────────────────────────────────┐
  │ Mock财务模块:                                    │
  │   GetPaymentConfig → 充值/提现方式               │
  │   CreatePayOrder → 充值下单                      │
  │   CreateWithdrawOrder → 提现下单                 │
  │ 充值全链路联调(含Mock回调)                        │
  │ 提现全链路联调(含Mock审核回调)                    │
  └─────────────────────────────────────────────────┘
```

### 12.3 编码阶段核心指导原则

```
原则1: 所有余额变动必须通过walletCore六大原子操作
  → 绝不允许在Service层直接写SQL修改balance
  → 任何新增的余额操作都必须走原子操作入口

原则2: 每个原子操作都必须幂等
  → orderNo唯一键是幂等基础
  → 先查再写, 已存在直接返回成功

原则3: 单事务 = 余额变动 + 流水写入
  → 不允许拆开, balance和flow必须在同一事务
  → defer tx.Rollback() 兜底

原则4: SQL乐观锁防并发
  → UPDATE ... WHERE available >= amount
  → affected_rows == 0 → 余额不足, 拒绝操作

原则5: 冻结和财务调用的顺序
  → 先冻结, 后调财务
  → 财务失败必须回滚冻结
  → 不允许反过来(先调财务后冻结 → 资金不安全)

原则6: 缓存策略统一
  → 写后DEL缓存(不是SET)
  → TTL 5分钟兜底
  → 读时Cache-Aside: 查Redis → 未命中查DB → 写回Redis

原则7: 外部依赖隔离
  → 所有外部RPC调用封装在独立方法中
  → 财务模块调用统一走FinanceClient(方便Mock替换)
  → Mock和真实实现通过接口切换

原则8: 金额全链路整数
  → i64 (int64), BIGINT, 最小单位存储
  → 不允许浮点数出现在金额计算链路中
  → 展示层才做格式化(小数点/千分位)
```

### 12.4 一句话总结每条链路

```
C1  GetWalletHome      : 查余额+币种信息, 纯内部, 走缓存
C2  GetCurrencyList    : 批量查币种+余额, 纯内部
C3  GetDepositMethods  : 查充值方式, 第一个跨模块调用点(→财务)
C4  CreateDepositOrder : 创建充值单, 调财务+活动, 不动余额
C5  CheckDuplicate     : 查重复转账, 纯DB查询
C6  GetDepositStatus   : 查充值状态, 纯读
C7  GetExchangeRate    : 查汇率+预览, 纯内部, 最独立
C8  DoExchange         : 执行兑换, 单事务7SQL, 跨币种原子, 最关键内部操作
C9  GetWithdrawMethods : 查提现方式, 3数据源交叉过滤, 逻辑最复杂
C10 GetWithdrawAccount : 查提现账户, 纯DB读
C11 SaveWithdrawAccount: 存提现账户, 校验KYC姓名(→ser-user)
C12 CreateWithdrawOrder: 创建提现单, 冻结→财务→回滚, 全系统最复杂接口
C13 GetRecordList      : 查记录列表, 分tab聚合查询
C14 GetRecordDetail    : 查记录详情, 提现最复杂(关联3表)
C15 GetBonusList       : 查红利列表, 调活动模块
C16 GetWithdrawStatus  : 查提现状态, 纯读

B1-B5 币种配置CRUD     : 纯内部+审计日志, 最标准的CRUD链路
B6-B7 用户钱包查看     : 纯读+批量用户信息(→ser-user)
B8-B9 补充查询         : 导出/详情, Should-Have

R1  CreditWallet       : 充值终点, 幂等+自动建钱包+入账
R2  DebitWallet        : 投注扣款, 双钱包分配+乐观锁
R3  FreezeBalance      : 冻结起始, 状态机entry
R4  UnfreezeBalance    : 解冻(拒绝/失败), 状态机分支A
R5  ConfirmDebit       : 确认扣除(打款成功), 状态机分支B, 不可逆
R6  ManualCredit       : 手动加款, 可指定钱包类型
R7  ManualDebit        : 手动扣款, 级联扣+部分扣除
R8  SupplementCredit   : 充值补单, 可同时中心+奖励
R9  GetBalance         : 余额查询, 全平台最高频, 必须走缓存
R10 BatchGetBalance    : 批量余额, Should-Have
R11 RewardCredit       : 奖励入账, 进奖励钱包+流水倍率
R12 GetCurrencyConfig  : 币种查询, Should-Have
R13 GetExchangeRateRpc : 汇率查询, Should-Have
```

---

## 附录A: 全链路检查清单

```
开发每个接口前, 对照以下清单:

□ 该接口涉及哪些跨模块RPC调用? 目标服务IDL是否可用?
□ 是否有余额变动? 如果有, 走的是哪个原子操作?
□ 是否需要幂等? orderNo从哪来? 唯一键在哪张表?
□ 是否有事务? 事务边界是什么? 几条SQL在事务内?
□ 异常时如何回滚? 特别是跨模块调用失败后是否有冻结需要回滚?
□ 缓存策略: 读走缓存? 写后DEL哪些key?
□ 并发安全: 是否需要SQL乐观锁? 是否需要分布式锁?
□ 返回值: 调用方需要什么信息? (如DebitWallet需返回分配比例)
□ 该接口在哪个Phase可以开发? 前置依赖是否就绪?
```

## 附录B: 接口→原子操作映射速查

```
┌──────────────────────────┬────────────────────────────┐
│ 接口                      │ 使用的原子操作              │
├──────────────────────────┼────────────────────────────┤
│ C4  CreateDepositOrder   │ 无(仅建单, 不动余额)        │
│ C8  DoExchange           │ debitCore + creditCore ×2   │
│ C12 CreateWithdrawOrder  │ freezeCore                  │
│ R1  CreditWallet         │ creditCore                  │
│ R2  DebitWallet          │ debitCore                   │
│ R3  FreezeBalance        │ freezeCore                  │
│ R4  UnfreezeBalance      │ unfreezeCore                │
│ R5  ConfirmDebit         │ confirmDebitCore             │
│ R6  ManualCredit         │ creditCore                  │
│ R7  ManualDebit          │ debitCore (级联)             │
│ R8  SupplementCredit     │ creditCore (×1或×2)         │
│ R11 RewardCredit         │ creditCore                  │
└──────────────────────────┴────────────────────────────┘

所有其他接口(C1-C3, C5-C7, C9-C11, C13-C16, B1-B9, R9-R10, R12-R13)
均为纯读操作或配置管理, 不涉及原子操作。
```

---

> **本文档定位**: 编码阶段的"地图" — 开发任何接口前先查此文档, 明确该接口在全链路中的位置、依赖关系、复杂度级别和注意事项。
> **下一步**: 按P0→P1→P2→P3的顺序推进开发, 每完成一个Phase做一次全链路验证。
