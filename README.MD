# 简化 Spring RESTful 后端中的 OAuth2 / OpenID


`8.1.24` 适配 Spring Boot `3.5.x`（Security `6.5.x` 和 Cloud `2025.0.x`）。

`9.0.5-M1` 适配 Spring Boot `4.0.0-M1`（Security `7.0.x` 和 Cloud `2025.1.x`）。在 Boot `4.0.0` 正式发布之前，源码在 boot-4 分支上维护。

[`spring-addons-starter-rest`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-rest) 在涉及 OAuth2、自签名 SSL 证书或 HTTP 代理的服务间调用场景中是一个颠覆性的工具，值得一试！

## Spring 后端所需的 OAuth2 基础知识

本仓库提供了[使用 OAuth2 / OIDC 配置 Spring RESTful 后端的教程](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials#securing-spring-applications-with-oauth2)。**切记，与其花几周时间反复试错，不如花 15 分钟好好读一遍 README。** 因此，在开始实现之前，请务必仔细阅读 [_OAuth2 essentials_](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials#1-oauth2-essentials) 章节，以及 [`spring-addons-starter-oidc`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-oidc#spring-addons-starter-oidc) 和 [`spring-addons-starter-rest`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-rest#auto-configure-restclient-or-webclient-beans) 的 README。

本仓库中有三篇教程已迁移至 Baeldung：
- [Getting started with Keycloak & Spring Boot](https://www.baeldung.com/spring-boot-keycloak)
- [Creating an OAuth2 BFF with `spring-cloud-gateway` and consuming it with Single-Page Applications](https://www.baeldung.com/spring-cloud-gateway-bff-oauth2)
- [Testing access control with mocked OAuth2 authentications](https://www.baeldung.com/spring-oauth-testing-access-control)

## [`spring-addons-starter-oidc`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-oidc)

仅使用 `spring-boot-starter-oauth2-client` 或 `spring-boot-starter-resource-server` 时，我们几乎总是需要自己编写 `Security(Web)FilterChain`，这不仅需要扎实的安全背景知识和对 Spring Security 内部机制的了解，而且代码往往十分冗长。

`spring-addons-starter-oidc` 构建在"官方" starter 之上，能显著减少安全配置代码的编写量，在大多数情况下甚至可以将其降至 0。

**我们对 `spring-addons-starter-oidc` 的自动配置拥有完全的控制权。** 当然可以通过应用配置属性来控制，也可以通过 bean 定义来控制：几乎所有自动配置的组件都标注了 `@ConditionalOnMissingBean`，这意味着每当应用中显式定义了某个组件时，`spring-addons` 就会退让。无需定义完整的 security filter-chain，只需定义要覆盖的那个组件即可。

resource server 的自动配置包括：
- 接受由多个受信任授权服务器签发的 token
- 从多种 claim（包括嵌套 claim）中映射 authorities，支持自定义前缀和大小写
- CORS 配置
- 使用 CORS 配置中的路径匹配器允许匿名预检请求

使用 `oauth2Login` 的 client 的自动配置包括：
- 在 authorization-code 流程和 RP-Initiated Logout 流程中自定义返回给前端的响应：
  - 在 `Location` 响应头中指定 URI，以便在登录/登出后激活对应路由（默认值可在应用配置中定义，前端可通过请求头或查询参数覆盖）
  - 避免与授权服务器之间的某些 CORS 问题：将 HTTP 状态码设置在 `2xx` 范围内，以便在 Javascript 代码中观察响应并处理重定向，而不是让浏览器在 Ajax 请求中直接跟随跳转。这些重定向本就不应该是跨域请求，而应该是普通的页面导航。
- 将 CSRF token 作为单页应用可访问的 cookie 暴露出来
- 从未严格实现 RP-Initiated Logout 的授权服务器（例如 Auth0 和 Amazon Cognito）注销
- 启用并配置 Back-Channel Logout
- 向授权请求和 token 请求添加额外参数（例如 Auth0 所需的 `audience`）
- CORS 配置
- 使用 CORS 配置中的路径匹配器允许匿名预检请求

## [`spring-addons-starter-rest`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-rest)

在 OpenAPI 规范可以从 REST API 源码自动生成、消费这些 API 的客户端代码又可以从规范自动生成的今天，服务间通信的主要挑战在于 REST client 的配置。

Spring 推荐使用 `RestClient` 或 `WebClient`，但以下场景的配置往往相当复杂、冗长且容易出错：
- `Basic` 或 `Bearer` 认证
- 通过 HTTP 代理
- 连接超时和读取超时
- 信任 SSL bundle 中配置的 SSL 证书
- 禁用 SSL 证书验证
- 将底层 HTTP 客户端库从 JDK 自带的切换为 `http-components` 或 `jetty`

`spring-addons-starter-rest` 让这些配置变得轻而易举。

示例用法
```yaml
com:
  c4-soft:
    springaddons:
      rest:
        client:
          # 暴露一个名为 machinClient 的 RestClient bean（在 WebFlux 应用中为 WebClient）
          machin-client:
            base-url: ${machin-api}
            authorization:
              oauth2:
                # 使用 security context 中的 Bearer token 授权出站请求（仅在 resource server 应用中可用）
                forward-bearer: true
                # 配置 RestClient / WebClient 信任 SSL bundle 中描述的证书（应指向 spring.ssl.bundle 配置中的某个条目）
                ssl-bundle: client
          # 暴露一个名为 biduleClientBuilder 的 RestClient.Builder bean（注意 "expose-builder: true"）
          bidule-client:
            base-url: ${bidule-api}
            # 暴露 builder 而非已构建好的 client（以便进一步微调配置）
            expose-builder: true
            authorization:
              oauth2:
                # 使用通过 OAuth2 client registration 获取的 Bearer token 授权出站请求
                oauth2-registration-id: bidule-registration
```
这会暴露预配置好的 `RestClient` 或 `WebClient` bean（或其 builder），可以在任意类型的 `@Component` 中自动注入——例如 `@Controller` 和 `@Service`——也可以在 `@Configuration` 中使用，例如为 `@HttpExchange` 接口生成实现类并将其暴露为 bean。

只要设置了 `HTTP_PROXY` 和 `NO_PROXY` 环境变量，代理配置默认会自动应用于 REST client。此行为可通过应用配置属性覆盖或禁用。

## 带安全上下文的单元测试与集成测试

测试访问控制需要在测试的 security context 中配置一个经过精细调整的 `Authentication` 实例。

为此，`spring-security-test` 提供了 `MockMvc` 请求后处理器和 `WebTestClient` mutator，但它们只能在请求上下文中工作，因此仅适用于 controller 的测试。**要测试任意类型的 `@Component`**（当然包括 `@Controller`，也包括 `@Service` 和 `@Repository`），只有两种选择：
- 自行构建测试 security context，并用 stub / mock 的 authentication 填充它 :cry:
- **使用注解**来代劳（这正是 [spring-addons-oauth2-test](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-oauth2-test) 大显身手的地方）:smiley:

此外，`@MockJwt` 与 `spring-security-test` 中注解的一个显著区别在于：**`spring-security-test` 会忽略 security 配置中定义的 authentication converter :sob:**。为了理解其影响，我们来看一下在使用 JWT decoder 的 resource server 中构建 security context 的流程：
1. JWT Bearer 字符串被 `JwtDecoder` 解码、验证，并转换为 `org.springframework.security.oauth2.jwt.Jwt`
2. 这个 `Jwt`（非 JWT）被 authentication converter 转换为继承自 `AbstractAuthenticationToken` 的对象。此步骤包括将 claim 转换为 authorities，以及选择具体的 `Authentication` 实现类。
3. `Authentication` 实例被放入 security context

使用 `@WithJwt` 时，只有第 1 步被 mock。系统会使用测试资源中的 JSON payload 构建一个 stub `Jwt`（非 JWT），并将其传给 authentication converter。而使用 `spring-security-test` 的后处理器和 mutator 时，工厂会跳过到第 3 步，自行构建 stub `Authentication`，并用测试代码中提供的内容设置其属性。因此，authorities 转换逻辑只有在使用 `@WithJwt` 时才会被执行。同样，自定义的 `Authentication` 实现类只有在工厂调用了 authentication converter 时才会在测试中被使用，因此使用 `@WithJwt` 时有效，使用 `.jwt()` 后处理器时则无效。

相关资源：
- [spring-addons-oauth2-test](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-oauth2-test) 包含测试注解，其 README 中有使用说明
- [spring-addons-starter-oidc-test](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-oidc-test) 适用于使用 `spring-addons-starter-oidc` 的项目
- [Baeldung article](https://www.baeldung.com/spring-oauth-testing-access-control)
- [samples](https://github.com/ch4mpy/spring-addons/tree/master/samples) 和 [tutorials](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials) 源码（包含大量单元测试和集成测试）

## 相关链接
- [`spring-addons-starter-oidc`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-oidc) 将 OAuth2 client 和 resource server 安全自动配置提升到新高度的 Spring Boot starter
- [`spring-addons-oauth2-test`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-oauth2-test) 用于向测试 security context 填充 OAuth2 authentication 实例的注解
- [`spring-addons-starter-oidc-test`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-oidc-test) 简化使用 `spring-addons-starter-oidc` 的应用的单元测试
- [`spring-addons-starter-rest`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-starter-rest) 针对 `RestClient`、`WebClient` 和 `@HttpExchange` 代理的实验性自动配置（base-URL、Basic 和 OAuth2 Bearer 认证）
- [Getting started with Keycloak & Spring Boot](https://www.baeldung.com/spring-boot-keycloak)
- [OAuth2 security configuration tutorials](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials#securing-spring-applications-with-oauth2)（含/不含 `spring-addons-starter-oidc` 两种版本）
- [OAuth2 BFF tutorial](https://www.baeldung.com/spring-cloud-gateway-bff-oauth2)
- [Release Notes](https://github.com/ch4mpy/spring-addons/tree/master/release-notes.md)
- [Maven-Central 备忘](https://github.com/ch4mpy/spring-addons/tree/master/maven-central.md)