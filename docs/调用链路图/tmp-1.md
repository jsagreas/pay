# 钱包模块宏观调用链路分析

> 编写目的：屏蔽代码细节，从"外部观测者"视角，完整呈现每个关键操作的请求流转全貌。
> 每个步骤标注：**所在位置** → **发生了什么** → **流向哪里** → **复杂度/挑战标记**
> 复杂度标记：🟢 简单 | 🟡 中等 | 🔴 复杂/关键 | ⚡ 存在挑战

---

## 目录

- [第一章 基础设施链路——所有请求的"高速公路"](#第一章)
- [第二章 跨模块交互拓扑——谁调用谁](#第二章)
- [第三章 C端余额查询链路](#第三章)
- [第四章 C端币种列表查询链路](#第四章)
- [第五章 充值完整链路（20步）](#第五章)
- [第六章 提现完整链路（21步）](#第六章)
- [第七章 兑换完整链路（内部闭环）](#第七章)
- [第八章 冻结三阶段（TCC模式）链路](#第八章)
- [第九章 人工操作链路（补单/加款/减款）](#第九章)
- [第十章 汇率定时刷新链路](#第十章)
- [第十一章 B端币种配置管理链路](#第十一章)
- [第十二章 钱包初始化链路](#第十二章)
- [第十三章 RPC暴露接口被调用链路](#第十三章)
- [第十四章 关键步骤复杂度/挑战全景矩阵](#第十四章)
- [第十五章 链路级故障传播与降级策略](#第十五章)
- [附录A 完整调用链路ASCII总图](#附录a)
- [附录B 各链路耗时预估与瓶颈分析](#附录b)

---

<a id="第一章"></a>
## 第一章 基础设施链路——所有请求的"高速公路"

> 无论是充值、提现、查余额，每个HTTP请求都必须经过这条"高速公路"。
> 理解它，就理解了所有请求的共同起点和终点。

### 1.1 C端请求完整基础链路

```
┌──────────────────────────────────────────────────────────────────┐
│                    C端用户 (App/H5/Web)                          │
└───────────────────────────┬──────────────────────────────────────┘
                            │ HTTP POST /api/wallet/xxx
                            │ Headers: X-User-Token, X-User-Language,
                            │          X-User-Device-ID, X-User-Device-Type
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ gate-font (Hertz HTTP网关)                                       │
│                                                                  │
│ ① IGlTracer.Start()                                    🟢       │
│    → 生成UUID作为 X-Trace-Id                                     │
│    → 设置CORS响应头 (Allow-Origin: *)                             │
│    → 写入 metainfo.WithPersistentValue(ctx, "X-Trace-Id", uuid) │
│                                                                  │
│ ② SetHeaderInfo()                                       🟢       │
│    → 从HTTP Header提取: X-User-Language, X-User-Device-ID,       │
│      X-User-Device-Type                                          │
│    → 从HTTP Header或Cookie提取: X-User-Token                     │
│    → 从连接信息提取: X-Client-IP (c.ClientIP())                   │
│    → 全部写入 metainfo 持久化上下文                                │
│                                                                  │
│ ③ BlackCheck()                                          🟡       │
│    → RPC调用 ser-ip.OnBlackList(ctx)                              │
│    → 命中黑名单 → 返回 BizErr(IpInBlackTable)                     │
│    → 调用失败 → 放行 (fail-open策略，仅warn日志)                   │
│                                                                  │
│ ④ FontAuthMiddleware()                                  🟡       │
│    → 从context取出 X-User-Token                                  │
│    → Redis查询: GET redis_token_key:{token}                       │
│    → 反序列化为 CUserTokenPayload {ID, NickName}                  │
│    → ID=0 → 返回 BizErr(TokenAnalysisFailed)                     │
│    → 写入context: X-User-ID, X-User-Name                         │
│                                                                  │
│ ⑤ rpc.Handler[REQ, RESP]()                              🟢       │
│    → 反射初始化请求结构体                                          │
│    → c.BindAndValidate(req) 绑定JSON + 校验tag                   │
│    → 调用RPC方法: rpcMethod(ctx, req)                             │
│    → 返回: c.JSON(ret.HttpOne(ctx, resp, I18nClient(), err))     │
└───────────────────────────┬──────────────────────────────────────┘
                            │ Kitex RPC (TTHeader协议)
                            │ 携带完整 metainfo 上下文:
                            │   X-Trace-Id, X-User-ID, X-User-Token,
                            │   X-Client-IP, X-User-Language,
                            │   X-User-Device-ID, X-User-Device-Type,
                            │   X-User-Name
                            │ 超时: 3秒
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ ser-wallet (Kitex RPC服务)                                       │
│                                                                  │
│ ⑥ RPC Server中间件                                      🟢       │
│    → transmeta.ServerTTHeaderHandler 接收元数据                    │
│    → 日志中间件记录: <TraceID> package.service.method              │
│      | cost=耗时 | err=错误 | from:来源IP:来源服务.方法             │
│                                                                  │
│ ⑦ WalletServiceImpl.XxxMethod(ctx, req)                          │
│    → 委托给具体Service方法 (薄代理层)                               │
│                                                                  │
│ ⑧ Service层业务逻辑 (核心)                               🔴       │
│    → 参数校验 → 业务校验 → 数据操作 → 缓存 → 审计日志              │
│    → 详见各链路章节                                               │
│                                                                  │
│ ⑨ Repository层数据持久化                                 🟡       │
│    → GORM Gen查询构建器                                           │
│    → 事务管理: query.Q.Transaction(func(tx) error)                │
│    → SELECT FOR UPDATE 行锁                                      │
│    → INSERT-ONLY流水写入                                          │
│                                                                  │
│ ⑩ Cache层缓存操作                                       🟢       │
│    → Redis读写: Get/Set/Del                                       │
│    → 空结果缓存防穿透                                             │
│    → 写操作后主动失效                                             │
└───────────────────────────┬──────────────────────────────────────┘
                            │ 响应沿原路返回
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ gate-font 响应封装                                                │
│                                                                  │
│ ⑪ ret.HttpOne(ctx, resp, I18nClient(), err)              🟢      │
│    → 成功: {traceId: "uuid", code: 0, msg: "", data: {...}}     │
│    → 失败: 提取 BizStatusError 错误码                             │
│           → RPC调用 ser-i18n.LangTransOne(ctx, code)              │
│           → {traceId: "uuid", code: 6015xxx, msg: "翻译后消息"}   │
│    → 永远返回 HTTP 200                                            │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 B端请求基础链路差异

```
B端用户 (管理后台)
    │ HTTP POST /admin/api/wallet/currency/xxx
    ▼
gate-back (Hertz HTTP网关)
    │
    ├─ ① IGlTracer.Start()        — 同C端
    ├─ ② SetHeaderInfo()           — 同C端
    ├─ ③ WhiteCheck()              — 【差异】IP白名单校验 (fail-closed)
    │     → 不在白名单 → 拒绝: BizErr(IpNotInWhiteTable)
    │     → 调用失败 → 拒绝 (更严格)
    ├─ ④ BackAuthMiddleware()      — 【差异】后台Token + URL权限校验
    │     → Redis查询后台token → BUserTokenPayload {ID, UserName}
    │     → RPC: ser-buser.CheckUrlAuth(ctx, c.FullPath())
    │     → URL权限不通过 → BizErr(CheckUrlAuthFailed)
    └─ ⑤ rpc.Handler()            — 同C端
        │
        ▼
    ser-wallet RPC → 同C端后续流程
```

**B端 vs C端关键差异汇总：**

| 环节 | C端 (gate-font) | B端 (gate-back) |
|------|-----------------|-----------------|
| IP检查 | BlackCheck (黑名单, fail-open) | WhiteCheck (白名单, fail-closed) |
| 认证 | FontAuth (用户token) | BackAuth (管理员token + URL权限) |
| 路由前缀 | `/api/wallet/` | `/admin/api/wallet/` |
| 请求体限制 | 默认 | 20MB (支持文件上传) |

### 1.3 RPC调用基础链路（服务间调用）

```
ser-finance / ser-betting / 活动系统 (调用方)
    │ Kitex RPC 直连调用
    │ ETCD服务发现 → 解析ser-wallet地址
    │ TTHeader协议传递元数据
    │ 超时: 3秒 (调用方配置)
    ▼
ser-wallet (Kitex RPC服务)
    │
    ├─ ServerTTHeaderHandler 接收元数据
    ├─ 日志中间件记录调用信息
    ├─ WalletServiceImpl.XxxMethod(ctx, req)
    │   → 直接进入Service层业务逻辑
    │   → 无HTTP层的Bind/Validate（Thrift IDL已保证类型）
    └─ 返回: (resp, err) 或 (nil, BizErr)
```

**关键区别：** RPC调用不经过HTTP网关，直接进入Service层。参数校验由Thrift IDL定义保证类型安全，但业务校验（如金额>0、余额充足）仍需Service层处理。

---

<a id="第二章"></a>
## 第二章 跨模块交互拓扑——谁调用谁

### 2.1 全局模块交互图

```
                         ┌─────────────┐
                         │   C端用户    │
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                    ┌────│  gate-font  │────┐
                    │    └─────────────┘    │
                    │                       │
            ┌───────▼───────┐       ┌───────▼───────┐
            │  ser-wallet   │       │  其他C端服务   │
            │  (我方核心)    │       │  (投注/直播等)  │
            └───┬───┬───┬──┘       └───────────────┘
                │   │   │
    ┌───────────┤   │   ├───────────┐
    │           │   │   │           │
    ▼           ▼   │   ▼           ▼
┌────────┐ ┌──────┐│┌───────┐ ┌────────┐
│ser-user│ │ser-  │││ser-s3 │ │ser-blog│
│(2接口) │ │kyc   │││(1接口)│ │(2接口) │
│用户状态 │ │(1接口│││图标上传│ │审计日志 │
│批量查询 │ │KYC  │││       │ │日志查询 │
└────────┘ └──────┘│└───────┘ └────────┘
                   │
                   ▼
            ┌──────────────┐
            │ ser-finance  │◄───── 三方支付渠道
            │ (支付/财务)   │
            │ 6个待开发接口  │
            └──────┬───────┘
                   │
                   │ 回调我方RPC
                   ▼
            ┌──────────────┐
            │  ser-wallet  │
            │ 12个RPC暴露   │
            └──────────────┘
```

### 2.2 调用方向明细

```
                    ┌─────────────────────────────────┐
                    │         ser-wallet               │
                    │     (我方，钱包+币种)              │
                    └─────────┬───────────────────────┘
                              │
        ╔═════════════════════╪═════════════════════════╗
        ║   我方主动调出 (7)   │   外部调入我方 (12+)     ║
        ╠═════════════════════╪═════════════════════════╣
        ║                     │                         ║
        ║  ser-user ──────────┤──── ser-finance         ║
        ║  ├ GetUserInfoExpend│     ├ CreditWallet      ║
        ║  └ BatchGetUserInfo │     ├ DebitWallet        ║
        ║                     │     ├ FreezeBalance      ║
        ║  ser-kyc ───────────┤     ├ UnfreezeBalance    ║
        ║  └ KycQuery         │     ├ ConfirmDebit       ║
        ║                     │     ├ GetBalance         ║
        ║  ser-blog ──────────┤     ├ BatchGetBalance    ║
        ║  ├ AddActLog(oneway)│     └ GetWalletFlow      ║
        ║  └ QueryActs        │                         ║
        ║                     │──── ser-finance(币种)    ║
        ║  ser-s3 ────────────┤     ├ GetCurrencyList    ║
        ║  └ UploadFile       │     ├ GetExchangeRate    ║
        ║                     │     └ GetBaseCurrency    ║
        ║  ser-buser ─────────┤                         ║
        ║  └ QueryUserByIds   │──── 投注/结算/活动系统   ║
        ║                     │     ├ CreditWallet(返奖) ║
        ║  ser-finance(TODO)──┤     └ DebitWallet(消费)  ║
        ║  ├ GetDepositMethods│                         ║
        ║  ├ GetWithdrawMethods│──── ser-user(注册)      ║
        ║  ├ GetWithdrawLimits│     └ InitWallet         ║
        ║  ├ SubmitWithdrawOrder                        ║
        ║  ├ CreateChannelOrder                         ║
        ║  └ QueryChannelOrder│                         ║
        ╚═════════════════════╧═════════════════════════╝
```

### 2.3 数据流向分类

| 分类 | 方向 | 数量 | 特征 |
|------|------|------|------|
| C端HTTP入口 | 用户→gateway→wallet | 19+2 | 查询为主，少量写操作 |
| B端HTTP入口 | 管理员→gateway→wallet | 7 | 配置管理操作 |
| 外部RPC调入 | finance/betting→wallet | 12 | 资金写操作为主，幂等性要求高 |
| 我方RPC调出 | wallet→user/kyc/blog/s3 | 7 | 已有服务，可直接调用 |
| 待开发RPC调出 | wallet→finance | 6 | 阻塞项，需要先Mock |
| 异步推送 | wallet→websocket | 1 | 余额变更推送 |
| 定时任务 | cron→wallet内部 | 1 | 汇率刷新 |

---

<a id="第三章"></a>
## 第三章 C端余额查询链路

> 最高频操作，每次打开App首页、钱包页、投注页都会触发。
> 必须快，必须准。

### 3.1 完整链路图

```
C端用户打开钱包页
    │
    │ GET /api/wallet/balance
    ▼
gate-font ──────────────────────────────────────────
    │ ① Tracer生成TraceId                    🟢
    │ ② SetHeaderInfo提取上下文               🟢
    │ ③ BlackCheck IP黑名单校验               🟡
    │ ④ FontAuth Token校验 → 提取userId       🟡
    │ ⑤ Handler绑定参数(无参或仅currencyCode)  🟢
    │
    │ RPC调用 ser-wallet.GetBalance
    ▼
ser-wallet ─────────────────────────────────────────
    │
    │ ⑥ 从context提取userId                  🟢
    │
    │ ⑦ 查Redis缓存                          🟢
    │    key = wallet:balance:{userId}:{currencyCode}
    │    ├── 命中 → 直接返回 ─────────────→ 跳到⑪
    │    └── 未命中 → 继续↓
    │
    │ ⑧ 查DB: wallet_account表               🟡
    │    SELECT * FROM wallet_account
    │    WHERE user_id = ? AND currency_code = ?
    │    → 返回5个子钱包记录:
    │      center(可用+冻结), reward(可用+冻结),
    │      streamer, agent(预留), venue(预留)
    │
    │ ⑨ 聚合计算                              🟢
    │    total_balance = Σ(available_amt + frozen_amt)
    │    available_balance = Σ(available_amt)
    │    frozen_balance = Σ(frozen_amt)
    │
    │ ⑩ 写入Redis缓存                        🟢
    │    TTL = 30秒 (短TTL，写操作主动失效)
    │
    │ ⑪ 构造响应                              🟢
    │    {centerBalance, rewardBalance, streamerBalance,
    │     totalAvailable, totalFrozen, currencyCode}
    │
    ▼ 返回
gate-font ──────────────────────────────────────────
    │ ⑫ ret.HttpOne包装响应                   🟢
    │    {traceId, code:0, msg:"", data:{...}}
    ▼
C端用户看到余额
```

### 3.2 关键步骤分析

| 步骤 | 复杂度 | 说明 |
|------|--------|------|
| ④ Token校验 | 🟡 | Redis查询，token过期=登录失效，用户需重新登录 |
| ⑦ Redis缓存 | 🟢 | 读操作首选缓存，30s TTL兜底 |
| ⑧ DB查询 | 🟡 | 单用户最多5条记录，性能不是问题；关键是索引 `(user_id, currency_code)` |
| ⑨ 聚合计算 | 🟢 | 内存计算，无IO |

**挑战点：**
- ⚡ **缓存一致性**：写操作（充值/提现/兑换）后必须主动失效缓存，否则用户看到旧余额
- ⚡ **并发读写**：查余额和充值入账同时发生，可能读到中间状态 → 30s短TTL + 写后失效双保障
- ⚡ **精度展示**：i64最小单位传输 → 前端按currency_config.decimal还原展示

### 3.3 预估耗时

```
总耗时(缓存命中): ~15ms
  gate-font中间件链: ~5ms (tracer+header+blackcheck+auth)
  RPC网络传输: ~2ms (内网)
  Redis GET: ~1ms
  响应封装: ~1ms
  网络回传: ~2ms

总耗时(缓存未命中): ~25ms
  gate-font中间件链: ~5ms
  RPC网络传输: ~2ms
  Redis GET miss: ~1ms
  DB查询: ~5ms (索引命中)
  Redis SET: ~1ms
  响应封装: ~1ms
  网络回传: ~2ms
```

---

<a id="第四章"></a>
## 第四章 C端币种列表查询链路

> 用户选择币种时触发，频率次于余额查询。
> 数据变更极低频（B端配置），重度依赖缓存。

### 4.1 完整链路图

```
C端用户打开币种选择器
    │
    │ GET /api/wallet/currencies
    ▼
gate-font → 中间件链(同第三章①-⑤)
    │
    │ RPC调用 ser-wallet.GetCurrencies
    ▼
ser-wallet ─────────────────────────────────────────
    │
    │ ⑥ 查Redis缓存                          🟢
    │    key = wallet:currency:list:enabled
    │    ├── 命中 → 直接返回 ─────────────→ 跳到⑩
    │    └── 未命中 → 继续↓
    │
    │ ⑦ 查DB: currency_config表              🟢
    │    SELECT * FROM currency_config
    │    WHERE status = 1 (已启用)
    │    ORDER BY sort_value ASC
    │    → 返回: [VND, IDR, THB, USDT, BSB, ...]
    │
    │ ⑧ 查询用户当前选中币种                    🟢
    │    (可能存在user_preference或默认币种逻辑)
    │
    │ ⑨ 写入Redis缓存                        🟢
    │    TTL = 5分钟 (变更极低频)
    │
    │ ⑩ 构造响应                              🟢
    │    [{code:"VND", name:"越南盾", symbol:"₫",
    │      decimal:0, icon:"url", selected:true},
    │     {code:"IDR", ...}, ...]
    │
    ▼ 返回
gate-font → ret.HttpOne → 响应
```

### 4.2 关键步骤分析

**整条链路复杂度极低(🟢)** — 纯粹的配置读取。

**唯一挑战点：**
- ⚡ **缓存失效时机**：B端修改币种配置后，必须同步清除C端缓存
- ⚡ **RPC调用场景**：ser-finance等模块也会RPC调用GetCurrencyList，需保证缓存对RPC调用同样生效

---

<a id="第五章"></a>
## 第五章 充值完整链路（20步）

> 跨越4个系统的长链路：C端 → ser-wallet → ser-finance → 三方支付 → ser-finance回调 → ser-wallet入账
> 这是最复杂的链路之一，涉及跨模块协作、异步回调、最终一致性。

### 5.1 完整链路图（阶段式）

```
════════════════ 阶段一：用户发起充值 (C端→ser-wallet→ser-finance) ════════════════

C端用户点击"充值"
    │
    │ GET /api/wallet/deposit/methods
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 1: 接收请求，提取 userId, currencyCode    🟢   │
│  Step 2: RPC调用 ser-finance.GetDepositMethods  🟡⚡ │
│          → (currencyCode, userId)                     │
│          → 返回: 可用充值方式列表                       │
│            [{methodId, name:"银行卡", icon, minAmt,   │
│              maxAmt, supportChains:[...]}, ...]        │
│  Step 3: 返回给C端展示                          🟢   │
└──────────────────────────────────────────────────────┘
    │
    ▼ 用户选择充值方式、输入金额、点击确认
    │
    │ POST /api/wallet/deposit/order
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 4: 参数校验                               🟡   │
│          → 金额 ≥ 方式最小值 && ≤ 最大值               │
│          → 币种有效 && 已启用                          │
│                                                       │
│  Step 5: 生成充值订单号 (C+16位)                🟢   │
│          → 例: C20260302143025001                      │
│                                                       │
│  Step 6: 创建 deposit_order 记录               🔴⚡  │
│          → status = Pending(0)                        │
│          → 写入: userId, currencyCode, amount,        │
│            methodId, orderNo, rateSnapshot             │
│          ★ 关键决策: 先建我方订单 还是 先调F5?          │
│            → 建议: 先建我方订单(状态Pending)，          │
│              再调F5，失败则更新为Failed                  │
│                                                       │
│  Step 7: RPC调用 ser-finance.CreateChannelOrder 🔴⚡ │
│          → (depositOrderNo, amount, methodId,         │
│             clientIp, deviceType)                      │
│          → 返回: {channelOrderNo, paymentType,        │
│            paymentUrl/qrCode/transferInfo, expireTime} │
│          ★ 调用失败处理:                               │
│            → 网络超时: 更新订单状态为Failed             │
│            → 业务拒绝: 更新订单状态为Failed + 原因      │
│                                                       │
│  Step 8: 更新 deposit_order                     🟢   │
│          → status = Paying(1)                         │
│          → 写入channelOrderNo, paymentInfo            │
│                                                       │
│  Step 9: 返回支付信息给C端                       🟢   │
│          → {paymentUrl/qrCode, expireTime, orderNo}   │
└──────────────────────────────────────────────────────┘
    │
    ▼ C端展示支付页面（二维码/跳转/转账信息）


════════════════ 阶段二：用户支付 (链外操作) ════════════════

    │
    ▼ 用户在银行App/钱包/链上完成支付
    │
    │ (此步骤在我方系统之外，不可控)
    │ 时间: 即时 ~ 数小时不等
    │
    │ 用户可以在C端轮询订单状态:
    │ GET /api/wallet/deposit/order/status
    │   → ser-wallet查本地deposit_order状态
    │   → 或RPC调用 ser-finance.QueryChannelOrder(F6)
    │


════════════════ 阶段三：三方回调 (三方→ser-finance→ser-wallet) ════════════════

    三方支付渠道
        │ HTTP回调通知
        ▼
┌─ ser-finance ────────────────────────────────────────┐
│  Step 10: 接收三方回调                          🟡   │
│           → 验签 + 幂等校验                           │
│                                                       │
│  Step 11: 更新渠道订单状态                       🟡   │
│           → channelOrder.status = Paid                │
│                                                       │
│  Step 12: RPC调用 ser-wallet.CreditWallet       🔴   │
│           → (bizType=1[充值入账], walletType=1[中心],  │
│              userId, currencyCode, amount,             │
│              refOrderNo=充值订单号)                     │
│                                                       │
│  Step 13: [如有赠金]                             🟡   │
│           RPC调用 ser-wallet.CreditWallet              │
│           → (bizType=2[充值赠金], walletType=2[奖励],  │
│              amount=赠金金额, turnoverMultiple=N)       │
└──────────────────────────────────────────────────────┘
    │
    ▼
┌─ ser-wallet (CreditWallet 内部链路) ─────────────────┐
│                                                       │
│  Step 14: 幂等校验（双层）                       🔴   │
│           Layer1: Redis SetNX                         │
│             key=wallet:idempotent:{bizType}:{refOrderNo}│
│             TTL=24h                                    │
│             → 已存在 → 返回成功(幂等保护)               │
│           Layer2: DB唯一索引兜底                       │
│             uk_order_op_wallet(order_no,op_type,wallet_type)│
│                                                       │
│  Step 15: DB事务操作                             🔴⚡ │
│           BEGIN TRANSACTION                           │
│           │                                           │
│           ├─ SELECT * FROM wallet_account             │
│           │  WHERE user_id=? AND wallet_type=?        │
│           │  AND currency_code=?                      │
│           │  FOR UPDATE  ← 行锁                       │
│           │                                           │
│           ├─ UPDATE wallet_account                    │
│           │  SET available_amt = available_amt + ?    │
│           │  WHERE id = ?                             │
│           │                                           │
│           ├─ INSERT INTO wallet_transaction           │
│           │  (order_no, op_type, direction=1[收入],   │
│           │   amount, available_before, available_after,│
│           │   frozen_before, frozen_after, ...)        │
│           │  ← INSERT-ONLY 不可变流水                  │
│           │                                           │
│           COMMIT                                      │
│                                                       │
│  Step 16: 更新 deposit_order.status = Credited  🟢   │
│                                                       │
│  Step 17: 缓存失效                              🟢   │
│           DEL wallet:balance:{userId}:*               │
│                                                       │
│  Step 18: 异步审计日志                           🟢   │
│           RPC(oneway): ser-blog.AddActLog             │
│           → 失败不阻塞主流程                           │
│                                                       │
│  Step 19: 余额推送                              🟢   │
│           → WebSocket/消息推送通知前端刷新余额          │
│                                                       │
│  Step 20: 返回成功给ser-finance                  🟢   │
│           → {success:true, afterBalance}              │
└──────────────────────────────────────────────────────┘
```

### 5.2 关键步骤深度分析

| 步骤 | 复杂度 | 挑战 | 详细说明 |
|------|--------|------|---------|
| Step 2 | 🟡⚡ | ser-finance未开发 | 需先Mock，定义好IDL合约。第一个阻塞项 |
| Step 6 | 🔴⚡ | 订单创建时序 | 先建本地订单再调外部API，失败时有回滚点 |
| Step 7 | 🔴⚡ | 跨模块RPC | 3秒超时。失败=用户充值失败。需考虑重试策略 |
| Step 12 | 🔴 | 回调入账 | ser-finance作为调用方，我方被动接收。幂等是核心 |
| Step 14 | 🔴 | 双层幂等 | Redis+DB双保障，防止网络重试导致重复入账 |
| Step 15 | 🔴⚡ | 事务+行锁 | FOR UPDATE锁粒度、事务时间、死锁预防是核心挑战 |

### 5.3 异常分支

```
异常场景                     处理策略
─────────────────────────────────────────────────
F5调用超时                   更新deposit_order为Failed，用户可重新发起
三方长时间无回调              deposit_order超时关闭(30min)
回调重复                     幂等保护，返回成功不重复入账
CreditWallet并发调用         Redis分布式锁 + DB行锁 + SQL原子操作
入账后DB提交失败              事务回滚，ser-finance重试会被幂等捕获
赠金入账失败                  主充值入账不受影响(独立幂等key)
ser-blog调用失败             不阻塞(oneway)，后续补偿
```

---

<a id="第六章"></a>
## 第六章 提现完整链路（21步）

> 最复杂的链路。涉及：冻结→审核→出款→确认扣款 四个阶段。
> 跨越 ser-wallet ↔ ser-finance ↔ 三方渠道，需要TCC模式保证资金安全。

### 6.1 完整链路图（阶段式）

```
════════════════ 阶段一：信息收集 (C端→ser-wallet→多模块) ════════════════

C端用户点击"提现"
    │
    │ GET /api/wallet/withdraw/info
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 1: 提取userId                            🟢   │
│                                                       │
│  Step 2: 并行RPC调用 (3个)                       🟡⚡ │
│          ┌─ ser-user.GetUserInfoExpend(userId)        │
│          │  → 用户状态(是否被封), 国家代码             │
│          │  → 封禁用户 = 不允许提现                    │
│          │                                            │
│          ├─ ser-kyc.KycQuery(userId)                  │
│          │  → KYC状态, 实名姓名                       │
│          │  → 未实名 = 限制提现渠道                    │
│          │                                            │
│          └─ ser-finance.GetWithdrawMethods(F2)        │
│             → (currencyCode, userId, kycStatus)       │
│             → 返回: 可用提现方式列表                   │
│               [{methodId, name:"银行卡", feeRate,     │
│                 minAmt, maxAmt, requireKyc}, ...]     │
│                                                       │
│  Step 3: 聚合信息返回C端                         🟢   │
│          → {kycStatus, realName, withdrawMethods,     │
│             userStatus, countryCode}                   │
└──────────────────────────────────────────────────────┘
    │
    ▼ 用户选择提现方式
    │
    │ GET /api/wallet/withdraw/limit
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 4: RPC调用 ser-finance.GetWithdrawLimits(F3)   │
│          → (currencyCode, userId, methodId)     🟡⚡ │
│          → 返回: {singleMin, singleMax,              │
│            dailyUsed, dailyRemain, feeRate, feeAmt}  │
│                                                       │
│  Step 5: 返回限额信息给C端                       🟢   │
└──────────────────────────────────────────────────────┘
    │
    ▼ 用户输入金额，选择/填写收款账户，点击确认


════════════════ 阶段二：冻结+提交 (C端→ser-wallet→ser-finance) ════════════════

    │ POST /api/wallet/withdraw/order
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 6: 参数校验                               🟡   │
│          → 金额范围: min ≤ amount ≤ max              │
│          → 日限额: dailyUsed + amount ≤ dailyLimit   │
│          → 币种有效 && 已启用                         │
│                                                       │
│  Step 7: 实名校验                               🔴⚡ │
│          → withdraw_account.accountName               │
│            == KYC realName (忽略大小写, trim空格)      │
│          → 不匹配 → 拒绝提现                          │
│          ★ 挑战: 跨语言姓名匹配(越南/泰/印尼)          │
│                                                       │
│  Step 8: 手续费计算                              🟡   │
│          → feeAmount = amount × feeRate              │
│          → actualAmount = amount - feeAmount         │
│          → 汇率快照锁定                               │
│          ★ 待确认: 手续费谁算? 我方算→你方校验?         │
│                                                       │
│  Step 9: ★ FreezeBalance (TCC-Try)              🔴⚡ │
│          BEGIN TRANSACTION                            │
│          │                                            │
│          ├─ Redis分布式锁                              │
│          │  key=wallet:lock:{userId}:1:{currency}     │
│          │  SetNX, TTL=10s                            │
│          │                                            │
│          ├─ SELECT * FROM wallet_account              │
│          │  WHERE user_id=? AND wallet_type=1         │
│          │  AND currency_code=? FOR UPDATE            │
│          │                                            │
│          ├─ 校验: available_amt >= amount             │
│          │  → 不足 → 返回余额不足错误                  │
│          │                                            │
│          ├─ UPDATE wallet_account                     │
│          │  SET available_amt = available_amt - ?,    │
│          │      frozen_amt = frozen_amt + ?           │
│          │  WHERE id = ? AND available_amt >= ?       │
│          │  → affected_rows=0 → 并发冲突，余额不足    │
│          │                                            │
│          ├─ INSERT INTO freeze_detail                 │
│          │  (user_id, currency_code, amount,          │
│          │   ref_order_no, status=1[frozen])          │
│          │                                            │
│          ├─ INSERT INTO wallet_transaction            │
│          │  (direction=3[冻结], op_type=12,           │
│          │   available_before/after,                  │
│          │   frozen_before/after)                     │
│          │                                            │
│          COMMIT                                       │
│          释放Redis锁                                   │
│                                                       │
│  Step 10: 创建 withdraw_order                    🟡   │
│           → orderNo = T+16位                          │
│           → status = Frozen(1)                        │
│           → 写入: amount, feeAmount, actualAmount,    │
│             methodId, accountId, rateSnapshot,         │
│             freezeOrderNo                              │
│                                                       │
│  Step 11: RPC调用 ser-finance.SubmitWithdrawOrder(F4) │
│           → (withdrawOrderNo, amount, feeAmount,  🔴⚡│
│              accountInfo, rateSnapshot, clientIp,     │
│              deviceType)                               │
│           → 返回: {success, message}                  │
│           ★ 调用失败:                                  │
│             → 不自动解冻（防止资金风险）                 │
│             → 标记withdraw_order为异常                  │
│             → 人工介入处理                              │
│                                                       │
│  Step 12: 返回成功给C端                          🟢   │
│           → {orderNo, status:"frozen", message}       │
└──────────────────────────────────────────────────────┘


════════════════ 阶段三：审核 (ser-finance内部 → 回调ser-wallet) ════════════════

    ser-finance B端 (风控 + 财务)
        │
        │ 风控审核
        ├─── 拒绝 ─────────────────────────────────────────┐
        │                                                   │
        │    RPC: ser-wallet.UnfreezeBalance          🔴   │
        │    → (userId, currencyCode, refOrderNo,           │
        │       reason="风控拒绝")                           │
        │    → ser-wallet解冻: frozen_amt→available_amt     │
        │    → freeze_detail.status = 2(unfrozen)           │
        │    → wallet_transaction: direction=4(解冻)         │
        │    → 更新withdraw_order.status = Rejected(5)      │
        │    → 缓存失效 + 推送通知                           │
        │                                                   │
        │ 通过 ↓                                            │
        │                                                   │
        │ 财务审核                                           │
        ├─── 拒绝 ──→ 同上，调用 UnfreezeBalance      🔴   │
        │                                                   │
        │ 通过 ↓                                            │
        ▼


════════════════ 阶段四：出款+确认 (ser-finance→三方→ser-finance→ser-wallet) ════

    ser-finance
        │
        │ Step 13: 调用三方渠道API发起出款              🟡
        │          → 银行转账/电子钱包/链上转账
        │
        ▼ 三方处理(可能T+N)
        │
        │ Step 14: 三方回调出款结果
        │
        ├─── 出款失败 ──────────────────────────────────┐
        │                                               │
        │    RPC: ser-wallet.UnfreezeBalance        🔴  │
        │    → (refOrderNo=withdrawOrderNo,             │
        │       reason="出款失败")                       │
        │    → 解冻资金返还用户可用余额                   │
        │    → 更新withdraw_order.status=PayoutFailed(6)│
        │    ★ 待确认: 出款失败是自动解冻还是人工介入?     │
        │                                               │
        ├─── 出款成功 ──────────────────────────────────┐
        │                                               │
        │    RPC: ser-wallet.ConfirmDebit           🔴  │
        │    → (userId, currencyCode,                   │
        │       refOrderNo=withdrawOrderNo)             │
        │    → ser-wallet执行:                          │
        │      BEGIN TRANSACTION                        │
        │      ├─ 查freeze_detail by refOrderNo         │
        │      │  → status必须=1(frozen)                │
        │      │  → 若=2(unfrozen)或3(confirmed)         │
        │      │    → 幂等返回/冲突拒绝                   │
        │      ├─ UPDATE wallet_account                 │
        │      │  SET frozen_amt = frozen_amt - ?       │
        │      │  WHERE id = ? AND frozen_amt >= ?      │
        │      ├─ UPDATE freeze_detail                  │
        │      │  SET status = 3(confirmed)             │
        │      ├─ INSERT wallet_transaction             │
        │      │  (direction=2[支出], op_type=13)       │
        │      COMMIT                                   │
        │                                               │
        │    Step 15: 更新withdraw_order.status    🟢  │
        │             = Completed(4) (终态)              │
        │                                               │
        │    Step 16: 缓存失效                     🟢  │
        │             DEL wallet:balance:{userId}:*     │
        │                                               │
        │    Step 17: 审计日志 + 推送通知           🟢  │
        │             ser-blog.AddActLog(oneway)        │
        │             WebSocket余额变更推送             │
        └───────────────────────────────────────────────┘
```

### 6.2 提现状态流转全景

```
Created(0) ──→ Frozen(1) ──→ RiskOK(2) ──→ Paying(3) ──→ Completed(4) ✓
                  │               │              │
                  │               │              └──→ PayoutFailed(6) + Unfreeze
                  │               └──→ Rejected(5) + Unfreeze
                  └──→ Rejected(5) + Unfreeze

freeze_detail.status:
  Frozen(1) ──→ Unfrozen(2) [拒绝/失败]
            ──→ Confirmed(3) [成功扣款]
  ★ Unfrozen和Confirmed互斥，同一refOrderNo只能走一个
```

### 6.3 关键步骤深度分析

| 步骤 | 复杂度 | 挑战 | 详细说明 |
|------|--------|------|---------|
| Step 2 | 🟡⚡ | 并行RPC | 3个RPC并行，任一失败需优雅降级。ser-finance未开发 |
| Step 7 | 🔴⚡ | 实名匹配 | 跨语言（越/泰/印尼）姓名比对，大小写/空格/特殊字符处理 |
| Step 9 | 🔴⚡ | 冻结操作 | TCC-Try阶段。分布式锁+行锁+原子SQL。是资金安全核心 |
| Step 11 | 🔴⚡ | 跨模块提交 | 提交失败不能自动解冻(防风险)，需人工介入 |
| Unfreeze | 🔴 | 互斥保护 | 与ConfirmDebit互斥，状态机必须严格 |
| ConfirmDebit | 🔴 | 最终扣款 | 不可逆操作，frozen_amt真正减少 |

### 6.4 异常分支全景

```
异常场景                         处理策略                        资金影响
─────────────────────────────────────────────────────────────────────────
GetUserInfoExpend超时            降级：不校验用户状态，继续         无
KycQuery超时                    降级：限制可用提现方式             无
GetWithdrawMethods(F2)超时       返回错误，用户重试                无
FreezeBalance余额不足            拒绝提现，返回错误                无
FreezeBalance并发冲突            Redis锁拦截/DB行锁串行化          无
SubmitWithdrawOrder(F4)超时      不解冻，标记异常，人工处理         冻结中
风控拒绝                         UnfreezeBalance                 释放冻结
财务拒绝                         UnfreezeBalance                 释放冻结
出款失败                         UnfreezeBalance(或人工)          释放冻结
出款成功                         ConfirmDebit                    最终扣款
Unfreeze+ConfirmDebit竞争        freeze_detail状态互斥保护        安全
ConfirmDebit重复调用             幂等(status已confirmed→返回成功)  安全
```

---

<a id="第七章"></a>
## 第七章 兑换完整链路（内部闭环）

> 纯内部操作，无外部RPC依赖。单个DB事务完成。
> 是所有链路中最"干净"的。

### 7.1 完整链路图

```
C端用户点击"兑换"
    │
    │ GET /api/wallet/exchange/preview
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 1: 提取userId, currencyCode, amount       🟢   │
│                                                       │
│  Step 2: 查询实时汇率                            🟢   │
│          → Redis缓存 or DB: exchange_rate             │
│          → depositRate (法币→BSB方向)                  │
│                                                       │
│  Step 3: 计算兑换金额                            🟡   │
│          exchangeAmount = fiatAmount × depositRate     │
│                           / BSB_anchor_ratio          │
│          ★ 必须用shopspring/decimal精确计算            │
│          ★ 不能用float64                              │
│                                                       │
│  Step 4: 计算赠金(如有活动)                       🟡   │
│          bonusAmount = 按活动规则计算                   │
│          turnoverMultiple = N                          │
│                                                       │
│  Step 5: 锁定汇率快照                            🟢   │
│          → 返回rateSnapshot + expireTime              │
│          → "所见即所得": 展示汇率 = 最终结算汇率        │
│                                                       │
│  Step 6: 返回预览信息给C端                        🟢   │
│          → {fiatAmount, exchangeAmount, bonusAmount,  │
│             rate, rateSnapshot, expireTime}            │
└──────────────────────────────────────────────────────┘
    │
    ▼ 用户确认兑换
    │
    │ POST /api/wallet/exchange/create
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 7: 参数校验                               🟡   │
│          → 汇率快照未过期                             │
│          → 金额 > 0                                  │
│          → 币种有效                                   │
│                                                       │
│  Step 8: 生成兑换订单号 (E+16位)                🟢   │
│                                                       │
│  Step 9: ★ 单事务原子操作                       🔴⚡ │
│          BEGIN TRANSACTION                            │
│          │                                            │
│          │── Redis分布式锁                             │
│          │   key=wallet:lock:{userId}:1:{currency}    │
│          │                                            │
│          │── 幂等校验 (Redis SetNX + DB唯一索引)      │
│          │                                            │
│          ├─ DebitWallet (法币中心钱包)                 │
│          │  SELECT FOR UPDATE wallet_account          │
│          │    WHERE user_id=? AND wallet_type=1       │
│          │    AND currency_code=fiat                  │
│          │  UPDATE available_amt -= fiatAmount        │
│          │    WHERE available_amt >= fiatAmount       │
│          │  INSERT wallet_transaction                 │
│          │    (op_type=11, direction=2[支出])          │
│          │                                            │
│          ├─ CreditWallet (BSB中心钱包)                │
│          │  SELECT FOR UPDATE wallet_account          │
│          │    WHERE user_id=? AND wallet_type=1       │
│          │    AND currency_code='BSB'                 │
│          │  UPDATE available_amt += exchangeAmount    │
│          │  INSERT wallet_transaction                 │
│          │    (op_type=6, direction=1[收入])           │
│          │                                            │
│          ├─ [如有赠金] CreditWallet (BSB奖励钱包)     │
│          │  SELECT FOR UPDATE wallet_account          │
│          │    WHERE user_id=? AND wallet_type=2       │
│          │    AND currency_code='BSB'                 │
│          │  UPDATE available_amt += bonusAmount       │
│          │  INSERT wallet_transaction                 │
│          │    (op_type=7, direction=1[收入],           │
│          │     turnover_multiple=N)                   │
│          │                                            │
│          ├─ INSERT exchange_order                     │
│          │  (orderNo, userId, fromCurrency, toCurrency,│
│          │   fromAmount, toAmount, bonusAmount,       │
│          │   rateSnapshot, status=Completed)          │
│          │                                            │
│          COMMIT                                       │
│          释放Redis锁                                   │
│                                                       │
│  Step 10: 缓存失效                              🟢   │
│           DEL wallet:balance:{userId}:*               │
│                                                       │
│  Step 11: 审计日志                              🟢   │
│           ser-blog.AddActLog(oneway)                  │
│                                                       │
│  Step 12: 余额推送                              🟢   │
│           WebSocket通知前端刷新                        │
│                                                       │
│  Step 13: 返回结果                              🟢   │
│           → {orderNo, fromAmount, toAmount,           │
│              bonusAmount, rate}                        │
└──────────────────────────────────────────────────────┘
```

### 7.2 关键特征

```
特征对比:
─────────────────────────────────────────────
充值:  4个阶段, 跨3个系统, 异步回调
提现:  4个阶段, 跨3个系统, TCC模式
兑换:  1个阶段, 纯内部,    单事务原子操作  ← 最简单
```

### 7.3 关键步骤分析

| 步骤 | 复杂度 | 挑战 | 详细说明 |
|------|--------|------|---------|
| Step 3 | 🟡 | 精度计算 | 必须shopspring/decimal，float64会丢精度 |
| Step 5 | 🟢 | 汇率锁定 | "所见即所得"，但汇率可能已过期→需校验expireTime |
| Step 9 | 🔴⚡ | 多钱包事务 | 同一事务操作2~3个wallet_account行，需固定锁序防死锁 |

**死锁预防规则：**
```
锁顺序: center钱包(fiat) → center钱包(BSB) → reward钱包(BSB)
即: wallet_type ASC, currency_code ASC
永远不能反向锁
```

---

<a id="第八章"></a>
## 第八章 冻结三阶段（TCC模式）链路

> 提现专用的资金安全机制。本章聚焦TCC三个阶段的内部链路。

### 8.1 TCC映射关系

```
标准TCC术语          钱包实现              触发时机
─────────────────────────────────────────────────
Try                 FreezeBalance         用户提交提现
Confirm             ConfirmDebit          三方出款成功
Cancel              UnfreezeBalance       审核拒绝/出款失败
```

### 8.2 Try阶段：FreezeBalance 内部链路

```
RPC入口: FreezeBalance(userId, currencyCode, amount, refOrderNo)
    │
    ▼
┌─ 幂等校验 ──────────────────────────────────────────┐
│  Redis: SetNX wallet:idempotent:freeze:{refOrderNo}  │
│  → 已存在 → 返回成功 (重复请求保护)             🟢   │
└──────────────────────────┬──────────────────────────┘
                           │ 新请求
                           ▼
┌─ Redis分布式锁 ─────────────────────────────────────┐
│  SetNX wallet:lock:{userId}:1:{currency}             │
│  TTL=10s                                       🟡   │
│  → 获取失败 → 返回"操作频繁"错误                      │
└──────────────────────────┬──────────────────────────┘
                           │ 获取成功
                           ▼
┌─ DB事务 ─────────────────────────────────────────────┐
│  ① SELECT FOR UPDATE wallet_account             🔴   │
│     WHERE user_id=? AND wallet_type=1                │
│     AND currency_code=?                              │
│                                                       │
│  ② 校验: available_amt >= amount                      │
│     → 不足 → ROLLBACK + 释放锁 + 返回余额不足错误     │
│                                                       │
│  ③ UPDATE wallet_account                         🔴  │
│     SET available_amt = available_amt - ?,            │
│         frozen_amt = frozen_amt + ?                   │
│     WHERE id = ? AND available_amt >= ?               │
│     → affected_rows=0 → 并发竞争失败                  │
│                                                       │
│  ④ INSERT freeze_detail                          🟡  │
│     (ref_order_no, status=1[frozen],                  │
│      amount, user_id, currency_code)                  │
│                                                       │
│  ⑤ INSERT wallet_transaction                     🟡  │
│     (direction=3[冻结], op_type=12,                   │
│      amount, available_before/after,                  │
│      frozen_before/after)                             │
│                                                       │
│  COMMIT                                               │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─ 后处理 ─────────────────────────────────────────────┐
│  释放Redis分布式锁                              🟢   │
│  缓存失效: DEL wallet:balance:{userId}:*              │
│  返回: {success, availableBalance, frozenBalance}     │
└──────────────────────────────────────────────────────┘

资金变化:
  available_amt:  100 → 70  (减少30)
  frozen_amt:       0 → 30  (增加30)
  total_balance:  100 → 100 (不变!)
```

### 8.3 Confirm阶段：ConfirmDebit 内部链路

```
RPC入口: ConfirmDebit(userId, currencyCode, refOrderNo)
    │
    ▼
┌─ 幂等校验 ──────────────────────────────────────────┐
│  Redis: SetNX wallet:idempotent:confirm:{refOrderNo}  │
│  → 已存在 → 返回成功                            🟢   │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─ DB事务 ─────────────────────────────────────────────┐
│  ① 查freeze_detail by refOrderNo                🔴   │
│     → status=1(frozen) → 正常继续                     │
│     → status=2(unfrozen) → 冲突！已被解冻，拒绝确认    │
│     → status=3(confirmed) → 幂等，返回成功             │
│     → 不存在 → 错误，冻结记录不存在                     │
│     ★ 这是Confirm/Cancel互斥的核心检查点               │
│                                                       │
│  ② UPDATE wallet_account                         🔴  │
│     SET frozen_amt = frozen_amt - ?                   │
│     WHERE user_id=? AND wallet_type=1                 │
│     AND currency_code=? AND frozen_amt >= ?           │
│                                                       │
│  ③ UPDATE freeze_detail                          🟡  │
│     SET status = 3(confirmed)                         │
│     WHERE ref_order_no = ? AND status = 1             │
│     → affected_rows=0 → 并发冲突(已被Cancel)          │
│                                                       │
│  ④ INSERT wallet_transaction                     🟡  │
│     (direction=2[支出], op_type=13[提现确认],          │
│      frozen_before/after)                             │
│                                                       │
│  COMMIT                                               │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
资金变化:
  available_amt:   70 → 70  (不变)
  frozen_amt:      30 →  0  (减少30, 真正扣除)
  total_balance:  100 → 70  (减少30! 用户钱真正少了)
```

### 8.4 Cancel阶段：UnfreezeBalance 内部链路

```
RPC入口: UnfreezeBalance(userId, currencyCode, refOrderNo, reason)
    │
    ▼
┌─ 幂等校验 ──────────────────────────────────────────┐
│  Redis: SetNX wallet:idempotent:unfreeze:{refOrderNo} │
│  → 已存在 → 返回成功                            🟢   │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─ DB事务 ─────────────────────────────────────────────┐
│  ① 查freeze_detail by refOrderNo                🔴   │
│     → status=1(frozen) → 正常继续                     │
│     → status=2(unfrozen) → 幂等，返回成功              │
│     → status=3(confirmed) → 冲突！已被确认扣款，拒绝   │
│     ★ 与Confirm完全对称的互斥检查                      │
│                                                       │
│  ② UPDATE wallet_account                         🔴  │
│     SET available_amt = available_amt + ?,            │
│         frozen_amt = frozen_amt - ?                   │
│     WHERE user_id=? AND wallet_type=1                 │
│     AND currency_code=? AND frozen_amt >= ?           │
│                                                       │
│  ③ UPDATE freeze_detail                          🟡  │
│     SET status = 2(unfrozen), reason = ?              │
│     WHERE ref_order_no = ? AND status = 1             │
│                                                       │
│  ④ INSERT wallet_transaction                     🟡  │
│     (direction=4[解冻], op_type=14[提现解冻])          │
│                                                       │
│  COMMIT                                               │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
资金变化:
  available_amt:   70 → 100 (恢复30)
  frozen_amt:      30 →   0 (释放30)
  total_balance:  100 → 100 (不变! 钱还给用户了)
```

### 8.5 TCC互斥保障总结

```
freeze_detail.status 状态转换:

    Frozen(1) ──→ Confirmed(3)  [ConfirmDebit]
        │
        └─────→ Unfrozen(2)    [UnfreezeBalance]

保障机制:
  1. freeze_detail.status = 1 是唯一可操作状态
  2. Confirm检查status=1，若=2则拒绝(已解冻)
  3. Unfreeze检查status=1，若=3则拒绝(已确认)
  4. UPDATE WHERE status=1 → affected_rows=0 = 并发竞争失败
  5. 两个操作永远不可能同时成功
  6. DB事务保证原子性
```

---

<a id="第九章"></a>
## 第九章 人工操作链路（补单/加款/减款）

> B端财务人员通过ser-finance后台发起，回调ser-wallet执行。
> 链路相对简单，核心是RPC幂等。

### 9.1 充值补单链路

```
ser-finance B端 (财务人员操作)
    │
    │ Step 1: 创建补单记录 (B+16位订单号)
    │ Step 2: 审核通过
    │
    │ RPC调用 ser-wallet.CreditWallet
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  (bizType=3[补单], walletType=1[中心],                │
│   refOrderNo=补单号, amount=补单金额)                  │
│                                                       │
│  → 幂等校验(Redis+DB)                           🔴   │
│  → DB事务: 加款 + 写流水                          🔴   │
│  → [如有赠金] 再次CreditWallet(bizType=2,reward)  🟡   │
│  → 缓存失效 + 审计日志                            🟢   │
└──────────────────────────────────────────────────────┘
```

### 9.2 人工加款链路

```
ser-finance B端 (管理员操作)
    │
    │ Step 1: 创建加款记录 (A+16位订单号)
    │         → 可指定任意walletType(1-5)
    │ Step 2: 审核通过
    │
    │ RPC调用 ser-wallet.CreditWallet (可能多次)
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  (bizType=4[人工加款], walletType=指定钱包,            │
│   refOrderNo=加款单号, amount=加款金额)                │
│                                                       │
│  ★ 一次加款可能涉及多个钱包:                           │
│    CreditWallet(walletType=1, amount=X)               │
│    CreditWallet(walletType=2, amount=Y)               │
│    幂等key = (refOrderNo + bizType + walletType)      │
│    → 不同walletType不冲突                             │
│                                                       │
│  → 每次调用独立: 幂等校验→事务→缓存失效         🔴   │
└──────────────────────────────────────────────────────┘
```

### 9.3 人工减款链路

```
ser-finance B端 (管理员操作)
    │
    │ Step 1: 创建减款记录 (M+16位订单号)
    │ Step 2: 审核通过
    │
    │ RPC调用 ser-wallet.DebitWallet
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  (bizType=15[人工减款], walletType=指定钱包,           │
│   refOrderNo=减款单号, amount=减款金额,                │
│   insufficientStrategy=1或2)                          │
│                                                       │
│  Strategy=1: 余额不足→拒绝                       🟡  │
│  Strategy=2: 余额不足→扣实际可用，返回实际扣款额  🟡  │
│                                                       │
│  → 幂等校验(Redis+DB)                           🔴   │
│  → Redis分布式锁                                 🟡   │
│  → DB事务:                                            │
│    SELECT FOR UPDATE                                  │
│    Strategy=1: available_amt >= amount 否则拒绝        │
│    Strategy=2: actualDebit = min(available_amt, amount)│
│    UPDATE available_amt -= actualDebit                 │
│    INSERT wallet_transaction                          │
│  → 缓存失效 + 审计日志                            🟢   │
│  → 返回: {success, afterBalance, actualDebitAmount}   │
└──────────────────────────────────────────────────────┘
```

### 9.4 余额查询（人工操作辅助）

```
ser-finance B端 (管理员查看用户余额)
    │
    │ RPC调用 ser-wallet.GetBalance
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  (userId, currencyCode=optional)                      │
│                                                       │
│  → Redis缓存查询                                🟢   │
│  → 或DB查询5个子钱包                              🟢   │
│  → 返回各子钱包可用+冻结金额                       🟢   │
│                                                       │
│  ★ 用于: 加款/减款前确认用户当前余额                   │
└──────────────────────────────────────────────────────┘
```

---

<a id="第十章"></a>
## 第十章 汇率定时刷新链路

> 后台定时任务，无用户触发。
> 定期从三方API获取实时汇率，计算偏差，决定是否更新平台汇率。

### 10.1 完整链路图

```
ser-cron (定时任务调度)
    │
    │ 每5分钟触发 (可配置)
    │ RPC调用 ser-wallet.RefreshExchangeRate
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│                                                       │
│  Step 1: 查询所有已启用币种                       🟢   │
│          SELECT * FROM currency_config               │
│          WHERE status = 1                             │
│          → [VND, IDR, THB, USDT]                     │
│                                                       │
│  遍历每个币种:                                         │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Step 2: 调用三方汇率API (HTTP)             🟡⚡ │ │
│  │         → API源1: 获取realtime rate              │ │
│  │         → API源2: 获取realtime rate              │ │
│  │         → API源3: 获取realtime rate              │ │
│  │         → 取平均值: avgRate                       │ │
│  │         ★ 挑战: 三方API不稳定，需超时+降级         │ │
│  │                                                   │ │
│  │ Step 3: 计算偏差                             🟡  │ │
│  │         currentPlatformRate = DB当前平台汇率       │ │
│  │         deviation = |avgRate - currentPlatformRate│ │
│  │                     / avgRate × 100%              │ │
│  │                                                   │ │
│  │ Step 4: 决策                                 🟡  │ │
│  │         deviation < threshold(如2%)               │ │
│  │         → 不更新 (波动在正常范围)                  │ │
│  │         deviation >= threshold                    │ │
│  │         → 更新平台汇率                            │ │
│  │                                                   │ │
│  │ Step 5: [如需更新] DB操作                    🟡  │ │
│  │         UPDATE currency_config                    │ │
│  │         SET platform_rate = newRate,               │ │
│  │             deposit_rate = 计算值,                 │ │
│  │             withdraw_rate = 计算值                 │ │
│  │         WHERE currency_code = ?                   │ │
│  │                                                   │ │
│  │ Step 6: [如需更新] 写入变更日志              🟢  │ │
│  │         INSERT INTO exchange_rate_log             │ │
│  │         (currency_code, old_rate, new_rate,       │ │
│  │          deviation, source, created_at)            │ │
│  │                                                   │ │
│  │ Step 7: [如需更新] 缓存失效                  🟢  │ │
│  │         DEL wallet:currency:rate:{code}           │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  Step 8: 返回刷新结果                            🟢   │
│          → {updatedCurrencies, skippedCurrencies,    │
│             errors}                                   │
└──────────────────────────────────────────────────────┘
```

### 10.2 关键挑战

| 挑战 | 描述 | 解决方案 |
|------|------|---------|
| ⚡ 三方API不稳定 | 超时、返回异常数据 | 多源取均值，单源失败不阻塞，最少2源有效才更新 |
| ⚡ 汇率突变 | 极端市场波动导致大幅偏差 | 设置最大偏差上限(如10%)，超过需人工确认 |
| ⚡ 定时任务幂等 | 同一时刻重复触发 | 分布式锁 key=wallet:cron:rate_refresh |

---

<a id="第十一章"></a>
## 第十一章 B端币种配置管理链路

> 管理员通过后台配置币种信息。
> 低频操作，但影响全局（所有用币种的模块）。

### 11.1 创建币种链路

```
B端管理员
    │ POST /admin/api/wallet/currency/create
    ▼
gate-back → WhiteCheck → BackAuth → Handler
    │
    │ RPC调用 ser-wallet.CreateCurrency
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 1: 参数校验                               🟡   │
│          → currencyCode格式(大写字母, 3-5位)          │
│          → decimal精度(0-8)                           │
│          → 名称/符号非空                               │
│                                                       │
│  Step 2: 唯一性校验                              🟡   │
│          → currencyCode是否已存在                     │
│          → 已存在 → 返回"币种代码已存在"错误           │
│                                                       │
│  Step 3: 图标上传(如有)                          🟢   │
│          → RPC: ser-s3.UploadFile                     │
│          → 返回图标URL                                │
│                                                       │
│  Step 4: DB写入                                  🟢   │
│          INSERT INTO currency_config                  │
│          (code, name, symbol, decimal, icon,          │
│           platform_rate, deposit_rate, withdraw_rate,  │
│           status=0[禁用], sort_value, ...)            │
│          ★ 新币种默认禁用，需单独启用                   │
│                                                       │
│  Step 5: 缓存失效                                🟢   │
│          DEL wallet:currency:list:*                   │
│                                                       │
│  Step 6: 审计日志                                🟢   │
│          ser-blog.AddActLog                           │
│                                                       │
│  Step 7: 返回                                    🟢   │
│          → {id, currencyCode}                         │
└──────────────────────────────────────────────────────┘
```

### 11.2 启用/禁用币种链路

```
B端管理员
    │ PUT /admin/api/wallet/currency/status
    ▼
gate-back → 中间件链
    │
    │ RPC调用 ser-wallet.UpdateCurrencyStatus
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 1: 查询当前币种                            🟢   │
│          → 不存在 → 错误                              │
│                                                       │
│  Step 2: 状态变更校验                            🟡   │
│          → 启用: 检查必填字段是否完整                   │
│            (rate, decimal, name, symbol等)             │
│          → 禁用: 检查是否有用户持有该币种余额     🟡⚡ │
│            ★ 挑战: 禁用有余额币种的影响范围？           │
│                                                       │
│  Step 3: 更新状态                                🟢   │
│          UPDATE currency_config SET status = ?        │
│                                                       │
│  Step 4: 缓存失效                                🟢   │
│          DEL wallet:currency:list:*                   │
│          ★ 重要: 这会影响所有模块的币种查询             │
│                                                       │
│  Step 5: 审计日志                                🟢   │
│          记录: 谁、什么时间、启用/禁用了哪个币种       │
└──────────────────────────────────────────────────────┘
```

### 11.3 设置基准币种链路

```
B端管理员
    │ PUT /admin/api/wallet/currency/base
    ▼
gate-back → 中间件链
    │
    │ RPC调用 ser-wallet.SetBaseCurrency
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  Step 1: 检查是否已设置基准币种                   🔴   │
│          → 已设置 → 拒绝修改 (一次性操作)              │
│          ★ 基准币种确定后不可变更                       │
│            (影响所有历史汇率数据的基准)                  │
│                                                       │
│  Step 2: 校验币种存在且已启用                     🟡   │
│                                                       │
│  Step 3: 设置标记                                🟢   │
│          UPDATE currency_config                       │
│          SET is_base = 1                              │
│          WHERE currency_code = ?                      │
│                                                       │
│  Step 4: 缓存失效 + 审计日志                     🟢   │
└──────────────────────────────────────────────────────┘
```

---

<a id="第十二章"></a>
## 第十二章 钱包初始化链路

> 用户注册时由ser-user调用，为新用户创建所有子钱包账户。
> 只调用一次，但要保证幂等。

### 12.1 完整链路图

```
新用户注册
    │
    │ ser-user 注册流程中
    │ RPC调用 ser-wallet.InitWallet(userId)
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│                                                       │
│  Step 1: 幂等校验                                🟢   │
│          → 查wallet_account是否已有该userId记录        │
│          → 已有 → 返回成功 (重复注册保护)              │
│                                                       │
│  Step 2: 查询所有已启用币种                       🟢   │
│          → [VND, IDR, THB, USDT, BSB]                │
│                                                       │
│  Step 3: 批量创建子钱包                           🟡   │
│          遍历每个启用币种:                             │
│            遍历每个激活的walletType(1,2,3):            │
│              INSERT INTO wallet_account               │
│              (user_id, wallet_type, currency_code,    │
│               available_amt=0, frozen_amt=0,          │
│               version=0)                              │
│                                                       │
│          → 5币种 × 3钱包类型 = 15条记录               │
│          ★ 用批量INSERT提升性能:                       │
│            INSERT INTO wallet_account VALUES          │
│            (...), (...), (...), ...                   │
│                                                       │
│  Step 4: 返回成功                                🟢   │
│          → {success: true}                            │
└──────────────────────────────────────────────────────┘
```

### 12.2 关键考量

| 考量 | 说明 |
|------|------|
| 幂等 | userId + wallet_type + currency_code 联合唯一索引保护 |
| 性能 | 批量INSERT而非逐条，减少DB交互次数 |
| 新增币种 | 启用新币种后，历史用户的钱包如何补建？→ 懒加载(首次充值时自动创建) 或 批量脚本 |
| 事务 | 15条记录在同一事务中，保证原子性 |

---

<a id="第十三章"></a>
## 第十三章 RPC暴露接口被调用链路

> 除了ser-finance，还有投注/结算/活动系统会调用我方RPC。
> 本章梳理所有"被调用"场景的链路特征。

### 13.1 调用方及场景总览

```
┌──────────────────────────────────────────────────────────────────┐
│                    ser-wallet RPC暴露接口                         │
├───────────┬──────────────────┬───────────────────────────────────┤
│  调用方    │ 调用接口          │ 场景                              │
├───────────┼──────────────────┼───────────────────────────────────┤
│ ser-finance│ CreditWallet     │ 充值入账、补单、赠金              │
│           │ DebitWallet       │ 人工减款                          │
│           │ FreezeBalance     │ (如果由finance发起冻结)            │
│           │ UnfreezeBalance   │ 审核拒绝、出款失败                │
│           │ ConfirmDebit      │ 出款成功                          │
│           │ GetBalance        │ B端查看用户余额                   │
│           │ BatchGetBalance   │ B端列表批量查余额                 │
│           │ GetWalletFlow     │ B端审计查流水                     │
│           │ InitWallet        │ (如果注册走finance)                │
├───────────┼──────────────────┼───────────────────────────────────┤
│ 投注/结算  │ DebitWallet       │ 下注扣款 (bizType=10)             │
│           │ CreditWallet      │ 结算返奖 (bizType=9)              │
├───────────┼──────────────────┼───────────────────────────────────┤
│ 活动系统   │ CreditWallet      │ 活动奖励 (bizType=5)              │
├───────────┼──────────────────┼───────────────────────────────────┤
│ ser-user   │ InitWallet        │ 新用户注册                        │
├───────────┼──────────────────┼───────────────────────────────────┤
│ 所有模块   │ GetCurrencyList   │ 需要币种列表的任何场景             │
│           │ GetExchangeRate   │ 需要汇率的任何场景                 │
│           │ GetBaseCurrency   │ 需要基准币种的场景                 │
└───────────┴──────────────────┴───────────────────────────────────┘
```

### 13.2 投注扣款链路

```
用户下注
    │
    │ ser-betting 投注服务
    │ RPC调用 ser-wallet.DebitWallet
    │ (bizType=10[消费], walletType=1[中心],
    │  userId, currencyCode='BSB', amount=投注金额,
    │  refOrderNo=投注订单号, insufficientStrategy=1[不足拒绝])
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  幂等校验 → 分布式锁 → DB事务(FOR UPDATE + 扣款 +    │
│  流水) → 缓存失效 → 返回                              │
│                                                       │
│  ★ 高频场景! 性能是核心:                               │
│    → Redis锁拦截99%并发                                │
│    → FOR UPDATE行级锁(不是表锁)                        │
│    → 单条UPDATE原子操作                                │
│    → 流水INSERT-ONLY无锁竞争                           │
│                                                       │
│  ★ 扣款优先级:                                        │
│    → 先扣奖励钱包(turnover消耗) → 再扣中心钱包          │
│    → 或由调用方指定walletType                          │
│    → 待确认: 扣款顺序策略                              │
└──────────────────────────────────────────────────────┘
```

### 13.3 结算返奖链路

```
赛事结算
    │
    │ ser-settlement 结算服务
    │ RPC调用 ser-wallet.CreditWallet
    │ (bizType=9[返奖], walletType=1[中心],
    │  userId, currencyCode='BSB', amount=返奖金额,
    │  refOrderNo=结算订单号)
    ▼
┌─ ser-wallet ─────────────────────────────────────────┐
│  幂等校验 → DB事务(FOR UPDATE + 加款 + 流水) →       │
│  缓存失效 → 推送余额更新 → 返回                       │
│                                                       │
│  ★ 结算可能批量返奖(同一赛事多人中奖)                   │
│    → 每个用户独立调用(不做批量API)                      │
│    → 每个调用独立幂等保护                              │
└──────────────────────────────────────────────────────┘
```

---

<a id="第十四章"></a>
## 第十四章 关键步骤复杂度/挑战全景矩阵

### 14.1 按链路维度

| 链路 | 总步骤 | 🔴关键步骤 | ⚡挑战步骤 | 跨模块RPC | 核心风险 |
|------|--------|-----------|-----------|----------|---------|
| 余额查询 | 12 | 0 | 1 | 0 | 缓存一致性 |
| 币种列表 | 10 | 0 | 0 | 0 | 缓存失效联动 |
| 充值 | 20 | 4 | 4 | 3(F1+F5+F6) | 重复入账、跨模块时序 |
| 提现 | 21 | 6 | 5 | 4(F2+F3+F4+多次回调) | 资金冻结安全、TCC互斥 |
| 兑换 | 13 | 1 | 1 | 0 | 多钱包死锁、精度计算 |
| TCC冻结 | 5 | 3 | 1 | 0 | 互斥保障 |
| TCC确认 | 4 | 2 | 0 | 0 | 与Cancel互斥 |
| TCC解冻 | 4 | 2 | 0 | 0 | 与Confirm互斥 |
| 人工补单 | 5 | 1 | 0 | 0 | 幂等 |
| 人工加款 | 5 | 1 | 0 | 0 | 幂等 |
| 人工减款 | 6 | 1 | 0 | 0 | 余额不足策略 |
| 汇率刷新 | 8 | 0 | 2 | 0(HTTP外部API) | 三方API稳定性 |
| 币种配置 | 7 | 0 | 0 | 1(ser-s3) | 缓存失效全局影响 |
| 钱包初始化 | 4 | 0 | 0 | 0 | 幂等+批量性能 |
| 投注扣款 | 5 | 1 | 0 | 0 | 高频并发 |
| 结算返奖 | 5 | 1 | 0 | 0 | 批量幂等 |

### 14.2 按技术挑战维度

```
┌──────────────────────────────────────────────────────────────────┐
│                      技术挑战影响矩阵                             │
├──────────────────┬───────────────────────────────────────────────┤
│ 并发安全          │ 充值入账、提现冻结、兑换、投注扣款              │
│ (分布式锁+行锁)   │ → 几乎所有写操作都需要                        │
├──────────────────┼───────────────────────────────────────────────┤
│ 幂等性            │ 所有RPC写接口 (12个)                          │
│ (Redis+DB双层)    │ → 重试/网络抖动是常态                         │
├──────────────────┼───────────────────────────────────────────────┤
│ 数据一致性        │ 充值(跨模块回调)、提现(TCC)、兑换(本地事务)     │
│ (事务+TCC)        │ → 三种模式对应三种场景                         │
├──────────────────┼───────────────────────────────────────────────┤
│ 精度计算          │ 兑换预览、汇率刷新、手续费计算                  │
│ (shopspring)      │ → 所有涉及金额×汇率的场景                     │
├──────────────────┼───────────────────────────────────────────────┤
│ 跨模块协作        │ 充值(ser-finance回调)、提现(双向RPC)            │
│ (IDL合约+Mock)    │ → 最大阻塞项，ser-finance未开发                │
├──────────────────┼───────────────────────────────────────────────┤
│ 死锁预防          │ 兑换(多钱包锁)、并发充值+提现(同用户)           │
│ (固定锁序)        │ → wallet_type ASC, currency_code ASC          │
├──────────────────┼───────────────────────────────────────────────┤
│ 缓存一致性        │ 余额缓存、币种缓存、汇率缓存                   │
│ (短TTL+写后失效)  │ → 每个写操作后都需要主动失效对应缓存            │
└──────────────────┴───────────────────────────────────────────────┘
```

### 14.3 按编码优先级

```
P0 (核心骨架，必须最先完成):
  ├─ wallet_account CRUD + 事务模板
  ├─ CreditWallet / DebitWallet 核心逻辑
  ├─ 双层幂等框架
  ├─ 分布式锁封装
  └─ 余额查询 + 缓存策略

P1 (核心业务流程):
  ├─ FreezeBalance / UnfreezeBalance / ConfirmDebit (TCC三件套)
  ├─ 充值链路 (deposit_order + 状态机)
  ├─ 提现链路 (withdraw_order + 状态机 + 冻结)
  ├─ 兑换链路 (exchange_order + 单事务)
  └─ 钱包初始化

P2 (B端管理):
  ├─ 币种CRUD
  ├─ 汇率管理 + 定时刷新
  ├─ B端余额/流水查询
  └─ 设置基准币种

P3 (增强功能):
  ├─ 提现账户管理
  ├─ 交易记录查询 (C端)
  ├─ WebSocket余额推送
  └─ 审计日志完善
```

---

<a id="第十五章"></a>
## 第十五章 链路级故障传播与降级策略

### 15.1 故障传播链

```
故障源                  影响范围                  传播链路
──────────────────────────────────────────────────────────────
Redis宕机              全部写操作降级             分布式锁获取失败
                       全部读操作降级             → 降级为DB直查(失去并发保护)
                                                 → 幂等Layer1失效，依赖Layer2(DB)

TiDB慢查询             全部操作超时              事务等待 → RPC 3秒超时
                                                 → gateway返回系统错误

ser-finance不可用       充值/提现链路阻断          F1-F6调用失败
                       人工操作链路阻断            → 返回"服务暂不可用"
                       ★ 余额查询和兑换不受影响     → 核心域隔离

ser-ip不可用            BlackCheck放行(fail-open)  不影响核心业务
                       WhiteCheck拒绝(fail-closed)  B端全部不可用

ser-i18n不可用          错误码返回原始消息          不影响业务逻辑
                                                   用户看到技术性错误文案

ser-blog不可用          审计日志丢失               oneway调用，不阻塞
                                                   后续可补偿

ser-user不可用          提现无法校验用户状态        提现被阻塞
                       ★ 其他操作不受影响           隔离域

ser-kyc不可用           提现无法校验KYC            提现被阻塞
                       ★ 其他操作不受影响           隔离域

三方汇率API不可用       汇率不刷新                  保持旧汇率
                       ★ 不影响正在进行的交易        快照已锁定

ETCD不可用              服务发现失败               新连接建立失败
                       配置无法更新                已有连接不受影响(缓存)
```

### 15.2 降级策略矩阵

| 故障 | 降级策略 | 用户感知 | 恢复方式 |
|------|---------|---------|---------|
| Redis不可用 | 跳过缓存，直查DB；分布式锁降级为DB行锁 | 响应变慢 | Redis恢复后自动 |
| ser-finance超时 | 返回"支付服务繁忙"，用户重试 | 充值/提现暂不可用 | ser-finance恢复 |
| ser-blog失败 | 忽略，日志记录warn | 无感知 | 自动 |
| ser-s3失败 | 图标URL为空，不阻塞保存 | 币种无图标 | 后续补传 |
| 汇率API失败 | 保持当前汇率，跳过本轮刷新 | 汇率可能不是最新 | 下轮自动重试 |
| DB事务超时 | 自动回滚，返回系统错误 | 操作失败，可重试 | 自动 |

### 15.3 链路隔离设计

```
核心不可变域 (任何外部故障不影响):
  ├─ 余额查询 (仅依赖wallet自身DB+Redis)
  ├─ 兑换操作 (纯内部事务)
  └─ 币种配置 (仅依赖wallet自身DB+Redis+ser-s3)

外部依赖域 (ser-finance故障=阻断):
  ├─ 充值流程
  ├─ 提现流程
  └─ 人工操作

弱依赖域 (故障=降级但不阻断):
  ├─ 审计日志 (ser-blog, oneway)
  ├─ IP校验 (ser-ip, C端fail-open)
  ├─ 错误翻译 (ser-i18n, 回退原始消息)
  └─ 汇率刷新 (三方API, 保持旧值)
```

---

<a id="附录a"></a>
## 附录A 完整调用链路ASCII总图

```
╔══════════════════════════════════════════════════════════════════════════╗
║                           钱包模块调用链路全景图                          ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐           ║
║  │ C端用户  │    │ B端管理  │    │ 定时任务  │    │ 其他服务  │           ║
║  │ App/H5   │    │ 后台    │    │ ser-cron │    │ betting  │           ║
║  └────┬─────┘    └────┬─────┘    └────┬─────┘    │ settle   │           ║
║       │               │               │          │ activity │           ║
║       │HTTP           │HTTP           │RPC       │ user     │           ║
║       ▼               ▼               │          └────┬─────┘           ║
║  ┌─────────┐    ┌─────────┐           │               │                 ║
║  │gate-font│    │gate-back│           │               │RPC              ║
║  │Tracer   │    │Tracer   │           │               │                 ║
║  │Header   │    │Header   │           │               │                 ║
║  │BlackChk │    │WhiteChk │           │               │                 ║
║  │FontAuth │    │BackAuth │           │               │                 ║
║  │Handler  │    │Handler  │           │               │                 ║
║  └────┬────┘    └────┬────┘           │               │                 ║
║       │RPC           │RPC            │               │                 ║
║       └──────┬───────┘               │               │                 ║
║              │                       │               │                 ║
║              ▼                       ▼               ▼                 ║
║  ╔═══════════════════════════════════════════════════════╗              ║
║  ║              ser-wallet (我方核心)                     ║              ║
║  ╠═══════════════════════════════════════════════════════╣              ║
║  ║                                                       ║              ║
║  ║  Handler层 (薄代理)                                   ║              ║
║  ║    │                                                  ║              ║
║  ║    ▼                                                  ║              ║
║  ║  Service层 (业务核心)                                  ║              ║
║  ║    ├─ 参数校验                                        ║              ║
║  ║    ├─ 幂等校验 (Redis SetNX)                          ║              ║
║  ║    ├─ 分布式锁 (Redis SetNX)                          ║              ║
║  ║    ├─ 业务逻辑                                        ║              ║
║  ║    │   ├─ 充值入账 (CreditWallet)                     ║              ║
║  ║    │   ├─ 提现冻结 (FreezeBalance)                    ║              ║
║  ║    │   ├─ 确认扣款 (ConfirmDebit)                     ║              ║
║  ║    │   ├─ 解冻返还 (UnfreezeBalance)                  ║              ║
║  ║    │   ├─ 兑换执行 (Exchange)                         ║              ║
║  ║    │   └─ 余额查询 (GetBalance)                       ║              ║
║  ║    │                                                  ║              ║
║  ║    ▼                                                  ║              ║
║  ║  Repository层 (数据持久化)                             ║              ║
║  ║    ├─ GORM Gen 查询构建                               ║              ║
║  ║    ├─ Transaction 事务管理                             ║              ║
║  ║    ├─ SELECT FOR UPDATE 行锁                          ║              ║
║  ║    └─ INSERT-ONLY 流水                                ║              ║
║  ║    │                                                  ║              ║
║  ║    ▼                                                  ║              ║
║  ║  ┌────────┐  ┌────────┐                               ║              ║
║  ║  │ TiDB   │  │ Redis  │                               ║              ║
║  ║  │ 10张表 │  │ 缓存   │                               ║              ║
║  ║  └────────┘  └────────┘                               ║              ║
║  ╚════════════════════╤══════════════════════════════════╝              ║
║                       │                                                 ║
║          ┌────────────┼────────────────────────┐                       ║
║          │            │                        │                       ║
║          ▼            ▼                        ▼                       ║
║    ┌──────────┐ ┌──────────┐            ┌──────────┐                   ║
║    │ser-user  │ │ser-kyc   │            │ser-blog  │                   ║
║    │用户状态   │ │KYC实名   │            │审计日志   │                   ║
║    │(2 APIs)  │ │(1 API)   │            │(2 APIs)  │                   ║
║    └──────────┘ └──────────┘            └──────────┘                   ║
║          ▲                                                              ║
║          │                                                              ║
║    ┌──────────┐ ┌──────────┐            ┌──────────┐                   ║
║    │ser-s3    │ │ser-buser │            │ser-finance│                   ║
║    │图标上传   │ │操作员查询 │            │支付/财务  │                   ║
║    │(1 API)   │ │(1 API)   │            │(6 APIs)  │                   ║
║    └──────────┘ └──────────┘            │  TODO!   │                   ║
║                                         └────┬─────┘                   ║
║                                              │                         ║
║                                         ┌────▼─────┐                   ║
║                                         │ 三方支付  │                   ║
║                                         │ 渠道API  │                   ║
║                                         └──────────┘                   ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

<a id="附录b"></a>
## 附录B 各链路耗时预估与瓶颈分析

### B.1 各链路预估耗时

| 链路 | 缓存命中 | 缓存未命中 | 瓶颈环节 | 备注 |
|------|---------|-----------|---------|------|
| 余额查询 | ~15ms | ~25ms | DB查询(索引) | 最高频，必须快 |
| 币种列表 | ~10ms | ~20ms | DB查询 | 重度缓存 |
| 充值(创建订单) | N/A | ~150ms | F5 RPC调用 | 含跨模块RPC |
| 充值(入账回调) | N/A | ~50ms | DB事务 | 核心路径 |
| 提现(信息聚合) | N/A | ~100ms | 3个并行RPC | 取最慢的 |
| 提现(冻结+提交) | N/A | ~200ms | 冻结事务+F4 RPC | 最复杂路径 |
| 兑换(预览) | ~15ms | ~25ms | 汇率查询 | 类似余额查询 |
| 兑换(执行) | N/A | ~80ms | 多钱包事务 | 2-3行锁 |
| 投注扣款 | N/A | ~30ms | DB事务 | 高频，需优化 |
| 结算返奖 | N/A | ~30ms | DB事务 | 类似投注 |
| 钱包初始化 | N/A | ~40ms | 批量INSERT | 一次性 |
| 汇率刷新 | N/A | ~3s | 三方HTTP API | 定时任务，不影响用户 |
| 币种配置 | N/A | ~50ms | DB写入+缓存失效 | 低频 |

### B.2 性能热点排行

```
热度排行 (按调用频率):

  Top 1: 余额查询          → 优化: Redis缓存, 30s TTL
  Top 2: 投注扣款          → 优化: 行级锁, 避免表锁
  Top 3: 结算返奖          → 优化: 同投注扣款
  Top 4: 币种列表          → 优化: Redis缓存, 5min TTL
  Top 5: 充值入账(回调)    → 优化: 幂等快速返回
  Top 6: 兑换执行          → 优化: 固定锁序防死锁
  Top 7: 提现冻结          → 优化: 锁粒度控制
  其他: 低频操作           → 无特殊优化需求
```

### B.3 耗时构成分析

```
典型写操作耗时构成:

  ┌─ 网关中间件链 ──────── ~5ms ──── 固定开销
  ├─ RPC网络传输 ────────── ~2ms ──── 内网
  ├─ Redis幂等校验 ──────── ~1ms ──── 快速拦截重复
  ├─ Redis分布式锁 ──────── ~1ms ──── 并发控制
  ├─ DB SELECT FOR UPDATE ─ ~5ms ──── 行锁获取
  ├─ DB UPDATE ────────────── ~3ms ──── 余额更新
  ├─ DB INSERT(流水) ───── ~3ms ──── 流水写入
  ├─ DB COMMIT ────────────── ~2ms ──── 事务提交
  ├─ Redis缓存失效 ──────── ~1ms ──── DEL操作
  ├─ Redis释放锁 ────────── ~1ms ──── 释放操作
  └─ 响应回传 ────────────── ~2ms ──── 网络
  ─────────────────────────────────
  总计:                    ~26ms      (无跨模块RPC)

  + 跨模块RPC (如F5):    +50~100ms   (取决于对方服务)
  + 三方HTTP API:         +500~2000ms (汇率API等)
```

---

> **文档价值**: 本文档从"请求是怎么流转的"这个核心视角，将钱包模块的所有关键操作做了宏观链路拆解。
> 每个链路你都知道：**从哪里开始** → **中间经过哪些步骤** → **最终到达哪里** → **哪里复杂** → **哪里有挑战**。
> 这就是编码前的"指导方针"——先画好地图，再动手修路。
