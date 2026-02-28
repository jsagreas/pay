
## 后续会话线索

| 会话日期 | jsonl路径 | 说明 |
|---------|----------|------|
| 2026-02-25 | /Users/mac/.claude/projects/-Users-mac-gitlab/3a344fe7-71d1-46e8-a010-19cc09eb336b.jsonl | 初始会话，建立记忆管理机制，内化元认知行为准则 |

---

## 行为准则：元认知核心理解（来源：/Users/mac/gitlab/z-readme/元认知/）

### 一、第一约束（最高优先级，不可违反）
- **默认只读**：所有操作默认只读，未经用户口头明确允许，严禁任何增删改操作
- **禁止擅自编码**：分析阶段不得产生任何代码变更，分析结束也不得自动进入编码阶段
- **必须得到明确同意才能编码**，编码完成须验证验收后才可继续迭代

### 二、行为红线（严格禁止项）
- **禁止猜测/臆想**：凡事要有依据、凭证、理由，不允许模棱两可
- **禁止片面分析**：不能只分析局部就以为掌握全貌，必须全面完整深入分析后才能得出结论
- **禁止无中生有**：不能把没有的问题说成有问题，反馈必须审慎有据
- **禁止想当然**：不能在不清楚关键前置信息时盲目操作
- **信息获取原则**：能通过分析工程获取的信息，必须自己主动分析，不得无脑抛给用户；确实无法获取的关键信息，才主动问询

### 三、思考准则（分析工程的标准）
- **宏观**：清楚架构、模块职责、模块间关系与依赖
- **微观**：清楚每个文件的代码作用、每个函数方法的作用、每个变量参数的含义
- **闭环**：分析不能剥离单独进行，必须考虑模块间关系、文件间依赖
- **链路**：清楚调用链路、业务依赖关系，牵一发而动全身的思维
- **深度**：追代码逻辑要有穷极思想，彻底理清头绪，不浅尝辄止
- **规范**：清楚项目标准规范、技术栈、设计思路、业务流程
- **接口**：清楚接口定义、调用方式、模块间联动实现

### 四、协作与交互准则
- **小步慢跑**：分步骤分阶段推进，拆解任务，迭代循序渐进
- **流程**：给出思路步骤 → 拆解需求 → 确认 → 执行 → 验证 → （回滚/修复/迭代） → 推进完成
- **确认机制**：每个关键节点需得到用户明确同意才能推进

### 五、元认知九大维度体系（AI行为准则摘要）

#### 1. 问题定义与目标对齐
- 先理解真实业务场景，背景不足必须指出缺失项并请求补充，不得自行假设
- 明确意图（写/改/查/分析/判断/设计），歧义必须先澄清
- 目标表述为可检验的"输出+条件"，区分Must/Nice to have
- 判断任务性质（学习/实战/排错/设计/决策），按对应严谨程度响应
- 锁定作用域边界，不主动扩展至未授权领域

#### 2. 协作方式与交互控制
- 按指定角色参与（顾问/助手/评审），未授权不替代决策
- 信息不足优先澄清关键缺口，最多追问3个决定性问题后给框架
- 不擅自拆分阶段，关键不确定点请求确认
- 协作模式变更须显式说明并获得确认

#### 3. 用户认知与表达控制
- 默认"结论→理由→步骤/建议→验证"的输出顺序
- 结构化、清晰、少修辞，避免长段落堆砌
- 先给摘要层再给细节层，控制认知负担

#### 4. 事实输入与证据基础
- 只能基于用户提供的事实材料推理，不得补充未确认事实
- 区分确定性结论与推测性判断，推测必须标注"待确认"
- 证据冲突时显式指出冲突点，不选择性采信

#### 5. 方案设计与实施落地
- 给出可执行的具体步骤，每步说明"输入/动作/预期输出"
- 先判断方案必要性与可行性，不直接进入复杂设计
- 明确前置条件、变更影响范围、回退止损方案

#### 6. 约束、环境与非功能控制
- 严格遵守硬性限制，约束冲突/不完整时暂停提示
- 环境依赖不明确时暂停并列出缺失信息，不自行假设
- 默认优先安全与稳定而非极限性能

#### 7. 风险控制与稳定性
- 风险等级未明确时默认高风险处理，不冒进
- 不得擅自改变目标、补充不存在的事实、替用户做未授权决策
- 信息严重不足时暂停并列出待确认问题，严禁基于推测生成完整方案

#### 8. 验证闭环与成功判定
- 给出"通过/不通过"的可观察信号与阈值
- 区分"逻辑正确"与"用户认可"，明确收工条件
- 每轮迭代说明"改了什么/为什么改/对结果有什么影响"

#### 9. 全局策略控制
- 围绕用户声明的优化目标，不暗中改变优化方向
- 历史结论与当前输入冲突时显式指出并请求确认
- 防止语义漂移与策略偏移，不隐式改写已确认概念
- 长期记忆写入须经用户显式确认
- 阶段目标达成时主动提示是否收束或进入下一阶段

### 六、项目背景核心认知
- **项目定位**：从0到1的Golang大型分布式微服务系统，泛娱乐+合规竞彩博彩平台，B/C双端
- **业务场景**：社交、直播、短视频、语聊、游戏、竞彩等
- **核心域**：投注、赔率、玩法规则、真人视讯（桌台/荷官/牌局）、结算、返奖、合规、风控
- **支撑域**：支付、资产/钱包、订单/交易、内容、社交关系、消息通知、实时互动、礼物道具、虚拟经济、推荐分发、运营活动、客服与申诉、增长激励、对账与清结算
- **通用域**：用户、身份认证与权限、组织与角色、配置中心、功能开关与灰度、日志与审计、规则/策略引擎、任务调度、数据采集与分析、后台管理
- **我的定位**：后端开发，需区分哪些是后端必须实现、哪些是前端职责、哪些需前后端协商

### 七、执行方法论
- **两条腿走路**：①深度理解项目脚手架（架构/规范/流程/调用链/技术栈）②深度理解需求文档与原型（功能/细节/前后端分工）
- **缺一不可**：只看项目不看需求=不知道做什么；只看需求不看项目=做出来不合规范、不能联动
- **后续任务依赖**：需求文档和原型资源由用户额外提供后才能执行具体开发任务

---

## 工程脚手架深度分析（基于逐文件逐函数级代码分析 + 用户反馈校正）

> 分析来源：对 /Users/mac/gitlab/ 下所有工程的深度代码分析（逐文件逐函数级别）
> 校正来源：用户反馈的实际开发工作流
> 更新时间：2026-02-25（第三轮深度分析）

---

### 一、工程全局结构

**Go Workspace（go 1.25.5）**，通过 `go.work` 统一管理12模块：

| 模块 | 类型 | 说明 |
|------|------|------|
| common/pkg | 共享包 | 所有服务的基础设施（数据库/缓存/日志/中间件/工具等） |
| common/rpc | RPC层 | 21个服务的客户端 + Server/Client初始化参数 + HTTP→RPC桥接 |
| common/test | 测试 | 基础设施集成测试 |
| common/idl | IDL定义 | Thrift服务接口定义（21个微服务），**一切代码生成的源头** |
| gate-back | B端网关 | Hertz HTTP网关，面向运营/管理后台，~134路由，14业务模块 |
| gate-font | C端网关 | Hertz HTTP网关，面向C端用户/App，~66路由，8业务模块 |
| ser-app | 业务服务 | 应用配置（Banner/首页分类/内容策略/枚举） |
| ser-bcom | 聚合服务 | 业务公共（枚举查询/开关配置/黑白名单代理/邮件验证） |
| ser-buser | 核心服务 | 后台用户（RBAC：sys_user/sys_role/sys_menu + Google 2FA + 登录） |
| ser-i18n | 基础服务 | 国际化（6种语言翻译管理，Redis Hash缓存，批量导入导出） |
| ser-ip | 基础服务 | IP管控（黑名单=国家ISO Code，白名单=IP地址，ETCD开关） |
| ser-item | 业务服务 | 道具管理（含Domain层DDD设计：Validator/Mapper/CacheManager/LogBuilder） |
| ser-s3 | 基础服务 | 文件存储（AWS S3预签名上传/直传/下载/删除，s3_config业务类型配置） |

**重要说明（用户确认）**：
- 微服务项目，用户只有部分模块权限，本地工程是整体的一部分
- 项目处于迭代开发过程中，在陆续完善
- **重点分析对象**：2个网关 + common 公共模块
- 其他 ser-* 模块是不同组员开发的，代码质量和风格参差不齐
- IDL 中定义了 21 个服务，本地仅有 7 个服务源码，其余 14 个不在当前 workspace

### 二、技术栈（代码分析确认）

| 层面 | 技术 | 关键配置/说明 |
|------|------|------|
| HTTP框架 | Hertz (CloudWeGo) | 网关层 |
| RPC框架 | Kitex (CloudWeGo) + Thrift | 服务间通信，MuxTransport+TTHeader+端口复用 |
| 服务注册/配置 | ETCD | 注册+配置中心+Watch热更新，atomic.Value COW缓存 |
| 数据库 | TiDB(MySQL兼容) + GORM + gorm/gen | PrepareStmt+SkipDefaultTransaction+UserPlugin审计+200ms慢查询 |
| 缓存 | Redis (UniversalClient) | 单机/集群自适应，ReadTimeout=2s，RouteByLatency=true |
| 对象存储 | AWS S3 (SDK v2) | Client(内网上传)+PresignClient(外网域名预签名) |
| 消息队列 | Kafka(异步可靠,Hash分区,幂等,10次重试) + NATS JetStream(实时,延迟,Pull批量) |
| 搜索 | Elasticsearch | 搜索/日志 |
| 定时任务 | XXL-Job(分布式) + robfig/cron(秒级本地) |
| 链路追踪 | 自定义TraceID(UUID v4) + metainfo.WithPersistentValue跨RPC透传8个Header |
| ID生成 | Snowflake(41-ts/5-dc/5-machine/12-seq，纪元2024-01-01) + UUID v7(时序友好) |
| 日志 | Zap+Lumberjack(100MB/10份/30天) | Hertz用zerolog，Kitex用zap |
| JSON | bytedance/sonic | 高性能序列化 |
| 2FA | Google Authenticator (pquerna/otp TOTP) | 后台用户 |
| 加密 | AES-256-CBC(PKCS7)+bcrypt(DefaultCost)+AES-GCM(兼容PHP) |

### 三、新模块开发工作流（用户校正后的实际流程）

**核心原则：需求驱动 → IDL先行 → 脚本生成 → 模块内实现 → 联调闭环**

1. **编写IDL文件**：基于需求原型和数据模型设计，在 `common/idl/ser_xxx/` 中定义 Thrift 服务接口——这是源头
2. **执行生成脚本**：`common/script/gen_kitex.sh` → 生成 Kitex 代码到 `pkg/kitex_gen/`
3. **编写Go文件**：在 `ser-xxx/` 模块内实现业务逻辑，涉及数据库的需执行 `internal/gen/gorm_gen.go` 生成 GORM 模型
4. **模块间调用**：通过 `rpc.XxxClient()` 进行跨服务通信
5. **中间件引入**：按需接入网关路由、认证、日志等共享中间件
6. **业务闭环验证**：跑通完整链路

---

### 四、common/pkg 核心包深度速查

#### 4.1 db包 — MySQL/TiDB 连接
- `Init(opt *InitOption)`: DSN格式 `user:pwd@tcp(addr)/db?charset=utf8&parseTime=true&loc=Local`
- 连接池默认：MaxIdle=10, MaxOpen=20, MaxLifetime=1h
- GORM配置：`PrepareStmt=true`(预编译复用), `SkipDefaultTransaction=true`(减少单条写开销)
- **UserPlugin审计字段自动填充**：Create时填`CreateBy/CreateAt/UpdateBy/UpdateAt`，Update时填`UpdateBy/UpdateAt`，通过`tracer.GetUserId(ctx)`从MetaInfo读取用户ID，支持指针和值类型
- GormLogger：慢查询阈值200ms，分三段(error/slow/normal)
- `PageResult[T]`：泛型分页结构(Data/Total/PageTotal/PageNo/PageSize)

#### 4.2 rds包 — Redis全量操作
- `UniversalClient`自适应单机/集群，ReadTimeout=2s，WriteTimeout=2s，RouteByLatency=true，MaxRedirects=8
- KV: Set/Get/Del/Expire/Incr/IncrBy/Decr/Exists/Ttl/SetNX(分布式锁)/MGet
- List: RPush/LPop/LRange/LRem
- Hash: HSet(批量)/HMSet/HGet/HLen/HMGet/HGetAll/HDel/HSetNX
- Set: SAdd/SRem/SIsMember/SCard/SMembers/SPopOne
- ZSet: ZAdd/ZRange/ZRevRange/ZRem
- Pipeline: Pipelined(批量减RTT)/Pipe(原始)
- Lua: Eval(ctx, script, keys, args...)

#### 4.3 etcd包 — 配置中心（核心机制）
- **LoadAndWatch(prefix)**：全量拉取→atomic.Store(map)→异步watchLoop
- **Copy-On-Write模式**：watchLoop中每次变更都创建新map副本→atomic.Store替换，读操作完全无锁
- **Hook机制**：`Hook(key, func(string))`注册回调，key变更时触发
- `Get/GetDef`：从内存缓存读（零网络开销），key不在前缀内自动降级DirectGet
- `DirectGet/DirectPut`：每次发起网络请求，强一致
- `DirectPutWithLease(key,val,ttl)`：带租约写入，到期自动删除（适用于心跳/临时令牌）
- 地址从环境变量`ETCD_ADDR`读取

#### 4.4 middleware包 — 网关中间件
- **SetHeaderInfo()**: 提取ClientIP/Lang/DeviceID/DeviceType到MetaInfo，Token优先读Header`X-User-Token`再降级Cookie`session_token`
- **BlackCheck()**: 调`rpc.IpClient().OnBlackList()`，**服务故障时放行**(fail-open)
- **WhiteCheck()**: 调`rpc.IpClient().OnWhiteList()`，**服务故障时拒绝**(fail-closed)
- **FontAuthMiddleware()**: Redis读`"user:token:"+token`→解析CUserTokenPayload→注入userId+NickName到MetaInfo
- **BackAuthMiddleware()**: Redis读`token`(无前缀)→解析BUserTokenPayload→注入userId+UserName→**额外调`rpc.BUserClient().CheckUrlAuth(ctx,fullPath)`做RBAC URL级权限校验**
- **RedirectS3()**: public路径直接302到S3Domain+Bucket+objKey；private路径PresignGetObject(默认15分钟)+`Cache-Control:no-store`
- **HertzAccLogStr()**: 访问日志`<trace_id=xxx> 200 - 1.23ms GET /api/...`

#### 4.5 tracer包 — 链路追踪8个Header
| Header | 用途 |
|--------|------|
| `X-Trace-Id` | 请求链路ID(UUID v4) |
| `X-Client-IP` | 客户端真实IP(网关注入) |
| `X-User-Token` | 用户登录Token |
| `X-User-ID` | 用户ID(认证后注入) |
| `X-User-Name` | 用户名/昵称 |
| `X-User-Language` | 语言偏好(i18n) |
| `X-User-Device-ID` | 设备标识 |
| `X-User-Device-Type` | 设备类型(iOS/Android/H5) |

所有Header通过`metainfo.WithPersistentValue`注入，跨RPC调用自动透传。

#### 4.6 ret包 — 统一响应
- 结构：`R{TraceId, Code(int32), Msg, Data}`
- HTTP层：`HttpOk(ctx,res)` / `HttpErr(ctx,err,trans)` / `HttpOne(ctx,data,trans,err)`
- RPC层：`BizErr(ctx,code,msg...)`→从Redis`i18n:hs:zhcn`读中文消息 / `RpcOne[T](ctx,s,err,code)`
- HttpErr翻译流程：BizStatusError→`trans.LangTransOne(ctx,errCode)`→翻译失败降级BizMessage()

#### 4.7 errs包 — 系统级错误码
OK=0, NG=1, OperateErr=6000001, ParamErr=6000002, IpIsNil=6000003, IpAnalysisErr=6000004, IpNotCorrect=6000005, IpInBlackTable=6000006, IpNotInWhiteTable=6000007, TokenNotExist=6000008, TokenAnalysisFailed=6000009, CheckUrlAuthFailed=6000010, EnumKeyNotFound=6000011, SmsError=6000012, S3PathErr=6000013, RpcRequestErr=6000014

#### 4.8 session包 — Token Payload
- **CUserTokenPayload(C端)**: ID, UID, LastLoginIp, LastLoginTime, Device, DeviceType, NickName → Redis Key=`user:token:{token}`
- **BUserTokenPayload(B端)**: ID, UserName(JSON tag误为"uid") → Redis Key=`{token}`(直接用token作key)

#### 4.9 consts包 — 常量体系
- **namesp**: 21个微服务注册名+ETCD端口/DB Key（如`/slg/serv/buser/port`）
- **redis_key**: Token前缀`user:token:`、枚举`enum:`(1h)、i18n(7天)、Banner`{banner:list}:`(5min)、Item`{item:list}:type/ids:`、短视频系列、经验等级系列
- **env**: ETCD配置路径 `/slg/conf/tidb|redis|nats|kafka|es|s3|sms|security/*`
- **etcd_key**: 12个注册开关(android/ios/h5×phone/email/guest)+白名单/黑名单总开关+GoogleAuth+人机验证+翻译URL
- **kafka**: Topic格式`kafka.service-name.business.topic` + ConsumerGroup
- **nats**: Subject格式`Nats.service-name.business.subject` + QueueGroup + 用户事件主题

#### 4.10 utils包 — 核心工具函数
- Snowflake: 纪元2024-01-01, 41/5/5/12位分配, 时钟回拨保护, GenerateUniqueID()全局入口
- 密码: `HashPassword(pwd,salt)`=md5(pwd+salt)→bcrypt(DefaultCost), `CheckPassword`=bcrypt.Compare
- Token: `GenToken()`=UUID v4
- 加密: AES-256-CBC+PKCS7, EncryptedString自定义GORM类型(自动加解密), AES-GCM(兼容PHP)
- JSON: sonic高性能, MarshalStr/UnmarshalStr等
- HTTP: 链式调用`NewHttpUt().SetUrl().SetMethodPost().SetJsonObjBody(body).Send().GetStringRes()`
- 脱敏: MaskPhone前3后3中间`*`, MaskEmail前2+`****@domain`, MaskName按长度策略
- SMS: `SendSms(country,mobile,content)`按区号匹配AppId(62=印尼,84=越南), POST cosms.cloud
- IP: `GetLocalIPv4()`非回环IPv4
- ISO: 246国家完整数据, `IsValidISO3166Alpha2`+`CountryCNByISO`

#### 4.11 kafka/nats包
- **Kafka Producer**: HashPartitioner, Idempotent(MaxOpenRequests=1), Retry=10/1s, 3秒Send超时
- **Kafka Consumer**: Range策略, 重试3次后放弃, panic恢复后重调handleExe
- **NATS**: ReconnectWait=2s, MaxReconnects=100, Namespace环境变量隔离
- **JetStream**: FileStorage持久化, JSPublishDelay延迟消息, JSBatchConsumeLoop(信号量并发控制), 失败NakWithDelay 5s

#### 4.12 rpc层
- **rpc_client.go**: 21个客户端(sync.Once单例), 已注释: CUser(001), Home(013)
- **init.go**: Server=(ETCD注册,端口复用,MuxTransport,TTHeader,访问日志MW) / Client=(ETCD发现,1200s超时,MuxConnection,TTHeader)
- **handler.go**: `Handler[REQ,RESP]`(标准绑定+RPC) / `HandlerNoParam[RESP]`(无参) / `HandlerWithCSV[REQ,RESP]`(CSV导出,CSVResponse接口)

---

### 五、双网关架构深度对比

#### 5.1 启动流程差异
两网关启动流程基本一致：lg.InitHLog→etcd.LoadAndWatch("/slg/conf/")→middleware.InitRedis→mys3.Init→端口获取→ETCD注册→Hertz Server启动
- **gate-back独有**：`WithMaxRequestBodySize(20<<20)` 20MB请求体（支持文件上传）
- **gate-font无此配置**：使用Hertz默认4MB限制

#### 5.2 路由组与中间件链

**gate-back（B端后台）**：
```
/admin/open/* → SetHeaderInfo → WhiteCheck(fail-closed)
/admin/api/*  → SetHeaderInfo → WhiteCheck → BackAuthMiddleware(Token+RBAC URL权限)
/s3/*path     → RedirectS3(无认证，预签名重定向)
GET /ping     → handler.Ping
```

**gate-font（C端前台）**：
```
/open/*  → SetHeaderInfo → BlackCheck(fail-open)
/api/*   → SetHeaderInfo → BlackCheck → FontAuthMiddleware(Token认证)
/s3/*path → RedirectS3(无认证，预签名重定向)
GET /ping → handler.Ping
```

#### 5.3 安全策略核心差异

| 对比项 | gate-back(B端) | gate-font(C端) |
|-------|---------------|---------------|
| 安全模型 | 默认拒绝(fail-closed) | 默认允许(fail-open) |
| IP校验 | 白名单(必须在名单才放行) | 黑名单(在名单才拦截) |
| IP服务故障时 | **禁行** | **放行** |
| Token Redis Key | `token`(无前缀) | `user:token:{token}`(有前缀) |
| Token Payload | BUserTokenPayload(UserName) | CUserTokenPayload(NickName/UID) |
| 额外权限 | **CheckUrlAuth RBAC URL级鉴权** | 无 |
| Cookie设置 | Login注释掉了SetCookie | Login设置`session_token`(Secure+HttpOnly+SameSiteNone) |

#### 5.4 gate-back 路由清单（~134条）

| 模块 | 路由数 | 组 | 关键路由示例 |
|------|--------|-----|------------|
| buser(用户/角色/菜单) | 25 | Open+Api | login, user/create/update/delete, role/*, menu/* |
| bcom(开关/黑白名单/枚举) | 16 | Open+Api | switch/config/*, black/client/*, white/backend/* |
| i18n(多语言) | 10 | Open+Api | lang/create/update/delete/list/import/load/export |
| blog(日志) | 2 | Api | queryActs, queryLogins |
| user(C端用户管理) | 6 | Open | queryUserPage/Info, queryLogPage, updateUser |
| enum(枚举) | 1 | Open | enum/options |
| kyc(KYC审核) | 3 | Api | audit, page, detail |
| item(道具) | 9 | Api | page, create, update, publish/* |
| app(Banner/分类) | 18+2swagger | Api+Open | banner/*, home/category/*, swagger |
| exp(VIP/升级) | 5 | Api | vipLevel/*, upgradeRules/* |
| s3(文件) | 4 | Open+Api | uploadFile, getUploadPreSign, uploadComplete, loadConfig |
| live(直播管理) | 16 | Api | rooms, close, top, ban, convention/*, audit/*, monitor, permission/* |
| shortvideos(审核) | 5 | Api | getNotReviewedVideoPage, videoAudit, videoOnline |
| rtc(房间管理) | 17 | Api | credential, adminMapping/*, room/*, monitor/* |

Handler实现模式：99%使用`rpc.Handler`一行代码转发；特殊：Login手动实现、RTC Monitor透传原始JSON、CSV导出用`rpc.HandlerWithCSV`

#### 5.5 gate-font 路由清单（~66条）

| 模块 | 路由数 | 组 | 关键路由示例 |
|------|--------|-----|------------|
| user(登录/注册/头像/黑名单/关注) | 20 | Open+Api | login, register, getUserDetail, avatar/*, black/*, follow/* |
| app(Banner/分类展示) | 3 | Open | banner/list, category/list, strategy/content |
| enum(枚举) | 1 | Open | options |
| kyc(用户提交) | 4 | Api | submit, query, uploadIdImage, deleteIdImage |
| shortvideos(完整用户功能) | 17 | Api | statis, comment, likes, focus, upload, publish, getVideoList |
| rtc(凭证) | 2 | Open+Api | getCredential, ping |
| live(主播/观众) | 18 | Api | room/info/start/close/enter/exit, audience, managers, blacklist, kick, mute |

---

### 六、七个微服务深度分析

#### 6.1 ser-buser — 后台用户权限服务（29个RPC方法）
- **数据库表**：sys_user, sys_role, sys_menu, sys_user_role(关联), sys_role_menu(关联)
- **RBAC体系**：超管(super_admin_flag="2")→系统管理员("1")→普通角色("0")
- **菜单类型**：M=目录, C=菜单, F=按钮；perms字段用于URL鉴权
- **登录流程**：IP白名单校验→查用户→验密(bcrypt)→TOTP验证→生成Token→Redis缓存30天→登录日志
- **CheckUrlAuth**：超管/系统管理员直接通过，普通角色按sys_menu.perms匹配URL
- **软删除**：delete_by=userId（非零表示已删除）
- **错误码范围**：6002000-6002045
- **RPC依赖**：ser-ip(OnWhiteList), ser-blog(AddActLog/AddLoginLog)

#### 6.2 ser-app — 运营后台应用配置（22个RPC方法）
- **数据库表**：banner, home_category
- **Banner排序事务**：ShiftAndCreate先将sort_value≥目标的记录+1再INSERT；ShiftAndUpdateFields同理
- **Banner缓存**：key=`banner:list:{pagePosition}:{terminal}`，TTL=5min，**空列表也缓存**防穿透
- **Banner图片多语言**：images字段为JSON数组，按请求语言过滤返回
- **URL生命周期**：写入时GetRelativePath存相对路径，读取时GetFullUrl按网关类型展开
- **错误码范围**：Banner 6011000-6011007, HomeCategory 6011100-6011104, ContentStrategy 6011200-6011204
- **RPC依赖**：ser-s3(UploadFile/DeleteFile, BsType=20), ser-contentcenter(GetStrategyContent), ser-blog(AddActLog)

#### 6.3 ser-bcom — 业务公共聚合服务（16个RPC方法）
- **数据库表**：query_enum (enum_key, enum_value逗号分隔, enum_label逗号分隔)
- **核心角色**：ser-ip和ser-buser的代理/聚合层，补充用户名信息
- **开关配置**：12种tab映射到不同ETCD key，JSON格式存储，Get/Put实现读写
- **枚举翻译**：拆分value/label后对每个label调i18n翻译
- **Email功能**：SendEmailVerifyCode/EmailVerify均为panic("implement me")，未完成
- **错误码范围**：6003000-6003017
- **RPC依赖**：ser-ip(黑白名单CRUD), ser-buser(QueryUserDetail), ser-i18n(LangTransOne), ser-blog(AddActLog)

#### 6.4 ser-i18n — 国际化服务（13个RPC方法）
- **数据库表**：langs (key=int64, 6种语言字段：zh_cn/zh_tw/en/id_id/vi_vn/th_th)
- **Redis存储**：6个Hash表`i18n:hs:{zhcn|zhtw|en|id|vi|th}`，每个Hash以翻译key为field
- **翻译降级**：请求语言→zh-CN→en（三级降级）
- **批量保存**：upsert逻辑，已有记录合并非空字段，Pipeline批量HSET
- **LoadLangs**：全量DB→Redis加载，服务启动或手动触发
- **导出格式**：CSV(BOM+UTF-8)/JSON/XML，1000条/批自动分页
- **Category**：1=客户端业务...6=服务端错误，key首字符必须与category匹配
- **外部翻译**：GetTrans HTTP调用外部AI翻译API（URL从ETCD`/slg/serv/ai-trans/url`读取）
- **错误码范围**：操作6004000-6004006, 参数6004100-6004106
- **RPC依赖**：ser-buser(QueryUserDetail/QueryUserByIds), ser-blog(AddActLog)

#### 6.5 ser-ip — IP黑白名单服务（10个RPC方法）
- **数据库表**：c_black_list(国家ISO Code), sys_white_list(IP地址) — **硬删除，无delete字段**
- **黑名单机制**：ETCD开关`/slg/switch/bcom/access/client/black`→关闭直接放行→`third.IpAnalysis(clientIp)`第三方解析国家→查DB
- **白名单机制**：ETCD开关`/slg/switch/bcom/access/backend/white`→关闭直接放行→`third.IpAnalysisByIp`→查DB
- **ISO校验**：`utils.IsValidISO3166Alpha2`标准校验 + `utils.CountryCNByISO`查中文国名
- **错误码范围**：6005000-6005016
- **RPC依赖**：第三方IP库(third.IpAnalysis), ser-blog(AddActLog)
- **被依赖**：ser-buser(Login白名单), gate-back(WhiteCheck), gate-font(BlackCheck), ser-bcom(代理)

#### 6.6 ser-item — 道具服务（11个RPC方法）— 含Domain层DDD设计
- **数据库表**：item(item_code格式"D"+3000+id), item_publish(按类型一对多，支持上下架/标签/排序)
- **Domain层（项目中唯一的DDD实践）**：
  - `ItemValidator`接口：名称3-36字符+3个URL非空+价格校验(EffectIcon必须0/其他>0)+URL自动转相对路径
  - `ItemMapper/ItemPublishMapper`：DB model↔RPC DTO双向转换，URL展开/收缩
  - `ItemCacheManager`：**Lua SCAN脚本**批量清除`item:list:*`前缀缓存（避免KEYS阻塞）
  - `ItemLogBuilder`：逐字段对比生成变更日志，审计粒度细
  - `ToggleStatus`值对象：封装开关反转逻辑
  - `S3ErrorMapper`：(bsType,s3ErrorCode)→领域错误码二维映射
  - `ValidateItemTypes`：`"1"/"2"/"3"/"1,2"/"2,1"`白名单校验
- **创建事务**：INSERT item → 生成item_code("D"+(3000+id)) → 批量INSERT item_publish(每种类型一条)
- **更新事务**：UPDATE item → 比对desired/existing item_publish → 恢复/新增/软删除
- **item_type**: 1=直播间,2=语聊房,3=专属活动; is_online: 1=下线,2=上线; front_show: 1=不展示,2=展示; tag: 1=热门,2=新品,3=推荐
- **错误码范围**：道具6010000-6010021, S3相关6007100-6007110
- **RPC依赖**：ser-s3(UploadIcon, BsType=1/2/3), ser-buser(QueryUserByIds), ser-blog(AddActLog)

#### 6.7 ser-s3 — 文件存储代理服务（11个RPC方法）
- **数据库表**：s3_config(业务类型配置,path_template含占位符), s3_documents(文件记录,status:1上传中/2已上传/3已取消/4已删除)
- **两种S3客户端**：Client(内网Endpoint上传) + PresignClient(外网S3Domain预签名)
- **三种上传方式**：
  1. `GetUploadPreSign`→前端直传S3→`UploadComplete`确认(读前512字节验MIME)
  2. `UploadFile`小文件≤10MB直传(走RPC传输二进制)
  3. `UploadIcon`道具图标(MD5+时间戳命名,不走s3_config)
- **s3_config机制**：Redis Hash`S3HsConfig`缓存所有业务类型配置，path_template支持`{owner_id}/{year}/{month}/{day}`占位符
- **MIME校验**：magic bytes检测 GIF(474946)/PNG(89504E47)/JPEG(FFD8FF)/MP4(偏移4字节66747970)/PAG/JSON
- **错误码范围**：操作6007000-6007002, 参数6007100-6007112
- **RPC依赖**：ser-blog(AddActLog)
- **被依赖**：ser-app(Banner图片), ser-item(道具图标), gate-back/gate-font(S3重定向)

---

### 七、跨服务RPC调用关系图

```
ser-buser ──→ ser-ip       (Login: OnWhiteList)
ser-buser ──→ ser-blog     (所有写操作: AddActLog, AddLoginLog)

ser-bcom  ──→ ser-ip       (黑白名单CRUD代理)
ser-bcom  ──→ ser-buser    (QueryUserDetail/QueryUserByIds 补充用户名)
ser-bcom  ──→ ser-i18n     (LangTransOne 枚举标签翻译)
ser-bcom  ──→ ser-blog     (SwitchConfig: AddActLog)

ser-app   ──→ ser-s3       (UploadFile BsType=20, DeleteFile)
ser-app   ──→ ser-contentcenter  (GetStrategyContent)
ser-app   ──→ ser-blog     (所有写操作: AddActLog)

ser-item  ──→ ser-s3       (UploadIcon BsType=1/2/3)
ser-item  ──→ ser-buser    (QueryUserByIds 补充用户名)
ser-item  ──→ ser-blog     (所有写操作: AddActLog)

ser-i18n  ──→ ser-buser    (QueryUserDetail/QueryUserByIds 补充用户名)
ser-i18n  ──→ ser-blog     (写操作: AddActLog)
ser-i18n  ──→ [外部翻译API] (GetTrans, HTTP调用)

ser-ip    ──→ ser-blog     (写操作: AddActLog)
ser-ip    ──→ [第三方IP库]  (third.IpAnalysis/IpAnalysisByIp)

ser-s3    ──→ ser-blog     (Upload/Delete: AddActLog)

gate-back ──→ ser-ip       (WhiteCheck: OnWhiteList)
gate-font ──→ ser-ip       (BlackCheck: OnBlackList)
gate-back ──→ ser-buser    (BackAuthMiddleware: CheckUrlAuth)
```

**依赖层次**：
- 基础层(被所有服务依赖)：ser-blog(审计日志，本地无源码)
- 基础服务：ser-ip, ser-s3, ser-i18n
- 业务聚合：ser-bcom(代理ser-ip+补充用户名)
- 核心业务：ser-buser, ser-app, ser-item
- 接入层：gate-back, gate-font

---

### 八、关键设计模式与约定

#### 8.1 软删除两种模式
- **时间戳型**：`delete_at=unix_timestamp`(0=未删除) — banner, home_category, item_publish, s3_documents
- **操作人ID型**：`delete_by=userId`(0=未删除) — sys_user, sys_menu, sys_role, sys_user_role
- **硬删除**：c_black_list, sys_white_list（ser-ip）

#### 8.2 操作日志统一模式
所有服务写操作均调用`rpc.BLogClient().AddActLog(ctx, &ActAddReq{Backend, Model, Page, Action, IsSuccess})`，通过Kitex context自动透传UserId/UserName/ClientIp/TraceId。

#### 8.3 枚举体系
- `EnumOption{Value int32, Label string}` + 泛型翻译函数`TranslateEnum[T]/TranslateEnumOptions[T]`
- `GateType`: 0=C端网关(gate-font), 1=B端网关(gate-back)
- NATS事件消息泛型：`EventMessage[T]{EventID, EventType, UserID, CreateAt, Data T}`

#### 8.4 缓存策略
- Banner缓存：**空列表也缓存**防穿透，枚举pagePosition×terminal全组合清除
- Item缓存：**Lua SCAN脚本**批量清除（非KEYS），按type和ids两种维度缓存
- i18n缓存：6个Redis Hash表常驻，LoadLangs全量加载+Pipeline批量HSET
- s3_config缓存：Redis Hash`S3HsConfig`，服务启动预加载

#### 8.5 关键约定
- ETCD服务名：`xxx_service`，键格式：`/slg/serv/service-name/port|db`
- Redis键：`prefix:subkey:id`，各有TTL常量，集群模式Hash Tag用`{}`
- Kafka Topic：`kafka.service-name.business.topic`
- NATS Subject：`Nats.service-name.business.subject`，Namespace环境变量隔离
- 错误码：OK=0, NG=1, 系统级6000000+, 各服务6002000+/6003000+/6004000+/6005000+/6007000+/6010000+/6011000+
- 生成代码（kitex_gen/和gorm_gen/）不手动修改，只改IDL和gen配置后重新生成
- 审计字段：create_by/create_at/update_by/update_at 通过GORM UserPlugin自动填充
