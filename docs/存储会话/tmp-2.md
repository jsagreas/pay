## 后续会话线索

| 会话日期 | jsonl路径 | 说明 |
|---------|----------|------|
| 2026-02-25 | /Users/mac/.claude/projects/-Users-mac-gitlab/059dad1b-a114-49ec-a933-4442f15c4bd4.jsonl | 初始会话，建立记忆管理机制，深度阅读元认知准则，完成全工程深度分析，正式任务尚未开始 |

---

## 行为准则核心理解（来源：/Users/mac/gitlab/z-readme/元认知/）

### 一、刚性约束（来源：know:严格约束.md）—— 红线，不可违反

**第一约束：默认只读**
- 所有操作默认只读，未经用户口头明确允许和执行指令，严禁增删改任何文件

**行为红线（严禁）：**
- 严禁猜测、臆想、想当然 —— 一切结论必须有依据
- 严禁局部片面分析 —— 必须全面完整深入，不能基于局部信息做推理
- 严禁分析过程中产生增删改操作 —— 分析归分析，不能擅自进入编码
- 严禁无中生有 —— 不能把没问题的说成有问题
- 严禁无脑抛问题给用户 —— 能自己通过分析获取的信息，必须自己分析获取
- 严禁在不清楚重要前置信息时想当然操作 —— 该问的必须主动问

**思考要求（必须）：**
- 宏观：清楚架构、模块功能、文件作用
- 微观：清楚代码职责、函数作用、变量含义
- 闭环：清楚模块间关系、依赖、调用链路、业务流程
- 深入：穷极追踪代码逻辑，不浅尝辄止
- 规范：清楚项目标准规范、技术栈、开发风格

**执行约束：**
- 必须同时理解"脚手架规范"+"需求文档"，两者缺一不可
- 定位是后端开发，区分前后端职责边界
- 必须得到用户明确同意才能编码
- 分步骤、分阶段、小步慢跑、迭代推进
- 流程：给思路步骤 → 拆解需求 → 确认 → 执行 → 验证 → 迭代

### 二、元认知框架（来源：know:元认知.md）—— 九大类行为准则

**文件结构：上半部分是用户侧维度定义，下半部分（分隔线后）是AI必须遵守的行为准则。两部分一一对应，AI侧更严格。**

| 大类 | 核心行为要求 |
|------|------------|
| 一、问题定义与目标对齐 | 先搞清背景/意图/目标/任务性质/阶段/边界/紧急度；信息不足先问不猜；区分阶段不跨阶段给结论；显性化所有假设前提 |
| 二、协作方式与交互控制 | 按指定角色参与不越权；信息不足优先澄清关键缺口（最多3个高影响问题）；不替代决策；关键节点确认前提防语义漂移；协作模式变更需显式确认 |
| 三、用户认知与表达控制 | 按用户水平调整深度；默认"结论→理由→步骤→验证"顺序；结构化输出；控制信息密度和认知负担；术语首次出现给人话释义 |
| 四、事实输入与证据基础 | 只基于提供的事实推理；区分确定结论/推测判断/假设前提；标注信息来源可信度；证据冲突必须显式指出；不足时给"最小必需证据清单" |
| 五、方案设计与实施落地 | 给可执行步骤而非方向性建议；每步说明输入/动作/预期输出；评估前置条件/变更影响/回退方案；先判断方案必要性和可行性 |
| 六、约束、环境与非功能控制 | 严守硬约束不绕过；冲突时列出请求排序；基于真实环境设计不假设；默认安全稳定优先；保守路径优先 |
| 七、风险控制与稳定性 | 默认按高风险处理；不越权不替代决策；信息严重不足时暂停不硬给；结论标注时效和前提；输出末尾自检 |
| 八、验证闭环与成功判定 | 给可观察的通过/不通过信号；区分"逻辑正确"和"用户认可"；支持多轮迭代每轮说明变更点；验证方案考虑执行成本 |
| 九、全局策略控制 | 围绕用户声明的优化目标；历史结论冲突时显式指出；不自行建立/修改长期记忆；区分正式结论和探索讨论；防策略漂移；阶段封版提示 |

### 三、项目背景（来源：know:项目背景.md）

- **产品定位：** 基于Go语言的大型分布式微服务系统，泛娱乐+合规竞彩平台，B/C双端
- **业务场景：** 社交、直播、短视频、语聊、游戏、竞彩、真人视讯
- **核心域（14大业务域）：**
  - 通用域：用户、认证权限(RBAC)、组织角色、配置中心、功能开关灰度、日志审计
  - 内容社交域：内容资产(UGC/PGC)、社交关系、互动行为、IM消息、通知推送、任务调度
  - 实时互动域：房间会话、上下麦连麦、实时状态同步、弹幕、玩法规则引擎、对局赛事
  - 支付交易域：支付、钱包资产、订单(状态机/幂等)、交易流水账本、对账清结算
  - 虚拟经济域：礼物道具配置、经济规则(产出消耗平衡)、分成收益、周期结算
  - 竞彩博彩核心域：赛事赛程、玩法投注规则、赔率盘口、投注单、投注风控、开奖结算、返奖派奖、合规监管、策略调控
  - 真人视讯域：桌台牌局管理、荷官操作审计、视频流控制、封盘控制、公平性追溯
  - 风控安全域：账号安全、内容审核、反洗钱、行为反作弊、业务/交易风控
  - 推荐增长域：推荐策略、信息流分发、榜单排序、拉新留存裂变
  - 运营活动域：活动任务、运营位资源位、运营策略
  - 客服仲裁域：工单流转、封禁申诉、投注争议
  - 数据平台域：埋点采集、BI报表、规则引擎、任务调度
  - 后台管理域：用户/内容/配置管理、权限审计、发布变更回滚
- **核心业务流：**
  - 用户流：拉新→注册→合规校验→场景触达→使用→付费→投注/游戏→结算→风控→留存
  - 资金流：充值→钱包入账→投注/下注→锁单→赛果→结算→返奖→账本→对账→风控→合规审计

---

## 工程深度分析成果（2026-02-25 完成）

### 整体架构：Go 1.25.5 Workspace 微服务

**模块清单（排除z-tmp/z-readme）：**
- common/pkg — 公共基础设施（日志/DB/Redis/ETCD/Kafka/NATS/ES/S3/工具/中间件）
- common/rpc — 21个微服务RPC客户端工厂+通用Handler
- common/test — 测试公共设施
- gate-back — B端后台网关（Hertz HTTP，~115 APIs，白名单+BackAuth）
- gate-font — C端前台网关（Hertz HTTP，~54 APIs，黑名单+FontAuth）
- ser-app — Banner管理+首页分类+内容策略（21 RPC，错误码6011xxx）
- ser-bcom — 黑白名单管理+开关配置+枚举查询（16 RPC，错误码6003xxx）
- ser-buser — 用户/角色/菜单/RBAC权限/登录认证（25 RPC，错误码6002xxx）
- ser-i18n — 多语言配置+翻译缓存+导出（13 RPC，错误码6004xxx）
- ser-ip — IP黑名单(按国家)+白名单(按IP)（10 RPC，错误码6001xxx）
- ser-item — 道具配置+发布管理+内存缓存（11 RPC，错误码6010xxx）
- ser-s3 — 文件上传/删除/预签名/配置（11 RPC，错误码6007xxx）

**核心技术栈：** Hertz(HTTP) + Kitex(RPC) + GORM/Gen(ORM) + TiDB(DB) + Redis + ETCD(配置/服务发现) + Kafka + NATS + ES + AWS S3 + Zap(日志) + OpenTelemetry(链路追踪)

**统一分层架构：** handler.go → internal/ser/(业务) → internal/rep/(仓储) → internal/gorm_gen/(生成CRUD) → internal/cfg/(配置) → internal/errs/(错误码) → internal/enum/(枚举)

**统一启动模式：** InitKLog → ETCD.LoadAndWatch → InitRedis → InitMySQL → NewService → NewServer(InitRpcServerParams) → Run

**IDL已定义但本地未部署的服务：** ser-user, ser-blog, ser-exp, ser-kyc, ser-live, ser-rtc, ser-shortvideos, ser-socket, ser-contentcenter, ser-recommend, ser-cron, ser-mall

**用户补充说明：** 这是一个微服务项目，本地只有部分模块（权限原因），处于迭代开发中。两个网关 + common 是重点分析对象；其他模块由不同组员开发，代码质量和风格参差不齐。

---

## IDL驱动开发 + 代码生成工作流（代码证据链）

> 此部分基于对gen_kitex.sh、IDL目录、kitex_gen产物、gorm_gen链路的实际代码深度分析，结合用户反馈修正整理。每条结论附代码依据。

### 一、IDL 规范与结构

**IDL根目录：** `common/idl/`（依据：`common/idl/idl.md` → "此目录用于存放.thrift文件，文件夹名与微服务名保持一致"）

**目录组织：**
- 每个服务一个子目录，目录名 = 服务名（如 `ser-app/`、`ser-buser/`）
- 每个服务目录下包含：
  - `service.thrift` — 主入口，定义 Service 接口和所有 RPC 方法
  - 若干 model thrift 文件 — 定义数据结构（Req/Resp/Item）
- `common/enum.thrift` — 公共枚举，各服务通过 `include "../common/enum.thrift"` 引用

**IDL编写规范**（依据：`common/idl/ser-app/service.thrift`）：
```thrift
namespace go ser_app              // Go包名 = 服务目录名下划线形式
include "banner.thrift"           // 引入同目录 model 文件
include "../common/enum.thrift"   // 引入公共枚举

service AppService {              // 服务名 = 目录名大驼峰 + Service
    banner.CreateBannerResp CreateBanner(1: banner.CreateBannerReq req)
    // 方法签名：Response MethodName(1: Request req)
}
```

**Req/Resp规范**（依据：`common/idl/ser-app/banner.thrift`）：
- struct 字段使用 `required` / `optional` 标注
- 分页请求包含 `pageNo`/`pageSize`，响应包含 `pageList`/`total`/`pageNo`/`pageSize`
- B端和C端使用不同的 Item 结构（如 `BannerItem` vs `BannerItemC`，C端精简字段）

**已有IDL的服务**（common/idl/ 下共 22 个子目录）：
ser-app, ser-bcom, ser-blog, ser-buser, ser-contentcenter, ser-cron, ser-exp, ser-i18n, ser-ip, ser-item, ser-kyc, ser-live, ser-mall, ser-recommend, ser-rtc, ser-s3, ser-shortvideos, ser-socket, ser-user + common（公共枚举）

### 二、gen_kitex.sh 代码生成脚本

**文件：** `common/script/gen_kitex.sh`

**执行前提：** 需安装 kitex 命令行工具（`go install github.com/cloudwego/kitex/tool/cmd/kitex@latest`）

**脚本工作流（逐步）：**

| 行号 | 步骤 | 说明 |
|------|------|------|
| 5-8 | 路径初始化 | SCRIPT_DIR→COMMON_DIR→WORKSPACE_DIR→IDL_BASE_DIR=common/idl |
| 12-15 | 工具检查 | 检查 kitex 命令是否可用 |
| 20 | 切换目录 | `cd "$WORKSPACE_DIR/common/pkg"` — 工作目录在 common/pkg |
| 23 | 遍历IDL目录 | `for service_name in $(ls "$IDL_BASE_DIR")` |
| 31 | 匹配规则 | `find "$idl_dir" -maxdepth 1 -name "*service.thrift"` — 只匹配含 service 的 thrift 文件 |
| 38-44 | 增量模式 | 如果 `ser-xxx/handler.go` + `main.go` 已存在，先拷贝到 common/pkg（Kitex 会增量更新方法桩） |
| 47-49 | **核心命令** | `kitex -module common/pkg -I $IDL_BASE_DIR/$service_name -service $service_name -gen-path ./kitex_gen/ $thrift_file` |
| 52-55 | 回写 | 更新后的 handler.go/main.go 拷贝回 ser-xxx/ |
| 58-68 | 清理 | 删除临时文件 bootstrap.sh、kitex_info.yaml、build.sh、scripts/ |

**核心命令参数解析：**
- `-module common/pkg` — 生成代码的 Go module 路径
- `-I $IDL_BASE_DIR/$service_name` — thrift include 搜索路径
- `-service $service_name` — 服务名
- `-gen-path ./kitex_gen/` — 生成代码输出到 common/pkg/kitex_gen/
- 最后参数 — thrift 文件路径

**生成产物位置：** `common/pkg/kitex_gen/{namespace}/`（全局共享，所有模块通过 import 引用）

**生成内容（以 ser_app 为例，依据 common/pkg/kitex_gen/ser_app/ 实际目录）：**
- `banner.go` / `home_category.go` / `s3.go` / `content_strategy.go` — 数据结构（Req/Resp/Item，由 thriftgo 生成）
- `service.go` — Service 接口定义
- `k-*.go` — 序列化/反序列化辅助代码
- `appservice/server.go` — `NewServer()` 工厂函数（创建 RPC 服务器）
- `appservice/client.go` — RPC 客户端
- `appservice/appservice.go` — 服务接口定义

**关键认知：** gen_kitex.sh 同时处理两件事：
1. 生成 `kitex_gen/` 下的 RPC 基础代码（结构体 + Client + Server）
2. 生成/更新 `ser-xxx/handler.go` 和 `main.go` 的方法桩（增量，不覆盖已实现的逻辑）

### 三、GORM 代码生成链路

**生成器核心：** `common/pkg/db/cmd/generate.go`

**两种调用方式：**
- `GenCodeWithAll(dbname, serviceName, rePrefix)` — 生成库中所有表（行36-38）
- `GenCode(dbname, serviceName, rePrefix, tableNames...)` — 生成指定表（行30-32）

**每个服务的入口文件**（以 ser-app 为例，依据 `ser-app/internal/gen/gorm_gen.go`）：
```go
func main() {
    etcd.InitEtcd()
    mysql_gen.GenCodeWithAll(etcd.DirectGet(ctx, namesp.EtcdAppDb), "ser-app", "")
}
```
即：从 ETCD 获取数据库名 → 连接 TiDB → 扫描表 → 生成代码

**生成器配置**（generate.go 行65-73）：
- 输出路径：`../{serviceName}/internal/gorm_gen`
- FieldNullable=true — 可空字段用指针类型
- Mode: WithoutContext | WithDefaultQuery | WithQueryInterface

**生成三层产物：**

| 层 | 路径 | 来源 | 说明 |
|----|------|------|------|
| Model | `internal/gorm_gen/model/` | GORM Gen 自动生成 | 表结构映射的 Go struct |
| Query | `internal/gorm_gen/query/` | GORM Gen 自动生成 | 类型安全的链式查询接口 |
| Repo | `internal/gorm_gen/repo/{表名}r/{表名}r.go` | 自定义模板生成 | 封装 17 个标准 CRUD 方法 |

**Repo 标准方法集**（依据 `common/pkg/db/cmd/repo_template.go`）：
Create, BatchCreate(200条/批), Update(忽略零值), BatchUpdate(200条/批+事务), BatchSave(ID=0新增否则更新), UpdateByIds, UpdateByConds, UpdateFields(不忽略零值), UpdateFieldsByConds, DeleteById, DeleteByConds, GetOneById, GetOneByConds, GetListByIds, GetListByConds(最多10000条), GetPageByConds(分页+排序), GetValuesByConds, Count, Exists

**加密字段特殊处理**（generate.go 行53-55）：`user_kyc` 表的 phone/name/id_number 字段映射为 `utils.EncryptedString` 类型

**各服务 gorm_gen 入口**（已确认存在）：
ser-app, ser-buser, ser-bcom, ser-i18n, ser-ip, ser-item, ser-s3 均有 `internal/gen/gorm_gen.go`

### 四、handler.go 与 kitex_gen 的衔接模式

**统一模式**（依据 ser-app/handler.go、ser-buser/handler.go、ser-bcom/handler.go 对比）：

```go
// 1. 导入 kitex_gen 生成的数据结构
import ser_app "common/pkg/kitex_gen/ser_app"

// 2. 定义实现类，聚合各业务服务
type AppServiceImpl struct {
    bannerService *ser.BannerService
    // ...
}

// 3. 实现 kitex_gen 定义的 Service 接口，委派到业务层
func (s *AppServiceImpl) CreateBanner(ctx context.Context, req *ser_app.CreateBannerReq) (*ser_app.CreateBannerResp, error) {
    return s.bannerService.CreateBanner(ctx, req)
}
```

**三种 handler 风格差异（实际代码中观察到）：**
- ser-app：handler 定义 struct 聚合多个 Service 实例，每个方法委派到对应 Service
- ser-buser：handler struct 直接持有一个 BUserService 接口（ser_buser.BUserService），更紧耦合
- ser-bcom：handler struct 持有 ComService 接口（ser_bcom.BComService），类似 buser 风格

> 注意：这反映了用户提到的"不同组员开发，代码质量和风格参差不齐"。ser-app 的解耦方式更合理。

### 五、main.go 统一启动模式

**RPC 服务启动流程**（依据 ser-app/main.go 和 ser-buser/main.go 对比）：

| 步骤 | ser-app (main.go) | ser-buser (main.go) | 共性 |
|------|-------------------|---------------------|------|
| 1.日志 | `lg.InitKLog()` 行22 | `lg.InitKLog()` 行19 | 统一 |
| 2.ETCD | `etcd.LoadAndWatch("/slg/")` 行24 | `etcd.InitEtcd()` + `etcd.LoadAndWatch("/slg/conf/")` 行20-21 | 略有差异 |
| 3.基础设施 | `cfg.InitRedis()` 行26 | 无显式Redis初始化 | 按需 |
| 4.端口 | `flag.String("port", etcd.DirectGet(...))` 行29 | 同 | 统一 |
| 5.业务服务 | 创建5个Service实例 行33-37 | 创建1个UserService 行27 | 统一模式 |
| 6.NewServer | `appservice.NewServer(&impl, rpc.InitRpcServerParams()...)` 行40-47 | `buserservice.NewServer(&impl, rpc.InitRpcServerParams()...)` 行28-36 | **统一** |
| 7.启动 | `srv.Run()` 行55 | `srv.Run()` 行39 | 统一 |

**关键函数 InitRpcServerParams**（依据 `common/rpc/init.go` 行24-68）：
- ETCD 服务注册
- 端口复用 `WithReusePort(true)`
- 多路复用 `WithMuxTransport()`
- RPC 访问日志中间件（记录 TraceID、包名、服务名、方法名、耗时）

**网关启动流程**（依据 `gate-back/main.go`）：
InitHLog → LoadAndWatch → InitRedis → InitS3 → 获取端口 → 获取本机IP → 创建ETCD注册 → `server.Default()` 创建Hertz → 日志中间件 → `InitMyRouter(h)` 创建路由组 → `register(h)` 注册所有模块路由 → `h.Spin()` 启动HTTP

### 六、RPC 客户端注册机制

**文件：** `common/rpc/rpc_client.go`

**模式：** 全局变量 + `sync.Once` 单例初始化

```go
var (appService *appservice.Client; appOnce sync.Once)

func AppClient() appservice.Client {
    appOnce.Do(func() {
        app, err := appservice.NewClient(namesp.EtcdAppService, InitRpcClientParams()...)
        appService = &app
    })
    return *appService
}
```

**已注册的客户端**（依据 rpc_client.go 实际代码）：
BUserClient(002), BComClient(003), I18nClient(004), IpClient(005), BLogClient(006), S3Client(007), UserClient(008), AppClient(011), LiveClient(019), MallClient(021) 等共 21 个

**InitRpcClientParams**（依据 `common/rpc/init.go` 行70-88）：
ETCD 服务发现 + 1200s 超时 + 多路复用 + TTHeader 链路传播

### 七、网关路由注册机制

**gate-back 路由注册链路：**
1. `gate-back/main.go` 行71 → `my_router.InitMyRouter(h)` 创建两个路由组：
   - `Open = h.Group("/admin/open", SetHeaderInfo(), WhiteCheck())` — 无需鉴权
   - `Api = h.Group("/admin/api", SetHeaderInfo(), WhiteCheck(), BackAuthMiddleware())` — 需鉴权
2. `gate-back/main.go` 行73 → `register(h)` → `GeneratedRegister(r)`
3. `gate-back/biz/router/register.go` 行22-37 → 逐个调用各模块的 Router 函数
4. 各模块 Router 函数（如 `app.RouterApp(r)`）→ 向 `my_router.Api` / `my_router.Open` 注册具体路由
5. 路由 handler（如 `gate-back/biz/handler/app/banner.go`）→ 调用 `rpc.Handler(ctx, c, rpc.AppClient().CreateBanner)` 统一代理到 RPC 服务

**新增模块需要的网关操作：**
- 在 `gate-back/biz/router/{module}/` 创建路由注册函数
- 在 `gate-back/biz/handler/{module}/` 创建 HTTP handler
- 在 `gate-back/biz/router/register.go` 的 `GeneratedRegister` 中添加调用

### 八、从 0-1 构建新模块的完整流程（经验证）

> 综合代码分析 + 用户反馈修正。用户原话："idl先行，根据原型需求来定义idl...粗粒度的说 编写IDL文件，编写GO文件，执行相关脚本，模块间调用，相关中间件引入使用，业务闭环验证"

**前置条件：** 需求文档 + 产品原型（IDL设计依据）

| 阶段 | 动作 | 产物 | 代码依据 |
|------|------|------|----------|
| **1. 编写IDL** | 在 `common/idl/ser-xxx/` 下编写 service.thrift + model thrift | IDL接口定义 | 参考 `common/idl/ser-app/service.thrift` |
| **2. 执行gen_kitex.sh** | 运行 `common/script/gen_kitex.sh` | `common/pkg/kitex_gen/ser_xxx/`（结构体+Client+Server）+ handler.go/main.go 方法桩 | gen_kitex.sh 行47-49 |
| **3. 创建模块目录** | 创建 `ser-xxx/`，放入 handler.go/main.go，`go mod init`，加入 go.work | 服务模块骨架 | 参考 go.work |
| **4. 建表 + gorm_gen** | TiDB 建表，编写 `internal/gen/gorm_gen.go`，执行生成 | `internal/gorm_gen/`（model/query/repo） | 参考 `ser-app/internal/gen/gorm_gen.go` |
| **5. 实现业务逻辑** | handler→internal/ser/（业务层）→internal/rep/（仓储层），定义 errs/enum/cfg | 内部业务代码 | 参考 ser-app/internal/ 目录结构 |
| **6. RPC客户端注册** | 在 `common/rpc/rpc_client.go` 添加 XxxClient() 工厂 | 服务间通信能力 | 参考 rpc_client.go AppClient() 模式 |
| **7. 网关路由注册** | gate-back/gate-font 添加路由 + handler，register.go 中注册 | HTTP入口 | 参考 gate-back/biz/router/app/ |
| **8. 中间件引入** | 按需接入 Redis/Kafka/NATS/ES/S3/审计日志等 | 基础设施集成 | 参考 common/pkg/ 各组件 |
| **9. 业务闭环验证** | 端到端调试 | 验收 | — |

**用户强调的核心原则：**
- IDL 先行，由需求原型驱动设计
- 脚本生成是关键衔接点，不是手写 kitex_gen 代码
- 分步迭代推进，每步确认后再进行下一步

---

## 深度工程分析——核心链路、架构、细节（2026-02-25 第二轮深度分析）

> 此部分基于对全工程所有模块的逐文件深度分析，每条结论附代码文件:行号依据。

### 一、HTTP请求完整处理链路

#### B端请求链路（以 POST /admin/api/app/banner/create 为例）

```
客户端 HTTP POST
  ↓
Hertz Server (gate-back, 端口来自ETCD/CLI)
  ↓
路由匹配: /admin/api/app/banner/create
  (gate-back/biz/router/app/app_router.go:12)
  ↓
路由组 /admin/api 中间件链 (gate-back/biz/router/my_router/my_router.go:19):
  ① tracer.IGlTracer{}.Start() → 生成UUID TraceID，设置CORS头，注入metainfo
     (common/pkg/tracer/tracer.go:26-36)
  ② SetHeaderInfo() → 提取5个HTTP头注入metainfo上下文传播
     (common/pkg/middleware/info.go:14-34)
     - X-Client-IP (c.ClientIP())
     - X-User-Language (请求头)
     - X-User-Device-ID (请求头)
     - X-User-Device-Type (请求头)
     - X-User-Token (请求头，降级到cookie)
  ③ WhiteCheck() → 调用 rpc.IpClient().OnWhiteList(ctx)
     (common/pkg/middleware/gate-auth.go:39-56)
     不在白名单 → 403拒绝
  ④ BackAuthMiddleware() → B端鉴权
     (common/pkg/middleware/gate-auth.go:92-133)
     - 取token: tracer.GetUserToken(ctx) (行96)
     - Redis查询: rds.Get(token) — 注意B端无前缀 (行102)
     - 反序列化: session.BUserTokenPayload{ID, UserName} (行107-109)
     - 注入UserID/UserName到metainfo (行110-112)
     - URL权限校验: rpc.BUserClient().CheckUrlAuth(ctx, c.FullPath()) (行118)
  ↓
Handler: app.CreateBanner(ctx, c)
  (gate-back/biz/handler/app/banner.go:11-12)
  ↓
rpc.Handler[CreateBannerReq, CreateBannerResp](ctx, c, rpc.AppClient().CreateBanner)
  (common/rpc/handler.go:20-50)
  ① 反射创建REQ实例 (行26-36)
  ② c.BindAndValidate(bindTarget) — HTTP参数绑定+校验 (行39)
  ③ rpcMethod(ctx, reqData) — 执行RPC调用 (行46)
  ④ ret.HttpOne(ctx, resp, rpc.I18nClient(), err) — 统一响应 (行49)
  ↓
RPC调用到 ser-app 服务 (通过ETCD服务发现，TTHeader协议，1200s超时)
  (common/rpc/rpc_client.go:271-283, common/rpc/init.go:70-88)
  ↓
返回 R{TraceId, Code, Msg, Data} JSON响应
```

#### C端请求链路差异（gate-font）

| 环节 | gate-back (B端) | gate-font (C端) |
|------|----------------|----------------|
| 路由前缀 | `/admin/open`, `/admin/api` | `/open`, `/api` |
| IP检查 | WhiteCheck()（默认拒绝，白名单放行）| BlackCheck()（默认放行，黑名单拦截）|
| 鉴权中间件 | BackAuthMiddleware | FontAuthMiddleware |
| Token Redis Key | `token`（无前缀）| `"user:token:" + token` |
| Token Payload | BUserTokenPayload{ID, UserName} | CUserTokenPayload{ID, UID, NickName, Device, DeviceType等7字段} |
| URL权限校验 | 有（CheckUrlAuth，RBAC菜单匹配）| 无 |
| 注册模块数 | 14个 | 8个 |
| 端点总数 | ~147个（130 Api + 17 Open） | ~57个（51 Api + 6 Open） |
| 定位 | 管理后台（含admin模块如buser/item/exp/blog/i18n） | 用户端（user/app/kyc/shortvideos/live/rtc） |

### 二、common/pkg 核心模块详解

#### 2.1 tracer — 链路追踪与上下文传播

**文件：** `common/pkg/tracer/tracer.go`

**8个上下文传播常量**（行12-21）：
```
X-Trace-Id, X-Client-IP, X-User-Token, X-User-ID,
X-User-Name, X-User-Language, X-User-Device-ID, X-User-Device-Type
```

**IGlTracer.Start**（行26-36）：
- 设置CORS头（Allow-Origin:*, Allow-Methods:GET/POST/PUT/DELETE/OPTIONS）
- 生成UUID作为TraceID
- 写入响应头 + metainfo上下文（跨RPC传播）

#### 2.2 ret — 统一响应格式与错误处理

**文件：** `common/pkg/ret/project_ret.go`

**R结构体**（行19-24）：`{TraceId string, Code errs.ErrorType, Msg string, Data interface{}}`

**BizErr**（行32-41）：
- 创建Kitex BizStatusError
- 若无msg参数，从Redis Hash `i18n:hs:zhcn` 按错误码取中文消息
- 用法：`ret.BizErr(ctx, errs.UserNotFound)` 或 `ret.BizErr(ctx, errs.UserNotFound, "自定义消息")`

**HttpErr**（行93-125）：
- 识别BizStatusError → 调用i18n服务 `trans.LangTransOne()` 翻译错误码
- 翻译为空则降级使用原始中文消息
- 非BizStatusError → 返回通用操作错误码

**HttpOne**（行128-133）：有错误走HttpErr，无错误走HttpOk — 调用方无需判断err

#### 2.3 session — Token Payload定义

**文件：** `common/pkg/session/login.go`

| 字段 | CUserTokenPayload (C端, 行3-11) | BUserTokenPayload (B端, 行12-15) |
|------|-------------------------------|-------------------------------|
| ID | int64 | int64 |
| UID | int64 | — |
| LastLoginIp | string | — |
| LastLoginTime | int64 | — |
| Device | string | — |
| DeviceType | string | — |
| NickName | string | — |
| UserName | — | string (json tag = "uid") |

#### 2.4 etcd — 配置中心（COW缓存 + Hook机制）

**文件：** `common/pkg/etcd/etcd_cache.go`

**COW缓存模式**（行64-103 watchLoop）：
1. Watch ETCD前缀变更
2. 读取当前map → 拷贝新map → 应用变更 → `atomic.Value.Store(newMap)` 原子替换
3. 触发已注册的Hook回调（行97-100）
4. 无锁读：高并发下通过原子操作保证一致性

**Hook机制**（行152-168）：`Hook(key, func(val))` 注册配置变更回调

**DirectGet vs Get**：
- Get（行115-128）：先查COW缓存，未命中则降级DirectGet
- DirectGet：直接查ETCD，用于未在监听前缀内的Key

#### 2.5 middleware — 中间件体系

**gate-auth.go 四大中间件**（行号见上文链路图）：
- BlackCheck(21-37)：C端IP黑名单，命中拦截
- WhiteCheck(39-56)：B端IP白名单，不在名单拦截
- FontAuthMiddleware(58-90)：C端Token校验（Redis key=`user:token:` + token）
- BackAuthMiddleware(92-133)：B端Token校验（Redis key=token本身）+ URL权限校验

**info.go SetHeaderInfo**（14-34）：提取5个HTTP头 → metainfo注入 → 跨RPC传播

#### 2.6 db/mysql — GORM配置

**文件：** `common/pkg/db/mysql/mysql.go`

**关键配置**（行39-45）：
- `PrepareStmt: true` — 预编译语句复用
- `SkipDefaultTransaction: true` — 跳过默认事务包裹，提升性能
- 慢查询阈值：`200ms`（grom_log.go:10）
- 连接池：MaxOpen/MaxIdle/MaxLifetime 可配置

#### 2.7 rds — Redis客户端

**文件：** `common/pkg/rds/redis_init.go`

- UniversalClient模式：支持单机/集群自动切换（行34逗号分隔地址）
- `RouteByLatency: true`（行46）— 请求路由到最低延迟节点
- `MaxRedirects: 8`（行47）— 集群重定向
- 读写超时2s（行42-43）
- 工具方法：Get/Set/Del/HGet/HSet/SetNX(分布式锁)/Pipeline(批量)/ZAdd/SAdd等

#### 2.8 lg — 日志系统

**文件：** `common/pkg/lg/log.go`

| 函数 | 用途 | 框架 |
|------|------|------|
| InitKLog (行56-70) | RPC服务日志 | Kitex + Zap |
| InitHLog (行73-87) | HTTP网关日志 | Hertz + Zerolog |

**日志轮转**（行146-154）：lumberjack — 100MB/文件，10个备份，30天保留，gzip压缩

#### 2.9 其他基础设施模块

| 模块 | 文件 | 核心能力 |
|------|------|---------|
| kafka | `common/pkg/kafka/producer.go` | Sarama AsyncProducer，Hash分区，幂等+10次重试，构建器模式发送 |
| nats | `common/pkg/nats/nats.go` | NATS+JetStream，Namespace前缀隔离，延迟发布(Nats-Delay头)，QueueSubscribe负载均衡，批量消费(信号量并发控制) |
| es | `common/pkg/es/client.go` | ES TypedClient，TLS 1.2+，1000条/批BulkSave，自动重试502/503/504/429，分页+排序查询 |
| mys3 | `common/pkg/mys3/mys3.go` | AWS S3 + PresignClient，PathStyle URL，Region/Domain从ETCD获取 |
| i18n | `common/pkg/i18n/i18n.go` | go-i18n库封装，Bundle+Localizer模式，支持TOML/JSON/YAML格式 |

### 三、common/rpc 统一通信层

#### 3.1 handler.go — 三种通用Handler

**文件：** `common/rpc/handler.go`

| Handler | 行号 | 签名 | 场景 |
|---------|------|------|------|
| Handler[REQ, RESP] | 20-50 | `(ctx, c, rpcMethod)` | 标准CRUD（~95%接口）|
| HandlerNoParam[RESP] | 52-63 | `(ctx, c, rpcMethod)` | 无参数查询（如枚举列表）|
| HandlerWithCSV[REQ, RESP] | 65-109 | `(ctx, c, rpcMethod)` | 文件下载（RESP需实现CSVResponse接口：GetFilename()+GetData()）|

**Handler核心流程**：反射创建REQ → BindAndValidate → RPC调用 → ret.HttpOne统一响应

#### 3.2 rpc_client.go — 21个RPC客户端工厂

**文件：** `common/rpc/rpc_client.go`（行134-418）

**模式：** `sync.Once` 单例 + ETCD服务发现 + 懒初始化

| 序号 | 客户端 | 服务名常量 |
|------|--------|-----------|
| 002 | BUserClient | ser-buser-service |
| 003 | BComClient | ser-bcom-service |
| 004 | I18nClient | ser-i18n-service |
| 005 | IpClient | ser-ip-service |
| 006 | BLogClient | ser-blog-service |
| 007 | S3Client | ser-s3-service |
| 008 | UserClient | ser-user-service |
| 009 | ExpClient | ser-exp-service |
| 010 | ItemClient | ser-item-service |
| 011 | AppClient | ser-app-service |
| 012 | KycClient | ser-kyc-service |
| 014 | RecommendClient | ser-recommend-service |
| 015 | ShortVideosClient | ser-shortvideos-service |
| 016 | ContentCenterClient | ser-contentcenter-service |
| 017 | WebSocketClient | ser-socket-service |
| 018 | RtcClient | ser-rtc-service |
| 019 | LiveClient | ser-live-service |
| 020 | CronClient | ser-cron-service |
| 021 | MallClient | ser-mall-service |

#### 3.3 init.go — RPC初始化参数

**InitRpcServerParams**（行24-68）：ETCD注册 + ReusePort + MuxTransport + 访问日志中间件（记录TraceID/包名/服务名/方法名/耗时/调用方信息）

**InitRpcClientParams**（行70-88）：ETCD发现 + **1200s超时**(20分钟) + MuxConnection + TTHeader协议

### 四、网关架构详解

#### 4.1 gate-back 启动流程

**文件：** `gate-back/main.go`（行1-76）

```
InitHLog(行35) → LoadAndWatch("/slg/conf/")(行36) → InitRedis(行37) → InitS3(行39-44)
→ 解析端口(行47-48) → 获取本机IP(行51) → ETCD注册 → Hertz server.Default(行53-66)
→ 20MB请求体限制(行63) → Tracer中间件(行65) → 日志中间件(行69)
→ InitMyRouter(行71) → register注册全部路由(行73) → h.Spin()(行75)
```

#### 4.2 gate-back 路由注册

**文件：** `gate-back/biz/router/register.go`（行22-37）

14个模块路由注册：buser, bcom, i18n, blog, user, enum, kyc, item, app, exp, s3, live, shortvideos, rtc

**端点统计：**

| 模块 | 端点数 | 路由基础 |
|------|--------|---------|
| buser | 37 | /admin/api/buser/ + /admin/open/buser/ |
| app | 20 | /admin/api/app/ |
| rtc | 20 | /admin/api/rtc/ |
| bcom | 13 | /admin/api/bcom/ + /admin/open/bcom/ |
| live | 12 | /admin/api/live/ |
| i18n | 10 | /admin/api/i18n/ + /admin/open/i18n/ |
| item | 9 | /admin/api/item/ |
| user | 6 | /admin/open/user/ |
| exp | 5 | /admin/api/exp/ |
| shortvideos | 5 | /admin/api/shortvideos/ |
| s3 | 4 | /admin/api/s3/ + /admin/open/s3/ |
| kyc | 3 | /admin/api/kyc/ |
| blog | 2 | /admin/api/blog/ |
| enum | 1 | /admin/open/enum/ |
| **合计** | **~147** | 130 Api + 17 Open |

#### 4.3 gate-font 路由注册

**文件：** `gate-font/biz/router/register.go`（行19-27）

8个模块路由注册：buser(空), user, app, enum, kyc, shortvideos, rtc, live

**端点统计：** ~57个（51 Api + 6 Open）

#### 4.4 网关Handler模式

**绝大多数Handler**（~95%）：
```go
func CreateBanner(ctx context.Context, c *app.RequestContext) {
    rpc.Handler(ctx, c, rpc.AppClient().CreateBanner)  // 一行委托
}
```

**少数特殊Handler**：
- HandlerNoParam：无请求体的查询（如SysRoleList）
- HandlerWithCSV：文件导出下载
- 手动实现：需要额外逻辑的（如Login手动绑定、MenuUserTree提取userId）

### 五、各服务核心业务逻辑与模式

#### 5.1 ser-app（Banner/分类/内容策略，23个RPC方法）

**handler.go**（行13-19）：AppServiceImpl聚合5个Service实例

**Banner缓存策略**（internal/cache/banner.go）：
- Redis Hash Key模式：`banner:list:{pagePosition}:{terminal}`
- 空值缓存防穿透（行47-52）：无数据时也缓存空列表
- 全量失效（DelAll）：枚举6个position × 3个terminal = 18个Key全部删除
- 缺失值用 `_all` 占位符

**跨服务调用**：ser-s3（图片上传/删除）、ser-recommend（内容策略）、ser-contentcenter（内容策略）

**错误码**：6011000-6011204

#### 5.2 ser-buser（RBAC权限/登录认证，26个RPC方法）

**handler.go**（行10-12）：嵌入BUserService接口（紧耦合风格）

**RBAC模型**（5张表）：
```
sys_user → sys_user_role → sys_role → sys_role_menu → sys_menu
```

**3级角色体系**（SuperAdminFlag字段）：
- `"0"` 普通角色 → 需菜单权限校验
- `"1"` 系统管理 → 跳过权限校验
- `"2"` 超级管理 → 跳过权限校验

**B端登录流程**（internal/ser/user_service.go:279-364）：
```
① IP白名单校验 → rpc.IpClient().OnWhiteList(ctx) (行283-291)
② 查询用户 → repo.QueryUserByUserName(ctx, req.UserName) (行293-297)
③ 状态校验 → status != enum.Fail (行298-301)
④ 密码校验 → utils.CheckPassword(req.Password, *user.Password, *user.Salt) (行302-305)
   密码算法：MD5(password+salt) → bcrypt (common/pkg/utils/register_util.go)
⑤ Google TOTP校验 → totp.Validate(code, secret) (行306-322)
   库：github.com/pquerna/otp/totp
⑥ Token生成存储 → utils.GenToken() → Redis SET token payload 30天TTL (行332-339)
   Payload: BUserTokenPayload{ID, UserName}
   Redis Key: token字符串本身（无前缀）
⑦ 审计日志 → rpc.BLogClient().AddLoginLog()
⑧ 响应 → Token + ID + BindGoogle + RoleId + RoleName
```

**CheckUrlAuth链路**（internal/ser/menu_service.go:258-274）：
```
① 取userId → tracer.GetUserId(ctx)
② 查角色 → repo.GetRoleByUserId(ctx, userId)
③ 角色状态 → status=0则拒绝
④ SuperAdminFlag "1"/"2" → 直接放行
⑤ SuperAdminFlag "0" → repo.CheckUrlAuth(ctx, url, role.ID)
   → 查sys_role_menu + sys_menu 匹配URL路径
```

**跨服务调用**：ser-ip（白名单校验）、ser-blog（登录/操作日志）

**错误码**：6002000-6002053

#### 5.3 ser-bcom（开关配置/黑白名单门面，18个RPC方法）

**架构角色**：**门面/适配层**，将黑白名单操作代理到ser-ip

**ETCD开关配置**（internal/ser/switch_service.go:22-70）：13个开关Key

| 分类 | 开关项 |
|------|--------|
| Android注册 | Guest/Email/Phone 3个开关 |
| iOS注册 | Phone/Email/Guest 3个开关 |
| H5注册 | Phone/Email/Guest 3个开关 |
| 访问控制 | BackendWhite（B端白名单）、ClientBlack（C端黑名单）|
| 验证控制 | GoogleAuth（谷歌验证）、ClientRobot（机器人检测）|

**开关配置结构体**：`{Status int32, Code string, OperatorId int64, Operator string, UpdateAt int64}`

**枚举查询**：数据库存储逗号分隔的value/label，查询时split解析

**错误码**：6003000-6003030

#### 5.4 ser-i18n（多语言翻译，11个RPC方法）

**Redis Hash Key模式**（internal/enum/enum.go）：
```
i18n:hs:zhcn  i18n:hs:zhtw  i18n:hs:en
i18n:hs:id    i18n:hs:vi    i18n:hs:th
```

**翻译查询**：每个语言独立Hash，Key=翻译ID/错误码，Value=翻译文本

**批量操作**：BatchSave先查重去空 → copier.Copy映射 → 批量入库 → 批量写Redis

**导出格式**：CSV（UTF-8 BOM）、JSON（MarshalIndent）、XML

**错误码**：6004000-6004105

#### 5.5 ser-ip（IP黑白名单，10个RPC方法）

**OnBlackList逻辑**（internal/ser/c_black_service.go:35-56）：
```
① ETCD取开关配置（type="1" → ClientBlack）
② 开关Status=0 → 直接返回false（不拦截）
③ 第三方IP分析服务获取CountryCode
④ 数据库匹配国家码是否在黑名单
```

**OnWhiteList逻辑**：类似，type="2" → BackendWhite，匹配IP地址

**ISO 3166校验**：`utils.IsValidISO3166Alpha2(isoCode)` 校验国家代码 + `utils.CountryCNByISO()` 映射中文国名

**第三方IP分析**（internal/mod/ip_analysis.go）：返回 `{Ip, Asn, AsName, AsDomain, CountryCode, Country, ContinentCode, Continent}`

**错误码**：6005000-6005016

#### 5.6 ser-item（道具配置/发布，11个RPC方法）

**特殊架构：Domain层模式**（internal/domain/）— 唯一使用此模式的服务

| 组件 | 文件 | 职责 |
|------|------|------|
| Validator | item_validator.go | 名称(3-36字符)/URL/价格校验 |
| Mapper | item_mapper.go | Model↔API响应转换，URL全路径重建 |
| CacheManager | item_cache.go | Redis缓存 `item:list:type:{itemType}` + `item:list:ids:{hash}` |
| LogBuilder | item_log_builder.go | 审计日志构造 |

**双表设计**：Item（基础属性：名称/图标/价格/类型）+ ItemPublish（发布状态：上下架/标签/排序/可见性）

**错误码**：6010000-6010021

#### 5.7 ser-s3（文件管理，10个RPC方法）

**文件类型魔数检测**（internal/ser/s3_service.go:262-318）：
```
GIF: "GIF87a"/"GIF89a"
PNG: 89 50 4E 47 (0x89 + "PNG")
JPEG: FF D8 FF
MP4: bytes[4:8] = "ftyp"
PAG: bytes[0:3] = "PAG"
JSON: 首字符 '{' 或 '['
```

**预签名URL**：AWS SDK v4 signer，上传60min有效，下载15min有效

**多段上传**：AWS S3 Multipart Upload API

**错误码**：6007000-6007112

### 六、跨服务依赖关系图

```
gate-back ──→ [所有RPC服务] (通过rpc.XxxClient())
gate-font ──→ [user/app/kyc/shortvideos/live/rtc等]

ser-app
 ├─→ ser-s3 (图片上传/删除)
 ├─→ ser-recommend (内容策略)
 └─→ ser-contentcenter (内容策略)

ser-buser
 ├─→ ser-ip (登录时白名单校验)
 └─→ ser-blog (登录日志+操作日志)

ser-bcom
 ├─→ ser-ip (黑白名单CRUD代理)
 └─→ ser-buser (用户信息查询)

ser-ip
 └─→ third-party IP分析服务

ser-item
 ├─→ ser-buser (更新人信息)
 └─→ ser-s3 (图标上传)

ser-i18n → 独立（无服务调用）
ser-s3 → 独立（直连AWS S3）
```

### 七、错误码体系

| 服务 | 前缀 | 范围 | 说明 |
|------|------|------|------|
| ser-buser | 6002 | 6002000-6002053 | 用户/角色/菜单/登录 |
| ser-bcom | 6003 | 6003000-6003030 | 开关/黑白名单/枚举/邮件 |
| ser-i18n | 6004 | 6004000-6004105 | 翻译操作/参数 |
| ser-ip | 6005 | 6005000-6005016 | IP黑白名单 |
| ser-s3 | 6007 | 6007000-6007112 | 文件操作/参数 |
| ser-item | 6010 | 6010000-6010021 | 道具操作 |
| ser-app | 6011 | 6011000-6011204 | Banner/分类/内容策略 |

**错误码使用模式**：`ret.BizErr(ctx, errs.XxxError)` → 创建BizStatusError → 网关HttpErr自动i18n翻译

### 八、关键代码约定与模式

#### 8.1 软删除
- 字段：`delete_at`（毫秒时间戳，0=未删除）+ `delete_by`（操作人ID）
- 查询条件：`WHERE delete_at = 0`

#### 8.2 审计日志
- 所有增删改操作调用 `rpc.BLogClient().AddActLog(ctx, &ser_blog.AddActLogReq{...})`
- 登录操作调用 `rpc.BLogClient().AddLoginLog()`

#### 8.3 事务模式
- GORM事务：`query.Q.Transaction(func(tx *query.Query) error { ... })`
- 批量操作默认200条/批

#### 8.4 分页规范
- 请求：`pageNo`（从1开始）+ `pageSize`（15/30/50/100）
- 响应：`pageList` + `total` + `pageNo` + `pageSize`

#### 8.5 Redis Key命名空间
```
user:token:{token}     — C端Token
{token}                — B端Token（直接存储）
banner:list:{pos}:{terminal} — Banner缓存
item:list:type:{type}  — 道具类型缓存
item:list:ids:{hash}   — 道具ID缓存
i18n:hs:{lang}         — 翻译Hash
enum:{key}             — 枚举缓存
IP:{ip}                — IP分析缓存
exp:vip:list           — VIP等级缓存
```

#### 8.6 metainfo上下文传播
通过 `metainfo.WithPersistentValue()` 在RPC调用间传播8个头信息（TraceID/ClientIP/Token/UserID/UserName/UserLang/DeviceID/DeviceType），实现全链路追踪和用户上下文透传

#### 8.7 Handler风格差异（实际观察）
- **ser-app**：handler struct聚合多个Service实例，解耦较好（推荐风格）
- **ser-buser/ser-bcom**：handler struct嵌入单个Service接口，紧耦合
- 反映"不同组员开发，代码质量和风格参差不齐"（用户原话）
