# ser-wallet 接口统计整合版（定稿参考）

> 整合时间：2026-02-28
> 整合来源：
> - 接口评估 11 份迭代文档（/Users/mac/gitlab/z-readme/接口评估/）
> - RPC 暴露分析（/Users/mac/gitlab/z-readme/rpc暴露/tmp-2.md）
> - RPC 调用分析（/Users/mac/gitlab/z-readme/rpc调用/tmp-2.md）
> - 需求分析（/Users/mac/gitlab/z-readme/result/tmp-2.md，152 张原型图提取）
> - 工程分析（/Users/mac/gitlab/z-readme/存储会话/c-2.md、c-3.md）
>
> 本文定位：将多轮评估的分歧和重叠收敛为**一份确定性清单**，作为进入编码前的最终接口参照。
>
> 统计口径说明：
> - **仅统计我方主责接口**（钱包模块 + 币种配置模块）
> - 财务管理模块（他方负责）单独列出供联动参考，不计入主责统计
> - 定时任务和外部 HTTP 调用单独计入
> - 优先级标记：**Must**（核心必须）/ **Should**（应该有）/ **Could**（可选/预留）

---

## 全局数字总览

```
我方主责接口统计
═══════════════════════════════════════════
  C端 HTTP 接口（gate-font）：  19 个
  B端 HTTP 接口（gate-back）：   6 个
  RPC 对外暴露（供他方调用）：  12 个
  RPC 主动调用（调用他方）  ：   6 个（确定） + 5 个（工程规范/待确认）
  定时任务 / 内部处理      ：   3 个
  ─────────────────────────────────────
  主责合计                 ：  46 个（确定性骨架）

优先级分布
═══════════════════════════════════════════
  Must（核心必须）：  27 个
  Should（应该有）：  14 个
  Could（可选预留）：  5 个
```

---
---

## 第一部分：C 端 HTTP 接口（19 个）

> 面向终端用户（iOS / Android / H5），经 gate-font 网关路由到 ser-wallet
> 路由规范：`POST /api/wallet/xxx`（需登录鉴权），`POST /open/wallet/xxx`（免鉴权）
> 需求依据：需求文档第三章（3.2~3.10）

---

### 1.1 钱包首页（2 个）

**C-1 获取钱包首页信息** `POST /api/wallet/overview` — **Must**
- 返回当前币种下各子钱包余额（中心钱包、奖励钱包、主播钱包、代理钱包）
- 资金钱包 = 中心 + 奖励合计
- 需求依据：3.2 币种结构 + 3.3 币种切换规则

**C-13 获取当前汇率信息** `POST /api/wallet/rate/current` — **Must**
- 返回当前币种的入款汇率 / 出款汇率 / BSB 换算比
- 充值、提现、兑换页面均需展示汇率信息
- 需求依据：3.4.1 "BSB 显示汇率" + 3.7 "统一格式" + 3.8.5 "BSB 提现汇率"

---

### 1.2 充值相关（5 个）

**C-2 获取充值方式列表** `POST /api/wallet/deposit/methods` — **Must**
- 按当前币种过滤可用充值方式（方式名称 / 图标 / 快捷档位 / 金额范围）
- 数据来源：调用财务模块 RPC GetPaymentMethods
- 需求依据：3.4.1 充值方式总览

**C-3 创建充值订单** `POST /api/wallet/deposit/create` — **Must**
- 选择充值方式 + 金额 + 奖金活动 → 校验重复转账 → 匹配通道 → 生成平台订单号（C+16位）→ 返回支付信息
- 需求依据：3.4.1 通用规则

**C-4 获取 USDT 充值信息** `POST /open/wallet/deposit/usdt` — **Must**
- 返回 USDT 链上充值信息：网络类型（TRC20/ERC20/BEP20）、收款地址、QR 码
- 路由组为 Open（游客也可访问）
- 需求依据：3.4.2 USDT 充值

**C-14 重复转账检测** `POST /api/wallet/deposit/duplicate-check` — **Should**
- 检测同金额同时间段的待支付订单数 → 1~2 笔弹警告 / 3 笔强拦截
- 可合并到 C-3 内部逻辑，但前端弹窗交互需先检测再决定是否创建
- 需求依据：3.5 重复转账拦截规则

**C-16 查询充值订单状态** `POST /api/wallet/deposit/status` — **Should**
- 前端从支付页返回后轮询订单状态
- 需求依据：3.4.1 "支付状态回传"

---

### 1.3 兑换相关（2 个）

**C-5 兑换预览** `POST /api/wallet/exchange/preview` — **Must**
- 输入法币金额 → 返回 BSB 到账数 + 赠送奖金 + 实际到账 + 流水要求
- 需求依据：3.7 兑换模块

**C-6 执行兑换** `POST /api/wallet/exchange/create` — **Must**
- 法币 → BSB 单向兑换：扣法币余额，入 BSB 余额（中心钱包 + 奖励钱包分别入账），记录稽核流水
- 需求依据：3.7 兑换计算

---

### 1.4 提现相关（4 个）

**C-7 获取提现方式列表** `POST /api/wallet/withdraw/methods` — **Must**
- 按当前币种过滤 + 根据 KYC 状态过滤（未 KYC 仅 USDT，已 KYC 全渠道）+ 充值路由规则
- 需求依据：3.8.1-3.8.2 KYC 关系 + 提现规则

**C-8 创建提现订单** `POST /api/wallet/withdraw/create` — **Must**
- 校验 KYC + 限额 + 账户信息 → 冻结余额 → 生成订单号（T+16位）→ 状态=待风控审核
- 需求依据：3.8.3-3.8.6 提现流程

**C-9 获取提现账户信息** `POST /api/wallet/withdraw/account` — **Must**
- 返回已绑定的提现账户信息（银行卡 / 电子钱包 / USDT 地址），非首次自动填充只读
- 需求依据：3.8.3 "非首次自动填充"

**C-10 保存提现账户信息** `POST /api/wallet/withdraw/account/save` — **Must**
- 首次保存提现账户（银行名 / 账号 / 持有人 / 网点等），保存后不可自行修改
- 持有人姓名必须与 KYC 认证姓名一致
- 需求依据：3.8.3 "首次提现字段" + 3.10 "用户不允许自行修改"

---

### 1.5 账变记录（2 个）

**C-11 获取账变记录列表** `POST /api/wallet/transaction/list` — **Must**
- 按 Tab 类型（充值 / 提现 / 兑换 / 奖励）分页查询，严格绑定当前币种
- 需求依据：3.9.1-3.9.2 记录分类 + 列表字段

**C-12 获取账变记录详情** `POST /api/wallet/transaction/detail` — **Must**
- 订单号 / 金额明细 / 状态 / 时间 / 方式 / 失败原因
- 需求依据：3.9.3 详情页

---

### 1.6 补充接口（4 个）

**C-15 获取奖金活动列表** `POST /api/wallet/deposit/bonus-activities` — **Should**
- 获取当前币种 + 金额匹配的奖金活动列表（活动门槛 / 奖金百分比 / 流水要求）
- 活动数据可能来自运营活动模块或财务支付配置
- 需求依据：3.6 奖金/优惠选择

**C-17 获取提现限额信息** `POST /api/wallet/withdraw/limits` — **Should**
- 每日限额 / 单笔最低 / 手续费率，数据来源于财务模块提现方式配置
- 可合并到 C-7 返回值中
- 需求依据：3.8.3 "每日提现限额" + 3.8.6 提现校验

**C-18 取消充值订单** `POST /api/wallet/deposit/cancel` — **Could**
- 取消超时未支付的充值订单
- 也可由后端定时任务自动处理
- 需求依据：3.4.4 "订单过期后 QR 码失效"

**C-19 获取可用币种列表** `POST /api/wallet/currencies` — **Could**
- C 端获取可用币种列表（已启用 + 排序 + 图标）
- 可合并到 C-1 返回值中
- 需求依据：3.3 币种切换规则

---

### C 端接口汇总

```
Must（核心必须）：13 个
  C-1  GetWalletOverview        钱包首页余额
  C-2  GetDepositMethods        充值方式列表
  C-3  CreateDepositOrder       创建充值订单
  C-4  GetUSDTDepositInfo       USDT 充值信息
  C-5  GetExchangePreview       兑换预览
  C-6  CreateExchange           执行兑换
  C-7  GetWithdrawMethods       提现方式列表
  C-8  CreateWithdrawOrder      创建提现订单
  C-9  GetWithdrawAccount       获取提现账户
  C-10 SaveWithdrawAccount      保存提现账户
  C-11 GetTransactionList       账变记录列表
  C-12 GetTransactionDetail     账变记录详情
  C-13 GetCurrentRate           当前汇率信息

Should（应该有）：4 个
  C-14 CheckDuplicateDeposit    重复转账检测
  C-15 GetBonusActivities       奖金活动列表
  C-16 GetDepositOrderStatus    充值订单状态查询
  C-17 GetWithdrawLimits        提现限额信息

Could（可选预留）：2 个
  C-18 CancelDepositOrder       取消充值订单
  C-19 GetAvailableCurrencies   可用币种列表
```

---
---

## 第二部分：B 端 HTTP 接口（6 个）

> 面向后台运营 / 管理人员，经 gate-back 网关路由到 ser-wallet
> 路由规范：`POST /api/wallet/xxx`（需后台登录鉴权 BackAuthMiddleware）
> 需求依据：需求文档第四章（4.4~4.6）
> 范围：仅币种配置模块，钱包模块当前无 B 端接口（余额查询和人工操作由财务 B 端承载）

---

**B-1 币种列表查询** `POST /api/currency/list` — **Must**
- 分页查询所有已配置币种，含 20 个配置字段（编码 / 名称 / 类型 / 图标 / 精度 / 汇率 / 浮动% / 阈值% / 状态 / 排序...）
- 支持筛选（币种代码 / 类型 / 状态）+ 实时汇率刷新倒计时
- 需求依据：4.4.1-4.4.2 页面布局 + 配置字段

**B-2 编辑币种** `POST /api/currency/edit` — **Must**
- 修改币种信息：汇率浮动% / 阈值% / 图标上传（SVG/PNG ≤ 5KB）/ 状态等
- 需求依据：4.4.3 编辑弹窗字段

**B-3 启用/禁用币种** `POST /api/currency/toggle-status` — **Must**
- 切换币种可用状态，基准币种不可禁用
- 需求依据：4.4.4 规则

**B-4 设置基准币种** `POST /api/currency/set-base` — **Must**
- 一次性设置基准币种（USDT），提交后按钮隐藏，不可修改，需二次确认
- 需求依据：4.3 + 4.4.4 基准币种配置规则

**B-5 汇率日志查询** `POST /api/currency/rate-log/list` — **Must**
- 筛选（日期范围 + 币种代码），字段（编号 / 时间 / 币种 / 实时汇率 / 平台汇率 / 入款汇率 / 出款汇率）
- 需求依据：4.6 汇率日志

**B-6 获取币种详情** `POST /api/currency/detail` — **Should**
- 获取单个币种详情（编辑弹窗数据填充）
- 可复用列表数据，但独立接口更规范
- 需求依据：4.4.3 编辑弹窗

---

### B 端接口汇总

```
Must（核心必须）：5 个
  B-1  GetCurrencyList          币种列表查询
  B-2  EditCurrency             编辑币种
  B-3  ToggleCurrencyStatus     启用/禁用币种
  B-4  SetBaseCurrency          设置基准币种
  B-5  GetExchangeRateLogs      汇率日志查询

Should（应该有）：1 个
  B-6  GetCurrencyDetail        币种详情
```

---
---

## 第三部分：RPC 对外暴露接口（12 个）

> ser-wallet 暴露给工程内其他服务调用的 Kitex RPC 接口
> 调用方：ser-finance（财务）、游戏/投注模块、直播/礼物模块、活动模块等
> IDL 位置（待创建）：common/idl/ser-wallet/wallet_rpc.thrift
> 详细分析见：/Users/mac/gitlab/z-readme/rpc暴露/tmp-2.md

---

### 3.1 余额操作类（6 个 — 写操作）

**R-1 CreditWallet（入账/加款）** — **Must**
- 向指定用户的指定子钱包增加余额，记录账变流水
- 调用场景：充值入账 / 补单 / 人工加款 / 投注返奖 / 主播收入 / 活动奖励
- 幂等键：orderNo + walletType
- 调用方：ser-finance、游戏模块、直播模块、活动模块

**R-2 DebitWallet（扣款/减款）** — **Must**
- 从指定用户的指定子钱包扣减余额，记录账变流水
- 调用场景：人工减款、指定钱包扣款
- 余额不足时明确拒绝并返回错误码 BALANCE_NOT_ENOUGH
- 调用方：ser-finance、游戏模块

**R-3 FreezeBalance（冻结余额）** — **Must**
- 可用余额 → 冻结余额（提现发起时）
- 与 R-4 / R-5 配套使用：正常出款 = Freeze → DeductFrozen；驳回 = Freeze → Unfreeze
- 调用方：内部（CreateWithdrawOrder）/ ser-finance

**R-4 UnfreezeBalance（解冻余额）** — **Must**
- 冻结余额 → 可用余额（提现驳回时）
- 风控审核驳回 / 财务审核驳回 / 出款驳回 均触发解冻
- 调用方：ser-finance

**R-5 DeductFrozenAmount（扣除冻结金额）** — **Must**
- 冻结余额减少（钱真正离开系统，出款成功场景）
- 与 R-4 的本质区别：Unfreeze 是"钱回到用户手里"，DeductFrozen 是"钱出去了"
- 调用方：ser-finance

**R-7 DebitByPriority（按优先级扣款）** — **Should**
- 按消费优先级自动扣款：先扣中心钱包 → 再扣奖励钱包
- 返回 deductDetails（扣款明细），调用方据此做比例返奖
- 调用场景：游戏投注 / 直播打赏 / 道具购买
- 调用方：游戏模块、直播模块、商城模块

---

### 3.2 余额查询类（2 个 — 读操作）

**R-6 QueryWalletBalance（查询余额）** — **Must**
- 获取用户各子钱包的当前余额（含可用 + 冻结）
- 返回中心 / 奖励 / 主播 / 代理四个子钱包的余额明细
- 调用方：ser-finance（加减款/补单表单）、游戏模块（投注前校验）、其他

**R-11 BatchQueryBalance（批量查询余额）** — **Could**
- 批量查询多用户余额，限制上限（如 100 个 userId）
- 调用方：运营报表、B 端用户管理
- 视后续运营需求决定是否首期实现

---

### 3.3 币种信息类（3 个 — 读操作）

**R-8 GetCurrencyConfig（获取币种配置）** — **Must**
- 返回可用币种列表（代码 / 名称 / 符号 / 图标 / 精度 / 千分位 / 状态 / 排序 / 是否基准）
- 高频读接口，建议调用方缓存
- 调用方：所有需要知道平台支持哪些币种的模块

**R-9 GetExchangeRate（获取汇率）** — **Must**
- 返回指定币种的汇率信息（实时 / 平台 / 入款 / 出款四种类型）
- 带更新时间戳，调用方可判断汇率新鲜度
- 调用方：所有需要汇率做金额换算的模块

**R-10 ConvertAmount（金额换算）** — **Should**
- 输入金额 + 源币种 + 目标币种 → 返回换算结果 + 使用的汇率值
- 按目标币种精度截断/四舍五入
- 调用方：ser-finance（统计报表跨币种汇总）

---

### 3.4 流水查询类（1 个 — 读操作）

**R-12 GetWalletFlowList（查询账变流水）** — **Could**
- 分页查询用户账变流水（含操作类型 / 变动金额 / 变动前后余额 / 订单号 / 时间）
- 调用方：B 端运营查看用户资金明细、审计合规
- 可能被 B 端直接查数据库替代

---

### RPC 对外暴露汇总

```
Must（核心必须）：6 个
  R-1  CreditWallet             入账/加款
  R-2  DebitWallet              扣款/减款
  R-3  FreezeBalance            冻结余额
  R-4  UnfreezeBalance          解冻余额
  R-5  DeductFrozenAmount       扣除冻结
  R-6  QueryWalletBalance       查询余额

Must（基础数据）：2 个
  R-8  GetCurrencyConfig        获取币种配置
  R-9  GetExchangeRate          获取汇率

Should（应该有）：2 个
  R-7  DebitByPriority          按优先级扣款
  R-10 ConvertAmount            金额换算

Could（可选预留）：2 个
  R-11 BatchQueryBalance        批量查询余额
  R-12 GetWalletFlowList        查询账变流水

优先级矩阵：P0 = 8 个 / P1 = 2 个 / P2 = 2 个
```

---
---

## 第四部分：RPC 主动调用接口（11 个）

> ser-wallet 作为客户端，主动调用工程内其他服务的 RPC 接口
> 详细分析见：/Users/mac/gitlab/z-readme/rpc调用/tmp-2.md

---

### 4.1 确定依赖 — 需求明确（6 个）

**O-1 ser-finance → GetPaymentMethods** — **Should**
- 获取充值 / 提现方式配置（名称 / 图标 / 档位 / 限额 / 关联通道）
- C 端 GetDepositMethods / GetWithdrawMethods 的数据来源
- 状态：ser-finance **尚未创建**，需 mock

**O-2 ser-finance → MatchAndCreateChannelOrder** — **Should**
- 充值时请求通道匹配（轮询算法）+ 创建通道侧订单
- 充值流程：钱包创建平台订单 → 财务匹配通道 → 返回支付信息
- 状态：ser-finance **尚未创建**，需 mock

**O-3 ser-finance → InitiateChannelPayout** — **Should**
- 提现出款时发起通道代付请求
- 调用归属待确认（大概率由财务模块自行发起，我方不需调用）
- 状态：ser-finance **尚未创建**，**且调用归属待确认**

**O-4 ser-finance → GetChannelOrderStatus** — **Could**
- 查询通道订单状态（补偿机制：定时查通道状态防止回调丢失）
- 状态：ser-finance **尚未创建**，需 mock

**O-5 ser-kyc → KycQuery** — **Must**
- 提现前查询用户 KYC 认证状态 + 获取认证姓名
- 未认证 → 仅 USDT 提现；已认证 → 全渠道
- 状态：**已存在可直接调用**（KycClient()）

**O-6 ser-user → GetUserInfoExpend** — **Must**
- 获取用户基本信息（countryCode 用于确定默认币种、status 用于账号状态校验）
- 提现时 KYC 姓名校验依赖此接口
- 状态：**已存在可直接调用**（UserClient()）

---

### 4.2 工程规范依赖（3 个）

**O-7 ser-blog → AddActLog** — **Must（工程规范）**
- B 端所有写操作记录操作日志（oneway 异步，不阻塞业务）
- 状态：**已存在可直接调用**（BLogClient()）

**O-8 ser-buser → QueryUserByIds** — **Should（工程规范）**
- B 端列表接口查询操作者名称（如汇率日志中的操作人）
- 状态：**已存在可直接调用**（BUserClient()）

**O-9 ser-socket → WebSocket 推送** — **Should（工程规范）**
- 余额变更后实时推送通知到用户 App
- 状态：基础设施已存在，但**推送接口待完善**

---

### 4.3 待确认依赖（2 个）

**O-10 活动模块 → GetMatchingActivities** — **Could**
- 获取匹配的奖金活动列表（充值页面奖金选择）
- 活动模块可能不存在，数据可能由财务支付配置管理
- 状态：活动模块 **IDL 不存在**，待评审确认

**O-11 ser-i18n → LangTransList** — **Could**
- 多语言文案翻译（错误提示、操作类型名称等）
- 大概率不需要直接调用（多语言通常由前端处理）
- 状态：**已存在可调用**，但大概率不需要

---

### RPC 主动调用汇总

```
确定依赖（需求明确）：6 个
  O-1  ser-finance  GetPaymentMethods        ❌ 需mock
  O-2  ser-finance  MatchAndCreateChannelOrder ❌ 需mock
  O-3  ser-finance  InitiateChannelPayout     ❌ 需mock + 归属待确认
  O-4  ser-finance  GetChannelOrderStatus     ❌ 需mock
  O-5  ser-kyc      KycQuery                  ✅ 可直接调
  O-6  ser-user     GetUserInfoExpend         ✅ 可直接调

工程规范依赖：3 个
  O-7  ser-blog     AddActLog (oneway)        ✅ 可直接调
  O-8  ser-buser    QueryUserByIds            ✅ 可直接调
  O-9  ser-socket   WebSocket推送             ⚠️ 待接口完善

待确认依赖：2 个
  O-10 活动模块     GetMatchingActivities     ❌ 不存在
  O-11 ser-i18n     LangTransList             ⚠️ 待观察
```

---
---

## 第五部分：定时任务 / 内部处理（3 个）

> 不暴露为外部接口，属于 ser-wallet 服务内部后台逻辑

---

**T-1 汇率定时更新** — **Must**
- 定时拉取 3 个三方汇率 API 取平均值 → 偏差检测 → 触发平台汇率更新 → 生成汇率日志
- 需求依据：4.5.2-4.5.3 汇率获取 + 阈值更新
- 外部依赖：3 个三方汇率 HTTP API（并行调用，互为容错）

**T-2 充值订单超时处理** — **Should**
- 充值订单超时自动失效（QR 码过期 / 支付超时 → 订单状态=支付失败）
- 需求依据：3.4.4 "订单过期后 QR 码失效"

**T-3 账务对账校验** — **Should**
- 账务流水与余额一致性校验（定时对账）
- 需求依据：3.10 "必须建立强一致性校验机制"

---
---

## 第六部分：财务管理模块（他方负责 — 联动参考）

> **此模块非我方实现**，但因跨模块 RPC 深度依赖，必须清楚其接口全貌
> 财务模块 IDL / 服务 / Client 均**尚未创建**，是当前最大联调阻塞项

---

### 6.1 财务 B 端 HTTP 接口（约 41 个）

```
通道配置（~8 个）
  - 代收通道列表 / 编辑 / 启禁用 / 轮询配置
  - 代付通道列表 / 编辑 / 启禁用 / 轮询配置

支付配置（~8 个）
  - 充值方式列表 / 新增 / 编辑 / 启禁用
  - 提现方式列表 / 新增 / 编辑 / 启禁用

充值记录（~3 个）
  - 列表 / 详情 / 导出

提现记录（~9 个）
  - 列表 / 详情 / 风控审核 / 财务审核 / 出款确认 / 锁定 / 解锁 / 强制解锁 / 导出

人工修正（~9 个）
  - 补单（列表 / 创建 / 审核）
  - 加款（列表 / 创建 / 审核）
  - 减款（列表 / 创建 / 审核）

数据统计（~4 个）
  - 代收通道统计 / 代付通道统计 / 充值统计 / 提现统计
```

### 6.2 财务回调接口（2 个）

```
  - 代收回调：三方代收（充值）结果通知
  - 代付回调：三方代付（提现）结果通知
```

### 6.3 财务 RPC 暴露（供钱包调用，~4 个）

```
  - GetPaymentMethods          返回充值/提现方式配置
  - MatchAndCreateChannelOrder 通道轮询匹配 + 创建通道订单
  - InitiateChannelPayout      发起通道代付
  - GetChannelOrderStatus      查询通道订单状态
```

### 6.4 财务 RPC 调用钱包（6 个）

```
  - CreditWallet        充值入账 / 补单 / 加款
  - DebitWallet          人工减款
  - FreezeBalance        提现冻结（如提现归财务创建）
  - UnfreezeBalance      提现驳回解冻
  - DeductFrozenAmount   出款成功扣冻结
  - QueryWalletBalance   加减款/补单表单查余额
```

### 财务模块合计：约 53 个（41 B端 + 2 回调 + 4 RPC供 + 6 RPC调）

---
---

## 第七部分：跨模块调用关系全景

### 7.1 调用方向图

```
                           ┌──────────────┐
                           │   ser-kyc    │
                           │  (KYC状态)   │
                           └──────┬───────┘
                                  │ O-5: KycQuery
                                  ▼
┌──────────────┐   R-8~10    ┌──────────────────────┐   R-1~6     ┌──────────────┐
│   其他模块    │◄────────────│      ser-wallet       │◄────────────│  财务管理模块  │
│  (游戏/直播   │  币种/汇率   │  ┌────────────────┐  │  入账/冻结   │  (通道/审核    │
│   礼物/投注)  │             │  │  钱包核心       │  │  解冻/扣冻   │   支付配置     │
│              │  R-1,2,7    │  │  (余额/账变)   │  │  查余额      │   人工修正)    │
│              │────────────►│  ├────────────────┤  │             │              │
│              │  扣款/入账   │  │  币种配置       │  │  O-1~3      │              │
│              │             │  │  (汇率/币种)   │  │────────────►│              │
└──────────────┘             │  └────────────────┘  │  支付方式    └──────┬───────┘
                             └──────────┬───────────┘  通道匹配          │
                                  │ O-6: GetUserInfo   代付请求          │
                                  ▼                                  回调 │
                           ┌──────────────┐                   ┌─────────▼────┐
                           │   ser-user   │                   │  三方支付通道  │
                           │  (用户信息)   │                   │  (代收/代付)  │
                           └──────────────┘                   └──────────────┘
```

### 7.2 双向依赖对照（ser-wallet ↔ ser-finance）

```
我方暴露给财务（6 个 RPC）          我方调用财务（4 个 RPC）
──────────────────────             ──────────────────────
R-1  CreditWallet      ←──┐  ┌──→ O-1  GetPaymentMethods
R-2  DebitWallet       ←──│  │──→ O-2  MatchAndCreateChannelOrder
R-3  FreezeBalance     ←──│  │──→ O-3  InitiateChannelPayout
R-4  UnfreezeBalance   ←──│  │──→ O-4  GetChannelOrderStatus
R-5  DeductFrozenAmount←──│  │
R-6  QueryWalletBalance←──┘  │
                              │
    合计 10 个 RPC 调用横跨两模块
```

---
---

## 第八部分：各维度优先级分布

### 8.1 我方主责总表

```
维度              Must    Should   Could    合计
──────────────────────────────────────────────
C端 HTTP           13       4        2       19
B端 HTTP            5       1        0        6
RPC 对外暴露        8       2        2       12
RPC 主动调用        2       3        1        6（确定依赖）
定时任务            1       2        0        3
──────────────────────────────────────────────
合计               27      14        5       46
```

### 8.2 含工程规范和待确认的完整统计

```
分类              数量    状态
──────────────────────────────────
确定性主责接口     46     已定型
工程规范依赖调用    3     O-7/O-8/O-9（ser-blog/buser/socket）
待确认依赖调用      2     O-10/O-11（活动模块/i18n）
外部 HTTP 依赖      1     三方汇率 API × 3
──────────────────────────────────
广义总计           52
```

### 8.3 三模块全局总览

```
模块            C端    B端    RPC供   RPC调   回调   定时   合计    职责
──────────────────────────────────────────────────────────────────────
钱包             19     —      12      6      —      3     40     主责
币种配置          —     6    (含在钱包RPC中)   —      —    (含在定时中)  6   主责
财务管理          —     41      4      6      2      —     ~53    他方
──────────────────────────────────────────────────────────────────────
合计             19     47     16     12      2      3     ~99
```

---
---

## 第九部分：接口全列表速查卡

### 9.1 C 端接口速查

```
编号   接口名                     路由                                优先级  核心动作
─────────────────────────────────────────────────────────────────────────────────
C-1    GetWalletOverview          POST /api/wallet/overview           Must    查
C-2    GetDepositMethods          POST /api/wallet/deposit/methods    Must    查
C-3    CreateDepositOrder         POST /api/wallet/deposit/create     Must    写
C-4    GetUSDTDepositInfo         POST /open/wallet/deposit/usdt      Must    查
C-5    GetExchangePreview         POST /api/wallet/exchange/preview   Must    查
C-6    CreateExchange             POST /api/wallet/exchange/create    Must    写
C-7    GetWithdrawMethods         POST /api/wallet/withdraw/methods   Must    查
C-8    CreateWithdrawOrder        POST /api/wallet/withdraw/create    Must    写
C-9    GetWithdrawAccount         POST /api/wallet/withdraw/account   Must    查
C-10   SaveWithdrawAccount        POST /api/wallet/withdraw/account/save Must 写
C-11   GetTransactionList         POST /api/wallet/transaction/list   Must    查
C-12   GetTransactionDetail       POST /api/wallet/transaction/detail Must    查
C-13   GetCurrentRate             POST /api/wallet/rate/current       Must    查
C-14   CheckDuplicateDeposit      POST /api/wallet/deposit/dup-check  Should  查
C-15   GetBonusActivities         POST /api/wallet/deposit/bonus      Should  查
C-16   GetDepositOrderStatus      POST /api/wallet/deposit/status     Should  查
C-17   GetWithdrawLimits          POST /api/wallet/withdraw/limits    Should  查
C-18   CancelDepositOrder         POST /api/wallet/deposit/cancel     Could   写
C-19   GetAvailableCurrencies     POST /api/wallet/currencies         Could   查
```

### 9.2 B 端接口速查

```
编号   接口名                     路由                                优先级  核心动作
─────────────────────────────────────────────────────────────────────────────────
B-1    GetCurrencyList            POST /api/currency/list             Must    查
B-2    EditCurrency               POST /api/currency/edit             Must    写
B-3    ToggleCurrencyStatus       POST /api/currency/toggle-status    Must    写
B-4    SetBaseCurrency            POST /api/currency/set-base         Must    写
B-5    GetExchangeRateLogs        POST /api/currency/rate-log/list    Must    查
B-6    GetCurrencyDetail          POST /api/currency/detail           Should  查
```

### 9.3 RPC 对外暴露速查

```
编号   接口名                     类型     优先级  被谁调          最高频场景
─────────────────────────────────────────────────────────────────────────────
R-1    CreditWallet               写入     P0     财务/游戏/直播  充值入账
R-2    DebitWallet                扣减     P0     财务/游戏       人工减款
R-3    FreezeBalance              冻结     P0     内部/财务       提现发起
R-4    UnfreezeBalance            解冻     P0     财务            提现驳回
R-5    DeductFrozenAmount         扣冻结   P0     财务            出款成功
R-6    QueryWalletBalance         查询     P0     财务            加减款表单
R-7    DebitByPriority            扣减     P1     游戏/直播       投注扣款
R-8    GetCurrencyConfig          查询     P0     所有模块        获取币种列表
R-9    GetExchangeRate            查询     P0     所有模块        获取汇率
R-10   ConvertAmount              计算     P1     财务统计        金额换算
R-11   BatchQueryBalance          查询     P2     运营报表        批量查余额
R-12   GetWalletFlowList          查询     P2     B端管理         查资金流水
```

### 9.4 RPC 主动调用速查

```
编号   目标服务        方法                           优先级   状态
──────────────────────────────────────────────────────────────────────
O-1    ser-finance    GetPaymentMethods              Should   ❌需mock
O-2    ser-finance    MatchAndCreateChannelOrder      Should   ❌需mock
O-3    ser-finance    InitiateChannelPayout           Should   ❌需mock+待确认
O-4    ser-finance    GetChannelOrderStatus           Could    ❌需mock
O-5    ser-kyc        KycQuery                        Must     ✅可直接调
O-6    ser-user       GetUserInfoExpend               Must     ✅可直接调
O-7    ser-blog       AddActLog (oneway)              Must*    ✅可直接调
O-8    ser-buser      QueryUserByIds                  Should*  ✅可直接调
O-9    ser-socket     WebSocket推送                   Should*  ⚠️待接口完善
O-10   活动模块        GetMatchingActivities           Could    ❌不存在
O-11   ser-i18n       LangTransList                   Could    ⚠️待观察
       (* 工程规范依赖)
```

### 9.5 定时任务速查

```
编号   任务名                     优先级   触发方式       核心逻辑
──────────────────────────────────────────────────────────────────
T-1    ExchangeRateUpdate         Must     定时(可配频率) 3API取均值→偏差检测→更新汇率
T-2    DepositOrderTimeout        Should   定时(分钟级)   超时订单自动失效
T-3    BalanceReconciliation      Should   定时(小时/日级) 流水与余额一致性校验
```

---
---

## 第十部分：多版本评估收敛说明

### 10.1 历次评估数据对比

```
评估版本                  我方主责接口数     全局总数     主要变化
──────────────────────────────────────────────────────────────
c-5.md 初版粗估           30-35            55-65       最粗粒度，无四维分类
tmp-1.md 首版粗估         26-30            55-62       补充说明，维持量级
tmp-2.md v2               ~29              ~57         补充币种配置RPC、财务人工修正
tmp-2.md v3               46               ~99         ★ 四维度+三层级，新增RPC调用+定时任务
d-3.md 细粒度版           69               132         每个RPC场景独立计数（最大值）
tmp-3.md 迭代版           49-56            —           四维度+三层级，RPC暴露更细化
c-5.md 迭代版             45               81          RPC暴露合并通用接口

本文整合版                 46               ~99         ★ 收敛为确定性清单
```

### 10.2 接口数量差异的根本原因

各版本接口数差异的核心在于 **RPC 暴露接口的粒度选择**：

```
粗粒度视角（本文 / tmp-2.md v3）：
  CreditWallet   = 通用入账接口（通过 opType 参数区分：充值/补单/加款/返奖/奖励/主播收入）
  DebitWallet    = 通用扣款接口（通过 opType 参数区分：减款/投注/购买/打赏）
  → 合并为 12 个 RPC 接口

细粒度视角（d-3.md）：
  CreditWallet   → 拆为 CreditWallet + SupplementCredit + ManualCredit + RewardCredit + BetCredit + AnchorCredit
  DebitWallet    → 拆为 DebitWallet + ManualDebit + BetDebit + GiftDebit
  → 展开为 18 个 RPC 接口
```

**本文选择粗粒度**，理由：
1. 参照项目现有模式（ser-live/ser-item 的 RPC 均为通用方法 + 参数区分）
2. 粗粒度减少 IDL 方法数，降低维护成本
3. 业务差异通过 opType 枚举 + 内部逻辑分支处理，不影响正确性
4. 如后续发现某场景逻辑差异过大（如 DebitByPriority 的优先级扣款），再独立为单独方法

---
---

## 第十一部分：预评审待确认事项

> 以下事项在当前需求文档中存在模糊地带或需要跨部门确认，会影响接口数量和归属

### 11.1 影响接口数量的事项

| 序号 | 事项 | 当前假设 | 如果假设不成立 |
|------|------|---------|--------------|
| 1 | 充值回调归属 | 归财务模块（持有通道密钥） | 如归钱包，C 端 +1 |
| 2 | 提现审核操作归属 | 归财务 B 端（需求 5.5 明确） | 如归独立风控模块，接口归属变化 |
| 3 | 充值订单表归属 | 钱包建平台订单，财务建通道订单 | 如全归财务，钱包 C-3 逻辑简化 |
| 4 | 奖金活动数据来源 | 财务支付配置关联活动 | 如独立活动模块，RPC 调用 +1 |
| 5 | 提现出款发起方 | 财务模块自行发起代付 | 如归钱包，O-3 变为确定依赖 |
| 6 | 钱包 B 端独立接口 | 无独立 B 端（通过财务 RPC 查余额） | 如需要，B 端 +2~3 |

### 11.2 影响接口设计的事项

| 序号 | 事项 | 说明 |
|------|------|------|
| 7 | amount 字段类型 | string（Decimal 字符串）vs i64（最小单位整数），推荐 string |
| 8 | walletType 枚举值 | 1=中心 / 2=奖励 / 3=主播 / 4=代理，需与财务模块完全一致 |
| 9 | 幂等键组成 | orderNo 单独 vs orderNo + walletType 组合，推荐后者 |
| 10 | opType 枚举值 | 充值=1 / 补单=2 / 加款=3 / 返奖=4 / 活动=5 / 主播收入=6 / ...需双方对齐 |
| 11 | USDT 收款地址模式 | 平台统一地址 vs 每用户独立地址，影响与链上监控集成 |
| 12 | 提现账户能否修改 | 3.10 明确"用户不允许自行修改"，是否有 B 端修改通道？ |

### 11.3 阻塞项

| 序号 | 阻塞项 | 影响范围 | 解除条件 |
|------|--------|---------|---------|
| 1 | ser-finance IDL 不存在 | O-1~O-4 全部需 mock | 同事创建 common/idl/ser-finance/ |
| 2 | FinanceClient 不存在 | RPC 调用无法初始化 | 在 common/rpc/rpc_client.go 新增 |
| 3 | 活动模块不存在 | O-10 无法联调 | 确认活动数据归属后决定 |
| 4 | ser-socket 推送接口缺失 | O-9 无法实现 | ser-socket 团队补充推送方法 |

---
---

## 第十二部分：开发阶段路径建议

### 12.1 按阶段实现顺序

```
Phase 2（核心能力层 — 内部功能，无外部依赖）
═══════════════════════════════════════════════
  B 端：B-1 ~ B-5（币种配置 CRUD + 汇率日志）
  RPC 暴露：R-1 ~ R-6, R-8, R-9（8 个 P0 接口）
  定时任务：T-1（汇率更新）

  说明：这一阶段全部是"被调用方"能力建设，不依赖任何外部模块。
  完成后，ser-wallet 具备了被财务/游戏/直播模块调用的能力。

Phase 3（C 端功能层 — 分批实现）
═══════════════════════════════════════════════
  先做（不依赖外部）：
    C-1   钱包首页
    C-5   兑换预览
    C-6   执行兑换         ← 纯内部操作，可最先端到端跑通
    C-9   获取提现账户
    C-10  保存提现账户
    C-11  账变记录列表
    C-12  账变记录详情
    C-13  当前汇率信息
    C-19  可用币种列表

  后做（依赖财务 mock / KYC / User）：
    C-2   充值方式列表     ← mock ser-finance
    C-3   创建充值订单     ← mock ser-finance
    C-4   USDT 充值信息    ← mock ser-finance
    C-7   提现方式列表     ← mock ser-finance + 真调 ser-kyc
    C-8   创建提现订单     ← 真调 ser-kyc + ser-user
    C-14 ~ C-18  补充接口

Phase 4（联调验证层）
═══════════════════════════════════════════════
  与财务模块联调：O-1 ~ O-4 从 mock 切换为真实调用
  补充暴露：R-7, R-10, R-11, R-12
  补充定时：T-2, T-3
```

### 12.2 独立开发期间的 Mock 策略

```
可直接调用（不需 mock）：
  ser-kyc   → KycClient()    ✅ 已存在
  ser-user  → UserClient()   ✅ 已存在
  ser-blog  → BLogClient()   ✅ 已存在
  ser-buser → BUserClient()  ✅ 已存在

需要 mock（对方未就绪）：
  ser-finance → FinanceClient()  ❌ 建议做接口抽象层（interface + mock 实现）
  活动模块    → —               ❌ 返回固定活动列表

建议在 ser-wallet 内部对 ser-finance 做一层调用抽象：
  internal/external/finance.go       → FinanceService interface
  internal/external/finance_rpc.go   → 真实 RPC 实现
  internal/external/finance_mock.go  → Mock 实现
  main.go 中通过配置切换
```

---
---

## 总结

1. **我方主责 46 个接口**，其中 27 个 Must 是结构性绕不过的核心骨架，评审后数量不会有大的偏差。

2. **C 端 19 个接口是用户直接感知的功能出口**，其中兑换链路（C-5/C-6）完全不依赖外部模块，是最先可以端到端跑通的业务链路。

3. **RPC 暴露 12 个接口是 ser-wallet 的核心能力输出**，其中 8 个 P0 接口是财务模块联调的硬依赖，必须最先实现。写操作 RPC（入账/扣款/冻结/解冻/扣冻结/优先级扣款）在项目中是独一无二的，需特别注意幂等性和事务原子性。

4. **ser-finance 是最大联调阻塞项**，4 个核心 RPC 调用指向一个尚未创建的服务。建议提前做好接口抽象和 Mock 层设计，确保独立开发期间不被阻塞。

5. **接口粒度选择粗粒度方案**（通用方法 + opType 参数区分），与项目现有模式一致。如后续某场景逻辑差异过大，再独立为单独方法。

6. **预评审 12 个待确认事项**中，6 个影响接口数量（可能 ±3 个），6 个影响接口设计细节（amount 类型、幂等键、枚举值等），4 个为开发阻塞项。
