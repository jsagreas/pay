# ser-wallet 需调用的外部 RPC 接口 — 综合分析

> **文档定位**：ser-wallet 模块在业务流程中，需要主动调用工程内其他模块的哪些 RPC 接口
> **与 rpc暴露/tmp-1.md 的关系**：暴露分析 = 别人调用我；本文档 = 我调用别人。方向相反，互为补充
> **分析依据**：136张需求/原型文档 + 工程全量IDL分析（73个thrift文件） + 现有模块实际RPC调用代码 + 前期需求分析
> **分析范围**：ser-wallet 作为 RPC 调用方，需要依赖哪些外部服务的能力
> **核心原则**：只列真正需要的依赖，不过度耦合；能通过内部逻辑解决的不引入外部调用

---

## 一、为什么要分析外部RPC依赖

### 1.1 两个方向缺一不可

```
RPC暴露（已完成）：别人需要我什么 → 我暴露什么接口
RPC调用（本文档）：我需要别人什么 → 我依赖哪些接口
```

不分析清楚"我依赖谁"，会导致：
- 编码时才发现需要别人的接口，对方还没实现 → 阻塞
- 不清楚依赖的接口入参出参 → 联调时才对齐 → 返工
- 遗漏必要的依赖调用（比如漏了KYC校验） → 业务流程不完整

### 1.2 分析方法

```
步骤1：遍历 ser-wallet 的所有业务场景（充值/提现/兑换/币种配置/余额管理等）
步骤2：在每个场景中识别"不是钱包自己的数据/能力，需要从外部获取"的环节
步骤3：确认该能力由哪个现有模块提供、对应哪个RPC接口
步骤4：区分"已有接口可直接调用"和"需要对方新增接口"
步骤5：标注每个依赖的必要性等级和触发频率
```

---

## 二、外部依赖全景

### 2.1 依赖关系图

```
                           ser-wallet
                        (我主动调用别人)
                              │
        ┌─────────┬──────────┼──────────┬──────────┐
        │         │          │          │          │
   ┌────┴───┐ ┌───┴───┐ ┌───┴───┐ ┌────┴───┐ ┌───┴────┐
   │ser-user│ │ser-kyc│ │ser-   │ │ser-s3  │ │ser-    │
   │(用户)  │ │(KYC)  │ │blog   │ │(文件)  │ │buser   │
   │        │ │       │ │(日志) │ │        │ │(B端用户)│
   └────────┘ └───────┘ └───────┘ └────────┘ └────────┘
     强依赖     强依赖     标配依赖    弱依赖     弱依赖
```

### 2.2 依赖清单总览

| 序号 | 被调用模块 | 接口数量 | 依赖强度 | 调用场景 |
|------|-----------|---------|---------|---------|
| 1 | **ser-user（用户服务）** | 2个 | **强依赖** | 提现验证用户状态、B端展示用户信息 |
| 2 | **ser-kyc（KYC认证）** | 1个 | **强依赖** | 提现前KYC状态与实名信息校验 |
| 3 | **ser-blog（操作日志）** | 2个 | **标配依赖** | 所有B端操作审计日志 + B端日志查询 |
| 4 | **ser-s3（文件存储）** | 1个 | **弱依赖** | B端币种配置上传币种图标 |
| 5 | **ser-buser（B端用户）** | 1个 | **弱依赖** | B端操作记录展示操作人名称 |

**合计：7个外部RPC接口调用**

---

## 三、强依赖 — ser-user（用户服务）

### 3.1 为什么是强依赖

ser-wallet 的多个核心业务场景需要获取用户信息：
- 提现时需要验证用户账号状态（是否封禁、是否正常）
- 提现时需要获取 kycStatus 判断可用提现渠道
- B端人工修正时需要展示被操作用户的基本信息
- B端充值/提现记录查询需要关联展示用户昵称、UID等

### 3.2 接口1：GetUserInfoExpend — 获取单用户信息

```
调用方：ser-wallet
被调用方：ser-user
ETCD服务名：user_service
IDL位置：/Users/mac/gitlab/common/idl/ser-user/service.thrift
接口状态：已存在，可直接调用
```

**接口签名**：
```thrift
user_rpc.UserInfoExpendResp GetUserInfoExpend(1: user_rpc.GetUserInfoReq req)
```

**入参**：
| 字段 | 类型 | 说明 |
|------|------|------|
| userId | i64 | 用户ID |
| uid | i64 | 用户业务UID |

**返回的关键字段（UserInfoExpendResp）**：
| 字段 | 类型 | 钱包使用场景 |
|------|------|-------------|
| userId | i64 | 关联钱包记录 |
| uid | i64 | 展示给用户 |
| nickname | string | B端记录展示 |
| status | i32 | 验证用户是否被封禁（封禁用户禁止充值/提现） |
| accountType | i32 | 区分正式用户/游客（游客限制部分功能） |
| countryCode | string | 判断默认币种（越南→VND，印尼→IDR，泰国→THB） |
| level | i32 | VIP等级（可能影响提现限额） |

**在钱包业务中的调用场景**：

| 场景 | 触发时机 | 使用的返回字段 | 频率 |
|------|---------|-------------|------|
| C端提现 | 用户进入提现页面 | status（封禁校验）、accountType（游客限制） | 高频 |
| C端充值 | 用户发起充值 | status、countryCode（默认币种推断） | 高频 |
| B端人工修正 | 财务人员输入用户ID查询 | nickname、uid（弹窗展示用户信息） | 低频 |
| B端记录查询 | 充值/提现记录列表 | nickname、uid（关联展示） | 中频 |

**代码调用示例**（参考工程现有模式）：
```go
// ser-wallet/internal/ser/wallet_service.go
userInfo, err := rpc.UserClient().GetUserInfoExpend(ctx, &ser_user.GetUserInfoReq{
    UserId: req.UserId,
})
if err != nil {
    lg.Log().CtxErrorf(ctx, "调用ser-user获取用户信息失败: %v", err)
    return nil, ret.BizErr(ctx, errs.RPCCallError)
}
// 校验用户状态
if userInfo.Status != 1 { // 非正常状态
    return nil, ret.BizErr(ctx, errs.UserStatusAbnormal)
}
```

---

### 3.3 接口2：BatchGetUserInfo — 批量获取用户信息

```
调用方：ser-wallet
被调用方：ser-user
接口状态：已存在，可直接调用
```

**接口签名**：
```thrift
user_rpc.BatchUserInfoResp BatchGetUserInfo(1: user_rpc.BatchGetUserInfoReq req)
```

**入参**：
| 字段 | 类型 | 说明 |
|------|------|------|
| userIds | list\<i64\> | 用户ID列表 |
| uids | list\<i64\> | 用户UID列表 |

**返回**：BatchUserInfoResp { list\<UserInfoExpendResp\> list, i64 total }

**调用场景**：

| 场景 | 说明 | 频率 |
|------|------|------|
| B端充值记录列表 | 列表中每行需展示用户昵称/UID，批量查询避免N+1 | 中频 |
| B端提现记录列表 | 同上 | 中频 |
| B端用户余额列表 | 如有批量展示用户余额的B端页面 | 低频 |

**设计要点**：
- 参考 ser-live 中 BatchGetLiveUser 的批量查询模式
- 单次建议上限100个用户ID，避免大量查询
- 结果按 userId 做map映射，方便O(1)查找

---

## 四、强依赖 — ser-kyc（KYC认证）

### 4.1 为什么是强依赖

需求文档明确规定：
- **未KYC认证用户**：仅支持USDT提现
- **已KYC认证用户**：支持所有提现渠道（银行卡/电子钱包/USDT）
- **提现账户持有人名称必须与KYC实名一致**（忽略大小写和多余空格比对）

这意味着提现流程中**必须**获取KYC认证状态和实名信息，否则无法完成业务校验。

### 4.2 为什么不能只用 ser-user 的 kycStatus 字段

ser-user 的 UserInfo 结构体中确实有 `kycStatus` 字段（i32），但它只提供状态码，**不包含 KYC 实名详情**（姓名、证件号等）。

提现时需要的信息对比：

| 需要的信息 | ser-user 能提供 | ser-kyc 能提供 |
|-----------|---------------|---------------|
| KYC状态（已认证/未认证） | kycStatus 字段 | status 字段 |
| 实名姓名（Name） | 不能 | **name 字段** |
| 国籍 | countryCode（不同含义） | **country 字段** |
| 证件类型 | 不能 | **idType 字段** |
| 证件号码 | 不能 | **idNumber 字段** |

**结论**：
- **快速校验**（判断是否已认证）→ 可以用 ser-user 的 kycStatus，避免多一次RPC
- **详细校验**（提现账户持有人名比对）→ **必须**调用 ser-kyc 获取实名信息

### 4.3 接口3：KycQuery — 查询用户KYC认证信息

```
调用方：ser-wallet
被调用方：ser-kyc
ETCD服务名：kyc_service
IDL位置：/Users/mac/gitlab/common/idl/ser-kyc/service.thrift
接口状态：已存在，可直接调用
```

**接口签名**：
```thrift
user_kyc.KycQueryResp KycQuery(1: user_kyc.KycQueryReq req)
```

**入参**：
| 字段 | 类型 | 说明 |
|------|------|------|
| uid | i64（optional） | 用户UID |

**返回（KycQueryResp → UserKycInfo）**：
| 字段 | 类型 | 钱包使用场景 |
|------|------|-------------|
| uid | i64 | 关联用户 |
| name | string | **提现账户持有人名比对** |
| country | string | 判断可用的提现方式 |
| phone | string | 安全验证 |
| idType | i32 | 证件类型（1身份证 2护照 3驾照） |
| idNumber | string | 安全审计 |
| status | i32 | **1=去认证 2=审核中 3=已通过 4=重新认证** |
| rejectReason | string | 展示给用户为何认证失败 |

**在钱包业务中的调用场景**：

| 场景 | 触发时机 | 使用字段 | 频率 |
|------|---------|---------|------|
| 提现 — 渠道校验 | 用户进入提现页面 | status（3=已通过才开放全渠道） | 高频 |
| 提现 — 持卡人校验 | 用户首次填写提现账户信息时 | name（与填写的持卡人姓名比对） | 中频 |
| 提现 — 引导认证 | 未认证用户点击银行卡提现 | status（非3则引导去认证） | 中频 |

**代码调用示例**：
```go
// 提现前KYC校验
kycResp, err := rpc.KycClient().KycQuery(ctx, &ser_kyc.KycQueryReq{
    Uid: &userInfo.Uid,
})
if err != nil {
    lg.Log().CtxErrorf(ctx, "调用ser-kyc查询KYC状态失败: %v", err)
    return nil, ret.BizErr(ctx, errs.RPCCallError)
}

// 判断KYC状态
if kycResp.KycInfo == nil || kycResp.KycInfo.Status != 3 {
    // 未通过KYC，仅允许USDT提现
    if req.WithdrawMethod != WithdrawMethodUSDT {
        return nil, ret.BizErr(ctx, errs.KycRequiredForBankWithdraw)
    }
}

// 持卡人姓名比对（已KYC的场景）
if kycResp.KycInfo != nil && kycResp.KycInfo.Status == 3 {
    kycName := strings.TrimSpace(strings.ToLower(kycResp.KycInfo.Name))
    holderName := strings.TrimSpace(strings.ToLower(req.AccountHolderName))
    if kycName != holderName {
        return nil, ret.BizErr(ctx, errs.AccountHolderNameMismatch)
    }
}
```

**重要的业务规则**：
- KYC status=3（已通过）→ 全渠道提现
- KYC status≠3 → 仅USDT提现（不需要实名）
- 持卡人名 = KYC实名（忽略大小写 + 去除首尾空格）
- 首次提现必须填写账户信息，后续复用不可自行修改

---

## 五、标配依赖 — ser-blog（操作日志）

### 5.1 为什么是标配依赖

工程中**所有B端模块**都调用 ser-blog 记录操作日志。这是工程标准规范，不是可选的。

代码实证——所有已有模块都在调用：
- ser-app → `rpc.BLogClient().AddActLog(...)` — Banner/分类管理日志
- ser-buser → `rpc.BLogClient().AddActLog(...)` — 用户/角色/菜单管理日志
- ser-bcom → `rpc.BLogClient().AddActLog(...)` — 黑白名单/配置管理日志
- ser-i18n → `rpc.BLogClient().AddActLog(...)` — 多语言配置日志

ser-wallet 作为包含B端管理页面（币种配置、汇率管理等）的模块，**必须遵循此规范**。

### 5.2 接口4：AddActLog — 异步记录操作日志

```
调用方：ser-wallet
被调用方：ser-blog
ETCD服务名：blog_service
IDL位置：/Users/mac/gitlab/common/idl/ser-blog/service.thrift
接口状态：已存在，可直接调用
调用方式：oneway void（异步单向，不等待响应，不阻塞主流程）
```

**接口签名**：
```thrift
oneway void AddActLog(1: model.ActAddReq req)
```

**入参（ActAddReq）**：
| 字段 | 类型 | 说明 | ser-wallet填写值 |
|------|------|------|-----------------|
| backend | string | 所属后台 | "wallet" |
| model | string | 操作模块 | "currency_config" / "wallet_manage" |
| page | string | 操作页面 | "币种配置" / "汇率管理" |
| action | string | 操作行为描述 | "新增 币种(VND)" / "修改 汇率浮动比例" |
| isSuccess | i32 | 操作状态 | 1=成功 / 0=失败 |

> 注意：IP、操作人ID、操作人名称等字段由 ser-blog 从 Context 中自动提取，无需传入

**在钱包业务中的调用场景**：

| B端操作 | action描述示例 | 频率 |
|---------|-------------|------|
| 新增币种 | "新增 币种(VND/越南盾)" | 低频 |
| 编辑币种 | "编辑 币种(VND) 小数位数: 0→2" | 低频 |
| 启用/禁用币种 | "禁用 币种(THB/泰国铢)" | 低频 |
| 上传币种图标 | "上传 币种图标(VND) 文件名: vnd.svg" | 低频 |
| 手动刷新汇率 | "手动刷新 汇率(VND) 实时汇率: 25312.50" | 低频 |
| 修改汇率浮动比例 | "修改 汇率浮动比例(VND) 入款: 0.5%→0.8%" | 低频 |

**代码调用示例**（参考 ser-app/banner.go 的标准模式）：
```go
// ser-wallet 定义日志常量
const (
    LogBackend       = "wallet"
    LogModelCurrency = "currency_config"
    LogModelWallet   = "wallet_manage"
    LogPageCurrency  = "币种配置"
    LogPageRate      = "汇率管理"
)

// 币种配置操作后记录日志
func addCurrencyLog(ctx context.Context, action string, status int32) error {
    return rpc.BLogClient().AddActLog(ctx, &ser_blog.ActAddReq{
        Backend:   LogBackend,
        Model:     LogModelCurrency,
        Page:      LogPageCurrency,
        Action:    action,
        IsSuccess: status,
    })
}

// 使用示例
action := fmt.Sprintf("新增 币种(%s/%s)", req.CurrencyCode, req.CurrencyName)
_ = addCurrencyLog(ctx, action, enum.HandleSuccess)
```

**关键设计要点**：
- **oneway 不阻塞**：日志记录失败不影响主业务流程
- **即使失败也只打 warn 日志**：不能因为日志服务不可用导致业务操作失败
- **action 字段要有业务含义**：便于后续审计追溯

---

### 5.3 接口5：QueryActs — B端操作日志查询

```
调用方：ser-wallet（B端币种配置页面的"操作日志"Tab）
被调用方：ser-blog
接口状态：已存在，可直接调用
```

**接口签名**：
```thrift
model.ActQueryResp QueryActs(1: model.ActQueryReq req)
```

**入参（ActQueryReq）**：
| 字段 | 类型 | 说明 |
|------|------|------|
| backend | string | "wallet" |
| model | string | "currency_config" |
| page | string | 可选筛选 |
| startTime | i64 | 开始时间戳 |
| endTime | i64 | 结束时间戳 |
| pageNo | i64 | 页码 |
| pageSize | i64 | 每页条数 |
| createBy | i64 | 操作人ID（可选筛选） |
| createName | string | 操作人名称（可选筛选） |

**返回（ActQueryResp）**：
| 字段 | 类型 | 说明 |
|------|------|------|
| data | list\<Act\> | 日志列表 |
| total | i64 | 总条数 |

Act 结构体关键字段：id, backend, model, page, action, ip, ipCountry, createAt, isSuccess, createBy, createName

**调用场景**：
- B端币种配置页面的"操作日志"Tab（需求原型中明确有此Tab）
- 筛选条件：按时间范围、操作人、操作类型

**依据**：币种配置产品原型中明确有"操作日志"Tab页，展示币种配置相关的所有操作记录

---

## 六、弱依赖 — ser-s3（文件存储）

### 6.1 接口6：UploadFile — 上传币种图标

```
调用方：ser-wallet（B端币种配置 — 上传币种图标）
被调用方：ser-s3
ETCD服务名：s3_service
接口状态：已存在，可直接调用
```

**接口签名**：
```thrift
model.UploadFileResp UploadFile(1: model.UploadFileReq reqData)
```

**入参（UploadFileReq）**：
| 字段 | 类型 | 说明 | ser-wallet填写值 |
|------|------|------|-----------------|
| bsType | i32 | 业务类型编号 | 待定义（如 30=币种图标） |
| ownerId | i64 | 所属ID | 币种记录ID |
| fileName | string | 文件名 | "vnd.svg" |
| content | binary | 文件二进制内容 | SVG/PNG文件数据 |
| duration | i64 | 有效期（0=永久） | 0 |

**返回（UploadFileResp）**：
| 字段 | 类型 | 说明 |
|------|------|------|
| s3Key | string | S3存储路径 |
| url | string | 文件访问URL |

**调用场景**：
| 场景 | 说明 | 频率 |
|------|------|------|
| B端新增币种 | 上传币种图标（SVG/PNG，≤5KB） | 极低频 |
| B端编辑币种 | 更换币种图标 | 极低频 |

**需求依据**：币种配置需求文档——后台配置页编辑弹窗中可上传币种图标，格式 SVG/PNG，≤5KB

**代码调用示例**（参考 ser-app 的上传模式）：
```go
resp, err := rpc.S3Client().UploadFile(ctx, &ser_s3.UploadFileReq{
    BsType:   enum.CurrencyIconBsType, // 30
    OwnerId:  currencyId,
    FileName: req.FileName,
    Content:  req.Content,
    Duration: 0,
})
if err != nil || resp.S3Key == "" {
    lg.Log().CtxErrorf(ctx, "上传币种图标失败: %v", err)
    return nil, ret.BizErr(ctx, errs.UploadFileErr)
}
// 保存 s3Key 到币种配置表的 icon_s3_key 字段
```

---

## 七、弱依赖 — ser-buser（B端用户）

### 7.1 接口7：QueryUserByIds — 批量查询B端操作人名称

```
调用方：ser-wallet（B端页面展示操作人名称）
被调用方：ser-buser
ETCD服务名：b_user_service
接口状态：已存在，可直接调用
```

**接口签名**：
```thrift
map<i64, string> QueryUserByIds(1: sys_user.RpcSysUserReq req)
```

**入参**：
| 字段 | 类型 | 说明 |
|------|------|------|
| ids | list\<i64\> | B端用户ID列表 |

**返回**：map\<i64, string\>（userId → userName 的映射）

**调用场景**：
| 场景 | 说明 | 频率 |
|------|------|------|
| B端币种配置列表 | 展示"最后修改人"列 | 低频 |
| B端操作日志列表 | 如需展示操作人名称（ser-blog 已返回 createName，一般不需要额外调用） | 极低频 |

**代码调用示例**（参考 ser-bcom 的标准模式）：
```go
// 获取操作人名称
userMap, err := rpc.BUserClient().QueryUserByIds(ctx, &ser_buser.RpcSysUserReq{
    Ids: operatorIds,
})
if err != nil {
    klog.CtxErrorf(ctx, "查询B端用户名称失败: %v", err)
    // 降级处理：不展示操作人名称，但不阻塞主流程
}
```

**设计要点**：
- 这是**展示增强**依赖，不是核心业务依赖
- 调用失败采用降级处理（不展示名称），不影响主流程

---

## 八、不需要的依赖（明确排除）

以下模块经过分析，ser-wallet **当前不需要**调用：

| 模块 | 原因 |
|------|------|
| **ser-i18n（多语言）** | 钱包错误码的翻译由网关层（gate-back/gate-font）统一处理，通过 `rpc.Handler` 自动调用 ser-i18n 翻译。ser-wallet 内部不需要直接调用。参考：ser-app、ser-buser 等模块内部也没有直接调用 ser-i18n（仅 ser-bcom 在特定枚举翻译场景调用） |
| **ser-bcom（通用配置）** | 钱包的配置（币种、汇率等）自己管理，不走 ser-bcom 的全局开关。除非有"全局维护模式开关"需求，目前需求文档未提及 |
| **ser-exp（VIP等级）** | 当前需求中钱包提现限额由后台独立配置，不与VIP等级绑定。如未来需要VIP等级影响提现限额，再引入 |
| **ser-ip（IP管理）** | IP黑白名单校验由网关层统一处理，ser-wallet 内部不需要重复校验 |
| **ser-live（直播）** | 钱包不需要直播信息，反而是直播需要调用钱包（打赏扣款/收益入账） |
| **ser-item（道具）** | 钱包与道具无直接业务关联 |

---

## 九、尚不存在但业务上需要的依赖

### 9.1 ser-finance（财务管理）— 他人负责开发

钱包的C端充值/提现流程中，存在以下需要从财务模块获取的信息：

| 需要的能力 | 说明 | 当前状态 |
|-----------|------|---------|
| 获取充值方式列表 | 用户充值时展示可用的充值方式（银行转账/电子钱包/USDT等） | **财务模块未开发，接口不存在** |
| 获取提现方式列表 | 用户提现时展示可用的提现方式 | **财务模块未开发** |
| 获取提现限额与费率 | 展示单笔限额、日限额、手续费率 | **财务模块未开发** |
| 提交提现订单到财务 | 用户确认提现后，订单进入财务审核流程 | **财务模块未开发** |

**当前处理策略**：
- 这些是**缺火级别**的依赖——没有这些接口，充值/提现的C端流程无法完整跑通
- 但这些接口由**财务模块团队负责定义和开发**
- 我方在编码时使用 **TODO/Mock 占位**，标注清楚"待财务模块接口就绪后对接"
- 先把钱包自身的核心能力（余额管理、币种配置）做好，充值/提现的财务对接环节后续联调时补齐

```go
// TODO: 待 ser-finance 接口就绪后替换
// 获取充值方式列表 — 当前Mock返回固定数据
func getRechargeMethodList(ctx context.Context, currencyCode string) ([]*RechargeMethod, error) {
    // TODO: rpc.FinanceClient().GetRechargeMethodList(ctx, &ser_finance.GetRechargeMethodListReq{...})
    return mockRechargeMethodList(currencyCode), nil
}
```

### 9.2 依赖分类：缺调料 vs 缺火

| 依赖 | 分类 | 理由 |
|------|------|------|
| ser-user (GetUserInfoExpend) | **缺调料** | 初期可用 mock 用户信息，不阻塞钱包核心逻辑开发 |
| ser-kyc (KycQuery) | **缺调料** | 初期可跳过KYC校验，后续完善 |
| ser-blog (AddActLog) | **缺调料** | 日志记录可后补，不影响核心业务 |
| ser-s3 (UploadFile) | **缺调料** | 币种图标可先用固定URL占位 |
| ser-buser (QueryUserByIds) | **缺调料** | 操作人名称展示可后补 |
| ser-finance (充值方式/提现方式/限额) | **缺火** | 充值/提现的C端完整流程依赖此，但财务模块未开发，需Mock占位并等待联调 |

---

## 十、按业务场景的RPC调用清单

### 10.1 C端充值流程

```
用户进入充值页面
  │
  ├─[内部] 查询用户钱包各币种余额
  ├─[内部] 查询启用的币种列表（币种配置，内部函数）
  ├─[内部] 查询各币种汇率（币种配置，内部函数）
  ├─[TODO] 查询充值方式列表 → ser-finance（待开发）
  │
  用户选择币种、方式、金额、奖金方案
  │
  ├─[内部] 汇率计算、金额校验
  ├─[内部] 创建充值订单
  │
  三方支付回调成功
  │
  ├─[被调用] 财务调用 ser-wallet.CreditWallet → 上账
  │
  完成

外部RPC调用次数：0（充值流程中钱包不主动调外部）
被调用次数：1（财务调用CreditWallet入账）
```

### 10.2 C端提现流程

```
用户进入提现页面
  │
  ├─[外部] ser-user.GetUserInfoExpend → 校验用户状态
  ├─[外部] ser-kyc.KycQuery → 获取KYC状态和实名信息
  ├─[内部] 查询用户余额
  ├─[内部] 查询提现汇率（币种配置，内部函数）
  ├─[TODO] 查询提现方式和限额 → ser-finance（待开发）
  │
  用户填写提现信息、确认
  │
  ├─ KYC姓名比对（kycResp.name vs 持卡人姓名）
  ├─[内部] FreezeBalance 冻结金额
  ├─[内部] 创建提现订单
  ├─[TODO] 提交订单到财务审核 → ser-finance（待开发）
  ├─[外部] ser-blog.AddActLog → 记录提现申请日志
  │
  完成（等待财务审核 → 解冻/确认扣除）

外部RPC调用次数：3（ser-user + ser-kyc + ser-blog）
```

### 10.3 C端兑换流程

```
用户进入兑换页面
  │
  ├─[内部] 查询平台汇率（币种配置，内部函数）
  ├─[内部] 查询基准币种和BSB锚定比例（内部函数）
  │
  用户输入金额、确认
  │
  ├─[内部] 扣法币、加BSB、加赠送（同一事务）
  │
  完成

外部RPC调用次数：0（兑换是纯内部操作）
```

### 10.4 B端币种配置管理

```
进入币种配置页面
  │
  ├─[内部] 查询币种列表

操作：新增/编辑币种
  │
  ├─[外部] ser-s3.UploadFile → 上传币种图标（如有）
  ├─[外部] ser-blog.AddActLog → 记录操作日志
  ├─[外部] ser-buser.QueryUserByIds → 列表展示操作人名称

查看操作日志Tab
  │
  ├─[外部] ser-blog.QueryActs → 查询操作日志列表

外部RPC调用次数：最多4（ser-s3 + ser-blog×2 + ser-buser）
```

### 10.5 被其他模块调用的RPC（暴露方向，参考对照）

```
ser-finance → ser-wallet.CreditWallet    （充值入账）
ser-finance → ser-wallet.FreezeBalance   （提现冻结）
ser-finance → ser-wallet.UnfreezeBalance （提现驳回）
ser-finance → ser-wallet.ConfirmDebit    （提现确认）
ser-finance → ser-wallet.DebitWallet     （人工减款）
ser-finance → ser-wallet.GetBalance      （查余额）
ser-user    → ser-wallet.InitWallet      （注册初始化）
```

---

## 十一、与现有工程RPC调用规范的对照

### 11.1 调用规范检查清单

基于对 ser-app、ser-bcom、ser-buser、ser-i18n 四个模块的实际代码审查，ser-wallet 的 RPC 调用需遵循：

| 规范项 | 工程标准（代码实证） | ser-wallet 应用 |
|--------|-------------------|----------------|
| 客户端获取 | `rpc.XXClient()` 单例模式（sync.Once + lazy load） | 遵循，在 common/rpc/rpc_client.go 中注册 |
| 超时设置 | 全局 3秒 RPC 超时 | 遵循默认值；如有特殊需要可单独设置 |
| 服务发现 | ETCD 注册中心 + 自动负载均衡 | 遵循 |
| 错误处理 | `kerrors.FromBizStatusError(err)` 区分业务/网络错误 | 遵循 |
| 降级处理 | 非核心调用失败不阻塞主流程（如翻译失败用原值） | ser-blog、ser-buser 调用失败采用降级 |
| oneway 调用 | 日志类操作使用 oneway void（异步不等响应） | AddActLog 使用 oneway |
| Context 传递 | 所有RPC调用必须传入 ctx（含TraceID、UserID等） | 遵循 |
| 日志记录 | RPC调用失败必须打 Error 日志 | `lg.Log().CtxErrorf(ctx, "...")` |

### 11.2 ser-wallet 需要在 common/rpc/rpc_client.go 中新增的客户端

ser-wallet 需要调用的5个外部服务中，以下客户端**已注册**（可直接使用）：

| 客户端 | 变量名 | 状态 |
|--------|--------|------|
| rpc.UserClient() | userService | 已注册（编号008） |
| rpc.KycClient() | kycService | 已注册（编号012） |
| rpc.BLogClient() | bLogService | 已注册（编号006） |
| rpc.S3Client() | s3Service | 已注册（编号007） |
| rpc.BUserClient() | bUserService | 已注册（编号002） |

**结论**：5个都已注册，ser-wallet **无需在 rpc_client.go 中新增任何客户端**，直接引用即可。

---

## 十二、总结

### 12.1 数量统计

| 分类 | 数量 | 模块 |
|------|------|------|
| 强依赖（核心业务必需） | 3个接口 | ser-user(2) + ser-kyc(1) |
| 标配依赖（工程规范要求） | 2个接口 | ser-blog(2) |
| 弱依赖（展示增强） | 2个接口 | ser-s3(1) + ser-buser(1) |
| **合计** | **7个接口** | **5个外部模块** |

### 12.2 按优先级排序

| 优先级 | 接口 | 模块 | 理由 |
|--------|------|------|------|
| P0 | KycQuery | ser-kyc | 提现流程硬依赖：无KYC校验则提现逻辑不完整 |
| P0 | GetUserInfoExpend | ser-user | 多处需要用户状态校验 |
| P0 | AddActLog | ser-blog | 工程标配：所有B端操作必须记审计日志 |
| P1 | BatchGetUserInfo | ser-user | B端列表展示需要 |
| P1 | QueryActs | ser-blog | B端操作日志Tab |
| P2 | UploadFile | ser-s3 | 币种图标上传（极低频） |
| P2 | QueryUserByIds | ser-buser | 操作人名称展示（可降级） |

### 12.3 对比：暴露 vs 调用

| 维度 | 暴露给别人（rpc暴露/tmp-1.md） | 调用别人（本文档） |
|------|------|------|
| 接口数量 | 12个 | 7个 |
| 核心角色 | **被依赖方**（资金操作提供方） | **依赖方**（用户/认证/日志消费方） |
| 复杂度 | 高（涉及幂等、事务、并发、状态机） | 低（大部分是简单查询） |
| 不可用影响 | 严重（其他模块充值/提现全停） | 可降级（除KYC外，其余可降级处理） |

**核心结论**：ser-wallet 在工程中的角色是**重度被依赖、轻度依赖他人**。它暴露12个RPC接口供5+个模块调用，但自身仅依赖7个外部接口，且大部分可降级处理。唯一的硬依赖是 ser-kyc 的 KYC 校验（提现流程）和 ser-user 的用户状态校验。
