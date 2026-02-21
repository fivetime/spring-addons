# 如何使用 `OAuthentication<OpenidClaimSet>` 配置 Spring REST API
本教程的目标是为一个带有 JWT 解码功能的 Spring Boot 3 resource server 配置安全机制，并使用自定义 `Authentication` 实现替代默认的 `JwtAuthenticationToken`。

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 前置条件
假设已完成[教程主 README 的前置条件章节](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials#prerequisites)，并已准备好至少 1 个 OIDC Provider（2 个更好），且已配置带有 authorization-code 流程的 client ID 和 secret。

另外，本教程将使用 `spring-addons-starter-oidc`。如果出于某种原因不想使用，可以参考 [`servlet-resource-server` 教程](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials/servlet-resource-server)，仅使用 `spring-boot-starter-oauth2-resource-server` 将 REST API 配置为 OAuth2 resource server。

## 2. 项目初始化
借助 https://start.spring.io/ 创建一个 Spring Boot 3 项目，需要以下依赖：
- Spring Web
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

## 3. Web Security 配置
`spring-oauth2-addons` 提供了适用于 REST API 项目的 web security 配置 `@AutoConfiguration`。我们只需要：
- 添加 `@EnableMethodSecurity` 以在组件方法上启用 `@PreAuthorize`
- 提供一个 `JwtAbstractAuthenticationTokenConverter` bean，将 `Authentication` 实现从 `JwtAuthenticationToken` 切换为 `OAuthentication<OpenidClaimSet>`
```java
@Bean
JwtAbstractAuthenticationTokenConverter authenticationConverter(
    Converter<Map<String, Object>, Collection<? extends GrantedAuthority>> authoritiesConverter,
    OpenidProviderPropertiesResolver opPropertiesResolver) {
  return jwt -> {
    final var opProperties = opPropertiesResolver.resolve(jwt.getClaims())
        .orElseThrow(() -> new NotAConfiguredOpenidProviderException(jwt.getClaims()));
    final var accessToken =
        new OpenidToken(new OpenidClaimSet(jwt.getClaims(), opProperties.getUsernameClaim()),
            jwt.getTokenValue());
    final var authorities = authoritiesConverter.convert(jwt.getClaims());
    return new OAuthentication<>(accessToken, authorities);
  };
}
```
这里，我们保留了 `spring-addons` 默认的 authorities converter 来从 token claim 中提取 Spring authorities。该 converter 需要由 `OpenidProviderPropertiesResolver` 解析的配置属性（`spring-addons` 的默认实现通过将 token 的 `iss` claim 与 YAML 中的 `iss` 属性匹配来解析属性）。

## 4. 应用配置属性
大部分安全配置由属性控制。关于此处设置的属性的详细说明，请参阅 [spring-addons starter 入门教程](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials/servlet-resource-server)：
```yaml
scheme: http
origins: ${scheme}://localhost:4200
keycloak-port: 8442
keycloak-issuer: ${scheme}://localhost:${keycloak-port}/realms/master
keycloak-secret: change-me
cognito-issuer: https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl
auth0-issuer: https://dev-ch4mpy.eu.auth0.com/

server:
  error:
    include-message: always
  ssl:
    enabled: false

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

com:
  c4-soft:
    springaddons:
      oidc:
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
          - "/actuator/health/readiness"
          - "/actuator/health/liveness"
          - "/v3/api-docs/**"
          - "/swagger-ui/**"

---
scheme: https
keycloak-port: 8443

server:
  ssl:
    enabled: true

spring:
  config:
    activate:
      on-profile: ssl
```

## 5. 示例 `@RestController`
请注意，OpenID 标准 claim 是有类型的，可以通过 getter 访问（而不像 `JwtAuthenticationToken` 那样使用 `Map<String, Object>`）：
``` java
@RestController
@PreAuthorize("isAuthenticated()")
public class GreetingController {

    @GetMapping("/greet")
    public MessageDto getGreeting(OAuthentication<OpenidClaimSet> auth) {
        return new MessageDto("Hi %s! You are granted with: %s and your email is %s."
                .formatted(auth.getName(), auth.getAuthorities(), auth.getClaims().getEmail()));
    }

    @GetMapping("/nice")
    @PreAuthorize("hasAuthority('NICE')")
    public MessageDto getNiceGreeting(OAuthentication<OpenidClaimSet> auth) {
        return new MessageDto("Dear %s! You are granted with: %s."
                .formatted(auth.getName(), auth.getAuthorities()));
    }

    static record MessageDto(String body) {
    }
}
```

## 5. 单元测试
请参阅源码以及 [Baeldung 上的专题文章](https://www.baeldung.com/spring-oauth-testing-access-control)。

## 6. 总结
本示例引导你构建了一个带有 JWT decoder 和 `OAuthentication<OpenidClaimSet>` 的 servlet（webmvc）应用。如需配置 webflux（响应式）resource server、access token introspection 或其他类型认证的帮助，请参阅其他教程和 [samples](https://github.com/ch4mpy/spring-addons/tree/master/samples)。