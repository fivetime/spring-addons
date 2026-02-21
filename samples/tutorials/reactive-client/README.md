# 带登录、登出和 Authorities 映射的响应式 OAuth2 Client
在本教程中，我们将一个响应式（WebFlux）Spring Boot 3 应用配置为带有登录、登出和 authorities 映射的 OAuth2 client，并使用 OIDC Provider 上定义的角色实现 RBAC，**不使用 `spring-addons-starter-oidc`**——这使得其安全配置比 BFF gateway 的配置要冗长得多。

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 项目初始化
本教程在完成[前置条件](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials#2-prerequisites)之后开始，假设至少已配置 1 个 OIDC Provider（2 个更好），并且每个 OP 上都有拥有和没有 `NICE` 角色的用户。

### 1.1. Spring Boot Starter
与往常一样，从 http://start.spring.io/ 开始，添加以下依赖：
- Spring Reactive Web
- OAuth2 Client
- Lombok

### 1.2. 应用配置属性
解压项目后，将 `src/main/resources/application.properties` 替换为以下 `src/main/resources/application.yaml`：
```yaml
scheme: http
keycloak-port: 8442
keycloak-issuer: ${scheme}://localhost:${keycloak-port}/realms/master
keycloak-secret: change-me
cognito-issuer: https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl
cognito-secret: change-me
auth0-issuer: https://dev-ch4mpy.eu.auth0.com/
auth0-secret: change-me

server:
  ssl:
    enabled: false
      
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  security:
    oauth2:
      client:
        provider:
          keycloak:
            issuer-uri: ${keycloak-issuer}
          cognito:
            issuer-uri: ${cognito-issuer}
          auth0:
            issuer-uri: ${auth0-issuer}
        registration:
          keycloak-confidential-user:
            authorization-grant-type: authorization_code
            client-name: a local Keycloak instance
            client-id: spring-addons-confidential
            client-secret: ${keycloak-secret}
            provider: keycloak
            scope: openid,profile,email,offline_access
          cognito-confidential-user:
            authorization-grant-type: authorization_code
            client-name: Amazon Cognito
            client-id: 12olioff63qklfe9nio746es9f
            client-secret: ${cognito-secret}
            provider: cognito
            scope: openid,profile,email
          auth0-confidential-user:
            authorization-grant-type: authorization_code
            client-name: Auth0
            client-id: TyY0H7xkRMRe6lDf9F8EiNqCo8PdhICy
            client-secret: ${auth0-secret}
            provider: auth0
            scope: openid,profile,email,offline_access

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
以下几点值得注意：
- 我们为 3 个不同的 provider 各定义了一个 `authorization_code` client registration，这意味着用户需要选择一个进行认证（如果只配置了一个 `authorization_code` client registration，则跳过选择提示）
- 有一个 `ssl` profile，用于通过 SSL 运行应用
- 你必须替换 issuer URI，以及按照[前置条件](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials#2-prerequisites)操作后获得的 client ID 和 secret（删除未配置的 provider 和 registration）

### 1.3. 静态首页
还需要一个静态的 `src/main/resources/static/index.html` 页面，以便在认证后有内容可看：
```html
<!DOCTYPE HTML>
<html>

<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta name="description" content="">
    <meta name="author" content="">
    <title>Reactive Application</title>
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-/Y6pD6FV/Vv2HJnA6t+vslU6fwYXjCFtcEpHbNJ0lyAFsXTsjBbfaDjzALeQsN6M" crossorigin="anonymous">
    <link href="https://getbootstrap.com/docs/4.0/examples/signin/signin.css" rel="stylesheet" crossorigin="anonymous"/>
</head>

<body>
<div class="container">
    <h1 class="form-signin-heading">Static Index</h1>
    <a href="/login"><button type="button">Login</button></a>
    <a href="/logout"><button type="button">Logout</button></a>
</div>
</body>
```
现在可以运行应用并访问 http://localhost:8080。

初步来看，一切似乎正常：我们可以在任意已配置的 OIDC Provider 上登录：
- 登录前无法访问首页，会被重定向到登录页
- 在任意已配置的 Provider 上登录后，可以访问首页
- 登出后，无法再访问首页

但稍加测试就会遇到第一个问题：如果再次登录到已认证过的 OIDC Provider，系统不会提示输入凭据（登录静默完成）。要解决这个问题，需要配置 [RP-Initiated Logout](https://openid.net/specs/openid-connect-rpinitiated-1_0.html)，使 client 上的 session 失效能传播到 OP。

## 2. RP-Initiated Logout
我们来手动接管 web security 配置，自行定义 security filter chain。

### 2.1. 标准 RP-Initiated Logout
Spring 为实现了 RP-Initiated Logout 的 OIDC Provider 提供了 `ServerLogoutSuccessHandler`：`OidcClientInitiatedServerLogoutSuccessHandler`。

首先声明配置属性：
```java
@Configuration
@EnableWebFluxSecurity
@EnableReactiveMethodSecurity
public class WebSecurityConfig {
    @Bean
    SecurityWebFilterChain clientSecurityFilterChain(ServerHttpSecurity http, ReactiveClientRegistrationRepository clientRegistrationRepo) {
        http.oauth2Login();
        http.logout(logout -> {
            final var handler = new OidcClientInitiatedServerLogoutSuccessHandler(clientRegistrationRepo);
            handler.setPostLogoutRedirectUri("{baseUrl}");
            logout.logoutSuccessHandler(handler);
        });
        http.authorizeExchange(ex -> ex.pathMatchers("/login/**", "/oauth2/**").permitAll().anyExchange().authenticated());
        return http.build();
    }
}
```
很好！登出在 Keycloak 上如预期工作，但对于偏离标准的 Auth0 和 Cognito 则是另一回事：`end_session_endpoint` 未在 `.well-known/openid-configuration` 中列出，且 `post_logout_redirect_uri` 的参数名也不标准。

### 2.2. 非标准 RP-Initiated Logout
我们来自己编写一个 `ServerLogoutSuccessHandler`，用于指定登出 URI 以及登出后跳转 URI 的参数名。

首先声明配置属性：
```java
@Data
@Configuration
@ConfigurationProperties(prefix = "logout")
static class LogoutProperties {
    private Map<String, ProviderLogoutProperties> registration = new HashMap<>();

    @Data
    static class ProviderLogoutProperties {
        private URI logoutUri;
        private String postLogoutUriParameterName;
    }
}
```
将对应属性添加到 yaml 中：
```yaml
logout:
  registration:
    cognito-confidential-user:
      logout-uri: https://spring-addons.auth.us-west-2.amazoncognito.com/logout
      post-logout-uri-parameter-name: logout_uri
    auth0-confidential-user:
      logout-uri: ${auth0-issuer}v2/logout
      post-logout-uri-parameter-name: returnTo
```
现在，我们可以定义一个 logout success handler，解析此配置用于非标准 RP-Initiated Logout（参考 `OidcClientInitiatedServerLogoutSuccessHandler` 实现）：
```java
@RequiredArgsConstructor
static class AlmostOidcClientInitiatedServerLogoutSuccessHandler implements ServerLogoutSuccessHandler {
    private final LogoutProperties.ProviderLogoutProperties properties;
    private final ClientRegistration clientRegistration;
    private final String postLogoutRedirectUri;
    private final RedirectServerLogoutSuccessHandler serverLogoutSuccessHandler = new RedirectServerLogoutSuccessHandler();
    private final ServerRedirectStrategy redirectStrategy = new DefaultServerRedirectStrategy();

    @Override
    public Mono<Void> onLogoutSuccess(WebFilterExchange exchange, Authentication authentication) {
        // @formatter:off
        return Mono.just(authentication)
                .filter(OAuth2AuthenticationToken.class::isInstance)
                .filter((token) -> authentication.getPrincipal() instanceof OidcUser)
                .map(OAuth2AuthenticationToken.class::cast)
                .flatMap(oauthentication -> {
                    final var oidcUser = ((OidcUser) oauthentication.getPrincipal());
                    final var endSessionUri = UriComponentsBuilder.fromUri(properties.getLogoutUri())
                            .queryParam("client_id", clientRegistration.getClientId())
                            .queryParam("id_token_hint", oidcUser.getIdToken().getTokenValue())
                            .queryParam(properties.getPostLogoutUriParameterName(), postLogoutRedirectUri(exchange.getExchange().getRequest()).toString()).toUriString();
                    return Mono.just(endSessionUri);
                }).switchIfEmpty(this.serverLogoutSuccessHandler.onLogoutSuccess(exchange, authentication).then(Mono.empty()))
                .flatMap((endpointUri) -> this.redirectStrategy.sendRedirect(exchange.getExchange(), URI.create(endpointUri)));
        // @formatter:on
    }

    private String postLogoutRedirectUri(ServerHttpRequest request) {
        if (this.postLogoutRedirectUri == null) {
            return null;
        }
        // @formatter:off
        UriComponents uriComponents = UriComponentsBuilder.fromUri(request.getURI())
                .replacePath(request.getPath().contextPath().value())
                .replaceQuery(null)
                .fragment(null)
                .build();

        Map<String, String> uriVariables = new HashMap<>();
        String scheme = uriComponents.getScheme();
        uriVariables.put("baseScheme", (scheme != null) ? scheme : "");
        uriVariables.put("baseUrl", uriComponents.toUriString());

        String host = uriComponents.getHost();
        uriVariables.put("baseHost", (host != null) ? host : "");

        String path = uriComponents.getPath();
        uriVariables.put("basePath", (path != null) ? path : "");

        int port = uriComponents.getPort();
        uriVariables.put("basePort", (port == -1) ? "" : ":" + port);

        uriVariables.put("registrationId", clientRegistration.getRegistrationId());

        return UriComponentsBuilder.fromUriString(this.postLogoutRedirectUri)
                .buildAndExpand(uriVariables)
                .toUriString();
        // @formatter:on
    }
}
```
此 handler 适用于非标准 OP，但如果我们希望对 Keycloak 继续使用 Spring 的 logout success handler（同时避免为其定义 logout 属性），则需要一个门面来整合这两种实现：
```java
@RequiredArgsConstructor
static class DelegatingOidcClientInitiatedServerLogoutSuccessHandler implements ServerLogoutSuccessHandler {
    private final Map<String, ServerLogoutSuccessHandler> delegates;

    public DelegatingOidcClientInitiatedServerLogoutSuccessHandler(
            InMemoryReactiveClientRegistrationRepository clientRegistrationRepository,
            LogoutProperties properties,
            String postLogoutRedirectUri) {
        delegates = StreamSupport.stream(clientRegistrationRepository.spliterator(), false)
                .collect(Collectors.toMap(ClientRegistration::getRegistrationId, clientRegistration -> {
                    final var endSessionEnpoint = (String) (clientRegistration.getProviderDetails().getConfigurationMetadata().get("end_session_endpoint"));
                    if (StringUtils.hasText(endSessionEnpoint)) {
                        final var handler = new OidcClientInitiatedServerLogoutSuccessHandler(clientRegistrationRepository);
                        handler.setPostLogoutRedirectUri(postLogoutRedirectUri);
                        return handler;
                    }
                    final var registrationProperties = properties.getRegistration().get(clientRegistration.getRegistrationId());
                    if (registrationProperties == null) {
                        throw new MisconfigurationException(
                                "OAuth2 client registration \"%s\" has no end_session_endpoint in OpenID configuration nor spring-addons logout properties"
                                        .formatted(clientRegistration.getRegistrationId()));
                    }
                    return new AlmostOidcClientInitiatedServerLogoutSuccessHandler(registrationProperties, clientRegistration, postLogoutRedirectUri);
                }));
    }

    @Override
    public Mono<Void> onLogoutSuccess(WebFilterExchange exchange, Authentication authentication) {
        return Mono.just(authentication).filter(OAuth2AuthenticationToken.class::isInstance).map(OAuth2AuthenticationToken.class::cast)
                .flatMap(oauthentication -> delegates.get(oauthentication.getAuthorizedClientRegistrationId()).onLogoutSuccess(exchange, authentication));
    }

}
```
此 handler 根据情况在 Spring 的 `OidcClientInitiatedServerLogoutSuccessHandler`（当 `.well-known/openid-configuration` 中暴露了 `end_session_endpoint` 时使用）和我们的 `AlmostOidcClientInitiatedServerLogoutSuccessHandler`（当存在 logout 配置属性时使用，否则抛出异常）之间切换。

最后，需要更新 security filter chain 以使用新的 `DelegatingOidcClientInitiatedServerLogoutSuccessHandler`：
```java
@Bean
SecurityWebFilterChain clientSecurityFilterChain(
        ServerHttpSecurity http,
        InMemoryReactiveClientRegistrationRepository clientRegistrationRepository,
        LogoutProperties logoutProperties) {
    http.oauth2Login();
    http.logout(logout -> {
        logout.logoutSuccessHandler(new DelegatingOidcClientInitiatedServerLogoutSuccessHandler(clientRegistrationRepository, logoutProperties, "{baseUrl}"));
    });
    http.authorizeExchange(ex -> ex.pathMatchers("/login/**", "/oauth2/**").permitAll().anyExchange().authenticated());
    return http.build();
}
```

## 3. 角色映射
我们将按以下规范实现从 OpenID 私有 claim 映射 Spring Security authorities：
- 可使用多个 claim 作为来源，所有 claim 均为字符串数组或逗号分隔的单个字符串（例如 Keycloak 可以在 `realm_access.roles` 和 `resource_access.{client-id}.roles` 中提供角色）
- 对每个 claim 独立配置大小写处理和前缀（例如对 `scp` claim 中的 scope 使用 `SCOPE_` 前缀，对 `realm_access.roles` 中的角色使用 `ROLE_` 前缀）
- 为每个 provider 提供独立的配置（Keycloak、Auth0 和 Cognito 各自使用不同的私有 claim 存储用户角色）

### 3.1. 依赖配置
为简化角色 claim 的解析，我们使用 [json-path](https://central.sonatype.com/artifact/com.jayway.jsonpath/json-path/2.8.0)，将其添加到依赖中：
```xml
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path</artifactId>
</dependency>
```

### 3.2. 应用配置属性
然后，我们需要一些额外的配置属性来满足上述灵活性需求：
```java
@Data
@Configuration
@ConfigurationProperties(prefix = "authorities-mapping")
public class AuthoritiesMappingProperties {
    private IssuerAuthoritiesMappingProperties[] issuers = {};

    @Data
    static class IssuerAuthoritiesMappingProperties {
        private URL uri;
        private ClaimMappingProperties[] claims;

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

    public IssuerAuthoritiesMappingProperties get(URL issuerUri) throws MisconfigurationException {
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
authorities-mapping:
  issuers:
  - uri: ${keycloak-issuer}
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

### 3.3. `GrantedAuthoritiesMapper`
根据[官方文档](https://docs.spring.io/spring-security/reference/reactive/oauth2/login/advanced.html#webflux-oauth2-login-advanced-map-authorities)，有两种选择：
- 提供一个 `GrantedAuthoritiesMapper` bean
- 提供并配置一个 `ReactiveOAuth2UserService<OidcUserRequest, OidcUser>`

我们选择第一种方案：更轻量、更简单，足以满足需求。以文档中的模板为起点：
```java
@Component
@RequiredArgsConstructor
static class GrantedAuthoritiesMapperImpl implements GrantedAuthoritiesMapper {
    private final AuthoritiesMappingProperties properties;

    @Override
    public Collection<? extends GrantedAuthority> mapAuthorities(Collection<? extends GrantedAuthority> authorities) {
        Set<GrantedAuthority> mappedAuthorities = new HashSet<>();

        authorities.forEach(authority -> {
            if (OidcUserAuthority.class.isInstance(authority)) {
                final var oidcUserAuthority = (OidcUserAuthority) authority;
                final var issuer = oidcUserAuthority.getIdToken().getClaimAsURL(JwtClaimNames.ISS);
                mappedAuthorities.addAll(extractAuthorities(oidcUserAuthority.getIdToken().getClaims(), properties.get(issuer)));

            } else if (OAuth2UserAuthority.class.isInstance(authority)) {
                try {
                    final var oauth2UserAuthority = (OAuth2UserAuthority) authority;
                    final var userAttributes = oauth2UserAuthority.getAttributes();
                    final var issuer = new URL(userAttributes.get(JwtClaimNames.ISS).toString());
                    mappedAuthorities.addAll(extractAuthorities(userAttributes, properties.get(issuer)));

                } catch (MalformedURLException e) {
                    throw new RuntimeException(e);
                }
            }
        });

        return mappedAuthorities;
    };

    private static
            Collection<GrantedAuthority>
            extractAuthorities(Map<String, Object> claims, AuthoritiesMappingProperties.IssuerAuthoritiesMappingProperties properties) {
        return Stream.of(properties.claims).flatMap(claimProperties -> {
            Object claim;
            try {
                claim = JsonPath.read(claims, claimProperties.jsonPath);
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
现在，我们可以在 Spring 应用中使用用户在任意 OIDC Provider 上定义的角色了。

### 3.4. 基于角色的访问控制
为演示 RBAC，我们定义一个新的 `src/main/resources/static/nice.html` 页面，该页面只有拥有 `NICE` 角色的用户才能访问：
```html
<!DOCTYPE HTML>
<html>

<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta name="description" content="">
    <meta name="author" content="">
    <title>Servlet Client</title>
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-/Y6pD6FV/Vv2HJnA6t+vslU6fwYXjCFtcEpHbNJ0lyAFsXTsjBbfaDjzALeQsN6M" crossorigin="anonymous">
    <link href="https://getbootstrap.com/docs/4.0/examples/signin/signin.css" rel="stylesheet" crossorigin="anonymous"/>
</head>

<body>
<div class="container">
    <h1 class="form-signin-heading">You are so nice!</h1>
</div>
</body>
```
这当然需要相应地更新 security 配置：
```java
        http.authorizeExchange(ex -> ex
                .pathMatchers("/login/**", "/oauth2/**").permitAll()
                .pathMatchers("/nice.html").hasAuthority("NICE")
                .anyExchange().authenticated());
```
现在，只有在[前置条件](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials#2-prerequisites)配置 OIDC Provider 时授予了 `NICE` 角色的用户，才能访问 [http://localhost:8080/nice.html](http://localhost:8080/nice.html) 页面（启用 `ssl` profile 时为 [https://localhost:8080/nice.html](https://localhost:8080/nice.html)）。

## 4. 防止同时登录多个身份
由于首页是静态页面（生成的登录页也是），它无法感知用户的认证状态。因此，已认证的用户可以继续用第二个 OIDC Provider 登录。

这在本质上没有根本性问题（用户确实可以在多个 OP 上拥有不同的数字身份），但 Spring 对此没有开箱即用的支持：`OAuth2AuthenticationToken` 绑定到单个 `OAuth2User`，而后者只支持一个 `subject`（最后一次认证的那个），且 authorized client repository 需要这个 `subject` 来检索 authorized client。

因此，如果用户在应用中依次使用多个 OP 登录，只有最后一个身份可用，且登出只会终止最后一个 OP 上的 session。

由于提供自定义的 `ServerOAuth2AuthorizedClientRepository` 相当复杂，我们接下来要实现一个防护机制，阻止已认证的用户再次登录：用户必须先登出才能重新登录。

### 4.1. Thymeleaf 首页
第一步是将静态首页替换为能根据用户认证状态动态调整的模板：向未认证用户显示 `login` 按钮，向已认证用户显示 `logout` 按钮。

为此，先将 [Thymeleaf starter](https://central.sonatype.com/artifact/org.springframework.boot/spring-boot-starter-thymeleaf/3.0.5) 添加到依赖中：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

然后需要一个 `@Controller` 来检查用户认证状态，并据此设置 `Model`：
```java
@Controller
public class IndexController {

    @GetMapping("/")
    public Mono<String> getIndex(Authentication auth, Model model) {
        model.addAttribute("isAuthenticated", auth != null && auth.isAuthenticated() && !(auth instanceof AnonymousAuthenticationToken));
        return Mono.just("index.html");
    }
}
```

最后，将 `src/main/resources/static/index.html` 复制到 `src/main/resources/templates/`，并编辑以使用 controller 中设置的 model：
```java
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">

<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta name="description" content="">
    <meta name="author" content="">
    <title>Servlet Client</title>
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-/Y6pD6FV/Vv2HJnA6t+vslU6fwYXjCFtcEpHbNJ0lyAFsXTsjBbfaDjzALeQsN6M" crossorigin="anonymous">
    <link href="https://getbootstrap.com/docs/4.0/examples/signin/signin.css" rel="stylesheet" crossorigin="anonymous"/>
</head>

<body>
<div class="container">
    <h1 class="form-signin-heading">Dynamic Index</h1>
    <a th:if="!${isAuthenticated}" href="/login"><button type="button">Login</button></a>
    <a th:if="${isAuthenticated}" href="/logout"><button type="button">Logout</button></a>
</div>
</body>
```

### 4.2. Security 配置更新
访问规则需要更新。现在我们希望：
- 允许所有用户（无论是否认证）访问首页
- 只允许未认证用户访问登录页

第一条规则很容易实现：将 `/` 添加到 `permitAll()` 列表中。但第二条规则需要在登录过滤器之前插入一个过滤器（有一个特殊的"登录页"过滤器会绕过请求授权）：
```java
private WebFilter loginPageWebFilter() {
    return (ServerWebExchange exchange, WebFilterChain chain) -> {
        return ReactiveSecurityContextHolder.getContext()
                .defaultIfEmpty(
                        new SecurityContextImpl(
                                new AnonymousAuthenticationToken("anonymous", "anonymousUser", AuthorityUtils.createAuthorityList("ROLE_ANONYMOUS"))))
                .flatMap(ctx -> {
                    final var auth = ctx.getAuthentication();
                    if (auth != null
                            && auth.isAuthenticated()
                            && !(auth instanceof AnonymousAuthenticationToken)
                            && exchange.getRequest().getPath().toString().equals("/login")) {
                        exchange.getResponse().setStatusCode(HttpStatus.TEMPORARY_REDIRECT);
                        exchange.getResponse().getHeaders().setLocation(URI.create("/"));
                        return exchange.getResponse().setComplete();
                    }
                    return chain.filter(exchange);
                });
    };
}
```
然后可以按如下方式更新 security filter chain 配置：
```java
@Bean
SecurityWebFilterChain clientSecurityFilterChain(
        ServerHttpSecurity http,
        InMemoryReactiveClientRegistrationRepository clientRegistrationRepository,
        LogoutProperties logoutProperties) {
    http.addFilterBefore(loginPageWebFilter(), SecurityWebFiltersOrder.LOGIN_PAGE_GENERATING);
    http.oauth2Login();
    http.logout(logout -> {
        logout.logoutSuccessHandler(
                new DelegatingOidcClientInitiatedServerLogoutSuccessHandler(clientRegistrationRepository, logoutProperties, "{baseUrl}"));
    });
    // @formatter:off
    http.authorizeExchange(ex -> ex
            .pathMatchers("/", "/login/**", "/oauth2/**").permitAll()
            .pathMatchers("/nice.html").hasAuthority("NICE")
            .anyExchange().authenticated());
    // @formatter:on
    return http.build();
}
```

## 5. 总结
在本教程中，我们配置了一个带有登录、登出和角色映射的 servlet OAuth2 client。

但且慢，我们在这里所做的一切相当冗长，而几乎每个 OAuth2 client 都会用到这些配置。难道每次都要重复写这些吗？其实不必：本仓库提供了 [`spring-addons-webflux-client`](https://github.com/ch4mpy/spring-addons/tree/master/webflux/spring-addons-webflux-client) Spring Boot starter 正是为此而生。如果出于某种原因不想使用它，也可以编写[自己的 starter](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration)，将我们在这里编写的配置封装起来。

另外，如果我们确实需要用户同时持有多个 authorized client（例如同时在 Google 和 Facebook 上认证，以从同一个 client 查询 Google API 和 Facebook Graph），可以像上一节建议的那样提供一个自定义的 `ServerOAuth2AuthorizedClientRepository`。本仓库的 client starter 提供了这样的实现，它在用户 session 中按 issuer 存储认证信息，并在尝试检索 authorized client 前解析正确的认证（及其 subject）。同样，如果不想使用这些 starter，也只能[自己实现](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration)。

最后，可以查看源码中的集成测试，了解如何通过 mock security context 验证 `/`、`/login` 和 `nice.html` 的访问控制规则。