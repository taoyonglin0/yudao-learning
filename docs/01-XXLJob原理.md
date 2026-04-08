# 01-XXLJob原理

> 定时任务 - XXL-Job 核心原理与表结构

---

## ① Why - 价值

**背景与痛点**：
- 原来每天手动同步数据到凌晨 3 点，运营苦不堪言
- 硬编码 @Scheduled 不可控制，改时间需要重启服务
- 多节点部署时任务重复执行，无法统一管理

**收益**：
- 系统自动跑，第二天上班数据已同步好
- 可视化管理，运营可直接配置任务
- 分布式锁解决多节点重复执行问题

**用户**：
- 运维人员、调度系统、后端开发者

---

## ② What - 定义

**一句话定义**：
XXL-Job 是一个分布式定时任务调度平台，有 3 个核心：调度中心、执行器、任务。

**核心组成**：
1. **调度中心** (Admin) - Web 管理后台，任务配置/执行监控
2. **执行器** (Executor) - 部署在业务服务中，接收调度指令执行任务
3. **任务** (JobHandler) - 具体业务逻辑方法

**关键术语**：
- Cron 表达式 - 任务触发时间配置
- JobHandler - 任务执行的方法
- XxlJobExecutor - 执行器核心类

---

## ③ How - 思维

**目标**：实现分布式定时任务调度

**范围**：
- 允许：yudao-framework/yudao-spring-boot-starter-job/
- 禁止：改动已有业务模块的 Service 层

**禁止**：
- 禁止修改数据库表结构 (已有 xxlog 表)
- 禁止引入新依赖
- 禁止改动已有 API 接口

**数据模型**：

```java
// XxlJobProperties 配置类
xxl.job:
  enabled: true
  accessToken: xxx
  admin:
    addresses: "http://xxl-job:8080/xxl-job-admin"
  executor:
    appName: yudao-infra
    ip: 
    port: -1  # 随机
    logPath: /tmp/xxl-job
    logRetentionDays: 30
```

**核心流程**：
```
调度中心(cron触发) 
  → 检查任务状态 
  → 选一台执行器 
  → 下发执行指令 
  → 执行器接收 
  → 执行业务逻辑 
  → 记录执行日志 
  → 返回结果
```

**关键类**：
- `YudaoXxlJobAutoConfiguration` - 自动配置类
- `XxlJobProperties` - 配置属性类
- `@XxlJob("accessLogCleanJob")` - 任务注解

---

## ④ Hard - 难点

**难点1：多台机器同时跑，任务重复执行？**
```
场景：A服务部署了3台，9点同时触发，执行3次
→ 分布式锁抢锁，只有一台能抢到并执行
```

**难点2：任务执行一半挂了？**
```
场景：同步10000条数据，同步到5000条服务器重启
→ 补偿机制，记录进度，自动重试从5001条开始
```

**难点3：想改任务参数不想重启？**
```
场景：运营想把同步时间从9点改成10点
→ 热更新，从数据库读取配置，修改立即生效
```

---

## ⑤ Metric - 衡量

| 指标 | 权重 | 说明 | 验证方法 |
|------|------|------|----------|
| 任务执行成功率≥99.9% | 30% | 成功次数/总次数 | 日志统计 |
| 任务重复执行次数=0 | 25% | 同一时间只执行一次 | 日志+分布式锁 |
| 任务修改生效时间≤1min | 20% | 修改配置到生效 | 手动测试 |
| 任务执行日志可查 | 15% | 日志保留30天 | 后台查询 |
| 支持动态修改不重启 | 10% | 改配置不用重启 | 手动测试 |

---

## ⑥ Select - 选型

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| XXL-Job | 文档全、管理后台、支持分布式 | 需额外部署 | 中大型项目 |
| ElasticJob | 支持分片、代码简洁 | 社区较小 | 需要分片场景 |
| Spring Task | 无需额外部署 | 不支持管理后台 | 小型项目 |
| Quartz | 通用、久经考验 | 配置复杂 | 传统项目 |

**选型理由**：
1. 文档最全，社区活跃
2. 有管理后��，运营可直接配置
3. 支持分布式锁，解决重复执行
4. 与 Spring Boot 集成简单

**官方文档**：https://www.xuxueli.com/xxl-job/

---

## ⑦ Impl - 实现

**数据模型**：

```java
// XxlJobProperties
@ConfigurationProperties("xxl.job")
public class XxlJobProperties {
    private Boolean enabled = true;
    private String accessToken;
    private AdminProperties admin;
    private ExecutorProperties executor;
}
```

**执行器代码**：

```java
@XxlJob("accessLogCleanJob")
@TenantIgnore
public void execute() {
    // 1. 获取服务
    // 2. 执行业务逻辑
    // 3. 记录日志
    log.info("[execute][定时执行清理访问日志数量 ({}) 个]", count);
}
```

**配置示例**：

```yaml
xxl:
  job:
    enabled: true
    admin:
      addresses: "http://127.0.0.1:8080/xxl-job-admin"
    executor:
      appName: yudao-infra
      logPath: /tmp/xxl-job
```

**关键步骤校验**：
- Step 1: 检查任务状态 (status == 1)
- Step 2: 抢分布式锁 (SETNX)
- Step 3: 执行任务 (返回成功)
- Step 4: 记录日志

**失败恢复**：
- 任务失败 → 重试机制(最多3次) → 记录错误日志
- 服务重启 → 补偿机制 → 读取断点日志

---

## ⑧ SKILL - 提炼

**触发条件**：
- 场景1：需要实现定时任务功能
- 场景2：需要管理后台配置任务
- 场景3：需要任务执行日志可查

**执行流程**：
```
Step 1: 引入依赖
  - 添加 xxl-job-core
  - 配置 executor

Step 2: 配置执行器
  - 配置 xxl.job.admin.addresses
  - 配置 xxl.job.executor.appName

Step 3: 编写任务
  - @XxlJob("jobHandlerName")
  - 注入 Service
  - 执行业务逻辑

Step 4: 注册到调度中心
  - 登录管理后台
  - 新增任务 -> 录入 jobHandler 名称
```

**配方**：
- 技术栈：Java 8+, Spring Boot 2.7+
- 依赖包：xxl-job-core
- 配置：xxl.job.* (application.yml)

**脚本**：
```bash
# 本地测试执行
curl -X POST http://localhost:48080/xxl-job/jobhandler -d "jobId=1"

# 查看日志
SELECT * FROM xxlog WHERE log_id ORDER BY id DESC LIMIT 10
```

**验收标准**：
- [ ] 功能正常：定时任务能正常执行
- [ ] 单元测试通过：核心方法测试通过
- [ ] 性能达标：单次执行<1分钟
- [ ] 日志可查：后台能看到执行日志