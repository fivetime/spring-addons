# 配置响应式 OAuth2 Resource Server（REST API）
在本教程中，我们将一个响应式（WebFlux）Spring Boot 3 应用配置为 OAuth2 resource server，并映射 authorities 以使用 OIDC Provider 上定义的角色实现 RBAC，*不使用 `spring-addons-starter-oidc`*——这使得配置比 samples 中的"webflux"项目要冗长得多。

我们还将了解如何接受由多个（可能异构的）OIDC Provider（或 Keycloak realm）签发的 access token。

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 项目初始化
本教程在完成[前置条件](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials#2-prerequisites)之后开始，假设至少已配置 1 个 OIDC Provider（2 个更好），并且每个 OP 上都有拥有和没有 `NICE` 角色的用户。与往常一样，从 http://start.spring.io/ 开始，添加以下依赖：
- Spring Reactive Web
- OAuth2 Resource Server
- Lombok

## 2. REST Controller
我们使用一个非常简单的 controller，仅访问基本的 `Authentication` 属性：
```java
@RestController
public class GreetingController {

    @GetMapping("/greet")
    public Mono<MessageDto> getGreeting(Authentication auth) {
        return Mono.just(new MessageDto("Hi %s! You are granted with: %s.".formatted(auth.getName(), auth.getAuthorities())));
    }

    static record MessageDto(String body) {
    }
}
```
这足以演示：根据签发 access token 的授权服务器不同（Keycloak、Auth0 或 Cognito），用户名和角色会从不同的 claim 中映射而来。

## 3. Security 配置
我们希望 REST API 的配置如下：
- 使用 OAuth2 进行请求授权
- 接受来自 3 个不同 OIDC 授权服务器（Keycloak、Auth0 和 Cognito）签发的身份
- 启用 CORS（allowed origins 可配置）
- 无状态 session 管理（无 session，用户状态仅存在于 access token 中）
- 禁用 CSRF（因为没有 session，所以是安全的）
- 有限的资源列表允许公开访问
- 非"公开"路由要求用户已认证，细粒度访问控制通过方法级安全（`@PreAuthorize` 等）实现
- 对受保护资源的请求在授权缺失或无效时返回 401（未授权），而非 302（重定向到登录页）

## 3.1. Security 配置属性
将 `application.properties` 替换为内容如下的 `application.yml`：
```yaml
scheme: http
origins: http://localhost:4200,https://localhost:4200
permit-all: /public/**
keycloak-port: 8442
keycloak-issuer: ${scheme}://localhost:${keycloak-port}/realms/master
cognito-issuer: https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl
auth0-issuer: https://dev-ch4mpy.eu.auth0.com/

server:
  error:
    include-message: always
  ssl:
    enabled: false

spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${keycloak-issuer}

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

---
spring:
  config:
    activate:
      on-profile: auth0
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${auth0-issuer}

---
spring:
  config:
    activate:
      on-profile: cognito
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${cognito-issuer}
```
以下几点值得注意：
- `spring.security.oauth2.resourceserver.jwt.issuer-uri` 是单值配置，这意味着 Spring Boot 一次只能接受来自单个 OIDC Provider 签发的 OAuth2 身份。我们通过 profile 在不同 OP 之间切换，这需要重启，但我们后面会介绍如何配置 spring-security 让单个 resource server 实例同时接受来自多个 OIDC Provider 的身份。
- 其中有些值（issuer URI）需要替换为完成[前置条件](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials#2-prerequisites)时获取的实际值。

### 3.2. Security Filter Chain
由于开发和生产环境允许的 origin 可能不同，我们需要一个以 allowed-origins 为参数的 CORS 配置函数。

另外，当对受保护资源的请求缺少或携带无效授权时，HTTP 状态码应为 401（`Unauthorized`），但 Spring 默认返回 302（重定向到登录页，这在 resource server 上毫无意义）。要修改这一行为，我们需要编写一个 access denied handler。

将安全配置整合到一起，提供一个满足所有规范的 security filter chain bean：
```java
@EnableWebFluxSecurity
@EnableReactiveMethodSecurity
@Configuration
public class WebSecurityConfig {

    @Bean
    SecurityWebFilterChain
            filterChain(ServerHttpSecurity http, ServerProperties serverProperties, @Value("origins") String[] origins, @Value("permit-all") String[] permitAll)
                    throws Exception {

        http.oauth2ResourceServer(resourceServer -> resourceServer.jwt());

        http.cors(cors -> cors.configurationSource(corsConfigurationSource(origins)));

        // 无状态 session（状态仅存在于 access token 中）
        http.securityContextRepository(NoOpServerSecurityContextRepository.getInstance());

        // 因为 session 是无状态的，禁用 CSRF
        http.csrf().disable();

        // 授权缺失或无效时返回 401（未授权），而非 302（重定向到登录页）
        http.exceptionHandling(exceptionHandling -> {
            exceptionHandling.accessDeniedHandler(accessDeniedHandler());
        });

        // 如果启用了 SSL，禁用 http（仅允许 https）
        if (serverProperties.getSsl() != null && serverProperties.getSsl().isEnabled()) {
            http.redirectToHttps();
        }

        http.authorizeExchange(exchange -> exchange.pathMatchers(permitAll).permitAll().anyExchange().authenticated());

        return http.build();
    }

    private UrlBasedCorsConfigurationSource corsConfigurationSource(String[] origins) {
        final var configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(Arrays.asList(origins));
        configuration.setAllowedMethods(List.of("*"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setExposedHeaders(List.of("*"));
    
        final var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
    
    private ServerAccessDeniedHandler accessDeniedHandler() {
        return (var exchange, var ex) -> exchange.getPrincipal().flatMap(principal -> {
            var response = exchange.getResponse();
            response.setStatusCode(principal instanceof AnonymousAuthenticationToken ? HttpStatus.UNAUTHORIZED : HttpStatus.FORBIDDEN);
            response.getHeaders().setContentType(MediaType.TEXT_PLAIN);
            var dataBufferFactory = response.bufferFactory();
            var buffer = dataBufferFactory.wrap(ex.getMessage().getBytes(Charset.defaultCharset()));
            return response.writeWith(Mono.just(buffer)).doOnError(error -> DataBufferUtils.release(buffer));
        });
    }
}
```

### 3.3. Authorities 映射
提醒一下，token 的 `scope` 定义的是 resource-owner 允许 OAuth2 client 代表其执行的操作，而"角色"则是表示 resource-owner 自身在 resource server 上被允许执行的操作的一种方式。

RBAC 是一种非常常见的访问控制模式，但 OAuth2 和 OpenID 都没有定义"角色"的标准表示方式。每个厂商都用自己的私有 claim 来实现。

Spring Security 默认的 authorities mapper 从 `scope` claim 中映射并添加 `SCOPE_` 前缀，这不能满足我们的需求（除非在授权服务器上将 `scope` claim 的用途扭曲为包含用户角色）。

我们将按以下规范实现从 OpenID 私有 claim 映射 Spring Security authorities：
- 可使用多个 claim 作为来源，所有 claim 均为字符串数组或逗号分隔的单个字符串（例如 Keycloak 可以在 `realm_access.roles` 和 `resource_access.{client-id}.roles` 中提供角色）
- 对每个 claim 独立配置大小写处理和前缀（例如对 `scp` claim 中的 scope 使用 `SCOPE_` 前缀，对 `realm_access.roles` 中的角色使用 `ROLE_` 前缀）
- 为每个 provider 提供独立的配置（Keycloak、Auth0 和 Cognito 各自使用不同的私有 claim 存储用户角色）

#### 3.3.1. 依赖配置
为简化角色 claim 的解析，我们使用 [json-path](https://central.sonatype.com/artifact/com.jayway.jsonpath/json-path/2.8.0)，将其添加到依赖中：
```xml
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path</artifactId>
</dependency>
```

#### 3.3.2. 应用配置属性
然后，我们需要一些额外的配置属性来灵活地为每个 issuer 配置包含用户名和角色的 claim：
```java
@Data
@Configuration
@ConfigurationProperties(prefix = "spring-addons")
public class SpringAddonsProperties {
    private IssuerProperties[] issuers = {};

    @Data
    static class IssuerProperties {
        private URL uri;
        private ClaimMappingProperties[] claims;
        private String usernameJsonPath = JwtClaimNames.SUB;

        @Data
        static class ClaimMappingProperties {
            private String jsonPath;
            private CaseProcessing caseProcessing = CaseProcessing.UNCHANGED;
            private String prefix = "";

            static enum CaseProcessing {
                UNCHANGED, TO_LOWER, TO_UPPER
            }
        }
    }

    public IssuerProperties get(URL issuerUri) throws MisconfigurationException {
        final var issuerProperties = Stream.of(issuers).filter(iss -> issuerUri.equals(iss.getUri())).toList();
        if (issuerProperties.size() == 0) {
            throw new MisconfigurationException("Missing authorities mapping properties for %s".formatted(issuerUri.toString()));
        }
        if (issuerProperties.size() > 1) {
            throw new MisconfigurationException("Too many authorities mapping properties for %s".formatted(issuerUri.toString()));
        }
        return issuerProperties.get(0);
    }

    static class MisconfigurationException extends RuntimeException {
        private static final long serialVersionUID = 5887967904749547431L;

        public MisconfigurationException(String msg) {
            super(msg);
        }
    }
}
```
还需要在 yaml 中添加与此配置对应的属性：
```yaml
spring-addons:
  issuers:
  - uri: ${keycloak-issuer}
    username-json-path: $.preferred_username
    claims:
    - jsonPath: $.realm_access.roles
    - jsonPath: $.resource_access.*.roles
  - uri: ${cognito-issuer}
    claims:
    - jsonPath: $.cognito:groups
  - uri: ${auth0-issuer}
    claims:
    - jsonPath: $.roles
    - jsonPath: $.groups
    - jsonPath: $.permissions
```

#### 3.3.3. Authorities 和 Authentication Converter
现在我们可以编写一个 converter bean，按照上述配置从 JWT claim 构建 Spring authorities：
```java
@RequiredArgsConstructor
static class JwtGrantedAuthoritiesConverter implements Converter<Jwt, Collection<? extends GrantedAuthority>> {
    private final SpringAddonsProperties.IssuerProperties properties;

    @Override
    @SuppressWarnings({ "rawtypes", "unchecked" })
    public Collection<? extends GrantedAuthority> convert(Jwt jwt) {
        return Stream.of(properties.claims).flatMap(claimProperties -> {
            Object claim;
            try {
                claim = JsonPath.read(jwt.getClaims(), claimProperties.jsonPath);
            } catch (PathNotFoundException e) {
                claim = null;
            }
            if (claim == null) {
                return Stream.empty();
            }
            if (claim instanceof String claimStr) {
                return Stream.of(claimStr.split(","));
            }
            if (claim instanceof String[] claimArr) {
                return Stream.of(claimArr);
            }
            if (Collection.class.isAssignableFrom(claim.getClass())) {
                final var iter = ((Collection) claim).iterator();
                if (!iter.hasNext()) {
                    return Stream.empty();
                }
                final var firstItem = iter.next();
                if (firstItem instanceof String) {
                    return (Stream<String>) ((Collection) claim).stream();
                }
                if (Collection.class.isAssignableFrom(firstItem.getClass())) {
                    return (Stream<String>) ((Collection) claim).stream().flatMap(colItem -> ((Collection) colItem).stream()).map(String.class::cast);
                }
            }
            return Stream.empty();
        }).map(SimpleGrantedAuthority::new).map(GrantedAuthority.class::cast).toList();
    }
}
```
该 authorities converter 将由如下 authentication converter 实例化并使用：
```java
@Component
@RequiredArgsConstructor
static class SpringAddonsJwtAuthenticationConverter implements Converter<Jwt, Mono<? extends AbstractAuthenticationToken>> {
    private final SpringAddonsProperties springAddonsProperties;

    @Override
    public Mono<? extends AbstractAuthenticationToken> convert(Jwt jwt) {
        final var issuerProperties = springAddonsProperties.get(jwt.getIssuer());
        final var authorities = new JwtGrantedAuthoritiesConverter(issuerProperties).convert(jwt);
        final String username = JsonPath.read(jwt.getClaims(), issuerProperties.getUsernameJsonPath());
        return Mono.just(new JwtAuthenticationToken(jwt, authorities, username));
    }
}
```

#### 3.3.4. Security 配置更新
最后缺少的一块配置是更新 security filter chain：将我们的 authentication converter 注入 resource server 配置中：
```java
@Bean
SecurityWebFilterChain filterChain(
        ServerHttpSecurity http,
        ServerProperties serverProperties,
        @Value("origins") String[] origins,
        @Value("permit-all") String[] permitAll,
        SpringAddonsProperties springAddonsProperties,
        SpringAddonsJwtAuthenticationConverter authenticationConverter)
        throws Exception {

    http.oauth2ResourceServer(resourceServer -> resourceServer.jwt(jwtConf -> jwtConf.jwtAuthenticationConverter(authenticationConverter)));

...

return http.build();
}
```
**太棒了，现在只需编辑一个配置属性，就能从任意 OIDC Provider 签发的任意 JWT access token 中映射 authorities 了！**

#### 3.3.5. 受限端点
为演示 **R**ole **B**ased **A**ccess **C**ontrol（RBAC），我们在 `@RestController` 中添加一个只有拥有 `NICE` 角色的用户才能访问的新端点：
```java
@GetMapping("/restricted")
@PreAuthorize("hasAuthority('NICE')")
public Mono<MessageDto> getRestricted() {
    return Mono.just(new MessageDto("You are so nice!"));
}
```

### 4. 多租户
到目前为止，我们的 resource server 可以接受来自任意 OIDC Provider 签发的身份，但每个 resource server 实例每次只能支持一个 OP。这在大多数情况下已经足够，但并非总是如此。

例如，在使用 Keycloak 时，可以将所有 realm 视为不同的 OP。要让单个 resource server 实例同时接受来自不同 realm 的用户请求，需要提供一些额外配置。

首先移除不适用于此场景的 `spring.security.oauth2.resourceserver.jwt.issuer-uri`，改为遍历 `spring-addons.issuers`。同时也可以移除 `auth0` 和 `cognito` profile。

然后定义一个 `ReactiveAuthenticationManagerResolver`：
```java
@Bean
ReactiveAuthenticationManagerResolver<ServerWebExchange>
        authenticationManagerResolver(SpringAddonsProperties addonsProperties, SpringAddonsJwtAuthenticationConverter authenticationConverter) {
    final Map<String, Mono<ReactiveAuthenticationManager>> jwtManagers = Stream.of(addonsProperties.getIssuers()).map(IssuerProperties::getUri)
            .map(URL::toString).collect(Collectors.toMap(issuer -> issuer, issuer -> Mono.just(authenticationManager(issuer, authenticationConverter))));
    return new JwtIssuerReactiveAuthenticationManagerResolver(issuerLocation -> jwtManagers.getOrDefault(issuerLocation, Mono.empty()));
}

JwtReactiveAuthenticationManager authenticationManager(String issuer, SpringAddonsJwtAuthenticationConverter authenticationConverter) {
    ReactiveJwtDecoder decoder = ReactiveJwtDecoders.fromIssuerLocation(issuer);
    var provider = new JwtReactiveAuthenticationManager(decoder);
    provider.setJwtAuthenticationConverter(authenticationConverter);
    return provider;
}
```

最后，在 security filter chain 中配置 resource server 时，将 authentication converter 配置替换为新的 authentication manager resolver：
```java
@Bean
SecurityWebFilterChain filterChain(
        ServerHttpSecurity http,
        ServerProperties serverProperties,
        @Value("origins") String[] origins,
        @Value("permit-all") String[] permitAll,
        SpringAddonsProperties springAddonsProperties,
        ReactiveAuthenticationManagerResolver<ServerWebExchange> authenticationManagerResolver)
        throws Exception {

    // 配置应用为 resource server，使用能够处理多租户的 authentication manager resolver
    http.oauth2ResourceServer(resourceServer -> resourceServer.authenticationManagerResolver(authenticationManagerResolver));
    ...
    return http.build();
}
```

## 5. 测试
请查阅源码，了解如何在单元测试和集成测试中 mock 身份，以及如何断言访问控制行为符合预期。所有示例和教程都包含详细的访问控制测试。

## 6. 总结
在本教程中，我们将一个响应式（WebFlux）Spring Boot 3 应用配置为 OAuth2 resource server，并映射 authorities 以使用任意数量的 OIDC Provider（或 Keycloak realm）上定义的角色实现 RBAC，无论这些 Provider 是否将用户角色存放在相同的 claim 中。

但且慢，我们在这里所做的一切相当冗长，而几乎每个 OAuth2 resource server 都会用到这些配置。难道每次都要重复写这些吗？其实不必：本仓库提供了 [`spring-addons-webflux-jwt-resource-server`](https://github.com/fivetime/spring-addons/tree/master/webflux/spring-addons-webflux-jwt-resource-server) Spring Boot starter 正是为此而生，[这里有一个使用示例](https://github.com/fivetime/spring-addons/tree/master/samples/webflux-jwt-default)。如果出于某种原因不想使用它，也可以编写[自己的 starter](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration)，将我们在这里编写的配置封装起来。