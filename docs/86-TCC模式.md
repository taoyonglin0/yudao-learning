# 86-TCC模式 - 分布式事务 TCC 模式

> 学习日期：2026-04-13
> 任务编号：86
> 所属模块：分布式事务

---

## ① Why - 价值 (为什么)

### 背景与痛点

在微服务架构中，AT 模式虽然好用，但有以下局限性：

| 局限性 | 说明 |
|--------|------|
| **依赖数据库** | AT 模式需要底层数据库支持本地 ACID 事务 |
| **性能瓶颈** | 一阶段需要等待全局锁，高并发下有性能损失 |
| **灵活性差** | 无法精确控制资源锁定粒度 |

### TCC 模式带来的价值

**TCC（Try-Confirm-Cancel）** 是一种高性能的分布式事务解决方案：

| 价值 | 说明 |
|------|------|
| **高性能** | 一阶段直接执行，无需等待全局锁 |
| **灵活性高** | 业务方控制资源锁定粒度 |
| **跨数据库** | 不依赖底层数据库，可跨多个数据库 |
| **适用性强** | 适合核心系统、对性能要求高的场景 |

### 适用场景

- 核心交易系统：对性能要求极高的支付、订单场景
- 跨库操作：涉及多个不同数据库的业务
- 非数据库资源：需要锁定 Redis、MQ 等非数据库资源
- 金融场景：对一致性要求高但需要高性能

---

## ② What - 定义 (是什么)

### 一句话定义

**TCC 模式** 是 Seata 分布式事务的一种模式，由业务方自行实现 Try（预留）、Confirm（确认）、Cancel（取消）三个阶段，两阶段协调完成分布式事务。

### 核心概念

| 概念 | 说明 |
|------|------|
| **Try 阶段** | 预留资源，检查业务状态 |
| **Confirm 阶段** | 执行真正的业务操作 |
| **Cancel 阶段** | 取消预留，释放资源 |
| **TM** | 事务管理器，发起全局事务 |
| **RM** | 资源管理器，管理 TCC 资源 |
| **TCC Resource** | TCC 接口在 Seata 中的资源抽象 |

### TCC 与 AT 对比

| 特性 | AT 模式 | TCC 模式 |
|------|---------|----------|
| 侵入性 | 无侵入（注解即可） | 高（需实现三阶段） |
| 性能 | 中等 | 高 |
| 依赖 | 依赖数据库 | 不依赖数据库 |
| 回滚方式 | 自动（undo_log） | 手动（业务实现） |
| 适用场景 | 大多数场景 | 高性能核心系统 |

---

## ③ How - 思维 (怎么做)

### 数据模型设计

TCC 模式**不需要 undo_log 表**，但需要业务方设计预留资源表：

#### 预留资源表（业务方设计）

```sql
-- 账户冻结余额表
CREATE TABLE `account_freeze` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) NOT NULL COMMENT '用户ID',
  `freeze_amount` decimal(10,2) NOT NULL DEFAULT '0.00' COMMENT '冻结金额',
  `freeze_status` tinyint(2) NOT NULL DEFAULT '0' COMMENT '状态:0-冻结,1-已扣减,2-已释放',
  `xid` varchar(100) NOT NULL COMMENT '全局事务ID',
  `branch_id` bigint(20) NOT NULL COMMENT '分支事务ID',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 库存冻结表示例

```sql
-- 库存冻结表
CREATE TABLE `stock_freeze` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `sku_id` bigint(20) NOT NULL COMMENT '商品ID',
  `warehouse_id` bigint(20) NOT NULL COMMENT '仓库ID',
  `freeze_quantity` int(11) NOT NULL DEFAULT '0' COMMENT '冻结数量',
  `freeze_status` tinyint(2) NOT NULL DEFAULT '0' COMMENT '状态:0-冻结,1-已扣减,2-已释放',
  `xid` varchar(100) NOT NULL,
  `branch_id` bigint(20) NOT NULL,
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_sku_warehouse` (`sku_id`, `warehouse_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 关键流程设计

#### 整体流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TCC 事务生命周期                              │
└─────────────────────────────────────────────────────────────────────┘

   TM (业务发起)                    TC (协调者)                     TCC Resource
       │                               │                               │
       │───── 开启全局事务 (XID) ─────▶│                               │
       │                               │                               │
       │                               │──── Try阶段 ─────────────────▶│
       │                               │    (预留资源/检查状态)          │
       │                               │                               │
       │                               │◀──── Try成功 ──────────────────│
       │                               │                               │
       │──── 执行业务 ────────────────▶│                               │
       │                               │                               │
       │                               │──── Confirm阶段 ─────────────▶│
       │                               │    (执行真正业务)               │
       │                               │                               │
       │                               │◀──── Confirm成功 ─────────────│
       │                               │                               │
       │──── 业务完成                  │                               │
       │                               │                               │
       │                               │  (如失败则执行Cancel)          │
       │                               │──── Cancel阶段 ──────────────▶│
       │                               │    (释放预留资源)              │
       │                               │                               │
       ◀──── 最终结果 ────────────────│                               │
                                    │                               │
```

#### Try 阶段详解

```
Step 1: 检查业务状态
  - 检查账户余额是否充足
  - 检查库存是否足够
  - 检查业务规则是否满足

Step 2: 预留资源
  - 冻结账户余额（账户余额不变，预留额度）
  - 冻结库存（库存不变，预留数量）
  - 记录预留状态

Step 3: 记录分支事务
  - 向 TC 注册分支
  - 保存 xid 和 branch_id

Step 4: 返回 Try 结果
  - 成功：资源已预留
  - 失败：业务不满足，回滚
```

#### Confirm 阶段详解

```
Step 1: 执行业务
  - 扣减实际余额（之前只是冻结）
  - 扣减实际库存（之前只是冻结）
  - 更新业务状态

Step 2: 清理预留记录
  - 删除或标记预留记录

Step 3: 上报结果
  - 报告 Confirm 成功给 TC
```

#### Cancel 阶段详解

```
Step 1: 查询预留记录
  - 根据 xid 和 branch_id 查询

Step 2: 释放资源
  - 解冻账户余额
  - 解冻库存

Step 3: 更新预留状态
  - 标记为已释放

Step 4: 上报结果
  - 报告 Cancel 成功给 TC
```

### 关键代码设计

#### TCC 接口定义

```java
// 账户服务 TCC 接口
@LocalTCC  // 本地 TCC 需添加此注解
public interface AccountTccService {
    
    /**
     * Try 阶段：预留资源
     * @param userId 用户ID
     * @param amount 扣减金额
     */
    @TwoPhaseBusinessAction(
        name = "accountTccService",
        commitMethod = "confirm",
        rollbackMethod = "cancel"
    )
    boolean tryDeduct(
        BusinessActionContext actionContext,
        @BusinessActionContextParameter(paramName = "userId") Long userId,
        @BusinessActionContextParameter(paramName = "amount") BigDecimal amount
    );
    
    /**
     * Confirm 阶段：确认扣减
     */
    boolean confirm(BusinessActionContext actionContext);
    
    /**
     * Cancel 阶段：取消预留
     */
    boolean cancel(BusinessActionContext actionContext);
}
```

#### TCC 接口实现

```java
@Service
public class AccountTccServiceImpl implements AccountTccService {
    
    @Autowired
    private AccountMapper accountMapper;
    
    @Autowired
    private AccountFreezeMapper freezeMapper;
    
    @Override
    public boolean tryDeduct(BusinessActionContext actionContext, 
                             Long userId, BigDecimal amount) {
        // 1. 检查账户余额
        AccountDO account = accountMapper.selectByUserId(userId);
        if (account.getBalance().compareTo(amount) < 0) {
            throw new BizException("余额不足");
        }
        
        // 2. 预留资源：冻结余额
        AccountFreezeDO freeze = new AccountFreezeDO();
        freeze.setUserId(userId);
        freeze.setFreezeAmount(amount);
        freeze.setFreezeStatus(0); // 冻结中
        freeze.setXid(actionContext.getXid());
        freeze.setBranchId(actionContext.getBranchId());
        freezeMapper.insert(freeze);
        
        // 3. 不实际扣减，只是预留
        return true;
    }
    
    @Override
    public boolean confirm(BusinessActionContext actionContext) {
        // 获取预留时传递的参数
        Long userId = actionContext.getActionContext("userId", Long.class);
        BigDecimal amount = actionContext.getActionContext("amount", BigDecimal.class);
        
        // 实际扣减余额
        accountMapper.deductBalance(userId, amount);
        
        // 更新冻结状态为已扣减
        freezeMapper.updateStatusByXid(actionContext.getXid(), 1);
        
        return true;
    }
    
    @Override
    public boolean cancel(BusinessActionContext actionContext) {
        Long userId = actionContext.getActionContext("userId", Long.class);
        BigDecimal amount = actionContext.getActionContext("amount", BigDecimal.class);
        
        // 释放冻结：删除冻结记录（资金回到原账户）
        freezeMapper.deleteByXid(actionContext.getXid());
        
        return true;
    }
}
```

#### 业务调用

```java
@GlobalTransactional(timeoutMills = 30000, name = "create-order-tcc")
public void createOrder(OrderDTO order) {
    // 1. 调用账户服务的 Try 方法
    accountTccService.tryDeduct(order.getUserId(), order.getTotalAmount());
    
    // 2. 调用库存服务的 Try 方法
    stockTccService.tryDeduct(order.getItems());
    
    // 3. 创建订单
    orderService.createOrder(order);
    
    // 如果后续调用失败，Seata 会自动调用各服务的 Cancel 方法
}
```

---

## ④ Hard - 难点 (挑战)

### 难点一：幂等性问题

**问题**：Confirm/Cancel 可能被重复调用

**场景**：
1. TM 发起全局回滚
2. TC 发送 Cancel 请求到 RM1
3. RM1 执行成功，上报结果
4. 网络超时，TC 认为失败
5. TC 重试 Cancel，RM1 再次执行

**解决方案**：

```java
@Override
public boolean confirm(BusinessActionContext actionContext) {
    // 通过 branch_id 或 xid 判断是否已执行
    int affected = freezeMapper.updateStatusByXid(
        actionContext.getXid(), 
        1  // 更新为已扣减状态
    );
    // affected = 0 表示已经处理过
    return affected > 0;
}
```

### 难点二：空回滚问题

**问题**：Try 阶段未执行，Cancel 被调用

**场景**：
1. TM 发起全局事务
2. RM1 的 Try 方法未执行（或超时）
3. TM 回滚，调用 RM1 的 Cancel
4. Cancel 发现没有预留记录

**解决方案**：

```java
@Override
public boolean cancel(BusinessActionContext actionContext) {
    // 查询是否有预留记录
    AccountFreezeDO freeze = freezeMapper.selectByXid(actionContext.getXid());
    
    // 空回滚：直接返回成功
    if (freeze == null) {
        // 记录空回滚日志
        log.info("空回滚, xid={}", actionContext.getXid());
        return true;
    }
    
    // 正常回滚：释放资源
    freezeMapper.deleteByXid(actionContext.getXid());
    return true;
}
```

### 难点三：悬挂问题

**问题**：Cancel 比 Try 先执行

**场景**：
1. RM1 Try 方法超时，未完成
2. TM 认为全局事务失败，发起回滚
3. Cancel 先执行，释放资源
4. Try 后续执行，预留资源
5. 资源被重复使用

**解决方案**：

```java
@Override
public boolean tryDeduct(Long userId, BigDecimal amount) {
    // 检查是否存在 Cancel 记录（悬挂）
    AccountFreezeDO freeze = freezeMapper.selectByUserId(userId);
    if (freeze != null && freeze.getFreezeStatus() == 2) {
        // 之前已取消，本次 Try 无效
        throw new BizException("事务悬挂，拒绝执行");
    }
    
    // 正常预留资源
    // ...
}
```

### 难点四：事务状态管理

**问题**：如何追踪 TCC 事务状态

**解决方案**：

```java
// 在 Try 阶段记录事务状态
public enum TccStatus {
    TRYING,      // Try 执行中
    CONFIRMING,  // Confirm 执行中
    CANCELING,   // Cancel 执行中
    CONFIRMED,   // 已确认
    CANCELED,    // 已取消
    SUSPENDED    // 悬挂
}
```

---

## ⑤ Metric - 衡量 (指标)

### 指标设计

| 指标 | 权重 | 说明 | 验证方法 |
|------|------|------|----------|
| **事务成功率** | 30% | 全局事务正常结束/总事务数 | TC 日志统计 |
| **性能提升** | 25% | TCC vs AT 的 RT 减少 | 压测对比 |
| **幂等性保证** | 20% | 重复执行不产生副作用 | 模拟重试测试 |
| **空回滚处理** | 15% | 空回滚正确处理率 | 异常场景测试 |
| **资源释放** | 10% | 预留资源正确释放 | 异常场景测试 |

### 性能对比参考

| 场景 | AT 模式 | TCC 模式 | 提升 |
|------|---------|----------|------|
| 单服务 | 15-20ms | 8-12ms | 40% |
| 跨服务(2个) | 30-40ms | 15-20ms | 50% |
| 高并发(1000 TPS) | 性能下降明显 | 稳定 | - |

---

## ⑥ Select - 选型 (选哪个)

### 选型决策树

```
是否需要分布式事务？
  │
  ├─ 否 → 本地事务即可
  │
  └─ 是 → 业务场景是？
           │
           ├─ 对性能要求极高 → TCC 模式
           │
           ├─ 业务流程长/多服务编排 → Saga 模式
           │
           └─ 通用场景 → AT 模式
```

### TCC vs AT vs Saga

| 特性 | AT 模式 | TCC 模式 | Saga 模式 |
|------|---------|----------|------------|
| 侵入性 | 低 | 高 | 中 |
| 性能 | 中 | 高 | 高 |
| 复杂度 | 低 | 中 | 高 |
| 一致性 | 强 | 强 | 最终一致 |
| 适用场景 | 通用 | 高性能 | 长流程 |

### 选型建议

**选择 TCC 当：**
- 核心交易系统，对性能要求高
- 需要跨多个数据库
- 需要锁定非数据库资源
- 能接受一定的开发工作量

**选择 AT 当：**
- 普通业务场景
- 对性能要求一般
- 希望快速集成

**选择 Saga 当：**
- 业务流程很长（10+ 步骤）
- 包含外部服务调用
- 能接受最终一致性

---

## ⑦ Impl - 实现 (细节)

### yudao-cloud 项目中的实现

**注意**：当前 yudao-cloud 项目**未集成 Seata TCC 模式**，使用本地事务。

#### 现有事务实现

```java
// 现有代码使用 Spring @Transactional
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderCreateDTO dto) {
    // 单库事务
    orderMapper.insert(dto);
    // 如果跨服务调用，没有分布式事务保证
}
```

#### 项目中类似 TCC 的模式

项目中实际使用的是**补偿机制**（类似 Saga 思想）：

```java
// 示例：支付回调中的补偿逻辑
@Transactional
public void handlePayNotify(PayOrderNotifyDTO notify) {
    // 1. 更新订单状态
    orderMapper.updateStatus(notify.getOrderId(), OrderStatus.PAID);
    
    // 2. 如果后续操作失败，需要手动补偿
    try {
        stockService.deduct(notify.getOrderId());
    } catch (Exception e) {
        // 补偿：恢复订单状态
        orderMapper.updateStatus(notify.getOrderId(), OrderStatus.UNPAID);
        throw e;
    }
}
```

### TCC 模式集成步骤

#### Step 1: 添加依赖

```xml
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>1.7.1</version>
</dependency>
```

#### Step 2: 配置 Seata

```yaml
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: ${spring.application.name}-tx-group
  registry:
    type: nacos
    nacos:
      server-addr: ${nacos.addr:127.0.0.1:8848}
```

#### Step 3: 实现 TCC 接口

```java
// 参考上面的代码设计
```

#### Step 4: 业务调用

```java
@GlobalTransactional
public void businessMethod() {
    // 调用 TCC 接口
}
```

### 验证步骤

```
验证点1: 正常流程
  → 验证方法: 执行业务，观察 Try -> Confirm 流程
  → 预期: 资源预留成功，业务完成

验证点2: 异常回滚
  → 验证方法: 在 Confirm 前抛出异常
  → 预期: 自动调用 Cancel，释放预留资源

验证点3: 幂等性
  → 验证方法: 重复调用 Confirm
  → 预期: 第二次直接返回，不重复执行

验证点4: 空回滚
  → 验证方法: Try 未执行，直接回滚
  → 预期: Cancel 返回成功，无异常

验证点5: 悬挂
  → 验证方法: Cancel 先执行，再执行 Try
  → 预期: Try 拒绝执行，抛出异常
```

---

## ⑧ SKILL - 提炼 (复用)

### 触发条件

```
场景1: 核心交易系统需要高性能分布式事务
场景2: 需要跨多个数据库的业务
场景3: 需要锁定非数据库资源（Redis、MQ）
场景4: AT 模式性能无法满足需求
```

### 执行流程

```
Step 1: 评估业务场景
  - 是否对性能要求极高
  - 是否需要跨数据库
  - 是否需要锁定非数据库资源

Step 2: 设计 TCC 接口
  - Try: 预留资源 + 检查状态
  - Confirm: 执行真正业务
  - Cancel: 释放预留资源

Step 3: 处理异常场景
  - 幂等性：使用状态字段
  - 空回滚：检查预留记录
  - 悬挂：检查 Cancel 状态

Step 4: 集成测试
  - 正常流程测试
  - 异常回滚测试
  - 并发测试
```

### 配方/素材

| 组件 | 版本 | 说明 |
|------|------|------|
| Seata | 1.7.1 | 事务协调器 |
| seata-spring-boot-starter | 1.7.1 | 客户端 |
| @TwoPhaseBusinessAction | - | TCC 注解 |
| @LocalTCC | - | 本地 TCC 注解 |

### 脚本

#### TCC 接口健康检查

```bash
# 检查 TCC 服务是否正常
curl -X POST http://localhost:8080/actuator/health

# 检查 Seata TC 服务
curl http://localhost:8091/health
```

#### 预留资源查询

```sql
-- 查询账户冻结余额
SELECT * FROM account_freeze 
WHERE xid = 'xxx' 
ORDER BY create_time DESC;

-- 查询库存冻结
SELECT * FROM stock_freeze 
WHERE freeze_status = 0 
ORDER BY create_time DESC;
```

### 验收标准

```
- [ ] TCC 接口实现完整（Try/Confirm/Cancel）
- [ ] 幂等性处理正确（状态字段判断）
- [ ] 空回滚处理正确（无记录时返回成功）
- [ ] 悬挂处理正确（检查Cancel状态）
- [ ] 正常流程测试通过
- [ ] 异常回滚测试通过
- [ ] 性能满足要求 (<20ms)
```

---

## 附录：相关任务

- 85-AT模式.md：AT 模式详解
- 87-空回滚与悬挂.md：TCC 异常场景详解

---

## 更新记录

| 日期 | 内容 | 操作人 |
|------|------|--------|
| 2026-04-13 | 初始版本 | AlexTao |