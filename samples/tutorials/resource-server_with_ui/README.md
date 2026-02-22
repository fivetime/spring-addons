# 在单个应用中混合 OAuth2 Client 和 Resource Server 配置
本教程的目标是**将一个 Spring 后端同时配置为 OAuth2 client 和 resource server，并允许用户从多个异构受信任授权服务器列表中选择认证**：一个本地 Keycloak realm 和一个云端 Auth0 实例。

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 前言
我们将定义两个有序的独立 security filter chain：
- 第一个采用 client 配置，包含登录、登出，并通过 security matcher 将其限制在 UI 资源范围内
- 第二个采用 resource server 配置。由于没有 security matcher 且优先级更低，它拦截所有未被第一个 filter chain 匹配的请求，作为所有其余资源（REST API）的默认处理器

需要特别注意的是，在此配置中，**浏览器并非 OAuth2 client**：它通过普通 session 保护。因此，**必须在 client filter chain 中启用 CSRF 和 BREACH 保护**。

UI 通过 session cookie 保护，REST 端点通过 JWT 保护，因此 Thymeleaf `@Controller` 内部使用 `RestClient` 从 API 获取数据并构建模板的 model，使用存储在 session 中的 token 对请求授权。

不同的 provider 在不同的 Spring profile 中配置。如果我们选择在主 profile 中配置多个 `provider`，并为每个 provider 分别创建带有 `authorization_code` 的 `registration`，**那么用户在切换 `registration` 登录时必须先登出**。这是因为 Spring Security 的 security context 只能包含一个 `Authentication`，而在 `oauth2Login` 场景下，该 `Authentication` 绑定到特定的 `registration`。

运行本示例前，请确保你的环境满足[教程前置条件](https://github.com/fivetime/spring-addons/blob/master/samples/tutorials/README.md#prerequisites)。

## 2. 场景详情
我们将实现一个包含以下功能的 Spring 后端：
- 一个 resource server（REST API）
  * 接受来自 3 个不同 issuer（Keycloak、Auth0 和 Cognito）的身份
  * 无状态（CSRF 禁用）
  * 对未授权请求返回 401（未授权）
  * 提供包含已认证用户名和角色的个性化问候消息
  * 对 `@Controller` 暴露的 REST 端点、Swagger REST 资源（OpenAPI 规范）和 actuator 定义访问控制
- 一个供上述 resource server 使用的 Thymeleaf client
  * 要求用户登录
  * 需要 session（因为浏览器的请求不会携带 Bearer token，同时也应启用 CSRF 保护）
  * 用户没有 session 时返回默认的 302（重定向到登录页）
  * 一个认证后加载的首页，包含指向 Thymeleaf 页面和 Swagger-UI 首页的链接
  * 对所有 OAuth2 client 和 UI 资源定义访问控制：登录、登出、授权回调和 Swagger-UI
  * 一个"问候"页面，用户可以在此：
    - 获取每个已连接 identity provider 的问候语
    - 从尚未认证的已配置 identity provider 中添加一个新身份
    - 逐个或全部从已连接的 identity provider 登出
    - 在不断开 identity provider 连接的情况下，使 Thymeleaf client 上的 session 失效

## 3. 项目初始化
从 https://start.spring.io/ 创建一个 Spring Boot 3 项目，添加以下依赖：
- lombok
- spring-boot-starter-web（REST API 和 UI servlet 均使用）
- spring-boot-starter-thymeleaf
- spring-boot-starter-actuator

然后添加以下依赖：
- [`spring-addons-starter-oidc`](https://central.sonatype.com/artifact/com.c4-soft.springaddons/spring-addons-starter-oidc)
- [`spring-addons-starter-rest`](https://central.sonatype.com/artifact/com.c4-soft.springaddons/spring-addons-starter-rest)
- [`spring-addons-starter-oidc-test`](https://central.sonatype.com/artifact/com.c4-soft.springaddons/spring-addons-starter-oidc-test)（`test` scope）
```xml
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc</artifactId>
    <version>${spring-addons.version}</version>
</dependency>
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-rest</artifactId>
    <version>${spring-addons.version}</version>
</dependency>
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc-test</artifactId>
    <version>${spring-addons.version}</version>
    <scope>test</scope>
</dependency>
```

## 4. Web Security 配置
本教程使用 `spring-addons-starter-oidc`，它会根据配置文件自动配置两个 `SecurityFilterChain` bean（一个带有 `oauth2ResourceServer`，一个带有 `oauth2Login`）。**这些 security filter chain 并未在 security 配置中显式定义，但它们确实存在！**

### 4.1. 应用配置属性
请参阅源码。

`rest` 下的属性定义了一个名为 `greetClient` 的 `RestClient` bean 的配置，该 bean 使用 client-credentials registration 对 REST API 的请求进行授权。

不要忘记将 issuer URI 以及 client ID 和 secret 替换为你自己的（或通过命令行参数、环境变量等方式覆盖）。

### 4.2. OAuth2 Security Filter Chain
**完全不需要编写任何 Java 代码。**

## 5. Resource Server 组件
由于用户名和角色已完成映射，从 security context 中的 `Authentication` 实例构建同时包含两者的问候语非常简单：
```java
@RestController
@RequestMapping("/api")
@PreAuthorize("isAuthenticated()")
public class ApiController {
    @GetMapping("/greet")
    public String getGreeting(JwtAuthenticationToken auth) {
        return "Hi %s! You are granted with: %s.".formatted(auth.getName(), auth.getAuthorities());
    }
}
```

## 6. `oauth2Login` 组件
`oauth2Login` 为 authorization code 和 refresh token 流程配置 OAuth2 client。提醒一下，带有 `oauth2Login` 的应用是有状态的（依赖 session，就像任何带有"登录"功能的服务端应用一样），token 存储在 session 中（位于服务端）。

### 6.1. 登出
这部分比较复杂。需要记住的是，每个用户在带有 `oauth2Login` 的 Spring 应用上有一个 session，在 OpenID Provider 上也有另一个独立的 session。

如果只使 client 上的 session 失效，下次用同一浏览器尝试登录时，很可能会静默完成（当用户在授权服务器上的 session 仍然有效时，授权服务器通常不会再次要求输入凭据）。要实现完整登出，**client 和授权服务器上的 session 都应被终止**。

OIDC 规定了两种登出协议：
- [RP-Initiated Logout](https://openid.net/specs/openid-connect-rpinitiated-1_0.html)：client（Relying Party）请求授权服务器终止用户 session
- [Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html)：授权服务器向已注册的 client 列表广播登出事件，每个 client 据此终止其持有的用户 session

本教程仅涵盖 RP-Initiated Logout，其流程如下：
1. User-agent 向 Relying Party（即带有 `oauth2Login` 的 Spring 应用，简称 RP）的 `/logout` 端点发送 `POST` 请求
2. RP 终止该 user-agent 的 session，并将其重定向到 OpenID Provider 的 `end_session_endpoint`（位于 `/.well-known/openid-configuration` 的 OpenID 配置中）
3. OP 终止该 user-agent 的 session，并将其重定向到请求参数中提供的 `post_logout_redirect_uri`

由于 RP 的安全机制基于 session，需要防范 CSRF 攻击。而且**由于初始登出请求是 `POST`，它必须携带有效的 CSRF token**。本配套项目通过 `javascript` Spring profile 在两种场景之间切换：
- 使用默认 profile 时，`POST` 请求通过普通 HTML 表单发送。保留 Spring Security 的默认 CSRF token repository（`HttpSessionCsrfTokenRepository`），token 处理由 Spring 的 Thymeleaf 集成透明完成（会在 DOM 中添加一个包含 `_csrf` token 值的 `hidden` 输入框）
- 激活 `javascript` profile 时，登出请求通过 JQuery 的 `ajax` 函数发送。一些额外的 `application.yml` 属性让 `spring-addons-starter-oidc`：
  - 使用 `CookieCsrfTokenRepository`，并将该 cookie 的 `HttpOnly` 标志设为 `false`（使 Javascript 可以读取 CSRF token 值）
  - 将 RP 响应（上述第 `2.` 步结束时）的状态码从 `302` 切换为 `202`。这使 Javascript 代码能够观察到响应，并通过普通导航跳转到 OP 的 `end_session_endpoint`（避免了 ajax 请求跨域重定向时的 CORS 错误）

### 6.2. 调用 REST 微服务
session 一旦"授权"（用户已登录），其中就包含一个 access token。REST client 可以配置为将此 token 作为 `Authorization` 请求头中的 `Bearer` 来授权其对 OAuth2 resource server 的请求。在本项目中，我们使用 `spring-addons-starter-rest` 自动配置一个带有 `OAuth2ClientHttpRequestInterceptor` 的 `RestClient` 实例来实现这一点。我们还为描述所要调用的 resource server 的 `@HttpExchange` 接口生成了代理（参见 `RestClientsConfig` 和 `GreetApi`）。

## 7. 总结
在本教程中，我们了解了如何配置不同的 security filter chain，并选择每个 filter chain 应用于哪些路由。我们设置了：
- 一个带有登录、登出和 session（以及 CSRF 保护）的 OAuth2 client filter chain，用于 UI
- 一个用于 REST API 的无状态（无 session 也无 CSRF 保护）filter chain

我们也体会到了 `spring-addons-starter-oidc` 和 `spring-addons-starter-rest` 在配置 RP 安全机制以及调用受 OAuth2 保护的 REST 微服务时有多么便捷。