# 使用 OAuth2 保护 Spring 应用
本系列教程专注于在带有 OpenID Provider 的 Spring Boot 3 应用中配置 OAuth2 安全机制。

**在直奔具体教程之前，请务必仔细阅读 [OAuth2 基础知识](#oauth_essentials) 章节。** 这将为你节省大量时间，并让你了解一些最新建议（如仅使用保密客户端、对用户设备隐藏 token 等），这些建议对单页应用有重大影响，也使得网络上大多数现有教程已经过时。

确定要配置的应用是 OAuth2 client 还是 OAuth2 resource server，并[设置至少一个 OIDC Provider](#prerequisites) 之后，参阅[教程场景](#scenarios)，选择符合你需求的那一个。

跳转至：
- [1. OAuth2 基础知识](#oauth_essentials)
- [2. 前置条件](#prerequisites)
- [3. 教程场景](#scenarios)


## 1. <a name="oauth_essentials"/>OAuth2 基础知识
OAuth2 client 和 resource server 的配置差异显著。**Spring 提供不同 starter 是有原因的。** 如果你对这两者的定义、用途和职责还不太确定，请花 5 分钟阅读本节再开始。你也可以在 [https://quiz.c4-soft.com/ui/quizzes](https://quiz.c4-soft.com/ui/quizzes) 上**通过专项问答测试你的 OAuth2 / OpenID 知识。**

### 1.1 参与方
- **resource-owner（资源所有者）**：可以理解为终端用户。通常是真实的人，但也可以是通过 client-credential 认证的批处理程序或其他受信任程序（甚至是通过某种流程认证的设备，此处略过不表）
- **authorization-server（授权服务器）**：负责签发和认证 resource-owner 及 client 身份的服务器，有时也被称为 *issuer* 或 *OIDC Provider*（*OP*）
- **client（客户端）**：需要访问一个或多个 resource server 上资源的软件。**它负责从授权服务器获取 token 并为其对 resource server 的请求授权**，因此需要处理 OAuth2 流程。有时也被称为 *Relying Party*（*RP*）
- **resource-server（资源服务器）**：通常是一个 API（最常见的是 REST API）。**它不应该关心登录、登出或任何 OAuth2 流程。** 从其视角来看，唯一重要的是请求是否携带有效的 access token，以及基于该 token 做出访问控制决策

值得注意的是，**前端不一定是 OAuth2 client**：在 **B**ackend **F**or **F**rontend（BFF）模式中，OAuth2 client 位于服务端，处于 resource server（通过 access token 保护）与 Web（Angular、React、Vue 等）或移动应用之间——后者通过 session 保护，从不接触 OAuth2 token。

### 1.2. Client 与 Resource Server 配置的区别
如前所述，两者的职责和安全要求差异显著，下面进一步展开。

#### 1.2.1. 对 Session 的需求
**Resource server 通常可以配置为无状态（不使用 session）。** "状态"与 access token 关联，足以恢复请求的 security context。这对可扩展性和容错性有重要价值：任何 resource server 实例都可以处理任何请求，无需共享 session。此外，access token 还能防范 CSRF 攻击，如果轮换频率足够高（约每分钟一次），也能防范 BREACH 攻击！

**浏览器访问的 client 通过 session cookie 而非 access token 来保护。** 这使其暴露于 CSRF 和 BREACH 攻击，我们需要针对性地配置缓解措施。另外，一旦需要考虑可扩展性和容错性，就必须将 session 从 client 实例中剥离出去。

#### 1.2.2. 请求授权
Resource server 期望请求通过包含 `Bearer` access token 的 `Authorization` 请求头来授权。

Client 负责为其对 resource server 的请求授权：设置这个 `Authorization` 请求头。Client 可以通过不同的 OAuth2 流程从授权服务器获取 token（详见下一节）。为避免每次请求都获取新 token，client 还需要保存 token，并且必须非常谨慎地选择足够安全的存储位置，以防 token 泄露给恶意代码（远程设备的持久化存储在这方面是个相当糟糕的选择）。

Resource server 不关心 access token 是如何获取的。它的职责仅限于验证 token 的有效性（issuer、audience、过期时间等），然后根据 token 中的 claim（token 内部或通过 introspection 获取）决定是否授予所请求的资源。

用户登录是 OAuth2 `authorization-code` 流程的一部分。因此，**OAuth2 登录（和登出）只在配置了 `authorization-code` 流程的 OAuth2 client 上才有意义**。

要向受保护的 resource server 发送请求，需要使用能够发送授权请求的 client，例如：
- 带 UI 的 REST 客户端，如 Postman
- 配置了 OAuth2 client 库来处理流程、token 存储和请求授权的"富"浏览器应用（Angular、React、Vue 等），作为公开客户端使用。但请注意[现在已不推荐这种方式](https://github.com/spring-projects/spring-authorization-server/issues/297#issue-896744390)
- 程序化 REST 客户端（`RestClient`、`WebClient`、`@FeignClient`、`RestTemplate` 等），用于在微服务之间调用受 OAuth2 保护的 API
- BFF。**B**ackend **F**or **F**rontend 是一种在服务端使用中间件（BFF）将 OAuth2 token 对浏览器隐藏的模式。浏览器与 BFF 之间的请求通过 session 保护，BFF 负责登录、登出、在 session 中存储 token，并在将浏览器请求转发给 resource server 之前将 session cookie 替换为 OAuth2 access token。[`spring-cloud-gateway` 可配合 `spring-boot-starter-oauth2-client` 和 `TokenRelay` 过滤器用作 BFF](https://www.baeldung.com/spring-cloud-gateway-bff-oauth2)

#### 1.2.3. 选择 `spring-boot-starter-oauth2-client` 还是 `spring-boot-starter-oauth2-resource-server`？
如果应用是 REST API，由于其"无状态"特性，应将其配置为 resource server，使用 `spring-boot-starter-oauth2-resource-server`，不配置 OAuth2 登录，要求 client 对请求授权（测试时使用 Postman 或类似工具）。

如果应用提供 UI 模板或用作 BFF，则使用 `spring-boot-starter-oauth2-client`。只有在这种情况下，登录和登出才会在 Spring 应用中配置（否则由 Postman 或其他 OAuth2 client 来管理）。

如果应用同时满足以上两种情况（例如同时公开暴露 REST API 和操作该 API 的 Thymeleaf UI）怎么办？如前所述，两者的配置要求差异太大，无法在同一个 security filter chain 中配置，但**可以定义多个 filter chain，前面的（按 `@Order`）带有 `securityMatcher` 来定义其适用的请求范围**：请求的路径（或其他任意请求属性如请求头）会依次与每个 security filter chain 的"matcher"进行匹配，第一个匹配的 `SecurityFilterChain` bean 将被应用于该请求。

### 1.3. 流程
OAuth2 流程有不少，其中 3 种与我们关系最大：authorization-code、client-credentials 和 refresh-token。

无论使用哪种流程，一旦 client 拥有了 token，就可以为其对 resource server 的请求授权：在请求头中设置带有 `Bearer` access token 的 `authorization` 字段。

Resource server 验证 token 并获取用户信息的方式有两种：
- 使用本地 JWT decoder，只需授权服务器的公钥（一次性获取，用于所有请求）
- 向授权服务器的 introspection 端点提交 token（对其处理的每个授权请求各调用一次，会引入延迟并对授权服务器造成较大压力）

#### 1.3.1. Authorization-Code
**用于代表终端用户（真实的人）对 client 进行认证。**

0. Client 和 resource server 从 OIDC Provider 获取 OpenID 配置
1. 前端"跳出"，通过系统浏览器将未授权用户重定向到授权服务器。若用户在授权服务器上已有打开的 session，则登录静默成功；否则，提示用户输入凭据、生物识别、MFA token 或 OP 上配置的其他方式
2. 用户认证成功后，授权服务器通过系统浏览器将用户重定向回 client，并附带一个一次性使用的 `code`
3. Client 联系授权服务器，用 `code` 换取 access token（以及可选的 ID token 和 refresh token）
4. 前端通过 OAuth2 client 中转向 resource server 发送 REST 请求（OAuth2 client 将 session cookie 替换为包含 `Bearer` access token 的 `Authorization` 请求头）
5. Resource server 验证 access token（使用一次性获取的 JWT 公钥，或在 OP 上 introspect 每个 token），并做出访问控制决策

![authorization-code flow](https://github.com/fivetime/spring-addons/blob/master/.readme_resources/authorization-code_flow.png)

在上图中，authorization-code 流程从第 1 步开始，到第 3 步结束。

对于原生应用，可以在第 2 步使用 Android App Links 或 iOS Universal Links 等机制将 authorization-code 传递给前端，前端再使用自己的 user agent 将 code 转发给 OAuth2 client。由于第 3 步获取的 token 存储在与提供 authorization-code 的 user agent 关联的 session 中，**前端发送 authorization-code 时使用的 user agent 必须与后续发送需授权 REST 请求时使用的 user agent 一致，这一点非常重要。**

对于 SPA，user agent 是系统浏览器，因此第 2 步将 authorization-code 发送给 OAuth2 client 无需特别处理。第 3 步结束时，OAuth2 client 以重定向响应前端（浏览器重新进入 SPA）。

对于服务端渲染 UI（Thymeleaf、JSF 等），OAuth2 client 即是前端，一切都在内部发生，用户几乎无感知。

#### 1.3.2. Client-Credential
**用于以 client 自身身份进行认证**（不依赖用户上下文）。通常向授权服务器提供 client-id 和 client-secret。**此流程只能用于运行在你信任的服务器上的 client**（能够真正保密地保存 secret），不适用于在浏览器或移动应用中运行的服务（代码可被逆向工程读取 secret）。此流程常用于服务间通信（获取配置、发送日志或追踪事件、消息发布/订阅等）。

#### 1.3.3. Refresh-Token
Client 将 refresh-token 发送给授权服务器，授权服务器返回新 token 以替换即将过期的旧 token。refresh-token 不应发送给授权服务器之外的任何服务器。

### 1.4. Token
#### 1.4.1. Token 格式
**JWT** 是 JSON Web Token，主要用作 OAuth2 的 access token 或 ID token。JWT 可以自行验证：只需授权服务器的公钥签名即可。

除了使用 JWT decoder，access token 也可以通过 introspection 验证。这种方式适用于任何 token 格式（不一定是 JWT，token 被视为*不透明的*），但需要 resource server 向授权服务器发送请求来确认 token 有效并获取 token "属性"（相当于 JWT 的 "claim"）。这一过程会在整体请求处理中引入延迟，并给授权服务器带来额外压力。

#### 1.4.2. Access Token
类似于你授权他人代表你投票的纸质委托书。以下是它通常包含的几个 claim：
- `iss`（issuer）：签发该 token 的授权服务器（负责认证委托双方身份的公证人等）
- `sub`（subject）：resource-owner 的唯一标识符（授予委托书的人）
- `azp`（authorized parties）：client 的唯一标识符（委托书授予的对象）
- `aud`（audience）：应接受该 token 的 resource server（委托书的使用场所）
- `scp`（scope）：resource-owner 允许 client 代表其执行的操作（投票、管理银行账户、在邮局取包裹等）
- `exp`（expiration）：该 token 的有效期至

Client 在向 resource server 发送请求时，将 token 作为 `Bearer` `Authorization` 请求头发送。access token 的内容应仅由授权服务器和 resource server 关注（client 不应尝试读取 access token）。

#### 1.4.3. Refresh-Token
Client 在 access token 过期时（或最好在过期前）发送给授权服务器以获取新 access token 的 token。refresh-token 的有效期通常很长，可用于获取多个 access token。一旦泄露，用户将面临严重的身份盗用风险。因此，client 必须非常谨慎地保存 token，并确保 refresh-token 只发送给签发它的授权服务器。

#### 1.4.4. ID-Token
OAuth2 的 OpenID 扩展的一部分，client 用于获取用户信息的 token。

### 1.5. Scope、角色、权限、分组等
需要特别注意的是，`scope` 并不代表用户在系统中被允许执行的操作（如角色、权限等），而是**他授权某个 client 代表其执行的操作**。可以将其理解为 client 访问 resource-owner 资源前覆盖的一层掩码。

因此，在大多数情况下，scope 并不适合（或不能单独）作为 Spring Security authorities 的来源，我们需要自定义 authorities converter，将授权服务器用于角色、权限、分组等的私有 claim 映射为 authority，从而做出基于角色的安全决策。

## 2. <a name="prerequisites"/>前置条件
运行这些教程至少需要一个 OIDC Provider（授权服务器），但为了充分体验，最好同时拥有下一小节中提到的 3 个。

此外，一个带 UI 的 REST 客户端非常实用，可以从授权服务器获取 token 并向 resource server 实例发送授权测试请求。[Postman](https://www.postman.com/) 是一个广为人知的选择。

最后，你需要了解各授权服务器在哪个私有 claim 中存放用户名和角色，这没有统一标准。Keycloak 使用 `realm_access.roles`（如果启用了 client roles mapper，还有 `resource_access.{clientId}.roles`），其他授权服务器则各有不同。可以使用 https://jwt.io 等工具检查 access token，找出某个 issuer 用于存放角色的 claim。

### 2.1. 授权服务器
所有示例均配置为接受来自以下 3 个来源的身份：
* [本地 Keycloak realm](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/keycloak.md)（Keycloak 是开源免费的）
* [Auth0](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/auth0.md)
* [Cognito](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/cognito.md)

Auth0 和 Cognito 均提供免费套餐，足以运行教程和示例。你需要注册自己的实例和 client，获取 client-id 和 client-secret，并更新配置文件。

配置完 OIDC Provider 后，记得更新教程中的相关配置。

### 2.2. SSL
在交换 access token 时务必使用 https，否则 token 可能泄露，用户身份可能被盗用。因此，许多工具和库在使用 http 时会发出警告。如果还没有证书，请为开发机[生成一个自签名证书](https://github.com/fivetime/self-signed-certificate-generation)。

## 3. <a name="scenarios"/>教程场景
以下内容首先介绍仅使用"官方" Spring Boot starter 的教程，然后是使用本仓库提供的替代 starter 的教程。

这样安排有三重目的：
- 演示使用 `spring-addons-starter-oidc` 后 OAuth2 配置有多简单
- 说明在"官方" starter 已有的自动配置基础上，额外自动配置了哪些内容
- 演示仅使用 `spring-addons-oauth2-test` 时测试注解的用法。`3.1.` 和 `3.2.` 项目中的测试有三种版本：
  * MockMvc 请求后处理器或 WebTestClient mutator
  * `@WithMockAuthentication`，以内联方式定义 authorities 和 name
  * `@WithMockJwt`，从 classpath 资源中加载 claim-set，并使用 security 配置中的 `Converter<Jwt, ? extends AbstractAuthenticationToken>` 将其转换为 Authentication 实例

### 3.1. 仅使用 `spring-boot-starter-oauth2-resource-server` 的 OAuth2 Resource Server
将 Spring Boot 3 应用配置为 OAuth2 resource server（REST API），并映射 authorities 以使用 OIDC Provider 上定义的角色实现 RBAC。

这些教程仅使用"官方" `spring-boot-starter-oauth2-resource-server`，提供 [servlet](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/servlet-resource-server) 和[响应式](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/reactive-resource-server)两个版本。

### 3.2. 仅使用 `spring-boot-starter-oauth2-client` 的 OAuth2 Client
将 Spring Boot 3 应用配置为 OAuth2 client（Thymeleaf UI），支持登录、登出，并映射 authorities 以使用 OIDC Provider 上定义的角色实现 RBAC。

这些教程仅使用"官方" `spring-boot-starter-oauth2-client`，提供 [servlet](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/servlet-client) 和[响应式](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/reactive-client)两个版本。

### 3.3. [`resource-server_with_oauthentication`](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/resource-server_with_oauthentication)
演示如何使用自定义 OAuth2 `Authentication` 实现：带有 OpenID claim 类型化访问器的 `OAuthentication<OpenidClaimSet>`。

本教程引入 `spring-addons-starter-oidc`，与 `3.1.` 章节相比大幅简化了 Java 配置：所有 Java 配置均由应用配置属性替代。

### 3.4. [`resource-server_with_specialized_oauthentication`](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/resource-server_with_specialized_oauthentication)
在上一节基础上进一步展示如何：
- 扩展 `OAuthentication<OpenidClaimSet>` 实现以添加自定义私有 claim
- 调整 `spring-addons-webmvc-jwt-resource-server` 的自动配置
- 丰富 security SpEL 表达式

### 3.5. [`resource-server_with_additional-header`](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/resource-server_with_additional-header)
在 access token 之外使用自定义请求头来构建自定义 authentication。

### 3.6. [`resource-server_with_introspection`](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/resource-server_with_introspection)
与 `resource-server_with_oauthentication` 类似，但使用 token introspection 而非 JWT decoder。请注意这可能对性能产生影响。

### 3.7. [`resource-server_with_ui`](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/resource-server_with_ui)
将 Spring Boot 3 应用同时配置为 OAuth2 client（Thymeleaf UI）和 OAuth2 resource server（REST API）。

实现方式是定义两个有序的独立 security filter chain：
- 第一个采用 client 配置，包含登录、登出，并通过 security matcher 将其限制在 UI 资源范围内
- 第二个采用 resource server 配置。由于没有 security matcher 且优先级更低，它拦截所有未被第一个 filter chain 匹配的请求，作为所有其余资源（REST API）的默认处理器

Thymeleaf 页面通过 session cookie 保护，REST 端点通过 JWT 保护，因此 Thymeleaf `@Controller` 内部使用 `WebClient` 从 API 获取数据并构建模板的 model，使用存储在 session 中的 token 对请求授权。

### 3.8. [使用 Spring Cloud Gateway 的 OAuth2 BFF](https://www.baeldung.com/spring-cloud-gateway-bff-oauth2)
介绍 OAuth2 **B**ackend **F**or **F**rontend 模式，使用 `spring-cloud-gateway` 作为中间件，连接通过 session cookie 保护的单页或移动应用与通过 JWT 保护的 Spring OAuth2 resource server。

包含使用 Angular、React（Next.js）和 Vue（Vite）编写的示例前端。

### 3.9. [具有动态租户的 Resource Server](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/resource-server_multitenant_dynamic)
在本教程中，resource server 应接受 Keycloak 服务器上任意 realm 签发的 access token（即使该 realm 是在 resource server 启动后才创建的）。