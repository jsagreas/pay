# ser-wallet 接口调用链路关系

> 分析时间：2026-02-26
> 分析视角：我负责币种配置 + 多币种钱包模块，需要清楚每个接口的内部调用链路和跨模块调用链路
> 信息源：
> - 需求理解：/Users/mac/gitlab/z-readme/需求理解/tmp-1.md
> - 接口评估：/Users/mac/gitlab/z-readme/接口评估/tmp-1.md
> - 依赖关系：/Users/mac/gitlab/z-readme/依赖关系/tmp-1.md
> - 原始需求：/Users/mac/gitlab/z-readme/result/tmp-1.md（~130张需求图片提取）

---

## 目录

1. [币种配置模块：各接口内部调用链路](#一币种配置模块各接口内部调用链路)
2. [钱包模块：各接口内部调用链路](#二钱包模块各接口内部调用链路)
3. [模块内部调用关系（币种配置 ↔ 钱包）](#三模块内部调用关系币种配置--钱包)
4. [跨模块调用链路（wallet ↔ finance）](#四跨模块调用链路wallet--finance)
5. [跨模块调用链路（wallet ↔ 其他服务）](#五跨模块调用链路wallet--其他服务)
6. [完整业务场景端到端调用链路](#六完整业务场景端到端调用链路)
7. [每个接口的"调用谁/被谁调用"速查表](#七每个接口的调用谁被谁调用速查表)

---

## 一、币种配置模块：各接口内部调用链路

> 说明：币种配置模块的接口相对独立，大部分是直接操作数据库的CRUD，内部调用链路比较短。
> 以下"→"表示调用方向，"[DB]"表示数据库操作，"[RPC]"表示跨服务RPC调用。

### 1.1 `CreateCurrency` — 新增币种

```
前端(B端) → gate-back → CreateCurrency
    │
    ├─ [校验] 币种代码唯一性 → [DB] 查询币种表是否已存在
    ├─ [校验] 币种类型合法性（法币/加密/平台币）
    ├─ [处理] 如果有图标 → [RPC] ser-s3.UploadFile（上传图标文件）→ 获取图标URL
    ├─ [DB] 插入币种表（20个字段：代码/名称/类型/国家/图标URL/符号/千分位/小数点/浮动%/阈值%/精度/排序/状态等）
    ├─ [RPC] ser-blog.AddActLog（审计日志，oneway异步）
    └─ 返回结果
```

**调用链路说明**：CreateCurrency 是起点接口，不依赖其他业务接口。内部仅涉及DB写入 + 图标上传 + 审计日志。
**需求依据**：币种配置需求文档3.1编辑弹窗——"币种数据初始由开发导入，新增需联系开发"，但接口层面仍需提供创建能力。

---

### 1.2 `UpdateCurrency` — 编辑币种

```
前端(B端) → gate-back → UpdateCurrency
    │
    ├─ [校验] 币种是否存在 → [DB] 查询币种表
    ├─ [校验] 不可编辑项：币种代码、币种类型（前端灰显，后端再校验）
    ├─ [处理] 如果更新了图标 → [RPC] ser-s3.UploadFile → 获取新图标URL
    ├─ [DB] 更新币种表（可编辑项：浮动%/阈值%/限额/图标/排序/精度/符号等）
    ├─ [RPC] ser-blog.AddActLog（审计日志，oneway异步）
    └─ 返回结果
```

**调用链路说明**：与 CreateCurrency 类似，独立操作，不触发其他接口联动。
**需求依据**：需求文档3.1编辑弹窗——"编辑弹窗支持上传币种图标SVG/PNG不超过5KB"。

---

### 1.3 `ToggleCurrencyStatus` — 启用/禁用币种

```
前端(B端) → gate-back → ToggleCurrencyStatus
    │
    ├─ [校验] 币种是否存在 → [DB] 查询币种表
    ├─ [校验] 如果是禁用操作：基准币种不允许禁用
    ├─ [DB] 更新币种状态字段（启用↔禁用）
    ├─ [RPC] ser-blog.AddActLog（审计日志，oneway异步）
    └─ 返回结果
```

**调用链路说明**：独立操作。禁用后 C 端的 `ListActiveCurrencies` 不再返回该币种，但这是查询侧过滤，不是 ToggleCurrencyStatus 主动通知。
**需求依据**：需求文档3.1——"基准币种无法禁用"，"币种禁用后C端不可见，已有余额需要特殊提示"。

---

### 1.4 `SetBaseCurrency` — 设置基准币种

```
前端(B端) → gate-back → SetBaseCurrency
    │
    ├─ [校验] 是否已经设置过基准币种 → [DB] 查询配置表/币种表的基准标记
    │         如果已设置 → 直接拒绝返回错误（一次性操作，不可修改）
    ├─ [校验] 指定的币种是否存在且状态正常
    ├─ [DB] 更新基准币种标记
    ├─ [RPC] ser-blog.AddActLog（审计日志，oneway异步）
    └─ 返回结果
```

**调用链路说明**：独立操作，一次性执行。设置后影响全局：所有汇率换算以 USDT 为基准，BSB 锚定 USDT（10 BSB = 1 USDT）。
**需求依据**：需求文档3.2——"只能设置一次，不可修改"，"后台有二次确认弹窗提醒"。

---

### 1.5 `QueryCurrencyPage` — 币种列表查询（B端）

```
前端(B端) → gate-back → QueryCurrencyPage
    │
    ├─ [DB] 分页查询币种表（20个字段全部返回）
    ├─ [处理] 操作人ID → [RPC] ser-buser.QueryUserByIds → 解析为操作人名称
    └─ 返回分页结果
```

**调用链路说明**：纯查询，仅读DB + 解析操作人。不依赖任何业务接口。

---

### 1.6 `QueryRateLogPage` — 汇率日志查询（B端）

```
前端(B端) → gate-back → QueryRateLogPage
    │
    ├─ [DB] 分页查询汇率日志表（日志编号/时间/币种代码/实时汇率/平台汇率/入款汇率/出款汇率）
    └─ 返回分页结果
```

**调用链路说明**：最简单的接口，纯DB读取，无任何外部调用。日志数据由汇率定时任务写入。
**需求依据**：需求文档4.4汇率日志——"平台汇率每次更新生成一条汇率日志记录"。

---

### 1.7 `GetCurrencyDetail` — 单币种详情（B端）

```
前端(B端) → gate-back → GetCurrencyDetail
    │
    ├─ [DB] 按币种ID/代码查询币种表单条记录
    └─ 返回完整字段（编辑弹窗回显用）
```

**调用链路说明**：纯查询，可以复用 QueryCurrencyPage 的单条逻辑。

---

### 1.8 `ListActiveCurrencies` — 可用币种列表（C端）

```
前端(C端) → gate-font → ListActiveCurrencies
    │
    ├─ [DB] 查询币种表，过滤条件：状态=启用
    ├─ [处理] 仅返回C端需要的字段：代码/名称/图标URL/货币符号/千分位符号/小数点符号/小数位数
    └─ 返回列表（按排序字段排序）
```

**调用链路说明**：纯查询，无外部调用。返回的数据用于C端钱包页的币种切换下拉框。
**需求依据**：钱包原型2-2币种切换列表——用户在钱包首页切换不同币种时，下拉框的数据源。

---

### 1.9 `GetExchangeRate` — 获取当前汇率（C端）

```
前端(C端) → gate-font → GetExchangeRate
    │
    ├─ [入参] 源币种、目标币种、方向（入款/出款）
    ├─ [DB] 查询币种表获取平台汇率、汇率浮动%
    ├─ [计算]
    │    如果方向=入款：返回汇率 = 平台汇率 × (1 + 浮动%)
    │    如果方向=出款：返回汇率 = 平台汇率 × (1 - 浮动%)
    └─ 返回：使用汇率 + 展示口径
```

**调用链路说明**：纯查询+计算，无外部调用。数据来源于币种表中的平台汇率字段（由定时任务维护）。
**需求依据**：需求文档4.2——"入款汇率 = 平台汇率 × (1 + 汇率浮动%)"，"出款汇率 = 平台汇率 × (1 - 汇率浮动%)"。

**被以下接口调用**：
- 钱包.GetDepositConfig → 充值页面展示汇率
- 钱包.PreviewExchange → 兑换试算时获取汇率
- 钱包.GetWithdrawConfig → 提现页面展示换算金额

---

### 1.10 `GetCurrencyConfig` — 获取币种配置（RPC对外）

```
调用方(RPC) → GetCurrencyConfig
    │
    ├─ [入参] 币种代码（单个或批量）
    ├─ [DB] 查询币种表
    └─ 返回：精度/小数位数/千分位符号/小数点符号/货币符号/币种类型/状态
```

**调用链路说明**：纯查询，面向内部服务。与 ListActiveCurrencies 区别在于返回更多内部字段（精度规则、格式化规则），且不限制状态=启用。
**被以下接口/模块调用**：
- 钱包模块所有涉及金额处理的接口（精度和格式化）
- 财务模块的订单金额处理
- 未来投注/礼物模块的金额处理

---

### 1.11 `CalcExchangeAmount` — 汇率换算计算（RPC对外）

```
调用方(RPC) → CalcExchangeAmount
    │
    ├─ [入参] 源币种、目标币种、金额、方向（入款/出款）
    ├─ 内部调用 GetExchangeRate 的同逻辑（获取对应方向的汇率）
    ├─ [计算] 换算金额 = 金额 × 汇率（或 金额 / 汇率，取决于方向）
    ├─ [计算] 按目标币种精度四舍五入（法币2位/加密8位）
    └─ 返回：换算金额 + 使用汇率 + 精度
```

**调用链路说明**：这是统一的汇率换算入口，避免各模块自行计算导致精度不一致。内部复用 GetExchangeRate 的汇率获取逻辑 + GetCurrencyConfig 的精度规则。
**被以下接口调用**：
- 钱包.PreviewExchange → 兑换试算
- 钱包.CreateExchangeOrder → 兑换执行
- 钱包.CreateDepositOrder → USDT充值时金额换算
- 财务模块 → 统计报表金额转换

---

### 1.12 `GetPlatformRate` — 批量获取平台汇率（RPC对外）

```
调用方(RPC) → GetPlatformRate
    │
    ├─ [入参] 币种代码列表（批量）
    ├─ [DB] 批量查询币种表的平台汇率字段
    └─ 返回：各币种的平台汇率Map
```

**调用链路说明**：与 GetExchangeRate 区别在于批量且不区分入款/出款方向，面向内部统计/对账场景。

---

### 1.13 汇率偏差检测定时任务（T1）

```
定时触发（cron） → 汇率偏差检测
    │
    ├─ [HTTP] 拉取3家三方汇率API → 取平均值 = 实时汇率
    ├─ [DB] 查询所有启用币种的当前平台汇率、汇率浮动%、阈值%
    ├─ [逐币种处理]
    │    ├─ 计算入款偏差 = |实时汇率 - 当前平台汇率| / 实时汇率 × 100%
    │    ├─ 计算出款偏差（同公式）
    │    ├─ 如果 入款偏差 ≥ 阈值% 或 出款偏差 ≥ 阈值%：
    │    │    ├─ [DB] 更新该币种的平台汇率 = 实时汇率快照
    │    │    ├─ [计算] 重算入款汇率 = 新平台汇率 × (1 + 浮动%)
    │    │    ├─ [计算] 重算出款汇率 = 新平台汇率 × (1 - 浮动%)
    │    │    ├─ [DB] 写入汇率日志表（日志编号/时间/币种/实时汇率/平台汇率/入款汇率/出款汇率）
    │    │    └─ [RPC] ser-blog.AddActLog（审计日志）
    │    └─ 如果 偏差 < 阈值%：跳过，不更新
    └─ 结束
```

**调用链路说明**：这不是接口，是后台定时任务。它是 GetExchangeRate / CalcExchangeAmount / GetPlatformRate 数据准确性的根基——没有这个定时任务运行，平台汇率就是静止的。
**需求依据**：需求文档4.3——"系统定时拉取3家三方汇率API，取平均值作为实时汇率"，"偏差值≥阈值%时更新平台汇率"，"入款和出款偏差值分别计算，任一方≥阈值都触发"。

---

## 二、钱包模块：各接口内部调用链路

> 说明：钱包模块的接口比币种配置复杂得多，很多接口内部会调用币种配置的 RPC 接口，也会调用外部服务。
> 以下用 `[内部]` 标记调用自己模块内的方法/接口，`[币种]` 标记调用币种配置模块的接口，`[外部RPC]` 标记调用其他服务。

### 2.1 `GetWalletOverview` — 钱包概览（C端）

```
前端(C端) → gate-font → GetWalletOverview
    │
    ├─ [入参] 用户ID + 当前币种代码
    ├─ [内部] 如果用户该币种下首次访问 → 初始化子钱包账户（中心/奖励/主播，代理/场馆预留）
    ├─ [DB] 查询子钱包余额表（中心/奖励/主播/代理）
    ├─ [计算] 资金钱包余额 = 中心钱包余额 + 奖励钱包余额
    ├─ [币种] GetCurrencyConfig → 获取精度规则，格式化余额数字
    └─ 返回：资金钱包余额 + 各子钱包余额明细
```

**调用链路说明**：主要是DB查询 + 币种配置获取精度。首次访问时有一个初始化逻辑——为用户在该币种下创建子钱包账户记录。
**需求依据**：需求文档4.1——"资金钱包余额 = 中心钱包余额 + 奖励钱包余额"，原型2-1个人中心余额展示。

---

### 2.2 `GetWalletDetail` — 子钱包明细（C端）

```
前端(C端) → gate-font → GetWalletDetail
    │
    ├─ [入参] 用户ID + 当前币种代码
    ├─ [DB] 查询子钱包余额表
    ├─ [币种] GetCurrencyConfig → 精度格式化
    └─ 返回：各子钱包（中心/奖励/主播/代理）独立余额 + 冻结金额
```

**调用链路说明**：与 GetWalletOverview 类似，但返回更详细的子钱包信息（含冻结金额）。可考虑与 GetWalletOverview 合并为一个接口通过参数控制返回粒度。

---

### 2.3 `GetDepositConfig` — 充值配置（C端）

```
前端(C端) → gate-font → GetDepositConfig
    │
    ├─ [入参] 用户ID + 当前币种代码
    ├─ [币种] ListActiveCurrencies → 确认当前币种有效
    ├─ [币种] GetExchangeRate(方向=入款) → 获取入款汇率（BSB充值页需展示"1 BSB = X USDT"）
    ├─ [待确认] 获取可用充值方式列表：
    │    方案A: [RPC] ser-finance.QueryDepositMethod → 读取财务的支付配置
    │    方案B: [DB] 查询本地缓存的支付配置表
    ├─ [DB/RPC] 获取各充值方式的配置（单笔限额/默认快捷金额档位/手续费）
    ├─ [内部] ListAvailableBonus 的同逻辑 → 获取当前可选的额外奖金活动列表
    └─ 返回：可用充值方式列表 + 各方式配置 + 汇率（BSB才有）+ 奖金活动列表
```

**调用链路说明**：这是一个**聚合接口**，汇总多个数据源为充值页面提供一次性渲染所需的全部数据。内部调用了币种配置的汇率接口，也可能调用财务模块的支付配置（待确认方向）。
**需求依据**：需求文档5.1——"每个币种有默认充值方式和默认最小档位快捷金额"，原型3-1~3-4各币种充值中心页面。"只有BSB充值页有汇率展示和兑换所得金额"。

---

### 2.4 `CreateDepositOrder` — 创建充值订单（C端）

```
前端(C端) → gate-font → CreateDepositOrder
    │
    ├─ [入参] 用户ID + 币种 + 充值方式 + 金额 + 奖金活动ID(可选)
    │
    ├─ [校验] 金额 ≥ 该方式最低限额
    │
    ├─ [内部] 重复转账拦截（仅银行转账和电子钱包，USDT不拦截）：
    │    ├─ [DB] 查询：同UserID + 同币种 + 同金额，状态=待支付/进行中 的订单数
    │    ├─ 如果 1~2笔 → 返回软拦截标记（前端弹窗警告，用户可选择继续）
    │    ├─ 如果 ≥3笔 → 返回硬拦截标记（前端弹窗禁止，不创建订单）
    │    └─ 如果 0笔 → 继续
    │
    ├─ [校验] 如果选了奖金活动：
    │    ├─ 校验金额 ≥ 该活动最低充值门槛
    │    ├─ 校验该订单未重复参与（单笔互斥）
    │    └─ 记录奖金活动ID和比例到订单
    │
    ├─ [处理] 如果是USDT充值（需要汇率换算）：
    │    ├─ [币种] CalcExchangeAmount(源=当前币种, 目标=USDT, 方向=入款)
    │    └─ 订单记录：原始金额 + 换算后USDT金额 + 使用汇率
    │
    ├─ [DB] 生成订单号（C+16位随机数）→ 插入充值订单表
    │
    ├─ [处理] 按充值方式分发：
    │    ├─ USDT → 不需要调财务，后续由 GetDepositAddress 生成收款地址
    │    ├─ 银行转账/电子钱包 → [待确认] 通知财务模块选通道创建支付
    │    │    方案A: [RPC] ser-finance.CreatePayment(订单号, 币种, 金额, 方式) → 返回三方跳转URL
    │    │    方案B: 写入消息队列，finance 消费后处理
    │
    ├─ [RPC] ser-blog.AddActLog（审计日志）
    └─ 返回：订单号 + 订单状态 + 跳转信息（三方URL或进入USDT地址页）
```

**调用链路说明**：这是充值流程最核心的接口，内部逻辑分支较多——重复转账拦截、奖金校验、USDT汇率换算、按充值方式分发。USDT场景在模块内闭环，银行/电子钱包场景需要与财务模块交互。
**需求依据**：需求文档5.2~5.5——充值三种方式各自的流程描述 + 重复转账拦截规则（1-2笔软拦截，≥3笔硬拦截）。

---

### 2.5 `GetDepositAddress` — USDT收款地址（C端）

```
前端(C端) → gate-font → GetDepositAddress
    │
    ├─ [入参] 用户ID + 订单号 + 网络类型（TRC20/ERC20/BEP20）
    ├─ [校验] 订单存在且状态正确
    ├─ [处理] 生成/获取该用户该网络类型的收款地址
    │    （可能是预分配地址池，也可能调用链上服务动态生成）
    ├─ [处理] 生成收款二维码
    └─ 返回：收款地址 + 二维码 + 网络类型 + 预计到账时间
```

**调用链路说明**：仅USDT充值场景使用，依赖 CreateDepositOrder 先创建了订单。生成收款地址的具体方式（地址池 vs 链上生成）取决于技术方案。
**需求依据**：需求文档5.2——"平台自己的页面（不跳三方），需要自己生成收款地址和二维码"，"网络选择：TRC20（默认）、ERC20、BEP20"。

---

### 2.6 `QueryDepositStatus` — 充值状态查询（C端）

```
前端(C端) → gate-font → QueryDepositStatus
    │
    ├─ [入参] 订单号
    ├─ [DB] 查询充值订单表的当前状态
    └─ 返回：状态（待支付/进行中/已完成/失败）+ 更新时间
```

**调用链路说明**：纯查询，C端充值进行中页面轮询使用。无外部调用。
**需求依据**：原型7-2充值进行中弹窗——"预计N分钟到账"。

---

### 2.7 `ListAvailableBonus` — 可选奖金活动（C端）

```
前端(C端) → gate-font → ListAvailableBonus
    │
    ├─ [入参] 用户ID + 当前币种
    ├─ [DB/RPC] 查询当前有效的额外奖金活动列表
    │    （数据源可能是活动模块或本地配置表，待评审确认）
    ├─ [处理] 过滤当前币种适用的活动
    └─ 返回：活动列表（活动ID/名称/奖金比例%/最低充值门槛/流水倍数要求）
```

**调用链路说明**：查询接口，数据源是额外奖金活动的配置。
**需求依据**：需求文档5.6——"默认'无奖金'，点击弹出半屏浮窗选择"，"有最低充值门槛"，"同一笔充值订单只能参与一个奖金活动"。

---

### 2.8 `PreviewExchange` — 兑换预览试算（C端）

```
前端(C端) → gate-font → PreviewExchange
    │
    ├─ [入参] 用户ID + 源币种（法币）+ 金额
    ├─ [校验] 源币种必须是法币（VND/IDR/THB），目标固定为 BSB
    ├─ [校验] 金额 ≤ 当前法币子钱包余额 → [DB] 查余额
    │    如果超出 → 返回提示"已达当前可兑换上限"
    ├─ [币种] GetExchangeRate(源=法币, 目标=BSB, 方向=出款)
    │    → 获取汇率，展示口径：1 BSB = X 法币
    ├─ [币种] CalcExchangeAmount(法币→BSB)
    │    → 兑换获得 BSB = 法币金额 / 汇率
    ├─ [计算] 赠送 BSB = 兑换获得 × 赠送比例%
    ├─ [计算] 实际到账 BSB = 兑换获得 + 赠送
    ├─ [币种] GetCurrencyConfig(BSB) → 精度处理（BSB保留2位小数）
    └─ 返回：兑换获得BSB + 赠送BSB + 实际到账BSB + 使用汇率 + 赠送比例
```

**调用链路说明**：调用了币种配置的三个接口（GetExchangeRate + CalcExchangeAmount + GetCurrencyConfig）。这是兑换功能的"试算"步骤，不产生实际资金变动。
**需求依据**：需求文档6.2~6.3——兑换计算公式和展示口径。

---

### 2.9 `CreateExchangeOrder` — 执行兑换（C端）

```
前端(C端) → gate-font → CreateExchangeOrder
    │
    ├─ [入参] 用户ID + 源币种 + 金额（+ 前端传入的预览汇率用于比对）
    ├─ [校验] 源币种 = 法币，目标 = BSB（仅法币→BSB单向）
    ├─ [校验] 金额 ≤ 当前法币余额 → [DB] 查余额
    │
    ├─ [币种] CalcExchangeAmount(法币→BSB) → 重新计算最终金额
    │    （不能直接用 PreviewExchange 的结果，因为两次调用之间汇率可能变化）
    ├─ [可选] 比对前端传入的预览汇率与当前实际汇率，偏差过大则提示用户
    │
    ├─ [DB] 事务操作：
    │    ├─ 法币子钱包余额 -= 法币金额
    │    ├─ BSB中心钱包余额 += 兑换获得BSB
    │    ├─ BSB奖励钱包余额 += 赠送BSB（如果有赠送，标记流水稽核要求）
    │    ├─ 写入兑换订单记录
    │    └─ 写入资金流水记录（法币扣减流水 + BSB入账流水）
    │
    ├─ [RPC] ser-blog.AddActLog（审计日志）
    └─ 返回：兑换成功 + 实际到账BSB金额（3秒内返回）
```

**调用链路说明**：这是兑换功能的核心执行接口。全程在钱包模块内部闭环，不涉及财务模块，不涉及三方支付。内部事务操作涉及两个币种的子钱包余额变动 + 流水记录，必须保证原子性。
**需求依据**：需求文档6.1~6.5——"仅法币→BSB单向兑换"，"赠送部分需完成X倍流水后可提现"，"3秒内兑换成功弹窗提示"。

---

### 2.10 `GetWithdrawConfig` — 提现配置（C端）

```
前端(C端) → gate-font → GetWithdrawConfig
    │
    ├─ [入参] 用户ID + 当前币种
    │
    ├─ [外部RPC] ser-kyc.GetUserKycStatus(用户ID)
    │    ├─ 未认证 → 可用提现方式 = 仅USDT
    │    └─ 已认证 → 可用提现方式 = 全渠道（银行卡/电子钱包/USDT）
    │
    ├─ [DB] 查询用户充值历史 → 确定提现币种规则：
    │    ├─ 仅法币充值 → 允许法币提现
    │    ├─ 仅USDT充值 → 允许USDT提现
    │    ├─ 法币+USDT混合 → 允许法币提现
    │    └─ 法币→兑换BSB → 追溯原始，允许法币提现
    │
    ├─ [待确认] 获取提现方式限额/手续费/到账时间配置：
    │    方案A: [RPC] ser-finance.QueryWithdrawMethod → 读取财务的支付配置
    │    方案B: [DB] 查询本地缓存的提现配置表
    │
    ├─ [DB] 查询用户当日已提现总额（用于单日限额校验）
    │
    ├─ [币种] GetExchangeRate(方向=出款)
    │    → BSB提现时需展示"提现 X BSB ≈ Y 法币/USDT"
    │
    └─ 返回：KYC状态 + 可用提现方式列表 + 各方式配置(限额/手续费/到账时间) + 当日已提额度 + 汇率(BSB场景)
```

**调用链路说明**：这是另一个**聚合接口**，比 GetDepositConfig 更复杂——需要综合 KYC 状态、充值历史、财务配置、当日额度四方面数据。其中 KYC 是跨服务的外部 RPC 调用。
**需求依据**：需求文档7.1~7.2——KYC 与提现的关系表 + 提现币种规则表。

---

### 2.11 `GetWithdrawAccount` — 查询提现账户（C端）

```
前端(C端) → gate-font → GetWithdrawAccount
    │
    ├─ [入参] 用户ID + 提现方式类型（银行卡/电子钱包/USDT）
    ├─ [DB] 查询提现账户表
    │    ├─ 有记录 → 返回已绑定的账户信息（只读，不可修改）
    │    └─ 无记录 → 返回空（前端进入首次填写流程）
    └─ 返回：银行名称/账号/持有人/网点 或 钱包类型/手机号/持有人 或 USDT链地址
```

**调用链路说明**：纯DB查询，无外部调用。
**需求依据**：需求文档7.3——"首次需填写，非首次自动带出已有信息（只读）"。

---

### 2.12 `SaveWithdrawAccount` — 保存提现账户（C端）

```
前端(C端) → gate-font → SaveWithdrawAccount
    │
    ├─ [入参] 用户ID + 提现方式类型 + 账户详情
    ├─ [校验] 该用户该方式类型是否已有账户 → [DB] 查询
    │    已有 → 拒绝（不可自行修改，风控要求）
    │
    ├─ [外部RPC] ser-kyc.GetUserKycInfo(用户ID) → 获取KYC认证姓名
    ├─ [校验] 账户持有人 vs KYC实名：
    │    去除多余空格 + 忽略大小写 → 比对
    │    不一致 → 拒绝并提示
    │
    ├─ [DB] 插入提现账户表
    ├─ [RPC] ser-blog.AddActLog（审计日志）
    └─ 返回：保存成功
```

**调用链路说明**：关键在于 KYC 姓名校验——需要调用 ser-kyc 获取实名信息做比对。这是提现安全的重要环节。
**需求依据**：需求文档7.4~7.5——"账户持有人必须 = KYC认证姓名（忽略大小写和多余空格进行比对）"，"首次填写后平台记录，不可自行修改"。

---

### 2.13 `CreateWithdrawOrder` — 创建提现订单（C端）

```
前端(C端) → gate-font → CreateWithdrawOrder
    │
    ├─ [入参] 用户ID + 币种 + 提现方式 + 金额 + 提现账户ID
    │
    ├─ [校验] 提现账户存在且归属该用户 → [DB] 查询提现账户表
    │
    ├─ [校验] 限额校验：
    │    ├─ 金额 ≥ 单笔最低限额
    │    ├─ 金额 ≤ 单笔最高限额
    │    └─ 当日已提总额 + 本次金额 ≤ 单日限额 → [DB] 查询当日提现记录
    │
    ├─ [计算] 手续费 = 金额 × 手续费率%
    ├─ [计算] 实际到账 = 金额 - 手续费
    │
    ├─ [处理] 如果是BSB提现：
    │    ├─ [币种] CalcExchangeAmount(BSB→目标币种, 方向=出款)
    │    └─ 记录换算金额和使用汇率
    │
    ├─ [DB] 事务操作（原子）：
    │    ├─ 生成订单号（T+16位随机数）→ 插入提现订单表
    │    ├─ [内部] WalletFreeze 逻辑：冻结用户钱包对应金额
    │    │    ├─ 子钱包可用余额 -= 冻结金额
    │    │    ├─ 子钱包冻结余额 += 冻结金额
    │    │    └─ 写入冻结记录（关联提现单号）
    │    └─ 写入资金流水记录
    │
    ├─ [跨模块] 通知财务模块开始审核（待确认机制）：
    │    方案A: [RPC] ser-finance.SubmitWithdrawReview(订单号)
    │    方案B: 写入消息队列，finance 消费
    │
    ├─ [RPC] ser-blog.AddActLog（审计日志）
    └─ 返回：订单号 + 状态(审核中) + 预计到账时间
```

**调用链路说明**：这是提现流程最核心的接口。内部包含**限额校验 + 冻结操作 + 通知财务**三大关键步骤。其中**创建订单和冻结必须在同一个事务中**——不能出现"订单创建了但钱没冻结"或"钱冻结了但订单没创建"的情况。通知财务审核的机制待确认。
**需求依据**：需求文档7.1~7.6——"用户发起 → 钱包冻结对应金额 → 风控审核 → 财务审核 → 出款"。

---

### 2.14 `QueryRecordPage` — 记录分页查询（C端）

```
前端(C端) → gate-font → QueryRecordPage
    │
    ├─ [入参] 用户ID + 当前币种 + 记录类型（充值/提现/兑换/奖励）+ 分页参数
    ├─ [DB] 按类型查询对应订单表，过滤条件：用户ID + 币种（币种隔离）
    ├─ [币种] GetCurrencyConfig → 格式化金额展示
    └─ 返回：记录列表（金额/方式/时间/状态/状态色标）
```

**调用链路说明**：纯查询，按当前币种隔离记录。四个Tab对应不同的数据源表。
**需求依据**：需求文档8.1~8.3——"当前币种 = 记录币种，切换币种即切换整套记录"。

---

### 2.15 `GetRecordDetail` — 记录详情（C端）

```
前端(C端) → gate-font → GetRecordDetail
    │
    ├─ [入参] 订单号 + 记录类型
    ├─ [DB] 查询订单详情
    ├─ [币种] GetCurrencyConfig → 格式化金额
    └─ 返回：订单号/金额明细/状态/时间/方式/失败原因(如有)
```

**调用链路说明**：纯查询。失败记录会返回失败原因分类（充值失败：支付超时/支付异常/卡片问题；提现失败：信息有误/系统维护/额度受限）。
**需求依据**：需求文档8.4~8.5——"提供'更换其他提现/充值方式'链接"。

---

### 2.16 `GetUserBalance` — 内部余额查询（RPC对外）

```
调用方(finance等) → RPC → GetUserBalance
    │
    ├─ [入参] 用户ID + 币种代码
    ├─ [DB] 查询子钱包余额表
    └─ 返回：各子钱包（中心/奖励/主播/代理）的可用余额 + 冻结余额 + 总余额
```

**调用链路说明**：与 C 端 GetWalletOverview 类似，但返回更多内部字段（含冻结金额），且面向内部 RPC 调用。
**被调用场景**：财务模块人工加/减款弹窗展示用户当前余额。

---

### 2.17 `WalletCredit` — 充值入账（RPC对外，被finance调用）

```
finance → RPC → WalletCredit
    │
    ├─ [入参] 订单号 + 用户ID + 币种 + 金额 + 来源类型(充值/补单)
    │
    ├─ [校验] 幂等检查 → [DB] 查询该订单号是否已入账
    │    已入账 → 直接返回成功（幂等）
    │
    ├─ [DB] 事务操作：
    │    ├─ 中心钱包余额 += 入账金额
    │    ├─ 更新充值订单状态 → 已完成
    │    └─ 写入资金流水记录（类型=充值入账，关联订单号）
    │
    ├─ [RPC] ser-blog.AddActLog（审计日志）
    └─ 返回：入账成功 + 入账后余额
```

**调用链路说明**：这是财务模块最频繁调用钱包的接口。幂等处理是核心要求——因为财务收到三方回调后调用此接口，网络问题可能导致重试，必须保证不重复入账。
**需求依据**：需求文档4.9——"充值回调必须做幂等处理，防止重复入账"。

---

### 2.18 `WalletCreditReward` — 奖金入账（RPC对外，被finance调用）

```
finance → RPC → WalletCreditReward
    │
    ├─ [入参] 关联订单号 + 用户ID + 币种 + 奖金金额 + 流水倍数要求 + 来源(充值奖金/活动奖励)
    │
    ├─ [校验] 幂等检查（同 WalletCredit）
    │
    ├─ [DB] 事务操作：
    │    ├─ 奖励钱包余额 += 奖金金额
    │    ├─ 写入流水稽核要求记录（奖金金额 × 流水倍数 = 需完成的投注/消费流水）
    │    └─ 写入资金流水记录（类型=奖金入账）
    │
    ├─ [RPC] ser-blog.AddActLog
    └─ 返回：入账成功
```

**调用链路说明**：紧跟 WalletCredit 之后调用。区别在于金额入**奖励钱包**而非中心钱包，且需要标记流水稽核要求。
**需求依据**：需求文档5.6——"入账规则：充值金额→中心钱包，奖金金额→奖励钱包"，"奖金部分需完成N倍流水后可提现"。

---

### 2.19 `WalletFreeze` — 冻结（RPC对外，被finance调用或内部调用）

```
finance / 内部 → RPC → WalletFreeze
    │
    ├─ [入参] 用户ID + 币种 + 冻结金额 + 关联提现单号
    │
    ├─ [校验] 可用余额 ≥ 冻结金额 → [DB] 查询子钱包余额
    │    不足 → 拒绝
    │
    ├─ [DB] 事务操作：
    │    ├─ 子钱包可用余额 -= 冻结金额
    │    ├─ 子钱包冻结余额 += 冻结金额
    │    └─ 插入冻结记录（冻结ID/金额/关联单号/状态=冻结中）
    │
    └─ 返回：冻结成功 + 冻结记录ID
```

**调用链路说明**：在 CreateWithdrawOrder 的事务中被内部调用，或由 finance 通过 RPC 调用（取决于架构方案）。冻结的金额在 WalletUnfreeze 或 WalletDeduct 中结束。
**需求依据**：需求文档7.6——"用户发起 → 钱包冻结对应金额"。

---

### 2.20 `WalletUnfreeze` — 解冻（RPC对外，被finance调用）

```
finance → RPC → WalletUnfreeze
    │
    ├─ [入参] 冻结记录ID 或 关联提现单号 + 解冻原因(驳回/出款失败)
    │
    ├─ [校验] 冻结记录存在且状态=冻结中
    │
    ├─ [DB] 事务操作：
    │    ├─ 子钱包冻结余额 -= 冻结金额
    │    ├─ 子钱包可用余额 += 冻结金额（归还）
    │    ├─ 更新冻结记录状态 → 已解冻
    │    └─ 写入资金流水记录（类型=解冻归还）
    │
    ├─ [RPC] ser-blog.AddActLog
    └─ 返回：解冻成功 + 归还后可用余额
```

**调用链路说明**：由 finance 在三种场景触发——风控审核驳回、财务审核驳回、代付出款失败。本质是"冻结的逆操作"，把钱还给用户的可用余额。
**需求依据**：财务需求文档"驳回后：通知钱包解冻金额"。

---

### 2.21 `WalletDeduct` — 最终扣款（RPC对外，被finance调用）

```
finance → RPC → WalletDeduct
    │
    ├─ [入参] 冻结记录ID 或 关联提现单号
    │
    ├─ [校验] 冻结记录存在且状态=冻结中
    │
    ├─ [DB] 事务操作：
    │    ├─ 子钱包冻结余额 -= 冻结金额（冻结部分扣除，不影响可用余额）
    │    ├─ 更新冻结记录状态 → 已扣款
    │    ├─ 更新提现订单状态 → 已完成
    │    └─ 写入资金流水记录（类型=提现扣款）
    │
    ├─ [RPC] ser-blog.AddActLog
    └─ 返回：扣款成功
```

**调用链路说明**：由 finance 在代付出款成功后调用。这一步是提现的最终动作——冻结的钱正式离开用户账户。与 WalletUnfreeze 互斥：同一笔冻结要么解冻要么扣款，不能同时发生。
**需求依据**：需求文档——"审核通过+出款成功 → 最终扣除冻结资金"。

---

### 2.22 `WalletManualAdd` — 人工加款（RPC对外，被finance调用）

```
finance → RPC → WalletManualAdd
    │
    ├─ [入参] 用户ID + 币种 + 各子钱包加款金额Map{中心:X, 奖励:Y, 主播:Z, 代理:W} + 关联修正单号 + 加款类型
    │
    ├─ [校验] 幂等检查（按修正单号）
    │
    ├─ [DB] 事务操作：
    │    ├─ 中心钱包余额 += X（如果X>0）
    │    ├─ 奖励钱包余额 += Y（如果Y>0）
    │    ├─ 主播钱包余额 += Z（如果Z>0）
    │    ├─ 代理钱包余额 += W（如果W>0）
    │    └─ 写入资金流水记录（各子钱包分别记录，类型=人工加款，关联修正单号）
    │
    ├─ [RPC] ser-blog.AddActLog
    └─ 返回：加款成功 + 各子钱包加款后余额
```

**调用链路说明**：由 finance 的人工加款审核通过后调用。特点是**按子钱包分别操作**，不是简单的"给用户加钱"，而是精确到中心/奖励/主播/代理各自加多少。
**需求依据**：财务需求文档"人工加款"——"选择用户 → 查询展示当前四种子钱包余额 → 分别填写加款金额"。

---

### 2.23 `WalletManualSub` — 人工减款（RPC对外，被finance调用）

```
finance → RPC → WalletManualSub
    │
    ├─ [入参] 用户ID + 币种 + 各子钱包减款金额Map{中心:X, 奖励:Y, 主播:Z, 代理:W} + 关联修正单号 + 减款类型
    │
    ├─ [校验] 幂等检查（按修正单号）
    │
    ├─ [DB] 事务操作（逐子钱包处理）：
    │    ├─ 中心钱包：实际扣款 = min(X, 当前余额)，余额 -= 实际扣款
    │    ├─ 奖励钱包：实际扣款 = min(Y, 当前余额)，余额 -= 实际扣款
    │    ├─ 主播钱包：实际扣款 = min(Z, 当前余额)，余额 -= 实际扣款
    │    ├─ 代理钱包：实际扣款 = min(W, 当前余额)，余额 -= 实际扣款
    │    └─ 写入资金流水记录（记录期望减款和实际减款）
    │
    ├─ [RPC] ser-blog.AddActLog
    └─ 返回：减款成功 + 各子钱包实际扣款金额 + 扣款后余额
```

**调用链路说明**：与 WalletManualAdd 对应。核心区别是"不超余额"规则——每个子钱包的实际扣款 = min(期望扣款, 当前余额)，保证不会出现负余额。返回值需要包含**实际扣款金额**，因为可能与期望金额不同。
**需求依据**：财务需求文档——"如果减款金额 > 当前余额，实际扣款 = 当前余额（不会变成负数）"。

---

### 2.24 `DepositNotify` — USDT充值回调（回调接口）

```
区块链监听服务 → DepositNotify
    │
    ├─ [入参] 链上交易ID + 收款地址 + 到账金额(USDT) + 网络类型 + 确认数
    │
    ├─ [校验] 根据收款地址匹配用户和订单 → [DB] 查询
    ├─ [校验] 确认数 ≥ 安全阈值（防双花）
    ├─ [校验] 幂等检查（同一链上交易ID不重复处理）
    │
    ├─ [币种] CalcExchangeAmount(USDT→用户币种, 方向=入款)
    │    → 将USDT到账金额换算为用户钱包币种金额
    │
    ├─ [内部] 复用 WalletCredit 逻辑：
    │    ├─ 中心钱包余额 += 换算后金额
    │    ├─ 更新充值订单状态 → 已完成
    │    └─ 写入资金流水记录
    │
    ├─ [内部] 如果订单关联了奖金活动 → 复用 WalletCreditReward 逻辑
    │
    ├─ [RPC] ser-blog.AddActLog
    └─ 返回：处理结果
```

**调用链路说明**：USDT链上充值的回调处理。这个接口的归属待确认——如果由 wallet 直接处理（方案A），就是上述链路；如果由 finance 统一处理（方案B），则 finance 收到链上回调后再调 wallet.WalletCredit。
**需求依据**：需求文档5.2——"区块链监听服务确认到账 → 自动上账"。

---

## 三、模块内部调用关系（币种配置 ↔ 钱包）

> 说明：币种配置和钱包虽然在同一个 ser-wallet 服务中，但逻辑上是两个独立模块。以下梳理它们之间的调用关系。
> 方向是单向的：**钱包 → 币种配置**（钱包调用币种配置的能力，币种配置不调用钱包）。

### 3.1 调用关系汇总

| 钱包侧接口（调用方） | 调用的币种配置接口 | 调用目的 |
|---------------------|------------------|---------|
| `GetWalletOverview` | → `GetCurrencyConfig` | 格式化余额展示（精度/千分位） |
| `GetWalletDetail` | → `GetCurrencyConfig` | 同上 |
| `GetDepositConfig` | → `ListActiveCurrencies` | 确认当前币种有效 |
| `GetDepositConfig` | → `GetExchangeRate`(入款) | BSB充值页展示汇率 |
| `CreateDepositOrder` | → `CalcExchangeAmount` | USDT充值金额换算 |
| `PreviewExchange` | → `GetExchangeRate`(出款) | 获取兑换汇率 |
| `PreviewExchange` | → `CalcExchangeAmount` | 计算兑换BSB数量 |
| `PreviewExchange` | → `GetCurrencyConfig` | BSB精度处理 |
| `CreateExchangeOrder` | → `CalcExchangeAmount` | 重新计算最终兑换金额 |
| `GetWithdrawConfig` | → `GetExchangeRate`(出款) | BSB提现展示换算金额 |
| `CreateWithdrawOrder` | → `CalcExchangeAmount` | BSB提现金额换算 |
| `QueryRecordPage` | → `GetCurrencyConfig` | 格式化金额展示 |
| `GetRecordDetail` | → `GetCurrencyConfig` | 格式化金额展示 |
| `DepositNotify` | → `CalcExchangeAmount` | USDT到账金额换算为用户币种 |

### 3.2 被调用频次排名

| 币种配置接口 | 被钱包侧调用次数 | 说明 |
|------------|---------------|------|
| `GetCurrencyConfig` | 5次 | 所有涉及金额展示的接口都需要，使用频率最高 |
| `CalcExchangeAmount` | 4次 | 所有涉及跨币种换算的接口都需要 |
| `GetExchangeRate` | 3次 | 充值/兑换/提现页面展示汇率 |
| `ListActiveCurrencies` | 1次 | 充值配置获取时确认币种有效 |

### 3.3 关键认知

由于币种配置和钱包在同一个 ser-wallet 服务进程中，这些"调用"在代码层面是**同进程内的方法调用**，不走网络 RPC。性能开销很小，但需要注意：
- 币种配置数据适合做**进程内缓存**（币种列表和配置变化频率低）
- 汇率数据也可以缓存，但需要在定时任务更新后刷新缓存
- GetCurrencyConfig 和 CalcExchangeAmount 被调用频率极高，缓存收益大

---

## 四、跨模块调用链路（wallet ↔ finance）

> 说明：这是最关键的部分——wallet 和 finance 分属两个不同的服务进程，走真实的网络 RPC 调用。

### 4.1 finance → wallet 的调用（9条RPC链路）

| # | finance 侧的触发动作 | → 调用 wallet 的接口 | wallet 内部执行链路 |
|---|---------------------|---------------------|-------------------|
| 1 | 收到三方代收回调 `CollectNotify`，验证成功后 | → `WalletCredit` | 幂等检查 → 中心钱包+余额 → 写流水 → 审计日志 |
| 2 | 同上，如果订单有额外奖金 | → `WalletCreditReward` | 幂等检查 → 奖励钱包+余额 → 标记流水稽核 → 写流水 |
| 3 | 提现订单创建时（如果冻结由finance发起） | → `WalletFreeze` | 校验余额充足 → 可用余额减/冻结余额加 → 写冻结记录 |
| 4 | 风控审核驳回 `ReviewWithdrawRisk` | → `WalletUnfreeze` | 冻结余额减/可用余额加 → 更新冻结记录 → 写流水 |
| 5 | 财务审核驳回 `ReviewWithdrawFinance` | → `WalletUnfreeze` | 同上 |
| 6 | 收到三方代付回调 `PayoutNotify`，出款失败 | → `WalletUnfreeze` | 同上 |
| 7 | 收到三方代付回调 `PayoutNotify`，出款成功 | → `WalletDeduct` | 冻结余额减（最终扣除）→ 更新冻结记录 → 写流水 |
| 8 | 人工加款审核通过 `ReviewCorrectionOrder` | → `WalletManualAdd` | 各子钱包分别加款 → 写流水 |
| 9 | 人工减款审核通过 `ReviewCorrectionOrder` | → `WalletManualSub` | 各子钱包分别减款(不超余额) → 写流水 → 返回实际扣款 |
| 10 | 补单审核通过 `ReviewCorrectionOrder` | → `WalletCredit` + `WalletCreditReward` | 同第1、2条 |
| 11 | 人工加/减款创建弹窗 `CreateManualAddOrder` / `CreateManualSubOrder` | → `GetUserBalance` | 查询各子钱包余额返回 |
| 12 | 订单金额处理 | → `GetCurrencyConfig` | 返回币种精度和格式化规则 |

### 4.2 wallet → finance 的调用（待确认，1~2条）

| # | wallet 侧的触发动作 | → 可能调用 finance 的接口 | 说明 |
|---|---------------------|-------------------------|------|
| 1 | `CreateDepositOrder`（银行/电子钱包充值） | → finance.CreatePayment（选通道+创建支付+返回跳转URL） | 待确认：是 wallet 主动调 finance，还是 finance 监听 wallet 的订单事件 |
| 2 | `GetDepositConfig` / `GetWithdrawConfig` | → finance.QueryPaymentConfig（读取可用充值/提现方式配置） | 待确认：是实时RPC调还是缓存/配置中心 |

### 4.3 跨模块调用的关键注意事项

**幂等性要求**：finance → wallet 的入账类接口（WalletCredit、WalletCreditReward、WalletManualAdd、WalletManualSub）必须做幂等处理。因为 finance 可能因网络超时重试，wallet 不能重复执行。

**事务边界**：跨服务调用无法做分布式事务。finance 调用 wallet.WalletCredit 后如果 finance 自身后续逻辑失败，wallet 侧已经入账了无法自动回滚。需要通过**补偿机制**处理（finance 调 WalletManualSub 冲正，或人工处理）。

**超时处理**：finance 调 wallet.WalletFreeze 时如果超时，finance 不知道 wallet 是否已经冻结了。需要 wallet 支持**查询冻结状态**（按提现单号查询），finance 超时后先查再决定是否重试。

---

## 五、跨模块调用链路（wallet ↔ 其他服务）

### 5.1 wallet → ser-kyc（2条链路）

| # | wallet 侧接口 | → 调用 ser-kyc 接口 | 用途 |
|---|-------------|-------------------|------|
| 1 | `GetWithdrawConfig` | → `GetUserKycStatus` | 查询KYC认证状态，决定可用提现渠道（未认证仅USDT） |
| 2 | `SaveWithdrawAccount` | → `GetUserKycInfo` | 获取KYC认证姓名，与账户持有人比对一致性 |

### 5.2 wallet → ser-blog（全覆盖）

| # | wallet 侧接口 | → 调用 ser-blog 接口 | 特点 |
|---|-------------|-------------------|------|
| 全部写操作 | 所有 Create/Update/Toggle/Set/Credit/Freeze/Deduct/ManualAdd/Sub | → `AddActLog`(oneway) | 异步不阻塞，工程强制规范 |

涉及接口：CreateCurrency、UpdateCurrency、ToggleCurrencyStatus、SetBaseCurrency、CreateDepositOrder、CreateExchangeOrder、CreateWithdrawOrder、SaveWithdrawAccount、WalletCredit、WalletCreditReward、WalletFreeze、WalletUnfreeze、WalletDeduct、WalletManualAdd、WalletManualSub、DepositNotify。共约16个接口在执行后需要上报审计日志。

### 5.3 wallet → ser-s3（1条链路）

| # | wallet 侧接口 | → 调用 ser-s3 接口 | 用途 |
|---|-------------|-------------------|------|
| 1 | `CreateCurrency` / `UpdateCurrency` | → `UploadFile` | 上传币种图标文件（SVG/PNG ≤5KB） |

### 5.4 wallet → ser-buser（1条链路）

| # | wallet 侧接口 | → 调用 ser-buser 接口 | 用途 |
|---|-------------|---------------------|------|
| 1 | `QueryCurrencyPage`（及其他B端列表接口） | → `QueryUserByIds` | 操作人ID解析为姓名 |

---

## 六、完整业务场景端到端调用链路

> 按用户操作场景，把上面拆散的单接口链路串起来，展示完整的端到端调用序列。

### 6.1 充值场景（USDT）完整链路

```
① C端 → GetDepositConfig
              → [币种] ListActiveCurrencies
              → [币种] GetExchangeRate(入款)
         ← 返回：可用充值方式 + 汇率 + 奖金活动

② C端 → ListAvailableBonus（可选，如果用户点击选择奖金）
         ← 返回：奖金活动列表

③ C端 → CreateDepositOrder(USDT, 金额, 奖金活动ID)
              → [币种] CalcExchangeAmount(用户币种→USDT)
              → [DB] 重复转账拦截检查（USDT不拦截，跳过）
              → [DB] 创建订单(C+16位)
              → [RPC] ser-blog.AddActLog
         ← 返回：订单号

④ C端 → GetDepositAddress(订单号, TRC20)
              → 生成收款地址+二维码
         ← 返回：收款地址 + 二维码

⑤ 用户链上转账...等待区块链确认...

⑥ 区块链监听 → DepositNotify(链上交易ID, 地址, USDT金额)
              → [币种] CalcExchangeAmount(USDT→用户币种)
              → [内部] WalletCredit 逻辑：中心钱包入账
              → [内部] WalletCreditReward 逻辑（如有奖金）：奖励钱包入账
              → [RPC] ser-blog.AddActLog
         ← 处理完成

⑦ C端 → QueryDepositStatus(订单号) ← 已完成
```

### 6.2 充值场景（银行转账/电子钱包）完整链路

```
①② 同USDT的①②

③ C端 → CreateDepositOrder(银行转账, 金额)
              → [DB] 重复转账拦截检查
                   1-2笔同条件订单 → 返回软拦截标记
                   ≥3笔 → 返回硬拦截，终止
              → [DB] 创建订单(C+16位)
              → [跨模块] 通知finance选通道创建支付（待确认方向）
              → [RPC] ser-blog.AddActLog
         ← 返回：订单号 + 三方跳转URL

④ 用户跳转三方托管页完成支付...

⑤ 三方 → finance.CollectNotify(订单号, 支付结果)
              → finance 更新订单状态
              → finance → wallet.WalletCredit(订单号, 金额)
                   → 幂等检查 → 中心钱包入账 → 写流水
              → finance → wallet.WalletCreditReward(如有奖金)
                   → 奖励钱包入账 → 标记流水稽核
         ← 处理完成

⑥ C端 → QueryDepositStatus(订单号) ← 已完成
```

### 6.3 提现场景完整链路

```
① C端 → GetWithdrawConfig(用户ID, 币种)
              → [外部RPC] ser-kyc.GetUserKycStatus → 判断KYC状态
              → [DB] 查充值历史 → 确定提现币种规则
              → [币种] GetExchangeRate(出款)（BSB场景）
         ← 返回：KYC状态 + 可用提现方式 + 限额/手续费/到账时间

② C端 → GetWithdrawAccount(用户ID, 银行卡)
         ← 返回：已绑定账户 或 空（需首次填写）

③ C端 → SaveWithdrawAccount(银行卡信息)（首次才需要）
              → [外部RPC] ser-kyc.GetUserKycInfo → 获取KYC姓名
              → 比对：账户持有人 == KYC姓名（忽略大小写/空格）
              → [DB] 保存账户
              → [RPC] ser-blog.AddActLog
         ← 返回：保存成功

④ C端 → CreateWithdrawOrder(用户ID, 币种, 方式, 金额, 账户ID)
              → [校验] 限额检查（单笔min/max + 单日限额）
              → [计算] 手续费 + 到账金额
              → [币种] CalcExchangeAmount（BSB场景）
              → [DB事务] 创建订单(T+16位) + WalletFreeze(冻结)
              → [跨模块] 通知finance开始审核（待确认机制）
              → [RPC] ser-blog.AddActLog
         ← 返回：订单号 + 状态(审核中)

⑤ B端(风控) → finance.ReviewWithdrawRisk
              → 通过：进入财务审核
              → 驳回：finance → wallet.WalletUnfreeze → 解冻归还

⑥ B端(财务) → finance.ReviewWithdrawFinance
              → 通过：发起代付出款
              → 驳回：finance → wallet.WalletUnfreeze → 解冻归还

⑦ finance → 三方代付通道 → 发起出款

⑧ 三方 → finance.PayoutNotify
              → 出款成功：finance → wallet.WalletDeduct → 最终扣除冻结资金
              → 出款失败：finance → wallet.WalletUnfreeze → 解冻归还

⑨ C端 → QueryRecordPage(提现Tab) → 查看提现状态
```

### 6.4 兑换场景完整链路

```
① C端 → PreviewExchange(法币, 金额)
              → [币种] GetExchangeRate(出款)
              → [币种] CalcExchangeAmount(法币→BSB)
              → [币种] GetCurrencyConfig(BSB) → 精度处理
         ← 返回：兑换BSB + 赠送BSB + 实际到账BSB + 汇率

② C端 → CreateExchangeOrder(法币, 金额)
              → [校验] 金额 ≤ 法币余额
              → [币种] CalcExchangeAmount(法币→BSB) → 重新计算
              → [DB事务] 法币余额减 + BSB中心余额加 + 赠送入奖励钱包(标记流水) + 写流水
              → [RPC] ser-blog.AddActLog
         ← 返回：兑换成功 + 实际到账BSB

说明：全程不涉及 finance 模块，不涉及三方支付，不涉及审核。wallet 内部闭环。
```

### 6.5 人工加款场景完整链路

```
① B端(运营) → finance.CreateManualAddOrder
              → finance → wallet.GetUserBalance(用户ID, 币种)
                   ← 返回：中心余额/奖励余额/主播余额/代理余额（含冻结）
              → 弹窗展示余额，运营分别填写各子钱包加款金额
              → finance 创建修正单(A+16位)

② B端(审核人) → finance.ReviewCorrectionOrder(审核通过)
              → finance → wallet.WalletManualAdd(用户ID, 币种, {中心:X, 奖励:Y, 主播:Z, 代理:W}, 修正单号)
                   → 幂等检查 → 各子钱包分别加款 → 写流水 → 审计日志
              ← 加款成功
```

### 6.6 人工减款场景完整链路

```
① B端(运营) → finance.CreateManualSubOrder
              → finance → wallet.GetUserBalance(用户ID, 币种)
                   ← 返回：各子钱包余额
              → 弹窗展示余额，运营分别填写各子钱包减款金额
              → finance 创建修正单(M+16位)

② B端(审核人) → finance.ReviewCorrectionOrder(审核通过)
              → finance → wallet.WalletManualSub(用户ID, 币种, {中心:X, 奖励:Y, ...}, 修正单号)
                   → 幂等检查 → 各子钱包分别减款(实扣=min(期望,余额)) → 写流水 → 审计日志
              ← 减款成功 + 各子钱包实际扣款金额
```

### 6.7 补单场景完整链路

```
① B端(运营) → finance.CreateSupplementOrder(用户ID, 币种, 充值金额, 额外奖金)
              → finance 创建补单(B+16位)

② B端(审核人) → finance.ReviewCorrectionOrder(审核通过)
              → finance → wallet.WalletCredit(用户ID, 币种, 充值金额, 补单号)
                   → 幂等检查 → 中心钱包入账 → 写流水
              → finance → wallet.WalletCreditReward(如有奖金)(用户ID, 币种, 奖金金额, 流水倍数, 补单号)
                   → 奖励钱包入账 → 标记流水稽核
              ← 入账成功
```

---

## 七、每个接口的"调用谁/被谁调用"速查表

### 7.1 币种配置模块接口速查

| 接口 | 调用了谁 | 被谁调用 |
|------|---------|---------|
| `CreateCurrency` | ser-s3.UploadFile, ser-blog.AddActLog | B端前端 |
| `UpdateCurrency` | ser-s3.UploadFile, ser-blog.AddActLog | B端前端 |
| `ToggleCurrencyStatus` | ser-blog.AddActLog | B端前端 |
| `SetBaseCurrency` | ser-blog.AddActLog | B端前端（一次性） |
| `QueryCurrencyPage` | ser-buser.QueryUserByIds | B端前端 |
| `QueryRateLogPage` | 无 | B端前端 |
| `GetCurrencyDetail` | 无 | B端前端 |
| `ListActiveCurrencies` | 无 | **钱包.GetDepositConfig**、C端前端 |
| `GetExchangeRate` | 无 | **钱包.GetDepositConfig**、**钱包.PreviewExchange**、**钱包.GetWithdrawConfig**、C端前端 |
| `GetCurrencyConfig` | 无 | **钱包.GetWalletOverview**、**钱包.GetWalletDetail**、**钱包.PreviewExchange**、**钱包.QueryRecordPage**、**钱包.GetRecordDetail**、**finance（金额处理）** |
| `CalcExchangeAmount` | GetExchangeRate逻辑 + GetCurrencyConfig逻辑 | **钱包.CreateDepositOrder**、**钱包.PreviewExchange**、**钱包.CreateExchangeOrder**、**钱包.CreateWithdrawOrder**、**钱包.DepositNotify**、**finance（统计换算）** |
| `GetPlatformRate` | 无 | **finance（统计报表）** |
| 汇率定时任务(T1) | 三方汇率API、ser-blog.AddActLog | cron触发 |

### 7.2 钱包模块接口速查

| 接口 | 调用了谁 | 被谁调用 |
|------|---------|---------|
| `GetWalletOverview` | 币种.GetCurrencyConfig | C端前端 |
| `GetWalletDetail` | 币种.GetCurrencyConfig | C端前端 |
| `GetDepositConfig` | 币种.ListActiveCurrencies, 币种.GetExchangeRate, (待确认)finance支付配置 | C端前端 |
| `CreateDepositOrder` | 币种.CalcExchangeAmount, ser-blog.AddActLog, (待确认)finance.CreatePayment | C端前端 |
| `GetDepositAddress` | 无（或链上地址服务） | C端前端 |
| `QueryDepositStatus` | 无 | C端前端（轮询） |
| `ListAvailableBonus` | 无 | C端前端 |
| `PreviewExchange` | 币种.GetExchangeRate, 币种.CalcExchangeAmount, 币种.GetCurrencyConfig | C端前端 |
| `CreateExchangeOrder` | 币种.CalcExchangeAmount, ser-blog.AddActLog | C端前端 |
| `GetWithdrawConfig` | ser-kyc.GetUserKycStatus, 币种.GetExchangeRate, (待确认)finance提现配置 | C端前端 |
| `GetWithdrawAccount` | 无 | C端前端 |
| `SaveWithdrawAccount` | ser-kyc.GetUserKycInfo, ser-blog.AddActLog | C端前端 |
| `CreateWithdrawOrder` | 币种.CalcExchangeAmount, WalletFreeze逻辑, ser-blog.AddActLog, (待确认)finance审核通知 | C端前端 |
| `QueryRecordPage` | 币种.GetCurrencyConfig | C端前端 |
| `GetRecordDetail` | 币种.GetCurrencyConfig | C端前端 |
| `GetUserBalance` | 无 | **finance.CreateManualAddOrder**, **finance.CreateManualSubOrder** |
| `WalletCredit` | ser-blog.AddActLog | **finance.CollectNotify**, **finance.ReviewCorrectionOrder(补单)** |
| `WalletCreditReward` | ser-blog.AddActLog | **finance.CollectNotify(奖金)**, **finance.ReviewCorrectionOrder(补单奖金)** |
| `WalletFreeze` | ser-blog.AddActLog | **CreateWithdrawOrder(内部)** 或 **finance** |
| `WalletUnfreeze` | ser-blog.AddActLog | **finance.ReviewWithdrawRisk(驳回)**, **finance.ReviewWithdrawFinance(驳回)**, **finance.PayoutNotify(失败)** |
| `WalletDeduct` | ser-blog.AddActLog | **finance.PayoutNotify(成功)** |
| `WalletManualAdd` | ser-blog.AddActLog | **finance.ReviewCorrectionOrder(加款通过)** |
| `WalletManualSub` | ser-blog.AddActLog | **finance.ReviewCorrectionOrder(减款通过)** |
| `DepositNotify` | 币种.CalcExchangeAmount, WalletCredit逻辑, WalletCreditReward逻辑, ser-blog.AddActLog | 区块链监听服务 |
