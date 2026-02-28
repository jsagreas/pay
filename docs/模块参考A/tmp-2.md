# Java 参考工程 — 钱包模块深度分析

> 编写时间：2026-02-27
> 定位：从已投产的 Java 钱包工程中**提取可借鉴的设计思想**，为 Go ser-wallet 模块提供参考
> 核心原则：**借鉴而非照搬** — 识别"与技术栈无关的设计本质"，过滤掉"Java/Spring 专属细节"
> 分析来源：`/Users/mac/gitlab/z-tmp/backend/`（完整 Java 微服务后端）
> 核心模块：`p9-services/wallet-service`（29 个 Java 源文件 + 3 个 Mapper XML + 配置，端口 8907）
> 关联模块：`p9-core-lib`（共享枚举/VO/常量/业务类）、`p9-services/merchant-service`（币种/汇率管理）
> 前端交互：`p9-live-services/websocket-service`（C端通过 WebSocket 进行充值/提现/查余额）
> 技术栈：Spring Boot 2.7.4 + Spring Cloud 2021.0.4 + MyBatis-Plus + RabbitMQ + Redis/Redisson + Java 17

---

## 目录

- [一、参考工程整体架构](#一参考工程整体架构)
- [二、wallet-service 工程结构](#二wallet-service-工程结构)
- [三、数据模型深度分析](#三数据模型深度分析)
- [四、核心业务逻辑拆解](#四核心业务逻辑拆解)
- [五、SQL 原子操作（5 条关键 SQL）](#五sql-原子操作5-条关键-sql)
- [六、冻结 / 解冻扣减 / 冻结回滚](#六冻结--解冻扣减--冻结回滚)
- [七、并发控制与分布式锁](#七并发控制与分布式锁)
- [八、消息队列驱动模型](#八消息队列驱动模型)
- [九、币种与汇率管理体系](#九币种与汇率管理体系)
- [十、跨服务调用拓扑](#十跨服务调用拓扑)
- [十一、C 端钱包交互链路](#十一c-端钱包交互链路)
- [十二、枚举与 VO 体系](#十二枚举与-vo-体系)
- [十三、定时任务](#十三定时任务)
- [十四、设计优劣评估](#十四设计优劣评估)
- [十五、可借鉴设计提炼 + Go 平移映射](#十五可借鉴设计提炼--go-平移映射)
- [附录 A：wallet-service 完整文件清单](#附录-a-wallet-service-完整文件清单)
- [附录 B：Feign 依赖 + Redis Key + MQ 队列速查](#附录-b-feign-依赖--redis-key--mq-队列速查)

---

## 一、参考工程整体架构

### 1.1 项目概览

```
p9-live-backend/                        # 根 POM，7 个子模块
├── p9-infrastructure/                  # API 网关（Spring Cloud Gateway，端口 8886）
├── p9-core-lib/                        # 共享 JAR（DTO/VO/枚举/工具/常量/业务类）
├── p9-services/                        # 核心业务服务（9 个）
│   ├── auth-service         (8903)     # 认证
│   ├── user-service         (8906)     # 用户管理
│   ├── wallet-service       (8907)     # ★ 钱包（本次分析重点）
│   ├── merchant-service     (8902)     # 商户/币种/汇率管理
│   ├── field-service        (8901)     # 游戏桌台管理
│   ├── api-service          (8910)     # 外部游戏 API 对接
│   ├── system-business-svc  (8904)     # 系统业务（报表/审计）
│   ├── file-service         (8909)     # 文件上传
│   └── report-service       (8915)     # 报表
├── p9-live-services/                   # 游戏服务（18+个，每种游戏一个微服务）
│   ├── order-service        (8908)     # 投注订单
│   ├── websocket-service    (8889)     # WebSocket 推送（16GB RAM 最重服务）
│   ├── common-service       (lib)      # 游戏公共逻辑（非独立部署）
│   ├── baccarat-service     (8905)     # 百家乐
│   ├── bull-service         (8911)     # 牛牛
│   └── ... 另 14 个游戏服务
├── p9-backstage-services/              # 管理后台（4 个服务）
├── p9-job-executor/                    # XXL-Job 定时任务
└── p9-datasource-services/             # Netty 数据源聚合
```

### 1.2 关键发现：pay-service 不在本仓库

**pay-service（端口 8916）是独立部署的外部服务**，本仓库未包含其代码。证据：
- 网关配置：`/third/pay/**` 和 `/callback/**` 路由到 `pay-service:8916`
- K8s 部署：有独立的 `kubernetes-pay-template.yml`，使用独立的 ConfigMap（`pay-bootstrap`、`pay-conf`）
- 本仓库 `p9-services/` 下没有 `pay-service` 目录

**这意味着**：充值/提现的**支付渠道对接、回调处理**都在另一个代码库。wallet-service 只负责**余额变动和账变记录**，不负责与支付渠道的交互。

### 1.3 技术栈对比

| 维度 | Java 参考工程 | 我们 Go 工程 |
|------|-------------|-------------|
| 框架 | Spring Cloud + Spring Boot 2.7.4 | CloudWeGo Kitex + Hertz |
| ORM | MyBatis-Plus 3.5.3 | GORM Gen |
| 数据库 | MySQL | TiDB（兼容 MySQL） |
| 缓存 | Redis + Redisson（分布式锁） | go-redis v9 |
| 消息队列 | RabbitMQ（重度依赖） | NATS/JetStream |
| 服务调用 | Feign（HTTP REST） | Kitex（Thrift RPC） |
| 注册中心 | K8s Service（硬编码 URL） | ETCD |
| ID 生成 | 自研 OrderUtil（前缀+时间戳+随机） | Snowflake |
| 定时任务 | XXL-Job | robfig/cron |
| 部署 | K8s + Jib | K8s |

---

## 二、wallet-service 工程结构

### 2.1 完整目录树

```
wallet-service/
├── pom.xml                                    # Maven 依赖
└── src/main/
    ├── java/com/maya/p9/
    │   ├── WalletServiceRunner.java           # 启动类（Spring Boot Main）
    │   │
    │   ├── controller/                        # ── HTTP 入口层 ──
    │   │   ├── WalletController.java          # 钱包核心：9 个端点（260行）
    │   │   ├── CoinRecordController.java      # 账变查询：7 个端点（100行）
    │   │   └── xxljob/
    │   │       └── DeleteTempController.java   # 定时清理入口
    │   │
    │   ├── service/                           # ── 业务层 ──
    │   │   ├── CoinRecordService.java         # 账变服务接口
    │   │   ├── CoinRecordDataService.java     # 账变数据查询接口
    │   │   ├── UserCoinService.java           # 钱包查询接口
    │   │   ├── CommonService.java             # 通用服务接口
    │   │   ├── AddCoinMessageService.java     # MQ 消息状态接口
    │   │   │
    │   │   ├── impl/                          # ── 业务实现 ──
    │   │   │   ├── CoinRecordServiceImpl.java # ★ 核心！522 行，所有余额变动逻辑
    │   │   │   ├── CoinRecordDataServiceImpl  # 账变分页/列表查询（182行）
    │   │   │   ├── UserCoinServiceImpl.java   # 余额查询（87行）
    │   │   │   ├── CommonServiceImpl.java     # 用户/商户信息获取+Redis缓存（74行）
    │   │   │   └── AddCoinMessageServiceImpl  # MQ 消息状态更新（21行）
    │   │   │
    │   │   └── feign/                         # ── 远程调用客户端 ──
    │   │       ├── OrderResource.java         # → order-service (8908)
    │   │       ├── ApiResource.java           # → api-service (8910)
    │   │       ├── UserService.java           # → user-service (8906)
    │   │       └── MerchantFeignResource.java # → merchant-service (8902)
    │   │
    │   ├── dto/                               # ── 数据库实体 ──
    │   │   ├── UserCoinDTO.java               # user_coin 表（32行）
    │   │   ├── CoinRecordDTO.java             # coin_record 表（59行）
    │   │   └── OrderMessageDTO.java           # add_coin_message 表
    │   │
    │   ├── repository/                        # ── 持久化层 ──
    │   │   ├── UserCoinRepository.java        # BaseMapper + 5 个自定义 SQL
    │   │   ├── CoinRecordRepository.java      # BaseMapper + 4 个自定义查询
    │   │   └── AddCoinMessageRepository.java  # 消息幂等表
    │   │
    │   ├── rabbitmq/                          # ── 消息驱动层 ──
    │   │   ├── listener/
    │   │   │   └── AddCoinListener.java       # 4 个队列消费者（147行）
    │   │   └── RabbitSender.java              # 消息发送
    │   │
    │   ├── rabbit/config/
    │   │   └── WallRabbitTemplateConfig.java  # RabbitMQ 模板配置
    │   │
    │   ├── task/
    │   │   └── BackWaterDayTask.java          # 数据清理任务（57行）
    │   │
    │   └── vo/
    │       └── BalanceVO.java                 # 余额操作 VO（id + amount + coinType）
    │
    └── resources/
        ├── bootstrap.yml                      # 端口 8907，RabbitMQ CORRELATED
        ├── logback.xml                        # 日志配置
        └── mapper/
            ├── UserCoinMapper.xml             # 5 条原子 SQL（44行）★ 最关键
            ├── CoinRecordMapper.xml           # 4 条复杂查询（284行）
            └── AddCoinMessageMapper.xml       # 消息状态更新
```

### 2.2 规模统计

| 维度 | 数量 |
|------|------|
| Java 源文件 | 29 个 |
| Mapper XML | 3 个 |
| HTTP 端点 | 16 个（WalletController 9 + CoinRecordController 7） |
| MQ 消费队列 | 4 个 |
| 数据表 | 3 张（user_coin / coin_record / add_coin_message） |
| Feign 远程调用 | 4 个（order / api / user / merchant） |
| 核心业务行数 | ~522 行（CoinRecordServiceImpl） |

---

## 三、数据模型深度分析

### 3.1 user_coin — 用户钱包主表

**来源**：`UserCoinDTO.java`（@TableName("user_coin")），继承 `BasicDataBaseDTO`

```sql
CREATE TABLE user_coin (
    id              BIGINT       PRIMARY KEY AUTO_INCREMENT,
    user_id         VARCHAR(64)  NOT NULL,              -- 用户 ID
    user_name       VARCHAR(128),                        -- 用户名（冗余）
    amount          DECIMAL(20,2) DEFAULT 0,             -- ★ 总余额（含冻结部分）
    freeze_amount   DECIMAL(20,2) DEFAULT 0,             -- ★ 冻结金额
    user_state      INT,                                 -- 游戏状态（不该放这里）
    game_type       INT,                                 -- 当前游戏（不该放这里）
    wallet_type     INT,                                 -- 钱包类型 1=转账 2=免转
    currency        VARCHAR(16),                         -- 币种代码
    remark          VARCHAR(512),
    creator         BIGINT,
    updater         BIGINT,
    created_time    DATETIME,
    updated_time    DATETIME
);
-- 可用余额 = amount - freeze_amount（应用层计算，非存储字段）
```

**关键设计特征**：

| 特征 | 具体表现 | 评价 |
|------|---------|------|
| 单用户单记录 | `selectOne(new QueryWrapper(userId))` — 一个用户只有一条 | 简单但不支持多钱包类型 |
| amount 含冻结 | 可用余额 = amount - freeze_amount | 需运算，不如直接存 available |
| 无版本号 | 完全依赖 Redis 分布式锁 | 缺少 DB 层并发兜底 |
| 单币种 | 一条记录一个 currency | 不支持同用户多币种 |
| 冗余游戏状态 | user_state / game_type 放在钱包表 | 耦合不当 |
| 精度 DECIMAL(20,2) | 只到 2 位小数 | 对加密货币不够（我们需要 DECIMAL(20,8)） |

### 3.2 coin_record — 账变流水表

**来源**：`CoinRecordDTO.java`（@TableName("coin_record")）

```sql
CREATE TABLE coin_record (
    id              BIGINT       PRIMARY KEY AUTO_INCREMENT,
    user_id         VARCHAR(64)  NOT NULL,
    user_name       VARCHAR(128),                        -- 冗余
    user_type       INT,                                 -- 0=正式 1=试玩
    nick_name       VARCHAR(128),                        -- 冗余
    user_belong     INT,                                 -- 用户所属
    merchant_id     BIGINT,                              -- 冗余
    merchant_name   VARCHAR(128),                        -- 冗余
    merchant_no     VARCHAR(64),                         -- 冗余
    agent_no        VARCHAR(64),                         -- 冗余
    agent_id        BIGINT,                              -- 冗余
    agent_name      VARCHAR(128),                        -- 冗余
    path            VARCHAR(512),                        -- 代理链路（冗余）
    order_no        VARCHAR(128) NOT NULL,               -- ★ 关联订单号
    coin_type       INT          NOT NULL,               -- ★ 账变类型（1~9）
    coin_detail     INT,                                 -- ★ 账变明细（1~8）
    coin_value      DECIMAL(20,2) NOT NULL,              -- ★ 变动金额（绝对值，正数）
    coin_from       DECIMAL(20,2),                       -- ★ 变动前余额快照
    coin_to         DECIMAL(20,2),                       -- ★ 变动后余额快照
    coin_amount     DECIMAL(20,2),                       -- 当前金额（= coin_to）
    balance_type    INT          NOT NULL,               -- ★ 1=收入 2=支出
    remark          VARCHAR(512),
    creator         BIGINT,
    updater         BIGINT,
    created_time    DATETIME,
    updated_time    DATETIME
    -- ⚠️ 没有 order_no 唯一索引！没有幂等保障！
);
```

**关键设计特征**：

| 特征 | 具体表现 | 评价 |
|------|---------|------|
| 重度冗余 | 10+ 个用户/商户/代理字段 | 查询无需 JOIN，但写入成本高 |
| 前后快照 | coin_from / coin_to 记录变动前后余额 | ★ 审计必需，可借鉴 |
| coin_value 恒正 | 通过 balance_type 区分收入/支出 | 设计合理 |
| 双层类型 | coin_type(大类) + coin_detail(明细) | 有一定灵活性但稍冗余 |
| ⚠️ 无幂等字段 | order_no 无唯一索引 | **严重缺陷** — MQ 重投可产生重复账变 |

### 3.3 add_coin_message — MQ 消息幂等表

```sql
CREATE TABLE add_coin_message (
    id              BIGINT       PRIMARY KEY AUTO_INCREMENT,
    message_id      VARCHAR(128),
    order_id_list   TEXT,             -- JSON 格式订单列表
    status          INT DEFAULT 1,    -- 1=成功 2=失败
    try_count       INT DEFAULT 0,
    create_time     DATETIME,
    update_time     DATETIME
);
```

> 这张表的设计初衷是保证 MQ 消费幂等，但实际代码中 `deleteAddCoinMessage()` 被注释掉了，说明可能实际并未生效。我们不使用 RabbitMQ，但幂等思想可以迁移到 RPC 幂等方案中。

### 3.4 数据模型对比总表

| 维度 | Java 参考工程 | 我们 ser-wallet 设计方向 |
|------|-------------|----------------------|
| 钱包粒度 | 1 用户 = 1 条记录 | 1 用户 × 钱包类型 × 币种 = 1 条 |
| 钱包类型 | 2 种（转账/免转） | 5 种（中心/奖励/主播/代理/场馆） |
| 余额字段 | amount(含冻结) + freeze_amount | available(可用) + frozen(冻结) 分开 |
| 精度 | DECIMAL(20,2) | DECIMAL(20,8) |
| 账变冗余 | 10+ 字段冗余 | 轻冗余（只存 user_id + wallet_id） |
| 幂等机制 | ⚠️ 无（order_no 无唯一索引） | 双层（Redis SetNX + DB unique(biz_no)） |
| 版本号 | 无 | 考虑引入 version |
| 币种管理 | merchant-service 持有 | 我们自己 ser-wallet 持有 |

---

## 四、核心业务逻辑拆解

### 4.1 addCoin — 最核心方法（CoinRecordServiceImpl:169-341，约 170 行）

这是整个钱包系统**唯一的余额变动入口**。所有加钱减钱（下注、结算、充值到账、提现扣款、打赏等）都走这一个方法。

**完整调用链**：

```
addCoin(CoinRecordVO)
  │
  ├── 1. 参数校验：coinType 必须 > 0
  ├── 2. commonService.getUserInfoByUserId(userId)
  │      └── Redis 缓存优先，miss 则 Feign 调 user-service
  ├── 3. setVOToUserInfo()  ← 填充用户/商户/代理信息到 coinRecord
  │
  ├── 4. 【分支】根据 walletType 分叉 ─────────────────────────
  │
  │   ├── [TRANSFER 转账钱包 — 本地管理余额] ────────────────
  │   │   │
  │   │   ├── 查询 user_coin
  │   │   ├── [已有记录]
  │   │   │   ├── insufficientBalance()  ← 余额校验
  │   │   │   │   └── 只对 TRANSFER_OUT / BET / SEND_REWARD 三种操作校验
  │   │   │   │       可用余额 = amount - freezeAmount
  │   │   │   │       要求 可用余额 >= coinValue
  │   │   │   │
  │   │   │   ├── [收入 balanceType=1]
  │   │   │   │   └── userCoinRepository.addBalance()  ← SQL 原子加
  │   │   │   │       coinTo = amount + coinValue
  │   │   │   │
  │   │   │   └── [支出 balanceType=2]
  │   │   │       └── userCoinRepository.subBalance()  ← SQL 原子减
  │   │   │           带 WHERE (amount - X) >= 0 兜底
  │   │   │           例外：coinType=5(取消) 或 6(回滚) 跳过非负检查
  │   │   │           返回 affected rows，0 则余额不足
  │   │   │           coinTo = amount - coinValue
  │   │   │
  │   │   └── [无记录 → 首次创建]
  │   │       ├── checkCoinChange() ← 首次只能是收入
  │   │       ├── INSERT user_coin(userId, amount=coinValue, freeze=0)
  │   │       └── coinFrom=0, coinTo=coinValue
  │   │
  │   └── [FREE_TRANSFER 免转钱包 — 三方管理余额] ──────────
  │       ├── 调 order-service 查订单
  │       ├── 调 merchant-service 获取商户详情
  │       ├── 包装 ThirdOrderRequestVO → 调 api-service 三方加减额
  │       ├── [成功] 记录 thirdOrderRecord + MQ 通知余额变更
  │       ├── [失败] 记录失败状态 + return false
  │       └── coinFrom/coinTo 设 null（免转不记本地余额）
  │
  └── 5. INSERT coin_record（账变流水）
```

**核心洞察**：

1. **单入口设计**：所有余额变动走 `addCoin` 一个方法（收入+支出通过 `balanceType` 区分），优点是集中控制，缺点是方法过长（170+ 行，两种钱包类型 + 收支双向全混在一起）

2. **双钱包分叉**：TRANSFER（本地余额） vs FREE_TRANSFER（三方管理）。**我们的系统没有 FREE_TRANSFER 概念**，整个 else 分支（约 80 行）可以忽略

3. **惰性钱包创建**：用户首次产生账变时才创建 user_coin 记录，不是注册时创建。节省了空间

4. **每次账变都查用户信息**：`commonService.getUserInfoByUserId()` 每次调用，返回用户名、商户、代理等 10+ 字段，写入 coinRecord 形成冗余。有 Redis 缓存兜底

### 4.2 deductCoin — 纯记录不改余额（CoinRecordServiceImpl:112-149）

```
deductCoin(CoinRecordVO)
  │
  ├── coinValue > 0 校验
  ├── getUserInfo → 填充信息
  ├── [FREE_TRANSFER] → coinFrom/coinTo 设 null
  └── 只写 coin_record，★ 不改 user_coin 余额
```

**只记录流水不改余额**。用于免转钱包场景（余额在三方管理）。我们的系统应该**不需要这个方法**。

### 4.3 getWallet — 余额查询（UserCoinServiceImpl:37-72）

```
getWallet(UserCoinVO)
  │
  ├── getUserInfo → 判断 walletType
  │
  ├── [TRANSFER]
  │   ├── 查询 user_coin
  │   ├── [有记录] → ★ 返回 amount - freezeAmount
  │   │   └── [API签名 + 非正式用户] → 强制返回 0（安全防线）
  │   └── [无记录] → 返回 0
  │
  └── [FREE_TRANSFER]
      └── 返回 amount=0（余额在三方）
```

**安全防线**：`md5Sign` 不为空 且 非正式用户 → 返回 0。防止试玩用户通过 API 接口获取到余额信息被恶意利用。

### 4.4 CommonServiceImpl — 用户信息缓存策略

```java
// getUserInfoByUserId 的缓存逻辑
String redisKey = "user-service::userInfo::userId::" + userId;
Object obj = redisUtil.get(redisKey);
if (obj instanceof UserInfoVO) {
    return (UserInfoVO) obj;   // ★ Redis 命中直接返回
}
// miss → Feign 调 user-service → 无 TTL 写入 Redis（由 user-service 侧管理过期）
UserInfoVO userInfoVO = userResource.getUserByUserId(userId);
return userInfoVO;
```

```java
// getMerchantDetailVO 的缓存逻辑
String redisKey = MerchantCacheBusiness.getRedisKeyForGetMerchantDetailVO(merchantId);
// miss → Feign 调 merchant-service → ★ 设 TTL 1 天
redisUtil.set(redisKey, merchantDetailVO, 1, TimeUnit.DAYS);
```

**借鉴点**：高频调用（用户信息）走 Redis 缓存 + 由来源服务管理过期。低频调用（商户详情）自行设 TTL。

---

## 五、SQL 原子操作（5 条关键 SQL）

**来源**：`UserCoinMapper.xml`（44 行，整个钱包系统**最底层的 5 个原子操作**）

### 5.1 五操作速查表

| 操作 | SQL 语义 | amount 变化 | freeze_amount 变化 | 产生账变 |
|------|---------|-----------|-------------------|---------|
| addBalance | `amount += X` | +X | 不变 | 是 |
| subBalance | `amount -= X`（条件非负*） | -X | 不变 | 是 |
| addFreezeBalance | `freeze += X` | 不变 | +X | 否 |
| subFreezeBalance | `amount -= X, freeze -= X` | -X | -X | 是 |
| rollbackFreezeBalance | `freeze -= X` | 不变 | -X | 否 |

### 5.2 逐条 SQL 分析

```xml
<!-- 1. 加余额：无条件，直接加 -->
<update id="addBalance">
    UPDATE user_coin SET amount = amount + #{vo.amount},
    updated_time = #{vo.updateTime}
    WHERE id = #{vo.id}
</update>

<!-- 2. 减余额：有条件非负检查，结算回滚/取消 除外 -->
<update id="subBalance">
    UPDATE user_coin SET amount = amount - #{vo.amount},
    updated_time = #{vo.updateTime}
    WHERE id = #{vo.id}
    <!-- ★ coinType=6(结算回滚) 或 5(取消) 跳过非负检查 → 允许余额变负 -->
    <if test="vo.coinType != 6 and vo.coinType != 5">
        AND (amount - #{vo.amount}) >= 0
    </if>
</update>

<!-- 3. 冻结：freeze 增加，amount 不变 → 可用余额隐式减少 -->
<update id="addFreezeBalance">
    UPDATE user_coin SET freeze_amount = freeze_amount + #{vo.amount},
    updated_time = #{vo.updateTime}
    WHERE id = #{vo.id}
</update>

<!-- 4. 解冻扣减：amount 和 freeze 同时减少 → 钱真正离开钱包 -->
<update id="subFreezeBalance">
    UPDATE user_coin SET amount = amount - #{vo.amount},
    freeze_amount = freeze_amount - #{vo.amount},
    updated_time = #{vo.updateTime}
    WHERE id = #{vo.id}
</update>

<!-- 5. 冻结回滚：只减 freeze → 可用余额恢复 -->
<update id="rollbackFreezeBalance">
    UPDATE user_coin SET freeze_amount = freeze_amount - #{vo.amount},
    updated_time = #{vo.updateTime}
    WHERE id = #{vo.id}
</update>
```

### 5.3 设计思想提炼（与技术栈无关）

**核心**：**余额变动在 SQL 层使用 `SET amount = amount ± value` 实现原子操作**，避免应用层的 读-改-写 竞态条件。

**对应到 GORM**：
```go
// 加余额
db.Model(&WalletAccount{}).Where("id = ?", id).
    Update("available", gorm.Expr("available + ?", amount))

// 减余额（含非负兜底）
result := db.Model(&WalletAccount{}).Where("id = ? AND available >= ?", id, amount).
    Update("available", gorm.Expr("available - ?", amount))
if result.RowsAffected == 0 { /* 余额不足 */ }
```

---

## 六、冻结 / 解冻扣减 / 冻结回滚

### 6.1 三操作对照

| 操作 | 方法 | amount | freeze | 可用余额 | 账变 | 场景 |
|------|------|--------|--------|---------|------|------|
| 冻结 | freezeCoin | 不变 | +X | -X | 否 | 提现申请时锁定资金 |
| 解冻扣减 | subFreezeBalance | -X | -X | 不变 | **是** | 提现成功，钱真正离开 |
| 冻结回滚 | rollbackFreezeBalance | 不变 | -X | +X | 否 | 提现失败/取消，资金解冻 |

### 6.2 余额模型图解

```
                      amount (总余额 = 可用 + 冻结)
┌──────────────────────────────────────────────┐
│   可用余额 (amount - freeze)   │  冻结 (freeze)  │
└──────────────────────────────────────────────┘

示例（提现 100）:
  初始：amount=1000, freeze=0,   可用=1000
  冻结：amount=1000, freeze=100, 可用=900     ← freezeCoin

  ┌── 提现成功 → subFreezeBalance
  │   结果：amount=900,  freeze=0,   可用=900   ← 钱真的走了
  │
  └── 提现失败 → rollbackFreezeBalance
      结果：amount=1000, freeze=0,   可用=1000  ← 钱回来了
```

### 6.3 前置校验分析

```java
// freezeCoin 校验：
if (amount < freezeAmount) return USER_WALLET_NOT_ENOUGH_BALANCE;

// subFreezeBalance 校验：
if (freeze_amount < freezeAmount) return USER_WALLET_NOT_ENOUGH_BALANCE;

// rollbackFreezeBalance 校验：
// ⚠️ 无校验！直接执行 freeze -= X
// 如果并发导致 freeze < X，可能出现负冻结
// 但外层有 Redisson 锁保护串行化
```

### 6.4 解冻扣减时产生账变

`subFreezeBalance()` 内部调用 `createOrder()` 方法产生一条 coin_record：

```java
// createOrder 逻辑（CoinRecordServiceImpl:495-520）
coinRecordDTO.setCoinTo(userCoin.getAmount() - freezeAmount);  // 扣减后余额
coinRecordDTO.setCoinValue(freezeAmount);                       // 变动金额
coinRecordDTO.setOrderNo(coinFreezeRecordVO.getOrderNo());      // 关联订单
coinRecordRepository.insert(coinRecordDTO);
```

### 6.5 我们应该如何借鉴

| 维度 | Java 做法 | 我们的增强方向 |
|------|----------|-------------|
| 字段设计 | amount(含冻结) + freeze | available + frozen 分开（更直观） |
| 冻结 | amount 不动 + freeze++ | available-- + frozen++ |
| 解冻扣减 | amount-- + freeze-- | frozen--（available 不变） |
| 回滚 | freeze-- | available++ + frozen-- |
| 冻结是否产生账变 | 否 | **建议产生**（可追溯冻结动作） |
| rollback 校验 | ⚠️ 无 | SQL 加 `AND frozen >= ?` 兜底 |

---

## 七、并发控制与分布式锁

### 7.1 锁实现

```java
// 锁 Key 模板（来自 p9-core-lib/RedisLockKey.java）
public static String ADD_COIN_LOCK_KEY = "add.coin.lock.key.%s";
// 实际 Key 示例："add.coin.lock.key.user_12345"

// 使用方式（Redisson RLock）
RLock rLock = redisUtil.getLock(RedisLockKey.getAddCoinLockKey(userId));
if (rLock.tryLock(20000, 30000L, TimeUnit.MILLISECONDS)) {
//                 ↑等待获锁20s  ↑持有锁30s
    try {
        // 业务操作
    } finally {
        if (rLock.isLocked() && rLock.isHeldByCurrentThread()) {
            rLock.unlock();
        }
    }
}
```

### 7.2 锁使用点全景

| 调用位置 | 操作 | 加锁 | 粒度 |
|---------|------|------|------|
| WalletController.addCoin | 单笔账变 | ✅ | userId |
| WalletController.addCoinList | 批量账变 | ✅ | ⚠️ list[0].userId |
| WalletController.freezeCoin | 冻结 | ✅ | userId |
| WalletController.subFreezeBalance | 解冻扣减 | ✅ | userId |
| WalletController.rollbackFreezeBalance | 冻结回滚 | ✅ | userId |
| AddCoinListener.NoticeAddCoinMessage | MQ单笔 | ✅ | userId |
| AddCoinListener.noticeDeductCoinOrderMessage | MQ批量扣款 | ⚠️ 否 | — |
| AddCoinListener.addCoinNotNoticeBalance | MQ批量扣款 | ⚠️ 否 | — |
| AddCoinListener.addCoinRecordAndNoticeBalanceQueue | MQ记录 | ⚠️ 否 | — |

### 7.3 问题分析

| 问题 | 表现 | 风险 |
|------|------|------|
| **锁在 Controller 层** | Service 层可被无锁调用 | 其他服务直调 Service 则无保护 |
| **批量只锁第一个用户** | `addCoinList` 取 list[0] 的 userId | 多用户批量有并发漏洞 |
| **MQ 消费部分无锁** | 3/4 个 MQ 监听方法未加锁 | MQ 与 HTTP 并发可导致数据不一致 |
| **无 DB 层兜底** | 完全依赖 Redis 锁 | Redis 故障 → 并发失控 |

### 7.4 我们的策略对比

| 维度 | Java 参考 | 我们 ser-wallet |
|------|----------|----------------|
| 锁工具 | Redisson RLock | go-redis SetNX + Lua |
| 锁粒度 | userId | userId + walletType（更细） |
| 锁位置 | Controller 层 | Service 层（所有入口统一保护） |
| DB 兜底 | 无 | SELECT FOR UPDATE |
| 幂等 | 无 | Redis SetNX(biz_no) + DB Unique |

---

## 八、消息队列驱动模型

### 8.1 四个队列全景

`AddCoinListener.java` 监听 4 个 RabbitMQ 队列：

| 队列名 | 方法 | 用途 | 有锁 | 通知余额 |
|--------|------|------|------|---------|
| wallet_add_coin_queue | NoticeAddCoinMessage | 单笔加减钱 | ✅ | ✅ |
| wallet_add_coin_order_queue | noticeDeductCoinOrderMessage | 批量下注扣款 | ❌ | ✅ |
| wallet_add_coin_not_notice_balance_queue | addCoinNotNoticeBalance | 批量扣款（不通知） | ❌ | ❌ |
| add_coin_record_and_notice_balance_queue | addCoinRecordAndNoticeBalanceQueue | 仅记录+通知 | ❌ | ✅ |

### 8.2 消息流转

```
┌───────────────────┐
│ 游戏服务(18+)      │  投注/结算/取消/打赏
│ baccarat/bull/... │
└────────┬──────────┘
         │ 发送 MQ 消息
         ▼
┌──────────────────────────┐
│ RabbitMQ (4 个队列)        │
└────────┬─────────────────┘
         │ 消费
         ▼
┌──────────────────────────┐
│ AddCoinListener           │
│ → CoinRecordService       │
│   → addCoin() / deductCoin│
│   → user_coin 余额变动     │
│   → coin_record 流水写入   │
└────────┬─────────────────┘
         │ 发送通知
         ▼
┌──────────────────────────┐
│ RabbitMQ                  │
│ exchange_notice_user_     │
│ balance_change (fanout)   │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ websocket-service         │
│ → 推送实时余额给前端       │
└──────────────────────────┘
```

### 8.3 异常处理

```java
// 下注扣款失败 → 回滚订单状态
catch (Exception e) {
    List<String> orderNos = coinRecordVOList.stream()
        .map(CoinRecordVO::getOrderNo).collect(Collectors.toList());
    orderResource.updateOrdersToInitialStatus(orderNos);  // ★ 补偿逻辑
}
```

**借鉴点**：扣款失败后主动通知上游服务回滚。我们需要设计类似的补偿策略。

**⚠️ 缺陷**：无死信队列、无重试次数限制、消费失败只 log 不重试、无幂等去重。

---

## 九、币种与汇率管理体系

### 9.1 架构归属

**Java 工程中币种由 merchant-service 管理**，wallet-service 只是消费者。

### 9.2 两张表

**currency 表**（币种基础信息）：

| 字段 | 类型 | 说明 |
|------|------|------|
| currency_code | VARCHAR | 货币代码（USD/CNY/TWD/INR/BRL） |
| currency_name | VARCHAR | 货币名称 |
| symbol | VARCHAR | 符号（$ / ¥ / NT$ / ₹ / R$） |
| proportion | VARCHAR | 比例 |
| status | INT | 1=启用 2=停用 |

**currency_rate 表**（汇率信息）：

| 字段 | 类型 | 说明 |
|------|------|------|
| currency_code | VARCHAR | 货币代码 |
| currency_name | VARCHAR | 货币名称 |
| currency_value | DECIMAL | **汇率值**（相对于基准货币） |
| symbol | VARCHAR | 符号 |
| status | INT | 1=启用 2=停用 |

### 9.3 汇率缓存策略（三级缓存）

```
CurrencyRateServiceImpl.setRedCurrency():
  │
  ├── 1. 查 DB 全量 currency_rate
  │
  ├── 2. 整体缓存 → Redis Key: "merchant-service::merchantCurrency"（JSON 数组）
  │
  ├── 3. 单币种缓存（两个 Key）：
  │   ├── "system-business-service::findRateByCurrencyCode::currencyCode::{CODE}" → BigDecimal
  │   └── "merchant-service::currencyCode::{CODE}" → JSON 完整对象
  │
  └── 4. 同步操作：
      ├── 刷新 currency 表缓存 → "all-service::currency"
      ├── 删除符号映射缓存 → "websocket-service::CurrencyToSymbol"（让下游重建）
      └── MQ 通知 → INIT_USER_LEVEL（触发会员等级重算）
```

### 9.4 汇率查询路径

```
前端查汇率
  → CurrencyRateController.findByCurrencyCode()
    ├── [Redis 命中] → 直接返回
    └── [Redis miss] → 查 DB → 写 Redis → 返回
```

### 9.5 CurrencyBusiness 共享工具类

`p9-core-lib/CurrencyBusiness.java` 被多个服务共用：

```java
getAllCurrencyRate()    // 读 Redis 全量缓存
getCurrencyRate(code)  // 读 Redis 单币种缓存
toUSDAmount(currency, amount)  // 按汇率转换为 USD
```

### 9.6 对我们的启示

| 维度 | Java 做法 | 我们的差异 |
|------|----------|----------|
| 归属 | merchant-service 管币种 | **我们自己 ser-wallet 管** |
| 汇率更新 | 手动编辑（管理后台） | 手动 + 定时拉取 |
| 缓存层次 | 全量 + 单 code 双缓存 | 可借鉴同思路 |
| 变更通知 | MQ 广播 | NATS Publish |
| 币种范围 | 5 种法币（USD/CNY/TWD/INR/BRL） | 法币 + 加密货币 + 平台币 |
| 硬编码枚举 | `CurrencyEnum` 写死 5 种 | 数据库驱动，不硬编码 |

---

## 十、跨服务调用拓扑

### 10.1 wallet-service 调出

```
wallet-service
  │
  ├──→ user-service (8906)
  │    └── getUserByUserId(userId) → UserInfoVO
  │        用途：每次 addCoin 获取用户完整信息（用户名/商户/代理/币种/钱包类型等）
  │        频率：★ 极高（每次账变必调），Redis 缓存兜底
  │
  ├──→ merchant-service (8902)
  │    └── getMerchantDetail(merchantId) → MerchantDetailVO
  │        用途：获取商户详情（三方 URL 等）
  │        频率：低（仅免转钱包分支），Redis 缓存 1 天
  │
  ├──→ order-service (8908)
  │    ├── findOrderByOrderNo(orderNo) → OrderVO         # 免转钱包查订单
  │    ├── updateOrdersToInitialStatus(orderNos)          # 扣款失败回滚订单
  │    └── saveThirdOrderRecord(record)                   # 保存三方转账记录
  │        频率：低/极低
  │
  └──→ api-service (8910)
       └── amountTransfer(request) → ThirdOrderResponseVO  # 免转钱包三方加减额
           频率：低（仅免转分支）
```

### 10.2 谁调用 wallet-service

```
调用 wallet-service 的服务：
│
├── 18+ 游戏服务 ── 通过 MQ 消息 ──→ wallet_add_coin_queue 等
│   投注扣减、结算返奖、取消退款、打赏
│
├── websocket-service ── 通过 Feign ──→ /webapi/wallet/*
│   ├── getWallet()                    # C端余额查询
│   ├── addCoin()                      # 直接加币（打赏等）
│   ├── userWithdrawApply()            # C端提现申请
│   ├── userCoinRecord/Page/List()     # C端账变记录
│   ├── addWithdrawLimitAmount()       # 提现流水限制
│   ├── createOrder()                  # USDT充值创建订单
│   ├── queryUserRechargingInfo()      # 查询充值中订单
│   └── userWithdrawConfigInfo()       # 提现配置
│
├── common-service（游戏公共库）── 通过 Feign ──→
│   ├── addCoin()                      # 结算/打赏加减钱
│   ├── getWallet()                    # 余额查询
│   └── addWithdrawLimitAmount()       # 投注后增加提现流水
│
├── admin-service（总台后台）── 通过 Feign ──→
│   账变记录查询、用户余额查询、手动加减款
│
└── pay-service（独立部署）── 通过 Feign/回调 ──→
    充值到账通知、提现审批通知
```

### 10.3 重要发现

**websocket-service 的 WalletResource Feign 接口暴露了一些 wallet-service 未见的端点**：

```java
// 这些端点在 wallet-service 当前代码中 未找到 对应 Controller：
@PostMapping("/userWithdraw/userWithdrawApply")     // 提现申请
@PostMapping("/userDeposit/queryUserRechargingInfo") // 充值查询
@PostMapping("/userDeposit/createOrder")             // 充值创建订单
@PostMapping("/userWithdraw/userWithdrawConfigInfo") // 提现配置
@PostMapping("/userOfflineDeposit/userRecharge")     // 线下充值
@PostMapping("/userOfflineDeposit/uploadVoucher")    // 上传凭证
@PostMapping("/userOfflineDeposit/cancelDepositOrder") // 取消充值
```

**结论**：wallet-service 可能还有其他代码（如 `UserWithdrawController`、`UserDepositController`）未包含在本仓库中，或者这些端点实际路由到了 pay-service（端口 8916）。考虑到网关配置中 `/third/pay/**` 路由到 pay-service，这些充值/提现相关端点很可能是 **pay-service 的职责**。

---

## 十一、C 端钱包交互链路

### 11.1 websocket-service 中的钱包操作

`UserWalletServiceImpl.java`（420 行）揭示了 C 端用户的完整钱包交互：

| 方法 | WebSocket 命令 | 调用链 |
|------|---------------|-------|
| userWithdraw | userWithdraw | 查用户→查余额→校验金额→提现申请 |
| userCoinRecord | userCoinRecord | 查账变记录（分页）+ 汇总收入/支出 |
| bankCardList | bankCardList | 查银行卡列表 |
| exchangeRate | exchangeRate | 查提现汇率 |
| rechargeTypeList | rechargeTypeList | 查充值方式列表 |
| rechargeExchangeRate | rechargeExchangeRate | 查充值汇率 |
| usdtRechargeInfo | usdtRechargeInfo | USDT 充值创建订单 |
| userRechargingInfo | userRechargingInfo | 查询正在进行的充值 |
| userWithdrawConfig | userWithdrawConfig | 查提现配置 |
| withdrawTypeList | withdrawTypeList | 查提现方式列表 |

### 11.2 提现流程（从代码还原）

```
用户点"提现"
  │
  ├── 1. 查 user-service 用户信息
  │   └── 试玩用户 → 抛异常
  │   └── USD 用户 + 银行卡 → 抛异常（USD 不支持银行卡提现）
  │
  ├── 2. 查 wallet-service 余额
  │   └── 可用余额 < 提现金额 → 余额不足
  │
  ├── 3. 获取连接信息（IP + 设备类型）
  │   └── 从 Redis 的 WebSocket 连接信息中读取
  │
  └── 4. 调 wallet-service userWithdrawApply
      └── 创建提现订单（进入审核流程）
```

### 11.3 USDT 充值流程（从代码还原）

```
用户点"USDT充值"
  │
  ├── 1. 查是否有进行中的充值订单
  │   └── 有 → 抛异常"有正在进行中的充值订单"
  │
  ├── 2. 校验支付方式 = VIRTUAL_CURRENCY
  │
  ├── 3. 查 system-business-service 获取充值汇率
  │   └── tradeCurrencyAmount = 充值金额 / 汇率
  │
  ├── 4. 获取用户 TRC20 钱包地址
  │   └── userInfoVO.getWalletAddress()（来自 user-service）
  │   └── 地址为空 → 抛异常"获取USDT订单充值地址失败"
  │
  ├── 5. 创建充值订单
  │   └── 调 walletResource.createOrder()
  │   └── 订单号格式："D" + 时间戳 + 随机
  │
  └── 6. 返回 USDT 充值信息
      └── 地址协议: TRC20
      └── 地址: walletAddress
      └── 订单号
      └── 充值金额
```

### 11.4 提现流水限制（WithdrawLimitAmountParamVO）

```java
// 提现流水限制类型：
type=1  // 后台人工加额 → 需要 N 倍流水才能提现
type=2  // 会员充值（三方）→ 需要 N 倍流水
type=3  // 会员充值（USDT）→ 需要 N 倍流水
type=4  // 注单 → 投注额度计入已完成流水
```

**这是反洗钱合规的核心机制**：充值后必须完成一定投注流水才能提现。每次充值记录 `withdrawalLimitAmount`，每次投注扣减已完成流水。提现时校验 已完成流水 >= 要求流水。

**我们的需求中也有类似概念**（提现打码量），可以参考此设计。

---

## 十二、枚举与 VO 体系

### 12.1 CoinChangeEnum — 账变大类（9 种）

| code | 名称 | 中文 | 方向 | 余额校验 |
|------|------|------|------|---------|
| 1 | TRANSFER_IN | 转入 | 收入 | 无（加钱） |
| 2 | TRANSFER_OUT | 转出 | 支出 | ★ 需校验 |
| 3 | BET | 下注 | 支出 | ★ 需校验 |
| 4 | SETTLEMENT | 派彩 | 收入 | 无（加钱） |
| 5 | ORDER_CANCEL | 注单取消 | 收入 | 无（退钱） |
| 6 | SETTLEMENT_ROLLBACK | 结算回滚 | 支出 | ⚠️ 跳过（允许负） |
| 7 | SETTLEMENT_SECOND | 二次结算 | 可正可负 | — |
| 8 | SEND_REWARD | 打赏 | 支出 | ★ 需校验 |
| 9 | LIGHTNING_BET | 闪电投注 | 支出 | — |

### 12.2 CoinChangeDetailEnum — 账变明细（8 种）

| code | 名称 | 中文 |
|------|------|------|
| 1 | TRANSFER_IN | 会员存款 |
| 2 | TRANSFER_OUT | 会员提款 |
| 3 | BET | 下注 |
| 4 | SETTLEMENT | 结算 |
| 5 | ORDER_CANCEL | 取消 |
| 6 | SETTLEMENT_CANCEL | 结算取消 |
| 7 | SEND_REWARD | 打赏 |
| 8 | LIGHTNING_BET | 闪电投注 |

### 12.3 WalletTypeEnum — 钱包类型

| code | 名称 | 说明 |
|------|------|------|
| 1 | TRANSFER | 转账钱包（本地管理余额） |
| 2 | FREE_TRANSFER | 免转钱包（三方管理余额） |

### 12.4 BalanceTypeEnum — 收支方向

| code | 名称 |
|------|------|
| 1 | income（收入） |
| 2 | expenses（支出） |

### 12.5 ThirdTransferType — 三方转账类型

| code | 名称 | 说明 |
|------|------|------|
| 1 | BET | 投注扣款 |
| 3 | NORMAL_SETTLEMENT | 正常结算 |
| 5 | SKIP_SETTLEMENT | 跳局结算 |
| 6 | CANCEL_ROUND | 取消局 |
| 7 | RE_CALCULATE_ROUND | 重算局 |
| 8 | REWARD | 打赏 |
| 9 | OPERATE_ROLLBACK | 操作回滚 |

### 12.6 核心 VO 清单（p9-core-lib 共享 13 个钱包 VO + 10 个支付 VO）

**钱包 VO**：

| VO | 用途 | 关键字段 |
|----|------|---------|
| CoinRecordVO | 账变操作入参+出参 | userId, orderNo, coinType, coinDetail, coinValue, balanceType, coinFrom, coinTo |
| UserCoinVO | 余额查询出参 | userId, amount, walletType, currency, md5Sign |
| CoinFreezeVO | 冻结/回滚入参 | userId, freezeAmount |
| CoinFreezeRecordVO | 确认扣减入参 | userId, freezeAmount, orderNo, coinType, coinDetail, balanceType |
| CoinRecordPageParamVO | B端账变分页查询 | 多维度过滤（用户/商户/代理/时间/金额/类型） |
| CoinRecordPageVO | B端账变分页结果 | 完整字段 + 类型名称翻译 |
| UserCoinRecordParamVo | C端账变查询 | userId, coinType, timeRange |
| UserCoinRecordPageVO | C端账变结果 | 含 incomeAmount / expensesAmount |
| QueryWalletReqVO | 批量余额查询 | userIdList |
| OrderMessageVO | MQ 消息 | messageId, orderIdList, status, tryCount |
| UserWithDrawParamVo | 提现申请 | userId, amount, walletType(1=银行卡/2=虚拟货币), currency |
| WithdrawLimitAmountParamVO | 提现流水限制 | userId, type(1~4), withdrawalLimitAmount |
| TrxUserWalletRequest | TRX 热钱包生成 | userId, walletAddress |

**支付 VO（10 个）**：PayWaySettingVO/BO、PayPlatformPageVO/BO、PayPayCodePageVO/BO、PayTypeBO、BankInfoVO 等 — 均为 pay-service 相关，我们暂不涉及。

---

## 十三、定时任务

### 13.1 BackWaterDayTask — 数据清理

```java
// 由 XXL-Job 触发：DeleteTempController.deleteTempTask(days)
public void deleteMessage(int days) {
    deleteAddCoinMessage(days);    // ⚠️ 被注释掉了，实际不执行
    deleteSystemCoinRecord(-5);    // 删除 5 天前 system 商户的试玩会员账变
}
```

**deleteSystemCoinRecord** 只删除 `merchantNo='system'` 且 `userType=1(试玩)` 的账变记录，保留正式用户的所有数据。

---

## 十四、设计优劣评估

### 14.1 值得借鉴的设计

| # | 设计点 | 具体表现 | 借鉴程度 |
|---|--------|---------|---------|
| 1 | **SQL 原子操作** | `amount = amount ± value` 在 SQL 层实现，避免读-改-写竞态 | ★★★★★ 必须采纳 |
| 2 | **冻结三操作分离** | freeze / subFreeze / rollback 职责清晰 | ★★★★★ 已在我们设计中体现 |
| 3 | **前后余额快照** | coin_from / coin_to 记录变动前后余额 | ★★★★★ 审计追踪必需 |
| 4 | **SQL 层非负兜底** | `AND (amount - value) >= 0` 防超扣 | ★★★★☆ |
| 5 | **特殊操作跳过余额检查** | 结算回滚(6)/取消(5) 允许余额变负 | ★★★★☆ 需评估我们的场景 |
| 6 | **惰性钱包创建** | 首次账变时创建，非注册时创建 | ★★★★☆ |
| 7 | **用户信息 Redis 缓存** | 高频调用走缓存，来源服务管理过期 | ★★★★☆ |
| 8 | **币种缓存双级** | 全量 + 单 code 双缓存 | ★★★★☆ |
| 9 | **提现流水限制** | 反洗钱合规的流水打码量机制 | ★★★☆☆ 需求可能涉及 |
| 10 | **扣款失败补偿** | MQ 消费失败后通知 order-service 回滚 | ★★★☆☆ |

### 14.2 存在的问题/缺陷

| # | 问题 | 具体表现 | 严重性 |
|---|------|---------|-------|
| 1 | **无幂等机制** | coin_record 的 order_no 无唯一索引；MQ 消费无去重 | 🔴 高 |
| 2 | **锁在 Controller 层** | Service 层裸奔；MQ 消费 3/4 方法未加锁 | 🔴 高 |
| 3 | **批量锁缺陷** | addCoinList 只锁第一个用户 | 🟡 中 |
| 4 | **无 DB 层并发兜底** | 无 version 字段，无 SELECT FOR UPDATE | 🟡 中 |
| 5 | **冻结回滚无校验** | rollbackFreezeBalance 不检查 freeze >= amount | 🟡 中 |
| 6 | **重度冗余** | 每条 coin_record 冗余 10+ 个字段 | 🟡 中 |
| 7 | **MQ 无死信/重试** | 消费异常只 log 不重试 | 🟡 中 |
| 8 | **SQL 拼接风险** | CoinRecordMapper.xml 用 `${vo.createdTimeBy}` 拼接排序字段 | 🟡 中 |
| 9 | **addCoin 方法过长** | 170+ 行，双钱包类型 + 收支双向全混一起 | 🟢 低（可读性） |
| 10 | **pay-service 缺失** | 充值/提现的支付渠道对接不在本仓库 | 🟢 信息不完整 |

### 14.3 总体评价

> Java 参考工程是一个 **"业务模型基本正确、工程质量有明显短板"** 的实现。
> 核心余额操作（SQL 原子增减 + 冻结三操作）设计合理，经过了生产验证。
> 但在并发安全、消息可靠性、幂等保障方面存在系统性缺陷。
> **适合借鉴其"业务模型和操作语义"，但必须在"工程质量维度"做全面增强。**

---

## 十五、可借鉴设计提炼 + Go 平移映射

### 15.1 核心设计思想提取（与技术栈无关）

| # | 设计思想 | Java 原实现 | Go 转化方式 |
|---|---------|-----------|------------|
| 1 | SQL 原子增减 | MyBatis XML `amount = amount ± value` | GORM `.Update("available", gorm.Expr("available + ?", amt))` |
| 2 | 冻结三操作 | freeze / subFreeze / rollback | Credit / Debit / Freeze / Unfreeze / DeductFrozen（5 方法） |
| 3 | 前后快照 | coinFrom / coinTo | balance_before / balance_after |
| 4 | 双层余额校验 | Java 层预检 + SQL WHERE 兜底 | Service 层预检 + SQL `AND available >= ?` |
| 5 | 余额变更事件广播 | MQ exchange_notice_user_balance_change | NATS Publish("wallet.balance.changed") |
| 6 | 用户信息缓存 | Redis 缓存 + Feign 回源 | Redis 缓存 + Kitex RPC 回源 |
| 7 | 币种双级缓存 | 全量 key + 单 code key | 同思路 |
| 8 | 扣款失败补偿 | catch → orderResource.rollback | catch → 通知上游回滚 |

### 15.2 反面教训（我们要避免的）

| # | Java 的问题 | 我们的规避方案 |
|---|-----------|-------------|
| 1 | 无幂等 | Redis SetNX(biz_no, 24h) + DB Unique Index(biz_no) |
| 2 | 锁在 Controller | 锁在 Service 层，所有入口统一保护 |
| 3 | 批量只锁单用户 | 按用户分组，每组独立加锁 |
| 4 | 纯 Redis 锁无 DB 兜底 | Redis + SELECT FOR UPDATE 双重 |
| 5 | rollback 无校验 | SQL 加 `AND frozen >= ?` |
| 6 | 账变重度冗余 | 轻冗余（只存 user_id + wallet_id） |
| 7 | MQ 无死信/重试 | JetStream MaxDeliver + 死信主题 |
| 8 | SQL 拼接 | GORM 参数化查询 |

### 15.3 关键路径对照表

| 业务场景 | Java 调用链 | Go 调用链 |
|---------|-----------|----------|
| 下注扣款 | MQ → AddCoinListener → addCoin(BET, expenses) | NATS → subscriber → ser.Debit(BET) |
| 结算派彩 | MQ → AddCoinListener → addCoin(SETTLEMENT, income) | NATS → subscriber → ser.Credit(SETTLEMENT) |
| 充值到账 | MQ → addCoin(TRANSFER_IN, income) | RPC from ser-finance → ser.Credit(DEPOSIT) |
| 提现冻结 | HTTP → freezeCoin | RPC → ser.Freeze |
| 提现成功 | HTTP → subFreezeBalance + createOrder | RPC → ser.DeductFrozen |
| 提现失败 | HTTP → rollbackFreezeBalance | RPC → ser.Unfreeze |
| 余额查询 | HTTP → getWallet → amount - freeze | RPC → ser.GetBalance → 直接读 available |
| B端账变查询 | HTTP → CoinRecordController | RPC → ser.GetTransactionList |

### 15.4 技术组件映射表

| Java 组件 | 用途 | Go 对应 |
|-----------|------|---------|
| Spring Boot | 框架 | Kitex + Hertz |
| @Transactional | 事务 | GORM db.Transaction(func(tx) error {}) |
| Redisson RLock | 分布式锁 | go-redis SetNX + Lua 脚本 |
| RabbitMQ | 消息队列 | NATS/JetStream |
| Feign Client | HTTP RPC | Kitex Client（Thrift） |
| MyBatis XML | 自定义 SQL | GORM Expr / Raw SQL |
| BeanUtils.copyProperties | 对象拷贝 | 手动赋值 |
| Redis 缓存 | 数据缓存 | go-redis 缓存 |
| XXL-Job | 定时任务 | robfig/cron |

---

## 附录 A: wallet-service 完整文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| WalletServiceRunner.java | ~15 | 启动类 |
| WalletController.java | ~260 | HTTP 入口：9 端点 + 分布式锁 |
| CoinRecordController.java | ~100 | 账变查询：7 端点 |
| DeleteTempController.java | ~20 | 定时清理入口 |
| CoinRecordServiceImpl.java | **~522** | **核心**：addCoin / deductCoin / freeze / subFreeze / rollback |
| CoinRecordDataServiceImpl.java | ~182 | 账变分页/列表查询 + 枚举名称翻译 |
| UserCoinServiceImpl.java | ~87 | 余额查询（TRANSFER / FREE_TRANSFER 分叉） |
| CommonServiceImpl.java | ~74 | 用户/商户信息获取 + Redis 缓存 |
| AddCoinMessageServiceImpl.java | ~21 | MQ 消息状态更新 |
| UserCoinDTO.java | ~32 | user_coin 表映射 |
| CoinRecordDTO.java | ~59 | coin_record 表映射 |
| OrderMessageDTO.java | — | add_coin_message 表映射 |
| BalanceVO.java | ~16 | 余额操作入参（id + amount + coinType） |
| UserCoinRepository.java | — | BaseMapper + 5 自定义 SQL |
| CoinRecordRepository.java | — | BaseMapper + 4 自定义查询 |
| AddCoinMessageRepository.java | — | 消息幂等表 |
| AddCoinListener.java | ~147 | 4 个 MQ 消费者 |
| RabbitSender.java | ~30 | 订单 MQ 消息发送 |
| BackWaterDayTask.java | ~57 | 数据清理（试玩用户账变 + MQ 消息） |
| OrderResource.java | ~31 | Feign → order-service |
| ApiResource.java | ~17 | Feign → api-service |
| UserService.java | ~15 | Feign → user-service |
| MerchantFeignResource.java | ~18 | Feign → merchant-service |
| WallRabbitTemplateConfig.java | — | RabbitMQ 模板配置 |
| UserCoinMapper.xml | **44** | **5 条原子 SQL** ← 最关键 |
| CoinRecordMapper.xml | 284 | 4 条复杂查询 |
| AddCoinMessageMapper.xml | — | 消息状态更新 SQL |

---

## 附录 B: Feign 依赖 + Redis Key + MQ 队列速查

### Feign 依赖

| Feign 接口 | 目标服务 | 端口 | 方法 |
|-----------|---------|------|------|
| OrderResource | order-service | 8908 | findOrderByOrderNo / updateOrdersToInitialStatus / saveThirdOrderRecord |
| ApiResource | api-service | 8910 | amountTransfer |
| UserService | user-service | 8906 | getUserByUserId |
| MerchantFeignResource | merchant-service | 8902 | getMerchantDetail |

### Redis Key

| Key 模板 | 用途 |
|---------|------|
| `add.coin.lock.key.{userId}` | 钱包操作分布式锁 |
| `user-service::userInfo::userId::{userId}` | 用户信息缓存 |
| `merchant-service::merchantCurrency` | 全量汇率缓存 |
| `merchant-service::currencyCode::{code}` | 单币种汇率对象 |
| `system-business-service::findRateByCurrencyCode::currencyCode::{code}` | 单币种汇率值 |
| `all-service::currency` | 全量币种列表 |
| `websocket-service::CurrencyToSymbol` | 币种→符号映射 |

### MQ 队列

| 队列 | 路由 | 生产者 | 用途 |
|------|------|--------|------|
| wallet_add_coin_queue | wallet_add_coin_key | 游戏服务 | 单笔加减钱 |
| wallet_add_coin_order_queue | — | order-service | 批量下注扣款 |
| wallet_add_coin_not_notice_balance_queue | — | 游戏服务 | 静默扣款 |
| add_coin_record_and_notice_balance_queue | — | 游戏服务 | 仅记录+通知 |
| exchange_notice_user_balance_change | (fanout) | wallet-service | 余额变更通知 |
| notice_user_balance_change | — | wallet-service | 免转余额变更通知 |

---

## 总结：三句话

1. **SQL 原子操作 + 冻结三操作 + 前后快照** 是最核心的可借鉴设计 — 已被生产验证，与技术栈无关，我们直接采纳其思想
2. Java 工程**缺失幂等、MQ 可靠性、DB 层并发兜底**等企业级特性 — 这恰好是我们在 Go 设计中已规划增强的部分
3. **pay-service（充值/提现渠道对接）是独立部署的**，本仓库只包含余额操作核心 — 验证了我们"ser-wallet 管余额，ser-finance 管支付渠道"的职责划分方向
