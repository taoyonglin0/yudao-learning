# 25-BPMN流程 - BPMN 流程设计与画法

> 学习时间：2026-04-29
> 任务编号：25
> 所属模块：流程引擎

---

## ① Why - 价值 (为什么)

### 背景与痛点

在企业级业务流程管理中，存在以下问题：

1. **流程难以可视化**：纯代码编写的流程难以直观理解，维护成本高
2. **业务与技术脱节**：业务人员无法参与流程设计，需求沟通成本高
3. **流程变更不灵活**：修改流程需要改代码、重新部署，响应业务慢
4. **合规审计困难**：无法直观展示完整业务流程轨迹

### 收益

- **可视化设计**：通过图形化界面设计流程，业务人员可参与
- **标准统一**：BPMN 2.0 国际标准，业界通用
- **灵活修改**：流程变更无需编码，热部署生效
- **全程追溯**：流程执行轨迹清晰可见，审计合规

### 使用者

- 流程设计人员
- 业务分析师
- 后端开发工程师

---

## ② What - 定义 (是什么)

### 一句话定义

**BPMN 流程设计**是使用 BPMN 2.0 标准（Business Process Model and Notation）通过可视化方式绘制业务流程图，实现业务流程的建模、设计与执行。

### 核心组成部分

```
┌─────────────────────────────────────────────────────────────┐
│                   BPMN 流程设计器                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ 开始事件 │──▶│ 用户任务 │──▶│ 服务任务 │──▶│ 结束事件 │ │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │
├─────────────────────────────────────────────────────────────┤
│                   Flowable 流程引擎                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ 流程定义│  │ 流程实例│  │ 任务节点│  │ 历史记录│  │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 关键术语

| 术语 | 说明 |
|------|------|
| ProcessDefinition | 流程定义（模板） |
| ProcessInstance | 流程实例（实际运行） |
| Task | 任务节点（需要人工处理） |
| Activity | 活动节点（泛指任务、事件等） |
| SequenceFlow | 顺序流（连线） |
| GateWay | 网关（分支/聚合） |

---

## ③ How - 思维 (怎么做)

### 目标

使用 Flowable 流程引擎实现企业级 BPMN 流程设计与执行。

### 范围

```
允许修改：
- yudao-module-bpm/                    (BPM 模块)
- yudao-framework/                    (框架层)

禁止修改：
- yudao-module-system/                (系统模块)
- yudao-module-pay/                (支付模块)
```

### 数据模型设计

**流程定义信息表 (bpm_process_definition_info)**：
```java
public class BpmProcessDefinitionInfoDO {
    private Long id;                  // ID
    private String processDefinitionId; // 流程定义ID
    private String modelId;            // 流程模型ID
    private Integer modelType;          // 模型类型
    private String category;            // 流程分类
    private String icon;               // 图标
    private String description;       // 描述
    
    private Integer formType;          // 表单类型
    private Long formId;            // 表单ID
    private String formConf;          // 表单配置
    private List<String> formFields;   // 表单字段
    
    private String simpleModel;        // 设计器模型数据
    private Boolean visible;         // 是否可见
    private Long sort;             // 排序
    
    private List<Long> startUserIds;    // 可发起用户
    private List<Long> startDeptIds; // 可发起部门
    private List<Long> managerUserIds; // 可管理用户
    
    private Boolean allowCancelRunningProcess; // 允许撤销
    private Boolean allowWithdrawTask;     // 允许撤回
}
```

### 关键流程设计

```
┌──────────────┐     ┌──────────────────┐     ┌────────────────┐
│  创建流程模型 │ ──▶ │   设计流程      │ ──▶ │  部署流程      │
│ (createModel) │     │ (BPMN 设计器)   │     │ (deploy)       │
└──────────────┘     └──────────────────┘     └────────────────┘
                                                         │
                     ┌───────────────────────────────────┤
                     │              │              │     │
               ┌─────▼─────┐  ┌────▼────┐  ┌─────▼─────┐
               │  发起流程 │  │ 审批任务 │  │  流程结束 │
               │  (start) │  │(task)   │  │  (end)   │
               └───────────┘  └─────────┘  └───────────┘
```

### 关键代码设计

**流程定义服务**：
```java
public interface BpmProcessDefinitionService {
    // 创建流程模型
    String createModel(BpmProcessDefinitionCreateReqDTO reqVO);
    
    // 部署流程
    String deployModel(String modelId);
    
    // 发起流程
    String startProcessInstance(Long userId, String processDefinitionKey, Map<String, Object> variables);
    
    // 办理任务
    void completeTask(String taskId, Map<String, Object> variables);
    
    // 撤回流程
    void withdrawProcessInstance(String processInstanceId);
}
```

---

## ④ Hard - 难点 (挑战)

### 难点 1：流程图的可视化设计

**场景**：业务人员需要通过图形界面设计流程。

**解决方案**：集成 Flowable Design 或自研设计器，支持拖拽式流程绘制。

### 难点 2：流程与表单的集成

**场景**：流程节点需要加载不同的表单。

**解决方案**：
- 动态表单：存储表单配置 JSON
- 自定义表单：支持 Vue 路由映射

### 难点 3：多实例任务（会签/或签）

**场景**：一个任务需要多个人审批（如部门负责人审批）。

**解决方案**：
- Flowable MultiInstanceLoopCardinality
- 支持顺序/并行执行

### 难点 4：流程版本管理

**场景**：流程修改后正在进行中的流程实例如何处理。

**解决方案**：
- 新流程部署不影响进行中的实例
- 支持流程定义升级

---

## ⑤ Metric - 衡量 (指标)

| 指标 | 权重 | 说明 | 验证方法 |
|------|------|------|----------|
| 支持流程节点类型 ≥ 8种 | 20% | 支持的 BPMN 节点类型 | 代码统计 |
| 流程部署成功率 ≥ 99% | 25% | 部署成��率 | 部署日志 |
| 流程启动响应时间 < 500ms | 15% | 发起流程的耗时 | 性能测试 |
| 审批通过率 ≥ 95% | 20% | 流程完成率 | 统计报表 |
| 流程可追溯性 | 20% | 历史轨迹完整 | 流程追踪 |

---

## ⑥ Select - 选型 (选哪个)

### 候选方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Flowable | 功能强大、社区活跃、Spring 集成好 | 学习曲线陡 | 企业首选 |
| Activiti | 早期 BPMN 标准、轻量 | 社区较弱 | 简单场景 |
| Camunda | 现代架构、纯 BPMN | 文档较少 | 新项目 |

### 选型理由

**选择 Flowable**，因为：
1. 完全支持 BPMN 2.0 标准
2. 与 Spring Boot 无缝集成
3. 社区活跃，持续更新
4. 文档丰富

### 工具集

- **流程设计**：Flowable Design / Flowable Modeler
- **API 测试**：Postman / Apifox
- **文档**：
  - Flowable 官方文档：https://flowable.com/
  - yudao-cloud BPM 模块：https://github.com/YunaiCloud/yudao-cloud

---

## ⑦ Impl - 实现 (细节)

### 核心实体类

```java
// 流程定义信息
public class BpmProcessDefinitionInfoDO {
    private Long id;
    private String processDefinitionId;
    private String modelId;
    private Integer modelType;
    private String category;
    private Integer formType;
    private Long formId;
    private String formConf;
    private Boolean visible;
    private Long sort;
}
```

### 核心实现类

**BpmProcessDefinitionServiceImpl 关键代码**：
```java
@Override
public String deployModel(String modelId) {
    // 1. 获取流程模型
    Model model = repositoryService.getModel(modelId);
    
    // 2. 转换为 BPMN XML
    byte[] bpmnBytes = deploymentHelper.convertToBpmn(model);
    
    // 3. 部署流程
    Deployment deployment = repositoryService.createDeployment()
        .name(model.getName())
        .addBytes(model.getDeploymentKey() + ".bpmn20.xml", bpmnBytes)
        .deploy();
    
    // 4. 返回流程定义ID
    ProcessDefinition processDefinition = repositoryService
        .createProcessDefinitionQuery()
        .deploymentId(deployment.getId())
        .singleResult();
    return processDefinition.getId();
}
```

### 关键步骤校验

| 步骤 | 校验点 | 验证方法 |
|------|--------|----------|
| 1. 创建模型 | 模型ID非空 | 创建返回 |
| 2. 设计流程 | XML 生成成功 | BPMN 导出 |
| 3. 部署流程 | deploymentId 非空 | 部署日志 |
| 4. 发起流程 | instanceId 非空 | 流程启动 |
| 5. 审批任务 | 任务完成 | 任务列表 |

### 失败恢复机制

- **场景 1：流程部署失败**
  - 触发：XML 解析错误
  - 恢复：返回错误信息，检查 XML

- **场景 2：审批超时**
  - 触发：任务未在时限内完成
  - 恢复：发送催办通知

- **场景 3：流程异常终止**
  - 触发：服务错误
  - 恢复：记录异常日志，管理员处理

---

## ⑧ SKILL - 提炼 (复用)

### 触发条件

```
场景1：需要可视化设计业务流程
场景2：需要动态调整审批流程
场景3：需要会签/或签多实例审批
场景4：需要流程历史追溯
```

### 执行流程

```
Step 1: 添加依赖
  - 在 pom.xml 中添加 flowable-spring-boot-starter
  - 配置流程引擎

Step 2: 绘制流程图
  - 打开 Flowable Design
  - 拖拽节点，配置属性

Step 3: 部署流程
  - 导出 BPMN XML
  - 调用部署 API

Step 4: 发起流程
  - 调用 startProcessInstance
  - 传入流程变量
```

### 配方/素材

**技术栈**：Java 17+, Spring Boot 3.x, Flowable 6.x

**核心依赖**：
```xml
<!-- Flowable -->
<dependency>
    <groupId>org.flowable</groupId>
    <artifactId>flowable-spring-boot-starter</artifactId>
</dependency>

<!-- Flowable UI -->
<dependency>
    <groupId>org.flowable</groupId>
    <artifactId>flowable-ui-modeler-starter</artifactId>
</dependency>
```

**配置文件**：
```yaml
spring:
  flowable:
    database-schema-mode: update
    history-level: full
```

### 验收标准

- [ ] 支持至少 8 种 BPMN 节点类型
- [ ] 可以通过设计器设计流程
- [ ] 流程可部署和发起
- [ ] 审批任务可办理
- [ ] 流程历史可追溯

---

## 附录：BPMN 节点类型参考

| 类别 | 节点 | 说明 |
|------|------|------|
| 事件 | 开始 | 流程开始 |
| 事件 | 结束 | 流程结束 |
| 事件 | 中间消息 | 消息捕获 |
| 活动 | 用户任务 | 人工审批 |
| 活动 | 服务任务 | 自动执行 |
| 活动 | 脚本任务 | 脚本执行 |
| 网关 | 排他网关 | 单一分支 |
| 网关 | 并行网关 | 多路并行 |
| 网关 | 包含网关 | 组合条件 |