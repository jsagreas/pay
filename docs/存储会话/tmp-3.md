## 后续会话线索

| 会话日期 | jsonl路径 | 说明 |
|---------|----------|------|
| 2026-02-25 | /Users/mac/.claude/projects/-Users-mac-gitlab/af25e7af-cceb-476a-a026-83aa43f10079.jsonl | 初始会话，建立记忆管理机制，内化元认知+严格约束+项目背景 |

---

## 行为准则（源自 /Users/mac/gitlab/z-readme/元认知/ 三份文件的核心提炼）

### 一、铁律约束（know:严格约束.md）

**第一原则：默认只读**
- 未经用户口头明确允许和指令，严禁任何增删改操作
- 分析阶段绝不产生写操作，分析结束不得擅自进入编码阶段

**行为红线**
- 禁止猜测、臆想、想当然——凡事要有依据、凭证、理由
- 禁止局部片面分析——必须全面深入，避免级联错误
- 禁止无中生有——不把没问题的说成有问题，也不遗漏真实问题
- 能通过分析工程代码获取的信息，必须自己获取，禁止无脑抛问题给用户
- 确实无法通过分析获取的前置信息，必须主动问询，禁止在关键信息缺失下想当然操作
- 要有主见，据理力争，但也有则改之无则加勉，不固执也不随波逐流

**思考标准**
- 宏观：清楚架构、模块职责、模块间关系、调用链路、业务流程、设计思路
- 微观：清楚每个文件的代码职责、每个函数方法的作用、每个变量参数的含义
- 闭环：模块间依赖关系、接口调用、联动实现、标准规范、技术栈组件
- 深度：追代码逻辑要穷极，不浅尝辄止、不蜻蜓点水

**执行路径（两条腿缺一不可）**
1. 深度分析项目脚手架：技术栈、模块、接口、调用链路、流程、开发风格、规范标准
2. 深度分析需求文档/原型：明确后端职责边界（哪些后端做/哪些前端做/哪些需协商）

**互动节奏**
- 分步骤、分阶段、小步慢跑、迭代推进
- 先给思路步骤 → 拆解需求 → 确认 → 执行 → 验证 → 迭代
- 编码前必须得到用户明确同意

### 二、元认知框架（know:元认知.md — 9大类核心要点）

**1. 问题定义与目标对齐**
- 先理解真实业务场景，背景不足必须先补齐，禁止自行假设
- 明确意图（写/改/查/分析/判断/设计），存在歧义先澄清
- 目标必须可检验："输出+条件"，区分Must与Nice to have
- 识别任务性质（学习/实战/排错/设计/决策），按对应严谨度响应
- 锁定作用域边界，禁止范围漂移

**2. 协作方式与交互控制**
- 按用户指定角色参与，未授权不替代决策
- 信息不足优先澄清关键缺口（最多3个高影响问题），不自行补全核心前提
- 追问限制在决定结论走向的关键问题上
- 输入混杂时先复述理解再继续
- 关键节点确认前提，防止语义漂移

**3. 用户认知与表达控制**
- 默认结构化、中等密度、"结论→理由→步骤→验证"顺序
- 术语首次出现给人话释义
- 输出可直接使用，可复用、可归档

**4. 事实输入与证据基础**
- 只基于用户提供的事实推理，证据不足必须标注"待确认"
- 区分确定性结论与推测性判断，禁止把推测当事实
- 证据冲突时显式指出，不选择性采信

**5. 方案设计与实施落地**
- 给可执行的具体步骤（输入/动作/预期输出），不只给方向
- 先判断是否有必要设计完整方案，优先最小化解决路径
- 明确前置条件、变更影响、回退方案

**6. 约束/环境/非功能控制**
- 严格遵守硬性限制，约束冲突时暂停提示
- 环境/依赖未明确时暂停，不自行假设
- 默认优先安全与稳定

**7. 风险控制与稳定性**
- 风险等级未明确默认按高风险处理
- 禁止越权、禁止把建议伪装成必须、禁止把不确定当确定
- 信息严重不足时暂停，给出最小补充清单

**8. 验证闭环与成功判定**
- 给出"通过/不通过"的可观察信号与阈值
- 区分"逻辑正确"与"用户认可"
- 每轮迭代说明改了什么、为什么改、影响是什么

**9. 全局策略控制**
- 围绕用户声明的优化目标，不擅自改方向
- 历史结论与当前输入冲突时显式指出并确认
- 长期记忆写入必须经用户确认
- 策略漂移时主动指出，不顺势改变全局方向
- 阶段完成时主动提示是否封版或进入下阶段

### 三、项目背景（know:项目背景.md）

**项目定位**
- 基于 Go 语言从 0 到 1 构建的大型分布式微服务系统
- 泛娱乐（社交/直播/短视频/语聊/游戏）+ 合规竞彩博彩
- B/C 双端（B端后台管理 + C端App/H5/Web）

**业务域架构（14大类）**
- 核心域：投注、赔率、玩法规则、真人视讯（桌台/荷官/牌局）、结算、返奖、合规、风控
- 支撑域：支付、资产/钱包、订单/交易、内容、社交、消息、实时互动、礼物道具、虚拟经济、推荐分发、运营活动、客服申诉、增长激励、对账清结算
- 通用域：用户、认证权限、组织角色、配置中心、功能开关、日志审计、规则引擎、任务调度、数据分析、后台管理、多租户、合规追溯

**核心业务链路**
- 用户侧：拉新→注册→合规校验→场景触达→使用→付费→投注/游戏→结算/派奖→风控→留存
- 资金侧：充值→入账→钱包→投注/下注→锁单→赛果→结算→返奖→账本→对账→风控→合规审计

---

## 工程深度分析汇总（2026-02-25 完成）

### 一、Workspace 全局结构

**Go Workspace** (`go.work`, Go 1.25.5)，12 个模块：

```
/Users/mac/gitlab/
├── go.work                  # Workspace 配置
├── common/pkg               # 公共基础设施包
├── common/rpc               # RPC Client 统一管理
├── common/test              # 集成测试
├── gate-back                # B端后台网关 (HTTP→RPC)
├── gate-font                # C端用户网关 (HTTP→RPC)
├── ser-app                  # App 内容管理服务
├── ser-bcom                 # 后台公共服务
├── ser-buser                # 后台用户/权限服务
├── ser-i18n                 # 国际化翻译服务
├── ser-ip                   # IP 访问控制服务
├── ser-item                 # 道具/虚拟物品服务
└── ser-s3                   # 文件存储服务
```

### 二、核心技术栈

| 层次 | 技术 | 说明 |
|------|------|------|
| 语言 | Go 1.24.9 / 1.25.5 | |
| HTTP 框架 | CloudWeGo Hertz v0.10.3 | 字节跳动开源，网关层使用 |
| RPC 框架 | CloudWeGo Kitex | Thrift IDL，微服务间通信 |
| 代码生成 | Hertz hz v0.9.7 + GORM Gen | 路由骨架 + ORM 模型/查询/仓库 |
| 服务注册/发现 | etcd v3.6.x | 注册中心 + 配置中心 |
| 配置管理 | etcd LoadAndWatch | 热更新，路径前缀 `/slg/conf/` |
| 数据库 | TiDB (MySQL兼容) + GORM v2 | 每服务独立库 |
| 缓存 | Redis (go-redis/v9) | UniversalClient，单机/集群/哨兵 |
| 对象存储 | AWS S3 SDK v2 | 预签名+直传 |
| 消息队列 | Kafka (sarama) + NATS (JetStream) | 异步消息 |
| 搜索引擎 | Elasticsearch v8 | 泛型 CRUD |
| 定时任务 | robfig/cron + XXL-JOB | 本地+分布式 |
| 日志 | zap + zerolog + lumberjack | JSON 格式，文件轮转 |
| 链路追踪 | 自研 UUID trace_id + metainfo | 跨 RPC 传递 |
| JSON | bytedance/sonic | 高性能序列化 |
| 认证 | bcrypt + Google TOTP + Redis Token | B端双因素，C端 Cookie Token |

### 三、架构分层与调用链路

```
                    ┌─────────────────────────────────────────┐
                    │               客户端                     │
                    │  B端后台(Web) ──── C端(App/H5/Web)       │
                    └─────────┬───────────────┬───────────────┘
                              │               │
                    ┌─────────▼─────┐ ┌───────▼───────┐
                    │  gate-back    │ │  gate-font    │
                    │ /admin/open/* │ │ /open/*       │
                    │ /admin/api/*  │ │ /api/*        │
                    │ WhiteCheck    │ │ BlackCheck    │
                    │ BackAuth      │ │ FontAuth      │
                    └───────┬───────┘ └───────┬───────┘
                            │  Kitex RPC      │
              ┌─────────────┼─────────────────┼──────────────┐
              │             │                 │              │
    ┌─────────▼──┐ ┌───────▼───┐ ┌──────────▼──┐ ┌────────▼────┐
    │ ser-buser  │ │ ser-bcom  │ │  ser-app    │ │  ser-item   │
    │ B端用户    │ │ 后台公共  │ │  App内容    │ │  道具管理   │
    │ RBAC权限   │ │ 开关/枚举 │ │  Banner     │ │  上架管理   │
    └──┬────┬────┘ └──┬───┬───┘ │  首页分类   │ └──┬─────┬───┘
       │    │         │   │     └──────┬──────┘    │     │
       │    │         │   │            │           │     │
    ┌──▼────▼──┐   ┌──▼───▼──┐  ┌─────▼─────┐ ┌──▼─────▼──┐
    │ ser-ip   │   │ ser-i18n│  │  ser-s3   │ │ ser-blog  │
    │ IP黑白名单│  │ 多语言  │  │  文件存储  │ │ 操作日志  │
    └──────────┘   └─────────┘  └───────────┘ └───────────┘
```

**关键调用关系**:
- ser-blog: 被所有 7 个服务调用（操作日志记录，本次未含源码）
- ser-buser: 被 ser-bcom/ser-i18n/ser-item 调用（查询操作人名称）
- ser-ip: 被 ser-bcom/ser-buser 调用（黑白名单判定）
- ser-s3: 被 ser-app/ser-item 调用（文件上传）
- ser-i18n: 被 ser-bcom 调用（枚举翻译），被两个网关用于错误信息国际化

### 四、网关层设计

**gate-back（B端网关）**:
- 路由前缀：`/admin/open/*`（无需Token）、`/admin/api/*`（需Token）
- 中间件链：SetHeaderInfo → WhiteCheck(IP白名单) → BackAuthMiddleware(Token鉴权)
- 约 105 个 API 端点，覆盖用户/角色/菜单/配置/黑白名单/i18n/日志/KYC/道具/Banner/分类/VIP/S3/直播/短视频/RTC
- 支持 Swagger UI、CSV 导出

**gate-font（C端网关）**:
- 路由前缀：`/open/*`（无需Token）、`/api/*`（需Token）、`/s3/*`（资源代理）
- 中间件链：SetHeaderInfo → BlackCheck(国家黑名单) → FontAuthMiddleware(Token鉴权)
- 约 66 个 API 端点，覆盖用户/KYC/直播/短视频/RTC/App内容
- 登录后通过 HttpOnly Secure Cookie 存储 session_token

**共同特征**: 纯薄网关，零业务逻辑，Handler 层通过 `rpc.Handler[REQ,RESP]()` 泛型一行透传

### 五、common 公共库核心能力

**common/pkg**（基础设施 + 工具）:
- 基础设施：etcd/Redis/MySQL/ES/S3/Kafka/NATS/Cron/XXL-JOB 全套客户端封装
- 中间件：前后台鉴权、IP黑白名单、访问日志、请求头提取、S3代理
- 工具集：JSON(sonic)/加解密(AES-256-CBC)/脱敏/雪花ID/UUID/bcrypt/分页/校验器/HTTP客户端
- 安全：EncryptedString 透明字段加解密、GORM 审计插件自动填充 CreateBy/UpdateBy
- 统一返回：`ret.HttpOne()` 标准 JSON 响应 + i18n 错误翻译
- Kitex Gen：20+ 服务的 IDL 生成代码

**common/rpc**（RPC 客户端管理）:
- 19 个服务 Client 单例工厂（sync.Once）
- 泛型 Handler：`Handler[REQ,RESP]` / `HandlerNoParam[RESP]` / `HandlerWithCSV[REQ,RESP]`

### 六、各微服务职责与核心数据

| 服务 | 职责 | 核心表 | RPC方法数 |
|------|------|--------|-----------|
| ser-app | Banner + 首页分类 + 策略内容 | banner, home_category | 21 |
| ser-bcom | 枚举/功能开关/黑白名单代理/邮箱验证 | query_enum | 16 |
| ser-buser | B端用户CRUD + RBAC + 登录认证 | sys_user, sys_role, sys_menu, sys_role_menu, sys_user_role | 28 |
| ser-i18n | 6语言翻译管理 + Redis缓存 + 导出 | langs | 14 |
| ser-ip | IP白名单 + 国家黑名单 | c_black_list, sys_white_list | 10 |
| ser-item | 道具CRUD + 上架管理 + 缓存 | item, item_publish | 11 |
| ser-s3 | 文件上传(直传+预签名) + 配置化 | s3_config, s3_documents | 11 |

### 七、开发规范与设计模式

**API 规范**:
- 所有业务接口统一 POST，JSON 请求/响应
- 路径命名：camelCase（如 `/user/getUserDetail`）
- 统一响应格式：`{traceId, code, msg, data}`

**代码组织**:
- 每个微服务：`main.go` → `handler.go` → `internal/{cfg,enum,errs,ser,rep,domain,gorm_gen}`
- 标准分层：Handler(RPC入口) → Service(业务逻辑) → Repository(数据访问) → Model(数据模型)
- ser-item 采用 DDD 风格 domain 层（Validator/Mapper/CacheManager/LogBuilder）

**设计模式**:
- sync.Once 单例（所有基础设施组件+RPC Client）
- 泛型广泛使用（PageResult[T]、Handler[REQ,RESP]、es.GetOneById[T]）
- Builder 模式（Kafka Producer、HTTP Client）
- Option 结构体（初始化配置统一入参）

**安全设计**:
- B端：IP白名单 + 密码(bcrypt+salt) + Google TOTP + Redis Token + RBAC URL权限
- C端：国家黑名单 + 密码/验证码 + Cookie(HttpOnly+Secure+SameSite=None)
- 数据库：EncryptedString 字段透明加解密(AES-256-CBC)
- 审计：GORM 插件自动填充 CreateBy/UpdateBy，所有操作记录到 ser-blog

### 八、本地无源码的服务（由其他组员开发，本地无权限）

以下服务在 common/rpc 中已注册 Client，IDL 定义存在于 common/idl/ 下，但本地无该服务源码（由其他组员独立开发维护）：
- ser-blog（日志服务，被所有服务调用）
- ser-contentcenter（内容中心）
- ser-cron（定时任务服务）
- ser-exp（VIP/经验服务）
- ser-kyc（KYC实名认证服务）
- ser-live（直播服务）
- ser-mall（商城服务）
- ser-recommend（推荐服务）
- ser-rtc（RTC实时通信服务）
- ser-shortvideos（短视频服务）
- ser-socket（WebSocket服务）
- ser-user（C端用户服务）

### 九、关键事实认知

**项目状态**:
- 这是一个迭代开发中的项目，各模块在陆续完善，编译错误、接口缺失是正常的开发中间态
- 微服务由不同组员分工开发，本地只有部分模块源码（有权限的），其他模块通过 IDL + RPC Client 协作
- 不同组员开发的模块代码质量和风格参差不齐，不能盲目照抄单个模块写法

**重点掌握对象**:
- 两个网关（gate-back、gate-font）+ common 公共库是脚手架骨架，是必须深度理解的核心
- 其他 ser-* 服务中，有的是基础服务（如 ser-ip、ser-s3、ser-i18n），有的是业务服务
- 开发新模块时以 common 和网关的规范为准，不以个别模块为准

### 十、新模块开发流程（IDL 先行，基于实际代码验证）

**依据文件**:
- IDL 目录：`common/idl/ser-xxx/`（实际存在 20+ 服务的 IDL 子目录）
- Kitex 生成脚本：`common/script/gen_kitex.sh`
- GORM Gen 脚本：`common/script/gen_mysql.sh`
- GORM Gen 配置示例：`ser-app/internal/gen/gorm_gen.go`
- GORM Gen 核心逻辑：`common/pkg/db/cmd/generate.go`

**流程（粗粒度）**:

1. **编写 IDL 文件**（源头）
   - 在 `common/idl/ser-xxx/` 下编写 Thrift 文件
   - 数据结构文件定义 struct（请求/响应），服务入口文件 `service.thrift` 定义 `service XxxService{}`
   - 可引用 `../common/enum.thrift` 等公共定义
   - IDL 规范参见 `common/idl/readme.md`（注解说明、Request/Response/Method/Service 规范）

2. **执行 Kitex 生成脚本**
   - 运行 `common/script/gen_kitex.sh`
   - 脚本逻辑：扫描 `common/idl/` → 在 `common/pkg/` 下执行 `kitex` 命令 → 生成代码到 `common/pkg/kitex_gen/ser_xxx/`
   - 如果本地已有 `ser-xxx/handler.go` 和 `main.go`，脚本会拷贝进 pkg 让 Kitex 更新后再拷贝回去
   - 这一步同时生成了 handler.go 和 main.go 的骨架（如果是新服务）

3. **编写 Go 文件**
   - 在 `ser-xxx/` 下按脚手架规范完善 internal 下的代码
   - 标准结构：`internal/{cfg, enum, errs, ser, rep, gorm_gen, gen}`

4. **涉及 MySQL 时执行 GORM Gen**
   - 编写 `ser-xxx/internal/gen/gorm_gen.go`（核心一行：调用 `mysql_gen.GenCodeWithAll(dbConnStr, "ser-xxx", "")`）
   - 通过 `common/script/gen_mysql.sh`（遍历所有 ser-* 下有 internal/gen 的项目执行）或直接 `go run internal/gen/gorm_gen.go`
   - 自动连接 TiDB 读取表结构 → 生成 model + query + repo 三层代码到 `internal/gorm_gen/`
   - 其中 `common/pkg/db/cmd/generate.go` 是核心生成逻辑，支持加密字段映射（EncryptedString）、表前缀去除、Repository 模板自动生成

5. **模块间调用 + 中间件引入**
   - 通过 `common/rpc` 的 Client 工厂调用其他服务
   - 网关层新增 handler + router 注册接入

6. **业务闭环验证**

### 十一、自我纠偏记录

| 原始认知 | 纠正后 | 依据 |
|---------|--------|------|
| 本地无源码的服务 = "未开发" | 其他组员在开发，本地无权限 | 用户反馈 |
| 开发流程从手动创建 go.mod 开始 | IDL 先行，脚本驱动生成 | gen_kitex.sh 实际逻辑 |
| 新模块需要手动注册到多处 common | 脚本自动生成 kitex_gen，handler/main 骨架也由脚本生成 | gen_kitex.sh 第38-55行 |
| 所有模块代码可作为规范参考 | 不同组员质量参差不齐，以 common + 网关为准 | 用户反馈 |
| 首次分析漏掉了 common/idl/ 和 common/script/ | 这两个目录是开发流程的核心入口 | 实际目录结构确认 |

---

## 深度分析补充（2026-02-25 二次深度分析）

以下内容基于全量代码逐文件分析，所有结论均附代码行级依据。

### 十二、完整请求处理链路（代码级证据）

#### 12.1 gate-back 启动序列

```
main.go:33 → lg.InitHLog()                          // 日志初始化
main.go:35 → etcd.LoadAndWatch("/slg/conf/")         // ETCD配置热加载(atomic.Value + watchLoop)
main.go:37 → middleware.InitRedis()                   // Redis客户端初始化(UniversalClient)
main.go:39 → s3.Init()                               // S3客户端初始化(AWS SDK v2)
main.go:41 → port 优先 flag > etcd                    // 端口来源
main.go:45 → hertz.New(WithHostPorts, WithMaxRequestBodySize(20*1024*1024))
main.go:50 → h.Use(middleware.IGlTracer.Start())      // UUID trace_id 注入
main.go:51 → h.Use(middleware.HertzAccLogStr())       // 访问日志
main.go:52 → router.GeneratedRegister(h)             // 路由注册
main.go:55 → h.Spin()                                // 启动HTTP监听
```

#### 12.2 gate-font 启动序列

与 gate-back 基本一致，差异：
- **无** `MaxRequestBodySize` 限制（gate-back 限 20MB）
- 端口和路由注册内容不同

#### 12.3 单次 HTTP 请求完整链路（以 gate-back `/admin/api/xxx` 为例）

```
[1] HTTP请求进入 Hertz
    ↓
[2] IGlTracer.Start()
    → tracer.go: 生成UUID trace_id → metainfo.WithPersistentValue(ctx, "X-Trace-Id", id)
    → 设置 CORS headers (Access-Control-Allow-Origin: *, Allow-Credentials: true)
    ↓
[3] HertzAccLogStr()
    → 记录请求开始时间、请求路径、Method
    → defer: 记录响应耗时、状态码
    ↓
[4] SetHeaderInfo()  [info.go:14-34]
    → 提取5个请求头/参数存入 metainfo:
      ├─ X-Client-IP    ← c.ClientIP()
      ├─ X-User-Language ← header "X-User-Language"
      ├─ X-User-Device-ID ← header "X-User-Device-ID"
      ├─ X-User-Device-Type ← header "X-User-Device-Type"
      └─ X-User-Token   ← header "Authorization" || cookie "session_token"
    ↓
[5] WhiteCheck()  [gate-auth.go:39-56]
    → 取 X-Client-IP → rpc.BComClient().OnWhiteList(ctx, ip)
    → RPC失败 → 拒绝请求（安全第一，宁可误杀）
    → 不在白名单 → 返回 IpNotInWhiteList 错误
    ↓
[6] BackAuthMiddleware()  [gate-auth.go:92-133]
    → 取 X-User-Token → Redis GET(token)  [注意: key=token本身，无前缀]
    → 解析 BUserTokenPayload → 设置:
      ├─ X-User-ID   ← payload.UserId
      └─ X-User-Name ← payload.UserName
    → URL权限校验: rpc.BUserClient().CheckUrlAuth(ctx, c.FullPath())
    → 权限不通过 → 返回 403
    ↓
[7] 路由匹配 → 具体 Handler 函数
    → 例: router_app.go 中 apiGroup.POST("/banner/pageBanner", app.PageBanner)
    ↓
[8] Handler 函数（gateway handler 文件）
    → 一行调用: rpc.Handler(ctx, c, rpc.AppClient().PageBanner)
    ↓
[9] rpc.Handler[REQ,RESP]()  [common/rpc/handler.go]
    → reflect.ValueOf(&reqData) 判断是否指针类型
    → c.BindAndValidate(bindTarget) 绑定+校验请求参数
    → rpcMethod(ctx, reqData) 发起 Kitex RPC 调用
    → c.JSON(ret.HttpOne(ctx, resp, I18nClient(), err))
    ↓
[10] RPC Client 调用  [common/rpc/rpc_client.go]
    → sync.Once 保证单例初始化
    → etcd Resolver 服务发现
    → TTHeader MetaHandler 透传 trace_id + user headers
    → MuxConnection(1) 多路复用
    → 超时: 1200秒
    ↓
[11] Kitex RPC → 目标微服务 handler.go
    → 进入 Service 层 → Repository 层 → GORM → TiDB
    → 返回结果沿链路反向传播
    ↓
[12] ret.HttpOne(ctx, resp, i18nClient, err)  [project_ret.go]
    → err == nil: 返回 {code:0, msg:"ok", data:resp, traceId:xxx}
    → err != nil:
      ├─ BizStatusError: 取 code → i18n翻译 → {code:xxx, msg:"翻译后消息"}
      └─ 其他错误: {code:1, msg:"系统异常"}
```

#### 12.4 gate-font `/api/xxx` 链路差异

```
[5'] BlackCheck()  [gate-auth.go:21-37]
    → rpc.BComClient().OnBlackList(ctx, ip)
    → RPC失败 → 放行（体验优先，宁可漏放）
    → 在黑名单 → 返回 IpIsInBlackList 错误

[6'] FontAuthMiddleware()  [gate-auth.go:58-90]
    → Redis GET("user:token:" + token)  [注意: key有前缀 "user:token:"]
    → 解析 CUserTokenPayload → 设置:
      ├─ X-User-ID   ← payload.UserId
      └─ X-User-Name ← payload.NickName  [注意: 是NickName不是UserName]
    → **无** URL权限校验（C端不做RBAC）
```

### 十三、ETCD 配置体系（路径 + 用途 + 代码依据）

#### 13.1 加载机制

```go
// common/pkg/etcd/etcd_cache.go
func LoadAndWatch(prefix string) {
    // 1. 首次加载: etcd Get(prefix, WithPrefix) → 全量写入 atomic.Value (map[string]string)
    // 2. 启动 watchLoop goroutine: Watch(prefix, WithPrefix) → 增量更新 atomic.Value
    // 3. Hook机制: 配置变更后触发注册的回调函数
}
func Get(key string) string    // 从 atomic.Value 读取（无锁，Copy-On-Write）
func GetDef(key, def string)   // 带默认值
func Hook(fn func())          // 注册配置变更回调
```

#### 13.2 核心配置路径

| 路径 | 用途 | 消费方 |
|------|------|--------|
| `/slg/conf/tidb/addr` | TiDB地址 | 所有ser-*服务 |
| `/slg/conf/tidb/port` | TiDB端口 | 所有ser-*服务 |
| `/slg/conf/tidb/username` | TiDB用户名 | 所有ser-*服务 |
| `/slg/conf/tidb/password` | TiDB密码 | 所有ser-*服务 |
| `/slg/conf/redis/addr` | Redis地址 | 网关+所有服务 |
| `/slg/conf/redis/password` | Redis密码 | 网关+所有服务 |
| `/slg/conf/s3/ak` | S3 AccessKey | ser-s3, 网关 |
| `/slg/conf/s3/sk` | S3 SecretKey | ser-s3, 网关 |
| `/slg/conf/s3/endpoint` | S3端点 | ser-s3, 网关 |
| `/slg/conf/s3/bucket` | S3桶名 | ser-s3, 网关 |
| `/slg/conf/s3/domain` | S3域名(CDN) | ser-s3, 网关 |
| `/slg/conf/s3/region` | S3区域 | ser-s3, 网关 |
| `/slg/conf/security/sensitive_data_key` | AES加密密钥 | common/pkg EncryptedString |
| `/slg/conf/security/sensitive_data_key_iv` | AES加密IV | common/pkg EncryptedString |
| `/slg/serv/{service-name}/port` | 各服务端口 | 各自main.go |

#### 13.3 业务级 ETCD 配置

| 路径 | 格式 | 用途 | 消费方 |
|------|------|------|--------|
| `/slg/conf/ip/black` | `{"status":0/1,"isoCodeList":["CN",...]}` | C端黑名单开关+国家列表 | ser-ip |
| `/slg/conf/ip/white` | `{"status":0/1}` | B端白名单开关 | ser-ip |

### 十四、Redis Key 设计规范与用途

| Key模式 | 数据类型 | 用途 | 服务 |
|---------|---------|------|------|
| `{token值}` | String(JSON) | B端用户Token → BUserTokenPayload | gate-back |
| `user:token:{token值}` | String(JSON) | C端用户Token → CUserTokenPayload | gate-font |
| `{banner:list}:{pagePosition}:{terminal}` | String(JSON) | Banner列表缓存(含防穿透空数组) | ser-app |
| `{item:list}:{分类key}` | String(JSON) | 道具列表缓存 | ser-item |
| `i18n:hs:zhcn` | Hash | 中文简体翻译键值对 | ser-i18n |
| `i18n:hs:zhtw` | Hash | 中文繁体翻译键值对 | ser-i18n |
| `i18n:hs:en` | Hash | 英文翻译键值对 | ser-i18n |
| `i18n:hs:vivn` | Hash | 越南语翻译键值对 | ser-i18n |
| `i18n:hs:idid` | Hash | 印尼语翻译键值对 | ser-i18n |
| `i18n:hs:thth` | Hash | 泰语翻译键值对 | ser-i18n |
| `s3:config` | Hash | S3上传配置(field=BsType) | ser-s3 |
| `enum:{enumType}` | String | 枚举缓存 | ser-bcom |
| `google2fa:{userId}` | String | Google验证器临时secret | ser-buser |

### 十五、统一返回体系（ret包详解）

#### 15.1 标准响应结构

```go
// common/pkg/ret/project_ret.go
type R struct {
    TraceId string      `json:"traceId"`
    Code    int         `json:"code"`
    Msg     string      `json:"msg"`
    Data    interface{} `json:"data"`
}
```

#### 15.2 核心函数链

```
ret.HttpOne(ctx, data, i18nClient, err)
  ├─ err == nil → (200, R{Code:0, Msg:"ok", Data:data})
  ├─ BizStatusError → (200, R{Code:bizCode, Msg:i18n翻译(bizCode)})
  └─ 其他error → (200, R{Code:1, Msg:"系统异常"})

ret.HttpOk(ctx, data) → (200, R{Code:0, Data:data})

ret.HttpErr(ctx, errCode, i18nClient)
  → trans.LangTransOne(ctx, i18nClient, errCode) 翻译错误码
  → (200, R{Code:errCode, Msg:翻译结果})

ret.BizErr(ctx, errCode)
  → 从 Redis Hash "i18n:hs:{lang}" 取翻译
  → kitex bizStatusError(code, msg) —— 用于 Service 层抛业务错误

ret.RpcOne[T](resp, err)  // 泛型，Service间RPC调用后解包
```

#### 15.3 错误码体系

```go
// common/pkg/errs/code.go
const SystemBaseError = 6000000
const (
    OK          = 0                    // 成功
    NG          = 1                    // 通用失败
    OperateErr  = SystemBaseError + iota  // 6000002 操作错误
    ParamErr                              // 6000003 参数错误
    IpIsNil                               // 6000004 IP为空
    TokenNotExist                         // 6000005 Token不存在
    TokenExpired                          // 6000006 Token过期
    NoAuth                                // 6000007 无权限
    IpNotInWhiteList                      // 6000008 IP不在白名单
    IpIsInBlackList                       // 6000009 IP在黑名单
)
```

各服务自定义错误码范围（依据各 `internal/errs/` 文件）：
- ser-app: 6001xxx
- ser-buser: 6002xxx
- ser-i18n: 6003xxx
- ser-item: 6004xxx
- ser-s3: 6005xxx

### 十六、GORM 基础设施深度解析

#### 16.1 MySQL初始化 [common/pkg/db/mysql/mysql.go:28-72]

```go
func Init(opt *InitOption) {
    db, _ := gorm.Open(mysql.Open(opt.dsn()), &gorm.Config{
        PrepareStmt:            true,   // PreparedStatement 缓存
        Logger:                 NewGormLogger(),
        SkipDefaultTransaction: true,   // 单条操作跳过事务包装(性能)
    })
    db.Use(&UserPlugin{})               // 注册审计插件
    sqlDB, _ := db.DB()
    sqlDB.SetMaxIdleConns(opt.MaxIdleConns)
    sqlDB.SetMaxOpenConns(opt.MaxOpenConns)
    sqlDB.SetConnMaxLifetime(opt.ConnMaxLifetime)
}
```

#### 16.2 UserPlugin审计插件 [user_plugin.go:21-28]

```
触发时机: GORM Before Create/Update 回调
工作机制:
  → 从 metainfo 读取 X-User-ID → 通过反射设置:
    ├─ CreateBy / UpdateBy (i64, 设为 userId)
    ├─ CreateAt / UpdateAt (i64, 设为 time.Now().UnixMilli())
    └─ 支持: 直接struct / *struct / 批量[]struct / 批量[]*struct
```

#### 16.3 EncryptedString [crypto_util.go:23-56]

```
写入DB: Value() → AES-256-CBC加密 → Base64编码 → 存密文
读取DB: Scan() → Base64解码 → AES-256-CBC解密 → 返回明文
密钥来源: etcd /slg/conf/security/sensitive_data_key + sensitive_data_key_iv
使用方式: Model字段类型声明为 EncryptedString，GORM自动透明加解密
```

#### 16.4 GORM Gen 代码生成流程 [common/pkg/db/cmd/generate.go:57-160]

```
输入: DSN连接串 + 服务名 + 表前缀
流程:
  1. gorm.Open 连接 TiDB
  2. gen.NewGenerator({OutPath, FieldNullable:true, FieldCoverable:true})
  3. 遍历 information_schema.tables → 过滤排除表
  4. 对每张表:
     a. GenerateModel(tableName, fieldOpts...) → model struct (internal/gorm_gen/model/)
     b. EncryptedString字段自动映射: 字段注释含"encrypted" → 类型改为 EncryptedString
     c. 去除表前缀(如 "t_" → 模型名不带前缀)
  5. g.Execute() → 生成 query 层 (internal/gorm_gen/query/)
  6. 生成 repo 层 (internal/gorm_gen/repo/) → 基于 repo_template.go 模板

repo_template.go 模板提供的18个方法:
  Create, CreateBatch, Update, UpdateFields, DeleteById, DeleteByIds,
  GetById, GetByIds, GetOne, GetList, GetAll, Count,
  GetPage(分页), Exists, GetByField, UpdateById, FirstOrCreate, Transaction
```

### 十七、核心业务模块深度解析

#### 17.1 Banner模块 [ser-app/internal/ser/banner.go, 767行]

**创建流程 CreateBanner:**
```
1. images 去重(按lang) + 校验defaultLang必须在images中
2. 校验bannerName在同一pagePosition下唯一
3. URL处理: 图片URL去掉域名前缀→存相对路径
4. 排序冲突处理: ShiftAndCreate
   → 查询 sort_value >= 目标值 的记录 → 全部+1 → 插入新记录(在同一事务内)
5. 缓存: 删除该 pagePosition 下所有终端的缓存
6. 审计日志: RPC调用 ser-blog 记录操作
```

**C端查询 ListBanner:**
```
1. Redis缓存: key = {banner:list}:{pagePosition}:{terminal}
   → 命中(非nil) → 返回(含空数组=防穿透)
   → 未命中(nil) → 查DB
2. DB查询: status=1, pagePosition匹配, terminal包含
3. 写缓存: 无数据写空数组(防缓存穿透), 有数据写完整列表
4. 过滤: onlineTime <= now (已到上线时间的才展示)
5. 多语言: 根据客户端lang匹配图片 → 无匹配则用 defaultLang 兜底
```

#### 17.2 RBAC模块 [ser-buser]

**数据模型:**
```
sys_user ──1:N──→ sys_user_role ──N:1──→ sys_role
sys_role ──N:M──→ sys_role_menu ──N:M──→ sys_menu

sys_menu 字段:
  - menu_type: 1目录 2菜单 3按钮
  - super_admin_flag: "1"=仅超级管理员可见(不可被角色分配修改)
  - perms: 权限标识(用于URL权限校验)
```

**URL权限校验链路:**
```
BackAuthMiddleware → rpc.BUserClient().CheckUrlAuth(ctx, fullPath)
→ ser-buser: 根据userId查角色 → 查角色关联菜单 → 匹配菜单perms与请求URL
→ 匹配成功=放行, 不匹配=403
```

**用户CRUD（事务保护）:**
```
UserCreate: TX{ INSERT sys_user + INSERT sys_user_role }
  密码: bcrypt(password + salt), salt随机生成存入sys_user
UserUpdate: TX{ UPDATE sys_user + DELETE旧sys_user_role + INSERT新sys_user_role }
```

#### 17.3 S3文件服务 [ser-s3/internal/ser/com_service.go]

**配置驱动上传:**
```
s3_config 表字段:
  - BsType: 业务类型标识(如 "avatar", "banner", "item")
  - PathTemplate: 路径模板，支持变量替换:
      {owner_id} → 上传者ID
      {year}/{month}/{day} → 日期
  - AllowedExt: 允许的文件扩展名(逗号分隔)
  - MaxFileSize: 最大文件大小(字节)
  - IsPublic: 是否公开(决定存储路径前缀 public/ 或 private/)

配置缓存: DB → Redis Hash "s3:config" (field=BsType, value=JSON)
```

**上传流程:**
```
1. GetPresignUrl: BsType→查配置 → PathTemplate变量替换 → UUID文件名 → 预签名URL(PUT)
2. 客户端直传S3
3. UploadComplete:
   → Range读取(bytes=0-511) → magic byte检测文件真实类型
   → 与声明的MIME/扩展名对比 → 不匹配则拒绝
   → 通过则记录 s3_documents 表
```

#### 17.4 i18n多语言 [ser-i18n/internal/ser/i18n_service.go]

**存储结构:**
```
DB: langs 表，每行含 6 个语言字段:
  lang_zh_cn, lang_zh_tw, lang_en, lang_vi_vn, lang_id_id, lang_th_th
  + lang_key(翻译键) + module(模块分类)

Redis: 6 个 Hash:
  i18n:hs:zhcn → {lang_key: 翻译值, ...}
  i18n:hs:zhtw → {lang_key: 翻译值, ...}
  i18n:hs:en   → {lang_key: 翻译值, ...}
  i18n:hs:vivn → {lang_key: 翻译值, ...}
  i18n:hs:idid → {lang_key: 翻译值, ...}
  i18n:hs:thth → {lang_key: 翻译值, ...}
```

**批量同步:** Pipeline，每批100条×6语言=600个HSet操作，减少Redis RTT

#### 17.5 IP访问控制 [ser-ip]

**黑名单（C端）:**
```
OnBlackList(ip):
  1. etcd读取开关: /slg/conf/ip/black → JSON{status, isoCodeList}
  2. status=0 → 功能关闭，放行
  3. IP解析: 调用第三方 IpAnalysis(ip) → 获取国家代码(ISO 3166-1)
  4. 国家代码 ∈ isoCodeList → 在黑名单中
```

**白名单（B端）:**
```
OnWhiteList(ip):
  1. etcd读取开关: /slg/conf/ip/white → JSON{status}
  2. status=0 → 功能关闭，放行
  3. DB查询 sys_white_list: ip精确匹配 → 在白名单中
```

#### 17.6 ser-item DDD模式 [ser-item/internal/domain/]

```
ItemValidator(接口):
  → DefaultItemValidator 实现:
    - ValidateName(3-36字符)
    - ValidateURLs(非空 + 转相对路径)
    - ValidatePrice(正数)

ItemMapper(结构体):
  → ToModel() / ToPublishModel() / ToListResp() 数据转换

CacheManager(接口):
  → RedisCacheManager 实现: 道具列表缓存读写

LogBuilder(接口):
  → AuditLogBuilder 实现: 构建审计日志RPC请求

4层调用: Handler → Service(编排) → Domain(校验/转换) → Repository(持久化)
```

### 十八、RPC 通信细节

#### 18.1 Server端初始化 [common/rpc/init.go]

```go
func InitRpcServerParams() []server.Option {
    return []server.Option{
        server.WithReusePort(true),       // SO_REUSEPORT
        server.WithMuxTransport(),         // 多路复用
        server.WithRegistry(etcdRegistry), // etcd注册
        server.WithMetaHandler(TTHeader),  // 元信息透传
        server.WithMiddleware(logMW),      // 日志中间件(trace_id+method+cost+error)
    }
}
```

#### 18.2 Client端初始化 [common/rpc/init.go]

```go
func InitRpcClientParams() []client.Option {
    return []client.Option{
        client.WithResolver(etcdResolver),      // etcd服务发现
        client.WithRPCTimeout(1200 * time.Second), // 超时1200秒(实际值，注释写3秒)
        client.WithMuxConnection(1),              // 单连接多路复用
        client.WithMetaHandler(TTHeader),         // 透传metainfo
    }
}
```

#### 18.3 Client单例模式 [common/rpc/rpc_client.go]

```go
var appClientOnce sync.Once
var appClient appservice.Client

func AppClient() appservice.Client {
    appClientOnce.Do(func() {
        var err error
        appClient, err = appservice.NewClient(
            namesp.EtcdAppService,      // etcd服务名
            InitRpcClientParams()...,
        )
        if err != nil {
            hlog.Error("init app client error", err)
            // 注意: 只打日志不panic，后续调用会NPE
        }
    })
    return appClient
}
// 共19个这样的Client工厂函数
```

#### 18.4 泛型Handler [common/rpc/handler.go]

```go
// 标准模式: 绑定请求 → RPC调用 → 统一响应
func Handler[REQ any, RESP any](ctx context.Context, c *app.RequestContext,
    rpcMethod func(context.Context, REQ, ...callopt.Option) (RESP, error)) {

    var reqData REQ
    val := reflect.ValueOf(&reqData).Elem()
    if val.Kind() == reflect.Ptr {
        // IDL生成的是指针类型，需要初始化
        val.Set(reflect.New(val.Type().Elem()))
        bindTarget = reqData
    } else {
        bindTarget = &reqData
    }

    err := c.BindAndValidate(bindTarget)  // Hertz 参数绑定+校验
    if err != nil { /* 返回参数错误 */ }

    resp, err := rpcMethod(ctx, reqData)   // Kitex RPC调用
    c.JSON(ret.HttpOne(ctx, resp, I18nClient(), err))  // 统一返回
}

// 无参模式: 无需请求体
func HandlerNoParam[RESP any](ctx, c, rpcMethod)

// CSV导出模式: 检查resp是否实现CSVResponse接口
func HandlerWithCSV[REQ, RESP any](ctx, c, rpcMethod)
  → resp实现GetFilename()+GetData() → Content-Disposition下载
  → 否则 → 正常JSON返回
```

### 十九、跨服务调用关系图

```
                    ┌──────────────────────────────────────────────────────┐
                    │                    调用方向: →                        │
                    └──────────────────────────────────────────────────────┘

gate-back ──→ ser-bcom.OnWhiteList()     (IP白名单校验)
gate-back ──→ ser-buser.CheckUrlAuth()   (URL权限校验)
gate-font ──→ ser-bcom.OnBlackList()     (IP黑名单校验)

ser-app   ──→ ser-blog  (Banner/分类/策略 操作审计日志)
ser-app   ──→ ser-s3    (图片URL处理)

ser-buser ──→ ser-blog  (用户/角色/菜单 操作审计日志)
ser-buser ──→ ser-ip    (白名单管理)

ser-bcom  ──→ ser-ip    (黑白名单转发)
ser-bcom  ──→ ser-i18n  (枚举翻译)

ser-i18n  ──→ ser-blog  (翻译条目 操作审计日志)

ser-item  ──→ ser-blog  (道具 操作审计日志)
ser-item  ──→ ser-s3    (道具图片)
ser-item  ──→ ser-buser (查询操作人名称)

ser-s3    ──→ ser-blog  (文件操作审计日志)

ser-ip    ──→ 第三方IP解析服务 (IpAnalysis)

所有服务的ret.BizErr() ──→ Redis "i18n:hs:{lang}" (错误码翻译, 非RPC调用, 直接读Redis)
```

### 二十、Kitex Gen 代码组织 [common/pkg/kitex_gen/]

```
common/pkg/kitex_gen/
├── ser_app/                    # ser-app IDL生成的代码
│   ├── banner.go              # Banner相关struct
│   ├── home_category.go       # 首页分类struct
│   ├── content_strategy.go    # 内容策略struct
│   └── appservice/            # AppService client/server接口
├── ser_bcom/                   # ser-bcom IDL生成
├── ser_buser/                  # ser-buser IDL生成
│   └── buserservice/          # BUserService client/server
├── ser_i18n/                   # ser-i18n IDL生成
├── ser_ip/                     # ser-ip IDL生成
├── ser_item/                   # ser-item IDL生成
├── ser_s3/                     # ser-s3 IDL生成
├── ser_blog/                   # ser-blog IDL生成(无本地服务源码)
├── ser_user/                   # ser-user IDL生成(无本地服务源码)
├── ser_live/                   # ser-live IDL生成(无本地服务源码)
├── ... (其余12个服务)
└── common/                    # 公共enum定义
```

### 二十一、微服务标准内部结构

```
ser-xxx/
├── main.go                     # 入口: 初始化基础设施 → Kitex server → Run()
├── handler.go                  # RPC Handler: IDL方法 → Service调用
├── internal/
│   ├── cfg/                    # 服务级配置(ETCD路径常量, DB初始化)
│   ├── enum/                   # 业务枚举常量
│   ├── errs/                   # 服务级错误码(6001xxx, 6002xxx...)
│   ├── ser/                    # Service层: 业务逻辑编排
│   ├── rep/                    # Repository层: 数据库操作(使用gorm_gen)
│   ├── cache/                  # 缓存层: Redis操作
│   ├── domain/                 # 领域层(可选, ser-item使用)
│   ├── gen/                    # GORM Gen入口(gorm_gen.go)
│   └── gorm_gen/               # 自动生成目录
│       ├── model/              # 表模型 struct
│       ├── query/              # 类型安全查询构建器
│       └── repo/               # 仓库方法(18个标准CRUD)
├── go.mod                      # 模块定义, require common/pkg
└── go.sum
```

### 二十二、关键代码位置索引

| 功能 | 文件路径 | 关键行 |
|------|---------|--------|
| ETCD热加载 | common/pkg/etcd/etcd_cache.go | LoadAndWatch, watchLoop, Hook |
| MySQL初始化 | common/pkg/db/mysql/mysql.go | :28-72 |
| 审计插件 | common/pkg/db/mysql/user_plugin.go | :21-28 |
| 字段加密 | common/pkg/utils/crypto_util.go | :23-56 |
| Redis初始化 | common/pkg/db/redis/redis_init.go | :30-60 |
| S3客户端 | common/pkg/s3/mys3.go | :32-56 |
| Kafka生产者 | common/pkg/mq/kafka/producer.go | :40-82 |
| Kafka消费者 | common/pkg/mq/kafka/consumer.go | :44-109 |
| NATS客户端 | common/pkg/mq/nats/nats.go | :29-76 |
| ES客户端 | common/pkg/es/client.go | :42-80 |
| 链路追踪 | common/pkg/tracer/tracer.go | Start(), 8个header常量 |
| 统一返回 | common/pkg/ret/project_ret.go | HttpOne, HttpOk, HttpErr, BizErr |
| 错误码 | common/pkg/errs/code.go | :19-42 |
| 雪花ID | common/pkg/utils/snowflake.go | 41+5+5+12位 |
| IP黑名单 | common/pkg/middleware/gate-auth.go | :21-37 BlackCheck |
| IP白名单 | common/pkg/middleware/gate-auth.go | :39-56 WhiteCheck |
| C端鉴权 | common/pkg/middleware/gate-auth.go | :58-90 FontAuth |
| B端鉴权 | common/pkg/middleware/gate-auth.go | :92-133 BackAuth |
| 请求头提取 | common/pkg/middleware/info.go | :14-34 |
| RPC Client工厂 | common/rpc/rpc_client.go | 19个sync.Once单例 |
| 泛型Handler | common/rpc/handler.go | Handler, HandlerNoParam, HandlerWithCSV |
| RPC初始化参数 | common/rpc/init.go | Server/Client Option配置 |
| GORM Gen核心 | common/pkg/db/cmd/generate.go | :57-160 |
| Repo模板 | common/pkg/db/cmd/repo_template.go | 18个CRUD方法 |
| Kitex生成脚本 | common/script/gen_kitex.sh | 全文69行 |
| GORM Gen脚本 | common/script/gen_mysql.sh | 全文18行 |
| gate-back启动 | gate-back/main.go | :33-76 |
| gate-back路由 | gate-back/biz/router/my_router/my_router.go | :16-24 |
| gate-back注册 | gate-back/biz/router/register.go | 14个模块 |
| gate-font启动 | gate-font/main.go | 同构gate-back |
| gate-font路由 | gate-font/biz/router/my_router/my_router.go | :16-25 |
| gate-font注册 | gate-font/biz/router/register.go | 8个模块 |
| Banner服务 | ser-app/internal/ser/banner.go | 767行完整CRUD |
| Banner缓存 | ser-app/internal/cache/banner.go | 防穿透策略 |
| RBAC用户 | ser-buser/internal/rep/user.go | 事务+bcrypt |
| i18n服务 | ser-i18n/internal/ser/i18n_service.go | 6Hash+Pipeline |
| S3服务 | ser-s3/internal/ser/com_service.go | 配置驱动+magic byte |
| IP服务 | ser-ip/internal/ser/c_black_service.go | etcd开关+IP解析 |
| Item领域 | ser-item/internal/domain/item_validator.go | DDD校验器 |
| IDL规范 | common/idl/readme.md | 注解说明 |
| IDL Banner | common/idl/ser-app/banner.thrift | B/C端结构 |
