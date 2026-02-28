# ser-wallet RPC 调用依赖分析（我们调用别人的接口）

> 分析时间：2026-02-28
> 分析范围：ser-wallet 模块（含多币种配置）需要**主动调用**的所有外部依赖
> 分析依据：需求文档（6文件夹 ~155张图片）+ 产品原型 + 接口评估 + 依赖关系 + 链路关系 + IDL设计 + 模块参考A/B + 工程 IDL 源文件验证 + 现有模块调用模式实证
> 分析角色：后端开发，主责钱包+币种配置模块
> 核心问题：**ser-wallet 需要调用哪些外部服务的什么方法？在什么业务场景下调？参数和返回值是什么？哪些现在就能接、哪些需要 Mock？**
> 姊妹篇：本文与《RPC 对外暴露接口分析》互为镜像——那篇分析"别人调我"，本篇分析"我调别人"

---

## 零、结论速览

```
ser-wallet 需要主动调用的外部依赖：

┌──────────────────────────────────────────────────────────────────────────────────────┐
│  依赖类型       │ 目标服务           │ 接口数 │ 状态         │ 开发阶段定位          │
├──────────────────────────────────────────────────────────────────────────────────────┤
│  RPC（已有IDL） │ ser-kyc            │   1    │ IDL已定义    │ 可直接接入            │
│  RPC（已有IDL） │ ser-user           │   2    │ IDL已定义    │ 可直接接入            │
│  RPC（已有IDL） │ ser-blog           │   1    │ IDL已定义    │ 可直接接入（oneway）  │
│  RPC（已有IDL） │ ser-s3             │  1-2   │ IDL已定义    │ 可直接接入            │
│  RPC（已有IDL） │ ser-buser          │   1    │ IDL已定义    │ 可直接接入            │
│  RPC（已有IDL） │ ser-bcom           │  0-1   │ IDL已定义    │ 按需评估              │
│  RPC（待定IDL） │ 财务模块(ser-pay?) │  2-3   │ 服务名未定   │ 需Mock占位            │
│  RPC（待定IDL） │ 活动模块(ser-act?) │  1-2   │ 服务名未定   │ 需Mock占位            │
│  RPC（空壳IDL） │ ser-cron           │  0-1   │ service为空  │ 需确认接入方式        │
│  非RPC          │ ser-i18n           │   0    │ Redis直读    │ 框架层已集成          │
│  外部HTTP       │ 三方汇率API        │   3    │ 三家取平均   │ 需实现HTTP Client     │
│  RPC（可选）    │ 风控模块(待定)     │  0-1   │ 服务名未定   │ 当前版本可不接        │
├──────────────────────────────────────────────────────────────────────────────────────┤
│  合计           │                    │ 12-18  │              │                       │
│  其中可直接接入 │                    │  6-8   │              │ IDL已有，Client已注册 │
│  其中需Mock占位 │                    │  3-5   │              │ 服务名/IDL未定        │
│  其中需确认     │                    │  1-2   │              │ 接入方式待定          │
│  外部HTTP       │                    │   3    │              │ 需独立HTTP Client     │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 一、为什么需要单独分析"我调别人"

### 1.1 与"别人调我"的本质区别

"别人调我"（RPC暴露）——我们是**被动方**，设计好接口等别人来调，主动权在我们。

"我调别人"（RPC依赖）——我们是**主动方**，业务流程中必须先拿到外部数据才能继续，**受制于对方的接口定义、可用性、响应速度**。

**核心差异：**

| 维度 | 别人调我 | 我调别人 |
|------|---------|---------|
| 主动权 | 在我方，我定义接口契约 | 在对方，我适配对方接口 |
| 阻塞性 | 不阻塞我的开发 | **可能阻塞**我的开发（对方接口没好） |
| 容错 | 对方处理我的错误 | **我必须处理**对方超时/不可用/返回异常 |
| 开发策略 | 我优先实现，对方后接入 | 对方没好→我先Mock，对方好了→替换Mock |

### 1.2 分析目标

1. **完整枚举**：列出所有需要调用的外部接口，不遗漏
2. **IDL验证**：核实每个接口在工程 IDL 中是否已定义、方法签名是否匹配
3. **模式参考**：从现有成熟模块（ser-app、ser-item）提取调用范式，避免自己摸索
4. **阻塞分类**：区分"可直接接入"和"需Mock占位"，指导编码节奏
5. **与暴露篇对齐**：确保依赖图中 wallet 的进出完整闭环

---

## 二、工程基础设施——RPC Client 注册机制

### 2.1 统一入口：rpc_client.go

路径：`/Users/mac/gitlab/common/rpc/rpc_client.go`

工程中所有跨服务 RPC 调用通过该文件的单例工厂函数获取 Client：

```go
var xxxClient serXxx.Client
var xxxOnce sync.Once

func XxxClient() serXxx.Client {
    xxxOnce.Do(func() {
        xxxClient, _ = serXxx.NewClient(namesp.EtcdXxxService, ...)
    })
    return xxxClient
}
```

**当前已注册 22 个 Client（ser-wallet 无需全用）：**

| Client 函数 | 目标服务 | 钱包是否需要调用 |
|-------------|---------|-----------------|
| `KycClient()` | ser-kyc | **是** — 提现KYC校验 |
| `UserClient()` | ser-user | **是** — 用户信息/KYC姓名 |
| `BLogClient()` | ser-blog | **是** — B端操作审计日志 |
| `S3Client()` | ser-s3 | **是** — 币种图标上传 |
| `BUserClient()` | ser-buser | **是** — B端操作人姓名回显 |
| `BComClient()` | ser-bcom | **可选** — 功能开关查询 |
| `CronClient()` | ser-cron | **待定** — 定时任务注册（IDL为空） |
| `I18nClient()` | ser-i18n | **非RPC** — 通过 Redis 直读 |
| 其余 14 个 | ser-item/live/app 等 | 否 — 钱包不依赖 |

### 2.2 ETCD 服务注册：namesp.go

路径：`/Users/mac/gitlab/common/pkg/consts/namesp/namesp.go`

```go
EtcdKycService  = "kyc_service"
EtcdUserService = "user_service"
EtcdBLogService = "blog_service"
EtcdS3Service   = "s3_service"
EtcdBUserService = "buser_service"
EtcdBComService  = "bcom_service"
EtcdCronService  = "cron_service"
// ... 共 22 个服务
// 注意：尚无 wallet_service 和 finance_service
```

**关键发现：**
- ser-wallet 自身尚未注册（需要新增 `EtcdWalletService` + `WalletClient()`）
- 财务模块（ser-pay / ser-finance）尚未注册（服务名未定）
- 活动模块尚未注册

### 2.3 调用模式分两种

**模式A：Gate 层直传（不构造参数）**
```go
// gate-font/gate-back handler 中，框架自动绑定 HTTP 请求到 Thrift 结构
rpc.Handler(ctx, c, rpc.XxxClient().Method)
```
→ ser-wallet 的 C端/B端接口**不用这种模式调外部 RPC**，因为钱包自己就是服务实现方，在 service 层内部调用外部 RPC。

**模式B：Service 层手动构造（ser-wallet 的场景）**
```go
// 手动创建 Req 结构体，调用 Client 方法，处理 Resp 和 err
resp, err := rpc.XxxClient().Method(ctx, &serXxx.SomeReq{
    Field1: value1,
    Field2: value2,
})
if err != nil {
    // 处理错误：日志 + 返回业务错误码
}
```
→ ser-wallet **所有对外 RPC 调用都用这种模式**。

---

## 三、依赖 #1：ser-kyc — KYC 认证状态查询

### 3.1 接口信息

| 项目 | 内容 |
|------|------|
| **调用方法** | `rpc.KycClient().KycQuery(ctx, req)` |
| **IDL 文件** | `/Users/mac/gitlab/common/idl/ser-kyc/service.thrift` |
| **IDL 状态** | ✅ 已定义，可直接调用 |
| **Client 注册** | ✅ `rpc_client.go` 已有 `KycClient()` |
| **需求依据** | 需求7.1 KYC与提现关系 |
| **优先级** | 核心必须（D1） |

### 3.2 方法签名（IDL 验证）

```thrift
// service.thrift
user_kyc.KycQueryResp KycQuery(1: user_kyc.KycQueryReq req)

// user_kyc.thrift
struct KycQueryReq {
    1: optional i64 uid    // 用户 UID（可选，不传查当前登录用户）
}

struct KycQueryResp {
    1: optional UserKycInfo kycInfo
}

struct UserKycInfo {
    1: i64 id
    2: i64 uid
    3: string name           // ← 钱包需要：KYC 认证姓名
    4: string country        // ← 钱包可能需要：国家（影响提现规则）
    5: string phone
    6: i32 idType
    7: string idNumber
    8: string idFrontUrl
    9: string idBackUrl
    10: string idFrontS3Key
    11: string idBackS3Key
    12: i32 status           // ← 钱包核心需要：KYC 状态
    13: string rejectReason
    14: i32 submitCount
    15: i64 createAt
    16: i64 updateAt
}
```

### 3.3 钱包中的触发场景

```
场景1：C端用户获取提现方式列表（C9 GetWithdrawMethods）
  → 调 KycQuery(uid) 判断 KYC 状态
  → status != 3(已通过) → 返回"请先完成KYC认证"，前端跳转 KYC 页面
  → status == 3 → 正常返回提现方式列表

场景2：C端用户创建提现订单（C12 CreateWithdrawOrder）
  → 调 KycQuery(uid) 二次校验
  → status != 3 → 拒绝创建提现订单
  → status == 3 → kycInfo.name 用于校验提现账户持有人一致性

场景3：C端用户保存提现账户信息（C11 SaveWithdrawAccount）
  → 调 KycQuery(uid) 获取 kycInfo.name
  → 自动填充"账户持有人" = KYC 认证姓名（不可编辑）
```

### 3.4 调用要点

- **KYC 状态枚举**：1=去认证（未提交）、2=审核中、3=已通过、4=重新认证（被驳回后）
- **只有 status=3 才允许提现**，其余状态均前置拦截
- **uid 必传**：钱包 service 层调用时必须显式传入 uid（不能依赖 token 自动提取，因为跨服务 RPC 上下文不同于 HTTP）
- **容错**：KYC 服务不可用时，提现相关接口应返回明确错误码，**绝不可默认放行**

### 3.5 参考代码模板

```go
// internal/ser/withdraw_service.go
func checkKycStatus(ctx context.Context, uid int64) (*ser_kyc.UserKycInfo, error) {
    resp, err := rpc.KycClient().KycQuery(ctx, &ser_kyc.KycQueryReq{
        Uid: &uid,
    })
    if err != nil {
        lg.Log().CtxErrorf(ctx, "KycQuery failed uid=%d err=%v", uid, err)
        return nil, ret.BizErr(ctx, errs.KycServiceUnavailable, "KYC服务异常")
    }
    if resp.KycInfo == nil || resp.KycInfo.Status != 3 {
        return nil, ret.BizErr(ctx, errs.KycNotPassed, "请先完成KYC认证")
    }
    return resp.KycInfo, nil
}
```

---

## 四、依赖 #2：ser-user — 用户信息查询

### 4.1 接口信息（两个方法）

| 项目 | GetUserInfoExpend | BatchGetUserInfo |
|------|------------------|-----------------|
| **调用方法** | `rpc.UserClient().GetUserInfoExpend(ctx, req)` | `rpc.UserClient().BatchGetUserInfo(ctx, req)` |
| **IDL 文件** | `common/idl/ser-user/service.thrift` | 同左 |
| **IDL 状态** | ✅ 已定义 | ✅ 已定义 |
| **Client 注册** | ✅ `UserClient()` 已有 | ✅ 同左 |
| **需求依据** | D2 — 需求7.1第6点 | D11 — B端用户昵称展示 |
| **优先级** | 核心必须 | 应该有 |
| **现有调用** | ⚠️ 工程中暂无 service 层调用实例 | ⚠️ 同左 |

### 4.2 方法签名（IDL 验证）

```thrift
// service.thrift — RPC 跨服务方法段
user_rpc.UserInfoExpendResp GetUserInfoExpend(1: user_rpc.GetUserInfoReq req)
user_rpc.BatchUserInfoResp BatchGetUserInfo(1: user_rpc.BatchGetUserInfoReq req)

// user_rpc.thrift
struct GetUserInfoReq {
    1: i64 userId   // 用户主键 ID
    2: i64 uid      // 用户业务 UID（两者传其一即可）
}

struct BatchGetUserInfoReq {
    1: list<i64> userIds   // 按 userId 批量查
    2: list<i64> uids      // 按 uid 批量查
}

struct UserInfoExpendResp {
    1: i64 userId
    2: i64 uid
    3: string nickname      // ← B端列表展示
    4: string mobile
    5: string email
    6: i32 status           // ← 账号状态（封禁检测）
    7: i32 accountType      // ← 账号类型
    8: string avatar
    9: i64 exp
    10: i32 level
    11: i32 totalFans
    12: string countryCode  // ← 国家代码（可能影响币种/提现规则）
    13: i64 createAt
    14: i32 gender
}

struct BatchUserInfoResp {
    1: list<UserInfoExpendResp> list
    2: i64 total
}
```

### 4.3 钱包中的触发场景

```
场景1：C端保存提现账户（C11）
  → 调 GetUserInfoExpend(uid) 获取用户基本信息
  → 结合 KycQuery 拿到的 kycInfo.name，自动填充账户持有人
  → countryCode 可能用于判断该用户可用的提现方式

场景2：B端用户钱包列表（B6 QueryUserWalletPage）
  → 钱包表只存 uid，不存用户昵称
  → 分页查出钱包列表后，收集所有 uid
  → 调 BatchGetUserInfo(uids) 批量获取昵称
  → 组装返回：钱包余额 + 用户昵称

场景3：B端手动加/减款（B3/B4，可选）
  → 输入 uid 后调 GetUserInfoExpend(uid) 确认用户存在且未封禁
  → status 校验：如果用户已封禁，拒绝加/减款操作
```

### 4.4 调用要点

- **注意**：`GetUserInfoExpend` 和 `BatchGetUserInfo` 是工程中新增的 RPC 方法，**目前没有任何 service 层调用实例**可参考，ser-wallet 将是第一个在 service 层调用它们的模块
- **userId vs uid**：请求中两者传其一即可。钱包内部统一用 uid 还是 userId，需与钱包表设计保持一致
- **批量查询性能**：B端分页列表场景一次最多查 20-50 条（pageSize 限制），BatchGetUserInfo 的 list 长度可控
- **用户不存在**：对方返回空 resp 不报错，钱包需检查 resp.list 长度 / resp 是否为 nil

### 4.5 参考代码模板

```go
// internal/ser/user_helper.go

// 单个用户查询（C端场景）
func getUserInfo(ctx context.Context, uid int64) (*ser_user.UserInfoExpendResp, error) {
    resp, err := rpc.UserClient().GetUserInfoExpend(ctx, &ser_user.GetUserInfoReq{
        Uid: uid,
    })
    if err != nil {
        lg.Log().CtxErrorf(ctx, "GetUserInfoExpend failed uid=%d err=%v", uid, err)
        return nil, ret.BizErr(ctx, errs.UserServiceErr, "用户服务异常")
    }
    return resp, nil
}

// 批量用户查询（B端列表场景），返回 map[uid]nickname
func batchGetUserNicknames(ctx context.Context, uids []int64) map[int64]string {
    if len(uids) == 0 {
        return map[int64]string{}
    }
    resp, err := rpc.UserClient().BatchGetUserInfo(ctx, &ser_user.BatchGetUserInfoReq{
        Uids: uids,
    })
    if err != nil {
        lg.Log().CtxErrorf(ctx, "BatchGetUserInfo failed err=%v", err)
        return map[int64]string{}  // 降级：昵称为空，不阻塞主流程
    }
    result := make(map[int64]string, len(resp.List))
    for _, u := range resp.List {
        result[u.Uid] = u.Nickname
    }
    return result
}
```

---

## 五、依赖 #3：ser-blog — B端操作审计日志

### 5.1 接口信息

| 项目 | 内容 |
|------|------|
| **调用方法** | `rpc.BLogClient().AddActLog(ctx, req)` |
| **IDL 文件** | `common/idl/ser-blog/service.thrift` |
| **IDL 状态** | ✅ 已定义 |
| **Client 注册** | ✅ `BLogClient()` 已有 |
| **方法特性** | `oneway void` — 客户端发出即返回，不等待响应 |
| **需求依据** | D3 — 工程规范，所有 B 端写操作必须审计留痕 |
| **优先级** | 核心必须 |
| **现有调用** | ✅ ser-app、ser-item、ser-buser、ser-bcom、ser-s3、ser-i18n 均已调用 |

### 5.2 方法签名（IDL 验证）

```thrift
// service.thrift
oneway void AddActLog(1: model.ActAddReq req)

// model.thrift
struct ActAddReq {
    1: string backend    // 所属后台（如 "运营后台"）
    2: string model      // 操作模块（如 "钱包管理"）
    3: string page       // 操作页面（如 "币种配置"）
    4: string action     // 操作行为描述（动态拼接，含关键参数）
    5: i32 isSuccess     // 1=成功 0=失败
}
```

### 5.3 钱包中的触发场景

```
【币种配置模块 — B端】
  B2 编辑币种 → 记录 "编辑 币种配置 BRL(巴西雷亚尔) 修改浮动比例 3%→5%"
  B2 启用/禁用币种 → 记录 "启用 币种配置 USDT" / "禁用 币种配置 BRL"
  B2 设置基准币种 → 记录 "设置基准币种 BRL→USD"
  B2 上传币种图标 → 记录 "上传 币种图标 BRL.svg"

【钱包管理模块 — B端（如有）】
  B3 人工加款审核 → 记录 "人工加款 uid=123456 BRL 1000.00 审核通过"
  B4 人工减款审核 → 记录 "人工减款 uid=123456 BRL 500.00 审核通过"
  B5 提现审核 → 记录 "提现审核 订单号=WD20260228001 BRL 2000.00 审核通过"
```

### 5.4 调用要点

- **oneway void**：ser-blog 的 AddActLog 是异步火忆（fire-and-forget），不影响主流程响应时间
- **即使 err != nil 也不应中断主业务**：日志记录失败只打 Error log，不向用户返回错误
- **IP、操作人信息**：由 ser-blog 从 RPC 上下文 metadata（token）中自动提取，钱包无需传
- **action 字段**：动态拼接业务关键参数，越详细越利于审计追溯

### 5.5 参考代码模板（来自 ser-app 的成熟模式）

```go
// internal/pkg/enum/log_enum.go — 常量定义
const (
    LogBackend      = "运营后台"
    LogModel        = "钱包管理"
    LogCurrencyPage = "币种配置"
    LogWalletPage   = "钱包管理"
    LogWithdrawPage = "提现审核"
    LogExchangePage = "汇率管理"

    ActionAdd    = "新增"
    ActionEdit   = "编辑"
    ActionEnable = "启用"
    ActionDisable = "禁用"
    ActionUpload = "上传"
    ActionApprove = "审核通过"
    ActionReject  = "审核驳回"
)

// internal/ser/log_helper.go — 统一封装
func addWalletLog(ctx context.Context, page, action string, status int32) {
    err := rpc.BLogClient().AddActLog(ctx, &ser_blog.ActAddReq{
        Backend:   enum.LogBackend,
        Model:     enum.LogModel,
        Page:      page,
        Action:    action,
        IsSuccess: status,
    })
    if err != nil {
        lg.Log().CtxErrorf(ctx, "addWalletLog failed page=%s action=%s err=%v", page, action, err)
        // 不 return error — 审计日志失败不中断主业务
    }
}

// 调用方示例
action := fmt.Sprintf("%s 币种配置 %s(%s) 浮动比例 %s→%s",
    enum.ActionEdit, req.CurrencyCode, req.CurrencyName, oldRate, newRate)
addWalletLog(ctx, enum.LogCurrencyPage, action, int32(enum.HandleSuccess))
```

---

## 六、依赖 #4：ser-s3 — 文件上传/图标上传

### 6.1 接口信息

| 项目 | 内容 |
|------|------|
| **调用方法** | `rpc.S3Client().UploadFile(ctx, req)` 或 `UploadIcon(ctx, req)` |
| **IDL 文件** | `common/idl/ser-s3/service.thrift` |
| **IDL 状态** | ✅ 已定义（注意：没有 GetPresignUrl，实际方法名是 `GetUploadPreSign`） |
| **Client 注册** | ✅ `S3Client()` 已有 |
| **需求依据** | D7 — 币种配置3.3，B端编辑币种时上传图标 |
| **优先级** | 应该有 |
| **现有调用** | ✅ ser-app（UploadFile + DeleteFile）、ser-item（UploadIcon） |

### 6.2 方法签名（IDL 验证）

```thrift
// 方式一：小文件直传（二进制流经后端，适合低频小图标）
model.UploadFileResp UploadFile(1: model.UploadFileReq reqData)

struct UploadFileReq {
    1: i32 bsType       // 业务类型（对应 s3_config 表中的校验规则）
    2: i64 ownerId      // 文件所有者 ID
    3: string fileName   // 文件名
    4: binary content    // 文件二进制内容
    6: i32 duration      // 有效期秒数（0=永久）
}
struct UploadFileResp {
    1: string fileUrl    // 完整访问 URL
    2: string s3Key      // S3 相对路径（存库用）
}

// 方式二：图标专用上传（ser-item 使用的方式）
string uploadIcon(1: model.UploadIconReq reqData)

struct UploadIconReq {
    1: i32 bsType
    2: binary content
}
// 返回 string 即图标 URL

// 方式三：预签名上传（大文件/高频场景，前端直传 S3）
model.GetUploadPreSignResp GetUploadPreSign(1: model.GetUploadPreSignReq reqData)

struct GetUploadPreSignReq {
    1: i32 bsType
    2: i64 ownerId
    3: string fileName
    4: i64 fileSize
    5: i32 duration
}
struct GetUploadPreSignResp {
    1: string uploadUrl     // 预签名上传 URL（前端 PUT）
    2: string contentType
    3: string s3Key
}

// 删除文件
model.IsSuccessResp DeleteFile(1: model.DeleteFileReq reqData)

struct DeleteFileReq {
    1: string s3Key   // 通过 mys3.ToS3Key(url) 转换
}
```

### 6.3 钱包中的触发场景

```
场景1：B端编辑币种上传图标（B2 UpdateCurrency）
  → B端上传 SVG/PNG 图标文件
  → 调 S3Client().UploadIcon(ctx, req) 或 UploadFile(ctx, req)
  → 返回 fileUrl / s3Key
  → 保存到币种配置表的 icon_url / icon_s3_key 字段
  → 如果是替换旧图标 → 调 DeleteFile(ctx, oldS3Key) 清理旧文件

场景2（可选）：C端提现收款凭证/转账截图上传
  → 如需用户上传银行转账凭证截图
  → 建议用 GetUploadPreSign 预签名方式（前端直传 S3，流量不过后端）
  → 但当前需求中未明确此场景，标为可选
```

### 6.4 调用要点

- **⚠️ 方法名修正**：之前分析中写的 `GetPresignUrl` 实际不存在，正确方法名是 `GetUploadPreSign`
- **bsType**：需要在 ser-s3 的 `s3_config` 配置表中为钱包注册一个业务类型（如 `bsType=10` 对应 wallet_icon），控制文件类型白名单和大小限制
- **UploadIcon vs UploadFile**：币种图标推荐用 `UploadIcon`（参考 ser-item），更简洁；如需更多控制用 `UploadFile`
- **删除旧图标**：替换图标时需先删除旧的 S3 对象，避免存储泄漏

### 6.5 参考代码模板（来自 ser-item）

```go
// internal/ser/s3_service.go
func (s *S3Service) UploadCurrencyIcon(ctx context.Context, req *ser_wallet.UploadIconReq) (*ser_wallet.UploadIconResp, error) {
    url, err := rpc.S3Client().UploadIcon(ctx, &ser_s3.UploadIconReq{
        BsType:  enum.CurrencyIconBsType,  // 钱包自定义的 bsType
        Content: req.Content,
    })
    if err != nil {
        addWalletLog(ctx, enum.LogCurrencyPage,
            fmt.Sprintf("%s 币种图标（失败）", enum.ActionUpload), 0)
        return nil, ret.BizErr(ctx, errs.UploadIconErr, "图标上传失败")
    }
    addWalletLog(ctx, enum.LogCurrencyPage,
        fmt.Sprintf("%s 币种图标", enum.ActionUpload), 1)
    return &ser_wallet.UploadIconResp{Url: url}, nil
}
```

---

## 七、依赖 #5：ser-buser — B端操作人姓名回显

### 7.1 接口信息

| 项目 | 内容 |
|------|------|
| **调用方法** | `rpc.BUserClient().QueryUserByIds(ctx, req)` |
| **IDL 文件** | `common/idl/ser-buser/service.thrift` |
| **IDL 状态** | ✅ 已定义 |
| **Client 注册** | ✅ `BUserClient()` 已有 |
| **需求依据** | 工程规范 — B端列表中 create_by / update_by 字段需要显示操作人名称 |
| **优先级** | 应该有 |
| **现有调用** | ✅ ser-item（推荐模式）、ser-bcom、ser-i18n |

### 7.2 方法签名（IDL 验证）

```thrift
// service.thrift
map<i64, string> QueryUserByIds(1: sys_user.RpcSysUserReq req)

// sys_user.thrift
struct RpcSysUserReq {
    1: required list<i64> ids   // 后台用户 ID 列表
}

// 返回 map<i64, string>：{userId: "用户名"} 的映射
```

### 7.3 钱包中的触发场景

```
场景1：B端币种配置列表（B1 QueryCurrencyPage）
  → 分页查出币种列表，每条记录有 create_by / update_by（后台用户ID）
  → 收集所有 ID → 调 QueryUserByIds(ids)
  → 用返回的 map 将 ID 替换为用户名显示

场景2：B端汇率变更日志（B5 PageExchangeRateLog）
  → 手动汇率变更记录中有操作人 ID
  → 同上模式

场景3：B端用户钱包列表中的人工加/减款记录
  → 记录中有审核人 ID
  → 同上模式
```

### 7.4 调用要点

- **返回 `map[int64]string`**：直接用 ID 做 key，很方便
- **ID 不存在不报错**：对方返回的 map 中直接不包含该 key，钱包侧取值时判空即可
- **批量效率**：一次查所有 ID，避免逐条查（参考 ser-item 而非 ser-bcom 的模式）
- **降级策略**：查询失败时操作人名称显示为空或 ID，不阻塞列表返回

### 7.5 参考代码模板（来自 ser-item）

```go
// internal/domain/user_service.go
func getOperatorNames(ctx context.Context, userIds []int64) map[int64]string {
    if len(userIds) == 0 {
        return map[int64]string{}
    }
    userMap, err := rpc.BUserClient().QueryUserByIds(ctx, &ser_buser.RpcSysUserReq{
        Ids: userIds,
    })
    if err != nil {
        lg.Log().CtxErrorf(ctx, "QueryUserByIds failed err=%v", err)
        return map[int64]string{}  // 降级：名称为空
    }
    return userMap
}
```

---

## 八、依赖 #6：财务模块（服务名待定）— 充值/提现通道配置

### 8.1 接口信息

| 项目 | 内容 |
|------|------|
| **目标服务** | 财务模块（ser-pay / ser-finance，服务名待定） |
| **IDL 状态** | ❌ 未定义 — ETCD 中无注册、IDL 中无文件 |
| **Client 注册** | ❌ 无 — rpc_client.go 中无对应 Client |
| **需求依据** | D4/D5/D6 — 依赖关系：钱包→财务(支付配置) |
| **优先级** | 核心必须（最重要的外部依赖） |
| **开发策略** | **必须 Mock 占位**，待财务模块确定后替换 |

### 8.2 需要调用的方法（根据需求推导）

#### 方法 A：GetPaymentConfig / GetDepositMethods（获取充值方式配置）

```
触发场景：C3 获取充值方式列表
输入（推测）：
  - currencyCode: string    // 当前币种
  - type: i32               // 1=充值 2=提现
输出（推测）：
  - list<PaymentMethod>     // 可用支付方式列表
    - methodId: i64         // 方式ID
    - methodName: string    // 方式名称（如"银行转账""USDT""PIX"）
    - methodIcon: string    // 方式图标
    - tiers: list<Tier>     // 档位列表（快充金额选项）
    - minAmount: i64        // 最低充值金额
    - maxAmount: i64        // 最高充值金额
```

#### 方法 B：GetWithdrawMethods（获取提现方式配置）

```
触发场景：C9 获取提现方式列表
输入（推测）：
  - currencyCode: string
  - uid: i64
输出（推测）：
  - list<WithdrawMethod>
    - methodId: i64
    - methodName: string
    - feeRate: double       // 手续费率
    - feeFixed: i64         // 固定手续费
    - minAmount: i64
    - maxAmount: i64
    - dailyLimit: i64       // 单日限额
    - dailyCount: i32       // 单日次数限制
```

#### 方法 C：CreatePayOrder（创建通道订单）

```
触发场景：C4 创建充值订单
输入（推测）：
  - uid: i64
  - currencyCode: string
  - amount: i64
  - methodId: i64
  - orderNo: string          // 钱包侧生成的订单号
  - callbackInfo: struct     // 回调信息
输出（推测）：
  - payUrl: string           // 支付跳转 URL
  - channelOrderNo: string   // 通道订单号
  - expireAt: i64            // 订单过期时间
```

### 8.3 阻塞性分析

```
阻塞程度：★★★★★（最高）

影响范围：
  C3 获取充值方式列表 — 完全依赖财务接口，无法自造数据
  C4 创建充值订单     — 必须通过财务模块对接支付通道
  C9 获取提现方式列表 — 完全依赖财务接口
  C12 创建提现订单    — 提现审核流程可能在财务侧

应对策略：
  1. 定义 interface（Go 接口），隔离财务依赖
  2. 实现 MockFinanceClient，返回硬编码测试数据
  3. 财务模块 IDL 确定后，实现 RealFinanceClient 替换 Mock
  4. 两者实现同一个 interface，通过配置/环境变量切换
```

### 8.4 Mock 接口定义（建议）

```go
// internal/domain/finance_client.go

type FinanceClient interface {
    // 获取充值/提现方式列表
    GetPaymentConfig(ctx context.Context, currencyCode string, bizType int32) (*PaymentConfigResp, error)
    // 创建通道订单
    CreatePayOrder(ctx context.Context, req *CreatePayOrderReq) (*CreatePayOrderResp, error)
}

// Mock 实现
type MockFinanceClient struct{}

func (m *MockFinanceClient) GetPaymentConfig(ctx context.Context, currencyCode string, bizType int32) (*PaymentConfigResp, error) {
    // TODO: 返回硬编码测试数据，待财务模块接口确定后替换
    return &PaymentConfigResp{...}, nil
}
```

---

## 九、依赖 #7：活动模块（服务名待定）— 奖金活动查询

### 9.1 接口信息

| 项目 | 内容 |
|------|------|
| **目标服务** | 活动模块（ser-activity / ser-act，服务名待定） |
| **IDL 状态** | ❌ 未定义 |
| **Client 注册** | ❌ 无 |
| **需求依据** | D8 — 需求5.6，活动关联充值 |
| **优先级** | 应该有 |
| **开发策略** | **Mock 占位**，当前版本充值流程可不关联奖金 |

### 9.2 需要调用的方法（根据需求推导）

#### 方法 A：GetAvailableBonuses（获取可用奖金活动）

```
触发场景：C6 获取可用奖金活动列表（充值页面下方展示可选奖金）
输入（推测）：
  - uid: i64
  - currencyCode: string
  - depositAmount: i64       // 充值金额（影响活动门槛筛选）
输出（推测）：
  - list<BonusActivity>
    - activityId: i64
    - activityName: string
    - bonusRate: double      // 奖金比例（如充100送10%）
    - turnoverMultiple: i32  // 流水倍数要求
    - minDeposit: i64        // 最低充值门槛
    - maxBonus: i64          // 奖金上限
```

#### 方法 B：ValidateBonus（校验奖金活动有效性，可选）

```
触发场景：C4 创建充值订单（用户选择了某个奖金活动时）
输入（推测）：
  - uid: i64
  - activityId: i64
  - depositAmount: i64
输出（推测）：
  - isValid: bool
  - bonusAmount: i64         // 实际奖金金额
  - turnoverMultiple: i32    // 流水倍数
```

### 9.3 阻塞性分析

```
阻塞程度：★★☆☆☆（较低）

原因：
  - 充值核心流程不依赖奖金活动
  - C6 接口可以返回空列表（无可用活动）
  - C4 中 bonusActivityId 可选参数，不传即不关联奖金
  - 奖金入账是活动模块调用 ser-wallet.RewardCredit（别人调我），不是我调别人

应对策略：
  - 在 C6 接口先返回空列表
  - C4 中如果传了 activityId 先忽略或记录到订单中
  - 活动模块就绪后再打通
```

---

## 十、依赖 #8：ser-bcom — 功能开关查询（可选）

### 10.1 接口信息

| 项目 | 内容 |
|------|------|
| **调用方法** | `rpc.BComClient().SwitchConfigList(ctx, req)` |
| **IDL 文件** | `common/idl/ser-bcom/service.thrift` |
| **IDL 状态** | ✅ 已定义 |
| **Client 注册** | ✅ `BComClient()` 已有 |
| **优先级** | 可选 |

### 10.2 钱包中的可能场景

```
场景：全局充值/提现开关
  → 运营后台配置"充值开关=关闭"
  → 用户打开充值页面时，钱包先查 SwitchConfigList
  → 开关关闭 → 返回"充值功能暂时维护中"

现状：
  → 当前仅 gate-back 在 handler 层调用 SwitchConfigList
  → 没有任何 service 模块在业务逻辑层调用
  → 钱包的充值/提现开关是否通过 ser-bcom 管理，需求中未明确
  → **建议暂不接入**，等确认后再评估
```

---

## 十一、依赖 #9：ser-cron — 定时任务注册（待确认）

### 11.1 接口信息

| 项目 | 内容 |
|------|------|
| **调用方法** | `rpc.CronClient().???` |
| **IDL 文件** | `common/idl/ser-cron/service.thrift` |
| **IDL 状态** | ⚠️ **空壳** — service CronService {} 无任何方法定义 |
| **Client 注册** | ✅ `CronClient()` 已注册（但无方法可调） |
| **需求依据** | D9 — 币种配置4.2汇率规则，定频刷新 |
| **优先级** | 应该有（汇率刷新是核心功能） |

### 11.2 钱包的定时任务需求

```
任务：汇率定时刷新
  触发频率：可配置（如每1分钟 / 5分钟）
  逻辑：
    1. 查询所有启用的非基准币种
    2. 并行调用 3 家三方汇率 API（HTTP）
    3. 计算平均汇率
    4. 偏差 >= 阈值% → 更新平台汇率 + 写变更日志
    5. 更新 Redis 缓存
```

### 11.3 接入方式分析

```
可能的方案（需与 ser-cron 负责人确认）：

方案A：ser-wallet 内置 cron（自治）
  → 使用 Go 的 robfig/cron 库或 time.Ticker
  → 优点：不依赖外部服务，开发简单
  → 缺点：多实例部署时需分布式锁防重复执行

方案B：ser-cron 通过 RPC 回调触发
  → ser-wallet 暴露一个 RefreshExchangeRate RPC 接口
  → ser-cron 按配置频率调用
  → 但 ser-cron IDL 为空，注册机制不明

方案C：ser-cron 通过消息队列/配置中心触发
  → 非 RPC 方式
  → 需确认 ser-cron 的实际实现机制

建议：先按方案A实现（内置 cron + 分布式锁），不依赖 ser-cron。
如果后续有统一调度需求，再迁移到 ser-cron 调度模式。
```

---

## 十二、依赖 #10：ser-i18n — 错误码国际化翻译（非 RPC）

### 12.1 接入方式

```
ser-i18n 不是通过 RPC 调用的，而是通过 Redis Hash 直读。

工程中已有统一框架支持：
  ret.BizErr(ctx, errs.XxxCode, "默认消息")
  → 框架内部自动查询 Redis Hash：i18n:{lang}:{errCode}
  → 有翻译用翻译，没有用默认消息

ser-wallet 需要做的：
  1. 定义自己的错误码常量（internal/pkg/errs/wallet_errors.go）
  2. 在 ser-i18n 的 B端管理页面录入各语言翻译
  3. 代码中用 ret.BizErr(ctx, errCode, defaultMsg) 即可
  4. 无需任何 RPC Client 调用
```

### 12.2 钱包需要的错误码（列举核心）

```
// 钱包通用
WalletNotFound          余额不足时
InsufficientBalance     余额不足
CurrencyDisabled        币种已禁用
CurrencyNotFound        币种不存在

// KYC 相关
KycNotPassed            请先完成KYC认证
KycServiceUnavailable   KYC服务异常

// 充值相关
DepositAmountInvalid    充值金额不合法
DuplicateDeposit        存在重复待支付订单

// 提现相关
WithdrawLimitExceeded   超出提现限额
WithdrawDailyLimitExceeded 超出单日提现次数
WithdrawAccountInvalid  提现账户信息不完整

// 兑换相关
ExchangeRateExpired     汇率已过期，请重新获取
ExchangeAmountInvalid   兑换金额不合法

// 通用 RPC
UserServiceErr          用户服务异常
FinanceServiceErr       财务服务异常
```

---

## 十三、依赖 #11：三方汇率 API（外部 HTTP 调用）

### 13.1 概述

| 项目 | 内容 |
|------|------|
| **调用类型** | HTTP GET/POST（非 RPC） |
| **调用数量** | 3 家 API 并行调用取平均值 |
| **触发场景** | 定时任务 — 汇率刷新（RefreshExchangeRate） |
| **需求依据** | 币种配置4.2 汇率规则 |
| **优先级** | 核心必须（汇率是多币种钱包的基础数据） |

### 13.2 调用模式

```
定时触发（每 N 分钟）
  → 查询所有启用的非基准币种（如 BRL、PHP、VND...）
  → 对每个币种：
      → goroutine 并行调 3 家 API
        → API-1.GetRate(baseCurrency=USD, target=BRL)
        → API-2.GetRate(baseCurrency=USD, target=BRL)
        → API-3.GetRate(baseCurrency=USD, target=BRL)
      → 3 家结果取平均（过滤超时/异常的）
      → 至少 2 家成功才视为有效
  → 对比当前平台汇率：
      → |实时汇率 - 平台汇率| / 实时汇率 × 100% >= 阈值%
        → 更新平台汇率 = 实时汇率
        → 入款汇率 = 实时汇率 × (1 + 浮动%)
        → 出款汇率 = 实时汇率 × (1 - 浮动%)
        → 写入汇率变更日志
      → 偏差 < 阈值%
        → 仅更新实时汇率显示值，不变更平台汇率
  → 更新 Redis 缓存
```

### 13.3 开发要点

```
1. HTTP Client 独立封装
   → internal/pkg/exchange/ 下定义 ExchangeRateProvider interface
   → 每家 API 实现一个 provider（如 provider_a.go, provider_b.go, provider_c.go）
   → 配置化：API URL / Key / 超时 从配置文件或配置中心读取

2. 容错策略
   → 单家 API 超时（建议 3-5 秒）不阻塞整体
   → 3 家中至少 2 家成功 → 取平均
   → 3 家中仅 1 家成功 → 使用该结果但标记告警
   → 3 家全失败 → 保持当前汇率不变 + 触发告警
   → 连续 N 次全失败 → 升级告警（可能 API Key 过期/服务方故障）

3. 精度处理
   → 三方 API 返回的通常是 float64
   → 内部存储和计算统一转为最小单位 i64
   → 汇率本身可用 float64 存储（仅用于兑换计算），最终金额用 i64

4. 开发阶段
   → 先实现 1 家 API 跑通链路
   → 后续增加第 2、3 家
   → 具体 API 选型需与产品/运营确认（常见：ExchangeRate-API、Open Exchange Rates、CurrencyLayer）
```

---

## 十四、依赖 #12：风控模块（可选/预留）

### 14.1 接口信息

| 项目 | 内容 |
|------|------|
| **目标服务** | 风控模块（服务名待定） |
| **IDL 状态** | ❌ 未定义 |
| **需求依据** | D12 — 需求10 风控与异常处理 |
| **优先级** | 可选/预留 |
| **开发策略** | 当前版本不接入，预留接口位 |

### 14.2 说明

```
需求10 提到"大额提现风控校验"、"异常频率检测"等功能。
但：
  1. 风控模块服务名、IDL 均未定义
  2. 风控策略（阈值、规则）未确定
  3. 当前版本的提现限额校验可以在钱包内部实现（简单的金额/次数校验）
  4. 复杂风控（设备指纹、行为模式、IP地理围栏）超出钱包职责

建议：
  → 钱包内部实现基础限额校验（每日限额、单笔限额、每日次数）
  → 在 CreateWithdrawOrder 流程中预留 riskCheck() 方法占位
  → 风控模块就绪后，riskCheck() 内部改为调用 RPC
```

---

## 十五、全量依赖清单（按开发优先级排序）

### 15.1 第一批：可直接接入（IDL 已定义，Client 已注册）

| 序号 | 目标服务 | 方法 | 场景 | 钱包哪个接口用到 |
|------|---------|------|------|-----------------|
| 1 | ser-blog | `AddActLog` | 所有B端写操作审计日志 | B1~B7 所有写操作 |
| 2 | ser-kyc | `KycQuery` | 提现KYC状态校验 | C9, C11, C12 |
| 3 | ser-user | `GetUserInfoExpend` | 获取用户信息/KYC姓名 | C11, B3/B4(可选) |
| 4 | ser-user | `BatchGetUserInfo` | B端批量获取用户昵称 | B6 |
| 5 | ser-s3 | `UploadIcon/UploadFile` | 币种图标上传 | B2 |
| 6 | ser-buser | `QueryUserByIds` | B端操作人姓名回显 | B1, B5 |

**开发建议**：这 6 个可以在骨架阶段就接入，调用模式参照 ser-app / ser-item 现有代码。

### 15.2 第二批：需 Mock 占位（服务名/IDL 待定）

| 序号 | 目标服务 | 方法（推测） | 场景 | 阻塞等级 |
|------|---------|-------------|------|---------|
| 7 | 财务模块 | GetPaymentConfig | 获取充值方式配置 | ★★★★★ |
| 8 | 财务模块 | GetWithdrawMethods | 获取提现方式配置 | ★★★★★ |
| 9 | 财务模块 | CreatePayOrder | 创建通道订单 | ★★★★★ |
| 10 | 活动模块 | GetAvailableBonuses | 获取可用奖金活动 | ★★☆☆☆ |

**开发建议**：定义 Go interface 抽象层 + Mock 实现，财务模块 IDL 确定后替换为 RPC 实现。

### 15.3 第三批：需确认接入方式

| 序号 | 依赖 | 说明 | 建议 |
|------|------|------|------|
| 11 | ser-cron | IDL 为空壳 | 内置 cron + 分布式锁 |
| 12 | 三方汇率API | HTTP 调用（3家） | 独立封装 ExchangeRateProvider |
| 13 | ser-bcom 开关 | 钱包是否需要 | 暂不接入 |
| 14 | 风控模块 | 服务未就绪 | 预留方法占位 |

### 15.4 非 RPC 依赖

| 序号 | 依赖 | 接入方式 | 说明 |
|------|------|---------|------|
| 15 | ser-i18n | Redis 直读 | 框架层已集成，只需定义错误码 |
| 16 | ETCD 配置 | SDK 直读 | 服务发现 + 可能的配置热更新 |
| 17 | Redis | 直连 | 缓存汇率/余额/分布式锁 |

---

## 十六、依赖关系全景图

```
                        ┌──────────────────────────────────────────────────┐
                        │               ser-wallet                        │
                        │         （钱包 + 多币种配置）                     │
                        └─┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬─────────────┘
                          │  │  │  │  │  │  │  │  │  │  │  │
      ┌───────────────────┘  │  │  │  │  │  │  │  │  │  │  └──────────────┐
      │         ┌────────────┘  │  │  │  │  │  │  │  │  └─────────┐       │
      │         │     ┌─────────┘  │  │  │  │  │  │  └────────┐   │       │
      │         │     │      ┌─────┘  │  │  │  │  └───────┐   │   │       │
      ▼         ▼     ▼      ▼        │  │  │  │          │   │   │       │
 ┌─────────┐┌──────┐┌────┐┌─────┐    │  │  │  │          │   │   │       │
 │ser-blog ││ser-  ││ser-││ser- │    │  │  │  │          │   │   │       │
 │ AddAct  ││kyc   ││user││s3   │    │  │  │  │          │   │   │       │
 │ Log     ││KycQ  ││GU/ ││Upld │    │  │  │  │          │   │   │       │
 │(oneway) ││uery  ││BGU ││Icon │    │  │  │  │          │   │   │       │
 └─────────┘└──────┘└────┘└─────┘    │  │  │  │          │   │   │       │
                                      │  │  │  │          │   │   │       │
  ✅可直接接入  ────────────────────────  │  │  │          │   │   │       │
                                         │  │  │          │   │   │       │
      ┌──────────────────────────────────┘  │  │          ▼   │   │       │
      │         ┌───────────────────────────┘  │     ┌──────┐ │   │       │
      ▼         ▼                              │     │ser-  │ │   │       │
 ┌──────────┐┌───────────┐                    │     │buser │ │   │       │
 │ 财务模块 ││ 活动模块  │                    │     │Query │ │   │       │
 │GetPay... ││GetAvail...│                    │     │Users │ │   │       │
 │CreatePay ││Validate.. │                    │     └──────┘ │   │       │
 │(Mock占位)││(Mock占位) │                    │    ✅直接接入 │   │       │
 └──────────┘└───────────┘                    │              │   │       │
                                               │              │   │       │
  ❌需Mock占位  ───────────────────────────────  │              │   │       │
                                                  │              │   │       │
      ┌───────────────────────────────────────────┘              │   │       │
      │                ┌─────────────────────────────────────────┘   │       │
      ▼                ▼                                             │       │
 ┌──────────┐    ┌──────────────┐                                   │       │
 │ ser-cron │    │ ser-bcom     │                                   │       │
 │ (空壳IDL)│    │ SwitchConfig │                                   │       │
 │ 建议内置  │    │ (暂不接入)   │                                   │       │
 └──────────┘    └──────────────┘                                   │       │
                                                                    │       │
  ⚠️需确认  ────────────────────────────────────────────────────────  │       │
                                                                     │       │
      ┌──────────────────────────────────────────────────────────────┘       │
      ▼                                                                      │
 ┌──────────────────┐                                                       │
 │ 三方汇率API ×3   │                                                       │
 │ (HTTP, 非RPC)    │                                                       │
 │ 取平均值         │                                                       │
 └──────────────────┘                                                       │
                                                                             │
      ┌──────────────────────────────────────────────────────────────────────┘
      ▼
 ┌──────────────────┐     ┌──────────────────┐
 │ 风控模块(预留)   │     │ ser-i18n         │
 │ RiskCheck        │     │ (非RPC, Redis)   │
 │ (不接入)         │     │ 框架已集成       │
 └──────────────────┘     └──────────────────┘
```

---

## 十七、与 RPC 暴露篇的对照闭环

```
完整依赖图 = 暴露篇 + 调用篇

                        别人调我（暴露篇）                我调别人（本篇）
                        ═══════════════════              ═══════════════════
财务模块  ──RPC──→  CreditWallet              ser-wallet ──RPC──→  财务模块
                    ConfirmDebit                           GetPaymentConfig
                    UnfreezeBalance                        CreatePayOrder
                    SupplementCredit
                    ManualCredit
                    ManualDebit

投注模块  ──RPC──→  DebitWallet               ser-wallet ──RPC──→  ser-kyc
                    CreditWallet                           KycQuery
                    GetBalance
                                              ser-wallet ──RPC──→  ser-user
直播模块  ──RPC──→  GetBalance                             GetUserInfoExpend
                                                           BatchGetUserInfo

活动模块  ──RPC──→  RewardCredit              ser-wallet ──RPC──→  ser-blog
                                                           AddActLog

任何模块  ──RPC──→  GetBalance                ser-wallet ──RPC──→  ser-s3
                    BatchGetBalance                        UploadIcon/File
                    GetCurrencyConfig
                    GetExchangeRateRpc        ser-wallet ──RPC──→  ser-buser
                                                           QueryUserByIds

                                              ser-wallet ──HTTP──→ 三方汇率API ×3

双向依赖：
  wallet ←→ 财务模块（唯一的双向依赖，需提前对齐接口契约）
```

---

## 十八、开发阶段 Mock vs 真实依赖分类

### 18.1 分类标准

根据用户之前给出的编码阶段指导原则，外部依赖分两类：

**可延迟型（Deferrable）**：本地逻辑校验，先写 TODO 占位，不影响主流程
**阻塞型（Blocking）**：必须从外部获取数据才能继续，需要 Mock 返回模拟数据

### 18.2 分类结果

| 依赖 | 类型 | 原因 | 骨架阶段处理 |
|------|------|------|-------------|
| ser-kyc.KycQuery | 可延迟 | 本地可 `// TODO: KYC校验` 跳过 | TODO 占位 |
| ser-user.GetUserInfoExpend | 可延迟 | 可硬编码假 name | TODO 占位 |
| ser-user.BatchGetUserInfo | 可延迟 | B端列表昵称可空 | TODO 占位 |
| ser-blog.AddActLog | 可直接接入 | oneway，Client已有，0风险 | 直接调用 |
| ser-s3.UploadIcon | 可延迟 | 图标功能不影响核心链路 | TODO 占位 |
| ser-buser.QueryUserByIds | 可延迟 | 操作人名称可空 | TODO 占位 |
| 财务.GetPaymentConfig | **阻塞型** | 充值方式列表必须有数据 | Mock 返回假数据 |
| 财务.CreatePayOrder | **阻塞型** | 充值下单必须有通道响应 | Mock 返回假订单 |
| 活动.GetAvailableBonuses | 可延迟 | 返回空列表即可 | 返回空 |
| 三方汇率API | **阻塞型** | 汇率数据是多币种基础 | Mock 固定汇率 |
| 风控.RiskCheck | 可延迟 | 默认放行 | TODO 占位 |

### 18.3 骨架阶段依赖就绪度

```
骨架阶段可运行条件（最小依赖集）：

✅ 无需外部服务即可开发验证的模块：
   → 币种配置 CRUD（纯数据库操作）
   → 钱包余额查询（纯数据库 + Redis）
   → 钱包核心操作（上账/扣款/冻结/解冻/确认扣除 — RPC 暴露接口的实现）
   → 交易记录查询（纯数据库查询）

⚠️ 需 Mock 才能开发验证的模块：
   → C3 充值方式列表 — Mock 财务接口
   → C4 创建充值订单 — Mock 财务接口
   → C7/C8 汇率查询/兑换 — Mock 汇率数据
   → C9 提现方式列表 — Mock 财务接口

✅ 可直接接入无风险的：
   → ser-blog.AddActLog（oneway，写日志）
```

---

## 十九、需跨团队确认的事项清单

| 序号 | 确认对象 | 确认内容 | 影响 | 紧急度 |
|------|---------|---------|------|--------|
| 1 | 财务模块开发者 | 服务名（EtcdXxxService）、IDL 文件位置 | D4/D5/D6 全部依赖 | ★★★★★ |
| 2 | 财务模块开发者 | GetPaymentConfig 接口契约（入参/出参） | C3/C9 的数据来源 | ★★★★★ |
| 3 | 财务模块开发者 | CreatePayOrder 接口契约 + 回调机制 | C4 充值下单全链路 | ★★★★★ |
| 4 | 活动模块开发者 | 是否已有 GetAvailableBonuses RPC 接口 | C6 奖金活动 | ★★☆☆☆ |
| 5 | ser-cron 负责人 | 定时任务注册机制（RPC？DB？MQ？） | 汇率定时刷新 | ★★★☆☆ |
| 6 | ser-s3 负责人 | 钱包的 bsType 编号分配 | B2 图标上传 | ★★☆☆☆ |
| 7 | 产品/运营 | 三方汇率 API 选型（哪 3 家） | 汇率刷新实现 | ★★★☆☆ |
| 8 | 产品 | 充值/提现是否需要全局开关（ser-bcom） | 是否接入 SwitchConfig | ★☆☆☆☆ |
| 9 | 风控团队 | 风控模块开发进度和接口契约 | D12 风控校验 | ★☆☆☆☆ |

---

## 二十、总结

### 核心认知

1. **ser-wallet 的外部依赖总量 12-18 个**，其中 6-8 个可直接接入（IDL 已有），3-5 个需 Mock 占位（服务未就绪），其余需确认

2. **最大阻塞项是财务模块**：充值方式、提现方式、通道订单三个接口的 IDL 和服务名未定，直接影响 C3/C4/C9/C12 四个核心 C 端接口的开发。**这是需要最优先跨团队沟通的事项**

3. **已有 IDL 的服务可放心使用**：ser-kyc、ser-user、ser-blog、ser-s3、ser-buser 的方法签名已通过 IDL 源文件验证，调用模式有 ser-app、ser-item 等成熟代码可参考

4. **ser-cron IDL 为空壳**：汇率定时刷新建议内置 cron + 分布式锁，不依赖 ser-cron

5. **三方汇率 API 是唯一的外部 HTTP 依赖**：需要独立封装 Provider interface，支持多家 API 可插拔、容错降级

6. **与暴露篇形成完整闭环**：暴露篇 13 个接口（别人调我）+ 本篇 12-18 个依赖（我调别人）= ser-wallet 的完整 RPC 交互图谱。其中仅 wallet↔财务模块 是双向依赖

### 编码节奏建议

```
第一步：骨架 + 纯内部逻辑（不依赖任何外部服务）
  → 币种配置 CRUD + 钱包核心余额操作 + 记录查询

第二步：接入已有 IDL 的服务
  → ser-blog（最简单，oneway，先接）
  → ser-buser（B端列表需要）
  → ser-s3（币种图标上传）
  → ser-kyc + ser-user（提现链路需要）

第三步：Mock 财务模块接口
  → 定义 FinanceClient interface
  → 实现 MockFinanceClient
  → 跑通 C3/C4/C9/C12 链路（用假数据）

第四步：Mock 汇率 API
  → 实现 1 家 ExchangeRateProvider
  → 跑通汇率刷新 + 兑换链路

第五步：财务模块就绪后替换 Mock
  → RealFinanceClient 替换 MockFinanceClient
  → 联调充值/提现完整链路
```
