# 参考钱包工程深度解剖分析

> 工程路径：/Users/mac/gitlab/z-tmp/backend
> 分析对象：wallet-service 及其跨服务联动模块
> 分析深度：代码级逐方法解剖，等同于"我从0到1实现了这个钱包"
> 目标：取其精华去其糟粕，为我们的 Go ser-wallet 提供参照基准

---

## 目录

- [第一章 技术栈全景](#第一章-技术栈全景)
- [第二章 工程结构与模块拓扑](#第二章-工程结构与模块拓扑)
- [第三章 钱包架构设计哲学](#第三章-钱包架构设计哲学)
- [第四章 数据模型与表结构](#第四章-数据模型与表结构)
- [第五章 余额模型深度解析](#第五章-余额模型深度解析)
- [第六章 核心服务逐方法解剖](#第六章-核心服务逐方法解剖)
- [第七章 并发控制体系](#第七章-并发控制体系)
- [第八章 SQL原子操作分析](#第八章-sql原子操作分析)
- [第九章 MQ消息架构](#第九章-mq消息架构)
- [第十章 接口设计与API清单](#第十章-接口设计与api清单)
- [第十一章 跨服务依赖与调用链路](#第十一章-跨服务依赖与调用链路)
- [第十二章 冻结/解冻/回滚三阶段](#第十二章-冻结解冻回滚三阶段)
- [第十三章 Bug清单与设计缺陷](#第十三章-bug清单与设计缺陷)
- [第十四章 取舍评价：精华与糟粕](#第十四章-取舍评价精华与糟粕)
- [第十五章 对我们Go实现的启示](#第十五章-对我们go实现的启示)
- [附录A 枚举值完整映射表](#附录a-枚举值完整映射表)
- [附录B 全服务Feign接口清单](#附录b-全服务feign接口清单)
- [附录C 关键代码片段索引](#附录c-关键代码片段索引)

---

## 第一章 技术栈全景

### 1.1 基础框架

| 维度 | 选型 | 版本 |
|------|------|------|
| 语言 | Java | 17 (EA JDK slim buster) |
| 核心框架 | Spring Boot | 2.7.4 |
| 微服务框架 | Spring Cloud | 2021.0.4 |
| ORM | MyBatis-Plus | 3.5.3 (推测，基于 BaseMapper 用法) |
| 分布式锁 | Redisson | (版本随 Spring Boot Starter) |
| 消息队列 | RabbitMQ | (Spring AMQP Starter) |
| 服务间调用 | OpenFeign | (Spring Cloud 2021.0.4) |
| 关系数据库 | MySQL | (JDBC URL 确认) |
| 文档数据库 | MongoDB | (部分服务使用) |
| 缓存 | Redis Cluster | 7节点集群 (7000-7006) |
| 对象存储 | MinIO | |
| 容器化 | Docker (Jib) | base: openjdk:17-ea-jdk-slim-buster |
| 编排 | Kubernetes | K8s SVC 服务发现 |
| 构建 | Maven | 多模块聚合 |

### 1.2 技术栈选型评价

**合理之处：**
- Spring Boot 2.7.4 + Spring Cloud 2021.0.4 是 Java 微服务的成熟组合
- MyBatis-Plus 相比纯 MyBatis 减少了大量样板代码，XML Mapper 只用于复杂 SQL
- Redisson 是 Java 生态中最成熟的分布式锁方案
- RabbitMQ 对需要 ACK 确认的金融场景比 Kafka 更合适（虽然本项目并未充分利用）
- OpenFeign 声明式调用比手写 RestTemplate 更清晰

**值得商榷：**
- Java 17 EA (Early Access) 用于生产环境存在风险，应使用 GA 版本
- MongoDB 在此项目中仅部分服务使用，引入增加了运维复杂度，且钱包核心不使用
- 没有使用 Nacos/Consul 等服务注册中心，直接 K8s SVC 做发现——简单但限制了非 K8s 环境
- publisher-confirm-type: CORRELATED 配置了 RabbitMQ 发布确认，但代码层面并未实现 ConfirmCallback

### 1.3 与我们Go技术栈的对应关系

| Java 参考项目 | 我们的 Go 项目 | 差异说明 |
|--------------|---------------|---------|
| Spring Boot | CloudWeGo Hertz | HTTP 框架 |
| OpenFeign | CloudWeGo Kitex | RPC（我们用 Thrift IDL，比 HTTP Feign 更高效） |
| MyBatis-Plus | GORM Gen | ORM（Gen 生成类型安全代码，比 MyBatis XML 更安全） |
| Redisson | go-redis + SetNX + Lua | 我们需自建，但更轻量 |
| RabbitMQ | (待定，可能 RabbitMQ 或 Kafka) | 消息队列 |
| MySQL | TiDB | 分布式数据库，天然支持水平扩展 |
| Redis Cluster | Redis Cluster | 相同 |
| K8s SVC | ETCD 服务发现 | Kitex 原生支持 ETCD |

---

## 第二章 工程结构与模块拓扑

### 2.1 顶层模块结构

```
z-tmp/backend/
├── pom.xml                          # 根 POM，聚合所有子模块
├── p9-infrastructure/               # 基础设施（网关等）
├── p9-core-lib/                     # 公共库（枚举、VO、工具类、Feign接口）
├── p9-services/                     # 9个业务服务
│   ├── wallet-service/              # ★ 钱包服务（本次分析重点）
│   ├── user-service/                # 用户服务
│   ├── merchant-service/            # 商户服务
│   ├── auth-service/                # 认证服务
│   ├── api-service/                 # API网关/三方对接服务
│   ├── field-service/               # 现场管理服务
│   ├── file-service/                # 文件服务
│   ├── system-business-service/     # 系统业务服务
│   └── report-service/              # 报表服务
├── p9-live-services/                # 19个真人游戏服务
│   ├── order-service/               # ★ 订单服务（直接操作钱包表，重点关注）
│   ├── websocket-service/           # ★ WS服务（钱包最大调用方）
│   ├── common-service/              # 公共服务（充值/提现中转）
│   ├── baccarat-service/            # 百家乐
│   ├── dragon-tiger-service/        # 龙虎
│   ├── bull-service/                # 牛牛
│   ├── sangong-service/             # 三公
│   ├── ... (15个其他游戏服务)
│   └── rac-service/                 # 风控服务
├── p9-backstage-services/           # 4个后台管理服务
│   ├── admin-service/               # 总后台
│   ├── admin-merchant-line-service/ # 商户线管理
│   ├── admin-field-service/         # 现场管理后台
│   └── admin-websocket-service/     # 后台WS
├── p9-job-executor/                 # 定时任务
└── p9-datasource-services/          # 数据源服务
```

**总计：32+ 微服务**

### 2.2 wallet-service 内部结构

```
p9-services/wallet-service/
├── pom.xml
├── src/main/java/com/maya/p9/
│   ├── WalletApplication.java              # 启动类
│   ├── controller/
│   │   └── WalletController.java           # ★ 9个REST端点，锁在此层
│   ├── service/
│   │   ├── CoinRecordService.java          # 接口
│   │   ├── UserCoinService.java            # 接口
│   │   ├── CommonService.java              # 用户信息查询封装
│   │   ├── impl/
│   │   │   ├── CoinRecordServiceImpl.java  # ★ 核心实现（522行，15个方法）
│   │   │   └── UserCoinServiceImpl.java    # 钱包余额查询
│   │   └── feign/
│   │       ├── OrderResource.java          # 调用 order-service
│   │       ├── ApiResource.java            # 调用 api-service（三方转账）
│   │       ├── UserService.java            # 调用 user-service
│   │       └── MerchantFeignResource.java  # 调用 merchant-service
│   ├── dto/
│   │   ├── UserCoinDTO.java                # user_coin 表映射
│   │   ├── CoinRecordDTO.java              # coin_record 表映射
│   │   └── ThirdOrderRecordDTO.java        # 三方订单记录
│   ├── repository/
│   │   ├── UserCoinRepository.java         # user_coin DAO
│   │   └── CoinRecordRepository.java       # coin_record DAO
│   ├── vo/wallet/
│   │   ├── CoinRecordVO.java               # 账变请求/响应
│   │   ├── CoinFreezeVO.java               # 冻结请求
│   │   ├── CoinFreezeRecordVO.java         # 解冻+账变请求
│   │   ├── UserCoinVO.java                 # 钱包余额VO
│   │   └── QueryWalletReqVO.java           # 批量查询请求
│   ├── rabbitmq/listener/
│   │   └── AddCoinListener.java            # ★ 4个MQ消费者
│   ├── constant/
│   │   └── MqConstants.java                # MQ队列/交换机常量
│   └── scheduled/
│       └── BackWaterDayTask.java           # 定时清理任务
└── src/main/resources/
    ├── bootstrap.yml                        # 服务配置（port=8907）
    └── mapper/
        └── UserCoinMapper.xml              # ★ 5个原子SQL操作
```

### 2.3 关键观察

1. **wallet-service 代码量极小**：核心逻辑集中在 CoinRecordServiceImpl.java (522行) + WalletController.java (251行) + AddCoinListener.java (147行) + UserCoinMapper.xml (44行)，总计不到 1000 行有效代码
2. **没有独立的 pay-service 源码**：支付服务代码不在此仓库，属于独立部署
3. **Controller 层承担了锁职责**：分布式锁在 Controller 而非 Service 层
4. **VO/DTO 分离不彻底**：CoinRecordVO 同时用于请求入参、MQ消息体、Feign传输

---

## 第三章 钱包架构设计哲学

### 3.1 双钱包模式（核心架构决策）

这是整个钱包最重要的设计决策：

```
┌─────────────────────────────────────────────────────┐
│                    Wallet Service                     │
│                                                       │
│   ┌─────────────────┐     ┌──────────────────────┐   │
│   │  Transfer Wallet │     │ Free-Transfer Wallet  │   │
│   │  (转账钱包)       │     │ (免转钱包)             │   │
│   │                   │     │                        │   │
│   │  余额存本地 DB    │     │  余额存三方平台         │   │
│   │  user_coin 表     │     │  通过 API 读写         │   │
│   │                   │     │                        │   │
│   │  操作：本地        │     │  操作：Feign→api→三方  │   │
│   │  addBalance()    │     │  amountTransfer()      │   │
│   │  subBalance()    │     │                        │   │
│   └─────────────────┘     └──────────────────────┘   │
│                                                       │
│   判断依据：userInfoVO.getWalletType()                │
│   1 = TRANSFER (本地)    2 = FREE_TRANSFER (三方)     │
└─────────────────────────────────────────────────────┘
```

**设计意图：**
- 转账钱包（TRANSFER）：余额由平台自己管理，用户充值后到本地 user_coin 表
- 免转钱包（FREE_TRANSFER）：余额由三方游戏平台管理，本地不存余额，每次操作通过 API 转发给三方

**判断时机：** 每次操作前通过 `commonService.getUserInfoByUserId()` 获取用户的 walletType，走不同代码分支

**评价：**
- 优点：一套代码支持两种商户接入模式
- 缺点：addCoin 方法内 if/else 分支导致 173 行巨型方法，Transfer 和 FreeTransfer 逻辑完全不同却糅合在一起，违反单一职责原则

### 3.2 账变驱动模型

```
每一笔资金操作 = 修改 user_coin 余额 + 插入 coin_record 账变记录

操作流程（Transfer Wallet）：
1. 查用户信息 → 获取 walletType
2. 查当前余额 → user_coin 表
3. 验证余额充足（支出类操作）
4. SQL 原子更新余额 → addBalance/subBalance
5. 插入账变记录 → coin_record 表
6. MQ 通知余额变更 → 前端实时刷新

核心原则：先改钱，后记账
```

**评价：**
- "先改钱后记账"在事务内是合理的（@Transactional 保证原子性）
- 但 coin_record 表是事后写入，如果服务崩溃在"改钱"之后、"记账"之前，事务会回滚，数据一致
- 问题在于：免转钱包路径中，三方API调用成功后写本地账变记录，如果本地写入失败，三方已扣款无法回滚

### 3.3 锁-验-改-记 四步模型

对于写操作，实际执行模型是：

```
Controller 层：
  ① 获取 Redisson 分布式锁 (userId 维度)
  ② 调用 Service 方法
Service 层：
  ③ 查询当前余额（普通 SELECT，非 FOR UPDATE）
  ④ 验证余额是否充足
  ⑤ SQL 原子更新（UPDATE SET amount = amount ± X WHERE id = ? [AND amount >= X]）
  ⑥ 插入账变记录
Controller 层：
  ⑦ MQ 通知余额变更
  ⑧ 释放锁
```

**关键观察：**
- 没有 SELECT FOR UPDATE（悲观锁），完全依赖 Redis 分布式锁 + SQL WHERE 条件
- 验证（步骤④）和更新（步骤⑤）之间有时间差，依赖分布式锁保证串行
- 如果锁失效（Redisson 30s lease 到期），理论上可能并发进入，此时 SQL WHERE 是最后防线

---

## 第四章 数据模型与表结构

### 4.1 表清单

| 表名 | 用途 | 核心字段数 | 所属服务 |
|------|------|-----------|---------|
| user_coin | 用户钱包余额 | 9 | wallet-service（被 order-service 越权访问） |
| coin_record | 账变流水 | 17 | wallet-service |
| add_coin_message | MQ消息追踪 | (未深入分析) | wallet-service |

### 4.2 user_coin 表结构

```sql
CREATE TABLE user_coin (
    -- 基础字段（继承 BasicDataBaseDTO）
    id          BIGINT       PRIMARY KEY AUTO_INCREMENT,
    creator     BIGINT,
    updater     BIGINT,
    created_time DATETIME,
    updated_time DATETIME,

    -- 业务字段
    user_id      VARCHAR(64)   NOT NULL,   -- 用户ID（字符串型）
    user_name    VARCHAR(64),              -- 用户名
    amount       DECIMAL(M,N)  NOT NULL,   -- ★ 总金额（包含冻结）
    freeze_amount DECIMAL(M,N) DEFAULT 0,  -- ★ 冻结金额
    user_state   INT,                      -- 用户游戏状态
    game_type    INT,                      -- 正在进行的游戏
    wallet_type  INT           NOT NULL,   -- 钱包类型：1=转账 2=免转
    currency     VARCHAR(16),              -- 币种
    remark       VARCHAR(256)              -- 备注
);

-- 关键关系：
-- 可用余额 = amount - freeze_amount（应用层计算，非数据库字段）
-- 每个 userId 对应一条记录（单币种钱包）
```

**深度评价：**

| 设计点 | 现状 | 问题 | 我们的改进方向 |
|-------|------|------|-------------|
| 余额精度 | BigDecimal（Java类型） | 表DDL未见，精度未知 | DECIMAL(20,4) 明确 |
| 可用余额 | 应用层计算 amount - freeze_amount | 正确，不存冗余字段 | 我们用 available_amt + frozen_amt 分离存储 |
| userId 类型 | VARCHAR(64) 字符串 | 非递增主键，WHERE查询需索引 | 我们可用 INT64 |
| 单币种限制 | 每用户一条记录 | 不支持多币种 | 我们按 (user_id, wallet_type, currency_code) 多行 |
| 版本号 | 无 | 无乐观锁支持 | 我们应加 version 字段 |
| 软删除 | 无 | 钱包记录不应删除，合理 | 对齐 |
| user_state/game_type | 存在钱包表中 | 不属于钱包域，应在用户表 | 不借鉴 |

### 4.3 coin_record 表结构

```sql
CREATE TABLE coin_record (
    -- 基础字段
    id           BIGINT       PRIMARY KEY AUTO_INCREMENT,
    creator      BIGINT,
    updater      BIGINT,
    created_time DATETIME,
    updated_time DATETIME,

    -- 用户信息（冗余快照）
    user_id      VARCHAR(64)  NOT NULL,
    user_name    VARCHAR(64),
    user_type    INT,                     -- 0=正式 1=试玩
    nick_name    VARCHAR(64),
    user_belong  INT,                     -- 用户所属
    merchant_id  BIGINT,
    merchant_name VARCHAR(128),
    merchant_no  VARCHAR(64),
    agent_id     BIGINT,                  -- 代理ID
    agent_no     VARCHAR(64),             -- 代理编号
    agent_name   VARCHAR(128),            -- 代理名称
    path         VARCHAR(512),            -- 上级列表（代理链路）

    -- 账变核心
    order_no     VARCHAR(64),             -- 关联订单号
    coin_type    INT          NOT NULL,   -- 账变类型（枚举，见附录A）
    coin_detail  INT,                     -- 账变明细
    coin_value   DECIMAL(M,N) NOT NULL,   -- 变动金额（正数）
    coin_from    DECIMAL(M,N),            -- 变动前余额
    coin_to      DECIMAL(M,N),            -- 变动后余额
    coin_amount  DECIMAL(M,N),            -- ★ 当前余额（和 coin_to 含义重叠，见Bug分析）
    balance_type INT          NOT NULL,   -- 收支类型：1=收入 2=支出
    remark       VARCHAR(512)             -- 备注
);
```

**深度评价：**

| 设计点 | 现状 | 评价 |
|-------|------|------|
| 用户信息冗余 | 8个字段 (userName/nickName/merchantId/merchantNo/merchantName/agentId/agentNo/agentName/path/userBelong/userType) | 过度冗余。快照用户名、商户名合理，但代理链路 path 等不应每条账变都复制 |
| coin_from/coin_to | 记录变动前后余额 | 好的设计——审计必备，可追溯资金链 |
| coin_amount | 和 coin_to 含义重叠 | Bug：两者在不同代码路径被赋不同值，语义混乱（详见Bug章节） |
| 无唯一约束 | order_no 无 UNIQUE 索引 | 无法防止重复账变（幂等性缺失） |
| 无 currency 字段 | 缺少币种字段 | 假设单币种，但 user_coin 有 currency 字段，不一致 |

### 4.4 ER 关系图

```
┌──────────────────┐         ┌──────────────────────┐
│    user_coin     │         │     coin_record       │
│                  │   1:N   │                        │
│  id (PK)         │────────→│  id (PK)               │
│  user_id (UK?)   │         │  user_id (FK 逻辑)     │
│  amount          │         │  order_no              │
│  freeze_amount   │         │  coin_type             │
│  wallet_type     │         │  coin_value            │
│  currency        │         │  coin_from / coin_to   │
└──────────────────┘         │  balance_type          │
                              └──────────────────────┘

                              ┌──────────────────────┐
                              │  add_coin_message     │
                              │  (MQ追踪表)            │
                              │  id (PK)               │
                              │  ... (未深入分析)       │
                              └──────────────────────┘
```

---

## 第五章 余额模型深度解析

### 5.1 余额计算公式

```
user_coin.amount = 总金额（包含被冻结的部分）
user_coin.freeze_amount = 已冻结金额

可用余额 = amount - freeze_amount
```

**这是一个关键的设计选择。** 有两种余额模型：

| 模型 | 公式 | 参考项目 | 我们的设计 |
|------|------|---------|----------|
| 总额包含冻结 | available = amount - freeze | 本项目 | 否 |
| 可用与冻结独立 | total = available + frozen | 我们的方案 | 是 |

### 5.2 各操作对余额的影响

#### 5.2.1 正常收入（如结算加钱）

```
操作前：amount=100, freeze_amount=0
操作：addBalance(+50)
操作后：amount=150, freeze_amount=0
可用：150
```

SQL: `UPDATE user_coin SET amount = amount + 50 WHERE id = ?`

#### 5.2.2 正常支出（如下注扣钱）

```
操作前：amount=100, freeze_amount=0
操作：subBalance(-30)
操作后：amount=70, freeze_amount=0
可用：70
```

SQL: `UPDATE user_coin SET amount = amount - 30 WHERE id = ? AND (amount - 30) >= 0`

#### 5.2.3 冻结

```
操作前：amount=100, freeze_amount=0
操作：addFreezeBalance(+20)
操作后：amount=100, freeze_amount=20
可用：80（应用层计算）
```

SQL: `UPDATE user_coin SET freeze_amount = freeze_amount + 20 WHERE id = ?`

**关键：冻结不改 amount！** 冻结只是"标记"一部分金额不可用。

#### 5.2.4 解冻扣款（确认扣除冻结资金）

```
操作前：amount=100, freeze_amount=20
操作：subFreezeBalance(-20)
操作后：amount=80, freeze_amount=0
可用：80
```

SQL: `UPDATE user_coin SET amount = amount - 20, freeze_amount = freeze_amount - 20 WHERE id = ?`

**关键：两个字段同时减少！** amount 减少表示真正扣钱，freeze_amount 减少表示释放冻结标记。

#### 5.2.5 回滚冻结（取消冻结，资金恢复可用）

```
操作前：amount=100, freeze_amount=20
操作：rollbackFreezeBalance(-20)
操作后：amount=100, freeze_amount=0
可用：100
```

SQL: `UPDATE user_coin SET freeze_amount = freeze_amount - 20 WHERE id = ?`

**关键：只减 freeze_amount，不动 amount！** 因为冻结时没加 amount，回滚也不需减。

### 5.3 余额模型的优缺点对比

**参考项目模型（总额包含冻结）：**

```
优点：
- amount 就是用户的"资产总额"，语义清晰
- 冻结操作只改一个字段，简单
- 解冻扣款两字段同减，原子性好

缺点：
- 可用余额需要计算 (amount - freeze_amount)
- 查可用余额时必须 SELECT 两个字段
- 如果有并发写 freeze_amount 和 amount 的场景，需要格外小心
```

**我们的模型（可用与冻结独立）：**

```
available_amt = 可用余额（直接存储）
frozen_amt = 冻结余额（直接存储）

优点：
- available_amt 直接可读，查询不需计算
- 冻结操作语义更直观：从 available 转移到 frozen
- 余额校验简单：WHERE available_amt >= X

缺点：
- 冻结操作需同时改两个字段（available -= X, frozen += X）
- 需确保两字段同事务内一起更新
```

**取舍结论：我们的模型更优。** 原因：
1. 查询是高频操作，直接读 available_amt 比计算 amount - freeze_amount 高效
2. 冻结操作是低频操作（仅提现），多改一个字段代价小
3. 我们有 version 乐观锁，不怕并发更新

---

## 第六章 核心服务逐方法解剖

### 6.1 CoinRecordServiceImpl 方法清单

文件：`p9-services/wallet-service/src/main/java/com/maya/p9/service/impl/CoinRecordServiceImpl.java`

共 522 行，15 个方法：

| # | 方法名 | 行数 | 事务 | 功能 | 复杂度 |
|---|--------|------|------|------|--------|
| 1 | queryCoinRecord | 76-93 | 无 | 查询账变列表 | 低 |
| 2 | queryCoinRecordByOrderNo | 96-108 | 无 | 按订单号查账变 | 低 |
| 3 | deductCoin | 111-149 | @Transactional | ★ 只写账变记录（不改余额） | 中 |
| 4 | sendUserBalance | 152-165 | 无 | MQ通知余额变更（fanout交换机） | 低 |
| 5 | **addCoin** | **169-341** | **@Transactional** | **★★★ 核心方法，173行** | **极高** |
| 6 | setVOToUserInfo | 343-351 | 无 | 工具方法：复制用户信息 | 极低 |
| 7 | checkCoinChange | 353-360 | 无 | 新用户不能首笔支出 | 低 |
| 8 | insufficientBalance | 366-384 | 无 | 余额不足校验 | 中 |
| 9 | addCoinBatch | 388-396 | @Transactional | 批量 addCoin | 低（循环调用 addCoin） |
| 10 | deductCoinBatch | 400-408 | @Transactional | 批量 deductCoin | 低（循环调用 deductCoin） |
| 11 | changeCoinAndNoticeBalance | 412-421 | @Transactional | 批量 deductCoin + 通知余额 | 低 |
| 12 | freezeCoin | 425-446 | @Transactional | 冻结金额 | 中 |
| 13 | subFreezeBalance | 449-473 | **无 ★BUG** | 解冻扣款 + 写账变 | 高 |
| 14 | rollbackFreezeBalance | 476-493 | **无 ★BUG** | 回滚冻结 | 中 |
| 15 | createOrder | 495-519 | 无 | 创建账变记录（subFreezeBalance的子方法） | 中 |

### 6.2 addCoin 方法深度解剖（行 169-341，核心中的核心）

这是整个钱包系统最重要的方法，173 行代码，承担了所有资金变动逻辑。

```java
@Transactional(rollbackFor = Exception.class)
public boolean addCoin(CoinRecordVO coinRecordVO) {
```

**执行流程图：**

```
addCoin(coinRecordVO)
│
├── [1] 参数校验：coinType 必须 > 0 (行 171-175)
│
├── [2] 查用户信息：commonService.getUserInfoByUserId() (行 176)
│       → Feign 调用 user-service
│
├── [3] 复制用户信息到 DTO (行 177-179)
│
├── [4] 判断钱包类型
│   │
│   ├── walletType == TRANSFER（转账钱包）(行 182-230)
│   │   │
│   │   ├── [4a] 查当前余额：userCoinRepository.selectOne() (行 184-186)
│   │   │
│   │   ├── [4b] 如果用户存在（userCoinDTO1 != null）(行 188-210)
│   │   │   ├── 验证余额：insufficientBalance() (行 190)
│   │   │   ├── 收入路径：userCoinRepository.addBalance() (行 198)
│   │   │   │   └── coinTo = 当前余额 + 变动金额 (行 199)
│   │   │   ├── 支出路径：userCoinRepository.subBalance() (行 202)
│   │   │   │   ├── 返回 0 → 余额不足，抛异常 (行 203-206)
│   │   │   │   └── coinTo = 当前余额 - 变动金额 (行 207)
│   │   │   └── 设置 coinTo, coinAmount, coinFrom (行 209-211)
│   │   │
│   │   ├── [4c] 如果用户不存在（首次创建钱包）(行 212-227)
│   │   │   ├── checkCoinChange() 首笔必须是收入 (行 214)
│   │   │   ├── INSERT user_coin (行 222)
│   │   │   └── 设置 coinTo, coinAmount, coinFrom (行 224-226)
│   │   │
│   │   └── [4d] INSERT coin_record (行 228-230)
│   │
│   └── walletType == FREE_TRANSFER（免转钱包）(行 231-338)
│       │
│       ├── [4e] 获取订单信息 (行 241-246)
│       │   └── Feign 调用 order-service
│       ├── [4f] 获取商户详情 (行 248)
│       │   └── Feign 调用 merchant-service
│       ├── [4g] 构造三方请求 (行 251-307)
│       │   └── 包装 ThirdOrderRequestVO
│       ├── [4h] 调用三方API (行 309)
│       │   └── apiResource.amountTransfer()
│       ├── [4i] 处理三方响应 (行 311-331)
│       │   ├── SUCCESS → 记录 + 通知余额 + 写账变
│       │   └── FAIL → 记录失败 + return false
│       └── [4j] INSERT coin_record (行 332-337)
│
└── return true (行 340)
```

### 6.3 addCoin 关键代码片段精析

#### 片段1：余额验证逻辑（insufficientBalance，行 366-384）

```java
private void insufficientBalance(CoinRecordVO coinRecordVO, UserCoinDTO userCoin) {
    // 只对 转出/下注/打赏 验证余额
    if (WalletUtilEnum.CoinChangeEnum.TRANSFER_OUT.getCode().equals(coinRecordVO.getCoinType())
            || WalletUtilEnum.CoinChangeEnum.BET.getCode().equals(coinRecordVO.getCoinType())
            || WalletUtilEnum.CoinChangeEnum.SEND_REWARD.getCode().equals(coinRecordVO.getCoinType())) {
        BigDecimal amount = coinRecordVO.getCoinValue();
        BigDecimal userCoinAmount = userCoin.getAmount();
        BigDecimal freezeAmount = userCoin.getFreezeAmount();
        BigDecimal balance = userCoinAmount.subtract(freezeAmount); // 可用余额
        if (balance.compareTo(amount) < 0) {
            throw new MayaDefaultException(ErrorCode.AMOUNT_NOT_ENOUGH);
        }
    }
}
```

**分析：**
- 只验证三种场景：转出(2)、下注(3)、打赏(8)
- 不验证的场景：结算(4)、取消(5)、结算回滚(6)、重算(7) → 这些可以让余额变负（业务合理：取消和回滚必须执行）
- 可用余额 = amount - freezeAmount（应用层计算）
- **隐患**：此时查到的 amount/freezeAmount 是查询时的快照，到 SQL 执行时可能已变化。依赖外层 Redis 锁保证串行

#### 片段2：coinTo 和 coinAmount 的赋值（行 199-211）

```java
// 收入路径
coinTo = userCoinDTO1.getAmount().add(coinRecordVO.getCoinValue());
// ...
coinRecordDTO.setCoinTo(coinTo);
coinRecordDTO.setCoinAmount(coinRecordDTO.getCoinTo());  // coinAmount = coinTo
coinRecordDTO.setCoinFrom(userCoinDTO1.getAmount());      // coinFrom = 操作前余额
```

**分析：**
- coinFrom = 操作前的 amount
- coinTo = 操作后的 amount（应用层计算，非数据库返回）
- coinAmount = coinTo（两者相同，存在冗余）
- **隐患**：coinFrom/coinTo 基于查询快照计算，而非 SQL 更新后的真实值。如果锁失效，快照可能不准

#### 片段3：免转钱包的 null bug（行 316-318）

```java
case SUCCESS:
    coinRecordDTO.setCoinTo(null);     // 设置为 null
    coinRecordDTO.setCoinFrom(null);   // 设置为 null
    coinRecordDTO.setCoinAmount(coinRecordDTO.getCoinTo()); // ★ getCoinTo() 是 null！
```

**这是一个确认的 Bug：** coinAmount 被赋值为 null。
- 免转钱包路径中，因为余额在三方，本地不知道变动前后余额，所以 coinTo/coinFrom 设为 null
- 但 coinAmount 也被设为 null 了，如果后续有依赖 coinAmount 的逻辑，会 NPE
- **幸运的是**：coin_record 表的 coin_amount 允许 NULL，所以数据库不报错，但语义上不正确

### 6.4 deductCoin 方法解析（行 111-149）

**名称误导：** 方法名 "deductCoin"（扣币），但实际上**只写账变记录，不修改余额**。

```java
@Transactional(rollbackFor = Throwable.class)
public boolean deductCoin(CoinRecordVO coinRecordVO) {
    // 1. 金额必须 > 0
    // 2. 查用户信息
    // 3. 复制用户信息到 DTO
    // 4. 免转钱包时清除 coinFrom/coinTo
    // 5. INSERT coin_record
    // 6. return true
    // ★ 注意：没有任何 UPDATE user_coin 的操作！
}
```

**为什么只写记录不改余额？**
因为 deductCoin 的调用方是 order-service 通过 MQ 发来的消息。order-service 已经**自己直接在数据库**改了 user_coin 的余额（共享数据库反模式），wallet-service 的 deductCoin 只负责补写账变记录。

这是架构设计的核心缺陷之一，后面章节详细分析。

### 6.5 freezeCoin / subFreezeBalance / rollbackFreezeBalance 三阶段

#### freezeCoin（冻结）—— 行 425-446

```java
@Transactional
public ErrorCode freezeCoin(CoinFreezeVO coinFreezeVO) {
    // 1. 查 user_coin
    // 2. 检查 amount >= freezeAmount（注意：没有检查 amount - 现有freeze >= 新冻结金额）
    // 3. addFreezeBalance（freeze_amount += X）
    return ErrorCode.SUCCESS;
}
```

**Bug：** 余额检查逻辑错误！
- 代码检查：`userCoinDTO.getAmount().compareTo(coinFreezeVO.getFreezeAmount()) < 0`
- 含义：总金额 >= 本次冻结金额
- 正确逻辑应该是：`(amount - 已有freeze_amount) >= 本次冻结金额`
- 如果用户已有冻结金额，这个检查会放过余额不足的情况
- 例：amount=100, 已冻结60, 再冻结50 → 检查 100 >= 50 通过，但实际可用只有 40

#### subFreezeBalance（解冻扣款）—— 行 449-473

```java
// ★ 没有 @Transactional！
public ErrorCode subFreezeBalance(CoinFreezeRecordVO coinFreezeRecordVO) {
    // 1. 查 user_coin
    // 2. 检查 freeze_amount >= 本次解冻金额
    // 3. subFreezeBalance SQL（amount -= X, freeze_amount -= X）
    // 4. createOrder（写账变记录）
    return ErrorCode.SUCCESS;
}
```

**严重 Bug：缺少 @Transactional。** 步骤 3 和步骤 4 不在同一事务中。如果步骤 3 成功但步骤 4 失败：
- 余额已被扣减
- 但没有账变记录
- 资金"凭空消失"

#### rollbackFreezeBalance（回滚冻结）—— 行 476-493

```java
// ★ 没有 @Transactional！
public ErrorCode rollbackFreezeBalance(CoinFreezeVO coinFreezeVO) {
    // 1. 查 user_coin
    // 2. rollbackFreezeBalance SQL（freeze_amount -= X）
    return ErrorCode.SUCCESS;
}
```

**缺少 @Transactional。** 虽然只有一个 SQL 操作（默认自动提交），但如果后续增加逻辑（如写回滚记录），会有事务风险。

### 6.6 UserCoinServiceImpl 逻辑

```java
// 简化后的核心逻辑
public UserCoinVO getWallet(UserCoinVO userCoinVO) {
    UserInfoVO userInfoVO = commonService.getUserInfoByUserId(userCoinVO.getUserId());

    if (walletType == TRANSFER) {
        UserCoinDTO dto = userCoinRepository.selectOne(...);
        if (dto != null) {
            // 返回可用余额 = amount - freezeAmount
            userCoinVO.setAmount(dto.getAmount().subtract(dto.getFreezeAmount()));

            // 特殊逻辑：API用户 + 非正式玩家 → 余额强制返回0
            if (md5Sign != null && userType != FORMAL_PLAY) {
                userCoinVO.setAmount(BigDecimal.ZERO);
            }
        } else {
            userCoinVO.setAmount(BigDecimal.ZERO);
        }
    } else {
        // 免转钱包：余额在三方，本地返回0
        userCoinVO.setAmount(BigDecimal.ZERO);
    }
}
```

**观察：**
- 试玩用户通过 API 查余额时返回 0，防止试玩资金泄露
- 免转钱包本地不存余额，返回 0（实际余额由前端直接从三方获取）

---

## 第七章 并发控制体系

### 7.1 锁架构总览

```
┌──────────────────────────────────────────────────────┐
│                   并发控制层级                          │
│                                                        │
│  第一层：Redisson 分布式锁                              │
│    ├── 锁粒度：userId                                  │
│    ├── 锁Key：add.coin.lock.key.{userId}              │
│    ├── 等待时间：20000ms (20秒)                         │
│    ├── 持有时间：30000ms (30秒，租约)                    │
│    ├── 持有位置：Controller 层 + MQ Listener #1        │
│    └── ★ 缺失位置：MQ Listener #2/#3/#4               │
│                                                        │
│  第二层：SQL WHERE 条件                                 │
│    └── subBalance: AND (amount - #{amount}) >= 0      │
│        → 扣款时防止余额为负（最后防线）                   │
│                                                        │
│  ★ 缺失层：SELECT FOR UPDATE（完全没有）                │
│  ★ 缺失层：乐观锁 version（完全没有）                    │
└──────────────────────────────────────────────────────┘
```

### 7.2 Redisson 锁实现模式

所有写操作端点的锁模式完全一致（以 addCoin 为例）：

```java
// WalletController.java, 行 60-78
RLock rLock = redisUtil.getLock(RedisLockKey.getAddCoinLockKey(coinRecordVO.getUserId()));
try {
    if (rLock.tryLock(20000, 30000L, TimeUnit.MILLISECONDS)) {
        boolean result = coinRecordService.addCoin(coinRecordVO);
        if (result) {
            coinRecordService.sendUserBalance(coinRecordVO.getUserId());
            return ResponseUtil.success();
        }
    } else {
        log.info("锁超时：{}", coinRecordVO.getOrderNo());
    }
} catch (Exception e) {
    log.error(e.getMessage(), e);
} finally {
    if (Objects.nonNull(rLock) && rLock.isLocked() && rLock.isHeldByCurrentThread()) {
        rLock.unlock();
        log.info("锁释放{}", coinRecordVO.getOrderNo());
    }
}
return ResponseUtil.fail(ErrorCode.FAIL);
```

**锁参数解析：**

| 参数 | 值 | 含义 |
|------|-----|------|
| tryLock waitTime | 20000ms | 获取锁的最大等待时间，超过 20s 放弃 |
| tryLock leaseTime | 30000ms | 锁自动释放时间，防止持有者崩溃后死锁 |
| 锁Key模式 | add.coin.lock.key.{userId} | 同一用户的所有钱包操作串行 |

**锁释放三重检查：**
```java
if (Objects.nonNull(rLock)           // 1. rLock 对象不为 null
    && rLock.isLocked()              // 2. 锁确实被持有
    && rLock.isHeldByCurrentThread() // 3. 且是当前线程持有
) {
    rLock.unlock();
}
```

这个三重检查模式是**值得借鉴的**：
- 防止释放别人的锁（线程 A 的锁超时释放后，线程 B 获取了锁，线程 A 不能释放线程 B 的锁）
- 防止重复释放

### 7.3 锁覆盖范围分析

| 入口 | 有锁？ | 锁位置 | 操作 |
|------|--------|--------|------|
| POST /addCoin | 有 | Controller | addCoin (改余额+写记录) |
| POST /addCoinList | 有 | Controller | addCoinBatch |
| POST /freezeCoin | 有 | Controller | freezeCoin |
| POST /subFreezeBalance | 有 | Controller | subFreezeBalance |
| POST /rollbackFreezeBalance | 有 | Controller | rollbackFreezeBalance |
| POST /getWallet | **无** | - | 读操作，合理不加锁 |
| POST /queryCoinRecord | **无** | - | 读操作，合理不加锁 |
| MQ Listener #1 (ADD_COIN_QUEUE) | **有** | Listener | addCoin |
| MQ Listener #2 (ADD_COIN_ORDER_QUEUE) | **无 ★危险** | - | deductCoinBatch |
| MQ Listener #3 (ADD_COIN_NOT_NOTICE_BALANCE_QUEUE) | **无 ★危险** | - | deductCoinBatch |
| MQ Listener #4 (ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE) | **无 ★危险** | - | changeCoinAndNoticeBalance |
| order-service 直接 SQL | **无 ★极危险** | - | subBalance/addBalance |

### 7.4 并发安全漏洞清单

**漏洞1：MQ Listener #2/#3 无锁操作**

```java
// AddCoinListener.java, 行 70-100
@RabbitListener(queues = {MqConstants.ADD_COIN_ORDER_QUEUE})
public void noticeDeductCoinOrderMessage(String jsonStr) {
    // ★ 没有获取 Redisson 锁！
    List<CoinRecordVO> coinRecordVOList = JSON.parseArray(jsonStr, CoinRecordVO.class);
    boolean result = coinRecordService.deductCoinBatch(coinRecordVOList);
    // ...
}
```

虽然 deductCoinBatch 内部只写 coin_record 记录不改余额（因为 order-service 已经改了），但在同一时刻，Controller 层的 addCoin 可能也在操作同一用户的余额，形成竞态。

**漏洞2：order-service 绕过钱包服务直接写表**

```java
// order-service 的 UserCoinMapper.xml
// ★ 和 wallet-service 操作同一张 user_coin 表，但完全没有分布式锁！
<update id="subBalance">
    update user_coin b
    set b.amount = b.amount - #{vo.amount},
        b.updated_time = #{vo.updateTime}
    where b.user_id = #{vo.id}
      AND (b.amount - #{vo.amount}) >= 0
</update>
```

这意味着：
- wallet-service 用 Redis 锁保护 user_coin 操作
- order-service 完全不用锁直接操作同一张表
- 两个服务的并发操作可能导致数据不一致

**漏洞3：锁在 Controller 而非 Service**

如果有人直接调用 Service 层方法（如通过 Spring 内部注入），会绕过锁。虽然当前代码没有这种情况，但架构上不安全。

### 7.5 对我们的启示

| 参考项目做法 | 我们的改进 | 原因 |
|-------------|-----------|------|
| 锁在 Controller | 锁在 Service 层 | 确保所有调用路径都有锁保护 |
| 只有 Redis 锁 | Redis 锁 + DB FOR UPDATE + SQL WHERE | 三层防线 |
| 无乐观锁 | version 乐观锁 | 额外安全保障 |
| MQ消费无锁 | MQ消费也加锁 | 防止并发竞态 |
| 共享数据库 | 微服务间只通过 RPC 交互 | 杜绝数据越权 |

---

## 第八章 SQL原子操作分析

### 8.1 五个自定义SQL操作

全部定义在 `UserCoinMapper.xml`：

#### 8.1.1 subBalance（扣款）

```xml
<update id="subBalance">
    update user_coin b
    set b.amount = b.amount - #{vo.amount},
        b.updated_time = #{vo.updateTime}
    where b.id = #{vo.id}
      <if test=" vo.coinType != 6 and vo.coinType != 5 ">
          AND (b.amount - #{vo.amount}) >= 0
      </if>
</update>
```

**精析：**
- `amount = amount - X`：原子减法，不依赖应用层先读后写
- `WHERE id = #{vo.id}`：主键定位，效率最高
- `AND (b.amount - #{vo.amount}) >= 0`：条件扣款，防止余额为负
- **特殊豁免**：coinType=5（取消）和 coinType=6（结算回滚）不检查余额 → 允许余额变负
  - 业务合理：取消和回滚是系统发起的补偿操作，必须执行
- **返回值**：受影响行数（0=余额不足或不存在，1=成功）
- **调用方检查**：`if (i == 0) throw new MayaDefaultException(ErrorCode.AMOUNT_NOT_ENOUGH)`

**评价：优秀的设计。** SQL 条件扣款是防止超扣的最后防线，即使应用层校验被绕过也安全。

#### 8.1.2 addBalance（加款）

```xml
<update id="addBalance">
    update user_coin b
    set b.amount = b.amount + #{vo.amount},
        b.updated_time = #{vo.updateTime}
    where b.id = #{vo.id}
</update>
```

**精析：**
- 无条件加款，amount 可以无限增加
- 没有上限检查 → 如果有 bug 导致重复加款，不会被拦截
- **缺失**：应该有 version 乐观锁或其他防重机制

#### 8.1.3 addFreezeBalance（增加冻结）

```xml
<update id="addFreezeBalance">
    update user_coin b
    set b.freeze_amount = b.freeze_amount + #{vo.amount},
        b.updated_time = #{vo.updateTime}
    where b.id = #{vo.id}
</update>
```

**精析：**
- 只增加 freeze_amount，不动 amount
- 没有检查 `amount - freeze_amount >= vo.amount`（可用余额是否足够冻结）
- **与应用层的 bug 呼应**：freezeCoin 方法的余额检查也是错的

#### 8.1.4 subFreezeBalance（解冻扣款）

```xml
<update id="subFreezeBalance">
    update user_coin b
    set b.amount = b.amount - #{vo.amount},
        b.freeze_amount = b.freeze_amount - #{vo.amount},
        b.updated_time = #{vo.updateTime}
    where b.id = #{vo.id}
</update>
```

**精析：**
- **两个字段同时减少**：amount 和 freeze_amount 都减 X
- 语义：扣除之前冻结的金额 → 总额减少，冻结额减少，可用余额不变
- **没有条件检查**：不检查 freeze_amount >= X，如果传入的金额大于实际冻结额，freeze_amount 会变负
- **没有 @Transactional 保护**（在 Service 层）

#### 8.1.5 rollbackFreezeBalance（回滚冻结）

```xml
<update id="rollbackFreezeBalance">
    update user_coin b
    set b.freeze_amount = b.freeze_amount - #{vo.amount},
        b.updated_time = #{vo.updateTime}
    where b.id = #{vo.id}
</update>
```

**精析：**
- 只减 freeze_amount，不动 amount
- 语义：取消冻结，资金恢复可用
- **同样没有条件检查**

### 8.2 order-service 的 SQL（越权操作）

```xml
<!-- order-service/src/main/resources/mapper/UserCoinMapper.xml -->
<update id="subBalance">
    update user_coin b
    set b.amount = b.amount - #{vo.amount},
        b.updated_time = #{vo.updateTime}
    where b.user_id = #{vo.id}
      AND (b.amount - #{vo.amount}) >= 0
</update>

<update id="addBalance">
    update user_coin b
    set b.amount = b.amount + #{vo.amount},
        b.updated_time = #{vo.updateTime}
    where b.user_id = #{vo.id}
</update>
```

**关键差异：**
- wallet-service 用 `WHERE b.id = #{vo.id}`（主键）
- order-service 用 `WHERE b.user_id = #{vo.id}`（业务键）
- 两个服务的 SQL 不完全一致，增加了维护风险

### 8.3 SQL 设计总评

| 维度 | 评价 | 评分 |
|------|------|------|
| 原子扣款 | 优秀：`amount = amount - X WHERE (amount - X) >= 0` | 9/10 |
| 特殊场景豁免 | 合理：取消/回滚允许负余额 | 8/10 |
| 冻结检查 | 缺失：addFreezeBalance 不检查可用余额 | 4/10 |
| 乐观锁 | 完全缺失：无 version 字段 | 2/10 |
| 返回值利用 | 部分利用：subBalance 检查返回值，其他不检查 | 5/10 |
| 跨服务一致 | 严重问题：两个服务的 SQL WHERE 条件不同 | 2/10 |

---

## 第九章 MQ消息架构

### 9.1 MQ 拓扑

```
                         RabbitMQ
                            │
        ┌───────────────────┼──────────────────────┐
        │                   │                       │
   ┌────▼─────┐     ┌──────▼───────┐     ┌────────▼────────┐
   │ 直连队列   │     │ 交换机绑定     │     │ 通知型交换机     │
   └────┬─────┘     └──────┬───────┘     └────────┬────────┘
        │                   │                       │
    4个消费队列          1个入口交换机           2个通知交换机
```

### 9.2 四个消费队列详解

| 队列名 | 常量 | 生产者 | 消费方法 | 有锁？ | 用途 |
|--------|------|--------|---------|--------|------|
| wallet_add_coin_queue | ADD_COIN_QUEUE | 结算服务等 | NoticeAddCoinMessage | 有 | 单笔加减币（结算/打赏） |
| wallet_add_coin_order_queue | ADD_COIN_ORDER_QUEUE | order-service | noticeDeductCoinOrderMessage | **无** | 批量写账变记录（下注扣款后补记录） |
| wallet_add_coin_not_notice_balance_queue | ADD_COIN_NOT_NOTICE_BALANCE_QUEUE | order-service | addCoinNotNoticeBalance | **无** | 写账变但不通知余额（防止客户端余额跳跃） |
| add_coin_record_and_notice_balance_queue | ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE | order-service | addCoinRecordAndNoticeBalanceQueue | **无** | 写账变+通知余额 |

### 9.3 消费逻辑差异对比

#### 队列1：ADD_COIN_QUEUE（最完整）

```
消息 → 反序列化 CoinRecordVO → 校验金额 → 获取 Redisson 锁 → addCoin()
→ 成功则 sendUserBalance() → 释放锁
```

特点：有锁保护，调用完整的 addCoin（改余额 + 写记录）

#### 队列2：ADD_COIN_ORDER_QUEUE（有缺陷）

```
消息 → 反序列化 List<CoinRecordVO> → deductCoinBatch() → sendUserBalance()
失败处理 → orderResource.updateOrdersToInitialStatus(orderNos)
```

特点：
- **无锁**
- 调用 deductCoinBatch，内部循环调用 deductCoin（只写记录不改余额）
- 失败时通过 Feign 回滚订单状态
- 成功后通知余额变更

#### 队列3：ADD_COIN_NOT_NOTICE_BALANCE_QUEUE（静默写记录）

```
消息 → 反序列化 List<CoinRecordVO> → deductCoinBatch()
失败处理 → orderResource.updateOrdersToInitialStatus(orderNos)
```

特点：
- **无锁**
- 和队列2几乎一样，但不通知余额变更
- 注释说明：避免客户端余额跳跃（先扣款后结算的场景，中间状态不通知）

#### 队列4：ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE

```
消息 → 反序列化 List<CoinRecordVO> → changeCoinAndNoticeBalance()
```

特点：
- **无锁**
- changeCoinAndNoticeBalance 内部循环调用 deductCoin + sendUserBalance
- **异常处理有 bug**：`log.info("下注帐变写入异常:{} 注单:{}", coinRecordVOList)` 少了异常参数

### 9.4 两种余额通知机制

```
方式1：直连队列通知
    rabbitTemplate.convertAndSend(
        MqConstants.NOTICE_USER_BALANCE_CHANGE,    // queue name
        JSON.toJSONString(userBalanceUpdateVO)
    );
    → 只有一个消费者能收到

方式2：Fanout 交换机广播
    rabbitTemplate.convertAndSend(
        MqConstants.EXCHANGE_NOTICE_USER_BALANCE_CHANGE,  // exchange name
        "",                                                 // routing key (fanout不需要)
        message
    );
    → 所有绑定的消费者都能收到（每台 websocket-service 实例）
```

**使用场景：**
- 方式1（直连队列）：在免转钱包的 addCoin 中使用 → 通知一台 ws 即可
- 方式2（Fanout 交换机）：在 sendUserBalance 中使用 → 广播给所有 ws 实例
- **不一致性**：两种方式混用，语义不同但都叫"通知余额变更"，容易混淆

### 9.5 MQ 设计评价

| 维度 | 评价 |
|------|------|
| 消息可靠性 | 配置了 publisher-confirm-type: CORRELATED，但代码没实现 ConfirmCallback |
| 消费幂等性 | 完全没有：相同消息重复消费会重复写账变记录 |
| 消费顺序性 | 没保证：同一用户的多笔操作可能乱序消费 |
| 死信处理 | 没有配置 DLQ（死信队列） |
| 重试机制 | 异常后不重试，直接 log.error 然后回滚订单 |
| 消息体设计 | 用 JSON 字符串，不是强类型 schema |

---

## 第十章 接口设计与API清单

### 10.1 REST API（WalletController）

基路径：`/webapi/wallet`，端口：8907

| # | 方法 | 路径 | 有锁 | 功能 | 请求体 | 响应 |
|---|------|------|------|------|--------|------|
| 1 | POST | /addCoin | 有 | 加减币（核心） | CoinRecordVO | ResponseVO |
| 2 | POST | /addCoinList | 有 | 批量加减币 | List\<CoinRecordVO\> | ResponseVO |
| 3 | POST | /queryCoinRecordByOrderNo | 无 | 按订单号查账变 | @RequestParam orderNo | ResponseVO\<CoinRecordVO\> |
| 4 | POST | /queryCoinRecord | 无 | 查账变列表 | CoinRecordVO | ResponseVO |
| 5 | POST | /getWallet | 无 | 查钱包余额 | UserCoinVO | ResponseVO\<UserCoinVO\> |
| 6 | POST | /users/wallet/list | 无 | 批量查钱包 | QueryWalletReqVO | ResponseVO\<List\<UserCoinVO\>\> |
| 7 | POST | /freezeCoin | 有 | 冻结金额 | CoinFreezeVO | ResponseVO |
| 8 | POST | /subFreezeBalance | 有 | 解冻扣款 | CoinFreezeRecordVO | ResponseVO |
| 9 | POST | /rollbackFreezeBalance | 有 | 回滚冻结 | CoinFreezeVO | ResponseVO |

### 10.2 接口设计问题

**问题1：全部用 POST**
- 查询操作（getWallet, queryCoinRecord 等）应该用 GET
- RESTful 最佳实践：查询用 GET，变更用 POST/PUT
- POST 用于查询会导致：不能被浏览器缓存、不能被 CDN 缓存、语义不清

**问题2：路径设计不 RESTful**
- `/addCoin` → 应该是 `POST /coins` 或 `POST /transactions`
- `/getWallet` → 应该是 `GET /wallet/{userId}`
- `/queryCoinRecord` → 应该是 `GET /coin-records?userId=xxx`

**问题3：响应类型不统一**
- addCoin 返回 `ResponseEntity<ResponseVO>`（双层包装）
- freezeCoin 返回 `ResponseVO<?>`（单层）
- 混用增加了调用方的适配成本

**问题4：缺少版本号**
- 路径没有版本前缀（如 /v1/webapi/wallet）
- 后续接口变更无法平滑过渡

**问题5：缺少幂等性设计**
- addCoin 没有幂等 key（如 requestId / idempotencyKey）
- 网络重试可能导致重复操作

### 10.3 我们的接口设计原则（对照改进）

```
我们的 Go ser-wallet 接口设计：
1. 写操作用 POST，读操作用 Kitex RPC（不走 HTTP REST）
2. 每个写操作必须有 idempotency_key 参数
3. 统一返回 ret.BizErr 错误码
4. Thrift IDL 定义强类型接口
5. 路径带版本号 /api/v1/wallet/...
```

---

## 第十一章 跨服务依赖与调用链路

### 11.1 wallet-service 的出站依赖（Feign 调用其他服务）

```
wallet-service
│
├──→ user-service (UserService.java)
│    └── GET /user/v1/getUserByUserId
│        → 获取用户信息（walletType, merchantId, 等）
│        → 每次操作都调用（未缓存 ★性能问题）
│
├──→ order-service (OrderResource.java)
│    ├── POST /order/bet/updateOrdersToInitialStatus
│    │   → MQ消费失败时回滚订单状态
│    ├── GET /order/v1/findOrderByOrderNo
│    │   → 免转钱包路径获取订单信息
│    └── POST /order/thirdOrderRecord/saveThirdOrderRecord
│        → 免转钱包保存三方订单记录
│
├──→ api-service (ApiResource.java)
│    └── POST /game/api/amountTransfer
│        → 免转钱包调用三方加减款
│
└──→ merchant-service (MerchantFeignResource.java)
     └── POST /adminApi/merchant/getMerchantDetail
         → 免转钱包获取商户详情（三方API地址等）
```

### 11.2 wallet-service 的入站调用（被其他服务调用）

```
调用方                        端点                              用途
───────────────────────────────────────────────────────────────────────
websocket-service (15端点)
  ├── getWallet()             POST /webapi/wallet/getWallet      查余额
  ├── getWalletListByUserId() POST /users/wallet/list            批量查余额
  ├── addCoin()               POST /webapi/wallet/addCoin        打赏等
  ├── userCoinRecord()        POST /webapi/wallet/userCoinRecordPage   账变分页
  ├── userCoinRecordList()    POST /webapi/wallet/userCoinRecordList   账变列表
  ├── userWithdrawApply()     POST /userWithdraw/userWithdrawApply     提现申请
  ├── queryUserRechargingInfo() POST /userDeposit/queryUserRechargingInfo 充值信息
  ├── createOrder()           POST /userDeposit/createOrder             充值订单
  ├── userWithdrawConfigInfo() POST /userWithdraw/userWithdrawConfigInfo 提现配置
  ├── userRecharge()          POST /userOfflineDeposit/userRecharge     线下充值
  ├── uploadVoucher()         POST /userOfflineDeposit/uploadVoucher    上传凭证
  ├── cancelDepositOrder()    POST /userOfflineDeposit/cancelDepositOrder 取消充值
  ├── depositOrderDetail()    POST /userOfflineDeposit/depositOrderDetail 充值详情
  ├── getOfflineCollectionList() POST /offlineReceived/getOfflineCollectionList 收款列表
  └── getQuickAmountList*()   POST /offlineReceived/getQuickAmountList*  快捷金额

common-service (3端点)
  ├── addCoin()               POST /webapi/wallet/addCoin        加减币
  ├── getWallet()             POST /webapi/wallet/getWallet      查余额
  └── addWithdrawLimitAmount() POST /userWithdraw/addWithdrawLimitAmount 提现限额

api-service (5端点)
  ├── getWallet()             POST /webapi/wallet/getWallet      查余额
  ├── addCoin()               POST /webapi/wallet/addCoin        三方加减币
  ├── queryCoinRecordByOrderNo() POST /queryCoinRecordByOrderNo  按单号查记录
  ├── findCoinResult()        POST /webapi/wallet/findCoinResult 查账变结果
  └── getCoinRecordPage()     POST /webapi/wallet/coinRecord/page 账变分页

order-service (★共享数据库)
  ├── 不通过 Feign 调用 wallet-service
  ├── 直接读写 user_coin 表
  │   ├── subBalance()        UPDATE user_coin SET amount -= X    下注扣款
  │   ├── addBalance()        UPDATE user_coin SET amount += X    结算/回滚加款
  │   └── selectOne()         SELECT * FROM user_coin             余额查询
  └── 通过 MQ 发消息给 wallet-service
      ├── ADD_COIN_ORDER_QUEUE                   补写账变记录
      ├── ADD_COIN_NOT_NOTICE_BALANCE_QUEUE      静默补写
      └── ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE 补写+通知
```

### 11.3 完整调用链路图

```
用户下注场景（最复杂链路）：

Browser/App
  │
  ▼
websocket-service（WS连接）
  │
  ├──→ order-service（创建注单）
  │       │
  │       ├── SELECT user_coin（查余额）★ 直接读表
  │       ├── UPDATE user_coin SET amount -= bet_amount ★ 直接写表
  │       │
  │       └── MQ → wallet_add_coin_order_queue
  │                   │
  │                   ▼
  │              wallet-service（AddCoinListener #2）
  │                   │
  │                   ├── deductCoinBatch()（只写 coin_record 记录）
  │                   └── sendUserBalance()（MQ 通知余额变更）
  │                           │
  │                           ▼
  │                   websocket-service（推送新余额给前端）
  │
  ▼
Browser/App（余额刷新）
```

```
用户充值场景（走支付）：

Browser/App
  │
  ▼
websocket-service
  │
  ├──→ wallet-service /userDeposit/createOrder（创建充值订单）
  │       │
  │       └──→ pay-service（不在此仓库，独立部署）
  │               │
  │               └── 支付回调 → wallet-service
  │                       │
  │                       ├── addCoin()（加余额 + 写记录）
  │                       └── sendUserBalance()（通知）
  │
  ▼
Browser/App（余额刷新）
```

### 11.4 共享数据库反模式深度分析

这是整个参考项目**最严重的架构问题**。

**问题表现：**
```
wallet-service                     order-service
     │                                  │
     │  UserCoinRepository              │  UserCoinRepository
     │  (WHERE b.id = #{id})            │  (WHERE b.user_id = #{id})
     │         │                        │         │
     └─────────┼────────────────────────┼─────────┘
               │      同一张表          │
               ▼                        ▼
         ┌─────────────────────────────────┐
         │         user_coin 表             │
         │   ★ 两个服务同时写，无协调      │
         └─────────────────────────────────┘
```

**风险矩阵：**

| 场景 | wallet-service | order-service | 冲突 |
|------|---------------|---------------|------|
| 用户同时下注+提现 | subBalance(提现) | subBalance(下注) | 两个 UPDATE 竞争，SQL WHERE 防护不足 |
| 用户结算+充值 | addBalance(充值) | addBalance(结算) | 无冲突（加法可交换），但账变记录可能乱序 |
| 用户冻结+下注 | addFreezeBalance | subBalance | wallet冻结后 order 仍能扣款（不检查冻结） |

**为什么 order-service 要直接操作？**
推测原因：性能考虑。下注是超高频操作（每秒可能数百笔），如果每次都 Feign 调用 wallet-service，延迟太高。直接 SQL 操作消除了一次网络跳转。

**代价：**
1. 架构边界被打破，钱包不再是"唯一的余额真相源"
2. 两个服务的 SQL 不完全一致（WHERE 条件不同）
3. wallet-service 的 Redis 锁无法保护 order-service 的操作
4. 账变记录不完整：order-service 改了余额后异步 MQ 补记录，如果 MQ 丢失则无账变
5. 维护成本：修改余额逻辑需要同时改两个服务

**我们的方案：绝对不复制这个模式。**
- 所有余额操作必须通过 ser-wallet 的 RPC 接口
- Kitex RPC 延迟远低于 HTTP Feign（通常 < 1ms 内网），性能不是放弃架构的理由
- 如果极端高频场景需要优化，使用本地缓存+异步批量提交，而非共享数据库

---

## 第十二章 冻结/解冻/回滚三阶段

### 12.1 冻结模型总览

参考项目实现了一个简化版的 TCC（Try-Confirm-Cancel）模式：

```
          ┌─────────┐
          │  正常状态 │
          │  可用=100 │
          │  冻结=0   │
          └────┬────┘
               │ freezeCoin(30)
               ▼
          ┌─────────┐
          │  已冻结   │
          │  可用=70  │  ← amount不变，freeze_amount+30
          │  冻结=30  │
          └────┬────┘
              ╱ ╲
             ╱   ╲
            ╱     ╲
           ▼       ▼
    subFreeze(30)  rollback(30)
    (确认扣款)      (取消冻结)
           │         │
           ▼         ▼
    ┌─────────┐  ┌─────────┐
    │ 已扣款   │  │ 已回滚   │
    │ 可用=70  │  │ 可用=100 │  ← amount不变，freeze_amount-30
    │ 冻结=0   │  │ 冻结=0   │
    │ 总额=70  │  │ 总额=100 │
    └─────────┘  └─────────┘
```

### 12.2 与标准 TCC 的差异

| TCC 阶段 | 标准 TCC | 参考项目 | 差异 |
|----------|---------|---------|------|
| Try（预留） | 资源预留 + 状态记录 | freezeCoin | 有预留，无独立状态记录表 |
| Confirm（确认） | 使用预留资源 | subFreezeBalance | 有确认，但无 @Transactional |
| Cancel（取消） | 释放预留资源 | rollbackFreezeBalance | 有取消，但无 @Transactional |
| 幂等控制 | 每阶段可重入 | 无 | 重复调用会重复扣/回滚 |
| 空回滚 | 支持 Try 未执行时的 Cancel | 不支持 | 可能出现负 freeze_amount |
| 悬挂防护 | Cancel 之后 Try 不能执行 | 不支持 | 无 |

### 12.3 应用场景

在此参考项目中，冻结机制主要用于**提现流程**：

```
1. 用户发起提现 → freezeCoin(提现金额)
   → freeze_amount += X，可用余额减少

2a. 提现成功 → subFreezeBalance(提现金额)
   → amount -= X, freeze_amount -= X，总额减少

2b. 提现失败/取消 → rollbackFreezeBalance(提现金额)
   → freeze_amount -= X，可用余额恢复
```

### 12.4 缺陷与改进

**缺陷1：无冻结记录表**
- 没有 freeze_detail 表记录每笔冻结的明细
- 如果有两笔提现同时冻结，freeze_amount 是累加的，无法区分
- 部分回滚时无法精确回滚某一笔

**缺陷2：无幂等保护**
- freezeCoin 可以重复调用（每次都加 freeze_amount）
- 如果网络重试导致 freezeCoin 被调两次，多冻结的金额会"锁死"

**缺陷3：事务缺失**
- subFreezeBalance 和 rollbackFreezeBalance 都没有 @Transactional
- subFreezeBalance 内部有两步操作（改余额 + 写账变），不在事务中

**我们的改进方案（已在核心方案文档中设计）：**
1. 独立的 freeze_detail 表记录每笔冻结
2. 状态机控制：pending → confirmed / cancelled
3. 每阶段幂等：通过 freeze_detail.status 的 WHERE 条件防重复
4. 空回滚/悬挂防护：检查 freeze_detail 是否存在

---

## 第十三章 Bug清单与设计缺陷

### 13.1 确认的 Bug

| # | 严重度 | 位置 | 描述 | 影响 |
|---|--------|------|------|------|
| B1 | **严重** | CoinRecordServiceImpl:318 | 免转钱包路径 `coinRecordDTO.setCoinAmount(coinRecordDTO.getCoinTo())` → coinTo 刚被设为 null，coinAmount 也变 null | coin_record 表的 coin_amount 字段为 null |
| B2 | **严重** | CoinRecordServiceImpl:449 | subFreezeBalance 方法缺少 @Transactional | 改余额成功但写账变失败时，资金凭空消失 |
| B3 | **严重** | CoinRecordServiceImpl:476 | rollbackFreezeBalance 方法缺少 @Transactional | 虽然当前只有一步操作，但后续扩展存在风险 |
| B4 | **高** | CoinRecordServiceImpl:434 | freezeCoin 余额检查逻辑错误：检查 `amount >= freezeAmount` 而非 `(amount - 已有freeze) >= freezeAmount` | 已有冻结金额时可能超额冻结 |
| B5 | **高** | UserCoinMapper.xml:29-35 | subFreezeBalance SQL 无 `freeze_amount >= amount` 条件检查 | freeze_amount 可能变负 |
| B6 | **高** | UserCoinMapper.xml:37-42 | rollbackFreezeBalance SQL 无 `freeze_amount >= amount` 条件检查 | freeze_amount 可能变负 |
| B7 | **中** | AddCoinListener:141 | `log.info("下注帐变写入异常:{} 注单:{}", coinRecordVOList)` 少了异常参数 `e` | 异常堆栈丢失，无法排查 |
| B8 | **中** | AddCoinListener:137 | 日志在解析之前打印：`log.info("收到...消息: {}", coinRecordVOList)` 此时 coinRecordVOList 还是 null | 日志永远打印 null |

### 13.2 设计缺陷

| # | 严重度 | 类别 | 描述 | 影响 |
|---|--------|------|------|------|
| D1 | **致命** | 架构 | order-service 直接读写 user_coin 表（共享数据库） | 打破服务边界，锁机制失效，数据一致性无保障 |
| D2 | **严重** | 并发 | MQ Listener #2/#3/#4 无分布式锁 | 并发消费时数据竞态 |
| D3 | **严重** | 幂等 | 全局无幂等机制（无请求ID、无唯一约束） | 网络重试导致重复操作 |
| D4 | **高** | 架构 | 锁放在 Controller 层而非 Service 层 | Service 被内部调用时绕过锁 |
| D5 | **高** | 性能 | 每次操作都 Feign 调用 user-service 获取用户信息，无缓存 | 高频操作时 user-service 成为瓶颈 |
| D6 | **高** | 数据模型 | coin_record 无 currency 字段 | 多币种场景无法区分 |
| D7 | **高** | 数据模型 | coin_record 无唯一约束 (order_no + coin_type) | 重复记录无法防止 |
| D8 | **中** | 代码质量 | addCoin 方法 173 行，Transfer/FreeTransfer 两种逻辑混在一起 | 可读性差，维护困难 |
| D9 | **中** | 代码质量 | deductCoin 名称误导（只写记录不改余额） | 新人理解成本高 |
| D10 | **中** | 接口 | 响应类型不统一（ResponseEntity\<ResponseVO\> vs ResponseVO\<?\>） | 调用方适配困难 |
| D11 | **中** | MQ | publisher-confirm 已配置但代码未实现 ConfirmCallback | 消息发送失败无感知 |
| D12 | **中** | MQ | 无死信队列、无重试机制、消费无幂等 | 消息丢失或重复无法处理 |
| D13 | **中** | 数据模型 | coin_record 过度冗余用户信息（8+字段） | 存储浪费，数据同步问题 |
| D14 | **低** | API | 查询接口用 POST 而非 GET | 不符合 RESTful 最佳实践 |
| D15 | **低** | 代码风格 | CoinRecordVO 同时用于请求、响应、MQ消息体、Feign传输 | 职责不清，字段混乱 |

### 13.3 Bug 风险矩阵

```
严重度 ↑
       │
  致命 │  D1(共享DB)
       │
  严重 │  B2(无事务)  B3(无事务)  D2(MQ无锁)  D3(无幂等)
       │
  高   │  B4(冻结检查)  B5/B6(SQL无检查)  D4(锁位置)  D5(无缓存)
       │  D6/D7(数据模型)
       │
  中   │  B7/B8(日志)  D8-D13
       │
  低   │  D14/D15
       │
       └──────────────────────────────────── 出现频率 →
              低          中          高
```

---

## 第十四章 取舍评价：精华与糟粕

### 14.1 精华（值得借鉴）

#### 精华1：SQL 原子条件扣款

```sql
UPDATE user_coin SET amount = amount - X WHERE id = ? AND (amount - X) >= 0
```

**借鉴价值：9/10**
- 这是防止超扣的最后防线，我们必须采用
- 我们的实现：`WHERE available_amt >= X` + `SET available_amt = available_amt - X`
- 通过返回值判断是否成功（affected rows = 0 即失败）

#### 精华2：特殊场景允许负余额

```xml
<if test=" vo.coinType != 6 and vo.coinType != 5 ">
    AND (b.amount - #{vo.amount}) >= 0
</if>
```

**借鉴价值：8/10**
- 结算回滚(6)和取消(5)允许余额变负——业务上合理
- 这些是系统发起的补偿操作，必须无条件执行
- 我们的实现：在 Service 层通过 allow_negative 参数控制

#### 精华3：Redisson 锁释放三重检查

```java
if (Objects.nonNull(rLock) && rLock.isLocked() && rLock.isHeldByCurrentThread()) {
    rLock.unlock();
}
```

**借鉴价值：7/10**
- 防止释放别人的锁，防止重复释放
- 我们用 Go SetNX + Lua 解锁，同样需要判断"只释放自己的锁"
- 对应 Go 实现：Lua 脚本中 `if redis.call("GET", key) == owner then redis.call("DEL", key) end`

#### 精华4：双钱包模式的概念

**借鉴价值：6/10**（概念借鉴，实现重做）
- Transfer/FreeTransfer 的概念值得理解：有些商户自管余额，有些第三方管理
- 我们的场景也有类似需求（法币钱包 vs 虚拟币钱包可能有不同管理模式）
- 但实现上不应像参考项目那样用 if/else 混在一个方法里，应该用策略模式

#### 精华5：冻结/解冻/回滚三阶段思路

**借鉴价值：6/10**（思路正确，实现太粗糙）
- freeze → subFreeze/rollback 的三阶段模式是正确的
- 我们已经在核心方案中设计了更完善的 TCC 实现
- 需要加上：freeze_detail 表、状态机、幂等、空回滚防护

#### 精华6：MQ 异步写账变 + 余额通知分离

**借鉴价值：5/10**
- 高频操作（下注）先改余额，异步补写账变记录——减少主链路延迟
- 但牺牲了一致性（MQ 丢失则账变丢失）
- 我们的方案：余额变更和账变记录在同一事务内，不做异步拆分

### 14.2 糟粕（必须避免）

#### 糟粕1：共享数据库反模式（致命）

**严重度：致命，绝对不能复制**
- order-service 直接操作 user_coin 表，完全绕过 wallet-service
- 破坏了微服务架构的核心原则
- 导致：锁失效、数据不一致、无法独立扩缩容、维护困难

**我们的方案：** 所有余额操作必须通过 ser-wallet RPC 接口，零例外

#### 糟粕2：MQ 消费无锁无幂等（严重）

**严重度：严重**
- 4个 MQ 消费者中只有1个加锁
- 消息重复消费没有任何防护
- 消费失败直接丢弃或回滚订单

**我们的方案：** MQ 消费必须加锁 + 幂等 key + 死信队列 + 重试机制

#### 糟粕3：事务注解缺失（严重）

**严重度：严重**
- subFreezeBalance/rollbackFreezeBalance 缺少 @Transactional
- 多步操作不在事务中，部分成功会导致数据不一致

**我们的方案：** 所有写操作函数必须在 query.Q.Transaction 事务中

#### 糟粕4：锁放在 Controller 层（高）

**严重度：高**
- Controller 是框架层，Service 是业务层
- 锁应该在业务层，确保所有调用路径（HTTP/MQ/内部调用）都有保护

**我们的方案：** 锁在 Service 层获取和释放

#### 糟粕5：无幂等机制（高）

**严重度：高**
- 全局没有任何幂等设计
- 没有 requestId/idempotencyKey
- coin_record 表无唯一约束

**我们的方案：** 双层幂等（Redis SetNX + DB unique index on (order_no, op_type, wallet_type)）

#### 糟粕6：用户信息每次 Feign 查询无缓存（高）

**严重度：高**
- addCoin 每次调用都 Feign 查 user-service 获取用户信息
- 用户信息变更频率极低，完全可以缓存

**我们的方案：** 用户信息本地缓存（5min TTL）+ 变更时通知失效

#### 糟粕7：coin_record 过度冗余（中）

**严重度：中**
- 每条账变记录冗余 8+ 个用户/商户/代理字段
- 数据量大时存储浪费严重

**我们的方案：** coin_record 只存 user_id + 快照必要字段（如 user_name），其他通过关联查询

#### 糟粕8：addCoin 巨型方法（中）

**严重度：中**
- 173 行，两种钱包类型混在一起
- 违反单一职责原则

**我们的方案：** 策略模式分离不同钱包类型的实现

### 14.3 取舍总结矩阵

```
                    实现质量
                    高        低
              ┌──────────┬──────────┐
    价值  高  │ SQL条件扣款│ 冻结三阶段 │
              │ 特殊场景豁免│ 双钱包模式 │
              │ 锁三重释放 │           │
              ├──────────┼──────────┤
          低  │ MQ通知分离 │ 共享数据库 │
              │           │ 无事务注解 │
              │           │ 无幂等设计 │
              │           │ Controller锁│
              └──────────┴──────────┘

    左上：直接借鉴    右上：借鉴思路，重做实现
    左下：按需借鉴    右下：完全不借鉴
```

---

## 第十五章 对我们Go实现的启示

### 15.1 架构对比

| 维度 | 参考项目 | 我们的设计 | 改进点 |
|------|---------|-----------|--------|
| 服务边界 | 模糊（order直接操作wallet表） | 严格（所有操作走 RPC） | 核心原则 |
| 并发控制 | Redis 锁（单层，部分覆盖） | Redis 锁 + DB FOR UPDATE + SQL WHERE（三层，全覆盖） | 防线纵深 |
| 事务管理 | 部分有 @Transactional | 所有写操作 query.Q.Transaction | 无遗漏 |
| 幂等性 | 无 | Redis SetNX + DB unique index | 双层防护 |
| 冻结模型 | 简单 freeze_amount | freeze_detail 表 + 状态机 | 完整 TCC |
| 余额模型 | amount 含冻结 | available_amt + frozen_amt 分离 | 查询更优 |
| 多币种 | 单币种（每用户一行） | 多行 (user_id, wallet_type, currency_code) | 扩展性 |
| 数据精度 | BigDecimal（未明确精度） | DECIMAL(20,4) + shopspring/decimal | 全链路精确 |
| 接口协议 | HTTP REST (OpenFeign) | Thrift RPC (Kitex) | 更高效 |
| 错误处理 | boolean/ErrorCode/Exception 混用 | 统一 ret.BizErr | 一致性 |

### 15.2 必须从参考项目汲取的教训

1. **绝不允许共享数据库**——这是此项目最大的败笔，所有下游服务必须通过 ser-wallet RPC 操作余额
2. **锁必须全覆盖**——每个写入路径（HTTP、RPC、MQ）都必须有锁保护
3. **事务必须显式声明**——不能依赖"隐含的自动提交"
4. **幂等必须是基础设施级别的**——不是可选项
5. **冻结需要独立明细表**——不能只靠 freeze_amount 一个字段

### 15.3 可直接复用的设计模式

1. **SQL 原子条件扣款** → `WHERE available_amt >= ?`
2. **特殊场景豁免负余额** → 取消/回滚操作的 allow_negative 参数
3. **锁释放前的安全检查** → Lua 脚本中检查 owner
4. **余额变更通知机制** → MQ 广播给所有 WS 实例
5. **账变记录的 coinFrom/coinTo** → 审计必备字段

### 15.4 需要重新设计的部分

1. **钱包类型处理** → 策略模式而非 if/else
2. **冻结体系** → 完整 TCC + freeze_detail 表 + 状态机
3. **MQ 消费体系** → 消费端加锁 + 幂等 + 死信 + 重试
4. **用户信息获取** → 本地缓存 + 变更通知
5. **接口设计** → Thrift IDL 强类型 + 幂等 key + 版本控制
6. **数据模型** → 多币种 + version 乐观锁 + 合理冗余

---

## 附录A 枚举值完整映射表

### A.1 CoinChangeEnum（账变类型）

| Code | Name | 英文名 | 方向 | 余额检查 | SQL条件 |
|------|------|--------|------|---------|---------|
| 1 | 转入 | TRANSFER_IN | 收入 | 不检查 | 无条件加 |
| 2 | 转出 | TRANSFER_OUT | 支出 | ★ 检查余额 | 条件减 |
| 3 | 下注 | BET | 支出 | ★ 检查余额 | 条件减 |
| 4 | 派彩 | SETTLEMENT | 收入 | 不检查 | 无条件加 |
| 5 | 取消 | ORDER_CANCEL | 收入 | 不检查 | ★ 允许负余额 |
| 6 | 结算回滚 | SETTLEMENT_ROLLBACK | 支出 | 不检查 | ★ 允许负余额 |
| 7 | 重算 | SETTLEMENT_SECOND | 可收可支 | 不检查 | 无条件 |
| 8 | 打赏 | SEND_REWARD | 支出 | ★ 检查余额 | 条件减 |
| 9 | 闪电投注 | LIGHTNING_BET | 支出 | (与BET相同逻辑) | 条件减 |

### A.2 CoinChangeDetailEnum（账变明细）

| Code | Name |
|------|------|
| 1 | 会员存款 |
| 2 | 会员提款 |
| 3 | 下注 |
| 4 | 结算 |
| 5 | 取消 |
| 6 | 结算取消 |
| 7 | 打赏 |
| 8 | 闪电投注 |

### A.3 BalanceTypeEnum（收支类型）

| Code | Name | 含义 |
|------|------|------|
| 1 | income | 收入（余额增加） |
| 2 | expenses | 支出（余额减少） |

### A.4 WalletTypeEnum（钱包类型）

| Code | Name | 含义 |
|------|------|------|
| 1 | TRANSFER | 转账钱包（本地管理余额） |
| 2 | FREE_TRANSFER | 免转钱包（三方管理余额） |

---

## 附录B 全服务Feign接口清单

### B.1 wallet-service 出站调用

#### → user-service

```java
@FeignClient(name = "user-service-feign", url = "http://${k8s.svc.user:localhost:8906}")
public interface UserService {
    @GetMapping("/user/v1/getUserByUserId")
    ResponseEntity<ResponseVO<UserInfoVO>> getUserByUserId(@RequestParam String userId);
}
```

#### → order-service

```java
@FeignClient(name = "order-service-feign", url = "http://${k8s.svc.order:localhost:8908}")
public interface OrderResource {
    @PostMapping("/order/bet/updateOrdersToInitialStatus")
    void updateOrdersToInitialStatus(@RequestBody List<String> oderNos);

    @GetMapping("/order/v1/findOrderByOrderNo")
    ResponseEntity<ResponseVO<OrderVO>> findOrderByOrderNo(@RequestParam String orderNo);

    @PostMapping("/order/thirdOrderRecord/saveThirdOrderRecord")
    ResponseEntity<ThirdOrderRecordDTO> saveThirdOrderRecord(@RequestBody ThirdOrderRecordDTO dto);
}
```

#### → api-service

```java
@FeignClient(name = "api-service-feign", url = "http://${k8s.svc.api:localhost:8910}")
public interface ApiResource {
    @PostMapping("/game/api/amountTransfer")
    ResponseVO<ThirdOrderResponseVO> amountTransfer(@RequestBody ThirdOrderRequestVO vo);
}
```

#### → merchant-service

```java
@FeignClient(name = "merchant-service-feign", url = "http://${k8s.svc.merchant:localhost:8902}")
public interface MerchantFeignResource {
    @PostMapping("/adminApi/merchant/getMerchantDetail")
    ResponseEntity<ResponseVO<MerchantDetailVO>> getMerchantDetail(@RequestBody IdVO vo);
}
```

### B.2 wallet-service 入站调用

#### websocket-service → wallet-service (15 端点)

```java
@FeignClient(name = "wallet-service-feign", url = "http://${k8s.svc.wallet:localhost:8907}")
public interface WalletResource {
    @PostMapping("/webapi/wallet/getWallet")
    ResponseVO<UserCoinVO> getWallet(@RequestBody UserCoinVO userCoinVO);

    @PostMapping("/webapi/wallet/users/wallet/list")
    ResponseVO<List<UserCoinVO>> getWalletListByUserId(@RequestBody QueryWalletReqVO vo);

    @PostMapping("/webapi/wallet/addCoin")
    ResponseVO<?> addCoin(@RequestBody CoinRecordVO coinRecordVO);

    @PostMapping("/webapi/wallet/userCoinRecordPage")
    ResponseVO<Page<UserCoinRecordPageVO>> userCoinRecord(@RequestBody UserCoinRecordParamVo vo);

    @PostMapping("/webapi/wallet/userCoinRecordList")
    ResponseVO<List<UserCoinRecordPageVO>> userCoinRecordList(UserCoinRecordParamVo vo);

    @PostMapping("/userWithdraw/userWithdrawApply")
    ResponseVO<ErrorCode> userWithdrawApply(@RequestBody UserWithDrawParamVo vo);

    @PostMapping("/userDeposit/queryUserRechargingInfo")
    ResponseVO<UserDepositRecordVO> queryUserRechargingInfo(@RequestParam String userId);

    @PostMapping("/userDeposit/createOrder")
    ResponseVO<ErrorCode> createOrder(@RequestBody UserDepositRecordVO vo);

    @PostMapping("/userWithdraw/userWithdrawConfigInfo")
    ResponseVO<UserWithdrawConfigVO> userWithdrawConfigInfo(@RequestParam String currency);

    @PostMapping("/userOfflineDeposit/userRecharge")
    ResponseVO<OrderNoVO> userRecharge(@RequestBody UserRechargeReqVo vo);

    @PostMapping("/userOfflineDeposit/uploadVoucher")
    ResponseVO<Integer> uploadVoucher(@RequestBody DepositOrderFileVO vo);

    @PostMapping("/userOfflineDeposit/cancelDepositOrder")
    ResponseVO<Integer> cancelDepositOrder(@RequestBody OrderNoVO vo);

    @PostMapping("/userOfflineDeposit/depositOrderDetail")
    ResponseVO<UserDepositOrderDetailVO> depositOrderDetail(OrderNoVO vo);

    @PostMapping("/offlineReceived/getOfflineCollectionList")
    ResponseEntity<ResponseVO<List<VirtualCollectionVO>>> getOfflineCollectionList();

    @PostMapping("/offlineReceived/getQuickAmountList")
    ResponseEntity<ResponseVO<QuickAmountListAndMaxMinBO>> getQuickAmountList(@RequestBody VirtualCollectionVO vo);
}
```

#### common-service → wallet-service (3 端点)

```java
@FeignClient(name = "wallet-service-resource", url = "http://${k8s.svc.wallet:localhost:8907}")
public interface WalletServiceResource {
    @RequestMapping("/webapi/wallet/addCoin")
    ResponseEntity<ResponseVO<?>> addCoin(@RequestBody CoinRecordVO coinRecordVO);

    @PostMapping("/webapi/wallet/getWallet")
    ResponseEntity<ResponseVO<UserCoinVO>> getWallet(@RequestBody UserCoinVO coinRecordVO);

    @PostMapping("/userWithdraw/addWithdrawLimitAmount")
    ResponseVO addWithdrawLimitAmount(@RequestBody WithdrawLimitAmountParamVO vo);
}
```

#### api-service → wallet-service (5 端点)

```java
@FeignClient(name = "wallet-service-feign", url = "http://${k8s.svc.wallet:localhost:8907}")
public interface WalletResource {
    @PostMapping("/webapi/wallet/getWallet")
    ResponseVO<UserCoinVO> getWallet(@RequestBody UserCoinVO userCoinVO);

    @PostMapping("/webapi/wallet/addCoin")
    ResponseVO<?> addCoin(@RequestBody CoinRecordOrderVO coinRecordVO);

    @PostMapping("/webapi/wallet/queryCoinRecordByOrderNo")
    ResponseVO<CoinRecordVO> queryCoinRecordByOrderNo(@RequestParam String orderNo);

    @PostMapping("/webapi/wallet/findCoinResult")
    ResponseVO<CoinRecordVO> findCoinResult(@RequestBody CoinRecordVO coinRecordVO);

    @PostMapping("/webapi/wallet/coinRecord/page")
    ResponseVO<Page<CoinRecordResponseVO>> getCoinRecordPage(@RequestBody CoinRecordParamVO vo);
}
```

### B.3 服务端口一览

| 服务 | 端口 | K8s SVC 变量 |
|------|------|-------------|
| wallet-service | 8907 | k8s.svc.wallet |
| user-service | 8906 | k8s.svc.user |
| order-service | 8908 | k8s.svc.order |
| merchant-service | 8902 | k8s.svc.merchant |
| api-service | 8910 | k8s.svc.api |

---

## 附录C 关键代码片段索引

### C.1 源文件索引

| 文件 | 路径 | 行数 | 核心作用 |
|------|------|------|---------|
| CoinRecordServiceImpl.java | p9-services/wallet-service/.../service/impl/ | 522 | 钱包核心逻辑 |
| WalletController.java | p9-services/wallet-service/.../controller/ | 251 | REST 端点 + 锁 |
| AddCoinListener.java | p9-services/wallet-service/.../rabbitmq/listener/ | 147 | MQ 消费 |
| UserCoinMapper.xml | p9-services/wallet-service/.../resources/mapper/ | 44 | 原子 SQL |
| UserCoinServiceImpl.java | p9-services/wallet-service/.../service/impl/ | ~80 | 余额查询 |
| UserCoinDTO.java | p9-services/wallet-service/.../dto/ | 32 | user_coin 映射 |
| CoinRecordDTO.java | p9-services/wallet-service/.../dto/ | 59 | coin_record 映射 |
| WalletUtilEnum.java | p9-core-lib/.../enums/wallet/ | ~150 | 账变枚举 |
| RedisLockKey.java | p9-core-lib/.../common/ | ~30 | Redis 锁 Key |
| MqConstants.java | p9-services/wallet-service/.../constant/ | ~250 | MQ 常量 |
| CoinRecordVO.java | p9-services/wallet-service/.../vo/wallet/ | ~50 | 账变 VO |
| CoinFreezeVO.java | p9-services/wallet-service/.../vo/wallet/ | ~12 | 冻结 VO |
| CoinFreezeRecordVO.java | p9-services/wallet-service/.../vo/wallet/ | ~20 | 解冻 VO |
| UserCoinVO.java | p9-services/wallet-service/.../vo/wallet/ | ~30 | 钱包 VO |

### C.2 order-service 越权文件索引

| 文件 | 路径 | 越权操作 |
|------|------|---------|
| UserCoinDTO.java | p9-live-services/order-service/.../vo/ | @TableName("user_coin") |
| UserCoinRepository.java | p9-live-services/order-service/.../repository/ | BaseMapper + subBalance/addBalance |
| UserCoinMapper.xml | p9-live-services/order-service/.../resources/mapper/ | UPDATE user_coin |
| OrderServiceImpl.java | p9-live-services/order-service/.../service/impl/ | subBalance() 调用 |
| OrderBetServiceImpl.java | p9-live-services/order-service/.../service/impl/ | addBalance/selectOne 调用 |
| OrderAddCoinListener.java | p9-live-services/order-service/.../rabbitmq/ | addBalance/selectOne 调用 |
| OrderDealListener.java | p9-live-services/order-service/.../rabbitmq/ | selectOne 调用 |

### C.3 关键行号速查

| 功能 | 文件 | 行号 |
|------|------|------|
| addCoin 核心方法 | CoinRecordServiceImpl.java | 169-341 |
| Transfer 钱包分支 | CoinRecordServiceImpl.java | 182-230 |
| FreeTransfer 钱包分支 | CoinRecordServiceImpl.java | 231-338 |
| null coinAmount bug | CoinRecordServiceImpl.java | 316-318 |
| 余额不足校验 | CoinRecordServiceImpl.java | 366-384 |
| freezeCoin 余额检查 bug | CoinRecordServiceImpl.java | 434 |
| subFreezeBalance 无事务 | CoinRecordServiceImpl.java | 449 |
| rollbackFreezeBalance 无事务 | CoinRecordServiceImpl.java | 476 |
| Redisson 锁模板 | WalletController.java | 60-78 |
| MQ Listener#1 有锁 | AddCoinListener.java | 37-67 |
| MQ Listener#2 无锁 | AddCoinListener.java | 70-100 |
| MQ Listener#4 日志bug | AddCoinListener.java | 137, 141 |
| SQL subBalance 条件扣款 | UserCoinMapper.xml | 4-13 |
| SQL subFreezeBalance | UserCoinMapper.xml | 29-35 |

---

> **文档总结**
>
> 这个参考钱包工程是一个**典型的"能跑但有隐患"的实现**。核心思路（SQL原子操作、分布式锁、冻结三阶段）方向正确，但在工程质量上存在多处关键缺陷：共享数据库反模式、事务缺失、幂等缺失、锁覆盖不全。
>
> 对我们的价值：
> - **概念验证**：证明了 SQL 条件扣款 + Redis 锁的组合可以工作
> - **反面教材**：清晰展示了不加事务、不做幂等、共享数据库的后果
> - **参数参考**：锁超时（20s/30s）、枚举值定义、表结构字段 都有参考价值
> - **架构警示**：性能优化不能以牺牲架构边界为代价
>
> 我们的 Go 实现应当"取其骨架，重铸血肉"——借鉴 SQL 原子操作和冻结三阶段的思路，在此基础上构建完整的三层并发控制、双层幂等、TCC 状态机、以及严格的微服务边界。
