# Spring OAuth2 访问控制测试指南

> 原文来源：Baeldung

## 1. 概述

本教程将探讨在使用 OAuth2 安全机制的 Spring 应用中，如何通过模拟身份来测试访问控制规则。我们将使用来自 `spring-security-test` 和 `spring-addons` 的 MockMvc 请求后处理器（request post-processors）、WebTestClient 变异器（mutators）以及测试注解。

---

## 2. 为什么使用 Spring-Addons？

在 OAuth2 领域，`spring-security-test` 仅提供了请求后处理器和变异器，分别需要依附于 MockMvc 或 WebTestClient 请求的上下文中使用。对于 `@Controller` 的测试来说这完全够用，但在测试 `@Service` 或 `@Repository` 上的方法安全（`@PreAuthorize`、`@PostFilter` 等）时就会遇到问题。

使用 `@WithJwt` 或 `@WithOidcLogin` 等注解，我们可以在对任意 `@Component` 进行单元测试时模拟安全上下文，适用于 Servlet 和响应式应用。这正是我们在部分测试中使用 `spring-addons-oauth2-test` 的原因——它为大多数 Spring OAuth2 `Authentication` 实现类型提供了对应的测试注解。

---

## 3. 测试目标

配套 GitHub 仓库包含两个资源服务器，具有以下共同特性：

- 使用 JWT 解码器进行安全保护（而非不透明令牌自省）
- 访问 `/secured-route` 和 `/secured-method` 需要 `ROLE_AUTHORIZED_PERSONNEL` 权限
- 认证缺失或无效（过期、Issuer 错误等）时返回 401，访问被拒绝（缺少角色）时返回 403
- 使用 Java 配置定义访问控制（Servlet 应用使用 `requestMatcher`，响应式应用使用 `pathMatcher`）以及方法安全
- 使用安全上下文中 `Authentication` 的数据来构建响应体

为了体现 Servlet 与响应式测试 API 之间的细微差异，两者各有一个实现。

本文将重点测试单元测试和集成测试中的访问控制规则，验证根据模拟用户身份，HTTP 响应状态码是否符合预期，或在测试 `@Service`、`@Repository` 等受 `@PreAuthorize`、`@PostFilter` 保护的非 `@Controller` 组件时是否抛出预期异常。

所有测试均无需启动授权服务器即可通过。如需启动被测资源服务器并通过 Postman 等工具查询，则需要一个运行中的授权服务器。配套项目提供了一个 Docker Compose 文件，可快速启动本地 Keycloak 实例：

- 管理控制台：`http://localhost:8080/admin/master/console/#/baeldung`
- 管理员账户：`admin / admin`
- 已预创建 `baeldung` Realm，包含一个机密客户端（`baeldung_confidential / secret`）和两个用户（`authorized` 和 `forbidden`，密码均为 `secret`）

---

## 4. 使用模拟身份进行单元测试

"单元测试"是指对单个 `@Component` 进行隔离测试，其他依赖均被 Mock。被测 `@Component` 可以是 `@WebMvcTest` 或 `@WebFluxTest` 中的 `@Controller`，也可以是普通 JUnit 测试中任意受保护的 `@Service`、`@Repository` 等。

MockMvc 和 WebTestClient 会忽略 `Authorization` 请求头，无需提供有效的访问令牌。当然，我们也可以在每个测试开始时手动实例化或 Mock 某个 `Authentication` 实现并创建安全上下文，但这样做过于繁琐。取而代之，我们将使用 `spring-security-test` 的 MockMvc 请求后处理器、WebTestClient 变异器，或 spring-addons 的注解，来向测试安全上下文中填充所需的模拟 `Authentication` 实例。

这里先用 `@WithMockUser` 举例，它会构建一个 `UsernamePasswordAuthenticationToken` 实例。这在 OAuth2 环境中经常会引发问题，因为 OAuth2 运行时配置会将其他类型的 `Authentication` 放入安全上下文：

- `JwtAuthenticationToken`：使用 JWT 解码器的资源服务器
- `BearerTokenAuthentication`：使用访问令牌自省（opaqueToken）的资源服务器
- `OAuth2AuthenticationToken`：使用 `oauth2Login` 的客户端
- 如果自定义认证转换器返回了其他类型，则完全取决于实现

从技术上说，OAuth2 认证转换器确实可以返回 `UsernamePasswordAuthenticationToken` 并在测试中使用 `@WithMockUser`，但这是非常不自然的选择，本文不做讨论。

### 4.1 重要说明

MockMvc 后处理器和 WebTestClient 变异器**不使用**安全配置中定义的 Bean 来构建测试用 `Authentication` 实例。因此，通过 `SecurityMockMvcRequestPostProcessors.jwt()` 或 `SecurityMockServerConfigurers.mockJwt()` 设置 OAuth2 Claims，**不会**对认证名称和权限产生任何影响，必须手动使用专用方法来设置名称和权限。

相比之下，spring-addons 注解背后的工厂会扫描测试上下文，若找到认证转换器则会使用它。因此在使用 `@WithJwt` 时，需要将自定义的 `JwtAuthenticationConverter` 作为 Bean 暴露出来（而非在安全配置中以 Lambda 内联方式定义）：

```java
@Configuration
@EnableMethodSecurity
@EnableWebSecurity
static class SecurityConf {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http,
        Converter<Jwt, AbstractAuthenticationToken> authenticationConverter) throws Exception {
        http.oauth2ResourceServer(resourceServer ->
            resourceServer.jwt(jwtResourceServer ->
                jwtResourceServer.jwtAuthenticationConverter(authenticationConverter)));
        // ...
    }

    @Bean
    JwtAuthenticationConverter authenticationConverter(
        Converter<Jwt, Collection<GrantedAuthority>> authoritiesConverter) {
        final var authenticationConverter = new JwtAuthenticationConverter();
        authenticationConverter.setPrincipalClaimName(StandardClaimNames.PREFERRED_USERNAME);
        authenticationConverter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return authenticationConverter;
    }
}
```

认证转换器以 `@Bean` 方式暴露，并被显式注入到安全过滤链中。这样，`@WithJwt` 背后的工厂就能用它从 Claims 构建 `Authentication`，与运行时处理真实令牌的方式完全一致。

另外请注意，当认证转换器返回的类型不是 `JwtAuthenticationToken`（或使用令牌自省的资源服务器中的 `BearerTokenAuthentication`）时，只有 spring-addons 的测试注解才能构建出预期类型的 `Authentication`。

### 4.2 测试环境搭建

对于 `@Controller` 单元测试，Servlet 应用应使用 `@WebMvcTest` 注解，响应式应用应使用 `@WebFluxTest` 注解。Spring 会自动注入 MockMvc 或 WebTestClient，由于是控制器单元测试，`MessageService` 将被 Mock。

Servlet 应用中空的 `@Controller` 单元测试骨架如下：

```java
@WebMvcTest(controllers = GreetingController.class)
class GreetingControllerTest {
    @MockBean
    MessageService messageService;

    @Autowired
    MockMvc mockMvc;

    // ...
}
```

响应式应用中对应的骨架如下：

```java
@WebFluxTest(controllers = GreetingController.class)
class GreetingControllerTest {
    private static final AnonymousAuthenticationToken ANONYMOUS =
        new AnonymousAuthenticationToken("anonymous", "anonymousUser",
            AuthorityUtils.createAuthorityList("ROLE_ANONYMOUS"));

    @MockBean
    MessageService messageService;

    @Autowired
    WebTestClient webTestClient;

    // ...
}
```

下面来看如何断言 HTTP 状态码是否符合规格要求。

### 4.3 使用 MockMvc 后处理器进行单元测试

要向测试安全上下文填充 `JwtAuthenticationToken`（这是使用 JWT 解码器的资源服务器的默认 `Authentication` 类型），需要对 MockMvc 请求使用 `jwt` 后处理器：

```java
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors;
```

以下是几个使用 MockMvc 发起请求、并根据端点和模拟 `Authentication` 验证响应状态码的示例：

```java
@Test
void givenRequestIsAnonymous_whenGetGreet_thenUnauthorized() throws Exception {
    mockMvc.perform(get("/greet")
        .with(SecurityMockMvcRequestPostProcessors.anonymous()))
        .andExpect(status().isUnauthorized());
}
```

上面的测试确保匿名请求无法获取问候信息，且正确返回 401。

以下是根据端点安全规则和测试 `JwtAuthenticationToken` 所赋予的权限，请求分别得到 200 OK 或 403 Forbidden 的示例：

```java
@Test
void givenUserIsGrantedWithRoleAuthorizedPersonnel_whenGetSecuredRoute_thenOk()
    throws Exception {
    var secret = "Secret!";
    when(messageService.getSecret()).thenReturn(secret);
    mockMvc.perform(get("/secured-route")
        .with(SecurityMockMvcRequestPostProcessors.jwt()
            .authorities(new SimpleGrantedAuthority("ROLE_AUTHORIZED_PERSONNEL"))))
        .andExpect(status().isOk())
        .andExpect(content().string(secret));
}

@Test
void givenUserIsNotGrantedWithRoleAuthorizedPersonnel_whenGetSecuredRoute_thenForbidden()
    throws Exception {
    mockMvc.perform(get("/secured-route")
        .with(SecurityMockMvcRequestPostProcessors.jwt()
            .authorities(new SimpleGrantedAuthority("admin"))))
        .andExpect(status().isForbidden());
}
```

### 4.4 使用 WebTestClient 变异器进行单元测试

在响应式资源服务器中，安全上下文中的 `Authentication` 类型与 Servlet 相同，都是 `JwtAuthenticationToken`。因此，使用 `mockJwt` 变异器：

```java
import org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers;
```

与 MockMvc 后处理器不同，WebTestClient 没有匿名变异器。不过，可以手动定义一个匿名 `Authentication` 实例，并通过通用的 `mockAuthentication` 变异器使用：

```java
private static final AnonymousAuthenticationToken ANONYMOUS =
    new AnonymousAuthenticationToken("anonymous", "anonymousUser",
        AuthorityUtils.createAuthorityList("ROLE_ANONYMOUS"));

@Test
void givenRequestIsAnonymous_whenGetGreet_thenUnauthorized() throws Exception {
    webTestClient.mutateWith(SecurityMockServerConfigurers.mockAuthentication(ANONYMOUS))
        .get().uri("/greet")
        .exchange()
        .expectStatus().isUnauthorized();
}

@Test
void givenUserIsAuthenticated_whenGetGreet_thenOk() throws Exception {
    var greeting = "Whatever the service returns";
    when(messageService.greet()).thenReturn(Mono.just(greeting));
    webTestClient.mutateWith(SecurityMockServerConfigurers.mockJwt()
        .authorities(List.of(
            new SimpleGrantedAuthority("admin"),
            new SimpleGrantedAuthority("ROLE_AUTHORIZED_PERSONNEL")))
        .jwt(jwt -> jwt.claim(StandardClaimNames.PREFERRED_USERNAME, "ch4mpy")))
        .get().uri("/greet")
        .exchange()
        .expectStatus().isOk()
        .expectBody(String.class).isEqualTo(greeting);
    verify(messageService, times(1)).greet();
}

@Test
void givenUserIsNotGrantedWithRoleAuthorizedPersonnel_whenGetSecuredRoute_thenForbidden()
    throws Exception {
    webTestClient.mutateWith(mockJwt()
        .authorities(new SimpleGrantedAuthority("admin")))
        .get().uri("/secured-route")
        .exchange()
        .expectStatus().isForbidden();
}
```

### 4.5 使用 Spring-Addons 注解对控制器进行单元测试

Spring-Addons 的测试注解在 Servlet 和响应式应用中的使用方式完全一致。

首先添加 `spring-addons-oauth2-test` 依赖：

```xml
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-oauth2-test</artifactId>
    <version>7.6.12</version>
    <scope>test</scope>
</dependency>
```

该库提供了若干注解，覆盖以下使用场景：

- **`@WithMockAuthentication`**：在测试基于角色的访问控制时通常已足够，可接受权限列表参数，也可指定用户名和 `Authentication`、`Principal` 的实现类型。
- **`@WithJwt`**：用于测试使用 JWT 解码器的资源服务器。它依赖一个认证工厂，该工厂从安全配置中获取 `Converter<Jwt, ? extends AbstractAuthenticationToken>`（响应式应用中为 `Converter<Jwt, ? extends Mono<? extends AbstractAuthenticationToken>>`），并读取测试 classpath 上的 JSON 文件作为令牌 Payload。这样可以完全控制 Claims，并获得与运行时处理相同 JWT Payload 时一致的 `Authentication` 实例。
- **`@WithOpaqueToken`**：与 `@WithJwt` 类似，用于使用令牌自省的资源服务器，依赖 `OpaqueTokenAuthenticationConverter`（或其响应式版本）。
- **`@WithOAuth2Login` / `@WithOidcLogin`**：用于测试使用 `oauth2Login` 的 OAuth2 客户端。

在编写测试之前，先在测试资源目录下定义一些 JSON 文件，用来模拟代表性用户（测试人物角色）的访问令牌 JSON Payload（或自省响应）。可以使用 [https://jwt.io](https://jwt.io) 等工具复制真实令牌的 Payload。

`ch4mpy` 是我们的测试用户，拥有 `AUTHORIZED_PERSONNEL` 角色：

```json
{
    "iss": "https://localhost:8443/realms/master",
    "sub": "281c4558-550c-413b-9972-2d2e5bde6b9b",
    "iat": 1695992542,
    "exp": 1695992642,
    "preferred_username": "ch4mpy",
    "realm_access": {
        "roles": [
            "admin",
            "ROLE_AUTHORIZED_PERSONNEL"
        ]
    },
    "email": "ch4mp@c4-soft.com",
    "scope": "openid email"
}
```

第二个用户没有 `AUTHORIZED_PERSONNEL` 角色：

```json
{
    "iss": "https://localhost:8443/realms/master",
    "sub": "2d2e5bde6b9b-550c-413b-9972-281c4558",
    "iat": 1695992551,
    "exp": 1695992651,
    "preferred_username": "tonton-pirate",
    "realm_access": {
        "roles": [
            "uncle",
            "skipper"
        ]
    },
    "email": "tonton-pirate@c4-soft.com",
    "scope": "openid email"
}
```

现在可以将身份模拟从测试方法体中移除，改为用注解装饰测试方法。为演示目的，下面同时使用了 `@WithMockAuthentication` 和 `@WithJwt`，实际测试中选一种即可。通常的选择原则是：只需指定权限或用户名时用前者，需要精细控制多个 Claims 时用后者：

```java
@Test
@WithAnonymousUser
void givenRequestIsAnonymous_whenGetSecuredMethod_thenUnauthorized() throws Exception {
    api.perform(get("/secured-method"))
        .andExpect(status().isUnauthorized());
}

@Test
@WithMockAuthentication({ "admin", "ROLE_AUTHORIZED_PERSONNEL" })
void givenUserIsGrantedWithRoleAuthorizedPersonnel_whenGetSecuredMethod_thenOk()
    throws Exception {
    final var secret = "Secret!";
    when(messageService.getSecret()).thenReturn(secret);
    api.perform(get("/secured-method"))
        .andExpect(status().isOk())
        .andExpect(content().string(secret));
}

@Test
@WithMockAuthentication({ "admin" })
void givenUserIsNotGrantedWithRoleAuthorizedPersonnel_whenGetSecuredMethod_thenForbidden()
    throws Exception {
    api.perform(get("/secured-method"))
        .andExpect(status().isForbidden());
}

@Test
@WithJwt("ch4mpy.json")
void givenUserIsCh4mpy_whenGetSecuredMethod_thenOk() throws Exception {
    final var secret = "Secret!";
    when(messageService.getSecret()).thenReturn(secret);
    api.perform(get("/secured-method"))
        .andExpect(status().isOk())
        .andExpect(content().string(secret));
}

@Test
@WithJwt("tonton-pirate.json")
void givenUserIsTontonPirate_whenGetSecuredMethod_thenForbidden() throws Exception {
    api.perform(get("/secured-method"))
        .andExpect(status().isForbidden());
}
```

注解与 BDD 范式高度契合：前置条件（Given）位于测试上下文（装饰测试的注解）中，测试方法体中只包含被测代码的执行（When）和结果断言（Then）。

### 4.6 对受方法安全保护的 @Service 或 @Repository 进行单元测试

测试 `@Controller` 时，在 MockMvc 后处理器（或 WebTestClient 变异器）与注解之间的选择主要是团队偏好问题，但在对 `MessageService::getSecret` 的访问控制进行单元测试时，`spring-security-test` 不再适用，必须使用 spring-addons 注解。

JUnit 测试的搭建要点：

- 使用 `@ExtendWith(SpringExtension.class)` 激活 Spring 自动装配
- 导入并自动装配 `MessageService` 以获得带切面增强的实例
- 如果使用 `@WithJwt`，需要同时导入包含 `JwtAuthenticationConverter` 的配置类以及 `AuthenticationFactoriesTestConf`；否则，仅在测试类上添加 `@EnableMethodSecurity` 即可

我们将断言：每当用户缺少 `ROLE_AUTHORIZED_PERSONNEL` 权限时，`MessageService` 会抛出异常。

以下是 Servlet 应用中对 `@Service` 的完整单元测试：

```java
@ExtendWith(SpringExtension.class)
@TestInstance(Lifecycle.PER_CLASS)
@Import({ MessageService.class, SecurityConf.class })
@ImportAutoConfiguration(AuthenticationFactoriesTestConf.class)
class MessageServiceUnitTest {
    @Autowired
    MessageService messageService;

    @MockBean
    JwtDecoder jwtDecoder;

    @Test
    void givenSecurityContextIsNotSet_whenGreet_thenThrowsAuthenticationCredentialsNotFoundException() {
        assertThrows(AuthenticationCredentialsNotFoundException.class,
            () -> messageService.getSecret());
    }

    @Test
    @WithAnonymousUser
    void givenUserIsAnonymous_whenGreet_thenThrowsAccessDeniedException() {
        assertThrows(AccessDeniedException.class, () -> messageService.getSecret());
    }

    @Test
    @WithJwt("ch4mpy.json")
    void givenUserIsCh4mpy_whenGreet_thenReturnGreetingWithPreferredUsernameAndAuthorities() {
        assertEquals("Hello ch4mpy! You are granted with [admin, ROLE_AUTHORIZED_PERSONNEL].",
            messageService.greet());
    }

    @Test
    @WithMockUser(authorities = { "admin", "ROLE_AUTHORIZED_PERSONNEL" }, username = "ch4mpy")
    void givenSecurityContextIsPopulatedWithUsernamePasswordAuthenticationToken_whenGreet_thenThrowsClassCastException() {
        assertThrows(ClassCastException.class, () -> messageService.greet());
    }
}
```

响应式应用中的 `@Service` 单元测试与之类似：

```java
@ExtendWith(SpringExtension.class)
@TestInstance(Lifecycle.PER_CLASS)
@Import({ MessageService.class, SecurityConf.class })
@ImportAutoConfiguration(AuthenticationFactoriesTestConf.class)
class MessageServiceUnitTest {
    @Autowired
    MessageService messageService;

    @MockBean
    ReactiveJwtDecoder jwtDecoder;

    @Test
    void givenSecurityContextIsEmpty_whenGreet_thenThrowsAuthenticationCredentialsNotFoundException() {
        assertThrows(AuthenticationCredentialsNotFoundException.class,
            () -> messageService.greet().block());
    }

    @Test
    @WithAnonymousUser
    void givenUserIsAnonymous_whenGreet_thenThrowsClassCastException() {
        assertThrows(ClassCastException.class, () -> messageService.greet().block());
    }

    @Test
    @WithJwt("ch4mpy.json")
    void givenUserIsCh4mpy_whenGreet_thenReturnGreetingWithPreferredUsernameAndAuthorities() {
        assertEquals("Hello ch4mpy! You are granted with [admin, ROLE_AUTHORIZED_PERSONNEL].",
            messageService.greet().block());
    }

    @Test
    @WithMockUser(authorities = { "admin", "ROLE_AUTHORIZED_PERSONNEL" }, username = "ch4mpy")
    void givenSecurityContextIsPopulatedWithUsernamePasswordAuthenticationToken_whenGreet_thenThrowsClassCastException() {
        assertThrows(ClassCastException.class, () -> messageService.greet().block());
    }
}
```

### 4.7 JUnit 5 参数化测试 @ParameterizedTest

JUnit 5 支持定义参数化测试，使同一测试方法以不同参数值多次运行。该参数可以是放入安全上下文的模拟 `Authentication`。

`@WithMockAuthentication` 独立于 Spring 上下文构建 `Authentication` 实例，因此在参数化测试中使用非常方便：

```java
@ParameterizedTest
@AuthenticationSource({
    @WithMockAuthentication(authorities = { "admin", "ROLE_AUTHORIZED_PERSONNEL" }, name = "ch4mpy"),
    @WithMockAuthentication(authorities = { "uncle", "PIRATE" }, name = "tonton-pirate")
})
void givenUserIsAuthenticated_whenGetGreet_thenOk(
    @ParameterizedAuthentication Authentication auth) throws Exception {
    final var greeting = "Whatever the service returns";
    when(messageService.greet()).thenReturn(greeting);
    api.perform(get("/greet"))
        .andExpect(status().isOk())
        .andExpect(content().string(greeting));
    verify(messageService, times(1)).greet();
}
```

上述代码的要点：

- 使用 `@ParameterizedTest` 替代 `@Test`
- 用包含 `@WithMockAuthentication` 数组的 `@AuthenticationSource` 装饰测试方法
- 在测试方法中添加 `@ParameterizedAuthentication` 参数

由于 `@WithJwt` 使用应用上下文中的 Bean 来构建 `Authentication` 实例，需要做一些额外工作：

```java
@TestInstance(Lifecycle.PER_CLASS)
class MessageServiceUnitTest {
    @Autowired
    WithJwt.AuthenticationFactory authFactory;

    private Stream<AbstractAuthenticationToken> allIdentities() {
        final var authentications =
            authFactory.authenticationsFrom("ch4mpy.json", "tonton-pirate.json").toList();
        return authentications.stream();
    }

    @ParameterizedTest
    @MethodSource("allIdentities")
    void givenUserIsAuthenticated_whenGreet_thenReturnGreetingWithPreferredUsernameAndAuthorities(
        @ParameterizedAuthentication Authentication auth) {
        final var jwt = (JwtAuthenticationToken) auth;
        final var expected = "Hello %s! You are granted with %s.".formatted(
            jwt.getTokenAttributes().get(StandardClaimNames.PREFERRED_USERNAME),
            auth.getAuthorities());
        assertEquals(expected, messageService.greet());
    }
}
```

与 `@WithJwt` 结合使用 `@ParameterizedTest` 的检查清单：

- 用 `@TestInstance(Lifecycle.PER_CLASS)` 装饰测试类
- 自动装配 `WithJwt.AuthenticationFactory`
- 定义一个方法，使用认证工厂生成所需认证的 Stream
- 使用 `@ParameterizedTest` 替代 `@Test`
- 用引用上述方法的 `@MethodSource` 装饰测试方法
- 在测试方法中添加 `@ParameterizedAuthentication` 参数

---

## 5. 使用模拟授权进行集成测试

我们将使用 `@SpringBootTest` 编写 Spring Boot 集成测试，让 Spring 装配真实的组件。为了继续使用模拟身份，配合 MockMvc 或 WebTestClient 使用。测试方法本身以及填充模拟身份的方式与单元测试相同，只有测试搭建有所不同：

- 不再需要 Mock 组件和参数匹配器
- 使用 `@SpringBootTest(webEnvironment = WebEnvironment.MOCK)` 替代 `@WebMvcTest` 或 `@WebFluxTest`，`MOCK` 环境最适合配合 MockMvc 或 WebTestClient 使用模拟授权
- 在测试类上显式添加 `@AutoConfigureMockMvc` 或 `@AutoConfigureWebTestClient` 以注入 MockMvc 或 WebTestClient

Servlet 应用的 Spring Boot 集成测试骨架：

```java
@SpringBootTest(webEnvironment = WebEnvironment.MOCK)
@AutoConfigureMockMvc
class ServletResourceServerApplicationTests {
    @Autowired
    MockMvc api;

    // 测试结构和模拟身份选项与单元测试相同
}
```

响应式应用中的等价形式：

```java
@SpringBootTest(webEnvironment = WebEnvironment.MOCK)
@AutoConfigureWebTestClient
class ReactiveResourceServerApplicationTests {
    @Autowired
    WebTestClient api;

    // 测试结构和模拟身份选项与单元测试相同
}
```

这种集成测试省去了 Mock 配置、参数捕获器等繁琐工作，但也比单元测试运行更慢、更脆弱。建议谨慎使用，覆盖率可以低于 `@WebMvcTest` 或 `@WebFluxTest`，主要用于验证自动装配和组件间通信是否正常。

---

## 6. 进阶内容

至此，我们测试的是使用 JWT 解码器、安全上下文中有 `JwtAuthenticationToken` 实例的资源服务器。所有自动化测试均使用模拟的 HTTP 请求，无需涉及任何授权服务器。

### 6.1 测试任意类型的 OAuth2 Authentication

如前所述，Spring OAuth2 安全上下文中可以存放其他类型的 `Authentication`，此时测试中应使用对应的注解、请求后处理器或变异器：

- 默认情况下，使用令牌自省的资源服务器安全上下文中有 `BearerTokenAuthentication` 实例，测试应使用 `@WithOpaqueToken`、`opaqueToken()` 或 `mockOpaqueToken()`
- 使用 `oauth2Login()` 的客户端通常安全上下文中有 `OAuth2AuthenticationToken` 实例，应使用 `@WithOAuth2Login`、`@WithOidcLogin`、`oauth2Login()`、`oidcLogin()`、`mockOAuth2Login()` 或 `mockOidcLogin()`
- 如果通过 `http.oauth2ResourceServer().jwt().jwtAuthenticationConverter(…)` 等方式配置了自定义 `Authentication` 类型，可能需要提供自定义的单元测试工具。以 spring-addons 的实现作为参考示例并不复杂，同一 GitHub 仓库中也包含了自定义 `Authentication` 和专用测试注解的示例

### 6.2 运行示例应用

示例项目配置了运行在 `https://localhost:8443` 的 Keycloak 实例（master realm）。切换到其他 OIDC 授权服务器只需修改 `issuer-uri` 属性以及 Java 配置中的权限映射器：将 `realmRoles2AuthoritiesConverter` Bean 改为从新授权服务器存放角色的私有 Claim 中映射权限即可。

关于 Keycloak 配置的更多细节，请参阅官方入门指南，独立 ZIP 发行版的方式可能是最容易上手的。

如需使用自签名证书在本地搭建带 TLS 的 Keycloak 实例，[这个 GitHub 仓库](https://github.com/ch4mpy/self-signed-certificate-generation)会很有帮助。

授权服务器至少需要：

- 两个声明用户，其中一个拥有 `ROLE_AUTHORIZED_PERSONNEL`，另一个没有
- 一个启用了授权码流程的声明客户端，供 Postman 等工具代表这两个用户获取访问令牌

---

## 7. 总结

本文探讨了在 Servlet 和响应式应用中，使用模拟身份对 Spring OAuth2 访问控制规则进行单元测试和集成测试的两种方案：

- 来自 `spring-security-test` 的 MockMvc 请求后处理器和 WebTestClient 变异器
- 来自 `spring-addons-oauth2-test` 的 OAuth2 测试注解

同时我们了解到：`@Controller` 可以使用 MockMvc 后处理器、WebTestClient 变异器或注解进行测试，但只有注解方式才能在测试其他类型组件时设置安全上下文。

完整代码可在 [GitHub](https://github.com/eugenp/tutorials) 上获取。
