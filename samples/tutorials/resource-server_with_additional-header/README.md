# 同时包含 Access Token 和自定义请求头数据的 Authentication

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 概述
在本教程中，我们假设除了 `Authorization` 请求头中的 JWT **access token** 之外，OAuth2 client 还会在 `X-ID-Token` 请求头中附带一个 JWT **ID token**。

请确保你的环境满足[教程前置条件](https://github.com/fivetime/spring-addons/blob/master/samples/tutorials/README.md#prerequisites)。

## 2. 项目初始化
借助 https://start.spring.io/ 创建一个 Spring Boot 3 项目，需要以下依赖：
- Spring Web
- OAuth2 Resource Server
- Lombok

然后添加以下依赖：
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

如果出于某种原因不想使用 `spring-addons-starter-oidc`，可以参考 [`servlet-resource-server`](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/servlet-resource-server) 或 [`reactive-resource-server`](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials/reactive-resource-server)，了解仅使用 `spring-boot-starter-oauth2-resource-server` 的基本配置方式。剧透：那样会繁琐得多，也更容易出错。

## 3. Web Security 配置
本配置将使用非常便捷的 [`HttpServletRequestSupport`](https://github.com/fivetime/spring-addons/blob/master/spring-addons-starter-oidc/src/main/java/com/c4_soft/springaddons/security/oidc/starter/synchronised/HttpServletRequestSupport.java)，它提供了访问当前请求（在本例中为其请求头）的工具方法。如果编写的是 WebFlux 应用，则应使用其响应式对应版本：[`ServerHttpRequestSupport`](https://github.com/fivetime/spring-addons/blob/master/spring-addons-starter-oidc/src/main/java/com/c4_soft/springaddons/security/oidc/starter/reactive/ServerHttpRequestSupport.java)。如果不使用 `spring-addons-starter-oidc`，可能需要从上述工具类中复制部分代码。

`spring-oauth2-addons` 提供了适用于 REST API 项目的 web security 配置 `@AutoConfiguration`。我们只需要：
- 添加 `@EnableMethodSecurity` 以在组件方法上启用 `@PreAuthorize`
- 定义自己的 authentication 类，用于在 access token 信息之外额外持有 ID token 字符串和 claim：
```java
@Data
@EqualsAndHashCode(callSuper = true)
public static class MyAuth extends OAuthentication<OpenidToken> {
  private static final long serialVersionUID = 1734079415899000362L;
  private final OpenidToken idToken;

  public MyAuth(Collection<? extends GrantedAuthority> authorities, String accessTokenString,
      OpenidClaimSet accessClaims, String idTokenString, OpenidClaimSet idClaims) {
    super(new OpenidToken(accessClaims, accessTokenString), authorities);
    this.idToken = new OpenidToken(idClaims, idTokenString);
  }

}
```
- 提供一个 `JwtAbstractAuthenticationTokenConverter` bean，将 `Authentication` 实现从 `JwtAuthenticationToken` 切换为 `MyAuth`：
```java
@Bean
JwtAbstractAuthenticationTokenConverter authenticationConverter(
    // 注入一个将 token claim 转换为 Spring authority 的 converter。若未自定义，spring-addons-starter-oidc 会提供默认实现
    Converter<Map<String, Object>, Collection<? extends GrantedAuthority>> authoritiesConverter) {
  return jwt -> {
    try {
      // 根据 token claim 解析对应的 JWT decoder（详见下文）
      final var jwtDecoder = getJwtDecoder(jwt.getClaims());
      final var authorities = authoritiesConverter.convert(jwt.getClaims());
      final var idTokenString =
          HttpServletRequestSupport.getUniqueRequestHeader(ID_TOKEN_HEADER_NAME);
      final var idToken = jwtDecoder == null ? null : jwtDecoder.decode(idTokenString);

      return new MyAuth(authorities, jwt.getTokenValue(), new OpenidClaimSet(jwt.getClaims()),
          idTokenString, new OpenidClaimSet(idToken.getClaims()));
    } catch (JwtException e) {
      throw new InvalidHeaderException(ID_TOKEN_HEADER_NAME);
    }
  };
}
```
- 为 ID token 的 JWT decoder 建立缓存（每个 ID token issuer 只实例化一个 decoder）。为此，在配置类中添加以下代码：
```java
private static final Map<String, JwtDecoder> idTokenDecoders = new ConcurrentHashMap<>();

private JwtDecoder getJwtDecoder(Map<String, Object> accessClaims) {
  if (accessClaims == null) {
    return null;
  }
  final var iss =
      Optional.ofNullable(accessClaims.get(JwtClaimNames.ISS)).map(Object::toString).orElse(null);
  if (iss == null) {
    return null;
  }
  if (!idTokenDecoders.containsKey(iss)) {
    idTokenDecoders.put(iss, JwtDecoders.fromIssuerLocation(iss));
  }
  return idTokenDecoders.get(iss);
}
```

## 4. 应用配置属性
这里没有什么特别之处，只是常规的 Spring Boot 和 spring-addons 配置（接受来自 3 个不同受信任 issuer 的身份）：
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
        - iss: ${cognito-issuer}
          username-claim: username
          authorities:
          - path: cognito:groups
        - iss: ${auth0-issuer}
          username-claim: $['https://c4-soft.com/user']['name']
          authorities:
          - path: $['https://c4-soft.com/user']['roles']
          - path: $.permissions
```

## 5. 示例 `@RestController`
请注意，OpenID 标准 claim 可以通过 getter 访问（而不像 `JwtAuthenticationToken` 那样使用 `Map<String, Object>`）：
``` java
@RestController
@PreAuthorize("isAuthenticated()")
public class GreetingController {

	@GetMapping("/greet")
	public MessageDto getGreeting(MyAuth auth) {
		return new MessageDto(
				"Hi %s! You are granted with: %s.".formatted(
						auth.getIdClaims().getEmail(), // 来自 X-ID-Token 请求头中的 ID token
						auth.getAuthorities())); // 来自 Authorization 请求头中的 access token
	}

	static record MessageDto(String body) {
	}
}
```

## 6. 总结
本示例引导你构建了一个从 access token 和自定义请求头两处提取安全数据的 servlet（webmvc）应用。

对于响应式（webflux）应用，主要区别在于：
- 使用 `spring-addons-webflux-jwt-resource-server` 作为依赖（替代 `spring-addons-webmvc-jwt-resource-server`）
- 使用 `ServerHttpRequestSupport`（替代 `HttpServletRequestSupport`）从请求头中获取 ID token