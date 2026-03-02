# P9工程钱包模块完整深度解剖

> **定位**：对参考工程 `/z-tmp/backend` 中 wallet-service 模块的逐层拆解分析
> **目的**：取其精华去其糟粕，为 ser-wallet 实现提供有数据支撑的借鉴参考
> **方法**：每个结论附源码行号/SQL/类名佐证，每章配"与我们方案的对比"
> **源码基准**：p9-services/wallet-service（29个Java文件）+ p9-core-lib（共享VO/Enum/Constant）

---

## 第1章：工程全景概述

### 1.1 项目定位

P9是一个**真人视讯 + 竞猜类游戏平台**，核心业务是线上真人荷官游戏（百家乐、龙虎、牛牛、三公、德州扑克、21点等十余种游戏），配套有投注、结算、商户管理、代理体系等模块。

与我们项目的关键定位差异：
- P9的"钱包"本质是**游戏币账户**，核心场景是投注扣款和结算派彩
- 我们的"钱包"是**多币种资产管理**，核心场景是充值、提现、兑换、RPC被调

### 1.2 技术栈

| 维度 | P9工程 | 我们的工程 |
|------|--------|-----------|
| 语言 | Java 8+ | Go 1.21+ |
| Web框架 | Spring Boot | CloudWeGo Hertz (HTTP) |
| RPC框架 | Feign HTTP (JSON) | CloudWeGo Kitex (Thrift IDL) |
| ORM | MyBatis-Plus | GORM Gen |
| 数据库 | MySQL | TiDB (MySQL兼容) |
| 缓存 | Redis (Redisson) | Redis (go-redis) |
| 消息队列 | RabbitMQ | 无(同步RPC) |
| 配置中心 | K8s环境变量 | ETCD |
| 事务管理 | @Transactional注解 | GORM手动Transaction |
| 分布式锁 | Redisson RLock | 无(CAS乐观锁替代) |
| 文档存储 | MongoDB | 无 |

### 1.3 工程模块划分

```
z-tmp/backend/
├── p9-infrastructure/       # 基础设施（2个模块: gateway, eureka-server）
├── p9-services/             # 核心业务服务（8个模块）
│   ├── api-service          # 公共API网关
│   ├── auth-service         # 认证服务
│   ├── merchant-service     # 商户管理（含币种配置）
│   ├── order-service        # 订单服务
│   ├── user-service         # 用户服务
│   ├── wallet-service       # ★ 钱包服务 (Port 8907)
│   ├── rac-service          # 风控服务
│   └── field-service        # 现场管理
├── p9-live-services/        # 实时游戏服务（7个模块）
│   ├── websocket-service    # WebSocket推送
│   ├── common-service       # 公共服务
│   ├── baccarat-service     # 百家乐
│   └── ...                  # 其他游戏服务
├── p9-backstage-services/   # 后台管理（6个模块）
│   ├── admin-service        # 总后台
│   ├── admin-merchant-line-service  # 包网后台
│   └── ...
├── p9-core-lib/             # ★ 共享库（VO/Enum/Constant跨服务复用）
└── p9-job-executor/         # 定时任务
```

**wallet-service在整体架构中的位置**：被7个服务通过Feign HTTP调用，是资金流转的核心节点。消费4个RabbitMQ队列处理异步余额变动。

---

## 第2章：钱包模块代码架构

### 2.1 目录结构

```
wallet-service/src/main/java/com/maya/p9/
├── controller/
│   ├── WalletController.java        # HTTP入口（余额变动 + 冻结操作）
│   └── CoinRecordPageController.java # HTTP入口（流水分页查询）
├── service/
│   ├── CoinRecordService.java       # 接口：核心业务逻辑
│   ├── UserCoinService.java         # 接口：余额查询
│   ├── CommonService.java           # 接口：用户/商户信息获取
│   ├── impl/
│   │   ├── CoinRecordServiceImpl.java   # ★ 核心实现（523行，最关键文件）
│   │   ├── UserCoinServiceImpl.java     # 余额查询实现
│   │   └── CommonServiceImpl.java       # 缓存+Feign获取用户/商户信息
│   └── feign/
│       ├── UserService.java         # → user-service
│       ├── MerchantFeignResource.java # → merchant-service
│       ├── OrderResource.java       # → order-service
│       └── ApiResource.java         # → api-service（三方接口）
├── repository/
│   ├── UserCoinRepository.java      # user_coin表 Mapper
│   ├── CoinRecordRepository.java    # coin_record表 Mapper
│   └── AddCoinMessageRepository.java # add_coin_message表 Mapper (MongoDB)
├── dto/
│   ├── UserCoinDTO.java             # user_coin表实体
│   ├── CoinRecordDTO.java           # coin_record表实体
│   └── ...
├── rabbitmq/
│   ├── listener/
│   │   └── AddCoinListener.java     # MQ消费者（4个队列监听）
│   └── config/
│       └── AddCoinMqConfig.java     # MQ队列配置
└── resources/mapper/
    └── UserCoinMapper.xml           # ★ SQL映射（5条UPDATE语句，45行）
```

### 2.2 核心类职责矩阵

| 类名 | 行数 | 职责 | 关键方法 |
|------|------|------|---------|
| CoinRecordServiceImpl | 523 | 全部余额变动业务逻辑 | addCoin(), deductCoin(), freezeCoin(), subFreezeBalance(), rollbackFreezeBalance() |
| WalletController | 252 | HTTP入口 + Redis锁获取 | addCoin(), freezeCoin(), subFreezeBalance(), rollbackFreezeBalance() |
| AddCoinListener | 148 | MQ消费 + Redis锁获取 | NoticeAddCoinMessage(), noticeDeductCoinOrderMessage() |
| UserCoinServiceImpl | 85 | 余额查询(只读) | getWallet(), getWallListByUserIdList() |
| CommonServiceImpl | 65 | 用户/商户信息获取+缓存 | getUserInfoByUserId(), getMerchantDetailVO() |

### 2.3 共享库 p9-core-lib 的角色

VO（CoinRecordVO, CoinFreezeVO, UserCoinVO）、Enum（WalletUtilEnum, BalanceTypeEnum）、Constant（MqConstants, RedisLockKey）全部放在p9-core-lib中，跨服务共享。

**类比我们的架构**：相当于我们的 `common/pkg/` 或 Thrift IDL定义。区别在于：
- 我们通过Thrift IDL定义接口契约，编译生成代码
- P9通过共享JAR包的方式复用VO，松耦合但版本管理困难

---

## 第3章：数据模型与表结构

### 3.1 user_coin 表（余额表）

```
表名: user_coin
来源: UserCoinDTO.java

字段:
├── id              BIGINT AUTO_INCREMENT  # 主键（来自BasicDataBaseDTO）
├── userId          VARCHAR    # 玩家ID
├── userName        VARCHAR    # 玩家名称
├── amount          DECIMAL    # 账户总金额（BigDecimal）
├── freezeAmount    DECIMAL    # 冻结金额（BigDecimal）
├── userState       INT        # 玩家游戏状态
├── gameType        INT        # 玩家正在进行的游戏
├── walletType      INT        # 钱包类型（1转账/2免转）
├── currency        VARCHAR    # 币种
├── remark          VARCHAR    # 备注
├── createdTime     DATETIME   # 创建时间
└── updatedTime     DATETIME   # 更新时间
```

**与我们 user_balance 的对比**：

| 维度 | P9 user_coin | 我们 user_balance |
|------|-------------|-------------------|
| 行粒度 | 一个用户一行 | 一个用户×一个币种×一个钱包类型 = 一行 |
| 可用余额 | 计算得出: amount - freezeAmount | 独立字段: available |
| 冻结余额 | freezeAmount字段 | frozen字段 |
| 金额类型 | BigDecimal | BIGINT (最小单位) |
| 多币种 | currency字段（但一用户一行意味着只支持单币种） | 天然支持多币种（多行） |
| 版本控制 | 无version字段 | 无(CAS替代) |
| 并发保护 | 无DB层保护(依赖Redis锁) | CAS WHERE条件 |

**关键设计问题**：
1. **单用户单行限制**：一个用户只有一行user_coin，意味着不支持多币种并存（不同于游戏币单一性场景，我们需要多币种）
2. **可用余额不是独立字段**：`available = amount - freezeAmount` 在查询时计算。这意味着冻结操作只需改freezeAmount，但并发风险增加——冻结时不减amount，另一个扣款操作可能用到被冻结的金额（通过 `insufficientBalance` 方法校验弥补，但有时间窗口）

### 3.2 coin_record 表（流水表）

```
表名: coin_record
来源: CoinRecordDTO.java

字段:
├── id              BIGINT AUTO_INCREMENT
├── userId          VARCHAR    # 玩家ID
├── userName        VARCHAR    # 玩家名称（冗余）
├── userType        INT        # 用户类型 0正式 1试玩（冗余）
├── nickName        VARCHAR    # 昵称（冗余）
├── userBelong      INT        # 用户所属（冗余）
├── merchantId      BIGINT     # 商户ID（冗余）
├── merchantName    VARCHAR    # 商户名称（冗余）
├── merchantNo      VARCHAR    # 商户号（冗余）
├── agentNo         VARCHAR    # 代理编号（冗余）
├── agentId         BIGINT     # 代理ID（冗余）
├── agentName       VARCHAR    # 代理名称（冗余）
├── path            VARCHAR    # 上级列表（冗余）
├── orderNo         VARCHAR    # 订单号
├── coinType        INT        # 账变类型(1-9)
├── coinDetail      INT        # 账变明细
├── coinValue       DECIMAL    # 变动金额
├── coinFrom        DECIMAL    # 变前余额
├── coinTo          DECIMAL    # 变后余额
├── coinAmount      DECIMAL    # 当前余额
├── balanceType     INT        # 收支类型 1收入 2支出
├── remark          VARCHAR    # 备注
├── createdTime     DATETIME
└── updatedTime     DATETIME
```

**账变类型枚举**（WalletUtilEnum.CoinChangeEnum）：
```
1=转入, 2=转出, 3=下注, 4=结算(派彩), 5=取消,
6=结算回滚, 7=重算(二次结算), 8=打赏, 9=闪电投注
```

**与我们 balance_flow 的对比**：

| 维度 | P9 coin_record | 我们 balance_flow |
|------|---------------|-------------------|
| coinFrom/coinTo | 变前/变后余额 ✓ | before_balance/after_balance ✓ |
| 唯一键约束 | **无** (致命缺陷) | UNIQUE(ref_order_no, change_type) |
| 冗余字段 | 12个冗余字段(userName等) | 无冗余, 通过userId关联查询 |
| 钱包类型 | 无walletType字段 | 有wallet_type字段 |
| 币种 | 无currency字段 | 有currency字段 |
| 关联订单 | orderNo(无类型区分) | ref_order_no + change_type |

**致命缺陷：无唯一键约束**
coin_record表没有在 `(orderNo, coinType)` 或任何组合上建立UNIQUE KEY。这意味着：
- 同一笔订单的同一种账变可以被重复插入
- DB层无法防止重复记录
- 完全依赖外部锁（Redis）来"间接防重"

### 3.3 币种配置表（在 merchant-service）

```
表名: currency（币种基础信息）
字段: currencyCode, currencyName, symbol, name, proportion, status

表名: currency_rate（汇率配置）
字段: currencyCode, currencyName, currencyValue(BigDecimal), symbol, status
```

**特点**：币种配置不在wallet-service中，而是在merchant-service中管理。wallet通过Feign调用或Redis缓存获取。
**与我们的对比**：我们将币种配置（currency_config）和汇率（exchange_rate）放在ser-wallet内管理，减少跨服务依赖。

---

## 第4章：接口设计与API清单

### 4.1 HTTP接口（WalletController）

| 端点 | 方法 | 功能 | Redis锁 | 备注 |
|------|------|------|---------|------|
| /webapi/wallet/addCoin | POST | 单笔账变(加/减) | 是 | 核心写接口 |
| /webapi/wallet/addCoinList | POST | 批量账变 | 是 | 循环调addCoin |
| /webapi/wallet/getWallet | POST | 查询余额 | 否 | 读操作 |
| /webapi/wallet/users/wallet/list | POST | 批量查询余额 | 否 | 读操作 |
| /webapi/wallet/queryCoinRecordByOrderNo | POST | 按订单号查流水 | 否 | 读操作 |
| /webapi/wallet/queryCoinRecord | POST | 条件查流水 | 否 | 读操作 |
| /webapi/wallet/freezeCoin | POST | 冻结余额 | 是 | 不产生流水 |
| /webapi/wallet/subFreezeBalance | POST | 扣除冻结(确认) | 是 | 产生流水 |
| /webapi/wallet/rollbackFreezeBalance | POST | 回滚冻结(退还) | 是 | 不产生流水 |

### 4.2 MQ接口（AddCoinListener，4个队列）

| 队列 | 场景 | 锁 | 余额通知 | 失败处理 |
|------|------|------|---------|---------|
| ADD_COIN_QUEUE | 单笔加减(通用) | 是 | 是 | 记日志 |
| ADD_COIN_ORDER_QUEUE | 批量扣减(下注) | 否 | 是 | 回滚订单状态 |
| ADD_COIN_NOT_NOTICE_BALANCE_QUEUE | 静默扣减 | 否 | 否 | 回滚订单状态 |
| ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE | 写流水+通知 | 否 | 是 | 记日志(吞没异常) |

### 4.3 接口设计评价

**优点**：
- HTTP + MQ双通道：同步场景走HTTP（查询/冻结），异步场景走MQ（结算/投注扣款）
- 静默扣减队列：避免前端余额频繁跳跃（UX层面考量，值得借鉴）

**缺点**：
- **接口粒度过粗**：`addCoin` 一个接口兼顾收入和支出，通过 `balanceType` 参数区分。不如我们的 `CreditWallet/DebitWallet` 语义清晰
- **缺少语义化命名**：没有明确的"充值入账""投注扣款""结算派彩"专用接口
- **冻结操作无orderNo关联**：`CoinFreezeVO` 只有 `userId + freezeAmount`，无法追溯是哪笔订单的冻结
- **批量操作是循环调单笔**：`addCoinBatch` 内部循环调 `addCoin`，任一失败全部回滚（事务特性保证），但性能不佳

**与我们接口设计的对比**：
- 我们：7个语义明确的RPC接口（CreditWallet, DebitWallet, Freeze, Unfreeze, DeductFrozen, DebitByPriority, SupplementCredit）
- P9：1个通用接口 + 参数区分（addCoin + balanceType + coinType）

---

## 第5章：调用链路与依赖关系

### 5.1 入站依赖（谁调用 wallet-service）

```
wallet-service (Port 8907) 被调用方:
│
├── auth-service ────────── addCoin()
│   └── 场景: 试玩用户登录时初始化2000游戏币
│
├── api-service ─────────── getWallet(), addCoin(), queryCoinRecord()
│   └── 场景: 公共API（第三方/外部对接）
│
├── user-service ────────── getWallet(), addCoin()
│   └── 场景: 用户信息聚合页展示余额
│
├── websocket-service ───── 全量接口(最重调用方)
│   ├── getWallet(), getWalletListByUserId()     → 余额展示
│   ├── addCoin()                                 → 打赏
│   ├── userWithdrawApply()                       → 提现申请
│   ├── createOrder(), queryUserRechargingInfo()  → 充值
│   └── userRecharge(), uploadVoucher()           → 线下充值
│
├── admin-service ───────── getWallet(), coinRecord分页/列表/计数
│   └── 场景: 总后台管理查询
│
├── admin-merchant-line ─── coinRecord分页/列表/计数
│   └── 场景: 包网后台查询
│
├── common-service(live) ── addCoin(), getWallet(), addWithdrawLimitAmount()
│   └── 场景: 直播公共服务（礼物/打赏）
│
└── job-executor ────────── deleteTempTask()
    └── 场景: 定时清理临时数据
```

### 5.2 出站依赖（wallet-service 调用谁）

```
wallet-service 出站调用:
│
├── user-service ── getUserByUserId()
│   └── 获取: userId, userName, walletType, merchantId, currency
│   └── 方式: Feign HTTP, 有Redis缓存(但未回写)
│
├── merchant-service ── getMerchantDetail()
│   └── 获取: 商户配置(addDeductionUrl, walletType等)
│   └── 方式: Feign HTTP, Redis缓存(TTL=1天)
│
├── order-service
│   ├── findOrderByOrderNo()          → 查订单详情
│   ├── saveThirdOrderRecord()        → 保存三方转账记录
│   └── updateOrdersToInitialStatus() → 扣款失败回滚订单
│
└── api-service ── amountTransfer()
    └── 免转钱包: 调第三方API做金额转移
```

### 5.3 MQ消息流

```
生产者 → wallet-service:
├── 结算服务 → ADD_COIN_QUEUE (单笔结算派彩)
├── 投注服务 → ADD_COIN_ORDER_QUEUE (批量下注扣款)
├── 游戏服务 → ADD_COIN_NOT_NOTICE_BALANCE_QUEUE (静默扣款)
└── 其他服务 → ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE (写流水+通知)

wallet-service → 消费者:
├── NOTICE_USER_BALANCE_CHANGE → websocket-service (余额变更推送给用户)
└── EXCHANGE_NOTICE_USER_BALANCE_CHANGE → websocket-service (后台加减款通知)
```

### 5.4 与我们依赖关系的对比

| 维度 | P9 | 我们 |
|------|------|------|
| 入站调用方 | 7个服务, Feign HTTP | 多个服务, Kitex RPC (Thrift IDL) |
| 出站调用 | 4个服务 | 1个(财务模块) |
| 消息队列 | 重度依赖RabbitMQ(4队列) | 无MQ依赖(全同步RPC) |
| 耦合程度 | 高(出站4个 + MQ) | 低(出站仅财务) |
| 通信协议 | HTTP JSON(松类型) | Thrift IDL(强类型编译时校验) |

---

## 第6章：核心业务流程 — addCoin 完整链路 ★★★

### 6.1 入口层（WalletController.addCoin，第50-81行）

```
请求到达 → POST /webapi/wallet/addCoin
│
├── Step1: 参数校验
│   ├── userId 非空
│   └── coinValue != 0
│
├── Step2: 获取Redis分布式锁
│   ├── key: "add.coin.lock.key.{userId}"
│   ├── tryLock(20000ms等待, 30000ms持有)
│   └── 获取失败 → 记日志 → 返回FAIL
│
├── Step3: 调用 coinRecordService.addCoin(coinRecordVO)
│   ├── 成功 → sendUserBalance(MQ通知余额变更)
│   └── 失败 → 返回FAIL
│
└── Step4: finally 释放锁
    └── 检查: rLock != null && isLocked && isHeldByCurrentThread
```

### 6.2 业务层（CoinRecordServiceImpl.addCoin，第168-341行）

```
@Transactional(rollbackFor = Exception.class)
│
├── Step1: coinType校验(>0)
│
├── Step2: ★ RPC获取用户信息 (在事务内!)
│   └── commonService.getUserInfoByUserId(userId)
│       → 查Redis → 未命中 → Feign调user-service
│
├── Step3: 复制属性到DTO
│
├── 分支: walletType判断
│
├── 【TRANSFER钱包路径】(第182-230行)
│   ├── 查user_coin (SELECT * WHERE userId=?)
│   │
│   ├── 用户存在:
│   │   ├── insufficientBalance() 余额校验
│   │   │   └── 仅coinType=2(转出)/3(下注)/8(打赏)校验
│   │   │   └── balance = amount - freezeAmount
│   │   │   └── balance < coinValue → 抛异常
│   │   │
│   │   ├── 收入(balanceType=1):
│   │   │   ├── userCoinRepository.addBalance(id, amount)
│   │   │   └── coinTo = 原amount + coinValue
│   │   │
│   │   └── 支出(balanceType=2):
│   │       ├── userCoinRepository.subBalance(id, amount, coinType)
│   │       ├── affected=0 → 余额不足异常
│   │       └── coinTo = 原amount - coinValue
│   │
│   ├── 用户不存在:
│   │   ├── 校验: 不能是支出(首次不能扣钱)
│   │   ├── INSERT user_coin (userId, amount=coinValue, freezeAmount=0)
│   │   └── coinFrom=0, coinTo=coinValue
│   │
│   ├── 设置 coinFrom = 原amount (变前余额)
│   ├── 设置 coinTo = 计算后余额 (变后余额)
│   └── INSERT coin_record
│
└── 【FREE_TRANSFER钱包路径】(第231-338行)
    ├── 构建三方请求 ThirdOrderRequestVO
    ├── ★ RPC调第三方 apiResource.amountTransfer() (在事务内!)
    ├── 成功 → 记录ThirdOrderRecord + MQ通知余额
    ├── 失败 → 记录失败记录 + return false
    └── INSERT coin_record (coinFrom/coinTo = null)
```

### 6.3 关键问题分析

**问题1：RPC在@Transactional事务范围内** (第176行, 第309行)

```java
// CoinRecordServiceImpl.java:168 — @Transactional开始
@Transactional(rollbackFor = Exception.class)
public boolean addCoin(CoinRecordVO coinRecordVO) {
    // :176 — RPC调用#1: 获取用户信息
    UserInfoVO userInfoVO = commonService.getUserInfoByUserId(coinRecordVO.getUserId());
    // ... 中间业务逻辑 ...
    // :309 — RPC调用#2: 三方金额转移(FREE_TRANSFER路径)
    ResponseVO<ThirdOrderResponseVO> thirdOrderResponse = apiResource.amountTransfer(...);
}
```

后果：
- 数据库连接从事务开始到RPC返回一直被占用
- RPC延迟100-500ms → 事务持续时间不可控
- 连接池（默认10-20个）在高并发下可能耗尽
- RPC失败导致事务回滚，但远端状态可能已变更

**问题2：无幂等检查** (addCoin入口无任何查重逻辑)

```java
// 期望看到但不存在的代码:
// CoinRecordDTO existing = coinRecordRepository.findByOrderNoAndCoinType(orderNo, coinType);
// if (existing != null) return true; // 幂等返回
```

**问题3：无DB层幂等保障** (coin_record表无UNIQUE KEY)

```
-- 期望但不存在的约束:
-- ALTER TABLE coin_record ADD UNIQUE KEY uk_order_coin_type (order_no, coin_type);
```

---

## 第7章：冻结/扣冻/回滚三步操作 ★★★

### 7.1 三步操作SQL（UserCoinMapper.xml）

```xml
<!-- 冻结: 只增加freeze_amount, 不减amount -->
<update id="addFreezeBalance">
    UPDATE user_coin b
    SET b.freeze_amount = b.freeze_amount + #{vo.amount},
        b.updated_time = #{vo.updateTime}
    WHERE b.id = #{vo.id}
</update>

<!-- 扣除冻结(确认): amount和freeze_amount同时减少 -->
<update id="subFreezeBalance">
    UPDATE user_coin b
    SET b.amount        = b.amount - #{vo.amount},
        b.freeze_amount = b.freeze_amount - #{vo.amount},
        b.updated_time  = #{vo.updateTime}
    WHERE b.id = #{vo.id}
</update>

<!-- 回滚冻结(退还): 只减少freeze_amount -->
<update id="rollbackFreezeBalance">
    UPDATE user_coin b
    SET b.freeze_amount = b.freeze_amount - #{vo.amount},
        b.updated_time  = #{vo.updateTime}
    WHERE b.id = #{vo.id}
</update>
```

### 7.2 冻结模型语义分析

P9的冻结模型与我们有**本质区别**：

| 操作 | P9 (amount/freezeAmount) | 我们 (available/frozen) |
|------|-------------------------|------------------------|
| 冻结 | amount不变, freezeAmount += X | available -= X, frozen += X |
| 扣除冻结 | amount -= X, freezeAmount -= X | frozen -= X |
| 回滚冻结 | amount不变, freezeAmount -= X | available += X, frozen -= X |
| 可用余额 | 查询时计算: amount - freezeAmount | 直接读: available |

**P9模型的含义**：`amount` 是"总资产"，`freezeAmount` 是"被冻结的部分"，可用 = 总资产 - 冻结。冻结只是"标记"一部分不可用，不改变总额。

**我们模型的含义**：`available` 是"可用余额"，`frozen` 是"冻结余额"。冻结时从可用移到冻结，是真实的"资金转移"。

### 7.3 P9冻结模型的隐患

**隐患1：冻结时未检查已冻结总额**

```java
// CoinRecordServiceImpl.java:434
if (userCoinDTO.getAmount().compareTo(coinFreezeVO.getFreezeAmount()) < 0) {
    return ErrorCode.USER_WALLET_NOT_ENOUGH_BALANCE;
}
```

校验的是 `amount >= 要冻结金额`，但没有考虑已有的freezeAmount。如果amount=1000, 已冻结500, 再冻结800：
- P9判断: 1000 >= 800 → 通过! (实际可用只有500)
- 我们判断: available(500) >= 800 → 拒绝 (正确)

**隐患2：冻结和回滚不产生流水**

```java
// freezeCoin() 第425-446行: 无 coin_record 插入
// rollbackFreezeBalance() 第476-493行: 无 coin_record 插入
// 只有 subFreezeBalance() 第449-473行: 调用 createOrder() 插入流水
```

这意味着：冻结和回滚操作无审计记录，无法追溯资金冻结/退还的历史。

**隐患3：SQL无WHERE条件保护**

三条SQL都是 `WHERE b.id = #{vo.id}`，没有 `AND freeze_amount >= #{amount}` 这样的条件检查。freeze_amount可以被减为负数。

### 7.4 与我们冻结方案的对比总结

| 维度 | P9方案 | 我们方案 | 评价 |
|------|--------|---------|------|
| 可用余额准确性 | 计算得出(有并发风险) | DB直接存储 | 我们更安全 |
| 冻结时余额校验 | 不考虑已冻结额 | available >= 冻结额 | 我们更准确 |
| 流水记录完整性 | 仅扣除冻结有流水 | 三步全有流水 | 我们更完整 |
| SQL安全性 | 无WHERE保护 | CAS条件(frozen>=?) | 我们更安全 |
| 订单关联 | 无orderNo | 有ref_order_no | 我们可追溯 |

---

## 第8章：并发安全机制 — Redis分布式锁模式 ★★★

### 8.1 锁实现细节

```java
// WalletController.java:60-78
RLock rLock = redisUtil.getLock(RedisLockKey.getAddCoinLockKey(coinRecordVO.getUserId()));
try {
    if (rLock.tryLock(20000, 30000L, TimeUnit.MILLISECONDS)) {
        // 20000ms = 最多等待20秒获取锁
        // 30000ms = 锁自动释放时间30秒
        boolean result = coinRecordService.addCoin(coinRecordVO);
        // ...
    } else {
        log.info("锁超时：{}", coinRecordVO.getOrderNo());
        // 获取锁失败 → 返回通用FAIL（不是"请重试"而是"失败"）
    }
} finally {
    if (Objects.nonNull(rLock) && rLock.isLocked() && rLock.isHeldByCurrentThread()) {
        rLock.unlock();
    }
}
```

### 8.2 锁的使用范围

**所有写操作入口都加锁**：
- WalletController: addCoin, addCoinList, freezeCoin, subFreezeBalance, rollbackFreezeBalance
- AddCoinListener: NoticeAddCoinMessage (ADD_COIN_QUEUE)

**读操作不加锁**：getWallet, queryCoinRecord

**锁粒度**：per-userId。Key格式：`add.coin.lock.key.{userId}`
含义：同一用户的所有写操作**串行执行**——包括加款、扣款、冻结、解冻全部互斥。

### 8.3 Redis锁 vs CAS 深度对比

| 维度 | Redis分布式锁 (P9) | CAS乐观锁 (我们) |
|------|-------------------|-----------------|
| 锁粒度 | 用户级（粗粒度） | 行级SQL WHERE条件（细粒度） |
| 并发能力 | 同用户所有操作串行 | 同用户不同币种/钱包可并行 |
| 吞吐量 | 低（串行化） | 高（无锁等待） |
| 额外依赖 | Redis必须可用 | 无额外依赖 |
| 故障影响 | Redis宕机→所有写操作阻塞 | 无单点故障 |
| 性能开销 | 网络往返(获取锁+释放锁) | 无额外开销 |
| 死锁风险 | Redisson看门狗机制缓解，但网络分区仍有风险 | 无死锁可能 |
| DB层保障 | 无（SQL无WHERE保护） | 有（affected_rows=0即失败） |
| 锁等待超时 | 最多20s（用户体验差） | 无等待，立即返回结果 |
| 实现复杂度 | 低（每个入口加锁代码模板化） | 中（需精心设计SQL条件） |

### 8.4 P9方案的根本问题

**Redis锁只保护了并发，但DB层完全不设防**：

```xml
<!-- subBalance SQL: -->
UPDATE user_coin b SET b.amount = b.amount - #{vo.amount}
WHERE b.id = #{vo.id}
<!-- 注意: coinType 5,6 跳过了 (amount - #{amount}) >= 0 检查 -->
<!-- 这意味着: 如果Redis锁失效, amount可以被减为负数 -->

<!-- addBalance SQL: -->
UPDATE user_coin b SET b.amount = b.amount + #{vo.amount}
WHERE b.id = #{vo.id}
<!-- 无任何校验条件, 直接加钱 -->
```

如果发生以下情况，DB层没有任何兜底：
1. Redis网络分区 → 两个实例同时认为自己持有锁
2. 锁过期（30s）但业务未完成（RPC在事务内导致超时）→ 另一个线程获取锁
3. Redisson客户端Bug → 锁提前释放

**我们的CAS方案**：即使没有任何外部锁，SQL本身就是安全的：
```sql
UPDATE user_balance SET available = available - ?
WHERE user_id = ? AND currency = ? AND wallet_type = ? AND available >= ?
-- affected_rows = 0 → 余额不足，业务层返回错误
```

---

## 第9章：金额处理与精度策略 ★★

### 9.1 P9的BigDecimal方案

**全链路使用BigDecimal**：
- DB字段：`amount DECIMAL`, `freezeAmount DECIMAL`, `coinValue DECIMAL`
- Java对象：`BigDecimal amount`, `BigDecimal coinValue`
- API传输：JSON直接序列化BigDecimal
- 计算：`amount.add(coinValue)`, `amount.subtract(coinValue)`

**比较操作**：
```java
// CoinRecordServiceImpl.java:113
if (coinRecordVO.getCoinValue().compareTo(BigDecimal.ZERO) <= 0) { ... }

// :377 余额校验
if (balance.compareTo(amount) < 0) { throw ... }
```

### 9.2 BigDecimal vs BIGINT 对比

| 维度 | BigDecimal (P9) | BIGINT最小单位 (我们) |
|------|----------------|---------------------|
| 精度安全 | 高(任意精度) | 高(整数无精度损失) |
| 性能 | 较慢(对象创建/GC开销) | 快(CPU原生int64运算) |
| 存储 | 变长(DECIMAL) | 固定8字节(BIGINT) |
| 编码复杂度 | 低(直接用) | 中(需要ToMinUnit/ToDisplayAmount换算) |
| 常见Bug | equals vs compareTo | 溢出(极端情况) |
| JSON序列化 | 有精度丢失风险(double中转) | 安全(int64直接传) |
| 跨语言兼容 | 差(每种语言BigDecimal不同) | 好(int64是通用类型) |
| 数据库索引效率 | 较差(DECIMAL变长) | 好(BIGINT定长) |

### 9.3 P9方案的一个编码陷阱

```java
// CoinRecordServiceImpl.java:180
BigDecimal coinTo = new BigDecimal("0");  // ✓ 正确用法

// 但在其他地方:
BigDecimal totalAmount = new BigDecimal("0");  // :258
totalAmount = totalAmount.add(coinRecordVO.getCoinValue());  // :260
// BigDecimal是不可变对象, add()返回新对象, 原对象不变
// 如果写成 totalAmount.add(coinRecordVO.getCoinValue()) 不赋值, 就是Bug
```

### 9.4 对我们的启示

P9的BigDecimal方案在**单语言(Java)、单币种**场景下可行，但我们的BIGINT方案更适合：
- 多币种(不同precision: VND=0, THB=2)
- 跨语言(Go→Thrift IDL→可能的前端)
- 高性能需求(int64运算零开销)

---

## 第10章：幂等性与重复防护 ★★★

### 10.1 P9的幂等现状

**结论：几乎无幂等保障**

逐层检查：

| 层级 | 机制 | P9是否有 |
|------|------|---------|
| 应用层查重 | 查流水表判断是否已处理 | **无** |
| DB唯一键 | coin_record表UNIQUE KEY | **无** |
| Redis锁防并发 | 串行化同一用户请求 | 有（但这不是幂等） |
| MQ消息去重 | add_coin_message表(MongoDB) | 有迹象但未在核心代码中使用 |

### 10.2 缺乏幂等的后果分析

**场景1：MQ重复投递**
```
投注服务 → ADD_COIN_ORDER_QUEUE → 消息投递2次
→ AddCoinListener消费2次
→ 每次都成功获取锁
→ 每次都执行 deductCoinBatch()
→ 用户余额被扣减2次!
```

**场景2：Feign重试**
```
websocket-service → POST /addCoin → 网络超时(实际已执行成功)
→ Feign自动重试
→ 再次执行addCoin
→ 用户余额被操作2次!
```

**场景3：前端重复提交**
```
用户快速点击"打赏" → 两个请求先后到达
→ 第一个获取锁执行成功
→ 第二个等待20s后获取锁也执行成功
→ 打赏金额翻倍!
```

### 10.3 Redis锁 ≠ 幂等

这是一个常见误区。Redis锁解决的是**并发**问题（同一时刻只有一个操作执行），但不解决**重复**问题（同一请求先后执行两次）。

```
时间线:
T1: 请求A获取锁 → T2: 请求A执行完释放锁 → T3: 请求A'(重试)获取锁 → T4: 请求A'执行完释放锁
└── 锁保证了T1-T2期间没有其他请求干扰
└── 但T3-T4是新的锁周期，无法识别A'是A的重试
```

### 10.4 与我们双层幂等方案的对比

```
我们的方案:
请求到达 → 查balance_flow(orderNo, changeType)
         → 已存在: 返回上次结果(幂等返回)
         → 不存在: 继续执行 → 写balance_flow
                             → 唯一键冲突: 视为幂等成功
                             → 写入成功: 正常返回

P9的方案:
请求到达 → 获取Redis锁
         → 直接执行addCoin
         → 无任何查重
         → 写coin_record(无唯一键)
         → 释放锁
```

**核心教训**：幂等 = 同一请求无论执行多少次，结果都一样。这需要**识别请求身份**(唯一键)，不是**控制执行顺序**(锁)。

---

## 第11章：事务边界与一致性 ★★★

### 11.1 P9的事务模式

```java
// 方式: Spring @Transactional 注解
@Transactional(rollbackFor = Exception.class)  // addCoin
@Transactional(rollbackFor = Throwable.class)  // deductCoin
@Transactional                                  // freezeCoin, addCoinBatch, deductCoinBatch
```

### 11.2 事务范围内的RPC调用（4处违规）

| 位置 | RPC调用 | 目标服务 | 风险 |
|------|---------|---------|------|
| addCoin:176 | getUserInfoByUserId() | user-service | 延迟50-200ms |
| addCoin:241 | findOrderByOrderNo() | order-service | 延迟50-200ms |
| addCoin:309 | amountTransfer() | api-service(三方) | 延迟200-2000ms |
| addCoin:315 | saveThirdOrderRecord() | order-service | 延迟50-200ms |

**最坏情况**：FREE_TRANSFER路径下，一个addCoin事务可能持有DB连接长达3000ms+：
```
getUserInfoByUserId(200ms) + amountTransfer(2000ms) + saveThirdOrderRecord(200ms) + DB操作(50ms)
= 约2450ms 事务持续时间
```

对比我们的铁律：**事务内只做DB操作，RPC一律在事务外**。

### 11.3 P9的一致性三件套分析

| 三件套 | P9是否保障 | 说明 |
|--------|----------|------|
| 余额变动 + 流水 同事务 | 部分 ✓ | addCoin中有, freezeCoin/rollback中无 |
| 流水 before/after 快照 | ✓ | coinFrom/coinTo (第199-211行) |
| 订单状态同事务 | ✗ | 订单在order-service, 不在同一事务 |

**快照机制实现**（正确的部分）：
```java
// CoinRecordServiceImpl.java:199-211
// 收入:
coinTo = userCoinDTO1.getAmount().add(coinRecordVO.getCoinValue());
coinRecordDTO.setCoinFrom(userCoinDTO1.getAmount());  // 变前
coinRecordDTO.setCoinTo(coinTo);                       // 变后

// 支出:
coinTo = userCoinDTO1.getAmount().subtract(coinRecordVO.getCoinValue());
coinRecordDTO.setCoinFrom(userCoinDTO1.getAmount());  // 变前
coinRecordDTO.setCoinTo(coinTo);                       // 变后
```

**问题**：这里的 `userCoinDTO1.getAmount()` 是事务开始时读取的值，但没有使用 `SELECT ... FOR UPDATE`。在Redis锁失效的情况下，可能读到脏数据。

### 11.4 与我们方案的对比

| 维度 | P9 | 我们 |
|------|------|------|
| 事务内RPC | 有(4处) | 严禁(铁律) |
| 事务持续时间 | 不可控(取决于RPC) | 可控(仅DB操作, 1-5ms) |
| 三件套完整性 | 部分(冻结无流水) | 完整(所有操作都有) |
| 快照一致性 | 无FOR UPDATE | 有精确快照 |
| 订单状态同步 | 跨服务(不同事务) | 同事务 |

---

## 第12章：缓存策略与数据获取 ★

### 12.1 用户信息获取（CommonServiceImpl.getUserInfoByUserId）

```java
// CommonServiceImpl.java:37-52
String redisKey = RedisConstants.CURRENCY_LIMIT_ID_MERCHANT_ID + userId;
Object obj = redisUtil.get(redisKey);
if (obj instanceof UserInfoVO) {
    return (UserInfoVO) obj;
}
// 未命中 → Feign调user-service
ResponseEntity<ResponseVO<UserInfoVO>> userInfoResponseVO = userResource.getUserByUserId(userId);
// ★ 注意: 查到后没有回写Redis！
return userInfoVO;
```

**问题**：查询Redis未命中后，结果没有回写缓存。这意味着只有其他地方写入的缓存才会被命中，用户信息每次可能都走Feign调用。

### 12.2 商户信息获取（CommonServiceImpl.getMerchantDetailVO）

```java
// CommonServiceImpl.java:22-35
String redisKey = MerchantCacheBusiness.getRedisKeyForGetMerchantDetailVO(merchantId);
Object obj = redisUtil.get(redisKey);
if (obj instanceof MerchantDetailVO) {
    return (MerchantDetailVO) obj;
}
// 未命中 → Feign调merchant-service
MerchantDetailVO merchantDetailVO = merchantResource.getMerchantDetail(idVO).getBody().getData();
// ✓ 有回写Redis, TTL=1天
redisUtil.set(redisKey, merchantDetailVO, 1, TimeUnit.DAYS);
```

### 12.3 币种配置缓存（在merchant-service中）

```
Redis Keys:
├── "merchantCurrency"                      → 全部汇率列表JSON
├── "findRateByCurrencyCode:{currencyCode}" → 单个汇率JSON
└── "merchantRateCurrencyCode:{currencyCode}" → 单个汇率JSON

更新时机: admin编辑汇率 → DB更新 → Redis刷新 → MQ通知
启动时机: merchant-service启动 → initCurrencyRedis() + setRedCurrency()
```

### 12.4 与我们Cache-Aside模式的对比

| 维度 | P9 | 我们 |
|------|------|------|
| 缓存模式 | 不统一(有的写回有的不写回) | 标准Cache-Aside |
| 用户信息 | 不回写(每次可能走Feign) | 不缓存(我们不需要跨服务取) |
| 商户信息 | 回写(TTL=1天) | 不适用 |
| 币种配置 | 在merchant-service中管理 | 在ser-wallet内管理(TTL=5min) |
| 余额缓存 | 不缓存(正确) | 不缓存(正确) |
| 缓存失效 | 启动初始化 + 手动刷新 | 写操作后defer删缓存 |

---

## 第13章：MQ消息驱动架构 ★★

### 13.1 四个队列的完整职责

**队列1：ADD_COIN_QUEUE（通用加减）**
```java
// AddCoinListener.java:37-67
@RabbitListener(queues = {MqConstants.ADD_COIN_QUEUE})
public void NoticeAddCoinMessage(String jsonStr) {
    CoinRecordVO coinRecordVO = JSONObject.parseObject(jsonStr, CoinRecordVO.class);
    // 参数校验 + 获取锁
    RLock rLock = redisUtil.getLock(RedisLockKey.getAddCoinLockKey(coinRecordVO.getUserId()));
    if (rLock.tryLock(20000, 30000L, TimeUnit.MILLISECONDS)) {
        boolean isSuc = coinRecordService.addCoin(coinRecordVO);
        if (isSuc) {
            coinRecordService.sendUserBalance(coinRecordVO.getUserId());  // 通知余额
        }
    }
    // 异常: catch + 记日志 (消息不会重试)
}
```

**队列2：ADD_COIN_ORDER_QUEUE（批量下注扣减）**
```java
// AddCoinListener.java:70-100
@RabbitListener(queues = {MqConstants.ADD_COIN_ORDER_QUEUE})
public void noticeDeductCoinOrderMessage(String jsonStr) {
    List<CoinRecordVO> coinRecordVOList = JSON.parseArray(jsonStr, CoinRecordVO.class);
    boolean result = coinRecordService.deductCoinBatch(coinRecordVOList);
    if (!result) {
        // ★ 补偿: 扣减失败 → 回滚订单状态到初始
        List<String> orderNos = coinRecordVOList.stream()
            .map(CoinRecordVO::getOrderNo).collect(Collectors.toList());
        orderResource.updateOrdersToInitialStatus(orderNos);
    }
    coinRecordService.sendUserBalance(...);  // 无论成败都通知余额
}
// 异常处理: catch中也回滚订单状态
```

**队列3：ADD_COIN_NOT_NOTICE_BALANCE_QUEUE（静默扣减）**
- 与队列2几乎相同，区别在于：不调 `sendUserBalance()`
- 场景：避免前端余额频繁跳跃（下注过程中间态不展示给用户）

**队列4：ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE（写流水+通知）**
```java
// AddCoinListener.java:133-144
@RabbitListener(queues = {MqConstants.ADD_COIN_RECORD_AND_NOTICE_BALANCE_QUEUE})
public void addCoinRecordAndNoticeBalanceQueue(String jsonStr) {
    List<CoinRecordVO> coinRecordVOList = JSON.parseArray(jsonStr, CoinRecordVO.class);
    coinRecordService.changeCoinAndNoticeBalance(coinRecordVOList);
    // ★ 问题: catch中只记日志, 异常被完全吞没
}
```

### 13.2 MQ模式的优劣分析

**优点**：
1. **异步解耦**：结算/投注服务不等待wallet的同步响应，降低链路延迟
2. **静默扣减**：UX层面考量，下注中间过程不展示余额变化，避免频繁跳跃
3. **批量处理**：一次MQ消息携带多笔扣减，减少网络往返
4. **补偿式错误处理**：扣减失败 → 回滚订单状态（有业务级补偿意识）

**缺点**：
1. **消息丢失风险**：catch异常后仅记日志，不重试不重发，消息等于丢失
2. **ACK机制不明确**：未见显式ACK/NACK，默认auto-ack可能在处理中途消息已确认
3. **无消费幂等**：MQ重复投递会导致重复执行（回到第10章的问题）
4. **日志打印Bug**：第137行 `log.info("收到消息: {}", coinRecordVOList)` 在 `parseArray` 之前打印，此时list还是null

### 13.3 对我们的借鉴

我们目前无MQ依赖（全同步RPC），但以下思路值得保留：
- **余额变更通知**：如果C端需要实时看到余额变化，可以在余额变动成功后通过WebSocket推送
- **静默操作**：部分中间状态可以不立即刷新前端（如兑换过程中的中间态）
- **批量失败补偿**：批量操作中任一失败后的回滚/补偿策略

---

## 第14章：安全与业务校验 ★

### 14.1 余额校验规则

```java
// CoinRecordServiceImpl.java:366-384 insufficientBalance()
// 仅以下三种coinType校验余额:
// 2=转出, 3=下注, 8=打赏
if (TRANSFER_OUT.getCode().equals(coinType)
    || BET.getCode().equals(coinType)
    || SEND_REWARD.getCode().equals(coinType)) {
    BigDecimal balance = userCoinAmount.subtract(freezeAmount);  // 可用 = 总额 - 冻结
    if (balance.compareTo(amount) < 0) {
        throw new MayaDefaultException(ErrorCode.AMOUNT_NOT_ENOUGH);
    }
}
```

**不校验余额的类型**：
- 4=结算(派彩): 只加钱，不需要校验
- 5=取消, 6=结算回滚: 设计意图是允许减为负（纠错场景）

```xml
<!-- UserCoinMapper.xml:4-13 subBalance -->
<if test=" vo.coinType != 6 and vo.coinType != 5 ">
    AND (b.amount - #{vo.amount}) >= 0
</if>
<!-- coinType=5(取消)和6(结算回滚) 跳过余额检查 -->
```

**评价**：允许特定类型减为负有合理性（纠错/回滚场景），但应该有更严格的控制（如只有管理员操作才允许）。

### 14.2 安全缺陷清单

| 缺陷 | 严重度 | 说明 |
|------|--------|------|
| 无C端身份校验 | ★★★ | userId来自请求参数，非从Token/Session提取 |
| 无敏感数据加密 | ★★ | 余额、流水金额明文存储（游戏币场景可接受） |
| 无审计日志 | ★★ | B端操作无日志记录 |
| 无限流/防刷 | ★★ | 无请求频率限制（依赖Redis锁的20s等待间接限流） |
| 无金额上限校验 | ★ | 只校验>0, 不校验上限 |
| 无币种有效性校验 | ★ | 不检查currency是否为启用状态 |
| 异常信息泄露 | ★ | Exception消息直接返回给调用方 |

### 14.3 与我们方案的对比

| 维度 | P9 | 我们 |
|------|------|------|
| C端身份 | 信任入参userId | tracer.GetUserId(ctx) 从Token提取 |
| 敏感加密 | 无 | AES加密(提现账户等) |
| B端鉴权 | 不明确 | BackAuthMiddleware + RBAC |
| 审计日志 | 无 | BLogClient().AddActLog() |
| 防刷限流 | 无(Redis锁间接) | 预留(中间件级) |
| 重复转账拦截 | 无 | 三级拦截(提醒/警告/阻断) |

---

## 第15章：综合评价 — 精华与糟粕清单 ★★★

### 15.1 值得借鉴的精华（按优先级）

**1. coinFrom/coinTo 快照模式** ★★★
- 位置：CoinRecordServiceImpl.java:199-211
- 做法：写流水时记录变前(coinFrom)和变后(coinTo)余额
- 借鉴：与我们的 before_balance/after_balance 异曲同工，**验证了方向正确**
- 差异：P9无FOR UPDATE保证读一致性，我们应使用事务内读取

**2. 冻结三步操作概念** ★★★
- 位置：UserCoinMapper.xml (3条SQL) + CoinRecordServiceImpl.java:425-493
- 做法：freeze → subFreeze(确认扣除) → rollbackFreeze(取消退还)
- 借鉴：三步模型的抽象正确，与我们 Freeze/DeductFrozen/Unfreeze 一致
- 改进：P9实现有缺陷(无流水/无WHERE保护)，我们已在设计中修正

**3. MQ余额变更通知** ★★
- 位置：CoinRecordServiceImpl.java:152-165, MqConstants.java
- 做法：余额变动后通过MQ通知WebSocket推送给前端
- 借鉴：异步通知模式解耦了钱包核心逻辑和前端展示
- 适配：我们暂无MQ，可在余额变动成功后通过其他方式通知C端

**4. 静默扣减队列** ★★
- 位置：AddCoinListener.java:102-131 (ADD_COIN_NOT_NOTICE_BALANCE_QUEUE)
- 做法：扣减成功但不通知前端余额变化，避免游戏中余额频繁跳跃
- 借鉴：UX层面的考量，某些中间态操作不应该立即反映到前端余额展示

**5. HTTP + MQ 双通道入口** ★★
- 做法：同步场景(查询/冻结)走HTTP，异步场景(结算/投注)走MQ
- 借鉴：根据业务时效性选择通信方式的思路
- 适配：我们全同步RPC，如果未来有异步场景可参考此模式

**6. 新用户首次充值自动创建余额行** ★
- 位置：CoinRecordServiceImpl.java:212-227
- 做法：user_coin行不存在时自动INSERT（校验首次必须是收入）
- 借鉴：首次余额行创建的防护逻辑（不允许首次为扣款）
- 适配：我们可在首次CreditWallet时使用 INSERT ON DUPLICATE KEY

**7. 下注扣减失败的订单回滚** ★
- 位置：AddCoinListener.java:88-99
- 做法：扣减失败 → 调 orderResource.updateOrdersToInitialStatus() 回滚订单
- 借鉴：跨服务补偿的简单实现，失败时主动通知上游回滚

### 15.2 必须规避的糟粕（按严重程度）

**1. 无DB级幂等保障** ★★★★
- 表现：coin_record表无UNIQUE KEY约束
- 后果：重复请求 = 重复扣款/加款（资金安全隐患）
- 我们的做法：balance_flow UNIQUE KEY (ref_order_no, change_type)

**2. RPC在事务内** ★★★★
- 表现：addCoin @Transactional范围内有4处RPC调用
- 后果：长事务(最差3000ms+) → 连接池耗尽 → 服务雪崩
- 我们的做法：铁律——RPC在事务外，事务内只做DB操作

**3. Redis锁替代CAS** ★★★
- 表现：所有写操作依赖Redisson RLock，SQL无WHERE条件保护
- 后果：Redis故障 → 并发安全失效 → DB层无兜底
- 我们的做法：CAS乐观锁(SQL WHERE条件)，无外部依赖

**4. 冻结操作无流水记录** ★★★
- 表现：freezeCoin/rollbackFreezeBalance不产生coin_record
- 后果：冻结/退还操作无法审计追溯
- 我们的做法：三步操作全部产生balance_flow

**5. SQL按主键更新** ★★★
- 表现：`WHERE b.id = #{vo.id}` 而非 `WHERE userId AND currency AND walletType`
- 后果：业务键不参与校验，如果id错误则操作错误用户
- 我们的做法：WHERE条件包含完整业务键 + 余额条件

**6. 可用余额计算而非存储** ★★
- 表现：可用 = amount - freezeAmount（查询时计算）
- 后果：冻结校验时不考虑已冻结额，可能超冻
- 我们的做法：available是独立字段，冻结时原子性减available加frozen

**7. 部分类型可减为负** ★★
- 表现：coinType=5(取消)和6(结算回滚)跳过余额检查
- 后果：特定类型可以将余额减为负数
- 评价：有合理性（纠错场景），但缺少权限控制

**8. MQ消费异常吞没** ★★
- 表现：catch异常后仅记日志，不重试不重发
- 后果：消息处理失败 = 消息丢失 = 余额变动丢失
- 建议：应使用手动ACK + 死信队列 + 重试机制

**9. 无C端身份校验** ★
- 表现：userId来自请求参数，非从Token提取
- 后果：接口调用方可以伪造userId操作他人余额
- 我们的做法：tracer.GetUserId(ctx) 强制从Token中提取

**10. BigDecimal而非BIGINT** ★
- 表现：全程BigDecimal
- 后果：性能开销(对象创建/GC)，跨语言兼容性差
- 评价：在Java单语言场景下可接受，但我们Go多币种场景BIGINT更优

### 15.3 综合评分矩阵

```
维度              P9评分  说明
──────────────────────────────────────────────────────
架构设计           6/10   三层清晰, HTTP+MQ双通道好, 但职责混乱(Service层过重)
数据模型           5/10   基本功能有, 但缺唯一键/多币种/索引设计
并发安全           4/10   有意识(Redis锁), 但方案脆弱(DB层不设防)
幂等保障           2/10   几乎无保障, 依赖外部串行化间接防重
事务管理           3/10   违反基本原则(RPC在事务内), 冻结无流水
安全防护           3/10   缺少关键防护(身份校验/加密/审计)
可扩展性           5/10   MQ解耦好, 但单用户单行不支持多币种
代码质量           5/10   可读性尚可, 但重复代码多/异常处理粗糙
──────────────────────────────────────────────────────
综合               4.1/10
```

**结论**：P9的钱包模块是一个**功能可用但工程质量欠佳**的实现。在游戏币单一场景下勉强可用，但在我们的**多币种资产管理**场景下，其核心机制（并发/幂等/事务/安全）均不满足要求。值得借鉴的主要是**概念层面**的设计（冻结模型、快照机制、通知模式），而**实现层面**需要完全重做。

---

## 附录A：P9关键源码节选与注释

### A.1 addCoin核心路径（精简注释版）

```java
// CoinRecordServiceImpl.java:168-341 (精简至核心逻辑)

@Transactional(rollbackFor = Exception.class)
public boolean addCoin(CoinRecordVO vo) {
    // [1] RPC: 获取用户信息 ← ★违规: 在事务内
    UserInfoVO userInfo = commonService.getUserInfoByUserId(vo.getUserId());

    if (userInfo.getWalletType() == TRANSFER) {
        // [2] 查余额行
        UserCoinDTO userCoin = userCoinRepository.selectOne(WHERE userId = vo.getUserId());

        if (userCoin != null) {
            // [3] 余额校验 (仅转出/下注/打赏)
            insufficientBalance(vo, userCoin);  // ← 可用 = amount - freezeAmount

            if (vo.getBalanceType() == INCOME) {
                // [4a] 加钱: UPDATE SET amount = amount + ?
                userCoinRepository.addBalance(id, amount);
                coinTo = amount + coinValue;
            } else {
                // [4b] 扣钱: UPDATE SET amount = amount - ? WHERE (amount-?)>=0
                int affected = userCoinRepository.subBalance(id, amount, coinType);
                if (affected == 0) throw AMOUNT_NOT_ENOUGH;  // ← 唯一的DB层保护
                coinTo = amount - coinValue;
            }
            // [5] 设置快照
            coinFrom = userCoin.getAmount();  // ← 变前(无FOR UPDATE)
            coinTo = 计算后余额;                // ← 变后
        } else {
            // [6] 首次: INSERT user_coin
            checkCoinChange(vo);  // 首次不允许支出
            userCoinRepository.insert(new UserCoinDTO(userId, coinValue, 0));
            coinFrom = 0;
            coinTo = coinValue;
        }
        // [7] 写流水 ← 无幂等检查, 无唯一键约束
        coinRecordRepository.insert(coinRecordDTO);
    }
    return true;
}
```

### A.2 UserCoinMapper.xml 5条SQL完整注释

```xml
<!-- SQL-1: 扣减余额 -->
<!-- 关键: WHERE id (非业务键), coinType=5/6跳过余额检查 -->
UPDATE user_coin SET amount = amount - #{amount}
WHERE id = #{id} [AND (amount - #{amount}) >= 0]  -- 条件动态

<!-- SQL-2: 增加余额 -->
<!-- 关键: 无任何校验条件, 直接加钱 -->
UPDATE user_coin SET amount = amount + #{amount} WHERE id = #{id}

<!-- SQL-3: 增加冻结 -->
<!-- 关键: 只加freeze_amount, 不减amount, 无上限检查 -->
UPDATE user_coin SET freeze_amount = freeze_amount + #{amount} WHERE id = #{id}

<!-- SQL-4: 扣除冻结(确认) -->
<!-- 关键: amount和freeze_amount同时减少, 无下限检查 -->
UPDATE user_coin SET amount = amount - #{amount},
       freeze_amount = freeze_amount - #{amount} WHERE id = #{id}

<!-- SQL-5: 回滚冻结(退还) -->
<!-- 关键: 只减freeze_amount, amount不变, 无下限检查 -->
UPDATE user_coin SET freeze_amount = freeze_amount - #{amount} WHERE id = #{id}
```

### A.3 WalletController 锁模式（模板化）

```java
// 每个写操作入口的固定模板:
RLock rLock = redisUtil.getLock(RedisLockKey.getAddCoinLockKey(userId));
try {
    if (rLock.tryLock(20000, 30000L, TimeUnit.MILLISECONDS)) {
        // 执行业务逻辑
    } else {
        log.info("锁超时：{}", orderNo);
        // ★ 返回通用FAIL, 不区分"锁竞争失败"和"业务失败"
    }
} catch (Exception e) {
    log.error(e.getMessage(), e);
    // ★ 异常被catch, 不上抛, 返回FAIL
} finally {
    if (rLock != null && rLock.isLocked() && rLock.isHeldByCurrentThread()) {
        rLock.unlock();
    }
}
// ★ 所有异常路径都返回同一个FAIL码, 调用方无法区分原因
```

---

## 附录B：P9 vs 我们设计方案对照速查表

| # | 维度 | P9方案 | 我们方案 | 取舍 |
|---|------|--------|---------|------|
| 1 | **并发安全** | Redis分布式锁(per-userId串行) | CAS乐观锁(SQL WHERE条件) | ✗ 规避Redis锁, ✓ 用CAS |
| 2 | **幂等保障** | 无(依赖锁间接防重) | 双层(应用层查流水 + DB唯一键) | ✗ 完全规避, ✓ 用双层 |
| 3 | **金额精度** | BigDecimal全程 | BIGINT最小单位 | ✗ 规避BigDecimal, ✓ 用BIGINT |
| 4 | **事务边界** | @Transactional含RPC | 铁律: RPC在事务外 | ✗ 严格规避, ✓ 遵守铁律 |
| 5 | **冻结模型** | freeze/subFreeze/rollback概念 | Freeze/DeductFrozen/Unfreeze | △ 借鉴概念, 改进实现 |
| 6 | **缓存策略** | 不统一(部分回写/部分不) | 标准Cache-Aside | ✗ 规避不统一, ✓ 标准化 |
| 7 | **消息驱动** | RabbitMQ 4队列 | 无MQ(全同步RPC) | △ 借鉴通知思路, 暂不引入MQ |
| 8 | **安全防护** | 信任入参userId, 无加密 | Token提取userId, AES加密 | ✗ 完全规避, ✓ 全面加固 |
| 9 | **接口设计** | 1个通用addCoin + 参数区分 | 7个语义明确RPC接口 | ✗ 规避粗粒度, ✓ 语义明确 |
| 10 | **数据模型** | 单行/BigDecimal/无唯一键 | 多行/BIGINT/唯一键 | ✗ 完全重做 |
| 11 | **依赖关系** | 出站4服务 + MQ | 出站仅财务模块 | ✓ 保持低耦合 |
| 12 | **技术栈** | Java Spring Boot | Go CloudWeGo | - 不同技术栈, 取其思想 |

**图例**：✓ = 借鉴/保持, ✗ = 规避/不采用, △ = 借鉴概念但改进实现

---

> **文档总结**：P9钱包模块提供了一个**功能可用但工程质量参差**的参考实现。
> 核心价值在于验证了冻结模型、快照机制等**概念方向的正确性**，
> 同时其并发/幂等/事务/安全方面的缺陷为我们提供了**明确的反面教材**。
> 在我们的ser-wallet设计中，已在前序文档（技术挑战/c-5.md、核心方案/tmp-2.md等）中
> 针对这些问题给出了更严谨的解决方案。
