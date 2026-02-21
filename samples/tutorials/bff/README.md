# 使用 Spring Cloud Gateway 实现 OAuth2 **B**ackend **F**or **F**rontend 模式
介绍以 `spring-cloud-gateway` 作为中间件，在通过 session cookie 保护的单页应用或移动应用，与通过 JWT 保护的 Spring OAuth2 resource server 之间实现 OAuth2 **B**ackend **F**or **F**rontend 模式。

包含使用 **Angular、React（Next.js）和 Vue（Vite）** 编写的前端示例。

OAuth2 BFF 教程现已上线 [Baeldung](https://www.baeldung.com/spring-cloud-gateway-bff-oauth2)。

### 定义
**B**ackend **F**or **F**rontend 是前端与 REST API 之间的中间件，可用于多种不同目的。本教程关注的是 **OAuth2 BFF**，它用于在**基于 session cookie 的请求授权**（前端提供）和**基于 Bearer token 的请求授权**（resource server 期望）之间架起桥梁。其职责包括：
- 使用"机密"OAuth2 client 驱动 authorization-code 流程
- 维护 session 并将 token 存储其中
- 在将前端请求转发给 resource server 时，将 session cookie 替换为 session 中的 access token

### 相较于公开 OAuth2 Client 的优势
核心价值在于安全性：
- BFF 运行在我们信任的服务器上，**授权服务器的 token endpoint 可以通过 secret 和防火墙规则加以保护，仅允许来自后端的请求**。这大幅降低了 token 被签发给恶意 client 的风险。
- **token 保存在服务器端（session 中），防止其在终端用户设备上被恶意程序窃取**。使用 session cookie 需要防范 CSRF，但 cookie 可以设置 `HttpOnly`、`Secure` 和 `SameSite` 标志，此时 cookie 在设备上的保护由浏览器自身强制执行。相比之下，配置为公开 client 的 SPA 需要直接访问 token，我们必须非常谨慎地处理 token 的存储方式：一旦恶意程序成功读取 access token 或 refresh token，对用户造成的后果可能是灾难性的（身份冒用）。

另一个优势是对用户 session 的完全掌控能力，以及即时撤销访问权限的能力。

### 代价
BFF 是系统中额外增加的一层，且处于关键路径上。在生产环境中，这意味着：
- 更多资源消耗（少量）
- 更高延迟（极少）
- 更多监控与故障恢复工作

此外，BFF 后面的 resource server 可以（也应该）是无状态的，但 OAuth2 BFF 本身需要 session，这要求采取专门措施来保证其可扩展性和容错性。

我们可以使用 Spring Boot 的 Maven 和 Gradle 插件轻松将 Spring Cloud Gateway 打包为原生镜像，使其极为轻量并能在极短时间内启动。但任何单实例都有其流量上限。当需要超过一个实例时，必须在 BFF 实例之间共享 session，或使用智能代理将来自同一设备的所有请求路由到同一个实例。

### 实现方案的选择
部分框架在没有明确提及或专门命名的情况下，实际上已经实现了 OAuth2 BFF 模式。例如 NextAuth 库，它使用服务器组件在 Node 服务端实例中实现 OAuth2（使用机密 client），足以获得 OAuth2 BFF 模式带来的安全性。

但得益于极为丰富的 Spring 生态，在监控、可扩展性和容错性方面，几乎没有比 Spring Cloud Gateway 更趁手的解决方案：
- `spring-boot-starter-actuator` 依赖提供了强大的审计功能
- Spring Session 是相对简单的分布式 session 解决方案
- `spring-boot-starter-oauth2-client` 和 `oauth2Login()` 处理 authorization-code 流程并将 token 存储在 session 中
- `TokenRelay=` 过滤器在将前端请求转发给 resource server 时，将 session cookie 替换为 session 中的 access token