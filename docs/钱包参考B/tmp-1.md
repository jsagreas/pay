# 参考钱包工程B深度解剖分析（TRC20虚拟币充值专项）

> 工程路径：/Users/mac/gitlab/z-tmp/tg-services
> 分析对象：wallet-server + user-server 虚拟币充值链路 + 关联服务
> 核心价值：**区块链 TRC20 USDT 充值完整链路**——从地址生成到链上归集到余额入账
> 分析深度：代码级逐行解剖，等同于"我从0到1实现了这个区块链钱包"
> 目标：为我们的 Go ser-wallet 虚拟币充值模块提供工程级参照

---

## 目录

- [第一章 技术栈与工程全景](#第一章-技术栈与工程全景)
- [第二章 钱包架构设计哲学](#第二章-钱包架构设计哲学)
- [第三章 数据模型与表结构](#第三章-数据模型与表结构)
- [第四章 区块链充值完整链路（核心）](#第四章-区块链充值完整链路核心)
- [第五章 地址生成与密钥管理](#第五章-地址生成与密钥管理)
- [第六章 链上交互层深度解析](#第六章-链上交互层深度解析)
- [第七章 归集机制（Collection/Sweep）](#第七章-归集机制collectionsweep)
- [第八章 提现链路分析](#第八章-提现链路分析)
- [第九章 外部签名系统](#第九章-外部签名系统)
- [第十章 定时任务与异步处理](#第十章-定时任务与异步处理)
- [第十一章 接口设计与API清单](#第十一章-接口设计与api清单)
- [第十二章 跨服务依赖与调用链路](#第十二章-跨服务依赖与调用链路)
- [第十三章 安全体系分析](#第十三章-安全体系分析)
- [第十四章 Bug清单与设计缺陷](#第十四章-bug清单与设计缺陷)
- [第十五章 取舍评价：精华与糟粕](#第十五章-取舍评价精华与糟粕)
- [第十六章 对我们Go实现的启示](#第十六章-对我们go实现的启示)
- [附录A 枚举值与常量映射表](#附录a-枚举值与常量映射表)
- [附录B 关键地址角色一览](#附录b-关键地址角色一览)
- [附录C 关键代码片段索引](#附录c-关键代码片段索引)

---

## 第一章 技术栈与工程全景

### 1.1 基础框架

| 维度 | 选型 | 版本/说明 |
|------|------|----------|
| 语言 | Java | 17 |
| 核心框架 | Spring Boot | 2.5.3 |
| ORM | Spring Data JPA | Hibernate, ddl-auto: update |
| 安全 | Spring Security | 完全禁用（全部 permitAll） |
| 数据库 | MySQL | InnoDB, database: cloud_wallet |
| 缓存 | Redis | Lettuce, database 11 |
| 分布式锁 | Redisson | 3.16.4 |
| 消息队列 | RabbitMQ | 声明但未实际使用MQ消费 |
| 区块链 | TRON (TRC20) | Nile 测试网 |
| 链交互 | gRPC 1.9.0 + HTTP API | TronGrid API |
| 密码学 | web3j 3.6.0 + BouncyCastle | EC 密钥对生成 |
| 密钥管理 | AWS KMS | ap-southeast-1 |
| 签名系统 | 外部独立服务 | RSA 加密通信 |
| 容器化 | Docker | OpenJDK 17 |

### 1.2 工程模块结构

```
tg-services/ (Java Maven 多模块)
├── pom.xml                       # 根 POM (13 个模块)
│
├── [公共库模块]
│   ├── module-common/            # 通用工具、枚举、VO、HTTP 客户端
│   ├── module-jjwt/              # JWT 工具
│   ├── module-authenticator/     # Google 2FA
│   ├── module-springcache-redis/ # Redis + Redisson
│   └── module-swagger2/          # Swagger 3.0
│
├── [业务服务模块]
│   ├── wallet-server/            # ★ 区块链钱包服务 (port: 9600)
│   ├── wallet-third-server/      # 三方担保钱包 (非区块链)
│   ├── user-server/              # ★ 用户服务 (port: 9300, 充值业务编排)
│   ├── game-server/              # 游戏服务
│   ├── authentication-server/    # 认证服务
│   ├── telegram-bot-api/         # ★ Telegram Bot 入口
│   ├── sms/                      # 短信服务
│   ├── file-upload-download/     # 文件服务 (MinIO)
│   └── casino/                   # 赌场模块 (4 子模块)
│       ├── casino-backend/
│       ├── casino-agentbackend/
│       ├── casino-web/           # ★ Web 网关
│       └── casino-core/
│
└── doc/                          # SQL 脚本、文档
```

### 1.3 wallet-server 内部结构

```
wallet-server/src/main/java/com/baisha/walletserver/
├── WalletServerApplication.java            # 启动类
├── controller/
│   ├── WalletController.java               # 6 个 REST 端点
│   └── TestController.java                 # 测试端点
├── business/
│   └── WalletTrxBusiness.java              # ★ 核心：地址生成、链上转账、签名 (538行)
├── driver/
│   ├── TrxDriver.java                      # ★ 编排层：归集、提现、余额路由 (338行)
│   └── AwsKmsDriver.java                   # AWS KMS 加解密 (110行)
├── client/trx/
│   ├── Trc20.java                          # TRC20 合约调用：balanceOf, transfer
│   ├── SmartContract.java                  # 智能合约 gRPC 调用
│   └── TronUtil.java                       # TRON gRPC 客户端、签名、广播
├── model/
│   ├── BaseEntity.java                     # JPA 基类
│   ├── TrxWalletEntity.java               # ★ 用户钱包表
│   ├── TrxWalletOrderEntity.java          # ★ 链上交易记录表
│   ├── TrxWalletCollectEntity.java        # 归集记录表
│   ├── TrxWalletConfigEntity.java         # 动态配置表
│   ├── enums/                              # 枚举
│   └── vo/                                 # 请求/响应 VO
├── repository/                             # JPA Repository
├── service/                                # 数据服务层
├── task/
│   └── CollectTrxJob.java                  # 每日 8:00 批量归集
└── util/                                   # AES, RSA, Base58, TronUtils, WalletUtil
```

**代码量统计：** wallet-server 核心代码约 1500 行（WalletTrxBusiness 538 + TrxDriver 338 + WalletController 144 + 其余服务/模型/工具约 500）

### 1.4 与参考工程A的本质区别

| 维度 | 参考工程A (p9) | 参考工程B (tg) | 差异 |
|------|---------------|---------------|------|
| 钱包核心 | 法币余额管理 | 区块链地址管理 | 根本不同 |
| 余额存储 | 本地 DB (user_coin) | 链上余额 (TRON) | 链上 vs 链下 |
| 充值方式 | 三方支付回调 | 链上 USDT 转入 | 技术栈完全不同 |
| 并发控制 | Redis 锁 + SQL WHERE | Redisson 锁 + 链上确认 | 链上操作天然串行 |
| 密钥管理 | 无（法币无私钥） | AWS KMS + AES 双层加密 | 新增维度 |

---

## 第二章 钱包架构设计哲学

### 2.1 核心架构模型：独立地址 + 归集

```
用户侧                    平台侧                     链上
┌────────────┐      ┌──────────────────────┐     ┌───────────────┐
│ 用户A       │      │    wallet-server      │     │   TRON 链      │
│ 地址: Txxx1 │─────→│ TrxWallet 表          │     │               │
│             │      │ (userId, address,     │     │  Txxx1 (用户A)  │
│ 用户B       │      │  enPrivateKey)        │     │  Txxx2 (用户B)  │
│ 地址: Txxx2 │─────→│                       │     │  Txxx3 (公司)   │
│             │      │ 归集: 用户地址 → 公司  │────→│  Txxx4 (出款)   │
│ ...         │      │ 出款: 出款地址 → 用户  │────→│  Txxx5 (Gas)   │
└────────────┘      └──────────────────────┘     └───────────────┘
```

**核心设计决策：每个用户一个独立的链上地址。**

这是区块链钱包的标准模式（HD Wallet 的简化版）：
- 优点：每个用户地址独立，可以精确追踪资金来源
- 优点：用户直接转账到自己的地址，不需要 memo/tag 区分
- 缺点：地址数量与用户数等同，管理复杂度高
- 缺点：每个地址的资金分散，需要归集机制

### 2.2 五类关键地址

```
┌─────────────────────────────────────────────────────────────────┐
│                     地址角色架构图                                │
│                                                                   │
│  ┌──────────────┐                                                │
│  │  用户地址      │ ← 用户充值 USDT 到这里                        │
│  │  (N个，每人1个)│                                               │
│  │  私钥: AES加密 │ ─→ 归集到公司地址                              │
│  └──────────────┘                                                │
│                           ↓                                      │
│  ┌──────────────┐                                                │
│  │  公司地址      │ ← 归集目标，资金汇聚地（冷钱包）               │
│  │  (companyAddr) │                                               │
│  └──────────────┘                                                │
│                                                                   │
│  ┌──────────────┐                                                │
│  │  出款地址      │ ← 提现源头，只有这个地址通过外部签名系统        │
│  │  (dispensing)  │ → 平台向用户出款                               │
│  └──────────────┘                                                │
│                                                                   │
│  ┌──────────────┐                                                │
│  │  Gas地址       │ ← 提供 TRX 作为 TRC20 转账的手续费            │
│  │  (gasAddress)  │ → 给用户地址/出款地址补 TRX                    │
│  └──────────────┘                                                │
│                                                                   │
│  公司地址/归集地址(mainCollectAddress) = companyAddress (同一个)   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 充值模型：声明式 + 余额验证 + 主动归集

**这不是标准的"链上扫块监听"模式，而是"声明金额 + 链上验证 + 主动归集"模式：**

```
标准模式（我们需要实现的）：
  扫块服务持续监听 → 发现转入交易 → 自动入账 → 异步归集

参考工程B的模式：
  用户在 TG 声明金额 → 系统查链上余额 → 余额 >= 声明金额 → 主动归集 → 入账
```

**这个差异极其关键——参考工程B没有区块链扫块服务。**

### 2.4 资金流全景

```
充值流：
  用户(外部钱包) ──USDT──→ 用户链上地址 ──归集──→ 公司地址(冷钱包)
                                                     ↓
                                              user-server 余额 +

提现流：
  user-server 余额 - ──→ 出款地址 ──USDT──→ 用户指定地址
                            ↑
                      外部签名系统签名

Gas 补充流：
  Gas地址 ──TRX──→ 用户地址/出款地址 (归集/提现前补充手续费)
```

---

## 第三章 数据模型与表结构

### 3.1 表清单

| 表名 | 所属 DB | 用途 | 行级增速 |
|------|--------|------|---------|
| TrxWallet | cloud_wallet | 用户链上钱包 | 低（每用户一条） |
| TrxWalletOrder | cloud_wallet | 链上交易记录 | 高（每笔交易一条） |
| trx_wallet_collect | cloud_wallet | 归集记录 | 中（每次归集一条） |
| TrxWalletConfig | cloud_wallet | 动态配置 | 极低（手动维护） |
| SsOrder | user-server DB | 充提订单 | 高 |

### 3.2 TrxWallet（用户钱包表）

```sql
CREATE TABLE TrxWallet (
    id             BIGINT AUTO_INCREMENT PRIMARY KEY,
    create_time    DATETIME,
    create_by      VARCHAR(255),
    update_time    DATETIME,
    update_by      VARCHAR(255),

    user_id        VARCHAR(10)   NOT NULL,   -- 用户ID
    user_name      VARCHAR(30),              -- 用户名
    en_private_key VARCHAR(250)  NOT NULL,   -- ★ AES 加密的私钥
    address        VARCHAR(100)  NOT NULL    -- TRON 地址 (T开头)
);

CREATE INDEX idx_user_id ON TrxWallet(user_id);
```

**关键设计：**
- 每个用户一行记录，一个 TRON 地址
- `en_private_key` 存储 AES 加密后的私钥（Base64 编码）
- 加密 Key = SHA1(KMS解密后的passphrase + address) 截取前 16 字节
- **不存余额** —— 余额实时从链上查询

**评价：**

| 要素 | 现状 | 评价 |
|------|------|------|
| 私钥存储 | AES 加密存 DB | 基本安全，但 AES-ECB 模式有缺陷（见安全章节） |
| 余额存储 | 不存，链上查 | 正确——避免链上/链下余额不一致 |
| 地址类型 | 只支持 TRON | 单链设计，扩展多链需重构 |
| userId | VARCHAR(10) | 长度太短，不利于扩展 |

### 3.3 TrxWalletOrder（链上交易记录表）

```sql
CREATE TABLE TrxWalletOrder (
    id             BIGINT AUTO_INCREMENT PRIMARY KEY,
    create_time    DATETIME,
    create_by      VARCHAR(255),
    update_time    DATETIME,
    update_by      VARCHAR(255),

    user_id        VARCHAR(10),
    tx_id          VARCHAR(200),    -- ★ 链上交易哈希
    ss_order_id    VARCHAR(200),    -- 业务订单号
    amount         DECIMAL(16,2),   -- 交易金额
    from_address   VARCHAR(250),    -- 发送方地址
    to_address     VARCHAR(250),    -- 接收方地址
    coin_name      VARCHAR(10),     -- 币种名称 (USDT/TRX/CXT)
    order_type     TINYINT(2),      -- 1=充值(RECHARGE) 2=提现(WITHDRAW)
    post_type      TINYINT(2)       -- 1=API调用 2=定时任务
);

CREATE INDEX idx_user_id ON TrxWalletOrder(user_id);
CREATE INDEX idx_tx_id ON TrxWalletOrder(tx_id);
CREATE INDEX idx_order_type ON TrxWalletOrder(order_type);
```

**关键设计：**
- `tx_id` 是链上交易哈希，唯一标识一笔链上交易
- `ss_order_id` 是业务系统的订单号，用于和 user-server 的 SsOrder 关联
- `order_type` 区分充值和提现
- `post_type` 区分是用户主动触发还是定时任务触发
- **金额精度 DECIMAL(16,2)**——只到小数点后 2 位，对于 USDT (6 位精度) 有精度丢失

**评价：**

| 要素 | 现状 | 问题 |
|------|------|------|
| tx_id 索引 | 有 | 好，链上查询需要 |
| 无 UNIQUE 约束 | ss_order_id 无唯一 | 重复记录无法防止 |
| 精度 | DECIMAL(16,2) | ★ 严重：USDT 精度 6 位，只存 2 位丢失精度 |
| 无状态字段 | 没有 status | 无法追踪交易确认状态 |

### 3.4 trx_wallet_collect（归集记录表）

```sql
CREATE TABLE trx_wallet_collect (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    create_time     DATETIME,
    create_by       VARCHAR(255),
    update_time     DATETIME,
    update_by       VARCHAR(255),

    user_id         VARCHAR(10),
    tx_id           VARCHAR(200),    -- 归集交易哈希
    collect_address VARCHAR(250),    -- 归集源地址
    amount          DECIMAL(16,2),   -- 归集金额
    collect_type    TINYINT(2)       -- 1=单笔归集 2=定时任务归集
);

CREATE INDEX idx_user_id ON trx_wallet_collect(user_id);
CREATE INDEX idx_tx_id ON trx_wallet_collect(tx_id);
CREATE INDEX idx_collect_type ON trx_wallet_collect(collect_type);
```

### 3.5 TrxWalletConfig（动态配置表）

```sql
CREATE TABLE TrxWalletConfig (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    type  VARCHAR(30),     -- 配置类型 (如 "sign")
    code  VARCHAR(50),     -- 配置键 (如 "rsa", "platCode", "disPendAddress")
    value VARCHAR(1000)    -- 配置值
);
```

当前存储的配置项：

| type | code | 用途 |
|------|------|------|
| sign | rsa | RSA 公钥，用于和签名系统通信 |
| sign | platCode | 平台代码，签名系统识别身份 |
| sign | disPendAddress | 出款地址（明文） |

### 3.6 SsOrder（user-server 的充提订单表）

```sql
-- 位于 user-server 数据库
CREATE TABLE SsOrder (
    id               BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_num        VARCHAR(64),       -- 订单编号
    user_id          BIGINT,
    tg_user_id       VARCHAR(64),       -- Telegram 用户ID
    nick_name        VARCHAR(64),
    amount           DECIMAL(16,2),     -- 订单金额
    real_amount      DECIMAL(16,2),     -- 实际金额
    order_type       INT,               -- 1=充值 2=提现
    order_status     INT,               -- 订单状态（见枚举）
    virtual_currency VARCHAR(16),       -- 虚拟币种 (USDT)
    tx_id            VARCHAR(200),      -- 链上交易ID
    payee_address    VARCHAR(250),      -- 收款地址（提现用）
    fee_amount       DECIMAL(16,2),     -- 手续费
    flow_multiple    DECIMAL(16,2),     -- 打码倍数
    complete_time    DATETIME,          -- 完成时间
    agent_id         BIGINT,            -- 代理ID
    agent_level      INT,               -- 代理层级
    user_type        INT,               -- 用户类型
    ...
);
```

### 3.7 ER 关系图

```
wallet-server DB (cloud_wallet)              user-server DB
┌─────────────────┐                          ┌─────────────────┐
│   TrxWallet     │                          │    SsOrder       │
│                 │     1:N (by userId)      │                  │
│ userId ─────────│──────────────────────────│──userId          │
│ address         │                          │ orderNum         │
│ enPrivateKey    │                          │ orderStatus      │
└────────┬────────┘                          │ virtualCurrency  │
         │ 1:N (by userId/address)           │ txId ────────────│──┐
         ▼                                   │ realAmount       │  │
┌─────────────────┐                          └─────────────────┘  │
│ TrxWalletOrder  │                                                │
│                 │←── txId 关联 ──────────────────────────────────┘
│ txId            │     (wallet-server 的 txId 回写到 SsOrder)
│ ssOrderId       │←── ssOrderId = SsOrder.id
│ amount          │
│ orderType       │
└─────────────────┘
```

---

## 第四章 区块链充值完整链路（核心）

### 4.1 链路全景图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        TRC20 USDT 充值完整链路                            │
│                                                                          │
│  Phase 1: 地址分配                                                       │
│  ┌──────────┐    ┌───────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ TG Bot    │───→│ casino-web │───→│ wallet-server │───→│ user-server  │  │
│  │ 点击充值  │    │  路由      │    │ /wallet/create│    │ 回写地址     │  │
│  └──────────┘    └───────────┘    │ 生成密钥对     │    └──────────────┘  │
│                                    │ AES加密私钥   │                      │
│                                    │ 存DB+返回地址 │                      │
│                                    └──────────────┘                      │
│                                                                          │
│  Phase 2: 用户链上转账（链外，不受系统控制）                               │
│  用户从自己的外部钱包，将 USDT 转到 Phase 1 分配的地址                     │
│                                                                          │
│  Phase 3: 声明金额 + 创建订单                                            │
│  ┌──────────┐    ┌───────────┐    ┌──────────────┐                      │
│  │ TG Bot    │───→│ casino-web │───→│ user-server  │                      │
│  │ 输入金额  │    │            │    │ 创建SsOrder  │                      │
│  │ 点击确认  │    │            │    │ status=WAIT  │                      │
│  └──────────┘    └───────────┘    └──────────────┘                      │
│                                                                          │
│  Phase 4: 链上余额验证 + 归集                                            │
│  ┌──────────────┐    ┌──────────────┐    ┌────────────┐                 │
│  │ user-server   │───→│ wallet-server │───→│ TRON 链     │                 │
│  │ 查余额        │    │ /getBalance   │    │ balanceOf() │                 │
│  │ 余额>=声明额 │    │              │    └────────────┘                 │
│  │               │    │              │                                    │
│  │ 触发归集      │───→│ /collectTrc20 │                                    │
│  │               │    │ ①补Gas(TRX) │───→ Gas地址 → 用户地址 (30 TRX)   │
│  │               │    │ ②归集(USDT) │───→ 用户地址 → 公司地址 (全额)    │
│  │               │    │ 返回 txId    │                                    │
│  └──────────────┘    └──────────────┘                                    │
│                                                                          │
│  Phase 5: 链上确认 + 入账（定时任务或再次点击）                            │
│  ┌──────────────┐    ┌──────────────┐    ┌────────────┐                 │
│  │ user-server   │───→│ wallet-server │───→│ TRON 链     │                 │
│  │ 定时2min轮询 │    │ /getTransInfo │    │ getTxById() │                 │
│  │               │    │              │    │ ret=SUCCESS │                 │
│  │ 确认成功      │    └──────────────┘    └────────────┘                 │
│  │ ①订单SUCCESS  │                                                       │
│  │ ②增加余额    │                                                       │
│  │ ③增加打码量  │                                                       │
│  │ ④通知统计    │                                                       │
│  └──────────────┘                                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Phase 1: 地址分配（createWallet）

**入口：** `POST /wallet/create`
**核心代码：** `WalletTrxBusiness.createWallet()` (行 99-136)

```
步骤详解：

① 幂等检查
   trxWalletService.findAddressByUserId(userId)
   → 如果已有地址，直接返回（不重复创建）

② 生成密钥对
   ECKeyPair ecKeyPair = Keys.createEcKeyPair()
   → 使用 web3j 的 EC 密钥对生成（secp256k1 曲线）
   → 私钥：256位随机数
   → 公钥：从私钥推导

③ 推导 TRON 地址
   WalletFile walletFile = Wallet.createStandard(passphrase, ecKeyPair)
   String originAddress = walletFile.getAddress()
   finalAddress = fromHexAddress("41" + originAddress)
   → "41" 是 TRON 地址前缀（区别于以太坊的 "0x"）
   → fromHexAddress 将 hex 转 Base58Check (T 开头)

④ 加密私钥
   keyStore = Numeric.toHexStringNoPrefix(ecKeyPair.getPrivateKey())
   enKey = encrypt(keyStore, newPassphrase + finalAddress)
   → 加密算法：AES-ECB-PKCS5Padding
   → 加密密钥：SHA1(passphrase + address) 截取前 16 字节
   → passphrase 通过 AWS KMS 解密获得

⑤ 持久化
   trxWalletService.save(userId, userName, enKey, finalAddress)
   → 存入 TrxWallet 表

⑥ 异步通知 user-server
   trxWalletService.updateUserAddress(trxUserWalletRequest)
   → HTTP POST to user-server /user/updateWalletAddressById
   → 将地址回写到用户表
```

### 4.3 Phase 2: 用户链上转账

这一步完全在系统控制范围之外：
- 用户打开自己的外部钱包（如 TronLink、Trust Wallet）
- 将 USDT-TRC20 转到系统分配的地址
- 链上确认（通常 1-20 个区块，约 3-60 秒）

**关键点：系统不知道用户何时转账、转了多少。** 这就是为什么需要 Phase 3 的"声明金额"。

### 4.4 Phase 3: 声明金额 + 创建订单

用户在 Telegram 输入充值金额：

```
① TG Bot 接收金额输入
   TelegramMessageHandler → 检测 Redis 标记 CHECKOUT_COUNTER_PLEASE_ENTER_RECHARGE_AMOUNT

② 创建 SsOrder
   → casino-web → user-server
   orderType = CHARGE_ORDER (1)
   orderStatus = VIRTUAL_CURRENCY_WAIT
   virtualCurrency = "USDT"
   realAmount = 用户输入的金额

③ TG Bot 展示确认按钮
   → 存 orderId 到 Redis
```

### 4.5 Phase 4: 余额验证 + 归集（核心链路）

用户点击"确认"按钮后的完整链路：

```java
// user-server: AsyncVirtualDepositService.checkCrypotcurrencyOrder() 行 96-188

// Step A: 查链上余额
balance = walletBusiness.getBalance(userId, virtualCurrency);
// → HTTP POST wallet-server /wallet/getBalance
// → wallet-server 查 TrxWallet 表获取地址
// → 调 TRON API triggerConstantContract(balanceOf) 获取链上 USDT 余额

// Step B: 余额验证
if (balance == null || balance == 0 || balance < realAmount) {
    return "余额不足";
}

// Step C: 触发链上归集
txId = walletBusiness.getTransferResult(virtualCurrency, orderId, userId, realAmount, postType);
// → HTTP POST wallet-server /wallet/collectTrc20
```

**wallet-server 归集执行：** `TrxDriver.singleCollect()` (行 175-228)

```
① 解密公司地址
   newMainAddress = awsKmsDriver.decryptData(companyAddress)

② 检查用户地址余额
   如果 reChargeAmount 已传入则使用，否则链上查询

③ Gas 补充（关键步骤！）
   if (币种是TRC20 而非 TRX原生币):
       查用户地址的 TRX 余额
       if (TRX < 30):
           从 Gas 地址转 30 TRX 到用户地址（支付后续 TRC20 转账手续费）
           → createTransactionForTrx(gasPassword, 30, gasAddress, userAddress, "trx", ssOrderId)
           → 记录这笔 TRX Gas 转账到 TrxWalletOrder

④ 执行归集转账
   createTransactionForTrx(null, amount, userAddress, companyAddress, usdtContract, ssOrderId)
   → 内部解密用户私钥
   → 调 TRON API /wallet/triggersmartcontract 构建 TRC20 transfer 交易
   → 本地签名（用户地址归集不走外部签名系统）
   → 调 TRON API /wallet/broadcasthex 广播
   → 返回 txId

⑤ 记录归集信息
   reverseNoticeCollectDetail(userId, fromAddress, txId, amount, collectType, ssOrderId, coinName, postType)
   → 写 trx_wallet_collect 表
   → 写 TrxWalletOrder 表
```

### 4.6 Phase 5: 链上确认 + 入账

归集交易广播后，链上确认需要时间（TRON 通常 3s）。系统通过两种方式检查确认：

**方式1：用户再次点击确认**（triggerByTg = true）
**方式2：定时任务每 2 分钟轮询**（triggerByTg = false）

```java
// user-server: AsyncVirtualDepositService.checkCrypotcurrencyOrder() 第二轮 (txId 已存在)

// Step A: 查交易状态
TransactionVO transactionVO = walletBusiness.getTransactionInfoById(String.valueOf(ssOrder.getId()));
// → wallet-server 通过 ssOrderId 查 TrxWalletOrder 获取 txId
// → 调 TRON API /wallet/gettransactionbyid
// → 检查 ret.contractRet == "SUCCESS"

// Step B: 确认成功 → 入账
if (confirmed) {
    updateOrderStatusAndSendMQ(orderId, userId, virtualCurrency, txId);
}
```

**入账逻辑：** `updateOrderStatusAndSendMQ()` (行 258-344)

```
① 更新订单状态
   ssOrderService.updateOrderStatusAndCompleteTime(
       OrderStatusEnum.ORDER_SUCCESS, completeTime, txId,
       orderId, userId, virtualCurrency,
       OrderStatusEnum.VIRTUAL_CURRENCY_WAIT  // WHERE 条件，CAS防并发
   )
   → 返回 affected rows，0 = 已被处理（幂等）

② 处理体验用户转正
   if (userType == PRACTICE):
       清空体验金 → 转为正式用户

③ 增加余额
   addBalance(ssOrder, user, realAmount)
   → userAssetsBusiness.doBalanceBusiness(user, balanceVO)
   → BalanceChangeEnum.TELEGRAM_CHARGE, BalanceTypeEnum.INCOME

④ 增加打码量
   playMoney = realAmount × flowMultiple
   doPlayMoney(ssOrder, user, playMoney)

⑤ 发送统计通知
   sendMQ(...) → 实际是 HTTP POST 到 backend-server（MQ 注释掉了）

⑥ 通知 TG（仅定时任务触发时）
   notifyWallet(currentBalance, orderId, orderType, txId)
   → HTTP POST 到 casino-web → TG Bot 发消息给用户
```

### 4.7 充值链路时序图

```
时间轴 ↓

用户         TG Bot       casino-web    user-server    wallet-server     TRON链
 │            │              │              │              │               │
 ├──点击充值─→│              │              │              │               │
 │            ├──get addr──→│──────────────│───create───→│               │
 │            │              │              │              ├──ECKeyPair──→│
 │            │              │              │              │←─address─────│
 │            │              │              │←─updateAddr──│               │
 │            │←──展示地址───│              │              │               │
 │            │              │              │              │               │
 │──链上转USDT──────────────────────────────────────────────────────────→│
 │            │              │              │              │               │
 ├──输入金额─→│              │              │              │               │
 │            ├──createOrder─│─────────────→│(SsOrder)     │               │
 │            │←─确认按钮────│              │              │               │
 │            │              │              │              │               │
 ├──点击确认─→│              │              │              │               │
 │            ├──charge────→│─────────────→│              │               │
 │            │              │              ├──getBalance──→│──balanceOf──→│
 │            │              │              │              │←──余额────────│
 │            │              │              │              │               │
 │            │              │              ├──collect────→│               │
 │            │              │              │              ├──补Gas(TRX)──→│
 │            │              │              │              ├──归集(USDT)──→│
 │            │              │              │←──txId───────│               │
 │            │              │              │              │               │
 │            │              │              │  (等待链上确认 ~3s)           │
 │            │              │              │              │               │
 │            │      [2min定时任务]          │              │               │
 │            │              │              ├──getTxInfo──→│──getTxById──→│
 │            │              │              │              │←──SUCCESS─────│
 │            │              │              │              │               │
 │            │              │              │──入账(余额+)  │               │
 │            │              │              │──订单SUCCESS  │               │
 │            │←──充值成功──│──────────────│              │               │
 │            │              │              │              │               │
```

---

## 第五章 地址生成与密钥管理

### 5.1 密钥生成流程

```java
// WalletTrxBusiness.createWallet() 行 116-123

// 1. 生成 EC 密钥对（secp256k1 曲线）
ECKeyPair ecKeyPair = Keys.createEcKeyPair();
// → web3j 底层使用 BouncyCastle 的 SecureRandom

// 2. 创建标准钱包文件（衍生地址）
WalletFile walletFile = Wallet.createStandard(passphrase, ecKeyPair);
String originAddress = walletFile.getAddress();
// → originAddress 是以太坊格式的地址（40 hex 字符）

// 3. 转换为 TRON 地址
String finalAddress = fromHexAddress("41" + originAddress);
// → "41" 前缀标识 TRON 主网
// → fromHexAddress: hex → byte[] → Base58Check → "T"开头地址

// 4. 提取私钥明文
String keyStore = Numeric.toHexStringNoPrefix(ecKeyPair.getPrivateKey());
// → 64 hex 字符的私钥

// 5. 加密私钥
String enKey = encrypt(keyStore, passphrase + finalAddress);
// → AES-ECB-PKCS5Padding 加密
// → 密钥 = SHA1(passphrase + address) 截取前16字节
```

### 5.2 私钥加密方案

```
┌─────────────────────────────────────────────────────┐
│                私钥加密链路                            │
│                                                       │
│  AWS KMS                                             │
│  ┌──────────┐                                        │
│  │ Master Key│→ 解密 → passphrase (助记词)           │
│  └──────────┘            │                            │
│                           ▼                            │
│  密钥推导: SHA1(passphrase + address) → 截取前16字节  │
│                           │                            │
│                           ▼                            │
│  AES-ECB-PKCS5: encrypt(privateKey, derivedKey)      │
│                           │                            │
│                           ▼                            │
│  enPrivateKey → 存入 TrxWallet.en_private_key         │
└─────────────────────────────────────────────────────┘

解密时（归集操作需要签名）:
  ① AWS KMS 解密 passphrase
  ② SHA1(passphrase + address) → 截取前16字节
  ③ AES-ECB 解密 → 得到私钥明文
  ④ 用私钥签名交易
```

### 5.3 安全评价

| 要素 | 现状 | 评价 |
|------|------|------|
| 密钥生成 | web3j ECKeyPair (SecureRandom) | 安全，行业标准 |
| 密钥加密算法 | AES-ECB | ★ 不安全：ECB 模式可被模式分析攻击，应用 CBC 或 GCM |
| 加密密钥推导 | SHA1 截取16字节 | ★ 弱：SHA1 已被攻破，应用 PBKDF2 或 Argon2 |
| passphrase 保护 | AWS KMS | 好：KMS 硬件安全模块保护 |
| 私钥存储位置 | 数据库 | 可接受：热钱包必须在线存储，KMS 加密提供了一层保护 |
| 地址推导 | "41" + ETH地址 → Base58Check | 正确的 TRON 地址推导方式 |

---

## 第六章 链上交互层深度解析

### 6.1 TRC20 转账实现

**核心方法：** `WalletTrxBusiness.trc20Transaction()` (行 209-290)

```
完整步骤：

① 地址验证
   validateAddress(to)
   → TRON API POST /wallet/validateaddress
   → 检查 Base58check 格式

② 合约验证
   tokenContract 必须以 "T" 开头（TRC20 合约地址）

③ 构建 transfer 调用参数
   function_selector: "transfer(address,uint256)"
   parameter: encoderAbi(toHexAddress, amount * 10^decimals)
   → web3j FunctionEncoder 编码 ABI
   owner_address: toHexAddress(from)
   fee_limit: 50,000,000 (50 TRX，实际约 14 TRX 手续费)

④ 调用 TRON API 构建交易
   POST trxUrl + "/wallet/triggersmartcontract"
   → 返回未签名交易的 JSON

⑤ 打包交易 (protobuf)
   TronUtils.packTransaction(transaction.toJSONString(), false)
   → JSON → Protocol.Transaction (protobuf 反序列化)

⑥ 签名（分两种路径）
   if (from == 出款地址):
       bytes = sendSign(signUrl, tx, ssOrderId, from)  // 外部签名系统
   else:
       bytes = signTransaction2Byte(tx.toByteArray(), privateKey)  // 本地签名

⑦ 广播
   POST trxUrl + "/wallet/broadcasthex"
   → body: { "transaction": hexSignedTransaction }
   → 检查 result.code == "SUCCESS"
   → 获取 txId

⑧ 返回 txId
```

### 6.2 TRX 原生转账实现

**核心方法：** `WalletTrxBusiness.trxOfflineSignature()` (行 323-362)

```
① Base58Check 解码地址
   byte[] fromAddress = WalletApi.decodeFromBase58Check(from)
   byte[] toHexAddress = WalletApi.decodeFromBase58Check(to)

② 金额转换
   amount = needSendAmount * 10^6 (TRX 精度为 6)

③ 创建原始交易
   Protocol.Transaction transaction = createTransaction(fromAddress, toHexAddress, amount)
   → 静态导入 TransactionSignDemo.createTransaction
   → 使用 gRPC 客户端构建 TransferContract

④ 外部签名（TRX 原生转账始终用外部签名系统）
   byte[] bytes = sendSign(signUrl, transaction, ssOrderId, from)

⑤ 广播
   boolean success = WalletApi.broadcastTransaction(bytes)
   → gRPC 广播
```

### 6.3 余额查询

**TRX 余额：**
```java
// WalletTrxBusiness.getBalanceByAddress() 行 381-406
POST trxUrl + "/wallet/getaccount"
→ body: { "address": hexAddress }
→ result.balance / 10^6
```

**TRC20 余额：**
```java
// Trc20.balanceOf() → SmartContract.triggerConstantContract()
// gRPC 调用: triggerConstantContract
// method: "balanceOf(address)"
// 解码 Uint256 → / 10^decimals
```

### 6.4 交易确认查询

```java
// WalletTrxBusiness.getTransactionInfoById() 行 517-537
POST trxUrl + "/wallet/gettransactionbyid"
→ body: { "value": txId }
→ result.ret[0].contractRet == "SUCCESS" → 确认成功
```

---

## 第七章 归集机制（Collection/Sweep）

### 7.1 归集的本质

归集（Collection/Sweep）= 将分散在各用户地址上的资金，统一转移到公司冷钱包地址。

**为什么需要归集：**
1. 用户地址是热钱包（私钥在线），安全性低
2. 资金分散在 N 个地址上，管理困难
3. 公司地址（冷钱包）可以离线保管私钥，安全性高
4. 提现出款从统一的出款地址执行，与用户地址分离

### 7.2 单笔归集（实时，充值触发）

**入口：** `POST /wallet/collectTrc20`
**核心代码：** `TrxDriver.singleCollect()` (行 175-228)

```
触发时机：用户点击确认充值时

步骤：
① 解密公司地址（KMS）
② 检查用户地址余额（链上查询）
③ 如果余额 <= 0，返回失败
④ TRC20 转账需要 TRX 作为 Gas：
   → 查用户地址 TRX 余额
   → 如果 < 30 TRX：从 Gas 地址转 30 TRX 到用户地址
   → 等待 Gas 到账（实际上没有等待确认！★问题）
⑤ 执行 TRC20 转账：用户地址 → 公司地址
⑥ 记录归集表 + 订单表
⑦ 返回 txId
```

**关键代码片段 (TrxDriver.java 行 191-210):**

```java
// Gas 补充
if (!TRXContractEnum.TRX.getContract().equalsIgnoreCase(tokenContract)) {
    BigDecimal addTrx = getAccountBalance(fromAddress, TRXContractEnum.TRX.getContract());
    if (FEE_BAL.compareTo(addTrx) > 0) {  // FEE_BAL = 30 TRX
        trxWalletResult = createTransactionForTrx(
            awsKmsDriver.decryptData(gasPassword),
            FEE_BAL,
            newGasAddress, fromAddress,
            TRXContractEnum.TRX.getContract(), ssOrderId);
        if(StringUtils.isBlank(trxWalletResult.getTxId())){
            return null;
        }
        // ★ 没有等待 Gas 交易确认就继续执行归集！
        trxWalletOrderService.save(...); // 记录 Gas 转账
    }
}

// 执行归集
trxWalletResult = createTransactionForTrx(null, trxBalance, fromAddress,
        newMainAddress, tokenContract, ssOrderId);
```

### 7.3 批量归集（定时，每日执行）

**入口：** `CollectTrxJob.collectUSDT()` (每日 08:00 Bangkok 时间)
**核心代码：** `TrxDriver.collectTrc20()` (行 230-255)

```java
@Scheduled(cron = "0 0 8 * * ?", zone="Asia/Bangkok")
public void collectUSDT(){
    TrcCollectRequest trcCollectRequest = new TrcCollectRequest();
    trcCollectRequest.setCoinName(TRXContractEnum.USDT.getContractName());
    trxDriver.collectTrc20(trcCollectRequest);
}
```

**执行逻辑：**
```
① 获取 TrxWallet 表所有记录
② 遍历每个用户钱包：
   → 跳过公司地址本身
   → 为每个用户生成 UUID 作为 ssOrderId
   → 调用 singleCollect()（不传 reChargeAmount → 归集全部余额）
   → collectType = COLLECT(2) 标识为定时任务归集
```

### 7.4 归集模式评价

| 维度 | 现状 | 评价 |
|------|------|------|
| Gas 补充时机 | 归集前检查并补充 | 合理，TRC20 转账需要 TRX |
| Gas 确认等待 | ★ 不等待确认 | 严重：Gas 未到账就执行归集，可能失败 |
| 归集金额 | 全额归集 | 简单但浪费 Gas（小额也归集） |
| 批量归集 | 串行遍历，每个用户一笔 | ★ 性能差，用户多时耗时长 |
| 归集失败处理 | log.error 后继续 | 没有重试机制 |
| 归集记录 | 双表记录（collect + order） | 好，可审计 |

---

## 第八章 提现链路分析

### 8.1 提现流程

**入口：** `POST /wallet/withdrawTraction`
**核心代码：** `TrxDriver.withdrawTraction()` (行 281-337)

```
① 获取出款地址（从 DB TrxWalletConfig 查询）
② 幂等检查：通过 ssOrderId 查 TrxWalletOrder
   → 如果已存在，直接返回已有的 txId（幂等）
③ 余额检查：查出款地址的链上余额
   → if (sendAmount > balance) 返回失败
④ Gas 补充（同归集逻辑）
   → TRC20 提现前检查出款地址 TRX 余额
⑤ 执行转账
   createTransactionForTrx(fromAddress, sendAmount, fromAddress, toAddress, tokenContract, ssOrderId)
   → ★ 注意 privateKey 传入的是 fromAddress（出款地址字符串而非私钥！）
   → 实际上走外部签名系统路径（因为 from == dispensingAddress）
⑥ 记录订单
   trxWalletOrderService.save(...)
```

### 8.2 提现与归集的签名差异

```
归集签名（用户地址 → 公司地址）：
  → 本地签名：解密用户私钥 → signTransaction2Byte()
  → 原因：用户私钥在我们的数据库中

提现签名（出款地址 → 用户地址）：
  → 外部签名系统：sendSign() → /api/admin/chain/sign
  → 原因：出款地址的私钥在独立的签名系统中，安全隔离
```

### 8.3 提现幂等设计

```java
// TrxDriver.withdrawTraction() 行 298-303
TrxWalletOrderEntity order = trxWalletOrderRepository.findBySsOrderId(ssOrderId);
if(ObjectUtil.isNotEmpty(order)){
    result = new TrxWalletResult();
    result.setTxId(order.getTxId());
    return result;  // 已存在则直接返回
}
```

**这是一个简单但有效的幂等设计：** 通过 ssOrderId 查已有记录，存在则返回。

**缺陷：** ssOrderId 在 DB 上没有 UNIQUE 约束，并发场景下可能写入两条。

---

## 第九章 外部签名系统

### 9.1 签名系统架构

```
┌────────────────┐                     ┌────────────────────┐
│  wallet-server  │      RSA加密        │   签名系统           │
│                 │ ──────────────────→ │   (独立部署)          │
│  ① 构建交易     │  POST /api/admin/   │                      │
│  ② SHA256(rawData)│  chain/sign       │  ③ RSA解密请求        │
│  ③ Base64编码    │                    │  ④ 验证 platCode      │
│  ④ RSA加密请求   │  Header:           │  ⑤ 用私钥签名 hash    │
│     {orderNo,    │   PLAT_CODE=xxx    │  ⑥ Base64编码返回     │
│      outAddressNo,│                   │                      │
│      chainType,  │  ←──────────────── │  Response:           │
│      hashId}     │                    │  {code:200, data:签名}│
│  ⑤ 组装签名+广播 │                    │                      │
└────────────────┘                     └────────────────────┘
```

### 9.2 签名请求构建

```java
// WalletTrxBusiness.sendSign() 行 292-321

// ① 计算交易哈希
byte[] txId = Sha256Sm3Hash.hash(tx.getRawData().toByteArray());
String signTxid = Base64.getEncoder().encodeToString(txId);

// ② 构建签名请求
Map<String, Object> signMap = new HashMap<>();
signMap.put("orderNo", ssOrderId);        // 业务订单号
signMap.put("outAddressNo", from);         // 出款地址
signMap.put("chainType", "TRON");          // 链类型
signMap.put("hashId", signTxid);           // 交易哈希(Base64)

// ③ RSA 加密请求体
String enSign = RSAUtil.encrypt(JSONObject.toJSONString(signMap), rsaPublicKey);

// ④ HTTP POST 发送
HttpResponse response = HttpRequest.post(signUrl + "/api/admin/chain/sign")
    .header("PLAT_CODE", platCode)     // 平台身份标识
    .body(enSign)
    .timeout(60000)                     // 60秒超时
    .execute();

// ⑤ 解析响应
if("200".equals(jsonObject.get("code"))){
    deSign = Base64.getDecoder().decode(jsonObject.getString("data"));
}

// ⑥ 组装完整签名交易
return tx.toBuilder().addSignature(ByteString.copyFrom(deSign)).build().toByteArray();
```

### 9.3 签名系统设计评价

| 维度 | 评价 |
|------|------|
| 私钥隔离 | **优秀**：出款地址私钥不在 wallet-server 中，物理隔离 |
| 通信安全 | RSA 加密请求体，防止中间人窃取交易哈希 |
| 身份验证 | PLAT_CODE 头部标识，但缺少签名/HMAC |
| 审计追踪 | 传入 orderNo，签名系统可记录审计日志 |
| 超时处理 | 60s 超时，合理 |
| 故障恢复 | 签名失败返回 null → 上层逻辑返回失败 |

**我们的借鉴：出款地址的私钥必须和业务服务物理隔离，独立签名系统是正确架构。**

---

## 第十章 定时任务与异步处理

### 10.1 定时任务清单

| 服务 | 类 | Cron | 用途 |
|------|-----|------|------|
| wallet-server | CollectTrxJob | 0 0 8 * * ? (Bangkok) | 每日 8AM 批量归集 USDT |
| user-server | HandleVirtualCurrencyTask | 0 */2 * * * ? (Bangkok) | 每 2 分钟处理充值订单 |

### 10.2 充值定时任务详解

```java
// HandleVirtualCurrencyTask.checkVirtualCurrencyTask()
// 每 2 分钟执行：

// 1. 处理超时订单（>30分钟）→ 标记为系统取消
//    UPDATE SsOrder SET orderStatus = ORDER_SYSTEM_CANCLE
//    WHERE createTime < (now - 30min)
//      AND orderStatus = VIRTUAL_CURRENCY_WAIT

// 2. 处理待处理订单（<30分钟）→ 异步检查每个订单
//    for each 待处理订单:
//        asyncVirtualDepositService.deposit(orderId, "USDT", false)
//        → @Async 异步执行
//        → Redisson fairLock 防并发
//        → tryLock(0s, 10s) 抢不到直接跳过（下次再来）
```

### 10.3 异步处理机制

```
user-server 异步处理充值：
  ┌─────────────────────────────────────────┐
  │  @Async + Redisson FairLock             │
  │                                          │
  │  fairLock = redisson.getFairLock(key)    │
  │  res = fairLock.tryLock(0, 10, SECONDS)  │
  │  → 0秒等待 = 立即返回                    │
  │  → 10秒租约 = 自动释放                   │
  │                                          │
  │  if (!res) return;  // 有人在处理，跳过   │
  │                                          │
  │  // 检查订单状态仍是 WAIT               │
  │  // 执行 checkCrypotcurrencyOrder       │
  └─────────────────────────────────────────┘
```

**评价：**
- FairLock(0s, 10s) 是好的设计：不阻塞、不堆积、自动释放
- @Async 确保不阻塞定时任务主线程
- 问题：10s 租约可能不够（链上交互可能超过 10s），锁提前释放导致并发

---

## 第十一章 接口设计与API清单

### 11.1 wallet-server REST API

基路径：`/wallet`，端口：9600

| # | 方法 | 路径 | 功能 | 请求体 | 响应 |
|---|------|------|------|--------|------|
| 1 | POST | /wallet/create | 生成钱包地址 | TrxUserWalletRequest(userId,userName) | WalletAddressVO(address) |
| 2 | POST | /wallet/withdrawTraction | TRC20提现 | TrxCreateTransactionRequest | TrxWalletResult(txId) |
| 3 | POST | /wallet/getBalance | 查链上余额 | UserBalanceRequest(userId,currency) | BigDecimal |
| 4 | POST | /wallet/collectTrc20 | 单笔归集 | TrcCollectRequest | TrxWalletResult(txId) |
| 5 | POST | /wallet/getTransactionInfoById | 查交易状态 | @RequestParam orderId | TransactionVO(txId,result) |
| 6 | POST | /wallet/getTrxWalletAccount | 加密钱包信息 | TrxWalletAccountRequest | TrxWalletAccountVO |

### 11.2 user-server 虚拟币相关 API

| # | 路径 | 功能 |
|---|------|------|
| 1 | POST /virtual/charge | 虚拟币充值（TG确认按钮触发） |
| 2 | POST /virtual/withdraw | 虚拟币提现 |
| 3 | POST /user/updateWalletAddressById | 回写钱包地址到用户表 |

---

## 第十二章 跨服务依赖与调用链路

### 12.1 服务调用图

```
telegram-bot-api
    │
    ├──HTTP──→ casino-web (路由/网关)
    │              │
    │              ├──HTTP──→ user-server (充值业务编排)
    │              │              │
    │              │              ├──HTTP──→ wallet-server (链上操作)
    │              │              │              │
    │              │              │              ├──HTTP──→ TRON API (链上查询/广播)
    │              │              │              │
    │              │              │              └──HTTP──→ 签名系统 (提现签名)
    │              │              │
    │              │              └──HTTP──→ backend-server (统计通知)
    │              │
    │              └──HTTP──→ wallet-server (地址生成)
    │
    └──HTTP──→ user-server (确认充值)
```

### 12.2 通信方式

**全部是 HTTP 同步调用，没有 RPC，没有 MQ（MQ 声明但注释掉了）。**

| 调用方 | 被调方 | 工具 | 超时 |
|-------|--------|------|------|
| telegram-bot → casino-web | casino-web | HttpClient4Util | 默认 |
| user-server → wallet-server | wallet-server | HttpClient4Util | 默认 |
| wallet-server → TRON API | TronGrid | HttpClient4Util / gRPC | 默认 |
| wallet-server → 签名系统 | 签名系统 | Hutool HttpRequest | 60s |
| wallet-server → user-server | user-server | RestTemplate | 默认 |

### 12.3 与我们的对比

我们的 Go 架构用 Kitex RPC 替代所有 HTTP 调用，延迟更低、类型更安全。区块链交互部分可以保留 HTTP/gRPC（TRON API 不支持 Kitex）。

---

## 第十三章 安全体系分析

### 13.1 安全层级

```
┌──────────────────────────────────────────────┐
│              安全层级架构                       │
│                                                │
│  L1: AWS KMS (硬件安全模块)                    │
│      → 保护 passphrase, companyAddress,        │
│        gasPrivateKey 等核心机密                 │
│                                                │
│  L2: AES 加密 (软件层)                         │
│      → 保护用户私钥                             │
│      → Key = SHA1(KMS_passphrase + address)    │
│                                                │
│  L3: 外部签名系统 (物理隔离)                    │
│      → 出款私钥不在业务服务中                    │
│      → RSA 加密通信                             │
│                                                │
│  L4: Spring Security (完全禁用)                 │
│      → ★ 安全隐患：无认证无授权                 │
│                                                │
│  L5: DB 配置表 (TrxWalletConfig)               │
│      → 出款地址、RSA公钥、平台代码              │
│      → 数据库级别保护                            │
└──────────────────────────────────────────────┘
```

### 13.2 安全问题清单

| # | 级别 | 问题 | 影响 |
|---|------|------|------|
| S1 | 严重 | AES-ECB 模式 | ECB 不使用 IV，相同明文产生相同密文，可被模式分析 |
| S2 | 严重 | SHA1 作为 KDF | SHA1 已被攻破，应使用 PBKDF2/Argon2 |
| S3 | 严重 | Spring Security 完全禁用 | 任何人可调用 wallet-server API |
| S4 | 高 | application.yml 明文 AWS 凭证 | AccessKey/SecretKey 在配置文件中 |
| S5 | 高 | 签名系统仅 PLAT_CODE 头验证 | 无签名/HMAC，可伪造请求 |
| S6 | 中 | 私钥解密后在内存中明文 | 无安全内存擦除 |

---

## 第十四章 Bug清单与设计缺陷

### 14.1 确认的 Bug

| # | 严重度 | 位置 | 描述 |
|---|--------|------|------|
| B1 | **严重** | TrxDriver:196-204 | Gas 补充后不等待链上确认就执行归集。Gas 未到账时归集交易会因手续费不足失败 |
| B2 | **严重** | TrxWalletOrder | DECIMAL(16,2) 精度。USDT 有 6 位小数精度，只存 2 位丢失精度 |
| B3 | **高** | TrxDriver:329 | withdrawTraction 传 fromAddress 字符串作为 privateKey 参数。虽然内部有 null 检查兜底，但语义混乱 |
| B4 | **高** | WalletTrxBusiness:265 | trc20Transaction 广播返回的 code 判断：`StringUtils.isBlank(isSuc) || !"SUCCESS".equalsIgnoreCase(isSuc)` — 成功时 TRON API 返回的 `code` 字段可能为空（成功时没有 code），导致误判为失败 |
| B5 | **中** | TrxDriver:182 | 批量归集时 gasAddress 来源不一致：collectTrc20 用 `trxWalletConfigRepository.findByType` 查 DB，但 gasAddress 实际用的是 `awsKmsDriver.decryptData(gasAddress)` 的 KMS 解密值。出款地址和 gas 地址来源混乱 |
| B6 | **中** | AsyncVirtualDepositService:199 | Redisson fairLock 租约 10s，但链上交互可能超过 10s，锁提前释放导致并发处理同一订单 |
| B7 | **低** | CollectTrxJob | 批量归集串行执行，大量用户时耗时过长 |

### 14.2 设计缺陷

| # | 严重度 | 类别 | 描述 |
|---|--------|------|------|
| D1 | **致命** | 架构 | 无区块链扫块监听服务。用户转账后系统不知道，必须用户声明金额 → 不适合自动化场景 |
| D2 | **严重** | 安全 | AES-ECB 加密私钥，SHA1 推导密钥。行业标准应为 AES-GCM + PBKDF2 |
| D3 | **严重** | 安全 | Spring Security 完全禁用，wallet-server API 无认证 |
| D4 | **高** | 数据 | TrxWalletOrder 无状态字段（pending/confirmed/failed），只能查链上确认 |
| D5 | **高** | 数据 | TrxWalletOrder 的 ss_order_id 无 UNIQUE 约束，幂等靠查询而非约束 |
| D6 | **高** | 架构 | 全部 HTTP 同步调用，链上交互慢（3-60s）时阻塞调用链 |
| D7 | **高** | 精度 | DECIMAL(16,2) 存储 USDT 金额，应为 DECIMAL(20,6) 或更高 |
| D8 | **中** | 扩展 | 只支持 TRON 链，合约地址硬编码/YML 配置，不支持多链 |
| D9 | **中** | 性能 | 批量归集串行执行，无并发控制 |
| D10 | **中** | 监控 | 无交易状态监控，归集失败只 log 不告警 |
| D11 | **低** | 代码 | WalletTrxBusiness 538 行巨型类，职责不清 |

---

## 第十五章 取舍评价：精华与糟粕

### 15.1 精华（值得借鉴）

#### 精华1：独立地址模式（每用户一地址）

**借鉴价值：9/10**

这是区块链钱包的正确做法：
- 每个用户有独立的链上地址
- 资金来源可精确追溯
- 不需要 memo/tag 区分用户
- 我们的 TRC20 充值必须采用这个模式

#### 精华2：私钥 KMS + AES 双层加密

**借鉴价值：8/10**（方案正确，具体算法需改进）

思路正确：
- 用户私钥不能明文存储
- AWS KMS 保护主密钥（passphrase）
- AES 加密用户私钥，密钥派生自 passphrase + address
- 每个地址的加密密钥不同（address 作为盐）

我们的改进：
- AES-ECB → AES-GCM（带 IV + 认证标签）
- SHA1 KDF → PBKDF2 或 Argon2
- 考虑 HashiCorp Vault 替代 AWS KMS（自建更灵活）

#### 精华3：出款地址 + 外部签名系统隔离

**借鉴价值：9/10**

核心安全原则：
- 出款地址的私钥不在业务服务中
- 独立的签名系统物理隔离
- RSA 加密通信防止中间人
- 审计追踪（orderNo 传入签名系统）

我们必须采用这个架构。

#### 精华4：Gas 地址自动补充机制

**借鉴价值：8/10**

TRC20 转账需要 TRX 作为手续费，参考工程自动处理：
- 归集/提现前检查目标地址的 TRX 余额
- 不足时从 Gas 地址自动补充 30 TRX
- 这个机制在 ERC20/TRC20 等代币转账中是必需的

需改进：补充后等待确认再执行主交易。

#### 精华5：归集模式（用户地址 → 公司地址）

**借鉴价值：8/10**

归集的核心思想：
- 用户地址是热钱包（在线），安全性低
- 公司地址是冷钱包（可离线），安全性高
- 资金及时归集减少热钱包暴露风险
- 双重归集：实时（充值时）+ 定时（每日兜底）

#### 精华6：提现幂等（通过 ssOrderId 查重）

**借鉴价值：6/10**（思路对，实现需加强）

查重避免重复出款，但缺少 DB UNIQUE 约束。

### 15.2 糟粕（必须避免）

#### 糟粕1：无区块链扫块监听（致命）

**最大缺陷。** 没有扫块服务意味着：
- 系统不知道用户何时转账
- 必须用户手动声明金额
- 如果用户多转了，多余部分直到每日归集才处理
- 无法支持自动化充值

**我们的方案：** 必须实现区块链扫块监听服务，实时检测链上转入交易并自动入账。

#### 糟粕2：Gas 补充不等待确认（严重）

补充 TRX Gas 后立即执行归集/提现，Gas 交易可能还未确认，导致后续交易失败。

**我们的方案：** Gas 补充后必须等待确认（轮询或事件回调），确认后再执行主交易。

#### 糟粕3：金额精度丢失 DECIMAL(16,2)（严重）

USDT 有 6 位小数精度，只存 2 位意味着：
- 1.123456 USDT 被截断为 1.12 USDT
- 累积丢失可导致资金不平

**我们的方案：** DECIMAL(20,8) 或更高精度。

#### 糟粕4：AES-ECB + SHA1 加密（严重）

ECB 模式和 SHA1 都有已知的安全缺陷。

**我们的方案：** AES-GCM + PBKDF2/Argon2。

#### 糟粕5：无 API 认证（严重）

wallet-server 的 API 完全无认证，任何人可以调用 createWallet、withdrawTraction 等敏感接口。

**我们的方案：** 服务间通信必须有认证（Kitex 中间件 + 签名验证）。

#### 糟粕6：链上交易无状态管理（高）

TrxWalletOrder 没有 status 字段，无法追踪 pending → confirmed → failed 的状态流转。

**我们的方案：** 链上交易记录必须有状态字段 + 状态机管理。

### 15.3 取舍总结

```
                    实现质量
                    高         低
              ┌───────────┬───────────┐
    价值  高  │ 独立地址模式│ 私钥加密   │
              │ 外部签名系统│ (方案对算法弱)│
              │ Gas自动补充│           │
              │ 归集模式   │           │
              ├───────────┼───────────┤
          低  │ 提现幂等   │ 无扫块监听  │
              │           │ Gas不等确认 │
              │           │ 精度丢失    │
              │           │ 无API认证  │
              └───────────┴───────────┘
```

---

## 第十六章 对我们Go实现的启示

### 16.1 区块链充值架构设计要点

基于参考工程B的分析，我们的 Go 实现需要：

```
┌─────────────────────────────────────────────────────────┐
│            我们的区块链充值架构（改进版）                   │
│                                                           │
│  [参考工程B有的] + [参考工程B没有的]                       │
│                                                           │
│  有的（借鉴）：                                           │
│  ✓ 每用户独立链上地址                                     │
│  ✓ 私钥加密存储（改进算法）                                │
│  ✓ 出款地址 + 外部签名系统隔离                             │
│  ✓ Gas 自动补充（改进：等待确认）                          │
│  ✓ 归集机制（实时 + 定时兜底）                             │
│  ✓ 提现幂等（改进：DB UNIQUE 约束）                       │
│                                                           │
│  没有的（新增）：                                         │
│  ★ 区块链扫块监听服务（核心差距）                          │
│  ★ 交易状态机（pending → confirmed → failed）             │
│  ★ 链上确认数管理（TRON: 19确认 ≈ 57s 才算最终确认）      │
│  ★ Gas 补充确认等待                                       │
│  ★ 多链支持架构（TRC20 + ERC20 + ...）                   │
│  ★ 充值自动入账（不需要用户声明金额）                      │
│  ★ API 认证（Kitex 中间件）                               │
│  ★ 高精度金额（DECIMAL(20,8) + shopspring/decimal）       │
└─────────────────────────────────────────────────────────┘
```

### 16.2 我们需要额外设计的核心组件

| 组件 | 参考工程B | 我们需要 | 说明 |
|------|----------|---------|------|
| 扫块监听服务 | 无 | ★ 必需 | 实时检测链上转入交易 |
| 地址分配服务 | 有（简单） | 优化 | 预生成地址池，避免实时生成延迟 |
| 归集服务 | 有（基础） | 增强 | 阈值归集、Gas 等待确认、并行归集 |
| 签名服务 | 有（外部） | 自建或复用 | 出款私钥隔离 |
| 交易状态追踪 | 无 | ★ 必需 | pending/confirming/confirmed/failed |
| 充值回调 | 无 | ★ 必需 | 扫块检测到转入后自动触发入账 |
| 提现审批流 | 无（直接出款） | 需要 | 大额提现人工审批 |

### 16.3 区块链充值的理想架构

```
用户转 USDT 到分配地址
         │
         ▼
┌─────────────────────┐
│  区块链扫块监听服务   │ ← 持续监听新区块
│  (独立部署)           │
│                       │
│  检测到 Transfer 事件 │
│  to = 我们的用户地址  │
│  amount = X USDT     │
│                       │
│  等待 N 个确认        │
│  (TRON: 19 blocks)   │
│                       │
│  确认后：             │
│  ① 创建充值记录       │
│  ② RPC 通知 ser-wallet│
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│    ser-wallet        │
│                      │
│  接收充值通知：       │
│  ① 验证交易有效性    │
│  ② 幂等检查 (txHash) │
│  ③ 增加用户余额      │
│  ④ 写充值记录        │
│  ⑤ 触发归集（阈值）  │
└──────────────────────┘
```

### 16.4 关键设计决策对照表

| 决策点 | 参考工程B方案 | 我们的建议 | 原因 |
|--------|-------------|-----------|------|
| 充值检测 | 用户声明+余额查询 | 扫块监听自动检测 | 自动化、准确 |
| 地址生成 | 实时生成 | 预生成地址池 | 避免延迟 |
| 私钥加密 | AES-ECB + SHA1 | AES-GCM + PBKDF2 | 安全性 |
| 密钥管理 | AWS KMS | HashiCorp Vault 或自建 | 灵活性 |
| Gas 补充 | 补充后不等确认 | 补充后等待确认 | 可靠性 |
| 归集触发 | 充值时+每日定时 | 阈值触发+定时兜底 | 节省Gas |
| 签名方式 | 外部签名系统 | 外部签名系统 | 一致 |
| 金额精度 | DECIMAL(16,2) | DECIMAL(20,8) | 精度 |
| 交易状态 | 无状态字段 | 状态机管理 | 可追踪 |
| 确认数 | 广播后立即返回 | 等待19确认 | 防双花 |
| 通信方式 | HTTP 同步 | Kitex RPC | 性能 |

---

## 附录A 枚举值与常量映射表

### A.1 TRXContractEnum（支持的币种）

| 枚举 | Code | 合约 | 名称 | 精度 | 类型 |
|------|------|------|------|------|------|
| TRX | 1 | "trx" | TRX | 6 | 原生币 |
| USDT | 2 | (YML配置) | USDT | 6 | TRC20 |
| CXT | 3 | TBfSX66Sp... | CXT | 18 | TRC20 |

### A.2 TrxOrderTypeEnum（订单类型）

| Code | Name | 含义 |
|------|------|------|
| 1 | RECHARGE | 充值（归集入账） |
| 2 | WITHDRAW | 提现（出款） |

### A.3 CollectTypeEnum（归集类型）

| Code | Name | 含义 |
|------|------|------|
| 1 | SINGLE | 单笔即时归集（充值触发） |
| 2 | COLLECT | 定时任务归集（每日批量） |

### A.4 OrderStatusEnum（订单状态，user-server）

| 状态 | 含义 |
|------|------|
| VIRTUAL_CURRENCY_WAIT | 待处理（充值已声明，等待链上确认） |
| ORDER_SUCCESS | 成功（已入账） |
| ORDER_SYSTEM_CANCLE | 系统取消（超时30分钟） |

### A.5 关键常量

| 常量 | 值 | 用途 |
|------|-----|------|
| FEE_BAL | 30 TRX | Gas 补充阈值 |
| fee_limit (TRC20) | 50,000,000 SUN (50 TRX) | TRC20 交易最大Gas |
| 充值超时 | 30 分钟 | 超时自动取消订单 |
| 定时任务频率 | 每 2 分钟 | 充值轮询周期 |
| 批量归集时间 | 每日 08:00 Bangkok | USDT 全量归集 |
| Redisson 锁租约 | 10 秒 | 充值订单处理锁 |

---

## 附录B 关键地址角色一览

| 角色 | 配置来源 | 加密方式 | 私钥位置 | 用途 |
|------|---------|---------|---------|------|
| 用户地址 | TrxWallet 表 | AES(KMS_passphrase + addr) | wallet-server DB | 接收用户充值 |
| 公司地址 | YML wallet.trx.companyAddress | AWS KMS | KMS | 归集目标（冷钱包） |
| 归集地址 | YML wallet.trx.mainCollectAddress | AWS KMS | KMS | = 公司地址 |
| 出款地址 | DB TrxWalletConfig | 明文存 DB | 外部签名系统 | 提现出款源 |
| Gas地址 | YML wallet.trx.gasAddress | AWS KMS | KMS | 提供 TRX 手续费 |

---

## 附录C 关键代码片段索引

### C.1 核心文件索引

| 文件 | 服务 | 行数 | 核心作用 |
|------|------|------|---------|
| WalletTrxBusiness.java | wallet-server | 538 | 地址生成、链上转账、签名、余额查询 |
| TrxDriver.java | wallet-server | 338 | 归集、提现编排、Gas补充 |
| WalletController.java | wallet-server | 144 | 6个REST端点 |
| AwsKmsDriver.java | wallet-server | 110 | AWS KMS 加解密 |
| Trc20.java | wallet-server | ~100 | TRC20 合约调用 |
| TronUtil.java | wallet-server | ~100 | gRPC 客户端、签名 |
| TronUtils.java | wallet-server | ~200 | 交易打包、离线签名 |
| AsyncVirtualDepositService.java | user-server | 454 | 充值业务编排 |
| VirtualCurrencyBusiness.java | user-server | 248 | 定时任务编排 |
| WalletBusiness.java | user-server | 234 | HTTP 调用 wallet-server |
| HandleVirtualCurrencyTask.java | user-server | 67 | 每2min充值轮询 |
| CollectTrxJob.java | wallet-server | 39 | 每日8AM批量归集 |

### C.2 关键行号速查

| 功能 | 文件 | 行号 |
|------|------|------|
| 地址生成 (createWallet) | WalletTrxBusiness.java | 99-136 |
| AES 加密私钥 | WalletTrxBusiness.java | 183-202 |
| TRC20 转账 | WalletTrxBusiness.java | 209-290 |
| TRX 原生转账 | WalletTrxBusiness.java | 323-362 |
| 外部签名调用 | WalletTrxBusiness.java | 292-321 |
| TRX 余额查询 | WalletTrxBusiness.java | 381-406 |
| TRC20 余额查询 | WalletTrxBusiness.java | 415-439 |
| 链上确认查询 | WalletTrxBusiness.java | 517-537 |
| 单笔归集 | TrxDriver.java | 175-228 |
| 批量归集 | TrxDriver.java | 230-255 |
| 提现执行 | TrxDriver.java | 281-337 |
| 提现幂等检查 | TrxDriver.java | 298-303 |
| Gas 补充 | TrxDriver.java | 191-210 |
| 充值链路入口 | AsyncVirtualDepositService.java | 96-188 |
| 余额入账 | AsyncVirtualDepositService.java | 258-344 |
| 定时任务轮询 | HandleVirtualCurrencyTask.java | 55-64 |
| 超时订单取消 | VirtualCurrencyBusiness.java | 98-118 |

---

> **文档总结**
>
> 这个参考钱包工程B是一个**功能可用但架构不完整的区块链钱包实现**。它的核心价值在于展示了 TRC20 虚拟币充值的完整代码级链路：地址生成 → 私钥加密 → 链上余额查询 → Gas 补充 → 归集转账 → 链上确认 → 余额入账。
>
> **最大的架构缺陷是没有区块链扫块监听服务**——系统不能自动检测链上充值，必须依赖用户声明金额。这在我们的实现中是必须补齐的核心组件。
>
> **最大的安全贡献是外部签名系统的隔离设计**——出款私钥不在业务服务中，通过独立的签名系统进行物理隔离。这是我们必须采用的架构模式。
>
> 总体定位：**概念验证级别的参考实现**，证明了 TRC20 充提的技术可行性，提供了完整的代码级参照，但在安全性、可靠性、自动化程度上都需要我们大幅改进。我们应当"借其骨架（地址模型+归集模型+签名隔离），补其灵魂（扫块监听+状态机+安全加密+高精度+多链扩展）"。
