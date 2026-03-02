# 钱包模块宏观请求链路全景分析

> **文档定位**：从外部观测者视角追踪每个请求的完整生命周期
> **核心价值**：指导方针——清楚每一步做什么，哪步复杂关键，哪步存在挑战
> **思维框架**：处理一件事 = 流程 + 判断 + 循环，每条链路都拆解为这三种模式
> **★标注系统**：★ = 需注意 / ★★ = 复杂关键 / ★★★ = 高风险挑战

---

## 目录

- [第1章：请求流转架构全景](#第1章请求流转架构全景)
- [第2章：六大业务链路分类总览](#第2章六大业务链路分类总览)
- [第3章：链路一 — 币种配置管理](#第3章链路一--币种配置管理)
- [第4章：链路二 — 钱包首页与余额查询](#第4章链路二--钱包首页与余额查询)
- [第5章：链路三 — 充值全链路](#第5章链路三--充值全链路)
- [第6章：链路四 — 兑换全链路](#第6章链路四--兑换全链路)
- [第7章：链路五 — 提现全链路](#第7章链路五--提现全链路)
- [第8章：链路六 — RPC被调与回调链路](#第8章链路六--rpc被调与回调链路)
- [第9章：跨链路共性模式提炼](#第9章跨链路共性模式提炼)
- [第10章：关键节点复杂度与挑战全景标注](#第10章关键节点复杂度与挑战全景标注)
- [附录：链路速查卡片](#附录链路速查卡片)

---

## 第1章：请求流转架构全景

### 1.1 平台级请求流转模型

每一个请求，无论是C端用户操作还是B端管理员操作，都遵循相同的基础流转路径：

```
用户(App/H5/Web)
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Hertz 网关层 (gate-font / gate-back)                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  中间件链（按顺序执行，任一失败即中断）                    │    │
│  │  ① TraceID生成 → ② Header提取 → ③ IP白名单(B端)       │    │
│  │  → ④ Token认证 → ⑤ URL权限校验(B端)                    │    │
│  └──────────────────────────────────────────────────────┘    │
│  ┌──────────────────┐                                        │
│  │ Handler（超薄）   │  ← 一行代码：rpc.Handler(ctx, c, fn)  │
│  │ 职责：绑参+转发   │     反射初始化请求体→BindAndValidate   │
│  └────────┬─────────┘     → 调RPC → 包装响应                │
└───────────┼─────────────────────────────────────────────────┘
            │ Kitex RPC (TTHeader协议，3秒超时)
            │ 上下文透传：metainfo → TTHeader → 跨服务重建
            ▼
┌─────────────────────────────────────────────────────────────┐
│  业务服务层 (ser-wallet / ser-finance / ser-user / ...)      │
│  ┌──────────────────┐                                        │
│  │ RPC Handler      │  ← 同样超薄：直接转发给Service         │
│  └────────┬─────────┘                                        │
│           ▼                                                   │
│  ┌──────────────────┐                                        │
│  │ Service (业务)    │  ← 核心逻辑：校验、计算、协调           │
│  │ 职责：编排业务流   │     可能调用其他RPC服务                 │
│  └────────┬─────────┘                                        │
│           ▼                                                   │
│  ┌──────────────────┐                                        │
│  │ Repository (数据) │  ← 数据访问：GORM Gen自动生成+自定义    │
│  │ 职责：DB CRUD     │                                        │
│  └────────┬─────────┘                                        │
└───────────┼─────────────────────────────────────────────────┘
            │
            ▼
    ┌───────────────┐
    │ TiDB / MySQL   │  GORM + UserPlugin(自动填充审计字段)
    │ + Redis        │  缓存：币种配置Cache-Aside / 余额不缓存
    └───────────────┘
```

### 1.2 中间件链执行序列

**C端请求中间件**（gate-font）：
```
① SetHeaderInfo()   → 提取ClientIP/Language/DeviceID/Token
                       Token优先级：X-User-Token header → session_token cookie
② FontAuthMiddleware → Redis查询：rds.Get("user:token:" + token)
                       解析CUserTokenPayload{ID, UID, NickName, ...}
                       注入userId/userName到metainfo上下文
```

**B端请求中间件**（gate-back）：
```
① SetHeaderInfo()     → 同上
② WhiteCheck()        → RPC调ser-ip: OnWhiteList(ctx)
                         失败/不在白名单 → 直接拦截返回403
③ BackAuthMiddleware  → Redis查询：rds.Get(token) ← 注意无前缀！
                         解析BUserTokenPayload{ID, UserName}
                         RPC调ser-buser: CheckUrlAuth(ctx, fullPath)
                         URL权限不通过 → 拦截返回403
```

### 1.3 上下文传播机制

```
请求进入 → metainfo.WithPersistentValue(ctx, key, value) 写入上下文
    │
    │ 同服务内部：直接通过ctx传递
    │
    │ 跨服务RPC：Kitex ClientTTHeaderHandler 序列化 → 网络传输
    │            → Kitex ServerTTHeaderHandler 反序列化 → 重建ctx
    │
    ▼
任何层级都可以：tracer.GetUserId(ctx) / tracer.GetTraceId(ctx) 读取
```

**透传的关键字段**：TraceId、UserId、UserName、UserToken、ClientIP、Language

### 1.4 统一响应格式

```json
{
    "traceId": "uuid-xxx",     // 链路追踪ID，全链路唯一
    "code": 0,                 // 0=成功，非0=业务错误码
    "msg": "",                 // 错误信息（成功时为空）
    "data": { ... }            // 业务数据
}
```

所有HTTP响应状态码均为200，业务错误通过code字段区分。

---

## 第2章：六大业务链路分类总览

### 2.1 链路全景图

```
                        复杂度    跨模块RPC数    耗时        核心挑战
                        ━━━━━    ━━━━━━━━━    ━━━━━━      ━━━━━━━━
链路1: 币种配置管理      ★☆☆☆☆    1(审计日志)   毫秒级      精度字段不可变
链路2: 钱包首页与余额    ★★☆☆☆    0             毫秒级      余额实时性
链路3: 充值全链路        ★★★★☆    3(2出+1入)    分钟~小时    异步回调+幂等
链路4: 兑换全链路        ★★★☆☆    0             毫秒级      单事务多表+CAS
链路5: 提现全链路        ★★★★★    4(2出+2入)    小时~天      冻结模型+补偿
链路6: RPC被调链路       ★★★☆☆    0(被调方)     毫秒级      双层幂等+CAS
```

### 2.2 每条链路的起点→终点

| 链路 | 起点 | 终点 | 性质 |
|------|------|------|------|
| 1-币种配置 | B端管理员操作 | DB写入+缓存失效 | 同步、B端、写 |
| 2-余额查询 | C端打开钱包页 | UI渲染余额数据 | 同步、C端、读 |
| 3-充值 | C端点击"充值" | 余额增加（异步） | 异步、跨模块、写 |
| 4-兑换 | C端点击"兑换" | 余额即时变动 | 同步、内部、写 |
| 5-提现 | C端点击"提现" | 冻结→最终扣除/退回（异步） | 异步、跨模块、写 |
| 6-RPC被调 | 外部模块RPC调用 | 余额变动+返回结果 | 同步、被动、写/读 |

### 2.3 链路依赖关系

```
开发顺序依赖关系图：

  链路1(币种配置) ──→ 链路2(余额查询) ──→ 链路4(兑换)
        │                    │                │
        │                    │                │
        │         链路6(RPC被调，部分)          │
        │              │                       │
        ▼              ▼                       ▼
  ─────────────── P2 期 ──────────────   P3 期（烟囱测试）
                                               │
                                               ▼
                               链路3(充值) + 链路5(提现)
                               + 链路6(RPC被调，完整)
                                     │
                                     ▼
                               ─── P4 期（财务集成）───

说明：
  链路1 是所有链路的基础（币种不存在，后续都无法操作）
  链路4(兑换) 是纯内部闭环，适合做烟囱测试
  链路3/5 强依赖财务模块，必须等IDL就绪
```

---

## 第3章：链路一 — 币种配置管理

### 3.1 链路概述

```
复杂度：★☆☆☆☆
请求入口：B端管理后台
请求类型：同步CRUD
跨模块依赖：仅ser-blog（审计日志，oneway）
数据表：currency_config + exchange_rate
开发期：P2
```

### 3.2 完整请求链路

```
B端管理员操作（新增/编辑/启停/汇率设置）
    │
    ▼
gate-back 中间件链
    ├─ [判断] IP白名单校验 → RPC ser-ip.OnWhiteList
    ├─ [判断] B端Token认证 → Redis查询 → 解析管理员身份
    └─ [判断] URL权限校验 → RPC ser-buser.CheckUrlAuth
    │
    ▼ （任一失败 → 403拦截，链路终止）
Handler (一行转发)
    │
    ▼ Kitex RPC
ser-wallet.CurrencyService
    │
    ├─ 新增币种:
    │   ├─ [判断] 参数校验（名称/代码/精度/限额...）
    │   ├─ [判断] 唯一性校验（币种代码不重复）
    │   ├─ [流程] 写入currency_config（默认status=禁用）
    │   ├─ [流程] 异步：删除Redis缓存（Cache-Aside失效）
    │   └─ [流程] 异步：RPC ser-blog.AddActLog（审计日志，oneway不等响应）
    │
    ├─ 编辑币种:
    │   ├─ [判断] 查询旧值
    │   ├─ [判断] ★ 精度字段(decimalPlaces)上线后不可修改
    │   ├─ [流程] 更新DB → 删缓存 → 审计日志（记录before/after）
    │
    ├─ 启用/禁用:
    │   ├─ [流程] 更新status字段 → 删缓存 → 审计日志
    │   └─ [影响] 禁用后C端不可见，但已有余额不受影响
    │
    └─ 设置汇率:
        ├─ [判断] 查询旧汇率
        ├─ [流程] 更新exchange_rate表 → 记录汇率变更日志
        └─ [影响] 下次兑换/换算使用新汇率

    │
    ▼
返回 → {traceId, code:0, data: ...}
```

### 3.3 关键节点标注

| 节点 | 等级 | 说明 |
|------|------|------|
| 精度字段immutable | ★ | currency_config.decimalPlaces 上线后不可修改，否则历史数据全部失效 |
| 缓存失效策略 | ★ | 写DB后异步删Redis，下次读时重建缓存，TTL 5分钟 |
| 审计日志 | - | oneway调用，不影响主链路性能，即使失败也不阻塞 |

**指导方针**：这是最简单的链路，零外部依赖，P2期首先实现。用来验证四层架构骨架是否通畅。

---

## 第4章：链路二 — 钱包首页与余额查询

### 4.1 链路概述

```
复杂度：★★☆☆☆
请求入口：C端打开钱包/个人中心
请求类型：同步读操作
跨模块依赖：无
数据表：currency_config（缓存）+ user_balance（实时）
开发期：P2
```

### 4.2 完整请求链路

```
用户打开钱包页面
    │
    ▼
gate-font 中间件链
    ├─ [流程] Header提取 + Token认证
    └─ [判断] Token有效？→ 无效则401
    │
    ▼
Handler → RPC → ser-wallet.WalletService
    │
    │  ═══ 并行两个分支 ═══
    │
    ├─ 分支1：可用币种列表
    │   ├─ [流程] 查Redis缓存 → 命中则返回
    │   ├─ [流程] 缓存未命中 → 查DB currency_config WHERE status=启用
    │   ├─ [流程] 写入Redis缓存（TTL 5分钟）
    │   └─ 返回：[{VND, 越南盾, icon, 精度0}, {USDT, ...}, {BSB, ...}]
    │
    └─ 分支2：用户余额明细
        ├─ [流程] ★ 直接查DB（余额不缓存）
        │   SELECT * FROM user_balance
        │   WHERE user_id = ? AND currency_code IN (启用币种列表)
        ├─ [流程] 聚合：每个币种4个子钱包 → 总可用+总冻结
        └─ 返回：[{VND: center=770K, reward=0, total=770K}, ...]
    │
    ├─ 合并两个分支结果
    ▼
返回C端 → 渲染钱包首页

    ═══ 币种切换操作 ═══

用户切换币种（如VND→USDT）
    │
    ▼
只刷新分支2（余额查询），分支1（币种列表）不变
    │
    ▼
返回新币种的余额明细
```

### 4.3 关键节点标注

| 节点 | 等级 | 说明 |
|------|------|------|
| 余额不缓存 | ★ | 资金数据必须实时准确，不能容忍脏读。每次查DB，走主键/索引，<5ms |
| 币种配置缓存 | ★ | Cache-Aside模式，写时删缓存，读时重建，TTL 5分钟 |
| 并行查询 | - | 两个分支无依赖，可以goroutine并行执行 |
| 币种切换 | - | 只刷新余额分支，不重新查币种列表（已在内存中） |

**指导方针**：纯读链路，主要关注查询性能。如果未来并发量大，user_balance表需要按user_id建好索引（复合主键本身已包含）。

---

## 第5章：链路三 — 充值全链路

### 5.1 链路概述

```
复杂度：★★★★☆
请求入口：C端点击"充值"
请求类型：异步跨模块（5个阶段）
跨模块依赖：ser-finance（2个出站RPC + 1个入站RPC回调）
数据表：deposit_order + user_balance + balance_flow
开发期：P4（依赖财务IDL）
```

### 5.2 五阶段全链路时序图

```
时间轴
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0                T1              T2          T3             T4
进入充值页        创建充值订单      用户支付     入账回调        补偿查询
(毫秒)           (秒级)          (分钟~小时)   (毫秒)         (分钟,可选)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

角色切换：
  T0-T1: 钱包主导，财务被调
  T2:    外部（用户+第三方），钱包不参与
  T3:    财务主导，钱包被调
  T4:    钱包主导（定时任务），财务被调
```

### 5.3 T0: 进入充值页

```
C端 → 钱包 GetDepositMethods
    │
    ├─ [流程] RPC → 财务.GetPaymentMethods(currency=VND, type=deposit)
    │   └─ 财务返回：[{银行转账, 快捷金额[100K,500K,1M], 限额50K~50M}, ...]
    │
    ├─ [可选] 查询奖金活动列表
    │   └─ 返回：[{首充送50%, 满100K送, 流水3倍}, ...]
    │
    ▼
C端渲染：充值方式列表 + 快捷金额按钮 + 奖金活动展示

[失败路径]
  财务RPC不可用 → 返回空列表 + "充值暂不可用" → 链路终止
  影响：★ 低（用户还没操作，无资金损失）
```

### 5.4 T1: 创建充值订单（钱包主导）

```
用户选方式(银行转账) + 输金额(500,000 VND) + 选活动(首充50%) + 确认
    │
    ▼
C端 → 钱包 CreateDepositOrder
    │
    ├─ [判断] ★ 重复转账三级拦截
    │   └─ 查本地DB: SELECT COUNT(*) FROM deposit_order
    │      WHERE user_id=X AND amount=500000 AND status=处理中
    │      ├─ 0笔 → 通过
    │      ├─ 1笔 → 返回 DUPLICATE_WARN_1，C端弹窗警告，用户可force=true继续
    │      ├─ 2笔 → 返回 DUPLICATE_WARN_2，C端强警告，用户可修改金额
    │      └─ ≥3笔 → 返回 DUPLICATE_BLOCK，拦截，必须修改金额或等待
    │
    ├─ [判断] 金额范围校验
    │   └─ 查币种配置: 50,000 ≤ 500,000 ≤ 50,000,000 → 通过
    │
    ├─ [判断] 奖金活动有效性校验
    │   └─ 活动存在？金额满足最低要求？活动未过期？
    │
    ├─ [流程] 生成平台订单号: C + 16位雪花ID (如 "C1234567890123456")
    │
    ├─ [流程] 写入本地 deposit_order
    │   └─ user_id / currency / amount / method / status=待支付
    │      order_no / bonus_activity_id / bonus_amount
    │
    ├─ [流程] ★★ RPC → 财务.MatchAndCreateChannelOrder
    │   入参: userId, currencyCode=VND, amount=500000,
    │         methodId=1, platformOrderNo="C1234567890123456"
    │   │
    │   │ 财务内部（钱包不管）:
    │   │   通道轮询匹配（成功率/时间/告警系数加权）
    │   │   → 调第三方渠道API创建支付单
    │   │
    │   └─ 返回: {channelOrderNo:"CH...", paymentType:1, paymentUrl:"https://pay.xxx"}
    │
    ├─ [流程] 更新本地订单: 写入channelOrderNo，状态保持=待支付
    │
    ▼
C端：跳转第三方支付页 / 展示QR码 / 展示USDT地址

[失败路径]
  财务RPC失败 → 本地订单标记=失败 → C端提示"暂不可用"
  影响：★ 低（充值是"先支付后入账"，此时余额未变动）

[USDT特殊分支]
  法币: paymentUrl → 跳转第三方支付页
  USDT: walletAddress + qrCodeUrl + network(TRC20/ERC20) → 展示收款地址+QR码
```

**★关键标注：O-2 RPC超时处理**
```
MatchAndCreateChannelOrder 是非幂等操作（创建资源）
  超时 → 不能重试（可能创建重复渠道订单）
  超时后处理：
    ├─ 本地订单标记为"待确认"
    ├─ 用T4补偿查询流程通过O-4确认状态
    └─ 或由财务侧做超时订单清理
```

### 5.5 T2: 用户在第三方支付

```
用户 → 第三方支付(银行/电子钱包/USDT链上)
    │
    │  钱包完全不参与这个阶段
    │  等待时间：几秒（银行）~ 几十分钟（USDT链上确认）
    │
    ▼
支付完成 → 第三方回调财务
```

### 5.6 T3: 入账回调（财务主导，钱包被调）★★

```
第三方 → 财务确认支付成功
    │
    ▼
财务 → RPC → 钱包.CreditWallet
    入参: userId, currencyCode=VND, walletType=CENTER(1),
          amount="500000", orderNo="C1234567890123456", opType=DEPOSIT(1)
    │
    ├─ [判断] ★★ 幂等校验（第一层 — 应用层快速路径）
    │   └─ 查balance_flow: WHERE ref_order_no="C..." AND change_type=CREDIT
    │      ├─ 已存在且成功 → 直接返回SUCCESS（不重复入账）
    │      └─ 不存在 → 继续
    │
    ├─ [判断] 订单状态校验
    │   └─ 查deposit_order: status必须=待支付
    │      ├─ 已成功 → 返回SUCCESS（幂等）
    │      └─ 其他异常状态 → 返回错误
    │
    ├─ [流程] ★★ [BEGIN 事务]
    │   ├─ UPDATE deposit_order SET status=成功
    │   ├─ UPDATE user_balance SET available = available + 500000
    │   │   WHERE user_id=X AND currency='VND' AND wallet_type=CENTER
    │   ├─ INSERT balance_flow (ref_order_no, change_type=CREDIT, amount=500000,
    │   │   balance_before=旧值, balance_after=新值)
    │   │   └─ UNIQUE KEY (ref_order_no, change_type) ← 第二层幂等兜底
    │   [COMMIT 事务]
    │
    ├─ [可选] 奖金入账（第二次调用）
    │   └─ 财务 → RPC → 钱包.CreditWallet
    │      walletType=REWARD(2), amount=250000(50%奖金),
    │      opType=BONUS(2), auditMultiplier=3
    │      └─ 同样的幂等+事务流程，但操作REWARD子钱包
    │
    ▼
返回财务: {success:true, newBalance:"500000", flowId:"F..."}

[失败路径] ★★★ 最高风险之一
  如果CreditWallet RPC失败（DB异常、网络断连等）:
    用户已支付 → 但余额未增加 → 用户感知"钱没了"

  财务侧责任:
    ├─ 收到失败/超时 → 等2秒 → 重试（钱包幂等，安全重试）
    ├─ 重试仍失败 → 等5秒 → 再重试
    ├─ 三次全失败 → 记录异常状态 → 告警运营
    └─ 运营通过"人工补单"功能手动触发CreditWallet
```

### 5.7 T4: 补偿查询（定时任务，可选）

```
钱包定时任务（每5分钟）
    │
    ├─ [循环] 查询所有status=待支付 且 创建时间超过30分钟的订单
    │
    ├─ [流程] RPC → 财务.GetChannelOrderStatus(channelOrderNo)
    │   └─ 返回: {status=成功/处理中/失败}
    │
    ├─ [判断] 渠道已成功但本地未入账？
    │   ├─ 是 → 异常告警（回调可能丢失），通知运营补单
    │   └─ 否 → 标记超时/继续等待
    │
    └─ [流程] 超时订单标记为"已过期"
```

### 5.8 充值链路总览图

```
C端          钱包           财务          第三方
 │            │              │              │
 │──T0──→    │──RPC O-1──→  │              │
 │           │←─方式列表──   │              │
 │←──────    │              │              │
 │            │              │              │
 │──T1──→    │              │              │
 │           │ 本地校验+建单  │              │
 │           │──RPC O-2──→  │──API──→      │
 │           │←─渠道订单──   │←─支付单──    │
 │←支付信息─ │              │              │
 │            │              │              │
 │──T2────────────────────────────────→     │
 │           │              │   ←─回调──   │
 │           │              │              │
 │           │←─RPC R-1── ★│              │
 │           │ 幂等+事务入账 │              │
 │           │──success──→  │              │
 │            │              │              │
 │←余额更新─ │              │              │
```

---

## 第6章：链路四 — 兑换全链路

### 6.1 链路概述

```
复杂度：★★★☆☆
请求入口：C端点击"兑换"（法币钱包页面）
请求类型：同步、内部闭环、单事务
跨模块依赖：无（纯钱包内部操作）
数据表：exchange_rate + user_balance(×2) + exchange_order + balance_flow(×3)
开发期：P3（烟囱测试用例）
```

### 6.2 完整请求链路

```
═══ T0: 获取汇率预览 ═══

C端 → 钱包 GetExchangePreview(currency=VND)
    │
    ├─ [流程] 查exchange_rate表: "1 BSB = 23,000 VND"
    ├─ [流程] 查兑换赠送规则: 赠送比例 = 5%
    └─ 返回: {rate: "23000", giftRatio: "0.05", multiplier: 3}
    │
    ▼
C端渲染: "当前汇率: 1 BSB = 23,000 VND  赠送5%  需完成3倍流水"

═══ T1: 用户输入金额 + 确认兑换 ═══

C端 → 钱包 CreateExchange(currency=VND, amount=230000)
    │
    ├─ [流程] ★ 重新获取最新汇率（不用T0页面上的旧汇率！）
    │   └─ 原因：用户可能在页面停留很久，汇率已变
    │   └─ 查exchange_rate表: 获取最新值
    │
    ├─ [流程] 计算:
    │   ├─ BSB数量 = 230,000 VND ÷ 23,000 = 10 BSB
    │   ├─ 赠送BSB = 10 × 5% = 0.5 BSB
    │   └─ 精度处理: 按各币种precision截断（不四舍五入）
    │
    ├─ [判断] ★ 余额校验:
    │   └─ VND center.available ≥ 230,000？
    │      └─ 不足 → 返回BALANCE_NOT_ENOUGH → 链路终止
    │
    ├─ [流程] ★★ [BEGIN 事务] — 五步原子操作
    │   │
    │   ├─ Step1: 扣源币种 center钱包
    │   │   UPDATE user_balance SET available = available - 230000
    │   │   WHERE user_id=X AND currency='VND' AND wallet_type=CENTER
    │   │   AND available >= 230000  ← CAS乐观锁
    │   │   └─ affected_rows=0 → 余额不足（并发竞争），回滚 → 链路终止
    │   │
    │   ├─ Step2: 加BSB center钱包（主兑换量）
    │   │   UPDATE user_balance SET available = available + 1000  ← 10 BSB最小单位
    │   │   WHERE user_id=X AND currency='BSB' AND wallet_type=CENTER
    │   │   └─ 如果记录不存在(首次操作BSB)？→ 需自动创建初始余额行
    │   │
    │   ├─ Step3: 加BSB reward钱包（赠送部分）
    │   │   UPDATE user_balance SET available = available + 50  ← 0.5 BSB最小单位
    │   │   WHERE user_id=X AND currency='BSB' AND wallet_type=REWARD
    │   │   └─ 同时记录: 此赠送需完成3倍流水方可提现
    │   │
    │   ├─ Step4: 写exchange_order
    │   │   INSERT exchange_order (from_currency=VND, from_amount=230000,
    │   │   to_currency=BSB, to_amount=10, gift_amount=0.5,
    │   │   rate_used=23000, gift_ratio=0.05, status=成功)
    │   │
    │   └─ Step5: 写3条balance_flow
    │       ├─ 流水1: VND center -230000 (type=EXCHANGE_OUT)
    │       ├─ 流水2: BSB center +10 (type=EXCHANGE_IN)
    │       └─ 流水3: BSB reward +0.5 (type=EXCHANGE_GIFT)
    │
    │   [COMMIT 事务]
    │
    ▼
返回C端: {success, 扣除230000VND, 获得10BSB, 赠送0.5BSB}

[失败路径]
  CAS竞争失败(affected_rows=0) → 事务回滚 → "余额不足，请重试"
  DB事务异常 → 事务回滚 → 余额未变 → "系统异常"
  影响：★ 低（全部在同一事务，要么全成功要么全回滚，无中间状态）
```

### 6.3 关键节点标注

| 节点 | 等级 | 说明 |
|------|------|------|
| 汇率在事务外获取 | ★ | 防止事务内查询拉长事务时间，但要接受"获取汇率和使用汇率之间的微小时间差" |
| 单事务写5张表 | ★★ | 2个user_balance + 1个exchange_order + 3个balance_flow，事务内操作数较多 |
| CAS乐观锁 | ★ | `AND available >= amount` 防止并发超扣，affected_rows=0表示竞争失败 |
| BSB首次创建 | ★ | 用户可能从未有过BSB余额，需要ON DUPLICATE KEY INSERT或事务内先查后插 |

**指导方针**：兑换是纯内部闭环，不涉及任何外部模块RPC，非常适合作为P3期的**烟囱测试用例**。如果兑换链路跑通，说明：余额操作、流水记录、CAS乐观锁、事务管理 四大核心能力都已验证。

---

## 第7章：链路五 — 提现全链路

### 7.1 链路概述

```
复杂度：★★★★★（全系统最高）
请求入口：C端点击"提现"
请求类型：异步跨模块（4个阶段）
跨模块依赖：ser-finance（2个出站RPC + 2个入站回调）+ ser-kyc + ser-user
数据表：withdraw_order + user_balance + balance_flow + withdraw_account
开发期：P4（依赖财务IDL）
核心挑战：冻结模型 + 本地补偿 + 双重补偿策略
```

### 7.2 四阶段全链路时序图

```
时间轴
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0                T1                  T2~T3            T4
进入提现页        创建提现订单          财务审核+出款     结果回调
(毫秒)           (秒级)              (小时~天)        (毫秒)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

角色切换：
  T0: 钱包主导（并行调3个外部服务）
  T1: 钱包主导（冻结+建单+提交审核）← 最复杂，有补偿逻辑
  T2~T3: 财务主导（钱包不参与）
  T4: 财务主导回调（钱包被调，执行最终操作）
```

### 7.3 T0: 进入提现页（4个并行请求）

```
C端 → 钱包 → 并行发起4个请求:

    ┌─ [RPC] ★ ser-kyc.KycQuery(uid)
    │   └─ 返回: kyc_status = VERIFIED / NOT_VERIFIED
    │   └─ [判断] 法币提现 → 必须VERIFIED，否则拦截引导KYC
    │             USDT提现 → 不需要KYC（加密货币匿名）
    │
    ├─ [RPC] 财务.GetPaymentMethods(currency=VND, type=withdraw)
    │   └─ 返回: 全部提现方式候选集
    │   └─ [判断] ★ 本地过滤（合规：资金从哪进就从哪出）
    │       └─ 查deposit_order: 用户历史充值过哪些方式？
    │          ├─ 只充过法币 → 只保留法币提现
    │          ├─ 只充过USDT → 只保留USDT提现
    │          ├─ 都充过 → 只保留法币提现（混合以法币为准）
    │          └─ 没充过 → 规则待定
    │
    ├─ [本地] 查币种配置
    │   └─ 返回: {单笔最低50K, 最高5M, 每日限额10M, 手续费5K, T+1天}
    │
    └─ [本地] 查已保存提现账户
        └─ 返回: {银行名BIDV, 卡号****1234, 户名Nguyen Van A}
        └─ 首次提现则返回空（C端展示表单）
    │
    ▼
C端渲染:
  KYC未认证 → 显示"请先完成实名认证" → 链路终止
  KYC已认证 → 正常展示提现页（方式列表+金额输入+限额提示+账户信息）
```

### 7.4 T1: 创建提现订单（最复杂的单阶段）★★★

```
用户选方式(银行转账) + 输金额(200,000 VND) + 确认账户信息 + 点击"确认提现"
    │
    ▼
C端 → 钱包 CreateWithdrawOrder

═══ 第一段：并行校验（互不依赖，任一失败则拒绝）═══

    ├─ [判断] 并行校验1: 限额
    │   ├─ 200,000 ≥ 50,000(最低)？ → 通过
    │   ├─ 200,000 ≤ 5,000,000(最高)？ → 通过
    │   └─ 今日已提现累计 + 200,000 ≤ 10,000,000(每日限额)？
    │      └─ SELECT COALESCE(SUM(amount),0) FROM withdraw_order
    │         WHERE user_id=X AND status IN (审核中,处理中)
    │         AND DATE(created_at) = TODAY
    │
    ├─ [判断] 并行校验2: 户名一致性
    │   ├─ [RPC] ser-user.GetUserInfo → 获取KYC认证真实姓名
    │   └─ 比对: 提现账户户名 vs KYC姓名（忽略大小写和空格）
    │      └─ 不一致 → "提现账户户名与实名信息不符" → 链路终止
    │
    └─ [判断] 并行校验3: 余额
        └─ VND center.available ≥ 200,000 + 5,000(手续费) = 205,000?
           └─ 不足 → "余额不足" → 链路终止

═══ 全部通过后：第二段串行执行 ═══

    ├─ [流程] ★★ Step1: 冻结余额（关键步骤）
    │   UPDATE user_balance SET
    │     available = available - 205000,
    │     frozen = frozen + 205000
    │   WHERE user_id=X AND currency='VND' AND wallet_type=CENTER
    │     AND available >= 205000  ← CAS乐观锁
    │
    │   affected_rows = 0 → 余额不足（并发竞争），链路终止
    │   affected_rows = 1 → 冻结成功，继续
    │
    │   此时用户看到: available=565K, frozen=205K
    │   钱没走，只是从"可用"移到了"冻结"
    │
    ├─ [流程] Step2: 创建提现订单
    │   INSERT withdraw_order (
    │     order_no = "T" + 16位雪花ID,
    │     user_id, currency=VND, amount=200000, fee=5000,
    │     method=银行转账, status=审核中, account_info=...
    │   )
    │
    ├─ [流程] Step3: 写冻结流水
    │   INSERT balance_flow (type=FREEZE, amount=205000, ...)
    │
    ├─ [流程] Step4: 保存提现账户（首次自动保存）
    │
    └─ [流程] ★★★ Step5: 提交财务审核
        RPC → 财务.SubmitWithdrawalAudit(
          userId, currency=VND, amount=200000, fee=5000,
          methodId, orderNo="T...", accountInfo={...}
        )
        返回: audit_order_no

        ┌──────────────────────────────────────────────────┐
        │ ★★★ 如果RPC失败 → 本地补偿（SAGA简化版）        │
        │                                                  │
        │ 此时状态: 余额已冻结，但财务没收到审核请求        │
        │ 后果: 用户的钱"卡住了"——冻结着，没人审核          │
        │                                                  │
        │ 立即执行补偿:                                    │
        │   ① 解冻: available += 205000, frozen -= 205000  │
        │   ② 订单状态 = 失败                              │
        │   ③ 写balance_flow（冻结回退）                   │
        │   ④ 返回C端: "提现暂不可用，请稍后重试"          │
        │                                                  │
        │ 不需要分布式事务框架 — 本地补偿就够了             │
        └──────────────────────────────────────────────────┘

    │
    ▼
返回C端: "提现申请已提交，等待审核"
```

### 7.5 T2~T3: 财务审核 + 渠道出款（钱包不参与）

```
财务内部流程（钱包只需理解，不需实现）:

  风控审核(1级) → 通过/驳回
      │
      ▼
  财务审核(2级) → 通过/驳回
      │
      ▼
  出款审核(3级) → 通过/驳回
      │
      ▼
  调度渠道出款 → 第三方渠道 → 渠道回调结果
      │
      ▼
  财务获得最终结果 → 回调钱包
```

### 7.6 T4: 结果回调（财务主导，钱包被调）★★★

```
═══ 路径A: 审核全通过 + 出款成功 ═══

财务 → RPC → 钱包.DeductFrozenAmount(
    userId, currency=VND, walletType=CENTER,
    amount=205000, orderNo="T..."
)
    │
    ├─ [判断] 幂等校验: 查订单状态已处理？→ 返回SUCCESS
    ├─ [判断] 订单状态 = 审核中？→ 不是则异常
    ├─ [流程] [BEGIN 事务]
    │   ├─ UPDATE user_balance SET frozen = frozen - 205000
    │   │   注意: 不动available（冻结时已减过）
    │   ├─ UPDATE withdraw_order SET status=成功
    │   └─ INSERT balance_flow (type=DEBIT, opType=WITHDRAW_DEDUCT)
    │ [COMMIT 事务]
    ▼
返回财务: SUCCESS
结果: 冻结金额彻底离开系统，用户银行卡收到200,000 VND

═══ 路径B: 审核驳回 OR 出款失败 ═══

财务 → RPC → 钱包.UnfreezeBalance(
    userId, currency=VND, walletType=CENTER,
    amount=205000, orderNo="T..."
)
    │
    ├─ [判断] 幂等校验
    ├─ [判断] 订单状态校验
    ├─ [流程] [BEGIN 事务]
    │   ├─ UPDATE user_balance SET
    │   │   available = available + 205000,
    │   │   frozen = frozen - 205000
    │   ├─ UPDATE withdraw_order SET status=失败, failReason=...
    │   └─ INSERT balance_flow (type=CREDIT, opType=WITHDRAW_REFUND)
    │ [COMMIT 事务]
    ▼
返回财务: SUCCESS
结果: 冻结的钱退回可用余额

═══ ★★★ 如果回调RPC失败（最高风险） ═══

    出款已成功但DeductFrozenAmount调用失败:
      → 用户的钱已到银行，但系统里frozen还未扣除
      → 对账时会发现不一致

    出款已失败但UnfreezeBalance调用失败:
      → 用户余额持续冻结，既取不出也用不了

    双重补偿策略:
      策略A(主路径): 财务侧重试
        失败 → 等2秒 → 重试 → 再失败 → 等5秒 → 重试 → 三次全失败 → 告警
      策略B(兜底): 钱包定时对账
        每5分钟扫描"审核中"超过N小时的订单
        → 主动查财务获取真实状态 → 本地自愈
```

### 7.7 提现状态转移图

```
用户发起提现
     │
     ▼
 冻结余额成功？
  ├─ 否 → status=失败 (余额不足/并发竞争)
  │
  ▼ 是
 提交审核RPC成功？
  ├─ 否 → ★★★本地补偿解冻 → status=失败
  │
  ▼ 是
 status=审核中 ← 等待财务处理
     │
     ▼
 财务最终结果
  ├─ 通过+出款成功 → DeductFrozenAmount → status=成功 (钱离开系统)
  │
  └─ 驳回/出款失败 → UnfreezeBalance → status=失败 (钱退回可用)
```

### 7.8 提现链路总览图

```
C端          钱包           财务          ser-kyc    第三方
 │            │              │              │          │
 │──T0──→    │              │              │          │
 │           │──RPC O-1──→  │              │          │
 │           │──RPC O-5──────────────────→  │          │
 │           │←─方式+KYC──  │              │          │
 │←──────    │              │              │          │
 │            │              │              │          │
 │──T1──→    │              │              │          │
 │           │ 并行校验      │              │          │
 │           │ 冻结余额 ★★  │              │          │
 │           │ 创建订单      │              │          │
 │           │──提交审核──→  │              │          │
 │           │  (失败→补偿★★★)              │          │
 │←提交成功─ │              │              │          │
 │            │              │              │          │
 │           │              │ 三级审核      │          │
 │           │              │──出款──────────────→     │
 │           │              │   ←──回调──────────      │
 │            │              │              │          │
 │           │←─RPC R-5/R-4★★★            │          │
 │           │ 扣冻结/解冻   │              │          │
 │           │──success──→  │              │          │
 │            │              │              │          │
 │←最终结果─ │              │              │          │
```

---

## 第8章：链路六 — RPC被调与回调链路

### 8.1 链路概述

```
复杂度：★★★☆☆
请求入口：外部模块通过Kitex RPC调用钱包
请求类型：同步RPC服务
调用方：ser-finance / 游戏模块 / 直播模块 / 活动模块
接口数：12个（6个写操作 + 6个读操作）
```

### 8.2 写操作RPC统一链路模式

所有6个写操作RPC（CreditWallet/DebitWallet/FreezeBalance/UnfreezeBalance/DeductFrozenAmount/DebitByPriority）遵循**完全相同的三步模式**：

```
外部模块 → Kitex RPC → ser-wallet RPC Handler
    │
    ├─ Step1 [判断] ★★ 幂等校验
    │   查balance_flow: WHERE ref_order_no=? AND change_type=?
    │   ├─ 已存在 → 直接返回SUCCESS（不重复执行）
    │   └─ 不存在 → 继续
    │
    ├─ Step2 [判断] 参数校验
    │   ├─ 用户存在？
    │   ├─ 币种启用？
    │   ├─ 金额 > 0？
    │   ├─ walletType ∈ {1,2,3,4}？
    │   └─ 任一不满足 → 返回对应错误码
    │
    └─ Step3 [流程] [事务] 余额变动 + 流水 + 订单
        [BEGIN TX]
          ├─ UPDATE user_balance (CAS条件更新)
          ├─ INSERT balance_flow (UNIQUE KEY兜底幂等)
          └─ UPDATE 相关订单状态（如有）
        [COMMIT TX]
    │
    ▼
返回: {success/error, newBalance, flowId}
```

### 8.3 各写操作RPC的差异点

| RPC方法 | 余额操作 | CAS条件 | 关联订单 | 特殊逻辑 |
|---------|---------|---------|---------|---------|
| **CreditWallet** | available += amount | 无（加法无需条件） | deposit_order / 无 | 一个orderNo可能调多次(4个子钱包) |
| **DebitWallet** | available -= amount | available >= amount | 无 | 余额不足返回BALANCE_NOT_ENOUGH |
| **FreezeBalance** | available -= amount, frozen += amount | available >= amount | withdraw_order | 提现发起时冻结 |
| **UnfreezeBalance** | available += amount, frozen -= amount | frozen >= amount | withdraw_order(→失败) | 提现驳回/失败时解冻 |
| **DeductFrozenAmount** | frozen -= amount | frozen >= amount | withdraw_order(→成功) | 出款成功后正式扣除 |
| **DebitByPriority** | center -= X, reward -= Y | 逐级检查 | 无 | ★★ 必须返回deductDetails[] |

### 8.4 DebitByPriority特殊链路

```
投注模块 → RPC → 钱包.DebitByPriority(userId, currency=BSB, amount=100, orderNo=...)
    │
    ├─ [判断] 幂等校验（同上）
    │
    ├─ [流程] ★★ 优先级扣减算法:
    │   ├─ 查center.available = 80 BSB
    │   ├─ 查reward.available = 50 BSB
    │   ├─ 需要扣100 BSB:
    │   │   ├─ 先扣center: min(80, 100) = 80 → center剩0
    │   │   └─ 再扣reward: 100 - 80 = 20 → reward剩30
    │   └─ 生成明细: deductDetails = [{CENTER, 80}, {REWARD, 20}]
    │
    ├─ [流程] [事务]
    │   ├─ UPDATE user_balance SET available -= 80 WHERE wallet_type=CENTER AND available >= 80
    │   ├─ UPDATE user_balance SET available -= 20 WHERE wallet_type=REWARD AND available >= 20
    │   ├─ INSERT balance_flow × 2
    │   [COMMIT]
    │
    ▼
返回: {success, deductDetails:[{CENTER,80},{REWARD,20}], remainingCenter:0, remainingReward:30}

★★ 关键：返回deductDetails是必须的！
    投注返奖时需要按比例分配：
      center贡献80% → 返奖到center
      reward贡献20% → 返奖到reward
    如果不返回明细 → 无法正确返奖 → 资金错误
```

### 8.5 读操作RPC链路

6个读操作RPC（QueryWalletBalance/GetCurrencyConfig/GetExchangeRate/ConvertAmount/BatchQueryBalance/GetWalletFlowList）的链路更简单：

```
外部模块 → RPC → ser-wallet
    │
    ├─ [判断] 参数校验（userId/currencyCode有效？）
    │
    ├─ [流程] 查询数据
    │   ├─ QueryWalletBalance → 直查DB user_balance（不缓存）
    │   ├─ GetCurrencyConfig → 优先Redis缓存，miss则查DB
    │   ├─ GetExchangeRate → 查DB exchange_rate
    │   ├─ ConvertAmount → 查rate + 计算 + 精度处理
    │   ├─ BatchQueryBalance → 批量查DB（限100用户）
    │   └─ GetWalletFlowList → 分页查DB balance_flow
    │
    ▼
返回: 查询结果（无事务、无幂等需求）
```

### 8.6 外部模块回调钱包的典型序列

```
═══ 场景1: 充值入账（最高频回调）═══

财务 → CreditWallet(orderNo, CENTER, amount, DEPOSIT)
  可能紧接着:
财务 → CreditWallet(orderNo, REWARD, bonus, BONUS)
  注意: 同一orderNo调两次，但walletType不同，幂等键不冲突

═══ 场景2: 人工加减款（B端操作触发）═══

财务B端审核通过后:
财务 → QueryWalletBalance(userId) ← 先查余额展示
财务 → CreditWallet(orderNo, CENTER, 70, MANUAL_CREDIT)
财务 → CreditWallet(orderNo, REWARD, 30, MANUAL_CREDIT)
  注意: 一笔加款最多调4次（center/reward/agent/anchor各一次）

═══ 场景3: 提现流程三步曲 ═══

T1: 钱包内部 → FreezeBalance(orderNo, CENTER, amount+fee)
    ...等待数小时~天...
T4a: 财务 → DeductFrozenAmount(orderNo, CENTER, amount+fee)
  或
T4b: 财务 → UnfreezeBalance(orderNo, CENTER, amount+fee)

═══ 场景4: 投注扣减+返奖 ═══

游戏模块 → DebitByPriority(userId, BSB, 100, betOrderNo)
  返回: deductDetails=[{CENTER,80},{REWARD,20}]
    ...比赛结束...
游戏模块 → CreditWallet(returnOrderNo, CENTER, 80*2.0, RETURN_AWARD)
游戏模块 → CreditWallet(returnOrderNo, REWARD, 20*2.0, RETURN_AWARD)
```

---

## 第9章：跨链路共性模式提炼

### 9.1 模式总览

```
7大共性模式跨所有链路:

模式1: 幂等→校验→事务        ← 所有写操作的三板斧
模式2: RPC外+DB内            ← 事务边界铁律
模式3: CAS乐观锁             ← 所有余额扣减的并发安全
模式4: 本地补偿               ← 跨模块失败后的回退
模式5: 并行校验→串行执行       ← 提现等多校验场景
模式6: 双向RPC               ← 钱包既调也被调
模式7: 订单+余额+流水同事务    ← 数据一致性三件套
```

### 9.2 模式1：幂等→校验→事务（三板斧）

```
定义: 每个写操作都按三步走——先查是否做过，再验参数合法，最后一个事务搞定

  Step1 幂等: 查balance_flow表 (orderNo, changeType)
    → 做过了 → 直接返回成功
    → 没做过 → 下一步

  Step2 校验: 用户存在? 币种启用? 金额>0? 余额够?
    → 不满足 → 返回具体错误码
    → 满足 → 下一步

  Step3 事务: [BEGIN TX] 改余额 + 写流水 + 改订单 [COMMIT TX]
    → DB唯一键兜底（第二层幂等）

出现在: CreditWallet / DebitWallet / Freeze / Unfreeze / DeductFrozen / DebitByPriority / 兑换
不出现在: 读操作、币种配置CRUD（无幂等需求）
```

### 9.3 模式2：RPC在事务外，DB在事务内

```
定义: 绝不允许在数据库事务内发起RPC调用

正确:
  ① 获取数据（RPC调用、配置查询）    ← 事务外
  ② 计算和验证                       ← 事务外
  ③ [BEGIN TX] DB操作 [COMMIT TX]     ← 事务内
  ④ 返回结果                         ← 事务外

错误:
  ① [BEGIN TX]
  ② RPC调用（如果超时3秒，事务持续持有锁）  ← 严禁！
  ③ DB操作
  ④ [COMMIT TX]

为什么: 事务内RPC → 超时 → 锁不释放 → 连接池耗尽 → 全系统阻塞
出现在: 兑换(先取汇率再事务)、提现(先查KYC/限额再冻结事务)、充值入账(无外部调用，直接事务)
```

### 9.4 模式3：CAS乐观锁

```
定义: 余额扣减操作使用"条件更新"，SQL中加 AND available >= amount

  UPDATE user_balance SET available = available - ?
  WHERE user_id = ? AND currency_code = ? AND wallet_type = ?
    AND available >= ?  ← 这就是CAS条件

  affected_rows = 1 → 成功
  affected_rows = 0 → 余额不足或并发竞争

为什么: 不用悲观锁(SELECT FOR UPDATE)，因为乐观锁在低冲突场景下性能更好
重试策略: 一般不重试（余额不足就是不足），直接返回BALANCE_NOT_ENOUGH
出现在: DebitWallet / FreezeBalance / DebitByPriority / 兑换(扣源币种)
不出现在: CreditWallet / UnfreezeBalance（加法操作，不需要条件）
```

### 9.5 模式4：本地补偿

```
定义: 跨模块操作中，本地操作已成功但远程RPC失败时，立即回退本地操作

典型场景:
  提现 → 冻结成功 → 提交审核RPC失败
    → 立即解冻（本地补偿）
    → 不需要分布式事务框架

为什么不用分布式事务:
  ① 项目规模不需要（Seata等框架太重）
  ② 所有跨模块操作都是"一方主导"模式
  ③ RPC接口都是幂等的，重试安全

出现在: 提现(T1 Step5失败) — 目前唯一需要本地补偿的场景
不出现在: 充值(先支付后入账，本地无需先操作)、兑换(无跨模块)
```

### 9.6 模式5：并行校验→串行执行

```
定义: 多个校验条件互不依赖时并行执行（提升性能），全部通过后串行执行写操作

  [goroutine 1] 限额校验    ──┐
  [goroutine 2] 户名校验    ──┤ 全部完成
  [goroutine 3] 余额校验    ──┘
                               │
                               ▼ 任一失败 → 聚合错误返回
                               ▼ 全部通过 → 串行执行
                               │
  Step1: 冻结余额              │
  Step2: 创建订单              │  必须串行（有先后依赖）
  Step3: 提交审核              │

出现在: 提现T1（3个并行校验）、提现T0（4个并行请求）
不出现在: 充值（校验较少，串行即可）、兑换（校验简单）
```

### 9.7 模式6：双向RPC

```
定义: 钱包既是RPC调用方（client），也是RPC服务方（server）

作为client（出站）:
  → 财务.GetPaymentMethods
  → 财务.MatchAndCreateChannelOrder
  → 财务.GetChannelOrderStatus
  → ser-kyc.KycQuery
  → ser-user.GetUserInfo
  → ser-blog.AddActLog

作为server（入站）:
  ← 财务.CreditWallet
  ← 财务.DebitWallet
  ← 财务.FreezeBalance / UnfreezeBalance / DeductFrozenAmount
  ← 财务.QueryWalletBalance
  ← 游戏.DebitByPriority
  ← 多模块.GetCurrencyConfig / GetExchangeRate / ...

含义: IDL设计时，钱包不仅要定义自己暴露的RPC，还要依赖财务定义的RPC
```

### 9.8 模式7：订单+余额+流水同事务

```
定义: 每笔资金操作，三类数据必须在同一事务中更新

  [BEGIN TX]
    ① 更新 user_balance（余额变动）
    ② 插入 balance_flow（流水记录，含before/after快照）
    ③ 更新 xxxxx_order（订单状态）
  [COMMIT TX]

为什么: 防止出现"余额变了但流水没写"或"订单状态更新了但余额没变"
验证: balance_flow.balance_after 应该等于当前 user_balance.available（连续性校验）
出现在: 所有写操作（充值入账、提现冻结/扣除/解冻、兑换、人工加减款）
```

---

## 第10章：关键节点复杂度与挑战全景标注

### 10.1 按风险等级的关键节点总表

#### 致命级 ★★★（失败 = 资金不一致 / 用户资金损失感知）

| 节点 | 所在链路 | 风险描述 | 应对策略 |
|------|---------|---------|---------|
| 充值回调CreditWallet失败 | 链路3 T3 | 用户已付款但余额未增 | 财务重试+人工补单 |
| 提现回调DeductFrozen/Unfreeze失败 | 链路5 T4 | 余额持续冻结 | 财务重试+钱包定时对账 |
| 提现冻结后审核RPC失败 | 链路5 T1 | 余额冻结但无人审核 | 本地补偿立即解冻 |

#### 严重级 ★★（失败 = 系统行为不正确 / 数据不一致）

| 节点 | 所在链路 | 风险描述 | 应对策略 |
|------|---------|---------|---------|
| CAS乐观锁竞争 | 链路4/5/6 | 并发操作同一用户余额 | affected_rows=0判断+返回错误 |
| 事务边界违规 | 所有写链路 | RPC在事务内导致长事务 | 代码审查：RPC必须在事务外 |
| 幂等失效 | 链路6(所有RPC) | 重复调用导致重复入账/扣款 | 双层幂等：应用层+DB唯一键 |
| DebitByPriority不返回明细 | 链路6 | 返奖比例无法计算 | 接口设计必须包含deductDetails |

#### 重要级 ★（失败 = 体验/合规问题 / 可人工修复）

| 节点 | 所在链路 | 风险描述 | 应对策略 |
|------|---------|---------|---------|
| 重复转账拦截策略 | 链路3 T1 | 三级拦截逻辑需精确实现 | 按需求文档严格实现三级 |
| 汇率在事务外获取 | 链路4 T1 | 获取和使用之间汇率可能变 | 时间窗口极短（毫秒级），可接受 |
| 提现方式过滤(合规) | 链路5 T0 | 混合充值来源的过滤规则 | 待确认合规规则后实现 |
| BSB首次创建余额行 | 链路4 T1 | 兑换时可能是首次操作BSB | ON DUPLICATE KEY INSERT处理 |
| 精度字段不可变 | 链路1 | 上线后修改精度导致数据全错 | 代码层面禁止修改+前端隐藏编辑 |

### 10.2 链路复杂度对比表

| 维度 | 链路1 币种配置 | 链路2 余额查询 | 链路3 充值 | 链路4 兑换 | 链路5 提现 | 链路6 RPC被调 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| **跨模块RPC数** | 1(审计) | 0 | 3 | 0 | 4 | 0(被调方) |
| **事务操作数** | 1(单表) | 0 | 3(余额+流水+订单) | 5(2余额+1订单+3流水) | 3+补偿 | 3 |
| **异常路径数** | 1 | 1 | 4 | 2 | 7 | 3 |
| **异步环节** | 无 | 无 | 有(T2~T3) | 无 | 有(T2~T4) | 无 |
| **补偿机制** | 无 | 无 | 人工补单 | 无 | 本地+双重 | 无 |
| **耗时** | ms | ms | min~hr | ms | hr~day | ms |
| **总复杂度** | ★☆☆☆☆ | ★★☆☆☆ | ★★★★☆ | ★★★☆☆ | ★★★★★ | ★★★☆☆ |

### 10.3 开发优先级建议（从链路角度）

```
P2期: 链路1(币种配置) + 链路2(余额查询)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  目标: 验证四层架构骨架、DB连接、缓存策略
  依赖: 零外部依赖
  验收: 可通过B端创建币种、C端查看余额
  产出: currency_config CRUD + user_balance 查询 + Redis Cache-Aside
  关键: 精度字段设计一步到位（不可修改）

P3期: 链路4(兑换) + 链路6(RPC暴露，部分)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  目标: 验证核心能力（CAS/事务/流水/幂等）
  依赖: 链路1完成（币种配置+汇率存在）
  验收: 烟囱测试——VND→BSB兑换端到端通过
  产出: 兑换Service + user_balance写操作 + balance_flow + CAS乐观锁
        + RPC暴露IDL（CreditWallet/DebitWallet/QueryBalance等）
  关键: 兑换成功 = 证明余额操作、流水记录、事务管理全部OK

P4期: 链路3(充值) + 链路5(提现) + 链路6(完整RPC被调)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  目标: 与财务模块集成，完成充提全链路
  依赖: 财务IDL已交付 + P3核心能力已验证
  验收: 充值端到端 + 提现端到端 + 异常补偿验证
  产出: deposit_order + withdraw_order + 财务RPC Client + 冻结模型
        + 本地补偿 + 定时对账
  关键: 冻结三步模型(Freeze→Deduct/Unfreeze)是全系统最复杂的逻辑
  阻塞: 财务IDL交付时间是最大外部阻塞因素
```

### 10.4 每条链路开发前的必确认Checklist

```
━━ 链路1(币种配置) 开发前确认 ━━
  □ 5个币种的精度位数最终值（VND:0? IDR:0? THB:2? USDT:6? BSB:2?）
  □ currency_config表字段是否完整
  □ Redis Key命名规范 + TTL时间
  □ ser-blog审计日志的IDL可用

━━ 链路2(余额查询) 开发前确认 ━━
  □ user_balance表结构确认（复合主键设计）
  □ 首次查询时余额行不存在怎么处理（返回0 vs 自动创建）
  □ 4个子钱包类型枚举值确认

━━ 链路4(兑换) 开发前确认 ━━
  □ 汇率值的存储精度和计算公式
  □ 兑换赠送比例的配置位置
  □ BSB余额行首次自动创建的策略
  □ exchange_order表字段确认

━━ 链路3(充值) 开发前确认 ━━
  □ 财务.GetPaymentMethods IDL已就绪
  □ 财务.MatchAndCreateChannelOrder IDL已就绪
  □ 充值奖金活动归属确认（钱包/财务/独立）
  □ 充值订单归属确认（双方各建 vs 只一方建）
  □ USDT充值地址模型确认
  □ 充值回调方式确认（直接RPC vs MQ vs Webhook）

━━ 链路5(提现) 开发前确认 ━━
  □ 上述链路3的全部确认项
  □ InitiateChannelPayout调用归属确认
  □ 提现手续费方向确认（额外扣 vs 金额内扣）
  □ 混合充值来源提现过滤规则确认
  □ 财务审核提交接口IDL确认
  □ 提现回调方式确认
```

---

## 附录：链路速查卡片

### A.1 六条链路一句话总结

| 链路 | 一句话 | 起点 | 终点 | 核心挑战 |
|------|--------|------|------|---------|
| 1-币种配置 | B端CRUD币种参数，删缓存，写日志 | B端操作 | DB+Cache | 精度不可变 |
| 2-余额查询 | 并行查币种(缓存)+余额(实时)，合并返回 | C端打开钱包 | UI渲染 | 余额不缓存 |
| 3-充值 | 拿方式→建订单→用户付→财务回调入账 | C端"充值" | 余额增加 | 回调失败=用户已付但未到账 |
| 4-兑换 | 取汇率→算金额→单事务扣A加B写流水 | C端"兑换" | 余额即时变 | CAS+单事务5表 |
| 5-提现 | 4并行查→冻结→提交审核→等回调→扣/退 | C端"提现" | 钱离开系统 | 冻结模型+双重补偿 |
| 6-RPC被调 | 幂等查→校验→事务(余额+流水+订单) | 外部RPC | 余额变动 | 双层幂等+CAS |

### A.2 核心数字速查

| 维度 | 数字 |
|------|------|
| 业务链路总数 | 6条 |
| 跨模块RPC出站 | 4个(全部调ser-finance) + ~5个(kyc/user/blog/buser/socket) |
| 跨模块RPC入站 | 12个(6写+6读) |
| 致命级风险节点 | 3个 |
| 严重级风险节点 | 4个 |
| 重要级风险节点 | 5个 |
| 共性设计模式 | 7种 |
| 开发分期 | 3期(P2/P3/P4) |

### A.3 请求链路核心思维总结

```
一切请求处理 = 流程 + 判断 + 循环

  流程: 做一件事 → 创建订单、更新余额、写流水、调RPC
  判断: 决定做不做 → 幂等检查、余额校验、限额校验、KYC校验
  循环: 反复做 → 定时任务对账、CAS重试、财务重试回调

掌握了这三种模式在每条链路中的分布，就掌握了整个系统的运行逻辑。
```

---

> **文档结束**
>
> 本文档共覆盖：10章 + 1附录
> - 6条完整业务链路时序图
> - 12个关键风险节点（3致命+4严重+5重要）
> - 7种跨链路共性模式
> - 4期开发优先级建议 + 开发前Checklist
> - 每步标注：流程/判断/循环 + ★风险等级
