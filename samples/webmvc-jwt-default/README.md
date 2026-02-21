# 使用 `spring-addons-starter-oidc` 的 Servlet Resource Server（JWT Decoder）
本示例使用对 `spring-boot-starter-oauth2-resource-server` 的轻量封装，几乎仅通过应用配置属性来配置一个 Spring Boot 3 servlet（WebMVC）resource server。

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 依赖配置
与往常一样，从 http://start.spring.io/ 开始，添加以下依赖：
- Spring Web
- OAuth2 Resource Server
- Spring Boot Actuator
- Lombok

然后添加 spring-addons 相关依赖：
- [`spring-addons-starter-oidc`](https://central.sonatype.com/artifact/com.c4-soft.springaddons/spring-addons-starter-oidc)
- [`spring-addons-starter-oidc-test`](https://central.sonatype.com/artifact/com.c4-soft.springaddons/spring-addons-starter-oidc-test)（`test` scope）
```xml
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc</artifactId>
    <version>${spring-addons.version}</version>
</dependency>
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc-test</artifactId>
    <version>${spring-addons.version}</version>
    <scope>test</scope>
</dependency>
```

## 2. 应用配置属性
如前言所述，大部分配置都在属性文件中。下面逐一解析 yaml 文件的内容。

第一部分定义了后续配置中复用的常量：
```yaml
scheme: http
origins: ${scheme}://localhost:4200
keycloak-port: 8442
keycloak-issuer: ${scheme}://localhost:${keycloak-port}/realms/master
cognito-issuer: https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl
auth0-issuer: https://dev-ch4mpy.eu.auth0.com/
```

接下来是 spring-addons 的核心配置，包括：
- 按路径匹配器配置 CORS（这里使用一个匹配器拦截 resource server filter chain 处理的所有端点）
- 3 个受信任的 OIDC Provider（issuer），每个配置：
  * `location`：issuer URI（必须与 access token `iss` claim 中的值完全一致），用于获取 OpenID 配置并为请求解析 authentication manager
  * `username-claim`：仅在需要使用 `sub` claim 以外的字段时配置
  * `authorities`：authority 映射配置，包括要使用的每个 claim（JSON path、大小写转换和前缀）
- 允许所有请求（包括匿名请求）访问的资源路径匹配器（未匹配的路径要求用户已认证）
- session 保持无状态（禁用），这是 spring-addons resource server 的默认行为
- CSRF 保护保持禁用，这是 spring-addons 在 session 无状态时的默认行为
- 同样保留 spring-addons resource server 的默认行为：当认证缺失或无效时返回 401（未授权），而非 302（重定向到登录页）
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        cors:
        - path: /**
          allowed-origin-patterns: ${origins}
        ops:
        - iss: ${keycloak-issuer} 
          username-claim: preferred_username
          authorities:
          - path: $.realm_access.roles
          - path: $.resource_access.*.roles
        - iss: ${cognito-issuer}
          username-claim: username
          authorities:
          - path: cognito:groups
        - iss: ${auth0-issuer}
          username-claim: $['https://c4-soft.com/user']['name']
          authorities:
          - path: $['https://c4-soft.com/user']['roles']
          - path: $.permissions
        resourceserver:
          permit-all:
          - "/greet/public"
          - "/actuator/health/readiness"
          - "/actuator/health/liveness"
          - "/v3/api-docs/**"
```

## 3. Java 配置
`spring-addons-webmvc-jwt-resource-server` 提供了自动配置的 `SecurityFilterChain`。

下面定义的 `AuthorizeExchangeSpecPostProcessor` 主要用于演示：展示如何在 Java 配置中使用 `spring-addons-starter-oidc` 编写细粒度的访问控制规则。它可以用 `GreetingController::securedRoute` 上的 `@PreAuthorize("hasRole('AUTHORIZED_PERSONNEL')")` 来替代，从而使（显式的）security 配置为空。
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
  @Bean
  ExpressionInterceptUrlRegistryPostProcessor expressionInterceptUrlRegistryPostProcessor() {
    return (AuthorizeHttpRequestsConfigurer<HttpSecurity>.AuthorizationManagerRequestMatcherRegistry registry) -> registry
        .requestMatchers("/secured-route").hasRole("AUTHORIZED_PERSONNEL")
        .anyRequest().authenticated();
  }
}
```

## 4. `@RestController`、`@Service` 和 `@Repository`
这部分没有什么特别之处，只是带有方法级安全的标准 Spring 组件。如果你把此 README 作为复现示例的教程，请从源码中复制代码。

## 5. 测试
源码包含针对所有访问控制规则的单元测试和集成测试，涵盖 `@Controller`，也包括 `@Service` 和 `@Repository`（后两者在仅使用 `spring-security-test` 的 OAuth2 场景下无法测试）。请务必仔细查阅。

## 6. 总结
在本示例中，我们在 `spring-boot-starter-oauth2-resource-server` 的基础上使用 `spring-addons-starter-oidc`，几乎仅通过应用配置属性就配置好了一个 Spring Boot 3 servlet（WebMVC）resource server，实现了：
- 无状态 session 管理
- 禁用 CSRF（因为 session 已禁用）
- 细粒度的 CORS 配置（部署到新环境时可轻松更改允许的来源）
- 多租户（接受来自多个受信任 OIDC Provider 的身份）
- 未授权请求的预期 HTTP 状态码
- 可通过方法级安全进一步精调的基础访问控制

是不是很 Bootiful？