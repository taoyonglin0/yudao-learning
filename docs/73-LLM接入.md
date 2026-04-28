# 73-LLM接入 - 多供应商接入

> 学习时间：2026-04-28
> 任务编号：73
> 所属模块：大模型接入

---

## ① Why - 价值 (为什么)

### 背景与痛点

在企业级 AI 应用开发中，单一 AI 供应商存在以下问题：

1. **供应商锁定风险**：依赖单一厂商，议价能力弱，服务稳定性无保障
2. **成本不可控**：不同厂商定价差异大，无法灵活切换到性价比更高的方案
3. **功能局限性**：各厂商模型能力不同，某些场景需要特定模型（如代码生成、创意写作）
4. **地域合规**：部分企业需要选择特定地区或合规的 AI 服务商

### 收益

- **成本优化**：可选择最具性价比的模型，按需切换
- **稳定性保障**：主供应商故障时快速切换备份，保证业务连续性
- **灵活性增强**：不同业务场景使用最适合的模型
- **自主可控**：不依赖单一供应商，降低供应链风险

### 使用者

- 后端开发工程师
- AI 应用开发者
- 系统架构师

---

## ② What - 定义 (是什么)

### 一句话定义

**LLM 多供应商接入**是指在系统中统一封装多个 AI 大模型供应商的接入能力，提供统一的接口抽象，使业务层可以无感切换和调用不同厂商的 AI 能力。

### 核心组成部分

```
┌─────────────────────────────────────────────────────────┐
│                    业务层 (Service)                      │
├─────────────────────────────────────────────────────────┤
│              AI 模型工厂 (AiModelFactory)                │
│                    统一入口                              │
├──────────┬──────────┬──────────┬──────────┬────────────┤
│ 通义千问  │ 文心一言  │ DeepSeek │ 智谱AI   │  ...更多   │
│ (阿里)    │ (百度)    │ (DeepSeek)│ (智谱)   │            │
├──────────┴──────────┴──────────┴──────────┴────────────┤
│           Spring AI 抽象层                               │
│     (ChatModel / ImageModel / EmbeddingModel)           │
└─────────────────────────────────────────────────────────┘
```

### 关键术语

| 术语 | 说明 |
|------|------|
| ChatModel | 对话模型，用于文本生成 |
| ImageModel | 图像生成模型 |
| EmbeddingModel | 向量 embedding 模型 |
| VectorStore | 向量存储 |
| API Key | 调用各厂商的凭证 |

---

## ③ How - 思维 (怎么做)

### 目标

实现一个支持多供应商的 AI 模型接入框架，满足企业级灵活切换和扩展需求。

### 范围

```
允许修改：
- yudao-module-ai/                    (AI 模块)
- yudao-framework/                    (框架层)

禁止修改：
- yudao-module-system/                (系统模块)
- yudao-module-bpm/                   (流程模块)
```

### 数据模型设计

**AI 模型表 (ai_model)**：
```java
public class AiModelDO {
    private Long id;              // 模型ID
    private String name;          // 模型名称
    private String platform;      // 平台标识 (如: TONG_YI, DEEP_SEEK)
    private String model;         // 模型名 (如: qwen-turbo)
    private Integer status;       // 状态 (0-停用, 1-启用)
    private Integer sort;         // 排序
    private Long creator;         // 创建者
    private Long updater;         // 更新者
    private Date createTime;      // 创建时间
    private Date updateTime;      // 更新时间
    private Boolean deleted;      // 是否删除
}
```

**AI API Key 表 (ai_api_key)**：
```java
public class AiApiKeyDO {
    private Long id;              // ID
    private Long modelId;         // 模型ID
    private String apiKey;        // API Key
    private String secretKey;     // Secret Key (部分厂商需要)
    private String url;           // 自定义 API 地址
    private Integer status;       // 状态
    private Date expireTime;      // 过期时间
    private Long creator;         // 创建者
    private Long updater;         // 更新者
    private Date createTime;      // 创建时间
    private Date updateTime;      // 更新时间
}
```

### 关键流程设计

```
┌──────────────┐     ┌──────────────────┐     ┌────────────────┐
│  业务调用     │ ──▶ │  AiModelFactory  │ ──▶ │  平台适配器    │
│ (getChatModel)│     │     (工厂)       │     │ (Switch Case) │
└──────────────┘     └──────────────────┘     └────────────────┘
                                                         │
                     ┌───────────────────────────────────┤
                     │              │              │     │
               ┌─────▼─────┐  ┌────▼────┐  ┌─────▼─────┐
               │ 通义千问   │  │DeepSeek │  │ 智谱AI    │
               │ DashScope │  │ SDK    │  │  SDK     │
               └───────────┘  └─────────┘  └───────────┘
```

### 关键代码设计

**工厂接口 (AiModelFactory.java)**：
```java
public interface AiModelFactory {
    // 获取或创建对话模型
    ChatModel getOrCreateChatModel(AiPlatformEnum platform, String apiKey, String url);
    
    // 获取默认配置的对话模型
    ChatModel getDefaultChatModel(AiPlatformEnum platform);
    
    // 获取图像模型
    ImageModel getDefaultImageModel(AiPlatformEnum platform);
    
    // 获取向量模型
    EmbeddingModel getOrCreateEmbeddingModel(AiPlatformEnum platform, String apiKey, String url, String model);
    
    // 获取向量存储
    VectorStore getOrCreateVectorStore(Class<? extends VectorStore> type, 
                                       EmbeddingModel embeddingModel,
                                       Map<String, Class<?>> metadataFields);
}
```

**平台枚举 (AiPlatformEnum.java)**：
```java
public enum AiPlatformEnum {
    // 国内平台
    TONG_YI("TongYi", "通义千问"),      // 阿里
    YI_YAN("YiYan", "文心一言"),        // 百度
    DEEP_SEEK("DeepSeek", "DeepSeek"),  // DeepSeek
    ZHI_PU("ZhiPu", "智谱"),            // 智谱 AI
    XING_HUO("XingHuo", "星火"),        // 讯飞
    DOU_BAO("DouBao", "豆包"),          // 字节
    HUN_YUAN("HunYuan", "混元"),        // 腾讯
    SILICON_FLOW("SiliconFlow", "硅基流动"),
    MINI_MAX("MiniMax", "MiniMax"),
    MOONSHOT("Moonshot", "月之暗面"),
    BAI_CHUAN("BaiChuan", "百川智能"),
    
    // 国外平台
    OPENAI("OpenAI", "OpenAI"),
    AZURE_OPENAI("AzureOpenAI", "AzureOpenAI"),
    ANTHROPIC("Anthropic", "Anthropic"),
    GEMINI("Gemini", "Gemini"),
    OLLAMA("Ollama", "Ollama"),
    GROK("Grok", "Grok");
}
```

---

## ④ Hard - 难点 (挑战)

### 难点 1：多厂商 SDK 差异统一

**场景**：每个 AI 厂商的 SDK 使用方式不同，参数命名和调用方式各异。

**解决方案**：通过 Spring AI 的统一抽象层，将各厂商差异封装在适配器内部，对上层提供统一接口。

### 难点 2：API Key 安全存储

**场景**：API Key 明文存储存在泄露风险。

**解决方案**：
- 加密存储 API Key
- 支持密钥管理服务（如阿里云 KMS）
- 定期轮换机制

### 难点 3：供应商切换的兼容性

**场景**：不同模型的输出格式、能力边界不同，切换后可能导致业务异常。

**解决方案**：
- 抽象统一响应格式
- 关键业务做模型能力检测
- 支持灰度切换和回滚

### 难点 4：Token 消耗统计与成本控制

**场景**：多供应商情况下，难以统一管理和控制成本。

**解决方案**：
- 统一埋点统计 token 消耗
- 设置成本预警阈值
- 支持按供应商独立计费

---

## ⑤ Metric - 衡量 (指标)

| 指标 | 权重 | 说明 | 验证方法 |
|------|------|------|----------|
| 供应商支持数量 ≥ 15家 | 20% | 支持的主流 AI 供应商数量 | 代码统计 |
| 接口调用成功率 ≥ 99.5% | 25% | 业务调用成功率 | 日志统计 |
| 切换响应时间 < 500ms | 15% | 切换供应商的耗时 | 性能测试 |
| API Key 安全性 | 20% | 敏感信息加密存储 | 安全审计 |
| 模型兼容性 | 20% | 统一抽象层的完整性 | 单元测试 |

---

## ⑥ Select - 选型 (选哪个)

### 候选方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Spring AI | 统一抽象、社区活跃、厂商支持全面 | 学习曲线较陡 | 新项目首选 |
| LangChain4j | Java 生态丰富、文档完善 | 厂商适配较少 | 需要复杂链式调用 |
| 自研适配层 | 完全可控、定制灵活 | 工作量大、维护成本高 | 有特殊定制需求 |

### 选型理由

**选择 Spring AI**，因为：
1. 官方已集成 30+ 主流 AI 供应商，开箱即用
2. 统一 ChatModel/ImageModel/EmbeddingModel 接口
3. 与 Spring Boot 无缝集成
4. 社区活跃度高，持续更新

### 工具集

- **代码分析**：IntelliJ IDEA / VSCode
- **API 测试**：Postman / Apifox
- **文档**：
  - Spring AI 官方文档：https://docs.spring.io/spring-ai/reference/
  - yudao-cloud AI 模块：https://github.com/YunaiCloud/yudao-cloud

---

## ⑦ Impl - 实现 (细节)

### 核心实体类

```java
// AI 模型
public class AiModelDO {
    private Long id;
    private String name;
    private String platform;
    private String model;
    private Integer status;
    private Integer sort;
}

// AI API Key
public class AiApiKeyDO {
    private Long id;
    private Long modelId;
    private String apiKey;
    private String secretKey;
    private String url;
    private Integer status;
}
```

### 核心实现类

**AiModelFactoryImpl 关键代码**：
```java
@Override
public ChatModel getOrCreateChatModel(AiPlatformEnum platform, String apiKey, String url) {
    String cacheKey = buildClientCacheKey(ChatModel.class, platform, apiKey, url);
    return Singleton.get(cacheKey, () -> {
        switch (platform) {
            case TONG_YI:
                return buildTongYiChatModel(apiKey);
            case DEEP_SEEK:
                return buildDeepSeekChatModel(apiKey);
            case ZHI_PU:
                return buildZhiPuChatModel(apiKey, url);
            // ... 其他平台
            default:
                throw new IllegalArgumentException("未知平台: " + platform);
        }
    });
}
```

### 关键步骤校验

| 步骤 | 校验点 | 验证方法 |
|------|--------|----------|
| 1. 配置 API Key | 配置文件正确加载 | 启动日志检查 |
| 2. 创建 ChatModel | 模型实例非空 | 单元测试 |
| 3. 调用模型 | 返回正常响应 | 集成测试 |
| 4. 切换模型 | 响应正常 | 切换测试 |

### 失败恢复机制

- **场景 1：API Key 无效**
  - 触发：调用时返回 401
  - 恢复：切换备用 API Key，记录告警

- **场景 2：供应商服务不可用**
  - 触发：网络超时或 5xx 错误
  - 恢复：自动切换到备用供应商

- **场景 3：响应格式异常**
  - 触发：解析失败
  - 恢复：记录异常日志，返回默认响应

---

## ⑧ SKILL - 提炼 (复用)

### 触发条件

```
场景1：需要接入新的 AI 供应商
场景2：需要在多个 AI 模型间灵活切换
场景3：需要统一管理 AI 供应商的 API Key
场景4：需要基于 AI 能力构建应用
```

### 执行流程

```
Step 1: 添加依赖
  - 在 pom.xml 中添加对应厂商的 Spring AI Starter
  - 示例：spring-ai-alibaba-starter

Step 2: 配置 API Key
  - 在 application.yaml 中配置
  - spring.ai.tongyi.api-key=xxx

Step 3: 注入使用
  - @Autowired AiModelFactory
  - ChatModel chatModel = factory.getOrCreateChatModel(platform, apiKey, url)

Step 4: 调用模型
  - UserMessage userMessage = new UserMessage("你好");
  - ChatResponse response = chatModel.call(userMessage);
```

### 配方/素材

**技术栈**：Java 17+, Spring Boot 3.x, Spring AI

**核心依赖**：
```xml
<!-- Spring AI 基础 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-core</artifactId>
</dependency>

<!-- 阿里云通义千问 -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
</dependency>

<!-- OpenAI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

**配置文件**：
```yaml
spring:
  ai:
    tongyi:
      api-key: ${TONGYI_API_KEY:}
    openai:
      api-key: ${OPENAI_API_KEY:}
      base-url: https://api.openai.com
```

### 验收标准

- [ ] 支持至少 15 家主流 AI 供应商
- [ ] 可以通过 API 动态切换供应商
- [ ] API Key 加密存储
- [ ] 单元测试覆盖核心方法
- [ ] 文档更新完毕

---

## 附录：支持的 AI 供应商列表

| 平台 | 标识 | 备注 |
|------|------|------|
| 通义千问 | TONG_YI | 阿里云 |
| 文心一言 | YI_YAN | 百度智能云 |
| DeepSeek | DEEP_SEEK | DeepSeek |
| 智谱 | ZHI_PU | 智谱 AI |
| 星火 | XING_HUO | 讯飞 |
| 豆包 | DOU_BAO | 字节 |
| 混元 | HUN_YUAN | 腾讯云 |
| 硅基流动 | SILICON_FLOW | 第三方 |
| MiniMax | MINI_MAX | MiniMax |
| 月之暗面 | MOONSHOT | KIMI |
| 百川智能 | BAI_CHUAN | 百川 |
| OpenAI | OPENAI | OpenAI |
| Azure OpenAI | AZURE_OPENAI | 微软 |
| Anthropic | ANTHROPIC | Claude |
| Gemini | GEMINI | 谷歌 |
| Ollama | OLLAMA | 本地部署 |
| Grok | GROK | xAI |