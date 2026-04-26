# 73-LLM接入 - 大模型接入之多供应商接入

> 本文档按照八步法（Why-What-How-Hard-Metric-Select-Impl-SKILL）分析 LLM 多供应商接入技术

---

## ① Why - 价值 (为什么)

### 背景与痛点

在大模型应用场景中，企业面临以下挑战：

1. **单一供应商风险**：依赖单一 API 可能导致服务中断、成本波动或政策变更
2. **成本优化需求**：不同供应商定价差异大，需要灵活切换以降低成本
3. **模型多样性**：不同模型擅长不同场景（如代码生成、创意写作、多语言处理）
4. **合规要求**：某些场景需要使用国产大模型（如通义千问、文心一言）

### 带来的价值

- ✅ **供应商隔离**：某供应商出问题可快速切换
- ✅ **成本控制**：根据需求选择性价比最高的模型
- ✅ **灵活扩展**：新增供应商只需实现接口，无需改动业务代码
- ✅ **国产化支持**：全面支持国产大模型，满足合规需求

### 使用者

- 后端开发工程师
- AI 应用开发者
- 需要构建智能客服、内容生成等业务的企业

---

## ② What - 定义 (是什么)

### 一句话定义

**LLM 多供应商接入**是指通过统一的抽象层，封装不同大模型供应商的 API 调用，实现模型灵活切换和统一管理。

### 核心组成部分

| 组件 | 说明 |
|------|------|
| **平台枚举 (AiPlatformEnum)** | 定义支持的 AI 平台（OpenAI、百度、阿里、讯飞等） |
| **模型工厂 (AiModelFactory)** | 统一创建和管理不同平台的 ChatModel |
| **配置属性 (YudaoAiProperties)** | 统一配置管理，支持多平台 API Key |
| **具体实现类** | 各平台特定的模型实现（Baichuan、DouBao、DeepSeek 等） |

### 关键术语

- **ChatModel**：Spring AI 定义的聊天模型接口
- **EmbeddingModel**：向量化模型接口
- **VectorStore**：向量数据库，用于 RAG 场景
- **Tool Calling**：函数调用，让 LLM 能调用外部工具

---

## ③ How - 思维 (怎么做)

### 数据模型设计

```java
// AI 平台枚举 - 支持 20+ 平台
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
    
    // 图像/音乐
    STABLE_DIFFUSION("StableDiffusion", "StableDiffusion"),
    MIDJOURNEY("Midjourney", "Midjourney"),
    SUNO("Suno", "Suno"),
    GROK("Grok", "Grok");
}
```

### 关键流程设计

```
用户请求 → Controller → Service → AiModelFactory 
                                    ↓
                         根据平台类型创建对应 ChatModel
                                    ↓
                         调用底层 SDK（如 Spring AI）
                                    ↓
                         返回 AI 响应
```

### 目录结构

```
yudao-module-ai/
├── yudao-module-ai-api/
│   └── src/main/java/cn/iocoder/yudao/module/ai/
│       └── enums/model/
│           ├── AiPlatformEnum.java        # 平台枚举
│           └── AiModelTypeEnum.java        # 模型类型
│
└── yudao-module-ai-server/
    └── src/main/java/cn/iocoder/yudao/module/ai/
        └── framework/ai/
            ├── config/
            │   ├── AiAutoConfiguration.java   # 自动配置
            │   └── YudaoAiProperties.java     # 配置属性
            └── core/model/
                ├── AiModelFactory.java        # 工厂接口
                ├── AiModelFactoryImpl.java    # 工厂实现（核心）
                ├── baichuan/BaiChuanChatModel.java
                ├── doubao/DouBaoChatModel.java
                ├── deepseek/DeepSeekChatModel.java
                ├── gemini/GeminiChatModel.java
                ├── hunyuan/HunYuanChatModel.java
                ├── xinghuo/XingHuoChatModel.java
                ├── siliconflow/               # 硅基流动（集成多模型）
                ├── midjourney/                # 图像生成
                └── suno/                      # 音乐生成
```

---

## ④ Hard - 难点 (挑战)

### 难点 1：多平台 API 差异

每个平台的 API 请求/响应格式不同，需要针对性适配：

```java
// 不同平台的请求参数差异
// 通义千问：使用阿里云 DashScope SDK
DashScopeChatModel.build(apiKey, options);

// 百度：使用 QianFan SDK
QianFanChatModel.build(apiKey, secretKey);

// DeepSeek：使用 Spring AI 原生支持
DeepSeekChatModel.build(apiKey, options);
```

### 难点 2：Token 计算方式不同

各平台对 Token 的计算方式不同，可能导致计费差异。

### 难点 3：模型版本管理

同一个平台有多个模型版本（如 GPT-4、GPT-4o、GPT-4o-mini），需要动态支持。

### 难点 4：国产大模型兼容

部分国产模型 SDK 不规范，需要额外适配。

---

## ⑤ Metric - 衡量 (指标)

| 指标 | 权重 | 说明 | 验证方法 |
|------|------|------|----------|
| 平台覆盖度 | 20% | 支持的主流平台数量 | 枚举类统计 |
| 切换灵活性 | 20% | 切换平台是否需要改代码 | 单元测试 |
| 可扩展性 | 15% | 新增平台的工作量 | 代码行数统计 |
| API 兼容性 | 15% | 请求/响应格式一致性 | 接口测试 |
| 文档完整性 | 10% | API 文档覆盖度 | 文档审查 |
| 社区活跃度 | 10% | Spring AI 社区支持 | GitHub Stars |
| 成本优化能力 | 10% | 是否支持成本监控 | 配置验证 |

---

## ⑥ Select - 选型 (选哪个)

### 候选方案

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **Spring AI** | 统一抽象、社区活跃、多供应商支持 | 学习曲线 | 中大型项目 |
| LangChain4j | Java 生态丰富 | 主要面向 Java 12+ | 需要复杂链式调用 |
| 自研 adapter | 完全可控 | 工作量大、维护成本高 | 特殊业务需求 |

### 选型理由

选择 **Spring AI**，因为：

1. ✅ **标准化接口**：提供统一的 ChatModel、EmbeddingModel 接口
2. ✅ **多供应商支持**：内置 30+ 供应商支持
3. ✅ **活跃社区**：Spring 官方维护，更新及时
4. ✅ **与 yudao-cloud 技术栈一致**：项目本身基于 Spring Boot

### 工具集

- **代码分析**：IntelliJ IDEA
- **API 测试**：Apifox、Postman
- **官方文档**：https://docs.spring.io/spring-ai/reference/

---

## ⑦ Impl - 实现 (细节)

### 核心配置

```yaml
# application.yaml
ai:
  platform: openai  # 当前使用的平台
  models:
    - platform: openai
      api-key: ${OPENAI_API_KEY}
      model: gpt-4o
    - platform: tongyi
      api-key: ${TONGYI_API_KEY}
      model: qwen-turbo
    - platform: deepseek
      api-key: ${DEEPSEEK_API_KEY}
      model: deepseek-chat
```

### 核心代码 - AiModelFactoryImpl

```java
@Override
public ChatModel getOrCreateChatModel(AiPlatformEnum platform, String apiKey, String url) {
    String cacheKey = buildClientCacheKey(ChatModel.class, platform, apiKey, url);
    return Singleton.get(cacheKey, () -> {
        switch (platform) {
            case TONG_YI:
                return buildTongYiChatModel(apiKey);
            case OPENAI:
                return buildOpenAIChatModel(apiKey);
            case DEEP_SEEK:
                return buildDeepSeekChatModel(apiKey);
            case ANTHROPIC:
                return buildAnthropicChatModel(apiKey);
            // ... 更多平台
            default:
                throw new IllegalArgumentException("不支持的平台: " + platform);
        }
    });
}

private ChatModel buildOpenAIChatModel(String apiKey) {
    OpenAiApi api = OpenAiApi.builder()
            .apiKey(apiKey)
            .build();
    return OpenAiChatModel.builder()
            .api(api)
            .defaultOptions(OpenAiChatOptions.builder()
                    .model("gpt-4o")
                    .build())
            .build();
}

private ChatModel buildTongYiChatModel(String apiKey) {
    DashScopeApi api = DashScopeApi.builder()
            .apiKey(apiKey)
            .build();
    return DashScopeChatModel.builder()
            .api(api)
            .build();
}
```

### 模型切换示例

```java
@Service
public class AiChatService {
    
    @Autowired
    private AiModelFactory aiModelFactory;
    
    public String chat(String platform, String message) {
        // 根据平台获取模型（自动缓存）
        ChatModel chatModel = aiModelFactory.getOrCreateChatModel(
            AiPlatformEnum.valueOf(platform),
            getApiKey(platform),
            null
        );
        
        // 统一调用方式
        ChatResponse response = chatModel.call(
            new UserMessage(message)
        );
        
        return response.getResult().getOutput().getText();
    }
}
```

### 支持的功能

1. **聊天completion**
2. **Function Calling（函数调用）**
3. **Embedding（向量化）**
4. **图像生成**（DALL-E、Midjourney、Stable Diffusion）
5. **音乐生成**（Suno）
6. **PPT 生成**（文心一言、讯飞）
7. **RAG 向量存储**（Milvus、Qdrant、Redis）

---

## ⑧ SKILL - 提炼 (复用)

### 触发条件

- 场景 1：需要接入多个 LLM 供应商
- 场景 2：需要动态切换 LLM 平台
- 场景 3：需要构建 RAG 应用
- 场景 4：需要调用 AI 生成图像/音乐

### 执行流程

```
Step 1: 引入依赖
  - 在 pom.xml 添加 yudao-module-ai 依赖
  - 添加具体平台的 SDK 依赖

Step 2: 配置 API Key
  - 在 application.yaml 配置 ai.platform
  - 配置各平台的 api-key

Step 3: 注入使用
  - 注入 AiModelFactory
  - 调用 getOrCreateChatModel 获取模型实例

Step 4: 调用 AI
  - 使用统一的 ChatModel 接口调用
  - 处理响应
```

### 配方/素材

**技术栈**：Java 17+, Spring Boot 3.x, Spring AI

**Maven 依赖**：
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-dashscope-spring-boot-starter</artifactId>
</dependency>
```

### 验收标准

- [ ] 能够切换使用不同平台的 LLM
- [ ] 能够正确处理各平台的 API 异常
- [ ] 支持 Function Calling
- [ ] 支持图像/音乐生成
- [ ] 单元测试通过

---

## 附录：支持的平台汇总

### 国内平台

| 平台 | 模型 | 说明 |
|------|------|------|
| 阿里云 | 通义千问 (TongYi) | qwen-turbo, qwen-plus, qwen-max |
| 百度 | 文心一言 (YiYan) | ernie-bot-turbo, ernie-bot-4 |
| 智谱 AI | 智谱 (ZhiPu) | glm-4, glm-4-flash |
| 讯飞 | 星火 (XingHuo) | spark-v3.0, spark-v3.5 |
| 字节 | 豆包 (DouBao) | doubao-pro-32k |
| 腾讯 | 混元 (HunYuan) | hunyuan-pro |
| MiniMax | MiniMax |abab6.5s-chat |
| 月之暗面 | Moonshot | moonshot-v1-8k |
| 百川智能 | BaiChuan | baichuan4 |
| 硅基流动 | SiliconFlow | 集成多种模型 |

### 国外平台

| 平台 | 模型 | 说明 |
|------|------|------|
| OpenAI | GPT-4o, GPT-4o-mini | 官方 API |
| Azure OpenAI | GPT-4 | 企业版 |
| Anthropic | Claude 3.5 | Claude 系列 |
| Google | Gemini | Gemini 系列 |
| DeepSeek | DeepSeek-V3 | 开源大模型 |
| Ollama | 本地模型 | 本地部署 |

### 生成能力

| 类型 | 平台 |
|------|------|
| 图像 | Midjourney, Stable Diffusion, DALL-E, 百度图片 |
| 音乐 | Suno |
| PPT | 文心一言、讯飞、文多多 |