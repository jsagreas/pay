# 模块参考B — tg-services/wallet-server 深度分析

> 参考工程：`/Users/mac/gitlab/z-tmp/tg-services`
> 核心模块：`wallet-server`（TRC20虚拟币充值完整实现）
> 关联模块：`user-server`（充值/提现业务编排）
> 分析目标：深度解剖TRC20-USDT充提链路，提取可复用设计思路

---

## 目录

1. [工程全景](#1-工程全景)
2. [技术栈对比](#2-技术栈对比)
3. [wallet-server 完整文件结构](#3-wallet-server-完整文件结构)
4. [四张核心表结构](#4-四张核心表结构)
5. [SsOrder 订单表结构（user-server）](#5-ssorder-订单表结构user-server)
6. [TRC20充值完整链路——最核心价值](#6-trc20充值完整链路最核心价值)
7. [TRC20提现完整链路](#7-trc20提现完整链路)
8. [每日归集（Collect）机制](#8-每日归集collect机制)
9. [钱包地址生成——热钱包模型](#9-钱包地址生成热钱包模型)
10. [链上交互技术细节](#10-链上交互技术细节)
11. [智能合约调用层（Trc20 + SmartContract）](#11-智能合约调用层trc20--smartcontract)
12. [gRPC与HTTP双通道架构](#12-grpc与http双通道架构)
13. [密钥管理体系——三层加密](#13-密钥管理体系三层加密)
14. [外部签名服务集成](#14-外部签名服务集成)
15. [Gas费自动补充机制](#15-gas费自动补充机制)
16. [定时任务编排](#16-定时任务编排)
17. [跨服务调用拓扑](#17-跨服务调用拓扑)
18. [枚举与常量体系](#18-枚举与常量体系)
19. [设计模式与架构特征提炼](#19-设计模式与架构特征提炼)
20. [值得借鉴的设计（直接采纳）](#20-值得借鉴的设计直接采纳)
21. [需要改进/不采纳的设计](#21-需要改进不采纳的设计)
22. [与我们Go工程的映射对照](#22-与我们go工程的映射对照)
23. [TRC20充值实现移植指南](#23-trc20充值实现移植指南)

---

## 1. 工程全景

### 1.1 工程概况

tg-services 是一个 Java Spring Boot 2.5.3 微服务工程（Maven多模块），共 **14个模块**：

| 分类 | 模块 | 职责 |
|------|------|------|
| **公共库** | module-common | 工具类、拦截器、VO/BO定义（178个Java文件） |
| **公共库** | module-swagger2 | OpenAPI文档 |
| **公共库** | module-jjwt | JWT认证 |
| **公共库** | module-authenticator | 认证框架 |
| **公共库** | module-springcache-redis | Redis缓存封装 |
| **核心服务** | **wallet-server** | **TRX/TRC20区块链钱包（本次分析重点）** |
| **核心服务** | wallet-third-server | 第三方支付网关（汇旺等） |
| **核心服务** | **user-server** | 用户管理 + 虚拟币充提业务编排 |
| **业务服务** | casino（4子模块） | 游戏平台（core/backend/web/agentbackend） |
| **业务服务** | game-server | 游戏逻辑 |
| **业务服务** | authentication-server | 用户认证 |
| **业务服务** | file-upload-download | 文件服务（MinIO） |
| **业务服务** | sms | 短信通知 |
| **业务服务** | telegram-bot-api | Telegram Bot集成 |

### 1.2 关键发现

这个项目的**核心价值**不在于它的钱包余额管理（那部分在user-server的Assets中，相对简单），而在于它实现了**完整的TRC20虚拟币链上充值/提现/归集全链路**，包括：

- 热钱包地址生成（per-user独立地址）
- 链上余额查询（TRX主币 + TRC20代币）
- TRC20智能合约调用（transfer）
- 交易签名（本地签名 + 外部签名服务双模式）
- 交易广播与确认轮询
- 资金归集（用户钱包 → 公司主钱包）
- Gas费自动补充
- AWS KMS密钥管理

---

## 2. 技术栈对比

| 维度 | 参考工程B（tg-services） | 我们的Go工程 |
|------|--------------------------|--------------|
| 语言 | Java 17 | Go 1.21+ |
| Web框架 | Spring Boot 2.5.3 | CloudWeGo Hertz（HTTP）+ Kitex（RPC） |
| ORM | Spring Data JPA / Hibernate | GORM Gen |
| 数据库 | MySQL 8.0 | TiDB |
| 缓存 | Redis + Lettuce | Redis |
| 消息队列 | RabbitMQ（HTTP REST替代） | NATS JetStream |
| 分布式锁 | Redisson（FairLock） | Redis分布式锁 / CAS乐观锁 |
| 区块链交互 | web3j + TRON wallet-cli + gRPC | 待定（需引入） |
| 密钥管理 | AWS KMS + AES | 待定 |
| 签名服务 | 外部RSA签名系统 | 待定 |
| 配置管理 | application.yml + DB配置表 | ETCD + 配置中心 |
| 定时任务 | @Scheduled（Spring） | 待定（cron / 独立scheduler） |
| 精度处理 | BigDecimal | BIGINT最小货币单位 |

---

## 3. wallet-server 完整文件结构

```
wallet-server/
├── pom.xml                              # Maven (web3j 3.6, grpc 1.9, tron wallet-cli 1.0, AWS KMS SDK)
├── src/main/java/com/baisha/walletserver/
│   ├── WalletServerApplication.java     # Spring Boot入口
│   │
│   ├── controller/
│   │   ├── WalletController.java        # 6个REST接口 (143行)
│   │   └── TestController.java          # 测试加密接口
│   │
│   ├── driver/                          # 驱动层（编排层）
│   │   ├── TrxDriver.java              # ★ TRX业务编排 (338行) - 归集/提现/查余额
│   │   └── AwsKmsDriver.java           # AWS KMS加解密 (110行)
│   │
│   ├── business/
│   │   └── WalletTrxBusiness.java      # ★★ 核心区块链业务 (538行) - 钱包生成/转账/签名
│   │
│   ├── client/trx/                      # ★★★ TRON区块链客户端
│   │   ├── SmartContract.java          # 智能合约调用 (91行) - gRPC方式
│   │   ├── Trc20.java                  # TRC20代币操作 (125行) - balanceOf/transfer/decimals
│   │   └── TronUtil.java              # gRPC客户端封装 (122行) - 签名/广播
│   │
│   ├── model/
│   │   ├── BaseEntity.java             # JPA审计基类 (id, createTime, updateTime)
│   │   ├── TrxWalletEntity.java        # ★ 用户钱包表 (userId, address, enPrivateKey)
│   │   ├── TrxWalletOrderEntity.java   # ★ 链上交易订单表 (txId, ssOrderId, amount, orderType)
│   │   ├── TrxWalletCollectEntity.java # 归集记录表 (userId, txId, collectAddress, amount)
│   │   ├── TrxWalletConfigEntity.java  # 配置表 (type, code, value)
│   │   ├── bo/TRXKeystore.java         # 密钥存储BO
│   │   ├── enums/
│   │   │   ├── TRXContractEnum.java    # 合约枚举 (TRX/USDT/CXT)
│   │   │   ├── TrxOrderTypeEnum.java   # 订单类型 (RECHARGE=1/WITHDRAW=2)
│   │   │   └── CollectTypeEnum.java    # 归集类型 (SINGLE=1/COLLECT=2)
│   │   └── vo/
│   │       ├── request/                # 5个请求VO
│   │       └── response/              # 4个响应VO
│   │
│   ├── repository/                      # JPA仓库层
│   │   ├── TrxWalletRepository.java
│   │   ├── TrxWalletOrderRepository.java
│   │   ├── TrxWalletConfigRepository.java
│   │   └── TrxWalletCollectRepository.java
│   │
│   ├── service/
│   │   ├── TrxWalletService.java       # 钱包CRUD + 私钥解密 (80行)
│   │   ├── TrxWalletOrderService.java  # 订单CRUD (55行)
│   │   └── TrxWalletCollectService.java # 归集记录CRUD
│   │
│   ├── task/
│   │   └── CollectTrxJob.java          # 每日08:00归集USDT定时任务
│   │
│   └── util/
│       ├── TronUtils.java             # ★★ TRON工具类 (473行) - 地址转换/签名/打包Transaction
│       ├── AES.java                   # AES/ECB/PKCS5Padding对称加密
│       ├── Base58.java                # Base58编解码
│       ├── ByteArray.java             # 字节数组工具
│       ├── RSAUtil.java               # RSA非对称加密（签名服务通信用）
│       ├── Sha256Hash.java            # SHA256哈希
│       ├── WalletUtil.java            # YAML配置读取（合约地址）
│       └── contants/
│           ├── CommonConstant.java    # 签名服务相关常量
│           ├── RedisConstants.java    # Redis键名
│           └── UserServerRequestConstants.java # user-server接口路径
│
└── src/main/resources/
    ├── application.yml                 # ★ 核心配置（TRON节点、KMS密钥、合约地址）
    ├── config.conf                     # TRON网络配置（gRPC节点、区块扫描起始高度）
    ├── wallet-cli-1.0.jar             # TRON钱包CLI库（2.8MB）
    └── commons-lang-2.5.jar
```

**文件总计**：48个Java源文件，核心业务代码约 **3500行**

---

## 4. 四张核心表结构

### 4.1 trx_wallet（用户钱包表）

```sql
CREATE TABLE `trx_wallet` (
  `id`             BIGINT AUTO_INCREMENT PRIMARY KEY,
  `user_id`        VARCHAR(10)  COMMENT '会员账号',        -- 索引
  `user_name`      VARCHAR(30)  COMMENT '会员姓名',
  `en_private_key` VARCHAR(250) COMMENT '加密后的私钥',     -- AES加密存储
  `address`        VARCHAR(100) COMMENT '钱包地址',         -- T开头的TRON地址
  `create_time`    DATETIME,
  `create_by`      VARCHAR(255),
  `update_time`    DATETIME,
  `update_by`      VARCHAR(255),
  INDEX idx_userId (user_id)
) COMMENT='用户热钱包地址表';
```

**设计要点**：
- 每个用户一个独立链上地址（per-user热钱包模型）
- 私钥 `en_private_key` 以 AES/ECB/PKCS5Padding 加密存储
- 加密密钥 = SHA1(KMS解密后的passphrase + address)的前16字节
- **不存余额**——余额实时查链上获取

### 4.2 trx_wallet_order（链上交易订单表）

```sql
CREATE TABLE `trx_wallet_order` (
  `id`            BIGINT AUTO_INCREMENT PRIMARY KEY,
  `user_id`       VARCHAR(10)    COMMENT '会员账号',
  `tx_id`         VARCHAR(200)   COMMENT '链上交易ID（hash）',
  `ss_order_id`   VARCHAR(200)   COMMENT '业务订单ID（关联user-server的SsOrder.id）',
  `amount`        DECIMAL(16,2)  COMMENT '金额',
  `from_address`  VARCHAR(250)   COMMENT '发送地址',
  `to_address`    VARCHAR(250)   COMMENT '接收地址',
  `coin_name`     VARCHAR(10)    COMMENT '币种类型（USDT/TRX）',
  `order_type`    TINYINT(2)     COMMENT '1=充值/归集, 2=提现',
  `post_type`     TINYINT(2)     COMMENT '1=接口方式, 2=定时任务方式',
  `create_time`   DATETIME,
  INDEX idx_user_id   (user_id),
  INDEX idx_tx_id     (tx_id),
  INDEX idx_order_type(order_type)
) COMMENT='链上交易记录表';
```

**关键查询**：
```sql
-- 根据业务订单ID查找USDT交易（防重复提现）
SELECT * FROM trx_wallet_order WHERE ss_order_id = ? AND coin_name = 'USDT'
```

### 4.3 trx_wallet_collect（归集记录表）

```sql
CREATE TABLE `trx_wallet_collect` (
  `id`              BIGINT AUTO_INCREMENT PRIMARY KEY,
  `user_id`         VARCHAR(10)    COMMENT '会员账号',
  `tx_id`           VARCHAR(200)   COMMENT '交易ID',
  `collect_address` VARCHAR(250)   COMMENT '归集源地址（用户钱包地址）',
  `amount`          DECIMAL(16,2)  COMMENT '归集金额',
  `collect_type`    TINYINT(2)     COMMENT '1=单笔即时归集, 2=定时批量归集',
  `create_time`     DATETIME,
  INDEX idx_userId      (user_id),
  INDEX idx_txId        (tx_id),
  INDEX idx_collectType (collect_type)
) COMMENT='资金归集记录表';
```

### 4.4 trx_wallet_config（配置表）

```sql
CREATE TABLE `trx_wallet_config` (
  `id`    BIGINT AUTO_INCREMENT PRIMARY KEY,
  `type`  VARCHAR(30)   COMMENT '配置类型（如 sign）',
  `code`  VARCHAR(50)   COMMENT '配置标识（如 rsa/platCode/disPendAddress）',
  `value` VARCHAR(1000) COMMENT '配置值'
) COMMENT='钱包运行时配置表';
```

**配置项说明**：
| type | code | value含义 |
|------|------|-----------|
| sign | rsa | 外部签名服务RSA公钥 |
| sign | platCode | 平台标识码（签名服务认证） |
| sign | disPendAddress | 出款地址（提现源地址） |

---

## 5. SsOrder 订单表结构（user-server）

这是跨服务的**业务订单表**，充值/提现订单在此创建，与wallet-server的链上交易通过 `ssOrderId ↔ SsOrder.id` 关联。

```sql
CREATE TABLE `ss_order` (
  `id`               BIGINT AUTO_INCREMENT PRIMARY KEY,
  `order_num`        VARCHAR(32)    COMMENT '订单编号',
  `user_id`          BIGINT(20)     COMMENT '会员ID',
  `tg_user_id`       VARCHAR(64)    COMMENT 'Telegram用户ID',
  `user_type`        TINYINT(2)     COMMENT '1正式 2测试 3机器人 4体验',
  `amount`           DECIMAL(16,2)  COMMENT '总金额',
  `real_amount`      DECIMAL(16,2)  COMMENT '实际到账金额',
  `fee_amount`       DECIMAL(16,2)  DEFAULT 0 COMMENT '手续费',
  `order_type`       TINYINT(2)     COMMENT '1=充值 2=提现',
  `order_status`     TINYINT(2)     DEFAULT 1 COMMENT '1等待 2成功 3失败 4取消',
  `virtual_currency` VARCHAR(20)    COMMENT '虚拟币种（USDT）',
  `payee_address`    VARCHAR(200)   COMMENT '收款地址（提现目标地址）',
  `tx_id`            VARCHAR(200)   COMMENT '链上交易ID',
  `flow_multiple`    DECIMAL(16,2)  DEFAULT 1 COMMENT '流水倍数',
  `complete_time`    DATETIME       COMMENT '完成时间',
  `nick_name`        VARCHAR(64)    COMMENT '会员昵称',
  `agent_id`         BIGINT(11)     COMMENT '当前代理ID',
  `agent_level`      TINYINT(2)     COMMENT '代理层级',
  -- ... 其他代理/渠道/审核字段略
  INDEX idx_userId      (user_id),
  INDEX idx_orderType   (order_type),
  INDEX idx_orderStatus (order_status),
  INDEX idx_tgUserId    (tg_user_id)
) COMMENT='充值/提现订单表';
```

**状态机**：
```
创建订单 → VIRTUAL_CURRENCY_WAIT(等待)
           ├→ ORDER_SUCCESS(成功)     ← 链上确认成功
           ├→ ORDER_SYSTEM_CANCLE(取消) ← 超时30分钟未完成
           └→ ORDER_FAIL(失败)        ← 链上确认失败
```

---

## 6. TRC20充值完整链路——最核心价值

这是本参考工程最有价值的实现，完整描述了"用户USDT充值到平台账户"的端到端流程。

### 6.1 充值全链路时序

```
┌──────────┐     ┌───────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────┐
│  用户(TG)  │     │ user-server│     │ wallet-server │     │  TRON区块链    │     │ web-server│
└────┬─────┘     └─────┬─────┘     └──────┬───────┘     └───────┬───────┘     └────┬─────┘
     │                  │                   │                     │                  │
     │ 1.输入充值金额    │                   │                     │                  │
     │ ─────────────────>                   │                     │                  │
     │                  │ 创建SsOrder       │                     │                  │
     │                  │ status=WAIT       │                     │                  │
     │                  │                   │                     │                  │
     │ 2.获取钱包地址    │                   │                     │                  │
     │ ─────────────────>                   │                     │                  │
     │                  │ /wallet/create ──>│                     │                  │
     │                  │                   │ 生成ECKeyPair       │                  │
     │                  │                   │ AES加密私钥存DB      │                  │
     │ <── 返回T开头地址 │<─ 返回address ───│                     │                  │
     │                  │                   │                     │                  │
     │ 3.用户链上转USDT  │                   │                     │                  │
     │ ────────────────────────────────────────────────────────>  │                  │
     │                  │                   │                     │ 链上确认          │
     │                  │                   │                     │                  │
     │ 4.定时任务(2min)  │                   │                     │                  │
     │                  │ checkCharge───┐   │                     │                  │
     │                  │ CryptOrder    │   │                     │                  │
     │                  │               ↓   │                     │                  │
     │                  │ 查询超时订单→取消  │                     │                  │
     │                  │ 查询30min内WAIT订单│                     │                  │
     │                  │ ──────────────────>│                     │                  │
     │                  │ /wallet/getBalance │                     │                  │
     │                  │                   │ Trc20.balanceOf() ─>│                  │
     │                  │                   │ <── USDT余额 ───────│                  │
     │                  │ <── 余额 ─────────│                     │                  │
     │                  │                   │                     │                  │
     │                  │ 余额>=充值金额?    │                     │                  │
     │                  │ YES → 触发归集     │                     │                  │
     │                  │ ──────────────────>│                     │                  │
     │                  │ /wallet/collectTrc20                    │                  │
     │                  │                   │                     │                  │
     │                  │                   │ ┌─ singleCollect ─┐│                  │
     │                  │                   │ │ 查TRX余额        ││                  │
     │                  │                   │ │ <30? → 补30TRX   ││                  │
     │                  │                   │ │ TRC20 transfer   ││                  │
     │                  │                   │ │ → 主钱包地址      ││                  │
     │                  │                   │ │ 签名+广播         ├>──────────────>  │
     │                  │                   │ │ 记录collect表     ││ <── txId ───── │
     │                  │                   │ │ 记录order表       ││                  │
     │                  │                   │ └─────────────────┘│                  │
     │                  │ <── txId ─────────│                     │                  │
     │                  │ 更新SsOrder.txId   │                     │                  │
     │                  │                   │                     │                  │
     │ 5.下次定时任务    │                   │                     │                  │
     │                  │ txId已有→查确认    │                     │                  │
     │                  │ ──────────────────>│                     │                  │
     │                  │ /wallet/getTransactionInfoById           │                  │
     │                  │                   │ gettransactionbyid─>│                  │
     │                  │                   │ <── contractRet ────│                  │
     │                  │ <── SUCCESS ──────│                     │                  │
     │                  │                   │                     │                  │
     │                  │ updateOrderStatus  │                     │                  │
     │                  │ = ORDER_SUCCESS    │                     │                  │
     │                  │ 增加用户余额       │                     │                  │
     │                  │ 增加打码量         │                     │                  │
     │                  │ 发送统计MQ         │                     │                  │
     │                  │ ─────────────────────────────────────────────────────────>  │
     │                  │                   │                     │   通知用户钱包    │
     │                  │                   │                     │                  │
```

### 6.2 充值链路关键代码路径

**阶段一：定时任务发现待处理订单**

```
HandleVirtualCurrencyTask.checkVirtualCurrencyTask()     // 每2分钟，Bangkok时区
  → VirtualCurrencyBusiness.checkChargeCrypotcurrencyOrder("USDT", 30)
    → 先处理超时：超过30分钟的WAIT订单 → ORDER_SYSTEM_CANCLE
    → 查询30分钟内的WAIT充值订单列表
    → 对每个订单：asyncVirtualDepositService.deposit(orderId, "USDT", false)
```

`VirtualCurrencyBusiness.java:75-96` 关键逻辑：
- 超时阈值：30分钟（`HandleVirtualCurrencyTask.timeoutMinutes`）
- 超时订单直接更新状态为 `ORDER_SYSTEM_CANCLE`
- 未超时订单异步分发给 `AsyncVirtualDepositService`

**阶段二：异步充值处理（含分布式锁）**

```
AsyncVirtualDepositService.deposit(orderId, "USDT", triggerByTg=false)
  → Redisson FairLock: key = "USER_SERVER_DEPOSIT_TASK_SS_ORDER_ID_" + orderId
  → tryLock(0秒等待, 10秒持有)   // 抢不到锁直接返回，等下次定时任务
  → 验证订单状态仍为 VIRTUAL_CURRENCY_WAIT
  → checkCrypotcurrencyOrder(...)
```

`AsyncVirtualDepositService.java:190-224` 的锁策略：
- 使用 Redisson **FairLock**（公平锁）而非普通锁
- 等待时间0秒——抢不到就放弃，下次定时任务再来
- 持有时间10秒——充值检查在10秒内完成

**阶段三：充值检查核心逻辑（两阶段轮询模型）**

`AsyncVirtualDepositService.java:96-188` — **这是最核心的充值确认逻辑**：

```java
// 第一次进入：txId为空 → 查余额 → 触发归集
if (StringUtils.isEmpty(txId)) {
    balance = walletBusiness.getBalance(userId, "USDT");    // 查链上余额
    if (balance < realAmount) return "余额不足";

    txId = walletBusiness.getTransferResult(...);            // 触发归集：用户钱包→主钱包
    ssOrderService.updateTxIdById(txId, orderId);            // 记录txId
    return "获取到txId后,下次进入";                            // ★ 关键：本次不继续，等下次定时任务
}

// 第二次进入：txId已有 → 查链上确认
transactionVO = walletBusiness.getTransactionInfoById(orderId);
if (transactionVO.getTransResult() == true) {
    updateOrderStatusAndSendMQ(orderId, userId, ...);        // 上分 + 通知
}
```

**两阶段轮询模型的设计智慧**：
1. 第一次：确认链上收到USDT → 发起归集交易 → 拿到txId → **立即返回不等待确认**
2. 第二次（2分钟后）：用txId查链上确认 → 确认成功 → 增加用户余额

这样设计避免了长时间阻塞等待链上确认（TRON确认通常需要数秒到数十秒）。

**阶段四：余额增加与通知**

`AsyncVirtualDepositService.java:258-344` — 充值成功后的处理：

```java
boolean updateOrderStatusAndSendMQ(...) {
    // 1. 更新订单状态为SUCCESS（CAS: 只有WAIT状态才能更新）
    ssOrderService.updateOrderStatusAndCompleteTime(
        ORDER_SUCCESS, completeTime, txId, orderId, userId,
        virtualCurrency, VIRTUAL_CURRENCY_WAIT);  // WHERE status = WAIT

    // 2. 体验用户首充转正式用户
    if (isPractice(userType)) {
        doClearPracticeAmount(user, orderId);
        userService.updateUserTypeByIdAndUserType(userId, NORMAL, PRACTICE);
    }

    // 3. 增加用户余额
    addBalance(ssOrder, user, realAmount);       // BalanceChangeEnum.TELEGRAM_CHARGE

    // 4. 增加打码量（流水要求）
    BigDecimal playMoney = realAmount * flowMultiple;
    doPlayMoney(ssOrder, user, playMoney);

    // 5. HTTP通知backend统计服务
    sendMQ(orderType, orderId, userId, amount, ...);
}
```

### 6.3 充值链路中wallet-server侧的关键实现

当user-server调用 `/wallet/collectTrc20` 时：

```
WalletController.collectTrc20(TrcCollectRequest)
  → TrxDriver.singleCollect(address, userId, "USDT", ssOrderId, amount, SINGLE, postType)
    → 查USDT余额：getAccountBalance(fromAddress, "USDT")
    → 余额 <= 0? → return null
    → 查TRX余额（gas费）：getAccountBalance(fromAddress, "TRX")
    → TRX < 30? → 从gasAddress转30 TRX到用户地址（作为gas费）
    → 从用户地址转全部USDT到companyAddress
    → 记录collect表 + order表
    → return TrxWalletResult(txId)
```

`TrxDriver.java:175-228` — singleCollect核心逻辑：

```java
public TrxWalletResult singleCollect(fromAddress, userId, coinName, ssOrderId, reChargeAmount, collectCode, postType) {
    BigDecimal trxBalance = reChargeAmount;
    String newMainAddress = awsKmsDriver.decryptData(companyAddress);  // KMS解密主钱包地址

    // 如果未传金额，主动查链上余额
    if (null == reChargeAmount) {
        trxBalance = getAccountBalance(fromAddress, coinName);
    }

    // 余额不足
    if (minSumAmount >= trxBalance) return null;

    // ★ TRC20转账需要TRX作为gas费 → 自动补充
    if (!TRX合约) {
        BigDecimal addTrx = getAccountBalance(fromAddress, "TRX");
        if (30 > addTrx) {
            // 从gasAddress转30 TRX到用户地址
            createTransactionForTrx(gasPassword, 30, gasAddress, fromAddress, "TRX", ssOrderId);
            // 记录gas补充订单
            trxWalletOrderService.save(userId, gasAddress, fromAddress, txId, ...);
        }
    }

    // ★ 执行TRC20归集转账：用户地址 → 主钱包地址
    trxWalletResult = createTransactionForTrx(null, trxBalance, fromAddress, newMainAddress, tokenContract, ssOrderId);

    // 记录归集表和订单表
    reverseNoticeCollectDetail(userId, fromAddress, txId, trxBalance, collectCode, ssOrderId, coinName, postType);

    return trxWalletResult;
}
```

---

## 7. TRC20提现完整链路

### 7.1 提现全链路时序

```
用户发起提现 → user-server创建SsOrder(type=WITHDRAW, status=WAIT)
            → 冻结用户余额(freezeAmount)
            → VirtualCurrencyWithdrawBusiness.withdraw()
            → walletBusiness.withdrawTransaction(ssOrder)
                → wallet-server /wallet/withdrawTraction
                    → TrxDriver.withdrawTraction()
                        → 从disPendAddress（出款地址）转USDT到payeeAddress
                        → 签名方式：外部签名服务（因为是出款地址）
                        → 记录order表
                        → return txId
            → 更新SsOrder.txId
            → 等待定时任务确认

定时任务(2min) → asyncVirtualWithdrawService.withdrawVirtualCurrencyTask()
              → 查询txId对应的链上交易状态
              → SUCCESS → 解冻 → 扣减余额 → 扣手续费 → 更新状态 → 通知
```

### 7.2 提现链路关键实现

**提现发起** — `VirtualCurrencyWithdrawBusiness.java:190-245`:

```java
public boolean withdraw(SsOrder ssOrder, Assets assets, String virtualCurrency) {
    // 防重复上链
    if (StringUtils.isNotEmpty(ssOrder.getTxId())) return true;
    String orderIdToTxIdKey = "order::withdraw_" + orderId;
    if (redisUtil.hasKey(orderIdToTxIdKey)) return true;

    // 超时检查（超过30分钟的提现订单不处理，交给定时任务取消）
    if (tooLong(ssOrder)) return false;

    // 调用wallet-server执行链上提现
    String txId = walletBusiness.withdrawTransaction(ssOrder);
    if (StringUtils.isNotEmpty(txId)) {
        ssOrderService.updateTxIdById(txId, orderId);
    }

    // 30天防重复Redis标记
    redisUtil.set(orderIdToTxIdKey, ..., 30*24, TimeUnit.HOURS);
    return true;
}
```

**wallet-server提现执行** — `TrxDriver.java:281-337`:

```java
public TrxWalletResult withdrawTraction(orderType, tokenContract, request) {
    // 出款地址：从配置表获取（不是KMS直接解密，而是DB配置表）
    String fromAddress = configList.filter(code == "disPendAddress").getValue();

    // 幂等检查：相同ssOrderId不重复执行
    TrxWalletOrderEntity order = trxWalletOrderRepository.findBySsOrderId(ssOrderId);
    if (order != null) {
        return new TrxWalletResult(order.getTxId());  // 返回已有txId
    }

    // 余额检查
    BigDecimal balance = getAccountBalance(fromAddress, coinName);
    if (sendAmount > balance) return null;

    // TRC20提现同样需要补充TRX gas费
    if (TRC20 && TRX余额 < 30) {
        transferTRX(gasAddress → fromAddress, 30 TRX);
    }

    // ★ 执行提现转账（fromAddress参数传的是出款地址本身，触发外部签名）
    result = createTransactionForTrx(fromAddress, sendAmount, fromAddress, toAddress, tokenContract, ssOrderId);

    // 记录订单
    trxWalletOrderService.save(..., orderType=WITHDRAW, ...);
    return result;
}
```

**提现确认** — `AsyncVirtualWithdrawService.java:125-201`:

```java
public boolean withdrawUSDT(orderId, virtualCurrency, txId) {
    // 查链上交易状态
    TransactionVO transactionVO = walletBusiness.getTransactionInfoById(orderId);
    if (!transactionVO.getTransResult()) return false;

    // 防重复处理
    if (isOrderStatus(orderId, ORDER_SUCCESS)) return false;

    // 1. 解冻 amount（从冻结金额中扣除）
    doFreezeAmount(ssOrder, user);

    // 2. 减少实际到账金额（BalanceChangeEnum: TELEGRAM_WITHDRAW）
    reduceRealAmount(ssOrder, user, ssOrder.getRealAmount());

    // 3. 减少手续费（BalanceChangeEnum: FEE）
    doBalanceFeeBusiness(ssOrder, user, ssOrder.getFeeAmount());

    // 4. 更新订单状态为SUCCESS
    ssOrderService.updateWithdrawOrderStatus(ORDER_SUCCESS, txId, orderId, ...);

    // 5. 通知统计服务
    sendMQforWithdraw(orderId, userId, amount, ...);

    // 6. 通知web-server更新前端钱包显示
    notifyWallet(currentBalance, orderId, orderType, txId);
    return true;
}
```

---

## 8. 每日归集（Collect）机制

### 8.1 归集定义

"归集"是将分散在用户热钱包地址上的代币集中转移到公司主钱包地址的过程。

### 8.2 归集触发方式

| 触发方式 | 场景 | 代码位置 |
|---------|------|---------|
| **即时归集（SINGLE=1）** | 用户充值确认后，立即归集该用户USDT到主钱包 | `WalletController.collectTrc20()` → `TrxDriver.singleCollect()` |
| **批量归集（COLLECT=2）** | 每日08:00（Bangkok时区），遍历所有用户钱包归集 | `CollectTrxJob.collectUSDT()` → `TrxDriver.collectTrc20()` |

### 8.3 批量归集流程

`CollectTrxJob.java:34-39`:
```java
@Scheduled(cron = "0 0 8 * * ?", zone = "Asia/Bangkok")
public void collectUSDT() {
    TrcCollectRequest request = new TrcCollectRequest();
    request.setCoinName("USDT");
    trxDriver.collectTrc20(request);
}
```

`TrxDriver.java:230-255`:
```java
public Boolean collectTrc20(TrcCollectRequest request) {
    String mainAddress = awsKmsDriver.decryptData(companyAddress);
    List<TrxWalletEntity> allWallets = trxWalletService.findAll();

    for (TrxWalletEntity wallet : allWallets) {
        if (mainAddress.equals(wallet.getAddress())) continue;  // 跳过主钱包

        String ssOrderId = UUID.randomUUID().toString().replaceAll("-","");
        singleCollect(wallet.getAddress(), wallet.getUserId(), "USDT",
                      ssOrderId, null, COLLECT, 2);
    }
    return true;
}
```

### 8.4 归集的意义

1. **安全性**：用户热钱包私钥在DB中，归集后资金集中到主钱包，减少暴露面
2. **管理便利**：统一出款地址，提现只从一个地址出
3. **Gas费优化**：集中后减少后续出款时的gas费预充次数

---

## 9. 钱包地址生成——热钱包模型

### 9.1 地址生成流程

`WalletTrxBusiness.java:99-136`:

```java
public WalletAddressVO createWallet(TrxUserWalletRequest request) {
    String userId = request.getUserId();
    String passphrase = awsKmsDriver.decryptData(this.passphrase);  // KMS解密口令

    // 1. 检查已有地址
    String address = trxWalletService.findAddressByUserId(userId);
    if (address != null) return existingAddress;

    // 2. 生成密钥对（web3j）
    ECKeyPair ecKeyPair = Keys.createEcKeyPair();
    WalletFile walletFile = Wallet.createStandard(passphrase, ecKeyPair);

    // 3. 转换为TRON地址格式：41 + ethereumAddress → Base58Check编码 → T开头
    String originAddress = walletFile.getAddress();   // 以太坊格式（无0x前缀）
    String finalAddress = fromHexAddress("41" + originAddress);  // TRON地址

    // 4. 提取私钥（hex格式）
    String keyStore = Numeric.toHexStringNoPrefix(ecKeyPair.getPrivateKey());

    // 5. AES加密私钥存储
    String enKey = encrypt(keyStore, passphrase + finalAddress);
    // 密钥派生：SHA1(passphrase + address) → 取前16字节 → AES/ECB/PKCS5Padding

    // 6. 保存到DB
    trxWalletService.save(userId, userName, enKey, finalAddress);

    // 7. 异步通知user-server更新用户钱包地址
    trxWalletService.updateUserAddress(request);

    return walletAddressVO;
}
```

### 9.2 地址格式

| 格式 | 示例 | 说明 |
|------|------|------|
| Ethereum原始 | `a4f4e00c...` | 40位hex |
| TRON Hex | `41a4f4e00c...` | 41前缀 + 40位hex（42字符） |
| TRON Base58 | `TUyKqdNXN...` | T开头，人类可读，含校验和 |

### 9.3 与以太坊钱包的关系

TRON的地址体系兼容以太坊（都基于secp256k1椭圆曲线），区别仅在于：
- 以太坊：0x前缀 + 20字节地址
- TRON：41前缀 + 20字节地址 → Base58Check编码

因此可以用 `web3j.crypto.Keys.createEcKeyPair()` 生成密钥对，只是地址编码不同。

---

## 10. 链上交互技术细节

### 10.1 TRC20转账核心实现

`WalletTrxBusiness.java:209-290` — `trc20Transaction()`:

```java
public TrxWalletResult trc20Transaction(keyStore, from, to, tokenContract, amount, ssOrderId) {
    // 1. 验证目标地址
    validateAddress(to);  // HTTP POST → TRON节点 /wallet/validateaddress

    // 2. 构建智能合约调用参数
    Map<String, Object> map = new HashMap<>();
    map.put("contract_address", toHexAddress(tokenContract));  // USDT合约地址(hex)
    map.put("function_selector", "transfer(address,uint256)");

    // 金额处理：乘以10^decimals（USDT是10^6）
    BigInteger integer = amount.multiply(BigDecimal.TEN.pow(6)).toBigInteger();
    String parameter = encoderAbi(toHexAddress(to).substring(2), integer);
    // parameter = ABI编码(address + uint256)

    map.put("parameter", parameter);
    map.put("owner_address", toHexAddress(from));
    map.put("call_value", 0);
    map.put("fee_limit", 50000000L);  // 50 TRX fee上限

    // 3. 创建未签名交易
    String result = HttpClient4Util.doPost(trxUrl + "/wallet/triggersmartcontract", JSON.toJSONString(map));
    JSONObject transaction = result.getJSONObject("transaction");

    // 添加交易备注数据
    transaction.getJSONObject("raw_data").put("data", Hex.toHexString("trc20".getBytes()));

    // 4. 打包为Protocol.Transaction（protobuf）
    Protocol.Transaction tx = TronUtils.packTransaction(transaction.toJSONString(), false);

    // 5. 签名（双模式）
    byte[] bytes;
    if (disPendAddress.equals(from)) {
        // 出款地址 → 外部签名服务
        bytes = sendSign(signUrl, tx, ssOrderId, from);
    } else {
        // 归集地址（用户地址） → 本地签名
        bytes = signTransaction2Byte(tx.toByteArray(), ByteArray.fromHexString(keyStore));
    }

    // 6. 广播已签名交易
    Map<String, Object> broadcastMap = new HashMap<>();
    broadcastMap.put("transaction", ByteArray.toHexString(bytes));
    String raw = HttpClient4Util.doPost(trxUrl + "/wallet/broadcasthex", JSON.toJSONString(broadcastMap));

    // 7. 检查广播结果
    JSONObject broadcastResult = JSON.parseObject(raw);
    if (!"SUCCESS".equalsIgnoreCase(broadcastResult.getString("code"))) return null;

    String txId = broadcastResult.getString("txid");
    return new TrxWalletResult(txId);
}
```

### 10.2 TRX主币转账实现

`WalletTrxBusiness.java:323-362` — `trxOfflineSignature()`:

```java
public TrxWalletResult trxOfflineSignature(keyStore, from, to, amount, ssOrderId) {
    byte[] fromAddress = WalletApi.decodeFromBase58Check(from);
    byte[] toAddress = WalletApi.decodeFromBase58Check(to);
    BigDecimal sunAmount = amount.multiply(BigDecimal.TEN.pow(6));  // 1 TRX = 10^6 SUN

    // 使用TRON SDK创建离线交易
    Protocol.Transaction transaction = createTransaction(fromAddress, toAddress, sunAmount.longValueExact());

    // TRX转账统一走外部签名服务
    byte[] bytes = sendSign(signUrl, transaction, ssOrderId, from);

    // 使用TRON SDK广播（gRPC方式）
    String hash = ByteArray.toHexString(Sha256Sm3Hash.hash(transaction.getRawData().toByteArray()));
    boolean success = WalletApi.broadcastTransaction(bytes);

    return success ? new TrxWalletResult(hash) : null;
}
```

### 10.3 交易确认查询

`WalletTrxBusiness.java:517-537`:

```java
public Boolean getTransactionInfoById(String txId) {
    Map<String, Object> map = new HashMap<>();
    map.put("value", txId);

    String result = HttpClient4Util.doPost(trxUrl + "/wallet/gettransactionbyid", JSON.toJSONString(map));
    JSONObject resultObject = JSON.parseObject(result);

    if (resultObject.isEmpty()) return false;  // 交易尚未上链

    // 检查合约执行结果
    String retValue = resultObject.getJSONArray("ret").getJSONObject(0).getString("contractRet");
    return "SUCCESS".equals(retValue);
}
```

---

## 11. 智能合约调用层（Trc20 + SmartContract）

### 11.1 TRC20余额查询

`Trc20.java:37-49`:
```java
public static BigDecimal balanceOf(String address, String contract) {
    String addressHex = TronUtil.toHexAddress(address);
    String param = encoderBalanceOfAbi(addressHex.substring(2));  // ABI编码address

    // 调用合约的只读方法
    String result = SmartContract.triggerConstantContract(address, contract, "balanceOf(address)", param);

    BigDecimal balance = decoderBalanceOf(result);   // web3j ABI解码Uint256
    Integer decimals = decimals(contract);            // 查精度（USDT=6）
    return balance.divide(BigDecimal.TEN.pow(decimals));
}
```

### 11.2 gRPC合约调用

`SmartContract.java:21-49` — 只读调用：
```java
public static String triggerConstantContract(ownerAddress, contractAddress, methodStr, data) {
    byte[] owner = WalletApi.decodeFromBase58Check(ownerAddress);
    byte[] inputBytes = Hex.decode(AbiUtil.parseMethod(methodStr, data, true));
    byte[] contractBytes = WalletApi.decodeFromBase58Check(contractAddress);

    // 构建合约调用参数
    TriggerSmartContract triggerContract = WalletApi.triggerCallContract(owner, contractBytes, 0, inputBytes, 0, null);

    // gRPC调用TRON全节点
    TransactionExtention result = TronUtil.getRpcCli().triggerConstantContract(triggerContract);

    // 返回hex编码的结果
    byte[] resultBytes = result.getConstantResult(0).toByteArray();
    return Hex.toHexString(resultBytes);
}
```

`SmartContract.java:51-90` — 状态变更调用（含签名）：
```java
public static String triggerContract(ownerAddress, privateKey, contractAddress, methodStr, data, feeLimit) {
    // ... 构建参数同上 ...

    // gRPC调用（写操作）
    TransactionExtention result = TronUtil.getRpcCli().triggerContract(triggerContract);

    // 设置fee上限
    rawBuilder.setFeeLimit(feeLimit);  // 10,000,000 SUN = 10 TRX

    // 签名并广播
    return TronUtil.processTransactionExtention(result, privateKey);
}
```

### 11.3 ABI编码

使用web3j库进行Solidity ABI编码：
```java
// balanceOf(address) 参数编码
private static String encoderBalanceOfAbi(String address) {
    ArrayList<Type> params = new ArrayList<>();
    params.add(new Address(address));
    return FunctionEncoder.encodeConstructor(params);
}

// transfer(address, uint256) 参数编码
private static String encoderTransferAbi(String hexAddress, BigInteger amount) {
    ArrayList<Type> params = new ArrayList<>();
    params.add(new Address(hexAddress));
    params.add(new Uint256(amount));
    return FunctionEncoder.encodeConstructor(params);
}
```

---

## 12. gRPC与HTTP双通道架构

wallet-server使用**两种方式**与TRON区块链交互：

### 12.1 HTTP JSON-RPC（主要用于wallet-server的WalletTrxBusiness）

| 接口 | 用途 | 调用方 |
|------|------|--------|
| `POST /wallet/getaccount` | 查TRX余额 | `getBalanceByAddress()` |
| `POST /wallet/validateaddress` | 验证地址格式 | `validateAddress()` |
| `POST /wallet/triggersmartcontract` | 创建TRC20交易 | `trc20Transaction()` |
| `POST /wallet/broadcasthex` | 广播已签名交易 | `trc20Transaction()` |
| `POST /wallet/gettransactionbyid` | 查交易状态 | `getTransactionInfoById()` |

端点：`https://nile.trongrid.io`（Nile测试网）

### 12.2 gRPC（主要用于SmartContract + TronUtil）

| 方法 | 用途 | 调用方 |
|------|------|--------|
| `triggerConstantContract` | 只读合约调用（balanceOf/decimals） | `SmartContract.triggerConstantContract()` |
| `triggerContract` | 写合约调用（transfer） | `SmartContract.triggerContract()` |
| `broadcastTransaction` | 广播交易 | `TronUtil.processTransactionExtention()` |
| `getTransactionSignWeight` | 验证签名权限 | `TronUtil.signTransaction()` |

端点：`grpc.nile.trongrid.io:50051`（config.conf配置）

### 12.3 为什么存在两套？

- **SmartContract + TronUtil**（gRPC）是基于TRON官方SDK的**原生调用方式**，用于Trc20.balanceOf()等只读查询
- **WalletTrxBusiness**（HTTP）是通过TronGrid HTTP API的**自定义调用方式**，用于需要更多控制的场景（如自定义签名流程、外部签名服务集成）

这种双通道设计是**历史演进的结果**，初期可能只用gRPC，后来加入外部签名服务后改用HTTP方式以获得更大的灵活性。

---

## 13. 密钥管理体系——三层加密

### 第一层：用户私钥 — AES对称加密

```
原始私钥(hex) → AES/ECB/PKCS5Padding(明文, SHA1(passphrase + address)[0:16]) → enPrivateKey(DB存储)
```

- 算法：AES/ECB/PKCS5Padding
- 密钥派生：SHA-1(passphrase + address) 取前16字节
- passphrase本身由第二层保护

### 第二层：passphrase — AWS KMS加密

```
passphrase明文 → AWS KMS Encrypt(keyId) → Base64编码 → application.yml配置值
```

- 所有yml中的地址/口令/私钥都是KMS加密后的密文
- 运行时调用 `AwsKmsDriver.decryptData()` 解密
- KMS区域：AP_SOUTHEAST_1

### 第三层：外部签名服务 — RSA加密通信

```
签名请求(orderNo, outAddressNo, chainType, hashId)
  → RSA Encrypt(RSA公钥from配置表)
  → HTTP POST到签名服务
  → 返回签名值(Base64)
```

- 用于高价值交易（提现出款）
- RSA公钥存在配置表（trx_wallet_config）
- 平台认证头：`PLAT_CODE: xxx`

### 密钥流向图

```
                    ┌─ AWS KMS ─────────────────────┐
                    │  keyId: mrk-2017a36c...       │
                    │  Region: ap-southeast-1       │
                    └──────────┬────────────────────┘
                               │ 解密
                    ┌──────────▼────────────────────┐
                    │ passphrase (助记词级口令)       │
                    │ companyAddress (主钱包地址)     │
                    │ gasAddress (gas费地址)          │
                    │ gasPrivateKey (gas账户私钥)     │
                    └──────────┬────────────────────┘
                               │ 作为AES密钥一部分
                    ┌──────────▼────────────────────┐
                    │ AES Decrypt                    │
                    │ key = SHA1(passphrase+address) │
                    │ → 用户私钥(hex)                │
                    └──────────┬────────────────────┘
                               │ 签名交易
                    ┌──────────▼────────────────────┐
                    │ ECKey.sign(txHash)             │
                    │ → 签名后的交易                  │
                    │ → 广播到TRON网络               │
                    └───────────────────────────────┘
```

---

## 14. 外部签名服务集成

### 14.1 签名服务调用流程

`WalletTrxBusiness.java:292-321` — `sendSign()`:

```java
private byte[] sendSign(String signUrl, Protocol.Transaction tx, String ssOrderId, String from) {
    // 1. 从配置表读取签名服务配置
    List<TrxWalletConfigEntity> list = trxWalletConfigRepository.findByType("sign");

    // 2. 计算交易hash
    byte[] txId = Sha256Sm3Hash.hash(tx.getRawData().toByteArray());
    String signTxid = Base64.getEncoder().encodeToString(txId);

    // 3. 构建签名请求
    Map<String, Object> signMap = new HashMap<>();
    signMap.put("orderNo", ssOrderId);
    signMap.put("outAddressNo", from);
    signMap.put("chainType", "TRON");
    signMap.put("hashId", signTxid);

    // 4. RSA加密请求体
    String rsaPublicKey = list.filter(code=="rsa").getValue();
    String enSign = RSAUtil.encrypt(JSON.toJSONString(signMap), rsaPublicKey);

    // 5. HTTP POST到签名服务
    HttpResponse response = HttpRequest.post(signUrl + "/api/admin/chain/sign")
            .header("PLAT_CODE", list.filter(code=="platCode").getValue())
            .body(enSign)
            .timeout(60000)
            .execute();

    // 6. 解析签名结果
    JSONObject result = JSON.parseObject(response.body());
    if ("200".equals(result.get("code"))) {
        byte[] deSign = Base64.getDecoder().decode(result.getString("data"));
        // 组装完整的已签名交易
        return tx.toBuilder().addSignature(ByteString.copyFrom(deSign)).build().toByteArray();
    }
    return null;
}
```

### 14.2 签名决策逻辑

```java
// trc20Transaction中的签名路由
if (disPendAddress.equals(from)) {
    // 出款地址 → 必须走外部签名服务（高安全级别）
    bytes = sendSign(signUrl, tx, ssOrderId, from);
} else {
    // 用户归集地址 → 本地签名（低安全级别，用户私钥在DB中）
    bytes = signTransaction2Byte(tx.toByteArray(), ByteArray.fromHexString(keyStore));
}
```

**设计思路**：出款地址持有大量资金，私钥不应在wallet-server本地存储，而是托管在独立的签名服务中，实现职责分离。

---

## 15. Gas费自动补充机制

TRC20代币转账需要TRX作为矿工费（Energy/Bandwidth），系统自动检测并补充。

### 15.1 补充逻辑

在 `singleCollect()` 和 `withdrawTraction()` 中都有相同的逻辑：

```java
// 常量定义
private static final BigDecimal FEE_BAL = new BigDecimal("30");  // 最低TRX余额

// 检查TRX余额是否足够
if (!TRX合约) {
    BigDecimal addTrx = getAccountBalance(fromAddress, "TRX");
    if (FEE_BAL.compareTo(addTrx) > 0) {
        // TRX不足30 → 从gas专用地址转30 TRX
        TrxWalletResult result = createTransactionForTrx(
            awsKmsDriver.decryptData(gasPassword),  // gas地址私钥
            FEE_BAL,                                 // 30 TRX
            gasAddress,                              // 从gas地址
            fromAddress,                             // 到用户/出款地址
            TRX合约,                                 // TRX主币
            ssOrderId
        );
        // 记录gas补充订单
        trxWalletOrderService.save(userId, gasAddress, fromAddress, txId, ssOrderId, FEE_BAL, ...);
    }
}
```

### 15.2 Gas费设计要点

| 参数 | 值 | 说明 |
|------|-----|------|
| 最低TRX余额阈值 | 30 TRX | 低于此值触发补充 |
| 单次补充量 | 30 TRX | 固定补充30 TRX |
| Gas地址 | 配置中的gasAddress | 专用gas费地址，KMS加密 |
| fee_limit | 50,000,000 SUN (50 TRX) | TRC20交易最大手续费 |

---

## 16. 定时任务编排

| 任务 | Cron | 时区 | 模块 | 职责 |
|------|------|------|------|------|
| `checkVirtualCurrencyTask` | `0 */2 * * * ?` | Bangkok | user-server | 每2分钟检查USDT充值订单 |
| `collectUSDT` | `0 0 8 * * ?` | Bangkok | wallet-server | 每日08:00批量归集USDT |
| `withdrawVirtualCurrencyTask` | 已注释 | Bangkok | user-server | 提现定时确认（改为即时触发） |

### 充值定时任务流程

```
每2分钟 → HandleVirtualCurrencyTask.checkVirtualCurrencyTask()
  → VirtualCurrencyBusiness.checkChargeCrypotcurrencyOrder("USDT", 30)
    ├→ timeoutOrder(): 超30min的WAIT充值订单 → SYSTEM_CANCLE
    └→ findWaitOrderWithinHours(): 30min内的WAIT充值订单
       → 逐个 AsyncVirtualDepositService.deposit()
         → Redisson FairLock(orderId)
         → checkCrypotcurrencyOrder()
           ├→ 第一次(无txId): 查余额→触发归集→保存txId→return
           └→ 第二次(有txId): 查确认→成功→增加余额→通知
```

---

## 17. 跨服务调用拓扑

```
┌────────────────────────────────────────────────────────────────┐
│                         user-server                            │
│                                                                │
│  VirtualCurrencyBusiness ─→ AsyncVirtualDepositService         │
│  VirtualCurrencyWithdrawBusiness ─→ AsyncVirtualWithdrawService│
│                    │                       │                   │
│                    └───── WalletBusiness ───┘                  │
│                              │ (HTTP REST)                     │
└──────────────────────────────┼──────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │  POST /wallet/ │                │
              │  ├ create      │                │
              │  ├ getBalance  │                │
              │  ├ collectTrc20│                │
              │  ├ withdrawTraction             │
              │  └ getTransactionInfoById       │
              │                │                │
┌─────────────┼────────────────┼────────────────┼─────────────────┐
│             ▼                                                   │
│                     wallet-server (port 9600)                   │
│                                                                 │
│  WalletController ─→ TrxDriver ─→ WalletTrxBusiness            │
│                                         │                       │
│                    ┌────────────────────┼──────────────────┐    │
│                    │                    │                  │    │
│                    ▼                    ▼                  ▼    │
│              TronUtils           Trc20/SmartContract  AwsKmsDriver│
│              (HTTP)              (gRPC)               (AWS SDK)  │
│                    │                    │                  │    │
└────────────────────┼────────────────────┼──────────────────┼────┘
                     │                    │                  │
              ┌──────▼──────┐   ┌────────▼────────┐  ┌─────▼─────┐
              │ TRON HTTP   │   │ TRON gRPC Node  │  │ AWS KMS   │
              │ nile.tron   │   │ grpc.nile.tron  │  │ ap-se-1   │
              │ grid.io     │   │ grid.io:50051   │  │           │
              └─────────────┘   └─────────────────┘  └───────────┘
                                                           │
                                                     ┌─────▼───────┐
                                                     │ 外部签名服务  │
                                                     │ 34.92.121.9 │
                                                     └─────────────┘
```

**wallet-server → user-server回调**：
- `POST /user/updateWalletAddressById`：创建钱包后通知更新用户钱包地址

**user-server → web-server通知**：
- `POST /wallet/user/userDepositAndWithdrawalNoticeTg`：充值/提现成功后通知前端

**user-server → backend-server统计**：
- `POST /wallet/user/userDepositAndWithdrawalNoticeTg`：发送统计数据（HTTP替代MQ）

---

## 18. 枚举与常量体系

### 18.1 代币合约枚举（TRXContractEnum）

```java
TRX(1, "trx",                                   "TRX",  6,  "TRX"),     // 主币
USDT(2, WalletUtil.getContractByCoin("USDT"),     "USDT", 6,  "TRC20"),   // 动态读取配置
CXT(3, "TBfSX66SpPigEGKgH1YaviR93DnNeeZeaE",   "CXT",  18, "TRC20"),   // 固定合约
```

**设计要点**：USDT合约地址从yml动态读取，支持主网/测试网切换而不改代码。

### 18.2 订单类型枚举

| 枚举值 | Code | 说明 |
|--------|------|------|
| RECHARGE | 1 | 充值/归集入账 |
| WITHDRAW | 2 | 用户提现出账 |

### 18.3 归集类型枚举

| 枚举值 | Code | 说明 |
|--------|------|------|
| SINGLE | 1 | 单笔即时归集（充值触发） |
| COLLECT | 2 | 定时批量归集（每日08:00） |

### 18.4 签名服务常量

```java
public static final String SIGN = "sign";
public static final String RSA = "rsa";
public static final String PLATCODE = "platCode";
public static final String DISPENDADDRESS = "disPendAddress";
public static final String PATH_SIGN = "/api/admin/chain/sign";
```

---

## 19. 设计模式与架构特征提炼

### 19.1 分层架构

```
Controller → Driver(编排层) → Business(区块链交互层) → client/trx(协议层)
                               └→ Service(数据层) → Repository(JPA)
```

- **Driver层**是关键的编排层，处理业务逻辑（余额检查、gas补充、幂等、记录）
- **Business层**专注于区块链交互（签名、广播、查询）
- **client/trx层**封装底层协议（gRPC/Protobuf）

### 19.2 两阶段轮询模型

充值确认采用"发起→等待→确认"的两阶段模型：
1. 第一次轮询：检查余额 → 发起归集 → 获得txId → **立即返回**
2. 第二次轮询：用txId查确认 → 确认成功 → 上分

**避免了**：长连接等待、WebSocket推送等复杂机制。
**代价是**：充值确认延迟最长4分钟（2次定时任务间隔）。

### 19.3 热钱包per-user模型

每个用户一个独立链上地址：
- **优点**：天然隔离，充值到账可直接关联用户
- **缺点**：地址数量膨胀，归集成本增加

### 19.4 集中出款模型

所有提现从单一出款地址（disPendAddress）发出：
- **优点**：资金集中管理，签名控制简单
- **缺点**：单点风险

### 19.5 双签名模式

- **本地签名**：归集操作（用户私钥在DB，风险可控）
- **外部签名**：提现操作（出款地址私钥不在本地，安全隔离）

---

## 20. 值得借鉴的设计（直接采纳）

### 20.1 两阶段轮询充值确认（★★★）

**原理**：定时任务轮询 + 两阶段状态推进（查余额→归集→确认），简单可靠。
**移植建议**：Go中用cron调度器（如 `robfig/cron`）替代Spring @Scheduled。

### 20.2 Gas费自动补充机制（★★★）

**原理**：TRC20转账前检查TRX余额，不足则从gas专用地址自动补充。
**关键参数**：阈值30 TRX，fee_limit 50 TRX。
**移植建议**：这是链上操作必须的环节，直接复制逻辑。

### 20.3 per-user热钱包地址模型（★★★）

**原理**：每个用户一个链上地址，充值自动关联用户。
**移植建议**：Go中用 `go-ethereum/crypto` 生成secp256k1密钥对。

### 20.4 归集机制（即时+每日定时）（★★★）

**原理**：充值即归集 + 每日兜底归集，资金不长时间停留在用户地址。
**移植建议**：直接复用，Go中用goroutine并发归集。

### 20.5 AWS KMS密钥管理（★★）

**原理**：敏感配置（地址、口令、私钥）经KMS加密后存放在配置中。
**移植建议**：Go有 `aws-sdk-go-v2/service/kms`，直接对接。

### 20.6 外部签名服务分离（★★）

**原理**：出款地址私钥不在业务服务中，通过RSA加密通信调用签名服务。
**移植建议**：根据安全等级决定是否需要，初期可简化为本地签名+HSM。

### 20.7 ssOrderId幂等防重（★★）

**原理**：通过业务订单ID查询链上交易表，已有记录直接返回txId。
**移植建议**：我们的唯一索引(order_no + flow_type)已覆盖，可进一步加入txId防重。

### 20.8 订单状态CAS更新（★★）

**原理**：更新订单状态时带上旧状态条件（WHERE status = WAIT），防止并发重复处理。
**移植建议**：我们已在使用 `(amount - X) >= 0` 的SQL级安全，可结合使用。

### 20.9 Redisson FairLock充值/提现并发控制（★）

**原理**：每个订单一把分布式锁，0秒等待，抢不到就下次再来。
**移植建议**：Go中用Redis SetNX或Redlock。

---

## 21. 需要改进/不采纳的设计

### 21.1 AES/ECB模式（安全缺陷）（✗）

ECB模式不使用IV，相同明文产生相同密文，存在模式泄露风险。
**建议**：改用AES/GCM或AES/CBC模式。

### 21.2 HTTP REST替代MQ（架构退化）（✗）

代码中有大量注释掉的RabbitMQ调用，改为HTTP同步调用：
```java
// rabbitTemplate.convertAndSend(MqConstants.BACKEND_ORDER_STATISTICS, jsonStr);
String url = backendServerUrl + BackendServerConstants.WALLET_USER_...;
HttpClient4Util.doPost(url, jsonStr);
```
**问题**：同步HTTP不可靠，目标服务不可用则丢失消息。
**建议**：我们使用NATS JetStream，保持异步消息投递。

### 21.3 私钥存数据库（高风险）（⚠）

用户私钥虽然AES加密后存DB，但只要拿到passphrase（KMS解密）就能批量解密所有私钥。
**建议**：考虑HD钱包（BIP44），只存主密钥，派生子密钥按需计算；或使用MPC签名。

### 21.4 无事务保证的多步操作（⚠）

提现流程中的"解冻→扣余额→扣手续费→更新状态"四步操作没有数据库事务包裹：
```java
doFreezeAmount(ssOrder, user);           // 步骤1
reduceRealAmount(ssOrder, user, ...);    // 步骤2
doBalanceFeeBusiness(ssOrder, user, ...); // 步骤3
ssOrderService.updateWithdrawOrderStatus(...); // 步骤4
```
任意步骤失败会导致数据不一致。
**建议**：我们的设计应在单次事务中完成所有余额操作。

### 21.5 SpringSecurity全放行（✗）

```java
// WebSecurityConfig.java
.authorizeRequests().anyRequest().permitAll()
```
wallet-server没有任何接口认证。
**建议**：内部服务间至少使用mTLS或Token认证。

### 21.6 硬编码的30分钟超时（⚠）

充值/提现超时阈值硬编码为30分钟，不可配置。
**建议**：改为配置中心管理。

### 21.7 BigDecimal精度处理不统一（⚠）

- trx_wallet_order.amount 为 DECIMAL(16,2)
- TRC20精度为6位（USDT）或18位（CXT）
- 业务中有些地方 setScale(2) 有些没有

**建议**：我们的BIGINT最小货币单位方案更统一。

### 21.8 config.conf中gRPC节点单点（⚠）

```conf
fullnode.ip.list = ["grpc.nile.trongrid.io:50051"]
```
只有一个gRPC节点，无failover。
**建议**：配置多个节点，客户端侧负载均衡。

### 21.9 CloseableHttpClient未复用（性能问题）（✗）

`TronUtils.java:451-452`:
```java
CloseableHttpClient httpClient = HttpClients.createDefault();  // 每次新建！
```
每次HTTP请求都创建新的HttpClient，无连接池。
**建议**：Go中使用http.Client单例 + 连接池。

---

## 22. 与我们Go工程的映射对照

| 参考工程概念 | Go工程对应 | 移植策略 |
|-------------|-----------|---------|
| TrxWalletEntity | `t_wallet_address`（新增表） | 用户链上地址表，存AES加密私钥 |
| TrxWalletOrderEntity | `t_chain_transaction`（新增表） | 链上交易记录表 |
| TrxWalletCollectEntity | `t_wallet_collect`（新增表） | 归集记录表 |
| TrxWalletConfigEntity | ETCD配置 / `t_system_config` | 链上配置（合约地址、签名服务等） |
| SsOrder | `t_recharge_order` / `t_withdraw_order` | 充值/提现订单（已有设计） |
| WalletTrxBusiness | `service/chain/tron.go` | TRON链上交互封装 |
| TrxDriver | `service/chain/driver.go` | 链上操作编排层 |
| Trc20 + SmartContract | `pkg/tron/trc20.go` | TRC20代币操作封装 |
| TronUtils | `pkg/tron/utils.go` | 地址转换、签名、Protobuf打包 |
| AwsKmsDriver | `pkg/kms/aws.go` | AWS KMS对接 |
| AES.java | `pkg/crypto/aes.go` | AES加解密（**改用GCM模式**） |
| CollectTrxJob | `task/collect.go` | 归集定时任务 |
| AsyncVirtualDepositService | `service/deposit.go` | 充值确认逻辑 |
| AsyncVirtualWithdrawService | `service/withdraw.go` | 提现确认逻辑 |
| HandleVirtualCurrencyTask | `task/deposit_check.go` | 充值轮询定时任务 |
| web3j | `github.com/fbsobreira/gotron-sdk` 或自建 | TRON Go SDK |
| wallet-cli-1.0.jar | Go原生gRPC客户端 | 直接用Protobuf/gRPC |

---

## 23. TRC20充值实现移植指南

### 23.1 核心移植清单

**必须移植的模块（按优先级）**：

1. **地址生成模块** — 生成TRON兼容密钥对，Base58Check编码
2. **TRC20余额查询** — 调用合约balanceOf
3. **TRC20转账** — 调用合约transfer（含ABI编码）
4. **交易签名** — 本地ECKey签名
5. **交易广播** — HTTP /wallet/broadcasthex
6. **交易确认查询** — HTTP /wallet/gettransactionbyid
7. **Gas费自动补充** — TRX余额检测 + 自动转账
8. **归集流程** — 单笔归集 + 批量归集
9. **充值轮询** — 两阶段轮询确认
10. **密钥加密存储** — AES/GCM加密私钥

### 23.2 Go生态中的替代库

| Java库 | Go替代 | 说明 |
|--------|--------|------|
| web3j/crypto (ECKeyPair) | `crypto/ecdsa` + `secp256k1` | 标准库 |
| web3j/abi (FunctionEncoder) | `github.com/ethereum/go-ethereum/accounts/abi` | ABI编码 |
| tron wallet-cli | `github.com/fbsobreira/gotron-sdk` | TRON Go SDK |
| gRPC (io.grpc) | `google.golang.org/grpc` | Go gRPC原生支持 |
| Protobuf | `google.golang.org/protobuf` | TRON Protobuf定义 |
| AWS KMS SDK | `github.com/aws/aws-sdk-go-v2/service/kms` | AWS Go SDK v2 |
| BouncyCastle | Go标准库 `crypto` | Go密码学库更完善 |

### 23.3 关键技术点（移植时注意）

**1. TRON地址生成**
```
secp256k1私钥 → 公钥 → Keccak256(pubKey[1:]) → 取后20字节 → 前缀0x41 → Base58Check → T开头地址
```

**2. USDT精度**
```
USDT精度 = 6位
转账金额 = 用户输入金额 × 10^6
例：100 USDT → 100000000（uint256）
```

**3. fee_limit设置**
```
TRC20 transfer 建议 fee_limit = 50,000,000 SUN = 50 TRX
TRX transfer 无需 fee_limit
```

**4. 交易确认时间**
```
TRON出块时间 ≈ 3秒
建议至少等待1-2个区块确认后查询
定时任务2分钟间隔已足够覆盖确认时间
```

**5. 测试网 vs 主网**
```
测试网：nile.trongrid.io / grpc.nile.trongrid.io:50051
主网：  api.trongrid.io  / grpc.trongrid.io:50051
USDT合约测试网：TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf
USDT合约主网：  TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t
```

---

## 附录A：application.yml关键配置项

```yaml
server:
  port: 9600

wallet:
  trx:
    url: https://nile.trongrid.io           # TRON HTTP RPC节点
    companyAddress: [KMS加密密文]             # 公司主钱包地址
    mainCollectAddress: [KMS加密密文]         # 归集目标地址（=companyAddress）
    mainCollectPassphrase: [KMS加密密文]      # 归集口令（AES密钥的一部分）
    mainDispensingAddress: [KMS加密密文]      # 出款地址
    mainDispensingPrivateKey: [KMS加密密文]   # 出款私钥
    gasAddress: [KMS加密密文]                 # Gas费专用地址
    gasPrivateKey: [KMS加密密文]              # Gas费地址私钥
    USDT: TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf # USDT合约地址（Nile测试网）

sign:
  url: http://34.92.121.9:80                # 外部签名服务地址

spring:
  datasource:
    url: jdbc:mysql://...:3306/cloud_wallet  # 独立的wallet数据库
  redis:
    database: 11                             # Redis独立库
  rabbitmq:                                  # RabbitMQ配置（实际已改为HTTP）

aws:
  accessKeyId: AKIA...                       # AWS访问密钥
  secretAccessKey: M2LU...                   # AWS密钥
  keyId: mrk-2017a36c...                     # KMS主密钥ID
```

## 附录B：config.conf（TRON网络配置）

```conf
net { type = mainnet }
fullnode = { ip.list = ["grpc.nile.trongrid.io:50051"] }
RPC_version = 2
blockNumberStartToScan = 22690588
```

## 附录C：关键源文件快速索引

| 文件 | 行数 | 核心方法 |
|------|------|---------|
| `WalletTrxBusiness.java` | 538 | createWallet(), trc20Transaction(), trxOfflineSignature(), sendSign(), getBalanceByAddress(), getTrc20Account(), getTransactionInfoById() |
| `TrxDriver.java` | 338 | singleCollect(), collectTrc20(), withdrawTraction(), getAccountBalance(), createTransactionForTrx() |
| `WalletController.java` | 143 | create, getBalance, collectTrc20, withdrawTraction, getTransactionInfoById, getTrxWalletAccount |
| `AsyncVirtualDepositService.java` | 454 | checkCrypotcurrencyOrder(), deposit(), updateOrderStatusAndSendMQ(), addBalance(), doPlayMoney() |
| `AsyncVirtualWithdrawService.java` | 361 | withdrawVirtualCurrencyTask(), withdrawUSDT(), doFreezeAmount(), reduceRealAmount(), doBalanceFeeBusiness() |
| `VirtualCurrencyBusiness.java` | 248 | checkChargeCrypotcurrencyOrder(), timeoutOrder(), handleChargeCrypotcurrencyOrder() |
| `VirtualCurrencyWithdrawBusiness.java` | 336 | checkVirtualWithdrawTask(), withdraw(), withdrawFail() |
| `WalletBusiness.java` | 234 | getBalance(), getTransferResult(), getTransactionInfoById(), withdrawTransaction() |
| `SmartContract.java` | 91 | triggerConstantContract(), triggerContract() |
| `Trc20.java` | 125 | balanceOf(), decimals(), transfer() |
| `TronUtil.java` | 122 | processTransactionExtention(), signTransaction(), getRpcCli() |
| `TronUtils.java` | 473 | packTransaction(), signTransactionByte(), toHexAddress(), createAddress(), signAndBroadcast() |
| `CollectTrxJob.java` | 40 | collectUSDT() |
| `HandleVirtualCurrencyTask.java` | 66 | checkVirtualCurrencyTask() |
| `SsOrder.java` | 220 | 28个字段的业务订单实体 |
