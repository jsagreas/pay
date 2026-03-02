# 钱包参考B — tg-services/wallet-server 完整深度解剖分析

> 分析对象：`/Users/mac/gitlab/z-tmp/tg-services/wallet-server`（Java Spring Boot）
> 核心价值：**TRON区块链虚拟币充值/提现/归集完整链路实现**
> 分析定位：作为我们 ser-wallet（Go）区块链充值模块的核心参考工程
> 分析原则：取其精华去其糟粕，借鉴/调整/规避三分类，每个决策有理有据

---

## 目录

- [第1章 项目概览与技术栈对比](#第1章-项目概览与技术栈对比)
- [第2章 架构设计与分层分析](#第2章-架构设计与分层分析)
- [第3章 数据模型与表结构](#第3章-数据模型与表结构)
- [第4章 区块链充值完整链路（核心重点）](#第4章-区块链充值完整链路核心重点)
- [第5章 区块链提现完整链路](#第5章-区块链提现完整链路)
- [第6章 归集机制（Collection）](#第6章-归集机制collection)
- [第7章 钱包地址生成与密钥管理](#第7章-钱包地址生成与密钥管理)
- [第8章 链上交互层（TRON RPC）](#第8章-链上交互层tron-rpc)
- [第9章 外部签名系统](#第9章-外部签名系统)
- [第10章 定时任务体系](#第10章-定时任务体系)
- [第11章 跨服务调用与依赖关系](#第11章-跨服务调用与依赖关系)
- [第12章 枚举与类型系统](#第12章-枚举与类型系统)
- [第13章 接口设计（API清单）](#第13章-接口设计api清单)
- [第14章 并发控制与安全机制分析](#第14章-并发控制与安全机制分析)
- [第15章 核心缺陷与不足清单](#第15章-核心缺陷与不足清单)
- [第16章 借鉴/调整/规避决策表](#第16章-借鉴调整规避决策表)
- [附录A 源码文件清单](#附录a-源码文件清单)
- [附录B 与钱包参考A（p9）对比矩阵](#附录b-与钱包参考ap9对比矩阵)
- [附录C 对我们ser-wallet区块链充值模块的设计启示](#附录c-对我们ser-wallet区块链充值模块的设计启示)

---

## 第1章 项目概览与技术栈对比

### 1.1 项目定位

tg-services 是一个**基于Telegram的泛娱乐竞彩平台**微服务集群，wallet-server 是其中专门负责**TRON区块链资产管理**的独立服务。

与 p9（钱包参考A）不同，这个钱包**不维护内部余额**——它是一个纯粹的**链上钱包服务**：
- 不存储用户余额（余额存在链上，查询链上实时余额）
- 核心职责：生成地址、链上转账、归集、查询交易状态
- 用户余额管理在 user-server 的 Assets 表中
- wallet-server 只负责"链上操作"，user-server 负责"账务操作"

**这个架构分离非常关键**：链上操作和账务操作是两个独立的事务域。

### 1.2 技术栈对比

| 维度 | tg-services wallet-server | 我们 ser-wallet |
|------|---------------------------|-----------------|
| 语言 | Java 17 | Go 1.21+ |
| 框架 | Spring Boot 2.5.3 | CloudWeGo Hertz + Kitex |
| ORM | JPA/Hibernate | GORM |
| 数据库 | MySQL 5.7 (cloud_wallet) | TiDB |
| 缓存 | Redis (Lettuce) | Redis (go-redis) |
| 消息队列 | RabbitMQ (AMQP) | 待定（Kafka/NATS） |
| 区块链SDK | wallet-cli-1.0.jar + web3j 3.6 | 待定（Go TRON SDK） |
| 区块链RPC | gRPC + HTTP API（TronGrid） | HTTP API（TronGrid/自建节点） |
| 密钥管理 | AWS KMS + AES-128-ECB | 待定（应采用更安全的方案） |
| 签名系统 | 外部独立签名服务（RSA通信） | 待定 |
| 服务通信 | HTTP REST（硬编码URL） | Kitex RPC |
| 定时任务 | Spring @Scheduled | 待定（cron/定时器） |
| 端口 | 9600 | 待定 |

### 1.3 工程生态全景

tg-services 共13个模块：

```
tg-services/
├── module-common          ← 公共库（MQ常量、响应封装、工具类、枚举）
├── module-jjwt            ← JWT认证库
├── module-authenticator   ← 认证中间件
├── module-springcache-redis ← Redis缓存集成
├── module-swagger2        ← API文档
├── wallet-server          ← ★ 区块链钱包服务（Port 9600）
├── wallet-third-server    ← 第三方游戏钱包（Port 9601）
├── user-server            ← ★ 用户/资产管理（Port 9300）
├── game-server            ← 游戏/结算（Port 9500）
├── casino                 ← 前台/后台（9401/9402/9403）
├── authentication-server  ← 认证网关
├── telegram-bot-api       ← TG机器人
├── sms                    ← 短信服务
└── file-upload-download   ← 文件存储（MinIO）
```

**钱包相关的核心交互链路**：
```
user-server ←→ wallet-server ←→ TRON区块链
     ↑                              ↑
     |                              |
casino-web ←→ game-server     外部签名服务
```

### 1.4 数据库分布

| 数据库 | 服务 | 核心表 |
|--------|------|--------|
| cloud_wallet | wallet-server | trx_wallet, trx_wallet_order, trx_wallet_collect, trx_wallet_config |
| cloud_user | user-server | user, assets, balance_change, ss_order, play_money_change |
| cloud_game | game-server | 下注记录、结算数据 |

---

## 第2章 架构设计与分层分析

### 2.1 wallet-server 内部分层

```
┌─────────────────────────────────────────────────┐
│                  Controller 层                    │
│  WalletController.java（6个REST端点）              │
│  TestController.java（测试端点）                    │
├─────────────────────────────────────────────────┤
│                  Driver 层                        │
│  TrxDriver.java ★（链上业务编排/调度中枢）          │
│  AwsKmsDriver.java（AWS KMS加解密驱动）             │
├─────────────────────────────────────────────────┤
│                  Business 层                      │
│  WalletTrxBusiness.java ★（核心链上交互实现）       │
│  （地址生成、签名、广播、余额查询、交易查询）         │
├─────────────────────────────────────────────────┤
│                  Service 层                       │
│  TrxWalletService.java（钱包CRUD）                 │
│  TrxWalletOrderService.java（订单CRUD）             │
│  TrxWalletCollectService.java（归集CRUD）            │
├─────────────────────────────────────────────────┤
│                  Repository 层                    │
│  TrxWalletRepository（JPA）                        │
│  TrxWalletOrderRepository（JPA + 原生SQL）          │
│  TrxWalletCollectRepository（JPA）                  │
│  TrxWalletConfigRepository（JPA）                   │
├─────────────────────────────────────────────────┤
│                  Client/Util 层                   │
│  Trc20.java（TRC20代币交互：余额/精度/转账）         │
│  SmartContract.java（智能合约调用封装）               │
│  TronUtil.java（gRPC客户端、签名、广播）              │
│  TronUtils.java（地址转换、交易打包）                 │
│  AES.java / RSAUtil.java（加密工具）                 │
│  Base58.java / ByteArray.java / Sha256Hash.java   │
├─────────────────────────────────────────────────┤
│                  Task 层                          │
│  CollectTrxJob.java（每日8点定时归集）                │
├─────────────────────────────────────────────────┤
│                  Model 层                         │
│  Entity: TrxWallet, TrxWalletOrder,               │
│          TrxWalletCollect, TrxWalletConfig        │
│  Enum: TRXContractEnum, TrxOrderTypeEnum,         │
│        CollectTypeEnum                            │
│  VO: Request DTOs (5) + Response DTOs (4)         │
└─────────────────────────────────────────────────┘
```

### 2.2 架构特点分析

**优点**：
1. **Driver-Business 两层分离**：TrxDriver 负责业务流程编排（先查余额→再转手续费→再转主币），WalletTrxBusiness 负责底层链上操作（构建交易→签名→广播）
2. **链上钱包与账务分离**：wallet-server 不碰余额，user-server 不碰私钥，职责清晰
3. **配置外置**：敏感配置（地址、私钥）全部通过 AWS KMS 加密存储在配置文件中

**缺点**：
1. **Driver 层职责过重**：TrxDriver 既做业务编排又做数据操作，338行代码承担了太多职责
2. **没有接口抽象**：所有类直接依赖实现类，没有 interface，无法替换或Mock
3. **没有异常分层**：全部用 Exception + log.error，没有自定义业务异常体系
4. **静态方法滥用**：WalletTrxBusiness 中大量 static 方法（encrypt/decrypt/fromHexAddress等），不利于测试

### 2.3 调用链路总览

```
Controller
  └→ TrxDriver（编排层）
       ├→ TrxWalletService（查地址/查私钥）
       ├→ TrxWalletOrderService（写订单记录）
       ├→ TrxWalletCollectService（写归集记录）
       ├→ AwsKmsDriver（解密敏感配置）
       ├→ TrxWalletConfigRepository（读动态配置）
       └→ WalletTrxBusiness（链上操作）
            ├→ TRON HTTP API（/wallet/triggersmartcontract等）
            ├→ TRON gRPC Client（通过wallet-cli）
            ├→ 外部签名服务（RSA加密通信）
            ├→ Trc20.java（TRC20余额查询）
            └→ SmartContract.java（合约调用封装）
```

---

## 第3章 数据模型与表结构

### 3.1 wallet-server 的4张表

#### 表1：trx_wallet（用户链上钱包）

```sql
CREATE TABLE trx_wallet (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     VARCHAR(10)  COMMENT '会员账号',      -- INDEX
    user_name   VARCHAR(30)  COMMENT '会员姓名',
    en_private_key VARCHAR(250) COMMENT '加密后的私钥', -- AES加密存储
    address     VARCHAR(100) COMMENT '钱包地址',       -- TRON T开头地址
    create_time TIMESTAMP,
    create_by   VARCHAR(255),
    update_time TIMESTAMP,
    update_by   VARCHAR(255),
    INDEX idx_user_id (user_id)
);
```

**关键设计分析**：
- 一个用户一个链上地址（1:1关系）
- 私钥AES加密存储，解密需要 `passphrase + address` 作为密钥
- **缺陷**：user_id 没有 UNIQUE 约束，理论上可能重复创建
- **缺陷**：address 没有 UNIQUE 约束，虽然生成重复地址概率极低但缺乏DB级保障
- **缺陷**：user_id 用 VARCHAR(10)，长度可能不够

#### 表2：trx_wallet_order（链上交易订单）

```sql
CREATE TABLE trx_wallet_order (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     VARCHAR(10)  COMMENT '会员账号',       -- INDEX
    tx_id       VARCHAR(200) COMMENT '交易id',          -- INDEX，链上交易哈希
    ss_order_id VARCHAR(200) COMMENT '充值/提现记录id',  -- 关联user-server的ss_order
    amount      DECIMAL(16,2) COMMENT '金额',
    from_address VARCHAR(250) COMMENT '发送地址',
    to_address  VARCHAR(250) COMMENT '接收地址',
    coin_name   VARCHAR(10)  COMMENT '币种类型',        -- TRX/USDT/CXT
    order_type  TINYINT(2)   COMMENT '收付方向',        -- 1=充值, 2=提现
    post_type   TINYINT(2)   COMMENT '调用方式',        -- 1=接口, 2=定时任务
    create_time TIMESTAMP,
    create_by   VARCHAR(255),
    update_time TIMESTAMP,
    update_by   VARCHAR(255),
    INDEX idx_user_id (user_id),
    INDEX idx_order_type (order_type),
    INDEX idx_tx_id (tx_id)
);
```

**关键设计分析**：
- tx_id 是区块链交易哈希，全局唯一
- ss_order_id 关联 user-server 的订单表
- **缺陷**：ss_order_id 没有 UNIQUE 约束——同一个订单可能被重复记录
- **缺陷**：金额用 DECIMAL(16,2) 只有2位小数，但 USDT 在 TRON 上有6位精度（1 USDT = 1,000,000 SUN），**精度丢失风险**
- **缺陷**：没有交易状态字段（pending/confirmed/failed），只记录了发起，无法追踪链上确认状态
- **亮点**：post_type 区分了手动触发和定时任务触发，便于问题排查

#### 表3：trx_wallet_collect（归集记录）

```sql
CREATE TABLE trx_wallet_collect (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(10)  COMMENT '会员账号',    -- INDEX
    tx_id           VARCHAR(200) COMMENT '交易id',       -- INDEX
    collect_address VARCHAR(250) COMMENT '汇款账号',     -- 用户地址（from）
    amount          DECIMAL(16,2) COMMENT '汇款金额',
    collect_type    TINYINT(2)   COMMENT '归集类型',     -- 1=单笔, 2=批量定时
    create_time     TIMESTAMP,
    create_by       VARCHAR(255),
    update_time     TIMESTAMP,
    update_by       VARCHAR(255),
    INDEX idx_user_id (user_id),
    INDEX idx_collect_type (collect_type),
    INDEX idx_tx_id (tx_id)
);
```

**关键设计分析**：
- 归集 = 把用户地址上的币转到公司主地址（集中管理资金）
- 区分单笔归集（充值后立即触发）和批量归集（定时任务）
- **缺陷**：没有记录归集目标地址（to_address），默认都是转到 companyAddress
- **缺陷**：同样没有状态字段，无法追踪归集交易是否成功

#### 表4：trx_wallet_config（钱包配置）

```sql
CREATE TABLE trx_wallet_config (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    type        VARCHAR(30)   COMMENT '类型',
    code        VARCHAR(50)   COMMENT '标识code',
    value       VARCHAR(1000) COMMENT '标识value',
    create_time TIMESTAMP,
    update_time TIMESTAMP
);
```

**配置项**：
- type=`sign`, code=`rsa` → RSA公钥（与签名服务通信）
- type=`sign`, code=`platCode` → 商户号
- type=`sign`, code=`disPendAddress` → 下发地址（提现出款地址）

**设计思路**：把频繁变更的配置（签名地址、RSA密钥等）放数据库而非配置文件，运行时可动态切换。

### 3.2 user-server 关联的3张核心表

#### 表5：assets（用户资产——余额在这里）

```sql
CREATE TABLE assets (
    user_id        VARCHAR(32) PRIMARY KEY COMMENT '用户ID',
    balance        DECIMAL(16,2) COMMENT '通用余额',
    balance_nn     DECIMAL(16,2) COMMENT '牛牛游戏余额',
    balance_bacc   DECIMAL(16,2) COMMENT '百家乐余额',
    freeze_amount  DECIMAL(16,2) COMMENT '冻结金额',
    play_money     DECIMAL(16,2) COMMENT '打码量',
    credit_amount  DECIMAL(16,2) COMMENT '信用额度',
    version        INT          COMMENT '乐观锁版本号',
    -- 代理/渠道层级字段（5级）...
    user_type      INT COMMENT '用户类型'
);
```

**关键设计分析**：
- **余额管理在 user-server，不在 wallet-server**
- freeze_amount 用于提现冻结：提现时先冻结 → 链上确认后扣减
- version 字段说明使用了乐观锁
- 多游戏余额分开管理（balance、balance_nn、balance_bacc）
- **缺陷**：DECIMAL(16,2) 精度不够，同 wallet-server 一样的问题

#### 表6：ss_order（充提订单——连接 user-server 和 wallet-server 的桥梁）

```sql
CREATE TABLE ss_order (
    id                 BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_num          VARCHAR(32)  COMMENT '订单号',
    user_id            VARCHAR(32),
    tg_user_id         VARCHAR(32)  COMMENT 'TG用户ID',
    order_type         INT          COMMENT '1=充值, 2=提现',
    order_status       INT          COMMENT '状态',
    -- 1=待处理, 2=成功, 3=失败, 4=取消, 50=虚拟币待处理
    amount             DECIMAL(16,2) COMMENT '订单金额',
    real_amount        DECIMAL(16,2) COMMENT '实际金额',
    fee_amount         DECIMAL(16,2) COMMENT '手续费',
    virtual_currency   VARCHAR(20)  COMMENT '虚拟币类型（USDT等）',
    payee_address      VARCHAR(200) COMMENT '提现收款地址',
    tx_id              VARCHAR(200) COMMENT '区块链交易ID',
    flow_multiple      INT          COMMENT '流水倍数',
    adjustment_type    INT          COMMENT '调整类型',
    audit_user         VARCHAR(50)  COMMENT '审核人',
    audit_time         TIMESTAMP    COMMENT '审核时间',
    audit_remark       VARCHAR(500) COMMENT '审核备注',
    withdraw_user      VARCHAR(50),
    withdraw_file_key  VARCHAR(200),
    complete_time      TIMESTAMP    COMMENT '完成时间',
    is_withdraw_lock   INT          COMMENT '提现锁定',
    create_time        TIMESTAMP,
    update_time        TIMESTAMP
);
```

**关键设计分析**：
- **这是整个充值/提现流程的状态机载体**
- order_status=50 是虚拟币专用的"待处理"状态
- amount 和 real_amount 分开：充值时 real_amount 由链上实际到账金额决定
- fee_amount 单独记录手续费
- tx_id 关联链上交易
- flow_multiple 是打码量倍数（充值金额 × 倍数 = 需要完成的打码量）
- 有审核相关字段（审核人、审核时间、审核备注）

#### 表7：balance_change（资金流水——审计日志）

```sql
CREATE TABLE balance_change (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id           VARCHAR(32),
    change_type       INT          COMMENT '变动类型',
    -- 1=充值, 2=下注, 3=派奖, 4=提现, 8=TG充值, 90=手续费...
    balance_type      INT          COMMENT '1=收入, 2=支出',
    before_amount     DECIMAL(16,2) COMMENT '变动前余额',
    amount            DECIMAL(16,2) COMMENT '变动金额',
    after_amount      DECIMAL(16,2) COMMENT '变动后余额',
    relate_id         BIGINT       COMMENT '关联订单ID',
    relate_order_num  VARCHAR(32)  COMMENT '关联订单号',
    business_type     INT          COMMENT '业务类型',
    order_type        INT,
    create_time       TIMESTAMP,
    UNIQUE KEY uk_change (change_type, relate_id, user_id, order_type)
);
```

**关键设计分析**：
- **有 UNIQUE KEY！** 这是防重复的关键——同一个变动类型+订单+用户只能写一次
- 记录了 before_amount / after_amount，满足审计溯源需求
- change_type=8 是 TG虚拟币充值专用类型
- **这正是我们 ser-wallet 的 wallet_flow 表应该学习的模式**

### 3.3 表关系图

```
cloud_wallet 库（wallet-server）:
┌──────────────┐    ┌──────────────────┐    ┌───────────────────┐
│  trx_wallet  │    │ trx_wallet_order │    │ trx_wallet_collect│
│              │    │                  │    │                   │
│ userId ──────┼──→ │ userId           │    │ userId            │
│ address      │    │ txId             │    │ txId              │
│ enPrivateKey │    │ ssOrderId ───────┼──→ │ collectAddress    │
│              │    │ fromAddress      │    │ amount            │
│              │    │ toAddress        │    │ collectType       │
│              │    │ amount           │    │                   │
│              │    │ coinName         │    │                   │
│              │    │ orderType        │    │                   │
└──────────────┘    └──────────────────┘    └───────────────────┘
                           │
                    ssOrderId 关联
                           │
                           ↓
cloud_user 库（user-server）:
┌──────────────┐    ┌──────────────────┐    ┌───────────────────┐
│   assets     │    │    ss_order      │    │  balance_change   │
│              │    │                  │    │                   │
│ userId ──────┼──→ │ userId           │ ←──┼─ relateId         │
│ balance      │    │ orderNum         │    │ userId            │
│ freezeAmount │    │ orderType        │    │ changeType        │
│ version      │    │ orderStatus      │    │ beforeAmount      │
│              │    │ amount/realAmount│    │ amount            │
│              │    │ feeAmount        │    │ afterAmount       │
│              │    │ txId             │    │ UK(type,id,user)  │
│              │    │ virtualCurrency  │    │                   │
└──────────────┘    └──────────────────┘    └───────────────────┘
```

---

## 第4章 区块链充值完整链路（核心重点）

> **这是本文档最重要的章节**——tg-services 对我们最大的参考价值就在于这条区块链充值链路。

### 4.1 充值模型概述

tg-services 的虚拟币充值模型是**"用户独立地址 + 链上监听 + 归集"模式**：

```
1. 每个用户分配一个独立的 TRON 链上地址
2. 用户向自己的地址转入 USDT（链外操作，用户自行操作）
3. 系统监听/查询用户地址余额，确认到账
4. 到账后触发归集：把用户地址的币转到公司主地址
5. 归集成功后，在 user-server 中给用户加余额
```

**注意**：这个工程的"充值"不是传统的支付网关充值，而是**用户自行向链上地址转币**，系统通过查询链上余额来确认充值。

### 4.2 充值链路分阶段详解

#### 阶段1：钱包地址生成（前置条件）

```
用户注册/首次充值
  → user-server 调用 wallet-server POST /wallet/create
  → WalletController.createWallet()
  → WalletTrxBusiness.createWallet()
```

**WalletTrxBusiness.createWallet() 详细流程**：
```java
// 源码：WalletTrxBusiness.java:99-136

1. 接收 TrxUserWalletRequest(userId, userName)
2. 解密 passphrase = awsKmsDriver.decryptData(passphrase)  // AWS KMS解密助记词
3. 查询是否已存在：trxWalletService.findAddressByUserId(userId)
   └→ 已存在：直接返回已有地址（幂等）
4. 生成密钥对：ECKeyPair ecKeyPair = Keys.createEcKeyPair()  // web3j
5. 创建钱包文件：Wallet.createStandard(passphrase, ecKeyPair)
6. 提取原始地址：walletFile.getAddress()  // 以太坊格式
7. 转换为TRON地址：fromHexAddress("41" + originAddress)  // 前缀41→Base58Check→T开头
8. 提取私钥hex：Numeric.toHexStringNoPrefix(ecKeyPair.getPrivateKey())
9. 加密私钥：AES.encrypt(privateKey, passphrase + address)
   └→ 密钥 = SHA-1(passphrase + address) 截取前16字节
   └→ 算法 = AES/ECB/PKCS5Padding
10. 持久化：trxWalletService.save(userId, userName, enKey, address)
11. 异步通知user-server：trxWalletService.updateUserAddress(request)
    └→ @Async POST user-server /user/updateWalletAddressById
12. 返回 WalletAddressVO(address, userId, userName)
```

**地址生成的核心技术点**：
- TRON 地址格式：以太坊地址前加 `0x41`，然后 Base58Check 编码，得到 T 开头地址
- 使用 web3j 的 ECKeyPair（secp256k1 椭圆曲线）
- 私钥加密密钥 = `SHA-1(passphrase + address)` 的前16字节 → AES-128-ECB

#### 阶段2：用户发起充值（创建订单）

```
用户在 TG Bot/Casino前端 发起充值
  → user-server 创建 ss_order 记录
     orderType = 1（充值）
     orderStatus = 50（VIRTUAL_CURRENCY_WAIT）
     virtualCurrency = "USDT"
     amount = 用户声称的充值金额
  → 前端展示用户的 TRON 地址，引导用户转账
```

**关键**：此时系统只是创建了一个"待确认"订单。用户需要自行在链上转账到自己的地址。

#### 阶段3：用户确认充值（触发余额查询）

```
用户在 TG 点击"确认已充值"
  → VirtualCurrencyController POST /virtual/charge
  → AsyncVirtualDepositService.checkCrypotcurrencyOrder()
```

**checkCrypotcurrencyOrder() 详细流程**：
```
1. 获取 Redisson Fair Lock（基于 orderId）
2. 查询 ss_order，验证状态 = VIRTUAL_CURRENCY_WAIT 且 realAmount 已设置
3. 判断 txId 是否已存在：

   【首次调用 — txId 为空】：
   a. 调用 wallet-server 查链上余额：
      POST /wallet/getBalance { userId, currency: "USDT" }
      → TrxDriver.getUserAddress(userId)  // 查用户地址
      → TrxDriver.getAccountBalance(address, "USDT")
        → WalletTrxBusiness.getTrc20Account(address, contractAddress)
          → Trc20.balanceOf(address, contract)
            → SmartContract.triggerConstantContract(...)
            → TRON RPC: triggerConstantContract (调用合约 balanceOf 方法)
            → 解码返回值得到余额
            → 除以 10^decimals 得到可读余额

   b. 验证余额：
      - 余额 = 0 → 失败（链上未收到）
      - 余额 < realAmount → 失败（金额不足）

   c. 触发归集（充值后立即把币转到公司地址）：
      POST /wallet/collectTrc20 {
        userId, ssOrderId, coinName: "USDT",
        amount: realAmount, postType: 1
      }
      → [详见第6章归集机制]
      → 返回 txId（归集交易哈希）

   d. 更新 ss_order.txId = txId
   e. 返回，等待下一次检查

   【第二次调用 — txId 已存在】：
   a. 查询归集交易状态：
      POST /wallet/getTransactionInfoById { orderId: ssOrderId }
      → TrxDriver.getTransactionInfoById(ssOrderId)
        → trxWalletOrderRepository.findBySsOrderId(ssOrderId)
        → WalletTrxBusiness.getTransactionInfoById(txId)
          → POST TRON API: /wallet/gettransactionbyid { value: txId }
          → 解析 ret[0].contractRet == "SUCCESS" ?

   b. 如果交易成功 → 进入结算
```

#### 阶段4：结算（加余额）

```
updateOrderStatusAndSendMQ()

1. 如果是试玩用户 → 清除试玩金额，转为正式用户
2. 加余额：
   userAssetsBusiness.doBalanceBusiness(
     userId, realAmount, INCOME,
     changeType=TELEGRAM_CHARGE,
     relateId=orderId
   )
   └→ Redisson Fair Lock(userId)
   └→ assets.balance += realAmount
   └→ 写 balance_change 记录（before/after）

3. 加打码量：
   playMoney += realAmount × flowMultiple

4. 更新订单：
   ss_order.orderStatus = SUCCESS(2)
   ss_order.txId = txId
   ss_order.completeTime = now

5. 发送 MQ 通知后台统计：
   RabbitMQ → user-backend-direct 交换机
   → backend_order_statistics 队列
```

### 4.3 充值链路全景时序图

```
用户(TG)          casino-web       user-server        wallet-server         TRON链
  │                  │                 │                   │                   │
  │──注册──────────→│──────────────→│                   │                   │
  │                  │                 │──POST /wallet/create──→│            │
  │                  │                 │                   │──ECKeyPair──→│  │
  │                  │                 │                   │←──地址+私钥──│  │
  │                  │                 │                   │                   │
  │                  │                 │←──地址─────────│                   │
  │←──展示充值地址──│←───────────│                   │                   │
  │                  │                 │                   │                   │
  │══用户自行在链上转 USDT 到该地址═══════════════════════════→│  链上转账    │
  │                  │                 │                   │                   │
  │──点击确认充值──→│──────────────→│                   │                   │
  │                  │                 │──POST /wallet/getBalance──→│        │
  │                  │                 │                   │──balanceOf()──→│
  │                  │                 │                   │←──余额────────│
  │                  │                 │←──余额─────────│                   │
  │                  │                 │                   │                   │
  │                  │                 │  [余额足够]       │                   │
  │                  │                 │──POST /wallet/collectTrc20──→│      │
  │                  │                 │                   │──TRC20转账──→│  │
  │                  │                 │                   │  (用户地址→公司地址)│
  │                  │                 │                   │←──txId────────│
  │                  │                 │←──txId─────────│                   │
  │                  │                 │                   │                   │
  │                  │                 │  [等待确认]       │                   │
  │                  │                 │──POST /wallet/getTransactionInfoById─→│
  │                  │                 │                   │──查交易────→│    │
  │                  │                 │                   │←──SUCCESS──│    │
  │                  │                 │←──confirmed────│                   │
  │                  │                 │                   │                   │
  │                  │                 │  [结算]           │                   │
  │                  │                 │  balance += amount│                   │
  │                  │                 │  写 balance_change│                   │
  │                  │                 │  ss_order=SUCCESS │                   │
  │                  │                 │  MQ → 后台统计    │                   │
  │←──充值成功通知──│←───────────│                   │                   │
```

### 4.4 充值链路中的关键技术细节

#### 4.4.1 余额查询的链上实现

查询 TRC20 代币余额的技术链路：

```
getBalance 请求
  → TrxDriver.getAccountBalance(address, "USDT")
    → 判断是 TRX 还是 TRC20：
      - TRX: POST trxUrl + /wallet/getaccount { address: hexAddress }
             → 解析 response.balance / 10^6
      - TRC20: Trc20.balanceOf(address, contractAddress)
             → SmartContract.triggerConstantContract(
                 ownerAddress = address,
                 contractAddress = USDT合约地址,
                 methodStr = "balanceOf(address)",
                 data = ABI编码的address参数
               )
             → 通过 gRPC 调用 TRON 节点
             → 解码返回的 Uint256 值
             → 除以 10^decimals (USDT=6位)
```

**底层调用栈**：
```java
Trc20.balanceOf(address, contract)
  → encoderBalanceOfAbi(hexAddress)     // web3j ABI编码
  → SmartContract.triggerConstantContract(...)
    → WalletApi.triggerCallContract(...)  // 构建 TriggerSmartContract protobuf
    → TronUtil.getRpcCli().triggerConstantContract(...)  // gRPC 调用
    → 解析 transactionExtention.getConstantResult(0)
  → decoderBalanceOf(result)             // web3j ABI解码 Uint256
  → balance / 10^decimals                // 转换精度
```

#### 4.4.2 地址验证

每次转账前都会验证目标地址：
```java
// WalletTrxBusiness.validateAddress()
POST trxUrl + /wallet/validateaddress { address: "T..." }
→ 检查 result=true && message="Base58check format"
```

#### 4.4.3 两种充值确认方式

1. **用户手动确认**（postType=1）：用户在TG点击"确认充值"按钮
2. **定时任务自动确认**（postType=2）：每2分钟扫描30分钟内的待处理订单

两种方式最终走同一条处理链路，只是触发源不同。

### 4.5 充值链路中的核心挑战与解决方式

| 挑战 | tg-services 的解决方式 | 评价 |
|------|------------------------|------|
| 如何知道用户充了多少？ | 查询链上余额（balanceOf） | 简单粗暴但有效，缺点是无法区分多笔 |
| 充值金额如何确认？ | 用户在前端填写 realAmount，系统查链上余额验证 | 依赖用户填写，有误差风险 |
| 如何防止重复入账？ | balance_change 的 UNIQUE KEY | DB级幂等，可靠 |
| 链上交易确认？ | 查询 gettransactionbyid 的 contractRet | 标准做法 |
| 资金安全（归集）？ | 充值后立即归集到公司地址 | 减少用户地址资金暴露时间 |
| 定时补偿？ | 每2分钟扫描待处理订单 | 兜底机制，防止用户不点确认 |

---

## 第5章 区块链提现完整链路

### 5.1 提现模型概述

```
1. 用户发起提现请求，填写目标地址和金额
2. user-server 冻结余额，创建 ss_order
3. 调用 wallet-server 发起链上转账（从公司下发地址 → 用户指定地址）
4. wallet-server 构建交易 → 调用外部签名服务签名 → 广播上链
5. 定时任务轮询确认 → 成功后扣减余额
```

### 5.2 提现链路详解

#### 阶段1：发起提现

```
用户在 TG 发起提现
  → VirtualCurrencyController POST /virtual/withdraw
  → 预检查：
    - 订单存在且 status = VIRTUAL_CURRENCY_WAIT(50)
    - 用户存在
    - Redis 防重（10秒冷却）
  → VirtualCurrencyWithdrawBusiness.withdraw()
```

#### 阶段2：冻结 + 链上转账

```
withdraw() 详细流程：

1. 查询订单，验证 txId 为空（未处理过）
2. Redis 防重放：SET order::withdraw_{orderId} (TTL=30天)
3. 检查订单年龄 < 30分钟（超时让定时任务处理）
4. 调用 wallet-server：
   POST /wallet/withdrawTraction {
     userId, toAddress: payeeAddress,
     amount: realAmount - feeAmount,  // 扣除手续费后的实际转出金额
     coinName: "USDT",
     ssOrderId: orderId
   }

5. wallet-server 内部处理：
   TrxDriver.withdrawTraction()

   a. 从 TrxWalletConfig 读取下发地址（disPendAddress）
   b. 幂等检查：findBySsOrderId(ssOrderId)，已存在直接返回 txId
   c. 查询下发地址链上余额，验证充足
   d. 如果是 TRC20 且下发地址 TRX 不够手续费：
      → 从 gasAddress 转 30 TRX 到下发地址作为手续费
      → 记录手续费转账订单
   e. 构建 TRC20 转账交易：
      → trc20Transaction() [详见第8章]
      → 因为是从下发地址转出 → 走外部签名服务
      → 签名 → 广播
   f. 记录提现订单到 trx_wallet_order
   g. 返回 txId

6. user-server 冻结余额：
   freezeAmount += realAmount
```

#### 阶段3：确认 + 结算

```
定时任务 HandleVirtualCurrencyTask（每2分钟）
  → checkVirtualWithdrawTask()

1. 扫描30分钟内 status=VIRTUAL_CURRENCY_WAIT 的提现订单
2. 对每个订单获取 Redisson Fair Lock(orderId)
3. 查询交易状态：
   POST /wallet/getTransactionInfoById { orderId: ssOrderId }

4. 如果交易成功（transResult=true）：
   withdrawUSDT()
   a. 解冻：freezeAmount -= realAmount
   b. 扣减余额：balance -= realAmount（变动类型=TELEGRAM_WITHDRAW）
   c. 扣减手续费：balance -= feeAmount（变动类型=FEE）
   d. 更新订单 status=SUCCESS, txId, completeTime
   e. MQ 通知后台统计

5. 如果交易失败或超时：
   取消订单，解冻余额
```

### 5.3 提现链路时序图

```
用户(TG)        user-server          wallet-server         签名服务        TRON链
  │                │                     │                   │              │
  │──发起提现────→│                     │                   │              │
  │                │──冻结余额──────→│  │                   │              │
  │                │                     │                   │              │
  │                │──POST /wallet/withdrawTraction──→│     │              │
  │                │                     │                   │              │
  │                │                     │──查下发地址余额───────────────→│
  │                │                     │←──余额足够─────────────────│
  │                │                     │                   │              │
  │                │                     │──构建TRC20交易──→│             │
  │                │                     │                   │              │
  │                │                     │──RSA加密签名请求─→│             │
  │                │                     │←──签名值───────│             │
  │                │                     │                   │              │
  │                │                     │──广播交易───────────────────→│
  │                │                     │←──txId──────────────────────│
  │                │                     │                   │              │
  │                │←──txId──────────│                   │              │
  │←──提现处理中──│                     │                   │              │
  │                │                     │                   │              │
  │    [定时任务，每2分钟]               │                   │              │
  │                │──查交易状态──────→│                   │              │
  │                │                     │──查链上确认─────────────────→│
  │                │                     │←──SUCCESS────────────────────│
  │                │←──confirmed───────│                   │              │
  │                │                     │                   │              │
  │                │──解冻+扣余额───→│                     │              │
  │                │──写流水────────→│                     │              │
  │                │──更新订单=SUCCESS│                     │              │
  │←──提现成功通知│                     │                   │              │
```

### 5.4 充值 vs 提现的关键差异

| 维度 | 充值 | 提现 |
|------|------|------|
| 资金方向 | 链上 → 用户地址 → 公司地址 | 公司下发地址 → 用户指定地址 |
| 触发方 | 用户自行链上转账 | 系统主动发起链上转账 |
| 签名方式 | 归集用用户私钥本地签名 | 提现用外部签名服务（更安全） |
| 余额操作 | 先确认后加余额 | 先冻结，确认后扣余额 |
| 手续费 | 归集手续费由gas地址垫付 | 从用户提现金额中扣除 |
| 失败处理 | 不加余额，订单取消 | 解冻余额，订单取消 |

---

## 第6章 归集机制（Collection）

### 6.1 什么是归集

归集（Collection）= 把分散在各用户地址上的代币**统一转到公司主地址**。

**为什么要归集**：
1. **安全性**：用户地址的私钥分散存储，归集到主地址后统一管理
2. **资金池**：提现时从主地址出款，需要有足够资金
3. **成本控制**：减少后续转账的手续费计算复杂度

### 6.2 单笔归集流程

```
TrxDriver.singleCollect() 详细流程：

输入：fromAddress(用户地址), userId, coinName, ssOrderId, reChargeAmount, collectCode, postType

1. 解密公司主地址：awsKmsDriver.decryptData(companyAddress)
2. 从DB读取下发地址：trxWalletConfigRepository.findByType("sign")
3. 确定归集金额：
   - 如果 reChargeAmount != null → 使用传入金额
   - 如果 reChargeAmount == null → 查询链上实际余额
4. 验证余额 > 0

5. 【关键步骤】如果是 TRC20 代币（非 TRX 原生币）：
   a. 查询用户地址的 TRX 余额
   b. 如果 TRX < 30 TRX（FEE_BAL）：
      → 从 gasAddress 转 30 TRX 到用户地址
      → 作为归集交易的手续费（能量/带宽费用）
      → 记录这笔手续费转账到 trx_wallet_order

   这一步非常关键：TRC20 转账需要 TRX 作为手续费，
   但用户地址可能只有 USDT 没有 TRX，
   所以系统先垫付 TRX 手续费。

6. 执行归集转账：
   createTransactionForTrx(null, balance, fromAddress, companyAddress, tokenContract, ssOrderId)

   → 因为 privateKey=null，会从DB读取用户私钥并解密
   → 调用 trc20Transaction() 构建+签名+广播
   → 注意：归集使用本地签名（用户私钥），不走外部签名服务

7. 记录归集：
   → trx_wallet_collect 表（归集记录）
   → trx_wallet_order 表（交易记录，orderType=RECHARGE）
```

### 6.3 批量归集（定时任务）

```
CollectTrxJob.collectUSDT()
  → 每天曼谷时间 8:00 AM 执行
  → TrxDriver.collectTrc20()
    → 查询所有用户钱包：trxWalletService.findAll()
    → 遍历每个用户钱包：
      - 跳过公司主地址本身
      - 对每个地址执行 singleCollect(postType=2)
      - 每个归集生成独立的 UUID 作为 ssOrderId
```

### 6.4 归集流程图

```
                    ┌─────────────┐
                    │ 触发归集     │
                    │ (手动/定时)  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ 查询用户地址  │
                    │ 链上代币余额  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ 余额 > 0 ?  │──否──→ 跳过
                    └──────┬──────┘
                           │ 是
                    ┌──────▼──────┐
                    │ 是TRC20代币? │──否──→ 直接TRX转账
                    └──────┬──────┘
                           │ 是
                    ┌──────▼──────┐
                    │ 用户地址     │
                    │ TRX >= 30?  │──是──→ 直接TRC20转账
                    └──────┬──────┘
                           │ 否
                    ┌──────▼──────┐
                    │ gas地址转    │
                    │ 30 TRX到    │
                    │ 用户地址     │
                    │ (垫付手续费) │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ TRC20转账    │
                    │ 用户地址→    │
                    │ 公司主地址   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ 记录归集表   │
                    │ 记录订单表   │
                    └─────────────┘
```

### 6.5 TRC20 归集的手续费垫付机制（重要参考）

这是区块链钱包的一个**通用问题**：

> TRC20 代币转账需要消耗 TRX 作为能量/带宽费用。
> 但用户充入的可能只有 USDT，地址上没有 TRX。
> 解决方案：由平台的 gas 地址垫付 TRX 手续费。

**tg-services 的实现**：
```
gasAddress（平台gas地址，预存大量TRX）
  → 转 30 TRX 到用户地址（作为手续费）
  → 然后用户地址才能执行 TRC20 转账

费用估算：
- TRC20 转账约需 5-15 TRX 的能量费
- 这里固定垫付 30 TRX（偏高，有一定浪费）
- FEE_LIMIT = 50,000,000 SUN = 50 TRX（交易最大消耗）
```

**对我们的启示**：
1. 必须有一个专门的 gas 垫付地址
2. 垫付金额应该可配置，不应硬编码
3. 可以考虑按实际需要计算垫付金额，而非固定值
4. 垫付记录必须入账记录

---

## 第7章 钱包地址生成与密钥管理

### 7.1 地址生成技术细节

```
TRON 地址生成流程（与以太坊兼容）：

1. 生成 secp256k1 椭圆曲线密钥对
   ECKeyPair ecKeyPair = Keys.createEcKeyPair()  // web3j

2. 从公钥派生以太坊格式地址
   WalletFile walletFile = Wallet.createStandard(passphrase, ecKeyPair)
   String ethAddress = walletFile.getAddress()  // 40位hex，无0x前缀

3. 转换为 TRON 地址
   String tronHex = "41" + ethAddress  // TRON前缀 = 0x41
   String tronAddress = Base58Check.encode(tronHex)  // T开头，如 TRrQtwH9...

4. 提取私钥
   String privateKeyHex = Numeric.toHexStringNoPrefix(ecKeyPair.getPrivateKey())
   // 64位hex字符串
```

### 7.2 密钥加密体系（三层）

```
┌─────────────────────────────────────────────┐
│ 第1层：用户私钥加密                          │
│ 算法：AES-128-ECB/PKCS5Padding              │
│ 密钥：SHA-1(passphrase + address)[0:16]     │
│ 存储：trx_wallet.en_private_key             │
│ 解密需要：passphrase（助记词） + address      │
├─────────────────────────────────────────────┤
│ 第2层：助记词/公司地址等敏感配置加密          │
│ 算法：AWS KMS（信封加密）                     │
│ 密钥：AWS KMS Key (mrk-2017...)             │
│ 存储：application.yml 中的密文               │
│ 解密需要：AWS accessKey + secretKey + keyId  │
├─────────────────────────────────────────────┤
│ 第3层：签名服务通信加密                       │
│ 算法：RSA-512                                │
│ 公钥：trx_wallet_config 表 (type=sign,code=rsa) │
│ 用途：与外部签名服务通信时加密请求体          │
└─────────────────────────────────────────────┘
```

### 7.3 密钥管理的安全缺陷

| 缺陷 | 严重性 | 说明 |
|------|--------|------|
| AES-ECB 模式 | **高** | ECB是最不安全的分组密码模式，相同明文产生相同密文，应使用CBC或GCM |
| SHA-1 派生密钥 | **中** | SHA-1已被攻破，应使用PBKDF2/Argon2等密钥派生函数 |
| RSA-512 密钥 | **高** | 512位RSA已可被暴力破解，应至少使用2048位 |
| passphrase 全局共享 | **中** | 所有用户私钥用同一个 passphrase 加密，泄露影响面大 |
| AWS 凭证明文配置 | **高** | accessKeyId 和 secretAccessKey 明文存在 application.yml |
| 私钥解密在内存 | **中** | 每次转账都解密私钥到内存，存在内存泄露风险 |

---

## 第8章 链上交互层（TRON RPC）

### 8.1 两套链上交互方式

tg-services 同时使用了**两套方式**与 TRON 区块链交互：

#### 方式1：HTTP API（TronGrid）

```
基础URL：https://nile.trongrid.io（测试网）
       https://api.trongrid.io（主网）

使用场景：
- 查询账户余额：POST /wallet/getaccount
- 验证地址：POST /wallet/validateaddress
- 触发智能合约（TRC20转账）：POST /wallet/triggersmartcontract
- 广播交易：POST /wallet/broadcasthex
- 查询交易状态：POST /wallet/gettransactionbyid

通信方式：HttpClient4Util.doPost(url, jsonBody)
```

#### 方式2：gRPC（wallet-cli）

```
初始化：GrpcClient rpcCli = WalletApi.init()
配置文件：config.conf (grpc.nile.trongrid.io:50051)

使用场景：
- 触发常量合约调用（查询余额）：rpcCli.triggerConstantContract(...)
- 触发合约调用（转账）：rpcCli.triggerContract(...)
- 广播交易：rpcCli.broadcastTransaction(...)
- 验证签名权重：WalletApi.getTransactionSignWeight(...)

通信方式：Protobuf + gRPC
```

### 8.2 TRC20 转账的完整技术链路

```java
// WalletTrxBusiness.trc20Transaction()
// 源码位置：WalletTrxBusiness.java:209-290

步骤1：验证目标地址
  validateAddress(to) → POST /wallet/validateaddress

步骤2：构建转账参数
  contract_address = toHexAddress(tokenContract)  // USDT合约地址转hex
  function_selector = "transfer(address,uint256)"  // ERC20标准方法
  parameter = ABI编码(目标地址, 金额 × 10^decimals)
  owner_address = toHexAddress(from)               // 发送方地址转hex
  call_value = 0                                    // 不转TRX
  fee_limit = 50,000,000                           // 最大50TRX手续费

步骤3：触发智能合约（获取未签名交易）
  POST trxUrl + /wallet/triggersmartcontract
  → 返回 { transaction: { raw_data: {...}, ... } }

步骤4：打包交易
  transaction.raw_data.data = hex("trc20")  // 添加交易备注
  Protocol.Transaction tx = TronUtils.packTransaction(json, false)
  → 将JSON格式的交易数据反序列化为Protobuf的Transaction对象

步骤5：签名（分两条路径）
  if (从下发地址转出) {
    // 提现 → 走外部签名服务（安全级别更高）
    bytes = sendSign(signUrl, tx, ssOrderId, from)
    // [详见第9章]
  } else {
    // 归集 → 本地签名（用用户私钥）
    bytes = signTransaction2Byte(tx.toByteArray(), ByteArray.fromHexString(privateKey))
    // ECKey.fromPrivate(privateKey) → 对交易hash签名 → 附加签名到交易
  }

步骤6：广播交易
  POST trxUrl + /wallet/broadcasthex { transaction: hexSignedTx }
  → 返回 { result: true, txid: "xxx", code: "SUCCESS" }

步骤7：返回 txId
```

### 8.3 TRX 原生币转账链路

```java
// WalletTrxBusiness.trxOfflineSignature()
// 源码位置：WalletTrxBusiness.java:323-362

步骤1：地址转换
  fromAddress = WalletApi.decodeFromBase58Check(from)  // T→41hex→bytes
  toAddress = WalletApi.decodeFromBase58Check(to)

步骤2：金额转换
  amount = needSendAmount × 10^6  // TRX精度=6，1 TRX = 1,000,000 SUN

步骤3：创建交易（通过wallet-cli SDK）
  transaction = createTransaction(fromAddress, toAddress, amount.longValueExact())
  // 这个方法来自 org.tron.demo.TransactionSignDemo

步骤4：签名
  bytes = sendSign(signUrl, transaction, ssOrderId, from)  // 外部签名

步骤5：广播
  success = WalletApi.broadcastTransaction(bytes)  // gRPC广播

步骤6：计算txId
  txId = SHA256(transaction.rawData)
```

### 8.4 ABI 编码细节

```java
// TRC20 transfer 的 ABI 编码
// 函数签名：transfer(address,uint256)

encoderAbi(hexAddress, amount):
  inputParameter = [
    new Address(hexAddress),     // 目标地址，32字节左填充0
    new Uint256(amount)          // 金额，32字节
  ]
  return FunctionEncoder.encodeConstructor(inputParameter)
  // 输出：64位hex的address + 64位hex的amount
  // 注意：不包含函数选择器（4字节），由 triggersmartcontract API 的 function_selector 参数指定
```

---

## 第9章 外部签名系统

### 9.1 签名系统架构

```
wallet-server 不直接持有提现用的私钥（下发地址的私钥）。
提现交易的签名通过独立的"签名服务"完成。

┌──────────────┐    RSA加密     ┌──────────────┐
│ wallet-server│──────────────→│  签名服务      │
│              │    请求体      │ (34.92.121.9) │
│              │←──────────────│              │
│              │    签名值      │              │
└──────────────┘               └──────────────┘
```

### 9.2 签名请求流程

```java
// WalletTrxBusiness.sendSign()
// 源码位置：WalletTrxBusiness.java:292-321

1. 计算交易哈希：
   txId = SHA256SM3(tx.getRawData())
   signTxid = Base64(txId)

2. 构建签名请求：
   {
     "orderNo": ssOrderId,
     "outAddressNo": fromAddress,
     "chainType": "TRON",
     "hashId": signTxid  // Base64编码的交易哈希
   }

3. RSA加密请求体：
   从 trx_wallet_config 读取 RSA 公钥 (type=sign, code=rsa)
   enSign = RSAUtil.encrypt(JSON.toJSONString(signMap), rsaPublicKey)

4. 发送签名请求：
   POST signUrl + /api/admin/chain/sign
   Header:
     Content-Type: application/json
     PLAT_CODE: <商户号> (从trx_wallet_config读取)
   Body: RSA加密后的密文

5. 解析响应：
   {
     "code": "200",
     "data": "<Base64编码的签名值>"
   }

6. 组装完整交易：
   deSign = Base64.decode(response.data)
   fullTx = tx.toBuilder()
               .addSignature(ByteString.copyFrom(deSign))
               .build()
               .toByteArray()
```

### 9.3 签名系统设计分析

**优点**：
1. **私钥隔离**：提现用的私钥不在 wallet-server 中，即使 wallet-server 被攻破也无法直接提币
2. **审计追踪**：签名服务可以独立记录每次签名请求
3. **RSA加密通信**：签名请求体经过RSA加密，防中间人

**缺陷**：
1. RSA-512 密钥太短，不安全
2. 没有双向认证（签名服务不验证wallet-server的身份）
3. 没有看到签名服务的限额/风控逻辑
4. HTTP通信（非HTTPS），签名服务IP是公网IP

---

## 第10章 定时任务体系

### 10.1 wallet-server 定时任务

| 任务 | 触发时间 | 功能 |
|------|----------|------|
| CollectTrxJob.collectUSDT() | 每天 08:00 曼谷时间 | 遍历所有用户钱包，批量归集 USDT |

**配置**：
- `@Scheduled(cron = "0 0 8 * * ?", zone="Asia/Bangkok")`
- 线程池大小 = 10（ScheduleConfig）

### 10.2 user-server 定时任务

| 任务 | 触发频率 | 功能 |
|------|----------|------|
| HandleVirtualCurrencyTask.deposit() | 每2分钟 | 扫描30分钟内待确认的充值订单 |
| HandleVirtualCurrencyTask.checkVirtualWithdrawTask() | 每2分钟 | 扫描30分钟内待确认的提现订单 |
| HandleVirtualCurrencyTask.cancelDeposit() | 每2分钟 | 取消超过30分钟未完成的充值订单 |

### 10.3 定时任务设计分析

**优点**：
1. 充值/提现都有定时补偿机制，不完全依赖用户主动触发
2. 30分钟超时取消机制，防止订单永远挂起

**缺陷**：
1. 批量归集没有并发控制——如果某个地址归集失败，不会影响其他地址，但也没有重试
2. 归集频率只有每天一次（08:00），两次归集之间的充值资金暴露在用户地址上
3. 定时任务粒度固定（2分钟），不能根据业务量自适应
4. 没有分布式任务锁，多实例部署会重复执行

---

## 第11章 跨服务调用与依赖关系

### 11.1 服务间调用清单

| 调用方 | 被调方 | 接口 | 方式 | 用途 |
|--------|--------|------|------|------|
| user-server | wallet-server | POST /wallet/create | HTTP | 创建钱包地址 |
| user-server | wallet-server | POST /wallet/getBalance | HTTP | 查链上余额 |
| user-server | wallet-server | POST /wallet/collectTrc20 | HTTP | 触发充值归集 |
| user-server | wallet-server | POST /wallet/withdrawTraction | HTTP | 发起提现转账 |
| user-server | wallet-server | POST /wallet/getTransactionInfoById | HTTP | 查交易状态 |
| wallet-server | user-server | POST /user/updateWalletAddressById | HTTP @Async | 回写钱包地址 |
| wallet-server | TRON RPC | gRPC + HTTP | gRPC/HTTP | 链上操作 |
| wallet-server | 签名服务 | POST /api/admin/chain/sign | HTTP | 提现签名 |
| game-server | user-server | RabbitMQ | MQ | 结算通知 |
| user-server | casino-backend | RabbitMQ | MQ | 统计通知 |

### 11.2 服务间通信方式评价

**现状**：全部使用 HTTP REST + 硬编码 URL

```java
// user-server 中硬编码的 wallet-server 地址
@Value("${wallet.server-url}")  // = http://192.168.30.244:9600
```

**问题**：
1. 没有服务发现——IP/端口变更需要重新部署
2. 没有负载均衡——单点故障风险
3. 没有熔断/降级——wallet-server 挂了，user-server 也会卡住
4. 没有重试机制——网络抖动可能导致操作丢失

### 11.3 依赖关系图

```
┌──────────────────────────────────────────────────┐
│                    外部系统                        │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────┐ │
│  │ TRON 区块链  │  │ AWS KMS  │  │  签名服务    │ │
│  └──────┬──────┘  └─────┬────┘  └──────┬──────┘ │
│         │               │              │         │
│  ┌──────┴───────────────┴──────────────┴──────┐ │
│  │            wallet-server (9600)             │ │
│  │  TrxDriver ← WalletTrxBusiness ← Trc20    │ │
│  └──────────────────┬─────────────────────────┘ │
│                     │ HTTP REST                   │
│  ┌──────────────────┴─────────────────────────┐ │
│  │            user-server (9300)               │ │
│  │  VirtualCurrencyController                  │ │
│  │  AsyncVirtualDepositService                 │ │
│  │  AsyncVirtualWithdrawService                │ │
│  │  UserAssetsBusiness                         │ │
│  └──────────────────┬─────────────────────────┘ │
│                     │ RabbitMQ                    │
│  ┌──────────────────┴─────────────────────────┐ │
│  │     casino-web / casino-backend             │ │
│  │     game-server                             │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
│  基础设施：MySQL + Redis + RabbitMQ              │
└──────────────────────────────────────────────────┘
```

---

## 第12章 枚举与类型系统

### 12.1 代币枚举（TRXContractEnum）

```java
TRX(1, "trx",                              "TRX",  6,  "TRX")    // 原生币
USDT(2, WalletUtil.getContractByCoin("USDT"), "USDT", 6,  "TRC20") // USDT-TRC20
CXT(3, "TBfSX66SpPig...",                   "CXT",  18, "TRC20")  // 自定义代币
```

**关键字段**：
- `code`：数字编码
- `contract`：合约地址（TRX用"TRX"标识，TRC20用实际合约地址）
- `contractName`：可读名称
- `decimal`：精度（TRX=6, USDT=6, CXT=18）
- `type`：协议类型（TRX/TRC10/TRC20）

**亮点**：USDT合约地址从YAML动态读取（`WalletUtil.getContractByCoin("USDT")`），便于切换网络。

### 12.2 订单类型枚举

```java
// TrxOrderTypeEnum
RECHARGE(1, "充值")   // 链上收到币
WITHDRAW(2, "提现")   // 链上发出币

// CollectTypeEnum
SINGLE(1)   // 单笔归集（充值后立即触发）
COLLECT(2)  // 批量归集（定时任务）
```

### 12.3 user-server 中的关联枚举

```java
// BalanceChangeEnum（资金变动类型）
TELEGRAM_CHARGE(8)     // TG虚拟币充值
TELEGRAM_WITHDRAW(4)   // TG虚拟币提现
FEE(90)                // 手续费

// OrderStatusEnum
ORDER_WAIT(1)                   // 待处理
ORDER_SUCCESS(2)                // 成功
ORDER_FAIL(3)                   // 失败
ORDER_SYSTEM_CANCLE(4)          // 系统取消
VIRTUAL_CURRENCY_WAIT(50)       // 虚拟币待处理（专用状态）
```

---

## 第13章 接口设计（API清单）

### 13.1 wallet-server REST API（6个端点）

| # | 方法 | 路径 | 功能 | 请求 | 响应 |
|---|------|------|------|------|------|
| 1 | POST | /wallet/create | 创建钱包地址 | TrxUserWalletRequest(userId, userName) | WalletAddressVO(address, userId, userName) |
| 2 | POST | /wallet/withdrawTraction | TRC提现 | TrxCreateTransactionRequest(userId, toAddress, amount, coinName, ssOrderId, postType) | TrxWalletResult(txId) |
| 3 | POST | /wallet/getBalance | 查询链上余额 | UserBalanceRequest(userId, currency) | BigDecimal |
| 4 | POST | /wallet/collectTrc20 | 单笔归集 | TrcCollectRequest(userId, ssOrderId, coinName, amount, postType) | TrxWalletResult(txId) |
| 5 | POST | /wallet/getTransactionInfoById | 查询交易状态 | orderId(query) | TransactionVO(transResult, txId) |
| 6 | POST | /wallet/getTrxWalletAccount | 加密钱包信息 | TrxWalletAccountRequest | TrxWalletAccountVO(encrypted fields) |

### 13.2 接口设计分析

**优点**：
- 接口职责清晰，每个接口做一件事
- 使用 Swagger 注解便于文档生成

**缺陷**：
- 全部是 POST，没有遵循 RESTful 规范（查询应该用 GET）
- 没有统一的错误码体系——失败返回 null 或空对象
- 没有认证/鉴权——WebSecurityConfig 禁用了所有安全检查
- getTransactionInfoById 用 query param 而非 request body，风格不一致
- 没有分页、排序等查询参数支持

---

## 第14章 并发控制与安全机制分析

### 14.1 wallet-server 的并发控制

**结论：wallet-server 几乎没有并发控制。**

| 场景 | 保护机制 | 评价 |
|------|----------|------|
| 创建钱包 | 查询后创建（check-then-act） | **不安全**：无锁无唯一索引，并发可能创建多个 |
| 提现 | ssOrderId 查询幂等（findBySsOrderId） | **不完整**：查询和写入不在同一事务中 |
| 归集 | 无 | **不安全**：定时任务和手动归集可能并发 |
| 余额查询 | 无 | 只读操作，无需控制 |

### 14.2 user-server 的并发控制

user-server 的并发控制**相对完善**：

| 机制 | 实现 | 用途 |
|------|------|------|
| Redisson Fair Lock(orderId) | 公平锁，30秒超时 | 充值/提现订单处理串行化 |
| Redisson Fair Lock(userId) | 公平锁 | 余额变更串行化 |
| Redis 防重（10秒冷却） | SET key TTL=10s | 防止快速重复提交 |
| Redis 防重放（30天） | SET key TTL=30d | 防止历史订单被重复处理 |
| 乐观锁（assets.version） | JPA @Version | 余额并发更新检测 |
| balance_change UNIQUE KEY | DB级唯一约束 | 防止重复入账 |

### 14.3 安全机制总结

| 层次 | 机制 | 评价 |
|------|------|------|
| 密钥存储 | AES加密私钥 + AWS KMS | 思路正确，但AES-ECB不安全 |
| 签名隔离 | 提现走外部签名服务 | 好的设计，私钥不出签名服务 |
| 通信加密 | RSA加密签名请求 | RSA-512太短，应至少2048位 |
| 认证鉴权 | 禁用（WebSecurityConfig permits all） | **严重缺陷**——任何人可调用 |
| 审计日志 | balance_change 表 | 有before/after，但wallet-server无审计 |
| 防重入账 | UNIQUE KEY on balance_change | DB级保障，可靠 |

---

## 第15章 核心缺陷与不足清单

### 15.1 安全类缺陷（P0）

| # | 缺陷 | 位置 | 影响 | 建议 |
|---|------|------|------|------|
| S1 | AES-ECB 模式加密私钥 | AES.java:19 | 相同私钥产生相同密文，可被模式分析攻破 | 改用 AES-256-GCM |
| S2 | SHA-1 派生密钥 | AES.java:21-26 | SHA-1已被碰撞攻击，密钥安全性不足 | 改用 PBKDF2 或 Argon2 |
| S3 | RSA-512 密钥 | RSAUtil.java:16 | 512位RSA可被暴力破解 | 改用 RSA-2048+ |
| S4 | AWS凭证明文存储 | application.yml:92-94 | 配置文件泄露=AWS全面沦陷 | 使用环境变量或Vault |
| S5 | 全局禁用认证 | WebSecurityConfig | 任何人可调用API | 至少加IP白名单 |
| S6 | 数据库密码明文 | application.yml:58 | MySQL/Redis/RabbitMQ密码全明文 | 加密或环境变量 |

### 15.2 数据一致性缺陷（P0）

| # | 缺陷 | 位置 | 影响 | 建议 |
|---|------|------|------|------|
| D1 | trx_wallet.user_id 无 UNIQUE | TrxWalletEntity | 并发创建可能产生多个钱包 | 加 UNIQUE 约束 |
| D2 | trx_wallet_order.ss_order_id 无 UNIQUE | TrxWalletOrderEntity | 同一订单可能被重复记录 | 加 UNIQUE 约束 |
| D3 | 幂等检查与写入非原子 | TrxDriver.withdrawTraction():298-303 | check-then-act 竞态条件 | 使用 INSERT IGNORE 或分布式锁 |
| D4 | 归集无并发控制 | TrxDriver.singleCollect() | 定时任务和手动归集可能双重执行 | 加分布式锁 |
| D5 | DECIMAL(16,2) 精度不足 | 所有金额字段 | USDT有6位精度，只存2位会丢失 | 使用BIGINT最小单位存储 |

### 15.3 架构设计缺陷（P1）

| # | 缺陷 | 位置 | 影响 | 建议 |
|---|------|------|------|------|
| A1 | 无接口抽象 | 全局 | 无法Mock测试，无法替换实现 | 定义 interface |
| A2 | 无自定义异常 | 全局 | try-catch Exception 无法区分错误类型 | 定义业务异常体系 |
| A3 | 无交易状态追踪 | trx_wallet_order | 不知道链上交易是pending还是confirmed | 增加status字段 |
| A4 | 无归集目标地址记录 | trx_wallet_collect | 无法审计归集去向 | 增加to_address字段 |
| A5 | 硬编码URL | 全局 | 地址变更需要重新部署 | 服务发现/配置中心 |
| A6 | 静态方法过多 | WalletTrxBusiness | 不利于单元测试 | 改为实例方法 |
| A7 | 无分布式任务锁 | CollectTrxJob | 多实例部署会重复归集 | 使用分布式锁 |
| A8 | 手续费固定30TRX | TrxDriver:93 | 浪费TRX | 动态计算或可配置 |
| A9 | 归集频率太低 | CollectTrxJob | 每天一次，资金长时间暴露 | 提高频率或事件驱动 |

### 15.4 代码质量缺陷（P2）

| # | 缺陷 | 位置 | 说明 |
|---|------|------|------|
| C1 | 大量注释代码 | TrxDriver, WalletTrxBusiness | 未清理的旧代码 |
| C2 | 异常被吞 | WalletTrxBusiness.createWallet():131 | catch后只打日志，返回空对象 |
| C3 | main方法遗留 | WalletTrxBusiness:138, Trc20:112, RSAUtil:79 | 测试代码未清理 |
| C4 | 变量命名不规范 | tixId vs txId | 拼写不一致 |
| C5 | findBySsOrderId 硬编码 USDT | TrxWalletOrderRepository:19-21 | SQL中 `coin_name = 'USDT'` 硬编码 |

---

## 第16章 借鉴/调整/规避决策表

### 16.1 值得借鉴的设计（直接参考）

| # | 设计点 | 出处 | 借鉴理由 |
|---|--------|------|----------|
| B1 | **链上钱包与账务分离** | 架构设计 | wallet-server 只管链上操作，user-server 管余额——职责隔离是正确的。我们的 ser-wallet 也应该把"链上交互"和"余额管理"分开 |
| B2 | **用户独立地址模式** | trx_wallet | 每个用户一个独立链上地址，便于充值追踪和归集管理。这是行业标准做法 |
| B3 | **TRC20 手续费垫付机制** | TrxDriver.singleCollect() | 归集TRC20代币前先从gas地址转TRX作为手续费——这是必须的设计，我们也需要 |
| B4 | **归集机制** | singleCollect + CollectTrxJob | 充值后归集到公司主地址——减少私钥暴露风险，集中管理资金 |
| B5 | **签名服务隔离** | sendSign() | 提现私钥不在业务服务中，走独立签名服务——安全架构的最佳实践 |
| B6 | **ssOrderId 关联** | trx_wallet_order.ssOrderId | 链上交易与业务订单通过 ssOrderId 关联——跨服务追踪的关键纽带 |
| B7 | **定时补偿机制** | HandleVirtualCurrencyTask | 充值/提现都有定时任务兜底，不依赖单次调用成功 |
| B8 | **balance_change UNIQUE KEY** | user-server | (changeType, relateId, userId, orderType) 防重复入账——DB级幂等保障 |
| B9 | **冻结→确认→扣减** 提现流程 | AsyncVirtualWithdrawService | 先冻结余额 → 链上确认 → 再扣减——避免链上失败后余额已扣的问题 |
| B10 | **代币合约地址可配置** | TRXContractEnum + WalletUtil | USDT合约地址从配置读取，便于切换测试网/主网 |

### 16.2 需要调整的设计（借鉴思路，改进实现）

| # | 设计点 | 问题 | 调整方案 |
|---|--------|------|----------|
| J1 | 充值确认方式 | 依赖用户手动填写 realAmount + 点击确认 | 应主动监听链上事件（区块扫描），自动检测充值到账 |
| J2 | 手续费固定30TRX | 浪费TRX，且链上能量价格会变 | 改为动态计算：查询当前能量价格 × 预估消耗，加上安全余量 |
| J3 | 归集频率 | 每天一次太低，充值后资金暴露在用户地址 | 充值确认后立即归集（已有单笔归集），定时任务作为兜底 |
| J4 | 金额精度 | DECIMAL(16,2) 只有2位小数 | 改用 BIGINT 最小单位存储（USDT: 1单位=0.000001 USDT），全链路整数计算 |
| J5 | 余额查询方式 | 每次查链上（慢） | 充值确认后缓存余额，提现扣减后更新缓存，链上查询作为校验手段 |
| J6 | 交易状态管理 | 无状态字段 | trx_wallet_order 增加 status 字段（pending→confirmed→failed），记录完整生命周期 |
| J7 | 配置管理 | AWS KMS密文在yml，sign配置在DB | 统一配置管理：敏感配置统一加密存储，使用配置中心（ETCD/Consul） |
| J8 | 服务间通信 | HTTP REST + 硬编码URL | 改用 Kitex RPC + ETCD 服务发现，自带重试/熔断 |
| J9 | 地址生成的web3j | Java生态（web3j） | Go生态需要对应的库：go-ethereum 或 gotron-sdk |

### 16.3 必须规避的设计（有缺陷，不能照搬）

| # | 设计点 | 缺陷 | 规避方案 |
|---|--------|------|----------|
| G1 | AES-ECB 加密 | 不安全的分组模式 | 使用 AES-256-GCM（带认证的加密） |
| G2 | SHA-1 密钥派生 | SHA-1已被攻破 | 使用 PBKDF2/scrypt/Argon2 |
| G3 | RSA-512 | 可被暴力破解 | RSA-2048+ 或改用 ECDSA |
| G4 | user_id 无 UNIQUE | 并发重复创建 | user_id + address 都加 UNIQUE 约束 |
| G5 | ss_order_id 无 UNIQUE | 重复订单记录 | order_no 全局唯一索引 |
| G6 | 幂等 check-then-act | 竞态条件 | INSERT ... ON CONFLICT 或分布式锁 |
| G7 | 认证全禁用 | 裸奔的API | 至少IP白名单 + 服务间Token认证 |
| G8 | AWS凭证明文 | 泄露即全面沦陷 | 环境变量 / Vault / KMS自身管理 |
| G9 | 异常被吞 | catch后返回null/空对象 | 定义业务异常，向上层传递明确错误 |
| G10 | 无审计日志（wallet-server） | 链上操作无法审计 | 每次链上操作写审计日志 |

---

## 附录A 源码文件清单

### wallet-server 核心文件（44个Java文件）

| 层级 | 文件 | 行数 | 核心职责 |
|------|------|------|----------|
| 入口 | WalletServerApplication.java | ~15 | Spring Boot启动类 |
| Controller | WalletController.java | 143 | 6个REST端点 |
| Controller | TestController.java | ~30 | 测试端点 |
| Driver | **TrxDriver.java** | 338 | ★ 业务编排中枢 |
| Driver | AwsKmsDriver.java | 110 | AWS KMS加解密 |
| Business | **WalletTrxBusiness.java** | 538 | ★ 核心链上交互 |
| Service | TrxWalletService.java | 80 | 钱包CRUD |
| Service | TrxWalletOrderService.java | 55 | 订单CRUD |
| Service | TrxWalletCollectService.java | 39 | 归集CRUD |
| Repository | TrxWalletRepository.java | 18 | JPA查询 |
| Repository | TrxWalletOrderRepository.java | 22 | JPA + 原生SQL |
| Repository | TrxWalletCollectRepository.java | 14 | JPA查询 |
| Repository | TrxWalletConfigRepository.java | ~15 | 配置查询 |
| Client | **Trc20.java** | 125 | TRC20代币操作 |
| Client | **SmartContract.java** | 91 | 智能合约调用 |
| Client | **TronUtil.java** | 122 | gRPC客户端+签名 |
| Util | **TronUtils.java** | 473 | 交易打包+地址转换 |
| Util | AES.java | 47 | AES加解密 |
| Util | RSAUtil.java | 110 | RSA加解密 |
| Util | Base58.java | ~100 | Base58编码 |
| Util | ByteArray.java | ~50 | 字节操作 |
| Util | Sha256Hash.java | ~60 | SHA256哈希 |
| Util | WalletUtil.java | ~30 | YAML配置读取 |
| Util | CommonConstant.java | 31 | 常量定义 |
| Util | RedisConstants.java | ~20 | Redis键常量 |
| Util | UserServerRequestConstants.java | ~10 | 用户服务URL常量 |
| Task | CollectTrxJob.java | 40 | 定时归集任务 |
| Config | WebSecurityConfig.java | ~20 | 安全配置（禁用） |
| Config | ScheduleConfig.java | ~15 | 线程池配置 |
| Entity | BaseEntity.java | 38 | 基础实体（审计字段） |
| Entity | TrxWalletEntity.java | 43 | 钱包实体 |
| Entity | TrxWalletOrderEntity.java | 65 | 订单实体 |
| Entity | TrxWalletCollectEntity.java | 48 | 归集实体 |
| Entity | TrxWalletConfigEntity.java | 39 | 配置实体 |
| Enum | TRXContractEnum.java | 123 | 代币枚举 |
| Enum | TrxOrderTypeEnum.java | 34 | 订单类型枚举 |
| Enum | CollectTypeEnum.java | ~25 | 归集类型枚举 |
| BO | TRXKeystore.java | ~15 | 密钥存储BO |
| Request | TrxUserWalletRequest.java | 35 | 创建钱包请求 |
| Request | TrxCreateTransactionRequest.java | 61 | 转账请求 |
| Request | TrcCollectRequest.java | 49 | 归集请求 |
| Request | UserBalanceRequest.java | 24 | 余额查询请求 |
| Request | TrxWalletAccountRequest.java | ~30 | 钱包信息请求 |
| Response | WalletAddressVO.java | ~20 | 地址响应 |
| Response | TrxWalletResult.java | ~15 | 交易结果响应 |
| Response | TransactionVO.java | ~15 | 交易状态响应 |
| Response | TrxWalletAccountVO.java | ~25 | 钱包信息响应 |

---

## 附录B 与钱包参考A（p9）对比矩阵

| 维度 | p9 wallet-service（参考A） | tg wallet-server（参考B） | 评价 |
|------|---------------------------|--------------------------|------|
| **定位** | 内部余额钱包（维护balance字段） | 链上钱包（不维护余额，查链上） | B更接近我们需求 |
| **技术栈** | Spring Boot + MyBatis Plus + Redisson | Spring Boot + JPA + Redis | 相近 |
| **表设计** | 2表（user_coin + coin_record） | 4表（wallet + order + collect + config） | B更完善 |
| **并发控制** | Redisson分布式锁（20s/30s） | 几乎没有（wallet-server内） | A更好 |
| **幂等** | 无（order_no无唯一索引） | ssOrderId查询幂等（但非原子） | 都有缺陷 |
| **冻结机制** | 有冻结字段，无状态机 | 在user-server有freeze_amount | B的分离更清晰 |
| **审计** | coin_record（缺before/after） | balance_change（有before/after+UK） | B更完善 |
| **区块链** | 无 | ★ 完整TRON链上交互 | B独有价值 |
| **签名** | 无 | 外部签名服务 | B有安全隔离 |
| **归集** | 无 | ★ 单笔+批量归集+手续费垫付 | B独有价值 |
| **MQ** | RabbitMQ（4队列） | RabbitMQ（10+队列） | 规模不同 |
| **加密** | 无特殊加密 | AES + AWS KMS + RSA | B有密钥管理 |

**结论**：
- **p9（参考A）** 的价值在于：内部余额管理的基础CRUD模式、分布式锁使用模式
- **tg（参考B）** 的价值在于：**区块链充值/提现/归集的完整链路**、密钥管理体系、签名服务架构、定时补偿机制
- 两者互补，我们需要综合两者的优点

---

## 附录C 对我们ser-wallet区块链充值模块的设计启示

### C.1 架构层面

1. **链上服务独立**：参考 tg-services 的 wallet-server / user-server 分离模式。我们的 ser-wallet 内部也应该把"链上交互"独立为一个子模块（或独立服务），与"余额管理"解耦。

2. **签名服务隔离**：提现/出款的私钥**绝对不能**放在业务服务中。应该有独立的签名服务（可以是我们自建，也可以接入第三方托管服务）。

3. **定时补偿必须有**：任何依赖外部系统（区块链）的操作都必须有定时任务兜底。充值确认、提现确认、归集确认都需要定时扫描补偿。

### C.2 充值链路设计

参考 tg-services 的充值模式，但做关键升级：

```
tg-services 模式（被动查询）：
用户转账 → 用户点确认 → 查链上余额 → 归集 → 加余额

我们应该做的模式（主动监听）：
用户转账 → 区块扫描服务监听到交易 → 确认到账 → 归集 → 加余额
```

**关键差异**：
- tg-services 依赖用户手动确认，我们应该**自动检测链上充值**
- 可以使用 TronGrid 的事件订阅 API，或自建区块扫描服务
- 扫描服务定期（如每10秒）查询最新区块中涉及我们地址的交易

### C.3 核心数据表设计参考

基于 tg-services 的经验，我们的区块链相关表应该包含：

```
1. chain_wallet — 用户链上钱包（类似 trx_wallet）
   + user_id UNIQUE     // 必须有唯一约束
   + address UNIQUE     // 必须有唯一约束
   + chain_type         // 支持多链
   + en_private_key     // AES-256-GCM 加密

2. chain_transaction — 链上交易记录（类似 trx_wallet_order）
   + order_no UNIQUE    // 必须有唯一约束
   + tx_hash            // 链上交易哈希
   + status             // pending/confirming/confirmed/failed（tg缺少的）
   + direction          // deposit/withdraw/collect
   + amount BIGINT      // 最小单位整数（tg用的DECIMAL有精度丢失）
   + confirmations      // 确认数（tg缺少的）

3. chain_collect — 归集记录（类似 trx_wallet_collect）
   + from_address       // 用户地址
   + to_address         // 公司地址（tg缺少的）
   + status             // pending/confirmed/failed（tg缺少的）
   + gas_tx_hash        // 手续费垫付交易哈希（tg没有关联记录）
```

### C.4 安全设计底线

从 tg-services 的安全缺陷中吸取教训：

1. **加密算法**：AES-256-GCM（不用ECB），PBKDF2密钥派生（不用SHA-1），RSA-2048+（不用512）
2. **私钥存储**：加密后存DB，解密只在需要签名的瞬间，用后立即清零
3. **签名服务**：独立部署，双向TLS认证，有签名限额和风控
4. **配置管理**：敏感配置用环境变量或Secret Manager，绝不明文
5. **API认证**：服务间调用需Token认证，不允许裸奔
6. **审计日志**：每次链上操作都写审计日志，包含 who/when/what/result

### C.5 手续费管理参考

tg-services 的手续费垫付机制（gasAddress → 用户地址 → 转TRX作为能量费）是必须学习的：

```
我们的实现方案：

1. 维护一个 gas_pool 地址，预存大量 TRX
2. 归集 TRC20 代币前，先检查用户地址 TRX 余额
3. 如果不足，从 gas_pool 转入刚好够的 TRX（动态计算，不固定30）
4. 归集交易完成后，用户地址的剩余 TRX 可以回收（tg没有做这一步）
5. gas_pool 余额设置预警阈值，不足时告警
```

### C.6 关键数字记忆

来自 tg-services 源码的关键参数：

| 参数 | 值 | 含义 | 我们是否采用 |
|------|-----|------|-------------|
| TRX 精度 | 6位（1 TRX = 10^6 SUN） | 链上原生精度 | 是，遵循链上精度 |
| USDT 精度 | 6位 | TRC20-USDT精度 | 是 |
| FEE_LIMIT | 50,000,000 SUN (50 TRX) | TRC20交易最大手续费 | 参考，可调 |
| 手续费垫付 | 30 TRX | 归集前的手续费垫付 | 应改为动态计算 |
| 归集时间 | 每天 08:00 曼谷时间 | 批量归集触发时间 | 应更频繁 |
| 订单超时 | 30分钟 | 充值/提现超时取消 | 参考，可配置 |
| 防重冷却 | 10秒 | 重复提交冷却 | 参考 |
| 定时扫描 | 2分钟 | 补偿任务频率 | 参考，可配置 |

---

## 总结

### 一句话概括

> tg-services/wallet-server 是一个**纯链上钱包服务**——它不管余额，只管链上操作（地址生成、充值确认、归集、提现转账）——它的核心价值在于提供了**TRON区块链充值/提现/归集的完整可运行参考实现**，但在安全性（加密算法）、数据一致性（缺少唯一约束）、并发控制（几乎没有）方面有明显不足。

### 核心收获

1. **充值模式确认**：用户独立地址 + 链上余额查询 + 归集到公司地址 = 行业标准模式
2. **归集是必须的**：充值后必须归集，否则资金分散在大量用户地址中
3. **手续费垫付是必须的**：TRC20归集需要TRX作为能量费，必须有gas垫付机制
4. **签名隔离是必须的**：提现私钥必须隔离在独立签名服务中
5. **定时补偿是必须的**：任何链上操作都需要定时任务兜底
6. **精度用最小单位**：避免浮点/DECIMAL精度丢失，全链路用整数

### 三个"绝对不能照搬"

1. **绝对不能用 AES-ECB**——必须用 AES-256-GCM
2. **绝对不能用 DECIMAL 存链上金额**——必须用 BIGINT 最小单位
3. **绝对不能裸奔API**——必须有认证鉴权
