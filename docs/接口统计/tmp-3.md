# ser-wallet 接口统计清单（整合版）

> 整合时间：2026-02-28
> 整合来源：11份接口评估方案（c-1~c-5、d-1~d-3、tmp-1~tmp-3）+ IDL设计 + RPC暴露分析 + RPC调用分析 + 链路关系 + 需求理解
> 整合原则：多数方案共识 → 确认采纳；少数方案独有 → 标注商榷；全员一致 → 核心必须
> 路由规范：全部 POST 方法（工程惯例），C端前缀 `/api`，B端前缀 `/admin/api`

---

## 总量速览

```
我方主责接口（钱包 + 币种配置）：

  C端 HTTP ........... 16 个（核心13 + 应该有3）
  B端 HTTP ...........  9 个（核心7 + 应该有2）
  RPC 对外暴露 ....... 13 个（核心10 + 应该有3）
  RPC 调用依赖 ....... 12 个（可直接接入6 + 需Mock占位4 + 需确认2）
  定时任务 ...........  1 个（核心1）
  外部 HTTP ..........  1 组（三方汇率API × 3家）

  合计定义实现 ........ 39 个（C16 + B9 + RPC暴露13 + 定时1）
  合计对接依赖 ........ 12 个（RPC调用）+ 1组（HTTP调用）

商榷项 ............... 8 个（需评审确认后决定增/删/调整）
```

---

## 一、C端 HTTP 接口（gate-font → ser-wallet）

> 路由前缀：`POST /api/wallet/...`
> 全部需要用户登录 Token 鉴权（走 Api 路由组）

### 1.1 钱包核心（2个）

```
[核心] POST /api/wallet/home
[核心] POST /api/wallet/currency/list
```

- **`/wallet/home`** — 钱包首页
  获取当前币种下的钱包余额信息：中心钱包余额 + 奖励钱包余额 + 资金钱包合计（中心+奖励）。
  同时返回当前操作币种基本信息（名称、图标、符号）。
  页面入口：个人中心 → 钱包Tab。
  内部调用：币种配置（校验币种状态）+ 钱包余额表（查余额）。
  **11份方案全部认为核心。**

- **`/wallet/currency/list`** — 可用币种列表
  返回所有已启用的币种列表，每个币种附带用户当前余额（有则查，无则显示0）。
  用于币种切换下拉框。当用户切换币种时，前端重新调用 `/wallet/home` 刷新余额。
  内部调用：币种配置（启用币种列表）+ 批量查余额。
  **11份方案全部认为核心。**

### 1.2 充值（4个）

```
[核心]   POST /api/wallet/deposit/methods
[核心]   POST /api/wallet/deposit/create
[核心]   POST /api/wallet/deposit/check
[应该有] POST /api/wallet/deposit/status
```

- **`/wallet/deposit/methods`** — 充值方式列表
  根据当前币种返回可用的充值方式（银行转账、扫码支付、USDT等），每种方式含：
  方式名称/图标/快捷金额档位/金额范围/默认选中方式。
  数据来源：财务模块（充值方式配置），如有 BSB 则附带入款汇率。
  **11份方案全部认为核心。充值流程第一步。**

- **`/wallet/deposit/create`** — 创建充值订单
  统一入口，根据充值方式类型内部分流（USDT / 银行转账 / 扫码支付）。
  流程：校验金额 → 检查重复转账 → 锁定汇率（BSB时）→ 生成平台订单号（C+16位）→ 调用财务模块匹配通道 → 返回支付信息（跳转URL 或 收款地址）。
  如用户选择了奖金活动，一并记录到订单中。
  **11份方案全部认为核心。充值主链路。**

- **`/wallet/deposit/check`** — 重复转账检测
  检查同用户 + 同币种 + 同金额是否存在未完成的充值订单。
  前端在用户输入金额后主动调用，有重复则弹窗警告。
  独立接口（不合并到 create 中），因为需要在提交前给用户选择。
  **7份方案认为核心，4份认为应该有。共识偏核心——独立调用点，前端已有交互设计。**

- **`/wallet/deposit/status`** — 充值订单状态查询
  用户从支付页面返回后，前端轮询此接口查询支付结果。
  返回：订单状态（待支付/支付中/成功/失败/超时）+ 金额 + 到账信息。
  **6份方案认为应该有，5份未单独列出。用户体验需要，但非充值链路核心。**

### 1.3 兑换（2个）

```
[核心] POST /api/wallet/exchange/rate
[核心] POST /api/wallet/exchange/do
```

- **`/wallet/exchange/rate`** — 兑换汇率查询 / 兑换预览
  输入法币金额后，返回：平台汇率 → BSB 到账数量 + 赠送比例 + 赠送金额 + 实际到账合计 + 流水倍数要求。
  BSB 锚定关系：10 BSB = 1 USDT。
  此接口同时承担"兑换预览"功能（部分方案将预览拆为独立接口，综合后合并到此接口）。
  **11份方案全部认为核心（含部分拆为 preview + rate 的合并后）。**

- **`/wallet/exchange/do`** — 执行兑换
  法币 → BSB 单向兑换。
  流程：锁定汇率 → 扣法币中心钱包余额 → 加 BSB 中心钱包余额 → 加 BSB 奖励钱包（赠送部分）→ 记录稽核流水（含流水倍数要求）→ 写流水记录。
  全部在钱包模块内部完成，不依赖外部服务。
  **11份方案全部认为核心。**

### 1.4 提现（4个）

```
[核心]   POST /api/wallet/withdraw/methods
[核心]   POST /api/wallet/withdraw/account
[核心]   POST /api/wallet/withdraw/account/save
[核心]   POST /api/wallet/withdraw/create
```

- **`/wallet/withdraw/methods`** — 提现方式列表
  根据当前币种 + 用户 KYC 状态返回可用提现方式。
  KYC 规则：未认证仅可 USDT 提现，已认证可用全渠道。
  每种方式含：方式名称 / 手续费率 / 固定手续费 / 单笔限额 / 单日限额 / 单日次数限制。
  内部调用：ser-kyc.KycQuery（KYC 状态）+ 财务模块（提现方式配置）。
  **11份方案全部认为核心。提现流程第一步。**

- **`/wallet/withdraw/account`** — 获取提现账户信息
  非首次提现时，返回用户已保存的提现账户信息（银行卡号/电子钱包号/USDT地址等），只读回显。
  **10份方案单独列出（1份合并到 methods 中）。核心共识。**

- **`/wallet/withdraw/account/save`** — 保存提现账户信息
  首次提现时保存银行卡/电子钱包/USDT 提现地址。
  账户持有人 = KYC 认证姓名（自动填充，不可编辑）。
  内部调用：ser-user.GetUserInfoExpend（获取 KYC 姓名）+ ser-kyc.KycQuery（确认已认证）。
  **10份方案单独列出。核心共识。**

- **`/wallet/withdraw/create`** — 创建提现订单
  流程：二次校验 KYC 状态 → 校验提现限额（单笔/单日金额/次数）→ 校验余额充足 → 冻结余额（FreezeBalance）→ 生成订单号（T+16位）→ 提交财务审核流程。
  提交后订单状态 = 待审核，用户钱包可用余额减少、冻结余额增加。
  **11份方案全部认为核心。提现主链路。**

### 1.5 记录查询（2个）

```
[核心] POST /api/wallet/record/list
[核心] POST /api/wallet/record/detail
```

- **`/wallet/record/list`** — 交易记录列表
  按 Tab 类型分页查询：充值记录 / 提现记录 / 兑换记录 / 奖励记录。
  严格按当前操作币种隔离，每个 Tab 独立查询。
  返回：订单号 / 金额 / 状态 / 时间 / 方式 等摘要字段。
  **11份方案全部认为核心。**

- **`/wallet/record/detail`** — 交易记录详情
  单条记录详情：订单号 / 金额明细 / 状态（含失败原因）/ 时间 / 方式 / 手续费 / 汇率快照等。
  **7份方案认为核心，4份认为应该有。综合判定：核心——详情页是基本的产品能力。**

### 1.6 奖金活动（1个）

```
[应该有] POST /api/wallet/bonus/list
```

- **`/wallet/bonus/list`** — 可用奖金活动列表
  充值时展示可叠加的额外奖金活动选项。
  每个活动含：活动名称 / 奖金比例（如充100送10%）/ 最低充值门槛 / 奖金上限 / 流水倍数要求。
  数据来源：活动模块 RPC（服务名待定，当前返回空列表亦不影响充值主流程）。
  **6份方案认为核心，5份认为应该有。综合判定：应该有——数据来源于待定模块，充值不依赖此接口。**

### 1.7 提现订单状态（1个）

```
[应该有] POST /api/wallet/withdraw/status
```

- **`/wallet/withdraw/status`** — 提现订单状态查询
  用户提交提现后，在记录列表点击可查看审核进度（待风控审核/待财务审核/出款中/已完成/已驳回）。
  **5份方案单独列出为应该有，6份归入记录详情中。综合判定：应该有——可增强用户体验，但可降级到 record/detail 中返回。**

### C端小计

```
核心必须 .... 13 个
应该有 ......  3 个（deposit/status、bonus/list、withdraw/status）
合计 ........ 16 个
```

---

## 二、B端 HTTP 接口（gate-back → ser-wallet）

> 路由前缀：`POST /admin/api/wallet/...`
> 全部需要后台 Token + RBAC 权限鉴权（走 Api 路由组）
> 所有写操作均需调用 ser-blog.AddActLog() 记录审计日志

### 2.1 币种配置管理（5个）

```
[核心] POST /admin/api/wallet/currency/page
[核心] POST /admin/api/wallet/currency/edit
[核心] POST /admin/api/wallet/currency/status
[核心] POST /admin/api/wallet/currency/setBase
[核心] POST /admin/api/wallet/rate/log/page
```

- **`/wallet/currency/page`** — 币种列表分页查询
  返回所有币种（含未启用），每条含 20+ 字段：
  币种代码 / 名称 / 类型（法币/虚拟币）/ 图标 / 状态 / 精度 / 千分位规则 /
  汇率浮动% / 偏差阈值% / 实时汇率 / 平台汇率 / 入款汇率 / 出款汇率 / 当前偏差值 /
  创建人 / 创建时间 / 更新人 / 更新时间。
  支持筛选：币种代码 / 类型 / 状态。
  **11份方案全部认为核心。B端首页。**

- **`/wallet/currency/edit`** — 编辑币种
  弹窗表单编辑：汇率浮动% / 偏差阈值% / 图标（上传 SVG/PNG ≤ 5KB）/ 精度规则 / 千分位规则 / 状态。
  不可编辑项：币种代码 / 币种类型（创建后锁定）。
  图标上传调用 ser-s3.UploadIcon。
  **11份方案全部认为核心。**

- **`/wallet/currency/status`** — 启用/禁用币种
  切换币种的启用/禁用状态。
  启用 → 币种出现在 C 端可用列表。
  禁用 → 需检查是否有用户持有该币种余额（有则提示"仍有 N 位用户持有余额"，可强制禁用）。
  基准币种不可禁用。
  **11份方案全部认为核心。**

- **`/wallet/currency/setBase`** — 设置基准币种
  一次性操作，设置后不可更改。推荐设置 USDT 为基准币种。
  设置时需二次确认弹窗。设置完成后页面上"设置基准币种"按钮隐藏。
  BSB 锚定关系：10 BSB = 1 USDT（基准币种确定后此比例固定）。
  **11份方案全部认为核心。**

- **`/wallet/rate/log/page`** — 汇率日志分页查询
  操作日志页面中的"汇率日志"Tab。
  每次平台汇率变更自动生成一条日志：变更前汇率 / 变更后汇率 / 触发方式（定时/手动）/ 变更时间。
  支持筛选：日期范围 / 币种代码。
  **11份方案全部认为核心。**

### 2.2 用户钱包管理（2个）

```
[核心] POST /admin/api/wallet/user/page
[核心] POST /admin/api/wallet/user/detail
```

- **`/wallet/user/page`** — B端用户钱包分页查询
  客服 / 风控 / 运营查看用户钱包余额的入口。
  列表字段：UID / 昵称 / 币种 / 中心钱包余额 / 奖励钱包余额 / 冻结金额 / 注册时间。
  用户昵称通过 ser-user.BatchGetUserInfo 批量获取。
  支持筛选：UID / 币种。
  **7份方案认为核心（列为 B 端必须项），4份认为应该有。综合判定：核心——客服/风控场景的基本需求。**

- **`/wallet/user/detail`** — B端用户钱包详情
  按币种 + 钱包类型展开，显示各子钱包（中心/奖励/主播/代理）的可用余额和冻结余额。
  可附带最近交易记录摘要。
  操作人姓名通过 ser-buser.QueryUserByIds 获取。
  **6份方案认为核心，5份归入 page 的子功能。综合判定：核心——点击列表行进入详情是标准交互。**

### 2.3 应该有（2个）

```
[应该有] POST /admin/api/wallet/rate/log/export
[应该有] POST /admin/api/wallet/currency/detail
```

- **`/wallet/rate/log/export`** — 汇率日志导出
  将汇率日志导出为 CSV/Excel 文件。参照工程惯例 `QueryXxxCsv` 方法。
  **5份方案列出，6份未提及。应该有——审计需求。**

- **`/wallet/currency/detail`** — 币种详情查询
  编辑弹窗回显时调用（也可复用 page 列表中的行数据，不独立调用）。
  **4份方案列出为应该有，7份未单独列出。应该有——取决于前端交互设计。**

### B端小计

```
核心必须 .... 7 个
应该有 ...... 2 个（rate/log/export、currency/detail）
合计 ........ 9 个
```

---

## 三、RPC 对外暴露接口（ser-wallet 作为 Server）

> 无 HTTP 路由，由 Kitex 框架监听独立端口（如 8022）直接提供 RPC 服务
> 调用方通过 `rpc.WalletClient().XxxMethod(ctx, req)` 调用
> 所有写操作必须支持幂等（orderNo 唯一键），金额统一使用 i64 最小单位

### 3.1 余额变更类（5个，供财务模块调用）

```
[核心] CreditWallet(uid, currencyCode, walletType, amount, orderNo, bizType)
[核心] FreezeBalance(uid, currencyCode, amount, orderNo)
[核心] UnfreezeBalance(uid, currencyCode, amount, orderNo)
[核心] ConfirmDebit(uid, currencyCode, amount, orderNo)
[核心] DebitWallet(uid, currencyCode, amount, orderNo, bizType)
```

- **CreditWallet** — 上账（加款）
  充值成功后，财务模块回调触发，将金额入账到用户指定币种的中心钱包。
  调用方：财务模块（充值回调确认后）。
  幂等：orderNo 唯一键，重复调用返回已入账结果。
  **11份方案全部认为核心。资金链路最核心的接口。**

- **FreezeBalance** — 冻结余额
  用户提交提现申请时，将对应金额从可用余额转移到冻结余额。
  调用方：ser-wallet 内部（CreateWithdrawOrder 流程中调用）或财务模块。
  **11份方案全部认为核心。**

- **UnfreezeBalance** — 解冻退回
  提现被驳回（风控/财务任一阶段）或出款失败时，将冻结金额退回可用余额。
  调用方：财务模块。
  **11份方案全部认为核心。**

- **ConfirmDebit** — 确认扣除冻结
  提现出款成功后，从冻结余额中最终扣除。资金离开用户钱包。
  调用方：财务模块（三方通道出款成功回调后）。
  状态机约束：同一 orderNo，ConfirmDebit 和 UnfreezeBalance 互斥，只能执行其一。
  **11份方案全部认为核心。**

- **DebitWallet** — 扣款
  消费/投注时从用户钱包扣减余额。
  扣款优先级：中心钱包优先 → 奖励钱包补足（需记录比例，返奖时按此比例分配）。
  调用方：财务模块（人工减款审核通过）、投注模块（下注扣款）。
  **11份方案全部认为核心。**

### 3.2 人工修正类（3个，供财务模块调用）

```
[核心] ManualCredit(uid, currencyCode, walletType, amount, orderNo, reason)
[核心] ManualDebit(uid, currencyCode, amount, orderNo, reason)
[核心] SupplementCredit(uid, currencyCode, amount, bonusAmount, orderNo)
```

- **ManualCredit** — 人工加款
  财务 B 端"人工加款"审核通过后触发。指定钱包类型（中心/奖励）加款。
  订单号前缀：A+16位。
  调用方：财务模块。
  **11份方案全部认为核心。**

- **ManualDebit** — 人工减款
  财务 B 端"人工减款"审核通过后触发。
  减款逻辑：如果指定钱包余额不足，按余额从高到低的钱包类型依次扣减（级联扣减）。
  订单号前缀：M+16位。
  调用方：财务模块。
  **11份方案全部认为核心。**

- **SupplementCredit** — 充值补单
  财务 B 端"充值补单"审核通过后触发。
  同时入账两笔：充值金额 → 中心钱包 + 赠送金额 → 奖励钱包（如有）。
  订单号前缀：B+16位。
  调用方：财务模块。
  **9份方案认为核心，2份未单独列出（合并到 CreditWallet）。综合判定：核心——补单含赠送逻辑，与普通上账不同。**

### 3.3 查询类（2个，多方调用）

```
[核心]   GetBalance(uid, currencyCode)
[应该有] BatchGetBalance(uids, currencyCode)
```

- **GetBalance** — 余额查询
  返回指定用户指定币种下各子钱包的可用余额 + 冻结余额。
  调用方：财务（人工修正表单展示余额）、投注（下注前校验）、直播（送礼前校验）、活动（发奖前查询）。
  高频读接口，建议走 Redis 缓存。
  **11份方案全部认为核心。调用方最多的接口。**

- **BatchGetBalance** — 批量余额查询
  一次查询多个用户的余额，上限 100 个 uid。
  调用方：财务（统计/对账）、报表模块。
  **6份方案列出为应该有，5份未单独列出。综合判定：应该有——批量场景确实存在，但非核心链路。**

### 3.4 返奖入账类（1个，供活动模块调用）

```
[核心] RewardCredit(uid, currencyCode, amount, orderNo, activityInfo)
```

- **RewardCredit** — 活动/任务返奖入账
  活动结算后，将奖金入账到用户奖励钱包。
  activityInfo 含：活动ID / 流水倍数要求（稽核流水用）。
  如用户没有该币种的奖励钱包 → 自动创建。
  调用方：活动/任务模块。
  **8份方案认为核心，3份认为应该有。综合判定：核心——活动返奖是明确的产品流程。**

### 3.5 币种数据类（2个，供多方调用）

```
[应该有] GetCurrencyConfig(currencyCode)
[应该有] GetExchangeRateRpc(currencyCode, direction)
```

- **GetCurrencyConfig** — 币种配置查询（RPC版）
  返回币种完整配置：精度 / 符号 / 千分位规则 / 状态 / 类型 / 图标。
  调用方：财务模块（配置充提方式时需要币种信息）、其他模块。
  **5份方案列出为应该有，6份归入内部调用。综合判定：应该有——财务模块高概率需要。**

- **GetExchangeRateRpc** — 汇率查询（RPC版）
  返回指定币种的平台汇率 / 入款汇率 / 出款汇率，纯数据无展示信息。
  调用方：财务模块（订单金额计算时需要汇率）。
  **5份方案列出为应该有。综合判定：应该有——与 C端 GetExchangeRate 区别在于无展示信息、无赠送计算。**

### RPC对外暴露小计

```
核心必须 .... 10 个
应该有 ......  3 个（BatchGetBalance、GetCurrencyConfig、GetExchangeRateRpc）
合计 ........ 13 个
```

---

## 四、RPC 调用依赖（ser-wallet 作为 Client）

> 以下是 ser-wallet 内部代码中需要 `rpc.XxxClient().Method()` 调用其他服务的接口
> 按可用状态分三批：可直接接入 → 需 Mock 占位 → 需确认接入方式

### 4.1 第一批：可直接接入（IDL 已定义、Client 已注册，6个）

```
① ser-blog   → AddActLog(backend, model, page, action, isSuccess)
② ser-kyc    → KycQuery(uid)
③ ser-user   → GetUserInfoExpend(uid)
④ ser-user   → BatchGetUserInfo(uids)
⑤ ser-s3     → UploadIcon(bsType, content) 或 UploadFile(...)
⑥ ser-buser  → QueryUserByIds(ids)
```

- **① ser-blog.AddActLog** — B端操作审计日志
  oneway void（异步，不等响应），所有 B 端写操作后调用。
  5个字段：后台名称 / 模块名称 / 页面名称 / 操作描述（动态拼接）/ 成功标记。
  失败不中断主业务，仅打 Error 日志。
  使用场景：B1~B7 所有写操作。
  参考代码：ser-app `addBannerLog()` helper 封装模式。
  **11份方案全部认为核心。工程规范强制要求。**

- **② ser-kyc.KycQuery** — KYC 认证状态查询
  提现链路前置校验：status=3（已通过）才允许提现。
  返回 UserKycInfo：status / name / country / idType 等。
  使用场景：C9 提现方式列表 / C11 保存提现账户 / C12 创建提现订单。
  容错要求：KYC 服务不可用时，提现接口必须返回明确错误码，绝不默认放行。
  **11份方案全部认为核心。**

- **③ ser-user.GetUserInfoExpend** — 获取用户信息
  提现账户保存时获取 KYC 认证姓名，自动填充"账户持有人"字段。
  也可用于 B 端人工加/减款前确认用户存在且未封禁。
  注意：此方法在工程中尚无 service 层调用实例，ser-wallet 将是首个使用者。
  使用场景：C11 保存提现账户 / B3-B4 人工加减款（可选）。
  **9份方案认为核心，2份认为应该有。综合判定：核心。**

- **④ ser-user.BatchGetUserInfo** — 批量获取用户昵称
  B 端用户钱包列表中，钱包表只存 uid，需批量查询用户昵称用于展示。
  返回 BatchUserInfoResp（list + total）。
  使用场景：B6 用户钱包列表。
  **5份方案列出，6份未单独列出。综合判定：应该有——B 端展示需求。**

- **⑤ ser-s3.UploadIcon / UploadFile** — 币种图标上传
  B 端编辑币种时上传 SVG/PNG 图标文件（≤ 5KB）。
  注意：IDL 中没有 `GetPresignUrl`，正确方法名是 `GetUploadPreSign`（大文件场景）或 `UploadIcon`（小文件图标场景）。
  使用场景：B2 编辑币种。
  参考代码：ser-item `UploadIcon` 调用模式。
  **9份方案列出为应该有。综合判定：应该有——图标上传不影响核心链路但产品要求有图标。**

- **⑥ ser-buser.QueryUserByIds** — B端操作人姓名回显
  B 端列表中 create_by / update_by 字段需要显示操作人名称。
  入参 `ids []int64`，返回 `map[int64]string`（id → 用户名）。
  使用场景：B1 币种列表 / B5 汇率日志。
  参考代码：ser-item `domain/user_service.go` 批量查模式。
  **7份方案列出。综合判定：应该有——B 端展示规范。**

### 4.2 第二批：需 Mock 占位（服务名/IDL 待定，4个）

```
⑦ 财务模块(待定) → GetPaymentConfig(currencyCode, type)
⑧ 财务模块(待定) → GetWithdrawConfig(currencyCode)
⑨ 财务模块(待定) → CreatePayOrder(orderInfo)
⑩ 活动模块(待定) → GetAvailableBonuses(uid, currencyCode)
```

- **⑦ 财务.GetPaymentConfig** — 获取充值方式配置
  C 端充值方式列表的数据来源。财务模块维护充值方式、档位、通道关联。
  阻塞等级：★★★★★（最高）——没有此接口，C3 充值方式列表无数据可展示。
  **11份方案全部认为核心依赖。**

- **⑧ 财务.GetWithdrawConfig** — 获取提现方式配置
  C 端提现方式列表的数据来源。财务模块维护提现方式、费率、限额。
  阻塞等级：★★★★★——没有此接口，C9 提现方式列表无数据可展示。
  **11份方案全部认为核心依赖。**

- **⑨ 财务.CreatePayOrder** — 创建通道订单
  C 端创建充值订单时，调用财务模块匹配通道并创建代收订单。
  返回支付跳转 URL 或收款信息。
  阻塞等级：★★★★★——没有此接口，充值流程无法完成。
  **11份方案全部认为核心依赖。**

- **⑩ 活动.GetAvailableBonuses** — 获取可用奖金活动
  C 端充值页面展示可选奖金活动。
  阻塞等级：★★☆☆☆（较低）——返回空列表不影响充值主流程。
  **6份方案认为应该有，5份归入可选。综合判定：应该有。**

### 4.3 第三批：需确认接入方式（2个）

```
⑪ ser-cron      → 定时任务注册（IDL 空壳，service 定义为空）
⑫ 风控模块(待定) → RiskCheck(uid, amount, type)
```

- **⑪ ser-cron — 定时任务注册**
  汇率定时刷新需要定时调度。但 ser-cron 的 IDL 是空壳（`service CronService {}` 无任何方法）。
  建议方案：ser-wallet 内置 cron（Go robfig/cron 库）+ 分布式锁防多实例重复执行。
  **需与 ser-cron 负责人确认接入机制。**

- **⑫ 风控.RiskCheck — 大额提现风控校验**
  需求提到"大额提现风控校验"，但风控模块服务名、IDL 均未定义。
  当前版本建议：钱包内部实现基础限额校验（每日限额/单笔限额/次数），预留 riskCheck() 方法占位。
  **7份方案标为可选/预留。综合判定：当前版本不接入。**

### 4.4 非 RPC 依赖（不计入接口数）

```
• ser-i18n  → Redis Hash 直读（框架层已集成 ret.BizErr 机制，只需定义错误码常量）
• ETCD      → 服务发现（SDK 直读，框架自动处理）
• Redis     → 缓存汇率 / 余额 / 分布式锁
```

### RPC调用依赖小计

```
可直接接入 ...... 6 个（ser-blog / ser-kyc / ser-user×2 / ser-s3 / ser-buser）
需 Mock 占位 .... 4 个（财务×3 / 活动×1）
需确认方式 ...... 2 个（ser-cron / 风控）
合计 ........... 12 个
```

---

## 五、定时任务（1个）

```
[核心] RefreshExchangeRate — 汇率定时刷新
```

- **RefreshExchangeRate** — 汇率偏差检测与更新
  触发频率：可配置（如每 1~5 分钟）。
  流程：查询所有启用的非基准币种 → 对每个币种并行调用 3 家三方汇率 API（HTTP）→ 取平均值 → 计算与当前平台汇率的偏差 → 偏差 ≥ 阈值% 则更新平台汇率（同时更新入款/出款汇率）→ 写入汇率变更日志 → 更新 Redis 缓存。
  容错要求：3 家 API 至少 2 家成功才视为有效；全失败则保持当前汇率不变 + 触发告警。
  实现方式：内置 cron + Redis 分布式锁（避免多实例重复执行）。
  **11份方案全部认为核心。多币种钱包的基础数据来源。**

---

## 六、外部 HTTP 调用（1组）

```
[核心] 三方汇率 API × 3 家（HTTP GET，非 RPC）
```

- 定时任务 RefreshExchangeRate 中调用，3 家取平均值。
- 需独立封装 `ExchangeRateProvider` interface，每家 API 实现一个 provider。
- 具体 API 选型需与产品/运营确认（常见：ExchangeRate-API / Open Exchange Rates / CurrencyLayer）。
- 超时设置 3~5 秒/家，互不阻塞。

---

## 七、商榷项（需评审确认，8个）

> 以下事项在 11 份方案中存在分歧或依赖外部确认，编码前需在评审会上明确

### 7.1 接口归属类（3个）

**① 充值回调接口归属**
- 部分方案认为充值回调（三方支付通知）应由钱包接收
- 部分方案认为应由财务模块接收，财务处理完后再 RPC 调用 wallet.CreditWallet 上账
- 影响范围：是否需要在 ser-wallet 中新增 DepositNotify 接口
- 当前判定：**倾向归财务模块**（回调涉及通道验签、订单匹配等财务职责），钱包只提供 CreditWallet RPC

**② 充值订单数据归属**
- 充值订单表建在 ser-wallet 库还是财务模块库？
- 如果归钱包：C端记录查询直接读本地库，但财务需跨服务查订单
- 如果归财务：C端记录查询需 RPC 调财务获取充值记录
- 影响范围：C13 记录列表的数据来源 + C4 创建订单的写入位置
- 当前判定：**倾向归钱包**（记录查询是钱包核心功能，本地读更高效）

**③ 提现账户存储位置**
- 存在 ser-wallet 库还是 ser-user 库？
- 如果归钱包：提现账户和钱包数据同库，事务简单
- 如果归用户：需 RPC 调 ser-user 读写，但用户信息集中管理
- 影响范围：C10/C11 接口的实现方式
- 当前判定：**倾向归钱包**（提现账户与钱包强关联，同库事务可靠）

### 7.2 接口增减类（3个）

**④ USDT 充值地址是否独立接口**
- 4 份方案拆出独立接口 `GetUSDTDepositInfo`（获取收款地址 + 链网络 + QR码）
- 7 份方案合并到 `CreateDepositOrder` 的返回值中
- 影响范围：是否新增 1 个 C 端接口
- 当前判定：**合并到 CreateDepositOrder 返回值**（充值方式为 USDT 时返回收款信息，无需独立接口）

**⑤ 币种新增接口是否需要**
- 5 份方案列出 `CreateCurrency` 作为独立接口
- 6 份方案认为币种是预置数据，只需 EditCurrency 编辑
- 影响范围：是否新增 1 个 B 端接口
- 当前判定：**需评审确认**——如果币种列表是运营手动逐个添加，则需要 CreateCurrency；如果是系统预置全球币种列表仅开关控制，则不需要

**⑥ 提现账户修改/删除能力**
- 3 份方案列出编辑提现账户、删除提现账户接口
- 8 份方案认为首次保存后不可自行修改（需联系客服）
- 影响范围：是否新增 1~2 个 C 端接口
- 当前判定：**倾向不可自行修改**（金融安全考虑，防止账户被篡改用于洗钱）

### 7.3 模块边界类（2个）

**⑦ 场馆钱包 / 代理钱包是否当前版本实现**
- 需求文档标注"预留"
- 如果实现：RPC 暴露接口增加 3~4 个（AgentIncome / VenueTransfer / TransferBetweenWallets）
- 影响范围：钱包子类型数量和 RPC 暴露接口数量
- 当前判定：**当前版本不实现**，数据模型上预留 walletType 字段即可

**⑧ 兑换赠送比例和流水倍数配置归属**
- 兑换赠送比例（如法币→BSB 赠送10%）由谁配置？
- 是 B 端币种配置中的字段？还是活动模块配置？
- 影响范围：EditCurrency 接口字段 + DoExchange 的赠送计算逻辑
- 当前判定：**需评审确认**——倾向归币种配置（与汇率同级的基础数据）

---

## 八、完整接口清单（按 REST 路径排序）

### 8.1 C端接口（16个）

```
POST /api/wallet/home                    [核心]   钱包首页余额信息
POST /api/wallet/currency/list           [核心]   可用币种列表
POST /api/wallet/deposit/methods         [核心]   充值方式列表
POST /api/wallet/deposit/create          [核心]   创建充值订单
POST /api/wallet/deposit/check           [核心]   重复转账检测
POST /api/wallet/deposit/status          [应该有] 充值订单状态查询
POST /api/wallet/bonus/list              [应该有] 可用奖金活动列表
POST /api/wallet/exchange/rate           [核心]   兑换汇率查询/预览
POST /api/wallet/exchange/do             [核心]   执行兑换
POST /api/wallet/withdraw/methods        [核心]   提现方式列表
POST /api/wallet/withdraw/account        [核心]   获取提现账户信息
POST /api/wallet/withdraw/account/save   [核心]   保存提现账户信息
POST /api/wallet/withdraw/create         [核心]   创建提现订单
POST /api/wallet/withdraw/status         [应该有] 提现订单状态查询
POST /api/wallet/record/list             [核心]   交易记录列表
POST /api/wallet/record/detail           [核心]   交易记录详情
```

### 8.2 B端接口（9个）

```
POST /admin/api/wallet/currency/page     [核心]   币种列表分页查询
POST /admin/api/wallet/currency/edit     [核心]   编辑币种
POST /admin/api/wallet/currency/status   [核心]   启用/禁用币种
POST /admin/api/wallet/currency/setBase  [核心]   设置基准币种
POST /admin/api/wallet/currency/detail   [应该有] 币种详情查询
POST /admin/api/wallet/rate/log/page     [核心]   汇率日志分页查询
POST /admin/api/wallet/rate/log/export   [应该有] 汇率日志导出
POST /admin/api/wallet/user/page         [核心]   B端用户钱包列表
POST /admin/api/wallet/user/detail       [核心]   B端用户钱包详情
```

### 8.3 RPC 对外暴露接口（13个）

```
[核心]   CreditWallet         充值成功上账 → 中心钱包
[核心]   DebitWallet           消费/投注扣款（中心优先→奖励）
[核心]   FreezeBalance         提现申请冻结余额
[核心]   UnfreezeBalance       提现驳回/失败解冻退回
[核心]   ConfirmDebit          提现出款成功确认扣除
[核心]   ManualCredit          人工加款（A+单号）
[核心]   ManualDebit           人工减款（M+单号，级联扣减）
[核心]   SupplementCredit      充值补单（B+单号，含赠送）
[核心]   GetBalance            余额查询（高频，走缓存）
[核心]   RewardCredit          活动/任务返奖 → 奖励钱包
[应该有] BatchGetBalance       批量余额查询（≤100人）
[应该有] GetCurrencyConfig     币种配置查询（RPC版）
[应该有] GetExchangeRateRpc    汇率查询（RPC版，纯数据）
```

### 8.4 RPC 调用依赖（12个）

```
可直接接入：
  ① ser-blog.AddActLog                 [核心]   B端审计日志（oneway）
  ② ser-kyc.KycQuery                   [核心]   提现KYC状态校验
  ③ ser-user.GetUserInfoExpend          [核心]   用户信息/KYC姓名
  ④ ser-user.BatchGetUserInfo           [应该有] B端批量用户昵称
  ⑤ ser-s3.UploadIcon                  [应该有] 币种图标上传
  ⑥ ser-buser.QueryUserByIds           [应该有] B端操作人姓名

需Mock占位：
  ⑦ 财务.GetPaymentConfig              [核心]   充值方式配置（★★★★★阻塞）
  ⑧ 财务.GetWithdrawConfig             [核心]   提现方式配置（★★★★★阻塞）
  ⑨ 财务.CreatePayOrder                [核心]   创建通道订单（★★★★★阻塞）
  ⑩ 活动.GetAvailableBonuses           [应该有] 可用奖金活动

需确认：
  ⑪ ser-cron / 内置cron                [核心]   定时任务（建议内置）
  ⑫ 风控.RiskCheck                     [预留]   大额提现风控（当前版本不接入）
```

### 8.5 定时任务 + 外部 HTTP（2个）

```
[核心] RefreshExchangeRate               汇率定时刷新（内置cron + 分布式锁）
[核心] 三方汇率API × 3家                 HTTP调用（ExchangeRateProvider interface）
```

---

## 九、按开发优先级排序

### P0 — 第一批（骨架阶段，纯内部逻辑，不依赖外部服务）

```
B端：
  /wallet/currency/page          币种列表
  /wallet/currency/edit          编辑币种
  /wallet/currency/status        启禁用币种
  /wallet/currency/setBase       设置基准币种
  /wallet/rate/log/page          汇率日志

C端：
  /wallet/home                   钱包首页
  /wallet/currency/list          币种列表
  /wallet/exchange/rate          汇率查询
  /wallet/exchange/do            执行兑换
  /wallet/record/list            记录列表
  /wallet/record/detail          记录详情

RPC暴露：
  CreditWallet / DebitWallet / FreezeBalance / UnfreezeBalance /
  ConfirmDebit / GetBalance（余额核心操作六件套）

定时任务：
  RefreshExchangeRate            汇率刷新（先Mock汇率API）
```

### P1 — 第二批（接入已有 IDL 的外部服务）

```
C端（依赖 ser-kyc / ser-user）：
  /wallet/withdraw/methods       提现方式（→ ser-kyc）
  /wallet/withdraw/account       获取提现账户
  /wallet/withdraw/account/save  保存提现账户（→ ser-user / ser-kyc）
  /wallet/withdraw/create        创建提现订单（→ ser-kyc）

B端（依赖 ser-blog / ser-buser / ser-s3）：
  /wallet/user/page              用户钱包列表（→ ser-user）
  /wallet/user/detail            用户钱包详情（→ ser-buser）

RPC暴露：
  ManualCredit / ManualDebit / SupplementCredit（人工修正三件套）
  RewardCredit（活动返奖）
```

### P2 — 第三批（Mock 财务模块，跑通充提主链路）

```
C端（依赖财务模块 Mock）：
  /wallet/deposit/methods        充值方式（→ Mock 财务）
  /wallet/deposit/create         创建充值（→ Mock 财务）
  /wallet/deposit/check          重复转账检测
  /wallet/withdraw/create        提现下单完整链路（→ Mock 财务）

应该有接口：
  /wallet/deposit/status         充值状态
  /wallet/bonus/list             奖金活动（→ Mock 活动 / 返空）
  /wallet/withdraw/status        提现状态

RPC暴露（应该有）：
  BatchGetBalance / GetCurrencyConfig / GetExchangeRateRpc
```

### P3 — 第四批（财务模块就绪后替换 Mock，联调完整链路）

```
替换 Mock → Real：
  财务.GetPaymentConfig → 真实 RPC
  财务.GetWithdrawConfig → 真实 RPC
  财务.CreatePayOrder → 真实 RPC

补充（如评审确认）：
  /wallet/currency/detail        币种详情
  /wallet/rate/log/export        汇率日志导出
```

---

## 十、与 IDL 设计的对应关系

```
IDL service.thrift 中需声明的方法数：

  C端接口 ............ 16 个方法
  B端接口 ............  9 个方法
  RPC对外暴露 ........ 13 个方法
  ─────────────────────────────
  合计 ............... 38 个方法

IDL 文件拆分：
  common/idl/ser-wallet/
  ├── service.thrift      38 个方法声明（含api/method注解）
  ├── wallet.thrift       钱包核心类型（余额/流水/子钱包）
  ├── currency.thrift     币种配置类型（币种/汇率/日志）
  ├── deposit.thrift      充值类型（订单/方式/档位）
  ├── withdraw.thrift     提现类型（订单/方式/账户）
  ├── exchange.thrift     兑换类型（预览/订单）
  ├── record.thrift       记录查询类型（列表/详情）
  └── wallet_rpc.thrift   RPC对外类型（供其他模块引用）

需新增的工程文件：
  namesp.go          → EtcdWalletService = "wallet_service"
  rpc_client.go      → WalletClient() 单例工厂
  gate-font/router/  → wallet_router.go（16个C端路由）
  gate-back/router/  → wallet_router.go（9个B端路由）
  register.go ×2     → 各加一行 wallet.RouterWallet(r)
```

---

## 十一、数量对比（11份方案 vs 整合版）

| 方案 | C端 | B端 | RPC暴露 | RPC调用 | 合计 |
|------|-----|-----|---------|---------|------|
| c-1（第一版） | 23 | 7 | — | — | ~30 |
| c-1（迭代版） | 22 | 5 | 12 | 9 | ~48 |
| c-2（迭代版） | 20 | 2 | 9 | 9 | ~40 |
| c-3（精细版） | 27 | 10 | 6 | 10 | ~53 |
| c-4（迭代版） | 15 | 6 | 8 | 4 | ~33 |
| c-5（精细版） | 21 | 12 | 12 | 8 | ~53 |
| d-1（迭代版） | 22 | 11 | 8 | 10 | ~51 |
| d-2（迭代版） | 15 | 6 | 8 | 4 | ~33 |
| d-3（精细版） | 21 | 11 | 18 | 10 | ~60 |
| tmp-1（精细版） | 16 | 8 | 13 | 7 | ~44 |
| tmp-2（迭代版） | 19 | 6 | 12 | 6 | ~43 |
| tmp-3（迭代版） | 18 | 9 | 17 | 12 | ~56 |
| **整合版** | **16** | **9** | **13** | **12** | **50** |

**整合版取值逻辑：**
- C端 16 = 取各方案高频共识接口，合并重复（兑换预览+汇率查询合并为1个、提现账户获取+保存拆为2个）
- B端 9 = 币种配置5 + 汇率日志1 + 日志导出1 + 用户钱包2，删去少数方案独有的"币种新增"
- RPC暴露 13 = 核心10（全员共识）+ 应该有3（高频提及）
- RPC调用 12 = 可接入6 + Mock4 + 确认2
