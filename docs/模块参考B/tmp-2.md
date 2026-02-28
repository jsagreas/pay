# 参考工程B深度分析：tg-services 虚拟币钱包系统

> **分析对象**：`/Users/mac/gitlab/z-tmp/tg-services/`
> **核心价值**：TRC20 虚拟币充值/提现完整实现思路
> **产品形态**：Telegram Bot + 在线博彩 + TRON链上钱包
> **技术栈**：Spring Boot 2.5.3 + JPA/Hibernate + web3j + TRON SDK + Redisson + AWS KMS

---

## 目录

1. [项目概述与架构总览](#1-项目概述与架构总览)
2. [工程结构与模块划分](#2-工程结构与模块划分)
3. [数据库设计](#3-数据库设计)
4. [钱包地址生成与密钥管理](#4-钱包地址生成与密钥管理)
5. [TRC20 充值完整链路](#5-trc20-充值完整链路)
6. [TRC20 提现完整链路](#6-trc20-提现完整链路)
7. [归集机制](#7-归集机制)
8. [Gas 费补充策略](#8-gas-费补充策略)
9. [智能合约交互层](#9-智能合约交互层)
10. [安全架构](#10-安全架构)
11. [定时任务体系](#11-定时任务体系)
12. [枚举与常量设计](#12-枚举与常量设计)
13. [跨服务通信架构](#13-跨服务通信架构)
14. [Bug与设计缺陷分析](#14-bug与设计缺陷分析)
15. [对 ser-wallet 的借鉴价值](#15-对-ser-wallet-的借鉴价值)
- [附录A：完整 API 端点清单](#附录a完整-api-端点清单)
- [附录B：完整 Redis Key 清单](#附录b完整-redis-key-清单)
- [附录C：关键文件索引](#附录c关键文件索引)

---

## 1. 项目概述与架构总览

### 1.1 产品定位

tg-services 是一个 **基于 Telegram Bot 的在线博彩 + 加密货币钱包平台**。用户通过 Telegram 进行注册、充值、投注、提现全链路操作，资金以 **TRC20 USDT** 为主要币种在 TRON 链上流转。

### 1.2 核心架构

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  telegram-bot-api│      │    web-server    │      │  backend-server  │
│   (TG Bot入口)   │─────→│   (WebSocket/    │─────→│   (后台管理)     │
│    port: 9100    │      │    前端通信)     │      │    port: 9200    │
└──────────────────┘      │    port: 9000    │      └──────────────────┘
        │                 └──────────────────┘              │
        ▼                                                   │
┌──────────────────┐      ┌──────────────────┐              │
│   user-server    │─────→│  wallet-server   │              │
│  (业务编排层)    │ HTTP │  (链上交互层)    │              │
│  port: 9300      │      │  port: 9600      │              │
│                  │      │                  │              │
│ - 订单管理       │      │ - 地址生成       │              │
│ - 余额变更       │      │ - 链上转账       │              │
│ - 充提编排       │      │ - 余额查询       │              │
│ - 定时任务       │      │ - 归集/提现      │              │
└──────────────────┘      │ - 签名管理       │              │
                          └──────────────────┘              │
                                  │                         │
                          ┌───────┴───────┐                 │
                          ▼               ▼                 │
                   ┌────────────┐  ┌────────────┐           │
                   │ TRON Full  │  │ 外部签名   │           │
                   │  Node API  │  │   服务     │           │
                   │ (trongrid) │  │ (HSM/冷钱包)│           │
                   └────────────┘  └────────────┘           │
                                                            ▼
                                                   ┌──────────────┐
                                                   │    MySQL     │
                                                   │ cloud_wallet │
                                                   │ (JPA管理)    │
                                                   └──────────────┘
```

### 1.3 关键设计决策

| 决策项 | 选择 | 原因 |
|--------|------|------|
| 链上交互方式 | HTTP REST API + gRPC 双通道 | HTTP用于交易构建/广播，gRPC用于合约调用 |
| 私钥存储 | AES加密存DB + AWS KMS加密密码 | 平衡安全性与可用性 |
| 提现签名 | 外部签名服务（HSM/冷钱包） | 提现资金从发币地址出，需冷钱包级安全 |
| 归集签名 | 本地签名（内存解密私钥） | 归集是用户地址→公司地址，私钥由系统生成 |
| 服务间通信 | HTTP REST（非RPC/MQ） | 简单直接，wallet-server 无状态 |
| 订单管理 | user-server 统一管理 | wallet-server 只做链上操作，不管业务状态 |

---

## 2. 工程结构与模块划分

### 2.1 模块全景

```
tg-services/
├── telegram-bot-api/       # TG Bot入口 (port 9100)
├── user-server/            # 用户+业务编排 (port 9300)
├── wallet-server/          # 链上钱包交互 (port 9600) ★核心
├── web-server/             # WebSocket通信 (port 9000)
├── backend-server/         # 后台管理 (port 9200)
├── wallet-third-server/    # 第三方游戏钱包 (HuiWang)
├── module-common/          # 共享枚举/VO/工具
├── module-jpa/             # JPA基础配置
├── module-springcache-redis/# Redis工具封装
└── ... (其他辅助模块)
```

### 2.2 wallet-server 内部结构（46个Java文件）

```
wallet-server/src/main/java/com/baisha/walletserver/
├── WalletServerApplication.java         # 启动类
├── business/
│   └── WalletTrxBusiness.java           # ★核心：链上交互业务（539行）
├── controller/
│   ├── WalletController.java            # REST API端点（144行）
│   └── TestController.java              # KMS加密测试
├── driver/
│   ├── TrxDriver.java                   # ★核心：业务驱动层（339行）
│   └── AwsKmsDriver.java               # AWS KMS加解密（111行）
├── client/trx/
│   ├── Trc20.java                       # TRC20合约调用（126行）
│   ├── SmartContract.java               # 智能合约底层（92行）
│   └── TronUtil.java                    # TRON gRPC客户端（123行）
├── util/
│   ├── TronUtils.java                   # ★离线签名/地址工具（474行）
│   ├── AES.java                         # AES加解密（47行）
│   ├── RSAUtil.java                     # RSA加解密（111行）
│   ├── WalletUtil.java                  # YAML配置读取（64行）
│   └── CommonConstant.java              # 常量定义
├── model/
│   ├── BaseEntity.java                  # JPA基类（38行）
│   ├── TrxWalletEntity.java             # 钱包地址表（44行）
│   ├── TrxWalletOrderEntity.java        # 交易订单表（66行）
│   ├── TrxWalletCollectEntity.java      # 归集记录表（49行）
│   ├── TrxWalletConfigEntity.java       # 动态配置表（40行）
│   └── enums/
│       ├── TRXContractEnum.java         # 代币合约枚举（124行）
│       ├── TrxOrderTypeEnum.java        # 订单类型枚举（35行）
│       └── CollectTypeEnum.java         # 归集类型枚举（34行）
├── repository/                          # JPA Repository层
├── service/                             # Service层（薄封装）
├── task/
│   └── CollectTrxJob.java               # 定时归集任务（41行）
└── config/
    ├── ScheduleConfig.java              # 线程池配置
    └── WebSecurityConfig.java           # 安全配置（全放行）
```

### 2.3 user-server 钱包相关文件

```
user-server/src/main/java/com/baisha/userserver/
├── contrloller/
│   └── VirtualCurrencyController.java   # 充提API入口（241行）
├── business/
│   ├── VirtualCurrencyBusiness.java     # 充值业务编排（249行）
│   ├── VirtualCurrencyWithdrawBusiness.java # 提现业务编排（337行）
│   ├── WalletBusiness.java              # wallet-server HTTP客户端（235行）
│   └── UserAssetsBusiness.java          # 资产操作核心（1294行）
├── service/
│   ├── AsyncVirtualDepositService.java  # 异步充值处理（455行）
│   └── AsyncVirtualWithdrawService.java # 异步提现处理（362行）
├── task/
│   └── HandleVirtualCurrencyTask.java   # 充值定时任务（67行）
└── model/
    ├── SsOrder.java                     # 订单实体（221行）
    └── Assets.java                      # 资产实体（119行）
```

### 2.4 职责分离原则

| 层级 | 服务 | 职责 | 知道什么 |
|------|------|------|----------|
| **业务编排层** | user-server | 订单生命周期、余额变更、定时任务调度 | 用户、订单、余额、业务规则 |
| **链上交互层** | wallet-server | 地址生成、链上转账、余额查询、归集 | TRON协议、密钥管理、合约交互 |
| **外部签名层** | 签名服务(独立) | 冷钱包签名 | 私钥、HSM |

> **关键洞察**：wallet-server **不管理业务余额**，它只是一个"链上操作代理"。所有业务余额（Assets.balance）都在 user-server 管理。wallet-server 只负责链上地址余额查询和转账执行。

---

## 3. 数据库设计

### 3.1 wallet-server 库：cloud_wallet（4张表）

#### 3.1.1 TrxWallet（钱包地址表）

```sql
CREATE TABLE TrxWallet (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    userId      VARCHAR(10)  COMMENT '会员账号',
    userName    VARCHAR(30)  COMMENT '会员姓名',
    enPrivateKey VARCHAR(250) COMMENT '加密后的私钥',
    address     VARCHAR(100) COMMENT '钱包地址',
    create_time DATETIME,
    create_by   VARCHAR(255),
    update_time DATETIME,
    update_by   VARCHAR(255),
    INDEX idx_userId (userId)
);
```

**设计要点**：
- 每个用户一个链上地址（`findAddressByUserId`查重）
- `enPrivateKey` 经 AES 加密存储，密钥 = `KMS解密(passphrase) + address`
- 地址格式为 TRON Base58Check（T开头）

#### 3.1.2 TrxWalletOrder（交易订单表）

```sql
CREATE TABLE TrxWalletOrder (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     VARCHAR(10)  COMMENT '会员账号',
    tx_id       VARCHAR(200) COMMENT '交易id (链上hash)',
    ss_order_id VARCHAR(200) COMMENT '充值/提现记录id (业务订单号)',
    amount      DECIMAL(16,2) COMMENT '金额',
    from_address VARCHAR(250) COMMENT '发送地址',
    to_address  VARCHAR(250) COMMENT '接收地址',
    coin_name   VARCHAR(10)  COMMENT '币种类型 (TRX/USDT/CXT)',
    order_type  TINYINT(2)   COMMENT '收付方向 (1充值 2提现)',
    post_type   TINYINT(2)   COMMENT '1:接口方式 2:定时任务方式',
    create_time DATETIME,
    create_by   VARCHAR(255),
    update_time DATETIME,
    update_by   VARCHAR(255),
    INDEX idx_user_id (user_id),
    INDEX idx_order_type (order_type),
    INDEX idx_tx_id (tx_id)
);
```

**设计要点**：
- `ss_order_id` 关联 user-server 的 `SsOrder.id`，是跨服务关联键
- `tx_id` 是链上交易哈希，用于链上确认查询
- `post_type` 区分是用户主动触发还是定时任务触发

#### 3.1.3 trx_wallet_collect（归集记录表）

```sql
CREATE TABLE trx_wallet_collect (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    userId          VARCHAR(10)  COMMENT '会员账号',
    txId            VARCHAR(200) COMMENT '交易id',
    collectAddress  VARCHAR(250) COMMENT '汇款账号 (用户地址)',
    amount          DECIMAL(16,2) COMMENT '汇款金额',
    collectType     TINYINT(2)   COMMENT '归集类型 (1单笔即时 2定时任务)',
    create_time     DATETIME,
    INDEX idx_userId (userId),
    INDEX idx_collectType (collectType),
    INDEX idx_txId (txId)
);
```

#### 3.1.4 TrxWalletConfig（动态配置表）

```sql
CREATE TABLE TrxWalletConfig (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    type        VARCHAR(30)   COMMENT '类型',
    code        VARCHAR(50)   COMMENT '标识code',
    value       VARCHAR(1000) COMMENT '标识value',
    create_time DATETIME,
    update_time DATETIME
);
```

**已知配置行**（代码中硬编码引用）：

| type | code | 用途 |
|------|------|------|
| `sign` | `rsa` | 外部签名服务的RSA公钥（Base64） |
| `sign` | `platCode` | 商户标识（签名请求Header） |
| `sign` | `disPendAddress` | 发币地址（提现出金地址） |

### 3.2 user-server 库：关键表

#### 3.2.1 SsOrder（订单表）

**源码**：`user-server/.../model/SsOrder.java`（221行）

核心字段（仅列出与虚拟币充提相关的）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `orderNum` | varchar(32) | 订单编号 |
| `userId` | bigint(20) | 会员ID |
| `tgUserId` | varchar(64) | Telegram用户ID |
| `amount` | decimal(16,2) | 总金额（含手续费） |
| `realAmount` | decimal(16,2) | 实际到账金额 |
| `feeAmount` | decimal(16,2) | 手续费，默认0 |
| `orderType` | tinyint(2) | 1充值 2提现 |
| `orderStatus` | tinyint(2) | 默认1，虚拟币用11（待处理） |
| `virtualCurrency` | varchar(20) | 虚拟币种（USDT/TRX/CXT） |
| `payeeAddress` | varchar(200) | 收款地址（提现目标地址） |
| `txId` | varchar(200) | 链上交易ID |
| `flowMultiple` | decimal(16,2) | 流水倍数，默认1 |
| `userType` | tinyint(2) | 1正式 2测试 3机器人 |

#### 3.2.2 Assets（资产表）

**源码**：`user-server/.../model/Assets.java`（119行）

| 字段 | 类型 | 说明 |
|------|------|------|
| `userId` | bigint(20) UNIQUE | 会员ID |
| `balance` | decimal(16,2) | 主余额，默认0 |
| `balanceNn` | decimal(16,2) | 牛牛余额 |
| `balanceBacc` | decimal(16,2) | 百家乐余额 |
| `freezeAmount` | decimal(16,2) | 冻结余额，默认0 |
| `playMoney` | decimal(16,2) | 打码量，默认0 |
| `creditAmount` | decimal(16,2) | 信用额度，默认0 |
| `version` | int | **乐观锁版本号**，默认1 |

> **关键设计**：Assets 使用 `@Version` 乐观锁。余额变更通过 Redisson 公平锁 + JPA乐观锁双重保护。

### 3.3 数据模型关系图

```
user-server                              wallet-server
┌─────────────┐                          ┌─────────────────┐
│   SsOrder   │    ss_order_id           │ TrxWalletOrder  │
│             │←─────────────────────────│                 │
│ id(PK)      │                          │ ssOrderId(关联) │
│ userId      │                          │ txId(链上hash)  │
│ amount      │                          │ fromAddress     │
│ realAmount  │                          │ toAddress       │
│ txId        │                          │ amount          │
│ orderStatus │                          │ orderType(1/2)  │
│ virtualCurr │                          │ coinName        │
│ payeeAddr   │                          └─────────────────┘
└─────────────┘                                  │
       │                                         │
       │ userId                                  │ 归集时生成
       ▼                                         ▼
┌─────────────┐                          ┌──────────────────┐
│   Assets    │                          │trx_wallet_collect│
│             │                          │                  │
│ userId(UK)  │                          │ userId           │
│ balance     │                          │ txId             │
│ freezeAmount│                          │ collectAddress   │
│ version(乐) │                          │ amount           │
└─────────────┘                          │ collectType(1/2) │
                                         └──────────────────┘
                                                 │
                                                 │ userId
                                                 ▼
                                         ┌──────────────────┐
                                         │   TrxWallet      │
                                         │                  │
                                         │ userId           │
                                         │ address(链上)    │
                                         │ enPrivateKey(密) │
                                         └──────────────────┘
```

---

## 4. 钱包地址生成与密钥管理

### 4.1 地址生成流程

**源码**：`WalletTrxBusiness.createWallet()`（539行中第1段）

```
用户注册 ──→ user-server ──→ POST /wallet/create ──→ wallet-server
                                                         │
                              ┌───────────────────────────┘
                              ▼
                    1. AWS KMS 解密 passphrase
                              │
                    2. 检查 userId 是否已有钱包
                       └─ 有 → 直接返回已有地址
                       └─ 无 → 继续
                              │
                    3. web3j ECKeyPair 生成密钥对
                       Keys.createEcKeyPair()
                              │
                    4. 生成 WalletFile
                       Wallet.createStandard(passphrase, ecKeyPair)
                              │
                    5. 构造 TRON 地址
                       "41" + walletFile.getAddress()
                       fromHexAddress() → T... 格式
                              │
                    6. AES 加密私钥
                       secret = passphrase + finalAddress
                       AES/ECB/PKCS5Padding
                       key = SHA-1(secret) 前16字节
                              │
                    7. 存入 DB
                       TrxWallet(userId, userName, enPrivateKey, address)
                              │
                    8. 异步通知 user-server 更新用户钱包地址
                       POST user-server/user/updateWalletAddressById
                              │
                    9. 返回 WalletAddressVO { userId, userName, address }
```

### 4.2 私钥加密方案

```
                    ┌─────────────────────────────────┐
                    │       私钥加密存储方案           │
                    └─────────────────────────────────┘

原始私钥(ECKeyPair.privateKey)
        │
        ▼
AES/ECB/PKCS5Padding 加密
  ├─ 明文: privateKey (hex string)
  ├─ 密钥: SHA-1(passphrase + address) 前16字节
  │         │           │
  │         │           └─ 用户链上地址 (公开信息)
  │         │
  │         └─ passphrase (AWS KMS 加密存储在 application.yml)
  │
  └─ 输出: Base64(ciphertext) → 存入 TrxWallet.enPrivateKey
```

**解密路径**（需要转账时）：
```java
// TrxWalletService.getPrivateKeyByAddress()
1. 从 DB 读取 enPrivateKey
2. AWS KMS 解密 passphrase
3. secret = passphrase + address
4. AES.decrypt(enPrivateKey, secret) → 原始私钥
```

### 4.3 密钥安全层次

| 层次 | 保护对象 | 加密方式 | 存储位置 |
|------|---------|---------|---------|
| L1 | 用户私钥 | AES/ECB/PKCS5 | MySQL TrxWallet.enPrivateKey |
| L2 | AES密码(passphrase) | AWS KMS | application.yml (KMS密文) |
| L3 | 公司地址/Gas地址/私钥 | AWS KMS | application.yml (KMS密文) |
| L4 | 签名服务通信 | RSA 512-bit | TrxWalletConfig.value |
| L5 | 提现私钥 | 外部HSM/冷钱包 | 签名服务内部 |

---

## 5. TRC20 充值完整链路

这是本参考工程**最核心的价值**——完整的虚拟币充值链路实现。

### 5.1 充值流程全景

```
                            ┌──────────────────────────────────────┐
                            │         TRC20 USDT 充值完整流程       │
                            └──────────────────────────────────────┘

阶段1: 用户发起
═══════════════
TG Bot ──→ 用户输入充值金额 ──→ 创建 SsOrder
                                  orderType=1 (CHARGE)
                                  orderStatus=11 (VIRTUAL_CURRENCY_WAIT)
                                  virtualCurrency="USDT"
                                  realAmount=用户输入金额

阶段2: 用户链上转账(链外)
═══════════════════════════
用户 ──→ 在外部钱包(如 TronLink) 向自己的 TG平台地址 转入 USDT
         (这一步发生在链上，平台无法控制)

阶段3: 平台确认
═══════════════
两条路径触发确认：
  路径A: 用户在TG点击"确认充值"按钮
         POST /virtual/charge → VirtualCurrencyController
  路径B: 定时任务每2分钟扫描
         HandleVirtualCurrencyTask.checkVirtualCurrencyTask()

两条路径最终都调用 → AsyncVirtualDepositService.checkCrypotcurrencyOrder()

阶段4: 链上验证 + 归集
═══════════════════════
checkCrypotcurrencyOrder() 执行流程：

  4.1 查询链上余额
      user-server → POST /wallet/getBalance → wallet-server
      wallet-server → TRON API → 返回用户地址上的USDT余额

  4.2 余额校验
      if (balance < realAmount) → 失败
      if (balance == 0) → 失败

  4.3 触发归集(TRC20转账: 用户地址 → 公司地址)
      user-server → POST /wallet/collectTrc20 → wallet-server
      wallet-server → TrxDriver.singleCollect()
        4.3.1 检查用户地址TRX余额 ≥ 30（gas费）
              └─ 不够 → 从gasAddress补充30 TRX
        4.3.2 AES解密用户私钥
        4.3.3 构建TRC20 transfer交易
        4.3.4 本地签名(用户私钥)
        4.3.5 广播到TRON网络
        4.3.6 返回 txId

  4.4 保存 txId
      ssOrderService.updateTxIdById(txId, orderId)

阶段5: 交易确认(第二轮)
═══════════════════════
下一次定时任务或用户再次点击确认时：

  5.1 发现 SsOrder 已有 txId
  5.2 查询链上交易状态
      POST /wallet/getTransactionInfoById → wallet-server
      wallet-server → TRON API gettransactionbyid
      检查 ret[0].contractRet == "SUCCESS"

  5.3 交易成功 → 执行上分
      updateOrderStatusAndSendMQ()
        - 更新 orderStatus → ORDER_SUCCESS(2)
        - 增加用户余额 (addBalance → UserAssetsBusiness.doBalanceBusiness)
        - 增加打码量 (playMoney = realAmount × flowMultiple)
        - 发送统计MQ
        - 通知TG Bot余额变更
```

### 5.2 充值核心代码调用链

```
VirtualCurrencyController.virtualCurrencyCharge()           # 入口
  ├─ Redisson公平锁: "user-server::deposit_or_charge::ss_order_id::{orderId}"
  ├─ 校验 orderStatus == VIRTUAL_CURRENCY_WAIT(11)
  └─ VirtualCurrencyBusiness.handleChargeCrypotcurrencyOrder()
       └─ AsyncVirtualDepositService.checkCrypotcurrencyOrder()
            ├─ [无txId时] walletBusiness.getBalance()           # 查链上余额
            │    └─ POST wallet-server/wallet/getBalance
            │         └─ TrxDriver.getAccountBalance()
            │              └─ WalletTrxBusiness.getTrc20Account()
            │                   └─ Trc20.balanceOf()            # gRPC调TRON合约
            │
            ├─ [余额足够] walletBusiness.getTransferResult()    # 触发归集
            │    └─ POST wallet-server/wallet/collectTrc20
            │         └─ TrxDriver.singleCollect()
            │              ├─ Gas费检查 + 补充
            │              ├─ WalletTrxBusiness.trc20Transaction()  # 构建+签名+广播
            │              └─ reverseNoticeCollectDetail()          # 写入归集记录+订单记录
            │
            ├─ [有txId时] walletBusiness.getTransactionInfoById()  # 确认交易
            │    └─ POST wallet-server/wallet/getTransactionInfoById
            │         └─ TrxDriver.getTransactionInfoById()
            │              └─ WalletTrxBusiness.getTransactionInfoById()  # HTTP查TRON
            │
            └─ [交易确认] updateOrderStatusAndSendMQ()
                 ├─ ssOrderService.updateOrderStatusAndCompleteTime()  # 订单→成功
                 ├─ addBalance()                                        # 增加余额
                 │    └─ UserAssetsBusiness.doBalanceBusiness()
                 │         ├─ Redisson锁: "userServer::assets::{userId}"
                 │         └─ JPA乐观锁(@Version)
                 ├─ doPlayMoney()                                       # 增加打码量
                 └─ sendMQ() → POST backend-server/统计接口             # 通知后台
```

### 5.3 充值状态机

```
                     用户在TG输入金额
                           │
                           ▼
                 ┌─────────────────┐
                 │ VIRTUAL_CURRENCY│
                 │   _WAIT (11)   │
                 └────────┬────────┘
                          │
              ┌───────────┼───────────┐
              ▼           │           ▼
     余额不足/            │     超过30分钟
     链上转账失败          │     未处理
     (保持11，等待)       │           │
                          │           ▼
                    交易确认成功  ┌──────────────┐
                          │     │ORDER_SYSTEM   │
                          ▼     │ _CANCLE (99)  │
                 ┌──────────────┐└──────────────┘
                 │ORDER_SUCCESS │
                 │    (2)       │
                 └──────────────┘
```

### 5.4 充值的两阶段设计

**第一阶段**：触发归集（获取 txId）
- 查链上余额 → 构建归集交易 → 签名广播 → 保存 txId
- 此时 orderStatus 仍为 `VIRTUAL_CURRENCY_WAIT(11)`

**第二阶段**：确认交易（上分）
- 查链上交易状态 → 确认 SUCCESS → 更新订单 + 增加余额
- orderStatus → `ORDER_SUCCESS(2)`

> **为什么分两阶段？** 因为 TRON 链上交易不是即时确认的。归集交易广播后需要等区块确认。第一次触发归集，返回 txId；下一次轮询/用户点击时，用 txId 查交易状态，确认后才上分。

---

## 6. TRC20 提现完整链路

### 6.1 提现流程全景

```
                            ┌──────────────────────────────────────┐
                            │         TRC20 USDT 提现完整流程       │
                            └──────────────────────────────────────┘

阶段1: 用户发起
═══════════════
TG Bot ──→ 用户输入提现金额 + 收款地址
       ──→ 后台审核（创建 SsOrder）
                orderType=2 (WITHDRAW)
                orderStatus=11 (VIRTUAL_CURRENCY_WAIT)
                payeeAddress=用户指定的链上收款地址
                amount=总金额, feeAmount=手续费
                realAmount=实际到账(amount - feeAmount)

阶段2: 前端触发提现
═══════════════════
POST /virtual/withdraw → VirtualCurrencyController
  ├─ 防刷: Redis 10秒冷却
  ├─ 校验订单状态
  ├─ Redisson公平锁: "userServer::virtualCurrency::withdraw_{orderId}"
  └─ VirtualCurrencyWithdrawBusiness.withdraw()

阶段3: 提现执行
═══════════════
withdraw() 执行流程：

  3.1 幂等检查
      - ssOrder.txId 非空 → 已上链，直接返回
      - Redis key "order::withdraw_{orderId}" 存在 → 不重复上链

  3.2 超时检查
      - 订单创建时间超过30分钟 → 不处理，等定时任务取消

  3.3 调用 wallet-server 执行提现
      walletBusiness.withdrawTransaction(ssOrder)
      └─ POST /wallet/withdrawTraction
           body: { amount: 总额-手续费, coinName, ssOrderId, toAddress, userId }

  3.4 wallet-server 执行
      TrxDriver.withdrawTraction()
        3.4.1 从 TrxWalletConfig 获取 disPendAddress（发币地址）
        3.4.2 查询发币地址余额 ≥ 提现金额
        3.4.3 幂等: ssOrderId 已有订单 → 返回已有 txId
        3.4.4 [TRC20] 检查发币地址 TRX ≥ 30，不够则补充
        3.4.5 构建TRC20 transfer交易
              from: disPendAddress → to: payeeAddress
        3.4.6 ★ 使用外部签名服务签名（非本地签名）
              sendSign() → POST sign-server/api/admin/chain/sign
        3.4.7 广播到TRON网络
        3.4.8 保存 TrxWalletOrder 记录

  3.5 保存 txId
      ssOrderService.updateTxIdById(txId, orderId)
      Redis设置30天冷却: "order::withdraw_{orderId}"

阶段4: 提现确认（定时任务）
═══════════════════════════
VirtualCurrencyWithdrawBusiness.checkVirtualWithdrawTask()
  - 查找超过30分钟的 VIRTUAL_CURRENCY_WAIT 提现订单
  - 查询链上交易状态

  成功路径：
    AsyncVirtualWithdrawService.withdrawUSDT()
      1. 解冻金额 (doFreezeAmount → EXPENSES类型)
      2. 扣减实际到账金额 (reduceRealAmount → BalanceChangeEnum.TELEGRAM_WITHDRAW)
      3. 扣减手续费 (doBalanceFeeBusiness → BalanceChangeEnum.FEE)
      4. 更新订单状态 → ORDER_SUCCESS(2)
      5. 发送统计MQ
      6. 通知TG Bot余额变更

  失败路径（transResult == false）：
      1. 解冻金额
      2. 更新订单状态 → ORDER_SYSTEM_CANCLE(99)
```

### 6.2 提现核心代码调用链

```
VirtualCurrencyController.virtualCurrencyWithdraw()           # 入口
  ├─ 防刷: RedisUtil.set("prevent::time::virtualCurrencyWithdraw::{orderId}", 10s)
  ├─ 校验 orderStatus
  ├─ Redisson公平锁: "userServer::virtualCurrency::withdraw_{orderId}"
  └─ VirtualCurrencyWithdrawBusiness.withdraw()
       ├─ 幂等: txId 非空 → return true
       ├─ 幂等: Redis "order::withdraw_{orderId}" → return true
       ├─ 超时: tooLong(30分钟) → return false
       └─ walletBusiness.withdrawTransaction(ssOrder)         # 调wallet-server
            └─ POST wallet-server/wallet/withdrawTraction
                 └─ WalletController.withdrawTraction()
                      └─ TrxDriver.withdrawTraction()
                           ├─ disPendAddress = TrxWalletConfig(sign, disPendAddress)
                           ├─ 余额校验 ≥ sendAmount
                           ├─ 幂等: ssOrderId 已有订单 → 返回
                           ├─ Gas费补充 (if TRC20 && TRX < 30)
                           └─ createTransactionForTrx()
                                └─ WalletTrxBusiness.trc20Transaction()
                                     ├─ triggersmartcontract 构建交易
                                     ├─ packTransaction(json, false) 打包
                                     ├─ ★ sendSign() → 外部签名服务
                                     │    ├─ SHA-256 计算 txId hash
                                     │    ├─ RSA加密请求体
                                     │    └─ POST sign-server/api/admin/chain/sign
                                     └─ broadcasthex 广播
```

### 6.3 充值 vs 提现 签名方式对比

| 维度 | 充值(归集) | 提现 |
|------|-----------|------|
| 资金方向 | 用户地址 → 公司地址 | 发币地址 → 用户外部地址 |
| from地址 | 用户链上地址 | disPendAddress（发币地址） |
| 签名方式 | **本地签名**（解密用户私钥） | **外部签名服务**（HSM/冷钱包） |
| 签名代码 | `signTransaction2Byte(tx, privateKey)` | `sendSign(signUrl, tx, ssOrderId, from)` |
| 风险等级 | 低（归集到公司地址） | 高（资金外流） |

> **为什么提现用外部签名？** 发币地址持有大量资金，其私钥不应出现在应用服务器上。通过外部签名服务（HSM/冷钱包），私钥永远不离开安全硬件，即使应用服务器被攻破也无法窃取资金。

### 6.4 提现确认的五步操作

提现确认成功后，`withdrawUSDT()` 执行5步操作（顺序关键）：

```
Step 1: 解冻金额
  FreezeAmountVO { amount=订单金额, type=EXPENSES(2) }
  → UserAssetsBusiness.doFreezeAmountBusiness()
  → Assets.freezeAmount -= 订单金额

Step 2: 扣减实际到账金额
  BalanceVO { amount=realAmount, type=EXPENSES(2), changeType=TELEGRAM_WITHDRAW(8) }
  → UserAssetsBusiness.doBalanceBusiness()
  → Assets.balance -= realAmount

Step 3: 扣减手续费
  BalanceVO { amount=feeAmount, type=EXPENSES(2), changeType=FEE(90) }
  → UserAssetsBusiness.doBalanceBusiness()
  → Assets.balance -= feeAmount

Step 4: 更新订单状态
  ssOrderService.updateWithdrawOrderStatus(ORDER_SUCCESS, txId, orderId, ...)

Step 5: 通知
  - POST backend-server 统计
  - POST web-server 通知TG Bot余额变更
```

> **⚠️ 设计缺陷**：这5步不在一个数据库事务中。如果 Step 2 成功但 Step 4 失败，会导致余额已扣但订单未更新。详见第14章。

---

## 7. 归集机制

### 7.1 归集概述

"归集"（Collect/Sweep）是将用户链上地址的资金转移到公司主地址的过程。这是虚拟币钱包系统的核心机制——用户充值到自己的专属地址，平台通过归集将分散在各用户地址的资金集中管理。

### 7.2 单笔即时归集

**触发时机**：充值确认时

```
TrxDriver.singleCollect(fromAddress, userId, coinName, ssOrderId, reChargeAmount, collectCode, postType)

参数说明：
  fromAddress    = 用户链上地址
  userId         = 用户ID
  coinName       = 币种名 (USDT)
  ssOrderId      = 业务订单号
  reChargeAmount = 充值金额 (可null，null时查链上余额)
  collectCode    = 归集类型 (1=单笔即时)
  postType       = 1接口/2定时

执行流程：
  1. KMS解密 mainAddress（公司主地址）
  2. 获取 disPendAddress（发币地址，从DB配置）
  3. 如果 reChargeAmount 为 null → 查链上实时余额
  4. 余额 ≤ 0 → 拒绝
  5. [TRC20] 检查用户地址 TRX ≥ 30
     └─ 不够 → 从 gasAddress 补充 30 TRX（见第8章）
  6. 执行转账: fromAddress → mainAddress
     └─ createTransactionForTrx(解密私钥, 金额, from, to, tokenContract, ssOrderId)
  7. 写入记录:
     └─ trx_wallet_collect（归集记录）
     └─ TrxWalletOrder（交易订单）
```

### 7.3 批量定时归集

**触发时机**：每日 08:00 曼谷时间

**源码**：`CollectTrxJob.java`

```java
@Scheduled(cron = "0 0 8 * * ?", zone = "Asia/Bangkok")
public void collectUSDT() {
    TrcCollectRequest req = new TrcCollectRequest();
    req.setCoinName(TRXContractEnum.USDT.getContractName());  // "USDT"
    trxDriver.collectTrc20(req);
}
```

**执行流程**：`TrxDriver.collectTrc20()`

```
1. trxWalletService.findAll() → 获取所有用户钱包
2. 遍历每个钱包:
   - skip if 地址 == mainAddress
   - singleCollect(address, userId, "USDT", null, null, COLLECT(2), 2)
     └─ 查链上余额，有余额就归集
3. 逐个处理，不批量打包
```

### 7.4 归集数据流

```
归集前：
  用户地址A: 100 USDT, 5 TRX
  用户地址B: 200 USDT, 0 TRX  ← 需要补充gas
  公司地址:  50000 USDT

归集时（用户A）：
  Step 1: 检查 TRX ≥ 30 → 5 TRX，够用（实际上30才够，这里有问题，见bug分析）
  Step 2: transfer(A → 公司, 100 USDT)
  Step 3: 写 trx_wallet_collect + TrxWalletOrder

归集时（用户B）：
  Step 1: 检查 TRX ≥ 30 → 0 TRX，不够
  Step 2: transfer(gasAddress → B, 30 TRX)  ← gas补充
  Step 3: transfer(B → 公司, 200 USDT)
  Step 4: 写 trx_wallet_collect + TrxWalletOrder

归集后：
  用户地址A: 0 USDT, ~2 TRX (扣了约3 TRX gas)
  用户地址B: 0 USDT, ~27 TRX (30-3 gas)
  公司地址:  50300 USDT
```

---

## 8. Gas 费补充策略

### 8.1 问题背景

TRC20 代币转账（如 USDT transfer）需要消耗 TRX 作为 gas 费用。用户地址在刚创建时只有 USDT 余额（用户充值的），没有 TRX，无法发起 TRC20 交易。

### 8.2 Gas 费补充机制

**源码**：`TrxDriver.singleCollect()` 和 `TrxDriver.withdrawTraction()`

```java
// 关键常量
private static final BigDecimal FEE_BAL = new BigDecimal("30");  // 需要30 TRX

// 检查逻辑
BigDecimal trxBalance = walletTrxBusiness.getBalanceByAddress(fromAddress);
if (trxBalance.compareTo(FEE_BAL) == -1) {  // < 30 TRX
    // 从 gasAddress 补充 30 TRX
    String gasAddressDe = awsKmsDriver.decryptData(gasAddress);
    String gasPasswordDe = awsKmsDriver.decryptData(gasPassword);
    walletTrxBusiness.trxOfflineSignature(gasPasswordDe, gasAddressDe, fromAddress, FEE_BAL, ssOrderId);
}
```

### 8.3 Gas 地址架构

```
┌──────────────────┐
│   Gas 地址       │
│ (专用gas补充)    │
│                  │
│ 余额: 大量 TRX   │
│ 私钥: KMS加密    │  ──→ 补充30 TRX ──→ ┌─────────────┐
│ 存储: yml配置    │                      │ 用户地址/   │
└──────────────────┘                      │ 发币地址    │
                                          │             │
      用途: 给TRX余额不足的地址           │ TRX < 30 时 │
      补充gas费，使其能发起              │ 触发补充    │
      TRC20转账                          └─────────────┘
```

### 8.4 多地址架构总结

| 地址角色 | 配置方式 | 私钥管理 | 用途 |
|---------|---------|---------|------|
| **用户地址** | DB (TrxWallet) | AES加密存DB | 接收用户充值 |
| **公司地址** (mainCollectAddress) | yml (KMS加密) | KMS + AES解密 | 归集目标地址 |
| **发币地址** (disPendAddress) | DB (TrxWalletConfig) | 外部签名服务 | 提现出金来源 |
| **Gas地址** (gasAddress) | yml (KMS加密) | KMS解密后内存使用 | 补充TRX gas费 |

---

## 9. 智能合约交互层

### 9.1 架构分层

```
                    ┌─────────────────────────┐
                    │     TrxDriver.java      │  ← 业务驱动层
                    │     (339行)             │     知道业务逻辑
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │  WalletTrxBusiness.java  │  ← 链上操作层
                    │     (539行)             │     知道TRON协议
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                   ▼
    ┌─────────────────┐ ┌───────────────┐  ┌──────────────┐
    │   Trc20.java    │ │SmartContract  │  │ TronUtil.java│
    │   (126行)       │ │  .java (92行) │  │  (123行)     │
    │                 │ │               │  │              │
    │ TRC20专用:      │ │ 合约通用:     │  │ gRPC客户端:  │
    │ - balanceOf     │ │ - trigger     │  │ - 签名       │
    │ - transfer      │ │   Constant    │  │ - 广播       │
    │ - decimals      │ │   Contract    │  │ - 地址转换   │
    └─────────────────┘ │ - trigger     │  └──────────────┘
                        │   Contract    │
                        └───────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │   TronUtils.java        │  ← 密码学工具层
                    │     (474行)             │     离线操作
                    │ - 地址生成              │
                    │ - 交易打包              │
                    │ - 离线签名              │
                    │ - Base58Check编解码     │
                    └─────────────────────────┘
```

### 9.2 双通道通信

wallet-server 与 TRON 网络通过**两个通道**通信：

**通道1: HTTP REST API** — 用于交易构建和广播

```
TRON Full Node: https://nile.trongrid.io (测试网) / https://api.trongrid.io (主网)

使用的端点：
  POST /wallet/triggersmartcontract     ← TRC20交易构建
  POST /wallet/broadcasthex            ← 交易广播
  POST /wallet/getaccount              ← 查询TRX余额
  POST /wallet/validateaddress         ← 验证地址格式
  POST /wallet/gettransactionbyid      ← 查询交易状态
```

**通道2: gRPC** — 用于合约只读调用

```
gRPC Client: WalletApi.init() (TronUtil.java)

使用的方法：
  rpcCli.triggerConstantContract()      ← balanceOf等只读调用
  rpcCli.triggerContract()              ← transfer等写操作
  rpcCli.broadcastTransaction()         ← 交易广播（备用通道）
```

### 9.3 TRC20 交易构建详解

**源码**：`WalletTrxBusiness.trc20Transaction()`

```
Step 1: ABI编码
  function_selector = "transfer(address,uint256)"
  parameter = encoderAbi(toAddress, amount * 10^decimals)
    ├─ Address: zero-pad to 32 bytes (去除"41"前缀)
    └─ Uint256: BigInteger → 32 bytes hex

Step 2: HTTP构建交易
  POST /wallet/triggersmartcontract
  body: {
    contract_address: tokenContract (hex, 41开头),
    function_selector: "transfer(address,uint256)",
    parameter: step1的ABI编码,
    owner_address: fromAddress (hex),
    call_value: 0,
    fee_limit: 50000000  // 50 TRX
  }

Step 3: 注入数据标记
  transaction.raw_data.data = hex("trc20")
  // 用途不明，可能是自定义标记

Step 4: 打包交易
  TronUtils.packTransaction(json, false)
  → Protocol.Transaction (Protobuf)
  → TriggerSmartContract 类型

Step 5: 签名
  if (from == disPendAddress):
    sendSign(signUrl, tx, ssOrderId, from)  // 外部签名
  else:
    signTransaction2Byte(tx, privateKey)     // 本地签名

Step 6: 广播
  POST /wallet/broadcasthex
  body: { transaction: hexString(signedTx) }

Step 7: 返回
  TrxWalletResult { txId }
```

### 9.4 TRC20 余额查询

**源码**：`Trc20.balanceOf()`

```java
public static BigDecimal balanceOf(String address, String contract) {
    // 1. ABI编码 balanceOf(address)
    String param = encoderBalanceOfAbi(toHexAddress(address).substring(2));

    // 2. triggerConstantContract (只读调用，不上链)
    String result = SmartContract.triggerConstantContract(
        address, contract, "balanceOf(address)", param);

    // 3. 解码返回值 (Uint256)
    BigDecimal balance = decoderBalanceOf(result);

    // 4. 除以 10^decimals
    Integer decimal = decimals(contract);
    return balance.divide(BigDecimal.TEN.pow(decimal));
}
```

### 9.5 fee_limit 配置

| 场景 | fee_limit | 说明 |
|------|-----------|------|
| TRC20 transfer (HTTP方式) | 50,000,000 sun = 50 TRX | `trc20Transaction()` 中硬编码 |
| TRC20 transfer (gRPC方式) | 10,000,000 sun = 10 TRX | `Trc20.FEE_LIMIT` |
| TRX transfer | N/A | 原生转账无fee_limit |

---

## 10. 安全架构

### 10.1 安全层次总览

```
┌────────────────────────────────────────────────────────────┐
│                     安全架构全景                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Layer 1: 传输安全                                         │
│  ├─ 服务间通信: 内网HTTP (无TLS)                           │
│  └─ TRON节点: HTTPS (TronGrid)                             │
│                                                            │
│  Layer 2: 密钥管理                                         │
│  ├─ 用户私钥: AES加密 → MySQL                             │
│  ├─ AES密码: AWS KMS加密 → application.yml                 │
│  ├─ 公司地址/Gas地址: AWS KMS加密 → application.yml        │
│  └─ 发币地址私钥: 外部HSM/冷钱包（从不出现在应用中）       │
│                                                            │
│  Layer 3: 签名安全                                         │
│  ├─ 归集签名: 本地签名（用户私钥AES解密后内存中使用）      │
│  └─ 提现签名: 外部签名服务（RSA加密通信）                  │
│                                                            │
│  Layer 4: 业务安全                                         │
│  ├─ Redisson分布式锁 (防并发)                              │
│  ├─ Redis防刷 (频率限制)                                   │
│  ├─ 幂等设计 (txId/ssOrderId 去重)                         │
│  └─ JPA乐观锁 @Version (余额变更)                          │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 10.2 AWS KMS 集成

**源码**：`AwsKmsDriver.java`（111行）

```java
// 配置
aws.accessKeyId: REDACTED_AWS_ACCESS_KEY_ID
aws.secretAccessKey: REDACTED_AWS_SECRET_ACCESS_KEY
aws.keyId: mrk-2017a36cde86493d8be54346415708a5
// Region: AP_SOUTHEAST_1 (新加坡)

// 加密
public String encryptData(String noEncryptData) {
    KmsClient client = createKmsClient();
    EncryptRequest req = EncryptRequest.builder()
        .keyId(keyId)
        .plaintext(SdkBytes.fromUtf8String(noEncryptData))
        .build();
    return Base64.getEncoder().encodeToString(
        client.encrypt(req).ciphertextBlob().asByteArray());
}

// 解密
public String decryptData(String afterEncryptionData) {
    KmsClient client = createKmsClient();
    DecryptRequest req = DecryptRequest.builder()
        .keyId(keyId)
        .ciphertextBlob(SdkBytes.fromByteArray(Base64.getDecoder().decode(afterEncryptionData)))
        .build();
    return client.decrypt(req).plaintext().asUtf8String();
}
```

**KMS加密的配置项**（application.yml）：

| 配置项 | 用途 |
|--------|------|
| `wallet.trx.companyAddress` | 归集目标地址（公司主账户） |
| `wallet.trx.mainCollectAddress` | 同上（冗余配置） |
| `wallet.trx.mainCollectPassphrase` | 钱包加密密码（AES密钥的一部分） |
| `wallet.trx.gasAddress` | Gas费补充地址 |
| `wallet.trx.gasPrivateKey` | Gas地址私钥 |

### 10.3 外部签名服务通信

**源码**：`WalletTrxBusiness.sendSign()`

```
签名请求流程：

1. 计算交易哈希
   txHash = Sha256Sm3Hash.hash(tx.getRawData().toByteArray())
   hashId = Base64.encode(txHash.getBytes())

2. 构建请求体
   body = {
     orderNo: ssOrderId,
     outAddressNo: fromAddress,
     chainType: "TRON",
     hashId: hashId (base64)
   }

3. RSA加密
   rsaPublicKey = TrxWalletConfig.findByType("sign")
                  .filter(code == "rsa").value
   encryptedBody = RSAUtil.encrypt(JSON.toJSONString(body), rsaPublicKey)

4. HTTP请求
   POST http://34.92.121.9:80/api/admin/chain/sign
   Header: PLAT_CODE = TrxWalletConfig(type=sign, code=platCode).value
   Body: encryptedBody

5. 解析响应
   response.data.signValue → Base64 decode → 签名字节
   → 附加到 tx.signature → 返回签名后的交易字节
```

### 10.4 AES 加密实现

**源码**：`AES.java` / `WalletTrxBusiness` 内联实现

```
算法: AES/ECB/PKCS5Padding
密钥派生: SHA-1(secret) → 取前16字节
密钥拼接: secret = passphrase + address

⚠️ 安全弱点:
  - ECB模式（不推荐，无IV，相同明文产生相同密文）
  - SHA-1已被认为不安全
  - 密钥派生不使用标准KDF（如PBKDF2/Argon2）
  - RSA 512-bit 密钥长度不足（建议2048+）
```

---

## 11. 定时任务体系

### 11.1 任务清单

| 任务 | 位置 | Cron | 时区 | 功能 |
|------|------|------|------|------|
| **充值检查** | HandleVirtualCurrencyTask | `0 */2 * * * ?` | Asia/Bangkok | 每2分钟扫描未处理充值订单 |
| **USDT归集** | CollectTrxJob | `0 0 8 * * ?` | Asia/Bangkok | 每日8:00批量归集 |
| ~~提现检查~~ | ~~HandleVirtualCurrencyTask~~ | ~~`0 */2 * * * ?`~~ | — | 已注释掉 |

### 11.2 充值定时任务详解

**源码**：`HandleVirtualCurrencyTask.checkVirtualCurrencyTask()`

```java
@Scheduled(cron = "0 */2 * * * ?", zone = "Asia/Bangkok")
public void checkVirtualCurrencyTask() {
    virtualCurrencyBusiness.checkChargeCrypotcurrencyOrder(
        VirtualCurrencyEnum.USDT.getName(),  // "USDT"
        timeoutMinutes                        // 30
    );
}
```

**执行逻辑**：`VirtualCurrencyBusiness.checkChargeCrypotcurrencyOrder()`

```
1. 计算 time0 = now - 30分钟

2. 超时处理 (timeoutOrder):
   - 查找 createTime < time0 且 orderStatus = VIRTUAL_CURRENCY_WAIT(11) 的充值订单
   - 批量更新 orderStatus → ORDER_SYSTEM_CANCLE(99)

3. 未超时订单处理:
   - 查找 createTime ≥ time0 且 orderStatus = VIRTUAL_CURRENCY_WAIT(11) 的充值订单
   - 对每个订单异步调用 asyncVirtualDepositService.deposit(orderId, "USDT", false)
     └─ @Async 异步执行
     └─ Redisson锁防并发: "user-server::deposit_or_charge::ss_order_id::{orderId}"
     └─ checkCrypotcurrencyOrder() → 查余额/归集/确认（见第5章）
```

### 11.3 任务线程池

**源码**：`ScheduleConfig.java`

```java
ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
scheduler.setPoolSize(10);
scheduler.setThreadNamePrefix("game-server-task-");
```

---

## 12. 枚举与常量设计

### 12.1 wallet-server 枚举

#### TRXContractEnum（代币合约）

| 值 | code | contract | name | decimals | type |
|----|------|----------|------|----------|------|
| TRX | 1 | "TRX" | TRX | 6 | TRX |
| USDT | 2 | yml动态读取 | USDT | 6 | TRC20 |
| CXT | 3 | TBfSX66Sp...aE | CXT | 18 | TRC20 |

> USDT 合约地址通过 `WalletUtil.getContractByCoin("USDT")` 从 application.yml 读取（测试网: `TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf`）

#### TrxOrderTypeEnum（订单类型）

| 值 | code | name |
|----|------|------|
| RECHARGE | 1 | 充值 |
| WITHDRAW | 2 | 提现 |

#### CollectTypeEnum（归集类型）

| 值 | code | name |
|----|------|------|
| SINGLE | 1 | 单笔即时归集 |
| COLLECT | 2 | 定时任务归集 |

### 12.2 module-common 枚举（user-server使用）

#### OrderStatusEnum（订单状态）

| 值 | code | name | 用途 |
|----|------|------|------|
| ORDER_WAIT | 1 | 待审核 | 法币充值初始状态 |
| ORDER_SUCCESS | 2 | 已完成 | 终态-成功 |
| ORDER_AUDIT_ING | 3 | 审核中 | - |
| ORDER_AUDIT_NO | 4 | 审核拒绝 | - |
| ORDER_WITHDRAW_ING | 5 | 待出款 | - |
| ORDER_WITHDRAW_LOCK | 6 | 出款中 | - |
| ORDER_CREDENTIALS | 9 | 待上传凭证 | - |
| **VIRTUAL_CURRENCY_WAIT** | **11** | **待处理** | **虚拟币充提初始状态** |
| ORDER_SYSTEM_CANCLE | 99 | 系统取消 | 终态-超时/失败 |

#### BalanceChangeEnum（余额变更类型）

| code | name | 场景 |
|------|------|------|
| 1 | 会员存款(后台) | 后台手动充值 |
| 2 | 下注 | 投注扣款 |
| 3 | 派彩 | 中奖加款 |
| 4 | 会员提款(后台) | 后台手动提现 |
| 5 | 游戏返水 | 返水 |
| **7** | **会员存款(前端)** | **虚拟币充值** |
| **8** | **会员提款(前端)** | **虚拟币提现** |
| **90** | **手续费** | **提现手续费** |
| 91 | 提款失败 | - |
| 96 | 结算回滚 | 减余额 |
| 97 | 二次结算 | 重新派奖 |
| 100 | 取消 | 加余额 |

#### OrderTypeEnum（订单类型）

| code | name |
|------|------|
| 1 | 充值 (CHARGE_ORDER) |
| 2 | 提现 (WITHDRAW_ORDER) |
| 3 | 活动 (ACTIVITY_ORDER) |
| 4 | 返水 (RETURN_AMOUNT_ORDER) |

#### BalanceTypeEnum（收支方向）

| code | name |
|------|------|
| 1 | 收入 (INCOME) |
| 2 | 支出 (EXPENSES) |

### 12.3 常量清单

**WalletServerConstants（user-server → wallet-server 接口路径）**：

| 常量 | 值 | 用途 |
|------|----|------|
| WALLET_COLLECT_TRC_20 | `/wallet/collectTrc20` | 归集/充值转账 |
| WALLET_GET_BALANCE | `/wallet/getBalance` | 查链上余额 |
| WALLET_GET_TRANSACTION_INFO_BY_ID | `/wallet/getTransactionInfoById` | 查交易状态 |
| WALLET_WITHDRAWTRACTION | `/wallet/withdrawTraction` | 提现 |

**CommonConstant（wallet-server内部）**：

| 常量 | 值 | 用途 |
|------|----|------|
| SIGN | `"sign"` | TrxWalletConfig.type |
| RSA | `"rsa"` | TrxWalletConfig.code for RSA公钥 |
| PLATCODE | `"platCode"` | TrxWalletConfig.code for 商户标识 |
| PATH_SIGN | `/api/admin/chain/sign` | 签名服务端点 |
| DISPENDADDRESS | `"disPendAddress"` | TrxWalletConfig.code for 发币地址 |

---

## 13. 跨服务通信架构

### 13.1 通信方式

**全部使用 HTTP REST**（无 RPC/MQ）

```
user-server → wallet-server:  HttpClient4Util.doPost()
user-server → web-server:     HttpClient4Util.doPost()
user-server → backend-server: HttpClient4Util.doPost()
wallet-server → user-server:  RestTemplate.postForObject()
wallet-server → TRON:         HttpURLConnection / gRPC
wallet-server → 签名服务:      hutool HttpUtil.post()
```

### 13.2 服务间调用矩阵

| 调用方 | 被调方 | 接口 | 用途 |
|--------|--------|------|------|
| user-server | wallet-server | POST /wallet/create | 创建钱包地址 |
| user-server | wallet-server | POST /wallet/getBalance | 查询链上余额 |
| user-server | wallet-server | POST /wallet/collectTrc20 | 触发归集 |
| user-server | wallet-server | POST /wallet/withdrawTraction | 执行提现 |
| user-server | wallet-server | POST /wallet/getTransactionInfoById | 查交易状态 |
| wallet-server | user-server | POST /user/updateWalletAddressById | 同步钱包地址 |
| wallet-server | 签名服务 | POST /api/admin/chain/sign | 提现签名 |
| user-server | web-server | POST /通知接口 | 通知TG Bot余额变更 |
| user-server | backend-server | POST /统计接口 | 发送统计数据 |

### 13.3 user-server 服务地址配置

```yaml
# user-server application.yml
url:
  walletServer: http://192.168.30.232:9600   # wallet-server
  webServer: http://...                       # web-server
  backend-server-domain: http://...           # backend-server
```

---

## 14. Bug与设计缺陷分析

### 14.1 严重问题

#### Bug-1: 提现五步操作无事务保护

**位置**：`AsyncVirtualWithdrawService.withdrawUSDT()`

```java
// Step 1: 解冻 → Step 2: 扣余额 → Step 3: 扣手续费 → Step 4: 更新订单 → Step 5: 通知
// 这5步没有 @Transactional 包裹
// 如果 Step 2 成功但 Step 4 失败 → 余额已扣但订单仍为"待处理"
```

**影响**：资金安全风险。余额扣除但订单未完成，用户投诉时难以定位。

#### Bug-2: TrxWalletOrderRepository 硬编码 coin_name

**位置**：`TrxWalletOrderRepository.findBySsOrderId()`

```java
@Query(value = "select * from trx_wallet_order a where a.ss_order_id = ?1 and coin_name = 'USDT'",
       nativeQuery = true)
TrxWalletOrderEntity findBySsOrderId(String ssOrderId);
```

**影响**：如果未来需要支持 TRX 或 CXT 提现，此查询会找不到记录。

#### Bug-3: RSA 512-bit 密钥长度不安全

**位置**：`RSAUtil.java`

```java
KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
keyPairGenerator.initialize(512);  // 512-bit 已被认为不安全
```

**影响**：512-bit RSA 可在数小时内被暴力破解。签名服务通信可被伪造。

#### Bug-4: AES/ECB 模式不安全

**位置**：`WalletTrxBusiness.encrypt()` / `AES.java`

```java
Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
// ECB模式: 相同明文 → 相同密文，存在模式泄露风险
```

### 14.2 设计缺陷

#### 缺陷-1: 充值确认依赖定时任务轮询

当前设计是每2分钟轮询一次检查充值订单状态。更好的方案是监听 TRON 链上事件（Event/Websocket），或使用 TronGrid 的 webhook 通知。

**影响**：最长可能延迟2分钟确认充值，用户体验不佳。

#### 缺陷-2: 归集与充值确认耦合

归集操作（资金从用户地址转到公司地址）是在充值确认流程中同步执行的。如果归集交易失败（gas不足、网络拥堵），充值确认也会失败。

**更好的设计**：归集与充值确认解耦。充值确认只需要确认"用户地址上有足够余额"，归集可以异步批量执行。

#### 缺陷-3: 无链上监听机制

没有监听 TRON 链上的 Transfer 事件。用户转错金额、转多次等场景无法自动处理。全靠定时任务轮询余额。

#### 缺陷-4: 提现定时任务被注释

**位置**：`HandleVirtualCurrencyTask.java`

提现的定时任务 `withdrawVirtualCurrencyTask()` 已被注释。但提现确认逻辑（`VirtualCurrencyWithdrawBusiness.checkVirtualWithdrawTask()`）仍然存在。这意味着提现成功确认可能依赖其他未知的触发机制，或者就是没有自动确认。

#### 缺陷-5: Gas 费补充缺乏限制

从 gasAddress 补充 TRX 时没有频率限制。恶意用户可以反复触发充值→归集流程，每次消耗 gasAddress 30 TRX，耗尽 gas 资金。

### 14.3 代码质量问题

| 问题 | 位置 | 说明 |
|------|------|------|
| 异常吞没 | 多处 `catch (Exception e) { e.printStackTrace(); }` | 异常只打印堆栈，无业务处理 |
| 锁释放不安全 | VirtualCurrencyController | `fairLock.unlock()` 在try块内，异常时在catch中也unlock，可能双重解锁 |
| 魔法数字 | `FEE_BAL = 30`, `fee_limit = 50000000` | 硬编码，无配置化 |
| 安全配置全放行 | WebSecurityConfig | `anyRequest().permitAll()`, `web.ignoring("**")` |
| 密码硬编码 | application.yml | 数据库密码、Redis密码、RabbitMQ密码明文 |
| 命名拼写错误 | `contrloller` (应为 controller) | user-server 包名拼写错误 |

---

## 15. 对 ser-wallet 的借鉴价值

### 15.1 可直接借鉴的设计

#### ✅ 多地址架构

**参考设计**：用户地址 / 公司地址 / 发币地址 / Gas地址 四种角色分离

**借鉴方式**：我们的 ser-wallet 可以采用类似的地址角色分离：
- 用户充值地址（每用户一个，或按币种分配）
- 归集目标地址（公司热钱包）
- 提现发币地址（与归集地址分离，更安全）
- Gas/手续费地址（专门用于补充链上手续费）

#### ✅ 归集机制（单笔即时 + 批量定时）

**参考设计**：
- 充值确认时即时归集（单笔）
- 每日定时批量归集（清扫所有用户地址余额）

**借鉴方式**：双重归集保证资金不分散在用户地址上。即时归集保证充值快速到账，定时归集保证遗漏的资金也能回收。

#### ✅ 外部签名服务（提现）vs 本地签名（归集）

**参考设计**：提现资金外流，使用外部HSM签名；归集资金内部流转，本地签名。

**借鉴方式**：高风险操作（提现）与低风险操作（归集）使用不同的签名策略，平衡安全性与效率。

#### ✅ 两阶段充值确认

**参考设计**：
- 第一阶段：触发归集，获取 txId
- 第二阶段：链上确认，执行上分

**借鉴方式**：区块链交易不是即时确认的，必须设计异步确认机制。分阶段处理是合理的。

#### ✅ Gas 费自动补充

**参考设计**：TRC20 转账需要 TRX 作为 gas，系统自动从 gasAddress 补充。

**借鉴方式**：任何 EVM/TRON 类链上的代币转账都需要原生币作 gas。必须设计自动补充机制。

### 15.2 需要改进的方面

| 参考工程的问题 | 我们应如何做 |
|---------------|-------------|
| 无事务保护 | 关键操作（解冻+扣款+订单更新）必须在同一事务中 |
| 轮询确认 | 考虑链上事件监听（WebSocket/webhook），减少延迟 |
| ECB加密模式 | 使用 AES/GCM 或 AES/CBC + HMAC |
| RSA 512-bit | 最低 RSA 2048-bit，推荐 Ed25519 |
| 硬编码参数 | fee_limit、gas阈值等配置化 |
| 异常吞没 | 统一错误处理，关键操作必须有告警 |
| 无链上监听 | 设计 event listener，主动发现充值到账 |
| Gas补充无限制 | 添加频率限制和每日上限 |

### 15.3 架构对照

| 维度 | tg-services | 我们的 ser-wallet |
|------|------------|-------------------|
| 语言 | Java/Spring Boot | Go/CloudWeGo |
| ORM | JPA/Hibernate | GORM |
| 服务通信 | HTTP REST | Kitex RPC |
| 消息队列 | RabbitMQ(已弃用→HTTP) | Kafka(计划中) |
| 缓存 | Redis + Redisson | Redis |
| 密钥管理 | AWS KMS + AES | 待定（可参考） |
| 链上交互 | web3j + TRON SDK | 待定（Go TRON SDK） |
| 数据库 | MySQL (JPA自动建表) | MySQL (GORM迁移) |

### 15.4 核心借鉴清单

```
优先级 P0（必须借鉴）：
  □ 多地址架构设计（用户地址/公司地址/发币地址/Gas地址）
  □ 归集机制（即时+定时双重保障）
  □ 两阶段充值确认流程
  □ 充值/提现的完整状态机
  □ Gas费自动补充策略

优先级 P1（建议借鉴）：
  □ 链上交互层抽象（业务驱动层/链上操作层/密码工具层分离）
  □ 外部签名服务架构（提现与归集签名分离）
  □ 私钥加密存储方案（改进加密算法后借鉴整体思路）
  □ 订单表设计（SsOrder的字段设计可参考）

优先级 P2（可选参考）：
  □ 动态配置表设计（TrxWalletConfig）
  □ 防刷机制（Redis频率限制）
  □ 幂等设计（txId/ssOrderId 去重）
```

---

## 附录A：完整 API 端点清单

### wallet-server（port 9600）

| Method | Path | 请求体 | 响应体 | 功能 |
|--------|------|--------|--------|------|
| POST | `/wallet/create` | TrxUserWalletRequest {userId, userName} | WalletAddressVO {address, userId, userName} | 创建钱包地址 |
| POST | `/wallet/getBalance` | UserBalanceRequest {userId, currency} | BigDecimal | 查询链上余额 |
| POST | `/wallet/collectTrc20` | TrcCollectRequest {userId, ssOrderId, coinName, amount, postType} | TrxWalletResult {txId} | 归集/充值转账 |
| POST | `/wallet/withdrawTraction` | TrxCreateTransactionRequest {ownerAddress?, ssOrderId, userId, toAddress, coinName, amount, postType} | TrxWalletResult {txId} | 提现 |
| POST | `/wallet/getTransactionInfoById` | orderId (String param) | TransactionVO {transResult, txId} | 查询交易状态 |
| POST | `/wallet/getTrxWalletAccount` | TrxWalletAccountRequest {各地址+私钥} | TrxWalletAccountVO {KMS加密后} | 工具：KMS加密 |

### user-server 虚拟货币相关（port 9300）

| Method | Path | 请求体 | 功能 |
|--------|------|--------|------|
| POST | `/virtual/charge` | SsOrderVirtualCurrencyChargeVO {orderId} | 虚拟货币充值确认 |
| POST | `/virtual/withdraw` | SsOrderVirtualCurrencyWithdrawVO {orderId} | 虚拟货币提现 |

---

## 附录B：完整 Redis Key 清单

### user-server Redis Keys

| Key 模式 | TTL | 用途 |
|----------|-----|------|
| `prevent::time::virtualCurrencyCharge::{orderId}` | 3s | 充值接口防刷 |
| `prevent::time::virtualCurrencyWithdraw::{orderId}` | 10s | 提现接口防刷 |
| `user-server::deposit_or_charge::ss_order_id::{orderId}` | Redisson锁 | 充值任务幂等锁 |
| `user-server::withdraw_task::ss_order_id::{orderId}` | Redisson锁 | 提现任务幂等锁 |
| `userServer::virtualCurrency::withdraw_{orderId}` | Redisson锁 | 提现接口并发锁 |
| `userServer::assets::{userId}` | Redisson公平锁 | 资产操作互斥锁 |
| `userServer::balance::{userId}` | Redisson锁 | 余额操作锁 |
| `userServer::freezeAmount::{userId}` | Redisson锁 | 冻结操作锁 |
| `order::withdraw_{orderId}` | 30天 | 提现上链幂等标记 |

### Redisson 锁参数

| 场景 | waitTime | leaseTime | 类型 |
|------|----------|-----------|------|
| 充值确认 | 0s | 10s | 公平锁 |
| 提现执行 | 10s | 5s | 公平锁 |
| 资产操作 | 10s | 5s | 公平锁 |
| 提现任务 | 0s | 5s | 公平锁 |

---

## 附录C：关键文件索引

### wallet-server 核心文件

| 文件 | 行数 | 路径 |
|------|------|------|
| WalletTrxBusiness.java | 539 | `wallet-server/.../business/WalletTrxBusiness.java` |
| TrxDriver.java | 339 | `wallet-server/.../driver/TrxDriver.java` |
| TronUtils.java | 474 | `wallet-server/.../util/TronUtils.java` |
| WalletController.java | 144 | `wallet-server/.../controller/WalletController.java` |
| Trc20.java | 126 | `wallet-server/.../client/trx/Trc20.java` |
| TRXContractEnum.java | 124 | `wallet-server/.../model/enums/TRXContractEnum.java` |
| TronUtil.java | 123 | `wallet-server/.../client/trx/TronUtil.java` |
| AwsKmsDriver.java | 111 | `wallet-server/.../driver/AwsKmsDriver.java` |
| RSAUtil.java | 111 | `wallet-server/.../util/RSAUtil.java` |
| SmartContract.java | 92 | `wallet-server/.../client/trx/SmartContract.java` |
| TrxWalletOrderEntity.java | 66 | `wallet-server/.../model/TrxWalletOrderEntity.java` |
| TrxWalletCollectEntity.java | 49 | `wallet-server/.../model/TrxWalletCollectEntity.java` |
| TrxWalletEntity.java | 44 | `wallet-server/.../model/TrxWalletEntity.java` |
| CollectTrxJob.java | 41 | `wallet-server/.../task/CollectTrxJob.java` |
| TrxWalletConfigEntity.java | 40 | `wallet-server/.../model/TrxWalletConfigEntity.java` |
| application.yml | 94 | `wallet-server/src/main/resources/application.yml` |

### user-server 钱包相关文件

| 文件 | 行数 | 路径 |
|------|------|------|
| UserAssetsBusiness.java | 1294 | `user-server/.../business/UserAssetsBusiness.java` |
| AsyncVirtualDepositService.java | 455 | `user-server/.../service/AsyncVirtualDepositService.java` |
| AsyncVirtualWithdrawService.java | 362 | `user-server/.../service/AsyncVirtualWithdrawService.java` |
| VirtualCurrencyWithdrawBusiness.java | 337 | `user-server/.../business/VirtualCurrencyWithdrawBusiness.java` |
| VirtualCurrencyBusiness.java | 249 | `user-server/.../business/VirtualCurrencyBusiness.java` |
| VirtualCurrencyController.java | 241 | `user-server/.../contrloller/VirtualCurrencyController.java` |
| WalletBusiness.java | 235 | `user-server/.../business/WalletBusiness.java` |
| SsOrder.java | 221 | `user-server/.../model/SsOrder.java` |
| Assets.java | 119 | `user-server/.../model/Assets.java` |
| HandleVirtualCurrencyTask.java | 67 | `user-server/.../task/HandleVirtualCurrencyTask.java` |

### module-common 枚举文件

| 文件 | 路径 |
|------|------|
| OrderStatusEnum.java | `module-common/.../enums/order/OrderStatusEnum.java` |
| OrderTypeEnum.java | `module-common/.../enums/order/OrderTypeEnum.java` |
| VirtualCurrencyEnum.java | `module-common/.../enums/order/VirtualCurrencyEnum.java` |
| BalanceChangeEnum.java | `module-common/.../enums/BalanceChangeEnum.java` |
| BalanceTypeEnum.java | `module-common/.../enums/BalanceTypeEnum.java` |
