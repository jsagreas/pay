# Java 虚拟币钱包工程深度分析（生产参考 B — TRC20 充提实现）

> 分析时间：2026-02-27
> 分析对象：/Users/mac/gitlab/z-tmp/tg-services（Java 生产工程，Spring Boot 微服务体系）
> 核心模块：wallet-server（区块链钱包服务，35+ 个 Java 文件，~8000 行）
> 关键模块：user-server（充提订单管理 + 钱包集成，含 5 个核心业务类 ~1800 行）
> 分析范围：wallet-server 全量 + user-server 充提链路 + 跨模块集成 + 配置体系
> 核心价值：**TRC20 虚拟币充值/提现/归集的完整生产实现**——这是我们需求中最难自研的部分
> 分析依据：完整源码逐文件阅读（Controller / Business / Driver / Service / Entity / Enum / Util / Config / Client / Task，共 50+ 文件）

---

## 零、分析结论速览（先给结论，后看推导）

```
┌─────────────────────┬──────────────────────────────────────────────────────────────────────┐
│ 维度                 │ 结论                                                                 │
├─────────────────────┼──────────────────────────────────────────────────────────────────────┤
│ 核心能力             │ TRON 链上钱包生成 + TRC20 USDT 充值归集 + TRC20 提现 + 链上查询         │
│                     │ → ✅ 完整的虚拟币充提生产方案，最高参考价值                               │
│ 地址模型             │ 4 类地址（用户热钱包 / 归集地址 / 出款地址 / Gas 地址）                   │
│                     │ → ✅ 典型的 HD 分离架构，可直接采纳                                     │
│ 加密体系             │ 3 层加密（AWS KMS / AES-256 / RSA 外部签名）                           │
│                     │ → ✅ 私钥安全方案值得借鉴，但需适配我们的基础设施                          │
│ 充值流程             │ 链上余额轮询 + 归集到主账户 + 定时任务                                  │
│                     │ → ⚠️ 轮询模式延迟高（分钟级），可改进为 Event 监听                       │
│ 提现流程             │ 出款地址 → 外部签名服务 → 广播上链 → 轮询确认                           │
│                     │ → ✅ 双重签名模式值得采纳（应用不持有出款私钥）                            │
│ Gas 管理             │ 自动检测 TRX 余额 + 自动充 Gas                                        │
│                     │ → ✅ TRC20 交易必需 TRX 作为手续费，此自动管理方案可直接复用               │
│ 数据模型             │ 4 张表（钱包 / 订单 / 归集 / 配置）+ user-server 的 SsOrder             │
│                     │ → 我们可以在现有 14 张表基础上扩展 2~3 张表覆盖链上操作                    │
│ 已知缺陷             │ 无链上 Event 监听、明文日志泄露风险、定时任务归集效率低                    │
│                     │ → 正好是我们可以改进的点                                                │
└─────────────────────┴──────────────────────────────────────────────────────────────────────┘
```

**与模块参考 A 的定位差异**：

```
模块参考 A（p9 wallet-service）→ 聚焦「法币余额操作」：余额加减 / 冻结解冻 / 账变流水 / 并发控制
模块参考 B（tg wallet-server） → 聚焦「虚拟币链上操作」：地址生成 / 链上转账 / 归集 / 私钥管理 / 签名

两者互补，共同构成完整的钱包系统参考：
  模块 A 回答「余额怎么管」
  模块 B 回答「币怎么充、怎么提」
```

---

## 一、工程全景与技术栈

### 1.1 项目模块结构

```
/Users/mac/gitlab/z-tmp/tg-services/
├── pom.xml                            ← 根 POM（Spring Boot 2.5.3, Java 17, 13 模块）
├── module-common/                     ← 公共库（VO / 常量 / 工具）
├── module-jjwt/                       ← JWT 认证
├── module-authenticator/              ← 二次验证
├── module-springcache-redis/          ← Redis 缓存
├── module-swagger2/                   ← API 文档
│
├── wallet-server/                     ← 🎯 区块链钱包服务（核心分析对象）
├── user-server/                       ← 🎯 用户服务（充提订单 + 钱包集成）
├── game-server/                       ← 游戏服务
├── casino/                            ← 赌场后台（含 admin 钱包管理）
├── telegram-bot-api/                  ← Telegram Bot 接入
├── authentication-server/             ← 认证服务
├── file-upload-download/              ← 文件服务
├── sms/                               ← 短信服务
└── wallet-third-server/               ← 三方钱包服务（疑似另一个支付通道）
```

### 1.2 技术栈清单

| 技术                     | 版本/说明          | 用途                    | 我们的对应方案             |
|--------------------------|-------------------|------------------------|--------------------------|
| Spring Boot              | 2.5.3             | 服务框架                 | CloudWeGo Hertz + Kitex  |
| Spring Data JPA          | Hibernate         | ORM + DDL-auto          | GORM Gen                 |
| web3j                    | 3.6.0 + 4.5.16    | 以太坊/TRON ABI 编码     | Go 版 go-web3 或自实现    |
| TRON wallet-cli          | 1.0 (自定义 JAR)  | TRON 协议/gRPC 通信      | Go 版 go-tron-sdk        |
| gRPC                     | -                 | TRON 节点通信            | Go 原生 gRPC              |
| AWS KMS                  | SDK v2            | 配置加密/密钥管理         | 阿里云 KMS 或自建 Vault   |
| RabbitMQ                 | -                 | 消息队列                 | 待定（Kafka/NSQ）         |
| Redis                    | Lettuce           | 缓存 + 分布式锁          | go-redis                 |
| Protobuf                 | -                 | TRON 交易序列化          | Go 原生 protobuf          |
| BouncyCastle             | -                 | 加密算法库               | Go crypto 标准库          |

---

## 二、wallet-server 模块剖析

### 2.1 文件结构全景

```
wallet-server/src/main/java/com/baisha/walletserver/
│
├── WalletServerApplication.java          ← Spring Boot 启动类
│
├── controller/
│   ├── WalletController.java             ← 144行 | 6 个 API（创建/提现/余额/归集/查询/配置）
│   └── TestController.java               ← 测试工具（KMS 加密测试）
│
├── business/
│   └── WalletTrxBusiness.java            ← 538行 | ⭐ 核心区块链逻辑（钱包生成/TRC20转账/TRX转账/余额查询）
│
├── driver/
│   ├── TrxDriver.java                    ← 338行 | ⭐ 业务编排层（归集/提现/余额/交易创建）
│   └── AwsKmsDriver.java                 ← 110行 | AWS KMS 加解密驱动
│
├── service/
│   ├── TrxWalletService.java             ← 81行  | 钱包 CRUD + 私钥解密 + 异步回调
│   ├── TrxWalletOrderService.java        ← -     | 订单记录 CRUD
│   └── TrxWalletCollectService.java      ← -     | 归集记录 CRUD
│
├── client/trx/
│   ├── SmartContract.java                ← 91行  | ⭐ 智能合约调用（只读查询 + 交易触发）
│   ├── Trc20.java                        ← 125行 | TRC20 代币操作（余额/精度/转账）
│   └── TronUtil.java                     ← -     | gRPC 客户端封装
│
├── model/
│   ├── TrxWalletEntity.java              ← 实体 | 用户热钱包（地址 + 加密私钥）
│   ├── TrxWalletOrderEntity.java         ← 实体 | 交易订单（txId + 金额 + 方向）
│   ├── TrxWalletCollectEntity.java       ← 实体 | 归集记录
│   ├── TrxWalletConfigEntity.java        ← 实体 | 动态配置（签名密钥/出款地址）
│   ├── BaseEntity.java                   ← 基类 | id + createTime + updateTime + audit
│   ├── enums/
│   │   ├── TRXContractEnum.java          ← 代币枚举（TRX/USDT/CXT + 合约地址 + 精度）
│   │   ├── TrxOrderTypeEnum.java         ← 订单类型（充值=1 / 提现=2）
│   │   └── CollectTypeEnum.java          ← 归集类型（单笔即时=1 / 定时批量=2）
│   ├── vo/request/                       ← 6 个请求 VO
│   └── vo/response/                      ← 4 个响应 VO
│
├── repository/
│   ├── TrxWalletRepository.java          ← JPA | 钱包查询（按地址/按用户）
│   ├── TrxWalletOrderRepository.java     ← JPA | 订单查询（按时间/按 ssOrderId）
│   ├── TrxWalletCollectRepository.java   ← JPA | 归集记录
│   └── TrxWalletConfigRepository.java    ← JPA | 配置查询（按 type）
│
├── task/
│   └── CollectTrxJob.java                ← 41行 | 定时归集（每日 8:00 曼谷时间）
│
├── config/
│   └── (RestTemplate / Security 等)
│
└── util/
    ├── AES.java                          ← AES/ECB/PKCS5Padding 加解密
    ├── RSAUtil.java                      ← RSA 512 加解密
    ├── Base58.java                       ← TRON 地址编解码
    ├── TronUtils.java                    ← 474行 | ⭐ TRON 协议工具（地址转换/交易打包/签名/广播）
    ├── ByteArray.java                    ← 字节数组工具
    ├── Sha256Hash.java                   ← SHA-256 哈希
    └── WalletUtil.java                   ← YAML 配置读取
```

### 2.2 API 接口全览

```
WalletController（6 个接口）
─────────────────────────────────────────────────────────────────
POST /wallet/create                    生成用户热钱包地址
POST /wallet/withdrawTraction          TRC20/TRX 提现
POST /wallet/getBalance                查询链上余额
POST /wallet/collectTrc20              单笔 TRC20 归集
POST /wallet/getTransactionInfoById    查询交易状态
POST /wallet/getTrxWalletAccount       查询加密钱包配置（B 端管理用）
```

---

## 三、四类地址架构（核心设计精华）

### 3.1 地址角色定义

```
┌──────────────────────┬────────────────────────────────┬─────────────────────────────────┐
│ 地址类型              │ 存储位置                        │ 用途                             │
├──────────────────────┼────────────────────────────────┼─────────────────────────────────┤
│ 用户热钱包            │ TrxWallet 表                   │ 每用户独立充值地址                │
│ (User Hot Wallet)    │ address + enPrivateKey         │ 用户向此地址转币 = 充值            │
│                      │                                │ 私钥加密存储（AES-256）           │
├──────────────────────┼────────────────────────────────┼─────────────────────────────────┤
│ 归集/公司地址         │ application.yml (KMS 加密)      │ 所有用户充值归集到此地址           │
│ (Company Address)    │ wallet.trx.companyAddress      │ = 公司资金池                     │
│                      │ wallet.trx.mainCollectAddress  │ 归集操作的目标地址                │
├──────────────────────┼────────────────────────────────┼─────────────────────────────────┤
│ 出款地址              │ TrxWalletConfig 表             │ 提现时从此地址向用户转币           │
│ (Dispensing Address) │ code="disPendAddress"          │ 私钥由外部签名服务管理            │
├──────────────────────┼────────────────────────────────┼─────────────────────────────────┤
│ Gas 地址              │ application.yml (KMS 加密)      │ 为 TRC20 交易支付 TRX 手续费     │
│ (Gas Account)        │ wallet.trx.gasAddress          │ 私钥本地加密存储                  │
│                      │ wallet.trx.gasPrivateKey       │ 自动充值 30 TRX 给需要的地址      │
└──────────────────────┴────────────────────────────────┴─────────────────────────────────┘
```

### 3.2 资金流转全景

```
                     充值流程
  用户 TRON 钱包                          公司资金池
  ─────────────                          ─────────
      │                                     ▲
      │ 1.转币到                             │ 3.归集
      ▼                                     │
  ┌──────────────┐    2.检测余额     ┌──────────────┐
  │ 用户热钱包     │ ──────────────→  │  归集操作     │
  │ (每人独立)     │                  │  (Gas 补充)   │
  └──────────────┘                  └──────────────┘
                                         │
                                    Gas 地址 补充
                                    TRX 手续费
                                         │
                                         ▼

                     提现流程
  公司出款地址                              用户提供的
  ─────────────                          收款地址
      │                                     ▲
      │ 1.构建交易                           │ 3.链上转账
      ▼                                     │
  ┌──────────────┐    2.外部签名     ┌──────────────┐
  │ 签名服务      │ ←──────────────  │  广播交易     │
  │ (RSA 加密)    │ ──────────────→  │  (/broadcasthex)│
  └──────────────┘                  └──────────────┘
```

**✅ 可直接采纳**：四类地址分离架构是虚拟币钱包的标准设计：

```
1. 用户热钱包——一人一地址，便于充值识别
2. 归集地址——统一管理，降低资金分散风险
3. 出款地址——与归集分离，私钥不在应用持有
4. Gas 地址——独立管理手续费，避免污染业务资金
```

---

## 四、钱包生成流程（createWallet）

### 4.1 源码剖析

```java
// WalletTrxBusiness.java:99-136
public WalletAddressVO createWallet(TrxUserWalletRequest request) {
    String userId = request.getUserId();

    // 1. 幂等检查：已有钱包直接返回
    String address = trxWalletService.findAddressByUserId(userId);
    if (StringUtils.isNotBlank(address)) {
        return existingWallet;
    }

    // 2. 生成密钥对（web3j）
    ECKeyPair ecKeyPair = Keys.createEcKeyPair();
    WalletFile walletFile = Wallet.createStandard(passphrase, ecKeyPair);

    // 3. 地址转换：以太坊格式 → TRON 格式
    String originAddress = walletFile.getAddress();       // 以太坊 hex 地址
    String finalAddress = fromHexAddress("41" + originAddress);  // 加 41 前缀 → Base58Check

    // 4. 私钥加密存储
    String privateKeyHex = Numeric.toHexStringNoPrefix(ecKeyPair.getPrivateKey());
    String encryptedKey = encrypt(privateKeyHex, passphrase + finalAddress);
    //    加密密钥 = SHA-1(passphrase + address) 截取前 16 字节

    // 5. 持久化
    trxWalletService.save(userId, userName, encryptedKey, finalAddress);

    // 6. 异步通知用户服务
    trxWalletService.updateUserAddress(request);  // @Async

    return walletAddressVO;
}
```

### 4.2 地址转换原理

```
以太坊地址（20 字节 hex）：
  e65a42065454dbf97b5aca0cf96b31ab562e5438

加上 TRON 前缀 0x41（1 字节）：
  41e65a42065454dbf97b5aca0cf96b31ab562e5438

Base58Check 编码（加校验位）：
  TUyKqdNXNEzAvjiBWQJ3ooQFJoSYUAqLrR

转换函数链：
  ECKeyPair → WalletFile.getAddress() → "41" + hex → Base58Check encode → T 开头地址
```

### 4.3 私钥加密方案

```
加密算法：AES/ECB/PKCS5Padding
密钥派生：SHA-1(passphrase + address)，截取前 16 字节作为 AES Key

存储：
  DB 字段：TrxWallet.enPrivateKey = AES.encrypt(privateKeyHex, passphrase + address)

解密：
  使用时：privateKey = AES.decrypt(enPrivateKey, passphrase + address)
  passphrase 来源：application.yml → AWS KMS 解密后获取

安全性分析：
  ✅ 私钥不明文存储
  ✅ passphrase 由 KMS 管理，不在代码/配置中明文出现
  ⚠️ AES/ECB 模式安全性不如 CBC/GCM
  ⚠️ SHA-1 已不推荐用于密钥派生（应使用 PBKDF2/Argon2）
```

**✅ Go 项目采纳建议**：

```go
// 采纳私钥加密存储思路，但改进加密算法
// 使用 AES-256-GCM + Argon2id 密钥派生

func EncryptPrivateKey(privateKey []byte, passphrase string, address string) ([]byte, error) {
    // 1. 密钥派生：Argon2id(passphrase + address)
    salt := sha256.Sum256([]byte(passphrase + address))
    key := argon2.IDKey([]byte(passphrase), salt[:], 1, 64*1024, 4, 32)

    // 2. AES-256-GCM 加密
    block, _ := aes.NewCipher(key)
    gcm, _ := cipher.NewGCM(block)
    nonce := make([]byte, gcm.NonceSize())
    io.ReadFull(rand.Reader, nonce)
    return gcm.Seal(nonce, nonce, privateKey, nil), nil
}
```

---

## 五、TRC20 充值归集流程（核心价值 — 完整链路）

### 5.1 充值全链路时序图

```
┌──────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 用户  │    │user-server│    │wallet-svc│    │TRON 链   │    │归集地址   │
└──┬───┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
   │             │              │              │              │
   │ 1.获取充值地址│              │              │              │
   │─────────────→│              │              │              │
   │             │ 2.查询钱包   │              │              │
   │             │──────────────→│              │              │
   │             │  返回 address │              │              │
   │             │←──────────────│              │              │
   │ 3.展示地址  │              │              │              │
   │←────────────│              │              │              │
   │             │              │              │              │
   │ 4.用户转币到│              │              │              │
   │  该地址     │              │              │              │
   │─────────────────────────────────────────→│              │
   │             │              │              │ 链上确认     │
   │             │              │              │              │
   │ 5.TG Bot   │              │              │              │
   │  输入金额   │              │              │              │
   │─────────────→│              │              │              │
   │             │ 6.创建充值订单│              │              │
   │             │ SsOrder(wait)│              │              │
   │             │              │              │              │
   │             │ 7.定时任务(5min)              │              │
   │             │──────────────→│              │              │
   │             │ getBalance   │ 8.查链上余额 │              │
   │             │              │──────────────→│              │
   │             │              │  返回余额     │              │
   │             │              │←──────────────│              │
   │             │ 9.余额>=金额 │              │              │
   │             │              │              │              │
   │             │ 10.collectTrc20              │              │
   │             │──────────────→│              │              │
   │             │              │ 11.检查Gas   │              │
   │             │              │──────────────→│              │
   │             │              │  TRX余额<30  │              │
   │             │              │              │              │
   │             │              │ 12.Gas补充    │              │
   │             │              │  gasAddr→用户 │              │
   │             │              │──────────────→│              │
   │             │              │              │              │
   │             │              │ 13.TRC20归集  │              │
   │             │              │  用户→归集地址│              │
   │             │              │──────────────→│──────────────→│
   │             │              │  返回 txId    │              │
   │             │              │←──────────────│              │
   │             │ 14.txId      │              │              │
   │             │←──────────────│              │              │
   │             │              │              │              │
   │             │ 15.更新订单   │              │              │
   │             │ status=成功  │              │              │
   │             │ 加余额到资产 │              │              │
   │             │ 加打码量     │              │              │
   │ 16.通知成功 │              │              │              │
   │←────────────│              │              │              │
```

### 5.2 充值归集核心代码解析

```java
// TrxDriver.java:175-228 — singleCollect（单笔归集）
public TrxWalletResult singleCollect(String fromAddress, String userId,
        String coinName, String ssOrderId, BigDecimal reChargeAmount,
        int collectCode, int postType) {

    BigDecimal trxBalance = reChargeAmount;
    String mainAddr = awsKmsDriver.decryptData(companyAddress);  // KMS 解密归集目标

    // 1. 如果未传金额，查链上真实余额
    if (null == reChargeAmount) {
        trxBalance = getAccountBalance(fromAddress, coinName);
    }

    // 2. 余额 <= 0 直接返回
    if (trxBalance <= 0) return null;

    // 3. ⭐ TRC20 Gas 管理（关键步骤）
    String tokenContract = TRXContractEnum.getByTokenContractByContractName(coinName);
    if (!是TRX主币) {
        BigDecimal trxBalance = getAccountBalance(fromAddress, "TRX");
        if (trxBalance < 30_TRX) {  // FEE_BAL = 30 TRX
            // 从 Gas 地址转 30 TRX 到用户地址（支付手续费）
            result = createTransactionForTrx(gasPrivateKey, 30,
                    gasAddress, fromAddress, "TRX", ssOrderId);
            // 记录 Gas 补充订单
            trxWalletOrderService.save(userId, gasAddress, fromAddress,
                    result.getTxId(), ssOrderId, 30, RECHARGE, "TRX", postType);
        }
    }

    // 4. 执行归集转账（用户地址 → 归集地址）
    result = createTransactionForTrx(null, trxBalance,
            fromAddress, mainAddr, tokenContract, ssOrderId);

    // 5. 记录归集 + 订单
    reverseNoticeCollectDetail(userId, fromAddress, txId,
            trxBalance, collectCode, ssOrderId, coinName, postType);

    return result;
}
```

### 5.3 TRC20 转账核心代码解析

```java
// WalletTrxBusiness.java:209-290 — trc20Transaction
public TrxWalletResult trc20Transaction(String keyStore, String from,
        String to, String tokenContract, BigDecimal amount, String ssOrderId) {

    // 1. 地址校验
    if (!validateAddress(to)) return null;
    if (!tokenContract.startsWith("T")) throw "不支持除TRC20的合约";

    // 2. 构建智能合约调用参数
    Map<String, Object> params = new HashMap<>();
    params.put("contract_address", toHexAddress(tokenContract));
    params.put("function_selector", "transfer(address,uint256)");

    // 3. ABI 编码：transfer(address, uint256)
    BigInteger amountInSmallestUnit = amount * 10^decimals;  // USDT: 6 位精度
    String parameter = encoderAbi(toHexAddress(to).substring(2), amountInSmallestUnit);
    params.put("parameter", parameter);
    params.put("owner_address", toHexAddress(from));
    params.put("fee_limit", 50_000_000L);  // 最大 50 TRX 手续费

    // 4. 调用 TRON HTTP API 创建未签名交易
    String response = HttpClient4Util.doPost(
        trxUrl + "/wallet/triggersmartcontract", JSON(params));
    JSONObject transaction = parse(response).getJSONObject("transaction");

    // 5. 打包成 Protobuf Transaction
    transaction.getJSONObject("raw_data").put("data", hex("trc20"));
    Protocol.Transaction tx = TronUtils.packTransaction(transaction, false);

    // 6. ⭐ 签名分支
    byte[] signedBytes;
    if (是出款地址(from)) {
        // 提现场景：调用外部签名服务（RSA 加密 hash 发送）
        signedBytes = sendSign(signUrl, tx, ssOrderId, from);
    } else {
        // 归集场景：本地签名（用户私钥已解密）
        signedBytes = signTransaction2Byte(tx.toByteArray(),
                ByteArray.fromHexString(keyStore));
    }

    // 7. 广播上链
    Map<String, Object> broadcastParams = new HashMap<>();
    broadcastParams.put("transaction", toHexString(signedBytes));
    String broadcastResult = HttpClient4Util.doPost(
        trxUrl + "/wallet/broadcasthex", JSON(broadcastParams));

    // 8. 解析结果
    String txId = parse(broadcastResult).getString("txid");
    return new TrxWalletResult(txId);
}
```

### 5.4 外部签名服务调用

```java
// WalletTrxBusiness.java:292-321 — sendSign
private byte[] sendSign(String signUrl, Protocol.Transaction tx,
        String ssOrderId, String from) {

    // 1. 计算交易哈希
    byte[] txId = Sha256Sm3Hash.hash(tx.getRawData().toByteArray());
    String signTxid = Base64.encode(txId);

    // 2. 构建签名请求
    Map<String, Object> signMap = new HashMap<>();
    signMap.put("orderNo", ssOrderId);
    signMap.put("outAddressNo", from);
    signMap.put("chainType", "TRON");
    signMap.put("hashId", signTxid);

    // 3. RSA 加密请求体
    String rsaPublicKey = trxWalletConfig.findByCode("rsa").getValue();
    String encryptedBody = RSAUtil.encrypt(JSON(signMap), rsaPublicKey);

    // 4. 发送签名请求
    HttpResponse response = HttpRequest.post(signUrl + "/api/admin/chain/sign")
        .header("PLAT_CODE", trxWalletConfig.findByCode("platCode").getValue())
        .body(encryptedBody)
        .timeout(60000)  // 60 秒超时
        .execute();

    // 5. 解析签名结果
    JSONObject result = parse(response.body());
    if ("200".equals(result.get("code"))) {
        byte[] signature = Base64.decode(result.getString("data"));
        // 6. 组装已签名交易
        return tx.toBuilder().addSignature(ByteString.copyFrom(signature))
                .build().toByteArray();
    }
    return null;  // 签名失败
}
```

**✅ 外部签名服务的安全价值**：

```
核心理念：应用服务器不持有出款地址的私钥

传统模式（不安全）：
  应用服务器 持有 出款私钥 → 应用被攻破 = 资金全失

本项目模式（更安全）：
  应用服务器 → 计算交易哈希 → RSA加密 → 发送给签名服务
  签名服务 → 解密 → 用出款私钥签名 → 返回签名

好处：
  1. 私钥不在应用服务器——即使应用被攻破，也无法转走资金
  2. 签名服务可以独立部署在安全网络中
  3. 签名服务可以增加审批/限额/频率控制

我们的采纳方案：
  → 出款地址的私钥由独立的 key-signer 微服务管理
  → 或使用 HSM (Hardware Security Module) / 云 KMS 直接签名
```

---

## 六、提现流程详解

### 6.1 完整提现链路

```
用户发起提现
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ user-server: VirtualCurrencyController.withdraw()        │
│ 1. Redis 锁防重                                          │
│ 2. 校验用户余额 >= 提现金额                                │
│ 3. 冻结余额                                              │
│ 4. 创建 SsOrder(orderType=提现, status=等待)               │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│ user-server: WalletBusiness.withdrawTransaction()        │
│ 5. 构建请求：{userId, toAddress, amount-fee, coinName}    │
│ 6. HTTP POST → wallet-server /wallet/withdrawTraction    │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│ wallet-server: TrxDriver.withdrawTraction()              │
│ 7. 获取出款地址 (TrxWalletConfig 表)                      │
│ 8. 幂等检查：findBySsOrderId → 已存在则返回缓存 txId      │
│ 9. 余额校验：出款地址余额 >= 提现金额                       │
│ 10. Gas 管理：TRC20 需检查出款地址 TRX >= 30               │
│     不足 → gasAddress 转 30 TRX 到出款地址                 │
│ 11. 构建 TRC20 交易                                       │
│ 12. 外部签名服务签名（RSA 加密）                            │
│ 13. 广播上链 /wallet/broadcasthex                         │
│ 14. 记录 TrxWalletOrder（txId + 金额 + 方向）             │
│ 15. 返回 txId                                            │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│ user-server: AsyncVirtualWithdrawService（定时任务）       │
│ 16. 查询交易状态 /wallet/getTransactionInfoById           │
│ 17-a. 成功：                                              │
│     → 解冻余额                                           │
│     → 扣减真实金额（balanceType=TELEGRAM_WITHDRAW）         │
│     → 扣减手续费（balanceType=FEE）                        │
│     → 更新订单状态（success + txId）                       │
│     → MQ 通知后台统计                                     │
│ 17-b. 超时/失败：                                         │
│     → 解冻余额（退回）                                    │
│     → 更新订单状态（cancelled）                            │
└─────────────────────────────────────────────────────────┘
```

### 6.2 提现关键设计要点

```
要点 1：先冻结后出款
  → user-server 先冻结用户余额
  → wallet-server 执行链上转账
  → 成功后才真正扣款
  → 失败则解冻退回
  → 与参考 A 的冻结三步法（freeze/sub/rollback）思路一致

要点 2：出款地址与归集地址分离
  → 出款地址（dispensing）不等于归集地址（collect）
  → 出款私钥由外部签名服务管理
  → 归集私钥在应用内加密管理
  → 职责分离 + 安全分层

要点 3：幂等机制（ssOrderId）
  → 每次提现携带 ssOrderId
  → wallet-server 检查 TrxWalletOrder.findBySsOrderId
  → 已存在 → 直接返回缓存的 txId
  → 防止 user-server 重复调用导致双重出款

要点 4：手续费处理
  → user-server 层面：amount（总额）- feeAmount（手续费）= 实际出款额
  → wallet-server 只看 amount 字段（已扣除手续费）
  → 手续费在确认成功后从用户余额单独扣除
```

---

## 七、数据模型深度分析

### 7.1 四张表结构

#### 表 1：TrxWallet（用户热钱包）

```
┌──────────────┬────────────────┬────────────────────────────────────┐
│ 字段          │ 类型            │ 说明                               │
├──────────────┼────────────────┼────────────────────────────────────┤
│ id           │ bigint AUTO     │ 自增主键                           │
│ user_id      │ varchar(10)     │ 会员账号（有索引）                  │
│ user_name    │ varchar(30)     │ 会员姓名                           │
│ en_private_key│ varchar(250)   │ AES 加密后的私钥                   │
│ address      │ varchar(100)    │ TRON T 开头地址                    │
│ create_time  │ datetime        │ 创建时间                           │
│ update_time  │ datetime        │ 更新时间                           │
│ create_by    │ varchar         │ 创建人                             │
│ update_by    │ varchar         │ 更新人                             │
└──────────────┴────────────────┴────────────────────────────────────┘

设计特征：
  - 一人一个地址（userId 有索引）
  - 私钥加密存储（AES + KMS 管理 passphrase）
  - 不存明文私钥，不存助记词
```

#### 表 2：TrxWalletOrder（链上交易记录）

```
┌──────────────┬────────────────┬────────────────────────────────────┐
│ 字段          │ 类型            │ 说明                               │
├──────────────┼────────────────┼────────────────────────────────────┤
│ id           │ bigint AUTO     │ 自增主键                           │
│ user_id      │ varchar(10)     │ 会员账号（索引）                    │
│ tx_id        │ varchar(200)    │ 链上交易 Hash（索引）               │
│ ss_order_id  │ varchar(200)    │ 上游订单 ID（幂等键）               │
│ amount       │ decimal(16,2)   │ 交易金额                           │
│ from_address │ varchar(250)    │ 发送地址                           │
│ to_address   │ varchar(250)    │ 接收地址                           │
│ coin_name    │ varchar(10)     │ 币种（USDT/TRX/CXT）               │
│ order_type   │ tinyint(2)      │ 方向 1=充值 2=提现                 │
│ post_type    │ tinyint(2)      │ 来源 1=接口 2=定时任务             │
│ create_time  │ datetime        │ 创建时间                           │
│ update_time  │ datetime        │ 更新时间                           │
└──────────────┴────────────────┴────────────────────────────────────┘

设计特征：
  - tx_id 是链上交易哈希（唯一凭证）
  - ss_order_id 关联上游业务订单（幂等键）
  - 记录 from/to 地址——可回溯每笔链上操作
  - order_type 区分充值和提现方向
```

#### 表 3：trx_wallet_collect（归集记录）

```
┌──────────────┬────────────────┬────────────────────────────────────┐
│ 字段          │ 类型            │ 说明                               │
├──────────────┼────────────────┼────────────────────────────────────┤
│ id           │ bigint AUTO     │ 自增主键                           │
│ user_id      │ varchar(10)     │ 会员账号（索引）                    │
│ tx_id        │ varchar(200)    │ 归集交易 Hash（索引）               │
│ collect_address│ varchar(250)  │ 汇款地址                           │
│ amount       │ decimal(16,2)   │ 归集金额                           │
│ collect_type │ tinyint(2)      │ 归集类型 1=单笔 2=批量             │
│ create_time  │ datetime        │ 创建时间                           │
└──────────────┴────────────────┴────────────────────────────────────┘
```

#### 表 4：TrxWalletConfig（动态配置）

```
┌──────────────┬────────────────┬────────────────────────────────────┐
│ 字段          │ 类型            │ 说明                               │
├──────────────┼────────────────┼────────────────────────────────────┤
│ id           │ bigint AUTO     │ 自增主键                           │
│ type         │ varchar(30)     │ 配置类型（如 "sign"）               │
│ code         │ varchar(50)     │ 配置项（rsa/platCode/disPendAddress）│
│ value        │ varchar(1000)   │ 配置值                             │
└──────────────┴────────────────┴────────────────────────────────────┘

存储的配置：
  type="sign", code="rsa"            → RSA 公钥（用于签名服务通信）
  type="sign", code="platCode"       → 平台标识码
  type="sign", code="disPendAddress" → 出款地址
```

#### 表 5：SsOrder（在 user-server，充提订单）

```
┌──────────────────┬────────────────┬────────────────────────────────────┐
│ 字段              │ 类型            │ 说明                               │
├──────────────────┼────────────────┼────────────────────────────────────┤
│ id               │ bigint AUTO     │ 自增主键                           │
│ order_num        │ varchar(32)     │ 订单编号                           │
│ user_id          │ bigint          │ 会员 ID                           │
│ tg_user_id       │ varchar(64)     │ Telegram 用户 ID                  │
│ amount           │ decimal(16,2)   │ 总金额                            │
│ real_amount      │ decimal(16,2)   │ 实际到账金额                       │
│ fee_amount       │ decimal(16,2)   │ 手续费                            │
│ order_type       │ tinyint(2)      │ 1=充值 2=提现                     │
│ order_status     │ tinyint(2)      │ 1=等待 2=成功 3=失败 4=取消        │
│ virtual_currency │ varchar(20)     │ 虚拟币种（USDT）                   │
│ payee_address    │ varchar(200)    │ 收款地址（提现目标地址）             │
│ tx_id            │ varchar(200)    │ 链上交易 ID                        │
│ is_lock          │ tinyint(4)      │ 是否锁单（审核中）                  │
│ audit_user       │ varchar(64)     │ 审核人                            │
│ audit_time       │ datetime        │ 审核时间                           │
│ audit_remark     │ varchar(200)    │ 审核原因                           │
│ complete_time    │ datetime        │ 完成时间                           │
│ flow_multiple    │ decimal(16,2)   │ 流水倍数（打码量 = amount × 倍数）  │
│ agent_id         │ bigint          │ 代理 ID                           │
│ is_dan_bao       │ tinyint(2)      │ 是否担保                           │
│ ...              │                 │ （更多审计字段）                    │
└──────────────────┴────────────────┴────────────────────────────────────┘
```

### 7.2 数据模型对照（与我们的 ser-wallet 设计）

```
tg-services                              Go / ser-wallet
═══════════════════════════════════════════════════════════════════════════
TrxWallet (用户热钱包)      ─────→      💡 新增：user_wallet_address 表
                                         （user_id + chain + address + en_key）

TrxWalletOrder (交易记录)   ─────→      可融入 deposit_order / withdraw_order
                                         增加 tx_id + from_addr + to_addr 字段

trx_wallet_collect (归集)   ─────→      💡 新增：wallet_collect_record 表
                                         或融入 wallet_flow（flow_type=归集）

TrxWalletConfig (动态配置)  ─────→      可融入 wallet_config 表
                                         type + code + value 的 KV 模式

SsOrder (充提订单)          ─────→      已有 deposit_order + withdraw_order
                                         增加 virtual_currency + tx_id 字段

我们额外需要的（Java 项目没有但我们已设计的）：
  ✅ currency_config        — 币种配置（含虚拟币种）
  ✅ wallet_flow            — 资金流水（含链上操作记录）
  ✅ wallet_freeze_record   — 冻结记录（提现冻结追溯）
  ✅ exchange_record        — 兑换记录（法币↔虚拟币）
```

---

## 八、Gas 管理机制（TRC20 必备）

### 8.1 核心问题

```
TRON 网络的手续费规则：
  - TRX 主币转账：消耗带宽（Bandwidth），不足时扣 TRX
  - TRC20 代币转账：必须消耗能量（Energy），不足时扣 TRX
  - 一笔 TRC20 USDT 转账约需 5~30 TRX 手续费

问题：
  用户充值 USDT 到热钱包后，热钱包没有 TRX → 无法归集 USDT
  出款地址发送 USDT 时，出款地址可能 TRX 不足

解决方案：专用 Gas 地址 + 自动补充
```

### 8.2 Gas 自动补充逻辑

```java
// TrxDriver 中的 Gas 检查逻辑（归集和提现都用）
private static final BigDecimal FEE_BAL = new BigDecimal("30"); // 30 TRX

// 在归集或提现前：
if (!是TRX主币(tokenContract)) {
    BigDecimal trxBalance = getAccountBalance(fromAddress, "TRX");
    if (FEE_BAL.compareTo(trxBalance) > 0) {
        // Gas 不足，从 Gas 地址转 30 TRX
        result = createTransactionForTrx(
            gasPrivateKey,      // Gas 地址私钥
            FEE_BAL,            // 30 TRX
            gasAddress,         // 来源：Gas 地址
            fromAddress,        // 目标：需要 Gas 的地址
            "TRX",              // 转的是 TRX
            ssOrderId
        );
        // 记录 Gas 补充订单
        trxWalletOrderService.save(...);
    }
}
```

**✅ Go 项目采纳方案**：

```
1. 配置项：gas_address + gas_private_key
2. 阈值：MIN_TRX_FOR_GAS = 30 (可配置)
3. 触发时机：每次 TRC20 归集/提现前检查
4. 操作：gasAddress → targetAddress 转 MIN_TRX_FOR_GAS
5. 记录：写入 wallet_flow（flow_type = GAS_REFUEL）
```

---

## 九、智能合约调用详解

### 9.1 TRC20 余额查询（只读调用）

```java
// Trc20.java:37-49
public static BigDecimal balanceOf(String address, String contract) {
    // 1. 地址转 hex 格式
    String addressHex = TronUtil.toHexAddress(address);

    // 2. ABI 编码：balanceOf(address)
    String param = encoderBalanceOfAbi(addressHex.substring(2));

    // 3. 调用只读合约方法（不消耗 Gas）
    String result = SmartContract.triggerConstantContract(
        address,                  // 调用方
        contract,                 // 合约地址
        "balanceOf(address)",     // 方法签名
        param                     // 编码后的参数
    );

    // 4. 解码返回值
    BigDecimal rawBalance = decoderBalanceOf(result);  // Uint256
    Integer decimals = decimals(contract);              // 获取精度
    return rawBalance.divide(BigDecimal.TEN.pow(decimals));
}
```

### 9.2 TRC20 转账（写入调用）

```
调用链：
  1. 编码 ABI 参数：transfer(address, uint256)
     → encoderTransferAbi(hexAddress, amount * 10^decimals)

  2. HTTP POST → trxUrl + "/wallet/triggersmartcontract"
     参数：{contract_address, function_selector, parameter, owner_address, fee_limit}
     返回：未签名的交易 JSON（含 raw_data 和 txID）

  3. 打包为 Protobuf：TronUtils.packTransaction(jsonTransaction)
     → 遍历 contract 数组，解析每种合约类型
     → 序列化为 Protocol.Transaction

  4. 签名：
     本地签名：signTransaction2Byte(tx.bytes, privateKeyBytes)
     外部签名：sendSign(signUrl, tx, orderNo, fromAddress)

  5. 广播：HTTP POST → trxUrl + "/wallet/broadcasthex"
     参数：{transaction: hexString(signedTransaction)}
     返回：{result: true/false, txid: "xxx"}
```

### 9.3 TRON API 端点汇总

```
TRON Full Node HTTP API（本项目使用的端点）：
─────────────────────────────────────────────────────────────
/wallet/getaccount              查询 TRX 主币余额
/wallet/validateaddress         校验地址格式（Base58Check）
/wallet/triggersmartcontract    触发智能合约（生成交易）
/wallet/triggerconstantcontract 只读智能合约调用（不上链）
/wallet/broadcasthex            广播已签名的交易
/wallet/gettransactionbyid      根据 txId 查询交易状态

gRPC API（SmartContract.java 使用）：
─────────────────────────────────────────────────────────────
triggerConstantContract         只读合约调用（查余额/精度）
triggerContract                 写入合约调用（转账）
```

**✅ Go 项目对应方案**：

```go
// 使用 go-tron-sdk 或直接 HTTP/gRPC 调用 TRON 节点
// HTTP 方式更简单，gRPC 方式更高效

type TronClient struct {
    baseURL    string  // "https://api.trongrid.io" (主网) 或 nile 测试网
    httpClient *http.Client
}

func (c *TronClient) GetBalance(address string) (*big.Int, error)
func (c *TronClient) TriggerSmartContract(params TriggerParams) (*Transaction, error)
func (c *TronClient) BroadcastHex(signedTx string) (*BroadcastResult, error)
func (c *TronClient) GetTransactionById(txId string) (*TransactionInfo, error)
```

---

## 十、三层加密体系详解

### 10.1 加密架构全景

```
Layer 1: AWS KMS（配置级加密）
─────────────────────────────────────────
  加密对象：application.yml 中的敏感配置
    - companyAddress（归集地址）
    - mainCollectPassphrase（归集助记词）
    - gasAddress + gasPrivateKey（Gas 地址和私钥）
    - mainDispensingAddress + privateKey（出款地址和私钥）

  工作方式：
    1. 部署前：明文 → AwsKmsDriver.encryptData() → 密文写入 YAML
    2. 运行时：密文 → AwsKmsDriver.decryptData() → 内存中明文使用
    3. AWS KMS Key ID: mrk-2017a36cde86493d8be54346415708a5
    4. 区域：AP_SOUTHEAST_1

Layer 2: AES-256/ECB（私钥存储加密）
─────────────────────────────────────────
  加密对象：每个用户热钱包的私钥

  加密过程：
    passphrase = KMS.decrypt(application.yml中的密文)
    secret = passphrase + userAddress  // 如 "heart essence...TUyKqd..."
    aesKey = SHA1(secret)[0:16]        // SHA-1 取前 16 字节
    encrypted = AES_ECB_PKCS5(privateKeyHex, aesKey)
    存储到 DB: TrxWallet.enPrivateKey = encrypted

  解密过程：
    passphrase = KMS.decrypt(...)
    secret = passphrase + userAddress
    privateKeyHex = AES.decrypt(enPrivateKey, secret)

Layer 3: RSA（提现签名通信加密）
─────────────────────────────────────────
  加密对象：提现签名请求

  工作方式：
    1. 应用 → 计算交易哈希 hashId
    2. 构建请求 {orderNo, outAddressNo, chainType, hashId}
    3. RSA.encrypt(JSON请求, 签名服务公钥)
    4. HTTP POST → 签名服务
    5. 签名服务 RSA.decrypt → 用出款私钥签名 → 返回签名值
```

### 10.2 加密方案评估

```
✅ 优点：
  1. 分层加密——不同层次的数据用不同方案保护
  2. 私钥永不明文存储——AES 加密后入库
  3. 出款私钥不在应用——外部签名服务隔离
  4. KMS 管理配置密钥——运维人员无法直接获取

⚠️ 改进空间：
  1. AES/ECB 模式不如 GCM——相同明文产生相同密文
  2. SHA-1 密钥派生强度不够——应使用 PBKDF2/Argon2
  3. RSA 512 位太短——现在推荐 2048+ 位
  4. 日志中可能泄露敏感信息——多处 log.info 打印地址和金额

Go 项目改进方案：
  1. AES-256-GCM 替代 AES-ECB
  2. Argon2id 替代 SHA-1 做密钥派生
  3. 使用阿里云 KMS 或 HashiCorp Vault 替代 AWS KMS
  4. 日志脱敏——地址只显示前 6 后 4 位
```

---

## 十一、user-server 充提订单管理

### 11.1 充值订单生命周期

```
         创建               轮询检测              归集成功              加余额
  ┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────┐
  │ status=1 │ ──→ │ 每 5 分钟     │ ──→ │ status=2     │ ──→ │ 加 real  │
  │ (等待)    │     │ 检查链上余额  │     │ (成功)       │     │ Amount  │
  └──────────┘     │ + 触发归集    │     └──────────────┘     │ + 打码量 │
                   └──────────────┘                           └──────────┘
                          │
                          ▼
                   ┌──────────────┐
                   │ 超时（>N分钟）│ ──→ status=4 (取消)
                   └──────────────┘

关键细节：
  1. 用户在 TG Bot 输入充值金额 → 创建 SsOrder(status=等待)
  2. 定时任务每 5 分钟轮询：
     → 调用 wallet-server /wallet/getBalance 查链上余额
     → 余额 >= 订单金额 → 调用 /wallet/collectTrc20 触发归集
     → 归集成功（返回 txId）→ 更新订单状态为成功
  3. 成功后：
     → addBalance（真实到账金额）
     → addPlayMoney（打码量 = amount × flowMultiple）
     → 首充用户特殊处理（清空体验金）
  4. 超时未到账 → 取消订单
```

### 11.2 提现订单生命周期

```
         创建               审核               出款               确认
  ┌──────────┐     ┌──────────────┐     ┌──────────┐     ┌──────────┐
  │ status=1 │ ──→ │ B 端审核通过  │ ──→ │ 冻结余额  │ ──→ │ 调用     │
  │ (等待)    │     │ isLock=1     │     │ 扣 fee   │     │ withdraw │
  └──────────┘     └──────────────┘     └──────────┘     │ Traction │
                                                          └────┬─────┘
                                                               │
                                              ┌────────────────┼────────────────┐
                                              ▼                                 ▼
                                       ┌──────────┐                      ┌──────────┐
                                       │ 成功      │                      │ 失败/超时 │
                                       │ 解冻+扣款 │                      │ 解冻退回  │
                                       │ status=2  │                      │ status=4  │
                                       └──────────┘                      └──────────┘

关键细节：
  1. 用户发起提现 → 冻结余额 → 创建 SsOrder(status=等待)
  2. 调用 wallet-server 执行链上提现
  3. 定时任务轮询交易状态：
     → 成功：解冻余额 + 扣减真实金额 + 扣减手续费 + MQ 通知
     → 失败/超时：解冻余额退回 + 标记取消
  4. 手续费单独扣减（BalanceChangeEnum.FEE = 90）
```

---

## 十二、已发现的设计缺陷与风险

### 12.1 安全问题

```
问题 1：AES/ECB 模式
  位置：AES.java / WalletTrxBusiness.java:183-195
  风险：相同私钥 + 相同地址 → 相同密文（无随机 IV）
  级别：中
  我们的规避：使用 AES-256-GCM（带随机 Nonce）

问题 2：RSA 512 位密钥
  位置：RSAUtil.java:5
  风险：512 位 RSA 已可被暴力破解
  级别：高
  我们的规避：使用 RSA 2048 位或 ECDSA

问题 3：配置文件明文存储敏感信息
  位置：application.yml
  风险：数据库密码、Redis 密码明文存储（非 KMS 加密部分）
  级别：中
  我们的规避：所有敏感配置统一 KMS 加密

问题 4：日志泄露
  位置：多处 log.info 打印完整地址、金额、私钥解密过程
  风险：日志被获取 → 可追踪资金流向
  级别：中
  我们的规避：日志脱敏中间件
```

### 12.2 架构问题

```
问题 5：无链上 Event 监听
  表现：充值依赖定时任务轮询（5 分钟间隔）
  影响：充值确认延迟可达 5~10 分钟
  改进：部署 TRON Event 监听节点 or 使用 TronGrid Event API
  → 实时监听 Transfer 事件，秒级确认

问题 6：归集效率低
  表现：CollectTrxJob 每日 8 点遍历所有用户钱包
  影响：用户量大时，归集耗时长（需逐个转账）
  改进：充值确认后立即归集（Event 触发），非定时批量

问题 7：无交易重试机制
  表现：链上操作失败仅返回 null，无自动重试
  影响：网络波动导致交易丢失
  改进：本地消息表 + 重试队列

问题 8：Gas 补充无等待确认
  位置：TrxDriver.java:196
  表现：Gas TRX 转账后立即执行 TRC20 归集
  影响：Gas 交易未确认时 TRC20 归集可能因 Gas 不足失败
  改进：Gas 转账后等待确认（查 getTransactionInfoById），或设置更大余量
```

---

## 十三、可采纳的设计模式汇总

### 13.1 直接采纳（✅ 生产验证方案）

| 序号 | 设计模式                           | 来源                           | Go 实现方案                         |
|------|-----------------------------------|-------------------------------|-------------------------------------|
| 1    | 四类地址分离架构                    | TrxDriver 配置                 | 配置表管理 4 类地址                  |
| 2    | 用户一人一地址                     | TrxWallet 表                   | user_wallet_address 表              |
| 3    | 私钥加密存储（AES + KMS）          | WalletTrxBusiness.encrypt      | AES-256-GCM + 阿里云 KMS           |
| 4    | 外部签名服务（出款私钥分离）        | sendSign 方法                   | 独立 key-signer 服务                |
| 5    | Gas 自动补充                       | singleCollect 中 FEE_BAL 检查  | TRC20 操作前自动检查 + 补充          |
| 6    | ssOrderId 幂等（防双重出款）        | withdrawTraction 中 findBy     | 唯一索引 + 先查后执行                |
| 7    | 先冻结后出款                       | user-server 提现流程            | wallet_freeze_record + 确认/回滚    |
| 8    | TRC20 ABI 编码（web3j）            | Trc20.java + encoderAbi        | go-ethereum/abi 包                  |
| 9    | Protobuf 交易打包                  | TronUtils.packTransaction      | go-tron protobuf 定义               |
| 10   | 定时归集任务                        | CollectTrxJob                  | Cron Job + 可配置时间               |

### 13.2 借鉴改进（⚠️ 思路可取，需优化）

| 序号 | 原始设计                    | 问题                         | 我们的改进                            |
|------|----------------------------|------------------------------|--------------------------------------|
| 1    | 轮询检测充值（5min）        | 延迟高                        | TRON Event 监听 + 轮询兜底双模式     |
| 2    | 每日定时归集                | 资金分散风险                   | 充值到账后立即归集                    |
| 3    | AES/ECB 加密               | 安全性不足                    | AES-256-GCM                          |
| 4    | SHA-1 密钥派生              | 强度不够                      | Argon2id                             |
| 5    | RSA 512 位                  | 可被破解                      | RSA 2048 或 ECDSA                    |
| 6    | Gas 补充无等待确认          | 可能失败                      | 补充后轮询确认再继续                  |
| 7    | HTTP 调 TRON API            | 性能一般                      | gRPC 为主，HTTP 为辅                  |
| 8    | 无重试机制                  | 交易丢失                      | 本地消息表 + 指数退避重试             |

### 13.3 明确不采纳（❌）

| 序号 | 设计                          | 不采纳原因                                    |
|------|------------------------------|----------------------------------------------|
| 1    | JPA/Hibernate DDL-auto       | 我们用 GORM Gen + 手动 DDL 管理               |
| 2    | Spring Security Basic Auth   | 我们用 JWT + 网关鉴权                          |
| 3    | AWS KMS                      | 我们可能使用阿里云 KMS 或自建 Vault            |
| 4    | wallet-cli JAR 依赖          | Go 项目用 Go 原生 TRON SDK                    |
| 5    | HttpClient4Util 同步 HTTP    | Go 项目用 Kitex RPC + 异步 HTTP Client         |

---

## 十四、我们的 ser-wallet 需要新增的设计

### 14.1 链上操作相关的新增表

```
💡 新增表 1：user_wallet_address（用户链上地址）
┌──────────────────┬───────────────┬────────────────────────────────────┐
│ 字段              │ 类型           │ 说明                               │
├──────────────────┼───────────────┼────────────────────────────────────┤
│ id               │ bigint        │ 自增主键                           │
│ user_id          │ bigint        │ 用户 ID                           │
│ chain            │ varchar(20)   │ 链类型（TRON/ETH/BSC）             │
│ address          │ varchar(128)  │ 链上地址                           │
│ encrypted_key    │ text          │ 加密后的私钥                       │
│ key_version      │ int           │ 密钥版本（支持密钥轮换）            │
│ status           │ tinyint       │ 1=正常 2=禁用                     │
│ created_at       │ datetime      │ 创建时间                           │
└──────────────────┴───────────────┴────────────────────────────────────┘
索引：UNIQUE(user_id, chain)

💡 新增表 2：chain_transaction（链上交易记录）
┌──────────────────┬───────────────┬────────────────────────────────────┐
│ 字段              │ 类型           │ 说明                               │
├──────────────────┼───────────────┼────────────────────────────────────┤
│ id               │ bigint        │ 自增主键                           │
│ tx_hash          │ varchar(128)  │ 链上交易 Hash                      │
│ chain            │ varchar(20)   │ 链类型                             │
│ from_address     │ varchar(128)  │ 发送地址                           │
│ to_address       │ varchar(128)  │ 接收地址                           │
│ amount           │ decimal(30,8) │ 金额（高精度）                     │
│ token            │ varchar(20)   │ 代币名称（TRX/USDT）               │
│ contract_address │ varchar(128)  │ 合约地址（TRC20 用）               │
│ tx_type          │ tinyint       │ 1=充值归集 2=提现 3=Gas补充         │
│ status           │ tinyint       │ 1=待确认 2=成功 3=失败             │
│ biz_order_id     │ varchar(64)   │ 关联业务订单号（幂等键）            │
│ retry_count      │ int           │ 重试次数                           │
│ confirmed_at     │ datetime      │ 链上确认时间                       │
│ created_at       │ datetime      │ 创建时间                           │
└──────────────────┴───────────────┴────────────────────────────────────┘
索引：UNIQUE(biz_order_id, chain), INDEX(tx_hash)
```

### 14.2 ser-wallet 新增的内部模块

```
ser-wallet/internal/
├── chain/                    ← 💡 新增：链上操作模块
│   ├── tron/
│   │   ├── client.go         ← TRON HTTP/gRPC 客户端
│   │   ├── trc20.go          ← TRC20 代币操作（查余额/转账/ABI 编码）
│   │   ├── address.go        ← 地址生成/转换（Base58Check ↔ Hex）
│   │   ├── signer.go         ← 交易签名（本地签名 + 外部签名服务）
│   │   └── gas.go            ← Gas 管理（自动补充逻辑）
│   ├── keystore/
│   │   ├── encrypt.go        ← 私钥加密/解密（AES-256-GCM + Argon2id）
│   │   └── kms.go            ← KMS 集成（阿里云/自建）
│   └── collector/
│       ├── collector.go      ← 归集逻辑（单笔+批量）
│       └── monitor.go        ← 链上事件监听（替代轮询）
│
├── ser/
│   ├── deposit_chain.go      ← 💡 新增：链上充值业务编排
│   └── withdraw_chain.go     ← 💡 新增：链上提现业务编排
```

---

## 十五、充值方案对比（轮询 vs Event 监听）

### 15.1 当前 Java 项目方案（轮询）

```
优点：实现简单，不依赖节点事件接口
缺点：延迟高（5 分钟），效率低，重复查询浪费资源

流程：
  定时任务(5min) → 查链上余额 → 余额 >= 订单金额 → 触发归集
```

### 15.2 推荐方案（Event 监听 + 轮询兜底）

```
主通道：TRON Event 监听
  1. 监听 TRC20 合约的 Transfer 事件
  2. 过滤 to_address = 我们的用户热钱包地址
  3. 匹配充值订单 → 立即触发归集
  4. 延迟：秒级

兜底通道：定时轮询
  1. 每 10 分钟扫描未确认的充值订单
  2. 查链上余额确认
  3. 处理 Event 遗漏的情况

TRON Event API（TronGrid 提供）：
  GET /v1/contracts/{contractAddress}/events?event_name=Transfer
  GET /v1/accounts/{address}/transactions/trc20

Go 实现：
  go func() {
      ticker := time.NewTicker(3 * time.Second)
      for range ticker.C {
          events := tronClient.GetTRC20Events(usdtContract, sinceTimestamp)
          for _, e := range events {
              if isOurAddress(e.To) {
                  handleDeposit(e)
              }
          }
      }
  }()
```

---

## 十六、总结与行动建议

### 16.1 核心收获

```
从这个生产运行的虚拟币钱包项目中，我们获得了以下关键参考：

1.【四类地址架构】用户热钱包 / 归集地址 / 出款地址 / Gas 地址 的分离设计
   → 这是虚拟币钱包的标准架构，直接采纳

2.【私钥安全管理】三层加密（KMS + AES + RSA 外部签名）
   → 核心思想采纳，算法升级（AES-GCM + Argon2id + RSA-2048）

3.【TRC20 完整链路】从钱包生成到充值归集到提现出款的完整代码
   → 最高参考价值，Go 语言重写时可以 1:1 映射逻辑

4.【Gas 自动管理】TRC20 交易的 TRX 手续费自动检测和补充
   → 直接复用，这是 TRC20 操作的必备机制

5.【外部签名服务】出款私钥不在应用服务器
   → 安全架构的关键设计，必须采纳

6.【幂等机制】ssOrderId 防双重出款
   → 链上操作的幂等是资金安全的底线
```

### 16.2 与模块参考 A 的综合对照

```
维度                参考 A（p9 wallet）        参考 B（tg wallet）        我们的综合方案
───────────────────────────────────────────────────────────────────────────────────────
余额管理            ✅ SQL 原子 UPDATE          ✗（链上余额）              采纳 A
冻结机制            ✅ freeze 三步法            ✅ 提现前冻结              综合采纳
账变流水            ✅ coin_record              ✅ TrxWalletOrder          wallet_flow 统一
并发控制            ✅ Redisson 锁 + SQL        ✗（无并发控制）            采纳 A
链上操作            ✗                           ✅ 完整 TRC20 链路         采纳 B
私钥管理            ✗                           ✅ 三层加密                采纳 B（改进算法）
签名服务            ✗                           ✅ RSA 外部签名            采纳 B（改进密钥长度）
Gas 管理            ✗                           ✅ 自动补充                采纳 B
归集流程            ✗                           ✅ 单笔+批量归集           采纳 B（改为 Event 触发）
幂等机制            ✗（无）                     ✅ ssOrderId               wallet_flow.idempotent_key
充值确认            ✗                           ⚠️ 5min 轮询               改进为 Event 监听+轮询兜底
```

### 16.3 下一步行动清单

```
优先级 P0（链上操作基础设施）：
  ① TRON 客户端封装（HTTP + gRPC 双通道）
  ② 地址生成 + Base58Check 编解码
  ③ TRC20 ABI 编码/解码
  ④ 交易签名（本地 ECDSA + 外部签名服务接口）

优先级 P1（充提核心链路）：
  ⑤ 私钥加密存储方案（AES-256-GCM + Argon2id + KMS）
  ⑥ 充值归集流程（Event 监听 + Gas 补充 + 归集执行）
  ⑦ 提现出款流程（冻结 → 签名 → 广播 → 确认/回滚）
  ⑧ chain_transaction 表 + user_wallet_address 表设计

优先级 P2（运维与监控）：
  ⑨ Gas 地址余额告警
  ⑩ 归集账户余额监控
  ⑪ 链上交易状态重试机制
  ⑫ 日志脱敏中间件
```

---

> 本文档基于 tg-services 工程源码逐文件深度分析完成，覆盖 wallet-server 全部 35+ 文件 + user-server 充提相关 10+ 文件 + 配置文件 + SQL 脚本。所有结论均有源码行号和方法名佐证。核心价值在于 TRC20 虚拟币充值归集和提现出款的完整生产实现链路，这是我们 ser-wallet 中最难自研的部分。
