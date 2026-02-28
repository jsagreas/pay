# Java 参考工程钱包模块 — 深度分析报告

> 分析时间：2026-02-27
> 分析对象：`/Users/mac/gitlab/z-tmp/backend` — 生产运行中的 Java 真人视讯平台
> 核心分析模块：`p9-services/wallet-service`（29 Java + 5 资源文件）
> 关联分析模块：`p9-core-lib`（共享枚举/VO）、`p9-services/merchant-service`（币种/汇率）
> 分析方法：全部源码逐文件阅读 → 数据模型提取 → 业务流程逆向还原 → 设计模式归纳 → 与 Go 设计对比映射
> 分析目的：提炼可借鉴的设计思想和实现模式，为 Go ser-wallet 实现提供参考（借鉴不照搬）
> 关联文档：
> - 我方架构设计：/Users/mac/gitlab/z-readme/架构设计/tmp-1.md
> - 我方表结构设计：/Users/mac/gitlab/z-readme/表结构设计/tmp-1.md
> - 我方钱包工程结构：/Users/mac/gitlab/z-readme/钱包结构/tmp-1.md

---

## 目录

1. [参考工程技术栈总览](#一参考工程技术栈总览)
2. [wallet-service 工程结构](#二wallet-service-工程结构)
3. [数据库表结构深度分析（核心3+关联2）](#三数据库表结构深度分析)
4. [核心业务逻辑深度拆解](#四核心业务逻辑深度拆解)
5. [并发控制与资金安全机制](#五并发控制与资金安全机制)
6. [冻结机制深度分析](#六冻结机制深度分析)
7. [消息队列驱动模式](#七消息队列驱动模式)
8. [币种与汇率体系](#八币种与汇率体系)
9. [跨服务依赖拓扑](#九跨服务依赖拓扑)
10. [枚举体系与VO/DTO设计](#十枚举体系与vo-dto设计)
11. [定时任务与数据清理](#十一定时任务与数据清理)
12. [设计模式提炼 — 值得借鉴](#十二设计模式提炼值得借鉴)
13. [设计缺陷提炼 — 需要规避](#十三设计缺陷提炼需要规避)
14. [全链路对比总表](#十四全链路对比总表)
15. [可借鉴清单（分三级）](#十五可借鉴清单)
16. [附录](#十六附录)

---

## 一、参考工程技术栈总览

| 维度 | 参考工程（Java） | 我们的工程（Go） | 差异影响 |
|------|-----------------|-----------------|---------|
| 框架 | Spring Cloud + Spring Boot | CloudWeGo Kitex + Hertz | 服务治理模式不同 |
| ORM | MyBatis-Plus（DTO 即 Entity） | GORM Gen（model + query 分离） | Go 分层更清晰 |
| 数据库 | MySQL | TiDB（MySQL 兼容） | 基本一致 |
| 缓存 | Redis（Redisson 分布式锁） | Redis | 锁策略不同 |
| 消息队列 | RabbitMQ（4个队列，重度依赖） | NATS JetStream / Kafka | 架构差异最大 |
| 服务调用 | Feign（HTTP JSON） | Kitex（Thrift RPC） | 性能差异大 |
| 注册中心 | K8S SVC（硬编码 URL） | ETCD | 无本质影响 |
| 共享库 | p9-core-lib（枚举/VO/工具） | common/（consts/utils） | 模式一致 |
| 金额类型 | BigDecimal(20,2) | BIGINT（最小货币单位） | 我方无浮点误差 |

**最关键的架构差异**：参考工程大量使用 MQ 解耦（游戏→钱包→通知），我们以同步 RPC 为主。这影响到几乎所有链路设计 — 参考工程的异步消费者模式需要转化为我方的同步 RPC handler 模式。

---

## 二、wallet-service 工程结构

### 2.1 完整文件清单（34文件 = 29 Java + 5 资源）

```
wallet-service/src/main/java/com/maya/p9/
├── WalletServiceRunner.java              // Spring Boot 启动类
├── controller/
│   ├── WalletController.java             // ★ 核心控制器（9个端点，全部写操作加锁）
│   ├── CoinRecordController.java         // 账变记录查询（7个端点，纯读操作）
│   └── xxljob/
│       └── DeleteTempController.java     // 定时清理入口（XXL-Job触发）
├── dto/
│   ├── UserCoinDTO.java                  // ★ user_coin 表实体（钱包主表）
│   ├── CoinRecordDTO.java                // ★ coin_record 表实体（账变流水）
│   └── OrderMessageDTO.java             // add_coin_message 表实体（MQ消息追踪）
├── service/
│   ├── CoinRecordService.java            // 核心业务接口（11个方法）
│   ├── UserCoinService.java              // 钱包查询接口（2个方法）
│   ├── AddCoinMessageService.java        // MQ消息状态接口（1个方法）
│   ├── CoinRecordDataService.java        // 账变数据查询接口（6个方法，分页/列表/统计）
│   ├── CommonService.java                // 公共服务接口（2个方法：取用户/取商户信息）
│   ├── impl/
│   │   ├── CoinRecordServiceImpl.java    // ★★★ 核心实现（522行，addCoin/freeze 全在这）
│   │   ├── UserCoinServiceImpl.java      // ★ 钱包查询（含转账/免转分支、API安全检查）
│   │   ├── CoinRecordDataServiceImpl.java// 账变查询（分页/列表，枚举名翻译）
│   │   ├── AddCoinMessageServiceImpl.java// MQ消息状态更新（极简，1个方法）
│   │   └── CommonServiceImpl.java        // ★ Redis缓存+Feign兜底（有设计缺陷）
│   └── feign/
│       ├── UserService.java              // → user-service:8906（取用户信息）
│       ├── MerchantFeignResource.java    // → merchant-service:8902（取商户详情）
│       ├── OrderResource.java            // → order-service:8908（查订单/回滚/三方记录）
│       └── ApiResource.java              // → api-service:8910（三方钱包余额操作）
├── rabbitmq/
│   ├── RabbitSender.java                 // MQ发送工具
│   └── listener/
│       └── AddCoinListener.java          // ★ 4个MQ消费者（下注扣款/结算返奖等入口）
├── rabbit/config/
│   └── WallRabbitTemplateConfig.java     // MQ模板配置
├── repository/
│   ├── UserCoinRepository.java           // MyBatis Mapper（含5个自定义原子SQL）
│   ├── CoinRecordRepository.java         // MyBatis Mapper（含分页/统计/最近提现查询）
│   └── AddCoinMessageRepository.java     // MyBatis Mapper（MQ消息状态更新）
├── task/
│   └── BackWaterDayTask.java             // 定时清理：试玩用户账变 + 过期MQ消息
└── vo/
    └── BalanceVO.java                    // 余额操作内部传参（id+amount+updateTime+coinType）

wallet-service/src/main/resources/
├── bootstrap.yml                         // 服务配置
├── logback.xml                           // 日志配置
└── mapper/
    ├── UserCoinMapper.xml                // ★★★ 5条核心余额SQL（精华中的精华）
    ├── CoinRecordMapper.xml              // 账变分页/列表/统计/最近提现 查询SQL
    └── AddCoinMessageMapper.xml          // MQ消息状态更新SQL
```

**关键数字**：核心代码不超过 **1200行**（不含查询/分页），非常精简。

### 2.2 架构分层

```
┌──────────────────────────────────────────────────────────────┐
│  Controller 层（3个控制器）                                    │
│  · WalletController — 加减钱 + 冻结 + 查余额（写操作全加锁）   │
│  · CoinRecordController — 账变记录分页/列表/统计/转账结果查询   │
│  · DeleteTempController — 定时清理入口                        │
│  职责：参数校验 → 获取分布式锁 → 调Service → 释放锁           │
├──────────────────────────────────────────────────────────────┤
│  Service 层（5个Service）                                     │
│  · CoinRecordServiceImpl ★★★ — 核心业务（addCoin/freeze/...) │
│  · UserCoinServiceImpl — 查余额（转账/免转分支）               │
│  · CoinRecordDataServiceImpl — 账变查询（枚举名翻译）          │
│  · CommonServiceImpl — 用户/商户信息获取（Redis缓存+Feign兜底）│
│  · AddCoinMessageServiceImpl — MQ消息状态                     │
│  职责：业务逻辑 + 余额校验 + 钱包类型分支 + 账变记录写入       │
├──────────────────────────────────────────────────────────────┤
│  Repository 层（MyBatis Mapper + XML）                        │
│  · UserCoinRepository — 5条原子SQL操作（加/减/冻结/解冻/回滚） │
│  · CoinRecordRepository — 分页查询 + 插入                     │
│  · AddCoinMessageRepository — 消息状态更新                    │
│  职责：纯数据库操作，SQL在XML中定义                            │
├──────────────────────────────────────────────────────────────┤
│  MQ Listener 层（1个监听器，4个队列）                          │
│  · AddCoinListener — 单条加减/批量下注扣减/静默扣减/记录+通知  │
│  职责：消息反序列化 → 获取分布式锁 → 调Service → 失败补偿      │
├──────────────────────────────────────────────────────────────┤
│  Feign 层（4个远程调用客户端）                                 │
│  · user-service / merchant-service / order-service / api-svc │
│  职责：获取用户信息、商户信息、订单信息、调三方API             │
└──────────────────────────────────────────────────────────────┘
```

**与我方设计的差异**：
- 参考工程是标准三层（Controller → Service → Repository），没有独立的 Ledger 层
- 我方设计了独立的记账核心层（handler → service → repository），通过 service 层封装 5 个原子操作 + 事务 + 幂等
- 参考工程的锁在 Controller 层，业务在 Service 层 — 锁和事务分离两层

### 2.3 一个重要边界发现

**wallet-service 只管「余额核心操作」，不管充值/提现/兑换流程**：

```
wallet-service 负责什么：
✅ 余额增减（addCoin — 加减钱总入口）
✅ 冻结/解冻（freezeCoin / subFreezeBalance / rollbackFreezeBalance）
✅ 余额查询（getWallet / getWalletListByUserId）
✅ 账变记录（queryCoinRecord / 分页/列表/统计）
✅ 余额变化通知（sendUserBalance → MQ fanout → WebSocket）

wallet-service 不负责什么：
❌ 充值流程（不在 wallet-service 代码中，可能在其他版本或独立服务）
❌ 提现流程（WebSocket Feign 中有 userWithdrawApply 指向 8907，但当前代码不含此控制器）
❌ 兑换流程（完全没有）
❌ 币种配置（归 merchant-service 管理）
```

对比我方 ser-wallet 职责范围更大（包含充值/提现/兑换/币种配置），参考工程的 wallet-service 更专注于纯粹的"账户余额操作"。

---

## 三、数据库表结构深度分析

### 3.1 总览：核心3张 + 关联2张

```
wallet-service 持有（3张）：
├── user_coin       — 用户钱包主表（余额权威来源）
├── coin_record     — 账变流水表（审计追踪）
└── add_coin_message — MQ消息幂等表（MQ场景专用）

merchant-service 持有（2张，被 wallet 间接依赖）：
├── currency        — 币种基础信息（代码/名称/符号/状态）
└── currency_rate   — 汇率配置（基于USD基准）
```

### 3.2 user_coin — 用户钱包主表

从 `UserCoinDTO.java` 反推的表结构：

```sql
CREATE TABLE user_coin (
    id              BIGINT       PRIMARY KEY AUTO_INCREMENT,
    user_id         VARCHAR(64)  NOT NULL,           -- 用户ID
    user_name       VARCHAR(128),                     -- 用户名称（冗余）
    amount          DECIMAL(20,2) NOT NULL DEFAULT 0, -- ★ 账户总金额（含冻结）
    freeze_amount   DECIMAL(20,2) NOT NULL DEFAULT 0, -- ★ 冻结金额
    user_state      INT,                              -- 游戏状态（不该放钱包表）
    game_type       INT,                              -- 正在进行的游戏（不该放钱包表）
    wallet_type     INT NOT NULL DEFAULT 1,           -- ★ 1=转账钱包 2=免转钱包
    currency        VARCHAR(16),                      -- 币种代码
    remark          VARCHAR(512),
    creator         BIGINT,
    updater         BIGINT,
    created_time    DATETIME,
    updated_time    DATETIME
);
-- 核心等式：可用余额 = amount - freeze_amount（查询时计算，不单独存储）
-- 一个用户一条记录（单钱包 × 单币种）
-- 无 version 字段 — 不使用乐观锁
```

**核心设计观察**：

| 观察点 | 参考工程 | 我方设计 | 分析 |
|--------|---------|---------|------|
| 用户×钱包关系 | 1用户=1行 | 1用户 × N币种 × 5子钱包 = N行 | 参考工程单钱包单币种，简单但扩展性差 |
| 金额精度 | BigDecimal(20,2) | BIGINT(最小单位) | 我方无浮点误差风险 |
| 可用余额 | 运行时算 amount-freeze | 同样不单独存字段 | 思路一致 |
| wallet_type | 转账(1)/免转(2)分支 | 无此概念（自建平台，全部自管） | 免转钱包≈三方游戏代管余额，我方暂不需要 |
| user_state/game_type | 存在钱包表 | 不放钱包表 | 参考工程设计耦合，不可取 |
| version字段 | **无** | 有（CAS乐观锁） | 我方更安全 |

**getWallet 查询的关键细节**（来自 `UserCoinServiceImpl.java`）：

```java
// getWallet 返回的 amount 是"可用余额"，不是"总额"
if (userCoin.getFreezeAmount() != null) {
    userCoinVO.setAmount(userCoin.getAmount().subtract(userCoin.getFreezeAmount()));
}

// API 安全检查：非正式用户从 API 入口查询，强制返回 0
if (ObjectUtil.isNotEmpty(userCoinVO.getMd5Sign())) {  // 签名非空=API来源
    if (!UserTypeEnum.FORMAL_PLAY.getCode().equals(userInfoVO.getUserType())) {
        userCoinVO.setAmount(BigDecimal.ZERO);  // 防止试玩用户余额被包网利用
    }
}
```

### 3.3 coin_record — 账变流水表

从 `CoinRecordDTO.java` 反推：

```sql
CREATE TABLE coin_record (
    id              BIGINT       PRIMARY KEY AUTO_INCREMENT,
    -- ★ 用户身份快照（7字段，每条记录冗余完整用户信息）
    user_id         VARCHAR(64)  NOT NULL,
    user_name       VARCHAR(128),
    user_type       INT,              -- 0=正式 1=试玩
    nick_name       VARCHAR(128),
    user_belong     INT,              -- 用户所属
    -- ★ 商户/代理快照（6字段）
    merchant_id     BIGINT,
    merchant_name   VARCHAR(128),
    merchant_no     VARCHAR(64),
    agent_id        BIGINT,
    agent_no        VARCHAR(64),
    agent_name      VARCHAR(128),
    path            VARCHAR(512),     -- 代理层级链路
    -- ★ 账变核心字段
    order_no        VARCHAR(128) NOT NULL,   -- 关联订单号（无唯一索引！）
    coin_type       INT NOT NULL,            -- 账变类型（9种枚举）
    coin_detail     INT,                     -- 账变明细（8种枚举）
    coin_value      DECIMAL(20,2) NOT NULL,  -- 变动金额（正数）
    coin_from       DECIMAL(20,2),           -- 变动前余额快照
    coin_to         DECIMAL(20,2),           -- 变动后余额快照
    coin_amount     DECIMAL(20,2),           -- 当前金额（≈coin_to，冗余）
    balance_type    INT NOT NULL,            -- 1=收入 2=支出
    remark          VARCHAR(512),
    creator         BIGINT,
    updater         BIGINT,
    created_time    DATETIME,
    updated_time    DATETIME
);
```

**核心设计观察**：

| 观察点 | 参考工程 | 我方设计 | 分析 |
|--------|---------|---------|------|
| 幂等保障 | **无** order_no 非唯一索引 | flow表 unique(order_no+flow_type) | 参考工程缺失幂等 ⚠️ |
| 信息冗余 | 13个用户/商户/代理字段冗余 | 精简冗余，按需JOIN | 参考工程冗余过重 |
| 余额快照 | coin_from/coin_to | before_balance/after_balance | ★ 审计追踪核心，必须借鉴 |
| 金额方向 | coin_value 正数 + balance_type | amount 正数 + direction | 思路一致 |
| coin_type 枚举 | 9种（见枚举体系章节） | 按需求定义 flow_type | 参考枚举值 |
| coin_detail | 二级分类（8种） | 合并为一层或扩展remark | 两层枚举过度设计 |
| coin_amount | ≈coin_to 冗余字段 | 不设此冗余字段 | 参考工程多余 |

**重度冗余的设计思想分析**：

```
为什么每条流水冗余13个用户/商户/代理字段？

原因1（查询解耦）：B端后台查账变时，不需要联表user/merchant/agent三张表
原因2（历史快照）：用户改名/换代理后，账变记录保留"发生时"的快照
原因3（数据归属）：merchant_id 直接在本表，方便商户维度过滤和统计

实际代价：每条记录多占 ~200-300 字节
日增100万条约多占 200-300 MB/天

我方评估：
- merchant_id 值得冗余（后台按商户筛选的刚需）
- user_name / nick_name 可不冗余（查询时JOIN即可）
- agent链路(path字段) 看是否有按代理树筛选需求再定
```

### 3.4 add_coin_message — MQ消息幂等表

```sql
CREATE TABLE add_coin_message (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    message_id      VARCHAR(128) NOT NULL,   -- MQ消息ID
    order_id_list   TEXT,                    -- 关联订单列表（JSON）
    status          INT DEFAULT 1,           -- 1=成功 2=失败
    try_count       INT DEFAULT 0,           -- 重试次数
    create_time     DATETIME,
    update_time     DATETIME
);
```

此表专为 MQ 消费幂等设计。**实际代码中未看到使用它做去重**（只有 status 更新方法被调用），存在"设计了但没用好"的问题。我方使用 RPC 不使用 MQ，此表设计不适用，但"幂等追踪"思想可借鉴到流水表唯一索引。

### 3.5 currency — 币种主数据表（merchant-service 持有）

从 `CurrencyDTO.java` 反推：

```sql
CREATE TABLE currency (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    currency_code   VARCHAR(16) NOT NULL,    -- 币种代码（USD/CNY/TWD/INR/BRL）
    currency_name   VARCHAR(64),             -- 币种名称
    symbol          VARCHAR(8),               -- 符号（$ CN¥ NT$ ₹ R$）
    name            VARCHAR(64),              -- 显示名称
    proportion      VARCHAR(32),              -- 比例（含义不明，可能精度相关）
    status          INT DEFAULT 1,            -- 1=启用 2=停用
    creator         BIGINT,
    updater         BIGINT,
    created_time    DATETIME,
    updated_time    DATETIME
);
```

### 3.6 currency_rate — 汇率配置表（merchant-service 持有）

从 `CurrencyRateDTO.java` 反推：

```sql
CREATE TABLE currency_rate (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    currency_code   VARCHAR(16) NOT NULL,    -- 币种代码
    currency_name   VARCHAR(64),             -- 币种名称
    currency_value  DECIMAL(20,4) NOT NULL,  -- ★ 汇率（1 USD = X 目标币种）
    symbol          VARCHAR(8),               -- 符号
    status          INT DEFAULT 1,            -- 1=启用 2=停用
    creator         BIGINT,
    updater         BIGINT,
    created_time    DATETIME,
    updated_time    DATETIME
);
-- 汇率语义：以 USD 为基准
-- 例：CNY 的 currency_value = 7.2 → 1 USD = 7.2 CNY
```

**币种/汇率 vs 我方设计对比**：

| 维度 | 参考工程 | 我方设计 |
|------|---------|---------|
| 归属 | merchant-service 管理 | ser-wallet 内的币种配置子模块 |
| 表数量 | 2张（currency + currency_rate） | 2张（t_currency + t_rate_log） |
| 基准货币 | USD 固定基准 | 待产品确认，USD 是行业惯例 |
| 汇率类型 | 仅1种（全局汇率） | 可能多种（充值/提现/平台汇率） |
| 精度 | DECIMAL(20,4) | 根据币种配置 precision 字段动态 |
| 缓存 | Redis 全量 + 单key 双层 | 类似，Redis全量 + 按code单key |
| 支持币种 | 5种（代码+枚举双重存在） | 数据库驱动，不硬编码 |

---

## 四、核心业务逻辑深度拆解

### 4.1 addCoin — 余额变动核心方法（心脏方法）

> 源码位置：`CoinRecordServiceImpl.addCoin()`（522行中最核心的170行）
> 所有余额变化最终都走这一个方法，通过 coinType 区分业务语义

**调用入口**：
```
两种入口，殊途同归：
1. HTTP API → WalletController.addCoin() → 获取Redisson锁 → service.addCoin()
2. MQ消费  → AddCoinListener.NoticeAddCoinMessage() → 获取Redisson锁 → service.addCoin()
```

**完整执行流程**：

```
addCoin(CoinRecordVO) 执行流程：

Step 1: 参数校验
├── coinType 必须 > 0
└── coinValue 必须 > 0（在Controller层校验，Service层也校验）

Step 2: 获取用户信息（★ 每次 addCoin 都要调）
├── commonService.getUserInfoByUserId(userId)
│   ├── 先查 Redis 缓存 → 命中直接返回
│   └── 未命中 → Feign 调 user-service → 返回（但未回写缓存！⚠️）
├── 获取：userName, nickName, walletType, currency, merchantId/No/Name,
│         userType, agentId/No/Name, path, userBelong
└── setVOToUserInfo() → 填充到 coinRecordDTO（冗余13个字段）

Step 3: ★★★ 按钱包类型分支（这是参考工程最重要的设计分支）

┌──────────────────────────────┬────────────────────────────────────┐
│ 转账钱包 (walletType == 1)   │ 免转钱包 (walletType == 2)          │
│ 余额存本地 user_coin 表      │ 余额由三方游戏平台管理               │
├──────────────────────────────┼────────────────────────────────────┤
│ 3a. 查 user_coin 表          │ 3a. 查订单信息 → gameTypeId         │
│                              │     （打赏固定 gameTypeId=100）      │
│ 3b. 用户存在:                │                                    │
│   · insufficientBalance()    │ 3b. 查商户详情 → addDeductionUrl    │
│   · 收入 → addBalance SQL   │                                    │
│   · 支出 → subBalance SQL   │ 3c. 组装 ThirdOrderRequest          │
│   · 计算 coinFrom/coinTo    │     含 transferNo, totalAmount,     │
│                              │     transferType（按coinType映射）  │
│ 3c. 用户不存在:              │                                    │
│   · 必须是收入操作           │ 3d. 调 api-service.amountTransfer() │
│   · 自动创建 user_coin 行   │     → 三方成功: 记录thirdOrder      │
│     (amount=coinValue,       │     → 三方失败: return false        │
│      freezeAmount=0)         │                                    │
│                              │ 3e. coinFrom/coinTo 设为 null       │
│                              │     （三方管理余额，本地不知道）     │
├──────────────────────────────┼────────────────────────────────────┤
│ 4. 插入 coin_record 流水     │ 4. 插入 coin_record 流水            │
└──────────────────────────────┴────────────────────────────────────┘
```

**关键细节 — insufficientBalance() 余额校验逻辑**：

```java
// 不是所有支出都检查余额！只有三种类型才检查：
if (coinType == TRANSFER_OUT     // 转出（提现下分）
    || coinType == BET           // 下注
    || coinType == SEND_REWARD)  // 打赏
{
    BigDecimal available = amount - freezeAmount;  // 可用余额 = 总额 - 冻结
    if (available < coinValue) throw AMOUNT_NOT_ENOUGH;
}

// 不检查余额的类型及原因：
// SETTLEMENT_ROLLBACK(6) → 结算回滚可以将余额扣到负数（追回错误结算）
// ORDER_CANCEL(5) → 实际是退款（收入），不该走到支出分支
// SETTLEMENT_SECOND(7) → 重算可能反向扣钱，允许负数
// TRANSFER_IN(1) → 充值（收入），不需要检查
// SETTLEMENT(4) → 结算（收入），不需要检查
```

**这里有一个非常重要的业务设计 — 允许负余额**：

```
为什么结算回滚允许余额为负？

场景：
1. 用户下注100，赢了结算200（余额 100→200）
2. 用户提走了200（余额 200→0）
3. 发现结算错误需要回滚（要追回200）
4. 此时余额只有0，但必须扣回200 → 余额变为 -200

如果不允许负数，钱追不回来。
这在竞彩平台是标准做法 — 宁可余额为负也要保证账目正确。

Go 实现中应在对应的 flow_type（如结算回滚）也允许负余额。
```

### 4.2 5条核心余额SQL（精华中的精华）

> 源码位置：`UserCoinMapper.xml` — 只有44行，但每一行都是钱包安全的基石

```xml
<!-- SQL 1: addBalance — 加余额（无条件，直接加）-->
UPDATE user_coin SET amount = amount + #{vo.amount},
    updated_time = #{vo.updateTime}
WHERE id = #{vo.id}

<!-- SQL 2: subBalance — 减余额（★★★ 最关键的SQL）-->
UPDATE user_coin SET amount = amount - #{vo.amount},
    updated_time = #{vo.updateTime}
WHERE id = #{vo.id}
    <!-- coinType 5=取消, 6=结算回滚 不检查非负 → 允许负余额 -->
    <if test="vo.coinType != 6 and vo.coinType != 5">
        AND (amount - #{vo.amount}) >= 0    -- ★ 数据库级非负保障
    </if>

<!-- SQL 3: addFreezeBalance — 增加冻结金额 -->
UPDATE user_coin SET freeze_amount = freeze_amount + #{vo.amount},
    updated_time = #{vo.updateTime}
WHERE id = #{vo.id}

<!-- SQL 4: subFreezeBalance — 确认扣减冻结（amount和freeze同时减） -->
UPDATE user_coin SET amount = amount - #{vo.amount},
    freeze_amount = freeze_amount - #{vo.amount},
    updated_time = #{vo.updateTime}
WHERE id = #{vo.id}

<!-- SQL 5: rollbackFreezeBalance — 回滚冻结（只减freeze，amount不变） -->
UPDATE user_coin SET freeze_amount = freeze_amount - #{vo.amount},
    updated_time = #{vo.updateTime}
WHERE id = #{vo.id}
```

**逐条分析**：

| SQL | 修改 amount | 修改 freeze | 可用余额变化 | 产生账变 | 非负检查 |
|-----|-----------|-----------|------------|---------|---------|
| addBalance | +X | 不变 | +X | 是 | 无 |
| subBalance | -X | 不变 | -X | 是 | 有（特殊类型除外） |
| addFreezeBalance | 不变 | +X | -X | 否 | 无 |
| subFreezeBalance | -X | -X | 不变 | 是 | 无 ⚠️ |
| rollbackFreezeBalance | 不变 | -X | +X | 否 | 无 |

**subBalance 的非负保障是整个系统最关键的安全机制**：

```
设计要点：
1. 原子操作：SET amount = amount - X（不是先查后改）
2. 条件检查：AND (amount - X) >= 0
3. 返回值判断：affected_rows == 0 → 余额不足
4. 选择性跳过：结算回滚/取消 允许负余额

即使两个请求同时通过了应用层的余额检查，
在SQL层面，第二个请求执行时余额已经被第一个扣减了，
(amount - X) >= 0 不再满足 → UPDATE影响0行 → 安全

Go GORM 实现方式：
result := db.Model(&WalletAccount{}).
    Where("id = ? AND (amount - ?) >= 0", id, deductAmount).
    Update("amount", gorm.Expr("amount - ?", deductAmount))
if result.RowsAffected == 0 { return ErrInsufficientBalance }
```

**重要发现 — subFreezeBalance 缺少非负保护**：

```xml
<!-- subFreezeBalance 没有 (amount-X)>=0 条件！-->
SET amount = amount - #{vo.amount}, freeze_amount = freeze_amount - #{vo.amount}
WHERE id = #{vo.id}
```

虽然被 Redisson 锁保护，但如果锁失效：
- amount 可能被扣到负数
- freeze_amount 也可能被扣到负数

**Go 实现应加上保护**：
```sql
UPDATE t_wallet_account
SET amount = amount - ?, freeze_amount = freeze_amount - ?
WHERE id = ? AND freeze_amount >= ? AND amount >= ?
```

### 4.3 deductCoin — 只写账变不改余额

```java
// deductCoin 不修改 user_coin 余额，只插入 coin_record 流水
public boolean deductCoin(CoinRecordVO coinRecordVO) {
    // 1. 校验 coinValue > 0
    // 2. 获取用户信息 → 填充冗余字段
    // 3. 如果是免转钱包，coinFrom/coinTo 设为 null
    // 4. 只插入 coin_record（不修改 user_coin）
    coinRecordRepository.insert(coinRecordDTO);
    return true;
}
```

用途：MQ 消费的 `ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE` 队列使用此方法 — 只补写账变记录，不二次修改余额。

### 4.4 UserCoinServiceImpl — getWallet 查询逻辑

```java
public UserCoinVO getWallet(UserCoinVO userCoinVO) {
    UserInfoVO userInfo = commonService.getUserInfoByUserId(userId);

    if (walletType == TRANSFER) {         // 转账钱包
        UserCoinDTO userCoin = selectOne(userId);
        if (userCoin != null) {
            // ★ 返回可用余额（总额-冻结），不是总额
            vo.setAmount(userCoin.getAmount().subtract(userCoin.getFreezeAmount()));

            // ★ API 安全：非正式用户从API入口查询，强制返回0
            if (md5Sign非空 && userType != 正式用户) {
                vo.setAmount(BigDecimal.ZERO);  // 防止试玩余额被利用
            }
        } else {
            vo.setAmount(BigDecimal.ZERO);  // 钱包不存在，返回0
        }
    } else {                              // 免转钱包
        vo.setAmount(BigDecimal.ZERO);    // 余额在三方，本地返回0
        vo.setWalletType(FREE_TRANSFER);
    }
}
```

---

## 五、并发控制与资金安全机制

### 5.1 Redisson 分布式锁模式

所有写操作（addCoin/freezeCoin/subFreezeBalance/rollbackFreezeBalance）和 MQ 消费者都使用同一模式：

```java
// WalletController.java — 锁模式模板
RLock rLock = redisUtil.getLock(RedisLockKey.getAddCoinLockKey(userId));
try {
    if (rLock.tryLock(20000, 30000L, TimeUnit.MILLISECONDS)) {
        // 获锁成功 → 执行业务
        boolean result = coinRecordService.addCoin(coinRecordVO);
        if (result) {
            coinRecordService.sendUserBalance(userId);  // 通知在锁内执行
        }
    } else {
        log.info("锁超时：{}", orderNo);
    }
} catch (Exception e) {
    log.error(e.getMessage(), e);
} finally {
    if (rLock != null && rLock.isLocked() && rLock.isHeldByCurrentThread()) {
        rLock.unlock();
    }
}
```

**锁参数设计**：

| 参数 | 值 | 含义 |
|------|------|------|
| 锁粒度 | userId | 同一用户所有余额操作串行化 |
| 等待超时 | 20秒 | 最多等20秒获取锁 |
| 持有超时 | 30秒 | 获锁后最多持有30秒（防死锁） |
| 锁Key | `"addCoin:" + userId` | 所有操作共用同一个key |

### 5.2 三层防护体系

```
参考工程的资金安全是"三层防护"：

Layer 1: Redisson 分布式锁（userId粒度）
├── 效果：同一用户所有写操作串行化
├── 位置：Controller 层 / MQ Listener 层
└── 缺陷：依赖 Redis 可用性，Redis 故障锁失效

Layer 2: 应用层余额检查（insufficientBalance）
├── 效果：快速失败，避免无意义SQL
├── 只对特定操作（转出/下注/打赏）
└── 缺陷：并发场景下可能被绕过

Layer 3: SQL层条件更新（(amount-X)>=0）
├── 效果：数据库行锁级终极保障
├── 位置：subBalance SQL
└── 意义：即使Layer 1和Layer 2都失效，SQL仍能阻止超扣

实际生产中：
Layer 1 拦截 99% 并发
Layer 2 拦截大部分余额不足
Layer 3 作为兜底防极端情况
```

### 5.3 与我方并发策略对比

| 维度 | 参考工程 | 我方设计 |
|------|---------|---------|
| 锁方式 | Redisson 分布式锁 | CAS 乐观锁（version字段） |
| 额外依赖 | 依赖 Redis 可用性 | 不依赖外部组件 |
| 吞吐量 | 较低（串行执行） | 较高（无阻塞等待） |
| 死锁风险 | 有（锁超时兜底） | 无 |
| SQL非负保障 | 有 | 必须同样实现 |

**结论**：不管用什么锁策略，SQL 层的 `(amount-X)>=0` 保障都必须保留，这是最后一道防线。

---

## 六、冻结机制深度分析

### 6.1 冻结三阶段模型

```
冻结生命周期（投注场景）：

阶段1: freezeCoin（下注时，冻结资金）
  · freeze_amount += 下注金额
  · amount 不变
  · 可用余额 = amount - freeze_amount → 减少
  · ★ 不产生账变记录（钱还没真正离开账户）

阶段2a: subFreezeBalance（结算确认扣除）
  · amount -= 下注金额
  · freeze_amount -= 下注金额
  · 可用余额不变（两者同时减少相同金额）
  · ★ 产生账变记录（钱真正扣了）
  · ★ 内部调 createOrder() 生成流水记录

阶段2b: rollbackFreezeBalance（取消/退还）
  · freeze_amount -= 下注金额
  · amount 不变
  · 可用余额恢复（freeze 减少 → 可用增加）
  · ★ 不产生账变记录（相当于什么都没发生）
```

### 6.2 余额变化数值示例

```
假设初始：amount=1000, freeze=0, 可用=1000

步骤                  │ amount │ freeze │ 可用(a-f) │ 产生流水
────────────────────┼────────┼────────┼──────────┼─────────
freezeCoin(200)     │ 1000   │ 200    │ 800      │ 否
────────────────────┼────────┼────────┼──────────┼─────────
情况A: 结算确认扣除
subFreezeBalance(200)│ 800    │ 0      │ 800      │ 是
→ 钱真正被扣了       │        │        │(可用不变)│
────────────────────┼────────┼────────┼──────────┼─────────
情况B: 取消回滚
rollbackFreeze(200) │ 1000   │ 0      │ 1000     │ 否
→ 钱回来了          │        │        │(可用恢复)│
```

### 6.3 subFreezeBalance 的 createOrder 流水生成

```java
// 确认扣减冻结时生成的流水记录
public void createOrder(CoinFreezeRecordVO freezeRecord, UserCoinDTO userCoin) {
    CoinRecordDTO record = new CoinRecordDTO();
    // 填充用户/商户信息（又调一次 user-service）
    UserInfoVO userInfo = commonService.getUserInfoByUserId(userId);
    // ... 填充13个冗余字段 ...

    // ★ 关键：coinTo = 当前amount - 冻结金额
    record.setCoinTo(userCoin.getAmount().subtract(freezeRecord.getFreezeAmount()));
    record.setCoinAmount(freezeRecord.getFreezeAmount());
    record.setOrderNo(freezeRecord.getOrderNo());
    record.setCoinValue(freezeRecord.getFreezeAmount());
    coinRecordRepository.insert(record);
}
```

### 6.4 与我方冻结设计对比

| 维度 | 参考工程 | 我方设计 |
|------|---------|---------|
| 冻结存储 | user_coin.freeze_amount 汇总值 | t_freeze_record 独立表每笔记录 |
| 可追溯性 | 只知道总冻结额，不知道哪些订单冻了多少 | 每笔冻结有独立记录和生命周期 |
| 部分释放 | 不支持（一次性释放） | 支持（按记录释放） |
| 性能 | 极简，一个字段搞定 | 多一张表查询 |
| 审计 | 冻结过程不可审计 | 冻结全生命周期可审计 |

**结论**：我方设计在审计和追溯上更优；参考工程在性能和简洁性上更优。两种方案各有适用场景。

---

## 七、消息队列驱动模式

### 7.1 4个 RabbitMQ 队列

```
队列1: wallet_add_coin_queue（通用加减钱）
├── 消息格式：单个 CoinRecordVO（JSON）
├── 处理逻辑：加锁 → addCoin() → 成功后 sendUserBalance()
├── 典型场景：结算返奖、转入
└── 注意：每条消息都获取一次 Redisson 锁

队列2: wallet_add_coin_order_queue（批量下注扣减）
├── 消息格式：List<CoinRecordVO>（JSON数组）
├── 处理逻辑：deductCoinBatch() → 失败则回滚注单 → sendUserBalance()
├── 特殊处理：扣减失败时 orderResource.updateOrdersToInitialStatus(orderNos)
└── 典型场景：批量投注扣款（一个用户同时多笔下注）

队列3: wallet_add_coin_not_notice_balance_queue（静默扣减）
├── 消息格式：List<CoinRecordVO>
├── 处理逻辑：deductCoinBatch() → 失败回滚注单（不发余额通知）
└── 典型场景：闪电投注（投注过程中不更新前端余额显示，避免闪烁）

队列4: add_coin_record_and_notice_balance_queue（记录+通知）
├── 消息格式：List<CoinRecordVO>
├── 处理逻辑：changeCoinAndNoticeBalance()（只写record不改余额+通知）
└── 典型场景：下注后的账变记录补写
```

### 7.2 余额变化通知机制

```java
// 通知所有 WebSocket 节点刷新用户余额
public void sendUserBalance(String userId) {
    UserBalanceUpdateVO vo = new UserBalanceUpdateVO();
    vo.setUserId(userId);
    // 发送到 fanout 交换机（广播给所有 WebSocket 节点）
    rabbitTemplate.convertAndSend(
        MqConstants.EXCHANGE_NOTICE_USER_BALANCE_CHANGE,  // fanout exchange
        "",                                                // routing key 为空
        message
    );
}
```

```
通知链路：
wallet-service
    │ RabbitMQ fanout exchange
    ▼
┌───────┐  ┌───────┐  ┌───────┐
│ws-node│  │ws-node│  │ws-node│  ← 每个WebSocket节点都收到
└───┬───┘  └───┬───┘  └───┬───┘
    ▼          ▼          ▼
  用户A      用户B      用户C    ← 各节点推送给自己持有连接的用户

设计精髓：
1. 消息只含 userId（不携带余额数字，避免过期数据）
2. WS 节点收到后主动查 wallet getWallet（保证余额一致性）
3. fanout 广播（无需知道有多少 WS 节点）
```

### 7.3 下注失败的 MQ 补偿回滚

```java
// AddCoinListener — 下注扣减失败时
try {
    boolean result = coinRecordService.deductCoinBatch(coinRecordVOList);
    if (!result) {
        // 扣减失败 → 回滚注单状态
        List<String> orderNos = coinRecordVOList.stream()
            .map(CoinRecordVO::getOrderNo).collect(Collectors.toList());
        orderResource.updateOrdersToInitialStatus(orderNos);
    }
} catch (Exception e) {
    // 异常也回滚注单
    orderResource.updateOrdersToInitialStatus(orderNos);
}
```

这是一种轻量级 **Saga 补偿模式**：
- 不需要分布式事务协调器
- 简单的"失败就补偿"
- 时间窗口内注单状态可能不准确（最终一致性）

---

## 八、币种与汇率体系

### 8.1 架构归属

```
币种/汇率不在 wallet-service，归 merchant-service 管理：

merchant-service
├── currency 表 → 币种基础信息
├── currency_rate 表 → 汇率配置
├── CurrencyServiceImpl → 加载币种到 Redis
├── CurrencyRateServiceImpl → CRUD汇率 + 刷新Redis缓存
└── CurrencyRateController → B端管理接口

p9-core-lib（共享库中的工具类，任何服务都能调）
├── CurrencyBusiness.java
│   · getAllCurrencyRate() → 从 Redis 读全量汇率
│   · getCurrencyRate(code) → 从 Redis 读单币种汇率
│   · toUSDAmount(currency, amount) → amount / rate 换算为 USD
└── CurrencyEnum.java → 5种币种硬编码枚举（USD/CNY/TWD/INR/BRL）
```

### 8.2 汇率缓存设计（双层缓存）

从 `CurrencyRateServiceImpl.setRedCurrency()` 源码反推：

```
Redis Key 设计（双层）：

Layer 1 — 全量列表：
  Key:   merchantCurrency
  Value: JSON Array（所有汇率对象列表）
  用途:  B端下拉选择、批量展示

Layer 2 — 单币种 Key（两种格式）：
  Key:   findRateByCurrencyCode:CNY
  Value: 7.2（BigDecimal 数值）
  用途:  换算时快速查单个汇率

  Key:   merchantRateCurrencyCode:CNY
  Value: JSON Object（完整汇率对象含symbol/status等）
  用途:  需要完整信息时使用

刷新机制：
  后台编辑汇率
    → CurrencyRateServiceImpl.updateCurrency()
    → 更新DB（含唯一性校验：currencyCode/currencyName 不可重复）
    → setRedCurrency()（刷新全量 + 逐个刷新单key）
    → currencyService.initCurrencyRedis()（刷新币种列表）
    → MQ通知刷新会员等级Redis（汇率变化可能影响等级计算）

读取链路：
  任何服务 → CurrencyBusiness.getCurrencyRate("CNY") → Redis → 返回7.2
  缓存不存在 → 抛异常（不回源DB！依赖定期刷新机制保持缓存有效）
```

### 8.3 汇率换算逻辑

```java
// CurrencyBusiness.toUSDAmount()
public BigDecimal toUSDAmount(String currency, BigDecimal amount) {
    if ("USD".equals(currency)) return amount;
    BigDecimal rate = getCurrencyRate(currency);
    return amount.divide(rate, 2, RoundingMode.HALF_UP);
}
// 例：100 CNY → USD，rate=7.2 → 100/7.2 = 13.89 USD
```

**如果我们用 BIGINT 存储的换算注意**：
```
BIGINT 场景：存储值是最小单位
例：100 CNY = 10000分（BIGINT存10000），rate=7.2

错误做法：10000 / 720 = 13 → 只有 $0.13（精度严重丢失）
正确做法：用高精度中间计算
  10000 * 100 / 720 = 1388.88 → 四舍五入 = 1389（分）→ $13.89
  或使用 shopspring/decimal 库做精确计算
```

---

## 九、跨服务依赖拓扑

### 9.1 wallet-service 调用谁（4个 Feign 依赖）

```
wallet-service
├── → user-service:8906
│     · getUserByUserId(userId) → UserInfoVO
│     · 频率：每次 addCoin 都要调（有Redis缓存但不回写！）
│     · 含：userName, walletType, currency, merchantId/No/Name, ...
│
├── → merchant-service:8902
│     · getMerchantDetail(merchantId) → MerchantDetailVO
│     · 频率：仅免转钱包分支调用（Redis缓存1天TTL）
│
├── → order-service:8908
│     · findOrderByOrderNo(orderNo) → OrderVO（免转钱包查订单）
│     · updateOrdersToInitialStatus(orderNos) → void（下注失败回滚）
│     · saveThirdOrderRecord(record) → void（保存三方订单记录）
│
└── → api-service:8910
      · amountTransfer(request) → ThirdOrderResponseVO
      · 仅免转钱包使用（调三方平台加减余额）
```

### 9.2 谁调用 wallet-service（7+服务）

```
调用 wallet-service 的服务：

┌────────────────────┬──────────┬───────────────────────────┬────────────┐
│ 服务               │ 调用方式  │ 使用接口                   │ 场景        │
├────────────────────┼──────────┼───────────────────────────┼────────────┤
│ order-service      │ MQ       │ addCoin（4个队列）          │ 下注/结算   │
│                    │ 共享DB⚠️ │ 直读 user_coin 表          │ 投注余额校验│
│ websocket-service  │ HTTP     │ getWallet, addCoin, 充提   │ C端全功能   │
│ api-service        │ HTTP     │ getWallet, addCoin, 查询   │ API聚合     │
│ auth-service       │ HTTP     │ getWallet                  │ 登录查余额  │
│ user-service       │ HTTP     │ getWallet                  │ 用户信息    │
│ admin-service      │ HTTP     │ getWallet, 账变分页        │ B端后台     │
│ admin-merchant-    │ HTTP     │ getWallet, 账变分页        │ 商户后台    │
│ line-service       │          │                            │            │
│ job-executor       │ HTTP     │ 清理临时数据               │ 定时任务    │
│ 12+ 游戏服务       │ MQ       │ addCoin 队列               │ 投注/结算   │
└────────────────────┴──────────┴───────────────────────────┴────────────┘
```

**重要发现 — order-service 直读 user_coin 表（共享数据库）**：

```
这是参考工程的一个架构妥协。

order-service 为了投注时的性能（避免 RPC 延迟），
直接访问了 wallet-service 的 user_coin 表进行余额校验。

问题：
· 破坏了微服务的数据隔离原则
· user_coin 表结构变更会影响 order-service
· 两个服务对同一张表的并发操作增加复杂性

我方必须避免：
· ser-wallet 数据完全自持
· 其他服务通过 Kitex RPC 接口查询余额
· 如果性能是瓶颈，用 Redis 缓存最新余额
```

### 9.3 CommonServiceImpl 的缓存设计（有缺陷）

```java
// getUserInfoByUserId — 获取用户信息
public UserInfoVO getUserInfoByUserId(String userId) {
    String redisKey = "CURRENCY_LIMIT_ID_MERCHANT_ID" + userId;
    Object obj = redisUtil.get(redisKey);
    if (obj instanceof UserInfoVO) {
        return (UserInfoVO) obj;   // 缓存命中 → 直接返回
    }
    // 缓存未命中 → Feign 调 user-service
    UserInfoVO userInfo = userResource.getUserByUserId(userId).getBody().getData();
    // ⚠️ 注意：这里没有写回缓存！
    return userInfo;
}

// getMerchantDetailVO — 获取商户详情
public MerchantDetailVO getMerchantDetailVO(Long merchantId) {
    String redisKey = getMerchantCacheKey(merchantId);
    Object obj = redisUtil.get(redisKey);
    if (obj instanceof MerchantDetailVO) {
        return (MerchantDetailVO) obj;  // 缓存命中
    }
    MerchantDetailVO detail = merchantResource.getMerchantDetail(id).getBody().getData();
    redisUtil.set(redisKey, detail, 1, TimeUnit.DAYS);  // ★ 写回缓存，TTL 1天
    return detail;
}
```

**缺陷分析**：`getUserInfoByUserId` 查 Redis → miss → 调 Feign，但**没有把结果写回 Redis**。这意味着每次 miss 都要走 Feign 调用，在高并发下成为瓶颈。而 `getMerchantDetailVO` 正确实现了"回写缓存"。

**我方应该做到**：查缓存 → miss → RPC → **回写缓存** → 返回。

---

## 十、枚举体系与VO/DTO设计

### 10.1 账变类型枚举 (CoinChangeEnum — 9种)

```
code │ 名称              │ 含义              │ 方向   │ 余额检查
─────┼───────────────────┼──────────────────┼───────┼─────────
  1  │ TRANSFER_IN       │ 转入（充值上分）   │ 收入   │ 否
  2  │ TRANSFER_OUT      │ 转出（提现下分）   │ 支出   │ ✓ 检查
  3  │ BET               │ 下注              │ 支出   │ ✓ 检查
  4  │ SETTLEMENT        │ 派彩（结算加钱）   │ 收入   │ 否
  5  │ ORDER_CANCEL      │ 注单取消（退钱）   │ 收入   │ 否(SQL跳过)
  6  │ SETTLEMENT_ROLLBACK│ 结算回滚（扣钱）  │ 支出   │ 否(SQL跳过,允许负)
  7  │ SETTLEMENT_SECOND │ 重算              │ 可收可支│ 否
  8  │ SEND_REWARD       │ 打赏              │ 支出   │ ✓ 检查
  9  │ LIGHTNING_BET     │ 闪电投注          │ 支出   │ ✓ 检查
```

### 10.2 账变明细枚举 (CoinChangeDetailEnum — 8种)

```
code │ 名称                │ 含义
─────┼────────────────────┼──────────
  1  │ TRANSFER_IN        │ 会员存款
  2  │ TRANSFER_OUT       │ 会员提款
  3  │ BET                │ 下注
  4  │ SETTLEMENT         │ 结算
  5  │ ORDER_CANCEL       │ 取消
  6  │ SETTLEMENT_CANCEL  │ 结算取消
  7  │ SEND_REWARD        │ 打赏
  8  │ LIGHTNING_BET      │ 闪电投注
```

**coinType vs coinDetail 的关系**：
- coinType = 业务逻辑驱动（`if coinType == BET` 决定走什么分支）
- coinDetail = 显示逻辑驱动（UI 显示什么文案，支持 i18n）
- 目前两者几乎一一对应，实际作用重叠
- 我方可简化为单层 flow_type + 扩展 remark/source 字段

### 10.3 其他枚举

| 枚举 | 值 | 含义 |
|------|------|------|
| WalletTypeEnum | 1=TRANSFER, 2=FREE_TRANSFER | 转账钱包/免转钱包 |
| BalanceTypeEnum | 1=income, 2=expenses | 收入/支出 |
| OrderPaymentMethodEnum | wechat/alipay/bank_card/VIRTUAL_CURRENCY/... | 支付方式 |

### 10.4 核心 VO 清单（p9-core-lib 共享）

| VO | 用途 | 关键字段 |
|------|------|---------|
| CoinRecordVO | 账变操作入参+出参（混用） | userId, orderNo, coinType, coinDetail, coinValue, balanceType, coinFrom, coinTo |
| UserCoinVO | 钱包余额查询出参 | userId, amount, walletType, currency, md5Sign |
| CoinFreezeVO | 冻结/回滚入参（极简） | userId, freezeAmount |
| CoinFreezeRecordVO | 确认扣减入参 | userId, freezeAmount, orderNo, coinType, coinDetail, balanceType |
| QueryWalletReqVO | 批量查询入参 | userIdList |
| BalanceVO | 内部余额操作传参 | id, amount, updateTime, coinType |
| WithdrawLimitAmountParamVO | 提现限额入参 | userId, type(1~4), withdrawalLimitAmount |
| UserWithDrawParamVo | 提现申请入参 | userId, amount, walletType(1=银行卡/2=虚拟货币), currency |

**VO 设计的问题**：CoinRecordVO 同时充当"操作入参"和"查询出参"，导致字段含义模糊。我方 Thrift IDL 天然将 Request/Response 分离，设计更清晰。

**WithdrawLimitAmountParamVO 揭示的业务逻辑**：提现有"流水限额"概念 — 用户充值后需要完成一定投注流水才能提现（反洗钱合规要求）。type 区分限制来源：1=后台人工加额, 2=会员充值(三方), 3=会员充值(USDT), 4=注单。

---

## 十一、定时任务与数据清理

```java
// BackWaterDayTask — 通过 XXL-Job 定时触发
// 入口：DeleteTempController.deleteTempTask(days)

清理内容：
1. deleteAddCoinMessage(days)
   · 删除 N 天前的 add_coin_message 记录
   · ★ 当前代码被注释掉了 — 实际未执行

2. deleteSystemCoinRecord(-5)
   · 删除 5 天前的试玩用户账变记录
   · 条件：merchantNo="system" AND merchantId=0 AND userType=1(试玩)
   · 目的：试玩用户数据不需要长期保留
```

**借鉴点**：试玩用户数据定期清理是合理的运维策略。我方可考虑类似机制。

---

## 十二、设计模式提炼（值得借鉴）

### 模式1: 数据库级非负余额保障 ★★★★★

```sql
UPDATE user_coin SET amount = amount - #{amount}
WHERE id = #{id} AND (amount - #{amount}) >= 0
-- 返回 affected_rows == 0 表示余额不足
```

**价值**：消除"先查后改"竞态窗口。即使没有锁，也能保证余额不为负。

**Go 实现**：
```go
result := db.Model(&WalletAccount{}).
    Where("id = ? AND (amount - ?) >= 0", id, deductAmount).
    UpdateColumn("amount", gorm.Expr("amount - ?", deductAmount))
if result.RowsAffected == 0 {
    return ErrInsufficientBalance
}
```

### 模式2: 前后余额快照 ★★★★★

```
每条流水记录保存：
  coin_from  = 操作前余额
  coin_to    = 操作后余额
  coin_value = 变动金额

验证公式：
  收入: coin_to = coin_from + coin_value
  支出: coin_to = coin_from - coin_value
```

**价值**：审计追踪时不依赖时序重算，每条记录自描述。出现异常可快速定位"从哪里开始不对"。

### 模式3: 冻结三阶段 ★★★★

```
freeze（预扣）→ confirm(subFreeze) / cancel(rollback)
```

经典预授权模式。冻结不产生账变（合理）— 中间状态不需要记录。

### 模式4: 特定操作跳过非负校验 ★★★★

```
结算回滚/取消 → 允许余额为负
转出/下注/打赏 → 必须检查余额
```

通过 SQL 条件动态控制（`vo.coinType != 6 and vo.coinType != 5` 时才加非负条件），灵活且安全。

### 模式5: 余额变更 fanout 广播通知 ★★★★

```
余额变动 → MQ fanout exchange → 所有 WebSocket 节点
消息只含 userId → WS 节点主动查 getWallet → 推送最新余额
```

**价值**：消息体极小，不存在消息与实际余额不一致问题。

### 模式6: 用户信息 Redis 缓存 + Feign 兜底 ★★★

```
频繁调用的用户信息走 Redis → miss 走 Feign（但应回写缓存！）
商户信息 Redis 缓存 1天 TTL
```

**注意**：参考工程 getUserInfo 未回写缓存是缺陷。我方应做到回写。

---

## 十三、设计缺陷提炼（需要规避）

### 缺陷1: 无幂等性保护 ⚠️⚠️⚠️⚠️⚠️

```
order_no 没有唯一索引。
MQ 消息重复投递 → 同一笔账变执行两次 → 余额多加/多扣。
add_coin_message 表虽然存在但未用于去重。
```

**我方对策**：t_wallet_flow 的 (order_no, flow_type) 唯一索引。

### 缺陷2: 无乐观锁（version字段）⚠️⚠️⚠️⚠️

```
user_coin 表没有 version 字段。
完全依赖 Redisson 锁 → Redis 故障时并发窗口。
```

**我方对策**：t_wallet_account 必须有 version 字段。

### 缺陷3: subFreezeBalance 缺少 SQL 非负保护 ⚠️⚠️⚠️

```
subFreezeBalance 的 SQL 没有 (amount-X)>=0 条件。
如果 Redisson 锁失效，amount 和 freeze_amount 都可能被扣到负数。
```

**我方对策**：冻结相关 SQL 也必须加条件检查。

### 缺陷4: getUserInfo 缓存不回写 ⚠️⚠️⚠️

```
每次 addCoin 都要获取用户信息。
缓存 miss 时走 Feign，但没有把结果写回 Redis。
高并发下每次都打到 user-service。
```

**我方对策**：RPC 结果必须回写缓存。

### 缺陷5: coinFrom 竞态问题 ⚠️⚠️

```java
coinRecordDTO.setCoinFrom(userCoinDTO1.getAmount());
// 这里用的是"查询时的余额"而非"更新后的数据库真实值"
// 被 Redisson 锁避免了，但如果锁失效 coinFrom 就不准确
```

### 缺陷6: SQL 注入风险 ⚠️⚠️

```xml
ORDER BY s.created_time ${vo.createdTimeBy}, s.id ${vo.createdTimeBy}
<!-- ${} 字符串替换，如果 createdTimeBy 来自用户输入可注入 -->
```

**我方对策**：排序字段通过白名单枚举校验。

### 缺陷7: 锁和事务边界不一致 ⚠️

```
锁在 Controller 层获取，事务在 Service 层 @Transactional。
分散两层增加维护成本。锁释放后事务可能还未提交（极端情况）。
```

### 缺陷8: addCoin 方法过长 ⚠️

```
addCoin() 170+ 行，转账/免转两种钱包类型 if-else 耦合在一起。
应使用策略模式或拆分方法。
```

### 缺陷9: DTO 既是 Entity 又是传输对象 ⚠️

```
UserCoinDTO 同时有 @TableName 做 ORM 映射，又在 Service 间传递。
DB 字段变更影响所有上下游。
```

**我方优势**：GORM Gen 天然分离 model 和 query。

---

## 十四、全链路对比总表

| 维度 | 参考工程（Java） | 我方设计（Go） | 结论 |
|------|-----------------|---------------|------|
| 分层 | Controller→Service→Repository（3层） | handler→service→repository（3层+Ledger） | 我方 Ledger 层职责更清晰 |
| 并发 | Redisson 分布式锁（Redis级） | CAS乐观锁（DB版本号） | 我方不依赖外部组件 |
| 幂等 | 无（流水表无唯一索引） | 流水表 unique(order_no+flow_type) | 我方更安全 |
| 金额类型 | BigDecimal(20,2) | BIGINT（最小单位） | 我方无浮点误差 |
| 冻结 | freeze_amount 汇总值 | t_freeze_record 独立表 | 我方审计更完整 |
| 非负保障 | SQL条件 (amount-X)>=0 | **应直接采用同模式** | ★ 核心借鉴 |
| 余额快照 | coinFrom/coinTo | before_balance/after_balance | ★ 核心借鉴 |
| 币种模型 | 单币种/单行 | 多币种 × 多子钱包 | 我方需求更复杂 |
| 汇率 | USD基准，Redis双层缓存 | 可借鉴缓存策略 | ★ 缓存方案借鉴 |
| 通信 | RabbitMQ（4队列重度依赖） | 同步RPC + NATS/Kafka | 架构差异最大 |
| 账变枚举 | coinType(9)+coinDetail(8)双层 | flow_type + direction 更清晰 | 简化层级 |
| 钱包边界 | 只管余额操作 | 含充值/提现/兑换/币种 | 我方范围更大 |
| 定时清理 | XXL-Job清理试玩数据 | 可借鉴 | 运维策略参考 |
| VO设计 | 入参出参混用同一VO | Thrift IDL Req/Resp分离 | 我方更清晰 |
| 共享库 | p9-core-lib（枚举/VO） | common/（consts/utils） | 模式一致 |

---

## 十五、可借鉴清单

### 15.1 直接采纳（已验证的好模式）

| # | 模式 | 具体落地 |
|---|------|---------|
| A1 | 数据库级非负保障 | `UPDATE SET amount = amount - ? WHERE id = ? AND (amount - ?) >= 0`，rows==0 表示余额不足 |
| A2 | 前后余额快照 | t_wallet_flow 包含 before_balance / after_balance / change_amount |
| A3 | 冻结三阶段 | Freeze → Confirm(Debit) / Cancel(Unfreeze) |
| A4 | 特定操作跳过非负校验 | 结算回滚允许负余额（通过参数控制SQL条件） |
| A5 | 汇率 Redis 双层缓存 | 全量列表key + 单币种key |
| A6 | 余额变更 fanout 广播 | NATS Pub → 所有 WS 节点，消息只含 userId |
| A7 | 首次加钱自动创建钱包 | lazy init 模式，首次收入时自动创建 wallet_account 行 |

### 15.2 借鉴思路但需改造

| # | 参考工程做法 | 我方改造方向 |
|---|------------|------------|
| B1 | coinType+coinDetail 双层枚举 | 改为 flow_type（业务标识）+ direction（收/支）|
| B2 | CoinRecordVO 入参出参混用 | Thrift IDL 天然 Request/Response 分离 |
| B3 | 用户信息 Redis 缓存 + Feign 兜底 | 同模式但修复回写缺陷，加 TTL |
| B4 | MQ 下注失败回滚注单 | 改为 RPC 回调或 NATS 补偿消息 |
| B5 | 试玩用户数据定期清理 | 后期运维阶段加入 |
| B6 | 提现流水限额(WithdrawLimitAmount) | 如有反洗钱需求，可参考此概念 |
| B7 | 流水表冗余 merchant_id | 保留此冗余（后台筛选刚需），其余用户字段按需 JOIN |

### 15.3 明确不采纳

| # | 参考工程做法 | 不采纳原因 |
|---|------------|-----------|
| C1 | Redisson 分布式锁 | CAS乐观锁更简单，不依赖额外组件 |
| C2 | 共享数据库（order直读user_coin） | 破坏微服务数据隔离原则 |
| C3 | 流水无幂等索引 | 必须有唯一索引保障幂等 |
| C4 | 币种枚举硬编码 | 数据库驱动，支持动态增删 |
| C5 | DTO 即 Entity | GORM Gen 已天然分离 |
| C6 | 免转钱包分支 | 当前需求无此场景 |
| C7 | BigDecimal 金额存储 | 已确定用 BIGINT |
| C8 | user_state/game_type 放钱包表 | 不相关字段不放钱包表 |
| C9 | subFreezeBalance 不加非负保护 | 必须加 SQL 条件检查 |

---

## 十六、附录

### 附录A: wallet-service API 端点总览

**WalletController（/webapi/wallet）— 核心写操作**：

| 方法 | 端点 | 用途 | 加锁 |
|------|------|------|------|
| POST | /addCoin | 单笔加减钱 | ✓ Redisson |
| POST | /addCoinList | 批量加减钱 | ✓ Redisson |
| POST | /getWallet | 查余额（返回可用余额） | 否 |
| POST | /users/wallet/list | 批量查余额 | 否 |
| POST | /queryCoinRecordByOrderNo | 按订单号查账变 | 否 |
| POST | /queryCoinRecord | 查账变列表 | 否 |
| POST | /freezeCoin | 冻结金额 | ✓ Redisson |
| POST | /subFreezeBalance | 确认扣减冻结 | ✓ Redisson |
| POST | /rollbackFreezeBalance | 回滚冻结 | ✓ Redisson |

**CoinRecordController（/webapi/wallet）— 账变查询**：

| 方法 | 端点 | 用途 |
|------|------|------|
| POST | /coinRecord/page | 账变分页查询（B端后台） |
| POST | /coinRecord/list | 账变列表查询 |
| POST | /coinRecord/count | 账变记录总数 |
| POST | /findCoinResult | 转账结果查询（api-service专用） |
| POST | /getCoinRecord | 单条账变查询 |
| POST | /userCoinRecordPage | C端用户账变分页 |
| POST | /userCoinRecordList | C端用户账变列表 |

### 附录B: Redis Key 一览

| Key | 内容 | 来源 | TTL |
|-----|------|------|-----|
| merchantCurrency | 全量汇率 JSON Array | merchant-service | 无过期 |
| findRateByCurrencyCode:{CODE} | 单币种汇率值 | merchant-service | 无过期 |
| merchantRateCurrencyCode:{CODE} | 单币种完整对象 | merchant-service | 无过期 |
| currency | 全量币种 JSON Array | merchant-service | 无过期 |
| CURRENCY_TO_SYMBOL | 币种→符号映射 | merchant-service | 按需刷新 |
| CURRENCY_LIMIT_ID_MERCHANT_ID:{userId} | 用户信息缓存 | wallet CommonService | 无过期（但不回写⚠️） |
| MerchantDetailVO:{merchantId} | 商户详情 | wallet CommonService | 1天 |
| addCoin:{userId} | 钱包操作锁 | wallet Controller/Listener | 30秒 |

### 附录C: MQ 队列清单

| 队列名 | 消息格式 | 消费者 | 用途 |
|--------|---------|--------|------|
| wallet_add_coin_queue | 单条 CoinRecordVO | AddCoinListener | 通用加减钱 |
| wallet_add_coin_order_queue | List\<CoinRecordVO\> | AddCoinListener | 批量下注扣减 |
| wallet_add_coin_not_notice_balance_queue | List\<CoinRecordVO\> | AddCoinListener | 静默扣减（不通知） |
| add_coin_record_and_notice_balance_queue | List\<CoinRecordVO\> | AddCoinListener | 补写记录+通知 |
| exchange_notice_user_balance_change | UserBalanceUpdateVO | websocket-service | 余额变化通知 |

### 附录D: 关键源码文件索引

| 文件 | 行数 | 核心价值 |
|------|------|---------|
| CoinRecordServiceImpl.java | 522 | ★★★ 全部核心业务逻辑 |
| WalletController.java | 251 | ★★ 锁模式 + 端点定义 |
| UserCoinMapper.xml | 44 | ★★★ 5条核心原子SQL |
| CoinRecordMapper.xml | 284 | 查询SQL（含SQL注入案例） |
| AddCoinListener.java | 148 | ★★ 4个MQ消费模式 |
| UserCoinServiceImpl.java | 87 | ★ getWallet逻辑+API安全检查 |
| CommonServiceImpl.java | 74 | ★ 缓存模式（含缺陷案例） |
| CurrencyRateServiceImpl.java | 146 | ★ 汇率Redis双层缓存设计 |
| CurrencyServiceImpl.java | 53 | 币种Redis初始化 |
| WalletUtilEnum.java | 153 | 账变类型+明细枚举定义 |
| CoinRecordDataServiceImpl.java | 181 | 分页查询+枚举名翻译 |
| UserCoinDTO.java | 33 | user_coin 表结构 |
| CoinRecordDTO.java | 60 | coin_record 表结构 |
| CurrencyDTO.java | 24 | currency 表结构 |
| CurrencyRateDTO.java | 31 | currency_rate 表结构 |

---

## 总结

```
参考工程钱包模块的本质：
  一个"极简但有效"的单币种钱包系统
  3张表 + 1个核心Service + 4个MQ监听 → 支撑了线上真人视讯平台

它做得好的（必须借鉴）：
  ① SQL级非负余额保障 — 不依赖锁也能防超扣
  ② 前后余额快照 — 审计追踪的基石
  ③ 冻结三阶段 — 投注预授权的行业标准
  ④ 特殊操作允许负余额 — 竞彩平台的业务刚需
  ⑤ 余额通知 fanout 广播 — 解耦、高效
  ⑥ 汇率 Redis 双层缓存 — 全量+单key，读取极快

它做得不好的（必须规避）：
  ① 无幂等保障 — 流水表无唯一索引
  ② 无乐观锁 — 完全依赖外部Redis锁
  ③ 冻结SQL缺保护 — subFreezeBalance 无非负条件
  ④ 缓存不回写 — getUserInfo 每次miss都打RPC
  ⑤ 共享数据库 — order-service 直读钱包表
  ⑥ 代码耦合 — addCoin 170行混合两种钱包类型

我方 Go ser-wallet 的相对优势：
  · 多币种 × 多子钱包（vs 参考工程的单钱包单币种）
  · GORM Gen 分离 model/query（vs 参考工程 DTO 混用）
  · 唯一索引保障幂等（vs 参考工程无幂等）
  · CAS 乐观锁不依赖外部组件（vs 参考工程依赖 Redis）
  · Thrift IDL 天然 Request/Response 分离（vs 参考工程 VO 混用）
  · 独立 t_freeze_record 表全生命周期审计（vs 参考工程仅汇总值）

核心结论：
  参考工程证明了 — 钱包系统不需要复杂，但关键的安全机制一个都不能少。
  SQL级非负保障、前后余额快照、冻结三阶段 → 这三个是不可妥协的底线。
```
