# 工程认知：从0到1开发新模块的完整工程指南

> 分析依据：基于 /Users/mac/gitlab 整个工程的代码级深度分析
> 核心聚焦：common、gate-font、gate-back 三个绕不过的模块
> 参考对标：ser-app（成熟模块）、ser-item、ser-bcom
> 目标：清楚知道开发 ser-wallet 需要做什么、怎么做、遵守什么规范

---

## 第一章：工程全貌

### 1.1 项目结构总览

```
/Users/mac/gitlab/
├── go.work                    # Go Workspace 总控文件（管理所有模块）
├── common/                    # 共享基础设施（IDL + 公共包 + RPC客户端）
│   ├── idl/                   #   22个服务的Thrift IDL定义
│   ├── pkg/                   #   公共代码库（29个子包）
│   ├── rpc/                   #   RPC服务端/客户端初始化
│   ├── script/                #   代码生成脚本（gen_kitex.sh）
│   └── test/                  #   测试工具
├── gate-font/                 # C端HTTP网关（面向前端用户）
├── gate-back/                 # B端HTTP网关（面向后台管理）
├── ser-app/                   # 业务服务：App配置（Banner、首页分类）
├── ser-bcom/                  # 业务服务：B端通用（枚举、开关、黑白名单）
├── ser-buser/                 # 业务服务：B端用户（角色、菜单、权限）
├── ser-i18n/                  # 基础服务：多语言翻译
├── ser-ip/                    # 基础服务：IP黑白名单
├── ser-item/                  # 业务服务：道具配置
├── ser-s3/                    # 基础服务：文件存储
├── ser-wallet/                # 【待开发】多币种钱包模块
└── z-readme/                  # 文档资料
```

### 1.2 技术栈清单

| 层级 | 组件 | 版本 | 用途 |
|------|------|------|------|
| 语言 | Go | 1.25.5 | 主语言，go.work管理多模块 |
| HTTP框架 | Hertz (CloudWeGo) | v0.10.3 | 网关层HTTP服务 |
| RPC框架 | Kitex (CloudWeGo) | v0.16.0 | 微服务间RPC通信 |
| IDL | Thrift | - | 接口定义语言，IDL先行 |
| ORM | GORM + Gen | v1.31.1 / v0.3.27 | 数据库操作+代码生成 |
| 数据库 | TiDB (MySQL兼容) | - | 关系型存储 |
| 缓存 | Redis (go-redis) | v9.17.2 | 缓存+分布式锁+队列 |
| 注册中心 | ETCD | v3.6.7 | 服务发现+配置中心 |
| 消息队列 | NATS/JetStream | v1.48.0 | 异步消息+延迟消息 |
| 消息队列 | Kafka (Sarama) | v1.46.3 | 大规模消息流 |
| 日志 | Zap (Uber) | v1.27.1 | 结构化日志 |
| 文件存储 | AWS S3 | - | 对象存储 |
| 定时任务 | Cron + XXL-Job | - | 定时调度 |
| ID生成 | Snowflake | - | 分布式唯一ID |

### 1.3 go.work 当前状态

```go
// 文件：/Users/mac/gitlab/go.work
go 1.25.5

use (
    ./common/pkg
    ./common/rpc
    ./common/test
    ./gate-back
    ./gate-font
    ./ser-app
    ./ser-bcom
    ./ser-buser
    ./ser-i18n
    ./ser-ip
    ./ser-item
    ./ser-s3
)
```

**关键发现：ser-wallet 尚未注册到 go.work**。开发时必须添加 `./ser-wallet` 到 `use` 列表。

---

## 第二章：common 模块 —— 共享基础设施

common 是所有模块的公共依赖，包含 IDL定义、公共包、RPC客户端三大部分。开发任何新模块都必须深入理解。

### 2.1 IDL 定义（common/idl/）

#### 2.1.1 目录组织

每个服务一个目录，目录名 = 服务名（ser-xxx）。目前22个服务：

```
common/idl/
├── common/           # 公共IDL（枚举等）
├── ser-app/          # service.thrift + banner.thrift + ...
├── ser-bcom/
├── ser-buser/
├── ser-i18n/
├── ser-ip/
├── ser-item/
├── ser-kyc/
├── ser-user/
├── ser-s3/
├── ser-blog/
├── ser-exp/
├── ser-live/
├── ser-rtc/
├── ser-socket/
├── ser-cron/
├── ser-mall/
├── ser-recommend/
├── ser-shortvideos/
└── ser-contentcenter/
```

**关键发现：common/idl/ 下没有 ser-wallet 目录**。这是第一步要创建的。

#### 2.1.2 IDL 文件结构规范

以 ser-app 为参考，每个服务至少包含一个 `service.thrift` 作为入口：

```thrift
// 文件：common/idl/ser-app/service.thrift
namespace go ser_app                    // 命名空间 = 模块名下划线风格

include "banner.thrift"                 // 引入本模块子IDL
include "home_category.thrift"
include "../common/enum.thrift"         // 引入公共枚举

service AppService {                    // 服务名 = 模块名 + Service
    // 注释说明方法用途
    banner.CreateBannerResp CreateBanner(1: banner.CreateBannerReq req)
    banner.ListBannerResp ListBanner(1: banner.ListBannerReq req)
    // 无参数方法
    home_category.ListHomeCategoryResp ListHomeCategory()
}
```

子IDL文件结构（以 banner.thrift 为例）：

```thrift
namespace go ser_app                    // 同一模块共享命名空间

// 数据结构定义
struct BannerItem {
    1:  i64 id;                         // 字段编号从1开始
    2:  string bannerName;              // 驼峰命名
    3:  i32 status;                     // i32对应int32
    4:  list<BannerImage> images;       // 支持列表嵌套
    5:  i64 createAt;                   // 时间戳用i64（毫秒）
}

// 请求结构
struct CreateBannerReq {
    1: required string bannerName;      // required = 必填
    2: optional i32 status;             // optional = 可选
    3: required list<BannerImage> images;
}

// 响应结构
struct CreateBannerResp {
    1: i64 id;                          // 返回创建后的ID
}
```

**命名规范总结**：
- namespace：`ser_模块名`（下划线连接）
- service名：`XxxService`（大驼峰）
- 方法名：`动词+名词`（大驼峰），如 `CreateBanner`、`PageBanner`、`ListBanner`
- 请求结构：`方法名Req`
- 响应结构：`方法名Resp`
- 字段名：驼峰
- 字段类型：i32(int32)、i64(int64)、string、bool、list<T>、map<K,V>

#### 2.1.3 对 ser-wallet 的启示

需要创建 `common/idl/ser-wallet/` 目录，包含：
- `service.thrift` — 服务入口，定义 WalletService
- `wallet.thrift` — 钱包相关结构和请求/响应
- `currency.thrift` — 币种配置相关结构
- `exchange.thrift` — 兑换相关结构
- 可能还有 `deposit.thrift`、`withdraw.thrift`、`record.thrift` 等

### 2.2 代码生成脚本（gen_kitex.sh）

#### 2.2.1 完整脚本逻辑

文件位置：`/Users/mac/gitlab/common/script/gen_kitex.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# 路径推导
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
COMMON_DIR="$(dirname "$SCRIPT_DIR")"          # common/
WORKSPACE_DIR="$(dirname "$COMMON_DIR")"       # gitlab/
IDL_BASE_DIR="$COMMON_DIR/idl"                 # common/idl/

# 进入 common/pkg 目录执行
cd "$WORKSPACE_DIR/common/pkg"

# 遍历 idl 下的每个服务目录
for service_name in $(ls "$IDL_BASE_DIR"); do
    idl_dir="$IDL_BASE_DIR/$service_name"
    [[ ! -d "$idl_dir" ]] && continue

    # 查找 *service.thrift 文件
    find "$idl_dir" -maxdepth 1 -name "*service.thrift" | while read -r thrift_file; do
        # 如果对应服务模块存在 handler.go + main.go，先拷贝过来
        if [[ -d $WORKSPACE_DIR/$service_name && -f "$WORKSPACE_DIR/$service_name/handler.go" ]]; then
            cp -r "$WORKSPACE_DIR/$service_name/handler.go" "$WORKSPACE_DIR/common/pkg/handler.go"
            cp -r "$WORKSPACE_DIR/$service_name/main.go" "$WORKSPACE_DIR/common/pkg/main.go"
        fi

        # 核心命令：kitex 代码生成
        kitex -module common/pkg \
              -I $IDL_BASE_DIR/$service_name \
              -service $service_name \
              -gen-path ./kitex_gen/ \
              $rel_thrift_path

        # 如果有handler/main，生成后拷回原服务目录
        if [[ -d $WORKSPACE_DIR/$service_name && ... ]]; then
            cp -r "$WORKSPACE_DIR/common/pkg/handler.go" "$WORKSPACE_DIR/$service_name/handler.go"
            cp -r "$WORKSPACE_DIR/common/pkg/main.go" "$WORKSPACE_DIR/$service_name/main.go"
        fi

        # 清理临时文件
        rm -f "./handler.go"
    done

    rm -f "./bootstrap.sh" "./kitex_info.yaml" "./build.sh" "./handler.go" "./main.go"
    rm -rf "./scripts"
done
```

#### 2.2.2 脚本执行流程图

```
执行 gen_kitex.sh
    │
    ├── 扫描 common/idl/ 下所有目录
    │   ├── ser-app/
    │   ├── ser-bcom/
    │   ├── ...
    │   └── ser-wallet/  （新增后才会被扫描）
    │
    ├── 对每个目录查找 *service.thrift 文件
    │
    ├── 如果对应的 ser-xxx/ 目录存在 handler.go + main.go
    │   └── 先拷贝到 common/pkg/ 临时使用
    │
    ├── 执行 kitex 命令生成代码
    │   └── 输出到 common/pkg/kitex_gen/ser_xxx/
    │       ├── xxxservice/    # 客户端接口 stub
    │       └── *.go           # 请求/响应结构体
    │
    ├── 如果有 handler/main，生成后拷回 ser-xxx/
    │   └── kitex 会更新 handler.go 新增方法签名
    │
    └── 清理临时文件
```

#### 2.2.3 对 ser-wallet 的操作顺序

1. 先在 `common/idl/ser-wallet/` 创建 IDL 文件
2. 执行 `bash common/script/gen_kitex.sh`
3. 生成物出现在 `common/pkg/kitex_gen/ser_wallet/`
4. 如果 `ser-wallet/handler.go` + `ser-wallet/main.go` 已存在，脚本会自动更新
5. 如果不存在，脚本只生成 kitex_gen 代码，handler/main 需手动创建

### 2.3 公共代码包（common/pkg/）

29个子包，按功能分类：

#### 2.3.1 核心基础包

| 包 | 路径 | 用途 | 新模块必须用 |
|----|------|------|-------------|
| etcd | `pkg/etcd/` | ETCD连接+配置加载+缓存Watch | 是 |
| lg | `pkg/lg/` | Zap日志（InitKLog/InitHLog） | 是 |
| rds | `pkg/rds/` | Redis客户端（40+操作方法） | 是 |
| db/mysql | `pkg/db/mysql/` | GORM连接初始化+插件 | 是 |
| db/cmd | `pkg/db/cmd/` | GORM Gen代码生成器 | 是（初始化阶段） |
| db/dto | `pkg/db/dto/` | PageResult分页结果通用结构 | 是 |
| ret | `pkg/ret/` | 统一响应格式（HttpOk/HttpErr/BizErr） | 是 |
| errs | `pkg/errs/` | 公共错误码定义 | 是 |
| tracer | `pkg/tracer/` | 链路追踪（trace_id, user_id等） | 是（隐式通过中间件） |
| utils | `pkg/utils/` | 22+工具函数 | 按需 |

#### 2.3.2 常量/配置包

| 包 | 路径 | 用途 |
|----|------|------|
| consts/namesp | `pkg/consts/namesp/` | ETCD服务注册名+端口+数据库键 |
| consts/env | `pkg/consts/env/` | 配置环境变量键名 |
| consts/enums | `pkg/consts/enums/` | 公共枚举（设备类型、账号类型等） |
| consts/redis_key | `pkg/consts/redis_key/` | Redis键名前缀+TTL常量 |

#### 2.3.3 中间件包

| 包 | 路径 | 用途 |
|----|------|------|
| middleware | `pkg/middleware/` | HTTP中间件（鉴权、IP检查、S3等） |
| session | `pkg/session/` | Token载荷结构定义 |

#### 2.3.4 消息/三方包

| 包 | 路径 | 用途 |
|----|------|------|
| nats | `pkg/nats/` | NATS/JetStream消息收发 |
| kafka | `pkg/kafka/` | Kafka生产消费 |
| mys3 | `pkg/mys3/` | S3对象存储 |
| i18n | `pkg/i18n/` | 多语言支持 |
| third | `pkg/third/` | 第三方API集成 |

### 2.4 ETCD 配置体系

#### 2.4.1 配置路径规范

所有配置存储在 ETCD，路径规范：

```
/slg/conf/                          # 全局配置根路径
├── tidb/
│   ├── addr                        # 数据库地址
│   ├── port                        # 数据库端口
│   ├── username                    # 用户名
│   └── password                    # 密码
├── redis/
│   ├── addr                        # Redis地址（集群用逗号分隔）
│   └── password                    # Redis密码
├── nats/addr                       # NATS地址
├── kafka/addr                      # Kafka地址
├── es/                             # Elasticsearch配置
├── s3/                             # S3配置（ak/sk/endpoint/bucket/domain/region）
├── sms/                            # 短信配置
└── security/                       # 加密密钥配置

/slg/serv/                          # 服务注册根路径
├── gate_font/
│   ├── port                        # 端口号
│   └── domain                      # 域名
├── gate_back/
│   ├── port
│   └── domain
├── app/
│   ├── port                        # 端口号
│   └── db                          # 数据库名（传给GORM）
├── buser/
│   ├── port
│   └── db
├── ... (每个服务一组 port+db)
```

#### 2.4.2 服务注册常量（namesp.go）

文件：`/Users/mac/gitlab/common/pkg/consts/namesp/namesp.go`

当前已注册21个服务（编号001-021），**ser-wallet 未注册**。

已使用编号：001(cuser停用)、002(buser)、003(bcom)、004(i18n)、005(ip)、006(blog)、007(s3)、008(user)、009(exp+item共享)、011(app)、012(kyc)、013(contentcenter)、014(recommend)、015(shortvideos)、017(websocket)、018(rtc)、019(live)、020(cron)、021(mall)

**ser-wallet 需要新增**（建议编号022）：
```go
// EtcdWalletService 022
EtcdWalletService = "wallet_service"
EtcdWalletPort    = "/slg/serv/wallet/port"
EtcdWalletDB      = "/slg/serv/wallet/db"
```

#### 2.4.3 配置加载机制

两种加载方式：

**方式一：LoadAndWatch（推荐，ser-app使用）**
```go
// 加载配置到内存缓存 + 监听变更自动刷新
etcd.LoadAndWatch("/slg/")

// 读取配置（从内存缓存读，高性能）
addr := etcd.Get(ctx, env.RedisAddr)

// 带默认值读取
val := etcd.GetDef(ctx, "key", "default")

// 注册配置变更钩子
etcd.Hook("key", func(val string) {
    // 配置变更时触发
})
```

**方式二：DirectGet（直接ETCD请求，用于初始化）**
```go
// 直接从ETCD读取（每次都发请求，用于一次性初始化场景）
etcd.InitEtcd()
dbName := etcd.DirectGet(ctx, namesp.EtcdWalletDB)
```

### 2.5 RPC 客户端注册体系

#### 2.5.1 客户端工厂模式

文件：`/Users/mac/gitlab/common/rpc/rpc_client.go`

所有RPC客户端采用 sync.Once 单例+懒加载模式：

```go
// 声明变量
var (
    walletService *walletservice.Client
    walletOnce    sync.Once
)

// 工厂方法
func WalletClient() walletservice.Client {
    walletOnce.Do(func() {
        ws, err := walletservice.NewClient(
            namesp.EtcdWalletService,     // ETCD服务名
            InitRpcClientParams()...,     // 标准客户端参数
        )
        walletService = &ws
        if err != nil {
            lg.Log().CtxErrorf(context.Background(), "RPC 创建ser-wallet客户端错误 %s", err)
        }
    })
    return *walletService
}
```

#### 2.5.2 客户端参数配置

文件：`/Users/mac/gitlab/common/rpc/init.go`

```go
func InitRpcClientParams() []client.Option {
    r, err := etcd.NewEtcdResolver(myetcd.Address())  // ETCD服务发现
    return []client.Option{
        client.WithResolver(r),                        // 注册中心解析
        client.WithRPCTimeout(3 * time.Second),        // 3秒超时
        client.WithMuxConnection(1),                   // 连接复用
        client.WithMetaHandler(transmeta.ClientTTHeaderHandler), // 链路元数据
        client.WithTransportProtocol(transport.TTHeader),
    }
}
```

#### 2.5.3 服务端参数配置

```go
func InitRpcServerParams(servName string, servPort int) []server.Option {
    r, _ := etcd.NewEtcdRegistry(myetcd.Address())
    addr, _ := net.ResolveTCPAddr("tcp", ":"+strconv.Itoa(servPort))
    return []server.Option{
        server.WithReusePort(true),          // 端口复用
        server.WithMuxTransport(),           // 多路复用
        server.WithRegistry(r),              // ETCD注册
        server.WithServerBasicInfo(&rpcinfo.EndpointBasicInfo{
            ServiceName: servName,
        }),
        server.WithMetaHandler(transmeta.ServerTTHeaderHandler), // 链路追踪
        server.WithServiceAddr(addr),
        server.WithMiddleware(/* 访问日志中间件 */),
    }
}
```

### 2.6 统一响应格式

#### 2.6.1 HTTP响应结构

文件：`/Users/mac/gitlab/common/pkg/ret/project_ret.go`

```go
type R struct {
    TraceId string         `json:"traceId"`  // 请求追踪ID（UUID）
    Code    errs.ErrorType `json:"code"`     // 0=成功, >0=业务错误码
    Msg     string         `json:"msg"`      // 错误消息（i18n翻译后）
    Data    interface{}    `json:"data"`     // 业务数据
}
```

#### 2.6.2 三个响应方法

```go
// 成功
ret.HttpOk(ctx, data)                    // → {code:0, data:...}

// 失败（自动翻译错误码）
ret.HttpErr(ctx, err, rpc.I18nClient())  // → {code:6011001, msg:"..."}

// 统一（根据err自动判断成功/失败）
ret.HttpOne(ctx, data, rpc.I18nClient(), err)
```

#### 2.6.3 业务错误创建

```go
// 在service层创建业务错误
return ret.BizErr(ctx, errs.BannerNameAlreadyExist, "Banner名称已存在")

// 简单RPC返回辅助
return ret.RpcOne(ctx, resp, err, errs.SomeError)
```

### 2.7 错误码体系

#### 2.7.1 公共错误码

文件：`/Users/mac/gitlab/common/pkg/errs/code.go`

```go
const (
    OK  ErrorType = 0          // 成功
    NG  ErrorType = 1          // 未捕获的系统异常

    SystemBaseError     = 6000000 + iota
    OperateErr          // 6000001 操作错误
    ParamErr            // 6000002 参数错误
    IpIsNil             // 6000003 IP为空
    IpAnalysisErr       // 6000004 IP解析错误
    IpNotCorrect        // 6000005 IP不正确
    IpInBlackTable      // 6000006 IP在黑名单
    IpNotInWhiteTable   // 6000007 IP不在白名单
    TokenNotExist       // 6000008 Token不存在
    TokenAnalysisFailed // 6000009 Token解析失败
    CheckUrlAuthFailed  // 6000010 URL权限校验失败
    EnumKeyNotFound     // 6000011 枚举键未找到
    SmsError            // 6000012 短信错误
    S3PathErr           // 6000013 S3路径错误
    RpcRequestErr       // 6000014 RPC请求错误
)
```

#### 2.7.2 服务错误码规范

每个服务拥有独立的错误码段，规则：`6 + 服务编号(3位) + 模块编号(3位)`

```go
// ser-app (编号011) 的错误码
const (
    BannerBaseError = 6011000 + iota     // Banner模块 6011000~6011099
    BannerNameAlreadyExist               // 6011001
    BannerNotFound                       // 6011002
)

const (
    HomeBaseError = 6011100 + iota       // 首页分类模块 6011100~6011199
    HomeReqParamErr                      // 6011101
)

const (
    ContentStrategyBaseError = 6011200 + iota  // 内容策略 6011200~6011299
)
```

**ser-wallet 的错误码规划**（假设编号022）：
```go
// 钱包模块 6022000~6022099
// 币种配置模块 6022100~6022199
// 充值模块 6022200~6022299
// 兑换模块 6022300~6022399
// 提现模块 6022400~6022499
// 交易记录模块 6022500~6022599
```

### 2.8 Redis 工具包

文件：`/Users/mac/gitlab/common/pkg/rds/redis_util.go`

提供40+方法，覆盖：

| 类别 | 方法 | 场景 |
|------|------|------|
| 字符串 | `Set/Get/SetNX/Incr/Decr` | 基础缓存、分布式锁、计数器 |
| 哈希 | `HSet/HGet/HMGet/HGetAll/HMSet` | 结构化缓存（如用户钱包信息） |
| 列表 | `RPush/LPop/LRange/LRem` | 队列操作 |
| 集合 | `SAdd/SRem/SIsMember/SMembers` | 去重集合 |
| 有序集合 | `ZAdd/ZRange/ZRevRange/ZRem` | 排行榜 |
| 高级 | `Pipelined/Eval` | 批量操作、Lua脚本 |
| 生命周期 | `Expire/Ttl/Del` | 过期管理 |

### 2.9 GORM Gen 代码生成

#### 2.9.1 生成器入口

文件：`/Users/mac/gitlab/common/pkg/db/cmd/generate.go`

核心函数：
```go
// 生成指定表的代码
GenCode(dbname, serviceName, rePrefix, tableNames...)

// 生成整个数据库所有表的代码
GenCodeWithAll(dbname, serviceName, rePrefix)
```

生成过程：
1. 从ETCD读取数据库连接信息
2. 连接数据库，反射表结构
3. 生成三类文件：
   - `internal/gorm_gen/model/*.gen.go` — 数据模型（对应表结构）
   - `internal/gorm_gen/query/*.gen.go` — 类型安全的查询构建器
   - `internal/gorm_gen/repo/*/` — CRUD仓库方法

#### 2.9.2 生成的仓库模板

文件：`/Users/mac/gitlab/common/pkg/db/cmd/repo_template.go`

每个表自动生成15个标准方法：

| 方法 | 签名 | 说明 |
|------|------|------|
| Create | `(ctx, *entity) error` | 新增（自增ID） |
| BatchCreate | `(ctx, []*entity) error` | 批量新增（每批200） |
| Update | `(ctx, *entity) error` | 更新（忽略零值） |
| BatchUpdate | `(ctx, []*entity) error` | 批量更新（事务） |
| BatchSave | `(ctx, []*entity) error` | 批量保存（自动区分新增/更新） |
| UpdateByIds | `(ctx, *entity, []int64) error` | 按ID批量更新 |
| UpdateByConds | `(ctx, *entity, ...conds) error` | 按条件更新 |
| UpdateFields | `(ctx, *entity, ...fields) error` | 更新指定字段（不忽略零值） |
| UpdateFieldsByConds | `(ctx, *entity, conds, ...fields) error` | 条件+指定字段 |
| DeleteById | `(ctx, ...int64) error` | 按ID删除 |
| DeleteByConds | `(ctx, ...conds) error` | 按条件删除 |
| GetOneById | `(ctx, int64) (*entity, error)` | 按ID查一条 |
| GetOneByConds | `(ctx, ...conds) (*entity, error)` | 按条件查一条 |
| GetListByIds | `(ctx, []int64) ([]*entity, error)` | 按ID批量查 |
| GetListByConds | `(ctx, ...conds) ([]*entity, error)` | 按条件查列表（上限10000） |
| GetPageByConds | `(ctx, pageNo, pageSize, orderby, ...conds) (*PageResult, error)` | 分页查询 |
| GetValuesByConds | `(ctx, field, ...conds) ([]T, error)` | 查某字段值列表 |
| Count | `(ctx, ...conds) (int64, error)` | 计数 |
| Exists | `(ctx, ...conds) (bool, error)` | 是否存在 |

#### 2.9.3 每个服务的 gorm_gen.go

文件位于 `ser-xxx/internal/gen/gorm_gen.go`，模式固定：

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
        etcd.DirectGet(context.Background(), namesp.EtcdWalletDB),  // 数据库名
        "ser-wallet",  // 服务名（影响生成路径）
        "",            // 表前缀（需要去掉的）
    )
}
```

**关键发现：当前 ser-wallet/internal/gen/gorm_gen.go 引用的是 `namesp.EtcdAppDb`（ser-app的数据库），这是错误的**。必须改为 `namesp.EtcdWalletDB`（等注册后）。

### 2.10 链路追踪（tracer包）

文件：`/Users/mac/gitlab/common/pkg/tracer/tracer.go`

#### 2.10.1 追踪头定义

```go
const (
    TraceIDHeader        = "X-Trace-Id"         // UUID，每请求生成
    ClientIPHeader       = "X-Client-IP"        // 客户端IP
    UserTokenHeader      = "X-User-Token"       // 认证令牌
    UserIdHeader         = "X-User-ID"          // 用户ID（int64→string）
    UserNameHeader       = "X-User-Name"        // 用户昵称
    UserLangHeader       = "X-User-Language"    // 语言偏好
    UserDeviceIDHeader   = "X-User-Device-ID"   // 设备ID
    UserDeviceTypeHeader = "X-User-Device-Type" // 设备类型
)
```

#### 2.10.2 上下文提取方法

```go
tracer.GetTraceID(ctx)    // 获取追踪ID
tracer.GetUserId(ctx)     // 获取用户ID（int64）
tracer.GetUserToken(ctx)  // 获取Token
tracer.GetClientIp(ctx)   // 获取客户端IP
tracer.GetUserLang(ctx)   // 获取语言
tracer.GetDeviceID(ctx)   // 获取设备ID
tracer.GetDeviceType(ctx) // 获取设备类型
tracer.GetUserName(ctx)   // 获取用户名
```

**重要**：这些值通过 `metainfo.WithPersistentValue()` 在RPC调用间传播（TTHeader协议），ser-wallet 的 service 层可直接通过 `tracer.GetUserId(ctx)` 获取当前用户ID，无需额外传参。

---

## 第三章：gate-font —— C端HTTP网关

### 3.1 职责定位

gate-font 是面向C端用户的HTTP网关，职责：
- HTTP请求接收与路由分发
- 请求参数绑定与校验
- C端用户认证（Token验证）
- IP黑名单检查
- 调用后端RPC服务
- 统一响应格式化（含i18n错误翻译）
- 链路追踪（trace_id生成与传播）
- S3文件访问代理

### 3.2 路由体系

#### 3.2.1 两个路由组

文件：`/Users/mac/gitlab/gate-font/biz/router/my_router/my_router.go`

```go
var Open *route.RouterGroup    // 无需鉴权
var Api *route.RouterGroup     // 需要鉴权

func InitMyRouter(h *server.Hertz) {
    Open = h.Group("/open",
        middleware.SetHeaderInfo(),       // 提取请求头
        middleware.BlackCheck())          // IP黑名单检查

    Api = h.Group("/api",
        middleware.SetHeaderInfo(),       // 提取请求头
        middleware.BlackCheck(),          // IP黑名单检查
        middleware.FontAuthMiddleware())  // C端Token认证

    s3Group := h.Group("/s3")
    s3Group.GET("/*path", middleware.RedirectS3())  // S3文件代理
}
```

#### 3.2.2 路由注册模式

文件：`/Users/mac/gitlab/gate-font/biz/router/register.go`

```go
func GeneratedRegister(r *server.Hertz) {
    buser.RouterRole(r)
    user.RouterUser(r)
    app.RouterApp(r)
    enum.RouterEnum(r)
    kyc.RouterKyc(r)
    shortvideos.RouterShortvideos(r)
    rtc.RouterRtc(r)
    live.RouterLive(r)
    // TODO: wallet.RouterWallet(r)  ← ser-wallet 需要在这里注册
}
```

每个模块的路由文件模式：

```go
// 文件：biz/router/wallet/wallet_router.go
package wallet

import (
    "gate-font/biz/handler/wallet"
    "gate-font/biz/router/my_router"
    "github.com/cloudwego/hertz/pkg/app/server"
)

func RouterWallet(r *server.Hertz) {
    // 无需鉴权
    my_router.Open.POST("/wallet/depositMethods", wallet.DepositMethods)

    // 需要鉴权
    my_router.Api.POST("/wallet/balance", wallet.GetBalance)
    my_router.Api.POST("/wallet/deposit", wallet.CreateDeposit)
    my_router.Api.POST("/wallet/exchange", wallet.Exchange)
    my_router.Api.POST("/wallet/withdraw", wallet.CreateWithdraw)
    my_router.Api.POST("/wallet/records", wallet.GetRecords)
}
```

#### 3.2.3 当前路由规模

| 模块 | 路由数 | Open | Api |
|------|--------|------|-----|
| user | 19 | 5 | 14 |
| live | 19 | 0 | 19 |
| shortvideos | 17 | 0 | 17 |
| app | 3 | 3 | 0 |
| kyc | 4 | 0 | 4 |
| rtc | 2 | 1 | 1 |
| enum | 1 | 1 | 0 |
| **合计** | **65** | **10** | **55** |

#### 3.2.4 路由命名规范

- HTTP方法：统一 POST（全站不使用RESTful GET/PUT/DELETE）
- URL格式：`/api/{module}/{action}` 或 `/open/{module}/{action}`
- action命名：驼峰动词，如 `getUserDetail`、`loginGuest`、`submitKyc`

### 3.3 Handler 模式

#### 3.3.1 标准模式（90%场景）— 一行代码

```go
// 文件：biz/handler/app/app.go
func ListBanner(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.AppClient().ListBanner)
}
```

背后 `rpc.Handler` 自动完成：
1. 通过反射初始化请求对象
2. `c.BindAndValidate` 绑定+校验
3. 调用RPC方法
4. `ret.HttpOne` 统一返回

#### 3.3.2 无参数模式

```go
func GetUserDetail(ctx context.Context, c *app.RequestContext) {
    rpc.HandlerNoParam(ctx, c, rpc.UserClient().GetUserDetail)
}
```

#### 3.3.3 手动绑定模式（需要额外逻辑时）

```go
func LoginGuest(ctx context.Context, c *app.RequestContext) {
    req := &ser_user.LoginReq{}
    err := c.BindAndValidate(req)
    if err != nil {
        c.JSON(ret.HttpOne(ctx, nil, rpc.I18nClient(), err))
        return
    }

    resp, err := rpc.UserClient().LoginGuest(ctx, req)
    c.JSON(ret.HttpOne(ctx, resp, rpc.I18nClient(), err))

    // 额外操作：设置Cookie
    if resp != nil {
        domain := string(c.URI().Host())
        c.SetCookie("session_token", resp.Token, -1, "/", domain,
            protocol.CookieSameSiteNoneMode, true, true)
    }
}
```

### 3.4 认证机制

#### 3.4.1 C端认证流程

```
HTTP请求
  → SetHeaderInfo: 从Header/Cookie提取Token到context
  → FontAuthMiddleware:
    1. 取 Token（metainfo传播的 X-User-Token）
    2. Token为空 → 返回 TokenNotExist(6000008)
    3. Redis查询：rds.Get("user:token:" + token)
    4. 结果为空 → 返回 TokenAnalysisFailed(6000009)
    5. JSON反序列化为 CUserTokenPayload{ID, NickName}
    6. 用户ID ≤ 0 → 返回 TokenAnalysisFailed
    7. 设置 X-User-ID 和 X-User-Name 到 context
    8. 继续后续处理
```

#### 3.4.2 Token载荷结构

```go
// C端用户Token
type CUserTokenPayload struct {
    ID            int64   // 用户ID
    UID           int64   // 用户UID
    LastLoginIp   string  // 最后登录IP
    LastLoginTime int64   // 最后登录时间
    Device        string  // 设备标识
    DeviceType    string  // 设备类型
    NickName      string  // 昵称
}

// Redis存储键：user:token:{token_value}
// 存储内容：上述结构的JSON序列化
```

### 3.5 gate-font 启动流程

文件：`/Users/mac/gitlab/gate-font/main.go`

```
main()
  ├── lg.InitHLog()                      // 初始化Hertz日志
  ├── etcd.LoadAndWatch("/slg/conf/")    // 加载ETCD配置+监听
  ├── middleware.InitRedis()             // 初始化Redis（用于Token验证等）
  ├── mys3.Init(...)                     // 初始化S3客户端
  ├── flag获取端口号                      // 命令行 > ETCD配置
  ├── etcdhz.NewEtcdRegistry()           // ETCD注册中心
  ├── server.Default(...)                // 创建Hertz服务器
  │   ├── WithHostPorts("0.0.0.0:port")  // 监听所有网卡
  │   ├── WithRegistry(...)              // 注册到ETCD
  │   └── WithTracer(IGlTracer{})        // 链路追踪
  ├── h.Use(HertzAccLogStr())            // 全局访问日志
  ├── my_router.InitMyRouter(h)          // 初始化路由组
  ├── register(h)                        // 注册所有模块路由
  └── h.Spin()                           // 启动服务
```

---

## 第四章：gate-back —— B端HTTP网关

### 4.1 与 gate-font 的关键差异

| 维度 | gate-font（C端） | gate-back（B端） |
|------|-----------------|-----------------|
| URL前缀 | `/open`、`/api` | `/admin/open`、`/admin/api` |
| IP安全 | `BlackCheck()`（黑名单拦截） | `WhiteCheck()`（白名单放行） |
| 认证中间件 | `FontAuthMiddleware()` | `BackAuthMiddleware()` |
| Token键 | `user:token:` + token | token本身作键（无前缀） |
| Token载荷 | CUserTokenPayload | BUserTokenPayload{ID, UserName} |
| 权限控制 | 仅Token验证 | Token + URL级RBAC |
| 请求体限制 | 默认 | 20MB（`WithMaxRequestBodySize`） |
| 路由数量 | 65 | 104 |

### 4.2 B端认证 + RBAC

```
HTTP请求
  → SetHeaderInfo: 提取Header信息
  → WhiteCheck: IP白名单检查
    └── 不在白名单 → IpNotInWhiteTable(6000007)
  → BackAuthMiddleware:
    1. 取Token
    2. Redis查询（键=token本身，无前缀）
    3. 反序列化 BUserTokenPayload{ID, UserName}
    4. 设置 X-User-ID 和 X-User-Name
    5. RBAC权限检查：rpc.BUserClient().CheckUrlAuth(ctx, fullPath)
       └── 未授权 → CheckUrlAuthFailed(6000010)
```

### 4.3 B端路由规模

| 模块 | 路由数 | 说明 |
|------|--------|------|
| buser | 25 | 用户/角色/菜单管理 |
| app | 18 | Banner/首页分类管理 |
| rtc | 17 | RTC房间管理 |
| live | 16 | 直播管理 |
| bcom | 16 | 通用配置/枚举/黑白名单 |
| i18n | 10 | 多语言管理 |
| item | 9 | 道具管理 |
| shortvideos | 6 | 短视频审核 |
| user | 6 | 用户查询/日志 |
| exp | 5 | VIP等级 |
| s3 | 4 | 文件上传 |
| kyc | 3 | KYC审核 |
| blog | 2 | 操作日志 |
| enum | 1 | 枚举查询 |
| **合计** | **104** | |

### 4.4 B端特有Handler模式

#### 4.4.1 CSV导出模式

```go
func QueryLogCsv(ctx context.Context, c *app.RequestContext) {
    rpc.HandlerWithCSV(ctx, c, rpc.UserClient().QueryLogCsv)
}
```

#### 4.4.2 文件上传模式

```go
func UploadFile(ctx context.Context, c *app.RequestContext) {
    reqData := &ser_s3.UploadFileReq{}
    c.BindAndValidate(reqData)

    fileHeader, _ := c.FormFile("content")     // 从表单获取文件
    file, _ := fileHeader.Open()
    defer file.Close()
    content, _ := io.ReadAll(file)
    reqData.Content = content

    resp, err := rpc.S3Client().UploadFile(ctx, reqData)
    c.JSON(ret.HttpOne(ctx, resp, rpc.I18nClient(), err))
}
```

### 4.5 gate-back 需要为 ser-wallet 新增的内容

1. `biz/handler/wallet/` — B端钱包管理Handler
2. `biz/router/wallet/wallet_router.go` — B端路由注册
3. `biz/router/register.go` — 添加 `wallet.RouterWallet(r)`

B端路由示例：
```go
func RouterWallet(r *server.Hertz) {
    // 币种配置
    my_router.Api.POST("/wallet/currency/page", wallet.CurrencyPage)
    my_router.Api.POST("/wallet/currency/update", wallet.CurrencyUpdate)
    my_router.Api.POST("/wallet/currency/status", wallet.CurrencyStatus)

    // 汇率管理
    my_router.Api.POST("/wallet/rate/list", wallet.RateList)
    my_router.Api.POST("/wallet/rate/log", wallet.RateLog)
}
```

---

## 第五章：服务模块标准结构（以 ser-app 为参考）

### 5.1 目录结构模板

```
ser-wallet/
├── main.go                      # 启动入口
├── handler.go                   # RPC方法实现（薄包装层）
├── go.mod                       # 模块依赖声明
├── Makefile                     # 构建脚本
├── .gitignore                   # Git忽略规则
├── .golangci.yml                # 代码检查配置
└── internal/                    # 私有实现（外部不可引用）
    ├── gen/
    │   └── gorm_gen.go          # 数据库代码生成脚本
    ├── gorm_gen/                # 【自动生成】不手动修改
    │   ├── model/               #   数据模型（对应表结构）
    │   ├── query/               #   类型安全查询构建器
    │   └── repo/                #   CRUD仓库方法
    ├── cfg/                     # 配置初始化
    │   ├── mysql.go             #   MySQL连接初始化
    │   └── redis.go             #   Redis连接初始化
    ├── ser/                     # 业务逻辑层（Service）
    │   ├── wallet_service.go
    │   ├── currency_service.go
    │   ├── deposit_service.go
    │   ├── exchange_service.go
    │   ├── withdraw_service.go
    │   └── record_service.go
    ├── rep/                     # 数据访问层（Repository）
    │   ├── wallet_repo.go
    │   ├── currency_repo.go
    │   └── ...
    ├── cache/                   # 缓存层
    │   ├── currency_cache.go
    │   └── rate_cache.go
    ├── enum/                    # 业务枚举常量
    │   ├── wallet.go
    │   ├── currency.go
    │   └── order.go
    └── errs/                    # 业务错误码
        └── code.go
```

### 5.2 main.go 标准模板

```go
package main

import (
    "context"
    "flag"
    "common/pkg/consts/namesp"
    "common/pkg/etcd"
    "common/pkg/kitex_gen/ser_wallet/walletservice"
    "common/pkg/lg"
    "common/pkg/utils"
    "common/rpc"
    "ser-wallet/internal/cache"
    "ser-wallet/internal/cfg"
    "ser-wallet/internal/ser"
    "github.com/cloudwego/kitex/pkg/klog"
)

func main() {
    // 1. 初始化日志
    lg.InitKLog()

    // 2. 加载ETCD配置 + 监听变更
    etcd.LoadAndWatch("/slg/")

    // 3. 初始化Redis
    cfg.InitRedis()

    // 4. 获取端口号（命令行优先 > ETCD配置）
    port := flag.String("port",
        etcd.DirectGet(context.Background(), namesp.EtcdWalletPort),
        "端口号")
    flag.Parse()

    // 5. 创建业务服务实例
    walletService := ser.NewWalletService()
    currencyService := ser.NewCurrencyService(cache.NewCurrencyCache())
    depositService := ser.NewDepositService()
    exchangeService := ser.NewExchangeService()
    withdrawService := ser.NewWithdrawService()
    recordService := ser.NewRecordService()

    // 6. 创建RPC服务器
    srv := walletservice.NewServer(
        &WalletServiceImpl{
            walletService:   walletService,
            currencyService: currencyService,
            depositService:  depositService,
            exchangeService: exchangeService,
            withdrawService: withdrawService,
            recordService:   recordService,
        },
        rpc.InitRpcServerParams(
            namesp.EtcdWalletService,
            utils.S2Int(context.Background(), *port),
        )...,
    )

    klog.Info("ser-wallet 服务启动成功，监听端口: " + *port)
    err := srv.Run()
    if err != nil {
        klog.Error("ser-wallet 运行错误", err.Error())
    }
}
```

### 5.3 handler.go 标准模板

handler 是薄包装层，只做方法分发，不含业务逻辑：

```go
package main

import (
    "context"
    ser_wallet "common/pkg/kitex_gen/ser_wallet"
    "ser-wallet/internal/ser"
)

type WalletServiceImpl struct {
    walletService   *ser.WalletService
    currencyService *ser.CurrencyService
    depositService  *ser.DepositService
    exchangeService *ser.ExchangeService
    withdrawService *ser.WithdrawService
    recordService   *ser.RecordService
}

// GetBalance 获取钱包余额
func (s *WalletServiceImpl) GetBalance(ctx context.Context, req *ser_wallet.GetBalanceReq) (resp *ser_wallet.GetBalanceResp, err error) {
    return s.walletService.GetBalance(ctx, req)
}

// CreateDeposit 创建充值订单
func (s *WalletServiceImpl) CreateDeposit(ctx context.Context, req *ser_wallet.CreateDepositReq) (resp *ser_wallet.CreateDepositResp, err error) {
    return s.depositService.CreateDeposit(ctx, req)
}

// ... 每个IDL方法一个薄包装
```

### 5.4 cfg/ 配置初始化标准

#### 5.4.1 mysql.go

```go
package cfg

import (
    "context"
    "sync"
    "common/pkg/consts/env"
    "common/pkg/consts/namesp"
    "common/pkg/db/mysql"
    "common/pkg/etcd"
    "ser-wallet/internal/gorm_gen/query"
    "gorm.io/gorm"
)

var (
    db   *gorm.DB
    once sync.Once
)

func InitMySQL() *gorm.DB {
    once.Do(func() {
        ctx := context.Background()
        opt := &mysql.InitOption{
            Addr:     etcd.DirectGet(ctx, env.TidbAddr) + ":" + etcd.DirectGet(ctx, env.TidbPort),
            Username: etcd.DirectGet(ctx, env.TidbUsername),
            Password: etcd.DirectGet(ctx, env.TidbPassword),
            Database: etcd.DirectGet(ctx, namesp.EtcdWalletDB),
        }
        mysql.Init(opt)
        db = mysql.DB
        query.SetDefault(db)    // 设置默认查询实例（使query.Q可用）
    })
    return db
}
```

#### 5.4.2 redis.go

```go
package cfg

import (
    "context"
    "sync"
    "common/pkg/consts/env"
    "common/pkg/etcd"
    "common/pkg/rds"
)

var (
    redisOnce sync.Once
    redisOpt  *rds.InitOption
)

func InitRedis() *rds.InitOption {
    redisOnce.Do(func() {
        ctx := context.Background()
        redisOpt = &rds.InitOption{
            Addr:     etcd.DirectGet(ctx, env.RedisAddr),
            Password: etcd.DirectGet(ctx, env.RedisPassword),
        }
        rds.Init(*redisOpt)
    })
    return redisOpt
}
```

### 5.5 service 层标准模式

```go
package ser

type WalletService struct {
    repo  *rep.WalletRepo
    cache *cache.WalletCache     // 可选
}

func NewWalletService() *WalletService {
    return &WalletService{
        repo: rep.NewWalletRepo(),
    }
}

func (s *WalletService) GetBalance(ctx context.Context, req *ser_wallet.GetBalanceReq) (*ser_wallet.GetBalanceResp, error) {
    // 1. 参数校验
    userId := tracer.GetUserId(ctx)   // 从context获取用户ID
    if userId <= 0 {
        return nil, ret.BizErr(ctx, errs.ParamErr, "用户ID无效")
    }

    // 2. 业务逻辑
    wallet, err := s.repo.GetByUserId(ctx, userId)
    if err != nil {
        return nil, err
    }

    // 3. 组装响应
    return &ser_wallet.GetBalanceResp{
        CenterBalance: wallet.CenterBalance,
        RewardBalance: wallet.RewardBalance,
    }, nil
}
```

### 5.6 repository 层标准模式

```go
package rep

type WalletRepo struct {
    db  *gorm.DB
    rds *rds.InitOption
}

func NewWalletRepo() *WalletRepo {
    return &WalletRepo{
        db:  cfg.InitMySQL(),
        rds: cfg.InitRedis(),
    }
}

func (r *WalletRepo) GetByUserId(ctx context.Context, userId int64) (*model.Wallet, error) {
    return query.Q.Wallet.WithContext(ctx).
        Where(query.Q.Wallet.UserID.Eq(userId)).
        Where(query.Q.Wallet.DeleteAt.Eq(0)).
        First()
}

func (r *WalletRepo) PageByFilter(ctx context.Context, f *WalletFilter) (*dto.PageResult[model.Wallet], error) {
    w := query.Q.Wallet
    q := w.WithContext(ctx).Where(w.DeleteAt.Eq(0))

    if f.CurrencyCode != "" {
        q = q.Where(w.CurrencyCode.Eq(f.CurrencyCode))
    }
    // ... 更多筛选条件

    list, total, err := q.Order(w.ID.Desc()).FindByPage(f.Offset(), f.PageSize)
    // ... 组装PageResult
}
```

### 5.7 错误码定义标准

```go
package errs

// ser-wallet 错误码定义
// 微服务编号：022，错误码前缀：6022
// 公共错误码见 common/pkg/errs/code.go

// Wallet 模块 6022000~6022099
const (
    WalletBaseError        = 6022000 + iota
    WalletNotFound         // 6022001 钱包不存在
    WalletInsufficientFund // 6022002 余额不足
    WalletFrozen           // 6022003 钱包已冻结
    WalletCurrencyMismatch // 6022004 币种不匹配
)

// Currency 模块 6022100~6022199
const (
    CurrencyBaseError     = 6022100 + iota
    CurrencyNotFound      // 6022101 币种不存在
    CurrencyDisabled      // 6022102 币种已禁用
    CurrencyRateExpired   // 6022103 汇率已过期
)

// Deposit 模块 6022200~6022299
const (
    DepositBaseError      = 6022200 + iota
    DepositOrderNotFound  // 6022201 充值订单不存在
    DepositAmountInvalid  // 6022202 充值金额无效
    DepositDuplicate      // 6022203 重复充值拦截
)

// Exchange 模块 6022300~6022399
const (
    ExchangeBaseError     = 6022300 + iota
    ExchangeRateInvalid   // 6022301 汇率无效
    ExchangeAmountInvalid // 6022302 兑换金额无效
)

// Withdraw 模块 6022400~6022499
const (
    WithdrawBaseError     = 6022400 + iota
    WithdrawKycRequired   // 6022401 需要KYC认证
    WithdrawLimitExceeded // 6022402 超出提现限额
    WithdrawOrderNotFound // 6022403 提现订单不存在
    WithdrawAuditPending  // 6022404 提现审核中
)
```

---

## 第六章：从0到1开发 ser-wallet 的完整步骤

### 6.1 阶段一：基础设施准备

按照先后依赖顺序，必须完成以下步骤：

```
步骤1: 注册ETCD命名空间
  文件: common/pkg/consts/namesp/namesp.go
  新增: EtcdWalletService / EtcdWalletPort / EtcdWalletDB

步骤2: 编写IDL文件
  目录: common/idl/ser-wallet/
  文件: service.thrift + 子IDL文件

步骤3: 执行代码生成
  命令: cd /Users/mac/gitlab && bash common/script/gen_kitex.sh
  产物: common/pkg/kitex_gen/ser_wallet/

步骤4: 注册RPC客户端
  文件: common/rpc/rpc_client.go
  新增: WalletClient() 工厂方法（供网关和其他服务调用）

步骤5: 创建服务模块
  目录: ser-wallet/
  文件: go.mod, main.go, handler.go, internal/...

步骤6: 注册到 go.work
  文件: go.work
  新增: ./ser-wallet

步骤7: ETCD配置
  设置: /slg/serv/wallet/port = "端口号"
  设置: /slg/serv/wallet/db = "数据库名"

步骤8: 创建数据库表
  手动或脚本在TiDB中创建表结构

步骤9: 执行GORM Gen
  修复: ser-wallet/internal/gen/gorm_gen.go (改为 namesp.EtcdWalletDB)
  命令: cd ser-wallet && go run internal/gen/gorm_gen.go
  产物: internal/gorm_gen/model/ + query/ + repo/
```

### 6.2 阶段二：网关路由接入

```
步骤10: gate-font 接入C端路由
  新增: gate-font/biz/handler/wallet/*.go
  新增: gate-font/biz/router/wallet/wallet_router.go
  修改: gate-font/biz/router/register.go (添加wallet路由注册)

步骤11: gate-back 接入B端路由
  新增: gate-back/biz/handler/wallet/*.go
  新增: gate-back/biz/router/wallet/wallet_router.go
  修改: gate-back/biz/router/register.go (添加wallet路由注册)
```

### 6.3 阶段三：业务实现

```
步骤12: 实现 service 层业务逻辑
  每个接口的具体实现

步骤13: 实现 repository 层数据访问
  在自动生成的repo基础上扩展复杂查询

步骤14: 实现 cache 层缓存策略
  币种配置、汇率等热数据缓存

步骤15: 配置 Redis 键名
  在 common/pkg/consts/redis_key/redis_key.go 新增键名常量

步骤16: 配置 i18n 错误码翻译
  在 i18n 服务中注册错误码对应的多语言文本
```

### 6.4 阶段四：联调验证

```
步骤17: 接口自测
  单独启动 ser-wallet，通过RPC客户端验证

步骤18: 网关联调
  启动 gate-font/gate-back，通过HTTP验证完整链路

步骤19: 跨模块联调
  与财务模块(ser-finance)联调RPC供/调接口

步骤20: 业务闭环验证
  完整业务流程：充值→入账→兑换→消费→提现
```

---

## 第七章：需要修改的 common 文件清单

开发 ser-wallet 必须修改的 common 下的文件：

| 文件 | 操作 | 内容 |
|------|------|------|
| `common/pkg/consts/namesp/namesp.go` | 新增 | EtcdWalletService/Port/DB 常量 |
| `common/idl/ser-wallet/service.thrift` | 新建 | 服务接口定义 |
| `common/idl/ser-wallet/*.thrift` | 新建 | 请求/响应结构定义 |
| `common/rpc/rpc_client.go` | 新增 | WalletClient() 工厂方法 |
| `common/pkg/consts/redis_key/redis_key.go` | 新增 | 钱包相关Redis键名常量 |
| `common/pkg/kitex_gen/ser_wallet/` | 自动生成 | gen_kitex.sh 脚本产物 |

需要修改的 gate 文件：

| 文件 | 操作 | 内容 |
|------|------|------|
| `gate-font/biz/handler/wallet/*.go` | 新建 | C端Handler |
| `gate-font/biz/router/wallet/wallet_router.go` | 新建 | C端路由 |
| `gate-font/biz/router/register.go` | 修改 | 注册C端路由 |
| `gate-back/biz/handler/wallet/*.go` | 新建 | B端Handler |
| `gate-back/biz/router/wallet/wallet_router.go` | 新建 | B端路由 |
| `gate-back/biz/router/register.go` | 修改 | 注册B端路由 |

需要修改的根文件：

| 文件 | 操作 | 内容 |
|------|------|------|
| `go.work` | 修改 | 添加 `./ser-wallet` |

### 7.1 当前 ser-wallet 现状诊断

```
已存在的文件：
  ✅ .gitignore
  ✅ .golangci.yml
  ⚠️ internal/gen/gorm_gen.go      ← 引用了错误的 namesp.EtcdAppDb
  ✅ tmp/                           ← 产品需求图片（152张）

缺失的文件：
  ❌ go.mod                         ← 模块定义
  ❌ main.go                        ← 启动入口
  ❌ handler.go                     ← RPC方法实现
  ❌ Makefile                       ← 构建脚本
  ❌ internal/cfg/mysql.go          ← 数据库初始化
  ❌ internal/cfg/redis.go          ← Redis初始化
  ❌ internal/ser/                   ← 业务逻辑层
  ❌ internal/rep/                   ← 数据访问层
  ❌ internal/cache/                 ← 缓存层
  ❌ internal/enum/                  ← 业务枚举
  ❌ internal/errs/code.go          ← 错误码定义
  ❌ internal/gorm_gen/             ← GORM生成代码（需建表后生成）

工程注册状态：
  ❌ go.work 未添加 ser-wallet
  ❌ namesp.go 未注册 wallet 服务
  ❌ common/idl/ 无 ser-wallet 目录
  ❌ rpc_client.go 无 WalletClient()
  ❌ gate-font 无 wallet 路由
  ❌ gate-back 无 wallet 路由
```

---

## 第八章：关键规范与约束

### 8.1 绝不能重复造轮子的公共能力

| 能力 | 正确做法 | 错误做法 |
|------|---------|---------|
| 用户认证 | 网关中间件自动处理 | 在service层重新验证Token |
| 用户ID获取 | `tracer.GetUserId(ctx)` | 让前端传user_id参数 |
| 响应格式 | `ret.HttpOne()`/`ret.BizErr()` | 自己定义响应结构 |
| 错误翻译 | `rpc.I18nClient()` + 错误码 | 直接返回中文错误消息 |
| Redis操作 | `rds.Set()`/`rds.Get()` 等 | 自己创建Redis客户端 |
| 日志记录 | `lg.Log().CtxInfof(ctx, ...)` | fmt.Println |
| 配置读取 | `etcd.Get(ctx, key)` | 写死配置值 |
| ID生成 | `utils.Snowflake()` | 自己实现ID生成 |
| 加密解密 | `utils.AesEncrypt()`/`utils.AesDecrypt()` | 自己实现加密 |
| 分页结果 | `dto.PageResult[T]` | 自己定义分页结构 |
| RPC调用 | `rpc.XxxClient().Method()` | 自己创建RPC连接 |

### 8.2 代码分层规范

```
                 ┌───────────────────────┐
  HTTP请求 ──→   │  gate-font / gate-back │  网关层：路由+中间件+Handler
                 │  Handler只做参数绑定    │
                 └───────────┬───────────┘
                             │ RPC调用
                 ┌───────────▼───────────┐
                 │     handler.go         │  RPC入口层：薄包装，只做方法分发
                 └───────────┬───────────┘
                             │ 调用service
                 ┌───────────▼───────────┐
                 │  internal/ser/*.go     │  业务逻辑层：校验+编排+转换
                 │  可调用多个repo/cache   │
                 └───────────┬───────────┘
                             │ 调用repo/cache
           ┌─────────────────┼─────────────────┐
  ┌────────▼────────┐  ┌─────▼──────┐  ┌───────▼───────┐
  │ internal/rep/   │  │ internal/  │  │ common/rpc/   │
  │ 数据访问层       │  │ cache/     │  │ 调用其他服务   │
  │ 使用gorm_gen    │  │ Redis缓存  │  │               │
  └─────────────────┘  └────────────┘  └───────────────┘
```

### 8.3 命名规范总结

| 维度 | 规范 | 示例 |
|------|------|------|
| 模块目录 | `ser-xxx` | ser-wallet |
| Go module | `ser-xxx` | `module ser-wallet` |
| IDL namespace | `ser_xxx` | `namespace go ser_wallet` |
| IDL service | `XxxService` | `service WalletService` |
| IDL方法 | `动词+名词` | `CreateDeposit`、`GetBalance` |
| IDL请求 | `方法名Req` | `CreateDepositReq` |
| IDL响应 | `方法名Resp` | `CreateDepositResp` |
| ETCD服务名 | `xxx_service` | `wallet_service` |
| ETCD配置路径 | `/slg/serv/xxx/` | `/slg/serv/wallet/port` |
| 错误码前缀 | `6+服务号+模块号` | `6022xxx` |
| Redis键前缀 | `模块:功能:` | `wallet:balance:` |
| Handler函数 | `PascalCase` | `GetBalance` |
| Service结构体 | `XxxService` | `WalletService` |
| Repo结构体 | `XxxRepo` | `WalletRepo` |
| C端URL | `/api/xxx/action` | `/api/wallet/balance` |
| B端URL | `/admin/api/xxx/action` | `/admin/api/wallet/currency/page` |

### 8.4 数据库表设计规范（从GORM Gen生成的model推断）

从 ser-app 的 Banner 模型可以看到标准字段：

```go
// 必备标准字段
ID           int64   `gorm:"column:id;primaryKey;autoIncrement:true"`   // 自增主键
CreateBy     int64   `gorm:"column:create_by;not null"`                 // 创建人ID
CreateByName string  `gorm:"column:create_by_name;not null"`           // 创建人名称
CreateAt     int64   `gorm:"column:create_at;not null"`                // 创建时间（毫秒时间戳）
UpdateBy     int64   `gorm:"column:update_by;not null"`                // 更新人ID
UpdateByName string  `gorm:"column:update_by_name;not null"`           // 更新人名称
UpdateAt     int64   `gorm:"column:update_at;not null"`                // 更新时间（毫秒时间戳）
DeleteAt     int64   `gorm:"column:delete_at;not null"`                // 软删除时间（0=未删除）
```

**关键特征**：
- 主键：int64自增
- 时间：毫秒时间戳（int64），不用time.Time
- 软删除：delete_at = 0 表示正常，> 0 表示已删除
- 审计字段：create_by/update_by 记录操作人ID和名称
- 可空字段：GORM Gen 配置了 `FieldNullable: true`，数据库可空字段用指针类型

---

## 第九章：请求处理完整链路

### 9.1 C端请求完整链路（以"获取钱包余额"为例）

```
用户APP
  │ POST /api/wallet/balance
  │ Header: X-User-Token: abc123
  ▼
gate-font (Hertz HTTP Server)
  │
  ├── [中间件] tracer.IGlTracer.Start()
  │   └── 生成 X-Trace-Id: uuid-xxxx
  │
  ├── [中间件] middleware.HertzAccLogStr()
  │   └── 记录访问日志
  │
  ├── [路由匹配] /api → Api路由组
  │
  ├── [中间件] middleware.SetHeaderInfo()
  │   ├── 提取 X-Client-IP
  │   ├── 提取 X-User-Token: abc123
  │   ├── 提取 X-User-Language
  │   └── metainfo.WithPersistentValue() 存入context
  │
  ├── [中间件] middleware.BlackCheck()
  │   └── rpc.IpClient().OnBlackList() → 通过
  │
  ├── [中间件] middleware.FontAuthMiddleware()
  │   ├── token = "abc123"
  │   ├── rds.Get("user:token:abc123") → JSON
  │   ├── 解析 CUserTokenPayload{ID:12345, NickName:"用户A"}
  │   ├── metainfo 设置 X-User-ID: "12345"
  │   └── metainfo 设置 X-User-Name: "用户A"
  │
  ├── [Handler] wallet.GetBalance()
  │   └── rpc.Handler(ctx, c, rpc.WalletClient().GetBalance)
  │       ├── c.BindAndValidate(&req)
  │       ├── rpc.WalletClient().GetBalance(ctx, req)
  │       │   │  (Kitex RPC调用，TTHeader携带trace_id/user_id)
  │       │   ▼
  │       │  ser-wallet (Kitex RPC Server)
  │       │   ├── [中间件] 打印RPC访问日志
  │       │   ├── handler.go: WalletServiceImpl.GetBalance()
  │       │   │   └── s.walletService.GetBalance(ctx, req)
  │       │   │       ├── userId := tracer.GetUserId(ctx) → 12345
  │       │   │       ├── s.repo.GetByUserId(ctx, 12345)
  │       │   │       │   └── query.Q.Wallet...First()  (GORM查询TiDB)
  │       │   │       └── return &GetBalanceResp{...}
  │       │   └── 返回响应
  │       │
  │       └── c.JSON(ret.HttpOne(ctx, resp, I18nClient(), nil))
  │
  └── HTTP 200
      {
        "traceId": "uuid-xxxx",
        "code": 0,
        "msg": "",
        "data": { "centerBalance": 10000, "rewardBalance": 500 }
      }
```

### 9.2 B端请求完整链路（以"币种配置分页查询"为例）

```
管理后台
  │ POST /admin/api/wallet/currency/page
  │ Header: X-User-Token: admin-token-xyz
  ▼
gate-back (Hertz HTTP Server)
  │
  ├── [中间件] tracer + 访问日志
  │
  ├── [路由匹配] /admin/api → Api路由组
  │
  ├── [中间件] SetHeaderInfo → 提取请求头
  │
  ├── [中间件] WhiteCheck()
  │   └── rpc.IpClient().OnWhiteList() → 通过（IP在白名单）
  │
  ├── [中间件] BackAuthMiddleware()
  │   ├── rds.Get("admin-token-xyz") → JSON
  │   ├── 解析 BUserTokenPayload{ID:1, UserName:"admin"}
  │   ├── 设置 X-User-ID, X-User-Name
  │   └── rpc.BUserClient().CheckUrlAuth(ctx, "/admin/api/wallet/currency/page")
  │       └── RBAC校验通过
  │
  ├── [Handler] wallet.CurrencyPage()
  │   └── rpc.Handler(ctx, c, rpc.WalletClient().CurrencyPage)
  │       └── RPC → ser-wallet → 查询 → 返回
  │
  └── HTTP 200 { traceId, code:0, data: {pageList, total, ...} }
```

### 9.3 RPC服务间调用链路（以"充值回调"为例）

```
财务模块 ser-finance
  │ 收到三方支付回调，充值成功
  │ 需要给用户钱包加款
  │
  ├── 调用 rpc.WalletClient().DepositCallback(ctx, &DepositCallbackReq{
  │       OrderNo: "C1234567890123456",
  │       UserId:  12345,
  │       Amount:  10000,     // 单位：分
  │       Currency: "VND",
  │   })
  │
  ▼
ser-wallet (Kitex RPC Server)
  ├── handler.go: DepositCallback()
  │   └── s.depositService.HandleCallback(ctx, req)
  │       ├── 1. 幂等校验（Redis SetNX 或 数据库唯一约束）
  │       ├── 2. 查询订单状态
  │       ├── 3. 更新订单状态为"已完成"
  │       ├── 4. 增加用户余额（中心钱包）
  │       ├── 5. 写入账变流水
  │       ├── 6. 如果有奖金，增加奖励钱包余额 + 写入稽核
  │       └── return &DepositCallbackResp{Success: true}
  │
  └── 返回给 ser-finance
```

---

## 第十章：ser-wallet 已知问题清单

| # | 问题 | 位置 | 影响 | 修复方式 |
|---|------|------|------|---------|
| 1 | gorm_gen.go 引用错误数据库 | `ser-wallet/internal/gen/gorm_gen.go:12` | 生成代码将连接错误的数据库 | 改 `namesp.EtcdAppDb` → `namesp.EtcdWalletDB` |
| 2 | go.work 未注册 | `go.work` | IDE无法解析、编译失败 | 添加 `./ser-wallet` |
| 3 | ETCD命名空间未注册 | `namesp.go` | 服务无法注册/发现 | 新增 EtcdWalletService/Port/DB |
| 4 | RPC客户端未注册 | `rpc_client.go` | 网关和其他服务无法调用 | 新增 WalletClient() |
| 5 | IDL未创建 | `common/idl/ser-wallet/` | 无法生成代码 | 编写thrift IDL |
| 6 | 网关路由未接入 | `gate-font` + `gate-back` | HTTP请求无法路由到服务 | 新增路由和Handler |
| 7 | 缺少 go.mod | `ser-wallet/go.mod` | 无法编译 | 创建go.mod |
| 8 | 缺少 main.go | `ser-wallet/main.go` | 无法启动 | 按模板创建 |
| 9 | 缺少 handler.go | `ser-wallet/handler.go` | 无法处理RPC | 按模板创建 |
| 10 | 缺少全部内部实现 | `internal/cfg/ser/rep/cache/enum/errs/` | 无业务逻辑 | 逐步实现 |

---

## 第十一章：和其他服务模块的对比参考

### 11.1 模块成熟度对比

| 模块 | main.go | handler.go | internal/ | IDL方法数 | 网关路由数 |
|------|---------|-----------|-----------|----------|-----------|
| ser-app | ✅ | ✅ 20方法 | ✅ 完整 | 20 | C端3 + B端18 |
| ser-item | ✅ | ✅ 12方法 | ✅ 完整 | 12 | B端9 |
| ser-bcom | ✅ | ✅ 17方法 | ✅ 完整 | 17 | B端16 |
| ser-wallet | ❌ | ❌ | ⚠️ 仅gen | 0 | 0 |

### 11.2 ETCD初始化方式选择

| 方式 | 使用场景 | 使用模块 |
|------|---------|---------|
| `etcd.LoadAndWatch("/slg/")` | 需要配置热更新的服务 | ser-app |
| `etcd.InitEtcd()` | 只需一次性初始化的服务 | ser-item, ser-bcom |

**ser-wallet 推荐**：`etcd.LoadAndWatch("/slg/")`，因为汇率配置等可能需要动态更新。

### 11.3 go.mod 依赖声明方式

| 方式 | 特点 | 使用模块 |
|------|------|---------|
| 直接 require common/pkg | 依赖go.work解析 | ser-app |
| replace common/pkg => ../ | 显式指定路径 | ser-item |

两种方式在 go.work 模式下都能工作，推荐与 ser-app 保持一致。

---

## 第十二章：开发过程中的关键检查点

### 12.1 IDL 编写检查

- [ ] namespace 是否为 `ser_wallet`
- [ ] service.thrift 是否 include 了所有子IDL
- [ ] 方法命名是否遵循 `动词+名词` 规范
- [ ] 请求/响应是否遵循 `方法名Req/Resp` 规范
- [ ] required/optional 标注是否正确
- [ ] 字段类型是否正确（i64时间戳、i32状态码、string文本）
- [ ] 是否引入了 `../common/enum.thrift`（如果需要公共枚举）

### 12.2 代码生成检查

- [ ] gen_kitex.sh 执行后 `common/pkg/kitex_gen/ser_wallet/` 是否存在
- [ ] walletservice/ 目录是否包含客户端 stub
- [ ] handler.go 和 main.go 是否被正确生成/更新
- [ ] gorm_gen.go 中数据库名引用是否正确

### 12.3 服务启动检查

- [ ] ETCD 是否已配置 `/slg/serv/wallet/port` 和 `/slg/serv/wallet/db`
- [ ] 数据库是否已创建且表结构已初始化
- [ ] Redis 连接是否正常
- [ ] RPC 服务是否成功注册到 ETCD

### 12.4 网关接入检查

- [ ] gate-font 路由是否正确指向 handler
- [ ] gate-back 路由是否正确指向 handler
- [ ] register.go 是否已添加注册调用
- [ ] B端路由是否已在 RBAC 系统注册（否则管理员无权限访问）

### 12.5 跨模块调用检查

- [ ] 其他服务调用 WalletClient() 是否正常
- [ ] ser-wallet 调用其他服务（如KYC、User）的客户端是否注册
- [ ] 幂等性是否保证（金融操作必须）
- [ ] 超时设置是否合理（默认3秒，金融操作可能需要更长）
