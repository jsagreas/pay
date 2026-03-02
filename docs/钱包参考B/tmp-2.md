# TG-Services 钱包模块完整深度解剖（区块链虚拟币钱包）

> **分析对象**：`/Users/mac/gitlab/z-tmp/tg-services/wallet-server` + 联动模块 `user-server`
> **工程性质**：Java Spring Boot 微服务 — Telegram即时通讯 + 真人视讯平台
> **钱包定位**：TRON链上虚拟币钱包，专注 TRC20/USDT 充值、提现、归集
> **核心价值**：区块链充值/提现/归集完整链路实现，Gas费管理机制，密钥安全方案
> **分析方法**：每个结论附源码文件:行号佐证，每章配"借鉴价值"+"风险规避"
> **源码基准**：wallet-server(~40个Java文件) + user-server(虚拟币相关~15个文件) + module-common(共享枚举/VO)

---

## 目录

- [第1章 工程全景与技术栈](#第1章-工程全景与技术栈)
- [第2章 系统架构与服务拓扑](#第2章-系统架构与服务拓扑)
- [第3章 数据模型与表结构](#第3章-数据模型与表结构)
- [第4章 接口设计全景图](#第4章-接口设计全景图)
- [第5章 钱包创建与密钥管理](#第5章-钱包创建与密钥管理)
- [第6章 区块链充值链路完整解剖](#第6章-区块链充值链路完整解剖)
- [第7章 区块链提现链路完整解剖](#第7章-区块链提现链路完整解剖)
- [第8章 归集机制深度分析](#第8章-归集机制深度分析)
- [第9章 Gas费管理与能量机制](#第9章-gas费管理与能量机制)
- [第10章 余额引擎与账变机制](#第10章-余额引擎与账变机制)
- [第11章 并发控制与一致性机制](#第11章-并发控制与一致性机制)
- [第12章 安全设计分析](#第12章-安全设计分析)
- [第13章 缺陷清单与风险评估](#第13章-缺陷清单与风险评估)
- [第14章 优秀设计借鉴清单](#第14章-优秀设计借鉴清单)
- [第15章 与 ser-wallet 的对比映射](#第15章-与-ser-wallet-的对比映射)
- [第16章 总结：取舍决策表](#第16章-总结取舍决策表)
- [附录A 完整文件清单与代码行数](#附录a-完整文件清单与代码行数)
- [附录B 关键代码片段索引](#附录b-关键代码片段索引)
- [附录C TRON链基础知识速查](#附录c-tron链基础知识速查)

---

## 第1章 工程全景与技术栈

### 1.1 工程结构

```
tg-services/                              # Telegram即时通讯 + 真人视讯平台（18个模块）
├── wallet-server/                        # ★ 区块链钱包服务（端口9600）
│   ├── business/WalletTrxBusiness.java   # 核心TRON操作（538行）
│   ├── driver/TrxDriver.java            # 高层钱包驱动/门面（338行）
│   ├── driver/AwsKmsDriver.java         # AWS KMS加解密驱动（110行）
│   ├── controller/WalletController.java  # REST控制器（143行，6个业务端点）
│   ├── client/trx/                       # TRON链客户端层
│   │   ├── TronUtil.java                # gRPC客户端包装（122行）
│   │   ├── SmartContract.java           # 智能合约调用（91行）
│   │   └── Trc20.java                   # TRC20代币接口（125行）
│   ├── model/                           # 4个实体 + 3个枚举 + 5个请求VO + 4个响应VO
│   ├── repository/                      # 4个JPA Repository
│   ├── service/                         # 3个Service
│   ├── task/CollectTrxJob.java          # 定时归集（每天8:00曼谷时间）
│   └── util/                            # 加密工具（AES/RSA/Base58/SHA256/TronUtils）
│
├── user-server/                          # 用户服务（端口9300）— wallet-server的主调方
│   ├── business/WalletBusiness.java      # 钱包HTTP调用封装（234行）
│   ├── business/VirtualCurrencyBusiness.java  # 虚拟币充值业务（248行）
│   ├── business/VirtualCurrencyWithdrawBusiness.java  # 虚拟币提现业务（336行）
│   ├── service/AsyncVirtualDepositService.java   # 异步充值处理（454行）
│   ├── service/AsyncVirtualWithdrawService.java  # 异步提现处理（361行）
│   ├── task/HandleVirtualCurrencyTask.java       # 定时任务（每2分钟）
│   ├── model/SsOrder.java               # 订单实体（220行，含区块链字段）
│   └── model/Assets.java                # 资产实体（118行，含乐观锁version）
│
├── module-common/                        # 共享库
│   ├── enums/BalanceChangeEnum.java      # 50+种账变类型
│   ├── enums/BalanceTypeEnum.java        # 收入INCOME(1) / 支出EXPENSES(2)
│   ├── enums/order/OrderStatusEnum.java  # 订单状态枚举
│   └── enums/order/OrderTypeEnum.java    # 充值CHARGE(1) / 提现WITHDRAW(2)
│
├── telegram-bot-api/                     # Telegram机器人（端口9000）— UI入口
├── casino/casino-web/                    # Casino前端（端口9401）
├── casino/casino-backend/                # Casino后台（端口9402）
├── game-server/                          # 游戏服务（端口9500）
└── wallet-third-server/                  # 第三方钱包（端口9601）
```

### 1.2 技术栈对比

| 维度 | TG工程 wallet-server | 我们的 ser-wallet |
|------|---------------------|------------------|
| 语言 | Java 8+ | Go 1.21+ |
| Web框架 | Spring Boot + Spring Security | CloudWeGo Hertz (HTTP) |
| RPC框架 | HTTP REST (Feign风格，但实际用HttpClient4Util) | CloudWeGo Kitex (Thrift IDL) |
| ORM | Spring Data JPA / Hibernate | GORM Gen |
| 数据库 | MySQL (Hibernate DDL auto-update) | TiDB (MySQL兼容) |
| 缓存 | Redis (Lettuce) + Redisson分布式锁 | Redis (go-redis) |
| 消息队列 | RabbitMQ (配置了但部分改HTTP调用) | 无（同步RPC） |
| 密钥管理 | AWS KMS (AP_SOUTHEAST_1) | 待定 |
| 私钥加密 | AES/ECB/PKCS5Padding (SHA-1派生key) | 待定 |
| 签名服务 | 外部独立签名系统 (RSA-512加密请求) | 待定 |
| 区块链交互 | TRON gRPC + HTTP Full Node API | 待定 |
| 定时任务 | @Scheduled (Spring) | 待定 |
| API文档 | Swagger2 | 待定 |

### 1.3 核心设计哲学

TG工程的钱包设计遵循**"热钱包+归集+冷钱包"三层架构**：

```
用户热钱包（每人一个TRON地址）
    │ 充值：用户自行转入
    │ 归集：定时/手动 → 汇总到公司地址
    ▼
公司主地址（companyAddress = mainCollectAddress）
    │ 余额集中管理
    │ 提现：从出款地址(dispensingAddress)转出
    ▼
出款地址（mainDispensingAddress）
    │ 提现时使用外部签名系统加签
    ▼
用户提现目标地址

Gas费地址（gasAddress）
    │ 为TRC20归集/提现提供TRX手续费
    ▼
各用户热钱包或出款地址
```

**关键设计决策**：
1. **每用户独立地址**：注册时生成独立TRON热钱包地址，便于充值追踪
2. **充值 = 链上转入 + 定时检测**：不是传统支付回调，而是轮询链上余额
3. **提现 = 中心化出款**：统一从出款地址发起，外部签名系统加签
4. **归集 = 资金汇总**：每天定时把各用户钱包余额汇总到公司主地址

---

## 第2章 系统架构与服务拓扑

### 2.1 服务调用关系图

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ telegram-bot-api │     │   casino-web     │     │ casino-backend   │
│    (9000)        │     │    (9401)        │     │    (9402)        │
│ UI入口：充值/    │     │ 充值/提现/       │     │ 订单管理/统计    │
│ 提现/查余额      │     │ 查余额           │     │                  │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │ HTTP                   │ HTTP                   │ HTTP
         ▼                        ▼                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      user-server (9300)                             │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │ VirtualCurrency  │  │ VirtualCurrency  │  │ UserAssets        │  │
│  │ Business(充值)   │  │ WithdrawBusiness │  │ Business(余额)    │  │
│  └───────┬─────────┘  └───────┬──────────┘  └──────────────────┘  │
│          │                     │                                    │
│  ┌───────▼─────────────────────▼──────────┐                        │
│  │         WalletBusiness                  │                        │
│  │  HTTP调用wallet-server的封装层          │                        │
│  │  - getBalance()                         │                        │
│  │  - getTransferResult()                  │                        │
│  │  - withdrawTransaction()                │                        │
│  │  - getTransactionInfoById()             │                        │
│  └───────────────────┬────────────────────┘                        │
│                      │ HTTP POST (JSON)                            │
└──────────────────────┼─────────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     wallet-server (9600)                             │
│  ┌──────────────────┐                                               │
│  │ WalletController  │ ← 6个REST端点                                │
│  └───────┬──────────┘                                               │
│          ▼                                                          │
│  ┌──────────────┐     ┌──────────────────┐                          │
│  │  TrxDriver    │────▶│ WalletTrxBusiness │                        │
│  │  (门面层)     │     │ (核心逻辑层)      │                        │
│  └──────┬───────┘     └───────┬──────────┘                          │
│         │                      │                                     │
│  ┌──────▼──────────────────────▼──────────────┐                     │
│  │           TRON 区块链交互层                   │                   │
│  │  ┌────────────┐  ┌──────────────┐  ┌──────┐│                    │
│  │  │ TronUtil   │  │SmartContract │  │Trc20 ││                    │
│  │  │(gRPC客户端)│  │(合约调用)    │  │(代币)││                    │
│  │  └──────┬─────┘  └──────┬───────┘  └──┬───┘│                    │
│  └─────────┼───────────────┼──────────────┼────┘                    │
│            ▼               ▼              ▼                          │
│     TRON Full Node (gRPC: grpc.nile.trongrid.io:50051)              │
│     TRON Full Node (HTTP: https://nile.trongrid.io)                 │
│                                                                      │
│  ┌────────────────┐     ┌──────────────────┐                        │
│  │  AwsKmsDriver  │     │ 外部签名系统      │                       │
│  │  (密钥加解密)  │     │ (http://34.92.121.9:80)│                  │
│  └────────────────┘     └──────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 数据库分布

| 服务 | 数据库 | 关键表 |
|------|--------|--------|
| wallet-server | cloud_wallet | trx_wallet, trx_wallet_order, trx_wallet_collect, trx_wallet_config |
| user-server | cloud_user | ss_order, assets, balance_change, user |

**关键观察**：wallet-server和user-server使用**独立数据库**，跨库数据一致性完全依靠HTTP调用+定时任务补偿，没有分布式事务。

### 2.3 Redis使用分布

| 服务 | Redis DB | 用途 |
|------|----------|------|
| wallet-server | db=11 | 暂未发现实际业务使用（配了但没用到） |
| user-server | db=0 | Redisson分布式锁 + 防重复提现Key + 各种业务缓存 |

---

## 第3章 数据模型与表结构

### 3.1 wallet-server 表结构（4张表）

#### 3.1.1 trx_wallet — 用户热钱包表

```sql
CREATE TABLE trx_wallet (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,    -- 自增主键
    user_id     VARCHAR(10) COMMENT '会员账号',        -- ★ 用户标识（varchar而非bigint！）
    user_name   VARCHAR(30) COMMENT '会员姓名',
    en_private_key VARCHAR(250) COMMENT '加密后的私钥', -- AES(ECB)加密存储
    address     VARCHAR(100) COMMENT '钱包地址',        -- TRON地址（T开头Base58）
    created_time DATETIME,
    updated_time DATETIME,
    created_by  VARCHAR(255),
    updated_by  VARCHAR(255),
    INDEX idx_user_id (user_id)
);
```

**源码佐证**：`TrxWalletEntity.java:23` — `@Table(name = "TrxWallet", indexes = {@Index(columnList = "userId")})`

**关键字段分析**：
- `user_id` 用 varchar(10) 而非 bigint — 与user-server的userId(bigint)类型不匹配，强制类型转换
- `en_private_key` 存加密后的私钥 — AES/ECB/PKCS5Padding，密钥=SHA1(passphrase+address)前16字节
- 每个用户**只有一条记录** — 通过 `findAddressByUserId` 查重保证幂等

#### 3.1.2 trx_wallet_order — 链上交易记录表

```sql
CREATE TABLE trx_wallet_order (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id      VARCHAR(10) COMMENT '会员账号',
    tx_id        VARCHAR(200) COMMENT '交易id',           -- TRON链上交易哈希
    ss_order_id  VARCHAR(200) COMMENT '充值/提现记录id',    -- 关联user-server的SsOrder.id
    amount       DECIMAL(16,2) COMMENT '金额',
    from_address VARCHAR(250) COMMENT '发送地址',
    to_address   VARCHAR(250) COMMENT '接收地址',
    coin_name    VARCHAR(10) COMMENT '币种类型',           -- TRX / USDT / CXT
    order_type   TINYINT(2) COMMENT '收付方向',            -- 1=RECHARGE / 2=WITHDRAW
    post_type    TINYINT(2) COMMENT '1:接口方式,2:定时任务方式',
    created_time DATETIME,
    updated_time DATETIME,
    INDEX idx_user_id (user_id),
    INDEX idx_order_type (order_type),
    INDEX idx_tx_id (tx_id)
);
```

**源码佐证**：`TrxWalletOrderEntity.java:25`

**关键设计**：
- `ss_order_id` 关联业务订单 — 跨库关联，无外键约束
- `post_type` 区分调用来源 — 1=API手动触发，2=定时任务自动触发
- `findBySsOrderId` 查询时硬编码 `coin_name = 'USDT'` — `TrxWalletOrderRepository.java:19-21`

#### 3.1.3 trx_wallet_collect — 归集记录表

```sql
CREATE TABLE trx_wallet_collect (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(10) COMMENT '会员账号',
    tx_id           VARCHAR(200) COMMENT '交易id',
    collect_address VARCHAR(250) COMMENT '汇款账号',      -- 归集来源地址
    amount          DECIMAL(16,2) COMMENT '汇款金额',
    collect_type    TINYINT(2) COMMENT '归集类型',         -- 1=SINGLE(手动) / 2=COLLECT(定时)
    created_time    DATETIME,
    updated_time    DATETIME,
    INDEX idx_user_id (userId),
    INDEX idx_collect_type (collectType),
    INDEX idx_tx_id (txId)
);
```

**源码佐证**：`TrxWalletCollectEntity.java:24`

**缺陷**：索引列名使用驼峰 `userId` 而非蛇形 `user_id`，与JPA默认命名策略可能冲突。

#### 3.1.4 trx_wallet_config — 钱包动态配置表

```sql
CREATE TABLE trx_wallet_config (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    type  VARCHAR(30) COMMENT '类型',         -- 如 "sign"
    code  VARCHAR(50) COMMENT '标识code',      -- 如 "rsa", "platCode", "disPendAddress"
    value VARCHAR(1000) COMMENT '标识value',   -- RSA公钥/商户码/出款地址
    created_time DATETIME,
    updated_time DATETIME
);
```

**源码佐证**：`TrxWalletConfigEntity.java:28-38`

**实际用途**：存储外部签名系统的配置——RSA公钥、商户编码(platCode)、出款地址(disPendAddress)。

**查询方式**：`trxWalletConfigRepository.findByType("sign")` 取回所有签名相关配置，然后在内存中用Stream过滤 — `TrxDriver.java:285-287`

### 3.2 user-server 关键表结构

#### 3.2.1 ss_order — 业务订单表（核心枢纽）

```sql
CREATE TABLE ss_order (
    id               BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_num        VARCHAR(32) COMMENT '订单编号',
    user_id          BIGINT(20) COMMENT '会员ID',
    tg_user_id       VARCHAR(64) COMMENT 'TG用户ID',
    user_type        TINYINT(2) COMMENT '用户类型 1正式 2测试 3机器人',
    amount           DECIMAL(16,2) COMMENT '总金额',
    real_amount      DECIMAL(16,2) COMMENT '实际到账金额',
    order_type       TINYINT(2) COMMENT '订单类型 1充值 2提现',
    order_status     TINYINT(2) DEFAULT 1 COMMENT '1等待 2成功 3失败 4取消',
    virtual_currency VARCHAR(20) COMMENT '虚拟币种',        -- "USDT"
    fee_amount       DECIMAL(16,2) DEFAULT 0 COMMENT '手续费',
    payee_address    VARCHAR(200) COMMENT '收款地址',        -- 提现目标地址
    tx_id            VARCHAR(200) COMMENT '交易记录Id',      -- TRON链交易哈希
    flow_multiple    DECIMAL(16,2) DEFAULT 1 COMMENT '流水倍数',
    complete_time    DATETIME COMMENT '出入款时间',
    -- ... 代理层级字段 × 5层，渠道字段 × 5层，审核字段
    INDEX idx_user_id (userId),
    INDEX idx_order_num (orderNum),
    INDEX idx_order_type (orderType),
    INDEX idx_order_status (orderStatus)
);
```

**源码佐证**：`SsOrder.java:22-219`

**关键设计**：
- `amount` vs `real_amount` — amount是用户声称金额，real_amount是实际到账。充值时real_amount可能>=amount（用户多转了）
- `fee_amount` — 手续费，提现时从amount中扣除后才发起链上转账
- `tx_id` — 非创建时写入，而是**异步回填**（先创建订单→调用wallet-server→拿到txId→update回来）
- `flow_multiple` — 流水倍数，充值成功后 playMoney = realAmount × flowMultiple

#### 3.2.2 assets — 用户资产表

```sql
CREATE TABLE assets (
    id             BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id        BIGINT(20) UNIQUE COMMENT '会员ID',
    balance        DECIMAL(16,2) DEFAULT 0 COMMENT '余额',
    balance_nn     DECIMAL(16,2) DEFAULT 0 COMMENT '牛牛余额',
    balance_bacc   DECIMAL(16,2) DEFAULT 0 COMMENT '百家乐余额',
    freeze_amount  DECIMAL(16,2) DEFAULT 0 COMMENT '冻结余额',
    play_money     DECIMAL(16,2) DEFAULT 0 COMMENT '打码量',
    credit_amount  DECIMAL(16,2) DEFAULT 0 COMMENT '信用额度',
    version        INT DEFAULT 1,                              -- ★ 乐观锁
    user_type      TINYINT(2) DEFAULT 1 COMMENT '1正式 2测试 3机器人',
    -- ... 代理层级字段
);
```

**源码佐证**：`Assets.java:49-51` — `@Version private Integer version = 1;`

**关键设计**：
- `version` 字段实现 JPA @Version 乐观锁 — 更新时自动检查版本号
- `freeze_amount` — 提现时先冻结，成功后解冻+扣款，失败后仅解冻
- **单行余额模型** — 每用户一行，所有余额类型在同一行。不同于我们的多币种多行设计

### 3.3 ER关系图

```
wallet-server (cloud_wallet DB)            user-server (cloud_user DB)
┌─────────────────┐                        ┌─────────────────┐
│  trx_wallet     │                        │  ss_order       │
│  ─────────────  │ userId(varchar10)       │  ─────────────  │
│  userId    ─────┼───────────────────────▶ │  userId(bigint) │ ← 类型不匹配！
│  address        │                        │  txId           │
│  enPrivateKey   │    ssOrderId           │  amount         │
└─────────────────┘    ┌───────────────────│  realAmount     │
┌─────────────────┐    │                   │  feeAmount      │
│ trx_wallet_order│    │                   │  virtualCurrency│
│  ─────────────  │    │                   │  payeeAddress   │
│  ssOrderId ─────┼────┘                   │  orderStatus    │
│  txId           │                        └────────┬────────┘
│  userId         │                                 │ userId
│  amount         │                        ┌────────▼────────┐
│  orderType      │                        │  assets         │
│  coinName       │                        │  ─────────────  │
└─────────────────┘                        │  userId(unique) │
┌─────────────────┐                        │  balance        │
│trx_wallet_collect│                       │  freezeAmount   │
│  ─────────────  │                        │  playMoney      │
│  userId         │                        │  version (乐观锁)│
│  txId           │                        └─────────────────┘
│  collectAddress │
│  collectType    │
└─────────────────┘
┌─────────────────┐
│trx_wallet_config│
│  ─────────────  │
│  type = "sign"  │
│  code/value     │
└─────────────────┘
```

---

## 第4章 接口设计全景图

### 4.1 wallet-server REST API（6个端点）

| # | 端点 | 方法 | 功能 | 入参 | 出参 | 源码位置 |
|---|------|------|------|------|------|----------|
| 1 | POST /wallet/create | createWallet | 为用户生成TRON热钱包 | TrxUserWalletRequest(userId,userName) | WalletAddressVO(address,userId,userName) | WalletController:48-56 |
| 2 | POST /wallet/withdrawTraction | withdrawTraction | TRC20/TRX提现 | TrxCreateTransactionRequest(userId,toAddress,amount,coinName,ssOrderId,postType) | TrxWalletResult(txId) | WalletController:65-77 |
| 3 | POST /wallet/getBalance | getBalance | 查询用户链上余额 | UserBalanceRequest(userId,currency) | BigDecimal | WalletController:79-90 |
| 4 | POST /wallet/collectTrc20 | collectTrc20 | 单笔归集(手动触发) | TrcCollectRequest(userId,ssOrderId,coinName,amount,postType) | TrxWalletResult(txId) | WalletController:92-107 |
| 5 | POST /wallet/getTransactionInfoById | getTransactionInfoById | 查询链上交易状态 | String orderId | TransactionVO(txId,transResult) | WalletController:109-118 |
| 6 | POST /wallet/getTrxWalletAccount | getTrxWalletAccount | 加密钱包地址信息 | TrxWalletAccountRequest(7个地址/私钥) | TrxWalletAccountVO(7个KMS加密值) | WalletController:120-140 |

### 4.2 user-server 对 wallet-server 的调用封装

```java
// WalletBusiness.java 封装了4个HTTP调用
public class WalletBusiness {
    // 1. 查询链上余额
    public BigDecimal getBalance(Long userId, String virtualCurrency)
    // 源码: WalletBusiness.java:206-232 → POST /wallet/getBalance

    // 2. 获取归集转账结果（充值确认）
    public String getTransferResult(String virtualCurrency, Long orderId, Long userId, BigDecimal realAmount, Integer postType)
    // 源码: WalletBusiness.java:99-130 → POST /wallet/collectTrc20

    // 3. 发起提现交易
    public String withdrawTransaction(SsOrder order)
    // 源码: WalletBusiness.java:36-92 → POST /wallet/withdrawTraction

    // 4. 查询交易状态
    public TransactionVO getTransactionInfoById(String orderId)
    // 源码: WalletBusiness.java:132-164 → POST /wallet/getTransactionInfoById
}
```

### 4.3 调用路径常量

```java
// WalletServerConstants.java
public static final String WALLET_WITHDRAWTRACTION = "/wallet/withdrawTraction";
public static final String WALLET_COLLECT_TRC_20 = "/wallet/collectTrc20";
public static final String WALLET_GET_BALANCE = "/wallet/getBalance";
public static final String WALLET_GET_TRANSACTION_INFO_BY_ID = "/wallet/getTransactionInfoById";
```

### 4.4 接口安全状态

**wallet-server 的 Spring Security 配置**（`WebSecurityConfig.java`）：

```java
// 所有端点均 permitAll()，无任何认证
http.csrf().disable()
    .authorizeRequests()
    .anyRequest().permitAll();
```

**⚠ 严重安全问题**：wallet-server的所有接口**完全无鉴权**，任何能访问内网的人都可以调用提现接口。

---

## 第5章 钱包创建与密钥管理

### 5.1 钱包创建流程

```
POST /wallet/create { userId: "12345", userName: "张三" }
    │
    ▼
WalletTrxBusiness.createWallet() — 源码:99-136
    │
    ├─ Step 1: 幂等检查
    │   trxWalletService.findAddressByUserId(userId)  — :110
    │   └── 如果已存在，直接返回已有地址  — :111-115
    │
    ├─ Step 2: 生成EC密钥对 (secp256k1)
    │   ECKeyPair ecKeyPair = Keys.createEcKeyPair()  — :116
    │   └── 使用web3j的密钥生成器
    │
    ├─ Step 3: 创建钱包文件
    │   WalletFile walletFile = Wallet.createStandard(passphrase, ecKeyPair)  — :117
    │   String originAddress = walletFile.getAddress()  — :118
    │   └── 得到以太坊格式地址（hex，无0x前缀）
    │
    ├─ Step 4: 转换为TRON地址
    │   finalAddress = fromHexAddress("41" + originAddress)  — :119
    │   └── "41"前缀 + Base58Check编码 → T开头的TRON地址
    │
    ├─ Step 5: 加密私钥存储
    │   keyStore = Numeric.toHexStringNoPrefix(ecKeyPair.getPrivateKey())  — :122
    │   enKey = encrypt(keyStore, passphrase + finalAddress)  — :123
    │   └── AES/ECB/PKCS5，密钥=SHA1(passphrase+地址)的前16字节
    │
    ├─ Step 6: 入库
    │   trxWalletService.save(userId, userName, enKey, finalAddress)  — :127
    │
    └─ Step 7: 异步回调user-server
        trxWalletService.updateUserAddress(request)  — :129
        └── @Async POST /user/updateWalletAddressById  — TrxWalletService.java:76-79
```

### 5.2 密钥加密方案解析

```
                                                   ┌───────────────────────┐
                    密钥派生                         │  加密后的私钥(enKey)   │
passphrase + address ──SHA-1──▶ 20字节摘要           │  存入 trx_wallet 表    │
                               │                    │  en_private_key 字段   │
                          取前16字节                  └───────────────────────┘
                               │                              ▲
                               ▼                              │
                       AES-128 密钥(16字节)                     │
                               │                              │
                               ▼                              │
          原始私钥(hex string) ──AES/ECB/PKCS5──▶ Base64(密文) ─┘
```

**加密函数**（`WalletTrxBusiness.java:183-188`）：
```java
public static String encrypt(String strToEncrypt, String secret) throws Exception {
    SecretKeySpec secretKeySpec = new SecretKeySpec(getKey(secret), "AES");
    Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
    cipher.init(Cipher.ENCRYPT_MODE, secretKeySpec);
    return Base64.getEncoder().encodeToString(cipher.doFinal(strToEncrypt.getBytes("UTF-8")));
}
```

**密钥派生函数**（`WalletTrxBusiness.java:197-202`）：
```java
private static byte[] getKey(String secret) throws Exception {
    byte[] bytes = secret.getBytes("UTF-8");
    MessageDigest sha = MessageDigest.getInstance("SHA-1");
    byte[] digest = sha.digest(bytes);
    return Arrays.copyOf(digest, 16);   // 取前16字节 → AES-128
}
```

**解密时机**：
- 归集操作需要用私钥签名 → `TrxWalletService.getPrivateKeyByAddress()` → AES解密  — `TrxWalletService.java:41-52`
- passphrase从AWS KMS解密获取 → `awsKmsDriver.decryptData(passphrase)` — `WalletTrxBusiness.java:108`

### 5.3 AWS KMS 保护机制

**保护对象**（application.yml中的KMS加密值）：
| 配置项 | 用途 | 解密时机 |
|--------|------|----------|
| companyAddress | 公司主地址（归集目标） | 归集时 |
| mainCollectAddress | 归集目标地址（=companyAddress） | 归集时 |
| mainCollectPassphrase | 钱包助记词/口令 | 创建钱包时解密私钥 |
| mainDispensingAddress | 出款地址 | 提现时（但实际从config表读） |
| mainDispensingPrivateKey | 出款私钥 | 提现时（但实际通过外部签名） |
| gasAddress | Gas费地址 | 转Gas时 |
| gasPrivateKey | Gas费私钥 | 转Gas时 |

**KMS调用**（`AwsKmsDriver.java:40-63`）：
- 区域：AP_SOUTHEAST_1（新加坡）
- 每次加密/解密都**新建KmsClient**并在finally中close — 性能隐患
- 错误处理：catch后仅log，返回null — 调用方需判空

**借鉴价值**：
- KMS管理敏感配置的思路值得借鉴，但应该缓存解密结果而非每次调用KMS
- 我们可以用ETCD加密存储替代，或使用Vault/KMS

**风险规避**：
- KMS密钥和Access Key直接写在application.yml中 — 应通过环境变量注入
- 每次操作都new KmsClient → 连接开销大，应单例+连接池

---

## 第6章 区块链充值链路完整解剖

### 6.1 充值总览

TG工程的充值模式是**"用户自主转入 + 系统轮询确认"**，不是传统支付的回调模式：

```
用户通过Telegram机器人获取充值地址
    │  用户在TRON钱包中自行向该地址转入USDT
    │  (这一步完全在链上，系统不介入)
    ▼
user-server 创建 SsOrder（orderStatus=1等待）
    │  用户在TG输入"我已转入XX USDT"
    ▼
user-server 定时任务（每2分钟）+ 用户确认按钮触发
    │  两种触发方式：
    │  1. 用户点击"确认充值"按钮 → triggerByTg=true
    │  2. 定时任务扫描未处理订单 → triggerByTg=false
    ▼
checkCrypotcurrencyOrder() — 充值确认核心流程
    │
    ├─ Phase 1: 查询链上余额（是否已到账）
    │   → POST /wallet/getBalance
    │   └─ 余额 >= realAmount？继续 : 返回等待
    │
    ├─ Phase 2: 触发归集（从用户钱包转到公司地址）
    │   → POST /wallet/collectTrc20
    │   └─ 获取 txId，回填 SsOrder.txId
    │
    ├─ Phase 3: 查询归集交易状态
    │   → POST /wallet/getTransactionInfoById
    │   └─ transResult=true？继续 : 返回等待（下次定时任务再查）
    │
    └─ Phase 4: 上分（增加用户余额）
        → updateOrderStatusAndSendMQ()
        → addBalance() + doPlayMoney()
        → sendMQ() 通知后台统计
```

### 6.2 充值确认核心代码逐行解析

**入口1：用户点击确认按钮**

`VirtualCurrencyBusiness.handleChargeCrypotcurrencyOrder()` — 源码:149-169

```java
public String handleChargeCrypotcurrencyOrder(long orderId, long userId,
        BigDecimal realAmount, String virtualCurrency) {
    // 调用充值检查核心方法，triggerByTg=true
    String errorResult = asyncVirtualDepositService
        .checkCrypotcurrencyOrder(virtualCurrency, orderId, userId, realAmount, true);
    // ...
}
```

**入口2：定时任务扫描**

`HandleVirtualCurrencyTask.checkVirtualCurrencyTask()` — 源码:55-64

```java
@Scheduled(cron = "0 */2 * * * ?", zone = "Asia/Bangkok")  // 每2分钟
public void checkVirtualCurrencyTask() {
    virtualCurrencyBusiness.checkChargeCrypotcurrencyOrder("USDT", 30);
    // 30 = 超过30分钟的订单算超时取消
}
```

**核心方法：checkCrypotcurrencyOrder()** — `AsyncVirtualDepositService.java:96-188`

```java
public String checkCrypotcurrencyOrder(String virtualCurrency, Long orderId,
        Long userId, BigDecimal realAmount, boolean triggerByTg) {

    // ★ 阶段一：如果没有txId，先检查余额再触发归集
    String txId = ssOrder.getTxId();
    if (StringUtils.isEmpty(txId)) {

        // 1. 调用wallet-server查询链上余额
        BigDecimal balance = walletBusiness.getBalance(userId, virtualCurrency);  // :117

        // 2. 余额必须 >= 用户声称的充值金额
        if (balance.compareTo(realAmount) == -1) {  // :126
            return "馀额 小於 充值数量";
        }

        // 3. 余额足够，触发归集（从用户钱包转到公司地址）
        int postType = triggerByTg ? 1 : 2;  // :132
        txId = walletBusiness.getTransferResult(                     // :133
            virtualCurrency, orderId, userId, realAmount, postType);

        // 4. 归集成功得到txId，回填到订单
        if (StringUtils.isNotEmpty(txId)) {
            ssOrderService.updateTxIdById(txId, orderId);  // :142
            return "获取到txId后,下次进入";  // ★ 注意！这里直接返回了！
        }
    }

    // ★ 阶段二：已有txId，查询链上交易状态
    TransactionVO transactionVO = walletBusiness
        .getTransactionInfoById(String.valueOf(ssOrder.getId()));  // :153

    Boolean transactionTxIdResult = transactionVO.getTransResult();
    if (!transactionTxIdResult) {
        return "获取 转帐結果 失败";  // 链上还没确认，下次再查
    }

    // ★ 阶段三：链上已确认成功，执行上分
    boolean result = updateOrderStatusAndSendMQ(orderId, userId, virtualCurrency, txId);
    // :180

    return result ? "SUCCESS" : "FAIL";
}
```

### 6.3 归集操作详解（充值确认的关键步骤）

当充值确认时，user-server调用 `POST /wallet/collectTrc20`，wallet-server执行：

`TrxDriver.singleCollect()` — 源码:175-228

```java
public TrxWalletResult singleCollect(String fromAddress, String userId,
        String coinName, String ssOrderId, BigDecimal reChargeAmount,
        int collectCode, int postType) {

    BigDecimal trxBalance = reChargeAmount;

    // 1. 解密公司主地址
    String newMainAddress = awsKmsDriver.decryptData(companyAddress);  // :179

    // 2. 如果没传金额，查询链上余额
    if (null == reChargeAmount) {
        trxBalance = getAccountBalance(fromAddress, coinName);  // :184
    }

    // 3. 余额检查
    if (minSumAmount >= trxBalance) return null;  // :187

    // 4. ★★★ TRC20需要TRX作为Gas费
    if (!TRXContractEnum.TRX.getContract().equalsIgnoreCase(tokenContract)) {
        BigDecimal addTrx = getAccountBalance(fromAddress, "TRX");  // :194
        if (FEE_BAL(30TRX) > addTrx) {
            // Gas不够！从gasAddress转30TRX到用户地址
            result = createTransactionForTrx(
                awsKmsDriver.decryptData(gasPassword),  // 解密gas私钥
                FEE_BAL,                                 // 30 TRX
                newGasAddress,                           // gas地址
                fromAddress,                             // 用户地址
                TRX,                                     // 主币
                ssOrderId);                              // :196-197

            // 记录Gas转账订单
            trxWalletOrderService.save(...);  // :203-204
        }
    }

    // 5. ★★★ 执行归集：从用户地址转到公司地址
    result = createTransactionForTrx(
        null,              // privateKey=null → 从DB解密
        trxBalance,        // 全部余额
        fromAddress,       // 用户地址
        newMainAddress,    // 公司地址
        tokenContract,     // USDT合约
        ssOrderId);        // :212-213

    // 6. 记录归集和订单
    reverseNoticeCollectDetail(...);  // :221-222

    return trxWalletResult;
}
```

### 6.4 充值过程中的余额变动

```
阶段               wallet-server (链上)              user-server (数据库)
─────────────────────────────────────────────────────────────────────
用户转入USDT        用户地址余额 +N USDT               无变化(SsOrder等待中)
                   (链上事件，系统被动感知)

定时检查            查询用户地址余额                    无变化
(getBalance)        → 发现余额 >= realAmount

触发归集            Gas地址 → 用户地址: 30 TRX(Gas费)   SsOrder.txId = 归集txId
(collectTrc20)      用户地址 → 公司地址: N USDT

交易确认            链上交易 contractRet=SUCCESS         SsOrder.orderStatus = 2(成功)
(getTransactionInfo)                                    SsOrder.completeTime = now

上分                无变化                              assets.balance += realAmount
(addBalance)                                            assets.playMoney += realAmount*flowMultiple
                                                       balance_change 记录一条账变
```

### 6.5 充值链路的关键观察

**① 两阶段确认设计**

充值不是一次性完成的。第一次检查（触发归集）得到txId后**立即返回**（`:143 return "获取到txId后,下次进入"`），等下一个定时周期再查交易状态。这意味着：

- 充值确认**至少需要2个定时周期**（4分钟+）
- 第1次：检查余额 → 触发归集 → 得到txId
- 第2次：查询txId状态 → 确认成功 → 上分

**② "充值"实质是"归集"**

用户感知的"充值"在系统内部实际是一次"归集操作"。用户把USDT转到自己的热钱包地址后，系统做的是把这笔USDT从用户地址转到公司地址。

**③ 轮询而非事件驱动**

系统通过**定时轮询**（每2分钟）来发现充值，而不是监听链上事件。这导致：
- 充值确认延迟至少2-4分钟
- 如果定时任务挂了，充值会卡住
- 30分钟超时后订单自动取消

---

## 第7章 区块链提现链路完整解剖

### 7.1 提现总览

```
用户在Telegram/Casino发起提现请求
    │  输入提现金额和目标TRON地址
    ▼
user-server 创建 SsOrder（orderType=2提现, orderStatus=1等待）
    │  冻结提现金额: assets.freezeAmount += amount
    ▼
VirtualCurrencyWithdrawBusiness.withdraw()
    │
    ├─ 检查是否已有txId（防重复上链）
    │   └─ ssOrder.txId不为空 或 Redis有"order::withdraw_{orderId}" → 跳过
    │
    ├─ 检查是否超时（>30分钟的不处理，等定时任务取消）
    │
    ├─ 调用 walletBusiness.withdrawTransaction(ssOrder)
    │   → POST /wallet/withdrawTraction
    │   └─ wallet-server从出款地址向用户目标地址转账
    │
    ├─ 获得txId后回填 SsOrder
    │
    └─ 设置Redis防重复Key（30天过期）

定时任务（每2分钟）检查有txId的提现订单
    │
    ├─ 调用 getTransactionInfoById 查询链上状态
    │
    ├─ 如果成功：
    │   ├─ 解冻 freezeAmount
    │   ├─ 扣减 balance（实际到账金额）
    │   ├─ 扣减 balance（手续费）
    │   ├─ 更新 orderStatus = SUCCESS
    │   └─ 发MQ通知后台
    │
    └─ 如果超时(30分钟)且失败：
        ├─ 解冻 freezeAmount
        └─ 更新 orderStatus = SYSTEM_CANCEL
```

### 7.2 wallet-server 提现核心代码

`TrxDriver.withdrawTraction()` — 源码:281-337

```java
public TrxWalletResult withdrawTraction(int orderType, String tokenContract,
        TrxCreateTransactionRequest request) {

    // 1. 从config表获取出款地址（不是从yml解密）
    List<TrxWalletConfigEntity> list = trxWalletConfigRepository.findByType("sign");
    String fromAddress = list.stream()
        .filter(obj -> obj.getCode().equals("disPendAddress"))
        .findFirst().get().getValue();  // :285-287

    // 2. ★ 幂等检查：查订单表是否已有该订单
    TrxWalletOrderEntity order = trxWalletOrderRepository.findBySsOrderId(ssOrderId);
    if (ObjectUtil.isNotEmpty(order)) {
        result = new TrxWalletResult();
        result.setTxId(order.getTxId());
        return result;  // 直接返回已有txId  — :298-303
    }

    // 3. 余额检查：出款地址余额 >= 提现金额
    BigDecimal balance = getAccountBalance(fromAddress, coinName);  // :294
    if (sendAmount > balance) {
        return null;  // :304-307
    }

    // 4. TRC20需要TRX Gas费（与归集逻辑完全相同）
    if (!TRX.equals(tokenContract)) {
        BigDecimal addTrx = getAccountBalance(fromAddress, "TRX");
        if (30 > addTrx) {
            // 从gas地址转30TRX到出款地址
            result = createTransactionForTrx(gasPassword, 30, gasAddress, fromAddress, TRX, ssOrderId);
            trxWalletOrderService.save(...);
        }
    }

    // 5. ★★★ 执行提现：从出款地址转到用户目标地址
    result = createTransactionForTrx(
        fromAddress,    // ★ 注意！这里传的是地址而非私钥
        sendAmount, fromAddress, toAddress, tokenContract, ssOrderId);  // :329-330

    // 6. 记录订单
    trxWalletOrderService.save(...);  // :332-333

    return result;
}
```

### 7.3 提现签名机制（外部签名系统）

提现的关键区别：归集时用**本地私钥签名**，提现时用**外部签名系统**。

判断逻辑在 `WalletTrxBusiness.trc20Transaction()` — 源码:248

```java
// 只有提款才会调用加签系统，归集加签走正常签名
if (deDispensingAddress.equals(from)) {
    // 从出款地址发起的交易 → 调用外部签名系统
    bytes = sendSign(signUrl, tx, ssOrderId, from);  // :249
} else {
    // 从用户地址发起的交易(归集) → 本地ECKey签名
    bytes = signTransaction2Byte(tx.toByteArray(), ByteArray.fromHexString(keyStore));  // :254
}
```

**外部签名流程**（`WalletTrxBusiness.sendSign()` — 源码:292-321）：

```
1. 计算交易哈希: txIdHash = SHA256(tx.rawData)                 — :295
2. Base64编码: signTxid = Base64(txIdHash)                      — :296
3. 构造签名请求:
   {
     "orderNo": ssOrderId,
     "outAddressNo": from(出款地址),
     "chainType": "TRON",
     "hashId": signTxid
   }                                                            — :297-301
4. RSA加密请求体（RSA-512，公钥从config表获取）                   — :302-303
5. HTTP POST to 签名系统:
   URL: http://34.92.121.9:80/api/admin/chain/sign
   Header: PLAT_CODE={商户码}
   Body: RSA加密后的字符串                                        — :305-311
6. 签名系统返回签名值:
   { "code": "200", "data": "Base64(签名字节)" }                 — :316-318
7. 组装完整交易:
   tx.addSignature(签名字节)                                     — :320
```

### 7.4 user-server 提现后处理

**提现成功后的4步操作**（`AsyncVirtualWithdrawService.withdrawUSDT()` — 源码:125-201）：

```java
// Step 1: 解冻金额
doFreezeAmount(ssOrder, user);  // :180
// → assets.freezeAmount -= amount, freezeAmountType=EXPENSES(2)

// Step 2: 减少实际到账金额
reduceRealAmount(ssOrder, user, ssOrder.getRealAmount());  // :183
// → assets.balance -= realAmount, changeType=TELEGRAM_WITHDRAW(8)

// Step 3: 减少手续费
doBalanceFeeBusiness(ssOrder, user, ssOrder.getFeeAmount());  // :186
// → assets.balance -= feeAmount, changeType=FEE(90)

// Step 4: 更新订单状态 + 发MQ
ssOrderService.updateWithdrawOrderStatus(SUCCESS, txId, orderId, ...);  // :188
sendMQforWithdraw(orderId, userId, amount, ...);  // :195
notifyWallet(currentBalance, orderId, orderType, txId);  // :199
```

### 7.5 提现手续费计算

```java
// WalletBusiness.withdrawTransaction() — 源码:47
BigDecimal afterAmount = amount.subtract(order.getFeeAmount())
    .setScale(2, RoundingMode.HALF_UP);
```

**逻辑**：提现金额 = 用户输入金额(amount) - 手续费(feeAmount)
- 链上实际转账的是 `afterAmount`
- 手续费从用户余额中额外扣除（Step 3）

---

## 第8章 归集机制深度分析

### 8.1 两种归集触发方式

| 类型 | 触发方式 | collectType | postType | 触发点 |
|------|----------|-------------|----------|--------|
| 单笔归集 | 充值确认时触发 | SINGLE(1) | 1(API)或2(定时) | user-server → /wallet/collectTrc20 |
| 批量归集 | 每天8:00定时任务 | COLLECT(2) | 2(定时) | CollectTrxJob.collectUSDT() |

### 8.2 定时批量归集流程

`CollectTrxJob.collectUSDT()` — 源码:34-39

```java
@Scheduled(cron = "0 0 8 * * ?", zone="Asia/Bangkok")  // 每天曼谷时间8:00
public void collectUSDT() {
    TrcCollectRequest request = new TrcCollectRequest();
    request.setCoinName("USDT");
    trxDriver.collectTrc20(request);
}
```

`TrxDriver.collectTrc20()` — 源码:230-255

```java
public Boolean collectTrc20(TrcCollectRequest request) {
    String newMainAddress = awsKmsDriver.decryptData(companyAddress);

    // 遍历所有用户钱包
    List<TrxWalletEntity> list = trxWalletService.findAll();  // ★ 全表扫描！
    for (TrxWalletEntity wallet : list) {
        String ssOrderId = UUID.randomUUID().toString().replaceAll("-","");

        if (newMainAddress.equals(wallet.getAddress())) {
            continue;  // 跳过公司地址自身
        }

        // 对每个钱包执行单笔归集
        singleCollect(wallet.getAddress(), wallet.getUserId(), "USDT",
            ssOrderId, null, COLLECT(2), 2);
    }
    return true;
}
```

### 8.3 归集过程的Gas费处理

```
对每个用户钱包执行归集：

1. 查询该用户地址的USDT余额
   └── 余额=0？跳过

2. 查询该用户地址的TRX余额（用于Gas费）
   └── TRX >= 30？直接执行转账

3. TRX < 30？需要"充能"
   ├── 从gasAddress向该用户地址转30 TRX
   ├── 这笔TRX转账本身也有Gas费（由gasAddress的TRX支付）
   └── 记录这笔Gas转账到trx_wallet_order

4. 执行USDT归集转账
   ├── 从用户地址向公司地址转全部USDT
   ├── 用户地址需要有足够TRX支付Gas（Step 3保证了）
   └── 签名：用用户的私钥（从DB解密）本地签名

5. 记录归集到 trx_wallet_collect + trx_wallet_order
```

### 8.4 归集的关键问题

**问题1：Gas费时序竞争**

从gasAddress转TRX到用户地址后，**立即**执行USDT归集。但TRON链上TRX转账需要区块确认时间（约3秒），如果在TRX尚未到账时就发起USDT转账，会因Gas不足而失败。

**源码证据**：`TrxDriver.singleCollect()` 在转Gas后没有任何等待/确认逻辑（:196-206），直接执行归集（:212-213）。

**问题2：批量归集的全表扫描**

`trxWalletService.findAll()` 会加载**所有用户钱包**到内存。如果用户量大（如10万），这会造成：
- 数据库压力（全表扫描无分页）
- 内存压力（所有Entity加载到JVM）
- 归集任务执行时间极长（每个用户2-3次链上交易）

**问题3：归集失败无重试**

单个用户归集失败（Gas不足/签名失败/网络超时），catch后仅log，下一个用户继续。失败的用户余额不会在当天重试，要等第二天的定时任务。

---

## 第9章 Gas费管理与能量机制

### 9.1 TRON Gas费模型

```
TRON链上交易费用模型：
├── TRX主币转账：带宽(Bandwidth)消耗，通常≈0.3 TRX（带宽充足时免费）
├── TRC20代币转账：
│   ├── 带宽消耗：≈0.35 TRX
│   └── 能量(Energy)消耗：≈27,000-65,000 Energy ≈ 13-30 TRX
│   └── 总计：约14-31 TRX（取决于是否质押了Energy）
└── 智能合约调用：取决于合约复杂度
```

### 9.2 系统中的Gas费策略

**常量定义**：

```java
// TrxDriver.java:93
private static final BigDecimal FEE_BAL = new BigDecimal("30");  // 归集Gas预算

// WalletTrxBusiness.java:236
map.put("fee_limit", 50000000L);  // trc20Transaction的fee_limit = 50 TRX

// Trc20.java:34
public static final Long FEE_LIMIT = 10000000L;  // Trc20.transfer的fee_limit = 10 TRX
```

**⚠ 不一致**：trc20Transaction中fee_limit=50TRX，但Trc20.transfer中fee_limit=10TRX。虽然不是同一个调用路径，但体现了Gas费策略的不统一。

### 9.3 Gas费流转图

```
gasAddress (Gas费专用地址)
    │
    │ 归集时：转30 TRX → 用户热钱包地址
    │ 提现时：转30 TRX → 出款地址(dispensingAddress)
    │
    │ ★ Gas费由谁承担？
    │   归集时：平台承担（从gasAddress出）
    │   提现时：平台先垫付Gas，但手续费从用户余额扣
    │
    ▼
用户地址/出款地址
    │ 有了TRX后才能执行TRC20转账
    │ 转账消耗的Gas费从该地址TRX余额扣
    ▼
TRON网络
```

### 9.4 Gas费地址的补充机制

**当前系统中没有gasAddress余额不足的监控和补充机制**。

如果gasAddress的TRX余额耗尽：
- 所有归集操作都会失败（因为无法为用户地址充TRX Gas）
- 所有提现操作都会失败（因为无法为出款地址充TRX Gas）
- 系统**不会报警**，仅在日志中记录失败

**借鉴与规避**：
- 必须监控Gas地址余额，设置阈值告警
- 当Gas不足时应该暂停归集/提现，而不是静默失败

---

## 第10章 余额引擎与账变机制

### 10.1 余额体系

TG工程的余额管理**不在wallet-server**，而在**user-server**的Assets表：

```
assets表（单行余额模型）
├── balance: 主余额（充值+、提现-、投注-、结算+）
├── freezeAmount: 冻结金额（提现时冻结，完成后解冻）
├── playMoney: 打码量（充值时增加，达标后可提现）
├── creditAmount: 信用额度
├── balanceNn: 牛牛游戏余额（分游戏的子余额）
├── balanceBacc: 百家乐余额
└── version: 乐观锁版本号
```

### 10.2 充值上分操作

`AsyncVirtualDepositService.addBalance()` — 源码:410-431

```java
private void addBalance(SsOrder ssOrder, User user, BigDecimal balance) {
    BalanceVO balanceVO = new BalanceVO();
    balanceVO.setAmount(balance);
    balanceVO.setBalanceType(BalanceTypeEnum.INCOME.getCode());      // 1=收入
    balanceVO.setChangeType(BalanceChangeEnum.TELEGRAM_CHARGE.getCode()); // 充值
    balanceVO.setUserId(ssOrder.getUserId());
    balanceVO.setRelateId(ssOrder.getId());
    balanceVO.setRelateOrderNum(ssOrder.getOrderNum());
    balanceVO.setRemark("前端虚拟币充值(增加馀额)");

    userAssetsBusiness.doBalanceBusiness(user, balanceVO);
}
```

### 10.3 提现扣款三步曲

```
Step 1: 解冻（提现发起时已冻结，现在释放冻结金额）
    freezeAmountType = EXPENSES(2)
    assets.freezeAmount -= amount

Step 2: 扣减实际到账金额
    balanceType = EXPENSES(2)
    changeType = TELEGRAM_WITHDRAW(8)
    assets.balance -= realAmount

Step 3: 扣减手续费
    balanceType = EXPENSES(2)
    changeType = FEE(90)
    assets.balance -= feeAmount
```

### 10.4 乐观锁机制

Assets表使用JPA `@Version` 注解实现乐观锁 — `Assets.java:49-51`：

```java
@Version
@Column(name = "version")
private Integer version = 1;
```

**工作原理**：
```sql
UPDATE assets SET balance = ?, version = version + 1
WHERE id = ? AND version = ?
```

如果version不匹配（并发修改），JPA抛出 `OptimisticLockingFailureException`。

**问题**：代码中**没有显式处理乐观锁冲突**的重试逻辑。如果充值和提现同时修改同一用户的Assets，其中一个会失败，但没有重试。

---

## 第11章 并发控制与一致性机制

### 11.1 并发控制手段总览

| 层次 | 机制 | 使用场景 | 源码位置 |
|------|------|----------|----------|
| 应用层 | Redisson FairLock | 充值/提现定时任务处理 | AsyncVirtualDepositService:197-199 |
| 应用层 | Redis Key防重复 | 防止提现重复上链 | VirtualCurrencyWithdrawBusiness:198 |
| 数据层 | JPA @Version乐观锁 | Assets表余额更新 | Assets.java:49-51 |
| 接口层 | ssOrderId幂等检查 | wallet-server提现接口 | TrxDriver.java:298-303 |
| 接口层 | txId非空检查 | 防重复上链 | VirtualCurrencyWithdrawBusiness:192-195 |

### 11.2 Redisson分布式锁详解

**充值任务锁**（`AsyncVirtualDepositService.deposit()` — 源码:191-224）：

```java
String key = "USER_SERVER_DEPOSIT_TASK_SS_ORDER_ID" + orderId;
RLock fairLock = redisson.getFairLock(key);
boolean res = fairLock.tryLock(0L, 10L, TimeUnit.SECONDS);
// 0秒等待：抢不到就放弃，等下次定时任务
// 10秒自动释放：防止持锁方异常退出后死锁

if (res) {
    try {
        // 重新查询订单状态（双重检查）
        ssOrder = ssOrderService.findById(orderId);
        if (ssOrder.getOrderStatus() != VIRTUAL_CURRENCY_WAIT) {
            return;  // 已被其他实例处理
        }
        // 执行充值确认
        checkCrypotcurrencyOrder(...);
    } finally {
        fairLock.unlock();
    }
}
```

**提现任务锁**（`AsyncVirtualWithdrawService.withdrawVirtualCurrencyTask()` — 源码:64-122）：
- 同样的FairLock模式
- 锁Key: `USER_SERVER_WITHDRAW_TASK_SS_ORDER_ID{orderId}`
- 超时: UNLOCK_TIME（配置值）

### 11.3 幂等性保障

**wallet-server提现幂等**：

```java
// TrxDriver.withdrawTraction() — :298-303
TrxWalletOrderEntity order = trxWalletOrderRepository.findBySsOrderId(ssOrderId);
if (ObjectUtil.isNotEmpty(order)) {
    result = new TrxWalletResult();
    result.setTxId(order.getTxId());
    return result;  // 已有订单，直接返回已有txId
}
```

**user-server提现防重复**：

```java
// VirtualCurrencyWithdrawBusiness.withdraw() — :192-199
if (StringUtils.isNotEmpty(ssOrder.getTxId())) {
    return true;  // 已有txId，不重复上链
}
String orderIdToTxIdKey = "order::withdraw_" + orderId;
if (redisUtil.hasKey(orderIdToTxIdKey)) {
    return true;  // Redis标记已处理
}
// ... 处理完成后
redisUtil.set(orderIdToTxIdKey, orderIdToTxIdKey, 30 * 24, TimeUnit.HOURS);
```

### 11.4 一致性问题分析

**问题1：跨库一致性无保障**

wallet-server(cloud_wallet)和user-server(cloud_user)是独立数据库，没有分布式事务。可能出现：
- wallet-server的trx_wallet_order已记录，但user-server因网络问题没收到txId
- 链上交易成功，但user-server上分失败

**当前补偿策略**：定时任务每2分钟重新检查。如果上次失败了，下一个周期会重试。

**问题2：归集成功但上分失败**

归集（链上转账）是不可逆的。如果归集成功但user-server上分失败，用户余额不会增加，但资金已经从用户地址转到公司地址。补偿手段只有定时任务重试。

**问题3：提现扣款非原子**

提现成功后的4步操作（解冻→扣余额→扣手续费→更新状态）不在同一个事务中。如果中间步骤失败，可能导致：
- 解冻了但没扣款（余额多了）
- 扣了余额但没扣手续费

---

## 第12章 安全设计分析

### 12.1 加密体系总览

```
                     安全层次
┌─────────────────────────────────────────────────────┐
│ 传输层                                               │
│ ├── 服务间通信: HTTP(明文) — 无HTTPS/mTLS           │
│ └── 与签名系统: HTTP(RSA-512加密请求体)              │
│                                                      │
│ 存储层                                               │
│ ├── 配置文件: AWS KMS加密（地址/私钥/助记词）         │
│ ├── 数据库: AES-128-ECB加密用户私钥                  │
│ └── 数据库连接: 明文密码在application.yml              │
│                                                      │
│ 签名层                                               │
│ ├── 归集签名: 本地ECKey(secp256k1)                   │
│ └── 提现签名: 外部签名系统(RSA-512通信)              │
└─────────────────────────────────────────────────────┘
```

### 12.2 安全缺陷清单

| # | 缺陷 | 严重程度 | 源码位置 | 说明 |
|---|------|----------|----------|------|
| S1 | AES-ECB模式加密私钥 | 高 | WalletTrxBusiness:185 | ECB模式相同明文产生相同密文，可被模式分析攻击。应使用CBC/GCM |
| S2 | RSA-512位密钥 | 高 | RSAUtil:16 | 512位RSA早已不安全（可在数小时内破解），应≥2048位 |
| S3 | 无API认证 | 致命 | WebSecurityConfig permitAll() | 内网任何主机可直接调用提现接口 |
| S4 | AWS凭证明文 | 高 | application.yml:91-94 | accessKeyId和secretAccessKey明文存储在配置文件 |
| S5 | 数据库密码明文 | 中 | application.yml:58 | password: 1Qaz@wsx |
| S6 | SHA-1密钥派生 | 中 | WalletTrxBusiness:199 | SHA-1已被证明不安全，应使用PBKDF2/Argon2/scrypt |
| S7 | 私钥解密后在内存中明文 | 中 | TrxWalletService:45 | 解密后的私钥作为String在JVM中无法保证及时清除 |
| S8 | KMS每次新建Client | 低 | AwsKmsDriver:98-107 | 非安全问题但影响性能，连接可能泄露 |

### 12.3 签名系统的安全设计

外部签名系统的设计思路是**密钥隔离**：

```
wallet-server（业务系统）    签名系统（独立部署）
    │                            │
    │  不持有出款私钥             │  持有出款私钥
    │  只发送交易哈希             │  只签名交易哈希
    │                            │
    │  ── RSA加密请求 ──▶        │
    │                            │  验证请求来源(PLAT_CODE)
    │                            │  对hashId执行secp256k1签名
    │  ◀── 返回签名值 ──         │
    │                            │
    │  组装完整交易+广播           │
```

**借鉴价值**：密钥隔离是正确的安全架构思路。出款私钥不在业务系统中，即使业务系统被攻破也无法直接提走资金。

**不足**：
- RSA-512加密太弱，通信安全形同虚设
- 仅通过PLAT_CODE（商户码）验证请求来源，无签名验证

---

## 第13章 缺陷清单与风险评估

### 13.1 功能缺陷

| # | 缺陷 | 影响 | 源码位置 |
|---|------|------|----------|
| F1 | userId类型不一致 | wallet用varchar(10)，user用bigint(20)。userId>9999999999时溢出 | TrxWalletEntity:29 vs SsOrder:35 |
| F2 | findBySsOrderId硬编码USDT | 如果支持其他币种提现会查不到 | TrxWalletOrderRepository:19-21 |
| F3 | Gas转账后无等待确认 | 归集时Gas可能未到账就执行TRC20转账 | TrxDriver:196-212 |
| F4 | 批量归集全表扫描 | 用户量大时OOM/超时 | TrxDriver:240 findAll() |
| F5 | 归集失败无重试 | 单用户归集失败后到第二天才重试 | TrxDriver:248-249 catch后continue |
| F6 | 乐观锁冲突无重试 | 并发修改Assets时可能丢失更新 | Assets.java @Version无retry |
| F7 | 提现扣款非原子 | 解冻/扣余额/扣手续费不在同一事务 | AsyncVirtualWithdrawService:179-195 |
| F8 | KMS每次新建Client | 每次加解密都创建新的KmsClient，资源泄露风险 | AwsKmsDriver:98-107 |
| F9 | 充值需要两个定时周期 | 最快4分钟确认，用户体验差 | AsyncVirtualDepositService:143 |
| F10 | config表Stream过滤 | 每次提现都查全量config+Stream过滤，效率低 | TrxDriver:285-287 |

### 13.2 架构缺陷

| # | 缺陷 | 影响 | 说明 |
|---|------|------|------|
| A1 | 服务间HTTP明文通信 | 中间人攻击风险 | wallet-server和user-server间无TLS |
| A2 | 无链上事件监听 | 充值确认依赖轮询，延迟高 | 应该监听TRC20 Transfer事件 |
| A3 | 无Gas余额监控 | gasAddress耗尽后所有操作静默失败 | 缺少告警和自动补充 |
| A4 | 无交易状态机 | 订单状态转换无严格校验 | 理论上可以从CANCEL跳到SUCCESS |
| A5 | 跨库一致性靠定时任务 | 补偿延迟高(2分钟)，中间态可见 | 无分布式事务或Saga |
| A6 | 单点签名系统 | 签名系统挂了所有提现都会失败 | 无备份/降级方案 |

### 13.3 运维缺陷

| # | 缺陷 | 影响 | 说明 |
|---|------|------|------|
| O1 | Hibernate DDL auto-update | 生产环境不应自动修改表结构 | application.yml:53 |
| O2 | 无健康检查端点 | 无法监控链上连接状态 | 链挂了/gRPC断了无感知 |
| O3 | 错误处理=catch+log+返回null | 调用方需要大量null判断 | 全局无统一异常处理 |
| O4 | Swagger生产环境开启 | 泄露API文档 | project.swagger.enable=true |

---

## 第14章 优秀设计借鉴清单

### 14.1 值得借鉴的设计

| # | 设计点 | 描述 | 借鉴方式 |
|---|--------|------|----------|
| G1 | 每用户独立链上地址 | 注册时生成专属TRON地址，充值追踪清晰 | 我们同样需要为用户生成独立充值地址 |
| G2 | 密钥隔离（外部签名系统） | 出款私钥不在业务系统中，即使被攻破也不丢资金 | 核心安全架构，必须采纳 |
| G3 | AWS KMS管理敏感配置 | 地址/私钥/助记词用KMS加密存储 | 可用ETCD加密或Vault替代 |
| G4 | 归集机制（资金汇总） | 定时+手动两种归集方式，资金集中管理 | 归集是虚拟币钱包的标配能力 |
| G5 | Gas费自动补充 | TRC20操作前自动检查TRX余额并补充 | 归集和提现都需要此机制 |
| G6 | ssOrderId幂等检查 | 提现前先查是否已有订单记录 | 简单有效的接口级幂等 |
| G7 | Redisson分布式锁+双重检查 | 获取锁后重新查询订单状态 | 防止定时任务重复处理的标准模式 |
| G8 | 充值两阶段确认 | 先归集拿txId，再查交易状态确认 | 区块链确认的合理模式 |
| G9 | 提现冻结→扣款→解冻模式 | 提现时先冻结余额，成功后扣款+解冻，失败仅解冻 | 防止提现期间余额被其他操作消费 |
| G10 | postType区分调用来源 | 1=API手动，2=定时任务。便于审计和问题排查 | 简单实用的审计字段 |

### 14.2 需要调整的设计

| # | 原设计 | 调整方向 |
|---|--------|----------|
| G1→ | 每2分钟轮询确认充值 | 改为链上事件监听（WebSocket/gRPC stream），轮询作为兜底 |
| G4→ | 全表扫描批量归集 | 分批归集 + 只归集有余额的地址（维护待归集队列） |
| G5→ | Gas转账后立即执行 | Gas转账后等待确认（查交易状态）再执行后续操作 |
| G6→ | ssOrderId幂等仅在wallet-server | user-server也应有业务幂等（目前靠Redis Key，但30天过期） |
| G9→ | 解冻+扣余额+扣手续费分开执行 | 应在同一个事务中完成，或用CAS原子操作 |

### 14.3 需要规避的设计

| # | 原设计 | 规避原因 |
|---|--------|----------|
| ✗ | AES-ECB加密 | 不安全，改用AES-GCM或AES-CBC |
| ✗ | RSA-512 | 已可破解，改用RSA-2048+ |
| ✗ | permitAll无认证 | 内网服务也需要认证（签名/Token/mTLS） |
| ✗ | Hibernate DDL auto-update | 生产环境必须手动管理DDL |
| ✗ | catch+log+return null | 应有统一错误码和异常处理 |
| ✗ | 私钥String存内存 | 应使用byte[]并尽快清零，或用安全内存区域 |

---

## 第15章 与 ser-wallet 的对比映射

### 15.1 职责边界对比

| 能力 | TG wallet-server | 我们的 ser-wallet |
|------|------------------|-------------------|
| 钱包创建 | TRON地址生成 | 多币种钱包创建（法币+虚拟币） |
| 充值 | 链上轮询确认+归集 | 法币充值(支付渠道回调)+虚拟币充值(链上确认) |
| 提现 | 链上TRC20/TRX转账 | 法币提现(银行卡)+虚拟币提现(链上转账) |
| 余额管理 | ✗（在user-server） | ★ 核心职责（多币种余额） |
| 兑换 | ✗ 无 | ★ 核心职责（币种间兑换） |
| 归集 | 定时+手动 | 虚拟币归集 |
| 币种配置 | 硬编码枚举(3种) | ★ 动态配置（后台管理） |
| 账变记录 | ✗（在user-server） | ★ 核心职责 |

### 15.2 架构模式对比

| 维度 | TG工程 | 我们的工程 |
|------|--------|-----------|
| 余额存储 | user-server.assets表（单行模型） | ser-wallet（多币种多行模型） |
| 并发控制 | JPA @Version乐观锁(无重试) | GORM CAS乐观锁(带重试) |
| 幂等保障 | ssOrderId查重 + Redis Key | 双层幂等（Redis + DB唯一键） |
| 事务管理 | @Transactional（单库），跨库无事务 | GORM手动Transaction + 补偿 |
| 服务通信 | HTTP REST（明文JSON） | Kitex RPC（Thrift IDL，内网高效） |
| 定时任务 | @Scheduled（单机） | 待定（建议分布式调度） |

### 15.3 我们需要从TG工程汲取的实现经验

**① 区块链充值确认链路**：
- 用户转入 → 余额查询 → 归集触发 → 交易确认 → 上分
- 两阶段确认模式（先拿txId，再确认状态）
- 超时取消机制（30分钟未确认自动取消）

**② 归集机制**：
- Gas费自动补充的必要性（TRC20转账需要TRX作为手续费）
- 单笔归集+批量归集的双模式
- 归集记录审计（单独的collect表）

**③ 提现安全架构**：
- 密钥隔离：出款私钥不放在业务系统
- 外部签名系统：业务系统只发交易哈希，签名系统负责加签
- 提现冻结机制：先冻结再转账，成功后扣款，失败后解冻

**④ Gas费管理**：
- 独立的Gas费地址
- TRC20操作前自动检查并补充Gas
- 30 TRX的Gas预算（覆盖1-2次TRC20转账）

---

## 第16章 总结：取舍决策表

### 16.1 完整取舍矩阵

| 设计点 | 采纳 | 调整 | 规避 | 优先级 | 理由 |
|--------|------|------|------|--------|------|
| 每用户独立充值地址 | ★ | | | P0 | 虚拟币充值的基础架构 |
| 密钥隔离(外部签名) | ★ | | | P0 | 资金安全的底线 |
| 充值两阶段确认 | ★ | | | P0 | 区块链确认的合理模式 |
| 提现冻结→扣款模式 | ★ | | | P0 | 防止超扣的标准做法 |
| Gas费自动补充 | ★ | | | P0 | TRC20转账的必要前提 |
| 归集机制(单笔+批量) | ★ | | | P1 | 资金集中管理的标配 |
| ssOrderId幂等 | ★ | | | P0 | 接口级防重 |
| Redisson分布式锁 | | ★ | | P1 | 我们用go-redis实现类似逻辑 |
| postType审计字段 | ★ | | | P2 | 低成本高价值 |
| 轮询确认充值 | | ★ | | P1 | 改为事件驱动+轮询兜底 |
| 全表扫描归集 | | | ★ | - | 改为分批+有余额队列 |
| Gas转账后无等待 | | | ★ | - | 必须等Gas确认后再执行 |
| AES-ECB加密 | | | ★ | - | 改用AES-GCM |
| RSA-512签名 | | | ★ | - | 改用RSA-2048 |
| 无API认证 | | | ★ | - | 即使内网也要认证 |
| Hibernate DDL auto | | | ★ | - | 手动管理DDL |
| catch+return null | | | ★ | - | 统一错误码 |
| 跨库无事务 | | ★ | | P1 | 采用Saga补偿模式 |
| KMS密钥管理 | | ★ | | P1 | 改用ETCD加密或Vault |
| 乐观锁@Version | | ★ | | P0 | GORM CAS+重试替代 |

### 16.2 核心结论

**TG工程wallet-server的最大价值**：完整实现了一套TRON链上虚拟币充值/提现/归集链路，虽然实现粗糙（安全/性能/可靠性都有明显缺陷），但**链路设计思路是对的**：

1. **充值 = 用户转入 + 余额检查 + 归集转账 + 链上确认 + 上分**
2. **提现 = 冻结余额 + 外部签名 + 链上转账 + 确认 + 解冻扣款**
3. **归集 = Gas补充 + 用户→公司转账 + 记录审计**
4. **安全 = 密钥隔离 + KMS管理 + 外部签名**

我们要做的是**取其链路设计精华**，在实现层面全面提升——更强的加密、更好的并发控制、更可靠的事务一致性、更高效的事件驱动机制。

---

## 附录A 完整文件清单与代码行数

### wallet-server 核心文件

| 文件 | 路径 | 行数 | 职责 |
|------|------|------|------|
| WalletTrxBusiness.java | business/ | 538 | TRON核心操作（创建钱包/TRC20转账/TRX转账/余额查询/签名） |
| TrxDriver.java | driver/ | 338 | 门面层（归集/提现/余额查询的编排） |
| AwsKmsDriver.java | driver/ | 110 | AWS KMS加解密 |
| WalletController.java | controller/ | 143 | REST API（6个端点） |
| TronUtil.java | client/trx/ | 122 | TRON gRPC客户端包装 |
| SmartContract.java | client/trx/ | 91 | 智能合约调用（constant/trigger） |
| Trc20.java | client/trx/ | 125 | TRC20代币操作（balanceOf/transfer/decimals） |
| TrxWalletEntity.java | model/ | 43 | 用户钱包实体 |
| TrxWalletOrderEntity.java | model/ | 65 | 交易订单实体 |
| TrxWalletCollectEntity.java | model/ | 48 | 归集记录实体 |
| TrxWalletConfigEntity.java | model/ | 39 | 配置实体 |
| TRXContractEnum.java | model/enums/ | 123 | 支持的币种枚举（TRX/USDT/CXT） |
| TrxWalletService.java | service/ | 80 | 钱包数据服务 |
| TrxWalletOrderService.java | service/ | 55 | 订单数据服务 |
| TrxWalletCollectService.java | service/ | 39 | 归集数据服务 |
| CollectTrxJob.java | task/ | 40 | 定时归集任务 |
| AES.java | util/ | 47 | AES加解密 |
| RSAUtil.java | util/ | 110 | RSA加解密 |
| Base58.java | util/ | ~150 | TRON地址编解码 |
| TronUtils.java | util/ | ~474 | TRON工具集 |

### user-server 钱包相关文件

| 文件 | 路径 | 行数 | 职责 |
|------|------|------|------|
| WalletBusiness.java | business/ | 234 | wallet-server HTTP调用封装 |
| VirtualCurrencyBusiness.java | business/ | 248 | 充值业务（超时处理/订单查询） |
| VirtualCurrencyWithdrawBusiness.java | business/ | 336 | 提现业务（发起/失败处理/超时） |
| AsyncVirtualDepositService.java | service/ | 454 | 异步充值处理（核心确认逻辑） |
| AsyncVirtualWithdrawService.java | service/ | 361 | 异步提现处理（确认+扣款） |
| HandleVirtualCurrencyTask.java | task/ | 66 | 定时任务（每2分钟） |
| SsOrder.java | model/ | 220 | 订单实体 |
| Assets.java | model/ | 118 | 资产实体（含乐观锁） |

---

## 附录B 关键代码片段索引

### B.1 创建钱包

```
WalletTrxBusiness.createWallet()
  → Keys.createEcKeyPair()                     :116  EC密钥对生成
  → Wallet.createStandard(passphrase, ecKeyPair) :117  创建钱包文件
  → fromHexAddress("41" + originAddress)         :119  转TRON地址
  → encrypt(keyStore, passphrase + address)      :123  AES加密私钥
  → trxWalletService.save()                      :127  入库
  → trxWalletService.updateUserAddress()         :129  异步回调user-server
```

### B.2 TRC20转账

```
WalletTrxBusiness.trc20Transaction()
  → validateAddress(to)                          :217  地址校验
  → encoderAbi(to, amount)                       :231  ABI编码transfer(address,uint256)
  → HttpClient4Util.doPost(/wallet/triggersmartcontract) :237  构建未签名交易
  → TronUtils.packTransaction()                  :242  JSON→Protobuf
  → if(dispensingAddress.equals(from)):           :248  判断签名方式
      sendSign()                                 :249  外部签名
    else:
      signTransaction2Byte()                     :254  本地签名
  → HttpClient4Util.doPost(/wallet/broadcasthex) :261  广播交易
```

### B.3 充值确认

```
AsyncVirtualDepositService.checkCrypotcurrencyOrder()
  → walletBusiness.getBalance()                  :117  查链上余额
  → walletBusiness.getTransferResult()           :133  触发归集
  → ssOrderService.updateTxIdById()              :142  回填txId
  → walletBusiness.getTransactionInfoById()      :153  查交易状态
  → updateOrderStatusAndSendMQ()                 :180  上分+通知
    → addBalance()                               :322  增加余额
    → doPlayMoney()                              :327  增加打码量
    → sendMQ()                                   :333  MQ通知后台
```

### B.4 提现流程

```
VirtualCurrencyWithdrawBusiness.withdraw()
  → ssOrder.getTxId() 检查                       :192  防重复
  → redisUtil.hasKey("order::withdraw_"+id)      :198  Redis防重复
  → walletBusiness.withdrawTransaction()         :231  发起提现
    → TrxDriver.withdrawTraction()
      → findBySsOrderId() 幂等检查               :298
      → getAccountBalance() 余额检查              :294
      → createTransactionForTrx() 转账            :329
  → ssOrderService.updateTxIdById()              :240  回填txId
  → redisUtil.set("order::withdraw_"+id, 30天)   :243  设置防重复Key
```

---

## 附录C TRON链基础知识速查

### C.1 地址格式

```
以太坊地址:  0x + 40字符hex = 42字符
TRON地址:   "41" + 40字符hex → Base58Check编码 → T开头34字符
转换关系:
  ETH: 0xAbCdEf...
  TRON hex: 41AbCdEf...
  TRON Base58: T + Base58Check(41AbCdEf...) = TUyKqdNXNEzAvjiBWQJ3oo...
```

### C.2 货币单位

```
1 TRX = 1,000,000 SUN (10^6)
1 USDT(TRC20) = 1,000,000 最小单位 (decimals=6)
```

### C.3 交易类型

```
TRX主币转账:      createTransaction() → signTransaction() → broadcastTransaction()
TRC20代币转账:    triggerSmartContract() → 签名 → broadcastHex()
                  function_selector: "transfer(address,uint256)"
                  合约地址: TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf (USDT on Nile Testnet)
```

### C.4 交易确认

```
TRON区块时间:     约3秒/块
交易确认:        19个确认块 ≈ 57秒
查询交易状态:     /wallet/gettransactionbyid → ret[0].contractRet = "SUCCESS"
```

### C.5 Gas费模型

```
TRX转账:          消耗Bandwidth，通常免费（新账户有1500 Bandwidth/天）
TRC20转账:        消耗Bandwidth + Energy
  Bandwidth消耗:  约345字节 ≈ 345 Bandwidth
  Energy消耗:     约29,000-65,000 Energy
  Energy单价:     约420 SUN/Energy (波动)
  总Gas费:        约14-31 TRX
fee_limit:       用户设置的最大Gas预算（SUN为单位）
                 50000000 SUN = 50 TRX（WalletTrxBusiness中的设置）
```

---

> **文档编号**：TMP-2（钱包参考B系列）
> **总行数**：约1200行
> **分析深度**：代码行号级佐证，每个结论可追溯到源文件:行号
> **覆盖维度**：架构/数据模型/接口/链路/安全/并发/缺陷/借鉴/对比/取舍 共16章+3附录
