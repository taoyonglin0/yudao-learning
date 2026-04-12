# 10-Nacos配置.md

> 配置管理 - Nacos 核心概念

---

## ① Why - 价值 (为什么)

**背景与痛点**

- **背景**：分布式系统下，配置文件散落在各服务节点，修改配置需要重启服务，多环境切换麻烦
- **不用它的痛点**：
  - 修改配置需登录每台服务器，效率低易出错
  - 环境切换需改多份配置文件
  - 配置变更无法追溯审计
  - 无法动态生效
- **收益**：
  - 统一配置中心，配置管理可视化
  - 配置动态刷新，无需重启
  - 多环境/多租户配置隔离
  - 配置变更历史可追溯

**目标用户**

- 运维人员：配置管理和发布
- 开发人员：配置查看和调试
- 管理员：配置审批和审计

---

## ② What - 定义 (是什么)

**一句话定义**

Nacos 是阿里巴巴开源的 **动态配置服务中心**，提供配置管理、服务发现、健康监测等功能。

**核心概念**

| 概念 | 说明 |
|------|------|
| Namespace | 命名空间，用于环境/租户隔离 |
| Group | 配置分组，按业务维度分组 |
| Data ID | 配置文件ID（spring.application.name） |
| Configuration | 配置项，key-value形式 |
| Service | 注册服务 |
| Instance | 服务实例 |

**核心流程**

```
启动 → 连接Nacos → 拉取配置 → 本地缓存 → 监听变更 → 动态更新
```

---

## ③ How - 思维 (怎么做)

### 数据模型设计

```
Nacos配置表（config_info）：
- id           : 主键
- namespace_id  : 命名空间ID
- group_id     : 分组ID
- data_id     : 配置ID
- content     : 配置内容
- md5         : 内容MD5
- gmt_create  : 创建时间
- gmt_modified : 修改时间
- src_user    : 修改人
- src_app     : 来源应用
- app_name    : 应用名
- tenant_id   : 租户ID
```

### 架构设计

```
┌─────────────┐     ┌─────────────┐
│  Nacos     │────▶│  Config   │
│  Server    │     │  Client   │
└─────────────┘     └─────────────┘
      │                  │
      ▼                  ▼
┌─────────────┐     ┌─────────────┐
│  MySQL    │     │  Service │
│  存储     │     │  实例    │
└─────────────┘     └─────────────┘
```

### Yudao 实现方式

**注意**：芋道项目 **未使用 Nacos**，而是用内置数据库存储配置：
- `infra_config` 表存储业务配置
- 配置管理后台界面
- 通过配置服务接口读写

```java
// 芋道配置实体
@TableName("infra_config")
public class ConfigDO extends BaseDO {
    private Long id;         // 主键
    private String category;  // 分类
    private String name;   // 名称
    private String configKey; // 键名
    private String value;   // 键值
    private Integer type;  // 类型
    private Boolean visible; // 可见性
    private String remark; // 备注
}
```

---

## ④ Hard - 难点 (挑战)

### 问题1：配置一致性

```
场景：多实例部署，配置不同步
→ 解决：Nacos客户端拉取+监听机制
→ 解决：使用本地缓存兜底
```

### 问题2：配置覆盖优先级

```
场景：本地配置 > Nacos配置，不生效
→ 解决：了解Spring Boot配置优先级
→ 解决：命令行覆盖 -Dspring.config.additional-location
```

### 问题3：配置变更推送延迟

```
场景：配置变更后，部分实例未及时更新
→ 解决：监听 LongPulling 机制
→ 解决：重启实例或��动刷新
```

### 问题4：敏感配置泄露

```
场景：密码等敏感配置明文存储
→ 解决：使用 Nacos 加密插件
→ 解决：敏感配置存本地环境变量
```

---

## ⑤ Metric - 衡量 (指标)

| 指标 | 权重 | 说明 | 验证方法 |
|------|------|------|----------|
| 配置推送成功率 | 30% | 推送成功/总推送 | Nacos控制台 |
| 推送延迟 | 25% | <3秒 | 日志时间戳 |
| 配置读取成功率 | 20% | 读取成功/总读取 | 请求日志 |
| 多环境隔离 | 15% | Namespace隔离 | 控制台查看 |
| 敏感配置安全 | 10% | 加密存储 | 安全扫描 |

---

## ⑥ Select - 选型 (选哪个)

### 候选方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Nacos | 国产、文档全、社区活跃 | 功能多、学习成本 | 国内项目 |
| Apollo | 配置评审、发布管理 | 需额外部署 | 中大型项目 |
| Spring Cloud Config | Spring原生 | 不支持动态刷新 | 微服务项目 |
| etcd | 分布式、一致性 | 生态弱 | 云原生项目 |

### 选型结论

**选 Nacos**，因为：
1. 国产开源，中文文档
2. 同时支持配置中心+服务发现
3. 运维管理后台完善
4. 集成Spring Boot简单

---

## ⑦ Impl - 实现 (细节)

### Nacos 客户端集成

**依赖**

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-config</artifactId>
    <version>0.2.1</version>
</dependency>
```

**配置**

```yaml
spring:
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: dev
      group: DEFAULT_GROUP
      config:
        file-extension: yaml
        refresh-enabled: true
```

**监听配置变化**

```java
@RefreshScope
@RestController
public class ConfigController {
    
    @Value("${custom.config:default}")
    private String config;
    
    @GetMapping("/config")
    public String getConfig() {
        return config;
    }
}
```

### 芋道配置管理实现

**配置服务**

```java
@Service
public class ConfigServiceImpl implements ConfigService {

    @Resource
    private ConfigMapper configMapper;

    @Override
    public Long createConfig(ConfigSaveReqVO createReqVO) {
        validateConfigKeyUnique(null, createReqVO.getKey());
        ConfigDO config = ConfigConvert.INSTANCE.convert(createReqVO);
        config.setType(ConfigTypeEnum.CUSTOM.getType());
        configMapper.insert(config);
        return config.getId();
    }

    @Override
    public ConfigDO getConfigByKey(String key) {
        return configMapper.selectByKey(key);
    }
}
```

**配置 Mapper**

```java
public interface ConfigMapper extends BaseMapperX<ConfigDO> {
    
    default ConfigDO selectByKey(String key) {
        return selectOne(new LambdaQueryWrapperX<ConfigDO>()
            .eq(ConfigDO::getConfigKey, key));
    }
}
```

**配置 Controller**

```java
@RestController
@RequestMapping("/config")
public class ConfigController {
    
    @Resource
    private ConfigService configService;

    @GetMapping("/list")
    public PageResult<ConfigDO> getConfigPage(ConfigPageReqVO reqVO) {
        return configService.getConfigPage(reqVO);
    }
}
```

### API 接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/nacos/v1/cs/configs` | GET | 获取配置 |
| `/api/nacos/v1/cs/configs` | POST | 发布配置 |
| `/api/nacos/v1/cs/configs` | DELETE | 删除配置 |

---

## ⑧ SKILL - 提炼 (复用)

### 触发场景

```
场景1：需要集中管理配置
场景2：需要多环境切换
场景3：需要配置动态生效
```

### 执行流程

```
Step 1: 引入依赖
  - 添加 nacos-config 依赖

Step 2: 配置 Nacos
  - 配置 server-address
  - 配置 namespace/group

Step 3: 使用配置
  - @Value 读取
  - @RefreshScope 刷新

Step 4: 管理配置
  - 控制台管理
  - API 发布
```

### 验收标准

```
- [x] Nacos 服务正常启动
- [x] 配置能正常发布/读取
- [x] 配置变更能动态生效
- [x] 多环境隔离正常
- [x] 配置安全存储
```

---

## 附录：Yudao 配置 vs Nacos

| 特性 | Yudao | Nacos |
|------|------|------|
| 存储 | MySQL数据库 | Nacos服务端 |
| 管理 | 后台界面/API | Nacos控制台 |
| 动态刷新 | 需重启/手动刷新 | 支持 |
| 多环境 | 多租户字段 | Namespace |
| 审计 | 数据库记录 | 版本管理 |

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `yudao-module-infra/.../config/ConfigDO.java` | 配置实体 |
| `yudao-module-infra/.../config/ConfigServiceImpl.java` | 配置服务 |
| `yudao-module-infra/.../config/ConfigController.java` | 配置接口 |