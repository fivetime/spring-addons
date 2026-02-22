# Keycloak 与 Spring Boot 使用快速指南

> 原文来源：Baeldung

## 1. 概述

本教程介绍如何使用 Spring Boot 和 Keycloak 配置基于 OAuth2 的后端服务。

我们将使用 Keycloak 作为 OpenID Provider（身份提供方），可以将其理解为一个负责认证和用户数据管理（角色、用户档案、联系信息等）的用户服务。它是目前最完整的 OpenID Connect（OIDC）实现之一，主要功能包括：

- 单点登录（SSO）与单点登出（后台通道登出）
- 身份代理、社交登录与用户联邦
- 服务器管理及用户账户管理 UI
- 用于程序化控制的管理员 REST API

在回顾 Spring Security 中 OAuth2 的配置选项之后，我们将配置两个不同的 Spring Boot 应用：

- 使用 `oauth2Login` 的有状态客户端
- 使用 `oauth2ResourceServer` 的无状态资源服务器

---

## 2. 使用 Docker 快速启动 Keycloak

本节将启动一个预配置了 Realm 的 Keycloak 服务器。关于如何创建此 Realm，请参见第 6 节。

### 2.1 Docker Compose 文件

在开发环境中沙箱化授权服务器，最简单的方式就是拉取 Keycloak Docker 镜像。我们通过 Docker Compose 文件进行配置：

```yaml
services:
  keycloak:
    container_name: baeldung-keycloak.openid-provider
    image: quay.io/keycloak/keycloak:25.0.1
    command:
      - start-dev
      - --import-realm
    ports:
      - 8080:8080
    volumes:
      - ./keycloak/:/opt/keycloak/data/import/
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_HTTP_PORT: 8080
      KC_HOSTNAME_URL: http://localhost:8080
      KC_HOSTNAME_ADMIN_URL: http://localhost:8080
      KC_HOSTNAME_STRICT_BACKCHANNEL: true
      KC_HTTP_RELATIVE_PATH: /
      KC_HTTP_ENABLED: true
      KC_HEALTH_ENABLED: true
      KC_METRICS_ENABLED: true
    extra_hosts:
      - "host.docker.internal:host-gateway"
    healthcheck:
      test: ['CMD-SHELL', '[ -f /tmp/HealthCheck.java ] || echo "public class HealthCheck { public static void main(String[] args) throws java.lang.Throwable { System.exit(java.net.HttpURLConnection.HTTP_OK == ((java.net.HttpURLConnection)new java.net.URL(args[0]).openConnection()).getResponseCode() ? 0 : 1); } }" > /tmp/HealthCheck.java && java /tmp/HealthCheck.java http://localhost:8080/auth/health/live']
      interval: 5s
      timeout: 5s
      retries: 20
```

### 2.2 使用配套项目的 Realm

配套项目包含一个 JSON 文件，其中定义了我们所需的 Keycloak 对象：

- 一个名为 `baeldung-keycloak` 的 Realm
- 一个名为 `baeldung-keycloak-confidential` 的客户端，密钥（secret）为 `secret`
- 一个映射器，用于将 Realm 角色添加到签发给 `baeldung-keycloak-confidential` 客户端的访问令牌和 ID 令牌中（默认情况下只有访问令牌包含 Realm 角色）
- 一个在 Realm 级别定义的 `NICE` 角色（Keycloak 也支持在客户端级别定义角色）
- 两个用户：`brice`（拥有 NICE 角色）和 `igor`（无 NICE 角色），两者的密码均为 `secret`

Compose 文件中的 `--import-realm` 参数告知 Keycloak 从 `data/import/` 目录下的 JSON 文件中加载对象。

根据 Compose 文件中定义的挂载卷，需要将配套项目中的 `keycloak/baeldung-keycloak-realm.json` 文件复制到与 Compose 文件同级的 `./keycloak/` 目录下。

### 2.3 启动 Docker 容器

为了简化初始配置，上述 Docker Compose 文件使用了 `KEYCLOAK_ADMIN_PASSWORD` 环境变量，在启动容器前需要先设置该变量：

```bash
export KEYCLOAK_ADMIN_PASSWORD=admin
docker compose up -d
```

执行以上命令后，Keycloak 开始启动。当日志中出现包含 `Keycloak 25.0.1 [...] started` 的行时，说明启动完成。

之后，可以通过浏览器访问 `http://localhost:8080`，使用 `admin/admin` 作为凭据登录 Keycloak 管理控制台。

---

## 3. Spring Security 中的 OAuth2

OAuth2 以及 Spring Security 提供的各种配置选项，往往令开发者感到困惑。本节将系统梳理相关概念。

### 3.1 OAuth2 中的角色

OAuth2 中有三个核心参与方：

**授权服务器（Authorization Server）**：负责对资源所有者进行认证，并向客户端签发令牌。本教程中使用 Keycloak 承担此角色。

**客户端（Client）**：驱动整个流程，从授权服务器获取令牌、存储令牌，并使用有效令牌对资源服务器的请求进行授权。当资源所有者是用户时，客户端使用授权码流程（或设备流程，适用于输入能力有限的设备）完成登录，并获取令牌以代表用户向资源服务器发起请求。

本教程中，客户端为配置了 `oauth2Login` 的 Spring 应用以及 Postman。

**资源服务器（Resource Server）**：提供对 REST 资源的安全访问，负责验证令牌的有效性（签发方、过期时间、受众等）并控制客户端的访问权限（客户端作用域、资源所有者角色、资源所有者与被访问资源之间的关系等）。我们将把一个 REST API 配置为资源服务器。

需要特别指出的是，选择何种流程从授权服务器获取令牌，是 OAuth2 **客户端**的职责。因此，用户登录和登出从来都不是资源服务器关心的问题。

### 3.2 oauth2Login

Spring Security 的 `oauth2Login` 配置了授权码流程和刷新令牌流程，同时配置了一个授权客户端仓库（默认存储在 Session 中）以及授权客户端管理器，以便在返回令牌前自动刷新过期令牌。

对于配置了 `oauth2Login` 的 Spring 客户端，请求通过 Session Cookie 来授权。因此，在包含 `oauth2Login` 的 `SecurityFilterChain` Bean 中，必须始终启用 CSRF 防护。

请求成功授权后，安全上下文中的 `Authentication` 类型为 `OAuth2AuthenticationToken`。

如果 Provider 通过 OpenID 自动配置，则 `OAuth2AuthenticationToken` 的 principal 为从 ID 令牌构建的 `OidcUser`；否则，需要额外调用 userinfo 端点来设置 `OAuth2User` 作为 principal。

Spring Security 的权限（authorities）通过 `GrantedAuthoritiesMapper` 或自定义的 `OAuth2UserService` 进行映射。

### 3.3 oauth2ResourceServer

Spring Security 的 `oauth2ResourceServer` 配置了 Bearer 令牌安全机制，提供令牌自省（即不透明令牌）和 JWT 解码两种方式。

对于资源服务器而言，用户状态由令牌的 Claims 承载，Session 可以被禁用，这带来两大好处：

- **可禁用 CSRF 防护**：CSRF 攻击依赖 Session，而此处不使用 Session。
- **资源服务器易于水平扩展**：无论请求被路由到哪个实例，客户端和资源所有者的状态都随 Claims 携带而来。

安全上下文中 `Authentication` 的类型可通过认证转换器（authentication converter）自定义。但默认认证类型、令牌转换方式以及权限解析方式，取决于资源服务器的种类：

- 使用 **JWT 解码器**时，默认认证类型为 `JwtAuthenticationToken`，权限通过 JWT 认证转换器基于访问令牌的 Claims 进行映射。
- 使用**访问令牌自省**时，默认认证类型为 `BearerTokenAuthentication`，权限通过自定义自省器基于自省端点的响应进行解析。

### 3.4 如何选择 oauth2Login 与 oauth2ResourceServer

由上可知，`oauth2Login` 和 `oauth2ResourceServer` 服务于不同的目的。Spring Security 为两者分别提供了不同的 Spring Boot Starter，因为这两者不应出现在同一个 `SecurityFilterChain` Bean 中：

- `oauth2Login` 基于 Session 进行授权，`oauth2ResourceServer` 基于 Bearer 令牌进行授权。
- 由于基于 Session，`oauth2Login` 需要启用 CSRF 防护；而由于无状态的特性，资源服务器不需要。
- 在 `oauth2Login`、使用 JWT 解码器的 `oauth2ResourceServer` 以及使用自省的 `oauth2ResourceServer` 中，权限映射方式和安全上下文中的 `Authentication` 类型各不相同。

由于无状态应用具有良好的可扩展性，我们通常倾向于使用 `oauth2ResourceServer` 来配置 REST 端点。但这要求请求的发起方（或路由方）能够从授权服务器获取令牌，将其存储在某种状态中（Session 或其他方式），在令牌过期时刷新，并在请求中携带访问令牌。

许多 REST 客户端可以做到这一点（如 Spring 的 `RestClient`、`WebClient`，或带 UI 的 Postman），但浏览器本身无法做到，除非借助 Angular、React、Vue 等框架（出于安全考虑，这种方式现在已不再推荐）。

`oauth2Login` 的主要使用场景是：服务端渲染（SSR）的 UI，以及作为 OAuth2 BFF（Backend For Frontend）使用的 Spring Cloud Gateway。但由于其有状态的特性，在进行水平扩展以实现高可用或负载均衡时，需要使用 Spring Session 或智能代理。

### 3.5 组合多种异构请求授权机制

为了对来自不同消费方的异构请求进行授权，有时需要在同一个应用中组合配置 `oauth2Login`、`oauth2ResourceServer`、`x509`、`formLogin`、Basic Auth 等多种机制。例如在 OAuth2 BFF 中，前端请求通过 `oauth2Login` 授权，而 Actuator 端点则通过 `oauth2ResourceServer` 或 Basic Auth 授权。

在这种情况下，应为每种请求授权机制配置单独的 `SecurityFilterChain` Bean，每个 Bean 都用不同的 `@Order` 标注，且除最后一个外，其余都应通过 `securityMatcher` 指定其负责的请求范围。没有 `securityMatcher` 的过滤链将作为兜底默认处理所有未被前面匹配到的请求。

---

## 4. 使用 Thymeleaf 实现登录

我们的第一个使用场景是：使用 OAuth2 和 Spring Boot 构建一个 Thymeleaf 应用，通过 OpenID Provider 对用户进行认证。该示例虽然简单，但完整演示了在有状态应用中使用 Keycloak 实现基于角色的访问控制（RBAC）。

几点说明：

- 在单页应用或移动应用体系下，我们会使用 OAuth2 BFF，其 OAuth2 客户端配置与此类似。
- 我们的 Thymeleaf 应用属于 OAuth2 客户端，因为它使用 `oauth2Login` 和 ID 令牌来构建用户认证，但并不使用访问令牌（不向资源服务器发请求）。如前所述，浏览器与 Spring 后端之间的请求通过 Session Cookie 授权。
- 若要水平扩展，需要在实例间共享 Session（使用 Spring Session）或借助智能代理将同一用户的请求固定路由到同一实例。

### 4.1 依赖项

启用 OAuth2 用户登录最核心的依赖是 `spring-boot-starter-oauth2-client`。由于我们需要创建一个渲染 Thymeleaf 模板的 Servlet 应用，还需要引入 `spring-boot-starter-web` 和 `spring-boot-starter-thymeleaf`。

在 `pom.xml` 中添加如下依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
    <version>3.5.7</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
    <version>3.5.7</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.5.7</version>
</dependency>
```

以上依赖足以满足运行时需求。如果还需要在测试中模拟认证来验证访问控制，则需要以 `test` 作用域引入 `spring-security-test`。

### 4.2 Provider 与注册配置

由于 Keycloak 在 `${issuer-uri}/.well-known/openid-configuration` 暴露其 OpenID 配置，且 Spring Security 可以基于 OpenID 配置自动配置 Provider，因此只需定义 `issuer-uri` 即可：

```properties
spring.security.oauth2.client.provider.baeldung-keycloak.issuer-uri=http://localhost:8080/realms/baeldung-keycloak
```

接下来配置使用上述 Provider 的客户端注册：

```properties
spring.security.oauth2.client.registration.keycloak.provider=baeldung-keycloak
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.keycloak.client-id=baeldung-keycloak-confidential
spring.security.oauth2.client.registration.keycloak.client-secret=secret
spring.security.oauth2.client.registration.keycloak.scope=openid
```

`client-id` 的值与 Keycloak 管理控制台中声明的 Confidential 客户端名称一致。`client-secret` 可从 Credentials 选项卡获取，使用导入的 Realm 时密钥为 `secret`。

### 4.3 将 Keycloak Realm 角色映射为 Spring Security 权限

在配置了 `oauth2Login` 的过滤链中，有多种方式可以映射权限，其中最简便的是暴露一个 `GrantedAuthoritiesMapper` Bean，这也是我们这里采用的方案。

首先，定义一个 Bean，负责从 Keycloak Claims 集合（可能来自 ID 令牌或访问令牌的 Payload，也可能来自 userinfo 或自省响应体）中提取权限：

```java
interface AuthoritiesConverter extends Converter<Map<String, Object>, Collection<GrantedAuthority>> {}

@Bean
AuthoritiesConverter realmRolesAuthoritiesConverter() {
    return claims -> {
        var realmAccess = Optional.ofNullable((Map<String, Object>) claims.get("realm_access"));
        var roles = realmAccess.flatMap(map -> Optional.ofNullable((List<String>) map.get("roles")));
        return roles.map(List::stream)
            .orElse(Stream.empty())
            .map(SimpleGrantedAuthority::new)
            .map(GrantedAuthority.class::cast)
            .toList();
    };
}
```

由于 JVM 的泛型类型擦除，以及应用上下文中可能存在多个输入输出类型各异的 `Converter` Bean，自定义的 `AuthoritiesConverter` 接口可以帮助 Bean 工厂更准确地查找 `Converter<Map<String, Object>, Collection<GrantedAuthority>>` 类型的 Bean。

由于我们使用 `issuer-uri` 自动配置了 OIDC Provider，`GrantedAuthoritiesMapper` 接收到的输入是 `OidcUserAuthority` 实例：

```java
@Bean
GrantedAuthoritiesMapper authenticationConverter(AuthoritiesConverter authoritiesConverter) {
    return (authorities) -> authorities.stream()
        .filter(authority -> authority instanceof OidcUserAuthority)
        .map(OidcUserAuthority.class::cast)
        .map(OidcUserAuthority::getIdToken)
        .map(OidcIdToken::getClaims)
        .map(authoritiesConverter::convert)
        .flatMap(roles -> roles.stream())
        .collect(Collectors.toSet());
}
```

注意这里如何注入并使用上面定义的 `authoritiesConverter` Bean。

### 4.4 组装 SecurityFilterChain Bean

以下是完整的 `SecurityFilterChain` 配置：

```java
@Bean
SecurityFilterChain clientSecurityFilterChain(
    HttpSecurity http,
    ClientRegistrationRepository clientRegistrationRepository) throws Exception {

    http.oauth2Login(Customizer.withDefaults());
    http.logout((logout) -> {
        var logoutSuccessHandler =
            new OidcClientInitiatedLogoutSuccessHandler(clientRegistrationRepository);
        logoutSuccessHandler.setPostLogoutRedirectUri("{baseUrl}/");
        logout.logoutSuccessHandler(logoutSuccessHandler);
    });
    http.authorizeHttpRequests(requests -> {
        requests.requestMatchers("/", "/favicon.ico").permitAll();
        requests.requestMatchers("/nice").hasAuthority("NICE");
        requests.anyRequest().denyAll();
    });
    return http.build();
}
```

逐块解析如下：

`oauth2Login()` 方法将 `OAuth2LoginAuthenticationFilter` 加入过滤链，该过滤器拦截请求，应用授权码流程和刷新令牌流程所需的逻辑，并将令牌存储到 Session 中。

要理解 OAuth2 登出的工作原理，需要记住：用户至少拥有两个独立的 Session——一个在授权服务器（本例中为 Keycloak），另一个在每个配置了 `oauth2Login` 的客户端上。要完整登出，必须终止所有 Session。

OpenID 标准定义了多种实现方式，我们这里聚焦于 **RP 发起的登出（RP-Initiated Logout）**，通过 `OidcClientInitiatedLogoutSuccessHandler` 进行配置。

在此流程中，用户代理（浏览器）首先从依赖方（RP，即我们配置了 `oauth2Login` 的 Spring 应用）登出，关闭该侧的 Session。RP 返回一个重定向响应，指向 OpenID Provider（OP），携带与待关闭 Session 关联的 ID 令牌以及登出后的跳转 URL（本例为 Thymeleaf 的首页）。OP 随后关闭其 Session，并将用户代理重定向到指定 URL。

最后，定义访问控制规则：
- 首页需允许匿名访问，用户才能从首页发起登录。
- `/nice` 路径仅允许拥有 `NICE` 权限的已认证用户访问。

### 4.5 Thymeleaf 页面

我们使用 Thymeleaf 创建两个页面：

- `index.html`：根据用户的登录状态显示登录或登出按钮。同时包含跳转到 `/nice` 的按钮，出于良好的用户体验，该按钮仅对拥有 `NICE` 权限的已认证用户可见。
- `nice.html`：仅包含静态内容。

`index.html` 包含条件逻辑，可以使用用户 Session 中的值：

```html
<div class="container">
    <h1 class="form-signin-heading">Baeldung: Keycloak &amp; Spring Boot</h1>
    <p>欢迎使用由 Spring Boot 提供服务、并通过 Keycloak 保护的 Thymeleaf UI！</p>
    <div th:if="${!isAuthenticated}">
        <a href="/oauth2/authorization/keycloak">
            <button type="button" class="btn btn-lg btn-primary btn-block">登录</button>
        </a>
    </div>
    <div th:if="${isAuthenticated}">
        <p>你好，<span th:utext="${name}">..!..</span>！</p>
        <a href="/logout">
            <button type="button" class="btn btn-lg btn-primary">登出</button>
        </a>
        <a th:if="${isNice}" href="/nice">
            <button type="button" class="btn btn-lg btn-primary">进入 NICE 用户专属页面</button>
        </a>
    </div>
</div>
```

### 4.6 控制器

`UiController` 负责构建首页的 Model（用户名、`isAuthenticated` 和 `isNice` 标志位），并解析 Thymeleaf 模板：

```java
@Controller
public class UiController {
    @GetMapping("/")
    public String getIndex(Model model, Authentication auth) {
        model.addAttribute("name",
            auth instanceof OAuth2AuthenticationToken oauth && oauth.getPrincipal() instanceof OidcUser oidc
                ? oidc.getPreferredUsername()
                : "");
        model.addAttribute("isAuthenticated",
            auth != null && auth.isAuthenticated());
        model.addAttribute("isNice",
            auth != null && auth.getAuthorities().stream().anyMatch(authority ->
                Objects.equals("NICE", authority.getAuthority())));
        return "index.html";
    }

    @GetMapping("/nice")
    public String getNice(Model model, Authentication auth) {
        return "nice.html";
    }
}
```

### 4.7 运行 Thymeleaf 应用

现在可以通过常用 IDE 启动应用，也可以在 Maven 父项目的上下文中执行以下命令：

```bash
./mvnw -pl spring-boot-mvc-client spring-boot:run
```

登录状态）。

以 `brice` 登录后（而非以 `igor` 登录时），才能看到跳转到 NICE 用户专属页面的按钮。

在 RP 发起的登出之后再次尝试登录，系统应当要求重新输入凭据。

如果从 Java 配置中移除 logout 部分，则从 Spring 应用登出时 Keycloak 的 Session 不会结束，此时再次尝试登录将会静默完成——Keycloak 不会显示登录表单，因为从它的角度来看用户仍处于登录状态。

---

## 5. 使用 JWT 解码器的 REST API

接下来，我们继续介绍使用 JWT 格式 Bearer 访问令牌的无状态 REST API 配置。

### 5.1 依赖项

这次最核心的依赖是 `spring-boot-starter-oauth2-resource-server`，以及 Servlet 应用所需的 `spring-boot-starter-web`：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    <version>3.5.7</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.5.7</version>
</dependency>
```

以上依赖足以满足运行时需求。如需在测试中模拟认证，同样需要以 `test` 作用域引入 `spring-security-test`。

### 5.2 配置 JWT 解码器

在资源服务器中验证访问令牌，可以选择 JWT 解码器或令牌自省两种方式。JWT 解码器要求授权服务器以 JWT 格式签发访问令牌，但效率远高于自省方式——后者需要资源服务器在每次请求时都向授权服务器发起调用，不仅引入额外延迟，还可能给授权服务器带来压力。

Keycloak 签发的访问令牌本身就是 JWT 格式，使用 Spring Boot 时，只需一个配置项即可完成资源服务器的 JWT 解码器配置：

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8080/realms/baeldung-keycloak
```

### 5.3 将 Keycloak Realm 角色映射为 Spring Security 权限

对于使用 JWT 解码器的资源服务器，需要配置认证转换器（authentication converter）。

我们可以复用前面定义的 `authoritiesConverter` Bean：

```java
@Bean
JwtAuthenticationConverter authenticationConverter(AuthoritiesConverter authoritiesConverter) {
    var authenticationConverter = new JwtAuthenticationConverter();
    authenticationConverter.setJwtGrantedAuthoritiesConverter(jwt -> {
        return authoritiesConverter.convert(jwt.getClaims());
    });
    return authenticationConverter;
}
```

注意，这次传入的是**访问令牌**的 Claims，而非 ID 令牌的 Claims。

### 5.4 组装 SecurityFilterChain Bean

现在可以定义资源服务器的安全过滤链：

```java
@Bean
SecurityFilterChain resourceServerSecurityFilterChain(
    HttpSecurity http,
    Converter<Jwt, AbstractAuthenticationToken> authenticationConverter) throws Exception {

    http.oauth2ResourceServer(resourceServer -> {
        resourceServer.jwt(jwtDecoder -> {
            jwtDecoder.jwtAuthenticationConverter(authenticationConverter);
        });
    });
    http.sessionManagement(sessions -> {
        sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }).csrf(csrf -> {
        csrf.disable();
    });
    http.authorizeHttpRequests(requests -> {
        requests.requestMatchers("/me").authenticated();
        requests.anyRequest().denyAll();
    });
    return http.build();
}
```

代码说明：

- 第一块：启用资源服务器配置，使用 JWT 解码器和自定义认证转换器，将 Keycloak 角色转换为 Spring Security 权限。
- 第二块：完全禁用 Session 和 CSRF 防护。
- 第三块：定义访问控制规则——仅携带有效 Bearer 令牌的请求才能访问 `/me` 端点，拒绝对任何其他资源的访问。

### 5.5 REST 控制器

为演示 Keycloak 角色在资源服务器中的映射效果，我们暴露一个端点，返回安全上下文中 `JwtAuthenticationToken` 的部分信息：

```java
@RestController
public class MeController {
    @GetMapping("/me")
    public UserInfoDto getGreeting(JwtAuthenticationToken auth) {
        return new UserInfoDto(
            auth.getToken().getClaimAsString(StandardClaimNames.PREFERRED_USERNAME),
            auth.getAuthorities().stream().map(GrantedAuthority::getAuthority).toList());
    }

    public static record UserInfoDto(String name, List<String> roles) {}
}
```

将 `auth` 直接转型为 `JwtAuthenticationToken` 是安全的，因为访问已限制为 `isAuthenticated()`，且我们的认证转换器返回的正是 `JwtAuthenticationToken`。如果路由设置为 `permitAll()`，则还需处理未授权请求（不携带 Bearer 令牌）对应的 `AnonymousAuthenticationToken` 情况。

### 5.6 测试 REST API

现在可以运行并测试 REST API 了。与 Thymeleaf 应用类似，可以通过 IDE 启动，也可以在 Maven 父项目上下文中执行：

```bash
./mvnw -pl spring-boot-resource-server spring-boot:run
```

然后，在 Postman 中从 Keycloak 获取访问令牌：打开集合或请求的 Authorization 选项卡，选择 OAuth2，并按照我们在 Keycloak 和 Spring 配置中设置的值（以及从 OpenID 配置中获取的值）填写表单。

![Postman OAuth2 配置](<images/Getting started with Keycloak & Spring Boot/screenshot_01.jpg>)

最后，向 `http://localhost:8082/me` 发送 GET 请求。响应应返回一个 JSON Payload，包含当前用户的 Keycloak 登录名和 Realm 角色。

![/me 接口响应](<images/Getting started with Keycloak & Spring Boot/screenshot_02.jpg>)

---

## 6. 创建 Keycloak Realm

在本教程中，我们将 Keycloak 对象沙箱化在一个自定义 Realm 中，并在创建 Docker 容器时将其导入。这对于新开发者入职团队或尝试不同配置场景非常有用。

本节将详细介绍这些对象的初始创建过程。

### 6.1 创建 Realm

点击左上角的 **Create Realm** 按钮：

![Create Realm 入口](<images/Getting started with Keycloak & Spring Boot/screenshot_03.jpg>)

在下一个界面将 Realm 命名为 `baeldung-keycloak`：

![Create Realm 命名](<images/Getting started with Keycloak & Spring Boot/screenshot_04.jpg>)

点击 **Create** 按钮后将跳转至该 Realm 的详情页。

后续所有操作均在此 `baeldung-keycloak` Realm 中进行。

### 6.2 创建用于用户认证的 Confidential 客户端

进入 **Clients** 页面，创建一个名为 `baeldung-keycloak-confidential` 的新客户端：

![Create Client 第一步](<images/Getting started with Keycloak & Spring Boot/screenshot_05.jpg>)

在下一个界面，确保启用了 **Client authentication**（这将使客户端变为 "Confidential" 类型），并且只勾选了 **Standard flow**（即启用授权码流程和刷新令牌流程）：

![Capability Config](<images/Getting started with Keycloak & Spring Boot/screenshot_06.jpg>)

最后，为客户端配置允许的重定向 URI 和来源（Origin）：

![Login Settings](<images/Getting started with Keycloak & Spring Boot/screenshot_07.jpg>)

由于配置了 `oauth2Login` 的 Spring Boot 客户端应用运行在 8081 端口，且注册 ID 为 `keycloak`，因此进行如下设置：

- **Valid redirect URIs**：`http://localhost:8081/login/oauth2/code/keycloak`
- **Valid post-logout URIs**：`http://localhost:8081/`（Thymeleaf 首页）
- **Web origins**：`+`（允许所有来自 Valid redirect URIs 中的来源）

### 6.3 配置哪些 JSON Payload 包含 Realm 角色

为确保 Keycloak 角色被添加到 Spring 应用所使用的各类 Payload 中，需要进行以下配置：

1. 在客户端详情中，进入 **Client scopes** 选项卡。

   ![Client Scopes 选项卡](<images/Getting started with Keycloak & Spring Boot/screenshot_08.jpg>)

2. 点击 `baeldung-keycloak-confidential-dedicated` Scope 进入其配置详情。

3. 点击 **Add predefined mapper** 按钮，选择 **realm roles** 映射器：

   ![添加 Realm Roles Mapper](<images/Getting started with Keycloak & Spring Boot/screenshot_09.jpg>)

4. 编辑该映射器，可以选择在哪些 JSON Payload 中包含 Realm 角色：

   ![编辑 Realm Roles Mapper](<images/Getting started with Keycloak & Spring Boot/screenshot_10.jpg>)

   - **access tokens**：供使用 JWT 解码器的资源服务器使用
   - **introspection**：供使用令牌自省（即 Spring Security 中的 opaque token）的资源服务器使用
   - **ID tokens**：供使用 `oauth2Login` 且配置了 OpenID（通过 `issuer-uri` 自动配置 Provider）的客户端使用
   - **userinfo**：供使用 `oauth2Login` 但未配置 OpenID（`issuer-uri` 为空，Provider 其他属性手动配置）的客户端使用

由于我们在 OpenID 客户端（Thymeleaf 应用，使用 `oauth2Login`）和使用 JWT 解码器的资源服务器中都实现了基于角色的访问控制（RBAC），因此需要确保 Realm 角色被同时添加到访问令牌和 ID 令牌中。

### 6.4 创建 Realm 角色

在 Keycloak 中，可以在整个 Realm 范围内或针对特定客户端定义角色，并将其分配给用户。本教程聚焦于 Realm 角色。

进入 **Realm roles** 页面，创建名为 `NICE` 的角色：

![创建 NICE 角色](<images/Getting started with Keycloak & Spring Boot/screenshot_11.jpg>)

### 6.5 创建用户并分配 Realm 角色

进入 **Users** 页面，创建两个用户：`brice`（分配 NICE 角色）和 `igor`（不分配任何 Realm 角色）。

1. 创建新用户 `brice`，点击 **Create** 按钮后可查看用户详情。

   ![创建用户 brice](<images/Getting started with Keycloak & Spring Boot/screenshot_12.jpg>)

2. 进入 **Credentials** 选项卡，为其设置密码。

   ![设置密码](<images/Getting started with Keycloak & Spring Boot/screenshot_13.jpg>)

3. 进入 **Role Mappings** 选项卡，分配 `NICE` 角色（注意使用 **Filter by realm roles** 下拉菜单筛选）。

   ![分配 NICE 角色](<images/Getting started with Keycloak & Spring Boot/screenshot_14.jpg>)

对 `igor` 重复以上步骤，但跳过角色分配环节。

### 6.6 导出 Realm

完成 Realm 配置后，可通过以下命令导出（在 Docker Desktop 中，使用运行中容器的 **Exec** 选项卡执行）：

```bash
cd /opt/keycloak/bin/
sh ./kc.sh export --dir /tmp/keycloak/ --users realm_file
```

随后可从 **Files** 选项卡中获取每个 Realm 对应的 JSON 文件。

---

## 7. 总结

本文介绍了如何使用 Spring Boot 和 Keycloak 配置基于 OAuth2 的后端服务。

除了使用 Docker 进行最小化 Keycloak 部署之外，我们还演示了如何导入和导出 Realm。

同时，在梳理了 Spring 应用中 OAuth2 的各种配置选项之后，我们分别将 Spring 应用配置为：使用 `oauth2Login` 的有状态 OAuth2 客户端，以及无状态的 OAuth2 资源服务器。

完整代码可在 GitHub 上获取。
