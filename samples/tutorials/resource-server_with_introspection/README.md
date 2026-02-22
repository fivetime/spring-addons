# 如何配置带有 Token Introspection 的 Spring REST API

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 概述
本教程的目标是为一个 Spring Boot 3 resource server 配置安全机制，使其能对**任意 OpenID 授权服务器**的 access token 进行 introspection：既包括在 OpenID 配置中暴露了 introspection endpoint 的 Provider（如 Keycloak），也包括仅暴露了 `/userinfo` endpoint 的 Provider（如 Auth0 和 Amazon Cognito）。

对于每个处理的请求，resource server 都会向授权服务器发送一个请求以获取 token 详情。与基于 JWT decoder 的安全机制（只需访问授权服务器一次以获取签名公钥）相比，这会**对性能产生明显影响**。

## 2. 授权服务器要求
假设已满足[教程前置条件](https://github.com/fivetime/spring-addons/blob/master/samples/tutorials/README.md#prerequisites)，并已配置至少 1 个 OIDC Provider，包含用于用户认证的 client 和 authorization-code。由于很难判断一个不透明 token 来自哪个 OP，我们只接受来自单个 issuer 的身份。若要同时支持多租户和 token introspection，则需要通过自定义请求头或其他方式携带 issuer URI，以便 resource server 知道应向哪里进行 introspection。这种额外复杂度超出了本教程的范围，因此我们改用 profile 在不同 OP 之间切换。

Introspection endpoint 通过 client-credentials 流程访问。对于符合 introspection 规范的每个 OP，应配置一个支持该流程的 client（可以是与用户认证相同的 client，也可以是另一个）。

对于 Keycloak，这意味着需要配置一个满足以下条件的 client：
- "Access Type" 为 `confidential`
- 启用 "Service Accounts Enabled"

如果还没有，请创建一个。保存配置后，可以在 "credentials" 标签页中获取 client-secret。

由于 Auth0 和 Amazon Cognito 不在其 OpenID 配置中暴露 `introspection_endpoint`，因此不需要配置支持 client-credentials 流程的 client：我们将使用待 introspect 的 access token 作为请求的 access token，直接查询 `/userinfo` endpoint。

## 3. 项目初始化
借助 https://start.spring.io/ 创建一个 Spring Boot 3 项目，需要以下依赖：
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

## 5. 应用配置属性
首先定义一些后续配置中使用的常量，部分常量在 profile 中会被覆盖：
```yaml
scheme: http
origins: ${scheme}://localhost:4200
keycloak-port: 8442
keycloak-issuer: ${scheme}://localhost:${keycloak-port}/realms/master
keycloak-secret: change-me
cognito-issuer: https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl
cognito-secret: change-me
auth0-issuer: https://dev-ch4mpy.eu.auth0.com/
auth0-secret: change-me
```
然后是服务器和带有 token introspection 的 OAuth2 resource server 的标准 Spring Boot 配置：
```yaml
server:
  error:
    include-message: always
  ssl:
    enabled: false
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          client-id: spring-addons-confidential
          client-secret: ${keycloak-secret}
          introspection-uri: ${keycloak-issuer}/protocol/openid-connect/token/introspect
```
接下来是 spring-addons 配置，包含：
- CORS 配置（例如可在部署到新环境时切换 allowed-origins）
- `issuers`：提供 authorities 映射配置（选取的 claim、大小写转换和前缀）
- `permit-all`："公开"资源的路径匹配器（允许未授权请求访问），未匹配的路径要求请求已授权（通过方法级安全进一步精调访问控制）
```yaml
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
        resourceserver:
          cors:
          - path: /**
            allowed-origin-patterns: ${origins}
          permit-all: 
          - "/actuator/health/readiness"
          - "/actuator/health/liveness"
          - "/v3/api-docs/**"
```
最后是在本服务器及与本地 Keycloak 实例通信时启用 SSL 的 profile：
```yaml
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

## 4. Web Security 配置
`spring-addons-starter-oidc` 会自动配置带有 token introspection 的 resource server security filter chain，我们只需要启用方法级安全即可，无需做其他任何配置。

## 6. 非标准 Introspection Endpoint
上述 token introspection 配置对在 OpenID 配置中暴露了 `introspection_endpoint` 的 OIDC Provider（如 Keycloak）运行正常，但部分 Provider 并不提供此 endpoint（如 Auth0 和 Amazon Cognito）。好在几乎所有 OP 都暴露了 `/userinfo` endpoint，对于 `Authorization` 请求头中携带的 access token 所对应的用户，该 endpoint 会返回其 OpenID claim。

### 6.1. 额外的应用配置属性
首先为没有 `introspection_endpoint` 的 OP 定义 Spring profile，将 `introspection-uri` 设置为 userinfo URI：
```yaml
---
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
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          client-id: TyY0H7xkRMRe6lDf9F8EiNqCo8PdhICy
          client-secret: ${auth0-secret}
          introspection-uri: ${auth0-issuer}userinfo
  config:
    activate:
      on-profile: auth0
---
com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: ${cognito-issuer}
          username-claim: $.username
          authorities:
          - path: $.cognito:groups
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          client-id: 12olioff63qklfe9nio746es9f
          client-secret: ${cognito-secret}
          introspection-uri: https://spring-addons.auth.us-west-2.amazoncognito.com/oauth2/userInfo
  config:
    activate:
      on-profile: cognito
```

### 6.2. 自定义 `OpaqueTokenIntrospector`
现在，我们可以编写自定义的 `OpaqueTokenIntrospector`，通过在 `Authorization` 请求头中携带待 introspect 的 token 来查询 `/userinfo`。如果收到响应，则说明 token 有效，返回的 claim 即为该 token 签发对象的 OpenID claim：
```java
@Component
@Profile("auth0 | cognito")
public static class UserEndpointOpaqueTokenIntrospector implements OpaqueTokenIntrospector {
    private final URI userinfoUri;
    private final RestTemplate restClient = new RestTemplate();

    public UserEndpointOpaqueTokenIntrospector(OAuth2ResourceServerProperties oauth2Properties)
            throws IOException {
        userinfoUri = URI.create(oauth2Properties.getOpaquetoken().getIntrospectionUri());
    }

    @Override
    @SuppressWarnings("unchecked")
    public OAuth2AuthenticatedPrincipal introspect(String token) {
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);
        final var claims = new OpenidClaimSet(restClient
                .exchange(userinfoUri, HttpMethod.GET, new HttpEntity<>(headers), Map.class).getBody());
        // 此处无需映射 authorities，稍后由 OpaqueTokenAuthenticationConverter 完成
        return new OAuth2IntrospectionAuthenticatedPrincipal(claims, List.of());
    }

}
```
将这样的 `OpaqueTokenIntrospector` 暴露为 bean 即可，`spring-addons-starter-oidc` 会自动使用它，而不再创建默认实现。

## 7. 示例 `@RestController`
``` java
@RestController
@RequestMapping("/greet")
public class GreetingController {

    @GetMapping()
    @PreAuthorize("hasAuthority('NICE')")
    public MessageDto getGreeting(Authentication auth) {
        return new MessageDto("Hi %s! You are granted with: %s.".formatted(
                auth.getName(),
                auth.getAuthorities()));
    }

    static record MessageDto(String body) {
    }
}
```

## 8. 总结
在本教程中，我们为 Spring Boot 3 resource server 配置了几乎适用于任意 OpenID 授权服务器的 access token introspection，包括那些在 `.well-known/openid-configuration` 中未暴露 `introspection_endpoint` 的 Provider：对于此类 Provider，我们使用了（几乎始终存在的）`/userinfo` endpoint 并编写了自定义的 `OpaqueTokenIntrospector`。