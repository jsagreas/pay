# 工程认知文档 — 从0到1开发 ser-wallet 所需的全部工程知识

> 基于对 common、gate-back、gate-font、ser-app 四个模块的深度代码分析
> 所有结论均有工程代码实证，非推测

---

## 目录

- [第一章 项目工程总览](#第一章-项目工程总览)
- [第二章 技术栈全景](#第二章-技术栈全景)
- [第三章 common 共享模块深度解析](#第三章-common-共享模块深度解析)
- [第四章 gate-back B端网关深度解析](#第四章-gate-back-b端网关深度解析)
- [第五章 gate-font C端网关深度解析](#第五章-gate-font-c端网关深度解析)
- [第六章 ser-app 业务服务模块范本解析](#第六章-ser-app-业务服务模块范本解析)
- [第七章 从0到1开发 ser-wallet 完整指南](#第七章-从0到1开发-ser-wallet-完整指南)
- [附录A 工程中所有服务编号表](#附录a-工程中所有服务编号表)
- [附录B common/pkg 工具包速查表](#附录b-commonpkg-工具包速查表)
- [附录C gate-back 与 gate-font 差异对照表](#附录c-gate-back-与-gate-font-差异对照表)

---

## 第一章 项目工程总览

### 1.1 Go Workspace 架构

项目采用 Go Workspace（go.work）管理多模块，根目录 `/Users/mac/gitlab/` 下包含 12+ 个独立模块：

```
/Users/mac/gitlab/
├── go.work                  # Go Workspace 配置
├── common/                  # 共享模块（IDL、pkg、rpc、script）
├── gate-back/               # B端网关（运营后台 HTTP 入口）
├── gate-font/               # C端网关（用户端 HTTP 入口）
├── ser-app/                 # APP配置服务（Banner、分类）
├── ser-bcom/                # 通用配置服务
├── ser-blog/                # 操作日志服务
├── ser-buser/               # 后台用户服务
├── ser-ip/                  # IP黑白名单服务
├── ser-kyc/                 # KYC实名认证服务
├── ser-live/                # 直播服务
├── ser-user/                # C端用户服务
├── ser-cron/                # 定时任务服务
├── ser-mall/                # 商城服务
└── ...                      # 其他服务
```

### 1.2 请求流转全景

```
外部请求 (HTTP)
    │
    ├─ B端请求 → gate-back (Hertz) → 中间件链(白名单+Token+URL权限) → RPC → ser-xxx (Kitex)
    │
    └─ C端请求 → gate-font (Hertz) → 中间件链(黑名单+Token) → RPC → ser-xxx (Kitex)

所有 ser-xxx 服务之间也可以通过 RPC 互相调用
```

### 1.3 核心设计原则

1. **IDL先行**：所有接口定义在 Thrift IDL 中，由 gen_kitex.sh 生成 RPC 代码
2. **网关转发**：gateway 只做 HTTP→RPC 转发 + 鉴权，不含业务逻辑
3. **服务独立**：每个 ser-xxx 拥有独立的数据库、错误码空间、internal 目录
4. **共享复用**：通过 common 模块复用 IDL、RPC客户端、中间件、工具包
5. **ETCD驱动**：所有配置（端口、数据库、Redis等）从 ETCD 读取，无本地配置文件
6. **GORM Gen**：数据库模型和查询代码由 gorm_gen.go 自动生成

---

## 第二章 技术栈全景

| 类别 | 技术 | 用途 |
|-----|------|------|
| HTTP框架 | CloudWeGo Hertz | 网关层 HTTP 服务 |
| RPC框架 | CloudWeGo Kitex | 服务间 RPC 通信 |
| IDL | Apache Thrift | 接口定义语言 |
| 数据库 | TiDB (兼容MySQL) | 关系型数据存储 |
| ORM | GORM + GORM Gen | 数据库访问层 |
| 缓存 | Redis | 会话、缓存、枚举 |
| 配置中心 | ETCD v3 | 服务配置 + 服务发现 + 服务注册 |
| 消息队列 | Kafka + NATS | 异步消息（双支持） |
| 对象存储 | AWS S3 | 文件上传/下载 |
| 搜索引擎 | Elasticsearch | 日志/搜索 |
| ID生成 | Snowflake | 分布式唯一ID |
| 国际化 | 自建 i18n 服务 | 错误码多语言翻译 |
| 链路追踪 | 自建 Tracer | X-Trace-Id 贯穿全链路 |

---

## 第三章 common 共享模块深度解析

### 3.1 目录结构

```
/Users/mac/gitlab/common/
├── idl/                         # Thrift IDL 定义（所有服务的接口契约）
│   ├── ser-app/service.thrift
│   ├── ser-buser/service.thrift
│   ├── ser-user/service.thrift
│   ├── ser-kyc/service.thrift
│   ├── ser-live/service.thrift
│   ├── ser-blog/service.thrift
│   └── ...                      # 每个服务一个目录
├── pkg/                         # 共享工具包
│   ├── consts/                  # 常量定义
│   │   ├── namesp/namesp.go    # 服务命名空间 + ETCD键
│   │   ├── redis_key/          # Redis键常量
│   │   ├── enums/              # 通用枚举
│   │   └── env/                # 环境变量键名
│   ├── errs/                    # 通用错误码
│   ├── ret/                     # 统一返回值结构
│   ├── tracer/                  # 链路追踪
│   ├── middleware/              # 中间件（鉴权、日志、IP校验等）
│   ├── session/                 # Token载体结构
│   ├── etcd/                    # ETCD客户端封装
│   ├── db/                      # MySQL/GORM封装
│   ├── rds/                     # Redis封装
│   ├── lg/                      # 日志封装
│   ├── mys3/                    # S3客户端封装
│   ├── mq/                      # 消息队列封装
│   ├── es/                      # Elasticsearch封装
│   ├── myid/                    # Snowflake ID生成器
│   ├── utils/                   # 通用工具函数
│   └── i18n/                    # 多语言客户端
├── rpc/                         # RPC 相关
│   ├── rpc_client.go            # RPC客户端工厂（20+服务的单例工厂）
│   ├── init.go                  # RPC初始化参数（服务端+客户端）
│   └── handler.go               # 通用Handler（泛型HTTP→RPC转发）
├── kitex_gen/                   # Kitex自动生成的代码（勿手编）
└── script/
    └── gen_kitex.sh             # IDL代码生成脚本
```

### 3.2 IDL 规范详解

**文件位置约定**：`common/idl/<ser-name>/service.thrift`

**命名规范**：
- 服务名：PascalCase，如 `AppService`、`UserService`
- 方法名：PascalCase，`<动词><资源>`，如 `CreateBanner`、`PageUser`
- 请求体：`<方法名>Req`，如 `CreateBannerReq`
- 响应体：`<方法名>Resp`，如 `CreateBannerResp`

**IDL 示例**（基于 ser-app 的 service.thrift）：

```thrift
namespace go ser_app

// 请求结构体：字段带序号和 api.body 注解
struct CreateBannerReq {
    1: required string bannerName   (api.body="bannerName")
    2: required string pagePosition (api.body="pagePosition")
    3: required i32 jumpType        (api.body="jumpType")
    4: optional string jumpTarget   (api.body="jumpTarget")
    5: required i32 sortValue       (api.body="sortValue")
    6: required string terminal     (api.body="terminal")
    7: optional i64 onlineTime      (api.body="onlineTime")
    8: required list<BannerImage> images (api.body="images")
    9: required string defaultLang  (api.body="defaultLang")
}

// 响应结构体
struct CreateBannerResp {
    1: i64 id
}

// 分页请求（通用模式）
struct PageBannerReq {
    1: required string pagePosition (api.body="pagePosition")
    2: optional string bannerName   (api.body="bannerName")
    3: optional i32 status          (api.body="status")
    4: required i32 pageNo          (api.body="pageNo")
    5: required i32 pageSize        (api.body="pageSize")
}

// 服务定义
service AppService {
    CreateBannerResp CreateBanner(1: CreateBannerReq req)
    UpdateBannerResp UpdateBanner(1: UpdateBannerReq req)
    DeleteBannerResp DeleteBanner(1: DeleteBannerReq req)
    StatusBannerResp StatusBanner(1: StatusBannerReq req)
    PageBannerResp PageBanner(1: PageBannerReq req)
    DetailBannerResp DetailBanner(1: DetailBannerReq req)
    ListBannerResp ListBanner(1: ListBannerReq req)
}
```

**IDL 关键规则**：
- 字段类型：`i32`（int32）、`i64`（int64）、`string`、`bool`、`double`、`list<T>`、`map<K,V>`
- `required` vs `optional`：影响生成的 Go 代码中是否为指针类型
- `api.body` 注解：指定 JSON 字段名（用于 Hertz 自动绑定）
- 通用引用：可用 `include "common.thrift"` 引入公共结构体

### 3.3 gen_kitex.sh 脚本详解

**文件位置**：`/Users/mac/gitlab/common/script/gen_kitex.sh`

**工作原理**：遍历 `common/idl/` 下的所有服务目录，对每个服务执行 Kitex 代码生成命令。

**核心命令**：

```bash
kitex -module common/pkg \
      -I <idl_dir> \
      -service <service_name> \
      -gen-path ./kitex_gen/ \
      <thrift_file>
```

**参数说明**：
- `-module common/pkg`：Go module 名称
- `-I <idl_dir>`：IDL 文件的搜索路径
- `-service <name>`：服务名
- `-gen-path ./kitex_gen/`：生成代码的输出目录
- `<thrift_file>`：入口 Thrift 文件

**生成产物**（位于 `common/kitex_gen/`）：
- `ser_app/` — 请求/响应结构体定义（Go struct）
- `ser_app/appservice/` — 客户端和服务端接口
  - `client.go` — RPC 客户端接口
  - `server.go` — RPC 服务端接口
  - `appservice.go` — 服务方法签名

**使用流程**：
1. 编写/修改 IDL 文件
2. 在 common 根目录执行 `bash script/gen_kitex.sh`
3. 生成的代码在 `kitex_gen/` 目录下，所有服务模块通过 `common/kitex_gen/...` 路径引用

### 3.4 RPC 客户端工厂

**文件位置**：`/Users/mac/gitlab/common/rpc/rpc_client.go`

**设计模式**：`sync.Once` + 延迟初始化 + 单例

```go
// 每个服务都有独立的全局变量和 Once
var (
    appService *appservice.Client
    appOnce    sync.Once
)

// 工厂函数：首次调用时初始化，后续直接返回
func AppClient() appservice.Client {
    appOnce.Do(func() {
        app, err := appservice.NewClient(
            namesp.EtcdAppService,      // ETCD注册名："app_service"
            InitRpcClientParams()...,   // 客户端参数
        )
        appService = &app
        if err != nil {
            lg.Log().CtxErrorf(context.Background(), "RPC 创建ser-app客户端错误 %s", err)
        }
    })
    return *appService
}
```

**当前已注册的客户端**（20+个）：
- `BUserClient()` — 后台用户服务
- `UserClient()` — C端用户服务
- `AppClient()` — APP配置服务
- `BComClient()` — 通用配置服务
- `I18nClient()` — 多语言服务
- `IpClient()` — IP服务
- `BLogClient()` — 操作日志服务
- `S3Client()` — S3文件服务
- `KycClient()` — KYC服务
- `LiveClient()` — 直播服务
- `ShortVideosClient()` — 短视频服务
- `RtcClient()` — RTC服务
- `ExpClient()` — VIP经验服务
- `ItemClient()` — 道具服务
- `ContentCenterClient()` — 内容中心服务
- `RecommendClient()` — 推荐服务
- `WebSocketClient()` — WebSocket服务
- `CronClient()` — 定时任务服务
- `MallClient()` — 商城服务

### 3.5 RPC 初始化参数

**客户端参数**（`InitRpcClientParams()`）：

```go
func InitRpcClientParams() []client.Option {
    r, _ := etcd.NewEtcdResolver(myetcd.Address())
    return []client.Option{
        client.WithResolver(r),                           // ETCD服务发现
        client.WithRPCTimeout(3 * time.Second),           // 3秒超时
        client.WithMuxConnection(1),                      // 连接复用
        client.WithMetaHandler(transmeta.ClientTTHeaderHandler), // 链路追踪
        client.WithTransportProtocol(transport.TTHeader),  // TTHeader协议
    }
}
```

**服务端参数**（`InitRpcServerParams(serviceName, port)`）：

```go
func InitRpcServerParams(serviceName string, port int) []server.Option {
    addr := utils.GetLocalIPv4() + ":" + strconv.Itoa(port)
    r, _ := etcd.NewEtcdRegistry(myetcd.Address())
    return []server.Option{
        server.WithServiceAddr(&net.TCPAddr{Port: port}),
        server.WithRegistry(r),                          // ETCD服务注册
        server.WithRegistryInfo(&registry.Info{
            ServiceName: serviceName,
            Addr:        utils.NewNetAddr("tcp", addr),
        }),
        server.WithMuxTransport(),                       // 多路复用传输
        server.WithMetaHandler(transmeta.ServerTTHeaderHandler), // 链路追踪
    }
}
```

### 3.6 通用 Handler（泛型 HTTP→RPC 转发）

**文件位置**：`/Users/mac/gitlab/common/rpc/handler.go`

```go
// 有参数的通用转发
func Handler[REQ any, RESP any](
    ctx context.Context,
    c *app.RequestContext,
    rpcMethod func(context.Context, REQ, ...callopt.Option) (RESP, error),
) {
    var reqData REQ
    var bindTarget any
    val := reflect.ValueOf(&reqData).Elem()
    if val.Kind() == reflect.Ptr {
        val.Set(reflect.New(val.Type().Elem()))
        bindTarget = reqData
    } else {
        bindTarget = &reqData
    }
    err := c.BindAndValidate(bindTarget)   // 自动绑定JSON→struct
    if err != nil {
        c.JSON(ret.HttpOne(ctx, nil, I18nClient(), err))
        return
    }
    resp, err := rpcMethod(ctx, reqData)    // 调用RPC
    c.JSON(ret.HttpOne(ctx, resp, I18nClient(), err)) // 统一返回
}

// 无参数的通用转发
func HandlerNoParam[RESP any](...) { ... }

// CSV导出的通用转发
func HandlerWithCSV[REQ any, RESP any](...) { ... }
```

**使用方式**（在网关Handler中）：

```go
// 一行代码完成：参数绑定 → RPC调用 → 响应返回
func CreateBanner(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.AppClient().CreateBanner)
}
```

### 3.7 统一返回值结构

**文件位置**：`/Users/mac/gitlab/common/pkg/ret/`

**返回结构体**：

```go
type R struct {
    TraceId string      `json:"traceId"`  // 链路追踪ID
    Code    int         `json:"code"`     // 业务码（0=成功）
    Msg     string      `json:"msg"`      // 提示信息
    Data    interface{} `json:"data"`     // 业务数据
}
```

**常用函数**：

```go
// 成功返回（HTTP 200）
ret.HttpOk(ctx, data, i18nClient)
// → { "traceId": "xxx", "code": 0, "msg": "Success", "data": {...} }

// 错误返回（HTTP 200，code != 0）
ret.HttpErr(ctx, err, i18nClient)
// → { "traceId": "xxx", "code": 6011001, "msg": "Banner名称已存在", "data": null }

// 统一处理（自动判断err是否为nil）
ret.HttpOne(ctx, data, i18nClient, err)

// 业务错误包装
ret.BizErr(ctx, errs.BannerNameAlreadyExist, "Banner名称已存在")
// → 生成一个带错误码的error对象，经i18n翻译后返回多语言文案
```

### 3.8 错误码体系

**编码规则**：`6{服务编号3位}{模块编号+序号3位}`

**基础码**：6000000

**示例**：
- `6002xxx` — ser-buser（编号002）
- `6008xxx` — ser-user（编号008）
- `6011xxx` — ser-app（编号011）
- `6012xxx` — ser-kyc（编号012）

**分段约定**（以ser-app为例）：
- `6011000 ~ 6011099` — Banner模块错误
- `6011100 ~ 6011199` — HomeCategory模块错误
- `6011200 ~ 6011299` — ContentStrategy模块错误

**定义方式**（使用 iota）：

```go
// 文件：internal/errs/code.go
const (
    BannerBaseError           = 6011000 + iota
    BannerNameAlreadyExist    // 6011001
    BannerNotFound            // 6011002
    BannerStatusNotDisabled   // 6011003
    BannerEnabledCannotDelete // 6011004
    ReqParamErr               // 6011005
)
```

### 3.9 链路追踪系统

**文件位置**：`/Users/mac/gitlab/common/pkg/tracer/`

**核心 Header**：

| Header | 变量名 | 说明 |
|--------|--------|------|
| `X-Trace-Id` | `TraceIDHeader` | 全链路追踪ID（自动生成） |
| `X-User-ID` | `UserIdHeader` | 用户ID（鉴权中间件注入） |
| `X-User-Name` | `UserNameHeader` | 用户名（鉴权中间件注入） |
| `X-User-Token` | `UserTokenHeader` | 用户Token |
| `X-Client-IP` | `ClientIPHeader` | 客户端IP |
| `X-User-Language` | `UserLangHeader` | 用户语言偏好 |
| `X-User-Device-ID` | `UserDeviceIDHeader` | 设备标识 |
| `X-User-Device-Type` | `UserDeviceTypeHeader` | 设备类型 |

**获取方式**（在服务代码中）：

```go
tracer.GetTraceID(ctx)   // 获取追踪ID
tracer.GetUserId(ctx)    // 获取用户ID（int64）
tracer.GetUserName(ctx)  // 获取用户名
tracer.GetUserToken(ctx) // 获取Token
tracer.GetUserLang(ctx)  // 获取语言
```

**传递机制**：
- HTTP请求 → `SetHeaderInfo()` 中间件提取到 context
- context → `metainfo.WithPersistentValue()` 注入
- RPC调用 → `transmeta.ClientTTHeaderHandler` 自动透传到下游服务

### 3.10 ETCD 配置管理

**ETCD地址**：从环境变量 `ETCD_ADDR` 读取（逗号分隔多地址）

**配置键规范**：

| 键前缀 | 说明 | 示例 |
|-------|------|------|
| `/slg/serv/<module>/port` | 服务端口 | `/slg/serv/app/port` → `9091` |
| `/slg/serv/<module>/db` | 数据库名 | `/slg/serv/app/db` → `db_app` |
| `/slg/conf/redis/addr` | Redis地址 | `redis:6379` |
| `/slg/conf/redis/password` | Redis密码 | |
| `/slg/conf/tidb/addr` | TiDB地址 | `tidb:4000` |
| `/slg/conf/tidb/username` | TiDB用户名 | |
| `/slg/conf/tidb/password` | TiDB密码 | |
| `/slg/conf/s3/ak` | S3 AccessKey | |
| `/slg/conf/s3/sk` | S3 SecretKey | |
| `/slg/conf/s3/endpoint` | S3端点 | |
| `/slg/conf/s3/bucket` | S3桶名 | |
| `/slg/conf/es/addr` | ES地址 | |
| `/slg/conf/kafka/addr` | Kafka地址 | |
| `/slg/conf/nats/addr` | NATS地址 | |

**配置加载与监听**：

```go
// 加载前缀下所有配置 + 自动监听变更
etcd.LoadAndWatch("/slg/conf/")

// 读取配置（从内存缓存读取，无网络开销）
value := etcd.Get(ctx, "/slg/conf/redis/addr")

// 直接从ETCD读取（有网络开销，用于初始化）
value := etcd.DirectGet(ctx, "/slg/serv/app/port")
```

**热更新机制**：
- `LoadAndWatch()` 启动后台协程监听 ETCD Watch 事件
- 配置变更时通过 Copy-On-Write 更新本地缓存（`sync.Map`）
- 支持注册 Hook 回调函数，配置变更时触发

### 3.11 服务注册与发现

**注册名格式**：`<module>_service`

**注册流程**（服务启动时）：
1. 服务读取端口号（ETCD或命令行参数）
2. 获取本机 IPv4 地址
3. 创建 ETCD 注册中心实例
4. 通过 Kitex Server Options 自动注册

**发现流程**（客户端调用时）：
1. RPC 客户端初始化时传入 `client.WithResolver(etcdResolver)`
2. Kitex 自动从 ETCD 解析目标服务的地址列表
3. 内置负载均衡选择具体实例

### 3.12 中间件体系

**文件位置**：`/Users/mac/gitlab/common/pkg/middleware/`

| 中间件 | 函数名 | 用于 | 功能 |
|-------|--------|------|------|
| 日志 | `HertzAccLogStr()` | 全局 | 打印请求日志（含trace_id） |
| Header | `SetHeaderInfo()` | 全局 | 提取HTTP头→context |
| IP黑名单 | `BlackCheck()` | gate-font | RPC调用ser-ip检查黑名单 |
| IP白名单 | `WhiteCheck()` | gate-back | RPC调用ser-ip检查白名单 |
| C端鉴权 | `FontAuthMiddleware()` | gate-font /api | Redis验证Token→注入用户信息 |
| B端鉴权 | `BackAuthMiddleware()` | gate-back /admin/api | Redis验证Token+URL权限校验 |
| S3代理 | `RedirectS3()` | 两个网关 /s3 | S3文件重定向（预签名URL） |
| Redis初始化 | `InitRedis()` | 网关启动 | 初始化Redis连接 |

### 3.13 Session/Token 结构

**C端Token载体**（`session.CUserTokenPayload`）：

```go
type CUserTokenPayload struct {
    ID            int64  `json:"id"`               // 用户ID
    UID           int64  `json:"uid"`              // 业务UID
    LastLoginIp   string `json:"last_login_ip"`    // 上次登录IP
    LastLoginTime int64  `json:"last_login_time"`  // 上次登录时间
    Device        string `json:"device"`           // 设备标识
    DeviceType    string `json:"deviceType"`       // 设备类型
    NickName      string `json:"nick_name"`        // 昵称
}
// Redis Key: "user:token:{token_value}"
```

**B端Token载体**（`session.BUserTokenPayload`）：

```go
type BUserTokenPayload struct {
    ID       int64  `json:"id"`        // 后台用户ID
    UserName string `json:"userName"`  // 用户名
}
// Redis Key: 直接用 token 值作为 key（无前缀）
```

### 3.14 数据库（GORM + TiDB）

**初始化模式**（`sync.Once` 单例）：

```go
var (
    db   *gorm.DB
    once sync.Once
)

func InitMySQL() *gorm.DB {
    once.Do(func() {
        opt := &mysql.InitOption{
            Addr:     etcd.DirectGet(ctx, env.TidbAddr) + ":" + etcd.DirectGet(ctx, env.TidbPort),
            Username: etcd.DirectGet(ctx, env.TidbUsername),
            Password: etcd.DirectGet(ctx, env.TidbPassword),
            Database: etcd.DirectGet(ctx, namesp.EtcdXxxDb),
        }
        mysql.Init(opt)
        db = mysql.DB
        query.SetDefault(db)  // 设置GORM Gen全局查询对象
    })
    return db
}
```

### 3.15 Redis

**初始化**：

```go
rds.Init(rds.InitOption{
    Addr:     etcd.DirectGet(ctx, env.RedisAddr),
    Password: etcd.DirectGet(ctx, env.RedisPassword),
})
```

**常用操作**：

```go
rds.Get(key)                    // 获取
rds.Set(ctx, key, value, ttl)   // 设置
rds.Del(keys...)                // 删除
rds.HGet(hashKey, field)        // Hash获取
```

### 3.16 消息队列、S3、Snowflake

**消息队列**：Kafka + NATS 双支持，位于 `common/pkg/mq/`

**S3 工具**：
```go
mys3.ToS3Key(url)                       // URL转S3相对路径
mys3.JoinUrl(enums.GateTypeBack, s3Key) // 拼接完整URL
```

**Snowflake ID**：
```go
myid.GenID()  // 生成分布式唯一ID
```

---

## 第四章 gate-back B端网关深度解析

### 4.1 目录结构

```
/Users/mac/gitlab/gate-back/
├── main.go              # 入口（初始化ETCD/Redis/S3，创建Hertz服务，注册路由）
├── router.go            # customizedRegister（自定义路由）
├── router_gen.go        # 自动生成的路由汇总入口
├── biz/
│   ├── handler/         # HTTP请求处理层（按服务分包）
│   │   ├── buser/       # 后台用户管理
│   │   ├── bcom/        # 通用配置
│   │   ├── blog/        # 操作日志
│   │   ├── app/         # Banner/分类
│   │   ├── kyc/         # KYC审核
│   │   ├── i18n/        # 多语言
│   │   ├── item/        # 道具
│   │   ├── exp/         # VIP经验
│   │   ├── user/        # 用户查询
│   │   ├── live/        # 直播管理
│   │   ├── rtc/         # RTC
│   │   ├── s3/          # 文件上传
│   │   ├── shortvideos/ # 短视频审核
│   │   └── ping.go      # 健康检查
│   └── router/          # 路由注册层（按服务分包）
│       ├── my_router/   # 路由组定义（Open/Api/S3）
│       ├── buser/       # 后台用户路由
│       ├── bcom/        # 通用配置路由
│       ├── blog/        # 日志路由
│       ├── ...          # 其他服务路由
│       └── register.go  # GeneratedRegister 总注册
```

### 4.2 main.go 启动流程

```go
func main() {
    lg.InitHLog()                                    // 1. 初始化日志
    etcd.LoadAndWatch("/slg/conf/")                  // 2. 加载ETCD配置
    middleware.InitRedis()                            // 3. 初始化Redis
    mys3.Init(...)                                   // 4. 初始化S3

    port := flag.String("port",                      // 5. 获取端口
        etcd.DirectGet(ctx, namesp.EtcdGateBackPort), "端口号")
    flag.Parse()

    addr := utils.GetLocalIPv4() + ":" + *port       // 6. 获取本机IP
    r, _ := etcdhz.NewEtcdRegistry(etcd.Address())   // 7. 创建注册中心

    h := server.Default(                              // 8. 创建Hertz服务
        server.WithHostPorts("0.0.0.0:"+*port),
        server.WithRegistry(r, &registry.Info{
            ServiceName: namesp.EtcdGateBack,         //    注册名 "gate_back"
            Addr:        hutils.NewNetAddr("tcp", addr),
        }),
        server.WithMaxRequestBodySize(20<<20),        //    请求体限制20MB
        server.WithTracer(tracer.IGlTracer{}),        //    链路追踪
    )

    h.Use(middleware.HertzAccLogStr())                // 9. 全局日志中间件
    my_router.InitMyRouter(h)                         // 10. 初始化路由组
    register(h)                                       // 11. 注册所有路由
    h.Spin()                                          // 12. 启动服务
}
```

### 4.3 路由组定义

```go
// my_router.go
func InitMyRouter(h *server.Hertz) {
    // 无需鉴权的开放接口
    Open = h.Group("/admin/open",
        middleware.SetHeaderInfo(),
        middleware.WhiteCheck())       // IP白名单

    // 需要鉴权的管理接口
    Api = h.Group("/admin/api",
        middleware.SetHeaderInfo(),
        middleware.WhiteCheck(),       // IP白名单
        middleware.BackAuthMiddleware()) // B端Token+URL权限

    // S3文件代理
    s3Group := h.Group("/s3")
    s3Group.GET("/*path", middleware.RedirectS3())
}
```

### 4.4 路由命名规范

**前缀**：
- `/admin/open/` — 无需鉴权（如登录）
- `/admin/api/` — 需要鉴权（如增删改查）

**路由格式**：`/admin/api/<服务>/<资源>/<动作>`

**示例**：

```
POST /admin/open/buser/user/login       # 登录（开放）
POST /admin/api/buser/user/create       # 创建用户（需鉴权）
POST /admin/api/buser/user/page         # 分页用户（需鉴权）
POST /admin/api/buser/role/create       # 创建角色
POST /admin/api/app/banner/create       # 创建Banner
POST /admin/api/kyc/page                # KYC分页
POST /admin/api/live/admin/live/rooms   # 直播房间列表
```

### 4.5 中间件执行链

```
HTTP请求
  → HertzAccLogStr()          # 全局：打印 <trace_id=xxx> 200 - 45ms POST /path
  → SetHeaderInfo()           # 路由组：提取IP/Token/Lang/Device到context
  → WhiteCheck()              # 路由组：RPC调ser-ip检查IP白名单（不在白名单→拒绝）
  → BackAuthMiddleware()      # 路由组（仅Api）：
      ├ 检查Token存在性
      ├ Redis验证Token有效性
      ├ 解析BUserTokenPayload
      ├ 注入UserID/UserName到context
      └ RPC调ser-buser.CheckUrlAuth()检查URL权限
  → Handler处理
```

### 4.6 Handler 实现模式

**模式1：通用转发（绝大多数接口）**

```go
func CreateUser(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.BUserClient().UserCreate)
}
```

**模式2：自定义处理（需要额外逻辑）**

```go
func Login(ctx context.Context, c *app.RequestContext) {
    req := &ser_buser.LoginReq{}
    err := c.BindAndValidate(req)
    if err != nil {
        c.JSON(ret.HttpOne(ctx, nil, rpc.I18nClient(), err))
        return
    }
    resp, err := rpc.BUserClient().Login(ctx, req)
    c.JSON(ret.HttpOne(ctx, resp, rpc.I18nClient(), err))
}
```

**模式3：文件上传（multipart/form-data）**

```go
func UploadFile(ctx context.Context, c *app.RequestContext) {
    reqData := &ser_s3.UploadFileReq{}
    c.BindAndValidate(reqData)
    fileHeader, _ := c.FormFile("content")
    file, _ := fileHeader.Open()
    defer file.Close()
    content, _ := io.ReadAll(file)
    reqData.Content = content
    resp, err := rpc.S3Client().UploadFile(ctx, reqData)
    c.JSON(ret.HttpOne(ctx, resp, rpc.I18nClient(), err))
}
```

### 4.7 路由注册（新增服务）

在 `biz/router/register.go` 的 `GeneratedRegister()` 函数中添加调用：

```go
func GeneratedRegister(r *server.Hertz) {
    buser.RouterRole(r)
    bcom.RouterRole(r)
    i18n.RouterI18n(r)
    // ... 现有服务
    wallet.RouterWallet(r)  // ← 新增
}
```

---

## 第五章 gate-font C端网关深度解析

### 5.1 目录结构

与 gate-back 结构一致，区别在于服务包不同：

```
/Users/mac/gitlab/gate-font/
├── main.go
├── router.go
├── router_gen.go
├── biz/
│   ├── handler/
│   │   ├── user/          # C端用户（34个函数）
│   │   ├── app/           # Banner、分类、策略
│   │   ├── kyc/           # KYC认证（4个函数）
│   │   ├── shortvideos/   # 短视频（15个函数）
│   │   ├── rtc/           # RTC（2个函数）
│   │   ├── live/          # 直播（16个函数）
│   │   └── ping.go
│   └── router/
│       ├── my_router/
│       ├── user/
│       ├── app/
│       ├── kyc/
│       ├── shortvideos/
│       ├── rtc/
│       ├── live/
│       ├── enum/
│       └── register.go
```

### 5.2 与 gate-back 的关键区别

| 对比项 | gate-font (C端) | gate-back (B端) |
|-------|-----------------|-----------------|
| 路由前缀 | `/open`、`/api` | `/admin/open`、`/admin/api` |
| IP策略 | `BlackCheck()`（黑名单，默认放行） | `WhiteCheck()`（白名单，默认拒绝） |
| 鉴权中间件 | `FontAuthMiddleware()` | `BackAuthMiddleware()` |
| Token结构 | `CUserTokenPayload`（含昵称、设备） | `BUserTokenPayload`（仅ID、用户名） |
| Token Redis Key | `"user:token:" + token`（有前缀） | 直接用token值（无前缀） |
| URL权限 | 无额外校验 | `CheckUrlAuth()` 校验URL级权限 |
| 请求体限制 | 默认 | 20MB |
| 服务名 | `gate_font` | `gate_back` |
| 注册的服务 | user, app, kyc, shortvideos, rtc, live | buser, bcom, blog, i18n, kyc, item, app, exp, s3, live, shortvideos, rtc, user |

### 5.3 C端路由组定义

```go
func InitMyRouter(h *server.Hertz) {
    Open = h.Group("/open",
        middleware.SetHeaderInfo(),
        middleware.BlackCheck())       // IP黑名单（与B端白名单相反）

    Api = h.Group("/api",
        middleware.SetHeaderInfo(),
        middleware.BlackCheck(),
        middleware.FontAuthMiddleware()) // C端Token鉴权

    s3Group := h.Group("/s3")
    s3Group.GET("/*path", middleware.RedirectS3())
}
```

### 5.4 C端鉴权流程

```
FontAuthMiddleware():
  1. tracer.GetUserToken(ctx) → 获取Token
  2. Token为空 → 返回401 TokenNotExist
  3. rds.Get("user:token:" + token) → Redis查询
  4. Redis为空 → 返回401 TokenAnalysisFailed
  5. json.Unmarshal → CUserTokenPayload
  6. base.ID == 0 → 返回401
  7. 注入 X-User-ID、X-User-Name 到 context
  8. c.Next(ctx) → 继续
```

### 5.5 C端路由示例

```
# 无需鉴权（/open）
POST /open/user/login              # 登录
POST /open/user/register           # 注册
POST /open/user/loginGuest         # 游客登录
POST /open/app/banner/list         # Banner列表
POST /open/app/category/list       # 分类列表
POST /open/enum/options            # 枚举选项

# 需要鉴权（/api）
POST /api/user/getUserDetail       # 获取用户信息
POST /api/user/updateUserInfo      # 更新用户信息
POST /api/user/avatar/upload       # 上传头像
POST /api/kyc/submit               # 提交KYC
POST /api/live/api/room/start      # 开始直播
POST /api/shortvideos/publish      # 发布短视频
```

---

## 第六章 ser-app 业务服务模块范本解析

> ser-app 是项目中最完整的业务服务模块，可作为开发 ser-wallet 的参考范本

### 6.1 目录结构

```
/Users/mac/gitlab/ser-app/
├── main.go                          # 服务入口
├── handler.go                       # RPC Handler（接口实现委托层）
├── go.mod
├── internal/
│   ├── cfg/                         # 配置初始化
│   │   ├── mysql.go                 # MySQL连接（sync.Once）
│   │   └── redis.go                 # Redis连接（sync.Once）
│   ├── errs/                        # 错误码定义
│   │   └── code.go                  # 6011xxx
│   ├── enum/                        # 枚举和常量
│   │   ├── banner.go                # Banner枚举
│   │   ├── category.go              # 分类枚举
│   │   └── enum_registry.go         # 枚举注册表（支持缓存+i18n）
│   ├── gen/                         # 代码生成入口
│   │   └── gorm_gen.go              # GORM模型生成工具
│   ├── gorm_gen/                    # 自动生成（勿手编）
│   │   ├── model/                   # 数据模型（banner.gen.go等）
│   │   └── query/                   # 查询DAO（gen.go + banner.gen.go等）
│   ├── rep/                         # 仓储层（数据访问）
│   │   ├── banner.go                # Banner仓储
│   │   └── home_category.go         # 分类仓储
│   ├── ser/                         # 服务层（业务逻辑）
│   │   ├── banner.go                # Banner服务
│   │   ├── home_category_service.go # 分类服务
│   │   ├── s3_service.go            # 文件服务
│   │   ├── enum_service.go          # 枚举服务
│   │   └── content_strategy.go      # 内容策略服务
│   └── cache/                       # 缓存层
│       └── banner.go                # Banner缓存（Redis）
```

### 6.2 分层架构

```
RPC Handler (handler.go)        ← 接收RPC请求，委托给Service
    ↓
Service (internal/ser/)         ← 业务逻辑：校验→检查→转换→持久化→缓存→日志
    ↓
Repository (internal/rep/)      ← 数据访问：封装GORM查询，支持事务
    ↓
Query (internal/gorm_gen/query/) ← GORM Gen自动生成的类型安全查询
    ↓
Model (internal/gorm_gen/model/) ← GORM Gen自动生成的数据模型
    ↓
Database (TiDB/MySQL)

辅助层：
- Enum (internal/enum/)      ← 常量、枚举、映射表
- Error (internal/errs/)     ← 错误码定义
- Config (internal/cfg/)     ← 数据库、Redis初始化
- Cache (internal/cache/)    ← Redis缓存逻辑
```

### 6.3 main.go 启动流程

```go
func main() {
    lg.InitKLog()                    // 1. 初始化Kitex日志
    etcd.LoadAndWatch("/slg/")       // 2. 加载ETCD配置（注意：服务用/slg/，网关用/slg/conf/）
    cfg.InitRedis()                  // 3. 初始化Redis

    port := flag.String("port",      // 4. 获取端口
        etcd.DirectGet(ctx, namesp.EtcdAppPort), "端口号")
    flag.Parse()

    // 5. 创建服务实例（依赖注入）
    bannerService := ser.NewBannerService(cache.NewBannerCache())
    categoryService := ser.NewCategoryService()
    // ...

    // 6. 创建Kitex RPC Server
    srv := appservice.NewServer(
        &AppServiceImpl{
            bannerService: bannerService,
            // ...
        },
        rpc.InitRpcServerParams(
            namesp.EtcdAppService,                    // 服务名 "app_service"
            utils.S2Int(context.Background(), *port), // 端口
        )...,
    )

    srv.Run()                        // 7. 启动（阻塞）
}
```

### 6.4 Handler层（handler.go）

```go
type AppServiceImpl struct {
    bannerService          *ser.BannerService
    categoryService        *ser.CategoryService
    // ...
}

// 每个方法一行委托，不含业务逻辑
func (s *AppServiceImpl) CreateBanner(ctx context.Context, req *ser_app.CreateBannerReq) (resp *ser_app.CreateBannerResp, err error) {
    return s.bannerService.CreateBanner(ctx, req)
}

func (s *AppServiceImpl) PageBanner(ctx context.Context, req *ser_app.PageBannerReq) (resp *ser_app.PageBannerResp, err error) {
    return s.bannerService.PageBanner(ctx, req)
}
```

**Handler层规则**：
- 职责单一：只做委托，不处理业务
- 方法签名：`(ctx, *XxxReq) → (*XxxResp, error)`
- 方法命名：`<动词><资源>`（与IDL一致）

### 6.5 Service层（internal/ser/）

**标准方法模板**：

```go
type BannerService struct {
    repo  *rep.BannerRepo
    cache *cache.BannerCache
}

func NewBannerService(cache *cache.BannerCache) *BannerService {
    return &BannerService{
        repo:  rep.NewBannerRepo(),
        cache: cache,
    }
}

func (s *BannerService) CreateBanner(ctx context.Context, req *ser_app.CreateBannerReq) (resp *ser_app.CreateBannerResp, retErr error) {
    // defer：失败时记录日志
    defer func() {
        if retErr != nil {
            addBannerLog(ctx, "新增Banner失败:"+bizErrMsg(retErr), enum.HandleFail)
        }
    }()

    // 1. 数据预处理
    req.Images = deduplicateImagesByLang(req.Images)

    // 2. 参数校验
    if err := validateCreateBanner(ctx, req); err != nil {
        return nil, err
    }

    // 3. 唯一性检查
    exists, err := s.repo.ExistsByName(ctx, req.PagePosition, req.BannerName, 0)
    if err != nil { return nil, err }
    if exists {
        return nil, ret.BizErr(ctx, errs.BannerNameAlreadyExist, "Banner名称已存在")
    }

    // 4. 数据转换
    banner := &model.Banner{
        BannerName:   req.BannerName,
        CreateByName: tracer.GetUserName(ctx),  // 从context获取操作人
        Status:       enum.StatusDisabled,       // 默认禁用
        // ...
    }

    // 5. 持久化（事务）
    if err := s.repo.ShiftAndCreate(ctx, req.PagePosition, req.SortValue, banner); err != nil {
        return nil, err
    }

    // 6. 清除缓存
    s.cache.DelAll(ctx)

    // 7. 记录成功日志
    addBannerLog(ctx, "新增Banner成功", enum.HandleSuccess)

    return &ser_app.CreateBannerResp{Id: banner.ID}, nil
}
```

**Service层关键设计模式**：
- **defer错误日志**：方法返回前自动记录失败日志
- **参数校验独立化**：`validateXxx()` 独立函数
- **错误包装**：`ret.BizErr(ctx, errCode, msg)` 统一包装
- **操作人追踪**：`tracer.GetUserName(ctx)` 获取操作人
- **缓存管理**：写操作后 `cache.DelAll()`
- **操作日志**：调用 `rpc.BLogClient().AddActLog()` 记录到日志服务

### 6.6 Repository层（internal/rep/）

**标准方法模板**：

```go
type BannerRepo struct{}

func NewBannerRepo() *BannerRepo { return &BannerRepo{} }

// 分页查询（带条件过滤）
func (r *BannerRepo) PageBanner(ctx context.Context, f *BannerFilter) (*dto.PageResult[model.Banner], error) {
    b := query.Q.Banner
    q := b.WithContext(ctx).Where(b.DeleteAt.Eq(0))  // 软删除过滤

    if f.BannerName != nil && *f.BannerName != "" {
        q = q.Where(b.BannerName.Like("%" + *f.BannerName + "%"))
    }
    // ... 其他条件

    offset := (f.PageNo - 1) * f.PageSize
    list, total, err := q.Order(b.ID.Desc()).FindByPage(offset, f.PageSize)
    return &dto.PageResult[model.Banner]{Data: list, Total: total, PageNo: f.PageNo, PageSize: f.PageSize}, err
}

// 单条查询
func (r *BannerRepo) GetByID(ctx context.Context, id int64) (*model.Banner, error) {
    b := query.Q.Banner
    banner, err := b.WithContext(ctx).Where(b.ID.Eq(id)).Where(b.DeleteAt.Eq(0)).First()
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, nil   // 不存在返回nil（非错误）
    }
    return banner, err
}

// 事务操作
func (r *BannerRepo) ShiftAndCreate(ctx context.Context, pagePosition string, sortValue int32, banner *model.Banner) error {
    return query.Q.Transaction(func(tx *query.Query) error {
        // 事务内操作1：排序值后移
        tx.Banner.WithContext(ctx).Where(...).UpdateSimple(b.SortValue.Add(1))
        // 事务内操作2：新增
        return tx.Banner.WithContext(ctx).Omit(tx.Banner.ID).Create(banner)
    })
}
```

**Repo层规则**：
- 所有查询加 `.Where(xxx.DeleteAt.Eq(0))`（软删除过滤）
- 不存在返回 `nil, nil`（非错误）
- 事务用 `query.Q.Transaction(func(tx *query.Query) error { ... })`
- Filter结构体封装复杂查询条件
- 更新用 `.Select(fields...).Updates(model)` 显式指定字段

### 6.7 Cache层（internal/cache/）

```go
type BannerCache struct{}

func (c *BannerCache) GetList(ctx context.Context, pos, terminal string) []*model.Banner {
    data := rds.Get(buildKey(pos, terminal))
    if data == "" { return nil }  // nil = 未命中
    var banners []*model.Banner
    utils.UnmarshalStr(ctx, data, &banners)
    return banners
}

func (c *BannerCache) SetList(ctx context.Context, pos, terminal string, banners []*model.Banner) {
    if banners == nil { banners = []*model.Banner{} }  // 空列表也缓存（防穿透）
    rds.Set(ctx, buildKey(pos, terminal), utils.MarshalStr(ctx, banners), redis_key.BannerListTTL)
}

func (c *BannerCache) DelAll(ctx context.Context) {
    // 枚举所有组合批量删除
    rds.Del(keys...)
}
```

### 6.8 gorm_gen.go（数据库代码生成）

**文件位置**：`internal/gen/gorm_gen.go`

```go
func main() {
    etcd.InitEtcd()
    dbName := etcd.DirectGet(context.Background(), namesp.EtcdAppDb)
    mysql_gen.GenCodeWithAll(dbName, "ser-app", "")
}
```

**使用方式**：

```bash
cd /Users/mac/gitlab/ser-app/internal/gen
go run gorm_gen.go
```

**生成产物**：
- `internal/gorm_gen/model/*.gen.go` — 数据模型结构体
- `internal/gorm_gen/query/*.gen.go` — 类型安全的查询DAO

**生成的Model示例**：

```go
type Banner struct {
    ID           int64   `gorm:"column:id;primaryKey;autoIncrement:true"`
    BannerName   string  `gorm:"column:banner_name;not null"`
    Status       int32   `gorm:"column:status;not null"`
    CreateByName string  `gorm:"column:create_by_name;not null"`
    CreateAt     int64   `gorm:"column:create_at;not null"`
    UpdateAt     int64   `gorm:"column:update_at;not null"`
    DeleteAt     int64   `gorm:"column:delete_at;not null"`  // 软删除时间戳
    // ...
}
```

**生成的Query使用方式**：

```go
// 全局Query对象（在cfg/mysql.go中SetDefault初始化）
query.Q.Banner.WithContext(ctx).
    Where(query.Q.Banner.ID.Eq(id)).
    Where(query.Q.Banner.DeleteAt.Eq(0)).
    First()
```

### 6.9 操作日志模式

所有写操作（增删改状态变更）都必须记录操作日志：

```go
func addBannerLog(ctx context.Context, action string, result string) error {
    _, err := rpc.BLogClient().AddActLog(ctx, &ser_blog.ActAddReq{
        Platform:   enum.LogBackend,      // "运营后台"
        Module:     enum.LogModel,        // "内容管理"
        Page:       enum.LogBannerPage,   // "Banner管理"
        Action:     action,               // "新增Banner（ID：123）"
        HandleType: result,               // "成功" / "失败"
    })
    return err
}
```

**注意**：日志服务调用是 oneway 异步的，不影响主流程。

---

## 第七章 从0到1开发 ser-wallet 完整指南

### Step 1：定义 IDL

**创建文件**：`/Users/mac/gitlab/common/idl/ser-wallet/service.thrift`

```thrift
namespace go ser_wallet

// 根据需求文档定义所有接口的请求/响应结构体和服务方法
// 命名规范：PascalCase方法名 + Req/Resp后缀
// 字段注解：(api.body="fieldName")
```

### Step 2：生成 Kitex 代码

```bash
cd /Users/mac/gitlab/common
bash script/gen_kitex.sh
```

生成产物位于 `common/kitex_gen/ser_wallet/walletservice/`

### Step 3：注册服务常量

**修改文件**：`/Users/mac/gitlab/common/pkg/consts/namesp/namesp.go`

```go
// 022 - 钱包服务
EtcdWalletService = "wallet_service"
EtcdWalletPort    = "/slg/serv/wallet/port"
EtcdWalletDb      = "/slg/serv/wallet/db"
```

### Step 4：添加 RPC 客户端

**修改文件**：`/Users/mac/gitlab/common/rpc/rpc_client.go`

```go
var (
    walletService *walletservice.Client
    walletOnce    sync.Once
)

func WalletClient() walletservice.Client {
    walletOnce.Do(func() {
        w, err := walletservice.NewClient(
            namesp.EtcdWalletService,
            InitRpcClientParams()...,
        )
        walletService = &w
        if err != nil {
            lg.Log().CtxErrorf(context.Background(), "RPC 创建ser-wallet客户端错误 %s", err)
        }
    })
    return *walletService
}
```

### Step 5：创建服务目录结构

```
/Users/mac/gitlab/ser-wallet/
├── main.go
├── handler.go
├── go.mod
├── internal/
│   ├── cfg/
│   │   ├── mysql.go        # sync.Once初始化MySQL
│   │   └── redis.go        # sync.Once初始化Redis
│   ├── errs/
│   │   └── code.go         # 6022xxx 错误码
│   ├── enum/
│   │   ├── wallet.go       # 钱包类型、状态等枚举
│   │   └── enum_registry.go
│   ├── gen/
│   │   └── gorm_gen.go     # 数据库代码生成
│   ├── gorm_gen/            # 自动生成（勿手编）
│   │   ├── model/
│   │   └── query/
│   ├── rep/                 # 仓储层
│   │   ├── wallet.go
│   │   ├── currency.go
│   │   └── transaction.go
│   ├── ser/                 # 服务层
│   │   ├── wallet_service.go
│   │   ├── currency_service.go
│   │   └── transaction_service.go
│   └── cache/               # 缓存层
│       ├── wallet.go
│       └── currency.go
```

### Step 6：创建数据库表 & 生成代码

1. 在 TiDB 中创建钱包相关表
2. 在 ETCD 中配置数据库名：`/slg/serv/wallet/db` → `db_wallet`
3. 编写 `internal/gen/gorm_gen.go`：

```go
func main() {
    etcd.InitEtcd()
    dbName := etcd.DirectGet(context.Background(), namesp.EtcdWalletDb)
    mysql_gen.GenCodeWithAll(dbName, "ser-wallet", "")
}
```

4. 执行：`cd internal/gen && go run gorm_gen.go`

### Step 7：编写 main.go

```go
func main() {
    lg.InitKLog()
    etcd.LoadAndWatch("/slg/")
    cfg.InitRedis()

    port := flag.String("port", etcd.DirectGet(ctx, namesp.EtcdWalletPort), "端口号")
    flag.Parse()

    walletService := ser.NewWalletService(cache.NewWalletCache())
    currencyService := ser.NewCurrencyService(cache.NewCurrencyCache())
    // ...

    srv := walletservice.NewServer(
        &WalletServiceImpl{
            walletService:   walletService,
            currencyService: currencyService,
        },
        rpc.InitRpcServerParams(
            namesp.EtcdWalletService,
            utils.S2Int(ctx, *port),
        )...,
    )

    klog.Info("ser-wallet 服务启动成功，监听端口: " + *port)
    srv.Run()
}
```

### Step 8：编写 handler.go

```go
type WalletServiceImpl struct {
    walletService   *ser.WalletService
    currencyService *ser.CurrencyService
}

func (s *WalletServiceImpl) CreateCurrency(ctx context.Context, req *ser_wallet.CreateCurrencyReq) (*ser_wallet.CreateCurrencyResp, error) {
    return s.currencyService.CreateCurrency(ctx, req)
}

func (s *WalletServiceImpl) GetBalance(ctx context.Context, req *ser_wallet.GetBalanceReq) (*ser_wallet.GetBalanceResp, error) {
    return s.walletService.GetBalance(ctx, req)
}

// ... 其他方法
```

### Step 9：编写 Service/Repo/Cache 层

参照 ser-app 的模式：
- Service：校验→检查→转换→持久化→缓存→日志
- Repo：GORM Gen查询，软删除过滤，事务支持
- Cache：Redis缓存，空结果防穿透

### Step 10：网关注册路由

**gate-back（B端管理接口）**：

```go
// biz/handler/wallet/wallet.go
func CreateCurrency(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.WalletClient().CreateCurrency)
}

// biz/router/wallet/wallet_router.go
func RouterWallet(r *server.Hertz) {
    my_router.Api.POST("/wallet/currency/create", wallet.CreateCurrency)
    my_router.Api.POST("/wallet/currency/page", wallet.PageCurrency)
    my_router.Api.POST("/wallet/balance/page", wallet.PageBalance)
    my_router.Api.POST("/wallet/transaction/page", wallet.PageTransaction)
    // ...
}

// biz/router/register.go → GeneratedRegister() 中添加
wallet.RouterWallet(r)
```

**gate-font（C端用户接口）**：

```go
// 同理，在gate-font中添加C端接口
func RouterWallet(r *server.Hertz) {
    my_router.Api.POST("/wallet/getBalance", wallet.GetBalance)
    my_router.Api.POST("/wallet/exchange", wallet.Exchange)
    // ...
}
```

### Step 11：ETCD 配置

在 ETCD 中设置以下键值：

```
/slg/serv/wallet/port   → 9022
/slg/serv/wallet/db     → db_wallet
```

### Step 12：go.work 注册

在 `/Users/mac/gitlab/go.work` 中添加 `ser-wallet` 模块路径。

### 完整修改清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `common/idl/ser-wallet/service.thrift` | 新增 | IDL接口定义 |
| `common/kitex_gen/...` | 自动生成 | 执行gen_kitex.sh |
| `common/pkg/consts/namesp/namesp.go` | 修改 | 添加服务常量 |
| `common/rpc/rpc_client.go` | 修改 | 添加WalletClient() |
| `ser-wallet/` | 新增目录 | 整个服务模块 |
| `gate-back/biz/handler/wallet/` | 新增 | B端Handler |
| `gate-back/biz/router/wallet/` | 新增 | B端路由 |
| `gate-back/biz/router/register.go` | 修改 | 注册路由 |
| `gate-font/biz/handler/wallet/` | 新增 | C端Handler |
| `gate-font/biz/router/wallet/` | 新增 | C端路由 |
| `gate-font/biz/router/register.go` | 修改 | 注册路由 |
| `go.work` | 修改 | 添加模块 |
| ETCD | 配置 | port、db键值 |

---

## 附录A 工程中所有服务编号表

| 编号 | 服务名 | ETCD注册名 | 错误码前缀 | 说明 |
|-----|--------|-----------|-----------|------|
| 001 | ser-cuser | c_user_service | 6001xxx | C端用户（已停用） |
| 002 | ser-buser | b_user_service | 6002xxx | B端后台用户 |
| 003 | ser-bcom | b_com_service | 6003xxx | 通用配置 |
| 004 | ser-i18n | i18n_service | 6004xxx | 多语言 |
| 005 | ser-ip | ip_service | 6005xxx | IP黑白名单 |
| 006 | ser-blog | blog_service | 6006xxx | 操作日志 |
| 007 | ser-s3 | s3_service | 6007xxx | S3文件 |
| 008 | ser-user | user_service | 6008xxx | C端用户 |
| 009 | ser-exp | exp_service | 6009xxx | VIP经验 |
| 010 | ser-item | item_service | 6010xxx | 道具 |
| 011 | ser-app | app_service | 6011xxx | APP配置 |
| 012 | ser-kyc | kyc_service | 6012xxx | KYC |
| 013 | ser-contentcenter | content_center_service | 6013xxx | 内容中心 |
| 014 | ser-recommend | recommend_service | 6014xxx | 推荐 |
| 015 | ser-shortvideos | shortvideos_service | 6015xxx | 短视频 |
| 017 | ser-websocket | web_socket_service | 6017xxx | WebSocket |
| 018 | ser-rtc | rtc_service | 6018xxx | RTC |
| 019 | ser-live | live_service | 6019xxx | 直播 |
| 020 | ser-cron | cron_service | 6020xxx | 定时任务 |
| 021 | ser-mall | mall_service | 6021xxx | 商城 |
| **022** | **ser-wallet** | **wallet_service** | **6022xxx** | **钱包（待开发）** |

---

## 附录B common/pkg 工具包速查表

| 包路径 | 导入名 | 核心函数/用法 |
|--------|--------|-------------|
| `common/pkg/lg` | `lg` | `lg.InitKLog()`, `lg.InitHLog()`, `lg.Log().CtxInfof(ctx, ...)`, `lg.Log().CtxErrorf(ctx, ...)` |
| `common/pkg/ret` | `ret` | `ret.BizErr(ctx, code, msg)`, `ret.HttpOk(ctx, data, i18n)`, `ret.HttpErr(ctx, err, i18n)`, `ret.HttpOne(ctx, data, i18n, err)` |
| `common/pkg/tracer` | `tracer` | `tracer.GetTraceID(ctx)`, `tracer.GetUserId(ctx)`, `tracer.GetUserName(ctx)`, `tracer.GetUserToken(ctx)`, `tracer.GetUserLang(ctx)` |
| `common/pkg/etcd` | `etcd` | `etcd.LoadAndWatch(prefix)`, `etcd.Get(ctx, key)`, `etcd.DirectGet(ctx, key)`, `etcd.InitEtcd()` |
| `common/pkg/rds` | `rds` | `rds.Get(key)`, `rds.Set(ctx, key, val, ttl)`, `rds.Del(keys...)`, `rds.HGet(key, field)` |
| `common/pkg/utils` | `utils` | `utils.MarshalStr(ctx, obj)`, `utils.UnmarshalStr(ctx, str, &obj)`, `utils.S2Int(ctx, str)`, `utils.GetLocalIPv4()` |
| `common/pkg/db/dto` | `dto` | `dto.PageResult[T]{Data, Total, PageNo, PageSize}` |
| `common/pkg/mys3` | `mys3` | `mys3.Init(opt)`, `mys3.ToS3Key(url)`, `mys3.JoinUrl(gateType, key)` |
| `common/pkg/myid` | `myid` | `myid.GenID()` — 生成Snowflake ID |
| `common/pkg/middleware` | `middleware` | `middleware.HertzAccLogStr()`, `middleware.SetHeaderInfo()`, `middleware.BlackCheck()`, `middleware.WhiteCheck()`, `middleware.FontAuthMiddleware()`, `middleware.BackAuthMiddleware()`, `middleware.RedirectS3()`, `middleware.InitRedis()` |
| `common/pkg/session` | `session` | `session.CUserTokenPayload`, `session.BUserTokenPayload` |
| `common/pkg/consts/namesp` | `namesp` | 所有ETCD服务名/端口/数据库键常量 |
| `common/pkg/consts/redis_key` | `redis_key` | Redis键前缀常量、`redis_key.GetKey(prefix, parts...)` |
| `common/pkg/consts/env` | `env` | ETCD配置键名常量（TidbAddr, RedisAddr等） |
| `common/pkg/consts/enums` | `enums` | `enums.GateTypeBack`, `enums.GateTypeFont`, `enums.EnumOption` |
| `common/pkg/db/mysql` | `mysql` | `mysql.Init(opt)`, `mysql.DB` — GORM实例 |
| `common/rpc` | `rpc` | `rpc.Handler(ctx, c, method)`, `rpc.HandlerNoParam(ctx, c, method)`, `rpc.InitRpcServerParams(name, port)`, `rpc.InitRpcClientParams()`, `rpc.XxxClient()` 各服务客户端工厂 |

---

## 附录C gate-back 与 gate-font 差异对照表

| 对比维度 | gate-back (B端) | gate-font (C端) |
|---------|----------------|-----------------|
| 服务名 | `gate_back` | `gate_font` |
| 端口键 | `/slg/serv/gate_back/port` | `/slg/serv/gate_font/port` |
| 日志初始化 | `lg.InitHLog()` | `lg.InitHLog()` |
| 路由前缀(开放) | `/admin/open` | `/open` |
| 路由前缀(鉴权) | `/admin/api` | `/api` |
| IP策略 | `WhiteCheck()` 白名单（默认拒绝） | `BlackCheck()` 黑名单（默认放行） |
| 鉴权中间件 | `BackAuthMiddleware()` | `FontAuthMiddleware()` |
| Token载体 | `BUserTokenPayload{ID, UserName}` | `CUserTokenPayload{ID, UID, NickName, Device, ...}` |
| Token Redis Key | `token`（无前缀） | `"user:token:" + token` |
| URL权限校验 | `CheckUrlAuth()` — 有 | 无 |
| 请求体大小限制 | 20MB | 默认 |
| 注入context字段 | UserID + UserName | UserID + UserName(NickName) |
| 适用场景 | 运营后台管理 | 用户端APP/H5 |

---

> 文档完毕。本文档基于 common、gate-back、gate-font、ser-app 四个模块的实际代码分析，所有结论均有代码实证。
