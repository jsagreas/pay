# 工程认知：从0到1开发新模块的完整工程理解

> 梳理时间：2026-02-26
> 梳理依据：深入分析 common、gate-back、gate-font 三个核心模块 + ser-app/ser-item 参考实现
> 梳理目的：站在钱包模块负责人视角，完整掌握工程全貌，确保开发符合规范、复用已有能力、不重复造轮子

---

## 一、整体技术架构

### 1.1 技术栈

| 层次 | 技术选型 | 版本 | 说明 |
|------|---------|------|------|
| 语言 | Go | 1.25 | Go Workspace 多模块管理 |
| HTTP框架 | Hertz (CloudWeGo) | v0.10.3 | C端/B端 API 网关 |
| RPC框架 | Kitex (CloudWeGo) | v0.16.0 | 微服务间通信 |
| IDL | Thrift | — | 接口定义语言，IDL先行开发 |
| ORM | GORM + GORM Gen | v1.31 / v0.3.27 | 数据库操作，代码生成 |
| 数据库 | TiDB（兼容MySQL） | — | 分布式数据库 |
| 缓存 | Redis | — | Token/缓存/消息 |
| 服务注册 | ETCD | v3.6.7 | 服务发现 + 配置中心 |
| 对象存储 | S3（兼容AWS S3） | — | 图片/文件存储 |
| 消息队列 | Kafka + NATS | — | 异步消息 |
| 搜索引擎 | Elasticsearch | — | 内容搜索 |
| 日志 | Zap + lumberjack | — | 结构化日志 + 滚动 |

### 1.2 工程目录总览

```
/Users/mac/gitlab/                    ← Go Workspace 根目录
├── go.work                           ← 工作区配置，声明所有模块
├── common/                           ← 【核心】公共模块（IDL + 生成代码 + 共享包）
│   ├── idl/                          ← 所有服务的 Thrift IDL 定义
│   ├── pkg/                          ← 共享 Go 包（kitex_gen/rpc/middleware/utils/...）
│   ├── rpc/                          ← RPC 服务端初始化 + 客户端工厂
│   └── script/                       ← 代码生成脚本（gen_kitex.sh）
├── gate-back/                        ← 【核心】B端 API 网关（后台管理）
├── gate-font/                        ← 【核心】C端 API 网关（App/H5）
├── ser-app/                          ← 业务服务：App配置（Banner/分类/内容策略）
├── ser-buser/                        ← 业务服务：后台用户/角色/权限
├── ser-bcom/                         ← 业务服务：后台通用配置
├── ser-item/                         ← 业务服务：道具/物品管理
├── ser-user/                         ← 业务服务：C端用户
├── ser-kyc/                          ← 业务服务：KYC认证
├── ser-i18n/                         ← 基础设施：国际化翻译
├── ser-ip/                           ← 基础设施：IP黑白名单
├── ser-blog/                         ← 基础设施：操作日志/审计
├── ser-s3/                           ← 基础设施：对象存储
├── ser-wallet/                       ← 【待开发】钱包模块（当前为空脚手架）
└── z-readme/                         ← 分析文档输出目录
```

### 1.3 请求链路总览

```
客户端 App/H5/Web
  │
  ▼ HTTP POST
网关层（gate-font / gate-back）
  │  Hertz 框架
  │  中间件链：IP检查 → Token校验 → 权限校验
  │
  ▼ RPC (Kitex + TTHeader)
服务层（ser-xxx）
  │  handler.go → internal/ser/ → internal/rep/
  │  ETCD服务发现，3秒超时
  │
  ▼ SQL/Redis/S3/MQ
基础设施层
  TiDB / Redis / S3 / Kafka / NATS / ES
```

---

## 二、common 模块深度解析

### 2.1 IDL 目录结构与规范

**路径：** `/Users/mac/gitlab/common/idl/`

```
common/idl/
├── common/
│   └── enum.thrift              ← 公共枚举定义，所有服务可 include
├── ser-app/
│   ├── service.thrift           ← 主文件：定义 AppService
│   ├── banner.thrift            ← 子文件：Banner 相关类型
│   ├── home_category.thrift     ← 子文件：分类相关类型
│   ├── content_strategy.thrift
│   └── s3.thrift
├── ser-kyc/
│   ├── service.thrift
│   ├── kyc.thrift
│   ├── user_kyc.thrift
│   └── enum.thrift
├── ser-item/
│   ├── service.thrift
│   ├── item.thrift
│   ├── item_publish.thrift
│   └── s3.thrift
└── [21个服务目录...]
```

**IDL 命名规范（从现有服务提取的规则）：**

| 项目 | 规范 | 示例 |
|------|------|------|
| 目录名 | `ser-{模块名}` | `ser-app`、`ser-kyc`、`ser-wallet` |
| 主文件 | `service.thrift` | 每个服务有且仅有一个 |
| 命名空间 | `namespace go ser_{模块名}` | `namespace go ser_app` |
| 服务名 | `{模块名}Service` | `AppService`、`KycService` |
| 请求类型 | `{操作名}Req` | `CreateBannerReq`、`KycSubmitReq` |
| 响应类型 | `{操作名}Resp` | `CreateBannerResp`、`KycSubmitResp` |
| 数据模型 | `{实体名}Info` 或 `{实体名}` | `UserKycInfo`、`ItemInfo` |
| Include | `include "../common/enum.thrift"` | 公共枚举引用 |

**IDL 方法命名规范：**

```thrift
service AppService {
    // 增删改查标准命名
    ser_app.CreateBannerResp    CreateBanner(1: ser_app.CreateBannerReq req)
    ser_app.UpdateBannerResp    UpdateBanner(1: ser_app.UpdateBannerReq req)
    ser_app.DeleteBannerResp    DeleteBanner(1: ser_app.DeleteBannerReq req)
    ser_app.StatusBannerResp    StatusBanner(1: ser_app.StatusBannerReq req)
    ser_app.PageBannerResp      PageBanner(1: ser_app.PageBannerReq req)
    ser_app.DetailBannerResp    DetailBanner(1: ser_app.DetailBannerReq req)
    ser_app.ListBannerResp      ListBanner(1: ser_app.ListBannerReq req)

    // 布尔返回
    bool UpdateItem(1: item.ItemUpdateReq req)

    // 无参方法
    ser_app.ListHomeCategoryResp ListHomeCategory()

    // 文件操作
    ser_app.UploadBannerImageResp UploadBannerImage(1: ser_app.UploadBannerImageReq req)
}
```

**Request/Response 结构模式：**

```thrift
// 分页请求
struct PageBannerReq {
    1: required i32 pageNo
    2: required i32 pageSize
    3: optional string bannerName       // 可选筛选条件
    4: optional i32 status
}

// 分页响应
struct PageBannerResp {
    1: optional list<BannerInfo> pageList
    2: optional i64 total
    3: optional i32 pageNo
    4: optional i32 pageSize
}

// 实体信息
struct BannerInfo {
    1: required i64 id
    2: required string bannerName
    3: optional string imageUrl
    // ... 所有字段
}
```

### 2.2 gen_kitex.sh 代码生成脚本

**路径：** `/Users/mac/gitlab/common/script/gen_kitex.sh`（71行）

**执行方式：**
```bash
cd /Users/mac/gitlab/common
bash script/gen_kitex.sh
```

**核心逻辑：**
1. 扫描 `common/idl/` 下所有子目录
2. 在每个子目录中查找 `*service.thrift` 文件
3. 对每个找到的 thrift 文件执行：
   ```bash
   kitex -module common/pkg -I $IDL_DIR -service $SERVICE_NAME -gen-path ./kitex_gen/ $THRIFT_FILE
   ```
4. 如果对应的服务目录（如 `ser-app/`）已存在 handler.go 和 main.go，会先复制到 pkg 目录处理，处理完再拷贝回去
5. 最终清理临时文件（bootstrap.sh、kitex_info.yaml、build.sh、scripts/）

**生成产物：**
- `common/pkg/kitex_gen/ser_{模块名}/` — 生成的类型定义
- `common/pkg/kitex_gen/ser_{模块名}/{模块名}service/` — 生成的服务客户端/服务端接口
- `{服务目录}/handler.go` — 更新的 handler 模板（如果服务目录存在）
- `{服务目录}/main.go` — 更新的 main 模板（如果服务目录存在）

**重要：** `kitex_gen/` 下的代码是**自动生成的，禁止手动编辑**。每次修改 IDL 后重新执行脚本会覆盖。

### 2.3 RPC Client 工厂模式

**路径：** `/Users/mac/gitlab/common/rpc/rpc_client.go`（419行）

**核心模式：sync.Once 单例工厂**

```go
// 每个服务固定三要素
var (
    appService *appservice.Client    // 客户端实例指针
    appOnce    sync.Once             // 保证只初始化一次
)

// 工厂函数
func AppClient() appservice.Client {
    appOnce.Do(func() {
        cls, err := appservice.NewClient(
            namesp.EtcdAppService,        // ETCD中的服务名
            InitRpcClientParams()...,     // 通用客户端参数
        )
        appService = &cls
        if err != nil {
            lg.Log().CtxErrorf(context.Background(), "RPC 创建客户端错误 %s", err)
        }
    })
    return *appService
}
```

**当前已注册的21个服务客户端：**

| 编号 | 函数名 | ETCD服务名 | 对应服务 |
|------|--------|-----------|---------|
| 002 | BUserClient() | b_user_service | ser-buser |
| 003 | BComClient() | b_com_service | ser-bcom |
| 004 | I18nClient() | i18n_service | ser-i18n |
| 005 | IpClient() | ip_service | ser-ip |
| 006 | BLogClient() | b_log_service | ser-blog |
| 007 | S3Client() | s3_service | ser-s3 |
| 008 | UserClient() | user_service | ser-user |
| 009 | ExpClient() | exp_service | ser-exp |
| 010 | ItemClient() | item_service | ser-item |
| 011 | AppClient() | app_service | ser-app |
| 012 | KycClient() | kyc_service | ser-kyc |
| 014 | RecommendClient() | recommend_service | ser-recommend |
| 015 | ShortVideosClient() | short_videos_service | ser-shortvideos |
| 016 | ContentCenterClient() | content_center_service | ser-contentcenter |
| 017 | WebSocketClient() | websocket_service | ser-websocket |
| 018 | RtcClient() | rtc_service | ser-rtc |
| 019 | LiveClient() | live_service | ser-live |
| 020 | CronClient() | cron_service | ser-cron |
| 021 | MallClient() | mall_service | ser-mall |

**注意：ser-wallet 尚未注册，编号应为 022（或需确认）。**

**RPC 客户端通用参数：** `/Users/mac/gitlab/common/rpc/init.go`

```go
func InitRpcClientParams() []client.Option {
    r, _ := etcd.NewEtcdResolver(myetcd.Address())
    return []client.Option{
        client.WithResolver(r),                    // ETCD 服务发现
        client.WithRPCTimeout(3 * time.Second),    // 3秒超时
        client.WithMuxConnection(1),               // 多路复用
        client.WithMetaHandler(transmeta.ClientTTHeaderHandler),  // TraceID 传播
        client.WithTransportProtocol(transport.TTHeader),
    }
}
```

**RPC 服务端通用参数：**

```go
func InitRpcServerParams(servName string, servPort int) []server.Option {
    r, _ := etcd.NewEtcdRegistry(myetcd.Address())
    addr, _ := net.ResolveTCPAddr("tcp", ":"+strconv.Itoa(servPort))
    return []server.Option{
        server.WithReusePort(true),              // 端口复用
        server.WithMuxTransport(),               // 多路复用
        server.WithRegistry(r),                  // ETCD 注册
        server.WithServerBasicInfo(&rpcinfo.EndpointBasicInfo{ServiceName: servName}),
        server.WithMetaHandler(transmeta.ServerTTHeaderHandler),
        server.WithServiceAddr(addr),
        server.WithMiddleware(rpcAccessLogMiddleware),  // RPC 访问日志
    }
}
```

### 2.4 namesp.go — 服务命名与 ETCD 路径

**路径：** `/Users/mac/gitlab/common/pkg/consts/namesp/namesp.go`（117行）

**命名规范：**

```go
// 每个服务固定三个常量
const (
    EtcdAppService = "app_service"           // 服务注册名（Kitex 用）
    EtcdAppPort    = "/slg/serv/app/port"    // 端口配置路径
    EtcdAppDb      = "/slg/serv/app/db"      // 数据库名配置路径
)
```

**ETCD 路径体系：**

```
/slg/
├── conf/                          ← 全局配置
│   ├── tidb/addr                  ← TiDB 地址
│   ├── tidb/port
│   ├── tidb/username
│   ├── tidb/password
│   ├── redis/addr
│   ├── redis/password
│   ├── s3/ak, sk, endpoint, bucket, domain, region
│   └── es/addr, username, password
│
└── serv/                          ← 服务配置
    ├── gate_font/port, domain
    ├── gate_back/port, domain
    ├── buser/port, db
    ├── bcom/port, db
    ├── app/port, db
    ├── item/port, db
    ├── kyc/port, db, secret
    └── [其他服务]/port, db, ...
```

### 2.5 HTTP Handler 泛型封装

**路径：** `/Users/mac/gitlab/common/rpc/handler.go`（110行）

这是连接 HTTP 网关和 RPC 服务的桥梁，三个关键函数：

```go
// 标准请求-响应转发（最常用）
func Handler[REQ any, RESP any](
    ctx context.Context,
    c *app.RequestContext,
    rpcMethod func(context.Context, REQ, ...callopt.Option) (RESP, error),
)
// 内部流程：BindAndValidate(JSON→REQ) → rpcMethod(ctx, req) → HttpOne(ctx, resp, i18n, err)

// 无参数请求转发
func HandlerNoParam[RESP any](
    ctx context.Context,
    c *app.RequestContext,
    rpcMethod func(context.Context, ...callopt.Option) (RESP, error),
)

// 支持 CSV 导出的转发
func HandlerWithCSV[REQ any, RESP any](
    ctx context.Context,
    c *app.RequestContext,
    rpcMethod func(context.Context, REQ, ...callopt.Option) (RESP, error),
)
// 如果 RESP 实现了 CSVResponse 接口，返回文件下载；否则返回 JSON
```

**使用方式（一行代码完成网关转发）：**

```go
// 在 gate-back 或 gate-font 的 handler 中
func CreateBanner(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.AppClient().CreateBanner)
}
```

### 2.6 统一响应格式

**路径：** `/Users/mac/gitlab/common/pkg/ret/project_ret.go`

```go
type R struct {
    TraceId string         `json:"traceId"`   // 链路追踪 UUID
    Code    errs.ErrorType `json:"code"`      // 0=成功，1=严重错误，其他=业务错误码
    Msg     string         `json:"msg"`       // 错误信息（i18n翻译后）
    Data    interface{}    `json:"data"`      // 业务数据
}
```

**成功响应示例：**
```json
{"traceId": "abc123", "code": 0, "msg": "", "data": {"id": 1, "name": "test"}}
```

**错误响应示例：**
```json
{"traceId": "abc123", "code": 6011001, "msg": "Banner名称已存在", "data": null}
```

**关键函数：**
- `ret.HttpOk(ctx, data)` → 成功响应
- `ret.HttpErr(ctx, err, i18nClient)` → 错误响应（自动 i18n 翻译）
- `ret.HttpOne(ctx, data, i18nClient, err)` → 自动判断成功/失败
- `ret.BizErr(ctx, errCode, msg)` → 在服务层抛出业务错误
- `ret.BizErrExt(ctx, errCode, extraMap, msg)` → 带扩展信息的业务错误

### 2.7 错误码体系

**系统级错误码：** `/Users/mac/gitlab/common/pkg/errs/code.go`

```go
const (
    OK                  = 0          // 成功
    NG                  = 1          // 未捕获严重错误
    SystemBaseError     = 6000000    // 系统基准
    OperateErr          = 6000001    // 操作错误
    ParamErr            = 6000002    // 参数错误
    TokenNotExist       = 6000008    // Token不存在
    TokenAnalysisFailed = 6000009    // Token解析失败
    CheckUrlAuthFailed  = 6000010    // URL权限校验失败
    RpcRequestErr       = 6000014    // RPC请求错误
)
```

**业务错误码规范：** 每个服务有独立范围

```
格式：60{服务编号}{模块编号}{序号}
ser-buser: 6002000~6002999
ser-bcom:  6003000~6003999
ser-app:   6011000~6011999
  ├── Banner:           6011000~6011099
  ├── HomeCategory:     6011100~6011199
  └── ContentStrategy:  6011200~6011299

ser-wallet 应使用: 60{编号}000（编号待确认）
```

### 2.8 共享工具包（禁止重复造轮子）

**路径：** `/Users/mac/gitlab/common/pkg/`

| 包 | 用途 | 关键函数/类型 |
|---|------|-------------|
| `utils` | 通用工具 | `S2Int()`, `S2Int64()`, `S2Float64()`, `MarshalStr()`, `UnmarshalStr()` |
| `utils/validator` | 校验 | `CheckPhone()`, `CheckEmail()`, `IsValidCode()` |
| `utils` (snowflake) | 分布式ID | `snowflake.NewNode()` → `node.Generate()` |
| `utils` (crypto) | 加解密 | `EncryptedString` 类型（GORM自动加解密） |
| `lg` | 日志 | `lg.Log().CtxInfof(ctx, "msg: %v", data)` |
| `tracer` | 链路追踪 | `GetTraceID(ctx)`, `GetUserId(ctx)`, `GetUserToken(ctx)` |
| `rds` | Redis | `rds.Set()`, `rds.Get()`, `rds.Del()`, `rds.HSet()`, `rds.ZAdd()` |
| `etcd` | 配置中心 | `etcd.DirectGet()`, `etcd.Get()`, `etcd.LoadAndWatch()` |
| `db/mysql` | 数据库 | `mysql.Init()`, `mysql.DB` |
| `mys3` | 对象存储 | `mys3.Init()`, `mys3.GetPresignUrl()` |
| `i18n` | 国际化 | 错误码翻译 |
| `session` | Token载荷 | `CUserTokenPayload`, `BUserTokenPayload` |
| `consts/redis_key` | Redis Key | 统一 Key 前缀管理 |
| `consts/kafka` | Kafka Topic | 消息主题定义 |
| `consts/nats` | NATS Subject | 消息主题定义 |
| `middleware` | 中间件 | IP检查/Token校验/权限校验/S3代理 |

---

## 三、gate-back（B端网关）深度解析

### 3.1 路由组织结构

```
gate-back/
├── main.go                          ← 入口：初始化 + 启动 Hertz
├── router.go                        ← 路由注册钩子
├── router_gen.go                    ← 调用 GeneratedRegister
├── biz/
│   ├── router/
│   │   ├── register.go              ← 中央注册：所有模块在此注册
│   │   ├── my_router/
│   │   │   └── my_router.go         ← 路由组初始化（Open/Api/S3）
│   │   ├── app/app_router.go
│   │   ├── item/item_router.go
│   │   ├── blog/blog_router.go
│   │   └── [更多模块路由]
│   └── handler/
│       ├── app/banner.go
│       ├── item/item.go
│       └── [更多模块handler]
```

### 3.2 路由组定义

**路径：** `gate-back/biz/router/my_router/my_router.go`

```go
var Open *route.RouterGroup   // 无需鉴权的路由
var Api  *route.RouterGroup   // 需要鉴权的路由

func InitMyRouter(h *server.Hertz) {
    // B端公开路由：/admin/open/...
    Open = h.Group("/admin/open",
        middleware.SetHeaderInfo(),      // 提取请求头信息
        middleware.WhiteCheck(),         // IP白名单校验（B端特有）
    )

    // B端鉴权路由：/admin/api/...
    Api = h.Group("/admin/api",
        middleware.SetHeaderInfo(),
        middleware.WhiteCheck(),
        middleware.BackAuthMiddleware(), // Token校验 + URL权限校验
    )

    // S3 代理
    s3Group := h.Group("/s3")
    s3Group.GET("/*path", middleware.RedirectS3())
}
```

### 3.3 路由注册模式（以 app 模块为例）

**路由文件：** `gate-back/biz/router/app/app_router.go`

```go
func RouterApp(r *server.Hertz) {
    // 所有B端接口用 my_router.Api（需要Token+权限）
    my_router.Api.POST("/app/banner/create", app.CreateBanner)
    my_router.Api.POST("/app/banner/update", app.UpdateBanner)
    my_router.Api.POST("/app/banner/delete", app.DeleteBanner)
    my_router.Api.POST("/app/banner/status", app.StatusBanner)
    my_router.Api.POST("/app/banner/page",   app.PageBanner)
    my_router.Api.POST("/app/banner/detail", app.DetailBanner)
    // ...
}
```

**中央注册：** `gate-back/biz/router/register.go`

```go
func GeneratedRegister(r *server.Hertz) {
    buser.RouterRole(r)
    bcom.RouterRole(r)
    i18n.RouterI18n(r)
    blog.RouterBlog(r)
    user.RouterUser(r)
    enum.RouterEnum(r)
    kyc.RouterKyc(r)
    item.RouterItem(r)
    app.RouterApp(r)
    exp.RouterExp(r)
    s3.RouterS3(r)
    live.RouterLive(r)
    shortvideos.RouterShortvideos(r)
    rtc.RouterRtc(r)
    // 新模块在此添加一行即可
}
```

### 3.4 Handler 模式

**路径：** `gate-back/biz/handler/app/banner.go`

```go
// 所有 handler 都是一行代码的薄封装
func CreateBanner(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.AppClient().CreateBanner)
}

func UpdateBanner(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.AppClient().UpdateBanner)
}

// CSV 导出用 HandlerWithCSV
func QueryUserCsv(ctx context.Context, c *app.RequestContext) {
    rpc.HandlerWithCSV(ctx, c, rpc.UserClient().QueryUserCsv)
}
```

### 3.5 B端 URL 路径规范

```
/admin/api/{模块}/{资源}/{操作}

示例：
POST /admin/api/app/banner/create
POST /admin/api/app/banner/page
POST /admin/api/item/page
POST /admin/api/item/publish/toggleOnline
POST /admin/api/bcom/switch/config/update
```

**关键约定：**
- 全部使用 **POST** 方法（包括查询）
- 路径格式：`/admin/api/{module}/{resource}/{action}`
- 所有请求体为 **JSON**
- 所有响应为统一 R 格式

### 3.6 B端中间件链

```
请求 → SetHeaderInfo() → WhiteCheck() → BackAuthMiddleware() → Handler
        提取请求头        IP白名单校验     Token+URL权限校验      业务处理
```

**BackAuthMiddleware 流程：**
1. 从 Header/Cookie 提取 Token
2. 从 Redis 读取 Token 对应的 `BUserTokenPayload`（直接用 token 作为 key，无前缀）
3. 解析管理员 ID 和用户名，注入 context
4. 调用 `BUserClient().CheckUrlAuth(ctx, c.FullPath())` 校验该管理员是否有该 URL 的权限
5. 任一步骤失败则 Abort

---

## 四、gate-font（C端网关）深度解析

### 4.1 与 gate-back 的核心差异

| 对比项 | gate-font（C端） | gate-back（B端） |
|--------|-----------------|-----------------|
| 路由前缀 | `/open`、`/api` | `/admin/open`、`/admin/api` |
| IP检查 | **黑名单**（BlackCheck） | **白名单**（WhiteCheck） |
| 鉴权中间件 | FontAuthMiddleware | BackAuthMiddleware |
| Token存储Key | `RedisTokenKey + token`（有前缀） | `token`（无前缀） |
| Token载荷 | `CUserTokenPayload`（含设备信息） | `BUserTokenPayload`（仅ID+用户名） |
| URL权限 | 无（只要登录就能访问） | 有（CheckUrlAuth 逐 URL 校验） |
| Token来源 | Header `X-User-Token` 或 Cookie `session_token` | Header `X-User-Token` 或 Cookie |

### 4.2 C端路由组定义

```go
func InitMyRouter(h *server.Hertz) {
    Open = h.Group("/open",
        middleware.SetHeaderInfo(),
        middleware.BlackCheck(),          // C端用黑名单
    )

    Api = h.Group("/api",
        middleware.SetHeaderInfo(),
        middleware.BlackCheck(),
        middleware.FontAuthMiddleware(),  // C端Token校验
    )
}
```

### 4.3 C端路由示例

```go
// user模块 — 公开接口（登录/注册）
my_router.Open.POST("/user/login", user.Login)
my_router.Open.POST("/user/register", user.Register)
my_router.Open.POST("/user/sendSmsCode", user.SendSmsCode)

// user模块 — 需登录接口
my_router.Api.POST("/user/getUserDetail", user.GetUserDetail)
my_router.Api.POST("/user/updateUserInfo", user.UpdateUserInfo)

// app模块 — 全部公开（首页内容）
my_router.Open.POST("/app/banner/list", app.ListBanner)
my_router.Open.POST("/app/category/list", app.QueryHomeCategoryList)
```

### 4.4 C端 URL 路径规范

```
/api/{模块}/{操作}

示例：
POST /api/user/getUserDetail
POST /api/shortvideos/publish
POST /api/live/api/room/start
POST /api/kyc/submit

公开：
POST /open/user/login
POST /open/app/banner/list
```

### 4.5 C端 Token 载荷

```go
type CUserTokenPayload struct {
    ID            int64   // 用户ID（主键）
    UID           int64   // 通用ID
    LastLoginIp   string  // 最后登录IP
    LastLoginTime int64   // 最后登录时间
    Device        string  // 设备信息
    DeviceType    string  // 设备类型（mobile/web）
    NickName      string  // 昵称
}
```

### 4.6 C端特有机制

- **Cookie Session**：登录成功后同时设置 Cookie（`session_token`），支持浏览器
- **双路 Token 获取**：先读 Header `X-User-Token`，无则读 Cookie `session_token`
- **CORS**：允许所有来源（`Access-Control-Allow-Origin: *`）
- **设备信息追踪**：Token 中含设备信息，中间件传播 `X-User-Device-ID`、`X-User-Device-Type`

---

## 五、微服务标准结构（以 ser-app 为参考）

### 5.1 目录结构

```
ser-app/
├── main.go                              ← 服务入口
├── handler.go                           ← RPC Handler（IDL方法实现，薄封装）
├── go.mod                               ← 模块定义
├── internal/
│   ├── cfg/                             ← 基础设施初始化
│   │   ├── mysql.go                     ← MySQL/GORM 初始化（sync.Once）
│   │   └── redis.go                     ← Redis 初始化（sync.Once）
│   ├── enum/                            ← 服务内枚举定义
│   │   ├── banner.go
│   │   └── enum_registry.go
│   ├── errs/                            ← 服务错误码
│   │   └── code.go
│   ├── ser/                             ← 业务逻辑层（Service）
│   │   ├── banner.go                    ← Banner 业务逻辑
│   │   ├── home_category_service.go
│   │   ├── s3_service.go
│   │   └── content_strategy.go
│   ├── rep/                             ← 数据访问层（Repository）
│   │   ├── banner.go                    ← Banner 数据库操作
│   │   └── home_category.go
│   ├── cache/                           ← 缓存层
│   │   └── banner_cache.go
│   ├── domain/                          ← 领域模型（可选）
│   ├── gen/                             ← 代码生成入口
│   │   └── gorm_gen.go                  ← GORM Gen 生成脚本
│   └── gorm_gen/                        ← GORM Gen 生成产物（禁止手动编辑）
│       ├── model/                       ← 数据库表模型
│       │   ├── banner.gen.go
│       │   └── home_category.gen.go
│       ├── query/                       ← 类型安全查询
│       │   ├── gen.go
│       │   ├── banner.gen.go
│       │   └── home_category.gen.go
│       └── repo/                        ← 通用 CRUD 操作
│           ├── bannerr/bannerr.go
│           └── homecategoryr/homecategoryr.go
```

### 5.2 main.go 标准模板

```go
func main() {
    // 1. 初始化日志
    lg.InitKLog()

    // 2. 初始化 ETCD + 配置监听
    etcd.LoadAndWatch("/slg/")

    // 3. 初始化基础设施（按需）
    cfg.InitRedis()
    // cfg.InitMySQL() 通常在 service 构造函数中懒加载

    // 4. 获取端口（命令行优先，ETCD兜底）
    port := flag.String("port", etcd.DirectGet(ctx, namesp.EtcdXxxPort), "端口号")
    flag.Parse()

    // 5. 创建业务 Service 实例
    xxxService := ser.NewXxxService()

    // 6. 创建 Kitex RPC 服务器
    srv := xxxservice.NewServer(
        &XxxServiceImpl{xxxService: xxxService},
        rpc.InitRpcServerParams(namesp.EtcdXxxService, utils.S2Int(ctx, *port))...,
    )

    // 7. 启动
    klog.Info("ser-xxx 服务启动成功，监听端口: " + *port)
    err := srv.Run()
    if err != nil {
        klog.Error("ser-xxx 运行错误", err.Error())
    }
}
```

### 5.3 handler.go 标准模板

```go
// handler.go 是 IDL 方法的直接实现
// 它只是一个薄封装层，将 RPC 请求转发给对应的 Service

type XxxServiceImpl struct {
    xxxService *ser.XxxService
    yyyService *ser.YyyService
}

// 每个 IDL 方法对应一个函数
func (s *XxxServiceImpl) CreateXxx(ctx context.Context, req *ser_xxx.CreateXxxReq) (resp *ser_xxx.CreateXxxResp, err error) {
    return s.xxxService.CreateXxx(ctx, req)
}

func (s *XxxServiceImpl) PageXxx(ctx context.Context, req *ser_xxx.PageXxxReq) (resp *ser_xxx.PageXxxResp, err error) {
    return s.xxxService.PageXxx(ctx, req)
}
```

**handler.go 的职责：** 仅做转发，不包含任何业务逻辑。

### 5.4 Service 层标准模式

```go
// internal/ser/xxx_service.go

type XxxService struct {
    repo  *rep.XxxRepo     // 数据访问
    cache *cache.XxxCache  // 缓存（可选）
}

func NewXxxService() *XxxService {
    return &XxxService{
        repo:  rep.NewXxxRepo(),
        cache: cache.NewXxxCache(),
    }
}

func (s *XxxService) CreateXxx(ctx context.Context, req *ser_xxx.CreateXxxReq) (*ser_xxx.CreateXxxResp, error) {
    // 1. 参数校验
    if req.Name == "" {
        return nil, ret.BizErr(ctx, errs.XxxParamErr, "名称不能为空")
    }

    // 2. 业务逻辑（如唯一性校验）
    exists, _ := s.repo.ExistsByName(ctx, req.Name)
    if exists {
        return nil, ret.BizErr(ctx, errs.XxxNameExists, "名称已存在")
    }

    // 3. 调用 Repository 写数据
    id, err := s.repo.Create(ctx, req)
    if err != nil {
        return nil, ret.BizErr(ctx, errs.XxxDBErr, err.Error())
    }

    // 4. 操作日志（B端操作需要）
    operator := tracer.GetUserName(ctx)
    rpc.BLogClient().CreateLog(ctx, ...)

    // 5. 返回结果
    return &ser_xxx.CreateXxxResp{Id: &id}, nil
}
```

### 5.5 Repository 层标准模式

```go
// internal/rep/xxx.go

type XxxRepo struct {
    db *gorm.DB
}

func NewXxxRepo() *XxxRepo {
    return &XxxRepo{db: cfg.InitMySQL()}  // 懒加载数据库连接
}

// 分页查询
func (r *XxxRepo) QueryPage(ctx context.Context, pageNo, pageSize int, filters ...) ([]*model.Xxx, int64, error) {
    q := query.Q.Xxx.WithContext(ctx).Where(query.Xxx.DeletedAt.Eq(0))
    if name != nil {
        q = q.Where(query.Xxx.Name.Like("%" + *name + "%"))
    }
    offset := (pageNo - 1) * pageSize
    return q.Order(query.Xxx.ID.Desc()).FindByPage(offset, pageSize)
}

// 创建
func (r *XxxRepo) Create(ctx context.Context, req *ser_xxx.CreateXxxReq) (int64, error) {
    now := time.Now().UnixMilli()
    entity := &model.Xxx{
        Name:      req.GetName(),
        CreatedAt: now,
        UpdatedAt: now,
    }
    err := query.Q.Xxx.WithContext(ctx).Create(entity)
    return entity.ID, err
}

// 软删除（设置 DeletedAt 字段）
func (r *XxxRepo) Delete(ctx context.Context, id int64) (int64, error) {
    now := time.Now().UnixMilli()
    res, err := query.Q.Xxx.WithContext(ctx).
        Where(query.Xxx.ID.Eq(id), query.Xxx.DeletedAt.Eq(0)).
        UpdateSimple(
            query.Xxx.DeletedAt.Value(now),
            query.Xxx.UpdatedAt.Value(now),
        )
    return res.RowsAffected, err
}
```

### 5.6 GORM Gen 代码生成

**生成入口：** `ser-app/internal/gen/gorm_gen.go`

```go
func main() {
    etcd.InitEtcd()
    mysql_gen.GenCodeWithAll(
        etcd.DirectGet(ctx, namesp.EtcdAppDb),  // 数据库名（从ETCD读取）
        "ser-app",                                // 服务名（决定输出路径）
        "",                                       // 去掉的表前缀
    )
}
```

**执行方式：**
```bash
cd /Users/mac/gitlab/ser-app/internal/gen
go run gorm_gen.go
```

**生成产物：**

```
ser-app/internal/gorm_gen/
├── model/                    ← 表结构体（自动映射）
│   ├── banner.gen.go         ← type Banner struct { ID int64; Name string; ... }
│   └── ...
├── query/                    ← 类型安全查询 DSL
│   ├── gen.go                ← query.SetDefault(db) 初始化
│   ├── banner.gen.go         ← query.Q.Banner.Name.Eq("xxx")
│   └── ...
└── repo/                     ← 通用 CRUD 模板（自动生成）
    └── bannerr/bannerr.go    ← bannerr.Create(), bannerr.GetOneById(), ...
```

**自动生成的 repo 模板提供 18 个通用方法：**

| 方法 | 说明 |
|------|------|
| `Create(ctx, entity)` | 新增 |
| `BatchCreate(ctx, entities)` | 批量新增 |
| `Update(ctx, entity)` | 实体更新（忽略零值） |
| `BatchUpdate(ctx, entities)` | 批量更新 |
| `BatchSave(ctx, entities)` | 批量保存（id=0新增，否则更新） |
| `UpdateByIds(ctx, entity, ids)` | 按ID批量更新 |
| `UpdateByConds(ctx, entity, conds...)` | 按条件更新 |
| `UpdateFields(ctx, entity, fields...)` | 更新指定字段（不忽略零值） |
| `UpdateFieldsByConds(ctx, entity, conds, fields...)` | 按条件更新指定字段 |
| `DeleteById(ctx, ids...)` | 按ID删除 |
| `DeleteByConds(ctx, conds...)` | 按条件删除 |
| `GetOneById(ctx, id)` | 按ID查单条 |
| `GetOneByConds(ctx, conds...)` | 按条件查单条 |
| `GetListByIds(ctx, ids)` | 按ID查列表 |
| `GetListByConds(ctx, conds...)` | 按条件查列表（上限10000） |
| `GetPageByConds(ctx, pageNo, pageSize, orderby, conds...)` | 分页查询 |
| `Count(ctx, conds...)` | 计数 |
| `Exists(ctx, conds...)` | 是否存在 |

**数据加密支持：** `encryptedFields` 映射表可指定需要加密的字段（如 KYC 的 phone/name/id_number），生成时自动使用 `utils.EncryptedString` 类型。

---

## 六、初始化链路与配置管理

### 6.1 服务启动完整链路

```
1. lg.InitKLog()
   ↓  初始化 Zap 日志（文件+stdout，JSON格式，100MB滚动）

2. etcd.InitEtcd()  → 读取 ETCD_ADDR 环境变量
   ↓  建立 ETCD 连接（单例）

3. etcd.LoadAndWatch("/slg/")
   ↓  加载所有配置到内存缓存 + 启动 Watch 监听变更
   ↓  支持 Hook 回调（key变更时触发）

4. cfg.InitRedis()  → 从 ETCD 读取 Redis 地址/密码
   ↓  初始化 Redis Universal Client（支持集群/哨兵）

5. cfg.InitMySQL()  → 从 ETCD 读取 TiDB 地址/用户/密码/库名
   ↓  初始化 GORM + query.SetDefault(db)
   ↓  连接池：idle=10, open=20, lifetime=1h

6. flag.Parse()     → 端口号从命令行或 ETCD 获取
   ↓

7. xxxservice.NewServer(impl, rpc.InitRpcServerParams(servName, port)...)
   ↓  注册到 ETCD，启用端口复用+多路复用+TraceID传播

8. srv.Run()        → Kitex 服务启动，开始接受 RPC 请求
```

### 6.2 ETCD 配置访问方式

```go
// 直接访问（不经过缓存，每次读 ETCD）
etcd.DirectGet(ctx, "/slg/serv/app/port")

// 缓存访问（从内存读取，需要先 LoadAndWatch）
etcd.Get("/slg/conf/tidb/addr")
```

### 6.3 环境配置 Key 统一定义

**路径：** `/Users/mac/gitlab/common/pkg/consts/env/env.go`

```go
const (
    TidbAddr     = "/slg/conf/tidb/addr"
    TidbPort     = "/slg/conf/tidb/port"
    TidbUsername = "/slg/conf/tidb/username"
    TidbPassword = "/slg/conf/tidb/password"
    RedisAddr    = "/slg/conf/redis/addr"
    RedisPassword = "/slg/conf/redis/password"
    S3Ak         = "/slg/conf/s3/ak"
    S3Sk         = "/slg/conf/s3/sk"
    // ...
)
```

---

## 七、链路追踪与上下文传播

### 7.1 Trace ID 生成与传播

```
客户端请求 → Hertz Tracer 生成 UUID → 写入 Response Header + Context
  → RPC 调用时通过 TTHeader 传播
    → 目标服务从 Context 提取 TraceID
      → 所有日志自动包含 TraceID
```

### 7.2 Context 中可提取的信息

```go
tracer.GetTraceID(ctx)       // → UUID 链路追踪ID
tracer.GetUserId(ctx)        // → int64 用户ID（Token中提取）
tracer.GetUserToken(ctx)     // → string Token
tracer.GetClientIp(ctx)      // → string 客户端IP
tracer.GetUserLang(ctx)      // → string 语言（"en", "zh_CN"等）
tracer.GetUserName(ctx)      // → string 用户名/昵称
tracer.GetDeviceID(ctx)      // → string 设备ID
tracer.GetDeviceType(ctx)    // → string 设备类型
```

### 7.3 Header 常量

```go
const (
    TraceIDHeader        = "X-Trace-Id"
    ClientIPHeader       = "X-Client-IP"
    UserTokenHeader      = "X-User-Token"
    UserIdHeader         = "X-User-ID"
    UserNameHeader       = "X-User-Name"
    UserLangHeader       = "X-User-Language"
    UserDeviceIDHeader   = "X-User-Device-ID"
    UserDeviceTypeHeader = "X-User-Device-Type"
)
```

---

## 八、从0到1开发 ser-wallet 的完整步骤

### 步骤1：创建 IDL 定义

```bash
mkdir -p /Users/mac/gitlab/common/idl/ser-wallet
```

创建文件结构：
```
common/idl/ser-wallet/
├── service.thrift           ← 主文件：定义 WalletService
├── wallet.thrift            ← 钱包相关类型（余额/流水/订单）
├── currency.thrift          ← 币种配置相关类型
├── exchange.thrift          ← 兑换相关类型
├── withdraw.thrift          ← 提现相关类型
└── deposit.thrift           ← 充值相关类型
```

service.thrift 示例结构：
```thrift
namespace go ser_wallet

include "wallet.thrift"
include "currency.thrift"
include "exchange.thrift"
include "withdraw.thrift"
include "deposit.thrift"
include "../common/enum.thrift"

service WalletService {
    // C端接口
    wallet.GetWalletHomeResp     GetWalletHome(1: wallet.GetWalletHomeReq req)
    wallet.GetCurrencyListResp   GetCurrencyList(1: wallet.GetCurrencyListReq req)
    deposit.GetDepositMethodsResp GetDepositMethods(1: deposit.GetDepositMethodsReq req)
    // ...

    // B端接口
    currency.PageCurrencyResp    PageCurrency(1: currency.PageCurrencyReq req)
    currency.EditCurrencyResp    EditCurrency(1: currency.EditCurrencyReq req)
    // ...

    // RPC 对外接口
    wallet.CreditWalletResp      CreditWallet(1: wallet.CreditWalletReq req)
    wallet.DebitWalletResp       DebitWallet(1: wallet.DebitWalletReq req)
    wallet.FreezeBalanceResp     FreezeBalance(1: wallet.FreezeBalanceReq req)
    // ...
}
```

### 步骤2：执行 Kitex 代码生成

```bash
cd /Users/mac/gitlab/common
bash script/gen_kitex.sh
```

**产物：**
- `common/pkg/kitex_gen/ser_wallet/` — 类型定义
- `common/pkg/kitex_gen/ser_wallet/walletservice/` — 服务接口
- `ser-wallet/handler.go` — Handler 模板（如果 ser-wallet 目录存在）
- `ser-wallet/main.go` — Main 模板

### 步骤3：注册 ETCD 命名空间

**编辑：** `/Users/mac/gitlab/common/pkg/consts/namesp/namesp.go`

```go
// EtcdWalletService 022
EtcdWalletService = "wallet_service"
EtcdWalletPort    = "/slg/serv/wallet/port"
EtcdWalletDb      = "/slg/serv/wallet/db"
```

### 步骤4：注册 RPC Client

**编辑：** `/Users/mac/gitlab/common/rpc/rpc_client.go`

```go
var (
    walletService *walletservice.Client
    walletOnce    sync.Once
)

func WalletClient() walletservice.Client {
    walletOnce.Do(func() {
        cls, err := walletservice.NewClient(
            namesp.EtcdWalletService,
            InitRpcClientParams()...,
        )
        walletService = &cls
        if err != nil {
            lg.Log().CtxErrorf(context.Background(), "RPC 创建 wallet 客户端错误 %s", err)
        }
    })
    return *walletService
}
```

### 步骤5：创建服务目录结构

```bash
mkdir -p ser-wallet/internal/{cfg,enum,errs,ser,rep,cache,domain,gen,gorm_gen}
```

### 步骤6：初始化数据库表结构

在 TiDB 中创建 ser-wallet 所需的表（钱包表、流水表、兑换记录表、币种配置表等），然后：

**创建：** `ser-wallet/internal/gen/gorm_gen.go`

```go
package main

import (
    "common/pkg/consts/namesp"
    mysql_gen "common/pkg/db/cmd"
    "common/pkg/etcd"
    "context"
)

func main() {
    etcd.InitEtcd()
    mysql_gen.GenCodeWithAll(
        etcd.DirectGet(context.Background(), namesp.EtcdWalletDb),
        "ser-wallet",
        "",
    )
}
```

执行：
```bash
cd /Users/mac/gitlab/ser-wallet/internal/gen
go run gorm_gen.go
```

**产物：** `ser-wallet/internal/gorm_gen/` 下生成 model/query/repo

### 步骤7：编写核心业务代码

按标准分层：
- `internal/cfg/mysql.go` — 数据库初始化
- `internal/cfg/redis.go` — Redis 初始化
- `internal/errs/code.go` — 错误码定义
- `internal/ser/*.go` — 业务逻辑（Service层）
- `internal/rep/*.go` — 数据访问（Repository层）
- `internal/cache/*.go` — 缓存操作
- `handler.go` — RPC Handler（薄封装，转发到 Service）
- `main.go` — 服务入口

### 步骤8：注册网关路由

**B端路由 — gate-back：**

创建 `gate-back/biz/handler/wallet/wallet.go`：
```go
package wallet

func PageCurrency(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.WalletClient().PageCurrency)
}
// ... 其他 B端 handler
```

创建 `gate-back/biz/router/wallet/wallet_router.go`：
```go
func RouterWallet(r *server.Hertz) {
    my_router.Api.POST("/wallet/currency/page", wallet.PageCurrency)
    my_router.Api.POST("/wallet/currency/edit", wallet.EditCurrency)
    // ...
}
```

在 `gate-back/biz/router/register.go` 添加：
```go
wallet.RouterWallet(r)
```

**C端路由 — gate-font：** 同理。

### 步骤9：配置 ETCD

在 ETCD 中写入：
```
/slg/serv/wallet/port  → "8022"（或其他可用端口）
/slg/serv/wallet/db    → "ser_wallet"（数据库名）
```

### 步骤10：更新 go.work

**编辑：** `/Users/mac/gitlab/go.work`

```go
use (
    // ... 已有模块
    ./ser-wallet
)
```

### 步骤11：接口自测 → 业务闭环验证

---

## 九、必须复用、禁止重复造轮子的清单

| 场景 | 使用 | 不要自己写 |
|------|------|-----------|
| 日志 | `lg.Log().CtxInfof(ctx, ...)` | 不要用 fmt.Println、log.Println |
| 错误返回 | `ret.BizErr(ctx, code, msg)` | 不要自己构造 error |
| HTTP响应 | `ret.HttpOne(ctx, data, i18n, err)` | 不要自己构造 JSON |
| RPC转发 | `rpc.Handler(ctx, c, client.Method)` | 不要手动 Bind+Call+Response |
| 类型转换 | `utils.S2Int()`, `S2Int64()`, `S2Float64()` | 不要自己 strconv |
| JSON序列化 | `utils.MarshalStr()`, `UnmarshalStr()` | 使用 sonic 高性能 |
| 分布式ID | `snowflake.Generate()` | 不要自己生成 |
| Redis | `rds.Set/Get/Del/HSet/ZAdd` | 不要直接用 go-redis |
| 数据库 | `cfg.InitMySQL()` + query.Q | 不要自己 sql.Open |
| ETCD配置 | `etcd.DirectGet()`, `etcd.Get()` | 不要自己连接 ETCD |
| 用户信息 | `tracer.GetUserId(ctx)`, `GetUserName(ctx)` | 不要自己解析 Token |
| 分页查询 | `repo.GetPageByConds()` 或 `query.FindByPage()` | 不要手动拼 LIMIT OFFSET |
| 操作日志 | `rpc.BLogClient().CreateLog()` | 不要自己记日志表 |
| 文件上传 | `rpc.S3Client()` | 不要自己对接 S3 |
| KYC查询 | `rpc.KycClient()` | 已有 8 个 RPC 方法 |
| 用户信息 | `rpc.UserClient()` | 已有 35 个 RPC 方法 |
| 校验 | `validator.CheckPhone/Email()` | 不要自己写正则 |
| 加密 | `utils.EncryptedString` | GORM 自动加解密 |

---

## 十、关键架构模式总结

### 10.1 分层架构

```
Gateway Handler    →  纯转发，无业务逻辑
  ↓
Service Handler    →  handler.go，转发到 Service 层
  ↓
Service Layer      →  internal/ser/，核心业务逻辑
  ↓
Repository Layer   →  internal/rep/，数据库操作
  ↓
Generated Layer    →  internal/gorm_gen/，自动生成的 model/query/repo
```

### 10.2 通信模式

```
同步链路：  Client → Gateway(Hertz) → Service(Kitex) → DB/Redis
异步链路：  Service → Kafka/NATS → Consumer Service
回调链路：  三方 → Gateway → Service → 钱包RPC
```

### 10.3 安全模式

```
C端：IP黑名单 → Token校验 → 用户ID注入
B端：IP白名单 → Token校验 → URL权限校验 → 管理员ID注入
```

### 10.4 配置管理模式

```
所有配置集中在 ETCD（/slg/ 前缀下）
服务启动时从 ETCD 读取 → 缓存到内存 → Watch 变更自动更新
数据库/Redis/S3 地址都不硬编码
```

### 10.5 错误处理模式

```
Service 层   →  ret.BizErr(ctx, errCode, "message")
Handler 层   →  自动传播 BizErr
Gateway 层   →  ret.HttpOne() 自动转换为 HTTP 响应 + i18n 翻译
```
