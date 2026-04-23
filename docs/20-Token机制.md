# 20-Token机制

> 认证登录 - Token 机制与 Refresh

---

## ① Why - 价值

**背景与痛点**：
- 原来用 Session 鉴权，分布式部署时 Session 共享问题复杂
- 移动端/API 无法使用 Cookie-Session 模式
- 每次请求都查数据库验证用户，性能差

**收益**：
- 无状态鉴权，水平扩展方便
- 支持多端登录（Web/App/API）
- Redis 缓存 Token，性能提升

**用户**：
- 前端开发者、移动端开发者、API 消费者

---

## ② What - 定义

**一句话定义**：
Token 机制是一种无状态的身份认证方式，通过 OAuth2.0 协议生成访问令牌和刷新令牌来实现安全鉴权。

**核心组成**：
1. **访问令牌 (Access Token)** - 短期令牌，用于接口鉴权
2. **刷新令牌 (Refresh Token)** - 长期令牌，用于续期 Access Token
3. **客户端 (OAuth2 Client)** - 第三方应用身份标识
4. **授权范围 (Scope)** - 令牌权限范围

**关键术语**：
- OAuth2.0 - 开放授权标准
- JWT - JSON Web Token 无token格式
- Access Token - 访问令牌（默认2小时有效期）
- Refresh Token - 刷新令牌（默认30天有效期）

---

## ③ How - 思维

**目标**：实现 OAuth2.0 Token 鉴权能力

**范围**：
- 允许：yudao-module-system/（系统模块）
- 禁止：改动业务模块核心逻辑

**数据模型**：

```java
// OAuth2 访问令牌表 (system_oauth2_access_token)
OAuth2AccessTokenDO:
  - id              : 编号
  - accessToken     : 访问令牌
  - refreshToken    : 刷新令牌
  - userId          : 用户编号
  - userType        : 用户类型(1=管理员 2=会员)
  - clientId        : 客户端编号
  - scopes          : 授权范围
  - expiresTime     : 过期时间
```

```java
// OAuth2 客户端表 (system_oauth2_client)
OAuth2ClientDO:
  - id              : 客户端ID
  - name            : 客户端名称
  - secret          : 客户端密钥
  - grantTypes      : 授权类型(password/authorization_code/client_credentials/refresh_token)
  - scopes          : 授权范围
  - accessTokenValidity : 访问令牌有效期(秒)
  - refreshTokenValidity: 刷新令牌有效期(秒)
```

**核心流程**：

```
流程1：密码模式登录
用户 → 输入账号密码 → 验证用户 → 创建 Token → 返回 AccessToken + RefreshToken

流程2：刷新 Token
携带 RefreshToken → 验证 RefreshToken → 创建新 AccessToken → 返回新 Token

流程3：验证 Token
请求带 AccessToken → 解析 Token → 检查过期 → 获取用户信息 → 放行
```

---

## ④ Hard - 难点

**难点1：Token 泄露风险？**
```
场景：Token 被盗用怎么办？
→ 短期 Access Token（2小时）+ 长期 Refresh Token（30天）
→ 敏感操作需要额外验证（如支付密码）
→ 支持 Token 撤销（removeAccessToken）
```

**难点2：多端登录冲突？**
```
场景：用户手机和电脑同时登录，旧登录顶掉新登录？
→ 支持多个 Access Token 关联同一个用户
→ 可配置是否允许重复登录
```

**难点3：分布式 Token 验证？**
```
场景：多台服务器如何验证 Token？
→ Redis 缓存 Access Token，优先读缓存
→ 缓存未命中时查 MySQL，并回填 Redis
→ 双写：Redis + MySQL 保证数据一致性
```

**难点4：Refresh Token 过期？**
```
场景：用户长时间未操作，Refresh Token 也过期了？
→ 需要重新走登录流程
→ 可通过配置延长 Refresh Token 有效期
```

---

## ⑤ Metric - 衡量

| 指标 | 权重 | 说明 | 验证方法 |
|------|------|------|----------|
| Token 签发成功率≥99.9% | 30% | 成功签发次数/总请求 | 压测验证 |
| Token 验证耗时≤50ms | 25% | 从请求到验证完成 | 性能测试 |
| 并发签发能力≥1000/s | 20% | 每秒签发 Token 数 | 压测报告 |
| Token 续期成功率≥99% | 15% | Refresh 成功率 | 单元测试 |
| 支持多端登录 | 10% | 同时在线设备数 | 手动测试 |

---

## ⑥ Select - 选型

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| JWT | 无状态、可自包含信息 | 无法撤销、Token 较长 | 简单鉴权 |
| OAuth2.0 | 标准协议、支持多模式 | 实现复杂 | 开放平台 |
| Session | 实现简单 | 分布式不友好 | 单机部署 |

**选型理由**：
1. yudao 采用 OAuth2.0 标准，兼容性好
2. 支持多种授权模式（密码/授权码/客户端/刷新）
3. 双层 Token 设计（Access + Refresh），安全与体验平衡
4. Redis + MySQL 双存储，性能与持久兼顾

---

## ⑦ Impl - 实现

**核心代码**：

```java
// OAuth2TokenServiceImpl - 创建访问令牌
@Override
@Transactional(rollbackFor = Exception.class)
public OAuth2AccessTokenDO createAccessToken(Long userId, Integer userType, 
        String clientId, List<String> scopes) {
    // 1. 验证客户端
    OAuth2ClientDO clientDO = oauth2ClientService.validOAuthClientFromCache(clientId);
    
    // 2. 创建刷新令牌
    OAuth2RefreshTokenDO refreshTokenDO = createOAuth2RefreshToken(userId, userType, clientDO, scopes);
    
    // 3. 创建访问令牌
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

```java
// 刷新访问令牌
@Override
@Transactional(noRollbackFor = ServiceException.class)
public OAuth2AccessTokenDO refreshAccessToken(String refreshToken, String clientId) {
    // 1. 验证刷新令牌
    OAuth2RefreshTokenDO refreshTokenDO = oauth2RefreshTokenMapper.selectByRefreshToken(refreshToken);
    
    // 2. 检查过期
    if (DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
        throw new ServiceException("刷新令牌已过期");
    }
    
    // 3. 创建新访问令牌
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

```java
// 验证访问令牌
@Override
public OAuth2AccessTokenDO checkAccessToken(String accessToken) {
    // 1. 获取 Token（优先 Redis，后 MySQL）
    OAuth2AccessTokenDO accessTokenDO = getAccessToken(accessToken);
    
    // 2. 检查存在性和过期
    if (accessTokenDO == null) {
        throw new ServiceException("访问令牌不存在");
    }
    if (DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        throw new ServiceException("访问令牌已过期");
    }
    return accessTokenDO;
}
```

**授权模式枚举**：

```java
public enum OAuth2GrantTypeEnum {
    PASSWORD("password"),              // 密码模式：用户直接给账号密码
    AUTHORIZATION_CODE("authorization_code"), // 授权码模式：第三方应用跳转授权
    IMPLICIT("implicit"),              // 简化模式（已废弃）
    CLIENT_CREDENTIALS("client_credentials"), // 客户端模式：机器间调用
    REFRESH_TOKEN("refresh_token"),   // 刷新模式：用 Refresh Token 换 Access Token
}
```

**配置示例**：

```yaml
# application.yml
oauth2:
  access-token:
    expire-seconds: 7200  # 访问令牌2小时
  refresh-token:
    expire-seconds: 2592000  # 刷新令牌30天
```

---

## ⑧ SKILL - 提炼

**触发条件**：
- 场景1：需要实现用户登录功能
- 场景2：需要实现 Token 续期功能
- 场景3：需要实现第三方登录接入

**执行流程**：
```
Step 1: 配置 OAuth2 客户端
  - 在管理后台添加客户端（clientId、clientSecret）
  - 配置授权类型和授权范围

Step 2: 登录获取 Token
  - 调用 /oauth2/token 接口
  - 传入 grant_type、username、password、clientId、clientSecret
  - 获取 access_token 和 refresh_token

Step 3: 使用 Access Token
  - 在请求头 Authorization: Bearer <access_token>
  - 调用业务接口

Step 4: 刷新 Token
  - 当 access_token 过期
  - 调用 /oauth2/token 接口
  - 传入 grant_type=refresh_token、refresh_token、clientId、clientSecret
  - 获取新的 access_token
```

**配方**：
- 技术栈：Java 8+, Spring Boot, Redis, MySQL
- 依赖包：yudao-module-system（系统模块）
- 存储：Redis（缓存）+ MySQL（持久化）

**接口清单**：

| 接口 | 说明 |
|------|------|
| POST /oauth2/token | 获取/刷新 Token |
| DELETE /oauth2/logout | 登出（撤销 Token）|
| GET /oauth2/getTokenInfo | 获取 Token 信息 |

**使用示例**：

```bash
# 1. 密码模式登录
curl -X POST http://localhost:48080/oauth2/token \
  -d "grant_type=password" \
  -d "username=admin" \
  -d "password=admin123" \
  -d "clientId=admin-app" \
  -d "clientSecret=admin-app-secret"

# 2. 刷新 Token
curl -X POST http://localhost:48080/oauth2/token \
  -d "grant_type=refresh_token" \
  -d "refresh_token=xxx" \
  -d "clientId=admin-app" \
  -d "clientSecret=admin-app-secret"

# 3. 使用 Token 调用接口
curl -H "Authorization: Bearer {access_token}" \
  http://localhost:48080/system/auth/get-permission-info
```

**验收标准**：
- [ ] 功能正常：登录能获取 Token
- [ ] 单元测试通过：核心方法测试通过
- [ ] 性能达标：Token 验证<50ms
- [ ] Token 可续期：Refresh Token 能刷新 Access Token
- [ ] 支持登出：能撤销 Token