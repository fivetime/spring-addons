# spring-addons-starter-oidc

本项目是一个 Spring Boot starter，用于在 `spring-boot-starter-oauth2-client` 或 `spring-boot-starter-oauth2-resource-server` 的基础上进一步**简化与任意 OpenID Provider（甚至多个异构 Provider）的 OAuth2 配置**。

```xml
    <properties>
        <springaddons.version>8.1.23</springaddons.version>
    </properties>
    
    <dependencies>
        <!-- 以下是对 spring-boot-starter-oauth2-resource-server 或 spring-boot-starter-oauth2-client（或两者）的补充，而非替代 -->
        <dependency>
            <groupId>com.c4-soft.springaddons</groupId>
            <artifactId>spring-addons-starter-oidc</artifactId>
            <version>${springaddons.version}</version>
        </dependency>
        <dependency>
            <groupId>com.c4-soft.springaddons</groupId>
            <artifactId>spring-addons-starter-oidc-test</artifactId>
            <version>${springaddons.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

跳转至：
- [0. 使用本库的风险与应对措施](#drawbacks)
- [1. 功能特性](#features)
  * [1.1. OAuth2 Resource Server](#features-resource-server)
    - [1.1.1. Resource Server `Security(Web)FilterChain`](#1-1-1)
    - [1.1.2. Authorities Converter](#1-1-2)
    - [1.1.3. Authentication Converter](#1-1-3)
    - [1.1.4. 多租户](#1-1-4)
    - [1.1.5. 访问控制](#1-1-5)
    - [1.1.6. CORS 配置](#1-1-6)
    - [1.1.7. 后处理 Resource Server Filter Chain](#1-1-7)
  * [1.2. 使用 `oauth2Login` 的 OAuth2 Client](#features-client)
    - [1.2.1. Client `Security(Web)FilterChain`](#1-2-1)
    - [1.2.2. 为 Client 设置 Base URI](#1-2-2)
    - [1.2.3. Authorization Code](#1-2-3)
    - [1.2.4. RP-Initiated Logout](#1-2-4)
    - [1.2.5. Back-Channel Logout](#1-2-5)
    - [1.2.6. Authorities Converter](#1-2-6)
    - [1.2.7. CSRF 保护](#1-2-7)
    - [1.2.8. 访问控制](#1-2-8)
    - [1.2.9. 后处理 Client Filter Chain](#1-2-9)
    - [1.2.10. 向 Token 请求添加参数](#1-2-10)
    - [1.2.11. 登录/登出后的跳转 URI](#1-2-11)
- [2. 常见问题](#faq)
  * [自动配置的具体内容是什么？](#2-1)
  * [`spring-addons-starter-oidc` 定义的所有 bean 都会被添加到应用上下文中吗？](#2-2)
  * [为什么要大费周章地处理 authorities 映射？直接用默认的 scope 不行吗？](#2-3)
  * [如何配置 resource server 以接受运行时动态创建的 Keycloak *realm* 签发的 token？](#2-4)
  * [如何为 Auth0 用户登录提供所需的 `audience` 参数？](#2-5)
  * [如何为 Auth0 client-credentials 提供所需的 `audience` 参数？](#2-6)
  * [前端可以覆盖 OAuth2 重定向的响应状态码吗？](#2-9)
  * [`iss` 配置属性是必填的吗？](#2-10)
  * [为什么 Microsoft 授权服务器很难配置？](#2-11)
  * [如何添加其他请求授权机制（如 `Basic` 认证）？](#2-12)
  * [如何配置 authorized client manager 通过 HTTP 代理？](#2-13)
  * [关于 cookie 的注意事项？](#2-14)
- [3. 使用方法](#usage)
  * [3.1. Resource Server](#3-1)
  * [3.2. Client](#3-2)
  * [3.3. Client 与 Resource Server 并存](#3-3)

## 0. <a name="drawbacks"/>使用本库的风险与应对措施
本仓库的维护者及所有贡献者均非 Spring Security 团队成员，充其量只是偶尔向"官方"框架提交 PR 的外部贡献者。

不过，除非你对 Spring Security 的 OAuth2 配置有深入了解，使用 `spring-addons-starter-oidc` 实际上可能比自行摸索配置更安全。而且本仓库的使用者越多，就越安全：更多人能发现潜在问题，更多人能在原作者失联时维护它，如果某些功能足够受欢迎，Spring 团队甚至可能将其纳入官方框架。

此外，如果你在任何时候想要不再使用 `spring-addons-starter-oidc`，只需自己编写 Spring Security 配置即可。如需参考，以下教程均未使用 `spring-addons-starter-oidc`（仅使用 `spring-boot-starter-oauth2-client` 或 `spring-boot-starter-oauth2-resource-server`）：
- [reactive-client](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/reactive-client)
- [reactive-resource-server](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/reactive-resource-server)
- [servlet-client](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/servlet-client)
- [servlet-resource-server](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/servlet-resource-server)

使用此类仓库资源有哪些已知风险，又该如何防范？
- *如果依赖更新停止、PR 不再被合并怎么办？* 你可以 fork 本仓库，从你喜欢的最后一个 tag 新建分支继续维护。
- *如果库的发展方向不符合我的需求怎么办？* 与上条相同。
- *如果仓库所有者删除或私有化此仓库怎么办？* 每个版本的源码都已发布到 maven-central。你可以从[那里](https://repo1.maven.org/maven2/com/c4-soft/springaddons/)或 GitHub 上的任意 fork（2023 年 8 月时约有 50 个）获取源码。
- *这些库中的代码引入漏洞的风险有多大？* 这取决于具体的库：
  * 测试期间使用的库（`spring-addons-oauth2-test` 和 `spring-addons-starter-oidc-test`）应以 `test` scope 引入，不会出现在运行时，生产环境无风险。
  * `spring-addons-starter-oidc` 会为你做一些自动配置：它定义了参与应用安全的 Spring bean。因此，如果此库中某个默认配置有误，确实可能影响你的应用。但请考虑以下几点：
    - 越来越多的用户在检查它，发现问题时会提 issue 或 PR（这个社区发现 Spring Security 配置问题的能力很可能远超你自己的团队）
    - 代码集中在一处、在多处复用，比在各自的应用中粗心犯错的风险更低

## 1. <a name="features"/>功能特性
根据 classpath 和应用配置属性，`spring-addons-starter-oidc` 最多可自动配置两个优先级较低的 security filter chain bean：
- 无状态的，使用 `oauth2ResourceServer`（基于 `Bearer` access token 的请求授权）
- 有状态的，使用 `oauth2Login`（基于 session cookie 的请求授权）

我们可以替换这些 filter chain 所使用的任意自动配置 bean。

如果需要自动配置之外的其他请求授权机制（`Basic` 认证、API key 等），可以定义优先级更高的额外 filter chain——并设置严格的 security-matcher，以确保自动配置的 filter chain 仍有机会处理它们应该处理的请求。

### 1.1. <a name="features-resource-server"/>OAuth2 Resource Server
提醒一下，OAuth2 resource server 的请求授权是通过使用 JWT decoder（或 introspection，但应尽量避免，因为其固有的延迟和扩展性问题）验证 access token 来实现的。

Resource server 只关心 token 是否有效，以及是否应根据 token 中的 claim 授予对资源的访问权限。Resource server 不关心 token 是如何获取的。因此，**登录和登出不属于 resource server 的配置范畴**。

#### 1.1.1. <a name="1-1-1"/>Resource Server `Security(Web)FilterChain`
如果 `spring-boot-starter-oauth2-resource-server` 在 classpath 中，且未设置 `com.c4-soft.springaddons.oidc.resourceserver.enabled=false`，则会创建一个具有以下默认配置的 `Security(Web)FilterChain`：
- `@Order(Ordered.LOWEST_PRECEDENCE)`，无 security matcher（作为默认处理器，处理所有未被其他优先级更高的 `Security(Web)FilterChain` 拦截的请求）
- 无状态（无 session，CSRF 保护禁用）
- 对未授权请求返回 401
- 若设置了 `spring.security.oauth2.resourceserver.opaquetoken.introspection-uri` 则使用 token introspection，否则使用 JWT decoder
- CORS 禁用（提醒：`cors` 属性配置的是全局过滤器）
- 允许匿名访问 `cors` 属性中路径匹配器对应的预检请求，以及路径匹配 `permit-all` 中条目的所有请求；其他所有请求均需有效认证

#### 1.1.2. <a name="1-1-2"/>Authorities Converter
Spring Security 通过所谓的 `GrantedAuthority`（通过 `Authentication#getAuthorities` 访问）实现基于角色的访问控制（RBAC）。

OpenID 和 OAuth2 均未包含 RBAC 规范（如 FAQ 中所述，scope 并非角色）。几乎所有 OpenID Provider 都实现了 RBAC，但它们不得不使用私有 claim 来实现。由于每个 OP 使用自己的私有 claim，将 OP 的*角色*映射为 Spring 的 *authority* 需要针对每个 issuer 进行适配。

默认情况下，`spring-addons-starter-oidc` 使用 `ConfigurableClaimSetAuthoritiesConverter`，该 converter 使用为每个 OP 定义的配置属性。默认的解析器 `ByIssuerOpenidProviderPropertiesResolver` 使用 access token 的 `iss` claim（*issuer*）来选择提供给 authorities 映射器的属性。

对于每个 OpenID Provider（OP），可以定义任意数量的 claim 组，每组可配置：
- `path`：指向要映射为 authority 的 claim 的 [JSON path](https://github.com/json-path/JsonPath)。可以使用 [https://jsonpath.com/](https://jsonpath.com/) 等工具对 access token payload（可用 [https://jwt.io](https://jwt.io) 等工具提取）测试 JSON path 表达式
- `prefix`：可选的前缀，添加到 OP 角色之前（默认为空）。例如，可以添加 `ROLE_` 前缀，从而使用 `hasRole('admin')` 而非 `hasAuthority('admin')`（前提是授权服务器提供的角色本身不是 `ROLE_admin`）
- `caze`：可选地将角色强制转为大写或小写（默认保持原样）

针对两个不同 OP 的示例配置：
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: https://oidc.c4-soft.com/auth/realms/master
          authorities:
          - path: $.realm_access.roles
          - path: $.resource_access.*.roles
        - iss: https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl
          authorities:
          - path: $.cognito:groups
            prefix: EXTERNAL_
            caze: upper
```
在上述配置中：
- 对于 `"iss": "https://oidc.c4-soft.com/auth/realms/master"` 的 token，authority 将从 token 的 *realm roles* 和所有可用的 *client roles* 中映射，不做任何转换。
- 对于 `"iss": "https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl"` 的 token，authority 将从 *cognito:groups* claim 中映射，强制转为大写并添加 `EXTERNAL_` 前缀（`"cognito:groups": ["machin", "truc"]` 将变为 `["EXTERNAL_MACHIN", "EXTERNAL_TRUC"]`）

若要使用其他 authorities 映射器，暴露一个类型为 `ClaimSetAuthoritiesConverter` 的 `@Bean` 即可。

若要修改 authorities 映射属性的解析方式（例如在使用"动态"多租户且无法在编写配置时确定所有可能的 issuer 的场景下），暴露一个类型为 `OpenidProviderPropertiesResolver` 的 `@Bean` 即可。

#### 1.1.3. <a name="1-1-3"/>Authentication Converter
Spring Security 的 `Authentication` 不仅仅是 *authority* 的容器：它还持有用户唯一标识符（`name`），以及在 resource server 场景下的 access token claim。

默认情况下，`spring-addons-starter-oidc` 使用 `(Reactive)JwtAbstractAuthenticationTokenConverter` 或 `(Reactive)OpaqueTokenAuthenticationConverter` 的实现，将 authority 转换委托给上下文中的 `@Bean`（见上一节）。默认输出与通常情况相同：
- 使用 JWT decoder 时输出 `JwtAuthenticationToken`
- 使用 introspection 时输出 `BearerTokenAuthentication`

通过暴露自定义的 authentication converter bean，可以使用自己的 `Authentication` 实现。以下是一个使用 JWT decoder 的 servlet 应用示例：
```java
@Bean
JwtAbstractAuthenticationTokenConverter authenticationConverter(
        Converter<Map<String, Object>, Collection<? extends GrantedAuthority>> authoritiesConverter,
        OpenidProviderPropertiesResolver opPropertiesResolver) {
    return jwt -> {
        final var opProperties = opPropertiesResolver.resolve(jwt.getClaims());
        final var claims = new OpenidClaimSet(jwt.getClaims(), opProperties.getUsernameClaim());
        final var authorities = authoritiesConverter.convert(jwt.getClaims());
        return new OAuthentication<>(claims, authorities, jwt.getTokenValue());
    };
}
```

#### 1.1.4. <a name="1-1-4"/>多租户
Resource server 的 JWT decoder 支持多租户。

多租户的核心组件是 `OpenidProviderPropertiesResolver`，负责解析用于构建 JWT decoder（及其内嵌验证器）的配置属性。

默认的 `OpenidProviderPropertiesResolver` 实现 `ByIssuerOpenidProviderPropertiesResolver` 支持"静态"多租户（即在编写配置时已知所有受信任的 issuer）。

你也可以通过暴露自定义的 `OpenidProviderPropertiesResolver` bean 来实现"动态"多租户。例如，该 bean 可以根据 issuer claim 动态解析构建新 JWT decoder 所需的属性，判断条件可以是：
- 以特定字符串开头
- 属于特定（子）域名
- 匹配某个模式、通过查询数据库的服务验证等任意条件

以下示例展示了如何接受 Keycloak 实例中任意 realm 签发的 token，即使该 realm 是在 resource server 启动后才创建的：
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: https://oidc.c4-soft.com/auth/realms/
          authorities:
          - path: $.realm_access.roles
```
```java
@Component
public class IssuerStartsWithOpenidProviderPropertiesResolver implements OpenidProviderPropertiesResolver {
    private final SpringAddonsOidcProperties properties;

    public IssuerStartsWithOpenidProviderPropertiesResolver(SpringAddonsOidcProperties properties) {
        this.properties = properties;
    }

    @Override
    public Optional<OpenidProviderProperties> resolve(Map<String, Object> claimSet) {
        final var tokenIss = Optional
            .ofNullable(claimSet.get(JwtClaimNames.ISS))
            .map(Object::toString)
            .orElseThrow(() -> new RuntimeException("Invalid token: missing issuer"));
        return properties.getOps().stream().filter(opProps -> {
            final var opBaseHref = Optional.ofNullable(opProps.getIss()).map(URI::toString).orElse(null);
            if (!StringUtils.hasText(opBaseHref)) {
                return false;
            }
            return tokenIss.startsWith(opBaseHref);
        }).findAny();
    }
}
```
这样，issuer claim 以 `https://oidc.c4-soft.com/auth/realms/` 开头的任意 token 都会被接受，并使用与同一 Keycloak 实例下任意 realm 签发的 token 相同的 converter 和配置属性映射为 `Authentication` 实例。

#### 1.1.5. <a name="1-1-5"/>访问控制
默认访问规则设置为 `isAuthenticated()`，但有两个例外：
- 路径匹配 `permit-all` 属性中条目的路由，允许匿名访问
- 预检请求（`OPTIONS` 请求且路径匹配 CORS 属性中的路径匹配器），除非在 `cors` 属性中禁用

定义细粒度访问控制最便捷的方式，可能是启用 `@Enable(Reactive)MethodSecurity` 并在 `@RestController` 方法上使用 `@PreAuthorize`。

对于偏好在配置中定义访问控制的场景（或不是自己编写 endpoint 的场景），可以暴露类型为 `ResourceServerExpressionInterceptUrlRegistryPostProcessor` 或 `ResourceServerAuthorizeExchangeSpecPostProcessor` 的 `@Bean`。

#### 1.1.6. <a name="1-1-6"/>CORS 配置
从版本 `7.8.7` 开始，CORS 配置通过全局过滤器实现，其行为由应用配置属性控制：
```java
com:
  c4-soft:
    springaddons:
      oidc:
        cors:
        - path: /machin/**
          allowed-origin-patterns: "*"
        - path: /truc/**
          allowed-origin-patterns:
          - "http://localhost:4200"
          - "http://*.chose.com"
```
在以下情况下，`spring-addons` 的 CORS 过滤器会退让：
- CORS 配置属性缺失（无 CORS 属性 -> 无 CORS 过滤器）
- 应用配置中已注册了 `Cors(Web)Filter` bean

默认情况下，`cors` 属性组中所有路径匹配器对应的匿名 `OPTIONS` 请求均被允许。

#### 1.1.7. <a name="1-1-7"/>后处理 Resource Server Filter Chain
通过暴露 `ResourceServer(Server)HttpSecurityPostProcessor` bean，可以在 `resourceServerSecurityFilterChain` 构建前完全控制其 `(Server)HttpSecurity` 的配置，从而修改任何预配置内容。

### 1.2. <a name="features-client"/>使用 `oauth2Login` 的 OAuth2 Client
OAuth2 client 是从授权服务器获取 token、再用 token 向 resource server 授权请求的应用。Spring Security 的 `oauth2Login` 配置了 authorization code 流程和 refresh token 流程。

当 filter chain 上配置了 `oauth2Login` 时，该 filter chain 必须是有状态的（token 存储在 session 中）并受 CSRF 保护。

注意，`spring-addons-starter-oidc` 会为所有 OAuth2 client 创建默认的 `(Reactive)OAuth2AuthorizedClientProvider` 和 `(Reactive)OAuth2AuthorizedClientManager`（即使禁用了带有 `oauth2Login` 的 `clientSecurityFilterChain`），以处理 token 请求中的附加参数。与往常一样，暴露自定义 bean 即可覆盖这些默认 bean。

#### 1.2.1. <a name="1-2-1"/>Client `Security(Web)FilterChain`
只有满足以下**所有**条件时，才会创建带有 `oauth2Login` 的 client filter chain：
- `spring-boot-starter-oauth2-client` 在 classpath 中
- `spring.security.oauth2.client.registration` 中至少有一个 `authorization-grant-type=authorization_code` 的条目
- `com.c4-soft.springaddons.oidc.client.security-matchers` 不为空

该 filter chain 的默认配置如下：
- `@Order(Ordered.LOWEST_PRECEDENCE + 1)`
- 应用配置中的 security-matcher
- 有状态（session 和 CSRF 保护启用）
- oauth2Login
- 启用 RP-Initiated Logout（前提是 OpenID 配置暴露了 end-session endpoint，或 spring-addons 属性中配置了非标准登出）
- 禁用 Back-Channel Logout
- 禁用 PKCE
- 禁用 CORS（提醒：`cors` 属性配置的是全局过滤器）
- 允许匿名访问预检请求及路径匹配 `permit-all` 中条目的所有请求；其他所有请求均需有效认证

#### 1.2.2. <a name="1-2-2"/>为 Client 设置 Base URI
Authorization code 流程和 RP-Initiated Logout 涉及重定向到授权服务器再重定向回 client。

Spring Security 会自动生成这些重定向 URI，但有时需要让其指向 client 本身之外的其他主机：网关、反向代理、ingress 等。`com.c4-soft.springaddons.oidc.client.client-uri` 正是为此而设。

#### 1.2.3. <a name="1-2-3"/>Authorization Code
可以通过设置 `com.c4-soft.springaddons.oidc.client.pkce-forced=true` 强制对保密客户端也启用 PKCE（默认情况下，Spring 仅对"公开"客户端启用 PKCE）。

其他功能主要在远程前端通过 `oauth2Login` 连接 Spring 后端（OAuth2 BFF）的场景下有用。

可以通过暴露以下各种 bean 来实现自定义：
- `PreAuthorizationCode(Server)RedirectStrategy`：默认实现为 `SpringAddonsPreAuthorizationCode(Server)RedirectStrategy`，允许修改发起 authorization code 流程时重定向到授权服务器的状态码（通过 `com.c4-soft.springaddons.oidc.client.oauth2-redirections.pre-authorization-code` 属性、`X-RESPONSE-STATUS` 请求头或 `response_http_status` 请求参数）。这对希望动态切换 HTTP 客户端的前端很有用（例如移动应用在 Spring client 上使用带 session 的程序化客户端，在授权服务器登录时使用另一个 session 的 web-view 或系统浏览器）
- `(Server)OAuth2AuthorizationRequestResolver`：默认实现为 `SpringAddons(Server)OAuth2AuthorizationRequestResolver`，功能包括：
  * 若 `pkce-forced` 设为 `true` 则强制使用 PKCE
  * 切换 client base URI（若有配置）
  * 将登录后成功/失败 URI 保存到 session（通过 `X-POST-LOGIN-SUCCESS-URI` & `X-POST-LOGIN-FAILURE-URI` 请求头或 `post_login_success_uri` & `post_login_failure_uri` 请求参数提供）
  * 添加在 `com.c4-soft.springaddons.oidc.client.authorization-request-params.{registrationId}` 下定义的可选请求参数（例如 Auth0 要求在授权请求中添加 `audience` 参数，见 FAQ 示例）
- `(Server)AuthenticationSuccessHandler`：默认实现会从 session 恢复登录后成功 URI，并应用 `com.c4-soft.springaddons.oidc.client.oauth2-redirections.post-authorization-code` 配置
- `(Server)AuthenticationFailureHandler`：默认实现会从 session 恢复登录后失败 URI，并应用 `com.c4-soft.springaddons.oidc.client.oauth2-redirections.post-authorization-code` 配置

补充说明：可以在应用配置中定义默认的登录后 URL，并在登录时由前端通过设置请求头或请求参数来覆盖。一个常见的用例是在 authorization code 流程完成后恢复前端的当前路由。

最后，Spring 对未授权请求的默认状态码（`302` 重定向到登录页）被保留，但可以通过 `com.c4-soft.springaddons.oidc.client.oauth2-redirections.authentication-entry-point` 修改为任意值。例如，对于单页或移动应用的 OAuth2 BFF，或有状态的 REST API，将其设置为 `UNAUTHORIZED` 可能更合适。

#### 1.2.4. <a name="1-2-4"/>RP-Initiated Logout
大多数 OpenID Provider 实现了 RP-Initiated Logout 并在 OpenID 配置中暴露了 `end_session_endpoint`，但部分 Provider（如 Auth0 和 Cognito）未严格遵循规范：它们有自己的登出机制，工作方式与 RP-Initiated Logout 类似，但 endpoint URI 和请求参数需查阅各自的文档。

以名为 `cognito-user` 和 `auth0-user` 的 registration 为例：
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        client:
          oauth2-logout:
            cognito-user:
              uri: https://spring-addons.auth.us-west-2.amazoncognito.com/logout
              client-id-request-param: client_id
              post-logout-uri-request-param: logout_uri
            auth0-user:
              uri: ${auth0-issuer}v2/logout
              client-id-request-param: client_id
              post-logout-uri-request-param: returnTo
```

#### 1.2.5. <a name="1-2-5"/>Back-Channel Logout
Back-Channel Logout 是一种 OpenID 标准，允许授权服务器将由其他 client 发起的登出事件通知给某个 client，即支持单点登录（SSO）系统的"单点登出"。Keycloak 是能够向 Spring client（如用作 OAuth2 BFF 的 Gateway）发送 Back-Channel Logout 的 OpenID Provider 示例。该功能默认禁用，可在配置属性中覆盖默认的内部登出 URI 和 session cookie 名称：
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        client:
          back-channel-logout:
            enabled: true
            # 以下两项为可选，大多数场景下默认值即可
            internal-logout-uri: ${gateway-uri}/logout/connect/back-channel/quiz-bff
            cookie-name: JSESSION-ID
```

#### 1.2.6. <a name="1-2-6"/>Authorities Converter
默认使用应用上下文中 authorities converter bean 的 `GrantedAuthoritiesMapper`。该 authorities converter 的默认实现与 resource server 共享：`ConfigurableClaimSetAuthoritiesConverter`，其配置由上下文中的 `OpenidProviderPropertiesResolver` 解析。

#### 1.2.7. <a name="1-2-7"/>CSRF 保护
OAuth2 client 的请求通过 session cookie 授权，这使其暴露于 CSRF 攻击。因此，**OAuth2 client 应始终启用 CSRF 保护**。

`spring-addons-starter-oidc` 的默认行为与 `spring-boot-starter-oauth2-client` 相同（使用 session）。

当设置 `com.c4-soft.springaddons.oidc.client.csrf=cookie-accessible-from-js` 时（单页应用和移动应用所需），CSRF token 会以 cookie 形式暴露，并注册所需的过滤器。

应用配置可以通过暴露 `Cookie[Server]CsrfTokenRepositoryPostProcessor` 来完全控制 CSRF cookie 和 repository。以下为 servlet 应用示例：
```java
@Bean
CookieCsrfTokenRepositoryPostProcessor csrfCookiePostProcessor() {
  return csrfCookieRepo -> {
    csrfCookieRepo.setCookieCustomizer(csrfCookie -> csrfCookie.sameSite("Strict"));
    return csrfCookieRepo;
  };
}
```

#### 1.2.8. <a name="1-2-8"/>访问控制
默认访问规则设置为 `isAuthenticated()`，但有两个例外：
- 路径匹配 `permit-all` 属性中条目的路由，允许匿名访问
- 预检请求（`OPTIONS` 请求且路径匹配 CORS 属性中的路径匹配器），除非在 `cors` 属性中禁用

定义细粒度访问控制最便捷的方式，可能是启用 `@Enable(Reactive)MethodSecurity` 并在 `@RestController` 方法上使用 `@PreAuthorize`。

对于偏好在配置中定义访问控制的场景（或不是自己编写 endpoint 的场景），可以暴露类型为 `ClientExpressionInterceptUrlRegistryPostProcessor` 或 `ClientAuthorizeExchangeSpecPostProcessor` 的 `@Bean`。

#### 1.2.9. <a name="1-2-9"/>后处理 Client Filter Chain
通过暴露 `Client(Server)HttpSecurityPostProcessor` bean，可以在 `clientSecurityFilterChain` 构建前完全控制其 `(Server)HttpSecurity` 的配置，从而修改任何预配置内容。

#### 1.2.10. <a name="1-2-10"/>向 Token 请求添加参数
某些 OpenID Provider 要求在 token endpoint 上传递额外参数。例如 Auth0 在 client-credentials token 请求中要求 `audience` 参数。这类参数可在 `com.c4-soft.springaddons.oidc.client.token-request-params.{registrationId}` 下定义（见 FAQ 示例）。

### 1.2.11. <a name="1-2-11"/>登录/登出后的跳转 URI
登录后的跳转 URI 可由前端通过 `X-POST-LOGIN-SUCCESS-URI` 和 `X-POST-LOGIN-FAILURE-URI` 请求头，或在发起 authorization code 流程的请求中通过 `post_login_success_uri` 和 `post_login_failure_uri` 参数提供。也可以通过 `post-login-redirect-host` 和 `post-login-redirect-path` 应用配置属性设置默认值。

类似地，登出后的跳转 URI 可由前端通过 `X-POST-LOGOUT-SUCCESS-URI` 请求头或发起 RP-Initiated Logout 请求中的 `post_logout_success_uri` 参数提供。该 URI 将作为请求参数添加到 OAuth2 client 关闭 session 后重定向至授权服务器的 Location 响应头中。也可以通过 `post-logout-redirect-host` 和 `post-logout-redirect-path` 应用配置属性设置默认值。

从 `8.1.13` 起，为防止[开放重定向（CWE-601）](https://cwe.mitre.org/data/definitions/601.html)攻击，登录/登出后的跳转 URI 必须在 `post-login-allowed-uri-patterns` 和 `post-logout-allowed-uri-patterns` 属性定义的白名单中。为保持向后兼容，若这些属性留空，则允许所有仅包含路径（无 scheme/authority）的跳转 URI，以及与 `client-uri` 属性具有相同 scheme 和 authority 的跳转 URI。

比默认值更严格的示例配置：
```yaml
reverse-proxy: https://localhost

com:
  c4-soft:
    springaddons:
      oidc:
        client:
          client-uri: ${reverse-proxy}/bff
          post-login-allowed-uri-patterns:
          # 每个条目编译为 java.util.regex.Pattern
          # 以下配置比默认值（^${reverse-proxy}(/.*)?$ 和 ^/.*$）更严格
          - ^${reverse-proxy}/ui(/.*)?$
          - ^/ui(/.*)?$
          post-login-redirect-path: /ui/greet
          post-logout-allowed-uri-patterns:
          - ^${reverse-proxy}/ui(/.*)?$
          - ^/ui(/.*)?$
          post-logout-redirect-path: /ui/
```
`post-login-allowed-uri-patterns`/`post-logout-allowed-uri-patterns` 中的每个条目都会被编译为 `java.util.regex.Pattern`。

`post-login-redirect-host`、`post-login-redirect-path`、`post-logout-redirect-host` 和 `post-logout-redirect-path` 在启动时进行验证，若不匹配则抛出异常。

若登录/登出后的请求头和请求参数中的跳转 URI 不在白名单中，则认证请求将返回 `401`。

## 2. <a name="faq"/>常见问题

### <a name="2-1"/>自动配置的具体内容是什么？
要全面了解加载的内容，请从 `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 入手，这是标准的 Spring Boot 资源文件，列出了 Spring Boot 在构建应用上下文时会使用的 `@AutoConfiguration` 文件。

你可以检查每个 `@AutoConfiguration`（及导入的 `@Configuration`）上的条件实现，将调查范围限制在实际加载到你的应用中的内容。

### <a name="2-2"/>`spring-addons-starter-oidc` 定义的所有 bean 都会被添加到应用上下文中吗？
**不会！** 所有 bean 都是有条件的，只有少数会被实例化。具体哪些 bean 会被实例化取决于应用类型（servlet 或响应式）、依赖（resource server、client 或两者）、配置属性，以及显式的 bean 定义（大多数 `spring-addons-starter-oidc` bean 都标注了 `@ConditionalOnMissingBean`）。

### <a name="2-3"/>为什么要大费周章地处理 authorities 映射？直接用默认的 scope 不行吗？
简单来说，部分 OpenID Provider 不会将角色放在 scope 中。

原因在于两者并非同一概念：在 RBAC 中，*角色*是用户的属性；而 OAuth2 scope 是客户端的属性。scope 定义了资源所有者允许 OAuth2 client 代表其在 audience 中指定的 resource server 上执行哪些操作，可以将 scope 理解为覆盖在用户角色之上的一层掩码。

scope 主要在你希望让用户控制哪些软件可以访问其哪些资源时有意义。

角色则主要在你希望控制哪些用户可以访问哪些资源、而不论其使用何种软件时有意义。

### <a name="2-4"/>如何配置 resource server 以接受运行时动态创建的 Keycloak *realm* 签发的 token？
参见 resource server 功能章节中的多租户部分。

### <a name="2-5"/>如何为 Auth0 用户登录提供所需的 `audience` 参数？
将其添加到 `com.c4-soft.springaddons.oidc.client.authorization-request-params.{registrationId}`。例如，假设 Boot 属性中有一个名为 `auth0-users` 的 registration：
```yaml
issuer: https://dev-ch4mpy.eu.auth0.com/

spring:
  security:
    oauth2:
      client:
        provider:
          auth0:
            issuer-uri: ${issuer}
        registration:
          auth0-user:
            provider: auth0
            client-id: change-me
            client-secret: change-me
            authorization-grant-type: authorization_code
            scope: openid, offline_access
com:
  c4-soft:
    springaddons:
      oidc:
        client:
          authorization-request-params:
            auth0-user:
              audience: demo.c4-soft.com
```
注意 `auth0-user` 这个 registration ID 同时出现在 Spring Boot 的 `registration` 和 spring-addons 的 `authorization-request-params` 中。

### <a name="2-6"/>如何为 Auth0 client-credentials 提供所需的 `audience` 参数？
将其添加到 `com.c4-soft.springaddons.oidc.client.token-request-params.{registrationId}`。例如，假设 Boot 属性中有一个名为 `auth0-api` 的 registration：
```yaml
issuer: https://dev-ch4mpy.eu.auth0.com/

spring:
  security:
    oauth2:
      client:
        provider:
          auth0:
            issuer-uri: ${issuer}
        registration:
          auth0-api:
            provider: auth0
            client-id: change-me
            client-secret: change-me
            authorization-grant-type: client_credentials
            scope: read:users
com:
  c4-soft:
    springaddons:
      oidc:
        client:
          token-request-params:
            auth0-api:
              audience: demo.c4-soft.com
```
注意 `auth0-api` 这个 registration ID 同时出现在 Spring Boot 的 `registration` 和 spring-addons 的 `token-request-params` 中。

### <a name="2-9"/>前端可以覆盖 OAuth2 重定向的响应状态码吗？
可以。`SpringAddonsOauth2(Server)RedirectStrategy` 是 OAuth2 重定向的默认实现，它会查找 `X-RESPONSE-STATUS` 请求头或 `response_http_status` 请求参数，若存在则用其覆盖应用配置属性中设置的默认值。

### <a name="2-10"/>`iss` 配置属性是必填的吗？
不是必填的。但请注意，不设置时 `iss`（issuer）claim 验证将被禁用，JWT decoder 只会校验 token 签名。由于在安全性达标的应用中 issuer 和 audience 应始终被验证，**以下绕过方法不推荐使用**。

在使用 JWT decoder 的 Spring `oauth2ResourceServer` 中，至少需要提供：
- `iss` 或 `jwk-set-uri` 之一，以便为 JWT decoder 提供 JWK-set（JWK-set URI 通常可从 `{iss}/.well-known/openid-configuration` 的 OpenID 配置中获取，因此当提供了 `iss` 属性且其值在网络上可访问时，`jwk-set-uri` 属性是可选的）
- 一个 `OpenidProviderPropertiesResolver`，用于从 token claim 中检索配置（及关联的 JWT decoder）。默认实现是将 `iss` claim 与 `iss` 属性进行匹配；若未提供 `iss` 属性（或其值与 token 中的值不完全匹配，如"动态"多租户场景），则需要暴露自定义的 `OpenidProviderPropertiesResolver` bean

注意，若配置中未提供 issuer，JWT decoder 将不配置 issuer 验证器。除非提供了 audience，否则唯一的验证器是签名验证器（使用 JWK-set）。示例：
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - jwk-set-uri: https://oidc.c4-soft.com/auth/realms/quiz/protocol/openid-connect/certs
```
```java
@Component
public class FirstOpenidProviderPropertiesResolver implements OpenidProviderPropertiesResolver {
    private final Optional<OpenidProviderProperties> opProperties;
    
    public FirstOpenidProviderPropertiesResolver(SpringAddonsOidcProperties properties) {
        this.opProperties = properties.getOps().isEmpty() ? Optional.empty() : Optional.of(properties.getOps().get(0));
    }

    @Override
    public Optional<OpenidProviderProperties> resolve(Map<String, Object> claimSet) {
        return opProperties;
    }
}
```
这看起来很笨，但在单租户场景下是可行的：第一个（也是唯一一个）OpenID Provider 属性组始终匹配，只有我们信任的 issuer 签名的 token 才被认为有效。

在使用 `oauth2Login` 的 Spring OAuth2 client 中，OpenID 自动配置依赖于将 *Issuer Identifier* 设置为 `spring.security.oauth2.client.provider.{provider-id}.issuer-uri`。若由于某种原因授权服务器无法通过 *Issuer Identifier* 访问（容器化环境配置错误）或不符合 OpenID 规范（Microsoft V1 token），则应将 `issuer-uri` 留空，并手动提供 `authorization`、`token`、`jwk-set` 和 `userinfo` endpoint 的 URI，而非依赖 OpenID 自动配置。

### <a name="2-11"/>为什么 Microsoft 授权服务器很难配置？
Microsoft 授权服务器（如 Entra ID，前身为 Azure Active Directory / AAD，或 Azure AD B2C 等）看起来像 OIDC Provider，但在默认的 V1 token 格式下并非如此。

[OIDC 发现规范](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig)规定：*"支持 Discovery 的 OpenID Provider 必须在 Issuer 拼接 `/.well-known/openid-configuration` 路径后的地址上提供一个 JSON 文档"*。同时，[OpenID token 验证规范](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation)要求：*"OpenID Provider 的 Issuer Identifier（通常在 Discovery 期间获取）必须与 iss（issuer）Claim 的值完全匹配"*。

要让 Microsoft Entra ID 遵循这些 OIDC 发现和 OpenID token 规范要求并签发 JWT access token，需要：
- 在 Entra 管理控制台的 `Applications` -> `App registrations` -> `{appName}` -> `Manifest` -> `Microsoft Graph App Manifest (New)` 中设置 `api.requestedAccessTokenVersion: 2`
- 确保混合流（hybrid flow）完全禁用
- 声明一个 audience（Entra 管理控制台中的 *"API"*），并在授权请求中将其作为 scope 请求（授权请求应包含 `scope=openid {apiId}`）

……直觉性极强的配置方式。

### <a name="2-12"/>如何添加其他请求授权机制（如 `Basic` 认证）？
正如 [OAuth2 Essentials](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials#1-oauth2-essentials) 中所述，每种请求授权机制应独立在各自的 security filter chain 中。

当 Spring 应用配置了多个 security filter chain 时，这些 filter chain 应满足：
- 有序：bean 定义应以不同的 `@Order` 值装饰
- 除最后一个 filter chain 外，所有 filter chain 都应包含 `securityMatcher`，精确定义其应拦截的请求（最后一个应设计为默认处理器，拦截所有未被更高优先级 filter chain 拦截的请求）。例如，用于 `Basic` 认证的 filter chain 应包含如下 `securityMatcher`：
```java
http.securityMatcher((HttpServletRequest request) -> {
  return Optional.ofNullable(request.getHeader(HttpHeaders.AUTHORIZATION)).map(h -> {
    return h.toLowerCase().startsWith("basic ");
  }).orElse(false);
});
```
由于 `spring-addons-starter-oidc` 自动配置的 security filter chain 优先级非常低（`oauth2ResourceServer` 为 `LOWEST_PRECEDENCE`，`oauth2Login` 为 `LOWEST_PRECEDENCE - 1`），**只需定义一个带有 `@Order(Ordered.LOWEST_PRECEDENCE - 2)`（或更高）和 `securityMatcher` 的 security filter chain bean，即可添加对 `oauth2ResourceServer` 和 `oauth2Login` 之外任意请求授权机制的支持**。

### <a name="2-13"/>如何配置 authorized client manager 通过 HTTP 代理？
`(Reactive)OAuth2AuthorizedClientManager` 使用 `(Reactive)OAuth2AuthorizedClientProvider`，后者内部使用 `RestClient` 或 `WebClient`。可以在 `WebClient.Builder::clientConnector` 和 `RestClient.Builder::requestFactory` 上设置代理属性。因此，要让 token 请求通过代理，需要手动配置这些 bean。

以下面的场景为例：
- `spring.security.oauth2.client.provider.external` provider 定义，OAuth2 client 只能通过代理访问
- `spring.security.oauth2.client.registration.m2m` 引用该 `external` provider

`spring-addons-starter-rest` 大大简化了带代理属性的 `RestClient.Builder` 配置，用于访问 `external` provider：
```yaml
com:
  c4-soft:
    springaddons:
      rest:
        client:
          token-client:
            expose-builder: true
            # 以下属性仅在 HTTP_PROXY 和 NO_PROXY 环境变量不够用时，或需要设置代理认证时才需要
#            http:
#              proxy:
#                host: https://proxy.corporate.com:8443
```
现在可以进一步配置此 `RestClient.Builder`，并配置一个 authorized client provider，在处理 `m2m` registration 时使用生成的 `RestClient`：
```java
// 暴露一个 RestClient bean，包含：
// - spring-addons-starter-rest 的代理自动配置
// - 所有 OAuth2 token endpoint 所需的 message converter
@Bean
RestClient tokenClient(RestClient.Builder tokenClientBuilder) {
  return tokenClientBuilder.messageConverters((messageConverters) -> {
    messageConverters.clear();
    messageConverters.add(new FormHttpMessageConverter());
    messageConverters.add(new OAuth2AccessTokenResponseHttpMessageConverter());
  }).defaultStatusHandler(new OAuth2ErrorResponseErrorHandler()).build();
}

// 用自定义的 RestClient 替换 spring-addons-starter-oidc 默认的 OAuth2AuthorizedClientProvider bean，
// 在使用 "m2m" registration 时生效
@Bean
OAuth2AuthorizedClientProvider oauth2AuthorizedClientProvider(
    SpringAddonsOidcProperties addonsProperties,
    InMemoryClientRegistrationRepository clientRegistrationRepository,
    RestClient tokenClient) {
  return new PerRegistrationOAuth2AuthorizedClientProvider(clientRegistrationRepository,
      addonsProperties, Map.of("m2m", tokenClient));
}
```
`spring-addons-starter-oidc` 会检测到此 authorized client provider 并使用它，而非自动配置一个。

### 2.14. <a name="2-14"/>关于 cookie 的注意事项？
使用 `oauth2Login` 时，请求授权基于 session cookie，需要防范 CSRF 攻击。对于单页应用（Angular、React、Vue 等）和移动端前端，后端应将 CSRF token 以 `HttpOnly=false` 的 cookie 形式暴露，并将该 cookie 用作 CSRF token repository。

session cookie 和 CSRF cookie 都应设置 `SameSite`（`Lax` 即可）和 `Secure`（配置了 SSL 的反向代理即可满足；在开发机上证书可以是自签名的）。session cookie 应始终设置 `HttpOnly=true`。

当多个应用托管在同一主机上时，默认 cookie 可能发生冲突，除非为每个应用设置不同的 cookie 名称或路径。session cookie 和 CSRF cookie 的名称没有特别限制，但需要注意浏览器对路径的安全行为：
- 单页应用（Angular、React、Vue 等）的 JavaScript 只有在 cookie 设置了 `HttpOnly=false` 且 cookie 路径是 SPA 静态资源路径的前缀时，才能读取该 cookie。若无法读取，前端将无法读取 token 值并在 `POST`、`PUT`、`PATCH`、`DELETE` 请求中设置 `X-XSRF-TOKEN` 请求头。
- cookie 只有在其路径是请求路径的前缀时，才会附加到前端请求中。若 cookie 缺失，后端会生成一个带新 CSRF token 值的新 cookie，验证将失败（提醒：CSRF token repository 是该 cookie，而非 session）。

因此，CSRF cookie 的路径只能使用 UI 静态资源路径和 REST 后端路径的公共前缀（例如 `/foo/ui/**` 和 `/foo/bff/v1/**` 的公共前缀为 `/foo`）。由于 session cookie 永远不会被前端 JavaScript 读取，将后端路径的前缀作为 session cookie 路径是没问题的（在上述例子中，`/foo/bff/v1` 的任意起始片段都可以）。

在 Spring Boot 中，session cookie 可通过标准属性配置。以 servlet 应用为例：
```yaml
server:
  servlet:
    session:
      cookie:
        name: X-FOO-SESSION
```

使用 `spring-addons-starter-oidc` 时，若 `csrf` 属性设置为 `cookie-accessible-from-js`，可配置 CSRF cookie 路径和名称：
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        client:
          csrf: cookie-accessible-from-js
          csrf-cookie-name: X-FOO-CSRF
          csrf-cookie-path: /
```

## 3. <a name="usage"/>基本使用方法

本节仅介绍最基本的使用方法。关于高级自动配置和默认值覆盖，请参阅 [1. 功能特性](https://github.com/fivetime/spring-addons/tree/master/spring-addons-starter-oidc#spring-addons-starter-oidc)。

**如果你不确定自己需要的是带 `oauth2Login` 的 OAuth2 client（基于 session 而非 access token 的安全机制）还是 OAuth2 resource server 配置（基于 access token 而非 session、不带 `oauth2Login` 的安全机制），请阅读教程中的 [OAuth2 essentials 章节](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials#1-oauth2-essentials)，这将为你节省大量时间和精力。**

在依赖中添加 `com.c4-soft.springaddons:spring-addons-starter-oidc`，以及 `org.springframework.boot:spring-boot-starter-oauth2-client` 或 `org.springframework.boot:spring-boot-starter-oauth2-resource-server`。

若配置 OAuth2 client（带 `oauth2Login`），需定义标准的 Spring Boot `provider` 和 `registration` 属性。

若配置使用 access token introspection 的 OAuth2 resource server，需定义标准的 Spring Boot `opaquetoken` 属性。

然后，根据你的使用场景定义相关的 `com.c4-soft.springaddons.oidc` 属性。可参考大量完整的 [samples](https://github.com/fivetime/spring-addons/tree/master/samples) 和 [tutorials](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials)，以下是针对不同使用场景和 OpenID Provider 的几个示例：

### 3.1. <a name="3-1"/>使用 JWT decoder 的 Resource Server

对于使用 JWT access token 保护的 REST API，需要：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <!-- 响应式应用请改用 spring-boot-starter-webflux -->
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>

<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc</artifactId>
</dependency>
```
以及：
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: https://oidc.c4-soft.com/auth/realms/master
          username-claim: preferred_username
          authorities:
          - path: $.realm_access.roles
          - path: $.resource_access.*.roles
        resourceserver:
          permit-all:
          - "/greet/public"
          cors:
          - path: /**
            allowed-origin-patterns: http://localhost:4200
```
上述配置将创建一个无 session 和 CSRF 保护的应用，对受保护资源的未授权请求将返回 401。

### <a name="3-2"/>3.2. Client

对于提供带登录和登出功能的 Thymeleaf 模板的应用：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <!-- 响应式应用请改用 spring-boot-starter-webflux -->
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>

<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc</artifactId>
</dependency>
```
以及：
```yaml
cognito-issuer: https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl
cognito-client-id: change-me
cognito-secret: change-me

spring:
  security:
    oauth2:
      client:
        provider:
          cognito:
            issuer-uri: ${cognito-issuer}
        registration:
          cognito-authorization-code:
            authorization-grant-type: authorization_code
            client-id: ${cognito-client-id}
            client-secret: ${cognito-secret}
            provider: cognito
            scope: openid,profile,email,offline_access
com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: ${cognito-issuer}
          username-claim: username
          authorities:
          - path: cognito:groups
        client:
          security-matchers:
          - /**
          permit-all:
          - /login/**
          - /oauth2/**
          - /
          # Auth0 和 Cognito 未严格遵循 OpenID RP-Initiated Logout 规范，需要特殊配置
          oauth2-logout:
            cognito-authorization-code:
              uri: https://spring-addons.auth.us-west-2.amazoncognito.com/logout
              client-id-request-param: client_id
              post-logout-uri-request-param: logout_uri
```
上述配置将创建一个基于 session（而非 access token）的安全应用，启用 CSRF 保护，对受保护资源的未授权请求将重定向至登录页。

### <a name="3-3"/>3.3. Client 与 Resource Server 并存

对于同时公开以下两种服务的应用：
- 基于 session 的 Thymeleaf 模板（带登录和登出），所有模板以 `/ui` 为前缀（首页除外，在 `/`）
- 基于 access token 的 REST API
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <!-- 响应式应用仅使用 spring-boot-starter-webflux -->
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <!-- 用于在提供 Thymeleaf 模板的 controller 中通过 WebClient 调用 REST API -->
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>

<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc</artifactId>
</dependency>
```
以及：
```yaml
auth0-issuer: https://oidc.c4-soft.com/auth/realms/master
auth0-client-id: change-me
auth0-secret: change-me

spring:
  security:
    oauth2:
      client:
        provider:
          auth0:
            issuer-uri: ${auth0-issuer}
        registration:
          auth0-authorization-code:
            authorization-grant-type: authorization_code
            client-id: ${auth0-client-id}
            client-secret: ${auth0-secret}
            provider: auth0
            scope: openid,profile,email,offline_access
com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: ${auth0-issuer}
          username-claim: $['https://c4-soft.com/user']['name']
          authorities:
          - path: $['https://c4-soft.com/user']['roles']
          - path: $.permissions
        client:
          security-matchers:
          - /login/**
          - /oauth2/**
          - /logout
          - /
          - /ui/**
          permit-all:
          - /login/**
          - /oauth2/**
          - /
          # Auth0 和 Cognito 未严格遵循 OpenID RP-Initiated Logout 规范，需要特殊配置
          oauth2-logout:
            auth0-authorization-code:
              uri: ${auth0-issuer}v2/logout
              client-id-request-param: client_id
              post-logout-uri-request-param: returnTo
          # Auth0 要求在 authorization-code 请求中携带 "audience" 参数以签发 JWT
          authorization-request-params:
            auth0-authorization-code:
            - name: audience
              value: demo.c4-soft.com
        resourceserver:
          permit-all:
          - "/greet/public"
```
上述配置将定义两个独立的 security filter chain：
- 一个 client filter chain，基于 session（启用 CSRF 保护），拦截所有 UI 模板请求以及登录和登出相关请求，将对受保护模板的未授权请求重定向至登录页。
- 一个 resource server filter chain，作为默认处理器（优先级最低，处理所有未被 client filter chain 的 `securityMatchers` 匹配的请求），无 session 和 CSRF 保护（请求通过 JWT access token 授权），对受保护资源的未授权请求返回 401。